# 系统相关

## 内核

查看内核信息

```shell
cat /etc/redhat-release

uname -r
```

## cpu

```shell
cat /proc/cpuinfo |more
```

关于该文件输出项含义，如下表：

| 输出项          | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| processor       | 系统中逻辑处理核的编号。对于单核处理器，则可认为是其CPU编号，对于多核处理器则可以是物理核、或者使用超线程技术虚拟的逻辑核 |
| vendor_id       | CPU制造商                                                    |
| cpu family      | CPU产品系列代号                                              |
| model           | CPU属于其系列中的哪一代的代号                                |
| model name      | CPU属于的名字及其编号、标称主频                              |
| stepping        | CPU属于制作更新版本                                          |
| cpu MHz         | CPU的实际使用主频                                            |
| cache size      | CPU二级缓存大小                                              |
| physical id     | 单个CPU的标号                                                |
| siblings        | 单个CPU逻辑物理核数                                          |
| core id         | 当前物理核在其所处CPU中的编号，这个编号不一定连续            |
| cpu cores       | 该逻辑核所处CPU的物理核数                                    |
| apicid          | 用来区分不同逻辑核的编号，系统中每个逻辑核的此编号必然不同，此编号不一定连续 |
| fpu             | 是否具有浮点运算单元（Floating Point Unit）                  |
| fpu_exception   | 是否支持浮点计算异常                                         |
| cpuid level     | 执行cpuid指令前，eax寄存器中的值，根据不同的值cpuid指令会返回不同的内容 |
| wp              | 表明当前CPU是否在内核态支持对用户空间的写保护（Write Protection） |
| flags           | 当前CPU支持的功能                                            |
| bogomips        | 在系统内核启动时粗略测算的CPU速度（Million Instructions Per Second） |
| clflush size    | 每次刷新缓存的大小单位                                       |
| cache_alignment | 缓存地址对齐单位                                             |
| address sizes   | 可访问地址空间位数                                           |

### cpu 个数

```shell
cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l
```

### cpu核心数

```shell
cat /proc/cpuinfo | grep "cpu cores" | uniq
```

## 内存

### free

free命令是一个快速查看内存使用情况的方法，它是对 /proc/meminfo 收集到的信息的一个概述。

#### 命令格式

```shell
free [选项]
```

#### 主要参数

- -h：通过调整 KB、GB 等数据单位，以人类可读的形式打印信息。
- -s：在给定的时间间隔后更新 free 输出。
- -t： 显示系统和交换内存的总量
- -g：以 GB 为单位显示数据。
- -m： 以 MB 为单位打印信息。
- -k： 以 KB 为单位显示输出。

#### 常见用法

可以使用 `-s` 标志以特定时间间隔刷新统计信息：

```shell
free -s <秒>
```

## 磁盘

Linux 磁盘管理常用三个命令为 **df**、**du** 和 **fdisk**。

- **df**（英文全称：disk free）：列出文件系统的整体磁盘使用量
- **du**（英文全称：disk used）：检查磁盘空间使用量
- **fdisk**：用于磁盘分区

### df

检查文件系统的磁盘空间占用情况。可以利用该命令来获取硬盘被占用了多少空间，目前还剩下多少空间等信息。

#### 命令格式

```shell
df [-ahikHTm] [目录或文件名]
```

#### 主要参数

- `-h`：以人类可读的方式显示输出结果（例如，使用 KB、MB、GB 等单位）。
- `-T`：显示文件系统的类型。
- `-t <文件系统类型>`：只显示指定类型的文件系统。
- `-i`：显示 inode 使用情况。
- `-H`：该参数是 `-h` 的变体，但是使用 1000 字节作为基本单位而不是 1024 字节。这意味着它会以 SI（国际单位制）单位（例如 MB、GB）而不是二进制单位（例如 MiB、GiB）来显示磁盘使用情况。
- `-k`：这个选项会以 KB 作为单位显示磁盘空间使用情况。
- `-a`：该参数将显示所有的文件系统，包括虚拟文件系统，例如 `proc`、`sysfs` 等。如果没有使用该选项，默认情况下，`df` 命令不会显示虚拟文件系统。

#### 常见用法

```shell
df -h
```



# 文件与文件夹

## pwd

显示当前工作目录的路径

## cat

用于查看文件内容（通常是一个文本文件），后跟文件名称。

```shell
# 查看openfire.xml文件
cat openfire.xml
```

### cat -n (显示行号)

在每一行前显示行号

```shell
# 查看openfire.xml显示文件的行号
cat -n openfire.xml
```

## head

显示文件的开头。 -n参数：指定显示的行数

