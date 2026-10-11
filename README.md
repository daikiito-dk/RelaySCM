# RelaySCM 🚚🔗

> **The Multi-Trading Data Relay for Microsoft 365 Copilot & AI Agents**  
> *Bridging disparate B2B shipping documents (PDF/Excel/Email) into structured, queryable MCP context for enterprise supply chain operations.*

---

## 1. Executive Summary & Problem Context

In modern manufacturing and import/export enterprises, **Supply Chain Management (SCM) and Logistics departments face daily operational bottlenecks**:

* **Information Fragmentation:** Daily intake of dozens of shipping documents—Bill of Lading (B/L), Arrival Notices, Customs Clearances, and Delay Notifications—arriving via unstructured emails, PDFs, and Excel attachments from multiple trading companies (*sogo shosha*) and freight forwarders.
* **Manual Data Entry ("Human API"):** Logistics managers spend hours manually reviewing PDFs/emails and re-keying data into internal ERPs (e.g., SAP) or spreadsheets.
* **Communication Friction:** Assessing delay risks or tracking container locations requires tedious back-and-forth phone calls and emails with trading partner sales reps.
* **Operational Risk:** Information delays directly lead to factory line stoppages and excess demurrage/detention charges.

**RelaySCM** solves this by acting as a **neutral, multi-trading data integration relay**. It automatically ingests, parses, and normalizes unstructured shipping communications, feeding structured data into **Microsoft 365 Copilot** and AI agents via the **Model Context Protocol (MCP)**—without requiring system modifications from trading partners.

---

## 2. Key Value Proposition & Architecture

```
 [Trading Partners / Forwarders]
 (Emails, PDF B/L, Excel Shipping Notices)
                 │
                 ▼ (Inbound Hook / Webhook)
 ┌─────────────────────────────────────────────────────────────┐
 │  1. RelaySCM Parsing Engine (Python / Unstructured)         │
 │     - Extract Vessel, Container No, POL/POD, ETA, Items     │
 │     - Normalize disparate schemas (e.g., ETA vs Arrival)    │
 └─────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────┐
 │  2. Serverless Storage & Data Lake (AWS S3 + DuckDB/Parquet) │
 │     - Zero always-on DB instance cost                      │
 │     - Grounding metadata tagged with Source PDF/Email ID    │
 └─────────────────────────────┬───────────────────────────────┘
                               │
                               ▼ (Model Context Protocol / SSE)
 ┌─────────────────────────────────────────────────────────────┐
 │  3. RelaySCM MCP Server (TypeScript / Node.js)             │
 │     - Microsoft 365 Copilot / Copilot Studio / Claude      │
 │     - Instant 10-second queries with 100% citation links    │
 └─────────────────────────────────────────────────────────────┘
```

### Core Architectural Principles
1. **Zero Partner Friction:** Operates purely on the documents and emails already received in the customer's inbox.
2. **Strict Grounding & Auditability:** Every AI answer provided to Copilot includes a 100% verifiable citation back to the original source PDF page or email body.
3. **Low-Cost Serverless Storage:** Uses AWS S3, Parquet, and in-memory DuckDB for fast, cost-efficient querying without expensive always-on vector DB instances.

---

## 3. Data Schema Specifications

RelaySCM normalizes disparate shipping communications into a single unified JSON Schema.

