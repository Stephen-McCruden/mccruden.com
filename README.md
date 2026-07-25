# mccruden.com

Stephen McCruden's infrastructure engineering portfolio and technical blog.

The site is built with Astro, published as an unprivileged NGINX container to
GHCR, and deployed to Kubernetes through Flux.

## Local development

Requires Node.js 22.12 or newer.

```sh
npm ci
npm run dev
```

The local site is available at `http://localhost:4321`.

Create a production build with:

```sh
npm run build
npm run preview
```

## Writing

Posts are Markdown files in `src/content/blog`. Copy `_template.md` to a
descriptive filename, fill in the frontmatter, and write the post in Obsidian
or any Markdown editor.

Keep `draft: true` while writing. Drafts are excluded from the site and RSS
feed. Set `draft: false`, run a local build, and open a pull request when the
post is ready to publish.

## Container publishing

Pushes to `main` and feature branches publish immutable images to:

```text
ghcr.io/stephen-mccruden/mccruden.com
```

Each image receives a commit SHA tag. The default branch also receives
`latest`. Kubernetes deployments should remain pinned to an immutable digest.

## Deployment environments

- `preview.mccruden.com` is the staging environment for branch builds and
  design review.
- `mccruden.com` will be the public production environment promoted from
  `main`.

Infrastructure manifests live separately in the
[`Stephen-McCruden/homelab`](https://github.com/Stephen-McCruden/homelab)
repository.
