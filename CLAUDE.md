# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is **Open Semantic Search (OSS)** - an integrated open source search engine, ETL framework, and collection of search applications for fulltext search, faceted search, named entity recognition, OCR, and knowledge graph search.

**IMPORTANT**: This repository is a collection of submodules used for building the OSS appliance. **We are examining the code, not building it.** Focus on code analysis, understanding architecture, and answering questions about the codebase.

**CRITICAL**: The Docker configurations and docker-compose files you see in this repository were the developer's future plans and **DO NOT WORK**. The actual production deployment uses the **older standalone appliance** - a single VM installation using `.deb` packages with systemd services. All Docker references are aspirational/non-functional.

The repository is organized as a **monorepo with Git submodules**. Each major component is maintained in its own subdirectory under the root, with many being separate Git repositories linked as submodules. The main integration happens in the `open-semantic-search` directory.

## Common Abbreviations

- **OSS** = Open Semantic Search
- **ETL** = Extract, Transform, Load
- **NER** = Named Entity Recognition
- **OCR** = Optical Character Recognition

## Reference: Build & Test Commands

These commands are documented for reference when examining the build process. **We are not building this repository.**

### Build Information

- Debian package: `./build-deb` (creates .deb for Debian/Ubuntu) - **This is what's actually used**
- Test scripts: `open-semantic-etl/src/opensemanticetl/testdata/run_tests.sh`

### Docker Information (Non-Functional)

**IMPORTANT:** The following Docker-related files exist in the repository but **DO NOT WORK** and are not used in production:
- `docker-compose.yml` - Developer's future plans, non-functional
- `docker-compose.etl.test.yml` - Non-functional
- `docker-compose.test.yml` - Non-functional
- Various `Dockerfile`s throughout the repository - Aspirational only

**Actual Deployment:** Single VM appliance using `.deb` packages installed via `dpkg`/`apt` with systemd service management.

## Architecture

### High-Level Data Flow

1. **Data Sources** → Files, websites, RSS feeds, databases
2. **Crawlers/Connectors** → Discover and queue documents for processing
3. **Task Queue (RabbitMQ + Celery)** → Manages parallel processing
4. **ETL Workers** → Process documents through plugin pipeline:
   - **Apache Tika** → Text and metadata extraction
   - **Tesseract OCR** → Text recognition from images
   - **spaCy NLP** → Named entity recognition (NER) via machine learning
   - **Entity Search API** → Entity extraction using thesaurus/ontologies
   - **Annotation plugins** → Add tags and human annotations
5. **Indexers** → Store in Apache Solr (primary) or Elasticsearch
6. **Graph Database (Neo4j)** → Optional linked data storage
7. **Web UI** → Search interface (Solr-PHP-UI + Django apps)

### Major Components

- **`open-semantic-etl/`** - Core ETL framework and plugins (Python)
  - ETL pipeline orchestration via Celery
  - Plugin architecture for data enrichment
  - Located: `/usr/lib/python3/dist-packages/opensemanticetl/` when installed
  - Main dependencies in: `src/opensemanticetl/requirements.txt`

- **`open-semantic-search-apps/`** - Django web applications (Python)
  - Thesaurus, ontologies, annotations, crawler management
  - Each subdirectory is a Django app: `annotate/`, `thesaurus/`, `ontologies/`, etc.
  - Main entry point: `src/manage.py`
  - Located: `/var/lib/opensemanticsearch/` when installed

- **`solr-php-ui/`** - Search user interface (PHP)
  - Responsive UI for search, faceted navigation, previews
  - Config: `/etc/solr-php-ui/`
  - Located: `/usr/share/solr-php-ui/` when installed

- **`solr.deb/`** - Apache Solr search server package
  - Preconfigured Solr with custom schema
  - Schema: `var/solr/data/opensemanticsearch/conf/managed-schema`
  - Data: `/var/solr/data/`

- **`tika-server.deb/`** - Apache Tika server package
  - Document parsing and metadata extraction
  - Integrated with Tesseract for OCR

- **`spacy-services.deb/`** - spaCy NLP microservice
  - Named entity recognition via machine learning models

- **`open-semantic-entity-search-api/`** - Entity extraction API
  - Dictionary/thesaurus-based entity linking
  - Integrates with Solr entities index

### Service Architecture (Appliance Deployment)

The OSS appliance runs as a collection of systemd services on a single VM:

**Core Services:**
1. **rabbitmq-server** - Message queue for Celery task distribution
2. **solr** - Apache Solr search index server (port 8983)
3. **neo4j** - Graph database for knowledge graph (optional but typically present)
4. **apache2** - Web server hosting:
   - Django web apps (thesaurus, ontologies, annotations, etc.)
   - Solr-PHP-UI search interface
   - Entity Search API
