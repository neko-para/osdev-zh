# [GCC交叉编译器 | GCC Cross-Compiler](https://wiki.osdev.org/GCC_Cross-Compiler)

本教程将侧重于如何为你自己的操作系统创建一套GCC交叉编译器。我们这里创建的编译器将会使用通用的目标架构（i686-elf），让你可以脱离当前的操作系统，也就是说你的操作系统不会使用当前系统中任何的头文件和库。

---

1. [简介](#简介)

    1.1. [为什么必须要交叉编译器](#为什么必须要交叉编译器)

    1.2. [选择哪个编译器版本](#选择哪个编译器版本)

    1.3. [选择哪个Binutils版本](#选择哪个Binutils版本)

    1.4. [决定目标平台](#决定目标平台)

    1.5. [关于arm-none-eabi-gcc](#关于arm-none-eabi-gcc)

2. [构建前的准备工作](#构建前的准备工作)

    2.1. [安装依赖](#安装依赖)

    2.2. [下载源代码](#下载源代码)

    2.3. [Linux用户构建系统编译器](#Linux用户构建系统编译器)

    2.4. [Gentoo用户](#Gentoo用户)

    2.5. [MacOS用户](#MacOS用户)

    2.6. [Windows用户](#Windows用户)

    2.7. [OpenBSD用户](#OpenBSD用户)

3. [构建](#构建)

    3.1. [准备工作](#准备工作)

    3.2. [Bintuils](#Binutils)

    3.3. [GCC](#GCC)

    3.4. [使用新编译器](#使用新编译器)

    3.5. [故障排除](#故障排除)

4. [更进一步](#更进一步)

5. [参考](#参考)

---

## 简介

通常，一个交叉编译器是指一个运行在A平台（宿主）但是生成B平台（目标）程序的编译器。这两个平台可以有（但不必须）CPU、操作系统、[可执行文件格式](Category:Excutable_Formats.md)的区别。对于我们的情况，宿主平台是你当前的操作系统，目标平台是你准备创建的操作系统。你应当理解这两个平台是不同的，你开发的操作系统和你正在使用的总是不同的。这就是为什么我们需要先创建交叉编译器，否则你几乎一定会遇到问题。

### 为什么必须要交叉编译器

> 主文章：[为什么我需要交叉编译器](Why_do_I_need_a_Cross_Compiler?.md)

除非你在你正在创建的操作系统上开发，否则你总是需要使用交叉编译器。编译器必须清楚正确的目标平台（CPU、操作系统），否则你会遇到问题。如果你使用你操作系统自带的编译器，它并不知道自己正在编译完全不同的东西。有些教程会建议你使用系统自带的编译器并且传入很多有问题的选项给编译器。这一定会在未来给你带来更多的问题，而解决方法就是构建一个交叉编译器。如果你已经尝试过不用交叉编译器开发操作系统，请阅读文章[为什么我需要交叉编译器](Why_do_I_need_a_Cross_Compiler?.md)。

### 选择哪个编译器版本

> 主文章：[构建GCC](Building_GCC.md)

推荐最新版本的[GCC](GCC.md)，因为它是最新和最好的发行版本。例如，如果你用GCC 4.6.3来构建GCC 4.8.0的交叉编译器，可能就会遇到问题。如果你的操作系统的GCC编译器不是最新的主版本，我们建议你[构建最新的GCC作为你系统的编译器](Building_GCC.md)。

你也可以使用足够好的旧版本。如果你本地的系统编译器不是过于老旧（至少GCC4.6.0），你也可以简单的选择最新的次版本（假设你系统编译器版本是4.6.1，考虑4.6.3）作为你交叉编译器来避免遇到问题。

你可以通过执行以下命令来查看你当前编译器的版本：

```shell
gcc --version
```

你也许也能通过低主版本的GCC构建一个高主版本的GCC。比如，GCC 4.7.3也许能够构建一个GCC 4.8.0交叉编译器。但是，如果你想使用最新最好的GCC交叉编译器，我们建议你先[自举最新的GCC](Building_GCC.md)来作为你的系统编译器。使用OS X 10.7或更老的用户也许希望构建一个系统GCC编译器（输出宿主目标Mach-O）或者升级本地安装的LLVM/Clang。使用OS X 10.8或更新的用户应当安装苹果的开发者网站上的命令行工具并且使用Clang来构建GCC交叉编译器。

### 选择哪个Binutils版本

> 主文章：[成功构建的交叉编译器](Cross-Compiler_Successful_Builds.md)

我们建议你使用最新最好的[Binutils](Binutils.md)。但是需要注意的是，并不是所有GCC和Binutils的组合能用。如果你遇到了问题，考虑使用一个和你想要的编译器版本发布时间几乎一样的Binutils。你也许需要最低Binutils 2.22，更好的话至少Binutils 2.23.2。你系统使用的Binutils版本不影响。你可以通过以下命令查看Binutils的版本。

```shell
ld --version
```

### 决定目标平台

> 主文章：[目标平台组合](Target_Triplet.md)

你应当已经了解这些。如果你正在跟随[骨架](Bare_Bones.md)教程，你会希望构建一个针对i686-elf的交叉编译器。

### 关于arm-none-eabi-gcc

在Debian/Ubuntu的apt-get上可以安装已经编译好的gcc-arm-none-eabi包，但你不应该使用它。它里面即没有libgcc.a也没有如stdint.h等独立环境下的C头文件。

相反，你应当使用arm-none-eabi作为目标来构建你自己的交叉编译器。

## 构建前的准备工作

GNU编译器套件（GNU Compiler Collection）是有依赖的复杂程序。你需要以下环境来构建GCC：

* 一个类Unix环境（Windows用户可以使用WSL（适用于Linux的Windows子系统）或Cygwin）

* 足够的内存和硬盘空间（看具体情况，256MiB不够）

* GCC（你希望替换的已存在的版本），或其它系统C编译器

* G++（如果GCC版本高于4.8.0），或其它系统C++编译器

* Make

* Bison

* Flex

* GMP

* MPFR

* MPC

* Texinfo

* ISL（可选）

* CLooG（可选）

### 安装依赖

|依赖\OS|源代码|Debian（Ubuntu、Mint、WSL……）|Gentoo|Fedora|Cygwin|OpenBSD|Arch|
| --- | --- | --- | --- | --- | --- | --- | --- |
|如何安装|直接构建|`sudo apt install foo`|`sudo emerge --ask foo`|`sudo dnf install foo`|Cygwin图形化安装器|`doas pkg_add foo`|`pacman -Syu foo`|
|编译器|/|build-essential|sys-devel/gcc|gcc gcc-c++|mingw64-x86_64-gcc-g++/mingw64-i686-gcc-g++|已预装|base-devel|
|Make|/|build-essential|sys-devel/make|make|make|已预装|base-devel|
|[Bison](https://www.gnu.org/software/bison)|[1](https://ftp.gnu.org/gnu/bison)|bison|sys-devel/bison|bison|bison|?|base-devel|
|[Flex](https://github.com/westes/flex)|[2](https://github.com/westes/flex/releases)|flex|sys-devel/flex|flex|flex|?|base-devel|
|[GMP](https://gmplib.org/)|[3](https://ftp.gnu.org/gnu/gmp/)|libgmp3-dev|dev-libs/gmp|gmp-devel|libgmp-devel|gmp|gmp|
|MPC|[4](https://ftp.gnu.org/gnu/mpc/)|libmpc-dev|dev-libs/mpc|libmpc-devel|libmpc-devel|libmpc|libmpc|
|[MPFR](https://www.mpfr.org/)|[5](https://ftp.gnu.org/gnu/mpfr/)|libmpfr-dev|dev-libs/mpfr|mpfr-devel|libmpfr-devel|mpfr|mpfr|
|[Texinfo](https://www.gnu.org/software/texinfo/)|[6](https://ftp.gnu.org/gnu/texinfo/)|texinfo|sys-apps/textinfo|texinfo|texinfo|texinfo|base-devel|
|[CLooG](https://www.cloog.org/)（可选）|[CLooG](https://www.cloog.org/)|libcloog-isl-dev|dev-libs/cloog|cloog-devel|libcloog-isl-devel|/|/|
|[ISL](http://isl.gforge.inria.fr/)（可选）|[7](http://isl.gforge.inria.fr/)|libisl-dev|dev-libs/isl|isl-devel|libisl-devel|/|/|

你需要安装Texinfo来构建Binutils。你需要安装GMP、MPC、MPFR来构建GCC。GCC可以利用CLooG库和ISL库。

例如，你可以通过运行`sudo apt install libgmp3-dev`来在Debian上安装`libgmp3-dev`。

注意：5.x（及更新）版本的Texinfo与Binutils 2.23.2（及更老）不兼容。你可以通过`makeinfo --version`来查看当前的版本。如果版本过新并且在构建时遇到了问题，你将要么使用Binutils 2.24（及更新），或者通过源代码安装一个旧版本的Textinfo，并且在构建Binutils时将其添加到PATH前。

注意：0.13（及更新）版本的ISL与CLoog 0.18.1（及更老）不兼容。需要使用0.12.2版本的ISL，否则构建会失败。

### 下载源代码

下载需要的源代码到一个合适的目录，如$HOME/src

* 你可以从[Binutils网站](https://gnu.org/software/binutils/)下载或直接访问[GNU主镜像](https://ftp.gnu.org/gnu/binutils/)。

* 你可以从[GCC网站](https://gnu.org/software/gcc/)下载或直接访问[GNU主镜像](https://ftp.gnu.org/gnu/gcc/)

注意：版本号使用的模式是每个点分隔整个数字。比如，Binutils 2.20.0比2.9.0新。如果你没见过这种版本模式的话，也许很奇怪：对于按照字母数字排序的文件列表，最下方的文件并不是最新的版本！获取最新版本的一个简单办法就是根据最后修改时间来排序，然后滚动到最下方。

### Linux用户构建系统编译器

你的发行版也许会提供经过它打过补丁、以使其能在该特定平台的Linux上运行的GCC和Binutil。这能够用于构建交叉编译器，但是你也许不能用它为你的Linux构建一个新的系统编译器。对于这种情况，请考虑使用一个新的GCC或者获取被打过补丁的源代码。

### Gentoo用户

> 待翻译

### MacOS用户

> 待翻译

### Windows用户

Windows用户需要准备一个类Unix环境如[MinGW](https://wiki.osdev.org/MinGW)或[Cygwin](https://wiki.osdev.org/Cygwin)。查看如Linux等的系统并且看看是否能满足你的需求是值得的，因为你通常需要在操作系统开发中使用很多类Unix的工具，而这在一个类Unix系统上更容易操作。**如果你才安装了基本的[Cygwin](https://wiki.osdev.org/Cygwin)包，那么你将不得不重新运行setup.exe并且安装如下包：**GCC、G++、Make、Flex、Bison、Diffutils、libintl-devel、libgmp-devel、libmpfr-devel、libmpc-devel、Texinfo。

MinGW配合MSYS也是一种选择，并且它直接使用本地的Windows API而不是POSIX的模拟层，会提供一套更快的工具链。一部分软件包可能并不能正确的在MSYS上构建，因为它们并不是为运行在Windows上设计的。对于所有本教程关注的内容，所有适用于Cygwin的也适用于MSYS，除非额外说明。请确保安装了C和C++编译器，以及MSYS基础系统。

在Windows 10周年更新上发布的“适用于Linux的Windows子系统（测试版）”，也是一个使用交叉编译器的可选项。（于2016年8月8日，使用GCC 6.1.0和Binutils 2.27通过测试）这个交叉编译器工作地相当快，即使作为测试状态，它不适合作为长期的开发平台。

> 译者注：现在WSL已经是正式版了

Cygwin注意：Cygwin在它自己bash的PATH中包含了Windows系统的PATH。如果你曾经使用过DJGPP，这可能会导致混乱。如，在Cygwin的bash调用GCC仍然会调用DJGPP的编译器。在卸载DJGPP后，你应当删除DJGPP的环境变量并从PATH中删除C:\djgpp目录（或任何你安装它的地方）。同样，将多个构建环境混在你系统的PATH变量中是一个坏主意。

MinGW注意：在MinGW主页的[作为交叉编译器宿主时的指导页](http://www.mingw.org/wiki/HostedCrossCompilerHOWTO)可以找到某些在构建交叉编译器时MinGW特有的信息。

WSL注意：你不能将你的交叉编译器构建在/mnt/c（或/mnt/"x"），这会在构建时发生错误，但是在$HOME/opt/cross构建没有问题。这在Windows更新KB3176929中被修复。

### OpenBSD用户

> 待翻译

## 构建

我们将构建一个运行在你的宿主系统、可以将源代码转换成适用于目标平台的目标文件。

你需要决定将新的编译器安装在何处。将其安装到系统目录中是一个非常危险的坏主意。你也需要决定这个新编译器是为全局还是你自己安装。如果你为自己安装（推荐），将其安装到$HOME/opt/cross通常情况下是一个的好主意。如果你想为全局安装，将其安装到/opt/cross通常情况下是一个的好主意。

请注意，作为最佳实践，我们将在源代码目录外构建所有东西。一些包只支持在目录树外构建，有些只能在内部，有些都支持（但也许没有在make中提供广泛的检查）。至少对于旧版本的GCC，在目录树内构建会可悲的失败。

### 准备工作

```shell
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"
```

我们将安装目录的PREFIX添加到当前shell会话的PATH中。这可以保证在我们构建了Binutils后，在编译器的构建过程中可以找到它们。

PREFIX会配置构建过程，因此你交叉编译环境的所有文件都将位于$HOME/opt/cross内。你可以将PREFIX改变成任何你喜欢的目录（如，/opt/cross、$HOME/cross都是可选项）。如果你有管理员权限并且你希望让交叉编译工具链能够被所有用户访问，你可以将其安装到/usr/local，或者/usr/local/cross，如果你希望更改系统配置信息（比如将该目录添加到所有用户的搜索目录中）。技术上说，你甚至可以将其直接安装到/usr，这样你的交叉编译器将会和你系统的编译器放在一起，但因为若干原因不建议这么做（比如如果你TARGET配置错了会有覆盖系统编译器的风险，导致你系统的包管理出现冲突）。

### Binutils

```shell
cd $HOME/src

mkdir build-binutils
cd build-binutils
../binutils-x.y.z/configure --target=$TARGET --prefix="$PREFIX" --with-sysroot --disable-nls --disable-werror
make
make install
```

这将会编译运行在你的系统但处理被$TARGET设置的平台的代码的Binutils（汇编器、反汇编器以及许多有用的东西）。

**--disable-nls**让Binutils不包含本地语言支持。这是可选的，但减少了依赖和编译时间。它也会导致生成英文的提示信息，让你在[论坛](http://forum.osdev.org/)上提问时其他人能够看懂。

**--with-sysroot**让Binutils启用sysroot支持，让其指向一个默认的空目录。默认情况下，链接器没有技术原因地拒绝使用sysroot，但GCC能够在运行时处理这两种情况。这在之后有用。

### GCC

> 参考：[配置gcc的官方步骤](http://gcc.gnu.org/install/configure.html)

现在，你可以构建GCC。

```shell
cd $HOME/src

which -- $TARGET-as || echo $TARGET-as is not in the PATH

mkdir build-gcc
cd build-gcc
../gcc-x.y.z/configure --target=$TARGET --prefix="$PREFIX" --disable-nls --enable-languages=c,c++ --without-headers
make all-gcc
make all-target-libgcc
make install-gcc
make install-target-gcc
```

我们构建libgcc，一个编译器在编译时期望存在的底层支持库。和libgcc链接提供了整数、浮点数、实数、堆栈恢复（在异常处理时有用）以及其它支持函数。注意我们*不*是简单的允许`make && make install`因为这会构建过多的内容，但不是所有gcc的部件能够适配你未完成的操作系统。

**--disable-nls**和上面Binutils的一样。

**--without-headers**让GCC不要依赖任何存在且适用于目标平台的C库（标准或是运行时）。

**--enable-languages**让GCC不要编译其它语言的前端而只支持C（以及可选的C++）。

构建你的交叉编译器会花费一些时间。

如果你在构建一个适用于x86_64的交叉编译器，你也许会考虑构建不带红区的libgcc：[不带红区的Libgcc](Libgcc_without_red_zone.md)。

### 使用新编译器

现在你已经有一个“裸”的交叉编译器了。它不能访问C库或C运行时，所以你不能使用任何标准头文件或者构建可执行文件。但是它足够用于编译你短期内开发的内核了。你的工具链位于$HOME/opt/cross（或你设置的PREFIX）。例如，你的GCC可执行文件被安装在$HOME/opt/cross/bin/$TARGET-gcc，它可以构建适用于目标平台的程序。

你可以类似如下的命令来运行你的新编译器：

```shell
$HOME/opt/cross/bin/$TARGET-gcc --version
```

注意这个编译器*不*能编译一般的C程序。只要你尝试去包含任何标准头文件（除了一小部分平台无关，由编译器自己生成的头文件）。这是正常的，因为你还没有一个适用于你目标系统的标准库。

C标准规定了两种执行环境：“freestanding（独立的）”和“hosted（有宿主的）”。这个的定义对于一般的应用程序开发者也许会相当模糊，但是当你做OS开发时是很清晰的：内核是“独立的”，任何你在用户空间做的都是“有宿主的”。一个“独立的”环境只需要提供C库的一个子集：float.h、iso646.h、limits.h、stdalign.h、stdarg.h、stdbool.h、stddef.h、stdint.h和stdnoreturn.h（C11的一部分）。所有的这些都只包含typedef和#define，所以你不用一个.c文件就能实现它们。

为了可以只需要执行$TARGET-gcc就能使用你的新编译器，通过以下命令将$HOME/opt/cross/bin添加到你的PATH中：

```shell
export PATH="$HOME/opt/cross/bin:$PATH"
```

这条命令会将你的新编译器添加到当前会话的PATH中。如果你希望将其永久添加到PATH中，将这条命令添加到你的`~/.profile`配置脚本或者类似的文件。查询你shell的文档来获取更多信息。

如果你是从[骨架](Bare_bones.md)来到这里，你可以用你新构建的编译器继续前进了。如果你构建了一个新的GCC来作为系统编译器然后用起构建你的交叉编译器，你现在可以安全的卸载它了，除非你还希望继续使用它。

### 故障排除

通常情况下，**核实**你是否仔细阅读了步骤并且准确地键入了命令。不要跳过步骤。如果你使用了新的shell会话且没有将其永久添加到你shell的配置文件中，你需要重新设置PATH。如果一次编译看上去完全混乱了，键入`make distclean`，然后重新启动make进程。确保你的解压软件不会改变换行符。

#### ld: cannot find -lgcc

你通过`-gcc`开关指定了需要链接GCC底层运行时库到你的可执行文件，但是忘记了构建并正确安装它。

如果你在安装libgcc时没有遇到警告或错误但仍然遇到这个问题，你可以复制libgcc到你项目的目录然后使用`-L. -lgcc`来链接它。

libgcc库位于$PERFIX/lib/gcc/$TARGET/<gcc版本>/libgcc.a。

#### Binutils 2.9

通过字母数字排序的列表的底部不一定是最新的版本。在2.9之后有2.10，2.11，2.12并且那里有更多更新的发行版，它们更加适合构建或支持你选择的GCC版本。

#### Building GCC: the directory that should contain system headers does not exist

你也许会在构建mingw32的目标（比如x86_64-w64-mingw32）是遇到这个问题。这里找不到的目录是$SYSROOT/mingw/includes。如果你查看你的sysroot，你当然会发现里面这些目录不存在。

解决方案就是简单的创建这些空的目录：

```shell
mkdir -p $SYSROOT/mingw/include
mkdir -p $SYSROOT/mingw/lib
```

这会让构建工作。这个问题发生的原因是mingw32（以及mingw自己）配置INCLUDE_PATH和LIBRARY_PATH为了/mingw/include和/mingw/lib，而不是默认的/usr/include和/usr/lib。至于为什么即使不需要这个目录中的任何内容构建还是会失败，以及为什么它不简单的创建这些目录，就不是我所知道的。

#### GCC libsanitizer failing to build

某些时候GCC无法构建libsanitizer，在这种情况下在配置命令中添加`--disable-libsanitizer`。

这只会在构建宿主编译器时遇到。

## 更进一步

使用这个简单的交叉编译器在相当长时间内都足够用了，但在某个时间点你会希望编译器能自动包含你自己的系统头文件和库。构建一个针对你自己操作系统的[OS相关工具链](OS_Specific_Toolchain)是你需要从这里更进一步的方向。

## 参考

> 待翻译