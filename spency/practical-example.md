# Practical Example: Adding a Product Feature

Let's walk through a concrete example of adding a new feature to this service architecture. Imagine we want to add the ability to create products with inventory tracking.

## Step 6: Define the Routes

Finally, we need to connect our HTTP handlers to specific URL routes:

```go
// app/domain/productapp/route.go
package productapp

import (
    "net/http"

    "github.com/ardanlabs/service/app/sdk/auth"
    "github.com/ardanlabs/service/app/sdk/authclient"
    "github.com/ardanlabs/service/app/sdk/mid"
    "github.com/ardanlabs/service/business/domain/productbus"
    "github.com/ardanlabs/service/foundation/logger"
    "github.com/ardanlabs/service/foundation/web"
)

// Config contains all the mandatory systems required by handlers
type Config struct {
    Log        *logger.Logger
    ProductBus *productbus.Business
    AuthClient *authclient.Client
}

// Routes adds specific routes for this group
func Routes(app *web.App, cfg Config) {
    const version = "v1"

    // Set up middleware
    authen := mid.Authenticate(cfg.AuthClient)
    ruleAdmin := mid.Authorize(cfg.AuthClient, auth.RuleAdminOnly)

    // Create an instance of our application handlers
    api := newApp(cfg.ProductBus)

    // Connect HTTP methods and paths to handler functions
    app.HandlerFunc(http.MethodGet, version, "/products", api.query, authen)
    app.HandlerFunc(http.MethodGet, version, "/products/{product_id}", api.queryByID, authen)
    app.HandlerFunc(http.MethodPost, version, "/products", api.create, authen, ruleAdmin)
    app.HandlerFunc(http.MethodPut, version, "/products/{product_id}", api.update, authen, ruleAdmin)
    app.HandlerFunc(http.MethodDelete, version, "/products/{product_id}", api.delete, authen, ruleAdmin)
}
```

## Integration with Main Service

Now that we have all the components for our product feature, we need to integrate it into the main service by:

1. Making sure the database has the necessary tables
2. Adding our product business to the service initialization

Here's how our database schema might look:

```sql
-- Migration file for adding products table
CREATE TABLE products (
    product_id    UUID        PRIMARY KEY,
    name          TEXT        NOT NULL,
    description   TEXT,
    price         DECIMAL     NOT NULL,
    inventory     INT         NOT NULL,
    date_created  TIMESTAMP   NOT NULL,
    date_updated  TIMESTAMP   NOT NULL
);

-- Add indexes for common queries
CREATE INDEX products_name_idx ON products (name);
```

And here's how we would wire up our product feature in the main application:

```go
// In main.go (or similar initialization file)

// Create database connection
db, err := sqldb.Open(sqldb.Config{
    User:     cfg.DB.User,
    Password: cfg.DB.Password,
    Host:     cfg.DB.Host,
    // Other database config...
})
if err != nil {
    return fmt.Errorf("connecting to db: %w", err)
}

// Initialize business layers
delegate := delegate.New(log)
userBus := userbus.NewBusiness(log, delegate, userdb.NewStore(log, db))
productBus := productbus.NewBusiness(log, userBus, delegate, productdb.NewStore(log, db))

// Set up routes
productapp.Routes(app, productapp.Config{
    Log:        log,
    ProductBus: productBus,
    AuthClient: authClient,
})
```

## Testing the Feature

With our service structure, we can write tests at multiple levels:

### Unit Tests for Business Logic

