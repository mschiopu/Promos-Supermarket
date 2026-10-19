# Promotion Matchlist

> An Android shopping-list app that helps clients discover when an item on their completed list is available through an active promotion.

**Status:** Product/technical design only — no implementation included  
**Scope:** MVP-first Android application  
**Primary audience:** Product, design, Android, backend, QA, and future data-import contributors

---

## Product overview

Promotion Matchlist stores promotions and lets a client create an acquisition/shopping list. When the client marks the list complete, the app evaluates each list item against promotions that are active for the completion date and proposes adding a specific promoted product to the client’s acquisition result.

A promotion contains:

- Start and end dates
- Promotion name
- Seller/company
- Price
- Technical details as JSON

The product deliberately separates **what the client wants** (a list item) from **what a seller promotes** (a specific product). Matching can therefore be transparent and explainable rather than silently replacing the client’s request.

### MVP success criteria

1. A client can create, rename, edit, and complete a list.
2. A client can add list items with a human-readable name and optional quantity/notes.
3. Active promotions can be imported or seeded and viewed.
4. On completion, matching promotions are found using explicit, predictable rules.
5. The client sees proposed promoted products and can accept or dismiss each proposal.
6. Accepted proposals are recorded as specific products associated with the completed list.
7. Core behavior is testable without a live network connection.

### Explicit defaults

These defaults keep the first release small and are intentionally open to revision:

- **Platform:** Android phones first; tablet optimization is later.
- **Language:** Kotlin.
- **UI:** Jetpack Compose.
- **Storage:** Local Room database for the MVP.
- **Dates:** Calendar dates, not times; the end date is inclusive.
- **Currency:** ISO 4217 currency code stored with each price; display locale defaults to the device locale.
- **Quantity:** One requested unit per item by default; quantity is optional in MVP.
- **Offline behavior:** Lists and previously synced promotions remain usable offline. A completed list can be matched against locally available promotions.
- **Identity:** No account is required for MVP. Data belongs to the current device profile.
- **Notifications:** None in MVP.
- **Promotion ingestion:** Local seed data plus a replaceable import boundary; a production feed/API is a later integration.

---

## Assumptions and open decisions

### Assumptions for MVP

- A promotion applies to a seller/company and a specific product, not merely to a broad category.
- Product names and aliases are sufficient for initial matching; barcode/catalog identity is not required.
- The user is the only client and does not need shared lists.
- A list completion is a point-in-time user action and is not automatically undone.
- Promotions with missing or invalid dates are rejected during import rather than guessed.
- A promotion with `startDate <= completionDate <= endDate` is active.
- Price is the promoted unit price unless `technicalDetails` explicitly documents another pricing basis.

### Open decisions to resolve before production

| Decision | MVP default | Later question |
|---|---|---|
| Promotion source | Seed/imported local records | Which partner API, feed, or admin workflow is authoritative? |
| Product identity | Normalized name plus optional SKU/barcode | Can a stable catalog ID be required? |
| Matching scope | Name/alias matching across all sellers | Should the client filter by location, seller, brand, or category? |
| Price semantics | `amount` + `currency`; unit unspecified unless documented | How should pack size, loyalty pricing, taxes, and membership rules work? |
| Date/time zone | Device local calendar date | Should promotions use seller/store time zones? |
| Multiple matches | Show ranked proposals; user chooses | Should the app auto-select the lowest valid price? |
| Data retention | Keep lists and decisions on device | Is cloud backup, sign-in, or account deletion required? |
| Manual correction | Edit/dismiss a proposal | Should a client map a list item to a product alias? |
| Compliance | No sensitive data intentionally collected | Are analytics, consent, or regional data policies needed? |

---

## User flow

1. **Open the app** and view active or recent lists.
2. **Create a list** with an optional title, such as “Office supplies”.
3. **Add requested items**, for example “A4 paper”, with optional quantity and note.
4. **Review and edit** item names before completion.
5. **Complete the list.** The app records the completion date and runs matching against active promotions available on the device.
6. **Review proposals.** Each proposal explains the matched item, promoted product, seller/company, price, promotion name, validity dates, and match reason.
7. **Accept or dismiss** each proposal. Acceptance adds a specific promoted product to the completed list’s acquisition result; dismissal preserves the original request without a recommendation.
8. **Inspect results** later from the completed-list history. The original request and the recommendation decision remain visible.

### Empty and error states

- No active promotions: explain that no local promotion matched and keep the completed list intact.
- No match for an item: show “No promotion found” without treating it as an error.
- Invalid imported promotion: reject it with a row-level reason.
- Duplicate list item: allow it, but offer a lightweight merge action later; do not silently merge in MVP.
- Partial import or matching failure: show completed local results and a retryable error for the failed operation.

