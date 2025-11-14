# Phase 1 Implementation Plan: MVP Pipeline (Months 1-3)

> **Goal:** Working Spec→Code→Test pipeline for a single project
> **Timeline:** 12 weeks (3 months)
> **Team Size:** 1-2 developers
> **Deliverable:** End-to-end automated pipeline from specification to tested code

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Month 1: Foundation](#month-1-foundation)
4. [Month 2: Code Generation](#month-2-code-generation)
5. [Month 3: Testing & Integration](#month-3-testing--integration)
6. [Testing Strategy](#testing-strategy)
7. [Troubleshooting Guide](#troubleshooting-guide)

---

## Overview

### What We're Building in Phase 1

```
┌─────────────────────────────────────────────────────────────┐
│                     PHASE 1 SCOPE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User Input (text description)                              │
│         ↓                                                   │
│  [Spec-Synthesizer-MCP]                                     │
│         ↓                                                   │
│  Structured Specification                                   │
│         ↓                                                   │
│  [Codesmith-MCP]                                            │
│         ↓                                                   │
│  Generated Code (in Git repo)                               │
│         ↓                                                   │
│  [Sandbox-Runner-MCP]                                       │
│         ↓                                                   │
│  Test Results                                               │
│                                                             │
│  All orchestrated by [Vibes-Director]                       │
│  All logged to [Observatory]                                │
│  All accessed via [Gateway-Hub]                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Services We'll Build

1. **Gateway-Hub** (Basic version) - Auth & routing
2. **Observatory** (OpenObserve deployment) - Logging & metrics
3. **Spec-Synthesizer-MCP** - Generate specs from text
4. **Codesmith-MCP** - Generate code from specs
5. **Sandbox-Runner-MCP** - Execute tests in containers
6. **Vibes-Director** (Enhanced) - Workflow orchestration

### What's NOT in Phase 1

- ❌ Control-Hub-Web (web UI) - Phase 2
- ❌ RAG-Engine-MCP (vector DB) - Phase 3
- ❌ Research-Harvester-MCP - Phase 3
- ❌ DevOps-Conductor-MCP (deployment) - Phase 2
- ❌ Full RBAC and quotas - Phase 2
- ❌ Advanced workflows - Phase 3

---

## Prerequisites

### Hardware Requirements
- **Development Machine:**
  - 16GB RAM minimum (32GB recommended)
  - 50GB free disk space
  - CPU: 4+ cores
  - OS: Linux (Ubuntu 22.04+), macOS (Ventura+), or Windows 11 with WSL2

### Software Requirements
- **Docker & Docker Compose:** v24.0+
- **Node.js:** v20.x (LTS)
- **Python:** 3.12+
- **Git:** 2.40+
- **Code Editor:** VS Code (recommended) or similar

### Accounts & API Keys
- **Claude API Key:** Anthropic account with Sonnet 4.5 access
- **GitHub/GitLab Account:** For code repositories
- **GitHub Personal Access Token:** For API operations (repo, PR creation)

### Knowledge Prerequisites
- TypeScript/JavaScript (intermediate)
- Python (intermediate)
- Docker basics
- Git workflows
- REST APIs and MCP protocol basics

---

## Month 1: Foundation

### Week 1-2: Infrastructure Setup

#### Task 1.1: Create Project Structure

**Action:** Set up the monorepo structure for all services.

```bash
# Create main directory structure
mkdir -p ecosystem/{services,infrastructure,shared}
cd ecosystem

# Create service directories
mkdir -p services/{gateway-hub,spec-synthesizer-mcp,codesmith-mcp,sandbox-runner-mcp}
mkdir -p infrastructure/{docker,observability}
mkdir -p shared/{types,utils}

# Initialize Git
git init
echo "node_modules/\n*.log\n.env\n*.pyc\n__pycache__/\n.DS_Store" > .gitignore
```

**File Structure:**
```
ecosystem/
├── services/
│   ├── gateway-hub/              # API Gateway (Go)
│   ├── spec-synthesizer-mcp/     # Spec generation (Python)
│   ├── codesmith-mcp/            # Code generation (TypeScript)
│   └── sandbox-runner-mcp/       # Test execution (Go)
├── infrastructure/
│   ├── docker/
│   │   ├── docker-compose.yml    # All services
│   │   ├── docker-compose.dev.yml # Development overrides
│   │   └── .env.example          # Environment template
│   └── observability/
│       └── openobserve/          # OpenObserve config
├── shared/
│   ├── types/                    # Shared TypeScript types
│   └── utils/                    # Shared utilities
├── docs/
│   ├── ECOSYSTEM_ARCHITECTURE.md
│   └── PHASE_1_IMPLEMENTATION_PLAN.md
└── README.md
```

---

#### Task 1.2: Docker Compose Configuration

**Action:** Create Docker Compose setup for all infrastructure services.

**File:** `infrastructure/docker/docker-compose.yml`

```yaml
version: '3.9'

services:
  # PostgreSQL - Main database
  postgres:
    image: postgres:16-alpine
    container_name: ecosystem-postgres
    environment:
      POSTGRES_DB: ecosystem
      POSTGRES_USER: ecosystem_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-dev_password}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ecosystem_user -d ecosystem"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - ecosystem-network

  # Redis - Caching and job queues
  redis:
    image: redis:7-alpine
    container_name: ecosystem-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - ecosystem-network

  # OpenObserve - Observability platform
  openobserve:
    image: public.ecr.aws/zinclabs/openobserve:latest
    container_name: ecosystem-openobserve
    environment:
      ZO_ROOT_USER_EMAIL: ${OPENOBSERVE_EMAIL:-admin@ecosystem.local}
      ZO_ROOT_USER_PASSWORD: ${OPENOBSERVE_PASSWORD:-admin_password}
      ZO_DATA_DIR: /data
    ports:
      - "5080:5080"
    volumes:
      - openobserve_data:/data
    networks:
      - ecosystem-network

  # Gateway Hub - API Gateway
  gateway-hub:
    build:
      context: ../../services/gateway-hub
      dockerfile: Dockerfile
    container_name: ecosystem-gateway-hub
    environment:
      PORT: 8080
      POSTGRES_URL: postgres://ecosystem_user:${POSTGRES_PASSWORD:-dev_password}@postgres:5432/ecosystem
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET:-dev_jwt_secret_change_in_production}
      OTEL_ENDPOINT: http://openobserve:5080
      LOG_LEVEL: ${LOG_LEVEL:-info}
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      openobserve:
        condition: service_started
    networks:
      - ecosystem-network

  # Spec Synthesizer MCP
  spec-synthesizer-mcp:
    build:
      context: ../../services/spec-synthesizer-mcp
      dockerfile: Dockerfile
    container_name: ecosystem-spec-synthesizer
    environment:
      PORT: 3001
      POSTGRES_URL: postgres://ecosystem_user:${POSTGRES_PASSWORD:-dev_password}@postgres:5432/ecosystem
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      GATEWAY_URL: http://gateway-hub:8080
      OTEL_ENDPOINT: http://openobserve:5080
      LOG_LEVEL: ${LOG_LEVEL:-info}
    ports:
      - "3001:3001"
    volumes:
      - ../../services/spec-synthesizer-mcp:/app
      - spec_repos:/repos  # Git repos for specs
    depends_on:
      postgres:
        condition: service_healthy
      gateway-hub:
        condition: service_started
    networks:
      - ecosystem-network

  # Codesmith MCP
  codesmith-mcp:
    build:
      context: ../../services/codesmith-mcp
      dockerfile: Dockerfile
    container_name: ecosystem-codesmith
    environment:
      PORT: 3002
      POSTGRES_URL: postgres://ecosystem_user:${POSTGRES_PASSWORD:-dev_password}@postgres:5432/ecosystem
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      GITHUB_TOKEN: ${GITHUB_TOKEN}
      GATEWAY_URL: http://gateway-hub:8080
      OTEL_ENDPOINT: http://openobserve:5080
      LOG_LEVEL: ${LOG_LEVEL:-info}
    ports:
      - "3002:3002"
    volumes:
      - ../../services/codesmith-mcp:/app
      - code_repos:/repos  # Git repos for code
      - /var/run/docker.sock:/var/run/docker.sock  # For running linters
    depends_on:
      postgres:
        condition: service_healthy
      gateway-hub:
        condition: service_started
    networks:
      - ecosystem-network

  # Sandbox Runner MCP
  sandbox-runner-mcp:
    build:
      context: ../../services/sandbox-runner-mcp
      dockerfile: Dockerfile
    container_name: ecosystem-sandbox-runner
    privileged: true  # Needed for Docker-in-Docker
    environment:
      PORT: 3003
      GATEWAY_URL: http://gateway-hub:8080
      OTEL_ENDPOINT: http://openobserve:5080
      LOG_LEVEL: ${LOG_LEVEL:-info}
    ports:
      - "3003:3003"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Docker socket
      - sandbox_cache:/cache  # Image cache
    depends_on:
      gateway-hub:
        condition: service_started
    networks:
      - ecosystem-network

volumes:
  postgres_data:
  redis_data:
  openobserve_data:
  spec_repos:
  code_repos:
  sandbox_cache:

networks:
  ecosystem-network:
    driver: bridge
```

**File:** `infrastructure/docker/.env.example`

```bash
# Database
POSTGRES_PASSWORD=secure_password_here

# Authentication
JWT_SECRET=generate_a_secure_random_string_here

# OpenObserve
OPENOBSERVE_EMAIL=admin@yourdomain.com
OPENOBSERVE_PASSWORD=secure_admin_password

# AI Models
ANTHROPIC_API_KEY=sk-ant-your-key-here

# GitHub
GITHUB_TOKEN=ghp_your_token_here

# Logging
LOG_LEVEL=info
```

**Action:** Copy and configure environment:
```bash
cd infrastructure/docker
cp .env.example .env
# Edit .env with your actual values
```

---

#### Task 1.3: Database Schema

**Action:** Create initial database schema for all services.

**File:** `infrastructure/docker/init-scripts/01_init_schema.sql`

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Users table (for Gateway-Hub authentication)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'developer', -- admin, developer, viewer
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- API Keys table
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    key_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    permissions JSONB DEFAULT '[]'::jsonb,
    last_used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);

-- MCP Services registry
CREATE TABLE mcp_services (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) UNIQUE NOT NULL,
    url VARCHAR(512) NOT NULL,
    version VARCHAR(50),
    status VARCHAR(50) DEFAULT 'healthy', -- healthy, unhealthy, unknown
    last_health_check TIMESTAMP,
    metadata JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Specifications table (for Spec-Synthesizer-MCP)
CREATE TABLE specifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    spec_type VARCHAR(100) NOT NULL, -- user_story, technical_design, api_contract, etc.
    content JSONB NOT NULL,
    git_repo_url VARCHAR(512),
    git_branch VARCHAR(255),
    git_commit_sha VARCHAR(40),
    status VARCHAR(50) DEFAULT 'draft', -- draft, validated, approved, implemented
    validation_result JSONB,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Code generation jobs table (for Codesmith-MCP)
CREATE TABLE code_generation_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    spec_id UUID REFERENCES specifications(id) ON DELETE CASCADE,
    git_repo_url VARCHAR(512) NOT NULL,
    git_branch VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending', -- pending, running, completed, failed
    progress INTEGER DEFAULT 0, -- 0-100
    result JSONB,
    error_message TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Test execution jobs table (for Sandbox-Runner-MCP)
CREATE TABLE test_execution_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code_job_id UUID REFERENCES code_generation_jobs(id) ON DELETE CASCADE,
    git_repo_url VARCHAR(512) NOT NULL,
    git_commit_sha VARCHAR(40) NOT NULL,
    test_framework VARCHAR(100), -- jest, pytest, go-test, etc.
    status VARCHAR(50) DEFAULT 'pending',
    container_id VARCHAR(255),
    test_results JSONB,
    coverage_report JSONB,
    logs TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Workflows table (for Vibes-Director)
CREATE TABLE workflows (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    definition JSONB NOT NULL, -- Workflow steps and configuration
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Workflow executions table
CREATE TABLE workflow_executions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    workflow_id UUID REFERENCES workflows(id) ON DELETE CASCADE,
    status VARCHAR(50) DEFAULT 'pending', -- pending, running, completed, failed, cancelled
    current_step INTEGER DEFAULT 0,
    context JSONB DEFAULT '{}'::jsonb, -- Execution context and variables
    result JSONB,
    error_message TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Workflow execution steps (for tracking individual step execution)
CREATE TABLE workflow_execution_steps (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    execution_id UUID REFERENCES workflow_executions(id) ON DELETE CASCADE,
    step_index INTEGER NOT NULL,
    step_name VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    input JSONB,
    output JSONB,
    error_message TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- AI model usage tracking (for cost management)
CREATE TABLE ai_model_usage (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id),
    service_name VARCHAR(255) NOT NULL,
    model_name VARCHAR(255) NOT NULL,
    operation VARCHAR(255),
    input_tokens INTEGER,
    output_tokens INTEGER,
    cost_usd DECIMAL(10, 6),
    request_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_specifications_project_id ON specifications(project_id);
CREATE INDEX idx_specifications_status ON specifications(status);
CREATE INDEX idx_code_jobs_spec_id ON code_generation_jobs(spec_id);
CREATE INDEX idx_code_jobs_status ON code_generation_jobs(status);
CREATE INDEX idx_test_jobs_code_job_id ON test_execution_jobs(code_job_id);
CREATE INDEX idx_test_jobs_status ON test_execution_jobs(status);
CREATE INDEX idx_workflow_executions_workflow_id ON workflow_executions(workflow_id);
CREATE INDEX idx_workflow_executions_status ON workflow_executions(status);
CREATE INDEX idx_workflow_execution_steps_execution_id ON workflow_execution_steps(execution_id);
CREATE INDEX idx_ai_usage_user_id ON ai_model_usage(user_id);
CREATE INDEX idx_ai_usage_created_at ON ai_model_usage(created_at);

-- Create default admin user (password: "admin123" - CHANGE IN PRODUCTION!)
INSERT INTO users (email, password_hash, role)
VALUES (
    'admin@ecosystem.local',
    crypt('admin123', gen_salt('bf')),
    'admin'
);
```

**Action:** This script will run automatically when PostgreSQL container starts for the first time.

---

#### Task 1.4: Gateway-Hub Implementation

**Action:** Build the API Gateway service in Go.

**File:** `services/gateway-hub/go.mod`

```go
module github.com/your-org/ecosystem/gateway-hub

go 1.22

require (
    github.com/golang-jwt/jwt/v5 v5.2.0
    github.com/labstack/echo/v4 v4.11.4
    github.com/lib/pq v1.10.9
    github.com/redis/go-redis/v9 v9.3.1
    go.opentelemetry.io/otel v1.21.0
    go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp v1.21.0
)
```

**File:** `services/gateway-hub/main.go`

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

type Config struct {
    Port         string
    PostgresURL  string
    RedisURL     string
    JWTSecret    string
    OTELEndpoint string
    LogLevel     string
}

func loadConfig() *Config {
    return &Config{
        Port:         getEnv("PORT", "8080"),
        PostgresURL:  getEnv("POSTGRES_URL", ""),
        RedisURL:     getEnv("REDIS_URL", ""),
        JWTSecret:    getEnv("JWT_SECRET", ""),
        OTELEndpoint: getEnv("OTEL_ENDPOINT", ""),
        LogLevel:     getEnv("LOG_LEVEL", "info"),
    }
}

func getEnv(key, fallback string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return fallback
}

// JWT Claims structure
type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type LoginResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    ExpiresIn    int    `json:"expires_in"`
}

func main() {
    config := loadConfig()

    // Initialize Echo
    e := echo.New()
    e.HideBanner = true

    // Middleware
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())
    e.Use(middleware.CORS())

    // Initialize database connection
    // db := initDatabase(config.PostgresURL)
    // defer db.Close()

    // Routes
    e.GET("/health", healthCheck)
    e.POST("/auth/login", login(config))
    e.POST("/auth/refresh", refreshToken(config))

    // MCP service routes (authenticated)
    mcp := e.Group("/mcp")
    mcp.Use(jwtMiddleware(config.JWTSecret))
    mcp.POST("/:service/:tool", proxyToMCPService)

    // Service registry routes (admin only)
    admin := e.Group("/admin")
    admin.Use(jwtMiddleware(config.JWTSecret))
    admin.Use(roleMiddleware("admin"))
    admin.GET("/services", listServices)
    admin.POST("/services", registerService)

    // Start server
    log.Printf("Gateway Hub starting on port %s", config.Port)
    if err := e.Start(":" + config.Port); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}

func healthCheck(c echo.Context) error {
    return c.JSON(http.StatusOK, map[string]string{
        "status":    "healthy",
        "service":   "gateway-hub",
        "timestamp": time.Now().Format(time.RFC3339),
    })
}

func login(config *Config) echo.HandlerFunc {
    return func(c echo.Context) error {
        var req LoginRequest
        if err := c.Bind(&req); err != nil {
            return c.JSON(http.StatusBadRequest, map[string]string{
                "error": "Invalid request",
            })
        }

        // TODO: Verify credentials against database
        // For now, hardcoded for development
        if req.Email != "admin@ecosystem.local" || req.Password != "admin123" {
            return c.JSON(http.StatusUnauthorized, map[string]string{
                "error": "Invalid credentials",
            })
        }

        // Generate access token
        accessToken, err := generateToken(config.JWTSecret, "user-123", req.Email, "admin", 1*time.Hour)
        if err != nil {
            return c.JSON(http.StatusInternalServerError, map[string]string{
                "error": "Failed to generate token",
            })
        }

        // Generate refresh token
        refreshToken, err := generateToken(config.JWTSecret, "user-123", req.Email, "admin", 7*24*time.Hour)
        if err != nil {
            return c.JSON(http.StatusInternalServerError, map[string]string{
                "error": "Failed to generate refresh token",
            })
        }

        return c.JSON(http.StatusOK, LoginResponse{
            AccessToken:  accessToken,
            RefreshToken: refreshToken,
            ExpiresIn:    3600, // 1 hour
        })
    }
}

func refreshToken(config *Config) echo.HandlerFunc {
    return func(c echo.Context) error {
        // TODO: Implement refresh token logic
        return c.JSON(http.StatusNotImplemented, map[string]string{
            "error": "Not implemented yet",
        })
    }
}

func generateToken(secret, userID, email, role string, duration time.Duration) (string, error) {
    claims := &Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(duration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}

func jwtMiddleware(secret string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            tokenString := c.Request().Header.Get("Authorization")
            if tokenString == "" {
                return c.JSON(http.StatusUnauthorized, map[string]string{
                    "error": "Missing authorization token",
                })
            }

            // Remove "Bearer " prefix
            if len(tokenString) > 7 && tokenString[:7] == "Bearer " {
                tokenString = tokenString[7:]
            }

            token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
                return []byte(secret), nil
            })

            if err != nil || !token.Valid {
                return c.JSON(http.StatusUnauthorized, map[string]string{
                    "error": "Invalid token",
                })
            }

            claims, ok := token.Claims.(*Claims)
            if !ok {
                return c.JSON(http.StatusUnauthorized, map[string]string{
                    "error": "Invalid token claims",
                })
            }

            // Store user info in context
            c.Set("user_id", claims.UserID)
            c.Set("email", claims.Email)
            c.Set("role", claims.Role)

            return next(c)
        }
    }
}

