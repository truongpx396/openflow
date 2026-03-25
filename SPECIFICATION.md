# OpenFlow Workflow Specification v1.0.0

> A vendor-neutral, open, YAML-based Domain Specific Language (DSL) for defining complex automation workflows.

## License & Design Lineage

This specification is an **original, independent work** released under the **Apache License 2.0**.

This DSL draws inspiration from well-established, open-source, vendor-neutral standards:
- [CNCF Serverless Workflow Specification](https://serverlessworkflow.io/) (Apache 2.0)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Argo Workflows](https://argoproj.github.io/argo-workflows/) (Apache 2.0)
- [Apache Airflow DAG concepts](https://airflow.apache.org/) (Apache 2.0)
- [Tekton Pipelines](https://tekton.dev/) (Apache 2.0)

---

## Table of Contents

1. [Overview](#overview)
2. [Design Principles](#design-principles)
3. [Document Structure](#document-structure)
4. [Metadata](#metadata)
5. [Triggers](#triggers)
6. [Variables & Context](#variables--context)
7. [Steps](#steps)
8. [Step Types](#step-types)
9. [Flow Control](#flow-control)
10. [Error Handling](#error-handling)
11. [Step Policies](#step-policies) — Cache, Rate Limit, Logging
12. [Expressions](#expressions)
13. [Secrets & Credentials](#secrets--credentials)
14. [Extensions](#extensions)
15. [Profiles & Environments](#profiles--environments)
16. [Execution Semantics](#execution-semantics)

---

## Overview

OpenFlow is a declarative, YAML-based workflow definition language designed to describe complex
multi-step automation pipelines. It supports:

- **Sequential, parallel, and conditional** execution flows
- **Loops and iteration** over data collections
- **Branching and merging** with fan-out/fan-in patterns
- **Error handling** with retries, fallbacks, circuit breakers, and try/catch/finally
- **Event-driven triggers** (webhooks, schedules, queues, file events, manual with input forms)
- **Database operations** for SQL, NoSQL, and key-value stores
- **AI / LLM integration** with prompt templating and structured output
- **Human-in-the-loop** approvals with form fields and multi-level escalation
- **Step policies** — caching, rate limiting, and structured logging
- **Extensibility** via custom step types, plugins, and function references
- **Data transformation** using expression language
- **Secret management** via external references (never inline)
- **Profiles** for multi-environment configuration (dev, uat, prod)

---

## Design Principles

1. **Vendor-neutral**: No dependency on any specific runtime or platform
2. **Human-readable**: YAML-first, designed for readability and editability
3. **Declarative**: Describe *what* should happen, not *how*
4. **Composable**: Workflows can reference other workflows (sub-flows)
5. **Type-safe**: Schema-validated with JSON Schema
6. **Secure by default**: Secrets are always referenced, never embedded
7. **Extensible**: Custom step types via plugin mechanism

---

## Document Structure

Every workflow document has this top-level structure:

```yaml
openflow: "1.0.0"                  # Required: spec version

metadata:                           # Required: workflow identity
  name: my-workflow
  version: "1.0.0"
  description: "What this workflow does"
  labels: {}
  annotations: {}

triggers: []                        # Optional: what starts this workflow

variables: {}                       # Optional: workflow-level variables

secrets: []                         # Optional: secret references

steps: []                           # Required: the ordered list of steps

profiles: {}                        # Optional: per-environment overrides

on_error: {}                        # Optional: global error handler

extensions: []                      # Optional: custom extensions
```

---

## Metadata

```yaml
metadata:
  name: invoice-document-processing      # Required: unique identifier (kebab-case)
  version: "2.1.0"                        # Required: semver
  description: "Process and validate hospital invoices"
  labels:                                  # Optional: key-value for filtering/grouping
    team: finance-automation
    environment: production
  annotations:                             # Optional: arbitrary metadata
    owner: document-processing-team
    sla: "5m"
```

---

## Triggers

Triggers define how a workflow is initiated.

### Webhook Trigger

```yaml
triggers:
  - type: webhook
    id: incoming-invoice
    config:
      method: POST
      path: /api/process-invoice
      response_mode: deferred          # "immediate" | "deferred" | "stream"
      authentication:
        type: bearer
        secret_ref: webhook-api-key
```

### Schedule Trigger

```yaml
triggers:
  - type: schedule
    id: nightly-batch
    config:
      cron: "0 2 * * *"
      timezone: Asia/Bangkok
```

### Queue Trigger

```yaml
triggers:
  - type: queue
    id: sqs-trigger
    config:
      provider: aws-sqs
      queue_url: "https://sqs.ap-southeast-1.amazonaws.com/123/my-queue"
      credential_ref: aws-credentials
      batch_size: 10
```

### Event Trigger

```yaml
triggers:
  - type: event
    id: file-uploaded
    config:
      source: s3
      event_type: object.created
      filter:
        bucket: my-bucket
        prefix: "incoming/"
```

### Manual Trigger

A manual trigger starts a workflow via a UI button or CLI command. Use `input_schema`
to declare structured input — runtimes render a form from this schema.

```yaml
triggers:
  - type: manual
    id: start-manually
    config:
      description: "Run the data migration pipeline"
      input_schema:
        type: object
        required: [source_env, target_env]
        properties:
          source_env:
            type: string
            title: "Source Environment"
            enum: [dev, uat, prod]
          target_env:
            type: string
            title: "Target Environment"
            enum: [dev, uat, prod]
          dry_run:
            type: boolean
            title: "Dry Run?"
            default: true
          batch_size:
            type: integer
            title: "Batch Size"
            minimum: 1
            maximum: 10000
            default: 100
```

Input values are available via `trigger.input`:

```yaml
variables:
  source_env: "${{ trigger.input.source_env }}"
  target_env: "${{ trigger.input.target_env }}"
  dry_run: "${{ trigger.input.dry_run }}"
```

---

## Variables & Context

Variables can be declared at workflow level and accessed throughout steps.

```yaml
variables:
  invoice_no: "${{ trigger.body.invoice_no }}"
  processing_date: "${{ now() }}"
  max_retries: 3
```

### Built-in Context Objects

| Context | Description |
|---------|-------------|
| `trigger` | Data from the trigger event (body, headers, params) |
| `steps.<step_id>.output` | Output from a completed step |
| `variables` | Workflow-level variables |
| `secrets` | Referenced secrets (resolved at runtime) |
| `env` | Environment variables |
| `workflow` | Workflow metadata (name, version, run_id) |

---

## Steps

Steps are the building blocks of a workflow. Each step has:

```yaml
steps:
  - id: upload-file                    # Required: unique step identifier
    name: "Upload document to storage" # Optional: human-readable name
    type: http                         # Required: step type
    description: "Upload the PDF"      # Optional: human-readable description
    config: {}                         # Optional: type-specific configuration
    input: {}                          # Optional: data mapping into the step
    output: {}                         # Optional: data mapping from the step
    depends_on: [previous-step]        # Optional: explicit DAG dependencies
    condition: "${{ ... }}"            # Optional: run only if expression is true
    retry: {}                          # Optional: retry policy
    timeout: 30s                       # Optional: step timeout (s/m/h/d)
    on_error: {}                       # Optional: error handler
    circuit_breaker: {}                # Optional: circuit breaker policy
    cache: {}                          # Optional: cache step output by key
    rate_limit: {}                     # Optional: limit execution frequency
    log: {}                            # Optional: structured logging
    merge_strategy: combine            # Optional: how to merge multi-dependency outputs
    merge_input: step-id               # Optional: pick specific step (when strategy=choose)
    execution_mode: per_item           # Optional: "per_item" | "once"
    emit_on_empty: false               # Optional: emit empty output if operation returns nothing
    metadata: {}                       # Optional: annotations
```

---

## Step Types

### `http` — HTTP Request

```yaml
- id: chunk-document
  type: http
  config:
    method: POST
    url: "https://api.example.com/chunk"
    headers:
      Content-Type: application/json
    body:
      env: "UAT"
      bucket: "my-bucket"
      key: "${{ steps.upload-file.output.key }}"
      max_pages: 15
    timeout: 60s
    response_format: json              # "json" | "text" | "binary"
```

#### Binary & Multipart Uploads

For file uploads, use `content_type: multipart/form-data` with `parts`:

```yaml
- id: upload-to-sharepoint
  type: http
  config:
    method: POST
    url: "https://api.example.com/upload"
    content_type: multipart/form-data
    parts:
      - name: env
        value: "UAT"
      - name: action
        value: "upload"
      - name: folderPath
        value: "/Documents/${{ variables.folder_name }}"
      - name: file
        binary_ref: "${{ steps.download-file.output.binary }}"
        filename: "${{ steps.download-file.output.filename }}"
    response_format: json
```

#### Binary Downloads

```yaml
- id: download-file
  type: http
  config:
    method: GET
    url: "https://api.example.com/files/${{ variables.file_id }}"
    response_format: binary            # Output includes .binary and .filename
```

### `transform` — Data Transformation

```yaml
- id: parse-input
  type: transform
  config:
    language: jsonata                  # "jsonata" | "jmespath" | "jq" | "cel"
    expression: |
      {
        "invoice_no": trigger.body.invoice_no,
        "pages": steps.aggregate.output.pages
      }
```

### `script` — Execute Code

```yaml
- id: validate-invoice
  type: script
  config:
    language: javascript               # "javascript" | "python" | "typescript"
    code: |
      const invoiceNo = input.excel.invoice_no.trim();
      const ocrPages = input.ocr_pages.filter(p => p.document_type === 'medical_invoices');
      const ocrInvoiceNos = ocrPages.map(p => p.content?.invoice_info?.invoice_number?.trim());
      const matched = ocrInvoiceNos.includes(invoiceNo);
      return {
        rule: 'Invoice No. Match',
        passed: matched,
        excel_value: invoiceNo,
        ocr_values: ocrInvoiceNos
      };
```

### `storage` — Cloud Storage Operations

```yaml
- id: upload-to-s3
  type: storage
  config:
    provider: s3
    credential_ref: aws-credentials
    operation: upload
    bucket: my-bucket
    key: "invoices/${{ variables.invoice_no }}/document.pdf"
    source: "${{ trigger.files[0] }}"
```

### `database` — Database Operations

Execute queries and commands against SQL and NoSQL databases.

#### SQL Query

```yaml
- id: fetch-orders
  type: database
  config:
    provider: postgres                 # "postgres" | "mysql" | "mssql" | "sqlite"
    credential_ref: db-credentials
    operation: query
    query: |
      SELECT id, customer_name, total, status
      FROM orders
      WHERE status = $1 AND created_at > $2
    params:
      - "${{ variables.target_status }}"
      - "${{ variables.since_date }}"
```

#### SQL Insert / Update

```yaml
- id: update-order-status
  type: database
  config:
    provider: postgres
    credential_ref: db-credentials
    operation: execute
    query: |
      UPDATE orders SET status = $1, updated_at = NOW()
      WHERE id = $2
    params:
      - "completed"
      - "${{ variables.order_id }}"
```

#### Batch Insert

```yaml
- id: insert-line-items
  type: database
  config:
    provider: postgres
    credential_ref: db-credentials
    operation: insert
    table: line_items
    columns: [order_id, product_id, quantity, unit_price]
    rows: "${{ steps.prepare-items.output.rows }}"
    on_conflict: skip                  # "skip" | "update" | "fail"
```

#### NoSQL Operations

```yaml
- id: save-document
  type: database
  config:
    provider: mongodb                  # "mongodb" | "dynamodb" | "firestore" | "redis"
    credential_ref: mongo-credentials
    operation: insert_one
    database: app_db
    collection: audit_logs
    document:
      workflow_run: "${{ workflow.run_id }}"
      action: "order_processed"
      timestamp: "${{ now() }}"
      details: "${{ steps.process.output }}"
```

```yaml
- id: find-customer
  type: database
  config:
    provider: mongodb
    credential_ref: mongo-credentials
    operation: find
    database: app_db
    collection: customers
    filter:
      email: "${{ variables.customer_email }}"
    projection:
      name: 1
      tier: 1
      credit_limit: 1
    limit: 1
```

#### Key-Value Store

```yaml
- id: check-cache
  type: database
  config:
    provider: redis
    credential_ref: redis-credentials
    operation: get
    key: "rate_limit:${{ variables.client_id }}"
```

```yaml
- id: set-cache
  type: database
  config:
    provider: redis
    credential_ref: redis-credentials
    operation: set
    key: "rate_limit:${{ variables.client_id }}"
    value: "${{ steps.compute-count.output.count }}"
    ttl: 3600                          # Seconds
```

Supported operations by provider:

| Provider | Operations |
|----------|------------|
| `postgres`, `mysql`, `mssql`, `sqlite` | `query`, `execute`, `insert`, `transaction` |
| `mongodb` | `find`, `find_one`, `insert_one`, `insert_many`, `update_one`, `update_many`, `delete_one`, `delete_many`, `aggregate` |
| `dynamodb` | `get_item`, `put_item`, `query`, `scan`, `delete_item` |
| `firestore` | `get`, `set`, `update`, `delete`, `query` |
| `redis` | `get`, `set`, `del`, `hget`, `hset`, `lpush`, `rpush`, `lpop`, `rpop`, `incr` |

### `document` — Document Manipulation (Spreadsheets, PDFs)

For direct manipulation of office documents (Excel, Google Sheets, PDFs):

```yaml
- id: update-header
  type: document
  config:
    provider: microsoft-excel          # "microsoft-excel" | "google-sheets" | "pdf"
    credential_ref: excel-credentials
    operation: update_range
    workbook_id: "${{ steps.clone-template.output.id }}"
    worksheet: "Sheet1"
    range: "A4:J13"
    data: "${{ steps.prepare-header.output.matrix }}"
```

```yaml
- id: add-table-row
  type: document
  config:
    provider: microsoft-excel
    credential_ref: excel-credentials
    operation: add_table_row
    workbook_id: "${{ steps.clone-template.output.id }}"
    worksheet: "Sheet1"
    table: "DetailTable"
    data: "${{ item }}"
```

Supported operations:

| Operation | Description |
|-----------|-------------|
| `update_range` | Write data to a cell range |
| `add_table_row` | Append a row to a named table |
| `read_range` | Read data from a cell range |
| `create_worksheet` | Add a new worksheet |
| `clone` | Clone a template file |
| `share` | Set sharing permissions |
| `download` | Download as binary |

### `ai` — AI / LLM Operations

Call large language models and AI services with prompt templating, structured output,
and model configuration.

#### Text Generation

```yaml
- id: classify-ticket
  type: ai
  config:
    provider: openai                   # "openai" | "anthropic" | "google" | "azure-openai" | "bedrock" | "ollama"
    credential_ref: openai-credentials
    operation: chat
    model: gpt-4o
    messages:
      - role: system
        content: |
          You are a support ticket classifier. Classify the ticket into one of:
          billing, technical, feature_request, other.
          Respond with JSON: { "category": "...", "confidence": 0.0-1.0, "summary": "..." }
      - role: user
        content: "${{ variables.ticket_body }}"
    temperature: 0.1
    max_tokens: 200
    response_format: json              # "text" | "json" | "structured"
```

#### Structured Output

Use `response_schema` to enforce a typed response from the model:

```yaml
- id: extract-invoice-data
  type: ai
  config:
    provider: openai
    credential_ref: openai-credentials
    operation: chat
    model: gpt-4o
    messages:
      - role: system
        content: "Extract structured data from the invoice text."
      - role: user
        content: "${{ steps.ocr.output.text }}"
    response_format: structured
    response_schema:
      type: object
      properties:
        invoice_number:
          type: string
        vendor_name:
          type: string
        line_items:
          type: array
          items:
            type: object
            properties:
              description: { type: string }
              quantity: { type: number }
              unit_price: { type: number }
        total:
          type: number
        currency:
          type: string
      required: [invoice_number, vendor_name, total]
```

#### Embeddings

```yaml
- id: embed-document
  type: ai
  config:
    provider: openai
    credential_ref: openai-credentials
    operation: embedding
    model: text-embedding-3-small
    input: "${{ steps.extract-text.output.text }}"
```

#### Image Analysis

```yaml
- id: analyze-receipt
  type: ai
  config:
    provider: openai
    credential_ref: openai-credentials
    operation: chat
    model: gpt-4o
    messages:
      - role: user
        content:
          - type: text
            text: "Extract the total amount and merchant name from this receipt."
          - type: image_url
            url: "${{ steps.download-receipt.output.url }}"
    response_format: json
```

#### Prompt Templates

Use `template_ref` to reference reusable prompt templates defined in extensions:

```yaml
- id: summarize-report
  type: ai
  config:
    provider: anthropic
    credential_ref: anthropic-credentials
    operation: chat
    model: claude-sonnet-4-20250514
    template_ref: summarize-document   # Defined in extensions
    template_vars:
      document: "${{ steps.fetch-report.output.text }}"
      max_length: 500
      language: "${{ variables.output_language }}"
    max_tokens: 1000
```

Supported operations:

| Operation | Description |
|-----------|-------------|
| `chat` | Send messages to a chat completion model |
| `embedding` | Generate vector embeddings from text |
| `completion` | Legacy text completion (non-chat models) |
| `moderate` | Run content moderation / safety check |

The output includes model metadata:

```yaml
# steps.<id>.output structure:
#   result: <model response — text, JSON, or parsed structured output>
#   model: "gpt-4o"
#   usage:
#     prompt_tokens: 150
#     completion_tokens: 45
#     total_tokens: 195
#   finish_reason: "stop"
```

### `queue` — Message Queue Operations

```yaml
- id: send-result
  type: queue
  config:
    provider: aws-sqs
    credential_ref: aws-credentials
    operation: send
    queue_url: "https://sqs.../my-queue"
    message: "${{ steps.prepare-result.output }}"
```

### `notification` — Send Notifications

```yaml
- id: notify-error
  type: notification
  config:
    channel: webhook                   # "email" | "slack" | "webhook" | "sms"
    url: "https://hooks.example.com/notify"
    payload:
      status_code: "ERR_01"
      invoice_no: "${{ variables.invoice_no }}"
      reason: "${{ steps.validation.output.reason }}"
```

### `subflow` — Call Another Workflow

```yaml
- id: run-ocr-pipeline
  type: subflow
  config:
    workflow_ref: ocr-processing-pipeline    # name or path to another workflow
    version: "^1.0.0"                         # semver range
    input:
      document_path: "${{ steps.upload.output.path }}"
    await: true                               # wait for completion
```

### `respond` — Send Response to Trigger

```yaml
- id: respond-success
  type: respond
  config:
    status_code: 200
    body: "${{ steps.prepare-result.output }}"
    headers:
      Content-Type: application/json
```

### `noop` — No Operation (Merge/Sync Point)

```yaml
- id: sync-point
  type: noop
  name: "Wait for all parallel branches"
  wait_for:
    - branch-a
    - branch-b
```

### `approval` — Human-in-the-Loop

Pauses workflow execution and waits for a human decision before continuing.
The runtime sends a notification to the assignees and exposes a callback
(webhook URL, form, or messaging action) for them to respond.

```yaml
- id: manager-approval
  type: approval
  config:
    prompt: "Approve order #${{ variables.order_id }} (${{ variables.total }})?" 
    assignees:
      - "manager@example.com"
    actions:                           # Available response actions
      - id: approve
        label: "Approve"
      - id: reject
        label: "Reject"
      - id: escalate
        label: "Escalate to VP"
    timeout: 24h                       # Max wait time
    on_timeout: reject                 # Action to take if no response
    notification:                      # How to notify assignees
      channel: email                   # "email" | "slack" | "webhook" | "sms"
      template: approval-request       # Optional notification template
```

The selected action is available downstream via `steps.<id>.output.action`
and any additional data via `steps.<id>.output.response`.

> **Note:** `approval.on_timeout` references one of the declared `actions[].id` values
> (e.g., `reject`, `escalate`). This differs from `wait.on_timeout`, which uses
> fixed keywords (`cancel`, `continue`, `fail`).

#### Approval with Form Fields

Use `fields` to collect structured data from the approver along with their decision:

```yaml
- id: shipping-details
  type: approval
  config:
    prompt: "Provide shipping details for order #${{ variables.order_id }}"
    assignees:
      - "warehouse@example.com"
    actions:
      - id: confirm
        label: "Confirm & Ship"
      - id: hold
        label: "Hold Order"
    fields:
      - id: carrier
        label: "Carrier"
        type: select
        options: ["FedEx", "UPS", "DHL", "USPS"]
        required: true
      - id: tracking_no
        label: "Tracking Number"
        type: text
        required: true
      - id: ship_date
        label: "Ship Date"
        type: date
      - id: notes
        label: "Notes"
        type: textarea
    timeout: 24h
    on_timeout: hold
```

Field values are available via `steps.<id>.output.fields`:

```yaml
- id: update-shipment
  type: http
  depends_on: [shipping-details]
  condition: "${{ steps.shipping-details.output.action == 'confirm' }}"
  config:
    method: POST
    url: "https://api.example.com/shipments"
    body:
      order_id: "${{ variables.order_id }}"
      carrier: "${{ steps.shipping-details.output.fields.carrier }}"
      tracking_number: "${{ steps.shipping-details.output.fields.tracking_no }}"
```

Supported field types: `text`, `textarea`, `number`, `select`, `multi_select`, `date`, `datetime`, `checkbox`, `email`, `url`, `file`.

#### Branching on Approval Result

```yaml
- id: handle-approval-result
  type: branch
  depends_on: [manager-approval]
  config:
    conditions:
      - id: approved
        when: "${{ steps.manager-approval.output.action == 'approve' }}"
        then: process-order
      - id: escalated
        when: "${{ steps.manager-approval.output.action == 'escalate' }}"
        then: vp-approval
      - id: rejected
        when: true
        then: cancel-order
```

#### Multi-Level Approval

Chain multiple approval steps for escalation workflows:

```yaml
- id: team-lead-approval
  type: approval
  config:
    prompt: "Review expense report #${{ variables.report_id }}"
    assignees: ["${{ variables.team_lead_email }}"]
    actions:
      - id: approve
        label: "Approve"
      - id: reject
        label: "Reject"
    timeout: 48h
    on_timeout: escalate

- id: director-approval
  type: approval
  depends_on: [team-lead-approval]
  condition: "${{ steps.team-lead-approval.output.action == 'approve' && variables.amount > 5000 }}"
  config:
    prompt: "Final approval for expense ${{ variables.amount }}"
    assignees: ["${{ variables.director_email }}"]
    actions:
      - id: approve
        label: "Approve"
      - id: reject
        label: "Reject"
    timeout: 72h
    on_timeout: reject
```

### `wait` — Pause Until Signal

Pauses workflow execution until a resume signal is received. Unlike `approval`,
which requires a human decision, `wait` is a generic primitive for pausing until
an external system, timer, or event resumes the workflow.

```yaml
- id: wait-for-payment
  type: wait
  config:
    resume_on: webhook                 # "webhook" | "event" | "timeout"
    timeout: 48h                       # Max wait duration
    on_timeout: cancel                 # "cancel" | "continue" | "fail"
```

#### Resume Modes

| `resume_on` | Description |
|-------------|-------------|
| `webhook` | Pauses until a callback URL is called by an external system |
| `event` | Pauses until a matching event is received from the event bus |
| `timeout` | Pauses for a fixed duration, then automatically continues |

#### Webhook Resume

The runtime generates a unique callback URL. When called, the workflow resumes
and the payload is available via `steps.<id>.output.payload`:

```yaml
- id: wait-for-payment-confirmation
  type: wait
  config:
    resume_on: webhook
    timeout: 72h
    on_timeout: fail

- id: process-payment
  type: script
  depends_on: [wait-for-payment-confirmation]
  config:
    language: javascript
    code: |
      const payment = input.payload;
      return { status: payment.status, amount: payment.amount };
  input:
    payload: "${{ steps.wait-for-payment-confirmation.output.payload }}"
```

#### Event Resume

```yaml
- id: wait-for-file-upload
  type: wait
  config:
    resume_on: event
    event:
      type: file.uploaded
      filter: "${{ event.bucket == 'incoming' }}"
    timeout: 24h
    on_timeout: cancel
```

#### Timeout Resume (Delay)

Used as a simple delay/sleep:

```yaml
- id: cool-down
  type: wait
  config:
    resume_on: timeout
    timeout: 5m                        # Wait exactly 5 minutes then continue
```

---

## Flow Control

### Sequential (Default)

Steps execute in order by default:

```yaml
steps:
  - id: step-1
    type: http
    config: { ... }
  - id: step-2
    type: transform
    config: { ... }
    depends_on: [step-1]               # Explicit dependency (optional for linear flows)
```

### Merge / Synchronization Join

When multiple independent steps run in parallel and must synchronize before continuing,
use `depends_on` with a `merge_strategy`:

```yaml
- id: update-header
  type: http
  depends_on: [prepare-data]
  config: { ... }

- id: update-footer
  type: http
  depends_on: [prepare-data]
  config: { ... }

- id: continue-after-both
  type: script
  depends_on: [update-header, update-footer]
  merge_strategy: combine              # "combine" | "first" | "last" | "choose"
  merge_input: update-footer           # when strategy="choose", pick this step's output
  config: { ... }
```

| Strategy | Behavior |
|----------|----------|
| `combine` | Merge all outputs into a single object (default) |
| `first` | Use output from the first step that completes |
| `last` | Use output from the last step that completes |
| `choose` | Use output from the step specified in `merge_input` |

### Conditional Branching

```yaml
- id: check-documents
  type: branch
  config:
    conditions:
      - id: all-docs-present
        when: "${{ steps.summarize.output.summary.medical_invoices >= 1 && steps.summarize.output.summary.national_id >= 1 }}"
        then: validate-invoice          # step id to jump to
      - id: docs-missing
        when: true                      # default/else branch
        then: handle-missing-docs
```

### Multi-Way Switch

For routing to different step pipelines based on a value (e.g., environment, status code):

```yaml
- id: route-by-env
  type: switch
  config:
    value: "${{ trigger.body.env | upper }}"
    cases:
      - match: "DEV"
        then: process-dev               # step id to jump to
      - match: "UAT"
        then: process-uat
      - match: "PROD"
        then: process-prod
    default: process-dev                 # fallback case
```

For reusing the same pipeline with different config, combine `switch` with **profiles** (see
[Profiles & Environments](#profiles--environments)).

### Parallel Execution

```yaml
- id: run-validations
  type: parallel
  config:
    branches:
      - steps:
          - id: validate-rule-1
            type: script
            config: { ... }
      - steps:
          - id: validate-rule-2
            type: script
            config: { ... }
      - steps:
          - id: validate-rule-3
            type: script
            config: { ... }
    join: all                           # "all" | "any" | integer (N of M)
    output_merge: combine               # "combine" | "first" | "last"
```

### Sequential Validation Chain (Gate Pattern)

For sequential validation where each step must pass before the next runs:

```yaml
- id: validation-chain
  type: gate_chain
  config:
    gates:
      - id: rule-1-invoice-no
        step_ref: validate-invoice-no
        pass_condition: "${{ output.passed == true }}"
      - id: rule-2-patient-name
        step_ref: validate-patient-name
        pass_condition: "${{ output.passed == true }}"
      - id: rule-3-hospital-stamp
        step_ref: validate-hospital-stamp
        pass_condition: "${{ output.passed == true }}"
    on_all_passed: success-handler
    on_gate_failed: error-handler
    collect_failures: true              # accumulate failed gate info
```

### Loop / Iteration

```yaml
- id: process-chunks
  type: for_each
  config:
    items: "${{ steps.split-chunks.output.chunks }}"
    iterator_var: chunk
    batch_size: 3                       # Process N items at a time
    concurrency: 3                      # Parallel within batch
    steps:
      - id: ocr-process
        type: http
        config:
          method: POST
          url: "https://api.example.com/ocr"
          body:
            file_path: "${{ chunk.path }}"
            type: "${{ chunk.type }}"
    output:
      aggregate: true                   # Collect all outputs into array
      field: results
```

### While Loop

```yaml
- id: poll-status
  type: while
  config:
    condition: "${{ output.status != 'completed' }}"
    max_iterations: 100
    delay_between: 5s
    steps:
      - id: check-status
        type: http
        config:
          method: GET
          url: "https://api.example.com/status/${{ variables.job_id }}"
```

### Try / Catch / Finally

Wrap a group of steps with shared error handling and guaranteed cleanup logic.
Unlike step-level `on_error`, `try` provides structured exception handling across
multiple steps with a `catch` block for recovery and a `finally` block for cleanup
that always runs.

```yaml
- id: process-with-cleanup
  type: try
  config:
    steps:
      - id: download-file
        type: http
        config:
          method: GET
          url: "https://api.example.com/files/${{ variables.file_id }}"
          response_format: binary

      - id: process-file
        type: script
        depends_on: [download-file]
        config:
          language: python
          code: |
            import json
            data = parse_document(input.binary)
            return { "records": data["records"], "count": len(data["records"]) }

      - id: upload-result
        type: storage
        depends_on: [process-file]
        config:
          provider: s3
          credential_ref: aws-credentials
          operation: upload
          bucket: results
          key: "processed/${{ variables.file_id }}.json"
          source: "${{ steps.process-file.output }}"

    catch:
      - id: log-failure
        type: notification
        config:
          channel: slack
          url: "${{ secrets.slack_webhook }}"
          payload:
            text: "Pipeline failed: ${{ error.message }}"
            file_id: "${{ variables.file_id }}"
            step: "${{ error.step_id }}"

      - id: quarantine-file
        type: storage
        config:
          provider: s3
          credential_ref: aws-credentials
          operation: move
          source_key: "incoming/${{ variables.file_id }}"
          destination_key: "quarantine/${{ variables.file_id }}"

    finally:
      - id: release-lock
        type: http
        config:
          method: DELETE
          url: "https://api.example.com/locks/${{ variables.file_id }}"

      - id: emit-metric
        type: script
        config:
          language: javascript
          code: |
            return {
              metric: "file_processed",
              file_id: input.file_id,
              success: !input.has_error,
              duration_ms: input.elapsed
            };
        input:
          file_id: "${{ variables.file_id }}"
          has_error: "${{ error != null }}"
          elapsed: "${{ workflow.elapsed_ms }}"
```

| Block | Runs When | Purpose |
|-------|-----------|--------|
| `steps` | Always (main execution path) | The primary steps to execute |
| `catch` | Only when a step in `steps` fails | Error recovery, alerting, compensation |
| `finally` | Always (after `steps` or `catch`) | Cleanup: release locks, close connections, emit metrics |

The `catch` block has access to the `error` context object:

| Property | Description |
|----------|-------------|
| `error.message` | Error message from the failed step |
| `error.step_id` | ID of the step that failed |
| `error.type` | Error type (e.g., `timeout`, `http_error`, `script_error`) |
| `error.status_code` | HTTP status code (for `http` step errors) |
| `error.stack` | Stack trace (if available) |

---

## Error Handling

### Step-Level Error Handling

```yaml
- id: call-api
  type: http
  config:
    method: POST
    url: "https://api.example.com/process"
  retry:
    max_attempts: 3
    delay: 2s
    backoff: exponential               # "fixed" | "exponential" | "linear"
    max_delay: 30s
    retry_on:
      - status_code: [500, 502, 503, 504]
      - error_type: timeout
  on_error:
    strategy: continue                 # "fail" | "continue" | "fallback"
    fallback_step: use-cached-result
    output:
      error: "${{ error.message }}"
      status: "degraded"
```

### Global Error Handler

```yaml
on_error:
  strategy: fail_fast
  handler:
    steps:
      - id: log-error
        type: notification
        config:
          channel: webhook
          url: "${{ secrets.error_webhook_url }}"
          payload:
            workflow: "${{ workflow.name }}"
            run_id: "${{ workflow.run_id }}"
            error: "${{ error }}"
      - id: respond-error
        type: respond
        config:
          status_code: 500
          body:
            success: false
            error: "${{ error.message }}"
```

### Circuit Breaker

```yaml
- id: external-api
  type: http
  config: { ... }
  circuit_breaker:
    failure_threshold: 5
    reset_timeout: 60s
    half_open_requests: 2
```

---

## Step Policies

Step policies are optional, cross-cutting properties that can be applied to any step
regardless of its type. They control caching, rate limiting, and observability.

### Cache

Cache step output to avoid redundant executions. The runtime stores the output
keyed by a cache key expression and serves it for subsequent runs until the TTL expires.

```yaml
- id: fetch-exchange-rates
  type: http
  config:
    method: GET
    url: "https://api.example.com/rates"
  cache:
    key: "exchange-rates:${{ variables.currency }}"
    ttl: 1h
```

With invalidation:

```yaml
- id: get-product-catalog
  type: database
  config:
    provider: postgres
    credential_ref: db-credentials
    operation: query
    query: "SELECT * FROM products WHERE active = true"
  cache:
    key: "product-catalog"
    ttl: 24h
    invalidate_on: "${{ event.type == 'product.updated' }}"
```

When a cache hit occurs, the step is skipped entirely and the cached output
is used. The `steps.<id>.output._cache_hit` boolean indicates whether the
result came from cache.

### Rate Limit

Limit how frequently a step can execute. This is proactive throttling (unlike
circuit breaker, which is reactive). Useful for honoring external API rate limits.

```yaml
- id: call-openai
  type: ai
  config:
    provider: openai
    credential_ref: openai-credentials
    operation: chat
    model: gpt-4o
    messages:
      - role: user
        content: "${{ variables.prompt }}"
  rate_limit:
    max: 60
    per: 1m
    on_limit: wait                     # "wait" | "fail" | "skip"
```

Global scope shares the limit across all concurrent workflow runs:

```yaml
- id: send-email
  type: notification
  config:
    channel: email
    to: "${{ variables.customer_email }}"
    subject: "Your order is confirmed"
    template: order-confirmation
  rate_limit:
    max: 100
    per: 1h
    scope: global                      # "workflow" (per-run, default) | "global" (across all runs)
    on_limit: wait
```

| Property | Description |
|----------|-------------|
| `max` | Maximum number of executions allowed in the window |
| `per` | Time window (e.g., `1m`, `1h`, `1s`) |
| `on_limit` | `wait` (queue until slot available), `fail` (throw error), `skip` (skip step) |
| `scope` | `workflow` (limit per run) or `global` (limit across all concurrent runs) |

### Log

Declare structured logging for observability. Log entries are emitted by the runtime
with step context, custom fields, and configurable trigger points.

```yaml
- id: process-order
  type: script
  config:
    language: javascript
    code: |
      // process the order...
      return { order_id: input.id, status: "processed" };
  log:
    level: info
    message: "Processing order ${{ variables.order_id }} for ${{ variables.customer_name }}"
    context:
      order_id: "${{ variables.order_id }}"
      customer_id: "${{ variables.customer_id }}"
      amount: "${{ variables.total }}"
    on_start: true
    on_complete: true
    on_error: true
```

Log entries include automatic fields injected by the runtime:

| Auto-injected Field | Description |
|---------------------|-------------|
| `workflow.name` | Workflow name |
| `workflow.run_id` | Current execution run ID |
| `step.id` | Step identifier |
| `step.type` | Step type |
| `timestamp` | ISO 8601 timestamp |
| `duration_ms` | Step execution duration (on complete/error) |
| `trace_id` | Distributed trace ID (if OpenTelemetry is enabled) |

Example log output (JSON):

```json
{
  "level": "info",
  "message": "Processing order ORD-12345 for Acme Corp",
  "workflow": { "name": "order-fulfillment", "run_id": "run-abc-123" },
  "step": { "id": "process-order", "type": "script" },
  "context": { "order_id": "ORD-12345", "customer_id": "C-100", "amount": 1250.00 },
  "timestamp": "2026-03-25T10:30:00Z",
  "duration_ms": 42,
  "trace_id": "abc123def456"
}
```

---

## Expressions

Expressions use the `${{ }}` syntax and support:

### Data Access
```yaml
"${{ trigger.body.invoice_no }}"
"${{ steps.step-1.output.result }}"
"${{ variables.max_retries }}"
"${{ secrets.api_key }}"
```

### Built-in Functions
```yaml
"${{ now() }}"                         # Current timestamp
"${{ uuid() }}"                        # Generate UUID
"${{ len(steps.split.output.items) }}" # Array length
"${{ format('{:.2f}', amount) }}"      # String formatting
"${{ abs(a - b) <= 0.01 }}"            # Math
"${{ lower(trim(name)) }}"            # String manipulation
"${{ json_path(data, '$.items[*].id') }}" # JSON path query
"${{ hash('sha256', value) }}"         # Hashing
```

### Comparison & Logic
```yaml
"${{ steps.validate.output.passed == true }}"
"${{ len(items) > 0 && status == 'active' }}"
"${{ value in ['a', 'b', 'c'] }}"
```

---

## Secrets & Credentials

Secrets are **never** stored in the workflow definition. They are referenced by name
and resolved at runtime by the execution engine.

```yaml
secrets:
  - id: aws-credentials
    provider: aws-secrets-manager       # "env" | "vault" | "aws-secrets-manager" | "gcp-secret-manager" | "azure-key-vault"
    path: "/prod/aws/s3-access"
  - id: api-key
    provider: env
    key: API_KEY
```

---

## Extensions

Custom step types and behaviors can be registered via extensions:

```yaml
extensions:
  - name: google-document-ai
    version: "1.2.0"
    source: "https://registry.example.com/extensions/google-doc-ai"
    config:
      project_id: "my-gcp-project-id"
      location: "us"
```

Then used in steps:

```yaml
- id: doc-ai-split
  type: ext/google-document-ai.split
  config:
    processor_id: "my-processor-id"
    input_path: "${{ steps.upload.output.path }}"
```

---

## Profiles & Environments

Profiles allow the same workflow to run with different configuration per environment,
avoiding pipeline duplication:

```yaml
profiles:
  dev:
    variables:
      folder_destination_id: "folder-dev-001"
      file_template_id: "template-dev-001"
      output_folder: "/Reports"
    secrets:
      excel_oauth: excel-dev-credentials
      drive_oauth: drive-dev-credentials
  uat:
    variables:
      folder_destination_id: "folder-uat-001"
      file_template_id: "template-uat-001"
      output_folder: "/Reports"
    secrets:
      excel_oauth: excel-uat-credentials
      drive_oauth: drive-uat-credentials
  prod:
    variables:
      folder_destination_id: "folder-prod-001"
      file_template_id: "template-prod-001"
      output_folder: "/Reports"
    secrets:
      excel_oauth: excel-prod-credentials
      drive_oauth: drive-prod-credentials
```

The active profile can be selected:
- At trigger time: `${{ trigger.body.env }}` → resolves to the matching profile
- At deployment time via CLI: `openflow run --profile=uat workflow.yaml`
- Via environment variable: `OPENFLOW_PROFILE=prod`

Inside steps, profile values are accessed via the same `variables` and `secrets` context:

```yaml
- id: clone-template
  type: document
  config:
    provider: microsoft-excel
    credential_ref: "${{ secrets.drive_oauth }}"
    operation: clone
    source_id: "${{ variables.file_template_id }}"
    destination_id: "${{ variables.folder_destination_id }}"
```

---

## Execution Semantics

### Step Execution Mode

By default, when a step receives multiple items, it runs **once per item**.
This can be controlled:

```yaml
- id: share-file
  type: document
  execution_mode: once                 # "once" | "per_item" (default: per_item)
  config: { ... }
```

| Mode | Behavior |
|------|----------|
| `per_item` | Step runs once for each input item (default) |
| `once` | Step runs exactly once regardless of input cardinality |

### Empty Output Handling

```yaml
- id: update-cells
  type: document
  emit_on_empty: true                  # Emit an empty output if the operation returns nothing
  config: { ... }
```

### Sequential Item Processing

For APIs that cannot handle concurrent writes (e.g., Excel table inserts):

```yaml
- id: insert-rows
  type: for_each
  config:
    items: "${{ steps.prepare-data.output.line_items }}"
    iterator_var: item
    concurrency: 1                     # Strictly sequential — one at a time
    steps:
      - id: add-row
        type: document
        config:
          provider: microsoft-excel
          operation: add_table_row
          table: "DetailTable"
          data: "${{ item }}"
```

---

## Full Example

See [examples/order-fulfillment.yaml](./examples/order-fulfillment.yaml),
[examples/report-excel-export.yaml](./examples/report-excel-export.yaml),
and [examples/support-ticket-triage.yaml](./examples/support-ticket-triage.yaml)
for complete real-world workflows using all features.


