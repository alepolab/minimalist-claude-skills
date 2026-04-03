# Comprehensive Code Review Instructions

> This file is read by Claude Code Review at runtime.
> It provides the full review checklist replacing CodeRabbit.

---

## PR Summary Format (Post as top-level PR comment via `gh pr comment`)

Every review MUST start with a single top-level comment containing:

### 📋 Review Summary

**What this PR does**: 2-3 sentence plain-English summary of the change and its purpose.

**Risk Assessment**: Rate as `🟢 Low` | `🟡 Medium` | `🔴 High` based on:
- Scope: How many files/systems touched?
- Blast radius: What breaks if this is wrong? (auth, payments, data integrity = high)
- Reversibility: Can this be rolled back easily?

**Changes Walkthrough**:

| File | Change Summary |
|------|---------------|
| `path/to/file.ts` | One-line description of what changed |

**Data Flow** (for non-trivial PRs): Describe how data moves through changed code paths.
Example: `HTTP Request → AuthMiddleware → UserService.update() → PostgreSQL → AuditLog.emit() → Response`

**Key Findings**: Bullet the top 3-5 issues with severity emoji.

---

## Inline Comment Format

Post each finding as an inline comment on the exact line. Format:

```
<SEVERITY_EMOJI> <Severity> [<Category>] — <Short title>

<Description>

**Impact**: <What happens if unfixed>

**Suggested fix**:
```<language>
<corrected code>
```
```

**Category tags**: [Bug], [Security], [Performance], [Error Handling], [Concurrency],
[API Design], [Testing], [Code Quality], [Documentation], [Breaking Change],
[Observability], [Dependency], [Infrastructure], [Input Validation]

---

## Review Dimensions

### 1. Correctness & Logic Bugs

- Off-by-one errors in loops, slices, pagination, boundary conditions
- Null/undefined dereferences, especially after optional chaining or map/filter
- Incorrect boolean logic (De Morgan violations, short-circuit side effects)
- Wrong comparison operators (== vs ===, < vs <=)
- Unreachable code and dead branches
- Variable shadowing that changes behavior
- Missing return statements in conditional branches
- Swapped or mismatched arguments in function calls
- Integer overflow/underflow, floating-point comparison issues
- State mutations in unexpected places (especially in functional code)
- Incorrect async/await (missing await, unhandled promise rejections, fire-and-forget)
- TOCTOU (time-of-check-to-time-of-use) bugs
- Incorrect regex patterns (missing anchors, greedy vs lazy, escape issues)
- Copy-paste errors (wrong variable names carried from copied code)

### 2. Security Vulnerabilities (OWASP/CWE)

Flag ALL security issues as 🔴 Important [Security].

**Injection (CWE-89, CWE-79, CWE-78)**:
- SQL injection: string concat/interpolation in queries → require parameterized queries
- XSS: unescaped user input in HTML, dangerouslySetInnerHTML, template literals
- Command injection: user input in exec(), system(), child_process, subprocess
- LDAP/XPath/header injection

**Auth & Access Control (CWE-287, CWE-862, CWE-863)**:
- Missing authentication on endpoints
- Missing authorization (can user X access resource Y?)
- Hardcoded credentials, API keys, tokens in source code
- Weak password hashing (MD5, SHA1 → require bcrypt/argon2/scrypt)
- JWT: missing signature verification, alg:none, secrets in payload, no expiry
- Session fixation, missing invalidation on logout
- IDOR (insecure direct object references)

**Data Exposure (CWE-200, CWE-532)**:
- Sensitive data in logs (passwords, tokens, PII, credit cards)
- Stack traces or internal paths leaked to users in error responses
- Secrets committed to version control (even in tests)
- PII sent to third-party services without consent

**SSRF & CSRF (CWE-918, CWE-352)**:
- User-controlled URLs passed to server-side HTTP clients without allowlist
- Missing CSRF tokens on state-changing endpoints
- Missing SameSite cookie attribute

**Insecure Deserialization (CWE-502)**:
- pickle.loads(), yaml.load() without SafeLoader, eval(), Function() on untrusted input
- JSON.parse on untrusted input without schema validation

**Cryptography**:
- Deprecated algorithms (DES, RC4, MD5 for hashing)
- Hardcoded/reused IVs and nonces
- Disabled TLS verification (verify=False, rejectUnauthorized: false)
- Custom crypto instead of battle-tested libraries

**Infrastructure Security**:
- Overly permissive CORS (Access-Control-Allow-Origin: * on authenticated endpoints)
- Missing security headers (CSP, HSTS, X-Content-Type-Options, X-Frame-Options)
- Docker containers running as root
- Exposed debug/admin endpoints in production
- Unencrypted secrets in env vars or config files

### 3. Input Validation

- Missing validation on API request bodies, query params, path params
- No schema validation (Zod, Joi, Pydantic, JSON Schema, etc.)
- Unbounded input accepted (no max length on strings, no max on arrays/files)
- Missing file upload validation (type, size, content-type sniffing)
- ReDoS-vulnerable regex patterns (catastrophic backtracking)
- Missing encoding/decoding (URL encoding, HTML entities, Unicode normalization)
- Trusting client-side validation without server-side re-validation

