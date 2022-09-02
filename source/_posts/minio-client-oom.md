---
title: MinIO Go Client OOM 引发的故障排查
date: 2022-09-02 08:00:00
tags: 
- 技术
- Docker
- MinIO
---

## 背景

本周我们系统为了提升包含多文件的任务处理效率，将原来的串行化文件处理做了一优化，改成了协程并行。

```Go
wg.Add(len(fileList))
for _, pdf := range fileList {
    SplitPdfFile(f)
    wg.Done()
}
wg.Wait()
```

SplitPdfFile 会调用 文件服务 进行 PDF 文件处理 （按页分割、合并页码）。

## 问题浮现

代码上线后，完了，功能完全不可用。原因是文件服务返回了 503 的错误响应。

## 问题排查与分析过程

### 1. 为什么会 503

通过 rancher 查看，服务被重启了。由于是服务直接被 k8s 重启，程序并没有记录日志，而且我们也无法进入服务器从外部查看 k8s 日志，没有进一步的信息，但其他 API 并不到返回 503，仅仅是这一个 API，于是在当时盲猜了几个原因：

- 是有没有 recovery 的 panic 导致程序异常了
- 因为是文件处理，可能文件异常

进一步分析与验证：

- 程序在全局有注册 recovery，按道理所有的 painc 都会被捕获并销记录错误日志，但此情况下是没有任务日志被记录，故排除。
- 对于第二点，把对应文件放在本地进行处理，程序完全正常，故排除。

关注点再次回到容器上，容器为什么会重启，重启的原因是什么？但我们能用的仅有 rancher，通过 google 搜索神器，我们找到在服务的 yaml 文件的 State 节点下，会记录容器上次重启的原因：**OOMKilled**

### 2. 什么原因导致程序 OOM？

老实讲，没有想过这个不太大的文件服务会 OOM，通过观察发现程序初始启动时消耗的内存大约在 1300MB，请求部分 API 后会稳定在 1500MB 左右，而服务中的内存 limit 是 2Gi，按道理是足够的。

另外还观察到，只要请求 PDF 文件处理 API，每次请求内点点用就会增加大约 0.6 ～ 1.5g，且不会释放，多来几次就被 kill 了。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sfaox8vtj20rk01kjrh.jpg)
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sfav9l6mj20ty01m74f.jpg)
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sfb1yf6aj20so01o3yn.jpg)

这个现象 100%复现，也就是可以进一步确认是 PDF 文件处理会导致 OOM

### 3. 会是不正确使用文件引发的 OOM 吗？
这是一个 PDF 文件处理功能，那是否存在打开的文件忘关了、重复载入了文件的可能性？不排除这种可能性，于是花费了一些时间对系统的全部文件操作进行审查，顺路优化了一些代码：

Before

```Go
 f := os.Open(path)
 
 anotherFun(f)
 
 func anotherFun() {
     // code
     
     defer f.Close()
     
     // code
 }
```

After

```Go
f := os.Open(path)
defer f.Close()

anotherFun(f)
func anotherFun(f *os.File) {
     // code
     
     defer f.Close()
     
     // code
}
```

Before

```Go
 for _, v := range fileList {
    f := os.Open(path)
    defer f.Close()
    
    // code
 }
```

After

```Go
for _, v := range fileList {
    f := os.Open(path)
    
    // code
    _ = f.Close()
 }
```

仔细排查文件操作之后，所有文件使用都已规范、文件关闭时机都很合理、也不存在重复读入的问题，但在线上问题依旧！

### 4. 上神器 pprof
借助pprof，我们观察到在本机运行，即使是 50 个并发循环 10 次这样量级内存占用依然是稳定的！并不会像服务器上一样出现 OOM 甚至崩溃的情况。

程序启动

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sfa7zki5j21800j0n27.jpg)

并发请求后

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sfagqbglj215o0is76u.jpg)

由些基本可以确定问题仅出现在线上环境（test-demo），于是把线上的 heap 信息down 到本地分析，最终发现是 MinIO Client 占用了大量内存没有释放。

### 5. 为啥在本地没有复现？
定位到问大概的问题，我们再回到本地，分析本地不能复现的原因。经过查太码，发现在本地运行模式和线上有一些差异。

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5sf94dstyj20u00uh760.jpg)

通过上图，我们能看到在本机模式下，程序并没有经过 MinIO Client，而是直连的 OSS，也就异致问题不能在本机进行复现。

### 6. 最后，MinIO Client 为什么会导致 OOM？
如果使用 “minio client oom” 在 google 进行搜索，会发现已有相关记录而并非是个例。 大家遇到的问题和我们是一样的。

OOM 其实是由 mc.PutObject() 这个函数触发， 具体的代码如下：

![[VTP5CEuCnh 1.jpg]]

注意第二个参数 size，此处的值为 -1。

Size 的作用是指定要上传的文件大小，MinIO 会根据不同的文件大小使用不同的上传策略。对于没有指定大不的文件（-1），MinIO Client 会认为该文件的大小为 5TB，并以 5G 的分片大小进行上传，每次会将该片的全部字节读入内存中，那如果如时操作多个文件，就会导致内存耗尽。

```Go
// PutObject creates an object in a bucket.
//
// You must have WRITE permissions on a bucket to create an object.
//
//  - For size smaller than 16MiB PutObject automatically does a
//    single atomic PUT operation.
//
//  - For size larger than 16MiB PutObject automatically does a
//    multipart upload operation.
//
//  - For size input as -1 PutObject does a multipart Put operation
//    until input stream reaches EOF. Maximum object size that can
//    be uploaded through this operation will be 5TiB.
//
//    WARNING: Passing down '-1' will use memory and these cannot
//    be reused for best outcomes for PutObject(), pass the size always.
//
// NOTE: Upon errors during upload multipart operation is entirely aborted.
func (c *Client) PutObject(ctx context.Context, bucketName, objectName string, reader io.Reader, objectSize int64,
   opts PutObjectOptions) (info UploadInfo, err error) {
```

其实，PutObject 方法原型中的 Waring 有提醒我们，使用该方法时都需要传递文件尺寸，奈何一开始没有注意到！

### 7. 解决方法
找到了问题，那解决方案也很简单了。在 size 处传递正确的文件尺寸即可。


## 总结

- 从业务上考虑我们要设置多大的文件
- 根据文件 Size 上限、最多支持 10, 000 part 、并发度控制，容器内存大小等因素来指定 part 大小
- minio 需要根据 part size 的大小来确定最高的并发度来防止容器 OOM，并且需要控制到 minio 驱动层，而不是业务层
- 虽然 Minio S3 接口不支持流式，但支持分片，所以上传大文件的时候仍然需要用流式，而不是把大文件都加载到内存才开始上传到 Minio