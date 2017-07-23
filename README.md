# Vocab
Docker image
> An Image is an ordered collection of root filesystem changes and the corresponding execution parameters for use within a container runtime.
> An image does not have state and it never changes.

Docker container
> A container is a runtime instance of a docker image.
> 
> A Docker container consists of
> * A Docker image
> * An execution environment
> * A standard set of instructions

Docker registry
> A Registry is a hosted service containing repositories of images.

GitLab
> GitLab is an online Git repository manager with a wiki, issue tracking, CI and CD.

Kubernetes
> Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications.

# Workflow

## Developing Features with the Github Flow
Every feature has it's own branch. When a feature is finished you create a Pull/Merge Request to merge the feature in the master branch. The feature will then be reviewed and the Pull/Merge Request accepted if it is ok.

![Github Flow](https://raw.githubusercontent.com/MarcoGlauser/asdftalk/master/images/github_flow.png)

## Deployment
The master branch represents the staging environment. The production branch represents the production environment. The access to the production branch is heavily restricted.

![Deployment](https://raw.githubusercontent.com/MarcoGlauser/asdftalk/master/images/production_branch.png)

## CI/CD
Every push to the git repository causes a build of the docker image and more importantly also causes the automated testing of the docker image.

![build](https://raw.githubusercontent.com/MarcoGlauser/asdftalk/master/images/branch_build.png)

If the master or production branch have been pushed, the docker image will also be deployed

![build](https://raw.githubusercontent.com/MarcoGlauser/asdftalk/master/images/merge_into_master.png)

## CI/CD Script
Gitlab makes it very easy to create a CI/CD Pipeline described above. An example could look like this:

```yaml
image: apgsga-registry.githost.io/adtech/kubernetes-deployer:1.1

stages:
    - build
    - test
    - staging
    - production

variables:
    REGISTRY_HOST: eu.gcr.io
    REGISTRY_IMAGE: ${REGISTRY_HOST}/${GOOGLE_PROJECT_ID}/${CI_PROJECT_NAME}/${CI_PROJECT_NAME}

before_script:
    - docker login -u _json_key -p "${GOOGLE_REGISTRY_TOKEN}" $REGISTRY_HOST
    - docker login -u gitlab-ci-token -p "$CI_BUILD_TOKEN" $CI_REGISTRY

build image:
    stage: build
    script:
        - docker build -t $REGISTRY_IMAGE:$CI_BUILD_REF .
        - docker tag $REGISTRY_IMAGE:$CI_BUILD_REF $REGISTRY_IMAGE:latest
        - docker push $REGISTRY_IMAGE:$CI_BUILD_REF
        - docker push $REGISTRY_IMAGE:latest

test image:
    stage: test
    script:
        - docker run $REGISTRY_IMAGE:$CI_BUILD_REF /bin/bash -c "coverage run--source='.' manage.py test --noinput --settings=gotthard.settings.test && coverage report -i"

deploy to staging:
    stage: staging
    script:
        - docker tag $REGISTRY_IMAGE:$CI_BUILD_REF ${REGISTRY_IMAGE}-${CI_ENVIRONMENT_SLUG}:$CI_BUILD_REF
        - docker push ${REGISTRY_IMAGE}-${CI_ENVIRONMENT_SLUG}:$CI_BUILD_REF
        - deploy "${CI_PROJECT_NAME}-${CI_ENVIRONMENT_SLUG}-worker"
        - deploy "${CI_PROJECT_NAME}-${CI_ENVIRONMENT_SLUG}-server"
    only:
        - master
    environment:
        name: staging

deploy to production:
    stage: production
    script:
        - docker tag $REGISTRY_IMAGE:$CI_BUILD_REF ${REGISTRY_IMAGE}-${CI_ENVIRONMENT_SLUG}:$CI_BUILD_REF
        - docker push ${REGISTRY_IMAGE}-${CI_ENVIRONMENT_SLUG}:$CI_BUILD_REF
        - deploy "${CI_PROJECT_NAME}-${CI_ENVIRONMENT_SLUG}-worker"
        - deploy "${CI_PROJECT_NAME}-${CI_ENVIRONMENT_SLUG}-server"
    only:
        - production
    environment:
        name: production
```

#### image
This is the identifier of a docker image. This docker image will get the source code injected and then run your build commands.
In this case it is our own kubernetes-deployer, which contains all the binaries and scripts to deploy to kubernetes easily. Since this image can be chosen arbitratrily, it's a good point to hide deployment complexity in.

#### stages
stages defines the different stages of the pipeline and their order

#### variables
variables offers to inject additional environment variables into the build process.
Gitlab injects various variables for every buid(CI_ENVIRONMENT_SLUG, $CI_BUILD_REF, ...)
Because you don't want to store your secrets in your repository, you can additionally define your own secret variables in the gitlab ui.(GOOGLE_REGISTRY_TOKEN)

#### before_script
before_script can be compared to the setup method on unittests. Because every pipeline stage is completely independent,(may even run on a different machine) you'd have to docker login during every single stage.

everything below those is the specific configuation of a stage.
