# Reusable Workflows

Reusable GitHub Actions workflows for Node.js and Java applications. This repository centralizes the common CI/CD stages used by the projects maintained by Pedro Nieto: validation, SonarCloud analysis, artifact packaging, and deployment to a VPS.

## What is included

| Workflow                                                                             | Purpose                                                                              |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| [`reusable-sonarqube-scan.yml`](.github/workflows/reusable-sonarqube-scan.yml)       | Runs a SonarCloud analysis for Node.js or Java projects and checks the quality gate. |
| [`reusable-publish-artifacts.yml`](.github/workflows/reusable-publish-artifacts.yml) | Builds a Node.js or Maven project and uploads a short-lived deployment artifact.     |
| [`reusable-deploy.yml`](.github/workflows/reusable-deploy.yml)                       | Transfers a published artifact to a VPS and restarts the application with PM2.       |
| [`workflow-sandbox-nodejs.yml`](.github/workflows/workflow-sandbox-nodejs.yml)       | Example Node.js pipeline that composes build, scan, publish, and deploy stages.      |
| [`workflow-sandbox-java.yml`](.github/workflows/workflow-sandbox-java.yml)           | Example Java pipeline that composes build, scan, publish, and deploy stages.         |

The sandbox workflows are examples intended to be adapted to an application repository. The `reusable-*` workflows are the public interface consumed with GitHub Actions `workflow_call`.

## Typical pipeline

```text
Build and test -> SonarCloud quality gate -> Publish artifact -> Deploy Application
```

SonarCloud and deployment are optional when a sandbox is started manually. On pushes and pull requests targeting `main`, the sandbox workflows run their configured pipeline automatically.

## Reusing a workflow

Reference a workflow from another repository using its full repository path and a ref. Pinning to a release tag or commit SHA is recommended for production consumers.

```yaml
jobs:
  publish:
    uses: pedroo-nietoo/reusable-workflows/.github/workflows/reusable-publish-artifacts.yml@main
    with:
      language: nodejs
      node_version: "22"
      artifact_name: application-package
```

For a complete composition, use the sandbox workflows as a starting point. Jobs that call reusable workflows must pass secrets explicitly or use `secrets: inherit` when the caller and reusable workflow are in the same organization.

## Workflow contracts

### SonarCloud scan

File: [`reusable-sonarqube-scan.yml`](.github/workflows/reusable-sonarqube-scan.yml)

Inputs:

| Input          | Required | Default | Description                             |
| -------------- | -------- | ------- | --------------------------------------- |
| `language`     | Yes      | -       | `nodejs` or `java`.                     |
| `node_version` | No       | `22`    | Node.js version used for Node.js scans. |
| `java_version` | No       | `17`    | Java version used for Java scans.       |

Secrets:

| Secret        | Required | Description                                 |
| ------------- | -------- | ------------------------------------------- |
| `SONAR_TOKEN` | Yes      | Token used to authenticate with SonarCloud. |

Node.js scans expect `npm ci`; Java scans expect a Maven project and run `mvn clean verify sonar:sonar`. The current configuration uses the SonarCloud organization `pedroo-nietoo`.

### Publish artifacts

File: [`reusable-publish-artifacts.yml`](.github/workflows/reusable-publish-artifacts.yml)

Inputs:

| Input           | Required | Default | Description                                       |
| --------------- | -------- | ------- | ------------------------------------------------- |
| `language`      | Yes      | -       | `nodejs` or `java`.                               |
| `node_version`  | No       | `22`    | Node.js version used to build the application.    |
| `java_version`  | No       | `17`    | Java version used to build the Maven application. |
| `artifact_name` | Yes      | -       | Name assigned to the uploaded artifact.           |

Node.js projects must provide `package-lock.json`, build into `dist/`, and expose an `npm run build` script. The workflow prunes development dependencies and packages `dist/`, `node_modules/`, `package.json`, and `package-lock.json` into `deployment-package.tar.gz`.

Java projects must provide a Maven build and produce a JAR under `target/`. The workflow runs `mvn clean package -DskipTests` and uploads the matching JAR. Artifacts are retained for one day.

### Deploy Application

File: [`reusable-deploy.yml`](.github/workflows/reusable-deploy.yml)

Inputs:

| Input                  | Required | Default | Description                                        |
| ---------------------- | -------- | ------- | -------------------------------------------------- |
| `language`             | Yes      | -       | `nodejs` or `java`.                                |
| `artifact_name`        | Yes      | -       | Artifact downloaded from the current workflow run. |
| `source_path`          | Yes      | -       | File or glob transferred to the VPS.               |
| `scp_strip_components` | No       | `0`     | Path components removed by the SCP action.         |
| `app_dir`              | Yes      | -       | Application directory on the VPS.                  |
| `app_name`             | Yes      | -       | PM2 process name.                                  |

Secrets:

| Secret        | Required | Description                                      |
| ------------- | -------- | ------------------------------------------------ |
| `VPS_HOST`    | Yes      | VPS hostname or IP address.                      |
| `VPS_USER`    | Yes      | SSH user.                                        |
| `VPS_SSH_KEY` | Yes      | Private SSH key used by the SCP and SSH actions. |
| `VPS_PORT`    | No       | SSH port; defaults to `22`.                      |

The target host must have `rsync`, `tar`, Java or Node.js as appropriate, and PM2 available to the deployment user. Node.js deployments assume the application starts at `dist/main.js` and use the Node.js 22 installation managed by NVM. Java deployments install the JAR as `app.jar` and start it through PM2 with `java -jar`.

## Sandbox inputs

Both sandbox workflows expose the following `workflow_dispatch` options:

| Input                  | Default | Effect                                                   |
| ---------------------- | ------- | -------------------------------------------------------- |
| `enable_sonar_scan`    | `false` | Enables the SonarCloud job for a manually triggered run. |
| `enable_deploy_option` | `false` | Enables the deployment job for a manually triggered run. |

The Node.js sandbox uses Node.js 22. The Java sandbox uses Java 17 and Maven. Both target the `main` branch for push and pull request events.

## Required repository configuration

Before enabling the workflows in an application repository:

1. Add `SONAR_TOKEN` as a repository or organization secret.
2. Add `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, and optionally `VPS_PORT` for deployment.
3. Ensure the caller has permission to use the reusable workflows.
4. Confirm the application follows the expected Node.js or Maven build layout.
5. Confirm the VPS user has access to the application directory and can manage its PM2 process.

Do not commit private keys, tokens, or host credentials to workflow files. Prefer environment-scoped secrets and restrict deployment permissions to trusted branches and environments.

## Versioning and changes

Changes to reusable workflows can affect every consumer. Review workflow changes as API changes, document breaking changes, and prefer immutable release tags or commit SHAs for production usage. Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/).

## License

This project is released under the [MIT License](LICENSE).

## Maintainer

Pedro Nieto — [@pedroo-nietoo](https://github.com/pedroo-nietoo)