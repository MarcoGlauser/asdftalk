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

## Dockerfile
```
FROM apgsga-registry.githost.io/adtech/django-base-image:1.0
MAINTAINER Marco Glauser

RUN apt-get update && apt-get install -y --no-install-recommends git

COPY ./requirements.txt $SERVICE_HOME/requirements.txt
RUN pip install -r $SERVICE_HOME/requirements.txt

COPY . $SERVICE_HOME

RUN python $SERVICE_HOME/manage.py collectstatic --noinput  # Collect static files

CMD ["/src/uwsgi_entrypoint.sh"]
```

## Docker Build output
```
docker build -t $REGISTRY_IMAGE:$CI_BUILD_REF --pull .
Sending build context to Docker daemon  2.514MB

Step 1/8 : FROM apgsga-registry.githost.io/adtech/django-base-image:1.0
1.0: Pulling from adtech/django-base-image
Digest: sha256:a55b83e0365f97d414b892981da3a0c779a927f87bc68cb820d0412fcaa4430a
Status: Image is up to date for apgsga-registry.githost.io/adtech/django-base-image:1.0
 ---> 04194636f674
Step 2/8 : MAINTAINER Marco Glauser
 ---> Using cache
 ---> 1fedc878f842
Step 3/8 : RUN apt-get update && apt-get install -y --no-install-recommends git
 ---> Using cache
 ---> 567474d7596d
Step 4/8 : COPY ./requirements.txt $SERVICE_HOME/requirements.txt
 ---> Using cache
 ---> 3c6185189aac
Step 5/8 : RUN pip install -r $SERVICE_HOME/requirements.txt
 ---> Using cache
 ---> de1326081e1a
Step 6/8 : COPY . $SERVICE_HOME
 ---> f18a83c0ca6e
Removing intermediate container 55acc181a857
Step 7/8 : RUN python $SERVICE_HOME/manage.py collectstatic --noinput  # Collect static files
 ---> Running in 4cd69ef228f3
409 static files copied to '/static', 409 post-processed.
 ---> c2e31a6e67d8
Removing intermediate container 4cd69ef228f3
Step 8/8 : CMD /src/uwsgi_entrypoint.sh
 ---> Running in 3ec43f4a16cc
 ---> 4c06e5dd71cf
Removing intermediate container 3ec43f4a16cc
Successfully built 4c06e5dd71cf
```
## Docker push output
```
$ docker push $REGISTRY_IMAGE:$CI_BUILD_REF
The push refers to a repository [eu.gcr.io/adtech-158707/gotthard/gotthard]
4b0f688d1fb9: Preparing
95d5cb3b10f1: Preparing
698d98a38b6e: Preparing
8e9867326f57: Preparing
c563335ade18: Preparing
bef3aa4a7aca: Preparing
82b082d5b92f: Preparing
6f735618a40e: Preparing
3f24b090534b: Preparing
6ba7a2c38d3d: Preparing
7205e853d9d6: Preparing
593e094824d2: Preparing
a2ae92ffcd29: Preparing
bef3aa4a7aca: Waiting
82b082d5b92f: Waiting
6f735618a40e: Waiting
3f24b090534b: Waiting
6ba7a2c38d3d: Waiting
7205e853d9d6: Waiting
593e094824d2: Waiting
a2ae92ffcd29: Waiting
8e9867326f57: Layer already exists
698d98a38b6e: Layer already exists
c563335ade18: Layer already exists
bef3aa4a7aca: Layer already exists
6f735618a40e: Layer already exists
82b082d5b92f: Layer already exists
7205e853d9d6: Layer already exists
6ba7a2c38d3d: Layer already exists
3f24b090534b: Layer already exists
a2ae92ffcd29: Layer already exists
593e094824d2: Layer already exists
95d5cb3b10f1: Pushed
4b0f688d1fb9: Pushed
3c60211292d79a7ee9cd1f8dcea732498630b2ce: digest: sha256:a17473aae61f5a80023b1c0da8d0d9bca97e2a872f4bacff51e7577e126974c6 size: 3051
```

## Deploy output
```
$ deploy "${CI_PROJECT_NAME}-${CI_ENVIRONMENT_SLUG}-worker"
Generating kubeconfig...
Using KUBE_CA_PEM...
Cluster "gitlab-deploy" set.
User "gitlab-deploy" set.
Context "gitlab-deploy" set.
Switched to context "gitlab-deploy".

deployment "gotthard-staging-worker" image updated
Waiting for deployment...
deployment "gotthard-staging-worker" successfully rolled out
```

```
$ deploy "${CI_PROJECT_NAME}-${CI_ENVIRONMENT_SLUG}-server"
Generating kubeconfig...
Using KUBE_CA_PEM...
Cluster "gitlab-deploy" set.
User "gitlab-deploy" set.
Context "gitlab-deploy" set.
Switched to context "gitlab-deploy".

deployment "gotthard-staging-server" image updated
Waiting for deployment...
deployment "gotthard-staging-server" successfully rolled out
```

# Learnings
* Create a Docker base image

  Docker offers inheritance of docker images. You should create a common base image containing your stack. Every Service with the same stack will inherit from the base image. This has a big impact on build time and caching.

* Have one configuration for all environments

  Inject the environment specific configurations and secrets through files or environment variables. This makes your service more portable and you can change the configuration without a new deployment.

* Don't use `latest` Docker tag

  Create a unique tag for each build. If you have a severe bug in an image and need to roll back, you can simply roll back to an old tag. You don't have to build the image all over again.
  Some systems cache the tags and don't pull a tag if it already exists.
  Tags are often used for versioning. (for example pyton:3.5)
