# AIOps Agent - Complete Deployment Guide

## 🎯 What You're Building

An intelligent AWS operations assistant powered by Bedrock that can:
- ✅ **Monitor**: Query CloudTrail events (READ-ONLY)
- ✅ **Operate**: Manage EC2 instances - start, stop, reboot (WRITE with safety)
- ✅ **Converse**: Natural language interface with safety confirmations

---

## 📋 Prerequisites Checklist

### ☑️ Required
- [ ] AWS Account with administrator access
- [ ] AWS CLI installed (`aws --version`)
- [ ] AWS CLI configured (`aws configure`)
- [ ] CloudTrail enabled in your account
- [ ] At least one EC2 instance for testing
- [ ] Bedrock model access enabled

### ☑️ Check Your Setup

```bash
# 1. Verify AWS CLI
aws --version
# Should show: aws-cli/2.x.x or higher

# 2. Verify credentials
aws sts get-caller-identity
# Should show your Account ID and User ARN

# 3. Check CloudTrail
aws cloudtrail describe-trails --region us-east-1
# Should list at least one trail

# 4. Check EC2 instances
aws ec2 describe-instances --region us-east-1 --query 'Reservations[].Instances[].InstanceId'
# Should list your instance IDs
```

### ☑️ Enable Bedrock Model Access

**If you haven't used Bedrock before:**

