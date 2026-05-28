# Phishing Email Analyzer

A cybersecurity automation project built with **n8n**, **Gmail**, **Google Sheets**, **Telegram**, and **VirusTotal**.

This project demonstrates how to build an automated phishing email analysis workflow that receives suspicious emails, extracts URLs and phishing indicators, checks URL reputation with VirusTotal, calculates a phishing risk score, saves the analysis report in Google Sheets, and sends a Telegram alert when the email is classified as high risk.

---

## Project Overview

The goal of this project is to automate the initial analysis of suspicious emails.

The workflow receives an email from Gmail, normalizes the email data, extracts URLs and suspicious indicators, checks the first detected URL with VirusTotal, calculates a phishing risk score, saves the report in Google Sheets, and sends a Telegram alert when the risk level is high.

The workflow supports two analysis paths:

1. **Emails with URLs**

   * URLs are extracted.
   * The first URL is submitted to VirusTotal.
   * The VirusTotal analysis result is used in the final risk score.

2. **Emails without URLs**

   * The email is analyzed using phishing-related keywords, sender indicators, and suspicious language patterns.
   * The report is still saved and alerts can still be triggered if the risk is high.

This makes the workflow useful not only for link-based phishing attempts, but also for social engineering emails that ask the user to reply, share credentials, or take urgent action.

---

## Tech Stack

* **n8n Cloud** — workflow automation platform
* **Gmail Trigger** — receives suspicious emails
* **Google Sheets** — stores phishing analysis reports
* **Telegram Bot API** — sends high-risk phishing alerts
* **VirusTotal API** — checks URL reputation
* **Code Nodes** — extracts URLs, detects indicators, and calculates risk score
* **IF Nodes** — controls branching logic
* **Wait Node** — waits for VirusTotal analysis completion

---

## Main Features

### Gmail-Based Email Intake

The workflow starts when an email is received or selected through Gmail.

For demo usage, emails can be filtered using a Gmail label such as:

```txt
Phishing-Check
```

This allows suspicious emails to be reviewed on demand without analyzing every incoming email.

---

### Email Data Normalization

The workflow normalizes raw Gmail data into a clean structure.

Normalized fields include:

```txt
reportId
timestamp
sender
subject
body
messageId
```

This makes the rest of the workflow easier to maintain and debug.

---

### URL Extraction

The workflow extracts URLs from the email body using a Code node.

Generated fields include:

```txt
detectedUrls
firstUrl
urlCount
hasUrls
```

For the first version of this project, only the first detected URL is submitted to VirusTotal.

---

### Suspicious Indicator Detection

The workflow detects common phishing-related keywords and suspicious patterns.

Examples of suspicious keywords:

```txt
urgent
immediately
verify
verification
password
account suspended
account closure
login
security alert
unusual activity
confirm your account
update your payment
payment failed
click here
limited time
within 24 hours
reset your password
billing problem
unauthorized access
```

The workflow also detects sender-related indicators, such as:

```txt
no-reply sender
security/support sender wording
brand-related sender wording
```

---

### VirusTotal URL Reputation Check

When the email contains at least one URL, the workflow sends the first detected URL to VirusTotal.

The VirusTotal path includes:

```txt
Submit URL to VirusTotal
↓
Wait for analysis
↓
Get VirusTotal analysis result
```

VirusTotal fields used in the scoring logic include:

```txt
malicious
suspicious
harmless
undetected
analysis status
```

---

### Risk Score Calculation

The phishing risk score is calculated using multiple factors.

Example scoring factors:

```txt
Email contains URL(s)
Multiple URLs detected
Suspicious phishing keywords detected
Suspicious sender wording detected
URL uses a raw IP address
VirusTotal malicious detections
VirusTotal suspicious detections
VirusTotal analysis status
```

The final score is capped at:

```txt
100
```

---

### Risk Level Classification

The workflow classifies the email based on the calculated risk score.

```txt
0 - 39    → Low
40 - 69   → Medium
70 - 100  → High
```

---

### Google Sheets Reporting

Every analyzed email is saved in Google Sheets.

This creates a simple phishing analysis log that can be reviewed later.

Stored information includes:

```txt
Report ID
Timestamp
Sender
Subject
URL Count
Detected URLs
Suspicious Keywords
VirusTotal Malicious
VirusTotal Suspicious
Risk Score
Risk Level
Recommendation
Action Taken
Notes
```

---

### Telegram High-Risk Alert

If the email is classified as high risk, the workflow sends a Telegram alert.

Example alert:

