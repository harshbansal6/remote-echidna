# remote-echidna

Run echidna on AWS with Terraform

## Description

This project creates an instance, together with other AWS resources, for each echidna job. The instance starts up, install all required components, runs the fuzz tests, and automatically shuts down itself afterwards.

In order to guarantee that all provisioned resources are deleted after the job, the job state is stored on a S3 bucket. A job can go through some stages:

1. Provisioned
2. Started
3. Running
4. Finished
5. Deprovisioned

Some of these states are managed from within the instance (2 -> 3 -> 4), while others are managed through Github actions (1 -> 2, 4 -> 5).

## Setup

Manual steps:

1. Create a [S3 bucket](./terraform/s3_bucket.tf) with private access to store and load echidna's output between runs
2. Create a [private key](./terraform/ec2_instance.tf) used to connect to the remote instance and download it to your local machine

This project creates the following infrastructure on AWS:

- [Security Group](./terraform/security_group.tf) with SSH access from anywhere, to allow for an easier debugging
- [IAM Policy](./terraform/iam_user.tf) with access to the S3 bucket
- [IAM User](./terraform/iam_user.tf) with the created IAM Policy
- [EC2 Instance](./terraform/ec2_instance.tf) that runs echidna on the desired git project and uses the IAM User credentials to upload results to S3

## Usage with Terraform Cloud

1. Add this project as a dependency of the repository you want to test. The easiest way is with a git module

```
git submodule add https://github.com/aviggiano/remote-echidna.git
```

2. Create a [Workspace](https://app.terraform.io/app/YOUR_ORG/workspaces/new) with `Version control workflow` on Terraform Cloud and link your Github project

3. Add the the parameters below to your [Workspace variables](https://app.terraform.io/app/YOUR_ORG/workspaces/YOUR_WORKSPACE/variables) on Terraform Cloud, including `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as environment variables

4. Configure your [Working Directory](https://app.terraform.io/app/YOUR_ORG/workspaces/YOUR_WORKSPACE/settings/general) as `remote-echidna`

5. Set `Include submodules on clone` on the [Version Control](https://app.terraform.io/app/YOUR_ORG/workspaces/YOUR_WORKSPACE/settings/version-control) settings

### Inputs

| Parameter               | Description                                                                  | Example                                                                                          | Required |
| ----------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | -------- |
| `ec2_instance_key_name` | EC2 instance key name. Needs to be manually created first on the AWS console | `key.pem`                                                                                        | Yes      |
| `project`               | Your project name                                                            | `smart-contracts`                                                                                | Yes      |
| `project_git_url`       | Project Git URL                                                              | `https://github.com/aviggiano/smart-contracts.git`                                               | Yes      |
| `project_git_checkout`  | Project Git checkout (branch or commit hash)                                 | `main`                                                                                           | Yes      |
| `s3_bucket`             | S3 Bucket name to store and load echidna's output between runs               | `remote-echidna-bucket`                                                                          | Yes      |
| `run_tests_cmd`         | Command to run echidna tests                                                 | `yarn && echidna-test test/Contract.sol --contract Contract --config test/config.yaml \|\| true` | Yes      |

### Example usage

```
steps:
- uses: hashicorp/setup-terraform@v2
  with:
    cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
- name: Terraform Init
  run: terraform init
- name: Terraform Apply
  env:
    project: 'smart-contracts'
    project_git_url: 'https://github.com/${{github.repository}}.git'
    project_git_checkout: ${{ github.head_ref || github.ref_name }}
    s3_bucket: 'remote-echidna'
    ec2_instance_key_name: 'key.pem'
    run_tests_cmd: 'yarn && echidna-test test/Contract.sol --contract Contract --config test/config.yaml'
  run: |
    terraform init
    terraform apply -var="ec2_instance_key_name=${{ env.ec2_instance_key_name }}" -var="project=${{ env.project }}" -var="project_git_url=${{ env.project_git_url }}" -var="project_git_checkout=${{ env.project_git_checkout }}" -var="run_tests_cmd=${{ env.run_tests_cmd }}" -var="s3_bucket=${{ env.s3_bucket }}" -no-color -input=false -auto-approve
```

## Development

#### 1. Create a `tfvars` file

Include the parameters required by [vars.tf](./terraform/vars.tf)

```
# vars.tfvars

project               = "echidna-project"
project_git_url       = "https://github.com/aviggiano/echidna-project.git"
project_git_checkout  = "main"
ec2_instance_key_name = "key.pem"
s3_bucket             = "remote-echidna"
run_tests_cmd         = "yarn && echidna-test test/Contract.sol --contract Contract --config test/config.yaml || true"
```

### 2. Run terraform

```
terraform apply -var-file vars.tfvars
```

## Next steps

- [ ] Improve state management to avoid conflicting runs from multiple people
- [ ] Perform cleanup of terraform state after the job finishes
- [ ] Create AMI with all required software instead of [installing everything](./terraform/user_data.tftpl) at each time (would speed up about 1min)
