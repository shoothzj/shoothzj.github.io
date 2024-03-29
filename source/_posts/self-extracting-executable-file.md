---
title: 创建自解压的可执行文件
date: 2023-10-23 20:18:50
tags:
---
# 为什么需要自解压的可执行文件

大部分软件的安装包是一个压缩包，用户需要自己解压，然后再执行安装脚本。常见的两种格式是`tar.gz`和`zip`。常见的解压执行脚本如下

## tar.gz

```bash
#!/bin/bash

tar -zxvf xxx.tar.gz
cd xxx
./install.sh
```

## zip

```bash
#!/bin/bash

unzip xxx.zip
cd xxx
./install.sh
```

在有些场景下，为了方便分发、安装，我们需要将多个文件和目录打包并与一个启动脚本结合。这样子就可以实现一键安装，而不需要用户自己解压文件，然后再执行启动脚本。

核心原理是，通过固定分隔符分隔脚本和压缩包部分，脚本通过分隔符将压缩包部分提取出来，然后解压，执行安装脚本，脚本不会超过固定分隔符。解压可以通过临时文件(zip)或流式解压(tar.gz)的方式实现。

# 创建包含zip压缩包的自解压可执行文件

## 构造一个zip压缩包

```bash
echo "hello zip" > temp.txt
zip -r temp.zip temp.txt
rm -f temp.txt
```

## 构造可执行文件 `self_extracting.sh`

以使用`__ARCHIVE_BELOW__`做分隔符为例，`self_extracting.sh`里面内容:

推荐把临时文件放在内存文件路径下，这样子可以避免磁盘IO

```bash
#!/bin/bash
CURRENT_DIR="$(dirname "$0")"

ARCHIVE_START_LINE=$(awk '/^__ARCHIVE_BELOW__/ {print NR + 1; exit 0; }' $0)

tail -n+$ARCHIVE_START_LINE $0 > /tmp/temp.zip
unzip /tmp/temp.zip" -d "$CURRENT_DIR"
rm "$CURRENT_DIR/temp.zip"

# replace the following line with your own code
cat temp.txt

exit 0

__ARCHIVE_BELOW__
```

将zip文件追加到`self_extracting.sh`文件的尾部
```
cat temp.zip >> self_extracting.sh
chmod +x self_extracting.sh
```

# 创建包含tar.gz压缩包的自解压可执行文件

## 构造一个tar.gz压缩包

```bash
echo "hello tar.gz" > temp.txt
tar -czf temp.tar.gz temp.txt
rm -f temp.txt
```

## 构造可执行文件 `self_extracting.sh`

以使用`__ARCHIVE_BELOW__`做分隔符为例，`self_extracting.sh`里面内容:

```bash
#!/bin/bash
CURRENT_DIR="$(dirname "$0")"

ARCHIVE_START_LINE=$(awk '/^__ARCHIVE_BELOW__/ {print NR + 1; exit 0; }' $0)
tail -n+$ARCHIVE_START_LINE $0 | tar xz -C "$CURRENT_DIR"

# replace the following line with your own code
cat temp.txt

exit 0

__ARCHIVE_BELOW__
```
