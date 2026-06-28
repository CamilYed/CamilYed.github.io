---
layout: post
title: "@Transactional: when the annotation starts getting in the way"
lang: en
ref: transactional-boundary
date: 2026-06-28
tags: [ java, kotlin, spring-boot, transactions, clean-architecture ]
permalink: /en/transactional-when-the-annotation-gets-in-the-way/
---

`@Transactional` is one of those Spring annotations that feels almost built into this popular
framework, so we often add it automatically. I still remember learning the basics of Spring, JPA and
Hibernate back in 2013, and at that time this annotation felt absolutely essential.

You add the annotation to a method, start the application, database writes work, rollback works, tests
are green.

And I really do not want to write an article titled: "`@Transactional` is bad, remove it from your
projects". That would simply be untrue. The problem starts somewhere else: very often the annotation
ends up where it is easiest to paste it, not where the transaction should actually start and end.

<!--more-->

In this article I want to show a few problems I regularly see in code based on `@Transactional`, and
an alternative I prefer to use in the application layer: an explicit transaction boundary based on a
lambda.

In this part I focus only on classic, imperative code: JDBC, JPA, Spring Data, `TransactionTemplate`.
No WebFlux, no R2DBC, no reactive streams. That deserves a separate article.

---

## 1. `@Transactional` is convenient, but it hides the boundary

The simplest example looks harmless:

```java

@Service
class AccountService {

    private final AccountRepository accountRepository;
    private final ExchangeRateClient exchangeRateClient;
    private final AccountHistoryRepository historyRepository;

    AccountService(
            AccountRepository accountRepository,
            ExchangeRateClient exchangeRateClient,
            AccountHistoryRepository historyRepository
    ) {
        this.accountRepository = accountRepository;
        this.exchangeRateClient = exchangeRateClient;
        this.historyRepository = historyRepository;
    }

    @Transactional
    AccountSnapshot exchange(ExchangeCurrencyCommand command) {
        var account = accountRepository.findById(command.accountId())
                .orElseThrow(AccountNotFoundException::new);

        var rate = exchangeRateClient.currentUsdRate();

        account.exchangePlnToUsd(command.amount(), rate);

        accountRepository.save(account);
        historyRepository.save(AccountHistory.from(account));

        return account.toSnapshot();
    }
}
```

At first glance, everything is OK. One method, one use case, one transaction.

But let’s ask one question: **what should actually be inside the transaction?**

Should fetching an exchange rate from an external API keep a database transaction open? Should command
validation run inside a transaction? Should mapping a DTO response run inside a transaction? Should
the whole method be transactional only because we do two database writes at the end?

And this is where the problem starts. `@Transactional` on a method makes the whole method
transactional. Not a "part of the method". Not "only save". The whole method.

☹️ **Sad little code:**

```java

@Transactional
AccountSnapshot exchange(ExchangeCurrencyCommand command) {
    validate(command);

    var rate = exchangeRateClient.currentUsdRate(); // external API

    var account = accountRepository.findById(command.accountId())
            .orElseThrow(AccountNotFoundException::new);

    account.exchangePlnToUsd(command.amount(), rate);

    accountRepository.save(account);
    historyRepository.save(AccountHistory.from(account));

    return account.toSnapshot();
}
```

Technically, it works.

The problem is that the transaction starts before validation and also stays open during the external
API call. If the API responds slowly, we hold transactional resources longer than necessary. If there
are locks inside, `SELECT ... FOR UPDATE`, higher load, or several parallel requests, things become
less pleasant.

Does it mean every method like this will immediately take production down? No. But this is exactly
the kind of code that "works" for a long time, until one day it starts hurting.

---

## 2. A transaction should have the smallest readable scope possible

I prefer when a transaction looks like a conscious decision in code, not a decoration on a class.

Example:

```java
public interface TransactionBoundary {

    <T> T inTransaction(Supplier<T> operation);

    void inTransaction(Runnable operation);
}
```

And a Spring-based implementation:

```java

@Component
class SpringTransactionBoundary implements TransactionBoundary {

    private final TransactionTemplate transactionTemplate;

    SpringTransactionBoundary(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }

    @Override
    public <T> T inTransaction(Supplier<T> operation) {
        return transactionTemplate.execute(status -> operation.get());
    }

    @Override
    public void inTransaction(Runnable operation) {
        transactionTemplate.executeWithoutResult(status -> operation.run());
    }
}
```

