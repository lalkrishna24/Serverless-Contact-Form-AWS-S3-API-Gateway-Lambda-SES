# Serverless Contact Form — AWS S3 + API Gateway + Lambda + SES

A fully serverless web application where a static contact form hosted on Amazon S3 captures user input and delivers real-time email notifications — zero servers, zero infrastructure management.

---

## Architecture Overview

![Architecture Diagram](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520119533.jpeg)

```
                         ┌──────────────────────────────────────────────┐
                         │           Contact Form Logic                  │
                         │                                               │
                ┌───────►│  API Gateway (HTTP API)                       │
                │        │       POST /contact                           │
                │        │            │                                  │
User ───────────┤        │            ▼                                  │
                │        │     Lambda Function                           │
                │        │   (contact-form-handler)                      │
                │        │            │                                  │
                │        │            ▼                                  │
                │        │      Amazon SES                               │
                │        │   (send_email → verified identity)            │
                │        └──────────────────────────────────────────────┘
                │
                │        ┌──────────────────────────────────────────────┐
                │        │         Static HTML Content                   │
                └───────►│                                               │
                         │   S3 Bucket (static website hosting)          │
                         │   samplewebsite-hosting123                    │
                         └──────────────────────────────────────────────┘
```

**Request Flow:**
1. User opens the HTML contact form served from **S3 static website**
2. On form submit, JavaScript sends a **POST** request to **API Gateway**
3. API Gateway routes the request to the **Lambda function**
4. Lambda parses the form data and calls **Amazon SES** `send_email`
5. Email is delivered to the verified inbox in real time

---

## Services Used

| Service | Role |
|---|---|
| Amazon S3 | Static website hosting (HTML/CSS/JS) |
| Amazon API Gateway (HTTP API) | REST endpoint `POST /contact` |
| AWS Lambda | Python serverless backend — processes form & sends email |
| Amazon SES | Email delivery (sandbox mode, verified identities) |
| AWS IAM | Lambda execution role with `ses:SendEmail` permission |
| Amazon CloudWatch | Lambda logs for debugging |

**Region:** `ap-south-1` (Asia Pacific — Mumbai)

---

## Step 1 — Host Static Website on Amazon S3

### Create & Configure the Bucket

```bash
# Create bucket
aws s3 mb s3://samplewebsite-hosting123 --region ap-south-1

# Enable static website hosting
aws s3 website s3://samplewebsite-hosting123 \
  --index-document "Sample website hosting.html"
```

### Disable Block Public Access + Attach Bucket Policy

