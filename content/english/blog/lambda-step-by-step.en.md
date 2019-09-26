---
title: "Lambda Step by Step"
date: 2019-09-26T19:07:46+02:00
draft: false
categories: ["Development"]
tags: ["aws", "lambda", "sam", "java8"]
---

During last weeks, I've worked a lot with [AWS Lambda](https://aws.amazon.com/it/lambda/). 

The main challenge I had was to look for a method to locally test those functions. Of course, it is possible to do some partial code execution as part of unit/integration test setup, but that won’t be close enough to production setup.

I'd like to build a sort of how-to guide in order to have a reference in the future. To do so, I have to:

- write a lambda that *does stuff*
- with the help of SAM (and its CLI), invoke and debug the function.

Let's start!

# SAM
SAM (*Serverless Application Model*) is an open source framework used to define, test and deploy serverless applications. SAM uses a *.yaml* file (also called *template*) that is then transformed into an AWS CloudFormation file.

In this scenario, I'll use SAM to define the lambda function endpoint. Using its CLI, I'll also be able to invoke (and debug) it locally.

[Here](https://aws.amazon.com/it/serverless/sam/) is the SAM official documentation.

# Lambda Creation

SAM CLI has a really helpful function to create a lambda stub:
```bash
sam init --runtime java8 --name StuffApi
```
This command creates an entire maven project with some example classes. The following structure is created:
```bash
StuffApi
├── README.md 
├── pom.xml
├── src
│   ├── main
│   │   └── java
│   │       └── helloworld
│   │           ├── App.java              <-- Lambda
│   │           └── GatewayResponse.java  <-- API Gateway Response POJO 
│   └── test
│       └── java
│           └── helloworld
│               └── AppTest.java
└── template.yaml
```
I'd suggest you to take a look at the `README.md`, which contains a lot of useful indications on next steps.

Now, it's time to code. Let's write a simple lambda function returning a `User` object:
```java 
public class StuffHandler implements RequestHandler<Void, GatewayResponse<UserDto>> {
    @Override
    public GatewayResponse<UserDto> handleRequest(final Void input, final Context context) {
        Map<String, String> headers = new HashMap<>();
        headers.put("Content-Type", "application/json");
        return new GatewayResponse<>(new UserDto.UserDtoBuilder().setEmail("foo@bar.com").setUsername("foobar").build(),
                headers,
                200);
    }
}
```
The response is wrapped inside a `GatewayResponse` object. It contains:

- `body`: the actual lambda response
- `headers`: custom response headers (e.g. `Content-Type`)
- `statusCode`: the response HTTP code (e.g. 200)

# SAM Template Creation
Let's move on to the most interesting thing, the SAM template. Here's the first template snippet:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
    StuffApi
    Sample SAM Template for StuffApi
Globals:
    Function:
        Timeout: 20
```
Here we can found generic information that SAM uses to parse the template (e.g. `AWSTemplateFormatVersion`). Then we have the `Globals` section: all lambda global variables are defined here.
```yaml
Resources:
  StuffFunction:
        Type: AWS::Serverless::Function
        Properties:
            CodeUri: target/StuffApi-1.0.jar
            Handler: dev.marcodenisi.stuff.StuffHandler::handleRequest
            Runtime: java8
            Events:
                Stuff:
                    Type: Api
                    Properties:
                        Path: /stuff
                        Method: get
```
In this second section, the actual *Lambda Function* is defined. We name it (`StuffFunction`) and we define the *jar* containing its source code along with the `handler` class.

Under `Events`, we define an API Gateway (`Type: Api`) responding to HTTP GET calls (`Method: get`) at the *path* `/stuff`.

# Invoking and debugging

First things first, we have to package the lambda source code. I'm using Java and Maven, so I have to run the `mvn package` command to create the *jar*.

To invoke the lambda, just run SAM CLI `invoke` command:
```bash
sam local invoke "StuffFunction" --no-event
... // service logs
{"body":{"username":"foobar","email":"foo@bar.com"},"headers":{"Content-Type":"application/json"},"statusCode":200}
```
This command does nothing but create a small docker container. In fact, the first run will be quite slow because of the docker image download. Next runs will be faster. [Here](https://hub.docker.com/r/lambci/lambda/) is the docker image documentation.

The `--no-event` option specifies that the lambda function won't accept anything in input. In the case input events are needed, the `sam local generate-event` command comes in handy.

To debug, just add the `-d` flag:
```bash
sam local invoke "StuffFunction" --no-event -d 5858
```
In this way, the lambda function won't be executed until a debugger is attached to the specified port (5858 in the example).

![That's all folks!](/gifs/thats_all_folks.gif)