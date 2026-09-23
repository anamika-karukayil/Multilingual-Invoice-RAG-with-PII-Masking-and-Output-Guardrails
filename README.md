# Multilingual Invoice RAG with PII Masking and Output Guardrails

## Overview

This project implements a privacy-aware multilingual Retrieval-Augmented Generation (RAG) pipeline for invoice documents.

The project focuses on protecting Personally Identifiable Information (PII) at two levels:

1. **Pre-LLM PII Masking** – detects and masks PII before invoice content is sent to the LLM.
2. **Output Guardrail** – checks the generated response and blocks the response if PII is detected.

The project also includes a quantitative evaluation of the PII masking performance and compares the original masking approach with an improved location-masking approach.

---

## Problem Statement

In a standard RAG pipeline, invoice documents are:

`Invoice PDF → Text Extraction → Chunking → Embeddings → Vector Database → Retrieval → LLM`

The main privacy concern is that retrieved invoice context may contain original PII such as:

- Person names
- Locations / addresses
- Postal codes

If this original PII is directly passed to the LLM, the sensitive information has already reached the model before an output guardrail can act.

Therefore, an output-only guardrail provides protection at the response level but does not prevent PII from entering the LLM context.

---

## Project Approach

The project was developed in three stages:

### Stage 1 – Baseline Invoice RAG with Output Guardrail

The baseline pipeline retrieves the original invoice context and sends it to the LLM.

### Baseline Architecture

`Invoice PDFs → Text Extraction → Chunking → Embeddings → ChromaDB → Retrieval → Original Invoice Context → LLM → PII Output Guardrail → Safe / Block`

In this stage, the output guardrail checks the generated response for PII.

### Limitation

Although the output guardrail can block a response containing PII, the original PII is still present in the retrieved context provided to the LLM.

This creates a privacy limitation because protection happens only after the LLM receives the sensitive information.

---

# Improved Privacy-Aware RAG Pipeline

To address this limitation, PII masking was introduced before the LLM stage.

### Improved Architecture

`Raw Invoice PDFs → Text Extraction → PII Detection → PII Masking → Masked Invoice Text → Chunking → Embeddings → ChromaDB → Retrieval → PII-Free Context → LLM → Output Guardrail → Safe / Block`

This creates a defense-in-depth approach:

- **First layer:** PII is detected and masked before reaching the LLM.
- **Second layer:** The generated output is checked by an output guardrail.

Therefore, the system does not rely only on output filtering.

---

## PII Types

The project focuses on the following PII types:

| PII Type | Description |
|---|---|
| `PERSON` | Person / customer names |
| `LOCATION` | Locations and address-related information |
| `POSTAL_CODE` | Postal / ZIP codes |

Microsoft Presidio is used for PERSON and LOCATION detection.

Postal codes are additionally detected using a regular expression, particularly within the `Ship To` section of invoices.

---

## PII Masking

Detected PII is replaced with placeholders before the invoice text is used for RAG processing.

Examples:

| Original Information | Masked Information |
|---|---|
| Person name | `<PERSON>` |
| Location / Address | `<ADDRESS>` |
| Postal code | `<POSTAL_CODE>` |

The masked invoice text is then used for chunking, embedding, storage, retrieval, and LLM generation.

---

## RAG Pipeline

The RAG workflow consists of:

1. Invoice PDF ingestion
2. Text extraction
3. PII detection
4. PII masking
5. Text chunking
6. Embedding generation
7. Storage in ChromaDB
8. Query-based retrieval
9. Retrieval of PII-free context
10. Answer generation using the LLM
11. Output PII guardrail validation

---

## Multilingual Testing

The pipeline was tested using invoice-related queries in multiple languages:

- English
- Hindi
- Marwari
- Malayalam

The testing included both normal invoice-related questions and attempts to extract PII from invoice content.

---

# Dataset

The project uses invoice documents containing information such as:

- Customer names
- Shipping locations
- Postal codes
- Invoice numbers
- Invoice dates
- Product information
- Invoice amounts
- Bill To information
- Ship To information

