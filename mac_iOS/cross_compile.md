# iOS应用开发中的交叉编译

## 什么情况下需要关注交叉编译

交叉编译是指在既定的机器平台下，为目标平台编译可执行机器代码。简单说就是代码运行环境和开发环境不在一起，有跨平台代码运行的诉求。可能的场景有

* 为嵌入式设备编写代码，通常嵌入式设备的体系结构不同，其本身也不太可能提供运行开发编译工具运行的环境。
* 代码需要跨多平台运行，这是通常应用开发的诉求，比如系统应用希望能跨Windows，Mac，Linux，甚至是Unix系统等，以从多个平台上获得更多的用户群体。

交叉编译本质上还是为了解决跨CPU体系结构。Apple生态内，包含其Mac个人电脑和移动设备，可穿戴设备等，主要由Intel和ARM两大体系结构组成。从应用开发的角度上看，
通常我们是不需要关注到这些细节，Xcode会处理好这些细节。但如果应用的功能诉求超出了OS内建的SDK能够提供的能力范畴之外，就需要自备扩展能力来解决。(比如游戏应用领域
对图形渲染有特殊要求等)。

拿iOS应用开发举例，我们开发的设备是Mac电脑，搭载macOS，运行环境是iPhone，搭载iPhoneOS系统，因为macOS主流运行环境是Intel(x86_64)体系，我们开发时使用的模拟器(simulator)是运行在Intel体系下的，包含其关联的需要链接的SDK，而iPhoneOS是运行ARM体系下，这已经需要我们的应用在一个完整的生命周期内跨不同体系运行。除此之外，如果我们对应用的兼容性有更高的要求(也是为了获得更多的客户群体)，我们还需要兼容32位和64位的CPU架构，甚至同体系下不同指令集也要单独支持。

| CPU体系 | 32位           | 64位           | 设备             |
| ----- | ------------- | ------------- | -------------- |
| Intel | i386          | x86_64        | 模拟器(Simulator) |
| ARM   | armv7, armv7s | arm64, arm64e | iPhone         |

## iOS下交叉编译的产出载体

交叉编译最终还是需要产出一个静态或者动态库，或者可以打包成Apple生态内的代码分发包(framework文件)，提供给Xcode在应用编译时链接。这里就会产生一个新的问题，因为开发环境和真实设备体系结构的不同，我们需要单独为它们提供不同的交叉编译库文件，由于库文件本身的构建流程是脱离Xcode应用集成环境之外，这就给应用开发维护上造成了巨大的负担和成本。

这里就需要使用到多体系二进制文件(```multiarchitecture binary```)，通常俗称```fat binary```(胖文件)，它本身是利用了跨体系通用的跳转指令来动态路由不同体系的入口，达成跨平台的目的。

胖文件的优缺点也比较明显，优点是集成多体系二进制代码到单个文件中，方便维护，缺点已经表现在名字上，因为整合了多套体系代码，文件大小会成倍增加。我们利用胖文件把模拟器和真机的目标代码都整合到一个文件上，打包成framework分发包，直接挂载到Xcode集成环境里。

## 交叉编译工具和实际操作

这里使用社区使用较多的工具cmake来作为构建工具，cmake本身是一个多种构建工具的上层抽象和封装，是一个构建工具生成器。支持生成Makefile，Xcode，Ninja等主流构建工具的工程结构。cmake历史悠久，社群也比较完善(关键是出了问题好找答案)。

### 示例工程

```shell
├── CMakeLists.txt
├── farmer.c
├── farmer.h
└── iosbuild.sh
```

基于一个普通的C工程```farmer```，提供一个定制的特殊C API供iOS应用层objc调用，API也比较简单，真实的场景下，API肯定复杂的多。最后提供的形式是一个支持多体系架构的胖文件，能够无缝整合进Xcode，运行在摸机器和真机上。

```c
// farmer.c
#include "farmer.h"
/**
 * lib version
 */
const char *farmer_get_version() {
  return "1.0.0";
}
```

### 生搬硬套法

基本的思想是分别为x86_64和arm64指令集编译两套代码(先简单考虑)，利用lipo命令将他们打包成一个胖文件。

cmake的思想是以项目为结构单元; 以产出导向来组织构建逻辑代码，也就是通常说的target。为了简单起见，采用Makefile工程结构构建，也就是采用cmake的```Unix Makefiles```生成器生成最终工程。

交叉编译本质上是选择兼容的编译器，以及要链接的对应平台的SDK。在macOS上，相关的工具链都组织在Xcode集成环境的相关目录下。

```shell
xcode-select -p
# /Applications/Xcode.app/Contents/Developer
```

编译器在macOS上自然是选择clang，只需要在编译时告诉编译器需要生成的目标平台代码即可，例如指定参数为```-arch x86_64```来生成在64位机器上的摸机器运行的代码。

在SDK的指定上，情况会稍微复杂。apple有多种产品设备，包括Mac，iPhone，穿戴设备watch，appleTV等。apple针对它们提供了不同的SDK，组织在Xcode相关目录下的对应平台内，抽象为Platform。

| SDK                    | Platform         | 实例版本                      |
| ---------------------- | ---------------- | ------------------------- |
| DriverKit              | driverkit        | -sdk driverkit21.4        |
| iOS SDKs               | iphoneos         | -sdk iphoneos15.5         |
| iOS Simulator SDK      | iphonesimulator  | -sdk iphonesimulator15.5  |
| macOS SDKs             | macosx           | -sdk macosx12.3           |
| tvOS SDKs              | appletvos        | -sdk appletvos15.4        |
| tvOS Simulator SDKs    | appletvsimulator | -sdk appletvsimulator15.4 |
| watchOS SDKs           | watchos          | -sdk watchos8.5           |
| watchOS Simulator SDKs | watchsimulator   | -sdk watchsimulator8.5    |

通过xcrun命令，就能找到我们需要的iphoneos的SDK位置

```shell
xcrun --show-sdk-path -sdk iphoneos15.5
# /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.5.sdk
```

最终传递给编译器的参数如下

```shell
-arch arm64 \
-pipe \
-isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.5.sdk \
-O3 \
-DNDEBUG \
-miphoneos-version-min=6.0 \
-fembed-bitcode
```

实际的生产项目中，platform以及SDK的位置需要使用脚本动态获取，xcode相关的命令工具可以帮助获得。思路是首先获得当前platform下xcode SDK最新的版本，利用版本可以获得完整的SDK路径。当然plaform和arch参数的映射关系需要自己维护。

```shell
#(1)获取platform iphoneos的SDK版本号
xcodebuild -showsdks | grep iphoneos | sort | tail -n 1 | awk '{print $2}'
# 15.5

#(2)通过platform的版本号，获取SDK的完整路径
xcrun --show-sdk-path -sdk iphoneos15.5
# /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.5.sdk
```

在cmake中，cmake_c_flags接受编译器参数，完整的cmake代码如下

```cmake
cmake_minimum_required(VERSION 3.7)
project(farmer C)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -arch arm64 -pipe -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.5.sdk -O3 -DNDEBUG -miphoneos-version-min=6.0 -fembed-bitcode")
add_library(farmer STATIC farmer.c)
target_include_directories(farmer PUBLIC ${CMAKE_SOURCE_DIR})
```

通过分别为x86_64和arm64构建cmake产出，在利用lipo命令合并2个体系的代码。最后按照framework的包规范打包整合到xcode中即可。



### 封装工具链法