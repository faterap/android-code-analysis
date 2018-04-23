# Download 模块分析（二）

> 上一篇文章介绍了`DownloadManager`下载文件的流程。`DownloadManager`告知`DownloadProvider`将任务加入到数据库当中，同时
> 开启后台服务`DownloadJobService`进行文件下载，下载过程中以及完成后通知`DownloadNotifier`进行下载进度更新。
>
> 之前提到原生`Android`不支持「文件断点续传」。`DownloadManager`以及`DownloadProvider`均有变量判断当前任务是否暂停，但是上层没有提供`相关接口`，因此进行下载时会直接忽略「是否暂停」的标志位。本文以`MT6757`为例，解析`MTK`平台如何实现「文件续传」功能。

### DownloadProvider:
先介绍一下`DownloadProvider`控制暂停/继续下载的相关字段：
- `COLUMN_STATUS`: 表示当前任务的状态，可取值`STATUS_*`等常量。暂停相关的常量：
 - `STATUS_WAITING_TO_RETRY`: 遇到网络错误，等待重新下载；
 - `STATUS_WAITING_FOR_NETWORK`: 等待网络重新连接；
 - `STATUS_QUEUED_FOR_WIFI`：等待连接`WiFi`环境。
- `COLUMN_REASON`: 可认为是`COLUMN_STATUS`字段的补充。下载失败时，可取值`ERROR_*`常量；下载暂停时，可取值`PAUSED_*`常量；
- `COLUMN_CONTROL`: 表示当前任务是否暂停，可取值`0`和`1`，分别代表「正在下载」和「暂停下载」。

如何判断任务是否暂停：

``` java
if(mInfo.queryDownloadControl() == CONTROL_PAUSED){
  //...
}
```

就是这么简单。终于可以开始正题了。。
