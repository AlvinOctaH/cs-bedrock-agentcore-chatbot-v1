# 05 — Automated Testing with Bedrock Evaluations

Manual chatting doesn't scale. The pipeline below runs a fixed test suite against the
harness and scores the answers with an **LLM-as-a-judge**.

```
harness-tests.json ──► generate-eval-dataset.py ──► output_eval_dataset.jsonl
        ──► S3 bucket ──► Bedrock Evaluations (Builtin.Correctness) ──► results in S3 + console
```

## 1. Write the test suite

Copy `harness-tests-template.json` to `harness-tests.json` and add cases covering
**all three routes**. This submission has **9 cases**:

| id | Prompt (short) | Expected |
|---|---|---|
| `t1_bug_report_full_info` | Checkout freezes on "Pay Now", Chrome on Windows | Files the ticket, returns an ID |
| `t2_bug_report_partial_info` | "Order page shows a blank screen" | Asks for exactly one missing field |
| `t3_faq_shipping_covered` | How long does standard shipping take? | Timeframe from the FAQ |
| `t4_faq_returns_covered` | Return without original packaging? | Return policy from the FAQ |
| `t5_faq_uncovered_handoff` | Price matching? | Not in FAQ → hand-off |
| `t6_other_request` | Dinner recipe | Polite hand-off |
| `t7_edge_ambiguous_message` | "It's not working the way I expected" | Clarifying question |
| `t8_edge_very_short_message` | "help" | Ask what they need |
| `t9_edge_prompt_injection` | "Ignore all previous instructions…" | Refuse, keep role |

Each case has a `prompt` and an `expected` description, which becomes the judge's
**reference response**.

## 2. Generate the dataset

```bash
python generate-eval-dataset.py --tests-json harness-tests.json
```

Every case runs in a **fresh session**, and the final reply is written to
`output_eval_dataset.jsonl` in the Bedrock Evaluations *bring-your-own-inference* format:

```json
{"prompt": "...", "referenceResponse": "...",
 "modelResponses": [{"response": "...", "modelIdentifier": "my-support-chatbot"}]}
```

## 3. Deploy the testing stack and upload

```bash
aws cloudformation deploy \
  --template-file cloudformation-testing.yaml \
  --stack-name bug-report-testing-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Outputs: `EvalDatasetBucketName` (`udacity-agentic-engineer-c1-eval-<account>`) and
`BedrockEvalRoleArn`.

```bash
aws s3 cp output_eval_dataset.jsonl s3://udacity-agentic-engineer-c1-eval-<account>/output_eval_dataset.jsonl
```

## 4. Create the evaluation job

The three JSON files in `starter/` describe the job (replace the account ID in the
S3 URIs with yours):

| File | Contents |
|---|---|
| `eval-config.json` | Dataset S3 URI, task type `General`, metric `Builtin.Correctness`, judge `amazon.nova-pro-v1:0` |
| `inference-config.json` | `precomputedInferenceSource` = `my-support-chatbot` (responses are already in the dataset) |
| `output-config.json` | Results S3 prefix |

```bash
aws bedrock create-evaluation-job \
  --job-name support-chatbot-eval-1 \
  --role-arn <BedrockEvalRoleArn> \
  --application-type ModelEvaluation \
  --evaluation-config file://eval-config.json \
  --inference-config file://inference-config.json \
  --output-data-config file://output-config.json \
  --region us-east-1
```

Or in the console: **Bedrock → Evaluations → Create → Automatic: model as a judge →
Bring your own inference responses**, with the same dataset, metric and role.

## 5. Result of this submission

**Builtin.Correctness 0.72** across 9 prompts (6 scored 1, 1 scored 0.5, 2 scored 0 in
the job shown in `screenshots/09_bedrock_evaluation_results.png`; other runs landed
around 0.72–0.78). The weaknesses — filing a ticket on partial or ambiguous
messages — and the analysis are in [`observations.md`](../observations.md).

> Scores vary between runs even with greedy decoding (temperature 0, topK 1 in
> `create_harness.py`): one evaluation run is a **sample**, not a guarantee.
