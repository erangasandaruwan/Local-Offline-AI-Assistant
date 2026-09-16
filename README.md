# Local Offline AI Assistant

### Building and Benchmarking a Local LLM System

This article describes how to build a **local AI assistant that runs entirely offline using Small Language Models (SLMs)**.

Instead of sending prompts and potentially sensitive information to an external LLM API, the model runs directly on local hardware.

The objective is not simply to run an LLM locally.

The project focuses on the engineering challenges around:

**Local Inference → Performance Benchmarking → Structured Output → Validation → Retry → Model Comparison → Quantization → Evaluation**

---

# Why Run AI Models Locally?

Cloud-hosted LLMs provide access to powerful models, but there are many situations where sending information to an external service may not be desirable or possible.

Examples include:

* Privacy-sensitive applications
* Financial and banking systems
* Healthcare environments
* Government systems
* Enterprise confidential information
* Edge computing
* Environments without reliable internet connectivity
* Applications requiring very low latency
* High-volume workloads where API cost becomes significant

A local AI architecture can provide several advantages.

| Area                    | Cloud LLM                   | Local LLM            |
| ----------------------- | --------------------------- | -------------------- |
| Internet required       | Usually                     | No                   |
| Data leaves environment | Usually                     | No                   |
| API cost                | Usage based                 | No per-token API fee |
| Hardware management     | Provider                    | Application owner    |
| Model control           | Limited                     | High                 |
| Model upgrades          | Provider controlled         | Developer controlled |
| Offline operation       | No                          | Yes                  |
| Model size              | Very large models available | Hardware dependent   |

However, local inference introduces another engineering challenge:

> How do we determine whether a smaller local model is sufficiently fast, reliable and accurate for a particular application?

This project attempts to answer that question through systematic benchmarking and evaluation.

---

# Overall Architecture

The initial architecture is:

```text
User
 │
 ▼
React / Angular
 │
 ▼
ASP.NET Core Web API
 │
 ▼
FastAPI Local AI Service
 │
 ├── Prompt Management
 ├── Structured Output
 ├── Pydantic Validation
 ├── Retry Handling
 └── Benchmark Collection
 │
 ▼
Ollama
 │
 ├── Model A
 ├── Model B
 └── Model C
 │
 ▼
Local CPU / GPU
```

The entire LLM inference pipeline can operate without calling an external AI API.

---

# Key Technologies and Concepts

| Area             | Technology / Concept  | Purpose                                         |
| ---------------- | --------------------- | ----------------------------------------------- |
| Local AI Runtime | Ollama                | Run LLMs locally                                |
| AI Model         | Llama                 | Local language model                            |
| AI Model         | Mistral               | Alternative local model                         |
| API              | FastAPI               | Expose local inference through REST             |
| Validation       | Pydantic              | Validate structured LLM responses               |
| Backend          | ASP.NET Core          | Enterprise application/API integration          |
| Frontend         | React / Angular       | User interface                                  |
| Containerization | Docker / Podman       | Package application services                    |
| Observability    | OpenTelemetry         | Capture inference telemetry                     |
| Metrics          | Tokens/sec            | Measure generation throughput                   |
| Metrics          | TTFT                  | Measure time to first token                     |
| Metrics          | Total Latency         | Measure complete response time                  |
| Reliability      | JSON Schema           | Enforce predictable output structure            |
| Reliability      | Retry                 | Recover from invalid model responses            |
| Evaluation       | Golden Prompt Dataset | Compare models consistently                     |
| Optimization     | Quantization          | Reduce memory and improve inference performance |

---

# Phase 1 — Running an LLM Locally

The first objective is simple:

> Run an AI model completely locally without depending on OpenAI, Azure OpenAI or another external inference API.

Install **Ollama** and download a suitable Small Language Model.

For example:

```bash
ollama pull llama3.2:3b
```

Then run the model:

```bash
ollama run llama3.2:3b
```

The interaction becomes:

```text
User Prompt
     │
     ▼
Local Application
     │
     ▼
Ollama
     │
     ▼
Local LLM
     │
     ▼
Response
```

At this stage, everything can run on the developer's machine.

---

# Building an API Around the Model

Running a model through a terminal is useful for experimentation, but production applications need a programmatic interface.

A lightweight **FastAPI** service can therefore sit between the application and Ollama.

```text
Client
   │
   ▼
POST /api/chat
   │
   ▼
FastAPI
   │
   ▼
Ollama
   │
   ▼
Local Model
```

