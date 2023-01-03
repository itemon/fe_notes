# 构建移动应用的本地全文搜索
搜索能力是移动应用能力很重要的一个方面。本地全文搜索主要解决以下几种场景的数据检索问题
* 离线文档，存储于本地，有检索的需求，如大部分的笔记应用等。
* 海量的个人数据，在本地有快速检索诉求，例如微信的聊天记录等。

# 本地全文检索的可选方案
* SQLite的FTS模块，当前开发到FTS5版本。FTS模块是一种SQLite的virtual table模块，virtual table机制是SQLite的一种逻辑表机制，它支持在核心代码上层自定义处理逻辑，带来的是灵活度的提升。当下大部分的移动应用的本地全文检索都是采用扩展FTS的方案，比如微信团队的本地联系人和聊天记录等数据的全文检索方案。

* 另外一种方案就是单独搭建一套完整独立的搜索方案，将本地数据双写到SQLite业务层和搜索引擎存储层，并建立数据的实时更新机制。引擎的可选方案有clucene，xapian等开源方案等。考虑到成熟度、项目维护度等因素，选用xapian作为引擎方案,iOS作为平台来搭建，以此展开本文。

# xapian的整合方案

## 示例工程
示例的搜索数据集采用了2021年财富500强企业列表数据。思路大致是，在应用首次启动时，将数据双写到Core Data和xapian。因为是数据集是中文的，写入到xapian的数据首先要做分词处理，分词采用的是iOS内建的string tokenier，当然也可以使用结巴分词等三方方案，灵活度更好。

整体步骤如下
* 通过交叉编译+合成共享库，准备好搜索引擎基础库
* 将搜索引擎基础库封装C的API提供给Objective-C调用
* 将准备好的财富500强企业列表数据双写到SQLite业务库和搜索引擎存储层
* 使用iOS的SearchController组件搜索查询诗词数据(搜索词依然采用iOS内建的分词器分词)

## xapian引擎
xapian是一款开源的搜索引擎方案，本身采用c++编写，提供了多种语言的binding层调用，同时还提供了搜索服务解决方案Omega。xapian的工程构建方案采用的是autotool，autotool是一种上层编译工具，最终会生成标准的Makefile来编译可执行代码，全称是GNU Autotool。Autotool包含三个工具集Autoconf, Automake, and Libtool。Autotool的工程的标准构建流程如下
```bash
#第一步
./configure --prefix=/path/to/folder

#第二步
make

#第三步
make install
```
Autotool构建相关的文件非常复杂，代码量很大，其实大部分都是工具自动生成的代码，工程的项目构建相关的逻辑都在Makefile.am和Makefile.ac里。它们一个定义「编什么」，一个定义「怎么编」。

因为我们需要同时在iOS模拟器上调式，在iPhone真机上运行，所以我们需要交叉编译出两套不同架构的二进制执行文件，再将它们合并为跨平台的共享库。在autotool上进行交叉编译需要弄清楚几个特定的参数。
```bash
--build=BUILD
```
指定当前编译代码的平台，参数的格式为```cpu-vendor-os```，如编译的机器为Intel处理器的Mac平台，参数可声明为```x86_64-apple-darwin```。

```bash
--host=BUILD
```
指定编译的目标平台，也就是代码真正运行时的平台，如编译为iPhone上的可执行代码，参数可声明为```arm64-apple-darwin```。

### 编译ARM架构共享库
此外我们还需要告诉编译器交叉编译工具链的目录参数等。如果想要编译出iPhone真机上的可执行代码，```host```参数指定为arm64架构。这里可执行文件的形式选择为共享动态库。在xapian工程根目录下新建```BUILD```和```INST```文件夹，分别存放构建中间文件和构成生成的共享库安装目录。
```bash
readonly ARCH=arm64
readonly FLG="-arch $ARCH -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS16.2.sdk -miphoneos-version-min=11.0 -fembed-bitcode -O3 -DNDEBUG"

updir=$(dirname `pwd`)

../configure \
  --host="${ARCH}-apple-darwin" \
  --build=x86_64-apple-darwin21.4.0 \
  --prefix=${updir}/INST \
  --enable-shared --disable-static \
  CXXFLAGS="$FLG" \
  CFLAGS="$FLG" \
  --disable-documentation

make
```

#### 解决xapian在iOS上文件锁的问题
上边编译的共享库如果直接在iOS上运行，实例化xapian存储数据库对象时会报错。原因是xapian的数据存储文件需要使用排它锁打开，默认情况下，它采用的是父子进程配合的方式，调用Posix风格的fcntl获取文件的独占锁，因为iOS系统本身禁用了fork系统调用，因此无法fork出子进程来获取锁操作。
为了解决这个系统限制，可以手动强制xapian使用BSD风格的flock系统调用获取文件锁，缺点是要修改xapian的源代码。

找到xapian根目录下configure.ac文件，修改931行，如果编译的目标平台系统是darwin内核，直接切换到flock锁机制。
```bash
  case $host_os in 
    *djgpp* | *msdos* | *darwin* ) #在这里追加darwin选项
      dnl DJGPP has a dummy implementation of fork which always fails.                                                 
      dnl                                                                                                              
      dnl For disk-based backends, use flock() for locking, which doesn't need                                         
      dnl fork() or socketpair().
      AC_DEFINE([FLINTLOCK_USE_FLOCK], 1, [Define to use flock() for flint-compatible locking])                        
      ;;                                                                                                               
  esac
```

然后更新下autotool构建文件，重新执行configure即可
```bash
autoreconf --install
```

### 编译x86_64架构共享库
由于编译机器是Intel架构处理器，所以需要给模拟器编译一份x86的调式共享库共模拟器使用
```bash
readonly ARCH=x86_64
readonly FLG="-arch $ARCH -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator16.2.sdk -miphoneos-version-min=11.0 -fembed-bitcode -O3 -DNDEBUG"

updir=$(dirname `pwd`)

../configure \
  --host="${ARCH}-apple-darwin" \
  --build=x86_64-apple-darwin21.4.0 \
  --prefix=${updir}/INST \
  --enable-shared --disable-static \
  CXXFLAGS="$FLG" \
  CFLAGS="$FLG" \
  --disable-documentation

make
```

为了能同时在模拟器和真机上无缝运行，利用lipo命令将2种体系架构下的共享库合并为统一的Mach-O可执行文件，优点是调试和提交应用商店时，都只需要链接同一份动态库，缺点是合并的动态库体积会大一倍，比如libxapian.dylib的大小从2M+增加到4M+。
```bash
#合并为统一的Fat binary file
lipo -create x86_64/libxapian.dylib arm64/libxapian.dylib -output libxapian.dylib

#修改动态链接库的查找路径为rpath(要不链接时找不到文件)
install_name_tool -id @rpath/libxapian.dylib ./libxapian.dylib
```

## 新建iOS工程
新建一个iOS工程，通过UIKit实现搜索交互调用xapian共享库实现搜索。首先需要把共享库添加到xcode中。
* 同时添加libc++.tbz和libz.tbz基础库到xcode中。
* 添加xapian的头文件组到xcode的系统头文件搜索路径中，方法是手动的把xapian工程根目录下的xapian.h和xapian目录下的头文件全部添加到工程中，并添加到系统文件搜索路径中。
* 使用UISearchController构建搜索交互界面。
* 添加待导入的原始数据，生产环境的正式应用可能从服务器接口获得同步的本地数据，本文采用一个SQLite本地数据作为本地数据的来源。