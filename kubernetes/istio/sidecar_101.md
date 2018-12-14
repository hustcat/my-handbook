# sidecar 101

## 手工注入Sidecar

使用`istioctl kube-inject`可以给原生的`controller configuration`，例如deployment增加`istio-proxy`容器：

```
# istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -

# kubectl get deployment sleep -o wide
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS          IMAGES                                     SELECTOR
sleep   1         1         1            1           68s   sleep,istio-proxy   tutum/curl,docker.io/istio/proxyv2:1.0.3   app=sleep
```

这会在原来[samples/sleep/sleep.yaml](https://github.com/istio/istio/blob/1.0.3/samples/sleep/sleep.yaml)的基础上增加sidecar容器的信息：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  name: sleep
spec:
  replicas: 1
  strategy: {}
  template:
...
    spec:
      containers:
      - command:
        - /bin/sleep
        - infinity
        image: tutum/curl
        imagePullPolicy: IfNotPresent
        name: sleep
        resources: {}
      - args:
        - proxy
        - sidecar
        - --configPath
        - /etc/istio/proxy
        - --binaryPath
        - /usr/local/bin/envoy
        - --serviceCluster
        - sleep
        - --drainDuration
        - 45s
        - --parentShutdownDuration
        - 1m0s
        - --discoveryAddress
        - istio-pilot.istio-system:15007
        - --discoveryRefreshDelay
        - 1s
        - --zipkinAddress
        - zipkin.istio-system:9411
        - --connectTimeout
        - 10s
        - --proxyAdminPort
        - "15000"
        - --controlPlaneAuthPolicy
        - NONE
...
```

sidecar容器：

```
# docker exec -it 4f0484047649 /bin/bash
istio-proxy@sleep-6c57768f87-qgwnl:/$ ps -ef                                                                                                                                                
UID        PID  PPID  C STIME TTY          TIME CMD
istio-p+     1     0  0 12:15 ?        00:00:00 /usr/local/bin/pilot-agent proxy sidecar --configPath /etc/istio/proxy --binaryPath /usr/local/bin/envoy --serviceCluster sleep --drainDurati
istio-p+    16     1  0 12:15 ?        00:00:00 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster sl
istio-p+    34     0  0 12:17 ?        00:00:00 /bin/bash
```

删除deployment:

```
# kubectl delete -f samples/sleep/sleep.yaml
```

## Sidecar的自动注入

使用K8S的[mutating webhook admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)，可以进行 Sidecar 的自动注入。使用这一功能之前首先要检查 kube-apiserver 的进程，是否具备 admission-control 参数，并且这个参数的值中需要包含`MutatingAdmissionWebhook`以及`ValidatingAdmissionWebhook`两项。

```
# kubectl api-versions | grep admissionregistration
admissionregistration.k8s.io/v1beta1
```

* 部署`sidecarInjectorWebhook`

```
# helm template install/kubernetes/helm/istio --name istio --namespace istio-system \
--set security.enabled=true \
--set ingress.enabled=false \
--set gateways.istio-ingressgateway.enabled=false \
--set gateways.istio-egressgateway.enabled=false \
--set galley.enabled=false \
--set sidecarInjectorWebhook.enabled=true \
--set mixer.enabled=false \
--set prometheus.enabled=false \
--set global.proxy.envoyStatsd.enabled=false \
--set pilot.sidecar=false > istio-sidecar.yaml

# kubectl apply -f istio-sidecar.yaml
# kubectl -n istio-system get pods -o wide
NAME                                     READY   STATUS      RESTARTS   AGE     IP              NODE         NOMINATED NODE
istio-citadel-6955bc9cb7-zc29g           1/1     Running     0          5m49s   192.168.9.236   kube-node1   <none>
istio-cleanup-secrets-7696f              0/1     Completed   0          5m49s   192.168.9.233   kube-node1   <none>
istio-pilot-b485b99bf-m9xlh              1/1     Running     0          5m49s   192.168.9.235   kube-node1   <none>
istio-security-post-install-vnj8l        0/1     Completed   0          5m49s   192.168.9.234   kube-node1   <none>
istio-sidecar-injector-99b476b7b-bskm9   1/1     Running     0          5m49s   192.168.9.237   kube-node1   <none>
```

注意`--set security.enabled=true`，否则会报类似下面的错误：

```
MountVolume.SetUp failed for volume "certs" : secret "istio.istio-sidecar-injector-service-account" not found
```

* 前提

(1) 由于在注入sidecar的过程中，`kube-apiserver`需要访问`admission webhook "sidecar-injector.istio.io"`。所以，需要保证`kube-apiserver`能够访问service网络，也就是说需要在master部署`kube-proxy`。否则，在注入过程中会报类似下面的错误：

```
dispatcher.go:72] Failed calling webhook, failing closed sidecar-injector.istio.io: failed calling admission webhook "sidecar-injector.istio.io": Post https://istio-sidecar-injector.istio-system.svc:443/inject?timeout=30s: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I1213 09:25:46.482209   10586 trace.go:76] Trace[1160785741]: "Create /api/v1/namespaces/default/pods" (started: 2018-12-13 09:25:16.480370512 +0000 UTC m=+10510.453149635) (total time: 30.001810914s):
```

(2) k8s需要部署DNS服务。


* 测试

默认情况下，部署`sleep`，只会启动一个容器：

```
# kubectl apply -f samples/sleep/sleep.yaml
# kubectl get deployment -o wide
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
sleep   1         1         1   
# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE
sleep-7549f66447-dsxb5   1/1     Running   0          22s   192.168.9.250   kube-node1   <none>
```

Label the `default` namespace with `istio-injection=enabled`：

```
# kubectl label namespace default istio-injection=enabled
namespace/default labeled
# kubectl get namespace -L istio-injection
NAME           STATUS   AGE     ISTIO-INJECTION
default        Active   6d13h   enabled
istio-system   Active   18h     
kube-public    Active   6d13h   
kube-system    Active   6d13h  
```

然后删除pod:

```
# kubectl delete pods/sleep-7549f66447-dsxb5
pod "sleep-7549f66447-dsxb5" deleted
# kubectl get pods -o wide                               
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE
sleep-7549f66447-qz8rk   2/2     Running   0          37s   192.168.9.253   kube-node1   <none>
```

可以看到，sleep容器被自动注入了一个sidecar容器:

```
# kubectl get pods/sleep-7549f66447-qz8rk -o yaml
...
spec:
  containers:
  - command:
    - /bin/sleep
    - infinity
    image: tutum/curl
    imagePullPolicy: IfNotPresent
    name: sleep
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-2jjcz
      readOnly: true
  - args:
    - proxy
    - sidecar
    - --configPath
    - /etc/istio/proxy
    - --binaryPath
    - /usr/local/bin/envoy
    - --serviceCluster
    - sleep
    - --drainDuration
    - 45s
    - --parentShutdownDuration
    - 1m0s
    - --discoveryAddress
    - istio-pilot.istio-system:15007
    - --discoveryRefreshDelay
    - 1s
    - --zipkinAddress
    - zipkin.istio-system:9411
    - --connectTimeout
    - 10s
    - --proxyAdminPort
    - "15000"
    - --controlPlaneAuthPolicy
    - NONE
...
```

更多相关介绍请参考[Automatic sidecar injection](https://istio.io/docs/setup/kubernetes/sidecar-injection/#automatic-sidecar-injection)。


相关清理操作：

```
# kubectl delete -f samples/sleep/sleep.yaml
# kubectl label namespace default istio-injection-
# kubectl delete -f istio-sidecar.yaml
```

## Refs

* [Installing the sidecar](https://istio.io/docs/setup/kubernetes/sidecar-injection)