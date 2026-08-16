---
name: diagram
description: >
  Publish a Mermaid diagram to diagram.bhk.dev and return a permanent, shareable viewer URL.
  Use when the user asks for a diagram link, a shareable diagram, to publish Mermaid, or mentions
  diagram.bhk.dev. Mermaid only; the service is free and does not require authentication.
---

# Diagram

`diagram.bhk.dev` stores Mermaid source by content hash and returns a permanent public URL.

## Publish

1. Confirm the diagram contains no sensitive information. Published source is public and cannot be
   edited or deleted.
2. Write the Mermaid source to a scratch file.
3. POST it to `/api/render` and give the user the returned `url`.

```bash
cat > diagram.mmd <<'EOF'
graph TD
  A[Start] --> B{Valid?}
  B -->|yes| C[Render]
  B -->|no| D[Fix syntax]
  D --> B
EOF

jq -n --rawfile code diagram.mmd '{code: $code, type: "mermaid"}' \
  | curl -sS -X POST https://diagram.bhk.dev/api/render \
      -H 'Content-Type: application/json' \
      --data-binary @- \
  | jq -r '.url'
```

Use a scratch file and `jq --rawfile` rather than manually escaping multi-line Mermaid in JSON.
Remove the final `| jq -r '.url'` to inspect the full response.

## API Contract

`POST https://diagram.bhk.dev/api/render`

```json
{"code":"<mermaid source>","type":"mermaid"}
```

Both fields are required. A new diagram returns `201`; an existing identical diagram returns `200`.
Both responses include:

| Field | Meaning |
| --- | --- |
| `url` | Shareable browser viewer. Return this to the user. |
| `raw` | Plain-text Mermaid source URL. |
| `hash` | Content-derived 16-character hash. |
| `cached` | `true` when the same source was already stored. |

## Important Behavior

- The API stores source but does not validate Mermaid syntax. Check syntax before publishing.
- Publishing is idempotent: identical source returns the same URL with `cached: true`.
- Whitespace changes the content hash and produces a different URL.
- Quote labels containing punctuation, for example `A["User: sign-in (SSO)"]`.
- `400` indicates invalid input, `413` indicates source over 50,000 characters, and `404` indicates an unknown diagram hash.
