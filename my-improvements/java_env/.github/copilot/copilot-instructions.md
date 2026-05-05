# GitHub Copilot Instructions

## Priority Guidelines

When generating code for this repository:

1. Respect exact project versions (Java 25, Quarkus BOM 3.32.3, Maven 3.9.11 wrapper).
2. Prefer patterns already present in `lib-logging-quarkus`.
3. Keep architecture boundaries: observability core (`br.com.vsjr.labs.observability`) vs example app (`br.com.vsjr.labs.example`).
4. Use structured logging DSL (`LOG`) instead of ad-hoc string logging.
5. Preserve MDC/tracing lifecycle and cleanup behavior exactly as implemented.

## Technology Baseline (Detected)

- **Java**: `25` (`lib-logging-quarkus/pom.xml` → `maven.compiler.release=25`)
- **Maven wrapper**: `3.9.11` (`.mvn/wrapper/maven-wrapper.properties` → `distributionUrl`)
- **Maven Wrapper JAR**: `3.3.2` (`wrapperVersion=3.3.2`)
- **Quarkus platform BOM**: `3.32.3` (`lib-logging-quarkus/pom.xml` → `quarkus.platform.version`)
- **Packaging**: multi-module Maven aggregator (`pom` root) + Quarkus module (`lib-logging-quarkus`)

### Key Quarkus dependencies (version managed by BOM `3.32.3`)

From `lib-logging-quarkus/pom.xml`:
- `io.quarkus:quarkus-opentelemetry`
- `io.quarkus:quarkus-logging-json`
- `io.quarkus:quarkus-smallrye-jwt`
- `io.quarkus:quarkus-smallrye-context-propagation`
- `io.quarkus:quarkus-arc`
- `io.quarkus:quarkus-micrometer-registry-prometheus`
- `io.quarkus:quarkus-rest`
- `io.quarkus:quarkus-rest-jackson`
- `io.quarkus:quarkus-rest-client` and `quarkus-rest-client-jackson`
- `io.quarkus:quarkus-hibernate-validator`
- `io.quarkus:quarkus-info`
- `io.quarkus:quarkus-jfr`
- `io.quarkus:quarkus-observability-devservices`
- `io.quarkus:quarkus-scheduler`
- Test: `io.quarkus:quarkus-junit`

## Repository and Path Reality

- Real module path: `lib-logging-quarkus/` (not `logging-quarkus`).
- Real infra path: `infra/` (not `containers-dev`).
- Drift exists in older docs, e.g. `README.md` and `infra/QUICKSTART.md` still mention old paths.
- For commands and folder references, prefer `AGENTS.md` and current filesystem layout.

## Architecture Boundaries

### Modules
- Root aggregator: `pom.xml` (module: `lib-logging-quarkus`).
- Main library/runtime code: `lib-logging-quarkus/src/main/java/br/com/vsjr/labs/observability/**`.
- Demo/example usage code: `lib-logging-quarkus/src/main/java/br/com/vsjr/labs/example/**`.
- Local observability stack: `infra/`.

### Internal boundaries in `lib-logging-quarkus`

- **DSL and event model**: `dsl/LOG.java`, `dsl/LogEtapas.java`, `dsl/LogEvento.java`
- **Request context lifecycle (MDC)**: `filtro/LogContextoFiltro.java`, `context/GerenciadorContextoLog.java`
- **Tracing lifecycle (OpenTelemetry)**: `interceptor/TracingInterceptor.java`, `tracing/GerenciadorTracing.java`
- **Logging interception + optional metrics**: `interceptor/LogInterceptor.java`
- **Extension points (chain-of-responsibility)**:
  - MDC enrichers: `context/enriquecedor/EnriquecedorContexto.java`
  - Span enrichers: `tracing/enriquecedor/EnriquecedorTracing.java`
- **Security masking**: `security/SanitizadorDados.java`
- **Canonical MDC keys**: `CamposMdc.java`

## Runtime Flow (`lib-logging-quarkus`)

1. **Inbound HTTP request** enters `LogContextoFiltro.filter(request)`.
   - Resolves user ID (`anonimo` fallback).
   - Initializes MDC with `userId` + `applicationName` via `GerenciadorContextoLog.inicializar`.
   - Syncs `traceId`/`spanId` from current OTel span via `GerenciadorTracing.sincronizarMdcRequisicao`.
   - Ref: `filtro/LogContextoFiltro.java`, `context/GerenciadorContextoLog.java`, `tracing/GerenciadorTracing.java`.

2. **CDI method interception order**:
   - `@Rastreado` → `TracingInterceptor` at `@Priority(APPLICATION - 10)` creates child span first.
   - `@Logged` → `LogInterceptor` at `@Priority(APPLICATION)` enriches MDC and executes method.
   - Ref: `interceptor/TracingInterceptor.java`, `interceptor/LogInterceptor.java`, `annotations/Rastreado.java`, `annotations/Logged.java`.

