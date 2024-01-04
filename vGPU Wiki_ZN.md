# GPT-Academic Report
## 翻译 private_upload\default\2024-01-04-10-38-43\vGPU Wiki.MD.part-0.md

﻿vGPU_Unlock Wiki
社区支持的项目
由DualCoder（Jonathan Johansson）创建的项目[GitHub]
由Krutav Shah编写的文档[www.krutav.com]
点击这里加入我们的Discord服务器。
由以下人员编辑：
Julius Arkenberg，
Luclu7，
还有其他社区成员。

最后更新日期：2021年12月8日

目录：

1. 前言
2. 支持的硬件和软件
   1. 支持的vGPU类型
   2. 支持的硬件
   3. 支持的操作系统
3. 如何开始使用vGPU_Unlock
   1. 设置硬件
   2. 获取驱动程序和许可证
   3. 准备Linux系统
      1. 下载vGPU_Unlock脚本
      2. Red Hat Enterprise Linux（推荐）
      3. Debian/Ubuntu
      4. Proxmox VE
   4. 创建vGPU
      1. 使用libvirt正常使用mdevctl
      2. （仅限Proxmox）使用Proxmox Web界面
4. 设置许可证服务器
5. 设置客户机虚拟机
   1. Microsoft Windows
   2. Linux
6. 在vGPU旁边使用主机图形（单GPU系统）
   1. 安装合并驱动程序
   2. 使用Looking Glass
7. 故障排除和常见问题
8. 限制、已知问题和修复
   1. 高OpenGL不稳定性
   2. 受限于宣传
9. Nvidia vGPU的替代方案

此项目不提供任何担保或支持，Nvidia vGPU和GRID是NVIDIA Corporation的商标。他们对其产品和商标保留所有权利。此处提到的其他所有商标和产品均为其各自所有者的财产。本项目确实允许绕过使用本文档中提到的技术所涉及的任何费用或成本，并且使用者以使用者的风险使用该技术，而不是该项目的维护者。项目维护者对使用者的任何损害也不负有责任。虽然极不可能发生任何物理硬件损坏，但项目维护者和/或创建者在任何情况下都不承担责任。为了帮助做出更明智的决策，本项目已经以开源的方式发布，任何用户都可以查看所有代码和资源。我们不为任何恶意或有害的环境提供此项目。此外，此工具不得在任何生产环境中使用。

# 1、前言

Nvidia vGPU是由Nvidia开发的用于数据中心的专有技术，可与特定的Tesla和Quadro卡一起使用。该技术旨在针对虚拟桌面基础设施（VDI）市场，目的是使远程用户通过远程桌面解决方案来进行工作。不同的远程应用包括CAD/CAM软件、游戏（比如GeForce Now）、建筑和许多其他图形加速程序。

该技术通过将受支持的图形卡资源分配给多个虚拟机（VM）来工作。与传统虚拟机相比，这种方法的优势在于完全的图形加速，使用CPU光栅化并不那么可行。虚拟机在数据中心和VDI场景中很受欢迎，因为它们易于迁移和配置给员工和用户。远程用户可以连接到中央VDI服务器，以访问自己的工作区在“轻便”计算机上，例如笔记本电脑或瘦客户端。对于需要较高的3D和/或计算能力的图形工作负载，不能总是在超薄本和其他“轻便”计算机上运行。

这对于办公和其他企业环境需要远程图形能力是有意义的。但是对您来说意味着什么呢？vGPU_unlock的目标是允许用户在较差的图形卡或面向vGPU设计的专业数据中心图形卡的消费者版本上运行Nvidia vGPU技术。通过这样，用户可以为一些虚拟机虚拟化自己的图形卡。这可以让用户运行一个能够为朋友游戏的虚拟机，或者在Linux上使用带有图形加速（使用Looking Glass）的Windows等等。当然还有其他更多的用途，但这只是一些理想的用例。 

![image-20240104104509848](C:\Users\tianyu\AppData\Roaming\Typora\typora-user-images\image-20240104104509848.png)

vGPU是如何工作的，来源:NVIDIA

需不需要vGPU？这取决于具体情况。首先，在进行设置之前，最好对使用Linux以及这项技术如何工作有一个很好的理解。如果您需要图形加速的Windows或Linux虚拟机，您可能会考虑在您的环境中使用vGPU。我们将在稍后介绍一些特定情况下的替代方案。