From this point, the use case can look like this:

🙂

```java

@Service
class AccountService {

    private final TransactionBoundary transaction;
    private final AccountRepository accountRepository;
    private final ExchangeRateClient exchangeRateClient;
    private final AccountHistoryRepository historyRepository;

    AccountService(
            TransactionBoundary transaction,
            AccountRepository accountRepository,
            ExchangeRateClient exchangeRateClient,
            AccountHistoryRepository historyRepository
    ) {
        this.transaction = transaction;
        this.accountRepository = accountRepository;
        this.exchangeRateClient = exchangeRateClient;
        this.historyRepository = historyRepository;
    }

    AccountSnapshot exchange(ExchangeCurrencyCommand command) {
        validate(command);

        var rate = exchangeRateClient.currentUsdRate();

        return transaction.inTransaction(() -> {
            var account = accountRepository.findById(command.accountId())
                    .orElseThrow(AccountNotFoundException::new);

            account.exchangePlnToUsd(command.amount(), rate);

            accountRepository.save(account);
            historyRepository.save(AccountHistory.from(account));

            return account.toSnapshot();
        });
    }
}
```

The difference in code is small, but the difference in intent is big.

Now it is clear that:

- validation is outside the transaction,
- the external API call is outside the transaction,
- the transaction contains only reading the current state, changing the aggregate and saving it,
- the boundary is visible exactly where I read the use case.

This is not a fight against Spring. Spring still does the work: opens the transaction, commits it,
rolls it back. The difference is that the application layer does not have to guess where exactly the
proxy works and which annotation will be intercepted.

---

## 3. The problem with private methods and self-invocation

This one is a classic.

☹️ **Sad little code:**

```java

@Service
class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;

    OrderService(OrderRepository orderRepository, PaymentRepository paymentRepository) {
        this.orderRepository = orderRepository;
        this.paymentRepository = paymentRepository;
    }

    OrderId create(CreateOrderCommand command) {
        validate(command);
        return createInsideTransaction(command);
    }

    @Transactional
    private OrderId createInsideTransaction(CreateOrderCommand command) {
        var order = Order.create(command.customerId(), command.amount());

        orderRepository.save(order);
        paymentRepository.reserve(order.id(), order.amount());

        return order.id();
    }
}
```

This is very easy to miss during code review. We see `@Transactional`, and our brain adds: "OK, there
is a transaction".

But a private method is not a place where a Spring proxy can normally intercept the call. What is
more, even when the method is public, but it is called from the same class through `this.someMethod()`,
we run into the self-invocation problem. We do not go through the proxy, so the annotation may not do
what we expect.

With a lambda, the problem disappears, because the transaction does not depend on whether the method
was public, private, called from another bean, or called from the same class.

🙂

```java

@Service
class OrderService {

    private final TransactionBoundary transaction;
    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;

    OrderService(
            TransactionBoundary transaction,
            OrderRepository orderRepository,
            PaymentRepository paymentRepository
    ) {
        this.transaction = transaction;
        this.orderRepository = orderRepository;
        this.paymentRepository = paymentRepository;
    }

    OrderId create(CreateOrderCommand command) {
        validate(command);

        return transaction.inTransaction(() -> createInsideTransaction(command));
    }

    private OrderId createInsideTransaction(CreateOrderCommand command) {
        var order = Order.create(command.customerId(), command.amount());

        orderRepository.save(order);
        paymentRepository.reserve(order.id(), order.amount());

        return order.id();
    }
}
```

Is the private method a problem now? No. It is just a helper method. The transaction boundary is at
the place where we call `transaction.inTransaction(...)`.

---

## 4. The annotation often mixes an architectural decision with a technical detail

In practice, `@Transactional` often lands on classes like `ApplicationService`, `UseCase`, `Facade`.

In other words, in places where we want to read the business scenario.

And then such a use case starts to look like a Christmas tree:

```java

@Service
@Transactional
class RegisterUserService {

    @Transactional(readOnly = true)
    UserView preview(PreviewUserCommand command) {
        // ...
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void auditRegistration(User user) {
        // ...
    }

    @Transactional(timeout = 5)
    UserId register(RegisterUserCommand command) {
        // ...
    }
}
```

Of course, these settings are sometimes needed. The problem is that they start living as decorations
on methods, not as a readable part of the process.