---

## Architecture and skeleton directory tree

Use a small single-module, feature-oriented structure for MVP. Add modules only when build times or team boundaries justify them.

```text
promotion-matchlist/
├── README.md
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── app/
│   ├── build.gradle.kts
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   └── kotlin/com/example/promotionmatchlist/
│       │       ├── App.kt
│       │       ├── MainActivity.kt
│       │       ├── navigation/
│       │       │   └── AppNavGraph.kt
│       │       ├── data/
│       │       │   ├── local/
│       │       │   │   ├── AppDatabase.kt
│       │       │   │   ├── PromotionDao.kt
│       │       │   │   └── ShoppingListDao.kt
│       │       │   ├── model/
│       │       │   └── repository/
│       │       │       ├── PromotionRepository.kt
│       │       │       └── ShoppingListRepository.kt
│       │       ├── domain/
│       │       │   ├── matching/
│       │       │   │   ├── PromotionMatcher.kt
│       │       │   │   └── MatchRanker.kt
│       │       │   └── usecase/
│       │       │       ├── CompleteListUseCase.kt
│       │       │       └── MatchCompletedListUseCase.kt
│       │       ├── feature/
│       │       │   ├── lists/
│       │       │   ├── listdetail/
│       │       │   ├── completion/
│       │       │   ├── promotions/
│       │       │   └── settings/
│       │       └── design/
│       │           ├── AppTheme.kt
│       │           └── Components.kt
│       └── test/
│           ├── domain/
│           ├── data/
│           └── feature/
└── docs/
    ├── matching-rules.md
    └── promotion-import-format.md
```

### Architectural boundaries

- **UI:** Compose screens and state holders; no direct database access.
- **Domain:** Completion and matching rules as deterministic Kotlin code.
- **Data:** Room entities/DAOs, import adapters, and repositories.
- **Navigation:** Routes and arguments only; pass IDs rather than large objects.
- **Import boundary:** Accept a typed promotion input and return validation errors so the future API can replace seed data without changing matching logic.

---

## Core data models and schema

The following is a logical schema, not an implementation requirement. Room entities may use UUID strings or generated `Long` IDs; the MVP default is UUID strings to simplify import/export and debugging.

### `promotion`

| Field | Type | Required | Notes |
|---|---|---:|---|
| `id` | string | yes | Stable promotion identifier |
| `name` | string | yes | Human-readable promotion name |
| `sellerCompany` | string | yes | Seller, retailer, or company |
| `productName` | string | yes | Specific promoted product |
| `normalizedProductName` | string | yes | Lowercase, whitespace-normalized matching key |
| `startDate` | `YYYY-MM-DD` | yes | Inclusive |
| `endDate` | `YYYY-MM-DD` | yes | Inclusive; must not precede start date |
| `priceAmount` | decimal | yes | Non-negative promoted price |
| `currency` | string | yes | Three-letter ISO 4217 code, e.g. `USD` |
| `technicalDetailsJson` | JSON object | yes | Validated, opaque extension data |
| `aliases` | string array | no | Optional product-name synonyms |
| `sourceUpdatedAt` | instant | no | When source data last changed |

Example:

```json
{
  "id": "promo-1001",
  "name": "Back-to-school paper offer",
  "sellerCompany": "Example Office Co.",
  "productName": "A4 recycled copy paper, 500 sheets",
  "normalizedProductName": "a4 recycled copy paper 500 sheets",
  "startDate": "2026-08-01",
  "endDate": "2026-10-31",
  "priceAmount": 4.99,
  "currency": "USD",
  "technicalDetailsJson": {
    "brand": "Example",
    "packCount": 1,
    "unit": "ream",
    "barcode": "0000000000000"
  },
  "aliases": ["A4 paper", "copy paper"]
}
```

### `shopping_list`

| Field | Type | Required | Notes |
|---|---|---:|---|
| `id` | string | yes | List identifier |
| `title` | string | yes | User-visible title |
| `status` | enum | yes | `ACTIVE` or `COMPLETED` in MVP |
| `createdAt` | instant | yes | Device timestamp |
| `completedAt` | instant | no | Set when status becomes completed |
| `completionDate` | `YYYY-MM-DD` | no | Device-local calendar date used for matching |

### `shopping_list_item`

