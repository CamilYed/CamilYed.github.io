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

## Czytelność i Sygnał (Ekspresja)

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

W kolejnych sekcjach zobaczymy, jak za pomocą wzorców takich jak Test Data Builder oraz Custom Assertions (zgodnie z
duchem Nat Pryce), możemy sprawić, że ten sam test będzie wyglądał niemal jak zdania w języku naturalnym.