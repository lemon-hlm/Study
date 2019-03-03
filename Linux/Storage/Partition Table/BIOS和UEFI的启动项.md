https://zhuanlan.zhihu.com/p/31365115

1. 传统BIOS
2. UEFI
3. UEFI的DXE与磁盘上的文件
4. EFI系统分区
5. Windows 8/8.1/10在UEFI和BIOS下，各种启动文件的顺序
    5.1 BIOS下

    5.2 UEFI下

6. 问题
    
    6.1 以前我一直装Ghost版的Windows，UEFI之后真的没法Ghost了么？
    
    6.2 Windows无法定位现有分区，也无法。。。。。
    
    6.3 不需要第三方工具就能做UEFI下的Windows安装盘？
    
    6.4 我电脑是UEFI的，想装Linux，但我手头没优盘，听说也能搞定？
    
    6.5 装个Linux，但我希望默认还是Windows；重装Windows，可是我开机不再默认Grub，怎么回Linux？
    
    6.6 重装Windows，提示我什么MBR、GPT，不让装？

## 1. 传统BIOS

一句话概括：**BIOS只认识设备，不认识分区、不认识文件**。

BIOS启动的时候，按照CMOS设置里的顺序，挨个存储设备看：（此处不讨论PXE和光盘）

- 这个存储设备的前512字节是不是以0x55 0xAA结尾？

- 不是，那就跳过。找下一个设备。

- 是的话，嗯，这个磁盘可以启动，**加载这512字节里的代码，然后执行**。

- **执行之后，后面的事，几乎就跟BIOS没啥关系（那些代码可能会使用BIOS的一些中断**）了。

至于后面启动什么系统，取决于这512字节里存了谁家的代码。这个代码是各家的系统安装程序写进去的，目的是启动自家系统。

- 比如你装（或者重装）了Windows，这里面就变成了Windows的启动代码。
- 比如你装（或者重装）了Linux，这里面就会变成Grub的启动代码。

顺便这512字节包含了MBR分区表的信息。**但是有人可能注意到，上面半句没提“系统装在哪个分区上了”，硬盘有几个分区**。

其实**BIOS并不认识分区表（不关心，BIOS开始执行512字节代码就不管了）**。哪怕磁盘上没有分区表，没分过区，只要前512字节有0x55 0xAA的结尾，有合适的引导代码，也是能启动的。

## 2. UEFI

（此处只讨论民用64位架构下的UEFI。）

一句话概括，**UEFI认识设备，还认识设备ROM，还认识分区表、认识文件系统以及文件**。

UEFI启动的时候，经过一系列初始化（SEC、CAR、DXE什么的，SEC、CAR你们不需要懂。下一节里会说DXE阶段是干嘛的）然后按照设置里的顺序，找启动项。**启动项分两种，设备启动项和文件启动项**：

- **文件启动项**，大约记录的是**某个磁盘的某个分区的某个路径下的某个文件**。对于文件启动项，固件会直接加载这个**EFI文件**，并执行。类似于DOS下你敲了个win.com就执行了Windows 3.2/95/98的启动。文件不存在则失败。

- **设备启动项**，大约记录的就是“某个U盘”、“某个硬盘”。（此处只讨论U盘、硬盘）对于设备启动项，UEFI标准规定了默认的路径“\EFI\Boot\bootX64.efi”。UEFI会加载**磁盘**上的**这个文件（\EFI\Boot\bootX64.efi）**。文件不存在则失败。

至于这个EFI文件会干嘛，主板是不管的。

但是随着Windows8.x，以及UEFI标准2.x，推出了一个叫做SecureBoot的功能。开了SecureBoot之后，主板会验证即将加载的efi文件的签名，如果开发者不是受信任的开发者，就会拒绝加载。

比如CloverX64.efi就好像没有签名。

## 3. UEFI的DXE与磁盘上的文件

一个磁盘**分区**，要**格式化**之后，才能往里存**文件**，格式化的时候，又能选择不同的**文件系统**。比如Win10可以选FAT32、NTFS、exFAT、ReFS几种，Linux可以选ext2、ext3、ext4、FAT32等，macOS可以选FAT32、HFS+、APFS、exFAT几种。

