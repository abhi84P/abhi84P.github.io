+++
title = "Scaffolding a .NET Clean Architecture Solution from Scratch"
date = "2026-06-09"
weight = 3
description = "Step-by-step CLI commands to scaffold a proper Clean Architecture solution in .NET with Domain, Application, Infrastructure, Persistence, and API layers."
tags = ["dotnet", "clean-architecture", "backend"]
+++

Clean Architecture enforces a dependency rule: inner layers know nothing about outer layers. Domain has no dependencies. Application depends only on Domain. Infrastructure and Persistence depend on Application. The API wires everything together.

This post walks through scaffolding that structure from scratch using only the .NET CLI.

---

## The Target Structure

```
CleanArchitectureDemo/
├── Core/
│   ├── Domain/          ← entities, value objects, domain events
│   └── Application/     ← use cases, interfaces, DTOs
├── Infrastructure/
│   ├── Infrastructure/  ← external services, email, storage, etc.
│   └── Persistence/     ← EF Core, repositories, migrations
└── Presentation/
    └── API/             ← controllers, middleware, DI composition root
```

---

## Step 1 — Create the Solution

```bash
mkdir CleanArchitectureDemo
cd CleanArchitectureDemo
dotnet new sln
dotnet new gitignore
```

Verify the solution file exists:

```bash
dotnet sln CleanArchitectureDemo.sln list
```

---

## Step 2 — Domain Layer

The innermost layer. No dependencies on anything — only pure C# classes.

```bash
dotnet new classlib -o Core/Domain
rm Core/Domain/Class1.cs
dotnet sln CleanArchitectureDemo.sln add Core/Domain
```

Put your entities, value objects, domain events, and domain exceptions here. Nothing from EF Core, nothing from ASP.NET.

---

## Step 3 — Application Layer

Orchestrates use cases. Depends on Domain, defines interfaces that outer layers implement.

```bash
dotnet new classlib -o Core/Application
rm Core/Application/Class1.cs
dotnet sln CleanArchitectureDemo.sln add Core/Application
```

Wire the reference:

```bash
dotnet add Core/Application/Application.csproj reference Core/Domain/Domain.csproj
```

This is where you put CQRS handlers (MediatR), DTOs, repository interfaces, and service interfaces. Application defines the contracts — Infrastructure fulfills them.

---

## Step 4 — Infrastructure Layer

Implements application interfaces for external concerns: email, file storage, third-party APIs, etc.

```bash
dotnet new classlib -o Infrastructure/Infrastructure
rm Infrastructure/Infrastructure/Class1.cs
dotnet sln CleanArchitectureDemo.sln add Infrastructure/Infrastructure
```

Wire the reference:

```bash
dotnet add Infrastructure/Infrastructure/Infrastructure.csproj reference Core/Application/Application.csproj
```

---

## Step 5 — Persistence Layer

Separate from Infrastructure to isolate database concerns. EF Core, repositories, migrations, and `DbContext` live here.

```bash
dotnet new classlib -o Infrastructure/Persistence
rm Infrastructure/Persistence/Class1.cs
dotnet sln CleanArchitectureDemo.sln add Infrastructure/Persistence
```

Wire the reference:

```bash
dotnet add Infrastructure/Persistence/Persistence.csproj reference Core/Application/Application.csproj
```

Keeping Persistence separate from Infrastructure means you can swap the ORM or database without touching email/storage code, and vice versa.

---

## Step 6 — API Layer

The composition root. Depends on both infrastructure layers, registers everything into the DI container, and exposes HTTP endpoints.

```bash
dotnet new webapi -o Presentation/API
dotnet sln CleanArchitectureDemo.sln add Presentation/API
```

Wire the references:

```bash
dotnet add Presentation/API/API.csproj reference Infrastructure/Infrastructure/Infrastructure.csproj
dotnet add Presentation/API/API.csproj reference Infrastructure/Persistence/Persistence.csproj
```

The API does not reference Domain or Application directly for business logic — it goes through Infrastructure and Persistence, which already reference Application.

---

## Step 7 — Verify

```bash
dotnet sln CleanArchitectureDemo.sln list
dotnet build
```

Expected output:

```
Core/Domain/Domain.csproj
Core/Application/Application.csproj
Infrastructure/Infrastructure/Infrastructure.csproj
Infrastructure/Persistence/Persistence.csproj
Presentation/API/API.csproj
```

---

## Dependency Flow (Summary)

```
API → Infrastructure → Application → Domain
API → Persistence   → Application → Domain
```

Domain knows nothing. Application knows only Domain. Infrastructure and Persistence know Application. API knows everything — but only to wire it up, not to hold business logic.

---

## What Goes Where

| Layer | Contents |
|---|---|
| **Domain** | Entities, Value Objects, Domain Events, Enums, Exceptions |
| **Application** | Use Cases, CQRS Handlers, Repository Interfaces, Service Interfaces, DTOs, Validators |
| **Infrastructure** | Email, Storage, External APIs, Identity, Logging implementations |
| **Persistence** | DbContext, EF Core Configurations, Migrations, Repository implementations |
| **API** | Controllers, Middleware, DI Registration, Program.cs |

---

From here, install MediatR in Application, EF Core in Persistence, and register everything in API's `Program.cs` using extension methods defined in each layer. That keeps `Program.cs` clean and each layer self-registering.

---

**Next:** [Wiring Up Dapper and a Clean Program.cs in .NET Clean Architecture](/blog/dotnet-clean-architecture-dapper-setup/) — how to set up Dapper with PostgreSQL, DbUp migrations, Unit of Work, and a `Program.cs` that stays three lines long.
