# 文件监控系统

# 一、UI设计

## 1.1 主界面UI设计
![alt text](./readme/1723461839445.jpg)

包含：
工具栏、监控目录显示（QTableView）、监控消息显示（QTableView）、监控目录的右键菜单栏、监控消息显示的字体大小调节按钮

## 1.2 新增与编辑UI设计

![alt text](./readme/image-8.png)

新增目录与编辑目录共用一个窗口，设计图如上，简单、快捷。
4个GroupBox分别包含了不同的过滤内容，主要用到QlineEdit，QCheckBox,QRadioButton,QPushButton等组件。

## 1.3 查询日志UI设计
![alt text](./readme/1723462132202.jpg)

包含消息日志、操作日志。
操作方式大同小异。

# 二、设计思路

## 2.1 整体架构设计

项目的整体架构设计思路图如下图所示：

![alt text](./readme/架构图.jpg)

主要包含如下组件：

* KToolBar
工具栏，继承QToolBar，包含新增目录、导入导出配置、查询日志、操作帮助、导出消息等功能，采用QAction实现。
* KFileWidget
新增目录与编辑目录的窗口，继承QWidget实现，对监控目录进行配置编辑。
* KDirectoryTableView、KDirectoryModel
采用MVD设计模式，将监控的目录信息与视图显示分开管理，利于数据更新，便于代码的后续维护与扩展，另外如后续有自定义组件需求，可增加自定义委托。
* KLogViewer、KLogModel、KHightLightDelegate

	监控信息的显示窗口，同样采用MVD设计模式。综合考虑，优势在于：
	* 监控消息属于“大量数据”的处理，自定义数据模型可以实现数据的按需加载，提高视图的性能。
	* 数据与视图分开管理，利于维护。
	* 自定义委托，实现搜索关键词高亮显示，提高用户体验。
* KLogDialog
	* QTabWidget
		* MessageLogTab、OperationLogTab
		* KLogViewer、QTableWidget

	查询日志的对话框，采用QTabWidget将消息日志与操作日志分开，复用KLogViewer以显示消息日志，开启自定义委托；而操作日志相对简单，直接使用QTableWidget显示。

* KConfigManager
配置管理器，负责导出当前配置为json文件，下次使用直接打开json文件，即可完成上次默认配置。
* KFileMonitorManager
	* KFileMonitor

	文件监控管理者与文件监控类，KFileMonitor由KFileMonitorManager管理，本系统采用单线程对应单对象模式。即新增一个文件监控目录，创建文件监控对象，创建一个线程，优势在于任意监控目录可随时暂停，开启，互不影响，响应速度快。创建的对象与线程统一由KFileMonitorManager管理，在此完成创建于析构。
* KSystemTray
系统最小化托盘类。

* KGlobalData
全局数据，采用单例模式实现，便于其他类获取监控目录数据与配置信息，并且采用单例模式的优势是仅初始化一份内存，便于管理维护。

项目类间关系UML图如下所示：

![alt text](./readme/uml.png)

## 2.2 全局数据的单例模式

设计思路：

![alt text](./readme/全局单例.jpg)

全局单例数据，管理了如下对象与数据：

* QList<KWatchedDirectoryInfo*>

* KOperationLogService*

* KMessageLogService*

* KMessageQueue<KEventMessage>


分别是监控目录信息，操作日志服务，消息日志服务，监控消息队列。（后续详细介绍）

全局单例数据类的UML图如下所述

![alt text](./readme/image.png)


## 2.3 本项目基础数据结构

如下图所示，主要定义了4个数据结构，分别是：监听消息体、操作日志信息、监控目录信息、线程安全的消息队列（存储与读取监听消息体）。详细信息见图。

![alt text](./readme/基本数据结构.jpg)


# 三、设计亮点

## 3.1 MVD设计模式运用
项目构思时，考虑到本项目频繁涉及到信息显示与编辑、大量数据显示与更新等问题，结合课程所学内容，分析监控文件夹目录配置信息与监听消息信息的特点，决定采用MVD设计模式，用于将数据、逻辑和界面分离，以提高代码可读性、可维护性和可扩展性。

#### 3.1.1 监控文件夹目录配置信息
界面显示自定义KDirectoryTableView，数据模型自定义KDirectoryModel。

* KDirectoryModel：
重写相关数据显示的虚函数，提供void setContent(QList<KWatchedDirectoryInfo*>& infos);接口，外部调用以更新数据，达到UI同时更新目的。

