# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] - 2026-02-12

### Added
- **BYOAI System**: Bring Your Own API Keys - users can now configure their own API keys
- **Multi-Tenant Support**: Each user can have their own keys for OpenAI, Brave, etc.
- **Multi-Node Dashboard**: Centralized platform to monitor all OpenClaw instances
- **Claude Code Integration**: Support for Claude API as alternative to OpenAI
- **AES-256 Encryption**: API keys are now encrypted at rest

### Changed
- **LemonSqueezy Integration**: Replaced Stripe with LemonSqueezy for billing
- **Pricing Model**: Unified Plan and PricingPlan models
- **IA Provider Priority**: Ollama (local) → OpenAI (client) → Claude (client)

### Fixed
- Security vulnerabilities in environment configuration
- Rate limiting on admin endpoints
- Input validation on license generation

## [1.5.0] - 2026-02-11

### Added
- **Feature Flags**: Gradual rollout and kill switches
- **A/B Testing**: Experiment management system
- **Subscription Usage Tracking**: Daily/monthly limits enforcement
- **Dunning System**: Failed payment recovery workflow

### Changed
- Improved PostgreSQL schema with better indexes
- Optimized message query performance

## [1.4.0] - 2026-02-10

### Added
- **Web Browsing**: Brave API integration for web search
- **License System**: API key licensing for resale
- **ollama Integration**: Local AI as fallback (free)

### Changed
- Refactored AI service to support multiple providers
- Updated Docker Compose for better resource management

## [1.3.0] - 2026-02-09

### Added
- **Customer Portal**: Self-service subscription management
- **Invoice Generation**: Automated billing PDFs
- **Support System**: Ticket management with Telegram integration

## [1.2.0] - 2026-02-08

### Added
- **Redis Caching**: Improved response times
- **Rate Limiting**: Per-user, per-endpoint rate limits
- **Audit Logging**: Complete audit trail for compliance

## [1.1.0] - 2026-02-07

### Added
- **Subscription Management**: Monthly/yearly billing cycles
- **Plan Management**: Admin interface for plan configuration
- **Webhook Handling**: LemonSqueezy webhooks for billing events

## [1.0.0] - 2026-02-06

### Added
- Initial release
- Telegram bot integration with grammY
- OpenAI GPT integration
- PostgreSQL database with Prisma ORM
- Docker Compose for deployment
- Caddy reverse proxy with automatic TLS
