## MongoDB


GoFr supports injecting MongoDB that supports the following interface. Any driver that implements the interface can be added
using `app.AddMongo()` method, and user's can use MongoDB across application with `gofr.Context`.
```go
type Mongo interface {
    Find(ctx context.Context, collection string, filter any, results any) error


    FindOne(ctx context.Context, collection string, filter any, result any) error


    InsertOne(ctx context.Context, collection string, document any) (any, error)


    InsertMany(ctx context.Context, collection string, documents []any) ([]any, error)


    DeleteOne(ctx context.Context, collection string, filter any) (int64, error)


    DeleteMany(ctx context.Context, collection string, filter any) (int64, error)


    UpdateByID(ctx context.Context, collection string, id any, update any) (int64, error)


    UpdateOne(ctx context.Context, collection string, filter any, update any) error


    UpdateMany(ctx context.Context, collection string, filter any, update any) (int64, error)


    CountDocuments(ctx context.Context, collection string, filter any) (int64, error)


    Drop(ctx context.Context, collection string) error
}
```


User's can easily inject a driver that supports this interface, this provides usability without
compromising the extensibility to use multiple databases.


Import the gofr's external driver for MongoDB:


```shell
go get gofr.dev/pkg/gofr/datasource/mongo@latest
```


### Example
```go
package main


import (
    "time"


    "go.mongodb.org/mongo-driver/bson"
    "gofr.dev/pkg/gofr/datasource/mongo"


    "gofr.dev/pkg/gofr"
)


type Person struct {
    Name string `bson:"name" json:"name"`
    Age  int    `bson:"age" json:"age"`
    City string `bson:"city" json:"city"`
}


func main() {
    app := gofr.New()


    db := mongo.New(mongo.Config{URI: "mongodb://localhost:27017", Database: "test", ConnectionTimeout: 4 * time.Second})


    // inject the mongo into gofr to use mongoDB across the application
    // using gofr context
    app.AddMongo(db)


    app.POST("/mongo", Insert)
    app.GET("/mongo", Get)


    app.Run()
}


func Insert(ctx *gofr.Context) (any, error) {
    var p Person
    err := ctx.Bind(&p)
    if err != nil {
        return nil, err
    }


    res, err := ctx.Mongo.InsertOne(ctx, "collection", p)
    if err != nil {
        return nil, err
    }


    return res, nil
}


func Get(ctx *gofr.Context) (any, error) {
    var result Person


    p := ctx.Param("name")


    err := ctx.Mongo.FindOne(ctx, "collection", bson.D{{"name", p}} /* valid filter */, &result)
    if err != nil {
        return nil, err
    }


    return result, nil
}
```
## MongoDB GoFr supports injecting MongoDB that supports the following interface. Any driver that implements the interface can be added using `app.AddMongo()` method, and user's can use MongoDB across application with `gofr.Context`. ```go type Mongo interface {     Find(ctx context.Context, collection string, filter any, results any) error     FindOne(ctx context.Context, collection string, filter any, result any) error     InsertOne(ctx context.Context, collection string, document any) (any, error)     InsertMany(ctx context.Context, collection string, documents []any) ([]any, error)     DeleteOne(ctx context.Context, collection string, filter any) (int64, error)     DeleteMany(ctx context.Context, collection string, filter any) (int64, error)     UpdateByID(ctx context.Context, collection string, id any, update any) (int64, error)     UpdateOne(ctx context.Context, collection string, filter any, update any) error     UpdateMany(ctx context.Context, collection string, filter any, update any) (int64, error)     CountDocuments(ctx context.Context, collection string, filter any) (int64, error)     Drop(ctx context.Context, collection string) error } ``` User's can easily inject a driver that supports this interface, this provides usability without compromising the extensibility to use multiple databases. Import the gofr's external driver for MongoDB: ```shell go get gofr.dev/pkg/gofr/datasource/mongo@latest ``` ### Example ```go package main import (     "time"     "go.mongodb.org/mongo-driver/bson"     "gofr.dev/pkg/gofr/datasource/mongo"     "gofr.dev/pkg/gofr" ) type Person struct {     Name string `bson:"name" json:"name"`     Age  int    `bson:"age" json:"age"`     City string `bson:"city" json:"city"` } func main() {     app := gofr.New()     db := mongo.New(mongo.Config{URI: "mongodb://localhost:27017", Database: "test", ConnectionTimeout: 4 * time.Second})     // inject the mongo into gofr to use mongoDB across the application     // using gofr context     app.AddMongo(db)     app.POST("/mongo", Insert)     app.GET("/mongo", Get)     app.Run() } func Insert(ctx *gofr.Context) (any, error) {     var p Person     err := ctx.Bind(&p)     if err != nil {         return nil, err     }     res, err := ctx.Mongo.InsertOne(ctx, "collection", p)     if err != nil {         return nil, err     }     return res, nil } func Get(ctx *gofr.Context) (any, error) {     var result Person     p := ctx.Param("name")     err := ctx.Mongo.FindOne(ctx, "collection", bson.D{{"name", p}} /* valid filter */, &result)     if err != nil {         return nil, err     }     return result, nil } ```
MongoDB with GoFr — Project Structure, Config, Docker, and Best Practices
Below is a clean, production-friendly setup for using MongoDB with GoFr. It includes:

Organized project layout

How to load configuration via app.Config.Get/GetOrDefault

Why configs/.env is the right place (GoFr auto-loads env files there)

MongoDB-specific Docker/Compose with healthchecks and volumes

Ready-to-run example code using the GoFr Mongo driver

Good practices for local and containerized environments

Project Structure
configs/

.env ← place env files here; GoFr auto-loads them

.env.docker ← overrides for Docker Compose (optional)

cmd/

server/

main.go ← app bootstrap; reads config via app.Config

internal/

handlers/ ← HTTP handlers (Mongo queries, DTO binding)

services/ ← business logic (optional)

storage/ ← repositories (optional)

models/ ← domain structs (e.g., Person)

Dockerfile

docker-compose.yml

go.mod

go.sum

Makefile ← helper tasks (optional)

README.md

Why configs/.env
GoFr automatically looks for environment files in configs and loads them at startup. No manual loader is needed.

Use app.Config.Get or app.Config.GetOrDefault anywhere in the app to read values that are loaded from configs/.env.

Keep multiple files if needed (e.g., configs/.env for local; configs/.env.docker for containers).

Example .env (configs/.env)
PORT=8000
MONGO_URI=mongodb://localhost:27017
MONGO_DATABASE=test
MONGO_CONN_TIMEOUT=4s

Notes:

Values are strings; when a type is required (like durations), parse appropriately in code.

Do not commit real production secrets; inject them via CI/secret managers.

Main Application (cmd/server/main.go)
Demonstrates app.Config.Get/GetOrDefault usage.

Injects Mongo via app.AddMongo.

Exposes sample POST/GET endpoints using ctx.Mongo.

go
package main

import (
    "fmt"
    "time"

    "go.mongodb.org/mongo-driver/bson"

    "gofr.dev/pkg/gofr"
    "gofr.dev/pkg/gofr/datasource/mongo"
)

type Person struct {
    Name string `bson:"name" json:"name"`
    Age  int    `bson:"age"  json:"age"`
    City string `bson:"city" json:"city"`
}

func main() {
    app := gofr.New()

    // GoFr auto-loads configs/.env; read values here.
    uri := app.Config.GetOrDefault("MONGO_URI", "mongodb://localhost:27017")
    dbName := app.Config.GetOrDefault("MONGO_DATABASE", "test")
    timeoutStr := app.Config.GetOrDefault("MONGO_CONN_TIMEOUT", "4s")

    connTimeout, err := time.ParseDuration(timeoutStr)
    if err != nil {
        panic(fmt.Errorf("invalid MONGO_CONN_TIMEOUT: %w", err))
    }

    // Init GoFr Mongo driver
    db := mongo.New(mongo.Config{
        URI:               uri,
        Database:          dbName,
        ConnectionTimeout: connTimeout,
    })
    app.AddMongo(db)

    // Routes
    app.POST("/mongo", Insert)
    app.GET("/mongo", GetByName)

    // Optional: bind PORT explicitly; GoFr respects PORT automatically when set.
    // listen := app.Config.GetOrDefault("PORT", "8000")
    // app.Run(":" + listen)

    app.Run()
}

