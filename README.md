# gmu-aws-cc-serverless
# Serverless AWS Chatbot

A fully serverless AI chatbot built on AWS, powered by Amazon Bedrock (Claude). Users can send messages through a web interface and receive AI-generated responses via a serverless backend.

> **Note:** This is a stateless chatbot — each message is processed independently. The bot does not retain conversation history between messages.

---

## Architecture

```
User → Amplify (Frontend) → API Gateway → Lambda → Bedrock (Claude)
                                                ↑
                                               IAM
```

| Service | Role |
|---|---|
| **API Gateway** | Front door — routes all incoming HTTP requests |
| **Lambda** | Backend logic — processes requests and calls Bedrock |
| **Amazon Bedrock** | LLM catalog — generates AI responses using Claude |
| **IAM** | Permissions — grants Lambda access to invoke Bedrock models |
| **Amplify** | Hosting — delivers the frontend to users |

---

## Prerequisites

- AWS account ([AWS Skill Builder](https://skillbuilder.aws/) recommended for free access)
- Basic familiarity with the AWS Console

---

## Setup

### 1. Lambda

1. In the AWS Console, search for **Lambda** and open it
2. Click **Create function**
   - Give it a name (e.g., `serverless-chatbot`)
   - Runtime: **Python 3.14**
   - Leave all other settings as default → click **Create Function**
3. Go to the **Configuration** tab
   - Under **General configuration** → increase timeout to **30 seconds**
4. Under **Configuration → Permissions**, click the IAM role link
   - Click **Add permissions → Attach policies**
   - Search for and attach **AmazonBedrockLimitedAccess**

### 2. API Gateway

1. Search for **API Gateway** and open it
2. Click **Create API → Build** (REST API)
   - Select **New API**
   - Give it a name → click **Build**
3. Click **Create Method**
   - Method type: **ANY**
   - Integration type: **Lambda function**
   - Toggle **Lambda proxy integration** ON
   - Select the Lambda function created above → click **Create Method**
4. Under Resources, click the slash (`/`) → click **Enable CORS**
   - Check **Default 4XX**, **Default 5XX**, and **GET**
   - Clear the `Access-Control-Allow-Headers` field and replace with `*`
   - Click **Save**
5. Click **Deploy API**
   - Stage: **New stage** → name it `Prod`
6. Note the **Invoke URL** — you'll need it shortly

### 3. Connect Lambda to Bedrock

1. Go back to your Lambda function
2. Replace the function code with the contents of [`lambda-basic`](https://github.com/aws-gmu/serverless/blob/main/lambda-basic)
3. Under **Configuration → Environment Variables → Edit → Add**:
   - Key: `BEDROCK_MODEL_ID`
   - Value: `us.anthropic.claude-opus-4-20250514-v1:0`
   - Click **Save**

### 4. Frontend

1. Download [`index-basic.html`](https://github.com/aws-gmu/serverless/blob/main/index-basic.html)
2. Open the file in a text editor
   - Find `<<INVOKE_URL>>` and replace it with your API Gateway Invoke URL
   - Save the file
3. Open the file in a browser — you now have a working chat interface

---

## Optional: Test via curl

```bash
curl -X POST https://<YOUR_INVOKE_URL>/prod \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Explain what AWS Lambda is in simple terms."}'
```

You should receive a JSON response containing the model's reply.

---

## Project Structure

```
├── lambda-basic        # Initial Lambda function (stateless)
├── lambda2             # Updated Lambda function
├── index-basic.html    # Basic frontend (stateless)
└── index.html          # Updated frontend
```

---

## Built With

- [AWS Lambda](https://aws.amazon.com/lambda/)
- [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
- [Amazon Bedrock](https://aws.amazon.com/bedrock/)
- [AWS Amplify](https://aws.amazon.com/amplify/)
- [AWS IAM](https://aws.amazon.com/iam/)
- Python 3.14

---

## License

This project is open source and available under the [MIT License](LICENSE).
