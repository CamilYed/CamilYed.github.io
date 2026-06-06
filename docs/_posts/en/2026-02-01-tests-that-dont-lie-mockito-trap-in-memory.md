---
layout: post
title: "Tests That Don't Lie, Part 2: The Mockito Trap and In-Memory Implementations"
lang: en
ref: modern-testing-practices-2
date: 2026-01-31
tags: [ java, testing, mockito, software-architecture ]
permalink: /en/tests-that-dont-lie-part-2/
---

In the previous part, we focused on how to write tests that are simply pleasant to read. But readability is only half the
battle. You can have the most beautifully written Given-When-Then section that... checks absolutely nothing.

Today we will talk about trust in our tests. Because the worst kind of test is one that gives you a sense of safety,
even though the code underneath does something completely different from what the test suggests.

### 1. The trap of testing implementation (White Box)

I have noticed that in many projects Mockito is added to tests "automatically". We generate a test class, mock all
dependencies, and done. Few people ask themselves then: **why am I actually using this mock?**

Imagine a simple service for updating user data.

☹️ **Sad code:**

```java
@Test
void shouldUpdateUserName() {
    // given
    var userId = 1L;
    var user = new User(userId, "Jan");

    // We have to "feed" the mock so the test can even start
    when(userRepository.findById(userId)).thenReturn(Optional.of(user));

    // when
    userService.updateName(userId, "Jan Kowalski");

    // then
    // We only check a technical method call.
    // Do we know whether the name was actually changed in the object before saving?
    // This test will say "YES" even if the service sends old data to save().
    verify(userRepository).save(any(User.class));
}
```

This test lies to you. It only checks whether the `save` method was called. If a developer mixes up fields and assigns
the new value to a completely different field in production code, or skips the assignment entirely, this test will still
be green. Instead of testing business behavior, which is changing the name, you test a technical library call.

Okay, but someone may notice that we can still verify the object state and try to use `ArgumentCaptor` for update logic.

☹️ Even sadder code:

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
    // This is where the trouble begins. We expose implementation details.
    verify(userRepository).save(userCaptor.capture());
    var savedUser = userCaptor.getValue();

    assertThat(savedUser.getName()).isEqualTo("Jan Kowalski");
}
```

So, success? Not exactly. We have just entered `White Box Testing` mode. Tests become fragile (`Fragile tests`) because:

- Refactoring becomes painful: change `save()` to `saveAll()` and the test blows up, even though the business logic still
  works.
- You test "how", not "what": you care whether a specific line of code was called, not what the result is for the user.
- Sonar lies: reports show line coverage, but you did not test those lines — you only executed them in an artificial
  environment.

#### Solution: In-Memory implementation

Instead of fighting Mockito, let's treat the service as a black box. We need something that pretends to be a database but
works in memory. A `ConcurrentHashMap` under the repository is the simplest approach.

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

A state-based test could look like this. Now the test does not need any `verify`. We simply execute the action and check
whether the state in the "database" is correct.

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

#### What about DSL?

Remember the first part? We can use those patterns to prepare the initial state even more cleanly. Instead of manually
calling `userRepository.save()` in the given section, we will use our "ability".

Happy code:

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

Is this still White Box? Someone might say: "Wait, but in the assertion you call the repository!". No. The difference is
fundamental:

- In Mockito (Interaction): You ask, "Did you call the save method?". If a developer changes the way the save works, the
  test fails.
- In-Memory (State): You ask, "System, no matter how you did it, does this user have a new name?".

In the `Black Box` approach, we treat the Service + InMemoryRepo pair as one black box. We do not care how many times the
service "talked" to the repository. We care about the final effect.

#### How it could look in the end

You may wonder: where does `Ability` get the repository from and is it definitely the same instance that the service
uses? This is the key point. For this to work, we need one source of truth.

The best way is to use interfaces with default methods.

```java
public interface UserAbility {
    UserRepository userRepository(); // Provider method

    default void thereIsAUser(UserBuilder user) {
        userRepository().save(user);
    }
}

// builder in another package, for example com.ourdomain.testing.dsl.builders

public class UserBuilder {
    private Long id = 1L; // Default ID
    private String name = "Jan"; // Default name
    // ... other fields

    public static UserBuilder anUser() {
        return new UserBuilder();
    }

    public UserBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public UserBuilder withName(String name) {
        this.name = name;
        return this;
    }

    public User build() {
        return new User(id, name);
    }
}
```

Let's create a base class for unit tests to hide technical implementation details of our DSL.

```java
public abstract class BaseUnitTest implements UserAbility {
    // One shared instance for the service, assertions, and all Abilities
    protected final InMemoryUserRepository userRepository = new InMemoryUserRepository();

    @Override
    public UserRepository userRepository() {
        return userRepository;
    }

    @BeforeEach
    void clearDatabase() {
        userRepository.clear(); // Important for isolating tests: clean in-memory database state before each test
    }
}
```

Thanks to this, your test class simply extends `BaseUnitTest` and can use our In-Memory implementation. Here is what the
final test class looks like. Notice how little technical noise is left. We focus only on business behavior.

```java
class UserServiceTest extends BaseUnitTest {

    // We inject the same repository that lives in BaseUnitTest
    private final UserService userService = new UserService(userRepository);

    @Test
    void shouldUpdateUserName() {
        // given
        thereIsAUser(anUser().withId(1L).withName("Jan"));

        // when
        userService.updateName(1L, "Jan Kowalski");

        // then
        var updatedUser = userRepository.findById(1L).orElseThrow();
        assertThat(updatedUser.getName()).isEqualTo("Jan Kowalski");
    }
}
```

### Summary

By combining In-Memory, Ability and a Base Class, our `BaseUnitTest`, we stop fighting the tools and start supporting the
process of delivering value. We managed to achieve three key goals:

- **Isolation**: Thanks to `@BeforeEach` in the base class, each test starts with an empty database. This eliminates
  errors caused by data leaking between tests, which is a nightmare in large test suites.
- **One source of truth**: `thereIsAUser` (Given), `userService.updateName` (When), and the assertion (Then) all operate
  on the same `InMemoryUserRepository` instance. You do not have to configure anything manually — what you save in Given
  is physically available in When and verifiable in Then.
- **Easier debugging**: You will feel the biggest difference when a test... fails. In the Mockito world, you often end up
  with an enigmatic "Wanted but not invoked" message. Here, instead of debugging the depths of the framework, you simply
  put a breakpoint in the `updateName` method and step into it.

### What is next?

We now have unit tests that run fast and do not lie. In the next part, we will deal with integration tests.

You will learn:

- How not to fall into the trap of unnecessarily reloading the Spring context and why the `@DirtiesContext` annotation is
  probably your enemy rather than your friend.
- Why `@SpyBean` is an invitation to trouble and how to avoid it.
- How to make `Testcontainers` work so that integration tests are almost as pleasant and stable as today's unit tests.
