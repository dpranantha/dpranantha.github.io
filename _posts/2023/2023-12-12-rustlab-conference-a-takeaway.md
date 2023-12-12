---
title: "Insights from a Rust Conference - Rustlab IT 2023"
permalink: /insight-from-rustlab-it-2023
date: 2021-02-09 21:30:00+01:00
categories:
- Coding
tags:
- Rust
- Backend development
- Machine learning
- Data science
---

Rust, a programming language celebrated for its commitment to performance, reliability, and memory safety, has found substantial success in domains like kernel and system programming. Its memory model, ensuring predictable performance and reliability without relying on a garbage collector, positions it as an enticing option for specific backend scenarios. Despite these advantages, Rust faces challenges that have slowed its widespread adoption. This article delves into the potential niche for Rust, examining both the obstacles and prospects within this dynamic landscape.

### What's Holding Rust Back?

1. Limited Talent Pool:
   The scarcity of professionals experienced in Rust poses a significant hurdle for backend development, impeding its broader adoption for projects requiring this language's prowess.

2. Ecosystem Complexity:
   Described as Lego-like, the Rust ecosystem can overwhelm beginners, especially with the introduction of asynchronous programming in 2019. The resultant surge in crates complicates project initiation, requiring a curated set of crates with coordinated versioning for improved interoperability.

3. Learning Curve:
   Rust's learning curve, especially when combined with asynchronous programming, can be suboptimal for beginners. The language's advanced constructs, while powerful, might be intimidating, and the plethora of choices can be bewildering. The shortage of experienced mentors further exacerbates the learning curve.

### How to address challenges?

To overcome these challenges and foster Rust's adoption in backend development, Luca Palmieri, the author of "Zero To Production In Rust," proposes key focus areas:

1. High-Quality Error Messages:
   Rust frameworks should prioritize error messages that resonate with the language of backend developers, enhancing debugging and troubleshooting intuitiveness.

2. Compile-Time Error Catching:
   Rust frameworks should catch errors at compile time, ensuring early identification and resolution of issues for enhanced overall code robustness.

3. Boring Rust for Majority Tasks:
   Encouraging the use of "boring" Rust for routine backend tasks can simplify development without compromising performance.

4. Problem Domain-Focused APIs:
   Rust frameworks should emphasize building APIs with comprehensive features, addressing common concerns such as authentication, logging, and metrics to streamline development.

One notable early-stage effort is Pavex, a framework for building APIs in Rust, taking a unique route from other famous Rust web frameworks like Actix and Rocket. Pavex operates as a specialized compiler for building Rust APIs, generating a standalone API server SDK crate according to specifications.

### Rust’s expansion

Despite challenges in backend engineering, Rust is gradually making strides into machine learning engineering (MLE) and data science (DS) fields. These domains demand optimized high-performance and efficient computing, areas where Rust excels. Notable examples include the Polars data frame interface, Qdrant vector databases,  and Hugging Face’s Candle ML framework, showcasing Rust's potential in these evolving fields. While choices for productivity are currently limited, Rust's entry into these domains signals a promising trajectory.

### In Conclusion

While Rust may not emerge as the default choice for mainstream backend development, its promise shines in niche areas requiring high-performance applications with a lower infrastructure footprint and stringent reliability requirements. As Rust continues to address ongoing challenges, it holds substantial potential in machine learning engineering and data science domains. The maturation of the ecosystem and the proliferation of expertise suggest that Rust's role in backend development may expand, providing an appealing alternative for projects seeking both performance and reliability. The journey of Rust, marked by innovation and collaboration, stands as a testament to its evolving significance in the programming landscape. For those considering Rust, starting in machine learning engineering and data science fields may offer an acceptable entry point.

