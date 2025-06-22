---
title: "Web Browser"
date: 2025-06-22T01:14:54+05:30
slug: "web-browser"
category: projects
summary: ""
description: ""
cover:
  image: ""
  alt: ""
  caption: ""
  relative: true
showtoc: true
draft: true
---


Motivation for my projects comes from a YouTube video where I saw Alexa Fazio showcasing her [projects](https://www.youtube.com/watch?v=6CJiM3E2mAA&t=1s).

---

## What is a Web Crawler and What Does It Do?

A **web crawler** simply visits a webpage, extracts all the hyperlinks from the page, and recursively visits those new pages. It can also extract metadata such as the **page title** and potentially the **full content**. 

One major use case is how the **Google Search engine** works.

> **Note:** The **Internet is not Google**.  
> Google first extracts information from the internet (such as links, page titles, meta tags, headings, etc.), organizes it—this is called **indexing**—and then uses its algorithm to present that information beautifully when you search for something.  
> A web crawler is used for this indexing process. The web crawler Google uses is known as **Googlebot**.

---

## My Web Crawler in Go

I have developed a web crawler in **Go**.

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

Features of my web-cralwer

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


