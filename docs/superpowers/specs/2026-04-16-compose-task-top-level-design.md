# Tasks as a top-level element of the Compose format

**Date**: 2026-04-16
**Status**: Design validated, pending user review before implementation plan

## Context and motivation

The current Compose format exposes several top-level elements: `services`, `networks`, `volumes`, `secrets`, `configs`, `models`. We want to introduce a new concept: **tasks** (`tasks`), which represent containers executed in a one-shot fashion, similar to a `docker run`.

A task differs from a service by the absence of any orchestration semantics: no replicas, no continuous health probes, no restart policy tied to a long-running lifecycle. Its success or failure is measured solely by its **exit code**.

A task definition is almost identical to a service definition (same container options: image, command, environment, volumes, networks, etc.). **The central challenge of this design is to avoid duplicating the JSON schema** that describes a service (~825 lines in `schema/compose-spec.json`).

Beyond that, this design must lay the foundations so that more advanced concepts (services with `scale`/`deploy`, future sidecars, init-containers) can reuse the same descriptive core.

## Goals

1. Introduce `tasks:` as a top-level element of the Compose file.
2. Extract a common `container_spec` core from the existing `service` definition, to avoid any duplication.
3. Refactor the Go `ServiceConfig` type to compose this core through embedding, without breaking existing public accessors.
4. Provide an extensible foundation for future container types (sidecars, init-containers).
5. Not implement execution logic — compose-go remains a parser/validator. The `docker compose` CLI will implement `compose run <task>` separately.

## Non-goals (future iterations)

- Actual task execution (delegated to the CLI).
- Sidecars and init-containers.
- `service → task` dependency with a `task_completed_successfully` condition.
- `task → task` dependency.

## Architecture

### Guiding principle

We extract a named `container_spec` definition that captures every container configuration field shared by `service` and `task`. `service` becomes `container_spec` + fields specific to long-running orchestration. `task` becomes a simple alias of `container_spec`.

### Field distribution

**`container_spec` core** (shared by service, task, future types):

`image`, `build`, `command`, `entrypoint`, `environment`, `env_file`, `working_dir`, `user`, `labels`, `label_file`, `volumes`, `volumes_from`, `volume_driver`, `networks`, `network_mode`, `net`, `ports`, `expose`, `external_links`, `links`, `extra_hosts`, `dns`, `dns_opt`, `dns_search`, `hostname`, `domainname`, `mac_address`, `ipc`, `pid`, `pids_limit`, `cgroup`, `cgroup_parent`, `cap_add`, `cap_drop`, `security_opt`, `privileged`, `read_only`, `init`, `tty`, `stdin_open`, `tmpfs`, `sysctls`, `ulimits`, `devices`, `device_cgroup_rules`, `gpus`, `cpu_count`, `cpu_percent`, `cpu_period`, `cpu_quota`, `cpu_rt_period`, `cpu_rt_runtime`, `cpus`, `cpuset`, `cpu_shares`, `mem_limit`, `mem_reservation`, `mem_swappiness`, `memswap_limit`, `oom_kill_disable`, `oom_score_adj`, `blkio_config`, `shm_size`, `storage_opt`, `platform`, `runtime`, `isolation`, `stop_signal`, `stop_grace_period`, `restart`, `secrets`, `configs`, `credential_spec`, `annotations`, `attach`, `logging`, `log_driver`, `log_opt`, `models`, `pull_policy`, `post_start`, `pre_stop`, `use_api_socket`, `depends_on`, `extends`, `container_name`, `dockerfile`, `userns_mode`, `uts`, `group_add`.

**Exclusive to `service`**:

- `deploy` — cluster directives
- `scale` — multi-replica count
- `healthcheck` — liveness probes for a long-running container (for a task, we wait for its exit code)
- `develop` — continuous development workflow (watch/sync)
- `profiles` — conditional activation (a task is always invoked explicitly)

### JSON Schema

New definition in `schema/compose-spec.json`:

```jsonc
"definitions": {
  "container_spec": {
    "type": "object",
    "description": "Configuration shared by any container-based element (task, service, ...).",
    "properties": {
      "image": { "type": "string" },
      "build": { /* ... */ },
      "command": { "$ref": "#/definitions/command" },
      // ... every core field
    },
    "patternProperties": { "^x-": {} },
    "additionalProperties": false
  },

  "service": {
    "allOf": [
      { "$ref": "#/definitions/container_spec" },
      {
        "type": "object",
        "properties": {
          "deploy":      { "$ref": "#/definitions/deployment" },
          "scale":       { "type": ["integer", "string"] },
          "healthcheck": { "$ref": "#/definitions/healthcheck" },
          "develop":     { "$ref": "#/definitions/development" },
          "profiles":    { "type": "array", "items": { "type": "string" } }
        }
      }
    ]
  },

  "task": {
    "$ref": "#/definitions/container_spec"
  }
}
```