5. **opensemanticetl** - Celery workers executing ETL pipeline
6. **tika** - Apache Tika server for text extraction (port 9998)
7. **spacy-services** - spaCy NLP service for NER (port 8080)

**Service Startup Order:**
Critical: RabbitMQ → Solr (wait ~10s) → Neo4j → Apache → ETL workers

### Configuration (Appliance)

- **ETL config**: `/etc/opensemanticsearch/`
- **UI config**: `/etc/solr-php-ui/`
- **Cron jobs**: `/etc/cron.d/open-semantic-search`
- **Django config**: `/etc/opensemanticsearch-django-webapps/` (if exists)

**IMPORTANT:** Configuration is fragile. Never modify `/etc/` configs unless absolutely necessary.

## Python Dependencies

Main ETL dependencies (`open-semantic-etl/src/opensemanticetl/requirements.txt`):
- celery - Task queue management
- pysolr - Solr client
- tika - Tika client
- requests - HTTP client
- lxml, feedparser - Parsing
- rdflib, SPARQLWrapper - Semantic web/ontologies
- py2neo - Neo4j client
- scrapy - Web scraping
- pyinotify - Filesystem monitoring

Install via pip:
```bash
pip3 install -r open-semantic-etl/src/opensemanticetl/requirements.txt
```

## Working with Submodules

This repo uses Git submodules extensively. Clone with:

```bash
git clone --recurse-submodules --remote-submodules https://github.com/opensemanticsearch/open-semantic-search.git
```

Submodules are defined in `.gitmodules` and located in subdirectories like:
- `open-semantic-etl/`
- `open-semantic-search-apps/`
- `solr-php-ui/`
- `tika-server.deb/`
- etc.

## Key File Locations (Appliance Installation)

When installed from .deb package on the appliance VM:

- **Python libraries**: `/usr/lib/python3/dist-packages/`
  - `opensemanticetl/` - ETL framework
  - `entity_linking/`, `entity_manager/` - Entity extraction
  - `spacy-services/` - NLP service

- **Web applications**: `/var/lib/opensemanticsearch/`
  - Django apps for thesaurus, ontologies, annotations, etc.
  - Entry point: `manage.py`

- **Search UI**: `/usr/share/solr-php-ui/`

- **Data directories** (cleared when resetting system):
  - `/var/solr/data/opensemanticsearch/data/` - Main document index data
  - `/var/solr/data/opensemanticsearch-entities/data/` - Entity index data
  - `/var/opensemanticsearch/db/db.sqlite3` - Django SQLite database (file names, configs)
  - `/var/opensemanticsearch/media/` - Uploaded files and thumbnails
  - `/var/cache/tesseract/` - OCR cache
  - `/var/lib/neo4j/data/` - Graph database

- **Configuration directories** (NEVER delete - system is fragile):
  - `/etc/opensemanticsearch/` - ETL configuration
  - `/etc/solr-php-ui/` - UI configuration
  - `/var/solr/data/opensemanticsearch/conf/` - Solr schema (keep!)
  - `/var/solr/data/opensemanticsearch-entities/conf/` - Entity schema (keep!)

- **Logs**:
  - `/var/solr/logs/` - Solr logs
  - `/var/log/apache2/` - Web server logs
  - `journalctl -u opensemanticetl` - ETL worker logs
  - `journalctl -u solr` - Solr service logs

## Documentation

- User/admin documentation: `open-semantic-search/docs/doc/`
- Architecture docs: `open-semantic-search/docs/doc/modules/README.md`
- Documentation site built with MkDocs: `open-semantic-search/mkdocs.yml`
- Build docs: `mkdocs build --config-file open-semantic-search/mkdocs.yml`

Online documentation: https://opensemanticsearch.org/doc/

## ETL Plugin Development

ETL plugins are Python modules in `open-semantic-etl/src/opensemanticetl/` that implement data enrichment. Each plugin typically:

1. Receives a document dictionary with metadata and content
2. Processes/analyzes the content
3. Adds enriched data back to the document dictionary
4. Returns the modified document

Common plugin patterns:
- `enhance_*.py` - Data enrichment plugins
- `export_*.py` - Output/indexing plugins (e.g., Solr, Neo4j, Elasticsearch)
- `etl_*.py` - Input connectors

The ETL worker (`tasks.py`) chains configured plugins together in sequence.

## System Reset Information

See `RESET_SYSTEM.md` for detailed procedures on wiping a production clone to prepare for new indexing jobs. Key targets:
- SQLite database with file names/paths (PRIMARY REASON FOR RESET)
- Both Solr indices (documents and entities)
- Neo4j graph database (production clones have data here)
- Media files and OCR cache

**Critical:** Always preserve configurations in `/etc/` - the system is fragile.
