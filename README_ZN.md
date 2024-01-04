# vgpu_unlock

解锁消费级 Nvidia GPU 的 vGPU 功能。

## 重要！

在某些情况下，无法保证此工具能够立即使用，因此请自行决定是否使用。

## 描述

此工具启用了在 NVIDIA vGPU 图形虚拟化技术中使用 GeForce 和 Quadro GPU 的功能。NVIDIA vGPU 通常仅通过设计支持一些数据中心的 Tesla 和专业的 Quadro GPU，而通过软件限制，不支持消费级图形卡。
这个 vgpu_unlock 工具旨在在基于 Linux 的系统上消除此限制，从而使大多数基于 Maxwell、Pascal、Volta（未经测试）和 Turing 架构的 GPU 能够使用 vGPU 技术。Ampere 支持目前正在进行中。
由 Krutav Shah 撰写的由社区维护的 Wiki 包含更多信息，[请点击此处查看](https://docs.google.com/document/d/1pzrWJ9h-zANCtyqRgS7Vzla0Y8Ea2-5z2HEi4X75d2Q/edit?usp=sharing)。

## 依赖：

* 此工具需要 Python3 和 Python3-pip；建议使用最新版本。
* 需要安装 python 包 "frida"。运行 `pip3 install frida` 进行安装。
* 该工具需要 NVIDIA GRID vGPU 驱动程序。
* 需要安装 "dkms"，因为它简化了驱动程序重建过程。通过操作系统的包管理器安装 DKMS。

## 安装：

在以下说明中，`<path_to_vgpu_unlock>` 需要替换为目标系统上此存储库的路径，`<version>` 需要替换为 NVIDIA GRID vGPU 驱动程序的版本。

安装 NVIDIA GRID vGPU 驱动程序，确保将其安装为 dkms 模块。
```
./nvidia-installer --dkms
```

修改 `/lib/systemd/system/nvidia-vgpud.service` 和 `/lib/systemd/system/nvidia-vgpu-mgr.service` 文件中以 `ExecStart=` 开头的行，将 `vgpu_unlock` 用作可执行文件，并将原始可执行文件作为第一个参数传递。示例：
```
ExecStart=<path_to_vgpu_unlock>/vgpu_unlock /usr/bin/nvidia-vgpud
```

重新加载 systemd 守护程序：
```
systemctl daemon-reload
```

修改文件 `/usr/src/nvidia-<version>/nvidia/os-interface.c`，在文件开头以 `#include` 开头的行之后添加以下行。
```
#include "<path_to_vgpu_unlock>/vgpu_unlock_hooks.c"
```

修改文件 `/usr/src/nvidia-<version>/nvidia/nvidia.Kbuild`，在文件底部添加以下行。
```
ldflags-y += -T <path_to_vgpu_unlock>/kern.ld
```
使用 dkms 删除 nvidia 内核模块：
```
dkms remove -m nvidia -v <version> --all
```
使用 dkms 重新构建和安装 nvidia 内核模块：
```
dkms install -m nvidia -v <version>
```

重新启动。

---
**注意**

此脚本仅适用于与其专业 Tesla 同代的显卡。因此，仅支持 Maxwell 及更新的 Nvidia GPU。
它不设计用于与低端显卡型号一起使用，因此并非所有显卡都能保证与 vGPU 一起正常工作。为了获得最佳体验，建议使用与 Tesla 显卡相同芯片型号的显卡。对于操作系统也是一样，某些最新的 Linux 发行版可能与 vGPU 软件不兼容。

---

## 工作原理

### 是否支持 vGPU？

为了确定某个 GPU 是否支持 vGPU 功能，驱动程序查看 PCI 设备 ID。此标识符与 PCI 供应商 ID 一起对于每种类型的 PCI 设备都是唯一的。为了启用 vGPU 支持，我们需要告诉驱动程序已安装 GPU 的 PCI 设备 ID 是 vGPU 支持的 GPU 使用的设备 ID 之一。

### 用户空间脚本：vgpu_unlock

用户空间服务 nvidia-vgpud 和 nvidia-vgpu-mgr 使用 ioctl 系统调用与内核模块通信。它们具体读取 PCI 设备 ID 并确定安装的 GPU 是否支持 vGPU。

Python 脚本 vgpu_unlock 截取了指定为第一个参数的可执行文件与内核之间的所有 ioctl 系统调用。然后，该脚本修改内核响应，以指示具有 vGPU 支持的 PCI 设备 ID 和支持 vGPU 的 GPU。

### 内核模块挂钩：vgpu_unlock_hooks.c

为了与 GPU 交换数据，内核模块将 PCI 总线的物理地址空间映射到自己的虚拟地址空间中。使用 ioremap\* 内核函数完成此操作。然后，内核模块读取和写入该映射的地址空间中的数据。这是使用 memcpy 内核函数完成的。

通过将 vgpu_unlock_hooks.c 文件包含到 os-interface.c 文件中，我们可以使用 C 预处理宏来替换和截取对 iormeap 和 memcpy 函数的调用。这样做允许我们保持对已映射的位置以及正在访问的数据的视图。

### 内核模块链接器脚本：kern.ld

这是 gcc 提供的默认链接器脚本的修改版本。该脚本已修改为将 nv-kernel.o 的 .rodata 部分放入 .data 而不是 .rodata，从而使其可写。该脚本还提供了符号 `vgpu_unlock_nv_kern_rodata_beg` 和 `vgpu_unlock_nv_kern_rodata_end`，以让我们知道该部分的起始和结束位置。

### 如何协同工作

启动后，nvidia-vgpud 服务会查询所有已安装 GPU 是否具有 vGPU 支持。此调用被 vgpu_unlock Python 脚本截取，并使 GPU 具有 vGPU 支持。如果找到支持 vGPU 的 GPU，则 nvidia-vgpu 创建一个 MDEV 设备，并系统创建了 /sys/class/mdev_bus 目录。

现在，可以通过将 UUID 回显到 mdev 总线表示中的 `create` 文件中来创建 vGPU 设备。这将创建表示新 vGPU 设备的其他结构。然后，这些设备可以分配给 VM，并在 VM 启动时，它将打开 MDEV 设备。这导致 nvidia-vgpu-mgr 开始使用 ioctl 与内核通信。同样，vgpu_unlock Python 脚本截取这些调用，当 nvidia-vgpu-mgr 询问 GPU 是否支持 vGPU 时，答案更改为是。检查完成后，它尝试初始化 vGPU 设备实例。

vGPU 设备的初始化由内核模块处理，并执行其自己的 vGPU 能力检查，这有点复杂。

内核模块将物理 PCI 地址范围 0xf0000000-0xf1000000 映射到其虚拟地址空间，然后执行一些我们不太了解的神奇操作。我们所知道的是，在这些操作之后，它访问物理地址 0xf0029624 处的 128 位值，我们称之为魔术值。内核模块还在物理地址 0xf0029634 处访问 128 位值，我们称之为密钥值。

然后，内核模块对魔术值有一些查找表，分别用于 vGPU 可能的 GPU 和其他 GPU。因此，内核模块在这两个查找表中查找魔术值，如果找到，则该表项还包含一组 AES-128 加密的数据块和一个 HMAC-SHA256 签名。

然后，通过使用先前提到的密钥值计算 HMAC-SHA256 签名，验证签名。如果签名正确，则使用 AES-128 解密数据块，该密钥相同。

在解密的数据块中再次包含 PCI 设备 ID。

因此，为了使内核模块接受 GPU 为具有 vGPU 能力，魔术值必须在 vGPU 能力的魔术值表中，密钥必须生成有效的 HMAC-SHA256 签名，并且 AES-128 解密的数据块必须包含具有 vGPU 能力的 PCI 设备 ID。如果这些检查中的任何一个失败，那么将返回错误代码 0x56 "不支持的调用"。

为了使这些检查通过，vgpu_unlock_hooks.c 中的挂钩将查找 ioremap 调用，该调用映射包含魔术和密钥值的物理地址范围，重新计算这些值的地址为内核模块的虚拟地址空间，监视读取这些地址处的 memcpy 操作，如果发生此类操作，则保留该值的副本，直到两者都已知，找到 nv-kernel.o 的 .rodata 部分中的查找表，找到签名和数据块，验证签名，解密块，编辑解密的数据中的 PCI 设备 ID，重新加密块，重新生成签名，并将魔术值、块和签名插入具有 vGPU 能力的魔术值表中。这就是它们的操作。
