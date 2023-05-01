# Azure devops pipeline (in yml) to deploy a Nextjs project in azure App service (Linux machine).

Let's say we have a Nextjs project -

- is maintained with yarn package management
- has .env files in the root (usually .env files are not part of a git repo)

### Good to know
TODO!



```
# nextjs project pipeline

trigger:
  - main

# Using windows vm agent to build
pool:
  name: "default"
  vmImage: "winagent"

# define variables to use during the build
variables:
  projectFolder: "."
  buildOutputFolder: ".next"

# installing node.js
steps:
  - task: NodeTool@0
    inputs:
      versionSpec: "18.x"
    displayName: "Install Node.js"

  # Run the yarn install
  - script: |
      yarn install

  # Download secure file from azure library
  - task: DownloadSecureFile@1
    inputs:
      secureFile: ".env.production"

  # Copy the .env file
  - task: CopyFiles@2
    inputs:
      sourceFolder: "$(Agent.TempDirectory)"
      contents: "**/*.env.production"
      targetFolder: "$(projectFolder)"
      cleanTargetFolder: false

  # Run the yarn build
  - script: |
      yarn build

  # Download secure file from azure library
  - task: DownloadSecureFile@1
    inputs:
      secureFile: "ecosystem.config.js"

  # Copy the server.js file
  - task: CopyFiles@2
    inputs:
      sourceFolder: "$(Agent.TempDirectory)"
      contents: "**/*ecosystem.config.js"
      targetFolder: "$(projectFolder)"
      cleanTargetFolder: false

  # Archive all the files into a zip file for publishing
  - task: ArchiveFiles@2
    inputs:
      # rootFolderOrFile: "$(Build.ArtifactStagingDirectory)"
      rootFolderOrFile: "$(projectFolder)"
      archiveType: "zip"
      archiveFile: "$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip"
      includeRootFolder: false

  # Publish the zip file
  - task: PublishBuildArtifacts@1
    inputs:
      pathtoPublish: "$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip"

```
