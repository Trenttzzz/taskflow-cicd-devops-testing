# TaskFlow CI/CD Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement end-to-end CI/CD pipeline for TaskFlow Go API including bug fixes, automated testing, Docker image build/push, smoke tests, rollback strategy, and security scanning.

**Architecture:** Multi-job GitHub Actions pipeline with explicit dependencies (CI → CD → Smoke Test → Notify). Go backend with PostgreSQL, multi-stage Docker build to scratch image, GHCR registry, conditional stable tagging. **Matrix testing across Go 1.21, 1.22, 1.23** — ensures backward compatibility.

**Tech Stack:** Go 1.21/1.22/1.23 (matrix), PostgreSQL 16, Docker (multi-stage), GitHub Actions, GitHub Container Registry (GHCR), govulncheck, gosec

---

## File Structure

| File | Responsibility |
|------|---------------|
| `internal/handler/handler.go:28-36` | Refactor routing for Go 1.21 compatibility (matrix testing) |
| `go.mod:3` | Lower Go directive to 1.21 for matrix testing |
| `internal/service/service.go:172` | Bug #1: integer division in CalculateCompletionRate |
| `internal/repository/memory.go:58` | Bug #2: inverted filter in FindByStatus |
| `internal/repository/postgres.go:113` | Bug #2 (postgres): inverted SQL operator |
| `internal/validator/validator.go:15` | Bug #3: invalid "urgent" priority accepted |
| `internal/service/service_test.go` | Add ≥2 new test cases |
| `internal/repository/memory_test.go` | Add ≥2 new test cases |
| `.github/workflows/ci-cd.yml` | Full CI/CD pipeline definition with Go matrix |
| `scripts/smoke-test.sh` | Smoke test script post-deploy |
| `scripts/notify.sh` | Notification script for Slack/Telegram |
| `ROLLBACK_PROCEDURE.md` | One-page rollback documentation |
| `Dockerfile` | Already exists — multi-stage builder → scratch |
| `Makefile` | Already exists — may need minor updates |

---

## Task 0: Refactor Routing for Go 1.21 Compatibility (Matrix Testing)

> **Why:** Kelompok 2 requires matrix testing across Go 1.21, 1.22, 1.23. The current code uses Go 1.22's `net/http` pattern routing (`"GET /health"`), which does not exist in Go 1.21. We must refactor to a manual router that works across all three versions.

**Files:**
- Modify: `internal/handler/handler.go:28-36`
- Modify: `go.mod:3`

- [ ] **Step 1: Update go.mod to Go 1.21**

Edit `go.mod` line 3:

```go
// BEFORE:
go 1.22.0

// AFTER:
go 1.21
```

- [ ] **Step 2: Refactor RegisterRoutes to manual routing**

Replace `RegisterRoutes` method in `internal/handler/handler.go` (lines 28-36):

```go
// BEFORE (Go 1.22 pattern routing — incompatible with Go 1.21):
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
	mux.HandleFunc("GET /health", h.Health)
	mux.HandleFunc("GET /api/v1/tasks", h.ListTasks)
	mux.HandleFunc("POST /api/v1/tasks", h.CreateTask)
	mux.HandleFunc("GET /api/v1/tasks/{id}", h.GetTask)
	mux.HandleFunc("PUT /api/v1/tasks/{id}", h.UpdateTask)
	mux.HandleFunc("DELETE /api/v1/tasks/{id}", h.DeleteTask)
	mux.HandleFunc("GET /api/v1/stats", h.GetStats)
}

// AFTER (manual routing — compatible with Go 1.21, 1.22, 1.23):
func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
	mux.HandleFunc("/health", h.route(h.Health, http.MethodGet))
	mux.HandleFunc("/api/v1/tasks", h.route(h.ListTasks, http.MethodGet, http.MethodPost))
	mux.HandleFunc("/api/v1/tasks/", h.routeTaskByID)
	mux.HandleFunc("/api/v1/stats", h.route(h.GetStats, http.MethodGet))
}

// route wraps a handler to enforce allowed HTTP methods.
func (h *Handler) route(handler http.HandlerFunc, methods ...string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		for _, m := range methods {
			if r.Method == m {
				handler(w, r)
				return
			}
		}
		w.WriteHeader(http.StatusMethodNotAllowed)
	}
}

// routeTaskByID routes /api/v1/tasks/{id} based on HTTP method.
func (h *Handler) routeTaskByID(w http.ResponseWriter, r *http.Request) {
	// Strip prefix to get the ID
	id := strings.TrimPrefix(r.URL.Path, "/api/v1/tasks/")
	switch r.Method {
	case http.MethodGet:
		h.GetTask(w, r.WithContext(context.WithValue(r.Context(), pathKey("id"), id)))
	case http.MethodPut:
		h.UpdateTask(w, r.WithContext(context.WithValue(r.Context(), pathKey("id"), id)))
	case http.MethodDelete:
		h.DeleteTask(w, r.WithContext(context.WithValue(r.Context(), pathKey("id"), id)))
	default:
		w.WriteHeader(http.StatusMethodNotAllowed)
	}
}

type pathKey string
```

