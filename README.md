## 1 The environment

This is the starting environment setup where I built the cluster. I used minikube on windows.

To be sure to start on a clean kubernetes cluster I deleted previous minikube env

```
$ minikube delete
* Deleting "minikube" in docker ...
* Deleting container "minikube" ...
* Removing C:\Users\[USER]\.minikube\machines\minikube ...
* Removed all traces of the "minikube" cluster.

$ minikube start
* minikube v1.26.1 on Microsoft Windows 10 Enterprise 10.0.19044 Build 19044
* Automatically selected the docker driver. Other choices: virtualbox, ssh
* Using Docker Desktop driver with root privileges
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
* Creating docker container (CPUs=2, Memory=4000MB) ...

[...]

* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

The following are the OS, minikube an k8s versions

```
$ ver
Microsoft Windows [Versione 10.0.19044.1826]

$ minikube version
minikube version: v1.26.1
commit: 62e108c3dfdec8029a890ad6d8ef96b6461426dc

$ kubectl version --short
Flag --short has been deprecated, and will be removed in the future. The --short output will become the default.
Client Version: v1.24.2
Kustomize Version: v4.5.4
Server Version: v1.24.3
```

# 2 Installing the operator

First thing we need is to download the bundle containing all Custom resources and custom resource definitions needed from the official github [repo](https://github.com/prometheus-operator/prometheus-operator)

```
curl -LOk https://github.com/prometheus-operator/prometheus-operator/raw/main/bundle.yaml
```

Since I want to install the prometheus stack in a specific namespaces I split the bundle.yaml in 2 different file:
- one cotaining all custom resource definitions (the ones with `kind: CustomResourceDefinition`)
     - **AlertmanagerConfig**: defines a namespaced AlertmanagerConfig to be aggregated across multiple namespaces configuring one Alertmanager cluster.
     - **Alertmanager**: describes an Alertmanager cluster
     - **PodMonitor**: defines monitoring for a set of pods
     - **Probe**: defines monitoring for a set of static targets or ingresses
     - **Prometheus**: defines a Prometheus deployment
     - **PrometheusRule**: defines recording and alerting rules for a Prometheus instance
     - **ServiceMonitor**: defines monitoring for a set of services
     - **ThanosRuler**: defines a ThanosRuler deployment
- one containing standard resources needed by the Operator to work porperly (all others)
     - one **ClusterRoleBinding** named prometheus-operator
     - one **ClusterRole** named prometheus-operator
     - one **Deployment** named prometheus-operator
     - one **ServiceAccount** named prometheus-operator
     - one **Service** named prometheus-operator

Then I can create the custom resource definitions (CRDs)

```
$ kubectl apply -f ./setup/CRDs.yaml --server-side
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com serverside-applied
```

> **_TIP:_** Using standard `kubectl apply` could (I always get) result in the following error: `The CustomResourceDefinition "prometheuses.monitoring.coreos.com" is invalid: metadata.annotations: Too long: must have at most 262144 bytes`. The fault is about the annotation `kubectl.kubernetes.io/last-applied-configuration` and there are 2 possible solutions:
> 1. **the imperative way**: use `kubectl create` instead of `kubectl apply`
> 2. **from k8s 1.22 on**: you can still use the declarative apply but specifying the `--server-side` option. More detail on the this [link](https://kubernetes.io/docs/reference/using-api/server-side-apply/)

Just for further confirmation

```
$ kubectl get crds
NAME                                        CREATED AT
alertmanagerconfigs.monitoring.coreos.com   2022-08-19T10:48:11Z
alertmanagers.monitoring.coreos.com         2022-08-19T10:48:11Z
podmonitors.monitoring.coreos.com           2022-08-19T10:48:11Z
probes.monitoring.coreos.com                2022-08-19T10:48:11Z
prometheuses.monitoring.coreos.com          2022-08-19T10:48:13Z
prometheusrules.monitoring.coreos.com       2022-08-19T10:48:13Z
servicemonitors.monitoring.coreos.com       2022-08-19T10:48:14Z
thanosrulers.monitoring.coreos.com          2022-08-19T10:48:15Z
```

Now I create the namespace where I want to put operator's standard resources and custom resources instances 

```
$ kubectl create ns observe
namespace/observe created
```

In all Operator's standard resources update the namespace to the newly created `observe`

At this point it is possible to create the custom resource instances

```
$ kubectl apply -f ./setup/instances.yaml
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
serviceaccount/prometheus-operator created
service/prometheus-operator created
```

Now the Operator is configured and we can start applyingdeploying the actual prometheus cluster by creating the custom resoucrces instances
  
# 2 Deploy the prometheus stack

Before to create the Prometheus instance we need to create some RBAC objects to grant the pods the right access to other cluster's resources:
1. serviceAccount
2. ClusterRole
3. ClusterRoleBinding

The serviceAccount definition

```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: observe

