# 原文:[7个小矮人：在GDB之外调试信息](https://landley.net/kdocs/ols/2007/ols2007v2-pages-35-44.pdf)

Arnaldo Carvalho de Melo
Red Hat, Inc.
acme@redhat.com
acme@ghostprotocols.net
Abstract
矮人调试信息格式是如此
在GDB等访问者中使用了远处的使用
SystemTap和Frysk等工具。
在本文中，作者将显示其他场景
此类信息可能有用的地方，例如：
•显示数据结构的布局；
•重组此类数据结构以消除对齐孔；
•改善CPU缓存利用率；
•显示有关函数内在的统计信息；
•从调试信息中重新创建结构和功能；
•显示二进制差异以帮助了解效果
任何代码更改。
以及更多。
1介绍
本文讨论了使用编译器插入二进制的矮人调试信息的新方法
例如GCC。
作者开发了几种允许的工具：
•提取有关数据结构的有用信息
布局;
•查找编译器插入的孔和填充物
遵循处理器体系结构中的对齐约束；
•找出减少此类数据的可能性
结构；
•使用有关功能参数的信息以及
返回类型以生成Linux内核模块
获取生成呼叫图所需的数据
和在运行时设置为字段的值；
•给定两个对象文件显示二进制文件的工具
差异以帮助理解来源的影响
功能和数据结构的大小的代码更改。
将出示一些用例，显示工具如何
可以用来解决现实世界中的问题。
与一些其他开发人员讨论但尚未讨论的想法
也将出现尝试，并有希望
最终让他们在实践中被感兴趣的读者进行测试。
2矮人调试格式
矮人[3]是许多许多人使用的调试文件格式
编译器存储有关数据结构的信息，
所需的变量，功能和其他语言方面
由高级调试者。
它进行了三个重大修订，第二个是
与第一个不兼容，第三是扩展
第二，增加了对更多C ++概念的支持，提供了消除数据重复的方法
在共享库和文件中调试数据比
4GB。
矮人调试信息在
对象文件中的几个精灵部分，其中一些将
在这里提到。请参阅矮人[3]规范以获取完整列表。最近的发展
诸如Elfutils [2]之类的工具允许使用
•35•
36•7个矮人：在GDB之外调试信息
调试信息，但常见的情况是
将包装和各自包装的信息
对象文件。
调试数据是在具有属性的标签中组织的。
标签可以嵌套以表示变量
内部词汇块，功能的参数和其他
分层概念。
例如，让我们看看无处不在的Hello
世界示例在“ .debug_info”部分中表示，这与该主题最相关的一个部分
本文：

```none
$ cat hello.c
int main(void)
{
printf("hello, world!\n");
}
Using gcc with the -g flag to insert the debugging information:
$ gcc −g hello.c −o hello
Now let us see the output, slightly edited for brevity,
from eu-readelf, a tool present in the elfutils package:
$ eu−readelf −winfo hello
DWARF section ‚.debug_info‚ at offset
0x6b3:
[Offset]
Compilation unit at offset 0:
Version: 2, Abbrev section offset: 0,
Addr size: 4, Offset size: 4
[b] compile_unit
stmt_list 0
high_pc 0x0804837a
low_pc 0x08048354
producer "GNU C 4.1.1"
language ISO C89 (1)
name "hello.c"
comp_dir "~/examples"
[68] subprogram
external
name "main"
decl_file 1
decl_line 2
prototyped
type [82]
low_pc 0x08048354
high_pc 0x0804837a
frame_base location list [0]
[82] base_type
name "int"
byte_size 4
encoding signed (5)
```

以[数字]开头的条目是矮标签
在工具的源代码中表示为dw_tag_
标签名。在上面的输出中，我们可以看到一些：dw_
tag_compile_unit，并提供有关
分析对象文件，dw_tag_subprogram，
为每个功能发射，例如“ main”和DW_
tag_base_type，为语言基本发出
类型，例如int。
每个标签都有许多属性，表示
源代码为dw_at_attribute_name。在里面
dw_tag_subprogram的“主”函数我们
有一些：dw_at_name（“ main”），dw_at_decl_
文件，这是另一个矮人部分的索引
使用源代码文件的名称，dw_at_
dect_line，源代码中的行
定义了函数，dw_at_type，返回类型
对于“主要”例程。它的价值是标签索引，在
这种情况是指[82]标签，即DW_TAG_
base_type用于“ int”，以及此功能的地址，dw_at_low_pc。
以下示例公开了一些其他
七个矮人中使用的矮标签和属性。
以下结构：