```txt
🚨 Phishing Email Alert

A high-risk phishing email was detected.

Risk Level: High
Risk Score: 85/100

Sender: fake-security@example.com
Subject: Urgent: Verify your account immediately

URL Count: 1
Detected URLs:
http://fake-login-security-example.com/verify

Suspicious Keywords:
urgent, verify, password, account suspended

VirusTotal:
Malicious: 1
Suspicious: 0
Status: completed

Recommendation:
Do not click any link. Report or delete the email and consider blocking the sender/domain.
```

---

## Workflow Structure

The workflow contains 11 main nodes plus one additional branch for emails without URLs.

```txt
01 Gmail Trigger - Receive Suspicious Email
02 Set - Normalize Email Data
03 Code - Extract URLs and Indicators
04 IF - URLs Found?
05 HTTP Request - Submit URL to VirusTotal
06 Wait - Wait for VirusTotal Analysis
07 HTTP Request - Get VirusTotal Analysis
08 Code - Calculate Phishing Risk Score
08B Code - Calculate Risk Without URL
09 Google Sheets - Save Analysis Report
10 IF - Is High Risk?
11 Telegram - Send Phishing Alert
```

---

## Workflow Flow

```txt
01 Gmail Trigger - Receive Suspicious Email
↓
02 Set - Normalize Email Data
↓
03 Code - Extract URLs and Indicators
↓
04 IF - URLs Found?
   ├── true
   │   ↓
   │   05 HTTP Request - Submit URL to VirusTotal
   │   ↓
   │   06 Wait - Wait for VirusTotal Analysis
   │   ↓
   │   07 HTTP Request - Get VirusTotal Analysis
   │   ↓
   │   08 Code - Calculate Phishing Risk Score
   │
   └── false
       ↓
       08B Code - Calculate Risk Without URL

08 / 08B
↓
09 Google Sheets - Save Analysis Report
↓
10 IF - Is High Risk?
   ├── true
   │   ↓
   │   11 Telegram - Send Phishing Alert
   └── false
       end
```

---

## Node Details

### 01 Gmail Trigger - Receive Suspicious Email

This node receives suspicious emails from Gmail.

Recommended demo setup:

```txt
Trigger: New Email / Message Received
Label: Phishing-Check
Polling: Every minute
```

Using a dedicated Gmail label allows manual control over which emails are analyzed.

---

### 02 Set - Normalize Email Data

This node creates a clean and consistent email object.

Fields:

```txt
reportId
timestamp
sender
subject
body
messageId
```

Example expressions:

```txt
reportId: {{ $execution.id }}
timestamp: {{ $now.format('yyyy-MM-dd HH:mm:ss') }}
sender: {{ $json.from }}
subject: {{ $json.subject }}
body: {{ $json.textPlain || $json.plainText || $json.text || $json.textHtml || $json.html || $json.snippet || '' }}
messageId: {{ $json.id || $json.messageId || $json.threadId || '' }}
```

---

### 03 Code - Extract URLs and Indicators

This node extracts URLs and suspicious indicators from the email.

Output fields:

```txt
detectedUrls
firstUrl
urlCount
suspiciousKeywords
suspiciousKeywordCount
senderIndicators
rawIpUrls
hasUrls
hasSuspiciousKeywords
hasRawIpUrl
```

The workflow uses:

```txt
firstUrl
```

for the VirusTotal analysis.

---

### 04 IF - URLs Found?

This node checks if the email contains URLs.

Condition:

```txt
{{ $json.hasUrls }} is true
```

If true, the workflow sends the first URL to VirusTotal.

If false, the workflow skips VirusTotal and calculates risk without URL reputation data.

---

### 05 HTTP Request - Submit URL to VirusTotal

This node submits the first detected URL to VirusTotal.

Configuration:

```txt
Method: POST
URL: https://www.virustotal.com/api/v3/urls
Body Content Type: Form URLencoded
```

Headers:

```txt
x-apikey: YOUR_VIRUSTOTAL_API_KEY
accept: application/json
```

Body parameter:

```txt
url: {{ $json.firstUrl }}
```

Important: never commit API keys to GitHub and never expose them in screenshots.

---

### 06 Wait - Wait for VirusTotal Analysis

This node waits before requesting the VirusTotal analysis result.

Recommended wait time:

```txt
20 seconds
```

If VirusTotal returns `queued` or `in-progress`, increase the wait time to:

```txt
30-45 seconds
```

---

### 07 HTTP Request - Get VirusTotal Analysis

This node retrieves the VirusTotal analysis result.

Configuration:

```txt
Method: GET
URL: https://www.virustotal.com/api/v3/analyses/{{ $('05 HTTP Request - Submit URL to VirusTotal').item.json.data.id }}
```

Headers:

```txt
x-apikey: YOUR_VIRUSTOTAL_API_KEY
accept: application/json
```

Important output fields:

