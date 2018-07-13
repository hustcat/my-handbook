# 使用API访问configmap


[示例](src/configmap_example.go)

```
# go run configmap.go -kubeconfig=./kube.conf            
key1=value1
key2=2222
```


## Interface

configmap的client接口：

```
// ConfigMapInterface has methods to work with ConfigMap resources.
type ConfigMapInterface interface {
	Create(*v1.ConfigMap) (*v1.ConfigMap, error)
	Update(*v1.ConfigMap) (*v1.ConfigMap, error)
	Delete(name string, options *meta_v1.DeleteOptions) error
	DeleteCollection(options *meta_v1.DeleteOptions, listOptions meta_v1.ListOptions) error
	Get(name string, options meta_v1.GetOptions) (*v1.ConfigMap, error)
	List(opts meta_v1.ListOptions) (*v1.ConfigMapList, error)
	Watch(opts meta_v1.ListOptions) (watch.Interface, error)
	Patch(name string, pt types.PatchType, data []byte, subresources ...string) (result *v1.ConfigMap, err error)
	ConfigMapExpansion
}

// configMaps implements ConfigMapInterface
type configMaps struct {
	client rest.Interface
	ns     string
}
```