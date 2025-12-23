# AIOps Agent - AWS Bedrock Intelligent Operations Assistant

An intelligent AWS operations assistant powered by Amazon Bedrock that provides natural language interface for monitoring CloudTrail events and managing EC2 instances with built-in safety confirmations.

## 🎓 Course Information

This repository is part of the learning materials for the Udemy course:
**[AI Fundamentals for Beginners: Learn LLM, Agentic AI & MCP](https://www.udemy.com/course/ai-fundamentals-for-beginners-learn-llm-agentic-ai-mcp/?referralCode=4D49F0BFDF7A68F7CF22)**

The course covers AI fundamentals, Large Language Models (LLMs), Agentic AI systems, and Model Context Protocol (MCP) implementations.

## 🚀 What This Agent Does

### 📊 Monitoring Capabilities (READ-ONLY)
- **CloudTrail Events**: Query AWS CloudTrail for activity monitoring
- **User Activity**: Track actions by specific users
- **Service Events**: Monitor events from specific AWS services
- **Recent Activity**: Get real-time operational insights

### ⚙️ Management Capabilities (WRITE with Safety)
- **EC2 Operations**: Start, stop, and reboot EC2 instances
- **Instance Status**: Get detailed instance information
- **Safety Confirmations**: Requires explicit approval for destructive actions
- **Instance Listing**: View all instances with filtering options

## 🏗️ Architecture

The agent consists of:
- **2 Lambda Functions**: CloudTrail queries and EC2 operations
- **Bedrock Agent**: Natural language processing and orchestration
- **IAM Roles**: Secure permissions for each component
- **OpenAPI Schemas**: Define available operations and parameters

## 🎯 Key Features

- **Natural Language Interface**: Ask questions in plain English
- **Safety First**: Confirmation required for instance operations
- **Intelligent Caching**: 60-second cache for CloudTrail queries
- **Cost Optimized**: ~$2/month for light usage
- **Scalable**: Serverless architecture with automatic scaling

## 📋 Prerequisites

- AWS Account with administrator access
- AWS CLI installed and configured
- CloudTrail enabled in your account
- At least one EC2 instance for testing
- Bedrock model access enabled (Claude 3 Sonnet/Haiku)

## AWS Authentication Setup

The application uses AWS Bedrock, so you need to configure AWS credentials. **AWS SSO is the recommended method** for secure authentication.

### Method 1: AWS SSO Configuration (Recommended)

AWS Single Sign-On (SSO) provides secure, temporary credentials and is the preferred authentication method.

#### Step 1: Configure AWS SSO
```bash
aws configure sso
```

Follow the prompts to configure your SSO profile.

#### Step 2: Login with SSO
```bash
aws sso login --profile your-profile-name
```

#### Optional: Make profile default (bash/zsh)
```bash
export AWS_PROFILE=your-profile-name
```

On Windows PowerShell, use:
```powershell
$env:AWS_PROFILE = 'your-profile-name'
```

#### Verify authentication
```bash
aws sts get-caller-identity
aws bedrock list-foundation-models --region us-east-1
```

### Method 2: AWS CLI Configuration (Alternative)
```bash
aws configure
```

### Method 3: Environment Variables (Development Only)

Linux/macOS:
```bash
export AWS_ACCESS_KEY_ID=your_access_key_here
export AWS_SECRET_ACCESS_KEY=your_secret_key_here
export AWS_DEFAULT_REGION=us-east-1
```

Windows PowerShell:
```powershell
$env:AWS_ACCESS_KEY_ID = 'your_access_key_here'
$env:AWS_SECRET_ACCESS_KEY = 'your_secret_key_here'
$env:AWS_DEFAULT_REGION = 'us-east-1'
```

### Refreshing Expired SSO Credentials

If you see `ExpiredTokenException` errors, refresh your SSO login:
```bash
aws sso login
```

## 🚀 Quick Start

1. **Deploy Infrastructure**:
   ```bash
   aws cloudformation create-stack \
     --stack-name aiops-agent \
     --template-body file://aiops-agent.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --region us-east-1
   ```

2. **Create Bedrock Agent**: Follow the detailed steps in `complete_deployment_guide.md`

3. **Test the Agent**:
   ```
   "Show me recent CloudTrail events"
   "List all running EC2 instances"
   "Stop instance i-xxxxx"
   ```

## 📖 Documentation

- **[Complete Deployment Guide](complete_deployment_guide.md)**: Step-by-step setup instructions
- **[CloudFormation Template](aiops-agent.yaml)**: Infrastructure as Code

## 🔐 Security Features

- **Least Privilege**: IAM roles with minimal required permissions
- **Confirmation Required**: Explicit approval for destructive operations
- **Audit Trail**: All actions logged in CloudTrail
- **Resource Validation**: Validates instance IDs and states before operations

## 💡 Example Interactions

```
User: "Show me who stopped instances today"
Agent: "I found 2 StopInstances events today:
        - john.doe stopped i-abc123 at 10:30 AM
        - admin stopped i-def456 at 2:15 PM"

User: "Stop instance i-abc123"
Agent: "Instance Details:
        - ID: i-abc123
        - Name: web-server
        - State: running
        
        This will shut down the instance. Shall I proceed?"

User: "Yes"
Agent: "✓ Instance i-abc123 is now stopping."
```

## 🛠️ Customization

The agent can be extended with additional capabilities:
- **RDS Management**: Add database operations
- **S3 Monitoring**: Track bucket activities
- **Cost Analysis**: Integrate with Cost Explorer
- **Alerting**: Add SNS notifications

## 📊 Cost Estimation

**Monthly cost for light usage (~1,000 requests):**
- Lambda: ~$0.25
- Bedrock tokens: ~$1.50
- **Total: ~$2/month**

## 🤝 Contributing

This is a learning project from the Udemy course. Feel free to:
- Report issues
- Suggest improvements
- Share your customizations
- Ask questions about the implementation

## 📚 Learn More

Enroll in the full course to learn:
- AI and LLM fundamentals
- Building agentic AI systems
- Model Context Protocol (MCP)
- Advanced AI integration patterns

**[Get the Course](https://www.udemy.com/course/ai-fundamentals-for-beginners-learn-llm-agentic-ai-mcp/?referralCode=4D49F0BFDF7A68F7CF22)**

## 📄 License

This project is part of educational content. See course materials for usage terms.
