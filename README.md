# AI Expense Tracker — n8n & Local AI

An automated personal expense-tracking project built with **n8n**, **Ollama**, **JavaScript**, and **Google Workspace**. It reads invoice attachments received by email, extracts expense information using a locally hosted vision-language model, stores the original files in Google Drive, and records structured data in Google Sheets for financial analysis.

> **Project status:** Personal learning project / work in progress. The workflow is not production-hardened. Review extracted values before relying on the dashboard for financial decisions.

## Features

- Polls Gmail for unread messages and checks the sender and subject (`n8n`, case-insensitive).
- Extracts supported attachments from incoming emails (PDF, JPEG, PNG are permitted by the filtering code).
- Processes attachments sequentially using **Loop Over Items** (batch size: 1).
- Uses **Ollama** with `qwen2.5vl:3b` to extract invoice fields from images.
- Parses the model's JSON response and performs basic date and total validation using JavaScript.
- Uploads original attachments to Google Drive.
- Matches the extraction results with the corresponding Drive file using an `attachmentId`.
- Appends expense records to a Google Sheets spreadsheet, which can power a dashboard showing weekly, monthly, yearly, and per-merchant spending.
- Marks the email as read after a successful append step (see limitations below).

## Architecture

```text
Gmail Trigger (unread messages)
            |
      Sender + subject IF
            |
     Extract attachments
            |
  Loop Over Items (batch = 1)
            |
       +----+----------------+
       |                     |
  Ollama: Analyze image   Google Drive: Upload
       |                     |
  Parse + validate        Restore attachmentId
       |                     |
       +------ Merge by attachmentId
                       |
                Map sheet columns
                       |
             Google Sheets: Append row
                       |
                Mark email as read
                       |
                Next loop item
```

The **Google Sheets dashboard is configured separately from the n8n workflow**. The exported workflow contains the data-ingestion automation; it does not create dashboard charts or spreadsheet formulas automatically.

## Tech stack

| Component | Technology |
| --- | --- |
| Workflow automation | n8n (developed with self-hosted v2.43.5) |
| Runtime | Docker / Docker Desktop |
| Local AI | Ollama + `qwen2.5vl:3b` |
| Data processing | JavaScript (n8n Code nodes) |
| Email input | Gmail / Google OAuth |
| File storage | Google Drive |
| Expense data and reporting | Google Sheets |

## Extracted invoice fields

The model is instructed to produce JSON with the following fields:

```json
{
  "supplier": "Example Store Ltd",
  "supplierNif": "PT000000000",
  "invoiceNumber": "EXAMPLE-001",
  "invoiceDate": "2026-08-18",
  "total": 26.73,
  "atcud": null
}
```

The validation step also adds `hasValidDate`, `hasValidTotal`, and `status` (`validated` or `pending_review`). The workflow associates each processed attachment with its Google Drive file via `attachmentId` and records the filename, file ID, and file URL. **`validated` only means the basic format checks passed; it does not prove the AI read the amount correctly.**

## Prerequisites

- n8n running locally or on a server.
- Docker / Docker Desktop if using the same deployment approach.
- Ollama installed and reachable from n8n.
- The Ollama model downloaded locally:

  ```bash
  ollama pull qwen2.5vl:3b
  ```

- A Google account and working OAuth credentials for Gmail, Google Drive, and Google Sheets.
- A Google Drive folder to receive invoice attachments.
- A Google Sheets spreadsheet with a worksheet named `Despesas` and headers matching the fields sent by the workflow.

### Suggested spreadsheet headers

The current workflow maps these fields:

```text
fileName,driveFileId,driveFileUrl,supplier,supplierNif,invoiceNumber,invoiceDate,total,atcud,hasValidDate,hasValidTotal,status
```

If you have an existing spreadsheet with additional historical columns, check the automatic field mapping before running the workflow. Make sure the `total` and `invoiceDate` columns are usable as numbers and dates in your locale.

## Installation and configuration

