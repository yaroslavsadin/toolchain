.. highlight:: none

How to build Linux with GNU tools and run on simulator
======================================================

This document describes how to build Linux kernel image from the perspective
of toolchain developer. This document doesn't aim to replace more complete and
thorough Linux-focused user guides and how to so. This document answers the
single question "How to confirm, that I can build Linux kernel with *that*
toolchain?"

To learn how to configure Linux, debug kernel itself and build extra software
please see `<https://github.com/foss-for-synopsys-dwc-arc-processors/linux/wiki>`_.


Prerequisites
-------------

* Host OS:

  * RHEL 7 or later

  * Ubuntu 18.04 LTS or later

* GNU toolchain 2021.09 for ARC or later

* make version at least 3.81

* rsync version at least 3.0

* git

Prerequisite packages can be installed on Ubuntu 18.04 with the following command as ``root``:

.. code-block:: shell

    apt-get install texinfo byacc flex build-essential git

On RHEL 7 those can be installed with following command as ``root``:

.. code-block:: shell

    yum groupinstall "Development Tools"
    yum install texinfo-tex byacc flex git


Overview
--------

There are two essential components to get a working Linux kernel image: root
file system and Linux image itself. For the sake of simplicity this guide
assumes that root file system is embedded into the Linux image.

To generate root file system this guide will use `Buildroot
<http://buildroot.org>`_ project, that automates this sort of things. Buildroot
is capable to build Linux image itself, feature that is also used in this
guide. Buildroot is also capable of building toolchain from the source, however
this feature is not used in this guide, instead binary toolchain distributed by
Synopsys will be used.


Configuring
-----------

Check `Buildroot downloads page <http://buildroot.org/download.html>`_ for
latest release. This guide further assumes latest snapshot. Get Buildroot
sources:

.. code-block:: shell

    wget https://buildroot.org/downloads/buildroot-2022.08.tar.gz
    tar -xf buildroot-2022.08.tar.gz
    mv buildroot-2022.08 buildroot
    mkdir buildroot/build
    cd buildroot/build

To build Linux and rootfs Buildroot should be configured. For the purpose of
this guide, a custom "defconfig" file will be created and then will be used to
configure Buildroot. Custom "defconfig" file can be located anywhere and have
any name. For example it can be ``buildroot/build/hs_defconfig``.
Contents of this file should be following::

    BR2_arcle=y
    BR2_archs38=y
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_uclibc_le_archs_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_GCC_VERSION_12_X=y
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_DEFCONFIG="haps_hs"
    BR2_LINUX_KERNEL_VMLINUX=y
    BR2_TARGET_ROOTFS_INITRAMFS=y

Important notes about modifying Buildroot defconfig:

* ``BR2_TOOLCHAIN_EXTERNAL_URL`` should point to a valid URL of GNU Toolchain
  for ARC distributable.
* ``BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_X_XX`` should be aligned to Linux headers
  version used for the toolchain build.

  =================== =======================
  Toolchain version   Linux headers version
  =================== =======================
  2023.03             5.16
  2022.09             5.16
  2021.09             5.1
  2021.03             5.1
  2020.09             5.7
  2020.03             4.15
  2019.09             4.15
  2019.03             4.15
  2018.09             4.15
  2018.03             4.15
  2017.09             4.12
  2017.03             4.9
  2016.09             4.8
  2016.03             4.6
  2015.06, 2015.12    3.18
  earlier             3.13
  =================== =======================

  This parameter identifies version of Linux that was used to build toolchain and
  is not related to version of Linux that will be *built by* the toolchain or where
  applications compiled by this toolchain will run.
* For building big endian linux you have to replace ``BR2_arcle=y`` to
  ``BR2_arceb=y`` and change value of ``BR2_TOOLCHAIN_EXTERNAL_URL`` to respective
  URL for your processor.
* Other Linux kernel defconfigs can be used.
* Building GDB or GDBserver is not necessary.


.. _linux-building-label:

Building
--------

To build Linux kernel image using that defconfig:

.. code-block:: shell

    make -C .. O=`readlink -e .` defconfig DEFCONFIG=`readlink -e ./hs_defconfig`
    make

