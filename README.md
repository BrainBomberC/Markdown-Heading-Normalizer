<h1 align="center">Markdown Heading Normalizer</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.8%2B-blue?style=flat-square&logo=python" alt="Python" />
  <img src="https://img.shields.io/badge/Markdown-Heading_Normalization-black?style=flat-square&logo=markdown" alt="Markdown" />
  <img src="https://img.shields.io/badge/LLM-Semantic_Judgement-purple?style=flat-square" alt="LLM" />
</p>

<p align="center">
  <b>Heading hierarchy repair and document structure reconstruction for OCR / MinerU-generated Markdown.</b>
</p>

---

## ✨ Overview

Markdown files produced by OCR or MinerU often contain extracted text but suffer from unreliable heading structures:

- Body text, author names, headers, and footers misidentified as `#` headings;
- Chaotic `#` / `##` / `###` hierarchy;
- Table-of-contents entries mixed with body headings;
- Section numbers present but not aligned with Markdown heading levels.

Our goal:

> **Batch-convert documents using a locally deployed model (cloud models are also supported). Experiments show that the 30B Qwen3 already meets requirements — transforming OCR / MinerU Markdown output into clean, well-structured documents ready for downstream RAG / GraphRAG / corpus construction.**

The core script:

```text
tools.py
```

**Typical input:**

```markdown
# Embedded Systems
# Hardware and Software Architecture
## Book Features
# Tammy Noergaard
# Table of Contents
```

**Typical output:**

```markdown
# Embedded Systems
## Hardware and Software Architecture
## Book Features
Tammy Noergaard
## Table of Contents
```

---

## 📌 Use Cases

| Document Type                    |  Suitability  | Notes                                                    |
| -------------------------------- | :------------: | -------------------------------------------------------- |
| Journal / conference papers      | ✅ Recommended | Short length, stable heading patterns                    |
| Master's / PhD theses            | ✅ Recommended | Clear chapter numbering and TOC structure                |
| Technical reports, lecture notes |  ⚠️ Usable  | Verify heading pattern consistency                       |
| 500+ page books                  | ⚠️ Cautious | Many heading candidates; risk of cross-batch level drift |

> **Note:** The project currently uses a locally deployed **Qwen3-30B** model, which performs well across the above scenarios, balancing semantic accuracy with inference speed.

---

## 🚀 Quick Start

### 1. Environment Setup

The core script `tools.py` has zero third-party dependencies — it uses only the Python standard library. Requires Python ≥ 3.8. `ipykernel` is only needed for `test_stepbystep.ipynb` (a debugging notebook for `tools.py`).

#### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pip
pip install -r requirements.txt
```

#### Ubuntu / macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip
pip install -r requirements.txt
```

#### Conda

```bash
conda create -n md-heading python=3.10
conda activate md-heading
pip install -r requirements.txt
```

---

### 2. Configure the LLM API in `tools.py`

This tool uses an LLM to judge heading semantics, so the model API must be configured first.

Open `tools.py` and fill in your service details:

```python
API_KEY = "your_api_key_here"
BASE_URL = "http://192.168.3.131:8000/v1/chat/completions"
MODEL = "deepseek-v4"
```

Or configure via environment variables:

```text
LLM_API_KEY / API_KEY
LLM_BASE_URL
LLM_MODEL
```

Input / output token settings (`tools.py` top-level constants):

```python
CONTEXT_LENGTH = 16384    # Model context window size
OUTPUT_TOKENS  = 2000     # Max output tokens per LLM call
TOKEN_BUDGET   = CONTEXT_LENGTH - OUTPUT_TOKENS - 700   # Available token budget for prompts
```

---

### 3. Activate the Environment and Run

#### Single File

```powershell
# Windows PowerShell
.\.venv\Scripts\Activate.ps1
python tools.py example.md
```

```bash
# Ubuntu / macOS
source .venv/bin/activate
python tools.py example.md
```

#### Batch Processing a Folder

Place `.md` files in a single folder (e.g. `inputs\`) and pass the folder path:

```powershell
# Windows PowerShell
.\.venv\Scripts\Activate.ps1
python tools.py inputs\
```

```bash
# Ubuntu / macOS
source .venv/bin/activate
python tools.py inputs/
```

> `tools.py` recursively discovers all `.md` files under the given folder and processes each one.

#### Customizing Output Paths

`--out-dir` and `--json-dir` are CLI arguments specified at runtime:

```bash
python tools.py inputs/ --out-dir my_output --json-dir my_json
```

| Argument        |       Default       | Description                                             |
| --------------- | :------------------: | ------------------------------------------------------- |
| `--out-dir`   | `standardized_md2` | Output directory for normalized Markdown                |
| `--json-dir`  |  `heading_json2`  | Output directory for heading decision JSONs             |
| `--cache-dir` |          —          | Enable LLM result caching to avoid repeated calls       |
| `--resume`    |          —          | Skip already-processed files (resume from interruption) |

Full example:

```bash
python tools.py inputs/ --out-dir outputs --json-dir json_results --cache-dir .cache --resume
```

---

## 🧠 First-Principles Design

Markdown heading normalization is not simply changing `#` to `##`. For every heading candidate, two questions must be answered:

| Question                        | Output Variable |
| ------------------------------- | --------------- |
| Is this line a real heading?    | `is_heading`  |
| If so, what level should it be? | `level`       |

The entire project can be understood as a **two-layer judgment system**:

![Two-Layer Architecture](架构图en.png)

### Why LLM?

Many heading problems cannot be solved by regex alone. For example:

```markdown
# Tammy Noergaard
```

It looks like a heading, but in context it could just be an author name. Such cases require semantic understanding — exactly what LLMs provide.

LLM responsibilities:

- Detecting whether author names, copyright pages, headers, and footers are misidentified as headings;
- Judging whether "Abstract", "Table of Contents", "References" are genuine document structure;
- Determining the semantic role of each candidate line based on context.

### Why Rules?

LLMs excel at semantic understanding but are not guaranteed to maintain global consistency. For instance:

```text
1.1 → judged as H3
1.2 → judged as H2
1.3 → judged as H3
```

All look like headings semantically, but structurally they must be uniform.

Thus the second layer handles structural repairs:

- TOC entries are excluded from body headings;
- Sibling-numbered items like `1.1`, `1.2`, `1.3` are unified to the same level;
- Level gaps such as `H2 → H4` are corrected;
- The entire document is guaranteed a reasonable, unbroken heading hierarchy.

Core principle:

> **LLM judges *whether* something is a heading; rules ensure *level consistency*.**

The final output preserves the original content while adjusting only the heading structure, providing a solid foundation for downstream chapter segmentation, RAG retrieval, GraphRAG knowledge base construction, and Neo4j chapter-node modeling.
