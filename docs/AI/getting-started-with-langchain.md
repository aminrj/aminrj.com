# Getting started with LangChain

LangChain helps you develop applications powered by language models such as ChatGPT. 

In this video we will start by installing LangChain and set up a new environment to build a simple application with langChain.

We will go through and explain the most basic common components of LangChain such as prompt templates, models and output parsers.

This tutorial is intended for complete beginners, so don’t worry, just stick around, we will explain all these concepts and demonstrate how to use them.

## Installation

### Jupyter Notebook

First, let’s quickstart our environment using the Python Jupyter Notebook.

Jupyter notebook is a web application for creating and running python programs on your browser.

### OLLAMA

[Ollama](https://ollama.ai/) allows you to run open-source large language models, such as Llama 2, locally.
First, follow these instructions to set up and run a local Ollama instance:

- [Download](https://ollama.ai/download)
- Fetch a model via `llama pull llama2`

```python
from langchain_community.llms import Ollama
llm = Ollama(model="llama2")
```