| Field | Type | Required | Notes |
|---|---|---:|---|
| `id` | string | yes | Item identifier |
| `listId` | string | yes | Parent list |
| `requestedName` | string | yes | Original client request |
| `normalizedRequestedName` | string | yes | Normalized matching key |
| `quantity` | integer | no | Positive when present |
| `note` | string | no | Free-text clarification |
| `position` | integer | yes | Stable display order |

### `match_proposal`

| Field | Type | Required | Notes |
|---|---|---:|---|
| `id` | string | yes | Proposal identifier |
| `listItemId` | string | yes | Original requested item |
| `promotionId` | string | yes | Matched promotion |
| `score` | decimal | yes | Internal ranking score, not a promise of relevance |
| `reason` | string | yes | Explainable reason, e.g. `alias match` |
| `decision` | enum | yes | `PENDING`, `ACCEPTED`, or `DISMISSED` |
| `createdAt` | instant | yes | Proposal creation time |

A proposal is a snapshot of a matching decision. If promotion data changes later, the historical proposal remains understandable; a future refresh can create a new proposal rather than mutating the old decision.

---

## Matching behavior

Matching runs only when a list is completed (or when the user explicitly retries matching). It uses the list’s `completionDate`, not the current date, so historical results are reproducible.

### MVP algorithm

1. Load promotions where `startDate <= completionDate <= endDate`.
2. For each list item, normalize text by trimming, lowercasing with a stable locale, collapsing whitespace, and removing harmless punctuation.
3. Match if either:
   - the normalized requested name equals the normalized product name;
   - the requested name equals a declared promotion alias; or
   - one normalized name contains the other **and** the shorter value is at least 4 characters.
4. Reject obvious weak matches where the shared text is only a generic stop word such as “item”, “pack”, or “new”.
5. Rank candidates in this order:
   - exact normalized product-name match;
   - exact alias match;
   - contains match;
   - lower promoted price as a tie-breaker;
   - seller/company name alphabetically as a stable final tie-breaker.
6. Create proposals for the top candidates. **Default:** maximum three proposals per list item; do not auto-accept.
7. Show the match reason and enough product/promotion details for the client to decide.

### Deliberate non-goals for MVP

- No machine-learning or semantic search.
- No automatic substitution of a requested item.
- No assumptions about pack size, brand, tax, shipping, loyalty membership, or geographic availability unless represented in `technicalDetailsJson` and shown to the user.
- No “cheapest overall” claim when currencies or units differ.

### Later matching enhancements

- Canonical product catalog IDs, UPC/EAN/barcode matching, brand and pack-size constraints.
- Fuzzy token matching with a measured confidence threshold.
- Seller/location eligibility and inventory status.
- User-maintained aliases and feedback-based ranking.
- Server-side matching for large promotion catalogs.

---

## Screens

### MVP screens

1. **List home** — active and completed lists, create-list action, empty state.
2. **Create/edit list** — title and item editor.
3. **List detail** — items, add/edit/delete item, complete action.
4. **Match results** — proposals grouped by requested item; accept/dismiss actions; no-match rows.
5. **Completed list detail** — original items, accepted promoted products, dismissed proposals, completion date.
6. **Promotions** — searchable local promotion inventory for inspection and testing.
7. **Import/settings** — load sample data and display import validation results; keep this minimal and developer-oriented in MVP.

### Later screens

- Seller/location filters.
- Promotion detail with terms and source link.
- Account, cloud sync, shared lists, and cross-device history.
- Notifications for saved items or expiring promotions.

### Accessibility and UX baseline

Use semantic Compose labels, minimum touch targets, readable contrast, support for large font sizes, and clear non-color status indicators. Every proposal decision must be available without a gesture-only interaction.

---

## Artifacts and deliverables

### MVP deliverables

- Android project with reproducible debug build.
- Compose UI for the MVP screens above.
- Room schema/migrations for lists, items, promotions, and proposals.
- Seed/import format and validation report.
- Deterministic matching use case with documented rules.
- Unit, DAO, and UI tests covering the critical flows.
- Sample promotion dataset with fictional companies/products.
- This README plus a short `docs/matching-rules.md` decision record.

### Later deliverables

- Authenticated backend/API and sync contract.
- Admin or partner ingestion pipeline.
- Product catalog identity service.
- Analytics/observability plan with explicit consent and retention decisions.
- Release signing, Play Store listing, privacy policy, and support process.

---

## Development phases

### Phase 0 — Product alignment

- Confirm open decisions, terminology, and example promotion data.
- Agree on exact matching examples and unacceptable false positives.
- Produce low-fidelity screen flows and accessibility checklist.

### Phase 1 — Local foundation

- Create Android/Kotlin/Compose project.
- Add Room entities, migrations, repositories, and seed/import validator.
- Add navigation and app theme.

