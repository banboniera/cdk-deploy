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

  - name: Setup CDK Environment
    uses: banboniera/setup-cdk@v1
    with:
      cdk-version: ${{ vars.CDK_VERSION }}
      aws-region: ${{ vars.AWS_REGION }}
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}

  - name: Synthesize CDK App
    run: npm run synth:staging

  - name: Prepare Deployment Artifacts
    run: cp package*.json ./cdk.out/

  - name: Upload Synthesized Template
    uses: actions/upload-artifact@v4
    with:
      name: cdk-synth
      path: ./cdk.out
      retention-days: 1

  - name: Deploy CDK Stack
    uses: banboniera/deploy-cdk-stack@v1
    with:
      cdk-version: '2.164.1'
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

## Example

```yaml
name: CDK Deploy Staging

on:
  workflow_dispatch: # Allows manual triggering of the workflow

env:
  APPLICATION: ${{ vars.APP_NAME }}-Staging

jobs:
  # Job 1
  build:
    name: Build and Synthesize
    runs-on: ubuntu-latest
    environment: STAGING
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    # Global Environment Variables:
    env:
      APP_NAME: ${{ vars.APP_NAME }}
      ENVIRONMENT: Staging
      DOMAIN_NAME: ${{ vars.DOMAIN_NAME }}
      VPS_IP: ${{ secrets.VPS_IP }}

    steps:
      # Step 1
      - name: Checkout Code
        uses: actions/checkout@v4

      # Step 2
      - name: Setup CDK Environment
        uses: banboniera/setup-cdk@v1
        with:
          cdk-version: ${{ vars.CDK_VERSION }}
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
          skip-dependencies: "false"

      # Step 3
      - name: Synthesize CDK App
        run: npm run synth:staging

      # Step 4
      - name: Prepare Deployment Artifacts
        run: cp package*.json ./cdk.out/

      # Step 5
      - name: Upload Synthesized Template
        uses: actions/upload-artifact@v4
        with:
          name: cdk-synth
          path: ./cdk.out
          retention-days: 1

  # Job 2
  deploy-zone:
    name: Deploy Zone Stack
    needs: [build]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-zone-${{ github.workflow }}
      cancel-in-progress: true

    steps:
      # Step 1
      - name: Deploy Stack
        uses: banboniera/deploy-cdk-stack@v1
        with:
          cdk-version: ${{ vars.CDK_VERSION }}
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
          stack-name: ${{ env.APPLICATION }}-Zone-Stack

  # Job 3
  deploy-state:
    name: Deploy State Stack
    needs: [build]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-state-${{ github.workflow }}
      cancel-in-progress: true

    steps:
      # Step 1
      - name: Deploy Stack
        uses: banboniera/deploy-cdk-stack@v1
        with:
          cdk-version: ${{ vars.CDK_VERSION }}
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
          stack-name: ${{ env.APPLICATION }}-State-Stack

  # Job 4
  deploy-security-resources:
    name: Deploy Security Resources Stack
    needs: [deploy-zone]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-security-resources-${{ github.workflow }}
      cancel-in-progress: true

    steps:
      # Step 1
      - name: Deploy Stack
        uses: banboniera/deploy-cdk-stack@v1
        with:
          cdk-version: ${{ vars.CDK_VERSION }}
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
          stack-name: ${{ env.APPLICATION }}-Security-Resources-Stack

  # Job 5
  deploy-static-site:
    name: Deploy Static Site Stack
    needs: [deploy-state, deploy-security-resources]
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-static-site-${{ github.workflow }}
      cancel-in-progress: true

    steps:
      # Step 1
      - name: Deploy Stack
        uses: banboniera/deploy-cdk-stack@v1
        with:
          cdk-version: ${{ vars.CDK_VERSION }}
          aws-region: ${{ vars.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_CDK }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_CDK }}
          stack-name: ${{ env.APPLICATION }}-Static-Site-Stack
```
