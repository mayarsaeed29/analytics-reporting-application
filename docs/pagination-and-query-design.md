# Pagination & Query Design

## Overview

This document describes the query and pagination strategy used by the Analytics & Reporting Application.

The application works with a large analytical dataset, so query performance and result consistency are important parts of the design.

The goal was to allow users to explore large result sets without relying on inefficient deep-offset pagination.

---

## Query Design Goals

The querying layer was designed to:

- Support many optional filters
- Keep SQL generation controlled
- Return predictable result sizes
- Support large datasets efficiently
- Preserve query consistency across pages
- Avoid expensive deep offsets
- Prevent stale pagination state from returning misleading results
- Keep filter-option queries separate from result queries

---

## Optional Filters

Most filters in the application are optional.

A request may contain only one filter, several filters, or none.

Conceptually:

```text
User Request
   ↓
Check Provided Filters
   ↓
Apply Only Active Conditions
   ↓
Execute Query
```

This allows the same analytical endpoint to support both broad searches and highly specific searches.

---

## Filter Categories

The application supports several filter types.

### Date Filters

Used for analytical date ranges such as booking or travel dates.

### Dropdown Filters

Used when values should come from the analytical dataset.

Examples include:

- Supplier
- Branch
- Status
- Airline

### Text Filters

Used for fields where users may search for a specific value.

Examples include:

- Client name
- Booking reference
- System reference

### Dependent Filters

Used when available values depend on another filter selection.

---

## Inclusive Date Filtering

Date ranges are inclusive.

Conceptually:

```text
From Date
→ include records on or after the selected date

To Date
→ include records on or before the selected date

From + To
→ include records inside the complete range
```

This behavior remains consistent across the UI, API, and SQL layer.

---

## Query Construction

The repository layer constructs analytical queries from validated request parameters.

Only approved filter fields are allowed to influence SQL generation.

This provides better control than allowing arbitrary query fragments from the client.

Conceptually:

```text
Validated Request
      ↓
Approved Filters
      ↓
Parameterized SQL
      ↓
PostgreSQL
```

Parameterized queries help keep the database layer predictable and reduce the risk of unsafe SQL construction.

---

## Result Limits

The application does not return the entire result set in one response.

Instead, results are returned in controlled batches.

For example:

```text
Request
   ↓
Return next 200 records
   ↓
Provide continuation cursor
```

This protects both the API and desktop application from unnecessarily large responses.

---

## Why Not OFFSET Pagination?

Traditional pagination often uses:

```sql
LIMIT 200 OFFSET 1000000
```

This approach can become increasingly expensive as the offset grows.

The database may still need to scan or skip a large number of rows before returning the requested page.

For large datasets, this can lead to:

- Slower response times
- Increasing query cost
- Less predictable performance

---

## Keyset Pagination

The application uses keyset pagination instead.

The next page is based on the last record from the current page.

Conceptually:

```text
First Request
     ↓
Records 1–200
     ↓
Cursor from last record
     ↓
Next Request
     ↓
Records after cursor
```

The query continues from a known position instead of repeatedly skipping earlier rows.

---

## Cursor Design

The cursor represents enough information to continue the ordered query from the previous result set.

The exact production cursor structure is intentionally omitted from this public case study.

Conceptually, it identifies:

- The current ordered position
- The dataset state associated with the query
- Enough information to continue safely

The client treats the cursor as an opaque continuation token.

---

## Stable Ordering

Keyset pagination requires a deterministic order.

The result set therefore uses an approved stable ordering rule.

A stable order helps ensure that:

- Records do not randomly change position between pages
- The continuation cursor remains meaningful
- Pagination behavior is reproducible

Where required, more than one field can participate in the ordering to ensure uniqueness.

---

## High-Watermark Concept

The application uses a high-watermark-style boundary to keep paginated results stable.

Conceptually:

```text
Search begins
      ↓
Capture current analytical boundary
      ↓
Return pages within that boundary
```

This helps prevent newly added records from unexpectedly shifting the user's active result sequence.

---

## Dataset Generation & Revision

The analytical dataset can change in different ways.

For example:

- A full controlled replacement
- An incremental revision
- New records added to the analytical layer

