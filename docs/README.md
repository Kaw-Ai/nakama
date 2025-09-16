# Nakama Technical Documentation

This directory contains comprehensive technical architecture documentation for Nakama, the distributed server for social and realtime games and apps.

## Documentation Structure

- [**Architecture Overview**](architecture/README.md) - High-level system architecture and design principles
- [**Component Architecture**](architecture/components.md) - Detailed component breakdown and interactions
- [**Database Architecture**](architecture/database.md) - Database design, schema, and data flow
- [**Authentication & Authorization**](architecture/auth.md) - Security architecture and flow diagrams
- [**Real-time Communication**](architecture/realtime.md) - WebSocket, matchmaking, and real-time features
- [**Runtime Extensions**](architecture/runtime.md) - Go, Lua, and JavaScript runtime architecture
- [**Deployment Architecture**](architecture/deployment.md) - Infrastructure and deployment patterns
- [**API Architecture**](architecture/api.md) - REST, gRPC, and protocol architecture

## Getting Started

If you're new to Nakama's architecture, start with the [Architecture Overview](architecture/README.md) to understand the high-level design, then dive into specific components based on your interests.

## Diagrams

All architecture diagrams are created using [Mermaid](https://mermaid.js.org/), which renders directly in GitHub and most modern documentation platforms. The diagrams are embedded in the markdown files and can be edited directly.

## Contributing

When updating architecture documentation:

1. Keep diagrams up-to-date with code changes
2. Use consistent terminology across all documents
3. Include both high-level overviews and detailed technical information
4. Verify Mermaid diagrams render correctly in GitHub
5. Cross-reference related components and flows

For questions about the architecture or to propose improvements, please create an issue or discussion in the repository.