library 'cb-days@master'
def dotnetpodYaml = libraryResource 'podtemplates/dotnet/dotnet-core.yaml'
def octopodYaml = libraryResource 'podtemplates/octopus-cli/octopus-cli-pod.yaml'
pipeline {
  // The following pipeline provides an opinionated template you can customize for your own needs.
  // 
  // Instructions for configuring the Octopus plugin can be found at
  // https://octopus.com/docs/packaging-applications/build-servers/jenkins#configure-the-octopus-deploy-plugin
  // 
  // Get a trial Octopus instance from https://octopus.com/start
  // 
  // This pipeline requires the following plugins:
  // * Pipeline Utility Steps Plugin: https://wiki.jenkins.io/display/JENKINS/Pipeline+Utility+Steps+Plugin
  // * Git: https://plugins.jenkins.io/git/
  // * Workflow Aggregator: https://plugins.jenkins.io/workflow-aggregator/
  // * Octopus Deploy: https://plugins.jenkins.io/octopusdeploy/.
  // * MSTest: https://plugins.jenkins.io/mstest/
  parameters {
    // Parameters are only available after the first run. See https://issues.jenkins.io/browse/JENKINS-41929 for more details.
    string(defaultValue: 'Default', description: '', name: 'SpaceId', trim: true)
    string(defaultValue: 'RandomQuotes-dotnet', description: '', name: 'ProjectName', trim: true)
    string(defaultValue: 'Development', description: '', name: 'EnvironmentName', trim: true)
    string(defaultValue: 'Octopus', description: '', name: 'ServerId', trim: true)
  }
  agent none
  stages {
    stage('Environment') {
      steps {
          echo "PATH = ${env.PATH}"
      }
    }
    stage('Checkout') {
      steps {
        // If this pipeline is saved as a Jenkinsfile in a git repo, the checkout stage can be deleted as
        // Jenkins will check out the code for you.
        script {
            /*
              This is from the Jenkins "Global Variable Reference" documentation:
              SCM-specific variables such as GIT_COMMIT are not automatically defined as environment variables; rather you can use the return value of the checkout step.
            */
            def checkoutVars = checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[url: 'https://github.com/OctopusSamples/RandomQuotes.git']]])
            env.GIT_URL = checkoutVars.GIT_URL
            env.GIT_COMMIT = checkoutVars.GIT_COMMIT
            env.GIT_BRANCH = checkoutVars.GIT_BRANCH
        }
      }
    }
    stage('Dependencies') {
      agent {
        kubernetes {
          label 'dotnet-core'
          yaml dotnetpodYaml
       }
      }
      steps {
       container('dotnet-core') {
        sh(script: 'dotnet restore')
        // Save the dependencies that went into this build into an artifact. This allows you to review any builds for vulnerabilities later on.
        sh(script: 'dotnet list package > dependencies.txt')
        archiveArtifacts(artifacts: 'dependencies.txt', fingerprint: true)
        // List any dependency updates.
        sh(script: 'dotnet list package --outdated > dependencieupdates.txt')
        archiveArtifacts(artifacts: 'dependencieupdates.txt', fingerprint: true)
      }
    }
    }
    stage('Build') {
      agent {
        kubernetes {
          label 'dotnet-core'
          yaml dotnetpodYaml
       }
      }
      steps {
       container('dotnet-core') {
        sh(script: 'dotnet build --configuration Release', returnStdout: true)
      }
    }
    }
    stage('Test') {
      agent {
        kubernetes {
          label 'dotnet-core'
          yaml dotnetpodYaml
       }
      }
      steps {
       container('dotnet-core') {
        sh(script: 'dotnet test -l:trx')
        mstest(testResultsFile: '**/*.trx', failOnError: false, keepLongStdio: true)
      }
    }
    }
    stage('Publish') {
      agent {
        kubernetes {
          label 'dotnet-core'
          yaml dotnetpodYaml
       }
      }
      steps {
        // Gitversion is available from https://github.com/GitTools/GitVersion/releases.
        // We attempt to run gitversion if the executable is available.
       container('dotnet-core') {
        sh(script: 'which gitversion && gitversion /output buildserver || true')
        // Capture the git version as an environment variable, or use a default version if gitversion wasn't available.
        // https://gitversion.net/docs/reference/build-servers/jenkins
        script {
            if (fileExists('gitversion.properties')) {
              def props = readProperties file: 'gitversion.properties'
              env.VERSION_SEMVER = props.GitVersion_SemVer
              env.VERSION_BRANCHNAME = props.GitVersion_BranchName
              env.VERSION_ASSEMBLYSEMVER = props.GitVersion_AssemblySemVer
              env.VERSION_MAJORMINORPATCH = props.GitVersion_MajorMinorPatch
              env.VERSION_SHA = props.GitVersion_Sha
            } else {
              env.VERSION_SEMVER = "1.0.0." + env.BUILD_NUMBER
            }
        }
        sh(script: 'dotnet publish --configuration Release /p:AssemblyVersion=${VERSION_SEMVER}')
        script {
            // Find published DLL files.
            def files = findFiles(glob: '**/publish/*.dll')
              .collect{it.path.substring(0, it.path.lastIndexOf("/"))}
              .unique(false)
            echo 'Found ' + files.size() + ' publish dirs'
            files.each{echo it}
            // Join the paths containing published application with colons.
            env.PUBLISH_PATHS = files.collect{it}.join(':')
            echo 'These paths are available from the PUBLISH_PATHS environment variable, separated by colons.'
        }
        script {
            env.PUBLISH_PATHS.split(":").each {
            	def packageId = "application"
            	dir("${env.WORKSPACE}/${it}/../../../..") {
            		 def projFiles = findFiles(glob: '*.csproj')
            		 if (projFiles.size() != 0) packageId = projFiles[0].path.substring(0, projFiles[0].path.lastIndexOf("."))		
            	}
            	dir("${env.WORKSPACE}/${it}") {
            		octopusPack(
            			additionalArgs: '', 
            			outputPath : "..",
            			includePaths: "**",
            			overwriteExisting: true, 
            			packageFormat: 'zip', 
            			packageId: packageId, 
            			packageVersion: env.VERSION_SEMVER, 
            			sourcePath: '', 
            			toolId: 'Default', 
            			verboseLogging: false)
            	}
            	dir("${env.WORKSPACE}/${it}/..") {
            		def artifact = "${pwd()}/${packageId}.${env.VERSION_SEMVER}.zip"
            		env.ARTIFACTS = artifact + ":" + env.ARTIFACTS
            		echo "Generated artifact at ${artifact}"
            	}
            }
            echo "Artifact paths have been saved in the ARTIFACTS environment variable"
        }
      }
    }
    }
    stage('Deployment') {
      agent {
        kubernetes {
          label 'octo-cli'
          yaml octopodYaml
       }
      }
      steps {
        // This stage assumes you perform the deployment with Octopus Deploy.
        // The steps shown below can be replaced with your own custom steps to deploy to other platforms if needed.
       container('octo-cli') {
        octopusPushPackage(additionalArgs: '',
          packagePaths: env.ARTIFACTS.split(":").join("\n"),
          overwriteMode: 'OverwriteExisting',
          serverId: params.ServerId,
          spaceId: params.SpaceId,
          toolId: 'Default')
        octopusPushBuildInformation(additionalArgs: '',
          commentParser: 'GitHub',
          overwriteMode: 'OverwriteExisting',
          packageId: env.ARTIFACTS.split(":")[0].substring(env.ARTIFACTS.split(":")[0].lastIndexOf("/") + 1, env.ARTIFACTS.split(":")[0].length()).replaceAll("\\." + env.VERSION_SEMVER + "\\..+", ""),
          packageVersion: env.VERSION_SEMVER,
          serverId: params.ServerId,
          spaceId: params.SpaceId,
          toolId: 'Default',
          verboseLogging: false,
          gitUrl: env.GIT_URL,
          gitCommit: env.GIT_COMMIT,
          gitBranch: env.GIT_BRANCH)
        octopusCreateRelease(additionalArgs: '',
          cancelOnTimeout: false,
          channel: '',
          defaultPackageVersion: '',
          deployThisRelease: false,
          deploymentTimeout: '',
          environment: params.EnvironmentName,
          jenkinsUrlLinkback: false,
          project: params.ProjectName,
          releaseNotes: false,
          releaseNotesFile: '',
          releaseVersion: env.VERSION_SEMVER,
          serverId: params.ServerId,
          spaceId: params.SpaceId,
          tenant: '',
          tenantTag: '',
          toolId: 'Default',
          verboseLogging: false,
          waitForDeployment: false)
        octopusDeployRelease(cancelOnTimeout: false,
          deploymentTimeout: '',
          environment: params.EnvironmentName,
          project: params.ProjectName,
          releaseVersion: env.VERSION_SEMVER,
          serverId: params.ServerId,
          spaceId: params.SpaceId,
          tenant: '',
          tenantTag: '',
          toolId: 'Default',
          variables: '',
          verboseLogging: false,
          waitForDeployment: true)
      }
    }
  }
 }
}
