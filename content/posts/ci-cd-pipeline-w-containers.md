+++
date = '2024-05-20T15:23:08+01:00'
title = 'A GitLab pipeline that re-uses containers to achieve faster run times'
+++

# Motivation

Continuos Integration and Continuos Deployment has become a necessary requirement in most (if not all) of our projects. We can get rid of many manual processes by building a pipeline in **GitLab**, or similar tool, which runs through the usual steps: **test**, **build**, **deploy**. However, these steps can involve redundant _automated_ actions, for example, the **test** steps needs to install the necessary dependencies of the project to run, while the **build** process might also need to install the same dependencies. In the case of `node_modules`, these can be trivially shared between environments, but it might not be as simple for `python` packages or other dependencies that are not as easily shared between pipeline steps or that require a large list of dependencies to be already present in the execution environment.

These _redundant automated_ actions take computation time and resources that are usually shared between teams or groups of developers in general. Also, I assume that few people enjoy waiting for a long repetitive pipeline to complete when deep down you know that, most likely, there is something that can be done about it.

In this blog entry, I show a way to share execution environments between pipeline jobs in the form of container images. These container images are built in the early stages of the pipeline and then consumed as required. I try to show that by doing this, we can save some time at the expense of space.

# Problem

In the problem I am presenting in this blog entry we have a `python` project where we are managing __code dependencies__ using `poetry`. Then, we build our code into an **artifact** that will __manifest__ our project in the execution environment. We deploy our code into the AWS infrastructure of choice using `Terraform`.

# Solution

The stages of the pipeline:

- build images
- unit test
- build package
- merge request
- cleanup
- deploy

We need to build the images for the different steps of the pipeline. The job of building these images is also going to become a step in the pipeline. Building images also implies that we need some repository where we are going to store them, and accessing some remote repository implies credentials.

Since we are already using **AWS** to deploy our project infrastructure, we can use **AWS ECR** to store our tagged images. Consequently, the **AWS** credentials that we are going to use in our pipeline also need the necessary permissions to push and retrieve images from **ECR**. To be a little more specific, the **IAM Role** that our credentials represent should've been granted the necessary policies to operate on **ECR** on the target **AWS Account**.

The **build images** stage runs on a `docker` image. It requires AWS credentials to upload to `AWS ECR` the images it generates. This stage will implement two jobs, one job generates the **app container** and the other generates the **terraform container**. The **app container** is in charge of all operation that require the python dependencies. The **terraform container** deals only with `Terraform` operations.

The **unit test** stage needs a `python3.12` image with all the code dependencies already installed. We need AWS credentials in the execution environment to pull the **app container** generated in the **build images** stage from `AWS ECR`.

The **build package** stage needs the `python3.12` image with all code dependencies installed. We need the necessary AWS credentials in the execution environment to pull the **app container** generated in the **build images** stage from `AWS ECR`.

The **merge request** stage needs an image with `terraform` installed. We need the necessary credentials in the execution environment to pull the **terraform container** generated in the **build images** stage from `AWS ECR`.

The **cleanup** stage needs an image with `terraform` installed. It also needs **AWS** credentials to retrieve an image from `AWS ECR`.

The **deploy production** stage needs and execution environment with `terraform` installed. It also needs **AWS** credentials to retrieve an image from `AWS ECR`.

Let's define the variables that our pipeline needs:

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

Most of these variables are self-explanatory but we can highlight that we are defining a repository name `APP_IMAGE_REPO_NAME` that the steps **test** and **build** are going to use to retrieve the image they need; and another repository name `TF_IMAGE_REPO_NAME` for the steps **deploy** and **cleanup**.

Also, we are defining a `ROLE_ARN` that represents a role that we have already defined in our target **AWS** account that has all the necessary permissions to perform the actions required by our pipeline, specifically, our infrastructure-related steps.

## Anchors

Then, we can define `YAML anchor`s as follows:

### Get Credentials Anchor

```yaml
.get_aws_credentials: &get_aws_credentials
  - apk update && apk add --no-cache aws-cli jq
  - echo "Assuming role $ROLE_ARN"
  - export TEMP_ROLE=$(aws sts assume-role --role-arn $ROLE_ARN --role-session-name gitlab-ci-session)
  - export AWS_ACCESS_KEY_ID=$(echo $TEMP_ROLE | jq -r '.Credentials.AccessKeyId')
  - export AWS_SECRET_ACCESS_KEY=$(echo $TEMP_ROLE | jq -r '.Credentials.SecretAccessKey')
  - export AWS_SESSION_TOKEN=$(echo $TEMP_ROLE | jq -r '.Credentials.SessionToken')
```

Let's assume that our pipeline is running under credentials that have the permission to assume `$ROLE_ARN`. Then, `TEMP_ROLE` contains temporary security credentials. After some transformation, we have the environment variables `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` that we can use to authenticate and authorize **AWS CLI** commands to access **AWS** resources with the permissions granted to the assumed role.

