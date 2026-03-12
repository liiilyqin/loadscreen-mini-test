# Implementation Notes — Employee Directory (Tasks 1–5)

---

## Overview

This document describes the design and implementation decisions behind Tasks 1–5 of the Employee Directory assignment. The focus is on architecture, data flow, and engineering rationale rather than line-by-line code explanation.

All five tasks are additive — no existing routes, schema, or frontend logic were modified. Every new capability layers on top of the existing read-only directory.

---

## System Architecture

```
Browser (index.html)
  │
  │  HTTP (JSON)
  ▼
Express.js API (server.js)
  │
  │  SQL (better-sqlite3)
  ▼
SQLite Database (employees.db)
```

**Frontend** (`public/index.html`) is a single-page vanilla JS application. It owns all rendering, state (sort column, current page, cached employee list), and user interaction. There is no client-side routing framework.

**Backend** (`server.js`) is a thin REST API. It handles validation, SQL execution, and error normalisation. It has no templating or session state — every response is JSON.

**Database** (`employees.db`) is a single SQLite file. It enforces the `UNIQUE` constraint on `email` and is the authoritative source for all employee data.

**High-level data flow:**
```
User action
  → fetch() call from frontend
    → Express route validates + executes SQL
      → JSON response
        → Frontend re-renders affected components
```

---

## Tasks 1 & 2 — Add and Edit Employee

### Design: Single Form for Two Operations

Rather than maintaining two separate form panels (one for add, one for edit), both operations share a single `#employeeFormPanel`. The mode is determined by a hidden `fEditId` field:

| `fEditId` | Mode | HTTP Method | Endpoint |
|---|---|---|---|
| `""` (empty) | Add | `POST` | `/api/employees` |
| `"<id>"` | Edit | `PUT` | `/api/employees/:id` |

This design reduces duplicated HTML, ensures consistent field validation, and means future field changes only need to be made in one place.

### Backend Data Flow

```
POST /api/employees
  → validate 7 required fields (400 if missing)
  → INSERT INTO employees
    → UNIQUE email conflict  → 409 Conflict
    → success               → 201 Created  { id }

PUT /api/employees/:id
  → validate 7 required fields (400 if missing)
  → UPDATE employees WHERE id = ?
    → no row affected        → 404 Not Found
    → UNIQUE email conflict  → 409 Conflict
    → success               → 200 OK
```

Both routes share identical field validation logic. Server-side validation is authoritative — the API is safe to call directly, not just from the browser.

### Frontend Behaviour

- Opening the add form calls `showAddForm()`, which clears `fEditId` and resets the panel title and button label.
- Opening the edit form calls `openEditForm(id)`, which reads from the in-memory `allEmployees` array — no extra network request is needed to pre-fill fields.
- On success, both paths call `hideEmployeeForm()` then `loadEmployees()`, which is the single source of truth for table state and stat cards.

### Department as `<select>`, Not Free Text

Department name is used for filtering, badge colouring, and salary grouping (Task 5). Free-text input risks inconsistencies (`"Engineering"` vs `"engineering dept"`). A `<select>` populated live from `GET /api/departments` enforces referential consistency without requiring a separate departments table.

---

## Task 3 — Delete Employee

### Backend Data Flow

```
DELETE /api/employees/:id
  → DELETE FROM employees WHERE id = ?
    → no row affected  → 404 Not Found
    → success         → 200 OK
```

### Two-Stage Confirmation

Delete uses a custom confirmation modal rather than `window.confirm()` for three reasons:

1. **Identity clarity** — the modal displays the employee's name (`"Are you sure you want to delete Jane Doe?"`), making accidental deletions less likely.
2. **Styling consistency** — the modal reuses existing card and button classes; `window.confirm()` is unstyled and browser-dependent.
3. **Non-blocking** — `window.confirm()` is synchronous and freezes the main thread. The custom modal is async-safe.

The confirm button's `onclick` handler is reassigned at open time via closure, capturing the target `id` without storing it in the DOM.

---

## Task 4 — Pagination

### Approach: Client-Side Slice

Pagination is implemented entirely on the frontend using `Array.slice()`. The server returns the full filtered and sorted result set as before; the client divides it into pages of 5.

**Why client-side, not server-side?**

Server-side pagination would require adding `limit` and `offset` query parameters to the API, coordinating them with existing `search`, `department`, and `sort` parameters, and resetting the page number on every filter change. For the dataset sizes relevant to this application, the added architectural complexity brings no perceptible performance benefit. Client-side slicing achieves the same user experience in ~15 lines.

**Trade-offs acknowledged:**
- Not suitable for very large datasets (thousands of rows) where returning all records per request would be costly.
- A server-side implementation would be the correct approach at scale (noted in Possible Improvements).

### Data Flow