# 2、支持的硬件和软件

### a、vGPU 类型 

| **vGPU Type** | **OS**         | **Use Case**            | **License** | **Display**           | **Notes**     |
| ------------- | -------------- | ----------------------- | ----------- | --------------------- | ------------- |
| A-series      | Windows, Linux | Virtual  Applications   | vApps       | 1280x1024, 1  display | Good for RDSH |
| B-series      | Windows, Linux | Basic PC work           | vPC         | Up to 5K, 2  displays | 45 FPS max    |
| C-series      | Linux          | Compute server          | vCS         | Up to 4K, 1  display  | CUDA Only     |
| Q-series      | Windows, Linux | ProfessionalWorkstation | vDWS        | Up to 8K, 4  displays | CUDA, OpenGL  |


​	注：有适当驱动程序的游戏系列 vGPU 类型也存在

### B. 支持的硬件

##### CPU 和主板：

需要虚拟化扩展。对于 Intel，这意味着 VT-X。AMD 系统上需要 AMD-V 来进行虚拟化。为了获得足够的虚拟机性能，应启用虚拟化扩展。请参考供应商提供的文档，确认您的 CPU 和主板是否支持虚拟化以及在 BIOS 中启用它的步骤。


请注意，某些系统上启用 vGPU 可能需要 IOMMU。Ampere 显卡需要启用 IOMMU。

##### 显卡： 

| **Nvidia vGPU cards** | **GPU Chip** | **vGPU unlock supported:** |
| --------------------- | ------------ | -------------------------- |
| Tesla M10             | GM107 x4     | Most Maxwell 1.0  cards    |
| Tesla M60             | GM204 x2     | Most Maxwell 2.0  cards    |
| Tesla P40             | GP102        | Most Pascal cards          |
| Tesla V100 16GB       | GV100        | Titan V, Quadro GV100      |
| Quadro RTX 6000       | TU102        | Most Turing cards          |
| RTX A6000             | GA102        | Ampere is  not supported   |

### C. 支持的操作系统

##### 主机：

* 红帽企业 Linux（经 Nvidia 认证适用于 vGPU，测试过的内核版本为 4.18）
* Proxmox VE（经测试的内核版本为 5.4）
* 这里可能还有其他未明确列出来的支持版本。


对于高于 5.10 的任何主机 Linux 核心，请使用补丁。
Linux 内核版本 5.13 似乎无法正常工作，不推荐使用。

##### 客户端虚拟机：

企业版 Linux 发行版（RHEL、CentOS、Fedora）
Debian/Ubuntu（20.04 LTS）
Windows 10、8.1、Server 2019 和 2016

________________

# 3、开始使用 vGPU_Unlock

##### 免责声明：

在不支持/未经认证的硬件上使用 vGPU 不被推荐，但此脚本仍可使某些显卡运行该技术。我们以「按您自己的风险使用」的方式向您提供此方案，请注意它可能不适用于您的情况。该项目以 MIT 许可证提供，不提供任何形式的保修。通过在线渠道，可能会提供社区对该项目的支持，但不保证可用性。

### A. 设置硬件

##### PC 硬件：

首先，在计算机关闭时将兼容的显卡安装到系统主板的可用 PCI Express 插槽中。这一步骤可能已经在之前完成。您还需要验证您的硬件是否支持运行 vGPU。有关详细信息，请参阅第 2 节。如果您的硬件不兼容，可能需要进行系统升级。

##### BIOS：

如果您的系统硬件准备好支持 vGPU，请打开系统并进入 BIOS。您需要查找两个选项，虚拟化扩展（Intel VT-x/AMD-V，虚拟化支持）和 IOMMU*（Intel VT-D/AMD-Vi/SR-IOV）。如果可能，请启用这两个功能以使用 vGPU。

*注意：IOMMU并非绝对必需。只有在安普系统和/或没有IOMMU时才需要。对于这两个过程，您应该参考您的系统供应商的文档，以获取有关如何实现所需配置和系统的具体细节。

### B. 获取驱动程序和许可证

