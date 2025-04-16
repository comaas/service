# The Delegate Pattern in Service Architecture

## What is the Delegate Pattern?

In this service architecture, the Delegate pattern provides a mechanism for cross-domain communication while maintaining separation of concerns. It allows one domain to notify other domains about significant events without creating direct dependencies between them.

The pattern acts as an event notification system where:
- Domains can register handlers for specific events
- Domains can trigger events when important changes occur
- The delegate routes these events to the appropriate handlers

## Implementation in the Service Architecture

The Delegate pattern is implemented through the `delegate` package in the business SDK:

```go
// Simplified implementation of the delegate package
package delegate

import (
    "context"
    "fmt"
    "sync"
)

// Action represents an event that domains can subscribe to
type Action string

// Data represents information about the event
type Data interface{}

// Handler defines a function that handles a specific action
type Handler func(ctx context.Context, data Data) error

// Delegate manages a set of handlers for different actions
type Delegate struct {
    mu       sync.RWMutex
    handlers map[Action][]Handler
}

// New constructs a new Delegate
func New(log *logger.Logger) *Delegate {
    return &Delegate{
        handlers: make(map[Action][]Handler),
    }
}

// Register adds a handler for a specific action
func (d *Delegate) Register(action Action, handler Handler) {
    d.mu.Lock()
    defer d.mu.Unlock()
    
    d.handlers[action] = append(d.handlers[action], handler)
}

// Call executes all handlers registered for a specific action
func (d *Delegate) Call(ctx context.Context, action Action, data Data) error {
    d.mu.RLock()
    handlers, exists := d.handlers[action]
    d.mu.RUnlock()
    
    if !exists {
        return nil
    }
    
    for _, handler := range handlers {
        if err := handler(ctx, data); err != nil {
            return fmt.Errorf("handler error: %w", err)
        }
    }
    
    return nil
}
```

## When and Why to Use the Delegate Pattern

### Cross-Domain Communication

The pattern is used when one domain needs to notify others about significant events. For example, when a user is updated:

```go
// In userbus.go
func (b *Business) Update(ctx context.Context, usr User, uu UpdateUser) (User, error) {
    // Update user logic...
    
    // Notify other domains about the user update
    if err := b.delegate.Call(ctx, ActionUpdatedData(uu, usr.ID)); err != nil {
        return User{}, fmt.Errorf("failed to execute `%s` action: %w", ActionUpdated, err)
    }
    
    return usr, nil
}
```

### Domain Actions

The user domain defines specific actions that other domains can subscribe to:

```go
// In userbus/action.go
const (
    ActionCreated Action = "user.created"
    ActionUpdated Action = "user.updated"
    ActionDeleted Action = "user.deleted"
)

// ActionUpdatedData constructs the data for the user.updated event
func ActionUpdatedData(uu UpdateUser, userID uuid.UUID) (Action, Data) {
    return ActionUpdated, struct {
        UpdateUser UpdateUser
        UserID     uuid.UUID
    }{
        UpdateUser: uu,
        UserID:     userID,
    }
}
```

### Registering Handlers

Other domains can register handlers for these actions during service initialization:

```go
// In main.go or during initialization
func setupDomains(log *logger.Logger, db *sqlx.DB) (*userbus.Business, *productbus.Business) {
    // Create the delegate
    delegate := delegate.New(log)
    
    // Create the user business
    userBus := userbus.NewBusiness(log, delegate, userdb.NewStore(log, db))
    
    // Create the product business and register a handler for user updates
    productBus := productbus.NewBusiness(log, userBus, delegate, productdb.NewStore(log, db))
    
    // Register a handler for when users are updated
    delegate.Register(userbus.ActionUpdated, func(ctx context.Context, data Data) error {
        // Cast the data to access the specific event information
        eventData := data.(struct {
            UpdateUser userbus.UpdateUser
            UserID     uuid.UUID
        })
        
        // Perform product-specific business logic when a user is updated
        return productBus.HandleUserUpdate(ctx, eventData.UserID, eventData.UpdateUser)
    })
    
    return userBus, productBus
}
```

## Benefits of the Delegate Pattern

### 1. Loose Coupling

The pattern allows domains to communicate without direct dependencies:
- The user domain doesn't need to import or directly call the product domain
- Domains only need to know about the delegate and the actions they care about

### 2. Extensibility

New domains can be added without modifying existing ones:
- The user domain emits events regardless of who's listening
- New domains can register handlers for existing events
- The system becomes more pluggable and modular

### 3. Separation of Concerns

Each domain can focus on its core responsibilities:
- The user domain manages users without worrying about side effects in other domains
- The product domain can react to user changes without handling user management
- Cross-cutting concerns are separated from domain-specific logic

### 4. Testability

Domains become easier to test in isolation:
- You can mock the delegate for unit tests
- You can verify that the correct events were triggered
- You can test event handlers independently

## Practical Examples

### Example 1: User Role Changes and Product Relationships

There are two approaches to handling how user role changes affect products:

#### Approach A: Authorization-Based (Preferred for Simple Cases)

In most systems, user access to products would be handled through authorization checks at request time:

```go
// In productapp.go
func (a *app) viewProduct(ctx context.Context, r *http.Request) web.Encoder {
    user, err := mid.GetUser(ctx)
    if err != nil {
        return errs.New(errs.Unauthorized, err)
    }
    
    productID, err := uuid.Parse(web.Param(r, "product_id"))
    if err != nil {
        return errs.New(errs.InvalidArgument, err)
    }
    
    product, err := a.productBus.QueryByID(ctx, productID)
    if err != nil {
        // Handle error...
    }
    
    // Authorization check happens at request time
    if !a.authorizer.CanAccessProduct(user.Roles, product) {
        return errs.New(errs.Forbidden, errors.New("insufficient permissions"))
    }
    
    return toAppProduct(product)
}
```

This approach is:
- Simpler (no cascading updates)
- More consistent (single point of truth)
- Typically more performant (no batch operations)

#### Approach B: Delegate Pattern (For Complex Scenarios)

The delegate pattern might be used in more complex scenarios where authorization alone isn't sufficient:

```go
// In productbus.go
func (b *Business) HandleUserRoleChange(ctx context.Context, userID uuid.UUID, oldRoles, newRoles []string) error {
    // This would be used when:
    // 1. The system maintains complex product-role relationships in a separate table
    // 2. There are business rules that go beyond simple authorization checks
    // 3. Role changes trigger specific business logic related to products
    
    // For example, when a user is promoted to admin:
    if containsRole(newRoles, "ADMIN") && !containsRole(oldRoles, "ADMIN") {
        // Grant them ownership of certain products
        if err := b.assignAdminProducts(ctx, userID); err != nil {
            return fmt.Errorf("assigning admin products: %w", err)
        }
    }
    
    // Or when a user loses a specialized role:
    if containsRole(oldRoles, "PRODUCT_MANAGER") && !containsRole(newRoles, "PRODUCT_MANAGER") {
        // Reassign their managed products to another manager
        if err := b.reassignManagedProducts(ctx, userID); err != nil {
            return fmt.Errorf("reassigning managed products: %w", err)
        }
    }
    
    return nil
}
```

This approach makes sense when:
1. Role changes trigger business logic beyond simple access control
2. The system needs to maintain complex relationships between users and products
3. There are actions that must happen immediately when roles change, not at next access
```

### Example 2: User Deletion Cascading to Products

When a user is deleted, their products might need cleanup:

```go
// Register a handler for user deletion
delegate.Register(userbus.ActionDeleted, func(ctx context.Context, data Data) error {
    userID := data.(uuid.UUID)
    
    // Archive or reassign the user's products
    return productBus.HandleUserDeletion(ctx, userID)
})
```

### Example 3: Multiple Domains Reacting to Events

Various domains can react to the same event:

```go
// Product domain reacting to user creation
delegate.Register(userbus.ActionCreated, func(ctx context.Context, data Data) error {
    newUser := data.(userbus.User)
    return productBus.CreateDefaultProducts(ctx, newUser.ID)
})

// Notification domain reacting to user creation
delegate.Register(userbus.ActionCreated, func(ctx context.Context, data Data) error {
    newUser := data.(userbus.User)
    return notificationBus.SendWelcomeEmail(ctx, newUser.Email)
})
```

## When Not to Use the Delegate Pattern

While powerful, the Delegate pattern isn't appropriate for all scenarios:

1. **When Direct Calls are Clearer**: For simple, direct dependencies where loose coupling isn't needed

2. **For Synchronous Operations that Must Succeed**: When the operation must succeed or fail atomically with the triggering operation

3. **When Order of Execution is Critical**: When handlers must execute in a specific order with guarantees

4. **For High-Performance Operations**: The pattern adds some overhead and indirection

5. **For Simple Authorization Checks**: Simple permission checks based on user roles are better handled through authorization middleware at request time rather than through delegates

## Delegate Pattern vs. Authorization

It's important to distinguish between scenarios that call for the delegate pattern versus those better handled by authorization:

### Use Authorization When:
- Checking if a user has permission to access a resource
- Enforcing role-based access control
- Validating permissions at request time
- Implementing consistent access control policies

### Use the Delegate Pattern When:
- Different domains need to react to events
- Complex business processes span multiple domains
- Changes in one domain trigger workflows in other domains
- Cross-cutting concerns need to be separated from core domain logic

## Implementation Guidelines

When implementing the Delegate pattern in your own services:

1. **Define Clear Actions**: Name actions clearly using a domain.verb pattern (e.g., "user.updated")

2. **Standardize Data Structures**: Use consistent structures for event data

3. **Handle Errors Appropriately**: Decide whether handler errors should fail the entire operation

4. **Consider Asynchronous Options**: For non-critical operations, consider using asynchronous processing

5. **Document Dependencies**: Clearly document which domains listen to which events

6. **Be Mindful of Cycles**: Avoid circular event dependencies between domains

## Conclusion

The Delegate pattern is a powerful tool in a service architecture that enables loose coupling between domains while allowing for complex cross-domain business logic. It supports a modular, extensible architecture where domains can evolve independently while still coordinating on important events.

By implementing this pattern, the service architecture achieves a balance between separation of concerns and necessary cross-domain communication, resulting in a more maintainable and flexible system.