```none
struct swiss_cheese {
char a;
int b;
};
is represented as:
[68] structure_type
name "swiss_cheese"
byte_size 8
[7d] member
name "a"
type [96]
data_member_location 0
[89] member
name "b"
type [9e]
data_member_location 4
[96] base_type
name "char"
byte_size 1
2007 Linux Symposium, Volume Two • 37
[9e] base_type
name "int"
byte_size 4
```

除了已经描述的标签外，我们现在还有
dw_tag_structure_type启动具有dw_at_byte_size的结构的表示形式
属性说明结构采用多少字节（8
在这种情况下）。还有另一个标签DW_
tag_member，代表每个结构成员。它
有dw_at_byte_size和dw_at_data_
Member_Location属性，结构中此成员的偏移。还有更多属性，但是
简洁，出于本文的目的，以上是
足以描述。
3 7矮人
七个矮人是使用矮人的工具
调试信息以检查数据结构布局
（Pahole），检查可执行的代码特性
（pfunct），比较可执行文件（CODIFF），与结构（ctracer）关联的函数的跟踪执行，
漂亮的印刷矮人信息（pdwtags），列表
全局符号（pglobal），并计算
使用每组标签的时间（prefcnt）。
有些非常简单，仍然需要工作，而另一些（例如Pahole和Plunct）已经在
对诸如Linux内核等开源项目有帮助，
Xine-lib和Perfmon2。一种可能的用途是在公开发布的二进制内核模块中意外留下的矮小信息。
所有这些工具都使用一个名为libdwarves的库，
用工具包装并使用矮人
在Elfutils中发现的图书馆[2]。通过使用Elfutils，许多
它的功能，例如搬迁，读取对象文件的
许多架构，使用单独的文件与调试
提供信息等，允许作者
专注于将在
下一个部分。
除非另有说明，否则以下示例
部分使用为X86-64构建的Linux内核图像
最新源代码的体系结构1
Linux内核配置选项配置_
必须选择debug_info来指导编译器
1
大约2.6.21-RC5。
插入矮小信息。这将使
图像更大，但对
产生的二进制，就像构建用户空间时
带有调试信息的程序。
3.1 Pahole
poke-a-hole，第一个矮人，用于查找对齐方式
结构中的孔。它是所有工具中最先进的
到目前为止，这个项目。
体系结构具有对齐约束，需要数据
将在记忆中对齐的类型
单词大小。虽然编译器确实会自动对齐数据
结构，开发人员的仔细计划是必不可少的
为了最大程度地减少正确的桨板（“孔”）
数据结构成员的对齐。
不良结构布局的一个例子是需要更好
说明这种情况：

```c
struct cheese {
char name[17];
short age;
char type;
int calories;
short price;
int barcode[4];
};
```

加起来一个人可能期望的成员的大小
结构奶酪的大小为17 + 2 + 1 + 4 + 2 +
16 = 42个字节。但是由于对齐限制了真实的
大小最终为48个字节。
使用pahole来漂亮印刷矮标签将
显示6个额外字节在哪里：

```none
/∗ <11b> ∼/examples/swiss_cheese.c:3 ∗/
struct cheese {
char name[17]; /∗ 0 17 ∗/
/∗ XXX 1 byte hole, try to pack ∗/
short age; /∗ 18 2 ∗/
char type; /∗ 20 1 ∗/
/∗ XXX 3 bytes hole, try to pack ∗/
int calories; /∗ 24 4 ∗/
short price; /∗ 28 2 ∗/
38 • The 7 dwarves: debugging information beyond gdb
/∗ XXX 2 bytes hole, try to pack ∗/
int barcode[4]; /∗ 32 16 ∗/
}; /∗ size: 48, cachelines: 1 ∗/
/∗ sum members: 42, holes: 3 ∗/
/∗ sum holes: 6 ∗/
/∗ last cacheline: 48 bytes ∗/
```

