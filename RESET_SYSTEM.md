# Resetting an OSS System to Clean State

This document describes how to wipe a clone of a production Open Semantic Search (OSS) system to get a clean starting point for new indexing jobs.

**IMPORTANT:** This guide is for the OSS appliance (single VM installation via .deb package). Docker deployment is NOT used.

## Overview of Persistent Storage

OSS stores data in multiple locations across different services. All must be cleared to achieve a truly clean state.

## Storage Locations (Appliance VM Installation)

### 1. Django SQLite Database
**Location:** `/var/opensemanticsearch/db/db.sqlite3`

**Contains:**
- Configured file paths and directories (`files` app) - **PRIMARY REASON FOR RESET**
- Web crawler configurations (`crawler` app)
- RSS feed configurations (`rss_manager` app)
- Thesaurus concepts and entities (`thesaurus` app)
  - Facets, Groups, Concepts (with prefLabel, altLabel, hiddenLabel)
  - SKOS relationships (broader, narrower, related)
- Ontologies (`ontologies` app)
- Document annotations and tags (`annotate` app)
- Morphology settings (`morphology` app)
- Search lists (`search_list` app)

**Reset Method:**
```bash
systemctl stop apache2
rm /var/opensemanticsearch/db/db.sqlite3
systemctl start apache2
```

The Django apps will recreate an empty schema on first access.

### 2. Solr Search Indices
**Location:** `/var/solr/data/`

**Contains TWO Solr cores:**

#### Core 1: `opensemanticsearch` (Main Document Index)
- Full-text indexed documents
- Extracted text from files
- Document metadata
- Facets for navigation
- **Data:** `/var/solr/data/opensemanticsearch/data/`
- **Schema:** `/var/solr/data/opensemanticsearch/conf/managed-schema` (DO NOT DELETE)

#### Core 2: `opensemanticsearch-entities` (Entity Index)
- Named entities extracted from documents
- Used by Entity Search API
- **Data:** `/var/solr/data/opensemanticsearch-entities/data/`
- **Schema:** `/var/solr/data/opensemanticsearch-entities/conf/managed-schema` (DO NOT DELETE)

**Reset Method (Option 1 - Delete index data files):**
```bash
systemctl stop solr
rm -rf /var/solr/data/opensemanticsearch/data/*
rm -rf /var/solr/data/opensemanticsearch-entities/data/*
systemctl start solr
```

**Reset Method (Option 2 - API delete, keeps schema):**
```bash
# For main document core
curl http://localhost:8983/solr/opensemanticsearch/update?commit=true -d '<delete><query>*:*</query></delete>'

# For entities core
curl http://localhost:8983/solr/opensemanticsearch-entities/update?commit=true -d '<delete><query>*:*</query></delete>'
```

### 3. Neo4j Graph Database
**Location:** `/var/lib/neo4j/data/` (default Neo4j installation)

**Contains:**
- Knowledge graph relationships
- Linked data exported by ETL
- **IMPORTANT:** This data probably exists in your production clone and MUST be cleared

**Reset Method (Option 1 - Full database deletion):**
```bash
systemctl stop neo4j
rm -rf /var/lib/neo4j/data/databases/*
rm -rf /var/lib/neo4j/data/transactions/*
systemctl start neo4j
```

**Reset Method (Option 2 - CYPHER query):**
```bash
# Connect to Neo4j and run:
cypher-shell -u neo4j -p <password>
MATCH (n) DETACH DELETE n;
:exit
```

**Note:** Check your Neo4j configuration for the actual data directory location. It may vary.

### 4. Media Files and Thumbnails
**Location:** `/var/opensemanticsearch/media/`

**Contains:**
- Uploaded ontology files
- Generated document thumbnails
- Uploaded CSV files for entity import

**Reset Method:**
```bash
rm -rf /var/opensemanticsearch/media/*
```

### 5. Tesseract OCR Cache
**Location:** `/var/cache/tesseract/`

**Contains:**
- Cached OCR results for images
- Prevents re-OCR of already processed images

**Reset Method:**
```bash
rm -rf /var/cache/tesseract/*
```

### 6. RabbitMQ Task Queue
**Location:** `/var/lib/rabbitmq/`

**Contains:**
- Pending ETL tasks in queue
- Celery job queue

**Reset Method:**
```bash
systemctl stop rabbitmq-server
systemctl stop opensemanticetl
# Queue will clear on restart
systemctl start rabbitmq-server
systemctl start opensemanticetl
```

