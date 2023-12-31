# remote-echidna

Run echidna on AWS

## Motivation

[Echidna](https://github.com/crytic/echidna) is a program designed for fuzzing/property-based testing of Ethereum smart contracts. Because of its high demand on computational resources, it is not very practical to run long runs on your local development environment. The purpose of remote-echidna is to allow developers to test their contracts on AWS while they can continue working on something else.

## Description

This project uses Terraform to create a virtual machine, together with other required AWS resources, for each echidna job. The instance starts up, install all required components, runs the fuzz tests, and automatically shuts itself down.

## Setup

### Manual steps:

1. Create a [S3 bucket](./terraform/s3_bucket.tf) with private access to store and load echidna's output between runs
2. Create a [private key](./terraform/ec2_instance.tf) used to connect to the remote instance if needed

### Automated steps:

This project creates the following infrastructure on AWS:

- [Security Group](./terraform/security_group.tf) with SSH access from anywhere, to allow for an easier debugging
- [IAM Policy](./terraform/iam_user.tf) with access to the S3 bucket
- [IAM User](./terraform/iam_user.tf) with the created IAM Policy
- [EC2 Instance](./terraform/ec2_instance.tf) that runs echidna on the desired git project and uses the IAM User credentials to upload results to S3

## Usage with GitHub Actions

Create a GitHub Action on your CI that reuses remote-echidna's [workflow](./.github/workflows/remote-echidna.yml). See a [sample project](https://github.com/pods-finance/yield-contracts) here

```
name: Pull Request

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  remote-echidna:
    name: Run remote-echidna
    uses: pods-finance/remote-echidna/.github/workflows/remote-echidna.yml@v2
    with:
      project: "https://github.com/${{github.repository}}.git"
      project_git_url: "https://github.com/${{github.repository}}.git"
      project_git_checkout: ${{ github.head_ref || github.ref_name }}
      run_tests_cmd: "yarn && echidna-test test/invariants/STETHVaultInvariants.sol --contract STETHVaultInvariants --config test/invariants/config-assertion.yaml --test-limit 100000"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      REMOTE_ECHIDNA_S3_BUCKET: ${{ secrets.REMOTE_ECHIDNA_S3_BUCKET }}
      REMOTE_ECHIDNA_EC2_INSTANCE_KEY_NAME: ${{ secrets.REMOTE_ECHIDNA_EC2_INSTANCE_KEY_NAME }}
```

You may also choose to include the following action to have the GitHub bot update your Pull Request with a comment monitoring the job status:

```
name: Update PR comments

on:
  workflow_dispatch:
  schedule:
    - cron: '* * * * *'

jobs:
  update-pr-comments:
    name: Update PR comments
    permissions:
      pull-requests: write
    uses: pods-finance/remote-echidna/.github/workflows/update-pr-comments.yml@v2
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      REMOTE_ECHIDNA_S3_BUCKET: ${{ secrets.REMOTE_ECHIDNA_S3_BUCKET }}
```

3. Configure the required input parameters below

### Inputs

| Parameter                              | Description                                                                  | Example                                                                                  | Required |
| -------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | -------- |
| `project`                              | Your project name                                                            | `yield-contracts`                                                                    | Yes      |
| `project_git_url`                      | Project Git URL                                                              | `https://github.com/${{github.repository}}.git`                                   | Yes      |
| `project_git_checkout`                 | Project Git checkout (branch or commit hash)                                 | `${{ github.head_ref \|\| github.ref_name }}`                                                                                   | Yes      |
| `run_tests_cmd`                        | Command to run echidna tests                                                 | `yarn && echidna-test test/invariants/STETHVaultInvariants.sol --contract STETHVaultInvariants --config test/invariants/config-assertion.yaml --test-limit 100000` | Yes      |
| `REMOTE_ECHIDNA_S3_BUCKET`             | S3 Bucket name to store and load echidna's output between runs               | `remote-echidna`                                                             | Yes      |
| `REMOTE_ECHIDNA_EC2_INSTANCE_KEY_NAME` | EC2 instance key name. Needs to be manually created first on the AWS console | `key`                                                                                | Yes      |

## Output

The job status is stored on a S3 bucket, alongside echidna's output:

0. Provisioning
1. Provisioned
2. Started
3. Running
4. Finished
5. Deprovisioned

Some of these states are managed from within the instance (2 -> 3 -> 4), while others are managed through Github actions (0 -> 1 -> 2, 4 -> 5).

Echidna's output will be automatically included on your pull request with as a GitHub bot comment.
