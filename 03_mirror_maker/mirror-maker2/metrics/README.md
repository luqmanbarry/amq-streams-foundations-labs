# SETTING UP KAFKA EXPORTER FOR CONSUMER LAG MONITORING
***
The metrics data is used, for example, to help identify slow consumers.

Kafka Exporter exposes metrics data for brokers, topics and consumer groups.

## Configuring Kafka Exporter

1. Edit the KafkaExporter properties for the Kafka resource under spec:

> ***Note:** 
> The resource.requests and resources.limits values can be tailored to cluster requirements
> We may want to observe the cluster for a while before we decide to change default values
           
```
kafkaExporter:
    groupRegex: ".*" 
    topicRegex: ".*" 
    resources: 
      requests:
        cpu: 200m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 128Mi
    logging: debug 
    enableSaramaLogging: true 
    template: 
      pod:
        securityContext:
          runAsUser: 1000001
          fsGroup: 0
        terminationGracePeriodSeconds: 120
    readinessProbe: 
      initialDelaySeconds: 15
      timeoutSeconds: 5
    livenessProbe: 
      initialDelaySeconds: 15
      timeoutSeconds: 5
```

2. Apply changes to Kafka cluster
```
oc apply -f your-kafka-file
```


> ### Reducing consumer lag
> Typical actions to reduce lag include:
> Scaling-up consumer groups by adding new consumers
> Increasing the retention time for a message to remain in a topic
> Adding more disk capacity to increase the message buffer


***
# SETTING UP CRUISE CONTROL
***

## Enable cruise control on CRD of kind Kafka

    Add cruiseControl config under **spec:**

```
cruiseControl:
    brokerCapacity:
      disk: 100Gi
      cpuUtilization: 100
      inboundNetwork: 10000KB/s
      outboundNetwork: 10000KB/s
    resources:
      requests:
        cpu: 200m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 128Mi
    logging:
      type: inline
      loggers:
        rootLogger.level: "INFO"
    readinessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
```

## Verify that Cruise Control was successfully deployed:

    ```
    oc get deployments -l app.kubernetes.io/name=cruise-control
    ```

    Following topics should exist and not be deleted:

    * strimzi.cruisecontrol.metrics
    * strimzi.cruisecontrol.partitionmetricsamples
    * strimzi.cruisecontrol.modeltrainingsamples

## Generate Optimization Proposals

1. Create a KafkaRebalance resource:
```
    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaRebalance
    metadata:
    name: my-rebalance
    labels:
        strimzi.io/cluster: <CLUSTER_NAME>
    spec: {}
```
    or look into amq_install_examples/examples/cruise-control/ for more customization

2. Configure user-provided optimization goals instead
```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
name: my-rebalance
labels:
    strimzi.io/cluster: <CLUSTER_NAME>
spec:
goals:
    - CpuCapacityGoal
    - NetworkInboundCapacityGoal
    - DiskCapacityGoal
    - RackAwareGoal
    - MinTopicLeadersPerBrokerGoal
    - NetworkOutboundCapacityGoal
    - ReplicaCapacityGoal
```
3. Create or update the resource
```
oc apply -f your-file
```
4. Review the optimization proposal
```
oc describe kafkarebalance rebalance-cr-name
``` 
The properties in the Optimization Result section describe the pending cluster rebalance operation.

[Click here for more](https://access.redhat.com/documentation/en-us/red_hat_amq/2021.q3/html-single/using_amq_streams_on_openshift/index#contents-optimization-proposals)

## Approving an optimization proposal

> ### Caution
> This is not a dry run. Before you approve an optimization proposal, you must:
> * Refresh the proposal in case it has become out of date.
> * Carefully review the contents of the proposal.

1. Refresh to pull latest optimization goals metrics
```
oc annotate kafkarebalance rebalance-cr-name strimzi.io/rebalance=refresh
```

1. Check the status of the KafkaRebalance resource -- Pause and Repeat until you status changes to `ProposalReady`

2. Approve the optimization proposal that you want Cruise Control to apply
```
oc annotate kafkarebalance rebalance-cr-name strimzi.io/rebalance=approve
```
5. Cruise Control returns one of three statuses:

 * Rebalancing: The cluster rebalance operation is in progress.
 * Ready: The cluster rebalancing operation completed successfully. To use the same `KafkaRebalance` custom resource to   generate another optimization proposal, apply the `refresh` annotation to the custom resource. This moves the custom resource to the `PendingProposal` or `ProposalReady` state. You can then review the optimization proposal and approve it, if desired.
 * NotReady: An error occurred—​see Section 8.10, [Fixing problems with a KafkaRebalance resource](https://access.redhat.com/documentation/en-us/red_hat_amq/2021.q3/html-single/using_amq_streams_on_openshift/index#proc-fixing-problems-with-kafkarebalance-str).

## Stopping a cluster rebalance
> Once started, a cluster rebalance operation might take some time to complete 
> and affect the overall performance of the Kafka cluster.

1. Annotate the KafkaRebalance resource in OpenShift
```
oc annotate kafkarebalance rebalance-cr-name strimzi.io/rebalance=stop
```
2. Check the status of the KafkaRebalance resource -- Pause and retry until status changes to `Stopped`
```
oc describe kafkarebalance rebalance-cr-name
```
    