After that there will be Linux kernel image file ``buildroot/build/images/vmlinux``.


Running on nSIM
---------------

Linux configuration in the provided Buildroot defconfig is for the standalone
nSIM. This kernel image can be run directly on nSIM, without any other
additional software. Assuming current directory is ``buildroot/build``.

Before running the image with nSIM create a file with properties ``hs4x.props``::

    nsim_fast=0
    nsim_isa_family=av2hs
    nsim_isa_core=1
    nsim_isa_atomic_option=1
    nsim_isa_ll64_option=1
    nsim_isa_mpy_option=9
    nsim_isa_div_rem_option=2
    nsim_isa_sat=1
    nsim_isa_code_density_option=2
    nsim_isa_enable_timer_0=1
    nsim_isa_enable_timer_1=1
    nsim_isa_rtc_option=1
    icache=65536,64,4,0
    dcache=65536,64,2,0
    mmu_version=4
    mmu_pagesize=8192
    nsim_mem-dev=uart0,kind=dwuart,base=0xf0000000,irq=24
    nsim_isa_number_of_interrupts=32
    nsim_isa_number_of_external_interrupts=32

Then run the image:

.. code-block:: shell

    nsimdrv -propsfile hs4x.props images/vmlinux

Username is ``root`` without a password. To halt target system issue ``halt``
command.

Use ``nsim_fast=1`` in props file if you have nSIM Pro license.

For more information visit this page: `How to run ARC Linux kernel and debug <https://github.com/foss-for-synopsys-dwc-arc-processors/linux/wiki/How-to-run-ARC-Linux-kernel-and-debug-%28with-MetaWare-Debugger%29>`_

Using different Linux configuration
-----------------------------------

It is possible to change Linux configuration used via altering
``BR2_LINUX_KERNEL_DEFCONFIG`` property of Buildroot defconfig. For example to
build kernel image for AXS103 SDP change its value to ``axs103``. After that
repeat steps from :ref:`linux-building-label` section of this document.  Refer
to `ARC Linux documentation
<https://github.com/foss-for-synopsys-dwc-arc-processors/linux/wiki>`_ for more
details about how to enable networking, HDMI and other hardware features of
AXS10x SDP.

Notable defconfigs available for ARC: ``axs101``, ``axs103``, ``axs103_smp``,
``vdk_hs38_smp``, ``hsdk``.


Using glibc toolchain
---------------------

Configuration for glibc toolchain is fairly similar for uClibc, with only minor
differences::

    BR2_arcle=y
    BR2_archs38=y
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM_GLIBC=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_glibc_le_archs_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_TOOLCHAIN_EXTERNAL_INET_RPC=n
    BR2_GCC_VERSION_12_X=y
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_DEFCONFIG="haps_hs"
    BR2_LINUX_KERNEL_VMLINUX=y
    BR2_TARGET_ROOTFS_INITRAMFS=y

But please note due to use of glibc it's required to disable use of built-in Sun RPC, which as of today is force-enabled for external glibc-based toolchains.
That's why ``BR2_TOOLCHAIN_EXTERNAL_INET_RPC=n`` is used too.


Linux for ARC 770 processors
----------------------------

Process of building kernel for ARC 770 is similar to what is for ARC HS. It is
required only to change several option in Buildroot defconfig: ``BR2_archs38``
(replace by ``BR2_arc770d``), ``BR2_TOOLCHAIN_EXTERNAL_URL`` and ``BR2_LINUX_KERNEL_DEFCONFIG``.
The resulting configuration file::

    BR2_arcle=y
    BR2_arc770d=y
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_uclibc_le_arc700_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_GCC_VERSION_12_X=y
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_DEFCONFIG="nsim_700"
    BR2_LINUX_KERNEL_VMLINUX=y
    BR2_TARGET_ROOTFS_INITRAMFS=y

Then repeat steps from :ref:`linux-building-label` section of this document to build
Linux kernel image.

