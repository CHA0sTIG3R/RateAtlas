# 🧭 RateAtlas

**RateAtlas** is a modular tax intelligence platform that ingests, normalizes, and visualizes historical IRS data to help users understand how U.S. marginal tax rates evolve over time.

The platform is composed of multiple interconnected services — each designed to handle a specific stage of the data lifecycle: ingestion, computation, and visualization.

---

## 🚦 Quick Start

Spin up the full stack locally with the steps below. Each module has deeper instructions in its own README, but this flow gets you from clone → data → API → UI.

### Prerequisites

| Tooling | Minimum Version | Used By |
| ------- | ---------------- | ------- |
| Git | latest | Repository management |
| Python | 3.11 | `rateatlas-ingest` ingestion job |
| pip + venv | matching Python | Python deps & virtual env |
| Java | 17 | `rateatlas-api` Spring Boot service |
| Maven | 3.8 | API build/run (`./mvnw` ships in repo) |
| Node.js | 20+ | `rateatlas-frontend` Vite app |
| npm | 10+ | Frontend dependencies & scripts |
| Docker + Compose | Latest engine | Local Postgres + API container |
| AWS credentials | Optional for local dry run, required for real S3 writes | Ingest + API import jobs |

### Steps

1. **Clone the repo**

```bash
   git clone https://github.com/CHA0sTIG3R/RateAtlas.git
   cd RateAtlas
```

1. **Ingestion (BracketForge) setup**

```bash
   cd rateatlas-ingest
   python3 -m venv .venv && source .venv/bin/activate
   pip install -r requirements-dev.txt
```

   Create `.env` with at least:

```ini
   S3_BUCKET=rateatlas-dev
   S3_KEY=history.csv
   DRY_RUN=1
   ENABLE_BACKEND_PUSH=0
   BACKEND_URL=http://localhost:8080
   DATABASE_URL=postgresql://user:password@host:5432/dbname?sslmode=require
```

   Then run the ingest pipeline (dry-run mode avoids touching AWS or the database):

```bash
   python -m tax_bracket_ingest.run_ingest
```

1. **API (TaxIQ) with Postgres**

```bash
   cd ../rateatlas-api
   cp .env.example .env.local
   docker compose -f docker-compose.yml -f docker-compose.local.yml up -d --build
   ./mvnw spring-boot:run
```

   Swagger UI will be live at <http://localhost:8080/swagger-ui/index.html>.

1. **Frontend (TaxLens)**

```bash
   cd ../rateatlas-frontend
   npm install
   echo "RATE_ATLAS_API_BASE_URL=http://localhost:8080/api/v1" > .env.local
   npm run dev
```

   Vite serves on <http://localhost:5173> and proxies `/api` to the backend automatically (see `vite.config.ts`).

### Verify Everything

- Run the ingest script with `DRY_RUN=0` once you connect it to real AWS S3 + backend credentials.
- Hit `GET /api/v1/tax/years` in Swagger to confirm data landed.
- Load the frontend dashboard and run a calculation to confirm API connectivity.

---

## 🧩 Architecture Overview

```mermaid
graph TD
  A["🏗️ BracketForge<br/>(rateatlas-ingest · AWS Lambda)"]
  A -->|"1. Probe IRS page date"| IRS["IRS Website"]
  IRS -->|"2. Page changed → full scrape"| A
  A -->|"3. Archive versioned CSV"| S3["AWS S3"]
  A -->|"4. POST /api/v1/tax/upload"| B
  A -->|"5. Update ingest_metadata"| DB

  B["🧮 TaxIQ<br/>(rateatlas-api · EC2)"]
  B -->|"Reads/writes"| DB["💾 PostgreSQL (RDS)"]
  B -->|"Bootstrap from S3 on startup"| S3
  B -->|"Expose /actuator/prometheus"| Alloy["Grafana Alloy (EC2)"]
  Alloy -->|"Remote write"| GC["☁️ Grafana Cloud"]

  B -->|"API responses"| C["📊 TaxLens<br/>(rateatlas-frontend)"]

  subgraph AWS Cloud
    A
    B
    DB
    S3
    Alloy
  end
  subgraph Client Side
    C
  end
```

---

## 📦 Core Modules

### 🏗️ [**rateatlas-ingest**](./rateatlas-ingest/README.md)

**Codename:** *BracketForge*
A Python-based ingestion engine deployed as an AWS Lambda that detects IRS page changes, scrapes, normalizes, and archives tax bracket data.

- **Signal-based change detection** — probes the IRS page date before scraping; exits early if nothing has changed
- Standardizes columns and schema across all four filing statuses for consistent storage
- Archives versioned datasets to AWS S3 and tracks metadata in Postgres (`ingest_metadata`)
- Pushes new records directly to TaxIQ via `POST /api/v1/tax/upload`

---

### 🧮 [**rateatlas-api**](./rateatlas-api/README.md)

