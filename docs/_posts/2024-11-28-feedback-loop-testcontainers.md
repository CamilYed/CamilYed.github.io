---
layout: post
title: "Shortening the Feedback Loop with TestContainers – A Strategic Advantage"
date: 2024-11-28
tags: [ development, testing ]
---

In today’s fast-paced software development landscape, organizations are constantly seeking ways to streamline processes,
improve code quality, and deliver value faster. One often overlooked yet crucial aspect of achieving these goals is the
**feedback loop** during development and testing.

Traditionally, many teams have operated with outdated practices that hinder their ability to respond quickly to issues.
Let’s explore how embracing tools like **TestContainers** can revolutionize testing workflows, save time and resources,
and have a strategic impact on your organization.

<!--more-->

### The Problem with Traditional Testing Practices

Many organizations suffer from inefficient testing practices that extend the feedback loop and delay the detection of
critical issues. These challenges often include:

1. **Delayed Testing Feedback**  
   Developers push changes, which are then handed off to QA. After a few days, testers report issues, sometimes with
   vague or incomplete descriptions. By the time the developer revisits the problem, context has been lost, and
   debugging becomes time-consuming.

2. **Weak or Unrealistic Tests**
    - Developers use mock databases or stubs instead of real environments, leading to gaps in test coverage.
    - Critical bugs slip through because the test environment does not mimic production conditions.

3. **Shared or Local Databases**
    - Teams rely on shared test databases, creating conflicts when multiple developers run tests simultaneously.
    - Alternatively, developers use local databases, which may be out of sync with production or other environments.

4. **Slow CI/CD Pipelines**  
   Real integration tests with production-like databases often only run in CI/CD pipelines. Developers must wait for the
   pipeline to finish to get feedback, leading to delays in identifying and fixing issues.

---

### Enter TestContainers: Modernizing Testing Workflows

