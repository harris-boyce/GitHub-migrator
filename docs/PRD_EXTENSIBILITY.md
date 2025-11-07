# Product Requirements Document: Multi-Platform Discovery and Analysis Extensibility

**Document Version:** 1.0.0
**Last Updated:** 2025-11-07
**Status:** Draft
**Target Audience:** Engineering, Product, Spec-kit Agentic Development

---

## Executive Summary

This PRD defines the architecture, requirements, and implementation process for extending the GitHub-migrator's discovery and analysis capabilities to support additional git hosting platforms (e.g., Azure DevOps, GitLab, Bitbucket Cloud). The current system is designed with extensibility in mind through a **Provider abstraction pattern**, which enables discovery and analysis of repositories from multiple source platforms while maintaining a consistent destination (GitHub).

### Key Objectives

1. **Extend Discovery Capabilities**: Enable repository discovery and analysis from non-GitHub platforms
2. **Maintain Architecture Integrity**: Leverage existing Provider abstraction without architectural changes
3. **Ensure Consistency**: Provide uniform analysis metrics across all platforms
4. **Enable Agentic Development**: Structure requirements to support Spec-kit implementation workflows

---

## Table of Contents

- [1. Current Architecture Overview](#1-current-architecture-overview)
  - [1.1 Provider Abstraction Layer](#11-provider-abstraction-layer)
  - [1.2 Discovery and Analysis Pipeline](#12-discovery-and-analysis-pipeline)
  - [1.3 Data Model](#13-data-model)
- [2. Extensibility Points](#2-extensibility-points)
  - [2.1 Source Provider Interface](#21-source-provider-interface)
  - [2.2 Discovery Components](#22-discovery-components)
  - [2.3 Configuration System](#23-configuration-system)
- [3. Extension Process](#3-extension-process)
  - [3.1 High-Level Workflow](#31-high-level-workflow)
  - [3.2 Implementation Phases](#32-implementation-phases)
- [4. Azure DevOps Extension Specification](#4-azure-devops-extension-specification)
  - [4.1 Provider Implementation](#41-provider-implementation)
  - [4.2 API Integration](#42-api-integration)
  - [4.3 Feature Mapping](#43-feature-mapping)
  - [4.4 Authentication](#44-authentication)
- [5. Testing and Validation](#5-testing-and-validation)
- [6. Spec-kit Development Guidelines](#6-spec-kit-development-guidelines)
- [7. Success Criteria](#7-success-criteria)
- [8. Future Considerations](#8-future-considerations)

---

## 1. Current Architecture Overview

### 1.1 Provider Abstraction Layer

The system implements a **Provider interface** (`internal/source/provider.go`) that abstracts source control system operations. This interface enables the core discovery and analysis logic to work with any git hosting platform.

#### Provider Interface Contract

```go
type Provider interface {
    // Type returns the provider type (github, gitlab, azuredevops)
    Type() ProviderType

    // Name returns a human-readable name for this provider instance
    Name() string

    // CloneRepository clones a repository to the specified directory
    CloneRepository(ctx context.Context, info RepositoryInfo, destPath string, opts CloneOptions) error

    // GetAuthenticatedCloneURL returns a clone URL with embedded credentials
    GetAuthenticatedCloneURL(cloneURL string) (string, error)

    // ValidateCredentials validates that the provider's credentials are valid
    ValidateCredentials(ctx context.Context) error

    // SupportsFeature indicates whether a specific feature is supported
    SupportsFeature(feature Feature) bool
}
```

#### Current Provider Implementations

| Provider | Implementation Status | Clone Support | Auth Method | Notes |
|----------|----------------------|---------------|-------------|-------|
| **GitHub** | ✅ Complete | ✅ Full | PAT + GitHub App | Primary platform, full feature support |
| **GitLab** | 🟡 Partial | ❌ Not implemented | oauth2:TOKEN | Skeleton only, not production-ready |
| **Azure DevOps** | 🟡 Partial | ❌ Not implemented | USERNAME:PAT | Skeleton only, URL normalization complete |

### 1.2 Discovery and Analysis Pipeline

The discovery system consists of three main components:

```
┌─────────────────────────────────────────────────────────────┐
│                    Discovery Pipeline                        │
└─────────────────────────────────────────────────────────────┘

1. COLLECTOR (internal/discovery/collector.go)
   ├── Uses Provider to clone repositories
   ├── Orchestrates parallel discovery workers
   ├── Manages per-organization discovery
   └── Coordinates Analyzer and Profiler

2. ANALYZER (internal/discovery/analyzer.go)
   ├── Platform-agnostic Git analysis (git-sizer)
   ├── Repository size calculation
   ├── LFS and submodule detection
   ├── GitHub migration limit validation
   └── Commit and file size analysis

3. PROFILER (internal/discovery/profiler.go)
   ├── Platform-specific feature detection
   ├── API-based metadata collection
   ├── Security features analysis
   └── Workflow and integration analysis
```

#### Key Insight: Separation of Concerns

- **Analyzer**: Works on cloned Git repositories (platform-agnostic)
- **Profiler**: Uses platform APIs (platform-specific)
- **Collector**: Orchestrates both using Provider abstraction

This separation enables:
- **Universal Git analysis** via git-sizer (works for all platforms)
- **Platform-specific enrichment** via API integrations
- **Consistent migration validation** regardless of source platform

### 1.3 Data Model

The `Repository` model (`internal/models/repository.go`) is designed to capture features across platforms:

**Core Git Properties** (Platform-Agnostic)
- Size, commit count, branch count
- LFS, submodules, large files
- Git-sizer metrics

**GitHub-Specific Properties** (Conditional)
- Actions workflows, GitHub Packages
- Rulesets, branch protections
- Code scanning, Dependabot

**Extensibility Pattern**
```go
// Platform features are stored as optional fields
// Fields default to false/0 if not applicable
HasActions         bool // GitHub-specific
HasPipelines       bool // Azure DevOps-specific (future)
HasGitLabCI        bool // GitLab-specific (future)
```

---

## 2. Extensibility Points

### 2.1 Source Provider Interface

**Location**: `internal/source/provider.go`

**Purpose**: Abstract source control system operations

**Extension Process**:
1. Create new provider struct (e.g., `AzureDevOpsProvider`)
2. Implement all Provider interface methods
3. Register in factory (`internal/source/factory.go`)
4. Add configuration support

**Example**: See `internal/source/github.go` for reference implementation

### 2.2 Discovery Components

**Collector** (`internal/discovery/collector.go`)
- **Extension Required**: None (uses Provider abstraction)
- **Consideration**: Ensure provider's clone operations work correctly

**Analyzer** (`internal/discovery/analyzer.go`)
- **Extension Required**: None (platform-agnostic Git analysis)
- **Works with**: Any Git repository cloned by Provider

**Profiler** (`internal/discovery/profiler.go`)
- **Extension Required**: Conditional platform-specific logic
- **Pattern**: Check provider type, call appropriate API methods
- **Example**:
```go
func (p *Profiler) ProfileFeatures(ctx context.Context, repo *models.Repository) error {
    switch p.provider.Type() {
    case source.ProviderGitHub:
        return p.profileGitHubFeatures(ctx, repo)
    case source.ProviderAzureDevOps:
        return p.profileAzureDevOpsFeatures(ctx, repo)
    // ... other providers
    }
}
```

### 2.3 Configuration System

**Location**: `internal/config/config.go`

**Source Configuration Structure**:
```go
type SourceConfig struct {
    Type         string // "github", "gitlab", "azuredevops"
    BaseURL      string // API base URL
    Token        string // Authentication token (PAT)
    Organization string // Organization name (ADO-specific)
    Username     string // Username (ADO-specific)

    // GitHub App authentication (optional)
    AppID             int64
    AppPrivateKey     string
    AppInstallationID int64
}
```

**Environment Variable Pattern**:
```bash
GHMIG_SOURCE_TYPE=azuredevops
GHMIG_SOURCE_BASE_URL=https://dev.azure.com
GHMIG_SOURCE_ORGANIZATION=my-org
GHMIG_SOURCE_TOKEN=<pat>
```

---

## 3. Extension Process

### 3.1 High-Level Workflow

```
┌────────────────────────────────────────────────────────────────┐
│          Adding Support for a New Git Platform                 │
└────────────────────────────────────────────────────────────────┘

PHASE 1: Provider Implementation
  └─> Implement Provider interface
  └─> Add authentication logic
  └─> Implement repository cloning
  └─> Add credential validation

PHASE 2: API Integration (Optional but Recommended)
  └─> Research platform API structure
  └─> Identify equivalent features
  └─> Create API client/wrapper
  └─> Map to Repository model fields

PHASE 3: Discovery Integration
  └─> Register provider in factory
  └─> Add configuration options
  └─> Extend Profiler for platform-specific features
  └─> Update documentation

PHASE 4: Testing and Validation
  └─> Unit tests for Provider
  └─> Integration tests with real repositories
  └─> Validate feature detection accuracy
  └─> Performance benchmarking
```

### 3.2 Implementation Phases

#### Phase 1: Core Provider Implementation

**Objective**: Enable basic repository cloning and authentication

**Tasks**:
1. **Create Provider struct**
   - File: `internal/source/<platform>.go`
   - Implement all Provider interface methods
   - Handle platform-specific authentication

2. **Implement CloneRepository**
   - Support all CloneOptions (shallow, bare, LFS, submodules)
   - Handle authentication in clone URLs
   - Implement error handling and cleanup

3. **Implement GetAuthenticatedCloneURL**
   - Format: Platform-specific (e.g., ADO uses `USERNAME:PAT@dev.azure.com`)
   - Security: Ensure tokens are not logged

4. **Implement ValidateCredentials**
   - Use platform API to verify token validity
   - Return ErrAuthenticationFailed for invalid credentials

5. **Implement SupportsFeature**
   - Return true/false for each Feature constant
   - Document feature equivalents in comments

**Deliverables**:
- Provider implementation file
- Unit tests with >80% coverage
- Integration test with real platform instance

#### Phase 2: API Client Development

**Objective**: Enable platform-specific feature detection

**Tasks**:
1. **Research Platform API**
   - Identify REST/GraphQL endpoints
   - Document authentication methods
   - Map API resources to Repository fields

2. **Create API Client**
   - Consider using official SDK if available
   - Implement rate limiting and retry logic
   - Handle pagination for list operations

3. **Map Features to Repository Model**
   - Identify feature equivalents (e.g., Pipelines → Actions)
   - Add new fields to Repository model if needed
   - Document mapping in comments

**Example Feature Mapping** (Azure DevOps → GitHub):

| Azure DevOps | GitHub | Repository Field | Notes |
|--------------|--------|------------------|-------|
| Pipelines | Actions | `HasPipelines` | New field needed |
| Wiki | Wiki | `HasWiki` | Direct mapping |
| Artifacts | Packages | `HasPackages` | Similar concept |
| Board Work Items | Issues | `IssueCount` | Similar concept |
| Branch Policies | Branch Protections | `BranchProtections` | Similar concept |

4. **Implement Profiler Extensions**
   - Add platform-specific profiling methods
   - Call from `ProfileFeatures` based on provider type
   - Handle API errors gracefully

**Deliverables**:
- API client library or wrapper
- Feature mapping documentation
- Profiler extensions with tests

#### Phase 3: Configuration and Factory Integration

**Objective**: Make the new provider accessible via configuration

**Tasks**:
1. **Update Factory** (`internal/source/factory.go`)
   ```go
   case "azuredevops", "ado":
       if cfg.Organization == "" {
           return nil, fmt.Errorf("organization is required for Azure DevOps")
       }
       return NewAzureDevOpsProvider(cfg.Organization, cfg.Token, cfg.Username)
   ```

2. **Add Configuration Examples**
   - Create `configs/config.azuredevops.example.yaml`
   - Add environment variable examples to README
   - Document required vs optional fields

3. **Update Validation**
   - Ensure required fields are validated at startup
   - Provide helpful error messages
   - Add configuration migration if needed

**Deliverables**:
- Updated factory with new provider case
- Example configuration files
- Updated configuration documentation

#### Phase 4: Testing and Documentation

**Objective**: Ensure quality and usability

**Tasks**:
1. **Unit Testing**
   - Provider interface implementation
   - API client methods
   - Feature detection logic

2. **Integration Testing**
   - Clone real repositories from platform
   - Validate git-sizer analysis
   - Verify feature detection accuracy

3. **End-to-End Testing**
   - Full discovery workflow
   - Large organization discovery
   - Error handling and recovery

4. **Documentation**
   - Update IMPLEMENTATION_GUIDE.md
   - Add platform-specific setup guide
   - Document feature mapping and limitations

**Deliverables**:
- Comprehensive test suite
- Platform-specific documentation
- Updated architecture diagrams

---

## 4. Azure DevOps Extension Specification

This section provides a detailed specification for implementing Azure DevOps support as a reference example.

### 4.1 Provider Implementation

**File**: `internal/source/azuredevops.go` (partial implementation exists)

**Current Status**: Skeleton with URL normalization, needs full implementation

**Required Methods**:

#### CloneRepository
```go
func (p *AzureDevOpsProvider) CloneRepository(ctx context.Context, info RepositoryInfo, destPath string, opts CloneOptions) error {
    // 1. Get authenticated clone URL
    authURL, err := p.GetAuthenticatedCloneURL(info.CloneURL)
    if err != nil {
        return fmt.Errorf("failed to get authenticated URL: %w", err)
    }

    // 2. Build git clone command with options
    args := []string{"clone"}
    if opts.Shallow {
        args = append(args, "--depth=1")
    }
    if opts.Bare {
        args = append(args, "--bare")
    }
    if !opts.IncludeSubmodules {
        args = append(args, "--no-recurse-submodules")
    }
    args = append(args, authURL, destPath)

    // 3. Execute git clone
    cmd := exec.CommandContext(ctx, "git", args...)
    var stderr bytes.Buffer
    cmd.Stderr = &stderr
    cmd.Env = append(cmd.Env, "GIT_TERMINAL_PROMPT=0")

    if err := cmd.Run(); err != nil {
        sanitizedErr := sanitizeGitError(stderr.String(), p.token)
        return fmt.Errorf("%w: %s", ErrCloneFailed, sanitizedErr)
    }

    // 4. Handle LFS if requested
    if opts.IncludeLFS {
        return p.fetchLFSObjects(ctx, destPath)
    }

    return nil
}
```

#### ValidateCredentials
```go
func (p *AzureDevOpsProvider) ValidateCredentials(ctx context.Context) error {
    // Use Azure DevOps REST API to validate credentials
    // GET https://dev.azure.com/{organization}/_apis/connectionData
    // This endpoint returns user/org info if authenticated

    client := &http.Client{Timeout: 10 * time.Second}
    url := fmt.Sprintf("https://dev.azure.com/%s/_apis/connectionData", p.organization)

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return err
    }

    // Azure DevOps uses Basic Auth with PAT
    req.SetBasicAuth(p.username, p.token)
    req.Header.Set("Accept", "application/json")

    resp, err := client.Do(req)
    if err != nil {
        return fmt.Errorf("failed to validate credentials: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == 401 || resp.StatusCode == 403 {
        return fmt.Errorf("%w: invalid token or insufficient permissions", ErrAuthenticationFailed)
    }

    if resp.StatusCode != 200 {
        return fmt.Errorf("unexpected response: %d", resp.StatusCode)
    }

    return nil
}
```

### 4.2 API Integration

**Azure DevOps REST API Overview**:
- Base URL: `https://dev.azure.com/{organization}/_apis/`
- API Version: `api-version=7.1` (latest as of 2024)
- Authentication: Basic Auth with PAT or OAuth

**Key Endpoints for Discovery**:

| Purpose | Endpoint | Response |
|---------|----------|----------|
| List Projects | `/projects` | Array of projects |
| List Repos | `/{project}/_apis/git/repositories` | Array of repositories |
| Get Repo | `/{project}/_apis/git/repositories/{repoId}` | Repository details |
| List Pipelines | `/{project}/_apis/pipelines` | Array of pipelines |
| List Policies | `/{project}/_apis/policy/configurations` | Branch policies |
| List Work Items | `/{project}/_apis/wit/workitems` | Work items (issues) |

**Recommended Approach**: Create Azure DevOps API client

```go
// File: internal/azuredevops/client.go

type Client struct {
    baseURL      string
    organization string
    token        string
    username     string
    httpClient   *http.Client
}

func NewClient(org, token, username string) *Client {
    return &Client{
        baseURL:      "https://dev.azure.com",
        organization: org,
        token:        token,
        username:     username,
        httpClient:   &http.Client{Timeout: 30 * time.Second},
    }
}

func (c *Client) ListProjects(ctx context.Context) ([]Project, error) {
    // Implementation
}

func (c *Client) ListRepositories(ctx context.Context, projectName string) ([]Repository, error) {
    // Implementation
}

func (c *Client) GetRepository(ctx context.Context, projectName, repoID string) (*Repository, error) {
    // Implementation
}

func (c *Client) ListPipelines(ctx context.Context, projectName string) ([]Pipeline, error) {
    // Implementation
}
```

### 4.3 Feature Mapping

**Azure DevOps → Repository Model Mapping**:

```go
// Proposed additional fields in models.Repository
type Repository struct {
    // ... existing fields ...

    // Azure DevOps specific (optional)
    HasPipelines       bool   `json:"has_pipelines"`
    PipelineCount      int    `json:"pipeline_count"`
    HasBoardWorkItems  bool   `json:"has_board_work_items"`
    WorkItemCount      int    `json:"work_item_count"`
    HasArtifacts       bool   `json:"has_artifacts"`
    PolicyCount        int    `json:"policy_count"` // Branch policies
}
```

**Feature Detection Logic**:

```go
// File: internal/discovery/profiler_azuredevops.go

func (p *Profiler) profileAzureDevOpsFeatures(ctx context.Context, repo *models.Repository) error {
    // Extract project and repo from full name
    // Azure DevOps format: "project/repository"
    parts := strings.Split(repo.FullName, "/")
    if len(parts) != 2 {
        return fmt.Errorf("invalid Azure DevOps repository name: %s", repo.FullName)
    }
    project := parts[0]
    repoName := parts[1]

    // Profile pipelines
    pipelines, err := p.adoClient.ListPipelines(ctx, project)
    if err == nil {
        // Filter pipelines that reference this repository
        repoPipelines := filterPipelinesByRepo(pipelines, repoName)
        repo.HasPipelines = len(repoPipelines) > 0
        repo.PipelineCount = len(repoPipelines)
    }

    // Profile branch policies
    policies, err := p.adoClient.ListBranchPolicies(ctx, project, repoName)
    if err == nil {
        repo.PolicyCount = len(policies)
        repo.BranchProtections = len(policies) // Map to existing field
    }

    // Profile work items (if repo has associated work items)
    workItems, err := p.adoClient.GetRepoWorkItems(ctx, project, repoName)
    if err == nil {
        repo.HasBoardWorkItems = len(workItems) > 0
        repo.WorkItemCount = len(workItems)
        // Map to IssueCount for consistency
        repo.IssueCount = len(workItems)
    }

    // Profile artifacts/packages
    hasArtifacts, err := p.adoClient.HasArtifacts(ctx, project)
    if err == nil {
        repo.HasArtifacts = hasArtifacts
        repo.HasPackages = hasArtifacts // Map to existing field
    }

    // Profile wiki
    hasWiki, err := p.adoClient.HasWiki(ctx, project)
    if err == nil {
        repo.HasWiki = hasWiki
    }

    return nil
}
```

### 4.4 Authentication

**Azure DevOps Authentication Options**:

1. **Personal Access Token (PAT)** - Recommended
   - Scopes needed: Code (Read), Build (Read), Work Items (Read), Project and Team (Read)
   - Format: Basic Auth with username (can be anything) and PAT as password

2. **OAuth 2.0** - Advanced
   - Requires app registration
   - More complex but supports user delegation

**Implementation**:
```go
// HTTP request with PAT
req.SetBasicAuth(username, token)

// Git clone URL format
// https://USERNAME:PAT@dev.azure.com/organization/project/_git/repository
```

**Security Considerations**:
- Never log tokens or include in error messages
- Use sanitization function for Git errors
- Store tokens in environment variables or secure vaults
- Validate token scopes on startup

---

## 5. Testing and Validation

### 5.1 Unit Testing

**Provider Tests** (`internal/source/<platform>_test.go`):
```go
func TestAzureDevOpsProvider_GetAuthenticatedCloneURL(t *testing.T) {
    provider, _ := NewAzureDevOpsProvider("myorg", "token123", "user")

    tests := []struct {
        name     string
        cloneURL string
        want     string
        wantErr  bool
    }{
        {
            name:     "standard URL",
            cloneURL: "https://dev.azure.com/myorg/project/_git/repo",
            want:     "https://user:token123@dev.azure.com/myorg/project/_git/repo",
            wantErr:  false,
        },
        // ... more test cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := provider.GetAuthenticatedCloneURL(tt.cloneURL)
            if (err != nil) != tt.wantErr {
                t.Errorf("GetAuthenticatedCloneURL() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if got != tt.want {
                t.Errorf("GetAuthenticatedCloneURL() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### 5.2 Integration Testing

**Repository Discovery Test**:
```go
func TestAzureDevOps_DiscoverRepository(t *testing.T) {
    // Requires real Azure DevOps credentials
    if os.Getenv("ADO_INTEGRATION_TEST") != "true" {
        t.Skip("Skipping integration test")
    }

    org := os.Getenv("ADO_ORGANIZATION")
    token := os.Getenv("ADO_TOKEN")

    provider, err := NewAzureDevOpsProvider(org, token, "")
    require.NoError(t, err)

    // Validate credentials
    err = provider.ValidateCredentials(context.Background())
    require.NoError(t, err)

    // Clone a test repository
    repoInfo := source.RepositoryInfo{
        FullName: "TestProject/TestRepo",
        CloneURL: "https://dev.azure.com/myorg/TestProject/_git/TestRepo",
    }

    tempDir, err := os.MkdirTemp("", "ado-test-*")
    require.NoError(t, err)
    defer os.RemoveAll(tempDir)

    err = provider.CloneRepository(context.Background(), repoInfo, tempDir, source.ShallowCloneOptions())
    require.NoError(t, err)

    // Verify clone
    gitDir := filepath.Join(tempDir, ".git")
    _, err = os.Stat(gitDir)
    require.NoError(t, err)
}
```

### 5.3 Feature Detection Validation

**Profiler Test**:
```go
func TestProfiler_AzureDevOpsFeatures(t *testing.T) {
    // Mock Azure DevOps API client
    mockClient := &MockADOClient{
        Pipelines: []Pipeline{
            {ID: "1", Name: "CI Pipeline"},
        },
        Policies: []Policy{
            {ID: "1", Type: "Required reviewers"},
        },
    }

    profiler := NewProfiler(nil, logger)
    profiler.adoClient = mockClient

    repo := &models.Repository{
        FullName: "MyProject/MyRepo",
    }

    err := profiler.profileAzureDevOpsFeatures(context.Background(), repo)
    require.NoError(t, err)

    assert.True(t, repo.HasPipelines)
    assert.Equal(t, 1, repo.PipelineCount)
    assert.Equal(t, 1, repo.PolicyCount)
}
```

### 5.4 Performance Testing

**Benchmark Discovery Performance**:
```go
func BenchmarkAzureDevOps_Discovery(b *testing.B) {
    // Setup
    provider, _ := NewAzureDevOpsProvider(org, token, "")

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Benchmark clone operation
        tempDir, _ := os.MkdirTemp("", "bench-*")
        _ = provider.CloneRepository(context.Background(), repoInfo, tempDir, source.ShallowCloneOptions())
        os.RemoveAll(tempDir)
    }
}
```

**Performance Targets**:
- Clone (shallow): < 30 seconds for typical repository
- Clone (full): < 2 minutes for typical repository
- API feature detection: < 5 seconds per repository
- Full discovery (100 repos): < 15 minutes with 5 workers

---

## 6. Spec-kit Development Guidelines

### 6.1 Project Structure for Agentic Development

To support Spec-kit and agentic development workflows, organize work into discrete, testable units:

**Recommended Task Breakdown**:

```
Task 1: Azure DevOps Provider Core Implementation
├── Subtask 1.1: Implement GetAuthenticatedCloneURL
├── Subtask 1.2: Implement ValidateCredentials
├── Subtask 1.3: Implement CloneRepository
├── Subtask 1.4: Implement SupportsFeature
└── Deliverable: Fully functional Provider with unit tests

Task 2: Azure DevOps API Client
├── Subtask 2.1: Create client structure and authentication
├── Subtask 2.2: Implement ListProjects
├── Subtask 2.3: Implement ListRepositories
├── Subtask 2.4: Implement feature-specific methods (pipelines, policies, etc.)
└── Deliverable: API client library with integration tests

Task 3: Profiler Extension
├── Subtask 3.1: Add Azure DevOps-specific profiling logic
├── Subtask 3.2: Implement feature mapping
├── Subtask 3.3: Add error handling and logging
└── Deliverable: Extended Profiler with feature detection

Task 4: Configuration and Integration
├── Subtask 4.1: Update factory registration
├── Subtask 4.2: Add configuration examples
├── Subtask 4.3: Update documentation
└── Deliverable: Fully integrated and documented extension
```

### 6.2 Specifications for Each Component

#### Spec 1: Provider Implementation

**Input**: Azure DevOps configuration (organization, token, username)
**Output**: Functional Provider that can clone repositories

**Acceptance Criteria**:
- [ ] Implements all Provider interface methods
- [ ] Handles authentication correctly
- [ ] Supports all CloneOptions (shallow, bare, LFS, submodules)
- [ ] Returns appropriate errors with sanitized messages
- [ ] Unit test coverage > 80%

**Dependencies**: None (uses standard library and exec package)

**Estimated Complexity**: Medium (similar to GitHubProvider)

#### Spec 2: API Client

**Input**: Azure DevOps organization and authentication
**Output**: Client library for Azure DevOps REST API

**Acceptance Criteria**:
- [ ] Authenticates successfully with PAT
- [ ] Lists projects and repositories
- [ ] Retrieves repository details
- [ ] Handles pagination correctly
- [ ] Implements rate limiting/retry logic
- [ ] Returns structured errors

**Dependencies**:
- Azure DevOps REST API documentation
- HTTP client library (standard lib)

**Estimated Complexity**: Medium-High (API comprehension required)

#### Spec 3: Feature Mapping

**Input**: Azure DevOps repository
**Output**: Repository model with populated fields

**Acceptance Criteria**:
- [ ] Maps Azure Pipelines to HasPipelines/PipelineCount
- [ ] Maps Branch Policies to BranchProtections
- [ ] Maps Work Items to IssueCount
- [ ] Maps Artifacts to HasPackages
- [ ] Handles missing/optional features gracefully
- [ ] Validates mapping accuracy with test cases

**Dependencies**:
- Azure DevOps API Client
- Repository model updates

**Estimated Complexity**: Medium (requires platform knowledge)

### 6.3 Specification Templates

**For Spec-kit Agents**, use this template structure:

````markdown
## Component Specification: [Component Name]

### Purpose
[One sentence describing what this component does]

### Inputs
- **Configuration**: [List config parameters]
- **Dependencies**: [List required services/libraries]
- **Context**: [Execution context, e.g., "Called by Collector during discovery"]

### Outputs
- **Success Case**: [Describe expected output]
- **Error Cases**: [List error scenarios and expected behavior]

### Implementation Requirements

#### Required Methods/Functions
```go
// Function signature with godoc
func MethodName(params) (returns, error) {
    // High-level algorithm
}
```

#### Error Handling
- Error scenario 1: [How to handle]
- Error scenario 2: [How to handle]

#### Testing Requirements
- Unit tests: [Specific scenarios to test]
- Integration tests: [End-to-end scenarios]
- Edge cases: [Boundary conditions]

### Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

### Success Metrics
- Performance: [e.g., "< 5 seconds per operation"]
- Reliability: [e.g., "99% success rate in integration tests"]
- Coverage: [e.g., "> 80% code coverage"]
````

### 6.4 Dependency Graph

```
┌─────────────────────────────────────────────────────────────┐
│              Azure DevOps Extension Dependencies            │
└─────────────────────────────────────────────────────────────┘

Level 1: Foundation (No dependencies)
  ├── Provider Interface (already exists)
  ├── Repository Model (already exists)
  └── Configuration System (already exists)

Level 2: Provider Implementation (depends on Level 1)
  ├── AzureDevOpsProvider struct
  ├── Authentication logic
  └── CloneRepository implementation

Level 3: API Integration (depends on Level 2)
  ├── Azure DevOps API Client
  ├── HTTP wrapper
  └── Response models

Level 4: Feature Mapping (depends on Level 3)
  ├── Profiler extensions
  ├── Feature detection logic
  └── Repository model population

Level 5: Integration (depends on Level 4)
  ├── Factory registration
  ├── Configuration updates
  └── Documentation

Level 6: Testing (depends on all levels)
  ├── Unit tests
  ├── Integration tests
  └── E2E validation
```

**Recommended Implementation Order**: Follow dependency levels (Level 1 → Level 6)

---

## 7. Success Criteria

### 7.1 Functional Requirements

| Requirement | Description | Validation Method |
|-------------|-------------|-------------------|
| **FR-1** | Provider can authenticate with Azure DevOps | Integration test with real credentials |
| **FR-2** | Provider can clone repositories (shallow & full) | Integration test with various repository types |
| **FR-3** | git-sizer analysis works on cloned repos | Unit test verifying Analyzer output |
| **FR-4** | Platform-specific features are detected | Integration test comparing API vs actual state |
| **FR-5** | Configuration supports Azure DevOps | Configuration validation test |
| **FR-6** | Error messages are clear and actionable | Manual review + user testing |
| **FR-7** | Multiple organizations are supported | Integration test with 2+ orgs |

### 7.2 Non-Functional Requirements

| Requirement | Target | Validation Method |
|-------------|--------|-------------------|
| **NFR-1: Performance** | Clone 100 repos in < 15 min (5 workers) | Benchmark test |
| **NFR-2: Reliability** | 99% success rate for valid credentials | Integration test (100 iterations) |
| **NFR-3: Security** | No token leakage in logs/errors | Security audit + code review |
| **NFR-4: Maintainability** | Code coverage > 80% | Coverage report |
| **NFR-5: Extensibility** | Adding new feature takes < 2 hours | Developer experience survey |

### 7.3 Acceptance Tests

**Test Scenario 1: Full Discovery Workflow**
```
Given: Azure DevOps organization with 10 repositories
When: Discovery is initiated with Azure DevOps provider
Then: All 10 repositories are discovered
And: Git metrics (size, commits, branches) are accurate
And: Platform features (pipelines, policies) are detected
And: Repositories are saved to database
```

**Test Scenario 2: Large Organization**
```
Given: Azure DevOps organization with 500+ repositories
When: Discovery is initiated with 5 workers
Then: Discovery completes in < 30 minutes
And: All repositories are successfully processed
And: Worker pool manages concurrency correctly
```

**Test Scenario 3: Error Handling**
```
Given: Invalid Azure DevOps credentials
When: Discovery is initiated
Then: Clear error message is displayed
And: Application does not crash
And: No sensitive information is logged
```

---

## 8. Future Considerations

### 8.1 Additional Platform Support

**Candidate Platforms** (Priority Order):
1. **Bitbucket Cloud** - Popular for enterprises
2. **Bitbucket Server (Data Center)** - On-premise alternative
3. **GitLab Self-Hosted** - Already has partial support
4. **Gitea** - Lightweight open-source option

**Estimated Effort per Platform**:
- With current architecture: 2-3 weeks per platform
- Without architecture: 6-8 weeks per platform

### 8.2 Destination Platform Extensibility

**Current State**: Destination is always GitHub (GitHub.com, GHEC, GHES)

**Future Opportunity**: Support multiple destination platforms
- Would require significant architecture changes
- Migration engine is GitHub-specific (uses GitHub Migrations API)
- Alternative: Use git-based migration (slower but universal)

**Recommendation**: Keep GitHub as destination for now, revisit if user demand exists

### 8.3 Advanced Features

#### Feature 1: Repository Filtering During Discovery
```yaml
source:
  type: azuredevops
  organization: myorg
  filters:
    project_names:
      - "ProjectA"
      - "ProjectB"
    exclude_archived: true
    min_commit_count: 10
```

#### Feature 2: Incremental Discovery
- Store discovery timestamps
- Only re-analyze changed repositories
- Reduce discovery time for large organizations

#### Feature 3: Custom Feature Detectors
- Plugin system for custom feature detection
- User-defined feature mappings
- Extensible without code changes

### 8.4 Platform Abstraction Improvements

**Potential Enhancement**: Multi-level abstraction

```
Current: Provider → Collector/Profiler
Future:  Provider → Platform Client → Feature Detector → Collector/Profiler
```

**Benefits**:
- Clearer separation of concerns
- Easier to add platform-specific features
- Better testability

**Considerations**:
- Increased complexity
- More layers to maintain
- May not be needed until 5+ platforms supported

---

## 9. Questions for Clarification

Before proceeding with implementation, the following questions should be addressed:

### 9.1 Azure DevOps Specific

1. **Project Scope**: Should we discover all projects in an organization, or allow filtering by project names?
2. **Repository Access**: Do we need to support both project-level and organization-level repositories?
3. **Authentication**: Should we support OAuth in addition to PAT, or is PAT sufficient?
4. **Feature Priority**: Which Azure DevOps features are most critical to detect? (Pipelines, Policies, Work Items, Artifacts, etc.)

### 9.2 Architecture

5. **Repository Model**: Should we add platform-specific fields, or keep a generic model? (e.g., `HasPipelines` vs `HasCI`)
6. **Profiler Design**: Should we create separate profilers per platform, or keep one profiler with conditional logic?
7. **API Client**: Should we use the official Azure DevOps SDK, or build a custom lightweight client?

### 9.3 User Experience

8. **Configuration**: Do users prefer YAML config files, environment variables, or both equally?
9. **Migration Planning**: Do users want to see which Azure DevOps features won't migrate to GitHub?
10. **Reporting**: Should we generate platform-specific migration readiness reports?

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **Provider** | Abstraction layer for source control platforms (GitHub, Azure DevOps, GitLab, etc.) |
| **Collector** | Component that orchestrates repository discovery across an organization |
| **Analyzer** | Component that performs platform-agnostic Git analysis using git-sizer |
| **Profiler** | Component that performs platform-specific feature detection via APIs |
| **git-sizer** | Open-source tool for analyzing Git repository metrics |
| **PAT** | Personal Access Token - credential for API authentication |
| **Repository Model** | Data structure representing a discovered repository with all its metadata |
| **Discovery** | Process of finding and analyzing repositories from a source platform |
| **Migration** | Process of moving repositories to GitHub (separate from discovery) |
| **Feature Mapping** | Translation of platform-specific features to common Repository model fields |
| **CloneOptions** | Configuration for how to clone a repository (shallow, bare, LFS, submodules) |

---

## Appendix A: Reference Implementation Checklist

Use this checklist when implementing support for a new platform:

### Phase 1: Provider Core
- [ ] Create `internal/source/<platform>.go`
- [ ] Define provider struct with required fields
- [ ] Implement `Type()` method
- [ ] Implement `Name()` method
- [ ] Implement `GetAuthenticatedCloneURL()` with tests
- [ ] Implement `ValidateCredentials()` with tests
- [ ] Implement `CloneRepository()` with all CloneOptions support
- [ ] Implement `SupportsFeature()` for all Feature constants
- [ ] Write unit tests achieving >80% coverage
- [ ] Write integration test with real platform instance

### Phase 2: API Client
- [ ] Research platform API documentation
- [ ] Create `internal/<platform>/client.go`
- [ ] Implement authentication
- [ ] Implement list projects/repositories
- [ ] Implement get repository details
- [ ] Implement feature-specific endpoints (CI/CD, policies, etc.)
- [ ] Add rate limiting and retry logic
- [ ] Handle pagination correctly
- [ ] Write unit tests with mocked responses
- [ ] Write integration tests with real API

### Phase 3: Feature Detection
- [ ] Document feature mapping (platform → Repository model)
- [ ] Add new fields to Repository model if needed
- [ ] Create `internal/discovery/profiler_<platform>.go`
- [ ] Implement platform-specific profiling logic
- [ ] Update `ProfileFeatures()` to call platform profiler
- [ ] Write tests validating feature detection accuracy
- [ ] Document limitations and unsupported features

### Phase 4: Configuration
- [ ] Update `internal/config/config.go` with platform-specific fields
- [ ] Update `internal/source/factory.go` to register provider
- [ ] Create `configs/config.<platform>.example.yaml`
- [ ] Add environment variable examples
- [ ] Update configuration validation
- [ ] Write configuration parsing tests

### Phase 5: Documentation
- [ ] Update IMPLEMENTATION_GUIDE.md with extension details
- [ ] Create platform-specific setup guide in `docs/`
- [ ] Document feature mapping and limitations
- [ ] Add troubleshooting section
- [ ] Update README.md with platform support notice
- [ ] Create architecture diagram showing integration

### Phase 6: Testing
- [ ] Run full unit test suite
- [ ] Run integration tests with real platform
- [ ] Perform end-to-end discovery test (10+ repos)
- [ ] Perform large organization test (100+ repos)
- [ ] Test error handling and recovery
- [ ] Benchmark performance vs targets
- [ ] Security audit (token handling, logging)

### Phase 7: Release
- [ ] Update CHANGELOG.md
- [ ] Create migration guide for existing users
- [ ] Update API documentation if endpoints changed
- [ ] Prepare release notes highlighting new platform support
- [ ] Tag release with semantic version

---

## Appendix B: Azure DevOps API Quick Reference

### Authentication
```http
GET https://dev.azure.com/{organization}/_apis/projects?api-version=7.1
Authorization: Basic base64({username}:{PAT})
```

### Key Endpoints

#### List Projects
```http
GET https://dev.azure.com/{organization}/_apis/projects?api-version=7.1
```

#### List Repositories
```http
GET https://dev.azure.com/{organization}/{project}/_apis/git/repositories?api-version=7.1
```

#### Get Repository
```http
GET https://dev.azure.com/{organization}/{project}/_apis/git/repositories/{repositoryId}?api-version=7.1
```

#### List Pipelines
```http
GET https://dev.azure.com/{organization}/{project}/_apis/pipelines?api-version=7.1
```

#### List Branch Policies
```http
GET https://dev.azure.com/{organization}/{project}/_apis/policy/configurations?repositoryId={repoId}&api-version=7.1
```

#### List Work Items (requires WIQL query)
```http
POST https://dev.azure.com/{organization}/{project}/_apis/wit/wiql?api-version=7.1
Content-Type: application/json

{
  "query": "SELECT [System.Id] FROM WorkItems WHERE [System.TeamProject] = '{project}'"
}
```

#### Get Wiki
```http
GET https://dev.azure.com/{organization}/{project}/_apis/wiki/wikis?api-version=7.1
```

#### List Artifacts (Feeds)
```http
GET https://feeds.dev.azure.com/{organization}/_apis/packaging/feeds?api-version=7.1-preview.1
```

### Rate Limiting
- Azure DevOps uses TSTUs (Team Services Time Units)
- Default limit: 200 TSTUs per user per second
- When exceeded: HTTP 429 with Retry-After header
- Recommended: Implement exponential backoff

---

## Appendix C: Comparison Matrix

### Platform Feature Comparison

| Feature | GitHub | Azure DevOps | GitLab | Bitbucket | Notes |
|---------|--------|--------------|--------|-----------|-------|
| **Git Hosting** | ✅ | ✅ | ✅ | ✅ | All support standard Git |
| **LFS** | ✅ | ✅ | ✅ | ✅ | Large file support |
| **Submodules** | ✅ | ✅ | ✅ | ✅ | Git submodules |
| **Wiki** | ✅ | ✅ | ✅ | ✅ | Documentation |
| **CI/CD** | Actions | Pipelines | CI/CD | Pipelines | Different implementations |
| **Package Registry** | Packages | Artifacts | Package Registry | - | Artifact storage |
| **Branch Protection** | Branch Rules | Policies | Protected Branches | Branch Permissions | Access control |
| **Issues** | Issues | Work Items | Issues | Issues | Task tracking |
| **Pull Requests** | PRs | PRs | Merge Requests | PRs | Code review |
| **Code Search** | ✅ | ✅ | ✅ | Limited | Search capabilities |
| **API** | REST + GraphQL | REST | REST + GraphQL | REST | API access |

### Migration Compatibility

| Feature | Migrates to GitHub? | Notes |
|---------|---------------------|-------|
| **Git History** | ✅ Full | Complete commit history |
| **Branches** | ✅ Full | All branches migrate |
| **Tags** | ✅ Full | All tags migrate |
| **LFS Objects** | ✅ Optional | Can exclude for size |
| **Pipelines → Actions** | ❌ Manual | Requires conversion |
| **Work Items → Issues** | ❌ Manual | Different structure |
| **Policies → Rules** | ❌ Manual | Must reconfigure |
| **Wiki** | ✅ Partial | May need adjustment |

---

**Document End**

This PRD provides a comprehensive foundation for extending the GitHub-migrator to support additional platforms. For implementation, follow the phased approach outlined in Section 3, using the specifications in Section 6 for Spec-kit-driven development.

**Next Steps**:
1. Review and approve this PRD
2. Create detailed Spec-kit specifications for Phase 1 tasks
3. Begin Azure DevOps Provider implementation
4. Iterate based on learnings

**Questions or Feedback**: Please direct to the engineering team or create a discussion in the repository.
