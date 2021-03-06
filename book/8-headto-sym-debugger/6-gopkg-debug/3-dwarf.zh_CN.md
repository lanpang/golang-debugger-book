## pkg debug/dwarf 应用

### 数据类型及关系

标准库提供了package `debug/dwarf` 来读取go编译工具链生成的DWARF数据，比如.debug_info、.debug_line。

go生成DWARF调试信息时，会对DWARF信息进行压缩再存储到不同的section中，比如描述types、variables、function定义的数据，开启压缩前是存储在.debug_info中，开启压缩后则被存储到.zdebug_info中，描述其他调试信息的section也会做类似处理。

从elf文件中读取调试信息时，需要检查信息是存储在.debug前缀的section中，还是存储在.zdebug前缀的section中，如果是后者需要执行zlib解压操作。`debug/dwarf`提供了部分DWARF信息的读取，内部处理了这部分检测读取哪个section、解压缩逻辑。但是标准库未提供调用栈信息的读取，这部分需要自行实现，因此了解这些信息是必要的。

package debug/dwarf中的相关重要数据结构，如下图所示：

![image-20201206022523363](assets/image-20201206022523363.png)

当我们打开了一个elf.File之后，便可以读取DWARF数据，当我们调用elf.File.Data()时便可以返回读取、解析后的DWARF数据（即类图中Data），接下来便是在此基础上进一步读取DWARF中的各类信息，以及与对源码的理解结合起来。

通过Data可以获取一个读取解析.debug_info（或.zdebug_info）section的Reader，通过它可以遍历DIE（即类图中Entry），每个DIE都由一个Tag和一系列Attr构成。

当我们读取到一个Tag类型为DW_TAG_compile_unit的DIE时，表明当前是一个编译单元，每个编译单元都有一个自己的行号表，通过Data即该DIE，可以得到一个读取.debug_line（或.zdebug_line）的LineReader，通过它可以读取行号表中的记录（即类图中LineEntry），它记录了虚拟内存地址、源文件名、行号、列号等的一些对应关系。

### 常用操作及示例

前面大致介绍了标准库提供的支持、局限性，以及标准库的大致使用方式，接下来我们提供几个示例来演示如何读取并解析DWARF调试信息，如何从中提取我们关心的内容。

#### 读取DWARF数据

首先要打开elf文件，然后再读取DWARF相关的多个section数据并解析，go标准库已经帮我们实现了DWARF数据是否压缩、是否需要解压缩的问题。

下面的程序打开一个elf文件并返回解析后的DWARF数据：

```go
import (
    "debug/elf"
     "fmt"
)
func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: go run main.go <prog>")
		os.Exit(1)
	}
	prog := os.Args[1]

	// open elf
	file, err := elf.Open(prog)
	if err != nil {
		panic(err)
	}
    
    // dwarf调试信息遍历
	dw, err := file.DWARF()
	if err != nil {
		panic(err)
	}
    fmt.Println("read dwarf ok")
}
```

运行测试`go run main.go ../testdata/loop2`，程序只是简单地打印一行读取成功的信息，在此基础上我们将实现DWARF数据中各类信息的读取。

#### 读取编译单元信息

当从elf文件拿到DWARF数据dwarf.Data之后，就可以从dwarf.Data中读取感兴趣的数据。在读取之前要理解DWARF数据的组织方式，以及go标准库实现的一点内容。

工程中可能包含了多个源文件，每个源文件是一个编译单元，一个目标文件中可能包含了多个编译单元。生成调试信息时每一个目标文件对应一个tag类型为DW_TAG_compile_unit的DIE。该DIE的childrens又包含了其他丰富的信息，如函数、函数中的变量等，这些信息都是通过DWARF DIE来表述的。

> go编译单元是如何产生的，go tool compile *.go，依赖的源文件也会被一同编译成同一个目标文件，而不是每个源文件一个目标文件。因为c、c++允许通过extern来声明外部定义的变量，然后在当前文件中使用，每个文件是可以独立编译成一个目标文件的，这点上go有些差异。

