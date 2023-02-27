# workshop-aws-go-lambda-function
A step-by-step guide to write a lambda function in golang. (90 min)

In this project you'll learn:
- How to set up your AWS credentials to use your AWS account
- How to write a Lambda function using Golang 1.x
- How to write a Makefile to build and deploy your Lambda function

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Create a simple golang program](#step-1-create-a-simple-golang-program)
- [Step 2 Transform your program into a Lambda function](#step-2-transform-your-program-into-a-lambda-function)
- [Step 3: Build your Lambda function](#step-3-build-your-lambda-function)
- [Step 4: Upload your code to AWS Lambda](#step-4-upload-your-code-to-aws-lambda)
- [Step 5: Test your Lambda function](#step-5-test-your-lambda-function)
- [Step 6: Deploy your Lambda function with the AWS cli](#step-6-deploy-your-lambda-function-with-the-aws-cli)
- [Step 7: Deploy your Lambda function with a Makefile](#step-7-deploy-your-lambda-function-with-a-makefile)
- [Step 8:  Use API Gateway to expose your Lambda function](#step-8-use-api-gateway-to-expose-your-lambda-function)
- [Step 9: Update our Lambda function to handle APIGatewayProxyRequest and APIGatewayProxyResponse_](#step-9-update-our-lambda-function-to-handle-apigatewayproxyrequest-and-apigatewayproxyresponse)

## Prerequisites

To deploy your lambda function in AWS, you will need the AWS cli installed on your machine.

- Get the [AWS cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

- Get your AWS Access keys. (Skip to [STEP 2](#step-2-transform-your-program-into-a-lambda-function)- if you already have an AWS profile configured)

- Log into your AWS account. Click on your name in the top right corner of the screen.
Your name > security credentials > Access Keys (Access Key ID and Secret Access Key) > Create New Access Key
Save your Access Key ID and Secret Access Key in a safe place.

- Create your AWS profile. [AWS Configure profiles documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html)

```bash
# Create a folder to store your AWS credentials
mkdir -p ~/.aws

# Add the following lines to the file ~/.aws/credentials 
vi ~/.aws/credentials
[yourname]
aws_access_key_id=AKXXXXXd
aws_secret_access_key=XXXX

# Add the following lines to the file ~/.aws/config
vi ~/.aws/config
[profile yourname]
region=ap-southeast-2
output=json
```
## Step 1 _Create a simple golang program_

NOTE: a golang script is executed locally. It will be compiled and deployed to AWS Lambda. We need to transform our 
local script into a Lambda function. [Step 2](#step-2-transform-your-program-into-a-lambda-function)
will explain how to do that.

- Create a file called main.go
```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("My program returns a string!")
}
```
- Run it with: `go run main.go`

When we execute the program, we can see the string printed in the terminal.

## Step 2 _Transform your program into a Lambda function_

NOTE: We want to execute our program in a Lambda function. This is not locally, it is in the AWS cloud. 
We need to transform our main function so that it can be called by the Lambda runtime.

- Edit the main.go file and replace the content with the following:

```
package main

import (
	"github.com/aws/aws-lambda-go/lambda"
)

// handleRequest is the entry point for the Lambda function
func handleRequest() (string, error) {
	return "My function returns a string!", nil
}

func main() {
	lambda.Start(handleRequest)
}
```

NOTE: now our program is relying on an external library to run since we imported github.com/aws/aws-lambda-go/lambda. Run `go mod init mylambda`. It will pull all the module dependencies.

NOTE: Try to run it locally with `go run main.go`. It will not work because the function is expecting to be called by the Lambda runtime.

## Step 3 _Build your Lambda function_

- Log into your AWS account. Search for Lambda in the search bar. Click on the Create function button.
- Select Author from scratch
- Name your function
- Select Go 1.x as the runtime
- Select Architecture: x86_64
- Click on the Create function button

NOTE: We created a function in AWS. Now we need to upload our code to it.
There are several ways to do that. We will first use the interface to upload a zip file.

## Step 4 _Upload your code to AWS Lambda_

- Open a terminal and go to the root of your project
- Build your code with `GOOS=linux GOARCH=amd64 go build -o main main.go`
- Zip your code with `zip main.zip main`

## Step 5 _Test your Lambda function_

In your AWS console, select your lambda function. Click on the Test tab. Click on the Configure test event button.


## Step 6 _Deploy your Lambda function with the AWS cli_

- Open a terminal and go to the root of your project
- Build your code with `GOOS=linux GOARCH=amd64 go build -o main main.go`
- Zip your code with `zip main.zip main`
- Deploy your code with `aws lambda update-function-code --function-name <your-function-name> --zip-file fileb://main.zip`

## Step 7 _Deploy your Lambda function with a Makefile_

- Create a Makefile with the following content:
```Makefile
# Set the AWS region and profile
AWS_REGION ?= ap-southeast-2
AWS_PROFILE ?= myprofile

# Name of your Lambda function ARN and its Golang entry file
LAMBDA_FUNCTION_NAME ?= arn:aws:lambda:ap-southeast-2:XXXX:function:go-lambda-function
LAMBDA_HANDLER ?= main

# Name of the ZIP file that contains the function code
ZIP_FILE_NAME ?= function.zip

# Build the function binary
build:
	GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o main main.go

# Create the ZIP file that will be uploaded to AWS
zip:
	zip $(ZIP_FILE_NAME) $(LAMBDA_HANDLER)

# Package the ZIP file and upload it to AWS
deploy: zip
	aws lambda update-function-code --region $(AWS_REGION) --function-name $(LAMBDA_FUNCTION_NAME) --zip-file fileb://$(ZIP_FILE_NAME) --profile $(AWS_PROFILE)

# Clean up the build artifacts
clean:
	rm -f $(ZIP_FILE_NAME) $(LAMBDA_HANDLER)

# All the steps to build, zip and deploy the function
all: build zip deploy clean

.PHONY: build zip deploy clean

```

## Step 8 _Use API Gateway to expose your Lambda function_

- In AWS console, Select your Lambda function. Click on the Add trigger button.
- Select API Gateway
- Select Create a new API
- Select HTTP API
- Select Open
- Click on the Add button
- Click on the API Gateway button
- Copy the API endpoint URL
e.g. https://j2jxgl21yj.execute-api.ap-southeast-2.amazonaws.com/default/go-lambda-remi

NOTE: Now we have an endpoint that we can call to execute our Lambda function.

## Step 9 _Update our Lambda function to handle APIGatewayProxyRequest and APIGatewayProxyResponse_

- Edit the main.go file and replace the content with the following:
```
package main

import (
	"context"
	"encoding/base64"
	"fmt"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"net/http"
)

func handleRequest(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	// Process the webhook request
	if request.HTTPMethod != "POST" {
		return events.APIGatewayProxyResponse{
			StatusCode: http.StatusBadRequest,
			Body:       "Invalid HTTP method",
		}, nil
	}

	// Extract the webhook data
	webhookData := request.Body
	// base64 encode webhook data
	webhookData = base64.StdEncoding.EncodeToString([]byte(webhookData))
	fmt.Println(webhookData)

	// Respond to the webhook request
	return events.APIGatewayProxyResponse{
		StatusCode:      http.StatusOK,
		Body:            webhookData,
		IsBase64Encoded: true,
	}, nil
}

func main() {
	lambda.Start(handleRequest)
}
```

NOTE: Now our function is expecting to be called by the API Gateway. It will return a response and an error.

- Try to call the API Gateway endpoint with a POST request. You should see the base64 encoded webhook data in the response.
```bash
curl --location --request POST 'LAMBDA_FUNCTION_ENDPOINT_URL' \
--header 'Content-Type: application/json' \
--data-raw '{
    "data": "This is the message"
}'
```


## Troubleshooting

Be mindful of the OS used when building using : `GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o main main.go`

e.g. On linux Mint when the build is zipped and uploaded to the lambda function, the test returned the following error.
``` 
2023/02/21 00:28:09 exit status 1
/var/task/main: /lib64/libc.so.6: version `GLIBC_2.32' not found (required by /var/task/main)
/var/task/main: /lib64/libc.so.6: version `GLIBC_2.34' not found (required by /var/task/main)
2023/02/21 00:31:33 exit status 1
START RequestId: d80b8a19-5ea9-4daa-9b47-8d3d7bda283a Version: $LATEST
RequestId: d80b8a19-5ea9-4daa-9b47-8d3d7bda283a Error: Runtime exited with error: exit status 1
Runtime.ExitError
END RequestId: d80b8a19-5ea9-4daa-9b47-8d3d7bda283a
REPORT RequestId: d80b8a19-5ea9-4daa-9b47-8d3d7bda283a	Duration: 16.05 ms	Billed Duration: 17 ms	Memory Size: 512 MB	Max Memory Used: 4 MB	```
