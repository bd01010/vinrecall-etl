# VinRecall ETL Pipeline

NHTSA data extraction, transformation, and campaign page generation for VinRecall.

## Architecture

This repository contains the backend ETL pipeline that:
- Polls NHTSA Recalls API (2 QPS rate limit)
- Transforms recall data into structured format
- Generates campaign pages with SEO-optimized HTML
- Updates site repository via GitHub API

## Components

- **Extract:** NHTSA API client with rate limiting
- **Transform:** Data normalization and enrichment
- **Load:** Database persistence (Cloud SQL)
- **Generate:** Campaign page HTML generation
- **Publish:** GitHub API integration for site updates

## Deployment

Cloud Run service with Cloud Scheduler for hourly polling.

## Related Repositories

- [vinrecall](https://github.com/bd01010/vinrecall) - Public site repository
