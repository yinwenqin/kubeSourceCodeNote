## 前言

APIServer的工作主要围绕着对各类资源对象的管控，因此，在开始阅读APIServer的源码之前，有必要笼统地列举一下它在运行中所用到的核心数据结构等基础性信息，当作是开胃菜篇吧。



## Group/Version/Kind/Resource

在K8s的设计中，resource是其最基础、最重要的概念，也是最小的管理单位，所有的管理对象都承载在一个个的resource实例上，为了实现这些resource的复杂管理逻辑，又进一步地将他们分组化、版本化，依照逻辑层次，形成了Group、Version、Kind、Resource核心数据结构：

- Group：资源组，也称APIGroup，常见的有core、apps、extensions等
- Version：资源版本，也称APIVersion，常见的有v1、v1beta1 (Resource可能属于拥有多个Version，这些version也会有优先级之分，例如deployment即属于apps/v1,又属于extensions/v1beta1，在不同k8s版本中，version的优先级可能会变化)
- Kind：资源种类，描述资源的类别，例如pod类别、svc类别等
- Resource：资源实例对象，也称为APIResource
- SubResource：子资源，部分资源实例会 有子资源，例如Deployment资源会拥有Status子资源
- CRD: Custom Resource Definitions，用户自定义资源类型



### 锚定形式

概念层面，在K8s中，常见的资源路径锚定形式为：<Group>/<Version>/<Resource>/<SubResource>，例如deployment对应的路径是：apps/v1/deployments/status

官方通常通过缩写词**GVR**(GroupVersionKind)来描述一个资源的明确锚定位置(类似于绝对路径？)，同理，**GVK**(GroupVersionKind)锚定资源的明确所属类型，在项目代码中也经常用到，例如：

`vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go:96`

```go
// GroupVersionResource unambiguously identifies a resource.  It doesn't anonymously include GroupVersion
// to avoid automatic coercion.  It doesn't use a GroupVersion to avoid custom marshalling
type GroupVersionResource struct {
	Group    string
	Version  string
	Resource string
}
```





### 资源结构体

而落实到代码中，每一种资源的结构体定义文件都位于其Group下的的types.go文件中，例如，Deployment资源的结构体定义在这里：

`pkg/apis/apps/types.go:268`

```go
type Deployment struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus
}

```

`metav1.TypeMeta`

```go
type TypeMeta struct {
	// Kind is a string value representing the REST resource this object represents.
	// Servers may infer this from the endpoint the client submits requests to.
	// Cannot be updated.
	// In CamelCase.
	// More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds
	// +optional
	Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

	// APIVersion defines the versioned schema of this representation of an object.
	// Servers should convert recognized schemas to the latest internal value, and
	// may reject unrecognized values.
	// More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#resources
	// +optional
	APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
```

`metav1.ObjectMeta`