- [ ] **Step 3: Update GetTask, UpdateTask, DeleteTask to read ID from context**

Edit `GetTask`, `UpdateTask`, `DeleteTask` in `handler.go` to read `id` from context instead of `r.PathValue`:

```go
func getID(r *http.Request) string {
	if v := r.Context().Value(pathKey("id")); v != nil {
		return v.(string)
	}
	return ""
}

// Update GetTask:
func (h *Handler) GetTask(w http.ResponseWriter, r *http.Request) {
	task, err := h.svc.GetByID(getID(r))
	...
}

// Update UpdateTask:
func (h *Handler) UpdateTask(w http.ResponseWriter, r *http.Request) {
	...
	task, err := h.svc.Update(getID(r), req)
	...
}

// Update DeleteTask:
func (h *Handler) DeleteTask(w http.ResponseWriter, r *http.Request) {
	deleted, err := h.svc.Delete(getID(r))
	...
}
```

- [ ] **Step 4: Add context import**

Add to imports in `handler.go`:

```go
import (
	"context"
	...
)
```

- [ ] **Step 5: Run tests locally with Go 1.22**

```bash
go mod tidy
go test ./... -v
```

Expected: All PASS (routing behavior unchanged)

- [ ] **Step 6: Verify build with Go 1.21 logic**

Since local Go is 1.22, manually verify syntax is 1.21-compatible (no `r.PathValue`, no pattern strings in HandleFunc).

- [ ] **Step 7: Commit**

```bash
git add internal/handler/handler.go go.mod
git commit -m "refactor: make routing compatible with Go 1.21 for matrix testing

- Replaced Go 1.22 pattern routing with manual method+path routing
- Added context-based ID passing for /api/v1/tasks/{id}
- Lowered go.mod directive to 1.21
- Enables matrix testing across Go 1.21, 1.22, 1.23"
```

---

## Task 1: Fix Bug #1 — Integer Division in CalculateCompletionRate

**Files:**
- Modify: `internal/service/service.go:172`
- Test: `internal/service/service_test.go` (existing tests already detect this)

- [ ] **Step 1: Run failing test to confirm bug**

```bash
go test ./internal/service -run TestCalculateCompletionRate -v
```

Expected: FAIL — "BUG TERDETEKSI — CalculateCompletionRate() = 0.00, want 33.33"

- [ ] **Step 2: Fix integer division**

Edit `internal/service/service.go` line 172:

```go
// BEFORE (bug):
return float64(completed/len(tasks)) * 100

// AFTER (fix):
return float64(completed) / float64(len(tasks)) * 100
```

- [ ] **Step 3: Run test to verify fix**

```bash
go test ./internal/service -run TestCalculateCompletionRate -v
```

Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add internal/service/service.go
git commit -m "fix: correct integer division in CalculateCompletionRate