Or purge specific queues while running:
```bash
rabbitmqctl purge_queue celery
```

### 7. Configuration Files (**KEEP THESE**)
**Locations:**
- `/etc/opensemanticsearch/` - ETL configuration
- `/etc/solr-php-ui/` - Search UI configuration
- `/etc/opensemanticsearch-django-webapps/` - Django web apps configuration (if exists)

**Contains:**
- ETL pipeline configuration
- Search UI settings
- Service configurations

**IMPORTANT:** DO NOT delete or modify these. The configs are fine and the system is fragile.

### 8. Elasticsearch (Not Enabled by Default)
**Location:** `/var/lib/elasticsearch/` (if installed)

**Contains:**
- Alternate document index (when enabled)

**Note:** Elasticsearch export is NOT enabled by default and typically has no data. You can ignore this unless specifically enabled.

**Reset Method (if enabled):**
```bash
curl -X POST "localhost:9200/opensemanticsearch/_delete_by_query" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  }
}'
```

## Complete System Reset Procedure (Appliance VM)

### IMPORTANT: System is Fragile

The running OSS appliance is fragile. Follow these procedures carefully and in order.

**ALWAYS KEEP:** Configuration directories in `/etc/`

### Standard Reset Procedure (Recommended)

This is the safe reset for a clone of production appliance to prepare for new indexing jobs:

```bash
#!/bin/bash
# OSS Appliance Reset Script
# Run as root or with sudo

echo "Stopping all OSS services..."
systemctl stop opensemanticetl
systemctl stop apache2
systemctl stop solr
systemctl stop neo4j
systemctl stop rabbitmq-server

echo "Clearing SQLite database (file names/paths)..."
rm -f /var/opensemanticsearch/db/db.sqlite3

echo "Clearing Solr indices..."
rm -rf /var/solr/data/opensemanticsearch/data/*
rm -rf /var/solr/data/opensemanticsearch-entities/data/*

echo "Clearing Neo4j graph database..."
rm -rf /var/lib/neo4j/data/databases/*
rm -rf /var/lib/neo4j/data/transactions/*

echo "Clearing media files and thumbnails..."
rm -rf /var/opensemanticsearch/media/*

echo "Clearing OCR cache..."
rm -rf /var/cache/tesseract/*

echo "Starting services..."
systemctl start rabbitmq-server
sleep 5
systemctl start solr
sleep 10
systemctl start neo4j
sleep 5
systemctl start apache2
sleep 5
systemctl start opensemanticetl

echo "Reset complete. System is ready for new indexing jobs."
echo "Verify at: http://localhost/search/"
```

**What gets cleared:**
- ✅ SQLite database with file paths/names (PRIMARY TARGET)
- ✅ Both Solr indices (documents and entities)
- ✅ Neo4j graph database (has production data)
- ✅ Media files and thumbnails
- ✅ OCR cache
- ✅ RabbitMQ queue (clears on restart)

**What is preserved:**
- ✅ All configuration in `/etc/`
- ✅ Service configurations
- ✅ ETL pipeline settings

### Alternative: Minimal Reset (Clear Documents Only, Keep Everything Else)

If you want to preserve thesaurus, ontologies, crawler configs, but remove indexed documents:

```bash
#!/bin/bash
# Minimal reset - documents only

echo "Clearing Solr indices via API..."
curl http://localhost:8983/solr/opensemanticsearch/update?commit=true -d '<delete><query>*:*</query></delete>'
curl http://localhost:8983/solr/opensemanticsearch-entities/update?commit=true -d '<delete><query>*:*</query></delete>'

echo "Clearing Neo4j graph..."
systemctl stop opensemanticetl
cypher-shell -u neo4j -p <password> "MATCH (n) DETACH DELETE n;"

echo "Clearing OCR cache..."
rm -rf /var/cache/tesseract/*

systemctl start opensemanticetl

echo "Minimal reset complete. Thesaurus and configs preserved."
```

**Warning:** This does NOT clear the file names in SQLite, so crawlers may skip files thinking they're already indexed.

## Verification After Reset

After reset, verify the appliance is clean:

1. **Check Solr document count:**
   ```bash
   curl 'http://localhost:8983/solr/opensemanticsearch/select?q=*:*&rows=0'
   # Should show numFound="0"

   curl 'http://localhost:8983/solr/opensemanticsearch-entities/select?q=*:*&rows=0'
   # Should show numFound="0"
   ```

