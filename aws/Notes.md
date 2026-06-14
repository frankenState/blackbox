# AWS Package Installation for TypeScript / Node

This guide provides step-by-step instructions on how to test an API Gateway that uses Lambda functions and DynamoDB.

## Amazon S3
```bash
npm install @aws-sdk/client-s3
```

## Amazon DynamoDB
```bash
npm install @aws-sdk/client-dynamodb
```

## Amazon SQS
```bash
npm install @aws-sdk/client-sqs
```

## Deploy to Different Environments
```bash
sam deploy \
  --stack-name your-stack-name \
  --parameter-overrides Environment=staging \
  --guided
```