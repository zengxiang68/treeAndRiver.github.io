---
layout:     post
title:      分析内核UAF-commit
subtitle:   107821669a9cbf234f260d576039983b64c7cb6d
date:       2020-11-23
author:     ZX
header-img: img/post-bg-unix-linux.png
catalog: true
tags:
    - UAF
    - kernel commit
    - vulnerability
---



# 分析UAF-commit

## commit——107821669a9cbf234f260d576039983b64c7cb6d[链接](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/diff/drivers/gpu/drm/i915/display/intel_global_state.c?id=107821669a9cbf234f260d576039983b64c7cb6d)

### 描述

1. 这次commit把原来直接的`atomic destory`操作换成了refcount的`put`操作，可能是为了避免并发执行`intel_atomic_get_global_obj_state`操作或`intel_atomic_swap_global_state`操作时，将`obj->state`销毁后又在这两个函数里面使用，导致UAF。

   问题1：能否在一个进程中`destory`后直接执行`intel_atomic_get_global_obj_state`操作或`intel_atomic_swap_global_state`操作？可以这样的话，根本不需要并发，也能触发UAF；答：有可能。

   问题2：只有这两个函数可能在`obj->state`被销毁后还可能再用到吗？答：不一定。需要去找这几个函数是否已经是顶层函数，在用户态通过syscall可以操控的，如果不是的话，那么应该存在别的代码来调用这些函数，那就要分析它的逻辑了。

2. 通过搜索内核源码可以发现，

   ![image-20200908095603485](https://i.loli.net/2020/09/08/WKenF4ZxfbCMkNi.png)

   ![image-20200908095718897](https://i.loli.net/2020/09/08/PhMU8xQX6ryOk9o.png)
   这两个包含destory操作的函数在内核源码里面都只出现了3次：

   * 1次头文件中函数声明（函数原型）
   * 1次函数定义（函数实现）
   * 1次被调用

   这么来看的话，它们还可能是顶层函数吗？（可以通过syscall调用）怎么确认呢？
   **可泛化为一个问题：如何确定某个内核函数是否是顶层函数？**（除了有些函数会赋值到`file operations`结构体而成为顶层函数，还有呢？）
   **想法：**做一个网状图，追踪代码，从syscall开始，找到syscall可能调用哪些函数，一步步追踪下去，画成图。这样就能知道，哪些函数是可以通过syscall直接操控的，而哪些函数是只能由syscall操纵的函数来操纵的，用户态无法任意调用。
   
   
   
   **UAF本质**  应该是：free之后没有置空（Null），因为如果置空了，另一个函数要用它的时候只需检查一下是否为Null就不会出错。在free之后，只是将malloc申请的内存空间释放，该地址很可能马上就被分配给了其他变量，地址里面放的东西就变了，这时候再来用这个变量（从这个地址取值），就会出现无法意料的错误。**Double Free为什么会有问题**：当程序使用同一参数两次调用 `free()` 时，程序中的内存管理数据结构会遭到破坏（与glibc有关）。
   
   编译代码如果代码（函数）里面直接存在doubel free的逻辑，编译器就会报错。
   
   另外补充: 根据free的设置定义，free(null)是没有安全问题的。

### 疑问

1. 下面这个函数里面，obj->state 是个局部变量，在栈上，理应由编译器来管理分配与释放，为什么需要用（引用计数）put 呢？

   ```C
   void intel_atomic_global_obj_cleanup(struct drm_i915_private *dev_priv)
   {
   	struct intel_global_obj *obj, *next;
   
   	list_for_each_entry_safe(obj, next, &dev_priv->global_obj_list, head) {
   		list_del(&obj->head);
   
   		drm_WARN_ON(&dev_priv->drm, kref_read(&obj->state->ref) != 1);
   		intel_atomic_global_state_put(obj->state);
   	}
   }
   ```

   答：这是个指针，指针指向的是一个堆上的内存地址，obj是从列表上取下来的。

2. 下面这个函数里面的 atomic_destroy_state 函数定义到底在哪里？（追踪代码找不到啊）

   ```C
   void intel_atomic_global_obj_cleanup(struct drm_i915_private *dev_priv)
   {
   	struct intel_global_obj *obj, *next;
   
   	list_for_each_entry_safe(obj, next, &dev_priv->global_obj_list, head) {
   		list_del(&obj->head);
   		obj->funcs->atomic_destroy_state(obj, obj->state);
   	}
   }
   ```

   答：肯定存在一个赋值，要么obj->funcs->atomic_destroy_state被赋值，要么obj->funcs被赋值，所以可以去搜`obj->funcs->atomic_destroy_state =` 和`obj->funcs =` ，结果可以发现是obj->funcs被赋值了，再继续追：
   
   <img src="https://i.loli.net/2020/10/14/3517NK2PiMwFIly.png" alt="image-20201014200442930" />

如图，funcs来自函数参数，去找intel_atomic_global_obj_init的reference：

![image-20201014200640280](https://raw.githubusercontent.com/treeAndRiver/treeAndRiver.github.io/master/img/rkqO53bLlRdPoTX.png)

继续：

![image-20201014200708294](https://i.loli.net/2020/10/14/f3EPeQNa8hnkuJq.png)

![image-20201014200733571](https://i.loli.net/2020/10/14/GlFYNcnK1u3g8Zh.png)