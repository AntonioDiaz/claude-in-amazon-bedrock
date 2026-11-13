# Module 3 — Prompt engineering

## Contents

- [Module 3 — Prompt engineering](#module-3--prompt-engineering)
  - [Contents](#contents)
  - [1. Prompt engineering](#1-prompt-engineering)
    - [The improvement loop](#the-improvement-loop)
    - [The running example](#the-running-example)
    - [Eval pipeline: `PromptEvaluator`](#eval-pipeline-promptevaluator)
    - [Generating the dataset](#generating-the-dataset)
    - [Baseline prompt (v1)](#baseline-prompt-v1)
    - [Running the evaluation](#running-the-evaluation)
    - [Reading the results](#reading-the-results)
  - [2. Being clear and direct](#2-being-clear-and-direct)
    - [Clear](#clear)
    - [Direct](#direct)
    - [Applied to the meal plan prompt](#applied-to-the-meal-plan-prompt)
    - [Result](#result)
  - [3. Being specific](#3-being-specific)
  - [4. Structure with XML tags](#4-structure-with-xml-tags)
  - [5. Providing examples](#5-providing-examples)
  - [6. Exercise on prompting](#6-exercise-on-prompting)
  - [7. Quiz on prompt engineering](#7-quiz-on-prompt-engineering)
  - [Common pitfalls](#common-pitfalls)

---

## 1. Prompt engineering

Prompt engineering = taking a prompt you wrote and improving it **step by step** to get more
reliable, higher-quality output, measuring every change.

### The improvement loop

```mermaid
flowchart LR
    A[Set a goal] --> B[Write initial prompt] --> C[Evaluate] --> D[Apply a technique] --> E[Re-evaluate]
    E -- better? keep iterating --> D
```

### The running example

A prompt that generates a **one-day meal plan for an athlete**, based on:

- Height
- Weight
- Physical goal
- Dietary restrictions

### Eval pipeline: `PromptEvaluator`

An improved version of the [Module 2](module-2-prompt-evaluations.md) pipeline, wrapped in a class
that handles **dataset generation** and **model grading**, and runs cases **concurrently**:

```python
evaluator = PromptEvaluator(max_concurrent_tasks=5)
```

> Start low (e.g. 3) to avoid **rate limit errors**, then raise it according to your quota.

### Generating the dataset

`generate_dataset` needs a task description, a spec of the prompt's inputs and how many cases:

```python
dataset = evaluator.generate_dataset(
    task_description="Write a compact, concise 1 day meal plan for a single athlete",
    prompt_inputs_spec={
        "height": "Athlete's height in cm",
        "weight": "Athlete's weight in kg",
        "goal": "Goal of the athlete",
        "restrictions": "Dietary restrictions of the athlete",
    },
    num_cases=3,
)
```

### Baseline prompt (v1)

Deliberately naive. `run_prompt` receives the test case inputs and returns Claude's answer:

```python
def run_prompt(prompt_inputs):
    prompt = f"""
What should this person eat?

- Height: {prompt_inputs["height"]}
- Weight: {prompt_inputs["weight"]}
- Goal: {prompt_inputs["goal"]}
- Dietary restrictions: {prompt_inputs["restrictions"]}
"""
    messages = []
    add_user_message(messages, prompt)
    return chat(messages)
```

### Running the evaluation

`extra_criteria` tells the grader what a good answer must contain:

```python
results = evaluator.run_evaluation(
    run_prompt_function=run_prompt,
    dataset_file="dataset.json",
    extra_criteria="""
The output should include:
- Daily caloric total
- Macronutrient breakdown
- Meals with exact foods, portions, and timing
""",
)
```

### Reading the results

The run writes an **`output.html`** report: open it in the browser to see, per test case, the score,
the grader's reasoning and the prompt's output.

| Version | Score |
|---|---|
| v1 · naive prompt | **2.3 / 10** |

A low first score is normal: it's the **baseline**. The next lessons apply one technique at a time
(clear and direct, specific, XML structure, examples) and re-run the eval to see if the score goes up.

📓 [001_prompting.ipynb](exercises/module_3/001_prompting.ipynb) · 📓 [002_prompting_completed.ipynb](exercises/module_3/002_prompting_completed.ipynb)

## 2. Being clear and direct

The **first line** of the prompt is the most important part: it sets up everything that follows.
Make it clear and direct.

### Clear

- Simple language anyone can understand.
- Say exactly what you want, without going around it.
- Open with a plain statement of Claude's task.

| ❌ Vague | ✅ Clear |
|---|---|
| *"I need to know about those things people put on their roofs that use sun — those solar panel things, I think they're called"* | *"Write three paragraphs about how solar panels work."* |

### Direct

- Use **instructions, not questions**.
- Start with an **action verb**: *Write*, *Create*, *Generate*, *Identify*…

| ❌ Question | ✅ Instruction |
|---|---|
| *"I was reading about renewable energy and geothermal energy sounds neat. What countries use it?"* | *"Identify three countries that use geothermal energy. Include generation stats for each."* |

### Applied to the meal plan prompt

```diff
- What should this person eat?
+ Generate a one-day meal plan for an athlete that meets their dietary restrictions.
```

The new first line tells Claude:

| | |
|---|---|
| **Action** | Generate |
| **What** | A meal plan |
| **Constraints** | One day · for an athlete · respecting dietary restrictions |

### Result

| Version | Change | Score |
|---|---|---|
| v1 | Naive prompt | 2.32 |
| v2 | Clear and direct first line | **3.92** |

> Treat Claude like a capable assistant who needs clear direction, not someone who has to guess.
> Lead with an action verb and the task.

## 3. Being specific

_Pending._

## 4. Structure with XML tags

_Pending._

## 5. Providing examples

_Pending._

## 6. Exercise on prompting

_Pending._

## 7. Quiz on prompt engineering

_Pending._

---

## Common pitfalls

- Vague first line → the model guesses the task.
- Mixing data and instructions without delimiters → the model may treat data as instructions.
- A single example that is too specific → the model copies it literally; use varied examples.
- A prompt instruction is not a security control → enforce content policies with **Bedrock Guardrails**.

---

[← Back to README](README.md)