func roleMiddleware(requiredRole string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            role := c.Get("role").(string)
            if role != requiredRole {
                return c.JSON(http.StatusForbidden, map[string]string{
                    "error": fmt.Sprintf("Requires %s role", requiredRole),
                })
            }
            return next(c)
        }
    }
}

func proxyToMCPService(c echo.Context) error {
    service := c.Param("service")
    tool := c.Param("tool")

    // TODO: Implement MCP service proxy logic
    // 1. Look up service in registry
    // 2. Forward request to service
    // 3. Return response

    return c.JSON(http.StatusOK, map[string]string{
        "message": fmt.Sprintf("Proxying to %s/%s (not implemented yet)", service, tool),
    })
}

func listServices(c echo.Context) error {
    // TODO: Query database for registered services
    return c.JSON(http.StatusOK, []map[string]string{
        {"name": "spec-synthesizer-mcp", "url": "http://spec-synthesizer-mcp:3001", "status": "healthy"},
        {"name": "codesmith-mcp", "url": "http://codesmith-mcp:3002", "status": "healthy"},
        {"name": "sandbox-runner-mcp", "url": "http://sandbox-runner-mcp:3003", "status": "healthy"},
    })
}

func registerService(c echo.Context) error {
    // TODO: Implement service registration
    return c.JSON(http.StatusNotImplemented, map[string]string{
        "error": "Not implemented yet",
    })
}
```

**File:** `services/gateway-hub/Dockerfile`

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build
RUN CGO_ENABLED=0 GOOS=linux go build -o gateway-hub .

# Final stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/gateway-hub .

EXPOSE 8080

CMD ["./gateway-hub"]
```

