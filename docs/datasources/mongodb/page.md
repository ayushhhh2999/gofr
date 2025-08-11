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

```graphql
Project Structure
Mongo-app
  configs/ 
       .env
  main.go
  Dockerfile
  docker-compose.yml
  go.mod
  go.sum
```
* GoFr automatically looks for environment files in a configs directory and loads them at startup.
```graphql
 internal/ 
       handlers/ (optional) 
       models/ (optional)
```
* At production level it's a good practice to create sperate files for your hadler function and models
* You can import them like this 
```go
import (
"github.com/repo-name/module-name/handlers"
"github.com/repo-name/module-name/models"
)
```


Import the gofr's external driver for MongoDB:

```shell
go get gofr.dev/pkg/gofr/datasource/mongo@latest
```

### Example
.env file
```.env
PORT="8000"
MONGODB_URI="mongodb://localhost:27017/"
MONGODB_DATABASE="test"
```
main.go
```go
package main

import (
	"time"

	"go.mongodb.org/mongo-driver/bson"
	"gofr.dev/pkg/gofr/datasource/mongo"

	"gofr.dev/pkg/gofr"
)

// Person struct represents the data model for MongoDB documents.
// The struct tags (`bson` & `json`) ensure correct mapping for both MongoDB and JSON APIs.
type Person struct {
	Name string `bson:"name" json:"name"`
	Age  int    `bson:"age" json:"age"`
	City string `bson:"city" json:"city"`
}

func main() {
	// Initialize a new Gofr application instance.
	app := gofr.New()
    Load MongoDB configuration from environment variables.
	db := mongo.New(mongo.Config{
		URI:               app.Config.Get("MONGODB_URI"),
		Database:          app.Config.Get("MONGODB_DATABASE"),
		ConnectionTimeout: 4 * time.Second, // Connection timeout to avoid hanging connections
	})

	// Inject the MongoDB client into Gofr's application context
	// This allows handlers to access MongoDB easily via ctx.Mongo
	app.AddMongo(db)

	// Define API routes
	app.POST("/mongo", Insert) // Route for inserting a document
	app.GET("/mongo", Get)     // Route for fetching a document by name

	// Start the server
	app.Run()
}

// Insert handles POST requests to add a new document to MongoDB.
func Insert(ctx *gofr.Context) (any, error) {
	var p Person

	// Bind incoming JSON request body to the Person struct
	err := ctx.Bind(&p)
	if err != nil {
		return nil, err
	}

	// Insert the Person object into the "collection" collection
	res, err := ctx.Mongo.InsertOne(ctx, "collection", p)
	if err != nil {
		return nil, err
	}

	// Return the MongoDB insertion result (e.g., inserted ID)
	return res, nil
}

// Get handles GET requests to retrieve a document by "name".
func Get(ctx *gofr.Context) (any, error) {
	var result Person

	// Get the "name" query parameter from the URL
	p := ctx.Param("name")

	// Find a single document with the given name
	err := ctx.Mongo.FindOne(ctx, "collection", bson.D{{"name", p}}, &result)
	if err != nil {
		return nil, err
	}

	// Return the found document
	return result, nil
}
```
Best Practice:
- Store secrets like DB credentials in `.env` or system environment variables.
- Use `app.Config.Get("KEY")` to fetch them.
- Alternatively, use `app.Config.GetOrDefault("KEY", "default_value")`
- to provide a fallback if the variable is missing.


Example:
```go
dbURI := app.Config.GetOrDefault("MONGODB_URI", "mongodb://localhost:27017")
```


Dockerfile
```Dockerfile
FROM golang:1.24 as builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN go build -o server .

EXPOSE 8000

CMD ["./server"]
```
docker-compose.yml
```docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - MONGODB_URI=mongodb://mongo:27017/test
      - MONGODB_DATABASE=test
    depends_on:
      - mongo
    networks:
      - app-network

  mongo:
    image: mongo:7.0
    ports:
      - "27017:27017"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```
