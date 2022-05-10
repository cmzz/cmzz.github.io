---
title: 深入分析 MinIO Gateway 存储网关
tags: 
- MinIO
- 存储
date: 2022-05-10 08:00:00
---

业务系统因文件存储需求，需新增通用文件服务，除公有云上使用，还要满足私用化场景，几番对比之最终选择基于 MinIO 构建，一来在公有上将其用作存储网关，在私有化环境中直接用作对象存储服务。

MinIO 是一个基于Apache License v2.0开源协议的对象存储服务。兼容 AWS S3 like API，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，最大支持5T。

MinIO 除了是对象存储服务外，它还内置了一个存对网关 MinIO Gateway，后端支持多种 S3 like 类型的存储系统，像 S3、NAS、HDFS、Google Cloud对象存储等。

由于 MinIO 存储网关的存在，使用系统具备较好的兼容性和可移植性。在必要时，可以非常方便的从 S3 迁移到 Google Cloud  的对象存储，而不用调整系统，甚至是使用在后端同时使用多个厂商存储服务实现多云混合，再或者部署多个网关实现分布式以提供更为强大的并发能力。

MinIO 支持格式众多的云存储服务，不过在支持的产品列表中却没有我们常用的阿里云 OSS 或是腾讯云的储存服务，不得不说多少有些遗憾。

目前 MinIO 官网的支持清单：
- Azure
- GCS
- S3
- HDFS

## MinIO Alibaba OSS
MinIO 在早期曾集成过 Alibaba OSS 的代理理服务，后因 Alibaba OSS 官方SDK 的 License 问题被 MinIO 移除。

时隔一年，虽 License 修复但重新申请合并的请求被拒绝了，此后 MinIO 官方也没有再支持国内主流厂商的计划。具体细节可查看 MinIO issue 中的相关讨论。

## MinIO 网关
MinIO 作为网关，主要有以下几个的功能。
首先，MinIO 网关能够屏蔽后台各存储产品的差异，便客户端提供统一的接口，使用 MinIO Clinent 即可在多种云之间切换。
其次，MinIO 网关能够向后端的云存储产品增加 MinIO 独有的一些功能，比如磁盘缓存、资源浏览器功能。
再次，通过部署多个网关，可实现分布的存储存架构，提升程序的可用性。

前面提到在现行的版本中，只是内置少量几个产品的Gateway，若要使用其他 OSS 或其或一些非 S3 like 的产品需要动手去扩展。

### Gateway的设计
Gateway 分为 GatewayLayer 和 ObjectLayer 及  Credential 2层。 GatewayLayer 包含网关名称和认证信息，而ObjectLayer 则是各对象存储的一个抽象。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h22cnt3q27j216q0i80tq.jpg)
￼￼
整个 MinIO Gateway 模块呈现一种分层的架构，如下图

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h22d5idcx6j20u010n76j.jpg)

#### 接口
网关的接口其实比较简单的，就 2 个方法，获取名称和实例化 Object Layer.

```go
// Gateway represents a gateway backend.
type Gateway interface {
	// Name returns the unique name of the gateway.
	Name() string

	// NewGatewayLayer returns a new  ObjectLayer.
	NewGatewayLayer(creds madmin.Credentials) (ObjectLayer, error)
}
```

ObjectLayer 接口由于是对象存储产品的抽象层，所以方法比较多，涵盖了 S3 like 的 Bucket 和 Object 的所有操作。