您可以在Nvidia的vGPU / GRID申请页面上申请90天的Nvidia vGPU评估。在提交之前，您将被要求填写各种信息。从那里，申请处理时间可能需要2分钟到48小时不等。您将收到一封电子邮件，以设置密码并登录Nvidia许可证门户。在这里，您可以找到系统的驱动程序、许可证服务器安装程序和实际的许可证。
一旦您登录许可证门户，下载最新版本的Linux KVM vGPU驱动程序。它将以ZIP文件的形式提供，其中包含PDF指南以及其中的主机和虚拟机驱动程序。安装驱动程序和许可证将在后面进行介绍。

### C. 设置Linux主机

注意：本指南的此部分显示了正常vGPU设置。主机显卡在此模式下无法使用。请参阅第6节。有关Linux基础系统设置vGPU的说明因分发版本而异。Nvidia已经认证了Red Hat Enterprise Linux KVM与其vGPU技术的兼容性，但他们还提供了一个Linux KVM软件包，我们将在这里介绍。我们还将展示从安装到最后一步的步骤，以尽可能包含更多内容。

- 下载vGPU_Unlock

此项目及其文件可在https://github.com/DualCoder/vgpu_unlock上下载。您可以使用命令行中的git直接克隆它，也可以从GitHub下载zip文件，然后使用SFTP将文件传输到主机上。为了本指南的目的，我们将向您展示如何作为root用户设置vGPU，并使用git CLI。

1. ##### 如果尚未成为root用户，请切换为root：
   
    ​    sudo -i
    
2. ##### 进入/opt目录

  cd /opt

3. ##### 使用`git`克隆存储库（如果未安装，先安装git）：

  git clone https://github.com/DualCoder/vgpu_unlock

4. ##### 使新下载的文件夹可执行：

  chmod -R +x vgpu_unlock



- 设置企业版Linux发行版（RHEL、CentOS）

#####  1、安装操作系统：

  可以通过免费的开发者订阅下载Red Hat Enterprise Linux。不推荐使用Fedora。

##### 2、使用USB烧录镜像，按照您希望的设置正常安装操作系统。我们强烈建议启用root用户，并与以下软件包一起使用“服务器”安装：虚拟化超级监视器、开发工具和无头管理。

##### 3、安装完成后，运行以下命令。

  sudo -i
  dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm && update 
  dnf update
  dnf --enablerepo="epel" install dkms kernel-devel python3 python3-pip mdevctl
  pip3 install frida

##### 4、切换到我们将要工作的/opt/目录。

  cd /opt

##### 5、下载vGPU_Unlock脚本

  见第3.C节的第I部分，涵盖了下载脚本的方法。

#####  6、启用IOMMU并为vGPU禁用nouveau，请参阅：

  Red Hat 8-在GRUB中启用IOMMU
  禁用Nouveau驱动程序
  对于Intel：intel_iommu=on iommu=pt
  对于AMD：amd_iommu=on iommu=pt

##### 7、使用DKMS安装vGPU驱动程序（将<version>替换为您的驱动程序版本）

  chmod +x NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run
  ./NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run --dkms

##### 8、根据GitHub的说明编辑4个nvidia文件。

   nano /lib/systemd/system/nvidia-vgpud.service
   nano /lib/systemd/system/nvidia-vgpu-mgr.service
   nano /usr/src/nvidia-<version>/nvidia/os-interface.c
   nano /usr/src/nvidia-<version>/nvidia/nvidia.Kbuild

##### 9、删除并重新构建Nvidia驱动的DKMS（将<version>替换为您的驱动程序版本）

   dkms remove -m nvidia -v <version> --all
   dkms install -m nvidia -v <version>

##### 10、重新启动

- 设置基于Debian的系统（例如Ubuntu）

##### 1、安装操作系统：

   获取一个内核版本小于5.10的ISO。任何更新版本，如Debian Bullseye 5.10或更高版本，都需要针对vGPU驱动程序进行补丁。这是一个补丁。

##### 2、使用USB烧录镜像，按照您希望的设置正常安装操作系统。

##### 3、安装完成后，运行以下命令。

   sudo -i
        apt-get update && upgrade
   apt-get install dkms python3 python3-pip 

##### 4、切换到我们将要工作的/opt/目录。

   cd /opt

##### 5、下载vGPU_Unlock脚本

   见第3.C节的第I部分，涵盖了下载脚本的方法。

##### 6、启用IOMMU并为vGPU禁用nouveau，请参阅：

   Red Hat 8-在GRUB中启用IOMMU
   禁用Nouveau驱动程序
   对于Intel：intel_iommu=on iommu=pt
   对于AMD：amd_iommu=on iommu=pt

