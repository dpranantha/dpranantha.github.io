---
layout: post
title: " Learning Go by Building a Local LLM API: My Journey with Gin, Fiber, and React"
permalink: /2025-04-24-learning-go-build-llm-api
date: 2025-04-24 15:15:00+01:00
categories:
- Software development
tags:
- Go
- Golang
- LLM
- Large Language Model
collection: blog
---

### 🚀 Introduction

As a developer already familiar with languages like Kotlin, Java, and Rust, I decided to dive into the world of Go by building something hands-on and exciting: a Local LLM API using Go and React. My goal was to implement an API supporting REST, Server-Sent Events (SSE), and WebSocket communication with a simple front-end for interaction. Along the way, I learned a lot—not just about Go, but also about structuring scalable projects, integrating with local LLMs like LLaMA 2 via Ollama, and using frontend technologies I hadn’t fully explored before.

### 🛠️ Project Overview: Go LLM API

The project is structured with two server versions—using Gin and Fiber, two popular Go web frameworks. It enables local LLM inference via Ollama and supports:

✅ REST and GraphQL APIs

✅ Streaming via WebSocket and SSE

✅ Simple React-based UI for interaction

You can toggle between Gin and Fiber using a single environment variable:

```
make run            # Gin
make run FRAMEWORK=fiber  # Fiber
```

### 🧠 Motivation and Why Go?

After working with Rust, I was curious how Go would feel in comparison. The lack of manual memory management, no ownership model, and a garbage collector (GC) made things smoother—especially for someone who’s already wrestled with lifetimes and borrowing in Rust.

Also, Go compiles into a single binary containing all runtime dependencies—including the GC and standard library—which makes distribution and deployment refreshingly simple.

### 🧩 Challenges: Learning Through Friction

1. Go Package Structure and Dependency Management
One of the steepest parts of the learning curve was organizing my Go code. Unlike OOP-heavy languages like Java or Kotlin, Go encourages composition over inheritance. Getting used to packages, interfaces, and dependency injection patterns took some trial and error. Eventually, following community conventions and reading source code from popular Go repos helped a lot.

2. React Framework
I had limited exposure to React before, so building a chat UI in React felt like learning a new language alongside Go. Fortunately, the build tools were straightforward (npm, vite, etc.), and the structure of components was similar enough to other frontend libraries I’d seen.

3. Learning GraphQL
This project also gave me the opportunity to explore GraphQL for the first time. I implemented a simple query endpoint to interface with the REST API, and it turned out to be a clean way to structure and test non-streaming requests. GraphQL’s flexibility and introspection made it an enjoyable addition to the stack.

### 📚 Learning Materials

I’m currently reading Learning Go by Jon Bodner, which has been a great companion throughout this process. It breaks down Go's syntax, idioms, and concurrency primitives in a way that’s accessible but deep.

### 🔬 Roadmap: What's Next?

Here’s what I plan to explore next in this project:

- 🔧 Add .env and Docker support: For better configuration and deployment

- 🧠 Memorization and Summarization: Potentially on the server (more efficient) vs. client (more privacy); will evaluate based on use cases

- 📦 Refactor LLM logic into reusable utilities

- 🔄 Benchmarking against Rust: Curious how Go stacks up performance-wise

- 🧵 Goroutines and Channels: Haven’t dived deep yet, but I’m intrigued by Go’s concurrency model and want to explore it for background jobs or concurrent inference

### 💡 Takeaways

- Go's simplicity is deceptive—in a good way. It feels lightweight, but powerful. 

- The mental model is easier than Rust (see Comparing to Rust section). 

- The Go ecosystem is mature, and the tooling is a joy to use. 

- Building a real-world app (even a small one) is still the best way to learn.

### ⚖️ Comparing to Rust
After learning Rust, Go feels refreshing. There's:

- ✅ No ownership or lifetimes to manage
- ✅ Garbage collection
- ✅ Single statically linked binary output
- ✅ Easier syntax for small tools and services

This simplicity makes Go feel approachable—yet still powerful. I’ll eventually benchmark this API against a Rust version to compare raw performance.

### 💡 Final Thoughts
This project barely scratches the surface of what Go can do—but it’s been a fantastic way to learn by building. I now feel more confident writing idiomatic Go, understanding its concurrency model, and integrating it into a modern web stack.

If you're also learning Go, I recommend picking a small but real project like this. It’s the best way to learn fast and build something fun in the process.

### 🧪 Try It Yourself!

Clone the project, follow the README, and run local LLM inference in minutes:

```
git clone https://github.com/dpranantha/go-llm-api.git
cd gollm-api
make run
```

#### 💬 Using Chat UI
Navigate to the frontend:

```
cd gollm-api/front-end/gochat
npm install
npm run dev
```

Open one of the chat interfaces in your browser:

REST API (non-streaming): http://localhost:5173/chat – Uses GraphQL

WebSocket (streaming): http://localhost:5173/chatws – Uses REST + WebSocket

SSE (non-standard POST streaming): http://localhost:5173/chatsse – Uses REST SSE

### 📚 Next on the blog:

Stay tuned—I'll be writing a follow-up post summarizing the key takeaways from [Learning Go by Jon Bodner](https://www.amazon.nl/Learning-Go-Jon-Bodner/dp/1492077216).