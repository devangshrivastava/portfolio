---
title: "Web crawler"
date: 2025-06-22T01:14:54+05:30
slug: "web-crawler"
category: projects
summary: ""
description: ""
cover:
  image: ""
  alt: ""
  caption: ""
  relative: true
showtoc: true
draft: false
---

[Github Repo](https://github.com/devangshrivastava/webCrawler)

[DeepWiki Analysis](https://deepwiki.com/devangshrivastava/webCrawler)

---

## What is a Web Crawler and What Does It Do?

A **web crawler** simply visits a webpage, extracts all the hyperlinks from the page, and recursively visits those new pages. It can also extract metadata such as the **page title** and potentially the **full content**. 

One major use case is how the **Google Search engine** works.

> **Note:** The **Internet is not Google**.  
> Google first extracts information from the internet (such as links, page titles, meta tags, headings, etc.), organizes it—this is called **indexing**—and then uses its algorithm to present that information beautifully when you search for something.  
> A web crawler is used for this indexing process. The web crawler Google uses is known as **Googlebot**.

---

## How to Build a Simple Web Crawler

1. **Start with a seed URL** – your entry point into the web.
2. **Parse the HTML** of the page:
   - Extract links
   - Extract the title
   - Optionally extract meta tags, body text, or even the full content
3. Think of the internet as a **massive graph**:
   - **Webpages are nodes**
   - **Links between pages are edges**
4. Choose a traversal strategy:
   - **Breadth-First Search (BFS)**
   - **Depth-First Search (DFS)**
   - Or a **hybrid approach**
5. Since the internet is huge, **limit your crawl** — for example, to only 100 pages.

Yep, it’s that simple to make one (or is it...?).

## 🧠 Features of My Web-Crawler

### 🔁 Customizable Strategy (BFS / DFS / Mixed)
You can actually choose how you want to explore the web-graph—layer by layer (BFS), deep dive (DFS), or a spicy mix of both (like 30% DFS, 70% BFS). Just pass it as a flag. Boom.

### 🔗 Smart Link Handling (Absolute + Relative URLs)
It doesn’t just follow obvious links like `https://...`. It’s smart enough to resolve relative links like `/about` into full URLs and crawl those too.

### 🤖 Respects Robots.txt (Be a good bot)
Doesn’t just barge into places. It politely checks `robots.txt` before visiting any page. If a page says "no bots allowed," my crawler listens.

### 🕓 Per-host Rate Limiting (No Hammering Servers)
It won’t slam a single site with 100 requests per second. Each domain has its own mini-speed limit to avoid getting IP-banned.

### 🧽 Clean Content Extraction
Instead of grabbing garbage like navbar text, it uses a smart DOM parser to keep just the juicy stuff—headings, paragraphs, main content. Also extracts the page title.

### 🔤 Word Count + Tokenized Text
Each page is not just dumped as raw text; it’s processed into clean tokens and stored with word count metadata. Useful if you're building a search engine later.

### 🗃️ MongoDB Powered Archive
Everything it finds—URLs, titles, content, word count—is stored in MongoDB. So you’re basically building your own mini-Google index.

### ⚙️ Fully Concurrent & Modular
Uses Go's goroutines to crawl multiple pages at once. And the code is modular AF—cleanly split into parser, crawler, frontier, storage, etc. Easy to extend.


---

# overview 

We start by defining all the configurations the app supports:

- Seeds:           Entry point(s) for the crawler
- MaxPages:        Maximum number of pages to crawl
- Workers:         Number of parallel worker goroutines
- Strategy:        Crawling strategy: bfs, dfs, or mixedN
- RequestsPerHost: Max requests per second per domain (rate limit)
- RobotsTimeout:   Timeout for fetching robots.txt
- UserAgent:       User-Agent header for HTTP requests
- Tokens:          Max number of tokens to parse from HTML content



## `main.go` →

```go
package main

import (
	"flag"
	"log"
	"time"
	"crawler-go/internal/crawler"
)

func main() {
	seed    := flag.String("seed", "https://www.cc.gatech.edu/", "initial URL to start crawling from")
	max     := flag.Int("maxPages", 5000, "stop after N pages")
	workers := flag.Int("workers", 32, "number of parallel fetchers")
	strat   := flag.String("strategy", "bfs", "can choose bfs (breadth-first), dfs (depth-first), or mixedN (bfs+dfs)")
	rps     := flag.Float64("maxPerHost", 2.0, "max requests/sec to one host")
	ua      := flag.String("userAgent", "GoCrawler/0.2", "HTTP User-Agent string")
	rt      := flag.Int("robotsTimeout", 5, "robots.txt timeout (sec)")
	token   := flag.Int("tokens", 5000, "max number of tokens to parse in HTML content")

	flag.Parse()

	opts := crawler.Options{
		Seeds:           []string{*seed},
		MaxPages:        *max,
		Workers:         *workers,
		Strategy:        *strat,
		RequestsPerHost: *rps,
		RobotsTimeout:   time.Duration(*rt) * time.Second,
		UserAgent:       *ua,
		Tokens:          *token,
	}

	if err := crawler.Run(opts); err != nil {
		log.Fatal(err)
	}
}



In this file, we primarily configure the crawler via command-line flags. The Options type is a custom struct defined in options.go and contains the following schema:


type Options struct {
	Seeds          []string
	MaxPages       int
	Workers        int
	Strategy       string
	RequestsPerHost float64       // NEW (rps)
	RobotsTimeout  time.Duration  // NEW
	UserAgent      string         // NEW
	mixPct        int            // NEW (0 = pure BFS, 100 = pure DFS)
	Tokens         int            // NEW (max tokens to parse in HTML content)
}

```
## `Run` Function → `internal/crawler/engine.go`



In the `Run` function, the first step is initializing MongoDB (we clear previous data; this behavior can be changed in `internal/storage/mongo.go`).

```go
func Run(opts Options) error {
	_ = godotenv.Load()

	// ----- MongoDB -----------------------------------------------------------
	access := os.Getenv("MONGODB_URI") != ""
	fmt.Println("Connecting to DB at:", os.Getenv("MONGODB_URI"))
	if !access {
		fmt.Println("MongoDB access disabled, running in no-op mode")
	}
	store, err := storage.New(os.Getenv("MONGODB_URI"), access)
	if err != nil {
		return fmt.Errorf("failed to connect to MongoDB: %w", err)
	}
	defer store.Close()

	// ----- Frontier / visited sets ------------------------------------------
	queue := frontier.NewQueue()
	visited := frontier.NewVisited()
	for _, s := range opts.Seeds {
		queue.Enqueue(s)
	}

	// ----- Host politeness manager ------------------------------------------
	hm := hostman.New(opts.UserAgent, opts.RequestsPerHost, opts.RobotsTimeout)

	// ---------- Strategy RNG ----------
	rng := opts.prepare()

	// ----- Worker pool channels ---------------------------------------------
	jobs := make(chan string, opts.Workers*2)
	wg := sync.WaitGroup{}
	ctx, cancel := context.WithCancel(context.Background())

	// -----------------------------------------------------------------------
	// METRICS SERVER  →  http://localhost:2112/metrics
	// -----------------------------------------------------------------------
	go func() {
		http.Handle("/metrics", promhttp.Handler())
		if err := http.ListenAndServe(":2112", nil); err != nil {
			fmt.Println("metrics server:", err)
		}
	}()

	// ----- Dispatcher --------------------------------------------------------
	go func() {
		for {
			if visited.Size() >= opts.MaxPages {
				cancel()
				close(jobs)
				return
			}

			// choose front/back according to strategy
			webURL, ok := opts.SelectURL(queue, rng)
			if !ok { // frontier empty
				time.Sleep(100 * time.Millisecond)
				continue
			}

			// politeness checks (robots + rate-limit)
			parsed, err := url.Parse(webURL)
			if err != nil {
				continue
			}
			if allow, wait := hm.Check(parsed); !allow {
				continue
			} else if err := wait(ctx); err != nil {
				continue
			}

			jobs <- parsed.String() // enqueue for workers
		}
	}()

	// ----- Workers -----------------------------------------------------------
	for i := 0; i < opts.Workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			runWorker(ctx, jobs, visited, queue, store, opts.Tokens)
		}()
	}

	// ----- Stats ticker ------------------------------------------------------
	start := time.Now()
	ticker := time.NewTicker(1 * time.Minute)
	go func() {
		for t := range ticker.C {
			fmt.Printf("[%.0f min] crawled=%d queued=%d\n",
				t.Sub(start).Minutes(), visited.Size(), queue.Size())
		}
	}()

	wg.Wait()
	ticker.Stop()

	fmt.Println("------- FINAL STATS -------")
	fmt.Printf("Crawled: %d pages\n", visited.Size())
	fmt.Printf("Queued : %d (never visited)\n", queue.Size())
	return nil
}

```


Then, to traverse the web as a graph, we create:

- A **queue** to manage URLs to visit  
- A **visited set** to avoid cycles

We also configure a **host manager** (`hostman`), which:

- Parses and stores `robots.txt` for each domain  
- Ensures we only crawl allowed paths  
- Applies rate limiting using a token bucket system per host

After that, we create one goroutine for the **dispatcher** and multiple goroutines (as per `Workers`) acting as **workers**.

The dispatcher is responsible for selecting URLs (based on the strategy), validating them, and pushing them into a channel.  
This channel (`jobs`) is read by workers—effectively implementing a multithreaded crawling system.

---

## Dispatcher Logic

The dispatcher is a simple `for` loop that traverses the internet using the selected strategy.  
The logic is handled in `SelectURL()` inside `options.go`, which applies BFS, DFS, or mixed strategy.

Once a URL is selected:

- It checks if crawling that URL is allowed using the host manager (`robots.txt` + rate limit).
- If allowed, it sends the URL into the `jobs` channel.
- Workers fetch url fomr here and they parse it.




## 🧵 Worker Logic

Each worker goroutine runs the `runWorker` function.

### Here's how it works:

1. **Context Check:**  
   The worker first checks `ctx.Done()` to see if it should terminate early. This allows graceful shutdowns.

2. **Job Channel Read:**  
   The worker reads a URL from the `jobs` channel. If the channel is closed, the worker exits.

3. **Visited Set:**  
   It checks whether the URL has already been visited. If yes, it skips processing.

4. **Fetch Page:**  
   Calls `fetch(url)` to download the HTML content. If the body is empty (error or blank page), it skips.

5. **Extract and Parse:** (crawler-go/internal/parser)
   - `Extract`: Retrieves page title, clean content, and word count.
   - `ParseHTML`: Finds all hyperlinks on the page.

6. **Store Result:**  
   Builds a `Webpage` object and inserts it into MongoDB via `store.Insert()`.

7. **Enqueue New Links:**  
   All new links are added to the frontier queue (if not already visited).

This repeats for every job received through the channel.


