# RHOAR Boosters CI

If you already know what this is and how it works, you might want to head over to [quick summary](SUMMARY.md).

## Intro

This is a CI infrastructure for RHOAR boosters.
It's built on top of OpenShift and Jenkins.
The Jenkins master runs permanently in OpenShift, while the Jenkins agents are provisioned dynamically in OpenShift using the Kubernetes plugin.
Jobs are defined using Jenkins Job Builder.
A dedicated dashboard is provided to present Jenkins jobs results in a more accessible way.

Structure of repositories in the GitHub `rhoar-ci` organization:

- `docs`: this documentation
- `jenkins-master`: Docker image for the Jenkins master, plus a few small utilities required for OpenShift deployment
- `jenkins-agent-maven`: Docker image for a Jenkins agent dedicated for Maven-based tests (tests for boosters for Java-based runtimes)
- `jenkins-agent-nodejs`: Docker image for a Jenkins agent dedicated for Node.js-based tests (end-to-end tests for all boosters)
- `jenkins-agent-jjb`: Docker image for a Jenkins agent dedicated to running Jenkins Job Builder
- `jenkins-jobs`: definition of all Jenkins jobs in the JJB format
- `dashboard`: the dashboard application that presents Jenkins results in a more accessible way
- `secrets`: set of templates for Kubernetes secrets that need to be defined in order to make the CI work; the actual values can be added to an `actual` subdirectory 

The primary reason why everything is in its own repository is to make reacting to GitHub webhooks extremely easy.

In the following, we describe the details, in order of how things need to be deployed.
We assume the already existing GitHub layout:
- `rhoar-bot`: bot account that Jenkins will use to connect to GitHub; it needs to have owner access to all booster repositories, so that it can mark pull requests passed or failed
- `rhoar-ci`: organization that contains all these repositories that constitute the RHOAR boosters CI infrastructure

As a development environment, we assume Minishift.
For production, we assume a regular OKD or OCP cluster.
We call this production cluster the "CI cluster", as opposed to any other external cluster on which boosters can also be tested.

### Minishift

In development, we always want to login to the OpenShift registry running in the Minishift VM:

```bash
eval $(minishift docker-env)
docker login -u $(oc whoami) -p $(oc whoami -t) $(minishift openshift registry)
```

## Namespaces

In development environment, a single namespace (a "project" in OpenShift parlance) is enough.

In production, 3 different namespaces are expected in the CI cluster:

- `rhoar-ci`: the "main" namespace in which the Jenkins master and dashboard run, and where the Jenkins agents are being provisioned
- `rhoar-ci-work-master`: a dedicated workspace for running tests when someone pushes a commit to the `master` branch of a booster repository
- `rhoar-ci-work-pr`: a dedicated workspace for running tests when someone submits a pull request to a booster repository

In addition to these 3 namespaces in the CI cluster, one namespace needs to be present on each external cluster that boosters should be tested on (such as OpenShift Online Starter).

Everything that needs to be deployed to OpenShift below is expected to be created in the "main" namespace of the CI cluster, unless explained otherwise.

## Secrets

Before we try to deploy anything else, secrets need to be deployed.
All the YAML files in the `secrets` directory have comments, so before continuing, go read them.

### In production

In production, we need to copy all the `*.yml` files into `secrets/actual` and fill in the actual values.
Specifically for `clusters.yml`, for each token, log into the target cluster, create or switch to the appropriate namespace and run this sequence of commands:

```bash
oc create serviceaccount rhoar-ci
oc policy add-role-to-user admin -z rhoar-ci
SECRET=$(oc get serviceaccount rhoar-ci -o json | jq -r '.secrets[] | select(.name | test("rhoar-ci-token")) | .name')
echo $(oc get secret $SECRET -o jsonpath='{.data.token}' | base64 -d)
```

The token is printed in the console, so copy it into the appropriate place in `clusters.yml`.
When you're done, run:

```bash
for S in $(ls secrets/actual/*.yml) ; do oc create -f $S ; done
```

### In development

In development, it's enough to deploy the "fake" secrets:

```bash
for S in $(ls secrets/*.yml) ; do oc create -f $S ; done
```

## Jenkins, part 1: preparation

In addition to the Docker image of the Jenkins master, the `jenkins-master` directory also contains a few other OpenShift resources.

First, we always need to make the `jenkins` service account an OpenShift project admin:

```bash
oc create -f jenkins-master/openshift/jenkins-sa-admin.yml
```

### In production

In production, we let OpenShift build the Jenkins-related images automatically.
There's a template that creates an ImageStream and a BuildConfig for all the Jenkins-related images.
We just create the template, instantiate it for each image, and then start the builds.

