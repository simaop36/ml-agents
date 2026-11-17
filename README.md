# ml-agents

AI agents that run end-to-end machine learning experiments: given a dataset and a goal, they explore and clean the data, train and compare models, check for data leakage and overfitting, and write a structured report.

> 🚧 **Work in progress.** Started November 2026. See the roadmap below for current status.

## Why this project

At university I built classification and regression workflows in KNIME for banking and energy datasets, following CRISP-DM. I cleaned the data, engineered features, tuned tree-based models and caught a data leakage problem by hand (best result: R² = 0.802 on energy production).

This project asks a simple question: **can a system of AI agents do the same work, on the same datasets, and how does it compare?** It is also how I am learning to build with LLMs properly: tool use, MCP, multi-agent orchestration, evaluation and cost control.

## Roadmap

- [ ] **Phase 0 – Setup:** repository, datasets and KNIME baseline documented
- [ ] **Phase 1 – Single agent:** hand-written tool-use loop on the Claude API, sandboxed code execution in Docker, validated JSON reports
- [ ] **Phase 2 – MCP server:** experiment history in SQLite, exposed through an MCP server
- [ ] **Phase 3 – Multi-agent:** orchestrator with cleaning, modelling and reviewer subagents (Claude Agent SDK), with hooks that block destructive actions
- [ ] **Phase 4 – Evaluation:** test suite of datasets with known answers, model and cost comparison, prompt-injection tests, CI

## Results

_Coming after Phase 1: KNIME baseline vs. agent, with cost per run._

## Tech

Python · Claude API · scikit-learn · pandas · Pydantic · pytest · Docker · SQLite · MCP

## Setup

```bash
git clone https://github.com/simaop36/ml-agents.git
cd ml-agents
python3 -m venv .venv && source .venv/bin/activate
cp .env.example .env   # then add your own Anthropic API key
```
