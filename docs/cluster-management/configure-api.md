---
layout: main
title: Configuring the API
category: Cluster Management
menu: menu
toc:
    - title: Managing the API
      url: "#managing-the-api"
      active: true
    - title: Packages
      url: "#packages"
    - title: Configuration
      url: "#configuration"
    - title: Authentication / Authorization
      url: "#authentication--authorization"
      subitem: true
    - title: Build Variables
      url: "#build-variables"
      subitem: true
    - title: Bookend Plugins
      url: "#bookend-plugins"
      subitem: true
    - title: Serving
      url: "#serving"
      subitem: true
    - title: Ecosystem
      url: "#ecosystem"
      subitem: true
    - title: Data Store
      url: "#datastore-plugin"
      subitem: true
    - title: Executors
      url: "#executor-plugin"
      subitem: true
    - title: Notifications
      url: "#notifications-plugin"
      subitem: true
    - title: Source Control Management (SCM)
      url: "#source-control-plugin"
      subitem: true
    - title: Webhooks
      url: "#webhooks"
      subitem: true
    - title: Rate Limiting
      url: "#rate-limiting"
      subitem: true
    - title: Canary Routing
      url: "#canary-routing"
      subitem: true
    - title: Configure Redis Lock
      url: "#configure-redis-lock"
      subitem: true
    - title: Logging
      url: "#logging"
      subitem: true
    - title: Extending the Docker container
      url: "#extending-the-docker-container"
---
# Managing the API

## Packages