1. **Import the workflow** into n8n using **Import from File** and select `workflows/expense-tracker.json`.
2. **Connect credentials** in the relevant nodes: Gmail Trigger, Google Drive Upload, Google Sheets Append Row, and Ollama Analyze Image. The exported public workflow intentionally contains no credential assignments.
3. **Configure the authorized sender** in the `If` node by replacing `your-email@example.com` with your own address. The subject check accepts variations of `n8n` irrespective of case.
4. **Choose a Google Drive folder** in `Upload file`, replacing the placeholder `REPLACE_WITH_DRIVE_FOLDER_ID`.
5. **Choose your spreadsheet** in `Append row in sheet`, replacing `REPLACE_WITH_SPREADSHEET_ID`; confirm the worksheet name is `Despesas`.
6. **Check Ollama connectivity** from inside the n8n environment. For Ollama running on the Windows host and n8n in Docker Desktop, `http://host.docker.internal:11434` may be suitable. Your configuration may differ.
7. **Verify the model** `qwen2.5vl:3b` is available in Ollama and selected in the Analyze Image node.
8. **Test with one supported image attachment** before testing multiple documents. Review the Drive upload, Merge output, and resulting spreadsheet row.
9. **Test multiple attachments**, then activate the workflow only after confirming item matching and error behavior in your environment.

> **Important:** Do not run both the original private workflow and the imported workflow against the same unread Gmail messages. Repeated test runs can generate duplicate uploads and spreadsheet rows.

## Example usage

1. Send an email from the authorized address with subject `n8n` (or `N8N`).
2. Attach an invoice image.
3. n8n checks the email, extracts the attachment, and sends it for local AI analysis.
4. The original document is uploaded to Google Drive.
5. The data and document reference are appended to the `Despesas` worksheet.
6. Spreadsheet formulas and dashboard charts can update based on the new row.

## Reporting dashboard

A companion Google Sheets dashboard can be built from the `Despesas` worksheet to show:

- Spending this week, month, and year.
- Monthly trends and comparisons against the previous year.
- Spending by store / supplier.
- Weekly spending trends.
- Filters for year and month.

Dashboard formulas, formatting, and charts are maintained **in Google Sheets**, not embedded in the workflow export. Screenshots may be added to the `screenshots/` directory after removing personal information.

## Known limitations

- **AI accuracy:** OCR/vision-language extraction may confuse totals or misread supplier details. Basic validation cannot detect every incorrect amount.
- **Error handling:** An Ollama HTTP 500, model interruption, or missing response may still interrupt or prevent complete processing. Sequential batches reduce concurrent load but do not guarantee recovery.
- **PDF support:** The attachment filter accepts PDFs, but the current Ollama node uses the **Analyze Image** operation. PDF analysis may require conversion to images or a separate document-processing path; PDF extraction is not verified end-to-end.
- **Duplicates:** There is currently no duplicate-invoice detection. Re-running an email can create duplicate Drive files and Sheet rows.
- **Message state:** Marking an email as read occurs in the per-attachment processing path. Do not assume it proves every attachment from the message was processed successfully.
- **Availability:** With n8n and Ollama hosted locally, the computer and required services must remain running for automatic processing.
- **Security:** Gmail sender and subject checks are workflow filters, not a substitute for stronger authentication or verification of message authenticity.

## Roadmap

- [ ] Robust per-attachment error handling and retry strategy.
- [ ] Duplicate detection using invoice identifiers and/or attachment hashes.
- [ ] Reliable PDF-to-image or PDF document processing.
- [ ] Manual expense entries from natural-language email messages (for example, *“I spent €0.50 on coffee”*).
- [ ] Expense categories and supplier normalization.
- [ ] More advanced dashboard filters and reporting.
- [ ] Optional local-file backup alongside Google Drive.
- [ ] Reproducible deployment instructions and sample Google Sheets template.

## Repository layout

```text
ai-expense-tracker-n8n/
├── README.md
├── workflows/
│   └── expense-tracker.json
└── screenshots/           # Optional, sanitized screenshots
```

## Privacy and security

The public workflow should contain **no OAuth tokens, API keys, personal email addresses, personal Drive folder IDs, or private spreadsheet IDs**. Set your own credentials and resource IDs *after* importing into n8n. Before publishing modifications, review node parameters, JavaScript expressions, pinned data, exported metadata, and screenshots for sensitive information.

Keep real invoice images and personal expense records out of the repository. Use synthetic or redacted samples for demonstrations.

## License

A license has not been selected yet. Add a `LICENSE` file if you intend to define reuse and distribution terms.
