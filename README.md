# stingar-ci-template

[![pipeline status](https://gitlab.oit.duke.edu/stingar/stingar-ci-template/badges/master/pipeline.svg)](https://gitlab.oit.duke.edu/stingar/stingar-ci-template/commits/master)

## Overview

CI is done on disposable VMs.  Each time the CI process is run, a base VM is
cloned, run, then deleted

Want that cool looking badge in your readme?  Check for the markdown link in Settings -> CI/CD

## Setup

### Github

First, ensure that the 'duke-stingar' user has admin rights to the repository.
This is required to allow gitlab to create and manage the webhooks

Settings -> Collaborators & Teams -> Add duke-stingar as Admin


### Gitlab

* Create a new project and select CI/CD for external repo

* Select 'GitHub'

* Enter the duke-stingar access token (`vault read -field gitlab-token secret/stingar/github-account`)

* Press the 'Connect' button next to the github project you want to do CI/CD for

## CI/CD Configuration

All configuration is stored in .gitlab-ci.yml

### PEP8 Testing

Ensures that all python code complies with pep8 standards

```yaml
pep8:
  stage: test
  tags:
    - linting

  before_script:
    - flake8 --version

  script:
    - flake8 .
```

### Container Building

This builds a container based on the Dockerfile in the root of your repository,
and pushes it to the gitlab container registry.  Note that the CI\_\* variables
are inserted automatically by gitlab

```yaml
build_container:
  stage: build
  tags:
    - container_scanning
  services:
    - docker:stable-dind
  script:
    - sudo docker login -u "gitlab-ci-token" -p "${CI_BUILD_TOKEN}" ${CI_REGISTRY}
    - sudo docker build --pull -t "${CI_APPLICATION_REPOSITORY}:${CI_APPLICATION_TAG}" .
    - sudo docker push "${CI_APPLICATION_REPOSITORY}:${CI_APPLICATION_TAG}"
```

### Dependency Scanning

Scans dependant libraries for security issues.  [Documentation](https://docs.gitlab.com/ee/user/project/merge_requests/dependency_scanning.html)

```yaml
dependency_scanning:
   image: docker:stable
   variables:
     DOCKER_DRIVER: overlay2
   allow_failure: true
   services:
     - docker:stable-dind
   tags:
     - dependency_scanning
   script:
     - export SP_VERSION=$(echo "$CI_SERVER_VERSION" | sed 's/^\([0-9]*\)\.\([0-9]*\).*/\1-\2-stable/')
     - sudo docker run
         --env DEP_SCAN_DISABLE_REMOTE_CHECKS="${DEP_SCAN_DISABLE_REMOTE_CHECKS:-false}"
         --volume "$PWD:/code"
         --volume /var/run/docker.sock:/var/run/docker.sock
         "registry.gitlab.com/gitlab-org/security-products/dependency-scanning:$SP_VERSION" /code
   artifacts:
     paths: [gl-dependency-scanning-report.json]
```

### Static Application Security Testing (SAST)

Scans the static code in your application for security issues and/or bad practices [Documentation](https://docs.gitlab.com/ee/user/project/merge_requests/sast.html)

```yaml
sast:
  image: docker:stable
  tags:
    - sast
  allow_failure: true
  services:
    - docker:stable-dind
  script:
    - export SP_VERSION=$(echo "$CI_SERVER_VERSION" | sed 's/^\([0-9]*\)\.\([0-9]*\).*/\1-\2-stable/')
    - sudo docker run
        --env SAST_CONFIDENCE_LEVEL="${SAST_CONFIDENCE_LEVEL:-3}"
        --volume "$PWD:/code"
        --volume /var/run/docker.sock:/var/run/docker.sock
        "registry.gitlab.com/gitlab-org/security-products/sast:$SP_VERSION" /app/bin/run /code
  artifacts:
    paths: [gl-sast-report.json]
```

### License Management

Looks at the licenses in the code dependancies [Documentation](https://docs.gitlab.com/ee/user/project/merge_requests/license_management.html)

```yaml
license_management:
  image: docker:stable
  stage: test
  tags:
    - license_management
  allow_failure: true
  services:
    - docker:stable-dind
  script:
    - export LICENSE_MANAGEMENT_VERSION=$(echo "$CI_SERVER_VERSION" | sed 's/^\([0-9]*\)\.\([0-9]*\).*/\1-\2-stable/')
    - sudo docker run
        --volume "$PWD:/code"
        "registry.gitlab.com/gitlab-org/security-products/license-management:$LICENSE_MANAGEMENT_VERSION" analyze /code
  artifacts:
    paths: [gl-license-management-report.json]
```

### Container Scanning

Scans the container using an embeded version of [Documentation](https://docs.gitlab.com/ee/ci/examples/container_scanning.html)

```yaml
container_scanning:
  image: docker:stable
  allow_failure: true
  services:
    - docker:stable-dind
  tags:
    - container_scanning
  script:
    - env
    - sudo docker pull ${CI_APPLICATION_REPOSITORY}:${CI_APPLICATION_TAG}
    - sudo docker run -d --name db arminc/clair-db:latest
    - sudo docker run -p 6060:6060 --link db:postgres -d --name clair --restart on-failure arminc/clair-local-scan:v2.0.5
    - sudo wget https://github.com/arminc/clair-scanner/releases/download/v8/clair-scanner_linux_amd64
    - sudo mv clair-scanner_linux_amd64 clair-scanner
    - sudo chmod +x clair-scanner
    - sudo touch clair-whitelist.yml
    - sudo docker ps
    - while( ! wget -O /dev/null http://127.0.0.1:6060/v1/namespaces ) ; do sleep 1 ; done
    - retries=0
    - echo "Waiting for clair daemon to start"
    - while( ! wget -T 10 -q -O /dev/null http://127.0.0.1:6060/v1/namespaces ) ; do sleep 1 ; echo -n "." ; if [ $retries -eq 10 ] ; then echo " Timeout, aborting." ; exit 1 ; fi ; retries=$(($retries+1)) ; done
    - sudo ./clair-scanner -c http://127.0.0.1:6060 --ip "$(hostname -I | awk '{print $1}')" -r gl-container-scanning-report.json -l clair.log -w clair-whitelist.yml ${CI_APPLICATION_REPOSITORY}:${CI_APPLICATION_TAG} || true
  artifacts:
    paths: [gl-container-scanning-report.json]
```

*Note* This one is a little funky.  It is doing a fresh install of Clair and clair-scanner every run.  Tried to work around this by using an external instance of Clair, however that would never work quite right in this instance, because clair-scanner is also opening up a port that the Clair host must connect back to.  We don't have a way to do that currently, using these disposable VMs
