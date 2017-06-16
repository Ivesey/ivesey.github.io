---
layout: post
title:  DevOps Engineering v1.5 Alternative Lab 3 Using Code Build
subtitle: Replace Jenkins with CodeBuild
date:   2016-12-07
category: training
tags: [labguides, devops, codebuild]
---

## Task 0: Give yourself permissions
You aren't explicitly denied access to CodeBuild in this lab, but because the IAM policy predates CodeBuild, you aren't explicitly allowed permission either, so you're _implicitly_ denied access. This procedure, not to be performed in the real world, is the easiest way around the problem (but not the correct way around it).

1. Go into the IAM console.
1. Choose "users" in the Navigation section.
1. Click on "awsstudent" in the list of users.
1. Click the "Add permissions" button.
1. Choose "Attach existing policies directly".
1. Find the "AdministratorAccess" policy and select it.
1. Click the "Next: Review" button.
1. Click the "Add permissions" button.

## Task 1.1
Still need to do all this as per the Lab Guide.

## Task 1.2
__NOTE:__ If you're using Windows, it is very important that you follow the instructions in Appendix A to connect to your instance. If you're using Putty, it is vital that you use Pageant to manage your keys and that you enable Agent Forwarding. If you don't, this step will go horribly wrong and basically it's a case of ENDing the lab and then STARTing it all over again.

Still need to do all this as per the Lab Guide.

## Task 1.3
Still need to do all this as per the Lab Guide.

## Task 1.4 - Configure a CodeBuild project
So we're going to ditch Jenkins like a bad date.

1. Go into the CodeBuild console
1. Click the "Get started" button. If there isn't one, click the "Create Project" button.
1. Configure your Project, using the following settings:
  - _Project name_ - Lab3Project (or anything else you feel like calling it)
  - _Source provider_ - AWS CodeCommit
  - _Repository_ - choose your repo from the list
  - _Environment image_ - leave the default managed image selected
  - _Operating system_ - leave as Ubuntu
  - _Runtime_ - choose "Base"
  - _Version_ - choose 14.04
  - _Build specification_ - choose "Use the buildspec..."
  - _Artifacts type_ - choose "No artifacts"
  - Allow AWS to create the service role for you.
1. Click "Continue".
1. Click "Save" (__not__ save and build).

## Task 1.4.3 - Allow CodeBuild to validate templates
The auto-generated CodeBuild role won't have permissions to validate your template, so it will fail to fail correctly (yes we are expecting it to fail first time around...), so you'll need to give it permissions.

Again, we'll do it the easy way rather than the right way.

1. Go into the IAM console.
1. Choose "Roles" in the Navigation section.
1. Click on "codebuild-_project-name_-service-role" in the list of roles (replace _project-name_).
1. Click the "Permissions" tab.
1. Click the "Attach policy" button.
1. Find the "AdministratorAccess" policy and select it.
1. Click the "AttachPolicy" button.

## Task 1.4.4 - Set up support for CodeBuild

1. Create a "buildspec.yml" file in the root of your repo (/opt/git/ci-project)
  - You could try `touch ./buildspec.yml`
1. Copy the contents of the ["lab-3-ci-buildspec.yml"](/assets/2016-12-07/lab-3-ci-buildspec.yml.txt) file into it
  - __NOTE:__ if you are not running in us-east-1 (Virginia), you may need to update the `environment_variables:` section of the buildspec file appropriately.
1. Copy the contents of the ["lab-3-ci-simple-test.sh"](/assets/2016-12-07/lab-3-ci-simple-test.sh.txt) file into a file of the same name in the root of your repo.
1. Execute the following command to make it executable:
  - `chmod +x ./lab-3-ci-simple-test.sh`
1. Execute the following commands to push the new files into your CodeCommit repo:
  - `git add .`
  - `git commit -m "added support for CodeBuild"`
  - `git push origin newWidget`

## Task 1.4.5 - Submit a Build
We'll now attempt to build the newWidget branch.

1. In the CodeBuild console's navigation pane, select "Build projects"
1. With your Lab3Project selected, click the "Start build" button.
1. In the "Start new build" wizard, accept all of the defaults except for:
  - _Branch_ - select "newWidget" from the dropdown list.
1. Click the "Start build" button.

