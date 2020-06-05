# Challenge

You are an AWS security engineer, and you are building a container with all of your security tools already packaged to be used across the environment. Eventually, this container will need to be pushed to Amazon Elastic Container Registry. ECR Images can be used with a variety of AWS services.

Using AWS CodeBuild, build a Docker image and push the image to ECR. Make sure that you install cfn-lint and aws-cli into the container.

Tips: You will need to create a CodeBuild buildspec.yml and a Dockerfile, which will be your CodeBuild source, called source.zip. This solution requires an S3 Bucket for the CodeBuild source and an ECR repository. The S3 Bucket and ECR name contains sj-docker-push-12345. The docker image tag is sj-docker-push. When you are creating the CodeBuild project, make sure to select the service role called sj-docker-push-codebuild and BE SURE TO UNCHECK the check-box "Allow Codebuild to modify the IAM role", if you do not do this, you will get an access denied.

Try to determine why a local docker push to ECR will not work. Once the CodeBuild job completes successfully, it will post the answer to Parameter Store under the Parameter SJ-Docker-Push-Answer. Parameter Store can be found under the Systems Manager service in the Console.

Make sure to replace 12345 with your account id in all hints and references. The exact names of the S3 Bucket and the ECR Repository will be included within the Output Properties tab to the left.

# Solution

The idea is to create a Dockerfile and push to ECR using codebuild.


You need to push to S3 your DockerFile and buildspec.yml
Build codebuild with Linux Image and use the S3 as source.
Run the build.


1. Run `aws codebuild create-project --generate-cli-skeleton`.
2. Copy all the output to 'create-project.json'
```
{
    "name": "samples-docker-project",
    "description": "",
    "source": {
        "type": "S3",
        "location": "sj-docker-push-AAAAAA/source.zip"
    },
    "artifacts": {
        "type": "NO_ARTIFACTS"
    },
    "environment": {
        "type": "LINUX_CONTAINER",
        "image": "aws/codebuild/standard:4.0",
        "computeType": "BUILD_GENERAL1_LARGE",
        "environmentVariables": [
            {
                "name": "AWS_DEFAULT_REGION",
                "value": "us-east-1"
            },
            {
                "name": "AWS_ACCOUNT_ID",
                "value": "AAAAAAAA"
            },
            {
                "name": "IMAGE_REPO_NAME",
                "value": "sj-docker-push-AAAAAAA"
            },
            {
                "name": "IMAGE_TAG",
                "value": "sj-docker-push"
            }
        ],
        "privilegedMode": true,
        "imagePullCredentialsType": "CODEBUILD"
    },
    "serviceRole": "sj-docker-push-codebuild"
}
```
3. Run `aws codebuild create-project --cli-input-json file://create-project.json`
3. Create buildspec.yml
```
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 12
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
artifacts:
  files:
```
4. Create Dockerfile
```
   # Use the standard Amazon Linux base, provided by ECR/KaOS
# It points to the standard shared Amazon Linux image, with a versioned tag.
FROM amazonlinux:2
LABEL Maintainer awsvs

# Framework Versions
ENV VERSION_NODE=10.16.3

# UTF-8 Environment
ENV LANGUAGE en_US:en
ENV LANG=en_US.UTF-8
ENV LC_ALL en_US.UTF-8

## Install OS packages
RUN touch ~/.bashrc
RUN yum -y install \
    jq \
    tar \
    unzip \
    zip \
    gzip \
    python3 \
    python3-devel \
    yum clean all && \
    rm -rf /var/cache/yum


## Install Node 10
#RUN curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
RUN /bin/bash -c ". ~/.nvm/nvm.sh && \
    nvm install $VERSION_NODE && nvm use $VERSION_NODE && \
    nvm alias default node && nvm cache clear"

ENV PATH /root/.nvm/versions/node/v${VERSION_NODE}/bin:/usr/local/rvm/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

## Environment Setup
RUN echo "nvm use ${VERSION_NODE} 1> /dev/null" >> ~/.bashrc

## Install awscli
RUN /bin/bash -c "pip3 install awscli && rm -rf /var/cache/apk/*"

# COPY envCache /usr/local/bin
# RUN chmod +x /usr/local/bin/envCache
COPY amplifyPush.sh /usr/local/bin
RUN chmod +x /usr/local/bin/amplifyPush.sh

RUN source ~/.bashrc

ENTRYPOINT [ "bash", "-c" ]
```

1. Run `aws codebuild start-build --project-name samples-docker-project` to start the build.    