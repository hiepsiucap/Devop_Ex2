# AWS CodePipeline with Tomcat Deployment using CloudFormation

This guide explains how to set up an AWS CodePipeline to automate the CI/CD process for a Java application deployed to Apache Tomcat on AWS Elastic Beanstalk using the `bt1.yaml` CloudFormation template.

## Overview
The pipeline uses:
- **AWS CodeCommit** as the source repository.
- **AWS CodeBuild** to build the Java application.
- **AWS Elastic Beanstalk** to deploy the application to a Tomcat environment.
- **CloudFormation** to define and deploy the infrastructure.

## Prerequisites
- **AWS Account**: Active AWS account with IAM permissions for CloudFormation, CodePipeline, CodeCommit, CodeBuild, Elastic Beanstalk, and S3.
- **AWS CLI**: Installed and configured (`aws configure`).
- **Java Application**: A Java web application (e.g., a `.war` file) in a CodeCommit repository.
- **Maven**: Used for building the application (ensure `pom.xml` is in the repository).

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
   ├── pom.xml
   ├── src/
   │   └── (Your Java application source code)
   └── README.md
   ```

3. **Create CodeCommit Repository**:
   - Create a repository in CodeCommit:
     ```bash
     aws codecommit create-repository --repository-name my-tomcat-repo
     ```
   - Push your Java application code (including `pom.xml` and `buildspec.yml`) to the repository.

## CloudFormation Template
The `bt1.yaml` CloudFormation template defines the pipeline, IAM roles, S3 bucket, CodeBuild project, and Elastic Beanstalk environment. Below is an example:

<xaiArtifact artifact_id="c568e312-18c8-4e46-bb2b-7e34d4d0dfd6" artifact_version_id="ff1cd942-fd83-4a7c-9921-a29721fdb239" title="bt1.yaml" contentType="text/yaml">
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS CodePipeline for Tomcat Deployment to Elastic Beanstalk

Parameters:
  PipelineName:
    Type: String
    Default: tomcat-pipeline
  RepositoryName:
    Type: String
    Default: my-tomcat-repo
  BranchName:
    Type: String
    Default: main
  CodeBuildProjectName:
    Type: String
    Default: tomcat-build-project
  ArtifactBucketName:
    Type: String
    Default: tomcat-pipeline-artifacts
  ElasticBeanstalkAppName:
    Type: String
    Default: tomcat-app
  ElasticBeanstalkEnvName:
    Type: String
    Default: tomcat-env

Resources:
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref ArtifactBucketName

  PipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: PipelinePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:*
                  - codecommit:*
                  - codebuild:*
                  - elasticbeanstalk:*
                  - iam:PassRole
                Resource: '*'

  CodeBuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: CodeBuildPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:*
                  - logs:*
                  - codecommit:*
                Resource: '*'

  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Ref CodeBuildProjectName
      Source:
        Type: CODECOMMIT
        Location: !Sub https://git-codecommit.${AWS::Region}.amazonaws.com/v1/repos/${RepositoryName}
      Artifacts:
        Type: S3
        Location: !Ref ArtifactBucket
        Name: build-output
      Environment:
        Type: LINUX_CONTAINER
        Image: aws/codebuild/amazonlinux2-x86_64-standard:5.0
        ComputeType: BUILD_GENERAL1_SMALL
        EnvironmentVariables:
          - Name: JAVA_HOME
            Value: /usr/lib/jvm/java-11-amazon-corretto
      ServiceRole: !GetAtt CodeBuildRole.Arn

  ElasticBeanstalkApp:
    Type: AWS::ElasticBeanstalk::Application
    Properties:
      ApplicationName: !Ref ElasticBeanstalkAppName

  ElasticBeanstalkEnv:
    Type: AWS::ElasticBeanstalk::Environment
    Properties:
      ApplicationName: !Ref ElasticBeanstalkApp
      EnvironmentName: !Ref ElasticBeanstalkEnvName
      SolutionStackName: 64bit Amazon Linux 2023 v4.0.10 running Tomcat 9 Corretto 11
      OptionSettings:
        - Namespace: aws:autoscaling:launchconfiguration
          OptionName: InstanceType
          Value: t2.micro

  CodePipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: !Ref PipelineName
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeCommit
                Version: '1'
              OutputArtifacts:
                - Name: SourceOutput
              Configuration:
                RepositoryName: !Ref RepositoryName
                BranchName: !Ref BranchName
              RunOrder: 1
        - Name: Build
          Actions:
            - Name: BuildAction
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              InputArtifacts:
                - Name: SourceOutput
              OutputArtifacts:
                - Name: BuildOutput
              Configuration:
                ProjectName: !Ref CodeBuildProjectName
              RunOrder: 1
        - Name: Deploy
          Actions:
            - Name: DeployAction
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: ElasticBeanstalk
                Version: '1'
              InputArtifacts:
                - Name: BuildOutput
              Configuration:
                ApplicationName: !Ref ElasticBeanstalkAppName
                EnvironmentName: !Ref ElasticBeanstalkEnvName
              RunOrder: 1
```

