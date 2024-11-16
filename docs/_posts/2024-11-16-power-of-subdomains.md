---
layout: post
title: "Discover, Decompose, Decoupled – The Power of Subdomains in Domain-Driven Design (DDD)"
date: 2024-11-16
tags: [ddd, subdomains]
---

# Discover, Decompose, Decoupled – The Power of Subdomains in Domain-Driven Design (DDD)

Domain-Driven Design (DDD) has become a powerful tool for modeling complex systems in a way that aligns with the business. One of the key concepts in DDD is the **subdomain**—a partitioning of the business logic that helps teams build maintainable and scalable systems. While many developers and architects understand the basics of domains and subdomains, there’s a lot of potential hidden behind these terms.

In this article, I’ll explore subdomains in detail and introduce a set of heuristics that can guide the discovery of subdomains. Along the way, we’ll discuss how these heuristics can be used in real-world scenarios, and I’ll suggest visual tools that can aid in their application.

<!--more-->

## What is a Domain?

In the context of Domain-Driven Design (DDD), a **domain** refers to the specific area of business that the software is intended to support. For a car-sharing company, the domain covers everything related to providing ride services, managing drivers, handling customers, and ensuring that the service runs smoothly.
<div style="float: right; margin: 0 0 10px 20px; max-width: 300px;">
  <img src="/assets/car-sharing.png" alt="Car Sharing Illustration" style="width: 100%; height: auto;">
</div>

The car-sharing business is complex, involving various processes, stakeholders, and technologies.


In a **car-sharing domain**, there are multiple areas of responsibility, such as:
- Managing and certifying drivers,
- Handling customer bookings,
- Optimizing ride matching between drivers and passengers,
- Processing payments for services,
- Managing vehicle maintenance.

The domain can be further divided into **subdomains** to handle specific aspects of the business, allowing for more manageable system architecture.

## Heuristics for Identifying Subdomains

### 1. Conway’s Law (Organizational Structure Heuristic)

<div style="float: right; margin: 0 0 10px 20px; max-width: 300px;">
  <img src="/assets/conway-law.png" alt="Car Sharing Illustration" style="width: 100%; height: auto;">
</div>

Conway's Law suggests that the system structure reflects the communication structure of the organization.
In a car-sharing company, departments such as **Driver Onboarding**, **Customer Support**, and **Operations**
handle distinct areas of the business, which naturally map to different subdomains in the software.

- **Example (Car-sharing)**: The **Driver Onboarding Team** is responsible for managing new drivers, checking their
  documentation, and approving them to provide rides. The **Operations Team** ensures that the ride matching algorithm
  runs smoothly, monitoring the real-time status of drivers and passengers. These two teams have different responsibilities
  and goals, so the software architecture should reflect this by separating the **Driver Management** and **Ride Matching**
  subdomains.

**Additional Insight (Conway’s Law)**: "The organizational structure of a car-sharing company, with separate teams for **Driver Onboarding** and **Operations**, naturally suggests that the system should be divided into subdomains such as **Driver Management** and **Ride Matching**."

### 2. Domain Experts (Expert Segmentation Heuristic)
Domain experts within an organization specialize in different areas of the business. In a car-sharing company, the experts managing driver recruitment and certification will have different knowledge and goals than those responsible for optimizing real-time ride matching. This segmentation helps define the boundaries of subdomains.

- **Example (Car-sharing)**: The **Driver Onboarding Team** includes experts who understand the legal and safety requirements for drivers. They work separately from the **Operations Team**, which focuses on ensuring that drivers are matched with riders as efficiently as possible. These two teams represent distinct subdomains: **Driver Management** and **Ride Operations**.

**Additional Insight (Domain Experts)**: "Experts responsible for driver certification and recruitment differ from those optimizing ride matching algorithms, which suggests that these should be treated as separate subdomains in the system: **Driver Management** and **Ride Operations**."

### 3. Ubiquitous Language (Language Heuristic)
Subdomains can be identified by analyzing how different terms are used across the organization. When the same term has different meanings for different teams, it’s a signal that those teams are working within separate subdomains.

