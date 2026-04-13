# Go-Kratos Skill Design Document

**Date:** 2026-04-13
**Status:** Approved for Implementation
**Scope:** Comprehensive Kratos microservice framework development assistant

---

## Overview

Create a skill to assist developers working with go-kratos (a Go microservice framework). The skill provides guidance on API definition, architecture patterns, configuration, HTTP customization, security, middleware, and troubleshooting.

---

## Skill Structure

### Directory Layout

```
go-kratos/
├── SKILL.md                        # Core decision guide + quick reference (~300 lines)
└── references/
    ├── proto-api-design.md         # API definition, buf, error codes
    ├── architecture.md             # Layered architecture, fx, type conversion
    ├── configuration.md            # Configuration, startup hooks
    ├── http-customization.md       # HTTP customization, WebSocket, file handling
    ├── security-auth.md            # JWT, Casbin, idempotency, data masking
    ├── middleware-logging.md       # Middleware, log filtering
    ├── troubleshooting.md          # Common issues, debugging tools (statsviz)
    └── advanced-features.md        # MCP support, other extension features
```

### Progressive Disclosure

- **SKILL.md**: Always in context (~300 lines), contains decision tree and quick code patterns
- **References**: Loaded as needed (200-300 lines each), domain-specific deep guides

---

## SKILL.md Content Plan

### 1. Frontmatter

```yaml
name: go-kratos
description: Go-Kratos microservice framework development assistant. TRIGGER when: user mentions kratos, protobuf API definition, Go microservice layered architecture, HTTP/gRPC service configuration, middleware development, JWT/Casbin auth, buf generation tool, proto validate, WebSocket/file upload. Trigger even without explicit "kratos" mention when these topics arise.
```

### 2. Quick Decision Tree

```
What are you doing?
├─ "Defining new API / proto file" → references/proto-api-design.md
├─ "Developing Service/Biz/Data layers" → references/architecture.md
├─ "Configuring project / startup params" → references/configuration.md
├─ "Customizing HTTP response / WebSocket / files" → references/http-customization.md
├─ "Adding auth / JWT / Casbin" → references/security-auth.md
├─ "Writing middleware / log handling" → references/middleware-logging.md
├─ "Encountering issues / errors" → references/troubleshooting.md
└─ "MCP / advanced extensions" → references/advanced-features.md
```

### 3. Quick Code Patterns Index

Essential snippets for immediate use without deep reference:

- Proto definition template (with buf.validate)
- Service layer skeleton
- fx Module registration pattern
- ResponseEncoder example
- JWT payload extraction from Context
- Middleware skeleton

### 4. Project Context Detection

Guide to check project structure (buf.yaml, internal/service, internal/biz) to confirm Kratos project.

### 5. Integration with Go Best Practices

When to combine with golang-patterns/effective-go skills.

---

## Reference Files Content Plan

### proto-api-design.md (~250 lines)

- HTTP proto definition syntax (path variables, pattern matching, pagination)
- buf.validate field validation rules
- buf.gen.yaml configuration explanation
- Common pitfalls (route override order, `additional_bindings`)
- OpenAPI documentation generation
- Error code design standards (Alibaba/Sina format reference)

### architecture.md (~250 lines)

- Layer responsibilities: Service (protocol conversion) → Biz (business logic) → Data (data access)
- fx dependency injection patterns (Module organization)
- Repository interface definition
- Dependency flow and boundaries
- Custom route inheriting middleware
- pb → struct type conversion (copier + copieroptpb)

### configuration.md (~200 lines)

- Bootstrap configuration definition (using proto)
- buf.validate pre-startup validation
- Multi-format config file extension (toml example)
- Startup hooks (BeforeStart/AfterStart/BeforeStop/AfterStop)
- Task dependency and reordering (processor interface)

### http-customization.md (~250 lines)

- ResponseEncoder / ErrorEncoder customization
- Zero-value field handling (`EmitUnpopulated`, `anypb.New`)
- File upload pattern (UploadHandlerWithMiddleware generic)
- File download / redirect (Redirector interface)
- WebSocket integration (gorilla/websocket)
- CORS configuration (gorilla/handlers or rs/cors)
- Static file serving (embed.FS)

### security-auth.md (~250 lines)

- JWT payload extraction from Context
- Casbin integration (keyMatch3 for Kratos URLs, watcher refresh)
- Idempotency middleware (x-idempotent token)
- Data masking rules (masking struct tags)

### middleware-logging.md (~200 lines)

- Middleware skeleton code
- Context → Transport/Header extraction
- Log filtering (FilterLevel, FilterKey, FilterValue, FilterFunc)
- Recovery configuration
- Middleware chain ordering suggestions

### troubleshooting.md (~200 lines)

- API route override issues
- Zero-value fields ignored in response
- Partial update with full-field validation
- Custom Encoder causes enum as string
- HTTP proto unsupported scenarios
- Debugging tools (statsviz, fgtrace)

### advanced-features.md (~150 lines)

- MCP server integration (kratos/contrib/transport/mcp)
- MCP tool definition and handler example
- Future extension placeholder

---

## Trigger Rules

### Should Trigger

- User explicitly mentions "kratos", "go-kratos"
- User in Go project discussing: protobuf API definition, HTTP/gRPC service, microservice layers
- User asks about: buf, proto validate, fx dependency injection, Casbin auth, JWT Context
- User encounters: route override, zero-value fields, Kratos error codes, custom ResponseEncoder
- Project directory contains: `buf.yaml`, `buf.gen.yaml`, `internal/service`, `internal/biz`

### Should Not Trigger

- Plain Go code without microservice framework
- Other Go frameworks (gin, echo, fiber)
- Pure gRPC without Kratos HTTP gateway

---

## Success Criteria

1. Developer can quickly find relevant guidance via decision tree
2. Quick code patterns cover 80% of common tasks without deep reference
3. References provide comprehensive coverage of core modules (1-6)
4. Skill correctly triggers for Kratos-related queries
5. Skill integrates well with existing Go best-practice skills

---

## Implementation Notes

- Source material: `/home/czyt/Documents/blog/content/post/go-kratos-usage-memo.md`
- Template project: `/home/czyt/code/go/kratos`
- Reference existing Go skills for formatting consistency
- Use imperative form in instructions
- Include "why" explanations for important patterns