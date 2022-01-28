
# Project Intent

This project spins up a Lambda function that is triggered by a Event Bridge Rule. This uses a cron job to execute once daily. 

It will calculate the monthly total spend of an AWS Organization and break out the spend via each account within the org. 

You can configure a daily change threshold, to monitor spikes in usage. For instance, if you know your daily charges are $100, set the threshold to $120 to be alerted to spikes. 

This will send data to Slack Web Hooks as well as email addresses. You will need to generate a SES email credential and use those in this JSON configuration. 

This is the JSON structure to configure the Slack as well as email configuration: 

```json
{
    "emails": [
      ""
    ],
    "slack": [
      ""
    ],
    "sesCredentials": {
      "accessKey": "",
      "secretKey": "",
      "from": "no-reply@example.com"
    }
  }
```