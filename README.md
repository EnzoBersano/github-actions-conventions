# Github-actions-conventions

A catalog of **reference GitHub Actions workflows** for the microservices of the `ingsis-bry` organization.

This repository is **not a technical dependency** (no *Reusable Workflows*, no `uses: org/repo/...`). It works as a **catalog to copy and adapt**: each microservice copies the files into its own repository and modifies them if it needs any particular behavior.

## Why copy instead of reference?

- **Autonomy:** each microservice owns its pipeline.
- **Less coupling:** a change in this repository does not automatically affect consumers.
- **Transparency:** anyone opening a microservice can see exactly what its CI does.
- **Easy adoption:** there are no extra dependencies to configure.

The trade-off is that if a bug is fixed here, the change must be propagated manually to the repositories that use these files.

This repository is complementary to `gradle-conventions`:

- `gradle-conventions`: standardizes how software is developed and built.
- `github-actions-conventions` (this repo): provides reference implementations to automate CI/CD.

## Contents

```text
.github/
└── workflows/
    ├── CI.yml
    └── publish-image.yml
```

## How to adopt them in a microservice

1. Copy `CI.yml` and `publish-image.yml` into the microservice's `.github/workflows/` directory.
2. Make the modifications described in each section below (image name, etc.).
3. Configure the secrets and permissions described below.
4. Commit and push: the workflows run automatically based on their triggers.

---

## 1. `CI.yml` — Continuous Integration

### What it does

Validates that the microservice's code builds and passes its checks before being merged into `main`.

| Aspect | Detail |
|---|---|
| **Name** | `CI` |
| **Triggers** | Every `pull_request` targeting `main`, and manually (`workflow_dispatch`) |
| **Permissions** | `contents: read` (read-only access to the code) |
| **Runner** | `ubuntu-latest` |

### Steps

1. **Checkout code** (`actions/checkout@v6`): downloads the code.
2. **Set up JDK 21** (`actions/setup-java@v5`): installs Java 21 (Amazon Corretto distribution).
3. **Setup Gradle** (`gradle/actions/setup-gradle@v6`): configures Gradle and caches dependencies between runs.
4. **Run check** (`./gradlew check`): runs Gradle's `check` task (tests, linters and any verification defined in the build, e.g. those provided by `gradle-conventions`).

### Requirements for the consuming repository

- A **Gradle project with the wrapper** (`gradlew` at the root, executable).
- Compatible with **JDK 21**.
- The main branch must be called `main` (otherwise, adjust `branches`).

### Required external configuration

- **Secrets:** none by default.
- **If the build resolves private dependencies** (for example `gradle-conventions` published to GitHub Packages), the job will need credentials. In that case, expose them to the `Run check` step as environment variables (for example from `secrets.GH_PACKAGES_USERNAME` / `secrets.GH_PACKAGES_TOKEN`, see section 2) and make sure the Gradle build reads them.
- 
### What can be modified

- `java-version` / `distribution` if the service uses a different Java version.
- `./gradlew check` for another task (e.g. `./gradlew build`) if needed.
- `branches` if the integration branch is not `main`.

It is recommended to **keep** `permissions: contents: read` (least privilege) and the use of `setup-gradle` (caching).

---

## 2. `publish-image.yml` — Docker Image Publishing

### What it does

Builds the microservice's Docker image and publishes it to **GitHub Container Registry (GHCR)** on every push to `main`.

| Aspect | Detail |
|---|---|
| **Name** | `Publish Docker Image` |
| **Triggers** | Every `push` to `main`, and manually (`workflow_dispatch`) |
| **Permissions** | `contents: read`, `packages: write` (required to push to GHCR) |
| **Runner** | `ubuntu-latest` |
| **Registry** | `ghcr.io` |

### Steps

1. **Checkout repository** (`actions/checkout@v4`).
2. **Set up Docker Buildx** (`docker/setup-buildx-action@v3`).
3. **Log in to GHCR** (`docker/login-action@v3`): authenticates with `github.actor` and the automatic `GITHUB_TOKEN` (you do not need to create it).
4. **Build and push image** (`docker/build-push-action@v6`): builds using the `Dockerfile` at the repository root (`context: .`) and pushes the image with two tags:
   - `:latest` → always points to the latest version of `main`.
   - `:<commit sha>` → immutable version, identical to the ID of the commit that produced it (traceability, GitOps and rollback; see below).

   It also passes two **build secrets** to Docker: `github_username` and `github_token`.

