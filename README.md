# mccruden.com

Stephen McCruden's infrastructure engineering portfolio and technical blog.

The site is built with Astro, packaged as an unprivileged NGINX container, and
deployed to Kubernetes through GitHub Actions and Flux.

## Local development

Requires Node.js 22.12 or newer.

```bash
npm ci
npm run dev
```

Create the production build with:

```bash
npm run build
```

## Writing

Posts are ordinary Markdown files under `src/content/blog/`. Copy
`_template.md`, rename it with a descriptive slug, and write in Obsidian or any
Markdown editor.

```text
src/content/blog/rebuilding-kubernetes-from-code.md
```

Keep `draft: true` while writing. Set `draft: false` when the article should be
included in the generated site.

## Automated environments

Every non-`main` branch publishes to the fixed preview environment. The
`main` branch publishes to production.

| Git branch | Image channel | Website |
|---|---|---|
| Any non-`main` branch | `preview-<run>-<commit>` | `preview.mccruden.com` |
| `main` | `production-<run>-<commit>` | `mccruden.com` |

GitHub Actions builds the container and publishes a unique image tag to GHCR.
Flux selects that channel's newest image, records its immutable digest in the
homelab Git repository, and rolls out the corresponding Kubernetes Deployment.

No routine Docker command, digest lookup, Kubernetes edit, or Flux
reconciliation is required.

See [docs/PUBLISHING.md](docs/PUBLISHING.md) for the complete preview,
production, verification, and rollback procedure.

## Runtime design

- multi-stage container build
- unprivileged NGINX runtime
- read-only Kubernetes root filesystem
- separate preview and production Deployments
- immutable image digest deployment
- rolling updates with health probes
- Git history for every release
- no database or persistent volume
