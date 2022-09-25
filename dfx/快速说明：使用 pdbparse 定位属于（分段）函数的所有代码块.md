[原文](https://auscitte.github.io/posts/Code-Fragments-With-Pdbparse)

## 使用 pdbparse 定位属于（分段）函数的所有代码块

我认为，开始这篇文章的最佳方式是引用 Sven B. Schreiber 的“Undocumented Windows 2000 Secrets”中的一句话：

> 通常，编译器倾向于将函数的代码保存在一个整体块中，并且不会拆分 if/else 分支。然而，在 Windows 2000 内核模块中，可以很容易地观察到具有大量 if/else 分支的大型函数非常分散。[…] 我的假设是这种拆分应该有助于处理器的指令预取。[…] 如果将不太频繁执行的分支与更频繁使用的分支分开，CPU 可以执行更有效的指令预取。

使用 Windows 10 系统库已经有一段时间了，我可以确认它们中的长函数也被拆分为不相邻的指令块。假设出现了从二进制模块中恢复所有此类块（属于特定功能）的任务。当然，有了一个好的反汇编程序，可以通过遵循函数体内的各种跳转指令来重建控制流图。然而，对于 Windows 模块，看到它们带有通常包含有关代码片段/功能关联信息的*符号文件，有一种更简单的方法。*

目前，我正在使用一个名为[***pdbparse***](https://github.com/moyix/pdbparse)*的 python 库从pdb 文件* 中提取数据。独立于 Microsoft 的 API，它允许在任何运行 python 解释器的操作系统下这样做。在这篇简短的文章中，我将展示如何在给定函数名称的情况下检索包含该函数的所有代码片段的地址。

首先，我们必须从全局符号流中获取与所讨论的函数相对应的“对过程的引用”符号。“对过程的引用”符号将指向模块流，可以在其中找到与函数有关的附加信息。如果以上任何一个对你来说听起来像是一首 Auyokawa 诗，我建议你认真阅读这篇[文章](https://auscitte.github.io/systems%20blog/Func-Prototypes-With-Pdbparse)。

下面是一个*模块流* 的 hexdump 的摘录，它对应`ServerDllImplementation()`于 Windows 的*basesrv.dll* 中定义的 compiland 。

![模块流的 hexdump](https://auscitte.github.io/resources/images/fragmods-dump.png)对于那些熟悉我提到的帖子的人来说，它应该看起来非常熟悉。观察`S_GPROC32 = 0x1110`（在[cvinfo.h](https://github.com/microsoft/microsoft-pdb/blob/master/include/cvinfo.h)中定义）表示`PROCSYM32`结构的开始和符号块结束标记*0x00060002* 。 `PROCSYM32`可用于定位第一个代码段。看一看。

来自 microsoft-pdb/include/cvinfo.h 的摘录

```c
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

该对`〈seg : off〉`指的是函数代码所在的 PE 部分（很可能是*.text ）的偏移量。* 但是，它只会给我们第一个片段。为了获得其余部分，应该寻找结构`S_SEPCODE = 0x1132`后面的标记`PROCSYM32`（并且可能还有一些与当前过程符号相关的可选调试信息）。`pEnd`指示当前`PROCSYM32`（和附加数据）在哪里结束。

来自 microsoft-pdb/include/cvinfo.h 的摘录

```c
// Separated code (from the compiler) support
S_SEPCODE       =  0x1132,
```

显然，Microsoft 将此类代码片段称为***“分隔代码”*** ，并且在长函数的主体中可能不止一个。

来自 microsoft-pdb/include/cvinfo.h 的摘录

```c
typedef struct SEPCODESYM {
    unsigned short  reclen;     // Record length
    unsigned short  rectyp;     // S_SEPCODE
    unsigned long   pParent;    // pointer to the parent
    unsigned long   pEnd;       // pointer to this block's end
    unsigned long   length;     // count of bytes of this block
    CV_SEPCODEFLAGS scf;        // flags
    CV_uoff32_t     off;        // sect:off of the separated code
    CV_uoff32_t     offParent;  // sectParent:offParent of the enclosing scope
    unsigned short  sect;       //  (proc, block, or sepcode)
    unsigned short  sectParent;
} SEPCODESYM;
```

`〈seg : off〉`与这对类似，`〈sect : off〉`它为我们提供了位置，而该`length`字段告诉我们此代码片段的结束位置。因此，行动计划如下：

1. 解析`PROCSYM32`；
2. 跳到`PROCSYM32`'s 块的末尾（关于参数、局部变量等的可选调试信息）；
3. 定位`S_SEPCODE`，如果找到，解析包含的`SEPCODESYM`结构；
4. 如果成功，重复第**3** 步。

瞧！

构造用于解析 PROCSYM32 的声明，后跟 (SEPCODESYM)*

```python
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
    #the stream starts at ctx._params.entry_offest offset in an input file, whereas ctx.PROCSYM32.pEnd is
    #relative to the beginning of the file; cs.Int32ul accounts for the end-of-sequence marker
    cs.Padding(lambda ctx: ctx.PROCSYM32.pEnd - ctx._params.entry_offest -\
        ctx._io.tell() + cs.Int32ul.sizeof()),
    "sepcodesyms" / cs.GreedyRange(
        "SEPCODESYM" / cs.Struct(
            "reclen" / cs.Int16ul,
            "rectyp" / cs.Const(S_SEPCODE, cs.Int16ul), #range over all records with rectyp = S_SEPCODE 
            "pParent" / cs.Int32ul, # pointer to the parent
            "pEnd" / cs.Int32ul,    # pointer to this block's end
            "length" / cs.Int32ul,  # count of bytes of this block
            "scf" / cs.Int32ul,     # flags
            "off" / cs.Int32ul,     # sect:off of the separated code
            "offParent" / cs.Int32ul, # sectParent:offParent of the enclosing scope
            "sect" / cs.Int16ul, # (proc, block, or sepcode)
            "sectParent" / cs.Int16ul,
            cs.Padding(lambda ctx: ctx.pEnd - ctx._params.entry_offest -\
                ctx._io.tell() + cs.Int32ul.sizeof())     
        ),
    )
)
```

到目前为止，我们设法在 PE 部分获得了补偿，根据您的目标，这可能就足够了。然而，如果一个人需要知道函数的边界，它可能是一些调试/二进制分析工作的一部分，在这种情况下，被调试者的地址空间中的地址有更大的用途。为此，我在[pefile](https://github.com/erocarrera/pefile)库的帮助下检索 dll 的首选基地址，并使用节的虚拟地址来计算其在被调试进程的地址空间中的地址。

**注意：** 当然，DLL 可能被加载到与其首选基地址不同的地址；为了解决这种情况，可以轻松地修改脚本，使其接受一个附加参数。

把它们放在一起，我们得到：

list_code_blocks() 实现

```python
def list_code_blocks(pdb, base, fname):
    fncs = list(filter(lambda s: s.leaf_type == S_PROCREF and s.name == fname,\
        pdb.STREAM_GSYM.globals))
    if len(fncs) == 0:
        print("There is no S_PROCREF-type reference to",\
            fname, "in the global symbols stream.")
        return
                
    data = pdb.streams[pdb.STREAM_DBI.DBIExHeaders[\
        fncs[0].iMod - 1].stream].data 
    fn = GlobalProc.parse(data[fncs[0].offset:], entry_offest = fncs[0].offset)
    segaddr = pdb.STREAM_SECT_HDR.sections[fn.PROCSYM32.seg – 1]\
        .VirtualAddress + base
    print("Function start:", hex(segaddr + fn.PROCSYM32.offset))
    print("Function end:", hex(segaddr + fn.PROCSYM32.offset +\
        fn.PROCSYM32.len), "( length = ", fn.PROCSYM32.len, ")")
    
    print("Separated blocks of code:")
    for s in fn.sepcodesyms:
        sectaddr = pdb.STREAM_SECT_HDR.sections[s.sect – 1]\
            .VirtualAddress + base
        print("\t", "Block start:", hex(sectaddr + s.off))
        print("\t", "Block end:", hex(sectaddr + s.off + s.length),\
            "( length = ",  s.length, ")")
        print()
```

为了结束我们关于分离代码主题的小讨论，让我们可以说，演示该方法的实际应用。

list_code_blocks() 演示

```shell
$ python3 pdb_list_code_blocks.py -p basesrv.pdb -m basesrv.dll -n ServerDllInitialization
Function start: 0x180001680
Function end: 0x1800023f2 ( length =  3442 )
Separated blocks of code:
	 Block start: 0x180004d06
	 Block end: 0x180004f8a ( length =  644 )
```

为了您的方便，所有相关的 Python 代码片段都收集在一个[脚本](https://gist.github.com/Auscitte/e2f7d69f4a1023ba64d8189995073399)中。享受！

——瑞·奥西特

## 后记

***更新。*** 最近我看到[一篇](https://codemachine.com/articles/x64_deep_dive.html)关于这个主题的文章。其中，音素据称是应用***基本块工具（BBT）*** 的结果，这是一种***“基于轮廓的优化”*** 。它旨在通过将模块中最常执行的分支组合在一起***来增加“代码的空间局部性” ，以便它们在可能的情况下适合单个页面，从而减少进程的工作集。*** 据说代码块的执行频率是在分析器的帮助下获得的。

尽管我自己没有研究（深入）这个主题，但这个策略对我来说听起来非常合理。

## 参考：

1. Sven B. Schreiber，2001，未记录的 Windows 2000 秘密：程序员的食谱，Addison-Wesley Longman Publishing Co., Inc.，美国。
2. Ry Auscitte，[关于使用 pdbparse 从 PDB 文件中检索类型信息](https://auscitte.github.io/systems%20blog/Func-Prototypes-With-Pdbparse)
3. [来自 Microsoft 的有关 pdb 格式的信息](https://github.com/Microsoft/microsoft-pdb)
4. CodeMachine Inc.，[X64 深潜](https://codemachine.com/articles/x64_deep_dive.html)

