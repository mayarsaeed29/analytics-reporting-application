# Architecture Diagrams

This folder contains visual documentation for the Analytics & Reporting Application.

The diagrams are simplified portfolio representations of the production architecture and are intended to explain the system design without exposing confidential implementation details.

## Planned Diagrams

### Application Architecture

Shows the main system layers:

Desktop Application → FastAPI → Analytics Service → Repository Layer → PostgreSQL

### Query & Pagination Flow

Shows how a user search moves through the application and how paginated results are returned using continuation cursors.

### Filter Interaction Flow

Shows how database-backed and dependent filters interact with the analytical backend.

## Confidentiality

The diagrams intentionally exclude:

- Production infrastructure details
- Internal URLs
- Credentials
- Database connection information
- Production schemas
- Confidential business logic
- Operational data

[View Application Architecture](application-architecture.png)
[View Query & Pagination Flow](query-pagination-flow.png)
