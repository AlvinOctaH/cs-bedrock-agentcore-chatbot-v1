# 02 — The Bug Report Tool and the AgentCore Gateway

When a customer reports a bug the chatbot must store it where engineering can follow
up. A **DynamoDB table** is the ticket store, a **Lambda function** is the tool, and an
**AgentCore Gateway** presents the Lambda to the model as a callable tool.

## 1. Deploy the tool stack

```bash
cd starter
aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

`CAPABILITY_NAMED_IAM` is required because the template creates named IAM roles.

Stack outputs:

| Output | Used for |
|---|---|
| `BugReportsTableName` / `BugReportsTableArn` | The ticket table (`bug-report-tool-stack-bug-reports`) |
| `LambdaFunctionArn` / `LambdaExecutionRoleArn` | The `create_bug_report` function |
| `GatewayRoleArn` | Lets the gateway invoke the Lambda |
| `HarnessExecutionRoleArn` | Lets the harness call Bedrock models and the gateway |

## 2. How the Lambda works (`create_bug_report.py`)

- The **tool arguments arrive directly as the Lambda event**:
  `{"description": "...", "stepsToReproduce": "...", "environment": "..."}` — no
  `messageVersion`/`parameters` envelope (that was Bedrock Agents Classic).
- The **tool name** arrives in the client context:
  `context.client_context.custom["bedrockAgentCoreToolName"]`, namespaced as
  `<targetName>___<toolName>` (three underscores).
- It writes the ticket to DynamoDB and **returns** a result containing the
  `ticketId` — whatever it returns goes back to the model.
- It prints the raw `EVENT:` to CloudWatch Logs (`/aws/lambda/bug-report-tool-stack-create-bug-report`)
  — the ground truth for what actually reached the tool.

### Test the Lambda in isolation (before involving the model)

Lambda console → the function → **Test** with:

```json
{ "description": "Checkout page is blank", "stepsToReproduce": "Add item, checkout, Pay Now",
  "environment": "Chrome on Windows 11" }
```

Then confirm the item exists:

```bash
aws dynamodb scan --table-name bug-report-tool-stack-bug-reports --region us-east-1
```

Evidence: `screenshots/02_lambda_test_result.png`, `screenshots/03_dynamodb_item.png`.

## 3. Create the gateway and register the tool

```bash
python setup_gateway.py
```

The script:
1. reads the stack outputs itself (no copy-paste);
2. creates an **AgentCore Gateway** (MCP protocol, `AWS_IAM` auth);
3. registers the Lambda as target **`bugreports`** with one tool
   `create_bug_report(description, stepsToReproduce, environment)` — all three required;
4. saves IDs/ARNs to `agentcore_config.json`.

The model sees the tool as **`bugreports___create_bug_report`**.

> Gateway **target names** may contain only letters, digits and underscores. A dash
> breaks Nova tool calling with *"Model produced invalid sequence as part of ToolUse"*.

> If it fails right after the stack finishes with a role access/validation error,
> that is **IAM propagation delay** — the script retries; if it still fails, run it
> again a minute later.
