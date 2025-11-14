# Phase 2 Implementation Plan: Production Ready (Months 4-6)

> **Goal:** Secure, observable, deployable system for small team (2-10 people)
> **Timeline:** 12 weeks (3 months)
> **Prerequisites:** Phase 1 complete (working Spec→Code→Test pipeline)
> **Deliverable:** Production-ready ecosystem with web UI, CI/CD, and team access control

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Month 4: Security & Access Control](#month-4-security--access-control)
4. [Month 5: Deployment & Observability](#month-5-deployment--observability)
5. [Month 6: Web Interface](#month-6-web-interface)
6. [Production Deployment Guide](#production-deployment-guide)
7. [Security Hardening Checklist](#security-hardening-checklist)

---

## Overview

### What We're Building in Phase 2

```
┌─────────────────────────────────────────────────────────────┐
│                     PHASE 2 SCOPE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  NEW: Control-Hub-Web (React Dashboard)                     │
│         ↓                                                   │
│  ENHANCED: Gateway-Hub (Full RBAC, Rate Limiting, Quotas)  │
│         ↓                                                   │
│  All Phase 1 Services                                       │
│         ↓                                                   │
│  NEW: DevOps-Conductor-MCP (CI/CD Automation)               │
│         ↓                                                   │
│  ENHANCED: Observatory (Cost Tracking, Alerting)            │
│                                                             │
│  RESULT: Team-ready system with deployment automation       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### New Services

1. **DevOps-Conductor-MCP** - CI/CD orchestration and deployment
2. **Control-Hub-Web** - Web dashboard for team management

### Enhanced Services

1. **Gateway-Hub** - Add RBAC, rate limiting, cost quotas
2. **Observatory** - Add cost dashboards, alerting (Slack integration)
3. **All MCP services** - Add comprehensive OpenTelemetry instrumentation

---

## Prerequisites

### Phase 1 Completion Checklist

- ✅ All Phase 1 services running and tested
- ✅ End-to-end pipeline working (description → code → tests)
- ✅ PostgreSQL schema deployed
- ✅ OpenObserve collecting logs
- ✅ Basic authentication working

### New Requirements for Phase 2

**Infrastructure:**
- Production server/VPS (16GB RAM, 8 vCPU minimum)
- Domain name (e.g., ecosystem.yourcompany.com)
- SSL certificate (Let's Encrypt recommended)
- Slack workspace (for alerts)

**Software:**
- Nginx (reverse proxy)
- Certbot (SSL certificates)
- Kubernetes (optional, for advanced deployment)

**Accounts:**
- Docker Hub or private registry account
- Slack app with incoming webhook

---

## Month 4: Security & Access Control

### Week 1-2: Enhanced Gateway-Hub with RBAC

#### Task 4.1: Implement Role-Based Access Control

**Action:** Add comprehensive RBAC system to Gateway-Hub.

**File:** `services/gateway-hub/internal/rbac/rbac.go`

```go
package rbac

import (
	"errors"
	"fmt"
)

// Role definitions
type Role string

const (
	RoleAdmin     Role = "admin"
	RoleDeveloper Role = "developer"
	RoleViewer    Role = "viewer"
	RoleAIAgent   Role = "ai_agent"
)

// Permission definitions
type Permission string

const (
	// Spec permissions
	PermSpecRead   Permission = "spec:read"
	PermSpecWrite  Permission = "spec:write"
	PermSpecDelete Permission = "spec:delete"

	// Code permissions
	PermCodeRead   Permission = "code:read"
	PermCodeWrite  Permission = "code:write"
	PermCodeDelete Permission = "code:delete"

	// Workflow permissions
	PermWorkflowRead    Permission = "workflow:read"
	PermWorkflowWrite   Permission = "workflow:write"
	PermWorkflowExecute Permission = "workflow:execute"
	PermWorkflowDelete  Permission = "workflow:delete"

	// Admin permissions
	PermUserManage    Permission = "user:manage"
	PermServiceManage Permission = "service:manage"
	PermConfigManage  Permission = "config:manage"

	// Test permissions
	PermTestExecute Permission = "test:execute"
	PermTestRead    Permission = "test:read"
)

// RolePermissions maps roles to their allowed permissions
var RolePermissions = map[Role][]Permission{
	RoleAdmin: {
		// Admin has all permissions
		PermSpecRead, PermSpecWrite, PermSpecDelete,
		PermCodeRead, PermCodeWrite, PermCodeDelete,
		PermWorkflowRead, PermWorkflowWrite, PermWorkflowExecute, PermWorkflowDelete,
		PermUserManage, PermServiceManage, PermConfigManage,
		PermTestExecute, PermTestRead,
	},
	RoleDeveloper: {
		// Developer can read/write but not delete
		PermSpecRead, PermSpecWrite,
		PermCodeRead, PermCodeWrite,
		PermWorkflowRead, PermWorkflowWrite, PermWorkflowExecute,
		PermTestExecute, PermTestRead,
	},
	RoleViewer: {
		// Viewer can only read
		PermSpecRead,
		PermCodeRead,
		PermWorkflowRead,
		PermTestRead,
	},
	RoleAIAgent: {
		// AI agents can execute workflows and tests
		PermSpecRead, PermSpecWrite,
		PermCodeRead, PermCodeWrite,
		PermWorkflowRead, PermWorkflowExecute,
		PermTestExecute, PermTestRead,
	},
}

// Checker provides permission checking functionality
type Checker struct{}

// NewChecker creates a new RBAC checker
func NewChecker() *Checker {
	return &Checker{}
}

// HasPermission checks if a role has a specific permission
func (c *Checker) HasPermission(role Role, permission Permission) bool {
	permissions, exists := RolePermissions[role]
	if !exists {
		return false
	}

	for _, p := range permissions {
		if p == permission {
			return true
		}
	}
	return false
}

// CheckPermission returns an error if the role doesn't have the permission
func (c *Checker) CheckPermission(role Role, permission Permission) error {
	if !c.HasPermission(role, permission) {
		return fmt.Errorf("role %s does not have permission %s", role, permission)
	}
	return nil
}

// GetRolePermissions returns all permissions for a role
func (c *Checker) GetRolePermissions(role Role) ([]Permission, error) {
	permissions, exists := RolePermissions[role]
	if !exists {
		return nil, errors.New("invalid role")
	}
	return permissions, nil
}

// MCPToolPermissionMap maps MCP tools to required permissions
var MCPToolPermissionMap = map[string]Permission{
	"spec-synthesizer-mcp/generate_user_stories":    PermSpecWrite,
	"spec-synthesizer-mcp/generate_technical_design": PermSpecWrite,
	"spec-synthesizer-mcp/generate_api_contract":     PermSpecWrite,
	"spec-synthesizer-mcp/validate_spec":             PermSpecRead,

	"codesmith-mcp/generate_project_scaffold": PermCodeWrite,
	"codesmith-mcp/generate_api_routes":       PermCodeWrite,
	"codesmith-mcp/create_pull_request":       PermCodeWrite,

	"sandbox-runner-mcp/execute_code":  PermTestExecute,
	"sandbox-runner-mcp/run_tests":     PermTestExecute,
	"sandbox-runner-mcp/create_sandbox": PermTestExecute,

	"vibes-director/create_workflow":  PermWorkflowWrite,
	"vibes-director/execute_workflow": PermWorkflowExecute,
	"vibes-director/list_workflows":   PermWorkflowRead,
}

// GetRequiredPermissionForTool returns the permission needed for an MCP tool
func (c *Checker) GetRequiredPermissionForTool(service, tool string) (Permission, error) {
	key := fmt.Sprintf("%s/%s", service, tool)
	permission, exists := MCPToolPermissionMap[key]
	if !exists {
		return "", fmt.Errorf("unknown tool: %s", key)
	}
	return permission, nil
}
```

**File:** `services/gateway-hub/internal/ratelimit/ratelimit.go`

```go
package ratelimit

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// Limiter provides rate limiting functionality
type Limiter struct {
	redis *redis.Client
}

// Config holds rate limit configuration
type Config struct {
	RequestsPerMinute int
	RequestsPerHour   int
	BurstSize         int
}

// DefaultConfig returns default rate limit config
func DefaultConfig() Config {
	return Config{
		RequestsPerMinute: 100,
		RequestsPerHour:   1000,
		BurstSize:         10,
	}
}

// NewLimiter creates a new rate limiter
func NewLimiter(redisClient *redis.Client) *Limiter {
	return &Limiter{redis: redisClient}
}

// CheckLimit checks if a user has exceeded their rate limit
func (l *Limiter) CheckLimit(ctx context.Context, userID string, config Config) error {
	now := time.Now()

	// Check per-minute limit
	minuteKey := fmt.Sprintf("ratelimit:user:%s:minute:%s", userID, now.Format("2006-01-02-15-04"))
	minuteCount, err := l.increment(ctx, minuteKey, 60*time.Second)
	if err != nil {
		return err
	}
	if minuteCount > int64(config.RequestsPerMinute) {
		return fmt.Errorf("rate limit exceeded: %d requests per minute (limit: %d)",
			minuteCount, config.RequestsPerMinute)
	}

	// Check per-hour limit
	hourKey := fmt.Sprintf("ratelimit:user:%s:hour:%s", userID, now.Format("2006-01-02-15"))
	hourCount, err := l.increment(ctx, hourKey, 60*60*time.Second)
	if err != nil {
		return err
	}
	if hourCount > int64(config.RequestsPerHour) {
		return fmt.Errorf("rate limit exceeded: %d requests per hour (limit: %d)",
			hourCount, config.RequestsPerHour)
	}

	return nil
}

// increment increments a counter with expiry
func (l *Limiter) increment(ctx context.Context, key string, expiry time.Duration) (int64, error) {
	pipe := l.redis.Pipeline()
	incr := pipe.Incr(ctx, key)
	pipe.Expire(ctx, key, expiry)

	if _, err := pipe.Exec(ctx); err != nil {
		return 0, err
	}

	return incr.Val(), nil
}

// GetCurrentUsage returns the current usage for a user
func (l *Limiter) GetCurrentUsage(ctx context.Context, userID string) (minute, hour int64, err error) {
	now := time.Now()
	minuteKey := fmt.Sprintf("ratelimit:user:%s:minute:%s", userID, now.Format("2006-01-02-15-04"))
	hourKey := fmt.Sprintf("ratelimit:user:%s:hour:%s", userID, now.Format("2006-01-02-15"))

	minute, err = l.redis.Get(ctx, minuteKey).Int64()
	if err == redis.Nil {
		minute = 0
		err = nil
	} else if err != nil {
		return 0, 0, err
	}

	hour, err = l.redis.Get(ctx, hourKey).Int64()
	if err == redis.Nil {
		hour = 0
		err = nil
	} else if err != nil {
		return 0, 0, err
	}

	return minute, hour, nil
}
```

**File:** `services/gateway-hub/internal/quota/quota.go`

```go
package quota

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// Manager handles cost quotas
type Manager struct {
	redis *redis.Client
}

// Quota represents spending limits
type Quota struct {
	DailyLimitUSD   float64
	MonthlyLimitUSD float64
	CurrentDailyUSD float64
	CurrentMonthlyUSD float64
}

// NewManager creates a new quota manager
func NewManager(redisClient *redis.Client) *Manager {
	return &Manager{redis: redisClient}
}

// CheckQuota verifies if spending is within limits
func (m *Manager) CheckQuota(ctx context.Context, userID string, costUSD float64) error {
	quota, err := m.GetQuota(ctx, userID)
	if err != nil {
		return err
	}

	if quota.CurrentDailyUSD+costUSD > quota.DailyLimitUSD {
		return fmt.Errorf("daily quota exceeded: current $%.2f + $%.2f > limit $%.2f",
			quota.CurrentDailyUSD, costUSD, quota.DailyLimitUSD)
	}

	if quota.CurrentMonthlyUSD+costUSD > quota.MonthlyLimitUSD {
		return fmt.Errorf("monthly quota exceeded: current $%.2f + $%.2f > limit $%.2f",
			quota.CurrentMonthlyUSD, costUSD, quota.MonthlyLimitUSD)
	}

	return nil
}

// RecordSpending records AI model spending
func (m *Manager) RecordSpending(ctx context.Context, userID string, costUSD float64) error {
	now := time.Now()
	dailyKey := fmt.Sprintf("quota:user:%s:daily:%s", userID, now.Format("2006-01-02"))
	monthlyKey := fmt.Sprintf("quota:user:%s:monthly:%s", userID, now.Format("2006-01"))

	pipe := m.redis.Pipeline()
	pipe.IncrByFloat(ctx, dailyKey, costUSD)
	pipe.Expire(ctx, dailyKey, 24*time.Hour)
	pipe.IncrByFloat(ctx, monthlyKey, costUSD)
	pipe.Expire(ctx, monthlyKey, 31*24*time.Hour)

	_, err := pipe.Exec(ctx)
	return err
}

// GetQuota retrieves current quota status
func (m *Manager) GetQuota(ctx context.Context, userID string) (*Quota, error) {
	// Get limits from database (hardcoded for now)
	quota := &Quota{
		DailyLimitUSD:   50.0,
		MonthlyLimitUSD: 500.0,
	}

	// Get current spending
	now := time.Now()
	dailyKey := fmt.Sprintf("quota:user:%s:daily:%s", userID, now.Format("2006-01-02"))
	monthlyKey := fmt.Sprintf("quota:user:%s:monthly:%s", userID, now.Format("2006-01"))

	daily, err := m.redis.Get(ctx, dailyKey).Float64()
	if err == redis.Nil {
		daily = 0
	} else if err != nil {
		return nil, err
	}
	quota.CurrentDailyUSD = daily

	monthly, err := m.redis.Get(ctx, monthlyKey).Float64()
	if err == redis.Nil {
		monthly = 0
	} else if err != nil {
		return nil, err
	}
	quota.CurrentMonthlyUSD = monthly

	return quota, nil
}

// SetQuota updates quota limits for a user
func (m *Manager) SetQuota(ctx context.Context, userID string, dailyLimit, monthlyLimit float64) error {
	// TODO: Store in PostgreSQL instead of Redis
	return nil
}
```

#### Task 4.2: Update Gateway-Hub Main Application

**File:** `services/gateway-hub/main.go` (updated sections)

```go
// Add to imports
import (
	"github.com/your-org/ecosystem/gateway-hub/internal/rbac"
	"github.com/your-org/ecosystem/gateway-hub/internal/ratelimit"
	"github.com/your-org/ecosystem/gateway-hub/internal/quota"
)

// Add to main() after initializing redis
rbacChecker := rbac.NewChecker()
rateLimiter := ratelimit.NewLimiter(redisClient)
quotaManager := quota.NewManager(redisClient)

// Enhanced middleware stack for MCP routes
mcp := e.Group("/mcp")
mcp.Use(jwtMiddleware(config.JWTSecret))
mcp.Use(rateLimitMiddleware(rateLimiter))
mcp.Use(rbacMiddleware(rbacChecker))
mcp.Use(quotaMiddleware(quotaManager))
mcp.POST("/:service/:tool", proxyToMCPService)

// Middleware implementations
func rateLimitMiddleware(limiter *ratelimit.Limiter) echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			userID := c.Get("user_id").(string)

			config := ratelimit.DefaultConfig()
			if err := limiter.CheckLimit(c.Request().Context(), userID, config); err != nil {
				return c.JSON(http.StatusTooManyRequests, map[string]string{
					"error": err.Error(),
				})
			}

			return next(c)
		}
	}
}

func rbacMiddleware(checker *rbac.Checker) echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			role := rbac.Role(c.Get("role").(string))
			service := c.Param("service")
			tool := c.Param("tool")

			requiredPerm, err := checker.GetRequiredPermissionForTool(service, tool)
			if err != nil {
				return c.JSON(http.StatusBadRequest, map[string]string{
					"error": "Unknown tool",
				})
			}

			if err := checker.CheckPermission(role, requiredPerm); err != nil {
				return c.JSON(http.StatusForbidden, map[string]string{
					"error": err.Error(),
				})
			}

			return next(c)
		}
	}
}

