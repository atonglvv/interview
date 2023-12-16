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



# 文件查看

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

## more

一页一页地显示文件内容，后跟文件名称 【**空格键**向下翻页，**Enter键**向下滚动一行，**Q键**退出】

``` shell
# 通过more命令查看openfire.xml
more openfire.xml
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

## less

# 文件查找



# 文件夹

## 新建文件夹

```shell
mkdir <folderName>
```

## 删除文件夹

```shell
rm -rf <folderName>
```



# 解压

```shell
tar zxvf <FileName.tar.gz>
```































