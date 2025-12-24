---
title: Python Environment Setup
description: (Mac) Python package manager with uv
date: 2025-10-27 12:00:00+0900
tags: 
    - python
    - uv
categories:
    - python
---

# Install uv

https://docs.astral.sh/uv/getting-started/installation/

```bash
brew install uv
uv --version
echo 'eval "$(uv generate-shell-completion zsh)"' >> ~/.zshrc
source ~/.zshrc
```

# uv Use Case

https://docs.astral.sh/uv/guides/install-python/

```bash
uv init
uv venv
uv add pandas
uv run script.py
```
