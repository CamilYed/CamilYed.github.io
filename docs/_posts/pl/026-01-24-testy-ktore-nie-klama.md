---
layout: post
title: "Czego nauczyłem się po latach na temat testów? Moje subiektywne Best Practices"
lang: pl
ref: modern-testing-practices
date: 2026-01-24
tags: [ java, testing, spring-boot, clean-code ]
permalink: /pl/testy-ktore-nie-klama/
---

W tym artykule chcę się podzielić jak podchodzę do pisania testów, które nie są wyłącznie po to aby pokryć tzw.
code-coverage, ale dają mi pewność, że po wdrożeniu nowej funkcjonalności - działa ona zgodnie z pierwotnymi
założeniami.

## Czytelność

Zacznijmy od podstaw. Jeśli test nie komunikuje jasno, co jest testowane i dlaczego padł, to cała reszta technologii
staje się niepotrzebnym ciężarem. Wiele razy przeglądając kod dostarczonych testów podczas procesu code review, muszę
naprawdę postarać się co one faktycznie testują nie ufająć nad zbyt samemu opisowi testu.

<!--more-->

### 1. Struktura Given-When-Then

Spójrzmy na pierwszy przykład, który nie jest w cale rozbudowany ale już sprawia, że trzeba nie co bardziej się wysilić
aby wyłuskać co jest na wejściu testu - input, co testujemy - zachowanie, oraz co na końcu sprawdzamy - asercja.

☹️ **Smutny kodzik:**

```java

@Test
void updateTest() {
    User user = new User("Jan", "Active");
    repository.save(user);
    user.changeStatus("Inactive");
    service.update(user);
    User updated = repository.findById(user.getId());
    assertEquals("Inactive", updated.getStatus());
}
```

Proste separatory kodu po przez np. komentarze - sprawiają, że już przy pierwszym spojrzeniu na test jest nam łatwiej
się
połapać gdzie tworzymy setup wejściowy - `// given `, co testujemy `// when`, oraz co weryfikujemy `// then`.

🙂

```java

@Test
void shouldChangeUserStatusToInactive() {
    // given
    var user = new User("Jan", "Active");
    repository.save(user);

    // when
    user.changeStatus("Inactive");
    service.update(user);

    // then
    var updated = repository.findById(user.getId());
    assertThat(updated.getStatus()).isEqualTo("Inactive");
}
```

W miarę jak nasze testy stają się bardziej realistyczne, sekcje przygotowania danych czy asercji mogą się rozrastać.
Zamiast tworzyć jeden wielki blok kodu pod `// given`, warto użyć słowa pomocniczego `// and`. Pozwala to logicznie
pogrupować operacje, np. oddzielić tworzenie użytkownika od ustawiania jego uprawnień lub stanu bazy danych.

Spójrzmy na nieco bardziej rozbudowany przykład:

☹️ **Smutny kodzik:**

```java

@Test
void complexUpdateTest() {
    User user = new User("Jan", "Active");
    user.setAddress(new Address("Warszawa", "Złota 44"));
    user.setRole("ADMIN");
    repository.save(user);
    AuditLog log = new AuditLog("INITIAL_CREATION", user.getId());
    auditRepository.save(log);
    user.changeStatus("Inactive");
    user.setDeactivationReason("User requested");
    service.update(user);
    User updated = repository.findById(user.getId());
    assertEquals("Inactive", updated.getStatus());
    assertEquals("User requested", updated.getDeactivationReason());
    assertNotNull(auditRepository.findByUserIdAndType(user.getId(), "STATUS_CHANGE"));
}
```

Nawet w tak małym teście zaczynamy mrużyć oczy, żeby zrozumieć, co jest tłem, a co akcją. Zastosowanie struktury z `and`
znacznie poprawia czytelność:

🙂

```java

@Test
void shouldDeactivateAdminUserAndLogEvent() {
    // given
    var user = new User("Jan", "Active");
    user.setRole("ADMIN");
    repository.save(user);
    // and
    var initialLog = new AuditLog("INITIAL_CREATION", user.getId());
    auditRepository.save(initialLog);

    // when
    user.deactivate("User requested");
    service.update(user);

    // then
    var updated = repository.findById(user.getId());
    assertThat(updated.getStatus()).isEqualTo("Inactive");
    assertThat(updated.getDeactivationReason()).isEqualTo("User requested");
    // and
    var statusLog = auditRepository.findByUserIdAndType(user.getId(), "STATUS_CHANGE");
    assertThat(statusLog).isNotNull();
}
```

