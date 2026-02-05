# Email-Based Time Tracking to Airtable

A production-ready n8n workflow that automatically processes time tracking emails and logs them to Airtable with intelligent deduplication and error handling.

## What This Does

This workflow monitors a Gmail account for time tracking emails (e.g., from automated daily logs, project management tools, or manual reports), extracts structured data from HTML-formatted content, and creates or updates records in Airtable. Perfect for consultants, freelancers, or teams that need automated time logging for invoicing.

### Key Features

- **Automated Email Processing**: Polls Gmail every 15 minutes for new messages
- **HTML Data Extraction**: Parses HTML comment markers to extract structured data
- **Smart Deduplication**: Searches Airtable before creating records to prevent duplicates
- **Robust Error Handling**: Labels emails that fail processing for manual review
- **Clean Inbox Management**: Archives and marks processed emails as read

## Use Cases

- Logging daily work summaries sent from project tracking systems
- Processing automated time reports from team members
- Capturing billable hours from email notifications
- Building a time-based invoicing system

## Data Flow

```
Gmail Trigger (every 15 min)
  → Get full email content
  → Parse HTML for data fields
  → Check if data extraction succeeded
    ├─ Success: Search Airtable for existing record
    │   ├─ Found: Update existing record
    │   └─ Not found: Create new record
    │       → Label email as processed
    │       → Mark as read
    │       → Archive email
    └─ Failed: Label as error → Stop workflow
```

## What Gets Extracted

The workflow expects emails with HTML comment markers and extracts:

- **Client** - Who the work was for
- **Date** - When the work was performed
- **Description** - What was done
- **Notes** - Additional context
- **RecordID** - Auto-generated unique identifier (Date + Client)

## Email Format Expected

Your emails should contain HTML with these comment markers:

```html
<!-- START_CLIENT -->Client Name<!-- END_CLIENT -->
<!-- START_DATE -->2024-01-15<!-- END_DATE -->
<!-- START_DESC -->Completed database optimization and cleanup<!-- END_DESC -->
<!-- START_NOTES -->Reduced query time by 40%<!-- END_NOTES -->
```

You can customize the markers in the JavaScript node to match your email format.

## Airtable Schema

Your Airtable table should have these fields:

| Field Name | Type | Description |
|------------|------|-------------|
| RecordID | Single line text | Unique identifier (auto-generated) |
| Date | Date | When work was performed |
| Client | Linked record | Client name (array format) |
| Description | Long text | Work description |
| Notes | Long text | Additional notes |

Additional fields like Hours, Billable, Invoiced, etc. can be added to your table - the workflow only updates the fields listed above.

## Setup Instructions

### 1. Prerequisites

- n8n instance (cloud or self-hosted)
- Gmail account with OAuth2 configured in n8n
- Airtable account with a Personal Access Token
- An Airtable base with a time tracking table

### 2. Import the Workflow

1. Download `workflow.json` from this repository
2. In n8n, click **"Import from File"**
3. Select the downloaded JSON file

### 3. Configure Credentials

Replace placeholder IDs with your actual credentials:

**Gmail OAuth2** (`YOUR_GMAIL_CREDENTIAL_ID`)
- Used in: Gmail Trigger, Get a message, all Gmail label operations
- Setup: n8n Settings → Credentials → Add Gmail OAuth2

**Airtable Personal Access Token** (`YOUR_AIRTABLE_CREDENTIAL_ID`)
- Used in: Search Time Logs, Update record, Create a record
- Setup: n8n Settings → Credentials → Add Airtable Personal Access Token
- Token needs: `data.records:read`, `data.records:write`, `schema.bases:read`

### 4. Configure Airtable IDs

Replace these placeholders with your actual IDs:

- `YOUR_AIRTABLE_BASE_ID` - Your Airtable base ID (starts with `app...`)
- `YOUR_AIRTABLE_TABLE_ID` - Your table ID (starts with `tbl...`)

