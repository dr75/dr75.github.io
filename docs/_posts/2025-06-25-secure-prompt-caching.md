---
title: "Secure Prompt Caching for Fast AI Inference"
date: 2025-06-25
layout: post
---

# Secure Prompt Caching for Fast AI Inference

Large language models (LLMs) have evolved into **stateful systems that retain conversation history in memory** to deliver responses faster and more efficiently. This is possible with prompt caching, a mechanism that keeps the LLM's internal state on the server in memory, thereby avoiding recomputation of the state on every request.

> _Prompt caching is a performance optimization in LLM inference and is the default among service providers. It is crucial for fast responses in multi-turn conversations and agentic systems. But storing context on the server introduces a security risk._

[Privatemode](https://www.privatemode.ai/) is the first AI inference service to support prompt caching with verifiable security, thanks to [confidential computing](https://www.edgeless.systems/wiki/what-is-confidential-computing/) and public source code. In this post we explain how we do it.

## Background: What are KV Cache and Prompt Caching?
The [transformer architecture](https://proceedings.neurips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf) introduced [_attention_](https://en.m.wikipedia.org/wiki/Attention_(machine_learning)) as a key mechanism for the model to understand the relative importance of input tokens in LLM inference. To compute attention for a token, so called _Key_ and _Value_ vectors of all other tokens must be computed, which is the main cause of latency when processing queries with large context.
The _Key-Value (KV) cache_ is a data structure in the inference engine for caching those vectors of all processed tokens to avoid recomputation for each newly generated token. The attention result for the next token is then computed using the cached Keys and Values of previous tokens. The current tokens' Keys and Values are appended to the cache, saving both time and compute. This allows for high generation speed and has become the state-of-the-art.

**_Prompt caching_**, or _context caching_, extends this idea by keeping the KV cache of an entire conversation across multiple requests. This allows multi-turn conversations or agent-driven workflows to continue seamlessly without having to start from zero every with every request.
For LLM inference, this means that the KV cache is kept on the server for longer periods of time, in unencrypted RAM or [even on hard disks](https://api-docs.deepseek.com/news/news0802). This is true for most inference service providers but details on how it is done are scarce.

> _"But it’s not the prompt itself, just an internal representation, a bunch of numbers. So why should I care?"_

Indeed, not the plain text prompt is stored, but the KV cache is just a different representation of the same data. It is what the LLM _sees_ and what it _thinks_ about your prompt, and if asked, a model could tell you what was in the prompt just using the KV cache. Security-wise, reading the KV cache is basically as good as reading the prompt itself, also for huge prompts.

So, how can it be made secure? You guessed it - confidential computing.

## Keeping Your Prompts Secure
With [confidential computing](https://www.edgeless.systems/wiki/what-is-confidential-computing/), data is not only encrypted during transit or at rest, but also in working memory and shielded from external access, including the host operating system, the rest of the infrastructure, and even the cloud service provider. Since [NVIDIA's H100](https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/), confidential computing can now also be extended to GPUs. By deploying LLMs within these confidential environments, we can safely cache prompts without the risk of exposing them. Even if the cloud provider or sysadmin are malicious, your prompts remain secure, verifiably so, due to the properties of confidential computing and the transparency of public source code.

### Side-Channel Leaks
While confidential computing protects data even during inference, it does not eliminate all risks. One key security challenge in computer systems is guarding against [side-channel attacks](https://en.wikipedia.org/wiki/Side-channel_attack)—in this case, timing-based attacks that could reveal whether a prompt was cached, allowing attackers to infer which prompts are currently stored in a system.

Privatemode uses [vLLM](http://docs.vllm.ai/en/stable/), a high-speed LLM inference engine. To use prompt caching, we had to protect vLLM’s [prefix cache](https://docs.vllm.ai/en/stable/features/automatic_prefix_caching.html) (its implementation of a prompt cache), against such side-channel leaks. We found that the cache [introduces measurable differences in response times](https://github.com/vllm-project/vllm/security/advisories/GHSA-4qjh-9fv9-r85r) between cache hits and cache misses. This timing gap becomes an attack surface: An attacker using the same backend could craft inputs and measure response times to determine whether a specific prompt has been seen before. This allows them to reconstruct a prompt step by step.

### Understanding the Attack Scenario
Let’s walk through a concrete attack scenario. Imagine an attacker who has access to the inference system you are using. This could be the service provider of the system, an administrator of a self-hosted system, or another regular user. If using confidential computing, they can’t read memory directly, which already covers many threats. With prompt caching, they can interact with the service and measure response times to extract the prompt of another user.

Here is how they would do it: The attacker begins by determining the length of the system prompt, for example, by examining token usage or measuring response times for their own requests with increasing lengths. Once the length of the system prompt is known, they can start to infer the prompt content. Assuming the cache operates in blocks of 16 tokens, the attacker needs to infer those 16 tokens, or ~10 words. This lets them test one block at a time.

With each request, they observe how long the system takes to respond:
- **Cache miss:** The response takes noticeably longer, since the system must compute the result from scratch.
- **Cache hit:** The response is faster, as it reuses the cached data.

This timing difference—sometimes as much as 15 ms on an NVIDIA H100 with Llama 3.3 setup—is enough for an attacker to know whether their input matches a cached block of tokens. They can use this feedback to extract an entire prompt block by block. While the number of possible inputs is very high, it can be reduced to only meaningful content using an approach based on dictionaries or even LLMs to sample valid text. Also, the first block is easier to infer as the system prompt is included, likely not aligning with the 16-token block boundary. After matching the first 16-token block, the attacker proceeds to the next, using previously discovered context to guide their attack.

This isn’t just a theoretical risk. [Research has shown](https://arxiv.org/html/2411.18191v1) that these attacks are effective in practice, exposing sensitive medical data. Attackers don’t need privileged access: They might be outsiders probing a public API, or insiders within your own organization, inferring patient records, legal documents, or other confidential messages recently handled by the system.

### How We Prevent Side Channel Attacks
To address the issue, we developed vLLMs [cache salting](https://github.com/vllm-project/vllm/issues/16016) mechanism, which separates prompt caches of different users. When prompts are stored or looked up in the cache, a secret, user-provided “salt” is added[^1], making otherwise identical inputs appear unique to the cache unless the user has permission. It is like providing random inputs in front of your prompt such that it becomes infeasible to infer it, except that we do it for every block, protecting the entire prompt.

[^1]: Note that the cache salt here is not just a typical cryptographic salt; it also protects the contents of the prompt. For this reason, it must remain secret, unlike conventional salts that do not need to be kept confidential.

<img src="../img/cache.png" alt="alt text" style="max-width:600px;">

As shown in the figure above, the user provides a cache salt together with the prompt.
For efficiency reasons, [vLLM organizes the prompt cache in blocks of tokens](https://blog.vllm.ai/2023/06/20/vllm.html) (two tokens in this example; usually longer).
The salt is added to the first block of tokens when computing the cache lookup key by hashing input tokens. To avoid hash collisions, we added support for [hashing of tokens using SHA-256](https://github.com/vllm-project/vllm/pull/15297). The hash of a block is an input for the hash of the next block, creating a unique sequence of cache lookup keys. This is needed as attention values contain information of tokens that came before in the context such that blocks cannot be reused in isolation. The salt does not have to be added directly to every hash as it is implicitly propagated via the chain of hashes. Incomplete blocks are not cached, i.e., Attention Keys and Values are recomputed for those tokens.

With this mechanism, a user can only access the cache in follow-up requests when providing the same cache salt. An attacker who doesn't know the secret salt won't get any cache hits and therefore can't measure differences in response times. This finally allowed us to enable prompt caching.

## Speed Gains
With prompt caching enabled, we can accelerate AI inference for agentic workloads and improved UX. We measured the impact of prompt caching on end-to-end response times using vLLM with cache salting, a quantized Llama 3.3 70B, an NVIDIA H100 GPU with confidential computing, and requests of different lengths (1,000 tokens and 10,000 tokens). We first sent an initial request with a document in context and then a follow-up request that contains the same document, once _without caching_ and once _with caching_ enabled.

<img src="../img/cache_perf.png" alt="alt text" style="max-width:480px;">

The results are striking, prompt caching minimizes response latency, especially with large contexts. Processing a document with 10,000 tokens (about 6,000 words or 12 pages of text) takes about 9 seconds without prompt caching. Every follow-up question to the model would also take 9 seconds or longer without the cache. With prompt caching enabled, this delay is eliminated by using cached attention inputs, allowing the model to start responding immediately after one second.


## Flexible and Collaborative Prompt Caching in Privatemode AI

Our implementation of prompt caching in Privatemode not only ensures the confidentiality of conversations for individual users but also enables granular configuration of a cache shared among trusted groups of users. This enables our customers to more effectively scale agentic workloads and other use cases across multiple users.

For example, coding agents work with large code bases and usage within a team reuses the same common context. By sharing a cache for users in a team, frequent recomputation of common code context is avoided, reducing response times. Similarly, teams working with large, sensitive documents benefit from fast responses due to shared cached document context while preventing attacks from other users that can access the system.

In our API, prompt caching is opt-in and [highly configurable](https://docs.privatemode.ai/guides/proxy-configuration#prompt-caching). You can choose to cache prompts for a single user or to share a prompt cache within a team, a project, or even a larger organization. And if you need extra control, you can always disable the shared [cache for specific queries](https://docs.privatemode.ai/guides/prompting#prompt-caching) that are more sensitive or private. The cache salt is generated on the client side and must stay secure. You can leave this to Privatemode, which uses secure random salts, or manage it yourself, e.g., to share a prompt cache within a team.

In the [Privatemode app](https://www.privatemode.ai/chat), prompt caching is enabled by default—but only for the same user. That is, long conversations or uploaded documents are processed fast and there’s no cache shared between users unless explicitly configured.

This flexible setup means you get all the speed and efficiency benefits of caching, while keeping your prompts always confidential and under your control.

## Conclusion
With confidential computing and public source code, security is verifiable by anyone. Prompt caching gives you fast responses for repeated context, but the cache must be secured. We introduced cache salting in vLLM, an open, secure, and flexible way to protect your cached prompts against timing-based attacks, also in multi-user and team settings. Combining this with confidential computing, you get _fast and secure_ AI inference.

If you are running vLLM in a multi-tenant environment, we encourage you to [enable secure SHA-256 hashing](https://docs.vllm.ai/en/latest/api/vllm/config.html#vllm.config.PrefixCachingHashAlgo) for the prompt cache
and to [use cache salting](https://docs.vllm.ai/en/stable/design/v1/prefix_caching.html) to prevent side-channel attacks. We recommend to use cache salting in any multi-tenant setup independent of confidential computing to reduce the risk of prompt leakage.

And if you want a ready-to-use service that combines confidential computing and prompt caching in a secure way, then you should try [Privatemode](https://www.privatemode.ai/).
