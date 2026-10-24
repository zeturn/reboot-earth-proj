# Reboot Earth

A full-stack forest data explorer for finding, querying and contributing ground-reference surveys and drone mission data. Reboot Earth combines an interactive map, PostgreSQL/PostGIS, an AI-assisted SQL workflow and a reviewed community-data import process.

[Live application](https://reboot.hollowdata.com/data/catalog) · [Repository](https://github.com/zeturn/reboot-earth-proj) · [Backend documentation](backend/README.md)

## What you can do

- **Explore forest data:** browse ground plots and drone missions on a Leaflet map, search the catalog, filter by metadata and data source, and collapse the filter sidebar for more map space.
- **Inspect and compare records:** view footprints, nearby and overlapping surveys, collect datasets, and compare selected records.
- **Query with SQL:** run bounded, read-only PostgreSQL/PostGIS queries and display their returned columns and rows in the browser.
- **Ask the AI assistant:** turn natural-language questions into SQL, execute it against the database, review the results, and retry using database errors or review feedback when needed.
- **Contribute small datasets:** upload a versioned JSON package, receive a validation and AI review report, inspect approved cleaning, and explicitly import accepted records.
- **Export real data:** download metadata, GeoJSON, ground-tree CSV files and ZIP bundles. Drone records can also link to externally hosted imagery products when available.

The frontend uses **Next.js 16, React 19, TypeScript, Tailwind CSS and Leaflet**. The backend uses **FastAPI, SQLAlchemy, GeoPandas and LangChain**, backed by **PostgreSQL 16 / PostGIS 3.4**. AI credentials stay on the server.

## Quick start

Install **Git** and **Docker Desktop with Docker Compose**, then make sure Docker is running. No separate Node.js or Python installation is needed for this path.

### 1. Clone the project and import the data

```bash
git clone https://github.com/zeturn/reboot-earth-proj.git
cd reboot-earth-proj
docker compose run --build --rm ofo-import
```

Compose starts PostGIS and waits for it to become healthy before importing the bundled GeoPackages. An API key is not needed for the import. Existing OFO tables are skipped rather than replaced.

### 2. Configure AI access

```bash
cp backend/.env.example backend/.env
```

On Windows PowerShell, `Copy-Item backend/.env.example backend/.env` does the same thing. Do not overwrite an existing `.env` containing your configuration.

Edit `backend/.env` and add your own key:

```dotenv
OPENAI_API_KEY=your-api-key
OPENAI_MODEL=gpt-4.1-mini
OPENAI_USE_RESPONSES_API=true
```

Keep the other example settings unchanged for the default Compose setup. The backend reads this file at runtime through its development bind mount; Compose supplies the database hostname used inside containers.

The catalog, SQL console and exports work without a model key. AI questions and contribution review require a configured provider. Optional Gemini settings are listed in [backend/.env.example](backend/.env.example); Gemini takes precedence when `GEMINI_API_KEY` is set. Model usage is billed or counted against the configured provider account; free usage is not guaranteed by this project.

Never commit `.env` files or put API keys in `NEXT_PUBLIC_*` variables, Docker build arguments or image layers.

### 3. Start the application

```bash
docker compose up -d --build
```

| Page or service | Local address |
| --- | --- |
| Data landing page | [localhost:3000/data](http://localhost:3000/data) |
| Map and catalog | [localhost:3000/data/catalog](http://localhost:3000/data/catalog) |
| SQL console | [localhost:3000/data/sql](http://localhost:3000/data/sql) |
| Collection comparison | [localhost:3000/data/compare](http://localhost:3000/data/compare) |
| Contribution upload | [localhost:3000/data/upload](http://localhost:3000/data/upload) |
| Interactive API documentation | [localhost:8000/docs](http://localhost:8000/docs) |

Useful local commands:

```bash
docker compose ps
docker compose logs -f backend frontend
docker compose restart backend
docker compose down
```

Restart the backend after changing AI settings. `docker compose down` keeps the database volume; adding `-v` deletes it. The supplied database credentials and loopback-only exposed ports are for local development, not production. If you change database credentials, update the importer and backend connection URLs in Compose as well.

## Data and spatial SQL

The bundled [Open Forest Observatory data](backend/data/ofo/README.md) comes from [Zenodo record 23402904](https://doi.org/10.5281/zenodo.23402904). A fresh import contains:

- **287** ground-reference plots in `ofo_plots`.
- **64,295** measured tree records in `ofo_trees`.
- **687** drone missions in `ofo_drone_missions`.

The API creates `catalog_datasets`, a normalized view of plot and mission records. Community contributions can increase these counts. Geometry is stored in EPSG:4326 with spatial indexes; cast to `geography` for distances in meters.

For example, paste this into the SQL console to find up to 100 trees no taller than 10 meters within 100 meters of a coordinate:

```sql
WITH search_point AS (
    SELECT ST_SetSRID(
        ST_MakePoint(-120.1862197, 39.40647418), 4326
    )::geography AS location
)
SELECT
    t.plot_id,
    t.species_code,
    t.height AS height_m,
    ROUND(ST_Distance(t.geom::geography, p.location)::numeric, 1) AS distance_m
FROM ofo_trees AS t
CROSS JOIN search_point AS p
WHERE t.height <= 10
  AND ST_DWithin(t.geom::geography, p.location, 100)
ORDER BY distance_m
LIMIT 100;
```

The SQL console and AI assistant share the same allowlisted executor: one read-only query, the restricted `ofo_reader` role, a five-second statement timeout and at most 1,000 returned rows. Writes and administrative SQL are rejected. Missing heights are not treated as zero.

GeoPackages contain survey records and mission metadata, not the full imagery or point-cloud collection. External imagery availability depends on its host. Preserve each dataset's attribution and license, and respect `withhold_from_training` restrictions; see the [source-data notes](backend/data/ofo/README.md).

## AI assistant

Each question is processed against the live database schema. The agent proposes SQL, executes it through the read-only service and reviews the returned data. SQL errors and unusable results feed back into a bounded retry loop; explicit user constraints are not silently relaxed to manufacture matches.

Answers can include the executed SQL, real result rows, retry history and dataset references. Proposed catalog actions are surfaced in the interface for the user to apply. Replies default to English, follow clearly non-English questions, and respect explicit language requests.

## Contributing JSON data

Use the **Contribute** page to upload a UTF-8 JSON file or paste JSON. Start with the downloadable [example package](http://localhost:8000/api/uploads/template) and [JSON Schema](http://localhost:8000/api/uploads/schema) after starting the backend. The authoritative contract is [backend/upload_schema.py](backend/upload_schema.py).

- Schema version: `1.0`; maximum package size: **1 MiB**.
- Maximum records per package: **200 plots, 5,000 trees and 200 missions**.
- Include contributor attribution, an accepted license and explicit permission to contribute the data.
- Use WGS84 coordinates and declare measurement units. Trees must reference plots in the same package; drone missions require valid footprints.

Review combines deterministic validation with an AI compatibility assessment. Small, explicit corrections such as trimming text, parsing numeric strings or converting declared units can be proposed and audited. The contribution reviewer cannot execute arbitrary generated code or SQL, invent measurements, or guess coordinates and ambiguous units.

Review stores a report; it does **not** immediately add records to the catalog. Only an approved package can be imported, in a separate atomic transaction. Namespaced identifiers, duplicate-package checks and provenance records protect existing data and track its source. Compatibility approval is not verification of scientific authenticity.

See the [backend contribution guide](backend/README.md#third-party-json-contributions) for the complete example, supported cleaning and API endpoints.

## Development and tests

```text
frontend/          Next.js application, map, catalog and contribution UI
backend/           FastAPI routes, AI workflow, SQL validation and import logic
backend/data/ofo/  Bundled GeoPackages and source metadata
deploy/            BeanCS deployment helpers and non-secret target configuration
docs/              Production operations and recovery notes
docker-compose.yml Local PostGIS, importer, API and frontend services
```

For development outside Docker, use Node.js 22 and Python 3.12, matching CI. See [backend setup](backend/README.md#run-locally). Frontend commands run from `frontend/`:

```bash
npm ci
npm run dev
npm run lint
npm run build
```

Run backend checks against a **local or disposable database**, not production; contribution tests exercise database writes. The default suite skips live model calls:

```bash
docker compose exec -T backend python -m unittest discover -s tests -v
```

Run deployment safety tests from the repository root with a local Python installation:

```bash
python -m unittest discover -s deploy/tests -v
```

Pull requests and pushes to `main` run [GitHub Actions CI](.github/workflows/ci.yml): a disposable PostGIS import, backend tests, deployment safety tests, frontend lint and a production build.

## Production and deployment

The live application runs on the BeanCS-managed k3s cluster at [reboot.hollowdata.com](https://reboot.hollowdata.com/data/catalog). Updates are currently performed **manually through SSH and the BeanCS V2 API**. The tag-triggered release workflow has been removed: pushing a tag does **not** currently deploy the site automatically.

Production uses `frontend/Dockerfile.prod` with an empty `NEXT_PUBLIC_API_URL` build argument, plus `backend/Dockerfile` without development bind mounts or reload. The ingress routes `/api` to FastAPI and other paths to Next.js. Database and model credentials belong in server-side Secrets, not Git.

Image updates must preserve the persistent PostGIS volume, community submissions and existing runtime configuration. The importer is non-destructive by default; never use `--replace` on a shared database or delete a database volume as a redeployment step. See the [production runbook](docs/DEPLOYMENT.md) for storage, release verification, credentials and recovery.

The public demo currently allows AI usage and contribution imports without account-level access control. SQL restrictions and ingress rate limits are safeguards, not authentication or a spending cap. Add authentication, authorization, moderation and usage budgets before using this as a trusted multi-user service. Keep independent database backups: bundled OFO files cannot recover later contributions.

## Contributing code and attribution

Open pull requests against [zeturn/reboot-earth-proj](https://github.com/zeturn/reboot-earth-proj). The project retains the original commit history and author attribution from [SrutiElan/reboot-the-earth](https://github.com/SrutiElan/reboot-the-earth); preserve that history when syncing upstream changes.

If you already have the original repository cloned, inspect `git remote -v`, keep it as `upstream`, and use `zeturn/reboot-earth-proj` as `origin` for future pushes. Original authors remain credited in the [Contributors page](https://github.com/zeturn/reboot-earth-proj/graphs/contributors). Data licenses are specified per source record; do not assume one blanket license covers every dataset.
