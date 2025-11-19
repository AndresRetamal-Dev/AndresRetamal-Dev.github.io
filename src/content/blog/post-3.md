---
title: "How to build the best payment-gateway"
slug: "post-3"
meta_title: ""
description: "this is meta description"
date: 2025-11-12T05:00:00Z
image: "/images/5-start.png"
categories: ["Software"]
author: "Andrés Retamal López"
tags: ["software", "tailwind"]
draft: false
---

According to my professional experience, and after several years working with payment systems in high-traffic environments,
the most comfortable and efficient way I’ve built a payment gateway is using a Hexagonal Architecture combined with a Microservices approach.

This setup allows you to design systems that are:

- Easier to maintain and scale,
- Highly modular and testable,
- And capable of handling multiple payment providers while remaining independent of them.

In this article, I’ll walk through how I approached the architecture, some of the design decisions that made the system flexible and clean, and show snippets that demonstrate how everything connects.
My goal isn’t to reinvent the wheel — it’s to share a real-world, production-ready implementation that actually worked in a complex environment.

## 🧠 Why Hexagonal Architecture?

Because it forces you to keep the core domain independent from external frameworks, databases, and third-party services.
If you ever decide to change the database engine or switch payment providers, you don’t have to rewrite the business logic — you simply plug in a new adapter.

It’s like building a system in concentric circles: each layer depends only on the inner ones, never outward.
That separation gives clarity, testability, and long-term maintainability.

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





## 4️⃣ Domain Models Layer

This is where the entities live — the core of the system.
It defines all models related to the payment flow: sessions, merchants, configurations, transactions, logs, etc.

Example entities:

- CheckoutSession
- MerchantPlan
- Payment
- PaymentLog

These models represent the real business and are shared across other layers through abstractions and DTOs.



## 5️⃣ Contracts Layer

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


The final piece.
This layer covers Unit and Functional testing for the entire microservice.
The goal is to maintain a minimum coverage threshold and guarantee that every layer works independently.

🔌 Payment Flow Example

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

This endpoint demonstrates:

- Clear separation of responsibilities
- Integration with caching
- Clean error handling
- Reusable service layer logic

> “Clean architecture gives you the freedom to evolve your business rules stay the same, even when everything else changes.”

