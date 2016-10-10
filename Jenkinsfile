// Gitlab connection properties
// Enter your gitlab server name
properties([[$class: 'GitLabConnectionProperty', gitLabConnection: '<gitlabserver.domain.local>']])

node('windows') {
  // Mark the code checkout 'stage'....
  stage('Checkout') {
    // Checkout code from repository
    checkout scm
  }

  gitlabCommitStatus('chocolatey') {
    stage('Build') {
      // create chocolatey package
      bat 'cpack'
    }

    stage('Test') {
      // chocolatey package install test
      bat 'powershell -command choco install (Get-Item *.nupkg)[0].name -dvyf'

      // chocolatey package uninstall test
      bat 'powershell -command choco uninstall -dvy ([xml](Get-Content (Get-Item *.nuspec)[0].name)).package.metadata.id'
    }
  }

  stage('Deploy') {
    // if this is the master branch, push package to repository
    if (env.BRANCH_NAME == 'master'){
      // replace with the Jenkins credential id that contains the chocolatey api key
      withCredentials([[$class: 'StringBinding', credentialsId: '<chocolatey server api key credential id>', variable: 'choco_api_key']]) {
        // push package to repo
        bat "powershell -command choco push (Get-Item *.nupkg)[0].name -s http://<chocolatey_server_url> -k ${choco_api_key} --force"
      }
    }
  }
}
