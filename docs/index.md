---
layout: home
title: Practical Notes on Confidential AI
description: Benchmarks, side-channels, attestation, and hybrid trust while pushing large-model inference into confidential or verifiable environments.
---
Welcome to my collection of practical notes and thoughts on confidential AI — things I'm exploring, breaking, or benchmarking.

Confidential AI is about running LLMs and other inference workloads on private data — without the cloud or infrastructure provider being able to peek inside. It's the alternative to traditional self-hosting, where you control the server but sacrifice performance and scalability. Many are rolling their own setups right now, often ending up with limited hardware, high costs, and questionable security. This space is for digging into what actually works when you want strong security _and_ decent performance.

How does this relate to FHE? Fully Homomorphic Encryption is the theoretical holy grail — you can compute on encrypted data without ever decrypting it. But today, FHE is still slow and mostly academic for real-world LLM inference. Confidential computing is the "now" solution: it gives you strong privacy by protecting your data in-use, with practical performance on current hardware. Both have the same goal, just radically different trade-offs for speed, usability, and trust assumptions.
