# Go Service Architecture - A Practical Guide

## What is a Service?

A service in Go is a standalone application that:
1. Runs as its own process
2. Has its own API (typically HTTP)
3. Manages its own data
4. Implements business logic for a specific domain
5. Can be deployed independently

Think of a service like a specialized worker in a company - it has a specific job, knows how to interact with others when needed, but is largely independent.

## Project Structure Overview

This project follows a clean architecture approach with clear separation of concerns. Let's break it down from the outside in:

```
service/
├── api/         # HTTP API and service entry points
├── app/         # Application-specific logic (handlers, middleware)
├── business/    # Business logic and domain models
├── foundation/  # Core infrastructure and utilities
└── zarf/        # DevOps configuration
```

## Service Architecture - Practical Example

Let's understand this with a practical example of a user management feature:

### 1. Main Application - The Starting Point

The entry point for a service is its `main.go` file. Here's what happens when you start your service:

```go
// In main.go:
func main() {
    // 1. Set up logging
    log := logger.New(...)
    
    // 2. Load configuration from environment variables
    cfg := loadConfig()
    
    // 3. Connect to database
    db := connectToDatabase(cfg)
    
    // 4. Set up business logic components
    userBus := setupUserBusiness(db)
    
    // 5. Create HTTP server and routes
    app := setupRoutes(userBus)
    
    // 6. Start the server
    server := http.Server{
        Addr:    cfg.APIHost,
        Handler: app,
    }
    server.ListenAndServe()
}
```

This pattern allows each service to:
- Load its specific configuration
- Connect to its own databases or other resources
- Set up only the business logic it needs
- Configure routes for its specific API endpoints

### 2. Layers of the Service

Let's trace through how a user registration request flows through the service:

```
HTTP Request (POST /users) → HTTP Handler → Business Logic → Database → Response
```

#### Layer 1: API Routes (app/domain/userapp/route.go)

```go
// First, routes are defined and connected to handlers
app.HandlerFunc(http.MethodPost, "v1", "/users", api.create, authen, ruleAdmin)
```

This line is saying:
- Handle POST requests to /v1/users
- Use the 'create' function in the userapp package
- Apply authentication and authorization middleware

#### Layer 2: HTTP Handlers (app/domain/userapp/userapp.go)

```go
// The handler converts HTTP to domain calls
func (a *app) create(ctx context.Context, r *http.Request) web.Encoder {
    // 1. Parse incoming JSON
    var app NewUser
    if err := web.Decode(r, &app); err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    // 2. Convert to business model
    nc, err := toBusNewUser(app)
    if err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    // 3. Call business logic
    usr, err := a.userBus.Create(ctx, nc)
    if err != nil {
        // Handle specific business errors
        if errors.Is(err, userbus.ErrUniqueEmail) {
            return errs.New(errs.Aborted, userbus.ErrUniqueEmail)
        }
        return errs.Newf(errs.Internal, "create: usr[%+v]: %s", usr, err)
    }

    // 4. Convert business model back to API response
    return toAppUser(usr)
}
```

The handler's job is to:
- Parse incoming request data
- Call the appropriate business logic
- Handle any errors
- Format the response back to the client

#### Layer 3: Business Logic (business/domain/userbus/userbus.go)

```go
// The business layer implements domain logic
func (b *Business) Create(ctx context.Context, nu NewUser) (User, error) {
    // 1. Add telemetry for monitoring
    ctx, span := otel.AddSpan(ctx, "business.userbus.create")
    defer span.End()

    // 2. Hash the password (security logic)
    hash, err := bcrypt.GenerateFromPassword([]byte(nu.Password), bcrypt.DefaultCost)
    if err != nil {
        return User{}, fmt.Errorf("generatefrompassword: %w", err)
    }

    // 3. Prepare the user entity
    now := time.Now()
    usr := User{
        ID:           uuid.New(),
        Name:         nu.Name,
        Email:        nu.Email,
        PasswordHash: hash,
        Roles:        nu.Roles,
        Department:   nu.Department,
        Enabled:      true,
        DateCreated:  now,
        DateUpdated:  now,
    }

    // 4. Call the data store layer
    if err := b.storer.Create(ctx, usr); err != nil {
        return User{}, fmt.Errorf("create: %w", err)
    }

    return usr, nil
}
```

The business layer:
- Contains all domain logic (password hashing, data validation, etc.)
- Is independent of HTTP or specific database technologies
- Uses interfaces for data access (Storer interface)
- Has its own domain-specific error types

#### Layer 4: Data Access (business/domain/userbus/stores/userdb/userdb.go)

```go
// The store layer handles database operations
func (s *Store) Create(ctx context.Context, usr userbus.User) error {
    const q = `
    INSERT INTO users
        (user_id, name, email, password_hash, roles, department, enabled, date_created, date_updated)
    VALUES
        (:user_id, :name, :email, :password_hash, :roles, :department, :enabled, :date_created, :date_updated)`

    if err := sqldb.NamedExecContext(ctx, s.log, s.db, q, toDBUser(usr)); err != nil {
        if errors.Is(err, sqldb.ErrDBDuplicatedEntry) {
            return fmt.Errorf("namedexeccontext: %w", userbus.ErrUniqueEmail)
        }
        return fmt.Errorf("namedexeccontext: %w", err)
    }

    return nil
}
```