2. **Check Django database:**
   ```bash
   cd /var/lib/opensemanticsearch
   python3 manage.py shell
   >>> from files.models import Files
   >>> Files.objects.count()
   0
   >>> from crawler.models import Crawler
   >>> Crawler.objects.count()
   0
   >>> exit()
   ```

3. **Check Neo4j:**
   ```bash
   cypher-shell -u neo4j -p <password> "MATCH (n) RETURN count(n);"
   # Should return 0
   ```

4. **Browse UI:**
   - Visit http://localhost/search/ (or http://localhost:8080/search/ if configured)
   - Search for `*:*` should return 0 results
   - UI should be responsive and show empty facets

5. **Check services are running:**
   ```bash
   systemctl status solr
   systemctl status apache2
   systemctl status opensemanticetl
   systemctl status rabbitmq-server
   systemctl status neo4j
   ```

## Service Management

The OSS appliance uses systemd services. Key services and their startup order:

1. **rabbitmq-server** - Message queue (start first)
2. **solr** - Search index (start second, wait ~10s for initialization)
3. **neo4j** - Graph database (can start in parallel with Solr)
4. **apache2** - Web server for UI and Django apps
5. **opensemanticetl** - Celery workers for ETL processing (start last)

**Check service status:**
```bash
systemctl status rabbitmq-server solr neo4j apache2 opensemanticetl
```

**View service logs:**
```bash
journalctl -u opensemanticetl -f        # ETL worker logs
journalctl -u solr -f                    # Solr logs
journalctl -u apache2 -f                 # Web server logs
tail -f /var/solr/logs/solr.log          # Detailed Solr logs
```

## Important Notes

1. **System is Fragile:** Follow procedures carefully, the appliance is fragile
2. **Always Keep Configs:** Never delete anything in `/etc/opensemanticsearch/` or `/etc/solr-php-ui/`
3. **Neo4j Data Exists:** Production clones likely have Neo4j data that MUST be cleared
4. **Thesaurus/Ontology Lost:** These are in Django SQLite DB, will be cleared in reset
5. **Crawler Configs Lost:** Also in Django DB, will need to be reconfigured after reset
6. **File Path Records:** The SQLite DB contains file names/paths - this is the PRIMARY reason for the reset
7. **No Undo:** File deletion is permanent - make backups if unsure
8. **Elasticsearch Usually Empty:** Not enabled by default, can be ignored
9. **Service Startup Order:** RabbitMQ → Solr (wait) → Neo4j → Apache → ETL workers
10. **Wait for Solr:** Always wait ~10 seconds after starting Solr before starting other services

## Quick Reference: What Needs Clearing

For a clone of production appliance being reset for new jobs:

**MUST CLEAR:**
- ✅ `/var/opensemanticsearch/db/db.sqlite3` - SQLite with file names/paths
- ✅ `/var/solr/data/opensemanticsearch/data/*` - Main document index
- ✅ `/var/solr/data/opensemanticsearch-entities/data/*` - Entity index
- ✅ `/var/lib/neo4j/data/databases/*` - Graph database (has production data)
- ✅ `/var/cache/tesseract/*` - OCR cache
- ✅ `/var/opensemanticsearch/media/*` - Uploaded files/thumbnails

**MUST KEEP:**
- ⛔ `/etc/opensemanticsearch/` - ETL configuration
- ⛔ `/etc/solr-php-ui/` - UI configuration
- ⛔ `/var/solr/data/*/conf/` - Solr schemas and configs
- ⛔ All other `/etc/` directories

**IGNORE:**
- ⚠️ Elasticsearch - Not enabled by default, no data

## Troubleshooting

**Solr won't start after reset:**
- Check that you didn't delete the `conf/` directories
- Check permissions: `chown -R solr:solr /var/solr/data/`
- View logs: `tail -f /var/solr/logs/solr.log`

**Django database errors:**
- Run migrations: `cd /var/lib/opensemanticsearch && python3 manage.py migrate`
- Check permissions: `chown -R www-data:www-data /var/opensemanticsearch/db/`

**ETL workers not processing:**
- Check RabbitMQ: `systemctl status rabbitmq-server`
- Check queue: `rabbitmqctl list_queues`
- View worker logs: `journalctl -u opensemanticetl -f`

**Neo4j authentication errors:**
- Reset password if needed after data wipe
- Check `/var/lib/neo4j/` permissions
