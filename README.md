# RHOAR Boosters Tests aka _Rourka_

## Deploy

`oc project xxx` or `oc new-project xxx`

`for S in $(ls secrets/*.yml) ; do oc create -f $S ; done`

```bash
oc create -f jenkins-slave-jjb/openshift/image.yml
oc start-build jenkins-slave-jjb

oc create -f jenkins-slave-maven/openshift/image.yml
oc start-build jenkins-slave-maven

oc create -f jenkins-master/openshift/image.yml
oc start-build jenkins-master

oc create -f jenkins-master/openshift/jenkins-sa-admin.yml

oc new-app --template=jenkins-ephemeral --param NAMESPACE=$(oc project -q)
oc patch dc/jenkins --patch "$(cat jenkins-master/openshift/patch.yml)"

oc create -f dashboard/build.yml
oc start-build rourka
```

Run the `zygote` job.

## Develop

Assumes Minishift with a `routing-suffix` of `minishift`.

### `jenkins-master`

Initial OpenShift config:
- same as in _Deploy_:
    - create secrets
    - create Jenkins slave image streams and builds configs, start Jenkins slave builds
    - create Jenkins service account admin role binding

Login to Minishift Docker:
- `eval $(minishift docker-env)`
- `docker login -u $(oc whoami) -p $(oc whoami -t) $(minishift openshift registry)`

Build a Jenkins master image:
- `docker build -t docker-registry-default.minishift:443/myproject/jenkins:latest .`
- `docker push docker-registry-default.minishift:443/myproject/jenkins:latest`

Deploy Jenkins master:
- `oc new-app --template=jenkins-ephemeral --param NAMESPACE=$(oc project -q)`
- `oc patch dc/jenkins --patch "$(cat openshift/patch.yml)"`

Changes:
- just `docker build` and `docker push` as above

### `dashboard`

- edit `src/main/resources/project-local.yml`
- `mvn clean package`
- `java -jar target/rourka-swarm.jar -S local`

### `jenkins-jobs`

- edit `local-config.ini`
- `docker run -it --rm --entrypoint bash --net host -v $(pwd):/home/jenkins/jenkins-jobs ladicek/rourka-jenkins-slave-jjb`
- `cd ~/jenkins-jobs`
- `PYTHONHTTPSVERIFY=0 jenkins-jobs --conf local-config.ini update .`
