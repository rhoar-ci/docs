# RHOAR Boosters CI

## Deploy

`oc project xxx` or `oc new-project xxx`

```bash
for S in $(ls secrets/*.yml) ; do oc create -f $S ; done

oc create -f jenkins-master/openshift/image-template.yml

oc process jenkins-image-template NAME=jenkins-slave-jjb   REPO_NAME=jenkins-slave-jjb   | oc apply -f -
oc process jenkins-image-template NAME=jenkins-slave-maven REPO_NAME=jenkins-slave-maven | oc apply -f -
oc process jenkins-image-template NAME=jenkins             REPO_NAME=jenkins-master      | oc apply -f -

oc start-build jenkins-slave-jjb
oc start-build jenkins-slave-maven
oc start-build jenkins

oc create -f jenkins-master/openshift/jenkins-sa-admin.yml

oc new-app --template=jenkins-persistent --param NAMESPACE=$(oc project -q)
oc patch dc/jenkins --patch "$(cat jenkins-master/openshift/patch.yml)"
```

Run the `zygote` job.

To deploy the dashboard, run the `infra-dashboard-master` job.

## Develop

Assumes Minishift.

### `jenkins-master`

Initial OpenShift config:
- same as in _Deploy_:
    - create secrets
    - create Jenkins image streams and builds configs, start Jenkins builds
    - create Jenkins service account admin role binding

Login to Minishift Docker:
- `eval $(minishift docker-env)`
- `docker login -u $(oc whoami) -p $(oc whoami -t) $(minishift openshift registry)`

Build a Jenkins master image:
- `docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins:latest .`
- `docker push $(minishift openshift registry)/$(oc project -q)/jenkins:latest`

Deploy Jenkins master:
- `oc new-app --template=jenkins-ephemeral --param NAMESPACE=$(oc project -q)`
- `oc patch dc/jenkins --patch "$(cat openshift/patch.yml)"`

Changes:
- just `docker build` and `docker push` as above

### `dashboard`

- edit `src/main/resources/project-local.yml`
- `mvn clean package`
- `java -jar target/dashboard-swarm.jar -S local`

### `jenkins-jobs`

First, build the `jenkins-slave-jjb` image locally: `docker build -t my-jjb-image jenkins-slave-jjb` (in the root dir).

- edit `local-config.ini`
- `docker run -it --rm --entrypoint bash --net host -v $(pwd):/home/jenkins/jenkins-jobs my-jjb-image`
- `cd ~/jenkins-jobs`
- `PYTHONHTTPSVERIFY=0 jenkins-jobs --conf local-config.ini update jobs`

## Access to OpenShift Online clusters

- secure an account on the target cluster
- `oc login` to it
- `oc new-project rhoar-ci`
- `oc create serviceaccount rhoar-ci`
- `oc policy add-role-to-user admin system:serviceaccount:rhoar-ci:rhoar-ci`
- `oc get serviceaccount rhoar-ci -o yaml`
- `oc get secret rhoar-ci-token-zzzzz -o jsonpath='{.data.token}'`