```

and its execution

```

$ kubectl apply -f ./resources/prometheus/prometheus-serviceAccount.yaml
serviceaccount/prometheus created

```

The ClusterRole definition

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]

```

and its execution

```

$ kubectl apply -f ./resources/prometheus/prometheus-clusterRole.yaml
clusterrole.rbac.authorization.k8s.io/prometheus created

```

The ClusterRoleBinding definition

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: observe

```

and its execution

```

$ kubectl apply -f ./resources/prometheus/prometheus-clusterRoleBinding.yaml
clusterrolebinding.rbac.authorization.k8s.io/prometheus created

```

At this point we should have RBAC ready

```

$ kubectl get sa,clusterrole,clusterrolebinding --selector=app=prometheus-stack --all-namespaces
NAMESPACE   NAME                        SECRETS   AGE
observe     serviceaccount/prometheus   0         3m43s

NAMESPACE   NAME                                               CREATED AT
            clusterrole.rbac.authorization.k8s.io/prometheus   2022-08-19T12:49:33Z

NAMESPACE   NAME                                                      ROLE                     AGE
            clusterrolebinding.rbac.authorization.k8s.io/prometheus   ClusterRole/prometheus   2m21s

```

Finally we can define the Prometheus object

```yaml

apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: observe
  labels:
    app: prometheus-stack
    env: dev
spec:
  image: quay.io/prometheus/prometheus:v2.38.0
  nodeSelector:
    kubernetes.io/os: linux
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  portName: http-web # 1 
  podMetadata: # 2
    labels:
      app: prometheus-stack
      env: dev
  scrapeInterval: 30s # 3
  scrapeTimeout: 30s #4
  serviceAccountName: prometheus #5
  version: v2.38.0 #6

```

Here some details about the most interesting fields in the definition

1. **portName**: Port name used for the pods and governing service. This defaults to web
2. **podMetadata**: PodMetadata configures Labels and Annotations which are propagated to the prometheus pods
3. **scrapeInterval**: Interval between consecutive scrapes. Default: `30s`
4. **scrapeTimeout**: Number of seconds to wait for target to respond before erroring
5. **serviceAccountName**: ServiceAccountName is the name of the ServiceAccount to use to run the Prometheus Pods. Pay attention to this because prometheus needs to have access to some resources in the cluster like listing pods or services
6. **version**: Version of Prometheus to be deployed

And execute 

```
$ kubectl apply -f ./resources/prometheus/prometheus-prometheus.yaml
prometheus.monitoring.coreos.com/prometheus created
```

and check 

```

$ kubectl get pods -n observe --selector=app=prometheus-stack
NAME                      READY   STATUS    RESTARTS   AGE
prometheus-prometheus-0   2/2     Running   0          64s
prometheus-prometheus-1   2/2     Running   0          64s

```

> **_Note:_** By default the operator create a service for Prometheus but it creates of type ClusterIP while we need - for semplicity - to expose Prometheus as NodePort so we can use `minikube service` command to access from outside the cluster
> ```
> $ kubectl get svc -n observe
> NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
> prometheus-operated   ClusterIP   None         <none>        9090/TCP   2m20s
> prometheus-operator   ClusterIP   None         <none>        8080/TCP   117m
> ```

Now we are going to define a service to expose the Prometheus cluster

```yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: prometheus-stack
    env: dev
  name: prometheus
  namespace: observe
spec:
  ports:
  - port: 9090
    protocol: TCP
    targetPort: http-web # 1
    name: http-web # 2
  selector:
    app: prometheus-stack
    env: dev
  type: NodePort

```

Some notes

1. **targetPort**: here I used a named port. In particular since it is the `targetPort` must match the `portName` defined in the Prometheus definition
2. **name**: Giving a name to the port is alway suseful even in service definition.


Following its execution

```

$ kubectl apply -f ./resources/prometheus/prometheus-service.yaml
service/prometheus created

```

Finally we can see some tangible result of our work executing

```

$ minikube service -n observe prometheus --url
http://127.0.0.1:[PORT]

