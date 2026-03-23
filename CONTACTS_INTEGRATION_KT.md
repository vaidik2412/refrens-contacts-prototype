# Contacts Integration — Knowledge Transfer

> **Last updated:** 2026-03-23
> **Prototype:** [refrens-contacts-prototype.vercel.app](https://refrens-contacts-prototype.vercel.app)
> **Repo:** [github.com/vaidik2412/refrens-contacts-prototype](https://github.com/vaidik2412/refrens-contacts-prototype)

---

## 1. Overview

Contacts is a standalone entity in Refrens representing **individual people** (not companies). A contact can be linked to **Clients** (via `contactRelations`) and **Leads** (embedded directly on the lead document). This document covers the full architecture — schema, backend services, frontend components, and integration flows.

### Repos Involved

| Repo | Role |
|---|---|
| **talos** | Mongoose schemas (contacts, contactRelations, contactActivities, leads) |
| **serana** | Backend services, hooks, validation, API routes (Feathers.js) |
| **jupiter** | Shared UI component library (LeadForm, LinkContactModal, LinkClientDrawer, ContactView) |
| **lydia** | App pages (lead create/edit, contacts index, client create/edit) |
| **fence** | Config constants (indexed field categories, activity types, audit field mappings) |

---

## 2. Data Models (Talos)

### 2.1 Contacts (`contacts.ts`)

Represents an individual person associated with a business.

| Field | Type | Notes |
|---|---|---|
| `uniqueKey` | String, Required, Unique | Typically `firstName-lastName-businessId` |
| `business` | ObjectId, Required | Parent business ref |
| `salutation` | String | Mr., Mrs., Dr., etc. |
| `firstName` | String, Required | Max 500 chars |
| `lastName` | String | Max 500 chars |
| `avatar` | String | Validated URL |
| `country` | String | ISO code (IN, US, etc.) |
| `email`, `emailAlt1`, `emailAlt2` | String | Email addresses |
| `phone`, `phoneAlt1`, `phoneAlt2` | String | Auto-formatted phone numbers |
| `social.linkedin/twitter/facebook/instagram/github` | String | Validated URLs per platform |
| `address.street/city/pincode/state/stateCode/gstState/district/building` | String | Full address object |
| `govIdentities.pan.number` | String | India-only validation |
| `govIdentities.aadhaar.number` | String | India-only validation |
| `govIdentities.passport.number` | String | — |
| `isMerged` | Boolean | Whether merged into another contact |
| `mergedTo` | ObjectId | Target contact after merge |
| `mergedBy` | ObjectId | User who performed merge |
| `isArchived` | Boolean | Soft archive |
| `isRemoved` | Boolean | Soft delete |
| `removedAt`, `removedBy`, `removeReason` | Date/ObjectId/String | Deletion metadata |
| `isHardRemoved` | Boolean | Permanent deletion flag |
| `creator` | ObjectId, Required | User who created the contact |
| `refrensUserId` | ObjectId | If contact is a registered Refrens user |
| `vendorFields` | IndexedCustomFieldsValue | Custom fields (indexed) |
| `params`, `_systemMeta` | Mixed | Flexible metadata |

**Elasticsearch Index:** `m_refrens_contacts` — stores partial fields (uniqueKey, business, firstName, lastName, avatar, email, phone, merge/archive/remove status, creator, timestamps).

---

### 2.2 Contact Relations (`contactRelations.ts`)

Links a contact to a **client**. This is specifically for client relationships — leads store contacts differently (see 2.4).

| Field | Type | Notes |
|---|---|---|
| `contact` | ObjectId, Required | Ref to contact |
| `client` | ObjectId, Required | Ref to client |
| `business` | ObjectId, Required | Ref to business |
| `creator` | ObjectId, Required | User who created the link |
| `isPrimary` | Boolean, Default: false | Only one primary per client |
| `department` | String | Contact's department in client org |
| `role` | String | Contact's job title/function |
| `isActive` | Boolean, Default: true | Active status |
| `isRemoved` | Boolean, Default: false | Soft delete (unlinking) |
| `removedAt`, `removeReason` | Date/String | Deletion metadata |
| `isHardRemoved` | Boolean | Permanent deletion |
| `vendorFields` | IndexedCustomFieldsValue | Custom fields (added via talos PR #701) |

**Uniqueness constraint:** One relation per contact-client pair per business.

**Primary constraint:** Only one contact can be marked `isPrimary: true` per client.

---

### 2.3 Contact Activities (`contactActivities.ts`)

Audit log for all contact changes and linking/unlinking operations.

| Field | Type | Notes |
|---|---|---|
| `business` | ObjectId, Required | — |
| `contact` | ObjectId, Required | Contact affected |
| `actor` | ObjectId, Required | User performing action |
| `actionType` | String, Enum | `CREATE`, `UPDATE`, `DELETE`, `MERGE`, `LINKED`, `UNLINKED` |
| `fieldChanges` | Array | `{ fieldName, oldValue, newValue, fieldPath }` |
| `relationshipChanges` | Object | `{ entityType, entityId, entityDetails }` |

**Tracked entity types:** `clients`, `leads`, `vendorLeads`, `invoices`

**Tracked fields:** All basic info, social profiles, address fields, government IDs, vendorFields, merge/archive/remove status. See `contactAuditFieldMapping.json` in fence for human-readable labels.

---

### 2.4 Leads — Contact Integration (`leads.js`)

Leads store contacts in **two** ways:

#### Embedded (legacy, always present)
```js
contact: {
  name, email, phone, country, designation, phoneAlt, emailAlt
}
```

#### Referenced (new, via talos PR #745)
```js
primaryContact: ObjectId  // ref to contacts collection
otherContacts: [ObjectId] // array of refs to contacts collection
```

**Design decision:** Leads embed contact info directly rather than using a separate relations collection (like clients do) to avoid data redundancy, since businesses have many leads each with potentially multiple contacts.

---

## 3. Backend Services (Serana)

All services use **Feathers.js** with the **FlexStore** pattern.

### 3.1 Contacts Service

**Routes:**
```
POST   /businesses/:business/contacts          — Create
GET    /businesses/:business/contacts          — List (with filters)
GET    /businesses/:business/contacts/:id      — Get one
PATCH  /businesses/:business/contacts/:id      — Update
DELETE /businesses/:business/contacts/:id      — Soft delete
GET    /contacts                               — Alias (without business param)
```

**Pre-hooks (before):**
1. `authenticate('jwt')`
2. `mapRouteResource` — extract business ID from route
3. `restrictParamResource` — scope to user's business
4. `limitWithParamResource` — add business filter to queries
5. **Find/Get:** Filter `isRemoved: false`, `isHardRemoved: false`, filter merged
6. **Create:** `checkContactUniqueKey()` → `restDiscard()` (prevent system field overrides) → `addsCreator()` → `validateContact()`
7. **Patch:** `restDiscard()` → `stashData()` (save original for diff) → `validateContact()`
8. **Remove:** `softRemove()`

**Post-hooks (after):**
- `leanFastPopulate()` — populate creator (name, avatar, email) and business (name, alias, urlKey, country, users)
- `recordContactActivityPostCreateOrUpdate()` — auto-detect action type, track field changes, write to contact-activities

---

### 3.2 Contact Relations Service

**Routes:**
```
POST   /businesses/:business/contacts-relations          — Link contact to client
GET    /businesses/:business/contacts-relations          — List relations
GET    /businesses/:business/contacts-relations/:id      — Get one
PATCH  /businesses/:business/contacts-relations/:id      — Update
DELETE /businesses/:business/contacts-relations/:id      — Unlink (soft delete)
GET    /contacts-relations                               — Alias
```

**Pre-hooks (before):**
1. Auth + route mapping + business scoping (same pattern)
2. **Create:** `restDiscard()` → `addsCreator()` → `validateContactRelations()` → `validateVendorFieldPayload()`
3. **Patch:** Similar validation + `stashData()`
4. **Remove:** `softRemove()`

**Post-hooks (after):**
- `leanFastPopulate()` — populate contact, business, client details
- `recordContactLinkActivities('clients')` — creates LINKED/UNLINKED activity entries
- `serializeFieldsFromMapToObject()` — normalize vendorFields format

---

### 3.3 Contact Activities Service

**Routes:**
```
GET /businesses/:business/contacts/:contact/activities   — List activities for a contact
GET /contact-activities                                  — Alias
```

**Read-only** — CREATE/UPDATE/PATCH/DELETE are disallowed for external API. Activities are recorded internally by hooks.

---

### 3.4 Validation Logic

#### `validate-contact.js`
- `firstName`: Required on create, max 500 chars
- Social media URLs: validated against expected domains (linkedin.com, x.com, facebook.com, instagram.com, github.com)
- Pincode: format validation via `validatePinCode()`
- Merge consistency: if `isMerged: true`, `mergedTo` must exist and vice versa
- PAN/Aadhaar/Passport: India-only validation

#### `validate-contact-relations.js`
- Required: client, contact, business (on create)
- `validateUniqueRelation()`: prevents duplicate client-contact pairs (added in serana PR #4003)
- Primary constraint: only one `isPrimary: true` per client
- Entity existence: validates client and contact actually exist in the business

#### `check-contact-unique-key.js`
- Required on create
- Must be unique per business (excluding merged contacts)
- Skipped on PATCH if uniqueKey not in payload

---

### 3.5 Lead-Contact Hooks

#### `patch-lead-contact.js`
Auto-fills missing contact phone from:
1. Related user's phone (if `clientUser` or `contact.email` matches a user)
2. Invoices related to `clientBusiness` (billedBy or billedTo phone)

Adds auto-comment: "Contact Phone number set to [phone]"

#### `patch-business-configuration-contact.js`
When a contact is linked to a business owner, auto-updates `business-configurations` with that contact if not already set.

---

## 4. Config (Fence)

### Activity Action Types (`contacts/activityActionTypes.json`)
```
CREATE, UPDATE, DELETE, MERGE, LINKED, UNLINKED
```

### Activity Entity Types (`contacts/activityEntityTypes.json`)
```
clients, leads, vendorLeads, invoices
```

### Audit Field Mapping (`contacts/contactAuditFieldMapping.json`)
Maps field paths to human-readable labels:
- `uniqueKey` → "Contact Key"
- `firstName` → "First Name"
- `address.street` → "Street"
- `social.linkedin` → "LinkedIn"
- `govIdentities.pan.number` → "PAN Number"
- ... (all tracked fields)

### Indexed Field Categories (`businessConfig/indexed-field-categories.json`)
`CONTACTS_RELATIONS` registered as a category (fence PR #592) — enables custom fields on contact relations that are searchable/filterable.

---

## 5. Frontend Components (Jupiter)

### 5.1 LinkContactModal

**Location:** `jupiter/src/components/LeadForm/LinkContactModal/`
**Purpose:** Modal for selecting/linking a contact when creating or editing a lead.
**Added in:** jupiter PR #2865

Key features:
- Typeahead contact search
- Create new contact inline
- Select contact for lead association
- Used by LeadForm components

### 5.2 LinkClientDrawer

**Location:** `jupiter/src/components/Contacts/LinkClientDrawer/`
**Purpose:** Drawer for linking a client to a contact (reverse direction — from contact view).
**Added in:** jupiter PR #2699

Key features:
- Client search with typeahead
- Custom field support (vendorFields)
- Schema validation (Yup)
- Used on the contacts index page

### 5.3 ContactView / LinkClients

**Location:** `jupiter/src/components/Contacts/ContactView/`, `LinkClients/`
**Enhanced in:** jupiter PR #2699

Key props added:
- `onLinkClient` / `onUnlinkClient` handlers
- Custom field options, indexed fields, quota props
- Unlink confirmation modal

### 5.4 LeadForm Refactor

**In:** jupiter PR #2865

Major changes:
- `FormFields.tsx` simplified, broken into sub-components: `CustomFieldsPanel`, `FirstSection`, `LabelsFormSection`, `PipelineStageFields`
- `FormCustomFields` gets `hideAddCustomFieldButton` and `hideDefaultCustomField` props
- New schema validations in `schema.ts`
- New types for lead form props

### 5.5 Contact Form Schema

**Location:** `jupiter/src/components/Contacts/ContactForm/schema.ts`
- Yup validation for all contact fields
- Country-specific rules (PAN/Aadhaar for India only)
- Social profile URL validation via `validateSocialProfile()`
- State/StateCode per country
- GST State only for India

---

## 6. App Pages (Lydia)

### 6.1 Lead Create/Edit — Contact Integration

**Files:** `lydia/src/pages/app/[business]/leads/new.jsx`, `edit.jsx`
**PR:** lydia #4716

Changes:
- Refactored from MobX class components to React hooks
- Integrated `useContactManagement` hook for contact search, selection, creation
- New props: `contactOptions`, `onContactSearch`, `onContactSubmit`
- `primaryContact` field always set (even if null)
- `sanitizeLeadValues` updated
- i18n via `useLydiaTranslation`

### 6.2 Contacts Index — Client Linking

**File:** `lydia/src/pages/app/[business]/contacts/index.jsx`
**PR:** lydia #4438

Changes:
- "Link Client" action added to contacts table
- `LinkClientDrawer` integrated
- Client search + recent clients loading
- Link/unlink handlers
- `contact` renamed to `contactRecord` for clarity

### 6.3 Custom Fields Management

**File:** `lydia/src/pages/app/[business]/settings/CustomFieldLabels.js`
**PR:** lydia #4438

Changes:
- `CONTACTS_RELATIONS` added to indexed field categories
- Custom field label management for contact relations
- `IndexedFieldQuotaTable` updated with contacts relations label

---

## 7. Relation Types

The confirmed relation types for contact-client links:

```
owner
accountant
director
authorised_signatory
partner
manager
employee
other
```

Custom relation types are **not yet supported** — planned as a future enhancement.

---

## 8. Integration Architecture

### Contacts ↔ Clients (via `contactRelations`)

```
Contact ──── contactRelations ──── Client
              │
              ├── isPrimary (one per client)
              ├── role (free text)
              ├── department (free text)
              ├── vendorFields (custom fields)
              └── isRemoved (soft delete = unlink)
```

- **Link:** Create a `contactRelations` record
- **Unlink:** Soft delete (`isRemoved: true`) the relation record. Contact itself is not affected.
- **Primary:** Only one contact can be primary per client. Setting a new primary should clear the previous one.
- **Uniqueness:** One relation per contact-client pair

### Contacts ↔ Leads (embedded + referenced)

```
Lead
 ├── contact: { name, email, phone, ... }        ← embedded (legacy)
 ├── primaryContact: ObjectId → Contact           ← referenced (new)
 └── otherContacts: [ObjectId] → [Contact]        ← referenced (new)
```

- Embedded contact info is always present (backward compat)
- Referenced contacts link to the full contact record
- Primary contact is populated on fetch (firstName, lastName via serana hooks)

### Activity Tracking

Every contact operation is logged:
- **CREATE/UPDATE/DELETE/MERGE** — tracked with field-level diffs
- **LINKED/UNLINKED** — tracked with entity type + ID (clients, leads, vendorLeads, invoices)
- All activities are read-only via API — created internally by hooks

---

## 9. Data Flow Diagrams

### Creating a Contact
```
User submits form
  → validateContact() [firstName required, social URLs, PAN/Aadhaar]
  → checkContactUniqueKey() [unique per business]
  → addsCreator() [set to current user]
  → Store in MongoDB
  → Index in Elasticsearch (partial fields)
  → recordContactActivity(CREATE) [log all field values]
```

### Linking Contact to Client
```
User selects contact + relation type
  → validateContactRelations() [both entities exist, no duplicate pair]
  → validateVendorFieldPayload() [custom fields if any]
  → Create contactRelations document
  → recordContactLinkActivities(LINKED) [log entity type + ID]
  → If first contact linked → auto-mark as primary
```

### Unlinking Contact from Client
```
User confirms unlink
  → softRemove() [isRemoved: true, removedAt: now]
  → recordContactLinkActivities(UNLINKED)
  → Contact record itself unchanged
  → If was primary → primary is cleared (no auto-reassignment)
```

### Merging Contacts
```
User selects source → target
  → Set source: isMerged: true, mergedTo: targetId, mergedBy: userId
  → recordContactActivity(MERGE)
  → Source filtered from all queries
```

---

## 10. PR Reference Map

### Phase 1: Contacts ↔ Clients Integration

| Repo | PR | What |
|---|---|---|
| talos | [#701](https://github.com/refrens/talos/pull/701) | Add `vendorFields` to contactRelations schema |
| fence | [#592](https://github.com/refrens/fence/pull/592) | Register `CONTACTS_RELATIONS` as indexed field category |
| serana | [#4003](https://github.com/refrens/serana/pull/4003) | Unique relation validation + vendor field handling |
| jupiter | [#2699](https://github.com/refrens/jupiter/pull/2699) | LinkClientDrawer, ContactView enhancements, unlink UI |
| lydia | [#4438](https://github.com/refrens/lydia/pull/4438) | Contacts page — link/unlink clients, custom field labels |

### Phase 2: Contacts ↔ Leads Integration

| Repo | PR | What |
|---|---|---|
| talos | [#745](https://github.com/refrens/talos/pull/745) | Add `primaryContact` + `otherContacts` to lead schema |
| serana | [#4277](https://github.com/refrens/serana/pull/4277) | Populate primaryContact on lead fetch |
| jupiter | [#2865](https://github.com/refrens/jupiter/pull/2865) | LinkContactModal, LeadForm refactor, sub-components |
| lydia | [#4716](https://github.com/refrens/lydia/pull/4716) | Lead create/edit — contact search, selection, creation |

---

## 11. Prototype: Client Form — Linked Contacts Section

A working prototype has been built for adding a **Linked Contacts** section to the Client create/edit form.

**Live URL:** [refrens-contacts-prototype.vercel.app](https://refrens-contacts-prototype.vercel.app)

### What the prototype covers:
1. **Link Contact** — drawer with typeahead search across all org contacts, relation type dropdown, department field, primary checkbox
2. **Create Contact inline** — secondary drawer (contact creation form) accessible from the link drawer when contact doesn't exist
3. **View Linked Contacts** — react-table style table with columns: Contact, Email, Phone, Relation, Department, Primary, Linked On, More
4. **More menu (⋮)** — dropdown with "Mark as Primary" and "Unlink" actions
5. **Unlink** — confirmation modal, soft deletes the relation
6. **Mark as Primary** — switches primary badge, only one per client
7. **Accordion form** — all sections (Basic Details, Tax Info, Shipping, etc.) use collapsible accordion; Linked Contacts section is always expanded

### UI decisions in prototype:
- Linked Contacts is **non-accordion** (expanded by default) while all other form sections use accordion
- Table follows disco/react-table pattern — no pagination, no column hide/show, no CSV download (small dataset)
- More column uses `position: fixed` dropdown to escape table overflow containers
- Dropdown opens left-aligned to avoid overlapping the next row's trigger button
- First linked contact is auto-marked as Primary
- Unlinking the primary contact clears primary (no auto-reassignment)
- Duplicate link prevention with inline validation error

---

## 12. Upcoming Enhancements (Out of Scope)

1. Display linked contacts on the **Client detail/view page** (read-only table)
2. **Bulk linking** — select and link multiple contacts at once
3. **Custom relation types** — user-defined relation labels beyond the enum
4. **Permission controls** — role-based access for linking/unlinking
5. **Contact merge UI** — visual merge flow with field selection
6. **Lead other contacts** — full UI for managing `otherContacts` array on leads (schema exists, UI pending)
