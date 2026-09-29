# 00 — Overview: What Was Built and Why

These notes explain the **Customer Support Chatbot** project (Udacity AWS Agentic AI —
Course 1, *Prompting & LLM Reasoning*) step by step, so it can be rebuilt from scratch
without help.

| File | Contents |
|---|---|
| [00-overview.md](00-overview.md) | Concepts, architecture, request flow (this file) |
| [01-setup.md](01-setup.md) | Tools, credentials, model access, Python environment |
| [02-bug-report-tool-and-gateway.md](02-bug-report-tool-and-gateway.md) | CloudFormation tool stack, Lambda, DynamoDB, AgentCore Gateway |
| [03-system-prompt.md](03-system-prompt.md) | The main deliverable: routing, bug-report checklist, FAQ grounding, injection hardening |
| [04-harness-and-chat.md](04-harness-and-chat.md) | Creating/updating the managed harness and chatting with it |
| [05-bedrock-evaluations.md](05-bedrock-evaluations.md) | Test suite → JSONL dataset → Bedrock Evaluations (LLM-as-a-judge) |
| [06-testing-and-submission.md](06-testing-and-submission.md) | Manual scenarios, evidence, checklist, clean-up |
| [07-standout.md](07-standout.md) | Stand-out extensions (injection hardening, edge-case tests, Guardrail) |
| [08-troubleshooting.md](08-troubleshooting.md) | Every real problem hit during the build, with the fix |

---

## 1. The business problem

A fictional online shop receives three kinds of customer messages:

| Type | Desired behaviour |
|---|---|
| **Bug report** | Collect the details over the conversation, then file a ticket |
| **Platform question** (orders, shipping, returns, payments…) | Answer from the shop's FAQ |
| **Anything else** | Politely hand off to the human support line |

## 2. The key idea: the prompt *is* the application

The project is about **prompt engineering**. There is no classifier model, no flow
graph and no condition nodes — all routing, information gathering and grounding
live in **one system prompt** (`starter/system_prompt.txt`). The AgentCore **managed
harness** supplies the agent loop (model calls, session memory, tool execution);
the prompt supplies the behaviour.

> The rubric still uses Bedrock *Flows* vocabulary (classifier node, condition
> nodes) because Bedrock Agents Classic closed to new customers on 30 July 2026 and
> the course moved to its successor, AgentCore. The mapping is in the README.

## 3. Architecture

```
Customer ── chat.py / generate-eval-dataset.py ──► AgentCore managed harness
                                                     │  model: us.amazon.nova-pro-v1:0
                                                     │  system prompt = system_prompt.txt
                                                     │                  + online_shop_faq.md ({{FAQ}})
                                                     │  session memory across turns
                                                     ▼
                                          AgentCore Gateway (MCP, AWS_IAM)
                                                     │  tool: bugreports___create_bug_report
                                                     ▼
                                          Lambda create_bug_report ──► DynamoDB (tickets)

Testing:  harness-tests.json → generate-eval-dataset.py → output_eval_dataset.jsonl
          → S3 → Bedrock Evaluations (Builtin.Correctness, judge: Nova Pro)
Safety:   Bedrock Guardrail (content filters + prompt-attack filter) attached per invoke
```

| Component | Created by | Role |
|---|---|---|
| DynamoDB table + Lambda + IAM roles | `cloudformation-tool.yaml` | Ticket store and the tool implementation |
| AgentCore Gateway + target | `setup_gateway.py` | Presents the Lambda to the model as `create_bug_report` |
| Managed harness | `create_harness.py` | Runs the chatbot with the system prompt |
| S3 bucket + evaluation role | `cloudformation-testing.yaml` | Inputs/outputs for Bedrock Evaluations |

## 4. Request flow: a bug report over several turns

1. *"Hi, something's broken on the checkout page."* → the prompt classifies it as a
   **bug report** → asks for the description.
2. *"The page goes blank when I click Pay Now."* → description ✓ → asks for the steps.
3. *"I added an item, went to checkout, filled my address, clicked Pay Now."* →
   steps ✓ → asks for the environment.
4. *"Chrome on a Windows 11 laptop."* → all three fields ✓ → **tool call**
   `bugreports___create_bug_report` → Lambda writes the ticket to DynamoDB → the bot
   relays the real `ticketId`.

The harness keeps **session state**, so the model remembers what was already
collected across turns — no code needed for that.

## 5. Key concepts learned

- **Routing as classification inside the prompt**: crisp category definitions and
  "pick exactly one category first".
- **Slot filling**: a checklist of required fields, one question at a time, and a
  hard rule not to call the tool before the checklist is complete.
- **Grounding**: answer only from the embedded FAQ, and treat "not in the FAQ" as a hand-off.
- **Prompt injection defence** in the prompt, backed by a Guardrail outside the prompt.
- **LLM-as-a-judge evaluation** with Bedrock Evaluations, and why a single run is a
  sample, not a guarantee.
