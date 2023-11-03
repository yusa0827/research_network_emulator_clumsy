# windowsアプリでネットワークアプリ clumsy


# install できるようにしたい　　
clumsy のフォルダを見ると、 genie.lua があった。　　
何者かはわからないため、調べてみる　　
  
## Lua　インストール  
[参考]  
Luaのインストールとhello world(Windows10)  
https://itsakura.com/lua-install  
  
インストールの方法がある。  
１．公式のページは、以下ですが分かりづらいので  
http://www.lua.org/  

joedfさんのLua Buildsのページを開きます  
https://joedf.ahkscript.org/LuaBuilds/  
  
lua-5.4.6_Win64_bin.zip  
をダウンロードし、展開する  
  
Cドライブ直下に lua-5.4.6_Win64_bin を配置する   
  
環境変数に下記を追加する  
C:\lua-5.4.6_Win64_bin  
  
  
参考サイト  
３．コマンドプロンプトを起動し、上記フォルダ配下に移動して、lua ファイル名と入力するとhello worldが表示されます。  
lua ファイル名  
  
参考サイトから、 lua ファイル名　でファイルを実行できるらしい  
  
>cd C:\Users\sasak\Desktop\Network_Emulator_clumsy\clumsy  
>lua --version  
lua の情報が出力さればOK  
ビルドを実行してみる  
>lua genie.lua  
C:\Users\sasak\Desktop\Network_Emulator_clumsy\clumsy>lua genie.lua  
lua: genie.lua:29: attempt to call a nil value (field 'getcwd')  
stack traceback:  
        genie.lua:29: in main chunk  
        [C]: in ?  
  
C:\Users\sasak\Desktop\Network_Emulator_clumsy\clumsy>  
  
エラーが出ているんだが。。。  
よくわからないため、ビルドはあきらめて、ソースコードの解析をする  
  
  
  
  
# ソースコードを解析  
  
