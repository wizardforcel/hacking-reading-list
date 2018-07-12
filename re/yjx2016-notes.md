# 一、分析角色 HP/MP 地址

我们的目标是这个，热血江湖。我们要找出基本信息中，所有数据的地址。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3otdfi6j20s10lqtce.jpg)

我们要用到一款工具，CE。打开之后点击左上角打开进程，会弹出进程列表，我们需要选择游戏的进程。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3pge39oj20ix0kp3zt.jpg)

我们可以点击下面的“窗口列表”，然后从打开的窗口中搜索，这样可能比较好找。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3pm6rpej208s0d73z2.jpg)

由于 HP 数值变动较快，我们把扫描类型改为“两者之间的数值”，在上面把数值设置为 352 和 370（视具体情况而定），然后点“首次扫描”。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3prus3wj20iy0kodgy.jpg)

左边就会出现结果。因为结果太多，我们不知道是哪个，需要进一步筛选。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3prtj7hj207t0dhjrt.jpg)

我们把扫描类型改为“增加的数值”，点击“再次扫描”。这个很重要，因为再次扫描是在上一次的结果中搜索，可以缩小范围。游戏中一些值是不变的，可以过滤一些。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3prxxewj20of0d00u5.jpg)

我们双击唯一的结果，它会在底部的列表中出现，双击描述可以修改名称。双击地址会弹出这样一个框。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3pruwiqj20j508tgm7.jpg)

我们可以看到，地址采用模块名称（基址）加偏移来描述。这是因为一些模块是共享库，加载时会改变基址。因为我们这是一个 EXE，不需要这个名称也可以。

接下来我们尝试寻找 MP（蓝的那个）的地址。我们不需要重新搜索。我们假设程序以`int`保存这些数值，而`int`在 windows x86/x64 上是四个字节。我们还假设这些东西是挨着存储的。所以我们将那个地址加四。