func quotaMiddleware(manager *quota.Manager) echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			// Estimate cost before execution (simplified)
			estimatedCost := 0.01 // Will be calculated based on tool

			userID := c.Get("user_id").(string)
			if err := manager.CheckQuota(c.Request().Context(), userID, estimatedCost); err != nil {
				return c.JSON(http.StatusPaymentRequired, map[string]string{
					"error": err.Error(),
				})
			}

			return next(c)
		}
	}
}
```

---

### Week 3-4: Secrets Management & Security Audit

#### Task 4.3: Implement Secrets Manager

**File:** `services/devops-conductor-mcp/internal/secrets/secrets.go`

```go
package secrets

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"errors"
	"io"
)

// Manager handles encrypted secrets
type Manager struct {
	key []byte
}

// NewManager creates a secrets manager with encryption key
func NewManager(encryptionKey string) (*Manager, error) {
	// Key should be 32 bytes for AES-256
	key := []byte(encryptionKey)
	if len(key) != 32 {
		return nil, errors.New("encryption key must be 32 bytes")
	}

	return &Manager{key: key}, nil
}

// Encrypt encrypts a secret value
func (m *Manager) Encrypt(plaintext string) (string, error) {
	block, err := aes.NewCipher(m.key)
	if err != nil {
		return "", err
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return "", err
	}

	ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
	return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// Decrypt decrypts a secret value
func (m *Manager) Decrypt(ciphertext string) (string, error) {
	data, err := base64.StdEncoding.DecodeString(ciphertext)
	if err != nil {
		return "", err
	}

	block, err := aes.NewCipher(m.key)
	if err != nil {
		return "", err
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	nonceSize := gcm.NonceSize()
	if len(data) < nonceSize {
		return "", errors.New("ciphertext too short")
	}

	nonce, ciphertext := data[:nonceSize], data[nonceSize:]
	plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
	if err != nil {
		return "", err
	}

	return string(plaintext), nil
}
```

#### Task 4.4: Security Hardening

**File:** `infrastructure/security/security-checklist.md`

```markdown
# Security Hardening Checklist

## Authentication & Authorization
- [x] JWT tokens with expiry (1 hour for access, 7 days for refresh)
- [x] RBAC implemented (Admin, Developer, Viewer, AI Agent roles)
- [x] Rate limiting (100 req/min, 1000 req/hour)
- [x] API key support for service-to-service auth
- [ ] OAuth 2.1 integration (external IdP)
- [ ] MFA (multi-factor authentication) for admin accounts

## Network Security
- [ ] HTTPS only (redirect HTTP → HTTPS)
- [ ] SSL certificate (Let's Encrypt)
- [ ] Firewall rules (only expose necessary ports)
- [ ] Internal service network (Docker network isolation)
- [ ] VPN for admin access (optional but recommended)

## Data Security
- [x] Secrets encryption (AES-256)
- [x] Database passwords encrypted
- [ ] Secrets rotation policy (90 days)
- [ ] Backup encryption
- [ ] PII data handling (GDPR compliance if applicable)

## Container Security
- [ ] Non-root containers
- [ ] Read-only filesystems where possible
- [ ] Resource limits (CPU, memory)
- [ ] Image scanning (Trivy, Grype)
- [ ] Minimal base images (alpine, distroless)
- [ ] No secrets in images

## API Security
- [x] Input validation (Zod, Pydantic)
- [x] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (CSP headers)
- [ ] CORS configuration
- [ ] Request size limits
- [ ] Timeout configuration

## Monitoring & Logging
- [ ] Centralized logging (all auth events)
- [ ] Alert on suspicious activity
- [ ] Audit trail (who did what when)
- [ ] Log retention policy (90 days)
- [ ] Regular security reviews

## Compliance
- [ ] Document data flows
- [ ] Privacy policy
- [ ] Terms of service
- [ ] Data retention policy
- [ ] Incident response plan
```

---

## Month 5: Deployment & Observability

### Week 1-2: DevOps-Conductor-MCP Implementation

#### Task 5.1: Create DevOps Conductor Service

**File:** `services/devops-conductor-mcp/main.py`

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel, Field
from typing import Dict, Any, List, Optional, Literal
import os
import logging
import asyncio
import subprocess
from datetime import datetime
import yaml

logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO").upper())
logger = logging.getLogger(__name__)

app = FastAPI(title="DevOps Conductor MCP")

# Models
class PipelineConfig(BaseModel):
    steps: List[Dict[str, Any]]
    environment: Dict[str, str] = {}
    on_failure: Literal["stop", "continue", "rollback"] = "stop"


class CreatePipelineRequest(BaseModel):
    repo_url: str
    branch: str = "main"
    config: PipelineConfig


class TriggerPipelineRequest(BaseModel):
    pipeline_id: str
    branch: Optional[str] = None
    variables: Dict[str, str] = {}


class DeploymentStrategy(BaseModel):
    type: Literal["rolling", "blue-green", "canary"] = "rolling"
    max_unavailable: int = 1
    health_check_url: Optional[str] = None
    rollback_on_failure: bool = True


class DeployRequest(BaseModel):
    service_name: str
    version: str  # Docker image tag or git commit
    environment: Literal["dev", "staging", "prod"]
    strategy: DeploymentStrategy = DeploymentStrategy()
    env_vars: Dict[str, str] = {}


class DeploymentStatus(BaseModel):
    deployment_id: str
    status: Literal["pending", "in_progress", "completed", "failed", "rolled_back"]
    progress: int  # 0-100
    message: str
    started_at: datetime
    completed_at: Optional[datetime] = None


# In-memory storage (should use PostgreSQL in production)
pipelines: Dict[str, Any] = {}
pipeline_runs: Dict[str, Any] = {}
deployments: Dict[str, DeploymentStatus] = {}


@app.get("/")
async def root():
    return {
        "name": "devops-conductor-mcp",
        "version": "0.1.0",
        "protocol": "mcp",
        "tools": [
            {
                "name": "create_pipeline",
                "description": "Create a CI/CD pipeline for a repository",
            },
            {
                "name": "trigger_pipeline",
                "description": "Trigger a pipeline execution",
            },
            {
                "name": "deploy",
                "description": "Deploy a service to an environment",
            },
            {
                "name": "get_deployment_status",
                "description": "Get the status of a deployment",
            },
            {
                "name": "rollback",
                "description": "Rollback a deployment to previous version",
            },
        ],
    }


@app.get("/health")
async def health():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}


@app.post("/tools/create_pipeline")
async def create_pipeline(request: CreatePipelineRequest):
    """Create a new CI/CD pipeline."""
    pipeline_id = f"pipeline-{len(pipelines) + 1}"

    pipelines[pipeline_id] = {
        "id": pipeline_id,
        "repo_url": request.repo_url,
        "branch": request.branch,
        "config": request.config.dict(),
        "created_at": datetime.utcnow().isoformat(),
    }

    logger.info(f"Created pipeline {pipeline_id} for {request.repo_url}")

    return {
        "pipeline_id": pipeline_id,
        "message": "Pipeline created successfully",
    }


@app.post("/tools/trigger_pipeline")
async def trigger_pipeline(request: TriggerPipelineRequest, background_tasks: BackgroundTasks):
    """Trigger a pipeline execution."""
    if request.pipeline_id not in pipelines:
        raise HTTPException(status_code=404, detail="Pipeline not found")

    run_id = f"run-{len(pipeline_runs) + 1}"
    pipeline = pipelines[request.pipeline_id]

    run = {
        "id": run_id,
        "pipeline_id": request.pipeline_id,
        "status": "pending",
        "started_at": datetime.utcnow().isoformat(),
        "variables": request.variables,
    }
    pipeline_runs[run_id] = run

    # Execute pipeline in background
    background_tasks.add_task(execute_pipeline, run_id, pipeline)

    return {
        "run_id": run_id,
        "status": "pending",
        "message": "Pipeline execution started",
    }


async def execute_pipeline(run_id: str, pipeline: Dict[str, Any]):
    """Execute pipeline steps."""
    run = pipeline_runs[run_id]
    run["status"] = "running"

    try:
        for i, step in enumerate(pipeline["config"]["steps"]):
            logger.info(f"Executing step {i + 1}: {step.get('name', 'unnamed')}")

            # Execute step (simplified - should handle different step types)
            if step["type"] == "shell":
                result = subprocess.run(
                    step["command"],
                    shell=True,
                    capture_output=True,
                    text=True,
                    timeout=300,  # 5 minutes
                )

                if result.returncode != 0:
                    run["status"] = "failed"
                    run["error"] = result.stderr
                    run["completed_at"] = datetime.utcnow().isoformat()
                    logger.error(f"Step failed: {result.stderr}")
                    return

            elif step["type"] == "docker_build":
                # Build Docker image
                logger.info(f"Building image: {step['image']}")
                # TODO: Implement docker build

            elif step["type"] == "test":
                # Run tests via sandbox-runner-mcp
                logger.info("Running tests")
                # TODO: Call sandbox-runner-mcp

        run["status"] = "completed"
        run["completed_at"] = datetime.utcnow().isoformat()
        logger.info(f"Pipeline {run_id} completed successfully")

    except Exception as e:
        run["status"] = "failed"
        run["error"] = str(e)
        run["completed_at"] = datetime.utcnow().isoformat()
        logger.error(f"Pipeline {run_id} failed: {e}")


@app.post("/tools/deploy", response_model=Dict[str, Any])
async def deploy(request: DeployRequest, background_tasks: BackgroundTasks):
    """Deploy a service to an environment."""
    deployment_id = f"deploy-{len(deployments) + 1}"

    deployment = DeploymentStatus(
        deployment_id=deployment_id,
        status="pending",
        progress=0,
        message="Deployment queued",
        started_at=datetime.utcnow(),
    )
    deployments[deployment_id] = deployment

    # Execute deployment in background
    background_tasks.add_task(execute_deployment, deployment_id, request)

    return {
        "deployment_id": deployment_id,
        "status": "pending",
        "message": "Deployment started",
    }


async def execute_deployment(deployment_id: str, request: DeployRequest):
    """Execute deployment steps."""
    deployment = deployments[deployment_id]
    deployment.status = "in_progress"
    deployment.message = "Pulling new image"
    deployment.progress = 10

    try:
        # Pull Docker image
        logger.info(f"Pulling image for {request.service_name}:{request.version}")
        await asyncio.sleep(2)  # Simulate pull
        deployment.progress = 30

        # Run health check on new version
        deployment.message = "Running health checks"
        await asyncio.sleep(1)
        deployment.progress = 50

        # Deploy based on strategy
        if request.strategy.type == "rolling":
            deployment.message = "Rolling update in progress"
            await rolling_update(request)
        elif request.strategy.type == "blue-green":
            deployment.message = "Blue-green deployment"
            await blue_green_deployment(request)
        elif request.strategy.type == "canary":
            deployment.message = "Canary deployment"
            await canary_deployment(request)

        deployment.progress = 90

        # Final health check
        deployment.message = "Verifying deployment"
        if request.strategy.health_check_url:
            # TODO: Perform health check
            pass

        deployment.status = "completed"
        deployment.progress = 100
        deployment.message = "Deployment completed successfully"
        deployment.completed_at = datetime.utcnow()

        logger.info(f"Deployment {deployment_id} completed")

    except Exception as e:
        logger.error(f"Deployment {deployment_id} failed: {e}")
        deployment.status = "failed"
        deployment.message = f"Deployment failed: {str(e)}"
        deployment.completed_at = datetime.utcnow()

        if request.strategy.rollback_on_failure:
            deployment.message = "Rolling back..."
            await rollback_deployment(deployment_id)


async def rolling_update(request: DeployRequest):
    """Perform rolling update."""
    logger.info(f"Rolling update for {request.service_name}")
    # TODO: Implement actual Docker/K8s rolling update
    await asyncio.sleep(3)


async def blue_green_deployment(request: DeployRequest):
    """Perform blue-green deployment."""
    logger.info(f"Blue-green deployment for {request.service_name}")
    # TODO: Implement blue-green deployment
    await asyncio.sleep(3)


async def canary_deployment(request: DeployRequest):
    """Perform canary deployment."""
    logger.info(f"Canary deployment for {request.service_name}")
    # TODO: Implement canary deployment with gradual rollout
    await asyncio.sleep(3)


async def rollback_deployment(deployment_id: str):
    """Rollback a deployment."""
    logger.info(f"Rolling back deployment {deployment_id}")
    # TODO: Implement rollback logic
    await asyncio.sleep(2)

    deployment = deployments[deployment_id]
    deployment.status = "rolled_back"
    deployment.message = "Deployment rolled back to previous version"


@app.get("/tools/get_deployment_status/{deployment_id}", response_model=DeploymentStatus)
async def get_deployment_status(deployment_id: str):
    """Get deployment status."""
    if deployment_id not in deployments:
        raise HTTPException(status_code=404, detail="Deployment not found")

    return deployments[deployment_id]


@app.post("/tools/rollback")
async def rollback(deployment_id: str, background_tasks: BackgroundTasks):
    """Manually trigger a rollback."""
    if deployment_id not in deployments:
        raise HTTPException(status_code=404, detail="Deployment not found")

    background_tasks.add_task(rollback_deployment, deployment_id)

    return {
        "deployment_id": deployment_id,
        "message": "Rollback initiated",
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=int(os.getenv("PORT", "3004")))
```

**File:** `services/devops-conductor-mcp/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    docker.io \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
RUN pip install fastapi uvicorn pydantic pyyaml

# Copy application
COPY . .

EXPOSE 3004

CMD ["python", "main.py"]
```

---

### Week 3-4: Enhanced Observatory with Alerting

#### Task 5.2: Slack Integration for Alerts

**File:** `services/observatory/alerts/slack.py`

```python
import os
import httpx
from typing import Dict, Any, List
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class SlackAlerter:
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    async def send_alert(
        self,
        title: str,
        message: str,
        severity: str = "warning",
        metadata: Dict[str, Any] = None
    ):
        """Send an alert to Slack."""
        color = {
            "info": "#36a64f",
            "warning": "#ff9900",
            "error": "#ff0000",
            "critical": "#8B0000",
        }.get(severity, "#808080")

        payload = {
            "attachments": [
                {
                    "color": color,
                    "title": title,
                    "text": message,
                    "fields": [
                        {
                            "title": "Severity",
                            "value": severity.upper(),
                            "short": True
                        },
                        {
                            "title": "Time",
                            "value": datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC"),
                            "short": True
                        }
                    ],
                    "footer": "Ecosystem Observatory",
                    "ts": int(datetime.utcnow().timestamp())
                }
            ]
        }

        # Add metadata fields
        if metadata:
            for key, value in metadata.items():
                payload["attachments"][0]["fields"].append({
                    "title": key,
                    "value": str(value),
                    "short": True
                })

        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(self.webhook_url, json=payload)
                response.raise_for_status()
                logger.info(f"Alert sent to Slack: {title}")
        except Exception as e:
            logger.error(f"Failed to send Slack alert: {e}")


# Alert conditions
class AlertRule:
    def __init__(
        self,
        name: str,
        condition: str,
        threshold: float,
        severity: str = "warning",
        message_template: str = None
    ):
        self.name = name
        self.condition = condition
        self.threshold = threshold
        self.severity = severity
        self.message_template = message_template or f"{name} threshold exceeded"


# Predefined alert rules
DEFAULT_ALERT_RULES = [
    AlertRule(
        name="High API Latency",
        condition="avg_latency > threshold",
        threshold=1000,  # ms
        severity="warning",
        message_template="Average API latency is {value}ms (threshold: {threshold}ms)"
    ),
    AlertRule(
        name="Error Rate Spike",
        condition="error_rate > threshold",
        threshold=5,  # percent
        severity="error",
        message_template="Error rate is {value}% (threshold: {threshold}%)"
    ),
    AlertRule(
        name="Daily Cost Exceeded",
        condition="daily_cost > threshold",
        threshold=100,  # USD
        severity="warning",
        message_template="Daily AI costs: ${value} (budget: ${threshold})"
    ),
    AlertRule(
        name="Monthly Cost Warning",
        condition="monthly_cost > threshold",
        threshold=500,  # USD
        severity="warning",
        message_template="Monthly AI costs: ${value} (budget: ${threshold})"
    ),
    AlertRule(
        name="Service Down",
        condition="service_health == unhealthy",
        threshold=0,
        severity="critical",
        message_template="Service {service_name} is unhealthy"
    ),
]


# Alert manager
class AlertManager:
    def __init__(self, slack_webhook_url: str):
        self.slack = SlackAlerter(slack_webhook_url)
        self.rules = DEFAULT_ALERT_RULES
        self.alert_history: List[Dict[str, Any]] = []

    async def check_and_alert(self, metrics: Dict[str, Any]):
        """Check metrics against rules and send alerts."""
        for rule in self.rules:
            should_alert = self._evaluate_rule(rule, metrics)
            if should_alert:
                await self._send_alert(rule, metrics)

    def _evaluate_rule(self, rule: AlertRule, metrics: Dict[str, Any]) -> bool:
        """Evaluate if a rule condition is met."""
        # Simplified evaluation - should use proper expression parser
        if rule.condition == "avg_latency > threshold":
            return metrics.get("avg_latency", 0) > rule.threshold
        elif rule.condition == "error_rate > threshold":
            return metrics.get("error_rate", 0) > rule.threshold
        elif rule.condition == "daily_cost > threshold":
            return metrics.get("daily_cost_usd", 0) > rule.threshold
        elif rule.condition == "monthly_cost > threshold":
            return metrics.get("monthly_cost_usd", 0) > rule.threshold
        elif rule.condition == "service_health == unhealthy":
            return metrics.get("service_health") == "unhealthy"
        return False

    async def _send_alert(self, rule: AlertRule, metrics: Dict[str, Any]):
        """Send an alert based on a rule."""
        # Check if we recently sent this alert (avoid spam)
        recent_alerts = [
            a for a in self.alert_history[-10:]
            if a["rule_name"] == rule.name
            and (datetime.utcnow() - a["timestamp"]).seconds < 300  # 5 minutes
        ]
        if recent_alerts:
            logger.info(f"Skipping alert {rule.name} - recently sent")
            return

        # Format message
        message = rule.message_template.format(
            threshold=rule.threshold,
            **metrics
        )

        await self.slack.send_alert(
            title=f"🚨 {rule.name}",
            message=message,
            severity=rule.severity,
            metadata=metrics
        )

        # Record alert
        self.alert_history.append({
            "rule_name": rule.name,
            "timestamp": datetime.utcnow(),
            "metrics": metrics
        })
```

---

## Month 6: Web Interface

### Week 1-2: Control-Hub-Web Foundation

#### Task 6.1: React Application Setup

**File:** `services/control-hub-web/package.json`

```json
{
  "name": "control-hub-web",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.21.0",
    "@tanstack/react-query": "^5.17.0",
    "zustand": "^4.4.7",
    "axios": "^1.6.2",
    "recharts": "^2.10.3",
    "react-flow-renderer": "^10.3.17",
    "socket.io-client": "^4.6.0",
    "lucide-react": "^0.300.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.43",
    "@types/react-dom": "^18.2.17",
    "@vitejs/plugin-react": "^4.2.1",
    "typescript": "^5.3.3",
    "vite": "^5.0.8",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.32"
  }
}
```

**File:** `services/control-hub-web/src/App.tsx`

```typescript
import React from 'react';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Dashboard } from './pages/Dashboard';
import { Workflows } from './pages/Workflows';
import { Projects } from './pages/Projects';
import { Team } from './pages/Team';
import { Settings } from './pages/Settings';
import { Login } from './pages/Login';
import { Layout } from './components/Layout';
import { useAuthStore } from './stores/authStore';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <AppRoutes />
      </BrowserRouter>
    </QueryClientProvider>
  );
}

function AppRoutes() {
  const { isAuthenticated } = useAuthStore();

  if (!isAuthenticated) {
    return (
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="*" element={<Navigate to="/login" replace />} />
      </Routes>
    );
  }

  return (
    <Layout>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/workflows" element={<Workflows />} />
        <Route path="/projects" element={<Projects />} />
        <Route path="/team" element={<Team />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="*" element={<Navigate to="/" replace />} />
      </Routes>
    </Layout>
  );
}

export default App;
```

**File:** `services/control-hub-web/src/stores/authStore.ts`

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import axios from 'axios';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8080';

interface User {
  id: string;
  email: string;
  role: string;
}

interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  refreshAccessToken: () => Promise<void>;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      refreshToken: null,
      isAuthenticated: false,

      login: async (email: string, password: string) => {
        try {
          const response = await axios.post(`${API_URL}/auth/login`, {
            email,
            password,
          });

          const { access_token, refresh_token, user } = response.data;

          set({
            accessToken: access_token,
            refreshToken: refresh_token,
            user,
            isAuthenticated: true,
          });

          // Set default auth header
          axios.defaults.headers.common['Authorization'] = `Bearer ${access_token}`;
        } catch (error) {
          console.error('Login failed:', error);
          throw error;
        }
      },

      logout: () => {
        set({
          user: null,
          accessToken: null,
          refreshToken: null,
          isAuthenticated: false,
        });
        delete axios.defaults.headers.common['Authorization'];
      },

      refreshAccessToken: async () => {
        const { refreshToken } = get();
        if (!refreshToken) {
          throw new Error('No refresh token available');
        }

        try {
          const response = await axios.post(`${API_URL}/auth/refresh`, {
            refresh_token: refreshToken,
          });

          const { access_token } = response.data;

          set({ accessToken: access_token });
          axios.defaults.headers.common['Authorization'] = `Bearer ${access_token}`;
        } catch (error) {
          // Refresh failed, logout user
          get().logout();
          throw error;
        }
      },
    }),
    {
      name: 'auth-storage',
    }
  )
);
```

**File:** `services/control-hub-web/src/pages/Dashboard.tsx`

```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { Activity, DollarSign, CheckCircle, AlertCircle } from 'lucide-react';
import { Card } from '../components/Card';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';
import axios from 'axios';

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8080';