## main 関数を見つけ出す  
git grep -i main  
----------------------------------------  
src/main.c:    // ! main loop won't return until program exit  
src/main.c:int main(int argc, char* argv[]) {  
  
→  
src/main.c が存在する  
  
  
  
## main.c を確認  
  
L.5  
#include <Windows.h>  
#include "iup.h"  
#include "common.h"  
  
→  
自作のヘッダーをインクルードしている  
  
  
L.504  
int main(int argc, char* argv[]) {  
    LOG("Is Run As Admin: %d", IsRunAsAdmin());  
    LOG("Is Elevated: %d", IsElevated());  
    init(argc, argv);  
    startup();  
    cleanup();  
    return 0;  
}  
  
### main.c IsRunAsAdmin() を確認  
IsRunAsAdmin()  
はどこで定義されているのか  
  
git grep IsRunAsAdmin  
----------------------------------------  
src/common.h:BOOL IsRunAsAdmin();  
src/elevate.c:BOOL IsRunAsAdmin()  
  
→  
src/common.h に宣言されてい、  
src/elevate.c を定義されていると予測できる  
  
elevate： 【他動】持ち上げる、昇進させる  
  
##### common.h IsRunAsAdmin()  
  
#include "iup.h"  
#include "windivert.h"  
  
→  
自作のヘッダーをインクルードしている  
IsRunAsAdmin() を調べる前に、#include "windivert.h"　が何者なのかを調べる  
  
  
  
AllocateAndInitializeSid()  
CheckTokenMembership()  
FreeSid()  
を使用  
  
##### common.h AllocateAndInitializeSid() CheckTokenMembership()  
  
> git grep AllocateAndInitializeSid  
src/elevate.c:    if (!AllocateAndInitializeSid(  
  
clumsy には src/elevate.c 以外に存在しない  
であれば、 #include "windivert.h" に存在するということか  
  
AllocateAndInitializeSid()  
AllocateAndInitializeSid 関数 (securitybaseapi.h)  
https://learn.microsoft.com/ja-jp/windows/win32/api/securitybaseapi/nf-securitybaseapi-allocateandinitializesid  
  
インターネットで検索したら、securitybaseapi.h　で定義されていた  
  
CheckTokenMembership()  
CheckTokenMembership 関数 (securitybaseapi.h)  
https://learn.microsoft.com/ja-jp/windows/win32/api/securitybaseapi/nf-securitybaseapi-checktokenmembership  
インターネットで検索したら、securitybaseapi.h　で定義されていた  
  
FreeSid()  
FreeSid 関数 (securitybaseapi.h)  
https://learn.microsoft.com/ja-jp/windows/win32/api/securitybaseapi/nf-securitybaseapi-freesid  
インターネットで検索したら、securitybaseapi.h　で定義されていた  
  
  
  
##### common.h IsElevated()  
OpenProcessToken()  
OpenProcessToken 関数 (processthreadsapi.h)  
GetCurrentProcess 関数 (processthreadsapi.h)  
Kernel32.lib  
GetTokenInformation 関数 (securitybaseapi.h)  
  
  
## src/main.c init()  
L.122  
void init(int argc, char* argv[]) {  
で定義されていた  
  
loadConfig();  
→  
external/iup-3.30_Win64_mingw6_lib/include/iup_plus.h:    int LoadConfig() { return IupConfigLoad(ih); }  
で宣言  
  
  
  
## src/main.c startup()  
  
L.263  
    // initialize seed  
    srand((unsigned int)time(NULL));  
  
    // kickoff event loops  
    IupShowXY(dialog, IUP_CENTER, IUP_CENTER);  
    IupMainLoop();  
  
IupMainLoop();  
が呼ばれている。。。  
  
main関数からの追跡はこれ以上できなかった。。。  
  
  
## src/bandwidth.c の関数がどこで使われているかを確認する  
  
static short bandwidthProcess(PacketNode *head, PacketNode* tail)  
  
  
//---------------------------------------------------------------------  
// module  
//---------------------------------------------------------------------  
Module bandwidthModule = {  
    "Bandwidth",  
    NAME,  
    (short*)&bandwidthEnabled,  
    bandwidthSetupUI,  
    bandwidthStartUp,  
    bandwidthCloseDown,  
    bandwidthProcess,  
    // runtime fields  
    0, 0, NULL  
};  
  
  
## src/main.c の関数がどこで使われているかを確認する  
  
// ! the order decides which module get processed first  
Module* modules[MODULE_CNT] = {  
    &lagModule,  
    &dropModule,  
    &throttleModule,  
    &dupModule,  
    &oodModule,  
    &tamperModule,  
    &resetModule,  
	&bandwidthModule,  
};  
  
## src/common.h の関数がどこで使われているかを確認する  
  
  
extern Module lagModule;  
extern Module dropModule;  
extern Module throttleModule;  
extern Module oodModule;  
extern Module dupModule;  
extern Module tamperModule;  
extern Module resetModule;  
extern Module bandwidthModule;  
extern Module* modules[MODULE_CNT]; // all modules in a list  
  
// module  
typedef struct {  
    /*  
     * Static module data  
     */  
    const char *displayName; // display name shown in ui  
    const char *shortName; // single word name  
    short *enabledFlag; // volatile short flag to determine enabled or not  
    Ihandle* (*setupUIFunc)(); // return hbox as controls group  
    void (*startUp)(); // called when starting up the module  
    void (*closeDown)(PacketNode *head, PacketNode *tail); // called when starting up the module  
    short (*process)(PacketNode *head, PacketNode *tail);  
    /*  
     * Flags used during program excution. Need to be re initialized on each run  
     */  
    short lastEnabled; // if it is enabled on last run  
    short processTriggered; // whether this module has been triggered in last step   
    Ihandle *iconHandle; // store the icon to be updated  
} Module;  
  
  
# "iup.h" "windivert.h"    
    
WinDivert    
http://reqrypt.org/windivert.html    
    
iup    
https://www.tecgraf.puc-rio.br/iup/    
2. Hello World    
https://www.tecgraf.puc-rio.br/iup/    
C/C++ の GUI     
    
clumsy は iup と WindDivert の2つのライブラリを利用しているらしい    
関数の定義は WindDivert あるかもしれない    
    
Divert/README    
https://github.com/basil00/Divert    
    
  
The basic architecture of WinDivert is as follows:  

```
                              +-----------------+
                              |                 |
                     +------->|    PROGRAM      |--------+
                     |        | (WinDivert.dll) |        |
                     |        +-----------------+        |
                     |                                   | (3) re-injected
                     | (2a) matching packet              |     packet
                     |                                   |
                     |                                   |
 [user mode]         |                                   |
 ....................|...................................|...................
 [kernel mode]       |                                   |
                     |                                   |
                     |                                   |
              +---------------+                          +----------------->
  (1) packet  |               | (2b) non-matching packet
 ------------>| WinDivert.sys |-------------------------------------------->
              |               |
              +---------------+
```

The WinDivert.sys driver is installed below the Windows network stack.  The
following actions occur:  
  
(1) A new packet enters the network stack and is intercepted by WinDivert.sys  
(2a) If the packet matches the PROGRAM-defined filter, it is diverted.  The  
    PROGRAM can then read the packet using a call to WinDivertRecv().  
(2b) If the packet does not match the filter, the packet continues as normal.  
(3) PROGRAM either drops, modifies, or re-injects the packet.  PROGRAM can  
    re-inject the (modified) using a call to WinDivertSend().  


WinDivert.sys が重要