A request could look like:

```json
{
  "prompt": "Explain the difference between authentication and authorization."
}
```

The API returns the generated response together with inference metadata.

For example:

```json
{
  "response": "Authentication verifies identity while authorization determines access permissions.",
  "model": "llama3.2:3b",
  "metrics": {
    "total_latency_ms": 1250,
    "tokens_per_second": 28.4
  }
}
```

This starts turning the experiment into an actual AI engineering system.

---

# Phase 2 — Benchmark Local Inference

Running the model successfully is only the beginning.

The next question is:

> How well does the model perform on the available hardware?

Several measurements should be collected.

## Time to First Token — TTFT

TTFT measures how long the user waits before the model begins generating its answer.

```text
Request
   │
   ├──────────── TTFT ────────────► First Token
   │
   ▼
Remaining Tokens
```

This is particularly important for interactive AI applications.

A model might generate tokens quickly after generation starts but still have a noticeable initial delay.

---

# Tokens Per Second

Tokens per second measures generation throughput.

For example:

```text
Generated Tokens: 150
Generation Time:   5 seconds

Tokens/sec = 30
```

Higher throughput generally means a faster user experience, although speed alone does not indicate answer quality.

---

# Total Response Latency

Total latency measures the complete time between receiving the request and finishing the response.

Conceptually:

```text
Total Latency =
Model Loading
+ Prompt Processing
+ Time to First Token
+ Token Generation
+ Output Processing
```

The project should capture these measurements for every benchmark run.

---

# Memory Consumption

Local models consume system RAM and, where GPU acceleration is available, VRAM.

Therefore record:

```text
Model
Parameters
Quantization
RAM Usage
VRAM Usage
TTFT
Tokens/sec
Total Latency
```

Example:

| Model   | RAM | TTFT | Tokens/sec | Total Latency |
| ------- | --: | ---: | ---------: | ------------: |
| Model A | TBD |  TBD |        TBD |           TBD |
| Model B | TBD |  TBD |        TBD |           TBD |
| Model C | TBD |  TBD |        TBD |           TBD |

Actual benchmark values should be recorded from the same machine.

This is important because comparing benchmark results collected from different hardware would make the comparison much less meaningful.

---

# Phase 3 — Structured and Deterministic AI Output

Natural-language responses are useful for chat applications.

Enterprise systems, however, frequently need structured information.

For example, instead of:

```text
The customer has a high priority support issue and it should
be assigned to the technical support team.
```

the application may require:

```json
{
  "category": "Technical Support",
  "priority": "High",
  "requires_human_review": true
}
```

This is much easier for downstream applications to process.

---

# JSON Schema Enforcement

The model is instructed to produce output matching a predefined schema.

For example:

```json
{
  "category": "string",
  "priority": "Low | Medium | High",
  "requires_human_review": "boolean"
}
```

However, instructing an LLM to return JSON does not by itself guarantee valid application data.

The response must therefore be validated.

---

# Pydantic Validation

A Pydantic model can represent the expected contract.

```python
from pydantic import BaseModel
from typing import Literal

class ClassificationResult(BaseModel):
    category: str
    priority: Literal["Low", "Medium", "High"]
    requires_human_review: bool
```

The processing pipeline becomes:

```text
Prompt
   │
   ▼
Local LLM
   │
   ▼
JSON Response
   │
   ▼
Pydantic Validation
   │
   ├── Valid ─────► Application
   │
   └── Invalid ───► Retry
```

This pattern introduces deterministic application boundaries around probabilistic model behavior.

---

# Validation + Retry Strategy

LLMs may occasionally return:

* Invalid JSON
* Missing fields
* Incorrect data types
* Unexpected enumeration values
* Additional explanatory text
* Responses that violate the required schema

Therefore the application implements a controlled retry.

```text
LLM Response
      │
      ▼
Validate
      │
 ┌────┴─────┐
 │          │
Valid     Invalid
 │          │
 ▼          ▼
Return    Re-prompt
             │
             ▼
          Validate
             │
       ┌─────┴─────┐
       │           │
     Valid       Invalid
       │           │
       ▼           ▼
     Return    Fail Gracefully
```

The retry prompt can inform the model why the previous response failed.

For example:

```text
Your previous response did not satisfy the required JSON schema.

Validation error:
priority must be Low, Medium or High.

Return ONLY valid JSON matching the supplied schema.
```