##### 7、使用DKMS安装vGPU驱动程序（将<version>替换为您的驱动程序版本）

   chmod +x NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run
   ./NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run --dkms

##### 8、根据GitHub的说明编辑这4个nvidia文件。

   nano /lib/systemd/system/nvidia-vgpud.service
   nano /lib/systemd/system/nvidia-vgpu-mgr.service
   nano /usr/src/nvidia-<version>/nvidia/os-interface.c
   nano /usr/src/nvidia-<version>/nvidia/nvidia.Kbuild

##### 9、删除并重新构建Nvidia驱动的DKMS（将<version>替换为您的驱动程序版本）

   dkms remove -m nvidia -v <version> --all
   dkms install -m nvidia -v <version>

##### 10重新启动

- 设置Proxmox VE系统

​    推荐使用Proxmox 5.4（内核版本4），而不是PVE 6.x

1. ##### 参见Debian安装说明

### D、创建vGPU

创建vGPU实例的方法因系统而异。大多数发行版都会使用libvirt作为事实标准的虚拟化库和工具。对于这些系统，mdevctl是管理中介设备（包括vGPU）的首选方式。然而，Proxmox等发行版在处理中介设备的管理时采用不同的界面。

##### I. 在libvirt中正常使用mdevctl

1. 首先，通过列出可用类型及其特性来确定要使用的vGPU。
    mdevctl types
    Red Hat文档 - 获取vGPU信息

2. 创建vGPU实例。适用于Red Hat系统的指南同样适用于其他发行版。
    Red Hat文档 - 创建vGPU实例

3. 通过编辑虚拟机的XML配置文件将vGPU连接到虚拟机上（您需要首先生成一个UUID）。
    Red Hat文档 - 连接vGPU实例
    下面是一个示例vGPU设备。将此代码块添加到虚拟机的XML配置文件的<devices>部分。

  ```
  <hostdev mode='subsystem' type='mdev' managed='no' model='vfio-pci'>
    <source>
      <address uuid='30820a6f-b1a5-4503-91ca-0c10ba58692a'/>
    </source>
  </hostdev>
  ```

  

##### II. 在Proxmox的Web界面中使用

1. 如果您想方便地列出要使用的vGPU类型，请先安装mdevctl。用以下命令列出所有类型：
mdevctl types
Red Hat文档 - 获取vGPU信息
2. 为虚拟机分配一个UUID。方法是使用uuidgen生成一个新的UUID，或者使用以UUID格式开始的VM ID。
3. 将新创建的UUID添加到虚拟机配置中。将<VM-ID>替换为您的虚拟机的数字ID。
nano /etc/pve/qemu-server/<VM-ID>.conf
然后，在文件的开头添加UUID（将<Your-UUID>替换为您生成的UUID）：
args: -uuid <Your-UUID>
4. 添加vGPU设备：
进入虚拟机的“Hardware”选项卡，点击“Add”，选择“PCI Device”。点击“Device:”，它将显示一个PCI设备列表，您要选择Nvidia显卡。一个新的菜单应该出现在旁边，名为“MDev Type”，您可以在其中选择要使用的vGPU配置文件。

# 4、设置许可证服务器
________________

许可证对于vGPU技术的正常运行至关重要，它是一种许可产品，需要Nvidia或其合作伙伴提供的有效许可证才能使用。要设置分发每个vGPU实例许可证的服务器，您需要开始下载服务器本身。您可以在Nvidia许可证门户上找到可供下载的最新许可证服务器，转到下载选项卡，并选择要下载的最新许可证服务器。这些下载文件可针对Windows和Linux操作系统运行许可证服务器。

**注意：Nvidia提供了相关官方文档和说明，建议参考。**

在Windows上，使用Nvidia安装程序UI非常简单。在安装许可证服务器之前，需要安装Oracle Java软件以及Tomcat Web服务器。

在Linux上，您需要首先安装Java和Tomcat Web服务器。

确保安装具有Web访问权限的许可证服务器，以便您可以从其他计算机访问它。安装并运行许可证服务器后，转到http://<serverIP>:8080/licserver，并单击“License Management”。

