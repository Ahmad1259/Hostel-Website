# Entity Relationship Diagram

Status: **Phase 1 draft.** Generated from `database-schema.md`. Diagrams are
Mermaid so they render on GitHub and can be regenerated when the schema changes.
Keep these two files in sync - the schema text is authoritative.

## Core relationships

```mermaid
erDiagram
    USERS ||--o{ USER_ROLES : "granted"
    ROLES ||--o{ USER_ROLES : ""
    HOSTELS ||--o{ USER_ROLES : "scoped to"
    HOSTELS ||--o{ HOSTEL_STAFF_ASSIGNMENTS : "staffed by"
    USERS ||--o{ HOSTEL_STAFF_ASSIGNMENTS : ""
    USERS ||--|| STUDENTS : "is a"
    STUDENTS ||--o{ ADMISSION_APPLICATIONS : "submits"
    HOSTELS ||--o{ ADMISSION_APPLICATIONS : "targeted at"
    ADMISSION_APPLICATIONS ||--o{ APPLICATION_DOCUMENTS : "contains"
    HOSTELS ||--o{ ROOMS : "has"
    ROOM_TYPES ||--o{ ROOMS : "typed as"
    ROOMS ||--o{ BEDS : "contains"
    HOSTELS ||--o{ FACILITIES : "offers"
    ROOMS ||--o{ ROOM_MEDIA : "shown by"
    STUDENTS ||--o{ BED_ASSIGNMENTS : "holds"
    BEDS ||--o{ BED_ASSIGNMENTS : "occupied via"
    ADMISSION_APPLICATIONS ||--o| BED_ASSIGNMENTS : "may lead to"
    BED_ASSIGNMENTS ||--o{ STAY_EVENTS : "check-in/out"
    BED_ASSIGNMENTS ||--o{ ROOM_TRANSFER_REQUESTS : "moved from/to"
    BED_ASSIGNMENTS ||--o{ CHALLANS : "billed by"
    CHALLANS ||--o{ PAYMENT_SUBMISSIONS : "paid via"
    PAYMENT_SUBMISSIONS ||--o| RECEIPTS : "receipted by"
    BED_ASSIGNMENTS ||--o{ DEPOSITS : "secures"
    STUDENTS ||--o{ NOTIFICATION_PREFERENCES : "opts into"
    NOTIFICATION_EVENTS ||--o{ NOTIFICATION_DELIVERIES : "attempted by"
    STUDENTS ||--o{ NOTIFICATION_EVENTS : "receives"
```

## Occupancy lifecycle (state, not just tables)

```mermaid
stateDiagram-v2
    [*] --> Available
    Available --> Reserved: warden assigns bed to approved application
    Reserved --> Occupied: check-in recorded
    Available --> Maintenance: flagged by staff
    Maintenance --> Available: marked ready
    Occupied --> Available: check-out + room ready
    Occupied --> Maintenance: check-out + room not ready
    Reserved --> Cancelled_or_Available: assignment cancelled
    Occupied --> Reserved: transfer approved (old ends, new starts)
```

## Money flow (verification is the gate)

```mermaid
flowchart LR
    A[Challan issued] --> B[Student uploads proof: submission = pending]
    A --> C[Staff records cash payment]
    B --> D{Staff verifies?}
    D -- rejected --> E[Notify student: balance unchanged]
    D -- verified --> F[Update challan paid amount in one transaction]
    C --> F
    F --> G[Issue unique receipt]
    G --> H[Balance = amount - paid]
    H --> I{Balance zero?}
    I -- yes --> J[Mark challan paid, stop reminders]
    I -- no --> K[Reminders continue per due-date policy]
```

## Reading notes for a non-programmer

- "One student can have only one active bed" is drawn as a rule on
  `BED_ASSIGNMENTS`, not on `STUDENTS` - the history of previous beds is kept.
- `PAYMENT_SUBMISSIONS` (what someone claims) and `RECEIPTS` (what the platform
  confirms) are deliberately separate tables. This is what makes "screenshot
  uploaded = paid" impossible by design.
- The public search page reads `availability_snapshot`, but that snapshot is
  rebuilt from real `BEDS` + `BED_ASSIGNMENTS` data every time they change, so
  it can never show an invented number.
