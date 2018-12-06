# 优先级与抢占(Priority and Preempt)


## Enabling priority and preemption

kube-apiserver/kube-scheduler/kubelet增加启动参数`--feature-gates=PodPriority=true`。

kube-apiserver增加启动参数`--runtime-config=scheduling.k8s.io/v1alpha1=true --admission-control=Controller-Foo,Controller-Bar,...,Priority`。


## 测试preempt

创建`PriorityClass`:

```
apiVersion: scheduling.k8s.io/v1alpha1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority pods"
```


```
# kubectl describe node/kube-node1
...
Capacity:
 cpu:     4
 memory:  4046588Ki
 pods:    110
Allocatable:
 cpu:     4
 memory:  3944188Ki
 pods:    110
...
```

创建一个低先级的Pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-1
spec:
  containers:
  - name: nginx-1
    image: nginx:latest
    imagePullPolicy: Never
    resources:
      limits:
        cpu: "4"
      requests:
        cpu: "4"
```
使用完`kube-node1`的所有CPU。

再创建一个高优先级的Pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-high
spec:
  containers:
  - name: nginx-high
    image: nginx:latest
    imagePullPolicy: Never
    resources:
      limits:
        cpu: "2"
      requests:
        cpu: "2"
  priorityClassName: high-priority
```

这会导致`nginx-1`被删除。


`kube-scheduler`的日志：
```
scheduler.go:301] Attempting to schedule pod: default/nginx-high
scheduler.go:176] Failed to schedule pod: default/nginx-high
factory.go:1062] Updating pod condition for default/nginx-high to (PodScheduled==False)
event.go:218] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-high", UID:"6be41680-f920-11e8-81bf-525460110101", APIVersion:"v1", ResourceVersion:"150681", FieldPath:""}): type: 'Warning' reason: 'FailedScheduling' No nodes are available that match all of the predicates: Insufficient cpu (1).
generic_scheduler.go:742] Node kube-node1 is a potential node for preemption.
scheduler.go:209] Preempting 1 pod(s) on node kube-node1 to make room for default/nginx-high.
```

`admission controller`会根据`Pod.Spec.PriorityClassName`字段计算`Pod.Spec.Priority`的值，该值越大，Pod的优先级越高：

```
// PodSpec is a description of a pod.
type PodSpec struct {
...
	// If specified, indicates the pod's priority. "system-node-critical" and
	// "system-cluster-critical" are two special keywords which indicate the
	// highest priorities with the former being the highest priority. Any other
	// name must be defined by creating a PriorityClass object with that name.
	// If not specified, the pod priority will be default or zero if there is no
	// default.
	// +optional
	PriorityClassName string `json:"priorityClassName,omitempty" protobuf:"bytes,24,opt,name=priorityClassName"`
	// The priority value. Various system components use this field to find the
	// priority of the pod. When Priority Admission Controller is enabled, it
	// prevents users from setting this field. The admission controller populates
	// this field from PriorityClassName.
	// The higher the value, the higher the priority.
	// +optional
	Priority *int32 `json:"priority,omitempty" protobuf:"bytes,25,opt,name=priority"`
```

```
# kubectl get pods/nginx-high -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    NominatedNodeName: kube-node1
...
spec:
  nodeName: kube-node1
  priority: 1000000
...
```

## Refs

* [Pod Priority and Preemption](https://v1-9.docs.kubernetes.io/docs/concepts/configuration/pod-priority-preemption)
* [Pod Preemption in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/pod-preemption.md)