func Insert(ctx *gofr.Context) (any, error) {
    var p Person
    if err := ctx.Bind(&p); err != nil {
        return nil, err
    }
    res, err := ctx.Mongo.InsertOne(ctx, "people", p)
    if err != nil {
        return nil, err
    }
    return res, nil
}

func GetByName(ctx *gofr.Context) (any, error) {
    var result Person
    name := ctx.Param("name")
    if name == "" {
        return nil, fmt.Errorf("name param is required")
    }
    if err := ctx.Mongo.FindOne(ctx, "people", bson.D{{Key: "name", Value: name}}, &result); err != nil {
        return nil, err
    }
    return result, nil
}
Key points:

app.Config.GetOrDefault ensures sane defaults for local dev.

app.Config.Get is preferred for required values (fail fast if missing).

ctx.Mongo gives access to the injected Mongo according to the GoFr Mongo interface.

Dockerfile
Multi-stage build for small, secure images.

Copies configs/ into the container so defaults exist at runtime.

text
# Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# Runtime
FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=builder /app/server /app/server
COPY configs/ /app/configs/
EXPOSE 8000
USER nonroot:nonroot
ENTRYPOINT ["/app/server"]
Docker Compose (docker-compose.yml)
MongoDB service with persistence and healthcheck.

App service with env_file pointing to configs/.env.docker (recommended) or configs/.env.

When running in Compose, set MONGO_URI to use the service name mongo as host.

text
version: "3.8"

services:
  mongo:
    image: mongo:6.0
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 10

  app:
    build: .
    depends_on:
      mongo:
        condition: service_healthy
    ports:
      - "8000:8000"
    env_file:
      - ./configs/.env.docker
    # optional healthcheck for the app; adjust to your readiness endpoint
    healthcheck:
      test: ["CMD", "sh", "-c", "wget -qO- http://localhost:8000/health || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  mongo-data: {}
Example configs/.env.docker:

PORT=8000
MONGO_URI=mongodb://mongo:27017
MONGO_DATABASE=test
MONGO_CONN_TIMEOUT=4s

Notes:

Inside Compose, use host mongo (the service name).

If adding auth, set MONGO_INITDB_ROOT_USERNAME/MONGO_INITDB_ROOT_PASSWORD on the mongo service and update MONGO_URI accordingly.

Using app.Config.Get/GetOrDefault
In main: use app.Config.GetOrDefault for optional settings (e.g., MONGO_CONN_TIMEOUT).

For required secrets, prefer app.Config.Get and validate at startup:

v := app.Config.Get("REQUIRED_KEY"); if v == "" { panic("missing REQUIRED_KEY") }

Handlers can also access config via ctx.Config.Get/ctx.Config.GetOrDefault when needed, but prefer centralizing reads in main to validate once.

Good Practices
Configuration

Keep configs/.env for local defaults; use configs/.env.docker when running Compose so hostnames match services.

Do not commit real credentials; use deployment-specific secret injection (GitHub Actions, Kubernetes secrets, etc.).

Validate required config at startup to fail fast.

MongoDB

Use a dedicated database name per environment (e.g., app_dev, app_staging, app_prod).

Create indexes via a startup routine or migration step where applicable.

Keep collection names constant; centralize them as constants to avoid typos.

Code Organization

Keep handlers thin; push logic into services and repositories as the app grows.

Define request/response DTOs in internal/models; avoid leaking DB-specific types across layers.

Docker

Multi-stage builds for small images; run as nonroot.

Healthchecks for both Mongo and app containers.

Use volumes for Mongo data in development.

Local vs Docker

Local: MONGO_URI=mongodb://localhost:27017

Docker: MONGO_URI=mongodb://mongo:27017 (service name)

Keep both in separate env files, or override via env_file in Compose.

Quick Start
Local without Docker:

Start MongoDB locally on 27017.

Put configs/.env with local values.

go run ./cmd/server

With Docker:

Create configs/.env.docker using mongo service host.

docker compose up --build

App available at http://localhost:8000

Example Endpoints
POST /mongo

Body: {"name": "Alice", "age": 30, "city": "Pune"}

Inserts a document into people collection.

GET /mongo?name=Alice

Returns the Person document with name=Alice.

This structure and setup align with GoFr’s configuration loading model, keep .env in configs/ so it’s loaded automatically, and provide a solid MongoDB foundation for both local development and containerized deployments.