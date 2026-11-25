# M-Entry Class Diagrams: Review and Final Documentation

**Task:** [ME-S1-09-06] Class Diagram Review and Documentation Finalization
**Project:** M-Entry Event Ticketing and Management System
**Author:** Moses Otieno Ochieng, Diploma in Software Engineering, Zetech University
**Diagrams:** CD-01, CD-02, CD-03, CD-04 (UML 2.x, editable draw.io)

## Review Status at a Glance

| Diagram | File | Approval status | Open items |
|---|---|---|---|
| CD-01 Core Domain | `diagrams/CD-01-Core-Domain-Class-Diagram.drawio` | Approved with conditions | R-09, R-10, R-11 (review) |
| CD-02 Ticketing and Payment | `diagrams/CD-02-Ticketing-and-Payment-Class-Diagram.drawio` | Conditionally approved | R-01, R-07, R-08, R-13 |
| CD-03 Scanner and Synchronization | `diagrams/CD-03-Scanner-and-Synchronization-Class-Diagram.drawio` | Conditionally approved | R-03, R-04, R-05 |
| CD-04 Application Services | `diagrams/CD-04-Application-Services-Class-Diagram.drawio` | Conditionally approved | R-02, R-03, R-05, R-13 |

No issue is rated Critical. Six are rated High. None of them requires a redesign; each is a missing operation or an unresolved decision that should be settled before the diagrams are frozen.

**Basis of this review.** The review was carried out against the specifications and source files of the four diagrams (their classes, attributes, operations, relationships, multiplicities and notes). Layout and rendering were checked by script for overlapping boxes and connectors passing through boxes. A visual check of each `.drawio` file in diagrams.net has not yet been done and is listed as R-25.

**Conventions used in this document.** *Confirmed* means stated in the approved requirements or the diagram specifications. *Recommendation* means a suggested change that has not been applied. *Unresolved* means the requirements do not settle the question and a decision is needed. No diagram has been changed by this review.

---

## 5.1 Introduction

Class diagrams record the static structure of M-Entry: which classes exist, what each is responsible for, and how they depend on one another. They sit between the requirements and the implementation. They give developers a shared vocabulary for the Flask backend and the scanner PWA, and they keep the database, the services and the sequence diagrams consistent with one another.

M-Entry is split into four class diagrams, not one, because a single diagram of 33 classes would not be readable. Each diagram has an owner area and a reference rule: a class is **defined** in exactly one diagram and only **referenced** (by name, with its source diagram) in the others.

## 5.2 Class Diagram Overview

| ID | Title | Primary responsibility | Defined here |
|---|---|---|---|
| CD-01 | Core Domain | Users, organizations, memberships, events, event staff, ticket tiers | 6 classes, 5 enumerations |
| CD-02 | Ticketing and Payment | Orders, items, payments, callbacks, tickets, refunds, payment gateway | 8 classes and interfaces, 5 enumerations |
| CD-03 | Scanner and Synchronization | Scanner devices, scan logs, local verification, local storage, sync | 5 classes, 1 enumeration |
| CD-04 | Application Services | Business coordination, payment processing, issuing, notifications, scanning, reporting, refunds | 14 classes and interfaces |

Total: 33 primary classes and interfaces, 11 enumerations. Repeated references are not counted twice.

**Shared visual conventions.** Each class box has three compartments (name, attributes, operations), uses `-` for attributes and `+` for operations, and has a light fill with a coloured border per category. External references are dashed grey boxes labelled with their source diagram. Interfaces carry the «interface» stereotype. Enumerations carry «enumeration» and a purple fill. Dependencies are dashed open arrows. Realizations are dashed lines with a hollow triangle.

**Repository layout (recommended).**

```text
docs/class-diagrams/
  README.md                                   (this document)
  diagrams/
    CD-01-Core-Domain-Class-Diagram.drawio
    CD-02-Ticketing-and-Payment-Class-Diagram.drawio
    CD-03-Scanner-and-Synchronization-Class-Diagram.drawio
    CD-04-Application-Services-Class-Diagram.drawio
  images/                                     (PNG or SVG exports from diagrams.net)
```