- **Example (Car-sharing)**: The term "driver" means different things to different teams. For the **Driver Onboarding Team**, a "driver" is someone in the process of being recruited, trained, and approved. For the **Operations Team**, a "driver" is someone actively providing rides. This difference in understanding suggests that these two areas should be treated as separate subdomains: **Driver Management** and **Ride Operations**.

**Additional Insight (Ubiquitous Language)**: "In a car-sharing company, the term *driver* means different things in different subdomains. For **Driver Onboarding**, it refers to someone being recruited, while for **Operations**, it refers to someone actively driving. This distinction suggests that **Driver Management** and **Ride Operations** are separate subdomains."

### 4. Business Value (Value-Based Heuristic)
Prioritize subdomains based on their business value. Core domains that provide the most competitive advantage should be the highest priority, while supporting and generic domains can be given less focus.

- **Example (Car-sharing)**: The **ride matching algorithm** is a high-value core domain, as it defines the car-sharing service’s competitive advantage. Ensuring that rides are quickly and efficiently matched to drivers is crucial. On the other hand, **Driver Management** is a supporting domain that, while necessary, doesn’t directly differentiate the service from competitors. **Payment Processing** is a generic domain that can be handled by a third-party service.

**Additional Insight (Business Value)**: "The **ride matching algorithm** is the core subdomain that provides a competitive edge to a car-sharing company, whereas **Driver Management** and **Payment Processing** are supporting and generic subdomains, respectively, which may be less critical in terms of business value."

### 5. Process Steps (Process Heuristic)
Analyzing the key steps in a business process can help identify subdomains. In a car-sharing service, each major step in the process from booking a ride to completing payment could represent a different subdomain.

- **Example (Car-sharing)**: The process of ordering a ride involves distinct steps: **ride request**, **driver assignment**, **ride completion**, and **payment processing**. Each of these steps can be modeled as separate subdomains, such as **Ride Operations** (for the actual ride process) and **Payment Processing** (for handling the transaction at the end).

**Additional Insight (Process Steps)**: "In a car-sharing system, the steps involved in booking a ride, such as **ride request**, **driver assignment**, **ride completion**, and **payment processing**, can each be treated as separate subdomains: **Ride Operations** and **Payment Processing**."

### 6. Change Coupling (Technical Heuristic)
If changes to one part of the system frequently require changes in another, the system may be too tightly coupled. This heuristic helps identify areas where subdomains need to be separated to reduce change coupling.

- **Example (Car-sharing)**: If modifying the **Ride Matching System** often requires changes to the **Payment System**, this is a sign that the two systems are too tightly coupled. These systems should be separated into distinct subdomains to allow for more flexibility in making changes.

**Additional Insight (Change Coupling)**: "If changes in the **Ride Matching System** frequently require changes to the **Payment System**, it’s an indication that these two areas are too tightly coupled and should be separated into distinct subdomains to allow for greater flexibility and ease of maintenance."

### 7. Bounded Contexts (Strategic Heuristic)
Subdomains should align with bounded contexts, ensuring that each part of the system operates within its own well-defined boundaries and doesn’t leak into other areas.

- **Example (Car-sharing)**: The **Ride Matching** context is responsible for handling the logic of matching drivers and passengers. The **Payment Processing** context, on the other hand, deals exclusively with handling transactions and ensuring that drivers are paid and passengers are charged correctly. These contexts should have clear boundaries and not overlap with each other.

**Additional Insight (Bounded Contexts)**: "Each subdomain, such as **Ride Matching** and **Payment Processing**, should correspond to a well-defined bounded context. This ensures that each subdomain has clear boundaries and doesn’t inadvertently affect or overlap with others."

## Conclusion
By using these heuristics, you can better discover and define subdomains in complex systems. Properly identifying these boundaries leads to more maintainable, flexible, and scalable architectures. Subdomains play a crucial role in ensuring that teams can work independently, and software components can evolve without unnecessary coupling.