> **Note**: Let's say your GitLab Runners are running under an `EC2` instance and you need the AWS Credentials associated to this instance to pass it to one of your containers. How do you capture and then pass these `EC2` credentials to your container? If you have studied your AWS documentation, something should be murmuring the words _Instance Metadata_, if not, it is a good time to review the documentation. Anyhow, from your GitLab runner you can `curl` the instance metadata, retrieve it, transform it and re-use it at your leisure. It includes the credentials. _The EC2 Assumed Role credentials_, to be somewhat more specific.

### Log into ECR Anchor

```yaml
.log_into_ecr_with_env_credentials: &log_into_ecr_with_env_credentials
  - echo "Logging into ECR"
  - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
```

We assume that the `aws-cli` has been installed. This anchor needs some credentials already existing in the environment. Those credentials come from the `.get_aws_credentials` anchor.

### Run App Container Anchor

```yaml
.run_app_container: &run_app_container
  - docker pull $APP_IMAGE_REPO_URL:$IMAGE_TAG
  - docker run -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
    -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
    -e AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN
    -e ENVIRONMENT=$ENVIRONMENT
    --privileged -d --name app-container $APP_IMAGE_REPO_URL:$IMAGE_TAG sleep infinity
```

This anchor uses the credentials already set in the environment and passes them to the container it is going to run: `app-container`.

### Run Terraform Container Anchor

```yaml
.run_terraform_container: &run_terraform_container
  - docker pull $TF_IMAGE_REPO_URL:$IMAGE_TAG
  - docker run -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
    -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
    -e AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN
    -e ENVIRONMENT=$ENVIRONMENT
    --privileged -d --name terraform-container $TF_IMAGE_REPO_URL:$IMAGE_TAG sleep infinity
```

This anchor uses the credentials already set in the environment and passes them to the container it is going to run: `terraform-container`.

## Tasks

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

Consider that we maintain two static deployments with this pipeline: `staging` and `production`. Then, on each **merge request**, a new `MR` environment is created.

_The details of how the pipeline is structured to manifest these environments are left as an exercise to the reader._

### Build App Image

_Dockerfile.python-poetry_

It is important to put the commands in a sequence that results in as few amount of layer updates as possible.

```Dockerfile
FROM python:3.12.8-alpine3.21

# Set environment variables
ENV PIPX_HOME=/root/.local
ENV PATH=$PIPX_HOME/bin:$PATH

# Install zip
RUN apk add --no-cache zip make

# Install pipx and Poetry
RUN python3 -m pip install --user pipx && \
    python3 -m pipx ensurepath && \
    /root/.local/bin/pipx install poetry

# Verify installations
RUN python3 -m pipx --version && \
    poetry --version

# Set the working directory
WORKDIR /app

# Copy only the dependency files first
COPY pyproject.toml poetry.lock ./

# Install poetry dependencies
RUN poetry install --no-root
```

_Build App Image_

```yaml
build-app-image:
  stage: build images
  rules:
    - if: $CI_MERGE_REQUEST_IID && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != 'main'
      variables:
        IMAGE_TAG: $FEATURE_BRANCH_TAG
    - if: $CI_COMMIT_BRANCH == 'main' || $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      variables:
        IMAGE_TAG: $RELEASE_TAG
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
  script:
    - docker build -t $APP_IMAGE_REPO_NAME:$IMAGE_TAG -f scripts/Dockerfile.python-poetry .
    - docker tag $APP_IMAGE_REPO_NAME:$IMAGE_TAG $APP_IMAGE_REPO_URL:$IMAGE_TAG
    - docker push $APP_IMAGE_REPO_URL:$IMAGE_TAG
    - echo "Pushed $APP_IMAGE_REPO_URL:$IMAGE_TAG"
```

Before the script, we get the AWS Credentials. Remember that this `anchor` sets the credentials as environment variables. Then, we _smoothly_ log into `ECR`.

Note that we introduce rules to determine the proper tag for our image. In short, if in an `MR` environment, tag the image as `MR-${CI_MERGE_REQUEST_IID}`, else `latest`.

This task will build the image based on the `Dockerfile` and push it with a new image tag into our `ECR` respository. Subsequent jobs will be able to re-use this image.

### Build Terraform Image

_Dockerfile.terraform_

```Dockerfile
FROM node:20.18.1-alpine3.20

# Set environment variables
ENV TERRAFORM_VERSION="1.9.8"

# Install dependencies
RUN apk add --no-cache aws-cli jq wget curl unzip libc6-compat

# Install Terraform
RUN echo "Terraform is getting installed..." && \
    wget "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip" && \
    unzip -o "terraform_${TERRAFORM_VERSION}_linux_amd64.zip" && \
    mv terraform /usr/local/bin/terraform && \
    terraform -v && \
    echo "Terraform is installed!"
```

