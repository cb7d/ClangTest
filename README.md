# Clang

作为一个iOS工程师，每次看到Xcode在进行漫长的编译的时候总是忍不住想深究一下自己手写的BUG是如何被生成的，所以下定决定研究一下我们的编译器

要探究首先要知道我们使用的是LLVM编译器

## LLVM（Low Level Virtual Machine）

> LLVM是一个自由软件项目，它是一种编译器基础设施，以C++写成，包含一系列模块化的编译器组件和工具链，用来开发编译器前端和后端。它是为了任意一种编程语言而写成的程序，利用虚拟技术创造出编译时期、链接时期、运行时期以及“闲置时期”的最优化。它最早以C/C++为实现对象，而当前它已支持包括ActionScript、Ada、D语言、Fortran、GLSL、Haskell、Java字节码、Objective-C、Swift、Python、Ruby、Rust、Scala以及C#等语言。

以上摘自维基百科

## 几种编译器简介

目前市面上常见的编译器有以下两种

- GCC（GNU Compiler Collection）
- LLVM

LLVM 我们上面已经稍微介绍过了，下面引用维基百科对GCC的定义

> GNU编译器套装（英语：GNU Compiler Collection，缩写为GCC），指一套编程语言编译器，以GPL及LGPL许可证所发行的自由软件，也是GNU项目的关键部分，也是GNU工具链的主要组成部分之一。GCC（特别是其中的C语言编译器）也常被认为是跨平台编译器的事实标准。1985年由理查德·马修·斯托曼开始发展，现在由自由软件基金会负责维护工作。
> 原名为GNU C语言编译器（GNU C Compiler），因为它原本只能处理C语言。GCC在发布后很快地得到扩展，变得可处理C++。之后也变得可处理Fortran、Pascal、Objective-C、Java、Ada，Go与其他语言。
> 许多操作系统，包括许多类Unix系统，如Linux及BSD家族都采用GCC作为标准编译器。

### LLVM与GCC

我们现在所使用的Xcode采用的是LLVM，以前曾经使用过GCC，见下表

| Xcode 版本 | 应用编译器     |
| ---------- | -------------- |
| < Xcode3   | GCC            |
| Xcode3     | GCC + LLVM     |
| Xcode4.2   | 默认LLVM-Clang |
| > Xcode5   | 废弃GCC        |

那么，同样是编译器，为何Xcode最终选择LLVM而舍弃Clang呢

- Apple对Objective-C新增的特性，GCC并未配合给予实现
- GCC编译器前后端代码耦合度过高
- license GCC限制了LLVM-GCC的开发

## LLVM 设计思想

以下是传统的三相设计思想

- 前端
- 优化器
- 后端

对于iOS开发者来说，整个流程可以简要概括为 Clang对代码进行处理形成中间层作为输出，llvm把CLang的输出作为输入生成机器码

### Frontend

下面就到了这篇文章的重点了，LLVM编译器的前端，Clang

### Clang

> 这个软件项目在2005年由苹果计算机发起，是LLVM编译器工具集的前端（front-end），目的是输出代码对应的抽象语法树（Abstract Syntax Tree, AST），并将代码编译成LLVM Bitcode。接着在后端（back-end）使用LLVM编译成平台相关的机器语言 。Clang支持C、C++、Objective C。
> 在Clang语言中，使用Stmt来代表statement。Clang代码的单元（unit）皆为语句（statement），语法树的节点（node）类型就是Stmt。另外Clang的表达式（Expression）也是语句的一种，Clang使用Expr来代表Expression，Expr本身继承自Stmt。节点之下有子节点列表（sub-node-list）。
> Clang本身性能优异，其生成的AST所耗用掉的内存仅仅是GCC的20%左右。FreeBSD操作系统自2014年1月发行的10.0版本开始将Clang/LLVM作为默认编译器[3]。

Clang的执行过程包含以下几步

1. 宏替换，头文件导入
2. 语法分析，代码切割为token
3. 组成AST（抽象语法树）
4. 生成中间码（IR）

下面我们创建一个CommandLine工程来试验一下，demo托管在Github

首先打开Xcode创建工程，语言选择objective-c接下来我们找到 main.m ,

#### 首先查看编译步骤

```bash
clang -ccc-print-phases ClangTest/main.m
```

可以看到输出

```sh
0: input, "ClangTest/main.m", objective-c
1: preprocessor, {0}, objective-c-cpp-output
2: compiler, {1}, ir
3: backend, {2}, assembler
4: assembler, {3}, object
5: linker, {4}, image
6: bind-arch, "x86_64", {5}, image
```

