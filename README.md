# Multi-Supplier Shopify Synchronization Platform

> This repository serves as a public technical overview of the project.
>
> The source code remains private because it contains proprietary business logic, supplier integrations, and production infrastructure used in active commercial operations.
>
> This README documents the system architecture, design decisions, synchronization workflows, and technical implementation details.

## Overview

A production-oriented backend platform that automates supplier feed ingestion, product normalization, inventory synchronization, pricing updates, and Shopify catalog synchronization.

The system aggregates product data from multiple suppliers, maintains a centralized product catalog, and synchronizes inventory and pricing changes with Shopify using GraphQL Bulk Operations.

---

## Highlights

- 50,000+ synchronized products
- Multiple supplier integrations
- XML and JSON feed processing
- Automated inventory synchronization
- Automated pricing synchronization
- Shopify GraphQL Bulk Operations
- Product normalization pipeline
- Change detection via hashing
- Linux VPS deployment

---

## Overview

Managing large e-commerce catalogs manually becomes increasingly difficult as supplier count and product volume grow.

Each supplier provides product information in different formats, update frequencies, and structures. Maintaining accurate inventory, pricing, descriptions, and product availability inside Shopify requires significant automation.

This platform was designed to solve that problem.

It continuously imports supplier feeds, normalizes product information into a unified internal model, detects changes, and synchronizes updates with Shopify automatically.

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

### Centralized Product Catalog

A local database acts as the system's source of truth.

The catalog stores:

- Products
- Suppliers
- Supplier offers
- Inventory information
- Pricing information
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

Only products with actual changes are exported for synchronization.

This significantly reduces processing time and API usage.

---

### Shopify Integration

The platform integrates directly with Shopify using the GraphQL Admin API.

Supported workflows include:

- Product creation
- Product updates
- Inventory synchronization
- Variant updates
- Product archival
- Bulk operations

Large product batches are processed through Shopify Bulk Operations, enabling scalable synchronization without exceeding API limitations.

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
      Shopify Store
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
- Product archival

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

#### migrate_shopify_ids.py

One-time migration utility used to associate previously existing Shopify products with local database records.

#### delete.py

Administrative product cleanup utility.

#### delete_all.py

Bulk deletion utility.

---

### Main Pipeline

#### main.py

Application entry point.

Execution flow:

1. Initialize database
2. Import supplier feeds
3. Normalize product data
4. Detect changes
5. Generate Shopify exports
6. Execute bulk synchronization
7. Update inventory and pricing
8. Archive inactive products

---

## Technology Stack

### Programming Language

- Python

### APIs & Integrations

- Shopify GraphQL Admin API

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