---

#### Task 1.5: Spec-Synthesizer-MCP Implementation

**Action:** Build the specification generation service in Python.

**File:** `services/spec-synthesizer-mcp/pyproject.toml`

```toml
[tool.poetry]
name = "spec-synthesizer-mcp"
version = "0.1.0"
description = "MCP server for generating specifications from requirements"
authors = ["Your Team"]

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.109.0"
uvicorn = "^0.27.0"
anthropic = "^0.18.0"
pydantic = "^2.6.0"
pydantic-settings = "^2.1.0"
psycopg2-binary = "^2.9.9"
gitpython = "^3.1.41"
jinja2 = "^3.1.3"
pyyaml = "^6.0.1"
opentelemetry-api = "^1.22.0"
opentelemetry-sdk = "^1.22.0"
opentelemetry-instrumentation-fastapi = "^0.43b0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

**File:** `services/spec-synthesizer-mcp/main.py`

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Dict, Any, List, Optional
import anthropic
import os
import logging
from datetime import datetime

# Configure logging
logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO").upper())
logger = logging.getLogger(__name__)

app = FastAPI(title="Spec Synthesizer MCP")

# Configuration
ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")
if not ANTHROPIC_API_KEY:
    raise ValueError("ANTHROPIC_API_KEY environment variable is required")

client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)


# Models
class GenerateUserStoriesRequest(BaseModel):
    description: str = Field(..., description="Project or feature description")
    project_context: Optional[str] = Field(None, description="Additional project context")
    num_stories: int = Field(5, ge=1, le=20, description="Number of user stories to generate")


class UserStory(BaseModel):
    title: str
    description: str
    acceptance_criteria: List[str]
    priority: str  # high, medium, low
    estimated_effort: str  # small, medium, large


class GenerateUserStoriesResponse(BaseModel):
    stories: List[UserStory]
    metadata: Dict[str, Any]


class GenerateTechnicalDesignRequest(BaseModel):
    user_stories: List[Dict[str, Any]]
    architecture_preferences: Optional[Dict[str, Any]] = None


class TechnicalDesign(BaseModel):
    overview: str
    architecture: Dict[str, Any]
    components: List[Dict[str, Any]]
    data_models: List[Dict[str, Any]]
    api_endpoints: List[Dict[str, Any]]
    dependencies: List[str]


class GenerateTechnicalDesignResponse(BaseModel):
    design: TechnicalDesign
    metadata: Dict[str, Any]


class GenerateAPIContractRequest(BaseModel):
    design_doc: str
    api_style: str = Field("rest", pattern="^(rest|graphql)$")


class GenerateAPIContractResponse(BaseModel):
    contract: str  # OpenAPI YAML or GraphQL schema
    format: str
    metadata: Dict[str, Any]


# MCP Server Info
@app.get("/")
async def root():
    return {
        "name": "spec-synthesizer-mcp",
        "version": "0.1.0",
        "protocol": "mcp",
        "tools": [
            {
                "name": "generate_user_stories",
                "description": "Generate user stories from a project description",
                "input_schema": GenerateUserStoriesRequest.model_json_schema(),
            },
            {
                "name": "generate_technical_design",
                "description": "Generate technical design document from user stories",
                "input_schema": GenerateTechnicalDesignRequest.model_json_schema(),
            },
            {
                "name": "generate_api_contract",
                "description": "Generate API contract (OpenAPI/GraphQL) from design doc",
                "input_schema": GenerateAPIContractRequest.model_json_schema(),
            },
        ],
    }


@app.get("/health")
async def health():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}


# Tool implementations
@app.post("/tools/generate_user_stories", response_model=GenerateUserStoriesResponse)
async def generate_user_stories(request: GenerateUserStoriesRequest):
    """Generate user stories from a project description using Claude."""

    logger.info(f"Generating {request.num_stories} user stories")

    prompt = f"""You are a product manager creating user stories for a software project.

Project Description:
{request.description}

{f"Additional Context:\n{request.project_context}" if request.project_context else ""}

Generate {request.num_stories} user stories in the following JSON format:
[
  {{
    "title": "Brief title for the user story",
    "description": "As a [user type], I want [goal] so that [benefit]",
    "acceptance_criteria": ["criterion 1", "criterion 2", "criterion 3"],
    "priority": "high|medium|low",
    "estimated_effort": "small|medium|large"
  }}
]

Guidelines:
- Use "As a..., I want..., so that..." format
- Each story should be independent and testable
- Acceptance criteria should be specific and measurable
- Prioritize based on business value and dependencies
- Estimate effort realistically

Return ONLY the JSON array, no additional text."""

    try:
        message = client.messages.create(
            model="claude-sonnet-4-5-20250929",
            max_tokens=4000,
            messages=[{"role": "user", "content": prompt}]
        )

        # Extract JSON from response
        content = message.content[0].text

        # Parse user stories (simplified - should add better error handling)
        import json
        stories_data = json.loads(content)

        stories = [UserStory(**story) for story in stories_data]

        return GenerateUserStoriesResponse(
            stories=stories,
            metadata={
                "generated_at": datetime.utcnow().isoformat(),
                "model": "claude-sonnet-4-5",
                "tokens_used": message.usage.input_tokens + message.usage.output_tokens,
            }
        )

    except Exception as e:
        logger.error(f"Error generating user stories: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/generate_technical_design", response_model=GenerateTechnicalDesignResponse)
async def generate_technical_design(request: GenerateTechnicalDesignRequest):
    """Generate technical design document from user stories."""

    logger.info("Generating technical design document")

    # Convert user stories to text
    stories_text = "\n\n".join([
        f"Story {i+1}: {story.get('title', 'Untitled')}\n{story.get('description', '')}"
        for i, story in enumerate(request.user_stories)
    ])

    prompt = f"""You are a software architect designing a system based on user stories.

User Stories:
{stories_text}

{f"Architecture Preferences:\n{request.architecture_preferences}" if request.architecture_preferences else ""}

Create a comprehensive technical design document in the following JSON format:
{{
  "overview": "High-level system overview (2-3 paragraphs)",
  "architecture": {{
    "pattern": "e.g., microservices, monolithic, event-driven",
    "layers": ["presentation", "business logic", "data access"],
    "key_decisions": ["decision 1", "decision 2"]
  }},
  "components": [
    {{
      "name": "Component name",
      "responsibility": "What it does",
      "dependencies": ["other components"]
    }}
  ],
  "data_models": [
    {{
      "entity": "Entity name",
      "attributes": ["attr1: type", "attr2: type"],
      "relationships": ["relationship description"]
    }}
  ],
  "api_endpoints": [
    {{
      "method": "GET|POST|PUT|DELETE",
      "path": "/api/endpoint",
      "description": "What it does",
      "auth_required": true|false
    }}
  ],
  "dependencies": ["library1", "library2"]
}}

Return ONLY the JSON object, no additional text."""

    try:
        message = client.messages.create(
            model="claude-sonnet-4-5-20250929",
            max_tokens=8000,
            messages=[{"role": "user", "content": prompt}]
        )

        content = message.content[0].text
        import json
        design_data = json.loads(content)

        design = TechnicalDesign(**design_data)

        return GenerateTechnicalDesignResponse(
            design=design,
            metadata={
                "generated_at": datetime.utcnow().isoformat(),
                "model": "claude-sonnet-4-5",
                "tokens_used": message.usage.input_tokens + message.usage.output_tokens,
            }
        )

    except Exception as e:
        logger.error(f"Error generating technical design: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/tools/generate_api_contract", response_model=GenerateAPIContractResponse)
async def generate_api_contract(request: GenerateAPIContractRequest):
    """Generate API contract (OpenAPI or GraphQL schema)."""

    logger.info(f"Generating {request.api_style} API contract")

    if request.api_style == "rest":
        format_instructions = """Generate an OpenAPI 3.1 specification in YAML format with:
- info section (title, version, description)
- servers section
- paths section with all endpoints
- components/schemas section with data models
- security schemes if authentication is needed"""
    else:
        format_instructions = """Generate a GraphQL schema with:
- Type definitions for all entities
- Query operations
- Mutation operations
- Input types where needed"""

    prompt = f"""You are an API designer creating a {request.api_style.upper()} API contract.

Technical Design:
{request.design_doc}

{format_instructions}

Return ONLY the {request.api_style.upper()} specification, no additional text or markdown formatting."""

    try:
        message = client.messages.create(
            model="claude-sonnet-4-5-20250929",
            max_tokens=8000,
            messages=[{"role": "user", "content": prompt}]
        )

        contract = message.content[0].text

        return GenerateAPIContractResponse(
            contract=contract,
            format=request.api_style,
            metadata={
                "generated_at": datetime.utcnow().isoformat(),
                "model": "claude-sonnet-4-5",
                "tokens_used": message.usage.input_tokens + message.usage.output_tokens,
            }
        )

    except Exception as e:
        logger.error(f"Error generating API contract: {e}")
        raise HTTPException(status_code=500, detail=str(e))


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=int(os.getenv("PORT", "3001")))
```

