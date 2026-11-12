# Module 0 — Course introduction & Claude models

**Lessons:** introduction to the course · overview of Claude models

## Claude model families

| Family | Profile | Typical use |
|---|---|---|
| Haiku | Fastest, cheapest | Classification, extraction, high-volume / real-time, sub-agents |
| Sonnet | Balance of capability, speed and cost | Default choice: coding, RAG, tool use, most apps |
| Opus | Most capable, slowest, most expensive | Complex reasoning, long agentic tasks, hard analysis |

- Choosing a model is a trade-off between **quality, latency and cost**. Start with the smallest model that passes your evals (see Module 2).

## On Amazon Bedrock

- Model access must be **enabled per account and Region** in the Bedrock console before the first call.
- Each model has a **model ID**. Recent Claude models are called through an **inference profile** ID with a `us.`, `eu.` or `global.` prefix, which routes requests across Regions.

---

[← Back to README](README.md)
