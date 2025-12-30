# Project Overview

s Directus?

**Directus** is a real-time API and App dashboard for managing SQL database content. It's a sophisticated, enterprise-grade headless CMS built as a **monorepo** using modern technologies including Vue 3, Express.js, and TypeScript.

## Key Characteristics

- **License**: Business Source License (BSL) 1.1 with additional use grant
- **Architecture**: Monorepo with pnpm workspaces
- **Node Version**: 22 (required)
- **Package Manager**: pnpm >= 10 < 11
- **Primary Purpose**: Headless CMS/Content Management Platform with REST/GraphQL APIs

## Core Features

### API Layer
- **REST & GraphQL API** - Instantly layers a blazingly fast Node.js API on top of any SQL database
- **Multi-Database Support** - Works with PostgreSQL, MySQL, SQLite, OracleDB, CockroachDB, MariaDB, and MS-SQL
- **Real-time Capabilities** - WebSocket support for live updates and collaboration
- **Type-Safe SDK** - Composable TypeScript client for seamless integration

### Admin Interface
- **Modern Dashboard** - Vue.js app that's safe and intuitive for non-technical users
- **No-Code Interface** - Visual content management without requiring technical training
- **Extensible Platform** - Built to white-label with modular, customizable components
- **Multi-Language Support** - Full internationalization with 40+ languages

### Enterprise Features
- **Multiple Authentication Providers** - Local, SAML, OpenID Connect, LDAP
- **Role-Based Access Control** - Fine-grained permissions at field and collection level
- **Content Versioning** - Track changes and manage content versions
- **Workflow Automation** - Visual workflow builder for business process automation
- **AI Integration** - Built-in AI capabilities with Anthropic and OpenAI support

## Architecture Highlights

### Monorepo Structure
```
directus/
├── directus/           # Main CLI and deployment package
├── app/               # Vue 3 admin dashboard
├── api/               # Express.js backend API
├── sdk/               # JavaScript/TypeScript SDK
├── packages/          # 30+ shared utility packages
└── tests/             # Comprehensive test suites
```

### Technology Stack
- **Frontend**: Vue 3 + TypeScript + Vite + Pinia
- **Backend**: Express.js + Knex + TypeScript
- **Database**: Multi-database support with schema abstraction
- **Real-time**: WebSocket integration for live updates
- **Build**: Vite for frontend, TypeScript compilation for backend
- **Testing**: Vitest for unit tests, comprehensive integration testing

## Deployment Options

### Self-Hosted
- **On-Premises** - Full control over infrastructure and data
- **Docker** - Containerized deployment with multi-stage builds
- **Cloud Providers** - Deploy on AWS, GCP, Azure, or any cloud platform
- **Process Management** - PM2 configuration for production scaling

### Directus Cloud
- **Managed Hosting** - Fully managed service starting at $15/month
- **Auto-Scaling** - Automatic resource scaling based on usage
- **Global CDN** - Worldwide content delivery network
- **Backup & Recovery** - Automated backups and disaster recovery

## Use Cases

### Content Management
- **Headless CMS** - Decouple content management from presentation
- **Multi-Channel Publishing** - Distribute content across web, mobile, and IoT
- **Editorial Workflows** - Content approval and publishing processes
- **Asset Management** - Centralized media and document management

### Application Backend
- **API-First Development** - Build applications with instant REST/GraphQL APIs
- **Rapid Prototyping** - Quickly scaffold backend services
- **Data Management** - Visual interface for complex data relationships
- **Integration Hub** - Connect multiple systems and services

### Enterprise Solutions
- **Digital Asset Management** - Organize and distribute brand assets
- **Customer Portals** - Self-service customer interfaces
- **Internal Tools** - Admin dashboards and business applications
- **Data Analytics** - Business intelligence and reporting dashboards

## Community & Support

### Open Source Community
- **GitHub Repository** - Active development with 25k+ stars
- **Community Forum** - Discussion and support community
- **Discord Chat** - Real-time community support
- **Documentation** - Comprehensive guides and API reference

### Commercial Support
- **Enterprise Licensing** - Commercial licenses for large organizations
- **Professional Services** - Implementation and consulting services
- **Priority Support** - Dedicated support channels for enterprise customers
- **Training Programs** - Official training and certification programs

## Next Steps

- **For Developers**: Start with [Frontend Overview](./07-frontend-overview.md) or [Backend Architecture](./04-backend-architecture.md)
- **For System Architects**: Review [Monorepo Structure](./02-monorepo-structure.md)
- **For DevOps**: Check [Build & Deployment](./03-build-deployment.md)
- **For Product Managers**: Explore [Extension System](./15-extension-system.md) for customization options
#
