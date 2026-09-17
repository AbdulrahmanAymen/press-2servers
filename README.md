# press-ubuntu24-two-server

Self-hosted [Frappe Press](https://github.com/frappe/press) on **Ubuntu 24.04**, running on
**two servers** instead of the standard four — without AWS Route 53 or cloud provider API access.

📖 **[Read the full guide → GUIDE.md](GUIDE.md)**

---

## What this covers

The official Press guides assume four servers, a real AWS account with Route 53, and IAM
credentials that let Press provision machines for you. This repo documents the setup where you
have neither — two self-provisioned servers, any hosting provider, manual DNS.

It also documents every Ubuntu 24.04 incompatibility encountered, in the order they occur, so
you can apply the fixes up front instead of hitting them one failed playbook at a time.

## Architecture

| Server | Roles | Spec |
|---|---|---|
| **Server 1** | Press dashboard + Proxy Server | 6 vCPU / 12 GB |
| **Server 2** | App + Database (Unified Server) | 6 vCPU / 12 GB |

Press has four roles but does not require four machines. The App and Database roles are merged
using Press's own `unified_server.yml` playbook, which runs both role sets in a single
coordinated Ansible play. The Proxy stays on Server 1 alongside the dashboard, which is safe
because the dashboard runs as user `press` under `/home/press/frappe-bench` (a plain bench with
no agent), while the Proxy runs as user `frappe` under `/home/frappe/agent` — different users,
different paths, no collision.

## The fixes, at a glance

| Area | Issue |
|---|---|
| Node | Press `develop` needs Node 20+, guides say 18 |
| MariaDB | `unix_socket` auth breaks `bench`; the 10.6 Rackspace repo has no Noble build |
| Ansible | `init_plugin_loader()` is never called — every `Setup Server` fails before SSH |
| Python deps | `ansible`, `stripe`, `sqlparse`, `cryptography`, `pyOpenSSL`, `setuptools` pins |
| TLS | `provider = "Let's Encrypt"` triggers a Route 53 path that permanently flips certs to `Failure` |
| certbot | Defaults to ECDSA; Press validates against RSA and rejects the key |
| SSH | Ubuntu names the unit `ssh` not `sshd`; AWS AMIs ship a forced-command root lock |
| Users | Default cloud user holds UID 1000, which Press needs for `frappe` |
| nginx | Agent template uses a VTS directive whose module isn't packaged for Noble |
| Agent | Resolves a `python3.14` path that doesn't exist, from an open `requires-python` range |

Full detail, with the reasoning and the code behind each one, is in
**[GUIDE.md](GUIDE.md)**.

## Quick start

1. Two Ubuntu 24.04 servers, static IPs, root access
2. A domain you control DNS for
3. Work through [GUIDE.md](GUIDE.md) in order — the parts are sequential and the fixes are
   positioned where they're needed

## Status

Working through to Deploy Candidate. Two items remain local patches rather than upstream fixes —
see [Known open issues](GUIDE.md#known-open-issues).

## Contributing

If you hit something this guide doesn't cover, or a fix here becomes unnecessary after a Press
update, open an issue or PR.
