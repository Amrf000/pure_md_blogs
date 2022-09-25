
# 原文 使用pdbparse获取Windows pdb文件中的类型信息

[Func-Prototypes-With-Pdbparse](https://auscitte.github.io/posts/Func-Prototypes-With-Pdbparse)

## 前言

最近，需要从Microsoft提取功能原型的方法 **_pdb_** 文件，一种可以在Linux下使用的方法，最好是以Python脚本的形式使用。我已经在使用[Brendan Dolan-Gavitt’s python library](https://github.com/moyix/pdbparse) 对于解析PDB文件，需要完成的只是将其扩展到处理原型相关信息的代码。在上面时，我将检索全局变量声明和结构定义添加到了堆中。这篇文章将带您完成我采取的步骤，以逆转PDB文件格式并提出可能的实现之一。也就是说，这里提供的不是一个完整的实施，而只是一个旨在帮助您开始的演示，如果您面临类似的任务。这也是我决定不向GitHub上的PDBPARSE存储库提交拉的请求的原因。事不宜迟，让我们开始。
任何认为源代码比记录它的一千个单词要好的人，可以通过单击直接进行实施 [this link](https://gist.github.com/Auscitte/37aa7b2d3be058cb6b4d5b8b4c13477a).

## 介绍

根据[Wikipedia](https://en.wikipedia.org/wiki/Program_database), **_Program Database (PDB)_** 是由Microsoft开发的专有文件格式，用于存储与调试相关的数据，例如类型，可变名称和地址，表，将二进制指令链接到源文件中的二进制指令等的表等。此数据是从源代码和链接的阶段提取的。将其包装到一个文件中，其名称与要构建的可执行文件（应用程序或动态链接库）和.pdb扩展名（除非编译器和链接器选项另有指定）。此类文件通常称为“符号文件”。

该格式可能是专有的，但是存在很长时间之后，它引发了一些实质性的反向工程工作：值得注意的是，Sven Schreiber在其“无证件的Windows 2000秘密”中描述了PDB的早期版本，然后有几个解析器作为独立库或软件包的一部分实现。最后，五年前，微软制作了创建PDB文件开源的代码（部分），因此可能会争辩该格式是否仍然可以称为专有。

我建议浏览Krzystof Kowalczyk在他的[“pdb format”](https://blog.kowalczyk.info/article/4ac18b52d1c4426185d0d69058ff9a62/pdb-format.html)在深入研究符号文件的内部工作之前发布。关于该主题的文章 [Oleg Starodumov](http://www.debuginfo.com) 也强烈推荐。我只会重申参考材料中发现的一些兴趣点：

* 符号文件由任意数量的单独（独立）组成**_streams_** 可以将其视为文件中的文件。在较低级别，每个流都进一步细分为固定尺寸（通常为4 kb）的_pages_，因此蒸汽可以在符号文件中占据非连续区域，并将各种蒸汽的页面混合在一起。这种结构本质上与NTFS文件的结构相似，并允许同时运行多个独立作家。
* 在较高级别上，该流是连续的，并且由记录组成，每个记录都遵循以下格式：〈record length〉〈record type〉〈record body〉，因此，一个带有有关长度和类型的信息的解析器可以安全地跳过其不了解的记录。正如Schreiber所指出的那样，这种记录结构是从古代OMF格式继承的，用于16位DOS对象文件。
* 在PDB文件中，有一组执行特定功能的预定义流。其中包括PDB，调试信息（DBI），类型信息（TPI）和全局符号流。这些流中的一些通过固定索引识别，而其他流的索引因文件而异，并存储在标题中。**_PDB stream_** 保留将符号文件与生成的可执行文件以及确定PDB中包含哪些功能的标志所需的信息。  **_DBI stream_** 提供有关编译的信息 (object files) 链接在一起以生成生成的可执行文件和相应的源文件。 **_TPI stream_** 包含类型定义其他流通过索引引用。最后，**_Global Symbols stream_** 列表符号在对象文件的边界上可见（所谓的 “symbols with linkage”) 例如全球变量。可以找到有关流类型的更多信息 [here](https://llvm.org/docs/PDB/index.html).

在继续之前，我还应该提到一件事。使用符号文件，很可能会遇到陷阱：其中包含的信息类型取决于生成其生成的开发工具链的版本（多年来，调试信息的存储方式经历了重大更改）和编译器/链接器设置用于以可调的方式省略/包括数据，这可能会“打破”某些解析。这也涉及我的代码。你被警告了。

## 重要版本信息

下面列出的是我在计算机上安装的PDBPARSE及其依赖项的版本。出于明显的原因，该帖子可能与其他版本的PDBParse无关。人们希望，在不久的将来，一些仁慈的灵魂将捐赠代码扩展图书馆，从而丢失了功能，从而使这里写下的所有内容都毫无用处。眨眼眨眼。

```python title="Versions of pdbparse and its Dependencies"
 
ubuntu@ubuntu:~$ pip3 list
Package                Version

---

construct              2.9.52
pdbparse               1.5
pefile                 2019.4.18
```

## 运行示例

无证件的文件格式（尤其是像PDB一样丰富的功能）可能看起来像是纠结的混乱，既不是头部，也不是尾巴。为了进行逆向工程，需要一些小型易于管理的东西，因此我使用Visual Studio来创建一个简单的控制台应用程序。这是源代码：

```c title="hiworld.cpp: 运行示例"
#include "stdafx.h"
#include <windows.h>

#define MAX_LEN 255

struct TextHolder {
WCHAR szBuffer[MAX_LEN];
DWORD dwLen;
} g_Message;

DWORD store_message(TextHolder* pBuf, LPCWSTR szMessage)
{
DWORD dwMaxLen = sizeof(TextHolder::szBuffer) / sizeof(TextHolder::szBuffer[0]);
wcscpy_s(pBuf->szBuffer, dwMaxLen, szMessage);
return (pBuf->dwLen = wcslen(szMessage));
}

int main()
{
store_message(&g_Message, L"Hello, World!");
return 0;
}
```

该程序足够简单，以至于不需要任何解释，因此让我们继续构建它。就符号文件生成而言，我正在使用的开发环境（VS 2015）提供了三个选项：程序数据库Edit And Continue (/ZI), Program Database (/Zi), C7-compatible (/Z7) – 以及其他一些调整，例如排除特定符号。如果您想了解有关这些选项的更多信息，我建议阅读Microsoft的文档和 [this](http://www.debuginfo.com/articles/gendebuginfo.html)文章_DebugInfo.com_但是，出于我们的目的，（三个可用的三个）都没关系您选择的调试信息格式 - 无论如何，结果将是相同的。

VS2015带有开发工具链版本14，从结果可执行文件的PE标头中可以明显看出：

``` title="Fragments of hiworld.exe's PE Header"
> dumpbin /HEADERS C:\Temp\hiworld\x64\Debug\hiworld.exe
> [...]
> OPTIONAL HEADER VALUES
> 20B magic # (PE32+)
> 14.00 linker version
> [...]
> Debug Directories

```

```none
Time Type        Size      RVA  Pointer
-------- ------- -------- -------- --------
5F877495 cv            3E 0000A868     8E68    Format: RSDS, {0EA205FF-0047-41E8-BAC5-FDA9FFCFB69E}, 1, C:\Temp\hiworld\x64\Debug\hiworld.pdb
5F877495 feat          14 0000A8A8     8EA8    Counts: Pre-VC++ 11.00=0, C/C++=35, /GS=35, /sdl=0, guardN=33
```

```none
[...]
```

观察存储在 `linker version` 场地。同样值得我们注意的是存储在调试目录中的数据，在该数据中，可以找到一对<guid，age>用于将可执行文件与相应的符号文件和.pdb文件的路径匹配。为了完整，让我们看一下使用PDBPARSE获得的PDB标题：

```python title="Signature and Verion Info Stored In hiworld.pdb"
Python 3.8.2 (default, Mar 13 2020, 10:14:16)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.

> > > import pdbparse
> > > pdb = pdbparse.parse("hiworld.pdb")
> > > pdb.STREAM_PDB.Version
> > > 20000404
> > > pdb.signature
> > > b'Microsoft C/C++ MSF 7.00\r\n\x1aDS\x00\x00\x00'
> > > 
```

请注意签名中存在的字母“ DS”；同一字母也以格式名称（“ RSD”）垃圾箱给我们出现。他们确定将调试信息存储在此特定文件中的格式。正如已经提到的那样，多年来的变态和各种幅度的其他变化，调试信息的存储方式已经消失。例如，“无证件Windows 2000 Secrets”中描述的实际上是字符串“ JG”表示的较早格式，并命名为“ Microsoft C/C ++程序数据库2.00”（请参阅Jeremy Gordon的[post](http://www.godevtool.com/Other/pdb.htm) 有关详细信息）。

现在已经建立了所使用的确切工具以确保结果的可重复性，现在该将修改引入PDBPARSE了。

## 临时解决方法

马上，pdbparse试图加载_hiworld.pdb_时，pdbparse抛出了例外。显然，枚举 `leaf_type` 缺少一些常数，当这些不计入值的值时，请取代预期 `contruct.Enum`，创建一个常规整数。当然，它看起来确实是这样，但是，我选择了这次不研究问题（如果可以的话，请进行黑客修复）。以下是运行后终端屏幕的屏幕截图 `diff --color -y -W 100 /usr/local/lib/python3.8/dist-packages/pdbparse/tpi.py tpi.py`

{% include fill-centered-fig.html filename="pdbparse_upd_diff3.png" alt="diff for tpi.resolve_typerefs()"

{% include fill-centered-fig.html filename="pdbparse_upd_diff2.png" alt="diff for tpi.merge_fwdrefs()"

{% include fill-centered-fig.html filename="pdbparse_upd_diff1.png" alt="diff for tpi.rename_2_7()"

不是解决方法，而是针对操作员优先问题的次要解决方案：

 ```python title="Fix for an Operator Precedence Issue"
  
ubuntu@ubuntu:~$ diff /usr/local/lib/python3.8/dist-packages/pdbparse/dbi.py dbi.py
160c160
< Name = "Name" / CString(encoding = "utf8").parse(Names[NameRef[j]:])
----------------------------------------------------------------------

> Name = ("Name" / CString(encoding = "utf8")).parse(Names[NameRef[j]:])
> ```

有了这些微小的修复程序，PDBPARSE已成功加载并解析了我们的主题，_hiworld.pdb_.

一种明智的做法，可确保人们避免受伤（对一个人的自我）开始，这是从热身开始的，在这种情况下，这将实施简单的事情，而这种事情不需要PDBPARSE代码中的更改。这就是阅读结构定义和全局变量声明的任务。

## 结构定义和全局变量

类型是在TPI流（位于.pdb文件内部）中定义的，并由16位整数键进行索引。相应的TPI记录的内存布局取决于类型是什么：数组，结构，联合，指向另一种类型的指针等。我广泛使用了Python的 `dir()` 功能以一一检查各种类型的内部组织。像这样：

```python title="Internal Structure of Arrays in TPI Stream"
> > > f = filter(lambda t: pdb.STREAM_TPI.types[t].leaf_type == "LF_ARRAY",
> > > ...  pdb.STREAM_TPI.types)
> > > list(filter(lambda s: not s.startswith("_"), dir(pdb.STREAM_TPI.types[next(f)])))
> > > ['clear', 'copy', 'element_type', 'index_type', 'items', 'keys', 'leaf_type', 'length', 'name', 'pop', 'popitem', 'search', 'search_all', 'size', 'tpi_idx', 'update', 'values']
> > > 
```

这里搜索了数组声明的记录（我知道有一个事实是因为`TextHolder` 列出了一个数组作为其字段之一），其属性省略了从下划线开始的属性。记下名为_“ element_type”的属性，很可能是指此数组元素的类型。

通过这种方式，类型将其他类型引用为它们的成分，从而形成了有向的无环图（DAG），其边缘体现了引用和顶点 - 类型本身（值得注意的是，该规则在LLVM文档中指出，该规则有例外，但是我不会详细说明）。因此，某种类型的人类可读名称可能是由一个递归遍历DAG的函数形成的，类似于下面的函数。

```python title="get_type_name() Impelementation"
def get_type_name(tp):
#a primitive type does not have a record
if not "tpi_idx" in dir(tp):
return str(tp)
#for structures and unions, just print out the name
if tp.leaf_type == "LF_UNION" or tp.leaf_type == "LF_STRUCTURE":
return tp.name
#a pointer to a known type
if tp.leaf_type == "LF_POINTER":
return get_type_name(tp.utype) + "*"
#handling 'const', 'volatile', and 'unaligned' modifiers
if tp.leaf_type == "LF_MODIFIER":
s = [ mod for mod in ['const', 'volatile', 'unaligned']
if tp.modifier[mod] ]
return " ".join(s) + " " + get_type_name(tp.modified_type)
#only 1D arrays are supported
if tp.leaf_type == "LF_ARRAY":
return get_type_name(tp.element_type) +
"[" + str(int(tp.size / base_type_size[tp.element_type])) + "]"
return "UNKNOWN"
```

当然，这只是一个玩具的例子。在现实生活中，人们必须照顾更多的技术细节。现在我们知道如何获得类型名称，打印结构定义变得容易：

```python  title="print_struct_definition() Impelementation"
def print_struct_definition(pdb, sname):
tps = list(filter(lambda t: 
pdb.STREAM_TPI.types[t].leaf_type == "LF_STRUCTURE"
and pdb.STREAM_TPI.types[t].name == sname,
pdb.STREAM_TPI.types))
if len(tps) == 0:
print("Structure", sname, "is not defined in the tpi stream.")
return
print("struct", sname, "{")
for f in pdb.STREAM_TPI.types[tps[0]].fieldlist.substructs:
print("\t", f.name, ":", get_type_name(f.index))
print("}")
```

鉴于获得人类而不是符合C ++语法的定义（后者需要更多的努力）是我的目标是我的目标，因此实施非常简单。让我们尝试一下！

```python title="print_struct_definition() Demo"
> > > print_struct_definition(pdb, "TextHolder")
> > > struct TextHolder {
> > > szBuffer : T_WCHAR[255]
> > > dwLen : T_ULONG
> > > }
```

打印全局变量声明的任务同样不受欢迎。对于全局变量，它在模块边界上可见，人们希望在“全局符号”流中找到匹配的符号。让我们看看。

```python title="Searching Global Symbols for g_Message"


> > > print(*[ ( hex(s.leaf_type), s.name) for s in pdb.STREAM_GSYM.globals
> > > ...  if "name" in dir(s) and "g_Message" in s.name ], sep="\n")
> > > ('0x110e', '?g_Message@@3UTextHolder@@A')
> > > 
```

驻留在符号之间的是一个可变的名称，可方便地装饰有此变量的类型。**_Symbol decoration_** (aka **_symbol mangling_**) 是将类型信息传递给链接器的一种技术，以进行语义错误检查，当然，可以从操纵名称中提取此类型信息。

```python title="Undecorating Variable Name with pdbparse"


> > > from pdbparse import undname
> > > undname.undname("?g_Message@@3UTextHolder@@A",
> > > ...  flags = undname.UNDNAME_COMPLETE)
> > > 'struct TextHolder g_Message'
> > >
```

干得好。这是您要寻找的变量声明。我告诉你那是小菜一碟。好吧，不完全是。问题在于，熔断计划尚未标准化，因此是编译器依赖性的。此外，.pdb文件中可能根本不存在装饰的符号。我们需要另一种方式。

我记得再见，相当大胆地说，打印变量声明并不需要对PDBPARSE的源代码进行任何修改。正如您将很快看到的那样，我在所有的恶毒中都欺骗了您，我信任的读者。但是一切都在及时。

```python title="Examining an Internal Structure of a Public Symbol"


> > > f = list(filter(lambda s: "name" in dir(s) and "g_Message" in s.name,
> > > ... pdb.STREAM_GSYM.globals))
> > > list(filter(lambda s: not s.startswith("_"), dir(f[0])))
> > > ['clear', 'copy', 'items', 'keys', 'leaf_type', 'length', 'name', 'offset', 'pop', 'popitem', 'search', 'search_all', 'segment', 'symtype', 'update', 'values']
> > > f[0].symtype
> > > 0
> > > hex(f[0].leaf_type)
> > > '0x110e'
> > > 
```

引起我注意的第一件事是缺乏对TPI流的参考。看一下符号属性列表：段和偏移，很可能指向该变量在内存中的位置，其余的是无关紧要的（“ Symtype”看起来很有希望，但事实证明这是其他的）。实际上，这是一个所谓的 **_public symbol_**，如其记录类型所指定(`0x110e`)实际上，它的结构已记录在 [Microsoft’s open-source project](https://github.com/Microsoft/microsoft-pdb)。文件 `/include/cvinfo.h` 是应该寻找相关定义的地方。

```c title="An Excerpt from microsoft-pdb/include/cvinfo.h"
//Symbol definitions
typedef enum SYM_ENUM_e {
// […]
S_PUB32 = 0x110e, // a public symbol (CV internal reserved)
// […]
};

typedef struct PUBSYM32 {
unsigned short  reclen;     // Record length
unsigned short  rectyp;     // S_PUB32
CV_PUBSYMFLAGS  pubsymflags;
CV_uoff32_t     off;
unsigned short  seg;
unsigned char   name[1];    // Length-prefixed name
} PUBSYM32;
```

的确，公共符号没有引用TPI流，但是其他一些符号必须完成，我决定寻找它们。首先，我需要知道全局符号流的索引。

```python title="An Index of the Global Symbols Stream"
> > > pdb.STREAM_DBI.DBIHeader.symrecStream
> > > 8
> > > 
```

然后我雇用了PDBPARSE的 **_pdb_dump_** 实用程序将符号文件剖分为组成流，并搜索关注的字符串。

```shell title="Searching Global Symbols Stream For g_Message"
ubuntu@ubuntu:~$ pdb_dump.py hiworld.pdb
ubuntu@ubuntu:~$ strings hiworld.pdb.008 | grep g_Message
?g_Message@@3UTextHolder@@A
g_Message
```

啊！有一个未装饰的变量版本`g_Message` 隐藏在符号流中的某个地方；但是，PDBPARSE在解析流时以某种方式跳过了与之相关的数据。源代码中的快速浏览使我们了解了为什么会发生：

```python title="An Excerpt from pdbparse/gdata.py"

gsym = Struct(
"leaf_type" / Int16ul, "data" / Switch(
lambda ctx: ctx.leaf_type, {
0x110E:
"data_v3" / Struct(
"symtype" / Int32ul,
"offset" / Int32ul,
"segment" / Int16ul,
"name" / CString(encoding = "utf8"),
),
0x1009:
"data_v2" / Struct(
"symtype" / Int32ul,
"offset" / Int32ul,
"segment" / Int16ul,
"name" / PascalString(lengthfield = "length" / Int8ul,
encoding = "utf8"),
),
}))

GlobalsData = "globals" / GreedyRange(
Struct(
"length" / Int16ul,
"symbol" / RestreamData(Bytes(lambda ctx: ctx.length), gsym),
))
```

PDBPARSE在构造库的帮助下解析流，该库可以以声明性的方式进行。该流是在 `ctx.length` sizes (see `RestreamData()`);两种类型符号的记录(`S_PUB32_ST = 0x1009` and `S_PUB32 = 0x110e`) 仅被识别并完全解析，其余的存储为对

```python title="Listing Types of Symbols Found in Global Symbols Stream"


> > > set([  hex(s.leaf_type) for s in pdb.STREAM_GSYM.globals ])
> > > {'0x1108', '0x1107', '0x110c', '0x110e', '0x1125', '0x110d', '0x1127'}
> > > 
```

或微软的实施：

```c title="An Excerpt from microsoft-pdb/include/cvinfo.h"

//Symbol definitions
typedef enum SYM_ENUM_e {
// […]
S_CONSTANT =  0x1107,  // constant symbol
S_UDT = 0x1108,  // User defined type
S_LDATA32 = 0x110c,  // Module-local symbol
S_GDATA32 = 0x110d,  // Global data symbol
S_PUB32 = 0x110e, // a public symbol (CV internal reserved)
S_PROCREF = 0x1125, // Reference to a procedure
S_LPROCREF = 0x1127, // Local Reference to a procedure
// […]
};
```

其中，由 `S_GDATA32`（全局数据符号）类型似乎正是我想要的，所以我找到了匹配的结构 (`DATASYM32`) in `cvinfo.h` 并增强了PDBPARSE的适当声明：

```python title="Adding construct Declarations for DATASYM32 (in pdbparse/gdata.py)"

gsym = Struct(
"leaf_type" / Int16ul, "data" / Switch(
lambda ctx: ctx.leaf_type, {
#[…]
0x110d: #adapted from struct DATASYM32 in cvinfo.h
"datasym" / Struct(
"typind" / Int32ul,
"offset" / Int32ul,
"segment" / Int16ul,
"name" / CString(encoding = "utf8"),
),
#[…]
}))
```

注意 `typeind` 场地！这是TPI流中的记录，这是整个努力的目标。对负责符号列表进行后处理的函数的一个小修改（如下所示），现在必要的是，不同类型的符号具有不同的属性集，而我很高兴。

{% include fill-centered-fig.html filename="pdbparse_upd_diff4.png" alt="diff for gdata.merge_structures()"

完成了所有准备工作后，我最终可以授予您打印任何全局变量的声明语句的功能。瞧！

```python title="print_variable_declaration() Implementation"

def print_variable_declaration(pdb, vname):
for s in pdb.STREAM_GSYM.globals:
if not "name" in s or s.name != vname:
continue
if not "typind" in s:
print("Found a symbol named", vname,
"but, it did not have an associated type.")
continue
print(get_type_name(pdb.STREAM_TPI.types[s.typind]), " ",
vname, ";", sep = "")
return
print("Could not find variable", sname)
```

This time it will actually work reliably. Check it out!

```python title="print_variable_declaration() Demo"


> > > print_variable_declaration(pdb, "g_Message")
> > > TextHolder g_Message;
> > > 
```

## 功能原型

ph！那是一个相当漫长的话语。我唯一的希望使我们为即将发生的事情做好了准备。

首先，我将基于未装修的最简单（但并非总是可靠的）技术。它就像在全球变量的情况下一样工作。这里没有什么新的。

```python title="Undecorating a Function Name with pdbparse"


> > > from pdbparse import undname
> > > print(*[ ( hex(s.leaf_type), s.name) for s in pdb.STREAM_GSYM.globals
> > > ...  if "name" in dir(s) and "store" in s.name ], sep="\n")
> > > ('0x110e', '?store_message@@YAKPEAUTextHolder@@PEB_W@Z')
> > > undname.undname("?store_message@@YAKPEAUTextHolder@@PEB_W@Z",
> > > ...  flags = undname.UNDNAME_COMPLETE)
> > > 'unsigned long __cdecl store_message(struct TextHolder * __ptr64,wchar_t const * __ptr64)'
> > > 
```

承担了检索功能原型的任务后，我面临着一个基本的哲学问题:-)。我们期望某种类型的某种类型和实例之间存在一对多的“ IS-A”关系。此外，该类型通常由其名称识别，而实例可能会或可能不会给出名称（标识符），并且此名称独立于其类型。但是，在函数（函数指针和界面旁）的情况下，该规则不得不存在。长话短说，功能的TPI记录不包含名称，因此无法通过简单地枚举TPI记录来找到原型。

```python title="Enumerating Prototypes In TPI Stream"

for t in pdb.STREAM_TPI.types:
if pdb.STREAM_TPI.types[t].leaf_type != "LF_PROCEDURE":
continue
formalparams = [ get_type_name(tp)
for tp in pdb.STREAM_TPI.types[t].arglist.arg_type ]
print(hex(t), pdb.STREAM_TPI.types[t].call_conv,
get_type_name(pdb.STREAM_TPI.types[t].return_type),
"(", ", ".join(formalparams), ")")
```

上面给出的脚本可用于列出TPI流中定义的函数原型（我们将讨论限制为全局功能，而成员和静态功能则持续了另一段时间）：

```python title="Enumerating Prototypes In TPI Stream: Output"
 
0x1343 NEAR_C T_VOID ( T_64PVOID )
0x135d NEAR_C T_64PVOID (  )
0x1365 NEAR_C T_INT4 (  )
0x13f3 NEAR_C UNKNOWN ( _EXCEPTION_RECORD*, T_64PVOID, _CONTEXT*, T_64PVOID )
0x149b NEAR_C T_VOID ( _TP_CALLBACK_INSTANCE*, T_64PVOID )
0x14a2 NEAR_C T_VOID ( T_64PVOID, T_64PVOID )
0x1551 NEAR_C T_HRESULT ( tagEXCEPINFO* )
0x16fb NEAR_C T_ULONG ( TextHolder*, const T_WCHAR* )
[…]
```

这些原型包括人们所希望的一切：调用约定，返回值的类型，形式参数的类型。除了名字！如果只知道一个函数名称，就可以在以下一个例程的帮助下获得半衰弱的输出。

```python title="print_function_declaration_from_tpi_by_idx() Implementation"

def print_function_declaration_from_tpi_by_idx(pdb, fname, typind):
if not typind in pdb.STREAM_TPI.types:
print("There is no record with the index",
typind, "in the TPI stream")
return
#not dealing with static and member functions
if pdb.STREAM_TPI.types[typind].leaf_type != "LF_PROCEDURE":
print("The type at", typind, "is not a fuction, but",
pdb.STREAM_TPI.types[typind].leaf_type)
return
formalparams = [ get_type_name(tp) for tp in
pdb.STREAM_TPI.types[typind].arglist.arg_type ]
print(pdb.STREAM_TPI.types[typind].call_conv, " ",
get_type_name(pdb.STREAM_TPI.types[typind].return_type), " ",
fname, "(", ", ".join(formalparams), ")", sep="")
```

在第二次将符号与其在TPI流中连接到其记录的问题之后，我确切地知道该怎么做。在PDB文件中明显缺乏连接，我再也不会感到不安！回想一下，尽管存在于全球符号流中，但被解析器忽略了。其中是具有属性的符号 `leaf_type` equal to `0x1125` (`S_PROCREF`，“参考程序”）。为什么我们不解析它们？等于

```python title="Adding a Parsing Construct for REFSYM2 (in pdbparse/gdata.py)"

gsym = Struct(
"leaf_type" / Int16ul, "data" / Switch(
lambda ctx: ctx.leaf_type, {
#[…]
0x1125: #adapted from struct REFSYM2 defined in cvinfo.h
"proc_ref" / Struct(
"sumname" / Int32ul,
"offset" / Int32ul,
"iMod" / Int16ul,
"name" / CString(encoding = "utf8"),
),
#[…]
}))
```

应用新添加的结构，我们得到：

```python title="Printing Out a List of References to Procedures in Global Symbols"


> > > print(*[ (s.name, s.iMod, hex(s.offset)) for s in pdb.STREAM_GSYM.globals
> > > ...     if s.leaf_type == 0x1125 ], sep="\n")
> > > [...]
> > > ('store_message', 3, '0x440')
> > > ('main', 3, '0x4e8')
> > > 
```

到那时，情况并没有惊人的希望。是的，我找到了符号`store_message()`以及我的源代码中唯一的其他功能 - `main()`，但他们都没有引用TPI蒸汽。我可以安全地推断出的所有内容是，与所讨论的功能有关 `3`，抵消 `0x440`。令人困惑的是，我决定放弃当前的潜在客户，并在整个文件中搜索字符串。

```shell title="Printing Out a List of References to Procedures in Global Symbols"
ubuntu@ubuntu:~$ find -name "hiworld.pdb.*" -type f -print0 | xargs -0 strings -f | grep store_message
./hiworld.pdb.014: store_message
./hiworld.pdb.008: ?store_message@@YAKPEAUTextHolder@@PEB_W@Z
./hiworld.pdb.008: store_message
```

我们已经知道这是由 _index = 8_ 是全球符号流；在流中，有两个相关符号 _leaf_types_`S_PUB32` （和一个被弄糊的名称）和 `S_PROCREF`。那流号码呢_14_? 它可以以某种方式对应于指向的模块_iMod = 3_?

好吧，在正常情况下，14不等于3，但这不是沮丧的原因。模块的概念和LLVM文档所谓的“编译”的概念很可能是一个或相同的。在这种情况下，DBI流应该寻找线索。经过一些折磨后，我发现了这一点：

```python title="An Interesing Header in the DBI Stream"


> > > pdb.STREAM_DBI.DBIExHeaders[2]
> > > Container(opened=0, range=Container(section=2, offset=1680, size=130, flags=1615859744, module=2, dataCRC=3438623728, relocCRC=1000694769), flags=2, stream=14, symSize=1336, oldLineSize=0, lineSize=504, nSrcFiles=16, offsets=25059472, niSource=56, niCompiler=12, modName=u'C:\\Temp\\hiworld\\hiworld\\x64\\Debug\\hiworld.obj', objName=u'C:\\Temp\\hiworld\\hiworld\\x64\\Debug\\hiworld.obj')
> > > 
```

I guess the module number (3 vs. 2) discrepancy is due to the fact that iMod counts modules starting from one whereas `DBIExHeaders` indices are zero-based... What should really capture our attention here is the attribute `stream` with a value of `14`. Why do we not peer inside the mysterious 14th stream?

{% include fill-centered-fig.html filename="pdbparse_hexdump14_1.png" alt="hexdump of hiworld.pdb.014"

It seems to have the same structure as the rest of them, with records preceded by their lengths and types. Look! There is `S_OBJNAME = 0x1101` (“path to object file name”) magic number from Microsoft’s `cvinfo.h` and what follows, indeed, looks very much like a path to an object file. Let us scroll down to the offset specified by `message_store`’s “global” symbol (`0x440`).

{% include fill-centered-fig.html filename="pdbparse_hexdump14_2.png" alt="hexdump of hiworld.pdb.014"

To all appearances, we are about to deal with a 0x0036 bytes-long record of type `S_GPROC32`  (0x1110), otherwise known as “global procedure start.” The matching C++ structure is given below.

```c title="An Excerpt from microsoft-pdb/include/cvinfo.h"
typedef struct PROCSYM32 {
unsigned short  reclen;     // Record length
unsigned short  rectyp;     // S_GPROC32, S_LPROC32, S_GPROC32_ID, S_LPROC32_ID, S_LPROC32_DPC or S_LPROC32_DPC_ID
unsigned long   pParent;    // pointer to the parent
unsigned long   pEnd;       // pointer to this blocks end
unsigned long   pNext;      // pointer to next symbol
unsigned long   len;        // Proc length
unsigned long   DbgStart;   // Debug start offset
unsigned long   DbgEnd;     // Debug end offset
CV_typ_t        typind;     // Type index or ID
CV_uoff32_t     off;
unsigned short  seg;
CV_PROCFLAGS    flags;      // Proc flags
unsigned char   name[1];    // Length-prefixed name
} PROCSYM32;
```

Do you see what I see? The `typind` field! It appears to be a conventional name for an index in the TPI stream. Armed with this knowledge, I wrote a little python script that, given a name, would find and print out a prototype for a function with this name employing the same technique for parsing data as pdbparse had done. Here it is.

```python title="print_function_declaration_from_tpi() Implementation"

import construct as cs
GlobalProcSym = "PROCSYM32" / cs.Struct(
"reclen" / cs.Int16ul,
"rectyp" / cs.Int16ul,
"pParent" / cs.Int32ul,
"pEnd" / cs.Int32ul,
"pNext" / cs.Int32ul,
"len" / cs.Int32ul,
"DbgStart" / cs.Int32ul,
"DbgEnd" / cs.Int32ul,
"typind" / cs.Int32ul,
"offset" / cs.Int32ul,
"seg" / cs.Int16ul,
"flags" / cs.Int8ul,
"name" / cs.CString(encoding = "utf8"),
)

def print_function_declaration_from_tpi(pdb, fname):
fncs = list(filter(lambda s: s.leaf_type == S_PROCREF and
s.name == fname, pdb.STREAM_GSYM.globals))
if len(fncs) == 0:
print("There is no S_PROCREF-type reference to",
fname, "in the global symbols stream.")
return
#Indices given by iMod are 1-based while pdb.STREAM_DBI.DBIExHeaders[]
#is a standard python list with 0-based indexing
data = pdb.streams[pdb.STREAM_DBI.DBIExHeaders[
fncs[0].iMod -1].stream].data
fn = GlobalProcSym.parse(data[fncs[0].offset:],
entry_offest = fncs[0].offset)
print_function_declaration_from_tpi_by_idx(pdb, fname, fn.typind)
```

First it looks for a “reference to procedure” record in the global symbols streams then uses its fields `iMod` and offset to locate a module stream and region of memory within it which is later parsed with construct. Take a look at `print_function_declaration_from_tpi` in action.

```python title="print_function_declaration_from_tpi() Demo"


> > > print_function_declaration_from_tpi(pdb, "store_message")
> > > NEAR_C T_ULONG store_message(TextHolder*, const T_WCHAR*)
> > > 
```

此声明有什么问题（除了晦涩的召集公约名称外）？没有什么错：这是一个完全可以接受的原型，但是人们不禁会觉得可以通过添加正式参数的名称来相当改进。出现这种感觉的原因是，在流垃圾场中，正式参数的名称要抓住，清晰可见。只需要推断内部组织。很容易发现表格的魔术数字 `0x11??` 和 `0x10??` 散布在“ store_message”和“ main”字符串之间的整个区域（后者显然指定了相关内存的边界）。我已经复制了与您接下来的讨论乐趣的相应定义。

```c title="An Excerpt from microsoft-pdb/include/cvinfo.h"
typedef enum SYM_ENUM_e {
[...]
S_CALLSITEINFO = 0x1139, // Indirect call site information
S_FRAMEPROC = 0x1012, // extra frame and proc information
S_REGREL32 = 0x1111, // register relative address
S_END = 0x0006
[...]
};
```

所以`PROCSYM32` 紧随其后的是一些额外的堆栈框架信息和零或更多寄存器相关的地址，每个函数参数和本地变量一个，然后获取呼叫瞄准器列表。我宣布了这些实体中每个实体的“构造”，以防将来需要这些（遵循在`cvinfo.h`)。开始了。

```python title="Construct Declarations for Parsing FRAMEPROCSYM, REGREL32, CALLSITEINFO"

ProcFrameData = cs.Struct(
"rectyp" / cs.Enum(cs.Int16ul, S_FRAMEPROC = 0x1012, S_CALLSITEINFO = 0x1139, S_REGREL32 = 0x1111),
"reminder" / cs.Switch(
lambda ctx: ctx.rectyp, {
"S_FRAMEPROC":
"FRAMEPROCSYM" / cs.Struct(
"cbFrame" / cs.Int32ul,
"offPad" / cs.Int32ul,
"cbSaveRegs" / cs.Int32ul,
"offExHdlr" / cs.Int16ul,
"flags" / cs.Int32ul,
),
"S_REGREL32":
"REGREL32" / cs.Struct(
"off" / cs.Int32ul,
"typind" / cs.Int32ul,
"reg" / cs.Int16ul,
"name" / cs.CString(encoding = "utf8"),
),
"S_CALLSITEINFO":
"CALLSITEINFO" / cs.Struct(
"off" / cs.Int32ul,
"sect" / cs.Int16ul,
"__reserved_0" / cs.Int16ul,
"typind" / cs.Int32ul,
),
}))
```

One cannot determine with certainty whether this list is exhaustive or not, but luckily, there is no need to do so as the “length” field allows for record skipping, which, in turn, is implemented with the help of `RestreamData` class. Thanks to `GreedyRange` a sequence of an arbitrary number of `REGREL32` and `CALLSITEINFO` instances, as many as can fit into the given region of memory, is parsed.

```python title="Construct Declaration to Parse a Sequence of FRAMEPROCSYM, REGREL32, CALLSITEINFO"

ProcFrameEntries = cs.GreedyRange(
cs.Struct(
"reclen" / cs.Int16ul,
"frame_entry" / cs.RestreamData(cs.Bytes(lambda ctx: ctx.reclen),
ProcFrameData),
)
)
```

Now we have to “confine” the chunk of stream parsed by `ProcFrameEntries`, i.e determine where the data relating to `store_message()` end and `main()`’s `PROCSYM32` structure begins. It turns out, `pEnd` field (in `PROCSYM32` structure) points to the data immediately following the last instance of `CALLSITEINFO`.  In our case, it is a 32-bit value `0x00060002` whose meaning I have not been able to figure out. Obviously, it is an end-of-something marker(s), what is more, the constant `S_END = 0x0006` is documented in the Microsoft's header; as for `0x0002`, however, I have not found anything meaningful in the current context. But does it even matter?

```python title="Construct Declaration to Parse Function-Related Data in Module Stream"

GlobalProc = cs.Struct(
"PROCSYM32" / cs.Struct(
"reclen" / cs.Int16ul,
"rectyp" / cs.Int16ul,
"pParent" / cs.Int32ul,
"pEnd" / cs.Int32ul,
"pNext" / cs.Int32ul,
"len" / cs.Int32ul,
"DbgStart" / cs.Int32ul,
"DbgEnd" / cs.Int32ul,
"typind" / cs.Int32ul,
"offset" / cs.Int32ul,
"seg" / cs.Int16ul,
"flags" / cs.Int8ul,
"name" / cs.CString(encoding = "utf8"),
),
#making sure the entire length of PROCSYM32 has been parsed
cs.Padding(lambda ctx: ctx.PROCSYM32.reclen +
cs.Int16ul.sizeof() - ctx._io.tell()),
"frame_data" / cs.RestreamData(
# ctx.PROCSYM32.pEnd points to the region immediately following
#the last element of ProcFrameEntries;
# ctx.PROCSYM32.reclen does not include the reclen field
#hence the cs.Int16ul.sizeof() correction
cs.Bytes(lambda ctx: ctx.PROCSYM32.pEnd - ctx._params.entry_offest -
ctx.PROCSYM32.reclen - cs.Int16ul.sizeof()),
ProcFrameEntries
)
)
```

在我们写下最终变体之前，必须解决一个问题 `print_function_declaration()` 常规。考虑到构造的方式， `ProcFrameEntries` 将具有嵌套结构：

```none
{
reclen
frame_entry {
rectyp
reminder {
cbFrame
...
}
}
}

```

为了方便起见，需要扁平。

```python title="A Postprocessing Step"

def flatten_frame_data(cont):
fd = cs.lib.ListContainer()
for c in cont:
dc = cs.lib.Container()
dc["reclen"] = c.reclen
dc["rectyp"] = c.frame_entry.rectyp
for k in c.frame_entry.reminder:
if k.startswith("_"):
continue
dc[k] = c.frame_entry.reminder[k]
fd.append(dc)
return fd
```

有趣的是，我找不到区分`REGREL32` 功能参数和本地变量的实例。因为前者似乎总是第一位出现，因此，argument-list列表的标记将有所帮助（这种价值甚至在cvinfo.h: `S_ENDARG = 0x000a`)但是，在检查流垃圾场时，找不到任何地方。因此，假设该函数的形式参数始终出现在第一位，从头到尾列出，我只需使用TPI流中的“参数数”值。该方法不能保证适用于所有呼叫惯例，并且也没有处理可变数量的参数案例。在这种实施“准备就绪”之前，这是很长的路要走。

什么是Regrel32的列表？正如已经提到的那样， `REGREL32` 代表相对于某些寄存器的变量/函数参数的地址，因此可以将其定位在呼叫堆栈上，目前我们毫无兴趣。有趣的是，它包含一个TPI索引和参数/变量名称，除了上述参数/变量为原始类型的情况。原始类型在TPI流中没有记录，而是由数字常数识别。这些常数变成`base_type` 枚举（定义在 _pdbparse.tpi_ 名称空间）由于它们被pdpbarse解析，这是他们期望采取的形式`get_type_name()` 我之前实施的功能。因此，我们必须模仿此步骤，以便获得任何类型的字符串表示。

将所有这些放在一起，我们得到：

```python title="The Ultimate print_function_declaration()"

from pdbparse import tpi
def print_function_declaration_from_mods_stream_named_params(pdb, fname):
fncs = list(filter(lambda s: s.leaf_type == S_PROCREF and
s.name == fname, pdb.STREAM_GSYM.globals))
if len(fncs) == 0:
print("There is no S_PROCREF-type reference to", fname,
"in the global symbols stream.")
return

```python
data = pdb.streams[pdb.STREAM_DBI.DBIExHeaders[\
    fncs[0].iMod - 1].stream].data 
fn = GlobalProc.parse(data[fncs[0].offset:], entry_offest = fncs[0].offset)
if not fn.PROCSYM32.typind in pdb.STREAM_TPI.types:
    print("There is no type record for", fname,\
        "( PROCSYM32.typind =", fn.PROCSYM32.typind, ") in the TPI stream")
    return

tp = pdb.STREAM_TPI.types[fn.PROCSYM32.typind]
paramcnt = tp.arglist.count
paramregs = list(filter(lambda k: k.rectyp == "S_REGREL32",\
    flatten_frame_data(fn.frame_data)))[0:paramcnt]
params = [ get_type_name(pdb.STREAM_TPI.types[e.typind]\
    if e.typind in pdb.STREAM_TPI.types\
    else tpi.base_type.parse(e.typind.to_bytes(16, byteorder='little'))) +\
        " " + e.name for e in paramregs ]

print(tp.call_conv, " ", get_type_name(tp.return_type), " ",\
    fname, "(", ", ".join(params), ")", sep = "")
```

最后，我可能会为欣赏我的劳动成果而高兴。加入并看！呵呵。

```python title="print_function_declaration() Demo"


> > > print_function_declaration_from_mods_stream_named_params(pdb,
> > > ... "store_message")
> > > NEAR_C T_ULONG store_message(TextHolder* pBuf, const T_WCHAR* szMessage)
> > > 
```

## 告别

散布在整个文本中的源代码的位（不包括PDBPARSE本身的更改） [here](https://gist.github.com/Auscitte/37aa7b2d3be058cb6b4d5b8b4c13477a).

我希望这个小帖子能为您节省一个或两个小时的杂物，并在十六进制和源代码中为您节省。对于我来说，这使我有机会在分享交易技巧的方式中展示有用的反向工程技术。

– Ry Auscitte

## References

1. Program Database, Wikipedia, available at: [https://en.wikipedia.org/wiki/Program_database](https://en.wikipedia.org/wiki/Program_database)
2. Jeremy Gordon, The RSDS pdb format, available at [http://www.godevtool.com/Other/pdb.htm](http://www.godevtool.com/Other/pdb.htm)
3. Oleg Starodumov, Generating debug information with Visual C++ , available at [http://www.debuginfo.com/articles/gendebuginfo.html](http://www.debuginfo.com/articles/gendebuginfo.html)
4. Oleg Starodumov, Matching Debug Information, available at [http://www.debuginfo.com/articles/debuginfomatch.html](http://www.debuginfo.com/articles/debuginfomatch.html)
5. Krzysztof Kowalczyk, pdb format, available at: [https://blog.kowalczyk.info/article/4ac18b52d1c4426185d0d69058ff9a62/pdb-format.html](https://blog.kowalczyk.info/article/4ac18b52d1c4426185d0d69058ff9a62/pdb-format.html)
6. Information from Microsoft about pdb format, available at: [https://github.com/Microsoft/microsoft-pdb](https://github.com/Microsoft/microsoft-pdb)
7. LLVM Compiler Infrastructure: The PDB File Format, available at: [https://llvm.org/docs/PDB/index.html](https://llvm.org/docs/PDB/index.html)
8. MSVC Compiler Reference: /Z7, /Zi, /ZI (Debug Information Format), available at
   [https://docs.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format](https://docs.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format)
9. Sven B. Schreiber, 2001, Undocumented Windows 2000 secrets: a programmer's cookbook, Addison-Wesley Longman Publishing Co., Inc., USA.