#### 查看预处理结果

也就是宏替换和头文件导入步骤

```sh
clang -E ClangTest/main.m
```

我们可以看到输出如下（前面部分省略）

```objc
# 1 "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/System/Library/Frameworks/Foundation.framework/Headers/FoundationLegacySwiftCompatibility.h" 1 3
# 185 "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/System/Library/Frameworks/Foundation.framework/Headers/Foundation.h" 2 3
# 10 "ClangTest/main.m" 2

int main(int argc, const char * argv[]) {
    @autoreleasepool {

        NSLog(@"Hello, World!");
    }
    return 0;
}
```

#### 代码切割token

```bash
clang -fmodules -fsyntax-only -Xclang -dump-tokens ClangTest/main.m
```

输出如下

```bash
annot_module_include '#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...'         Loc=<ClangTest/main.m:9:1>
int 'int'        [StartOfLine]  Loc=<ClangTest/main.m:11:1>
identifier 'main'        [LeadingSpace] Loc=<ClangTest/main.m:11:5>
l_paren '('             Loc=<ClangTest/main.m:11:9>
int 'int'               Loc=<ClangTest/main.m:11:10>
identifier 'argc'        [LeadingSpace] Loc=<ClangTest/main.m:11:14>
comma ','               Loc=<ClangTest/main.m:11:18>
const 'const'    [LeadingSpace] Loc=<ClangTest/main.m:11:20>
char 'char'      [LeadingSpace] Loc=<ClangTest/main.m:11:26>
star '*'         [LeadingSpace] Loc=<ClangTest/main.m:11:31>
identifier 'argv'        [LeadingSpace] Loc=<ClangTest/main.m:11:33>
l_square '['            Loc=<ClangTest/main.m:11:37>
r_square ']'            Loc=<ClangTest/main.m:11:38>
r_paren ')'             Loc=<ClangTest/main.m:11:39>
l_brace '{'      [LeadingSpace] Loc=<ClangTest/main.m:11:41>
at '@'   [StartOfLine] [LeadingSpace]   Loc=<ClangTest/main.m:12:5>
identifier 'autoreleasepool'            Loc=<ClangTest/main.m:12:6>
l_brace '{'      [LeadingSpace] Loc=<ClangTest/main.m:12:22>
identifier 'NSLog'       [StartOfLine] [LeadingSpace]   Loc=<ClangTest/main.m:14:9>
l_paren '('             Loc=<ClangTest/main.m:14:14>
at '@'          Loc=<ClangTest/main.m:14:15>
string_literal '"Hello, World!"'                Loc=<ClangTest/main.m:14:16>
r_paren ')'             Loc=<ClangTest/main.m:14:31>
semi ';'                Loc=<ClangTest/main.m:14:32>
r_brace '}'      [StartOfLine] [LeadingSpace]   Loc=<ClangTest/main.m:15:5>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<ClangTest/main.m:16:5>
numeric_constant '0'     [LeadingSpace] Loc=<ClangTest/main.m:16:12>
semi ';'                Loc=<ClangTest/main.m:16:13>
r_brace '}'      [StartOfLine]  Loc=<ClangTest/main.m:17:1>
eof ''          Loc=<ClangTest/main.m:17:2>
```

可以看到，括号，符号，关键字等等都被切割出来了

#### 语法分析，组成AST（抽象语法树）

```bash
clang -fmodules -fsyntax-only -Xclang -ast-dump ClangTest/main.m
```

可以看到AST输出如下:

```sh
TranslationUnitDecl 0x7f95b28032e8 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x7f95b2803b80 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x7f95b2803880 '__int128'
|-TypedefDecl 0x7f95b2803be8 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x7f95b28038a0 'unsigned __int128'
|-TypedefDecl 0x7f95b2803c80 <<invalid sloc>> <invalid sloc> implicit SEL 'SEL *'
| `-PointerType 0x7f95b2803c40 'SEL *' imported
|   `-BuiltinType 0x7f95b2803ae0 'SEL'
|-TypedefDecl 0x7f95b2803d58 <<invalid sloc>> <invalid sloc> implicit id 'id'
| `-ObjCObjectPointerType 0x7f95b2803d00 'id' imported
|   `-ObjCObjectType 0x7f95b2803cd0 'id' imported
|-TypedefDecl 0x7f95b2803e38 <<invalid sloc>> <invalid sloc> implicit Class 'Class'
| `-ObjCObjectPointerType 0x7f95b2803de0 'Class' imported
|   `-ObjCObjectType 0x7f95b2803db0 'Class' imported
|-ObjCInterfaceDecl 0x7f95b2803e88 <<invalid sloc>> <invalid sloc> implicit Protocol
|-TypedefDecl 0x7f95b28465e8 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x7f95b2846400 'struct __NSConstantString_tag'
|   `-Record 0x7f95b2803f50 '__NSConstantString_tag'
|-TypedefDecl 0x7f95b2846680 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x7f95b2846640 'char *'
|   `-BuiltinType 0x7f95b2803380 'char'
|-TypedefDecl 0x7f95b2846948 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list 'struct __va_list_tag [1]'
| `-ConstantArrayType 0x7f95b28468f0 'struct __va_list_tag [1]' 1
|   `-RecordType 0x7f95b2846770 'struct __va_list_tag'
|     `-Record 0x7f95b28466d0 '__va_list_tag'
|-ImportDecl 0x7f95b30612f8 <ClangTest/main.m:9:1> col:1 implicit Foundation
|-FunctionDecl 0x7f95b30615a8 <line:11:1, line:17:1> line:11:5 main 'int (int, const char **)'
| |-ParmVarDecl 0x7f95b3061348 <col:10, col:14> col:14 argc 'int'
| |-ParmVarDecl 0x7f95b3061460 <col:20, col:38> col:33 argv 'const char **':'const char **'
| `-CompoundStmt 0x7f95b2260ae8 <col:41, line:17:1>
|   |-ObjCAutoreleasePoolStmt 0x7f95b2260aa0 <line:12:5, line:15:5>
|   | `-CompoundStmt 0x7f95b2260a88 <line:12:22, line:15:5>
|   |   `-CallExpr 0x7f95b2260a40 <line:14:9, col:31> 'void'
|   |     |-ImplicitCastExpr 0x7f95b2260a28 <col:9> 'void (*)(id, ...)' <FunctionToPointerDecay>
|   |     | `-DeclRefExpr 0x7f95b2260910 <col:9> 'void (id, ...)' Function 0x7f95b30616e8 'NSLog' 'void (id, ...)'
|   |     `-ImplicitCastExpr 0x7f95b2260a70 <col:15, col:16> 'id':'id' <BitCast>
|   |       `-ObjCStringLiteral 0x7f95b22609b0 <col:15, col:16> 'NSString *'
|   |         `-StringLiteral 0x7f95b2260978 <col:16> 'char [14]' lvalue "Hello, World!"
|   `-ReturnStmt 0x7f95b2260ad0 <line:16:5, col:12>
|     `-IntegerLiteral 0x7f95b2260ab0 <col:12> 'int' 0
`-<undeserialized declarations>
```

#### 生成IR(intermediate representation)

这一步CodeGen会自顶向下遍历AST，产出中间层，也就是IR

```bash
clang -S -fobjc-arc -emit-llvm ClangTest/main.m -o main.ll
```

```sh
; ModuleID = 'ClangTest/main.m'
source_filename = "ClangTest/main.m"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.14.0"

%struct.__NSConstantString_tag = type { i32*, i32, i8*, i64 }

@__CFConstantStringClassReference = external global [0 x i32]
@.str = private unnamed_addr constant [14 x i8] c"Hello, World!\00", section "__TEXT,__cstring,cstring_literals", align 1
@_unnamed_cfstring_ = private global %struct.__NSConstantString_tag { i32* getelementptr inbounds ([0 x i32], [0 x i32]* @__CFConstantStringClassReference, i32 0, i32 0), i32 1992, i8* getelementptr inbounds ([14 x i8], [14 x i8]* @.str, i32 0, i32 0), i64 13 }, section "__DATA,__cfstring", align 8

; Function Attrs: noinline optnone ssp uwtable
define i32 @main(i32, i8**) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i8**, align 8
  store i32 0, i32* %3, align 4
  store i32 %0, i32* %4, align 4
  store i8** %1, i8*** %5, align 8
  %6 = call i8* @objc_autoreleasePoolPush() #2
  notail call void (i8*, ...) @NSLog(i8* bitcast (%struct.__NSConstantString_tag* @_unnamed_cfstring_ to i8*))
  call void @objc_autoreleasePoolPop(i8* %6)
  ret i32 0
}

declare i8* @objc_autoreleasePoolPush()

declare void @NSLog(i8*, ...) #1

declare void @objc_autoreleasePoolPop(i8*)

attributes #0 = { noinline optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #2 = { nounwind }

!llvm.module.flags = !{!0, !1, !2, !3, !4, !5, !6, !7}
!llvm.ident = !{!8}