**Codename:** *TaxIQ*
A Spring Boot backend that exposes endpoints for tax calculations, dataset freshness tracking, and marginal rate lookups.

- Computes marginal, average, and effective tax rates
- Exposes `GET /api/v1/datasets/latest` with IRS page date, last ingest timestamp, and computed freshness state
- Redis caching on bracket lookups, calculations, and no-tax year checks — sub-100ms response times on cache hits
- Rate limiting via Bucket4j token bucket (100 req/min per IP or API key)
- Resilience4j circuit breakers on all DB-hitting methods with cache-first fallback
- Distributed tracing via OTel Java agent exporting to Grafana Cloud Tempo
- Bootstraps Postgres from S3 on first startup if no data is present
- Prometheus metrics scraped by Grafana Alloy and forwarded to Grafana Cloud
- Deployed on AWS EC2 behind Cloudflare

---

### 📊 [**rateatlas-frontend**](./rateatlas-frontend/README.md)

**Codename:** *TaxLens*
A modern React + TypeScript + Vite interface for interactive exploration of U.S. tax history.

- Visualizes rate trends and bracket thresholds over time
- Offers side-by-side year comparisons
- Built with Tailwind CSS and Recharts for responsive design

---

### 💾 **TaxGrid (Data Layer)**

Internal PostgreSQL / S3 storage architecture that supports versioned data retrieval and analysis.

- Central schema for normalized IRS data managed via Flyway migrations
- Supports both local and AWS RDS setups
- `ingest_metadata` table tracks IRS page update dates, ingest timestamps, and run/skip counts

---

## 🔍 Data Flow

1. **BracketForge** (`rateatlas-ingest`) runs on a weekly schedule via AWS EventBridge. It first probes the IRS page for its "Last Updated" date and compares it against the last seen date stored in Postgres. If the page hasn't changed, the Lambda exits early — no scraping, no S3 writes, no backend push.
2. When the IRS page has changed, BracketForge fetches and parses the full HTML, normalizes each filing status into a canonical CSV, and archives the updated data to S3.
3. BracketForge then pushes the fresh data to **TaxIQ** via `POST /api/v1/tax/upload` and updates `ingest_metadata` with the new page date and ingest timestamp.
4. **TaxIQ** (`rateatlas-api`) persists the ingested data into Postgres via Spring Data JPA. On first startup with an empty database, it bootstraps from the same S3 history file automatically.
5. **TaxLens** (`rateatlas-frontend`) calls the API for bracket, calculation, and history endpoints to render charts, calculators, and comparisons in the browser.

S3 serves as the immutable historical archive, Postgres as the serving layer, and the frontend as the visualization surface.

---

## ⚙️ Configuration Reference

| Component                       | Key Variables                                                                       | Purpose / Notes                                      |
| ------------------------------- | ----------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Ingestion (`rateatlas-ingest`)  | `S3_BUCKET`, `S3_KEY`                                                               | Location of `history.csv` in S3                      |
|                                 | `DRY_RUN`, `ENABLE_BACKEND_PUSH`                                                    | Toggle writes to S3/backend                          |
|                                 | `BACKEND_URL`, `INGEST_API_KEY`                                                     | Push target and auth for `POST /api/v1/tax/upload`   |
|                                 | `DATABASE_URL`                                                                      | Postgres connection for `ingest_metadata` read/write |
|                                 | `AWS_REGION`, AWS credentials                                                       | Needed when not using instance roles/OIDC            |
| API (`rateatlas-api`)           | `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD` | Database connection                                  |
|                                 | `APP_INGEST_API_KEY`                                                                | Authenticates ingest pushes from BracketForge        |
|                                 | `S3_BUCKET`, `S3_KEY`                                                               | S3 source for startup bootstrap                      |
|                                 | `REDIS_URL`                                                                         | Redis Cloud connection string                        |
|                                 | `PROMETHEUS_SCRAPE_USERNAME`, `PROMETHEUS_SCRAPE_PASSWORD`                          | Basic auth protecting `/actuator/prometheus`         |
|                                 | `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`                         | OTel trace export to Grafana Cloud Tempo             |
| Frontend (`rateatlas-frontend`) | `RATE_ATLAS_API_BASE_URL` or `VITE_API_BASE_URL`                                    | Base URL for API client                              |
| Shared                          | `AWS_REGION`                                                                        | Used by both ingest and API when touching AWS        |

Keep production secrets in your deployment systems (GitHub Actions secrets, AWS SSM Parameter Store, etc.) and never commit `.env` files.

---

## ☁️ Deployment & Infrastructure

| Environment       | Description                                                         |
| ----------------- | ------------------------------------------------------------------- |
| **Development**   | Docker Compose setup for local ingestion + API testing              |
| **Production**    | AWS EC2 (API), Lambda (ingest), RDS (Postgres), S3, ECR, Cloudflare |
| **CI/CD**         | GitHub Actions with OIDC for build, test, and deploy pipelines      |
| **Observability** | Prometheus metrics → Grafana Alloy → Grafana Cloud                  |

