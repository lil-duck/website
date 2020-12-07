---
title: "如何实现一个文件系统"
date: 2020-12-05T10:32:25+08:00
author: "康华"
keywords: ["文件系统"]
categories : ["文件系统"]
banner : "img/blogimg/2.png"
summary : "本文目的是分析在Linux系统中如何实现新的文件系统。在介绍文件系统具体实现前先介绍文件系统的概念和作用，抽象出了文件系统概念模型。"
---

# 如何实现一个文件系统(之一)

**本文作者**：**康华**：计算机硕士，主要从事Linux操作系统内核、Linux技术标准、计算机安全、软件测试等领域的研究与开发工作，现就职于信息产业部软件与集成电路促进中心所属的MII-HP Linux软件实验室。如果需要可以联系通过kanghua151@msn.com联系他。

**摘要** ：本文目的是分析在Linux系统中如何实现新的文件系统。在介绍文件系统具体实现前先介绍文件系统的概念和作用，抽象出了文件系统概念模型。熟悉文件系统的内涵后，我们再近一步讨论Linux系统中和文件系统的特殊风格和具体文件系统在Linux中组成结构，为读者勾画出Linux中文件系统工作的全景图。最后，我们再通过Linux中最简单的Romfs作实例分析实现文件系统的普遍步骤。（我们假定读者已经对Linux文件系统初步了解）

## 什么是文件系统

首先要谈的概念就是什么是文件系统，它的作用到底是什么。

文件系统的概念虽然许多人都认为是再清晰不过的了，但其实我们往往在谈论中或多或少地夸大或片缩小了它的实际概念（至少我时常混淆），或者说，有时借用了其它概念，有时说的又不够全面。

比如在操作系统中，*文件系统*这个术语往往既被用来描述磁盘中的物理布局，比如有时我们说磁盘中的“文件系统”是EXT2或说把磁盘格式化成FAT32格式的“文件系统”等——这时所说的“文件系统”是指磁盘数据的物理布局格式；另外，文件系统也被用来描述内核中的逻辑文件结构，比如有时说的“文件系统”的接口或内核支持Ext2等“文件系统”——这时所说的文件系统都是内存中的数据组织结构而并非磁盘物理布局。还有些时候说“文件系统”负责管理用户读写文件——这时所说的“文件系统”往往描述操作系统中的“文件管理系统”，也就是文件子系统。

虽然上面我们列举了混用文件系统的概念的几种情形，但是却也不能说上述说法就是错误的，因为文件系统概念本身就囊括众多概念，几乎可以说在操作系统中自内存管理、系统调度到I/O系统、设备驱动等各个部分都和文件系统联系密切，有些部分和文件系统甚至未必能明确划分——所以不能只知道文件系统是系统中数据的存储结构，一定要全面认识文件系统在操作系统中的角色，才能具备自己开发新文件系统的能力。

为了澄清文件系统的概念，必须先来看看文件系统在操作系统中处于何种角色，分析文件系统概念的内含外延。所以我们先抛开Linux文件系统的实例，而来看看操作系统中文件系统的普遍体系结构，从而增强对文件系统的理论认识。

***我们针对各层做以简要分析：***

首先我们来分析最低层——*设备驱动层*，该层负责与外设——磁盘等——通讯。基于磁盘的文件系统都需要和存储设备打交道，而系统操作外设离不开驱动程序。所以内核对文件的最后操作行为就是调用设备驱动程序完成从主存（内存）到辅存（磁盘）的数据传输。文件系统相关的多数设备都属于块设备，常见的块设备驱动程序有磁盘驱动，光驱驱动等，之所以称它们为块设备，一个原因是它们读写数据都是成块进行的，但是更重要的原因是它们管理的数据能够被随机访问——不需要向字符设备那样必须顺序访问。