export function Dashboard() {
  const { data: stats } = useQuery({
    queryKey: ['dashboard-stats'],
    queryFn: async () => {
      const response = await axios.get(`${API_URL}/api/dashboard/stats`);
      return response.data;
    },
  });

  const { data: costData } = useQuery({
    queryKey: ['cost-trends'],
    queryFn: async () => {
      const response = await axios.get(`${API_URL}/api/dashboard/cost-trends`);
      return response.data;
    },
  });

  return (
    <div className="p-6 space-y-6">
      <h1 className="text-3xl font-bold">Dashboard</h1>

      {/* Stats Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
        <Card>
          <div className="flex items-center justify-between">
            <div>
              <p className="text-sm text-gray-600">Active Workflows</p>
              <p className="text-3xl font-bold">{stats?.activeWorkflows || 0}</p>
            </div>
            <Activity className="h-12 w-12 text-blue-500" />
          </div>
        </Card>

        <Card>
          <div className="flex items-center justify-between">
            <div>
              <p className="text-sm text-gray-600">Today's Cost</p>
              <p className="text-3xl font-bold">${stats?.dailyCost?.toFixed(2) || '0.00'}</p>
            </div>
            <DollarSign className="h-12 w-12 text-green-500" />
          </div>
        </Card>

        <Card>
          <div className="flex items-center justify-between">
            <div>
              <p className="text-sm text-gray-600">Completed Today</p>
              <p className="text-3xl font-bold">{stats?.completedToday || 0}</p>
            </div>
            <CheckCircle className="h-12 w-12 text-green-500" />
          </div>
        </Card>

        <Card>
          <div className="flex items-center justify-between">
            <div>
              <p className="text-sm text-gray-600">Failed Today</p>
              <p className="text-3xl font-bold">{stats?.failedToday || 0}</p>
            </div>
            <AlertCircle className="h-12 w-12 text-red-500" />
          </div>
        </Card>
      </div>

      {/* Cost Trends Chart */}
      <Card>
        <h2 className="text-xl font-bold mb-4">Cost Trends (Last 7 Days)</h2>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={costData || []}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="date" />
            <YAxis />
            <Tooltip />
            <Line type="monotone" dataKey="cost" stroke="#8884d8" strokeWidth={2} />
          </LineChart>
        </ResponsiveContainer>
      </Card>

      {/* Recent Workflows */}
      <Card>
        <h2 className="text-xl font-bold mb-4">Recent Workflows</h2>
        <div className="space-y-2">
          {stats?.recentWorkflows?.map((workflow: any) => (
            <div
              key={workflow.id}
              className="flex items-center justify-between p-3 bg-gray-50 rounded-lg"
            >
              <div>
                <p className="font-medium">{workflow.name}</p>
                <p className="text-sm text-gray-600">{workflow.createdAt}</p>
              </div>
              <span
                className={`px-3 py-1 rounded-full text-sm ${
                  workflow.status === 'completed'
                    ? 'bg-green-100 text-green-800'
                    : workflow.status === 'failed'
                    ? 'bg-red-100 text-red-800'
                    : 'bg-blue-100 text-blue-800'
                }`}
              >
                {workflow.status}
              </span>
            </div>
          ))}
        </div>
      </Card>
    </div>
  );
}
```

---

### Week 3-4: Workflow Builder & Team Management

#### Task 6.2: Visual Workflow Builder

**File:** `services/control-hub-web/src/pages/Workflows.tsx`

```typescript
import React, { useCallback, useState } from 'react';
import ReactFlow, {
  addEdge,
  Background,
  Connection,
  Controls,
  Edge,
  Node,
  useEdgesState,
  useNodesState,
} from 'react-flow-renderer';
import { Play, Save, Plus } from 'lucide-react';
import { Card } from '../components/Card';