The dataset contains **10 invoice documents** used for the PII evaluation.

The dataset source and related information are documented separately in:

`data/README.md`

The dataset source repository is:

`TGS-2025059028 – Build and Deploy Agentic AI Apps with CrewAI, Autogen, ADK and Streamlit`

Source: https://github.com/tertiarycourses/TGS-2025059028-Build-and-Deploy-Agentic-AI-Apps-with-CrewAI-Autogen-ADK-and-Streamlit
---

# PII Masking Evaluation

A separate evaluation was performed to measure how effectively the system detects and protects PII.

The evaluation uses manually defined ground-truth PII annotations.

Each ground-truth entity contains:

- PII text
- PII label

Example:

`{"text": "Aaron Hawkins", "label": "PERSON"}`

An extracted PII entity is considered a match when both the entity text and label match the ground truth.

---

## Detection Evaluation Metrics

The following metrics were used:

### True Positive (TP)

PII entities correctly detected by the masking system.

### False Positive (FP)

Entities detected as PII even though they were not part of the ground-truth PII annotations.

### False Negative (FN)

Ground-truth PII entities that were not detected by the system.

### Precision

Measures how many detected PII entities were actually PII.

`Precision = TP / (TP + FP)`

### Recall

Measures how many ground-truth PII entities were successfully detected.

`Recall = TP / (TP + FN)`

### F1 Score

Provides a combined measure of precision and recall.

`F1 = 2 × (Precision × Recall) / (Precision + Recall)`

---

# Detection Results

Evaluation was performed on:

- **10 invoice documents**
- **39 ground-truth PII entities**

Results:

| Metric | Result |
|---|---:|
| True Positives (TP) | 35 |
| False Positives (FP) | 7 |
| False Negatives (FN) | 4 |
| Precision | 83.33% |
| Recall | 89.74% |
| F1 Score | 86.42% |

These metrics evaluate the ability of the masking system to detect the annotated PII entities.

---

# PII Protection Evaluation

Detection performance alone does not show whether the original PII was actually removed from the invoice context.

Therefore, a separate protection-level evaluation was performed.

The following metrics were used:

### Total PII

Total number of ground-truth PII entities.

### Protected PII

Original PII entities that were successfully removed or masked.

### Leaked PII

Original PII entities that remained unmasked.

### Protection Rate

`Protection Rate = Protected PII / Total PII × 100`

### Leakage Rate

`Leakage Rate = Leaked PII / Total PII × 100`

---

# Original Masking Results

For the original masking approach:

| Metric | Result |
|---|---:|
| Total PII | 39 |
| Protected PII | 38 |
| Leaked PII | 1 |
| Protection Rate | 97.44% |
| Leakage Rate | 2.56% |

One PII entity remained unmasked.

This leakage was related to location information in the `Ship To` section of an invoice.

---

# Location Masking Improvement

During evaluation, a limitation was identified in LOCATION masking for a particular invoice.

To address this, a custom location-masking approach was introduced for the `Ship To` section.

The improved approach:

1. Extracts the `Ship To` section.
2. Separates the location lines.
3. Ignores postal codes.
4. Ignores lines containing numbers.
5. Detects the remaining location information.
6. Replaces the detected location with `<ADDRESS>`.

This improves protection of address/location information that may not be consistently detected by the general PII recognizer.

---

# Improved Masking Results

After applying the improved location masking:

| Metric | Original Masking | Improved Masking |
|---|---:|---:|
| Total PII | 39 | 39 |
| Protected PII | 38 | 39 |
| Leaked PII | 1 | 0 |
| Protection Rate | 97.44% | 100.00% |
| Leakage Rate | 2.56% | 0.00% |

The improved approach protected all 39 ground-truth PII entities in the evaluated invoice dataset.

---

# Technology Stack

## Programming Language

- Python

## RAG & Retrieval

- LangChain Text Splitters
- ChromaDB
- Sentence Transformers
- BAAI/bge-base-en-v1.5

