# AI-Powered Inpatient Medical Coding System - Problem Definition & Architecture Design

## Objective

Design a production-grade AI system capable of accurately performing **Inpatient Medical Coding** from clinical documentation.

The primary objective is to generate:

- Principal Diagnosis (PDX)
- Secondary Diagnoses (SDX)
- (Future) ICD-10-PCS Procedure Codes
- DRG-aware sequencing

The system should produce coding decisions that are as close as possible to an experienced inpatient medical coder while minimizing hallucinations and maximizing deterministic validation.

---

# Current Problem

Medical coding is **not** simply an ICD lookup problem.

It involves multiple reasoning tasks:

1. Understanding the complete clinical documentation.
2. Identifying clinically significant diagnoses.
3. Distinguishing confirmed vs ruled-out vs historical conditions.
4. Selecting the correct ICD-10-CM code.
5. Selecting the **most specific billable code**.
6. Applying official ICD coding rules.
7. Applying Includes notes.
8. Applying Excludes1 edits.
9. Applying Excludes2 edits.
10. Applying Code First / Use Additional Code rules.
11. Applying sequencing rules.
12. Understanding CC/MCC impact.
13. Calculating DRG.
14. Choosing the optimal PDX/SDX sequence.

Trying to solve all of these in a single LLM prompt significantly increases the chance of:

- Hallucinated ICD codes
- Incorrect specificity
- Missed coding rules
- Context loss
- Inconsistent sequencing
- Higher token usage

---

# Existing Resources

## 1. ICD Validation Package

We already have an ICD package capable of:

- Validate ICD code
- Return code description
- Return parent code
- Return all child codes
- Includes notes
- Excludes1 notes
- Excludes2 notes
- Billable status
- Additional ICD metadata

This package acts as the deterministic source of truth for ICD-10-CM.

---

## 2. DRG Package

Available capabilities include:

- DRG calculation
- Relative weight
- CC/MCC detection
- Sequencing-related information
- Other DRG metadata

This package acts as the deterministic source of truth for DRG assignment.

---

# Current Challenges

## Challenge 1 — Wrong ICD Generation

The LLM may correctly understand the diagnosis but generate:

- Wrong ICD
- Non-billable ICD
- Parent code instead of child code
- Less specific code

Example:

Clinical diagnosis:

Acute systolic heart failure

LLM outputs:

I50.9

Instead of:

I50.21

Although the ICD package can provide parent and child relationships, the LLM currently has no structured way to use that information during reasoning.

---

## Challenge 2 — Missing Specificity

Clinical notes often imply more specific coding than what the LLM initially generates.

The validation package already exposes:

- Parent
- Children
- Descriptions

These can be used to guide the LLM toward the most specific valid code instead of accepting the first generated code.

---

## Challenge 3 — Includes / Excludes Rules

The LLM may generate clinically correct codes that violate ICD coding rules.

Examples include:

- Excludes1 conflicts
- Excludes2 situations
- Missing Includes diagnoses
- Required additional codes

These are deterministic rules and should not rely solely on the LLM's memory.

---

## Challenge 4 — Sequencing

PDX selection is influenced by:

- Clinical documentation
- Official coding guidelines
- CC/MCC
- DRG impact
- Relative weight
- Code interactions

Sequencing is therefore a combination of:

Clinical reasoning

+

Deterministic coding rules

The architecture must support both.

---

## Challenge 5 — Token Efficiency

Providing the LLM with:

- Parent
- Children
- Includes
- Excludes
- Descriptions
- DRG
- Coding rules

for every generated ICD can quickly consume context.

Therefore the architecture should:

- Minimize repeated information
- Avoid unnecessary package outputs
- Preserve only reasoning-relevant information

---

## Challenge 6 — Context Loss

A single prompt asking the LLM to simultaneously:

- Understand medicine
- Generate ICDs
- Validate ICDs
- Apply Includes
- Apply Excludes
- Calculate DRG
- Perform sequencing

creates a very large reasoning space.

Even state-of-the-art reasoning models can become inconsistent when required to solve all of these tasks together.

---

# Available LLMs

The system can use state-of-the-art reasoning models such as:

- GPT-5
- GPT-5.4 (if available)
- Claude Opus
- Other frontier reasoning models

Accuracy is significantly more important than latency or token cost.

