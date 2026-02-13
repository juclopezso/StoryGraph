# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

StoryGraph is a web platform for creating and reading non-linear interactive stories. Readers navigate through story "nodes" (text fragments) connected via choices, creating unique paths through graph-structured narratives. This is a **course project currently in the design/architecture phase** — no source code has been implemented yet.

All documentation is written in **Spanish**.

## Planned Technology Stack

- **Frontend:** React.js or Vue.js SPA, served via S3 + CloudFront
- **Backend:** Node.js + Express or Python + FastAPI
- **Databases:** PostgreSQL (users, metadata), MongoDB (story nodes/graph structure), Redis (cache/sessions)
- **Search:** Elasticsearch / OpenSearch
- **Events:** Amazon EventBridge / Kafka / SNS+SQS (CloudEvents format)
- **Auth:** JWT tokens
- **Graph Visualization:** D3.js or Cytoscape.js
- **Deployment:** AWS (ECS Fargate or EKS), API Gateway, CloudFront

## Architecture

The system follows a **microservices + event-driven** architecture with 7 services:

| Service | Responsibility | Database |
|---------|---------------|----------|
| **Auth Service** | Registration, login, JWT, roles (Creator/Reader) | PostgreSQL |
| **Story Service** | Story CRUD, metadata, publication | PostgreSQL + MongoDB |
| **Graph Service** | Narrative graph engine: nodes, edges, validation | MongoDB |
| **Reader Service** | Reading experience, progress, choice history | MongoDB + Redis |
| **Analytics Service** | Consumption statistics, popular paths, abandonment | PostgreSQL/ClickHouse |
| **Notification Service** | Email/push on story publish, welcome emails | — (SES/SendGrid) |
| **Search Service** | Full-text story indexing and search | OpenSearch |

### Key Patterns

- **CQRS:** Write models (MongoDB) separated from read-optimized stores (OpenSearch, analytics aggregations)
- **Event Sourcing:** Reader choice history reconstructed from `reader.choice_made` event sequence
- **Database per Service:** No shared databases between microservices
- **API Gateway:** Single entry point handling auth, routing, and rate limiting

### Event Flow

Primary services (Auth, Story, Reader) **emit** events. Secondary services (Analytics, Notification, Search) **consume** events asynchronously via the event bus. Key events: `story.published`, `reader.choice_made`, `reader.story_completed`, `user.registered`.

### Main API Routes

- `/auth/*` — Auth Service
- `/stories/*` — Story Service
- `/stories/:id/nodes/*`, `/stories/:id/graph`, `/stories/:id/validate` — Graph Service
- `/read/:storyId/*` — Reader Service
- `/search` — Search Service
