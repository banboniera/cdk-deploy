# Deploy CDK Stack Action

A GitHub composite action that handles AWS CDK stack deployment with validation and detailed reporting.

## Features

- 🚀 Automated CDK stack deployment
- 🔍 Pre-deployment stack validation with diff checking
- ⏱️ Deployment timing and duration tracking
- 📊 Detailed deployment summary with JSON outputs
- 🔄 Status reporting and error handling
- ⏲️ Configurable deployment timeout
- 🔐 AWS IAM role assumption support
- 🎯 Concurrent deployment support with unique execution IDs

## Usage

```yaml
steps:
  - name: Prepare CDK Environment
    id: prepare-cdk
    uses: banboniera/cdk-prepare@v2
    with:
      aws-region: ${{ vars.AWS_REGION }}
      role-to-assume: ${{ env.CDK_ROLE_ARN }}
      synth-command: npm run synth:staging

  - name: Deploy CDK Stack
    uses: banboniera/cdk-deploy-stack@v1
    with:
      stack-name: 'my-stack-name'
      aws-region: ${{ vars.AWS_REGION }}
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      artifact-name: 'cdk-deployment-package'
      artifact-path: 'deployment'
      node-version: '22'
      timeout-seconds: 1800
      skip-validation: 'false'
      execution-id: ${{ github.run_id }}-${{ github.run_number }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `stack-name` | The name of the stack to deploy | true | |
| `aws-region` | The AWS region to deploy the stack to | true | |
| `role-to-assume` | AWS IAM role ARN to assume | true | |
| `artifact-name` | Name for the deployment artifact | false | `cdk-deployment-package` |
| `artifact-path` | Path to store deployment files | false | `deployment` |
| `node-version` | The version of Node.js to use | false | `22` |
| `timeout-seconds` | Timeout duration for stack deployment in seconds | false | `1800` |
| `skip-validation` | Skip stack validation before deployment | false | `false` |
| `execution-id` | Unique identifier for concurrent deployments | false | `run_id-run_number` |

## Outputs

| Output | Description |
|--------|-------------|
| `deployment-status` | The status of the deployment (success/failure) |
| `failure-reason` | Error message if deployment failed |
| `stack-outputs` | JSON containing stack outputs after deployment |

## Deployment Summary

The action provides a detailed deployment summary including:

- Stack name and region
- Deployment status with visual indicators
- Deployment duration in human-readable format
- JSON outputs from the CDK deployment (if any)
- Failure reason if deployment failed

## Error Handling

The action includes comprehensive error handling:

- Pre-deployment validation with `cdk diff`
- Timeout monitoring and reporting
- Detailed error messages with exit codes
- Proper status propagation to GitHub Actions
- Concurrent deployment support with unique execution IDs

## Example

```yaml
name: CDK Deploy Staging

on:
  workflow_dispatch: # Allows manual triggering of the workflow

env:
  APPLICATION: ${{ vars.APP_NAME }}-Staging

jobs:
  build-synth:
    name: Build and Synthesize
    runs-on: ubuntu-latest
    environment: staging
    concurrency:
      group: build-synth-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 5

    env:
      APP_NAME: ${{ vars.APP_NAME }}
      ENVIRONMENT: staging
      DOMAIN_NAME: ${{ vars.ORG_NAME }}.${{ vars.TLD_STAGING }}
      VPS_IP: ${{ secrets.VPS_IP }}
      VPS_IP_PORTAINER: ${{ secrets.VPS_IP_PORTAINER }}
      CDK_ROLE_ARN: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ vars.APP_NAME }}-Role-CDK

    outputs:
      stacks: ${{ steps.prepare-cdk.outputs.stacks }}
      dependencies: ${{ steps.prepare-cdk.outputs.dependencies }}

    steps:
      - name: Prepare CDK Environment
        id: prepare-cdk
        uses: banboniera/cdk-prepare@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}
          synth-command: npm run synth:staging

  deploy-zone-public:
    name: Deploy Public Zone Stack
    needs: [build]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-zone-public-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 3

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-PublicHostedZone-Stack
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}

  deploy-certificate-waf:
    name: Deploy Certificate WAF Stack
    needs: [deploy-zone-public]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-certificate-waf-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-Certificate-Waf-Stack
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}

  deploy-ecr-repositories:
    name: Deploy ECR Repositories Stack
    needs: [build]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-ecr-repositories-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 4

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-ECRRepositories-Stack
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}

  deploy-static-site:
    name: Deploy Static Site Stack
    needs: [deploy-ecr-repositories, deploy-certificate-waf]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-static-site-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 10

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy-stack@v1
        with:
          stack-name: ${{ env.APPLICATION }}-Static-Site-Stack
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}
```