Export each diagram from diagrams.net (File, Export as, PNG or SVG) into `images/` and embed it like this once the files exist:

```markdown
![CD-01 Core Domain](images/CD-01-Core-Domain-Class-Diagram.png)
```

---

## 5.3 CD-01: Core Domain Class Diagram

**Purpose.** Defines who the people and organizations are, what events they run, and who may do what at an event.

### Classes

| Class | Type | Responsibility |
|---|---|---|
| User | Entity | Person identity, platform role, phone verification state. Not subclassed. |
| Organization | Entity | Owner of events. Holds `ownerUserId` and a typed `type`. |
| OrganizationMember | Entity (association class) | Links a User to an Organization with an `OrganizationMemberRole`. Key: `organizationId` + `userId`. |
| Event | Entity | Event details, lifecycle, `signingKeyId` and `publicKey` for ticket signatures. |
| EventStaff | Entity (association class) | Assigns a User to one Event with a `StaffRole`. Key: `eventId` + `userId`. |
| TicketTier | Entity | Tier definition, price in minor units, quantities (total, held, sold), sale window. |

**Enumerations.** PlatformRole (`user`, `admin`); OrganizationMemberRole (`owner`, `manager`); StaffRole (`usher`, `manager`); EventStatus (`draft`, `published`, `cancelled`, `completed`); OrganizationType (values not defined, taken from the approved schema).

### Relationships

| Source | Target | Type | Multiplicity | Label |
|---|---|---|---|---|
| User | Organization | Association | 1 to 0..* | owns |
| Organization | OrganizationMember | Association | 1 to 0..* | has members |
| User | OrganizationMember | Association | 1 to 0..* | participates through |
| Organization | Event | Association | 1 to 0..* | organizes |
| Event | TicketTier | Association | 1 to 0..* | contains |
| Event | EventStaff | Association | 1 to 0..* | assigns staff |
| User | EventStaff | Association | 1 to 0..* | serves through |

### Design decisions

- **Roles are modelled by relationship, not inheritance.** The platform role lives on User. Organizer authority comes from OrganizationMember. Usher authority comes from EventStaff. One person can be an attendee, an organizer for one organization and an usher at an unrelated event.
- **Event to TicketTier is a plain association.** It does not imply cascade deletion, because tiers carry financial history through order items. This supersedes the composition suggested in the earlier preparation document.
- **Inventory rules are split.** TicketTier holds the domain-level operations (`hold`, `release`, `commit`, `available`). Transaction control is in InventoryService (CD-04).

---

## 5.4 CD-02: Ticketing and Payment Class Diagram

**Purpose.** Defines how a purchase becomes a paid order and then individually signed tickets, and how refunds relate to it.

### Classes and interfaces

| Class | Type | Responsibility |
|---|---|---|
| Order | Entity | Purchase record. `buyerUserId` is optional (guest checkout). Totals, status, expiry, `idempotencyKey`. |
| OrderItem | Entity | One tier line: `tierId`, `quantity`, `unitPriceMinor`, `unitFeeMinor`. |
| Payment | Entity | One payment attempt: `checkoutRequestId`, optional `providerReceipt`, status, failure reason. |
| WebhookEvent | Entity | Stored provider callback. Unique on `provider` + `externalId`. |
| Ticket | Entity | One issued ticket unit: `serial`, `codeHash`, `signature`, `keyId`, status. |
| Refund | Entity | Refund request tied to an Order and optionally a Ticket. |
| PaymentGateway | Interface | Provider contract: `initiate`, `query`, `parseCallback`. |
| MpesaGateway | Class | M-Pesa Daraja implementation of PaymentGateway. No credentials as attributes. |

**Enumerations.** OrderStatus (`pending`, `paid`, `expired`, `failed`, `refunded`, `partially_refunded`); PaymentStatus (`initiated`, `pending`, `succeeded`, `failed`, `timed_out`, `reversed`); TicketStatus (`issued`, `scanned`, `void`, `refunded`); RefundStatus (`flagged`, `approved`, `rejected`, `processed`); PaymentProvider (values not defined; only M-Pesa is confirmed).