!0 = !{i32 2, !"SDK Version", [2 x i32] [i32 10, i32 14]}
!1 = !{i32 1, !"Objective-C Version", i32 2}
!2 = !{i32 1, !"Objective-C Image Info Version", i32 0}
!3 = !{i32 1, !"Objective-C Image Info Section", !"__DATA,__objc_imageinfo,regular,no_dead_strip"}
!4 = !{i32 4, !"Objective-C Garbage Collection", i32 0}
!5 = !{i32 1, !"Objective-C Class Properties", i32 64}
!6 = !{i32 1, !"wchar_size", i32 4}
!7 = !{i32 7, !"PIC Level", i32 2}
!8 = !{!"Apple LLVM version 10.0.1 (clang-1001.0.46.4)"}
```


#### 查看生成的汇编代码

```sh
clang -S -fobjc-arc ClangTest/main.m -o main.s
```

如下：

```assembly
        .section        __TEXT,__text,regular,pure_instructions
        .build_version macos, 10, 14    sdk_version 10, 14
        .globl  _main                   ## -- Begin function main
        .p2align        4, 0x90
_main:                                  ## @main
        .cfi_startproc
## %bb.0:
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register %rbp
        subq    $32, %rsp
        movl    $0, -4(%rbp)
        movl    %edi, -8(%rbp)
        movq    %rsi, -16(%rbp)
        callq   _objc_autoreleasePoolPush
        leaq    L__unnamed_cfstring_(%rip), %rsi
        movq    %rsi, %rdi
        movq    %rax, -24(%rbp)         ## 8-byte Spill
        movb    $0, %al
        callq   _NSLog
        movq    -24(%rbp), %rdi         ## 8-byte Reload
        callq   _objc_autoreleasePoolPop
        xorl    %eax, %eax
        addq    $32, %rsp
        popq    %rbp
        retq
        .cfi_endproc
                                        ## -- End function
        .section        __TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
        .asciz  "Hello, World!"

        .section        __DATA,__cfstring
        .p2align        3               ## @_unnamed_cfstring_
L__unnamed_cfstring_:
        .quad   ___CFConstantStringClassReference
        .long   1992                    ## 0x7c8
        .space  4
        .quad   L_.str
        .quad   13                      ## 0xd

        .section        __DATA,__objc_imageinfo,regular,no_dead_strip
L_OBJC_IMAGE_INFO:
        .long   0
        .long   64


.subsections_via_symbols
```

#### 生成目标文件

```sh
clang -fmodules -c ClangTest/main.m -o main.o  
```

#### 生成可执行文件

```sh
clang main.o -o main
```

运行

```sh
./main
```

以这个例子来说，虽然我们一行代码都没有写，但是看得出来，编译器为我们做的事情可不少，接下来我想在编译器中添加一个插件，打印工程中所有的方法名
## Clang

说了这么多

下面我们从官网下载并编译最新的Clang

## 编译源码

1. 下载源码 git clone https://github.com/llvm/llvm-project.git
2. 创建build目录 cd llvm-project && mkdir build && cd build
3. 编译源码 cmake -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm 
4. cmake -build . 
5. make clang

### 测试是否成功

输入

```sh
./bin/clang-9 --version
```

正常输出clang就没什么问题了

## 替换Xcode的Clang

还是用刚才那个ClangTest工程，这里我们要替换掉Xcode使用的Clang为我们自己编译的版本

点击工程，找到Build Settings，点击加号，选择 Add User-Defined settings

添加如下两条

- CC /Users/felix/Documents/llvm-project/build/bin/clang-9
- CXX /Users/felix/Documents/llvm-project/build/bin/clang-9

将路径替换为你编译文件的路径

然后搜索Enable Index-While-Building Functionality ,将值更改为No

接下来可以Command+B进行编译
## Clang Plugin

下面来开发我们的第一个插件，打印所有的方法名

### 编写第一个CLang插件

首先，进入 llvm-project/clang/examples， 创建新文件夹，命名为Find

在Find下新建两个文件 DemoPlugin.cpp、CMakeLists.txt

打开 llvm-project/clang/examples/CMakeLists.txt，在文件末尾追加

```
add_subdirectory(DemoPlugin)
```

现在让我们编辑Find插件，


CMakeLists.txt

```c++
# If we don't need RTTI or EH, there's no reason to export anything
# from the plugin.
if( NOT MSVC ) # MSVC mangles symbols differently, and
               # PrintFunctionNames.export contains C++ symbols.
  if( NOT LLVM_REQUIRES_RTTI )
    if( NOT LLVM_REQUIRES_EH )
      set(LLVM_EXPORTED_SYMBOL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/DemoPlugin.exports)
    endif()
  endif()