### 3.1 Normalized Shipping Document Schema (`shipping_item.schema.json`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "RelaySCM_NormalizedShipment",
  "type": "object",
  "properties": {
    "shipment_id": { "type": "string", "description": "Normalized internal shipment ID" },
    "bl_number": { "type": "string", "description": "Bill of Lading Number" },
    "trading_partner": { "type": "string", "description": "Trading company / Forwarder name" },
    "vessel_name": { "type": "string" },
    "voyage_number": { "type": "string" },
    "containers": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "container_number": { "type": "string" },
          "seal_number": { "type": "string" },
          "type": { "type": "string", "example": "40HC" }
        },
        "required": ["container_number"]
      }
    },
    "port_of_loading": { "type": "string", "example": "USLAX" },
    "port_of_discharge": { "type": "string", "example": "JPTYO" },
    "eta": { "type": "string", "format": "date-time", "description": "Estimated Time of Arrival" },
    "etd": { "type": "string", "format": "date-time", "description": "Estimated Time of Departure" },
    "status": { 
      "type": "string", 
      "enum": ["ON_SCHEDULE", "DELAYED", "CUSTOMS_HOLD", "ARRIVED", "DISCHARGED"] 
    },
    "delay_reason": { "type": "string", "nullable": true },
    "cargo_items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "part_number": { "type": "string" },
          "description": { "type": "string" },
          "quantity": { "type": "integer" },
          "unit": { "type": "string", "example": "PCS" }
        }
      }
    },
    "source_metadata": {
      "type": "object",
      "properties": {
        "source_type": { "type": "string", "enum": ["PDF_BL", "EMAIL_BODY", "EXCEL_ANNOUNCEMENT"] },
        "file_name": { "type": "string" },
        "received_at": { "type": "string", "format": "date-time" },
        "source_url_s3": { "type": "string" }
      },
      "required": ["source_type", "file_name", "received_at"]
    }
  },
  "required": ["bl_number", "trading_partner", "eta", "status", "source_metadata"]
}
```

---

## 4. MCP (Model Context Protocol) Tool Specifications

RelaySCM implements standard MCP tools exposed to Microsoft 365 Copilot, Copilot Studio agents, and Claude Desktop.

### 4.1 Tool Definitions

#### 1. `get_shipment_status`
Query current locations, ETAs, and status for specific B/L or Container numbers.
* **Input Parameters:**
  * `identifier` (string, required): B/L number or Container number.
* **Returns:** Structured shipment details including latest ETA and current status.

#### 2. `check_delay_risks`
Find shipments with delayed status or changed ETAs affecting production schedules within a date range.
* **Input Parameters:**
  * `date_range_days` (number, default: `7`): Days ahead to check.
  * `trading_partner` (string, optional): Filter by specific trading company.
* **Returns:** List of delayed shipments, delay reasons, and primary source citations.

#### 3. `get_source_citation`
Fetch the original PDF snippet or email text for auditing and verification.
* **Input Parameters:**
  * `bl_number` (string, required)
* **Returns:** Source document metadata, S3 URL, and exact extracted text snippet.

---

## 5. North Star Metric (PIN) & Business Model

RelaySCM measures product effectiveness and enterprise ROI through a single North Star Metric:

* **Primary Metric (PIN):** **Copilot Query Executions (MCP Tool Invocation Count)**
  * *Why this metric?* Every Copilot query answered by RelaySCM represents a manual data lookup, phone call, or email search eliminated for SCM personnel.

### Business & Pricing Model
* **Usage-Based Tiering:**
  * **Free / Developer Tier:** Up to 1,000 queries/month (Single facility/PoC).
  * **Enterprise Tier:** Base monthly platform fee + $0.05 per Copilot query execution.
* **ROI Dashboard:** Provides monthly automated reports quantifying hours saved (e.g., *"3,400 queries processed this month = ~115 SCM manual operational hours saved"*).

---

## 6. Quick Start Guide

### Prerequisites
* Node.js >= 18.x
* Python >= 3.10 (for ingestion engine)
* AWS Account (S3 bucket configured for data lake storage)
* Microsoft 365 Copilot or Copilot Studio environment

### Installation

```bash
# Clone repository
git clone https://github.com/your-org/RelaySCM.git
cd RelaySCM

# Install MCP Server dependencies
npm install

# Build the TypeScript MCP Server
npm run build
```

### Environment Configuration

Create a `.env` file in the root directory:

```env
AWS_REGION=us-east-1
S3_DATA_LAKE_BUCKET=relayscm-data-lake-prod
MCP_SERVER_PORT=3000
```

### Local Testing with Claude Desktop / Copilot Studio

Add RelaySCM to your local MCP client configuration (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "relayscm": {
      "command": "node",
      "args": ["/path/to/RelaySCM/dist/index.js"],
      "env": {
        "AWS_REGION": "us-east-1",
        "S3_DATA_LAKE_BUCKET": "your-s3-bucket-name"
      }
    }
  }
}
```

### Running Ingestion Engine Test

```bash
# Ingest sample B/L PDF
python scripts/ingest_document.py --file samples/sample_bl_maersk.pdf --partner "TradingCo_A"
```

---

## 7. Development Roadmap

- [x] **Phase 1 (Core):** Baseline PDF/Email Parser for standard Bills of Lading & Notification Emails.
- [x] **Phase 2 (MCP Integration):** MCP Server implementation with `get_shipment_status` and `check_delay_risks` tools.
- [ ] **Phase 3 (Copilot Studio Plugin):** Pre-packaged Microsoft 365 Copilot Studio plugin manifest and Azure Marketplace listing.
- [ ] **Phase 4 (Analytics):** Automated ROI and query-telemetry dashboard for enterprise IT managers.

---

## 8. License

Distributed under the MIT License. See `LICENSE` for more information.
