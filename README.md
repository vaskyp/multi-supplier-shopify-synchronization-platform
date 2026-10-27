# Multi-Supplier Shopify Synchronization Platform

> This repository serves as a public technical overview of the project.
>
> The source code remains private because it contains proprietary business logic, supplier integrations, and production infrastructure used in active commercial operations.
>
> This README documents the system architecture, design decisions, synchronization workflows, and technical implementation details.

## Overview

A production-oriented backend platform that automates supplier feed ingestion, product normalization, product data enrichment, inventory synchronization, pricing updates, and Shopify catalog synchronization.

The system aggregates product data from multiple suppliers, maintains a centralized product catalog, enriches products with third-party product content, and synchronizes inventory, pricing, and content changes with Shopify using GraphQL Bulk Operations.

---

## Highlights

- 50,000+ synchronized products
- Multiple supplier integrations
- XML and JSON feed processing
- Automated inventory synchronization
- Automated pricing synchronization
- Automated product content enrichment (descriptions, specifications, PDF leaflets, reviews)
- Shopify GraphQL Bulk Operations
- Product normalization pipeline
- Change detection via hashing
- Concurrent enrichment with resumable batch processing
- Linux VPS deployment

---

## Background

Managing large e-commerce catalogs manually becomes increasingly difficult as supplier count and product volume grow.

Each supplier provides product information in different formats, update frequencies, and structures. Maintaining accurate inventory, pricing, descriptions, and product availability inside Shopify requires significant automation. Supplier data is also often incomplete, with missing descriptions, specifications, or documentation.

This platform was designed to solve that problem.

It continuously imports supplier feeds, normalizes product information into a unified internal model, fills content gaps from external product databases, detects changes, and synchronizes updates with Shopify automatically.

The platform currently supports catalogs exceeding 50,000 products across multiple suppliers.

---

## Core Capabilities

### Supplier Feed Processing

The platform imports and processes supplier feeds provided in different formats.

Supported feed types include:

- XML feeds
- JSON feeds
- Compressed feeds (GZIP)
- Full catalog feeds
- Incremental offer feeds

Each supplier implementation contains dedicated parsing and transformation logic while exposing a common interface through a shared supplier abstraction layer.

---

### Product Normalization

Supplier feeds rarely follow the same structure.

The normalization layer transforms supplier-specific data into a unified product model.

Normalized fields include:

- Product name
- Description
- Category
- Brand
- Weight
- Images
- Stock levels
- Pricing
- EAN identifiers

This allows products from different suppliers to be processed consistently throughout the system.

---

### Product Data Enrichment

Products are matched by EAN against an external product content database to fill gaps left by supplier feeds.

Enrichment includes:

- Product descriptions
- Technical specifications
- PDF product leaflets
- Customer reviews and rating scores

Key design decisions:

- **Fill gaps, don't overwrite:** existing descriptions are never replaced by external content
- **Query once:** every product is checked at most once, so later runs only process newly added products
- **Concurrent lookups:** external requests run in parallel worker threads
- **Queued pushes:** enriched data is flagged locally and pushed to Shopify afterwards; failed pushes stay queued and are retried on the next run

Descriptions are synchronized to the Shopify product, while specifications and PDF leaflets are stored in Shopify metafields. Reviews are published to the store's review platform through its API.

Dedicated maintenance scripts allow re-processing a specific set of products (for example, only descriptions or only specifications) without sweeping the whole catalog, including a dry-run mode for safe inspection.

---

### Centralized Product Catalog

A local database acts as the system's source of truth.

The catalog stores:

- Products
- Suppliers
- Supplier offers
- Inventory information
- Pricing information
- Enrichment data (descriptions, specifications, leaflets, reviews)
- Shopify identifiers
- Synchronization metadata

The database layer includes:

- Schema migrations
- Performance indexing
- In-memory caching
- Batch operations
- Shopify ID persistence

---

### Change Detection Engine

To avoid unnecessary Shopify API calls, the platform implements a change detection mechanism.

Each supplier offer is fingerprinted using:

- Price
- Suggested retail price
- Stock quantity

When a supplier update is processed, hashes are compared against previously stored values.

A second fingerprint covers the fields actually exported to Shopify. Only products whose export fingerprint differs from the last successful push are included in the update stream.

This significantly reduces processing time and API usage.

---

### Shopify Integration

The platform integrates directly with Shopify using the GraphQL Admin API.

Supported workflows include:

- Product creation
- Product updates
- Inventory synchronization
- Variant updates
- Metafield updates
- Product archival
- Bulk operations

Large product batches are processed through Shopify Bulk Operations, enabling scalable synchronization without exceeding API limitations.

Variant and inventory updates after bulk creation run in parallel with checkpoint-based resume support, so an interrupted run can continue where it stopped.

---

### Inventory & Pricing Automation

The synchronization layer automatically:

- Updates inventory quantities
- Updates product pricing
- Updates variant information
- Publishes new products
- Archives inactive products

This ensures that the Shopify catalog remains aligned with supplier availability and pricing.

---

### Product Lifecycle Management

The platform manages the complete lifecycle of a product.