### Relationships

| Source | Target | Type | Multiplicity | Label |
|---|---|---|---|---|
| Order | OrderItem | Composition | 1 to 1..* | contains |
| Order | Payment | Association | 1 to 0..* | payment attempts |
| OrderItem | Ticket | Association | 1 to 0..* | generates |
| Order | Refund | Association | 1 to 0..* | refund requests |
| Ticket | Refund | Association | 0..1 (Ticket end) to 0..* (Refund end) | relates to |
| MpesaGateway | PaymentGateway | Realization | n/a | implements |
| Order, Ticket | Event (CD-01) | Dependency | n/a | eventId |
| OrderItem, Ticket | TicketTier (CD-01) | Dependency | n/a | tierId |
| Refund | User (CD-01) | Dependency | n/a | raisedBy, reviewedBy |

### Design decisions

- **Order to OrderItem is the only composition.** Items have no meaning outside their order. Financial records are still never hard-deleted.
- **Payment retries are modelled as many Payment rows per Order.** There is no separate "attempt" class.
- **Tickets are not created with the order.** OrderItem to Ticket is `1 to 0..*` and a constraint note states that after successful payment the number of tickets for an item equals its quantity.
- **WebhookEvent has no relationship to Payment.** The diagram says so in a note. Callback processing correlates events with payment attempts in application logic. This avoids inventing a foreign key that the approved schema does not have.
- **M-Pesa is reached only through the PaymentGateway abstraction.** No other provider is introduced.
- **Refund approval and payment reversal are separate stages.** `approve` and `markProcessed` are separate operations on Refund.

---

## 5.5 CD-03: Scanner and Synchronization Class Diagram

**Purpose.** Defines the offline-capable scanning design and its boundary with the backend.

### Classes

| Class | Location | Responsibility |
|---|---|---|
| ScannerDevice | Backend domain | Registered scanner for one user and one event; records last sync. |
| ScanLog | Backend domain | Server-side record of one scan. `ticketId` is optional. Unique on `deviceId` + `clientSeq`. |
| TicketVerifier | Client PWA | Local Ed25519 signature check, event check, manifest check, local duplicate check. |
| LocalScanStore | Client PWA | IndexedDB persistence of manifests and pending scans; local cleanup. |
| SyncClient | Client PWA | Manifest and delta download, batch upload, connectivity, clock drift check. |

**Enumeration.** ScanResult (`valid`, `duplicate`, `invalid_signature`, `unknown_ticket`, `wrong_event`, `void_ticket`).

### Relationships

| Source | Target | Type | Multiplicity | Label |
|---|---|---|---|---|
| Event (CD-01) | ScannerDevice | Association | 1 to 0..* | has registered scanners |
| User (CD-01) | ScannerDevice | Association | 1 to 0..* | registers |
| ScannerDevice | ScanLog | Association | 1 to 0..* | records scans |
| Ticket (CD-02) | ScanLog | Association | 0..1 (Ticket end) to 0..* (ScanLog end) | records scan history |
| TicketVerifier | LocalScanStore | Dependency | n/a | reads local data |
| SyncClient | LocalScanStore | Dependency | n/a | reads and updates |
| SyncClient | ScannerDevice | Dependency (across the API boundary) | n/a | synchronizes for |
| SyncClient | ScanLog | Dependency (across the API boundary) | n/a | submits scan batches |

### Design decisions

- **Two packages.** Backend domain classes and client PWA classes are in separate package boundaries. Backend classes do not depend on client classes. The client reaches the backend through the synchronization API.
- **ScanLog may have no ticket.** An unreadable or unknown QR code still produces a scan record, so Ticket to ScanLog is optional on the Ticket side.
- **Six rules appear as notes on the diagram:** local signature verification with the private key kept on the server, saving locally before a scan is considered queued, duplicates across disconnected devices, earliest-valid-scan conflict resolution, clock drift, and local cleanup on logout or event end.
- **No global duplicate prevention is claimed.** Two disconnected scanners can both accept the same ticket. The design detects and flags this after synchronization.

---

## 5.6 CD-04: Application Services Class Diagram

