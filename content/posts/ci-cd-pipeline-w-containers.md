+++
date = '2024-12-01T15:23:08+01:00'
draft = true
title = 'A GitLab pipeline with containers to achieve faster pipelines'
+++

## Motivation

Continuos Integration and Continuos Deployment have become a necessary requirement in most of our projects. We can get rid of many manual processes by building a pipeline in **GitLab**, or similar tool, which runs through the usual steps: **test**, **build**, **deploy**. However, these steps can involve redundant actions, for example, the **test** steps needs to install the necessary dependencies of the project to run, while the **build** process might also need to install the same dependencies. In the case of `node_modules`, these can be easily shared between environments, but might not be as easy for `python` packages or other dependencies that are not as easily shared between pipeline steps or that require a large list of dependencies to be installed first in the execution environment.

In this experiment, blog entry, we show a way to share execution environments between pipeline jobs in the form of container images that can be used by the pipeline. We try to show that by doing this, we can save some time at the expense of space.

## Problem

In the problem I am presenting in this blog entry, we have a `python` project where we are managing __code dependencies__ using `poetry`. Then, we build an **artifact** that __manifests__ our project in the execution environment. Then, we deploy our code into the AWS infrastructure of choice using `Terraform`.

## Solution

The basic stages of the pipeline will be:

- build images
- unit test
- build package
- merge request
- cleanup
- deploy

The **build images** stage runs on a `docker` image. It requires AWS credentials to upload to `AWS ECR` the images it generates. This stage will implement two jobs, one job generates the **app container** and the other generates the **terraform container**.

The **unit test** stage needs a `python3.12` image with all the code dependencies installed. We need AWS credentials in the execution environment to pull the **test container** generated in the **build images** stage from `AWS ECR`.

The **build package** stage needs the `python3.12` image with all code dependencies installed. We need the necessary AWS credentials in the execution environment to pull the **test container** generated in the **build images** stage from `AWS ECR`.

The **merge request** stage needs an image with `terraform` installed. We choose a `node:20` image as base. We need the necessary credentials in the execution environment to pull the **terraform container** generated in the **build images** stage from `AWS ECR`.

The **cleanup** stage needs an image with `terraform` installed. It also needs **AWS** credentials to retrieve an image from `AWS ECR`.

The **deploy production** stage needs and execution environment with `terraform` installed. It also needs **AWS** credentials to retrieve an image from `AWS ECR`.

We need to build the images that the different steps of the pipeline need. The job of building these images is also going to become a step in the pipeline. Building images also implies that we need some repository where we are going to store them, and accessing some remote repository implies credentials.

Since we are already using **AWS** to deploy our project infrastructure, we can use **AWS ECR** to store our tagged images. Consequently, the **AWS** credentials that we are going to use in our pipeline also need the necessary permissions to push and retrieve images from **ECR**. Properly speaking, the **IAM Role** that our credentials represent should've been granted the necessary policies to operate on **ECR** on the target **AWS Account**.

First, let's define the variables that our pipeline will need:

```yaml
variables:
  AWS_ACCOUNT_ID: "<Your_Account_Id>"
  AWS_ECR_REGISTRY: "${AWS_ACCOUNT_ID}.dkr.ecr.eu-west-1.amazonaws.com"
  AWS_ECR_REPOSITORY: "pipelines/async-data-discovery-metadata-updater"
  APP_IMAGE_REPO_NAME: "${AWS_ECR_REPOSITORY}"
  APP_IMAGE_REPO_URL: "${AWS_ECR_REGISTRY}/${APP_IMAGE_REPO_NAME}"
  TF_IMAGE_REPO_NAME: "${AWS_ECR_REPOSITORY}-tf"
  TF_IMAGE_REPO_URL: "${AWS_ECR_REGISTRY}/${TF_IMAGE_REPO_NAME}"
  AWS_DEFAULT_REGION: "eu-west-1"
  ROLE_ARN: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/terraform-deployment-role"
  FEATURE_BRANCH_TAG: "MR-${CI_MERGE_REQUEST_IID}"
  RELEASE_TAG: "latest"
```

Most of these variables are self-explanatory but we can highlight that we are defining a repository name `APP_IMAGE_REPO_NAME` for our application steps **test** and **build**; and another repository name `TF_IMAGE_REPO_NAME` for `Terraform` related steps ** **deploy** and **cleanup**.

Also, we are defining a `ROLE_ARN` the represents a role that we have already defined in our **AWS** account that has all the necessary permissions to perform the actions required by our pipelin.

Then, we can define a `YAML anchor` as follows:

```yaml
.get_aws_credentials: &get_aws_credentials
  - apk update && apk add --no-cache aws-cli jq
  - echo "Assuming role $ROLE_ARN"
  - export TEMP_ROLE=$(aws sts assume-role --role-arn $ROLE_ARN --role-session-name gitlab-ci-session)
  - export AWS_ACCESS_KEY_ID=$(echo $TEMP_ROLE | jq -r '.Credentials.AccessKeyId')
  - export AWS_SECRET_ACCESS_KEY=$(echo $TEMP_ROLE | jq -r '.Credentials.SecretAccessKey')
  - export AWS_SESSION_TOKEN=$(echo $TEMP_ROLE | jq -r '.Credentials.SessionToken')
```

We assume that our pipeline is running under credentials that have the permission to assume `$ROLE_ARN`. Then, `TEMP_ROLE` contains temporary security credentials. After some transformation, we have the environment variables `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` that we can use to authenticate and authorize **AWS CLI** commands to access **AWS** resources with the permissions granted to the assumed role.

As a reminder, these are our pipeline steps:

```yaml
stages:
  - build images
  - unit test
  - build package
  - merge request
  - cleanup
  - deploy
```

We need that this step runs when we commit to a **merge request** and also when merginng into the **main** branch. We store the credentials retrieved by the anchor using the `dotenv` functionality.