Why not simply name it like this?

```java
public interface TransactionBoundary {

    <T> T inTransaction(Supplier<T> operation);

    <T> T inReadOnlyTransaction(Supplier<T> operation);

    <T> T inNewTransaction(Supplier<T> operation);
}
```

Implementation:

```java

@Component
class SpringTransactionBoundary implements TransactionBoundary {

    private final TransactionTemplate required;
    private final TransactionTemplate readOnly;
    private final TransactionTemplate requiresNew;

    SpringTransactionBoundary(PlatformTransactionManager transactionManager) {
        this.required = new TransactionTemplate(transactionManager);

        this.readOnly = new TransactionTemplate(transactionManager);
        this.readOnly.setReadOnly(true);

        this.requiresNew = new TransactionTemplate(transactionManager);
        this.requiresNew.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }

    @Override
    public <T> T inTransaction(Supplier<T> operation) {
        return required.execute(status -> operation.get());
    }

    @Override
    public <T> T inReadOnlyTransaction(Supplier<T> operation) {
        return readOnly.execute(status -> operation.get());
    }

    @Override
    public <T> T inNewTransaction(Supplier<T> operation) {
        return requiresNew.execute(status -> operation.get());
    }
}
```

Usage:

```java
UserView preview(PreviewUserCommand command) {
    return transaction.inReadOnlyTransaction(() -> {
        var user = userRepository.findByEmail(command.email())
                .orElseThrow(UserNotFoundException::new);

        return UserView.from(user);
    });
}
```

In the code, I immediately see that this is a read. I do not have to look at the annotation on the
method, on the class, in the interface, in the superclass, or wonder whether the call accidentally
bypasses the proxy.

---

## 5. Long transactions are not only a technical problem, but also a modelling problem

Very often, long transactions are a symptom that a use case does too much.

☹️ **Sad little code:**

```java

@Transactional
OrderId placeOrder(PlaceOrderCommand command) {
    var customer = customerRepository.findById(command.customerId())
            .orElseThrow(CustomerNotFoundException::new);

    var pricing = pricingClient.calculate(command.items());
    var order = Order.place(customer, command.items(), pricing);

    orderRepository.save(order);

    invoiceClient.createInvoice(order);
    emailSender.sendOrderConfirmation(order);

    orderStatusRepository.save(OrderStatus.confirmed(order.id()));

    return order.id();
}
```

We have here:

- reading the customer,
- a call to an external pricing service,
- saving the order,
- a call to an invoicing system,
- sending an email,
- saving the status.

All of this inside one method and one transaction. If the external invoicing system responds after
three seconds, the transaction waits. If the email is not sent, should we roll back the order? Maybe
yes, maybe no. But that should be a business decision, not a side effect of marking the whole method
with a single annotation.

A better direction:

```java
OrderId placeOrder(PlaceOrderCommand command) {
    validate(command);

    var pricing = pricingClient.calculate(command.items());

    var orderId = transaction.inTransaction(() -> {
        var customer = customerRepository.findById(command.customerId())
                .orElseThrow(CustomerNotFoundException::new);

        var order = Order.place(customer, command.items(), pricing);

        orderRepository.save(order);
        outboxRepository.save(OrderPlacedEvent.from(order));

        return order.id();
    });

    return orderId;
}
```

External side effects can be handled later: through an outbox, an event handler, an asynchronous
process, a scheduler, a queue. You do not always have to bring event-driven architecture to a small
problem, but it is worth building the habit of asking: **does this side effect really have to be in
the same database transaction?**

---

## 6. Kotlin variant