```

and then opening the url in a browser

![prometheus!](/img/1.prometheus.jpg)
*Finally Prometheus*

One last thing is missing: something to monitor and since Prometheus itself expose metrics the first thing we can do is configure it to monitor itself.

![prometheus-no-targets!](/img/2.prometheus-no-targets.jpg)
*No target, no metrics*

First of all we create our second custom resource: the ServiceMonitor

```yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-self
  namespace: observe
  labels:
    app: prometheus-stack
    env: dev
spec:
  endpoints: # 1
  - port: http-web # 1
  selector:
    matchLabels:
      operated-prometheus: "true" # 3
  namespaceSelector: # 4
    any: true

```

Following are some details about the most interesting fields

1. **endpoints**: A list of endpoints allowed as part of this ServiceMonitor. Endpoint defines a scrapeable endpoint serving Prometheus metrics. 
2. **selector**: used to select Endpoints objects. Here I used the `matchLabels` options (there is matchExpressions as alternative) that works like any other label selector in kubernetes. The interesting part here is that I defined a label present only in the `prometheus-operated` (the one created by the operator) and not in `prometheus` the one we used for access in the browser. The reason is that this way we can monitor Prometheus by only one service and not both. Using a label present in both would result in 4 targets monitored:
     - 2 for the 2 endpoints handled by `prometheus-operated` service
     - 2 for the 2 endpoints handled by `prometheus`service
3. **namespaceSelector**: used to select which namespaces the Kubernetes Endpoints objects are discovered from. Using `any: true` as we've done allow us to monitor endpoint in different namespaces

Let's do it

```

$ kubectl apply -f ./resources/prometheus/prometheus-serviceMonitor.yaml
servicemonitor.monitoring.coreos.com/prometheus-self created

```

Again 

```

$ minikube service -n observe prometheus --url
http://127.0.0.1:[PORT]

```

and open the browser

![prometheus-targets!](/img/3.prometheus-targets.jpg)
*Finally some targets*

> **_Note:_** Be patient sometimes the operator takes some times (even minutes) to update its configuration

# 3 Deploy Grafana

Fist of all we need to setup some configuration

A persistent volume

```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv
  namespace: observe
  labels:
    type: local
    app: grafana
    env: dev
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: "/mnt/data"

```

and the claim

```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: observe
  labels:
    app: grafana
    env: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

These are neede by grafana itself during execution

```
$ kubectl apply -f ./resources/grafana/grafana-pv.yaml
persistentvolume/grafana-pv created

$ kubectl apply -f ./resources/grafana/grafana-pvc.yaml
persistentvolumeclaim/grafana-pvc created

```

> **_TIP:_** Before to go on while face with PVC I always check that it is bound to the volume
> ```
>$ kubectl get pvc -n observe
>NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
>grafana-pvc   Bound    pvc-e0424987-73f2-48e5-ab8c-1b46ea31bb42   1Gi        RWO            standard       16s
> ```

Then we need to create a couple oof secrets:
- one for configure datasources (like prometheus)
- one to define a minimal grafana.ini

```yaml

apiVersion: v1
kind: Secret
metadata:
  labels:
    app: grafana
    env: dev
  name: grafana-datasources
  namespace: observe
stringData:
  datasources.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
                "access": "proxy",
                "editable": false,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus.observe.svc:9090",
                "version": 1
            }
        ]
    }
type: Opaque

```

```yaml

apiVersion: v1
kind: Secret
metadata:
  labels:
    app: grafana
    env: dev
  name: grafana-config
  namespace: observe
stringData:
  grafana.ini: |
    [date_formats]
    default_timezone = UTC
type: Opaque


```

Execute

```

$ kubectl apply -f ./resources/grafana/grafana-secret-datasources.yaml
secret/grafana-datasources created

$ kubectl apply -f ./resources/grafana/grafana-secret-config.yaml
secret/grafana-config created

```

Just to be sure

```

$ kubectl get secret -n observe --selector=app=grafana
NAME                  TYPE     DATA   AGE
grafana-config        Opaque   1      39s
grafana-datasources   Opaque   1      51s

```

Now we can deploy Grafana with the following definition


