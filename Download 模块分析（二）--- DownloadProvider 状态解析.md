# Download 模块分析（二）

> 上一篇文章介绍了`DownloadManager`下载文件的流程。`DownloadManager`告知`DownloadProvider`将任务加入到数据库当中，同时
> 开启后台服务`DownloadJobService`进行文件下载，下载过程中以及完成后通知`DownloadNotifier`进行下载进度更新。
>
> 本文将结合`DownloadProvider`侧重分析下载「状态」如何变化，以及对`DownloadManager`进行功能扩展。

### 状态常量
先介绍一下`DownloadProvider`状态相关常量：
- `COLUMN_STATUS`: 表示当前任务的状态，可取值`STATUS_*`等常量;
- `COLUMN_REASON`: 可认为是`COLUMN_STATUS`字段的补充。下载失败时，可取值`ERROR_*`常量；下载暂停时，可取值`PAUSED_*`常量；
- `COLUMN_CONTROL`: 表示当前任务是否暂停，可取值`0`和`1`，分别代表「正在下载」和「暂停下载」。

对于COLUMN_STATUS`字段，`DownloadManager.CursorTranslator`将`DownloadProvider`中若干状态映射为`5`个：

``` java
private int translateStatus(int status) {
            switch (status) {
                case Downloads.Impl.STATUS_PENDING:
                    return STATUS_PENDING
                case Downloads.Impl.STATUS_RUNNING:
                    return STATUS_RUNNING
                case Downloads.Impl.STATUS_PAUSED_BY_APP:
                case Downloads.Impl.STATUS_WAITING_TO_RETRY:
                case Downloads.Impl.STATUS_WAITING_FOR_NETWORK:
                case Downloads.Impl.STATUS_QUEUED_FOR_WIFI:
                    return STATUS_PAUSED
                case Downloads.Impl.STATUS_SUCCESS:
                    return STATUS_SUCCESSFUL
                default:
                    assert Downloads.Impl.isStatusError(status);
                    return STATUS_FAILED;
            }
        }
```

- `STATUS_PENDING`: 下载等待开始
- `STATUS_RUNNING`：正在下载
- `STATUS_PAUSED`: 等待重试下载，或处于暂停状态，等待继续下载
- `STATUS_SUCCESS`: 成功下载
- `STATUS_FAILED`: 下载失败，失败后不再重试

对于`COLUMN_REASON`字段：

``` java
private long getReason(int status) {
            switch (translateStatus(status)) {
                case STATUS_FAILED:
                    return getErrorCode(status);
                case STATUS_PAUSED:
                    return getPausedReason(status);
                default:
                    return 0;
            }
        }
```

`ERROR`和`PAUSED`会根据不同`STATUS`对应不同的`REASON`，不再展开。

### 状态变化
再次回到`DownloadThread.run()`中,仅保留状态相关代码：

``` java
public void run() {
        //...
        try {
            //准备下载，更改状态
            mInfoDelta.mStatus = STATUS_RUNNING;
            //状态更新到数据库
            mInfoDelta.writeToDatabase();

            //...

            if (mNetwork == null) {
                throw new StopRequestException(STATUS_WAITING_FOR_NETWORK,
                        "No network associated with requesting UID");
            }

            // ...
            // 开始下载
            executeDownload();
            // 下载完成，更改状态
            mInfoDelta.mStatus = STATUS_SUCCESS;
            //...
        } catch (StopRequestException e) {
            //...
            if (isStatusRetryable(mInfoDelta.mStatus)) {
                //...
                if (mInfoDelta.mNumFailed < Constants.MAX_RETRIES) {
                    final NetworkInfo info = mSystemFacade.getNetworkInfo(mNetwork, mInfo.mUid,
                            mIgnoreBlocked);
                    if (info != null && info.getType() == mNetworkType && info.isConnected()) {
                        // 网络正常，等待重试
                        mInfoDelta.mStatus = STATUS_WAITING_TO_RETRY;
                    } else {
                        // 网络无法连接，等待重试
                        mInfoDelta.mStatus = STATUS_WAITING_FOR_NETWORK;
                    }

                    if ((mInfoDelta.mETag == null && mMadeProgress)
                            || DownloadDrmHelper.isDrmConvertNeeded(mInfoDelta.mMimeType)) {
                        // 不存在 ETag 校验器，无法继续下载
                        mInfoDelta.mStatus = STATUS_CANNOT_RESUME;
                    }
                }
            }

            if (mInfoDelta.mStatus == STATUS_WAITING_FOR_NETWORK
                    && !mInfo.isMeteredAllowed(mInfoDelta.mTotalBytes)) {
                // 等待连接 Wi-Fi
                mInfoDelta.mStatus = STATUS_QUEUED_FOR_WIFI;
            }
        } catch (Throwable t) {
            // 未知异常
            mInfoDelta.mStatus = STATUS_UNKNOWN_ERROR;
            //...
        } finally {
            //更新状态到数据库
            mInfoDelta.writeToDatabase();
            //...
        }

        if (Downloads.Impl.isStatusCompleted(mInfoDelta.mStatus)) {
            //...
        } else if (mInfoDelta.mStatus == STATUS_WAITING_TO_RETRY
                || mInfoDelta.mStatus == STATUS_WAITING_FOR_NETWORK
                || mInfoDelta.mStatus == STATUS_QUEUED_FOR_WIFI) {
            //下载重试
            Helpers.scheduleJob(mContext, DownloadInfo.queryDownloadInfo(mContext, mId));
        }
        //...
    }
```

- `DownloadThread`在下载开始前会修改状态为`STATUS_RUNNING`，并写入到数据库当中；下载完成后将状态修改为`STATUS_SUCCESS`。
- 下载过程中会出现异常，部分为「可重试异常」，如网络、存储空间原因；抛出「不可重试异常」时直接终止下载。

之前提到原生`Android`不支持「文件断点续传」。从源码中可以得知，在外界原因（如网络、存储空间）发生变化时，下载任务会暂停或重试。但是`DownloadThread`以及`DownloadProvider`中并没有对`COLUMN_CONTROL`字段进行赋值或判断，因此每次下载时候`mControl`都不可能为`CONTROL_PAUSED`：

``` java
// DownloadInfo.java
public boolean isReadyToSchedule() {
        if (mControl == Downloads.Impl.CONTROL_PAUSED)
            return false;
        }
        //...
    }
```

### 功能扩展

我们同时可以发现，关于「暂停下载」的几个变量:

``` java
                case Downloads.Impl.STATUS_PAUSED_BY_APP:
                case Downloads.Impl.STATUS_WAITING_TO_RETRY:
                case Downloads.Impl.STATUS_WAITING_FOR_NETWORK:
                case Downloads.Impl.STATUS_QUEUED_FOR_WIFI:
```

从源码中得知`STATUS_PAUSED_BY_APP`状态并没有得到处理，而该状态正是用户「暂停下载」处理的状态。

总结一下，要想对原生`DownloadManager`进行功能扩展，大致思路如下：
1. `DownloadManager`增加暂停下载、继续下载的接口；
2. 暂停下载时候需要将`COLUMN_CONTROL`值置为`CONTROL_PAUSED`，`COLUMN_STATUS`值置为`STATUS_PAUSED_BY_APP`；继续下载将`COLUMN_CONTROL`将`CONTROL_RUN`，`COLUMN_STATUS`字段置为`STATUS_RUNNING`。

----------

参考资料：
> www.trinea.cn/android/android-downloadmanager/
