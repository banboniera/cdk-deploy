# CDK Deploy Action

A GitHub composite action that handles AWS CDK stack deployment with validation and detailed reporting.

## Features

- 🔄 Support for both individual stack deployments and full stack deployments
- 🔍 Optional pre-deployment stack validation with diff checking
- 📊 Detailed deployment summary with JSON outputs
- 🔐 AWS IAM role assumption support
- 📦 Flexible artifact handling with customizable paths

## Usage

```yml
steps:
  - name: CDK Deploy Stack
    uses: banboniera/cdk-deploy@v2
    with:
      aws-region: 'eu-central-1'
      role-to-assume: 'arn:aws:iam::123456789012:role/github-actions-role'
      stack-name: 'my-stack-name'  # optional, if not provided destroys all stacks
      artifact-name: 'cdk-deployment-package'  # optional
      artifact-path: 'deployment'  # optional
      node-version: '22'  # optional
      timeout-seconds: 1800  # optional
      skip-validation: 'false'  # optional
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `stack-name` | Name of the CDK stack to deploy | false | |
| `aws-region` | Target AWS region for deployment | true | |
| `role-to-assume` | AWS IAM role ARN to assume | true | |
| `artifact-name` | Name for the deployment artifact | false | `cdk-deployment-package` |
| `artifact-path` | Path to store deployment files | false | `deployment` |
| `node-version` | Node.js version to use | false | `22` |
| `timeout-seconds` | Maximum duration for deployment in seconds | false | `1800` |
| `skip-validation` | Skip stack validation before deployment | false | `true` |

## Deployment Process

1. Downloads the pre-synthesized CDK template from artifacts
2. Sets up Node.js environment with specified version
3. Configures AWS credentials using role assumption
4. Optionally validates the stack using `cdk diff`
5. Deploys the stack with timeout protection
6. Generates a detailed deployment summary including:
   - Deployment status
   - Stack outputs
   - Error details (if any)

## Example Workflow

```yml
jobs:
  build-synth:
    name: Build and Synthesize
    runs-on: ubuntu-latest

    steps:
      - name: Prepare CDK Environment
        uses: banboniera/cdk-prepare@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ vars.APP_NAME }}-Role-CDK

  deploy:
    name: CDK Deploy All Stacks
    needs: [build-synth]
    runs-on: ubuntu-latest

    steps:
      - name: CDK Deploy All Stacks
        uses: banboniera/cdk-deploy@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ vars.APP_NAME }}-Role-CDK
```

## Real Example Workflow

```yml
name: CDK Deploy Staging

on:
  workflow_call:
    secrets:
      AWS_ACCOUNT_ID:
        description: 'AWS Account ID'
        required: true
      VPS_IP:
        description: 'IP address of the VPS'
        required: true
      VPS_IP_PORTAINER:
        description: 'IP address of the VPS Portainer'
        required: true

permissions:
  contents: read
  id-token: write

env:
  APPLICATION: ${{ vars.APP_NAME }}-Staging
  CDK_ROLE_ARN: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/${{ vars.APP_NAME }}-Role-CDK

jobs:
  build-synth:
    name: Build and Synthesize
    runs-on: ubuntu-latest
    environment: staging
    concurrency:
      group: build-synth-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 1

    env:
      APP_NAME: ${{ vars.APP_NAME }}
      ENVIRONMENT: staging
      DOMAIN_NAME: ${{ vars.ORG_NAME }}.${{ vars.TLD }}
      VPS_IP: ${{ secrets.VPS_IP }}
      VPS_IP_PORTAINER: ${{ secrets.VPS_IP_PORTAINER }}

    steps:
      - name: Prepare CDK Environment
        uses: banboniera/cdk-prepare@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}
          synth-command: npm run synth:staging

  zone-public:
    name: Deploy Public Zone Stack
    needs: [build-synth]
    runs-on: ubuntu-latest
    concurrency:
      group: zone-public-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 3

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}
          stack-name: ${{ env.APPLICATION }}-PublicHostedZone-Stack

  certificate-waf:
    name: Deploy Certificate WAF Stack
    needs: [zone-public]
    runs-on: ubuntu-latest
    concurrency:
      group: certificate-waf-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}
          stack-name: ${{ env.APPLICATION }}-Certificate-Waf-Stack

  static-site:
    name: Deploy Static Site Stack
    needs: [certificate-waf]
    runs-on: ubuntu-latest
    concurrency:
      group: static-site-${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true
    timeout-minutes: 14

    steps:
      - name: Deploy Stack
        uses: banboniera/cdk-deploy@v2
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ env.CDK_ROLE_ARN }}
          stack-name: ${{ env.APPLICATION }}-Static-Site-Stack
```

## Notes

- The action requires a pre-synthesized CDK template to be available as an artifact
- Stack validation is disabled by default (`skip-validation: true`)
- Deployment timeout is set to 30 minutes by default
- Detailed deployment results are available in the GitHub Actions step summary
