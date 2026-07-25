# Publishing mccruden.com

This site has two automated deployment channels:

- every non-`main` branch updates `preview.mccruden.com`
- `main` updates `mccruden.com`

The container build and Kubernetes rollout happen automatically after the Git
push. The commands below are the complete routine workflow.

## Create a blog post

Start from an up-to-date `main` branch and create a short-lived content branch:

```bash
git switch main
git pull --ff-only
git switch -c post/<article-slug>
```

Copy the template:

```bash
cp src/content/blog/_template.md \
  src/content/blog/<article-slug>.md
```

Edit the frontmatter and article body. Use `draft: true` while the post is not
ready to appear. Before publishing to preview, set:

```yaml
draft: false
```

## Deploy to preview

Commit and push the Markdown file:

```bash
git add src/content/blog/<article-slug>.md
git commit -S -m "Publish <article title>"
git push --set-upstream origin post/<article-slug>
```

That push performs the rest of the preview release:

1. GitHub Actions builds the Astro site.
2. The workflow publishes a unique `preview-*` container image to GHCR.
3. Flux discovers the image and resolves its immutable digest.
4. Flux records the selected image in the homelab repository.
5. Kubernetes rolls out the preview Deployment.

Review:

```text
https://preview.mccruden.com
```

A typical rollout completes within a few minutes. The preview ingress sends a
`noindex` response header so search engines do not index staging content.

## Promote to production

Open a pull request from the content branch into `main`. Review the preview,
then merge the pull request.

The merge triggers the production channel automatically:

1. GitHub Actions builds the exact `main` commit.
2. The workflow publishes a unique `production-*` image.
3. Flux records the selected immutable digest in the production Deployment.
4. Kubernetes performs a zero-unavailable rolling update.

Verify:

```text
https://mccruden.com
```

Delete the content branch after merge. Your next article starts from the latest
`main`.

## Direct publishing

For a small correction that does not need preview review, committing the
Markdown file directly to `main` is enough:

```bash
git add src/content/blog/<article-slug>.md
git commit -S -m "Update <article title>"
git push
```

That push deploys production automatically. The preview-first pull-request
workflow remains the recommended path for new articles.

## Verification

GitHub build status:

```bash
gh run list --workflow "Publish container" --limit 5
```

Flux image automation:

```bash
flux get image repository mccruden-site
flux get image policy mccruden-site-preview
flux get image policy mccruden-site-production
flux get image update mccruden-site
```

Kubernetes rollouts:

```bash
kubectl --namespace mccruden-site rollout status \
  deployment/mccruden-site --timeout=5m

kubectl --namespace mccruden-site-production rollout status \
  deployment/mccruden-site --timeout=5m
```

Running image references:

```bash
kubectl --namespace mccruden-site get deployment mccruden-site \
  --output=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl --namespace mccruden-site-production get deployment mccruden-site \
  --output=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## Rollback

Revert the site commit that introduced the problem:

```bash
git switch main
git pull --ff-only
git revert <commit>
git push
```

The revert creates a new production image containing the previous site content,
and the normal automation deploys it. This keeps the rollback auditable and
avoids making an out-of-band Kubernetes change.

For a preview-only problem, revert or amend the non-`main` branch and push it
again.

## One-time platform prerequisite

Flux needs write access to the homelab repository so its image automation
controller can record selected digests. The Flux deploy key in GitHub must have
**Allow write access** enabled. This is a one-time platform setting, not part
of the routine publishing workflow.