Before running the image with nSIM create a file with properties ``arc700.props``::

    nsim_fast=0
    nsim_mem-dev=uart0,kind=dwuart,base=0xf0000000,irq=24
    icache=32768,64,2,0
    dcache=32768,64,4,0
    nsim_mmu=3
    nsim_isa_family=a700
    nsim_isa_atomic_option=1
    nsim_isa_dpfp=none
    nsim_isa_shift_option=2
    nsim_isa_swap_option=1
    nsim_isa_bitscan_option=1
    nsim_isa_sat=1
    nsim_isa_mpy32=1
    nsim_isa_enable_timer_0=1
    nsim_isa_enable_timer_1=1
    isa_counters=1
    nsim_isa_pct_counters=8
    nsim_isa_pct_size=48

Then run the image:

.. code-block:: shell

    nsimdrv -propsfile arc700.props images/vmlinux


Linux for ARC HS VDK
--------------------

This section is specific to ARC HS VDK which is distributed along with nSIM
(nSIM Pro license is required).

Buildroot defconfig for VDK differs from the one for a simple nSIM:

* Linux defconfig is ``vdk_hs38_smp``.
* Ext2 file of root file system should be created, instead of being linked into
  the kernel.

With those changes Buildroot defconfig for ARC HS VDK is::

    BR2_arcle=y
    BR2_archs38=y
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_uclibc_le_archs_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_GCC_VERSION_12_X=y
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_DEFCONFIG="vdk_hs38_smp"
    BR2_LINUX_KERNEL_VMLINUX=y
    BR2_TARGET_ROOTFS_EXT2=y

Save this defconfig to some file (for example ``vdk_defconfig``). Then use same
process as in :ref:`linux-building-label` section:

.. code-block:: shell

    make -C .. O=`readlink -e .` defconfig DEFCONFIG=`readlink -e ./vdk_defconfig`
    make

ARC HS VDK already includes Linux kernel image and root file system image. To
replace them with your newly generated files:

.. code-block:: shell

    cd <VDK-directory>/skins/ARC-Linux
    mv rootfs.ARCv2.ext2{,.orig}
    ln -s <path-to-Buildroot-output/images/rootfs.ext2 rootfs.ARCv2.ext2
    mv ARCv2/vmlinux_smp{,.orig}
    ln -s <path-to-Buildroot-output/images/vmlinux ARCv2/vmlinux_smp

Before running VDK if you wish to have a working networking connection on Linux
for ARC system it is required to configure VDK VHub application. By default
this application will pass all Ethernet packets to the VDK Ethernet model,
however on busy networks that can be too much to handle in a model, therefore
it is highly recommended to configure destination address filtering. Modify
``VirtualAndRealWorldIO/VHub/vhub.conf``: set ``DestMACFilterEnable`` to
``true``, and append some random valid MAC address to the list of
``DestMACFilter``, or use one of the MAC address examples in the list. This
guide will use ``D8:D3:85:CF:D5:CE`` - this address is already in the list. Note
that is has been observed that it is not possible to assign some addresses to
Ethernet device model in VDK, instead of success there is an error "Cannot
assign requested address".

Note, that due to the way how VHub application works, it is impossible to
connect to the Ethernet model from the host on which it runs on and vice versa.
Therefore to use networking in target it is required to either have another
host and communicate with it.

Run VHub application as ``root``:

.. code-block:: shell

    VirtualAndRealWorldIO/VHub/vhub -f VirtualAndRealWorldIO/VHub/vhub.conf

In another console launch VDK:

.. code-block:: shell

    . setup.sh
    ./skins/ARC-Linux/start_interactive.tcl

After VDK will load, start simulation. After Linux kernel will boot, login into
system via UART console: login ``root``, no password. By default networking is
switched off. Enable ``eth0`` device, configure it is use MAC from address
configured in VHub::

    [arclinux] # ifconfig eth0 hw ether d8:d3:85:cf:d5:ce
    [arclinux] # ifconfig eth0 up

Linux kernel will emit errors about failed PTP initialization - those are
expected. Assign IP address to the target system. This example uses DHCP::

    [arclinux] # udhcpc eth0

Now it is possible to mount some NFS share and run applications from it::

    [arclinux] # mount -t nfs public-nfs:/home/arc_user/pub /mnt
    [arclinux] # /mnt/hello_world


Linux for AXS103 SDP
--------------------

