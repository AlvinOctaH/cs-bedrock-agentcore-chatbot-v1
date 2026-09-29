# 06 — Manual Scenarios, Evidence, Submission & Clean-up

## 1. Manual scenarios (`python chat.py`, one fresh session each)

| Scenario | Example message | Expected behaviour | Evidence |
|---|---|---|---|
| Lambda in isolation | Console test event | Ticket written | `02_lambda_test_result.png`, `03_dynamodb_item.png` |
| Bug report (multi-turn) | "Hi, something's broken on the checkout page." | Asks description → steps → environment one at a time, then `[tool call]` + real ticket ID | `04_chat_bug_report.png` |
| Ticket persisted | `aws dynamodb scan ...` | The ticket from the chat is in the table | `05_dynamodb_scan_from_chat.png` |
| FAQ covered | e.g. a shipping or returns question | Answer from the FAQ only | `06_chat_faq_covered.png` |
| FAQ not covered | e.g. price matching | Hand-off to 1-800-555-0199 | `07_chat_faq_uncovered.png` |
| Other request | e.g. a dinner recipe | Polite hand-off | `08_chat_other_request.png` |
| Automated evaluation | Bedrock Evaluations job | Correctness score per prompt | `09_bedrock_evaluation_results.png` |
| Guardrail (stand-out) | Guardrail console + injection attempt | Filters enabled; injection blocked | `10_guardrail_config.png`, `11_guardrail_blocked.png` |

All screenshots are in [`screenshots/`](../screenshots/).

## 2. Deliverables

| Deliverable | File |
|---|---|
| System prompt | [`starter/system_prompt.txt`](../starter/system_prompt.txt) |
| Test suite | [`starter/harness-tests.json`](../starter/harness-tests.json) |
| Evaluation dataset | [`starter/output_eval_dataset.jsonl`](../starter/output_eval_dataset.jsonl) |
| Written analysis | [`observations.md`](../observations.md) |
| Evidence | [`screenshots/`](../screenshots/) |

## 3. Submission checklist

- [x] System prompt routes all three categories
- [x] Bug reports: all three fields collected before the tool call; real ticket ID relayed
- [x] FAQ answers grounded; uncovered questions handed off
- [x] Test suite covering all three routes (+ 3 edge cases)
- [x] Bedrock Evaluations job run and analysed
- [x] Evidence screenshots

## 4. Clean-up

Empty the evaluation bucket **first** — CloudFormation cannot delete a non-empty
bucket, and the testing stack would end up in `DELETE_FAILED`:

```bash
cd starter
python cleanup_agentcore.py                       # harness, gateway target, gateway
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1
aws s3 rm s3://udacity-agentic-engineer-c1-eval-<account> --recursive
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
```

Also delete the Guardrail `bug-report-chatbot-guardrail` (Bedrock → Guardrails).
If the testing stack already shows `DELETE_FAILED`, empty the bucket and run its
`delete-stack` again.
