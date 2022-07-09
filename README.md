# CliTestFramework

## 背景

1. 输入
命令(配置/显示)
2. 输出
字符串(配置/运行数据)

## 需求

1. 检测与预期结果的差距，如不符，报错或记录
2. 可以覆盖所有可能的输入路径

扩展：
1. 记录下发的命令和捕获的字段

## 需求分解

1. 输入命令下发,读取
2. 提取命令结果
3. 读取/遍历命令线索
4. 命令结果的表示与存储
5. 查找对应命令结果

## 方案选型

运行环境：

使用python编写，需要远程执行，最好是一个Linux环境，首选编译服务器。

目标：

先考虑最简单的实现--1.仅仅检测配置的一致性，而不检查显示命令的某字段是否正确2.遍历的命令与其对应结果通过手工输入，后续再扩展

### 需求1：输入命令下发,读取

相关python库：

[invoke]: https://www.pyinvoke.org/

Invoke可用于管理面向shell的子流程，并将可执行的Python代码组织为cli可调用的任务。

[Paramiko]: https://www.paramiko.org/

一个纯Python实现的sshv2协议，提供了客户端和服务端的功能。

[Fabric]: https://www.fabfile.org/

一个更高层的python库，被设计通过SSH远程执行shell命令，并返回有用的Python对象。它构建在Invoke(子进程命令执行和命令行特性)和Paramiko (SSH协议实现)之上，扩展它们的api以相互补充并提供额外的功能。

使用Fabric库能够保证输入输出以及捕获：

```bash
from fabric import Connection
>>> c=Connection(
... host="link.local",
... user="",
... connect_kwargs={"password":""}
... )
>>> res=c.run("ps")
    PID TTY          TIME CMD
    620 ?        00:00:00 systemd
    621 ?        00:00:00 (sd-pam)
    690 ?        00:00:00 pipewire
...
>>> type(res.stdout)
<class 'str'>
>>> print(res.stdout)
    PID TTY          TIME CMD
    620 ?        00:00:00 systemd
    621 ?        00:00:00 (sd-pam)
    690 ?        00:00:00 pipewire
...
```

对于CNOS系统，有些配置维护在两个地方，cdb数据库和FRR中。ssh连接后，位于klish，此时能显示的配置都是cdb的。而FRR中的配置必须通过进入shell来查看，而进入shell需要二维码认证。应对方案如下：

1.去除认证

1.1 编译时去除认证模块。需要查看Makefile，成功率未知

1.2 使用命令去除认证模块，成功率未知

2.其他方式查看

2.1 通过klish新加命令，可以通过不同关键字直接输出FRR配置。实现成功率高，容易执行，只需要修改xml,正式版本也能用。

3.进入shell

3.1 如果密码短时间不变，可以先手动获取验证码，然后传入。太麻烦，否决。

3.2 集成扫码工具。放弃，理由同上。

方案2，3都需要在shell和klish间切换，采用方案2.1，可以直接在klish查看FRR配置，选用方案2.1。

### 需求2：提取命令结果

通过invoke的库来实现命令结果初步获取。

使用正则提取，类似与下面的方式：

```bash
>>> print(re.findall(r'^\s+\d+.+',res.stdout,re.M)[0])
    620 ?        00:00:00 systemd
```

如果匹配不到，则认为失败。

关于需要提取的模式，可以手工指定，或者复用c代码，但是需要编makefile.

### 需求3：读取/遍历命令线索

1. 手工遍历所有可能的命令线索

2. 自动化，考虑使用klish或invok。但是后续对正确结果的存储与匹配处理可能会很复杂。

### 需求4：命令结果的表示与存储

1. 若手工指定了命令线索，直接存储

2. 可以针对解析的结果，做一个映射，自动生成

通过invoke的库来实现需要下发命令的存储。

### 需求5：查找对应命令结果

根据序号一一对应
