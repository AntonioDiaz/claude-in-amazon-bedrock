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
    - [Two types of guidelines](#two-types-of-guidelines)
    - [Applied to the meal plan prompt (quality guidelines)](#applied-to-the-meal-plan-prompt-quality-guidelines)
    - [Result](#result-1)
    - [When to add process steps](#when-to-add-process-steps)
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

Don't leave everything to the model's interpretation: give **guidelines or steps** that point Claude
to the output you want.

*"Write a short story about a character who discovers a hidden talent"* could become 200 or 2,000
words, one character or five, any structure. Specific guidelines narrow that down.

### Two types of guidelines

Often used together in real applications:

| | Quality guidelines | Process steps |
|---|---|---|
| Controls | **What** the output looks like | **How** Claude gets to it |
| Examples (short story) | • Keep under 1,000 words<br>• Include a clear action that reveals the talent<br>• At least one supporting character | 1. Brainstorm 3 talents that create dramatic tension<br>2. Pick the most interesting<br>3. Outline a pivotal scene that reveals it<br>4. Brainstorm 3 supporting character types that increase the impact |

### Applied to the meal plan prompt (quality guidelines)

```text
Generate a one-day meal plan for an athlete that meets their dietary restrictions.

- Height: {prompt_inputs["height"]}
- Weight: {prompt_inputs["weight"]}
- Goal: {prompt_inputs["goal"]}
- Dietary restrictions: {prompt_inputs["restrictions"]}

Guidelines:
1. Include accurate daily calorie amount
2. Show protein, fat, and carb amounts
3. Specify when to eat each meal
4. Use only foods that fit restrictions
5. List all portion sizes in grams
6. Keep budget-friendly if mentioned
```

### Result

| Version | Change | Score |
|---|---|---|
| v1 | Naive prompt | 2.32 |
| v2 | Clear and direct first line | 3.92 |
| v3a | + quality guidelines | **7.86** |
| v3b | + process steps (calories → macros → meal timing…) | 7.3 |

Quality guidelines more than doubled the score.

### When to add process steps

Quality guidelines are enough for most prompts. Add steps for:

- Troubleshooting hard problems
- Decision making
- Critical thinking
- Any time you want Claude to take a **wider view**

Example: *"Why did the sales team's numbers drop 30% last quarter?"* → give steps that force it to look
at market conditions, individual performance, organizational changes and customer feedback, instead of
jumping to the first obvious explanation.

> Being specific turns results from luck into something consistent: guidelines, steps or both give
> Claude a clear frame to work in.

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
