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




