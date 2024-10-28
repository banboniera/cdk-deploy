# Deploy CDK Stack Action

A GitHub composite action that handles AWS CDK stack deployment with validation and detailed reporting.

## Features

- 🚀 Automated CDK stack deployment
- 🔍 Pre-deployment stack validation
- ⏱️ Deployment timing and duration tracking
- 📊 Detailed deployment summary
- 🔄 Status reporting and error handling
- 📁 Supports custom working directory

## Usage

```yaml
steps:
  - name: Checkout Code
    uses: actions/checkout@v4

  # First, synthesize your CDK app and save the output
  - name: Synthesize CDK App
    run: npx cdk synth

  # Save the synthesized template as an artifact
  - name: Upload Synthesized Template
    uses: actions/upload-artifact@v4
    with:
      name: cdk-synth
      path: cdk.out/
      retention-days: 1

  # Deploy the stack
  - name: Deploy CDK Stack
    uses: banboniera/deploy-cdk-stack@v1
    with:
      cdk-version: '2.161.0'
      aws-region: 'eu-central-1'
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      stack-name: 'my-stack-name'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node-version` | The version of Node.js to use | false | `22` |
| `cdk-version` | The version of AWS CDK to use | true | |
| `aws-region` | The AWS region to deploy the stack to | true | |
| `aws-access-key-id` | The AWS access key ID | true | |
| `aws-secret-access-key` | The AWS secret access key | true | |
| `working-directory` | Working directory for npm commands | false | `.` |
| `stack-name` | The name of the stack to deploy | true | |

## Outputs

| Output | Description |
|--------|-------------|
| `deployment-status` | The status of the deployment (success/failure) |
| `start-time` | The start time of the deployment |
| `end-time` | The end time of the deployment |
