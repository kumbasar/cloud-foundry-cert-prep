# Cloud Foundry Workspace

[Suse Cloud Application Platform Developer Sandbox](https://developer.suse.com/capsandbox/) 

- Cloud Foundry is a Platform ad a Service (PaaS)
- Applications run on cells => containers
- Buildpack => deployment of applications. Supports a variety of languages
## CF Login

**Target**: Cloud Foundry instance

API endpoint convention: `api.<cloudfoundry-system-domain>`

```bash
cf api https://api.cap.explore.suse.dev
cf login -u welcome -p <PASSWORD>
```

Or login oneshot:

```bash
cf login -a https://api.cap.explore.suse.dev -u welcome -p <PASSWORD>
```

## CF Help

**Basic**
```bash
cf help
```

**All**
```bash
cf help -a
```

**Command help**
```bash
cf <command> --help 
```

Example:
```bash
cf login --help 
```

## Orgs and Spaces

- To list `orgs` in target:
```bash
cf orgs
```
- To list `spaces` in target:
```bash
cf spaces
```

Example:
```bash
Getting spaces in org welcome as welcome...

name
dev
samples
test
```
- To verify the current `target`:
```bash
cf target
```

Example output:
```bash
api endpoint:   https://api.cap.explore.suse.dev
api version:    2.153.0
user:           welcome
org:            welcome
space:          dev
```

## CF cURL - API version

```bash
cf curl /v2 info
```

Example output
```bash
{
   "name": "IBM Cloud",
   "build": "v15.5.0",
   "support": "http://ibm.biz/bluemix-supportinfo",
   "version": 0,
   "description": "IBM Bluemix",
   "authorization_endpoint": "https://iam.cloud.ibm.com/cloudfoundry/login/uk-south",
   "token_endpoint": "https://uaa.eu-gb.cf.cloud.ibm.com",
   "min_cli_version": null,
   "min_recommended_cli_version": null,
   "app_ssh_endpoint": "ssh.eu-gb.cf.cloud.ibm.com:2222",
   "app_ssh_host_key_fingerprint": "c7:1f:89:2a:62:3b:78:a9:08:c9:33:81:fb:39:26:da",
   "app_ssh_oauth_client": "ssh-proxy",
   "doppler_logging_endpoint": "wss://doppler.eu-gb.cf.cloud.ibm.com:443",
   "api_version": "2.158.0",
   "osbapi_version": "2.15",
   "user": "ba6f3c88-3ab3-4e21-a4b3-a52b5210ce10"
}
```

cli version shoud be greater or equal to the `min_cli_version` of the target

## Debugging request and request

### Verbose flag
Append the verbose flag `-v` to the command

```bash
cf <command> -v
```

Example:

```bash
cf target -v
```

### CF_TRACE variable

To enable `CF_TRACE`

```bash
export CF_TRACE=true
cf <command>
cf <command>
```

To disable tracing:
```bash
export CF_TRACE=false
```

or run tracing for a single command

```bash
CF_TRACE=true cf <command>
```

### Tracing via cf config

```bash
cf config --trace true
```

To disable:
```bash
cf config --trace false
```

## Concerns

### CLI 
- Written in GoLang
- Interacts with CF (== Cloud Controller)
- Target and authenticate (login) to CF.
- Makes REST call to the exposed CF API
- Functionality can be extended via [plugins](https://plugins.cloudfoundry.org)
- CLI <- Rest -> cloud conroller api (capi)

### Cloud Controller
- Exposes the REST APIs of Cloud Foundry
- Answers to the CLI
- Works with other components

### Diego
- Responsible for the lifecycle of applications/tasks
- Contains one or more compute nodes (== VM) called Cells. Cells run containers, which execute our applications and tasks.
- cells < Diego

### Router
- For routing traffic into applications (==Diego) and to the cloud controllers

### Buildpacks
- Providing required runtime dependencies (JRE, npm) for the applications
- Responsible for preparing applications for execution inside containers in Diego.
- Droplet: Application + runtime dependencies => Executed inside a container

### User Account and Authentication (UAA)
- OAuth2 provider
- issue tokens for client apps


### Loggregator
- Aggregate logs (apps), events and metrics from the cells.
- Cells (Diego) - logs/evets/metrics -> loggreator - logs/evets/metrics -> users

### Service Broker
-  Allows back'ng services to be provisioned and consumed using the CF APIs, without knowledge of the underlying service
- Cloud Controller <-> Service Brocker API/Service Broker <-> external services (Postgresql, MySQL)

### Deploying Apps aka cf push


Example:

- App: `rest-data-service.jar`
- Instance size: `1`
- Memory: `750M`
- Buildpackage: `java_buildpack`
- Route: `random-route`

```bash
cf push roster -p rest-data-service.jar -i 1 -m 750M -b java_buildpack --random-route
```

Output:
```bash
name:              roster
requested state:   started
routes:            roster-bright-tasmaniandevil-ys.cap.explore.suse.dev
last uploaded:     Tue 02 Feb 23:13:00 +03 2021
stack:             sle15
buildpacks:        java

type:            web
instances:       1/1
memory usage:    750M
start command:   JAVA_OPTS="-agentpath:$PWD/.java-buildpack/open_jdk_jre/bin/jvmkill-1.16.0_RELEASE=printHeapHistogram=1 -Djava.io.tmpdir=$TMPDIR
                 -XX:ActiveProcessorCount=$(nproc) -Djava.ext.dirs=$PWD/.java-buildpack/container_security_provider:$PWD/.java-buildpack/open_jdk_jre/lib/ext
                 -Djava.security.properties=$PWD/.java-buildpack/java_security/java.security $JAVA_OPTS" &&
                 CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-3.13.0_RELEASE -totMemory=$MEMORY_LIMIT
                 -loadedClasses=15880 -poolType=metaspace -stackThreads=250 -vmOptions="$JAVA_OPTS") && echo JVM Memory Configuration: $CALCULATED_MEMORY &&
                 JAVA_OPTS="$JAVA_OPTS $CALCULATED_MEMORY" && MALLOC_ARENA_MAX=2 SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java
                 $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher
     state     since                  cpu    memory           disk           details
#0   running   2021-02-02T20:13:24Z   0.0%   116.2M of 750M   140.2M of 1G
```