这表明在此架构中，对齐规则
指出，短相位必须在2的倍数上对齐
从结构的开头抵消，INT必须是
以4的倍数对齐，单词大小的大小
示例架构。
另一个对齐规则方面是在32位体系结构上完美布置的结构，例如：

```none
$ pahole long
/∗ <67> ∼/examples/long.c:1 ∗/
struct foo {
int a; /∗ 0 4 ∗/
void ∗b; /∗ 4 4 ∗/
char c[4]; /∗ 8 4 ∗/
long g; /∗ 12 4 ∗/
}; /∗ size: 16, cachelines: 1 ∗/
/∗ last cacheline: 16 bytes ∗/
```

建立在具有不同的建筑上时有孔
单词大小：

```none
$ pahole long
/∗ <6f> ∼/examples/long.c:1 ∗/
struct foo {
int a; /∗ 0 4 ∗/
/∗ XXX 4 bytes hole, try to pack ∗/
void ∗b; /∗ 8 8 ∗/
char c[4]; /∗ 16 4 ∗/
/∗ XXX 4 bytes hole, try to pack ∗/
long g; /∗ 24 8 ∗/
}; /∗ size: 32, cachelines: 1 ∗/
/∗ sum members: 24, holes: 2 ∗/
/∗ sum holes: 8 ∗/
/∗ last cacheline: 32 bytes ∗/
```

这是因为在x86-64上，指针的大小和长
整数是8个字节，有一致规则需要
这些基本类型要在8个字节的倍数处对齐
从结构的开头。
在这些情况下，Pahole提供了
--reorganize选项，它将重组
结构试图实现有关的最佳位置
记忆消耗，遵循对齐方式
规则。
在我们获得的X86-64平台上运行它：

```none
$ pahole −−reorganize −C foo long
struct foo {
int a; /∗ 0 4 ∗/
char c[4]; /∗ 4 4 ∗/
void ∗b; /∗ 8 8 ∗/
long g; /∗ 16 8 ∗/
}; /∗ size: 24, cachelines: 1 ∗/
/∗ last cacheline: 24 bytes ∗/
/∗ saved 8 bytes! ∗/
```

还有另一个选项，-show_reorg_steps
阐明了所做的事情：

```none
$ pahole −−show_reorg_steps
−−reorganize −C foo long
/∗ Moving ’c’ from after ’b’ to after ’a’ ∗/
struct foo {
int a; /∗ 0 4 ∗/
char c[4]; /∗ 4 4 ∗/
void ∗b; /∗ 8 8 ∗/
long g; /∗ 16 8 ∗/
}; /∗ size: 24, cachelines: 1 ∗/
/∗ last cacheline: 24 bytes ∗/
```

虽然在这种情况下只完成了一步，但在更复杂的结构中使用此选项可能涉及
许多步骤，这将被证明有助于理解所执行的更改。其他步骤
--reorganize 算法包括：
•组合单独的位字段
•将位字段降级为较小的基本类型
所使用的类型的位比
位字段中的成员（例如int a：1，b：2;
被降级到char a：1，b：2;）
•将成员从结构的末端移动到填充
孔
•将结构末端的填充与
一个洞
2007 Linux研讨会，第二卷•39
几种模式来总结有关所有的信息
还实现了对象文件中的结构。他们会
在以下示例中呈现。
排名前十的结构是：

```none
$ pahole −−sizes vmlinux | sort −k2 −nr
| head
hid_parser: 65784 0
hid_local: 65552 0
kernel_stat: 33728 0
module: 16960 8
proto: 16640 2
pglist_data: 14272 2
avc_cache: 10256 0
inflate_state: 9544 0
ext2_sb_info: 8448 2
tss_struct: 8320 0
```

第二个数字代表对齐的数量
结构中的孔。
是的，有些很大，甚至作者都对前几个的大小印象深刻，这是一个
使用此工具来查找区域的常见方法
可以在减少数据结构大小方面获得一些帮助。所以
下一步将是精心打印此特定结构，

