# ci-cd-test-ocr 
This is an AWS SAM app that uses Rekognition APIs to detect text in S3 Objects and stores labels in DynamoDB.

## Project structure
Here is a brief overview of that has been generated.
```bash
.
├── README.md                   <-- This instructions file
├── src                         <-- Source code for the Lambda function
│   ├── __init__.py
│   └── app.py                  <-- Lambda function code
├── template.yaml               <-- SAM template
└── SampleEvent.json            <-- Sample S3 event
```


## Requirements
* AWS CLI
* [Python 3.6 installed](https://www.python.org/downloads/)
* [Docker installed](https://www.docker.com/community-edition)
* [Python Virtual Environment](http://docs.python-guide.org/en/latest/dev/virtualenvs/)


## We will deploy the SAM project in two ways.
#### 1. CLI Commands to package and deploy your application(Manual command line process)
#### 2. Minimal pipeline to package and deploy your application(Automated CI/CD process)
# 1. CLI Commands to package and deploy your application
CLI commands to package, deploy and describe outputs defined within the cloudformation stack.

First, we need an `S3 bucket` where we can upload our Lambda functions packaged as ZIP before we deploy anything - If you don't have a S3 bucket to store code artifacts then this is a good time to create one:

```bash
aws s3 mb s3://BUCKET_NAME
```

Next, run the following command to package your Lambda function. The `sam package` command creates a deployment package (ZIP file) containing your code and dependencies, and uploads them to the S3 bucket you specify. 

```bash
sam package \
    --template-file template.yaml \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME
```

The `sam deploy` command will create a Cloudformation Stack and deploy your SAM resources.
```bash
sam deploy \
    --template-file packaged.yaml \
    --stack-name REPLACE_THIS_WITH_YOUR_STACK_NAME \
    --capabilities CAPABILITY_IAM \
```

ALL DONE . . .

# 2. Minimal pipeline to package and deploy your application

**This is an example of how to create a minimal pipeline for SAM based Serverless Apps**

![Pipeline Sample Image](pipeline-sample.png)

## Additional Project structure
Here is a brief overview of that has been generated.
```bash
.
├── pipeline                    <-- CI/CD managed folder
│   ├── pipeline.yaml           <-- The CI/CD AWS resource configuration file to automate deployment
│   └── buildspec.yaml          <-- The file which Build and Deploy source code after git PUSH. Usage of this file describes at the end of instructions. 
```

## Requirements

* AWS CLI already configured with Administrator access 
    - Alternatively, you can use a [Cloudformation Service Role with Admin access](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html)
* [Github Personal Token](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) with full permissions on **admin:repo_hook and repo**

## Configuring GitHub Integration

This Pipeline is configured to look up for GitHub information stored on [EC2 System Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) such as Branch, Repo, Username and OAuth Token.

Replace the placeholders with values corresponding to your GitHub Repo and Token:

```bash
aws ssm put-parameter \
    --name "ocr-pipeline-github-repo" \
    --description "Github Repository name for Cloudformation Stack ocr-pipeline-stack" \
    --type "String" \
    --value "Github Repo Name"

aws ssm put-parameter \
    --name "ocr-pipeline-github-token" \
    --description "Github Token for Cloudformation Stack ocr-pipeline-stack" \
    --type "String" \
    --value "GITHUB TOKEN"

aws ssm put-parameter \
    --name "ocr-pipeline-github-user" \
    --description "Github Username for Cloudformation Stack ocr-pipeline-stack" \
    --type "String" \
    --value "GITHUB USER"
```

**NOTE 1:** Keep in mind that these Parameters will only be available within the same region you're deploying this Pipeline stack. Also, if these values ever change you will need to [update these parameters](https://docs.aws.amazon.com/cli/latest/reference/ssm/put-parameter.html) as well as update the "ocr-pipeline-stack" Cloudformation stack.

**NOTE 2:** You can replace --name parameter value to easily identify by choosing a name

## Pipeline creation

<details>
<summary>If you don't use Python or don't want to trigger the Pipeline from the `master` branch click here...</summary>
Before we create this 3-environment Pipeline through Cloudformation you may want to change a couple of things to fit your environment/runtime:

* **CodeBuild** uses a `Python` build image by default and if you're not using `Python` as a runtime you can change that
    - [CodeBuild offers multiple images](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html) and you can  update the `Image` property under `pipeline.yaml` file accordingly

```yaml
    CodeBuildProject:
        Type: AWS::CodeBuild::Project
        Properties:
            ...
            Environment: 
                Type: LINUX_CONTAINER
                ComputeType: BUILD_GENERAL1_SMALL
                Image: aws/codebuild/python:3.6.5 # More info on Images: https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html
                EnvironmentVariables:
                  - 
                    Name: BUILD_OUTPUT_BUCKET
                    Value: !Ref BuildArtifactsBucket
...
```

* **CodePipeline** uses the `master` branch to trigger the CI/CD pipeline and if you want to specify another branch you can do so by updating the following section in the `pipeline.yaml` file.
```yaml
    Stages:
        - Name: Source
            Actions:
            - Name: SourceCodeRepo
                ActionTypeId:
                # More info on Possible Values: https://docs.aws.amazon.com/codepipeline/latest/userguide/reference-pipeline-structure.html#action-requirements
                Category: Source
                Owner: ThirdParty
                Provider: GitHub
                Version: "1"
                Configuration:
                Owner: !Ref GithubUser
                Repo: !Ref GithubRepo
                Branch: master
                OAuthToken: !Ref GithubToken
                OutputArtifacts:
                - Name: SourceCodeAsZip
                RunOrder: 1
```
</details>

Run the following AWS CLI command from pipeline folder to create your first pipeline for your SAM based Serverless App:

```bash
aws cloudformation create-stack \
    --stack-name REPLACE_THIS_WITH_YOUR_STACK_NAME_THAT_DOES_NOT_EXIST \
    --template-body file://pipeline.yaml \
    --capabilities CAPABILITY_NAMED_IAM
```

This may take a couple of minutes to complete, therefore give it a minute or two and then run the following command to retrieve the Git repository:

```bash
aws cloudformation describe-stacks \
    --stack-name YOUR_STACK_NAME \
    --query 'Stacks[].Outputs'
```

## Release through the newly built Pipeline

Although CodePipeline will orchestrate this 3-environment CI/CD pipeline we need to learn how to integrate our toolchain to fit the following sections:

> **Source code**

All you need to do here is to initialize a local `git repository` for your existing service if you haven't done already and connect to the `git repository` that you retrieved in the previous section.

```bash
git init
```

Next, add a new Git Origin to connect your local repository to the remote repository:
* [Git Instructions for HTTPS access](https://help.github.com/articles/adding-a-remote/)

> **Build steps**

This Pipeline expects `buildspec.yaml` to be at the root of this `git repository` and **CodeBuild** expects will read and execute all sections during the Build stage.

Open up `buildspec.yaml` using your favourite editor and customize it to your needs - Comments have been added to guide you what's needed

> **Triggering a deployment through the Pipeline**

The Pipeline will be listening for new git commits pushed to the `master` branch (unless you changed), therefore all we need to do now is to commit to master and watch our pipeline run through:

```bash
git add . 
git commit -m "Kicking the tires of my first CI/CD pipeline"
git push origin master
```
