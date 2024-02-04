## [SQLite3接入UE.docx](https://www.yuque.com/attachments/yuque/0/2022/docx/26747865/1667206560092-de8f81bc-9baf-4b8e-a270-dae1c779c685.docx?_lake_card=%7B%22src%22%3A%22https%3A%2F%2Fwww.yuque.com%2Fattachments%2Fyuque%2F0%2F2022%2Fdocx%2F26747865%2F1667206560092-de8f81bc-9baf-4b8e-a270-dae1c779c685.docx%22%2C%22name%22%3A%22SQLite3%E6%8E%A5%E5%85%A5UE.docx%22%2C%22size%22%3A596518%2C%22type%22%3A%22application%2Fvnd.openxmlformats-officedocument.wordprocessingml.document%22%2C%22ext%22%3A%22docx%22%2C%22source%22%3A%22%22%2C%22status%22%3A%22done%22%2C%22download%22%3Atrue%2C%22taskId%22%3A%22u312c5ba2-ad22-49be-813b-ab5cb900249%22%2C%22taskType%22%3A%22upload%22%2C%22__spacing%22%3A%22both%22%2C%22id%22%3A%22u3ad654ec%22%2C%22margin%22%3A%7B%22top%22%3Atrue%2C%22bottom%22%3Atrue%7D%2C%22card%22%3A%22file%22%7D)
## 1 下载源码

1. 首先下载sqlite需要的文件，去Sqlite官方下载源码[https://www.sqlite.org/download.html](https://www.sqlite.org/download.html)，总共下载两个文件，一个是Source Code下面的sqlite-amalgamation-3300100，以及Precompiled Binaries for Windows下面的sqlite-dll-win64-x64-3300100(3300100 为版本号，可选择最新版本)

![[编程/UE/功能模块/Image/SQLite3接入UE/83caf4f3451926b99f7cdb2dd3863ad4_MD5.png]]
![[编程/UE/功能模块/Image/SQLite3接入UE/2962dc5194d018f546ba41c75b4fe04f_MD5.png]]
## 2 将sqlite库编译成UE插件

1. 进入UE编辑器插件界面，新建空白插件，将 sqlite3.h    sqlite3ext.h 拷贝到插件源目录下Public文件夹下，将  sqlite3.c  文件拷贝到Private文件夹下

1. 在 插件名.Build.cs 目录下添加如下代码：

using _System_;
// 避免在VS编译时出现C4668错误
     bEnableUndefinedIdentifierWarnings = false;

1. 进入UE编辑器新建c++类 继承于 UObject，结构图如下

![[编程/UE/功能模块/Image/SQLite3接入UE/dd6d1712d5c7ea420f7d016a188816e0_MD5.png]]
方框内结构体是为了解决，UE原生TMap，TArray不支持结构嵌套的问题

Cpp结构图如下：
![[编程/UE/功能模块/Image/SQLite3接入UE/219a1b39b8ecf8cc985fbd4e81b73559_MD5.png]]
![[编程/UE/功能模块/Image/SQLite3接入UE/069cccea43cbe0ea34c175859df1fbc7_MD5.png]]
![[编程/UE/功能模块/Image/SQLite3接入UE/ce1bc495964fb769050a9e2fc3de5882_MD5.png]]
有一个问题就是，sqlite3.exec 接口需要传入回调函数，数据结构从第三个参数 void* 传入，如果在匿名函数定义 [] 处 使用 [=] 或 [&] 这类格式的话，接口处提示错误(也可能是我没用好)
## 3 工程中引入插件
在 工程名.Build.cs 添加依赖模块，如下图:
![[编程/UE/功能模块/Image/SQLite3接入UE/7e05d8f6bc6ff63232f60f68b923dbc0_MD5.png]]
编译成功启动UE编辑器后 点击编辑器界面生成 .d.ts 的按钮生成相对应得 ue.d.ts文件，下图代表生成成功
![[编程/UE/功能模块/Image/SQLite3接入UE/b15037a7345f498a4e5bb327bfcdcf1c_MD5.png]]
## 4 建立 .ts 文件调用接口

1. 在typescript 文件夹下新建 .ts 文件
2. 导入 UE 模块

import * as UE from 'ue'

初始化数据库连接类
![[编程/UE/功能模块/Image/SQLite3接入UE/e3b1764485e0b3c7fb9f7eb3eadbfb94_MD5.png]]
执行sql语句 接口定义格式
![[编程/UE/功能模块/Image/SQLite3接入UE/a67e313c8d28e02a125b518d94da4b27_MD5.png]]
如果c里定义的接口是返回值
![[编程/UE/功能模块/Image/SQLite3接入UE/8e55eaadf85d62d2c82134dc54b6fbdb_MD5.png]]
则如下方式获取返回数据
let execResult = this.ExecSql(sql);

如果c里定义的接口是引用对象
![[编程/UE/功能模块/Image/SQLite3接入UE/06ce1027f32c9ff570ff0624c2fbae2f_MD5.png]]
则如下方式获取返回数据
let execResult = UE.NewArray(UE.DBValueMap);
this.TSSQLite3.TSsqlite3_exec_ref(sql, $ref(execResult))

## 5 启动文件监视自动生成 js
用 vscode 打开工程结构，点击终端->运行任务，选择typescript文件夹
![[编程/UE/功能模块/Image/SQLite3接入UE/7a4f882c9f390dd815a6aabad8e56d17_MD5.png]]
监视下面这个文件
![[编程/UE/功能模块/Image/SQLite3接入UE/356b47c54f396088370b241f21f772a4_MD5.png]]
文件变化后会自动生成到 Content\JavaScript 目录下
## 6 调用/测试方式
原理：自定义GameInstance游戏启动时触发代码，到编辑器 把GameInstace改成自定义的这个 GameInstance 启动就OK了
![[编程/UE/功能模块/Image/SQLite3接入UE/6a660876e55e27b2c38b45d12f59a124_MD5.png]]
![[编程/UE/功能模块/Image/SQLite3接入UE/08acddf5b85b4b036facf963479e3749_MD5.png]]
