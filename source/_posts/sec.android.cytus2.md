---
title: 国服Cytus2解密与注入
date: 2019-07-20 14:51:54
categories: reverse
tags: [reverse]
---

本文详细记录了一次对国服Cytus2的研究，该样本采用企业版360壳，并对U3D的mono运行时做出了独到的加密。

<!-- more -->

## 目标
国服Cytus2是Unity3d游戏，未采用il2cpp运行时，而是mono。这给Cytus2自制（导入自行制作的关卡）留下可能。
目标：修改Assembly-CSharp.dll并成功运行新增的代码。
## 资源
本文以com.ilongyuan.cytus2.ly.TapTap-2300.apk为对象，版本为2.3.0。
其余使用IDA 7.0，一部root安卓手机（本人型号Mi 6, 8.1, LineageOS 15.1)。

## 写在前面
仅供学习使用，任何用于非法用途后果自负。
本文主要用于记录2019年初的移动应用安全手段。
本文省略的技术都可以被百度到。
文章对libjiagu.so的分析实际上对实现目标并无作用。
## Step 1
安卓机设为全局debuggable。
安装Taptap与游戏。
复制apk到电脑上，用ApkTool Box解包。 
先看Assembly-CSharp.dll，显然是被加密了。
![](https://i.loli.net/2019/07/20/5d32d26d8c6eb61824.png)
首先是文件头被改了，mz可以作为一个突破口。其次应该是分段表被加密了。后面还知道所有的il指令全部被改写。
## Step 2
看了一下libjiagu.so，应该是一款360企业版壳。从这里开始，笔者着手开始研究libjiagu.so的行为，实际上是走上了弯路，对于目标来说，没有什么必要，你可以**跳过**这一章节。
连接手机到电脑，手机上装好debugserver，libjiagu.so扔进IDA分析好，然后给Cytus2安排上

```
adb shell am start -D -n "com.ilongyuan.cytus2.ly.TapTap/com.ilongyuan.cytus2.remaster.MainActivity"
```
定位到JNI_OnLoad，先下个断点。
![](https://i.loli.net/2019/07/20/5d32d26f8fb6d52914.png)
先打开Android Device Monitor。

```
H:\AndroidSdk\tools\monitor.bat
```
![](https://i.loli.net/2019/07/20/5d32d2706e1e592234.png)
然后设置一下端口转发，并启动手机端的debugserver（在IDA目录dbgsrv，建议改个名字，这里随便改了个yy，然后放到手机里，并赋予执行权限）

```
adb root
adb forward tcp:23947 tcp:23947
adb shell /data/data/yy -p23947
```
![](https://i.loli.net/2019/07/20/5d32d270d355757816.png)
然后IDA连接到手机的dbgsrv，注意端口号不要23946，如上，23947或其他都行。 

![](https://i.loli.net/2019/07/20/5d32d27259fb210172.png)
![](https://i.loli.net/2019/07/20/5d32d272e55db35774.png)
![](https://i.loli.net/2019/07/20/5d32d2745ca6443332.png)
![](https://i.loli.net/2019/07/20/5d32d2751fbbf84384.png)
这个一律YES，其他提示如不特别说，都是YES。
![](https://i.loli.net/2019/07/20/5d32d2759027c29383.png)
然后等IDA加载完成，停在libc之后，attach jdb让进程恢复运行。

```
jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700
```
这个一律SAME。
![](https://i.loli.net/2019/07/20/5d32d2767864f97094.png)
这样就停在最开始那个JNI_OnLoad断点了，可以开始分析一番。
![](https://i.loli.net/2019/07/20/5d32d27ba7b4b45150.png)
废话不多说，一路跟进
![](https://i.loli.net/2019/07/20/5d32d27c3032324636.png)
![](https://i.loli.net/2019/07/20/5d32d27ca5bf760563.png)
![](https://i.loli.net/2019/07/20/5d32d27d1684767294.png)
来到了重要的地方，这里程序将完成一系列反调试，反动态分析的手段。
![](https://i.loli.net/2019/07/20/5d32d27dbac6b72112.png)
经过长时间的跟踪分析，具体是在case 31进行的。
![](https://i.loli.net/2019/07/20/5d32d27e5dfa356862.png)
在这里下个断，先不急着恢复运行，在Module表中找到linker。
![](https://i.loli.net/2019/07/20/5d32d27ed7e3392122.png)
找到rtld_db_dlactivity，稍后程序第一步反调就是判断这里是否被设为断点指令。
![](https://i.loli.net/2019/07/20/5d32d27f428be10421.png)
双击进入
![](https://i.loli.net/2019/07/20/5d32d27fb98f884100.png)
把0x10 0xDE改为0x00 0xBF（nop），以后每次重新调试都得改。
现在可以恢复执行了。
![](https://i.loli.net/2019/07/20/5d32d28051b7292824.png)
断点命中，跟进去看看干了些什么。由于我以前跟的是2.1.1版本，2.3.0好像没什么改变，且比较费时，这里只说明反调手段及反制措施。先退出调试。
0.gettimeofday，后面还会进行一次，目的就是判断是否被调试了，被调试运行时间一定会非常长。反制手段就是直接返回0。
![](https://i.loli.net/2019/07/20/5d32d28584df819409.png)
1.两次memcpy，两次解码（xor5A，按位取反，程序内重要字符串被简单加密了），及后续操作是判断linker中的rtld_db_dlactivity是否为0xDE10。反制措施如上。
![](https://i.loli.net/2019/07/20/5d32d2861e47920947.png)
2.获取TracerPid，判断是否有调试器。反制措施：
	![](https://i.loli.net/2019/07/20/5d32d36511ecc12515.png)
	修改strtol为返回0，即
	

```
mov r0, #0 
mov pc, lr
nop
```
![](https://i.loli.net/2019/07/20/5d32d286a2d5783642.png)
3.获取tcp，判断23946端口是否被占据（即IDA）。反制措施就是改为其他端口，我寻思这是防小白？
4.fork结合raise假信号迫使IDA出错，解决方案就是raise直接返回，并且把调用fork的地方给nop了。
![](https://i.loli.net/2019/07/20/5d32d288b833a80297.png)
可见子进程继续运行，父进程死于非命。sub_CAAC9E00就是signal
![](https://i.loli.net/2019/07/20/5d32d2893377a50279.png)

```
ret = mov pc, lr
```
5._ZN3art3Dbg15gDebuggerActiveE，这个字符串是被加密的，不太好找，建议用010Editor全文件xor5A,Invert后定位，然后记录偏移量，在IDA中定位即可。如果你不是art，那么dalvik同理。
![](https://i.loli.net/2019/07/20/5d32d289d594497965.png)
我寻思这和2.1.1一毛一样的位置。
即便找到了，那么判断的函数还是不好定位，这里得解除以上几步之后，自己跟，发现会跳转到这里
![](https://i.loli.net/2019/07/20/5d32d28aac15141107.png)
注释是我自己加的，那么很简单，把函数跳转的地方nop就好了
![](https://i.loli.net/2019/07/20/5d32d28b9e1a575219.png)
6.检查cmdline，这里会检查一些莫名其妙的东西
![](https://i.loli.net/2019/07/20/5d32d28c0f84d90353.png)
判断是否有这些东西，我就没管
7.顺便把调用thread_create的地方给nop了，实际上我不知道是否创建线程进行反调了。
![](https://i.loli.net/2019/07/20/5d32d28c9c3cc97737.png)
8.除此之外，我还直接返回了exit，kill，time等函数，不知道效果。
把这些都改好之后，Apply Patched Bytes，然后把改过的libjiagu.so放到手机里，开始调试新的libjiagu.so。
![](https://i.loli.net/2019/07/20/5d32d291a9b3c46361.png)
在case 35也放置一个断点
在经过反调之后，libjiagu会解密，解压，释放新的elf，修复分段表，然后将程序移交新elf，进行jni native层的函数注册等操作。
不得不承认，这超出了我的能力，而且新的elf我也没找到好方法动调。
开始运行，一路F7 F9，在这里解压
![](https://i.loli.net/2019/07/20/5d32d2924a5ec76631.png)
这里dlopen之后进入释放的JNI_OnLoad
![](https://i.loli.net/2019/07/20/5d32d2930d29f87353.png)
这里就是释放的elf，经过分析是真的JNI_OnLoad，注册了native层函数，并且还进行了一些签名校验，如果想要破解应该从这里入手。因为和目标无关，暂且收手，开始寻求新的思路。
![](https://i.loli.net/2019/07/20/5d32d2940df0667346.png)
## Step 3 
用jadx看一下java层代码
![](https://i.loli.net/2019/07/20/5d32d294b84bf54373.png)
可见libjiagu在native层完成注册后，加载了libmono。
libmono经过分析也是自己编译的版本。
![](https://i.loli.net/2019/07/20/5d32d296d588713221.png)
这个libmono和libjiagu一样，只不过没有任何反调试手段，但是最后也会释放真正的libmono，并加载，以保证unity的正常运行。用同样手段，可以在uncompress函数处找到解压后的数据地址，并且dump出来，加以分析。然后可以再进行动态调试，在获取被加载的基地址之后，跳转过去，然后下断点即可。
要进行dump，我们需要加入一段代码等待调试器接入，反编译classes.dex，与2.1.1不同的是，2.3.0有3个dex文件，不过不影响，打开com.stub.stubApp的smali，在loadLibrary前添加waitForDebugger

```
:try_start_1 
invoke-static {}, Landroid/os/Debug;->waitForDebugger()V  
const-string v0, "mono"
invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
:try_end_1
```
压缩新的classes.dex到压缩包，复制到手机中进行安装，需要先使用Lucky Patcher解除签名验证和zip校验。
然后对libmono进行调试，注意设置Debugger Options
![](https://i.loli.net/2019/07/20/5d32d2976cd5d78948.png)
点开修改的Cytus2，应该处于黑屏状态，随后IDA attach，恢复运行，然后jdb attach。
![](https://i.loli.net/2019/07/20/5d32d2989c9b299404.png)
可见libmono已被载入
![](https://i.loli.net/2019/07/20/5d32d2991901612520.png)
定位到他的JNI_Onload
![](https://i.loli.net/2019/07/20/5d32d299c765633468.png)
没有反调试，一路跟到uncompress
![](https://i.loli.net/2019/07/20/5d32d29f406cc24247.png)
uncompress定义如下

```
uncompress(outbuf, &outbufsize, inbuf, filesize);
```
因此，记录一下outbuf的地址和大小即可
![](https://i.loli.net/2019/07/20/5d32d29fa7e3760636.png)
导出脚本

```
import idaapi
data = idaapi.dbg_read_memory(0xd4c00000, 0x486444)  
fp = open('H:\\Cytus\\mono_full_dump.bin', 'wb')
fp.write(data)
fp.close()
```
![](https://i.loli.net/2019/07/20/5d32d2a0044da91020.png)
前0x7098字节不知道是什么，把他先去掉，放进IDA进行分析。
我们需要一份正常的libmono作为比较，Cytus2是Unity2017.4.5f1，应该能随便找一份拿来比较，毕竟差异不大。
经过分析libunity.so得知，是通过调用mono_image_open_from_data_with_name来加载托管dll的。经过搜索"data-%p"可以很轻松的在dump中发现位置。
![](https://i.loli.net/2019/07/20/5d32d2a0a028b67209.png)
接下来的工作很枯燥，就是递归是地对比，把符号从正常的libmono中复制到dump中，通过比较发现有以下函数被修改：
![](https://i.loli.net/2019/07/20/5d32d2a11b06468558.png)
modified是我的标记，意思是和原函数有很大不同，不过最大的不同如前文所说，是整套il的更换
![](https://i.loli.net/2019/07/20/5d32d2a19a9d258953.png)
也是我自己做的标记。
既然函数找到了，我们就可以开始动调了。
找到偏移位置，在对应位置下断点，以本次为例，基地址D4A00000+7098
则函数mono_image_open_from_data_with_name位置D4BD8C68，跳转并下断点。

但是此时意外发生，2.3.0似乎增加了某种反调，原本能直接过到libunity加载，现在却不行了。于是对于2.3.0这条路也暂时行不通。
## Step 4
还有最后一个方法：在libunity加载前下断点，直接分析libunity，然后跳转到释放的libmono。
![](https://i.loli.net/2019/07/20/5d32d2a205ba659115.png)
同理，在此处加waitForDebugger，然后把classes3.dex替换了，运行程序。
此时成功进入捕获到libunity.so的加载，我们只需要在他调用mono_image_open_from_data_with_name的地方钓鱼即可。
但是还是被kill了，我决定一探究竟
![](https://i.loli.net/2019/07/20/5d32d2a350a5516668.png)
突然想起来好像java层还有一些反调，我好像忘记清了...
则成功过了
![](https://i.loli.net/2019/07/20/5d32d2a4b52e092139.png)
这个位置就是libunity调用libmono加载托管dll地方，位置可以搜字符串得到
![](https://i.loli.net/2019/07/20/5d32d2a8f103b75574.png)
F7进入
![](https://i.loli.net/2019/07/20/5d32d2ae48fac13867.png)
则基地址可以求得：
![](https://i.loli.net/2019/07/20/5d32d2aeba0d461999.png)
D5A51BD0 - 1D1BD0 = D588 0000‬
这样其他函数地址都可以求得了
我们知道Assembly-CSharp.dll的文件头是mz,不是MZ,观察R0，直到发现它
![](https://i.loli.net/2019/07/20/5d32d2af933f674473.png)
经过几十次的F7，终于找到了，期间如果遇到SIGPWR,SIGXCPU可以Pass，不必惊慌
![](https://i.loli.net/2019/07/20/5d32d2b02363060754.png)
那么我们对照着正常的libmono一步步看
![](https://i.loli.net/2019/07/20/5d32d2b10bc2c17904.png)
![](https://i.loli.net/2019/07/20/5d32d2b226f9c73703.png)
对照两图，基本上一目了然，跟入do_mono_image_load
![](https://i.loli.net/2019/07/20/5d32d2b2e63e345672.png)
既然Assembly-CSharp.dll的文件头被改为了mz，那必然有鬼。对dump出来的dll进行分析，符号表自然是不存在了，但是mono是开源的，于是可以对照着正常的libmono进行对比，看看哪里有鬼。比如搜一下mz这种东西，一定能看出端倪。
我的前期比较显示，一共有两个地方和正常的不一样。和dll也一样，一个是前面的分段表不一样，一个是中部的表流定义不一样
![](https://i.loli.net/2019/07/20/5d32d2b3b3ff557716.png)
那么我们一个一个来
![](https://i.loli.net/2019/07/20/5d32d2b4844c457344.png)
这里就完成了section table的解密工作，前面的109 122就是mz的判断。
![](https://i.loli.net/2019/07/20/5d32d2b508c4560927.png)
这是解密函数，可见是对一个字符串作循环异或，这点是静态分析看不出来的，于是我们可以做一个解密函数了。

```
        static void Main(string[] args)
        {
            byte[] bin = File.ReadAllBytes("./Assembly-CSharp.dll");
            DecryptSectionTable(bin);
            File.WriteAllBytes("./Temp.dll", bin);
        }
        static int DecryptSectionTableInternal(byte[] k, byte[] bin, int pos, byte length, byte p)
        {
            int kp = 0;
            for (int i = 0; i < length; ++i)
            {
                bin[pos + i] ^= k[kp++];
                if (kp == k[8]) kp = 0;
            }
            for (int i = 0; i < length; ++i)
            {
                byte newk = bin[pos + i];
                newk ^= p;
                k[i] = newk;
            }
            k[8] = length;
            return pos + length;
        }
        static void DecryptSectionTable(byte[] bin)
        {
            byte[] k = { 0x6d, 0x70, 0x7a, 0x65, 0x65, 0x7a, 0x70, 0x6d, 0x08 };
            int pos = 0x178;
            for (byte i = 0; i < 3; ++i)
            {
                pos = DecryptSectionTableInternal(k, bin, pos, 8, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 2, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 2, i);
                pos = DecryptSectionTableInternal(k, bin, pos, 4, i);
            }
        }
```
这样分段表便能解密成功。
剩下这个部分
![](https://i.loli.net/2019/07/20/5d32d2ba40fa729301.png)
地址可以通过计算RVA得到，由于该段落在.text里，则地址为
![](https://i.loli.net/2019/07/20/5d32d2baa358e10685.png)
ADDR = VirtualAddress - PointerToRawData + cor20.Metadata.VirtualAddress + MetadataHeader.pools[0].ioffset
不小心碰了下屏幕触发了ANR，在这里提醒一定不要碰屏幕，不然凉凉，得重新进
![](https://i.loli.net/2019/07/20/5d32d2bb239f199381.png)
这里判断是否需要解密，如果是正常的，那么这里是
![](https://i.loli.net/2019/07/20/5d32d2bb801c510938.png)
但是事实上这里被加密了，则
![](https://i.loli.net/2019/07/20/5d32d2bbdeb4466869.png)
因此开始研究一下解密
![](https://i.loli.net/2019/07/20/5d32d2bc6aa0f38684.png)
360便是在reserve里做文章。
初步看了一下，就是把maskvalid和sorted8字节按位异或即可，其他的操作没有深入研究。
至此，扔进dnSpy就可以读了。不过方法体都是空的，因此指令一定是被加密了。
![](https://i.loli.net/2019/07/20/5d32d2be57f0764242.png)
继续分析，il执行机制，可以发现il的编译就是在这个巨型函数里实现的
![](https://i.loli.net/2019/07/20/5d32d2c012fb255100.png)
不过360比较骚，请看
![](https://i.loli.net/2019/07/20/5d32d2c078d6883305.png)
函数名称是我做的标记，意思就是360自己把il转换了……这着实让人头大。
不过办法总是有的，既然是基于switchcase，那么我把这两个函数的反汇编进行了比较，还写了一个简陋的程序
![](https://i.loli.net/2019/07/20/5d32d2c14eb5258571.png)
就这么手工比较，~~**也就两个小时**~~ 不到对应关系就找出来了，那么我们开始转义为原il~
首先转义之前，il是被加密了
![](https://i.loli.net/2019/07/20/5d32d2c67954f65881.png)
密钥需要动调到这里进行获取，也是按位异或，位置20454C自己计算一下。
![](https://i.loli.net/2019/07/20/5d32d2c7346b255646.png)
是不是很熟悉？
完了之后还要整体异或0x30，所以还是挺复杂的
![](https://i.loli.net/2019/07/20/5d32d2c78ebf829974.png)
我们有了解密方法，转义方法，按照函数表的il指令位置一条条转换就可以了。
最后一步就是定位Method表，那么也很简单，甚至你手动定位一下就行。
![](https://i.loli.net/2019/07/20/5d32d2c8698e650583.png)
最后就是见证奇迹的时刻，运行！
![](https://i.loli.net/2019/07/20/5d32d2c8e20a622512.png)
![](https://i.loli.net/2019/07/20/5d32d2c94d12a92228.png)
放进dnSpy
![](https://i.loli.net/2019/07/20/5d32d2ca856bd87259.png)
所有指令还原成功！
## Step 5
注入环节就更简单了，写C#就完事了，由于2.1.1我已经写过了，2.3.0我就不重新写了，效果都是一样
请看视频

<iframe height=720 width= 100% src="//player.bilibili.com/player.html?aid=47793781&cid=83718840&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

这是2.1.1注入的截图
![](https://i.loli.net/2019/07/20/5d32d2cbc252594956.png)
只要你把文件头改回MZ，那么具体执行起来就会使用原来的il转换，意思就是替换就完事了。
但是具体执行之后我发现有时候会出现invaild il code和堆栈问题，不过使用dnSpy重新编译一下就好了，不知道为什么。
## 写在最后
2.1.1用了我10天，即便是重新2.3.0，也花了我1天时间，虽然我认为这个加固已经可以了，但是山外有山，学习不能停啊
向所有安全研究人员致敬！