**Purpose.** Defines the service layer that coordinates domain objects, payment and messaging integrations, and the scanner backend.

### Package organization

| Package | Members |
|---|---|
| A: Identity and Event Management | AuthService, EventService |
| B: Ticketing and Payment Coordination | InventoryService, CheckoutService, PaymentService, TicketIssuer, SignatureService |
| C: Communication | NotificationService, MessageProvider (interface), WhatsAppProvider, SmsProvider |
| D: Scanner Backend Operations | ScanService |
| E: Reporting and Refunds | ReportingService, RefundService |

### Service responsibilities

| Service | Operations | Responsibility |
|---|---|---|
| AuthService | `register`, `login`, `issueOtp`, `verifyOtp`, `issueSession` | Registration, authentication, phone OTP, sessions |
| EventService | `createEvent`, `editEvent`, `publishEvent`, `cancelEvent`, `completeEvent`, `manageTicketTiers`, `assignUsher` | Event lifecycle, tier configuration, staff assignment |
| InventoryService | `hold`, `release`, `commit`, `sweepExpired` | Transactional inventory holds and release |
| CheckoutService | `createOrder`, `startPayment` | Guest or registered checkout up to payment start |
| PaymentService | `handleCallback`, `reconcileDaily` | Idempotent callback processing and reconciliation |
| SignatureService | `sign`, `verify` | Ed25519 signing, server-side verification |
| TicketIssuer | `issueForOrder` | One ticket per purchased unit after confirmed payment |
| NotificationService | `sendTicket`, `resend` | WhatsApp first, SMS fallback, outcome recording |
| ScanService | `buildManifest`, `ingestScans` | Manifests (full and delta), batch ingestion, conflict resolution |
| ReportingService | `salesSummary`, `weeklyReport`, `exportCsv` | Sales reports and CSV export |
| RefundService | `flag`, `review` | Refund intake, administrator review, coordination of voiding and reversal |
| MessageProvider | `send` | Interface for message channels |
| WhatsAppProvider, SmsProvider | `send` | Realize MessageProvider |

### Dependencies

| From | To | Meaning |
|---|---|---|
| CheckoutService | InventoryService | Requests and releases stock holds |
| CheckoutService | PaymentService | Starts payment |
| PaymentService | TicketIssuer | Triggers issuing after confirmed payment |
| TicketIssuer | SignatureService | Requests signatures |
| PaymentService | PaymentGateway (CD-02) | Provider abstraction |
| NotificationService | MessageProvider | Channel abstraction |
| NotificationService | NotificationLog | Records attempts |
| WhatsAppProvider, SmsProvider | MessageProvider | Realization |
| All services | CD-01, CD-02, CD-03 packages | "uses" arrows to external reference packages, labelled with class names |

**Design decisions.**

- There are no circular service dependencies. The chain is CheckoutService to InventoryService and PaymentService, PaymentService to TicketIssuer, TicketIssuer to SignatureService.
- Service attributes are not invented. Each service shows an empty attribute compartment.
- Domain classes are shown as external references, in three dashed grey packages. Arrows end at the package and the label names the classes. This was chosen for readability, as the specification allowed.
- Scanner client classes do not appear as backend dependencies. A note states that they run in the PWA and reach ScanService through the API.

---

## 5.7 Cross-Diagram Relationships

### How the diagrams fit together

```text
CD-01 Core Domain
   Event, TicketTier, User
        |  referenced by
        v
CD-02 Ticketing and Payment ---- Ticket, Order, Payment, Refund, PaymentGateway
        |  referenced by                       |
        v                                      v
CD-03 Scanner and Sync            CD-04 Application Services
   ScannerDevice, ScanLog              coordinates all three through service dependencies
```

**Purchase flow.** CheckoutService creates an Order and OrderItems (CD-02) for a TicketTier (CD-01), asks InventoryService to hold stock, then asks PaymentService to start payment through PaymentGateway. A verified callback is stored as a WebhookEvent, updates Payment and Order, and triggers TicketIssuer, which asks SignatureService to sign each Ticket. NotificationService delivers the tickets.

