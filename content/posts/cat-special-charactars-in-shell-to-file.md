---
date: "2018-04-20T20:18:57+08:00"
draft: false
title: "将shell 命令中特殊字符 cat 到文件的方法"
subtitle: "一行或多行命令的中特殊字符输出方法"
categories: "linux-practice"
tags: ["Linux","cat","shell-script"]
description: "Linux 实践错误记录"
---

有时候，我们会通过cat 的方式在终端敲入多行shell 命令到一个文件中，其中会遇到诸如“ ` ”  （ESC 下边的键）或者 “$” 这两个特殊的字符。把这两个字符敲入文件，需要做转义处理。

例如，我们看到这样的一个shell 脚本。

```
vi example.txt

for  i  in `find derectory1 -type f -name "*.tgz"`
do
  gunzip -c  $i | docker load
done
```

在终端通过cat 输出上边的文件，需要这样做：

- 对特殊字符使用转义字符

```
cat > example.txt << EOF
for i in \`find docker_tar -type f -name "*.tgz" \`
do
  gunzip -c \$i |docker load
done
EOF
```

- 或者，如果对于一个文件中有多处需要转义的字符，最简单的办法就是在开始的EOF 两边加上引号就可以 ，具体如下:

```
cat > example.txt << "EOF"
for i in `find docker_tar -type f -name "*.tgz" `
do
  gunzip -c $i |docker load
done
EOF
```






