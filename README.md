# DragonForge Linux

Teach IT on infrastructure that behaves like infrastructure.

DragonForge is a Debian 13 (Trixie) based distribution for IT classrooms. It gives a course one
base to standardize on and a set of role builds to vary from, so that a networking module, a
cybersecurity module, and a server module all run on the same platform instead of three lab
images nobody maintains.

Standardize the base. Vary the build.

## Why this exists

Classroom labs accumulate distributions. One course ships Ubuntu, another ships Kali, a third
ships a hand-rolled image that worked in 2019. Students spend the first week of every term
learning a different desktop, instructors spend the first week debugging images instead of
teaching, and nobody can answer the question "what are we actually running."

DragonForge is one answer to that question. A shared base, role-based metapackages on top of it,
and a repository you control.

## Roles

Role metapackages select a workload without replacing the base:

| Role | Metapackage | What it brings |
| --- | --- | --- |
| Server | `dragonforge-server` | Server administration tooling, service management, logging |
| Desktop | `dragonforge-desktop` | Classroom desktop environment and base configuration |
| Web stack | `dragonforge-server-webstack` | nginx, application runtime, database, supporting tools |
| Cybersecurity | `dragonforge-cyber` | Analysis and assessment tooling for defensive coursework |
| Networking | `dragonforge-network` | Packet analysis, configuration, and troubleshooting tools |
| Programming | `dragonforge-programming` | Toolchains and editor support for programming coursework |

The base package (`dragonforge-base`) is the floor. Everything else layers onto it.

## Repository layout

The project runs its own APT repository rather than pointing students at a third party:

```
testing/dists/trixie/main/binary-amd64/Packages
testing/pool/main/<letter>/<package>/
unstable/
```

- `testing` carries what has been built and verified.
- `unstable` carries what is still moving.
- A client sources line pointing at a DragonForge repository expects `trixie` as the suite.

## Install

Installation scripts and the current repository URL live with the build infrastructure, not in
this repository. This repository holds the project, its documentation, and the record of what
was built.

## Status

DragonForge is under active development in a working classroom. The server system was delivered
for the 2026-04-24 milestone. Current work is testing, student onboarding, and CI for package
builds. Desktop packaging follows.

**This is a live build, not a released distribution.** Package names, repository paths, and
metapackages are still changing. If you are evaluating DragonForge for a lab, open an issue and
describe the environment you would be running it in.

## Requirements

- Debian 13 (Trixie) as the base
- amd64 for the current builds
- Network access to a DragonForge repository during provisioning

## Contributing

Issues are the front door. Describe the environment, what you expected, and what happened. For
package problems, include the package name, version, and the exact command that failed.

Build and repository changes are coordinated through issues before code.

## License

MIT. See [LICENSE](LICENSE).

## Links

- Repository: <https://repo.dragonforgelinux.org/>
- Project writeups: <https://crperdue.com>
