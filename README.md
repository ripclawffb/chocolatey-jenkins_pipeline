# chocolatey_jenkins_pipeline
An example of setting up a chocolatey package pipeline in Jenkins 2.0 and integrating with Gitlab allowing continuous integration and continuous delivery.

# Continuous Integration / Delivery with Jenkins, Gitlab and Chocolatey

## Background:
I maintain an internal chocolatey repository for our windows infrastructure with over 50 chocolatey packages and it is constantly growing. When creating the chocolatey packages, I need to ensure the package installation and uninstallation scripts work properly. Once all tests have passed, then I publish the package to our internal repository.

## Challenges:
These were some challenges I faced with the growing number of packages

  * Testing of package installation and uninstallation on a machine in a clean state
  * Version control of chocolatey packages
  * Status of chocolatey package builds in a central location
  * Manually publishing packages to the internal repository
  * Remembering the order of operations when creating/updating the chocolatey packages
  * Amount of time required to publish a fully working and tested chocolatey package

## Solution:
Here's the solution that I implemented that help me solve those challenges

*Note: You will need a Jenkins Master that's running windows with Git and Chocolatey installed. If you have Jenkins installed on a linux machine, then you will need a Windows Jenkins Agent that has Git and Chocolatey installed on it. You will also need a chocolatey server if you want builds to be pushed to the repository by Jenkins.*

The first step was to integrate Jenkins with Gitlab so the build status would be reported next to the Gitlab project based on the Jenkins testing of the package.

###Create a chocolatey project in Gitlab:
1. Go to your dashboard on Gitlab
2. Click on "New project" in the upper right hand corner
3. Fill out the required information
4. Click "Create project"

###Integrate Jenkins with Gitlab
In order for Jenkins to checkout projects from Gitlab and report back commit status, we need to add some credentials and API tokens.

