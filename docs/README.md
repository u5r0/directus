# Directus Architecture Documentation

This directory contains comprehensive documentation of the Directus project architecture, organized by component and functionality.

## Documentation Structure

### Core Architecture
- [**Project Overview**](./01-project-overview.md) ✅ - High-level project structure and purpose
- [**Monorepo Structure**](./02-monorepo-structure.md) ✅ - Workspace organization and shared packages
- [**Build & Deployment**](./03-build-deployment.md) ✅ - Build configuration and deployment setup
- [**Package Interactions**](./20-package-interactions.md) ✅ - **How /api, /app, /packages, and /sdk interact**

### Backend (API)
- [**Backend Architecture**](./04-backend-architecture.md) ✅ - Express.js API structure and patterns
- [**Database Layer**](./05-database-layer.md) ✅ - Database schema, models, and data flow
- [**Authentication & Security**](./06-auth-security.md) ✅ - Authentication, authorization, and security features

### Frontend (App)
- [**Frontend Overview**](./07-frontend-overview.md) ✅ - Vue 3 application structure and architecture
- [**UI Component System**](./08-ui-components.md) ✅ - Complete component library and interactions
- [**Interface System**](./09-interface-system.md) ✅ - Input interfaces and field types
- [**Display System**](./10-display-system.md) ✅ - Output displays and data visualization
- [**Layout System**](./11-layout-system.md) ✅ - Content layout and presentation modes
- [**State Management**](./12-state-management.md) ✅ - Pinia stores and data flow
- [**API Integration**](./13-api-integration.md) ✅ - Frontend-backend communication patterns

### SDK & Extensions
- [**SDK Architecture**](./14-sdk-architecture.md) ✅ - TypeScript SDK and client integration
- [**Extension System**](./15-extension-system.md) ✅ - Plugin architecture and extensibility

### Advanced Features
- [**Real-time Features**](./16-realtime-features.md) ✅ - WebSocket integration and live updates
- [**AI Integration**](./17-ai-integration.md) ✅ - AI features and Model Context Protocol
- [**Workflow Engine**](./18-workflow-engine.md) ✅ - Flows, operations, and automation
- [**Performance & Scaling**](./19-performance-scaling.md) ✅ - Optimization and scalability features

## Quick Navigation

### For Developers
- **New to Directus?** Start with [Project Overview](./01-project-overview.md)
- **Understanding the Architecture?** Read [Package Interactions](./20-package-interactions.md) - **Essential reading!**
- **Frontend Development?** See [Frontend Overview](./07-frontend-overview.md) and [UI Components](./08-ui-components.md)
- **Backend Development?** See [Backend Architecture](./04-backend-architecture.md) and [Database Layer](./05-database-layer.md)
- **Building Extensions?** Check [Extension System](./15-extension-system.md)
- **Using the SDK?** Review [Package Interactions](./20-package-interactions.md) for SDK usage patterns

### For Architects
- **System Design?** Review [Monorepo Structure](./02-monorepo-structure.md) and [Performance & Scaling](./19-performance-scaling.md)
- **Security Review?** See [Authentication & Security](./06-auth-security.md)
- **Integration Planning?** Check [SDK Architecture](./14-sdk-architecture.md) and [API Integration](./13-api-integration.md)

## Contributing to Documentation

When updating this documentation:
1. Keep each file focused on a single topic
2. Use consistent formatting and structure
3. Include code examples where relevant
4. Update cross-references when adding new content
5. Maintain the table of contents in each file

## Documentation Standards

- **File Naming**: Use numbered prefixes for logical ordering
- **Headers**: Use consistent header hierarchy (H1 for main topics, H2 for sections)
- **Code Blocks**: Include language specification for syntax highlighting
- **Cross-References**: Use relative links between documentation files
- **Examples**: Provide practical code examples and use cases