`rd := dwarf.Data.Reader()`会返回一个reader对象，通过`rd.Next()`将能够遍历编译单元中及其包含的所有类型、变量、函数等对应的DIE列表。

下面我们先尝试遍历所有的编译单元信息。

```go
package main

import (
	"debug/dwarf"
	"debug/elf"
	"fmt"
	"os"
	"text/tabwriter"
)

func main() {
    ...
    dw, err := file.DWARF()
    ...
    
    err = iterateComplilationUnit(dw)
	if err != nil {
		fmt.Println(err)
	}
}

func iterateComplilationUnit(dw *dwarf.Data) error {
	rd := dw.Reader()

	tw := tabwriter.NewWriter(os.Stdout, 0, 4, 3, ' ', 0)
	fmt.Fprintf(tw, "No.\tTag\tName\tLanguage\tStmtList\tLowPC\tRanges\tOthers\n")
	defer tw.Flush()

	for idx := 0; ; idx++ {
		entry, err := rd.Next()
		if err != nil {
			return fmt.Errorf("iterate entry error: %v", err)
		}
		if entry == nil {
			fmt.Println("iterate entry finished")
			return nil
		}
		if entry.Tag != dwarf.TagCompileUnit {
			continue
		}
		fmt.Fprintf(tw, "%d\t%s\t%v\t%v\t%v\t%v\t%v\n",
			idx,
			entry.Tag.String(), entry.Field[0].Val,
			entry.Field[1].Val, entry.Field[2].Val,
			entry.Field[3].Val, entry.Field[4].Val, )
	}
}
```

执行测试`go run main.go ../testdata/loop2`，程序输出了如下信息：

```bash
$ go run main.go ../testdata/loop2

Tag           Name                      Language   StmtList   LowPC     Ranges   Others
CompileUnit   sync                      22         0          4724928   0
CompileUnit   internal/cpu              22         3626       4198400   32
CompileUnit   internal/cpu              22         4715       4201888   80
CompileUnit   runtime/internal/sys      22         4846       4202336   112
CompileUnit   fmt                       22         5513       4906048   144
CompileUnit   runtime/internal/atomic   22         14330      4202560   176
CompileUnit   strconv                   22         160219     4653184   944
...........   .......                   ..         ......     .......   ...
CompileUnit   syscall                   22         167358     4883104   992
CompileUnit   internal/oserror          22         170142     4882624   1040
CompileUnit   io                        22         170356     4881888   1072
CompileUnit   internal/fmtsort          22         170746     4873280   1104
CompileUnit   sort                      22         171968     4870400   1136
CompileUnit   unicode/utf8              22         172957     4676128   1168
CompileUnit   reflect                   22         174048     4767616   1200
CompileUnit   sync/atomic               22         194816     4658240   1248
CompileUnit   sync/atomic               22         195127     4658976   1280
CompileUnit   unicode                   22         195267     4742624   1312
CompileUnit   runtime                   22         195635     4631616   1344
CompileUnit   reflect                   22         195725     4855840   1376
```

这里显示了每个编译单元的信息，如名称、编程语言、语句列表数量、地址范围。

#### 读取函数定义

DIE描述代码，前面提到了编译单元是tag为DW_TAG_compile_unit的DIE来描述的，读取完该DIE之后，可继续读取编译单元中的函数定义，即tag为DW_TAG_subprogram的一系列DIE。读取了每个函数的同时，函数内部又包含一些局部变量定义等，即tag为DW_TAG_variable的一系列DIE。

它们之间的关系，大致如下所示：

```bash
  DW_TAG_compile_unit
    ...
    DW_TAG_subprogram
      ...
      DW_TAG_variable
        DW_AT_name: "a"
        DW_AT_type: (signature) 0xd681845c 21a14576
        DW_AT_location: ...
    ...
```

这里我们以读取main.main为例，演示下如何读取编译单元中的函数、变量信息。

**main.go**

