---
title: "When Active Record Betrayed Me: Moving to Domain-Driven Design for Reusable npm Packages"
slug: "active-record-vs-ddd-reusable-npm-packages"
date: "2026-03-17"
description: "I built a reusable npm package using Active Record pattern — it worked great, until a new project needed Drizzle instead of Sequelize. Here's how that failure led me to DDD, and why separating domain logic from persistence made my packages genuinely reusable."
tags: ["Architecture", "System Design", "TypeScript", "DDD"]
readingTime: "7 min read"
---

One of my long-standing principles as a Solution Architect is that backend solutions should be **modular**. Business logic shouldn't live scattered across API routes and controllers — it should be encapsulated in reusable npm packages that any consuming API can import and use consistently.

For years, this worked well. Until it didn't.

## The Original Design: Active Record Packages

The packages I built followed the **Active Record pattern**. Each domain class — `Rental`, `Order`, `Agreement` — carried its own repository instances as static attributes. The class knew how to persist itself.

```typescript
// Inside the Rental domain class
protected static _Repo = new RentalRepository();
protected static _AgreementRepo = new AgreementRepository();
protected static _JointHirerRepo = new JointHirerRepository();
protected static _AgreementSignatureRepo = new AgreementSignatureRepository();
protected static _AgreementHistoryRepo = new AgreementHistoryRepository();
```

And the repository classes were built on top of Sequelize entities:

```typescript
import { RepositoryBase, IRepositoryBase } from '@tomei/general';
import { RentalModel } from '../../models/rental.entity';

export class RentalRepository
  extends RepositoryBase<RentalModel>
  implements IRepositoryBase<RentalModel>
{
  // All DB operations encapsulated here, tightly coupled to Sequelize
}
```

The appeal was obvious. At the API level, creating an order was a single line:

```typescript
await Order.create(payload); // DB operation already encapsulated inside
```

Clean. Concise. The developer consuming the package didn't need to think about persistence at all. It just worked.

## The Problem: "Reusable" Was a Lie

The design held up perfectly — right up until a new project came along that used **Drizzle ORM** instead of Sequelize.

Suddenly, the package I had marketed internally as reusable couldn't be reused. The `RentalRepository` extended `RepositoryBase<RentalModel>` which was built on Sequelize models. Drizzle has a completely different query API, different model definitions, different transaction handling. There was no clean way to swap one for the other.

The package wasn't reusable. It was a Sequelize package with domain logic bolted on top.

This is the core failure of Active Record when applied to shared libraries: **the persistence layer and the domain layer are fused together**. You can't take one without the other. If the consuming project doesn't use your ORM of choice, the package is useless to them.

## The Solution: Domain-Driven Design

When I dug into this problem — and got a nudge from GPT in the right direction — the answer was **Domain-Driven Design (DDD)**. Specifically, the principle that domain classes should only do domain things. They should know nothing about how or where they are stored.

Under DDD, the domain class holds business logic and enforces invariants:

```typescript
// Domain class - pure business logic, zero ORM knowledge
export class Order {
  private status: OrderStatus;
  private items: OrderItem[];

  confirm(): void {
    if (this.items.length === 0) {
      throw new Error('Cannot confirm an order with no items');
    }
    this.status = OrderStatus.CONFIRMED;
  }

  addItem(item: OrderItem): void {
    if (this.status !== OrderStatus.DRAFT) {
      throw new Error('Cannot add items to a confirmed order');
    }
    this.items.push(item);
  }
}
```

No `new SequelizeRepository()`. No `static _Repo`. The `Order` class has no idea whether your project uses Sequelize, Drizzle, Prisma, or a plain JSON file.

The repository lives at the **API level**, defined and injected by the consuming project:

```typescript
// At the API / service layer — the consumer decides the ORM
const order = new Order(payload);

order.confirm();           // business logic — lives in the package
await orderRepo.insert(order);  // DB insert — lives in the API
```

Two lines instead of one. But those two lines represent a clean separation of concerns that makes the package genuinely portable.

## What Changed in Practice

Before, a consuming API using Drizzle would look at the package and find it unusable. The Sequelize dependency would pull in the entire Sequelize ecosystem, and the repository classes would fail at runtime because the models weren't compatible.

After the DDD refactor, the package exports pure domain classes. The consuming project brings its own repository implementation:

```typescript
// Project A — uses Sequelize
import { Order } from '@tomei/rental-domain';
import { SequelizeOrderRepository } from './repositories/order.repository';

const orderRepo = new SequelizeOrderRepository();
const order = new Order(payload);
order.confirm();
await orderRepo.insert(order);
```

```typescript
// Project B — uses Drizzle
import { Order } from '@tomei/rental-domain';
import { DrizzleOrderRepository } from './repositories/order.repository';

const orderRepo = new DrizzleOrderRepository();
const order = new Order(payload);
order.confirm();
await orderRepo.insert(order);
```

Same domain package. Different persistence layer. Both work.

## The Tradeoff You Have to Accept

The DDD approach asks the developer consuming the package to do a little more work. They need to implement their own repository that satisfies an interface the package exposes:

```typescript
// The package exports this interface — consuming project implements it
export interface IOrderRepository {
  insert(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
  update(order: Order): Promise<void>;
}
```

This is a real cost. The one-liner `Order.create()` was genuinely ergonomic. Two lines with explicit repository calls is more verbose.

But the alternative — a "reusable" package that only works with one specific ORM — isn't actually reusable. It's just a library with aspirations.

## What I Carry Forward

The lesson wasn't that Active Record is bad. It's that Active Record is the wrong pattern for a layer that's meant to be portable. Active Record is excellent inside a single application where the ORM is a fixed, known quantity. It shines in Rails apps, in Laravel projects, in any context where the persistence layer is a settled decision.

But when you're building shared domain packages that need to survive across multiple projects — projects that may make different infrastructure choices — you need the domain to be sovereign. It can't depend on anything outside itself.

DDD gave me that. The domain package now truly knows only one thing: the business rules. Everything else — ORM, database dialect, connection pooling — is someone else's problem, solved at the level where it belongs.

The `@tomei/rental-domain` package works with Sequelize. It works with Drizzle. It'll work with whatever ORM the next project brings. That's what reusable actually means.