Navigated to **S3 → samplewebsite-hosting123 → Permissions → Block public access** and turned it **Off**, then applied this bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::samplewebsite-hosting123/*"
    }
  ]
}
```

![S3 Bucket Policy — PublicReadGetObject enabled](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118755.jpeg)

### HTML Contact Form (`Sample website hosting.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Contact Form</title>
</head>
<body>
  <input type="text"     id="name"    placeholder="Name">
  <input type="email"    id="email"   placeholder="Email">
  <textarea              id="message" placeholder="Message"></textarea>
  <button onclick="submitForm()">Send</button>

  <script>
    async function submitForm() {
      const payload = {
        name:    document.getElementById('name').value,
        email:   document.getElementById('email').value,
        message: document.getElementById('message').value
      };

      const res = await fetch(
        'https://rbnd2q8pl3.execute-api.ap-south-1.amazonaws.com/contact',
        {
          method:  'POST',
          headers: { 'Content-Type': 'application/json' },
          body:    JSON.stringify(payload)
        }
      );

      const data = await res.json();
      if (res.ok) {
        alert('Success: ' + data);
      } else {
        alert('Error: ' + data);
      }
    }
  </script>
</body>
</html>
```

Uploaded the file to the bucket:

```bash
aws s3 cp "Sample website hosting.html" s3://samplewebsite-hosting123/
```

![Live contact form served from S3](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118139.jpeg)

---

## Step 2 — Verify Email Identity in Amazon SES

Before Lambda can send emails, SES requires the sender address to be **verified** (in sandbox mode, the recipient must also be verified).

**Console path:** SES → Verified identities → Create identity → Email address

![SES verified identity — lalkrishnalal24@gmail.com](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118180.jpeg)

```
Identity status : ✅ Verified
ARN             : arn:aws:ses:ap-south-1:675254203925:identity/lalkrishnalal24@gmail.com
Region          : Asia Pacific (Mumbai)
```

> **SES Sandbox note:** In sandbox mode, both the sender and recipient email addresses must be individually verified. Production access requires a support request to AWS to lift the sandbox restriction.

---

## Step 3 — Create Lambda Function

### Function Setup

- **Function name:** `contact-form-handler`
- **Runtime:** Python 3.x
- **Execution role:** IAM role with `AmazonSESFullAccess` + `AWSLambdaBasicExecutionRole`

![Lambda function overview — contact-form-handler with API Gateway trigger](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118823.jpeg)

### Lambda Code — `lambda_function.py`

```python
import json
import boto3

ses = boto3.client('ses')

def lambda_handler(event, context):
    try:
        body = event.get('body')

        if isinstance(body, str):
            body = json.loads(body)

        name    = body.get('name')
        email   = body.get('email')
        message = body.get('message')

        response = ses.send_email(
            Source='lalkrishnalal24@gmail.com',
            Destination={
                'ToAddresses': ['lalkrishnalal24@gmail.com']
            },
            Message={
                'Subject': {
                    'Data': f'New message from {name}'
                },
                'Body': {
                    'Text': {
                        'Data': f'Name: {name}\nEmail: {email}\nMessage: {message}'
                    }
                }
            }
        )

        return {
            'statusCode': 200,
            'headers': {
                'Access-Control-Allow-Origin':  '*',
                'Access-Control-Allow-Headers': '*',
                'Access-Control-Allow-Methods': 'OPTIONS,POST'
            },
            'body': json.dumps('Email sent successfully!')
        }

    except Exception as e:
        print("Error:", str(e))
        return {
            'statusCode': 500,
            'headers': {
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps('Error sending email')
        }
```

![Lambda code — send_email call with name, email, message fields](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118873.jpeg)

![Lambda code — CORS headers in success and error responses](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118347.jpeg)

### IAM Execution Role Policy (inline)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ses:SendEmail",
      "Resource": "*"
    }
  ]
}
```

---

## Step 4 — Configure API Gateway (HTTP API)

### API Details

| Property | Value |
|---|---|
| API Name | `contact-api` |
| API Type | HTTP API |
| API ID | `rbnd2q8pl3` |
| Invoke URL | `https://rbnd2q8pl3.execute-api.ap-south-1.amazonaws.com` |

### Routes Created

```
/contact
├── POST     → Integration: contact-form-handler (Lambda)
└── OPTIONS  → CORS preflight
```

![API Gateway routes — POST and OPTIONS on /contact](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118814.jpeg)

**Route ARN:**
```
arn:aws:apigateway:ap-south-1::/apis/rbnd2q8pl3/routes/rhpguw9
```

### CORS Configuration

Configured at the API level (HTTP API CORS settings):

```
Allow Origins  : *
Allow Methods  : POST, OPTIONS
Allow Headers  : Content-Type
Max Age        : 300
```

> The OPTIONS route handles preflight requests from the browser before the actual POST fires. Without this, all cross-origin form submissions will be blocked by the browser's CORS policy.

### Deploy API

```
Stage Name : $default (auto-deploy enabled)
```

---

## Step 5 — End-to-End Test

### Submitting the Contact Form

Opened the live S3 URL in a browser, filled in the form fields, and clicked **Send**:

```
URL     : samplewebsite-hosting123.s3.ap-south-1.amazonaws.com/Sample+website+hosting.html
Name    : Madan
Email   : pubgmadan24@gmail.com
Message : Hi this is a sample website message
```

![Form submission — "Email sent successfully!" alert](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520118457.jpeg)

### Email Received in Gmail

The Lambda function successfully triggered SES, and the email arrived (routed to spam on first delivery — expected SES sandbox behaviour; marking "Not spam" resolves this in subsequent sends):

```
From    : lalkrishnalal24@gmail.com via amazonses.com
Subject : New message from Madan
Body    :
  Name: Madan
  Email: pubgmadan24@gmail.com
  Message: Hi this is a sample website message
```

![Email received in Gmail — delivered via amazonses.com](https://raw.githubusercontent.com/lalkrishna24/cncn/97e2c22fd54c09ebd0ab87cc5b7cfaf44ad5d7b6/1774520119104.jpeg)

---

## Debugging & Troubleshooting

### CloudWatch Logs

All Lambda errors and print statements are captured in CloudWatch:

```bash
# View Lambda logs via CLI
aws logs tail /aws/lambda/contact-form-handler --follow
```

### Issues Encountered & Fixed

| Issue | Root Cause | Fix |
|---|---|---|
| `CORS error` in browser | Missing `Access-Control-Allow-Origin` header | Added CORS headers in both success and error Lambda responses |
| `500 Internal Server Error` | `body` was a JSON string, not a dict | Added `json.loads(body)` when `isinstance(body, str)` |
| Email landed in spam | New SES identity + first send | Expected in sandbox; mark "Not spam" to train Gmail |
| OPTIONS route missing | HTTP API doesn't auto-create preflight routes | Manually created `OPTIONS /contact` route in API Gateway |

### cURL Test Command

```bash
curl -X POST \
  https://rbnd2q8pl3.execute-api.ap-south-1.amazonaws.com/contact \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","email":"test@example.com","message":"Hello from cURL"}'
```

Expected response:
```json
"Email sent successfully!"
```

---

## Project Structure

```
serverless-contact-form/
├── lambda_function.py          # Python Lambda handler
├── Sample website hosting.html # Static HTML contact form
├── s3-bucket-policy.json       # Public read policy for S3
└── README.md
```

---

## Key Learnings

- **Serverless glue pattern:** API Gateway + Lambda + SES is a clean, cost-free-at-low-scale backend for any static site's contact form — no EC2, no web server.
- **CORS must be handled in two places:** at the API Gateway level (preflight OPTIONS route) AND in the Lambda response headers. Missing either one causes browser-side failures.
- **SES sandbox restrictions** mean both sender and recipient must be verified. Planning for production access removal (via AWS Support) before going live is essential.
- **Lambda body parsing:** API Gateway HTTP API sends the body as a JSON string when the request has `Content-Type: application/json`. Always check `isinstance(body, str)` and call `json.loads()` before accessing fields.
- **CloudWatch Logs** are indispensable — every `print()` in Lambda becomes a log entry, making debugging 500 errors fast and precise.
- **IAM least privilege** matters even in serverless: the Lambda role was scoped to only `ses:SendEmail` rather than broad admin access.

---

## References

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Amazon SES — Sending Email](https://docs.aws.amazon.com/ses/latest/dg/send-email.html)
- [API Gateway HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html)
- [S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [Configuring CORS for HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-cors.html)

---

## Tags

`#AWS` `#Serverless` `#Lambda` `#APIGateway` `#AmazonSES` `#AmazonS3` `#Python` `#CORS` `#StaticWebsite` `#CloudWatch` `#IAM` `#ap-south-1` `#CloudNative` `#ContactForm` `#HandsOn`
