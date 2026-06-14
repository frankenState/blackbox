# API Gateway Testing with Lambda and DynamoDB

This guide provides step-by-step instructions on how to test an API Gateway that uses Lambda functions and DynamoDB.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Testing with Postman](#testing-with-postman)
4. [Testing with AWS CLI](#testing-with-aws-cli)
5. [Testing with Cognito Authentication](#testing-with-cognito-authentication)
6. [Testing DynamoDB Queries](#testing-dynamodb-queries)
7. [Debugging and Troubleshooting](#debugging-and-troubleshooting)

---

## Prerequisites

- AWS CLI installed and configured with appropriate credentials
- Postman or similar API testing tool
- AWS Account with API Gateway, Lambda, and DynamoDB resources deployed
- Cognito User Pool (if using authentication)
- Basic understanding of AWS services

### Install AWS CLI
```bash
# For Windows
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# For macOS
brew install awscli

# For Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Configure AWS CLI
```bash
aws configure
# You will be prompted for:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region
# - Default output format (json)
```

---

## Initial Setup

### Step 1: Identify Your API Gateway Resources

1. Go to AWS Console → API Gateway
2. Note down:
   - API Gateway ID
   - Stage name (e.g., dev, prod)
   - API endpoint URL
   - Resource paths
   - HTTP methods (GET, POST, PUT, DELETE)

Example API endpoint:
```
https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev
```

### Step 2: Identify Lambda Function

1. Go to AWS Console → Lambda
2. Find your Lambda function connected to the API Gateway
3. Note the function name and ARN
4. Verify IAM role has DynamoDB permissions

### Step 3: Identify DynamoDB Table

1. Go to AWS Console → DynamoDB
2. Note the table name
3. Verify table has:
   - Partition key defined
   - Sort key (if applicable)
   - Correct read/write capacity or on-demand billing

---

## Testing with Postman

### Step 1: Set Up Postman Collections

1. Open Postman
2. Create a new collection: `API Gateway Tests`
3. Create a new environment: `AWS Dev Environment`

### Step 2: Add Environment Variables

In your Postman environment, add:
```
{
  "api_endpoint": "https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev",
  "resource_path": "/users",
  "auth_token": "",
  "user_id": "test-user-123"
}
```

### Step 3: Create Test Requests

#### GET Request Example
```
Method: GET
URL: {{api_endpoint}}{{resource_path}}/{{user_id}}
Headers:
  - Content-Type: application/json
  - Authorization: Bearer {{auth_token}}
```

#### POST Request Example
```
Method: POST
URL: {{api_endpoint}}{{resource_path}}
Headers:
  - Content-Type: application/json
  - Authorization: Bearer {{auth_token}}

Body (raw JSON):
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}
```

#### PUT Request Example
```
Method: PUT
URL: {{api_endpoint}}{{resource_path}}/{{user_id}}
Headers:
  - Content-Type: application/json
  - Authorization: Bearer {{auth_token}}

Body (raw JSON):
{
  "name": "Jane Doe",
  "age": 31
}
```

#### DELETE Request Example
```
Method: DELETE
URL: {{api_endpoint}}{{resource_path}}/{{user_id}}
Headers:
  - Authorization: Bearer {{auth_token}}
```

---

## Testing with AWS CLI

### Step 1: Get API Gateway Details

```bash
# List all API Gateways
aws apigateway get-rest-apis

# Get specific API details
aws apigateway get-rest-api --rest-api-id abc123xyz

# Get resources
aws apigateway get-resources --rest-api-id abc123xyz

# Get stages
aws apigateway get-stages --rest-api-id abc123xyz
```

### Step 2: Test API Endpoints Using curl

#### GET Request
```bash
curl -X GET https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users/user-123 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### POST Request
```bash
curl -X POST https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30
  }'
```

#### PUT Request
```bash
curl -X PUT https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users/user-123 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "Jane Doe",
    "age": 31
  }'
```

#### DELETE Request
```bash
curl -X DELETE https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users/user-123 \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Step 3: Inspect Lambda Logs

```bash
# List log streams
aws logs describe-log-streams \
  --log-group-name /aws/lambda/your-function-name

# Get log events
aws logs get-log-events \
  --log-group-name /aws/lambda/your-function-name \
  --log-stream-name 'stream-name' \
  --start-time $(date -d '10 minutes ago' +%s)000
```

---

## Testing with Cognito Authentication

### Step 1: Create Cognito User Pool and App Client

```bash
# Create User Pool
aws cognito-idp create-user-pool \
  --pool-name my-user-pool

# Create App Client
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_xxxxxxxxx \
  --client-name my-app-client \
  --explicit-auth-flows ADMIN_NO_SRP_AUTH USER_PASSWORD_AUTH
```

### Step 2: Create a Test User

```bash
aws cognito-idp admin-create-user \
  --user-pool-id us-east-1_xxxxxxxxx \
  --username test@example.com \
  --message-action SUPPRESS \
  --temporary-password TempPassword123!
```

### Step 3: Set Permanent Password

```bash
aws cognito-idp admin-set-user-password \
  --user-pool-id us-east-1_xxxxxxxxx \
  --username test@example.com \
  --password FinalPassword123! \
  --permanent
```

### Step 4: Authenticate Using USER_PASSWORD_AUTH Flow

```bash
aws cognito-idp initiate-auth \
  --auth-flow USER_PASSWORD_AUTH \
  --client-id YOUR_APP_CLIENT_ID \
  --auth-parameters USERNAME=test@example.com,PASSWORD=FinalPassword123! \
  --region us-east-1
```

**Response:**
```json
{
  "AuthenticationResult": {
    "AccessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "IdToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "RefreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "ExpiresIn": 3600,
    "TokenType": "Bearer"
  }
}
```

### Step 5: Use Access Token for API Calls

Extract the `AccessToken` from the response and use it in your API calls:

```bash
# Store token in variable
ACCESS_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Make authenticated API call
curl -X GET https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Step 6: Refresh Token When Expired

```bash
# Refresh the access token
aws cognito-idp admin-initiate-auth \
  --auth-flow REFRESH_TOKEN_AUTH \
  --client-id YOUR_APP_CLIENT_ID \
  --auth-parameters REFRESH_TOKEN=YOUR_REFRESH_TOKEN \
  --region us-east-1
```

---

## Testing DynamoDB Queries

### Step 1: Direct DynamoDB Query via AWS CLI

```bash
# Get item from DynamoDB
aws dynamodb get-item \
  --table-name users-table \
  --key '{"userId":{"S":"user-123"}}' \
  --region us-east-1
```

### Step 2: Scan Table

```bash
# Scan all items
aws dynamodb scan \
  --table-name users-table \
  --region us-east-1

# Scan with filter
aws dynamodb scan \
  --table-name users-table \
  --filter-expression "age > :val" \
  --expression-attribute-values '{":val":{"N":"25"}}' \
  --region us-east-1
```

### Step 3: Query with Partition Key

```bash
aws dynamodb query \
  --table-name users-table \
  --key-condition-expression "userId = :uid" \
  --expression-attribute-values '{":uid":{"S":"user-123"}}' \
  --region us-east-1
```

### Step 4: Put Item to DynamoDB

```bash
aws dynamodb put-item \
  --table-name users-table \
  --item '{
    "userId": {"S": "user-456"},
    "name": {"S": "Alice Smith"},
    "email": {"S": "alice@example.com"},
    "age": {"N": "28"}
  }' \
  --region us-east-1
```

### Step 5: Update Item in DynamoDB

```bash
aws dynamodb update-item \
  --table-name users-table \
  --key '{"userId":{"S":"user-456"}}' \
  --update-expression "SET #n = :val" \
  --expression-attribute-names '{"#n":"name"}' \
  --expression-attribute-values '{":val":{"S":"Alice Johnson"}}' \
  --region us-east-1
```

---

## Debugging and Troubleshooting

### Step 1: Enable API Gateway Logging

```bash
# Create IAM role for API Gateway logging
aws iam create-role \
  --role-name api-gateway-logging-role \
  --assume-role-policy-document file://trust-policy.json

# Attach policy
aws iam attach-role-policy \
  --role-name api-gateway-logging-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# Enable logging on API Gateway
aws apigateway update-stage \
  --rest-api-id abc123xyz \
  --stage-name dev \
  --patch-operations \
    op=replace,path=/*/*/logging/loglevel,value=INFO \
    op=replace,path=/*/*/logging/dataFullyLoggingEnabled,value=true \
    op=replace,path=/accessLogSetting/destinationArn,value=arn:aws:logs:us-east-1:ACCOUNT_ID:log-group:api-gateway-logs
```

### Step 2: Check CloudWatch Logs

```bash
# Get Lambda function logs
aws logs tail /aws/lambda/your-function-name --follow

# Get API Gateway logs
aws logs tail api-gateway-logs --follow

# Get DynamoDB logs (if enabled)
aws logs tail /aws/dynamodb/your-table-name --follow
```

### Step 3: Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| 403 Forbidden | Check API key, authorization header, and IAM permissions |
| 404 Not Found | Verify resource path and HTTP method in API Gateway |
| 500 Internal Server Error | Check Lambda function logs in CloudWatch |
| DynamoDB throttling | Increase read/write capacity or enable on-demand billing |
| Cognito auth failed | Verify user exists, password is correct, and auth flow is enabled |
| CORS errors | Enable CORS in API Gateway and set appropriate headers |

### Step 4: Test Lambda Locally (Optional)

```bash
# Using AWS SAM
sam local start-api

# Using serverless framework
serverless offline start

# Using docker (for Node.js)
docker run -p 9000:8080 -v ~/my-lambda:/var/task:ro,delegated public.ecr.aws/lambda/nodejs:18 index.handler
```

### Step 5: Performance Testing

```bash
# Using Apache Bench
ab -n 1000 -c 10 -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users

# Using wrk
wrk -t12 -c400 -d30s -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/users
```

---

## Quick Reference Checklist

- [ ] AWS CLI configured with credentials
- [ ] API Gateway endpoint URL identified
- [ ] Lambda function permissions verified
- [ ] DynamoDB table exists with correct schema
- [ ] Cognito User Pool and App Client created
- [ ] Test user created in Cognito
- [ ] API Gateway has CORS enabled (if needed)
- [ ] CloudWatch logging enabled for debugging
- [ ] Authentication token obtained
- [ ] Test requests validated in Postman or curl
- [ ] DynamoDB queries tested directly
- [ ] Lambda logs checked for errors
- [ ] Performance baseline established

---

## Additional Resources

- [AWS API Gateway Documentation](https://docs.aws.amazon.com/apigateway/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [AWS DynamoDB Documentation](https://docs.aws.amazon.com/dynamodb/)
- [AWS Cognito Documentation](https://docs.aws.amazon.com/cognito/)
- [AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/)
