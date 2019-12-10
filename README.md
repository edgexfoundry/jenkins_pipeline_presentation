# Geneva Transformation

## Table of Contents

* [Geneva Transformation](#geneva-transformation)
  * [Authors](#authors)
  * [Background](#background)
  * [Plugins](#plugins)
  * [Job types](#job-types)
    * [Verify Job](#verify-job)
    * [Merge Job](#merge-job)
    * [Stage Job](#stage-job)
  * [What changes with Jenkins pipeline](#what-changes-with-jenkins-pipeline)
  * [Jenkins UI Navigational Changes](#jenkins-ui-navigational-changes)
    * [Jenkins main page (Mock Up)](#jenkins-main-page-mock-up)
    * [EdgeX Foundry Organization (Mock Up)](#edgex-foundry-organization-mock-up)
  * [git-semver](#git-semver)
    * [What is git-semver?](#what-is-git-semver)
    * [How does it work?](#how-does-it-work)
  * [How Pipeline and git-semver Simplify the Release Management Process](#how-pipeline-and-git-semver-simplify-the-release-management-process)
    * [Nexus Image Tracking](#nexus-image-tracking)
    * [Complete Release Automation](#complete-release-automation)
    * [git-semver FAQ:](#git-semver-faq)
  * [New Geneva Pre-Built Jenkins Pipelines](#new-geneva-pre-built-jenkins-pipelines)
    * [edgeXBuildGoApp](#edgexbuildgoapp)
    * [edgeXBuildDocker](#edgexbuilddocker)
    * [edgeXGeneric](#edgexgeneric)
  * [Migrating Existing Pipelines](#migrating-existing-pipelines)
    *+* [Examples](#examples)
  * [New CI Requirements](#new-ci-requirements)
    * [All Repositories](#all-repositories)
    * [Microservice Type Repositories](#microservice-type-repositories)
    * [Leverage Docker as the build runtime, rather than underlying host.](#leverage-docker-as-the-build-runtime-rather-than-underlying-host)
      * [Benefits](#benefits)
    * [Implementation](#implementation)
    * [Building the base image](#building-the-base-image)
    * [Building the final image](#building-the-final-image)
    * [Build base image Dockerfile.build](#build-base-image-dockerfilebuild)
    * [Build final docker image](#build-final-docker-image)

## Authors

* Ernesto Ojeda
* Lisa Rashidi-Ranjbar
* James Gregg
* Emilio Reyes

## Background

* What is a Jenkinsfile?
  * A Jenkinsfile is a text file that contains the build definition (all the components required to build and deploy the source code of your project) of a Jenkins Pipeline and is checked into source control. A Jenkinsfile is written in Groovy. 

  * Immediate benefits of a Jenkinsfile include:
    * Code review/iteration on the Pipeline
    * Audit trail for the Pipeline
    * Single source of truth for the Pipeline, which can be viewed and edited by multiple members of the project.

### Example: Jenkinsfile (Declarative Pipeline)
```
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
```
* Difference/benefits between JJB freestyle and Jenkins Pipeline.
  * **JJB Freestyle**: Yaml based, managed separate from source code that it builds. Hard to trace through all the templates and scripts. Relies on old paradigms to manage Jenkins jobs.
  * **Jenkins Pipeline**: Suite of plugins that enables **[Pipeline-As-Code](https://jenkins.io/doc/book/pipeline-as-code/)**. It allows users to define a build descriptor (Jenkinsfile) right next to the code to more easily manage code builds. More industry adoption/support. Officially supported by Cloudbees (makers of Jenkins).

## Plugins
  * Before: [GitHub Pull Request Builder](https://github.com/jenkinsci/ghprb-plugin) aka. GHPRB. This plugin is currently responsible for building all the pull requests. Plugin is no longer being actively developed and is up for adoption. Comments direct users to use the GitHub Branch Source plugin. Automatic builds on code push and allows triggering via GitHub comments (i.e. recheck).
  * After: [GitHub Branch Source](https://docs.cloudbees.com/docs/admin-resources/latest/plugins/github-branch-source). Automatic scanning of GitHub at the organization level. Any branch or PR where a Jenkinsfile is found will automatically be built. Builds are triggered via GitHub webhooks, untrusted code, i.e. code that is committed by users without committer privileges, would require a manual trigger via the Jenkins UI.

## Job types

### Verify Job
  * **Summary**: This is a Jenkins freestyle job created per branch/release stream (master, edinburgh, fuji, etc). This job typically does unit tests and builds the code and docker images.
  * **Trigger**: [GitHub Pull Request Builder](https://github.com/jenkinsci/ghprb-plugin) plugin. Triggered when a user opens a pull request or recheck.
  * **Management**: JJB template in the ci-management repository.
  * **Example**: ([edgex-go-master-verify-go](https://jenkins.edgexfoundry.org/job/edgex-go-master-verify-go/)). 

### Merge Job
  * **Summary**: This is a Jenkins freestyle job created per branch/release stream (master, edinburgh, fuji, etc). This job typically does the same steps as the **verify** job and will push any generated docker images/artifacts to the **snapshots** nexus3 and nexus2 repositories.
  * **Trigger**: [GitHub Webhook](https://help.github.com/en/github/extending-github/about-webhooks). Triggered when a pull request is merged into **master/release stream branch** (master, edinburgh, fuji, etc).
  * **Management**: JJB template in ci-management.
  * **Example**: [edgex-go-master-merge-go](https://jenkins.edgexfoundry.org/job/edgex-go-master-merge-go/). 

### Stage Job
  * **Summary**: This is a Jenkins freestyle job created per branch/release stream (master, edinburgh, fuji, etc). Similar to the merge job, this job typically does the same steps as the **verify** job and will push any generated docker images/artifacts to the **staging** nexus3 and nexus2 repositories.
  * **Trigger**: Nightly cron.
  * **Management**: JJB template in ci-management.
  * **Example**: [edgex-go-master-stage-go](https://jenkins.edgexfoundry.org/job/edgex-go-master-stage-go/). 

## What changes with Jenkins pipeline
* **Q: For Geneva jobs will there still be need a need for verify, merge and stage jobs?**
* A: No. All details related to build verification, pushing docker images will be moved into the Jenkinsfile. Also, stage jobs will no longer be needed as we will be pushing docker images to nexus staging on merges into master.
* **Q: Will there still be a need for any JJB templates?**
* A: Yes. Some JJB templates will remain for older releases, i.e. Edinburgh, Fuji. In addition, there will be one JJB template for the edgexfoundry GitHub organization as a whole which will discover branches to build.
* **Q: Will it be easier to find build failures?**
* A: Yes, Jenkins pipeline defines the concept of named stages which can help pinpoint where failures are happening more easily.
* **Q: Will it be simpler to navigate jobs?**
* A: Yes, there will be only one high level job and you will be able to easily navigate to the branch/PR  you are concerned about.
* **Q: For Geneva jobs can I still trigger the build with a 'recheck'?**
* A: No, just pushing your branch will trigger the build. If you require a new build and the code has not changed, you can push and empty commit.

### Jenkins UI Navigational Changes

#### Jenkins main page (Mock Up)

![edgeXOrg](images/edgeXOrg.png)

#### EdgeX Foundry Organization (Mock Up)

![edgeXOrg](images/edgeXOrgRepos.png)

## git-semver

### What is git-semver?

A command line tool that tracks semantic versioning for a git repository.

### How does it work?

Git-semver uses a detached branch in the git repository called `semver` to track the semantic versioning for the repository. In this `semver` branch git-semver will create files that match the name of the branch to track the **next** version for the branch.

In this screenshot you can see that git-semver is tracking different versions for two branches: `master` and `edinburgh`:

![gitsemver_semverBranch.png](images/gitsemver_semverBranch.png)

By using a detached branch we are able to keep commits related to the versioning out of the main commit stream. 

## How Pipeline and git-semver Simplify the Release Management Process 

### Nexus Image Tracking

For the repositories that have git-semver and a pipeline already set up we are able to easily assoicate the images in Nexus with our commits in Github. Using `support-rulesengine` as an example:

![gitsemver_releases.png](images/gitsemver_releases.png)

### Complete Release Automation

Moving to Jenkins pipelines allows the DevOps team to get closer to automatic releases. For Go Modules we are already doing automatic releases because we are simply releasing git tags only. If you look at the Go Modules we have set them to auto release on every merge to master.

![gitsemver_gomodules.png](images/gitsemver_gomodules.png)

### git-semver FAQ:

* Q: **What happens to the `VERSION` file?** 
* A: After turning on git-semver in a repository the correct thing to do is to delete the `VERSION` file out of the main directory and add it .gitignore.

* Q: **How does git-semver change how we branch?** 
* A: Since git-semver allows us to track the semantic version outside of the main commit stream we do not have to branch to release a new version. We are able to release directly off the master branch. This allows us to branch only when we need to make changes to an already released version.

## New Geneva Pre-Built Jenkins Pipelines

Geneva introduces a set of pre-built Jenkins pipelines for use by the EdgeX community. These pipelines 

### edgeXBuildGoApp

New pipeline that will be used for Go microservices. Leverages Docker to test/build Go code. Includes a Clair docker image scan as well as Snyk security scanning of Go dependencies. Uses git-semver to manage versions by default.

![edgeXBuildGoApp.png](images/edgeXBuildGoApp.png)

### edgeXBuildDocker

A generic pipeline that builds and scans docker images. Can be used to build any sort of docker image. Optionally, git-semver can be used as well to manage versions.

![edgeXBuildDocker.png](images/edgeXBuildDocker.png)

### edgeXGeneric

Easily convert existing JJB templates to pipelines. Will be used for more complicated jobs as a temporary stop-gap until a more custom pipeline can be created.

![edgeXGeneric.png](images/edgeXGeneric.png)


## Migrating Existing Pipelines

### Examples

* Before Jenkinsfile
  * [app-service-configurable](samples/app-service-configurable/Jenkinsfile.before)
  * [device-bacnet-c](samples/device-bacnet-c/Jenkinsfile.before)
* After Jenkinsfile
  * [app-service-configurable](samples/app-service-configurable/Jenkinsfile.after)
  * [device-bacnet-c](samples/device-bacnet-c/Jenkinsfile.after)

## New CI Requirements

### All Repositories

* Jenkinsfile at the root of the repository.
* git-semver manages versions wherever possible.
* Repository must contain a Makefile
* Standard Makefile targets:
  * **test**: Run unit tests
  * **build**: Compile code
  * **version**: `cat VERSION 2>/dev/null || echo 0.0.0`. In this case the VERSION file is only used in the Jenkins pipeline and git-semver automatically writes the file to the workspace. When running locally a developer's version would be 0.0.0. This could be overridden by the developer if they so care to do so.

### Microservice Type Repositories

### Leverage Docker as the build runtime, rather than underlying host.

#### Benefits

* Optimization of builds due to caching of docker image layers.
* Installation of custom tooling can be done inside of docker rather than Packer...allowing for easier re-use.

### Implementation

* **Dockerfile.build**: containing dependencies needed for code build. This image is used to run `make test` and/or `make build`.
* **Dockerfile**: multi-stage build. Parameterized `FROM` would allow reusing the image generated from Dockerfile.build as the builder image. Example:

#### Dockerfile.build

```Dockerfile
FROM golang:1.12-alpine
RUN apk add --update make git zmq-dev

COPY . .

RUN go mod download
```

### Building the base image

The CI process now creates the build image and subsequent make commands are executed inside a running container.

`docker build -t base-build -f Dockerfile.build .`

Then run make commands inside the base image

`docker run --rm -t -v /path/to/workspace:/path/to/workspace base-build make test && make ...`

### Building the final image

#### Dockerfile

```Dockerfile
ARG BASE=golang:1.12-alpine
FROM ${BASE} as builder
RUN apk add --update make git zmq-dev
COPY . .
RUN make build

FROM alpine
COPY --from=builder /go-binary /go-binary
ENTRYPOINT ["/go-binary"]
```

### Build base image Dockerfile.build

`docker build -t base-build -f Dockerfile.build .`

### Build final docker image

`docker build -t docker-edgex-binary -f Dockerfile --build-arg BASE=base-build .`
