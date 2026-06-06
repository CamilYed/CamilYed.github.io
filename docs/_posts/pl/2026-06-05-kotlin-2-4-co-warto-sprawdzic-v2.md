---
layout: post
title: "Kotlin context parameters: nie DI, tylko kontekst wykonania"
lang: pl
ref: kotlin-context-parameters-practical
date: 2026-06-05
tags: [ kotlin, ddd, testing, software-engineering ]
permalink: /pl/kotlin-context-parameters-nie-di-tylko-kontekst-wykonania/
---

Kiedy zobaczyłem `context(...)` przy funkcji, pierwsze skojarzenie było dość oczywiste:

> czy Kotlin dostał kolejny sposób na dependency injection?

Po kilku próbach patrzę na to inaczej.

`context parameters` nie tworzą obiektów. Nie zarządzają cyklem życia zależności. Nie zastępują Springa, Koina ani Daggera.

One opisują coś prostszego:

> ta funkcja może wykonać się tylko wtedy, gdy w miejscu wywołania istnieje wymagany kontekst.

I ten warunek jest sprawdzany przez kompilator.

To jest dla mnie najważniejsze zdanie w całym temacie. Nie runtime. Nie magiczny provider. Nie globalny singleton. Kompilator.

## Najprostszy przykład

Weźmy zegar.

```kotlin
interface ClockProvider {
    fun now(): Instant
}
```

Jeśli funkcja potrzebuje aktualnego czasu, zwykle mamy trzy opcje.

Możemy przekazać zegar jako zwykły parametr:

```kotlin
fun confirmedAt(clock: ClockProvider): Instant =
    clock.now()
```

Możemy trzymać go w konstruktorze jak klasyczne dependency injection:

```kotlin
class OrderService(
    private val clock: ClockProvider,
) {
    fun confirmedAt(): Instant =
        clock.now()
}
```

Albo możemy powiedzieć, że zegar jest kontekstem wykonania tej funkcji:

```kotlin
context(clock: ClockProvider)
fun confirmedAt(): Instant =
    clock.now()
```

Wywołanie:

```kotlin
val clock = FixedClockProvider(
    Instant.parse("2026-06-05T10:15:30Z"),
)

context(clock) {
    confirmedAt()
}
```

Bez `context(clock)` ta funkcja nie powinna się skompilować.

I to jest różnica względem service locatora. Zależność nadal jest widoczna w sygnaturze funkcji, ale nie przepychamy jej jako zwykłego argumentu przez każde wywołanie.

## Parametr, konstruktor czy context?

Nie traktowałbym `context parameters` jako nowej domyślnej odpowiedzi na wszystko.

Dla mnie podział jest mniej więcej taki:

```text
zwykłe parametry      -> dane wejściowe operacji
constructor injection -> stałe zależności obiektu
context parameters    -> kontekst wykonania
```

`orderId`, `amount`, `email`, `newAddress` — to są normalne parametry.

`OrderRepository`, `PaymentGateway`, `HttpClient` — często pasują do konstruktora use case'a albo serwisu.

`ClockProvider`, `UserContext`, `DomainEvents`, `Transaction`, `TenantContext`, `CorrelationId` — to są dobrzy kandydaci na kontekst wykonania.

Oczywiście nie zawsze. Ale ta granica jest dla mnie dużo zdrowsza niż myślenie: „super, teraz wszystko wrzucamy w `context(...)`”.

## Przykład z repo

Do artykułu zrobiłem małe repo z przykładem DDD / hexagonal:

```text
src/main/kotlin/io/github/camilyed/contextparameters
├── domain
├── application
└── adapter
```

Nie ma tutaj Springa. Celowo.

Chciałem, żeby przykład pokazywał samą składnię i decyzję projektową, a nie integrację z frameworkiem.

Domena ma jeden agregat: `Order`.

Zamówienie może być szkicem albo może być potwierdzone:

```kotlin
sealed interface OrderState {

    data object Draft : OrderState

    data class Confirmed(
        val confirmedAt: Instant,
    ) : OrderState
}
```

Snapshot agregatu jest prosty:

```kotlin
data class OrderSnapshot(
    val id: OrderId,
    val ownerId: UserId,
    val state: OrderState,
)
```

I teraz sama reguła.

Zamówienie można potwierdzić tylko wtedy, gdy:

- jest jeszcze szkicem,
- aktualny użytkownik jest właścicielem albo może potwierdzać zamówienia,
- zapisujemy czas potwierdzenia,
- publikujemy event domenowy.

W agregacie wygląda to tak:

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

To jest zwykła metoda domenowa.

Ale jej sygnatura mówi coś ważnego:

```kotlin
context(clock: ClockProvider, events: DomainEvents, user: UserContext)
fun confirm()
```

Czyli:

> potrafię potwierdzić zamówienie, ale tylko w kontekście zegara, eventów domenowych i aktualnego użytkownika.