This demonstrates an important production AI engineering pattern:

**Generation → Constraint → Validation → Recovery**

---

# Structured Output Success Rate

An additional reliability metric should be captured.

Suppose 50 standardized prompts are executed.

```text
Total Requests:              50
Valid on First Attempt:      43
Valid after Retry:            6
Failed after Retry:           1
```

Then:

```text
First-Pass Success Rate = 43 / 50 = 86%

Final Success Rate = 49 / 50 = 98%
```

This provides measurable evidence about the reliability of a model.

It also allows different local models to be compared objectively.

---

# Temperature Experiment

Language models are probabilistic systems.

One way of observing this behavior is to run identical prompts using different temperature configurations.

For example:

```text
Temperature = 0
```

versus:

```text
Temperature = 0.7
```

Run each test prompt multiple times.

Then compare:

* Response consistency
* JSON validity
* Classification consistency
* Output variation
* Response quality

For structured enterprise workflows, lower temperature may often be appropriate because predictability is more important than creativity.

For creative workloads, higher temperature may provide more diverse outputs.

The important point is not that one temperature is universally better.

The configuration should match the application's requirements.

---

# Phase 4 — Model Comparison Study

The next stage is one of the most important parts of the project.

Instead of selecting a model because it is popular, models should be compared using measurable evidence.

Select approximately three models that can realistically run on the target hardware.

```text
             Golden Prompt Dataset
                      │
           ┌──────────┼──────────┐
           ▼          ▼          ▼
       Model A     Model B     Model C
           │          │          │
           ▼          ▼          ▼
       Benchmark   Benchmark   Benchmark
           │          │          │
           └──────────┼──────────┘
                      ▼
                Compare Results
```

---

# Golden Prompt Dataset

Create a standardized evaluation dataset containing approximately **30–50 prompts**.

For example:

| ID  | Category          | Prompt                    | Expected Result    |
| --- | ----------------- | ------------------------- | ------------------ |
| 001 | Classification    | Classify support ticket   | Technical Support  |
| 002 | Extraction        | Extract invoice number    | INV-10239          |
| 003 | Summarization     | Summarize incident        | Expected facts     |
| 004 | Reasoning         | Analyze application error | Expected diagnosis |
| 005 | Structured Output | Return customer details   | Valid schema       |

Every model receives exactly the same prompts.

This makes the evaluation repeatable.

---

# What Should Be Compared?

For each model capture:

| Metric                  | Purpose                        |
| ----------------------- | ------------------------------ |
| TTFT                    | Initial responsiveness         |
| Tokens/sec              | Generation performance         |
| Total Latency           | End-to-end performance         |
| RAM                     | System memory requirement      |
| VRAM                    | GPU memory requirement         |
| First-Pass JSON Success | Structured-output reliability  |
| Final JSON Success      | Reliability after retry        |
| Output Quality          | Accuracy/usefulness            |
| Consistency             | Variance between repeated runs |

This turns model selection into an engineering decision.

---

# Model Benchmark Report

The final benchmark could look like:

| Metric                  | Model A | Model B | Model C |
| ----------------------- | ------: | ------: | ------: |
| Parameters              |     TBD |     TBD |     TBD |
| RAM                     |     TBD |     TBD |     TBD |
| VRAM                    |     TBD |     TBD |     TBD |
| TTFT                    |     TBD |     TBD |     TBD |
| Tokens/sec              |     TBD |     TBD |     TBD |
| Total Latency           |     TBD |     TBD |     TBD |
| First-Pass JSON Success |     TBD |     TBD |     TBD |
| Final JSON Success      |     TBD |     TBD |     TBD |
| Quality Score           |     TBD |     TBD |     TBD |

Do not populate this table with numbers from unrelated online benchmarks.

Run all three models on the same hardware and record the actual results.

That becomes part of the value of the project.

---

# Phase 5 — Quantization

Local LLMs can require significant memory.

Quantization reduces the precision used to represent model weights.

For example:

```text
Original Model
     │
     ▼
FP16
     │
     ▼
Quantization
     │
 ┌───┴────┐
 ▼        ▼
Q5       Q4
```

GGUF-based quantized models are commonly available in variants such as Q4 and Q5.

The trade-off being investigated is:

```text
Smaller Model Representation
          +
Lower Memory Consumption
          +
Potentially Faster Inference
          │
          ▼
Possible Quality Reduction
```

The correct question therefore isn't simply:

> Which model is fastest?

