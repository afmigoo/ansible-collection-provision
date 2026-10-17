# Ansible Role Conventions

General standards for creating Ansible roles, informed by the gold-standard
nginx role in the powerops.services repo (`roles/nginx`); clone it if you need
the example.

## Variable Naming (3-tier)

- **Configurable vars**: `<prefix>_*` — settable by the user, defined in
  `defaults/main.yml` or passed via `--extra-vars`. The prefix is the role
  name with hyphens replaced by underscores (e.g. `nginx_enable`,
  `nginx_servers`).

- **Internal/utility vars**: `_<prefix>_*` — programmatically set within the
  role (`set_fact`, rendered/computed data), not meant for user override.

- **Local task vars**: `__<prefix>_*` — loop vars (used with
  `loop_control.loop_var`), `register:` vars, task-scoped temporaries. Never
  used outside the task that defines them.

## Directory Structure

```
roles/<name>/
  defaults/
    main.yml               # All configurable defaults (non-sensitive)
  meta/
    main.yml                # galaxy_info + dependencies
    argument_specs.yml      # Typed argument specs, per entry point
  tasks/
    main.yml                # Top-level entry point, gated by <prefix>_enable
    preparation.yml         # Assertions, fact gathering, rendering
    verification.yml        # Post-deploy verification (standalone entry point)
    _<category>/            # Private subtasks (underscore-prefixed dirs)
      *.yml
  templates/                # Jinja2 templates, mirrors target filesystem layout
  files/                    # Static files to copy to target hosts
  vars/                     # Test/inventory variable overrides
    test.yml                # Full-role test vars
    test-<scenario>.yml     # Scenario-specific test vars
  README.md
```

See the nginx role in powerops.services (`roles/nginx`) for a
concrete example. Its internal subdirectories (`tasks/_main/`,
`tasks/_preparation/`, `tasks/_cert/`) follow its own domain-specific
organization — tailor yours to the role's needs, not a copy of nginx's.

## Check Mode

Roles **must** support `--check` mode. Every task should be safe to run in
dry-run:
- Read-only probes (`command`/`shell` whose result later tasks use) need
  `check_mode: false` and `changed_when: false`; otherwise they are skipped
  in `--check` and their registered result has no stdout. Commands that
  change state need `creates:`/`removes:` or a real `changed_when:`.
- Template and file tasks are natively check-mode safe.
- Guard destructive tasks with `when: not ansible_check_mode`.

## Entry Points

- Every entry point (`main`, `verification`, etc.) is declared as a separate
  section in `meta/argument_specs.yml` with its own `short_description`,
  `description`, `author`, and `options` block.
- `main` is the primary entry point; `verification` is a standalone
  post-deploy diagnostic that can be run independently.

## Defaults Pattern

- `<prefix>_enable: false` — master on/off gate (defaults to `false` so the
  role is opt-in).
- `<prefix>_enable_<subsystem>: true` — per-subsystem toggles (defaults to
  `true` so subsystems are enabled when the role is enabled).
- There is no universal rule for baking defaults into Jinja2 templates via
  `| default()` vs. letting the template fail on missing values. Both are
  valid — use whichever makes sense for the specific variable. Baked defaults
  are convenient when users only need to override a subset of fields;
  failing on missing values is appropriate when the value is always required
  and forgetting it would be a configuration error.

## Task Organization

- **Public task files** live at the role root level: `main.yml`,
  `preparation.yml`, `verification.yml`.
- **Private subtask files** live in underscore-prefixed subdirectories
  (e.g. `_preparation/`, `_main/`). Name these after the domain concern
  they handle.
- Use `import_tasks` for always-run subtasks, `include_role` for cross-role
  dependencies, `include_tasks` for conditional/dynamic inclusion.
- Wrap related tasks in `block:` for logical grouping and error isolation.

## Tags

- Use tags meaningfully — there is no rigid naming scheme. Tag tasks by
  logical concern so operators can selectively run or skip phases of the
  role.
- `always` on critical preparation tasks that must never be skipped.
- Don't forget to apply tags to every task; untagged tasks can't be
  selectively skipped.

## Service Restarts/Reloads

- Use **Ansible handlers** for service restarts and reloads. Trigger with
  `notify:` from tasks that modify configuration.
- Do NOT accumulate a manual `_<prefix>_reload_needed` variable across tasks
  — handlers are the idiomatic Ansible mechanism for deferred restarts.

## Validation

- `ansible.builtin.assert` for pre-conditions (required packages, facts
  populated, version requirements), post-conditions, and internal invariants.
- `failed_when: false` + separate assert for non-fatal command checks —
  collects diagnostics without aborting the playbook, then presents a
  meaningful assertion message.
- `quiet: true` on looped assertions that produce repetitive output.

## Jinja2 Templates

- Template directory structure mirrors the target filesystem layout.
- Always use `lstrip_blocks: true` for clean rendered output.

## Cross-Role Dependencies

- Declare in `meta/main.yml` via `dependencies:` for automatic inclusion.
- Use `include_role` in tasks with explicit `vars:` when you need fine-grained
  control over how variables are passed.
