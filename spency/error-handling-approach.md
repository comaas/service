# Error Handling in the Service Architecture

This document outlines the sophisticated error handling approach used in the service architecture, explaining how errors are defined, propagated, and transformed across different layers.

## Core Error Handling Principles

The error handling in this architecture follows several key principles:

1. **Errors as API**: Exported error variables and types are considered part of a package's public API
2. **Layer-specific errors**: Each layer defines errors that are meaningful at that level of abstraction
3. **Error transformation**: Errors are converted between layers to maintain appropriate abstraction
4. **Context preservation**: Error context is added while preserving the original error
5. **Sentinel errors**: Pre-defined, exported error values for expected error conditions

## Error Definitions and Types

### Business Layer Error Definitions

The business layer defines domain-specific errors as exported variables:

```go
// In business/domain/userbus/userbus.go
var (
    ErrNotFound              = errors.New("user not found")
    ErrUniqueEmail           = errors.New("email is not unique")
    ErrAuthenticationFailure = errors.New("authentication failed")
)
```

These exported errors serve as part of the package's API contract, allowing callers to check for specific error conditions.

### API Layer Error Types

The API layer uses a custom error package that adds HTTP semantics to errors:

```go
// In app/sdk/errs/errors.go (example structure)
type Error struct {
    Code    string
    Message string
    Err     error
}

// Error codes that map to HTTP status codes
const (
    InvalidArgument = "INVALID_ARGUMENT"   // 400
    Unauthorized    = "UNAUTHORIZED"       // 401
    NotFound        = "NOT_FOUND"          // 404
    Aborted         = "ABORTED"            // 409 (conflict)
    Internal        = "INTERNAL"           // 500
)

func New(code string, err error) *Error {
    return &Error{
        Code:    code,
        Message: err.Error(),
        Err:     err,
    }
}

func Newf(code string, format string, args ...interface{}) *Error {
    return &Error{
        Code:    code,
        Message: fmt.Sprintf(format, args...),
    }
}
```

## Error Flow Through the Layers

Errors flow through the system layers with transformations at each boundary:

### 1. Data Store Layer → Business Layer

Database-specific errors are converted to business domain errors:

```go
// In business/domain/userbus/stores/userdb/userdb.go
func (s *Store) Create(ctx context.Context, usr userbus.User) error {
    const q = `INSERT INTO users (...) VALUES (...)`

    if err := sqldb.NamedExecContext(ctx, s.log, s.db, q, toDBUser(usr)); err != nil {
        // Convert database-specific error to domain error
        if errors.Is(err, sqldb.ErrDBDuplicatedEntry) {
            return fmt.Errorf("namedexeccontext: %w", userbus.ErrUniqueEmail)
        }
        return fmt.Errorf("namedexeccontext: %w", err)
    }

    return nil
}
```

Key points:
- Database constraint violations are mapped to domain-specific errors
- Context is added with `fmt.Errorf`
- Original error is wrapped with `%w` for preservation

### 2. Business Layer → API Layer

Business domain errors are converted to HTTP-appropriate errors:

```go
// In app/domain/userapp/userapp.go
func (a *app) create(ctx context.Context, r *http.Request) web.Encoder {
    // ... [parsing and validation] ...

    usr, err := a.userBus.Create(ctx, nc)
    if err != nil {
        // Convert domain errors to HTTP-appropriate errors
        if errors.Is(err, userbus.ErrUniqueEmail) {
            return errs.New(errs.Aborted, userbus.ErrUniqueEmail)
        }
        return errs.Newf(errs.Internal, "create: usr[%+v]: %s", usr, err)
    }

    return toAppUser(usr)
}
```

Key points:
- Business errors are mapped to specific HTTP error codes
- Known errors get specific status codes (like 409 Conflict for duplicate emails)
- Unknown errors become 500 Internal Server Errors
- Implementation details are hidden from API clients

### 3. API Layer → HTTP Response

The web framework converts API errors to HTTP responses:

```go
// Conceptual implementation in foundation/web
func (a *App) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // ... [route matching and handler execution] ...

    if err, ok := result.(*errs.Error); ok {
        status := errorToStatusCode(err.Code)
        respond.Error(w, status, err)
        return
    }

    respond.JSON(w, http.StatusOK, result)
}

func errorToStatusCode(code string) int {
    switch code {
    case errs.InvalidArgument:
        return http.StatusBadRequest
    case errs.Unauthorized:
        return http.StatusUnauthorized
    case errs.NotFound:
        return http.StatusNotFound
    case errs.Aborted:
        return http.StatusConflict
    default:
        return http.StatusInternalServerError
    }
}
```

## Error Checking Patterns

The codebase uses Go 1.13+ error checking patterns:

```go
// Using errors.Is to check for specific error types
if errors.Is(err, userbus.ErrNotFound) {
    return errs.New(errs.NotFound, err)
}

// Using errors.As to extract error details (when needed)
var valErr validator.ValidationErrors
if errors.As(err, &valErr) {
    // Handle validation errors specifically
    return handleValidationErrors(valErr)
}
```

## Error Logging and Observability

Errors are logged at appropriate places with context:

```go
if err := a.userBus.Delete(ctx, usr); err != nil {
    log.Error(ctx, "deleting user", "userID", usr.ID, "error", err)
    return errs.Newf(errs.Internal, "delete: userID[%s]: %s", usr.ID, err)
}
```

The logging includes:
- Structured context (operation, identifiers)
- Full error chains
- Request tracing IDs (from context)

## Benefits of This Approach

This layered error handling approach provides several benefits:

1. **Abstraction boundaries**: Each layer deals with errors appropriate to its level of abstraction
2. **Diagnostics**: Error messages contain helpful context for debugging
3. **Client-appropriate responses**: API clients receive appropriate HTTP status codes and messages without internal implementation details
4. **Error handling flexibility**: Known errors can be handled specifically, while unknown errors have sensible defaults
5. **Error type safety**: Using exported error variables allows for reliable error checking
6. **Observability**: Errors are consistently logged with appropriate context

## Implementation Guidelines

When implementing error handling in this architecture:

1. Define expected errors as exported variables in your business domain packages
2. Use `fmt.Errorf` with `%w` to add context while preserving original errors
3. Convert errors at layer boundaries (data → business → api)
4. Check for specific errors using `errors.Is()` and `errors.As()`
5. Map domain errors to appropriate HTTP status codes at the API layer
6. Log errors with contextual information
7. Don't expose internal implementation details to API clients

## Applying These Principles to Your Own Code

This section outlines the key philosophies and guidelines for implementing robust error handling in your own Go applications, based on the patterns in this service architecture.

### Core Philosophies

1. **Errors are part of your API**: Just like functions and types, exported errors are part of your package's contract with consumers. Design them thoughtfully.

2. **Error abstraction between layers**: Each layer should define errors at its appropriate level of abstraction, hiding implementation details from higher layers.

3. **Errors should be informative but not revealing**: Errors should contain enough information for debugging but shouldn't expose internal implementation details to end users.

4. **Predictable error handling**: Consumers of your API should be able to reliably check for and handle specific error conditions.

### Practical Guidelines

#### 1. Define Package-Level Errors

For each package, define sentinel errors for expected error conditions:

```go
// In your package
var (
    ErrNotFound        = errors.New("resource not found")
    ErrInvalidInput    = errors.New("invalid input provided")
    ErrPermissionDenied = errors.New("permission denied")
)
```

Use consistent naming:
- Prefix with `Err`
- Use clear, descriptive names
- Keep error messages concise and actionable

#### 2. Error Transformation at Boundaries

When crossing# Error Handling in the Service Architecture

This document outlines the sophisticated error handling approach used in the service architecture, explaining how errors are defined, propagated, and transformed across different layers.

## Core Error Handling Principles

The error handling in this architecture follows several key principles:

1. **Errors as API**: Exported error variables and types are considered part of a package's public API
2. **Layer-specific errors**: Each layer defines errors that are meaningful at that level of abstraction
3. **Error transformation**: Errors are converted between layers to maintain appropriate abstraction
4. **Context preservation**: Error context is added while preserving the original error
5. **Sentinel errors**: Pre-defined, exported error values for expected error conditions

## Error Definitions and Types

### Business Layer Error Definitions

The business layer defines domain-specific errors as exported variables:

```go
// In business/domain/userbus/userbus.go
var (
    ErrNotFound              = errors.New("user not found")
    ErrUniqueEmail           = errors.New("email is not unique")
    ErrAuthenticationFailure = errors.New("authentication failed")
)
```

These exported errors serve as part of the package's API contract, allowing callers to check for specific error conditions.

### API Layer Error Types

The API layer uses a custom error package that adds HTTP semantics to errors:

```go
// In app/sdk/errs/errors.go (example structure)
type Error struct {
    Code    string
    Message string
    Err     error
}

// Error codes that map to HTTP status codes
const (
    InvalidArgument = "INVALID_ARGUMENT"   // 400
    Unauthorized    = "UNAUTHORIZED"       // 401
    NotFound        = "NOT_FOUND"          // 404
    Aborted         = "ABORTED"            // 409 (conflict)
    Internal        = "INTERNAL"           // 500
)

func New(code string, err error) *Error {
    return &Error{
        Code:    code,
        Message: err.Error(),
        Err:     err,
    }
}

func Newf(code string, format string, args ...interface{}) *Error {
    return &Error{
        Code:    code,
        Message: fmt.Sprintf(format, args...),
    }
}
```

