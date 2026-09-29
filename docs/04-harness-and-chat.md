# 04 — The Managed Harness and `chat.py`

## 1. What the AgentCore managed harness is

The harness is a **managed agent loop**: you give it a model, a system prompt and
tools; it handles model calls, **session memory across turns** and tool execution.
There is no "prepare" or redeploy step — prompt changes are live as soon as the
harness update finishes.

## 2. Create (or update) the harness

```bash
python create_harness.py     # first run ~2–3 minutes
```

What `create_harness.py` does:
1. Reads `system_prompt.txt` and replaces `{{FAQ}}` with `online_shop_faq.md`.
2. Creates the harness `support_chatbot` with the model pinned to
   `us.amazon.nova-pro-v1:0` and **greedy decoding** (`temperature: 0.0`, `topK: 1`,
   AWS's recommendation for reliable Nova tool calling). If the harness already
   exists it is **updated** instead.
3. Waits until the harness is `READY` and records its ARN in `agentcore_config.json`.

## 3. Chat with it

```bash
python chat.py
```

```
Connected to harness support_chatbot (session <uuid>-support-chat).
you> Hi, something's broken on the checkout page.
bot> Could you please describe what is broken on the checkout page?
...
[tool call] bugreports___create_bug_report
bot> ... your bug report has been submitted with the ticket ID 875ad038-...
```

- Each `chat.py` run is **one fresh session** (new `runtimeSessionId`).
- `[tool call] bugreports___create_bug_report` proves the model used the tool. If it
  never appears, the prompt isn't telling the model clearly when to use it.
- Nova Pro prints its reasoning in `<thinking>…</thinking>` blocks — useful for
  seeing how the prompt's classification step is applied.

### This submission's change to `chat.py` (stand-out)

`chat.py` pins the model on every invoke; this submission also attaches the
**Bedrock Guardrail** there:

```python
model={
    "bedrockModelConfig": {
        "modelId": config.get("model_id", "us.amazon.nova-pro-v1:0"),
        "additionalParams": {
            "guardrailConfig": {
                "guardrailIdentifier": "arn:aws:bedrock:us-east-1:<account>:guardrail/<id>",
                "guardrailVersion": "1",
                "trace": "enabled_full",
            }
        },
    }
},
```

When rebuilding in a new account, replace the guardrail ARN with your own
(see [07-standout](07-standout.md)).

## 4. Verify tickets really land in DynamoDB

```bash
aws dynamodb scan --table-name bug-report-tool-stack-bug-reports --region us-east-1
```

Evidence: `screenshots/04_chat_bug_report.png` (conversation) and
`screenshots/05_dynamodb_scan_from_chat.png` (the ticket from that conversation).
