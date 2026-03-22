# Serverless AWS Chatbot — My Thought Process

This project was built as part of the **GMU AWS Cloud Club**. The code was provided by the club, but I was responsible for deploying and configuring all the AWS services end-to-end. This document captures the decisions I made and why — not just what I clicked.

---

## Architecture

```
User → Amplify (Frontend) → API Gateway → Lambda → Bedrock (Claude)
                                                ↑
                                               IAM
```

| Service | Role |
|---|---|
| **API Gateway** | Front door — routes all incoming HTTP requests to the backend |
| **Lambda** | Backend logic — processes requests and calls Bedrock |
| **Amazon Bedrock** | LLM catalog — generates AI responses using Claude |
| **IAM** | Permissions — grants Lambda the access it needs, nothing more |
| **Amplify** | Hosting — delivers the frontend to users |

---

## How I Set This Up

The code for this project comes from the **GMU AWS Cloud Club** repository. I wanted to practice a real Git workflow, so here's what I actually did:

1. **Forked the club's repo** to my own GitHub account as a starting point
2. **Hit an issue pushing changes** to the fork — I ran into push errors I couldn't resolve at the time
3. **Worked around it** by creating a fresh repo (`rllyblaze/gum-aws-cc-serverless`), cloning the original, moving the files over, and setting up my own remote
4. **Initialized Git**, connected it to my new GitHub remote, and pushed the code up

```bash
git init
git remote add origin https://github.com/rllyblaze/gum-aws-cc-serverless.git
git add .
git commit -m "initial commit"
git push origin main
```

It wasn't the cleanest path, but working through that friction gave me hands-on experience with Git setup, remotes, and pushing — which is exactly what I was after. Debugging Git issues is part of the workflow too.

---

## My Setup & Decision Making

### Lambda

I started with Lambda because it's the core of the application — everything flows through it.

I named the function `aws-cc-serverless` and selected **Python 3.14** as the runtime since that's what the club's backend code was written in.

One of the first things I changed was the **timeout — bumping it from the default 3 seconds up to 30 seconds**. The reason: API Gateway has its own timeout limit, and if Lambda takes too long to respond (which it can when waiting on an LLM to generate a response), API Gateway will cut the connection before Lambda even finishes. Increasing the timeout gives the function enough runway to get a response back from Bedrock without the request failing on the gateway side.

### IAM Permissions

Lambda needs permission to call Bedrock — it can't do that by default. I went into the IAM role that Lambda auto-created and attached the **AmazonBedrockLimitedAccess** policy.

I specifically chose this policy over something broader like `AdministratorAccess` because it follows the **principle of least privilege** — the function only needs to talk to Bedrock, so I only gave it access to Bedrock. Granting more permissions than necessary is a security risk, and in a real production environment that kind of misconfiguration can be costly.

### API Gateway

API Gateway acts as the front door — every request from the frontend hits API Gateway first before it reaches Lambda.

When creating the method, I set the integration type to **Lambda proxy integration**. This means API Gateway passes the full HTTP request — headers, body, query parameters — directly to Lambda as-is, and Lambda returns the full response back. Without proxy integration, you'd have to manually map request and response fields, which adds complexity without much benefit for a project like this.

I also configured **CORS** (Cross-Origin Resource Sharing), which is required any time a browser-based frontend makes requests to a different domain or origin. I set `Access-Control-Allow-Headers` to `*` to allow any header from the frontend without restriction. In a production app you'd want to be more specific, but for this project it keeps things simple and functional.

After deploying the API to a **Prod** stage, I noted the **Invoke URL** — this is the endpoint the frontend uses to reach the backend.

### Connecting Lambda to Bedrock

With the infrastructure in place, I updated the Lambda function code to actually call Bedrock. Rather than hardcoding the model ID in the code, I set it as an **environment variable** (`BEDROCK_MODEL_ID`). This is a best practice — it keeps configuration separate from code, and makes it easy to swap models later without touching the function itself.

The model used: `us.anthropic.claude-opus-4-20250514-v1:0`

---

## Troubleshooting with CloudWatch

Getting the Lambda function to successfully call Bedrock wasn't immediate — I hit three distinct errors, which I tracked down using **CloudWatch Logs**. CloudWatch gave me the exact error messages coming out of Lambda, which pointed me to what needed to be fixed each time.

**Error 1: ResourceNotFoundException — use case details not submitted**
```
An error occurred (ResourceNotFoundException) when calling the InvokeModel operation:
Model use case details have not been submitted for this account.
```
The fix here wasn't technical — Anthropic requires you to fill out a use case form before accessing Claude models through Bedrock. Once submitted, access is granted within about 15 minutes.

**Error 2: ValidationException — on-demand throughput not supported**
```
An error occurred (ValidationException) when calling the InvokeModel operation:
Invocation of model ID anthropic.claude-opus-4-20250514-v1:0 with on-demand throughput isn't supported.
Retry your request with the ID or ARN of an inference profile that contains this model.
```
This one taught me the difference between a base model ID and an inference profile. The model ID I was using (`anthropic.claude-opus-4-20250514-v1:0`) doesn't support on-demand invocation directly. I needed to use the **cross-region inference profile** version, which has a region prefix: `us.anthropic.claude-opus-4-20250514-v1:0`. That `us.` prefix tells Bedrock to route through an inference profile, which does support on-demand throughput.

**Error 3: ResourceNotFoundException — legacy model access denied**
```
An error occurred (ResourceNotFoundException) when calling the InvokeModel operation:
Access denied. This Model is marked by provider as Legacy and you have not been actively using
the model in the last 15 days.
```
This error appeared when using an older model ID. The fix was making sure the environment variable `BEDROCK_MODEL_ID` was set to the correct, active inference profile ID: `us.anthropic.claude-opus-4-20250514-v1:0`.

Using CloudWatch throughout this process was key — without it I would have just been guessing. Being able to see the exact exception type and message from Lambda made each fix straightforward once I understood what the error was actually saying.

---

## What I Learned

- **IAM and least privilege matter from day one.** It's easy to just attach a broad policy and move on, but understanding *why* you scope permissions down is fundamental to working safely in the cloud.
- **API Gateway timeouts are easy to overlook.** The default Lambda timeout of 3 seconds seems fine until you're calling an LLM and wondering why every request is failing.
- **CORS errors are a rite of passage.** Every developer hits them. Understanding that they exist to protect users — and that you need to explicitly allow cross-origin requests — is an important concept for any cloud/web project.
- **CloudWatch is your best friend when debugging Lambda.** Without logs, you're guessing. CloudWatch surfaces the exact error coming out of your function, which makes troubleshooting systematic rather than random.
- **Environment variables over hardcoding.** Keeping configuration outside the code makes applications more flexible and easier to maintain.

---

## Note on Statefulness

This chatbot is **stateless** — each message is processed independently with no memory of previous messages. This is a natural next challenge: adding conversation history would require storing context somewhere (e.g., a session store or database) and passing it along with each request to Bedrock.

---

## Built With

- [AWS Lambda](https://aws.amazon.com/lambda/)
- [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
- [Amazon Bedrock](https://aws.amazon.com/bedrock/)
- [AWS IAM](https://aws.amazon.com/iam/)
- [AWS Amplify](https://aws.amazon.com/amplify/)
- Python 3.14

---

*Code and instructions were provided by GMU AWS Cloud Club. Deployment, configuration, and documentation by Blaise.*