```shell
# 查看openfire.xml的前5行
head -5 openfire.xml
```

## tail

显示文件的结尾。 -n参数：指定显示的行数

```shell
# 查看openfire.xml的后3行
tail -3 openfire.xml
```

### tail -f 

循环读取

```shell
# 循环读取 filename.log 文件
tail -f filename.log
```

## more

一页一页地显示文件内容，后跟文件名称 【**空格键**向下翻页，**Enter键**向下滚动一行，**Q键**退出】

``` shell
# 通过more命令查看openfire.xml
more openfire.xml
```

## less

逐页显示文本文件内容。在 `less` 环境下，可以使用方向键或 Page Up/Page Down 键来滚动浏览文件。按 `q` 键可以退出 `less`。

`less` 不必一次性读取整个文件。这对于大文件非常有用，因为用户可以立即开始浏览文件，而不需要等待文件完全加载。

```shell
less openfire.xml
```

## 新建文件夹

```shell
mkdir <folderName>
```

## 删除文件夹

```shell
rm -rf <folderName>
```

## 删除空文件夹

```shell
rmdir <folderName>
```

注意，该命令只能删除空文件夹，如果文件夹不为空，则报错：Directory not empty

## cp：复制

```shell
cp source_file destination
# 递归复制目录及其内容
cp -r source_directory destination  
```

## mv：移动或重命名文件或目录

`mv` 命令在Linux系统中用于移动文件或目录，同时也可以用于重命名文件或目录。

### 命令格式

```shell
mv [选项] 源文件或目录 目标文件或目录
```

### 主要参数

- -i：交互式移动，在覆盖文件之前提示用户确认。
- -f：强制移动，不提示用户确认覆盖。
- -n：不覆盖已存在的目标文件。
- -u：仅当源文件比目标文件新，或者目标文件不存在时，才移动文件。
- -v：详细模式，显示命令的执行过程。

### 常见用法及示例

#### 移动文件

将文件 file1.txt 移动到目录 dir1 中：

```shell
mv file1.txt dir1/
```

#### 重命名文件
将文件 oldname.txt 重命名为 newname.txt：

```shell
mv oldname.txt newname.txt
```

#### 交互式移动文件
移动文件 file2.txt 到 dir2，如果 dir2 中已有同名文件，则提示用户确认：

```shell
mv -i file2.txt dir2/
```

#### 强制移动文件
移动文件 file3.txt 到 dir3，即使 dir3 中已有同名文件也不提示确认：

```shell
mv -f file3.txt dir3/
```

#### 移动多个文件

将 file4.txt 和 file5.txt 移动到 dir4 目录中：

```shell
mv file4.txt file5.txt dir4/
```

#### 使用通配符移动文件
将所有 .txt 文件移动到 dir5：

```shell
mv *.txt dir5/
```

### 注意事项

使用 mv 命令时要确保具有对源文件以及目标目录的适当权限。
在移动文件时，如果目标位置已有同名文件，除非使用 -i 参数，否则原文件会被覆盖而不会有提示。
对于重要文件，在执行 mv 命令前进行备份是一个好习惯。

## touch：创建空文件或更新文件的时间戳

```shell
touch file_name
```

# 打包与解压

## tar

tar 是Linux中最常用的打包压缩工具，该命令可以把一系列文件打包到一个大文件中，也可以把一个大文件恢复一系列文件。

### 主要参数

- -c：生成档案文件，创建打包文件
- -x：解开档案文件
- -v：列出归档接档的详细过程，显示进度
- -f：指定档案文件，f后面一定是.tar 文件，必须放选项最后

### 常见用法

```shell
# 压缩目录
tar -czvf archive.tar.gz directory_name
# 一次可以打包多个文件
tar -cvf pkg.tar a.txt b.txt c.txt 

# 解压文件
tar -xzvf archive.tar.gz  
```

# 进程

## ps

ps，是在Linux中是查看进程的命令。ps查看正处于Running的进程，`ps aux`查看所有的进程。

### 主要参数

- `-a`：显示一个终端的所有进程，除了会话引线。
- `-u`：显示进程的归属用户及内存的使用情况。
- `-x`：显示没有控制终端的进程。
- `-e`选项显示所有进程，包括系统守护进程。

### 常见用法

查找指定进程格式：

```shell
ps -ef | grep 进程关键字
```

## kill

kill 命令用于终止正在运行的进程。

kill 命令可以发送不同的信号给目标进程，来实现不同的操作，如果不指定信号，默认会发送 TERM 信号（15），即终止。若仍无法终止该程序，可使用 SIGKILL(9) 信息尝试强制删除程序。

### 常见用法

```shell
kill -9 PID
```



























