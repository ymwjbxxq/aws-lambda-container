# test-container

This project contains source code and supporting files for a serverless application that you can deploy with the SAM CLI. It includes the following files and folders.

- hello-world - Code for the application's Lambda function and Project Dockerfile.
- events - Invocation events that you can use to invoke the function.
- hello-world/tests - Unit tests for the application code.
- template.yml - A template that defines the application's AWS resources.

The application uses several AWS resources, including Lambda functions and an API Gateway API. These resources are defined in the `template.yml` file in this project. You can update the template to add AWS resources through the same deployment process that updates your application code.

## Deploy the sample application

The Serverless Application Model Command Line Interface (SAM CLI) is an extension of the AWS CLI that adds functionality for building and testing Lambda applications. It uses Docker to run your functions in an Amazon Linux environment that matches Lambda. It can also emulate your application's build environment and API.

To use the SAM CLI, you need the following tools.

* Docker - [Install Docker community edition](https://hub.docker.com/search/?type=edition&offering=community)
* SAM CLI - [Install the SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)

You may need the following for local testing.

* Node.js - [Install Node.js 12](https://nodejs.org/en/), including the NPM package management tool.

To build and deploy your application for the first time, run the following in your shell:

```bash
sam build
sam deploy --guided
```

The first command will build a docker image from a Dockerfile and then the source of your application inside the Docker image. The second command will package and deploy your application to AWS, with a series of prompts:

* **Stack Name**: The name of the stack to deploy to CloudFormation. This should be unique to your account and region, and a good starting point would be something matching your project name.
* **AWS Region**: The AWS region you want to deploy your app to.
* **Confirm changes before deploy**: If set to yes, any changesets will be shown to you before execution for manual review. If set to no, the AWS SAM CLI will automatically deploy application changes.
* **Allow SAM CLI IAM role creation**: Many AWS SAM templates, including this example, create AWS IAM roles required for the AWS Lambda function(s) included to access AWS services. By default, these are scoped down to minimum required permissions. To deploy an AWS CloudFormation stack that creates or modified IAM roles, the `CAPABILITY_IAM` value for `capabilities` must be provided. If permission isn't provided through this prompt, to deploy this example you must explicitly pass `--capabilities CAPABILITY_IAM` to the `sam deploy` command.
* **Save arguments to samconfig.toml**: If set to yes, your choices will be saved to a configuration file inside the project so that in the future you can just re-run `sam deploy` without parameters to deploy changes to your application.

You can find your API Gateway Endpoint URL in the output values displayed after deployment.

## Use the SAM CLI to build and test locally

Build your application with the `sam build` command.

```bash
test-container$ sam build
```

The SAM CLI builds a docker image from a Dockerfile and then installs dependencies defined in `hello-world/package.json` inside the docker image. The processed template file is saved in the `.aws-sam/build` folder.

Test a single function by invoking it directly with a test event. An event is a JSON document that represents the input that the function receives from the event source. Test events are included in the `events` folder in this project.

Run functions locally and invoke them with the `sam local invoke` command.

```bash
test-container$ sam local invoke HelloWorldFunction --event events/event.json
```

The SAM CLI can also emulate your application's API. Use the `sam local start-api` to run the API locally on port 3000.

```bash
test-container$ sam local start-api
test-container$ curl http://localhost:3000/
```

The SAM CLI reads the application template to determine the API's routes and the functions that they invoke. The `Events` property on each function's definition includes the route and method for each path.

```yaml
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

## Add a resource to your application
The application template uses AWS Serverless Application Model (AWS SAM) to define application resources. AWS SAM is an extension of AWS CloudFormation with a simpler syntax for configuring common serverless application resources such as functions, triggers, and APIs. For resources not included in [the SAM specification](https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md), you can use standard [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) resource types.

## Fetch, tail, and filter Lambda function logs

To simplify troubleshooting, SAM CLI has a command called `sam logs`. `sam logs` lets you fetch logs generated by your deployed Lambda function from the command line. In addition to printing the logs on the terminal, this command has several nifty features to help you quickly find the bug.

`NOTE`: This command works for all AWS Lambda functions; not just the ones you deploy using SAM.

```bash
test-container$ sam logs -n HelloWorldFunction --stack-name test-container --tail
```

You can find more information and examples about filtering Lambda function logs in the [SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-logging.html).

## Cleanup

To delete the sample application that you created, use the AWS CLI. Assuming you used your project name for the stack name, you can run the following:

```bash
aws cloudformation delete-stack --stack-name test-container
```

## Resources

See the [AWS SAM developer guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html) for an introduction to SAM specification, the SAM CLI, and serverless application concepts.

Next, you can use AWS Serverless Application Repository to deploy ready to use Apps that go beyond hello world samples and learn how authors developed their applications: [AWS Serverless Application Repository main page](https://aws.amazon.com/serverless/serverlessrepo/)


## From CI like Gitlab

```bash

.test_variables: &test_variables
  STAGE: test
  REGION: ${AWS_REGION}
  AWS_ROLE: arn:aws:iam::${DEV_ACCOUNT}:role/${ROLE_NAME}
  DOCKER_ECR_NAME: lambda-service-test
  DOCKER_PACKAGE_IMAGE_URI: xxxxxxxx.dkr.ecr.eu-central-1.amazonaws.com/${DOCKER_ECR_NAME}:latest
  CI_BUCKET: my-test-deployment

stages:
  - deploy:image
  - deploy:lambda

.deploy-image: &deploy-image
  stage: deploy:image
  before_script:
    # Install dependencies
    - pip install awscli --upgrade
    - pip install aws-sam-cli --upgrade
  script:
     # Setup Docker repository only for first time
    - ECR_EXISTS=$(aws ecr describe-repositories | grep $DOCKER_ECR_NAME || true)
    - (if [ -z "$ECR_EXISTS" ]; then
          aws cloudformation deploy 
                          --template ecr.yml
                          --stack-name lambda-container-ecr
                          --s3-bucket ${CI_BUCKET}
                          --capabilities CAPABILITY_NAMED_IAM
                          --region ${AWS_REGION} 
                          --parameter-overrides awsAdminRole=${AWS_ROLE} stage=${STAGE} Name=${DOCKER_ECR_NAME}
                          --force-upload
                          --no-fail-on-empty-changeset
                          --debug ;
      fi);
    #upload docker
    - cd src/
    - $(aws ecr get-login --no-include-email --region eu-central-1)
    - docker build -t $DOCKER_ECR_NAME .
    - docker tag $DOCKER_ECR_NAME:latest $DOCKER_PACKAGE_IMAGE_URI
    - docker push $DOCKER_PACKAGE_IMAGE_URI

      
deploy:image:test:
  <<: *deploy-image
  variables:
    <<: *test_variables
  environment:
    name: test
  when: manual


.deploy-lambda: &deploy-lambda
  stage: deploy:lambda
  before_script:
    # Install dependencies
    - pip install awscli --upgrade
    - pip install aws-sam-cli --upgrade
  script:
    # Compile for lambas folder
    - sam build
    - sam deploy --template-file function.yml 
                 --stack-name lambda-container
                 --s3-bucket ${CI_BUCKET}
                 --capabilities CAPABILITY_NAMED_IAM 
                 --region ${AWS_REGION} 
                 --force-upload
                 --no-fail-on-empty-changeset
                 --debug
      
deploy:lambda:test:
  <<: *deploy-lambda
  variables:
    <<: *test_variables
  environment:
    name: test
  when: manual

```
