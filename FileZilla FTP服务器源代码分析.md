# FileZilla FTP服务器源代码分析

FileZilla是开源的[FTP服务器](https://so.csdn.net/so/search?q=FTP%E6%9C%8D%E5%8A%A1%E5%99%A8&spm=1001.2101.3001.7020)，用C++写的，通过分析它的源代码，可以掌握C++网络编程以及高并发服务器的设计。  
FileZilla 是[<u>http://sourceforge.net</u>](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2F)上的项 目，主页是[<u>http://sourceforge.net/projects/filezilla</u>](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2Fprojects%2Ffilezilla)， 我们要研究的版本是：FileZilla Server 0_9_18，其实FileZilla还包括客户端软件。

下载后安装，安装时选择安装source，即安装了源代码。

安装完成后，可以直接打开工程自带的FileZilla server.sln，这个是vs 2003.net工程，里面有三个project，  
FZS Interface：这个是FTP服务器的设置以及监控界面  
Service：这个是核心的FTP服务器部分  
GFtp：打不开，不知是干 什么用的，老外也会如此马虎？ :)

直接编译是会出错，主要是FileZilla需要依赖两个第三方包：zlib（压缩算法包）以及regexp（正则表达式的包）

先搞定zlib，下载[<u>http://www.winimage.com/zLibDll/zlib123.zip</u>](http://blog.chinaunix.net/link.php?url=http://www.winimage.com%2FzLibDll%2Fzlib123.zip)  
解 开，生成目录zlib123，用.net 2003打开zlib123\projects\visualc6\zlib.dsw  
打开 生成|配置管理器，设置活动的解决方案配置为：DLL Release，编译生成项目zlib，成功后会在目录Win32_DLL_Release下生成zlib1.lib和zlib1.dll

然后，regexp用到了boost第三方包，这里面有很多公用的C++组件，下载地址：  
[<u>http://internap.dl.sourceforge.net/sourceforge/boost/boost_1_33_1.zip</u>](http://blog.chinaunix.net/link.php?url=http://internap.dl.sourceforge.net%2Fsourceforge%2Fboost%2Fboost_1_33_1.zip)  
解 开后，在cmd下，  
第一步：需要编译bjm，这是编译boost的编译器，晕  
cd boost_1_33_1\tools\build\jam_src  
build.bat  
在当前的bin.ntx86目录下，生成了 bjam.exe文件。  
第二步：编译boost  
cd boost_1_33_1  
将刚才生成的bjam.exe拷过来  
set VC7_ROOT="C:\Program Files\Microsoft Visual Studio.NET 2003\Vc7"  
bjam "-sTOOLS=vc7" install  
这个步骤需要很久时间，其时我们只用了里面的regexp包，应该可以只编译这个包，但我没细研 究。  
编译完成后即在C盘根目录下生成了boost目录，这个目录下面.net工程要用到。

在.net工程里，工具|选项|项目|VC++目录，添加  
可执行文件：zlib123\projects\visualc6 \Win32_DLL_Release  
包含文件：zlib123目录以及c:\Boost\include\boost-1_33_1  
库 文件：zlib123\projects\visualc6\Win32_DLL_Release以及c:\Boost\lib

这时FZS Interface工程应该可以编译成功了，编译完成后会在Debug目录生成FileZilla Server Interface.exe。

生成Service工程时，需要调整一下工程设置：  
语言设置：打开工程属性页，配置属性|常规，选择字符集为“使用 Unicode 字符集”，否则编译时会出错unicode必须使用；  
链接设置：打开工程属性页，配置属性|链接器，将输入zlib.lib改成 zlib1.lib（因为zlib123工程生成的是zlib1.lib）  
下面生成一下，应该可以了。在生成Service工程完成时已经自动安 装了"FileZilla Server FTP server"服务，也可以手工安装服务:  
cd FileZilla_Server\Debug  
"FileZilla server.exe" install auto

可以试一下这个FTP服务器了，运行FileZilla_Server\interface\Debug\FileZilla Server Interface.exe，这是FTP服务器的配置监控程序，试着加几个user，设置home dir，再用FTP客户端连接一下试试，应该可以了。

下面开始分析这个服务器的实现。

WM_FILEZILLA_SERVERMSG：  
     
wParam代表大的分类, 即是何种消息, lParam是附加的信息, 具体根据wParam的不同而不同。  
     
wParam有以下几种:  
FSM_STATUSMESSAGE:  
 记 录当前活动, 并将活动信息在admin窗口上显示, 并且记录到log文件中（需要设置相应选项），打开admin窗口，上半部分显示的内容就是从里来的。  
 lParam带的是 t_statusmsg结构, 里面记录了当前活动的user, ip, time, message等  
 如：  
 (000001) 2006-7-23 16:03:56 - (not logged in) (127.0.0.1)> USER whg  
 (000001) 2006-7-23 16:03:59 - (not logged in) (127.0.0.1)> 331 Password required for whg  
 (000001) 2006-7-23 16:04:05 - (not logged in) (127.0.0.1)> PASS *  
 (000001) 2006-7-23 16:04:11 - robert (127.0.0.1)> 230 Logged on  
   
FSM_CONNECTIONDATA:    
 这是跟 connection相关的消息，如新建了连接，用户登录成功，用户退出等等。信息发送到admin窗口，显示在admin窗口的下半部分，即ID、 Account、IP等内容。  
 lParam带的是t_connop结构, 结构中成员op代表更详细的connection分类，可能值有:  
 USERCONTROL_CONNOP_ADD  
  有 新用户进行连接（还未登录）  
 USERCONTROL_CONNOP_CHANGEUSER  
  登录成功  
 USERCONTROL_CONNOP_REMOVE  
  用 户退出，或者因为time被强行注销  
 USERCONTROL_CONNOP_TRANSFERINIT  
  传输开始或结束, 即与客户端有数据通讯，如开始传输数据，ls命令也会导致数据的传输。  
 USERCONTROL_CONNOP_TRANSFEROFFSETS  
  显 示传输进度，如在进行文件传输的过程中，需要向admin窗口显示当前的传输字节数，传输速率等  
    
 所有的这些信息都会在admin窗 口的下半部分中显示。  
   
FSM_THREADCANQUIT:  
 系统退出时会发出这些消息，系统在处理这个消息时，会结束线程  
   
FSM_SEND:  
 系 统只要发送了数据，都会发送这个消息，并且在admin窗口的状态条中显示当前用户用发送了多少数据  
   
FSM_RECV:  
 系统 只要接收到数据，都会发送这个消息，并且在admin窗口的状态条中显示当前用户用收到了多少数据

可见，CServer处理的消息应该只是一些admin或者status消息，这些消息应该在线程listen, accep处理相应的ftp请求时发出。真正的FTP处理并不在这里。

win32同步控制机制（Synchronization）回顾

1、Critical Sections（关键域）  
最简单的一种同步机制，创建和销毁的函数是：  
InitializeCriticalSection()  
DeleteCriticalSection()  
在 被创建后，使用如下函数实现线程同步，  
EnterCritSection()  
... 需要同步的代码  
LeaveCritSection()  
即 在同一时间内，EnterCritSection和LeaveCritSection中间的代码只能被一个线程处理。注意的问题是：  
Critical Sections类型的变量并不是一个核心对象，即没有handle；  
它存在于进程的内存空间中，即不可跨进程使用；  
可能会导致死锁；

2、Mutexes（互斥器）  
创建和销毁的函数是：  
CreateMutex()  
CloseHandle()  
如果 Mutex已经被创建，这样打开和关闭：  
OpenMutex()  
ReleaseMutex()  
使用的时候，用wait函数来等待 Mutex，一旦没有线程拥有这个Mutex，这个线程就会获得这个Mutex，在这个线程处理完以后，调用ReleaseMutex()可以释放这个 Mutex，其它等待中的线程就会重新竞争这个Mutex，同时只能有一个线程获得这个Mutex，没竞争到的线程则处于blocking阻塞状态。常见 的wait函数有：  
WaitForSingleObject() // 等待一个mutex  
WaitForMultipleObjects() // 同时等待多个mutex，要不同时拥有多个，要不一个也没有

和Critical Sections相比，mutex是一个核心对象，因此是跨进程的，即多个进程可以使用同一个mutex，并且CreateMutex()的开销比 InitializeCriticalSection()要大得多。  
相比而言，mutex更重量级，更慢，但也更灵活。

3、信号量（Semaphores）  
创建和销毁的函数是：  
CreateSemaphore()  
CloseHandle()  
获 取这个Semaphore的函数同样是那些wait函数WaitForSingleObject(), WaitForMultipleObjects()等等。  
使用Semaphores的含义是：Semaphores可以同时被多个线程拥有，但在 CreateSemaphore()时会指定一个同时拥有这个Semaphore的最大线程数，即每个线程调用wait函数获取Semaphore 时，Semaphore内部的可用值就会减1，一旦可用值为0，则线程必须等待。当拥到Semaphore的线程运行完后，也应该调用 ReleaseSemaphore()来释放。  
同Mutexes不一样的是，调用ReleaseSemaphore()的线程并一定是调用 wait并获得拥有权的那个线程，即任何线程都可以在任何时间调用ReleaseSemaphore()来解除被任何线程锁定的Semaphore。  
在 某种意义上，Mutexes可以看成是Semaphore的一个特例，即只能同时被一个线程锁定的Semaphore。

Semaphore也是核心对象。

4、事件（Event Objects）  
win32中最灵活的应该是events了，它也是一种核心对象。  
events的含义在 于：events有两种状态，激活和非激活，在events被激活时，那些等待着的线程会被唤醒。  
创建和销毁的函数是：  
CreateEvent()， 创建时可以指定events是manual或automatic，manual的含义是events的状态是由程序员设定的。automatic的含义是 events变成激话后，立即自动变成非激活。  
CloseHandle()  
获取这个events的函数同样是那些wait函数 WaitForSingleObject(), WaitForMultipleObjects()等等。  
下面三个方法可以改变events的状 态：  
SetEvent():  把events设为激活  
ResetEvent(): 把events设为非激活  
PulseEvent(): 激活events，然后立即高为非激活。如果events是manual的，则唤醒所有等待的线程，如果events是automatic的，同唤醒“一 个”等待的线程。

注意，如果event是manual时，这时调用SetEvent后，如果不调用ResetEvent，则等待中的线程会被不断的唤醒，即不断地执 行CreateThread时指定的lpStartAddress方法。  
还有一点，windows系统可以保证被唤醒的线程是一个接着一个的，即 不会有的线程总是被唤醒，而有些线程被饿死。 

CServer在Create()时，通过CListenSocket来监听标准的FTP 21端口，通过CAdminListenSocket来监听admin端口（缺省是14147），这两个类都继承于CAsyncSocketEx，这个类 是FileZilla中所有socket处理的基类，搞清楚这个类可以清楚明白socket处理的机制。

这个类的名字来源于MFC类CAsyncSocket，CAsyncSocketEx完全兼容于CAsyncSocket，在 CAsyncSocket上写的代码可以一字不动的在CAsyncSocketEx下编译通过，CAsyncSocketEx还做了一些功能上的扩展和性 能上的优化。

CAsyncSocketEx和另两个类CAsyncSocketExHelperWindow以及 CAsyncSocketExLayer紧密相关，CAsyncSocketExLayer的作用类似于J2EE中的Interceptor的作用，这里 可以先不讨论。

CAsyncSocketEx采用的是消息处理的机制，即监听的端口有活动，如有数据要接收、发送等，通过发送消息来实现这种信息的通讯，这里具体 到windows socket的API就是WSAAsyncSelect，它的原型是：  
int WSAAsyncSelect(  
  SOCKET s,  
  HWND hWnd,  
  unsigned int wMsg,  
  long lEvent  
);

Parameters  
s   
[in] Descriptor identifying the socket for which event notification is required.   
hWnd   
[in] Handle identifying the window that will receive a message when a network event occurs.   
wMsg   
[in] Message to be received when a network event occurs.   
lEvent   
[in] Bitmask that specifies a combination of network events in which the application is interested.

简单地说，这个方法可以让windows在SOCKET s指定的socket上，当指定的事件lEvent发生时，发送消息wMsg到窗口hWnd。

由于windows的消息机制必须使用一个windows窗口，因此CAsyncSocketEx必须创建一个windows窗口来接收这种消息， 这就是类CAsyncSocketExHelperWindow的主要作用，当然这个窗口并不是必须被显示出来的，只要让系统知道有这个windows存 在(即有hWnd)就可以了。

在CAsyncSocketEx中，定义了一个static的链表：  
 static struct t_AsyncSocketExThreadDataList  
 {undefined  
  t_AsyncSocketExThreadDataList *pNext;  
  t_AsyncSocketExThreadData *pThreadData;  
 } *m_spAsyncSocketExThreadDataList;  
这个链表维护了一个t_AsyncSocketExThreadData 链，看一下这个struct的定义：  
 struct t_AsyncSocketExThreadData  
 {undefined  
  CAsyncSocketExHelperWindow *m_pHelperWindow;  
  int nInstanceCount;   
  DWORD nThreadId;  
  std::list<CAsyncSocketEx*> layerCloseNotify;  
 } *m_pLocalAsyncSocketExThreadData;  
看名称就知道，这是一 个与线程thread有关的结构，事实上这个结构描述了一个分发线程。

在FileZilla的实现中，整个静态的类关系是这样的：

一个CAsyncSocketEx代表了一个socket，即在某个端口进行监听的socket，如前面提到的标准的FTP 21端口、admin端口等等。

一个CAsyncSocketExHelperWindow代表了一个负责消息分发的线程，即负责接收到 socket(CAsyncSocketEx)的活动，然后分发到不同的处理类CAsyncSocketEx。每一个 CAsyncSocketExHelperWindow一一对应于一个分发线程，即一个分发线程只有一个 CAsyncSocketExHelperWindow，反之亦然。结构t_AsyncSocketExThreadData即描述了分发线程与 CAsyncSocketExHelperWindow的关系。

CAsyncSocketExHelperWindow可以为多个CAsyncSocketEx进行分发，而CAsyncSocketEx只能由一 个CAsyncSocketExHelperWindow进行分发。现在仔细研究一下结构t_AsyncSocketExThreadData：  
 struct t_AsyncSocketExThreadData  
 {undefined  
  CAsyncSocketExHelperWindow *m_pHelperWindow; // 这个线程对应的CAsyncSocketExHelperWindow  
  int nInstanceCount;  // 当前分发线程对应了几个CAsyncSocketEx  
  DWORD nThreadId; // 当前线程的threadID  
  std::list<CAsyncSocketEx*> layerCloseNotify; // 这个以后再说  
 } *m_pLocalAsyncSocketExThreadData;  
这段代码是在类CAsyncSocketEx 中定义的，即m_pLocalAsyncSocketExThreadData定义了当前CAsyncSocketEx所对应的分发线程，即 CAsyncSocketExHelperWindow。

全局的m_spAsyncSocketExThreadDataList则定义了一个t_AsyncSocketExThreadData(即分发 线程)的链表，也就是说FileZilla可以有多个分发线程，每个分发线程对应多个socket，即CAsyncSocketEx。

举一个实际的场景：

在FileZilla Server启动时，缺省监听了两个端口：21和admin端口，因此就有两个socket，即两个CAsyncSocketEx。  
这两个 CAsyncSocketEx共用一个分发线程：t_AsyncSocketExThreadData

当有用户通过FTP连接上server并通过get/mget命令下载文件时，这时FTP服务器会启动一个传输线程在一个临时端口进行监听，这时会 增加一个CAsyncSocketEx，同时也增加一个负责这个CAsyncSocketEx的分发线程，因此 m_spAsyncSocketExThreadDataList里也会增加一个结点。

这时的状况是：  
一个m_spAsyncSocketExThreadDataList链，两个 t_AsyncSocketExThreadData，三个CAsyncSocketEx。

下面仔细分析具体的实现。

在CServer的Create()中，创建对象CListenSocket来监听21端口，来看看具体的代码实现：

 CListenSocket *pListenSocket = new CListenSocket(this, ssl);  
 ...  
   
 if (!pListenSocket->Create(nPort, SOCK_STREAM, FD_ACCEPT, NULL) || !pListenSocket->Listen())  
 ...

基本上分三步：  
1、new CListenSocket：没有什么特别的，基本就是初始化成员变量

2、Create  
 注：在所有的代码中，先不看大量的if (m_pFirstLayer)这种代码，这是CAsyncSocketExLayer的机制。  
   
 Create其实调用的是父类 CAsyncSocketEx的create()方法，这个方法中第一件事情就是建立m_spAsyncSocketExThreadDataList 链、分发线程t_AsyncSocketExThreadData以及CAsyncSocketEx之间的关系，CAsyncSocketEx的 create()方法首先调用InitAsyncSocketExInstance()，下面是 CAsyncSocketEx::InitAsyncSocketExInstance()代码片断：  
   
 DWORD id=GetCurrentThreadId();  
   
 ...

 //Get thread specific data  
 if (m_spAsyncSocketExThreadDataList) // 这个链已经建立了  
 {undefined  
  t_AsyncSocketExThreadDataList *pList=m_spAsyncSocketExThreadDataList;  
  while (pList) // 遍历链  
  {undefined  
   ASSERT(pList->pThreadData);  
   ASSERT(pList->pThreadData->nInstanceCount>0);

   if (pList->pThreadData->nThreadId==id) // 对当前线程已经有分发线程了，就把当前socket的分发由这个分发线程来代理  
   {undefined  
    m_pLocalAsyncSocketExThreadData=pList->pThreadData;  
    m_pLocalAsyncSocketExThreadData->nInstanceCount++; // 多了这一个socket  
    break;  
   }  
   pList=pList->pNext;  
  }  
  //Current thread yet has no sockets  
  if (!pList) // 当前线程还没有分发线程，则创建一个  
  {undefined  
   //Initialize data for current thread  
   pList=new t_AsyncSocketExThreadDataList;  
   pList->pNext=m_spAsyncSocketExThreadDataList;  
   m_spAsyncSocketExThreadDataList=pList;  
   m_pLocalAsyncSocketExThreadData=new t_AsyncSocketExThreadData;  
   m_pLocalAsyncSocketExThreadData->nInstanceCount=1; // 只挂了当前的socket  
   m_pLocalAsyncSocketExThreadData->nThreadId=id; // 这个分发线程的threadID  
   m_pLocalAsyncSocketExThreadData->m_pHelperWindow=new CAsyncSocketExHelperWindow(m_pLocalAsyncSocketExThreadData); // 为这个分发线程创建CAsyncSocketExHelperWindow  
   m_spAsyncSocketExThreadDataList->pThreadData=m_pLocalAsyncSocketExThreadData;  
  }  
 }  
 else // 如果分发线程链还没有创建，则创建一个  
 { //No thread has instances of CAsyncSocketEx; Initialize data  
  m_spAsyncSocketExThreadDataList=new t_AsyncSocketExThreadDataList;  
  m_spAsyncSocketExThreadDataList->pNext=0;  
  m_pLocalAsyncSocketExThreadData=new t_AsyncSocketExThreadData; // 第一个分发线程  
  m_pLocalAsyncSocketExThreadData->nInstanceCount=1; // 只挂了当前的socket  
  m_pLocalAsyncSocketExThreadData->nThreadId=id; // 这个分发线程的threadID  
  m_pLocalAsyncSocketExThreadData->m_pHelperWindow=new  CAsyncSocketExHelperWindow(m_pLocalAsyncSocketExThreadData); // 为这个分发线程创建CAsyncSocketExHelperWindow  
  m_spAsyncSocketExThreadDataList->pThreadData=m_pLocalAsyncSocketExThreadData;

 ...  
 }

 下面看一个创建CAsyncSocketExHelperWindow的过程：  
 CAsyncSocketExHelperWindow(CAsyncSocketEx::t_AsyncSocketExThreadData* pThreadData)  
 {undefined  
  // m_pAsyncSocketExWindowData是一个t_AsyncSocketExWindowData数组，  
  // 数组的每一个元素代表了一个CAsyncSocketEx，即要服务的socket  
  //Initialize data  
  m_pAsyncSocketExWindowData = new t_AsyncSocketExWindowData[512]; //Reserve space for 512 active sockets  
  memset(m_pAsyncSocketExWindowData, 0, 512*sizeof(t_AsyncSocketExWindowData));  
  m_nWindowDataSize=512; // 当前数组的大小，这是可自动扩充的，不过最大不能超过一个最大值  
  m_nSocketCount=0; // 当前数组中CAsyncSocketEx的数量  
  m_nWindowDataPos=0; // 如果要加一个新的CAsyncSocketEx进来，加到数组的哪个位置  
  m_pThreadData = pThreadData; // 这个CAsyncSocketExHelperWindow对应的分发线程，两者一一对应的

  // 下面创建一个标准的窗口，不过并不显示出来  
  //Create window  
  WNDCLASSEX wndclass;  
  wndclass.cbSize=sizeof wndclass;  
  wndclass.style=0;  
  wndclass.lpfnWndProc=WindowProc;  
  wndclass.cbClsExtra=0;  
  wndclass.cbWndExtra=0;  
  wndclass.hInstance=GetModuleHandle(0);  
  wndclass.hIcon=0;  
  wndclass.hCursor=0;  
  wndclass.hbrBackground=0;  
  wndclass.lpszMenuName=0;  
  wndclass.lpszClassName=_T("CAsyncSocketEx Helper Window");  
  wndclass.hIconSm=0;

  RegisterClassEx(&wndclass);

  m_hWnd=CreateWindow(_T("CAsyncSocketEx Helper Window"), _T("CAsyncSocketEx Helper Window"), 0, 0, 0, 0, 0, 0, 0, 0, GetModuleHandle(0));  
  ASSERT(m_hWnd);  
  SetWindowLongPtr(m_hWnd, GWL_USERDATA, (LONG)this);  
 };

在调用完InitAsyncSocketExInstance()之后，CAsyncSocketEx的create()方法然后：

 SOCKET hSocket = socket(m_SocketData.nFamily, nSocketType, 0); // 这是真正的socket api，建立一个socket  
 if (hSocket == INVALID_SOCKET)  
  return FALSE;  
 m_SocketData.hSocket = hSocket;  
 AttachHandle(hSocket); // 将当前创建的socket加到分发线程管理中，这样可以让分发线程来负责这个socket的消息

AttachHandle()调用了CAsyncSocketExHelperWindow的AddSocket方法：  
BOOL AddSocket(CAsyncSocketEx *pSocket, int &nSocketIndex)  
{undefined  
 ...  
 //Search for free slot  
 // 从m_nWindowDataPos开始搜索, 共搜m_nWindowDataSize个位置  
 // 由于是下面的模运算i%m_nWindowDataSize，因此到达数组尾时，从重从绕回来，即收遍数组的每一个位置  
 for (int i=m_nWindowDataPos;i<(m_nWindowDataSize+m_nWindowDataPos);i++)  
 {undefined  
  // 注意模运算  
  if (!m_pAsyncSocketExWindowData[i%m_nWindowDataSize].m_pSocket) // 这个位置是空的  
  {undefined  
   m_pAsyncSocketExWindowData[i%m_nWindowDataSize].m_pSocket=pSocket;  
   nSocketIndex=i%m_nWindowDataSize; // 在list中的pos  
   m_nWindowDataPos=(i+1)%m_nWindowDataSize; // 以后从下一个搜索位置开台  
   m_nSocketCount++;  
   return TRUE;  
  }  
 }   
 ...  
}  
即 在CAsyncSocketExHelperWindow管理的socket数组中，加上这次的这个CAsyncSocketEx。

CAsyncSocketEx的create()方法然后：

 if (!AsyncSelect(lEvent))  
 {undefined  
  Close();  
  return FALSE;  
 }  
这 里AsyncSelect()方法里调用了windows socket api: WSAAsyncSelect()，这个方法可以让windows在CAsyncSocketEx指定的socket上，当socket事件 accept, read, write等发生时，发送消息到CAsyncSocketExHelperWindow中的窗口hWnd，然后 CAsyncSocketExHelperWindow再通过回调函数WindowProc将消息发回到负责处理这个消息的CAsyncSocketEx 上（这部分下面再详细分析）。

继续CAsyncSocketEx的create()方法：  
 if (!Bind(nSocketPort, lpszSocketAddress))  
 {undefined  
  Close();  
  return FALSE;  
 }  
Bind() 实际上调用了socket api: bind()方法，实现了local address和socket的绑定。

3、Listen  
create完以后，就是listen，这个比较简单，直接调用了socket api: listen()，在指定地址、端口进行监听。

程序运行到这里，核心的类已经初始化完成了，下面分析当socket有活动时，消息是如何从CAsyncSocketExHelperWindo分 发到CAsyncSocketEx的。

CAsyncSocketExHelperWindow的消息callback方法是WindowsProc，这是在创建窗口时指定的：  
 wndclass.lpfnWndProc=WindowProc;

WindowProc处理了5种类型的message  
(1) >= WM_SOCKETEX_NOTIFY  
(2) WM_USER  
 CAsyncSocketExLayer用到的消息，如果不使用layer，不会有这个消息  
(3) WM_USER+1  
(4) WM_USER + 2  
(5) WM_TIMER

我们先分析最重要的第一种消息，message>= WM_SOCKETEX_NOTIFY，这个消息是socket上有指定的lEvent发生时，发送来的。使用的windows api是WSAAsyncSelect()。

下面条件指定了，如果message大于WM_SOCKETEX_NOTIFY+pWnd->m_nWindowDataSize，就不处理 这条消息。  
if (message<static_cast<UINT>(WM_SOCKETEX_NOTIFY+pWnd->m_nWindowDataSize))  
{undefined  
 ...  
}

这里，WM_SOCKETEX_NOTIFY是一个基线，即所有socket消息中message都是从WM_SOCKETEX_NOTIFY开始 的，CAsyncSocketExHelperWindow的成员变量m_pAsyncSocketExWindowData维护了一个需要分发消息的 socket数组，根据FileZilla规则，socket在这个数组中的位置决定了消息的message值。即数组中第0个socket的所有消息 message都是WM_SOCKETEX_NOTIFY + 0，数组中第1个socket的所有消息message都是WM_SOCKETEX_NOTIFY + 1，依次类推，因此如果message大于WM_SOCKETEX_NOTIFY+pWnd->m_nWindowDataSize，实际就意味着 这不是一个合法的消息。这里，我们可以看一下create时，调用WSAAsyncSelect时的代码：  
  if ( !WSAAsyncSelect(m_SocketData.hSocket, GetHelperWindowHandle(), m_SocketData.nSocketIndex+WM_SOCKETEX_NOTIFY, lEvent) )  
   return TRUE;

参数中，  
m_SocketData.hSocket指的是CAsyncSocketEx通过socket()创建的windows socket，  
GetHelperWindowHandle()指的就是CAsyncSocketExHelperWindow，  
m_SocketData.nSocketIndex+WM_SOCKETEX_NOTIFY 指的就是发送消息的message值，是基数加上在数组中的位置：nSocketIndex  
lEvent指的是socket有何种event时， 发送消息

往下看消息的处理过程：  
 CAsyncSocketEx *pSocket=pWnd->m_pAsyncSocketExWindowData[message-WM_SOCKETEX_NOTIFY].m_pSocket;  
 SOCKET hSocket=wParam;  
 ...  
 int nEvent=lParam&0xFFFF;  
 int nErrorCode=lParam>>16;  
这里通过message-WM_SOCKETEX_NOTIFY找到这条消息所属的 CAsyncSocketEx，这里wParam指定了socket，这其实和pSocket是一回事，两者都对应一个socket，或者说是 CAsyncSocketEx。lParam指定发生了何种event。

一般情况下if (!pSocket->m_pFirstLayer)都会成立，即没有m_pFirstLayer，因此我们看if (!pSocket->m_pFirstLayer)对应的代码，  
switch (nEvent)  
{undefined  
 ...  
}  
显 然是对nEvent进行分析，中间代码有很多  
#ifndef NOSOCKETSTATES  
#endif //NOSOCKETSTATES  
这是用来设置socket状态的，暂时可以忽略，即可以删掉这个中间的代码，整个程序会清爽很多：

switch (nEvent)  
{undefined

case FD_READ:  
 if (pSocket->m_lEvent & FD_READ)  
 {undefined  
  DWORD nBytes = 0;  
  if (!nErrorCode)  
   if (!pSocket->IOCtl(FIONREAD, &nBytes)) // 看看能读多少个字节  
    nErrorCode = WSAGetLastError();  
  if (nBytes != 0 || nErrorCode != 0)  
   pSocket->OnReceive(nErrorCode);  
 }  
 break;  
case FD_FORCEREAD: //Forceread does not check if there's data waiting  
 if (pSocket->m_lEvent & FD_READ)  
 {undefined  
  pSocket->OnReceive(nErrorCode);  
 }  
 break;  
case FD_WRITE:  
 if (pSocket->m_lEvent & FD_WRITE)  
 {undefined  
  pSocket->OnSend(nErrorCode);  
 }  
 break;  
case FD_CONNECT:  
 if (pSocket->m_lEvent & FD_CONNECT)  
  pSocket->OnConnect(nErrorCode);  
 break;  
case FD_ACCEPT:  
 if (pSocket->m_lEvent & FD_ACCEPT)  
  pSocket->OnAccept(nErrorCode);  
 break;  
case FD_CLOSE:  
 pSocket->OnClose(nErrorCode);  
 break;  
}

很清楚，nEvent以下几种类型，分别调用了相应CAsyncSocketEx的On方法：  
 FD_READ  
  CAsyncSocketEx::OnReceive  
 FD_FORCEREAD  
  CAsyncSocketEx::OnReceive  
 FD_WRITE  
  CAsyncSocketEx::OnSend  
 FD_CONNECT  
  CAsyncSocketEx::OnConnect  
 FD_ACCEPT  
  CAsyncSocketEx::OnAccept  
 FD_CLOSE  
  CAsyncSocketEx::OnClose  
    
打 开这些On方法看一下，CAsyncSocketEx提供了空的实现，这是合理的，CAsyncSocketEx是基类，它并不清楚应该怎样处理这些消 息，这肯定应该由子类来实现这些On方法。

分析了CAsyncSocketEx类以后，我们大致清楚了整个socket处理的机制，下面就要对一个个具体的继承于 CAsyncSocketEx的类进行分析，看看到底怎样来处理这些从socket分发来的消息。

CListenSocket是CAsyncSocketEx类的子类，在启动的时候用来监听21端口。

 pListenSocket->Create(nPort, SOCK_STREAM, FD_ACCEPT, NULL)

可见，CListenSocket只处理FD_ACCEPT消息。

看一下：  
void CListenSocket::OnAccept(int nErrorCode)  
{undefined  
 CAsyncSocketEx socket;   
 if (!Accept(socket)) // 这里调用了win API: accept方法，建立了一个新连接，socket就这个新连接的SOCKET  
 {undefined  
  int nError = WSAGetLastError();  
  CStdString str;  
  str.Format(_T("Failure in CListenSocket::OnAccept(%d) - call to CAsyncSocketEx::Accept failed, errorcode

%d"), nErrorCode, nError);  
  SendStatus(str, 1);  
  SendStatus(_T("If you use a firewall, please check your firewall configuration"), 1);  
  return;  
 }

 // 权限检查，先不管  
 if (!AccessAllowed(socket))  
 {undefined  
  CStdStringA str = "550 No connections allowed from your IP\r\n";  
  socket.Send(str, str.GetLength());  
  return;  
 }

 // 检查FileZilla Server是否处于锁定状态，即不允许建立新连接  
 if (m_bLocked)  
 {undefined  
  CStdStringA str = "421 Server is locked, please try again later.\r\n";  
  socket.Send(str, str.GetLength());  
  return;  
 }

 // 下面从可用的线程中，找出目前负荷最小的线程，即线程中负责的connection最少的线程  
 int minnum = 255*255*255;  
 CServerThread *pBestThread=0;;  
 for (std::list<CServerThread *>::iterator iter=m_pThreadList->begin(); iter!=m_pThreadList->end(); iter++)  
 {undefined  
  int num=(*iter)->GetNumConnections();  
  if (num<minnum && (*iter)->IsReady()) // 找出connection最少的线程  
  {undefined  
   minnum=num;  
   pBestThread=*iter;  
   if (!num)  
    break;  
  }  
 }  
 if (!pBestThread)  
 {undefined  
  char str[] = "421 Server offline.";  
  socket.Send(str, strlen(str)+1);  
  socket.Close();  
  return;  
 }

 /* Disable Nagle algorithm. Most of the time single short strings get  
  * transferred over the control connection. Waiting for additional data  
  * where there will be most likely none affects performance.  
  */  
 BOOL value = TRUE;  
   
 socket.SetSockOpt(TCP_NODELAY, &value, sizeof(value), IPPROTO_TCP);  // 设置不使用Nagle算法，参见TCP协议Nagle算法部分

 SOCKET sockethandle = socket.Detach();

 pBestThread->AddSocket(sockethandle, m_ssl); // 转交服务线程来处理

 CAsyncSocketEx::OnAccept(nErrorCode); // 父类的缺省处理为空  
}

可见，CListenSocket::OnAccept主要工作是  
1、创建一个socket来接收新的客户端连接  
2、进行一些检查 设置，如权限检查，Nagle算法设置等  
3、找到一个负荷最小的后台服务线程，由交那个线程处理

为了更清楚服务线程CServerThread的机制，先看回顾一下当时这个服务线程是如何被创建的。

CServer类的Create()片断：

 for (int i = 0; i < num; i++) // 这里num是需要创建的服务线程的数量  
 {undefined  
  int index = GetNextThreadNotificationID(); // 得到这个线程的一个标识，即在线程数组std::vector<CServerThread*>

m_ThreadNotificationIDs中的index  
  CServerThread *pThread = new CServerThread(WM_FILEZILLA_SERVERMSG + index);  
  m_ThreadNotificationIDs[index] = pThread;  
  // If the CREATE_SUSPENDED flag is specified, the thread is created in a suspended state,   
  // and will not run until the ResumeThread function is called.   
  // If this value is zero, the thread runs immediately after creation.   
  if (pThread->Create(THREAD_PRIORITY_NORMAL, CREATE_SUSPENDED))  
  {undefined  
   pThread->ResumeThread();  
   m_ThreadArray.push_back(pThread);  
  }  
 }

看一下pThread->Create(THREAD_PRIORITY_NORMAL, CREATE_SUSPENDED)，由于CServerThread继承于CThread，因此调用了CThread的create：

BOOL CThread::Create(int nPriority /*=THREAD_PRIORITY_NORMAL*/, DWORD dwCreateFlags /*=0*/)  
{undefined  
 m_hThread=CreateThread(0, 0, ThreadProc, this, dwCreateFlags, &m_dwThreadId); // 调用win api创建一个线程  
 if (!m_hThread)  
 {undefined  
  delete this;  
  return FALSE;  
 }  
 ::SetThreadPriority(m_hThread, nPriority);  
 return TRUE;  
}  
注意创建线程的时候，指定线程的初始状态为 CREATE_SUSPENDED。

创建成功后，调用pThread->ResumeThread()：

DWORD CThread::ResumeThread()  
{undefined  
 // 下面使用win API：ResumeThread启动这个线程，线程开动后, 会自动跑到ThreadProc函数（create时指定）  
 DWORD res=::ResumeThread(m_hThread);  // 这个函数过后，有两个线程在跑，一个是刚才的主线程，一个是刚启动的线程  
 if (!m_started) // 主线程运行到这里，由于m_started还是0，所以通过下面的WaitForSingleObject，进行了等待状态  
 {undefined  
  WaitForSingleObject(m_hEventStarted, INFINITE);  
 }  
 return res;  
}

刚启动的线程进入了ThreadProc函数:  
DWORD WINAPI CThread::ThreadProc(LPVOID lpParameter)  
{undefined  
 // 在CreateThread时指定的参数LPVOID lpParameter为this，即CThread  
 return ((CThread *)lpParameter)->Run();  
}  
即 运行Run方法：  
DWORD CThread::Run()  
{undefined  
 InitInstance(); // 这里CServerThread类重写了这个方法，因此进入CServerThread::InitInstance()，进行了一些内存变量的初始

化  
 // The SetEvent function sets the specified event object to the signaled state.  
 SetEvent(m_hEventStarted); // 设置event为active，使得刚才在等待的主线程复活，继续CServer的启动工作  
 m_started = true;  
 MSG msg;  
 while (GetMessage(&msg, 0, 0, 0)) // 进入这个线程的消息循环  
 {undefined  
  TranslateMessage(&msg);  
  if (!msg.hwnd)  
   OnThreadMessage(msg.message, msg.wParam, msg.lParam); // 调用OnThreadMessage处理消息  
  DispatchMessage(&msg);  
 }  
 DWORD res=ExitInstance();  
 delete this;  
 return res;  
}

可见，服务器启动后，刚开始没有消息时，CServerThread在GetMessage()时进入了block状态，一旦有消息到来，这个服务 线程就苏醒，接着

处理消息。

下面回到最初的CListenSocket::OnAccept()，最后调用了 pBestThread->AddSocket(sockethandle, m_ssl); // 转交服务线程来处理

仔细看一下后台的服务线程是如何处理消息的。  
void CServerThread::AddSocket(SOCKET sockethandle, bool ssl)  
{undefined  
 // 调用了父类的方法  
 PostThreadMessage(WM_FILEZILLA_THREADMSG, ssl ? FTM_NEWSOCKET_SSL : FTM_NEWSOCKET, (LPARAM)sockethandle);  
}

接着：  
BOOL CThread::PostThreadMessage(UINT message, WPARAM wParam, LPARAM lParam)  
{undefined  
 // posts a message to the message queue of the specified thread.  
 BOOL res=::PostThreadMessage(m_dwThreadId, message, wParam, lParam);;  
 ASSERT(res);  
 return res;  
}

PostThreadMessage是windows API，作用是把消息message发送到线程m_dwThreadId，

根据前面的代码，在这里就是把消息WM_FILEZILLA_THREADMSG，以及参数FTM_NEWSOCKET, sockethandle发送到那个负荷最小的后台服务线程，

由于在启动时，那个后台线程处于GetMessage()的block中，因此收到这个消息到，那个后台线程苏醒，接着调用 OnThreadMessage来处理这个WM_FILEZILLA_THREADMSG消息。

服务线程苏醒后，调用OnThreadMessage来处理这个WM_FILEZILLA_THREADMSG消息，参数是 FTM_NEWSOCKET, sockethandle，接着进入AddNewSocket方法，表示有一个新的客户端需要连接上来。

void CServerThread::AddNewSocket(SOCKET sockethandle, bool ssl)  
{undefined  
 // 首先创建了新的CControlSocket类，这也是继承于CAsyncSocketEx类的，从下面起经过一些初始化之后，就由这个 CControlSocket类来接管这个客户连接了。  
   
 CControlSocket *socket = new CControlSocket(this);  
 socket->Attach(sockethandle); // 加入到FileZilla消息机制中, 建立与分发线程等之间的关系  
 CStdString ip;  
 unsigned int port;  
   
 SOCKADDR_IN sockAddr;  
 memset(&sockAddr, 0, sizeof(sockAddr));  
 int nSockAddrLen = sizeof(sockAddr);  
 BOOL bResult = socket->GetPeerName((SOCKADDR*)&sockAddr, &nSockAddrLen); // 获取socket客户端的信息  
 if (bResult)  
 {undefined  
  port = ntohs(sockAddr.sin_port); // 端口  
  ip = inet_ntoa(sockAddr.sin_addr); // IP地址  
 }  
 else  
 {undefined  
  socket->m_RemoteIP = _T("ip unknown");  
  socket->m_userid = 0;  
  socket->SendStatus(_T("Can't get remote IP, disconnected"), 1);  
  socket->Close();  
  delete socket;  
  return;  
 }  
 socket->m_RemoteIP=  ip;  
 EnterCritSection(m_GlobalThreadsync);  
 int userid = CalcUserID(); // 自动为当前socket连接生成一个客户号：userID  
 if (userid == -1)  
 {undefined  
  LeaveCritSection(m_GlobalThreadsync);  
  socket->m_userid = 0;  
  socket->SendStatus(_T("Refusing connection, server too busy!"), 1);  
  socket->Send(_T("421 Server too busy, closing connection. Please retry later!"));  
  socket->Close();  
  delete socket;  
  return;  
 }  
 socket->m_userid = userid;  
 t_socketdata data;  
 data.pSocket = socket;  
 data.pThread = this;  
 m_userids[userid] = data; // m_userids是static的,定义为static std::map<int, t_socketdata> m_userids;

 // hammering这块可以先不管  
 // Check if remote IP is blocked due to hammering  
 std::map<DWORD, int>::iterator iter = m_antiHammerInfo.find(sockAddr.sin_addr.s_addr);  
 if (iter != m_antiHammerInfo.end())  
 {undefined  
  if (iter->second > 10)  
   socket->AntiHammerIncrease(25); // ~6 secs delay  
 }  
 LeaveCritSection(m_GlobalThreadsync);  
 EnterCritSection(m_threadsync);  
   
 // 下面记录这个服务线程所处理的CControlSocket  
 m_LocalUserIDs[userid] = socket;   
 LeaveCritSection(m_threadsync);

 t_connectiondata_add *conndata = new t_connectiondata_add;  
 t_connop *op = new t_connop;  
 op->data = conndata;  
 op->op = USERCONTROL_CONNOP_ADD; // 新用户连接即将连接，在CServer的OnServerMessage中要用过  
 op->userid = userid;  
 conndata->pThread = this;

 memset(&sockAddr, 0, sizeof(sockAddr));  
 nSockAddrLen = sizeof(sockAddr);  
 bResult = socket->GetPeerName((SOCKADDR*)&sockAddr, &nSockAddrLen);  
 if (bResult)  
 {undefined  
  conndata->port = ntohs(sockAddr.sin_port);  
#ifdef _UNICODE  
  _tcscpy(conndata->ip, ConvFromLocal(inet_ntoa(sockAddr.sin_addr))); // 拷贝字符串  
#else  
  _tcscpy(conndata->ip, inet_ntoa(sockAddr.sin_addr));  
#endif  
 }

 // 这里往全局的hMainWnd发送消息,   
 // 消息的wParam类型为FSM_CONNECTIONDATA, 指示消息是跟connection相关的消息，参数是t_connop  
 // 这些在CServer的WindowProc中处理这个消息时用到，这个消息处理结束后，会在admin窗口的下边显示  
 // 类似000001 (not logged in) 127.0.0.1 的信息  
 SendNotification(FSM_CONNECTIONDATA, (LPARAM)op);

 if (ssl) // SSL相关, 可以先跳过  
  if (!socket->InitImplicitSsl())  
   return;  
   
 socket->AsyncSelect(FD_READ|FD_WRITE|FD_CLOSE); // 对socket上这些event建立侦听关系  
   
 // SendStatus最终还是调用SendNotification方法，不过发送的参数是FSM_STATUSMESSAGE，  
 // 因此在CServer中的处理并不一样  
 socket->SendStatus(_T("Connected, sending welcome message..."), 0);

 // 这时，admin窗口的下半部分会显示类似(000003) 2006-8-24 3:26:47 - (not logged in) (127.0.0.1)> Connected, sending welcome message...  
   
 // 下面格式化欢迎信息  
 CStdString msg = m_pOptions->GetOption(OPTION_WELCOMEMESSAGE);  
 if (m_RawWelcomeMessage != msg)  
 {undefined  
  m_RawWelcomeMessage = msg;  
  m_ParsedWelcomeMessage.clear();

  msg.Replace(_T("%%"), _T("\001"));  
  msg.Replace(_T("%v"), GetVersionString());  
  msg.Replace(_T("\001"), _T("%"));  
   
  ASSERT(msg != _T(""));  
  int oldpos = 0;  
  msg.Replace(_T("\r\n"), _T("\n"));  
  int pos=msg.Find(_T("\n"));  
  CStdString line;  
  while (pos!=-1)  
  {undefined  
   ASSERT(pos);  
   m_ParsedWelcomeMessage.push_back(_T("220-") +  msg.Mid(oldpos, pos-oldpos) );  
   oldpos=pos + 1;  
   pos=msg.Find(_T("\n"), oldpos);  
  }  
   
  line = msg.Mid(oldpos);  
  if (line != _T(""))  
   m_ParsedWelcomeMessage.push_back(_T("220 ") + line);    
  else  
  {undefined  
   m_ParsedWelcomeMessage.back()[3] = 0;  
  }  
 }  
   
 // hideStatus指示这个欢迎消息要不要发给admin port  
 bool hideStatus = m_pOptions->GetOptionVal(OPTION_WELCOMEMESSAGE_HIDE) != 0;  
 ASSERT(!m_ParsedWelcomeMessage.empty());  
 for (std::list<CStdString>::iterator iter = m_ParsedWelcomeMessage.begin(); iter != m_ParsedWelcomeMessage.end(); iter++)  
  // 发送给socket客户, 并且发送消息到admin port上，发送的参数是FSM_STATUSMESSAGE  
  if (!socket->Send(*iter, !hideStatus))   
   break;  
     
 // 运行到这里，客户的登录界面上、admin窗口上半部已经出现了welcome信息，类似：  
 // (000003) 2006-8-24 3:27:19 - (not logged in) (127.0.0.1)> 220-FileZilla Server version 0.9.18 beta  
 // ((000003) 2006-8-24 3:27:22 - (not logged in) (127.0.0.1)> 220-written by Tim Kosse ([Tim.Kosse@gmx.de](mailto:Tim.Kosse@gmx.de))  
 // ((000003) 2006-8-24 3:27:29 - (not logged in) (127.0.0.1)> 220 Please visit[http://sourceforge.net/projects/filezilla/](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2Fprojects%2Ffilezilla%2F)  
}

看一下发消息的过程：

void CServerThread::SendNotification(WPARAM wParam, LPARAM lParam)  
{undefined  
 EnterCritSection(m_threadsync);  
 t_Notification notification;  
 notification.wParam = wParam;  
 notification.lParam = lParam;

 // 由于在创建这个线程时, 指定了m_nNotificationMessageI = WM_FILEZILLA_SERVERMSG + index  
 // 因此在CServer的WindowProc处理中, 通过message就可以知道是哪个服务线程发来的消息  
 // 也就得到了这个线程的m_pendingNotifications，m_pendingNotifications有详细的wParam, lParam，  
 // CServer然后再调用CServer的OnServerMessage处理这个消息  
 // 因此实际上PostMessage只是发了个消息，关于这个消息的具体的信息还在m_pendingNotifications中  
 if (m_pendingNotifications.empty())   
  PostMessage(hMainWnd, m_nNotificationMessageId, 0, 0); // hMainWnd是全局变量,就是CServer在Create时创建的那个窗口

 m_pendingNotifications.push_back(notification);

 // Check if main thread can't handle number of notifications fast enough, throttle thread if neccessary  
 // 下面检查m_pendingNotifications里的数量，如果太多了，就降低这个线程的优先级，  
 // 让这个线程运行得慢一些  
 if (m_pendingNotifications.size() > 200 && m_throttled < 3)  
 {undefined  
  SetPriority(THREAD_PRIORITY_IDLE);  
  m_throttled = 3;  
 }  
 else if (m_pendingNotifications.size() > 150 && m_throttled < 2)  
 {undefined  
  SetPriority(THREAD_PRIORITY_LOWEST);  
  m_throttled = 2;  
 }  
 else if (m_pendingNotifications.size() > 100 && !m_throttled)  
 {undefined  
  SetPriority(THREAD_PRIORITY_BELOW_NORMAL);  
  m_throttled = 1;  
 }  
   
 LeaveCritSection(m_threadsync);  
}

到这里我们回去看一下CServer的相关代码：  
LRESULT CALLBACK CServer::WindowProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)  
{undefined  
 ...  
   
 else if (message >= WM_FILEZILLA_SERVERMSG) // 指定的消息种类  
 {undefined  
  UINT index = message - WM_FILEZILLA_SERVERMSG;  
  if (index >= pServer->m_ThreadNotificationIDs.size())  
   return 0;

  CServerThread *pThread = pServer->m_ThreadNotificationIDs[index]; // 发这个消息的CServerThread  
  if (pThread)  
  {undefined  
   std::list<CServerThread::t_Notification> notifications;  
   // 与pThread中原有的m_pendingNotifications进行交换，  
   // 等于是处理了pThread中所有的notifications，然后再删掉  
   pThread->GetNotifications(notifications);   
   for (std::list<CServerThread::t_Notification>::const_iterator iter = notifications.begin(); iter != notifications.end(); iter++)  
    if (pServer->OnServerMessage(pThread, iter->wParam, iter->lParam) != 0)  
     break;  
  }  
  return 0;  
 }  
   
 return ::DefWindowProc(hWnd, message, wParam, lParam);  
}

处理函数：

LRESULT CServer::OnServerMessage(CServerThread* pThread, WPARAM wParam, LPARAM lParam)  
{undefined  
 // 发送的参数是FSM_STATUSMESSAGE在这里处理  
 if (wParam == FSM_STATUSMESSAGE) // 记录当前活动, 将活动信息在admin窗口上显示, 并且记录到log文件中  
 {undefined  
  t_statusmsg *msg = reinterpret_cast<t_statusmsg *>(lParam);  
  ...  
 }  
 // 发送的参数是FSM_CONNECTIONDATA在这里处理  
 else if (wParam == FSM_CONNECTIONDATA) // 跟connection相关的消息  
 {undefined  
  // lParam这里t_connop类型的，与AddNewSocket中的对应  
  t_connop *pConnOp = reinterpret_cast<t_connop*>(lParam);    
  if (!pConnOp)  
   return 0;  
    
  int len;  
  unsigned char *buffer;

  switch (pConnOp->op)  
  {undefined  
  case USERCONTROL_CONNOP_ADD: // 新用户连接即将连接，在AddNewSocket时指定的  
 ...  
}

仔细比较一下AddNewSocket发送消息和OnServerMessage处理消息的代码，两者肯定是一一对应的。

最后看一下CControlSocket::Send的代码，在AddNewSocket中，发送客户端是些welcome信息。在发送给 客户端的过程中，要求发送的数据实际上可能没有全部发送出去，这时程序把没有发出去的数据放到内存m_pSendBuffer中，然后给 HelperWindow发送事件FD_WRITE，经过消息的分发机制后，程序最终回到CControlSocket的OnOnSend方法，在 OnOnSend中，再根据m_pSendBuffer中的数据进行重新发送。

在进行代码调试的时候，最好打开FileZilla Server Interface.exe，仔细看一下随着每一步的进行，在admin窗口上发生了什么。

程序运行完AddNewSocket后，只是在客户端显示了welcome信息，下面用户要登录到FileZilla Server了。

FTP客户通过ftp localhost命令与FileZilla服务器建立socket连接后，FileZilla Server显示了welcome信息，这时屏幕上显示类似（我们以windows下的ftp命令作为sample）：

Connected to dell.  
220-FileZilla Server version 0.9.18 beta  
220-written by Tim Kosse ([Tim.Kosse@gmx.de](mailto:Tim.Kosse@gmx.de))  
220 Please visit [http://sourceforge.net/projects/filezilla/](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2Fprojects%2Ffilezilla%2F)  
User (dell:(none)):

提示输入用户名，假设这时用户输入whg，回车，这时ftp客户端会将这用户输入的字符翻译成标准的FTP命令"USER whg"发送到服务器，因为这时是CControlSocket对这个socket进行监听，并且recv相关的事件通过前面提到的分发机制，最终分发到 CControlSocket的OnReceive方法，下面我看一下这个方法：

m_antiHammeringWaitTime还不知是起什么作用，在对源代码进行跟踪的时候，其刚开始的值是0，因此先跳过这个。

下段是获得传输速度限制SpeedLimit，如果没有限制，则为-1。

再往下：  
 int numread = Receive(buffer, len); // 调用recv来获得socket数据，取长度为len的数据放到buffer中  
   
读取成功后，将buffer中的接收到的数据一个字节一 个字节放到m_RecvBuffer中：  
 m_RecvBuffer[m_nRecvBufferPos++] = buffer[i];

然后将将刚才收到的m_RecvBuffer放入m_RecvLineBuffer：  
 m_RecvLineBuffer.push_back(m_RecvBuffer);  
   
m_RecvLineBuffer 相当于一个命令池，里面存放着用户发送来，但还没有处理的命令。

最后当这个recv处理完后，调用ParseCommand()来解释这个命令。

首先通过GetCommand()取出m_RecvLineBuffer中最前面的命令，并解释成命令command，以及参数args，如刚才的 命令USER whg就被解释成command=USER, args=whg

下面的循环：  
 for (int i = 0; i < (sizeof(commands)/sizeof(t_command)); i++)  
通过在预先定义的FTP Server所有命令commands中，查找是否包含command，从而校验刚才收到的命令的合法性，如果command不在commands中，显 示command是非法命令，这时发送客户端   
 Send(_T("500 Syntax error, command unrecognized."));

即使命令是合法的，但如果参数不对（bHasargs指定这个命令是否需要参数），即有些命令必须带参数，而args没有，这时会发送：  
 Send(_T("501 Syntax error"));  
下面：  
 if (!m_RecvLineBuffer.empty())  
  m_pOwner->PostThreadMessage(WM_FILEZILLA_THREADMSG, FTM_COMMAND, m_userid);  
表示如果命令缓冲区中还有未处理的命令，则发送消息给 CServerThread，CServerThread在方法OnThreadMessage中处理这个消息：  
 else if (wParam==FTM_COMMAND)  
 { //Process a command sent from a client  
  CControlSocket *socket=GetControlSocket(lParam);  
  if (socket)  
   socket->ParseCommand();  
 }  
在 GetControlSocket()方法中：  
CControlSocket * CServerThread::GetControlSocket(int userid)  
{undefined  
 CControlSocket *ret=0;  
 EnterCritSection(m_threadsync);  
 // 下面这个map是user -> CControlSocket，即通过userid找到服务这个userid的CControlSocket  
 std::map<int, CControlSocket *>::iterator iter=m_LocalUserIDs.find(userid);  
 if (iter!=m_LocalUserIDs.end())  
  ret=iter->second;  
 LeaveCritSection(m_threadsync);  
 return ret;  
}

可见，发送这个消息的作用是让CControlSocket继续调用ParseCommand()来处理下一个命令。

回到最初的ParseCommand()，如果命令参数也没有问题，下面检查这个命令是否必须先登录再使用（由 bValidBeforeLogon指定），比如：get命令是必须先登录的，而USER命令不用，如果必须先登录，发送：  
 Send(_T("530 Please log in with USER and PASS first."));  
下面同样  
 m_pOwner->PostThreadMessage(WM_FILEZILLA_THREADMSG, FTM_COMMAND, m_userid);

命令都合格的话，下面：  
 switch (nCommandID)  
来处理不同的命令，由于这时是COMMAND_USER命令，我 们看一下处理过程：  
经过一些处理后，下面发送  
 Send(_T("331 Password required for ") + args);  
要求用户输入密码，时客户端屏幕上会显示：  
331 Password required for whg  
Password:

用户输入密码后，回车，这时ftp客户端会翻译成标准的FTP命令"PASS 123456"发送到服务器，我们看一下ParseCommand()对这的处理：  
 case COMMAND_PASS:  
  else if (DoUserLogin(args))  
   Send(_T("230 Logged on"));  
在 DoUserLogin()认定成功登录后，发送成功登录消息给客户端，否则会发送错误消息：  
 Send(_T("530 Login or password incorrect!"));  
   
仔细看一下CPermissions::CheckUserLogin()，会发现密 码是经过MD5加密的，并且在CServerThread创建时，跟权限相关的成员变量就初始化了：  
m_pPermissions = new CPermissions;

在CPermissions::Init()中，调用ReadSettings()，从配置文件中，将所有的用户信息（包括密码）都读到内存了，因 此刚才的密码校验只是内存中的字符串比对。

用户成功登录后，FTP客户端显示：

C:\Documents and Settings\Administrator>ftp localhost  
Connected to dell.  
220-FileZilla Server version 0.9.18 beta  
220-written by Tim Kosse ([Tim.Kosse@gmx.de](mailto:Tim.Kosse@gmx.de))  
220 Please visit [http://sourceforge.net/projects/filezilla/](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2Fprojects%2Ffilezilla%2F)  
User (dell:(none)): whg  
331 Password required for whg  
Password:  
230 Logged on  
ftp>

下面FTP服务器等待新的FTP命令了。

在进一步分析代码之前，先复习一下FTP协议，下图是FTP的结构图。

![](https://p-blog.csdn.net/images/p_blog_csdn_net/wanghaoguang/Slide0001.gif)

客户端和服务器是通过两个连接来进行通讯的：

一个是控制连接，也就是传输些控制命令，客户端发出FTP命令，服务器给出应答，例如：USER，PASS命令等等。这个连接中，FTP服务器的端 口就是熟知的21端口，连接是由客户端发起的，例如：ftp 192.168.0.1。有一点注意，用户是通过“用户接口”来操作的，一般的用户接口是指cuteFTP这些FTP客户端，或者ftp.exe这种命令 行程序，用户在用户接口使用的是ftp命令，如ls, get, cd等，这些ftp命令并不是真正与FTP服务器交互的命令，这些ftp命令还需要由“用户协议解释器”翻译成真正的ftp协议命令，如USER, PASS，才能与服务器进行交互。

一个是数据连接，即真正的文件传输是在这个连接上进行的。服务器端的数据连接端口是20，客户端的数据连接端口是随机生成的。数据连接只在传输文件 时存在，文件传完后，这个连接就断了，如果需要再次传送文件，会再次建立一个数据连接（客户端的端口是随机的，不一定是上次的那个）。数据连接的模式有两 种，一种是主动方式，一种是被动方式，两者的区别在于数据连接是由谁发起。

我们来看一个典型的FTP交互过程，用的是windows的ftp.exe程序，先建立一个连接，然后ls看一下文件列表，用get命令下 载一个文件，最后quit关闭。下面-d选项可以显示交互的细节，注意-->开头的行是ftp客户端发给FTP服务器的请求，3个数字开头的行是服 务器的应答，如220, 331等开头的行：

C:\>ftp -d localhost  
Connected to dell.  
220-FileZilla Server version 0.9.18 beta  
220-written by Tim Kosse ([Tim.Kosse@gmx.de](mailto:Tim.Kosse@gmx.de))  
220 Please visit [http://sourceforge.net/projects/filezilla/](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2Fprojects%2Ffilezilla%2F)  
User (dell:(none)): robert  
---> USER robert  
331 Password required for robert  
Password:  
---> PASS test  
230 Logged on  
ftp> ls  
---> PORT 127,0,0,1,4,173  
200 Port command successful  
---> NLST  
150 Opening data channel for directory list.  
Manual.txt  
226 Transfer OK  
ftp: 收到 175 字节，用时 0.00Seconds 175000.00Kbytes/sec.  
ftp> get Manual.txt  
---> PORT 127,0,0,1,4,174  
200 Port command successful  
---> RETR Manual.txt  
150 Opening data channel for file transfer.  
226 Transfer OK  
ftp: 收到 17319 字节，用时 0.09Seconds 192.43Kbytes/sec.  
ftp> quit  
---> QUIT  
221 Goodbye

C:\>

刚开始，客户端发出建立连接的请求：  
C:\>ftp -d localhost // 建立连接  
Connected to dell. // 连接已建立

然后服务器发送欢迎信息，并要求输入用户名：  
220-FileZilla Server version 0.9.18 beta  
220-written by Tim Kosse ([Tim.Kosse@gmx.de](mailto:Tim.Kosse@gmx.de))  
220 Please visit [http://sourceforge.net/projects/filezilla/](http://blog.chinaunix.net/link.php?url=http://sourceforge.net%2Fprojects%2Ffilezilla%2F)  
User (dell:(none)):

客户端输入用户名robert，然后回车：  
---> USER robert // ftp.exe生成FTP命令：USER，发送给服务器

服务器要求输入密码：  
331 Password required for robert  
Password:

客户端输入密码，然后回车：  
---> PASS test // ftp.exe生成FTP命令：PASS，发送给服务器

服务器通过密码验证：  
230 Logged on

客户端键入ls命令  
ftp> ls

ftp.exe生成FTP命令：PORT，告诉服务器客户端的随机端口是什么  
---> PORT 127,0,0,1,4,173 // 127,0,0,1是IP地址，4 * 256 + 173 = 1197是随机端口号  
200 Port command successful // 服务器响应PORT命令  
---> NLST // 客户端发出NLST命令，要求列出文件列表  
150 Opening data channel for directory list. // 服务器会在20端口与客户端的1197端口建立数据连接，传输数据，注意ls命令的结果是在“数据连接”中传输的  
Manual.txt // 只有一个文件  
226 Transfer OK // FTP服务器响应，传输完毕  
ftp: 收到 175 字节，用时 0.00Seconds 175000.00Kbytes/sec. // FTP客户端显示的传输结果

下面客户端要求下载Manual.txt文件  
ftp> get Manual.txt

---> PORT 127,0,0,1,4,174 // 告诉服务器客户端新的随机端口4 * 256 + 174 = 1198  
200 Port command successful // // 服务器响应PORT命令  
---> RETR Manual.txt // 告诉服务器下载Manual.txt文件  
150 Opening data channel for file transfer. // 服务器会在20端口与客户端的1198端口建立数据连接，传输数据  
226 Transfer OK // FTP服务器响应，传输完毕  
ftp: 收到 17319 字节，用时 0.09Seconds 192.43Kbytes/sec. // FTP客户端显示的传输结果

最后客户端退出  
ftp> quit  
---> QUIT // 发出QUIT命令  
221 Goodbye // 服务器最后响应

仔细阅读上面的交互过程，可以发现，用户手工输入的一个FTP命令，可能会被ftp.exe处理成与FTP服务器的多次交互。如ls, get命令。

要想详细了解FTP命令的细节，可以参见FTP的RFC，或者相关的资料，不过由于我们阅读源代码的主要目的不是研究FTP细节，而在于掌握高并发 的网络编程的技术，所以，我们只以上面这个简单的FTP交互来看一下，在代码中这个过程是如何实现的。  

前面已经分析过了FTP客户登录服务器的过程，现在来看一下常见的ls命令的处理过程。

用户在FTP客户端输入ls命令后，ftp.exe首先发出port请求给服务器，在CControlSocket的ParseCommand() 中被处理。

PORT命令的参数是形如：127.0.0.1.4.9，前4个表示客户端的IP地址，后两个根据规则4 * 256 + 9 = 1033，表示FTP客户端临时建立的用来与服务器建立数据连接的端口，例子所示为1033端口。

PORT命令的处理过程的代码中前面都是用来获取IP和临时端口的：

case COMMAND_PORT:  
 ...  
 port += 256 * _ttoi(args.Right(args.GetLength() - (i + 1))); // add ms byte to server socket  
 ip = args.Left(i);  
 ...

下面：  
 m_transferstatus.ip = ip;  
 m_transferstatus.port = port;  
 m_transferstatus.pasv = 0;  
 Send(_T("200 Port command successful"));  
 break;  
只是将FTP客 户端提供的临时端口记录到m_transferstatus中，然后发出200 Port command successful，等待FTP客户端的下一个命令。由于用户输入的是ls命令，ftp.exe在PORT之后，发出NLST命令。

在case COMMAND_NLST的处理中,先是进行了一系列的参数、权限检查，一切OK后：  
 if (!m_transferstatus.pasv) // 主动模式  
 {undefined  
  ...  
 }  
 else // 被动模式  
 {undefined  
  ...  
 }  
由 于主动模式是缺省值，因此看一下里面的代码：  
CTransferSocket *transfersocket = new CTransferSocket(this);  
m_transferstatus.socket = transfersocket;  
transfersocket->Init(pResult, TRANSFERMODE_NLST); // 只是一些参数的初始化  
if (m_transferMode == mode_zlib) // 传输方式是否使用压缩方式，缺省不使用，详细参见FTP规范  
{undefined  
 if (!transfersocket->InitZLib(m_zlibLevel))  
 {undefined  
  Send(_T("550 could not initialize zlib, please use MODE S instead"));  
  ResetTransferstatus();  
  break;  
 }  
}

if (!CreateTransferSocket(transfersocket)) // 建立数据连接  
 break;

SendTransferinfoNotification(TRANSFERMODE_LIST, physicalDir, logicalDir); // Use TRANSFERMODE_LIST instead of TRANSFERMODE_NLST.  
Send(_T("150 Opening data channel for directory list."));

先看一下建立数据连接的代码：  
BOOL CControlSocket::CreateTransferSocket(CTransferSocket *pTransferSocket)  
{undefined  
 ...  
 if (pTransferSocket->Connect(m_transferstatus.ip,m_transferstatus.port)==0)  
 ...  
}  
无 非是常规的socket方法建立连接，需要关注的是由服务主动发起连接，这正是主动模式的含义。我们先看完这一段,再看一下被动模式。

在CreateTransferSocket()完成后，调用：

SendTransferinfoNotification(TRANSFERMODE_LIST, physicalDir, logicalDir);

看一下里面：  
void CControlSocket::SendTransferinfoNotification(const char transfermode, const CStdString& physicalFile, const CStdString& logicalFile, __int64 startOffset, __int64 totalSize)  
{undefined  
 t_connop *op = new t_connop;  
 op->op = USERCONTROL_CONNOP_TRANSFERINIT;  
 op->userid = m_userid;

 t_connectiondata_transferinfo *conndata = new t_connectiondata_transferinfo;  
 conndata->transferMode = transfermode;  
 conndata->physicalFile = physicalFile;  
 conndata->logicalFile = logicalFile;  
 conndata->startOffset = startOffset;  
 conndata->totalSize = totalSize;  
 op->data = conndata;

 m_pOwner->SendNotification(FSM_CONNECTIONDATA, (LPARAM)op);  
}  
可 见发送了一个消息给CServer，wParam参数是FSM_CONNECTIONDATA，表示这是跟connection相关的消息，lParam 带的参数是USERCONTROL_CONNOP_TRANSFERINIT，表示传输开始或结束，我回去看一下CServer中的 OnServerMessage()相关代码，在admin窗口的下面显示了将用传输的信息。

下面，  
Send(_T("150 Opening data channel for directory list."));  
发 送给FTP客户端数据连接创建的消息，真正的数据传输的任务是交给数据连接了，即CTransferSocket。

我们回到被动模式，如果是被动模式：  
if (!m_transferstatus.pasv)  
{undefined  
 ...  
}  
else // 被动模式  
{undefined  
 ...  
 m_transferstatus.socket->PasvTransfer();  
}  
看 一下PasvTransfer()的实现：

void CTransferSocket::PasvTransfer()  
{undefined  
 if(bAccepted)  
  if (!m_bStarted)  
   InitTransfer(FALSE);  
}

非常简单，由于是被动模式，即由客户端发起数据连接，因此CTransferSocket只需等待客户端的连接就可以了，下面分析 CTransferSocket的时候再仔细看一下相关的实现。