### Phase 2 — List management

- Implement list and item CRUD.
- Add active/completed states and completion date.
- Add empty, validation, and persistence states.

### Phase 3 — Matching and review

- Implement date filtering, normalization, ranking, and proposal persistence.
- Build match-results and completed-list screens.
- Make accept/dismiss decisions durable and reversible only through an explicit later action.

### Phase 4 — Quality and release readiness

- Run automated tests and manual accessibility checks.
- Test database migration from a prior schema version.
- Verify offline behavior, import failures, and large-but-realistic datasets.
- Produce an internal debug/release candidate and known-limitations list.

### Phase 5 — Post-MVP

Prioritize based on evidence: backend sync, catalog identity, seller/location filtering, improved matching, accounts, notifications, and sharing. Avoid adding these before the local matching loop is reliable and understandable.

---

## Test strategy

### Unit tests

- Date boundary: promotion is active on both start and end dates.
- Before/after date: inactive promotions never generate proposals.
- Text normalization: case, whitespace, punctuation, and aliases.
- Exact match outranks alias and contains match.
- Ranking is deterministic, including ties.
- Generic weak matches are rejected.
- Invalid price, currency, dates, and JSON are rejected with actionable errors.
- One item can produce multiple pending proposals, capped at the configured maximum.

### Data and repository tests

- Create, edit, complete, and reload a list.
- Foreign-key relationships prevent orphan items/proposals.
- Accepted and dismissed decisions persist across app restart.
- Import is idempotent for the same promotion ID.
- Migration tests cover every schema version shipped after the first release.

### UI tests

- Create a list and add an item.
- Complete a list and reach match results.
- Accept one proposal and dismiss another.
- Verify no-match and no-promotions states.
- Verify large text, screen-reader labels, rotation/process recreation where supported, and disabled/loading/error states.

### End-to-end acceptance scenario

Given a list item “A4 paper”, a promotion for “A4 recycled copy paper, 500 sheets” with alias “A4 paper”, and a completion date inside the promotion period, the app must show a pending proposal containing the product name, seller/company, price, promotion name, validity dates, and an alias-match explanation. Accepting it must show the promoted product in the completed-list result without changing the original requested name.

### Test data rules

Use fictional sellers and products in source control. Include boundary dates, duplicate names from different sellers, different currencies, invalid records, multiple candidates, and zero-match items. Do not commit real client or shopping-history data.

---

## Suggested stack

| Concern | Suggested MVP choice | Rationale |
|---|---|---|
| Language | Kotlin | Native Android support and concise domain code |
| UI | Jetpack Compose + Material 3 | Fast, accessible declarative UI |
| State | ViewModel + Kotlin Coroutines/Flow | Lifecycle-aware screen state |
| Navigation | Navigation Compose | Small, conventional route graph |
| Local database | Room | Typed persistence and migrations |
| Serialization | `kotlinx.serialization` | Typed import/export and JSON handling |
| Date/time | `java.time` / `kotlinx-datetime` | Explicit calendar-date handling |
| Dependency injection | Manual factories initially; Hilt when justified | Avoid framework overhead in a small MVP |
| Networking later | Retrofit or Ktor client | Replaceable behind repository/import boundary |
| Build | Gradle Kotlin DSL + version catalog | Reproducible dependency management |
| Quality | JUnit, AndroidX Test, Compose UI Test, MockK or fakes | Unit, persistence, and UI coverage |
| CI later | GitHub Actions or equivalent | Build, lint, test, and artifact checks |

### Suggested package principles

- Keep matching pure and independent of Android classes.
- Keep Room entities separate from domain models when transformations become non-trivial.
- Prefer fakes over mocking for repository tests.
- Add a server or multi-module architecture only when a concrete requirement demands it.

---

## Non-goals and product guardrails

- This README does not define or implement a backend, authentication, payment, inventory reservation, or checkout flow.
- A promotion proposal is informational; it is not a purchase, price guarantee, or availability guarantee.
- The app must not silently overwrite a client’s requested item.
- Technical-details JSON is extensible but must not be rendered as trusted executable content.
- Any future analytics, accounts, or synchronization must be added with explicit privacy, retention, and deletion decisions.

---

## Definition of done for MVP

The MVP is ready for internal release when a tester can install the app, load fictional promotions, create and complete a list, receive deterministic active-promotion proposals, accept/dismiss them, close and reopen the app, and see the same decisions. Automated tests cover date boundaries, normalization, ranking, persistence, import validation, and the primary UI flow; known limitations are documented rather than hidden.