Build process using Buildroot is the same as for standalone nSIM. Buildroot
defconfig is::

    BR2_arcle=y
    BR2_archs38=y
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_uclibc_le_archs_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_GCC_VERSION_12_X=y
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_DEFCONFIG="axs103_smp"
    BR2_TARGET_ROOTFS_INITRAMFS=y

This defconfig will create a uImage file instead of vmlinux. Please refer to
`ARC Linux wiki
<https://github.com/foss-for-synopsys-dwc-arc-processors/linux/wiki/Getting-Started-with-Linux-on-ARC-AXS103-Software-Development-Platform-(SDP)>`_
for more details on using u-boot with AXS103.

Linux for ARC HS58
--------------------

Get Buildroot sources:

.. code-block:: shell

    git clone -b arc-2022.09 https://github.com/foss-for-synopsys-dwc-arc-processors/buildroot
    mkdir buildroot/build
    cd buildroot/build

To build Linux and rootfs Buildroot should be configured. For the purpose of
this guide, a custom "defconfig" file will be created and then will be used to
configure Buildroot. Custom "defconfig" file can be located anywhere and have
any name. For example it can be ``buildroot/build/arc32_defconfig``.
Contents of this file should be following::

    BR2_arcle=y
    BR2_arc32=y
    # BR2_STRIP_strip is not set
    BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_5_16=y
    BR2_ROOTFS_POST_IMAGE_SCRIPT="board/synopsys/arc64/post-image.sh"
    BR2_ROOTFS_POST_SCRIPT_ARGS="$(LINUX_DIR)"
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_CUSTOM_GIT=y
    BR2_LINUX_KERNEL_CUSTOM_REPO_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/linux.git"
    BR2_LINUX_KERNEL_CUSTOM_REPO_VERSION="arc64"
    BR2_LINUX_KERNEL_DEFCONFIG="haps_hs5x"
    BR2_LINUX_KERNEL_IMAGE_TARGET_CUSTOM=y
    BR2_LINUX_KERNEL_IMAGE_TARGET_NAME="loader"
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_arc32_uclibc_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_INET_RPC=n
    BR2_TOOLCHAIN_EXTERNAL_GCC_12=y
    BR2_PACKAGE_LINUX_TOOLS_PERF=y
    BR2_TARGET_ROOTFS_INITRAMFS=y

To build Linux kernel image using that defconfig:

.. code-block:: shell

    make -C .. O=`readlink -e .` defconfig DEFCONFIG=`readlink -e ./arc32_defconfig`
    make

After that Linux kernel images will be in ``buildroot/build/images``: ``vmlinux`` and ``loader``.
The ``loader`` image must be used for running with nSIM.

Before running the image with nSIM create a file with properties ``hs5x.props``::

    nsim_fast=0
    nsim_isa_family=av3hs
    nsim_isa_dc_hw_prefetch=1
    nsim_isa_dual_issue_option=1
    nsim_isa_atomic_option=2
    nsim_isa_m128_option=0
    nsim_isa_ll64_option=1
    nsim_isa_mpy_option=9
    nsim_isa_div_rem_option=2
    nsim_isa_enable_timer_0=1
    nsim_isa_enable_timer_1=1
    nsim_isa_rtc_option=1
    icache=16384,64,4
    dcache=16384,64,2
    mmu_version=16
    mmu_pagesize=4096
    mmu_address_space=32
    nsim_mem-dev=uart0,kind=dwuart,base=0xf0000000,irq=24
    nsim_isa_number_of_interrupts=32
    nsim_isa_number_of_external_interrupts=32
    nsim_isa_has_fp=1
    nsim_isa_fp_dds_option=1
    nsim_isa_fp_div_option=1
    nsim_isa_fp_dp_option=1
    nsim_isa_fp_hp_option=1
    nsim_isa_fp_vec_option=1
    nsim_isa_fp_wide_option=1

Then run the image:

.. code-block:: shell

    nsimdrv -propsfile hs5x.props images/loader


Linux for ARC HS68
--------------------

Get Buildroot sources:

.. code-block:: shell

    git clone -b arc-2022.09 https://github.com/foss-for-synopsys-dwc-arc-processors/buildroot
    mkdir buildroot/build
    cd buildroot/build