[TestContainers](https://testcontainers.com) is a library that simplifies running integration tests by spinning up 
**real containers** for services like databases, queues, and more. With **Docker** as its backbone, TestContainers
creates ephemeral, isolated environments for each test.

Here’s how TestContainers addresses the pain points of traditional testing:

1. **Instant, Reliable Feedback**  
   By using TestContainers, developers can run **real integration tests locally** before pushing changes. Each test
   spins up a fresh, production-like database (or other service) in a container, ensuring consistent and reliable
   results. This shortens the feedback loop and empowers developers to catch and fix issues immediately.

2. **Realistic Test Environments**  
   Tests use real services (e.g., PostgreSQL, MySQL, Redis) running in containers. This eliminates the guesswork and
   inaccuracies of using mocks or shared databases. Your tests become more robust and aligned with production behavior.

3. **No Shared Resources**  
   Each test gets its own containerized environment, removing conflicts caused by shared or local databases. Developers
   can run tests independently without worrying about data corruption or inconsistencies.

4. **Reduced Pipeline Bottlenecks**  
   Since developers can run comprehensive integration tests locally, fewer bugs reach CI/CD pipelines. This reduces
   pipeline load and prevents unnecessary delays in detecting issues.

---

### The Cost of Waiting: Feedback Delays Add Up (Monthly Perspective)

#### Assumptions:
- **100–200 integration tests per feature**: Developers typically write between 100 and 200 tests for a significant
  feature each month.
- **4 code pushes per day**: On average, a developer pushes code to the repository **4 times a day**.
- **15-minute pipeline duration**: Each CI/CD pipeline run takes approximately **15 minutes** to complete.
- **20 workdays per month**: We assume a standard work month of **20 workdays**.
- **QA feedback delay**: The delay for receiving QA feedback is typically **2–3 days** for each feature with traditional
  practices.

---

#### Monthly Waiting Time with Traditional Practices

1. **Waiting on CI/CD Pipelines**
    - For each push: **15 minutes** waiting time.
    - **4 pushes/day** × **15 minutes/push** = **1 hour/day** spent waiting.
    - Over a month: **1 hour/day × 20 days** = **20 hours** spent waiting per developer.

2. **QA Feedback Delay**
    - Writing **100–200 integration tests** takes **3 days** on average.
    - QA feedback delays stretch the timeline to **5–6 days**, adding **2–3 days of idle waiting**.
    - If a developer writes 2 such features in a month, the total QA waiting time is **4–6 days** (32–48 hours).

3. **Total Waiting Time (Traditional Practices)**
    - CI/CD: **20 hours**
    - QA: **32–48 hours**
    - **Total: 52–68 hours per developer/month**.

---

### Time Savings with TestContainers

1. **Eliminating CI/CD Waiting Time**
   - By using TestContainers, developers can run integration tests locally in seconds. This reduces the need to wait for CI/CD pipeline runs, which usually take around **15 minutes** each.
   - **Saving: 20 hours/month per developer** by eliminating the need to wait for pipeline feedback.

2. **Reducing QA Feedback Delays**
   - TestContainers allow developers to test integrations locally and fix issues before pushing them to QA, significantly reducing the reliance on manual QA feedback.
   - Assuming that QA delays are cut by **50%** (from 3 days to 1-2 days), developers can save **16–24 hours per month**.

3. **Total Savings**
   - CI/CD waiting time saved: **20 hours/month**
   - QA feedback delay reduced: **16–24 hours/month**
   - **Total: 36–44 hours saved per developer per month** (approximately **1 full workweek**).

---

### Strategic Impact of Shortening the Feedback Loop

#### Across a Team of 5 Developers:

- **Traditional waiting time**: **260–340 hours/month** for the team.
- **Savings with TestContainers**: **180–220 hours/month**.
- **Result**: The team gains **4–5 extra productive weeks per month**.

#### Financial Perspective:

- Assume a developer’s cost is **$50/hour**.
- **Traditional cost of waiting**: **$13,000–$17,000/month** for a 5-person team.
- **Savings with TestContainers**: **$9,000–$11,000/month**.

---

### A Practical Example: Integration Testing with TestContainers

To illustrate the power of TestContainers in real-world scenarios, let’s look at an example from the
[Currency Exchange API repository](https://github.com/CamilYed/currency-exchange-api). Below is a sample integration
test
demonstrating how TestContainers can facilitate realistic and robust testing.

#### The Code

Below is a sample Kotlin test case that demonstrates how TestContainers can be used to validate a currency exchange operation. In this example, TestContainers creates a real PostgreSQL database in a Docker container to simulate a production-like environment for integration testing:

```kotlin
@Test
fun `should exchange PLN to USD successfully`() {
    // given
    val accountId = thereIsAnAccount(
        aCreateAccount()
            .withInitialBalance("1000.00")
    )

    // and
    currentExchangeRateIs("4.0")

    // when
    val response = exchangePlnToUsd(
        anExchangePlnToUsd()
            .withAccountId(accountId)
            .withAmount("400"),
    )

    // then
    expectThat(response)
        .isOkResponse()
        .hasPlnAmount("600.00")
        .hasUsdAmount("100.00")
}
```

#### What This Test Does

- **Setup (`given`)**: Creates an account with an initial balance of 1000 PLN.
- **Precondition (`and`)**: Sets the current exchange rate (PLN to USD) at 4.0.
- **Action (`when`)**: Executes a request to exchange 400 PLN to USD for the given account.
- **Validation (`then`)**: Asserts that:
    - The response is successful.
    - The remaining PLN balance is 600.00.
    - The USD balance after exchange is 100.00.

#### The Power of TestContainers

This test runs against a real PostgreSQL database instance provided by TestContainers. By isolating the test environment
with containers, it ensures accuracy, eliminates data conflicts, and mimics production behavior seamlessly.

---

### Watch the Test in Action

Below is a video showing the execution of this integration test using TestContainers. You’ll see the database container being spun up, the test being executed, and the container being cleaned up automatically after the test completes:

<video muted autoplay controls style="width: 100%; border-radius: 8px;">
  <source src="/assets/videos/testcontainers.mp4" type="video/mp4" />
  Your browser does not support the video tag.
</video>

#### Why This Matters

This is a genuine integration test, leveraging a live database environment powered by TestContainers.
It demonstrates how you can achieve:

- **Accuracy**: Tests run in production-like conditions.
- **Isolation**: Each test operates in a clean, independent environment.
- **Efficiency**: Automated container setup and teardown save time and effort.

Start leveraging TestContainers in your projects to experience the benefits of faster, more reliable integration
testing.

## Conclusion

Shortening the feedback loop is a critical step toward improving your development workflow and achieving strategic
business goals. Tools like **TestContainers** empower developers to test in realistic environments, catch issues early,
and save valuable time and resources.

By modernizing your testing approach, you not only enhance the quality of your software but also foster a more agile,
innovative, and competitive organization.

Ready to transform your testing workflow? Start integrating TestContainers today, and see the impact it can have on your
team and organization.

*For more examples and practical guides, check out
the [Currency Exchange API repository](https://github.com/CamilYed/currency-exchange-api).*