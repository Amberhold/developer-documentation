# CLI Client — the `amberhold` operator command

> Discovery-phase design. Authored from the `cli-client` openspec change.
> The decisions D-C1–D-C7 here fix how the `amberhold` command (module
> `amberhold/cli`, repo `nas/cli`) acts as a thin client over the v1 API:
> verb-first grammar, kind-to-path inference, the login → token → config
> flow, TLS trust against the management plane (ADR-0028), and output
> control. The API contract is the source of truth (`contracts/openapi/
> v1.yaml`, ADR-0019); the CLI is a pure consumer and adds no daemon or
> contract surface. Feature 6 (API + CLI) lives in `docs/architecture/
> 01-os-feature-map.md`; authentication is ADR-0020.

## 1. Purpose

The v1 API has been built controller-by-controller, but no external client
exercises it — only `core`'s own tests. `amberhold` is the operator-facing
daily tool and the first real consumer of the API, RBAC, and audit paths. It
is deliberately *thin*: it applies declarative desired-state files and reads
status, with zero client-side reconciliation logic. It is also the natural
harness for the Web-UI (feature 8), whose OIDC sign-in flow (ADR-0029) is
implemented but still has no browser client.

## 2. Goals / Non-Goals

**Goals:**
- Cover the entire v1 resource surface (all collection and singleton
  resources, all action endpoints) through one table-driven client.
- Operator ergonomics: `apply -f` as the write path; `get`, `delete`,
  `action` as the read/mutate/imperative paths.
- Feature-rich output: color, `-o table|json|yaml`, generated help and
  shell completion.
- A robust bootstrap: password login → one-time bearer token → config file,
  and TLS trust for the built-in CA default (ADR-0028).

**Non-Goals:**
- No changes to the v1 contract or the `core` daemon — the CLI is a consumer
  only.
- No OpenAPI codegen toolchain; the client is hand-written and thin.
- No OIDC for the CLI — OIDC (ADR-0029) is a browser flow for the Web-UI;
  the CLI authenticates with password sessions + API tokens (ADR-0020).
- No full per-field imperative flags; declarative `apply -f` is the write
  path.
- No published SDK beyond the `amberhold` binary.

## 3. Command surface

Verb-first, kubectl-style. The resource set is the full v1 surface: `pools`,
`disks`, `datasets`, `zvols`, `snapshots`, `schedules`, `file-shares`,
`block-shares`, `apps`, `backups`, `users`, `roles`, `sessions`, `tokens`,
`audit`, and the singletons `network`, `updates`, `telemetry`,
`certificates`, `oidc`.

```
amberhold apply -f pool.yaml            # PUT/PATCH desired state; path from kind
amberhold apply -f network.yaml         # singleton write
amberhold get pools                     # table; -o json|yaml for scripting
amberhold get pools <id>                # spec + status envelope
amberhold delete disks <id>             # delete by kind + id
amberhold delete -f pool.yaml           # delete by kind + metadata.name
amberhold action pools <id> scrub       # imperative action (ADR-0002)
amberhold action disks <id> replace
amberhold action updates trigger|rollback

amberhold login --server https://nas    # password -> session -> token
amberhold logout
amberhold token create / token revoke <id>
amberhold config set server https://nas
amberhold version
amberhold completion bash|zsh|fish
```

The `apply`/`delete -f` write path resolves the resource path from the file's
`kind` via a static `kind → path` table mirrored from the contract (the same
mapping `core` uses in `singularKind`, `core/internal/api/handlers.go`). A
file without `kind`, or with an unknown kind, fails before any request.

## 4. Architecture

```
+--------------------------------------------------------------+
|  operator machine                                   appliance |
|                                                            |
|  amberhold (cli)      HTTPS + bearer          core daemon    |
|  +--------------+    (built-in CA,          +--------------+ |
|  | commands     |    ADR-0028)              | /v1/*        | |
|  | (cobra)      |----+   +----------------+ | admission    | |
|  +--------------+    |   | client (thin)  | | (ADR-0018)   | |
|  | schema table |    +-->|  bearer        | | controllers  | |
|  | kind -> path |        |  cookie jar    | | /v1/sessions | |
|  | verbs/actions|        |  TLS roots     | | /v1/tokens   | |
|  +--------------+        |  error decode  | +--------------+ |
|  | config       |        +----------------+                  |
|  | output/color |                                             |
|  +--------------+                                             |
+--------------------------------------------------------------+
```

Layers, all in module `amberhold/cli`:

| Path | Purpose |
|------|---------|
| `cmd/amberhold` | Entrypoint; root command + persistent flags |
| `internal/schema` | Resource table: kind, path, verbs, action list — derived from `contracts/openapi/v1.yaml` |
| `internal/client` | Thin HTTP layer: bearer injection, transient cookie jar, TLS root pool, API-error decoding |
| `internal/commands` | Cobra commands driven by the schema table (verbs + auth + config + completion) |
| `internal/config` | `~/.config/amberhold/config` read/write (0600) + precedence |
| `internal/output` | Table/JSON/YAML renderers + color control |

## 5. Auth flow

`POST /v1/sessions` is public (contract `security: []`); `POST /v1/tokens`
and every resource route require a confirmed principal. Login is therefore a
two-step chain:

```
login --server https://nas
  | 1. prompt password (hidden, no echo)
  v
POST /v1/sessions  (username, password)     -> session cookie (in-memory only)
  v
POST /v1/tokens    (cookie)                 -> secret shown once
  v
write ~/.config/amberhold/config (0600)     -> server, token
```

- The session cookie lives in a transient, in-memory cookie jar used only to
  mint the token; cookies are never persisted.
- The token secret (`<lookupId>.<random>`, ADR-0020) is stored in the config
  file with owner-only permissions; it is never echoed.
- `logout` removes the token from the config file (and may revoke it).
- `token create` prints the secret exactly once; `token revoke <id>` kills a
  leaked credential.

## 6. Config and precedence

Resolution order: explicit flags → environment variables (`AMBERHOLD_SERVER`,
`AMBERHOLD_TOKEN`, `AMBERHOLD_OUTPUT`, `AMBERHOLD_COLOR`) → config file
(`~/.config/amberhold/config`). Missing server fails before any request with
a "run `amberhold login` or set `--server`/`AMBERHOLD_SERVER`" hint. The
config file is written 0600 and never read from a world-writable location.

## 7. TLS trust

The management plane serves HTTPS only (ADR-0028). The v1 default trust is
the built-in CA, so a stock client will fail verification against a fresh
appliance:

- `--ca-file <pem>` appends a PEM root to the per-request pool — pin the
  appliance's built-in CA.
- `--insecure` disables verification with a visible warning (development
  only).
- No plain-HTTP fallback; an `http://` server URL is rejected.
- On verification failure during `login`, the CLI prints guidance (fetch the
  built-in CA, or pass `--ca-file`).

## 8. Output, color, help, completion

- `-o table|json|yaml` — `table` is the human default; `json`/`yaml` print the
  full envelope undecorated for scripting. Invalid values are rejected.
- `--color auto|always|never`, default `auto`; `NO_COLOR` is honored.
- Every command and flag has help (Cobra).
- `completion bash|zsh|fish` emits shell completion scripts.

## 9. Error handling

Non-2xx responses decode the API error envelope (`code`, `message`) and exit
non-zero with the message. A 401 on a resource verb prints a re-login hint; a
403 surfaces the RBAC capability denial verbatim (ADR-0018). Contract
resources whose controller does not exist on the target daemon (e.g.
`updates`) 404 with a clear "resource not supported by this server" error —
no special-casing.

## 10. Decisions

### D-C1: `amberhold/cli` module, `amberhold` command
New Go module `amberhold/cli` in the `cli` submodule; binary `amberhold`
(the daemon keeps `amberhold-core`). Matches "one module per repo" and the
"thin client over the API" framing.

### D-C2: Verb-first grammar with kind-to-path inference
`apply`, `get`, `delete`, `action`, `login`, `logout`, `token`, `config`,
`version`, `completion`. `apply`/`delete -f` infer the path from `kind`.
Resource-first grammar was rejected as weaker for scripted `apply -f`.

### D-C3: Hand-written table-driven client
`schema` table + thin `client` HTTP layer + Cobra commands. No codegen;
a conformance test parses `contracts/openapi/v1.yaml` and asserts every
kind/path/verb/action in the schema table exists in the document and
vice-versa. oapi-codegen was rejected: it would guarantee conformance but
adds a toolchain the project doesn't have and heavy typed output.

### D-C4: Cobra + Viper + minimal renderers
Cobra (help, completion, subcommands), Viper (flag > env > file
precedence), `fatih/color` (ANSI color), `olekukonko/tablewriter`
(tables), `gopkg.in/yaml.v3` (YAML in/out), `golang.org/x/term` (hidden
password). Consistent with `core`'s curated dependency style.

### D-C5: Two-step login, one-time secret, 0600 config
Password → session (in-memory cookie) → token (secret once) → config.
Sessions are never persisted; token secrets are written 0600 and never
echoed. Direct username/password token issuance was rejected: `POST
/tokens` requires a confirmed principal, so the session hop is unavoidable.

### D-C6: TLS trust via `--ca-file` and `--insecure`
HTTPS only; `--ca-file` pins the built-in CA; `--insecure` is an explicit,
warned dev escape hatch. No plain-HTTP fallback.

### D-C7: Error surfacing
API error envelope decoded and printed; 401 → re-login hint; 403 → RBAC
denial verbatim; unsupported resource → clear 404 message.

## 11. Risks / Trade-offs

- **Hand-written client drifts from the contract** → conformance test against
  `contracts/openapi/v1.yaml` in the repo's test suite.
- **Version skew with the daemon** → generic verbs degrade to a clear
  "resource not supported" error; no special-casing per resource.
- **Built-in CA bootstrap is a chicken-and-egg** → `--ca-file` + explicit
  `login` guidance; `--insecure` remains an explicit operator choice.
- **Secrets in config** → 0600 on write, no echo; `AMBERHOLD_TOKEN` is the
  documented automation path.

## 12. Open Questions

- Table columns for `get` (cosmetic; decided during implementation without
  touching the spec).
- Per-action verb aliases (e.g. `amberhold scrub <id>` sugar) — deferred; the
  generic `action` verb (D-C2) is the v1 surface.