**Finding these IDs:**
- Base ID: In Airtable, go to Help → API Documentation → find "The ID of this base is..."
- Table ID: Same API docs, or use Airtable's metadata API

### 5. Configure Gmail Labels (Optional)

Replace these placeholders:

- `YOUR_SUCCESS_LABEL_ID` - Label for successfully processed emails (or remove this node)
- `YOUR_ERROR_LABEL_ID` - Label for emails that failed processing

**Finding Label IDs:**
- Use n8n's Gmail "Get Labels" node to list all labels and their IDs
- Or use Gmail's API explorer

### 6. Customize Email Parsing (If Needed)

The JavaScript parsing node uses HTML comment markers. If your emails use different markers or format:

1. Open the **"Parse the Email (JavaScript)"** node
2. Modify the regex patterns to match your email format:

```javascript
const clientMatch = text.match(/<!-- START_CLIENT -->([\\s\\S]*?)<!-- END_CLIENT -->/)?.[1]?.trim();
```

Replace `<!-- START_CLIENT -->` and `<!-- END_CLIENT -->` with your markers.

### 7. Test the Workflow

1. Send a test email to your Gmail account with the expected format
2. Manually trigger the workflow in n8n
3. Check that data appears correctly in Airtable
4. Verify the email was labeled and archived

### 8. Activate

Once tested, activate the workflow - it will now run every 15 minutes automatically.

## Customization Options

### Change Polling Frequency

In the **Gmail Trigger** node, modify:
```javascript
"value": 15,  // Change to desired minutes
"unit": "minutes"
```

### Add Email Filters

In the **Gmail Trigger** node, add filters:
```javascript
"filters": {
  "labelIds": ["INBOX"],
  "q": "subject:Daily Log"  // Filter by subject, sender, etc.
}
```

### Map Additional Fields

In the **Create a record** and **Update record** nodes, add more field mappings:
```javascript
"Hours": "={{ $('Parse the Email (JavaScript)').item.json.hours }}",
"Billable": "={{ $('Parse the Email (JavaScript)').item.json.billable }}"
```

Don't forget to also extract these in the JavaScript parsing node.

## Error Handling

The workflow includes comprehensive error handling:

- **Parse Failures**: If data can't be extracted, email is labeled and workflow stops
- **Airtable Failures**: If record creation/update fails, email is labeled for review
- **Missing Data**: Gracefully handles missing fields with null values

Check your "error" labeled emails periodically to catch any issues.

## Production Tips

1. **Start with label filters** - Don't process all incoming email. Use Gmail labels or filters to only process time tracking emails.

2. **Test your HTML markers** - Send test emails and verify the JavaScript regex correctly extracts data before going live.

3. **Monitor error labels** - Set up a reminder to check error-labeled emails weekly.

4. **Backup strategy** - Airtable keeps revision history, but consider exporting time logs monthly as backup.

5. **Rate limits** - Gmail API has quotas. With 15-min polling, you're well under limits, but be aware if scaling up.

## Troubleshooting

**Workflow not triggering:**
- Check Gmail OAuth2 credentials are valid
- Verify polling is enabled (workflow is "Active")
- Check n8n execution logs for errors

**Data not extracted:**
- Test the email body contains HTML markers
- View the "Get a message" node output to see raw email text
- Adjust regex patterns if your format differs

**Airtable errors:**
- Verify Personal Access Token has write permissions
- Check field names match exactly (case-sensitive)
- Ensure linked record fields exist (Client field must link to a Clients table)

**Duplicates created:**
- RecordID formula relies on Date + Client being unique
- Adjust the RecordID generation logic if this doesn't fit your use case

## License

MIT - Feel free to use, modify, and distribute. Attribution appreciated but not required.

## About

Created by [Ken Davis](https://github.com/kendavisdev) at [Lodgepole I/O](https://lodgepole.io) - Workflow design and operational systems consulting.
