PART 2: THE CORE WORKFLOW (n8n Automation)

Step 3.1: Import This Complete Workflow
In n8n:
Click "Workflows" → "Import from File" → OR paste this JSON
JSON
{
  "name": "AI Review Monitor & Responder",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "hours",
              "hoursInterval": 1
            }
          ]
        }
      },
      "id": "schedule-trigger",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "operation": "search",
        "base": {
          "__rl": true,
          "value": "YOUR_BASE_ID",
          "mode": "list"
        },
        "table": {
          "__rl": true,
          "value": "Clients",
          "mode": "list"
        },
        "options": {
          "filterByFormula": "{Status} = 'Active'"
        }
      },
      "id": "airtable-get-clients",
      "name": "Airtable - Get Active Clients",
      "type": "n8n-nodes-base.airtable",
      "typeVersion": 2,
      "position": [450, 300],
      "credentials": {
        "airtableTokenApi": {
          "id": "YOUR_CREDENTIAL_ID",
          "name": "Airtable account"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// Split into items for processing\nreturn items[0].json.map(item => ({\n  json: {\n    clientId: item.id,\n    clientName: item.fields['Client Name'],\n    placeId: item.fields['Google Place ID'],\n    tone: item.fields['Review Response Tone'] || 'Professional and friendly',\n    ownerEmail: item.fields['Owner Email']\n  }\n}));"
      },
      "id": "split-clients",
      "name": "Split Clients",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [650, 300]
    },
    {
      "parameters": {
        "url": "=https://maps.googleapis.com/maps/api/place/details/json?place_id={{ $json.placeId }}&fields=reviews&key=YOUR_GOOGLE_API_KEY",
        "options": {}
      },
      "id": "google-reviews",
      "name": "Google Places API",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [850, 300]
    },
    {
      "parameters": {
        "jsCode": "const reviews = $input.first().json.result?.reviews || [];\nconst newReviews = [];\n\nfor (const review of reviews) {\n  // Check if review is less than 2 hours old\n  const reviewTime = review.time * 1000;\n  const twoHoursAgo = Date.now() - (2 * 60 * 60 * 1000);\n  \n  if (reviewTime > twoHoursAgo && !review.text?.includes('(Responded by AI)')) {\n    newReviews.push({\n      json: {\n        clientId: $input.first().json.clientId,\n        clientName: $input.first().json.clientName,\n        placeId: $input.first().json.placeId,\n        tone: $input.first().json.tone,\n        ownerEmail: $input.first().json.ownerEmail,\n        reviewId: review.author_url || review.time,\n        author: review.author_name,\n        rating: review.rating,\n        text: review.text,\n        reviewTime: review.time\n      }\n    });\n  }\n}\n\nreturn newReviews;"
      },
      "id": "filter-new-reviews",
      "name": "Filter New Reviews",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1050, 300]
    },
    {
      "parameters": {
        "model": "gpt-4.1",
        "messages": {
          "message": [
            {
              "role": "system",
              "content": "=You are a professional reputation manager. Write a response to this Google review for {{ $json.clientName }}. \n\nBusiness context: {{ $json.tone }}\n\nRules:\n- Keep it under 150 words\n- Thank the reviewer by name\n- Address specific points they mentioned\n- For 1-2 star reviews: apologize, acknowledge the issue, invite them to contact us directly\n- For 3 stars: acknowledge mixed feedback, mention improvements\n- For 4-5 stars: express gratitude, mention looking forward to their return\n- Never be defensive\n- Sign as the business owner\n- Do NOT include '(Responded by AI)' in the response"
            },
            {
              "role": "user",
              "content": "=Review by {{ $json.author }} ({{ $json.rating }} stars):\n\n{{ $json.text }}"
            }
          ]
        },
        "options": {}
      },
      "id": "openai-generate",
      "name": "OpenAI - Generate Response",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [1250, 300],
      "credentials": {
        "openAiApi": {
          "id": "YOUR_OPENAI_CRED_ID",
          "name": "OpenAI account"
        }
      }
    },
    {
      "parameters": {
        "fromEmail": "reviews@yourdomain.com",
        "toEmail": "={{ $json.ownerEmail }}",
        "subject": "=New {{ $json.rating }}-Star Review - Response Ready for Approval",
        "text": "=Hi there,\n\nA new {{ $json.rating }}-star review was posted for {{ $json.clientName }} by {{ $json.author }}.\n\nReview:\n\"{{ $json.text }}\"\n\nSuggested Response:\n\"{{ $json.response }}\"\n\nTo approve and post: Reply \"APPROVE\"\nTo edit: Reply with your edited version\nTo skip: Reply \"SKIP\"\n\n---\nAI Review Agent"
      },
      "id": "email-owner",
      "name": "Email Owner for Approval",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2,
      "position": [1450, 200],
      "credentials": {
        "smtp": {
          "id": "YOUR_SMTP_CRED_ID",
          "name": "SMTP account"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// Store pending approval in memory or database\n// For MVP, we'll use a simple webhook approach\n\nreturn [{\n  json: {\n    clientId: $input.first().json.clientId,\n    reviewId: $input.first().json.reviewId,\n    response: $input.first().json.response,\n    status: 'pending_approval',\n    timestamp: new Date().toISOString()\n  }\n}];"
      },
      "id": "store-pending",
      "name": "Store Pending Approval",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1450, 400]
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "Airtable - Get Active Clients",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Airtable - Get Active Clients": {
      "main": [
        [
          {
            "node": "Split Clients",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Split Clients": {
      "main": [
        [
          {
            "node": "Google Places API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Places API": {
      "main": [
        [
          {
            "node": "Filter New Reviews",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Filter New Reviews": {
      "main": [
        [
          {
            "node": "OpenAI - Generate Response",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "OpenAI - Generate Response": {
      "main": [
        [
          {
            "node": "Email Owner for Approval",
            "type": "main",
            "index": 0
          },
          {
            "node": "Store Pending Approval",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
Setup Steps:
Replace YOUR_BASE_ID with your Airtable Base ID
Replace YOUR_GOOGLE_API_KEY with your Google Places API key
Replace YOUR_CREDENTIAL_ID with actual credential IDs from n8n
Replace reviews@yourdomain.com with your email
Setup SMTP credentials in n8n (Gmail or SendGrid)

Step 3.2: Setup SMTP (Email Sending)
Option A: Gmail (Free, 100 emails/day)
n8n Credentials → SMTP
Host: smtp.gmail.com
Port: 587
User: your Gmail
Password: Use App Password (not regular password)
Secure: TLS
Option B: SendGrid (Recommended for scale)
Sign up at sendgrid.com
Free tier: 100 emails/day
Create API Key
n8n Credentials → SMTP
Host: smtp.sendgrid.net
Port: 587
User: apikey
Password: Your SendGrid API Key

Step 3.3: Approval Webhook (Reply to Post)
Create a second workflow in n8n:
Webhook Trigger → Parse Email → Post to Google
JSON
{
  "name": "Review Approval Handler",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "review-approval",
        "responseMode": "responseNode"
      },
      "name": "Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "jsCode": "// Parse the email reply\nconst body = $input.first().json.body;\nconst subject = $input.first().json.subject || '';\n\n// Extract review ID from subject or body\nconst reviewIdMatch = subject.match(/Review ID: (\\w+)/);\nconst reviewId = reviewIdMatch ? reviewIdMatch[1] : null;\n\nconst isApproved = body.toLowerCase().includes('approve');\nconst isSkipped = body.toLowerCase().includes('skip');\n\nlet response = body;\nif (isApproved) {\n  response = null; // Use AI generated response\n}\n\nreturn [{\n  json: {\n    reviewId,\n    isApproved,\n    isSkipped,\n    response,\n    rawBody: body\n  }\n}];"
      },
      "name": "Parse Email",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [450, 300]
    }
  ],
  "connections": {
    "Webhook": {
      "main": [
        [
          {
            "node": "Parse Email",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
Note: For production, use a service like Mailgun or SendGrid Inbound Parse to receive emails and forward to your webhook.