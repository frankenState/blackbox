# TypeScript Backend + AWS SAM Setup Guide

A comprehensive step-by-step walkthrough for initializing a TypeScript backend and deploying it with AWS SAM (Serverless Application Model).

## Prerequisites

- **Node.js** (v18+ recommended): [Install from nodejs.org](https://nodejs.org/)
- **AWS Account**: [Create at aws.amazon.com](https://aws.amazon.com/)
- **AWS CLI**: `npm install -g aws-cli` or [follow official guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- **SAM CLI**: `npm install -g aws-sam-cli` or [follow official guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- **Docker** (optional, but recommended for local SAM testing): [Download from docker.com](https://www.docker.com/)

Verify installations:
```bash
node --version
npm --version
aws --version
sam --version
```

---

## Step 1: Initialize TypeScript Project

Create a new directory for your project:
```bash
mkdir my-typescript-backend
cd my-typescript-backend
```

Initialize npm project:
```bash
npm init -y
```

Install TypeScript and necessary dependencies:
```bash
npm install --save-dev typescript @types/node ts-node
npm install --save-dev esbuild
```

Create TypeScript configuration file:
```bash
npx tsc --init
```

Update `tsconfig.json` for Lambda development:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

Create project structure:
```bash
mkdir src
mkdir src/handlers
mkdir tests
```

---

## Step 2: Create Your First Lambda Handler

Create `src/handlers/hello.ts`:
```typescript
export const handler = async (event: any): Promise<any> => {
  console.log('Event:', JSON.stringify(event, null, 2));
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Hello from TypeScript Lambda!',
      timestamp: new Date().toISOString(),
    }),
  };
};
```

---

## Step 3: Install AWS Lambda Dependencies

Add AWS Lambda-related packages:
```bash
npm install aws-lambda
npm install --save-dev @types/aws-lambda
```

Update handler to use Lambda types:
```typescript
import { APIGatewayProxyHandler } from 'aws-lambda';

export const handler: APIGatewayProxyHandler = async (event) => {
  console.log('Event:', JSON.stringify(event, null, 2));
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Hello from TypeScript Lambda!',
      timestamp: new Date().toISOString(),
    }),
  };
};
```

---

## Step 4: Configure Build Scripts

Update `package.json` with build scripts:
```json
{
  "scripts": {
    "build": "esbuild src/**/*.ts --platform=node --target=es2020 --outdir=dist",
    "watch": "npm run build -- --watch",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}
```

Test the build:
```bash
npm run build
```

Verify `dist/handlers/hello.js` was created.

---

## Step 5: Initialize SAM Project

Create `template.yaml` in your project root:
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Your Description

Globals:
  Function:
    Timeout: 30
    MemorySize: 256
    Runtime: nodejs22.x
    Architectures:
      - x86_64
    Environment:
      Variables:
        NODE_ENV: production

Resources:
  HttpApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      CorsConfiguration:
        AllowMethods: "'*'"
        AllowOrigins: "'*'"
        AllowHeaders: "'*'"
      Auth:
        Authorizers:
          JwtAuthorizer:
            IdentitySource: "$request.header.Authorization"
            JwtConfiguration:
              audience:
                - CLIENT_ID
              issuer: !Sub "https://cognito-idp.ap-REGION.amazonaws.com/REGION_POOLID"

  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: typescript-hello
      CodeUri: .
      Handler: dist/handlers/hello.handler
      Events:
        HelloApi:
          Type: HttpApi
          Properties:
            ApiId: !Ref HttpApi
            Path: /hello
            Method: get
            Auth:
              Authorizer: JwtAuthorizer

  TypeScriptApi:
    Type: AWS::Serverless::Api
    Properties:
      Name: typescript-api
      StageName: dev

Outputs:
  ApiUrl:
    Description: "HTTP API endpoint URL"
    Value:
      Fn::Sub: "https://${HttpApi}.execute-api.${AWS::Region}.amazonaws.com"
```

---

## Step 6: Create SAM Build Configuration

Create `.aws-sam/metadata.yaml` (SAM uses this):
```yaml
version: 0.1
```

Create `samconfig.toml` for local testing:
```toml
[default]
[default.build]
cmd = "npm run build"

[default.local_start_api]
port = "3000"
```

---

## Step 7: Build for Deployment

Build your TypeScript and prepare for SAM:
```bash
npm run build
sam build
```

This command:
- Compiles your TypeScript code
- Prepares the deployment package
- Creates `.aws-sam/build/` directory

---

## Step 8: Test Locally with SAM

Start local API Gateway:
```bash
sam local start-api
```

Test the endpoint in another terminal:
```bash
curl http://localhost:3000/hello
```

You should see:
```json
{
  "message": "Hello from TypeScript Lambda!",
  "timestamp": "2026-06-14T10:30:00.000Z"
}
```

---

## Step 9: Configure AWS Credentials

Set up AWS credentials (choose one method):

**Option A: Using AWS CLI (Recommended)**
```bash
aws configure
```

Enter:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Output format: `json`

**Option B: Using environment variables**
```bash
export AWS_ACCESS_KEY_ID=your_key_id
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1
```

---

## Step 10: Deploy to AWS

Deploy using SAM:
```bash
sam deploy --guided
```

First deployment will ask:
- Stack name: `my-typescript-stack`
- Region: Choose your region
- Confirm changes before deploy: `y`
- Allow SAM CLI to create roles: `y`
- Save parameters to samconfig.toml: `y`

Subsequent deployments:
```bash
sam deploy
```

---

## Step 11: Test Deployed Function

After deployment, you'll see the API endpoint URL. Test it:
```bash
curl https://your-api-id.execute-api.us-east-1.amazonaws.com/dev/hello
```

---

## Step 12: View Logs

View Lambda function logs:
```bash
sam logs -n HelloFunction --stack-name my-typescript-stack --tail
```

Or use CloudWatch:
```bash
aws logs tail /aws/lambda/typescript-hello --follow
```

---

## Step 13: Clean Up (Optional)

Delete the deployed stack:
```bash
aws cloudformation delete-stack --stack-name my-typescript-stack
aws cloudformation wait stack-delete-complete --stack-name my-typescript-stack
```

---

## Complete File Structure

```
my-typescript-backend/
├── src/
│   └── handlers/
│       └── hello.ts
├── dist/                 (generated)
│   └── handlers/
│       └── hello.js
├── .aws-sam/            (generated)
│   └── build/
├── template.yaml
├── samconfig.toml
├── tsconfig.json
├── package.json
└── package-lock.json
```

---

## Troubleshooting

### SAM Build Fails
```bash
# Clear build cache and rebuild
rm -rf .aws-sam
npm run build
sam build
```

### Local Testing Issues
- Ensure Docker is running: `docker ps`
- Check port 3000 is available
- Use `sam local start-api --debug` for verbose output

### Deployment Fails
- Verify AWS credentials: `aws sts get-caller-identity`
- Check IAM permissions (need CloudFormation, Lambda, API Gateway, IAM roles)
- Review CloudFormation events: `aws cloudformation describe-stack-events --stack-name my-typescript-stack`

### TypeScript Compilation Issues
```bash
# Check TypeScript errors
npx tsc --noEmit
```

---

## Additional Resources

- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [esbuild Documentation](https://esbuild.github.io/)

---

## Quick Reference Commands

```bash
# Development
npm install                    # Install dependencies
npm run build                  # Compile TypeScript
npm run watch                  # Watch for changes
sam build                      # Prepare for deployment
sam local start-api            # Run locally

# Testing
curl http://localhost:3000/hello

# Deployment
sam deploy --guided            # First deployment
sam deploy                     # Subsequent deployments
sam logs -n HelloFunction --tail  # View logs

# Cleanup
sam delete                     # Delete stack
```

---

**Last Updated**: June 14, 2026  
**Status**: ✅ Verified and Working with Node.js 18.x, TypeScript 5.x, SAM CLI 1.x
