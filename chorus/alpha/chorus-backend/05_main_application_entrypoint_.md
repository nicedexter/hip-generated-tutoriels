# Chapter 5: Main Application Entrypoint

In [Chapter 4: Logger Tool](04_logger_tool_.md) we learned how to make sense of raw JSON logs. Now it’s time to see **how the whole backend actually starts**—from reading your settings, wiring up the database and HTTP/gRPC handlers, to finally listening for requests. This is the **“chorus” binary**, the main server application.

## Why Do We Need an Entrypoint?

Think of your backend as a car. The **Main Application Entrypoint** is the ignition switch. When you “turn it on,” it:

- Reads your configuration (which tracks, fuel, and oil levels).
- Hooks up every component (engine, steering, lights).
- Connects to the database (your fuel tank).
- Starts the HTTP/gRPC servers (the wheels rolling).
- Waits for incoming requests (you start driving).

Without this entrypoint, nothing runs!

## Core Concepts

1. **CLI Binary**  
   The `chorus` program you invoke to launch the server.

2. **Configuration Loading**  
   Flags or environment variables that tell the app which port to use, where the database lives, and what log level you want.

3. **Component Initialization**  
   Setting up the logger, database connection, HTTP router, gRPC server, and any other services.

4. **Server Startup**  
   Kicking off HTTP and gRPC listeners so your backend can receive requests.

## Running the Backend

Here’s a minimal example of starting the server in development:

```bash
go run cmd/chorus/main.go \
  --db-url "postgres://admin:password@localhost:5432/chorus?sslmode=disable" \
  --http-port 8080 \
  --grpc-port 9090 \
  --log-level debug \
| go run cmd/logger/main.go
```

What happens:

1. `main.go` invokes the root command.
2. Flags set your DB URL, ports, and log level.
3. Logs flow through **cmd/logger** for nice colors and filtering.
4. Your backend listens on HTTP port 8080 and gRPC port 9090.

## What Happens Under the Hood?

Here’s a bird’s-eye view of the startup flow:

```mermaid
sequenceDiagram
  participant Dev as You
  participant CLI as chorus
  participant CFG as Config Loader
  participant INIT as Component Setup
  participant SRV as HTTP/gRPC Servers

  Dev->>CLI: run chorus --flags
  CLI->>CFG: parse flags & env
  CFG-->>CLI: return config object
  CLI->>INIT: init logger, DB, routers
  INIT-->>CLI: components ready
  CLI->>SRV: start HTTP & gRPC listeners
```

1. **parse flags & env** → build a `Config` struct.  
2. **init logger** → set global log level.  
3. **connect DB** → open Postgres connection.  
4. **setup routers** → register REST and gRPC handlers.  
5. **start servers** → begin listening for requests.

## A Peek at the Code

### 1. Entry File: `cmd/chorus/main.go`

This is the only file your `go run` touches directly:

```go
package main

import "github.com/CHORUS-TRE/chorus-backend/internal/cmd"

func main() {
    cmd.Execute()
}
```

- `cmd.Execute()` sets up and runs the root Cobra command.

### 2. Defining the Root Command: `internal/cmd/root.go`

```go
var rootCmd = &cobra.Command{
    Use:   "chorus",
    Short: "Start the chorus backend server",
    RunE:  runServer, // function that actually boots everything
}

func Execute() error {
    // Define flags
    rootCmd.Flags().String("db-url", "", "Database connection URL")
    rootCmd.Flags().Int("http-port", 8080, "HTTP listen port")
    rootCmd.Flags().Int("grpc-port", 9090, "gRPC listen port")
    rootCmd.Flags().String("log-level", "info", "Log verbosity")
    return rootCmd.Execute()
}
```

- We declare flags for DB, ports, and logging.  
- `RunE: runServer` points to the function that sets up and runs servers.

### 3. Bootstrapping Everything: `internal/cmd/root.go` (continued)

```go
func runServer(cmd *cobra.Command, args []string) error {
    // 1. Load config from flags
    cfg := loadConfig(cmd)

    // 2. Initialize logger
    logger := initLogger(cfg.LogLevel)

    // 3. Connect to database
    db, err := connectDB(cfg.DBURL)
    if err != nil {
        return err
    }
    defer db.Close()

    // 4. Prepare HTTP and gRPC servers
    httpSrv := newHTTPServer(db, logger)
    grpcSrv := newGRPCServer(db, logger)

    // 5. Run servers concurrently
    go httpSrv.ListenAndServe(cfg.HTTPPort)
    go grpcSrv.Serve(cfg.GRPCPort)

    // 6. Block until shutdown signal (Ctrl+C)
    waitForShutdown()

    return nil
}
```

- **loadConfig** grabs flag values into a struct like:
  ```go
  type Config struct {
    DBURL     string
    HTTPPort  int
    GRPCPort  int
    LogLevel  string
  }
  ```
- **initLogger** sets up global logging (see [Chapter 4: Logger Tool](04_logger_tool_.md)).
- **connectDB** opens a Postgres connection (see [Chapter 3: Development Database Setup](03_development_database_setup_.md)).
- **newHTTPServer** and **newGRPCServer** wire your REST and gRPC handlers (see [Chapter 2: Service API Definitions](02_service_api_definitions_.md) and [Chapter 1: Domain Model Schemas](01_domain_model_schemas_.md)).

## Summary

In this chapter, you learned how the **chorus** binary:

- Parses CLI flags and environment into a `Config`.  
- Initializes the logger and database.  
- Sets up HTTP and gRPC servers.  
- Starts listening and handles graceful shutdowns.

Next up, we’ll embed our OpenAPI docs and UI right into the server so you can browse your API in a web page. See [Chapter 6: OpenAPI & UI Embedding](06_openapi___ui_embedding_.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)