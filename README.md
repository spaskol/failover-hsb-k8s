# High Availability in Kubernetes using failover with hot-standby web application

In this repository I will describe my journey investigating possibility to run web application in failover with hot-standby mode in Kubernetes.

## Assumptions

- I am using TIBCO EBX 5.9.26 for the test.
- I discovered proposal [#45300](https://github.com/kubernetes/kubernetes/issues/45300) which is issue in Kubernetes GitHub.
- I also will consider [this](https://stackoverflow.com/questions/47291581/kubernetes-service-with-clustered-pods-in-active-standby) article in Stackoverflow.
- and [this](https://github.com/amelbakry/kubernetes-active-passive) github repository.

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

### Create statefulset with readiness probe

After some tests I realized that I don't need one main and one standby instances but two standby.

I created two statefulsets "sfs-a" and "sfs-b" with the same standby configurations as described in the TIBCO EBX documentation and the following redinessProbe:

``` yaml
readinessProbe:
   exec:
     command:
       - /bin/sh
       - -c
       - |
          result=$(curl -s http://localhost:8080/ebx/?repositoryInformationRequest=state)
          if [ "$result" = "O" ]; then
            exit 0
          else
            exit 1
          fi
     initialDelaySeconds: 60
     timeoutSeconds: 10
     periodSeconds: 5
     successThreshold: 12
```
With this configuration the two pods will run sfs-a-0 and sfs-b-0 but will be not ready until you do request as explained in TIBCO EBX documentation to take the ownership of the repository. After one of the pod takes the ownership its state became "O" and the pod is read thus accepting traffic. If you take the ownership from the other one (sts-b-0) the first one (sts-a-0) became dead ("D") thus not ready and after around 40 seconds the other pod takes the ownership.

This setup is similar to Blue-green deployment strategy. This means that improves the operational resilience of Kubernetes workloads, allowing developers to safely test the new deployment in production cluster without immediately exposing the changes to users.

Note: TIBCO EBX 5 doesn't support health operations but the 6th version does.