您需要一个许可证文件（.bin）。要获取该文件，返回Nvidia许可证门户。点击“Create server”按钮并填写详细信息，例如服务器名称、描述和最重要的是运行该服务器的MAC地址。可以使用Windows上的ipconfig /all或Linux上的ip a命令来查找系统中网络适配器的地址。

名称和描述可以随意填写。但是，MAC地址对于功能是重要的，并且必须设置为与主要网络适配器相同的MAC地址。

现在，您需要向服务器本身添加许可证（功能）。有不同类型的许可证，您可以将多种类型添加到一个许可证服务器文件中。不同的vGPU配置文件需要特定的许可证类型才能使用，例如A系列配置文件需要vApps许可证。

在门户上创建许可证服务器后，您可以下载关联的许可证（.bin）文件以在服务器上使用。

进入许可证服务器的Web界面，转到“License Management”部分，并上传已下载的许可证文件。

# 5、设置虚拟机客户机

________________

本节将介绍如何在分配了Nvidia vGPU中介设备的虚拟机上安装驱动程序。假设您已经在虚拟机上安装了支持的操作系统，可以继续安装vGPU的设备驱动程序。请注意，驱动程序仅适用于Microsoft Windows和Linux客户机。不支持基于BSD的操作系统和MacOS，并且也不太可能会得到支持。

### A. Microsoft Windows

Nvidia vGPU驱动程序和文档PDF格式的ZIP文件中包含了适用于Windows 10/Server 2019/2016和Windows 8/Server 2012的驱动程序。

要安装驱动程序，只需在虚拟机上运行相应的可执行文件，并按照图形化安装程序中的步骤安装驱动程序。安装完成后，可能需要重启虚拟机。

在极少数情况下，可能会提示用户重新启动，这可能意味着驱动程序由于某种原因无法正常工作。例如，代码43错误就是这样一种错误，可能由于配置错误或使用错误的驱动程序而导致。

您还必须启动Nvidia控制面板应用程序，方法是在桌面上右键单击并选择它。然后，将其连接到您创建的许可证服务器。在控制面板菜单中找到许可证选项卡，输入服务器的IP地址或主机名，并输入端口号。7070是默认端口号，除非您更改了它，否则必须指定新的端口号。

### B. Linux

有多种Linux发行版可供选择，但Nvidia只对其中的几个进行了认证，可以与vGPU驱动程序兼容。企业版Linux和基于Debian的系统具有特定的内核版本，最有可能能够正常工作。其他发行版或具有不同和更新的内核的较新版本可能会在安装过程中出现问题，并且可能需要修补程序进行修复。

在安装之前，您需要将nouveau驱动程序加入黑名单并重新构建initramfs映像。之后，您需要重新启动。
红帽文档-将Nouveau驱动程序加入黑名单
Debian/Ubuntu-将Nouveau驱动程序加入黑名单
要安装驱动程序，需要使包含的.run文件可执行，然后像正常安装软件一样安装它。安装程序将在命令行中显示一个图形界面，相对容易跟随。除非您有自己的要求，否则请回答它给您的问题，并选择“是”。

```
chmod +x NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run
./NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run
```

从这里开始，您可以重新启动虚拟机，然后应该能够使用加速图形桌面环境。您接下来要做的是启动Nvidia设置应用程序，并将其连接到您创建的许可证服务器。在应用程序中找到许可证选项卡，然后输入服务器的IP地址/主机名和端口号。默认端口号为7070，除非您更改了端口号，否则必须指定新的端口号。

# 6、在vGPU旁边使用主机图形

________________

许多使用vGPU_Unlock脚本的人可能只有一张显卡在他们的系统中，并希望能够将该系统用于除提供图形加速虚拟机的服务器外的其他用途。事实证明，实际上可以将常规的GeForce/Quadro驱动程序与vGPU二进制文件合并在一起，同时仍然正常使用您的Linux主机的图形桌面解锁vGPU功能。 
使用此方法仍然存在一些限制，例如主机是否可以使用CUDA和OpenCL等方面的不确定性。客户端虚拟机仍会正常运行，但如果主机上的用户正在主动执行任务，则可能会出现性能损失。

创建这个“合并的驱动程序”是一个比较复杂的过程，如果您是初学者，最好下载一个预制的驱动程序，而不是从头开始自己做。

