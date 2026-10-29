# Adding Local / Custom Models to OpenClaw

How to register a local or LAN/tailnet OpenAI-compatible endpoint as a provider in
OpenClaw, verified against a real working config. This is the procedure that makes
`deepseek-local/deepseek-v41-abliterated` and `local-qwen/qwen3.8-27b-abliterated-mtplx-6bit`
selectable on this host.

## 0. Where the config lives

```
~/.openclaw/openclaw.json
```

The relevant tree is `models.providers`. `models.mode` is `merge` here, which means
these entries are merged with the built-in catalog rather than replacing it.

## 1. Probe the endpoint before writing config

Never trust the model name the user said out loud. Ask the server.

```bash
# model discovery — the real id
curl -sS -m 10 http://<host>:<port>/v1/models

# a real completion, long enough to survive a reasoning channel
curl -sS -m 60 http://<host>:<port>/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"<id-from-models>","messages":[{"role":"user","content":"Reply OK"}],"max_tokens":900}'
```

Success = `/v1/models` returns `data[].id`, and the completion returns
`finish_reason: "stop"` with non-empty `content`.

If the completion returns `finish_reason: "length"` with empty `content`, the model
spent its budget on a reasoning channel — raise `max_tokens` and retry before
concluding it is broken. Inspect `reasoning_content` in the response to decide
whether the `reasoning` flag should be true; the docs are not authoritative here,
the payload is.

## 2. Diagnose an unreachable endpoint by layer

`tailscale ping <ip>` proves the tunnel is up, not that the port is open. ACLs can
drop a port while ping still answers.

- **Connection timed out** → packets dropped: bind address, host firewall, or ACL.
- **Connection refused** → host answered, nothing listening: wrong port.

On the model host:

```bash
ss -tlnp | grep <port> || lsof -nP -iTCP:<port> -sTCP:LISTEN
curl -s http://<tailscale-ip>:<port>/v1/models
```

If `ss` shows `127.0.0.1:<port>`, the server is localhost-only — restart it bound to
`0.0.0.0` or the tailnet address. A local curl succeeding while a remote one times
out points at firewall/ACL, not the process.

## 3. Write the provider entry

Mirror the shape of an existing working provider. `openclaw config get models` first.

```json
{
  "models": {
    "providers": {
      "<provider-id>": {
        "baseUrl": "https://<tailnet-host>:9443/v1",
        "apiKey": "***",
        "api": "openai-completions",
        "timeoutSeconds": 300,
        "models": [
          {
            "id": "<exact-id-from-/v1/models>",
            "name": "<human label>",
            "contextWindow": 32768,
            "maxTokens": 8192,
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
          }
        ]
      }
    }
  }
}
```

Rules that decide whether the model registers at all:

- **Set `apiKey` even when the server ignores it.** Keyless local providers do not
  become credential-resolvable and never appear in discovery.
- **Set `apiKey`, not `auth`.** Accepted `auth` values are `api-key`, `aws-sdk`,
  `oauth`, `token`. Anything else fails schema validation. Omit the field.
- **`request.allowPrivateNetwork`** only needed when the origin differs from the
  configured `baseUrl` origin (different host or port). The configured
  `scheme://host:port` origin is already trusted for guarded requests, including
  loopback, LAN and tailnet.
- **One provider key per server**, not per model. Model ids go inside `models[]`.
- **`agents.defaults.models["provider/model"]` sets aliases/metadata only.** It does
  not register a runtime model. The provider entry is what makes it selectable.

## 4. Two real entries from this host

Tailnet-served DeepSeek:

```json
"deepseek-local": {
  "baseUrl": "https://m5u-1.dinosaur-alphard.ts.net:9443/v1",
  "apiKey": "***",
  "api": "openai-completions",
  "timeoutSeconds": 300,
  "models": [
    {
      "id": "deepseek-v41-abliterated",
      "name": "DeepSeek V4.1 Abliterated (local)",
      "contextWindow": 32768,
      "maxTokens": 8192,
      "reasoning": false,
      "input": ["text"],
      "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
    }
  ]
}
```

LAN-served Qwen (note the larger window and the reasoning flag — taken from the
probe payload, not assumed):

```json
"local-qwen": {
  "baseUrl": "http://100.81.151.96:8081/v1",
  "apiKey": "***",
  "api": "openai-completions",
  "timeoutSeconds": 300,
  "models": [
    {
      "id": "qwen3.8-27b-abliterated-mtplx-6bit",
      "name": "Qwen 3.8 27B Abliterated (local)",
      "contextWindow": 262144,
      "maxTokens": 32768,
      "reasoning": true,
      "input": ["text"],
      "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
    }
  ]
}
```

## 5. Verify it registered

```bash
openclaw models list
```

Confirm the entry appears with the exact id and the expected `reasoning`/window
values. If it is missing, the usual causes are, in order: no `apiKey` set, an
invalid `auth` value, or a `baseUrl` that does not answer on `/v1/models`.

Select it without changing the default:

```bash
openclaw models set <provider-id>/<model-id>
```

Or set it as a per-agent override in `agents.defaults.models` alongside the default
primary/fallbacks — but remember: metadata only, registration comes from the provider
entry.

## 6. Non-obvious failure modes

- **Model id vs name mismatch.** A community finetune's real id often differs from
  the name people quote. Use the id the server reports.
- **Empty first response.** Reasoning models return empty `content` until they finish
  thinking. Give the probe budget; don't call it broken early.
- **Tailnet vs localhost.** `ss` showing `127.0.0.1` is the most common "it works on
  the box but not remotely" cause.
- **Config silently ignored.** If the provider key is outside `models.providers` or
  `models.mode` is `replace` and you didn't include the built-ins, entries vanish.
- **Credentials in the doc.** `apiKey` values here are `"***"` — local servers rarely
  enforce them, but do not commit real keys.

## 7. Rule for agents using this doc

Probe first, write config second, verify third. Show the diff and get explicit
approval before editing `openclaw.json` — config edits can break the gateway.