Nie chcę trzymać `ClockProvider` w konstruktorze encji. Zegar nie opisuje zamówienia.

Nie chcę też chować użytkownika w globalnym `CurrentUserProvider`.

Ta operacja po prostu działa w konkretnym kontekście.

## Warstwa aplikacji

W use case dochodzi jeszcze transakcja.

I tutaj przykład robi się ciekawszy, bo widać podział odpowiedzialności:

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

`orderRepository` jest w konstruktorze, bo to stała zależność use case'a.

`command` jest zwykłym parametrem, bo to dane wejściowe operacji.

`clock`, `events`, `user` i `transaction` są w `context(...)`, bo opisują otoczenie, w którym wykonuje się operacja.

Ten podział jest dla mnie najczytelniejszy.

## A gdzie adaptery?

Interfejsy są w domenie:

```kotlin
interface OrderRepository {
    fun findById(id: OrderId): Order
    fun save(order: Order)
}
```

Adapter daje implementację:

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

Czyli klasyczny kierunek zależności zostaje zachowany.

Domena zna abstrakcję.

Adapter zna implementację.

Use case dostaje repozytorium przez konstruktor, a kontekst wykonania przez `context parameters`.

## Dlaczego nie Service Locator?

Bo Service Locator usuwa zależności z sygnatury.

```kotlin
fun confirm() {
    val clock = ServiceLocator.get<ClockProvider>()
    val events = ServiceLocator.get<DomainEvents>()
    val user = ServiceLocator.get<UserContext>()

    // ...
}
```

Na pierwszy rzut oka metoda wygląda czyściej.

Ale to jest czystość na kredyt.

Czytając sygnaturę `confirm()`, nie wiemy już, czego ta metoda potrzebuje. Trzeba wejść do środka. A jeśli czegoś zabraknie w locatorze, często dowiemy się o tym dopiero w runtime.

Przy `context parameters` zależność nadal jest częścią kontraktu:

```kotlin
context(clock: ClockProvider, events: DomainEvents, user: UserContext)
fun confirm()
```

Czytelnik widzi wymagania.

Kompilator ich pilnuje.

## Test bez kontenera

W testach nie potrzebuję Springa ani kontenera DI.

Mam fake zegara:

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

I test zachowania agregatu:

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

To jest nadal zwykły test Given-When-Then.

Kontekst testu jest jawny:

```kotlin
context(clock, domainEvents, currentUser) {
    order.confirm()
}
```

Ale sama operacja domenowa nie ma sztucznej listy parametrów.

## Kiedy bym tego użył?

Przy rzeczach, które są kontekstem wykonania:

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

To nie są główne dane wejściowe.

To nie są też zawsze stałe zależności obiektu.

To jest otoczenie, w którym dana logika ma sens.

## Kiedy bym tego nie użył?

Nie wrzucałbym do `context(...)` połowy aplikacji.

☹️ Smutny kodzik:

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

To nie jest kontekst wykonania.

To jest kontener DI zapisany inną składnią.

Nie dawałbym tam też zwykłych danych wejściowych:

```kotlin
context(orderId: OrderId)
fun confirmOrder()
```

`orderId` jest argumentem operacji.

Nie kontekstem.

Czytelniej:

```kotlin
fun confirmOrder(orderId: OrderId)
```

Nie wszystko musi używać nowej składni.

## Uwaga na zbyt ogólne typy

Context jest rozwiązywany po typach, więc warto uważać na prymitywy i zbyt ogólne typy.

☹️ Smutny kodzik:

```kotlin
context(id: String)
fun loadSomething() {
    // ...
}
```

Jaki string?

Id użytkownika?

Id tenanta?

Correlation id?

Lepiej użyć małych typów:

```kotlin
@JvmInline
value class UserId(val value: String)

@JvmInline
value class TenantId(val value: String)

data class TenantContext(
    val tenantId: TenantId,
)
```

Przy `context parameters` dobre typy są jeszcze ważniejsze, bo to po nich kompilator rozpoznaje, czego szuka.

## Podsumowanie

`context parameters` nie są zamiennikiem DI.

Nie tworzą obiektów.

Nie zarządzają cyklem życia.

Nie zastępują konstruktora.

Najprostszy podział, który mi się sprawdza:

```text
constructor injection -> stałe zależności obiektu
zwykłe parametry      -> dane wejściowe operacji
context parameters    -> kontekst wykonania
```

W małym fragmencie domeny to potrafi fajnie oczyścić kod.

Nie dlatego, że parametry magicznie znikają.

Tylko dlatego, że lepiej nazywamy ich rolę.

Przykładowe repo: [kotlin-context-parameters-ddd](https://github.com/CamilYed/kotlin-context-parameters-ddd)

Źródła techniczne:

- [Context parameters - Kotlin Documentation](https://kotlinlang.org/docs/context-parameters.html)
- [What's new in Kotlin 2.4.0](https://kotlinlang.org/docs/whatsnew24.html)
