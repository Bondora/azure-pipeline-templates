# Introduction 
Repository for commonly used Azure Pipeline templates.

# How to contribute

1. Create new feature branch
2. Use the branch reference (`ref: refs/heads/branch-name`) in pipeline to test out changes in this branch, see below for examples
3. Do the changes, push, create PR
4. Merge to main branch
5. Add *new version tag* (vN+1) if there are breaking changes to exiting template(s) or *move version tag* (set same tag) to latest commit.

## Breaking changes
Avoid any breaking changes (interface aka required parameters or changing template file name or location) as much as possible.
When adding parameters to templates, add default value, so that pipelines using the template do not fail.
Do not rename the template file or move to another directory. This will break the reference from pipelines using the template.

Adding new template file is not breaking change but new feature. As soon as any pipeline starts using the template you risk breaking the pipeline when changing the template interface (name and required parameters).

### When you need to do breaking changes
1. You have to find all the usages for the template and change those pipelines, when only few pipelines use it.
2. Add new version tag in format of vX (like v1 or v2) where X is integer. Then you can use the specific tag reference and reference new and old version (tag) in pipelines.

## After adding non-breaking changes
Move the tag to new commit (tag the commit with same version) after merging the changes to main branch so that pipelines referencing specific tag version get the changes.

## How to use templates in other repositories

```yaml
resources:
  repositories:
  - repository: templates
    type: git
    name: azure-devops-project-name/azure-pipeline-templates

stages:
- template: stages/set-version.yml@templates
```

## Specify pool for the stages/jobs in your pipeline, if you use main pipeline templates

### For Azure agent pool use `pool.vmImage`
```yaml
pool:
  vmImage: 'ubuntu-latest'
```

### For self-hosted aka private agent pool use `pool` or `pool.name`
```yaml
pool: 'my-agent-pool'
# or
pool:
  name: 'my-agent-pool'
```

## Note
You cannot use other files, like scripts, from the templates repository. Only the template files are combined with original pipeline and compiled into one yaml.

Use tag reference to anchor against specific tag, so that any breaking changes to templates do not break your pipeline.
If you need to change templates and test templates from other branch than main branch then use ´ref´ keyword.

```yaml
resources:
  repositories:
  - repository: templates
    type: git
    name: azure-devops-project-name/azure-pipeline-templates
    ref: refs/tags/v1 # Note: Referenc specific tag. For testing you can specify branch name, refs/heads/branch-name
```

# Template usage

## Templates for Building and publishing Nuget packages
.NET Nuget package test, analyze, build and publish to Azure Artifactory. 

### Template "pipelines/pipeline-dotnet-nuget.yml"
This template is used to build and publish _.NET / .NET Core / .NET Standad_ Nuget projects.

```yaml
trigger:
- master

# Set the default agent pool for all stages
pool:
  vmImage: 'ubuntu-latest'

resources:
  repositories:
  - repository: templates
    type: git
    name: azure-devops-project-name/azure-pipeline-templates
    ref: refs/tags/v1

extends:
  template: pipelines/pipeline-dotnet-nuget.yml@templates
  parameters:
    projectName: 'Bondora.Sample.Nuget' # Mandatory, used for project name (projectName.csproj)

```

## Use Stage templates inside your pipeline for custom pipeline

```yaml
parameters:
- name: serviceName
  type: string

variables:
- group: 'azure-container-registry'

pool:
  vmImage: 'ubuntu-latest'

resources:
  repositories:
  - repository: templates
    type: git
    name: azure-devops-project-name/azure-pipeline-templates
    ref: refs/tags/v1

stages:
- template: stages/set-version.yml@templates

- template: stages/build-test-analyze-dotnet.yml@templates
  parameters:
    sonarProjectKey: ${{ parameters.serviceName }}

- template: stages/build-and-push-docker-image.yml@templates
  parameters:
    dockerImage: ${{ parameters.serviceName }}

- template: stages/add-version-tag.yml
  parameters:
    dependsOn: Build_Publish

- template: stages/deploy-to-kubernetes-with-helm.yml@templates
  parameters:
    dependsOn: AddVersionTag
    environmentName: Staging
    serviceName: ${{ parameters.serviceName }}

- template: stages/deploy-to-kubernetes-with-helm.yml@templates
  parameters:
    dependsOn: AddVersionTag
    environmentName: Production
    serviceName: ${{ parameters.serviceName }}
```