_Build Terraform Image_

```yaml
build-terraform-image:
  stage: build images
  rules:
    - if: $CI_MERGE_REQUEST_IID && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != 'main'
      variables:
        IMAGE_TAG: $FEATURE_BRANCH_TAG
    - if: $CI_COMMIT_BRANCH == 'main' || $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      variables:
        IMAGE_TAG: $RELEASE_TAG
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
  script:
    - echo "Logging into ECR"
    - docker build -t $TF_IMAGE_REPO_NAME:$IMAGE_TAG -f scripts/Dockerfile.terraform .
    - docker tag $TF_IMAGE_REPO_NAME:$IMAGE_TAG $TF_IMAGE_REPO_URL:$IMAGE_TAG
    - docker push $TF_IMAGE_REPO_URL:$IMAGE_TAG
    - echo "Pushed $TF_IMAGE_REPO_URL:$IMAGE_TAG"
```

Similar to **Build App Image** but uses the terraform Dockerfile instead.

### Unit Test

```yaml
unit-test:
  stage: unit test
  rules:
    - if: $CI_MERGE_REQUEST_IID && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != 'main'
      variables:
        IMAGE_TAG: $FEATURE_BRANCH_TAG
    - if: $CI_COMMIT_BRANCH == 'main' || $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      variables:
        IMAGE_TAG: $RELEASE_TAG
  variables:
    ENVIRONMENT: "mr-${CI_MERGE_REQUEST_IID}"
    AWS_REGION: $AWS_DEFAULT_REGION
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
    - *run_app_container
  script:
    - docker cp . app-container:/app
    - docker exec app-container ls -la /app
    - docker exec app-container make test
  after_script:
    - docker stop app-container
    - docker rm app-container
```

This is the first task where we re-use one of the images we already pushed into our `ECR` respository. We use the `app container` image to run our unit tests. Depending on the nature of our tests, we might not need to pass credentials to the container as we do in the anchor `.run_app_container`. Observe the syntax to execute a command inside the container using docker.

_If I need to use an environment variable in the container and I reference it in the command I am executing, is the value coming from my Job environment or the Container enviroment?_.

Also, notice that there is no need for other plugins or utilities to be able to run a container inside a container.

### Build Package

```yaml
build-package:
  stage: build package
  rules:
    - if: $CI_MERGE_REQUEST_IID && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != 'main'
      variables:
        ENVIRONMENT: "mr-${CI_MERGE_REQUEST_IID}"
        IMAGE_TAG: $FEATURE_BRANCH_TAG
    - if: $CI_COMMIT_BRANCH == 'main'
      variables:
        ENVIRONMENT: "prod"
        IMAGE_TAG: $RELEASE_TAG
    - if: $CI_COMMIT_BRANCH == 'staging'
      variables:
        ENVIRONMENT: "staging"
        IMAGE_TAG: $RELEASE_TAG
  variables:
    AWS_REGION: $AWS_DEFAULT_REGION
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
    - docker pull $APP_IMAGE_REPO_URL:$IMAGE_TAG
    - docker run -e ENVIRONMENT=$ENVIRONMENT --privileged -d --name build-container $APP_IMAGE_REPO_URL:$IMAGE_TAG sleep infinity
  script:
    - docker cp . build-container:/app
    - docker exec build-container ls -la /app
    - docker exec build-container make build
    - docker cp build-container:/app/artifact.zip .
  after_script:
    - docker stop build-container
    - docker rm build-container
  artifacts:
    when: always
    paths:
      - artifact.zip
    expire_in: 1 week
```

Before the script, we get credentials and log into `ECR`. Then, we run the `app container` as `build-container`. We don't reuse the `.run_app_container` anchor because in this case we don't need to pass credentials to the container.

Then, we build the project, extract the generated `artifact.zip` and store it in the pipeline.

We have defined some rules to make sure that we are pulling the correct image tag. We can use these rules to set other `VARIABLES` if required.

### Merge Request

```yaml
merge-request:
  stage: merge request
  dependencies:
    - build-package
  rules:
    - if: $CI_MERGE_REQUEST_IID && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != 'main'
      variables:
        ENVIRONMENT: "mr-${CI_MERGE_REQUEST_IID}"
        IMAGE_TAG: $FEATURE_BRANCH_TAG
        AWS_REGION: $AWS_DEFAULT_REGION
  environment:
    name: mr-${CI_MERGE_REQUEST_IID}
    on_stop: destroy-merge-request
    auto_stop_in: 1 week
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
    - *run_terraform_container
  script:
    - docker cp . tf-deploy-container:/app
    - docker exec  tf-deploy-container sh -c "chmod +x /app/scripts/terraform_apply.sh && /app/scripts/terraform_apply.sh"
  after_script:
    - docker stop tf-deploy-container
    - docker rm tf-deploy-container
```

