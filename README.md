# Cloud Foundry Certified Developer (CFCD) Exam Preparations

[Suse Cloud Application Platform Developer Sandbox](https://developer.suse.com/capsandbox/) 

- Cloud Foundry is a Platform as a Service (PaaS)
- Applications run on cells => containers
- Buildpack => deployment of applications. Supports a variety of languages
- Runs on the top an IaaS (AWS, GCP or OpenStack)
- Cloud Foundry supports OCI-compliant (Docker) container images.
- Spaces can be used to separate env (development, staging, production).

## CF Login and Auth

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

**Non-Interactive Authentication**

Environment variables: `CF_USERNAME` and `CF_PASSWORD`

```bash
cf auth --help
```

**One-time passcode**

```bash
cf login --sso
```

**Logout**
```bash
cf logout
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

**Scope**

```bash
cf share-service SERVICE_INSTANCE -s OTHER_SPACE
```

## Global Roles

### Org Roles
- OrgManager: Administer the org
- OrgAuditor: Read-only access to the org
- BillingManager

### Space Roels:
- SpaceManager: Administer role
- SpaceDeveloper: Manage apps, services and routes
- SpaceAuditor: Read-only access to the space

## CF cURL - API version

```bash
cf api --version
```

Output:
```bash
api endpoint:   https://api.cap.explore.suse.dev
api version:    2.153.0
```

or

```bash
cf curl /v2/info
```

Example output:
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
- Responsible for starting applications and tasks
- Making sure applications stay running
- Placing applications across many VMs for resilience
- Orchestrations
- **Garden**: A platform-agnostic API. Pluggable backends for the Open Container Initiative (OCI) specifications. Also supports Docke images. Two backends:
    1. Guardian: Linux runc
    2. Greenhouse: Windows backend

### App Instance
- An App Instance contains: RootFS, Droplet files and Json Environment objects (`VCAP_APPLICATION` abn `VCAP_SERVICES`)
- Runs inside a Diego cell
- `$PORT` used for network traffic


## Application Execution and Security Groups (ASGs)

- Allows to define various egress network access rules for containers
- Two pre-configured defaults:
1. **public_networks**: 
    - Allow public network access. 
    - Blocks access to private networks
    - Block access to link=local access
2. **dns**: Allow access to DNS (port #53)
- Runtime ASGs: Allow access application instances over the network to any resources they require. White-listed via CF-CLI
- Uses iptables?
- Staging-Time ASGs: To pull resources from the network when an application is being transformed into a droplet via Could Controller API, not CF CLI.
- Both ASG: Org-/Space-scoped extensions of white-listed access
## Router
- For routing traffic into applications (==Diego) and to the cloud controllers

## Buildpacks
- Providing required runtime dependencies (JRE, npm) for the applications
- Responsible for preparing applications for execution inside containers in Diego.
- Droplet: Application + runtime dependencies => Executed inside a container
- Runtime Standardization
 
## User Account and Authentication (UAA)
- OAuth2 provider
- issue tokens for client apps

## Loggregator
- Aggregate logs (apps), events and metrics from the cells.
- Cells (Diego) - logs/evets/metrics -> loggreator - logs/evets/metrics -> users

## Service Broker
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

## Scale

```bash
cf scale --help
```

```bash
cf scale APP_NAME [-i INSTANCES] [-k DISK] [-m MEMORY] [-f]
```
- Vertical scaling (memory/disk): Downtime and slow => New container with the new resources (restart required)
- Horizintal scaling (instance size): No downtime and fast => New container with already cached droplet.

## App Status

```bash
cf app <APP_NAME>
```

## Logging&Metrics

- Metron agents collects the app logs on the cells in Diego
- Metron agents fwd the logs yo the Doppler servers (loggregator)
- Diego/Cells/Metron - logs, events, metrics -> Doppler/Loggregator
- Logs: Standart out and error outputs
- Labels:
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

## Resiliency
- BOSH: creating and managing Cloud Foundry on top of different cloud providers.
- Deploy app in different AZ
- Failed app instances are automatically recreated
- Diego deploys instances of the same app across different cells and AZs

## Application Lifecycle

**The Concerns:**
- Application Code (Continous Delivery)
- Application Language Runtime (buildpacks)
- Root Filesystem
- Cell Operation System (Stemcells)

## Services

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

## Restage vs Restart

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
- Example: `VCAP_SERVICES`

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


## Application Environment Variables

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

## CUPS - User-Provided Service Instance

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

## Routes
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

## Staging and Running

**Staging**: the buildpack combines the application code with any framework and/or runtime dependencies to produce a droplet. It reproduce a droplet.


## Pushing an Application

- User executes: `cf push` via CF CLI
- CF CLI -> Cloud Controller (CCNG)
- App metadata (space, app name' instance size, allocated resoures) stored in CCDB
- App files stored in CCNG blobstore
- Stage app -> Diego Cell (Staging)
- Store app droplet in CCNG blobstore
- Start Staged app inside Diego Cell (Running)


## Buildpacks

List buildpacks:
```bash
cf buildpacks
```

Example output:

```bash
buildpack               position   enabled   locked   filename                                       stack
hugo_buildpack          1          true      false    hugo-buildpack.zip                             sle15
staticfile_buildpack    2          true      false    staticfile-buildpack-sle15-v1.5.12.1.zip       sle15
staticfile_buildpack    3          true      false    staticfile-buildpack-cflinuxfs3-v1.5.9.zip     cflinuxfs3
nginx_buildpack         4          true      false    nginx-buildpack-sle15-v1.1.15.1.zip            sle15
nginx_buildpack         5          true      false    nginx-buildpack-cflinuxfs3-v1.1.12.zip         cflinuxfs3
java_buildpack          6          true      false    java-buildpack-sle15-v4.32.1.1.zip             sle15
java_buildpack          7          true      false    java-buildpack-cflinuxfs3-v4.32.1.zip          cflinuxfs3
ruby_buildpack          8          true      false    ruby-buildpack-sle15-v1.8.25.1.zip             sle15
ruby_buildpack          9          true      false    ruby-buildpack-cflinuxfs3-v1.8.23.zip          cflinuxfs3
nodejs_buildpack        10         true      false    nodejs-buildpack-sle15-v1.7.30.1.zip           sle15
nodejs_buildpack        11         true      false    nodejs-buildpack-cflinuxfs3-v1.7.25.zip        cflinuxfs3
go_buildpack            12         true      false    go-buildpack-sle15-v1.9.19.1.zip               sle15
go_buildpack            13         true      false    go-buildpack-cflinuxfs3-v1.9.16.zip            cflinuxfs3
python_buildpack        14         true      false    python-buildpack-sle15-v1.7.23.1.zip           sle15
python_buildpack        15         true      false    python-buildpack-cflinuxfs3-v1.7.18.zip        cflinuxfs3
php_buildpack           16         true      false    php-buildpack-sle15-v4.4.22.1.zip              sle15
php_buildpack           17         true      false    php-buildpack-cflinuxfs3-v4.4.19.zip           cflinuxfs3
binary_buildpack        18         true      false    binary-buildpack-sle15-v1.0.36.1.zip           sle15
binary_buildpack        19         true      false    binary-buildpack-cflinuxfs3-v1.0.36.zip        cflinuxfs3
dotnet-core_buildpack   20         true      false    dotnet-core-buildpack-sle15-v2.3.16.1.zip      sle15
dotnet-core_buildpack   21         true      false    dotnet-core-buildpack-cflinuxfs3-v2.3.13.zip   cflinuxfs3
r_buildpack             22         true      false    r-buildpack-cflinuxfs3-v1.1.7.zip              cflinuxfs3
```

Multiple buildbacks:

```bash
cf push -b buildpack1 -b buildpack2 -b buildpack3
```

Here `buildpack1` and `buildpack2` are non-final and `buildpack3` is the final buildpack. Non-finals are only supply dependencies.

Buildpack API:
- detect: determines the rigth buildpack
- supply: adds dependency to the droplet. Executes for every buildpack.
- finalize: prepares the app or launch
- release: provides metadata. Start command

Buildpack can be:
1. Offline: Fully contained, no download
2. Online: Dependencies are downloaded either remotely or from a local source

**Stacks**: Provide the root filesystem (rootFS)

**Launcher**: `Droplet` + `stack` > `Garden container`

## SSH

```bash
cf enable-ssh <APP_NAME>
cf ssh <APP_NAME>
```

Example:
```bash
cf enable-ssh roster
Enabling ssh support for 'roster'...

OK
```

To check:
```bash
cf ssh-enabled roster      
ssh support is enabled for 'roster'
```

SSH to instance:

```bash
cf ssh <APP_NAME> --app-instance-index <INSTANCE_INDEX> 
```

Start command: `/var/vcap/staging_info.yml`

### Task

Running a task example:

```bash
cf run-task training-app -m 8M -k 64M --name printenv "tasks/printenv.sh"
```

Status:
```bash
cf tasks training-app
```

### Route Services 

Route services are bount to a route, not an appplication

Provides:
- Authentication/Authorization
- Rate limiting
- Caching services

**Fully-Brokered Service: CF Router-first**
User -> Router -> Route Service Instance -> Router -> Diego/Cell/Container

**Static, Brokered Service: Service Intansce first**
- route service is out side CF
- User -> Route Service Instance -> Router -> Diego/Cell/Container

**Rate limit service** example

```bash
cf push rate-limit -f manifest-limit.yml
cf cups limit-service -r https://roster-service-limit.cap.explore.suse.dev
cf bind-route-service cap.explore.suse.dev --hostname web-ui-delightful-bear-pa limit-service
```

Test page: [Web UI](https://web-ui-delightful-bear-pa.cap.explore.suse.dev/people?)

### Docker

Push a image

```bash
cf push <APP_NAME> -o <DOCKER_IMAGE>
```

Example: push docker image and disable health check
```bash
cf push worker -o engineerbetter/worker-image --health-check-type none
```

## Service-key

```bash
cf create-service-key --help
```

Example: Create a service key and get cred info:
```bash
cf create-service-key dbroster dbrosterkey
cf service-key dbroster dbrosterkey
```


## User Account and Authentication (UAA)

- OAuth2 provider:
    - Issuing client tokens
    - authenticate users with their CF credentials
- OAuth2 hase 4 modes (grant types):
    1. authorization code
    2. password
    3. client credentials
    4. implicit
- Tokens:
    1. Access Tokens
    2.  Refesh tokens

## Create Route

```bash
cf create-route SPACE DOMAIN [--hostname HOSTNAME] [--path PATH]
```

Sample:

```bash
cf create-route dev cap.explore.suse.dev --hostname kumbasar-uaa
```

Check:
```bash
cf routes
```

## Internal domain

```bash
cf domains
```

Example output

```bash
Getting domains in org welcome as welcome...
name                          status   type   details
cap.explore.suse.dev          shared
tcp.cap.explore.suse.dev      shared   tcp
apps.internal                 shared          internal
volkan.cap.explore.suse.dev   owned
```

**internal** domain: `apps.internal `

### Container to container network

For networking, it's require to add a network policy:

```bash
cf add-network-policy --help
```

```bash
cf add-network-policy SOURCE_APP --destination-app DESTINATION_APP
```

The default port is: `8080`

Example:
```bash
cf add-network-policy web-ui --destination-app roster
```

Check:
```bash
cf network-policies
Listing network policies in org welcome / space dev as welcome...

source   destination   protocol   ports    destination space   destination org
web-ui   roster        tcp        8080     dev                 welcome
```


## Logging

**span**: A basic unit of work. Contains unique IDs, timing information, and other meta information. `X-B3-SpanId`. Parant span: `X-B3-ParentSpan`
**trace**:A set of spans. Represenet a logical request. `X-B3-TraceId`

## Coping Strategies
1. Additive Changes: Adding new features instead of changing existing. Doesn't allow feature remove.
2. Version Mediation: Versioning scheme
    - One Application Supporting Many Contracts
    - Client-Aware Routing
    - Opaque Routing


## References
- [EXAM PREPARATION CLOUD FOUNDRY CERTIFIED DEVELOPER](https://cfcd-prep.cloudfoundry.org)
- [Get Started with Cloud Foundry](https://www.cloudfoundry.org/get-started/)
- [Cloud Foundry Certified Developer](https://acloudguru.com/course/cloud-foundry-certified-developer)
- [Aiming for the Cloud Foundry Certified Developer (CFCD) certification? Here are some tips.](https://medium.com/@dchucks/aiming-for-the-cloud-foundry-certified-developer-cfcd-certification-here-are-some-tips-5c37732b4f34)