# Python Coding Agent Specification – LLM-Based Inpatient Medical Coding Pipeline

## Objective

Implement a modular and production-ready pipeline for automated inpatient (IP) medical coding using LLMs, Retrieval-Augmented Generation (RAG), and deterministic validation.

The pipeline should support:

* ICD-10-CM diagnosis coding
* ICD-10-PCS procedure coding
* DRG optimization
* Code validation and fallback recovery

The implementation should be modular so that each stage can be independently tested, replaced, or improved.

---

# Overall Workflow

The complete workflow consists of the following stages:

```
Clinical Documents
        │
        ▼
──────────────────────────────
LLM Call #1
Clinical Understanding
──────────────────────────────
        │
        ├──────────────► Diagnosis List
        │
        └──────────────► Procedure List
                │
                ▼
        Parallel Processing
        │                 │
        ▼                 ▼
 ICD-CM Pipeline      ICD-PCS Pipeline
        │                 │
        ▼                 ▼
Validated ICD-CM     Validated ICD-PCS
        │
        ▼
DRG Grouper
        │
        ▼
Highest Revenue DRG
```

---

# Stage 1 – Clinical Understanding

## Purpose

Make a single LLM call that reads all inpatient clinical documentation and generates a structured understanding of the patient's hospitalization.

The model should **not generate ICD codes** in this step.

Its responsibility is only to understand the medical record.

---

## Expected Output

Return structured JSON containing:

* Principal diagnosis candidates
* Secondary diagnoses
* Additional diagnoses
* Procedures performed
* Hospital course
* Treatment timeline
* Present on Admission (POA) indicators
* Supporting clinical evidence
* Laboratory findings
* Imaging findings
* Operative findings
* Discharge diagnoses

Each diagnosis and procedure should include supporting evidence extracted from the documentation.

---

# Stage 2 – ICD-10-CM Coding

## Objective

Convert the extracted diagnoses into finalized ICD-10-CM diagnosis codes.

---

## Context Construction

For each diagnosis returned from Stage 1:

Construct a rich coding context using Retrieval-Augmented Generation (RAG).

Retrieve:

* Top 5–10 semantic vector search results
* Sibling ICD codes for every retrieved candidate
* Inclusion notes
* Exclusion notes
* Code first / Use additional code rules
* Additional CMS coding guidelines
* Relevant coding instructions

The objective is to provide the LLM with all candidate codes and official coding guidance required to select the most appropriate ICD-10-CM code.

---

## Example Input

```json
{
  "clinical_context": {
    "target_condition": "Acute cholecystitis with cholelithiasis",
    "POA": "Y",
    "supporting_evidence": [
      "...",
      "...",
      "..."
    ]
  },

  "candidate_codes": [
    {
      "code": "K80.10",
      "description": "...",
      "severity_category": "CC"
    },
    {
      "code": "K80.11",
      "description": "...",
      "severity_category": "MCC"
    }
  ],

  "coding_guidelines": {
    "include_notes": [],
    "exclude1": [],
    "exclude2": [],
    "additional_guidelines": [],
    "notes": []
  }
}
```

---

## Expected Output

```json
{
  "codes": [
    {
      "Final_ICD_CM_code": "",
      "Type": "PDX | SDX | ADX",
      "Description": "",
      "Severity_Category": "",
      "POA": "",
      "Supporting_Evidence": [],
      "Justification": ""
    }
  ]
}
```

Each generated code must include:

* Final ICD-10-CM code
* Code description
* Diagnosis type (PDX / SDX / ADX)
* Severity category
* POA indicator
* Supporting evidence
* Detailed reasoning

---

# Stage 3 – ICD-10-CM Validation

Every generated ICD-10-CM code must be validated before being accepted.

Validation should verify:

* Code exists
* Code is active
* Code is billable
* Code format is valid
* Code belongs to the retrieved candidate set (when applicable)

If validation fails:

* Trigger a fallback LLM call.
* Provide the original diagnosis, retrieved context, and validation error.
* Request a corrected valid ICD-10-CM code.
* Repeat validation before accepting the corrected code.

No invalid code should be returned from the pipeline.

---

# Stage 4 – ICD-10-PCS Coding

Procedure coding should execute in parallel with the ICD-10-CM pipeline after Stage 1.

---

## Workflow

For every extracted procedure:

1. Build procedure-specific RAG context.
2. Retrieve:

   * Top semantic matches
   * Sibling PCS codes
   * Root operation definitions
   * Body system
   * Body part
   * Approach
   * Device
   * Qualifier
   * Official PCS guidelines
3. Call the PCS coding LLM.
4. Validate the generated PCS code.
5. Retry using a fallback LLM if validation fails.

---

# Stage 5 – DRG Optimization

Once both ICD-10-CM and ICD-10-PCS codes have been finalized:

Pass the complete coding set into the DRG Grouper.

The DRG stage should:

* Determine the appropriate DRG
* Evaluate all valid DRG possibilities
* Select the highest valid reimbursement DRG according to coding rules
* Return the selected DRG along with justification

---

# Final Output

The pipeline should return:

```json
{
  "diagnosis_summary": {},

  "ICD_CM": [],

  "ICD_PCS": [],

  "DRG": {
    "code": "",
    "description": "",
    "expected_revenue": "",
    "justification": ""
  }
}
```

---

# Expected Performance

The current ICD-10-CM pipeline achieves approximately **90% coding accuracy**.

The implementation should preserve or improve this performance while maintaining a modular architecture that supports future enhancements.

---

# Implementation Requirements

The Python implementation should be organized into reusable modules with clear separation of responsibilities.

Suggested modules include:

* Document ingestion
* Clinical understanding (LLM Call #1)
* Diagnosis extraction
* Procedure extraction
* Timeline generation
* RAG context builder
* ICD-10-CM coding
* ICD-10-CM validation
* ICD-10-PCS coding
* ICD-10-PCS validation
* DRG grouping
* Fallback/retry manager
* Prompt management
* LLM client
* Logging and tracing
* Configuration management

Each module should expose well-defined interfaces and be independently testable.

The project should follow clean architecture principles, use type hints throughout, define request/response models with Pydantic, implement structured logging, support configuration via environment variables, and include comprehensive error handling and retry mechanisms.