At the document root level:

```jsonc
"properties": {
  // ... existing
  "tasks": {
    "type": "object",
    "patternProperties": {
      "^[a-zA-Z0-9._-]+$": { "$ref": "#/definitions/task" }
    },
    "additionalProperties": false,
    "description": "One-shot tasks triggered explicitly via `compose run`."
  }
}
```

#### Tricky point: `allOf` + `additionalProperties: false`

JSON Schema draft-07 evaluates `additionalProperties` independently in each sub-schema of an `allOf`. For the composition to work:

- `container_spec` carries `additionalProperties: false` and lists its core fields.
- The second sub-schema of `service`'s `allOf` (the one that adds `deploy`/`scale`/etc.) **MUST NOT** carry `additionalProperties: false`, otherwise it would reject every core field.

**Risk**: the `gojsonschema` validator used by the project could have implementation subtleties. The **first step of the implementation plan** consists of verifying this behavior with an isolated test before committing to the full refactor. If the behavior is incorrect, a plan B is considered (see "Risks" section).

### Go types

New file `types/container.go`:

```go
// ContainerConfig describes the configuration of a container, shared by
// ServiceConfig, TaskConfig, and any future container type.
type ContainerConfig struct {
    Name         string       `yaml:"name,omitempty" json:"-"`
    Annotations  Mapping      `yaml:"annotations,omitempty" json:"annotations,omitempty"`
    Attach       *bool        `yaml:"attach,omitempty" json:"attach,omitempty"`
    Build        *BuildConfig `yaml:"build,omitempty" json:"build,omitempty"`
    // ... every core field, copied from the former ServiceConfig
    Extensions   Extensions   `yaml:"#extensions,inline,omitempty" json:"-"`
}
```

`ServiceConfig` in `types/types.go` is refactored:

```go
type ServiceConfig struct {
    ContainerConfig `yaml:",inline" mapstructure:",squash"`

    Profiles    []string           `yaml:"profiles,omitempty" json:"profiles,omitempty"`
    Deploy      *DeployConfig      `yaml:"deploy,omitempty" json:"deploy,omitempty"`
    Scale       *int               `yaml:"scale,omitempty" json:"scale,omitempty"`
    HealthCheck *HealthCheckConfig `yaml:"healthcheck,omitempty" json:"healthcheck,omitempty"`
    Develop     *DevelopConfig     `yaml:"develop,omitempty" json:"develop,omitempty"`
}
```

New file `types/tasks.go`:

```go
type TaskConfig struct {
    ContainerConfig `yaml:",inline" mapstructure:",squash"`
}

type Tasks map[string]TaskConfig

func (t Tasks) Filter(predicate func(TaskConfig) bool) Tasks { /* ... */ }
```

`Config` in `types/config.go`:

```go
type Config struct {
    // ... existing
    Services Services `yaml:"services" json:"services"`
    Tasks    Tasks    `yaml:"tasks,omitempty" json:"tasks,omitempty"`
    // ...
}
```

#### Access to promoted fields

Thanks to Go embedding, `svc.Image`, `svc.Environment`, `svc.Ports` keep working unchanged. Methods defined on `*ContainerConfig` are promoted onto `*ServiceConfig` and `*TaskConfig`.

#### Distribution of existing `ServiceConfig` methods

- Methods that only depend on core fields → moved onto `ContainerConfig` (e.g., path resolution for `build`, `env_file`, `volumes`).
- Methods that depend on `Deploy`/`HealthCheck`/`Develop` → stay on `ServiceConfig` (e.g., `NeedsHealthCheck`).

#### mapstructure tags

The `mapstructure:",squash"` tag is the equivalent of `yaml:",inline"` for the `mapstructure` library used by the loader. It is required on the embedded field for decoding to work.

## Task semantics

### Invocation

**Tasks are not started by default.** They are never launched by `compose up`. They are triggered solely via `compose run <task>` (implemented by the CLI, outside the scope of compose-go).

### Dependencies (v1 iteration)

- A **task may `depends_on`** one or more **services**. When the task is invoked, the CLI must ensure the targeted services are started and (if `service_healthy` condition) healthy.
- A **task CANNOT** `depends_on` another task → validation error.
- A **service CANNOT** `depends_on` a task → validation error.