```txt
data.attributes.status
data.attributes.stats.malicious
data.attributes.stats.suspicious
data.attributes.stats.harmless
data.attributes.stats.undetected
```

---

### 08 Code - Calculate Phishing Risk Score

This node calculates the risk score for emails that contain URLs.

Inputs:

```txt
Email indicators from node 03
VirusTotal analysis from node 07
```

Output fields:

```txt
riskScore
riskLevel
riskReasons
recommendation
actionTaken
vtStatus
vtMalicious
vtSuspicious
vtHarmless
vtUndetected
```

Risk logic:

```txt
URL present → adds risk
Multiple URLs → adds risk
Suspicious keywords → adds risk
Suspicious sender indicators → adds risk
Raw IP URL → adds risk
VirusTotal malicious detections → adds risk
VirusTotal suspicious detections → adds risk
```

---

### 08B Code - Calculate Risk Without URL

This node handles emails without URLs.

It calculates risk using:

```txt
Suspicious keywords
Sender indicators
Urgency language
Password-related wording
Account suspension or closure wording
Payment or billing wording
Verification language
```

This branch still outputs the same fields as node 08, so the workflow can continue into Google Sheets and Telegram without breaking.

VirusTotal fields are set to fallback values:

```txt
vtStatus: not_applicable
vtMalicious: 0
vtSuspicious: 0
vtHarmless: 0
vtUndetected: 0
```

---

### 09 Google Sheets - Save Analysis Report

This node saves the phishing analysis report in Google Sheets.

Document name:

```txt
Phishing Email Analysis
```

Sheet name:

```txt
Reports
```

Columns:

```txt
Report ID
Timestamp
Sender
Subject
URL Count
Detected URLs
Suspicious Keywords
VirusTotal Malicious
VirusTotal Suspicious
Risk Score
Risk Level
Recommendation
Action Taken
Notes
```

Example mappings:

```txt
Report ID: {{ $json.reportId }}
Timestamp: {{ $json.timestamp }}
Sender: {{ $json.sender }}
Subject: {{ $json.subject }}
URL Count: {{ $json.urlCount }}
Detected URLs: {{ $json.detectedUrls.join(', ') }}
Suspicious Keywords: {{ $json.suspiciousKeywords.join(', ') }}
VirusTotal Malicious: {{ $json.vtMalicious }}
VirusTotal Suspicious: {{ $json.vtSuspicious }}
Risk Score: {{ $json.riskScore }}
Risk Level: {{ $json.riskLevel }}
Recommendation: {{ $json.recommendation }}
Action Taken: {{ $json.actionTaken }}
Notes: {{ $json.riskReasons.join(' | ') }}
```

---

### 10 IF - Is High Risk?

This node checks if the analyzed email is high risk.

Condition:

```txt
{{ $json.riskLevel }} equals High
```

If true, Telegram sends an alert.

If false, the workflow ends after saving the report.

---

### 11 Telegram - Send Phishing Alert

This node sends an alert when a high-risk phishing email is detected.

Recommended message:

```txt
🚨 Phishing Email Alert

A high-risk phishing email was detected.

Risk Level: {{ $json.riskLevel }}
Risk Score: {{ $json.riskScore }}/100

Sender: {{ $json.sender }}
Subject: {{ $json.subject }}

URL Count: {{ $json.urlCount }}
Detected URLs:
{{ Array.isArray($json.detectedUrls) ? $json.detectedUrls.join('\n') : $json.detectedUrls }}

Suspicious Keywords:
{{ Array.isArray($json.suspiciousKeywords) ? $json.suspiciousKeywords.join(', ') : $json.suspiciousKeywords }}

VirusTotal:
Malicious: {{ $json.vtMalicious }}
Suspicious: {{ $json.vtSuspicious }}
Status: {{ $json.vtStatus }}

Risk Reasons:
{{ Array.isArray($json.riskReasons) ? $json.riskReasons.join('\n') : $json.riskReasons }}

Recommendation:
{{ $json.recommendation }}
```

---

## Google Sheets Structure

Google Sheet:

```txt
Phishing Email Analysis
```

Tab:

```txt
Reports
```

Columns:

```txt
Report ID
Timestamp
Sender
Subject
URL Count
Detected URLs
Suspicious Keywords
VirusTotal Malicious
VirusTotal Suspicious
Risk Score
Risk Level
Recommendation
Action Taken
Notes
```

Example row:

| Report ID | Timestamp           | Sender                                                        | Subject                                 | URL Count | Detected URLs                                 | Suspicious Keywords      | VirusTotal Malicious | VirusTotal Suspicious | Risk Score | Risk Level | Recommendation                                     | Action Taken             | Notes                          |
| --------- | ------------------- | ------------------------------------------------------------- | --------------------------------------- | --------: | --------------------------------------------- | ------------------------ | -------------------: | --------------------: | ---------: | ---------- | -------------------------------------------------- | ------------------------ | ------------------------------ |
| 128       | 2026-05-28 20:30:00 | [fake-security@example.com](mailto:fake-security@example.com) | Urgent: Verify your account immediately |         1 | http://fake-login-security-example.com/verify | urgent, verify, password |                    1 |                     0 |         87 | High       | Do not click any link. Report or delete the email. | High-risk alert prepared | Email contains external URL(s) |

---

## Test Emails

### Test 1 — Phishing Email With URL

Subject:

```txt
Urgent: Verify your account immediately
```

Body:

```txt
Hello,

Your account has been suspended. Please verify your password immediately by clicking this link:

http://fake-login-security-example.com/verify

Failure to act within 24 hours will result in account closure.

Security Team
```

Expected result:

```txt
URL is extracted
VirusTotal analysis runs
Risk score is calculated
Report is saved in Google Sheets
Telegram alert is sent if risk level is High
```

---

### Test 2 — Phishing Email Without URL

Subject:

```txt
Urgent: Your account has been suspended
```

Body:

```txt
Hello,

Your account has been suspended due to unusual activity.

Please verify your password immediately by replying to this email with your account information.

Failure to act within 24 hours will result in account closure.

Security Team
```

Expected result:

```txt
No URL is detected
VirusTotal is skipped
Risk is calculated using suspicious keywords and message patterns
Report is saved in Google Sheets
Telegram alert is sent if risk level is High
```

---

### Test 3 — Low-Risk Email

Subject:

```txt
Meeting notes
```

Body:

```txt
Hi,

Here are the meeting notes from today.

Thanks.
```

Expected result:

```txt
No URL or suspicious keywords detected
Risk level is Low
Report is saved in Google Sheets
No Telegram alert is sent
```

---

## Business and Security Value

This project demonstrates a practical phishing triage workflow.

It helps automate repetitive security checks by:

* collecting suspicious emails
* extracting URLs
* detecting phishing-related language
* checking URL reputation with VirusTotal
* calculating a risk score
* logging every analysis report
* sending alerts only for high-risk emails

This type of workflow can support:

* small business security monitoring
* internal IT support teams
* cybersecurity portfolio projects
* SOC automation demos
* email security awareness workflows
* manual phishing triage processes

---

## Possible Real-World Use Cases

This workflow can be adapted for:

* phishing mailbox monitoring
* suspicious email reporting
* user-submitted phishing checks
* URL reputation triage
* security awareness programs
* internal help desk workflows
* automated incident notification
* lightweight SOC automation

---

## Future Improvements

Possible upgrades:

* Analyze all URLs instead of only the first URL
* Add URLScan.io integration
* Add attachment scanning
* Add sender domain reputation checks
* Add DMARC/SPF/DKIM validation
* Add domain age lookup
* Add screenshot capture of suspicious URLs
* Add Slack or Microsoft Teams alerts
* Add Jira/Trello/Notion incident ticket creation
* Add AI-generated phishing summary
* Add multi-level alerting for Medium and High risk
* Add whitelist/allowlist for trusted senders
* Add automatic Gmail label update after analysis
* Add error handling for VirusTotal API limits
* Add dashboard reporting
* Add weekly phishing summary report

---

## Security Notes

Before using this in production:

* Protect API keys and credentials.
* Never publish VirusTotal API keys.
* Avoid exposing real email content in screenshots.
* Use a dedicated mailbox or Gmail label for analysis.
* Review privacy and data retention requirements.
* Do not rely only on this workflow for email security.
* Add error handling for API failures.
* Add authentication and validation where needed.
* Review third-party API terms before processing real data.
* Avoid sending sensitive email content to external services unless approved.

---

## Project Status

Current version: working demo

Implemented:

* Gmail email intake
* Email normalization
* URL extraction
* Suspicious keyword detection
* VirusTotal URL analysis
* Risk score calculation
* No-URL risk analysis branch
* Google Sheets report logging
* High-risk IF logic
* Telegram phishing alert

---

## Author

Built by **Iosif Castrucci**

GitHub: `iosif-castrucci-hub`
Email: `contact.iosifcastrucci@gmail.com`


---

## Disclaimer

This project is a demo cybersecurity automation workflow created for portfolio and educational purposes.

It is not a complete phishing detection platform and should not be used as the only method for detecting malicious emails in production environments.

For production use, add proper email security controls, API error handling, privacy review, secure credential management, logging, monitoring, and incident response processes.

---

## License

This repository is intended for portfolio and educational purposes.
You may adapt the workflow structure for your own projects.
