# High Availability in Kubernetes using failover with hot-standby web application

In this repository I will describe my journey investigating possibility to run web application in failover with hot-standby mode in Kubernetes.

## Assumptions

- I am using TIBCO EBX 5.9.26 for the test.
- I discovered proposal [#45300](https://github.com/kubernetes/kubernetes/issues/45300) which is issue in Kubernetes GitHub
- I also will consider [this](https://stackoverflow.com/questions/47291581/kubernetes-service-with-clustered-pods-in-active-standby)
- and [this](https://github.com/amelbakry/kubernetes-active-passive)

## TIBCO EBX failover with hot-stanby

The official TIBCO EBX documentation which is available [here](https://docs.tibco.com/pub/ebx/5.9.26/doc/html/en/index.html) contains explanation that EBX supports single JVM per repository and the exclusion mechanism is compatible with failover architectures, where only one server is active at any given time in an active/passive cluster. 

To ensure that, the main server must declare the property 'ebx.repository.ownership.mode=failovermain'.
A backup server can still start up, but it will not have access to the repository. It must declare the property 'ebx.repository.ownership.mode=failoverstandby'. 
Its status can be retrieved using the Java API, through an HTTP request or using UI.

### Configuring failover in ebx.properties

ebx.properties file contains section where you set the ownership over the repository.
To be able to do it relatively I use environment variables.

```bash
#################################################
## Mode used to qualify the way in which a server accesses the repository. 
## Possible values are: unique, failovermain, failoverstandby.
## Default value is: unique.
#################################################
ebx.repository.ownership.mode=${repository_ownership_mode}
 
## Activation key used in case of failover. The backup server must include this
## key in the HTTP request used to transfer exclusive ownership of the repository.
## The activation key must be an alphanumeric ASCII string longer than 8 characters.
ebx.repository.ownership.activationkey=${repository_ownership_activationkey}
 
## Specifies whether to hide heartbeat logging in DEBUG mode.
## Default value is true.
#ebx.repository.ownership.hideHeartBeatLogForDebug=true
```


### Checking the repository status

A log of all attempted Java process connections to the repository is available in the Administration area under 'History and logs' > 'Repository connection log'.

It is also possible to get the repository status information using an HTTP request that includes the parameter 'repositoryInformationRequest' (http[s]://<host>[:<port>]/ebx?repositoryInformationRequest) with one of following values:

**state**
The state of the repository in terms of ownership registration.

    D: Java process is stopped. (Dead)

    O: Java process has exclusive ownership of the database. (Owned)

    S: Java process is started in failover standby mode, but is not yet allowed to interact with the repository. (Standby)

    N: Java process has tried to take ownership of the database but failed because another process is holding it. (Not Owned)

**heart_beat_count**
The number of times that the repository has made contact since associating with the database.

**info**
Detailed information for the end-user regarding the repository's registration status. The format of this information may be subject to modifications in the future without explicit warning.

### Activate standby server

By requesting using parameter activationKeyFromStandbyMode, and the value equal to the activation key declared in ebx.properties, standby server takes the ownership.

The format of the request URL must be:

    http[s]://<host>[:<port>]/ebx?activationKeyFromStandbyMode={value}

## Kubernetes settings

### Create statefulset

will test and let you know.

name every POD like demo-main-0, demo-standby-1

use redinesss and livliness probes

TIBCO EBX 5 doesn't support health operations so I need to use different aproach to monitor. So I check for a unique string that shows application is started and create a dummy file

readinessProbe:
   exec:
     command:
       - sh
       - -c
       - test -f /tmpready/ready