**Czy to już koniec?**

Mimo że powyższy kod wygląda znacznie lepiej niż "ściana tekstu", to wciąż mamy tu sporo szumu technicznego (ręczne
ustawianie pól, settery, techniczne asercje jedna po drugiej).

W kolejnych sekcjach zobaczymy, jak za pomocą wzorców takich jak Test Data Builder oraz Custom Assertions, możemy
sprawić, że ten sam test będzie wyglądał niemal jak zdania w języku naturalnym.

### 2. Test Data Builder

Wróćmy do naszego „smutnego kodzika” z sekcji wyżej. Dlaczego on tak naprawdę kuje w oczy? Bo za każdym razem, gdy
chcemy stworzyć użytkownika, musimy wywołać konstruktor ze wszystkimi polami albo zestaw setterów. To tworzy ogromny
szum informacyjny. Nie wszystkie przecież dane ustawione w konstruktorze mogą wpływać na wynik asercji, co więcej, jeśli
konstruktor zostałby np. rozszerzony o kolejny argument, nasze testy wymagałyby zmian, w każdym miejscu, gdzie ten
konstruktor wywołujemy, mamy wtedy do czynienia z pojęciem `fragile tests`.

Rozwiązaniem na tę sytuację jest wzorzec **Test Data Builder** opisany
przez [Nat Pryce'a](http://www.natpryce.com/articles/000714.html). Ideą jest stworzenie klasy pomocniczej, która posiada
sensowne, domyślne wartości dla wszystkich pól. W samym teście nadpisujemy tylko te parametry, które są kluczowe dla
danego scenariusza.

Zanim przejdziemy do implementacji Buildera, zobaczmy, jak wygląda sekcja `// given` w momencie, gdy nasza domena staje
się bogatsza. Załóżmy, że aby zapisać użytkownika w bazie, musimy spełnić szereg wymagań technicznych: adres, dane
kontaktowe, daty, uprawnienia.

W teście deaktywacji te dane to tylko tło, ale w kodzie zajmują pierwszy plan:

☹️ **Smutny kodzik:**

```java

@Test
void shouldDeactivateAdminUserAndLogEvent() {
    // given
    var address = new Address("Warszawa", "Złota 44", "00-123", "Polska");
    var user = new User("Jan", "Kowalski", "jan.k@example.com", "Active");
    user.setAddress(address);
    user.setRole("ADMIN");
    user.setCreatedAt(LocalDateTime.now());
    user.setLastLogin(LocalDateTime.now().minusDays(1));
    repository.save(user);

    // and - przygotowanie logów technicznych, które muszą być w systemie
    var initialLog = new AuditLog("INITIAL_CREATION", user.getId(), LocalDateTime.now(), "SYSTEM");
    auditRepository.save(initialLog);

    // when
    user.deactivate("User requested");
    service.update(user);

    // then - sprawdzamy tylko pole status
    var updated = repository.findById(user.getId());
    assertThat(updated.getStatus()).isEqualTo("Inactive");
}
```

Mamy tutaj aż 8 linii kodu tylko po to, żeby przygotować obiekt do testu. Czy którykolwiek z tych parametrów ma wpływ na
to, czy administrator zostanie poprawnie zdeaktywowany? Oczywiście, że nie. Skoro te techniczne detale są nieistotne z
punktu widzenia logiki biznesowej deaktywacji, powinny zostać ukryte.

Właśnie tutaj z pomocą przychodzi Test Data Builder. Pozwala on na zdefiniowanie sensownych, domyślnych wartości dla
wszystkich wymaganych pól w jednym miejscu. Co więcej, możemy pójść krok dalej i zastosować kompozycję builderów. Jeśli
nasz User posiada Address, a adres nas w danym teście nie interesuje – builder użytkownika po prostu użyje domyślnego
buildera adresu.

Spójrzmy na implementację:

```java
public class UserBuilder {
    private String name = "Jan";
    private String lastName = "Kowalski";
    private String status = "Active";
    private String role = "USER";
    private String deactivationReason = null;
    private AddressBuilder addressBuilder = AddressBuilder.anAddress();

    private UserBuilder() {
    }

    public static UserBuilder aUser() {
        return new UserBuilder();
    }

    public UserBuilder withRole(String role) {
        this.role = role;
        return this;
    }

    public UserBuilder withStatus(String status) {
        this.status = status;
        return this;
    }

    public UserBuilder withAddress(AddressBuilder addressBuilder) {
        this.addressBuilder = addressBuilder;
        return this;
    }

    public User build() {
        User user = new User(name, lastName, status, role);
        user.setAddress(addressBuilder.build());
        user.setDeactivationReason(deactivationReason);
        return user;
    }
}

public class AddressBuilder {
    private String city = "Warszawa";
    private String street = "Złota 44";

    public static AddressBuilder anAddress() {
        return new AddressBuilder();
    }

    public AddressBuilder withCity(String city) {
        this.city = city;
        return this;
    }

    public Address build() {
        return new Address(city, street);
    }
}
```

Co zyskujemy:

🙂

```java

@Test
void shouldDeactivateAdminUserAndLogEvent() {
    // given
    var user = aUser()
            .withRole("ADMIN")
            .build();

}
```

Jak widać, ukryliśmy cały nieistotny szum informacyjny w sekcji given, gdzie skupiamy się jedynie na roli użytkownika,
to ona ma znaczenie w tym teście.

### 3. Asercje-najczęstsze błędy

Mając już idealnie przygotowane dane wejściowe, musimy zadbać o to, by wynik testu faktycznie o czymś nas informował.
Istnieją dwie popularne praktyki, które dają złudne poczucie bezpieczeństwa.

#### 3.1 Współdzielone zmienne

Bardzo często kusi nas, aby raz zdefiniowaną wartość (np. imię użytkownika) wykorzystać zarówno w sekcji `// given` jak
i
w `// then`. To błąd. Jeśli przez pomyłkę zmienisz wartość zmiennej na początku testu, asercja na końcu nadal będzie
"zielona", mimo że system może zachować się błędnie.

☹️ **Smutny kodzik:**

```java
var expectedName = "Jan"; // Jeśli tu zmienisz na "Anna"...
var user = aUser().withName(expectedName).build();

// dalszy kod testu ...

assertThat(result.getName()).

isEqualTo(expectedName); // ...test nadal przejdzie!
```

Aby uodpornić powyższy kod na możliwe wystąpienie takiej sytuacji, wystarczy użyć literałów tekstowych.

🙂

```java
var user = aUser().withName("Jan").build();

// dalszy kod testu ...

assertThat(result.getName()).

isEqualTo("Jan");
```

#### 3.2 Klasy DTO w asercjach

W przypadku testów API często można spotkać się z taką praktyką, gdzie w teście odpowiedź z danego endpointu jest
mapowana na klasę DTO odpowiadającej reprezentacji w JSON.

☹️ **Smutny kodzik:**

```java

@Test
void shouldGetUserDetails() {
    // when
    ResponseEntity<UserResponse> response = restTemplate.getForEntity("/users/1", UserResponse.class);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    assertThat(response.getBody().getFullName()).isEqualTo("Jan Kowalski");
}
```

Czemu to jest niezalecana praktyka? Jeśli zmienisz nazwę pola w klasie `UserResponse` np. z `fullName` na `name`, IDE
za pomocą refaktoryzacji automatycznie zaktualizuje tę nazwę również w teście. Wynik jest taki, że test nadal
przechodzi,
ale kontrakt endpointu jest już złamany. Jest to częsty przykład testów `False Positive`.

W testach integracyjnych warto sprawdzać surową odpowiedź (np. jako String lub mapę) lub użyć biblioteki JsonPath, która
zagląda bezpośrednio w strukturę JSON-a.

🙂

```java

@Test
void shouldGetUserDetailsAndValidateContract() {
    // pominięty kod, setup testu ustawienie użytkownika w bazie ...

    // when
    ResponseEntity<String> response = restTemplate.getForEntity("/users/1", String.class);

    // then
    assertThat(JsonPath.read(response.getBody(), "$.fullName")).isEqualTo("Jan Kowalski");
}
```

### 4. Custom Assertion
