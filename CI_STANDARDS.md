# LISA CI Standards

## 1. Purpose

Define consistent, reusable Continuous Integration (CI) standards for Project LISA and its related application repositories.

## 2. Pull Request Quality Gates

Every applicable pull request must pass the following checks before it can be merged:

1. **Code quality:** Run the configured linter, formatter checks, and type checks where supported.
2. **Automated tests:** Run relevant unit tests and integration tests.
3. **Build validation:** Verify that application dependencies install correctly and the application builds successfully.
4. **Security:** Run applicable dependency vulnerability checks and secret detection.
5. **Container validation:** Build the Docker image and validate its configuration when the repository produces a container image.

A check must not be silently skipped merely because a tool or test command has not yet been configured. Missing checks must be documented and addressed as part of onboarding the repository.

## 3. Reusable Workflow Design

- Shared CI logic should be maintained in the `lisa-cicd` repository.
- Application repositories should invoke approved reusable workflows rather than independently duplicating common pipeline logic.
- Language-specific checks must reflect the actual technology stack and supported project commands.
- Reusable workflows should expose only the inputs required for legitimate project differences.
- Shared workflows must be versioned and referenced through an explicitly pinned release tag or commit SHA.

## 4. Security Requirements

- Never commit credentials, tokens, private keys, or production secrets.
- Grant GitHub Actions only the permissions required by each job.
- Do not expose deployment credentials to untrusted pull-request workflows.
- Do not embed secrets in Docker images or image build arguments.
- Pin third-party GitHub Actions to full commit SHAs where practical.
- Treat dependency and container vulnerability findings according to an explicitly defined severity policy.

## 5. Build and Artifact Integrity

- Use project lockfiles and pinned tool versions where available.
- Build artifacts from a known source revision.
- Use immutable Git commit SHA tags for container images.
- Record the source commit and image digest for published artifacts.
- Never rely on the mutable `latest` tag as the sole release identifier.

## 6. Separation of CI and CD

- CI validates changes and builds artifacts.
- Continuous Deployment (CD) deploys approved artifacts to an environment.
- Pull-request validation must not automatically deploy application changes.
- Deployment credentials and environment-specific configuration must remain separate from ordinary CI checks.

## 7. Adoption Strategy

1. Inspect each repository's actual structure, dependencies, tests, and startup commands.
2. Implement and validate the shared workflow against one representative application.
3. Adapt the workflow for other supported technology stacks.
4. Add cross-repository integration checks where dependencies require them.
5. Introduce image publishing and deployment only after the relevant build and runtime checks are reliable.

## 8. Exceptions

Any temporary exception must identify the missing check, its reason, its owner, and the plan to resolve it. Exceptions must not silently weaken security requirements.