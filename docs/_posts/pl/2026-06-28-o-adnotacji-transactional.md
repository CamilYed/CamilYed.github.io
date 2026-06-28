---
layout: post
title: "@Transactional: kiedy adnotacja zaczyna przeszkadzać"
lang: pl
ref: transactional-boundary
date: 2026-06-28
tags: [ java, kotlin, spring-boot, transactions, clean-architecture ]
permalink: /pl/transactional-kiedy-adnotacja-przeszkadza/
---

`@Transactional` to jedna z tych adnotacji w Springu, które są jakby wbudowane w ten popularny 
framework, że dodajemy ją często z automatu. Pamiętam, w roku 2013 ucząc się podstaw Springa, JPA, Hibernate, że ta
adnotacja była jak na tamte czasy nieodzowna.

Dodajesz adnotację na metodzie, odpalasz aplikację, zapis do bazy działa, rollback działa, testy są zielone.

I naprawdę nie mam zamiaru pisać tekstu pod tytułem: "`@Transactional` jest zły, usuńcie go z projektów". To byłoby
zwyczajnie nieprawdziwe. Problem zaczyna się gdzie indziej: bardzo często adnotacja trafia tam, gdzie najłatwiej ją
wkleić, a nie tam, gdzie faktycznie powinna zaczynać się i kończyć transakcja.

<!--more-->

W tym artykule chciałbym pokazać kilka problemów, które regularnie widzę w kodzie opartym o `@Transactional`, oraz
alternatywę, którą wolę stosować w warstwie aplikacyjnej: jawny transaction boundary oparty o lambdę.

W tej części skupiam się wyłącznie na klasycznym, imperatywnym kodzie: JDBC, JPA, Spring Data, `TransactionTemplate`.
Bez WebFluxa, bez R2DBC, bez reactive streams. O tym można zrobić osobny tekst.

---

## 1. `@Transactional` jest wygodne, ale ukrywa granicę

Najprostszy przykład wygląda niewinnie:

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

Na pierwszy rzut oka jest OK. Jedna metoda, jeden use case, jedna transakcja.

Ale zadajmy sobie pytanie: **co tak naprawdę powinno być w transakcji?**

Czy pobranie kursu waluty z zewnętrznego API powinno trzymać otwartą transakcję bazodanową? Czy walidacja komendy
powinna działać w transakcji? Czy mapowanie odpowiedzi DTO powinno działać w transakcji? Czy cała metoda ma być
transakcyjna tylko dlatego, że na końcu robimy dwa zapisy do bazy?

I tutaj zaczyna się problem. `@Transactional` na metodzie robi transakcyjną całą metodę. Nie "fragment metody".
Nie "tylko save". Całą metodę.

☹️ **Smutny kodzik:**

```java

@Transactional
AccountSnapshot exchange(ExchangeCurrencyCommand command) {
    validate(command);

    var rate = exchangeRateClient.currentUsdRate(); // zewnętrzne API

    var account = accountRepository.findById(command.accountId())
            .orElseThrow(AccountNotFoundException::new);

    account.exchangePlnToUsd(command.amount(), rate);

    accountRepository.save(account);
    historyRepository.save(AccountHistory.from(account));

    return account.toSnapshot();
}
```

Technicznie działa.

Tylko że transakcja zaczyna się przed walidacją i trwa również podczas wywołania zewnętrznego API. Jeśli API odpowiada
wolno, trzymamy zasoby transakcyjne dłużej niż trzeba. Jeśli w środku mamy jeszcze locki, `SELECT ... FOR UPDATE`,
większy load albo kilka równoległych requestów, to robi się mniej przyjemnie.

Czy to znaczy, że każda taka metoda od razu położy produkcję? Nie. Ale to jest dokładnie ten typ kodu, który przez
długi czas "działa", aż pewnego dnia zaczyna boleć.

---

## 2. Transakcja powinna mieć możliwie mały i czytelny zakres

Wolę, kiedy transakcja wygląda jak świadoma decyzja w kodzie, a nie jak dekoracja klasy.

Przykład:

```java
public interface TransactionBoundary {

    <T> T inTransaction(Supplier<T> operation);

    void inTransaction(Runnable operation);
}
```

I implementacja oparta o Springa:

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

Od tej chwili use case może wyglądać tak:

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