其实**每个操作系统**，都先有**文件系统驱动**，然后才能读取某种**文件系统**。

比如Windows带了上述四种文件系统等驱动，Linux带了FAT32、ext等文件系统等驱动，macOS带了上述四种驱动以及NTFS等只读驱动。

**设备**也是一样的，先有**设备驱动**，然后才能**读取设备**。

原版的WinXP只带了IDE驱动，没有SATA驱动；原版Win7只带了IDE和SATA驱动，没带NVMe驱动；Win8/Win10则带了IDE/SATA和NVMe三种驱动；macOS10.12带了SATA驱动以及苹果专用NVMe磁盘的驱动；macOS10.13带了SATA和标准NVMe驱动。

UEFI作为一个模糊了**固件**和**操作系统界限**的东西，作为一个设计之初就考虑到了扩展性的东西，它也是**有驱动程序**的。启动过程中的**DXE阶段**，全称叫Driver eXecution Environment，就是**加载驱动**用的。

首先各种**PCI-E的设备**，比如显卡，比如PCI-E的NVMe固态硬盘，都有**固件**。其中**支持UEFI**的设备，比如10系列的Nvidia显卡，**固件里就会有对应的UEFI的驱动**。

UEFI启动后，**进入了DXE阶段，就开始加载设备驱动（这里是设备驱动），然后UEFI就会有设备列表**了。

对于其中的磁盘，**UEFI会加载对应的驱动解析其中的分区表（GPT和MBR）**。然后UEFI就会有**所有分区的列表**了。然后UEFI就会用**内置的文件系统驱动（这里是文件系统驱动）**，解析每个**分区**。然后UEFI就会**认识分区里的文件**了。比如“\EFI\Boot\bootX64.efi”。

作为UEFI标准里，钦定的文件系统，**FAT32.efi是每个主板都会带的**。**所有UEFI的主板都认识FAT32分区**。这就是UEFI的Windows安装盘为啥非得是FAT32的。除此之外，苹果的主板还会支持hfs分区。如果某天Linus Torvalds推出了主板，我猜这主板一定会带EXT4.efi，哈哈哈哈哈。

如同Windows可以安装驱动一样，**UEFI也能在后期加载驱动**。比如CloverX64.efi启动之后，会加载\EFI\Clover\drivers64UEFI下的所有驱动。包括VboxHFS.efi等各种efi。网上你也能搜到NTFS.efi。再比如，**UEFIShell下，你可以手动执行命令加载驱动**。

## 4. EFI系统分区

UEFI规范里，在**GPT分区表**的基础上，规定了一个**EFI系统分区（EFI System Partition，ESP），ESP要格式化成FAT32（这个分区是要格式化为某个文件系统的），EFI启动文件要放在“\EFI\<厂商>”文件夹下面**。

- 比如Windows的UEFI启动文件，都在“\EFI\Microsoft”下面。
- 比如Clover的东西，全都放在“\EFI\Clover”下面。

但是Apple比较特殊，它的主板直接去HFS/APFS分区找启动文件。然而即便如此，Mac的ESP里还是会有一堆Apple的文件。

Macbook上的ESP分区里的“\EFI\Apple”文件夹：

![config](images/12.jpg)

**UEFI下，启动盘是ESP分区，跟Windows不是同一个分区。**

根据UEFI标准里说的，你可以把U盘里的“\EFI\Clover”文件夹，拷贝到**硬盘里的ESP对应的路径**下。然后把“\EFI\Clover\CloverX64.efi”添加为**UEFI的文件启动项**就行了。

Windows的BCD命令，其实也可以添加UEFI启动项，然而我没搞懂怎么弄。我更喜欢用EasyUEFI来搞这些操作。但是免费版的EasyUEFI不支持企业版Windows哦～某些Win10用户要被拒之门外了。