## Task 1.5 - Validate and repair
You'll note that, unlike with the official lab instructions, your build is not running automatically. This is because the glue that binds Jenkins and CodeCommit is Jenkins, and we're not using Jenkins any more. The solution in this case is to use CodePipeline, but as we haven't covered that service yet, I've left those instructions out. In the meantime, feel free to try and set it up yourself _once you've got the standard lab working_.

1. Check the build progress of your CodeBuild project.
1. After a couple of minutes, it should move into a "Failed" state.
1. The build logs might show you the problem, but more likely you'll need to click on the "View entire log" link to get to the bottom of things. This will open a new console windows showing you the CloudWatch logs for your project.
1. Around half-way down, you should see an entry beginning with "An error occurred (ValidationError)..."
  - expand that entry to observe the text:
  - [Container] yyyy/MM/dd hh:mm:ss An error occurred (ValidationError) when calling the ValidateTemplate operation: Template error: resource TestWebInstance does not support attribute type __PublicIpAddress__ in __Fn::GetAtt__ ?????
1. To correct the issue, follow steps 1.5.4 to 1.5.11 (inclusive) from the official Lab Guide, and then re-submit the build job as per Task 1.4.5 of this guide.
1. After a few minutes, the build should have succeeded! Yay!

## Task 1.6 - Merge, Validate, Deploy
The newWidget seems to work, so we need to create a new CodeBuild project for the master branch, merge our changes with that branch and then kick off another build. Again, we could integrate all of this with CodePipeline and that should be the goal...

1. Go to the CloudFormation console, select your qwiklab stack and find the Resource whose Logical ID is "WebInstanceEC2InstanceProfile". It should be down towards the bottom of the list.
  1. Copy its Physical Id into a handy text file.
  1. It should look something like "qls-12345-0123456789abcdef-WebInstanceEC2InstanceProfile-12345ABCDE"
  1. I'll refer to this as "_WebProfile_" later on.
1. Copy the Physical Id for the s3ArtifactBucket resource. I'll refer to that as "_S3Bucket_" later on.
1. Now go to the "Parameters" tab and copy the Value of the KeyName into your text file. This will be known as "_KeyName_".
1. Create a new CodeBuild project.
  - Call it Lab3DeployStaging (or whatever else you'd like to call it).
  - Choose CodeCommit for the source provider
  - Select your repo name from the drop-down list
  - Use the Ubuntu image and the Base runtime, v 14.04
  - In _Build specification_, change the radio button to "Insert build commands"
  - Use the following command: `/bin/bash ./lab-3-ci-job-script.sh`
  - No artifacts
  - Choose the previously-generated service role from your account (there should only be one item in the drop-down list)
1. Now expand the "Show advanced settings" section
1. Scroll down to the "Environment variables" section and add the following four variables, then "Continue" and "Save":

Name | Value
---- | ---------
REGION | us-east-1
KEYNAME | Paste the  _KeyName_ value in here
ARTIFACT_BUCKET | Paste the _S3Bucket_ value in here
WEBInstanceROLE | Paste the _WebProfile_ value in here
{:.mbtablestyle}


## Task 1.6.1 - Merge
That script we told CodeBuild to use doesn't exist yet. We'll need to create it and make it executable. But we don't need it in the newWidget branch, only in the master. So we'll switch back to the master branch, merge our changes, and then create the file, push the new version and then manually build it using CodeBuild (again, CodePipeline would do this automatically for us).

Head back to your SSH session from earlier (you should still be seeing the results of your previous push to newWidget).

1. Execute the following two commands (from 1.6.16 and 1.6.17 in the Lab Guide):
  - `git checkout master`
  - `git merge newWidget`
1. Copy the contents of the ["lab-3-ci-job-script.sh"](/assets/2016-12-07/lab-3-ci-job-script.sh.txt) file into a file of the same name in the root of your repo.
1. Execute the following command to make the script executable:
  - `chmod +x ./lab-3-ci-job-script.sh`
1. Add the file to git:
  - `git add ./lab-3-job-script.sh`
1. Commit the changes:
  - `git commit -m "added build script"`
1. And push to master:
  - `git push origin master`

## Task 1.6.2 - Validate and Deploy

1. In the CodeBuild console, manually build the project (choosing the `master` branch)
1. After a while, the build process should have emitted a log entry with the URL of your application.
1. Browse to that URL.
1. You're done!
  - [...unless you want to try to automate the process and time allows it...]