## Error Flow Through the Layers

Errors flow through the system layers with transformations at each boundary:

### 1. Data Store Layer → Business Layer

Database-specific errors are converted to business domain errors:

```go
// In business/domain/userbus/stores/userdb/userdb.go
func (s *Store) Create(ctx context.Context, usr userbus.User) error {
    const q = `INSERT INTO users (...) VALUES (...)`

    if err := sqldb.NamedExecContext(ctx, s.log, s.db, q, toDBUser(usr)); err != nil {
        // Convert database-specific error to domain error
        if errors.Is(err, sqldb.ErrDBDuplicatedEntry) {
            return fmt.Errorf("namedexeccontext: %w", userbus.ErrUniqueEmail)
        }
        return fmt.Errorf("namedexeccontext: %w", err)
    }

    return nil
}
```

Key points:
- Database constraint violations are mapped to domain-specific errors
- Context is added with `fmt.Errorf`
- Original error is wrapped with `%w` for preservation

### 2. Business Layer → API Layer

Business domain errors are converted to HTTP-appropriate errors:

```go
// In app/domain/userapp/userapp.go
func (a *app) create(ctx context.Context, r *http.Request) web.Encoder {
    // ... [parsing and validation] ...

    usr, err := a.userBus.Create(ctx, nc)
    if err != nil {
        // Convert domain errors to HTTP-appropriate errors
        if errors.Is(err, userbus.ErrUniqueEmail) {
            return errs.New(errs.Aborted, userbus.ErrUniqueEmail)
        }
        return errs.Newf(errs.Internal, "create: usr[%+v]: %s", usr, err)
    }

    return toAppUser(usr)
}
```

Key points:
- Business errors are mapped to specific HTTP error codes
- Known errors get specific status codes (like 409 Conflict for duplicate emails)
- Unknown errors become 500 Internal Server Errors
- Implementation details are hidden from API clients

### 3. API Layer → HTTP Response

The web framework converts API errors to HTTP responses:

```go
// Conceptual implementation in foundation/web
func (a *App) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // ... [route matching and handler execution] ...

    if err, ok := result.(*errs.Error); ok {
        status := errorToStatusCode(err.Code)
        respond.Error(w, status, err)
        return
    }

    respond.JSON(w, http.StatusOK, result)
}

func errorToStatusCode(code string) int {
    switch code {
    case errs.InvalidArgument:
        return http.StatusBadRequest
    case errs.Unauthorized:
        return http.StatusUnauthorized
    case errs.NotFound:
        return http.StatusNotFound
    case errs.Aborted:
        return http.StatusConflict
    default:
        return http.StatusInternalServerError
    }
}
```

## Error Checking Patterns

The codebase uses Go 1.13+ error checking patterns:

```go
// Using errors.Is to check for specific error types
if errors.Is(err, userbus.ErrNotFound) {
    return errs.New(errs.NotFound, err)
}

// Using errors.As to extract error details (when needed)
var valErr validator.ValidationErrors
if errors.As(err, &valErr) {
    // Handle validation errors specifically
    return handleValidationErrors(valErr)
}
```

## Error Logging and Observability

Errors are logged at appropriate places with context:

```go
if err := a.userBus.Delete(ctx, usr); err != nil {
    log.Error(ctx, "deleting user", "userID", usr.ID, "error", err)
    return errs.Newf(errs.Internal, "delete: userID[%s]: %s", usr.ID, err)
}
```

The logging includes:
- Structured context (operation, identifiers)
- Full error chains
- Request tracing IDs (from context)

## Benefits of This Approach

This layered error handling approach provides several benefits:

1. **Abstraction boundaries**: Each layer deals with errors appropriate to its level of abstraction
2. **Diagnostics**: Error messages contain helpful context for debugging
3. **Client-appropriate responses**: API clients receive appropriate HTTP status codes and messages without internal implementation details
4. **Error handling flexibility**: Known errors can be handled specifically, while unknown errors have sensible defaults
5. **Error type safety**: Using exported error variables allows for reliable error checking
6. **Observability**: Errors are consistently logged with appropriate context

## Implementation Guidelines

When implementing error handling in this architecture:

1. Define expected errors as exported variables in your business domain packages
2. Use `fmt.Errorf` with `%w` to add context while preserving original errors
3. Convert errors at layer boundaries (data → business → api)
4. Check for specific errors using `errors.Is()` and `errors.As()`
5. Map domain errors to appropriate HTTP status codes at the API layer
6. Log errors with contextual information
7. Don't expose internal implementation details to API clients
