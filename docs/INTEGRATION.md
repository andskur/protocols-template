# Integration Guide

This guide explains how to integrate this protocols repository with your microservices.

## Table of Contents

- [Integration Methods](#integration-methods)
- [Git Subtree Integration](#git-subtree-integration-recommended)
- [Git Submodule Integration](#git-submodule-integration)
- [Why not `go get`](#why-not-go-get)
- [Version Pinning](#version-pinning)
- [Development Workflow](#development-workflow)
- [Troubleshooting](#troubleshooting)

## Integration Methods

Two supported approaches for integrating protocols. In both, the **consuming service generates the
Go code** with its own module path (see [Generating Code](#generating-code)); this repository
contains only `.proto` files and has no `go.mod` (see [Why not `go get`](#why-not-go-get)).

| Method | Pros | Cons | Use Case |
|--------|------|------|----------|
| **Git Subtree** | • Simple workflow<br>• No submodule complexity<br>• Files committed to repo | • Larger repo size<br>• Updates require subtree pull | **Recommended** for most projects |
| **Git Submodule** | • Small repo size<br>• Clear dependency tracking | • More complex workflow<br>• Easy to forget updates | Advanced users, multiple proto repos |

## Git Subtree Integration (Recommended)

Git subtree embeds the protocols repository into your microservice repository.

### Initial Setup

In your microservice repository:

```bash
# Add protocols repository as a subtree
make proto-setup PROTO_REPO=https://github.com/andskur/protocols-template.git
```

This:
1. Adds protocols repository as a remote
2. Pulls protocols into `protocols/` directory
3. Commits to your repository

### Manual Setup

If your microservice doesn't have the Makefile targets:

```bash
# Add remote
git remote add protocols https://github.com/andskur/protocols-template.git

# Pull as subtree
git subtree add \
  --prefix protocols \
  protocols main \
  --squash

# Commit
git commit -m "Add protocols via git subtree"
```

### Directory Structure

After setup, your microservice will have:

```
your-microservice/
├── cmd/
├── internal/
├── protocols/              # ← Protocols repository
│   ├── common/v1/
│   │   ├── status.proto
│   │   ├── pagination.proto
│   │   └── ...
│   ├── user/v1/
│   │   └── user.proto
│   ├── buf.yaml
│   └── README.md
├── go.mod
└── README.md
```

### Updating Protocols

Pull latest changes from protocols repository:

```bash
make proto-update
```

Or manually:

```bash
git subtree pull \
  --prefix protocols \
  protocols main \
  --squash
```

### Generating Code

The `go_package` options in these `.proto` files point at this repository
(`github.com/andskur/protocols-template/...`). That path is not importable from your service, so
generate with Buf **managed mode** and override the prefix with your service's module path. Put a
`buf.gen.yaml` in your service root (outside `protocols/`, which is replaced on every subtree pull):

```yaml
# buf.gen.yaml (service root)
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/your-org/your-service/protocols   # <your module>/protocols
plugins:
  - local: protoc-gen-go          # pinned via `go install ...@vX.Y.Z`
    out: protocols
    opt: [paths=source_relative]
  - local: protoc-gen-go-grpc
    out: protocols
    opt: [paths=source_relative]
```

```bash
buf generate protocols --template buf.gen.yaml
```

Generated files go to `protocols/<package>/v1/*.pb.go` and are imported as
`github.com/your-org/your-service/protocols/<package>/v1`. Do not commit them back to this repository.

[go-microservice-template](https://github.com/andskur/go-microservice-template) ships exactly this
setup: `make proto-setup`, `make proto-update` and `make generate`.

### Example Makefile Targets

Add to your microservice's `Makefile` (with the `buf.gen.yaml` above):

```makefile
PROTO_DIR ?= protocols
PROTO_REPO ?= https://github.com/andskur/protocols-template.git
PROTO_BRANCH ?= main

.PHONY: proto-setup
proto-setup:
	git subtree add --prefix=$(PROTO_DIR) $(PROTO_REPO) $(PROTO_BRANCH) --squash

.PHONY: proto-update
proto-update:
	git subtree pull --prefix=$(PROTO_DIR) $(PROTO_REPO) $(PROTO_BRANCH) --squash

.PHONY: proto-generate
proto-generate:
	buf generate $(PROTO_DIR) --template buf.gen.yaml
```

## Git Submodule Integration

Git submodules link to external repositories without copying files.

### Initial Setup

```bash
# Add protocols as submodule
git submodule add \
  https://github.com/andskur/protocols-template.git \
  protocols

# Initialize and update
git submodule update --init --recursive

# Commit
git commit -m "Add protocols submodule"
```

### Cloning Repository with Submodules

```bash
# Clone with submodules
git clone --recursive https://github.com/yourorg/your-microservice.git

# Or if already cloned
git submodule update --init --recursive
```

### Updating Submodule

```bash
# Update to latest
cd protocols
git pull origin main
cd ..
git add protocols
git commit -m "Update protocols submodule"
```

### Generating Code

Use the same service-side `buf.gen.yaml` as for subtrees:

```bash
buf generate protocols --template buf.gen.yaml
```

### Pros and Cons

**Pros:**
- Smaller repository size
- Clear version tracking
- Easy to see which version is used

**Cons:**
- More complex workflow
- Team members may forget to update submodules
- CI/CD needs special handling

### Makefile Targets for Submodules

```makefile
.PHONY: proto-init
proto-init:
	@git submodule update --init --recursive

.PHONY: proto-update
proto-update:
	@git submodule update --remote protocols
	@git add protocols
	@git commit -m "Update protocols submodule" || true

.PHONY: proto-generate
proto-generate:
	@cd protocols && buf generate
```

## Why not `go get`

This repository intentionally has **no `go.mod` and no committed generated code**:

- When it is added to a service as a git subtree, a `go.mod` at its root would turn `protocols/`
  into a nested Go module. The service's own module then no longer contains
  `<module>/protocols/...` and builds fail with
  `no required module provides package <module>/protocols/user/v1`.
- Committed `.pb.go` files would carry this repository's import paths and would be overwritten in
  the service by the `go_package_prefix` regeneration on every build, so every `subtree pull` would
  conflict with local changes.

If you need a `go get`-able Go package, publish generated code from a separate repository (or a
`gen/go` submodule with its own tags) built from these protos.

## Version Pinning

### Pin to Specific Version

#### Git Subtree

```bash
# Pull specific tag
git subtree pull \
  --prefix protocols \
  protocols v1.2.0 \
  --squash
```

#### Git Submodule

```bash
cd protocols
git checkout v1.2.0
cd ..
git add protocols
git commit -m "Pin protocols to v1.2.0"
```

### Version Tracking

Document which version you're using in `README.md`:

```markdown
## Protocol Dependencies

- protocols-template: v1.2.0
```

## Development Workflow

### Full Integration Workflow

1. **Initial Setup**
   ```bash
   make proto-setup PROTO_REPO=https://github.com/andskur/protocols-template.git
   ```

2. **Generate Code**
   ```bash
   make generate        # go-microservice-template; or: buf generate protocols --template buf.gen.yaml
   ```

3. **Import in Your Code**
   ```go
   import userv1 "your-module/protocols/user/v1"
   
   func GetUser(ctx context.Context, id string) (*userv1.User, error) {
       // Implementation
   }
   ```

4. **Build and Test**
   ```bash
   go build ./...
   go test ./...
   ```

5. **Update Protocols (When Needed)**
   ```bash
   make proto-update
   make generate
   go mod tidy
   ```

### Example: Using User Service

1. **Import Generated Code**
   ```go
   package main
   
   import (
       "context"
       "log"
       
       userv1 "your-module/protocols/user/v1"
       "google.golang.org/grpc"
   )
   
   func main() {
       conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
       if err != nil {
           log.Fatal(err)
       }
       defer conn.Close()
       
       client := userv1.NewUserServiceClient(conn)
       
       resp, err := client.GetUser(context.Background(), &userv1.GetUserRequest{
           Id: "user-123",
       })
       if err != nil {
           log.Fatal(err)
       }
       
       log.Printf("User: %s (%s)", resp.User.Name, resp.User.Email)
   }
   ```

2. **Implement Server**
   ```go
   package server
   
   import (
       "context"
       
       userv1 "your-module/protocols/user/v1"
       commonv1 "your-module/protocols/common/v1"
       "google.golang.org/protobuf/types/known/timestamppb"
   )
   
   type UserServer struct {
       userv1.UnimplementedUserServiceServer
   }
   
   func (s *UserServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
       user := &userv1.User{
           Id:        req.Id,
           Email:     "[email protected]",
           Name:      "John Doe",
           Status:    commonv1.CommonStatus_COMMON_STATUS_ACTIVE,
           CreatedAt: timestamppb.Now(),
           UpdatedAt: timestamppb.Now(),
       }
       
       return &userv1.GetUserResponse{User: user}, nil
   }
   ```

### CI/CD Integration

#### GitHub Actions Example

```yaml
name: Build

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # Pin actions by commit SHA in real workflows.
      - uses: actions/checkout@v5
        with:
          submodules: recursive  # If using submodules
      
      - uses: actions/setup-go@v6
        with:
          go-version: '1.26'
      
      - name: Install Buf and plugins (pinned)
        run: |
          go install github.com/bufbuild/buf/cmd/buf@v1.75.0
          go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.36.12
          go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.6.2
      
      - name: Generate protocols
        run: |
          buf generate protocols --template buf.gen.yaml
      
      - name: Build
        run: go build ./...
      
      - name: Test
        run: go test ./...
```

## Troubleshooting

### Subtree Pull Conflicts

```
CONFLICT (content): Merge conflict in protocols/...
```

**Solution:**

```bash
# Resolve conflicts manually
git mergetool

# Or abort and try with --squash
git merge --abort
git subtree pull --prefix protocols protocols main --squash
```

### Submodule Not Initialized

```
fatal: not a git repository: protocols/.git
```

**Solution:**

```bash
git submodule update --init --recursive
```

### Import Path Issues

```go
import "your-module/protocols/user/v1"  // Not found
```

**Solution:** Check that the `go_package_prefix` in your service's `buf.gen.yaml` is
`<module from go.mod>/protocols`, that `protocols/` contains no `go.mod`, and regenerate.

### Generated Files Missing

```
user/v1/user.pb.go: no such file or directory
```

**Solution:**

```bash
buf generate protocols --template buf.gen.yaml
```

### Buf Not Found in CI

```
buf: command not found
```

**Solution:** Install in CI:

```yaml
- name: Install Buf
  run: go install github.com/bufbuild/buf/cmd/buf@v1.75.0
```

## Best Practices

1. **Pin to Versions:** Use specific versions (tags) in production

2. **Document Integration:** Add to README which method you use

3. **Automate Generation:** Include in build scripts/CI

4. **Test After Updates:** Run tests after updating protocols

5. **Version in README:** Document which protocol version you're using

6. **Use Make Targets:** Standardize commands across team

## Next Steps

- Read [PROTOCOL_DEVELOPMENT.md](PROTOCOL_DEVELOPMENT.md) for contributing protocols
- See [BUF_GUIDE.md](BUF_GUIDE.md) for Buf usage
- Check [VERSIONING.md](VERSIONING.md) for version management
