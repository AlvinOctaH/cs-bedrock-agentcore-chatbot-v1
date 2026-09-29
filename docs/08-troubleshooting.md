# 08 — Troubleshooting (problems actually hit)

## Setup & infrastructure

| Symptom | Cause | Fix |
|---|---|---|
| Harness fails / asks for a Marketplace subscription | The harness **default** model needs an AWS Marketplace subscription that lab accounts can't complete | Pin `us.amazon.nova-pro-v1:0` everywhere (already done in the scripts) |
| `setup_gateway.py` fails right after the stack finishes with a role access/validation error | IAM propagation delay | The script retries; if it still fails, run it again a minute later |
| Nova tool calling fails with *"Model produced invalid sequence as part of ToolUse"* | A dash in the gateway **target name** | Only letters, digits and underscores (`bugreports`) |
| Testing stack stuck in `DELETE_FAILED` | CloudFormation can't delete a non-empty S3 bucket | `aws s3 rm s3://<bucket> --recursive`, then `delete-stack` again |
| Expired lab credentials | Udacity lab sessions are temporary | Start a new session and reconfigure the AWS CLI |

## Prompt behaviour

| Symptom | Cause | What was done |
|---|---|---|
| No `[tool call]` line in `chat.py` | The prompt doesn't say clearly when to use the tool | Explicit checklist + "call the tool once you have all three" |
| Ticket filed with only a description (`t2`) or on a vague message (`t7`, sometimes `t8`) | Model bias toward completing the "helpful" action | Documented as a known weakness; production fix = validate in the Lambda (reject empty/placeholder fields) instead of relying only on the prompt |
| Invented ticket IDs | Model filling the gap when the tool result isn't used | "Only report the ticket ID the tool actually returns; if the tool fails, escalate manually" |
| A stricter negative instruction made things worse (model fabricated a fictitious prior conversation) | Heavy-handed negative instructions can backfire | Reverted to the simpler prompt that scored most stably |
| Same test, different result across runs | LLM non-determinism — observed even though `create_harness.py` uses greedy decoding (temperature 0, topK 1) | Treat one evaluation run as a sample (scores ~0.72–0.78); compare prompt versions over several runs |
| Replies contain `<thinking>…</thinking>` | Nova Pro's visible reasoning | Expected in `chat.py`; useful for debugging the classification step |

## Rubric wording

| Symptom | Explanation |
|---|---|
| The rubric asks for a Bedrock *Flow*, classifier node, condition nodes, `flow-tests.json` | Bedrock Agents Classic closed to new customers on 30 July 2026; the course moved to the AgentCore harness but the rubric text wasn't fully updated. See the terminology mapping in the README. |
