# Auto-Instrumentation not working with resource quota defined in K8s env namespace

## Requirements
K8s cluster [How to Install Kubernetes Cluster on Ubuntu 24.04 LTS](https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99)

## Installation steps

1. Install the Tomcat app using helm chart
```
helm install --namespace tomcat --create-namespace tomcat --set persistence.enabled=false --set resources.limits.cpu=1 --set resources.limits.memory=512Mi --set resources.requests.cpu=0.5 --set resources.requests.memory=256Mi bitnami/tomcat
```

2. Set the external IP address of the tomcat service
```
kubectl patch svc tomcat -n tomcat -p '{"spec": {"type": "LoadBalancer", "externalIPs":["10.202.12.167"]}}'
```
*replace 10.202.12.167 with your own public ip address*
  
4. Create a resource quota in the tomcat namespace
```
kubectl create -f resourcequota.yaml -n tomcat

apiVersion: v1
kind: ResourceQuota
metadata:
 name: cpu-mem-quota
spec:
 hard:
  cpu: "2"
  memory: 2Gi
```

4. Install the Otel collector
```
helm install splunk-otel-collector --set="splunkObservability.accessToken=***,clusterName=mycluster1,splunkObservability.realm=us1,gateway.enabled=false,splunkObservability.profilingEnabled=true,environment=lab,operator.enabled=true,certmanager.enabled=true,agent.discovery.enabled=true" splunk-otel-collector-chart/splunk-otel-collector --namespace splunk-otel --create-namespace
```
*replace *** with your o11y INGEST token*

5. Add the annotation to the tomcat deployment so the autoinstrumentation injects the java agent into the container
```
kubectl patch deployment tomcat -n tomcat -p '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"splunk-otel/splunk-otel-collector"}}}}}'
```

6. Nothing happens after that because the replicaset blocks pod creation
```
kubectl events -n tomcat
80s                 Warning   FailedCreate        ReplicaSet/tomcat-6c6588c664   Error creating: pods "tomcat-6c6588c664-c2h45" is forbidden: failed quota: cpu-mem-quota: must specify cpu for: opentelemetry-auto-instrumentation-java; memory for: opentelemetry-auto-instrumentation-java
``` 

7. If you remove the quota and delete the tomcat pod, a new pod will be created but without resource requests and limits within the init-container
```
kubectl delete quota cpu-mem-quota -n tomcat
kubectl delete pod tomcat-6bc756cc68-97rhh -n tomcat
kubectl describe pod tomcat-6bc756cc68-9xg4m -n tomcat

Init Containers:
  opentelemetry-auto-instrumentation-java:
    Container ID:    containerd://a14b5c5f87a813e616b2d3668863094b584896fbd03653aa65fc7fb7f6258163
    Image:           ghcr.io/signalfx/splunk-otel-java/splunk-otel-java:v1.32.2
    Image ID:        ghcr.io/signalfx/splunk-otel-java/splunk-otel-java@sha256:13fda11e64ec200aa77a63a6d9ca40b2c4ea0d0ad7e87bab99cc4a3e5638df76
    Port:            <none>
    Host Port:       <none>
    SeccompProfile:  RuntimeDefault
    Command:
      cp
      /javaagent.jar
      /otel-auto-instrumentation-java/javaagent.jar
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 21 Aug 2024 19:09:00 +0000
      Finished:     Wed, 21 Aug 2024 19:09:00 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /otel-auto-instrumentation-java from opentelemetry-auto-instrumentation-java (rw)
```

## Workaround

1. Pass resource limits and requests through values.yaml for the operator
```
helm upgrade splunk-otel-collector --values resources.yaml --set="splunkObservability.accessToken=***,clusterName=mycluster1,splunkObservability.realm=us1,gateway.enabled=false,splunkObservability.profilingEnabled=true,environment=lab,operator.enabled=true,certmanager.enabled=true,agent.discovery.enabled=true" splunk-otel-collector-chart/splunk-otel-collector --namespace splunk-otel --create-namespace

operator:
  enabled: true
  instrumentation:
    spec:
      java:
        repository: ghcr.io/signalfx/splunk-otel-java/splunk-otel-java
        tag: v1.32.2
        resources:
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
```
*replace *** with your o11y INGEST token*
[Operator API docs](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api.md)

2. Delete the current tomcat pod and wait for the creation of a new one
```
kubectl delete pod tomcat-6c6588c664-ph59k -n tomcat
```

3. Check the init-container of the pod recently created (now, init-container contains resource limits and requests)
```
Init Containers:
  opentelemetry-auto-instrumentation-java:
    Container ID:    containerd://729360796255d6866db2e0721952bfae0a116d735a8a0566fd7f9091a8ffeeb6
    Image:           ghcr.io/signalfx/splunk-otel-java/splunk-otel-java:v1.32.2
    Image ID:        ghcr.io/signalfx/splunk-otel-java/splunk-otel-java@sha256:13fda11e64ec200aa77a63a6d9ca40b2c4ea0d0ad7e87bab99cc4a3e5638df76
    Port:            <none>
    Host Port:       <none>
    SeccompProfile:  RuntimeDefault
    Command:
      cp
      /javaagent.jar
      /otel-auto-instrumentation-java/javaagent.jar
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 21 Aug 2024 19:21:27 +0000
      Finished:     Wed, 21 Aug 2024 19:21:27 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  64Mi
    Requests:
      cpu:        50m
      memory:     64Mi
    Environment:  <none>
    Mounts:
      /otel-auto-instrumentation-java from opentelemetry-auto-instrumentation-java (rw)
```

### Notes

This issue is fixed in our upstream collector. Link is here: (https://github.com/open-telemetry/opentelemetry-operator/issues/895)

