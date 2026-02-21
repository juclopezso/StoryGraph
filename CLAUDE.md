# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

StoryGraph is a web platform for creating and reading non-linear interactive stories. Readers navigate through story "nodes" (text fragments) connected via choices, creating unique paths through graph-structured narratives. This is a **course project currently in the design/architecture phase** — no source code has been implemented yet.

All documentation is written in **Spanish**.

## Planned Technology Stack

- **Frontend:** Two SPAs (UI Usuario + UI Administración) built with React.js or Vue.js, served via S3 + CloudFront
- **Backend:** Node.js + Express or Python + FastAPI
- **Databases:** PostgreSQL (auth, users, story metadata, reading progress, notifications), MongoDB (stories and narrative nodes), Redis (reading cache)
- **Search:** OpenSearch / Elasticsearch
- **Message Broker:** Apache Kafka / RabbitMQ / Amazon SNS+SQS (CloudEvents format)
- **Auth:** JWT tokens
- **Graph Visualization:** D3.js or Cytoscape.js
- **Deployment:** AWS (ECS Fargate or EKS), API Gateway, CloudFront

## Architecture

The system follows a **microservices + event-driven** architecture with 10 components:

### Frontends

| Component | Responsibility |
|-----------|---------------|
| **UI Usuario** | SPA for end users: login, story browsing, reading, creation |
| **UI Administración** | SPA for admins: user management, roles, platform config |

### Infrastructure

| Component | Responsibility |
|-----------|---------------|
| **API Gateway** | Single entry point: routing, JWT auth, rate limiting, orchestration |
| **Message Broker** | Async messaging from Story/Reading services to Notification/Search |

### Microservices

| Service | Responsibility | Database |
|---------|---------------|----------|
| **Auth Service** | Registration, login, JWT, roles, permissions | PostgreSQL |
| **User Service** | User profiles, demographics, profile photo | PostgreSQL + S3 |
| **Story Service** | Story CRUD, classifications, narrative nodes, graph validation | PostgreSQL + MongoDB |
| **Reading Service** | Reading interaction, progress, history, completed stories | PostgreSQL + Redis |
| **Notification Service** | Notifications to readers/writers on publish, completion, etc. | PostgreSQL |
| **Search Service** | Full-text story indexing and search | OpenSearch |

### Key Patterns

- **CQRS:** Write models (PostgreSQL/MongoDB) separated from read-optimized stores (OpenSearch)
- **Event Sourcing:** Reader choice history reconstructed from `reader.choice_made` event sequence
- **Database per Service:** No shared databases between microservices
- **API Gateway:** Single entry point handling auth, routing, rate limiting, and orchestration

### Event Flow

Story Service and Reading Service **emit** events to the Message Broker. Notification Service and Search Service **consume** events asynchronously. Key events: `story.published`, `story.updated`, `story.deleted`, `reader.choice_made`, `reader.story_completed`, `user.registered`.

### Main API Routes

- `/auth/*` — Auth Service
- `/users/*` — User Service
- `/stories/*` — Story Service (including nodes, graph, validation)
- `/read/*` — Reading Service
- `/search/*` — Search Service
- `/admin/*` — User Service / Auth Service