The data store layer:
- Contains SQL queries and database operations
- Maps between domain models and database models
- Handles database-specific errors and converts them to domain errors

## Practical Benefits of This Architecture

1. **Separation of Concerns**: Each layer has a specific responsibility, making the code easier to understand and maintain.

2. **Testability**: Business logic can be tested independently of HTTP or database concerns.

3. **Flexibility**: You can change databases or API formats without changing business logic.

4. **Scalability**: Services can be deployed and scaled independently.

5. **Developer Independence**: Different teams can work on different services without stepping on each other's toes.

## Common Service Patterns in This Code

### Middleware Pattern

The code uses middleware for cross-cutting concerns:

```go
// Authentication and authorization middleware
authen := mid.Authenticate(cfg.AuthClient)
ruleAdmin := mid.Authorize(cfg.AuthClient, auth.RuleAdminOnly)

// Apply middleware to routes
app.HandlerFunc(http.MethodPost, "v1", "/users", api.create, authen, ruleAdmin)
```

Middleware functions wrap HTTP handlers to add functionality like:
- Authentication and authorization
- Logging
- Request tracing
- Error handling

### Repository Pattern

The code uses the repository pattern for data access:

```go
// Define an interface for data access
type Storer interface {
    Create(ctx context.Context, usr User) error
    Update(ctx context.Context, usr User) error
    // ...other methods
}

// Business logic depends on the interface, not the implementation
func NewBusiness(log *logger.Logger, delegate *delegate.Delegate, storer Storer) *Business {
    return &Business{
        log:      log,
        delegate: delegate,
        storer:   storer,
    }
}
```

This allows you to:
- Swap database implementations
- Mock the database for testing
- Add caching or other enhancements without changing business logic

### Dependency Injection

The code uses constructor injection for dependencies:

```go
// Business layer receives its dependencies through constructor
func NewBusiness(log *logger.Logger, delegate *delegate.Delegate, storer Storer) *Business {
    return &Business{
        log:      log,
        delegate: delegate,
        storer:   storer,
    }
}
```

This approach:
- Makes dependencies explicit
- Simplifies testing
- Avoids global state

## Practical Examples of Using This Service

### Example 1: Creating a New User

```go
// HTTP POST to /v1/users with JSON body
{
    "name": "John Doe",
    "email": "john@example.com",
    "password": "secret123",
    "roles": ["USER"],
    "department": "Engineering"
}

// Handler parses this and calls business layer
userBus.Create(ctx, NewUser{...})

// Business layer hashes password, creates UUID, sets timestamps
// Then calls data store
storer.Create(ctx, User{...})

// Data store executes SQL insert
// Response returns the created user (without password hash)
```

### Example 2: Querying Users with Filters

```go
// HTTP GET to /v1/users?name=John&page=2&rows=10

// Handler parses query parameters
filter := QueryFilter{Name: "John"}
page := page.New(2, 10)

// Business layer calls data store
users, err := userBus.Query(ctx, filter, orderBy, page)

// Data store builds SQL with WHERE clause for name
SELECT * FROM users WHERE name LIKE '%John%' LIMIT 10 OFFSET 10

// Response returns users and metadata about total count and pagination
```

## Key Software Engineering Decisions in This Design

1. **Clean Architecture**: The code uses clean architecture principles to separate concerns and dependencies.

2. **Domain-Driven Design**: Business logic is organized around domain concepts (users, products, etc.).

3. **Explicit Error Handling**: Errors are wrapped with context and converted between layers.

4. **Observability**: Built-in logging and tracing for production monitoring.

5. **Configuration Management**: Environment-based configuration with sensible defaults.

6. **Transaction Support**: Business logic can use database transactions when needed.

7. **Graceful Shutdown**: The service handles shutdown signals properly.

8. **Modular Routes**: Routes can be selectively compiled into different binaries.

## How to Extend This Service

### Adding a New Feature

To add a new feature like "user password reset":

1. Add a new handler in `userapp.go`:
   ```go
   func (a *app) resetPassword(ctx context.Context, r *http.Request) web.Encoder {
       // Parse request
       // Call business logic
       // Return response
   }
   ```

2. Add a new route in `route.go`:
   ```go
   app.HandlerFunc(http.MethodPost, version, "/users/reset-password", api.resetPassword, authen)
   ```

3. Add business logic in `userbus.go`:
   ```go
   func (b *Business) ResetPassword(ctx context.Context, email mail.Address) error {
       // Find user
       // Generate token
       // Send email
       // Return result
   }
   ```

### Adding a New Service

To add a completely new service:

1. Create a new directory in `api/services/newservice/`
2. Create your own `main.go` that wires everything together
3. Define your business models and logic in `business/domain/newbus/`
4. Create your API handlers in `app/domain/newapp/`
5. Update deployment configurations in `zarf/`

This modular approach allows you to grow the system over time while maintaining clear boundaries between services.
