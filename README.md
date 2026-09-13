# Customer Support Chatbot — Amazon Bedrock AgentCore

A customer support chatbot for a fictional online shop, built on the **Amazon Bedrock AgentCore managed harness**. It routes every incoming customer message to one of three behaviors — bug report collection, FAQ answering, or a polite hand-off — using a single system prompt, with no separate classifier or condition nodes.

## ⚠️ A note on terminology (please read before reviewing)

This project's official rubric still refers to **Bedrock Flows**, **Classifier nodes**, and **Condition nodes**. This is because Bedrock Agents Classic — the service the course originally used — was closed to new customers on July 30, 2026. The course was updated to use its successor, **Bedrock AgentCore**, but the rubric text was not fully updated to match. The table below maps the rubric's terminology to what is actually implemented in this submission.

| Rubric term (Bedrock Flow era) | What's actually implemented (AgentCore) |
|---|---|
| Bedrock Flow | `system_prompt.txt` — a single system prompt |
| Classifier node / prompt | The routing instructions inside `system_prompt.txt` |
| Condition node expressions | Natural-language conditional instructions in the prompt (no visual node graph) |
| Separate Output node per path | Three behaviors handled within one response: bug report, FAQ answer, hand-off |
| "Bug Report Agent" | The `create_bug_report` tool, invoked via an AgentCore Gateway, defined in the same prompt |
| `flow-tests.json` | `harness-tests.json` |
| Flow diagram screenshot | `system_prompt.txt` content + `chat.py` conversation transcripts showing routing in action |

The official starter repository (already migrated to AgentCore) is here: `https://github.com/udacity/aws-c1-prompting-llm-reasoning-nd905-cd14762-project`

## What this chatbot does

1. **Bug reports** — collects `description`, `stepsToReproduce`, and `environment` across a multi-turn conversation (asking for one missing field at a time), then calls the `create_bug_report` tool through an AgentCore Gateway to file a ticket in DynamoDB, and relays the ticket ID to the customer.
2. **Platform questions** (orders, shipping, returns, payments, etc.) — answered strictly from the embedded FAQ document (`online_shop_faq.md`, injected via the `{{FAQ}}` placeholder). Questions not covered by the FAQ are handed off to a human support line instead of being guessed.
3. **Anything else** — politely redirected to the human support line (1-800-555-0199, Mon–Fri).

The prompt also includes hardening against prompt-injection attempts (e.g. "ignore your previous instructions"), addressing one of the project's stand-out suggestions.

## Screenshots

![FAQ question answered correctly](screenshots/06_chat_faq_covered.png)

![Multi-turn bug report collected and filed](screenshots/04_chat_bug_report.png)

![Prompt-injection attempt blocked by Guardrails](screenshots/11_guardrail_blocked.png)

![Automated Bedrock Evaluations results](screenshots/09_bedrock_evaluation_results.png)

![Off-topic request handed off to human support](screenshots/08_chat_other_request.png)

The full evidence set (including Lambda/DynamoDB verification and Guardrail configuration) is in the [Evidence index](#evidence-index) below.

## Architecture

- **AWS Lambda** (`create_bug_report.py`) — writes bug reports to DynamoDB
- **Amazon DynamoDB** — ticket storage (`bug-report-tool-stack-bug-reports`)
- **AgentCore Gateway** — exposes the Lambda as a tool (`bugreports___create_bug_report`) to the model
- **AgentCore managed harness** — runs the agent loop, session memory, and tool execution, using `us.amazon.nova-pro-v1:0`
- **Amazon Bedrock Evaluations** — automated, LLM-as-a-judge testing of the chatbot's responses
- **Amazon Bedrock Guardrails** *(stand-out)* — an independent content-safety layer blocking harmful content and prompt-injection attempts before they reach the model

## Files in this submission

| File | Description |
|---|---|
| `system_prompt.txt` | The chatbot's system prompt — the main deliverable |
| `harness-tests.json` | 9 test cases covering all three routes plus edge cases (ambiguous message, very short message, prompt injection) |
| `output_eval_dataset.jsonl` | Output of `generate-eval-dataset.py`, used as input to Bedrock Evaluations |
| `observations.md` | Written analysis of the evaluation results, including known weaknesses and the debugging process |
| `screenshots/` | Evidence for every rubric criterion (see below) |

## Evidence index

| # | File | What it shows |
|---|---|---|
| 1 | `screenshots/02_lambda_test_result.png` | Manual test of the `create_bug_report` Lambda in isolation |
| 2 | `screenshots/03_dynamodb_item.png` | Ticket created by the manual Lambda test, visible in DynamoDB |
| 3 | `screenshots/04_chat_bug_report.png` | Full `chat.py` conversation: multi-turn bug report collection ending in a `[tool call]` and a ticket ID |
| 4 | `screenshots/05_dynamodb_scan_from_chat.png` | DynamoDB scan confirming the ticket from the chatbot conversation (not the manual test) was persisted |
| 5 | `screenshots/06_chat_faq_covered.png` | A platform question answered correctly from the FAQ |
| 6 | `screenshots/07_chat_faq_uncovered.png` | A platform question not covered by the FAQ, correctly handed off to human support |
| 7 | `screenshots/08_chat_other_request.png` | An off-topic request correctly redirected to human support |
| 8 | `screenshots/09_bedrock_evaluation_results.png` | Bedrock Evaluations job results (Builtin.Correctness score, 9 prompts) |
| 9 | `screenshots/10_guardrail_config.png` | Bedrock Guardrail configuration — content filters and prompt-attack filter, all enabled |
| 10 | `screenshots/11_guardrail_blocked.png` | Guardrail successfully blocking a prompt-injection attempt |

## Results summary

The final Bedrock Evaluations run scored **~0.72–0.78** on Builtin.Correctness across 9 test prompts (7 scored 1, one scored 0.5, one scored 0). The chatbot reliably handles clear-cut cases in all three routes. Its main weakness is being slightly too eager to file a bug-report ticket on ambiguous or incomplete messages, instead of asking a clarifying question first. Full analysis, including the debugging process and what was tried, is in `observations.md`.

## How to run this project

See the official project instructions in the starter repository linked above (Environment Setup → Instructions → Testing Framework pages) for the full step-by-step: deploying the CloudFormation stacks, creating the AgentCore Gateway, building the harness from `system_prompt.txt`, and running the evaluation pipeline.

## Stand-out suggestions implemented

- ✅ **Prompt-injection hardening** — explicit instructions in `system_prompt.txt` refusing to reveal the prompt or change behavior based on customer instructions
- ✅ **Edge-case tests** — `harness-tests.json` includes an ambiguous message, a very short message, and a prompt-injection attempt
- ✅ **Guardrails** — an Amazon Bedrock Guardrail with content filters and a prompt-attack filter, both set to block, configured as an independent safety layer
- ⬜ Bedrock Knowledge Base (not implemented — out of scope for this submission's timeline; the FAQ remains embedded directly in the prompt as described in the project instructions)
