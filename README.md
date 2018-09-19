# RHOAR Boosters CI

## Deploy

`oc project xxx` or `oc new-project xxx`

```bash
for S in $(ls secrets/*.yml) ; do oc create -f $S ; done

oc create -f jenkins-master/openshift/image-template.yml

oc process jenkins-image-template NAME=jenkins-agent-jjb    REPO_NAME=jenkins-agent-jjb    | oc apply -f -
oc process jenkins-image-template NAME=jenkins-agent-maven  REPO_NAME=jenkins-agent-maven  | oc apply -f -
oc process jenkins-image-template NAME=jenkins-agent-nodejs REPO_NAME=jenkins-agent-nodejs | oc apply -f -
oc process jenkins-image-template NAME=jenkins              REPO_NAME=jenkins-master       | oc apply -f -

oc start-build jenkins-agent-jjb
oc start-build jenkins-agent-maven
oc start-build jenkins-agent-nodejs
oc start-build jenkins

oc create -f jenkins-master/openshift/jenkins-sa-admin.yml

oc new-app --template=jenkins-persistent --param NAMESPACE=$(oc project -q) --param JENKINS_IMAGE_STREAM_TAG=jenkins:latest
oc patch dc/jenkins --patch "$(cat jenkins-master/openshift/patch.yml)"
```

Run the `zygote` job.

To deploy the dashboard, run the `infra-dashboard-master` job.

Configure webhooks on GitHub for the `jenkins-agent-*` and `jenkins` build configs.

## Develop

Assumes Minishift.

Initial config, in the root directory:

```bash
for S in $(ls secrets/*.yml) ; do oc create -f $S ; done

oc create -f jenkins-master/openshift/jenkins-sa-admin.yml
```

Login to Minishift Docker:

```bash
eval $(minishift docker-env)
docker login -u $(oc whoami) -p $(oc whoami -t) $(minishift openshift registry)
```

The following always happens in the respective component's directory, unless stated otherwise.

### `jenkins-agent-*`

Jenkins Job Builder:

```bash
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins-agent-jjb:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins-agent-jjb:latest
```

Maven:

```bash
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins-agent-maven:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins-agent-maven:latest
```

Node.js:

```bash
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins-agent-nodejs:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins-agent-nodejs:latest
```

### `jenkins-master`

Build a Jenkins master image:

```bash
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins:latest
```

Deploy Jenkins master:

```bash
oc new-app --template=jenkins-persistent --param NAMESPACE=$(oc project -q) --param JENKINS_IMAGE_STREAM_TAG=jenkins:latest
oc patch dc/jenkins --patch "$(cat openshift/patch.yml)"
```

Changes:
- just `docker build` and `docker push` as above

### `dashboard`

- edit `src/main/resources/project-local.yml`
- `mvn clean package`
- `java -jar target/dashboard-swarm.jar -S local`

### `jenkins-jobs`

First, build the `jenkins-agent-jjb` image locally: `docker build -t my-jjb-image jenkins-agent-jjb` (in the root dir,
in a terminal session that _didn't_ run `eval $(minishift docker-env)`).

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
- `oc get secret rhoar-ci-token-zzzzz -o jsonpath='{.data.token}' | base64 -d`