```
Any filter / search / sort change
  → loadEmployees() fetches full filtered+sorted list
    → currentPage resets to 1
      → renderTable() slices to current page
        → updatePagination() updates controls

Previous / Next click
  → goToPage(n) clamps n to [1, totalPages]
    → renderTable() re-slices — no network call
      → updatePagination() updates controls
```

### Key Decisions

- `currentPage` resets inside `loadEmployees()`, not in each event listener. Any future data-changing trigger automatically resets pagination without additional wiring.
- Page clamping happens inside `renderTable()`, not in `goToPage()`. This handles the case where a search reduces the result count while the user is on a later page — `renderTable` is the only place this out-of-bounds condition can arise from an external cause.
- The pagination bar hides entirely when `totalPages <= 1` to avoid unnecessary chrome on small datasets.

---

## Task 5 — Department Salary Pie Chart

### Backend: `GET /api/salary-by-department`

```
SELECT department, SUM(salary) AS total
FROM employees
GROUP BY department
ORDER BY department
```

Aggregation is performed in SQL rather than in JavaScript for two reasons: the database can compute the aggregate in a single pass, and keeping aggregation logic in the query layer means the frontend receives clean, ready-to-render data with no post-processing.

The endpoint returns the global aggregate regardless of the current table filter — the chart is intended as an organisation-wide overview, not a filtered view.

### Frontend Design

**Separation of concerns across three functions:**

| Function | Responsibility |
|---|---|
| `loadSalaryChart()` | Fetch data, filter invalid rows, compute `grandTotal` and `colors` once, delegate to renderers |
| `drawPieChart(canvas, data, colors, grandTotal)` | Canvas rendering only — no data logic |
| `renderChartLegend(data, colors, grandTotal)` | HTML legend only — no canvas operations |

`grandTotal` and `colors` are computed once in `loadSalaryChart()` and passed down. This guarantees the pie slices and legend dots always use identical values — if each function recomputed them independently, any execution-order difference could cause a mismatch.

**Why Canvas, not a chart library?**
Task requirement. Canvas also keeps the dependency footprint at zero and produces a single DOM element with no children, making the chart easy to reason about.

**Why render the legend in HTML, not on the Canvas?**
Canvas text does not scale cleanly across DPI levels, is not selectable, and is harder to maintain. An HTML legend (`<div>` per row) inherits the page's font rendering, is accessible to screen readers, and can be updated by simply setting `innerHTML`.

**Color strategy:**
Chart slice colors are derived from the existing `DEPT_COLORS` map (the same accent colors used by table badges), so Engineering is always `#1565c0` and Marketing always `#c2185b` across both UI elements. For departments not in `DEPT_COLORS`, a fallback palette cycles via `index % palette.length` — the system never crashes on unexpected department names.

**Refresh triggers:**
`loadSalaryChart()` is called at page init and after any mutation (add, edit, delete). It is deliberately *not* called on search or filter changes — those operations do not mutate salary data, so the chart is unchanged and the network call would be wasted.

---

## Edge Cases & Robustness

| Scenario | Handling |
|---|---|
| Duplicate email on add or edit | SQLite `UNIQUE` constraint caught explicitly → `409 Conflict` with message surfaced inline in the form |
| Missing required fields | Server returns `400` with field list; HTML `required` attributes provide instant client-side feedback before submission |
| Delete a non-existent employee | `changes === 0` after `DELETE` → `404 Not Found` |
| Edit a non-existent employee | `changes === 0` after `UPDATE` → `404 Not Found` |
| Search returns zero results | Empty-state illustration rendered; pagination bar hidden |
| Filter reduces results while on page 3 | `currentPage` clamped down inside `renderTable()` on every render |
| Chart with no employees / all-zero salaries | `drawPieChart` renders "No salary data available" text and returns early |
| Invalid or non-finite salary values in chart data | Filtered out in `loadSalaryChart()` before any rendering |
| Unknown department in chart | Color resolved from fallback palette via `fi++ % palette.length` — no crash |

---

## Possible Improvements

Given more time, the following improvements would be worthwhile:

- **Server-side pagination** — `LIMIT` / `OFFSET` in SQL with page parameters on the API, necessary for large datasets.
- **Normalised departments table** — a `departments` table with a foreign key on `employees.department_id` would enforce referential integrity at the database level rather than relying on the UI `<select>`.
- **Authentication / RBAC** — currently all operations are unauthenticated. Role-based access control (e.g. read-only vs admin) would be required for a production deployment.
- **Chart interactivity** — hover tooltips on pie slices (canvas `mousemove` hit-testing) and click-to-filter integration with the table.
- **Optimistic UI updates** — for add/edit/delete, the row could be updated locally before the server confirms, with a rollback on failure, to reduce perceived latency.
- **Input sanitisation** — currently trusting `trim()` on text fields; a validation library (e.g. Zod on the server) would provide stronger type guarantees.
