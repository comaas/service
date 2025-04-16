# Data Transfer Objects (DTOs) in Service Architecture

This document explains how and why DTOs (Data Transfer Objects) are used in the service architecture, with practical examples and implementation guidelines.

## What are DTOs?

Data Transfer Objects are objects designed specifically to transfer data between subsystems or layers of an application. In this service architecture, DTOs are used to transfer data between:

1. **API Layer** ↔ **Business Layer**
2. **Business Layer** ↔ **Data Store Layer**

Each layer has its own representation of entities, optimized for its specific concerns.

## Why DTOs are Essential in This Architecture

### 1. Separation of Concerns

Each layer has different responsibilities and data requirements:

- **API Layer**: Concerned with JSON serialization, HTTP validation, client-facing contracts
- **Business Layer**: Concerned with domain logic, validation rules, business operations
- **Data Store Layer**: Concerned with persistence, database mapping, query optimization

DTOs allow each layer to have its own optimized data model.

### 2. Layer-Specific Types

Different layers use different types for the same conceptual data:

```go
// API Layer (string IDs for JSON compatibility)
type ProductAPI struct {
    ID          string  `json:"id"`
    Name        string  `json:"name"`
    Price       float64 `json:"price"`
    DateCreated string  `json:"date_created"`
}

// Business Layer (rich types for domain functionality)
type ProductBusiness struct {
    ID          uuid.UUID
    Name        string
    Price       decimal.Decimal
    DateCreated time.Time
}

// Data Store Layer (types optimized for the database)
type ProductDB struct {
    ID          string    `db:"product_id"`
    Name        string    `db:"name"`
    Price       float64   `db:"price"`
    DateCreated time.Time `db:"date_created"`
}
```

### 3. Security and Information Hiding

DTOs act as a filter to prevent sensitive data from moving between layers:

```go
// Business model with sensitive data
type User struct {
    ID           uuid.UUID
    Email        mail.Address
    PasswordHash []byte
    APIKey       string
    IsAdmin      bool
}

// API model without sensitive data
type UserResponse struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    // Note: PasswordHash, APIKey, and IsAdmin are intentionally omitted
}
```

### 4. API Stability and Versioning

Internal models can change without breaking public APIs:

```go
// Business model can evolve
type UserBusiness struct {
    ID              uuid.UUID
    Name            string
    Email           mail.Address
    PasswordHash    []byte
    MFAEnabled      bool        // New field
    PreferredTheme  string      // New field
}

// API model stays stable for clients
type UserResponse struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}
```

### 5. Validation Control

DTOs provide clear boundaries for input validation:

```go
func toBusNewUser(api NewUserRequest) (userbus.NewUser, error) {
    if api.Password != api.PasswordConfirm {
        return userbus.NewUser{}, errors.New("passwords do not match")
    }
    
    email, err := mail.ParseAddress(api.Email)
    if err != nil {
        return userbus.NewUser{}, fmt.Errorf("invalid email: %w", err)
    }
    
    return userbus.NewUser{
        Name:     api.Name,
        Email:    *email,
        Password: api.Password,
    }, nil
}
```

## DTO Implementation in the Service Architecture

The implementation of DTOs in this architecture follows a consistent pattern:

### 1. DTOs for Different Operations

Different operations often need different DTOs:

```go
// For creating a new user
type NewUser struct {
    Name     string   `json:"name" validate:"required"`
    Email    string   `json:"email" validate:"required,email"`
    Password string   `json:"password" validate:"required,min=8"`
    Roles    []string `json:"roles" validate:"required"`
}

// For updating a user
type UpdateUser struct {
    Name     *string   `json:"name"`
    Email    *string   `json:"email" validate:"omitempty,email"`
    Password *string   `json:"password" validate:"omitempty,min=8"`
    Roles    []string  `json:"roles"`
    Enabled  *bool     `json:"enabled"`
}

// For returning a user
type User struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Email       string    `json:"email"`
    Roles       []string  `json:"roles"`
    Department  string    `json:"department"`
    Enabled     bool      `json:"enabled"`
    DateCreated string    `json:"date_created"`
    DateUpdated string    `json:"date_updated"`
}
```

### 2. Conversion Functions

Explicit functions convert between different model types:

```go
// Converting from API model to Business model
func toBusNewUser(app NewUser) (userbus.NewUser, error) {
    email, err := mail.ParseAddress(app.Email)
    if err != nil {
        return userbus.NewUser{}, fmt.Errorf("parsing email: %w", err)
    }
    
    return userbus.NewUser{
        Name:       app.Name,
        Email:      *email,
        Password:   app.Password,
        Roles:      app.Roles,
        Department: app.Department,
    }, nil
}

// Converting from Business model to API model
func toAppUser(usr userbus.User) User {
    return User{
        ID:          usr.ID.String(),
        Name:        usr.Name,
        Email:       usr.Email.String(),
        Roles:       usr.Roles,
        Department:  usr.Department,
        Enabled:     usr.Enabled,
        DateCreated: usr.DateCreated.Format(time.RFC3339),
        DateUpdated: usr.DateUpdated.Format(time.RFC3339),
    }
}
```

### 3. Pointer Fields for Optional Updates

For update operations, pointer fields indicate which fields to update:

```go
// In the API handler
func (a *app) update(ctx context.Context, r *http.Request) web.Encoder {
    var app UpdateUser
    if err := web.Decode(r, &app); err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    // Convert to business model
    uu, err := toBusUpdateUser(app)
    if err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    // Get the user to update
    usr, err := mid.GetUser(ctx)
    if err != nil {
        return errs.Newf(errs.Internal, "user missing in context: %s", err)
    }

    // Update the user with the business logic
    updUsr, err := a.userBus.Update(ctx, usr, uu)
    if err != nil {
        return errs.Newf(errs.Internal, "update: userID[%s] uu[%+v]: %s", usr.ID, uu, err)
    }

    // Convert back to API model for response
    return toAppUser(updUsr)
}

// In the business layer
func (b *Business) Update(ctx context.Context, usr User, uu UpdateUser) (User, error) {
    // Only update fields that were provided
    if uu.Name != nil {
        usr.Name = *uu.Name
    }
    
    if uu.Email != nil {
        usr.Email = *uu.Email
    }
    
    if uu.Roles != nil {
        usr.Roles = uu.Roles
    }
    
    if uu.Password != nil {
        pw, err := bcrypt.GenerateFromPassword([]byte(*uu.Password), bcrypt.DefaultCost)
        if err != nil {
            return User{}, fmt.Errorf("generatefrompassword: %w", err)
        }
        usr.PasswordHash = pw
    }
    
    usr.DateUpdated = time.Now()
    
    if err := b.storer.Update(ctx, usr); err != nil {
        return User{}, fmt.Errorf("update: %w", err)
    }
    
    return usr, nil
}
```

## Guidelines for Using DTOs in Your Own Code

When implementing DTOs in your own service architecture, follow these guidelines:

### 1. Define Clear Layer Boundaries

Decide which layers need their own data models:
- API/Presentation Layer
- Business/Domain Layer
- Data Access Layer

### 2. Create Explicit Conversion Functions

Always use explicit functions to convert between models:
- Name them clearly (e.g., `toBusinessUser`, `toAPIUser`)
- Keep them in the same file as the DTO definitions
- Add proper error handling for type conversions

### 3. Be Deliberate About Data Hiding

Consider which fields should be exposed at each layer:
- Never expose password hashes, internal IDs, or sensitive data in API models
- Include only the fields necessary for each operation
- Use separate DTOs for different operations (create vs. read vs. update)

### 4. Use Appropriate Types for Each Layer

Choose appropriate types for each layer:
- API Layer: JSON-friendly types (string, number, boolean, arrays)
- Business Layer: Rich domain types (uuid.UUID, time.Time, custom types)
- Data Layer: Database-compatible types

### 5. Include Validation Tags

Add validation tags to API DTOs:
```go
type NewUser struct {
    Name     string `json:"name" validate:"required"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}
```

### 6. Handle Optional Fields Appropriately

For update operations:
- Use pointer types for optional fields
- Check if the pointer is non-nil before updating
- Document which fields are optional

```go
type UpdateProduct struct {
    Name        *string  `json:"name"`
    Description *string  `json:"description"`
    Price       *float64 `json:"price" validate:"omitempty,gt=0"`
}
```

## Practical Example: Product Management

Here's a complete example showing the DTO pattern for a product management feature:

### API Models (app/domain/productapp/model.go)

```go
package productapp

import (
    "time"
    "github.com/ardanlabs/service/business/domain/productbus"
)

// NewProduct represents a product creation request
type NewProduct struct {
    Name        string  `json:"name" validate:"required"`
    Description string  `json:"description"`
    Price       float64 `json:"price" validate:"required,gt=0"`
    Inventory   int     `json:"inventory" validate:"gte=0"`
}