These restrictions will be lifted in a future iteration with the introduction of new `task_completed_successfully` conditions.

### `extends`

The existing `extends` field is enriched with a `task` field mutually exclusive with `service`:

```yaml
tasks:
  migrate:
    extends:
      task: base-migration     # references another task
      file: common.yaml        # optional, as for service
    command: ["./migrate.sh"]
```

**Rules**:

- A **task can only extend another task** (`extends.task`). `extends.service` inside a `tasks:` context → validation error.
- A **service can only extend another service** (`extends.service`). `extends.task` inside a `services:` context → validation error.
- The short form `extends: <name>` remains valid. Thanks to the cross-namespace uniqueness rule, `<name>` refers to either a service or a task without ambiguity, but the "task can only extend a task" constraint still applies: if a task uses `extends: bar` and `bar` only exists under `services`, this is a validation error (and vice versa for a service).

**Implementation note**: `extends` is part of the `container_spec` core, so it syntactically accepts `service` or `task` in the JSON schema. The semantic validation (prohibition of cross-type extension) is performed **in code inside the loader**, not in the JSON schema. This simplifies the schema and centralizes the rule.

### Namespaces and name uniqueness

The `services` and `tasks` maps are **structurally separate** (two top-level sections, two distinct Go maps). However, for compatibility reasons with the existing CLI semantics, **names must be unique across the union `services ∪ tasks`**:

- `compose run <name>` has historically accepted a **service** name (legacy invocation of a single container based on a service definition).
- `compose run <name>` will also accept a **task** name.

Allowing a collision `services.foo` + `tasks.foo` would create an ambiguity that cannot be resolved at the CLI level. The **loader validation therefore rejects any non-empty intersection** between the set of `services` keys and the set of `tasks` keys.

This constraint is specific to the `services`/`tasks` pair. Other pairs (e.g., `services` / `volumes`) are not affected: they do not share a CLI invocation space.

## Impact on internal pipelines

### Loader (`loader/`)

- **`normalize`** → extend to process tasks: `extends` resolution, relative paths (`build.context`, `env_file`, `volumes` bind, `secrets` file). The logic must be factored onto `*ContainerConfig` to be shared.
- **`validate`** → add the new rules: cross-namespace name uniqueness (no key may appear in both `services` and `tasks`); `tasks.*.depends_on` may only target a service; `tasks.*.extends.service` is invalid; `services.*.extends.task` is invalid; `services.*.depends_on` may only target another service.
- **`resolveDependsOn`** → resolution restricted to the `services` namespace in both cases (from a service or from a task).
- **`extends`** → new code path for the `task` attribute in the extends config.
- **`include`** → collect `tasks` from included files just like `services`.

### Override / Merge (`override/`)

Factor the per-field merge logic into `mergeContainerSpec`. `mergeService` calls `mergeContainerSpec` then merges specific fields (`deploy`, `scale`, etc.). `mergeTask` calls only `mergeContainerSpec`. This applies to `override/merge.go` AND `override/merge_node.go` (yaml.Node path introduced in phase 2 of the ongoing refactor).

### Interpolation (`interpolation/`)

Type-by-path rules are defined in `interpolation/paths.go`. Mechanically duplicate each `services.*.<field>` entry to `tasks.*.<field>`. Same logic in `interpolation/node.go`.

### Transform / Canonical (`transform/canonical.go`)

Canonical transformations (string ports → struct, string volumes → struct) are defined by path. Duplicate `services.*` entries to `tasks.*`.

### Paths (`paths/paths.go`)

Relative path resolution functions (build context, env_file, volumes bind, secrets file, label_file) must operate on `*ContainerConfig`. The refactor consists of changing signatures from `*ServiceConfig` to `*ContainerConfig` wherever functions only use core fields.

### Graph (`graph/`)

The dependency graph remains service-oriented in this v1, since tasks can only depend on services (and not the other way around). No major change at this level — we just need to make sure tasks are visited to resolve their `depends_on` pointing at services.

## Testing strategy

1. **Schema parity task ↔ service (core)** — iterate over `container_spec` keys and verify that each one is accepted on both `services.*` and `tasks.*`.

2. **Rejection of orchestration fields on tasks** — negative tests for `tasks.foo.deploy`, `tasks.foo.scale`, `tasks.foo.healthcheck`, `tasks.foo.develop`, `tasks.foo.profiles`.