Like the other services, the API is shipped as a [Docker image](https://hub.docker.com/r/screwdrivercd/screwdriver/) with port 8080 exposed.

```bash
$ docker run -d -p 9000:8080 screwdrivercd/screwdriver:stable
$ open http://localhost:9000
```

Our images are tagged for their version (eg. `1.2.3`) as well as a floating `latest` and `stable`.  Most installations should be using `stable` or the fixed version tags.

## Configuration
Screwdriver already [defaults most configuration](https://github.com/screwdriver-cd/screwdriver/blob/master/config/default.yaml), but you can override defaults using a `config/local.yaml` or environment variables. All the possible environment variables are [defined here](https://github.com/screwdriver-cd/screwdriver/blob/master/config/custom-environment-variables.yaml).

### Authentication / Authorization

Configure how users can and who can access the API.

| Key                    | Required | Description                                                                                                               |
|:-----------------------|:---------|:--------------------------------------------------------------------------------------------------------------------------|
| JWT_ENVIRONMENT | No      | Environment to generate the JWT for. Ex: `prod`, `beta`. If you want the JWT to not contain `environment`, don't set this environment variable (do not set it to `''`). |
| SECRET_JWT_PRIVATE_KEY | Yes      | A private key uses for signing jwt tokens. Generate one by running `$ openssl genrsa -out jwt.pem 2048`                   |
| SECRET_JWT_PUBLIC_KEY  | Yes      | The public key used for verifying the signature. Generate one by running `$ openssl rsa -in jwt.pem -pubout -out jwt.pub` |
| SECRET_JWT_QUEUE_SVC_PUBLIC_KEY  | Yes      | The public key used for verifying the signature when plugin is queue. Generate one by running `$ openssl rsa -in jwtqs.pem -pubout -out jwtqs.pub` |
| SECRET_COOKIE_PASSWORD | Yes      | A password used for encrypting session data. **Needs to be minimum 32 characters**                                        |
| SECRET_PASSWORD        | Yes      | A password used for encrypting stored secrets. **Needs to be minimum 32 characters**                                      |
| IS_HTTPS               | No       | A flag to set if the server is running over https. Used as a flag for the OAuth flow (default to `false`)                 |
| SECRET_WHITELIST       | No       | Whitelist of users able to authenticate against the system. If empty, it allows everyone. (JSON Array format)             |
| SECRET_ADMINS          | No       | List of admins with elevated access to the cluster. If empty, it allows everyone. (JSON Array format)             |

```yaml
# config/local.yaml
auth:
    jwtPrivateKey: |
        PRIVATE KEY HERE
    jwtPublicKey: |
        PUBLIC KEY HERE
    jwtQueueServicePublicKey: |
        QUEUE SVC PUBLIC KEY HERE
    cookiePassword: 975452d6554228b581bf34197bcb4e0a08622e24
    encryptionPassword: 5c6d9edc3a951cda763f650235cfc41a3fc23fe8
    https: false
    whitelist:
        - github:batman
        - github:robin
    admins:
        - github:batman
```

### Multibuild Cluster

By default, the [build cluster feature](configure-buildcluster-queue-worker) is turned off.

| Key | Default| Description |
|:----|:-------|:------------|
| MULTI_BUILD_CLUSTER_ENABLED | `false` | Whether build cluster is on or off. Options: `true` or `false` |

```yaml
# config/local.yaml
multiBuildCluster:
    enabled: true
```

### Build Variables

#### Environment
You can preset default environment variables for all builds in your cluster. By default, this field is `{ SD_VERSION: 4 }`.

| Key | Default| Description |
|:----|:-------|:------------|
| CLUSTER_ENVIRONMENT_VARIABLES | `{ SD_VERSION: 4 }` | Default environment variables for build. For example: `{ SD_VERSION: 4, SCM_CLONE_TYPE: "ssh" }` |

#### Remote join
By default, [the remote join feature](../user-guide/configuration/workflow#remote-join) is turned off.

| Key | Default| Description |
|:----|:-------|:------------|
| EXTERNAL_JOIN | `false` | Whether remote join feature is on or not. Options: `true` or `false` |

```yaml
# config/local.yaml
build:
    environment: CLUSTER-ENV-IN-JSON-FORMAT
    externalJoin: true
```

### Bookend Plugins

You can globally configure which built-in bookend plugins will be used during a build. Bookend plugins can be configured for each cluster.
By default, `scm` is enabled to begin builds with a SCM checkout command.

If you're looking to include a custom bookend in the API, please refer [here](#extending-the-docker-container).

| Key | Default| Description |
|:----|:-------|:------------|
| BOOKENDS | None | The associative array of bookends to be executed at the beginning and end of every build. Take the forms of `{"default": {"setup": ["scm", ...], "teardown": [...]}, "clusterA": {"setup": ["scm", ...], "teardown": [...]}}` |


```yaml
# config/local.yaml
bookends:
  default: 
    setup:
      - scm
      - my-custom-bookend
    teardown:
      - screwdriver-artifact-bookend
      - screwdriver-cache-bookend
  clusterA:
    setup: ...
    teardown: ...
  clusterB:
    setup: ...
    teardown: ...
```
For `clusterA` and `clusterB`, specify the build cluster provided by the cluster administrator. please refer [here](configure-buildcluster-queue-worker).  
If a build is assigned to a build cluster other than the options available in `bookends` configuration, then the bookend plugin set in `default` will be used.

#### Coverage bookends

We currently support [SonarQube](https://github.com/screwdriver-cd/coverage-sonar) for coverage bookends.

##### Sonar

In order to use Sonar in your cluster, set up a Sonar server (see example at [our sonar pipeline](https://github.com/screwdriver-cd/sonar-pipeline)). Then configure the following environment variables:

| Key             | Required | Description           |
|:----------------|:---------|:----------------------|
| COVERAGE_PLUGIN | Yes      | Should be `sonar`     |
| COVERAGE_PLUGIN_DEFAULT_ENABLED | No | Whether coverage-bookend will execute coverage scanning as default or not; default `true` |
| URI             | Yes      | Screwdriver API url   |
| ECOSYSTEM_UI    | Yes      | Screwdriver UI url    |
| COVERAGE_SONAR_HOST | Yes  | Sonar host URL        |
| COVERAGE_SONAR_ADMIN_TOKEN | Yes | Sonar admin token |
| COVERAGE_SONAR_ENTERPRISE | No | Whether using Enterprise(true) or open source edition of SonarQube(false); default `false` |
| COVERAGE_SONAR_GIT_APP_NAME | No | Github app name for Sonar pull request decoration; default `Screwdriver Sonar PR Checks`; This feature requires Sonar enterprise edition. Follow [instructions in the Sonar docs](https://docs.sonarqube.org/latest/analyzing-source-code/pull-request-analysis) for details. |

You’ll also need to add the `screwdriver-coverage-bookend` along with the `screwdriver-artifact-bookend` as teardown bookends by setting the `BOOKENDS` variable (in JSON format). See the Bookend Plugins section above for more details. Using Enterprise edition of SonarQube will default to _pipeline_ scope for SonarQube project keys and names. Will also allow for usage of PR analysis and prevent creation of separate projects for each Screwdriver job. Using non-Enterprise SonarQube will default to _job_ scope for SonarQube project keys and names.

### Serving

Configure the how the service is listening for traffic.

| Key       | Default             | Description                                                                                                                                                                                                |
|:----------|:--------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| PORT      | 80                  | Port to listen on                                                                                                                                                                                          |
| HOST      | 0.0.0.0             | Host to listen on (set to localhost to only accept connections from this machine)                                                                                                                          |
| URI       | http://localhost:80 | Externally routable URI (usually your load balancer or CNAME)                                                                                                                                              |
| HTTPD_TLS | false               | SSL support; for SSL, replace `false` with a JSON object that provides the options required by [`tls.createServer`](https://nodejs.org/api/tls.html#tls_tls_createserver_options_secureconnectionlistener) |

```yaml
# config/local.yaml
httpd:
    port: 443
    host: 0.0.0.0
    uri: https://localhost
    tls:
        key: |
            PRIVATE KEY HERE
        cert: |
            YOUR CERT HERE
```

### Ecosystem

Specify externally routable URLs for your UI and Artifact Store service.

| Key              | Default                                                     | Description                                  |
|:-----------------|:------------------------------------------------------------|:---------------------------------------------|
| ECOSYSTEM_UI     | https://cd.screwdriver.cd                                   | URL for the User Interface                   |
| ECOSYSTEM_STORE  | https://store.screwdriver.cd                                | URL for the Artifact Store                   |
| ECOSYSTEM_QUEUE  | http://sdqueuesvc.screwdriver.svc.cluster.local             | Internal URL for the Queue Service to be used with queue plugin |

```yaml
# config/local.yaml
ecosystem:
    # Externally routable URL for the User Interface
    ui: https://cd.screwdriver.cd
    # Externally routable URL for the Artifact Store
    store: https://store.screwdriver.cd
    # Internally routable FQDNS of the queue svc
    queue: http://sdqueuesvc.screwdriver.svc.cluster.local
```

### Datastore Plugin

To use Postgres, MySQL, and Sqlite, use [sequelize](https://github.com/screwdriver-cd/datastore-sequelize) plugin.

#### Sequelize
Set these environment variables:

| Environment name             | Required       | Default Value | Description                                      |
|:-----------------------------|:---------------|:--------------|:-------------------------------------------------|
| DATASTORE_PLUGIN             | Yes            |               | Set to `sequelize`                               |
| DATASTORE_SEQUELIZE_DIALECT  | No             | mysql         | Can be `sqlite`, `postgres`, `mysql`, or `mssql` |
| DATASTORE_SEQUELIZE_DATABASE | No             | screwdriver   | Database name                                    |
| DATASTORE_SEQUELIZE_USERNAME | No for sqlite  |               | Login username                                   |
| DATASTORE_SEQUELIZE_PASSWORD | No for sqlite  |               | Login password                                   |
| DATASTORE_SEQUELIZE_STORAGE  | Yes for sqlite |               | Storage location for sqlite                      |
| DATASTORE_SEQUELIZE_HOST     | No             |               | Network host                                     |
| DATASTORE_SEQUELIZE_PORT     | No             |               | Network port                                     |
| DATASTORE_SEQUELIZE_RO       | No             |               | JSON format that includes host, port, database, username, password, etc. for the readonly datastore instance - used for Metrics endpoints only |

```yaml
# config/local.yaml
datastore:
    plugin: sequelize
    sequelize:
        dialect: TYPE-OF-SERVER
        storage: STORAGE-LOCATION
        database: DATABASE-NAME
        username: DATABASE-USERNAME
        password: DATABASE-PASSWORD
        host: NETWORK-HOST
        port: NETWORK-PORT
        readOnly: DATASTORE-READONLY-INSTANCE
```

### Executor Plugin

We currently support [kubernetes](https://github.com/screwdriver-cd/executor-k8s),  [docker](http://github.com/screwdriver-cd/executor-docker), [VMs in Kubernetes](https://github.com/screwdriver-cd/executor-k8s-vm), [nomad](http://github.com/lgfausak/executor-nomad), [Jenkins](https://github.com/screwdriver-cd/executor-jenkins), and [queue](https://github.com/screwdriver-cd/executor-queue) executors. See the [custom-environment-variables file](https://github.com/screwdriver-cd/screwdriver/blob/master/config/custom-environment-variables.yaml) for more details.

#### Kubernetes (k8s)
If you use this executor, builds will run in pods in Kubernetes.

| Environment name   | Default Value | Description                                |
|:-------------------|:--------------|:-------------------------------------------|
| EXECUTOR_PLUGIN    | k8s           | Default executor (eg: `k8s`, `docker`, `k8s-vm`, `nomad`, `jenkins`, or `queue`) |
| LAUNCH_VERSION     | stable        | Launcher version to use                    |
| EXECUTOR_PREFIX    |               | Prefix to append to pod names              |
| EXECUTOR_K8S_ENABLED | true        | Flag to enable Kubernetes executor         |
| K8S_HOST           | kubernetes.default | Kubernetes host                       |
| K8S_TOKEN          | Loaded from `/var/run/secrets/kubernetes.io/serviceaccount/token` by default | JWT for authenticating Kubernetes requests |
| K8S_JOBS_NAMESPACE | default       | Jobs namespace for Kubernetes jobs URL     |
| K8S_CPU_MICRO      | 0.5           | Number of CPU cores for micro              |
| K8S_CPU_LOW        | 2             | Number of CPU cores for low                |
| K8S_CPU_HIGH       | 6             | Number of CPU cores for high               |
| K8S_CPU_TURBO      | 12            | Number of CPU cores for turbo              |
| K8S_MEMORY_MICRO   | 1             | Memory in GB for micro                     |
| K8S_MEMORY_LOW     | 2             | Memory in GB for low                       |
| K8S_MEMORY_HIGH    | 12            | Memory in GB for high                      |
| K8S_MEMORY_TURBO   | 16            | Memory in GB for turbo                     |
| K8S_BUILD_TIMEOUT  | 90            | Default build timeout for all builds in this cluster (in minutes) |
| K8S_MAX_BUILD_TIMEOUT | 120        | Maximum user-configurable build timeout for all builds in this cluster (in minutes) |
| K8S_NODE_SELECTORS | `{}`          | K8s node selectors for pod scheduling (format `{ label: 'value' }`) <https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#step-one-attach-label-to-the-node> |
| K8S_PREFERRED_NODE_SELECTORS | `{}`| K8s node selectors for pod scheduling (format `{ label: 'value' }`) <https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature> |
| K8S_POD_DNS_POLICY | ClusterFirst  | DNS Policy for build pod <https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy> |
| K8S_POD_IMAGE_PULL_POLICY | Always | Build pod Image Pull policy <https://kubernetes.io/docs/concepts/containers/images/#updating-images> |
| K8S_POD_LABELS | `{ app: 'screwdriver', tier: 'builds', sdbuild: buildContainerName }` | K8s pod labels for cluster settings (eg: { network-egress: 'restricted' } to execute builds where public internet access is blocked by default) |
| K8S_IMAGE_PULL_SECRET_NAME | ''    | K8s image pull secret name (optional) <https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret> |
| DOCKER_FEATURE_ENABLED | false     | Flag to enable a Docker In Docker container in the build pod |
| K8S_RUNTIME_CLASS | ''             | Runtime class |
| TERMINATION_GRACE_PERIOD_SECONDS | 60             | Termination Grace period before build pod  |


```yaml
# config/local.yaml
executor:
    plugin: k8s
    k8s:
        options:
            kubernetes:
                host: YOUR-KUBERNETES-HOST
                token: JWT-FOR-AUTHENTICATING-KUBERNETES-REQUEST
                jobsNamespace: default
            launchVersion: stable
```

#### VMs in Kubernetes (k8s-vm)
If you use the `k8s-vm` executor, builds will run in VMs in pods in Kubernetes.

| Environment name   | Default Value | Description                                |
|:-------------------|:--------------|:-------------------------------------------|
| EXECUTOR_PLUGIN    | k8s           | Default executor (set to `k8s-vm`) |
| LAUNCH_VERSION     | stable        | Launcher version to use                    |
| EXECUTOR_PREFIX    |               | Prefix to append to pod names              |
| EXECUTOR_K8SVM_ENABLED | true      | Flag to enable Kubernetes VM executor      |
| K8S_HOST           | kubernetes.default | Kubernetes host                       |
| K8S_TOKEN          | Loaded from `/var/run/secrets/kubernetes.io/serviceaccount/token` by default | JWT for authenticating Kubernetes requests |
| K8S_JOBS_NAMESPACE | default       | Jobs namespace for Kubernetes jobs URL     |
| K8S_BASE_IMAGE     |               | Kubernetes VM base image                   |
| K8S_CPU_MICRO      | 1             | Number of CPU cores for micro              |
| K8S_CPU_LOW        | 2             | Number of CPU cores for low                |
| K8S_CPU_HIGH       | 6             | Number of CPU cores for high               |
| K8S_CPU_TURBO      | 12            | Number of CPU cores for turbo              |
| K8S_MEMORY_MICRO   | 1             | Memory in GB for micro                     |
| K8S_MEMORY_LOW     | 2             | Memory in GB for low                       |
| K8S_MEMORY_HIGH    | 12            | Memory in GB for high                      |
| K8S_MEMORY_TURBO   | 16            | Memory in GB for turbo                     |
| K8S_VM_BUILD_TIMEOUT  | 90         | Default build timeout for all builds in this cluster (in minutes) |
| K8S_VM_MAX_BUILD_TIMEOUT | 120     | Maximum user-configurable build timeout for all builds in this cluster (in minutes) |
| K8S_VM_NODE_SELECTORS | `{}`       | K8s node selectors for pod scheduling (format `{ label: 'value' }`) <https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#step-one-attach-label-to-the-node> |
| K8S_VM_PREFERRED_NODE_SELECTORS | `{}`| K8s node selectors for pod scheduling (format `{ label: 'value' }`) <https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature> |

```yaml
# config/local.yaml
executor:
    plugin: k8s-vm
    k8s-vm:
        options:
            kubernetes:
                host: YOUR-KUBERNETES-HOST
                token: JWT-FOR-AUTHENTICATING-KUBERNETES-REQUEST
            launchVersion: stable
```

#### Jenkins (jenkins)
If you use the `jenkins` executor, builds will run using Jenkins.

| Environment name       | Default Value | Description          |
|:-----------------------|:--------------|:---------------------|
| EXECUTOR_PLUGIN        | k8s           | Default executor. Set to `jenkins` |
| LAUNCH_VERSION         | stable        | Launcher version to use            |
| EXECUTOR_JENKINS_ENABLED | true        | Flag to enable Jenkins executor    |
| EXECUTOR_JENKINS_HOST  |               | Jenkins host |
| EXECUTOR_JENKINS_PORT  | 8080          | Jenkins port   |
| EXECUTOR_JENKINS_USERNAME | screwdriver | Jenkins username |
| EXECUTOR_JENKINS_PASSWORD |            | Jenkins password/token used for authenticating Jenkins requests |
| EXECUTOR_JENKINS_NODE_LABEL | screwdriver | Node labels of Jenkins slaves |
| EXECUTOR_JENKINS_DOCKER_COMPOSE_COMMAND | docker-compose | Path to the docker-compose command |
| EXECUTOR_JENKINS_DOCKER_PREFIX | `''`  | Prefix to the container |
| EXECUTOR_JENKINS_LAUNCH_VERSION | stable | Launcher container tag to use |
| EXECUTOR_JENKINS_DOCKER_MEMORY | 4g    | Memory limit (docker run `--memory` option) |
| EXECUTOR_JENKINS_DOCKER_MEMORY_LIMIT | 6g | Memory limit include swap (docker run `--memory-swap` option)
|
| EXECUTOR_JENKINS_BUILD_SCRIPT | `''`   | The command to start a build with |
| EXECUTOR_JENKINS_CLEANUP_SCRIPT | `''` | The command to clean up the build system with |
| EXECUTOR_JENKINS_CLEANUP_TIME_LIMIT | 20 | Time to destroy the job (in seconds) |
| EXECUTOR_JENKINS_CLEANUP_WATCH_INTERVAL | 2 | Intercal to detect the stopped job (in seconds)

```yaml
# config/local.yaml
executor:
    plugin: jenkins
    jenkins:
        options:
            jenkins:
                host: jenkins.default
                port: 8080
                username: screwdriver
                password: YOUR-PASSWORD
            launchVersion: stable
```

#### Docker (docker)
Use the `docker` executor to run in Docker. [sd-in-a-box](./running-locally) also runs using Docker.

| Environment name       | Default Value | Description          |
|:-----------------------|:--------------|:---------------------|
| EXECUTOR_PLUGIN        | k8s           | Default executor. Set to `docker` |
| LAUNCH_VERSION         | stable        | Launcher version to use           |
| EXECUTOR_DOCKER_ENABLED | true         | Flag to enable Docker executor    |
| EXECUTOR_DOCKER_DOCKER | `{}`          | [Dockerode configuration](https://www.npmjs.com/package/dockerode#getting-started) (JSON object) |
| EXECUTOR_PREFIX    |               | Prefix to append to pod names              |

```yaml
# config/local.yaml
executor:
    plugin: docker
    docker:
        options:
            docker:
                socketPath: /var/lib/docker.sock
            launchVersion: stable
```

#### Queue (queue)
Using the `queue` executor will allow builds to be queued to a [remote queue service](./configure-queue-service) running a Redis instance containing Resque.

| Environment name       | Default Value | Description          |
|:-----------------------|:--------------|:---------------------|
| EXECUTOR_PLUGIN        | k8s           | Default executor. Set to `queue` |

```yaml
# config/local.yaml
executor:
    plugin: queue
    queue: ''
```

#### Nomad (nomad)
Set these environment variables:

| Environment name       | Default Value | Description                                 |
|:-----------------------|:--------------|:--------------------------------------------|
| EXECUTOR_PLUGIN        | nomad         | Nomad executor                              |
| LAUNCH_VERSION         | latest        | Launcher version to use                     |
| EXECUTOR_NOMAD_ENABLED | true          | Flag to enable Nomad executor               |
| NOMAD_HOST             | nomad.default | Nomad host (e.g. http://192.168.30.30:4646) |
| NOMAD_CPU              | 600           | Nomad cpu resource in Mhz                   |
| NOMAD_MEMORY           | 4096          | Nomad memory resource in MB                 |
| EXECUTOR_PREFIX        | sd-build-     | Nomad job name prefix                       |

```yaml
# config/local.yaml
executor:
    plugin: nomad
    nomad:
        options:
            nomad:
                host: http://192.168.30.30:4646
            resources:
                cpu:
                    high: 600
                memory:
                    high: 4096
            launchVersion:  latest
            prefix:  'sd-build-'
```

### Notifications Plugin
We currently support [Email notifications](https://github.com/screwdriver-cd/notifications-email) and [Slack notifications](https://github.com/screwdriver-cd/notifications-slack).

Set these environment variables:

| Environment name   | Required | Default Value | Description                   |
|:-------------------|:---------|:--------------|:------------------------------|
| NOTIFICATIONS      | No      | {}             | JSON object with notifications settings |

#### Email Notifications

Configure the SMTP server and sender address that email notifications will be sent from.

```yaml
# config/local.yaml
notifications:
    email:
        username: your-username # optional SMTP username
        password: your-password # optional SMTP password
        host: smtp.yourhost.com
        port: 25
        from: example@email.com
```

Configurable authentication settings have not yet been built, but can easily be added. We’re using the [nodemailer](https://nodemailer.com/about/) package to power emails, so authentication features will be similar to any typical nodemailer setup. Contribute at: [screwdriver-cd/notifications-email](https://github.com/screwdriver-cd/notifications-email)

#### Slack Notifications

Create a `screwdriver-bot` [Slack bot user](https://api.slack.com/bot-users) in your Slack instance. Generate a Slack token for the bot and set the `token` field with it in your Slack notifications settings as below.

```yaml
# config/local.yaml
notifications:
    slack:
        defaultWorkspace: 'your-workspace'
        workspaces:
            your-workspace:
                token: 'YOUR-SLACK-BOT-TOKEN-HERE'
            another-workspace:
                token: 'ANOTHER-SLACK-BOT-TOKEN-HERE'
```

#### Custom Notifications

You can create custom notification packages by extending [notifications-base](https://github.com/screwdriver-cd/notifications-base).
The format of the package name must be `screwdriver-notifications-<your-notification>`.

The following is an example snippet of `local.yaml` configuration when you use email notification and your custom notification:

```yaml
# config/local.yaml
notifications:
    email:
        host: smtp.yourhost.com
        port: 25
        from: example@email.com
    your-notification:
        foo: bar
        abc: 123
```

If you want to use [scoped package](https://docs.npmjs.com/misc/scope), the configuration is as below:

```yaml
# config/local.yaml
notifications:
    your-notification:
        config:
            foo: bar
            abc: 123
        scopedPackage: '@scope/screwdriver-notifications-your-notification'
```

#### Notifications Options
Any overarching notifications options will go in this section.

```yaml
#config/local.yaml
notifications:
    options:
        throwValidationErr: false # default true; boolean to throw error when validation fails or not
    slack:
        defaultWorkspace: 'your-workspace'
        workspaces:
            your-workspace:
                token: 'YOUR-SLACK-BOT-TOKEN-HERE'
            secondary-workspace:
                token: 'ANOTHER-SLACK-BOT-TOKEN-HERE'
    email:
        host: smtp.yourhost.com
        port: 25
        from: example@email.com

```

### Source Control Plugin

We currently support [GitHub and GitHub Enterprise](https://github.com/screwdriver-cd/scm-github), [Bitbucket.org](https://github.com/screwdriver-cd/scm-bitbucket), and [GitLab](https://github.com/screwdriver-cd/scm-gitlab). Check the [SCM chart](../user-guide/scm) for a breakdown of feature support.

#### Step 1: Set up your OAuth Application
You will need to set up an OAuth Application and retrieve your OAuth Client ID and Secret.

##### GitHub:
1. Navigate to the [GitHub OAuth applications](https://github.com/settings/developers) page.
2. Click on the application you created to get your OAuth Client ID and Secret.
3. Fill out the `Homepage URL` and `Authorization callback URL` to be the IP address of where your API is running.

##### GitLab:
1. Navigate to the [GitLab applications](https://gitlab.com/-/profile/applications) page.
2. Fill out the form with `Redirect URI` as `https://YOUR_IP/v4/auth/login/gitlab:gitlab.com/web`.
3. Click `Save Application`.

##### Bitbucket.org:
1. Navigate to the Bitbucket OAuth applications: [https://bitbucket.org/account/user/{your-username}/api](https://bitbucket.org/account/user/{your-username}/api)
2. Click on `Add Consumer`.
3. Fill out the `URL` and `Callback URL` to be the IP address of where your API is running.

#### Step 2: Configure your SCM plugin
Set these environment variables:

| Environment name   | Required | Default Value | Description                   |
|:-------------------|:---------|:--------------|:------------------------------|
| SCM_SETTINGS       | Yes      | {}            | JSON object with SCM settings |

##### GitHub example
```yaml
# config/local.yaml
scms:
    github:
        plugin: github
        config:
            oauthClientId: YOU-PROBABLY-WANT-SOMETHING-HERE # The client id used for OAuth with GitHub. GitHub OAuth (https://developer.github.com/v3/oauth/)
            oauthClientSecret: AGAIN-SOMETHING-HERE-IS-USEFUL # The client secret used for OAuth with GitHub
            secret: SUPER-SECRET-SIGNING-THING # Secret to add to GitHub webhooks so that we can validate them
            gheHost: github.screwdriver.cd # [Optional] GitHub enterprise host
            username: sd-buildbot # [Optional] Username for code checkout
            email: dev-null@screwdriver.cd # [Optional] Email for code checkout
            https: false # [Optional] Is the Screwdriver API running over HTTPS, default false
            commentUserToken: A_BOT_GITHUB_PERSONAL_ACCESS_TOKEN # [Optional] Token for writing PR comments in GitHub, needs "public_repo" scope
            privateRepo: false # [Optional] Set to true to support private repo; will need read and write access to public and private repos (https://developer.github.com/v3/oauth/#scopes)
            autoDeployKeyGeneration: false # [Optional] Set to true to allow automatic generation of private and public deploy keys and add them to the build pipeline and Github for repo checkout, respectively
    ...
```

###### Private Repo
If users want to use private repo, they also need to set up `SCM_USERNAME` and `SCM_ACCESS_TOKEN` as [secrets](/user-guide/configuration/secrets) in their `screwdriver.yaml`.

###### Meta PR Comments
In order to enable [meta PR comments](/user-guide/metadata), you’ll need to create a bot user in Git with a personal access token with the `public_repo` scope. In Github, create a new user. Follow instructions to [create a personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token), set the scope as `public_repo`. Copy this token and set it as `commentUserToken` in your `scms` settings in your [API config yaml](https://github.com/screwdriver-cd/screwdriver/blob/master/config/custom-environment-variables.yaml#L296).

###### Deploy Keys
Deploy Keys are SSH keys that grant access to a single GitHub repository. This key is attached directly to the repository instead of to a personal user account, as opposed to Github personal access tokens. Github personal access tokens give user-wide access for all the repositories while deploy keys, on the other hand, give access to a single repository. Due to their limited access, deploy keys are preferred for private repositories.

If users want to use deploy keys in their pipeline they have 2 options:
* Enable automatic generation and handling of deploy keys as a part of the pipeline by setting the `autoDeployKeyGeneration` flag to `true` in their `config/local.yaml`. With this flag enabled, the user will get an option to actually trigger the generation in the UI.
* Manually generate the public and private key pair using `openssl genrsa -out jwt.pem 2048` and `openssl rsa -in jwt.pem -pubout -out jwt.pub`. Now add the public key as a deploy key to the repo. The private key needs to be **base64 encoded** and added as a secret `SD_SCM_DEPLOY_KEY` in the pipeline. Refer [secrets](/user-guide/configuration/secrets) for adding secrets.

###### Read-only SCM
Sometimes you might want to have a SCM with read-only access. Users will be able to indirectly create pipelines for an SCM by listing them as a [child pipeline](../user-guide/configuration/externalConfig). Below is an example of an SCM configuration you would add to your SCMs for a read-only one. Users cannot login to the SCM in the UI.

1. Create a headless user in your read-only SCM. Create a personal access token for the user.
2. Set up your read-only SCM config (as shown in the example below).
3. Add the new build cluster with the corresponding `scmContext` to `POST https://YOUR_IP/v4/buildclusters`. See [API](../user-guide/api) for more information. The payload would look something like this:
```json
{
    "name": "readOnlyScm",
    "managedByScrewdriver": true,
    "maintainer": "foo@bar.com",
    "description": "Read-only open source mirror",
    "scmContext": "gitlab:gitlab.screwdriver.cd",
    "scmOrganizations": [],
    "isActive": true,
    "weightage": 100
}
```

##### GitLab example
```yaml
# config/local.yaml
scms:
    ...
    # Below is read-only SCM example
    gitlab:
        plugin: gitlab
        config:
            oauthClientId: YOU-PROBABLY-WANT-SOMETHING-HERE # The client id used for OAuth with GitLab.
            oauthClientSecret: AGAIN-SOMETHING-HERE-IS-USEFUL # The client secret used for OAuth with GitLab
            gitlabHost: gitlab.screwdriver.cd # [Optional] GitLab enterprise host
            commentUserToken: A_BOT_GITLAB_PERSONAL_ACCESS_TOKEN # [Optional] Token for writing PR comments in GitLab
            https: false # [Optional] Is the Screwdriver API running over HTTPS, default false
            readOnly: # [Optional] Block for read-only SCM
                enabled: true # Set to true to enable read-only mode
                username: sd-buildbot # Headless username
                accessToken: GITLAB-TOKEN # Headless GitLab token with read-only access to repos and ability to add webhooks
                cloneType: https # [Optional] Set to ssh or https; default https
```

##### Bitbucket.org example
```yaml
# config/local.yaml
scms:
    bitbucket:
        plugin: bitbucket
        config:
            oauthClientId: YOUR-APP-KEY # The client id used for OAuth with Bitbucket.
            oauthClientSecret: YOUR-APP-SECRET # The client secret used for OAuth with Bitbucket
            https: true # [Optional] Is the Screwdriver API running over HTTPS, default false
            username: sd-buildbot # [Optional] Username for code checkout
            email: dev-null@screwdriver.cd # [Optional] Email for code checkout
```

### Webhooks

Set these environment variables:

| Environment name             | Required       | Default Value | Description                                      |
|:-----------------------------|:---------------|:--------------|:-------------------------------------------------|
| SCM_USERNAME                 | No             | sd-buildbot   | Obtains the SCM token for a given user. If a user does not have a valid SCM token registered with Screwdriver, it will use this |
| IGNORE_COMMITS_BY            | No             | []            | Ignore commits made by these users               |
| RESTRICT_PR                  | No             | none          | Restrict PR: all, none, branch, or fork |

```yaml
# config/local.yaml
webhooks:
  username: SCM_USERNAME
  ignoreCommitsBy:
    __name: IGNORE_COMMITS_BY
    __format: json
  restrictPR: RESTRICT_PR
```

### Rate Limiting

Set these environment variables to configure rate limiting by authentication token:

| Environment name     | Default Value | Description          |
|:---------------------|:--------------|:---------------------|
| RATE_LIMIT_VARIABLES | `'{ "enabled": false, "limit": 300, "duration": 300000 }'` | JSON string configuration for rate limiting |

Or override the default with the following `config/local.yaml` file.

```yaml
# config/local.yaml
rateLimit:
    enabled: true
    # limit to max 60 requests in 1 minute
    limit: 60
    duration: 60000
```

### Canary Routing

If your Screwdriver Kubernetes Cluster is using [nginx Canary ingress](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary), then set this environment variable to have API server set a cookie for a limited duration such that subsequent API requests are served by same canary API pods.

| Environment name     | Example Value | Description          |
|:---------------------|:--------------|:---------------------|
| RELEASE_ENVIRONMENT_VARIABLES | `'{ "cookieName": "release", "cookieValue": "canary"}'` | JSON string configuration for release information |

Or override the default with the following `config/local.yaml` file.

```yaml
# config/local.yaml
# environment release information
release:
    mode: stable
    cookieName: release
    cookieValue: stable
    cookieTimeout: 2 # in minutes
    headerName: release
    headerValue: stable
```

### Configure Redis Lock
Redis lock is used by Screwdriver api to force sequential build update, leaving it disabled could result in data loss during build update and builds that never start.

Set these environment variables to configure Redis lock:

| Environment Variable                | Required            | Default              | Description       |
|:--------------------------|:---------------------|:---------------------|:-----------------------------|
| REDLOCK_ENABLED           | Yes                  | false                | Enable Redis lock            |
| REDLOCK_RETRY_COUNT       | Yes                  | 200                  | Maximum retry limit to obtain lock                  |
| REDLOCK_DRIFT_FACTOR      | No                   | 0.01                 | The expected clock drift     |
| REDLOCK_RETRY_DELAY       | No                   | 500                  | The time in milliseconds between retry attempts            |
| REDLOCK_RETRY_JITTER      | No                   | 200                  | The maximum time in milliseconds randomly added to retries             |
| REDLOCK_REDIS_HOST        | Yes                  | 127.0.0.1            | Redis host                   |
| REDLOCK_REDIS_PORT        | Yes                  | 9999                 | Redis port                   |
| REDLOCK_REDIS_PASSWORD    | No                   | THIS-IS-A-PASSWORD   | Redis password               |
| REDLOCK_REDIS_TLS_ENABLED | No                   | false                | Redis tls enabled            |
| REDLOCK_REDIS_DATABASE    | No                   | 0                    | Redis db number              |

```yaml
# config/local.yaml
redisLock:
  # set true to enable redis lock
  enabled: true
  options:
    # maximum retry limit to obtain lock
    retryCount: 200
    # the expected clock drift
    driftFactor: 0.01
    # the time in milliseconds between retry attempts
    retryDelay: 500
    # the maximum time in milliseconds randomly added to retries
    retryJitter: 200
    # Configuration of the redis instance
    redisConnection:
        host: "127.0.0.1"
        port: 6379
        options:
            password: '123'
            tls: false
        database: 0
        prefix: ""
```

### Logging
For more verbose or precise logging, you can configure these environment variables:

| Environment Variable      | Required            | Default              | Description       |
|:--------------------------|:--------------------|:---------------------|:-----------------------------|
| LOG_AUDIT_ENABLED         | No                  | false                | Enable audit logs for all API calls            |
| LOG_AUDIT_SCOPE           | No                  | []                   | Target token scopes (e.g. pipeline, build, temporal, admin, guest, user) |

```yaml
# config/local.yaml
log:
  audit:
    # set true to enable audit logs for all API calls
    enabled: false
    # add target scope tokens(pipeline, build, temporal, admin, guest, user)
    scope: []
```

Example output logs:
```
{"level":"info","message":"[Login] User tkyi get /v4/events/41/builds","timestamp":"2022-11-04T22:19:33.039Z"}
{"level":"info","message":"[Login] Pipeline 7 post /v4/pipelines/7/sync","timestamp":"2022-11-04T22:19:33.985Z"}
```

## Extending the Docker container

There are some scenarios where you would prefer to extend the Screwdriver.cd Docker image, such as using custom Bookend plugins. This section is not meant to be exhaustive or complete, but will provide insight into some of the fundamental cases.

### Using a custom bookend

Using a custom bookend is a common case where you would extend the Screwdriver.cd Docker image.

In this chosen example, we want to have our bookend execute before the `scm` (which checks out the code from the configured SCM). Although the bookend plugins can be configured by environment variables, we will show how to accomplish the same task with a `local.yaml` file.

This is shown in the following `local.yaml` snippet:

```yaml
# local.yaml
---
  ...
bookends:
  default:
    setup:
      - my-custom-bookend
      - scm
```

For building our extended Docker image, we will need to create a `Dockerfile` that will have our extra dependencies installed. If you would prefer to save the `local.yaml` configuration file in the Docker image instead of mounting it in later, you may do so in the Dockerfile as well.

```dockerfile
# Dockerfile
FROM screwdrivercd/screwdriver:stable

# Install additional NPM bookend plugin
RUN cd /usr/src/app && /usr/local/bin/npm install my-custom-bookend

# Optionally save the configuration file in the image
ADD local.yaml /config/local.yaml
```

Once you build the Docker image, you will need to deploy it to your Screwdriver.cd cluster. For instance, if you're using Kubernetes, you would replace the `screwdrivercd/api:stable` image to your custom Docker image.

The following is an example snippet of an updated Kubernetes deployment configuration:

```yaml
# partial Kubernetes configuration
  ...
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: screwdriver-api
        # The image name is the one you specified when built
        # The tag name is the tag you specified when built
        image: my_extended_docker_image_name:tag_name
```