Supported states include:

- New products
- Active products
- Updated products
- Out-of-stock products
- Archived products

Products that no longer meet business rules can be automatically archived and excluded from future synchronization cycles.

---

### Image Processing & Optimization

Product images are automatically collected from supplier feeds and transformed into Shopify-compatible assets.

The platform supports:

- Multi-image products
- Image normalization
- CDN integration
- CDN cache warm-up before product creation
- Optimized image delivery
- Supplier-specific image handling

---

## Architecture

```text
Supplier Feeds
(XML / JSON / GZIP)
            │
            ▼
    Supplier Parsers
            │
            ▼
 Product Normalization
            │
            ▼
     Local Database
            │
            ├──────────────► Product Enrichment
            │                (descriptions, specs, leaflets, reviews)
            │                        │
            │◄───────────────────────┘
            ▼
   Change Detection
            │
            ▼
 Shopify JSONL Export
            │
            ▼
 Shopify Bulk Operations
            │
            ▼
      Shopify Store  ◄──── Review Platform (reviews)
```

---

## Project Structure

### Core Layer

#### database.py

Responsible for:

- Database initialization
- Schema management
- Migrations
- Product persistence
- Supplier persistence
- Change tracking
- Enrichment data storage
- Batch operations
- Shopify ID storage

---

#### shopify_api.py

Shopify integration layer.

Responsibilities include:

- GraphQL communication
- Rate-limit handling
- Retry logic
- Bulk operations
- Staged uploads
- Inventory updates
- Product updates
- Description and metafield pushes
- Product archival

---

#### content_client.py

Client for the external product content API.

Responsibilities include:

- Lookup by EAN
- Response normalization (description, specifications, leaflet URL, reviews)
- Retry handling

---

#### review_client.py

Client for the review platform.

Responsibilities include:

- Publishing reviews per product
- Mapping stored review data to the platform's format

---

#### supplier_base.py

Abstract base class for supplier integrations.

Provides:

- Feed acquisition
- Connection management
- Parsing orchestration
- Shared synchronization workflow

---

### Supplier Integrations

#### supplier_X.py

Each supplier implementation contains dedicated parsing logic for handling supplier-specific feed structures.

---

### Enrichment Layer

#### enrichment.py

Runs the enrichment sweep and the follow-up pushes:

1. Select products with an EAN that have not been checked yet
2. Fetch external product data concurrently
3. Store results locally using the fill-gaps policy
4. Push queued descriptions, specifications, leaflets, and reviews

Supports test runs (limited batches), sweep-only mode, and push-only mode.

---

### Export Layer

#### export_shopify_jsonl.py

Generates Shopify-compatible JSONL files for:

- Product creation
- Product updates
- Variant updates
- Inventory synchronization

Files are automatically split into chunks suitable for Shopify Bulk Operations.

---

### Utility Scripts

#### bootstrap_from_shopify.py

Builds the local database from an existing Shopify store.

#### migrate_shopify_ids.py

One-time migration utility used to associate previously existing Shopify products with local database records.

#### Enrichment maintenance scripts

Re-process descriptions or specifications for a defined set of products, with dry-run support.

#### reviews.py

Recalculates rating scores and publishes stored reviews to the review platform.

#### check_image_domains.py

Audits which external image hosts are referenced by the catalog.

#### delete.py / delete_all.py

Administrative product cleanup utilities. Deletion requires explicit confirmation and creates a backup first.

---

### Main Pipeline

#### main.py

Application entry point.

Execution flow:

1. Initialize database
2. Import supplier feeds
3. Normalize product data
4. Enrich products with external content and push enrichment data
5. Archive products below business-rule thresholds
6. Detect changes and generate Shopify exports
7. Warm the image CDN cache for new products
8. Execute bulk creation
9. Update variants, inventory and pricing
10. Bulk update existing products

---

## Technology Stack

### Programming Language

- Python

### APIs & Integrations

- Shopify GraphQL Admin API
- External product content API
- Review platform API

### Data Processing

- XML
- JSON
- JSONL
- GZIP

### Database

- SQLite

### Infrastructure

- Linux
- VPS Hosting
- Scheduled execution (cron)
- Environment-based configuration

### Integrations

- Shopify
- ImageKit CDN
- Supplier Catalog Systems

---

## Scalability Considerations

The platform was designed around large product catalogs and supplier-driven synchronization workflows.

Key optimizations include:

- Streaming XML parsing
- Batch database operations
- In-memory caching
- Change detection via hashing
- Shopify bulk mutations
- Chunked exports
- Concurrent external lookups
- Checkpoint-based resume support
- Retry and rate-limit handling

These optimizations allow efficient synchronization of catalogs containing tens of thousands of products.

---

## Future Improvements

Planned enhancements include:

- PostgreSQL migration
- Web-based administration dashboard
- Real-time synchronization workflows
- Advanced monitoring and analytics
- Distributed background processing

---

## Status

Production use for active e-commerce operations.

The implementation remains private due to commercial use and proprietary supplier integrations. This repository is intended to provide a technical overview of the platform architecture and synchronization workflows.