```go
package main

import (
	"debug/dwarf"
	"debug/elf"
	"fmt"
	"os"
)

func main() {
    ...
	dw, err := file.DWARF()
	...
    
	err = parseDebugInfo(dw)
	if err != nil {
		fmt.Println(err)
	}
}

// Variable 函数局部变量信息
type Variable struct {
	Name string
}

// Function 函数信息，包括函数名、定义的源文件、包含的变量
type Function struct {
	Name      string
	DeclFile  string
	Variables []*Variable
}

// CompileUnit 编译单元，包括一系列源文件、函数定义
type CompileUnit struct {
	Source []string
	Funcs  []*Function
}

var compileUnits = []*CompileUnit{}

func parseDwarf(dw *dwarf.Data) error {
	rd := dw.Reader()

	var curCompileUnit *CompileUnit
	var curFunction *Function

	for idx := 0; ; idx++ {
		entry, err := rd.Next()
		if err != nil {
			return fmt.Errorf("iterate entry error: %v", err)
		}
		if entry == nil {
			return nil
		}

		if entry.Tag == dwarf.TagCompileUnit {
			lrd, err := dw.LineReader(entry)
			if err != nil {
				return err
			}

			cu := &CompileUnit{}
			curCompileUnit = cu

			for _, v := range lrd.Files() {
				if v == nil {
					continue
				}
				cu.Source = append(cu.Source, v.Name)
			}
			compileUnits = append(compileUnits, cu)
		}

		if entry.Tag == dwarf.TagSubprogram {
			fn := &Function{
				Name:     entry.Val(dwarf.AttrName).(string),
				DeclFile: curCompileUnit.Source[entry.Val(dwarf.AttrDeclFile).(int64)-1],
			}
			curFunction = fn
			curCompileUnit.Funcs = append(curCompileUnit.Funcs, fn)

            // 如果是main.main函数，打印一下entry，方便我们印证
			if fn.Name == "main.main" {
				printEntry(entry)
				fmt.Printf("main.main is defined in %s\n", fn.DeclFile)
			}
		}

		if entry.Tag == dwarf.TagVariable {
			variable := &Variable{
				Name: entry.Val(dwarf.AttrName).(string),
			}
			curFunction.Variables = append(curFunction.Variables, variable)
            // 如果当前变量定义在main.main中，打印一下entry，方便我们印证
			if curFunction.Name == "main.main" {
				printEntry(entry)
			}
		}
	}
	return nil
}

// 打印每个DIE的详细信息，调试使用，方便我们根据具体结构编写代码
func printEntry(entry *dwarf.Entry) {
	fmt.Println("children:", entry.Children)
	fmt.Println("offset:", entry.Offset)
	fmt.Println("tag:", entry.Tag.String())
	for _, f := range entry.Field {
		fmt.Println("attr:", f.Attr, f.Val, f.Class)
	}
}
```

在执行测试之前，我们也说一下用来测试的源程序，注意我们在main.main中定义了一个变量pid。

**testdata/loop2.go**

```go
     1  package main
     2  
     3  import "fmt"
     4  import "os"
     5  import "time"
     6  
     7  func init() {
		....
    14  }
    15  func main() {
    16      pid := os.Getpid()
    17      for {
    18          fmt.Println("main.main pid:", pid)
    19          time.Sleep(time.Second * 3)
    20      }
    21  }

```

执行测试`go run main.go ../testdata/loop2`，程序输出如下信息：

```bash
$ go run main.go ../testdata/loop2 
children: true
offset: 324423
tag: Subprogram
attr: Name main.main ClassString
attr: Lowpc 4949376 ClassAddress
attr: Highpc 4949656 ClassAddress
attr: FrameBase [156] ClassExprLoc
attr: DeclFile 2 ClassConstant
attr: External true ClassFlag

main.main is defined in /root/debugger101/testdata/loop2.go

children: false
offset: 324457
tag: Variable
attr: Name pid ClassString
attr: DeclLine 16 ClassConstant
attr: Type 221723 ClassReference
attr: Location [145 160 127] ClassExprLoc
```

