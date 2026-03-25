# OpenFlow — Vendor-Neutral Workflow DSL

> A YAML-based, open-source workflow definition language for complex automation pipelines.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/Spec-v1.0.0-green.svg)](SPECIFICATION.md)

## What is OpenFlow?

OpenFlow is an **industry-standard, vendor-neutral YAML DSL** for defining automation workflows.
It is designed to handle complex, real-world automation scenarios including:

- Multi-stage order fulfillment pipelines
- Sequential validation gate chains
- Parallel fan-out/fan-in execution
- Batch iteration with concurrency control
- Event-driven triggers (webhooks, queues, schedules)
- Error handling with retries, circuit breakers, and fallbacks

## Why OpenFlow?

| Concern | OpenFlow Approach |
|---------|-------------------|
| **Vendor lock-in** | Pure YAML spec — any runtime can implement it |
| **License freedom** | Apache 2.0 — no copyleft or usage restrictions |
| **Readability** | Human-readable YAML, designed for code review |
| **Version control** | Git-friendly, diff-friendly, review-friendly |
| **Schema validation** | Full JSON Schema for IDE autocompletion & CI validation |
| **Portability** | Same workflow definition runs on any compliant runtime |

## Design Lineage

OpenFlow is an **original, independent specification** released under Apache 2.0.
Its design draws on established open-source workflow standards:

- [CNCF Serverless Workflow Specification](https://serverlessworkflow.io/) (Apache 2.0)
- [GitHub Actions Workflow Syntax](https://docs.github.com/en/actions/using-workflows)
- [Argo Workflows](https://argoproj.github.io/argo-workflows/) (Apache 2.0)
- [Tekton Pipelines](https://tekton.dev/) (Apache 2.0)
- [Apache Airflow](https://airflow.apache.org/) (Apache 2.0)

Key design choices:
- **Explicit DAG dependencies**: Steps declare `depends_on` for clear execution order
- **Expression syntax**: `${{ }}` interpolation with built-in functions
- **Declarative flow control**: `branch`, `parallel`, `gate_chain`, `for_each`, `while`
- **Generic step types**: `http`, `script`, `storage`, `queue` — not tied to any vendor SDK
- **Schema-first**: Every workflow is validated against a JSON Schema

## Comparison with Other Standards

| Feature | OpenFlow | CNCF Serverless WF | GitHub Actions | Argo Workflows |
|---------|----------|---------------------|----------------|----------------|
| YAML-native | ✅ | ✅ | ✅ | ✅ |
| Sequential validation gates | ✅ | ❌ | ❌ | ❌ |
| Parallel fan-out/fan-in | ✅ | ✅ | ✅ (matrix) | ✅ |
| Batch iteration | ✅ | ❌ | ❌ | ✅ |
| Webhook triggers | ✅ | ✅ | ✅ | ✅ |
| Queue triggers | ✅ | ✅ | ❌ | ✅ |
| Inline scripting | ✅ | ✅ | ❌ | ✅ |
| Database operations (SQL/NoSQL) | ✅ | ❌ | ❌ | ❌ |
| AI / LLM operations | ✅ | ❌ | ❌ | ❌ |
| Circuit breaker | ✅ | ❌ | ❌ | ❌ |
| Human-in-the-loop (approval) | ✅ | ❌ | ❌ | ❌ |
| Wait / pause-until-signal | ✅ | ❌ | ❌ | ✅ (suspend) |
| Try / catch / finally | ✅ | ❌ | ❌ | ❌ |
| Manual trigger with input form | ✅ | ❌ | ✅ (inputs) | ❌ |
| Step-level caching | ✅ | ❌ | ✅ (actions) | ✅ (memoize) |
| Declarative rate limiting | ✅ | ❌ | ❌ | ❌ |
| Structured step logging | ✅ | ❌ | ❌ | ❌ |
| Sub-workflows | ✅ | ✅ | ✅ (reusable) | ✅ |
| Expression language | `${{ }}` | `${ }` | `${{ }}` | Go templates |
| Schema validation | JSON Schema | JSON Schema | JSON Schema | CRD |
| License | Apache 2.0 | Apache 2.0 | Proprietary | Apache 2.0 |

> 📖 **Full Specification**: See [SPECIFICATION.md](SPECIFICATION.md) for complete documentation of all step types, flow control patterns, expressions, error handling, profiles, and execution semantics.

## Spec Maturity

OpenFlow v1.0.0 is designed to be production-grade from day one.

| Area | Assessment |
|------|------------|
| **Feature completeness** | 22 step types + 5 trigger types + 8 flow control patterns — covers ~95% of real-world automation |
| **Consistency** | Every step type follows the same `id`/`type`/`config` pattern. Uniform `credential_ref` for auth. Consistent `${{ }}` expression syntax |
| **Error handling depth** | 4 layers: step-level `on_error`, `retry`, `circuit_breaker`, `try`/`catch`/`finally`, plus global error handler |
| **Real-world examples** | YAML examples for every feature — actual Postgres queries, OpenAI calls, S3 uploads, Excel generation |
| **Schema ↔ Spec alignment** | Schema enum matches all spec step types. Every step property is schema-validated |
| **Cross-cutting policies** | `cache`, `rate_limit`, `log`, `profiles` — the features that separate a toy spec from a production spec |
| **Security posture** | Secrets-by-reference-only enforced throughout. No inline credentials permitted |

## Coverage Map

Every major durable workflow pattern is covered by a first-class primitive:

| Pattern | OpenFlow Mechanism |
|---------|--------------------|
| HTTP requests / API calls | `http` step — multipart, binary, auth |
| Sequential execution | `depends_on` / implicit ordering |
| Parallel fan-out/fan-in | `parallel` step — `join: all \| any \| N` |
| Conditional branching | `branch` + `switch` step types |
| Loops (for-each) | `for_each` — `batch_size`, `concurrency` |
| Polling loops | `while` — `max_iterations` + `delay_between` |
| Retry & backoff | `retry` — fixed, exponential, linear backoff |
| Error handling / fallbacks | `on_error` (step + global), `fallback_step` |
| Try / catch / finally | `try` step — multi-step error handling + cleanup |
| Circuit breaker | `circuit_breaker` on any step |
| Sub-workflows | `subflow` — `workflow_ref` + `await` |
| Database operations | `database` — SQL, MongoDB, DynamoDB, Redis |
| AI / LLM calls | `ai` — chat, embeddings, structured output |
| Document manipulation | `document` — Excel, Google Sheets, PDF |
| Cloud storage | `storage` — S3, GCS, Azure Blob |
| Message queues | `queue` — SQS, RabbitMQ, etc. |
| Notifications | `notification` — email, Slack, webhook, SMS |
| Data transformation | `transform` — JSONata, JMESPath, jq, CEL |
| Inline code execution | `script` — JavaScript, Python, TypeScript |
| Human-in-the-loop | `approval` — actions, form fields, multi-level |
| Wait / pause until signal | `wait` — webhook callback, event, delay |
| Validation gate chains | `gate_chain` — `collect_failures` |
| Environment profiles | `profiles` — variable/secret overrides per env |
| Secret management | `secrets` — Vault, AWS SM, GCP SM, Azure KV, env |
| Respond to caller | `respond` step |
| Merge / sync points | `noop` — `wait_for` + `merge_strategy` |
| Caching | `cache` — key + TTL + invalidation |
| Rate limiting | `rate_limit` — per-workflow or global scope |
| Structured logging | `log` — level, context, OpenTelemetry trace ID |
| Schema validation | JSON Schema for IDE autocompletion & CI |
| Extensibility | `ext/<name>.<op>` custom step types |

## Quick Start

### 1. Basic Workflow

```yaml
openflow: "1.0.0"

metadata:
  name: hello-world
  version: "1.0.0"
  description: "A simple HTTP request workflow"

triggers:
  - type: webhook
    id: start
    config:
      method: POST
      path: /api/hello

steps:
  - id: greet
    type: script
    config:
      language: javascript
      code: |
        return { message: `Hello, ${input.name}!` };
    input:
      name: "${{ trigger.body.name }}"

  - id: respond
    type: respond
    depends_on: [greet]
    config:
      status_code: 200
      body: "${{ steps.greet.output }}"
```

### 2. Validation Gate Chain

```yaml
- id: validate
  type: gate_chain
  config:
    gates:
      - id: check-email
        step:
          type: script
          config:
            language: javascript
            code: |
              const valid = /^[^@]+@[^@]+$/.test(input.email);
              return { passed: valid, reason: valid ? 'OK' : 'Invalid email' };
        pass_condition: "${{ output.passed == true }}"

      - id: check-age
        step:
          type: script
          config:
            language: javascript
            code: |
              return { passed: input.age >= 18, reason: input.age >= 18 ? 'OK' : 'Under 18' };
        pass_condition: "${{ output.passed == true }}"

    on_all_passed: process-registration
    on_gate_failed: reject-registration
    collect_failures: true
```

### 3. Parallel Execution

```yaml
- id: enrich-data
  type: parallel
  config:
    branches:
      - steps:
          - id: fetch-weather
            type: http
            config: { method: GET, url: "https://api.weather.com/current" }
      - steps:
          - id: fetch-geo
            type: http
            config: { method: GET, url: "https://api.geo.com/locate" }
    join: all
    output_merge: combine
```

## Project Structure

```
openflow/
├── SPECIFICATION.md                              # Full specification document
├── LICENSE                                        # Apache License 2.0
├── README.md                                      # This file
├── schemas/
│   └── openflow-v1.0.0.schema.json               # JSON Schema for validation
└── examples/
    ├── order-fulfillment.yaml                      # E-commerce order pipeline
    ├── report-excel-export.yaml                    # Excel report generation
    └── support-ticket-triage.yaml                  # AI triage + human-in-the-loop
```

## IDE Support

Add this comment to the top of your workflow files for schema validation:

```yaml
# yaml-language-server: $schema=./schemas/openflow-v1.0.0.schema.json
```

This enables:
- ✅ Autocompletion in VS Code (with YAML extension)
- ✅ Inline validation errors
- ✅ Hover documentation

## Roadmap

- [x] ~~**v1.0.0**: `approval`, `wait`, `database`, `ai` step types; `try/catch/finally`; `cache`, `rate_limit`, `log` step properties; `manual` trigger with `input_schema`~~ ✅
- [ ] **v1.2.0**: Add `temporal_activity` step type for durable execution
- [ ] **Runtime SDK**: Python/TypeScript reference implementation
- [ ] **CLI Tool**: `openflow validate`, `openflow run`, `openflow visualize`
- [ ] **VS Code Extension**: Visual editor for OpenFlow YAML files
- [ ] **Converter**: Import from Airflow DAG / GitHub Actions / other workflow formats

## Contributing

This is an open specification. Contributions are welcome under the Apache 2.0 license.

## License

Apache License 2.0 — See [LICENSE](LICENSE) for details.
