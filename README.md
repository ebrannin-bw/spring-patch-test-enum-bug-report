# Spring Data REST — JSON Patch `test` operation fails for enum fields

Minimal reproducer for a bug in Spring Data REST where a JSON Patch `test`
operation on an enum-typed field always fails, even when the field value matches
the expected value.

## The bug

When a PATCH request body like this is sent:

```json
[
  { "op": "test",    "path": "/status", "value": "NEW" },
  { "op": "replace", "path": "/name",   "value": "Updated Widget" }
]
```

Spring Data REST returns **400** with `PatchException: Test against path '/status' failed`,
even though the `status` field is `NEW`.

### Root cause

`JsonPatchPatchConverter.valueFromJsonNode` returns a raw `String` for textual
JSON nodes instead of wrapping them in a `JsonLateObjectEvaluator`.
`TestOperation.perform` then calls `ObjectUtils.nullSafeEquals(String, Enum)`,
which is always `false`.

The fix would be a one-line change: treat textual nodes the same as object/array
nodes — wrap them in `JsonLateObjectEvaluator` so the deferred conversion to the
correct target type happens at comparison time.

### Why a bean override won't work

`valueFromJsonNode` is `private` in every version of `spring-data-rest-webmvc`
(3.x through 5.x). `JsonPatchPatchConverter` is instantiated directly with `new`
inside the package-private `JsonPatchHandler` — it is not a Spring bean and
cannot be replaced via the context.

## Structure

```
common/             Shared entity, repository, and abstract test class
spring-boot-41/     Runs the test suite against Spring Boot 4.1.x
spring-boot-40/     Runs the test suite against Spring Boot 4.0.x (SDR 5.0.x / Jackson 3.x)
spring-boot-35/     Spring Boot 3.5.x — confirms bug is not a 4.x regression
spring-boot-34/     Spring Boot 3.4.x
spring-boot-33/     Spring Boot 3.3.x
spring-boot-32/     Spring Boot 3.2.x
spring-boot-31/     Spring Boot 3.1.x
spring-boot-30/     Spring Boot 3.0.x
```

Each version module declares `spring-boot-starter-parent` as its own Maven
parent (not the root aggregator pom), so each module resolves the full Spring
Boot BOM for its target version independently.

## Running

```bash
cd tools/spring-patch-test-enum-bug-report
mvn package -fae
```

Expected result for every module:

```
Tests run: 9, Failures: 3, Errors: 0, Skipped: 0
```

The three failing tests are `testStatusReplaceName`, `testStatusReplaceAmount`,
and `testStatusReplaceStatus` (the same-field case). They are written to expect a
2xx response and will remain red until the framework is fixed. The six passing
tests cover non-enum `test` operations and confirm the rest of JSON Patch works
correctly.

## Version matrix

| Module | Spring Boot | Spring Data REST | Jackson |
|---|---|---|---|
| spring-boot-41 | 4.1.0  | 5.1.x | 3.x (`tools.jackson`) |
| spring-boot-40 | 4.0.7  | 5.0.x | 3.x (`tools.jackson`) |
| spring-boot-35 | 3.5.15 | 4.5.x | 2.x |
| spring-boot-34 | 3.4.13 | 4.4.x | 2.x |
| spring-boot-33 | 3.3.13 | 4.3.x | 2.x |
| spring-boot-32 | 3.2.12 | 4.2.x | 2.x |
| spring-boot-31 | 3.1.12 | 4.1.x | 2.x |
| spring-boot-30 | 3.0.13 | 4.0.x | 2.x |

The 3.x modules confirm the bug is not a regression introduced in Spring Boot 4.x — it has been present since at least 3.0.

### Spring Boot 4.x notes

Spring Boot 4.0 introduced `spring-boot-starter-webmvc-test` as a separate
dependency for `MockMvc` auto-configuration (no longer bundled in
`spring-boot-starter-test`). `@AutoConfigureMockMvc` also moved to a new
package: `org.springframework.boot.webmvc.test.autoconfigure`. Jackson 3.x
changed its groupId from `com.fasterxml.jackson` to `tools.jackson`.