```go
func TestCreateProduct(t *testing.T) {
    // Set up test dependencies
    log := logger.New(os.Stdout, logger.LevelInfo, "TEST")
    
    // Create mock store that implements the Storer interface
    mockStore := &mockProductStore{
        CreateFunc: func(ctx context.Context, product productbus.Product) error {
            // Verify product values
            if product.Name != "Test Product" {
                t.Fatalf("Expected product name %q, got %q", "Test Product", product.Name)
            }
            return nil
        },
    }
    
    // Create business with mock store
    bus := productbus.NewBusiness(log, nil, nil, mockStore)
    
    // Call the method to test
    product, err := bus.Create(context.Background(), productbus.NewProduct{
        Name:        "Test Product",
        Description: "Test Description",
        Price:       9.99,
        Inventory:   10,
    })
    
    // Assert results
    if err != nil {
        t.Fatalf("Error creating product: %v", err)
    }
    
    if product.Name != "Test Product" {
        t.Fatalf("Expected product name %q, got %q", "Test Product", product.Name)
    }
}
```

### API Tests

```go
func TestAPICreateProduct(t *testing.T) {
    // Set up test server with our routes
    app := web.NewApp(
        web.Config{
            Log: logger.New(os.Stdout, logger.LevelInfo, "TEST"),
        },
    )
    
    // Add product routes with test dependencies
    productapp.Routes(app, productapp.Config{
        Log:        log,
        ProductBus: testProductBusiness,
        AuthClient: testAuthClient,
    })
    
    ts := httptest.NewServer(app)
    defer ts.Close()
    
    // Create test request
    payload := `{"name":"Test Product","description":"Test Description","price":9.99,"inventory":10}`
    req, err := http.NewRequest(http.MethodPost, ts.URL+"/v1/products", strings.NewReader(payload))
    if err != nil {
        t.Fatalf("Error creating request: %v", err)
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+testToken)
    
    // Send request
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        t.Fatalf("Error sending request: %v", err)
    }
    defer resp.Body.Close()
    
    // Verify response
    if resp.StatusCode != http.StatusOK {
        t.Fatalf("Expected status code %d, got %d", http.StatusOK, resp.StatusCode)
    }
    
    var result struct {
        ID   string `json:"id"`
        Name string `json:"name"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        t.Fatalf("Error decoding response: %v", err)
    }
    
    if result.Name != "Test Product" {
        t.Fatalf("Expected product name %q, got %q", "Test Product", result.Name)
    }
}
```

## Using the API

Once deployed, clients can interact with our product API:

### Creating a Product

```
POST /v1/products
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Ergonomic Keyboard",
  "description": "Comfortable typing for long coding sessions",
  "price": 129.99,
  "inventory": 50
}
```

### Getting Product List

```
GET /v1/products?page=1&rows=10
Authorization: Bearer <token>
```

### Updating a Product

```
PUT /v1/products/5c2e4dce-e679-4d9c-a3c1-98f8a6956c8b
Authorization: Bearer <token>
Content-Type: application/json

{
  "price": 119.99,
  "inventory": 45
}
```

## Advantages of This Architecture

This approach to building product features provides:

1. **Separation of Concerns**: Each layer has a specific responsibility
2. **Testability**: We can test each layer independently
3. **Maintainability**: Easy to extend with new fields or validations
4. **Security**: Authentication and authorization are handled consistently
5. **API Consistency**: All APIs follow the same patterns

By structuring our code this way, we make it easy to understand and modify the system as requirements change, while maintaining a clean separation between the web API, business logic, and data storage.
```

## Step 1: Understanding What We're Building

First, let's define what we're building in plain language:

- Users should be able to create new products with name, description, price, and inventory count
- Products need validation (price > 0, name not empty)
- Only admin users can create products
- We need API endpoints for creating, updating, listing, and deleting products

## Step 2: Define the Data Models

We'll start by defining our data models at each layer. In this architecture, we typically have:

1. API Models - What the HTTP API receives and returns
2. Business Models - Internal domain representations
3. Database Models - How data is stored in the database

### API Model (app/domain/productapp/model.go)