* KDirectoryTableView：
	* 继承QTableView，负责显示KDirectoryModel中的数据。
	* 重写鼠标双击事件，双击打开编辑目录。
	* 将自定义的KTableContextMenu作为私有成员，右键打开菜单。

    ```C++
        KDirectoryModel* m_pModel;
        KTableContextMenu* m_pContextMenu;
        virtual void mouseDoubleClickEvent(QMouseEvent* event) override;
    ```

UML关系图（MV设计模式，架构清晰，方便维护）：

![alt text](./readme/image-1.png)


#### 3.1.2 监听消息显示

视图KLogViewer，数据模型KLogModel，自定义委托KHighlightDelegate, 排序模型KSortFilterProxyModel。

* KLogModel
消息日志数据模型，提供批量添加数据与整体设置数据接口，以实现主UI消息显示与查询日志窗口消息显示，实现代码复用。（即主界面消息显示与查询日志共用一个视图显示类，提供不同的数据设置接口）。

```C++
    void addLogEntries(const QVector<KEventMessage>& logEntries); // 批量添加数据
    void setContent(QVector<KEventMessage>& messages); // 整体设置数据（搜索查询使用）
```

* KSortFilterProxyModel
继承QSortFilterProxyModel，实现点击表头从而去对表格内的数据排序，原始排序对数字排序无法达到从小到大的排序目的，故继承重写lessthan虚函数，实现数字从小到大排序。

```C++
    virtual bool lessThan(const QModelIndex& left, const QModelIndex& right) const override;
```

* KLogViewer
	* 继承QTableView，负责显示监听消息的数据。
	* 构造函数通过设置bool m_isQueryLog，判断是否使用高亮委托与排序模型（区分显示与查询窗口）。
	* 加入两个按钮，调节显示字体大小，提高用户体验。

* KHighlightDelegate
高亮委托，用户关键字查询的时候，黄色高亮显示关键字，通过重写画图函数与字体实现。

```C++
    virtual void paint(QPainter* painter, const QStyleOptionViewItem& option, const QModelIndex& index) const override;
    virtual QSize sizeHint(const QStyleOptionViewItem& option, const QModelIndex& index) const override;
```

UML关系图（MVD设计模式，架构清晰，方便维护）：

![alt text](./readme/image-2.png)

## 3.2 分层架构设计

分析软件需求，得知消息日志与操作日志的保存属于长久存储内容，故考虑采用数据库存储相关日志数据，采用三层架构设计方案。此方案便于维护扩展，各个模块职责也相对更加清晰。具体如下：

* 表示层：
	* KLogDialog、MessageLogTab、OperationLogTab
    * 消息日志查询UI，操作日志查询UI
        
* 业务逻辑层：
    * 业务逻辑层负责处理系统的业务逻辑，包括数据插入、查询处理等任务
    * KOperationLogService负责操作日志的查询、插入
	* KMessageLogService负责消息日志的查询、插入
        
* 数据访问层：
    * 数据访问层负责与数据库Sqlite3进行交互，以便从数据库中获取和存储数据
	* KSQLiteManager采用单例模式实现，提供数据库访问的功能，连接、初始化、创建表格、查询数据、插入数据


各自类的实现如下，三层架构设计清晰。
数据访问层为单例模式，业务逻辑层作为全局数据的私有成员，可在消息生成时，即插即用。

![alt text](./readme/image-3.png)


# 四、各模块功能实现思路

根据软件需求，对功能进行划分，大致划分为如下几个模块，各自实现思路如下。

## 4.1 模块1：用户根据需求进行配置
针对此功能，考虑到实际使用体验，设计一个监控目录配置窗口，用户点击新增或者双击当前监控目录，都可以对监控路径进行配置，适应用户需求。
定义数据结构KWatchedDirectoryInfo，存储监控目录信息与对应配置信息；自定义类KFileWidget，继承QDialog，实现配置窗口，用户根据自身需求，可以在此对监控目录进行配置并保存配置信息；工具栏提供导出配置与导出配置功能，自定义类KConfigManager，用户可以导出当前监控配置，与导入历史配置。配置信息保存成json文件。

## 4.2 模块2：文件监控与管理

### 4.2.1 模块设计思路
分析需求，需要实现监听、多文件监听、监听事件配置（排除、包含、事件等）。实现思路如下：

自定义类KFileMonitor，通过传入监控文件信息，实现对文件监控，提供开始与暂停槽函数。

