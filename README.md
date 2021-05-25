# ACM Operator repository

## ACM Features

![ACM diagram](img/GitOps-tools-ACM.png)

1. ACM Hub is the only resource that needs access to Git/Helm repos
1. ACM Agent initiates connections to the Hub, not vice versa
1. ACM Agent doesn't need access to Git/Helm repos
1. Built-in policy engine
1. Build-in observability platform

## Repository modeling

![Repo modelling](img/ACM-repository-modeling.png)

## What does it mean to onboard a Team?

1. New `git` repo created
1. ACM granted RO access to the repo
1. New tenant added to [projects](/projects) folder
    1. `gitops` repository defined in `channels` section
    1. `gitops` repository reference secret from previous step
    1. `helm` repository defined in `channels` section
1. Install tenant helm chart in ACM ARO cluster

    ```bash
    make deploy <project_name>
    ```

## Application lifecycle

### Creating a new application

1. `feature/new-app-<application_name>` branch created in the teams gitops repository
1. `Application` Resource Definition added to `applications` folder in teams gitops repository
1. `Subscription` Resource Definition added to respective zone in `zones` folder
1. PR raised for the changes, which trigger basic checks: syntax checks
1. Once PR is  merged, changes automatically picked up by ACM and deployed to the cluster

### Deploy an existing application

1. Developer submits PR for the application code changes
1. PR merge trigggers a release pipeline
1. Release pipeline modifies application version for the dev env in teams gitops repository
1. ACM deploys application to the respective environment
1. Pipelines polls ACM to validate status of deployment
1. Pipeline executes tests against environment
1. Once deployment succedded and user confirmation required to proceed with deployment into upper environments
1. Pipeline repeats steps 3-7 for upper environments
![Delivery pipeline](img/ACM-delivery-pipeline.png)

### Mappings

* Zone -> Cluster Namespace

---

## Tips

### Force subscription reconcile

```bash
oc label subscriptions.apps.open-cluster-management.io the_subscription_name reconcile=true
```

## Open questions

1. Multitenancy - still not very well defined
1. Monitoring of GitOps engine events (not supported_)
1. Recovery strategy
1. Granting `subscription-admin` privileges to certain namespaces only

## Known issues

### UI

1. UI doesn't show status of managed object
1. UI doesn't show target cluster of deployed object
![UI bug #1](img/ACM-ui-bug.png)

### Reconcile errors

1. Example of git branch misconfiguration. Problem can be discovered only in ACM container logs

    ```bash
    multicluster-operators-standalone-subscription I0429 12:38:33.048544       1 subscription_controller.go:291] Exit Reconciling subscription: fusion-operate/fusion-operate
    multicluster-operators-standalone-subscription E0429 12:38:33.141429       1 gitrepo.go:198] couldn't find remote ref "refs/heads/master"Failed to git clone: couldn't find remote ref "refs/heads/master"
    multicluster-operators-standalone-subscription E0429 12:38:33.141468       1 git_subscriber_item.go:195] couldn't find remote ref "refs/heads/master"Unable to clone the git repo https://github.com/finastra-engineering/gitops-operating-platform.git
    multicluster-operators-standalone-subscription I0429 12:38:33.141483       1 git_subscriber_item.go:198] exit doSubscription: fusion-operate/fusion-operate
    multicluster-operators-standalone-subscription E0429 12:38:33.141490       1 git_subscriber_item.go:149] couldn't find remote ref "refs/heads/master"Subscription error.
    ```

### Subscription pod failures

```bash
$ kubectl get pods
...
multicluster-operators-hub-subscription-9bf46886b-4kgrl          0/1     CreateContainerError   0          6d6h
...

$ kubectl describe pod
...
Warning  Failed   17m (x15774 over 3d23h)   kubelet  (combined from similar events): Error: error reserving ctr name k8s_multicluster-operators-hub-subscription_multicluster-operators-hub-subscription-9bf46886b-4kgrl_acm_6879d59a-9c35-481c-bad3-fa9dd262c693_1 for id a118fe4608910cb727795039e9f907292cddf8674668537a3c1d0db4c6c17a46: name is reserved
...

$ kubectl get pod <pod_name> -o yaml
...
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2021-03-31T15:34:36Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2021-04-02T21:57:41Z"
    message: 'containers with unready status: [multicluster-operators-hub-subscription]'
    reason: ContainersNotReady
    status: "False"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2021-04-02T21:57:41Z"
    message: 'containers with unready status: [multicluster-operators-hub-subscription]'
    reason: ContainersNotReady
    status: "False"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2021-03-31T15:34:36Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: cri-o://2416db1fc0af9f1e8ca5802e308cac3eba3746c5301d62774a9a8fd889bbcc62
    image: registry.redhat.io/rhacm2/multicluster-operators-subscription-rhel8@sha256:4323ee9b7d1deaa666c93f891cafb48518bf543fa671cb58777572775f813c64
    imageID: registry.redhat.io/rhacm2/multicluster-operators-subscription-rhel8@sha256:4323ee9b7d1deaa666c93f891cafb48518bf543fa671cb58777572775f813c64
    lastState:
      terminated:
        containerID: cri-o://2416db1fc0af9f1e8ca5802e308cac3eba3746c5301d62774a9a8fd889bbcc62
        exitCode: 137
        finishedAt: "2021-04-02T21:57:40Z"
        reason: OOMKilled
        startedAt: "2021-03-31T15:34:40Z"
    name: multicluster-operators-hub-subscription
    ready: false
    restartCount: 0
    started: false
    state:
      waiting:
        message: 'error reserving ctr name k8s_multicluster-operators-hub-subscription_multicluster-operators-hub-subscription-9bf46886b-4kgrl_acm_6879d59a-9c35-481c-bad3-fa9dd262c693_1
          for id 55ffd716cc74425f806cefb1018a61b95fc5022e5fafd6f5a6a81bd6481f3d58:
          name is reserved'
        reason: CreateContainerError
...
```
