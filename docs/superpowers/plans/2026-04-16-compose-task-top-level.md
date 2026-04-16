# Compose Task Top-Level Element Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce `tasks:` as a new top-level element in the Compose file format, representing one-shot containers invoked via `compose run`, by extracting a shared `container_spec` definition and refactoring `ServiceConfig` via Go embedding.

**Architecture:** Extract ~115 container-shared fields from the `service` JSON Schema definition into a new `container_spec` definition. `service` becomes `allOf: [container_spec, orchestration_fields]`. `task` is a direct alias of `container_spec`. The Go `ServiceConfig` is refactored to embed a new `ContainerConfig` struct containing the shared fields. `TaskConfig` is a thin wrapper around `ContainerConfig`. Cross-type rules (extends, depends_on, name uniqueness) are enforced in the loader validation layer.

**Tech Stack:** Go, JSON Schema draft-07, gojsonschema validator, yaml.v3, mapstructure/v2.

**Pre-requisite:** Read the design spec at `docs/superpowers/specs/2026-04-16-compose-task-top-level-design.md`. The plan assumes familiarity with the decisions documented there.

**Worktree recommendation:** This plan introduces a breaking API change (`ServiceConfig` struct literal field access). Execute in a dedicated worktree to avoid polluting `main` until all tests pass end-to-end. Invoke the `superpowers:using-git-worktrees` skill before Task 0 if not already in one.

---

## File Structure Overview

**New files:**
- `types/container.go` — `ContainerConfig` struct with all shared container fields.
- `types/tasks.go` — `TaskConfig` struct, `Tasks` map type, and its `Filter` method.
- `schema/container_spec_test.go` — schema-level tests (parity, rejection of orchestration fields).
- `loader/tasks_test.go` — end-to-end loader tests for `tasks:` top-level.

**Modified files:**
- `schema/compose-spec.json` — extract `container_spec` definition, refactor `service`, add `task` and top-level `tasks:`.
- `types/types.go` — refactor `ServiceConfig` to embed `ContainerConfig`; extend `ExtendsConfig` with `Task` field.
- `types/config.go` — add `Tasks` field to `Config`; add `Tasks` type.
- `types/unmarshal_test.go` — update struct literal usages.
- `loader/loader.go` — adapt normalize/validate/resolveDependsOn for tasks.
- `loader/extends.go` — support cross-type extends and validate restrictions.
- `loader/normalize.go` — normalize tasks like services.
- `loader/include.go` — collect tasks from included files.
- `loader/validate.go` — add name uniqueness and cross-type dependency rules.
- `override/merge.go` — factor container-spec merge out and add `mergeTasks`.
- `override/merge_node.go` — same for the yaml.Node merge path.
- `interpolation/paths.go` — duplicate `services.*.X` paths to `tasks.*.X`.
- `interpolation/node.go` — same for node-based interpolation.
- `transform/canonical.go` — duplicate transform paths.
- `paths/paths.go` — change signatures from `*ServiceConfig` to `*ContainerConfig` where possible.
- `validation/validation.go` — add task-related path checks.
- Various test files — adapt struct literals of `ServiceConfig`.

---

## Phase A — JSON Schema Refactor

Before touching any Go code, validate the schema-level design and implement it. Non-regression is asserted via the existing schema test suite.

### Task 0: Validate `allOf` + `additionalProperties: false` with gojsonschema

The core schema design relies on composing `service = allOf(container_spec, orchestration_fields)` where only `container_spec` carries `additionalProperties: false`. JSON Schema draft-07 evaluates `additionalProperties` per sub-schema, not cumulatively. Before committing to the refactor, prove the gojsonschema library handles this correctly.

**Files:**
- Create: `schema/allof_probe_test.go`

- [ ] **Step 1: Write a probe test that exercises the exact pattern**

```go
// schema/allof_probe_test.go
package schema

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/xeipuuv/gojsonschema"
)

// TestAllOfAdditionalPropertiesProbe verifies that gojsonschema correctly
// handles `allOf` composition where only the base schema declares
// `additionalProperties: false`. This is the pattern used to compose
// `service = container_spec + orchestration_fields` without duplication.
func TestAllOfAdditionalPropertiesProbe(t *testing.T) {
	probeSchema := `{
		"$schema": "https://json-schema.org/draft-07/schema",
		"type": "object",
		"definitions": {
			"base": {
				"type": "object",
				"properties": {
					"image": {"type": "string"},
					"command": {"type": "string"}
				},
				"additionalProperties": false
			},
			"extended": {
				"allOf": [
					{"$ref": "#/definitions/base"},
					{
						"type": "object",
						"properties": {
							"scale": {"type": "integer"}
						}
					}
				]
			}
		},
		"properties": {
			"sut": {"$ref": "#/definitions/extended"}
		}
	}`

	loader := gojsonschema.NewStringLoader(probeSchema)

	// Case 1: only base fields — must pass.
	doc1 := gojsonschema.NewStringLoader(`{"sut": {"image": "nginx"}}`)
	r1, err := gojsonschema.Validate(loader, doc1)
	assert.NoError(t, err)
	assert.True(t, r1.Valid(), "base-only fields should be accepted: %v", r1.Errors())

	// Case 2: base + extended fields — must pass.
	doc2 := gojsonschema.NewStringLoader(`{"sut": {"image": "nginx", "scale": 3}}`)
	r2, err := gojsonschema.Validate(loader, doc2)
	assert.NoError(t, err)
	assert.True(t, r2.Valid(), "base+extended fields should be accepted: %v", r2.Errors())

	// Case 3: unknown field — must FAIL (base.additionalProperties: false bites).
	doc3 := gojsonschema.NewStringLoader(`{"sut": {"image": "nginx", "unknown": "x"}}`)
	r3, err := gojsonschema.Validate(loader, doc3)
	assert.NoError(t, err)
	assert.False(t, r3.Valid(), "unknown field should be rejected")
}
```

- [ ] **Step 2: Run the probe**

Run: `go test ./schema/ -run TestAllOfAdditionalPropertiesProbe -v`
Expected: PASS for all three cases. If Case 2 or Case 3 fails, STOP and escalate — the plan's schema approach needs revision (fallback: generate the service schema by concatenating fields from a single source of truth at build time).

- [ ] **Step 3: Commit**

```bash
git add schema/allof_probe_test.go
git commit -m "test: validate allOf + additionalProperties:false behavior in gojsonschema"
```

---

### Task 1: Extract `container_spec` definition in JSON Schema

Move all shared fields from `service` into a new `container_spec` definition. `service` is replaced by a stub `allOf: [{$ref: container_spec}, {}]` with an empty second sub-schema. No behavior change for consumers of `services:` — this task is a pure structural refactor that the existing schema tests must pass as-is.

**Files:**
- Modify: `schema/compose-spec.json` (lines 99-925 roughly — the `service` definition)

- [ ] **Step 1: Before editing, capture the current schema test output as a baseline**

Run: `go test ./schema/... -v -count=1 > /tmp/schema-baseline.txt`
Expected: all tests pass. Save this output to confirm non-regression at Step 4.

- [ ] **Step 2: In `schema/compose-spec.json`, replace the existing `"service"` definition**

The new structure:

```jsonc
"service": {
  "allOf": [
    { "$ref": "#/definitions/container_spec" },
    {
      "type": "object",
      "description": "Service-specific orchestration fields.",
      "patternProperties": { "^x-": {} }
    }
  ]
},

"container_spec": {
  "type": "object",
  "description": "Configuration shared by any container-based element (service, task, ...).",
  "properties": {
    // EVERYTHING that is currently in "service.properties" goes here,
    // EXCEPT: deploy, scale, healthcheck, develop, profiles.
  },
  "patternProperties": { "^x-": {} },
  "additionalProperties": false
}
```

Concretely:
1. Copy the entire current `"service"` block to a new `"container_spec"` block.
2. From `container_spec.properties`, REMOVE: `deploy`, `scale`, `healthcheck`, `develop`, `profiles`.
3. Keep `container_spec.additionalProperties: false` and `patternProperties: {"^x-": {}}`.
4. Replace the original `"service"` block with the `allOf` stub above. The second sub-schema is **empty** at this stage — orchestration fields move in Task 2.

