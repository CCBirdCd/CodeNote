# 自动喝药
AutoUse:=0

$F4::  ;自动喝药开关
AutoUse := AutoUse!
Return
$F5::   ;自动喝药启动
Loop
{
    if WinActive("ahk_class POEWindowClass") or WinActive("ahk_class" "POEWindowClass")
    { 
        if(AutoUse)
        {
            send 1 ;喝血药快捷键
        }
        Sleep 4000  ;延时多久检测一次
    }
}