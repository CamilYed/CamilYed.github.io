---
layout: post
title: "Testy, które nie kłamią cz. 2: Pułapka Mockito i implementacje In-Memory"
lang: pl
ref: modern-testing-practices-2
date: 2026-01-31
tags: [ java, testing, mockito, software-architecture ]
permalink: /pl/testy-ktore-nie-klamia-cz2/
---

W poprzedniej części skupiliśmy się na tym, jak pisać testy, które po prostu dobrze się czyta. Ale czytelność to tylko
połowa sukcesu. Możesz mieć najpiękniej napisaną sekcję Given-When-Then, która... kompletnie nic nie sprawdza.

Dziś pogadamy o zaufaniu do naszych testów. Bo najgorszy rodzaj testu to taki, który daje Ci poczucie bezpieczeństwa,
mimo że Twój kod pod spodem robi zupełnie coś innego, na co by wskazywał sam test.

### 1. Pułapka testowania implementacji (White Box)

Zauważyłem, że w wielu projektach Mockito dodaje się do testów "z automatu". Generujemy klasę testową, mockujemy
wszystkie zależności i cyk – robota zrobiona. Mało kto zadaje sobie wtedy pytanie: **po co ja właściwie tego mocka
używam?**

Wyobraź sobie prosty serwis do aktualizacji danych użytkownika.

☹️ **Smutny kodzik:**

```java

@Test
void shouldUpdateUserName() {
    // given
    var userId = 1L;
    var user = new User(userId, "Jan");
    // Musimy "nakarmić" mocka, żeby test w ogóle ruszył
    when(userRepository.findById(userId)).thenReturn(Optional.of(user));

    // when
    userService.updateName(userId, "Jan Kowalski");

    // then
    // Sprawdzamy tylko techniczne wywołanie metody. 
    // Czy wiemy, czy imię faktycznie zostało zmienione w obiekcie przed zapisem? 
    // Ten test powie "TAK", nawet jeśli serwis wyśle do save() stare dane.
    verify(userRepository).save(any(User.class));
}
```

Ten test Cię oszukuje. Sprawdza tylko, czy zawołano metodę save. Jeśli programista pomyli pola i w kodzie produkcyjnym
przypisze nową wartość do zupełnie innego pola (albo w ogóle pominie przypisanie), ten test nadal przejdzie na zielono!
Zamiast testować zachowanie biznesowe (zmiana imienia), testujesz techniczne wywołanie biblioteki.

No dobra, ale ktoś zauważy, że możemy jednak zweryfikować stan obiektu i spróbuje użyć `ArgumentCaptor` przy logice
aktualizacji danych (update).

☹️ Jeszcze bardziej smutny kodzik:

```java

@Test
void shouldUpdateUserName_CaptorVersion() {
    // given
    var userId = 1L;
    var existingUser = new User(userId, "Jan");
    var userCaptor = ArgumentCaptor.forClass(User.class);

    when(userRepository.findById(userId)).thenReturn(Optional.of(existingUser));

    // when
    userService.updateName(userId, "Jan Kowalski");

    // then
    // Tutaj zaczynają się schody. Odsłaniamy szczegóły implementacji.
    verify(userRepository).save(userCaptor.capture());
    var savedUser = userCaptor.getValue();

    assertThat(savedUser.getName()).isEqualTo("Jan Kowalski");
}
```

I co? Sukces? No nie do końca. Właśnie weszliśmy w tryb `White Box Testing.` Testy stają się kruche (`Fragile tests`),
bo:

- Refaktoryzacja to ból: Zmieniasz `save()` na `saveAll()`? Test wybucha, mimo że logika biznesowa działa.
- Testujesz "jak", a nie "co": Obchodzi Cię, czy wywołałeś konkretną linię kodu, a nie jaki jest wynik dla użytkownika.
- Sonar kłamie: Raporty pokazują pokrycie linii, ale Ty ich nie przetestowałeś – Ty je tylko wywołałeś w sztucznym
  środowisku.

#### Rozwiązanie: Implementacja In-Memory

Zamiast walczyć z Mockito, potraktujmy serwis jako czarną skrzynkę. Potrzebujemy czegoś, co udaje bazę danych, ale
działa w pamięci. `ConcurrentHashMap` pod spodem repozytorium to najprostszy sposób.

```java
public class InMemoryUserRepository implements UserRepository {
    private final Map<Long, User> db = new ConcurrentHashMap<>();

    @Override
    public User save(User user) {
        db.put(user.getId(), user);
        return user;
    }

    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(db.get(id));
    }

    public void clear() {
        db.clear();
    }
}
```

Tak mógłby wyglądać wtedy test oparty na stanie `State-based`.
Teraz nasz test nie potrzebuje żadnych `verify`. Po prostu wywołujemy akcję i sprawdzamy, czy stan w "bazie" się zgadza.

```java
class UserServiceTest {
    private final InMemoryUserRepository userRepository = new InMemoryUserRepository();
    private final UserService userService = new UserService(userRepository);

    @BeforeEach
    void setup() {
        userRepository.clear();
    }

    @Test
    void shouldUpdateUserName() {
        // given
        userRepository.save(new User(1L, "Jan"));

        // when
        userService.updateName(1L, "Jan Kowalski");

        // then
        var updatedUser = userRepository.findById(1L).orElseThrow();
        assertThat(updatedUser.getName()).isEqualTo("Jan Kowalski");
    }
}
```