To build Linux and rootfs Buildroot should be configured. For the purpose of
this guide, a custom "defconfig" file will be created and then will be used to
configure Buildroot. Custom "defconfig" file can be located anywhere and have
any name. For example it can be ``buildroot/build/arc64_defconfig``.
Contents of this file should be following::

    BR2_arcle=y
    BR2_arc64=y
    # BR2_STRIP_strip is not set
    BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_5_16=y
    BR2_ROOTFS_POST_IMAGE_SCRIPT="board/synopsys/arc64/post-image.sh"
    BR2_ROOTFS_POST_SCRIPT_ARGS="$(LINUX_DIR)"
    BR2_LINUX_KERNEL=y
    BR2_LINUX_KERNEL_CUSTOM_GIT=y
    BR2_LINUX_KERNEL_CUSTOM_REPO_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/linux.git"
    BR2_LINUX_KERNEL_CUSTOM_REPO_VERSION="arc64"
    BR2_LINUX_KERNEL_DEFCONFIG="haps_arc64"
    BR2_LINUX_KERNEL_IMAGE_TARGET_CUSTOM=y
    BR2_LINUX_KERNEL_IMAGE_TARGET_NAME="loader"
    BR2_TOOLCHAIN_EXTERNAL=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM=y
    BR2_TOOLCHAIN_EXTERNAL_CUSTOM_GLIBC=y
    BR2_TOOLCHAIN_EXTERNAL_DOWNLOAD=y
    BR2_TOOLCHAIN_EXTERNAL_URL="https://github.com/foss-for-synopsys-dwc-arc-processors/toolchain/releases/download/arc-2023.03-rc1/arc_gnu_2023.03-rc1_prebuilt_arc64_glibc_linux_install.tar.gz"
    BR2_TOOLCHAIN_EXTERNAL_HEADERS_5_16=y
    BR2_TOOLCHAIN_EXTERNAL_LOCALE=y
    BR2_TOOLCHAIN_EXTERNAL_HAS_SSP=y
    BR2_TOOLCHAIN_EXTERNAL_CXX=y
    BR2_TOOLCHAIN_EXTERNAL_INET_RPC=n
    BR2_TOOLCHAIN_EXTERNAL_GCC_12=y
    BR2_PACKAGE_LINUX_TOOLS_PERF=y
    BR2_PACKAGE_ELFUTILS=y
    BR2_TARGET_ROOTFS_INITRAMFS=y

To build Linux kernel image using that defconfig:

.. code-block:: shell

    make -C .. O=`readlink -e .` defconfig DEFCONFIG=`readlink -e ./arc64_defconfig`
    make

After that Linux kernel images will be in ``buildroot/build/images``: ``vmlinux`` and ``loader``.
The ``loader`` image must be used for running with nSIM.

Before running the image with nSIM create a file with properties ``hs6x.props``::

    nsim_fast=0
    nsim_isa_dual_issue_option=1
    nsim_isa_has_hw_pf=1
    nsim_isa_m128_option=1
    nsim_isa_has_hw_pf=1
    nsim_isa_vec64=1
    nsim_isa_family=arc64
    nsim_isa_enable_timer_0=1
    nsim_isa_enable_timer_1=1
    nsim_isa_rtc_option=1
    nsim_isa_addr_size=64
    nsim_isa_pc_size=64
    icache=16384,64,4,o
    dcache=16384,64,4,o
    mmu_version=16
    mmu_pagesize=4096
    mmu_address_space=48
    nsim_mem-dev=uart0,kind=dwuart,base=0xf0000000,irq=24
    nsim_isa_mpy_option=9
    nsim_isa_mpy64=1
    nsim_isa_div64_option=1
    nsim_isa_div_rem_option=2
    nsim_isa_atomic_option=2
    nsim_isa_has_fp=1
    nsim_isa_fp_dds_option=1
    nsim_isa_fp_div_option=1
    nsim_isa_fp_dp_option=1
    nsim_isa_fp_hp_option=1
    nsim_isa_fp_vec_option=1
    nsim_isa_fp_wide_option=1

Then run the image:

.. code-block:: shell

    nsimdrv -propsfile hs6x.props images/loader