```C++
    void startMonitoringSlot();
    void stopMonitoringSlot();
    KWatchedDirectoryInfo* m_currentInfo;
```

自定义类KFileMonitorManager，属于业务逻辑层，负责响应UI的操作，创建监控对象，管理监控对象，提供暂停、删除
、编辑、开始监控等功能。每当用户添加一个新的监控路径，创建一个新的监控对象，并创建新的子线程，将监控任务分配给子线程进行处理，由Qlist统一管理新创建的对象。

```C++
    QList<KFileMonitor*> m_pMonitors;
    QList<QThread*> m_pMonitorThreads;
```

具体类间关系图如下：

![alt text](./readme/image-4.png)

### 4.2.2 监控与监控信息处理流程
监控信息处理流程划分为三步：
1、前处理，在ReadDirectoryChangesW函数就配置是否监听子目录，并通过SetMonitorType(dwNotifyFilter, m_currentInfo->getType());设置监听事件的类型。
2、ReadDirectoryChangesW读取文件变化信息。
3、后处理通过自定义函数processChangeNotifications，过滤用户不需要的信息。
下述详细分析，监控信息处理流程。

* 开始监控，根据配置信息，创建对应监控路径的句柄。

```C++
void KFileMonitor::createHandle()
{
    QString directoryPath = m_currentInfo->getDirectory(); // 监控文件的路径

    m_hDir = CreateFileW(
        reinterpret_cast<LPCWSTR>(directoryPath.utf16()),
        FILE_LIST_DIRECTORY,
        FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE,
        nullptr,
        OPEN_EXISTING,
        FILE_FLAG_BACKUP_SEMANTICS | FILE_FLAG_OVERLAPPED,
        nullptr
    );

}
```

* 调用windows api `ReadDirectoryChangesW`去获取文件动态信息，调用后处理函数`processChangeNotifications`对用户所需要的消息进行过滤。

```C++
void KFileMonitor::monitorDirectory()
{
    createHandle(); // 创建文件句柄
	... 代码省略
    // 设置监控事件类型，仅文件or文件夹
    SetMonitorType(dwNotifyFilter, m_currentInfo->getType());

    while (m_monitoring)
    {
        if (ReadDirectoryChangesW(
            m_hDir,
            &buffer,
            sizeof(buffer),
            m_currentInfo->isWatchingSubdirectories(), //是否监控子文件夹
            dwNotifyFilter, // 过滤条件-- 文件 -- 文件夹
            &bytesReturned,
            nullptr,
            nullptr))
        {
            if (bytesReturned > 0) //如果检测的回应大于0，才执行过滤函数
				processChangeNotifications(buffer, bytesReturned); // 过滤事件，增删改等
        }
		... 代码省略
    }
}
```

* 后处理，过滤不需要的信息，例如，通过`if (shouldIncludeExtention(filename) && !shouldExcludeExtention(filename) && shouldIncludeFile(entireFilePath) && !shouldExcludeFile(entireFilePath))`过滤包含、排除的后缀文件信息，过滤排除、包含指定文件，并调用`KGlobalData::getGlobalDataIntance()->getMessageQue().put(eventMsg)`;将收取到的信息添加到全局的消息队列，后续`KLogView`会定时定量取消息显示到UI。

```C++
void KFileMonitor::processChangeNotifications(char* buffer, DWORD bytesReturned)
{
    FILE_NOTIFY_INFORMATION* pInfo = reinterpret_cast<FILE_NOTIFY_INFORMATION*>(buffer);

    while (pInfo != nullptr)
    {
        char fileName[MAX_PATH];
        memset(fileName, 0, MAX_PATH);
        WideCharToMultiByte(CP_ACP, 0, pInfo->FileName, pInfo->FileNameLength / sizeof(WCHAR), 
            fileName, MAX_PATH, nullptr, nullptr);
        QString filename = QString::fromLocal8Bit(fileName);  // 转换为string
		... 代码省略

        if (shouldIncludeExtention(filename) && !shouldExcludeExtention(filename) &&
            shouldIncludeFile(entireFilePath) && !shouldExcludeFile(entireFilePath))
        {
				... 代码省略
                KGlobalData::getGlobalDataIntance()->getMessageQue().put(eventMsg);

            }
        }

        // 下一个通知
        if (pInfo->NextEntryOffset == 0)
            break;

        pInfo = reinterpret_cast<FILE_NOTIFY_INFORMATION*>(reinterpret_cast<char*>(pInfo) + pInfo->NextEntryOffset);
    }
}
```