### 4. Error Handling & Resilience

- Empty catch blocks that silently swallow errors
- Catching generic Exception/Error instead of specific types
- Errors caught but not re-thrown or logged
- Missing timeouts on network calls, DB queries, external service requests
- Missing retry logic with exponential backoff for transient failures
- Resource leaks: unclosed files, DB connections, HTTP clients, streams
- Missing finally blocks or context managers for cleanup
- Missing circuit breaker for external dependencies
- Wrong HTTP status codes (200 on failure, 500 on client error)
- Missing graceful degradation when dependencies are unavailable
- Unhandled promise rejections or missing .catch() on promise chains

### 5. Performance

- **N+1 queries**: loops executing a query per iteration instead of batching
- **Missing indexes**: queries filtering/sorting on unindexed columns
- **Unbounded queries**: SELECT * without LIMIT, missing pagination on list endpoints
- **Memory leaks**: event listeners not removed, growing caches without eviction, closures over large objects
- **Unnecessary allocations**: object creation in hot loops, string concat in loops
- **Algorithm complexity**: O(n²) or worse when O(n log n) or O(n) exists
- **Missing caching**: repeated expensive computations, redundant API/DB calls
- **Blocking I/O in async context**: sync file/network ops in async code paths
- **Large payloads**: returning full objects when few fields needed, missing field selection
- **Missing compression**: large responses without gzip/brotli
- **No connection pooling**: new DB/HTTP connections per request
- **Missing parallelization**: sequential awaits that could be Promise.all() or concurrent futures
- **Frontend**: oversized bundles, missing lazy loading, unoptimized images
- **Database**: full table scans, missing EXPLAIN analysis for complex queries

### 6. Concurrency & Thread Safety

- Race conditions in shared mutable state
- Missing locks/mutexes on critical sections
- Deadlock potential (lock ordering violations)
- TOCTOU bugs (check-then-act without atomicity)
- Unsafe concurrent collection access
- Missing atomic operations where needed
- Thread-unsafe singleton/lazy initialization
- Goroutine/async task leaks (missing cancellation, unbounded spawning)
- Incorrect use of volatile/atomic types

### 7. API Design & Contracts

- Breaking changes to existing API contracts (removed fields, type changes, renamed endpoints)
- Missing API versioning for breaking changes
- Inconsistent naming conventions (camelCase vs snake_case mixing)
- Non-RESTful patterns (verbs in URLs, wrong HTTP methods for operations)
- Missing rate limiting on public or expensive endpoints
- Missing pagination on list endpoints
- Inconsistent error response format across endpoints
- Missing request/response validation schemas
- Backward-incompatible changes to event/message schemas
- Internal implementation details leaked in public API responses
- Missing idempotency keys for non-idempotent operations

### 8. Testing

- Missing tests for new code paths (every new public function/endpoint needs tests)
- Missing edge case tests: empty, null, boundary, max-size, Unicode, special chars
- Missing negative tests: invalid input, unauthorized access, failure scenarios
- Flaky test indicators: time-dependent, sleep/delay, shared mutable state, non-deterministic order
- Weak assertions: only checking no exception, missing value/state assertions
- Test isolation: tests modifying shared state, missing setup/teardown
- Over-mocking: mocking internals instead of interfaces, hiding real bugs
- Missing integration tests for critical paths (auth, payments, data migrations)
- If PR fixes a bug → where is the regression test?
- Test data using hardcoded IDs that could collide

### 9. Code Quality & Maintainability

- Cyclomatic complexity: functions with >10-15 branches → suggest extraction
- Function length: >50 lines → suggest breaking into smaller functions
- Parameter count: >4-5 params → suggest parameter objects/builders
- Naming: abbreviations, single-letter vars (outside loop indices), misleading names
- SOLID violations: god classes, functions doing too many things, tight coupling
- DRY violations: duplicated behavior across files (not just text — logic duplication)
- Magic numbers/strings: unexplained literals → named constants
- Deep nesting: >3 levels → suggest early returns or method extraction
- Code smells: feature envy, shotgun surgery, data clumps, primitive obsession
- Dead code: unused imports, unreachable branches, commented-out code
- Overly clever one-liners that sacrifice readability
- Missing type annotations where types add clarity (TS, Python, Go)
- Inconsistent patterns within the same module without justification

### 10. Dependency Analysis

- Dependencies with known CVEs
- Unpinned versions that could break (*, latest, loose ranges in lockfile-less projects)
- Unnecessary new deps when stdlib or existing deps suffice
- License incompatibility (GPL in MIT/Apache projects)
- Deprecated packages being newly introduced
- Dependencies from unknown/low-trust publishers
- Missing lockfile updates when manifest files change

### 11. Documentation