3. **`extends`**:
   - `tasks.foo.extends.task: bar` where `bar` exists → OK
   - `tasks.foo.extends.service: bar` → error
   - `services.foo.extends.task: bar` → error
   - Short form `extends: bar` per context → OK

4. **`depends_on`**:
   - `tasks.foo.depends_on: [service_a]` → OK
   - `tasks.foo.depends_on: [other_task]` → error
   - `services.foo.depends_on: [task_a]` → error

5. **Cross-namespace name uniqueness**:
   - `services: { foo: ... }` + `tasks: { bar: ... }` → OK
   - `services: { foo: ... }` + `tasks: { foo: ... }` → validation error with explicit message

6. **Non-regression** — the entire existing test suite must pass unchanged. This is the primary criterion validating that the Go embedding is transparent (`ServiceConfig` remains usable as before).

7. **Merge / Override / Interpolation / Include / Paths** — each pipeline gets its own dedicated tests on `tasks:`.

8. **Binary compatibility** — an existing compose file without `tasks:` must produce exactly the same `Config` as before the refactor.

## Compatibility and migration

### Compatibility guarantees

- All public `ServiceConfig` accessors (`svc.Image`, `svc.Environment`, etc.) work without modification, thanks to Go field promotion.
- All public `ServiceConfig` methods remain callable.
- Every existing compose file without `tasks:` is parsed identically.
- Every existing compose file with `services:` is parsed identically and produces a bit-identical `Services`.

### Breaking changes

**Struct literals**: external code that instantiates a `ServiceConfig` via a struct literal with named fields will find its core fields moved to `ContainerConfig`.

```go
// Before — no longer compiles after the refactor:
svc := types.ServiceConfig{Name: "api", Image: "myapp"}

// After:
svc := types.ServiceConfig{
    ContainerConfig: types.ContainerConfig{Name: "api", Image: "myapp"},
}
```

This change is **unavoidable** with the chosen embedding approach. It will be documented in the CHANGELOG and release notes. A major version bump is appropriate.

### Internal migration

The project itself builds `ServiceConfig` via struct literals in several tests (`loader/loader_test.go`, `types/project_test.go`, etc.). Those literals must be adapted — this is mechanical and part of the implementation plan.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `allOf` + `additionalProperties: false` mishandled by `gojsonschema` | Isolated test as the **very first step** of the implementation plan. On failure, plan B: `container_spec` becomes purely an internal documentation reference in the schema, and both `service` and `task` copy their fields explicitly via generation or textual inclusion. |
| Go embedding ↔ `mapstructure` interaction | Add `mapstructure:",squash"` on the embedded field. Existing loader tests will catch any regression. |
| Go embedding ↔ ongoing yaml.Node refactor (phases 1-4) interaction | The `UnmarshalYAML` methods added in phase 1 on `ServiceConfig` must be adapted. Embedding with `yaml:",inline"` is natively supported by `yaml.v3`, no custom code required. |
| Duplication of interpolation and transform path rules | Programmatic generation (function that iterates over the list of core fields and emits both `services.*.X` and `tasks.*.X` paths) rather than manual duplication. |
| `ServiceConfig` methods using `s.HealthCheck` or `s.Deploy` being called on a `TaskConfig` via promotion | They remain defined on `*ServiceConfig`, not on `*ContainerConfig`, so they are not callable on `TaskConfig`. No risk. |

## Future iterations (for context)

This architecture prepares:

1. **Sidecars** — a sidecar is a `ContainerConfig` attached to a service with an additional `parent` field.
2. **Init-containers** — same principle with pre-main-container execution semantics.
3. **`service → task` dependency** — addition of the `task_completed_successfully` condition in `depends_on`, and lifting of the corresponding validation restriction.
4. **`task → task` dependency** — lifting of the corresponding restriction.

## Implementation plan sections (outline)

The detailed plan is written separately via the `writing-plans` skill. Anticipated major steps:

1. Validate `allOf` + `additionalProperties: false` with `gojsonschema` (isolated test).
2. Extract `container_spec` in the JSON schema (schema refactor + non-regression tests).
3. Add top-level `tasks:` in the schema + `task` definition.
4. Create the `ContainerConfig` Go type with every core field.
5. Refactor `ServiceConfig` via embedding.
6. Create `TaskConfig`, `Tasks`, add to `Config`.
7. Adapt pipelines: loader, override, interpolation, paths, include.
8. Adapt `extends` (new `task` field).
9. Add cross-type dependency validation rules.
10. Full test suite (parity, rejections, non-regression).