[Google Drive link (Merged-driver 460.32.04)](https://drive.google.com/file/d/1y04jXsyWaObP5ed-EdEQ8_CHmgbcUUVD/view?usp=sharing)

[Google Drive link (Merged-driver 460.73.01) * 用于手动解锁*](https://drive.google.com/file/d/19HMkYV_Uu3bI-hxY1IxLy5ABXj0feLPg/view)

[Google Drive link (Merged-driver 460.73.01) *预解锁的版本，推荐使用！*](https://drive.google.com/file/d/1dCyUteA2MqJaemRKqqTu5oed5mINu9Bw/view?usp=sharing)



注意：如果您有预解锁版本的合并驱动程序，则可以跳过以下步骤。您可以像安装原始vGPU驱动程序一样正常安装它。

### A. 安装合并的驱动程序

此驱动程序可能缺少不必要的文件，安装时可能显示错误，但您可以忽略这些错误并继续安装。此驱动程序仅已在内核4.18至5.9的情况下进行过测试，且不需要补丁。对于较新的内核（例如5.12），请参阅此补丁文档。

要使用此合并驱动程序，只需按照先前概述的常规vGPU驱动程序的指令运行.RUN文件。在安装过程中可能会遇到一些错误，但这些错误不应该妨碍您的安装。

在文件nvidia-vgpud.service和nvidia-vgpu-mgr.service中，添加环境变量Environment="__RM_NO_VERSION_CHECK=1"到ExecStop的下一行。这些文件位于/lib/systemd/system/。

如果没有此环境变量，您将会遇到API不匹配的错误，因为合并的驱动程序包含有稍微不同驱动版本号的geforce驱动程序与vgpu-kvm驱动程序。

### B. 使用Looking Glass

Looking Glass允许虚拟机和主机之间共享内存缓冲区，以便携带虚拟机的帧缓冲区并在主机或其他虚拟机上显示。如果您想在窗口中镜像显示vGPU虚拟机的显示，这可能特别有用。您可以考虑使用它来提供给您的vGPU虚拟机“物理显示输出”，或者以实时视图的方式查看，而无需使用外部软件和硬件编码。

设置Looking Glass与将常规显卡通过给虚拟机传递的方式相同，无论是在Windows还是Linux上使用vGPU。因此，我们在这里链接了官方的说明文档。

注意：使用NVFBC API和vGPU的Looking Glass需要一个512MB的IVSHMEM大小。如果您使用DXGI的Looking Glass，可以使用他们的公式根据显示器分辨率计算大小。

# 7、故障排除和常见问题

________________

在使用需要一些设置与原设计为即插即用的外部脚本时，出现问题是非常常见的。本节的目标是回答一些常见问题及如何修复某些问题。随着问题的出现，将在此处添加更多场景。

要找出您可能遇到的问题，请使用journalctl -u nvidia-vgpu-mgr命令来获取Nvidia vGPU服务的日志，并向下滚动以查找最新的错误。其他错误信息的来源包括journalctl -u nvidia-vgpud和dmesg命令。

* Q: 需要什么驱动程序？
A: 您需要使用专有的Nvidia vGPU和GRID驱动程序，具体来说是Linux KVM版本。我们已经测试了R440至R460版本与此解锁工具的兼容性。

* Q: 我的系统有多个GPU，这个工具可以解锁所有的GPU吗？
A: 不可以，除非所有的GPU都具有相同的设备ID。

* Q: 我可以同时使用主机GPU显示和vGPU吗？
A: Nvidia vGPU是设计为在没有显示输出的GPU上运行的，因此只能使用vGPU技术。然而，只有一张GPU的人可能仍然希望使用主机GPU，在第6节中，我们介绍了社区制作的“合并驱动程序”，使其成为可能。

* 错误：vGPU在PCI Id上不受支持：
确保已将/path_to_vgpu_unlock/添加到/lib/systemd/system/nvidia-vgpud.service中，并使用systemctl restart nvidia-vgpud命令重启服务。

* 错误：vGPU仅在支持VGX的板卡上受支持：
确保已将/path_to_vgpu_unlock/添加到/lib/systemd/system/nvidia-vgpu-mgr.service中，并使用systemctl stop nvidia-vgpu-mgr; killall nvidia-vgpu-mgr; systemctl start nvidia-vgpu-mgr命令重启服务。

* 错误：NVRM：API版本不匹配：客户端具有版本___：
在/lib/systemd/system/的两个Nvidia vGPU服务文件中的ExecStopPost行下面添加Environment="__RM_NO_VERSION_CHECK=1"，然后重启两个Nvidia vGPU服务。

# 8、限制、已知问题和解决方案

________________

在使用vGPU_Unlock时，可能会遇到一些问题和限制。

### A. 高OpenGL不稳定性

OpenGL是许多程序用于在屏幕上渲染应用程序的图形API，它在专业软件中最常见。

问题是OpenGL似乎无法与vGPU GRID R440版本之后的Q系列vGPU配置文件一起工作。 A系列和B系列vGPU配置文件仍然具有正常的OpenGL功能，但是在其中存在一些不稳定性。有一些方法可以让Q系列卡片上的OpenGL正常工作，其中一种方法是安装R440 vGPU主机驱动程序和R440 GRID客户机驱动程序。

另一种解决方案是“欺骗”vGPU以正常的Quadro型号，例如使用虚拟机配置中的mediated vGPU设备的x-pci参数将vGPU伪装为Quadro P4000。通过这样做并使用低于450版本的驱动程序，我们能够实现非常良好的OpenGL性能，但无法使用CUDA和OpenCL计算技术。

### B. Ampere显卡不适用

在撰写本文时，Nvidia的Ampere一代显卡是他们最新和最先进的产品。基于Ampere的支持vGPU的Tesla和Quadro现在使用称为“SR-IOV”的功能，以提供更多基于硬件的传统vGPU解决方案，具有更好的性能。问题在于这是一项硬件功能，非vGPU认证卡的固件VBIOS上很可能被禁用。这使得越来越难以确定如何在这些Ampere消费级显卡上实现vGPU功能，除非进行一些更高级别的修改。

目前，我们能够将vGPU实例传递到一个带有RTX 3090的虚拟机中，但是在vGPU驱动程序初始化时，虚拟机将立即崩溃并蓝屏。不太可能支持Ampere显卡的vGPU解锁。

# 9、Nvidia vGPU的替代方案

________________

值得注意的是，Nvidia vGPU可能并不适用于每个人，特别是如果您拥有一张旧的或不受支持的显卡。不过，还有一些替代方案，以下是其中一些。

### A. Intel GVT-G

这项技术专为英特尔图形处理器设计，使它们可以像Nvidia vGPU一样进行虚拟化。在撰写本文时，从英特尔Broadwell一代CPU到Comet Lake一代CPU的图形处理器都得到支持。驱动程序支持良好，但这些集成GPU的性能不那么出色。

### B. AMD MxGPU

这项技术专为某些AMD专业显卡设计，使它们可以像Nvidia vGPU一样进行虚拟化。在撰写本文时，仅支持少数几张显卡，例如Radeon Instinct系列、GCN3.0 FirePro S7150以及一些数据中心Radeon Pro。该技术还利用了SR-IOV，在某些领域是有益的。然而，获得这些显卡很困难，只有FirePro S7150s相对容易获得。但是，该显卡存在一些限制，如缺乏硬件编码和性能较差。

### C. Microsoft GPU-P

Microsoft开发了一种更好的后继技术，取代了它为用户提供的旧RemoteFX图形加速，以为Hyper-V虚拟机提供图形加速。这项技术称为GPU-P或GPU分区化/虚拟化，允许在Hyper-V虚拟机上运行的虚拟机访问主机图形卡资源并获得高性能的图形加速。它可以与许多显卡配合使用，甚至可以过度分配。它可用于免费安装在Windows 10和Windows Server上的Hyper-V虚拟机管理程序中，是我们推荐的解决方案，特别是在Windows环境中，因为它具有巨大的资源和成本节省潜力。

### D. “免费”的Nvidia vGPU

在Nvidia仍在生产Kepler架构显卡时，他们发布了几款基于Kepler架构的显卡，包括GRID K1和GRID K2，这些显卡能够运行vGPU直到GRID 4。此后，支持已被停止，Nvidia不再为这些卡发布任何驱动程序。这一代的产品已于2020年终止。这种第一代图形虚拟化技术没有任何许可费用，除了该虚拟机管理程序的费用，该虚拟机管理程序在这种情况下要么是Citrix XenServer，要么是VMware ESXi。这些显卡现在相当便宜，但我们无法建议您在现在这个时代购买并使用它们。

