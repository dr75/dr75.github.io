---
layout: home
title: Practical Notes on Confidential AI
description: Benchmarks, side-channels, attestation, and hybrid trust while pushing large-model inference into confidential or verifiable environments.
---
Welcome to my collection of practical notes and thoughts on confidential AI — things I'm exploring, breaking, or benchmarking.

**Confidential AI** is about running LLMs and other inference workloads on private data — without the cloud or infrastructure provider being able to peek inside. It's the alternative to traditional self-hosting, where you control the server but sacrifice performance, scalability, high costs, and questionable security.
It's also the practical alternative to _FHE (Fully Homomorphic Encryption)_, where you compute on encrypted data without decrypting — the holy grail of privacy. But it’s still too slow for real-world LLMs. Confidential computing protects data in use with great performance on current hardware.