**File:** `services/spec-synthesizer-mcp/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Poetry
RUN pip install poetry

# Copy dependency files
COPY pyproject.toml ./

# Install dependencies
RUN poetry config virtualenvs.create false && \
    poetry install --no-dev --no-root

# Copy application code
COPY . .

EXPOSE 3001

CMD ["python", "main.py"]
```

---

### Week 3-4: Testing Infrastructure Setup

#### Task 1.6: Start Services and Verify

**Action:** Start all services and verify they're running.

```bash
cd infrastructure/docker

# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f gateway-hub
docker-compose logs -f spec-synthesizer-mcp

# Test Gateway health
curl http://localhost:8080/health

# Test Spec Synthesizer health
curl http://localhost:3001/health

# Test authentication
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@ecosystem.local", "password": "admin123"}'

# Save the access token from the response
export ACCESS_TOKEN="<token_from_response>"

# Test MCP tool (should work after implementing proxy)
curl -X POST http://localhost:8080/mcp/spec-synthesizer-mcp/generate_user_stories \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Build a todo list application with user authentication",
    "num_stories": 3
  }'
```

**Expected Results:**
- All containers running
- Health checks passing
- Authentication working
- OpenObserve accessible at http://localhost:5080

---

## Month 2: Code Generation

### Week 1-2: Codesmith-MCP (Part 1)

