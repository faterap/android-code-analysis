# Download 模块分析(一)

> `Android`中`Download`由三个部分组成：
> - `DocumentsUI` -----> `/frameworks/base/packages/DocumentsUI/`
> - `DownloadManager` ---->`/frameworks/base/core/java/android/app/`
> - `DownloadProvider` ---->`/packages/providers/DownloadProvider/`
>
> 本文主要研究`DownloadManager.java`中的功能，包括不仅限于「开始下载」和「取消下载」。

首先了解一下`Download`模块相关的类：
- `DownloadProvider` --  数据库操作的封装，继承自`ContentProvider`
- `DownloadManager` -- 作为中间层，管理对数据库层的操作，供外部调用
- `DownloadJobService` -- 封装文件`download`，`delete`等操作，并且操纵下载的`notification`。继承自`Service`
- `DownloadNotifier` -- 控制状态栏`Notification`
- `DownloadReceiver` -- 配合`DownloadNotifier`进行文件的操作及其`Notification`。

### 开始下载

``` java
Uri uri = Uri.parse("your own uri");
        DownloadManager.Request mRequest = new DownloadManager.Request(uri);
        DownloadManager manager = (DownloadManager) getSystemService(Context.DOWNLOAD_SERVICE);
        if (manager != null) {
            // 自动开始下载任务
            manager.enqueue(mRequest);
        }
```
调用`enqueue`方法之后，只要数据连接可用并且 `DownloadManager` 可用，下载就会自动开始。

#### 流程分析

**DownloadManager.java**

`DownloadManager.enqueue()`：
``` java
public long enqueue(Request request) {
        ContentValues values = request.toContentValues(mPackageName);
        Uri downloadUri = mResolver.insert(Downloads.Impl.CONTENT_URI, values);
        long id = Long.parseLong(downloadUri.getLastPathSegment());
        return id;
    }
```
`enqueue()`方法会直接开启所指定的下载任务。其中`request.toContentValues()`会将`Request`类中的属性写入到 `ContentValues`当中，最后将该记录写入到`Downloads.Impl.CONTENT_URI`所在数据库当中。

查看 `DownloadProvider.insert()`:

``` java
    public Uri insert(final Uri uri, final ContentValues values) {
        //...
        final long token = Binder.clearCallingIdentity();
        try {
            Helpers.scheduleJob(getContext(), rowID);
        } finally {
            Binder.restoreCallingIdentity(token);
        }
        //...
        return ContentUris.withAppendedId(Downloads.Impl.CONTENT_URI, rowID);
    }
```
将新增下载任务写入到数据库中，此处过滤了`insert()`中大部分操作下载数据库字段的不相关代码。之前的版本中，`DownloadProvider`通过`context.startService()`启动`DownloadService`。在 Android N 版本之后，`DownloadProvider`引入了`JobScheduler`进行异步下载任务的处理。而且`JobScheduler`可以在设备达到某些条件（如连接充电器、在WiFi环境下等）才执行任务，可以节省用户设备的资源。

**Helpers.java**

`Helpers`类是控制异步任务下载的一个工具类，查看`Helpers.scheduleJob()`:
``` java
    public static void scheduleJob(Context context, long downloadId) {
        final boolean scheduled = scheduleJob(context,
                DownloadInfo.queryDownloadInfo(context, downloadId));
        if (!scheduled) {
            // If we didn't schedule a future job, kick off a notification
            // update pass immediately
            getDownloadNotifier(context).update();
        }
    }
```
`Helpers.scheduleJob()`方法中使用`rowId`将那条下载信息查询出来，然后调用绑定的`DownloadJobService`进行下载任务。如果线程调度失败，会返回`false`。`getDownloadNotifier(context).update()`和更改通知栏 UI 相关，不进行详述。

