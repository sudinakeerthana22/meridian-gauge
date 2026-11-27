# Model gateway — spec

Owner: Hanisha. Folder: `gateway/`. Status: draft, waiting for approval.

## What it does

The gateway is the one way every part talks to a model. A caller names a model from
`config.yaml`; the gateway sends the request to that model's host, retries temporary errors,
and returns the reply together with tokens, cost, time and the model that actually answered.
Switching models means editing the config, not the code.

It replaces the temporary `OpenAICompatibleClient` in `coding_agent/model_client.py`. The
coding agent's loop does not change, because the gateway offers the same `chat(messages, tools)`
method and returns the same `ModelReply`.

## What it reads and writes

Reads:
- `config.yaml`: the `models:` list (and `budgets.max_retry_wait_seconds`). Same format as
  `config.example.yaml`.
- `.env`: the API keys, by the name given in each model's `api_key_env`.

Returns, for every call (the `ModelReply` the agent already uses):
`message`, `tool_calls`, `tokens_in`, `tokens_out`, `cost_usd`, `model_served`, plus
`seconds` (time the host took, not counting waits) and `waited_seconds` (time spent waiting out rate limits).

Writes: one line per call to `gateway_calls.jsonl` in the run folder (see open question 2).

Feeds the fields of `examples/attempt.json`: `model_requested`, `model_served`, `host`,
`tokens_in`, `tokens_out`, `cost_usd`.

## The steps, in order

1. Load `config.yaml` and look up the requested model name. Unknown name: stop with an error
   that lists the valid names.
2. Read the API key from the environment variable named by `api_key_env`. Missing: stop with an
   error naming the variable. Keys never appear in the config, the code or the logs.
3. Send the request to the model's `base_url` (OpenAI-compatible), with its temperature.
4. On a temporary error (408, 409, 429, 500, 502, 503, 504, timeout, dropped connection, empty
   reply) wait and retry, up to `max_retry_wait_seconds` in total. A daily quota error is not
   retried. Then stop with a clear error saying which model, which host and why.
5. Check the answer: compare the model that answered with the one asked for. A different model
   is rejected with an error (see open question 1).
6. Work out cost: `tokens_in * price_in_per_m + tokens_out * price_out_per_m`, per million tokens.
   Free and self-hosted models have price 0; their GPU cost is computed by the orchestrator
   from `gpu_price_per_hour` (see open question 3).
7. Append the call to `gateway_calls.jsonl` and return the reply.

The Claude API is added last, as its own small adapter with the same `chat()` method and the
same reply, so callers cannot tell the difference.

## Files

| File | Purpose |
| --- | --- |
| `gateway/__init__.py` | `load_gateway(config)` and `get_client(model_name)` |
| `gateway/client.py` | The OpenAI-compatible client with retries and the model check |
| `gateway/claude_adapter.py` | Claude API adapter (last) |
| `gateway/README.md` | A short note for Keerthana and Zayna: how to call the gateway |
| `tests/test_gateway.py` | Tests below |

## How I will test it ("Done when")

Done when two hosts work by changing only the model name, and a wrong model name is caught.

- Unit tests with a fake host (no internet, no keys): a normal reply is returned with tokens
  and cost; a 429 then success is retried; a daily quota error is not retried; a persistent
  error stops with a clear message; a missing key is named; an unknown model name lists the valid names.
- Wrong-model test: the fake host answers with a different model name; the gateway rejects it.
- Live check: the same script runs against two hosts (Groq and OpenRouter or Ollama) changing
  only the model name in the call.
- The coding agent's existing tests still pass with the gateway swapped in.
- No key appears in any log file or in the repository.

## Open questions for the team

1. **Model-name check.** Hosts often return a slightly different name than the one sent
   (for example a provider prefix or a version suffix). Proposal: add an optional `model:` field
   to each model in `config.yaml` (the host's id; defaults to the config key), and accept the reply if
   the served name equals it after lowercasing and removing the provider prefix. Is a config format change OK?
2. **Call log.** Should the gateway write `gateway_calls.jsonl` itself, or only return the data
   and let the orchestrator write it? The README says parts exchange files in the run folder.
3. **GPU cost.** `attempt.json` has `gpu_hours`. Should the orchestrator compute it from the
   attempt's minutes, or should the gateway track it?
4. **Reuse.** Keerthana's retry code in `coding_agent/model_client.py` already works and is tested. Move
   it into `gateway/` and have the agent import it, or copy it?
5. **Claude API.** Who holds the key and the budget for the last adapter?
6. **Hosts.** Which hosts must work first? Hugging Face's free tier returned 402 for us.