In Kotlin, this kind of boundary looks even more natural. In one of my
[projects](https://github.com/CamilYed/currency-exchange-api/tree/main), I used a similar idea with
an `inTransaction { ... }` function, because this syntax fits the style of use cases very well.

Minimal version:

```kotlin
interface TransactionBoundary {

    fun <T> inTransaction(block: () -> T): T

    fun <T> inReadOnlyTransaction(block: () -> T): T
}
```

Spring implementation:

```kotlin
@Component
class SpringTransactionBoundary(
    transactionManager: PlatformTransactionManager,
) : TransactionBoundary {

    private val required = TransactionTemplate(transactionManager)

    private val readOnly = TransactionTemplate(transactionManager).apply {
        isReadOnly = true
    }

    override fun <T> inTransaction(block: () -> T): T {
        return required.execute<T> { block() } as T
    }

    override fun <T> inReadOnlyTransaction(block: () -> T): T {
        return readOnly.execute<T> { block() } as T
    }
}
```

Usage in an application service:

```kotlin
@Service
class AccountService(
    private val transaction: TransactionBoundary,
    private val accountRepository: AccountRepository,
    private val accountOperationRepository: AccountOperationRepository,
    private val exchangeRateProvider: ExchangeRateProvider,
) {

    fun exchange(command: ExchangeCurrencyCommand): AccountSnapshot {
        val rate = exchangeRateProvider.currentUsdRate()

        return transaction.inTransaction {
            val account = accountRepository.findBy(command.accountId)
                ?: throw AccountNotFoundException(command.accountId)

            account.exchangePlnToUsd(command.amount, rate)

            accountRepository.save(account)
            accountOperationRepository.save(account.pullEvents())

            account.toSnapshot()
        }
    }
}
```

Is this much more complicated than `@Transactional`? No.

But it is more explicit. And for me, that is the main benefit.

---

## 7. Testing becomes simpler

If the use case depends on a small interface:

```java
public interface TransactionBoundary {

    <T> T inTransaction(Supplier<T> operation);
}
```

then in a unit test I do not need Spring, proxies, or a real transaction manager.

```java
final class ImmediateTransactionBoundary implements TransactionBoundary {

    @Override
    public <T> T inTransaction(Supplier<T> operation) {
        return operation.get();
    }
}
```

And I can test the use case logic normally:

```java

@Test
void shouldCreateOrderAndReservePayment() {
    // given
    var transaction = new ImmediateTransactionBoundary();
    var orderRepository = new InMemoryOrderRepository();
    var paymentRepository = new InMemoryPaymentRepository();

    var service = new OrderService(transaction, orderRepository, paymentRepository);

    // when
    var orderId = service.create(new CreateOrderCommand("customer-1", BigDecimal.TEN));

    // then
    assertThat(orderRepository.findById(orderId)).isPresent();
    assertThat(paymentRepository.existsFor(orderId)).isTrue();
}
```

In an integration test, of course, I still want to verify that the Spring implementation works with
the database. But I do not have to start the whole world for every business test only because there
is a `@Transactional` annotation somewhere on a method.

---

## 8. So should we throw `@Transactional` away?

No.

There are places where `@Transactional` is good enough:

- simple CRUD,
- a small admin application,
- a method that really is one database operation from start to finish,
- infrastructure code where a dependency on Spring does not bother us,
- a quick prototype where an explicit boundary would only add noise.

The problem is not that the annotation exists. The problem is that we often use it as the default
answer to every question about data consistency.

And a transaction is a design decision.

Where does a consistent write start? Where does the effect of rollback end? Should the external call
be inside? Should the event be saved together with the aggregate? Should an email roll back the
order? Should audit have its own transaction?

The annotation hides these questions. A lambda-based boundary forces us to see them.

---

## Summary

`@Transactional` is convenient, but it has a few traps:

1. It covers the whole method, so it is easy to create a transaction scope that is too wide.
2. It works through proxies, so private methods and self-invocation can surprise you.
3. It mixes technical configuration with the description of the use case.
4. It makes it harder to see what exactly should be rolled back.
5. It tempts us to put things inside one transaction that should not be there.

The alternative does not have to be complicated. Sometimes a small interface is enough:

```java
transaction.inTransaction(() -> {
    // only what really has to be inside the transaction
});
```

For me, the most important side effect of this approach is that the use case starts telling the truth
about the process. It does not hide boundaries behind an annotation, does not require remembering how
Spring proxies work, and does not pretend that the whole method is always a good transaction scope.

And if the transaction boundary is not visible in code, it is very easy to assume it is where we wish
it was.

Unfortunately, code does not run on wishes.

---

## What next

This article was about the classic, imperative approach: JDBC/JPA + `TransactionTemplate`.

In the reactive world, the topic becomes even more interesting, because Reactor context, `Mono`,
`Flux`, subscription cancellation and `TransactionalOperator` enter the picture. For this reason I
prepared a separate project: `spring-reactive-transaction-boundary`.

Repository:

```text
https://github.com/CamilYed/spring-reactive-transaction-boundary
```