```none
hid_local：
$ pahole −C hid_local vmlinux
/∗ <175c261>
∼/net-2.6.22/include/linux/hid.h:300 ∗/
struct hid_local {
uint usage[8192]; // 0 32768
// cacheline 512 boundary (32768 bytes)
uint cindex[8192]; // 32768 32768
// cacheline 1024 boundary (65536 bytes)
uint usage_index; // 65536 4
uint usage_minimum; // 65540 4
uint delimiter_depth; // 65544 4
uint delimiter_branch;// 65548 4
}; /∗ size: 65552, cachelines: 1025 ∗/
/∗ last cacheline: 16 bytes ∗/
```

因此，这确实是要调查的东西，而不是错误
在Pahole。
如前所述，第二列是
对齐孔。通过此列进行排序提供了正在分析的项目的另一张图片
帮助寻找进一步工作的领域：

```none
$ pahole −−sizes vmlinux | sort −k3 −nr
| head
net_device: 1664 14
vc_data: 432 11
tty_struct: 1312 10
task_struct: 1856 10
request_queue: 1496 8
module: 16960 8
mddev_s: 672 8
usbhid_device: 6400 6
device: 680 6
zone: 2752 5
```

有很多机会可以使用 - 团结一致
结果，但是在某些情况下，这是不正确的，因为
孔是由于指定的成员对齐约束
由程序员。
例如，当一组一组时，需要对齐提示
结构中的字段是“大部分读”，而其他字段则是
定期写信。因此，使得更有可能
“主要阅读” Cachelines并未被写入
SMP机器，属性用于结构成员，指示编译器对齐某些成员处
Cacheline边界。
这是在Linux内核中的一个示例，在结构Net_device上的对齐暗示出现在
上述输出：

```none
/∗
∗ 缓存线主要用于接收
∗ path (including eth_type_trans())
∗/
struct list_head poll_list
____cacheline_aligned_in_smp;
If we look at the excerpt in the pahole output for this
struct where poll_list is located we will see one of
the holes:
/∗ cacheline 4 boundary (256 bytes) ∗/
void ∗dn_ptr; /∗ 256 8 ∗/
void ∗ip6_ptr; /∗ 264 8 ∗/
void ∗ec_ptr; /∗ 272 8 ∗/
void ∗ax25_ptr; /∗ 280 8 ∗/
/∗ XXX 32 bytes hole, try to pack ∗/
/∗ cacheline 5 boundary (320 bytes) ∗/
struct list_head poll_list;
/∗ 320 16 ∗/
```

40 • 7个矮人：在GDB之外调试信息
这些注释在
矮化信息，所以当前 --reorganize
算法不能精确。一个想法是使用
矮标签，每个文件和行的位置
成员解析源代码寻找对齐方式
注释模式，但这尚未尝试。
已经说过以前可能的不准确性
--reorganize 算法，使用仍然很有趣
它在对象文件中的所有结构中都显示
算法成功找到一个的结构
保存字节的新布局。
使用上述内核图像作者找到了165
可以组合孔以节省一些字节的结构。
发现最大的节省是：

```none
$ pahole −−packable vmlinux | sort −k4
−nr | head
vc_data 432 176 256
net_device 1664 1448 216
module 16960 16848 112
hh_cache 192 80 112
zone 2752 2672 80
softnet_data 1792 1728 64
rcu_ctrlblk 128 64 64
inet_hashinfo 384 320 64
entropy_store 128 64 64
task_struct 1856 1800 56
```

这些列为：结构名称，当前大小，重组大小和保存字节。在上面的结构列表中
从源代码检查中，只有少数几个不明确
具有任何明确的对齐约束。需要进一步分析以验证明确的约束是否为
在子系统的发展之后仍然需要
如果确实需要隔离孔，请使用此类结构
成员组或可以重复使用。
这--expand 选项可用于分析崩溃
垃圾场，可用线索是从
复杂的结构，需要乏味的手动计算
确切找出涉及的领域。它起作用
“展开”结构，如下所示
例子。
在具有以下结构的程序中：

