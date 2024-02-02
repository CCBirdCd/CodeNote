<a name="faULy"></a>
## [SQLite3接入UE.docx](https://www.yuque.com/attachments/yuque/0/2022/docx/26747865/1667206560092-de8f81bc-9baf-4b8e-a270-dae1c779c685.docx?_lake_card=%7B%22src%22%3A%22https%3A%2F%2Fwww.yuque.com%2Fattachments%2Fyuque%2F0%2F2022%2Fdocx%2F26747865%2F1667206560092-de8f81bc-9baf-4b8e-a270-dae1c779c685.docx%22%2C%22name%22%3A%22SQLite3%E6%8E%A5%E5%85%A5UE.docx%22%2C%22size%22%3A596518%2C%22type%22%3A%22application%2Fvnd.openxmlformats-officedocument.wordprocessingml.document%22%2C%22ext%22%3A%22docx%22%2C%22source%22%3A%22%22%2C%22status%22%3A%22done%22%2C%22download%22%3Atrue%2C%22taskId%22%3A%22u312c5ba2-ad22-49be-813b-ab5cb900249%22%2C%22taskType%22%3A%22upload%22%2C%22__spacing%22%3A%22both%22%2C%22id%22%3A%22u3ad654ec%22%2C%22margin%22%3A%7B%22top%22%3Atrue%2C%22bottom%22%3Atrue%7D%2C%22card%22%3A%22file%22%7D)
<a name="CMjcD"></a>
## 1 下载源码

1. 首先下载sqlite需要的文件，去Sqlite官方下载源码[https://www.sqlite.org/download.html](https://www.sqlite.org/download.html)，总共下载两个文件，一个是Source Code下面的sqlite-amalgamation-3300100，以及Precompiled Binaries for Windows下面的sqlite-dll-win64-x64-3300100(3300100 为版本号，可选择最新版本)

![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334479967-50162cfa-7b4b-4f50-af18-7b745e6ad249.png#id=kjiBg&originHeight=195&originWidth=676&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334480574-eaeb66c7-23cb-4d02-aad8-07bfdbf2e6e9.png#id=Ua3Qc&originHeight=165&originWidth=734&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
<a name="Mb56O"></a>
## 2 将sqlite库编译成UE插件

1. 进入UE编辑器插件界面，新建空白插件，将 sqlite3.h    sqlite3ext.h 拷贝到插件源目录下Public文件夹下，将  sqlite3.c  文件拷贝到Private文件夹下

1. 在 插件名.Build.cs 目录下添加如下代码：

using _System_;<br />// 避免在VS编译时出现C4668错误<br />     bEnableUndefinedIdentifierWarnings = false;

1. 进入UE编辑器新建c++类 继承于 UObject，结构图如下

![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334481380-e4df5f7a-d5ba-42dc-90d0-0591494ad197.png#id=kxM0Z&originHeight=849&originWidth=1515&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />方框内结构体是为了解决，UE原生TMap，TArray不支持结构嵌套的问题

Cpp结构图如下：<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334482338-b66ae1b5-f123-4d09-a54f-3c6e3725ad1c.png#id=CRwdG&originHeight=742&originWidth=1096&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334482932-b025e4b7-0a73-41b4-b0c5-e39eae96e9c3.png#id=tFrPS&originHeight=501&originWidth=1019&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334483484-ae810b2e-7e88-4f03-9efe-b953c2e5d182.png#id=aMJaa&originHeight=496&originWidth=908&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />有一个问题就是，sqlite3.exec 接口需要传入回调函数，数据结构从第三个参数 void* 传入，如果在匿名函数定义 [] 处 使用 [=] 或 [&] 这类格式的话，接口处提示错误(也可能是我没用好)
<a name="1x9nX"></a>
## 3 工程中引入插件
在 工程名.Build.cs 添加依赖模块，如下图:<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334484064-6942b37a-d4b1-439c-a264-3d4c061bb1cc.png#id=C4kz9&originHeight=642&originWidth=1269&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />编译成功启动UE编辑器后 点击编辑器界面生成 .d.ts 的按钮生成相对应得 ue.d.ts文件，下图代表生成成功<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334484498-f8790253-e46e-443f-8e3a-c785e546ddd3.png#id=Mml5D&originHeight=758&originWidth=859&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
<a name="BLMYe"></a>
## 4 建立 .ts 文件调用接口

1. 在typescript 文件夹下新建 .ts 文件
2. 导入 UE 模块

import * as UE from 'ue'

初始化数据库连接类<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334485085-b5d9e4d7-19ae-46bd-914f-31d6c7d21812.png#id=alewS&originHeight=367&originWidth=594&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />执行sql语句 接口定义格式<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334485440-82b3053e-a3c2-4103-9a08-469fe128f060.png#id=NIIbK&originHeight=182&originWidth=623&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />如果c里定义的接口是返回值<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334485696-6d48ae78-7993-44df-9612-46431c6ac79c.png#id=h85Q8&originHeight=60&originWidth=526&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />则如下方式获取返回数据<br />let execResult = this.ExecSql(sql);

如果c里定义的接口是引用对象<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334486171-88422ff7-4ea4-4c7c-bc69-87f2542be73a.png#id=yb13H&originHeight=56&originWidth=567&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />则如下方式获取返回数据<br />let execResult = UE.NewArray(UE.DBValueMap);<br />this.TSSQLite3.TSsqlite3_exec_ref(sql, $ref(execResult))

<a name="afBQo"></a>
## 5 启动文件监视自动生成 js
用 vscode 打开工程结构，点击终端->运行任务，选择typescript文件夹<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334486769-3e520c77-bf2b-465d-b2b3-30aded137140.png#id=ZPAUw&originHeight=273&originWidth=1299&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />监视下面这个文件<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334487314-1e390e1a-d525-48b8-bc16-3bb5e27c22db.png#id=C0EG7&originHeight=207&originWidth=828&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />文件变化后会自动生成到 Content\JavaScript 目录下
<a name="dcWvC"></a>
## 6 调用/测试方式
原理：自定义GameInstance游戏启动时触发代码，到编辑器 把GameInstace改成自定义的这个 GameInstance 启动就OK了<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334487897-09bbee3c-1263-493e-85a7-c1e96fdcff59.png#id=K7qc9&originHeight=501&originWidth=1339&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/26747865/1650334488777-29005639-9303-43d3-bf79-dca7f22cd783.png#id=WFe7O&originHeight=452&originWidth=1387&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
