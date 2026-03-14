---
layout: "../../layouts/BlogPost.astro"
title: "From One Big Submit to Step-by-Step: Redesigning a Lab Order System with Draft Persistence"
slug: "multi-step-form-draft-persistence-architecture"
date: "2026-02-28"
description: "How I redesigned a gemological lab management system's order creation flow from a monolithic single-page form into a resilient multi-step wizard with per-step API inserts and draft recovery — and why the real win wasn't UX, it was data integrity."
tags: ["Architecture", "System Design", "React", "API Design"]
readingTime: "9 min read"
---

Every engineering decision has a forcing function. For this one, it was a frustrated front-desk staff member who had spent 12 minutes filling in a job order form — customer details, item descriptions, service selections, certificate preferences, terms acknowledgement — only for a network hiccup to wipe the whole thing. She had to start over. From scratch.

That was the conversation that prompted a proper redesign of the order creation flow in **Gemlab Admin**, an internal lab management system for a gemological certification laboratory in Malaysia.

## The Original Architecture

The original flow was a classic single-page form. All fields — customer, items, services, certificates — lived on one screen. When staff clicked *Submit*, a single `POST /orders` API call fired, carrying the entire payload. The backend would then execute a series of inserts across multiple tables: customers, orders, order_items, item_services, item_certificates.

```
Client                         Server
  │                               │
  │── POST /orders ─────────────►│
  │   { customer, items[],        │
  │     services[], certs[] }     │  INSERT customers
  │                               │  INSERT orders
  │                               │  INSERT order_items (loop)
  │                               │  INSERT item_services (loop)
  │                               │  INSERT item_certificates (loop)
  │◄── 200 OK / 500 Error ───────│
  │                               │
```

This worked. Until it didn't.

### What Went Wrong

Three categories of failure surfaced over time:

**1. Any failure meant total loss.** A timeout, a validation error on item 3 of 5, a network drop — any of these rolled everything back. The user faced a blank form. No recovery, no draft, no partial save.

**2. The API took longer as orders grew.** A job order with 4 items, each with its own service and certificate type, meant a single HTTP request had to wait for 12+ sequential inserts before responding. On slow connections — common in the lab environment — this translated to visible spinner time.

**3. Validation feedback was buried.** Because all data arrived at once, validation errors for deeply nested fields (e.g. "item 3 has no service selected") came back in a batch response. The frontend had to map those back to the right fields, and the UX for communicating them was awkward.

## The Redesigned Architecture

The redesign centred on one principle: **each step of the form should have its own API endpoint, and each endpoint should persist immediately.**

```
Step 1 → POST /orders/draft          → { orderId: "GLJ2600012" }
Step 2 → POST /orders/:id/items      → { itemIds: [1, 2] }
Step 3 → PATCH /orders/:id/services  → { updated: true }
Step 4 → PATCH /orders/:id/certs     → { updated: true }
Step 5 → (review — read only, no API call)
Step 6 → POST /orders/:id/confirm    → { status: "in_custody" }
Step 7 → POST /orders/:id/deposit    → { depositId: 44 }
```

The order is created as a `draft` record at Step 1 — the moment the customer's details are saved. Every subsequent step updates that draft. The order only transitions to `confirmed → in_custody` when staff completes the digital acknowledgement at Step 6.

### The Status Machine

Status transitions are now explicit and enforced at the API level:

```
draft
  │
  ▼
confirmed        (Step 5 confirm clicked)
  │
  ▼
in_custody       (Step 6 acknowledgement signed)
  │
  ▼
in_grading       (assigned to gemologist)
  │
  ▼
grading_completed
  │
  ▼
awaiting_payment
  │
  ▼
paid
  │
  ▼
certificate_generated
  │
  ▼
ready_for_collection
  │
  ▼
collected → closed
```

Previously, `draft` didn't exist. Orders jumped straight from nothing to `confirmed`. That meant any partial completion was invisible to the system.

### Draft Recovery

The real operational win is draft recovery. When a session drops mid-wizard — whether due to a browser crash, accidental navigation, or network failure — the order already exists in the database as a `draft`. The system can now:

1. Show a *"You have an incomplete order"* banner on the dashboard.
2. Let staff click *Resume* — loading the draft and skipping any already-completed steps.
3. Highlight exactly where the wizard needs to continue from.

