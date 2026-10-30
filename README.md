# E-Shop Microservices (.NET 8)

An e-commerce backend built as a set of microservices with **ASP.NET Core 8**, covering **Catalog, Basket, Discount and Ordering** services. It combines NoSQL (Marten document DB on PostgreSQL, Redis) and relational (SQL Server, SQLite) storage, uses **gRPC** for synchronous calls, **RabbitMQ** for event-driven communication, and a **YARP API Gateway** as the single entry point.

## Architecture

| Service | Style / Pattern | Storage | Communication |
|---|---|---|---|
| **Catalog.API** | Vertical Slice, CQRS | PostgreSQL (Marten) | REST (Carter) |
| **Basket.API** | REST, Decorator + Cache-aside | PostgreSQL (Marten) + Redis | gRPC client → Discount, publishes `BasketCheckoutEvent` |
| **Discount.Grpc** | gRPC server | SQLite (EF Core) | gRPC |
| **Ordering.API** | Clean Architecture, DDD, CQRS | SQL Server (EF Core) | Consumes `BasketCheckoutEvent` |
| **YARP API Gateway** | Gateway Routing | – | Reverse proxy to all services |
| **Shopping.Web** | Razor Pages (Bootstrap 5) | – | Refit → Gateway |

## Features

### Catalog service
- ASP.NET Core Minimal APIs on .NET 8 / C# 12
- **Vertical Slice Architecture** with feature folders
- **CQRS** using MediatR, with validation pipeline behaviors (FluentValidation)
- **Marten** as a transactional document DB on PostgreSQL
- **Carter** for minimal API endpoint definitions
- Cross-cutting concerns: logging, global exception handling, health checks

### Basket service
- REST API with full CRUD
- **Redis** as a distributed cache on top of the basket store
- Proxy, Decorator and Cache-aside patterns
- Calls the Discount **gRPC** service to calculate the final product price
- Publishes the `BasketCheckout` event with **MassTransit + RabbitMQ**

### Discount service
- ASP.NET Core **gRPC server** with **Protobuf** contracts
- Entity Framework Core with **SQLite** and migrations
- Containerized with its own database

### Ordering service
- **DDD, CQRS and Clean Architecture** (Domain / Application / Infrastructure / API)
- CQRS with MediatR, FluentValidation and Mapster
- Domain events, EF Core interceptors (auditing and domain event dispatching)
- Consumes the `BasketCheckout` event via MassTransit + RabbitMQ
- SQL Server with automatic migration on startup

### Service communication
- **Synchronous:** gRPC (Basket → Discount)
- **Asynchronous:** RabbitMQ publish/subscribe through MassTransit (Basket → Ordering)
- Shared `BuildingBlocks.Messaging` library for integration events

### YARP API Gateway
- Gateway Routing pattern: routes, clusters, path transforms, destinations
- **Rate limiting** with a fixed-window limiter

### Shopping.Web
- ASP.NET Core Razor Pages with Bootstrap 5
- Calls the gateway APIs using **Refit** with `HttpClientFactory`

### Docker Compose
- All microservices and databases containerized
- Environment variables overridden in `docker-compose.override.yml`

## Project structure

```
src/
├── ApiGateways/YarpApiGateway
├── BuildingBlocks/
│   ├── BuildingBlocks            # CQRS, behaviors, exceptions, pagination
│   └── BuildingBlocks.Messaging  # MassTransit/RabbitMQ + integration events
├── Services/
│   ├── Catalog/Catalog.API
│   ├── Basket/Basket.API
│   ├── Discount/Discount.Grpc
│   └── Ordering/{Ordering.API, Ordering.Application, Ordering.Domain, Ordering.Infrastructure}
├── WebApps/Shopping.Web
├── docker-compose.yml
├── docker-compose.override.yml
└── eshop-microservices.sln
```

## Getting started

### Prerequisites
- [Visual Studio 2022](https://visualstudio.microsoft.com/downloads/) (optional)
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [Docker Desktop](https://www.docker.com/products/docker-desktop)

### Run
1. Clone the repository and start Docker Desktop (recommended: at least **4 GB memory** and **2 CPUs**).
2. From the `src` folder (where `docker-compose.yml` lives), run:

   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d
   ```

   Or, in Visual Studio, set **docker-compose** as the startup project and run without debugging.
3. Wait for all containers to start. Some services need a little extra time, so retry after a minute if something is not ready.
4. Open the Shopping Web UI at **https://localhost:6065**. Add products to the basket and check out, then follow the message on the RabbitMQ management dashboard.

