# Dependencies - Backend

## Package Info
- **Name**: backend (not NPM package)
- **Consumers**: web, mobile (via GraphQL)

## Upstream
- `@kaze/models` - Types, schemas, GraphQL
- `@kaze/scraper` - Manga scrapers
- `database` - Schema and queries

## Downstream
- `web` - Consumes GraphQL API
- `mobile` - Consumes GraphQL API

## Coordination
Deploy backend before updating clients.