Multiple LLM calls are acceptable if they improve coding quality.

---

# Design Philosophy

The LLM should perform:

## Semantic reasoning

Examples:

- Read clinical documentation.
- Understand diseases.
- Infer diagnoses.
- Interpret physician intent.
- Resolve ambiguous documentation.

The Python engine should perform:

## Deterministic reasoning

Examples:

- ICD validation
- ICD hierarchy lookup
- Includes rules
- Excludes rules
- DRG calculation
- CC/MCC
- Rule validation

The LLM should never memorize deterministic coding rules when they can be computed programmatically.

---

# Proposed Architecture

Instead of asking one LLM to solve every problem simultaneously, divide the workflow into specialized reasoning stages.

```
Clinical Note
      │
      ▼
──────────────────────────────
Stage 1
Clinical Understanding
──────────────────────────────

↓

Structured Diagnoses

↓

──────────────────────────────
Stage 2
ICD Candidate Generation
──────────────────────────────

↓

Candidate ICD Codes

↓

──────────────────────────────
Python Validation Engine
──────────────────────────────

Validate Codes

↓

Parent / Children

↓

Billable Status

↓

Includes

↓

Excludes1

↓

Excludes2

↓

Code First

↓

Use Additional Code

↓

CC / MCC

↓

DRG

↓

Validation Report

↓

──────────────────────────────
Stage 3
Coding Review
──────────────────────────────

LLM reviews validation report

↓

Resolve conflicts

↓

Choose best codes

↓

──────────────────────────────
Stage 4
Final Sequencing
──────────────────────────────

PDX

↓

SDX

↓

Final Output
```

---

# Why This Architecture?

Each reasoning stage has exactly one responsibility.

Rather than asking:

> "Generate ICDs, validate them, apply Includes, Excludes, DRG, sequencing..."

we instead ask:

Stage 1:

> "Understand the patient."

Stage 2:

> "Generate candidate ICDs."

Stage 3:

> "Review deterministic validation."

Stage 4:

> "Perform final sequencing."

This dramatically reduces cognitive load on the LLM.

---

# Advantages

## Lower hallucination rate

The LLM reasons over validated data instead of relying solely on memory.

---

## Deterministic rule enforcement

Official coding rules remain in Python rather than in prompts.

---

## Better maintainability

Updating ICD versions or DRG logic only requires updating the validation engine.

No prompt modifications are necessary.

---

## Better explainability

Each stage has a clear purpose.

Failures become easier to diagnose.

---

## Reduced context overload

Instead of exposing the raw output of every package function, the validation engine can generate concise, structured summaries containing only information relevant to subsequent reasoning.

---

# Future Improvement

One important missing capability is:

Diagnosis → ICD Candidate Search

Currently the LLM must recall ICD codes from its own knowledge.

Instead, a future enhancement could introduce a retrieval system:

Clinical Diagnosis

↓

Vector Search / RAG

↓

Top ICD Candidates

↓

LLM Selection

↓

Validation Engine

↓

Final Review

This would reduce hallucinations while improving specificity.

---

# Open Questions

## 1. Should sequencing optimization prioritize:

- Official ICD coding guidelines?
- Maximum DRG reimbursement?
- Or a balanced combination?

---

## 2. How should conflicts be resolved?

Example:

Clinical interpretation suggests one sequence.

DRG optimization suggests another.

Which should take precedence?

---

## 3. Should the validation engine automatically suggest more specific child codes before involving the LLM?

---

## 4. How should uncertain diagnoses be represented?

For example:

- Possible
- Probable
- Suspected
- Ruled Out
- History Of

Should these be normalized into structured categories before ICD generation?

---

## 5. Should diagnosis-to-ICD retrieval (RAG) be introduced before ICD generation to reduce hallucinations and improve specificity?

---

# Ultimate Goal

Build an AI-assisted inpatient medical coding system where:

- The LLM performs clinical reasoning.
- Python performs deterministic coding validation.
- The LLM performs informed coding decisions using validated evidence.
- Final PDX/SDX sequencing follows official inpatient coding guidelines while incorporating DRG, CC/MCC, Includes, Excludes, and ICD specificity.

The architecture should maximize accuracy, minimize hallucinations, remain maintainable across future ICD updates, and optimize token usage by separating semantic reasoning from deterministic validation.
