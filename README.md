# Azure devops pipeline (in yml) to deploy a Nextjs project in azure App service (Linux machine).

Let's say we have a Nextjs project -

- is maintained with yarn package management
- has .env files in the root (usually .env files are not part of a git repo)

### Prepare files for the Azure Library

- Prepare the .env file (OR .env.production).
- Prepare the ecosystem.config.js file.
```
module.exports = {
  apps: [
    {
      name: "your-app-name",
      script: "./node_modules/next/dist/bin/next",
      args: "start -p " + (process.env.PORT || 3000),
      watch: false,
      autorestart: true,
    },
  ],
};
```
- Add all those files (.env OR .env.production and ecosystem.config.js) to azure devops library as secure files. You can download these secure files in the build machine using a DownloadSecureFile@1 pipeline task (yml). This way we are making sure the correct .env file is provided in the build machine.


<img src="/pipeline_library.png" />

### Azure Build pipeline
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

  # Rename the .env.production file to .env before the build
  - task: PowerShell@2
    inputs:
      targetType: 'inline'
      script: |
        Rename-Item -Path $(projectFolder)\.env.production -NewName $(projectFolder)\.env

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
      archiveFile: "$(Build.ArtifactStagingDirectory)/next.zip"
      includeRootFolder: false

  # Publish the zip file
  - task: PublishBuildArtifacts@1
    inputs:
      pathtoPublish: "$(Build.ArtifactStagingDirectory)/next.zip"

```
### Azure Release pipeline
- In the release pipeline task ```Deploy Azure App Service (Linux machine)```, make sure to have the following command in the Startup command
``` pm2 start /home/site/wwwroot/ecosystem.config.js --no-daemon ```
