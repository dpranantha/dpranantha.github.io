---
layout: post
title: " Building LLMs (from Scratch) by Sebastian Raschka – A Deep Dive into the Foundations of Modern Language Models Building a LLM"
permalink: /2025-04-10-building-llm-from-scratch
date: 2025-04-10 17:15:00+01:00
categories:
- Data science
tags:
- LLM
- Large Language Model
collection: books
---


As the field of Large Language Models (LLMs) continues to evolve at lightning speed, it becomes increasingly important not just to use these models, but to understand how they work under the hood. The book Building LLMs from Scratch with PyTorch is a refreshing, hands-on resource that takes this philosophy seriously. It's not just another “how-to”; it’s a from-first-principles guide that walks you through what makes LLMs tick, line by line of code.

In this review, I’ll break down why [this book](https://amzn.eu/d/be8CowF) is so useful—especially if you want to learn about LLMs in detail, understand their architecture, and get your hands dirty with PyTorch-based implementation.

### 📘 A Solid Introduction: What Are LLMs and Why They Matter?

The book begins with a thoughtful and clear introduction to LLMs—what they are, where they’re used (e.g., chatbots, summarization, translation), and how they fit into the broader AI landscape. It walks through architectural basics, including the difference between encoder, decoder, and encoder-decoder architectures, with an emphasis on the decoder-only model, which is foundational to GPT-style LLMs.

This section lays the groundwork and is especially helpful for readers who may know what transformers are but don’t yet have an intuitive understanding of how they work.

### 🧠 Text as Data: Embeddings, Tokenization, and Representation
One of the strengths of this book is how it demystifies the way text is transformed into data. The authors take their time to explain:

- Embeddings as a way to represent words in continuous vector space
- Tokenization strategies, including Byte Pair Encoding (BPE)
- The use of sliding windows for handling long documents
- Word position embeddings to capture sequence order in an otherwise order-agnostic architecture

It even touches briefly on n-gram mechanics and introduces word2vec as a conceptual precursor to transformer-based models—an excellent move that helps bridge traditional NLP with modern deep learning approaches.

### ⚡ The Magic of Attention
The next segment focuses on the attention mechanism—arguably the core innovation behind transformers and LLMs. The book does a great job explaining:

- Why attention is crucial for modeling long-range dependencies
- How self-attention enables the model to attend to different parts of the input
- How it starts with non-trainable weights and builds up to trainable attention
- The use of causal masking to enable next-word prediction (by hiding future tokens)
- Techniques like dropout for regularization and robustness

The evolution from single-head to multi-head attention is also clearly explained and coded, reinforcing both conceptual and practical understanding.

### 🧱 Building GPT from Scratch

With the theory covered, the book transitions into implementing a GPT-style architecture from scratch. This includes:
- Connecting attention layers with linear layers inside a transformer block
- Using Layer Normalization and GELU activations
- Stacking multiple transformer blocks to deepen the network
- Pretraining on unlabeled data with cross-entropy loss
- Explaining backpropagation, gradient flow, and how the model learns to maximize the likelihood of correct tokens

It also explains perplexity as an evaluation metric—essentially an exponentiation of cross-entropy that helps quantify how well the model predicts real text distributions.
The use of Adam vs. AdamW optimizers is explained in the context of regularization, with a clear preference for AdamW due to its improved generalization performance.

### 🎲 Controlling Generation: Decoding Strategies
Text generation is not just about prediction; it's also about controlling randomness and creativity. This section offers an excellent breakdown of decoding techniques:
- Temperature scaling to adjust output distribution sharpness
- Top-k sampling to limit the candidate pool
- Discussion of the top-p (nucleus sampling) technique, even though it's only mentioned in passing—something that could be expanded on
- Saving and loading trained models, including loading pretrained GPT-2 from OpenAI

### 🧪 Fine-Tuning for Classification
Next, the book moves into fine-tuning pretrained models for downstream tasks—specifically binary classification (e.g., spam detection). It covers:
- Adding a classification head on top of the transformer
- Replacing the output layer to map to two classes
- Training only selected layers vs. full fine-tuning
- Calculating classification loss and accuracy using softmax on the last token's output
- Visualizing training progress with Matplotlib and deciding how many epochs are sufficient

This section is highly practical and helps solidify the idea of transformers as flexible backbones for many NLP tasks.

### 🧩 Fine-Tuning for Instruction Following
The final chapter touches on instruction tuning—a crucial stepping stone toward models like ChatGPT. Here, the focus shifts from classification to dialogue-style datasets, where:
- Dataset preparation is more labor-intensive, involving prompt-response pairs
- Formatting (prompt engineering) is discussed briefly
- Evaluation becomes more subjective and qualitative
- The Alpaca dataset is used as an example

This section provides a good bridge into more advanced topics like reinforcement learning from human feedback (RLHF) and qualitative evaluation—topics further explored in books like Chip Huyen’s AI Engineering.

### 🧠 Final Thoughts
Overall, Building LLMs from Scratch with PyTorch is an excellent resource for anyone who wants to go beyond using prebuilt models and truly understand how LLMs are constructed, trained, and fine-tuned. It is:

- ✅ Clear and well-paced in its explanations
- ✅ Hands-on, with PyTorch code that brings theory to life
- ✅ Thorough, covering everything from tokenization to training and decoding
- ✅ A great stepping stone for advanced topics in AI engineering

While some areas (like top-p sampling or RLHF) are only briefly mentioned or omitted, the core concepts are all covered in depth, and the book does a fantastic job of showing how the pieces fit together.

If your goal is to become a builder of LLMs, not just a user—this book is a great place to start.

### 📚 Further Reading
Depending on your goals, here are a few other highly recommended reads to complement this book:

- 🧩 [Hands-On LLMs by Jay Alammar & Maarten Grootendorst](https://amzn.eu/d/9Re2kiS)
If you're interested in using LLMs effectively—think prompt engineering, real-world applications, and patterns for summarization, generation, or RAG (retrieval-augmented generation)—this book is packed with practical techniques and examples. It’s great for product builders, data scientists, and anyone working on integrating LLMs into apps.

- 🛠 [AI Engineering by Chip Huyen](https://amzn.eu/d/cIIe4jE)
This one zooms out to the engineering side: how to bring LLM systems to production, set up feedback loops, monitor drift, and evaluate models in real-world settings (including using AI-as-a-judge or human-in-the-loop approaches). Especially useful if you're involved in scaling or operationalizing AI systems.