* 暂停监控、恢复监控、编辑监控目录配置、删除监控目录

	鼠标右键点击监控目录，弹出上下文菜单栏，选取对应操作。右键会传递点击目录的对应index，通过这个信息找到对应目录文件，进行后续操作。

![alt text](./readme/image-5.png)

收到UI操作的信号之后，逻辑层面由KFileMonitorManager统一处理，停止操作会暂停监视器，通过暂停循环的方式，并且也会将线程安全退出，但不会析构线程和监视器对象；编辑监控目录配置会将新的配置信息设置到监视器，并重启监视器；删除监控目录会删除监视器对象与线程，并移除配置信息，防止内存泄漏。

```C++
void KFileMonitorManager::stopSelectedMonitor(const int index)
{
    m_pMonitors[index]->stopMonitoring();
    m_pMonitors[index]->m_currentInfo->setWatchStatus(KGlobalData::KWatchStatus::Pause); 
    m_pMonitorThreads[index]->quit();
    m_pMonitorThreads[index]->wait();
}

void KFileMonitorManager::startSelectedMonitor(const int index)
{
    m_pMonitorThreads[index]->start();
    m_pMonitors[index]->m_currentInfo->setWatchStatus(KGlobalData::KWatchStatus::Active); // 更新
}

void KFileMonitorManager::deleteSelectedMonitor(const int index)
{
    m_pMonitors[index]->stopMonitoring();
    m_pMonitorThreads[index]->quit();
    m_pMonitorThreads[index]->wait();
    // 删除掉
    KGlobalData::getGlobalDataIntance()->deleteSelectData(index);
    delete m_pMonitors[index];
    delete m_pMonitorThreads[index];
    m_pMonitors.removeAt(index);
    m_pMonitorThreads.removeAt(index);
}
```

## 4.3 模块3：监听目录信息显示
监听目录信息显示数据量不大，考虑代码扩展性与UI设计美观，采用MVD模式。通过给数据模型设置数据，更新目录信息显示到UI，并给UI重写了鼠标双击事件，右键菜单栏，用户根据需求可查看与编辑监听目录信息。

## 4.4 模块4：监听消息显示
分析监听消息显示需求，监听消息具有频繁更新且数据量大特性。故采用MVD设计模式。为了处理大量数据，**保证消息不丢失，且响应快**，采用**消息队列**方式取更新与获取数据，定义线程安全的消息队列, 便于多线程存储与读取消息数据。

```C++
template <typename T>
class KMessageQueue
{
public:
    KMessageQueue() {}
    void put(const T& value)
    {
        QMutexLocker locker(&m_mutex);
        m_queue.enqueue(value);
        m_condition.wakeOne(); 
    }
    T take()
    {
        QMutexLocker locker(&m_mutex);
        while (m_queue.isEmpty())
        {
            m_condition.wait(&m_mutex);
        }
        return m_queue.dequeue();
    }
	......
    QQueue<T> m_queue;
    mutable QMutex m_mutex;
    QWaitCondition m_condition;
};
```

有了基础消息队列数据结构，即可高效完成下列数据传输。
具体显示获取与显示流程如下：
1、消息队列数据添加，每个监视器在监视的后处理过程中都会通过`KGlobalData::getGlobalDataIntance()->getMessageQue().put(eventMsg);`函数，拿到锁之后，会将数据放入到全局消息队列。
2、消息获取，KLogViewer中设置了定时器，定时检查消息队列是否有数据存在，并通过`KEventMessage eventMsg = KGlobalData::getGlobalDataIntance()->getMessageQue().take();`获取数据。

```C++
        m_pUpdateTimer = new QTimer(this);
        connect(m_pUpdateTimer, &QTimer::timeout, this, &KLogViewer::processQueue);
        m_pUpdateTimer->start(100);  // 每100ms处理一次队列中的数据

	void KLogViewer::processQueue()
	{
		...
		QVector<KEventMessage> batchEntries;
		{
			int batchSize = 100;  // 每次处理100条消息
			while (!KGlobalData::getGlobalDataIntance()->getMessageQue().isEmpty() && batchSize > 0)
			{
				KEventMessage eventMsg = KGlobalData::getGlobalDataIntance()->getMessageQue().take();
				batchEntries.append(eventMsg);
				batchSize--;
			}
		}
		...
	}
```
3、视图更新，通过`addLogEntries`将获取到的消息批量更新到数据模型，同时完成消息视图更新。