这一节的最后，再说说“\EFI\Boot”这个文件夹。这个文件夹，放谁家的程序都行。无论是“\EFI\Microsoft\Boot\Bootmgfw.efi”，还是“\EFI\Clover\CloverX64.efi”，只要**放到“\EFI\Boot”下并且改名“bootX64.efi”（设备启动项）**，就能在**没添加文件启动项**的情况下，**默认加载对应的系统**。

举个例子：**一个U盘，你想做成Windows安装盘+Hackintosh安装盘**，该怎么做？

- 你划分俩分区，第一个分区格式化成FAT32，第二个分区HFS+。
- 苹果系统下把第二个分区做成安装盘。苹果启动盘做好了。
- 把Windows的ISO镜像里的文件拷贝到第一个分区。Windows安装盘做好了。
- 然后Clover拷贝到第一个分区的“\EFI\Clover”文件夹下。Clover的东西也做好了。
- 最后，怎么让这个U盘插到任何电脑上都默认启动Clover呢？答案是，把“\EFI\Boot”下的“bootX64.efi”换成Clover的就可以了。那个文件夹放谁家的efi文件，都要改名“bootX64.efi”哦。

嗯，这一节就写到这吧。

## 5. Windows 8/8.1/10在UEFI和BIOS下，各种启动文件的顺序

### 5.1 BIOS下

MBR->PBR->bootmgr->WinLoad.exe

![config](images/13.jpg)

按照前文说的，**BIOS加载某个磁盘MBR的启动代码**，这里特指Windows的引导代码，这段代码会查找活动分区（**BIOS不认识活动分区，但这段代码认识活动分区，0x80**）的位置，加载并执行活动分区的PBR（另一段引导程序）。

Windows的PBR认识FAT32和NTFS两种分区，找到分区根目录的bootmgr文件，加载、执行bootmgr。

bootmgr没了MBR和PBR的大小限制，可以做更多的事。它会加载并分析BCD启动项存储。而且bootmgr可以跨越磁盘读取文件了。所以无论你有几个磁盘，你在多少块磁盘上装了Windows，一个电脑只需要一个bootmgr就行了。bootmgr会去加载某磁盘某NTFS分区的“\Windows\System32\WinLoad.exe”，后面启动Windows的事就由WinLoad.exe来完成了。

重点来了，为什么图中有两组虚线？

**因为“启动磁盘”和“装系统的磁盘”可以是同一个磁盘，也可以不是同一个**。“启动分区”和“系统分区”可以是不同磁盘的不同分区，也可以是相同磁盘的不同分区，也可以是同一个分区。

这就解释了，为什么，有的时候，Windows装在磁盘2上，却要在BIOS里选磁盘0启动了。因为bootmgr可能在磁盘0上。

### 5.2 UEFI下

UEFI固件->bootmgfw.efi->WinLoad.efi

![config](images/14.jpg)

根据前文说的，**UEFI启动项分为文件启动项和设备启动项**。

给UEFI指定特定的文件启动项（**某个设备某个分区某个文件，但是必须手动加载设备驱动和文件系统驱动**）或直接指定特定的设备启动项（该设备必须存在FAT分区，并且在里面必须是\EFI\Boot\bootX64.efi）。

UEFI查找**硬盘分区**中第一个**FAT分区**内的引导文件进行系统引导。

通常情况：主板UEFI初始化，然后找到了默认启动项“Windows Boot Manager”。里面写了bootmgfw.efi的位置。固件加载bootmgfw.efi（**ESP，EFI系统分区，FAT32文件系统**）。bootmgfw.efi根据**BCD启动项存储**，**找到装Windows的磁盘的具体分区**，加载其中的WinLoad.efi。由WinLoad.efi完成剩下的启动工作。

其中的虚线，跟上面的一样，意思是，**Windows启动盘和EFI启动盘，可以是一个硬盘，也可以是不同的硬盘**。所以**对于UEFI来说，启动盘是bootmgfw.efi所在的那个盘**。

## 6. 问题

### 6.1 以前我一直装Ghost版的Windows，UEFI之后真的没法Ghost了么？

