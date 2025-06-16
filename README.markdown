# AWS CodePipeline Infrastructure Setup with CloudFormation

This guide explains how to set up the infrastructure for an AWS CodePipeline using the `bt1.yaml` CloudFormation template. The pipeline is configured to use AWS CodeCommit as the source, AWS CodeBuild for building, and AWS Elastic Beanstalk with a Tomcat environment, but it does not include application code deployment.

## Overview
The pipeline sets up:
- **AWS CodeCommit**: Source repository for pipeline triggers.
- **AWS CodeBuild**: Build environment for processing source changes.
- **AWS Elastic Beanstalk**: Tomcat environment as the deployment target.
- **CloudFormation**: Defines the infrastructure in `bt1.yaml`.

## Prerequisites
- **AWS Account**: Active AWS account with IAM permissions for CloudFormation, CodePipeline, CodeCommit, CodeBuild, Elastic Beanstalk, and S3.
- **AWS CLI**: Installed and configured (`aws configure`).
- **CodeCommit Repository**: An empty or existing repository to trigger the pipeline.

## Installation
1. **Install AWS CLI**:
   - Download from [AWS CLI](https://aws.amazon.com/cli/).
   - Verify installation:
     ```bash
     aws --version
     ```

## Project Setup
1. **Clone the Repository** (if applicable):
   ```bash
   git clone <repository-url>
   cd <repository-directory>
   ```

2. **Directory Structure**:
   ```
   .
   ├── bt1.yaml
   ├── buildspec.yml
   └── README.md
   ```

3. **Create CodeCommit Repository**:
   - Create a repository in CodeCommit:
     ```bash
     aws codecommit create-repository --repository-name my-tomcat-repo
     ```
   - Optionally, push an empty `buildspec.yml` or placeholder files to the repository.

## Configuration
1. **AWS Credentials**:
   - Configure AWS CLI:
     ```bash
     aws configure
     ```

2. **Buildspec File**:
   - Create a minimal `buildspec.yml` in the repository root to satisfy CodeBuild requirements:
     ```yaml
     version: 0.2
     phases:
       build:
         commands:
           - echo "No application code to build. Infrastructure setup only."
     artifacts:
       files:
         - README.md
       discard-paths: yes
     ```

3. **Validate Template**:
   - Validate the `bt1.yaml` CloudFormation template:
     ```bash
     aws cloudformation validate-template --template-body file://bt1.yaml
     ```

## Deploying the Infrastructure
1. **Create the Stack**:
   - Deploy the CloudFormation stack to set up the pipeline and infrastructure:
     ```bash
     aws cloudformation create-stack \
       --stack-name tomcat-pipeline-stack \
       --template-body file://bt1.yaml \
       --capabilities CAPABILITY_NAMED_IAM \
       --parameters \
         ParameterKey=PipelineName,ParameterValue=tomcat-pipeline \
         ParameterKey=RepositoryName,ParameterValue=my-tomcat-repo \
         ParameterKey=BranchName,ParameterValue=main \
         ParameterKey=CodeBuildProjectName,ParameterValue=tomcat-build-project \
         ParameterKey=ArtifactBucketName,ParameterValue=tomcat-pipeline-artifacts \
         ParameterKey=ElasticBeanstalkAppName,ParameterValue=tomcat-app \
         ParameterKey=ElasticBeanstalkEnvName,ParameterValue=tomcat-env
     ```

2. **Monitor Stack Creation**:
   - Check stack status:
     ```bash
     aws cloudformation describe-stacks --stack-name tomcat-pipeline-stack
     ```

3. **Monitor Pipeline**:
   - View pipeline execution in the AWS Management Console or via CLI:
     ```bash
     aws codepipeline get-pipeline-state --name tomcat-pipeline
     ```

4. **Access the Elastic Beanstalk Environment**:
   - Find the Elastic Beanstalk environment URL in the AWS Console or via:
     ```bash
     aws elasticbeanstalk describe-environments --environment-names tomcat-env
     ```
   - Note: No application is deployed, so the environment will run the default Tomcat page.

5. **Update the Stack** (if needed):
   - Update with changes to `bt1.yaml`:
     ```bash
     aws cloudformation update-stack \
       --stack-name tomcat-pipeline-stack \
       --template-body file://bt1.yaml \
       --capabilities CAPABILITY_NAMED_IAM \
       --parameters \
         ParameterKey=PipelineName,ParameterValue=tomcat-pipeline \
         ParameterKey=RepositoryName,ParameterValue=my-tomcat-repo \
         ParameterKey=BranchName,ParameterValue=main \
         ParameterKey=CodeBuildProjectName,ParameterValue=tomcat-build-project \
         ParameterKey=ArtifactBucketName,ParameterValue=tomcat-pipeline-artifacts \
         ParameterKey=ElasticBeanstalkAppName,ParameterValue=tomcat-app \
         ParameterKey=ElasticBeanstalkEnvName,ParameterValue=tomcat-env
     ```

6. **Delete the Stack** (if needed):
   - Remove all resources:
     ```bash
     aws cloudformation delete-stack --stack-name tomcat-pipeline-stack
     ```

## Best Practices
- **Secure IAM Roles**: Use least-privilege policies for pipeline and CodeBuild roles.
- **Version Control**: Store `bt1.yaml` and `buildspec.yml` in a repository, excluding sensitive data.
- **Logging**: Enable CloudWatch Logs for CodePipeline, CodeBuild, and Elastic Beanstalk.
- **Testing**: Test the infrastructure in a non-production environment first.
- **Backup**: Back up the CodeCommit repository and S3 artifacts regularly.

## Troubleshooting
- **Stack Creation Fails**:
   - Check CloudFormation events:
     ```bash
     aws cloudformation describe-stack-events --stack-name tomcat-pipeline-stack
     ```
- **Pipeline Fails at Source**:
   - Verify CodeCommit repository and branch exist.
   - Check IAM role permissions for CodeCommit.
- **Build Stage Errors**:
   - Ensure `buildspec.yml` is present and valid.
   - Verify CodeBuild environment settings.
- **Deployment Issues**:
   - Confirm Elastic Beanstalk environment settings and Tomcat compatibility.
   - Check CloudWatch Logs for Elastic Beanstalk errors.

For detailed documentation, refer to:
- [AWS CodePipeline Docs](https://docs.aws.amazon.com/codepipeline)
- [AWS CloudFormation Docs](https://docs.aws.amazon.com/cloudformation)
- [Elastic Beanstalk Tomcat Docs](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/java-tomcat-platform.html)