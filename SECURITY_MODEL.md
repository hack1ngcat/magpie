# Magpie security model

## Scope and evidence

This model covers the whole repository: desktop, headless/server deployments, plugins, the public website and release automation. It consolidates the source and subsystem references cited below. Documented controls describe intended behavior; each runtime path still needs verification.

The owner's confirmed priorities and risk decisions are distinguished from architecture observations. Assumptions inferred from code do not establish accepted risks or justify changing a finding's severity.

## System overview and components

Magpie connects local coding agents to model providers and subscription accounts through a Go gateway. It has a desktop UI, a browser management UI and headless/container deployments. Plugins supply providers, middleware and tools. Agent configuration, credentials, usage and optional conversation data live on the host. Encrypted backup and sync connect machines. A separately deployed Cloudflare Worker serves the public website, release downloads and plugin metadata.

| Stable component | Code covered | Boundary and responsibility |
| --- | --- | --- |
| Gateway | [`internal/gateway/`](internal/gateway/), [`internal/access/`](internal/access/), [`internal/netguard/`](internal/netguard/), [`internal/upstream/`](internal/upstream/), [`internal/gwbudget/`](internal/gwbudget/), [`internal/redact/`](internal/redact/), [`internal/pii/`](internal/pii/), gateway-facing built-in [`plugins/`](plugins/) | Local and remote agent HTTP/MCP calls to privileged provider requests, fetched URLs, masking and key/spend enforcement. |
| Management UI | [`internal/gui/`](internal/gui/), desktop and web entry points | Desktop webview and authenticated browser administration to local settings, credentials, files and processes. |
| Providers and accounts | [`internal/provider/`](internal/provider/), provider-specific packages, [`internal/mcpauth/`](internal/mcpauth/) | Vendor API/OAuth services and locally held API keys, subscription sign-ins and refresh credentials. |
| Plugin runtime | [`internal/plugin/`](internal/plugin/), [`internal/pluginrt/`](internal/pluginrt/), [`internal/middleware/`](internal/middleware/), plugin-facing built-ins | Third-party package installation and Bun execution; separate restricted plugin HTML pages and panels. |
| Agent wiring and tools | [`internal/agent/`](internal/agent/), [`internal/agentenv/`](internal/agentenv/), [`internal/library/`](internal/library/), [`internal/memory/`](internal/memory/), [`internal/sessions/`](internal/sessions/), [`internal/cli/`](internal/cli/), [`internal/proc/`](internal/proc/) | Host agent configuration, private conversation logs, instructions and MCP servers; local subprocess, browser and filesystem capabilities. |
| Sync and backup | [`internal/backup/`](internal/backup/), [`internal/davsync/`](internal/davsync/), associated remote backup backends | Sensitive local configuration and optional sign-ins crossing file, WebDAV, S3 and GitHub storage boundaries. |
| Public site and releases | [`site/`](site/), [`.github/workflows/`](.github/workflows/), [`build/`](build/), [`Dockerfile`](Dockerfile), [`internal/update/`](internal/update/), install packages | Public Cloudflare endpoints, download mirrors and software supply chain to installed executable code. |

If a finding spans components, tag the component owning its vulnerable entry point. For other paths, map by the subsystem they serve and explain an ambiguous mapping.

## Entry points and reachability

- The gateway defaults to `127.0.0.1:3425` ([`internal/gateway/gateway.go`](internal/gateway/gateway.go), `DefaultAddr`). Local agents use the conventional `magpie` bearer string; it is not a secret establishing local-user identity. LAN sharing, remote magpie, tunnels and server deployments change reachability. The Docker image explicitly sets `MAGPIE_ADDR=0.0.0.0:3425` and exposes 3425/3430 ([`Dockerfile`](Dockerfile)). Actual firewall, proxy, TLS and ingress policies are unknown.
- Model endpoints (`/v1/...`, `/v1beta/...`, native backend paths) and `/mcp/...` accept agent requests. [`internal/gateway/lan.go`](internal/gateway/lan.go), [`cors.go`](internal/gateway/cors.go) and [`internal/access/`](internal/access/) enforce remote keys, origin/host checks and per-key restrictions. Remote keyed clients must be considered distinct from the owner.
- The browser management page uses a run key and cookie ([`internal/gui/web.go`](internal/gui/web.go), `StartWeb`, `webGuard`, `webOwnPage`). Browser access can be enabled on loopback or the network; configured LazyCat sign-in introduces a separate ingress-header trust boundary. The desktop API is in-process. See [`docs/subsystems/gui-shell.md`](docs/subsystems/gui-shell.md).
- Local CLI commands, agent configuration/library imports, custom deep links, backup restore, plugin install/update, plugin HTML bridges and subscription sign-in callbacks consume data crossing into local privileges.
- MCP routing may start local command servers. The documented contract excludes host-only services from remote callers and restricts held keys to external HTTP servers; unrestricted gateway keys can reach other enabled command servers with the host user's rights ([`docs/subsystems/mcp-router.md`](docs/subsystems/mcp-router.md), Constraints). This is intended behavior to evaluate, not an acceptance of arbitrary privilege escalation.
- Public website requests reach [`site/worker.js`](site/worker.js), including update/download endpoints, `/api/icon`, plugin metadata/statistics and documented redirects/relays. Scheduled workers use KV, R2 and D1 bindings in [`site/wrangler.jsonc`](site/wrangler.jsonc). Release workflows and plugin/updater downloads cross a software-supply-chain boundary.

## Assets and sensitivity

The owner explicitly selected credentials and private conversations as the highest-priority assets. Other asset sensitivities below are derived from the code and remain in scope:

- Provider API keys, OAuth access/refresh tokens, subscription sign-ins, gateway keys, backup passphrases and sync-storage credentials: secret, with account and billing consequences.
- Prompts, replies, conversation history, memory, library instructions and working-directory/agent context: potentially confidential user or source-code data. Review each storage/export path on its own; do not assume all usage rows contain bodies.
- Host files, executable configuration and processes: integrity and execution authority of the OS user, including MCP servers and installed plugins.
- Provider/model/account restrictions, budgets and usage records: authorization, billing integrity and availability.
- Published releases, updater metadata, plugin package resolution and installer artifacts: integrity affecting downstream installations.

The owner's priority applies to verified paths reaching credentials or private conversations. It does not make every finding critical or justify raising an unverified or hard-to-exploit finding. Other proposed sensitivities alone do not raise scanner ratings.

## Attacker profiles

The owner explicitly excluded no attacker or deployment mode and accepted no listed risk. Keep all plausible attackers in consideration: unauthenticated web/LAN clients when a listener is reachable; malicious websites interacting with localhost or a shared origin; legitimate gateway-key holders attempting to escape restrictions; hostile upstream responses and fetch destinations; malicious or compromised packages/update sources; and local users/processes with access different from the owner. Actual deployment-specific exposure and installed-package trust remain unknown; unknowns are not exclusions.

## Existing controls and their limits

- Gateway origin/host and LAN authorization guards, named gateway keys, model/account restrictions and budget checks are in [`lan.go`](internal/gateway/lan.go), [`cors.go`](internal/gateway/cors.go), [`native.go`](internal/gateway/native.go) and [`internal/access/`](internal/access/); these must be traced by the scanner for each entry point rather than assumed universal.
- The browser management guard separates its key from gateway keys and uses `webOwnPage`/Go cross-origin protection ([`internal/gui/web.go`](internal/gui/web.go); [`docs/subsystems/gui-shell.md`](docs/subsystems/gui-shell.md)). Proxy and LazyCat trust depend on the actual deployment.
- URL fetching has a common public-address classification policy, including mapped and translated addresses ([`internal/netguard/netguard.go`](internal/netguard/netguard.go)). Callers still choose resolution, redirects, address pinning and proxy behavior; the helper alone does not prove SSRF resistance.
- Plugin HTML pages and panels have a scripts-only iframe sandbox, a run-bound path token, CSP, declared read permissions and a source-checked message bridge ([`internal/gui/pluginpage.go`](internal/gui/pluginpage.go), [`pluginpages.js`](internal/gui/assets/pluginpages.js), [`internal/plugin/page.go`](internal/plugin/page.go); [`docs/subsystems/plugin-runtime.md`](docs/subsystems/plugin-runtime.md)). This HTML isolation must not be assumed to sandbox server-side Bun plugins ([`internal/plugin/host.go`](internal/plugin/host.go), [`host.js`](internal/plugin/host.js)).
- Backup files use AES-256-GCM with a PBKDF2-SHA256-derived key ([`internal/backup/backup.go`](internal/backup/backup.go)). Sign-ins are opt-in. Local automatic backups and saved settings have their own scope and file-permission rules; encryption of an exported file does not establish encryption of all local state. See [`docs/subsystems/sync-backup.md`](docs/subsystems/sync-backup.md).
- The container uses a nonroot UID and persistent `/config` volume ([`Dockerfile`](Dockerfile)). This does not replace network authentication or determine host volume permissions.
- Repository documentation distinguishes retired built-in providers from plugins that now serve them ([`AGENTS.md`](AGENTS.md), [`docs/subsystems/provider-plugins.md`](docs/subsystems/provider-plugins.md)). The scan covers the Magpie host, migration and integration code present here; external community-plugin implementation is outside the submitted repository set.

## Infrastructure evidence and unknowns

Infrastructure evidence is in this repository: [`Dockerfile`](Dockerfile), [`site/wrangler.jsonc`](site/wrangler.jsonc) and [`.github/workflows/`](.github/workflows/). No separate infrastructure repository was used. Production reverse proxies, firewall rules, cloud IAM permissions, deployment secrets, actual listener settings and operator-specific configurations are not established by this checkout.

## Security priorities, accepted risks and ranking policy

The owner confirmed whole-repository scope, credentials and private conversations as the highest-priority assets, and no excluded attackers, deployment modes or accepted risks.

Start from the scanner's severity and its verified exploit preconditions. Change it only for an explicit owner-provided fact about attackers, assets or accepted risk, including documented security contracts where they actually establish such a fact. Code-derived assumptions move nothing. Never raise a hard-to-exploit or unverified finding. The owner rules nothing out, so no finding is lowered for accepted risk; higher priority is rare and requires a scanner-confirmed path to the credentials or private conversations the owner prioritized, with exploitability supporting it. Explain likelihood and impact without inventing a risk acceptance.

## Finding tags

The owner requested `Component`, `CVE`, `Network exposure`, `Owning team` and `Compliance scope`.

- `Component`: the stable component name above, assigned by the vulnerable entry point.
- `CVE`: an actual CVE identifier when the finding identifies one; do not invent an identifier for an application bug.
- `Network exposure`: reachability and required access established in the finding, such as internet-facing, LAN, localhost/local process or authenticated gateway client; retain any deployment preconditions.
- `Owning team`: a team supported by explicit ownership evidence; do not infer a team from a directory name.
- `Compliance scope`: a clearly applicable compliance framework/control supported by the finding and system context; no framework has been designated by the owner, so omit unsupported labels.

## Open questions

Actual listener/proxy/firewall configurations, installed-plugin trust, owning teams and applicable compliance frameworks are unknown. These unknowns are not treated as accepted risk. The owner can supply or correct them at any time.