This task deploys the `MR` environment. It re-uses the `terraform container` to run the required script. It only runs under specific conditions, see the `rules` section. It does not run if the pipeline is running for the `staging` or `main` branches.

### Deploy

```yaml
deploy:
  stage: deploy
  dependencies:
    - build-package
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      variables:
        ENVIRONMENT: "prod"
        IMAGE_TAG: $RELEASE_TAG
        AWS_REGION: $AWS_DEFAULT_REGION
    - if: $CI_COMMIT_BRANCH == "staging"
      variables:
        ENVIRONMENT: "staging"
        IMAGE_TAG: $RELEASE_TAG
        AWS_REGION: $AWS_DEFAULT_REGION
  environment:
    name: production
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
    - *run_terraform_container
  script:
    - docker cp . tf-deploy-container:/app
    - docker exec  tf-deploy-container sh -c "chmod +x /app/scripts/terraform_apply.sh && /app/scripts/terraform_apply.sh"
  after_script:
    - docker stop tf-deploy-container
    - docker rm tf-deploy-container
```

This task deploys the staging or production environments. It re-uses the `terraform container`. _Do I need to set `environment.name` to `staging` when running the pipeline for the `staging` branch?_ Yeah, maybe.

### Destroy Merge Request

```yaml
destroy-merge-request:
  stage: cleanup
  environment:
    name: mr-${CI_MERGE_REQUEST_IID}
    action: stop
  variables:
    ENVIRONMENT: "mr-${CI_MERGE_REQUEST_IID}"
    AWS_REGION: $AWS_DEFAULT_REGION
    IMAGE_TAG: "MR-${CI_MERGE_REQUEST_IID}"
  dependencies:
    - merge-request
    - build-package
  rules:
    - if: $CI_MERGE_REQUEST_IID && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != 'main'
      when: manual
  before_script:
    - *get_aws_credentials
    - *log_into_ecr_with_env_credentials
    - *run_terraform_container
  script:
    - docker cp . tf-deploy-container:/app
    - docker exec  tf-deploy-container sh -c "chmod +x /app/scripts/terraform_destroy.sh && /app/scripts/terraform_destroy.sh"
  after_script:
    - docker stop tf-deploy-container
    - docker rm tf-deploy-container
```

This task destroys the `MR` environment. It re-uses the `terraform container`. It does not run for `staging` nor `main`.

## Container usage

In short:

- Terraform Container: Used in the tasks
    - Merge Request
    - Destroy Merge Request
    - Deploy

- App Container: Used in the tasks
    - Unit Test
    - Build Package

## Conclusions

There are two evident contention points with this approach. First, and perhaps not as important, the need to pass credentials from the pipeline job environment to the containers that are spawned by the pipeline. It creates the need to retrieve and keep (temporary) these credentials as environment variables to pass them to the containers. The current approach relies on the `EC2` credentials that the GitLab Runners assume which is then used to assume the `Role` that we need. We can achieve some sense of security by restricting the GitLab Runner credentials so it only has permission to "assume" the required role and then the `Role` only has permission to deploy the infrastructure that we need and nothing more. Another way, I guess, could be to give the Gitlab Runner credentials all the permissions it needs to deploy the infrastructure we need. I am not totally sure here about which of these is better in the security sense but I'd say that since less permissions is usually best, then I'd say we can go with the first approach. I also guess that there is a better way to handle these credentials.

Second, the exchage of time for space might not be worth it. I guess it depends on how much you are spending on your `ECR` bill and how much value you put into that compared to the time you save on your pipeline. From my experience, the cost does not sky rocket. However, it is very helpful to define effective and *safe* `Lifecycle Rules` on your `ECR` respositories. It is important to emphasize *Safety* here, you don't to define a rule that deletes and old image that is also tagged as your `latest`. A single image can have multiple `tags` in `ECR`.

Then, it is also important to consider what optimizations `AWS` implements on the `ECR` respositories, I guess that *layers* are re-used as much as possible to save space so that the `size` value you see next to each image is the representation of the size of that particular image when downloading it for the first time and not the actual size it takes in `ECR` storage.

One important observation is that this approach becomes more effective as the number of steps increases in your pipeline. If your pipeline is simple enough, then of course, don't bother too much with this kinf of optimization, it is not worth it. However, it might be a good exercise of pipeline expertise.

There is so much to say about the different components and technology involved in this pipeline but I will leave that for the future me. I hope you have enjoyed this short text, I certainly enjoyed the trial and error process that took me to implement it the first time. I wish I had some graphs and numbers to exactly quantify how much time is saved by re-using images between pipeline steps but our intuition will have to suffice on this occasion.