```C++
    beginInsertRows(QModelIndex(), firstRow, lastRow);
    for (const auto& entry : logEntries)
    {
        m_logEntries.append({ entry.eventType, entry.timestamp, entry.fileSize, entry.filePath });
    }
    endInsertRows();
```

## 4.5 模块5：日志管理
日志的显示与用户交互采用独立对话框，非模态，可与主界面同时进行操作。
### 4.5.1 消息日志
项目亮点中已经对消息日志的存储的代码架构设计与实现进行了介绍，此处不在赘述。此处主要描述消息日志的查询与关键字搜索。本项目日志存储使用了sqlite3数据库，故查询仅需要使用sql语句即可完成。为了保证搜索的高效，用户可先根据操作时间与监控消息类型进行初筛，初筛会从数据库中查询到对应数据，缩小检索范围，后续关键字搜索可以提高效率。

![alt text](./readme/image-6.png)

* 查询
	通过消息日志的服务类，调用数据库的查询函数,根据操作时间与变化类型进行筛选。

```C++
QVector<KEventMessage> KSQLiteManager::queryMessageRecord(const QDateTime& startTime, const QDateTime& endTime, const QString& eventType)
{
    QVector<KEventMessage> result;
    QString querySQL = "SELECT eventType, timestamp, fileSize, filePath FROM MessageLogs WHERE timestamp BETWEEN ? AND ?";
    if (eventType == QString::fromLocal8Bit("重命名"))
        querySQL += " AND eventType IN (?, ?)";
    else if (!eventType.isEmpty())
        querySQL += " AND eventType = ?";
	....
    while (query.next())
    {
        KEventMessage message;
        message.eventType = query.value("eventType").toString();
		....
        result.append(message);
    }

    return result;
}
```

* 关键字搜索
	关键字搜索的初始数据来源于筛选后的数据，搜索的实现思路如下：
	采用了QT的搜索引擎，自定义`KSortFilterProxyModel`模型，继承`QSortFilterProxyModel`，将前面自定义的消息数据模型设置为原模型。通过`setFilterFixedString`函数，将关键字设置上去即可完成搜索，经过测试，搜索效率比较高。

	```C++
		const QString& text = m_pLineEditMes->text();
		m_pLogViewer->m_pProxyModel->setFilterFixedString(text);
		m_pLogViewer->m_pModel->setSearchKeyword(text);

	```
	补充:最初完成搜索功能的时候，没有采用QT的QSortFilterProxyModel模型，而是自定义的搜索方案。查询资料，发现**倒排索引**的搜索方式效率较高，也完成了代码实现，KMessageLogSearcher。（代码未采用，但保留了cpp）
	倒排索引思路：
	1、将所有数据遍历，根据自己的单词划分规则，先构建倒排索引表。
	2、多线程去根据关键字搜索，得到对应索引结果。
	该方法的缺点就是如果数据更新，就需要重新构建倒排索引表，而构建过程耗时较长，经过测试11W条数据，构建时间是55s。优势在于，构建完成倒排索引表之后，后续的搜索会比较快。

	经过与QT的搜索方法对比之后，弃用倒排索引的方案，QT的搜索的方案平均效率较高。

* 排序
排序功能的实现较为简单，点击列表头即可，同样通过KSortFilterProxyModel实现，但原本的排序方案无法满足按数字大小排序，故重新了虚函数`virtual bool lessThan(const QModelIndex& left, const QModelIndex& right) const override; `可以对数字完成排序。

### 4.5.2 操作日志

分析整体的软件特性，明确操作日志所需要达到的目标----记录所有用户在系统中的**操作过程和操作结果**，如新增目录、编辑目录等。操作日志主要是为用户服务，帮助他们查看历史操作记录，因此对可读性要求较高，故定义`KOperationLogInfo`操作日志数据结构，记录操作时间、操作人、操作模块、操作类型、操作详细信息。

划分为三个操作模块：
* 操作模块
    * 目录管理
    * 配置管理
    * 监视管理

具体如下：
* 目录管理
    * 打开监控目录
    * 新增监控目录
    * 编辑监控目录

* 配置管理
    * 导出配置
    * 导入配置

* 监视管理
    * 开始
    * 暂停
    * 删除

根据需求定义操作日志数据结构，操作日志的添加与查询的逻辑同消息日志。都是通过全局变量获取到相应的服务类，从而完成日志插入与查询：