先说一句，真不推荐用网上的Ghost版Windows安装盘来装系统了。微软公开放出了官方的原版Win10下载链接，而且还有启动盘制作程序。链接在这：[下载 Windows 10](https://link.zhihu.com/?target=https%3A//www.microsoft.com/zh-cn/software-download/windows10)。写这个的原因，是因为，有时候，自己做的Ghost备份，还是挺好用的。

并不是不能Ghost了。但是**传统的Ghost盘，都是只Clone了C盘**，没有考虑到“**UEFI下，启动盘是ESP分区（FAT32文件系统），跟Windows不是同一个分区**”的事。

其次，Ghost备份，并**不能备份分区的GUID**。还原之后，ESP分区里的BCD中记录的Windows在“某某GUID的分区上”，就可能找不到对应的GUID了。这时候，需要用bcdedit命令，或者BCDBoot命令，修改BCD存储。鉴于目前的Ghost盘，很少基于DOS了，如果是基于WinPE的，bcdedit命令和bcdboot命令都是已经内置了的。只要制作者在批处理文件里，在Ghost之后，调用bcdedit命令改一下bcd配置就行了。

而且，即使没Ghost备份ESP分区，你依然可以用**bcdboot命令来生成ESP分区的内容**。同样，在WinPE下，批处理文件里，Ghost还原之后，使用BCDBoot命令生成启动文件就行了。

总结一下，Ghost还原Windows分区之后，调用BCDBoot配置启动项即可。

### 6.2 Windows无法定位现有分区，也无法。。。。。

这种报错信息，如果是**在UEFI模式下，一般是因为，你有多块硬盘，而且超过一块硬盘上，有ESP分区**。只要把不想用的ESP分区删掉，或者拔掉对应的硬盘，保证装Windows的时候，只有一个硬盘上有ESP分区，即可。

如果实在做不到，考虑用DISM.exe安装Windows吧。Win7的DISM.exe真的太弱了。尽量用Win10安装盘或者Win10PE里的DISM.exe。

### 6.3 不需要第三方工具就能做UEFI下的Windows安装盘？

确实啊，根据上文说的，**U盘，格式化成FAT32**，然后把Windows安装盘的ISO里面的东西拷贝到U盘就行了。（适用于Win8/8.1/10以及WinServer2012/2012R2/2016。WinVista x64/Win7x64以及WinServer2008x64/2008R2需要额外操作，WinVista x86/Win7x86/WinServer2008x86不支持UEFI）

打开ISO之后，是这四个文件夹、四个文件，拷贝到优盘。

![config](images/15.jpg)

### 6.4 我电脑是UEFI的，想装Linux，但我手头没优盘，听说也能搞定？

对啊对啊，搞个FAT32的分区，把Linux安装盘的iso镜像里面的文件拷贝进去，然后在Windows下，用工具给那个分区的BOOTx64.efi，添加为UEFI文件启动项，开机时候选那个启动项，就能启动到Linux安装盘了。下面示意图是Ubuntu的。记得查看一下你的Linux支不支持SecureBoot哦！如果你要装的Linux不支持SecureBoot，记得**关掉主板的SecureBoot设置**哦。

Ubuntu安装盘ISO镜像内的UEFI启动文件：

![config](images/16.jpg)

### 6.5 装个Linux，但我希望默认还是Windows；重装Windows，可是我开机不再默认Grub，怎么回Linux？

如果是UEFI模式，那就跟之前一样，改启动项顺序就行了。

如果是传统BIOS的，要么用bootsect.exe把MBR改成Windows的。要么用工具把MBR刷成Grub的。也可以考虑Linux下用dd命令备份MBR的前446字节，到时候再还原回去。

### 6.6 重装Windows，提示我什么MBR、GPT，不让装？

这个就是Windows安装程序的限制了。BIOS模式下的Windows只允许被装在MBR分区表下面。UEFI模式下的Windows只允许被装在GPT分区下。但事实上，MBR分区表，也能启动UEFI模式下的Windows。