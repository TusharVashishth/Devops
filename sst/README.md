# SST — Serverless Stack Quick Guide

This document gives a concise overview of using SST (Serverless Stack) in this project, including a sample config, deployment workflow, prerequisites, and helpful commands.

## Overview

- **Project**: `landing-page` (example SST app shown below)
- **AWS region** used in the example: `ap-south-1`

> Ensure you have AWS CLI configured locally with credentials that have the necessary IAM permissions to create the resources used by SST.

## SST Config Example

The following `sst` config demonstrates creating a VPC, ECS cluster, and an ECS service with a load balancer.

```ts
// eslint-disable-next-line @typescript-eslint/triple-slash-reference
/// <reference path="./.sst/platform/config.d.ts" />

export default $config({
  app(input) {
    return {
      name: "landing-page",
      removal: input?.stage === "production" ? "retain" : "remove",
      home: "aws",
      providers: {
        aws: { region: "ap-south-1" },
      },
    };
  },
  async run() {
    const vpc = new sst.aws.Vpc("MyVpc", { bastion: false, nat: "managed" });
    const cluster = new sst.aws.Cluster("MyCluster", { vpc });

    const service = new sst.aws.Service("MyService", {
      cluster,
      cpu: "0.25 vCPU",
      memory: "0.5 GB",
      storage: "20 GB",
      scaling: { min: 1, max: 10, cpuUtilization: 70, memoryUtilization: 70 },
      loadBalancer: { ports: [{ listen: "80/http", forward: "3000/http" }] },
      environment: { NODE_ENV: "production" },
      dev: { command: "npm run dev" },
    });

    return { url: service.url };
  },
});
```

## Prerequisites

- `node` >= 18 (example used Node 20 in CI)
- `npm` or `pnpm` for installing dependencies
- AWS credentials configured locally (`aws configure`) or via environment variables
- SST installed locally or run via `npx sst`

## GitHub Actions: Deploy to AWS (example)

Use this workflow to automatically deploy to AWS when you push to `main`.

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Deploy with SST
        run: npx sst deploy --stage production
```

## Useful Commands

- Deploy locally using SST (dev):

```bash
npx sst dev
```

- Deploy to a stage (example `production`):

```bash
npx sst deploy --stage production
```

- If SST cannot find the ECR repository, create it manually (replace `repo-name`):

```bash
aws ecr create-repository --repository-name repo-name --region ap-south-1
```

## Troubleshooting & Notes

- Confirm AWS credentials and region: `aws sts get-caller-identity` and `aws configure list`.
- If IAM permissions are insufficient, the deployment will fail when creating VPC/Cluster/ECR/ECS resources—request appropriate policies from your AWS admin.
- SST may create or reference resources (ECR repos, VPCs, load balancers); check the CloudFormation/SST logs if something is missing.

## References

- SST docs: https://docs.sst.dev
- AWS CLI: https://docs.aws.amazon.com/cli/

---

Updated for clarity and structured guidance. If you want, I can also:

- add a short `Getting Started` section with exact install commands, or
- create the GitHub Actions workflow file in `.github/workflows/deploy.yml`.
