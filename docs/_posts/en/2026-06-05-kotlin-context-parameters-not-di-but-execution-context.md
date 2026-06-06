---
layout: post
title: "Kotlin context parameters: not DI, but execution context"
lang: en
ref: kotlin-context-parameters-practical
date: 2026-06-05
tags: [ kotlin, ddd, testing, software-engineering ]
permalink: /en/kotlin-context-parameters-not-di-but-execution-context/
---

When I first saw `context(...)` next to a function, the obvious question was:

> did Kotlin get another way to do dependency injection?

After playing with it for a while, I do not see it that way.

`context parameters` do not create objects. They do not manage dependency lifecycles. They do not replace Spring, Koin or Dagger.

They describe something simpler:

> this function can run only when the required context exists at the call site.

And that condition is checked by the compiler.

That is the important part. Not runtime. Not a magic provider. Not a global singleton. The compiler.

## The simplest example

Take a clock.

```kotlin
interface ClockProvider {
    fun now(): Instant
}
```

If a function needs the current time, we can pass the clock as a normal parameter:

```kotlin
fun confirmedAt(clock: ClockProvider): Instant =
    clock.now()
```

We can also keep it in the constructor, like in classic dependency injection:

```kotlin
class OrderService(
    private val clock: ClockProvider,
) {
    fun confirmedAt(): Instant =
        clock.now()
}
```

Or we can say that the clock is part of the execution context of this function:

```kotlin
context(clock: ClockProvider)
fun confirmedAt(): Instant =
    clock.now()
```

The call site looks like this:

```kotlin
val clock = FixedClockProvider(
    Instant.parse("2026-06-05T10:15:30Z"),
)

context(clock) {
    confirmedAt()
}
```

Without `context(clock)`, this function should not compile.

That is the key difference from a service locator. The dependency is still visible in the function signature, but we do not have to pass it as a regular argument every time.

## Parameter, constructor or context?

I would not treat `context parameters` as a new default answer for everything.

For me the split looks like this:

```text
normal parameters     -> operation input
constructor injection -> stable object dependencies
context parameters    -> execution context
```

`orderId`, `amount`, `email`, `newAddress` — these are normal parameters.

`OrderRepository`, `PaymentGateway`, `HttpClient` — these often belong in the constructor of a use case or service.

`ClockProvider`, `UserContext`, `DomainEvents`, `Transaction`, `TenantContext`, `CorrelationId` — these are good candidates for execution context.

Not always, of course. But this boundary feels much healthier than: "great, now everything goes into `context(...)`".

## Example repository

For this article I created a small DDD / hexagonal example repository:

```text
src/main/kotlin/io/github/camilyed/contextparameters
├── domain
├── application
└── adapter
```

There is no Spring here. On purpose.

I wanted the example to show the language feature and the design decision, not framework integration.

The domain contains one aggregate: `Order`.

An order can be a draft or confirmed:

```kotlin
sealed interface OrderState {

    data object Draft : OrderState

    data class Confirmed(
        val confirmedAt: Instant,
    ) : OrderState
}
```

The aggregate snapshot is simple:

```kotlin
data class OrderSnapshot(
    val id: OrderId,
    val ownerId: UserId,
    val state: OrderState,
)
```

Now the rule.

An order can be confirmed only when:

- it is still a draft,
- the current user is the owner or can confirm orders,
- confirmation stores the current time,
- confirmation publishes a domain event.

Inside the aggregate it looks like this:

```kotlin
class Order private constructor(
    val id: OrderId,
    val ownerId: UserId,
    private var state: OrderState,
) {
    context(clock: ClockProvider, events: DomainEvents, user: UserContext)
    fun confirm() {
        require(state == OrderState.Draft) {
            "Only draft order can be confirmed"
        }

        require(user.userId == ownerId || user.canConfirmOrders) {
            "User cannot confirm this order"
        }

        val now = clock.now()

        state = OrderState.Confirmed(
            confirmedAt = now,
        )

        events.publish(
            OrderConfirmed(
                orderId = id,
                confirmedAt = now,
            ),
        )
    }
}
```

This is still a normal domain method.

But its signature says something important:

```kotlin
context(clock: ClockProvider, events: DomainEvents, user: UserContext)
fun confirm()
```

In other words:

> I can confirm an order, but only with a clock, domain events and the current user in context.

I do not want to keep `ClockProvider` in the entity constructor. A clock does not describe an order.

I also do not want to hide the current user behind a global `CurrentUserProvider`.

This operation simply runs in a specific context.

## Application layer

The use case adds a transaction.

This is where the example becomes more interesting, because the responsibilities are visible:

```kotlin
package io.github.camilyed.contextparameters.application

import io.github.camilyed.contextparameters.domain.ClockProvider
import io.github.camilyed.contextparameters.domain.DomainEvents
import io.github.camilyed.contextparameters.domain.OrderRepository
import io.github.camilyed.contextparameters.domain.OrderSnapshot
import io.github.camilyed.contextparameters.domain.Transaction
import io.github.camilyed.contextparameters.domain.UserContext

class ConfirmOrderUseCase(
    private val orderRepository: OrderRepository,
) {
    context(
        clock: ClockProvider,
        events: DomainEvents,
        user: UserContext,
        transaction: Transaction,
    )
    fun confirm(command: ConfirmOrderCommand): OrderSnapshot =
        transaction.within {
            val order = orderRepository.findById(command.orderId)

            order.confirm()

            orderRepository.save(order)

            order.toSnapshot()
        }
}
```

