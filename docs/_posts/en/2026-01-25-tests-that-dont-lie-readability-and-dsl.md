---
layout: post
title: "Tests That Don't Lie, Part 1: Readability and DSL"
lang: en
ref: modern-testing-practices
date: 2026-01-25
tags: [ java, testing, spring-boot, clean-code ]
permalink: /en/tests-that-dont-lie/
---

In this article, I want to share how I approach writing tests that are not there only to increase so-called
code coverage, but give me confidence that after deploying a new feature, it works according to the original
assumptions.

## Readability

Let's start with the basics. If a test does not clearly communicate what is being tested and why it failed, then all the
technology around it becomes unnecessary weight. Many times, while reviewing tests during code review, I really have to
make an effort to understand what they are actually testing, without trusting the test name too much.

<!--more-->

### 1. Given-When-Then structure

Let's look at the first example. It is not complex at all, but already at the start it makes us work harder to extract
what is the test input, what behavior we are testing, and what assertion we check at the end.

☹️ **Sad code:**

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

Simple code separators, for example comments, make it much easier at first glance to understand where we create the
input setup — `// given`, what we test — `// when`, and what we verify — `// then`.

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

As our tests become more realistic, the data preparation and assertion sections can grow. Instead of creating one huge
block of code under `// given`, it is worth using the helper word `// and`. It lets us logically group operations, for
example separate creating a user from setting permissions or preparing database state.

Let's look at a slightly more complex example:

☹️ **Sad code:**

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

Even in such a small test, we start squinting to understand what is background and what is the actual action. Using the
`and` structure significantly improves readability:

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

**Is that it?**

Even though the code above looks much better than a wall of text, there is still a lot of technical noise here: manually
setting fields, setters, and technical assertions one after another.

In the next sections, we will see how patterns such as Test Data Builder and Custom Assertions can make the same test
look almost like sentences in natural language.

---

### 2. Test Data Builder

Let's return to the "sad code" from the section above. Why does it actually hurt to look at it? Because every time we
want to create a user, we have to call a constructor with all fields or a set of setters. This creates a lot of
information noise. Not all data set in the constructor has any impact on the assertion result. What is more, if the
constructor is extended with one more argument, our tests require changes everywhere that constructor is called. This is
when we deal with `fragile tests`.