endif()

add_llvm_library(DemoPlugin MODULE DemoPlugin.cpp PLUGIN_TOOL clang)

if(LLVM_ENABLE_PLUGINS AND (WIN32 OR CYGWIN))
	target_link_libraries(DemoPlugin PRIVATE
    clangAST
    clangBasic
    clangFrontend
    LLVMSupport
    )
endif()
```

DemoPlugin.cpp

```c++
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "clang/AST/AST.h"
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Frontend/CompilerInstance.h"

using namespace clang;

namespace
{
    // 可以深度优先搜索整个AST，并访问每一个基类，遍历需要处理的节点
    class DemoPluginVisitor : public RecursiveASTVisitor<DemoPluginVisitor>
    {
    private:
        CompilerInstance &Instance;
        ASTContext *Context;
        
    public:
        void setASTContext (ASTContext &context)
        {
            this -> Context = &context;
        }
        
        DemoPluginVisitor (CompilerInstance &Instance)
        :Instance(Instance) {}
        
        // 查找类名
        bool VisitObjCInterfaceDecl(ObjCInterfaceDecl *declaration) {
            if(isUserSourceCode(declaration)) {
                DiagnosticsEngine &D = Instance.getDiagnostics();
                unsigned diagID = D.getCustomDiagID(DiagnosticsEngine::Warning, "查找到一个类名: %0");
                D.Report(declaration->getBeginLoc(), diagID) << declaration->getName();
            }
            return true;
        }
        
        // 查找方法名
        bool VisitObjCMethodDecl(ObjCMethodDecl *declaration) {
            if(isUserSourceCode(declaration)) {
                DiagnosticsEngine &D = Instance.getDiagnostics();
                unsigned diagID = D.getCustomDiagID(DiagnosticsEngine::Warning, "查找到一个方法名: %0");
//                D.Report(declaration->getLocStart(), diagID).AddString(declaration->getSelector().getAsString());
                 D.Report(declaration->getBeginLoc(), diagID) << declaration->getSelector().getAsString();
            }
            return true;
        }
        
        // 是否用户代码
        bool isUserSourceCode (Decl *decl){
            std::string filename = Instance.getSourceManager().getFilename(decl->getSourceRange().getBegin()).str();
            
            if (filename.empty())
                return false;
            // 定义非Xcode中的源码都是用户源码
            if(filename.find("/Applications/Xcode.app/") == 0)
                return false;
            
            return true;
        }
    };
    
    class DemoPluginConsumer : public ASTConsumer
    {
    private:
        DemoPluginVisitor visitor;
        CompilerInstance &Instance;
        std::set<std::string> ParsedTemplates;
    public:
        DemoPluginConsumer(CompilerInstance &Instance,
                           std::set<std::string> ParsedTemplates)
        : Instance(Instance), ParsedTemplates(ParsedTemplates), visitor(Instance) {}
        
        // 每次分析到一个顶层定义时会回调此函数，返回true表示处理
        bool HandleTopLevelDecl(DeclGroupRef DG) override
        {
            return true;
        }
        
        // ASTConsumer的入口函数
        void HandleTranslationUnit(ASTContext& context) override
        {
            visitor.setASTContext(context);
            visitor.TraverseDecl(context.getTranslationUnitDecl());
        }
    };
    
    class DemoPluginASTAction : public PluginASTAction
    {
        std::set<std::string> ParsedTemplates;
    protected:
        std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &CI,
                                                       llvm::StringRef) override
        {
            return llvm::make_unique<DemoPluginConsumer>(CI, ParsedTemplates);
        }
        // 插件的入口函数
        bool ParseArgs(const CompilerInstance &CI,
                       const std::vector<std::string> &args) override {
            return true;
        }
    };
}

static clang::FrontendPluginRegistry::Add<DemoPluginASTAction>
X("DemoPlugin", "demo plugin");
```

## 加载插件

首先，我们需要编译插件，

```sh
cmake -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm
make DemoPlugin 
```

编译成功后我们可以在 build的lib 目录下找到 DemoPlugin.dylib


在工程中加入配置 Other C Flags ，加入以下配置

```sh
-Xclang -load -Xclang /Users/felix/Documents/llvm-project/build/lib/DemoPlugin.dylib -Xclang -add-plugin -Xclang DemoPlugin
```

## 调试

下面我们就可以开始编译工程了，可以看到每个方法名和类名都被找到拉

![clangTest](https://github.com/FelixScat/Pub/blob/master/image/clangTest.png?raw=true)