**Scanning flow.** ScanService builds a manifest for an Event (CD-01) from its Tickets (CD-02). SyncClient downloads it into LocalScanStore. TicketVerifier validates scans locally. SyncClient uploads batches and ScanService persists ScanLog records (CD-03) and resolves conflicts.

### Cross-diagram consistency matrix

Key: **D** = defined (owner), **R** = referenced as an external element, **-** = not present.

| Element | CD-01 | CD-02 | CD-03 | CD-04 | Result |
|---|---|---|---|---|---|
| User | D | R (Refund) | R (registers) | R | Consistent |
| Organization | D | - | - | R | Consistent |
| OrganizationMember | D | - | - | - | Consistent |
| Event | D | R (Order, Ticket) | R (ScannerDevice) | R | Consistent |
| EventStaff | D | - | - | - | Consistent |
| TicketTier | D | R (OrderItem, Ticket) | - | R | Consistent |
| Order | - | D | - | R | Consistent |
| OrderItem | - | D | - | R | Consistent |
| Payment | - | D | - | R | Consistent |
| WebhookEvent | - | D | - | - | Consistent |
| Ticket | - | D | R (ScanLog) | R | Consistent |
| Refund | - | D | - | R | Consistent |
| PaymentGateway | - | D | - | R | Consistent |
| MpesaGateway | - | D | - | Not drawn (mentioned in a note) | Consistent |
| ScannerDevice | - | - | D | R | Consistent |
| ScanLog | - | - | D | R | Consistent |
| TicketVerifier, LocalScanStore, SyncClient | - | - | D | Not drawn (mentioned in a note) | Consistent |
| NotificationLog | - | - | - | R (external schema entity) | **Not defined anywhere** (R-06) |
| TicketStatus | - | D | used by ScanResult mapping | - | Consistent |
| RefundStatus | - | D | - | Parameter type | See R-14 |
| ScanResult | - | - | D | Return type (via note) | Consistent |

### Attribute and type consistency spot checks

| Check | Result |
|---|---|
| `id` as primary key name and UUID type across entities | Consistent |
| Foreign keys named `<entity>Id` (`eventId`, `tierId`, `orderId`, `orderItemId`, `deviceId`, `userId`) | Consistent, with `buyerUserId`, `raisedBy`, `reviewedBy`, `invitedBy` as role-named user references |
| Money as integer minor units (`priceMinor`, `subtotalMinor`, `amountMinor`) | Consistent, but currency is missing outside TicketTier (R-07) |
| `provider` type | **Inconsistent**: `String` on WebhookEvent, `PaymentProvider` on Payment (R-08) |
| Same operation name and signature repeated across diagrams (`SyncResult`, `PaymentGateway.initiate`) | Consistent |

---

## 5.8 Design Decisions and Constraints

### Confirmed decisions

| ID | Decision |
|---|---|
| D-01 | User has a platform role (`user` or `admin`). Organizer, Attendee, Usher and Admin are not subclasses. |
| D-02 | Organization authority comes from OrganizationMember; event authority comes from EventStaff. |
| D-03 | Guest checkout is supported: `Order.buyerUserId` is optional. |
| D-04 | Money is stored as integer minor units. Financial records are never hard-deleted. |
| D-05 | An Order has one or more OrderItems (composition) and zero or more Payments. |
| D-06 | Tickets are created after confirmed payment, one per purchased unit. |
| D-07 | WebhookEvent is unique on provider and external ID and has no direct relationship to Payment. |
| D-08 | M-Pesa is reached only through PaymentGateway. |
| D-09 | The private signing key stays on the server. Scanners receive only the public key. |
| D-10 | Offline scans are stored in IndexedDB and synchronized in batches. Synchronization is idempotent. |
| D-11 | Conflicts across disconnected devices are resolved at synchronization. The earliest valid scan wins. The design does not claim to prevent them. |
| D-12 | Services depend on domain classes; domain classes never depend on services. There are no circular service dependencies. |
| D-13 | Scanner client components stay separate from backend services. |

### Technical constraints