The solution to this situation is the **Test Data Builder** pattern described
by [Nat Pryce](http://www.natpryce.com/articles/000714.html). The idea is to create a helper class that has reasonable
default values for all fields. In the test itself, we override only the parameters that are important for a given
scenario.

Before we move to the Builder implementation, let's see what the `// given` section looks like when our domain becomes
richer. Let's assume that in order to save a user in the database, we must satisfy several technical requirements:
address, contact details, dates, permissions.

In a deactivation test, these details are only background, but in the code they take the foreground:

☹️ **Sad code:**

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

    // and - preparing technical logs that must exist in the system
    var initialLog = new AuditLog("INITIAL_CREATION", user.getId(), LocalDateTime.now(), "SYSTEM");
    auditRepository.save(initialLog);

    // when
    user.deactivate("User requested");
    service.update(user);

    // then - we only verify the status field
    var updated = repository.findById(user.getId());
    assertThat(updated.getStatus()).isEqualTo("Inactive");
}
```

We have as many as 8 lines of code only to prepare the object for the test. Does any of these parameters affect whether
the administrator is correctly deactivated? Of course not. Since these technical details are irrelevant from the point of
view of the deactivation business logic, they should be hidden.

This is exactly where `Test Data Builder` helps. It allows us to define reasonable default values for all required fields
in one place. What is more, we can go one step further and compose builders. If our `User` has an `Address`, and the
address does not matter in a given test, the user builder simply uses the default address builder.

Let's look at the implementation:

```java
public class UserBuilder {
    private String name = "Jan";
    private String lastName = "Kowalski";
    private String status = "Inactive";
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

What do we gain?

🙂

```java

@Test
void shouldDeactivateAdminUserAndLogEvent() {
    // given
    var user = aUser()
            .withRole("ADMIN")
            .build();

    // rest of the code...
}
```

As you can see, we have hidden all irrelevant information noise in the given section, where we focus only on the user's
role, because that is what matters in this test.

---

### 3. Assertions - the most common mistakes

Now that we have perfectly prepared input data, we must make sure that the test result actually tells us something.
There are two popular practices that give a false sense of safety.

#### 3.1 Shared variables

It is very tempting to define a value once, for example a user's name, and use it both in the `// given` and `// then`
sections. This is a mistake. If you accidentally change the value at the beginning of the test, the assertion at the end
will still be green, even though the system may behave incorrectly.

☹️ **Sad code:**

```java
var expectedName = "Jan"; // If you change this to "Anna"...
var user = aUser().withName(expectedName).build();

// rest of the test ...

assertThat(result.getName()).isEqualTo(expectedName); 
// ...the test still passes!
```

To make the code above resistant to this kind of situation, it is enough to use string literals.

🙂

```java
var user = aUser().withName("Jan").build();

// rest of the test ...

assertThat(result.getName()).isEqualTo("Jan");
```

#### 3.2 DTO classes in assertions

In API tests, it is common to see a practice where the response from an endpoint is mapped to a DTO class representing
the JSON response.

☹️ **Sad code:**

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

Why do I consider this an unsafe practice? If you rename a field in the `UserResponse` class, for example from `fullName`
to `name`, the IDE refactoring will automatically update this name in the test as well. The result is that the test still
passes, but the endpoint contract is already broken. This is a common example of a `False Positive` test.

In integration tests, it is better to check the raw response, for example as a String or a map, or use a library such as
JsonPath, which looks directly into the JSON structure.

🙂

```java

@Test
void shouldGetUserDetailsAndValidateContract() {
    // omitted code, test setup that puts the user in the database ...

    // when
    ResponseEntity<String> response = restTemplate.getForEntity("/users/1", String.class);

    // then
    assertThat(JsonPath.read(response.getBody(), "$.fullName")).isEqualTo("Jan Kowalski");
}
```

---

### 4. Custom Assertion

In the `// then` section, I often see code that tries to verify the system state after a complex process. Instead of a
clear signal, we get logic, loops, and manual data extraction.

☹️ **Sad code:**

```java
// then
var logs = auditRepository.findByUserId(user.getId());

// We must check whether anything came back at all
assertNotNull(logs);
assertEquals(3, logs.size());

// We look for a specific deactivation log among many others
AuditLog deactivationLog = null;
for (AuditLog log : logs) {
    if ("USER_DEACTIVATED".equals(log.getEventName())) {
        deactivationLog = log;
    }
}

// We check details - a lot of technical assertions
assertNotNull(deactivationLog, "The deactivation log should exist!");
assertEquals("PROCESSED", deactivationLog.getStatus());
assertEquals("AUTH_SERVICE", deactivationLog.getSystemName());
assertEquals("ADMIN_123", deactivationLog.getActorId());
assertTrue(deactivationLog.getTimestamp().isAfter(LocalDateTime.now().minusMinutes(1)));

// And we also check the state of the remaining logs
logs.forEach(l -> assertEquals("SUCCESS", l.getDeliveryStatus()));
```

Why do I consider this approach bad?

1. A loop in a test: If you have a for or if in a test, you are effectively writing an algorithm. And algorithms can be
   wrong. Do we now need a test for the test?
2. Low-level details: When reading this, you have to analyze how the `for` works, how we compare Strings, and whether
   `assertNotNull` is in the right place. The business intention ("the user was deactivated and this was recorded in the
   audit log") disappears in a thicket of technical instructions.
3. Domino effect: If the log structure changes, you have to fix these 15 lines of code in every test that checks it.

Solution: A custom assertion class, internally using for example the `AssertJ` library:

```java
public class UserAssert extends AbstractAssert<UserAssert, User> {

    public UserAssert hasAuditLog(String eventName) {
        isNotNull();
        var logs = TestBeanProvider.getBean(AuditRepository.class).findByUserId(actual.getId());

        // AssertJ does the loop for us and prints a readable error if it does not find the element
        assertThat(logs)
                .extracting(AuditLog::getEventName)
                .contains(eventName);

        return this;
    }
    // ... rest of the methods
}
```

Another advantage is a more precise message when the assertion fails. We do not get a generic, meaningless message like
`expected true but was false`, but for example:
`Expected logs to contain 'USER_DEACTIVATED' but found ['INITIAL_CREATION', 'LOGIN_SUCCESS']`

Example usage:

```java
// then
assertThat(user)
.isDeactivated()
.hasAuditLog("USER_DEACTIVATED")
.isProcessedBy("AUTH_SERVICE") 
.issuedBy("ADMIN_123");
```

---

### 5. Domain Specific Language

Can we go even further and try to make the test resemble the actual business requirements, which should be agnostic to
the technologies used? After all, why should the business care whether we store user data in Mongo or in a relational
database? Secondly, new people joining the project can more easily absorb domain knowledge. We can move to a level where
tests not only confirm that our system works according to certain rules, but also become living business documentation,
because the truth is in the code, not in requirements written on `Confluence`.

#### 5.1 Interface with a default implementation as the basic building block

Instead of calling a repository, the test "has the ability" to manage users. We will use Ability interfaces for that.
They let us "inject" behavior into the test without cluttering it with `@Autowired` annotations or technical
infrastructure code.

```java
interface UserAbility {
    // We hide Spring and the database
    default User thereIs(UserBuilder builder) {
        var user = builder.build();
        return TestBeanProvider.getBean(UserRepository.class).save(user);
    }

    // Domain: what happens in the system
    default void userIsDeactivated(User user, String reason) {
        var service = TestBeanProvider.getBean(UserService.class);
        user.deactivate(reason);
        service.update(user);
    }
}

interface AuditAbility {
    default void thereIsAnInitialLogFor(User user) {
        var log = new AuditLog("INITIAL_CREATION", user.getId(), LocalDateTime.now(), "SYSTEM");
        TestBeanProvider.getBean(AuditRepository.class).save(log);
    }
}
```

#### 5.2 Example usage

```java
class UserDeactivationTest implements UserAbility, AuditAbility {

    @Test
    void shouldDeactivateAdminUserAndLogEvent() {
        // given
        var admin = thereIs(aUser().withRole("ADMIN").withStatus("Active"));

        // and
        thereIsAnInitialLogFor(admin);

        // when
        userIsDeactivated(admin, "User requested");

        // then
        assertThatUser(admin)
                .isDeactivated()
                .hasDeactivationReason("User requested")
                .hasAuditLog("STATUS_CHANGE")
                .isProcessedBy("AUTH_SERVICE");
    }

}
```

And here is how we can easily pull beans from Spring in tests for the needs of `Ability` interfaces, for example:

```java

@Component
public class TestBeanProvider implements ApplicationContextAware {
    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        context = applicationContext;
    }

    public static <T> T getBean(Class<T> beanClass) {
        return context.getBean(beanClass);
    }
}
```

It is worth remembering that the solution above, based on the Spring context, is dedicated to integration tests. In pure
unit tests, Ability interfaces can simply receive dependencies through a constructor or use In-Memory implementations,
which I will talk about in the next part.

---

#### 5.3 What do we gain from this abstraction?

The test above is no longer code that only a programmer can understand, but a readable description of system behavior.
Moving to this level of abstraction brings concrete architectural benefits:

- Technology agnosticism: If a year from now the decision is made to change the database from relational to document, the
  test scenario itself remains untouched. You only change the implementation inside `UserAbility`, and the business logic
  of the test still correctly verifies the system.

- Protection against outdated documentation: Documentation in Confluence or Jira becomes outdated a second after a task
  is closed. A test written this way is an `Executable Specification` — a specification that cannot lie, because if it
  becomes outdated, the system simply will not pass the `CI/CD` process.

- Faster onboarding: A new developer on the team does not have to analyze which repositories and services are needed to
  prepare the database state. They use ready-made "abilities" (`Abilities`), so they learn business processes instead of
  focusing on technological information noise.

- Ubiquitous Language: The test code starts to sound like a conversation with the `Product Owner`. "There is an admin",
  "User is deactivated" — these are terms everyone understands, not only developers.

---

### Summary (Part 1)

A good testing culture is not just a high percentage in a code coverage report. It is primarily trust in your own
solution and the ease of evolving it. In this part, we focused on readability and communication. We moved:

From technical noise and a "wall of text", through patterns such as `Test Data Builder` and `Custom Assertions`.

All the way to creating our own Domain DSL, which makes a test become a business specification, not just a piece of code
understood by developers.

Remember: if a test is hard to read, nobody will maintain it. **And a dead test is worse than no test at all**.

---

### What is next

Readability is only half the battle. Even the nicest test will be useless if every second build in the pipeline is red
for no clear reason (`flaky tests`), runs slowly, or starts lying to us because we mocked everything around it with for
example Mockito, and debugging it brings no solution.

In the next part, we will talk about:

- **Why I avoid Mockito and test "Black Box"**: I prefer to test real implementations, often with In-Memory versions for
  unit tests, instead of writing tests that only verify whether we called a mock.

- **Silent performance killers:** Why annotations such as `@DirtiesContext` and `@SpyBean` are evil because they make
  Spring reload the context over and over again and suddenly make the build much longer.

- **Controlling time:** How to stop fighting `LocalDateTime.now()` and start using your own `Clock Provider`, so date
  tests become predictable.

- **Asynchronicity:** How to get rid of `Thread.sleep()` and replace it with `Awaitility`, so the test does not wait even
  one second too long and becomes more resilient to the passage of time.

- **Isolation and no state**: Why I use a `Database Cleaner` instead of `@Transactional` annotations on test classes.

- **Infrastructure**: A short introduction to Testcontainers and Wiremock, or how to test with a real database and API
  without pretending that "it works on H2 on my machine".
