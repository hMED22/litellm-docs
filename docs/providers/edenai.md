import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Eden AI

## Overview

| Property | Details |
|-------|-------|
| Description | Eden AI is an AI gateway: one API key and one bill for 1000+ LLMs from 30+ providers (OpenAI, Anthropic, Google, Mistral, DeepSeek, xAI, Amazon Bedrock, Azure and more), with the real cost of every request reported in the response. |
| Provider Route on LiteLLM | `edenai/` |
| Link to Provider Doc | [Eden AI Documentation ↗](https://www.edenai.co/docs) |
| Base URL | `https://api.edenai.run/v3` |
| Supported Operations | [`/chat/completions`](#usage---litellm-python-sdk) |

<br />
<br />

https://www.edenai.co/docs

**We support ALL Eden AI chat models, just set `edenai/` as a prefix when sending completion requests**

## Required Variables

```python showLineNumbers title="Environment Variables"
os.environ["EDENAI_API_KEY"] = ""  # your Eden AI API key
```

Keys are created in the Eden AI dashboard at https://app.edenai.run under Settings, API Keys.

## Optional Variables

```python showLineNumbers title="Environment Variables"
os.environ["EDENAI_API_BASE"] = "https://api.eu.edenai.run/v3"  # EU endpoint, same key. Default is https://api.edenai.run/v3
```

The EU endpoint accepts the same API key but serves only the subset of the catalog hosted in the EU, so model ids that work on the default host, including the `openai/gpt-mini-latest` and `anthropic/claude-sonnet-latest` examples on this page, may not exist there. Check https://api.eu.edenai.run/v3/models for the ids the EU host serves before switching.

## Model Names

Eden AI model ids are `provider/model`, for example `openai/gpt-mini-latest`, `anthropic/claude-sonnet-latest` or `google/gemini-3.7-flash`. Add the `edenai/` prefix and LiteLLM strips only that prefix, so `edenai/openai/gpt-mini-latest` reaches Eden AI as `openai/gpt-mini-latest`. A bare model name such as `edenai/mistral-small-latest` also works: Eden AI then picks the seller that serves the model (provider routing). Eden AI's own stable aliases, the catalog entries with an `alias_of` such as `openai/gpt-mini-latest`, only resolve with their vendor prefix. Region variants keep their suffix, as in `edenai/vertex/gemini-3.7-flash@eu`.

The catalog is public at https://app.edenai.run/models. From LiteLLM, `litellm.get_valid_models(custom_llm_provider="edenai", check_provider_endpoint=True)` returns the same list with the `edenai/` prefix applied.

## Route Every Eden AI Model Through One Deployment

The proxy can expose the whole catalog with a wildcard deployment. With `check_provider_endpoint` on, `/v1/models` lists every model from `https://api.edenai.run/v3/models` under the `edenai/` prefix, and a request for any of them, such as `edenai/mistral/mistral-small-latest`, routes through this deployment.

```yaml showLineNumbers title="config.yaml"
model_list:
  - model_name: edenai/*
    litellm_params:
      model: edenai/*
      api_key: os.environ/EDENAI_API_KEY

litellm_settings:
  check_provider_endpoint: true
```

## Cost Tracking

Every Eden AI response reports the request's cost in USD, after any account discount, and LiteLLM records that number as the request's spend instead of a price map estimate. On streams the cost arrives on the final usage chunk. LiteLLM always asks Eden AI for that chunk, and forwards it to your client only when you set `stream_options={"include_usage": True}`, so streaming clients see exactly the OpenAI behavior they expect.

## Usage - LiteLLM Python SDK

### Non-streaming

```python showLineNumbers title="Eden AI Non-streaming Completion"
import os
from litellm import completion

os.environ["EDENAI_API_KEY"] = ""  # your Eden AI API key

messages = [{"content": "Hello, how are you?", "role": "user"}]

response = completion(
    model="edenai/openai/gpt-mini-latest",
    messages=messages,
)

print(response)
```

### Streaming

```python showLineNumbers title="Eden AI Streaming Completion"
import os
from litellm import completion

os.environ["EDENAI_API_KEY"] = ""  # your Eden AI API key

messages = [{"content": "Write a short story about AI", "role": "user"}]

response = completion(
    model="edenai/anthropic/claude-sonnet-latest",
    messages=messages,
    stream=True,
    stream_options={"include_usage": True},
)

for chunk in response:
    print(chunk)
```

### Eden AI parameters: fallbacks and routing

Eden AI accepts a few fields beyond the OpenAI set. `fallbacks` lists up to three `provider/model` ids tried in order when the primary model fails, and `routing` steers provider routing for bare model names (`sort` is `cost`, `speed`, `latency` or `exact`, and `allowed_providers` restricts the sellers). Pass them through `extra_body`:

```python showLineNumbers title="Eden AI fallbacks and routing"
import os
from litellm import completion

os.environ["EDENAI_API_KEY"] = ""  # your Eden AI API key

response = completion(
    model="edenai/gpt-5-mini",
    messages=[{"content": "Hello, how are you?", "role": "user"}],
    extra_body={
        "fallbacks": ["anthropic/claude-sonnet-latest"],
        "routing": {"sort": "latency", "allowed_providers": ["openai", "azure"]},
    },
)

print(response)
```

## Usage - LiteLLM Proxy Server

```yaml showLineNumbers title="config.yaml"
model_list:
  - model_name: gpt-mini-latest
    litellm_params:
      model: edenai/openai/gpt-mini-latest
      api_key: os.environ/EDENAI_API_KEY
  - model_name: claude-sonnet
    litellm_params:
      model: edenai/anthropic/claude-sonnet-latest
      api_key: os.environ/EDENAI_API_KEY
  - model_name: gemini-flash-eu
    litellm_params:
      model: edenai/vertex/gemini-3.7-flash
      api_key: os.environ/EDENAI_API_KEY
      api_base: https://api.eu.edenai.run/v3
```

```bash showLineNumbers title="Start LiteLLM Proxy"
litellm --config config.yaml

# RUNNING on http://0.0.0.0:4000
```

<Tabs>
<TabItem value="openai-sdk" label="OpenAI SDK">

```python showLineNumbers title="Eden AI via Proxy"
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:4000",  # Your proxy URL
    api_key="your-proxy-api-key",      # Your proxy API key
)

response = client.chat.completions.create(
    model="gpt-mini-latest",
    messages=[{"role": "user", "content": "Hello, how are you?"}],
)

print(response.choices[0].message.content)
```

</TabItem>

<TabItem value="litellm-sdk" label="LiteLLM SDK">

```python showLineNumbers title="Eden AI via Proxy - LiteLLM SDK"
import litellm

response = litellm.completion(
    model="litellm_proxy/gpt-mini-latest",
    messages=[{"role": "user", "content": "Hello, how are you?"}],
    api_base="http://localhost:4000",
    api_key="your-proxy-api-key",
)

print(response.choices[0].message.content)
```

</TabItem>

<TabItem value="curl" label="cURL">

```bash showLineNumbers title="Eden AI via Proxy - cURL"
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-proxy-api-key" \
  -d '{
    "model": "gpt-mini-latest",
    "messages": [{"role": "user", "content": "Hello, how are you?"}]
  }'
```

</TabItem>
</Tabs>

The proxy's `x-litellm-response-cost` response header and the spend logs carry the cost Eden AI reported for the request.

## Supported OpenAI Parameters

Eden AI takes the full OpenAI chat completions parameter set: `temperature`, `top_p`, `max_tokens`, `max_completion_tokens`, `n`, `stop`, `seed`, `stream`, `stream_options`, `tools`, `tool_choice`, `parallel_tool_calls`, `response_format`, `reasoning_effort`, `logprobs`, `top_logprobs`, `frequency_penalty`, `presence_penalty`, `logit_bias`, `web_search_options`, `modalities`, `audio`, `prediction` and `service_tier`. Provider-specific parameters go through `extra_body`, which Eden AI passes to the underlying provider.