1. Go to [AWS Bedrock Console](https://console.aws.amazon.com/bedrock)
2. Click **"Model access"** in left sidebar
3. Click **"Modify model access"**
4. Check these models:
   - ✅ Claude 3 Haiku
   - ✅ Claude 3 Sonnet
   - ✅ Claude Sonnet 4
5. Click **"Request model access"**
6. Wait for approval (usually instant ⚡)

---

## 🚀 Deployment Method: CloudFormation

We'll use CloudFormation to deploy all infrastructure automatically.

### Step 1: Download Deployment Files

Create a project directory:

```bash
mkdir aiops-agent-deployment
cd aiops-agent-deployment
```

### Step 2: Save CloudFormation Template

Save the CloudFormation template from the artifact above as `aiops-agent.yaml`

Or download directly:
```bash
# Save the CloudFormation template to aiops-agent.yaml
# (Copy the YAML content from the artifact above)
```

### Step 3: Deploy the Stack

```bash
# Deploy with default settings
aws cloudformation create-stack \
  --stack-name aiops-agent \
  --template-body file://aiops-agent.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Wait for stack creation (takes ~3 minutes)
aws cloudformation wait stack-create-complete \
  --stack-name aiops-agent \
  --region us-east-1

echo "✅ Stack created successfully!"
```

**What this creates:**
- ✅ 2 Lambda functions (CloudTrail + EC2)
- ✅ 3 IAM roles (CloudTrail Lambda, EC2 Lambda, Bedrock Agent)
- ✅ Lambda permissions for Bedrock
- ✅ All necessary policies

### Step 4: Get Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name aiops-agent \
  --region us-east-1 \
  --query 'Stacks[0].Outputs'
```

**Save these values - you'll need them:**
- `CloudTrailLambdaArn`
- `EC2LambdaArn`
- `BedrockAgentRoleArn`

---

## 📝 Step 5: Create OpenAPI Schemas

### 5.1: CloudTrail OpenAPI Schema

Save this as `cloudtrail-openapi.json`:

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "CloudTrail Events API",
    "version": "1.0.0",
    "description": "API for querying AWS CloudTrail events"
  },
  "paths": {
    "/events/lookup": {
      "get": {
        "summary": "Lookup CloudTrail events",
        "description": "Search CloudTrail events with filters",
        "operationId": "lookupEvents",
        "parameters": [
          {"name": "event_name", "in": "query", "description": "Event name filter", "required": false, "schema": {"type": "string"}},
          {"name": "username", "in": "query", "description": "Username filter", "required": false, "schema": {"type": "string"}},
          {"name": "hours", "in": "query", "description": "Hours to look back", "required": false, "schema": {"type": "integer", "default": 24}},
          {"name": "max_results", "in": "query", "description": "Max results", "required": false, "schema": {"type": "integer", "default": 5}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/events/by-user": {
      "get": {
        "summary": "Get events by user",
        "description": "Get CloudTrail events for specific user",
        "operationId": "getEventsByUser",
        "parameters": [
          {"name": "username", "in": "query", "description": "Username", "required": true, "schema": {"type": "string"}},
          {"name": "hours", "in": "query", "description": "Hours to look back", "required": false, "schema": {"type": "integer", "default": 24}},
          {"name": "max_results", "in": "query", "description": "Max results", "required": false, "schema": {"type": "integer", "default": 5}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/events/by-service": {
      "get": {
        "summary": "Get events by service",
        "description": "Get CloudTrail events for specific AWS service",
        "operationId": "getEventsByService",
        "parameters": [
          {"name": "service", "in": "query", "description": "AWS service name", "required": true, "schema": {"type": "string"}},
          {"name": "hours", "in": "query", "description": "Hours to look back", "required": false, "schema": {"type": "integer", "default": 12}},
          {"name": "max_results", "in": "query", "description": "Max results", "required": false, "schema": {"type": "integer", "default": 5}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/events/recent": {
      "get": {
        "summary": "Get recent events",
        "description": "Get most recent CloudTrail events",
        "operationId": "getRecentEvents",
        "parameters": [
          {"name": "hours", "in": "query", "description": "Hours to look back", "required": false, "schema": {"type": "integer", "default": 1}},
          {"name": "max_results", "in": "query", "description": "Max results", "required": false, "schema": {"type": "integer", "default": 5}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    }
  }
}
```

### 5.2: EC2 OpenAPI Schema

Save this as `ec2-openapi.json`:

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "EC2 Operations API",
    "version": "1.0.0",
    "description": "API for managing EC2 instances"
  },
  "paths": {
    "/instances/stop": {
      "post": {
        "summary": "Stop an EC2 instance",
        "description": "Stops a running EC2 instance gracefully",
        "operationId": "stopInstance",
        "parameters": [
          {"name": "instance_id", "in": "query", "description": "EC2 instance ID (i-xxxxx)", "required": true, "schema": {"type": "string"}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/instances/start": {
      "post": {
        "summary": "Start an EC2 instance",
        "description": "Starts a stopped EC2 instance",
        "operationId": "startInstance",
        "parameters": [
          {"name": "instance_id", "in": "query", "description": "EC2 instance ID (i-xxxxx)", "required": true, "schema": {"type": "string"}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/instances/reboot": {
      "post": {
        "summary": "Reboot an EC2 instance",
        "description": "Reboots a running EC2 instance",
        "operationId": "rebootInstance",
        "parameters": [
          {"name": "instance_id", "in": "query", "description": "EC2 instance ID (i-xxxxx)", "required": true, "schema": {"type": "string"}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/instances/status": {
      "get": {
        "summary": "Get instance status",
        "description": "Get detailed status of an EC2 instance",
        "operationId": "getInstanceStatus",
        "parameters": [
          {"name": "instance_id", "in": "query", "description": "EC2 instance ID (i-xxxxx)", "required": true, "schema": {"type": "string"}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    },
    "/instances/list": {
      "get": {
        "summary": "List EC2 instances",
        "description": "List instances with optional state filtering",
        "operationId": "listInstances",
        "parameters": [
          {"name": "state", "in": "query", "description": "Filter by state", "required": false, "schema": {"type": "string", "enum": ["all", "running", "stopped"], "default": "all"}},
          {"name": "max_results", "in": "query", "description": "Max results", "required": false, "schema": {"type": "integer", "default": 10}}
        ],
        "responses": {"200": {"description": "Success"}}
      }
    }
  }
}
```

---

## 🤖 Step 6: Create Bedrock Agent (Console)

Now we'll create the agent in AWS Console.

### 6.1: Navigate to Bedrock

1. Go to [AWS Bedrock Console](https://console.aws.amazon.com/bedrock)
2. Click **"Agents"** in left sidebar
3. Click **"Create Agent"** button

### 6.2: Agent Configuration

#### Basic Information
- **Agent name**: `AIOpsAgent`
- **Description**: `AWS operations assistant for CloudTrail monitoring and EC2 management`
- **User input**: ✅ Enable (checked)

Click **Next**

#### Select Foundation Model
- **Model**: Select **`Anthropic Claude 3 Sonnet`** or **`Claude Sonnet 4`**
- **Instructions for the Agent**: Paste this:

```
You are an AWS operations assistant with both monitoring and management capabilities.

CAPABILITIES:
1. MONITORING (CloudTrail - READ-ONLY): Query CloudTrail events, track activities, investigate incidents
2. MANAGEMENT (EC2 - WRITE): Stop, start, reboot EC2 instances, get status, list instances

SAFETY RULES FOR EC2 OPERATIONS:
- ALWAYS show instance details (ID, name, type, state) BEFORE taking action
- ALWAYS ask for explicit confirmation: "Shall I proceed?"
- Wait for confirmation: "yes", "proceed", or "confirm"
- Never assume confirmation from casual language
- If instance ID missing, ask for it specifically
- After operations, confirm success and show new state

RESPONSE STYLE:
- Present CloudTrail data in readable format
- Use clear confirmation prompts for EC2 operations
- Be concise and helpful
- Explain impacts of operations

EXAMPLE:
User: "Stop instance i-abc123"
You: "Instance Details:
- ID: i-abc123
- Name: web-server
- Type: t2.micro
- State: running

This will shut down the instance. Shall I proceed?"

User: "Yes"
You: "✓ Instance i-abc123 is now stopping."
```

Click **Next**

### 6.3: Add Action Group 1 - CloudTrail

1. Click **"Add"** in Action Groups section
2. **Action group name**: `CloudTrailQueries`
3. **Action group type**: Select **"Define with API schemas"**
4. **Action group invocation**: Select **"Select an existing Lambda function"**
5. **Lambda function**: Select `AIOpsAgent-cloudtrail-query`
6. **API Schema**: 
   - Select **"Upload API schema"**
   - Click **"Choose file"**
   - Upload `cloudtrail-openapi.json`
7. Click **"Add action group"**

### 6.4: Add Action Group 2 - EC2 Operations

1. Click **"Add"** again in Action Groups section
2. **Action group name**: `EC2Operations`
3. **Action group type**: Select **"Define with API schemas"**
4. **Action group invocation**: Select **"Select an existing Lambda function"**
5. **Lambda function**: Select `AIOpsAgent-ec2-operations`
6. **API Schema**: 
   - Select **"Upload API schema"**
   - Click **"Choose file"**
   - Upload `ec2-openapi.json`
7. Click **"Add action group"**

### 6.5: Advanced Settings (Optional)

- **Session timeout**: 5 minutes (default is fine)
- **Guardrails**: None (or configure if needed)
- **Knowledge bases**: None (not needed for this use case)

Click **Next**

### 6.6: Review and Create

1. Review all settings
2. Click **"Create Agent"**
3. Wait for creation (~30 seconds)

---

## 🎉 Step 7: Prepare and Test

### 7.1: Prepare the Agent

**CRITICAL STEP - Don't skip!**

1. On the agent details page, look for the **"Prepare"** button at the top right
2. Click **"Prepare"**
3. Wait 1-2 minutes for preparation to complete
4. Status will change from "Not prepared" to "Prepared" ✅

### 7.2: Test the Agent

In the **Test** panel on the right side of the agent page:

#### Test 1: CloudTrail Query (Read-Only)
```
Show me recent CloudTrail events from the last hour
```

**Expected Response:**
```
I found X CloudTrail events in the last hour:
- Event 1: [details]
- Event 2: [details]
...
```

#### Test 2: List EC2 Instances
```
List all running EC2 instances
```

**Expected Response:**
```
You have X running instances:
1. i-abc123 - [name] (t2.micro)
2. i-def456 - [name] (t2.small)
...
```

#### Test 3: Get Instance Status
```
What's the status of instance i-xxxxx?
```
*(Replace i-xxxxx with your actual instance ID)*

**Expected Response:**
```
Instance i-xxxxx:
- Name: [name]
- State: running
- Type: t2.micro
- Private IP: 10.x.x.x
```

#### Test 4: EC2 Operation with Confirmation (WRITE)
```
Stop instance i-xxxxx
```
*(Replace with a TEST instance - not production!)*

**Expected Response:**
```
Instance Details:
- ID: i-xxxxx
- Name: [name]
- Type: t2.micro
- State: running

This will shut down the instance. Shall I proceed?
```

**Then respond:**
```
Yes
```

**Expected Response:**
```
✓ Instance i-xxxxx is now stopping.
```

#### Test 5: Combined Workflow
```
Show me who stopped instances today, then show me the status of i-xxxxx
```

**Expected Response:**
```
I found 2 StopInstances events today:
- john.doe stopped i-abc at 10:30 AM
- admin stopped i-def at 2:15 PM

Instance i-xxxxx status:
- Name: [name]
- State: stopped
- Type: t2.micro
```

---

## 📊 Step 8: Monitor Your Agent

### CloudWatch Logs

View Lambda execution logs:

```bash
# CloudTrail Lambda logs
aws logs tail /aws/lambda/AIOpsAgent-cloudtrail-query --follow --region us-east-1

# EC2 Lambda logs
aws logs tail /aws/lambda/AIOpsAgent-ec2-operations --follow --region us-east-1
```

### View Agent Trace

In the Bedrock console test panel:
1. Click **"Show trace"** toggle
2. See the agent's reasoning process:
   - What tool it chose
   - What parameters it extracted
   - What results it got

---

## 🎨 Step 9: Create an Alias (Optional but Recommended)

Create a stable endpoint for your agent:

1. In agent details, go to **"Aliases"** tab
2. Click **"Create alias"**
3. **Alias name**: `production` or `v1`
4. **Description**: `Production version of AIOps Agent`
5. Click **"Create alias"**

Now you can invoke the agent via API using the alias:

```python
import boto3

bedrock_agent_runtime = boto3.client('bedrock-agent-runtime')

response = bedrock_agent_runtime.invoke_agent(
    agentId='YOUR_AGENT_ID',
    agentAliasId='YOUR_ALIAS_ID',
    sessionId='unique-session-id',
    inputText='List running instances'
)
```

---

## 🔧 Troubleshooting

### Issue: "Agent not prepared"
**Solution**: Click the "Prepare" button and wait for completion

### Issue: "Permission denied" errors
**Solution**: 
```bash
# Verify stack created properly
aws cloudformation describe-stacks --stack-name aiops-agent --region us-east-1

# Check Lambda permissions
aws lambda get-policy --function-name AIOpsAgent-cloudtrail-query --region us-east-1
```

### Issue: "No CloudTrail events found"
**Solution**: CloudTrail may not have recent events. Try:
```
Show me CloudTrail events from the last 24 hours
```

### Issue: "Instance not found"
**Solution**: Verify instance exists:
```bash
aws ec2 describe-instances --region us-east-1
```

### Issue: Lambda timeout
**Solution**: Default is 60 seconds. If needed, increase:
```bash
aws lambda update-function-configuration \
  --function-name AIOpsAgent-cloudtrail-query \
  --timeout 120 \
  --region us-east-1
```

---

## 🔐 Security Best Practices

### 1. Limit EC2 Permissions (Recommended)

Edit the CloudFormation template to add conditions:

```yaml
EC2OperationsPolicy:
  PolicyDocument:
    Statement:
      - Effect: Allow
        Action:
          - ec2:StopInstances
          - ec2:StartInstances
        Resource: '*'
        Condition:
          StringEquals:
            ec2:ResourceTag/ManagedBy: 'AIOpsAgent'
```

Then tag your instances:
```bash
aws ec2 create-tags \
  --resources i-xxxxx \
  --tags Key=ManagedBy,Value=AIOpsAgent \
  --region us-east-1
```

### 2. Enable CloudTrail for Agent Actions

The agent's actions are logged, but for compliance:
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=AIOpsAgent \
  --region us-east-1
```

### 3. Add Guardrails (Optional)

In Bedrock Agent console:
1. Go to **Guardrails**
2. Create guardrail with:
   - Content filters (block harmful content)
   - Denied topics (e.g., "production database operations")
   - PII filtering
3. Apply to your agent

---

## 🚀 Next Steps

### Integration Options

#### 1. Slack Bot
```python
from slack_bolt import App
import boto3

bedrock = boto3.client('bedrock-agent-runtime')

@app.command("/aiops")
def handle_command(ack, command, say):
    ack()
    response = bedrock.invoke_agent(
        agentId='YOUR_AGENT_ID',
        agentAliasId='YOUR_ALIAS_ID',
        sessionId=command['user_id'],
        inputText=command['text']
    )
    say(response['output']['text'])
```

#### 2. CLI Tool
```bash
# Create a simple CLI wrapper
cat > aiops-cli.sh << 'EOF'
#!/bin/bash
aws bedrock-agent-runtime invoke-agent \
  --agent-id YOUR_AGENT_ID \
  --agent-alias-id YOUR_ALIAS_ID \
  --session-id $(uuidgen) \
  --input-text "$1"
EOF

chmod +x aiops-cli.sh
./aiops-cli.sh "List running instances"
```

#### 3. Web Dashboard
Use the Bedrock Agent Runtime API to build a React/Vue web interface

---

## 💰 Cost Estimation

### Monthly Cost (Light Usage)
- **Lambda invocations**: 1,000 requests = ~$0.20
- **Lambda compute**: 1,000 requests × 1s × 256MB = ~$0.05
- **Bedrock Agent**: ~$0 (pay per token)
- **Bedrock Claude tokens**: 100K input + 50K output = ~$1.50
- **CloudTrail**: First copy free
- **Total**: **~$2/month** for light usage

### Optimization Tips
- ✅ Results are cached for 60 seconds
- ✅ Default result limit is 5 events (saves tokens)
- ✅ Lambda cold starts minimized with keep-warm

---

## 📚 Additional Resources

- [AWS Bedrock Agents Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html)
- [CloudTrail API Reference](https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/)
- [EC2 API Reference](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/)
- [OpenAPI Specification](https://swagger.io/specification/)

---

## 🎯 Summary

You now have a fully functional AIOps Agent that:
- ✅ Monitors AWS activity via CloudTrail
- ✅ Manages EC2 instances safely
- ✅ Uses natural language
- ✅ Requires confirmation for dangerous operations
- ✅ Scales automatically
- ✅ Costs ~$2/month for light usage

**Enjoy your new AI operations assistant!** 🎉