```c
struct spinlock {
int magic;
int counter;
};
struct sock {
int protocol;
struct spinlock lock;
};
struct inet_sock {
struct sock sk;
long daddr;
};
struct tcp_sock {
struct inet_sock inet;
long cwnd;
long ssthresh;
};
```

- expand选项，应用于tcp_sock结构，生产：

```none
struct tcp_sock {
struct inet_sock {
struct sock {
int protocol; /∗ 0 4 ∗/
struct spinlock {
int magic; /∗ 4 4 ∗/
int counter; /∗ 8 4 ∗/
} lock; /∗ 4 8 ∗/
} sk; /∗ 0 12 ∗/
long daddr; /∗ 12 4 ∗/
} inet; /∗ 0 16 ∗/
long cwnd; /∗ 16 4 ∗/
long ssthresh; /∗ 20 4 ∗/
}; /∗ size: 24 ∗/
```

偏移相对于顶级结构的开始
（上述示例中的tcp_sock）。
3.2 pfunct
虽然Pahole专门研究数据结构，但
专注于功能的各个方面，例如：
•Goto标签的数量
•功能名称长度
•参数数
•功能的大小
•变量数量
2007 Linux研讨会，第二卷•41
•内联扩展的大小
它还具有显示符合多个标准的功能的过滤器，包括：
•具有作为参数指示的功能
结构
•外部功能
•被宣布为内联，不在编译器上
•未宣布为编译器的内联声明
此外，还有一组统计信息，例如数字
在时间的时候扩展了内联函数和总和
在这些扩展中，有助于寻找候选人的裁员，从而减少了二进制的大小。
大小的前十个功能：

```none
$ pfunct −−sizes vmlinux | sort −k2 −nr
| head
hidinput_connect: 9910
load_elf32_binary: 6793
load_elf_binary: 6489
tcp_ack: 6081
sys_init_module: 6075
do_con_write: 5972
zlib_inflate: 5852
vt_ioctl: 5587
copy_process: 5169
usbdev_ioctl: 4934
```

DW_AT_SUBPROGRM的属性之一
DWARF 标签，代表函数，is DW_AT_
内联，可以具有以下值之一：
•dw_inl_not_inlined  - 编译器既不是内联声明也不被嵌入
•DW_INL_INLIND  - 未在编译器上串制，而是在编译器上划分
•DW_INL_DECLARED_NOT_INLIEN  -  inline，但没有由编译器划分
•dw_inl_declared_inlined  - 编译器宣布并串联
The --cc_inlined and --cc_uninlined 选项
使用此信息。这里有些例子
未明确标记为内联的功能
程序员，但被GCC绑架：

```none
$ pfunct −−cc_inlined vmlinux | tail
do_initcalls
do_basic_setup
smp_init
do_pre_smp_initcalls
check_bugs
setup_command_line
boot_cpu_init
obsolete_checksetup
copy_bootdata
clear_bss
```

为了完整性，内部函数的数量为
2526.
3.3 codiff
对象文件差异工具Codiff采用两个版本的
二进制，从两个文件加载调试信息，
比较它们并显示结构和
功能，产生类似于已知的输出
差异工具。
考虑一个具有print_tag函数的程序，
处理以下结构：

```c
struct tag {
int type;
int decl_file;
char ∗decl_line;
};
```

在较新的版本中，结构已更改为
新布局，而print_tag函数仍保留
不变：

```c
struct tag {
char type;
int decl_file;
char ∗decl_line;
int refcnt;
};
```

```none
Codiff产生的输出将是：
$ codiff tag−v1 tag−v2
tag.c:
42•7个矮人：在GDB之外调试信息
struct tag | +4
1 struct changed
print_tag | +4
1 function changed, 4 bytes added
```

它类似于差异工具，显示了多少个字节
被添加到修改的结构和此效果
在处理该结构实例的例程中更改。

- Verbose选项告诉我们详细信息：

```none
$ codiff −V tag−v1 tag−v2
tag.c:
struct tag | +4
nr_members: +1
+int refcnt /∗ 12 4 ∗/
type
from: int /∗ 0 4 ∗/
to: char /∗ 0 1 ∗/
```