```
Dashboard → "Resume Draft: GLJ2600012 — Stopped at Step 4 (Certificate)"
                                          ↓
                        Wizard opens at Step 4, Steps 1–3 already ticked ✓
```

This is a fundamentally different recovery model from the previous one (which was: start over).

## The Frontend Wizard

On the React side, the wizard is built as a stateful shell (`CreateOrderWizard.jsx`) that holds the shared `formData` object and a `currentStep` index. Each step is an isolated component that receives `formData`, `updateFormData`, `next`, and `prev` as props.

```jsx
// CreateOrderWizard.jsx (simplified)
const [current, setCurrent] = useState(resumeFromStep ?? 0);
const [formData, setFormData] = useState(draftData ?? INITIAL_STATE);

const updateFormData = (patch) =>
  setFormData(prev => ({ ...prev, ...patch }));
```

When a step completes, it calls `updateFormData` with its slice of state, then fires its own API call. Only on success does it call `next()`. If the API call fails, the user stays on the same step — but the data already entered is still in local state, so nothing is lost for the session.

### Component Breakdown

| Component | Responsibility |
|---|---|
| `CreateOrderWizard.jsx` | Stepper shell, shared state, step orchestration |
| `Step1Customer.jsx` | Individual/Company toggle, phone-based auto-lookup |
| `Step2Items.jsx` | Dynamic `Form.List` for multi-item submission |
| `Step3Service.jsx` | Per-item service selection with report type inference |
| `Step4Certificate.jsx` | Skips automatically for Verbal Test Report items |
| `Step5Review.jsx` | Read-only summary before commitment |
| `Step6Acknowledgement.jsx` | T&C display + digital staff signature |
| `Step7Deposit.jsx` | Optional; can be deferred to order detail page |

The reusable sub-components — `ItemForm`, `ServiceSelector`, `CertSelector`, `SignatureCanvas` — are kept in a `components/` folder and can be imported independently. This matters for the gemologist-facing grading screens, which reuse `ServiceSelector` in a different context.

## The Paperless Decision

A parallel decision made during this redesign: eliminate the printed Good Received Note entirely.

Previously, the flow was: staff fills form → system confirms → staff prints a PDF → hands physical receipt to customer. The customer keeps this as proof of submission and presents it at collection.

The redesigned flow: staff completes Step 6 (digital acknowledgement) → staff signs digitally on screen → system generates a PDF stored server-side → customer receives an email with their Job Order No.

This is legally valid under Malaysia's **Electronic Commerce Act 2006**, which recognises electronic records and digital signatures as equivalent to physical ones. The Job Order No. itself becomes the claim token — no paper required at any point in the workflow.

The audit trail is actually *better* in the paperless model: every state transition is timestamped and tied to a specific staff member via session, the signature image is stored immutably alongside the PDF, and the full record is queryable. A physical receipt can be lost, forged, or damaged. A server-side record cannot.

## What This Architecture Enables

Beyond the immediate UX improvement, the multi-step + draft model unlocks capabilities that weren't possible before:

**Analytics on abandonment.** You can now query: at which step do most drafts get abandoned? If 40% of drafts never make it past Step 3 (Service Selection), that's a signal the service catalogue is confusing or too granular.

**Partial data is useful data.** A draft with just customer and item information — even without a service selected — is still a customer record and an item record in the database. The lab can see a customer came in, even if the order was never completed.

**Progressive validation.** Errors surface at the step they belong to, not as a batch at the end. A missing photo in Item 2 stops you at Step 2, with clear context — not after you've already filled in everything else.

**Recovery without starting over.** The staff experience goes from "I lost everything" to "I can pick up where I left off." That's the one that matters most to the people who use this daily.

---

The underlying principle here isn't new — it's the same logic behind two-phase commit, event sourcing, and saga patterns in distributed systems. Break a large, all-or-nothing operation into smaller, individually committed units. Accept partial state. Design for recovery. Build in observability at each seam.

What made this satisfying to design was seeing those patterns — usually discussed in the context of microservices and distributed databases — apply just as cleanly to a wizard form in a React application.

The user who lost her 12 minutes of work was the right forcing function. Sometimes the most important architectural improvements start with one person, one frustrating afternoon, and a blank form.