---

## 🧠 Future Roadmap

- [ ] Add inflation-adjusted rate comparisons
- [ ] Support state-level tax data
- [ ] Integrate interactive "what-if" calculators
- [x] ~~Enable public API documentation (Swagger / Redoc)~~ — live at [api.ratesatlas.com/swagger-ui/index.html](https://api.ratesatlas.com/swagger-ui/index.html)
- [x] ~~Host frontend at ratesatlas.com~~
- [x] ~~Redis caching layer~~ — bracket lookups, calculations, no-tax year checks
- [x] ~~Rate limiting / throttling~~ — 100 req/min per IP or API key via Bucket4j
- [ ] NPM widget package `@rateatlas/tax-estimator`

---

## 🧾 Example Use Case

> "How has the top marginal tax rate changed from 1980 to 2025?"

1. **BracketForge** detects the IRS page has updated and ingests the latest data.
2. **TaxIQ** computes the marginal rate differences across years.
3. **TaxLens** visualizes the comparison graphically.

Result: users get a historical perspective and effective tax visualization in seconds.

---

## 🧰 Tech Stack Summary

| Layer         | Technology                                                                                    |
| ------------- | --------------------------------------------------------------------------------------------- |
| Ingestion     | Python 3.11, Pandas, AWS SDK (Boto3), Psycopg, Pytest                                         |
| API           | Java 17, Spring Boot 3, Maven, Spring Security, Micrometer/Prometheus, Resilience4j, Bucket4j |
| Frontend      | React 18, TypeScript, Vite, Tailwind CSS, Recharts                                            |
| Infra         | AWS EC2 / Lambda / S3 / ECR / RDS · Cloudflare                                                |
| Observability | Prometheus, Grafana Alloy, Grafana Cloud, OpenTelemetry, Grafana Tempo                        |
| Testing       | Pytest, JUnit 5, Testcontainers, GitHub Actions CI                                            |
| Deployment    | Docker multi-stage builds, AWS ECR images, GitHub Actions OIDC                                |

---

## 🧪 Testing & CI

| Layer | Local Command | Notes |
| ----- | ------------- | ----- |
| Ingestion | `pytest` (or `pytest -m "not integration"` / `pytest -m integration`) | Requires Python venv |
| API | `./mvnw test` | Spins up Testcontainers Postgres; ensure Docker is running |
| Frontend | `npm run lint` / `npm run build` | Uses Vite + ESLint |

GitHub Actions pipelines run on every push, enforce coverage (ingestion), build container images (API), and deploy to AWS on merges to `main`. Use `[skip ci]` in your commit message to bypass pipelines for documentation-only changes.

---

## 🛠 Operations & Monitoring

- **Scheduling ingestion:** BracketForge runs weekly via AWS EventBridge (every Friday at 12:00 UTC). The signal-based gate ensures no work is done unless the IRS page has actually changed.
- **API health:** Spring Boot Actuator exposes `/actuator/health` and `/actuator/info`. Swagger UI at `/swagger-ui/index.html` for manual endpoint checks.
- **Metrics & observability:** `/actuator/prometheus` (basic auth required) is scraped by Grafana Alloy on EC2 and forwarded to Grafana Cloud. The live dashboard tracks API uptime, p95 request latency, data freshness, and calculation throughput. Distributed traces are exported to Grafana Cloud Tempo via the OTel Java agent and utilizing Grafana Explore to correlate latency spikes with specific business operations.
- **Logging:** Ingestion logs to stdout/file path defined in `.env`; API logs structured JSON.
- **Deployment:** Production runs on AWS (EC2, Lambda, RDS, S3, ECR) behind Cloudflare. Local Docker Compose (`rateatlas-api/docker-compose.local.yml`) mirrors the stack with Postgres and the API container.

---

## 📄 Licensing

- **rateatlas-api:** Apache License 2.0 (`rateatlas-api/LICENSE`)
- **rateatlas-ingest & rateatlas-frontend:** License to be finalized (currently unlicensed)

Until a unified license is published at the repo root, treat each module individually and avoid redistributing unlicensed components.

---

## 🗺️ Ecosystem Layout

```txt
RateAtlas/
├── rateatlas-ingest/        # BracketForge - IRS ingestion pipeline (Python · Lambda)
├── rateatlas-api/           # TaxIQ - Spring Boot backend API (EC2)
├── rateatlas-frontend/      # TaxLens - React frontend
└── README.md                # Umbrella documentation
```

---

## 🔗 Links

- **Website:** [https://ratesatlas.com](https://ratesatlas.com)
- **Live API:** [https://api.ratesatlas.com/swagger-ui/index.html](https://api.ratesatlas.com/swagger-ui/index.html)
- **Author:** [@CHA0sTIG3R](https://github.com/CHA0sTIG3R)

---

> *"Mapping how rates change — because understanding the past helps design a smarter tax future."*