- Missing doc comments on public functions, classes, interfaces
- Outdated comments that contradict the code
- Missing README updates when public API or setup changes
- Missing changelog/migration notes for user-facing changes
- TODO/FIXME/HACK without linked issue tracker reference
- Comments explaining "what" instead of "why"

### 12. Logging & Observability

- Missing logging on error paths and catch blocks
- Logging sensitive data (passwords, tokens, PII)
- No structured logging (use key-value pairs, not string interpolation)
- Missing correlation/request IDs for distributed tracing
- Missing metrics for business-critical operations
- Missing audit trail for security actions (login, permission changes, data deletion)
- Log levels misused (INFO for errors, DEBUG for important events)
- Missing health check endpoints for new services

### 13. Breaking Changes Detection

Flag as 🔴 Important [Breaking Change]:
- DB schema changes not backward-compatible (column renames, type changes, drops)
- API response shape changes (removed fields, type changes)
- Config key renames or removals
- Event/message schema changes in async systems
- Changed function signatures in shared libraries
- Removed or renamed exports/public API
- Changed default values with behavioral impact
- Environment variable renames

### 14. Infrastructure & DevOps

When PRs touch infra files:
- Dockerfiles: multi-stage builds, non-root user, pinned base images, .dockerignore
- CI/CD: no hardcoded secrets, pinned action versions, caching configured
- K8s manifests: resource limits, readiness/liveness probes, no latest tag
- Terraform/IaC: no hardcoded secrets, state locking, modules versioned
- DB migrations: reversible, idempotent, no data loss, tested with existing data

### 15. Language-Specific Checks

**JavaScript/TypeScript**:
- `any` type usage without justification
- Missing strict mode in tsconfig
- Prototype pollution vectors
- eval(), innerHTML, document.write()
- Missing error boundaries in React components
- Uncontrolled re-renders, missing deps in useEffect/useMemo/useCallback
- Improper key usage in list rendering

**Python**:
- Mutable default arguments (def foo(bar=[]))
- Using `is` for value comparison instead of `==`
- Bare `except:` clauses
- os.system() instead of subprocess.run()
- Missing type hints on public functions
- Global state mutation

**Java/Kotlin**:
- Resource leaks (missing try-with-resources)
- Unnecessary !! operators in Kotlin
- Missing @Override annotations
- Mutable collections exposed from APIs
- Thread safety of Spring beans (prototype vs singleton scope)

**Go**:
- Unchecked errors (_ = someFunc())
- Goroutine leaks (missing context cancellation)
- Data races (missing sync primitives)
- Deferred close on writable files (missed flush errors)
- Nil pointer after type assertions

**SQL**:
- Missing WHERE on UPDATE/DELETE
- Cartesian joins (missing JOIN condition)
- SELECT * in production queries
- Non-sargable WHERE (functions on indexed columns)
- Missing indexes for query patterns in changed code

---

## Review Behavior Rules

### Incremental Review
- Focus on NEWLY ADDED OR MODIFIED lines only.
- Do not comment on unchanged code unless it directly interacts with changes.
- Analyze the CUMULATIVE diff, not individual commits.
- If re-reviewing after fixes, acknowledge fixes and focus on remaining/new issues.
- For 🟣 Pre-existing: only flag when the PR makes them more likely to trigger.

### Skip — Do Not Comment On
- Auto-generated files (protobuf, GraphQL codegen, OpenAPI clients, ORM auto-migrations)
- Lock files (package-lock.json, yarn.lock, poetry.lock, go.sum, Cargo.lock) unless security concerns
- Vendored/third-party code (vendor/, third_party/, node_modules/)
- Binary files, images, fonts
- Pure formatting changes (whitespace, import ordering) — leave to linters
- Minified/bundled JS/CSS
- .gitignore, .editorconfig, IDE configs — unless security issues
- Test fixture data files — unless they contain secrets or PII
- CHANGELOG entries and version bumps — unless factually wrong

### Interaction
- When a developer replies with a question, answer clearly with code examples.
- If developer says "I'll fix later" with a linked issue, accept it.
- Do NOT re-raise already-acknowledged issues.
- If asked to generate a test or docs, provide complete, runnable code.
- Do not argue subjective style if developer pushes back. Only insist on correctness, security, and data integrity.

### Severity Decision Guide

**🔴 Important** — Will cause incorrect behavior, data loss, security vulnerability, or production outage:
- Any security vulnerability
- Logic bugs that produce wrong results
- Data loss or corruption risks
- Breaking changes without migration path
- Resource leaks that degrade production over time
- Missing error handling that will crash the service

**🟡 Nit** — Worth fixing but not blocking:
- Missing tests (when code is otherwise correct)
- Suboptimal performance (not a production risk yet)
- Naming improvements
- Documentation gaps
- Code quality suggestions
- Minor style inconsistencies not caught by linters

**🟣 Pre-existing** — Bug in untouched code that interacts with this PR:
- PR adds a caller to a function that already has a bug
- PR's changes make a pre-existing race condition more likely
- PR depends on a function whose error handling was already broken

