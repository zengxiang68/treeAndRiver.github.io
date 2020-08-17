---
layout:     post
title:      About syzkaller
subtitle:   syzkaller安装与原理
date:       2020-08-17
author:     ZX
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Tools
    - syzkaller
    - fuzz
---



# syzkaller

## 一、安装与运行

同时参考[教程1](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md)和[教程2](https://snappyjack.github.io/articles/2020-05/%E4%BD%BF%E7%94%A8Syzkaller%E8%BF%9B%E8%A1%8C%E5%86%85%E6%A0%B8fuzz)即可，过程中遇到错误则根据具体情况满足错误提示所需即可。

**注意：**

1. 为了让启用`kvm`不用`root`权限，需要添加用户到`kvm`组，添加后需要注销重新登陆才会生效。
2. 真正运行`syzkaller`时，无需手动保持虚拟机开启以及和虚拟机的`ssh`连接。

可在浏览器中查看运行情况：

<img src="https://i.loli.net/2020/08/03/aZ6MNe2CXpmubwL.png" alt="2020-08-02 22-45-56屏幕截图" style="zoom:80%;" />

​																											图1

<img src="https://i.loli.net/2020/08/03/Ppnj2t5hw9bZfm4.png" alt="2020-08-02 22-46-49屏幕截图" style="zoom: 80%;" />

​																											图2

## 二、解读syzkaller log

### 1、[实验设置](https://snappyjack.github.io/articles/2020-05/%E4%BD%BF%E7%94%A8Syzkaller%E8%BF%9B%E8%A1%8C%E5%86%85%E6%A0%B8fuzz)

> <img src="https://i.loli.net/2020/08/12/yRf98GI6nuormNp.png" alt="批注 2020-08-12 182957" style="zoom:80%;" />

> 使得当连续两次`chmod`调用的`mode`参数值为0时会产生空指针解引用异常。

### 2、log见输出文件

### 3、下一步

修改实验，不指定只测试`chmod`相关的系统调用，让syzkaller自行发现，再来理解输出的log文件。



## 三、syzkaller相关论文

 [网页链接](https://github.com/google/syzkaller/blob/master/docs/research.md)

> * [Agamotto: Accelerating Kernel Driver Fuzzing with Lightweight Virtual Machine Checkpoints](https://www.usenix.org/conference/usenixsecurity20/presentation/song) ([source code](https://github.com/securesystemslab/agamotto))  //描述syzkaller不多
> * [Task selection and seed selection for Syzkaller using reinforcement learning](https://groups.google.com/d/msg/syzkaller/eKPD4ZpJ66o/UqO_K-SMFwAJ) (announce only)  //无论文
> * [Empirical Notes on the Interaction Between Continuous Kernel Fuzzing and Development](http://users.utu.fi/kakrind/publications/19/vulnfuzz_camera.pdf)  //描述syzkaller不多
> * [FastSyzkaller: Improving Fuzz Efficiency for Linux Kernel Fuzzing](https://iopscience.iop.org/article/10.1088/1742-6596/1176/2/022013)  //**可选**
> * [Charm: Facilitating Dynamic Analysis of Device Drivers of Mobile Systems](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-talebi.pdf)  //描述syzkaller不多
>   ([video](https://www.usenix.org/conference/usenixsecurity18/presentation/talebi),
>   [slides](https://www.usenix.org/sites/default/files/conference/protected-files/security18_slides_talebi.pdf))
> * [ALEXKIDD-FUZZER: Kernel Fuzzing Guided by Symbolic Information](https://www.cerias.purdue.edu/assets/symposium/2018-posters/829-D1B.pdf)  //无论文
> * [DIFUZE: Interface Aware Fuzzing for Kernel Drivers](https://acmccs.github.io/papers/p2123-corinaA.pdf)  //描述syzkaller不多         **值得看**
> * [MoonShine: Optimizing OS Fuzzer Seed Selection with Trace Distillation](http://www.cs.columbia.edu/~suman/docs/moonshine.pdf)  //描述syzkaller不多
> * [RAZZER: Finding Kernel Race Bugs through Fuzzing](https://lifeasageek.github.io/papers/jeong:razzer.pdf)  //有部分syzkaller描述
> * [SemFuzz: Semantics-based Automatic Generation of Proof-of-Concept Exploits](https://www.informatics.indiana.edu/xw7/papers/p2139-you.pdf)
> * [Towards Automating Exploit Generation for Arbitrary Types of Kernel Vulnerabilities](https://i.blackhat.com/us-18/Thu-August-9/us-18-Wu-Towards-Automating-Exploit-Generation-For-Arbitrary-Types-of-Kernel-Vulnerabilities-wp.pdf)
> * [KOOBE: Towards Facilitating Exploit Generation of Kernel Out-Of-Bounds Write Vulnerabilities](https://www.usenix.org/system/files/sec20summer_chen-weiteng_prepub.pdf)
> * [Synthesis of Linux Kernel Fuzzing Tools Based on Syscall](http://dpi-proceedings.com/index.php/dtcse/article/download/14990/14503)  //有部分syzkaller描述
> * [Drill the Apple Core: Up & Down](https://i.blackhat.com/eu-18/Wed-Dec-5/eu-18-Juwei_Lin-Drill-The-Apple-Core.pdf)
> * [WSL Reloaded](https://www.slideshare.net/AnthonyLAOUHINETSUEI/wsl-reloaded)



## 四、关于syzkaller和fuzz的原理知识

### syzkaller原理

> ![process_structure](https://i.loli.net/2020/08/15/TlExKLQNq3sOZbi.png)
>
> [图源链接](https://github.com/google/syzkaller/blob/master/docs/internals.md)

注：KCOV 是一个Linux内核模块，用来收集代码覆盖信息。

### kernel sanitizers

#### KASAN

(KernelAddressSanitizer)

> <img src="https://i.loli.net/2020/08/17/qOsKeMzr76VH1Zk.png" alt="批注 2020-08-17 090033" style="zoom: 67%;" />

#### KMSAN

(KernelMemorySanitizer)

> <img src="https://i.loli.net/2020/08/17/STKirgUYu9NQfvD.png" alt="批注 2020-08-17 091132" style="zoom:67%;" />

#### KTSAN

(KernelThreadSanitizer)

> <img src="https://i.loli.net/2020/08/17/rVLs7cCzDFgIloX.png" alt="批注 2020-08-17 092254" style="zoom:67%;" />

### 关于测试用例

#### 1、模糊测试的测试用例生成方法有两种

* 基于变异——根据已知数据样本通过变异的方法生成新的测试用例；
* 基于生成——根据已知的协议或接口规范进行建模，生成测试用例；

syzkaller同时使用覆盖率和模板来指导测试，既是基于变异也是基于生成。

#### 2、syzkaller测试用例的形式与结构

* syzkaller使用声明式描述的系统调用作为测试用例。
* 一列系统调用组成一个 prog 结构，作为一个测试用例。
* 所有prog 的集合，就是测试用例语料库（corpus）。

##### syscall顺序的决定

* syz-fuzzer会维护一个优先级表
* 优先级由静态和动态算法来计算，静态算法基于参数类型的分析，动态算法基于syscall在语料库中出现的频繁情况。
* syz-fuzzer根据优先级表和配置中启用的syscall列表，建立一个syscall的选择表（ChoiceTable）。

#### 3、syzkaller测试用例的变异策略

##### 变异操作（更具体见[代码](https://github.com/google/syzkaller/blob/ed8812ac86c117831a001923d3048b0acd04ed3e/prog/mutation.go)）

* splice——从corpus以外找一个随机的prog，拼接到当前prog的随机下标之后。
* insertCall——随机选择一个位置插入新的syscall。
* removeCall——移除syscall。
* mutateArg——随机找一个syscall，更改其参数。

不同操作的发生概率是不同的。

##### prog minimization

找到prog最小化的等价prog。