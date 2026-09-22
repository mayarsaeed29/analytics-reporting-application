# Technical Architecture

## Overview

This document describes the technical architecture of the Analytics & Reporting Application.

The application was designed to provide controlled access to a large analytical dataset through a desktop interface without requiring users to work directly with raw files or PostgreSQL.

The system follows a layered architecture:

```text
Desktop Application
        ↓
FastAPI
        ↓
Analytics Service
        ↓
Repository Layer
        ↓
PostgreSQL
```

Each layer has a clearly defined responsibility.

---

## Architecture Goals

The application was designed to:

- Support large analytical datasets
- Keep database access controlled
- Separate UI logic from database logic
- Provide flexible filtering
- Support efficient pagination
- Maintain stable query behavior
- Keep the desktop interface responsive and usable
- Allow the analytical database to evolve independently from the UI

---

## Desktop Layer

The desktop interface is built using **Python and PySide / Qt**.

Its responsibilities include:

- Displaying filters
- Collecting user input
- Sending analytical requests
- Displaying tabular results
- Managing pagination
- Preserving filter state
- Handling interface interactions

The desktop layer does not contain direct PostgreSQL access.

---

## API Layer

**FastAPI** acts as the communication boundary between the desktop application and the analytical backend.

The API handles:

- Request validation
- Filter parameters
- Pagination state
- Response formatting
- Error handling
- Controlled analytical endpoints

The API prevents database implementation details from being exposed directly to the user interface.

---

## Analytics Service

The analytics service contains application-level analytical logic.

Its responsibilities include:

- Interpreting filter requests
- Coordinating query execution
- Applying query rules
- Managing pagination behavior
- Validating dataset state
- Returning structured analytical results

This layer keeps business and analytical logic separate from both the UI and the database adapter.

---

## Repository Layer

The repository layer is responsible for database interaction.

It provides controlled methods for:

- Query execution
- Filter-option retrieval
- Result pagination
- Dataset-state checks
- Analytical record access

This prevents SQL queries from being scattered throughout the application.

---

## PostgreSQL Layer

PostgreSQL acts as the analytical data store.

The application accesses an approved reporting dataset through a dedicated read-only service account.

This provides:

- Centralized analytical storage
- SQL-based filtering
- Efficient querying
- Controlled schema access
- Integration with reporting systems
- Separation from raw source files

---

## Request Flow

A typical user search follows this path:

```text
User selects filters
        ↓
Desktop application builds request
        ↓
FastAPI validates request
        ↓
Analytics service processes query rules
        ↓
Repository builds database query
        ↓
PostgreSQL returns records
        ↓
Results return through API
        ↓
Desktop displays results
```

---

## Filter Architecture

Filters are treated as optional query parameters.

The application supports combinations of:

- Date filters
- Dropdown filters
- Text filters
- Identifier filters
- Dependent filters

Only filters selected by the user are applied.

This allows both broad and highly targeted searches.

---

## Database-Backed Filter Options

Dropdown values are retrieved from the analytical dataset rather than being permanently hard-coded.

This helps ensure that:

- Available values reflect real data
- New values can appear without redesigning the UI
- Invalid options are reduced
- Filter behavior remains consistent with the database

---

## Dependent Filters

Some filters depend on another selected value.

Conceptually:

```text
Parent Filter
      ↓
Selected Value
      ↓
Query Available Child Values
      ↓
Update Child Dropdown
```

This improves usability by reducing irrelevant filter options.

---

## Keyset Pagination

The application uses keyset pagination for large result sets.

Instead of using increasingly expensive offsets:

```text
OFFSET 1000000
```

the backend continues from the last known record.

Conceptually:

```text
Page 1
↓
Last Record Cursor
↓
Page 2
↓
New Cursor
↓
Page 3
```

This provides more predictable performance when browsing millions of records.

---

## Query Consistency

The analytical dataset can change while a user is browsing paginated results.

To avoid returning inconsistent pages, the backend maintains dataset-state information.

If the dataset has changed in a way that invalidates the current paging state, the request can be rejected and refreshed rather than silently returning inconsistent results.

---

## Read-Only Access

The application uses a read-only analytical database role.

This ensures that normal application usage cannot modify production analytical records.

The application is designed for:

```text
Search
Filter
Explore
Report
```

rather than:

```text
Insert
Update
Delete
```

---

## UI State Management

The interface preserves user filter selections during common UI actions.

For example:

```text
Configure filters
        ↓
Run search
        ↓
Collapse filter panel
        ↓
Explore results
        ↓
Reopen filters
        ↓
Selections remain available
```

This improves usability when users repeatedly refine large analytical searches.

---

## Scrollable Filter Design

A large number of filters can exceed the available vertical window space.

The filter body therefore uses a dedicated scrollable area.

This prevents controls from shrinking or becoming inaccessible while keeping the results area independent.

---

## Separation of Concerns

The architecture intentionally separates:

### UI

Responsible for presentation and user interaction.

### API

Responsible for communication and request validation.

### Service

Responsible for analytical rules and application logic.

### Repository

Responsible for database access.

### Database

Responsible for analytical storage and query execution.

This separation makes the system easier to maintain, test, and extend.

---

## Confidentiality

This document describes the architectural approach without exposing:

- Production source code
- Database credentials
- Internal URLs
- Operational datasets
- Proprietary business rules
- Customer or passenger information
- Internal infrastructure details

The purpose of this case study is to demonstrate the design and technical decisions behind the application while preserving confidentiality.