#### A co z DSL?

Pamiętasz pierwszą część? Możemy użyć tamtych wzorców, aby przygotować stan początkowy jeszcze czyściej. Zamiast ręcznie
wywoływać userRepository.save() w sekcji given, użyjemy naszej "zdolności" (Ability).

🙂 Uśmiechnięty kodzik:

```java

@Test
void shouldUpdateUserName() {
    // given
    thereIsAUser(anUser().withId(1L).withName("Jan").build());

    // when
    userService.updateName(1L, "Jan Kowalski");

    // then
    var updatedUser = userRepository.findById(1L).orElseThrow();
    assertThat(updatedUser.getName()).isEqualTo("Jan Kowalski");
}
```

Czy to nadal White Box? Ktoś powie: "Zaraz, ale w asercji wywołujesz repozytorium!". Nie. Różnica jest fundamentalna:

- W Mockito (Interakcja): Pytasz: "Czy zawołałeś metodę save?". Jeśli programista zmieni sposób zapisu, test padnie.
- W In-Memory (Stan): Pytasz: "Systemie, nieważne jak to zrobiłeś, czy ten użytkownik ma nowe imię?".

W podejściu `Black Box` traktujemy parę Service + InMemoryRepo jako jedną czarną skrzynkę. Nie obchodzi nas, ile razy
serwis "gadał" z repozytorium. Obchodzi nas efekt końcowy.

#### Jak to mogło by wyglądać ostatecznie

Możesz się zastanawiać: Skąd `Ability` bierze repozytorium i czy to na pewno ta sama instancja, której używa serwis? To
kluczowy punkt. Żeby to działało, musimy mieć jedno źródło prawdy.

Najlepszym sposobem jest użycie interfejsów z domyślnymi implementacjami (default methods).

```java
public interface UserAbility {
    UserRepository userRepository(); // Metoda "dostawca"

    default void thereIsAUser(User user) {
        userRepository().save(user);
    }
}
```

Stwórzmy sobie klasę bazową dla testów unitowych, aby ukryć detale technicznej implementacji naszego DSL.

```java
public abstract class BaseUnitTest implements UserAbility {
    
    // Jedna, wspólna instancja dla serwisu, asercji i wszystkich Ability
    protected final InMemoryUserRepository userRepository = new InMemoryUserRepository();

    @Override
    public UserRepository userRepository() {
        return userRepository;
    }

    @BeforeEach
    void clearDatabase() {
        userRepository.clear(); // Ważne aby izolować testy czyścimy stan bazy w pamięci przed każdym
    }
}
```

Dzięki temu Twoja klasa testowa po prostu dziedziczy po `BaseUnitTest` i ma już możliwość skorzystania z naszej
implementacji `In-Memory`.Oto jak wygląda ostateczny kod Twojej klasy testowej. Zauważ, jak mało "szumu technicznego" tu
zostało. Skupiamy się wyłącznie na zachowaniu biznesowym.

```java
class UserServiceTest extends BaseUnitTest {

    // Wstrzykujemy to samo repozytorium, które siedzi w BaseUnitTest
    private final UserService userService = new UserService(userRepository);

    @Test
    void shouldUpdateUserName() {
        // given 
        thereIsAUser(anUser().withId(1L).withName("Jan").build());

        // when
        userService.updateName(1L, "Jan Kowalski");

        // then
        var updatedUser = userRepository.findById(1L).orElseThrow();
        assertThat(updatedUser.getName()).isEqualTo("Jan Kowalski");
    }
}
```

### Podsumowanie

Stosując połączenie In-Memory, Ability oraz Base Class (nasza klasa `BaseUnitTest`), przestajemy walczyć z narzędziami,
a zaczynamy wspierać proces dostarczania wartości. Udało nam się osiągnąć trzy kluczowe cele:

- **Izolacja**: Dzięki `@BeforeEach` w klasie bazowej każdy test startuje z pustą bazą. Eliminuje
  to błędy wynikające z wyciekania danych między testami, co jest zmorą dużych zestawów testowych.
- **Jedno źródło prawdy**: Zarówno `thereIsAUser` (Given), `userService.updateName` (When), jak i asercja (Then) operują
  na tej samej instancji `InMemoryUserRepository`. Nie musisz niczego konfigurować ręcznie – to, co zapiszesz w Given,
  jest fizycznie dostępne w When i weryfikowalne w Then.
- **Łatwiejszy debug**: Największą różnicę odczujesz, gdy test... nie przejdzie. W świecie Mockito często kończysz z
  enigmatycznym komunikatem Wanted but not invoked. Tutaj, zamiast debugować czeluści frameworka, po prostu stawiasz
  breakpoint w metodzie `updateName` i robisz Step Into.


### Co dalej?

Mamy już unity, które działają błyskawicznie i nie kłamią. W kolejnej części zajmiemy się testami integracyjnymi.

Dowiesz się:

- Jak nie wpaść w pułapkę przeładowywania niepotrzebnie kontekstu Springa i dlaczego adnotacja `@DirtiesContext` to
  jednak raczej Twój wróg niż przyjaciel.
- Dlaczego `@SpyBean` to zaproszenie do kłopotów i jak go unikać.
- Jak zaprząc `Testcontainers` do pracy tak, aby testy integracyjne były niemal tak przyjemne i stabilne, jak nasze
  dzisiejsze unity.