## Configuration
1. **AWS Credentials**:
   - Configure AWS CLI:
     ```bash
     aws configure
     ```

2. **Buildspec File**:
   - Create a `buildspec.yml` in the repository root:
     ```yaml
     version: 0.2
     phases:
       install:
         runtime-versions:
           java: corretto11
       build:
         commands:
           - echo "Building the Java application"
           - mvn clean package
     artifacts:
       files:
         - target/*.war
       discard-paths: yes
     ```

3. **Java Application**:
   - Ensure your Java application has a `pom.xml` for Maven.
   - Example `pom.xml` snippet:
     ```xml
     <project>
       <modelVersion>4.0.0</modelVersion>
       <groupId>com.example</groupId>
       <artifactId>my-tomcat-app</artifactId>
       <version>1.0-SNAPSHOT</version>
       <packaging>war</packaging>
       <dependencies>
         <dependency>
           <groupId>javax.servlet</groupId>
           <artifactId>javax.servlet-api</artifactId>
           <version>4.0.1</version>
           <scope>provided</scope>
         </dependency>
       </dependencies>
       <build>
         <plugins>
           <plugin>
             <groupId>org.apache.maven.plugins</groupId>
             <artifactId>maven-war-plugin</artifactId>
             <version>3.3.2</version>
           </plugin>
         </plugins>
       </build>
     </project>
     ```

4. **Validate Template**:
   - Validate the CloudFormation template:
     ```bash
     aws cloudformation validate-template --template-body file://bt1.yaml
     ```

## Deploying the Pipeline
1. **Create the Stack**:
   - Deploy the CloudFormation stack:
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

4. **Access the Application**:
   - Once deployed, find the Elastic Beanstalk environment URL in the AWS Console or via:
     ```bash
     aws elasticbeanstalk describe-environments --environment-names tomcat-env
     ```

5. **Update the Stack** (if needed):
   - Update with changes:
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
- **Version Control**: Store `bt1.yaml`, `buildspec.yml`, and application code in a repository, excluding sensitive data.
- **Logging**: Enable CloudWatch Logs for CodePipeline, CodeBuild, and Elastic Beanstalk.
- **Testing**: Test the pipeline in a staging environment before production.
- **Backup**: Regularly back up the CodeCommit repository and S3 artifacts.

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
   - Ensure `buildspec.yml` and `pom.xml` are valid.
   - Verify CodeBuild environment (e.g., Java version).
- **Deployment Issues**:
   - Confirm Elastic Beanstalk environment settings and Tomcat compatibility.
   - Check application logs in CloudWatch or Elastic Beanstalk.

For detailed documentation, refer to:
- [AWS CodePipeline Docs](https://docs.aws.amazon.com/codepipeline)
- [AWS CloudFormation Docs](https://docs.aws.amazon.com/cloudformation)
- [Elastic Beanstalk Tomcat Docs](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/java-tomcat-platform.html)