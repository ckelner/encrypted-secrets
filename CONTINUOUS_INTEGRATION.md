# Continuous Integration

Detailing how to use the secrets repository with automated/continuous integration environment with a select set of tools (Jenkins, Packer, and Puppet).

For the purposes of this project the CI server that will be used is Jenkins (plus we already had it setup). 

Assumptions made with this approach:
- The CI server has a strong security posture.
- Having CI "know" secrets is acceptable.

As part of keeping Jenkins secure, consider GitHub OAuth Plugin:
http://wiki.jenkins-ci.org/display/JENKINS/Github+OAuth+Plugin

## Table of Contents
- [Puppet](#puppet): Server Configuration
- [Jenkins](#jenkins): Server Provisioning
- [Job 1: Decrypting secrets](#job-1-decrypting-secrets)
  - [Configuration](#configuration)

## Puppet

See: http://docs.puppetlabs.com/puppet/

This example will use masterless puppet for server configuration.

TBD...

## Jenkins

See: https://jenkins-ci.org/

This example uses Jenkins as the "glue" to get the secrets decrypted, run Puppet to configure the server, run Packer to build a machine image, and custom scripts to provision the server.

TBD...

### Job 1: Decrypting secrets

The first job will pull down the secrets repo and unencrypt the secret to be deployed.  This job will be a "free-style software project" job.

Needed Jenkins Plugins:
- GitHub Plugin: https://wiki.jenkins-ci.org/display/JENKINS/GitHub+Plugin

#### Configuration
1. Create a "free-style software project" job.
![Step Two through Four](http://i.imgur.com/3CSaXcBh.png)
2. Fill in the "GitHub project" text fields with the HTTPS URL to the secrets repository.
3. Under the "Source Code Management" heading select "Git".
4. Add the "git@github.com/<project path>.git" for the secrets repository.
![Step Five through Eight](http://i.imgur.com/sBRjA2nh.png)
5. Under the "Build" heading add an "Execute Shell" build step.
6. Copy the "smudge_filter_openssl" file contents into the "Execute Shell" "Command" textfield.
7. Replace the "$PASS_FIXED" value with your password; Alternatively it should be set as an environment variable on the Jenkins server.
8. In the example screenshot above we decrypting exactly one file for our project by specifying the file's path using the "-in" flag and redirecting output to a new file.
9. Add a "Post-build Action" of type "Archive the artifacts" -- then specify the file name you redirected output to for decryption from step eight.  This will make the decrypted file available to other Jenkin's jobs.