1个结构更改
print_tag |+4＃29→33
1个功能更改，添加了4个字节
有关修改结构的额外信息包括：
•添加和/或删除的成员数量
•新成员和/或删除成员列表
•从结构开始的成员抵消了成员
•成员的类型和大小
•更改其类型的成员
以及功能：
•尺寸差异
•上一个尺寸 - >新尺寸
•新功能和/或删除功能的名称
3.4 ctracer
类示踪剂，Ctracer是创建的实验
来自矮化信息的有效源代码。
对于ctracer，一种方法是接收到的任何功能
其参数之一是指定结构的指针。它
寻找所有此类方法并生成Kprobes条目和退出功能。在这些探测点，它收集了
有关数据结构内部状态的信息，在该时间点保存其成员中的值，以及
将其记录在继电器缓冲区中。后来收集数据
在用户空间和后处理中，生成HTML + CSS
呼叫图。
Ctracer中使用的一种技术涉及根据某些标准创建数据结构的子集，
例如成员名称或类型。到目前为止，此工具只是滤除了任何非全体类型成员，并应用了
--reorganize 生成的迷你结构上的Code2
可能会减少继电器所需的记忆空间
此信息给用户空间。
可能会追求的一个想法是产生
SystemTap [1]脚本而不是C语言源文件
使用Kprobes，利用基础架构
和在SystemTap中安装的安全警卫。
3.5 pdwtags
一个简单的工具，pdwtags用于矮矮人
数据结构（结构，工会），枚举和枚举和标签
功能，在对象文件中。它作为一个例子有用
如何使用libdwarves。
这是Hello World计划的一个示例：

```none
$ pdwtags hello
/∗ <68> /home/acme/examples/hello.c:2 ∗/
int main(void)
{
}
```

这显示了 “main” DW_TAG_subprogram tag， 和
它的返回类型。
在前几节中
提出了格式，还标记变量的标签，
如果功能参数，goto标签，也将出现
PDWTAGS用于同一对象文件。
在Pahole部分中有2个见名。
2007 Linux研讨会，第二卷•43
3.6 pglobal
pglobal 是打印全球变量和功能的实验，由撰稿人Davi Arnault撰写。
这个示例：

```none
$ cat global.c
int variable = 2;
int main(void)
{
printf("variable=%d\n",
variable);
}
```

将向此输出呈现PGLOBAL：

```none
$ pglobal −ve hello
/∗ <89> /home/acme/examples/global.c:1 ∗/
int变量;
```

它显示了及其类型的全局变量列表，
源代码文件和在定义的位置行。
3.7预先
prefcnt是试图进行参考的尝试
标签，试图在任何地方找到一些未引用的标签，可以从源文件中删除。
4可用性
这些工具是在GIT存储库中维护的，可以
被浏览<http://git.kernel.org/?p>=
linux/kernel/git/acme/pahole.git， 和
可用于几个架构的RPM软件包可在<http://oops.ghostprotocols.net>:
81/acme/dwarves/rpm/.
致谢
作者要感谢Davi Arnault的Pglobal，
证明Libdwarves对于工具来说并不那么恐怖
作家;所有为编写这些工具提供补丁，建议和鼓励的人；
和阿德玛·里斯（Ademar Reis），阿里斯通·罗赞斯基（Aristeu Rozanski），克劳迪奥·马苏卡（Claudio Matsuoka），格劳伯·科斯塔（Glauber Costa），爱德华多·哈博斯特（Eduardo Habkost），尤金·蒂奥（Eugene Teo），
Leonardo Chiquitto，Randy Dunlap，Thiago Santos和
威廉·科恩（William Cohen）审查并建议改进本文的几种草案。

## 参考

[1] Systemtap.
<http://sourceware.org/systemtap>.
[2] Ulrich Drepper. elfutils home page.
<http://people.redhat.com/drepper>.
[3] DWARF Debugging Information Format
Workgroup. Dwarf debugging information format,
December 2005. <http://dwarfstd.org>.


