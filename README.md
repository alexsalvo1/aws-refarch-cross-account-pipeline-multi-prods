## Initial Configuration
To refer on the differents Account, I have created these AWS profiles:
- Pipeline-Tools
- Pipeline-Dev
- Pipeline-Test
- Pipeline-Prod
- Pipeline-Prod2


#### 1. Create a sample application using Serverless Application Model

##### Create the sample application locally

```console
cd ~/Projects/Pipeline
sam init 
# Use the values: 
# - Template source:  AWS Quick Start Templates
# - Zip (artifact is a zip uploaded to S3)
# - Runtime: python3.9
# - Project Name: sample-lambda
# - AWS quick start application templates: Hello World Example
```

This creates a directory named `sample-lambda` in your current directory, which contains the code for a serverless application.

Navigate to the project folder and initialize the git client
```console
cd sample-lambda/
git init
```

#### 2. Create [AWS CodeCommit](code-commit-url) repository in Development Account

```console
aws codecommit create-repository --repository-name sample-lambda --repository-description "Sample Serverless App" --profile Pipeline-Dev
```

Note the cloneUrlHttp URL in the response from above CLI.

#### 3. Add a new remote

From your terminal application, execute the following command:

```console
git remote add AWSCodeCommit $HTTP_CLONE_URL
git config --global credential.helper '!aws --profile Pipeline-Dev codecommit credential-helper $@'
```

#### 4. Push the code AWS CodeCommit

From your terminal application, execute the following command:

```console
git add .
git commit -m "First push of my SAM app"
git push AWSCodeCommit master
```

### Create Pipeline cross acccounts and with two productions environments

#### 1. - In the Tools account, deploy this CloudFormation template. It will create the customer master keys (CMK) in AWS Key Management Service (AWS KMS), 
    grant access to Dev, Test, Prod and Prod2 accounts to use these keys, and create an Amazon S3 bucket to hold artifacts from AWS CodePipeline.
    
    Replace xxxxxxxxxxxx with the AWS Accounts

```console
cd ~/Projects/Pipeline/aws-refarch-cross-account-pipeline-multi-prods

export ENTER_TOOLS_ACCT=xxxxxxxxxxxx
export ENTER_DEV_ACCT=xxxxxxxxxxxx
export ENTER_TEST_ACCT=xxxxxxxxxxxx
export ENTER_PROD_ACCT=xxxxxxxxxxxx
export ENTER_PROD2_ACCT=xxxxxxxxxxxx

aws cloudformation deploy --stack-name pre-reqs \
--template-file ToolsAcct/pre-reqs.yaml --parameter-overrides \
DevAccount=$ENTER_DEV_ACCT TestAccount=$ENTER_TEST_ACCT \
ProductionAccount=$ENTER_PROD_ACCT ProductionAccount2=$ENTER_PROD2_ACCT \
--profile Pipeline-Tools
```

#### 2. - In the Dev account, which hosts the AWS CodeCommit repository, deploy this CloudFormation template. This template will create the IAM roles, 
    which will later be assumed by the pipeline running in the Tools account. Enter the AWS account number for the Tools account and the CMK ARN.
    
    Replace arn:aws:kms:xx-xxxx-x:xxxxxxxxxxxx:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx with the one generated in the first cloud formation

```console
export CMKARN=arn:aws:kms:xx-xxxx-x:xxxxxxxxxxxx:key/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

aws cloudformation deploy --stack-name toolsacct-codepipeline-role \
--template-file DevAccount/toolsacct-codepipeline-codecommit.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--parameter-overrides ToolsAccount=$ENTER_TOOLS_ACCT CMKARN=$CMKARN \
--profile Pipeline-Dev
```

#### 3. - In the Test and Prods accounts where you will deploy the Lambda code, execute this CloudFormation template. 
    This template creates IAM roles, which will later be assumed by the pipeline to create, deploy, and update the sample AWS Lambda function through CloudFormation.

    Replace pre-reqs-artifactbucket-xxxxxxxxxxxx with the one generated in the first cloud formation

```console
export S3Bucket=pre-reqs-artifactbucket-xxxxxxxxxxxx

aws cloudformation deploy --stack-name toolsacct-codepipeline-cloudformation-role \
--template-file TestAccount/toolsacct-codepipeline-cloudformation-deployer.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--parameter-overrides ToolsAccount=$ENTER_TOOLS_ACCT CMKARN=$CMKARN \
S3Bucket=$S3Bucket \
--profile Pipeline-Test

aws cloudformation deploy --stack-name toolsacct-codepipeline-cloudformation-role \
--template-file TestAccount/toolsacct-codepipeline-cloudformation-deployer.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--parameter-overrides ToolsAccount=$ENTER_TOOLS_ACCT CMKARN=$CMKARN \
S3Bucket=$S3Bucket \
--profile Pipeline-Prod

aws cloudformation deploy --stack-name toolsacct-codepipeline-cloudformation-role \
--template-file TestAccount/toolsacct-codepipeline-cloudformation-deployer.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--parameter-overrides ToolsAccount=$ENTER_TOOLS_ACCT CMKARN=$CMKARN \
S3Bucket=$S3Bucket \
--profile Pipeline-Prod2
```

#### 4. - In the Tools account, which hosts AWS CodePipeline, execute this CloudFormation template. 
    This creates a pipeline, but does not add permissions for the cross accounts (Dev, Test, Prod and Prod2).

```console
aws cloudformation deploy --stack-name sample-lambda-pipeline \
--template-file ToolsAcct/code-pipeline.yaml \
--parameter-overrides DevAccount=$ENTER_DEV_ACCT TestAccount=$ENTER_TEST_ACCT \
ProductionAccount=$ENTER_PROD_ACCT ProductionAccount2=$ENTER_PROD2_ACCT CMKARN=$CMKARN \
S3Bucket=$S3Bucket --capabilities CAPABILITY_NAMED_IAM \
--profile Pipeline-Tools
```

#### 5. - In the Tools account, execute this CloudFormation template, which give access to the role created in step 4. 
    This role will be assumed by AWS CodeBuild to decrypt artifacts in the S3 bucket. This is the same template that was used in step 1, but with different parameters.

```console
aws cloudformation deploy --stack-name pre-reqs \
--template-file ToolsAcct/pre-reqs.yaml \
--parameter-overrides CodeBuildCondition=true \
--profile Pipeline-Tools
```

#### 6. - In the Tools account, execute this CloudFormation template, which will do the following:
    Add the IAM role created in step 2. This role is used by AWS CodePipeline in the Tools account for checking out code from the AWS CodeCommit repository in the Dev account.
    Add the IAM role created in step 3. This role is used by AWS CodePipeline in the Tools account for deploying the code package to the Test and Prods accounts.

```console
aws cloudformation deploy --stack-name sample-lambda-pipeline \
--template-file ToolsAcct/code-pipeline.yaml \
--parameter-overrides CrossAccountCondition=true \
--capabilities CAPABILITY_NAMED_IAM \
--profile Pipeline-Tools
```