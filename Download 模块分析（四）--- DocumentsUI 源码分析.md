# Download 模块分析（四）--- DocumentsUI 源码分析

> 上一篇文章简单介绍了`SAF`框架，内容包括客户端、选取器（`Picker`）、内容提供者，而`DocumentsUI`充当了`Picker`选取器的角色。
>
> 本文基于`Android O`的系统源码，简单介绍`DocumentsUI`的整体框架，并从数据加载，和文件操作（复制、移动、删除、解压、重命名、压缩/解压等）的整个流程进行源码分析。

### 框架

#### 接口定义

![Alt text](./img/sturcture.png)

如上图所示，`DocumentsUI`中只有两个`Activity`，`FilesActivity`是正常打开应用显示的界面，`PickActivity`是进行文件操作后「跳转到」的另一个`Activity`，但两者布局基本一致，不一致的地方是是否可以进行多选、新建窗口后的逻辑等。

`Injector`和`ActionHandler`是`DocuemntsUI`中很重要的两个类：
- `Injector`为`Fragment`和`Activity`通过提供依赖注入功能，注入如`Features`, `MenuManager`, `SearchViewManager`等逻辑类。`BaseActivity`中提供了`getInjector()`抽象接口，因此子类可以实现不同的`Injector`，实现不同的逻辑细节。
- `ActionHandler`规定了`Fragment`和`Activity`中的所有「行为响应」，也就是说响应`UI`事件的接口，如全选/反选所有文件，打开特定文件，打开其他分类文件等。基本界面上的所有可执行操作，都以接口的形式写在`ActionHandler`中。可以理解类似`MVP`架构中`Presenter`层的契约类。
- `Injector`中持有了`ActionHandler`的引用，并且通过`public`访问域形式公开。

#### 数据来源

![Alt text](./img/device.png)

`DocumentsUI`数据均来源自`DocumentsProvider`及其子类，和其他`ContentProvider`。如上图所示，`DocumentsUI`中包含图片，视频，音频，最近浏览文件，下载，内部存储设备和`SD`卡，以及压缩包结构等数据。
* 图片、视频、音频等「多媒体」数据由`MediaDocumentsProvider`提供。`MediaDocumentsProvider`实际上通过`MediaProvider`访问媒体数据库获得各个分类数据。
* 最近浏览文件由`LastAccessedProvider`进行提供，提供者通过查询本地数据库的记录获取数据。所有最近浏览文件数据通过`DocumentStack`进行存储。
* 下载分类由`DownloadProvider`进行提供。
* 内部存储和`SD`卡数据由`ExternalStorageProvider`进行提供，`ExternalStorageProvider`直接继承`FileSystemProvider`，而`FileSystemProvider`继承`DocumentsProvider`。`ExternalStorageProvider`通过`File.listFiles()`获得所有文件目录及子文件。
* 压缩包层次结构由`ArchiveProvider`进行提供，`ArchiveProvider`直接继承`DocumentsProvider`。创建、解压和查看文件夹等接口通过`Archive`类统一封装，由`ArchiveProvider`统一向外暴露。


#### 数据加载
`DocumentsUI`中所有数据由`Loader`进行异步加载，其他主动更新数据的任务由`AsyncTask`封装实现。主要包含以下`Loader`：
- `RootsLoader`：除「最近浏览」之外的其他分类的加载器，包括图片、视频、音频和内部存储等。
- `RecentsLoader`：最近浏览分类加载器。
- `DirectoryLoader`：内部存储目录页面加载器。在进入不同的文件夹时候会重新加载。

由于`Loader`本身限制，「业务逻辑」和「`UI`相关」的代码不能很好地解耦，因此`DocumentsUI`中`Activity`和`Fragment`里的代码都比较臃肿，可以考虑使用`Clean Architecture`进行解耦。

#### 整体框架

![Alt text](./img/arc.png)

可以分为传统三层结构：
- 数据层：由`Provider`和`Bean`组成。`Provider`提供数据来源，包括数据库和文件目录。`Bean`包括RootInfo`和`DocumentInfo`，分别代表`SAF`框架中的`Root`根目录和`Document`单个文档。
- 控制层：由`Loader`和`ActionHandler`组成。`ActionHandler`定义所有页面的功能接口，`Loader`通过`Provider`进行动态数据加载。
- 视图层：由`Activity`和`Fragment`组成。

### 流程

以下分别对`DocumentsUI`的「数据加载」和「文件操作」流程进行分析。

#### 数据加载

最近浏览：




#### 文件操作
