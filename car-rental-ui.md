# Car Rental — UI Specification (Freestyle SAPUI5)

The frontend for the [car-rental domain](car-rental.md): a **freestyle SAPUI5**
application consuming the OData V4 service `ZSB_CarRental_<NN>`. This document
is the implementation spec — it is written to be handed to an AI coding
assistant (GitHub Copilot in BAS) together with the generated project skeleton.

**Before using this spec, replace `<NN>` everywhere with your two-digit student
number** (or `FLEXSO` if you use the shared reference service).

## Ground rules

- **Freestyle SAPUI5** — no Fiori Elements, no `sap.fe`, no annotation-driven
  pages. All views, controllers, and bindings are written by hand.
- **TypeScript**, XML views, and the project skeleton produced by the SAP Fiori
  generator (template *Basic*, TypeScript enabled). Keep the generated
  `Component.ts`, `manifest.json`, and folder layout; extend them, don't fight
  them.
- The OData model is **`sap.ui.model.odata.v4.ODataModel`**, configured in
  `manifest.json` by the generator. Follow the
  [OData V4 rules](#odata-v4-rules) below — V2 patterns do not exist here.
- The backend is the single source of truth for business logic. The frontend
  **never re-implements validations** (overlaps, capacity, pickup location) —
  it submits and displays the backend's messages.

## Coding standards

These are hard requirements, not suggestions:

1. **ES module imports only.** Every controller/class imports its dependencies
   (`import Button from "sap/m/Button"`), never accesses `sap.m.*` /
   `sap.ui.*` globals, and never uses the `sap.ui.define` callback style in
   TypeScript sources. In XML views, load binding types and formatters with
   `core:require`.
2. **Typed event handlers.** Use the control-specific event types
   (`Button$PressEvent`, `SearchField$SearchEvent`, …) imported from the
   control's module — not the generic `Event` — and let the compiler check
   `getSource()` / `getParameter()` usage. No `any`, no `@ts-ignore`; when a
   cast is unavoidable, cast to the concrete UI5 type.
3. **No inline scripts or styles** (CSP). Logic lives in controller files,
   styling in `css/style.css`. Prefer standard controls and their properties
   over custom CSS at all.
4. **Every user-visible text comes from `i18n/i18n.properties`.** No hardcoded
   labels, titles, button texts, or messages in views or controllers.
5. **Format with binding types, not formatters.** Use
   `sap.ui.model.odata.type.*` types in the binding (`Date`, `Decimal`,
   `String`); custom formatters only where no type fits (e.g. mapping
   `StatusCriticality` to a `ValueState`).
6. **Forms use `sap.ui.layout.form.Form` with `ColumnLayout`**
   (`columnsM="2" columnsL="3" columnsXL="4"`), never `SimpleForm`.
7. **Amounts are currency-aware.** Bind `TotalPrice` together with its
   `Currency` element using `sap.ui.model.odata.type.Currency` (composite
   binding), so `1 250,00 EUR` renders per locale.
8. **Stable IDs** on views and on every control that a test would need to find.
9. The project must **compile cleanly** (`npm run ts-typecheck` or the
   project's build script) and pass `ui5lint` with no errors.

## Application structure

```
webapp/
├── Component.ts                     (generated — unchanged)
├── manifest.json                    (routing + models configured here)
├── i18n/i18n.properties
├── view/
│   ├── App.view.xml                 sap.m.App shell, busy handling
│   ├── RentalList.view.xml          route: rentals   (default)
│   └── RentalDetail.view.xml        route: rental/{rentalId}
├── controller/
│   ├── App.controller.ts
│   ├── RentalList.controller.ts
│   └── RentalDetail.controller.ts
├── model/
│   └── formatter.ts                 (only what binding types cannot do)
└── fragment/
    └── RentalCreateDialog.fragment.xml   (Tier 2)
```

Routing is declared in `manifest.json` (`sap.ui5/routing`): two routes,
`rentals` (pattern `""`) and `rental` (pattern `rental/{rentalId}`), each with
its own target view inside the `App` view's pages aggregation. Navigation uses
the router (`this.getOwnerComponent().getRouter().navTo(...)`) — never direct
view manipulation.

The main entity set is **`Rental`**; `Car`, `Customer`, and `Location` are
read-only reference data reached through associations and value helps. Display
**association texts, never UUIDs**: the service exposes text elements
(car license plate + model, customer name, location names) directly on the
Rental projection — bind those.

---

## Tier 1 — required scope

### 1. Rental list (`RentalList.view.xml`)

A `sap.m.Page` containing a **`sap.m.Table`** bound to `/Rental`:

| Column      | Bound to                          | Notes                                        |
| ----------- | --------------------------------- | -------------------------------------------- |
| Car         | license plate + model text fields | `ObjectIdentifier` (title + text)            |
| Customer    | customer name text field          |                                              |
| Period      | `StartDate` – `EndDate`           | `odata.type.Date`; two Texts or one combined |
| Pickup      | pickup location name              |                                              |
| Drop-off    | drop-off location name            |                                              |
| Status      | `Status`                          | `ObjectStatus`; coloring is Tier 2           |
| Total price | `TotalPrice` + `Currency`         | `odata.type.Currency`, right-aligned         |

- **Search:** a `SearchField` in the table toolbar filtering server-side
  (`ODataListBinding.filter`) on the text-like fields (license plate, customer
  name, location names) with `FilterOperator.Contains` combined via OR. Never
  filter on UUIDs or amounts.
- **Sort:** default sort `StartDate` descending, set in the items binding.
- **Count:** table header shows the total (`$count: true` on the binding).
- Row press navigates to the detail route, passing the rental's key.
- `growing="true"` on the table; let the V4 binding page the data.

### 2. Rental detail (`RentalDetail.view.xml`)

A `sap.f.DynamicPage`. Title area: car text + status. Content: one
`form:Form` (ColumnLayout) with four `FormContainer`s:

1. **Rental** — start date, end date, total price (currency-bound), status
2. **Car** — license plate, model
3. **Customer** — name
4. **Locations** — pickup name, drop-off name

The controller binds the view to the routed rental via
`view.bindElement({ path: "/Rental(" + rentalId + ")" })` (V4 key syntax —
mind that the key is a UUID/`Edm.Guid`). Handle the not-found case: if the
context resolves to nothing, show a message page or navigate back.

### 3. Lifecycle action buttons

Buttons **Pick up**, **Return car**, **Cancel** in the detail page header
(and optionally per row in the list):

- Invoke via an operation binding on the rental's context:

  ```ts
  const action = (context.getModel() as ODataModel).bindContext(
      "com.sap.gateway.srvd.zsd_carrental_<nn>.v0001.pickUp(...)",
      context
  );
  await action.invoke();
  ```

  Read the exact action namespace from `$metadata` once and keep it in a
  single constant — don't scatter it across controllers.
- **Enablement comes from the service, not from client-side status logic.**
  The RAP dynamic feature control publishes per-instance operation
  availability: `$metadata` annotates each action with
  `Core.OperationAvailable` pointing at a property of the entity. Request that
  property in the binding and bind each button's `enabled` to it. Inspect
  `$metadata` for the exact property names — do not hardcode
  `Status === 'CREATED'`-style rules.
- After a successful invoke, **refresh the rental's context** so status, button
  states, and the list all reflect the new state; show a `MessageToast`.
- On failure, surface the backend message via `MessageBox.error` — RAP returns
  the human-readable reason (see [error handling](#error-handling)).

## Tier 2 — stretch goals

In order of value; pick up only when Tier 1 is done and demonstrated.

1. **Status criticality coloring.** Bind `ObjectStatus.state` to the service's
   `StatusCriticality` element via a small formatter mapping
   `0..3 → None / Error / Success / Warning` (criticality convention:
   1 = red, 2 = yellow, 3 = green).
2. **Create a rental.** A dialog fragment (`RentalCreateDialog`) with a
   `form:Form`: car, customer, pickup location, drop-off location, start and
   end date (`DatePicker`s). Create via `ODataListBinding.create(...)` on the
   list binding; on success `MessageToast` + the table updates itself, on
   validation errors show the backend messages and keep the dialog open.
   This is where the ETag and the backend validations become visible — that is
   the point of the exercise.
3. **Value helps** for car, customer, and locations in the create dialog: the
   service exposes value-help metadata (`@Consumption.valueHelpDefinition`).
   Minimum: a `ComboBox`/`Select` bound to `/Car`, `/Customer`, `/Location`
   showing text, keeping the UUID as key. Nicer: `Input` with suggestions.
4. **Fleet KPI header** above the rental list: total rentals, total revenue,
   average duration, fleet utilization (definitions in
   [car-rental.md → KPIs](car-rental.md#kpis)). The dataset is deliberately
   small: read the needed data with dedicated list bindings and compute the
   figures in the controller into a local JSON model. Render as a row of
   `GenericTile` or `ObjectNumber`s. No analytical page.
5. **Per-car view**: a simple page listing cars with their per-car KPI elements
   (rental count, revenue, derived current location) — these are plain elements
   on the Car entity, no client-side computation needed.

## OData V4 rules

The V4 model is not the V2 model. Copilot: generate **only** V4 patterns.

- There is **no** `model.read()`, `model.create()`, `model.remove()`,
  `model.callFunction()`. Everything goes through bindings.
- Read: list/context bindings declared in views, or `bindList`/`bindContext`
  in controllers. Create: `ODataListBinding.create()`. Delete:
  `Context.delete()`. Actions: `bindContext("...action(...)", ctx).invoke()`.
- Updates are batched into groups — the generated model uses group `$auto`;
  keep it, and don't call `submitBatch` unless you introduce a deferred group
  (the create dialog may use one, e.g. `$$updateGroupId: "rentalCreate"`).
- After an action or update, refresh the affected context
  (`context.refresh()`) or request side effects; server-computed fields
  (`TotalPrice`, `Status`, `StatusCriticality`) never update by themselves.
- Concurrency is via **ETag** — the model handles it; on a 412 tell the user
  the rental changed and refresh.
- Filtering/sorting on list bindings goes server-side
  (`binding.filter(...)`, `binding.sort(...)`), not through JSON-model tricks.
- Import the V4 types from their modules (`sap/ui/model/odata/v4/ODataModel`,
  `.../ODataListBinding`, `.../Context`) so calls are compiler-checked.

## Error handling

- RAP validation failures arrive as OData error responses with message texts.
  Show them; never swallow them. For bound messages, use the message model
  (`sap.ui.core.Messaging`) and a `MessageButton`/`MessagePopover` on the
  create dialog (Tier 2); `MessageBox.error` with the response message is the
  Tier 1 minimum for action failures.
- Every remote operation sets a busy state (`busyIndicatorDelay` default;
  page- or control-level `busy`, not global blocking where avoidable).
- `invoke()` and `create()` return promises — always `await`/`.catch()` them;
  an unhandled rejection is a failed checklist item.

## Acceptance checklist

Tier 1 is done when:

- [ ] App starts from the BAS preview without console errors.
- [ ] `npm run build` (TypeScript compile) and `ui5lint` pass with no errors.
- [ ] Rental list shows seeded data with readable texts — no UUIDs anywhere.
- [ ] Search narrows the list server-side; clearing it restores the full list.
- [ ] Clicking a rental opens its detail page; browser back returns to the list.
- [ ] Prices render as localized currency amounts; dates per locale.
- [ ] On a `Created` rental: *Pick up* and *Cancel* enabled, *Return car*
      disabled — driven by service data, not client logic.
- [ ] Invoking *Pick up* turns the rental `Active` on screen without a manual
      reload; the button states follow.
- [ ] A failed action shows the backend's message text.
- [ ] All texts come from i18n; no inline styles anywhere.
