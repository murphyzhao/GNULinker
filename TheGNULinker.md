
# The GNU Linker

> 本文档由 [MurphyZhao](https://github.com/murphyzhao/) 与 2019 年 8 月 23 日翻译。  
> 文档来源于 **gcc-arm-none-eabi-4_8-2014q3-20140805-linux.tar.bz2** 中的 `share/doc/gcc-arm-none-eabi/html/ld.html`。

```
            ld
(GNU Binutils)
Version 2.23.2
```

此文件基于 GNU 链接器 ld（GNU Binutils）版本 2.23.2。

本文档根据 GNU 自由文档许可证 1.3 版的条款分发。许可证的副本包含在标题为 “GNU 自由文档许可证” 的部分中。

# 1 概览

ld 软件组合一系列的 object 和 archive（目标/归档） 文件，重定位他们的数据并绑定符号引用（symbol references）。通常在程序编译的最后一步执行 ld 软件进行程序链接。

ld 软件接受以 AT＆T 的 **链接编辑器命令语言** 语法的超集编写的链接器命令语言文件，以提供对链接过程的显式和完全控制。

这个版本，LD 使用通用 BFD 库来操作目标文件。这允许 LD 以多种不同的格式读取、组合和写入目标文件 -  例如，COFF 或 a.out。可以将不同的格式链接在一起以产生任何可用类型的目标文件（object file）。有关更多信息，请参阅 BFD。

除了灵活性之外，gnu 链接器在提供诊断信息方面比其他链接器更有帮助。许多链接器在遇到错误时立即放弃执行; 只要有可能，LD  继续执行，允许您识别其他错误（或者，在某些情况下，尽管出现错误，仍然可以获取输出文件）。

# 2 调用 / 祈祷（Invocation）

GNU 链接器 ld 旨在涵盖广泛的情况，并尽可能与其他链接器兼容。因此，您有很多选择来控制其行为。

- Optios：命令行选项
- Environment：环境变量

## 2.1 命令行选项

链接器支持大量的命令行选项，但在实际操作中，只有他们中很少的一部分被用于特定的上下文。 例如，LD 经常被用于在标准的、受支持的 Unix 系统上链接标准的 Unix 目标文件。在这样的系统上，去链接文件 hello.o 如下所示：

```
    ld -o output /lib/crt0.o hello.o -lc
```

上述代码表示 LD 使用 `/lib/crt0.o` 和 `hello.o` 以及 `libc.a` 生成一个名为 *output* 的输出文件，这些输入文件来自标准搜索目录。（参考下面关于 `-l` 选项的讨论）



# 3 链接脚本

每一次链接（link）都是由链接*脚本脚本*。此脚本使用链接器命令语言编写。

链接脚本的主要目的是描述输入文件中的各部分（sections）应如何映射到输出文件，以及控制输出文件的内存布局。大多数链接器脚本仅会执行这些操作。但是，必要时，链接脚本还可以使用下面描述的命令指示链接器执行更多其他操作。

链接器始终使用链接脚本。如果您自己不提供，则链接器将使用编译到链接器可执行文件中的默认脚本。你可以使用 `--verbose` 命令行选项显示默认链接脚本。某些命令行选项，例如 `-r` 要么 `-N`，会影响默认的链接描述文件。

您可以使用 `-T` 命令行选项指定自己的链接描述文件。执行此操作时，链接描述文件将替换默认链接描述文件。

您也可以隐式使用链接描述文件，将它们命名为链接器的输入文件，就好像它们是要链接的文件一样。请参阅[隐式链接描述文件]()。

- Basic Script Concepts: 基本的链接脚本概念
- Script Format: 链接脚本格式
- Simple Example: 简单的链接脚本示例
- Simple Commands: 简单的链接脚本命令
- Assignments: 分配，将值分配给符号
- SECTIONS: SECTIONS 命令
- MEMORY: MEMORY 命令
- PHDRS: PHDRS 命令
- VERSION: VERSION 命令
- Expressions: 链接脚本中的表达式
- Implicit Linker Scripts: 隐式链接描述文件

## 3.1 基本的链接脚本概念

我们需要定义一些基本概念和词汇表来描述链接器脚本语言。

链接器将输入文件合并为单个输出文件。输出文件和每个输入文件采用称为 *目标文件格式* 的特殊数据 格式。每个文件都称为 *目标文件*（object file）。输出文件通常称为 *可执行文件*，但出于我们的目的，我们也将其称为目标文件。除了别的以外，每个目标文件都有一个 `*sections*` 列表。我们有时将输入文件中的一个 section 称为 *输入 section*；类似地，输出文件中的 section 是 *输出 section*。

目标文件中的每个 section 都有名称和大小。大多数 section 还有一个相关的数据块，称为 *section 内容*。一个 section 可能被标记为 *可加载*，这意味着在运行输出文件时，应将 section 内容加载到内存中。没有内容的 section 可以是 *可分配的*，这意味着应该在内存中留出一片区域，但是不应该在那里加载任何内容（在某些情况下，该内存必须被清零）。既不可加载也不可分配的 section 通常包含某种调试信息。

每个可加载或可分配的输出 section 都有两个地址。第一个地址是 *VMA* 或虚拟内存地址，这是输出文件运行时的地址。第二个是 LMA 或装载存储器地址（load memory address），这是加载该 section 时的地址。在大多数情况下，两个地址将是相同的。它们可能不同的一个例子是当数据 section 被加载到 ROM 中，然后在程序运行时复制到 RAM 中（这种技术通常用于在基于 ROM 的系统中初始化全局变量）。在这种情况下，ROM 地址将是 LMA，RAM 地址将是 VMA。

您可以使用 `objdump` 程序加 `-h` 选项查看目标文件中的 sections。

每个目标文件还有一个*符号表*（symbol table）。符号可以定义或未定义。每个符号都有一个名称，每个定义的符号都有一个地址，以及其他信息。如果将 C 或 C++ 程序编译为目标文件，则将为每个已定义的函数和全局或静态变量分配 *已定义的符号*。输入文件中引用的每个未定义函数或全局变量都将成为 *未定义的符号*。

您可以使用 nm 程序查看目标文件中的符号（symbols），或使用 objdump 程序加 `-t` 选项。

## 3.2 链接脚本格式

链接脚本是文本文件。

你将使用一系列命令编写链接脚本。每个命令都是关键字，可能命令后跟参数，或者是符号的赋值。您可以使用*分号*分隔命令。空格通常被忽略。

通常可以直接输入文件或格式名称等字符串。如果文件名包含*逗号*等字符，您可以将文件名放在双引号中，否则逗号将用于分隔文件名。无法在文件名中使用双引号字符。

您可以在链接脚本中包含注释，就像在 C 中一样，由 “/*” 和 “*/” 分隔。与在 C 中一样，注释在语法上等同于空格。

## 3.3 简单的链接脚本示例

许多链接脚本都相当简单。

最简单的链接脚本只有一个命令：`SECTIONS`。使用 `SECTIONS` 命令描述输出文件的内存布局。

`SECTIONS` 是一个强大的命令。在这里，我们将描述它的简单用法。假设您的程序仅包含代码，初始化数据和未初始化数据，这三部分将分别存在于 `.text`, `.data` 和 `.bss` 段（section）。我们进一步假设这些是输入文件中出现的唯一部分。

对于这个例子，假设代码加载到 0x10000 地址，并且数据从地址 0x8000000 开始。那么链接脚本如下所示：

```
SECTIONS
{
    . = 0x10000;
    .text : { *(.text) }
    . = 0x8000000;
    .data : { *(.data) }
    .bss : { *(.bss) }
}
```

在链接脚本中将 `SECTIONS` 命令写为关键字 `SECTIONS`，后跟一系列用大括号括起来的符号赋值和输出段（output section）描述。

上面例子，在 `SECTIONS` 命令中的第一行设置特殊符号 `.` 的值，`.` 是位置计数器。如果未以其他方式指定输出段的地址（后面将介绍其他方式），则会根据位置计数器的当前值设置地址。然后，位置计数器增加输出段的大小。在 `SECTIONS` 命令的起始位置处，位置计数器值为 0。

第二行定义了 `.text` 输出段。冒号是必需的语法，现在可以忽略。在输出段名称后的花括号内，列出应放入此输出段的输入段的名称。`*` 是一个匹配任何文件名的通配符。表达方式 `*(.text)` 意味着所有输入文件中的 `.text` 段。

当 `.text` 段定义的时候，位置计数器值为 0x10000，那么链接器将设置 `.text` 段的地址为 0x10000。

其余的行在输出文件中定义了 `.data` 和 `.bss` 段。链接器将 `.data` 输出段放置到 0x8000000 地址。链接器放置 `.data` 段后，位置计数器的值将是 0x8000000 加上 `.data` 段的大小。效果是，在内存中链接器将 `.bss` 段放置到 `.data` 段后面。

链接器将在必要时通过增加位置计数器值来确保每个输出段具有所需的对齐。在此示例中，为 `.text` 和 `.data` 指定的地址可能会满足任何对齐约束，但链接器可能必须在 `.data` 和 `.bss` 段之间创建一个小的间隙，以满足对齐约束。

以上就是全部！这是一个简单且完整的链接脚本。

## 3.4 简单的链接脚本命令

在本节中，我们将介绍简单的链接脚本命令。

- Entry Point：设置入口点
- File Commands：处理文件的命令
- Format Commands：处理目标文件格式的命令
- REGION_ALIAS：将别名分配到内存区域
- Miscellaneous Commands：其它命令

### 3.4.1 设置入口点

在程序中执行的第一条指令称为 *entry point(入口点)*。您可以使用 `ENTRY` 链接脚本命令设置入口点。`ENTRY` 命令的参数是符号名称：

```
ENTRY(symbol)
```

有几种方法可以设置入口点。链接器将按顺序尝试以下每个方法来设置入口点，当其中一个方法成功时停止设置入口点：

- `-e` 命令行选项;
- 在链接脚本中使用 `ENTRY(symbol)` 命令;
- 目标特定符号的值，如果已定义; 对于许多目标而言，这是开始，但是基于 PE 和 BeOS 的系统会检查一系列可能的入口符号，匹配找到的第一个符号。
- `.text` 段的第一个字节的地址（如果 `.text` 段存在）;
- 地址 0。

### 3.4.2 处理文件的命令

处理文件的几个链接脚本命令。

#### INCLUDE

```
INCLUDE filename
```

在当前点引入链接脚本文件 *filename*。将在当前目录以及使用 `-L` 选项指定的任何目录中搜索该文件。你可以嵌套调用 `INCLUDE` 最多 10 级。

你可以将 `INCLUDE` 放到 `MEMORY` 命令、`SECTIONS` 命令或输出段描述的顶层。（*此处翻译有待商榷*）

#### INPUT

```
INPUT(file, file, ...)
INPUT(file file ...)
```

`INPUT` 命令指示链接器在链接时包含命令列出的文件，跟在命令行上使用的一样。

例如，如果你总是想在链接的时候包含 `subr.o`，但你不想把它放到每个链接命令上，那么你可以在链接器脚本中加入 `INPUT(subr.o)`。

实际上，如果您愿意，可以在链接脚本中列出所有输入文件，然后使用 `-T` 选项调用链接器。

如果配置了 `sysroot 前缀`，并且 filename 以 `/` 字符开头，并且正在处理的脚本位于 *sysroot 前缀* 内，则将在 *sysroot 前缀* 中查找 filename。 否则，链接器将尝试在当前目录中打开该文件。如果未找到，链接器将搜索库搜索路径。请参阅[命令行选项]()中的 `-L` 说明。

如果使用 `INPUT(-lfile)`，链接器 ld 会将名字转换为 libfile.a，与命令行参数 `-l` 一样。

在隐式链接脚本中使用 `INPUT` 命令时，文件在链接脚本被包含时才会被加入。这可能会影响库搜索。

#### GROUP

```
GROUP(file, file, ...)
GROUP(file file ...)
```

`GROUP` 命令类似于 `INPUT`，除了 GROUP 命令列出的文件都应该是库文件，并且重复搜索它们，直到没有创建新的未定义引用。请参阅[命令行选项]()中的 `-(` 的说明。

#### AS_NEEDED

```
AS_NEEDED(file, file, ...)
AS_NEEDED(file file ...)
```

此构造只能出现在 `INPUT` 或 `GROUP` 命令中，位于其他命令中间。此命令中列出的文件将被视为直接出现在 `INPUT` 或 `GROUP` 命令中，但 `ELF` 共享库除外，`ELF` 共享库只有在实际需要时才会被添加。这个构造实质上为此命令中列出的所有文件启用了 `--as-needed` 选项，并且恢复此前的 `--as-needed` 设置，此后的 `--no-as-needed`。

#### OUTPUT

```
OUTPUT(filename)
```

该 OUTPUT 命令命名输出文件。在链接脚本中使用 `OUTPUT(filename)` 跟在命令行中使用 `-o filename` 一样（参阅[命令行选项]()）。如果两者都使用，则命令行选项优先。

您可以使用该 `OUTPUT` 命令为输出文件定义默认名称，而不是通常的默认名称的 `a.out`。 

#### SEARCH_DIR

```
SEARCH_DIR(path)
```

`SEARCH_DIR` 命令将 *path* 添加到 ld 搜索库路径列表中。使用 `SEARCH_DIR(path)` 就像使用在命令行上使用 `-L` 选项（请参阅[命令行选项]()）。如果两者都使用，则链接器将搜索两个路径。首先搜索使用命令行选项指定的路径。 

#### STARTUP

```
STARTUP(filename)
```

`STARTUP` 命令就像 `INPUT` 命令一样，除了 *filename* 将成为要链接的第一个输入文件，就好像它是在命令行中首先指定的那样。在一些把第一个文件当做入口点的系统上，该命令会很有用。

### 3.4.3 处理目标文件格式的命令

一些处理目标文件格式的链接脚本命令。

#### OUTPUT_FORMAT

```
OUTPUT_FORMAT(bfdname)
OUTPUT_FORMAT(default, big, little)
```

`OUTPUT_FORMAT` 命令使用 `BFD` 格式定义输出文件（参见 [BFD]()）。使用 `OUTPUT_FORMAT(bfdname)` 跟在命令行上使用 `--oformat bfdname` 一样（请参阅[命令行选项]()）。如果两者都使用，则命令行选项优先。

可以使用三个参数（*default, big, little*）的 `OUTPUT_FORMAT` 来基于 `-EB` 和 `-EL`命令行选项使用不同的格式。这允许链接脚本根据所需的字节序设置输出文件格式。

如果没有 `-EB` 或 `-EL` 使用，那么输出文件格式将默认使用第一个参数（*default*）。如果使用了 `-EB`，输出格式将是第二个参数（*big*：大端）；如果 `-EL` 使用，输出格式将是第三个参数（*little*：小端）。

例如，`MIPS ELF` 目标的默认链接描述文件使用以下命令：

```
OUTPUT_FORMAT(elf32-bigmips, elf32-bigmips, elf32-littlemips)
```

这表示输出文件的默认格式是 `ELF32-bigmips`，但如果用户使用 `-EL` 命令行选项，输出文件将在以 `ELF32-littlemips` 格式创建。 

#### TARGET

```
TARGET(bfdname)
```

`TARGET` 命令配置在读取输入文件时使用的 `BFD` 格式。它影响后续 `INPUT` 和 `GROUP` 命令。跟在命令行上使用 `-b bfdname` 选项一样（请参阅[命令行选项]()）。如果 `TARGET` 使用但未使用 `OUTPUT_FORMAT`，则最后一个 `TARGET` 命令也用于设置输出文件的格式。见 [BFD]()。

### 3.4.4 Assign alias names to memory regions（为内存区域指定别名）

可以为 MEMORY 命令创建的内存区域指定别名。每个名字最多对应一个区域。

```
REGION_ALIAS(alias, region)
```

`REGION_ALIAS` 函数为内存区域 *region* 指定 *alias* 别名。这允许灵活地将输出部分映射到存储器区域。一个例子如下:

假设我们有一个嵌入式系统的应用程序，它带有各种内存存储设备。它们都具有通用的易失性存储器 RAM，允许代码执行或数据存储。一些可能具有只读，非易失性存储器 ROM，其允许代码执行和只读数据访问。最后一个变体是只读，非易失性存储器 ROM2，具有只读数据访问和无代码执行能力。我们有四个输出部分：`.text` 程序代码; `.rodata` 只读数据; `.data` 可读写的已初始化数据; `.bss` 可读写的未初始化数据。

- .text 程序代码;
- .rodata 只读数据;
- .data 已初始化的可读可写数据;
- .bss 未初始化的可读可写数据（默认初始化为 0）。

目标是提供一个链接脚本，其中包含系统无关的输出段和系统相关的输出段，将输出段映射到系统上可用的内存区。我们的嵌入式系统具有三种不同的内存设置 A、B、C：

```
Section  Variant A	Variant B	Variant C 
.text	 RAM	    ROM     	ROM 
.rodata	 RAM	    ROM	        ROM2 
.data	 RAM	    RAM/ROM	    RAM/ROM2 
.bss 	 RAM	    RAM	        RAM 
```

符号 `RAM/ROM` 或 `RAM/ROM2` 表示将此部分分别加载到区域 ROM 或 ROM2。请注意，本例中的三个 `.data` 部分的加载地址（起始地址）位于 `.rodata` 段的末尾。

下面是处理输出段的基本链接脚本。它包括描述内存布局的 `linkcmds.memory` 系统相关文件：

链接脚本：

```
INCLUDE linkcmds.memory

SECTIONS
{
    .text :
    {
        *(.text)
    } > REGION_TEXT
    .rodata :
    {
        *(.rodata)
        rodata_end = .;
    } > REGION_RODATA
    .data : AT (rodata_end)
    {
        data_start = .;
        *(.data)
    } > REGION_DATA
    data_size = SIZEOF(.data);
    data_load_start = LOADADDR(.data);
    .bss :
    {
        *(.bss)
    } > REGION_BSS
}
```

现在我们需要三个不同的 linkcmds.memory 文件来定义内存区域和别名。下面分别是 A、B、C 对应的 linkcmds.memory 内容：

**A:**

> 所有的内容都存入 RAM 空间。

```
MEMORY
    {
    RAM : ORIGIN = 0, LENGTH = 4M
    }

REGION_ALIAS("REGION_TEXT", RAM);
REGION_ALIAS("REGION_RODATA", RAM);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

**B:**

> 程序代码和只读数据存放在 ROM 空间。可读写数据存入 RAM 空间。已初始化的数据被加载到 ROM，并在系统启动期间复制到 RAM。

```
MEMORY
{
    ROM : ORIGIN = 0, LENGTH = 3M
    RAM : ORIGIN = 0x10000000, LENGTH = 1M
}

REGION_ALIAS("REGION_TEXT", ROM);
REGION_ALIAS("REGION_RODATA", ROM);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

**C:**

> 程序代码存在 ROM 空间。只读数据存入 ROM2 空间。可读写数据进入 RAM 空间。已初始化的数据被加载到 ROM2，并在系统启动期间复制到 RAM。

```
MEMORY
{
    ROM : ORIGIN = 0, LENGTH = 2M
    ROM2 : ORIGIN = 0x10000000, LENGTH = 1M
    RAM : ORIGIN = 0x20000000, LENGTH = 1M
}

REGION_ALIAS("REGION_TEXT", ROM);
REGION_ALIAS("REGION_RODATA", ROM2);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

如果需要，可以编写一个通用的系统初始化程序将 `.data` 段从 ROM 或 ROM2 复制到 RAM：

```
#include <string.h>

extern char data_start[];
extern char data_size[];
extern char data_load_start[];

void copy_data(void)
{
    if (data_start != data_load_start)
    {
        memcpy(data_start, data_load_start, (size_t) data_size);
    }
}
```

### 3.4.5 Other Linker Script Commands（其它的链接脚本命令）

还有一些其他链接脚本命令。

#### ASSERT

```
ASSERT(exp, message)
```

确保 exp 不为零。如果为零，则并打印 *message* 内容，然后退出链接器。

#### EXTERN

```
EXTERN(symbol symbol ...)
```

强制 *symbol* 在输出文件中作为未定义的符号。例如，这样做可以触发标准库中其他模块的链接。您可以为每个 `EXTERN` 罗列出多个符号，并且多次使用 `EXTERN`。此命令与 `-u` 命令行选项具有相同的效果。 

#### FORCE_COMMON_ALLOCATION

```
FORCE_COMMON_ALLOCATION
```

此命令与 `-d` 命令行选项具有相同的效果：目的是让 ld 为普通符号分配空间，即便是使用了 `-r` 的重定位输出文件。

#### INHIBIT_COMMON_ALLOCATION

```
INHIBIT_COMMON_ALLOCATION
```

此命令与 `--no-define-common` 命令行选项具有相同的效果：让 ld 不为普通符号分配空间，即便是一个非可重定位输出文件。

#### INSERT

```
INSERT [ AFTER | BEFORE ] output_section
```

此命令通常用于由 `-T` 指定的脚本中，以使用例如 *overlays* 来扩充默认的 `SECTIONS`。它在 *output_section* 之后（或之前）插入所有先前的链接脚本，并还使 `-T` 不覆盖默认的链接脚本。确切的插入点与 `orphan` 段相同。参考[位置计数器]()。插入发生在链接器将输入节映射到输出节之后。在插入之前，由于 `-T` 脚本在默认链接脚本之前被解析，因此 `-T` 脚本中的语句先于位于脚本的内部的默认链接脚本执行。特别是，输入段分配将在默认脚本中的输出段之前输出到 `-T` 指定的输出段。以下是使用 `INSERT` 的 `-T` 脚本示例：

```
SECTIONS
{
    OVERLAY :
    {
        .ov1 { ov1*(.text) }
        .ov2 { ov2*(.text) }
    }
}
INSERT AFTER .text;
```

#### NOCROSSREFS

```
NOCROSSREFS(section section ...)
```

此命令可用于告诉 ld 报告 `NOCROSSREFS` 命令中的输出段之间的任何引用的错误。

在某些类型的程序中，特别是在使用 *overlays* 时的嵌入式系统上，当一个段被加载到内存中时，另一个段不加载到内存。这两个部分之间的任何直接引用都是错误。例如，如果一个段中的代码调用另一端中的函数，则会出错。

`NOCROSSREFS` 命令采用输出段名称列表。如果 ld 检测到 *section* 之间的任何交叉引用，它将报告错误并返回非零退出状态。请注意，`NOCROSSREFS` 命令使用输出节名称，而不是输入节名称。

#### OUTPUT_ARCH

```
OUTPUT_ARCH(bfdarch)
```

指定特定的机器体系结构。参数是 BFD 库使用的名称之一（参见 [BFD]()）。您可以使用带有 `-f` 选项的 objdump 程序来查看目标文件的体系结构。

#### LD_FEATURE

```
LD_FEATURE(string)
```

此命令可用于修改 ld 行为。如果 *string* 是 “SANE_EXPR”，则脚本中的绝对符号和数字在任何地方都被视为数字。参考 [Expression Section]()。

## 3.5 Assigning Values to Symbols 为符号指定值

可以在链接脚本中为符号（symbols）指定值。这将定义符号并将其放入全局符号表中。

- Simple Assignments: 简单赋值
- HIDDEN: HIDDEN
- PROVIDE: PROVIDE
- PROVIDE_HIDDEN: PROVIDE_HIDDEN
- Source Code Reference: 如何在源代码中使用链接脚本中定义的符号

### 3.5.1 Simple Assignments

可以使用任何 C 赋值运算符给符号赋值：

```
symbol = expression ;
symbol += expression ;
symbol -= expression ;
symbol *= expression ;
symbol /= expression ;
symbol <<= expression ;
symbol >>= expression ;
symbol &= expression ;
symbol |= expression ;
```

第一种情况定义了一个值为 *expression* 的符号。在其他情况下，必须是已定义的符号（symbol），并相应地调整值。

特殊符号名称 `.` 表示位置计数器。您只能在 `SECTIONS` 命令中使用它。[见位置计数器]()。

`expression` 后的 **分号** 是必需的。

表达式定义如下; 见表达式。

您可以将符号赋值作为命令单独编写，也可以作为命令中的语句编写，或者作为SECTIONS命令中输出节描述的一部分编写SECTIONS。

符号的部分将从表达式的部分设置; 有关更多信息，请参阅表达式部分。

以下示例显示了可以使用符号分配的三个不同位置：

```
floating_point = 0;
SECTIONS
{
    .text :
    {
        *(.text)
        _etext = .;
    }
    _bdata = (. + 3) & ~ 3;
    .data : { *(.data) }
}
```

在这个例子中，符号'浮点'将被定义为零。符号'_etext'将被定义为最后一个之后的地址'。文本'输入部分。符号'_bdata'将被定义为'后面的地址'。文本'输出部分向上对齐到4字节边界。

## 3.6 SECTIONS 命令
## 3.7 MEMORY 命令
## 3.8 PHDRS 命令
## 3.9 VERSION 命令
## 3.10 链接脚本中的表达式
## 3.11 隐式链接描述文件