3. **Business log emission** via DSL:
   - `LOG.registrando(...).em(...).[porque()][como()][comDetalhe()...].info|debug|warn|erro`
   - DSL writes structured fields to MDC (`log_*`, `detalhe_*`) only during emission, then cleans them in `finally`.
   - Sensitive detail values are sanitized by key (`****` or `[PROTEGIDO]`).
   - Ref: `dsl/LOG.java`, `security/SanitizadorDados.java`, `CamposMdc.java`.

4. **Response phase cleanup**:
   - `LogContextoFiltro.filter(request,response)` calls `gerenciador.limpar()` (`MDC.clear()`).
   - Prevents MDC leakage across reused worker threads.
   - Ref: `filtro/LogContextoFiltro.java`, `context/GerenciadorContextoLog.java`.

## Coding Patterns to Follow

- Package naming is Portuguese-domain style (`observability`, `enriquecedor`, `rastreado`, etc.).
- Constructor injection is preferred for CDI beans (no field injection in core classes).
- Use `record` and `sealed interface` patterns where already used:
  - `LogEvento` (record), `GerenciadorTracing.ContextoSpan` (record)
  - DSL contracts in `LogEtapas` (sealed interfaces)
- Keep constants centralized (`CamposMdc`, `ValoresPadrao`); avoid duplicated string literals for canonical keys.
- Keep method names and log DSL methods in Portuguese style consistent with existing APIs.

## Logging Patterns (Required)

- Prefer DSL logs over direct logger calls for business events.
  - Good: `LOG.registrando(...).em(...).info();`
  - Ref: `example/rest/HelloService.java`, `example/rest/HelloworldResource.java`
- Use `.comDetalhe("key", value)` for extra fields; do not append raw JSON in message strings.
- Never hardcode canonical MDC keys; use `CamposMdc.<...>.chave()`.
- Preserve MDC cleanup in `finally` paths (already required in `LOG.emitir`, `LogInterceptor`, `LogContextoFiltro`).

## Error-Handling Patterns (Required)

- Preserve original business exception propagation from interceptors:
  - `LogInterceptor.interceptar` rethrows `Exception` and `Error`, wraps unknown `Throwable` in `RuntimeException`.
- Infrastructure side-effects (metrics/tracing finalization) are isolated:
  - metrics registration failures log `warn` and do not break business flow (`LogInterceptor`).
  - span close failures log `erro` in `TracingInterceptor` finally block.
- Use `erroERelanca` when logging + rethrowing the same exception in one fluent call.
  - Ref: `dsl/LOG.java`, `example/rest/HelloService.java`.

## Testing Patterns to Follow

- Test framework: JUnit 5 + Quarkus test harness.
- Integration tests use `@QuarkusTest`; config overrides use `@TestProfile` + `QuarkusTestProfile`.
  - Ref: `interceptor/LogInterceptorMetricsIntegrationTest.java`, `interceptor/MetricsEnabledTestProfile.java`
- Assert metrics by explicit tags from `CamposMdc`.
- Include both enabled and disabled metrics scenarios.
  - Ref: `LogInterceptorMetricsIntegrationTest.java`, `LogInterceptorMetricsDisabledIntegrationTest.java`
- Unit tests for DSL behavior capture JBoss log records with custom `ExtHandler`.
  - Ref: `dsl/LOGTest.java`, `dsl/LogEventoTest.java`

## Configuration Patterns

From `src/main/resources/application.properties`:
- JSON logging enabled globally: `quarkus.log.json=true`
- OTel OTLP protocol: `grpc`; propagators: `tracecontext,baggage`
- Dev OTLP endpoint: `http://localhost:4317`
- Prod OTLP endpoint: `http://otel-collector:4317`
- Micrometer disabled by default; enabled selectively in test profile.
- MDC propagation enabled: `quarkus.arc.context-propagation.mdc=true`

## Copilot Do/Don’t for this Repo

### Do
- Generate code compatible with Java 25 and Quarkus 3.32.3 APIs.
- Keep interceptors, filters, DSL, and managers separated by current boundaries.
- Use `@Logged` + `@Rastreado` patterns and current priority behavior.
- Use existing Portuguese DSL and naming style.

### Don’t
- Don’t reference or generate paths using `logging-quarkus/` or `containers-dev/` as canonical locations.
- Don’t introduce alternate logging frameworks/patterns not present in this repo.
- Don’t bypass `SanitizadorDados` for sensitive details.
- Don’t add raw MDC key literals for canonical fields already defined in `CamposMdc`.

