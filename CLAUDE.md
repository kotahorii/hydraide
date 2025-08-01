# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

HydrAIDE is a high-performance, real-time data engine written in Go that provides distributed data storage, locking mechanisms, and pub/sub capabilities. The project consists of a gRPC server, CLI tool (hydraidectl), and multi-language SDKs.

## Core Architecture

### Application Structure
- **app/core/**: Core engine components
  - `hydra/`: Main data engine with swamp/treasure abstraction
  - `filesystem/`: File system operations and disk I/O
  - `compressor/`: Data compression utilities
  - `settings/`: Configuration management
  - `safeops/`: Thread-safe operations
  - `zeus/`: Resource management and cleanup

- **app/server/**: gRPC server implementation
  - `server/`: Core server logic
  - `gateway/`: HTTP gateway for REST API
  - `loghandlers/`: Multi-format logging (Graylog, fallback, slog)
  - `observer/`: System monitoring and observability

- **app/hydraidectl/**: CLI management tool
  - Commands: `init`, `stop`, `restart`, `destroy`, `list`
  - Port validation and server configuration
  - TLS certificate generation and management

### Data Model Concepts
- **Swamp**: Primary data container (analogous to database/collection)
- **Treasure**: Individual data records within swamps
- **Island**: Deterministic location for swamp storage
- **Beacon**: Metadata and indexing for treasures
- **Chronicler**: Data persistence and serialization

## Development Workflow and Branch Strategy

### Branch Strategy
The HydrAIDE project uses feature branches for component-specific development:

- **hydraidectl changes**: Create PRs against `feature/hydraidectl` branch
- **Server/Core changes**: Create PRs against `main` branch
- **General changes**: Create PRs against `main` branch

### Working with hydraidectl
When working on CLI-related features or fixes:

```bash
# Fetch latest changes
git fetch upstream

# Create feature branch from hydraidectl feature branch
git checkout upstream/feature/hydraidectl
git checkout -b fix/your-feature-name

# After implementation, create PR against feature/hydraidectl
# Example: kotahorii:fix/your-feature-name -> hydraide:feature/hydraidectl
```

**Important**: All hydraidectl-related work should target the `feature/hydraidectl` branch to keep related changes consolidated before merging to main.

## Development Commands

### Building and Testing
```bash
# Build Docker image (production-ready)
make build

# Build and push Docker image to GHCR
make build-push

# Generate Go protocol buffers and tidy dependencies
make build-go

# Generate only Go bindings (fast)
make proto-go

# Run all tests
go test ./...

# Run specific package tests with verbose output
go test -v ./app/core/hydra/...

# Run single test with race detection
go test -race -run TestValidatePort ./app/hydraidectl/cmd/

# Run tests with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Build specific binaries
go build -o dist/hydraidectl ./app/hydraidectl/
go build -o dist/hydraide-server ./app/server/
```

### Protocol Buffer Generation

The project uses a sophisticated multi-language proto generation system:

```bash
# Generate for all available languages
make proto-go          # Go (always available)
make proto-python      # Python (via uv and grpc_tools)
make proto-node        # Node.js (requires grpc_tools_node_protoc_plugin)
make proto-rust        # Rust (requires protoc-gen-prost)
make proto-java        # Java (requires protoc-gen-grpc-java)
make proto-csharp      # C#/.NET (requires protoc-gen-grpc-csharp)

# Clean all generated files
make clean
```

Generated files locations:
- Go: `generated/hydraidepbgo/`
- Python: `sdk/python/hydraidepy/src/hydraidepy/generated/`
- Other languages: `generated/hydraidepb{lang}/`

### Docker Operations
```bash
# Build Docker image with specific tag
IMAGE_TAG=v2.1.0 make build

# Build and push to GitHub Container Registry
make build-push

# Local development with Docker
docker build -t hydraide-dev .
docker run -e PUID=1001 -e PGID=1001 -p 4900:4900 hydraide-dev
```

## Key Dependencies and Versions

### Go Version
- **Target**: Go 1.24.1 (as specified in go.mod)

### Core Libraries
- `github.com/spf13/cobra`: CLI framework for hydraidectl
- `google.golang.org/grpc`: gRPC server and client (v1.74.2)
- `google.golang.org/protobuf`: Protocol buffer support (v1.36.6)
- `github.com/google/uuid`: UUID generation for treasures
- `github.com/stretchr/testify`: Testing framework

### Compression and Performance
- `github.com/klauspost/compress`: High-performance compression
- `github.com/golang/snappy`: Fast compression algorithm
- `github.com/pierrec/lz4`: LZ4 compression support
- `github.com/cespare/xxhash/v2`: Fast hashing

### System Integration
- `github.com/shirou/gopsutil`: System metrics and monitoring
- `github.com/joho/godotenv`: Environment variable management

## Testing Strategy

### Test Structure
- All core packages have comprehensive `*_test.go` files
- Integration tests in `app/server/e2etests/`
- Test data in `app/core/compressor/test-data/`
- CLI validation tests with comprehensive edge cases

### Pre-commit Hooks
The project uses comprehensive pre-commit hooks (`.pre-commit-config.yaml`):

**Installation:**
```bash
pre-commit install --hook-type pre-commit --hook-type commit-msg
```

**Hook Categories:**
- **General**: Large file checks, JSON/YAML validation, merge conflicts, private key detection
- **Code Quality**: End-of-file fixing, AST validation, debug statement detection
- **Commit Messages**: Conventional commit format enforcement (`--strict`)
- **Python SDK**: Ruff linting/formatting and MyPy type checking for `sdk/python/hydraidepy/`

## Development Workflow

### Working with Core Engine
1. Understand the Swamp/Treasure abstraction before modifying core logic
2. All data operations go through the Hydra interface (`app/core/hydra/hydra.go`)
3. Use safeops for thread-safe operations (`app/core/safeops/safeops.go`)
4. Filesystem package handles all disk I/O (`app/core/filesystem/filesystem.go`)

### Adding CLI Commands
1. Create command file in `app/hydraidectl/cmd/`
2. Register in `root.go`
3. Follow existing validation patterns:
   - Port validation: `validatePort()` (1-65535 range, strict validation)
   - Log level validation: `validateLoglevel()` (debug/info/warn/error)
   - Message size validation: `parseMessageSize()` (10MB-10GB, human-readable input)
4. Add comprehensive tests with edge cases

### Server Modifications
1. gRPC service definitions in `proto/hydraide.proto`
2. Regenerate bindings with `make proto-go` (or `make build-go` for full update)
3. Server implementation in `app/server/server/server.go`
4. Add structured logging through loghandlers (`app/server/loghandlers/`)

### CLI Tool Validation Examples
The CLI includes robust validation with user-friendly error handling:
- **Port Validation**: Range 1-65535, conflict detection between main and health check ports
- **Message Size Parsing**: Supports raw bytes and human-readable format (100MB, 1.5GB)
- **Input Validation Loops**: Re-prompts on invalid input with clear error messages

## Multi-Language SDK Support

The project maintains SDKs across multiple languages with varying maturity levels:

### Production Ready
- **Go** (`sdk/go/hydraidego/`): Full-featured, production-tested

### In Development/Planning
- **Python** (`sdk/python/hydraidepy/`): Active development with uv package management
- **Node.js, Rust, Java, C#, Swift, Kotlin, C++**: Various stages of planning/development

When modifying the gRPC protocol:
1. Update `proto/hydraide.proto`
2. Run `make proto-go` for Go bindings
3. Run language-specific proto commands for other SDKs
4. Test generated bindings with existing SDK code

## Docker and Deployment

### Docker Configuration
- **Base Image**: Multi-stage build with Go compilation
- **Entry Point**: `entrypoint.sh` handles user/group creation for non-root execution
- **Environment Variables**: `PUID` and `PGID` for user/group mapping
- **Production Examples**: Located in `docs/install-scripts/docker/`

### Configuration Management
- **Environment Loading**: Uses `github.com/joho/godotenv` for .env file support
- **TLS Certificates**: Auto-generation and management via hydraidectl
- **Health Checks**: Separate port configuration with conflict detection
- **gRPC Settings**: Configurable message size limits with validation

## Architecture Insights

### Performance Characteristics
- **Memory Management**: Swamps load on-demand, auto-cleanup when idle
- **Concurrency**: Per-object locking with deadlock-free critical sections  
- **Storage**: Deterministic folder-based distribution for horizontal scaling
- **Compression**: Multiple algorithms (Snappy, LZ4) for space efficiency

### Production Usage
- Currently powers [Trendizz.com](https://trendizz.com) with millions of indexed websites
- Sub-second search across hundreds of millions of words without preloading
- 2+ years in production, replacing Redis, MongoDB, Kafka, and their integration complexity

### Real-time Capabilities
- Native pub/sub subscriptions on all data operations
- Event-driven architecture with built-in reactivity
- No polling required - events flow through structured channels