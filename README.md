# Pipeliner - CCE Templates for CICD Pipelines

Pipeliner is a set of CICD pipeline templates for GitLab based on [Trunk-Based Development](https://trunkbaseddevelopment.com) git workflows. Pipelines are tailored to production deployments in Docker Swarm clusters maintained by CCE, but can also be easily modified for deployment in other types of infrastructure.

## Quick start

Add the following snippet to `.gitlab-ci.yml` in your repository:
```yaml
include:
  project: cce/pipeliner
  ref: 1.0.0
  file:
    - /main.yml
    - /pipelines/release-from-trunk.yml

variables:
  APP_NAME: myapp
```

Customize `APP_NAME` with your application name and adjust other variables if needed (see [Variables](#variables) below for details). If you prefer a [Release from Tag](#release-from-tag) pipeline use `/pipelines/release-from-tag.yml` instead of `/pipelines/release-from-trunk.yml`. Other examples can be found in [examples](./examples/).

Note that `ref` should be a tag. Using a branch can lead to including wrong templates from [main.yml](./main.yml), where `ref` is hard-coded to a tag. This constraint will be lifted once [GitLab's issue #219065](https://gitlab.com/gitlab-org/gitlab/-/issues/219065) is resolved.

## Pre-defined Pipelines

Pipeliner defines two ready-to-use pipelines without further user modifications needed:
- [Release from Trunk](#release-from-trunk)
- [Release from Tag](#release-from-tag)

Both pipelines follow the base principles of [Trunk-Based Development](https://trunkbaseddevelopment.com), i.e. using short-lived branches for any development, be it a new feature, a bug fix, or a simple code refactoring. Once the work is done and tested the code is merged into the trunk, at which point the code is considered to be production ready. Pipelines differ in timing for production release and deployment.

Pipelines have the following stages:
1. build
1. test
1. pre_release
1. release
1. deploy

### Release from Trunk

A new release is created and deployed to production on every push to the default branch (usually `master` or `main`). Ideally, this should happen via a merge request, but it is not enforced.

Developers needs to carefully maintain a `VERSION` file in the repository's root to define the release version (git tags) and docker image tags. Jobs in the `pre_release` will check for version conflicts.

See [pipelines/release-from-trunk.yml](./pipelines/release-from-trunk.yml) for the full pipeline definition.

### Release from Tag

A new release is created and deployed to production when a tag is pushed (or manually created via GitLab's UI).

This workflow allows for multiple independently developed features to be release at the same time. Although a `VERSION` file is not needed because a git tag will be used to create the release, it is considered a good practice to maintain a `VERSION` file in sync with the tags. Jobs in the `pre_release` will check for version conflicts.

See [pipelines/release-from-tag.yml](./pipelines/release-from-tag.yml) for the full pipeline definition.

## Template Jobs

Ready-to-use pipelines are constructed from template jobs. This approach allows developers to customize specific jobs, environments and stages while reusing other jobs, or even create their own custom pipelines from these templates.

Pipeliner provides two types of templates: jobs and environments.

### Jobs

Job templates are defined in [jobs](./jobs/) directory and implement foundational jobs for each stage.

Scripts shared by multiple jobs are compiled together in [jobs/scripts.yml](./jobs/scripts.yml) and used by jobs via GitLab's custom YAML [!reference](https://docs.gitlab.com/ee/ci/yaml/README.html#reference-tags) syntax.

#### Build

[This job](./jobs/build.yml) builds a docker image and pushes it to the project's registry using the commit's SHA as image tag, i.e. `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA`. If the latest images (`:$CI_COMMIT_BEFORE_SHA` shorten to 8 characters, and `:latest`) are present in the registry, they will be used as cache to speed up the build process.

If the `Dockerfile` uses [Label Schema](http://label-schema.org) to label the image, the build process will pass the following build arguments:
- `BUILD_DATE`: with value `date -u +'%Y-%m-%dT%H:%M:%SZ'`
- `VCS_REF`: with value `$CI_COMMIT_SHORT_SHA`
- `VCS_URL`: with value `$CI_PROJECT_URL`

To reduce the burden on the GitLab server, it is advised to configure the project to clean the registry by deleting images with commit SHAs regularly, while keeping images with version tags (https://docs.gitlab.com/ee/user/packages/container_registry/#cleanup-policy).

#### Test

[This job](./jobs/test.yml) is a placeholder for a test job that always passes. Developers should implement their own test job(s).

#### Pre Release

[Pre_release jobs](./jobs/pre_release.yml) perform checks on the release version to avoid conflicts. Currently, the job checks if a tag is already present in the remote repository with the same version. If so, the job will fail and downstream jobs will be blocked unless it was run on an open merge request. In this case the pipeline is allowed to proceed but developers are advised to fix the version before merging.

See [Release Jobs](#release) below to understand how release versions are found. Users can customize this as well and how to perform version checks.

#### Release

[Two jobs](./jobs/release.yml) are executed in parallel in this stage: (`create_release` and `tag_image`).

The first job one leverages GitLab's [release keyword](https://docs.gitlab.com/ee/ci/yaml/README.html#release) to create a new [Release in GitLab](https://docs.gitlab.com/ee/user/project/releases) with the specified version. This job also creates a tag in the remote repository if one doesn't exists already.

The second job will push a tag to the image with the specified version. Additionally, a second tag `:latest` is also pushed.

For ready-to-use pipelines release version are set as follows:
- [Release from Trunk](#release-from-trunk) pipelines use the value found in `VERSION` file.
- [Release from Tag](#release-from-tag) use `$CI_COMMIT_TAG`.

Users can customize this value by overwriting `tag_name` for release jobs and variable `APP_VERSION` for tag image jobs, for example:
```yaml
Create Release:
  extends: .create_release
  release:
    tag_name: 1.2.3

Tag Image:
  extends: .tag_image
  variables:
    APP_VERSION: 1.2.3
```

Users can also customize the `.set_app_version` script provided in [jobs/scripts.yml](./jobs/scripts.yml) to set `APP_VERSION`.

#### Deploy

A [deploy job](./jobs/release.yml) consists of the following steps:

1. Setup access to the swarm cluster. See [variables](#variables) below for a list of variables needed to connect to the swarm cluster.
1. Prepare application secrets. All environment variables with prefix `SECRET_` will be processed and stored in directory `~/.secrets` (in the docker container) were they will be available for the swarm stack. Those with suffix `_BASE64` will first be decoded.
1. Version docker secrets and configs. Produces version environment variables (`*_VERSION`) for each secret and config present in the swarm stack file. Version is based on the md5sum of the secret/config value such that the docker secret/config has a maximum length of 64 characters.
1. Login into the docker registry.
1. Set application release version.
1. Deploy stack from the swarm stack file.

A second job is defined for automatically stopping the deployment in non-production environments when the associated branch is deleted from the project's repository, or when the environments is manually stopped via GitLab's UI.

Note that [environment templates](#environments) further customize deploy jobs.

### Environments

Environment templates in [environments](./environments/) directory build upon jobs templates to provide specialized jobs for staging and production environments, and guarantee unique names for all resources in the swarm cluster. Users can define jobs for other custom environments if needed.

Environment templates only define jobs for deploy stages, but users can easily define custom specialized jobs for other stages. For example, if the build stage is different in staging and production environments.

Available (or active) and stopped environments for a project can be found in the project's `Operations | Environment` page.

You can learn more about the [application variables](#application-properties) that govern environment behavior below.

#### Production

[Production environments](./environments/production.yml) are long-lived deployments that ideally follow the default branch (`master` or `main`). Deployment is done to the same swarm stack by replacing it with new docker images, secrets and variables, etc...

Ready-to-use pipelines do not provide a "stop deploy" job for this environment.

Stack name for production environment defaults to `$APP_NAME`, and in the case of webapps, the URL defaults to `https://$APP_NAME.$APP_DOMAIN`.

#### Staging

[Staging environments](./environments/staging.yml) are meant to examine or review a feature branch. Multiple staging environments can co-exist concurrently, each one following a different branch. Updates to a branch will trigger a replacement of the associated staging environment with the new code (via docker images).

Stack names for staging environments contain the `$APP_NAME` and `$CI_COMMIT_REF_SLUG` to isolate them. For webapps, the URL is based on the stack name and cluster domain, i.e. `https://$APP_NAME-$CI_COMMIT_REF_SLUG.$SWARM_CLUSTER`

By default, staging environments automatically stop when the branch is merged or deleted, or after 3 days. This promotes short-lived feature branches and helps prevent cluttering the swarm cluster with too many environments. This setting can be customized by the user.

### Customizing jobs and pipelines

Jobs can be customized in multiple ways. For small changes, changing [variables](#variables) might be enough. Other times, [template jobs](./jobs/) or [template scripts](./jobs/scripts.yml) can be redefined in project's pipeline to override certain aspects or the full job.

Several customization examples can be found in [examples](./examples).

## Variables

Following variables need to be available to the pipeline via GitLab's UI or .gitlab-ci-yml.

### Swarm Cluster

Ask cluster admin for values.

- `SWARM_CLUSTER`: Hostname or IP for a Swarm Cluster manager.
- `CA_TLS_CERT`: CA TLS cert to verify client cert.
- `CLIENT_TLS_CERT`: Client TLS cert.
- `CLIENT_TLS_KEY_BASE64`: Client TLS key (base64 encoded).

### Application properties

- `APP_NAME`: App prefix to isolate deployments (default: "$CI_PROJECT_NAME").
- `APP_DOMAIN`: Domain for app URL in production (default: "rockefeller.edu").
- `STACK_FILE`: Relative path to docker-compose file (default: "stack.yml").

The following variables are computed by the pipeline, and in some cases can be overridden. See environments directory for default values.

- `APP_ENV`: Environment to run on.
- `APP_VERSION`: Version to build/tag images, create releases and deployments.
- `STACK_NAME`: Name of deployed stack, used in swarm stacks, secrets and configs to isolate deployments.
- `STACK_DOMAIN`: Domain URL for deployments.

### Secrets

- `SECRET_*`: App secrets to create in swarm stack, after removing the prefix SECRET_ (optional).
- `SECRET_*_BASE64`: Same as above but base64 encoded, after removing suffix _BASE64 too (optional).

## Author Information

Luis Gracia while at [The Rockefeller University](https://www.rockefeller.edu):
- lgracia [at] rockefeller.edu
- GitHub at [luisico](https://github.com/luisico)