Różnica jest mała w kodzie, ale duża w intencji.

Teraz widać, że:

- walidacja jest poza transakcją,
- call do zewnętrznego API jest poza transakcją,
- transakcja obejmuje tylko odczyt aktualnego stanu, zmianę agregatu i zapis,
- granica jest widoczna tam, gdzie czytam use case.

To nie jest walka ze Springiem. Spring nadal robi robotę: otwiera transakcję, commituje, rollbackuje. Różnica polega na
tym, że warstwa aplikacyjna nie musi zgadywać, gdzie dokładnie działa proxy i która adnotacja zostanie przechwycona.

---

## 3. Problem prywatnych metod i self-invocation

To jest klasyk.

☹️ **Smutny kodzik:**

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

Na code review bardzo łatwo to przeoczyć. Widzimy `@Transactional`, mózg dopowiada: "OK, transakcja jest".

Tylko że prywatna metoda nie jest miejscem, w którym Springowe proxy może normalnie przeciąć wywołanie. Co więcej,
nawet gdy metoda jest publiczna, ale wywoływana z tej samej klasy przez `this.someMethod()`, wchodzimy w problem
self-invocation. Nie przechodzimy przez proxy, więc adnotacja może nie zrobić tego, czego się spodziewamy.

Z lambdą problem znika, bo transakcja nie zależy od tego, czy metoda była publiczna, prywatna, zawołana z innego beana
czy z tej samej klasy.

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

Czy prywatna metoda jest teraz problemem? Nie. To zwykła metoda pomocnicza. Granica transakcji jest w miejscu, gdzie
wywołujemy `transaction.inTransaction(...)`.

---

## 4. Adnotacja często miesza decyzję architektoniczną z detalem technicznym

`@Transactional` w praktyce często ląduje na klasach typu `ApplicationService`, `UseCase`, `Facade`.

Czyli w miejscu, gdzie chcemy czytać scenariusz biznesowy.

A potem taki use case zaczyna wyglądać jak choinka:

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

Oczywiście, te ustawienia są czasem potrzebne. Problem polega na tym, że zaczynają żyć jako dekoracje metod, a nie jako
czytelny fragment procesu.

Czemu po prostu nie nazwać tego w taki sposób?:

```java
public interface TransactionBoundary {

    <T> T inTransaction(Supplier<T> operation);

    <T> T inReadOnlyTransaction(Supplier<T> operation);

    <T> T inNewTransaction(Supplier<T> operation);
}
```

Implementacja:

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

Użycie:

```java
UserView preview(PreviewUserCommand command) {
    return transaction.inReadOnlyTransaction(() -> {
        var user = userRepository.findByEmail(command.email())
                .orElseThrow(UserNotFoundException::new);

        return UserView.from(user);
    });
}
```

W kodzie od razu widać, że to jest odczyt. Nie muszę patrzeć na adnotację nad metodą, nad klasą, w interfejsie, w
superklasie, albo zastanawiać się, czy przypadkiem wywołanie nie omija proxy.

---

## 5. Długie transakcje to nie tylko problem techniczny, ale też modelowania

Bardzo często długie transakcje są objawem tego, że use case robi za dużo.

☹️ **Smutny kodzik:**

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

Mamy tutaj:

- odczyt klienta,
- call do zewnętrznego pricingu,
- zapis zamówienia,
- call do systemu faktur,
- wysyłkę maila,
- zapis statusu.

To wszystko w jednej metodzie i jednej transakcji. Jeśli zewnętrzny system faktur odpowie po 3 sekundach, transakcja
czeka. Jeśli mail nie wyjdzie, rollbackujemy zapis zamówienia? Może tak, może nie. Ale to powinna być decyzja
biznesowa, a nie efekt tego, że cała metoda była oznaczona jedną adnotacją.

Lepszy kierunek:

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

Zewnętrzne efekty uboczne można obsłużyć później: przez outbox, event handler, proces asynchroniczny, scheduler,
kolejkę. Nie zawsze trzeba robić event-driven architecture z armaty do muchy, ale warto mieć odruch pytania: **czy ten
efekt uboczny naprawdę musi być w tej samej transakcji bazodanowej?**

---

## 6. Kotlinowy wariant