## Template "jobs/notify-deploy-started-to-slack.yml"

- Parameter `serviceName` is required, so that service name can be displayed inside Slack message.
- Parameter `infoMessage` is optional informational message in Slack message.

```yaml
stages:
- stage: Deploy_Stage
  jobs:
  - job: Deploy_Job
    steps:
    - bash: 'echo success'
  - template: jobs/notify-deploy-started-to-slack.yml
    parameters:
      serviceName: 'MyServiceName'
```

## Template "jobs/notify-deploy-finished-to-slack.yml"

- Parameter `serviceName` is required, so that service name can be displayed inside Slack message.
- Parameter `deployJobIdentifier` is required. Value is "StageName.JobName", if you use deployments, then add ".Deploy" suffix to the end ("StageName.JobName.Deploy").
- Parameter `dependsOn` value should be set to deploy Job name.
- Parameter `infoMessage` is optional informational message in Slack message.

```yaml
stages:
- stage: Deploy_Stage
  jobs:
  - job: Deploy_Job
    steps:
    - bash: 'echo success'
  - template: jobs/notify-deploy-finished-to-slack.yml
    parameters:
      serviceName: 'MyServiceName'
      deployJobIdentifier: 'Deploy_Stage.Deploy_Job'
      dependsOn: Deploy_Job
```

## Template "jobs/notify-deploy-to-slack.yml"

### Note

Use `jobs/notify-deploy-started-to-slack.yml` and `jobs/notify-deploy-finished-to-slack.yml` templates for deploy notifications - before and after deploy job.

### Parameters

You can use this template inside the deploy stage before and after deployment job(s) to notify slack before and after deployment.

- Parameter `serviceName` is required, so that service name can be displayed inside Slack message.
- Parameter `deployState` allowed values are `started` and `ended`.
- When `deployState: started` parameter is specified, then service name, version and first 10 commit messages are shown is Slack message.
- When `deployState: ended` parameter is specified, then deployment result (succeeded, failed, skipped, canceled) and errors, when present, are shown.
- Parameter `deployJobIdentifier` is required only if `deployState` parameter value is `ended`. Value is "StageName.JobName", if you use deployments, then add ".Deploy" suffix to the end ("StageName.JobName.Deploy").
- Parameter `dependsOn` value should be set as deploy job name when `deployState: ended` is specified.
- Parameter `infoMessage` is optional informational message in Slack message.

### Send deploy started notification

```yaml
- template: ../jobs/notify-deploy-to-slack.yml
  parameters:
    serviceName: 'My service name'
    deployState: started
```

Example
```yaml
stages:
- stage: TestNotifySlackJobSuccess
  jobs:
  - template: jobs/notify-deploy-to-slack.yml
    parameters:
      serviceName: 'test-notify-slack-job-success-started'
  - job: TestNotifySlackJobSuccess_Job
    steps:
    - checkout: none
    - bash: 'echo success'
  - template: jobs/notify-deploy-to-slack.yml
    parameters:
      serviceName: 'test-notify-slack-job-success-ended'
      deployState: 'ended'
      deployJobIdentifier: 'TestNotifySlackJobSuccess.TestNotifySlackJobSuccess_Job'
      dependsOn: TestNotifySlackJobSuccess_Job
```

### Send deploy ended notification

```yaml
- template: ../jobs/notify-deploy-to-slack.yml
  parameters:
    serviceName: 'My service name'
    deployJobIdentifier: 'Deploy_Stage.Deploy_Job'
    dependsOn: Deploy_Job
    deployState: ended
```

Example
```yaml
stages:
- stage: TestNotifySlackJobFailed
  jobs:
  - template: jobs/notify-deploy-to-slack.yml
    parameters:
      serviceName: 'test-notify-slack-job-failed-started'
  - job: TestNotifySlackJobFailed_Job
    steps:
    - checkout: none
    - bash: 'exit 1'
  - template: jobs/notify-deploy-to-slack.yml
    parameters:
      serviceName: 'test-notify-slack-job-failed-ended'
      deployState: 'ended'
      deployJobIdentifier: 'TestNotifySlackJobFailed.TestNotifySlackJobFailed_Job'
      dependsOn: TestNotifySlackJobFailed_Job
```

# References
- [How to use template from other Repository](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#use-other-repositories)
- [Predefined (build) variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml)
- [Deployment jobs](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs?view=azure-devops)