The backend tracks dataset state so pagination can determine whether the current cursor still belongs to a compatible version of the data.

---

## Stale Cursor Protection

A cursor created against an older incompatible dataset state should not silently continue against a different data version.

Conceptually:

```text
Client sends cursor
       ↓
Backend checks dataset state
       ↓
Compatible?
   /          \
 Yes          No
  ↓            ↓
Continue     Reject stale request
```

This protects users from browsing a logically inconsistent sequence of pages.

---

## Pagination Response

A paginated response conceptually contains:

```text
Records
Continuation Cursor
Dataset State
More Results Indicator
```

This gives the client enough information to request the next page safely.

---

## First Page

The initial search does not require an existing cursor.

The backend:

1. Validates the requested filters
2. Establishes dataset state
3. Executes the ordered query
4. Returns the first page
5. Generates a cursor when more records exist

---

## Next Page

For subsequent pages:

1. The client sends the continuation cursor
2. The backend validates the cursor
3. Dataset state is checked
4. The query continues after the last known record
5. The next result batch is returned

---

## Filter Consistency Across Pages

Pagination is tied to the original query context.

Changing filters effectively creates a new search.

For example:

```text
Search:
Supplier = A
Status = Confirmed
        ↓
Cursor belongs to this search

User changes:
Supplier = B
        ↓
New search
        ↓
New pagination sequence
```

This avoids mixing pages from different filter conditions.

---

## Filter Option Queries

Dropdown option queries are kept separate from the main record query.

For example:

```text
Get suppliers
Get branches
Get statuses
Get airlines
```

These queries return distinct approved values rather than complete analytical records.

This provides a cleaner boundary between:

- Filter metadata
- Analytical result retrieval

---

## Dependent Option Queries

Some option queries depend on another filter.

Example:

```text
Selected Supplier
       ↓
Request available related values
       ↓
Repository queries approved dataset
       ↓
Return filtered options
```

This allows dependent dropdowns to reflect the real analytical relationships in the data.

---

## Searchable Dropdowns

Large option sets can become difficult to use.

Searchable dropdowns allow users to locate values more quickly while still selecting from approved database-backed options.

This is preferable to free-text entry when valid values should remain constrained.

---

## Query Safety

The querying layer follows several safety principles:

- Only approved fields can be filtered
- Query values are parameterized
- Database access is read-only
- Result sizes are controlled
- Cursor state is validated
- SQL is isolated from the UI
- Arbitrary client SQL is not accepted

---

## Performance Considerations

The query design focuses on reducing unnecessary work.

Key strategies include:

- Keyset pagination
- Controlled page sizes
- Stable ordering
- Database-backed filter options
- Selective query conditions
- Dedicated analytical storage
- Avoiding direct processing of raw files during user searches

---

## Why PostgreSQL?

PostgreSQL provides a suitable analytical foundation for the application because it supports:

- Structured SQL querying
- Indexing
- Large datasets
- Controlled permissions
- Stable ordering
- Parameterized queries
- Integration with Python
- Read-only roles

The desktop application therefore queries an analytical database rather than repeatedly scanning raw operational files.

---

## User Experience

The query architecture also affects the desktop experience.

Users can:

- Configure filters
- Run targeted searches
- Browse results page by page
- Continue through large result sets
- Adjust filters and start a new search
- Use searchable database-backed dropdowns

The complexity of the underlying query logic remains hidden behind the interface.

---

## Design Principles

### Query Only What Is Needed

Only active filters are applied.

### Keep Result Sizes Controlled

Large result sets are paginated instead of returned in one response.

### Use Deterministic Ordering

Stable ordering is required for reliable keyset pagination.

### Protect Pagination State

Cursors should not continue silently against incompatible dataset versions.

### Separate Filter Metadata From Record Retrieval

Dropdown-option queries and analytical result queries have different responsibilities.

### Keep SQL Behind the Repository Layer

The desktop application and API should not contain scattered database-query logic.

---

## Confidentiality

This document explains the pagination and query architecture without exposing:

- Production SQL
- Internal database schemas
- Production cursor encoding
- Confidential identifiers
- Operational datasets
- Database credentials
- Internal infrastructure details

The goal is to demonstrate the technical design while preserving production confidentiality.
