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

## 1. Pułapka testowania implementacji (White Box)

Zauważyłem, że w wielu projektach Mockito dodaje się do testów "z automatu". Generujemy klasę testową, mockujemy
wszystkie zależności w konstruktorze i cyk – robota zrobiona. Mało kto zadaje sobie wtedy pytanie: **po co ja właściwie
tego mocka używam**?

Wyobraź sobie prosty serwis, który przed zapisem użytkownika do bazy ma mu nadać domyślną rolę.

☹️ **Smutny kodzik:**

```java

@Test
void shouldSaveUserWithDefaultRole() {
    // given
    var user = new User("Jan");
    // Mockujemy repozytorium
    when(userRepository.save(any())).thenReturn(user);

    // when
    userService.register(user);

    // then
    // Sprawdzamy tylko techniczne wywołanie metody. 
    // Czy wiemy, czy rola faktycznie została przypisana w obiekcie przed zapisem? 
    // Ten test powie "TAK", nawet jeśli serwis przekaże do save() pusty obiekt.
    verify(userRepository).save(any(User.class));
}
```

Przecież, tak naprawdę ten test Cię oszukuje. sprawdza tylko czy zawołaną metodę `save`. Jeśli usuniesz w kodzie linie
odpowiadającą za przypisane roli, to ten test nadal przejdzie! Mcckito przyjmie cokolwiek, co mu przekażesz.
Zamiast testować zachowanie biznesowe (użytkownik ma mieć rolę), testujesz techniczne wywołanie biblioteki.

No dobra, ale ktoś zauważy, że możemy jednak zweryfikować przypisanie roli i wtedy spróbuje użyć kolejnych features,
które
oferuje Mockito np. `ArgumentCaptor`.

☹️ **Jeszcze bardziej smutny kodzik:**

```java

@Test
void shouldSaveUserWithDefaultRole_CaptorVersion() {
    // given
    var user = new User("Jan");
    var userCaptor = ArgumentCaptor.forClass(User.class);

    // when
    userService.register(user);

    // then
    verify(userRepository).save(userCaptor.capture());
    var savedUser = userCaptor.getValue();

    assertThat(savedUser.getRole()).isEqualTo(Role.USER);
}
```

I co? Sukces? No nie do końca. Właśnie weszliśmy w tryb `White Box Testing`, odsłaniamy szczegóły implementacji
co powoduje, że nie testujemy kodu jako czarnej skrzynki - coś było na wejściu, coś się zadziało i coś mamy na wyjściu.
Znowu nasze testy stają się kruche (`Fragile tests`).

Podsumowując, krótko czemu to jest pułapka:

- Refaktoryzacja to ból: Zmieniasz nazwę metody w repozytorium albo zamiast `save()` używasz `saveAll()`? Twój test
  wybucha, mimo że logika biznesowa (nadanie roli) nadal działa poprawnie.
- Testujesz "jak", a nie "co": Twojego testu nie obchodzi wynik. Obchodzi go to, czy wywołałeś konkretną linię kodu,
  gdzieś głęboko w wewnątrz serwisu. To sprawia, że testy odkrywają szczegóły implementacyjne, przesłaniając tym samym,
  co chcemy przetestować, powodując, że musisz skupiać się na wewnętrznych szczegółach zamiast na celu biznesowym, jaki
  chcemy mieć przetestowany.
- Sonar kłamie: Narzędzia do pokrycia kodu (`Code Coverage`) pokazują, że "przetestowałeś" te linie. Ale Ty ich nie
  przetestowałeś – Ty je tylko wywołałeś w kontrolowanym, sztucznym środowisku.