



# active questions

### uv 为什么要有 uv sync， uv lock .是为了解决什么问题？  

### 以下 pyproject.toml  如何升级 0.1.0 ，如何add "openai>=2.16.0", 

```md
[project]
name = "mem-extract"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
authors = [
    { name = "Zhangda Xu", email = "just4xzd@icloud.com" }
]
requires-python = ">=3.12"
dependencies = [
    "openai>=2.16.0",
    "httpx>=0.27.0",
]

[build-system]
requires = ["uv_build>=0.9.26,<0.10.0"]
build-backend = "uv_build"

```