- Offline verification can only see data in the downloaded manifest and the local scan history. It cannot see scans on other devices.
- Hold duration for an order is 10 minutes.
- A QR payload is signed with Ed25519. The verifier needs the event's public key.
- ScanLog records are meant to be audit records and should not be rewritten (see R-04).

### Recommendations (not applied)

See Section 5.9, items rated High and Medium.

---

## 5.9 Review Findings

Priority scale: **Critical** blocks approval. **High** should be decided before the diagrams are frozen. **Medium** should be resolved before implementation. **Low** is a clarification.

### Issue register

| ID | Diagram | Problem | Technical impact | Recommended correction | Priority |
|---|---|---|---|---|---|
| R-01 | CD-02, CD-04 | The requirement says callbacks are verified, but PaymentGateway has only `initiate`, `query`, `parseCallback`. No operation expresses verification. | Verification could be forgotten or duplicated in PaymentService. | Decide whether `parseCallback` includes verification, or add a verification operation to PaymentGateway and MpesaGateway. | High |
| R-02 | CD-04 | CheckoutService depends on PaymentService and `startPayment` returns Payment, but PaymentService has only `handleCallback` and `reconcileDaily`. | The dependency has no operation to call; the initiation responsibility is unowned. | Add an initiation operation to PaymentService, or document that CheckoutService calls PaymentGateway directly (and draw that dependency). Mark for decision. | High |
| R-03 | CD-03, CD-04 | The scanner verifies the manifest (`verifyManifest`), but `SignatureService.sign` accepts only a Ticket and `buildManifest` returns an unsigned Object. How the public key reaches scanners is also not shown. | The manifest authenticity requirement cannot be traced to a signing operation. | Generalize signing to a payload, or add a manifest signing responsibility. Document that the public key is delivered with the manifest or event data. | High |
| R-04 | CD-03 | ScanLog should be immutable, but earliest-scan-wins can require changing an already stored record (a later scan synced first, then an earlier scan arrives). The same applies to Ticket status. | Conflicting with the immutability rule or with the conflict rule. | Decide how resolution is stored: controlled update of resolution fields by ScanService only, or a separate resolution record. Original scan fields stay unchanged. | High |
| R-05 | CD-03, CD-04 | The scanner device is registered, but no service operation registers a ScannerDevice. | Registration has no owner in the service layer. | Add a registration operation to ScanService (or AuthService), or document the owning service. | High |
| R-06 | CD-04 | NotificationLog is used by NotificationService and exists in the approved schema (`notification_logs`) but is not defined in any diagram. | The only database entity without a class. | Define NotificationLog as a CD-04 or CD-02 entity, or document why it is excluded. | Medium |
| R-07 | CD-02 | Order, Payment and Refund have minor-unit amounts but no currency code, while TicketTier has `currency`. The rule requires a currency code with minor units. | Ambiguous currency on financial records. | Confirm against the schema. Add `currency` where the schema has it, or document that it is derived from the tier. | Medium |
| R-08 | CD-02 | `WebhookEvent.provider` is `String`; `Payment.provider` is `PaymentProvider`. | Inconsistent typing of the same concept. | Use one type for both, or document why they differ. | Medium |
| R-09 | CD-01 | StaffRole includes `manager`, but the authorization model defines an usher as the only event-level role, and organizers are organization members. | An unexplained event-level role may duplicate organization authority. | Confirm whether event-level `manager` exists and what it may do. | Medium |
| R-10 | CD-01 | The organization owner appears three ways: `Organization.ownerUserId`, the `owns` association, and the `owner` member role. | Possible disagreement between them. | Document which is authoritative and the invariant between them. | Medium |
| R-11 | CD-01, CD-04 | Operations overlap between entities and services (`Event.assignUsher`, `Event.addTier`, `Organization.createEvent`, `Organization.addMember`, `User.canManage` versus EventService and AuthService). | Unclear whether rules live in the entity or the service. | Record the rule: entities hold state changes and invariants; services own authorization, transactions and persistence. Review each overlapping operation. | Medium |
| R-12 | CD-04 | There is no explicit authorization operation. Permission rules are described across EventService, ReportingService and RefundService. | Authorization could be implemented inconsistently. | Document one authorization path (AuthService), or add an explicit check operation. | Medium |
| R-13 | CD-02, CD-04 | RefundService coordinates payment reversal and PaymentStatus has `reversed`, but PaymentGateway has no reversal operation. | The reversal path is not representable. | Decide whether M-Pesa reversal is in scope. If so, add an operation to PaymentGateway; if not, document manual settlement. | High |
| R-14 | CD-04 | `RefundService.review` takes `decision: RefundStatus`, which allows any status, including `flagged` and `processed`. | Invalid decisions are type-legal. | Document that only `approved` and `rejected` are valid, or use a narrower type. | Low |
| R-15 | CD-04 | Nothing states who triggers ticket delivery after TicketIssuer runs. | A required step has no owner. | Decide between PaymentService, TicketIssuer or an event handler; draw the chosen dependency. | Medium |
| R-16 | CD-02 | `parseCallback` returns `PaymentCallbackResult`, which is not in any supporting types list. | Undefined return type. | Add it to the supporting types list in CD-02 and CD-04 documentation. | Low |
| R-17 | CD-02, CD-03 | `Order.buyerUserId` and `ScanLog.eventId` are attributes without a drawn relationship to User and Event. | Minor gaps in the relationship set. | Add optional references or document them as attribute-only. | Low |
| R-18 | CD-02 | The operation name `void()` on Ticket collides with the language keyword in many languages. | Awkward or invalid code generation. | Rename in implementation (for example `markVoid`), keeping the diagram as approved until reviewed. | Low |
| R-19 | All | Money is typed `Integer`. Minor-unit totals may need 64-bit integers. | Possible overflow in implementation. | Use a 64-bit type in code and in the schema. | Low |
| R-20 | CD-04 | Dependencies end at external packages with class names in labels, not at individual classes. | Less precise than per-class arrows. | Accept for readability, or split into per-class arrows in a later revision. | Low |
| R-21 | CD-03 | Clock drift is detected (`detectClockDrift` returns Boolean) but nothing states where it is reported. | A requirement is only partly represented. | Decide where the warning is shown or recorded. | Low |
| R-22 | CD-03 | Local data is cleared by `clearEventData` and `clearAllData`, but nothing assigns who calls them on logout or event completion. | Cleanup could be missed. | Document the trigger owner (client logic or SyncClient). | Low |
| R-23 | CD-02 | `OrderStatus` has `refunded` and `partially_refunded`; `PaymentStatus` has both `initiated` and `pending`. | The distinction is not explained. | Document the intended difference. | Low |
| R-24 | Preparation document | ME-S1-09-01 uses different names (for example `userId`, `phoneNumber`, `OrganizationRole`) and suggested composition for Event to TicketTier. The diagrams follow the later specifications. | The two documents disagree. | Treat the diagrams and this document as authoritative and update ME-S1-09-01. | Medium |
| R-25 | All | The `.drawio` files were checked by script, not by visual inspection in diagrams.net. | Layout defects may remain. | Open each file, check labels and routing, then export PNG or SVG. | Medium |

