---
title: "How to build the best payment-gateway"
slug: "post-3"
meta_title: ""
description: "this is meta description"
date: 2025-06-01T05:00:00Z
image: "/images/5-start.png"
categories: ["Software"]
author: "Andrés Retamal López"
tags: [
  ".NET",
  "Clean Architecture",
  "Microservices",
  "Docker",
  "Azure",
  "Fintech"
]


draft: false
---

If a payment gateway fails, the business loses money. Period.

This diagram provides a complete high-level view of the payment gateway’s architecture before diving into each layer in detail.



![Workflow Diagram](/images/API.png)

After years working on high-traffic systems, I’ve learned what makes a payment gateway reliable, scalable, and easy to evolve.

According to my professional experience — and after several years working with payment systems in high-traffic environments — the most comfortable and efficient way I’ve built a payment gateway is using a Hexagonal Architecture combined with a Microservices approach.

This setup allows you to design systems that are:

- Easier to maintain and scale,
- Highly modular and testable,
- And capable of handling multiple payment providers while remaining independent of them.

In this article, I’ll walk through how I approached the architecture, some of the design decisions that made the system flexible and clean, and show snippets that demonstrate how everything connects.
My goal isn’t to reinvent the wheel — it’s to share a real-world, production-ready implementation that actually worked in a complex environment.

## 🧠 Why Hexagonal Architecture?

This diagram summarizes the full architecture behind a scalable and provider-agnostic payment gateway built with Hexagonal principles.

Here is a high-level overview of the architecture before breaking down each layer.

![Workflow Diagram](/images/Diagrama3.png)

Hexagonal Architecture keeps your business logic completely independent from external frameworks, databases, and payment providers.
If you ever replace the storage engine or switch to another payment processor, nothing in the core domain needs to change, you simply connect a new adapter.

Think of it like a set of concentric circles: inner layers never depend on outer layers.
This separation guarantees clarity, scalability, testability, and long-term maintainability.

### 📁 Service Folder Structure (High-Level Overview)


```csharp
Service/
├─ PaymentGateway.Api
├─ PaymentGateway.Application
├─ PaymentGateway.Contracts
├─ PaymentGateway.Domain.Models
└─ PaymentGateway.Infrastructure
```

## 1️⃣ API Layer


This is an ASP.NET Core Web API project using the Minimal API approach (no controllers).
Instead of the classic Controllers, we define endpoints through an interface called IEndpointDefinition.
Each endpoint is registered in a lightweight, declarative way:

```csharp
app.MapPost("/session", CreateSession)
   .RequireAuthorization()
   .Produces<CheckoutSessionResponse>()
   .Produces(StatusCodes.Status201Created)
   .ProducesProblem(StatusCodes.Status400BadRequest)
   .ProducesProblem(StatusCodes.Status500InternalServerError);
```


## 2️⃣ Application Layer

The Application Layer handles all business logic for the microservice.
Here’s where I define services, use cases, mappings, and abstractions.
AutoMapper is used to handle transformations between domain models and DTOs, while dependency injection keeps everything testable and loosely coupled.

Typical structure:

```csharp
Application/
├─ Abstract/
├─ Mappers/
├─ Services/
├─ Startup.cs
└─ Utils.cs
```

```csharp
public async Task<CheckoutSessionDetailResponse> CreateCheckoutSession(Guid accountId, CheckoutSessionRequest request)
{
    // Business logic: validation, call to provider SDK, persistence
    var result = await _paymentProvider.CreateSessionAsync(accountId, request);
    return result;
}
```




## 3️⃣ Infrastructure Layer

Handles all external integrations — databases, SDKs, or other microservices.
In this case, the payment gateway integrates with Adyen’s official .NET SDK, which allows processing transactions and managing checkout sessions securely.

The repository follows a clear pattern using both Entity Framework Core (for commands) and Dapper (for optimized queries).

Structure example:

```csharp
Infrastructure/
├─ Data/
│  ├─ Context/
│  └─ Repositories/
├─ Integrations/
│  └─ PaymentProvider/
└─ Startup.cs
```

This separation ensures that even if you migrate to a different payment provider or database, your domain logic doesn’t change.





## 4️⃣ Domain Models — The Heart of the Business

In Hexagonal Architecture, the Domain Models layer contains the pure business rules, untouched by frameworks, databases, or providers. These entities define how the payment gateway behaves at its core.

```csharp
   [Domain Models]
       |
   [Application]
       |
    [API]
```

This is where the entities live — the core of the system.
It defines all models related to the payment flow: sessions, merchants, configurations, transactions, logs, etc.

Example entities:

- CheckoutSession
- MerchantPlan
- Payment
- PaymentLog

These models represent the real business and are shared across other layers through abstractions and DTOs.



## 5️⃣ Contracts Layer

The Contracts layer defines the communication rules between all layers and microservices. It ensures consistency, prevents coupling, and acts as the shared language of the entire system.

A transversal project that contains DTOs, Enums, and Interfaces shared between the main layers.
Here is where we define the public contracts for communication and data exchange between microservices.

Example folders:

```csharp
Contracts/
├─ Agents/
├─ Commands/
├─ DTO/
├─ Enums/
├─ Integrations/
└─ Queries/
```


## 6️⃣ Test Layer

Unit tests validate isolated business logic (Application + Domain).
Functional tests ensure the full payment flow works end-to-end.

The final piece.
This layer covers Unit and Functional testing for the entire microservice.
The goal is to maintain a minimum coverage threshold and guarantee that every layer works independently.

🔌 Payment Flow Example

The following endpoint shows how the layers interact in a real request, following the exact flow described above.

Here’s a simplified view of how a Checkout Session is created in the system:

```csharp
public class CheckoutEndpointDefinition : IEndpointDefinition
{
    private readonly IDistributedCache _cache;
    private readonly IHttpContextAccessor _contextAccessor;
    private const string CACHE_KEY_SESSION_DETAIL = "CheckoutSession_{0}";

    public CheckoutEndpointDefinition(IDistributedCache cache, IHttpContextAccessor contextAccessor)
    {
        _cache = cache;
        _contextAccessor = contextAccessor;
    }

    public void DefineEndpoint(WebApplication app)
    {
        app.MapPost("/session", CreateSession)
           .RequireAuthorization()
           .Produces<CheckoutSessionResponse>()
           .Produces(StatusCodes.Status201Created);
    }

    internal async Task<IResult> CreateSession(ICheckoutService service, CheckoutSessionRequest request)
    {
        if (request == null)
            return Results.BadRequest("Invalid request.");

        if (_contextAccessor.HttpContext == null)
            return Results.Problem("Inaccessible context.");

        var detail = await service.CreateCheckoutSession(GetAccountId(), request);

        if (detail.RequestSuccess)
        {
            await _cache.SetRecordAsync(string.Format(CACHE_KEY_SESSION_DETAIL, detail.Id), detail);
            var response = service.SimplifyCheckoutSessionResponse(detail);
            return Results.Created($"/session/{response.Id}", response);
        }

        return Results.Problem(detail.RequestErrors, statusCode: (int)HttpStatusCode.InternalServerError);
    }
}
```

![Workflow Diagram](/images/CheckoutSession.png)

This endpoint demonstrates:

- Clear separation of responsibilities
- Integration with caching
- Clean error handling
- Reusable service layer logic

In the end, a payment gateway is not just a microservice

it’s one of the most critical pieces of the business.

A payment gateway is not just a microservice, it is the financial engine of the business.
By keeping the architecture simple, scalable, replaceable, and fully testable, you guarantee stability where it matters the most: the money flow.


If you want the full repository structure, sample implementation, or more advanced flows (refunds, webhooks, retries), let me know and I’ll publish a follow-up article.

> “Clean architecture gives you the freedom to evolve your business rules stay the same, even when everything else changes.”

