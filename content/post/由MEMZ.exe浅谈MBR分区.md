---
title: 由MEMZ.exe浅谈MBR分区
date: 2025-03-29T09:44:37+08:00
categories:
  - Malware analysis
tags:
  - malware
---
初中的时候很喜欢在B站看彩虹猫病毒的视频。视频内容要么是“彩虹猫在xx系统运行”，要么就是"彩虹猫大战各种杀软"，实在没活了之后又能换款病毒像前两种继续水视频，基本上没有太多技术含量而言。如今也想不明白自己当初为什么会着迷这种视频。

关于第一篇博客的题材想了许久，干脆就重拾当时的“兴趣”，尝试做些技术面分析。
## 0x01 MBR分区介绍
**MBR（Master Boot Record）**，中文翻译为**主引导记录**、主引导扇区，位于磁盘的首个扇区（LBA 0）。计算机开机启动时，BIOS会首先读取MBR，之后完成一系列开机操作MEMZ.exe就是通过修改MBR分区，从而实现开机时播放彩虹猫动画，并操控蜂鸣器播放音乐。

MBR占用一个扇区，即512个字节。在该扇区中，前446字节称作**主引导例程（Master Boot Routine）**，随后的64个字节为**分区表（Disk Partition Table, DPT）**，最后两个字节为结束标识符，值为55AAH。

MBR的[检查代码](https://thestarman.pcministry.com/asm/mbr/STDMBR.htm)有三个版本，他们分别是：
- 标准版本，历经MS-DOS 3.30至Windows 95版本；
- 第二个版本，适用于Windows 95B, 98, 98SE, ME版本；
- 第三个版本，适用于Windows 7, 8/8.1, 10版本。

当然Linux和macOS也支持或曾经支持MBR。
### 主引导例程
不同版本的主引导例程随着不同版本的检查代码而有些许差异，但大体结构不变。
![image.png](https://cdn.jsdelivr.net/gh/studyHard282/imgHost@main/blog/20250325165014.png)

以标准版本为例，绿色部分为可执行代码，随后的紫色部分为报错信息。红色和黄色部分分别为分区表和结束标识符，当然他们不在主引导例程的范围之内了。若读者需要进一步学习，可阅读MBR检查代码里的opcode。
### 分区表
分区表占据 $4\times16$ 个字节。每个分区的信息占据16字节，由此可以看出MBR型最多能划分4个主分区（或者3个主分区+1个扩展分区）。每个分区占据的16字节[规划](https://thestarman.pcministry.com/asm/mbr/PartTables.htm)如下。

| 偏移量  | 长度（字节） | 内容与含义                                    |
| ---- | ------ | ---------------------------------------- |
| 0x00 | 1      | 引导标志。0x80表示活动分区，0x00表示非活动分区。             |
| 0x01 | 3      | 起始CHS地址。三个字节分别表示分区的起始柱面（C）、磁头（H）、扇区（S）。  |
| 0x04 | 1      | 分区类型标志位。                                 |
| 0x05 | 3      | 结束CHS地址。三个字节分别表示分区的起始柱面（C）、磁头（H）、扇区（S）。  |
| 0x08 | 4      | 起始LBA。记录从磁盘的第一个扇区（LBA 0）到当前分区第一个扇区的扇区数量。 |
| 0x0C | 4      | 本分区的总扇区数。                                |
## 0x02 MEMZ.exe修改MBR分区
MEMZ.exe中覆写MBR分区的代码部分如图所示。
![image.png](https://cdn.jsdelivr.net/gh/studyHard282/imgHost@main/blog/20250326232845.png)
样本首先以读写方式打开PhysicalDrive0（通常为主硬盘），之后分配一个大小为64KB（0x10000字节）的内存缓冲区。随后向内存缓冲区中写入byte_402118和byte_402248的内容。

byte_402118的内容如下图所示。
![image.png](https://cdn.jsdelivr.net/gh/studyHard282/imgHost@main/blog/20250327154217.png)
byte_402118长度为303字节，从缓冲区首个位置开始写入，很显然其目的是覆盖主引导例程部分。此部分的opcode主要用于为播放彩虹猫动画和音乐做准备。

byte_402248篇幅较长，若读者感兴趣可自行分析。其前两个字节的内容如下图所示。
![image.png](https://cdn.jsdelivr.net/gh/studyHard282/imgHost@main/blog/20250327160758.png)
byte_402248偏移了510个字节后向内存写入，而其开头的两个字节的内容为55h，AAh，很显然是为了补足MBR结束标识符，使得MBR的检查代码能够成功识别MBR。随后的内容为显示彩虹猫动画、播放音乐、阻止系统响应输入等。
## 0x03 更好的替代品
20世纪90年代末期，Intel开发了一种新的分区表格式，随后成为了UEFI的一部分。[GPT](https://uefi.org/specs/UEFI/2.11/05_GUID_Partition_Table_Format.html)也随UEFI而诞生。**GPT（GUID Partition Table）**，中文翻译为**全局唯一标识分区表**，相较于MBR有了更多优势。
### 可支持的硬盘容量更大
受限于MBR的分区表长度限制，即上文提到的仅有4个字节（32位）存储LBA值，此时MBR可支持的最多扇区数为 $2^{32}−1=4,294,967,295$。扇区大小按照512字节计算，则MBR可支持的硬盘容量大小为 $4,294,967,295\times512=2,199,023,255,040$ 个字节，即大多数资料所描述的2.2TB。而GPT分区使用了8个字节（64位）存储LBA值，可支持的扇区数目就是MBR的 $2^{32}$ 倍。此时，可支持的硬盘容量受限的瓶颈在于操作系统、文件系统等等软件的限制了（当然现实中也很难造出如此大容量的硬盘）。
### 可支持的分区更多
先前提到MBR的分区表大小为64个字节，按照设计每个分区的信息占据16字节，因此MBR最多只能划分4个主分区。而GPT的分区表是动态扩展的，理论上对分区数目没有限制。但一些操作系统（例如[微软](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions#partition-requirements)）默认GPT最多可支持128个分区。当然现实中需要用到128个分区以上的情况也是极少数的。

除此之外，GPT分区相较于MBR还有提供备份分区表、使用CRC32保证完整性等等优势。随着时间的车轮滚滚向前，MBR与BIOS两兄弟在GPT与UEFI的光芒下显得黯然失色。即使保持着兼容性，如今的操作系统也逐渐不认可MBR分区了。
GPT分区的LBA 0内容为传统的MBR（Legacy Master Boot Record）或保护性MBR（Protective MBR），这意味着MEMZ.exe仍然可以破坏这些地方。但可能UEFI不再会识别MEMZ.exe覆写MBR的代码，计算机感染后开机也不会播放彩虹猫动画和音乐。如今MEMZ.exe也已经迭代了[更安全的版本](https://www.youtube.com/watch?v=C19JfFKAogE)，可以在不破坏电脑的情况下重温病毒感染时的动画效果。