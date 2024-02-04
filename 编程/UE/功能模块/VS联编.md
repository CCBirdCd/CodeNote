# 1 安装 incredibuild

## 1.1 安装
![[编程/UE/功能模块/Image/VS联编/8dedd258a89eaed316699dbc94376080_MD5.png]]
![[编程/UE/功能模块/Image/VS联编/4d20686d1387c7640ff7d8348ca762e2_MD5.png]]
![[编程/UE/功能模块/Image/VS联编/9d25501827981e4f585d42dda598efcc_MD5.png]]
最后 确认桌面有如下图标![[编程/UE/功能模块/Image/VS联编/25fae81f557d96270ce9cc1ccc77070f_MD5.png]]以及打开vs后右键工程时 有 IncrediBuild选项则安装完成，没有图标的话手动搜索启动下
## 1.1 问题解决

### 问题一 编译时提示缺少 English ，package之类的字符
解决方式，修复vs然后在修复界面里的语言包勾选英文语言包，然后全部next操作就行
### 问题二：编译时 输出源IncrediBuild出现下图 0,1的错误
![[编程/UE/功能模块/Image/VS联编/c9270eb04362b3eac1b80228bc6c2492_MD5.png]]
解决方式：
![[编程/UE/功能模块/Image/VS联编/b3aecea1f7dd363a10dcf51fd22dd4e0_MD5.png]]
流程：打开注册表，找到对应文件夹，右键新建一个字符串，修改字符串名称为UseMSBuild,右键这个项点击修改，改为1    结果如下图：![[编程/UE/功能模块/Image/VS联编/7ac64ab4cf5854bde6d00c04599a8355_MD5.png]]
然后关闭注册表重试即可
