# Why Concurrency?

> **Difficulty:** 🟢 Beginner  
> **Reading Time:** ~20 minutes  
> **Prerequisites:** None

---

## Introduction

Imagine opening your favorite web browser.

Within a fraction of a second, it begins performing dozens of different tasks:

- Rendering the user interface
- Loading web pages
- Downloading images
- Executing JavaScript
- Playing videos
- Receiving keyboard and mouse events
- Saving browsing history
- Communicating with remote servers

To you, everything appears to happen simultaneously.

But here's the interesting part:

> A CPU core can execute only one instruction at a time.

So how can a computer appear to perform so many activities at once?

Answering this question is the first step toward understanding **concurrency**.

Before learning about Java threads, we first need to understand why concurrency exists at all.

---