```bash
oc create -f jenkins-master/openshift/image-template.yml

oc process jenkins-image-template NAME=jenkins-agent-jjb    REPO_NAME=jenkins-agent-jjb    | oc apply -f -
oc process jenkins-image-template NAME=jenkins-agent-maven  REPO_NAME=jenkins-agent-maven  | oc apply -f -
oc process jenkins-image-template NAME=jenkins-agent-nodejs REPO_NAME=jenkins-agent-nodejs | oc apply -f -
oc process jenkins-image-template NAME=jenkins              REPO_NAME=jenkins-master       | oc apply -f -

oc start-build jenkins-agent-jjb
oc start-build jenkins-agent-maven
oc start-build jenkins-agent-nodejs
oc start-build jenkins
```

The `NAME` parameter is used as a name of the resulting ImageStream and BuildConfig, while the `REPO_NAME` parameter is a repository name in the `rhoar-ci` GitHub organization.

All the created build configs (`jenkins` and `jenkins-agent-*`) are also configured for receiving webhooks from GitHub.
That way, if a commit is pushed to one of these Jenkins-related repositories, a new image is built automatically.
It's a good idea to create the webhooks in GitHub, using the secrets that were automatically generated by OpenShift.

### In development

In development, it's possible to also use build configs as above, but here, we describe how to build the images manually.

Note that the `docker build` and `docker push` commands, as written below, are also enough to rebuild and redeploy during development.

#### `jenkins-agent-jjb`

```bash
cd jenkins-agent-jjb
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins-agent-jjb:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins-agent-jjb:latest
```

#### `jenkins-agent-maven`

```bash
cd jenkins-agent-maven
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins-agent-maven:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins-agent-maven:latest
```

#### `jenkins-agent-nodejs`

```bash
cd jenkins-agent-nodejs
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins-agent-nodejs:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins-agent-nodejs:latest
```

#### `jenkins-master`

```bash
cd jenkins-master
docker build -t $(minishift openshift registry)/$(oc project -q)/jenkins:latest .
docker push $(minishift openshift registry)/$(oc project -q)/jenkins:latest
```

## Jenkins, part 2: deployment and usage

Now that the Jenkins master image and also images of all the Jenkins agents were successfully built, it's time to deploy the Jenkins master.

```bash
oc new-app --template=jenkins-persistent --param NAMESPACE=$(oc project -q) --param JENKINS_IMAGE_STREAM_TAG=jenkins:latest
oc patch dc/jenkins --patch "$(cat openshift/patch.yml)"
```

What the patch in `openshift/patch.yml` does is that it mounts the `github-secret` secret, as defined in `secrets/github.yml`, into the Jenkins master filesystem.
This secret contains an access token for the `rhoar-bot` GitHub account.
This token is then imported to the Jenkins credential store, as we'll describe below.

Now that the Jenkins server is running, you can open its web interface.
Note that to login, you will use the standard OpenShift authentication flow.
There are no special user accounts.
After logging in, you will see that there's just a single job, called `zygote` (we'll explain below how it got there).
Run the job to populate the Jenkins server with all the jobs that should be there.

As the jobs are created, webhooks in the booster repositories in GitHub will be created automatically.
These webhooks will make sure that jobs are triggered whenever a commit is pushed to a `master` branch or a pull request is opened in the booster repository.
For more details about the jobs, see the subsequent chapter _Jenkins jobs_.

## Jenkins, part 3: implementation details

There's a couple of important implementation details that are not visible directly, but are vital to making the entire Jenkins setup work.
Let's take a look at them, in the `Dockerfile` order.

### Plugins

First, the `Dockerfile` copies `plugins.txt` into the container and invokes `install-plugins.sh`.
This is how most of the plugins get installed.
The `plugins.txt` file has a comment for each plugin installed, so refer to that.

### Plugin overrides

Second, the `Dockerfile` copies the `plugins` directory into the container, overriding a few Jenkins plugins that were previously installed.
The plugins are first installed through `plugins.txt` to make sure all their dependencies are installed as well.
Only after that are the overrides copied into the container.

So far, there is only one reason why we have to override these plugins: to make sure that GitHub webhooks are created with disabled SSL verification.
The cluster we run Jenkins on uses a self-signed certificate.
Work is in progress to make this configurable in the plugins themselves.
More information about the overrides is in the `plugins` directory in the `README.md`.

### Configuration

Third (and last), the `Dockerfile` copies the `scripts` directory into the container, together with all the customizations.
There are two kinds of customizations: initialization scripts and user content.

