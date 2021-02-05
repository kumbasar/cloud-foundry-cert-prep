# Cloud Foundry Workspace

[Suse Cloud Application Platform Developer Sandbox](https://developer.suse.com/capsandbox/) 

- Cloud Foundry is a Platform as a Service (PaaS)
- Applications run on cells => containers
- Buildpack => deployment of applications. Supports a variety of languages
- Runs on the top an IaaS (AWS, GCP or OpenStack)
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
cf curl /v2/info
```

Example output
```bash
{
   "name": "KubeCF",
   "build": "2.1.0",
   "support": "https://www.suse.com/support/",
   "version": 2,
   "description": "SUSE version of CloudFoundry KubeCF",
   "authorization_endpoint": "https://login.cap.explore.suse.dev",
   "token_endpoint": "https://uaa.cap.explore.suse.dev",
   "min_cli_version": null,
   "min_recommended_cli_version": null,
   "app_ssh_endpoint": "ssh.cap.explore.suse.dev:2222",
   "app_ssh_host_key_fingerprint": "8d:4f:be:68:3e:94:9a:3e:92:b8:4d:ab:06:dd:db:c7",
   "app_ssh_oauth_client": "ssh-proxy",
   "doppler_logging_endpoint": "wss://doppler.cap.explore.suse.dev:443",
   "api_version": "2.153.0",
   "osbapi_version": "2.15",
   "routing_endpoint": "https://api.cap.explore.suse.dev/routing",
   "user": "ac7b46ca-075a-4181-94c3-2f548cbcfb7d"
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
- Runtime Standardization
 
### User Account and Authentication (UAA)
- OAuth2 provider
- issue tokens for client apps


### Loggregator
- Aggregate logs (apps), events and metrics from the cells.
- Cells (Diego) - logs/evets/metrics -> loggreator - logs/evets/metrics -> users

### Service Broker
-  Allows back'ng services to be provisioned and consumed using the CF APIs, without knowledge of the underlying service
- Cloud Controller <-> Service Brocker API/Service Broker <-> external services (Postgresql, MySQL)
- Extends platform
- Supports different SW org

## Deploying Apps aka cf push

CLI - `cf push` -> cloud controller -- cc Db

Steps:
1. App metadata (space, number of instances)
2. Route reservation if requested. Assocaited with app
3. Upload app binary
4. Stating - Preparing buildpack to run the app in a binary package (droplet). Droplet is stored in blobstore
5. Run the app package (droplet) inside a container on a cell inside diego
6. Available. Router routes app traffic to running instance(s)

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

Check out [roster](https://roster-bright-tasmaniandevil-ys.cap.explore.suse.dev)

### Scale

```bash
cf scale --help
```

```bash
cf scale APP_NAME [-i INSTANCES] [-k DISK] [-m MEMORY] [-f]
```
- Vertical scaling (memory/disk): Downtime and slow => New container with the new resources (restart required)
- Horizintal scaling (instance size): No downtime and fast => New container with already cached droplet.

### App Status

```bash
cf app <APP_NAME>
```

### Logging&Metrics

- Metron agents collects the app logs on the cells in Diego
- Metron agents fwd the logs yo the Doppler servers (loggregator)
- Diego/Cells/Metron - logs, events, metrics -> Doppler/Loggregator
- Logs: Standart out and error outputs
- Lables:
    - **STG**: staging logs
    - **APP/PROC/<name>/<index>**: App instance runtime logs
    - **CELL/<index>**: Diego cell logs
    - **RTR**: Router logs
- To fetch the logs: `cf logs <APP_NAME>
- Loggregator / Traffic Controller - logs -> tail/recent - websocket -> CLI => Logs for developers
- Loggregator / Traffic Controller -> firehouse - logs/metrics -> Nozzle => Log plus metrics (cpu etc)
- Loggregator / Doppler -> drain -> elastic => Archive
- Logs can be droped to not affects application performance
- Diego Metron -> doppler  <- router

```bash
cf logs --help
```

`--recent`: show past logs from the buffer

To see events:

```bash
cf events --help
```

Example:
```bash
Getting events for app roster in org welcome / space dev as welcome...

time                          event                      actor                  description
2021-02-03T20:16:00.00+0300   audit.app.update           vkumbasar@icloud.com   instances: 2
2021-02-03T20:10:03.00+0300   audit.app.update           vkumbasar@icloud.com   state: STARTED
2021-02-03T20:10:02.00+0300   audit.app.update           vkumbasar@icloud.com   state: STOPPED
2021-02-03T20:10:00.00+0300   audit.app.update           vkumbasar@icloud.com   memory: 1024
2021-02-02T23:13:00.00+0300   audit.app.droplet.create   vkumbasar@icloud.com
2021-02-02T23:12:38.00+0300   audit.app.update           vkumbasar@icloud.com   state: STARTED
2021-02-02T23:12:37.00+0300   audit.app.build.create     vkumbasar@icloud.com
2021-02-02T23:12:23.00+0300   audit.app.upload-bits      vkumbasar@icloud.com
2021-02-02T23:12:18.00+0300   audit.app.map-route        vkumbasar@icloud.com
2021-02-02T23:12:18.00+0300   audit.app.create           vkumbasar@icloud.com   instances: 1, memory: 750, state: STOPPED, environment_json: [PRIVATE DATA HIDDEN]
```

### Resiliency
- BOSH: creating and managing Cloud Foundry on top of different cloud providers.
- Deploy app in different AZ
- Failed app instances are automatically recreated
- Diego deploys instances of the same app across different cells and AZs

### Application Lifecycle

**The Concerns:**
- Application Code (Continous Delivery)
- Application Language Runtime (buildpacks)
- Root Filesystem
- Cell Operation System (Stemcells)

### Services

**Service**: External element that the app can interact. In CF service means the offer type. Whereas, service instance means the instance of the service which the apps interacts.

* Service == Class
* Service Instance == Object

**Managed Services**: Provision service resources on demand via Service Broker API.
=> Instance of Managed Service => created and bound

Cloud Controller <-> Service Broker API / Service Broker <-> External service

**Marketplace**: Managed service offers

Managed service offer:
1. `cf marketplace`: List all offers
Example:
```bash
Getting services from marketplace in org welcome / space dev as welcome...
OK

service      plans     description                 broker
postgresql   11-6-0    Helm Chart for postgresql   minibroker1.0
rabbitmq     3-8-1     Helm Chart for rabbitmq     minibroker1.0
redis        5-0-6     Helm Chart for redis        minibroker1.0
mariadb      10-3-21   Helm Chart for mariadb      minibroker1.0
mongodb      4-2-3     Helm Chart for mongodb      minibroker1.0

TIP: Use 'cf marketplace -s SERVICE' to view descriptions of individual plans of a given service.
```

To see plan exmaple:
```bash
Getting service plan information for service mariadb as welcome...
OK

service plan   description                                                                                                                                                                                                                                                free or paid
10-3-21        Fast, reliable, scalable, and easy to use open-source relational database system. MariaDB Server is intended for mission-critical, heavy-load production systems as well as for embedding into mass-deployed software. Highly available MariaDB cluster.   free
```


2. `cf create-service`: Call broker to provision of the service with plan
Example:
```bash
cf create-service mariadb 10-3-21 dbroster
```

To checkout
```bash
cf services
```

Example output:
```bash
Getting services in org welcome / space dev as welcome...

name       service   plan      bound apps   last operation     broker          upgrade available
dbroster   mariadb   10-3-21                create succeeded   minibroker1.0
```

3. `cf bind-service`: Broker provides credentials to app via env vars `VCAP_SERVICES`
Example:
```bash
cf bind-service roster dbroster
```
1. `cf restage or restage`: Restart/restage app to access the env vars `VCAP_SERVICES`

To check the environment variables:

```bash
cf env <APP_NAME>
```

**User-Provided Service Instances (UPSI)**
- Legacy systems
- 3rd party systems (outside of CF)
- Systems which are not integrated in CF


**Space Scoped Brokers**
Space developers might register a broker with the cloud controller

```bash
cf create-service-broker --space-scoped
```

### Restage vs Restart

```bash
cf restage <APP_NAME>
```

```bash
cf restart <APP_NAME>
```

Restage: Buildpackage update
Restart: Droplet won't change, restart of application


## Twelve-Factor Applications
Recommendations, not requirements

1. Codebase
- Version control is a must
- Library - dependency
- Singular codebase -> different envs: prod, test and dev.

2. Dependencies
- Declares all dependencies via manifest

3. Configuration
- Example: `VCAP+SERVICES`

4. Backing Services
- Any service over the network

5. Build, Release, Run
- Build stage: code 2 executable bundle
- Release stage: Combines build with deployment's current configuration
- Run stage: run the application in the execution environment

6. Processes
- Are stateless and share-nothing
- Persist data is stored in stateful backing service (db)
- Horizontally scale

7. Port Binding
- CF supports both HTTP and TCP port bindings. Env var: `$PORT`
- Port should not be hard coded.

8. Concurrency
- In CF vertical and horizontal scale
- Adding new microservices
- Cloud Controllers handles user-initiated restarts and shutdows
- Diego response to crashed processes
- Loggreator to manage output streams

9. Disposability
- Minimize startup time
- Elastic scaling
- Caching droplets
- Shut down gracefully
- In CF it takes 5 min to start up.

10. Dev/Prod Parity
- Gap between dev and prod is small

11. Logs
- Running process writes its event stream, unbuffered, to stdout.
- In CF: Loggreator
- Logs == event streams

12. Administrative Processses


### Application Environment Variables

```bash
cf env <APP_NAME>
```

Set env var:
```bash
cf set-env APP_NAME ENV_VAR_NAME ENV_VAR_VALUE
```

Example:
```bash
cf set-env roster ROSTER_A ROSTER_A_VAR
cf set-env roster ROSTER_B ROSTER_B_VAR
cf set-env roster ROSTER_C ROSTER_C_VAR
```

### Manifest

Example:
```yml
version: 1
applications:
- name: roster
  memory: 750M
  instances: 1
  random-route: true
  buildpacks:
  - java_buildpack
  path: ./rest-data-service.jar
  env:
    ROSTER_A: ROSTER_A_VAR
    ROSTER_B: ROSTER_B_VAR
    ROSTER_C: ROSTER_C_VAR
  services:
  - dbroster 
```

Push:
```bash
cf push
```

### CUPS - User-Provided Service Instance

Usage:
```bash
cf create-user-provided-service SERVICE_INSTANCE [-p CREDENTIALS] [-l SYSLOG_DRAIN_URL] [-r ROUTE_SERVICE_URL] [-t TAGS]
```

Log Drain example:

Free service: [Papertrail](https://papertrailapp.com/)

```bash
URL="syslog-tls://logs2.papertrailapp.com:XXXXX"
cf cups papertrail -l $URL
cf bs roster papertrail
cf restage roster
```

Checkout [Dashboard](https://papertrailapp.com/dashboard)

### Routes
- By default HTTP 80 and 443 is supported
- TCP is also supported (IoT solutions)
- GoRouter routes incomming traffice to Cloud Controller or to App (Diego Cell)
- GoRouter provides:
    - Round-robin load-balancing
    - HTTPS traffic. `X-Forwarded-Proto` header.
    - Ticky sessions when the `JSESSIONID` cookie appears.
- HTTP route format: <hostname>.<appdomain>/<contextpath>
- Random routes should not be used for production applications.
- API URL: api.<systemdomain>
- DNS CNAME record
- System domain => Used by CF compenents like cloud controllers
- Shared apps domain: shared across CF instance
- Private apps domain: scopred to one or more Orgs
- Separating Applications and Routes
    - Zero Downtime updates
    - Many routes/urls for the same app (A/B testing, branding)
    - Testing and debugging
- Blue-Green Deployments:

Example:
```bash
cf se roster APP_VERSION blue
cf restage roster
cf push roster-green
cf a
cf map-route roster-green cap.explore.suse.dev --hostname roster-fantastic-oribi-nm
cf unmap-route roster roster-fantastic-oribi-nm.cap.explore.suse.dev
cf delete roster
cf rename roster-green roster
cf se roster APP_VERSION blue
cf restage roster
```