```go
// ObjectLayer implements primitives for object API layer.
type ObjectLayer interface {
	// Locking operations on object.
	NewNSLock(bucket string, objects ...string) RWLocker

	// Storage operations.
	Shutdown(context.Context) error
	NSScanner(ctx context.Context, bf *bloomFilter, updates chan<- DataUsageInfo, wantCycle uint32, scanMode madmin.HealScanMode) error
	BackendInfo() madmin.BackendInfo
	StorageInfo(ctx context.Context) (StorageInfo, []error)
	LocalStorageInfo(ctx context.Context) (StorageInfo, []error)

	// Bucket operations.
	MakeBucketWithLocation(ctx context.Context, bucket string, opts BucketOptions) error
	GetBucketInfo(ctx context.Context, bucket string) (bucketInfo BucketInfo, err error)
	ListBuckets(ctx context.Context) (buckets []BucketInfo, err error)
	DeleteBucket(ctx context.Context, bucket string, opts DeleteBucketOptions) error
	ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
	ListObjectsV2(ctx context.Context, bucket, prefix, continuationToken, delimiter string, maxKeys int, fetchOwner bool, startAfter string) (result ListObjectsV2Info, err error)
	ListObjectVersions(ctx context.Context, bucket, prefix, marker, versionMarker, delimiter string, maxKeys int) (result ListObjectVersionsInfo, err error)
	// Walk lists all objects including versions, delete markers.
	Walk(ctx context.Context, bucket, prefix string, results chan<- ObjectInfo, opts ObjectOptions) error

	// function MUST NOT return a non-nil ReadCloser.
	GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, lockType LockType, opts ObjectOptions) (reader *GetObjectReader, err error)
	GetObjectInfo(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)
	PutObject(ctx context.Context, bucket, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error)
	CopyObject(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (objInfo ObjectInfo, err error)
	DeleteObject(ctx context.Context, bucket, object string, opts ObjectOptions) (ObjectInfo, error)
	DeleteObjects(ctx context.Context, bucket string, objects []ObjectToDelete, opts ObjectOptions) ([]DeletedObject, []error)
	TransitionObject(ctx context.Context, bucket, object string, opts ObjectOptions) error
	RestoreTransitionedObject(ctx context.Context, bucket, object string, opts ObjectOptions) error

	// Multipart operations.
	ListMultipartUploads(ctx context.Context, bucket, prefix, keyMarker, uploadIDMarker, delimiter string, maxUploads int) (result ListMultipartsInfo, err error)
	NewMultipartUpload(ctx context.Context, bucket, object string, opts ObjectOptions) (uploadID string, err error)
	CopyObjectPart(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, uploadID string, partID int,
		startOffset int64, length int64, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (info PartInfo, err error)
	PutObjectPart(ctx context.Context, bucket, object, uploadID string, partID int, data *PutObjReader, opts ObjectOptions) (info PartInfo, err error)
	GetMultipartInfo(ctx context.Context, bucket, object, uploadID string, opts ObjectOptions) (info MultipartInfo, err error)
	ListObjectParts(ctx context.Context, bucket, object, uploadID string, partNumberMarker int, maxParts int, opts ObjectOptions) (result ListPartsInfo, err error)
	AbortMultipartUpload(ctx context.Context, bucket, object, uploadID string, opts ObjectOptions) error
	CompleteMultipartUpload(ctx context.Context, bucket, object, uploadID string, uploadedParts []CompletePart, opts ObjectOptions) (objInfo ObjectInfo, err error)

	// Policy operations
	SetBucketPolicy(context.Context, string, *policy.Policy) error
	GetBucketPolicy(context.Context, string) (*policy.Policy, error)
	DeleteBucketPolicy(context.Context, string) error

	// Supported operations check
	IsNotificationSupported() bool
	IsListenSupported() bool
	IsEncryptionSupported() bool
	IsTaggingSupported() bool
	IsCompressionSupported() bool
	SetDriveCounts() []int // list of erasure stripe size for each pool in order.

	// Healing operations.
	HealFormat(ctx context.Context, dryRun bool) (madmin.HealResultItem, error)
	HealBucket(ctx context.Context, bucket string, opts madmin.HealOpts) (madmin.HealResultItem, error)
	HealObject(ctx context.Context, bucket, object, versionID string, opts madmin.HealOpts) (madmin.HealResultItem, error)
	HealObjects(ctx context.Context, bucket, prefix string, opts madmin.HealOpts, fn HealObjectFn) error

	// Backend related metrics
	GetMetrics(ctx context.Context) (*BackendMetrics, error)

	// Returns health of the backend
	Health(ctx context.Context, opts HealthOptions) HealthResult
	ReadHealth(ctx context.Context) bool

	// Metadata operations
	PutObjectMetadata(context.Context, string, string, ObjectOptions) (ObjectInfo, error)

	// ObjectTagging operations
	PutObjectTags(context.Context, string, string, string, ObjectOptions) (ObjectInfo, error)
	GetObjectTags(context.Context, string, string, ObjectOptions) (*tags.Tags, error)
	DeleteObjectTags(context.Context, string, string, ObjectOptions) (ObjectInfo, error)
}
```

### 数据结构
Nas gateway 非常之简单： 

```go
// NAS implements Gateway.
type NAS struct {
	path string
}
```

就是一个 path 参数，但在 `NewGatewayLayer()` 中实例化了一个`FSObjects`，其结构是这样的：

```go
// FSObjects - Implements fs object layer.
type FSObjects struct {
	GatewayUnsupported

	// Path to be exported over S3 API.
	fsPath string
	// meta json filename, varies by fs / cache backend.
	metaJSONFile string
	// Unique value to be used for all
	// temporary transactions.
	fsUUID string

	// This value shouldn't be touched, once initialized.
	fsFormatRlk *lock.RLockedFile // Is a read lock on `format.json`.

	// FS rw pool.
	rwPool *fsIOPool

	// ListObjects pool management.
	listPool *TreeWalkPool

	diskMount bool

	appendFileMap   map[string]*fsAppendFile
	appendFileMapMu sync.Mutex

	// To manage the appendRoutine go-routines
	nsMutex *nsLockMap
}
```