### Priority summary

| Priority | Count | Issues |
|---|---|---|
| Critical | 0 | None |
| High | 6 | R-01, R-02, R-03, R-04, R-05, R-13 |
| Medium | 10 | R-06, R-07, R-08, R-09, R-10, R-11, R-12, R-15, R-24, R-25 |
| Low | 9 | R-14, R-16, R-17, R-18, R-19, R-20, R-21, R-22, R-23 |

Total: 25 issues.

### Verification of the specific requirement areas

**Authorization (Section 2.4)**

| Requirement | Status |
|---|---|
| Platform role is `user` or `admin` | Represented (PlatformRole) |
| Organizer is a user with an organization membership | Represented (OrganizationMember). `manager` in StaffRole needs review (R-09). |
| Usher is a user assigned to an event through EventStaff | Represented |
| Guest attendee can buy without an account | Represented (`buyerUserId` optional) |
| Administrator has platform-level oversight | Represented through `platformRole`; no explicit authorization operation (R-12) |
| No Organizer, Attendee, Usher, Admin subclasses | Confirmed: none exist |

**Offline scanner (Section 2.5)**

| Requirement | Status |
|---|---|
| Installable Android Chrome PWA | Represented (client package label) |
| Manifest downloaded before offline operation | Represented (`downloadManifest`, `buildManifest`) |
| Local Ed25519 verification | Represented (`verifySignature`, note) |
| Private key on server only | Represented (notes, SignatureService) |
| Public keys distributed to scanners | Partly: `publicKey` attributes exist, delivery not shown (R-03) |
| IndexedDB for scan records | Represented (LocalScanStore) |
| Batch upload | Represented (`uploadBatch`, `ingestScans`) |
| Delta manifests | Represented (`downloadDeltaManifest`, `sinceVersion`) |
| Clock drift detected and reported | Partly: detection only (R-21) |
| Local data cleared | Partly: operations exist, trigger not owned (R-22) |
| Duplicates resolved at sync | Represented, with a design tension (R-04) |
| No claim of global duplicate prevention | Confirmed (notes) |

