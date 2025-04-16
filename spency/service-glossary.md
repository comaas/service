# Go Service Architecture Terminology

This glossary explains key terms and concepts used in the service architecture we've examined.

## Core Concepts

### Service
A standalone application that performs specific business functions and can be deployed independently. In this architecture, a service typically exposes HTTP endpoints, contains business logic, and interacts with databases.

### Microservices
An architectural style that structures an application as a collection of loosely coupled services. In this codebase, there are multiple services (auth, metrics, sales) that work together.

### API (Application Programming Interface)
Defines how other software components can interact with the service. In this codebase, HTTP APIs are defined in the `api` directory with routes and handlers.

### REST (Representational State Transfer)
An architectural style for designing networked applications. The APIs in this codebase follow REST principles with HTTP methods like GET, POST, PUT, DELETE operating on resources.

## Architectural Layers

### Presentation Layer
The outermost layer that handles HTTP requests and responses. In this codebase, this is in the `api` and `app` directories.

### Business Layer
Contains domain-specific logic and rules. In this codebase, this is in the `business/domain` directory with packages like `userbus`, `productbus`, etc.

### Data Access Layer
Handles database operations and data persistence. In this codebase, this is in subdirectories like `business/domain/userbus/stores/userdb`.

## Code Organization

### Handler
A function that processes HTTP requests. In this codebase, these are methods in the `app` structs like `userapp.create`.

### Middleware
Software that acts as a bridge between different components. In this codebase, middleware functions like `mid.Authenticate` add functionality (like authentication) to request processing.

### Router
Directs incoming requests to the appropriate handlers. In this codebase, this is handled by the `web.App` struct with routes defined in `route.go` files.

### Business Logic
Core application rules and workflows. In this codebase, these are implemented in the `Business` structs in packages like `userbus`.

### Repository Pattern
A design pattern that abstracts data access logic. In this codebase, this is implemented through interfaces like `Storer`.

### Dependency Injection
A technique where objects receive their dependencies rather than creating them. In this codebase, dependencies are passed through constructors like `NewBusiness()`.

## Data Modeling

### Model
Represents data structures used in the application. In this architecture, there are typically three types:
- **API Models**: Defined in the presentation layer for HTTP requests/responses
- **Business Models**: Used by business logic
- **Database Models**: Used for database operations

### DTO (Data Transfer Object)
Objects used to transfer data between layers. In this codebase, functions like `toAppUser()` and `toBusUser()` convert between these models.

## Technical Terms

### Context
Go's context package provides a way to carry deadlines, cancellation signals, and request-scoped values across API boundaries. Used extensively throughout the codebase.

### Middleware
Functions that wrap HTTP handlers to add functionality like authentication, logging, or error handling.

### Handler Function
A function that processes HTTP requests and returns responses. In this codebase, these are methods on app structs.

### Route
A mapping between an HTTP method, URL path, and handler function.

### Business
A struct that encapsulates business logic for a specific domain, like `userbus.Business`.

### Store/Storer
An interface or implementation that handles data persistence operations.

### Query Filters
Parameters used to filter database queries, implemented as structs like `QueryFilter`.

### Pagination
The process of dividing data into discrete pages. Implemented through the `page.Page` type.

### Ordering
Specifying the sequence of results from a query. Implemented through the `order.By` type.

## Service-Specific Terms

### Auth Service
Handles authentication and authorization for the system.

### Sales Service
Manages product and sales-related functionality.

### Metrics Service
Collects and reports system metrics and performance data.

### Delegate Pattern
A pattern for notifying other parts of the system about events. Implemented through the `delegate.Delegate` type.

## Core Infrastructure

### Web Application
The foundation HTTP server and router implementation in `foundation/web`.

### Logger
Structured logging implementation in `foundation/logger`.

### Database Connection
Connection pool and helpers for database operations in `business/sdk/sqldb`.

### Tracing
Distributed tracing implementation in `foundation/otel`.

### Configuration
Environment-based configuration using `ardanlabs/conf`.

## Design Patterns

### Clean Architecture
A software design philosophy emphasizing separation of concerns. This codebase follows this pattern with its layered approach.

### Repository Pattern
Abstracts data access behind interfaces. Implemented through the `Storer` interfaces.

### Middleware Pattern
Adds cross-cutting concerns like authentication. Implemented through middleware functions.

### Dependency Injection
Components receive their dependencies rather than creating them. Used throughout the codebase.

### Command Query Responsibility Segregation (CQRS)
Separates read and write operations. Partially implemented with separate Query and Command methods.

## Common Functionality

### Authentication
Verifying the identity of a user or system. Implemented through the `auth` package.

### Authorization
Determining if a user has permission to perform an action. Implemented through middleware like `mid.Authorize`.

### Validation
Ensuring data meets requirements. Implemented using struct tags and validation functions.

### Error Handling
Managing and reporting errors. The codebase uses custom error types and wrapping.

### Transactions
Ensuring database operations succeed or fail as a unit. Implemented through the `CommitRollbacker` interface.

## Deployment Terms

### Container
A standardized unit of software that packages code and dependencies. Referenced in Docker files.

### Kubernetes
An orchestration system for containerized applications. Referenced in the `zarf` directory.

### CircleCI
A continuous integration and delivery platform. Configuration in `.circleci` directory.

### Makefile
A file containing commands for building and managing the application.

### Migration
Scripts to update database schemas. Typically located in database-related directories.