const initialNodes: Node[] = [
  {
    id: '1',
    type: 'input',
    data: { label: 'Start: User Input' },
    position: { x: 250, y: 0 },
  },
];

const initialEdges: Edge[] = [];

export function Workflows() {
  const [nodes, setNodes, onNodesChange] = useNodesState(initialNodes);
  const [edges, setEdges, onEdgesChange] = useEdgesState(initialEdges);
  const [selectedNodeType, setSelectedNodeType] = useState<string>('');

  const onConnect = useCallback(
    (params: Connection) => setEdges((eds) => addEdge(params, eds)),
    [setEdges]
  );

  const addNode = (type: string) => {
    const newNode: Node = {
      id: `${nodes.length + 1}`,
      type: 'default',
      data: { label: type },
      position: { x: Math.random() * 400, y: Math.random() * 400 },
    };
    setNodes((nds) => [...nds, newNode]);
  };

  const nodeTypes = [
    { id: 'research', label: 'Research (Web Search)', color: 'blue' },
    { id: 'spec', label: 'Generate Spec', color: 'purple' },
    { id: 'code', label: 'Generate Code', color: 'green' },
    { id: 'test', label: 'Run Tests', color: 'orange' },
    { id: 'deploy', label: 'Deploy', color: 'red' },
  ];

  return (
    <div className="p-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Workflows</h1>
        <div className="flex gap-2">
          <button className="flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700">
            <Play className="h-4 w-4" />
            Run Workflow
          </button>
          <button className="flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700">
            <Save className="h-4 w-4" />
            Save Workflow
          </button>
        </div>
      </div>

      <div className="grid grid-cols-4 gap-6">
        {/* Toolbox */}
        <Card className="col-span-1">
          <h2 className="text-lg font-bold mb-4">Add Steps</h2>
          <div className="space-y-2">
            {nodeTypes.map((type) => (
              <button
                key={type.id}
                onClick={() => addNode(type.label)}
                className={`w-full flex items-center gap-2 p-3 rounded-lg border-2 border-${type.color}-200 hover:border-${type.color}-400 transition-colors`}
              >
                <Plus className="h-4 w-4" />
                {type.label}
              </button>
            ))}
          </div>
        </Card>

        {/* Canvas */}
        <Card className="col-span-3" style={{ height: '600px' }}>
          <ReactFlow
            nodes={nodes}
            edges={edges}
            onNodesChange={onNodesChange}
            onEdgesChange={onEdgesChange}
            onConnect={onConnect}
            fitView
          >
            <Background />
            <Controls />
          </ReactFlow>
        </Card>
      </div>
    </div>
  );
}
```

---

## Production Deployment Guide

### Step 1: Server Preparation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose-plugin

# Install Nginx
sudo apt install nginx

# Install Certbot (for SSL)
sudo apt install certbot python3-certbot-nginx
```