`orderRepository` is in the constructor because it is a stable dependency of the use case.

`command` is a normal parameter because it is the operation input.

`clock`, `events`, `user` and `transaction` are in `context(...)` because they describe the environment in which the operation runs.

This split is the most readable one for me.

## What about adapters?

The interfaces live in the domain:

```kotlin
interface OrderRepository {
    fun findById(id: OrderId): Order
    fun save(order: Order)
}
```

The adapter provides an implementation:

```kotlin
class InMemoryOrderRepository : OrderRepository {

    private val orders = mutableMapOf<OrderId, Order>()

    override fun findById(id: OrderId): Order =
        orders[id]
            ?: throw OrderNotFoundException("Order with id ${id.value} not found")

    override fun save(order: Order) {
        orders[order.id] = order
    }
}
```

So the dependency direction stays classic.

The domain knows the abstraction.

The adapter knows the implementation.

The use case gets the repository through the constructor and the execution context through `context parameters`.

## Why not Service Locator?

Because Service Locator removes dependencies from the signature.

```kotlin
fun confirm() {
    val clock = ServiceLocator.get<ClockProvider>()
    val events = ServiceLocator.get<DomainEvents>()
    val user = ServiceLocator.get<UserContext>()

    // ...
}
```

At first glance, the method looks cleaner.

But that cleanliness is borrowed.

Looking at `confirm()`, we no longer know what the method needs. We have to open the body. And if something is missing from the locator, we often find out at runtime.

With `context parameters`, the dependency is still part of the contract:

```kotlin
context(clock: ClockProvider, events: DomainEvents, user: UserContext)
fun confirm()
```

The reader sees the requirements.

The compiler checks them.

## Test without a container

In tests I do not need Spring or a DI container.

I have a fake clock:

```kotlin
class TestingClockProvider(
    private var currentInstant: Instant,
) : ClockProvider {

    override fun now(): Instant =
        currentInstant

    fun setNow(value: Instant) {
        currentInstant = value
    }
}
```

And a Given-When-Then aggregate test:

```kotlin
@Test
fun `should confirm draft order by owner`() {
    // given
    val order =
        Order.fromSnapshot(
            anOrder()
                .withId("ORD-123")
                .ownedBy("user-123")
                .build(),
        )

    // and
    currentUserIs("user-123")

    // and
    currentTimeIs("2026-06-05T10:15:30Z")

    // when
    context(clock, domainEvents, currentUser) {
        order.confirm()
    }

    // then
    expectThat(order.toSnapshot())
        .isConfirmed()
        .wasConfirmedAt("2026-06-05T10:15:30Z")

    // and
    expectThat(domainEvents)
        .hasPublishedOrderConfirmed(
            orderId = "ORD-123",
            confirmedAt = "2026-06-05T10:15:30Z",
        )
}
```

This is still a normal Given-When-Then test.

The test context is explicit:

```kotlin
context(clock, domainEvents, currentUser) {
    order.confirm()
}
```

But the domain operation itself does not have a noisy parameter list.

## Where would I use it?

For things that are execution context:

```kotlin
context(clock: ClockProvider)
fun Order.isExpired(): Boolean
```

```kotlin
context(user: UserContext)
fun Document.canBeEdited(): Boolean
```

```kotlin
context(tx: Transaction)
fun OrderRepository.save(order: Order)
```

```kotlin
context(events: DomainEvents)
fun Order.markAsPaid()
```

These are not the main input data.

They are also not always stable object dependencies.

They are the environment in which a piece of logic makes sense.

## Where would I not use it?

I would not put half of the application into `context(...)`.

☹️ Sad code:

```kotlin
context(
    user: UserContext,
    tenant: TenantContext,
    clock: ClockProvider,
    events: DomainEvents,
    logger: Logger,
    paymentGateway: PaymentGateway,
    orderRepository: OrderRepository,
    invoiceClient: InvoiceClient,
    emailSender: EmailSender,
)
fun confirmOrder(orderId: OrderId) {
    // ...
}
```

This is not execution context.

This is a DI container written with different syntax.

I also would not put normal input data there:

```kotlin
context(orderId: OrderId)
fun confirmOrder()
```

`orderId` is an operation argument.

Not context.

This is clearer:

```kotlin
fun confirmOrder(orderId: OrderId)
```

Not everything has to use the new syntax.

## Watch out for overly generic types

Context is resolved by types, so I would be careful with primitives and generic types.

☹️ Sad code:

```kotlin
context(id: String)
fun loadSomething() {
    // ...
}
```

Which string?

User id?

Tenant id?

Correlation id?

A small type is better:

```kotlin
@JvmInline
value class UserId(val value: String)

@JvmInline
value class TenantId(val value: String)

data class TenantContext(
    val tenantId: TenantId,
)
```

Good types matter even more with `context parameters`, because the compiler uses them to find the required context.

## Summary

`context parameters` are not a replacement for DI.

They do not create objects.

They do not manage lifecycles.

They do not replace constructors.

The split that works best for me is:

```text
constructor injection -> stable object dependencies
normal parameters     -> operation input
context parameters    -> execution context
```

In a small domain fragment, this can clean up the code nicely.

Not because parameters magically disappear.

Because we name their role better.

Example repository: [kotlin-context-parameters-ddd](https://github.com/CamilYed/kotlin-context-parameters-ddd)

Technical sources:

- [Context parameters - Kotlin Documentation](https://kotlinlang.org/docs/context-parameters.html)
- [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html)