#### Task 2.1: Project Scaffold Generator

**Action:** Implement the code generation service.

**File:** `services/codesmith-mcp/package.json`

```json
{
  "name": "codesmith-mcp",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.18.0",
    "express": "^4.18.2",
    "simple-git": "^3.22.0",
    "@octokit/rest": "^20.0.2",
    "zod": "^3.22.4",
    "dotenv": "^16.4.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.0",
    "tsx": "^4.7.0",
    "typescript": "^5.3.3"
  }
}
```

**File:** `services/codesmith-mcp/src/index.ts`

```typescript
import express from 'express';
import { Anthropic } from '@anthropic-ai/sdk';
import simpleGit from 'simple-git';
import { Octokit } from '@octokit/rest';
import { z } from 'zod';
import path from 'path';
import fs from 'fs/promises';

const app = express();
app.use(express.json());

// Configuration
const PORT = process.env.PORT || 3002;
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const GITHUB_TOKEN = process.env.GITHUB_TOKEN;
const REPOS_DIR = process.env.REPOS_DIR || '/repos';

if (!ANTHROPIC_API_KEY) {
  throw new Error('ANTHROPIC_API_KEY is required');
}

const anthropic = new Anthropic({ apiKey: ANTHROPIC_API_KEY });
const octokit = GITHUB_TOKEN ? new Octokit({ auth: GITHUB_TOKEN }) : null;

// Schemas
const GenerateScaffoldSchema = z.object({
  spec: z.object({
    projectName: z.string(),
    description: z.string(),
    techStack: z.object({
      language: z.enum(['typescript', 'python', 'go']),
      framework: z.string().optional(),
      database: z.string().optional(),
    }),
  }),
  gitRepoUrl: z.string().optional(),
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// MCP server info
app.get('/', (req, res) => {
  res.json({
    name: 'codesmith-mcp',
    version: '0.1.0',
    protocol: 'mcp',
    tools: [
      {
        name: 'generate_project_scaffold',
        description: 'Generate a complete project scaffold from a specification',
        input_schema: GenerateScaffoldSchema,
      },
    ],
  });
});

// Generate project scaffold
app.post('/tools/generate_project_scaffold', async (req, res) => {
  try {
    const input = GenerateScaffoldSchema.parse(req.body);
    const { spec } = input;

    console.log(`Generating scaffold for: ${spec.projectName}`);

    // Create project directory
    const projectPath = path.join(REPOS_DIR, spec.projectName);
    await fs.mkdir(projectPath, { recursive: true });

    // Generate files based on tech stack
    const files = await generateProjectFiles(spec);

    // Write files to disk
    for (const [filePath, content] of Object.entries(files)) {
      const fullPath = path.join(projectPath, filePath);
      await fs.mkdir(path.dirname(fullPath), { recursive: true });
      await fs.writeFile(fullPath, content, 'utf-8');
    }

    // Initialize git repo
    const git = simpleGit(projectPath);
    await git.init();
    await git.add('.');
    await git.commit('Initial commit: Project scaffold');

    // Create branch
    const branchName = `feature/initial-scaffold`;
    await git.checkoutLocalBranch(branchName);

    res.json({
      success: true,
      projectPath,
      branch: branchName,
      filesGenerated: Object.keys(files).length,
      files: Object.keys(files),
    });
  } catch (error: any) {
    console.error('Error generating scaffold:', error);
    res.status(500).json({ error: error.message });
  }
});

async function generateProjectFiles(spec: any): Promise<Record<string, string>> {
  const files: Record<string, string> = {};

  if (spec.techStack.language === 'typescript') {
    // Generate TypeScript/Node.js project
    files['package.json'] = JSON.stringify(
      {
        name: spec.projectName,
        version: '0.1.0',
        description: spec.description,
        type: 'module',
        scripts: {
          dev: 'tsx watch src/index.ts',
          build: 'tsc',
          start: 'node dist/index.js',
          test: 'jest',
        },
        dependencies: {
          express: '^4.18.2',
          dotenv: '^16.4.0',
        },
        devDependencies: {
          '@types/express': '^4.17.21',
          '@types/node': '^20.11.0',
          typescript: '^5.3.3',
          tsx: '^4.7.0',
          jest: '^29.7.0',
        },
      },
      null,
      2
    );

    files['tsconfig.json'] = JSON.stringify(
      {
        compilerOptions: {
          target: 'ES2022',
          module: 'ES2022',
          moduleResolution: 'node',
          outDir: './dist',
          rootDir: './src',
          strict: true,
          esModuleInterop: true,
          skipLibCheck: true,
        },
        include: ['src/**/*'],
        exclude: ['node_modules'],
      },
      null,
      2
    );

    files['src/index.ts'] = `import express from 'express';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(PORT, () => {
  console.log(\`Server running on port \${PORT}\`);
});
`;

    files['.gitignore'] = `node_modules/
dist/
.env
*.log
.DS_Store
`;

    files['README.md'] = `# ${spec.projectName}

${spec.description}

## Getting Started

\`\`\`bash
npm install
npm run dev
\`\`\`

## Build

\`\`\`bash
npm run build
npm start
\`\`\`
`;
  } else if (spec.techStack.language === 'python') {
    // Generate Python/FastAPI project
    files['pyproject.toml'] = `[tool.poetry]
name = "${spec.projectName}"
version = "0.1.0"
description = "${spec.description}"

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.109.0"
uvicorn = "^0.27.0"

[tool.poetry.dev-dependencies]
pytest = "^7.4.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
`;

    files['main.py'] = `from fastapi import FastAPI
import uvicorn

app = FastAPI(title="${spec.projectName}")

@app.get("/health")
async def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
`;

    files['.gitignore'] = `__pycache__/
*.py[cod]
*$py.class
.env
*.log
.DS_Store
`;

    files['README.md'] = `# ${spec.projectName}

${spec.description}

## Getting Started

\`\`\`bash
poetry install
poetry run python main.py
\`\`\`
`;
  }

  return files;
}

app.listen(PORT, () => {
  console.log(`Codesmith MCP server running on port ${PORT}`);
});
```

**File:** `services/codesmith-mcp/Dockerfile`

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Install git
RUN apk add --no-cache git

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

EXPOSE 3002

CMD ["npm", "start"]
```

---

### Week 3-4: Codesmith-MCP (Part 2)

**This section would continue with:**
- API route generation from OpenAPI specs
- Database model generation
- Test file generation
- Linting and formatting integration

*Due to length constraints, I'll summarize the remaining months...*

---

## Month 3: Testing & Integration

### Week 1-2: Sandbox-Runner-MCP

**Key components:**
- Docker SDK integration (Go)
- Container lifecycle management
- Test framework detection and execution
- Result parsing (JUnit XML, TAP)
- Log streaming via WebSocket

### Week 3-4: Vibes-Director Integration

**Key components:**
- Workflow definition format (YAML)
- Step execution engine
- Event system for progress tracking
- Error handling and retries
- Integration tests for full pipeline

---

## Testing Strategy

### Unit Tests
- Each service has its own test suite
- Target: 80%+ code coverage
- Run in CI on every commit

### Integration Tests
- Test service-to-service communication
- Test MCP protocol compliance
- Test authentication/authorization flows

### End-to-End Tests
- Full pipeline test: description → spec → code → tests
- Success criteria: Generated code passes tests
- Performance benchmarks: < 5 minutes for simple project

---

## Troubleshooting Guide

### Common Issues

**1. Services won't start**
```bash
# Check logs
docker-compose logs <service-name>

# Restart specific service
docker-compose restart <service-name>

# Rebuild if code changed
docker-compose up -d --build <service-name>
```

**2. Authentication failures**
```bash
# Verify JWT_SECRET is set
echo $JWT_SECRET

# Generate new token
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@ecosystem.local", "password": "admin123"}'
```

**3. Database connection errors**
```bash
# Check PostgreSQL is running
docker-compose ps postgres

# Connect to database
docker-compose exec postgres psql -U ecosystem_user -d ecosystem

# Check tables
\dt
```

**4. Claude API errors**
```bash
# Verify API key
echo $ANTHROPIC_API_KEY

# Test API key manually
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

---

## Success Criteria for Phase 1

✅ **Infrastructure:**
- All services running in Docker Compose
- PostgreSQL and Redis operational
- OpenObserve collecting logs

✅ **Spec-Synthesizer-MCP:**
- Can generate user stories from text description
- Can generate technical design from user stories
- Can generate API contracts (OpenAPI)
- Specs saved to Git repository

✅ **Codesmith-MCP:**
- Can generate project scaffold (TypeScript, Python)
- Can generate API routes from OpenAPI spec
- Can create Git branches and commits
- Can create Pull Requests

✅ **Sandbox-Runner-MCP:**
- Can spin up isolated containers
- Can execute tests (Jest, Pytest)
- Can parse test results
- Can stream logs in real-time

✅ **Integration:**
- Vibes-Director can orchestrate full pipeline
- End-to-end test passes: description → working, tested code
- All services instrumented with OpenTelemetry
- Logs and metrics visible in OpenObserve

---

## Next Steps After Phase 1

Once Phase 1 is complete:

1. **Deploy to staging environment** (not just local Docker)
2. **Begin Phase 2:** Control-Hub-Web, DevOps-Conductor-MCP
3. **Gather feedback** from initial usage
4. **Refine prompts** for better code generation quality
5. **Add more templates** for different project types

---

## Document Maintenance

**Update this document** as you:
- Complete tasks
- Discover better approaches
- Hit blockers
- Learn lessons

**Version:** 1.0
**Last Updated:** 2025-11-14
**Next Review:** After completing Month 1