### Step 2: Domain & SSL Setup

```bash
# Point your domain to server IP in DNS

# Get SSL certificate
sudo certbot --nginx -d ecosystem.yourcompany.com

# Auto-renewal (runs twice daily)
sudo systemctl enable certbot.timer
```

### Step 3: Nginx Configuration

**File:** `/etc/nginx/sites-available/ecosystem`

```nginx
upstream gateway {
    server localhost:8080;
}

upstream web {
    server localhost:3000;
}

server {
    listen 80;
    server_name ecosystem.yourcompany.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ecosystem.yourcompany.com;

    ssl_certificate /etc/letsencrypt/live/ecosystem.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ecosystem.yourcompany.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Web interface
    location / {
        proxy_pass http://web;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # API Gateway
    location /api/ {
        proxy_pass http://gateway/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # WebSocket support
    location /ws {
        proxy_pass http://gateway;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/ecosystem /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Step 4: Deploy Services

```bash
# Clone repository
git clone https://github.com/your-org/ecosystem.git
cd ecosystem

# Set up environment
cd infrastructure/docker
cp .env.example .env
# Edit .env with production values

# Pull and start services
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

---

## Security Hardening Checklist

### Before Going to Production

- [ ] Change all default passwords
- [ ] Generate strong JWT secret (32+ characters)
- [ ] Enable HTTPS only (redirect HTTP)
- [ ] Configure firewall (UFW or iptables)
- [ ] Set up automated backups (PostgreSQL, Redis)
- [ ] Configure log rotation
- [ ] Enable audit logging
- [ ] Set up monitoring alerts
- [ ] Review and restrict CORS settings
- [ ] Implement rate limiting
- [ ] Set up intrusion detection (Fail2ban)
- [ ] Regular security updates schedule