Instead:

> Which model provides the best performance, memory usage, reliability and output quality for the application's requirements?

---

# Quantization Benchmark

Run the same evaluation dataset against multiple quantization levels.

For example:

| Metric       | Higher Precision |  Q5 |  Q4 |
| ------------ | ---------------: | --: | --: |
| Model Size   |              TBD | TBD | TBD |
| RAM          |              TBD | TBD | TBD |
| TTFT         |              TBD | TBD | TBD |
| Tokens/sec   |              TBD | TBD | TBD |
| Quality      |              TBD | TBD | TBD |
| JSON Success |              TBD | TBD | TBD |

This demonstrates the practical engineering trade-off between:

**Quality ↔ Memory ↔ Speed**

---

# Observability

AI inference should be observable like any other production workload.

Capture telemetry such as:

```text
Request ID
Model
Model Version
Quantization
Prompt Tokens
Generated Tokens
TTFT
Tokens/sec
Total Latency
Validation Result
Retry Count
Error
Timestamp
```

OpenTelemetry can be introduced to instrument the API and inference pipeline.

The architecture then becomes:

```text
User
 │
 ▼
Frontend
 │
 ▼
ASP.NET Core
 │
 ▼
FastAPI
 │
 ├──────────────► OpenTelemetry
 │
 ▼
Validation / Retry
 │
 ▼
Ollama
 │
 ▼
Local Model
 │
 ▼
CPU / GPU
```

This makes it possible to investigate questions such as:

* Which model has the lowest latency?
* Which model consumes the most memory?
* Which model produces the most validation failures?
* How often is retry required?
* Does quantization materially affect quality?
* Does temperature affect structured-output reliability?

---

# Suggested Repository Structure

```text
Local-Offline-AI-Assistant/
│
├── README.md
├── SOLUTION_OVERVIEW.md
├── benchmark/
│   ├── prompts.json
│   ├── benchmark.py
│   ├── evaluator.py
│   └── results/
│
├── local-ai-service/
│   ├── main.py
│   ├── models/
│   ├── schemas/
│   ├── services/
│   └── telemetry/
│
├── dotnet-api/
│   └── ...
│
├── frontend/
│   └── ...
│
├── docker/
│   └── ...
│
├── docs/
│   ├── architecture.md
│   ├── benchmarking.md
│   ├── model-comparison.md
│   └── quantization.md
│
└── tests/
    └── ...
```

---

# Production-Oriented Architecture

The final implementation can evolve toward:

```text
                    User
                      │
                      ▼
                React / Angular
                      │
                      ▼
              ASP.NET Core API
                      │
                      ▼
              Local AI Gateway
                FastAPI
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
 Prompt Manager   Validation    Observability
                      │
                      ▼
                 Retry Policy
                      │
                      ▼
                    Ollama
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Model A     Model B     Model C
          │           │           │
          └───────────┼───────────┘
                      ▼
                 CPU / GPU
                      │
                      ▼
              Benchmark Engine
                      │
                      ▼
                Evaluation
                      │
                      ▼
               Technical Report
```

---

# What This Project Demonstrates

This project goes beyond simply running an open-source LLM on a laptop.

It demonstrates:

* Local LLM engineering
* Small Language Model deployment
* Offline AI architecture
* AI API development
* Structured generation
* Schema validation
* Retry and resilience patterns
* AI observability
* Performance engineering
* Model benchmarking
* AI evaluation
* Model selection
* Quantization
* Deterministic AI workflows
* Privacy-oriented AI architecture
* Production AI engineering

---

# Core Takeaway

Building a local AI application is not simply:

```text
Prompt → Ollama → LLM → Answer
```

A more mature implementation is:

```text
Prompt
   ↓
Local AI API
   ↓
Structured Generation
   ↓
Local LLM
   ↓
Schema Validation
   ↓
Retry / Recovery
   ↓
Response
   ↓
Telemetry
   ↓
Benchmarking
   ↓
Evaluation
```

And model selection should not be:

```text
Popular Model → Use It
```

It should be:

```text
Candidate Models
      ↓
Standardized Dataset
      ↓
Same Hardware
      ↓
Performance Benchmark
      ↓
Quality Evaluation
      ↓
Reliability Evaluation
      ↓
Quantization Analysis
      ↓
Evidence-Based Model Selection
```

That is what turns a basic local chatbot experiment into a **Local AI Engineering project** suitable for demonstrating production-oriented AI development skills.