设备驱动层的上一层是*物理**I/O**层*，该层主要作为计算机外部环境和系统的接口，负责系统和磁盘交换数据块。它要知道据块在磁盘中存储位置，也要知道文件数据块在内存缓冲中的位置，另外它不需要了解数据或文件的具体结构。可以看到这层最主要的工作是标识别磁盘扇区和内存缓冲块[[2\]](http://wwww.kerneltravel.net/jiaoliu/002.htm#_ftn2)之间的映射关系。

再上层是*基础**I/O**监督层*，该层主要负责选择文件 I/O需要的设备，调度磁盘请求等工作，另外分配I/O缓冲和磁盘空间也在该层完成。由于块设备需要随机访问数据，而且对速度响应要求较高，所以操作系统不能向对字符设备那样简单、直接地发送读写请求，而必须对读写请求重新优化排序，以能节省磁盘寻址时间，另外也必须对请求提交采取异步调度（尤其写操作）的方式进行。总而言之，内核对必须管理块设备请求，而这项工作正是由该层负责的。

倒数第二层是*逻辑**I/O**层*，该层允许用户和应用程序访问记录。它提供了通用的记录（record）I/O操作，同时还维护基本文件数据。由于为了方便用户操作和管理文件内容，文件内容往往被组织成记录形式，所以操作系统为操作文件记录提供了一个通用逻辑操作层。

和用户最靠近的是*访问方法层*，该层提供了一个从用户空间到文件系统的标准接口，不同的访问方法反映了不同的文件结构，也反映了不同的访问数据和处理数据方法。这一层我们可以简单地理解为文件系统给用户提供的访问接口——不同的文件格式（如顺序存储格式、索引存储格式、索引顺序存储格式和哈希存储格式等）对应不同的文件访问方法。该层要负责将用户对文件结构的操作转化为对记录的操作。

对比上面的层次图我们再来分析一下数据流的处理过程，加深对文件系统的理解。

假如用户或应用程序操作文件（创建/删除），首先需要通过文件系统给用户空间提供的访问方法层进入文件系统，接着由使用逻辑I/O层对记录进行给定操作，然后记录将被转化为文件块，等待和磁盘交互。这里有两点需要考虑——第一，磁盘管理（包括再磁盘空闲区分配文件和组织空闲区）；第二，调度块I/O请求——这些由基础I/O监督层的工作。再下来文件块被物理I/O层传递给磁盘驱动程序，最后磁盘驱动程序真正把数据写入具体的扇区。至此文件操作完毕。

当然上面介绍的层次结构是理想情况下的理论抽象，实际文件系统并非一定要按照上面的层次或结构组织，它们往往简化或合并了某些层的功能（比如Linux文件系统因为所有文件都被看作字节流，所以不存在记录，也就没有必要实现逻辑I/O层，进而也不需要在记录相关的处理）。但是大体上都需要经过类似处理。如果从处理对象上和系统独立性上划分，文件系统体系结构可以被分为两大部分：——文件管理部分和操作系统I/O部分。文件管理系统负责操作内存中文件对象，并按文件的逻辑格式将对文件对象的操作转化成对文件块的操作；而操作系统I/O部分负责内存中的块与物理磁盘中的数据交换。

数据表现形式再文件操作过程中也经历了几种变化：在用户访问文件系统看到的是字节序列，而在字节序列被写入磁盘时看到的是内存中文件块（在缓冲中），在最后将数据写入磁盘扇区时看到的是磁盘数据块[[3\]](http://wwww.kerneltravel.net/jiaoliu/002.htm#_ftn3)。

本文所说的实现文件系统主要针对最开始讲到第二种情况——内核中的逻辑文件结构（但其它相关的文件管理系统和文件系统磁盘存储格式也必须了解），我们用数据处理流图来分析一下逻辑文件系统主要功能和在操作系统中所处的地位。

 I/O调度层 

其中文件系统接口与物理布局管理是逻辑文件系统要负责的主要功能。

*文件系统接口*为用户提供对文件系统的操作，比如open、close、read、write和访问控制等，同时也负责处理文件的逻辑结构。

*物理存储布局管理*，如同虚拟内存地址转化为物理内存地址时，必须处理段页结构一样，逻辑文件结构必须转化到物理磁盘中，所以也要处理物理分区和扇区的实际存储位置，分配磁盘空间和内存中的缓冲也要在这里被处理。

​    所以说要实现文件系统就必须提供上面提到的两种功能，缺一不可。

 

在了解了文件系统的功能后，我们针对Linux操作系统分析具体文件系统如何工作，进而掌握实现一个文件系统需要的步骤。

## Linux 文件系统组成结构

Linux 文件系统的结构除了我们上面所提到的概念结构外，最主要有两个特点，一个是文件系统抽象出了一个通用文件表示层——虚拟文件系统或称做VFS。另外一个重要特点是它的文件系统支持动态安装（或说挂载、登陆等），大多数文件系统都可以作为根文件系统的叶子接点被挂在到根文件目录树下的子目录上。另外Linux系统在文件读写的I/O操作上也采取了一些先进技术和策略。

我们先从虚拟文件系统入手分析linux文件系统的特性，然后介绍有关文件系统的安装、注册和读写等概念。

### 虚拟文件系统

虚拟文件系统为用户空间程序提供了文件系统接口。系统中所有文件系统不但依赖VFS共存，而且也依靠VFS系统协同工作。通过虚拟文件系统我们可以利用标准的UNIX文件系统调用对不同介质上的不同文件系统进行读写操作[[4\]](http://wwww.kerneltravel.net/jiaoliu/002.htm#_ftn4)。

虚拟文件系统的目的是为了屏蔽各种各样不同文件系统的相异操作形式，使得异构的文件系统可以在统一的形式下，以标准化的方法访问、操作。实现虚拟文件系统利用的主要思想是引入一个通用文件模型——该模型抽象出了文件系统的所有基本操作(该通用模型源于Unix风格的文件系统)，比如读、写操作等。同时实际文件系统如果希望利用虚拟文件系统，既被虚拟文件系统支持，也必须将自身的诸如，“打开文件”、“读写文件”等操作行为以及“什么是文件”，“什么是目录”等概念“修饰”成虚拟文件系统所要求的（定义的）形式，这样才能够被虚拟文件系统支持和使用。

我们可以借用面向对象的一些思想来理解虚拟文件系统，虚拟文件系统好比一个抽象类或接口，它定义（但不实现）了文件系统最常见的操作行为。而具体文件系统好比是具体类，它们是特定文件系统的实例。具体文件系统和虚拟文件系统的关系类似具体类继承抽象类或实现接口。而在用户看到或操作的都是抽象类或接口，但实际行为却发生在具体文件系统实例上。至于如何将对虚拟文件系统的操作转化到对具体文件系统的实例，就要通过注册具体文件系统到系统，然后再安装具体文件系统才能实现转化，这点可以想象成面向对象中的多态概念。

我们个实举例来说明具体文件系统如何通过虚拟文件系统协同工作。

例如：假设一个用户输入以下shell命令：

```shell
cp /hda/test1 /removable/test2
```

其中 /removable是MS-DOS磁盘的一个安装点，而 /hda 是一个标准的第二扩展文件系统（ Ext2）的目录。cp命令不用了解test1或test2的具体文件系统，它所看到和操作的对象是VFS。cp首先要从ext3文件系统读出test1文件，然后写入MS-DOS文件系统中的test2。VFS会将找到ext3文件系统实例的读方法，对test1文件进行读取操作；然后找到MS-DOS（在Linux中称VFAT）文件系统实例的写方法，对test2文件进行写入操作。可以看到 VFS是读写操作的统一界面，只要具体文件系统符合VFS所要求的接口，那么就可以毫无障碍地透明通讯了。

## Unix风格的文件系统

虚拟文件系统的通用模型源于*Unix**风格*的文件系统，所谓Unix风格是指Unix传统上文件系统传统上使用了四种和文件系统相关的抽象概念：文件(file)、目录项(dentry)、索引节点(inode)和安装点(mount point)。

*文件**—*—在Unix中的文件都被看做是一有序字节串，它们都有一个方便用户或系统识别的名称。另外典型的文件操作有读、写、创建和删除等。

*目录项*——不要和目录概念搞混淆，在Linux中目录被看作文件。而目录项是文件路径中的一部分。一个文件路径的例子是“/home/wolfman/foo”——根目录是/，目录home,wolfman和文件foo都是目录项。

*索引节点*——Unix系统将文件的相关信息（如访问控制权限、大小、拥有者、创建时间等等信息），有时被称作文件的*元数据*（也就是说，数据的数据）被存储在一个单独的数据结构中，该结构被称为*索引节点*(inode)。

*安装点*——在Unix中，文件系统被安装在一个特定的安装点上，所有的已安装文件系统都作为根文件系统树中的叶子出现在系统中。

上述概念是Unix文件系统的逻辑数据结构，但相应的Unix文件系统（Ext2等）磁盘布局也实现了部分上述概念，比如文件信息（文件数据元）存储在磁盘块中的索引节点上。当文件被载如内存时，内核需要使用磁盘块中的索引点来装配内存中的索引接点。类似行为还有超级块信息等。

对于非Unix风格文件系统，如FAT或NTFS，要想能被VFS支持，它们的文件系统代码必须提供这些概念的虚拟形式。比如，即使一个文件系统不支持索引节点，它也必须在内存中装配起索引节点结构体——如同本身固有一样。或者，如果一个文件系统将目录看作是一种特殊对象，那么要想使用VFS，必须将目录重新表示为文件形式。通常，这种转换需要在使用现场引入一些特殊处理，使得非Unix文件系统能够兼容Unix文件系统的使用规则和满足VFS的需求。通过这些处理，非Unix文件系统便可以和VFS一同工作了，是性能上多少会受一些影响[[5\]](http://wwww.kerneltravel.net/jiaoliu/002.htm#_ftn5)。这点很重要，我们实现自己文件系统时必须提供（模拟）Unix风格文件系统的抽象概念。

### Linux文件系统中使用的对象

Linux文件系统的对象就是指一些数据结构体，之所以称它们是对象，是因为这些数据结构体不但包含了相关属性而且还包含了操作自身结构的函数指针，这种将数据和方法进行封装的思想和面向对象中对象概念一致，所以这里我们就称它们是对象。

Linux文件系统使用大量对象，我们简要分析以下VFS相关的对象，和除此还有和进程相关的一些其它对象。

#### VFS相关对象

这里我们不展开讨论每个对象，仅仅是为了内容完整性，做作简要说明。

**VFS**中包含有四个主要的对象类型，它们分别是：

超级块对象，它代表特定的已安装文件系统。

*索引节点*对象，它代表特定文件。

*目录项*对象，它代表特定的目录项。

*文件*对象，它代表和进程打开的文件。

每个主要对象中都包含一个操作对象，这些操作对象描述了内核针对主要对象可以使用的方法。最主要的几种操作对象如下：

super_operations对象，其中包括内核针对特定文件系统所能调用的方法，比如read_inode()和sync_fs()方法等。

inode_operations对象，其中包括内核针对特定文件所能调用的方法，比如create()和link()方法等。

dentry_operations对象，其中包括内核针对特定目录所能调用的方法，比如d_compare()和d_delete()方法等。

*file*对象，其中包括，进程针对已打开文件所能调用的方法，比如read()和write()方法等。

除了上述的四个主要对象外,VFS还包含了许多对象，比如每个注册文件系统都是由file_system_type对象表示——描述了文件系统及其能力（如比如ext3或XFS）；另外每一个安装点也都利用vfsmount对象表示——包含了关于安装点的信息，如位置和安装标志等。

#### 其它VFS对象

系统上的每一进程都有自己的打开文件，根文件系统，当前工作目录，安装点等等。另外还有几个数据结构体将VFS层和文件的进程紧密联系，它们分别是：file_struct 和fs_struct

file_struct结构体由进程描述符中的files项指向。所有包含进程的信息和它的文件描述符都包含在其中。第二个和进程相关的结构体是fs_struct。该结构由进程描述符的fs项指向。它包含文件系统和进程相关的信息。每种结构体的详细信息不在这里说明了。 

#### 缓存对象

除了上述一些结构外，为了缩短文件操作响应时间，提高系统性能，Linux系统采用了许多缓存对象，例如目录缓存、页面缓存和缓冲缓存（已经归入了页面缓存），这里我们对缓存做简单介绍。

*页高速缓存*（cache）是 Linux内核实现的一种主要磁盘缓存。其目的是减少磁盘的I/O操作，具体的讲是通过把磁盘中的数据缓存到物理内存中去，把对磁盘的I/O操作变为对物理内存的I/O操作。页高速缓存是由RAM中的物理页组成的，缓存中每一页都对应着磁盘中的多个块。每当内核开始执行一个页I/O操作时（通常是对普通文件中页大小的块进行磁盘操作），首先会检查需要的数据是否在高速缓存中，如果在，那么内核就直接使用高速缓存中的数据，从而避免了访问磁盘。

但我们知道文件系统只能以每次访问数个块的形式进行操作。内核执行所有磁盘操作都必须根据块进行，一个块包含一个或多个磁盘扇区。为此，内核提供了一个专门结构来管理缓冲buffer_head。缓冲头[[6\]](http://wwww.kerneltravel.net/jiaoliu/002.htm#_ftn6)的目的是描述磁盘扇区和物理缓冲之间的映射关系和做I/O操作的容器。但是缓冲结构并非独立存在，而是被包含在页高速缓存中，而且一个页高速缓存可以包含多个缓冲。我们将在文件后面的文件读写部分看到数据如何被从磁盘扇区读入页高速缓存中的缓冲中的。

  

## 文件系统的注册和安装

使用文件系统前必须对文件系统进行注册和安装，下面分别对这两种行为做简要介绍。

### 文件系统的注册

VFS要想能将自己定义的接口映射到实际文件系统的专用方法上，必须能够让内核识别实际的文件系统，实际文件系统通过将代表自身属性的文件类型对象(file_system_type)注册(通过register_filesystem()函数)到内核，也就是挂到内核中的文件系统类型链表上，来达到使文件系统能被内核识别的目的。反过来内核也正是通过这条链表来跟踪系统所支持的各种文件系统的。

我们简要分析一下注册步骤：

```c
struct file_system_type {

    const char *name;                   /*文件系统的名字*/

  int fs_flags;                       /*文件系统类型标志*/

/*下面的函数用来从磁盘中读取超级块*/

    struct super_block * (*read_super) (struct file_system_type *, int,

                  const char *, void *);

    struct file_system_type * next;          /*链表中下一个文件系统类型*/

    struct list_head fs_supers;             /*超级块对象链表*/

};
```

其中最重要的一项是read_super()函数，它用来从磁盘上读取超级块，并且当文件系统被装载时，在内存中组装超级块对象。要实现一个文件系统首先需要实现的结构体便是file_system_type结构体。

注册文件系统只能保证文件系统能被系统识别，但此刻文件系统尚不能使用，因为它还没有被安装到特定的安装点上。所以在使用文件系统前必须将文件系统安装到安装点上。

文件系统被实际安装时，将在安装点创建一个vfsmount结构体。该结构体用代表文件系统的实例——换句话说，代表一个安装点。

vfsmount结构被定义在<linux/mount.h>中，下面是具体结构

```c
struct vfsmount

{

    struct list_head mnt_hash;      /*哈希表*/

    struct vfsmount *mnt_parent;       /*父文件系统*/

    struct dentry *mnt_mountpoint;  /*安装点的目录项对象*/

    struct dentry *mnt_root;           /*该文件系统的根目录项对象*/

    struct super_block *mnt_sb;    /*该文件系统的超级块*/

    struct list_head mnt_mounts;    /*子文件系统链表*/

    struct list_head mnt_child;      /*和父文件系统相关的子文件系统*/

    atomic_t mnt_count;          /*使用计数*/

    int mnt_flags;            /*安装标志*/

    char *mnt_devname;      /*设备文件名字*/

    struct list_head mnt_list;       /*描述符链表*/

};
```

文件系统如果仅仅注册，那么还不能被用户使用。要想使用它还必须将文件系统安装到特定的安装点后才能工作。下面我们接着介绍文件系统的安装[[7\]](http://wwww.kerneltravel.net/jiaoliu/002.htm#_ftn7)过程。

### 安装过程 

用户在用户空间调用mount()命令——指定安装点、安装的设备、安装类型等——安装指定文件系统到指定目录。mount()系统调用在内核中的实现函数为sys_mount(),该函数调用的主要例程是do_mount()，它会取得安装点的目录项对象，然后调用do_add_mount()例程。

  do_add_mount()函数主要做的是首先使用do_kern_mount()函数创建一个安装点，再使用graft_tree（）将安装点作为叶子与根目录树挂接起来。

  整个安装过程中最核心的函数就是do_kern_mount()了，为了创建一个新安装点（vfsmount）,该函数需要做一下几件事情：

​     1   检查安装设备的权利，只有root权限才有能力执行该操作。

​     2   Get_fs_type()在文件链表中取得相应文件系统类型（注册时被填加到练表中）。

​     3   Alloc_vfsmnt()调用slab分配器为vfsmount结构体分配存储空间，并把它的地址存放在mnt局部变量中。

​     4   初始化mnt->mnt_devname域

​     5   分配新的超级块并初始化它。do_kern_mount( )检查file_system_type描述符中的标志以决定如何进行如下操作：根据文件系统的标志位，选择相应的方法读取超级块(比如对Ext2,romfs这类文件系统调用get_sb_dev()；对于这种没有实际设备的虚拟文件系统如 ramfs调用get_sb_nodev())——读取超级块最终要使用文件系统类型中的read_super方法。

  安装过程做的最主要工作是创建安装点对象，挂接给定文件系统到根文件系统的指定接点下，然后初始化超级快对象，从而获得文件系统基本信息和相关操作方法(比如读取系统中某个inode的方法)。

 

总而言之，注册过程是告之内核给定文件系统存在于系统内；而安装是请求内核对给定文件系统进行支持，使文件系统真正可用。

​                   

 *<**待续* *…**..>*

 