```go
package productapp

// NewProduct represents what a client can send to create a product
type NewProduct struct {
    Name        string  `json:"name" validate:"required"`
    Description string  `json:"description"`
    Price       float64 `json:"price" validate:"required,gt=0"`
    Inventory   int     `json:"inventory" validate:"gte=0"`
}

// Product represents a product returned to the client
type Product struct {
    ID          string  `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Inventory   int     `json:"inventory"`
    DateCreated string  `json:"date_created"`
    DateUpdated string  `json:"date_updated"`
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

// Convert API model to business model
func toBusNewProduct(np NewProduct) (productbus.NewProduct, error) {
    return productbus.NewProduct{
        Name:        np.Name,
        Description: np.Description,
        Price:       np.Price,
        Inventory:   np.Inventory,
    }, nil
}
```

### Business Model (business/domain/productbus/product.go)

```go
package productbus

import (
    "time"
    "github.com/google/uuid"
)

// Product represents the business model for product entities
type Product struct {
    ID          uuid.UUID
    Name        string
    Description string
    Price       float64
    Inventory   int
    DateCreated time.Time
    DateUpdated time.Time
}

// NewProduct represents the data needed to create a product
type NewProduct struct {
    Name        string
    Description string
    Price       float64
    Inventory   int
}
```

### Database Model (business/domain/productbus/stores/productdb/models.go)

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
```

## Step 3: Create the Business Logic Layer

The business layer contains our domain logic:

```go
// business/domain/productbus/productbus.go
package productbus

import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/ardanlabs/service/business/sdk/delegate"
    "github.com/ardanlabs/service/business/sdk/sqldb"
    "github.com/ardanlabs/service/foundation/logger"
    "github.com/google/uuid"
)

// Define common errors
var (
    ErrNotFound = errors.New("product not found")
)

// Define the data store interface
type Storer interface {
    NewWithTx(tx sqldb.CommitRollbacker) (Storer, error)
    Create(ctx context.Context, product Product) error
    Update(ctx context.Context, product Product) error
    Delete(ctx context.Context, product Product) error
    Query(ctx context.Context, filter QueryFilter, orderBy order.By, page page.Page) ([]Product, error)
    Count(ctx context.Context, filter QueryFilter) (int, error)
    QueryByID(ctx context.Context, productID uuid.UUID) (Product, error)
}

// Business manages the set of APIs for product access
type Business struct {
    log      *logger.Logger
    userBus  *userbus.Business
    delegate *delegate.Delegate
    storer   Storer
}

// NewBusiness constructs a product business API for use
func NewBusiness(log *logger.Logger, userBus *userbus.Business, delegate *delegate.Delegate, storer Storer) *Business {
    return &Business{
        log:      log,
        userBus:  userBus,
        delegate: delegate,
        storer:   storer,
    }
}

// Create adds a new product to the system
func (b *Business) Create(ctx context.Context, np NewProduct) (Product, error) {
    now := time.Now()

    product := Product{
        ID:          uuid.New(),
        Name:        np.Name,
        Description: np.Description,
        Price:       np.Price,
        Inventory:   np.Inventory,
        DateCreated: now,
        DateUpdated: now,
    }

    if err := b.storer.Create(ctx, product); err != nil {
        return Product{}, fmt.Errorf("create: %w", err)
    }

    return product, nil
}

// Update modifies information about a product
func (b *Business) Update(ctx context.Context, product Product, up UpdateProduct) (Product, error) {
    if up.Name != nil {
        product.Name = *up.Name
    }
    
    if up.Description != nil {
        product.Description = *up.Description
    }
    
    if up.Price != nil {
        product.Price = *up.Price
    }
    
    if up.Inventory != nil {
        product.Inventory = *up.Inventory
    }
    
    product.DateUpdated = time.Now()
    
    if err := b.storer.Update(ctx, product); err != nil {
        return Product{}, fmt.Errorf("update: %w", err)
    }
    
    return product, nil
}

// Delete removes a product from the system
func (b *Business) Delete(ctx context.Context, product Product) error {
    if err := b.storer.Delete(ctx, product); err != nil {
        return fmt.Errorf("delete: %w", err)
    }
    
    return nil
}

// Query retrieves a list of existing products
func (b *Business) Query(ctx context.Context, filter QueryFilter, orderBy order.By, page page.Page) ([]Product, error) {
    products, err := b.storer.Query(ctx, filter, orderBy, page)
    if err != nil {
        return nil, fmt.Errorf("query: %w", err)
    }
    
    return products, nil
}

// QueryByID finds the product by the specified ID
func (b *Business) QueryByID(ctx context.Context, productID uuid.UUID) (Product, error) {
    product, err := b.storer.QueryByID(ctx, productID)
    if err != nil {
        return Product{}, fmt.Errorf("query: productID[%s]: %w", productID, err)
    }
    
    return product, nil
}
```