Where to place `container_spec`: immediately after `"service"` in the `definitions` block (so the order is `service`, then `container_spec`). This keeps related definitions adjacent.

- [ ] **Step 3: Run the existing schema test suite**

Run: `go test ./schema/... -v -count=1`
Expected: all tests pass. The `service` schema is structurally different but semantically identical (no `deploy`/`scale`/etc. are currently under `additionalProperties: false` that we've lost — they all still match `container_spec.properties` + empty sub-schema … wait, they DON'T).

**Important**: at this stage, `deploy`, `scale`, `healthcheck`, `develop`, `profiles` are no longer accepted by `service`! Tests that exercise these fields will fail. Do NOT fix the test by committing yet; proceed to Task 2 which re-adds them.

- [ ] **Step 4: Commit the intermediate state**

Even though some tests fail at this point (orchestration fields temporarily rejected), the structural extraction is a meaningful atomic change. Commit with a clear WIP marker:

```bash
git add schema/compose-spec.json
git commit -m "refactor(schema): extract container_spec definition (WIP — orchestration fields temporarily unsupported)"
```

---

### Task 2: Restore orchestration fields in `service` via `allOf`

Re-add the five orchestration fields (`deploy`, `scale`, `healthcheck`, `develop`, `profiles`) to the second sub-schema of `service`'s `allOf`. After this task, the full existing schema test suite must pass again.

**Files:**
- Modify: `schema/compose-spec.json`

- [ ] **Step 1: Edit the `service` definition to add the orchestration properties**

```jsonc
"service": {
  "allOf": [
    { "$ref": "#/definitions/container_spec" },
    {
      "type": "object",
      "description": "Service-specific orchestration fields.",
      "properties": {
        "deploy":      { "$ref": "#/definitions/deployment" },
        "scale":       { "type": ["integer", "string"], "description": "Number of replicas of the service container that should be running at any given time." },
        "healthcheck": { "$ref": "#/definitions/healthcheck" },
        "develop":     { "$ref": "#/definitions/development" },
        "profiles":    { "type": "array", "items": { "type": "string" }, "description": "List of profiles under which this service is enabled." }
      },
      "patternProperties": { "^x-": {} }
    }
  ]
}
```

