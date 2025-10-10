---
title: "Droidian适配以及跟进lineage 23.0时的问题"
description:
date: 2025-10-08T16:14:23+08:00
image: 
math: 
license: 
categories:
  - Tech
tags:
  - 技术总结
  - 内核配置
  - LineageOS
  - Droidian
  - 设备适配
  - 调试
  - bootloop
  - pstore
hidden: false
comments: false
isCJKLanguage: true
draft: false
---

## 起因

最近发现上游lineage的sony sm8250 平台内核源码准备更新到23.0了 (即Android 16, 当前版本lineage-22.2的下一个版本)。于是，这边droidian的设备内核也准备跟进了。结果，合并代码之后，发现上游改用了defconfig fragments。也就是说，不再使用之前单一的pdx206_defconfig一个文件进行配置了。最初，我尝试沿用旧的单一配置文件方式，手动合并了多个fragment文件，但这引发了一系列编译问题。比如不生成dtb，缺bpf的函数定义什么的。后面排查发现，是我手动拼的配置有冲突。比如，某段配置里出现了`# CONFIG_MACH_SONY_PDX206 is not set`, 而定义这个配置的vendor/pdx206.config却在前面。没想到，这样却把实际要用到的`CONFIG_MACH_SONY_PDX206=y`给屏蔽了。(和lindroid那边遇到的SYSVIPC配置被屏蔽的问题一样。[https://t.me/linux_on_droid/1263](https://t.me/linux_on_droid/1263))因为这些原因，我不得不彻底地重写配置，重新阅读droidian linux-packaging-snippets的配置脚本。鉴于这些细节在官方文档中可能未被明确提及，我决定将此次排查经验系统性地整理成文。

## Kernel Config的拼接逻辑

这部分的逻辑写在`/usr/share/linux-packaging-snippets/kernel-snippet.mk`里。可以在官方 docker 构建镜像(`quay.io/droidian/build-essential`)里查看。最终生成的 Kernel Config 是由`debian/kernel-info.mk`中配置的多段配置文件拼接而成的。**配置文件的顺序决定了优先级，后加载的片段可能覆盖前者的设置**。**配置的最终值会被最后一次对其赋值的片段决定**。例如，如果片段A设置了 `CONFIG_X=y`，而片段B设置了 `# CONFIG_X is not set`，则最终 `CONFIG_X` 不会被启用。

主要会用到的几个config配置参数是（按实际拼接顺序排序）：

- `KERNEL_DEFCONFIG`: 配置使用的默认defconfig. 实际会配置使用`$(KERNEL_SOURCES)/arch/$(KERNEL_ARCH)/configs/$(KERNEL_DEFCONFIG)`
- `KERNEL_CONFIG_COMMON_FRAGMENTS`: Droidian内核通常用到的内核配置片段。一般不需要配置，会默认添加`$(KERNEL_SOURCES)/droidian/common_fragments/halium.config`和`$(KERNEL_SOURCES)/droidian/common_fragments/droidian.config`.
- `KERNEL_CONFIG_DEVICE_FRAGMENTS`: 设备相关的配置片段。会按下列配置按顺序包含文件进去。
  - DEVICE_PLATFORM: 实际会使用`$(KERNEL_SOURCES)/droidian/$(DEVICE_PlATFORM).config`
  - DEVICE_MODEL: 实际会使用`$(KERNEL_SOURCES)/droidian/$(DEVICE_MODEL).config`
  - `KERNEL_CONFIG_EXTRA_FRAGMENTS`: 其他需要添加的片段配置文件，需要放到`$(KERNEL_SOURCES)/droidian/`下。支持多个配置文件。

## 如何找到本设备使用的内核配置文件

这里因为当前的上游是LineageOS，所以会按照LineageOS的配置方式进行讲解。其他Android系统上游则仅供参考。

通常你会需要看两个项目：`android_device_{VENDOR}_{SOC}-common`和`android_device_{VENDOR}_{MODEL}`

对于`Sony Xperia 5 II`来说，它的`VENDOR`是`Sony`, 设备代号是`pdx206`. 所以，首先需要看的项目是`android_device_sony_pdx206`。然后，从`BoardConfig.mk`中找`TARGET_KERNEL_CONFIG`即可拿到设备的内核配置文件。

```makefile
TARGET_KERNEL_CONFIG += vendor/pdx206.config
```

但是，通常它还要搭配额外的SOC平台内核配置文件一起使用。这就需要到对应的`android_device_{VENDOR}_{SOC}-common`项目中去找。通常你可以在设备项目中的`lineage.dependencies`找到。例如，https://github.com/LineageOS/android_device_sony_pdx206/blob/lineage-23.0/lineage.dependencies 

```json
[
  {
    "repository": "android_device_sony_sm8250-common",
    "target_path": "device/sony/sm8250-common"
  }
]
```

可以看到对应的SOC平台项目是`android_device_sony_sm8250-common`.接下来，就需要去这个项目里找SOC内核配置文件。

SOC平台项目的配置文件是`BoardConfigCommon.mk`。还是一样的，在其中查找`TARGET_KERNEL_CONFIG`，即可找到剩下的配置文件。

```makefile
TARGET_KERNEL_CONFIG := \
    vendor/kona-perf_defconfig \
    vendor/edo.config
```

## 额外内容：如何调试bootloop问题

这段是因为之前使用了错误的kernel config, 导致启动失败发生bootloop的情况。以前对此情况是真的束手无策。不过，现在偶然间查到了正确的调试方法，再也不是抓瞎的状态了。

当时的问题是这样的，我刷入了合入上游更新的内核。当我重启之后，它bootloop了几下之后，便正常进入系统。`uname`看内核信息，还是旧的内核版本。查看journalctl日志，没有bootloop时的信息。都是正常启动之后的日志。为此，我意识到，我必须要进行设备启动阶段的调试和日志收集了。我后面便查到了以下两种方法：

### ADB on boot

一种方式是开机默认允许所有adb调试链接。具体参考 https://johannes.truschnigg.info/writing/2022-05_android_bootloop_debugging/。但因为它有两个问题，导致我没用上：

1. 需要修改system分区的build.prop配置。这个因为在droidian上是loopback只读挂载，正常进系统后无法修改。在recovery模式下，system目录看不到这个文件。所以，无从使用。
2. 假如问题发生在kernel bootup阶段，那么还没等进系统就已经崩溃了，根本无从启动adbd。而在我这的情况，后来看来bootloop的原因是发生了kernel panic，所以，不适用这种方法进行调试。

### pstore

另一种方法是使用pstore将kernel bootup阶段的崩溃日志保留在内存中，下一次启动的时候便可以在`/sys/fs/pstore`中找到kernel panic信息以及崩溃时的内核日志。具体参考 https://docs.kernel.org/admin-guide/pstore-blk.html 

我使用的时候，忘记是没看清文档，正常启动的时候没有挂载pstore到`/sys/fs/pstore`上去还是正常启动时，可能由于系统设计会清空pstore区域，或者默认没有在 `/sys/fs/pstore` 中保留上一次启动的日志，导致这个路径里是空的。随后，便看到有人说recovery模式下可以看到pstore。便尝试了一下bootloop之后强制关机，直接接入fastboot模式重启进recovery。果然看到东西了。终于让我第一次看到了启动时kernel panic的原因了：

```bash
[    1.341874] qpnp-pdphy c440000.qcom,spmi:qcom,pm8150b@2:qcom,usb-pdphy@1700: Linked as a consumer to regulator.10
[    1.342419] list_add corruption. next->prev should be prev (ffffff9528ef5210), but was 0000000000000000. (next=fffffff44eb62710).
[    1.342487] ------------[ cut here ]------------
[    1.342521] kernel BUG at /build/sources/lib/list_debug.c:29!
[    1.342583] Internal error: Oops - BUG: 0 [#1] PREEMPT SMP
[    1.342617] Modules linked in:
[    1.342651] Process kworker/6:0 (pid: 70, stack limit = 0x00000000307301e5)
[    1.342713] CPU: 6 PID: 70 Comm: kworker/6:0 Tainted: G S                4.19-325-sony-pdx206 #1
[    1.342773] Hardware name: Sony Mobile Communications. PDX-206(KONA) (DT)
[    1.342813] Workqueue: events deferred_probe_work_func
[    1.342873] pstate: 60800085 (nZCv daIf -PAN +UAO)
[    1.342910] pc : __list_add_valid+0x94/0xb0
[    1.342971] lr : __list_add_valid+0x94/0xb0
[    1.343003] sp : ffffff8010403a10
[    1.343035] x29: ffffff8010403a10 x28: 0000000000000402
[    1.343095] x27: fffffff47fb65160 x26: 000000007fb6970d
[    1.343128] x25: 0000000000000000 x24: 000000000000002a
[    1.343188] x23: fffffff44eb62110 x22: ffffff9528ef5210
[    1.343220] x21: fffffff44eb62710 x20: 0000000000000000
[    1.343280] x19: fffffff44eb62100 x18: 0000000000000098
[    1.343313] x17: 0000000000000000 x16: ffffff95273a8a98
[    1.343374] x15: ffffff9527a70d4f x14: 3020736177207475
[    1.343406] x13: ffffff80105d982b x12: 0000000000000000
[    1.343467] x11: 0000000000000000 x10: ffffffffffffffff
[    1.343501] x9 : 934cf3dcfcf6cd00 x8 : 934cf3dcfcf6cd00
[    1.343534] x7 : 00f300be002000f3 x6 : fffffff475a4441e
[    1.343567] x5 : 0000000000000000 x4 : 000000000000000e
[    1.343599] x3 : 0000000000000010 x2 : 0000000000000001
[    1.343659] x1 : 0000000000000000 x0 : 0000000000000075
[    1.343693] Call trace:
[    1.343755]  __list_add_valid+0x94/0xb0
[    1.343790]  wakeup_source_register+0x120/0x158
[    1.343822]  device_wakeup_enable+0x58/0x100
[    1.343883]  device_init_wakeup+0x70/0xe8
[    1.343918]  usbpd_create+0xa8/0x7e8
[    1.343951]  pdphy_probe+0x220/0x2e8
[    1.344010]  platform_drv_probe+0x80/0xb8
[    1.344043]  really_probe+0x468/0x6a4
[    1.344075]  driver_probe_device+0xb0/0x138
[    1.344136]  __device_attach_driver+0x88/0x1ac
[    1.344170]  bus_for_each_drv+0x8c/0xd4
[    1.344203]  __device_attach+0xbc/0x168
[    1.344264]  device_initial_probe+0x20/0x2c
[    1.344297]  bus_probe_device+0x34/0x9c
[    1.344330]  deferred_probe_work_func+0x5c/0xf0
[    1.344393]  process_one_work+0x270/0x43c
[    1.344427]  worker_thread+0x314/0x4b0
[    1.344488]  kthread+0x140/0x150
[    1.344521]  ret_from_fork+0x10/0x18
[    1.344555] Code: f000fc20 9113b000 aa0803e1 97ec8a43 (d4210000)
[    1.344617] ---[ end trace 6cd728b9c400b2c8 ]---
[    1.346506] Kernel panic - not syncing: Fatal exception
[    1.346569] SMP: stopping secondary CPUs
[    1.986572] cnss: Crash shutdown with driver_state 0x0
[    1.986611] cnss: cnss_pci_collect_dump_info: PCIe link is in suspend state
[    2.199896] ipa ipa3_active_clients_panic_notifier:305
[    2.199896] ---- Active Clients Table ----
[    2.199896]
[    2.199896] Total active clients count: 0
[    2.199896]
[    2.199959] Kernel Offset: 0x1515800000 from 0xffffff8010000000
[    2.200021] CPU features: 0x00000254,a2002238
[    2.200054] Memory Limit: none
[    2.200115] NPU_INFO: npu_panic_handler: 733 Apps crashed
[    2.202015] SMP: stopping secondary CPUs
[    2.202051]
[    2.202111] Restarting Linux version 4.19-325-sony-pdx206 (root@a5849d50bdf6) (Android (6051079 based on r370808) clang version 10.0.1 (https://android.googlesource.com/toolchain/llvm-project b9738d6d99f614c8bf7a3e7c769659b313b88244), GNU ld (binutils-2.27-41d8fcb) 2.27.0.20170315) #1 SMP PREEMPT Sun Oct 5 17:06:50 UTC 2025
[    2.202111]
[    2.202201] Going down for restart now
```

看了一下内核代码，大约是usbpd的某个设备，唤醒初始化的时候，设备信息节点出了问题，注册的时候prev空指针了。此类链表错误通常与内核配置冲突或驱动未正确初始化有关，需验证相关配置（如`CONFIG_USB_PDPHY`）是否启用。不过，后来意识到内核配置出了问题，修改了内核配置文件方式，就没有再出现这个问题了。不过，假如今后再出现类似问题，应该也可以按这个思路调试。
