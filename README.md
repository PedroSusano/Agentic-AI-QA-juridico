# Agentic AI Course

This repository is a starting point for developing **agentic artificial intelligence** systems.  
It provides the data, notebooks and code structure needed to build, train and deploy a SageMaker‑based model that can act as an autonomous agent.

> **TL;DR** – Clone → install deps → configure AWS → set up environment variables → train & deploy.

---

## 📁 Project Structure

```
├── data/                   # Raw / processed training & test datasets
├── notebooks/              # Jupyter notebooks for exploration, cleaning & feature engineering
├── scripts/                # Python Entrypoints
├── src/                    # Core application code – models, pipelines, utilities
├── tests/                  # Unit‑test suite (pytest)
├── docs/                   # Project‑specific documentation, diagrams etc.
├── requirements.txt        # Python package dependencies
├── setup.py                # Package installation & metadata
└── README.md               # This file
```

---

## 🚀 Quick Start

1. **Clone the repository**  
   ```bash
   git clone https://github.com/iseg-pg-ai/agentic-ai-course.git
   cd agentic-ai-course
   ```

2. **Create a virtual environment (recommended)**  
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate          # Linux/macOS
   .\.venv\Scripts\activate           # Windows
   ```

3. **Install dependencies**  
   ```bash
   pip install -r requirements.txt
   ```

4. **Create a `.env` file**  
   *Copy the example and fill in your values:*  
     ```bash
     cp .env.example .env
     ```
   *Open `.env` and replace the placeholders:*

   | Variable       | Description                                     | Example        |
   |----------------|-------------------------------------------------|----------------|
   | `VLLM_API_KEY` | VLLM Endpoint API in EC2 to use the GPT OSS 20B | `YOUR-API-KEY` |

