---
title:  "DRM（Direct Rendering Manager）学习简介"
date:   2018-10-25 23:20:00
categories: text
---

学习DRM一年多了，由于该架构较为复杂，代码量较多，且国内参考文献较少，初学者学习起来较为困难。因此决定将自己学习的经验总结分享给大家，希望对正在学习DRM的同学有所帮助，同时交流经验。

由于本人工作中只负责Display驱动，因此分享的DRM学习经验都只局限于Display这一块，对于GPU这一块本人无能为力，如果大家有相关经验分享，还请告知一声，大家相互学习。


## DRM

DRM是Linux目前主流的图形显示框架，相比FB架构，DRM更能适应当前日益更新的显示硬件。比如FB原生不支持多层合成，不支持VSYNC，不支持DMA-BUF，不支持异步更新，不支持fence机制等等，而这些功能DRM原生都支持。同时DRM可以统一管理GPU和Display驱动，使得软件架构更为统一，方便管理和维护。

DRM从模块上划分，可以简单分为3部分：**`libdrm`**、**`KMS`**、**`GEM`**

![DRM_architecture.svg-44.9kB][1]
（图片来自Wiki）

### libdrm
对底层接口进行封装，向上层提供通用的API接口，主要是对各种IOCTL接口进行封装。

### KMS
Kernel Mode Setting，所谓Mode setting，其实说白了就两件事：`更新画面`和`设置显示参数`。
**更新画面：**显示buffer的切换，多图层的合成方式，以及每个图层的显示位置。
**设置显示参数：**包括分辨率、刷新率、电源状态（休眠唤醒）等。

### GEM
Graphic Execution Manager，主要负责显示buffer的分配和释放，也是GPU唯一用到DRM的地方。



## 基本元素

DRM框架涉及到的元素很多，大致如下：
KMS：**`CRTC`**，**`ENCODER`**，**`CONNECTOR`**，**`PLANE`**，**`FB`**，**`VBLANK`**，**`property`**
GEM：**`DUMB`**、**`PRIME`**、**`fence`**

![Screenshot from 2018-11-04 17:02:56.png-34.9kB][2]
（图片来源：[The DRM/KMS subsystem from a newbie's point of view][3]）

| 元素 | 说明|
|--- |--- |
| CRTC | 对显示buffer进行扫描，并产生时序信号的硬件模块，通常指Display Controller |
| ENCODER | 负责将CRTC输出的timing时序转换成外部设备所需要的信号的模块，如HDMI转换器或DSI Controller |
| CONNECTOR | 连接物理显示设备的连接器，如HDMI、DisplayPort、DSI总线，通常和Encoder驱动绑定在一起 |
| PLANE | 硬件图层，有的Display硬件支持多层合成显示，但所有的Display Controller至少要有1个plane |
| FB | Framebuffer，单个图层的显示内容，唯一一个和硬件无关的基本元素 |
| VBLANK | 软件和硬件的同步机制，RGB时序中的垂直消影区，软件通常使用硬件VSYNC来实现 |
| property | 任何你想设置的参数，都可以做成property，是DRM驱动中最灵活、最方便的Mode setting机制 |
| | |
| DUMB | 只支持连续物理内存，基于kernel中通用CMA API实现，多用于小分辨率简单场景 |
| PRIME | 连续、非连续物理内存都支持，基于DMA-BUF机制，可以实现buffer共享，多用于大内存复杂场景 |
| fence | buffer同步机制，基于内核dma_fence机制实现，用于防止显示内容出现异步问题 |

学习DRM驱动其实就是学习上面各个元素的实现及用法，如果你能掌握这些知识点，那么在编写DRM驱动的时候就能游刃有余。

## 目录（正在更新中）
本篇博客将作为本人DRM学习教程的目录汇总，后续我会以示例代码的形式和大家分享上述知识点的学习过程，并不断更新目录链接，敬请期待！

## 参考资料
1. Wiki: [Direct Rendering Manager][4]
2. wowotech: [Linux graphic subsystem(2)_DRI介绍][5]
3. Boris Brezillon: [The DRM/KMS subsystem from a newbie's point of view][3]
4. 线·飘零 博客园：[Linux环境下的图形系统和AMD R600显卡编程(1)][7]
4. Younix脏羊 CSDN博客：[Linux DRM（二）基本概念和特性][6]


  [1]: http://static.zybuluo.com/hexiaolong2008/7z1ub87kcam4zobz93shg56y/DRM_architecture.svg
  [2]: http://static.zybuluo.com/hexiaolong2008/b5ipgkkxjw3szofaq81grjt4/Screenshot%20from%202018-11-04%2017:02:56.png
  [3]: https://events.static.linuxfound.org/sites/events/files/slides/brezillon-drm-kms.pdf
  [4]: https://en.wikipedia.org/wiki/Direct_Rendering_Manager
  [5]: http://www.wowotech.net/linux_kenrel/dri_overview.html
  [6]: https://blog.csdn.net/dearsq/article/details/78394388
  [7]: https://www.cnblogs.com/shoemaker/p/linux_graphics01.html