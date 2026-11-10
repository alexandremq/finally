---
name: openrouter
description:  Use this to write code to call an LLM using LiteLLM with OpenRouter inference provider 
---

# Calling an LLM via Openrouter

These instructions allow you write code to call an LLM with OpenRouter specified in the .env file environment variable OPENROUTER_MODEL.

## Setup

The `OPENROUTER_API_KEY` and `OPENROUTER_MODEL` (default: `openrouter/openrouter/free`)
must be set in the `.env` file in the project root and loaded as environment variables.
`docker compose` loads `.env` via `env_file`; for local runs call `load_dotenv()`.

The uv project must include litellm python SDK (https://github.com/BerriAI/litellm),
pydantic (https://pydantic.dev), and python-dotenv.
`uv add litellm pydantic python-dotenv`

## Code snippets

Use code like these examples in order to use OpenRouter.

### Imports and constants

```python
import os

from dotenv import load_dotenv
from litellm import acompletion, completion

load_dotenv()
MODEL = os.getenv("OPENROUTER_MODEL", "openrouter/openrouter/free")
EXTRA_BODY = {"provider": {"order": ["openrouter"]}}
```

Only pass `reasoning_effort="low"` for reasoning-capable models; omit it otherwise.

### Code to call via OpenRouter for a text response

```python
response = completion(model=MODEL, messages=messages, extra_body=EXTRA_BODY)
result = response.choices[0].message.content
```

### Code to call via OpenRouter for a Structured Outputs response

```python
response = completion(model=MODEL, messages=messages, response_format=MyBaseModelSubclass, extra_body=EXTRA_BODY)
result = response.choices[0].message.content
result_as_object = MyBaseModelSubclass.model_validate_json(result)
```

### Code to stream a text response (as in `backend/app/llm.py::stream_chat_response`)

```python
async def stream_chat_response(messages: list[dict]):
    response = await acompletion(model=MODEL, messages=messages, stream=True, extra_body=EXTRA_BODY)
    async for chunk in response:
        if chunk.choices and chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content
```

### Code to extract structured fields (as in `backend/app/llm.py::extract_fields`)

```python
async def extract_fields(messages: list[dict]) -> MyBaseModelSubclass:
    try:
        response = await acompletion(
            model=MODEL, messages=messages, response_format=MyBaseModelSubclass, extra_body=EXTRA_BODY
        )
        result = response.choices[0].message.content
        if not result:
            return MyBaseModelSubclass()
        return MyBaseModelSubclass.model_validate_json(result)
    except Exception:
        return MyBaseModelSubclass()
```