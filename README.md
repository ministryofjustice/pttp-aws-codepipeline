# PTTP CI / CD

This repository holds the Terraform code to create a [CodeBuild](https://aws.amazon.com/codebuild/) / [CodePipeline](https://aws.amazon.com/codepipeline/) service in AWS.

## How to use this repo

The source code in this repository is provided only as a reference.
Please speak to someone on the PTTP team to get a pipeline set up.

This pipeline will be integrated with a Github repository, and build your project according to your [buildspec](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html) files.

Depending on your build process, you may require 3 files to do linting, testing and deployment.

### Linting

If you are doing static code analysis as part of your build, please create a `buildspec.lint.yml` file, and place it in the root of your project.

example:

```yaml
version: 0.2

phases:
  install:
    commands:
      - make lint
```

### Testing

To run automated tests, create a `buildspec.test.yml` file, and place it in the root of your project.

example:

```yaml
version: 0.2

phases:
  install:
    commands:
      - make test
```

### Deployment

For deployments, create a `buildspec.yml` file.

example:

```yaml
version: 0.2

env:
  variables:
    key: "value"
    key: "value"

phases:
  install:
    commands:
      - pip install boto3
      - wget https://releases.hashicorp.com/terraform/0.12.24/terraform_0.12.24_linux_amd64.zip
      - unzip terraform_0.12.24_linux_amd64.zip
      - mv terraform /bin
      - terraform init
  build:
    commands:
      - terraform apply --auto-approve
```

## To create your own Pipeline

For experimenting with AWS CodePipeline / CodeBuild, you can execute the Terraform in this repository.

An OAuth token is required to pull your source code from Github.

Create an access token for your repository and add it to a .tfvars file.

```shell script
github_oauth_token = "abc123"
```

Run Terraform with the variables file:

```shell script
terraform apply -var-file=".tfvars"
```

## Deploying across environments

CodePipeline can be used to centralise deployment initiation across multiple environments to a single location. This is the recommended approach for moving artefacts up through environments.

### Terminology

- The AWS account that allows another account to perform actions on it is referred to as the "trusting account".
- The AWS account assumes a role and performs actions on a trusting account is referred to as the "trusted account".

### Diagram

![CodePipeline Cross-Environment Deployments](/documentation/images/code-pipeline-cross-environment-deployments.png)

In this diagram:

- Deployments to each trusting account are initiated from CodePipeline in a Shared Services AWS account.
- The Terraform state for each trusting account is held in a single S3 bucket in the Shared Services account.
- The Shared Services CodePipeline deploys to trusting accounts by assuming a role in the trusting account account using the [AWS Security Token Service](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html).

### Getting Started

In order to deploy to multiple environments using this method you will need:

- A CodeBuild pipeline in the PTTP Shared Services AWS account.
- An IAM role in each trusting account with sufficient priviledges to run Terraform scripts on that trusting account.
- A policy on each of these roles which allows the Shared Services AWS account to assume the role (see `Example A` below).
- A policy on your CodeBuild to tell it to assume a role in the trusting account (see `Example B` below).

### Policy On Trusting Account (Example A)

This example policy would be created on an IAM role in a trusting account.

It allows the user `root` in an AWS trusted account (which has an AWS account ID of `555555555555`) to assume a role on the trusting account.

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::555555555555:root" },
    "Action": "sts:AssumeRole"
  }
}
```

### Assume Role On CodeBuild (Example B)

This IAM role policy snippet tells the CodeBuild pipeline on the trusted account to assume a role named `production-deployment-role` on the trusting account (which has an AWS account ID of `999999999999`).

```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::999999999999:role/production-deployment-role"
}
```

### Getting Help

As mentioned early in this document, please speak to someone on the PTTP team if you need help getting your pipelines and related Security Token Service configuration set up.