---

## Success Criteria for Phase 2

✅ **Security & Access Control:**
- RBAC implemented (4 roles with permissions)
- Rate limiting active (100 req/min)
- Cost quotas enforced ($50/day, $500/month)
- Secrets encrypted
- Security audit completed

✅ **Deployment & CI/CD:**
- DevOps-Conductor-MCP operational
- Can deploy services with rolling updates
- Rollback capability working
- Health checks configured

✅ **Observability:**
- Enhanced Observatory with cost dashboards
- Slack alerts configured and tested
- All services instrumented with OpenTelemetry
- Alert rules active (latency, errors, costs)

✅ **Web Interface:**
- Control-Hub-Web deployed and accessible
- Dashboard showing real-time metrics
- Workflow builder functional
- Team management working
- WebSocket real-time updates

✅ **Production Deployment:**
- Running on production server
- HTTPS configured with valid SSL
- Nginx reverse proxy
- Automated backups
- Monitoring and alerts active

---

## Next Steps After Phase 2

1. **User feedback collection** - Gather feedback from team
2. **Performance optimization** - Identify and fix bottlenecks
3. **Begin Phase 3** - RAG Engine and Research Automation
4. **Documentation** - User guides and API docs
5. **Training** - Team onboarding sessions

---

**Document Version:** 1.0
**Last Updated:** 2025-11-14
**Next Review:** After completing Month 4
