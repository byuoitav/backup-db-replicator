# Backup Couch Database Replicator

A robust Go service for replicating Apache CouchDB databases from a source to a target database on a configurable time interval. Unlike CouchDB's native replication which operates either continuously or on-demand, this utility bridges the gap by providing **scheduled, interval-based replication** - perfect for automated backup strategies.

## Overview

CouchDB natively supports two replication modes:
- **Continuous**: Always replicating, consuming resources
- **Ad-hoc**: One-time manual replication

This service solves the need for **periodic backups** by executing replication jobs on a fixed schedule. It's ideal for maintaining synchronized backup servers of production CouchDB instances.

## Features

- **Scheduled Replication**: Execute replication jobs at configurable time intervals
- **Multiple Databases**: Replicate multiple CouchDB databases in a single configuration
- **Selective Replication**: Filter documents by ID selector for targeted backups
- **HTTP API**: Trigger manual replication via REST endpoint
- **Health Checks**: Automatic verification of source and target database connectivity
- **Structured Logging**: Detailed logging using Uber's Zap logger with configurable log levels
- **Docker Support**: Pre-built Docker images for easy deployment
- **Cross-platform Builds**: Compiled binaries for Linux AMD64 and ARM architectures

## Architecture

The service consists of three main components:

1. **HTTP Server** (default port 7012): Exposes REST endpoints for manual replication triggers
2. **Replicator Engine**: Manages scheduled replication jobs based on configured intervals
3. **Database Connections**: Maintains connections to source and target CouchDB instances

The replicator continuously:
- Verifies database connectivity at startup
- Executes replication jobs on the configured time interval
- Logs replication status and any errors
- Exposes an endpoint for manual replication triggers

## Installation

### Prerequisites

- Go 1.19 or higher
- Apache CouchDB 2.0+ (source and target instances)
- Network connectivity between replicator and both CouchDB instances

### Building from Source

```bash
# Clone the repository
git clone https://github.com/byuoitav/db-replicator.git
cd db-replicator

# Build binaries
make build

# Binaries will be created in dist/
```

### Docker

Pull pre-built images from GitHub Package Registry:

```bash
docker pull docker.pkg.github.com/byuoitav/db-replicator/db-replicator:latest
```

## Configuration

The service requires a JSON configuration file specifying source and target databases along with replication jobs.

### Configuration Schema

| Field | Type | Description |
|-------|------|-------------|
| `source` | object | Source CouchDB connection credentials |
| `target` | object | Target CouchDB connection credentials |
| `jobs` | array | Array of replication job definitions |
| `time_interval` | integer | Interval between replications (in minutes) |

### Connection Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | Yes | CouchDB server URL (e.g., `http://localhost:5984`) |
| `username` | string | Yes | CouchDB username |
| `password` | string | Yes | CouchDB password |

### Job Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `database` | string | Yes | Name of the database to replicate |
| `continuous` | boolean | Yes | Whether to use continuous replication (typically `false`) |
| `id_selector` | string | No | Filter documents by ID prefix (empty string = replicate all) |

### Example Configuration

```json
{
    "source": {
        "address": "http://localhost:42069",
        "username": "admin",
        "password": "secure_password"
    },
    "target": {
        "address": "http://localhost:42068",
        "username": "admin",
        "password": "secure_password"
    },
    "jobs": [
        {
            "database": "devices",
            "continuous": false,
            "id_selector": "ABC-123"
        },
        {
            "database": "rooms",
            "continuous": false,
            "id_selector": ""
        }
    ],
    "time_interval": 120
}
```

## Usage

### Starting the Service

```bash
# Run with default settings
./dist/db-replicator-linux-amd64 -c config.json

# With custom options
./dist/db-replicator-linux-amd64 \
  -c config.json \
  -p 7012 \
  -l Debug
```

### Command-line Flags

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--config` | `-c` | (required) | Path to configuration JSON file |
| `--port` | `-p` | `7012` | HTTP server port |
| `--log` | `-l` | `Info` | Log level (Debug, Info, Warn, Error) |

### Docker Deployment

```bash
# Using Docker with volume mount
docker run -d \
  -v /path/to/config.json:/app/config.json \
  -p 7012:7012 \
  --name db-replicator \
  docker.pkg.github.com/byuoitav/db-replicator/db-replicator:latest \
  -c /app/config.json
```

## API Endpoints

### Manual Replication Trigger

Trigger an immediate replication cycle without waiting for the scheduled interval.

**Request:**
```
GET /replication/start
```

**Response:**
```json
"replication started"
```

**Example:**
```bash
curl http://localhost:7012/replication/start
```

## Development

### Project Structure

```
.
├── cmd/
│   ├── main.go           # Application entry point
│   ├── http.go           # HTTP server setup
│   ├── deps.go           # Dependency injection
│   └── logger.go         # Logging configuration
├── replication/
│   ├── replicator.go     # Core replication logic
│   ├── config.go         # Configuration parsing
│   ├── couch.go          # CouchDB client operations
│   └── couch_test.go     # CouchDB tests
├── dockerfile            # Docker image definition
├── makefile              # Build automation
├── go.mod                # Go module definition
├── go.sum                # Go dependencies
└── README.md             # This file
```

### Building and Testing

```bash
# Download dependencies
make deps

# Run tests
make test

# Run tests with coverage
make test-cov

# Run linter
make lint

# Build binaries for all platforms
make build

# Build and create Docker images
make docker

# Deploy Docker images to registry
make deploy

# Clean build artifacts
make clean
```

### Adding a New Replication Job

1. Update your configuration JSON with a new entry in the `jobs` array
2. Restart the service with the updated configuration
3. (Optional) Trigger manual replication via the `/replication/start` endpoint

## Troubleshooting

### Connection Issues

**Problem**: "waiting for source database to start"

**Solution**: Verify the source CouchDB instance is running and accessible at the configured address.

```bash
curl -u username:password http://source-address:5984
```

### Authentication Errors

**Problem**: Replication fails with 401 Unauthorized

**Solution**: Verify credentials in the configuration file match those in your CouchDB instance.

### Log Levels

Increase verbosity for debugging:

```bash
./dist/db-replicator-linux-amd64 -c config.json -l Debug
```

## Contributing

Contributions are welcome! Please ensure:
- Code follows Go best practices
- Tests are added for new functionality
- All tests pass: `make test`
- Linting passes: `make lint`

## License

See LICENSE file for details.