1. Install Jenkins
2. Install the recommended plugins
3. Install the Jenkins Gitlab Plugin
4. Create an [SSH key pair](https://docs.gitlab.com/ce/ssh/README.html) for a new user to be used as a deploy key by Jenkins so the agents can checkout branches from Gitlab via ssh.
   *Note: Leave the passphrase blank, otherwise, Jenkins will not be able to checkout branches
5. Create a deploy key for the Gitlab project
    1. Navigate to the Gitlab project dashboard
    2. In the uppper right hand corner, click on the drop down menu (with gear icon)
    3. Click on "Deploy Keys"
    4. Enter a "Title" for the deploy key
    5. Paste in the contents of the private key (id_rsa) in the "Key" field
    6. Click "Add key"
    7. Verify the key is enabled for the project
6. Add the contents of the public key (id_rsa.pub) as a credential in Jenkins
    1. Navigate to your Jenkins instance web interface
    2. Log in if required
    3. Click on "Credentials" on the left hand side
    4. Click on "System", then "Global Credentials"
    5. Create a new [credential](https://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin) by clicking on "Add Credentials"
      1. Under "Kind", select "SSH Username with private key"
      2. Select a "Scope"
      3. Enter the "Username" that you used to create the ssh key pair
      4. Select "Enter directly" under "Private Key" and paste the contents of the publick key (id_rsa.pub) in the "Key" field
      5. Leave the "ID and Passphrase" fields blank
      6. Enter a "Description" for the credential (optional)
      7. Click "OK"
7. Now, we need to generate an API key so Jenkins can talk to Gitlab
    1. In Gitlab, create a [new user](https://docs.gitlab.com/ce/workflow/add-user/add-user.html) for Jenkins with appropriate permissions on the chocolatey project
    2. Log into Gitlab as that Jenkins user, click on the hamburger icon in the upper left and click on Profile Settings
    3. Click on the Access Tokens tab
    4. Enter a "Name" for the token and an expiration date (optional)
    5. Click "Create Personal Access Token"
    6. Copy the new personal access token
8. Next, we will add the new API token to Jenkins
    1. Go back to the Jenkins web interface, and log in if necessary
    2. Click on "Credentials"
    3. Click on "System", then "Global Credentials"
    4. Create a new [credential](https://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin) by clicking on "Add Credentials"
      1. Under "Kind", select "Gitlab API token"
      2. Select a "Scope"
      3. Paste the API token copied previously in the "API Token" field
      4. Leave the "ID" field blank
      5. Enter a "Description" for the credential (optional)
      6. Click "OK"
9. We are ready to configure the Jenkins Gitlab plugin
    1. In the Jenkins web interface, click on Manage Jenkins on the left hand side
    2. Click on "Configure System"
    3. Under the Gitlab section, enter the "Connection name" and "Gitlab host URL" (base url of your internal Gitlab)
    4. Under "Credentials", select the GitLab API token we added previously
    5. Click "Advanced" and select any applicable options
    6. Click "Test Connection" and verify you get a "Success" message
10. Congratulations, Gitlab and Jenkins should now be integrated!

###Create a Jenkins Multibranch pipeline.
This job will be configured so it scans the Gitlab repository for a Jenkinsfile and automatically create a job for any branches that contain this file. This is known as pipeline as a code.

1. Open a browser to your Jenkins web interface and login if necessary
2. Click on New Item in the upper left
3. Enter name of the job you would like to create (ex: the name of the chocolatey repo)
4. Select Multibranch Pipeline
5. Click OK
6. Configure the new Multibranch Pipline Job
    1. Under "Branch Sources", click "Add source" and choose Git
    2. Under "Project Repository", enter the SSH gitlab link (this can be found on the project dashboard under Gitlab)
      1. Example: git@repo.domain.local:chocolatey/package.git
    3. Under "Credentials", select the SSH credentials we added for Gitlab previously
    4. Click "Save"

###Add a Webhook for Jenkins
In order for Jenkins to be notified when changes to a branch are committed, we need to add a webhook in Gitlab to trigger the Multibranch pipeline to run.

1. Open a browser to the Gitlab repository for the chocolatey package
2. Click on the drop down menu next to the gear icon in the upper right and click “Webhooks”
    1. Enter `http://<jenkins.master.fqdn>:8080/project/<gitlab-repo-name>` for the url
    2. Select “Push Events” and “Merge Request” events under Trigger
    3. De-select “Enable SSL verification” if necessary
    4. Click “Add Webhook”
    5. Click “Test”
    6. Verify "HTTP 200" status is returned

###Create Jenkins credential for Chocolatey API
Before you can push packages to the chocolatey server, you will need to create a Jenkins credential that contains the chocolatey server API key.

1. Go back to the Jenkins web interface, and log in if necessary
2. Click on "Credentials"
3. Click on "System", then "Global Credentials"
4. Create a new [credential](https://wiki.jenkins-ci.org/display/JENKINS/Credentials+Plugin) by clicking on "Add Credentials"
    1. Under "Kind", select "Secret text"
    2. Select a "Scope"
    3. Type in the chocolatey server API key into the "Secret" field
    4. Leave the "ID" field blank
    5. Enter a "Description" for the credential (optional)
    6. Click "OK"

###Clone the chocolatey repository
You will now need to clone the chocolatey project you created earlier to your local workstation.

*Note: A sample Jenkinsfile is included in my repository.*

1. Navigate to the Gitlab project for the chocolatey package
2. Copy the HTTPS URL from the project dashboard
3. Open Git on your computer
4. Change directories to the parent folder where you want the project to be created
5. Type `git clone <url>` where the url is what you copied previously
6. An empty repository should be created
7. Change directories to the new folder
8. Copy the chocolatey tools folder, .nuspec file, Jenkinsfile to this location and any other support files (if necessary) to this folder
9. Edit the Jenkinsfile and replace the contents with your gitlab server info and the credential id that contains your chocolatey server api key (created earlier)
10. Type `git add .` to start tracking all the new files
12. Type `git commit -m "commit message"` to commit the changes
12. Type `git push -u origin master` to push the files to Gitlab
13. This should trigger the Jenkins Multibranch pipeline job created previously
14. If the chocolatey package has no issues, the job should finish successfully and shown as a passing build in Gitlab

###Testing
You should now have a complete pipeline that does the following after committing a chocolatey package to Gitlab.

1. Source control in Gitlab
2. Jenkins will spin up a VM (if configured - not covered in this article) and revert to a clean snapshot
3. Jenkins will create a chocolatey package
4. Jenkins will execute an installation and ensure it is successful
5. Jenkins will execute an uninstallation and ensure it is successful
6. Jenkins will push the tested build to the repository (only if the test was successful on the master branch)

*Note: I put in a conditional statement to push packages to the repository under the master branch to ensure a merge request was approved. This ensures that packages in production are not overwritten while a chocolatey package is undergoing testing. If your protected branch has a different name, make sure to update the Jenkinsfile with the correct name. Also, in order to overwrite existing packages, this option needs to be enabled on the chocolatey server.*