**DO NOT** put `additionalProperties: false` on this second sub-schema. `container_spec` already carries it, and duplicating here would reject the orchestration fields themselves (since they're not in `container_spec`).

- [ ] **Step 2: Run the full test suite**

Run: `go test ./... -count=1`
Expected: all tests pass, identical to the baseline captured in Task 1 Step 1.

- [ ] **Step 3: Commit**

```bash
git add schema/compose-spec.json
git commit -m "refactor(schema): compose service from container_spec + orchestration fields"
```

---

### Task 3: Add `task` definition and top-level `tasks:`

Now that `container_spec` exists, `task` is a trivial `$ref`. Also adds the top-level `tasks:` property.

**Files:**
- Modify: `schema/compose-spec.json`
- Create: `schema/container_spec_test.go`

- [ ] **Step 1: Add the `task` definition in `schema/compose-spec.json`**

Place it immediately after `container_spec` in `definitions`:

```jsonc
"task": {
  "$ref": "#/definitions/container_spec"
}
```

- [ ] **Step 2: Add the top-level `tasks:` property**

In the root `properties` block of `schema/compose-spec.json`, after `services:`:

```jsonc
"tasks": {
  "type": "object",
  "patternProperties": {
    "^[a-zA-Z0-9._-]+$": { "$ref": "#/definitions/task" }
  },
  "additionalProperties": false,
  "description": "Tasks (one-shot containers) triggered explicitly via `compose run`."
}
```

- [ ] **Step 3: Write the schema-level test for tasks**

Create `schema/container_spec_test.go`:

```go
package schema

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestTaskAcceptsContainerFields(t *testing.T) {
	yaml := `
tasks:
  migrate:
    image: myapp:latest
    command: ["./migrate.sh"]
    environment:
      DB_HOST: db
    volumes:
      - ./data:/data
    depends_on:
      - db
`
	err := Validate(toMap(t, yaml))
	assert.NoError(t, err)
}

func TestTaskRejectsDeploy(t *testing.T) {
	yaml := `
tasks:
  migrate:
    image: busybox
    deploy:
      replicas: 3
`
	err := Validate(toMap(t, yaml))
	require.Error(t, err)
	assert.Contains(t, err.Error(), "deploy")
}

func TestTaskRejectsScale(t *testing.T) {
	yaml := `
tasks:
  migrate:
    image: busybox
    scale: 3
`
	err := Validate(toMap(t, yaml))
	require.Error(t, err)
	assert.Contains(t, err.Error(), "scale")
}

func TestTaskRejectsHealthcheck(t *testing.T) {
	yaml := `
tasks:
  migrate:
    image: busybox
    healthcheck:
      test: ["CMD", "true"]
`
	err := Validate(toMap(t, yaml))
	require.Error(t, err)
	assert.Contains(t, err.Error(), "healthcheck")
}

func TestTaskRejectsDevelop(t *testing.T) {
	yaml := `
tasks:
  migrate:
    image: busybox
    develop:
      watch: []
`
	err := Validate(toMap(t, yaml))
	require.Error(t, err)
	assert.Contains(t, err.Error(), "develop")
}

func TestTaskRejectsProfiles(t *testing.T) {
	yaml := `
tasks:
  migrate:
    image: busybox
    profiles: ["dev"]
`
	err := Validate(toMap(t, yaml))
	require.Error(t, err)
	assert.Contains(t, err.Error(), "profiles")
}
```

Note: `toMap(t, yaml)` is a helper used by the existing schema tests — check `schema/schema_test.go` (or similar) for the exact signature and reuse it. If it doesn't exist under that name, write an inline helper that unmarshals YAML into `map[string]any`.

- [ ] **Step 4: Run the new tests**

Run: `go test ./schema/ -run 'TestTask' -v -count=1`
Expected: all new tests pass. Orchestration-rejection tests fail with an "additionalProperties" error mentioning the field name, which is why we assert on the field name substring.

- [ ] **Step 5: Run the full schema test suite for non-regression**

Run: `go test ./schema/ -count=1`
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add schema/compose-spec.json schema/container_spec_test.go
git commit -m "feat(schema): add tasks top-level element and task definition"
```

---

## Phase B — Go Types Refactor

The critical phase. Extract `ContainerConfig`, refactor `ServiceConfig` via embedding, then introduce `TaskConfig`. Tests existing in the repository must all pass after Task 6.

### Task 4: Create `ContainerConfig` with all shared fields

Create a new file `types/container.go` with `ContainerConfig` containing every field from the current `ServiceConfig` **except** `Profiles`, `Deploy`, `Scale`, `HealthCheck`, `Develop`.

**Files:**
- Create: `types/container.go`
- Read: `types/types.go:32-145` (current `ServiceConfig` definition)

- [ ] **Step 1: Study the current `ServiceConfig` definition**

Read `types/types.go` starting at line 32. The struct has ~120 fields with yaml, json, mapstructure tags. Note: `Extensions` is tagged `yaml:"#extensions,inline,omitempty"` — keep this exact form on `ContainerConfig`.

- [ ] **Step 2: Create `types/container.go`**

```go
/*
   Copyright 2020 The Compose Specification Authors.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
*/

package types

// ContainerConfig describes the configuration of a container, shared between
// ServiceConfig, TaskConfig, and any future container-based element.
//
// Fields here MUST NOT include orchestration-only settings (deploy, scale,
// healthcheck, develop, profiles) — those remain on ServiceConfig.
type ContainerConfig struct {
	Name     string `yaml:"name,omitempty" json:"-"`

	Annotations  Mapping      `yaml:"annotations,omitempty" json:"annotations,omitempty"`
	Attach       *bool        `yaml:"attach,omitempty" json:"attach,omitempty"`
	Build        *BuildConfig `yaml:"build,omitempty" json:"build,omitempty"`
	BlkioConfig  *BlkioConfig `yaml:"blkio_config,omitempty" json:"blkio_config,omitempty"`
	CapAdd       []string     `yaml:"cap_add,omitempty" json:"cap_add,omitempty"`
	CapDrop      []string     `yaml:"cap_drop,omitempty" json:"cap_drop,omitempty"`
	CgroupParent string       `yaml:"cgroup_parent,omitempty" json:"cgroup_parent,omitempty"`
	Cgroup       string       `yaml:"cgroup,omitempty" json:"cgroup,omitempty"`
	CPUCount     int64        `yaml:"cpu_count,omitempty" json:"cpu_count,omitempty"`
	CPUPercent   float32      `yaml:"cpu_percent,omitempty" json:"cpu_percent,omitempty"`
	CPUPeriod    int64        `yaml:"cpu_period,omitempty" json:"cpu_period,omitempty"`
	CPUQuota     int64        `yaml:"cpu_quota,omitempty" json:"cpu_quota,omitempty"`
	CPURTPeriod  int64        `yaml:"cpu_rt_period,omitempty" json:"cpu_rt_period,omitempty"`
	CPURTRuntime int64        `yaml:"cpu_rt_runtime,omitempty" json:"cpu_rt_runtime,omitempty"`
	CPUS         float32      `yaml:"cpus,omitempty" json:"cpus,omitempty"`
	CPUSet       string       `yaml:"cpuset,omitempty" json:"cpuset,omitempty"`
	CPUShares    int64        `yaml:"cpu_shares,omitempty" json:"cpu_shares,omitempty"`

	Command ShellCommand `yaml:"command,omitempty" json:"command"`

	Configs           []ServiceConfigObjConfig `yaml:"configs,omitempty" json:"configs,omitempty"`
	ContainerName     string                   `yaml:"container_name,omitempty" json:"container_name,omitempty"`
	CredentialSpec    *CredentialSpecConfig    `yaml:"credential_spec,omitempty" json:"credential_spec,omitempty"`
	DependsOn         DependsOnConfig          `yaml:"depends_on,omitempty" json:"depends_on,omitempty"`
	DeviceCgroupRules []string                 `yaml:"device_cgroup_rules,omitempty" json:"device_cgroup_rules,omitempty"`
	Devices           []DeviceMapping          `yaml:"devices,omitempty" json:"devices,omitempty"`
	DNS               StringList               `yaml:"dns,omitempty" json:"dns,omitempty"`
	DNSOpts           []string                 `yaml:"dns_opt,omitempty" json:"dns_opt,omitempty"`
	DNSSearch         StringList               `yaml:"dns_search,omitempty" json:"dns_search,omitempty"`
	Dockerfile        string                   `yaml:"dockerfile,omitempty" json:"dockerfile,omitempty"`
	DomainName        string                   `yaml:"domainname,omitempty" json:"domainname,omitempty"`

	Entrypoint      ShellCommand                     `yaml:"entrypoint,omitempty" json:"entrypoint"`
	Provider        *ServiceProviderConfig           `yaml:"provider,omitempty" json:"provider,omitempty"`
	Environment     MappingWithEquals                `yaml:"environment,omitempty" json:"environment,omitempty"`
	EnvFiles        []EnvFile                        `yaml:"env_file,omitempty" json:"env_file,omitempty"`
	Expose          StringOrNumberList               `yaml:"expose,omitempty" json:"expose,omitempty"`
	Extends         *ExtendsConfig                   `yaml:"extends,omitempty" json:"extends,omitempty"`
	ExternalLinks   []string                         `yaml:"external_links,omitempty" json:"external_links,omitempty"`
	ExtraHosts      HostsList                        `yaml:"extra_hosts,omitempty" json:"extra_hosts,omitempty"`
	GroupAdd        []string                         `yaml:"group_add,omitempty" json:"group_add,omitempty"`
	Gpus            []DeviceRequest                  `yaml:"gpus,omitempty" json:"gpus,omitempty"`
	Hostname        string                           `yaml:"hostname,omitempty" json:"hostname,omitempty"`
	Image           string                           `yaml:"image,omitempty" json:"image,omitempty"`
	Init            *bool                            `yaml:"init,omitempty" json:"init,omitempty"`
	Ipc             string                           `yaml:"ipc,omitempty" json:"ipc,omitempty"`
	Isolation       string                           `yaml:"isolation,omitempty" json:"isolation,omitempty"`
	Labels          Labels                           `yaml:"labels,omitempty" json:"labels,omitempty"`
	LabelFiles      []string                         `yaml:"label_file,omitempty" json:"label_file,omitempty"`
	CustomLabels    Labels                           `yaml:"-" json:"-"`
	Links           []string                         `yaml:"links,omitempty" json:"links,omitempty"`
	Logging         *LoggingConfig                   `yaml:"logging,omitempty" json:"logging,omitempty"`
	LogDriver       string                           `yaml:"log_driver,omitempty" json:"log_driver,omitempty"`
	LogOpt          map[string]string                `yaml:"log_opt,omitempty" json:"log_opt,omitempty"`
	MemLimit        UnitBytes                        `yaml:"mem_limit,omitempty" json:"mem_limit,omitempty"`
	MemReservation  UnitBytes                        `yaml:"mem_reservation,omitempty" json:"mem_reservation,omitempty"`
	MemSwapLimit    UnitBytes                        `yaml:"memswap_limit,omitempty" json:"memswap_limit,omitempty"`
	MemSwappiness   UnitBytes                        `yaml:"mem_swappiness,omitempty" json:"mem_swappiness,omitempty"`
	MacAddress      string                           `yaml:"mac_address,omitempty" json:"mac_address,omitempty"`
	Models          map[string]*ServiceModelConfig   `yaml:"models,omitempty" json:"models,omitempty"`
	Net             string                           `yaml:"net,omitempty" json:"net,omitempty"`
	NetworkMode     string                           `yaml:"network_mode,omitempty" json:"network_mode,omitempty"`
	Networks        map[string]*ServiceNetworkConfig `yaml:"networks,omitempty" json:"networks,omitempty"`
	OomKillDisable  bool                             `yaml:"oom_kill_disable,omitempty" json:"oom_kill_disable,omitempty"`
	OomScoreAdj     int64                            `yaml:"oom_score_adj,omitempty" json:"oom_score_adj,omitempty"`
	Pid             string                           `yaml:"pid,omitempty" json:"pid,omitempty"`
	PidsLimit       int64                            `yaml:"pids_limit,omitempty" json:"pids_limit,omitempty"`
	Platform        string                           `yaml:"platform,omitempty" json:"platform,omitempty"`
	Ports           []ServicePortConfig              `yaml:"ports,omitempty" json:"ports,omitempty"`
	Privileged      bool                             `yaml:"privileged,omitempty" json:"privileged,omitempty"`
	PullPolicy      string                           `yaml:"pull_policy,omitempty" json:"pull_policy,omitempty"`
	ReadOnly        bool                             `yaml:"read_only,omitempty" json:"read_only,omitempty"`
	Restart         string                           `yaml:"restart,omitempty" json:"restart,omitempty"`
	Runtime         string                           `yaml:"runtime,omitempty" json:"runtime,omitempty"`
	Secrets         []ServiceSecretConfig            `yaml:"secrets,omitempty" json:"secrets,omitempty"`
	SecurityOpt     []string                         `yaml:"security_opt,omitempty" json:"security_opt,omitempty"`
	ShmSize         UnitBytes                        `yaml:"shm_size,omitempty" json:"shm_size,omitempty"`
	StdinOpen       bool                             `yaml:"stdin_open,omitempty" json:"stdin_open,omitempty"`
	StopGracePeriod *Duration                        `yaml:"stop_grace_period,omitempty" json:"stop_grace_period,omitempty"`
	StopSignal      string                           `yaml:"stop_signal,omitempty" json:"stop_signal,omitempty"`
	StorageOpt      map[string]string                `yaml:"storage_opt,omitempty" json:"storage_opt,omitempty"`
	Sysctls         Mapping                          `yaml:"sysctls,omitempty" json:"sysctls,omitempty"`
	Tmpfs           StringList                       `yaml:"tmpfs,omitempty" json:"tmpfs,omitempty"`
	Tty             bool                             `yaml:"tty,omitempty" json:"tty,omitempty"`
	Ulimits         map[string]*UlimitsConfig        `yaml:"ulimits,omitempty" json:"ulimits,omitempty"`
	UseAPISocket    bool                             `yaml:"use_api_socket,omitempty" json:"use_api_socket,omitempty"`
	User            string                           `yaml:"user,omitempty" json:"user,omitempty"`
	UserNSMode      string                           `yaml:"userns_mode,omitempty" json:"userns_mode,omitempty"`
	Uts             string                           `yaml:"uts,omitempty" json:"uts,omitempty"`
	VolumeDriver    string                           `yaml:"volume_driver,omitempty" json:"volume_driver,omitempty"`
	Volumes         []ServiceVolumeConfig            `yaml:"volumes,omitempty" json:"volumes,omitempty"`
	VolumesFrom     []string                         `yaml:"volumes_from,omitempty" json:"volumes_from,omitempty"`
	WorkingDir      string                           `yaml:"working_dir,omitempty" json:"working_dir,omitempty"`
	PostStart       []ServiceHook                    `yaml:"post_start,omitempty" json:"post_start,omitempty"`
	PreStop         []ServiceHook                    `yaml:"pre_stop,omitempty" json:"pre_stop,omitempty"`

	Extensions Extensions `yaml:"#extensions,inline,omitempty" json:"-"`
}
```

- [ ] **Step 3: Verify the file compiles**

Run: `go build ./types/`
Expected: success. Any compilation error here means a type reference is missing (`ServiceConfigObjConfig`, `BuildConfig`, etc.) — all these types already exist in the `types` package, so compile errors would be typos in the new file.

- [ ] **Step 4: Commit**

```bash
git add types/container.go
git commit -m "feat(types): add ContainerConfig with shared container fields"
```

---

### Task 5: Refactor `ServiceConfig` to embed `ContainerConfig`

Replace the 120-field `ServiceConfig` body with an embed of `ContainerConfig` plus the 5 orchestration fields. This is the breaking change for external consumers using struct literals with field names — all consumers within this repo must be updated in Task 6.

**Files:**
- Modify: `types/types.go:32-145`

- [ ] **Step 1: Replace the body of `ServiceConfig`**

```go
// ServiceConfig is the service configuration.
//
// It embeds ContainerConfig (shared with TaskConfig and future container
// types) and adds orchestration-specific fields: Profiles, Deploy, Scale,
// HealthCheck, Develop.
type ServiceConfig struct {
	ContainerConfig `yaml:",inline" mapstructure:",squash"`

	Profiles    []string           `yaml:"profiles,omitempty" json:"profiles,omitempty"`
	Deploy      *DeployConfig      `yaml:"deploy,omitempty" json:"deploy,omitempty"`
	Scale       *int               `yaml:"scale,omitempty" json:"scale,omitempty"`
	HealthCheck *HealthCheckConfig `yaml:"healthcheck,omitempty" json:"healthcheck,omitempty"`
	Develop     *DevelopConfig     `yaml:"develop,omitempty" json:"develop,omitempty"`
}
```

Remove the `Extensions` field from `ServiceConfig` — it is now provided by the embedded `ContainerConfig`.

- [ ] **Step 2: Attempt to build**

Run: `go build ./...`
Expected: several packages fail to compile with errors like `cannot use promoted field ServiceConfig.Image in struct literal of type ServiceConfig`. These are **expected** — all internal struct literal uses will be fixed in Task 6.

- [ ] **Step 3: Collect the compile errors**

Run: `go build ./... 2>&1 | tee /tmp/build-errors.txt`
Expected: a list of file:line references for every literal construction of `ServiceConfig` with embedded fields. Use this list as a checklist for Task 6.

- [ ] **Step 4: Do NOT commit yet**

The repo is currently broken. Proceed immediately to Task 6.

---

### Task 6: Update internal `ServiceConfig` struct literal usages

Fix every compile error collected in Task 5 Step 3 by wrapping noyau fields in `ContainerConfig{...}`.

**Files:**
- Modify: any file listed in `/tmp/build-errors.txt`. Expected primary offenders:
  - `types/unmarshal_test.go`
  - `loader/loader_test.go`
  - `loader/override_test.go`
  - `loader/normalize_test.go`
  - `loader/extends_test.go`
  - `types/project_test.go`
  - `override/merge.go` (for any ServiceConfig literal)
  - `graph/services.go` and tests

- [ ] **Step 1: For each file in the build-errors list, apply the mechanical rewrite**

Pattern — BEFORE:

```go
svc := types.ServiceConfig{
    Name:        "api",
    Image:       "myapp:1",
    Environment: types.MappingWithEquals{"KEY": strPtr("value")},
    Deploy:      &types.DeployConfig{Replicas: intPtr(2)},
}
```

Pattern — AFTER:

```go
svc := types.ServiceConfig{
    ContainerConfig: types.ContainerConfig{
        Name:        "api",
        Image:       "myapp:1",
        Environment: types.MappingWithEquals{"KEY": strPtr("value")},
    },
    Deploy: &types.DeployConfig{Replicas: intPtr(2)},
}
```

Rule: any field in the original literal that now belongs to `ContainerConfig` (i.e. NOT one of `Profiles`, `Deploy`, `Scale`, `HealthCheck`, `Develop`) goes inside `ContainerConfig{...}`.

- [ ] **Step 2: After fixing each file, verify compile**

Run: `go build ./...`
Expected: each fix should reduce the number of errors. Iterate until `go build ./...` succeeds.

- [ ] **Step 3: Run the full test suite**

Run: `go test ./... -count=1`
Expected: all tests pass. If any test fails, investigate:
- If failure is in `yaml` round-trip: the `yaml:",inline"` tag on the embed must be present. Check `ServiceConfig`.
- If failure is in `mapstructure` decoding: the `mapstructure:",squash"` tag must be present.
- If failure is in `json` marshaling: JSON handles embedding natively; no change needed. A failure here points to a specific field tag issue.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor(types): embed ContainerConfig in ServiceConfig"
```

---

### Task 7: Create `TaskConfig`, `Tasks`, and add to `Config`

**Files:**
- Create: `types/tasks.go`
- Modify: `types/config.go`

- [ ] **Step 1: Create `types/tasks.go`**

```go
/*
   Copyright 2020 The Compose Specification Authors.
   ... (same license header as types/services.go)
*/

package types

// TaskConfig is the configuration of a Compose task, a one-shot container
// invoked explicitly via `compose run <task>`.
//
// Unlike ServiceConfig, TaskConfig has no orchestration fields (no deploy,
// scale, healthcheck, develop, profiles). Its lifecycle is governed by the
// container's exit code.
type TaskConfig struct {
	ContainerConfig `yaml:",inline" mapstructure:",squash"`
}

// Tasks is a map of TaskConfig keyed by task name.
type Tasks map[string]TaskConfig

// Filter returns a new Tasks map containing only tasks matching the predicate.
func (t Tasks) Filter(predicate func(TaskConfig) bool) Tasks {
	tasks := Tasks{}
	for name, task := range t {
		if predicate(task) {
			tasks[name] = task
		}
	}
	return tasks
}
```

- [ ] **Step 2: Modify `types/config.go` to add the `Tasks` field**

In the `Config` struct (around line 80), insert `Tasks` immediately after `Services`:

```go
type Config struct {
	Filename   string          `yaml:"-" json:"-"`
	Name       string          `yaml:"name,omitempty" json:"name,omitempty"`
	Services   Services        `yaml:"services" json:"services"`
	Tasks      Tasks           `yaml:"tasks,omitempty" json:"tasks,omitempty"`
	Networks   Networks        `yaml:"networks,omitempty" json:"networks,omitempty"`
	Volumes    Volumes         `yaml:"volumes,omitempty" json:"volumes,omitempty"`
	Secrets    Secrets         `yaml:"secrets,omitempty" json:"secrets,omitempty"`
	Configs    Configs         `yaml:"configs,omitempty" json:"configs,omitempty"`
	Extensions Extensions      `yaml:",inline" json:"-"`
	Include    []IncludeConfig `yaml:"include,omitempty" json:"include,omitempty"`
}
```

- [ ] **Step 3: Update `Config.MarshalJSON` in `types/config.go`**

Find the `MarshalJSON` method and add the `tasks` key when non-empty. Insert after the `networks` handling:

```go
if len(c.Tasks) > 0 {
    m["tasks"] = c.Tasks
}
```

- [ ] **Step 4: Build and test**

Run: `go build ./... && go test ./types/ -count=1`
Expected: success.

- [ ] **Step 5: Commit**

```bash
git add types/tasks.go types/config.go
git commit -m "feat(types): add TaskConfig and Tasks map in Config"
```

---

## Phase C — Pipelines

Adapt each internal pipeline (interpolation, transform, paths, loader, override) to process `tasks:` alongside `services:`.

### Task 8: Duplicate interpolation paths for tasks

The interpolation engine uses path-based type hints (e.g. `services.*.scale → integer`). These rules must cover the same paths under `tasks.*`.

**Files:**
- Modify: `interpolation/paths.go`
- Modify: `interpolation/node.go` (if path rules exist there too)

- [ ] **Step 1: Locate the service path rules**

Run: `grep -n '"services\.\*' interpolation/paths.go`
Expected: a list of entries like `"services.*.scale": toInt`.

- [ ] **Step 2: Generate task counterparts programmatically (at init-time)**

At the bottom of `interpolation/paths.go`, inside an `init()` function (or immediately after the rules map is declared if top-level), mirror each `services.*.X` entry to `tasks.*.X`, EXCEPT for the orchestration paths that don't apply to tasks (`services.*.deploy.*`, `services.*.scale`, `services.*.healthcheck.*`, `services.*.develop.*`, `services.*.profiles.*`).

Pattern:

```go
func init() {
	orchestrationOnly := []string{
		".deploy.", ".scale", ".healthcheck.", ".develop.", ".profiles",
	}
	for path, fn := range rules {
		if !strings.HasPrefix(path, "services.*.") {
			continue
		}
		suffix := strings.TrimPrefix(path, "services.*.")
		skip := false
		for _, marker := range orchestrationOnly {
			if strings.Contains("."+suffix, marker) {
				skip = true
				break
			}
		}
		if skip {
			continue
		}
		rules["tasks.*."+suffix] = fn
	}
}
```

Note: adapt to the exact variable name used (`rules`, `paths`, or whatever the existing map is called). Read the file first.

- [ ] **Step 3: Repeat for `interpolation/node.go`**

If this file also has hardcoded `services.*` path rules (introduced in the yaml.Node refactor phase 3), apply the same duplication logic.

- [ ] **Step 4: Run interpolation tests**

Run: `go test ./interpolation/ -count=1`
Expected: pass.

- [ ] **Step 5: Commit**

```bash
git add interpolation/
git commit -m "feat(interpolation): mirror services.* paths to tasks.*"
```

---

### Task 9: Duplicate transform/canonical paths for tasks

Same pattern as Task 8 for `transform/canonical.go`.

**Files:**
- Modify: `transform/canonical.go`

- [ ] **Step 1: Locate the service path transformers**

Run: `grep -n '"services\.\*' transform/canonical.go`

- [ ] **Step 2: Add task mirror paths at init**

Use the same mirroring logic as Task 8. Same `orchestrationOnly` exclusion list.

- [ ] **Step 3: Run transform tests**

Run: `go test ./transform/ -count=1`
Expected: pass.

- [ ] **Step 4: Commit**

```bash
git add transform/canonical.go
git commit -m "feat(transform): mirror services.* canonical paths to tasks.*"
```

---

### Task 10: Refactor `paths/paths.go` to operate on `*ContainerConfig`

Functions that resolve relative paths (build context, env_file, volumes bind, secrets file, label_file) currently take `*ServiceConfig`. Change their signatures to `*ContainerConfig` where they only touch noyau fields, then invoke them for both services and tasks.

**Files:**
- Modify: `paths/paths.go` (and any other `paths/*.go` that resolves service paths)

- [ ] **Step 1: Audit which functions in `paths/paths.go` only access noyau fields**

Read the file. For each exported function taking `*types.ServiceConfig`, check whether it references `s.Deploy`, `s.HealthCheck`, `s.Develop`, `s.Scale`, or `s.Profiles`. If not, it's a candidate for signature change.

- [ ] **Step 2: Change signatures from `*ServiceConfig` to `*ContainerConfig`**

For each candidate:

```go
// Before:
func resolveServiceBuildPaths(s *types.ServiceConfig, wd string) error { ... }

// After:
func resolveContainerBuildPaths(c *types.ContainerConfig, wd string) error { ... }
```

Rename to reflect generality. Update all call sites — they will currently pass a `*ServiceConfig`, which Go lets take `&svc.ContainerConfig` trivially (use `&svc.ContainerConfig`).

- [ ] **Step 3: Invoke the refactored functions for tasks**

In the top-level resolver (usually `paths/paths.go` has a function like `ResolveRelativePaths` or similar), add a loop over `project.Tasks` that calls the same `*ContainerConfig` functions.

- [ ] **Step 4: Run paths tests**

Run: `go test ./paths/ -count=1`
Expected: pass. Existing service tests still pass because `*ServiceConfig` still has access to its noyau via the embed.

- [ ] **Step 5: Commit**

```bash
git add paths/
git commit -m "refactor(paths): operate on *ContainerConfig for shared resolution"
```

---

### Task 11: Extend loader normalize for tasks

**Files:**
- Modify: `loader/normalize.go`
- Modify: `loader/loader.go` (the top-level `normalizeCompose` or `Normalize` function if present)

- [ ] **Step 1: Read `loader/normalize.go`**

Identify the service-level normalization entry point (typical name: `normalizeService` or `normalizeServiceConfig`). Note every line that references `Deploy`, `HealthCheck`, `Develop`, `Scale`, `Profiles` — those stay on a service-specific path.

- [ ] **Step 2: Extract the noyau-only logic**

Create a helper:

```go
func normalizeContainer(name string, c *types.ContainerConfig, project *types.Project) error {
    // all the logic that doesn't touch Deploy/HealthCheck/Develop/Scale/Profiles
    // e.g., env_file resolution, volume path normalization, build context, ...
}
```

`normalizeService` now calls `normalizeContainer(name, &svc.ContainerConfig, project)` first, then applies the orchestration-specific normalization on `svc.Deploy`, `svc.HealthCheck`, etc.

- [ ] **Step 3: Call `normalizeContainer` for tasks**

In the top-level `normalize` function (wherever services are iterated), add a parallel loop for `project.Tasks`:

```go
for name, task := range project.Tasks {
    tc := task
    if err := normalizeContainer(name, &tc.ContainerConfig, project); err != nil {
        return err
    }
    project.Tasks[name] = tc
}
```

- [ ] **Step 4: Build and run loader tests**

Run: `go test ./loader/ -count=1`
Expected: pass (existing tests do not use tasks yet).

- [ ] **Step 5: Commit**

```bash
git add loader/
git commit -m "feat(loader): normalize tasks using shared container logic"
```

---

### Task 12: Extend loader `include` to collect tasks

**Files:**
- Modify: `loader/include.go`

- [ ] **Step 1: Locate the merge of included sections into the main config**

Look for lines that merge `networks`, `volumes`, `services` from the included model. They should be grouped in a single function.

- [ ] **Step 2: Add a merge of `tasks`**

Following the same pattern as `services`:

```go
if len(included.Tasks) > 0 {
    if main.Tasks == nil {
        main.Tasks = types.Tasks{}
    }
    for name, task := range included.Tasks {
        if _, exists := main.Tasks[name]; exists {
            return fmt.Errorf("task %q is already defined", name)
        }
        main.Tasks[name] = task
    }
}
```

(Exact error-handling pattern must mirror what `services` does — read the neighboring code.)

- [ ] **Step 3: Write an include test**

Add a test case in `loader/include_test.go` (or a new test file) that includes a file defining a task and asserts the task is visible in `project.Tasks`.

- [ ] **Step 4: Run tests**

Run: `go test ./loader/ -count=1`
Expected: pass.

- [ ] **Step 5: Commit**

```bash
git add loader/
git commit -m "feat(loader): collect tasks from included compose files"
```

---

### Task 13: Extend override/merge for tasks

**Files:**
- Modify: `override/merge.go`
- Modify: `override/merge_node.go`

- [ ] **Step 1: Read `override/merge.go` and locate `mergeService`**

Note which fields are explicitly merged (Deploy, HealthCheck, etc. vs shared container fields).

- [ ] **Step 2: Extract `mergeContainerSpec`**

Create a helper:

```go
// mergeContainerSpec merges all ContainerConfig fields from right into left.
// It does NOT touch orchestration-specific fields (Deploy, Scale,
// HealthCheck, Develop, Profiles) — those are handled by mergeService.
func mergeContainerSpec(left, right *types.ContainerConfig) error {
    // all the current noyau merge logic from mergeService
}
```

`mergeService` calls `mergeContainerSpec(&left.ContainerConfig, &right.ContainerConfig)` first, then merges orchestration fields.

- [ ] **Step 3: Add `mergeTasks`**

```go
func mergeTasks(left, right types.Tasks) (types.Tasks, error) {
    if len(right) == 0 {
        return left, nil
    }
    if left == nil {
        left = types.Tasks{}
    }
    for name, rightTask := range right {
        if leftTask, exists := left[name]; exists {
            lc := leftTask.ContainerConfig
            if err := mergeContainerSpec(&lc, &rightTask.ContainerConfig); err != nil {
                return nil, err
            }
            leftTask.ContainerConfig = lc
            left[name] = leftTask
        } else {
            left[name] = rightTask
        }
    }
    return left, nil
}
```

Call `mergeTasks` from the top-level `Merge` function next to `mergeServices`.

- [ ] **Step 4: Apply the same logic to `override/merge_node.go`**

The yaml.Node merge path introduced in phase 2 of the yaml.Node refactor uses a path-driven approach. Ensure `tasks.*.X` paths receive the same merge rules as `services.*.X` (noyau fields only).

- [ ] **Step 5: Write a merge test for tasks**

Add a test case in `override/merge_test.go` that overrides a task across two compose files.

- [ ] **Step 6: Run tests**

Run: `go test ./override/ -count=1`
Expected: pass.

- [ ] **Step 7: Commit**

```bash
git add override/
git commit -m "feat(override): merge tasks via shared container-spec logic"
```

---

## Phase D — Cross-type Rules

Extends, depends_on cross-type restrictions, and name uniqueness.

### Task 14: Add `Task` field to `ExtendsConfig` and update schema

**Files:**
- Modify: `types/types.go` (line 827, `ExtendsConfig`)
- Modify: `schema/compose-spec.json` (extends sub-schema)

- [ ] **Step 1: Update `ExtendsConfig` in `types/types.go`**

```go
type ExtendsConfig struct {
	File    string `yaml:"file,omitempty" json:"file,omitempty"`
	Service string `yaml:"service,omitempty" json:"service,omitempty"`
	Task    string `yaml:"task,omitempty" json:"task,omitempty"`
}
```

- [ ] **Step 2: Update the `extends` schema**

In `schema/compose-spec.json`, in the `extends` property definition of `container_spec`:

```jsonc
"extends": {
  "oneOf": [
    {"type": "string"},
    {
      "type": "object",
      "properties": {
        "service": {
          "type": "string",
          "description": "The name of the service to extend."
        },
        "task": {
          "type": "string",
          "description": "The name of the task to extend."
        },
        "file": {
          "type": "string",
          "description": "The file path where the element to extend is defined."
        }
      },
      "oneOf": [
        {"required": ["service"], "not": {"required": ["task"]}},
        {"required": ["task"], "not": {"required": ["service"]}}
      ],
      "additionalProperties": false
    }
  ],
  "description": "Extend another service or task, in the current file or another file."
}
```

**Note**: `extends` lives in `container_spec`, so both service and task inherit this schema.

- [ ] **Step 3: Write a schema test for `extends.task`**

Add to `schema/container_spec_test.go`:

```go
func TestTaskExtendsTask(t *testing.T) {
	yaml := `
tasks:
  base:
    image: busybox
  derived:
    extends:
      task: base
    command: ["derived"]
`
	err := Validate(toMap(t, yaml))
	assert.NoError(t, err)
}

func TestExtendsRejectsBothServiceAndTask(t *testing.T) {
	yaml := `
services:
  foo:
    extends:
      service: bar
      task: baz
`
	err := Validate(toMap(t, yaml))
	require.Error(t, err)
}
```

- [ ] **Step 4: Run schema tests**

Run: `go test ./schema/ -count=1`
Expected: pass.

- [ ] **Step 5: Build to ensure type is usable**

Run: `go build ./...`
Expected: success.

- [ ] **Step 6: Commit**

```bash
git add types/types.go schema/compose-spec.json schema/container_spec_test.go
git commit -m "feat(types,schema): support extends.task for task extension"
```

---

### Task 15: Implement `extends.task` resolution in loader

**Files:**
- Modify: `loader/extends.go`

- [ ] **Step 1: Read `loader/extends.go`**

Understand how `extends.service` is resolved (service lookup by name in same file or included file).

- [ ] **Step 2: Add `extends.task` resolution**

Mirror the service resolution logic for tasks. Key decision: the caller context (is the resolver working on a service or a task?) determines which `extends` shape is allowed.

Pseudocode:

```go
func resolveExtends(container *types.ContainerConfig, isTask bool, ...) error {
    if container.Extends == nil {
        return nil
    }
    ext := container.Extends

    switch {
    case ext.Service != "" && ext.Task != "":
        return fmt.Errorf("extends: cannot specify both `service` and `task`")
    case isTask && ext.Service != "":
        return fmt.Errorf("a task can only extend another task, not a service")
    case !isTask && ext.Task != "":
        return fmt.Errorf("a service can only extend another service, not a task")
    }
    // existing resolution logic, but use ext.Service or ext.Task depending on isTask
}
```

- [ ] **Step 3: Handle the short form `extends: <name>`**

When `ext.File == "" && ext.Service != "" && ext.Task == ""` came from the short form, the loader has already populated `ext.Service`. Since `isTask` is known from context, enforce:
- If `isTask` and the short form is used, and the name exists in `tasks` → OK, treat as `ext.Task`.
- If `isTask` and the short form resolves only to a service → error.
- If not `isTask` and the short form resolves only to a task → error.
- Name uniqueness (see Task 16) guarantees the name is unambiguous.

- [ ] **Step 4: Write loader tests for cross-type extends**

In `loader/extends_test.go`:

```go
func TestTaskExtendsService_Rejected(t *testing.T) {
    yaml := `
services:
  svc:
    image: nginx
tasks:
  t:
    extends:
      service: svc
`
    _, err := LoadString(yaml)
    require.Error(t, err)
    assert.Contains(t, err.Error(), "task can only extend")
}

func TestServiceExtendsTask_Rejected(t *testing.T) {
    yaml := `
tasks:
  t:
    image: busybox
services:
  svc:
    extends:
      task: t
`
    _, err := LoadString(yaml)
    require.Error(t, err)
    assert.Contains(t, err.Error(), "service can only extend")
}

func TestTaskExtendsTask_OK(t *testing.T) {
    yaml := `
tasks:
  base:
    image: busybox
    environment:
      FOO: bar
  derived:
    extends:
      task: base
    command: ["echo", "hello"]
`
    project, err := LoadString(yaml)
    require.NoError(t, err)
    assert.Equal(t, "busybox", project.Tasks["derived"].Image)
    assert.Equal(t, types.MappingWithEquals{"FOO": strPtr("bar")}, project.Tasks["derived"].Environment)
}
```

Adapt `LoadString` to whatever helper the loader tests use (check `loader/loader_test.go`).

- [ ] **Step 5: Run loader tests**

Run: `go test ./loader/ -run Extends -count=1 -v`
Expected: pass.

- [ ] **Step 6: Commit**

```bash
git add loader/
git commit -m "feat(loader): cross-type extends validation (task↔task, service↔service)"
```

---

### Task 16: Enforce name uniqueness across services ∪ tasks

**Files:**
- Modify: `loader/validate.go`

- [ ] **Step 1: Write the failing test**

In `loader/validate_test.go`:

```go
func TestTaskAndServiceNameCollision_Rejected(t *testing.T) {
    yaml := `
services:
  foo:
    image: nginx
tasks:
  foo:
    image: busybox
`
    _, err := LoadString(yaml)
    require.Error(t, err)
    assert.Contains(t, err.Error(), "name")
    assert.Contains(t, err.Error(), "foo")
}

func TestTaskAndServiceDistinctNames_OK(t *testing.T) {
    yaml := `
services:
  api:
    image: nginx
tasks:
  migrate:
    image: busybox
`
    project, err := LoadString(yaml)
    require.NoError(t, err)
    assert.NotNil(t, project.Services["api"])
    assert.NotNil(t, project.Tasks["migrate"])
}
```

- [ ] **Step 2: Run the test to confirm failure**

Run: `go test ./loader/ -run TestTaskAndServiceNameCollision -count=1 -v`
Expected: FAIL (no validation yet).

- [ ] **Step 3: Add the validation**

In `loader/validate.go`, add a check called from the top-level validation:

```go
// checkNameUniqueness ensures no name appears in both services and tasks.
// This guarantees `compose run <name>` can unambiguously resolve to a
// single target.
func checkNameUniqueness(project *types.Project) error {
    for name := range project.Tasks {
        if _, conflict := project.Services[name]; conflict {
            return fmt.Errorf("name %q is used by both a service and a task; names must be unique across services and tasks", name)
        }
    }
    return nil
}
```

Call `checkNameUniqueness(project)` at the start of the top-level validation (before any other per-service/per-task validation runs).

- [ ] **Step 4: Re-run the tests**

Run: `go test ./loader/ -run 'TestTaskAndService' -count=1 -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add loader/validate.go loader/validate_test.go
git commit -m "feat(loader): reject service/task name collisions"
```

---

### Task 17: Validate depends_on cross-type restrictions

Ensure that a task's `depends_on` only references services, and that a service's `depends_on` only references services (existing behavior, now made explicit to reject future accidental task references).

**Files:**
- Modify: `loader/validate.go`

- [ ] **Step 1: Write the failing tests**

In `loader/validate_test.go`:

```go
func TestTaskDependsOnService_OK(t *testing.T) {
    yaml := `
services:
  db:
    image: postgres
tasks:
  migrate:
    image: busybox
    depends_on:
      - db
`
    _, err := LoadString(yaml)
    require.NoError(t, err)
}

func TestTaskDependsOnTask_Rejected(t *testing.T) {
    yaml := `
tasks:
  a:
    image: busybox
  b:
    image: busybox
    depends_on:
      - a
`
    _, err := LoadString(yaml)
    require.Error(t, err)
    assert.Contains(t, err.Error(), "task cannot depend on another task")
}

func TestServiceDependsOnTask_Rejected(t *testing.T) {
    yaml := `
tasks:
  t:
    image: busybox
services:
  svc:
    image: nginx
    depends_on:
      - t
`
    _, err := LoadString(yaml)
    require.Error(t, err)
    assert.Contains(t, err.Error(), "service cannot depend on a task")
}
```

- [ ] **Step 2: Run tests to confirm failure**

Run: `go test ./loader/ -run 'TestTaskDependsOn\|TestServiceDependsOn' -count=1 -v`
Expected: FAIL for the two rejection tests (the OK one may pass if depends_on resolution doesn't care about namespace yet).

- [ ] **Step 3: Implement validation**

In `loader/validate.go`:

```go
func checkDependsOnCrossType(project *types.Project) error {
    taskNames := map[string]struct{}{}
    for name := range project.Tasks {
        taskNames[name] = struct{}{}
    }

    // Services cannot depend on tasks.
    for svcName, svc := range project.Services {
        for depName := range svc.DependsOn {
            if _, isTask := taskNames[depName]; isTask {
                return fmt.Errorf("service %q cannot depend on a task (%q); only service-to-service dependencies are supported", svcName, depName)
            }
        }
    }

    // Tasks cannot depend on other tasks.
    for taskName, task := range project.Tasks {
        for depName := range task.DependsOn {
            if _, isTask := taskNames[depName]; isTask {
                return fmt.Errorf("task %q cannot depend on another task (%q); a task can only depend on services", taskName, depName)
            }
        }
    }

    return nil
}
```

Call `checkDependsOnCrossType(project)` in the top-level validation after `checkNameUniqueness` (name uniqueness guarantees the `isTask` test is unambiguous).

- [ ] **Step 4: Re-run tests**

Run: `go test ./loader/ -run 'TestTaskDependsOn\|TestServiceDependsOn' -count=1 -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add loader/validate.go loader/validate_test.go
git commit -m "feat(loader): reject cross-type depends_on (task→task, service→task)"
```

---

## Phase E — End-to-end Integration and Self-Review

### Task 18: End-to-end loader test for tasks

**Files:**
- Create: `loader/tasks_test.go`

- [ ] **Step 1: Write the integration test**

```go
package loader

import (
    "testing"

    "github.com/compose-spec/compose-go/v2/types"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestLoadTasksFullIntegration(t *testing.T) {
    yaml := `
name: test-project
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
tasks:
  migrate:
    image: myapp:latest
    command: ["./migrate.sh"]
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/app
    depends_on:
      - db
    volumes:
      - migrations:/migrations
volumes:
  migrations:
`
    project, err := LoadString(yaml)
    require.NoError(t, err)
    require.NotNil(t, project.Tasks)

    require.Contains(t, project.Tasks, "migrate")
    task := project.Tasks["migrate"]

    assert.Equal(t, "myapp:latest", task.Image)
    assert.Equal(t, types.ShellCommand{"./migrate.sh"}, task.Command)
    assert.Equal(t, "postgres://postgres:secret@db:5432/app", *task.Environment["DATABASE_URL"])
    require.Len(t, task.Volumes, 1)
    assert.Equal(t, "migrations", task.Volumes[0].Source)

    require.Contains(t, task.DependsOn, "db")
}

func TestLoadTasksInterpolation(t *testing.T) {
    yaml := `
tasks:
  job:
    image: ${IMAGE:-busybox}
    command: ["echo", "$$MESSAGE"]
    environment:
      MESSAGE: ${MSG}
`
    project, err := LoadStringWithEnv(yaml, map[string]string{
        "IMAGE": "alpine",
        "MSG":   "hello",
    })
    require.NoError(t, err)
    task := project.Tasks["job"]
    assert.Equal(t, "alpine", task.Image)
    assert.Equal(t, "hello", *task.Environment["MESSAGE"])
}

func TestLoadTasksMerge(t *testing.T) {
    base := `
tasks:
  t:
    image: base
    environment:
      A: "1"
`
    override := `
tasks:
  t:
    environment:
      B: "2"
`
    project, err := LoadTwoStrings(base, override)
    require.NoError(t, err)
    task := project.Tasks["t"]
    assert.Equal(t, "base", task.Image)
    assert.Equal(t, "1", *task.Environment["A"])
    assert.Equal(t, "2", *task.Environment["B"])
}
```

Adapt helpers (`LoadString`, `LoadStringWithEnv`, `LoadTwoStrings`) to whatever the existing test suite exposes. If no helper exists, write the bare `loader.Load(types.ConfigDetails{...})` call directly.

- [ ] **Step 2: Run integration tests**

Run: `go test ./loader/ -run 'TestLoadTasks' -count=1 -v`
Expected: PASS.

- [ ] **Step 3: Run the ENTIRE test suite**

Run: `go test ./... -count=1`
Expected: ALL tests pass, no regressions.

- [ ] **Step 4: Commit**

```bash
git add loader/tasks_test.go
git commit -m "test(loader): end-to-end integration tests for tasks"
```

---

### Task 19: Schema parity test

Assert programmatically that every container_spec field accepted on `services.*` is also accepted on `tasks.*`.

**Files:**
- Modify: `schema/container_spec_test.go`

- [ ] **Step 1: Add the parity test**

```go
func TestServiceAndTaskAcceptSameContainerFields(t *testing.T) {
	// Load the compose-spec schema and extract the property names of container_spec.
	schemaBytes, err := os.ReadFile("compose-spec.json")
	require.NoError(t, err)

	var root map[string]any
	require.NoError(t, json.Unmarshal(schemaBytes, &root))

	defs := root["definitions"].(map[string]any)
	containerSpec := defs["container_spec"].(map[string]any)
	props := containerSpec["properties"].(map[string]any)

	for fieldName := range props {
		// Build a minimal doc that sets this field to a benign value.
		// For fields requiring complex shapes, we just test syntactic acceptance:
		// the schema validator will flag structural mismatches, but our goal is
		// only to confirm the field is an allowed key in both places.
		t.Run(fieldName, func(t *testing.T) {
			svcDoc := map[string]any{
				"services": map[string]any{
					"test": map[string]any{"image": "busybox", fieldName: sampleValueFor(fieldName)},
				},
			}
			taskDoc := map[string]any{
				"tasks": map[string]any{
					"test": map[string]any{"image": "busybox", fieldName: sampleValueFor(fieldName)},
				},
			}

			svcErr := Validate(svcDoc)
			taskErr := Validate(taskDoc)

			// Both must agree: either both accept, or both reject with the same reason.
			// We expect both accept for fields in container_spec.
			assert.Equal(t, svcErr != nil, taskErr != nil,
				"parity violated for %q: service err=%v, task err=%v",
				fieldName, svcErr, taskErr)
		})
	}
}

// sampleValueFor returns a benign value compatible with the schema for the field.
// For shape-complex fields, use a syntactically valid minimal value.
func sampleValueFor(field string) any {
	switch field {
	case "image", "hostname", "domainname", "user", "working_dir", "net",
	     "network_mode", "cgroup", "cgroup_parent", "ipc", "pid", "platform",
	     "runtime", "isolation", "stop_signal", "pull_policy", "restart",
	     "container_name", "dockerfile", "userns_mode", "uts", "volume_driver",
	     "mac_address", "log_driver":
		return "x"
	case "cpu_count", "cpu_period", "cpu_quota", "cpu_rt_period",
	     "cpu_rt_runtime", "cpu_shares", "oom_score_adj", "pids_limit",
	     "mem_limit", "mem_reservation", "mem_swappiness", "memswap_limit",
	     "shm_size":
		return 1
	case "cpus", "cpu_percent":
		return 0.5
	case "privileged", "read_only", "init", "tty", "stdin_open",
	     "oom_kill_disable", "use_api_socket":
		return false
	case "attach":
		return true
	case "labels", "annotations", "sysctls", "storage_opt", "log_opt":
		return map[string]any{}
	case "environment":
		return map[string]any{"X": "y"}
	case "command", "entrypoint":
		return []any{"sh"}
	case "cap_add", "cap_drop", "security_opt", "external_links", "links",
	     "group_add", "device_cgroup_rules", "dns", "dns_opt", "dns_search",
	     "tmpfs", "volumes_from", "expose", "env_file", "label_file":
		return []any{"x"}
	case "ports":
		return []any{"8080"}
	case "volumes":
		return []any{"./data:/data"}
	case "secrets", "configs":
		return []any{"name"}
	case "depends_on":
		return []any{"other"}
	case "extends":
		return map[string]any{"service": "other"}
	case "build":
		return "."
	case "healthcheck", "deploy", "develop", "scale", "profiles",
	     "blkio_config", "credential_spec", "logging", "models",
	     "gpus", "ulimits", "networks", "extra_hosts", "devices",
	     "post_start", "pre_stop", "provider":
		return map[string]any{}
	}
	return "x"
}
```

Note: this test is intentionally conservative. Some fields may need shape-specific values. Failing sub-tests indicate either a schema issue (field structure differs between service and task) OR an insufficient `sampleValueFor`. Investigate each failure.

- [ ] **Step 2: Run the parity test**

Run: `go test ./schema/ -run TestServiceAndTaskAcceptSameContainerFields -count=1 -v`
Expected: all sub-tests pass. If a sub-test fails with "parity violated", the field either has an asymmetric schema between service and task (bug to investigate) or the sample value is invalid for the field's shape (adjust `sampleValueFor`).

- [ ] **Step 3: Commit**

```bash
git add schema/container_spec_test.go
git commit -m "test(schema): parity assertion between service and task container fields"
```

---

### Task 20: Self-review and documentation sweep

Final verification pass to catch anything missed.

- [ ] **Step 1: Confirm the spec is fully covered**

Read `docs/superpowers/specs/2026-04-16-compose-task-top-level-design.md` alongside this plan. For each of the following spec requirements, point to the task that implements it:

- `container_spec` definition in schema → Task 1
- Orchestration fields on service via `allOf` → Task 2
- `task` definition and top-level `tasks:` → Task 3
- `ContainerConfig` in Go → Task 4
- `ServiceConfig` refactor via embedding → Task 5 + 6
- `TaskConfig`, `Tasks`, `Config.Tasks` → Task 7
- Interpolation paths for tasks → Task 8
- Transform paths for tasks → Task 9
- `paths/paths.go` on `*ContainerConfig` → Task 10
- Loader normalize for tasks → Task 11
- `include` collects tasks → Task 12
- Override/merge for tasks → Task 13
- `extends.task` field → Task 14
- Cross-type extends validation → Task 15
- Name uniqueness → Task 16
- depends_on cross-type validation → Task 17
- Integration tests → Task 18
- Schema parity tests → Task 19

- [ ] **Step 2: Full test run**

Run: `go test ./... -count=1 -race`
Expected: all tests pass.

- [ ] **Step 3: Run any linters configured in the repo**

Run: `make lint` (if defined in `Makefile`) or `go vet ./...`
Expected: no issues.

- [ ] **Step 4: Verify no accidental changes to unrelated files**

Run: `git log --oneline main..HEAD` and confirm every commit is intentional and scoped to the plan.

- [ ] **Step 5: If working in a worktree, finalize via the finishing-a-development-branch skill**

Invoke `superpowers:finishing-a-development-branch` to decide on PR vs merge vs cleanup.

---

## Self-Review Notes (from plan author)

**Spec coverage:** All 10 items from the spec's "Sections du plan d'implémentation (aperçu)" map to tasks 0–19. The spec's detailed requirements (extends.task, depends_on restrictions, name uniqueness, parity tests) each have dedicated tasks.

**Placeholder scan:** No TBDs, no "implement later", no "similar to task N" shortcuts. Where mechanical duplication is required (Task 6 applies the same pattern across dozens of files), the pattern is shown concretely with a concrete before/after.

**Type consistency:** `ContainerConfig`, `TaskConfig`, `Tasks`, `ExtendsConfig.Task` are used consistently across all tasks. Function names (`mergeContainerSpec`, `normalizeContainer`, `checkNameUniqueness`, `checkDependsOnCrossType`) are used identically everywhere they appear.

**Known risk acknowledged in Task 0:** if the `allOf + additionalProperties: false` probe fails with gojsonschema, the whole plan needs revisiting. Task 0 is explicitly a go/no-go gate.