```go
type ObjectMeta struct {

	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`

	GenerateName string `json:"generateName,omitempty" protobuf:"bytes,2,opt,name=generateName"`

	Namespace string `json:"namespace,omitempty" protobuf:"bytes,3,opt,name=namespace"`

	SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,4,opt,name=selfLink"`

	UID types.UID `json:"uid,omitempty" protobuf:"bytes,5,opt,name=uid,casttype=k8s.io/kubernetes/pkg/types.UID"`

	ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,6,opt,name=resourceVersion"`

	Generation int64 `json:"generation,omitempty" protobuf:"varint,7,opt,name=generation"`

	CreationTimestamp Time `json:"creationTimestamp,omitempty" protobuf:"bytes,8,opt,name=creationTimestamp"`

	DeletionTimestamp *Time `json:"deletionTimestamp,omitempty" protobuf:"bytes,9,opt,name=deletionTimestamp"`

	DeletionGracePeriodSeconds *int64 `json:"deletionGracePeriodSeconds,omitempty" protobuf:"varint,10,opt,name=deletionGracePeriodSeconds"`

	Labels map[string]string `json:"labels,omitempty" protobuf:"bytes,11,rep,name=labels"`

	Annotations map[string]string `json:"annotations,omitempty" protobuf:"bytes,12,rep,name=annotations"`

	OwnerReferences []OwnerReference `json:"ownerReferences,omitempty" patchStrategy:"merge" patchMergeKey:"uid" protobuf:"bytes,13,rep,name=ownerReferences"`

	Initializers *Initializers `json:"initializers,omitempty" protobuf:"bytes,16,opt,name=initializers"`

	Finalizers []string `json:"finalizers,omitempty" patchStrategy:"merge" protobuf:"bytes,14,rep,name=finalizers"`

	ClusterName string `json:"clusterName,omitempty" protobuf:"bytes,15,opt,name=clusterName"`

	ManagedFields []ManagedFieldsEntry `json:"managedFields,omitempty" protobuf:"bytes,17,rep,name=managedFields"`
}
```



### 资源操作方法

概念层面，每种resource都有对应的管理操作方法，目前支持的有这几种：

- get
- list
- create
- update
- patch
- delete
- deletecolletction
- watch

分别归属于 增、删、改、查四类。

落实到代码里，资源对应的操作方法，都在metav1.Verbs结构体中归纳：

`vendor/k8s.io/apimachinery/pkg/apis/meta/v1/types.go:970`

```go
// APIResource specifies the name of a resource and whether it is namespaced.
type APIResource struct {
  ...
	// verbs is a list of supported kube verbs (this includes get, list, watch, create,
	// update, patch, delete, deletecollection, and proxy)
	Verbs Verbs `json:"verbs" protobuf:"bytes,4,opt,name=verbs"`
  ...
}

type Verbs []string

func (vs Verbs) String() string {
	return fmt.Sprintf("%v", []string(vs))
}
```

使用[]string结构来描述资源所对应的操作，而[]string终归只是描述，需要与实际的存储资源CRUD操作关联，因此，不难猜测，每种string描述的方法会map到具体的方法上去，结构类似于: map[string]Function，这里不继续展开，后面的章节中会提及。



### 内部和外部Version

在k8s的设计中，资源版本分外部版本(external)和内部版本(internal)之分，外部版本(例如v1/v1beta1/v1beta2)提供给外部使用，而对应的内部版本仅在APIServer内部使用。

区分内外版本的作用：

- 提供不同版本之间的转换功能，例如从v1beta1-->v1的过程实际是v1beta1--> internal -->v1，转换函数会注册到scheme表中

- 减少复杂度，方便版本维护，避免维护多个版本的对应代码，实际APIServer端处理的都是转换后的内部版本
- 不同外部版本资源之间的字段/功能可能存在些许差异，而内部版本包含所有版本的字段/功能，这为它作为外部资源版本之间转换的桥梁提供了基础。



内部版本和外部版本对于资源结构体定义显著的区别是，内部版本是不带json和proto标签的，因为其不需要结构化提供给外部。以Deployment资源为例

内部版本的代码路径为：`pkg/apis/apps/types.go:268`

```go
type Deployment struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus
}