W Kotlinie taki boundary wygląda jeszcze naturalniej. W jednym z moich [projektów](https://github.com/CamilYed/currency-exchange-api/tree/main) robiłem podobną rzecz właśnie przez
funkcję `inTransaction { ... }`, bo taki zapis bardzo dobrze pasuje do stylu use case'ów.

Minimalna wersja:

```kotlin
interface TransactionBoundary {

    fun <T> inTransaction(block: () -> T): T

    fun <T> inReadOnlyTransaction(block: () -> T): T
}
```

Implementacja Springowa:

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

Użycie w serwisie aplikacyjnym:

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

Czy to jest dużo bardziej skomplikowane niż `@Transactional`? Nie.

Ale jest bardziej jawne. I dla mnie to jest główny zysk.

---

## 7. Testowanie staje się prostsze

Jeśli use case zależy od małego interfejsu:

```java
public interface TransactionBoundary {

    <T> T inTransaction(Supplier<T> operation);
}
```

to w teście jednostkowym nie potrzebuję Springa, proxy ani prawdziwego transaction managera.

```java
final class ImmediateTransactionBoundary implements TransactionBoundary {

    @Override
    public <T> T inTransaction(Supplier<T> operation) {
        return operation.get();
    }
}
```

I testuję logikę use case'a normalnie:

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

W teście integracyjnym oczywiście nadal chcę sprawdzić, czy implementacja Springowa działa z bazą. Ale nie muszę do
każdego testu biznesowego odpalać całego świata tylko dlatego, że gdzieś na metodzie jest `@Transactional`.

---

## 8. Czyli `@Transactional` wyrzucamy do kosza?

Nie.

Są miejsca, gdzie `@Transactional` jest wystarczająco dobre:

- prosty CRUD,
- mała aplikacja administracyjna,
- metoda, która naprawdę w całości jest jedną operacją bazodanową,
- kod infrastrukturalny, gdzie nie przeszkadza nam zależność od Springa,
- szybki prototyp, gdzie jawna granica byłaby tylko dodatkowym szumem.

Problem nie polega na tym, że adnotacja istnieje. Problem polega na tym, że często używamy jej jako domyślnej odpowiedzi
na każde pytanie o spójność danych.

A transakcja jest decyzją projektową.

Gdzie zaczyna się spójny zapis? Gdzie kończy się wpływ rollbacka? Czy zewnętrzny call powinien być w środku? Czy event
ma być zapisany razem z agregatem? Czy mail ma rollbackować zamówienie? Czy audyt ma mieć własną transakcję?

Adnotacja ukrywa te pytania. Boundary przez lambdę zmusza, żeby je zobaczyć.

---

## Podsumowanie

`@Transactional` jest wygodne, ale ma kilka pułapek:

1. Obejmuje całą metodę, więc łatwo stworzyć zbyt szeroki scope transakcji.
2. Działa przez proxy, więc prywatne metody i self-invocation potrafią zaskoczyć.
3. Miesza konfigurację techniczną z opisem use case'a.
4. Utrudnia zobaczenie, co dokładnie ma zostać rollbackowane.
5. Kusi, żeby wrzucać do jednej transakcji rzeczy, które nie powinny tam być.

Alternatywa nie musi być skomplikowana. Czasem wystarczy mały interfejs:

```java
transaction.inTransaction(() ->{
        // tylko to, co naprawdę ma być w transakcji
        });
```

Dla mnie najważniejszy jest efekt uboczny tego podejścia: kod use case'a zaczyna mówić prawdę o procesie. Nie ukrywa
granic za adnotacją, nie wymaga pamiętania o Springowym proxy i nie udaje, że cała metoda zawsze jest dobrym zakresem
transakcji.

A jeśli granicy transakcji nie widać w kodzie, to bardzo łatwo założyć, że jest tam, gdzie chcielibyśmy, żeby była.

Niestety kod nie działa na życzeniach.

---

## Co dalej

Ten tekst dotyczył klasycznego, imperatywnego podejścia: JDBC/JPA + `TransactionTemplate`.

W świecie reaktywnym temat robi się jeszcze ciekawszy, bo dochodzi Reactor context, `Mono`, `Flux`, anulowanie
subskrypcji i `TransactionalOperator`. Z tego powodu przygotowałem osobny projekt:
`spring-reactive-transaction-boundary`.

Repozytorium jest tutaj:

```text
https://github.com/CamilYed/spring-reactive-transaction-boundary
```