```C++
KGlobalData::getGlobalDataIntance()->getOperationLogService()->addLog(logInfo);
KGlobalData::getGlobalDataIntance()->getOperationLogService()->queryLogs(startTime, endTime, module, user);
```

## 4.6 模块6：最小化托盘

需求：最小化托盘,并且监听到文件变化的时候有消息提示。
自定义`KSystemTray`类，创建托盘图标和托盘的右键菜单，通过提供函数，当主窗口不可见的时候，会展示相应的文件变动信息。

```C++
void showMessage(const KEventMessage& eventMessage, QSystemTrayIcon::MessageIcon icon = QSystemTrayIcon::Information, int msecs = 3000);
```

类间关系图：
![alt text](./readme/image-7.png)

## 4.7 补充
### 4.7.1 UI设计--提高用户体验

![alt text](./readme/image-9.png)
* 设计1：监控状态通过“红绿灯”的方式显示，用户可以更直观的感受到当前目录的监控状态。
* 设计2：监控消息消失栏，右上角提高两个字体大小调节按钮，用户可以自由调节字体大小，满足不同需求。
* 设计3：主界面提高操作帮助入口，为软件的使用提供简单的入门指引。

![alt text](./readme/image-10.png)

* 设计4：查询消息时，表头会有排序类型显示，通过红色与绿色箭头表示当前列的排序方式，更直观。
* 设计5：关键字搜索，会对搜索到的关键字进行黄色高亮显示，便于用户找到对应结果。

### 4.7.2 解决监控性能瓶颈（压力测试界面卡顿问题）

方案：监控多个目录，涉及到多线程竞争资源等问题，为保证消息准确传递，不丢失，不重复，采用线程安全的消息队列存储信息。



# 五、性能优化与内存分析

## 5.1 性能分析与优化

* 测试背景：同时监控两个文件目录，通过脚本各产生5W条消息，共计10w条消息，测试系统性能。

使用vs探测器，观察程序性能，找到相关热点函数，在收集到监控信息的后处理过程中，发现`QDateTime::currentDateTime().toString()`性能消耗较大。优化掉此函数，直接传原始时间信息`QDateTime`，尽可能少的调用toString()。

![alt text](./readme/image-12.png)
![alt text](./readme/image-11.png)

## 5.2 内存泄漏检测
使用vld检测工具，测试时发现两处内存泄漏，一个是监视器对象删除后，内存泄漏；一个是数据库管理的单例类创建时，未析构。
![alt text](./readme/image-13.png)
![alt text](./readme/image-14.png)

解决方案：
手动释放监视器。
![alt text](./readme/image-15.png)
单例采用静态变量实现。
![alt text](./readme/image-16.png)

成功解决内存泄漏。
![alt text](./readme/image-17.png)

## 5.3 内存增长测试

* 步骤1，添加监视文件目录，创建监视器。
* 步骤2，测试产生1w条信息，消息日志增长，qvector数据增长。
* 步骤3，清除屏幕，即删除logmodel中容器的数据，堆大小下降。

![alt text](./readme/image-19.png)
![alt text](./readme/image-18.png)

# 六、项目中遇到的问题与解决方案

## 6.1 消息队列消息“异常丢失”
**问题描述：**
第一次实现消息队列时，确定使用的是线程安全的消息队列，但多次测试1w条消息，UI更新总会消息丢失。

**解决思路：** 
通过在`void KLogViewer::processQueue()`打印线程，发现有两个线程在取消息；同时打印取的消息总数量，发现会交替打印两种数量，数量相加刚好1w条。由此确定消息并没有丢失，而是被另外一个线程取走，奇怪点在于我取消息明明没有创建线程，却多出一个线程。 仔细调试，发现由于KLogViewer是主界面和查询日志复用类，也就是，创建了两个`KLogViewer`对象，同时在取消息。至此，找到bug，通过设置是否是查询框的方式，选择是否实例化`qtimer`，也就是查询框不会参数消息处理。
`bool m_isQueryLog; // 判断是否是用作查询对话框的显示，如果是，则不参与线程抽取数据操作`

## 6.2 数据库插入日志，软件卡顿
**问题描述：**
使用消息队列，并定时读取，软件不卡顿，但加入日志数据库插入操作后，软件卡顿。
**解决思路：** 
性能分析，发现数据库IO会比较耗时，而之前是单条消息插入数据库，改为数据库批量插入，成功解决。


# 五、未来优化与扩展思路