## PII Detection & Masking

- Microsoft Presidio Analyzer
- Regular Expressions
- Custom location masking

## LLM & Guardrails

- OpenRouter
- `deepseek/deepseek-v4-flash`
- OpenAI Agents SDK
- `google/gemma-3-12b-it`
- Pydantic structured output
- Output Guardrail with tripwire-based blocking

## Document Processing

- PyPDF

---

# Project Workflow

The complete privacy-aware workflow can be summarized as:

`Invoice PDFs`

↓

`Text Extraction`

↓

`PII Detection`

↓

`PII Masking`

↓

`Masked Invoice Text`

↓

`Chunking`

↓

`Embeddings`

↓

`ChromaDB`

↓

`Retrieval`

↓

`PII-Free Context`

↓

`LLM`

↓

`Output Guardrail`

↓

`Safe Response / Blocked Response`

↓

`PII Protection Evaluation`

---

# Project Notebooks

The project was developed through three main notebooks.

### 1. Baseline Invoice RAG + Output Guardrail

`01_Baseline_Invoice_RAG_Output_Guardrail.ipynb`

Implements the baseline invoice RAG workflow where the original invoice context is retrieved and passed to the LLM, followed by an output-level PII guardrail.

### 2. Invoice RAG + PII Masking + Output Guardrail

`02_Invoice_RAG_PII_Masking_Output_Guardrail.ipynb`

Implements the improved privacy-aware pipeline where PII is detected and masked before the invoice context reaches the LLM, followed by an output guardrail.

### 3. PII Masking Evaluation

`03_PII_Masking_Evaluation.ipynb`

Evaluates the PII masking system using ground-truth annotations and calculates:

- TP
- FP
- FN
- Precision
- Recall
- F1 Score
- Total PII
- Protected PII
- Leaked PII
- Protection Rate
- Leakage Rate

It also compares the original masking approach with the improved location-masking approach.

---

# Key Findings

The project demonstrates that an output guardrail alone does not prevent sensitive information from reaching the LLM context.

Introducing PII masking before the LLM provides an additional privacy layer.

The evaluation also showed that:

- The original masking approach achieved a **97.44% protection rate**.
- One PII entity was leaked.
- The leaked entity was related to location information.
- A custom `Ship To` location-masking approach was introduced.
- The improved masking achieved **100.00% protection** on the evaluated 39 ground-truth PII entities.
- The output guardrail provides an additional response-level protection layer.

---

# Project Objective

The objective of this project is to combine:

**Pre-LLM PII Masking + RAG + Output Guardrails + Quantitative PII Evaluation**

to build a multilingual invoice RAG workflow with stronger privacy protection.

The project evaluates not only whether PII can be detected, but also whether the original PII is actually protected before reaching the LLM.

---

# Repository Structure

The currently maintained project structure is:

`RAG_Guadrail_POC/`

`├── data/`

`│   └── invoice dataset`

`│`

`└── README.md`

The dataset source information is maintained inside:

`data/README.md`

---

# Security Note

The invoice documents used in this project may contain sensitive information.

Do not upload raw PII-containing invoice documents or API keys to a public GitHub repository.

For public release, use:

- Synthetic data
- Fully redacted data
- Anonymized data

API keys and other secrets should be stored using environment variables or other secure secret-management methods.

---

# Conclusion

This project demonstrates a privacy-aware multilingual invoice RAG pipeline using two complementary protection layers:

**1. Pre-LLM PII Masking**

Sensitive information is detected and masked before invoice context is passed to the LLM.

**2. Output Guardrail**

The generated response is checked for PII and can be blocked when sensitive information is detected.

The evaluation provides quantitative measurements of PII detection and protection, while the location-masking improvement demonstrates how evaluation results can be used to identify and address specific masking limitations.

Overall, the project combines:

**PII Detection → PII Masking → RAG → LLM → Output Guardrail → Quantitative Evaluation**

to create a more privacy-aware invoice question-answering workflow.