Bug: float64(completed/len(tasks)) always truncated to 0
Fix: cast to float64 before division"
```

---

## Task 2: Fix Bug #2 — Inverted Status Filter

**Files:**
- Modify: `internal/repository/memory.go:58`
- Modify: `internal/repository/postgres.go:113`
- Test: `internal/repository/memory_test.go` and `internal/repository/postgres_test.go` (existing tests detect this)

- [ ] **Step 1: Run failing memory test**

```bash
go test ./internal/repository -run TestFindByStatus_HanyaTodo -v
```

Expected: FAIL — "BUG TERDETEKSI — FindByStatus(todo) = 1 task, want 2"

- [ ] **Step 2: Fix memory repository filter**

Edit `internal/repository/memory.go` line 58:

```go
// BEFORE (bug):
if t.Status != status { // BUG: seharusnya == bukan !=

// AFTER (fix):
if t.Status == status {
```

- [ ] **Step 3: Fix postgres repository filter**

Edit `internal/repository/postgres.go` line 113:

```go
// BEFORE (bug):
FROM tasks WHERE status != $1 ORDER BY created_at DESC`, // BUG: != seharusnya =

// AFTER (fix):
FROM tasks WHERE status = $1 ORDER BY created_at DESC`,
```

- [ ] **Step 4: Run memory tests**

```bash
go test ./internal/repository -run TestFindByStatus -v
```

Expected: PASS

- [ ] **Step 5: Run postgres integration tests**

```bash
make db-up
export DATABASE_URL=postgres://taskflow:taskflow_secret@localhost:5432/taskflow?sslmode=disable
go test -tags=integration ./internal/repository -run TestPostgres_FindByStatus -v
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add internal/repository/memory.go internal/repository/postgres.go
git commit -m "fix: correct inverted status filter in FindByStatus

Bug: != operator returned tasks NOT matching the status
Fix: changed to == in memory.go and = in postgres.go"
```

---

## Task 3: Fix Bug #3 — Invalid Priority "urgent"

**Files:**
- Modify: `internal/validator/validator.go:15`
- Test: `internal/validator/validator_test.go` (existing test detects this)

- [ ] **Step 1: Run failing test**

```bash
go test ./internal/validator -run TestIsValidPriority -v
```

Expected: FAIL — "BUG TERDETEKSI — IsValidPriority(\"urgent\") = true, want false"

- [ ] **Step 2: Remove invalid priority**

Edit `internal/validator/validator.go` line 15:

```go
// BEFORE (bug):
"urgent": true, // BUG: "urgent" seharusnya tidak ada di sini

// AFTER (fix): remove the line entirely
```

The map should only contain: "low", "medium", "high"

- [ ] **Step 3: Run test to verify**

```bash
go test ./internal/validator -run TestIsValidPriority -v
```

Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add internal/validator/validator.go
git commit -m "fix: remove invalid 'urgent' from valid priorities

Bug: 'urgent' was incorrectly accepted as valid priority
Fix: removed from valid priority map"
```

---

## Task 4: Add New Test Cases (Minimum 2)

**Files:**
- Modify: `internal/service/service_test.go`
- Modify: `internal/repository/memory_test.go`

- [ ] **Step 1: Add TestGetStats_CompletionRate to service_test.go**

Add at the end of `internal/service/service_test.go` (after line 263):

```go
func TestGetStats_CompletionRate(t *testing.T) {
	svc := newSvc()

	// Create 4 tasks: 1 done, 2 todo, 1 in_progress
	svc.Create(model.CreateTaskRequest{Title: "Done 1", Status: model.StatusDone})
	svc.Create(model.CreateTaskRequest{Title: "Todo 1"})
	svc.Create(model.CreateTaskRequest{Title: "Todo 2"})
	svc.Create(model.CreateTaskRequest{Title: "InProgress 1", Status: model.StatusInProgress})

	// Update first task to done
	tasks, _ := svc.GetAll("")
	for _, task := range tasks {
		if task.Title == "Done 1" {
			done := model.StatusDone
			svc.Update(task.ID, model.UpdateTaskRequest{Status: &done})
		}
	}

	stats, err := svc.GetStats()
	if err != nil {
		t.Fatalf("GetStats() error = %v", err)
	}
	if stats.Total != 4 {
		t.Errorf("Total = %d, want 4", stats.Total)
	}
	// 1 done out of 4 = 25%
	if stats.CompletionRate != 25.0 {
		t.Errorf("CompletionRate = %.2f, want 25.0", stats.CompletionRate)
	}
}
```

- [ ] **Step 2: Add TestCount_AfterDelete to memory_test.go**

Add at the end of `internal/repository/memory_test.go` (after line 162):

```go
func TestCount_AfterDelete(t *testing.T) {
	r := newRepo(t)
	saveTask(t, r, "c1", "Task 1", model.StatusTodo)
	saveTask(t, r, "c2", "Task 2", model.StatusDone)
	saveTask(t, r, "c3", "Task 3", model.StatusInProgress)

	count, _ := r.Count()
	if count != 3 {
		t.Errorf("Count = %d, want 3", count)
	}

	// Delete one
	r.Delete("c2")
	count, _ = r.Count()
	if count != 2 {
		t.Errorf("Count after delete = %d, want 2", count)
	}

	// Delete non-existent
	r.Delete("non-existent")
	count, _ = r.Count()
	if count != 2 {
		t.Errorf("Count after deleting non-existent = %d, want 2", count)
	}
}
```

- [ ] **Step 3: Run all tests to verify new tests pass**

```bash
go test ./... -v
```

Expected: All PASS

- [ ] **Step 4: Check coverage**

```bash
go test ./... -coverprofile=coverage.out -covermode=atomic
go tool cover -func=coverage.out
```

Expected: Coverage ≥ 75% overall

- [ ] **Step 5: Commit**

```bash
git add internal/service/service_test.go internal/repository/memory_test.go
git commit -m "test: add GetStats completion rate and Count after delete tests

- TestGetStats_CompletionRate: validates completion rate calculation
- TestCount_AfterDelete: validates count accuracy after deletes"
```

---

## Task 5: Create GitHub Actions CI Pipeline with Go Matrix Testing

> **Kelompok 2 Focus:** Matrix testing across Go 1.21, 1.22, 1.23. The `ci` job runs in parallel for each Go version, ensuring backward compatibility.

**Files:**
- Create: `.github/workflows/ci-cd.yml`

- [ ] **Step 1: Create workflow directory**

```bash
mkdir -p .github/workflows
```

- [ ] **Step 2: Write CI/CD workflow with Go matrix**

Create `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── CI: Vet + Test + Coverage (Matrix: Go 1.21, 1.22, 1.23) ────────────────
  ci:
    name: CI - Go ${{ matrix.go-version }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        go-version: ['1.21', '1.22', '1.23']
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: taskflow
          POSTGRES_PASSWORD: taskflow_secret
          POSTGRES_DB: taskflow
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}

      - name: go vet
        run: go vet ./...

      - name: Unit test + race detector
        run: go test -race ./... -timeout 30s

      - name: Integration test
        env:
          DATABASE_URL: postgres://taskflow:taskflow_secret@localhost:5432/taskflow?sslmode=disable
        run: go test -tags=integration -race ./... -timeout 60s

      - name: Coverage check
        run: |
          go test ./... -coverprofile=coverage.out -covermode=atomic
          go tool cover -func=coverage.out
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          echo "Total coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 75" | bc -l) )); then
            echo "❌ Coverage $COVERAGE% is below 75%"
            exit 1
          fi
          echo "✅ Coverage $COVERAGE% meets requirement"

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report-go-${{ matrix.go-version }}
          path: coverage.out

      - name: Build binary
        run: |
          CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o bin/taskflow-api ./cmd/server

      - name: Upload binary artifact
        uses: actions/upload-artifact@v4
        with:
          name: taskflow-api-binary-go-${{ matrix.go-version }}
          path: bin/taskflow-api

  # ── Security Scan ───────────────────────────────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: ci
    if: always() && needs.ci.result == 'success'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Run govulncheck (SCA)
        run: |
          govulncheck -json ./... > govulncheck-report.json || true
          govulncheck ./... || true

      - name: Install gosec
        run: go install github.com/securego/gosec/v2/cmd/gosec@latest

      - name: Run gosec (SAST)
        run: |
          gosec -fmt json -out gosec-report.json ./... || true
          gosec ./... || true

      - name: Upload security reports
        uses: actions/upload-artifact@v4
        with:
          name: security-reports
          path: |
            govulncheck-report.json
            gosec-report.json

  # ── CD: Build & Push Docker Image ───────────────────────────────────────────
  cd:
    name: CD - Docker Build & Push
    runs-on: ubuntu-latest
    needs: ci
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract short SHA
        id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ steps.vars.outputs.sha_short }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Output image tag
        run: echo "Image pushed to ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ steps.vars.outputs.sha_short }}"

  # ── Smoke Test ──────────────────────────────────────────────────────────────
  smoke-test:
    name: Smoke Test
    runs-on: ubuntu-latest
    needs: cd
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: taskflow
          POSTGRES_PASSWORD: taskflow_secret
          POSTGRES_DB: taskflow
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract short SHA
        id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Pull and run image
        env:
          DATABASE_URL: postgres://taskflow:taskflow_secret@localhost:5432/taskflow?sslmode=disable
        run: |
          docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ steps.vars.outputs.sha_short }}
          docker run -d --rm \
            --name taskflow-api \
            -p 8080:8080 \
            -e DATABASE_URL="$DATABASE_URL" \
            -e PORT=8080 \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ steps.vars.outputs.sha_short }}

      - name: Wait for startup
        run: sleep 5

      - name: Smoke test /health
        run: |
          curl -sf http://localhost:8080/health || (echo "❌ Health check failed"; exit 1)
          echo "✅ /health OK"

      - name: Smoke test /api/v1/stats
        run: |
          curl -sf http://localhost:8080/api/v1/stats || (echo "❌ Stats endpoint failed"; exit 1)
          echo "✅ /api/v1/stats OK"

      - name: Cleanup container
        if: always()
        run: docker stop taskflow-api || true

  # ── Tag Stable ──────────────────────────────────────────────────────────────
  tag-stable:
    name: Tag Stable
    runs-on: ubuntu-latest
    needs: [cd, smoke-test]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push' && always() && needs.cd.result == 'success' && needs.smoke-test.result == 'success'
    permissions:
      contents: read
      packages: write

    steps:
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract short SHA
        id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Tag and push stable
        run: |
          docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ steps.vars.outputs.sha_short }}
          docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ steps.vars.outputs.sha_short }} ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:stable
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:stable
          echo "✅ Tagged stable"

  # ── Notification ────────────────────────────────────────────────────────────
  notify:
    name: Notification
    runs-on: ubuntu-latest
    needs: [ci, cd, smoke-test, tag-stable]
    if: always()

    steps:
      - name: Notify Success
        if: needs.ci.result == 'success' && needs.cd.result == 'success' && needs.smoke-test.result == 'success' && needs.tag-stable.result == 'success'
        run: |
          echo "✅ Pipeline SUCCESS"
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "Run URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          # TODO: Add Slack/Telegram webhook call here
          # Example Slack:
          # curl -X POST -H 'Content-type: application/json' \
          #   --data '{"text":"✅ TaskFlow Pipeline SUCCESS\nBranch: ${{ github.ref_name }}\nCommit: ${{ github.sha }}\n${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}' \
          #   ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Failure
        if: needs.ci.result == 'failure' || needs.cd.result == 'failure' || needs.smoke-test.result == 'failure'
        run: |
          echo "❌ Pipeline FAILED"
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "Run URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          # TODO: Add Slack/Telegram webhook call here
```

- [ ] **Step 3: Commit workflow**

```bash
git add .github/workflows/ci-cd.yml
git commit -m "ci: add GitHub Actions CI/CD pipeline with Go matrix testing

- Matrix testing: Go 1.21, 1.22, 1.23 in parallel (Kelompok 2 focus)
- Multi-job pipeline: CI → Security → CD → Smoke Test → Tag Stable → Notify
- CI: vet, unit test + race, integration test with postgres, coverage gate ≥75%
- Security: govulncheck (SCA) + gosec (SAST)
- CD: multi-stage docker build, push to GHCR with SHA tag
- Smoke test: /health and /api/v1/stats checks
- Stable tag: only updated when all stages pass"
```

---

## Task 6: Verify CI Pipeline (Red/Green Proof)

**Files:**
- None (verification task)

- [ ] **Step 1: Push to trigger pipeline**

```bash
git push origin main
```

- [ ] **Step 2: Verify green pipeline**

Go to GitHub → Actions → verify all jobs pass (green checkmarks).

**For matrix testing:** Verify 3 parallel CI jobs (Go 1.21, 1.22, 1.23) all pass.

- [ ] **Step 3: Temporarily reintroduce bug to prove pipeline blocks**

```bash
# Create temporary branch
git checkout -b test-pipeline-red

# Reintroduce bug #1 (integer division)
git revert HEAD~3 --no-edit  # or manually edit service.go

git push origin test-pipeline-red
```

- [ ] **Step 4: Verify red pipeline**

Go to GitHub → Actions → verify CI job fails (red X) due to failing test.

**For matrix testing:** Verify all 3 matrix variants (Go 1.21, 1.22, 1.23) show failure.

- [ ] **Step 5: Clean up test branch**

```bash
git checkout main
git branch -D test-pipeline-red
git push origin --delete test-pipeline-red
```

---

## Task 7: Configure Docker & Registry Settings

**Files:**
- Modify: `Makefile` (update REGISTRY default)
- Verify: `Dockerfile`

- [ ] **Step 1: Update Makefile registry default**

Edit `Makefile` line 4:

```makefile
# BEFORE:
REGISTRY ?= ghcr.io/your-username

# AFTER:
REGISTRY ?= ghcr.io/Trenttzzz
```

Replace `Trenttzzz` with actual GitHub username if different.

- [ ] **Step 2: Verify Dockerfile is correct**

`Dockerfile` should already be multi-stage. Verify:

```dockerfile
# Stage 1: Builder
FROM golang:1.22-alpine AS builder
...

# Stage 2: Runtime (scratch)
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/taskflow-api /taskflow-api
...
```

- [ ] **Step 3: Build and verify image size locally**

```bash
make docker-build
docker images ghcr.io/Trenttzzz/taskflow-api --format "{{.Repository}}:{{.Tag}} | Size: {{.Size}}"
```

Expected: Size ≤ 15 MB (typically ~8-10 MB for scratch-based Go binary)

- [ ] **Step 4: Compare with single-stage build**

Create temporary single-stage Dockerfile:

```bash
cat > Dockerfile.single << 'EOF'
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o taskflow-api ./cmd/server
EXPOSE 8080
ENTRYPOINT ["/app/taskflow-api"]
EOF

docker build -f Dockerfile.single -t taskflow-api:single .
docker images taskflow-api:single --format "Single-stage size: {{.Size}}"

# Cleanup
rm Dockerfile.single
docker rmi taskflow-api:single
```

Document comparison for report.

- [ ] **Step 5: Commit Makefile update**

```bash
git add Makefile
git commit -m "chore: update default registry to GHCR"
```

---

## Task 8: Create Rollback Documentation & Makefile Target

**Files:**
- Create: `ROLLBACK_PROCEDURE.md`
- Verify: `Makefile` (rollback target already exists)

- [ ] **Step 1: Verify rollback target in Makefile**

Check that `Makefile` lines 68-81 contain the rollback target:

```makefile
rollback:
	@test -n "$(ROLLBACK_TAG)" || (echo "❌ Set ROLLBACK_TAG=sha-xxxxx"; exit 1)
	@echo "→ Rolling back to $(REGISTRY)/$(IMAGE):$(ROLLBACK_TAG)"
	docker pull $(REGISTRY)/$(IMAGE):$(ROLLBACK_TAG)
	docker stop taskflow-api 2>/dev/null || true
	docker run -d --rm \
	  --name taskflow-api \
	  -p 8080:8080 \
	  -e DATABASE_URL=$(DATABASE_URL) \
	  $(REGISTRY)/$(IMAGE):$(ROLLBACK_TAG)
	@echo "⏳ Waiting for server..."
	@sleep 5
	curl -sf http://localhost:8080/health || (echo "❌ Health check failed!"; exit 1)
	@echo "✅ Rollback successful to $(ROLLBACK_TAG)"
```

- [ ] **Step 2: Create ROLLBACK_PROCEDURE.md**

Create `ROLLBACK_PROCEDURE.md`:

```markdown
# Rollback Procedure — TaskFlow API

## When to Use
Production incident: `/api/v1/stats` returning wrong data, application crash, or any critical failure after deployment.

## Prerequisites
- Docker installed
- Access to GHCR (GitHub Container Registry)
- `DATABASE_URL` environment variable configured
- Known good image tag (e.g., previous `sha-xxxxx` or `stable`)

## Steps

### 1. Detect Problem
- Check `/health` endpoint: `curl http://localhost:8080/health`
- Check `/api/v1/stats` endpoint: `curl http://localhost:8080/api/v1/stats`
- Review application logs: `docker logs taskflow-api`

### 2. Identify Target Version
- List available images: `docker images | grep taskflow-api`
- Or check GHCR: https://github.com/Trenttzzz/taskflow-cicd-devops-testing/pkgs/container/taskflow-api
- Note the last known good SHA tag (e.g., `sha-a3f2c1d`)

### 3. Execute Rollback
```bash
export DATABASE_URL=postgres://taskflow:taskflow_secret@localhost:5432/taskflow?sslmode=disable
make rollback ROLLBACK_TAG=sha-a3f2c1d
```

### 4. Verify Rollback
- Health check: `curl http://localhost:8080/health` → should return 200
- Stats check: `curl http://localhost:8080/api/v1/stats` → should return correct data
- Application logs: `docker logs taskflow-api` → no errors

### 5. Post-Rollback
- Notify team via Slack/Telegram
- Create incident report
- Fix bug in development branch
- Deploy fix through normal CI/CD pipeline

## Emergency: Rollback to Stable
If unsure which version is good:
```bash
make rollback ROLLBACK_TAG=stable
```
Note: `stable` tag only updates when full pipeline passes.
```

- [ ] **Step 3: Test rollback locally**

```bash
# Build current image
make docker-build

# Simulate: run current image
docker run -d --rm --name taskflow-api -p 8080:8080 ghcr.io/Trenttzzz/taskflow-api:sha-$(git rev-parse --short HEAD)
sleep 5
curl http://localhost:8080/health
docker stop taskflow-api

# Test rollback to stable (or previous SHA)
make rollback ROLLBACK_TAG=stable
```

- [ ] **Step 4: Commit**

```bash
git add ROLLBACK_PROCEDURE.md
git commit -m "docs: add rollback procedure documentation"
```

---

## Task 9: Setup Pre-commit Hook for Secret Scanning

**Files:**
- Create: `.git/hooks/pre-commit` (local only, not committed)

- [ ] **Step 1: Install gitleaks locally**

```bash
# macOS
brew install gitleaks

# Or via Docker
docker pull zricethezav/gitleaks:latest
```

- [ ] **Step 2: Create pre-commit hook**

```bash
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
# Pre-commit hook: scan for secrets before allowing commit

echo "🔍 Scanning for secrets..."

# Try native gitleaks first, fallback to docker
if command -v gitleaks >/dev/null 2>&1; then
    gitleaks protect --staged --redact --verbose
else
    echo "⚠️  gitleaks not found locally, using docker..."
    docker run --rm -v "$(pwd):/path" zricethezav/gitleaks:latest protect --staged --source /path --redact --verbose
fi

if [ $? -ne 0 ]; then
    echo "❌ Secret detected! Commit blocked."
    echo "   Review the output above and remove any secrets."
    exit 1
fi

echo "✅ No secrets found."
EOF

chmod +x .git/hooks/pre-commit
```

- [ ] **Step 3: Test pre-commit hook**

```bash
# Create a test file with fake secret
echo "API_KEY=sk-test123456789" > test-secret.txt
git add test-secret.txt
git commit -m "test secret"  # Should be blocked by gitleaks

# Cleanup
rm test-secret.txt
git reset HEAD test-secret.txt
```

Expected: Commit should be blocked with "Secret detected! Commit blocked."

- [ ] **Step 4: Document in README**

Add to README.md under a new "Development Setup" section:

```markdown
## Development Setup

### Pre-commit Hook (Recommended)

To prevent accidentally committing secrets:

```bash
# Install gitleaks
brew install gitleaks

# The pre-commit hook is already configured in .git/hooks/pre-commit
# It will run automatically on every commit.
```
```

---

## Task 10: Final Integration & Verification

**Files:**
- None (integration testing)

- [ ] **Step 1: Full pipeline test — push to main**

```bash
git push origin main
```

Verify in GitHub Actions:
1. **Matrix CI jobs pass** (Go 1.21, 1.22, 1.23 — all green)
2. Security job passes (govulncheck, gosec)
3. CD job pushes image to GHCR with SHA tag
4. Smoke test job pulls image and verifies endpoints
5. Tag stable job updates `:stable` tag
6. Notify job sends success notification

- [ ] **Step 2: Verify image in GHCR**

Go to: https://github.com/Trenttzzz/taskflow-cicd-devops-testing/pkgs/container/taskflow-api

Verify:
- Image exists with `sha-xxxxx` tag
- Image exists with `stable` tag
- Can pull image: `docker pull ghcr.io/Trenttzzz/taskflow-api:stable`

- [ ] **Step 3: Test pipeline failure path**

Create a branch with a failing test:

```bash
git checkout -b test-failure
echo "func TestForceFail(t *testing.T) { t.Fatal(\"force fail\") }" >> internal/service/service_test.go
git add .
git commit -m "test: force failure to test pipeline"
git push origin test-failure
```

Verify CI job fails, CD does NOT run, notification shows failure.

Cleanup:
```bash
git checkout main
git branch -D test-failure
git push origin --delete test-failure
```

- [ ] **Step 4: Run complete test suite locally**

```bash
make vet
make test-race
make test-integration
make test-cover
make build
```

All should pass.

---

## Spec Coverage Checklist

| Spec Requirement | Implementation Task |
|-----------------|-------------------|
| Fix 3 bugs | Tasks 1, 2, 3 |
| All tests PASS + race detector | Tasks 1-4, Step 4 of Task 10 |
| Coverage ≥ 75% | Task 4 Step 4, Task 5 CI job |
| Add ≥2 new test cases | Task 4 |
| **Matrix testing (Go 1.21, 1.22, 1.23)** | **Task 0 (routing refactor) + Task 5 Step 2 (strategy.matrix)** |
| **Go 1.21 compatibility** | **Task 0 (manual routing instead of pattern routing)** |
| CI trigger on push/PR | Task 5 Step 2 (on: push/pull_request) |
| go vet blocks pipeline | Task 5 Step 2 (vet step) |
| Unit test + race detector | Task 5 Step 2 |
| Integration test with PostgreSQL | Task 5 Step 2 (services: postgres) |
| Coverage gate ≥ 75% | Task 5 Step 2 (coverage check step) |
| Coverage artifact | Task 5 Step 2 (upload-artifact) |
| Build binary | Task 5 Step 2 |
| Multi-stage Docker build | Task 7 (verify Dockerfile) |
| Tag image with SHA | Task 5 Step 2 (CD job) |
| Push to registry | Task 5 Step 2 (CD job) |
| CD depends on CI | Task 5 Step 2 (needs: ci) |
| Smoke test /health | Task 5 Step 2 (smoke-test job) |
| Smoke test /api/v1/stats | Task 5 Step 2 (smoke-test job) |
| Pipeline fails if smoke test fails | Task 5 Step 2 (exit 1 on failure) |
| Notification success/failure | Task 5 Step 2 (notify job) |
| Tag stable conditional | Task 5 Step 2 (tag-stable job with if conditions) |
| make rollback works | Task 8 Step 1, 3 |
| Rollback procedure documented | Task 8 Step 2 |
| Security scan ≥2 categories | Task 5 Step 2 (security job: govulncheck + gosec) |
| Security scan artifacts | Task 5 Step 2 (upload-artifact for reports) |
| Pre-commit hook | Task 9 |

---

## Placeholder Scan

No placeholders found. All steps contain:
- Exact file paths
- Complete code snippets
- Exact commands with expected output
- No "TBD", "TODO", or "implement later"

---

## Type Consistency Check

| Name | Defined | Used |
|------|---------|------|
| `ROLLBACK_TAG` | Makefile line 68, Task 8 | Makefile line 70-71,77,81 |
| `REGISTRY` | Makefile line 4, Task 7 | Makefile line 51,57,63,70-71,77 |
| `IMAGE_NAME` | Workflow env | Workflow CD job tags |
| `sha_short` | Workflow steps.vars | Workflow CD/Smoke/Tag jobs |
| `go-version` | Workflow matrix | Workflow CI job setup-go, artifact names |

All references consistent.

---

## Next Steps After Plan Completion

1. Execute tasks sequentially (Task 0 → Task 1 → Task 2 → ... → Task 10)
2. Task 0 is critical: must be done first to enable matrix testing
3. Each task includes commit steps — push frequently
4. After Task 5, verify pipeline in GitHub Actions before continuing
5. Document image size comparison for final report
6. Collect screenshots: green pipeline (all 3 matrix variants), red pipeline, GHCR tags, smoke test output

**Plan saved to:** `docs/superpowers/plans/2026-05-05-taskflow-cicd-implementation.md`