#### Initialization scripts

We use the [post-initialization scripts](https://wiki.jenkins.io/display/JENKINS/Post-initialization+script) mechanism to alter Jenkins server configuration in an automated way.
This is crucial to make sure redeployment can be fully automated.
The Jenkins server is not expected to be configured manually, through the UI.
All configuration changes are performed by the `init.groovy.d` scripts.

- `01-root-url.groovy`: set the root URL of the running Jenkins server in Jenkins configuration, because some plugins need to read it early
- `02-job-cleanup.groovy`: delete the sample job that is created by default in the OpenShift image of the Jenkins server
- `03-markup-formatter.groovy`: set a markup formatter that allows actually rendering (sanitized) HTML in job descriptions, build descriptions etc.
- `04-theme.groovy`: set the Jenkins theme to a custom CSS (see _User content_ below)
- `10-credentials-github-token.groovy`: add the `rhoar-bot` GitHub token, which is mounted on the filesystem from the OpenShift secret, to the Jenkins credentials store, so that Jenkins jobs and plugins can use it to connect to GitHub
- `20-github.groovy`: configure the GitHub Jenkins plugin to use the `rhoar-bot` GitHub token
- `21-ghprb.groovy`: configure the GitHub Pull Request Builder Jenkins plugin to use the `rhoar-bot` GitHub token
- `30-pod-templates.groovy`: configure pod templates in the Kubernetes Jenkins plugin; this plugin is responsible for spawning new Jenkins agents as pods in OpenShift
- `99-zygote-job.groovy`: create the `zygote` job using the Jenkins Java API

##### Pod templates

Pod templates configure how Jenkins agents are being created and matched to Jenkins jobs.
Here's how it works in detail:

1. In Jenkins, a _node_ is a machine on which an agent runs.
   In our case, we use the Jenkins Kubernetes plugin, which creates and disposes pods as necessary, inside the CI cluster.
   Each of these pods serves as a Jenkins node.
2. Jenkins nodes can have many labels.
   Nodes created by the Kubernetes plugin have the labels as specified in the pod template.
   For example, the `maven` pod template specifies the `maven` label.
3. In Jenkins, each job can be configured to only run on nodes that match a specific label expression (e.g. "run on node that has this label, but doesn't have that label").
   In our case, all jobs use very simple label expression: just the desired label.
   For example, the Java-based jobs use a `maven` label.
4. When a job is started, Jenkins tries to run it on a node that has a matching set of labels.
   If no such node is available, the job simply stays in the queue.
   The Kubernetes plugin watches the queue and creates new pods/nodes as necessary. 

The pod templates are also configured in a way that after a job finishes, the pod remains alive for some amount of time.
In case a new job is started during that time, it's not required to create a new pod, as the existing one can be reused.

#### User content

Finally, we add a CSS file with a custom Jenkins web UI theme.
With this CSS file, Jenkins actually looks much better.
See the `README.md` for information about where the CSS is downloaded from. 

## Jenkins agents

In addition to installing all the required dependencies, all the Jenkins agent images contain some customizations.
These are described here.

### `jenkins-agent-jjb`

This image inherits from the base OpenShift image for Jenkins.
On top of it, we install PIP, the Python packaging system, and using it, we install Jenkins Job Builder.
We force a specific JJB version, because we then apply a few textual patches directly to some `.py` files.
The patches are:

- `jjb-git-scm.patch`: Jenkins has an `honor-refspec` option for the Git SCM, which JJB doesn't expose; this patch adds support for it
- `jenkins-token-auth.patch`: by default, HTTP Basic auth is used when connecting to the Jenkins HTTP API; this patch adds a rudimentary support for using a Bearer token 

Finally, we add a few files:
 
- `/etc/jenkins_jobs/jenkins_jobs.ini`: JJB configuration file with some common settings; most importantly, it sets `http://jenkins/` as the default Jenkins server URL, which makes sense inside the CI cluster
- `/usr/local/bin/jjb`: script to invoke JJB from inside the CI cluster; simply calls the standard `jenkins-jobs` script with token-based authentication

Together, these two files make JJB invocation in the infrastructural jobs really easy.
The `jenkins_jobs.ini` file sets the Jenkins server URL, and the `jjb` script takes care of authentication.
The infrastructural jobs don't have to deal with these concerns.

### `jenkins-agent-maven`

This image inherits from the Maven-specific OpenShift image for Jenkins.
On top of it, it just installs a few useful packages such as `jq` or `xmlstarlet`.

It also installs [`dumb-init`](https://github.com/Yelp/dumb-init), a minimalistic init system for use in containers, and sets it as an image entrypoint.
By using an actual init system, we make sure that signals are handled properly and that zombie processes are reaped.
This is to ensure a clean environment between multiple jobs running in the same pod.

### `jenkins-agent-nodejs`

This image inherits from the Node.js-specific OpenShift image for Jenkins.
On top of it, it just installs a few useful packages such as `jq` or `xmlstarlet`.
`dumb-init` is also installed, just like in case of the Maven image above.

Then, most importantly, the Google Chrome browser is installed.
This is because the end-to-end tests for boosters, which are written in TypeScript, require a headless web browser (specifically Chrome, but they could be switched to Firefox relatively easily).
In addition to Chrome, the [Microsoft core fonts](http://corefonts.sourceforge.net/) are installed, so that screenshots from the tests have properly rendered text.

## Jenkins jobs

There is one Jenkins job for each booster and target cluster and test variant:

- for each booster:
  - on the CI cluster:
    - after a commit has been pushed to `master`:
      - 1 job for testing the Fabric8 Maven plugin (FMP) flow
      - 1 job for testing the Source to Image (S2I) flow
      - 1 job for the end-to-end (E2E) tests
    - after a pull request has been submitted:
      - 1 job for testing the FMP flow
  - on the other clusters:
      - 1 job for testing the FMP flow after a commit has been pushed to `master`

This structure allows creating extra reports outside of Jenkins (see the _Dashboard_ chapter below).
In addition to these jobs, there's a couple of "infra" jobs that help maintaining the CI infrastructure.

Jobs are defined and maintained using [Jenkins Job Builder](http://jenkins-job-builder.readthedocs.io/en/stable/).
More specifically, macros and templates are used heavily.

### Building blocks

First, let's describe some of the important building blocks.
All the macros are defined in `jobs/macros.yml`, and they often include shell scripts located in `jobs/scripts`.

#### Setting up the OpenShift client binary

One of the first things that jobs interacting with OpenShift do is setting up the OpenShift client binary (`oc`).
For that, a `setup-oc` macro is provided.
What this macro does is executing a shell script, the source of which is located in `jobs/scripts/setup-oc.sh`, and makes sure the `oc` binary is in the `PATH`.

What the shell script does is:

1. Connect to the API server of the target OpenShift cluster and read the OpenShift version.
2. Download corresponding version of the client binary archive from GitHub.
3. Unpack the archive into an extra directory.
4. Call `oc login`, using the corresponding API token.

Note that the script needs to be updated with each OpenShift major release!

#### GitHub Pull Request Builder plugin

Testing pull requests is done using the GitHub Pull Request Builder plugin (GHPRB).
The `github-pr` macro is provided to set up the build trigger in the Jenkins job.
It configures all the necessary details, including a whitelist of GitHub users.
For these users, the job will be started without approval.
Otherwise, approval is necessary, because pull requests can contain arbitrary changes, including malicious code.
Members of the GitHub organization in which the repository is present are also automatically whitelisted.

#### Scripts for running FMP flow tests

The `fmp-booster-basic` and `fmp-booster-secured` macros contain the main body of the FMP flow tests.
They basically just run `mvn clean verify` with the correct arguments (such as `-Popenshift,openshift-it`).
There is nothing interesting about the FMP flow tests, they are the easiest to run.

One important thing about the underlying tests in the boosters is that they are written using Arquillian Cube.
In the FMP flow tests, we let Arquillian Cube deploy and undeploy the application automatically -- this works because Arquillian Cube knows how to work with the FMP outcomes.
However, Arquillian Cube can also be instructed to _not_ deploy/undeploy anything, just run the test.
We'll use that in the S2I tests.

#### Scripts for running S2I flow tests

The `s2i-booster-basic` macro contains the body of the S2I flow tests.
The jobs run the same tests as the FMP flow jobs, that is, the integration tests inside the boosters themselves.
The main difference is that in the S2I jobs, we explicitly configure Arquillian Cube to _not_ deploy/undeploy the application, using the `env.init.enabled` configuration property.

Instead, we'll deploy/undeploy the application manually, using the YAML files each booster has in the `.openshiftio` directory.
We actually closely emulate what Fabric8 Launcher does.
This emulation code is in the `jobs/scripts/s2i-functions.sh` script.
The script is actually meant to be used as a "library", with the "public API" being the `s2i_setup` and `s2i_teardown` functions.
All other functions in that script are a "private" implementation detail, though this isn't really enforced in any way.

#### Scripts for running E2E tests

The `e2e-booster-basic` macro contains the body of the E2E tests.
These tests expect the tested applications to already be deployed -- for that, the same mechanism as in the S2I flow tests is used.
When the tested application is deployed, `npm` is used to start the TypeScript tests.
The tests will launch a headless browser automatically.
Albeit the E2E tests are written in TypeScript, they produce a JUnit-style XML with results.
In addition to that, they also produce a screenshot report.

#### Avoiding collisions between jobs

The `build-block` macro provides a simple configuration for blocking potentially colliding builds.
This is to make sure that at most one job accesses a target cluster at the same time.
One exception is the CI cluster, where we have two namespaces: one for pull request jobs and the other for `master` branch merge jobs.

Build blocking is based on a naming convention: all jobs targetting the same cluster (or a namespace in the CI cluster) share the same name prefix.
The Build Blocker Jenkins plugin is used, because it allows blocking jobs based on a regular expression.
Other plugins that offer similar functionality are available, but this one is easiest to understand.

### Job definitions

All jobs targetting the CI cluster, both the PR and `master` branch namespaces, are defined in the `ci-fmp-s2i-e2e.yml` file.
The file contains 4 templates:

- `ci-pr-fmp-{job-name}`: FMP flow tests for pull request jobs
- `ci-master-fmp-{job-name}`: FMP flow tests for `master` branch changes
- `ci-master-s2i-{job-name}`: S2I flow tests for `master` branch changes
- `ci-master-e2e-{job-name}`: E2E tests for `master` branch changes

For each booster, this template is instantiated.

All jobs targetting the OpenShift Online Starter clusters are defined in the `oso-fmp.yml` file.
The file contains 1 template:

- `oso-{openshift-cluster}-fmp-{job-name}`: FMP flow tests for `master` branch changes

This template is instantiated for each booster and each target cluster.

#### Job descriptions

One particularly interesting part of the job definitions is the job description.
The description isn't a free-form text; instead, it's a JSON with 3 fields:

- `cluster`: name of the target OpenShift cluster
- `description`: name of the booster, in the _Runtime | ID: Mission_ format
- `type`: name of the tested flow (FMP, S2I, E2E)

All jobs targetting the same cluster are expected to have identical content of the `cluster` fields.
Same for `description` and `type`.
This information is used in the dashboard application for creating a tabular report.

### Infrastructure jobs

The `infra.yml` file defines a couple of jobs to help maintaining the CI infrastructure.
These are:

- `infra-jobs-pr`: triggered after a pull request was subbmitted to the `jenkins-jobs` repository, runs JJB validation
- `infra-jobs-master`: triggered after a commit was pushed to the `jenkins-jobs` repository, updates all the jobs in Jenkins
- `infra-dashboard-pr`: triggered after a pull request was submitted to the `dashboard` repository, runs Maven verification build
- `infra-dashboard-master`: triggered after a commit was pushed to the `dashboard` repository, deploys the dashboard using Fabric8 Maven plugin
- `infra-e2e-tests-pr`: triggered after a pull request was submitted to the boosters E2E tests repository, runs the in-project checks 

### Views

When looking at the Jenkins user interface, which is sometimes necessary, it's good to have the jobs categorized into views.
JJB actually allows to define one kind of views -- job lists, based on a regular expression match against job name.
This is what `views.yml` does.

## Dashboard

The dashboard is a simple web application written with WildFly Swarm.
It runs in the same namespace as the Jenkins server and connects to it using the Jenkins HTTP API.
It reads the current status of the test jobs and presents it in a more accessible way, using colored tabular format.
It also allows rerunning the test jobs, in case of intermittent failures.

One interesting thing is how Jenkins HTTP API authentication and authorization is handled.
We've already mentioned that it's possible to log into Jenkins using the OpenShift auth flow.
This also applies to the HTTP API.
Each service account token can be used as a bearer token for authentication with the Jenkins HTTP API.

### In development

The dashboard application by default expects to be running in a production environment, that is, in an OpenShift namespace, alongside the Jenkins server.
However, when working on the dashboard itself, it's pretty easy to run it locally and let it connect to an existing Jenkins server in an existing OpenShift cluster (including Minishift).
To do that, just edit `project-local.yml`:

- `dashboard.jenkins.url`: OpenShift route to the Jenkins server
- `dashboard.openshift.url`: URL of the OpenShift API server
- `dashboard.openshift.token`: OpenShift API token, with permissions to the namespace in which Jenkins is running

After setting the correct values in `project-local.yml`, just rebuild the application and run the uberjar in the `local` profile:

```bash
mvn clean package
java -jar target/dashboard-swarm.jar -S local
```
