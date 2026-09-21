# Car Rental — Domain Specification

A sample domain for building a RAP (ABAP RESTful Application Programming Model)
application: manage a fleet of cars across multiple locations and track rentals
from pickup to drop-off.

## Overview

- The company operates **multiple locations**, each with a **car park**.
- A **rental** links a car and a customer for a period of time, with a pickup and drop-off location.
- A rental moves through a **lifecycle** (Created → Active → Completed, or
  Cancelled) driven by actions, not by editing a status field.
- A car has an immutable **home location** (where it enters service), but its
  **current location** and **availability** are not stored — they are derived
  from its rentals.
- All amounts are in a **single currency (EUR)** — no currency handling, to keep
  the exercise focused.

The application is built in two clearly separated layers: a **RAP backend
exposed as a plain OData V4 API** with no UI annotations, and a **freestyle
SAPUI5 frontend** built separately in BAS. See [Frontend / UI](#frontend--ui).

## Entities

### Location

| Field           | Type    | Notes                   |
| --------------- | ------- | ----------------------- |
| ID              | UUID    | Primary key             |
| name            | String  | Display name            |
| address         | String  | Street / city           |
| carParkCapacity | Integer | Number of parking spots |

### Car

| Field        | Type                   | Notes                                    |
| ------------ | ---------------------- | ---------------------------------------- |
| ID           | UUID                   | Primary key                              |
| licensePlate | String                 | Unique                                   |
| model        | String                 | Make + model                             |
| dailyRate    | Decimal                | Price per calendar day (EUR)             |
| homeLocation | Association → Location | Where the car enters service (immutable) |

### Customer

| Field         | Type   | Notes            |
| ------------- | ------ | ---------------- |
| ID            | UUID   | Primary key      |
| name          | String | Full name        |
| email         | String | Contact          |
| licenseNumber | String | Driver's license |

### Rental

| Field           | Type                   | Notes                                     |
| --------------- | ---------------------- | ----------------------------------------- |
| ID              | UUID                   | Primary key                               |
| car             | Association → Car      | Rented car                                |
| customer        | Association → Customer | Who rented it                             |
| startDate       | Date                   | Rental start (first charged day)          |
| endDate         | Date                   | Rental end (last charged day)             |
| pickupLocation  | Association → Location | Where the car is picked up                |
| dropoffLocation | Association → Location | Where the car is returned                 |
| status          | Enum                   | `Created` / `Active` / `Completed` / `Cancelled` — read-only on the UI, changed only via actions |
| totalPrice      | Decimal                | Calculated — see [Determinations](#determinations) |

## Relationships

- A **Location** is referenced by rentals as pickup and drop-off, and is a car's `homeLocation`.
- A **Car** has many **Rentals** over time; its availability and whereabouts are
  derived from those rentals rather than stored on the car.
- A **Customer** has many **Rentals**.
- A **Rental** references exactly one **Car**, one **Customer**, and two **Locations** (pickup + drop-off).

## Rental Lifecycle

Rentals are **created and edited via standard CRUD**, but **state
transitions happen only through actions**. The `status` field is read-only on
the UI (field feature control); the actions are the only way it changes.

```
            pickUp            returnCar
 Created ──────────► Active ──────────► Completed
    │
    │ cancel
    ▼
 Cancelled
```

| Action      | Allowed in status | Effect                                            |
| ----------- | ----------------- | ------------------------------------------------- |
| `pickUp`    | Created           | Car handed over to the customer → `Active`        |
| `returnCar` | Active            | Car returned at the drop-off location → `Completed` |
| `cancel`    | Created           | Rental never happens → `Cancelled`                |

- A new rental always starts in status `Created`.
- Actions are **enabled/disabled dynamically** based on the current status
  (RAP dynamic feature control) — e.g. `returnCar` is only enabled on an
  `Active` rental.
- `Cancelled` rentals are ignored by all constraints, derivations, and KPIs.
- These actions are exposed on the OData service; students wire them up as
  buttons in their own frontend — see [Frontend / UI](#frontend--ui).

> **Design note:** we deliberately do *not* model creation/editing as actions.
> Plain CRUD lets the validations do the guarding and keeps the service simple;
> actions are reserved for transitions where a plain field edit would be
> semantically wrong — you do not "become" picked up by typing a status.

## Business Rules & Constraints

The application must enforce a few obvious real-world constraints. In RAP these
are **validations** (and supporting **determinations**) on the `Rental` entity.

### Derived current location

A car's **current location** is:

> the drop-off location of its **completed** rental with the latest `endDate`,
> or its `homeLocation` if it has no completed rentals.

`homeLocation` is master data set once at onboarding — it is not the same as
"current location" and never changes operationally.

### Constraints

Constraints apply to all rentals except `Cancelled` ones.

1. **Pickup must match the car's current location.** A car can only be rented
   from location A if it is actually there. Reject a rental whose
   `pickupLocation` ≠ the car's current location.
2. **Drop-off must have parking space.** A car can only be returned to location
   B if that car park has a free spot. Count the cars whose
   [derived current location](#derived-current-location) is B, and reject the
   rental if that count is already at B's `carParkCapacity`.

   The check is against **current** occupancy — where the cars are now, not
   where they will be on the drop-off date.
3. **No overlapping or back-to-back rentals for the same car.** After a rental
   ends, the car is kept **overnight** for cleaning and maintenance, so the
   next rental may start **at the earliest the day after** the previous one
   ends. Formally: for any two rentals of the same car, the later one's
   `startDate` must be **strictly greater than** the earlier one's `endDate`
   (`startDate ≥ endDate + 1 day`). Ranges may neither overlap nor touch.

## Determinations

- **totalPrice** = charged days × `car.dailyRate`, where
  **charged days = `endDate` − `startDate` + 1** (both days inclusive; a
  same-day rental counts as 1 day).
  Recalculated whenever `startDate`, `endDate`, or `car` changes.
  
## RAP Behavior

| Aspect                    | Decision                                                                                             |
| ------------------------- | ---------------------------------------------------------------------------------------------------- |
| Implementation type       | **Managed**, **without draft**                                                                         |
| Concurrency               | Optimistic locking via **ETag** (`LOCAL_LAST_CHANGED_AT` is the total ETag)                            |
| Actions                   | `pickUp`, `returnCar`, `cancel` — instance actions with dynamic feature control                        |
| Numbering                 | Managed **UUID** numbering (standard RAP pattern; the framework fills the key on create)               |
| Authorizations            | None needed — `authorization master ( global )` with no checks                                         |
| Currency                  | `@Semantics.amount.currencyCode`; every amount carries a currency field, hidden where it has no meaning |
| KPIs                      | Plain computed elements in the CDS layer (see [KPIs](#kpis)). **No** analytical page                   |
| Car / Location / Customer | **Read-only** — maintained only by the seed class `ZCL_CARRENTAL_SEED_<NN>`, never edited through the UI |
| Value helps               | Car, Customer and Location associations expose `@Consumption.valueHelpDefinition` for the frontend      |

The service binding `ZSB_CarRental_<NN>` is of type **OData V4 – UI** and
carries **no `@UI` annotations** — no `@UI.lineItem`, `@UI.identification`,
`@UI.facet`, `@UI.headerInfo`. Only *semantic* and *consumption* annotations
belong in the backend: value helps, currency semantics, text associations, and
the criticality element. Building the user interface is the students' job — see
[Frontend / UI](#frontend--ui).

## Frontend / UI

Each student builds a **freestyle SAPUI5** frontend (TypeScript, XML views) in
SAP Business Application Studio (BAS) against `ZSB_CarRental_<NN>`, writing the
views, controllers and binding code by hand.

The full frontend specification — screens, coding standards, OData V4 usage
rules, and the acceptance checklist — lives in
**[car-rental-ui.md](car-rental-ui.md)**. That file is the document students
hand to Copilot in BAS.

### Generating the app

In BAS: **SAP Fiori application generator** → template **Basic** (SAPUI5
freestyle, **TypeScript** enabled) → data source **Connect to a System** → the
destination pointing at steampunk → service `ZSB_CarRental_<NN>`. The result is
an empty view wired to an OData V4 model; everything else follows
[car-rental-ui.md](car-rental-ui.md).

## KPIs

### Fleet-level

Intended to be surfaced by the students' frontend, typically as a header area
above the rental list — **no live aggregation, no analytical page**. The backend
only makes the underlying data available; displaying it is part of the frontend
exercise and is a [stretch goal](#stretch-goals).

| KPI                         | Definition                                                        |
| --------------------------- | ----------------------------------------------------------------- |
| **Total rentals**           | Count of all rentals (excluding `Cancelled`)                      |
| **Total revenue**           | Sum of `totalPrice` over `Active` + `Completed` rentals           |
| **Average rental duration** | Mean of charged days across non-cancelled rentals                 |
| **Fleet utilization (%)**   | Cars with an `Active` rental ÷ total cars × 100                   |

### Per-car

| KPI                    | Definition                                          |
| ---------------------- | --------------------------------------------------- |
| **Rentals for this car** | Count of the car's non-cancelled rentals          |
| **Revenue for this car** | Sum of `totalPrice` over its `Active` + `Completed` rentals |
| **Current location**     | Derived — see [above](#derived-current-location)  |


### Naming convention (important)

**DDIC and repository object names are system-global, not package-scoped** — two
students cannot both have a table called `ZCAR`. Every object therefore carries
a **student suffix** `_<NN>`, where `<NN>` is the student's two-digit number
(e.g. `_01`, `_02`, …, `_25`):

- Packages: `ZCARRENTAL_<NN>`
- Tables: `ZLOCATION_<NN>`, `ZCAR_<NN>`, `ZCUSTOMER_<NN>`, `ZRENTAL_<NN>`
- Classes: `ZCL_CARRENTAL_SEED_<NN>`
- Everything students create during the exercises (CDS views, behavior
  definitions, service bindings, …) follows the same `_<NN>` suffix —
  put this rule in the student handout, or their AI-generated objects will
  collide with each other.

Use two-digit student numbers rather than initials: names must fit **16
characters for tables** and **30 for CDS entities**, and numbers keep the
scheme scriptable.

### RAP artifacts (per student `<NN>`)

| Layer                | Object(s)                                                              |
| -------------------- | ---------------------------------------------------------------------- |
| Package              | `ZCARRENTAL_<NN>`                                                       |
| Tables               | `ZLOCATION_<NN>`, `ZCAR_<NN>`, `ZCUSTOMER_<NN>`, `ZRENTAL_<NN>`         |
| Seed class           | `ZCL_CARRENTAL_SEED_<NN>`                                               |
| Interface CDS        | `ZI_Location_<NN>`, `ZI_Customer_<NN>`, `ZI_Car_<NN>`, `ZI_Rental_<NN>` |
| Behavior definitions | `ZI_Rental_<NN>` (managed, no draft), `ZI_Car_<NN>` (read-only)         |
| Behavior pool        | `ZBP_I_RENTAL_<NN>`                                                     |
| Projection CDS       | `ZC_Rental_<NN>`, `ZC_Car_<NN>`, `ZC_Location_<NN>`, `ZC_Customer_<NN>` |
| Service definition   | `ZSD_CarRental_<NN>`                                                    |
| Service binding      | `ZSB_CarRental_<NN>` (OData V4 – UI, **no `@UI` annotations**)          |

Additional fields (added by the setup class, not in the original entity spec):

- **Currency**: a `CURRENCY` (`cuky`, fixed `EUR`) key field on every table that
  carries an amount (`ZCAR_<NN>`, `ZRENTAL_<NN>`), referenced by
  `@Semantics.amount.currencyCode` in the CDS layer.
- **RAP admin/ETag fields** on `ZRENTAL_<NN>`: `CREATED_BY`, `CREATED_AT`,
  `LAST_CHANGED_BY`, `LAST_CHANGED_AT`, `LOCAL_LAST_CHANGED_AT`
  (`LOCAL_LAST_CHANGED_AT` is the total ETag, giving the BO its optimistic
  concurrency). The seed class fills these on
  insert (author + timestamps) since seed inserts bypass the managed BO.