**Payment and issuance (Section 2.6)**

| Requirement | Status |
|---|---|
| Order has multiple items | Represented (composition 1 to 1..*) |
| Payment retries | Represented (1 to 0..*) |
| Verified callbacks and reconciliation | Partly: reconciliation represented; verification unclear (R-01) |
| Tickets after confirmed payment | Represented (0..* multiplicity, note, TicketIssuer) |
| Ticket quantity equals purchased quantity | Represented (constraint note) |
| Callback idempotency | Represented (WebhookEvent uniqueness, note) |
| WebhookEvent independently identifiable | Represented |
| M-Pesa through PaymentGateway | Represented |
| No unsupported providers or payment states | Confirmed (PaymentProvider left empty) |

### Open decisions

The decisions that need an answer before approval are R-01, R-02, R-03, R-04, R-05 and R-13. Medium items R-06 to R-12 and R-15 should be settled before implementation begins.

---

## 5.10 Conclusion

The four diagrams describe one consistent architecture. The core domain, ticketing and payment, scanner, and service responsibilities are separated, shared classes are defined once and referenced elsewhere, and the authorization model is preserved without subclassing User. The payment and offline scanner requirements are represented, and the diagrams make no claim that disconnected scanners can prevent duplicate acceptance.

The review found no critical problems. It found six High items that are gaps in operations or unresolved decisions around callback verification, payment initiation, manifest signing, scan-record immutability, device registration, and refund reversal. None of these requires redesigning the system. Once they are decided and the diagrams are checked visually in diagrams.net, the set is ready for approval and for inclusion in the repository.

---

## Final Approval Checklist

| Check | CD-01 | CD-02 | CD-03 | CD-04 |
|---|---|---|---|---|
| Class names consistent across diagrams | Yes | Yes | Yes | Yes |
| Shared classes referenced, not duplicated | Yes | Yes | Yes | Yes |
| Interfaces and realizations correct | n/a | Yes | n/a | Yes |
| Multiplicities checked | Yes | Yes | Yes | n/a |
| Composition and aggregation justified | Yes | Yes | n/a | n/a |
| Dependencies drawn dashed | n/a | Yes | Yes | Yes |
| Authorization model preserved | Yes | n/a | n/a | Open (R-12) |
| Offline scanner requirements represented | n/a | n/a | Partly (R-03, R-04, R-21, R-22) | Partly (R-03, R-05) |
| Payment and issuance requirements represented | n/a | Partly (R-01, R-07, R-13) | n/a | Partly (R-02, R-15) |
| No invented classes or states | Yes | Yes | Yes | Yes |
| Visual check in diagrams.net | Pending | Pending | Pending | Pending |
| **Approval status** | **Approved with conditions** | **Conditionally approved** | **Conditionally approved** | **Conditionally approved** |

### Suggested next steps

1. Decide the six High items (R-01 to R-05 and R-13).
2. Open each `.drawio` file in diagrams.net and confirm layout and labels (R-25).
3. Export PNG or SVG files into `images/` and embed them in this document.
4. Update ME-S1-09-01 so its names agree with the diagrams (R-24).
5. Commit the diagrams and this document under `docs/class-diagrams/`.