上面程序中打印了main.main对应的subprogram的详细信息，并展示了main.main是定义在testdata/loop2.go这个源文件中（行信息依赖行表，稍后介绍），还展示了main.main中定义的局部变量pid。

遍历编译单元CompileUnit，并从中编译单元中依次读取各个函数Subprogram，以及函数中的一系列变量Variable的过程，大致可以由上述示例所覆盖。当然我们还要提取更多信息，比如函数定义在源文件中的行号信息、变量在源文件中的行号、列号信息等等。

#### 读取行号表信息

每个编译单元CompileUnit都有自己的行号表信息，当我们从DWARF数据中读取出一个tag类型为DW_TAG_compile_unit的DIE时，就可以尝试去行表.debug_line/.zdebug_line中读取行号表信息了。这里debug/dwarf也提供了对应的实现，dwarf.LineReader每次从指定编译单元中读取一行行表信息dwarf.LineEntry。

后续基于行表数据可以轻松实现源文件位置和虚拟地址之间的转换。

我们先实现行号表的读取，只需在此前代码基础上做少许变更即可：

```go
func main() {
    ...
	err = parseDwarf(dw)
	...
	pc, err := find("/root/debugger101/testdata/loop2.go", 16)
	if err != nil {
		panic(err)
	}

	fmt.Printf("found pc: %#x\n", pc)
}

type CompileUnit struct {
	Source []string
	Funcs  []*Function
	Lines  []*dwarf.LineEntry
}

func parseDwarf(dw *dwarf.Data) error {}
	...
	for idx := 0; ; idx++ {
        ...
        
		if entry.Tag == dwarf.TagCompileUnit {
			lrd, err := dw.LineReader(entry)
			...

			for {
				var e dwarf.LineEntry
				err := lrd.Next(&e)
				if err == io.EOF {
					break
				}
				if err != nil {
					return err
				}
				curCompileUnit.Lines = append(curCompileUnit.Lines, &e)
			}
		}
        ...
    }
}
```

我们查找下源文件位置`testdata/loop2.go:16`对应的虚拟地址（当前我们是硬编码的此位置），执行测试`go run main.go ../testdata/loop2`：

```bash
$ go run main.go ../testdata/loop2

found pc: 0x4b85af
```

程序正确找到了上述源文件位置对应的虚拟内存地址。

读者朋友可能想问，为什么示例程序中不显示出源文件位置对应的函数定义呢？这里涉及到对.debug_frame/.zdebug_frame调用栈信息表的读取、解析。很遗憾go标准库没有对这些sections提供良好的支持，我们要靠自己了。没关系，it's easy!

有了调用栈信息表之后，我们就可以根据每个栈帧覆盖的虚拟内存地址范围，找到pc对应的栈帧，进而确定出对应的是哪个函数定义。

#### 读取调用栈信息

elf文件中，调用栈信息表存储在.debug_frame或.zdebug_frame中，go标准库`debug/dwarf`不支持这部分信息的解析，为了避免大家对标准库实现产生理解上的偏差，我们将在《挺近DWARF小节》中详细描述如何读取、解析、应用调用栈信息。



### 参考内容

1. How to Fool Analysis Tools, https://tuanlinh.gitbook.io/ctf/golang-function-name-obfuscation-how-to-fool-analysis-tools

2. Go 1.2 Runtime Symbol Information, Russ Cox, https://docs.google.com/document/d/1lyPIbmsYbXnpNj57a261hgOYVpNRcgydurVQIyZOz_o/pub

3. Some notes on the structure of Go Binaries, https://utcc.utoronto.ca/~cks/space/blog/programming/GoBinaryStructureNotes

4. Buiding a better Go Linker, Austin Clements, https://docs.google.com/document/d/1D13QhciikbdLtaI67U6Ble5d_1nsI4befEd6_k1z91U/view


5.  Time for Some Function Recovery, https://www.mdeditor.tw/pl/2DRS/zh-hk