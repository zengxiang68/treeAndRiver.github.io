---
layout:     post
title:      LLVM和Clang编译Linux内核(得到bc文件)
subtitle:   
date:       2020-11-23
author:     ZX
header-img: img/post-bg-hacker.png
catalog: true
tags:
    - Tools
    - LLVM
    - Clang
	- Kernel
---



# LLVM/Clang编译Linux内核（得到bc文件）

## 安装

因为我已经用apt安装了6.0版本，系统默认的Clang只能有一个，所以不再使用apt安装，直接去官网下载编译好的安装包即可。

如果想从源码编译，参考其[github主页](https://github.com/llvm/llvm-project)及[博文1](https://my.oschina.net/u/943779/blog/3025984)、[博文2](https://blog.csdn.net/weixin_42826139/article/details/88382253)即可。（虚拟机2GB内存不够，会导致错误）

## 编译内核5.0

### 1、修改部分文件

暂跳过

## 2、编译

#### 第一步：生成bc文件

1. 使用`make CC=clang ARCH=x86_64 defconfig` 生成`.config`文件（要加 CC=clang，这里使用默认的6.0版本，最好也加上ARCH）
2. `make CC=clang -j4`即可生成`bzImage`。

​		或者使用那个[sh文件](https://github.com/ChengyuSong/lll-414/blob/master/clang-emit-bc.sh)才可生成llbc文件: `make CC=./clang-emit-bc.sh ARCH=x86_64 -j4`

#### 第二步：进行Link

1. 继续使用6.0版本进行link会报错，换成10.0.1版本的llvm-link也是同样的错误：

   ```bash
   error: linking module flags 'wchar_size': IDs have conflicting values in './a20.llbc' and 'llvm-link'
   ```

   换成3.9版本的llvm-link又报错版本不兼容（应该是只能向后兼容）。

2. 现在换成LLVM/Clang3.9来编译，对应的使用llvm-link3.9来link，结果报错：

   ```bash
   ERROR: Linking globals named '__this_module': symbol multiply defined
   ```

   用更高版本的llvm-link报一样的错。

3. 试了`KAMain -dump-caller-graph ./*`（尝试将2000+llbc文件一次性输入）报错，无法一次性输入这么多文件。看来必须要link。

4. 按学长给的代码剔除了部分llbc文件，还是报同类型的新错误。

   ```python
   def readfiles(path):
       fileList = []
       List = os.walk(path)
       for tup in List:
           if tup[2] != [] and './arch/x86/boot' not in tup[0] and './arch/x86/realmode/rm' not in tup[0] \
           and 'netfilter' not in tup[0] and 'efivarfs' not in tup[0] \
           and 'rtl81' not in tup[0] and 'vt6655' not in tup[0] and 'rtlwifi' not in tup[0]:
               for f in tup[2]:
                #   if 'vclock_gettime' in f or 'nf_' in f:
                 #      continue
                   if 'fdomain_core' in f:
                       continue
                   if 'odm_DIG' in f:
                       continue
                   if 'r8192U_core' in f:
                       continue
                   if 'rtl81' in f:
                       continue
                   if 'halinit' in f:
                       continue
                   if 'phy.llbc' in f:
                       continue
                   if 'odm.llbc' in f:
                       continue
                   if 'odm_HW' in f:
                       continue
                   if 'hal_intf' in f:
                       continue
                   if 'hal_com' in f:
                       continue
                   if 'rtw_mlme' in f:
                       continue
                   if 'rtw_ap' in f:
                       continue
                   if 'rtw_debug' in f:
                       continue
                   if 'rtw_ieee80211' in f:
                       continue
                   if 'rtw_r' in f:
                       continue
                   if 'rtw_pwr' in f:
                       continue
                   if 'rtw_efuse'in f:
                       continue
                   if 'rtw_sta_mgt' in f:
                       continue
                   if 'rtw_ioctl_set' in f:
                       continue
                   if 'vsprintf' in f:
                       continue
                   if 'vclock_gettime' in f:
                       continue
                   parse = f.split('.')
                   if len(parse) == 2 and parse[1] == 'llbc':
                       # f = parse[0] + '.llbc'
                       if 'string.llbc' in f:
                           continue
                       fileList.append(tup[0] + '/' + f)
       print len(fileList)
       raw_input()
       return fileList
   ```

   新错误：

   ```bash
   error: Linking globals named 'init_module': symbol multiply defined!
   ```

### 问题总结：

### 1、无法成功link（即使是5.0版本LLVM去编译5.0内核）

答：确实是无法Link的。

### 2、`KAMain -dump-caller-graph ./*`，无法一次性输入所有2000+llbc文件（5.0内核）

**发现了报错原因：**依然是参数太长的问题（将输出写入文件再用编辑器打开，会更明显发现最后一行非常长）解决方法：不再使用 `/*.llbc` 来传递文件，结合`find`和`-exec`命令可解决问题。

### 3、针对之前的漏洞，需要的是5.7以上内核，需要修改某些文件。

### 4、如果我直接把内核源码中的.c文件一个个用-emit-llvm编译成bc文件，是否可以？有什么不同？

答：本质上没有不同，只是内核源码中的.c文件恐怕难以一个个用-emit-llvm编译成bc文件，会报错找不到头文件之类的错误，这就正是make defconfig做的事。



## 再次尝试

用llvm/clang10.0编译Linux内核5.7，结果2268个llbc文件，KAMain有1861个无法处理（无法加载）。

### 学长建议

- 使用make allyesconfig ，不指定架构，会有10000+llbc文件。

* 编译KA的版本最好和编内核的版本一致

## 继续

先拿6.0来编译内核吧。——走不通，5.7内核的asm要求clang版本 >=8：

![image-20201002101846328](https://i.loli.net/2020/10/02/D8xNpOyzcPIgQlT.png)

用clang/llvm10.0来编译还是报版本不合理，但只是警告，不是错误：

![image-20201002101743089](https://i.loli.net/2020/10/02/5XyNsLpFcGS3MAk.png)

换成clang/llvm8.0不仅报和10.0一样的警告，还报asm-goto错误：

![image-20201002111702438](https://i.loli.net/2020/10/02/rpin4XYVTQajZJw.png)

看来只能用10.0来编译，才不会报asm-goto错误。

（使用make allyesconfig）运行了4.5小时，结果报错：

![image-20201002161321878](https://i.loli.net/2020/10/02/L9m7ChUlJq6sDeA.png)

那就尝试10.0编译内核5.8。

| Experiment num | Kernel | LLVM/Clang |         Results         |
| :------------: | :----: | :--------: | :---------------------: |
|       1        |  5.7   |    6.0     |     asm-goto error      |
|       2        |  5.7   |     8      |     asm-goto error      |
|       3        |  5.7   |    10.0    | target 'drivers' failed |
|       4        |  5.8   |     8      |     asm-goto error      |
|       5        |  5.8   |    10.0    | target 'drivers' failed |

 Clang/LLVM < 9.0.0不支持`asm goto`，所以一定会报asm-goto错误。

我觉得关键还是要解决`target 'drivers' failed`错误，即使解决了`asm-goto`错误，后面很可能还是会遇到`target 'drivers' failed`。

原来之前编译kernel-analyzer用的是llvm/clang6.0，竟然不是用的默认的10.0，10.0无法编译KA，而修改KA目前来说太难，那就还是看能否解决asm-goto吧。

宋老师有对asm-goto问题修改代码，去看他怎么修改的。

#### 补充：如何用LLVM/Clang6.0编译KA？

1. 在Makefile里面修改Clang版本。（无法修改LLVM因为这里要提供LLVM build目录，而我的LLVM6.0是用apt装的）
2. 在生成的build文件夹里有个CMakeCache.txt，它相当于cmake的Makefile，在它里面修改LLVM版本。
3. 运行`cmake ./`和`make`。

### 继续

对照宋老师那个repo修改了4个文件，没有在一开始就报asm-goto错误了。新错误：

```bash
arch/x86/kvm/vmx/ops.h error: 'asm goto' constructs are not supported yet
```

解决办法：[修改asm_volatile_goto](https://github.com/iovisor/bcc/issues/2119)。

再编译，新错误：

```bash
arch/x86/kernel/signal.c:461:2: error: incompatible pointer types initializing 'typeof (*(&frame->pretcode))' (aka 'char *') with an expression of type '__sigrestore_t' (aka 'void (*)(void)') [-Werror,-Wincompatible-pointer-types]
        unsafe_put_user(ksig->ka.sa.sa_restorer, &frame->pretcode, Efault);
```

即使有新错误，也已经编译了约9000个llbc文件。

先把这9000个用KA分析看看

上面错误的源码：

```c
unsafe_put_user(ksig->ka.sa.sa_restorer, &frame->pretcode, Efault);
```

修改为：

```C
unsafe_put_user(ksig->ka.sa.sa_restorer, (unsigned long __user *)&frame->pretcode, Efault);
```

不管对不对，重新编译后此错误消失了。

## Kernel-analyzer

#### 问题1

kernel-analyzer的命令`-dump-call-graph` 和 `-dump-caller-graph`有什么不同啊？仅使用后者就可以从输出中找到”函数调用对“了啊
而且使用`-dump-call-graph`输出不太对，截图：

![image-20201008101613560](https://i.loli.net/2020/10/08/oelAQSJajwgX9nV.png)

#### 答：根据KA源码，使用`-dump-call-graph`输出就是这样，没有不对。`-dump-call-graph` 和 `-dump-caller-graph`前者输出应该更详细。

#### 问题2

学长写的`link.py`有几个地方我不太懂

![image-20201008102017211](https://i.loli.net/2020/10/08/Xmn1g5QxHGuKoPZ.png)

#### 答：这几个命令参数是学长自己改了KA源码。

## 回到UAF-commit分析

之前那个commit相关的文件已经编译并用KA分析出来了。