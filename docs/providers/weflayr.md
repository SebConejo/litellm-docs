---
title: "Weflayr API with LiteLLM: Python SDK and Gateway"
sidebar_label: "Weflayr"
description: Call models and Weflayr routing strategies through LiteLLM with the Python SDK or Gateway. Configure your API key, model name, and optional API base.
---

# Weflayr

[Weflayr](https://weflayr.com) is an OpenAI-compatible LLM gateway hosted in the EU. It benchmarks models on your own past calls and routes each call to the model you choose with a routing strategy. Use the `weflayr/` prefix with a model name or a strategy id from your Weflayr project.

<SinceVersion v="1.113.0" />

Provider support was added in [PR #54447](https://github.com/BerriAI/litellm/pull/54447). Use this version or a later build that includes the provider.

## Overview

| Property | Details |
|----------|---------|
| Provider prefix | `weflayr/` |
| API key environment variable | `WEFLAYR_API_KEY` |
| Default API base | `https://gateway.weflayr.com/v1` |
| Optional API base override | `WEFLAYR_API_BASE` or the `api_base` parameter |
| Upstream endpoints | `/chat/completions`, `/responses`, `/embeddings` |
| Provider documentation | [Weflayr docs](https://docs.weflayr.com) |

## Setup

Create an API key in the Weflayr dashboard and export it before you start the SDK or proxy:

```bash
export WEFLAYR_API_KEY="your-weflayr-api-key"
```

Use `weflayr/<model-name>` for a model from the Weflayr catalog, for example `weflayr/{{openai_small}}`. LiteLLM removes the `weflayr/` prefix before it sends the model value to Weflayr.

## LiteLLM Python SDK

```python
from litellm import completion

response = completion(
    model="weflayr/{{openai_small}}",
    messages=[{"role": "user", "content": "Say hello in five words"}],
)

print(response.choices[0].message.content)
```

To call a routing strategy, put its id after `strategy/`. The `weflayr` field names the call, so Weflayr can group it for benchmarks and per-customer costs:

```python
from litellm import completion

response = completion(
    model="weflayr/strategy/production-default",
    messages=[{"role": "user", "content": "Say hello in five words"}],
    extra_body={"weflayr": {"call_name": "greeting", "customer_name": "acme"}},
)
```

## LiteLLM Gateway

Save this configuration as `config.yaml`:

```yaml
model_list:
  - model_name: {{openai_small}}
    litellm_params:
      model: weflayr/{{openai_small}}
      api_key: os.environ/WEFLAYR_API_KEY
```

Start the proxy with the API key exported in its environment:

```bash
litellm --config config.yaml
```

Send a request using the configured model alias:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "{{openai_small}}",
    "messages": [{"role": "user", "content": "Say hello in five words"}]
  }'
```

If your proxy requires authentication, also send `Authorization: Bearer <your-litellm-key>`. This is your LiteLLM key, not your Weflayr API key.

## Cost tracking

Weflayr returns the charge for each chat completions and Responses call in `usage.cost`, and the proxy reports that value in `x-litellm-response-cost`. Embeddings responses carry no `usage.cost`, so the proxy shows no cost for them.