![](http://ww1.sinaimg.cn/large/841aea59gy1fmz3prv2luj20t70cmdhj.jpg)

我们看到右边的 251 正好的游戏的 MP，说明我们找对了。

## 二、分析角色金钱基址

我们查看金币数量，是 173061：

![](https://wx3.sinaimg.cn/large/841aea59ly1fody0h0u65j21270m6q6u.jpg)

这个数值不太可能有重的，所以我们直接搜索：

![](https://wx3.sinaimg.cn/large/841aea59ly1fody31ogimj206s09f3ye.jpg)

我们选上面那个，因为它和我们上节课的地址在一个段里面。

我们找到了金钱的地址。但这样一个一个找太麻烦了，有没有可能一次找到全部呢？

首先记下基本信息：

![](https://wx2.sinaimg.cn/large/841aea59ly1fody9ultz2j20bi0c7q3n.jpg)

我们打开 OD 并用它附加游戏。

![](https://wx3.sinaimg.cn/large/841aea59ly1fody54zcxxj20u40iu0wt.jpg)

然后执行`dd 2f86170`，在左下角的窗口中，我们可以看到这个地址附近的数据。

![](https://wx3.sinaimg.cn/large/841aea59ly1fodybxz0b4j20ke0c0t9d.jpg)

我们双击第一行第一列，将第一列转换为偏移形式：

![](https://wx3.sinaimg.cn/large/841aea59ly1fodyeda2iaj20800azt8z.jpg)

我们看到第二列是十六进制形式，需要将其转换为十进制。

| 地址 | 数值 | 属性 |
| --- | --- | --- |
| `(0x2f86170)+0x0` | 381 | HP |
| `0x4` | 252 | MP |
| `0x8` | 390 | 愤怒值 |
| `0xc` | 381 | 最大 HP |
| `0x10` | 252 | 最大 MP |
| `0x14` | 1000 | 最大愤怒值 |
| `0x18 QWORD` | 12185 | 经验值 |
| `0x20 QWORD` | 24782 | 下一级所需的经验值 |
| `0x28` | 10 | ？ |
| `0x2c` | 27138 | 历练 |
| `0x30` | 51 | 心 |
| `0x34` | 54 | 体 |
| `0x38` | 31 | 气 |
| `0x3c` | 91 | 魂 |
| `0x40` | 0 | ？ |
| `0x44` | 0 | ？ |

我们可以根据数值猜出绝大部分。但是 HP 之前还有个昵称，这里没有，可能我们需要向前找找。

我们点击右键，点击“文本->ASCII（32 字符）”：

![](https://wx4.sinaimg.cn/large/841aea59ly1fodz8nzqjyj20bv0660sy.jpg)

在`-0x80`的地方找到了角色名称。我们切换为十六进制视图，然后把这个地方作为新的基址：

![](https://wx2.sinaimg.cn/large/841aea59ly1fodz9103wyj209o0avaae.jpg)

我们得到了一些新的东西：

| 地址 | 数值 | 属性 |
| --- | --- | --- |
| `(0x2f860f0)+0x0 STR` | - | 角色名称 |
| `0x30` | 11 | ？ |
| `0x34 BYTE` | 0x17 | 等级 |
| `0x35 BYTE` | 0x1 | 几转 |
| `0x36 STR` | - | 名声 |


并且之前那些地址需要加上`0x80`，这里就不再写一遍了。

我们回到 CE，可以点击右边的“手动加入地址”，保存它们。

![](https://wx1.sinaimg.cn/large/841aea59ly1fodz9d79fwj20im0bwwfg.jpg)

## 三、分析角色气功加点

这次我们要分析角色的气功点数：

![](https://wx2.sinaimg.cn/large/841aea59ly1foe2eqdatpj20ha07it97.jpg)

我们首先寻找第一个，因为其它气功很可能在第一个后面。

并且，我们不知道这个属性用几个字节来表示。但如果多于一个字节，那么`14`应该在它的最低字节。也就是说，无论怎么表示，我们都可以搜索一个字节`14`。

（实际上气功点数最大为 20，剩余点数最大为 100，不超出一个字节的最大值。就算它多于一个字节，高字节也用不上。）

![](https://wx1.sinaimg.cn/large/841aea59ly1foe2f2itfaj20ix0axdgo.jpg)

搜索结果太多了，我们让它变化一下，给它加一点变成 15，然后再搜。

![](https://wx3.sinaimg.cn/large/841aea59ly1foe2fgb8s9j20j107z74u.jpg)

最上面的两个以`0x02f`开头，和上一节的其它数据在同一个段里面。那么到底哪个是呢？

我们用 OD 附加进程（其它很多软件都可以），查看具体的内存布局。首先是第一个`0x02f861e4`：

![](https://wx4.sinaimg.cn/large/841aea59ly1foe2g1tspdj20ju080jsn.jpg)

`0x02f861e4`是第一个气功点数，每隔`0x4`就有一个气功点数。`0x02f861e0`是剩余点数。所有点数都是一字节。

然后是第二个`0x02f888a0`：

![](https://wx4.sinaimg.cn/large/841aea59ly1foe2gewpw2j20fs02kt8t.jpg)

这个地址中没有剩余点数，而且都是紧密挨着的。

下面我们验证一下，将第二个气功的点数加一。

![](https://wx4.sinaimg.cn/large/841aea59ly1foe2gr3dccj20br066weo.jpg)

这是第一个地址`0x02f861e0`：

![](https://wx1.sinaimg.cn/large/841aea59ly1foe2gyv4e2j20fu02ujrk.jpg)

我们看到第二个气功的点数变成了 2。

然后是第二个地址`0x02f888a0`：

![](https://wx2.sinaimg.cn/large/841aea59ly1foe2h8go5xj20fw01eweg.jpg)

也变了，说明两个地址都有效。我们选择第一个，因为它和我们上一节的基址近一些。我们减一下，得到第一个地址的偏移是`0xf0`。

下面我们总结一下信息：

| 地址 | 数值 | 属性 |
| --- | --- | --- |
| `(0x2f860f0)+0xf0 BYTE` | 2 | 气功剩余点数 |
| `0xf4 BYTE` | 15 | 第一个气功点数 |
| `0xf0+4*i BYTE` | - | 第`i`个气功点数 |

## 四、注入 DLL

一般来说，在同一个进程中读取数据比较方便。所以我们编写 DLL，将其注入同一个进程中。

打开 VS，新建项目，选择“MFC DLL”。创建项目完成后，我们的目录是这样：

![](https://wx3.sinaimg.cn/large/841aea59ly1foeo5i4zfdj208e098dfz.jpg)

接下来我们创建窗口，点击资源视图（左下角），然后右键添加资源对话框（Dialog）：

![](https://wx1.sinaimg.cn/large/841aea59ly1foeo5v6mrmj20qz0dkq3m.jpg)

然后我们新建类`CMainDialogWnd`，使用 MFC 创建类向导：

![](https://wx1.sinaimg.cn/large/841aea59ly1foeo6ag49yj20lc0d9t9d.jpg)

然后打开“源文件->`MainDialogWnd.h`”，代码是这样。


```cpp
class CMainDialogWnd: public CDialogEx
{
    DECLARE_DYNAMIC(CMainDialogWnd)

public:
    CMainDialogWnd(CWnd* pParent=NULL); //标准构造函数
    virtual ~CMainDialogWnd();
    
    //对话框数据
    enum { IDD = IDD_DIALOG1 }

protected:
    virtual void DoDataExchange(CDataExchange* pDX) //DDX/DDV 支持

    DECLARE_MESSAGE_MAP()
}
```

我们打开`MFC_DLL.cpp`，创建全局变量：

```cpp
CMainDialogWnd *PMainDialog;
```

在`CMFC_DLLApp::InitInstance`中添加：

```cpp
PMainDialog = new CMainDialogWnd;
PMainDialog->DoModal();
delete PMainDialog;
// 释放 DLL，以便反复注入
FreeLibraryAndExitThread(theApp.m_hInstance, 1);
```

但这样有个问题，这个窗口是模态的。窗口显示的时候会卡住游戏。我们可以将其放到子线程中。把上面的代码移到一个函数中：


```cpp
DWORD WINAPI ShowDialog(LPARAM lpData) 
{
    PMainDialog = new CMainDialogWnd;
    PMainDialog->DoModal();
    delete PMainDialog;
    // 释放 DLL，以便反复注入
    FreeLibraryAndExitThread(theApp.m_hInstance, 1);
    
    return TRUE;
}
```

在`CMFC_DLLApp::InitInstance`中编写：

```cpp
CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)ShowDialog, NULL, NULL, NULL);
```

我们编译它，在`debug`目录下面得到`MFC_DLL.dll`。然后我们打开`CodeInEx`注入工具，点击左上角的按钮：

![](https://wx3.sinaimg.cn/large/841aea59ly1foeo6ne20jj20nz0k9diw.jpg)

我们首先在上面的列表中选择要注入的进程，然后点击下面的“注入DLL”按钮，会弹出一个选择框。我们在里面选择刚才的 DLL。

之后我们发现我们的窗口打开了，并且游戏还有反应。

## 五、手动编写注入代码

上一节中，我们使用工具来注入 DLL。这一节我们尝试自己编程来实现。

首先新建 Win32 控制台项目，在“源文件”目录下创建`InjectDll.cpp`（名字不重要）。

我们首先要获取窗体类名，之后要拿它获取窗口句柄。为什么这样，是因为窗体类名是永远不变的，句柄可能每次启动都要变。我们打开`Spy++`：

![](https://wx2.sinaimg.cn/large/841aea59ly1foew0r530kj20jq0jwtb0.jpg)

句柄是`D3D Window`。我们在代码开头定义一个常量：

```cpp
#define GameClassName "D3D Window"
```

我们还需要定义 DLL 的路径：

```cpp
#define DllFullPath "path\\to\\your\\dll"
```

之后我们编写函数`InjectDll`：

```cpp
bool InjectDll() 
{
    // 根据窗口类名获得句柄
    HWND hWnd = FindWindow(GameClassName, NULL);
    if(hWnd == NULL) 
        return false;
        
    DWORD pid = 0;
    // 根据窗口句柄获取 PID
    GetWindowThreadProcessId(hWnd, &pid);
    if(pid == 0)
        return false;

    // 根据 PID 获取进程句柄
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if(hProcess == NULL)
        return false;
        
    // 在游戏内存中分配一片空间
    LPVOID address = VirtualAllocEx(hProcess, NULL, 256, MEM_COMMIT, PAGE_READWRITE);
    if(address == NULL)
        return false;
    
    // 写入 DLL 全路径名
    DWORD bytesWritten;
    WriteProcessMemory(hProcess, address, DllFullPath, strlen(DllFullPath) + 1, &bytesWritten);
    if(bytesWritten < strlen(DllFullPath))
        return false;
    
    // 在目标进程中启动线程
    // 加载动态链接库
    HANDLE hThread = CreateRemoteThread(hProcess, NULL, NULL, (LPTHREAD_START_ROUTINE)LoadLibraryA, address, NULL, NULL);
    
    // 等待
    WaitForSingleObject(hThread, 0xffffffff);
    
    // 回收
    CloseHandle(hThread);
    VirtualFreeEx(hProcess, address, 256, MEM_DECOMMIT);
    CloseHandle(hProcess);
    
    return true;
}
```

然后编写`main`：

```cpp
int main() 
{
    // 注入 DLL 代码
    printf("注入 DLL\n");
    if(!InjectDll())
        printf("注入 DLL 失败\n");
        
    // 让控制台停住
    getchar();
    
    return 0;
}
```

编译运行之后，DLL 就被注入，我们也就能看到熟悉的窗口了。

# 六、相对路径

我们希望把 DLL 和这个程序放到一起，那么 DLL 路径就是程序所在路径加上 DLL 的名称。

将`DllFullPath`的定义注释掉，换成`DllName`：

```cpp
#define DllName "mfc_dll.dll"
```

然后修改`InjectDll`的参数：

```
bool InjectDll(const char *dllFullPath)
```

所有`DllFullPath`都改成`dllFullPath`。

然后试一试`GetCurrentDirectory`函数：

```cpp
int main()
{
    // 用于存放目录
    char dirName[256] = "";
    GetCurrentDirectory(sizeof(dirName), dirName);
    printf("%s\n", dirName);

    // 停住
    getchar();
    
    return 0;
}
```

这是运行结果：

![](https://wx3.sinaimg.cn/large/841aea59ly1foewzj4ny2j20fj02d749.jpg)

我们发现，这个程序是在项目目录的`debug`目录中（见窗口标题），但是取到了另一个目录。这是因为我们用调试模式启动程序，正常启动就好了。

现在我们拼接 DLL 全路径：

```cpp
// 保存 DLL 全路径
char dllFullPath[256] = "";

strcpy_s(dllFullPath, sizeof(dllFullPath), dirName);

// 我们需要补上斜杠
strcat_s(dllFullPath, sizeof(dllFullPath), "\\");

strcat_s(dllFullPath, sizeof(dllFullPath), DllName);
```

之后我们调用`InjectDll`：

```cpp
if(!InjectDll(dllFullPath))
    // ...
```

我们还需要清理项目目录，编写一个`bat`文件：

```bat
REM clear.bat
del *.sdf *.log *.user *.filters *.ipch *.aps /s
del *.exe *.dll /s
del *.suo /s /a h
del *.ilk *.pdb *.exp *.lib *.tlog *.manifest *.res *.lastbuildstate /s
del *.obj *.pch /s
pause
```

## 七、读取人物属性

打开第四节的 MFC 项目，添加一个按钮，把标题（`Caption`属性）改为“测试”。

![](https://wx4.sinaimg.cn/large/841aea59ly1foexqx1sugj20gm0b9q30.jpg)

然后双击这个按钮，程序会自动创建回调：

```cpp
void CMainDialogWnd::OnBnClickedButtonTest()
{
    // TODO:
}
```

我们在里面写一些东西，比如说调试信息：

```cpp
TRACE("GameDebug:我的调试信息\r\n");
```

`TRACE`函数的输出使用工具才能看到。我们打开`DebugView`，点击“编辑->Filter/Highlight”：

![](https://wx1.sinaimg.cn/large/841aea59ly1foexr5hootj20m00dz75p.jpg)

在弹出的窗口中，我们在`Include`后面的编辑框中输出`GameDebug*`：

![](https://wx3.sinaimg.cn/large/841aea59ly1foexrlrcpcj20eg07174j.jpg)

这样，我们就看不到不是我们发出的调试信息了。之后运行我们运行我们的程序，点击“测试”，就能看到调试信息：

![](https://wx2.sinaimg.cn/large/841aea59ly1foexs039f0j20jm0a1gma.jpg)

好，下面读取人物属性。在全局定义：

```cpp
#define BaseRole 0x2f860f0
```

然后在回调中添加：

```cpp
TRACE("GameDebug: 人物名=%s\r\n", BaseRole);
TRACE("GameDebug: 人物等级=%d\r\n", *(BYTE*)(BaseRole+0x34));
```

运行，点击按钮，我们可以看到属性信息：

![](https://wx1.sinaimg.cn/large/841aea59ly1foexs7knavj20lx052jrr.jpg)

## 八、人物信息的封装

> 视频里面的方法太啰嗦，我这里提供一个比较好的方法。

我们需要一种简便的方式，一次性读取所有信息。

首先创建`RoleProperty.h`，定义基址：

```cpp
#define RolePropertyBase 0x2f86170
```

然后定义结构体，参照之前的地址表格。

```cpp
struct RoleProperty
{
    // 0x0 名称
    char name[10];
    
    char padding1[42];
    
    // 0x34 等级
    BYTE level;
    // 0x35 职业（几转）
    BYTE job;
    // 0x36 名声
    char honor[10];
    
    char padding2[64];
    
    // 0x80 HP
    DWORD hp;
    // 0x84 MP
    DWORD mp;
    // 0x88 愤怒值
    DWORD angry;
    // 0x8c 最大 HP
    DWORD maxHp;
    // 0x90 最大 MP
    DWORD maxMp;
    // 0x94 最大愤怒值
    DWORD maxAngry;
    // 0x98 经验值
    QWORD exp;
    // 0xa0 最大经验
    QWORD maxExp;
    
    DWORD padding3;
    
    // 0xac 历练
    DWORD liLian;
    // 0xb0 心
    DWORD xin;
    // 0xb4 体
    DWORD ti;
    // 0xb8 气
    DWORD qi;
    // 0xbc 魂
    DWORD hun;
    
    QWORD padding4;
    
    // 0xc8 攻击
    DWORD attack;
    // 0xcc 防御
    DWORD defense;
    // 0xd0 命中
    DWORD accuracy;
    // 0xd4 回避
    DWORD evasion;
    
    char padding5[12];
    
    // 0xe4 金钱
    QWORD money;
    // 0xec 负重
    WORD load;
    // 0xee 最大负重
    WORD maxLoad;
    // 0xf0 剩余气功点数
    BYTE restQiGong;
    char padding6[3];
    
    // 0xf4 32 个气功
    struct {
        BYTE qiGong;
        char padding[3];
    } qiGongList[32];
    
    void GetData();
}
```

我们需要让它的布局与内存中完全一致，然后可以批量读取。这个就简单了，在`RoleProperty.cpp`中定义：

```
RoleProperty prop;
prop = *(RoleProperty*)RolePropertyBase;
```

## 九、静态库的加载

上一节的代码写在名为`GameData`的不同目录里面。我们希望能在`MFC_DLL`中引用。如果直接引用，会提示找不到头文件。

![](https://wx3.sinaimg.cn/large/841aea59ly1fof5tbpweoj20850183yc.jpg)

我们需要在“配置属性->C/C++->常规”中，设置附加包含目录。由于在不同的项目中，我们设为`..\\GameData\`。

![](https://wx2.sinaimg.cn/large/841aea59ly1fof5tlvpfcj20q30gu75o.jpg)

再次编译，又遇到了新的问题。

![](https://wx2.sinaimg.cn/large/841aea59ly1fof5ttqo92j20qe024jrr.jpg)

这个是由于静态库没有配置好。`GameData`项目会生成`GameData.lib`，在它自己的文件夹中。

我们在“配置属性->链接器->输入”中，将附加依赖项设为`GameData.lib`。

![](https://wx4.sinaimg.cn/large/841aea59ly1fof5u3ky1rj20q00gwt9y.jpg)

在“配置属性->链接器->常规”中，将附加库目录配置为`..\\lib\`。它表示项目目录中的`lib`。

![](https://wx1.sinaimg.cn/large/841aea59ly1fof5ud413kj20q10gtta8.jpg)

之后，打开`GameData`的属性页，在“配置属性->常规”中，将输出目录也改成这个。

![](https://wx3.sinaimg.cn/large/841aea59ly1fof5upqbn5j20py0gtwfz.jpg)

这个时候还是有冲突，对于所有项目，在“配置属性->C/C++->代码生成”中，将运行库设为“多线程调试”，就没冲突了。

然后，人物属性也不用一个一个输出了：

![](https://wx4.sinaimg.cn/large/841aea59ly1fof5ux8emsj20ny0a4jsu.jpg)

编译运行，之后点测试按钮，我们会看到：

![](https://wx1.sinaimg.cn/large/841aea59ly1fof5v76tq0j20m205dmxm.jpg)

## 十、物品数量

我们随便找个物品，比如第一个，看到它的数量是 66，这是个比较容易的突破口。

![](https://wx2.sinaimg.cn/large/841aea59ly1fof781qh1tj20a20473yn.jpg)

在 CE 里面扫描：

![](https://wx3.sinaimg.cn/large/841aea59ly1fof78aq540j20iy0a2q3p.jpg)

我们使用一个物品，变成了 65，再次扫描：

![](https://wx2.sinaimg.cn/large/841aea59ly1fof78lfq1rj20iu047q30.jpg)

一下子我们就找到了。但是这只是物品数量的地址，我们还要找到整个物品的基址。可以假设物品的各个属性用结构体存放。我们在这个地址上右键，“找出是什么访问了这个地址”：

![](https://wx4.sinaimg.cn/large/841aea59ly1fof78xz8t3j20iu07nmxi.jpg)

会弹出新的窗口，并且现在还是空的。我们将鼠标移到物品上面，再来看窗口：

![](https://wx2.sinaimg.cn/large/841aea59ly1fof7961n8pj20jj0ggdgr.jpg)

我们可以认为，ESI 的值`0x2ce59620`就是物品基址。

然后要找物品栏。可以假设，物品栏是物品指针的数组。所以需要再找找，是什么地方存放了这个地址：

![](https://wx2.sinaimg.cn/large/841aea59ly1fof79klbobj20iv09wwf2.jpg)

一共有四个结果。我们可以这样验证，把第一个物品移走。如果一个位置没有物品，那么数组里面的这个位置应该是`NULL`。

![](https://wx1.sinaimg.cn/large/841aea59ly1fof79secezj206x09jdft.jpg)

第二个地址`1a022ab8`变成了 0，所以它应该是物品栏那个位置的地址。

我们再来搜一下谁访问了这个地址：

![](https://wx1.sinaimg.cn/large/841aea59ly1fof7a37seoj20f90am0tr.jpg)

我们选那些计数为 1 的，因为为了访问物品栏的第 N 个位置，必须使用背包基址加上物品栏偏移加上指针大小乘物品位置。

我们选取第一个`0x0079c999`，在 OD 里转到这个地址：

![](https://wx2.sinaimg.cn/large/841aea59ly1fof7adna1yj20k805qmxo.jpg)

可以看到这个物品地址传给了 EAX，底下将这个物品的某个属性（不知道，因为没分析物品结构）传给了 ECX，在将这两个值传给底下的 CALL。这个 CALL 极有可能是物品使用 CALL。

我们在这个 CALL 上下断点，回到游戏里使用那个物品，就断下了，我们观察栈：

![](https://wx4.sinaimg.cn/large/841aea59ly1fof7ar0n4wj205g01v744.jpg)

这是前两个参数无误。之后打开代码注入器，将用这两个参数调用这个 CALL：

![](https://wx4.sinaimg.cn/large/841aea59ly1fof7bgftb8j20id07kglv.jpg)

点两下之后发现游戏没有反应，说明这个 CALL 不是。

我们在转到第二个地方`0079d2d8`，找到它下面的一个 CALL：

![](https://wx1.sinaimg.cn/large/841aea59ly1fof7bnsa1oj20gv02x0sq.jpg)

观察栈：

![](https://wx2.sinaimg.cn/large/841aea59ly1fof7by46g4j205c02gjr9.jpg)

这个时候 ECX 是`106226a8`。我们在代码注入器里面编写代码：

![](https://wx2.sinaimg.cn/large/841aea59ly1fof7c8jzmcj20cc03umx0.jpg)

运行，发现游戏中使用了这个物品。

![](https://wx3.sinaimg.cn/large/841aea59ly1fof7cgvdnpj207l048weg.jpg)

# 11 背包分析

上一节中我们发现，背包对象储存物品对象的指针，并且如果某一栏没有物品，那么那个位置就是`NULL`。我们可以以此快速寻找某个位置的地址。

比如说，我们先把第二个位置留空，在 CE 中搜索 0。搜索过程中，有些地址的数值会变化，所以多点几下“再次扫描”：

![](https://wx4.sinaimg.cn/large/841aea59ly1fogxx9aun6j20it09yjs4.jpg)

把第四个物品移到第二个，搜索比 0 大的数值：

![](https://wx2.sinaimg.cn/large/841aea59ly1fogxxkqeejj20iz09yjsb.jpg)

再把物品移动回去，搜索 0：

![](https://wx3.sinaimg.cn/large/841aea59ly1fogxxuwjsdj20j10a4wf8.jpg)

反复几次之后，就只剩一个结果了：

![](https://wx4.sinaimg.cn/large/841aea59ly1fogxy4e1s9j20ey0a0jrp.jpg)

我们可以验证一下。这个数值加 4 就是第三个物品的位置，再 4 就是第四个物品的位置。我们将物品 2-4 添加到底下：

![](https://wx2.sinaimg.cn/large/841aea59ly1fogxyh0v5qj20iw02wt8s.jpg)

我们再次把第四个移到第二个，数值也会相应变化。

![](https://wx4.sinaimg.cn/large/841aea59ly1fogxyq5m39j20is02f0ss.jpg)

所以我们找对了地址。

下一步寻找背包基址，在任意一个地址上右键，“找出是什么访问了这个地址”。

![](https://wx2.sinaimg.cn/large/841aea59ly1fogxz42yw8j20iw0exq41.jpg)

中间有几个指令，是`背包基址+物品栏偏移+物品序号*4`的形式，所以就是它们了。我们选取第一个`00656358`，背包基址应该是 EDX 的值`1b4d6238`。

这条指令上面的`31a8b3c`就是存放背包指针的地址。

然后我们挑选第三个物品`2f8eb250`，分析它的属性。切换为 ASCLL 视图：

![](https://wx4.sinaimg.cn/large/841aea59ly1fogxzrs67jj20kq08gwf9.jpg)

可以整理出一张表：

| 偏移 | 属性 |
| --- | --- |
| `0x5c` | 物品名称 |
| `0xf1` | 物品描述 |
| `0x244` | 数量 |

## 12 背包数据的封装

首先在`BaseGame.h`中定义背包基址的地址：

![](https://wx2.sinaimg.cn/large/841aea59ly1foh06dxfhoj20kz051t8x.jpg)

然后在`StructGame.h`中定义背包列表结构和物品结构：

![](https://wx4.sinaimg.cn/large/841aea59ly1foh06n1vt7j20d809sq3q.jpg)

然后在`StructGame.cpp`中实现`GetData`。先定义偏移常量：

![](https://wx4.sinaimg.cn/large/841aea59ly1foh06wjpowj20b503bmx5.jpg)

这是`GetData`的实现：

![](https://wx2.sinaimg.cn/large/841aea59ly1foh075rt02j20o60ctmxy.jpg)

然后到`CMainDialogWnd.cpp`的按钮回调中，添加输出代码：

![](https://wx3.sinaimg.cn/large/841aea59ly1foh07f83r8j20es08g0t0.jpg)

我们可以看到输出信息：

![](https://wx3.sinaimg.cn/large/841aea59ly1foh07qgrihj20t70e876v.jpg)

## 13 使用任意物品

这是第十节中的物品使用 CALL：

![](https://wx1.sinaimg.cn/large/841aea59ly1foh5cnlhfij20qb08njsc.jpg)

我们用 OD 附加游戏，在这个 CALL 上下断点：

![](https://wx3.sinaimg.cn/large/841aea59ly1foh5d0iw16j20uz0k3adf.jpg)

我们发现，第一个参数是 0，第二个参数是 1，这些没有变化。唯一变化的是第三个参数，经过试验，它是物品的下标（从 0 开始）。

ECX 是前面的 EDI，在之前的分析中，它是背包基址，但它是动态分配的，和之前相比也变化了。我们用 CE 看看哪个位置存放了这个地址：

![](https://wx2.sinaimg.cn/large/841aea59ly1foh5d9shfkj20ct0a5gm5.jpg)

第一个结果就是我们第十一节中的那个地址。

编程的逻辑是这样的，我们遍历物品列表，找到金疮药的下标。然后再调用这个 CALL。

![](https://wx4.sinaimg.cn/large/841aea59ly1foh5dhhnr8j20ck03rjr8.jpg)

## 14 编程使用物品

首先定义物品使用 CALL 的地址：

![](https://wx2.sinaimg.cn/large/841aea59ly1foh5usrkobj20ij00ojr8.jpg)

在物品列表结构中定义方法`UseGoodForIndex`，它接受下标，使用指定下标处的物品：

![](https://wx2.sinaimg.cn/large/841aea59ly1foh5v3e1vmj20if0cst8v.jpg)

我们还需要定义一个方法，使用指定名称的物品，在此之前我们还需要一个方法`GetGoodIndexForName`，按照名称寻找物品，并返回下标：

![](https://wx4.sinaimg.cn/large/841aea59ly1foh5vd00gcj20fj08g74g.jpg)

然后就可以定义方法`UseGoodForName`：

![](https://wx4.sinaimg.cn/large/841aea59ly1foh5vkc3glj20ke06r74k.jpg)

我们在回调中调用这个方法，然后判断结果：

![](https://wx4.sinaimg.cn/large/841aea59ly1foh5vujo5gj20d9031wee.jpg)

可以看到物品使用成功：

![](https://wx1.sinaimg.cn/large/841aea59ly1foh5w2xcwzj20ha0aidh9.jpg)

## 15 实现自己的`TRACE`

这一节中我们要实现自己的`TRACE`。这个函数有两个特点：

+   接受格式字符串和格式化参数，格式化参数是可变的。
+   调用 API`OutputDebugStringA`，这个函数不是变参的。

思路是，我们可以用`sprintf`将输出字符串格式化好，存到一个地方。然后用它调用`OutputDebugStringA`。

但是变参函数不能调用变参函数，我们需要使用它的非变参版本`vsprintf`，它接受缓存区地址，格式字符串和格式化参数的起始地址。这就像 Java 中，编译器把可变参数放进数组中，然后再传给函数。

我们还需要用到三个宏：

+   `va_list`实际上就是`char*`，没啥特别的。
+   `va_start(p, arg)`首先将`arg`的地址加上`arg`的大小赋给`p`。实际上将`p`指向`arg`的下一个位置。
+   `va_end(p)`清空`p`。

除此之外还需要给输出字符串添加前缀，以便在工具中过滤它，这可以通过`strcat_s`实现。

这是我们所实现的函数：

![](https://wx4.sinaimg.cn/large/841aea59ly1foh6xw2mxhj20cf09ldga.jpg)

## 16 消除异常

异常原因是，游戏进程中，主线程和我们启动的线程同时访问一块内存，产生了[竞争条件](https://zh.wikipedia.org/wiki/%E7%AB%9E%E4%BA%89%E6%9D%A1%E4%BB%B6)。

如果可以向主线程注入代码，就可以避免这种情况。

## 17 向主线程注入代码

有两种方式。第一种是设置消息钩子，`SetWindowsHookEx`，触发`WndProc`之前首先会触发钩子。第二种是直接通过`SetWindowsLong`直接改`WndProc`。这里我们使用第一种。

我们在 MSDN 上查看`SetWindowsHookEx`：

```cpp

HHOOK WINAPI SetWindowsHookEx(
  _In_ int       idHook,
  _In_ HOOKPROC  lpfn,
  _In_ HINSTANCE hMod,
  _In_ DWORD     dwThreadId
);
```

第一个参数`idHook`是钩子的类型，我们选择`WH_CALLWNDPROC`，表示系统将消息发给目标窗口之前拦截。

第二个参数`lpfn`是钩子的回调。

第三个参数`hMod`是动态链接库句柄，由于这是全局钩子，所以为空就行了。

第四个参数`dwThreadId`是窗口所在的线程 ID，可以通过窗口句柄和`GetWindowThreadProcessId`获取。

首先获取游戏句柄：

![](https://wx3.sinaimg.cn/large/841aea59gy1fohb4os16oj20b409bdg4.jpg)

为了便于多开。我们寻找存放句柄的地方。

![](https://wx3.sinaimg.cn/large/841aea59gy1fohb4yetk8j20j709pmxw.jpg)

这四个在游戏空间里面，我们使用这四个之一。选择第一个`fa0674`。

![](https://wx2.sinaimg.cn/large/841aea59gy1fohb59lwoyj20in00tq2s.jpg)

编写读取句柄的函数。

![](https://wx2.sinaimg.cn/large/841aea59gy1fohb5jwcgtj20id07kdft.jpg)

编写设置钩子的函数。

![](https://wx1.sinaimg.cn/large/841aea59gy1fohb5kpyvvj20ka06hwf6.jpg)

我们在 MSDN 里面搜索`CallWndProc`，它是钩子的回调类型：

```cpp
LRESULT CALLBACK CallWndProc(
  _In_ int    nCode,
  _In_ WPARAM wParam,
  _In_ LPARAM lParam
);
```

第一个参数`nCode`表示是否可以处理。如果`nCode`是`HC_ACTION`，那么久处理它。如果小于 0，就必须把消息传给`CallNextHookEx`，并返回它的返回值。

第二个参数`wParam`表示是否由当前线程创建。如果是则为 0，不是为非 0。

第三个参数`lParam`是`CWPSTRUCT*`，表示消息的具体信息。

首先注册一个自己的消息码，然后编写钩子回调。

![](https://wx1.sinaimg.cn/large/841aea59gy1fohb5k8t5fj20ij0aata6.jpg)

编写释放钩子的函数，和测试函数。测试函数向游戏窗口发送了我们的自定义消息，并带有一个字符串。

![](https://wx2.sinaimg.cn/large/841aea59gy1fohb5k9mh8j20ew06jq2z.jpg)

然后在界面上添加几个按钮，编写回调。

![](https://wx3.sinaimg.cn/large/841aea59gy1fohb6v5sbuj20ij09e3zg.jpg)

首先点击“挂接主线程”，然后点击“使用金疮药（小）”，它就收到消息了。

![](https://wx1.sinaimg.cn/large/841aea59gy1fohb6v4xb5j20l30alwez.jpg)

然后把`DbgPrintf`改成`UseGoodForName`，就可以真正使用物品了。

## 18 分析怪物列表

这一节中我们要分析怪物对象的地址：

![](https://wx2.sinaimg.cn/large/841aea59ly1foi6q340zcj20a90a0t8t.jpg)

这个怪物的 HP 应该在 200 到 1300 之间，搜索这个范围内的数值：

![](https://wx1.sinaimg.cn/large/841aea59ly1foi6qdzu0bj20j509v0te.jpg)

结果很多，可以用“没有变化的数值”过滤：

![](https://wx1.sinaimg.cn/large/841aea59ly1foi6qp396vj20j604wq33.jpg)

还是很多，我们可以攻击怪物，然后搜索“减少的数值”：

![](https://wx2.sinaimg.cn/large/841aea59ly1foi6qx56rzj20j609y3yw.jpg)

由于刚才怪物被打死了，所以 HP 应该为 0。第一个是正确地址。怪物死后，对象不会销毁而是复用，所以过一会儿就能看到它变回了 1300。

把这个地址添加到下面的列表中，右键，“谁访问了这个地址”。这个时候列表还没有动静。

我们需要找到这个对象表示的怪物，应该就在附近。找到之后攻击，HP 应该会减少，我们可以由此判断是否找对了。同时，窗口中也会显示访问这个地址的指令：

![](https://wx2.sinaimg.cn/large/841aea59ly1foi6rngprjj20nk0gndgx.jpg)

我们发现有两个类型的指令，一个是`eax+0x5b4`，`eax=30235698`，另一个是`esi+0x5bc`，`esi=30235690`。这两个数值都可能是怪物对象的基址。在 CE 中分别搜索这两个数值。

这是`30235698`的结果，第三个`31d342c`是不变的，其它的不断变化：

![](https://wx4.sinaimg.cn/large/841aea59ly1foi6r7eicmj20ja0b2q3h.jpg)

这是`30235690`的结果，两个都是不变的：

![](https://wx4.sinaimg.cn/large/841aea59ly1foi6rwahrgj20j70b0gm4.jpg)

我们忽略掉后一个结果，把`30235698`当做怪物的基址。在 OD 中查看`31d342c`：

![](https://wx4.sinaimg.cn/large/841aea59ly1foi6s4wx9ej209i09b0sz.jpg)

我们找到了一系列怪物对象的地址。执行`dd [31d342c] + 0x5b4`，就是那个怪物对象的血量，它是`0x514 = 1300`。

![](https://wx3.sinaimg.cn/large/841aea59ly1foi6sbkd05j209f09dt8x.jpg)

血量附近肯定有怪物名称，切换 ASCII 视图：

![](https://wx4.sinaimg.cn/large/841aea59ly1foi6siz0prj20l509gmy1.jpg)

偏移`0x320`处就是怪物名称。

然后换一个怪物试试看，给列表地址加 4：

![](https://wx3.sinaimg.cn/large/841aea59ly1foi6spyyczj20lc09jjsc.jpg)

仍然是个有效的对象，但是当我们访问下标为 3 的对象：

![](https://wx3.sinaimg.cn/large/841aea59ly1foi6sz179qj20l009igmr.jpg)

可以看到它是个物品，那么这个列表是物品和怪物混着保存的。

接下来找怪物/物品列表的基址，这次直接在 ID 里面找，对`31d342c`下内存断点：

![](https://wx1.sinaimg.cn/large/841aea59ly1foi6tj380ej20jc0ck3zm.jpg)

找到了这里，基址就是`31ce740`。

然后在 OD 中查看`45e4aac`，它是包含怪物基址的列表，这里的数据是不断变化的。

![](https://wx1.sinaimg.cn/large/841aea59ly1foi6twhbu9j209a08i0sy.jpg)

这个列表中的对象也是怪物对象。但是无论我们访问哪一个，它都是怪物对象。

![](https://wx2.sinaimg.cn/large/841aea59ly1foi6u7di3bj20lb09ewfl.jpg)

对这个地址下内存断点，停在了这里：

![](https://wx4.sinaimg.cn/large/841aea59ly1foi6uj8jkmj20ip04n3yv.jpg)

`45e4a88`就是这个列表的基址。这个列表可能是什么呢？应该是周围的怪物列表。

下面找怪物的坐标，切换到浮点数：

![](https://wx1.sinaimg.cn/large/841aea59ly1fomqolusbaj20oj08at9b.jpg)

怪物坐标和自己的坐标应该差不多。这里有两个坐标，所以应该是怪物的两个移动的点。

怪物应该还有个属性，代表是否死亡。我们搜索 0 和 1 之间的数值：

![](https://wx1.sinaimg.cn/large/841aea59ly1fomqoz2kmsj20j20a23zb.jpg)

第一个结果就是，因为它正好在怪物对象里面。也可以攻击所表示的那个怪物来验证，怪物死亡之后，这个值变成了 0：

![](https://wx1.sinaimg.cn/large/841aea59ly1fomqp7m49dj208z08fdg0.jpg)

然后我们把怪物对象的属性整理一下：

| 偏移 | 类型 | 属性 |
| --- | --- | --- |
| 0x314 | BOOL | 活着（1）死亡（0） |
| 0x320 | STR | 名字 |
| 0x5b4 | DWORD | 血量 |
| 0x5b8 | DWORD | 等级 |
| 0x1018 | FLOAT32 | 横坐标1 |
| 0x1020 | FLOAT32 | 纵坐标1 |
| 0x1024 | FLOAT32 | 横坐标2 |
| 0x102c | FLOAT32 | 纵坐标2 |

## 19 封装怪物列表

首先在`BaseGame.h`中添加基址。

![](https://wx3.sinaimg.cn/large/841aea59ly1fomrb3hi3fj20cx00qweb.jpg)

`StructGame.h`中定义怪物和怪物列表结构：

![](https://wx3.sinaimg.cn/large/841aea59ly1fomrbd8jbtj207o0ewt9o.jpg)

实现`GetData`：

![](https://wx2.sinaimg.cn/large/841aea59ly1fomrbkiij9j20e309wwfc.jpg)

实现`dbgPrintMessage`：

![](https://wx2.sinaimg.cn/large/841aea59ly1fomrbtuljkj20fz09kglu.jpg)

执行结果：

![](https://wx1.sinaimg.cn/large/841aea59ly1fomrc1epqcj20sy0fkgnz.jpg)

## 20 再次分析怪物属性

`45e4a88`的列表中也包括角色对象：

![](https://wx3.sinaimg.cn/square/841aea59ly1fonwypgxayj20ks09egmu.jpg)

然后我们选一个角色对象和一个怪物对象，把内存复制出来作比较。

![](https://wx1.sinaimg.cn/square/841aea59ly1fonwz0b644j20mk0niq5c.jpg)

可以发现，0x8 是对象类型编号，`0x31`是玩家。

然后我们改一个对象的名称，方便在地图上找到它：

![](https://wx4.sinaimg.cn/square/841aea59ly1fonwzg5rrkj20lk09f0tm.jpg)

我们选中它，发现 0x314 为 1，不选中它又为 0。所以 0x314 是选中状态。

那么是否死亡呢？这个属性可能死亡为 1，活着的时候为 0。我们换一种搜索方式，在 CE 里面搜索 0。

![](https://wx1.sinaimg.cn/square/841aea59ly1fonwzve9iqj20ja0axq3m.jpg)

攻击怪物死亡，搜索 1：

![](https://wx1.sinaimg.cn/square/841aea59ly1fonx0684x2j20jb0b9q3m.jpg)

在对象基址附近的只有前三个。

这个时候怪物刷新了，前两个是零。所以排除第三个。

![](https://wx4.sinaimg.cn/square/841aea59ly1fonx0e08rdj20if01udfr.jpg)

第一个是偏移 0x380，第二个位置是偏移 0x768，这两个位置可能是死亡状态。

再攻击一下它，发现只有第一个变为了 1。选取它当死亡状态。

![](https://wx2.sinaimg.cn/square/841aea59ly1fonx0ksefqj20in01g3ye.jpg)

## 21 分析动作数组

目前为止，我们的思路是：

+   搜索对象中的数值，找到属性地址
+   根据属性地址搜索指令（内存断点），找到对象基址
+   根据对象基址搜索指令，找到列表基址
+   根据列表基址找到 CALL。

动作一共有 12 个。如果我们能找到动作对象的调用 CALL，就能实现挂机打怪。

![](https://wx1.sinaimg.cn/square/841aea59ly1foogsti9daj20bq05waa6.jpg)

但是动作对象并没有展示的数值。我们假设，有个地方保存当然选中的动作对象的地址，没有选中时它为 0。

首先搜索 0：

![](https://wx4.sinaimg.cn/square/841aea59ly1foogt6tdplj20d70acq3c.jpg)

选中并拖动动作，搜索“比 0 大的数值”：

![](https://wx4.sinaimg.cn/square/841aea59ly1foogteff2zj20cz0a5dgc.jpg)

把动作放回，搜索 0，这里可以多搜几次：

![](https://wx1.sinaimg.cn/square/841aea59ly1foogtnjbg7j20d00a6q3b.jpg)

重复前面两个步骤。如果结果还是很多，可以这样：

+   选中并拖动
+   搜索一次“变化的数值”
+   搜索几次“未变化的数值”
+   放回
+   搜索一次“变化的数值”
+   搜索几次“未变化的数值”
+   重复以上步骤

最后只剩下三个结果。观察第一个结果，未选中时为 0，选中时不为 0，并且选中不同的对象时值不同。

![](https://wx3.sinaimg.cn/square/841aea59ly1foogty05knj207k03tglh.jpg)

然后我们可以选中不同对象，记录对象的值：

| 序号 | 值 |
| --- | --- |
| 1 | 0x225f5050 |
| 2 | 0x225f5298 |
| 3 | 0x225f54e0 |

然后这个是等差的，所以我们可以推断，这些是动作对象的地址，对象大小为`584`。

然后搜索第一个地址：

![](https://wx4.sinaimg.cn/square/841aea59ly1foogu75df2j20fq04umx9.jpg)

查看第一个结果`031d22c0`的内存：

![](https://wx2.sinaimg.cn/square/841aea59ly1fooguhrigmj20fx098gmp.jpg)

发现它连续存放了动作对象的地址（第二个也是）。那么它应该是动作列表的基址。

我们在基址上面下内存断点。