```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: grafana
    env: dev
  name: grafana
  namespace: observe
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
      env: dev
  template:
    metadata:
      labels:
        app: grafana
        env: dev
    spec:
      automountServiceAccountToken: false
      containers:
      - image: grafana/grafana:9.0.7
        name: grafana
        ports:
        - containerPort: 3000
          name: http
        readinessProbe:
          httpGet:
            path: /api/health
            port: http
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana-pv
        - mountPath: /etc/grafana/provisioning/datasources
          name: grafana-datasources
          readOnly: false
        - mountPath: /etc/grafana
          name: grafana-config
          readOnly: false
      volumes:
      - name: grafana-datasources
        secret:
          secretName: grafana-datasources
      - name: grafana-config
        secret:
          secretName: grafana-config
      - name: grafana-pv
        persistentVolumeClaim:
          claimName: grafana-pvc

```

> **_Note:_** The deployment is mostly taken from Grafana documentation [Deploy Grafana on Kubernetes](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/). I removed everything it seems to me not strictly necessary to focus on the kubernetes part and not on Grafana itself.

```

$ kubectl apply -f ./resources/grafana/grafana-deployment.yaml
deployment.apps/grafana created

```

We the expose Grafana

```yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: grafana
    env: dev
  name: grafana
  namespace: observe
spec:
  ports:
  - name: http
    port: 3000
    targetPort: http
  selector:
    app: grafana
    env: dev
  type: NodePort

```

execute

```
$ kubectl apply -f ./resources/grafana/grafana-service.yaml
service/grafana created

```

and check

```
$ minikube service grafana -n observe --url
http://127.0.0.1:[PORT]

```

![grafana-login!](/img/4.grafana-login.jpg)
*Grafana login*

> **_Note:_** First time you login in Grafana you have to do that with admin credentials. Their valus can be easily found on Grafana documentation: I don't want to write there because thay can cahnge.

![grafana-prometheus-metrics!](/img/5.grafana-prometheus-metrics.jpg)
*Grafana dashboard shoowing Prometheus metrics*

In this screenshot a lot of interesting things confirm us that we are watching Prometheus darta
1. Prometheus shown in as Data source
2. the promql query `rate(prometheus_http_request_duration_seconds_count{}[5m])` executed on Prometheus metric

# 4 Configure Prometheus for Grafana monitoring

Fisrt of all we need to add to `grafana` service the label we decided must be selected for monitoring `operated-prometheus: "true"`

The service definition became

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: grafana
    env: dev
    operated-prometheus: "true" # Only things added
  name: grafana
  namespace: observe
spec:
  ports:
  - name: http
    port: 3000
    targetPort: http
  selector:
    app: grafana
    env: dev
  type: NodePort
 ```

 ```

$ kubectl apply -f ./resources/grafana/grafana-service.yaml
service/grafana configured

 ```

Then we must create a second ServiceMonitor: this time dedicated to Grafana

```yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: grafana
    env: dev
  name: grafana
  namespace: observe
spec:
  endpoints:
  - interval: 15s
    port: http
  selector:
    matchLabels:
      operated-prometheus: "true"
  namespaceSelector:
    any: true

 ```
Execution

 ```

$ kubectl apply -f ./resources/grafana/grafana-serviceMonitor.yaml
servicemonitor.monitoring.coreos.com/grafana created

 ```

Now we check targets in Prometheus 

```

$ minikube service -n observe prometheus --url
http://127.0.0.1:[PORT]

```

![prometheus-all-targets!](/img/6.prometheus-all-targets.jpg)
*Grafana being scraped by Prometheus*

# 5 Bonus

THe following are some of my favorite tecnique for troubleshooting

## 5.1 Fire and forget pod

```

$ kubectl run -it --rm busybox --image=busybox -n observe -- sh               
If you don't see a command prompt, try pressing enter.
/ # wget -O - http://prometheus.observe.svc:9090 > /dev/null

Connecting to prometheus.observe.svc:9090 (10.100.44.141:9090)

writing to stdout                                                             
-                    100% |******************************|   714  0:00:00 ETA 
written to stdout                                                             
/ #                                                                           
Session ended, resume using 'kubectl attach busybox -c busybox -i -t' command  when the pod is running                                                       
pod "busybox" deleted                                                         

```

It creates a pod but due to the `--rm` flag it is delete after executed (in the case of `sh` after exit the shell)

## 5.2 Impersonification

```
$ kubectl auth can-i list pod --as system:serviceaccount:observe:prometheus
yes

$ kubectl auth can-i delete pod --as system:serviceaccount:observe:prometheus
no
```

It allow you to check if RBAC object are created properly by impersonificate user or even service account and check grants on objects