// UpdateProduct represents a product update request
type UpdateProduct struct {
    Name        *string  `json:"name"`
    Description *string  `json:"description"`
    Price       *float64 `json:"price" validate:"omitempty,gt=0"`
    Inventory   *int     `json:"inventory" validate:"omitempty,gte=0"`
}

// Product represents a product response
type Product struct {
    ID          string  `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Inventory   int     `json:"inventory"`
    DateCreated string  `json:"date_created"`
    DateUpdated string  `json:"date_updated"`
}

// Convert API model to business model
func toBusNewProduct(np NewProduct) productbus.NewProduct {
    return productbus.NewProduct{
        Name:        np.Name,
        Description: np.Description,
        Price:       np.Price,
        Inventory:   np.Inventory,
    }
}

// Convert API model to business model (update)
func toBusUpdateProduct(up UpdateProduct) productbus.UpdateProduct {
    return productbus.UpdateProduct{
        Name:        up.Name,
        Description: up.Description,
        Price:       up.Price,
        Inventory:   up.Inventory,
    }
}

// Convert business model to API model
func toAppProduct(p productbus.Product) Product {
    return Product{
        ID:          p.ID.String(),
        Name:        p.Name,
        Description: p.Description,
        Price:       p.Price,
        Inventory:   p.Inventory,
        DateCreated: p.DateCreated.Format(time.RFC3339),
        DateUpdated: p.DateUpdated.Format(time.RFC3339),
    }
}

// Convert a slice of business models to API models
func toAppProducts(products []productbus.Product) []Product {
    result := make([]Product, len(products))
    for i, p := range products {
        result[i] = toAppProduct(p)
    }
    return result
}
```

### Business Models (business/domain/productbus/product.go)

```go
package productbus

import (
    "time"
    "github.com/google/uuid"
)

// Product represents a product in the business domain
type Product struct {
    ID          uuid.UUID
    Name        string
    Description string
    Price       float64
    Inventory   int
    DateCreated time.Time
    DateUpdated time.Time
}

// NewProduct represents data needed to create a product
type NewProduct struct {
    Name        string
    Description string
    Price       float64
    Inventory   int
}

// UpdateProduct contains fields that can be updated
type UpdateProduct struct {
    Name        *string
    Description *string
    Price       *float64
    Inventory   *int
}
```

### Database Models (business/domain/productbus/stores/productdb/models.go)

```go
package productdb

import (
    "time"
    "github.com/google/uuid"
    "github.com/ardanlabs/service/business/domain/productbus"
)

// product represents a product in the database
type product struct {
    ID          string    `db:"product_id"`
    Name        string    `db:"name"`
    Description string    `db:"description"`
    Price       float64   `db:"price"`
    Inventory   int       `db:"inventory"`
    DateCreated time.Time `db:"date_created"`
    DateUpdated time.Time `db:"date_updated"`
}

// Convert business model to database model
func toDBProduct(p productbus.Product) product {
    return product{
        ID:          p.ID.String(),
        Name:        p.Name,
        Description: p.Description,
        Price:       p.Price,
        Inventory:   p.Inventory,
        DateCreated: p.DateCreated,
        DateUpdated: p.DateUpdated,
    }
}

// Convert database model to business model
func toBusProduct(p product) (productbus.Product, error) {
    id, err := uuid.Parse(p.ID)
    if err != nil {
        return productbus.Product{}, err
    }

    return productbus.Product{
        ID:          id,
        Name:        p.Name,
        Description: p.Description,
        Price:       p.Price,
        Inventory:   p.Inventory,
        DateCreated: p.DateCreated,
        DateUpdated: p.DateUpdated,
    }, nil
}

// Convert a slice of database models to business models
func toBusProducts(products []product) ([]productbus.Product, error) {
    result := make([]productbus.Product, len(products))
    for i, p := range products {
        var err error
        result[i], err = toBusProduct(p)
        if err != nil {
            return nil, err
        }
    }
    return result, nil
}
```

## Conclusion

DTOs are a critical pattern in this service architecture that provide clear separation between layers, improve security, maintain API stability, and support proper validation. While they add some overhead in terms of code and conversions, the benefits in maintainability, security, and architectural clarity far outweigh the costs.

By following the patterns established in this architecture, you can create robust, secure, and maintainable services with clean separation between different concerns.
