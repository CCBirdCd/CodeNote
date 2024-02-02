默认："$(SolutionDir)Incarnation.uproject" -skipcompile<br />调用方式：   "$(SolutionDir)Incarnation.uproject" -run=ImportStringTableCsv

创建文件夹引用：
```cpp
@echo off
cd /d %~dp0
mklink /D "%cd%\CSV" "%cd%\..\config\CSV"
@echo finish
@pause
```

批处理调用：
```cpp
cd /d %~dp0
"..\ue5\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" %~dp0\Incarnation.uproject -run=ImportStringTableCsv
```