`scheduleJob()`中设置了`JobScheduler`的相关属性：
``` java
    public static boolean scheduleJob(Context context, DownloadInfo info) {
        //...
        final JobInfo.Builder builder = new JobInfo.Builder(jobId,
                new ComponentName(context, DownloadJobService.class));
        //...
        scheduler.scheduleAsPackage(builder.build(), packageName, UserHandle.myUserId(), TAG);
        return true;
    }
```
此处配置了`JobScheduler`的相关属性，详细逻辑在`DownloadJobService`里进行实现。

**DownloadJobService.java**

下载任务通过`DownloadJobService`进行调度，`onStartJob()`中开始下载:
``` java
    public boolean onStartJob(JobParameters params) {
        final int id = params.getJobId();
        // Spin up thread to handle this download
        final DownloadInfo info = DownloadInfo.queryDownloadInfo(this, id);
        //...
        final DownloadThread thread;
        synchronized (mActiveThreads) {
            thread = new DownloadThread(this, params, info);
            mActiveThreads.put(id, thread);
        }
        thread.start();
        return true;
    }
```
`DownloadThread`中实现相关逻辑。

**DownloadThread.java**

我们主要关注`run()`方法：

``` java
@Override
    public void run() {
      //...
        try {
          // 开始下载
            executeDownload();
            //...
        } catch (StopRequestException e) {
          //...
          // Some errors should be retryable, unless we fail too many times.
            if (isStatusRetryable(mInfoDelta.mStatus)) {
              //...
                if (mInfoDelta.mNumFailed < Constants.MAX_RETRIES) {
                  //...
                }
            }
          //...
        }
        //...
        } finally {
          // 失败：删除已删除文件；成功：移动文件到下载目录
          finalizeDestination();

          // 将任务更新到数据库
            mInfoDelta.writeToDatabase();
            //...
        }

        if (Downloads.Impl.isStatusCompleted(mInfoDelta.mStatus)) {
            // 扫描相关数据库
                DownloadScanner.requestScanBlocking(mContext, mInfo.mId, mInfoDelta.mFileName,
                        mInfoDelta.mMimeType);
        } else if (mInfoDelta.mStatus == STATUS_WAITING_TO_RETRY
                || mInfoDelta.mStatus == STATUS_WAITING_FOR_NETWORK
                || mInfoDelta.mStatus == STATUS_QUEUED_FOR_WIFI) {
                  // 尝试重新下载
            Helpers.scheduleJob(mContext, DownloadInfo.queryDownloadInfo(mContext, mId));
        }

        // 结束 JobScheduler 的任务
        if (!mShutdownRequested)
        mJobService.jobFinishedInternal(mParams, false);
    }
```

`DownloadThread`的`run()`方法中，核心下载方法为`executeDonwload()`：
``` java
    private void executeDownload() throws StopRequestException {
        final boolean resuming = mInfoDelta.mCurrentBytes !=
        //...
        int redirectionCount = 0;
        while (redirectionCount++ < Constants.MAX_REDIRECTS) {
            if ((!cleartextTrafficPermitted) && ("http".equalsIgnoreCase(url.getProtocol()))) {
                throw new StopRequestException(STATUS_BAD_REQUEST,
                        "Cleartext traffic not permitted for UID " + mInfo.mUid + ": "
                                + Uri.parse(url.toString()).toSafeString());
            }
            HttpURLConnection conn = null;
            try {
                //...
                checkConnectivity();
                conn = (HttpURLConnection) mNetwork.openConnection(url);
                //...
                addRequestHeaders(conn, resuming);
                final int responseCode = conn.getResponseCode();
                switch (responseCode) {
                    case HTTP_OK:
                        if (resuming) {
                            throw new StopRequestException(
                                    STATUS_CANNOT_RESUME, "Expected partial, but received OK");
                        }
                        parseOkHeaders(conn);
                        transferData(conn);
                        return;

                    case HTTP_PARTIAL:
                        if (!resuming) {
                            throw new StopRequestException(
                                    STATUS_CANNOT_RESUME, "Expected OK, but received partial");
                        }
                        transferData(conn);
                        return;
                    case HTTP_MOVED_PERM:
                    case HTTP_MOVED_TEMP:
                    case HTTP_SEE_OTHER:
                    case HTTP_TEMP_REDIRECT:
                        final String location = conn.getHeaderField("Location");
                        url = new URL(url, location);
                        if (responseCode == HTTP_MOVED_PERM) {
                            // Push updated URL back to database
                            mInfoDelta.mUri = url.toString();
                        }
                        continue;
                    //...
                }
            //...
        }
        throw new StopRequestException(STATUS_TOO_MANY_REDIRECTS, "Too many redirects");
    }
```