不难看出，Nas 实际上是其于 MinIO 内置 `FSObjects` 来实现的。 `FSObjects` 是一种其于文件系统的网关，即使用本地文件系统来作为存储基础，这和 Nas 是一致的。

再来看 s3 gateway，相对来说就复杂得多。 

```go
type s3Objects struct {
	minio.GatewayUnsupported
	Client     *miniogo.Core
	HTTPClient *http.Client
	Metrics    *minio.BackendMetrics
}
```

 s3Objects 通过 `NewGatewayLayer()` 实例化，参数是 s3 认证要素 Credentials，内部会实例化一个 `http.Transport`，后续所有操作都会使用该http 客户端访问 s3 api 完成相应的功能实现。

### 启动过程分析
#### Gateway 启动过程
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h234nd8mp2j21w40u0djj.jpg)
MinIO Gateway 是一个相关独立的系统，从命令行启动，具体的过程如上图。

`main.go` 文件的中引入了 `cmd/gateway/` 这个包，`gateway` 包在`init` 过程中又引入了 `nas-gateway.go` 和 `s3-gateway.go` 。

这个 2 个就是系统默认附带的 NAS  网关 和 S3 网关的具体实现，它们在初始化时分别调用了 `cmd/gateway-main.go` 了 `RegisterGatewayCommand()` 方法，将自身注册成为 `gateway` 的子命令。

```go
minio.RegisterGatewayCommand(cli.Command{
		Name:               minio.S3BackendGateway,
		Usage:              "Amazon Simple Storage Service (S3)",
		Action:             s3GatewayMain,
		CustomHelpTemplate: s3GatewayTemplate,
		HideHelpCommand:    true,
	})
```

所有网关初始化完成后，`main()` 函数执行。过程中调用 `cmd/main.go` 中 `Main()` ，同时会通过 `NewApp` 创建一个 app 实例，最后运行 app 实例的  run 方法执行命令功能。

```go
func NewApp() *App {
	return &App{
		Name:         filepath.Base(os.Args[0]),
		HelpName:     filepath.Base(os.Args[0]),
		Usage:        "A new cli application",
		UsageText:    "",
		Version:      "0.0.0",
		BashComplete: DefaultAppComplete,
		Action:       helpCommand.Action,
		Compiled:     compileTime(),
		Writer:       os.Stdout,	}
}
```

App 实例的初始化工作中包括 `gateway` 命令的注册。由于前面在 `init()`过程中已经将 `nas / s3` 的网关注册成为 `gateway` 的 `subCommands` ，因此 gateway 注册之后便 可以通过 `gateway s3` 子命令来启动 s3 网关。

```go
func RegisterGatewayCommand(cmd cli.Command) error {
	cmd.Flags = append(append(cmd.Flags, ServerFlags...), GlobalFlags...)
	gatewayCmd.Subcommands = append(gatewayCmd.Subcommands, cmd)
	return nil
}
```

在完成所有命令注册之后，会根据输入参数调用对应的 Gateway 子命，基中调用 `StartGateway()` 完成整个 Gateway 启动。

#### S3 Gateway 启动
在注册 s3 Gateway 子命令到 Gateway 时需要传递一个 Action，即 子命令的入口程序。

```go
minio.RegisterGatewayCommand(cli.Command{
	Name:               minio.S3BackendGateway,
	Usage:              "Amazon Simple Storage Service (S3)",
	Action:             s3GatewayMain,
	CustomHelpTemplate: s3GatewayTemplate,
	HideHelpCommand:    true,
})
```

`s3GatewayMain()` 接受 Context 作接参数，在验证参数合法之后便会调用 `StartGateway()` 启动网关。

```go
// StartGateway - handler for 'minio gateway <name>'.
func StartGateway(ctx *cli.Context, gw Gateway) {
	// ... more code
}
```

由于`StartGateway` 程序比较复杂，在这里就不贴代码，仅详细分析一下启动过程中所进行的主要操作。

- Gateway 在启动之后是一个常驻进程，因此首先需要为其注册系统信号监听
- 初始化终端日志
- 初始化 Gateway 级别的全局 Locker
- 初始化系统配置，MinIO 使用的是自己实的配置解析系统。
- 在完成基础设置之后会初使化 Gateway router， 用以给客户端暴露一组接口
- 初使化管理功能路由（admin router ）
- 初使化健康检查功能路由（healthy check router ）
- 初使化 Metric router
- 使用 Credentials 调用 NewGatewayLayer 获对 ObjectLayer 实例
- 启动 IAM 子程序 （IAM sub-system）
- 启动 httpServer 监听客户端请求

## 总结
上面从代码级别梳理了 Gateway 的设计，在了解 Gateway 原理和启动过程之后，我们为基添加默认没有支持的存储产品也就变得非常简单。

参考：
- [从源代码级别看懂MinIO对象存储网关的实现](https://www.flysnow.org/2020/10/19/minio-gateway-sourcecode.html)