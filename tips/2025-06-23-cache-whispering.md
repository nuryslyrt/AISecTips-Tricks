# 🤖🔒 AI-Sec Tip — 2025-06-25

## 🔍 Cache Whispering: Exploiting Hardware Cache Side-Channels in LLM Inference

### 🎯 Background

Side-channel attacks exploiting hardware characteristics have long haunted cryptographic systems, but their entrance into Large Language Model (LLM) ecosystems has introduced unique threats. 
Recent research demonstrates how attackers can exploit CPU cache-access patterns during local inference to reconstruct highly sensitive details about user inputs and model outputs.

### ⚙️ Attack Methodology

The "Prime+Probe" attack involves two primary phases:

1. **Prime**: The attacker populates the cache with known data by accessing specific memory locations, causing known cache lines to be loaded.
2. **Probe**: After the victim’s inference task runs, the attacker measures the time to access those same cache lines again.
3. **Cache lines accessed by the victim (for token embeddings)** will exhibit significantly different latencies due to cache misses or hits, thereby leaking token IDs and their sequence positions.

This timing difference (as subtle as a few nanoseconds) allows attackers to reconstruct tokens accurately.

![img](../assets/2025-06-23-cache-whispering.png)


### 🔧 Simplified Proof-of-Concept (PoC)

Here's an simplified snippet for cache-based timing measurements:

```c
// Detailed Prime+Probe PoC
#include <stdint.h>
#include <x86intrin.h>  // for rdtsc()

#define EMBEDDING_BASE 0x400000
#define TOKEN_STRIDE 256

uint64_t prime_probe(int token_index) {
    volatile char *target = (volatile char *)(EMBEDDING_BASE + token_index * TOKEN_STRIDE);
    uint64_t t0, t1;

    // Prime phase
    _mm_clflush((void *)target);  // Evict cache line explicitly

    // Let victim process run (e.g., inference happens here externally)

    // Probe phase
    t0 = __rdtscp(&token_index);
    volatile char dummy = *target;
    t1 = __rdtscp(&token_index);

    return (t1 - t0); // return timing difference
}
```

Repeated runs and statistical analysis reveal distinct access patterns and token embeddings reliably.

### 🚨 Real-world Impact

* Tests on popular frameworks (`llama.cpp`,local forks of an LLM, etc.) have shown recovery of tokens with up to **98.7% cosine similarity**.
* No special OS or hardware privileges required, making this a stealthy and potent exploit vector.

### 🛡️ Robust Mitigation Measures

| Mitigation Strategy              | Explanation                                                         | Overhead        |
| -------------------------------- | ------------------------------------------------------------------- | --------------- |
| **Embedding Table Shuffling**    | Randomizing memory locations per inference to reduce correlation.   | Moderate (\~5%) |
| **Dummy Accesses (Noise)**       | Insert random embedding accesses to obfuscate true cache patterns.  | Low (\~1–2%)    |
| **Dedicated VM Isolation**       | Run inference workloads on isolated virtual machines or containers. | Moderate (\~7%) |
| **High-Resolution Timer Audits** | Detecting suspicious use of high-resolution timing APIs.            | Negligible      |

### 📌 Lab-Ready Checklist

For your home experiments:

* [ ] Profile your model to observe timing variances per embedding access.
* [ ] Implement cache-shuffling algorithms within your memory allocator.
* [ ] Insert random dummy embedding lookups at inference time to frustrate attackers.
* [ ] Deploy monitoring to alert on high-resolution timing queries.

### 🤖 Bottom Line

Cache whispers can loudly reveal your model’s secrets—ensure your caches don't spill confidential tokens by taking proactive defensive measures.


**The resources that I sued to create this tip:**

[1] ["I Know What You Said: Unveiling Hardware Cache Side‑Channels in Local Large Language Model Inference"](https://arxiv.org/abs/2505.06738)

[2] ["The Early Bird Catches the Leak: Unveiling Timing Side Channels in LLM Serving Systems"](https://arxiv.org/abs/2409.20002)