以上代码分为几部分进行分析。
`while`循环表示重定向尝试次数，超过限定次数的重定向，下载任务会终止并抛出异常。简单介绍一下重定向的概念：

**重定向**：

> 理想情况下，一项资源只有一个访问位置，也就是只有一个 URL 。但是由于种种原因，需要为资源设定不同的名称（即不同的域名，例如带有和不带有www 前缀的URL，以及简短易记的 URL 等）。在这种情况下，实用的方法是将其重定向到那个实际的（标准的）URL，而不是复制资源。
>
> HTTP 版本站点的请求会被重定向至采用了 HTTPS 协议的版本。

接着看`while`循环里面的逻辑：
``` java
if ((!cleartextTrafficPermitted) && ("http".equalsIgnoreCase(url.getProtocol()))) {
                throw new StopRequestException(STATUS_BAD_REQUEST,
                        "Cleartext traffic not permitted for UID " + mInfo.mUid + ": "
                                + Uri.parse(url.toString()).toSafeString());
            }
```

首先需要传输用户 `UID`。用户 `UID` 强制使用明码传输，因此不能使用 `HTTPS` 协议传输。因为 `HTTP` 重定向过程中可能会切换到 `HTTPS` 协议。

`checkConnectivity()`中会检查网络相关状况，包括是否有网络连接，是否使用蜂窝数据，以及是否显示使用数据大小。

最后`DownloadInfoDelta`中相关信息（如请求字节数，压缩算法等）需要写入到`http`请求头中：

**`HTTP` 请求头：**

``` java
    private void addRequestHeaders(HttpURLConnection conn, boolean resuming) {
        //...
        conn.setRequestProperty("Accept-Encoding", "identity");
        conn.setRequestProperty("Connection", "close");

        if (resuming) {
            if (mInfoDelta.mETag != null) {
                conn.addRequestProperty("If-Match", mInfoDelta.mETag);
            }
            conn.addRequestProperty("Range", "bytes=" + mInfoDelta.mCurrentBytes + "-");
        }
    }
 ```
`addRequestHeaders()`方法中的第二个参数`resuming`表示当前下载任务是否需要继续下载，即当前任务处于暂停状态。
可以通过从数据库查询相应下载任务已写入文件字节数，若不为`0`则表示需要继续下载。

**请求头中相关字段介绍：**

-  `Accept-Encoding`: 客户端能够理解的内容编码方式。`identity`表示不使用任何压缩算法进行编码。
-  `Connection`: 决定当前的事务完成后，是否会关闭网络连接。`close`表示任务完成后关闭连接。
-  `If-Match`：`HTTP`缓存相关字段，表示一个条件请求。在请求方法为 `GET` 和 `HEAD` 的情况下，服务器仅在请求的资源满足此首部列出的 `ETag` 之一时才会返回资源。`value`为`ETag`值。
-  `Range`: 告知服务器请求返回文件的哪一部分。`value`为请求文件比特数的范围。

查看`http`响应报文：

**`HTTP` 响应：**

