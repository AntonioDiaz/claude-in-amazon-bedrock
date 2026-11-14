# Module 4 — Tool use

## Contents

- [Module 4 — Tool use](#module-4--tool-use)
  - [Contents](#contents)
  - [1. Introducing tool use](#1-introducing-tool-use)
  - [2. Tool functions](#2-tool-functions)
  - [3. JSON Schema for tools](#3-json-schema-for-tools)
  - [4. Handling tool use responses](#4-handling-tool-use-responses)
  - [5. Running tool functions](#5-running-tool-functions)
  - [6. Sending tool results](#6-sending-tool-results)
  - [7. Multi-Turn conversations with tools](#7-multi-turn-conversations-with-tools)
  - [8. Adding multiple tools](#8-adding-multiple-tools)
  - [9. Batch tool use](#9-batch-tool-use)
  - [10. Structured data with tools](#10-structured-data-with-tools)
  - [11. Flexible tool extraction](#11-flexible-tool-extraction)
  - [12. The text editor tool](#12-the-text-editor-tool)
  - [Common pitfalls](#common-pitfalls)

---

## 1. Introducing tool use

Tools let Claude get information or take actions outside its training data (current time, APIs,
databases…). **Claude never runs a tool itself**: it asks for it, your code runs it and sends back the
result.

```mermaid
sequenceDiagram
    participant A as Your code
    participant C as Claude
    A->>C: messages + tool definitions
    C-->>A: stopReason = tool_use (name + input)
    A->>A: run the function
    A->>C: toolResult
    C-->>A: final answer
```

## 2. Tool functions

_Pending._

## 3. JSON Schema for tools

_Pending._

## 4. Handling tool use responses

_Pending._

## 5. Running tool functions

_Pending._

## 6. Sending tool results

_Pending._

## 7. Multi-Turn conversations with tools

_Pending._

## 8. Adding multiple tools

_Pending._

## 9. Batch tool use

_Pending._

## 10. Structured data with tools

_Pending._

## 11. Flexible tool extraction

_Pending._

## 12. The text editor tool

_Pending._

---

## Common pitfalls

- Returning only some of the `toolResult` blocks when the model asked for several → every `toolUseId` needs a result.
- Raising exceptions instead of returning `status: "error"` → the model can't recover or retry.
- Unbounded loop → add a max number of iterations.
- "Batch tool use" (several calls in one turn) ≠ **Bedrock batch inference** (async jobs over S3).
- Managed alternative on AWS: **Bedrock Agents** action groups backed by **Lambda**.

---

[← Back to README](README.md)