#### Why two tags on every publish (`latest` + commit SHA)?

Every image is pushed with both tags on purpose:

**Easy to identify the latest image:** 
- the same image is tagged both `latest` and `<sha>`, so it is the only image carrying the `latest` tag. Anyone looking at the package in GHCR can immediately tell which one is the most recent, and consumers that just want the newest build can pull `latest`.
**GitOps and traceability:** 
- the SHA tag is the **same ID as the commit** in the microservice's repository that produced the image. 
- Given a running container, you know exactly which commit it was built from. 
- Given a commit, you know which image to look for.

This introduces the GitOps idea of having the repository be the source of truth: 
- deployments can reference an immutable, commit-addressed image (`:<sha>`) instead of a moving tag, which also makes rollbacks trivial (just deploy the image of a previous commit).

### Requirements for the consuming repository

- A **`Dockerfile`** at the repository root.
- If the Dockerfile needs the credentials (e.g. so Gradle can download private packages during the build), it must consume them as BuildKit secrets:

  ```dockerfile
  RUN --mount=type=secret,id=github_username \
      --mount=type=secret,id=github_token \
      GITHUB_USERNAME=$(cat /run/secrets/github_username) \
      GITHUB_TOKEN=$(cat /run/secrets/github_token) \
      ./gradlew build
  ```

  (Variable names depend on how the build reads the credentials.)

### Required external configuration

#### a) Repository secrets

In the microservice's repository: **Settings → Secrets and variables → Actions → New repository secret**.

| Secret | Content |
|---|---|
| `GH_PACKAGES_USERNAME` | GitHub username that owns the token |
| `GH_PACKAGES_TOKEN` | Personal Access Token (classic) with the `read:packages` scope (plus whatever is needed to access the private packages the build consumes) |

> `GITHUB_TOKEN` does **not** need to be configured: GitHub provides it automatically.
> If the organization allows it, these secrets can be defined at the **organization** level to avoid repeating them in every microservice.
> Never commit tokens to the repository.

#### b) Actions permissions

**Settings → Actions → General → Workflow permissions**: workflows requesting `packages: write` must be allowed to publish (the `permissions` block in the file already requests it; if the organization restricts the token to read-only, enable it or adjust the policy).

#### c) Package visibility and access

The first time it is published, the package is created in GHCR under the organization. Check in **Packages → (package) → Package settings**:
- That the microservice's repository has write access to the package.
- The desired visibility (private/public) and who can *pull* it (e.g. the deployment environment).

### What must be modified when copying it

The file ships with values from a specific service. Before using it in another microservice, **replace**:

1. `env.IMAGE_NAME`: it contains the placeholder `ingsis-bry/<name of your microservice>`.
2. The `tags` of `build-push-action`: they are currently hardcoded as `ghcr.io/ingsis-bry/snippet-service:...`. Change `snippet-service` to the microservice's name (or, better, use `${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}` so it stays consistent with the variable defined above).

You may freely adjust the branch that triggers the publish, extra tags, and the `context`/`file` if the Dockerfile is not at the root. It is recommended to **keep** the minimal permissions and the double tag (`latest` + SHA).

---

## Quick checklist for a new microservice

- [ ] Copy `CI.yml` and `publish-image.yml` into `.github/workflows/`.
- [ ] Verify there is a `gradlew` and a `Dockerfile` at the root.
- [ ] Replace the service name in `publish-image.yml` (`IMAGE_NAME` and `tags`).
- [ ] Create the `GH_PACKAGES_USERNAME` and `GH_PACKAGES_TOKEN` secrets.
- [ ] Confirm Actions permissions and package access in GHCR.
- [ ] Open a test PR and verify `CI` passes; merge it and verify the image appears in GHCR.

## Future

If the number of microservices and workflows grows (e.g. 15 services, 5 shared workflows, frequent security changes), migrating to *Reusable Workflows* to centralize execution can be considered. For now, copying is the simplest option.