```

外部版本代码路径：`vendor/k8s.io/api/apps/v1/types.go:254`

```go
type Deployment struct {
   metav1.TypeMeta `json:",inline"`
   // Standard object metadata.
   // +optional
   metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

   // Specification of the desired behavior of the Deployment.
   // +optional
   Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

   // Most recently observed status of the Deployment.
   // +optional
   Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

以向APIServer发起创建一个资源实例为例，在解码时，APIServer首先从 HTTP 路径中获取对应的外部version，然后使用 scheme 以外部version创建一个空对象，然后将该对象转换成内部版本对应的对象结构进行持久存储(写入etcd).

而在查询请求中，资源对象会被从内部版本转换为路径对应的外部版本，响应给请求端。

关于内外部版本的说明，这篇文章讲得很详细，可参考：

https://www.kubernetes.org.cn/6839.html



## Scheme注册表

每一种Resource都有对应的Kind，为了更便于分类管理这些资源，APIServer设计了一种名为scheme的结构体，类似于注册表，运行时数据存放内存中，提供给各种资源进行注册，scheme有如下作用：

- 提供资源的版本转换功能
- 提供资源的序列化/反序列化功能

Scheme支持注册两种类型的资源：

- UnversionedType 无版本资源。这个在现版本的k8s中使用非常少，可以忽略

- VersionedType 几乎所有的资源都是携带版本的，是常用的类型

  

### scheme结构体

 `vendor/k8s.io/apimachinery/pkg/runtime/scheme.go:47`

```go
type Scheme struct {
	// versionMap allows one to figure out the go type of an object with
	// the given version and name.
	gvkToType map[schema.GroupVersionKind]reflect.Type

	// typeToGroupVersion allows one to find metadata for a given go object.
	// The reflect.Type we index by should *not* be a pointer.
	typeToGVK map[reflect.Type][]schema.GroupVersionKind

	// unversionedTypes are transformed without conversion in ConvertToVersion.
	unversionedTypes map[reflect.Type]schema.GroupVersionKind

	// unversionedKinds are the names of kinds that can be created in the context of any group
	// or version
	// TODO: resolve the status of unversioned types.
	unversionedKinds map[string]reflect.Type

	// Map from version and resource to the corresponding func to convert
	// resource field labels in that version to internal version.
	fieldLabelConversionFuncs map[schema.GroupVersionKind]FieldLabelConversionFunc

	// defaulterFuncs is an array of interfaces to be called with an object to provide defaulting
	// the provided object must be a pointer.
	defaulterFuncs map[reflect.Type]func(interface{})

	// converter stores all registered conversion functions. It also has
	// default converting behavior.
	converter *conversion.Converter

	// versionPriority is a map of groups to ordered lists of versions for those groups indicating the
	// default priorities of these versions as registered in the scheme
	versionPriority map[string][]string

	// observedVersions keeps track of the order we've seen versions during type registration
	observedVersions []schema.GroupVersion

	// schemeName is the name of this scheme.  If you don't specify a name, the stack of the NewScheme caller will be used.
	// This is useful for error reporting to indicate the origin of the scheme.
	schemeName string
}
```

重点关注字段：

- gvkToType ：用map结构存储gvk和Type的映射关系，一个gvk只会具体对应一个Type
- typeToGVK：用map结构存储type和gvk的映射关系，不同的是，一个type可能会对应多个gvk
- converter：map结构，存放资源版本转换的方法



### 注册方法

scheme表提供两个注册方法：`AddKnownTypes` | `AddKnownTypeWithName` ，使用reflect反射的方式获取type obj的gvk然后进行注册。

`vendor/k8s.io/apimachinery/pkg/runtime/scheme.go:172`

```go
func (s *Scheme) AddKnownTypes(gv schema.GroupVersion, types ...Object) {
	s.addObservedVersion(gv)
	for _, obj := range types {
		t := reflect.TypeOf(obj)
		if t.Kind() != reflect.Ptr {
			panic("All types must be pointers to structs.")
		}
		t = t.Elem()
		s.AddKnownTypeWithName(gv.WithKind(t.Name()), obj)
	}
}
```

`vendor/k8s.io/apimachinery/pkg/runtime/scheme.go:188`

```go
func (s *Scheme) AddKnownTypeWithName(gvk schema.GroupVersionKind, obj Object) {
	s.addObservedVersion(gvk.GroupVersion())
	t := reflect.TypeOf(obj)
	if len(gvk.Version) == 0 {
		panic(fmt.Sprintf("version is required on all types: %s %v", gvk, t))
	}
	if t.Kind() != reflect.Ptr {
		panic("All types must be pointers to structs.")
	}
	t = t.Elem()
	if t.Kind() != reflect.Struct {
		panic("All types must be pointers to structs.")
	}

	if oldT, found := s.gvkToType[gvk]; found && oldT != t {
		panic(fmt.Sprintf("Double registration of different types for %v: old=%v.%v, new=%v.%v in scheme %q", gvk, oldT.PkgPath(), oldT.Name(), t.PkgPath(), t.Name(), s.schemeName))
	}

	s.gvkToType[gvk] = t

	for _, existingGvk := range s.typeToGVK[t] {
		if existingGvk == gvk {
			return
		}
	}
	s.typeToGVK[t] = append(s.typeToGVK[t], gvk)
}
```



## runtime.Object 

Runtime库被称为运行时，几乎是所有程序的核心库，在k8s中也不例外，k8s的运行时实现在`vendor/k8s.io/apimachinery/pkg/runtime/`路径下。

其中，runtime.Object是K8s中所有资源类型结构的基石，作为interface被封装，所有的资源对象均以runtime.Object为基础结构实现了相应的interface方法，runtime.Object是一种通用的struct结构体，资源对象可以与runtime.Object通用对象互相转换。runtime.Object结构如下：

`vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go:231`

```go
// Object interface must be supported by all API types registered with Scheme. Since objects in a scheme are
// expected to be serialized to the wire, the interface an Object must provide to the Scheme allows
// serializers to set the kind, version, and group the object is represented as. An Object may choose
// to return a no-op ObjectKindAccessor in cases where it is not expected to be serialized.
type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}
```

`vendor/k8s.io/apimachinery/pkg/runtime/schema/interfaces.go:22`

```go
// All objects that are serialized from a Scheme encode their type information. This interface is used
// by serialization to set type information from the Scheme onto the serialized version of an object.
// For objects that cannot be serialized or have unique requirements, this interface may be a no-op.
type ObjectKind interface {
	// SetGroupVersionKind sets or clears the intended serialized kind of an object. Passing kind nil
	// should clear the current setting.
	SetGroupVersionKind(kind GroupVersionKind)
	// GroupVersionKind returns the stored group, version, and kind of an object, or nil if the object does
	// not expose or provide these fields.
	GroupVersionKind() GroupVersionKind
}
```

- schema.ObjectKind提供两个方法，分别用于设置和获取对象的gvk
- DeepCopyObject()则用于深复制对象，这个方法会被经常用到，尤其是在写操作的场景中，避免直接操作对象本身。



## 序列化与反序列化

APIServer对资源的描述支持yaml和json格式，分别对应不同的Serializer，Serializer内置有bool类型的yaml字段，来辨别是否是yaml Serializer。

`vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go:60`

```go
type Serializer struct {
	meta    MetaFactory
	creater runtime.ObjectCreater
	typer   runtime.ObjectTyper
	yaml    bool
	pretty  bool
}
```



序列化代码位于：

`vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go:223`

```go
// Encode serializes the provided object to the given writer.
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
	if s.yaml {
    
		json, err := caseSensitiveJsonIterator.Marshal(obj)
		if err != nil {
			return err
		}
		data, err := yaml.JSONToYAML(json)
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		return err
	}

	if s.pretty {
		data, err := caseSensitiveJsonIterator.MarshalIndent(obj, "", "  ")
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		return err
	}
	encoder := json.NewEncoder(w)
	return encoder.Encode(obj)
}

```

可以得知，默认以json格式响应，而对于yaml格式，先将其转换为json格式，再转换回yaml格式响应

反序列化：

`vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go:86`

```go
func (customNumberDecoder) Decode(ptr unsafe.Pointer, iter *jsoniter.Iterator) {
	switch iter.WhatIsNext() {
	case jsoniter.NumberValue:
		var number jsoniter.Number
		iter.ReadVal(&number)
		i64, err := strconv.ParseInt(string(number), 10, 64)
		if err == nil {
			*(*interface{})(ptr) = i64
			return
		}
		f64, err := strconv.ParseFloat(string(number), 64)
		if err == nil {
			*(*interface{})(ptr) = f64
			return
		}
		iter.ReportError("DecodeNumber", err.Error())
	default:
		*(*interface{})(ptr) = iter.Read()
	}
}
```



## go-restful

k8s选用的Restful框架是go-restful，简单说明一下go-restful的结构，辅助后面对于APIServer工作流程的理解。

go-restful层级结构概念自顶上下依次有:

- Container: 一个Container就是一个独立的http server，可拥有独立的地址端口组合(类似nginx的server层级)

- WebService： 大粒度的分类，某一类别的服务可归属到同一个WebService中，其下包含多个Route
- Route: 每个Route对应具体的uri路径，将该路径路由到对应的handler函数上