``` java
    switch (responseCode) {
                    case HTTP_OK:
                        if (resuming) {
                            throw new StopRequestException(
                                    STATUS_CANNOT_RESUME, "Expected partial, but received OK");
                        }
                        parseOkHeaders(conn);
                        transferData(conn);
                        return;
                    case HTTP_PARTIAL:
                        if (!resuming) {
                            throw new StopRequestException(
                                    STATUS_CANNOT_RESUME, "Expected OK, but received partial");
                        }
                        transferData(conn);
                        return;
                    case HTTP_MOVED_PERM:
                    case HTTP_MOVED_TEMP:
                    case HTTP_SEE_OTHER:
                    case HTTP_TEMP_REDIRECT:
                        final String location = conn.getHeaderField("Location");
                        url = new URL(url, location);
                        if (responseCode == HTTP_MOVED_PERM) {
                            // Push updated URL back to database
                            mInfoDelta.mUri = url.toString();
                        }
                        continue;
                    //...
                }
```
**下载任务相关 `HTTP` [状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Redirections)：**
- `HTTP_OK`: 200。表示请求成功
- `HTTP_PARTIAL`: 206。成功状态响应代码表示请求已成功，并且主体包含所请求的数据区间，该数据区间是在请求的 `Range` 首部指定的。
-  `HTTP_MOVED_PERM`: 301。`URL` 永久重定向，`GET` 方法不会发生变更，其他方法有可能会变更为 `GET` 方法。
-  `HTTP_MOVED_TEMP`: 302。`URL`临时重定向，	`GET` 方法不会发生变更，其他方法有可能会变更为 `GET` 方法。
-  `HTTP_SEE_OTHER`: 303。`URL`临时重定向，`GET` 方法不会发生变更，其他方法会变更为 `GET` 方法（消息主体会丢失）。
-  `HTTP_TEMP_REDIRECT`307。`URL`临时重定向，方法和消息主体都不发生变化。

下载过程中，重定向的相关状态码处理一致，均继续执行循环，直到达到重定向次数上限。

1. 状态码返回`200`：
``` java
    private void parseOkHeaders(HttpURLConnection conn) throws StopRequestException {
        if (mInfoDelta.mFileName == null) {
            // ...
            try {
                mInfoDelta.mFileName = Helpers.generateSaveFile(mContext, mInfoDelta.mUri,
                        mInfo.mHint, contentDisposition, contentLocation, mInfoDelta.mMimeType,
                        mInfo.mDestination);
            }
            //...
        }
        //...
        mInfoDelta.writeToDatabaseOrThrow();
    }
```
首先来看请求成功的情况。`parseOkHeaders()`解析响应头中的不同字段，并且存到内部类`DownloadInfoDelta`当中。`Helps.generateSaveFile()`生成下载相关文件。`writeToDatabaseOrThrow()`则通过`DownloadProvider.update()`将任务下载到数据库当中。

查看`transferData()`代码：
``` java
    private void transferData(HttpURLConnection conn) throws StopRequestException {
        //...
        try {
            //...
            try {
                outPfd = mContext.getContentResolver()
                        .openFileDescriptor(mInfo.getAllDownloadsUri(), "rw");
                outFd = outPfd.getFileDescriptor();

                if (DownloadDrmHelper.isDrmConvertNeeded(mInfoDelta.mMimeType)) {
                    drmClient = new DrmManagerClient(mContext);
                    out = new DrmOutputStream(drmClient, outPfd, mInfoDelta.mMimeType);
                } else {
                    out = new ParcelFileDescriptor.AutoCloseOutputStream(outPfd);
                }

                if (mInfoDelta.mTotalBytes > 0) {
                  //...
                    StorageUtils.ensureAvailableSpace(mContext, outFd, newBytes);
                    try {
                        // We found enough space, so claim it for ourselves
                        Os.posix_fallocate(outFd, 0, mInfoDelta.mTotalBytes);
                    }
                    //...
                }

                Os.lseek(outFd, mInfoDelta.mCurrentBytes, OsConstants.SEEK_SET);
            }
            //...
            transferData(in, out, outFd);
            // ...
        }
        //...
    }
```

