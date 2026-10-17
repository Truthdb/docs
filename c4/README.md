# C4 model of the TruthDB workspace

[LikeC4](https://likec4.dev) model covering every repo in the Truthdb organisation: the database, the installer chain, the build and release workflows, the apt repository, the orchestrator, the website, and the documentation and brand repos.

## Viewing

- VS Code: install the LikeC4 extension and open any `.c4` file; the preview button renders its views.
- Browser: `npx likec4 start docs/c4` from the workspace root.
- Check after editing: `npx likec4 validate docs/c4`.

## Files

| File | Contents |
|---|---|
| `specification.c4` | Element kinds, `repo_*` tags (which repo an element lives in), deployment node kinds |
| `model.c4` | People and external systems (GitHub, Firebase, kernel.org, Debian mirror, SQL Server clients) |
| `truthdb.c4` | The database: server, CLI, bench, database file; server crates and their modules |
| `installer.c4` | The installer as it runs: GRUB, kernel, initramfs `/init`, `truthdb-installer` steps, payload, target disk |
| `delivery.c4` | CI/CD workflows, release artifacts, apt repository, orchestrator and local build scripts |
| `website.c4` | Website SPA, and the docs, assets and `.github` repos |
| `deployment.c4` | Bare-metal appliance, apt-installed host, primary and standby, local Docker, Firebase Hosting |
| `views.c4` | All views |

## Views

- Landscape: `index`
- Database: `truthdbContainers`, `serverCrates`, `engineComponents`, `tdsComponents`, `relationalComponents`, `coreComponents`
- Database flows: `nativeQuery`, `tdsSession`, `replicationStream`
- Installer: `installerContainers`, `installerSteps`, `installFlow`
- Release: `deliveryOverview`, `installerReleaseChain`, `releaseIso`, `aptPublishing`
- Tooling: `toolingOverview`, `orchestratorComponents`
- Website and docs: `websiteContainers`, `websiteComponents`, `knowledgeOverview`
- Deployment: `deployAppliance`, `deployAptHost`, `deployReplicated`, `deployDocker`, `deployWebsite`

Element names, file paths, ports and workflow names are taken from the repos as of 2026-10-18 (truthdb `f54baca`). When a crate, workflow or listener changes, update the matching element here.
