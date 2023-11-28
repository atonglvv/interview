# 查看Linux内核

cat /etc/redhat-release

uname -r



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



# 新建文件夹

```shell
mkdir <folderName>
```



# 解压

```shell
tar zxvf <FileName.tar.gz>
```