流对象通过`DRM`框架进行创建，简单介绍一下：
>[DRM](https://developer.android.google.cn/reference/android/drm/package-summary.html)，英文全称为`Digital Rights Management`，译为数字版权管理。它是目前业界使用非常广泛的一种数字内容版权保护技术。`DRM`框架提供一套机制对用户使用手机上的媒体内容（如`ringtone`, `mp3`等）进行限制，如限制拷贝给第三方，限制使用次数或时限等，从而保护内容提供商的权利。

`DrmManagerClient`是`Android`中`DRM`框架的核心接口类。如果下载文件为版权保护文件，则通过文件描述符、`mimeType`等变量创建`DrmOutPutStream`，反之创建`ParcelFileDescriptor`中相应流对象（`ParcelFileDescriptor`是可以用于进程间`Binder`通信的`FileDescriptor`）

`StorageUtils.ensureAvailableSpace()`检查存储空间是否满足当前下载。该方法通过`PackageManager`删除一些缓存目录和较早前已下载文件，来释放存储空间。

`Os.sleek(FileDescriptor, long offset, int whence)`通过文件描述符设置当前偏移量`mInfoDelta.mCurrentBytes`，即支持从文件中间某个字节开始读写。可以从底层支持文件断点续传。

`transferData()`中核心方法为`transferData(InputStream, OutputStream, FileDescriptor)`，代码如下:
``` java
    private void transferData(InputStream in, OutputStream out, FileDescriptor outFd)
            throws StopRequestException {
        while (true) {
            int len = -1;
            if (mShutdownRequested) {
                throw new StopRequestException(STATUS_HTTP_DATA_ERROR,
                        "Local halt requested; job probably timed out");
            }
            try {
                len = in.read(buffer);
            }
            //...

            try {
                out.write(buffer, 0, len);
                mMadeProgress = true;
                mInfoDelta.mCurrentBytes += len;
                updateProgress(outFd);
            }
            //...
        }
        // 检查已下载文件是否完整
        if (mInfoDelta.mTotalBytes != -1 && mInfoDelta.mCurrentBytes != mInfoDelta.mTotalBytes) {
            throw new StopRequestException(STATUS_HTTP_DATA_ERROR, "Content length mismatch");
        }
    }
```

`mShutdownRequested`变量周期性检查任务是否被取消，由`DownloadThread.requestShutDown()`进行控制。
接下来的逻辑就是通过普通`I/O`流进行文件读写了。读写完固定字节数后通过`updateProgress()`更新下载进度。

2. 返回`206`，则请求目标下载文件的部分数据。流程与请求所有数据大致相同，不再进行站看。
3. 返回`307`等重定向相关状态码：如果 `URL`永久重定向，则将重定向后的`URL`写入到下载数据库当中；如果是临时重定向，则继续执行循环执行请求，直到达到重定向最大限定次数。

以上为`executeDonwload()`的整个流程。下载过程出现以下异常，`DownloadManager`=会尝试重新下载，最多不超过`20`次：
 - 网络连接错误
 - 存储空间不足
 - 所下载文件超过蜂窝数据上限，需要等待`WiFi`连接继续下载

 ``` java
 if (mInfoDelta.mStatus == STATUS_WAITING_TO_RETRY
                || mInfoDelta.mStatus == STATUS_WAITING_FOR_NETWORK
                || mInfoDelta.mStatus == STATUS_QUEUED_FOR_WIFI) {
                  // 重新进行下载任务
            Helpers.scheduleJob(mContext, DownloadInfo.queryDownloadInfo(mContext, mId));
        }
 ```

下载完毕后需要执行`DownloadThread.run()`中的`finally`的代码块:

``` java
finally {
  // 失败：删除已删除文件；成功：移动文件到下载目录
  finalizeDestination();

  // 将任务更新到数据库
    mInfoDelta.writeToDatabase();
    //...
}
```

`writeToDatabase()`通过`DownloadProvider`进行`update()`，数据库更新完成后执行`DownloadInfo.sendIntentIfRequested()`。最后`sendIntentIfRequested()`中会发送系统广播，通知文件下载已完成。

## 取消下载

取消下载通过`DownloadManager.remove(long id)`实现。如果下载任务正在进行，该任务会被中断；如果任务已经完成（包括下载成功或失败），取消后文件将被删除。

``` java
public int remove(long... ids) {
        return markRowDeleted(ids);
    }
```

继续往下看`markRowDeleted(ids)`:

```java
public int markRowDeleted(long... ids) {
        if (ids == null || ids.length == 0) {
            // called with nothing to remove!
            throw new IllegalArgumentException("input param 'ids' can't be null");
        }
        return mResolver.delete(mBaseUri, getWhereClauseForIds(ids), getWhereArgsForIds(ids));
    }
```

查看`DownloadProvider.delete(...)`方法：

``` java
    public int delete(final Uri uri, final String where, final String[] whereArgs) {
        //...
        switch (match) {
            case MY_DOWNLOADS:
            case MY_DOWNLOADS_ID:
            case ALL_DOWNLOADS:
            case ALL_DOWNLOADS_ID:
            ///...
                try (Cursor cursor = db.query(DB_TABLE, null, selection.getSelection(),
                        selection.getParameters(), null, null, null)) {

                    while (cursor.moveToNext()) {
                      //...
                        // 取消当前任务
                        scheduler.cancel((int) info.mId);

                        if (!TextUtils.isEmpty(path)) {
                          //...
                          // 删除已下载文件
                            file.delete();
                        }

                        if ((!TextUtils.isEmpty(path) && new File(path).exists())
                            || toDeleteFromMediaDB) {
                                //...
                                  // 删除 MediaProvider 记录
                                    int count_media = getContext().getContentResolver().delete(
                                            Uri.parse(mediaUri), null, null);
                                  // ...
                        }
                        //...
                    }
                }
                //...
                // 删除 DownloadProvider 记录
                count = db.delete(DB_TABLE, selection.getSelection(), selection.getParameters());
                break;
                //...
        }
        //...
        return count;
    }
```

`DownloadProvider`中有一个字段`mediaprovider_uri`，`value`指的是该记录在`MediaProvider`中`uri`所对应的值。因此`DownloadProvider`中记录发生改变时，`MediaProvider`记录要同步改变。

#### 简单总结整个任务下载的流程：

1. `checkConnectivity()`检查网络状态；
2. `HttpURLConnection`建立网络连接，将任务相关数据写入请求，并等待响应；
3. 解析响应报文，将任务通过`DownloadProvider`写入到数据库，并且更新界面；
4. 检查磁盘是否有足够存储空间，不够的话尝试释放缓存目录以及已下载文件；
5. 开始进行文件`I/O`读写，周期性检查当前任务是否被取消。输出流写入文件时再次检查是否有足够空间；
6. 下载成功后更新响应界面，发送系统广播，更新下载数据库，媒体数据库中条目相关字段；
7. 若下载过程出现网络、存储空间错误，会尝试进行重新下载。达到一定次数后终止下载任务。其他错误则终止下载任务。

![Alt text](./img/9b492b3c-72bd-4bdc-b435-7e7d49e04130.png)

整体外源应用层通过`FrameWork`层`DownloadManager API`调用到`DownloadProvider`，通过`DownloadProvider`对下载数据库进行增删查改，最后通过`DownloadService`进行线程调度完成下载流程。整个下载流程由`DownloadProvider`作为中间模块进行过渡调用，数据库与`Service`都通过`DownloadProvider`进行隔离。

**Note**:
- `DownloadManager`还提供了删除下载（`DownloadManager.remove(long)`），查询下载信息(`DownloadManager.query(Query)`)等接口，实际上还是对`DownloadProvider`进行操作，此处不再详述。
- `DownloadProvider`以及数据库中均提供断点续传相关实现，但是`DownloadManager`没有相关暂停/继续下载接口，需要开发者自行实现。如何实现可查看下一篇文章。

----------

参考资料：

> https://www.jianshu.com/p/c9dc04af2f54
> https://blog.csdn.net/chaoy1116/article/details/22384841
