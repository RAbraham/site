---
categories:
- markdown
date: '2020-03-31'
description: Run PureScript 0.12 on AWS Lambda, using Express on the Serverless platform
layout: post
title: PureScript on AWS Lambda. Using Express and Serverless
toc: true

---

*This post is an ported, edited version of the [original](https://medium.com/@rajiv.abraham/purescript-on-aws-lambda-7cf04bbcc25e)*

### Purpose

This post shows you how to run PureScript 0.12 on AWS Lambda, using [Express](https://expressjs.com/) on the [Serverless](https://serverless.com/) platform.

### Setup

The final output of this article is also a [repo](https://github.com/RAbraham/hello-purescript-serverless)

### Prerequisites.

You should have nodejs (i.e. nvm, npm) setup on your machine, preferably 8.10. AWS Lambda uses 8.10. I’m new to nodejs land but I’m guessing that higher minor versions should be ok.

### Software Setup
```
npm install -g purescript
npm install -g pulp bower
npm install -g serverless
```

### AWS Credentials

If you haven’t already, generate an AWS key and secret. This user must have AdministratorAccess permissions. Here are the [docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey_CLIAPI) or a slightly outdated [video](https://www.youtube.com/watch?v=HSd9uYj2LJA). For the video, follow on to 1:40 and ignore the Serverless Dashboard instructions(around 0:58 to 1:00), we are going to do that on the command line.

```
serverless config credentials --provider aws --key your-aws-key --secret your-aws-secret
```

### Project Setup
```
mkdir hello-purescript-serverless
cd hello-purescript-serverless
npm init # fill in as directed
pulp init
pulp build

```

If all goes well, you should see something like below:
```
* Building project in /Users/rabraham/Documents/dev/purescript/hello-purescript-serverless
Compiling Data.Symbol
Compiling Type.Data.RowList
...
Compiling Main
Compiling PSCI.Support
Compiling Effect.Class.Console
* Build successful.

```

Now let’s install our project specific packages.
```
bower install --save purescript-aws-lambda-express purescript-express
npm install --save aws-serverless-express express
npm install serverless-offline --save-dev

```

purescript-express is a wrapper on express while purescript-aws-lambda-express provides the wrapper for AWS Lambda. serverless-offline allows us to test the code locally before deploying it to AWS Lambda.

At time of writing, purescript-express has an issue where we have to install the following two packages too. Try a `pulp build` right now and if that fails, run the following commands
```
bower install --save purescript-test-unit
bower install --save purescript-aff

```

Let’s build it
```
pulp build
```

You should see some warnings but at the end, you should see `* Build successful`.

### Main Course

In your `src/Main.purs`, delete the previous code and paste the following:
```
module Main where
 
import Node.Express.App (App, get)
import Node.Express.Handler (Handler)
import Node.Express.Response (sendJson)
import Network.AWS.Lambda.Express as Lambda
 
-- Define an Express web app
 
indexHandler :: Handler
indexHandler = do
  sendJson { status: "ok" }
 
app :: App
app = do
  get "/" indexHandler
 
-- Define the AWS Lambda handler
 
handler :: Lambda.HttpHandler
handler =
  Lambda.makeHandler app

```

Build Your App
```
pulp build
```

### Serverless Setup

In the root of your project, create the file `serverless.yml` and paste the following:
```yaml
service: purescript-aws-lambda-express-test
 
provider:
  name: aws
  runtime: nodejs8.10
  memorySize: 128
  # stage: ${opt:stage dev}
  region: us-east-1
 
functions:
  lambda:
    handler: output/Main/index.handler
    events:
      - http:
          path: / # this matches the base path
          method: ANY
      - http:
          path: /{any+}
          method: ANY
 
plugins:
  - serverless-offline

```

Let’s test this locally: On one terminal.
```
serverless offline start
```

Open another terminal and do:
```
curl http://localhost:3000
```

You should see `{"status":"ok"}`

### Deploy

Once it works locally, let’s deploy to AWS.
```
serverless deploy -v
```

Output should look like something below. Note, your endpoint will be different:
```
Serverless: Packaging service
...
Service Information
service: purescript-aws-lambda-express-test
stage: dev
region: us-east-1
...
Stack Outputs
...
ServiceEndpoint: https://l4qajv7v95.execute-api.us-east-1.amazonaws.com/dev
....

```

Copy the link shown as `ServiceEndpoint` and you know what to do!
```
curl https://l4qajv7v95.execute-api.us-east-1.amazonaws.com/dev
```

Output:
```
{"status":"ok"}%
```

### Undeploy
```
serverless remove -v
```
I hope this enables you to make PureScript web applications! Thanks to [purescript-express](https://github.com/nkly/purescript-express) and [purescript-aws-lambda-express](https://github.com/lpil/purescript-aws-lambda-express) for making this possible.

