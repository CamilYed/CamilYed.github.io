---
layout: post
title: "Shortening the Feedback Loop with TestContainers – A Strategic Advantage"
lang: en
ref: testcontainers-feedback-loop
date: 2024-11-28
tags: [development, testing]
permalink: /en/testcontainers-feedback-loop/
---

Modernizing software architecture in a financial institution often feels like an uphill battle. Strict security
policies, siloed teams, and legacy processes create obstacles that hinder agility, innovation, and collaboration. As the
sole developer in such an environment, you may feel overwhelmed trying to align the technical needs of the system with
the rigid structures of the organization.
<!--more-->
This post explores how **socio-technical alignment**—harmonizing the structure of your organization with its technical
architecture—can help overcome these challenges. Using a hypothetical yet relatable scenario, we'll discuss practical
steps to break down silos, streamline testing workflows, and foster collaboration.

---

## The Problem: Siloed Teams and Legacy Processes

Imagine this scenario:

- You work in a **financial institution** with strict security requirements and highly compartmentalized teams.
- The **database team (DBAs)** controls every aspect of the schema, requiring all changes to go through them, while
  rejecting tools like Liquibase that could automate versioning.
- **Testing integration** is complicated: you can’t use local admin privileges, and running tools
  like [Testcontainers](https://www.testcontainers.org/) is off-limits unless pre-approved images are supplied by the
  DBAs.
- **Network and security teams** operate independently, often enforcing policies that limit your ability to experiment
  or innovate.
- There’s **no dedicated DevOps or Platform Team**, leaving infrastructure management and automation entirely on you.

The result? Friction, slow feedback loops, and inefficiency, as every small change requires navigating bureaucratic
processes.

---

## Key Challenges

Let’s break down the core challenges:

1. **Database Control**: The database team operates as a silo, managing schema changes without collaboration or
   automation, leading to delays and inefficiencies.
2. **Testing Restrictions**: Without access to tools like Testcontainers or Docker, it’s challenging to create realistic
   test environments, forcing reliance on manual or incomplete tests.
3. **Organizational Disconnect**: Teams responsible for network, security, and databases work independently, with
   conflicting priorities that make cross-functional collaboration difficult.
4. **No DevOps Support**: Without a DevOps or Platform Team, infrastructure automation, CI/CD pipelines, and testing
   environments are non-existent or entirely manual.

---

## A Path Forward: Bridging Silos with Socio-Technical Alignment

To address these challenges, we need to align the organization’s structure with its technical architecture. Here’s how:

### 1. Build Bridges Between Teams

Silos thrive in environments where teams work in isolation. To break this cycle:

- **Host collaborative workshops**: Use approaches like [EventStorming](https://eventstorming.com/) to bring DBAs,
  security teams, and developers together. Focus on shared understanding of the system and identify overlapping goals.
- **Negotiate incremental changes**: For example, ask DBAs to provide pre-configured Docker images of the database that
  can be used for local and CI/CD testing. This approach respects their control while enabling development speed.
- **Communicate impact**: Share examples of how delays in testing or schema changes affect overall delivery timelines,
  making the case for tighter collaboration.

### 2. Enable Realistic Testing with Testcontainers

Testing in isolated environments is critical for fast feedback loops, even in a restricted environment:

- **Leverage pre-approved images**: If you can’t run Testcontainers with arbitrary databases, request the DBAs provide
  Docker images with the exact schema and stored procedures you need.
- **Integrate testing in CI/CD**: Use a controlled pipeline to run Testcontainers-based tests in an environment where
  permissions are less restrictive.
- **Document and demonstrate value**: Show how such tests catch bugs earlier and reduce manual testing effort.

### 3. Advocate for a Platform Team

In the long term, your organization needs a **Platform Team** to manage infrastructure, pipelines, and testing
environments. Their responsibilities would include:

- Providing standardized tools and environments, such as containerized databases or managed CI/CD pipelines.
- Automating infrastructure setup and deployments to reduce manual effort.
- Supporting developers with self-service tools for common tasks (e.g., database migrations).

A Platform Team acts as an enabler, allowing Stream-Aligned Teams to focus on delivering business value without being
bogged down by infrastructure concerns.

### 4. Adopt Team Topologies

The book [Team Topologies](https://teamtopologies.com/) offers a framework for designing effective team structures. For
your case:

- Transition to a **Stream-Aligned Team** responsible for end-to-end delivery of the system.
- Build a **Platform Team** to support development and testing efforts.
- Use **facilitating interactions** like Embedded Collaboration or Enabling Teams to improve cross-functional
  communication.

### 5. Focus on Incremental Automation

Even without full DevOps support, there are ways to improve:

- **Automate what you can**: Set up simple scripts for repetitive tasks like schema testing or environment setup.
- **Build small feedback loops**: Gradually introduce automated tests or lightweight CI pipelines to demonstrate value.
- **Show success stories**: Highlight instances where automation saved time or improved quality.

---

## The Happy Ending: Faster Feedback and Better Collaboration

By implementing these strategies, your organization can:

- Reduce feedback loop time by enabling realistic tests in controlled environments.
- Improve collaboration by aligning team goals and responsibilities.
- Lay the groundwork for a sustainable, socio-technical system that supports agility and innovation.

---

## Conclusion

Modernizing architecture in a siloed organization isn’t easy, but it’s achievable with a clear strategy. By fostering
collaboration, adopting team topologies, and making incremental technical improvements, you can bridge the gap between
teams and technology.

The result? A more efficient, aligned, and effective organization ready to meet the demands of modern software
development.
