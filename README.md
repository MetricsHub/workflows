# GitHub Actions Workflows

![GitHub release (with filter)](https://img.shields.io/github/v/release/metricshub/workflows)
![GitHub top language](https://img.shields.io/badge/language-YAML-blue)
![License](https://img.shields.io/github/license/metricshub/workflows)

This repository provides common workflows to be [used as GitHub Actions](https://docs.github.com/en/actions/using-workflows/reusing-workflows).

- [GitHub Actions Workflows](#github-actions-workflows)
  - [Maven Build](#maven-build)
    - [Inputs](#inputs)
    - [How to use it?](#how-to-use-it)
    - [Troubleshooting the build](#troubleshooting-the-build)
      - [Debug mode](#debug-mode)
      - [SSH in the Runner](#ssh-in-the-runner)
  - [Maven Central Deploy](#maven-central-deploy)
    - [Requirements](#requirements)
    - [Inputs](#inputs-1)
    - [How to use it?](#how-to-use-it-1)
  - [Maven Central Release](#maven-central-release)
    - [Requirements](#requirements-1)
    - [Inputs](#inputs-2)
    - [How to Use It?](#how-to-use-it-2)
    - [Troubleshooting](#troubleshooting)
      - [Failure during *Prepare Release*](#failure-during-prepare-release)
      - [Failure during *Perform Release*](#failure-during-perform-release)
      - [Failures with GitHub Pages](#failures-with-github-pages)
      - [Retrying a release](#retrying-a-release)

## Maven Build

The `maven-build.yml` workflow simply builds a Maven project with the `mvn verify site` phases. It's designed to be invoked on `push` on PRs, so that this action is considered as a *Check* on the PR, i.e. it will prevent the build from being merged if the build fails. Produced reports such as checkstyle, PMD and spotbugs will then be added as comments on pull-request.

### Inputs

| Input | Description | Default value |
|---|---|---|
| `jdkVersion` | Version of the JDK to setup to run Maven. See [supported syntax](https://github.com/actions/setup-java#supported-version-syntax). | `17` |
| `nodeVersion` | Version of the NodeJS to setup for this project. **Optional** (if not specified, NodeJS won't be installed). See [supported syntax](https://github.com/actions/setup-node#supported-version-syntax). | *None* |
| `debug` | Whether to run Maven in debug mode (with `-X` option) | `"false" |
| `ssh` | Whether to open a SSH session in the GitHub Actions runner, to let you interact with the system that performed the build. **SSH session automatically closes after 10 minutes of inactivity.** | `"false"` |
| `uploadCheckstyleReport` | Whether to upload any Checkstyle report created by Maven | `"true"` |
| `uploadPMDReport` | Whether to upload any PMD report created by Maven | `"true"` |
| `uploadSpotbugsReport` | Whether to upload any Spotbugs report created by Maven | `"true"` |

### How to use it?

To use this workflow, you must define a `job` that `uses: metricshub/workflows/.github/workflows/maven-build.yml@main`.

> [!IMPORTANT]
> Instead of `@main`, it is recommended to use an actual version, to avoid breaking your build if the workflow required inputs change in the future, like `@v1`.

To run this workflow as an automatic check in the PRs or a repository, and also allow users to trigger the workflow manually (with debug options), create a `.github/workflows/build.yml` with the below content:

```yaml
name: Maven Build

on:
  # Run on all PRs to "main" (update to include the protected branches of your repository)
  pull_request:
    branches: [ "main" ]

  # Run the workflow manually
  workflow_dispatch:
    inputs:
      debug:
        type: boolean
        description: Maven debug mode
        required: false
        default: false
      ssh:
        type: boolean
        description: Open SSH session in the runner
        required: false
        default: false
      uploadCheckstyleReport:
        type: boolean
        description: Upload Checkstyle report created by Maven
        required: false
        default: true
      uploadPMDReport:
        type: boolean
        description: Upload PMD report created by Maven
        required: false
        default: true
      uploadSpotbugsReport:
        type: boolean
        description: Upload Spotbugs report created by Maven
        required: false
        default: true

jobs:
  build:
    # Call this shared workflow (use `@v1` to use a specific version, instead of the latest)
    uses: sentrysoftware/workflows/.github/workflows/maven-build.yml@main
    with:
      jdkVersion: "17"
      nodeVersion: "20.x"
      debug: ${{ github.event_name == 'workflow_dispatch' && inputs.debug }}
      ssh: ${{ github.event_name == 'workflow_dispatch' && inputs.ssh }}
      uploadCheckstyleReport: ${{ inputs.uploadCheckstyleReport }}
      uploadPMDReport: ${{ inputs.uploadPMDReport }}
      uploadSpotbugsReport: ${{ inputs.uploadSpotbugsReport }}
```

### Troubleshooting the build

In case of a build failure, the standard output of Maven may not be sufficient to troubleshoot the build.

#### Debug mode

To enable Maven's debug mode (`-X` or `--debug` option), simply re-run the workflow manually on the branch you need (typically the branch of your PR), with the `Maven debug mode` option set to `true` (check the corresponding box).

#### SSH in the Runner

Sometimes even the debug output of Maven is not enough to understand what is wrong with your build. You may want to check some files yourself, or execute some commands in the build *Runner* itself. This is possible with the `Open SSH session in the runner` option.

When enabled, this option starts an SSH daemon in the *Runner* at the beginning of the build. You can open a session either with your favorite SSH client, or using the built-in Web client. Instructions to connect to the *Runner* are provided in the output of the build, with repeated messages.

The SSH daemon prevents the build from completing, and wait for the user to open a session for 10 minutes. After a period of 10 minutes of inactivity (no session open, no interaction), the SSH daemon is automatically closed and the build completes.

## Maven Central Deploy

The `maven-central-deploy.yml` workflow builds a Maven project and *deploys* its artifacts to [Maven Central](https://central.sonatype.org/publish/publish-portal-maven/). It is designed to be invoked on `push` on the `main` branch of the repository, so the **SNAPSHOT** version built from the `main` branch can be used by other projects as a dependency. Basically, the project is built with the `mvn deploy` command.

> [!IMPORTANT]
> This workflow is **NOT** designed to deploy release versions of the artifact to Maven Central repositories, as it doesn't fulfill the requirements for Maven Central (signatures, etc.).

The workflow also updates the dependency tree for [dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates).

### Requirements

The below secrets must be declared in the repository or the GitHub organization:

| Secret | Description |
|---|---|
| `MAVEN_CENTRAL_USERNAME` | Username to connect to [Maven Central Repository](https://central.sonatype.com) |
| `MAVEN_CENTRAL_TOKEN` | The corresponding token obtained from Sonatype |

### Inputs

| Input | Description | Default value |
|---|---|---|
| `jdkVersion` | Version of the JDK to setup to run Maven. See [supported syntax](https://github.com/actions/setup-java#supported-version-syntax). | `17` |
| `nodeVersion` | Version of the NodeJS to setup for this project. **Optional** (if not specified, NodeJS won't be installed). See [supported syntax](https://github.com/actions/setup-node#supported-version-syntax). | *None* |

### How to use it?

To use this workflow, you must define a `job` that `uses: metricshub/workflows/.github/workflows/maven-central-deploy.yml@main`.

> [!IMPORTANT]
> Instead of `@main`, it is recommended to use an actual version, to avoid breaking your build if the workflow required inputs change in the future, like `@v1`.

To run this workflow automatically on the `main` branch, create a `.github/workflows/deploy.yml` with the below content:

```yaml
name: Maven Deploy

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    uses: metricshub/workflows/.github/workflows/maven-central-deploy.yml@main
    with:
      jdkVersion: "17"
      nodeVersion: "20.x"
    secrets: inherit
```

## Maven Central Release

The `maven-central-release.yml` workflow performs all the necessary actions to release a Maven project to [Maven Central](https://central.sonatype.com), which is available in Maven Central.

This workflow **must** be triggered manually and run from the `main` branch of the project, and will perform the below actions:

* Create and checkout a `release/vX.Y.Z` branch
* [Prepare the release with `maven-release-plugin:prepare`](https://maven.apache.org/maven-release/maven-release-plugin/usage/prepare-release.html), which itself will:
    * Check there are no *SNAPSHOT* dependencies
    * Update `pom.xml` with the release version (`releaseVersion` input, non-*SNAPSHOT*)
    * Build the project and site, and run all tests with `mvn clean verify site`
    * Commit and tag as `vX.Y.Z`
    * Update `pom.xml` again with the new development version (`developmentVersion` input, *SNAPSHOT*)
    * Build the project again
    * Commit the changes
* [Perform the actual release with `maven-release-plugin:perform`](https://maven.apache.org/maven-release/maven-release-plugin/usage/perform-release.html)
    * Checkout the `vX.Y.Z` in a temporary folder (`target/release`)
    * Build the project with `mvn deploy`
    * Note: The `deploy` phase is enhanced with [Nexus Staging Maven Plugin](https://github.com/sonatype/nexus-maven-plugins/tree/main/staging/maven-plugin)
    * Sign the artifacts with GPG
    * Deploy artifacts to a new staging repository on [Sonatype's Nexus Repository Manager](https://s01.oss.sonatype.org)
    * Release the repository immediately if `autoRelease` input is `true`
* Update the site (`target/site/*`) to GitHub Pages
* Create a GitHub Release with the produced artifacts
* Create a Pull Request from the `release/vX.Y.Z` branch to `main`
* Deploy the GitHub Pages

### Requirements

The below secrets must be declared in the GitHub repository or the GitHub organization:

| Secret | Description |
|---|---|
| `MAVEN_CENTRAL_USERNAME` | Username to connect to [Sonatype's Nexus Repository Manager](https://s01.oss.sonatype.org) |
| `MAVEN_CENTRAL_TOKEN` | The corresponding token obtained from Sonatype |
| `MAVEN_GPG_PRIVATE_KEY` | The GPG private key to [sign the artifacts](https://central.sonatype.org/publish/requirements/gpg/) |
| `MAVEN_GPG_PASSPHRASE` | The associated passphrase |

Additionally, the `pom.xml` must declare a `release` [profile](https://maven.apache.org/guides/introduction/introduction-to-profiles.html), to declare a few custom phases for the release, as below:

```xml
  <profiles>

  <!-- Profile for releasing the project -->
  <profile>
   <id>release</id>
   <build>
    <plugins>

     <!-- gpg to sign the released artifacts -->
     <plugin>
      <artifactId>maven-gpg-plugin</artifactId>
      <version>3.1.0</version>
      <executions>
       <execution>
        <id>sign-artifacts</id>
        <phase>verify</phase>
        <goals>
         <goal>sign</goal>
        </goals>
        <configuration>
         <updateReleaseInfo>true</updateReleaseInfo>
         <gpgArguments>
          <arg>--pinentry-mode</arg>
          <arg>loopback</arg>
         </gpgArguments>
        </configuration>
       </execution>
      </executions>
     </plugin>

     <!-- release -->
     <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-release-plugin</artifactId>
      <version>3.0.1</version>
      <configuration>
       <tagNameFormat>v@{project.version}</tagNameFormat>
      </configuration>
      <executions>
       <execution>
        <id>default</id>
        <goals>
         <goal>perform</goal>
        </goals>
       </execution>
      </executions>
     </plugin>
    </plugins>
   </build>
  </profile>
```

### Inputs

| Input | Description | Default value |
|---|---|---|
| `releaseVersion` | Version to release (will be set in `pom.xml` for this release) | *None* |
| `developmentVersion` |  Version of the new *SNAPSHOT* version (will be set in `pom.xml` **after** the release) | *None* |
| `autoRelease` | Whether to release the staging repository on Sonatype Nexus immediately | `false` |
| `jdkVersion` | Version of the JDK to setup to run Maven. See [supported syntax](https://github.com/actions/setup-java#supported-version-syntax). | `17` |
| `nodeVersion` | Version of the NodeJS to setup for this project. **Optional** (if not specified, NodeJS won't be installed). See [supported syntax](https://github.com/actions/setup-node#supported-version-syntax). | *None* |

### How to Use It?

To use this workflow, you must define a `job` that `uses: metricshub/workflows/.github/workflows/maven-central-release.yml@main`.

> [!IMPORTANT]
> Instead of `@main`, it is recommended to use an actual version, to avoid breaking your build if the workflow required inputs change in the future, like `@v1`.

To be able to trigger this workflow manually, create a `.github/workflows/release.yml` with the below content:

```yaml
name: Release to Maven Central
run-name: Release v${{ inputs.releaseVersion }} to Maven Central

on:
  workflow_dispatch:
    inputs:
      releaseVersion:
        description: "Release version"
        required: true
        default: ""
      developmentVersion:
        description: "New SNAPSHOT version"
        required: true
        default: ""

jobs:
  release:
    uses: metricshub/workflows/.github/workflows/maven-central-release.yml@main
    with:
      releaseVersion: ${{ inputs.releaseVersion }}
      developmentVersion: ${{ inputs.developmentVersion }}
      autoRelease: true
      jdkVersion: "17"
      nodeVersion: "20.x"
    secrets: inherit
```

To trigger the release workflow, select it in the **Actions** tab, and use the `Run workflow` button. You'll be required to manually enter:

* **Release version**: the version that will be released (`pom.xml` will be updated accordingly before the release)
* **New SNAPSHOT version**: the new development version, ending with `-SNAPSHOT` (`pom.xml` will be updated after the release in a dedicated Pull Request)

The workflow performs several builds (so it may take some time to complete).

If the workflow completes successfully, you will need to perform 2 additional tasks:

1. Login to [Maven Central](https://central.sonatype.com), and **Release** the *staging repository* that has been created by the workflow (the exact name of the repository is listed in the workflow summary). This operation has already been performed if the workflow was called with `autoRelease: true`.
2. Approve and merge the Pull Request that has been created by the workflow (for the `release/vX.Y.Z` branch)

### Troubleshooting

#### Failure during *Prepare Release*

If the workflow fails during the *Prepare Release* step, check the output of `maven-release-plugin`. While preparing the release, a few validation checks are performed (e.g. the presence of *SNAPSHOT* dependencies) and will fail the build if necessary.

#### Failure during *Perform Release*

If the workflow fails while trying to *close* the staging repository on Sonatype's OSSRH Maven repository, it probably means the artifacts don't comply with the [validation rules for Maven Central](https://central.sonatype.org/publish/requirements/).

#### Failures with GitHub Pages

The GitHub repository must be [configured to publish *GitHub Pages* through *GitHub Actions*](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow).

#### Retrying a release

In case of a release failure that doesn't require updating the code, you can simply restart the workflow. The `release/vX.Y.Z` branch and the `vX.Y.Z` tag will be reset properly by the workflow, so you don't have to worry about these.

If the code needs to be updated, you will need to push changes to the `main` branch (from which the release workflow is executed). **Don't use `bugfix/` branches on the `release/vX.Y.Z` branch**. Then re-execute the release workflow from the updated `main` branch.
