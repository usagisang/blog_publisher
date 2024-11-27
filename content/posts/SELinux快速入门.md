---
tags: 
title: SELinux快速入门
date: 2024-11-27
---

> 本文大量参考了 Gentoo Linux 文档中与 SELinux 相关的 [wiki](https://wiki.gentoo.org/wiki/SELinux)

SELinux 安全子系统是对基于 UNIX 权限位的 Linux 常规访问控制的补充，不同于常规访问控制，SELinux 提供的访问控制更加安全但同时也更加难以维护。SELinux 灵活且复杂，但它的工作原理其实很简单，我们将逐步深入，介绍 SELinux 支持的各种功能。

## 访问控制与SELinux上下文

为了说明 SELinux 到底是如何进行访问控制的，让我们来假设这样一个使用场景：现在有一名已登录的用户，向 shell 进程发送指令，要求 shell 进程读取一个文件 File。那么，SELinux 现在需要判断 shell 进程是否有权限读取文件 File。

SELinux 进行判断的依据非常简单。首先，无论是进程还是文件，SELinux 都会维护它们的”标签“，比如说 shell 进程的”标签“是 user_t，而文件 File 的”标签“是 lib_t，然后根据它们两个的标签， SELinux 将在规则集合中寻找相应的条目，如果找到的条目为 allow，即允许访问，那么 SELinux 将放行此次访问，否则拒绝。

这就是 SELinux 的工作原理：基于”标签“来查找匹配的规则。

给”标签“打上双引号是因为这只是方便读者理解原理而使用的名字，它的正式称呼是 **SELinux 上下文**。另外，我们所述的 user_t 并不是完整的 SELinux 上下文。完整的 SELinux 上下文由以下3个部分组成（有时是4个部分）：

1. 第一部分是 SELinux 用户
2. 其次是 SELinux 角色
3. 接下来是 SELinux 类型
4. 最后是可选部分，表示敏感度级别

比如说，一个文件的 SELinux 上下文可能会是这样的：`system_u:object_r:lib_t:s0`。`system_u`表示 SELinux 用户，`object_r`表示 SELinux 角色，`lib_t`是 SELinux 类型，最后的`s0`则表示敏感度级别。

![](https://wiki.gentoo.org/images/6/69/SELinux_context.png)

或许你已经注意到，SELinux 用户、角色、类别都有一个后缀，这只是一个命名惯例。在 SELinux 世界中，基本上可以认为，带有`_u`后缀的名字是在指代 SELinux 用户，带有`_r`后缀的名字是在指代 SELinux 角色，带有`_t`后缀的名字在指代 SELinux 类型。我们接下来讨论 SELinux 也会不时运用这些命名惯例，所以当你看到`_t`后缀的名字但没说明它是什么的时候，请不要惊讶。

每个进程和文件都有各自的上下文，在启用了 SELinux 的系统上，如果想要查看文件或进程的 SELinux 上下文，请在调用相关命令时添加`-Z`参数，比如`ls -Z`可以显示文件的上下文，`ps -Z`可以显示进程的上下文。

SELinux 使用每个字段来决定对访问的控制，我们将逐步介绍 SELinux 上下文中这些字段的具体作用。但实际上大多数规则都是围绕着 SELinux 类型来制定的，从这个角度出发，我们决定先介绍 SELinux 上下文中最重要的信息： SELinux 类型。

## 类型强制规则

类型强制（Type Enforcement，简称TE）基于 SELinux 上下文中的 SELinux 类型。TE 模型是 SELinux 发挥作用的基石，构成了绝大多数 SELinux 策略。

TE 模型规则的底层逻辑可以归结为三个单词组成的句子："**Subject-Access-Object**"，即“主体-操作-客体”。

在 SELinux 中，主体（Subject）指的是进程。操作（Access）指代的是一系列动作，比如 read、write、ioctl 等系统调用。客体（Object）指的是操作适用的系统资源，包括文件、进程、socket 等，客体是被动的，不会主动执行任何操作——当它主动执行任何操作时它就成为了主体。

进程既可以成为主体，也可以作为客体，比如，一个进程向另一个进程发送终止信号时，进程就既是主体也是客体。

规则底层逻辑的这三个要素体现在了具体的规则中。一个典型的 TE 规则的形式如下所示：

```
kind source target:class permissions;
```

- kind——有几种选项，常用的是 allow 和 neverallow。allow 表示允许；neverallow 表示不允许；
- source——规则的主体的类型。“谁在请求访问”

- target——客体的类型。“请求访问什么”
- class——类别。正在访问的客体（file、 socket、process 等）的类别
- permissions——正在执行的操作（或一组操作）（例如，read、write）

一个实际的示例如下：

```
allow user_t bin_t:file { read write };
```

上面的规则的含义是：允许类型为 user_t 的进程，读取或写入，类型为 bin_t 而且类别（class）为 file 的客体。

需要注意的是，由于主体和客体都拥有 SELinux 类型，为了方便区分这两者，与 SELinux 相关的文档中常常把赋予进程的 SELinux 类型称作域（domain），客体的则仍称作“类型”，我们也会延续这种习惯。

客体需要区分类别，因为不同的系统资源所支持的操作集不同，比如，主体可以向进程但无法向文件发送信号。从这个角度来看，客体是由类型(bin_t)和类别(file)共同组成的。

> 如果想查询客体有哪些类别，请执行`ls /sys/fs/selinux/class`
>
> 如果想知道某一个类别的操作集，比如说 file 类别的操作集，请执行`ls /sys/fs/selinux/class/file/perms/`

### 属性

SELinux 中常常包含上千种 SELinux 类型，因此，经常会遇到大部分主体对某个客体的权限完全一样的情况，如果规则只接受单个主体到单个客体的映射，会导致大量对同一个客体的重复定义。比如说，我们有三个类型对同一个客体的访问权限一致：

```
allow trusted_app app_data_file:file { read write };
allow untrusted_app app_data_file:file { read write };
allow isolated_app app_data_file:file { read write };
```

为了解决这种繁琐的重复定义问题，SELinux 支持对访问控制规则进行分组，这项特性被称为属性（attributes）。

我们可以将域或类型分配到某一个属性，在定义访问控制规则时，可以利用属性进行定义，属性可以用在主体级别、对象级别或同时用于两者。

属性定义以及将属性用于规则的示例如下：

```
# Associate the attribute appdomain with the type untrusted_app.
typeattribute untrusted_app appdomain;

# Associate the attribute appdomain with the type isolated_app.
typeattribute isolated_app appdomain;

allow appdomain app_data_file:file { read write };
```

至此，我们总结一下前面的内容。SELinux 绝大部分规则都是基于类型的规则。根据这些类型，SELinux 将授予或拒绝进程对文件的相应操作。多数情况下，在 SELinux 那冗长的上下文中我们只需要关注类型字段，而且由于完整的上下文真的太长了，下文也常常用 SELinux 类型来指代 SELinux 上下文这个概念。

## 上下文继承

默认情况下，若 SELinux 中没有其他指定的策略，则新的进程和文件的 SELinux 类型是从其父级继承而来的。比如说：

- 以`foo_t`上下文运行的进程 fork 另外一个新进程时，此进程的上下文也为`foo_t`
- 在上下文为`bar_t`的目录中创建的文件或目录也将获得`bar_t`上下文

如同所有的进程的源头都是 init 进程一样，追溯 SELinux 的上下文继承体系，也有一个所谓的根上下文。进程和文件的根上下文都由相应的 SELinux 策略定义。一般来说，在 Linux 中文件的根上下文是`root_t`，进程的根上下文是`kernel_t`。

## 域转换

> [进程如何进入特定的上下文](https://wiki.gentoo.org/wiki/SELinux/Tutorials/How_does_a_process_get_into_a_certain_context)

既然进程有根上下文，那么 SELinux 势必要支持域转换，否则所有的进程只会拥有同一个上下文。不过，不要误会，域转换并不是说一个进程可以动态修改它的 SELinux 类型，SELinux 的域转换只能发生在旧进程创建新进程的时候，旧有的进程可以以不同的安全上下文启动新进程。

进程可以通过两种方式进行域转换，第一种是为不支持 SELinux （不使用 libselinux ）且对 SELinux 一无所知的应用准备的，第二种是使用 libselinux 相应的 API 来指定新进程的域。但无论如何，定义域转换只是第一步，不要忘记 SELinux 对权限的控制有多严格！SELinux 并不会自动授予相关的允许权限，所以在第二步，我们必须编写相应的 allow 规则以允许域转换。

### 定义域转换

进程可以通过两种方式来定义域转换。

#### type_transition语句

这是最常用的方式。当不支持 SELinux 的程序执行`exec`系统调用时，SELinux 将自动为其进行域转换。定义语句的示例如下：

```
type_transition init_t initrc_exec_t : process initrc_t;
```

它的含义是：当 init_t 进程执行（通过`exec`）上下文为 initrc_exec_t 的文件时，生成的进程应在 initrc_t 上下文中运行。在满足后面我们会提到的条件后，SELinux 将自动执行域转换。

上面的示例展示了定义域转换所需的三个要素：原来的域，可执行文件的类型，新的域。

采用这种方式定义域转换不存在侵入性，因而非常适合那些不方便引入 libselinux 库的程序，init 进程就是一个很好的例子——即使没有 libselinux，init 进程也应该正常地初始化整个环境。

#### 使用 libselinux 中的 setexeccon()

支持 SELinux 的应用可以使用 libselinux 中的 setexeccon() 来指定新进程的域。这个函数将设置用于下一个 exec 调用的上下文。另外，应用必须具有 setexec 权限才能使用这个 API，例如：

```
allow crond_t self:process setexec;
```

`self:process`是一种特殊的客体写法，但应该不难理解，它指明客体就是主体的这个进程。

### 允许域转换

无论使用哪种方式定义了域转换，这种转换最终都和`exec()`系统调用脱不开关系。而为了允许域转换，我们需要额外添加三条 allow 规则。这些策略用来满足以下三个条件：

1. 原始域对文件具有执行权限
2. 文件上下文本身被标识为目标域的入口点（entrypoint）
3. 允许原始域转换到目标域

我们以`initrc_t`进程通过执行`sshd_exec_t`文件来将新进程转换到`sshd_t`这个域为例。

首先，`exec()`调用需要指定某个可执行文件，这当然需要授予一个允许进程执行文件的权限。比如，允许`initrc_t`进程执行`sshd_exec_t`文件：

```
allow initrc_t sshd_exec_t : file { read getattr execute open } ;
```

然后，SELinux 并不知道`sshd_exec_t`文件和`sshd_t`域之间到底有什么关系，因此我们需要添加一条规则来告诉 SELinux，执行`sshd_exec_t`文件时，可以将域转换到`sshd_t`。这种特性被我们称之为 entrypoint，定义的方式为，在通常的 allow 规则中，主体`sshd_t`对客体`sshd_exec_t`的操作集中添加 `entrypoint`，示例如下：

```
allow sshd_t sshd_exec_t:file { ioctl read getattr lock execute execute_no_trans open entrypoint };
```

> 由于“主体-操作-客体”的 SELinux 底层逻辑的限制，我们没办法以客体为主语，因此 entrypoint 的 allow 规则显得比较抽象，遇到 entrypoint 时，试着反过来理解 allow 规则，逻辑会更顺畅。

最后，我们还需要告诉 SELinux，允许从`initrc_t`域转换到`sshd_t`域。

```
allow initrc_t sshd_t:process transition;
```

为什么我们最后需要定义一个 allow 规则来允许一个域向另一个域转换？这是因为一个文件可以成为多个域的 entrypoint，比如说，`sshd_exec_t`同时还是`xm_ssh_t`域和`ssh_t`域的 entrypoint。因此，SELinux 不允许一个域获得一个文件的执行权限就可以根据 entrypoint 定义转换到其他所有域， SELinux 要求明确指出，从哪个域到哪个域的转换才是被允许的。

## 客体的类型转换

和进程间的域转换类似，我们也可以为客体定义相应的类型转换语句，让 SELinux 在满足相应条件后自动执行类型转换。例如，现在有一个域为`ext_gateway_t`的进程，希望在类型为`in_queue_t`的文件夹内保存文件时，该文件的类型不继承父目录的类型，而是使用另一个类型`in_file_t`。示例的定义语句如下：

```
type_transition | source_domain |  target_type :    object
----------------▼---------------▼--------------▼-----------------
type_transition   ext_gateway_t    in_queue_t  : file in_file_t;
```

这个 type_transition 语句的含义是，当在`ext_gateway_t`域（source_domain） 中运行的进程想要在类型为`in_queue_t`的目录中创建`file`对象时，如果策略允许，则应将该文件重新标记为`in_file_t`

为了能够创建文件，我们还需要在 SELinux 中添加相应的 allow 规则。

- 源域需要有权限将`file`添加到`in_queue_t`目录中

  ```
  allow ext_gateway_t in_queue_t:dir { write search add_name };
  ```

- 源域需要有创建`in_file_t`文件的权限

  ```
  allow ext_gateway_t in_file_t:file { write create getattr };
  ```

## SELinux上下文与其他安全模型

我们已经围绕着 TE 模型及其规则介绍了许多内容，相信你已经体会到它的强大之处。靠着 TE 模型，SELinux 似乎已经可以工作得很好，但人类的需求总是复杂多变。不同于 TE 模型的设计思路，人类经常以用户、角色、项目等比较抽象的概念来管理权限。SELinux 为这些希望使用不同安全模型的需求提供了直接的支持，实际上，安全上下文中我们尚未介绍到的那些字段全都是用来支持其他安全模型的。

SELinux 角色与 SELinux 用户直接支持了 RBAC  （Role-Based Access Control，基于角色的访问控制）模型；SELinux 敏感度级别则支持了 MCS（Multi-Level Security）和 MLS（Multi-Category Security）模型。

这些名词你可能很熟悉也可能很陌生，我们稍后会解释这些名词的含义。但重要的是要理解，这些安全模型都是建立在 TE 模型之上的额外的“高级”模型，只有通过 SELinux 类型的检查后，SELinux 上下文的其他部分才会继续发挥作用。SELinux 提供这些功能只是为了便于人类进行管理，但不使用它们，SELinux 也能正常运行。

## 角色与RBAC模型

首先，需要说明的一点是，SELinux 角色只针对主体，对客体（文件）来说没有意义，因此，如果你使用`ls -Z`来查看文件的角色，你会发现它们清一色都是`object_r`。这个字符串只是因 SELinux 上下文不能缺少角色字段而存在的一个占位符，将进程更改为以 object_r 角色运行或尝试为文件分配不同的角色始终会被内核拒绝。

那么，SELinux 中的角色与主体的访问控制之间有什么联系呢？为了回答这个问题，我们首先捋清楚 Linux 用户、SELinux 用户、SELinux 角色和域（SELinux 类型）之间的关系，如下图所示。

![](https://wiki.gentoo.org/images/a/a0/SELinux_users.png)

SELinux 用户与  Linux 用户不同，它是一个独立的概念。在 SELinux 中，每个 Linux 用户都必须映射到且只能映射到一个 SELinux 用户，但支持多个 Linux 用户映射到同一个 SELinux 用户上；SELinux 用户与 SELinux 角色则是一种多对多的关系，每个 SELinux 用户都可以持有多个角色；而对于每一个 SELinux 角色来说，它们可以关联到不同的域。

这种关联是在规定 SELinux 角色可以进入哪些域（上下文）。当然，用户控制的进程需要先通过域转换这一关， SELinux 角色的限制则在于，即使被允许转换到目标域，如果该域未附加到相应角色，转换也会失败。

我们举一个例子说明这一点：

假设一名开发人员试图从命令行启动 mysql 守护进程。我们知道，shell 进程被允许执行`mysqld_exec_t`文件并进行域转换，新的守护进程的上下文应该是`mysqld_t`，但不巧的是，这名人员的角色是`user_r`而不是数据管理员`dbadm_r`，`mysqld_t`这个域只与数据管理员关联，而没有关联到`user_r`这个角色上，因此本次操作将被 SELinux 阻止。

总的来说，角色进一步限制了用户能够与哪些进程打交道。

### RBAC模型

SELinux 角色可以用于实现 RBAC （Role-Based Access Control，基于角色的访问控制），RBAC 是一种抽象的访问控制模型， 核心理念是：

- 权限始终通过角色授予，不直接分配给用户
- 必须明确授予用户相应角色，没有角色，就没有权限

## MLS与MCS

我们介绍 SELinux 上下文中的最后那一部分：敏感度级别。其实在之前的示例中，我们没有完整展现这一部分的内容，实际上敏感度级别分为两个维度的内容，分别是秘密等级和类别集。比如说下面的上下文：

`user_u:user_r:user_t:s0:c0,c1`

`s0`表示秘密等级，`c0,c1`表示类别集是0和1。

秘密等级可以看作对现实中的秘密等级的抽象实现。如果你了解《中华人民共和国保守国家秘密法》，应该知道所谓的“国家秘密”分为三种：绝密、机密和秘密。体现到 SELinux 上，我们可以规定，s3 表示绝密，s2 表示机密，s1 表示秘密，s0 表示公开。这样一来，即使 SELinux 类型允许进程访问高等级秘密文件，但由于秘密等级不匹配，进程依旧会读取失败。比如说，对于 s2 的进程来说，它可以读取 s2 或更低级别的文件，但无法读取更高级别比如 s3 文件。秘密等级实现了所谓的多级安全性（Multi-Level Security，MLS）。

结合上面的秘密等级，类别集可以理解为，我们有两个秘密等级相同的项目，比如有两个 s3 绝密项目，虽然它们秘密等级相同，我们也不希望这两个项目之间的人员（进程）互相查看对方的资料（文件），因此，我们使用类别来区分这两个项目。比如说，项目 X 被划分到`c0`，项目 Y 被划分到`c1`，那么，持有`s3:c0`的进程可以访问项目 X 的文件，但无法访问项目 Y 的文件了。SELinux 使用类别集，实现了所谓的多类别安全性（Multi-Category Security，MCS）。

最后，我们说明一下这部分上下文的一些特殊形式。对于秘密等级，可以使用`-`符号来表示范围，比如`s0-s2`就表示从 s0 到 s2 的等级。对于类别集，可以使用`.`符号来表示范围，比如`c0.c15`表示从类别集 c0 到 c15。

## 拒绝日志

本节简单介绍一下 SELinux 拒绝日志里面包含了什么内容，至于如何根据拒绝日志添加 SELinux 策略，这是一个比较复杂的问题，本节不会讨论。

在查看拒绝信息之前，我们需要注意以下几点：

1. 在日志中发现的拒绝并非都是大问题。有些拒绝只是表面上发生了，但不会影响应用程序的行为。这通常是由于应用程序开发不当（例如未正确关闭文件描述符）或由于高级库函数（应用程序仅使用了一小部分功能）造成的。
2. 拒绝一出现就会被记录下来。这意味着在日志中我们将看到大量的拒绝，尽管许多拒绝彼此相关（一个拒绝导致另一个拒绝），但大部分拒绝与正在调查的问题无关。
3. 如果连续出现太多拒绝，Linux 内核可能会抑制这些拒绝。因此可能看不到 SELinux 报告的所有内容。

拒绝日志的示例如下：

```
avc: denied { open } for pid=1003 comm=”mediaserver” path="/dev/kgsl-3d0”
dev="tmpfs" scontext=u:r:mediaserver:s0 tcontext=u:object_r:device:s0
tclass=chr_file permissive=1
```

下表给出了每一部分日志的解释

| 日志                 | 描述                                                                                 |
| ------------------ | ---------------------------------------------------------------------------------- |
| avc:               | 告知用户这是哪种类型的日志条目。在本例中，这是 AVC 日志条目。                                                  |
| denied             | SELinux 最终的反应，可以是 denied 或 granted。注意，如果 SELinux 处于宽容模式，尽管实际没有拒绝，在日志中依然会被记录 denied |
| \{ open \}         | 试图执行的操作，有时会包含一组操作，如\{ read write \}                                                |
| pid=1003           | 进程 pid                                                                             |
| comm=”mediaserver” | 进程命令（不带参数，且限制为 15 个字符），帮助用户在进程已经死亡的情况下识别该进程是什么                                     |
| path=              | 目标的绝对路径。注意，此字段很大程度上取决于目标类别，因此可能是path=、name=、capacity=、src= 等等                      |
| dev="tmpfs"        | 目标所在的设备                                                                            |
| scontext=          | 进程的上下文（域）                                                                          |
| tcontext=          | 目标资源（本例中为文件）的上下文                                                                   |
| tclass=            | 目标的类别                                                                              |
| permissive=1       | 表示是否允许此次操作，为1表示允许                                                                  |

## 本文的SELinux策略语法

SELinux 有两套[策略语言](http://selinuxproject.org/page/PolicyLanguage#CIL_Policy_Language)，分别是 CIL 策略语言和内核策略语言。本文所展示的 SELinux 策略代码均基于内核策略语言。

