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

## PriorityClassName => Priority

`admission controller`会根据`Pod.Spec.PriorityClassName`字段计算`Pod.Spec.Priority
`的值，该值越大，Pod的优先级越高：

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

相关实现参考`plugin/pkg/admission/priority/admission.go`:

```
// admitPod makes sure a new pod does not set spec.Priority field. It also makes sure that the PriorityClassName exists if it is provided and resolves the pod priority from the PriorityClassName.
// Note that pod validation mechanism prevents update of a pod priority.
func (p *priorityPlugin) admitPod(a admission.Attributes) error {
	operation := a.GetOperation()
	pod, ok := a.GetObject().(*api.Pod)
	if !ok {
		return errors.NewBadRequest("resource was marked with kind Pod but was unable to be converted")
	}

	// Make sure that the client has not set `priority` at the time of pod creation.
	if operation == admission.Create && pod.Spec.Priority != nil {
		return admission.NewForbidden(a, fmt.Errorf("the integer value of priority must not be provided in pod spec. Priority admission controller populates the value from the given PriorityClass name"))
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.PodPriority) {
		var priority int32
		if len(pod.Spec.PriorityClassName) == 0 {
			var err error
			priority, err = p.getDefaultPriority()
			if err != nil {
				return fmt.Errorf("failed to get default priority class: %v", err)
			}
		} else {
			// First try to resolve by system priority classes.
			priority, ok = SystemPriorityClasses[pod.Spec.PriorityClassName]
			if !ok {
				// Now that we didn't find any system priority, try resolving by user defined priority classes.
				pc, err := p.lister.Get(pod.Spec.PriorityClassName)
				if err != nil {
					return fmt.Errorf("failed to get default priority class %s: %v", pod.Spec.PriorityClassName, err)
				}
				if pc == nil {
					return admission.NewForbidden(a, fmt.Errorf("no PriorityClass with name %v was found", pod.Spec.PriorityClassName))
				}
				priority = pc.Value
			}
		}
		pod.Spec.Priority = &priority
	}
	return nil
}
```

## Refs

* [Pod Priority and Preemption](https://v1-9.docs.kubernetes.io/docs/concepts/configuration/pod-priority-preemption)
* [Pod Preemption in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/pod-preemption.md)