## Step 4: Create the Database Access Layer

The database layer handles SQL operations:

```go
// business/domain/productbus/stores/productdb/productdb.go
package productdb

import (
    "bytes"
    "context"
    "errors"
    "fmt"

    "github.com/ardanlabs/service/business/domain/productbus"
    "github.com/ardanlabs/service/business/sdk/sqldb"
    "github.com/ardanlabs/service/foundation/logger"
    "github.com/google/uuid"
    "github.com/jmoiron/sqlx"
)

// Store manages the set of APIs for product database access
type Store struct {
    log *logger.Logger
    db  sqlx.ExtContext
}

// NewStore constructs the api for data access
func NewStore(log *logger.Logger, db *sqlx.DB) *Store {
    return &Store{
        log: log,
        db:  db,
    }
}

// Create inserts a new product into the database
func (s *Store) Create(ctx context.Context, product productbus.Product) error {
    const q = `
    INSERT INTO products
        (product_id, name, description, price, inventory, date_created, date_updated)
    VALUES
        (:product_id, :name, :description, :price, :inventory, :date_created, :date_updated)`

    if err := sqldb.NamedExecContext(ctx, s.log, s.db, q, toDBProduct(product)); err != nil {
        return fmt.Errorf("namedexeccontext: %w", err)
    }

    return nil
}

// QueryByID gets the specified product from the database
func (s *Store) QueryByID(ctx context.Context, productID uuid.UUID) (productbus.Product, error) {
    data := struct {
        ID string `db:"product_id"`
    }{
        ID: productID.String(),
    }

    const q = `
    SELECT
        product_id, name, description, price, inventory, date_created, date_updated
    FROM
        products
    WHERE 
        product_id = :product_id`

    var dbProd product
    if err := sqldb.NamedQueryStruct(ctx, s.log, s.db, q, data, &dbProd); err != nil {
        if errors.Is(err, sqldb.ErrDBNotFound) {
            return productbus.Product{}, fmt.Errorf("db: %w", productbus.ErrNotFound)
        }
        return productbus.Product{}, fmt.Errorf("db: %w", err)
    }

    return toBusProduct(dbProd)
}

// Other methods (Update, Delete, Query, Count, etc.) would follow...
```

## Step 5: Create the API Handlers

Now we create the HTTP handlers that connect to our business logic:

```go
// app/domain/productapp/productapp.go
package productapp

import (
    "context"
    "errors"
    "net/http"

    "github.com/ardanlabs/service/app/sdk/errs"
    "github.com/ardanlabs/service/app/sdk/mid"
    "github.com/ardanlabs/service/app/sdk/query"
    "github.com/ardanlabs/service/business/domain/productbus"
    "github.com/ardanlabs/service/foundation/web"
    "github.com/google/uuid"
)

type app struct {
    productBus *productbus.Business
}

func newApp(productBus *productbus.Business) *app {
    return &app{
        productBus: productBus,
    }
}

// create adds a new product to the system
func (a *app) create(ctx context.Context, r *http.Request) web.Encoder {
    var app NewProduct
    if err := web.Decode(r, &app); err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    np, err := toBusNewProduct(app)
    if err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    product, err := a.productBus.Create(ctx, np)
    if err != nil {
        return errs.Newf(errs.Internal, "create: product[%+v]: %s", product, err)
    }

    return toAppProduct(product)
}

// queryByID returns a product based on the ID
func (a *app) queryByID(ctx context.Context, r *http.Request) web.Encoder {
    id, err := uuid.Parse(web.Param(r, "product_id"))
    if err != nil {
        return errs.Newf(errs.InvalidArgument, "invalid product ID: %s", err)
    }

    product, err := a.productBus.QueryByID(ctx, id)
    if err != nil {
        if errors.Is(err, productbus.ErrNotFound) {
            return errs.New(errs.NotFound, err)
        }
        return errs.Newf(errs.Internal, "querybyid: productID[%s]: %s", id, err)
    }

    return toAppProduct(product)
}

// query returns a list of products with filtering and pagination
func (a *app) query(ctx context.Context, r *http.Request) web.Encoder {
    qp, err := parseQueryParams(r)
    if err != nil {
        return errs.New(errs.InvalidArgument, err)
    }

    page, err := page.Parse(qp.Page, qp.Rows)
    if err != nil {
        return errs.NewFieldErrors("page", err)
    }

    filter, err := parseFilter(qp)
    if err != nil {
        return err.(*errs.Error)
    }

    orderBy, err := order.Parse(orderByFields, qp.OrderBy, productbus.DefaultOrderBy)
    if err != nil {
        return errs.NewFieldErrors("order", err)
    }

    products, err := a.productBus.Query(ctx, filter, orderBy, page)
    if err != nil {
        return errs.Newf(errs.Internal, "query: %s", err)
    }

    total, err := a.productBus.Count(ctx, filter)
    if err != nil {
        return errs.Newf(errs.Internal, "count: %s", err)
    }

    return query.NewResult(toAppProducts(products), total, page)
}

// update modifies a product in the system
func (a *app) update(ctx context.Context, r *http.Request) web.Encoder {
    id, err := uuid.Parse(web.Param(r, "product_id"))
    if err != nil {
        return errs.Newf(errs.InvalidArgument, "invalid product ID: %s", err)
    }
    
    var app UpdateProduct
    if err := web.Decode(r, &app); err != nil {
        return errs.New(errs.InvalidArgument, err)
    }
    
    up, err := toBusUpdateProduct(app)
    if err != nil {
        return errs.New(errs.InvalidArgument, err)
    }
    
    product, err := a.productBus.QueryByID(ctx, id)
    if err != nil {
        if errors.Is(err, productbus.ErrNotFound) {
            return errs.New(errs.NotFound, err)
        }
        return errs.Newf(errs.Internal, "querybyid: productID[%s]: %s", id, err)
    }
    
    updProduct, err := a.productBus.Update(ctx, product, up)
    if err != nil {
        return errs.Newf(errs.Internal, "update: productID[%s] up[%+v]: %s", id, up, err)
    }
    
    return toAppProduct(updProduct)
}

// delete removes a product from the system
func (a *app) delete(ctx context.Context, r *http.Request) web.Encoder {
    id, err := uuid.Parse(web.Param(r, "product_id"))
    if err != nil {
        return errs.Newf(errs.InvalidArgument, "invalid product ID: %s", err)
    }
    
    product, err := a.productBus.QueryByID(ctx, id)
    if err != nil {
        if errors.Is(err, productbus.ErrNotFound) {
            return errs.New(errs.NotFound, err)
        }
        return errs.Newf(errs.Internal, "querybyid: productID[%s]: %s", id, err)
    }
    
    if err := a.productBus.Delete(ctx, product); err != nil {
        return errs.Newf(errs.Internal, "delete: productID[%s]: %s", id, err)
    }
    
    return nil
}