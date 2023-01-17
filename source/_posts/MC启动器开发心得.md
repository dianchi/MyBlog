---
title: MC启动器开发心得
date: 2023-01-11 21:50:38
tags: 技术
---
这几天翻仓库发现我以前写过MC的JE原版启动器，现在看看简直是堆屎山，就来以未来人视角审视一下自己的代码问题，并且谈一些MC启动器开发要用到的技术，这篇文章不重点将代码，而是启动器运行机理，和怎样做好启动器。
## 为什么有启动器
### 启动器干什么
启动器根据名称来看就知道，是一种用来启动的程序，启动什么？启动MC啊，还能启动什么。那启动器干了什么呢？
我们都知道MC:JE版是由Java编写的游戏，Java程序运行在JVM虚拟机中，而启动方式也和正常的二进制文件不同（当然部分Java写的发行程序会被打包成二进制程序，至少MC没这样做），我们常见的启动方式就是 `java -jar xxx.jar` 这样的启动方式，但MC肯定不能这样做啊，不然那么一大坨 /.minecraft/ 文件夹干啥的，启动器还需要将MC所需要的依赖库，资源索引，玩家信息，JVM参数加载到虚拟机中一起启动，这样我们的MC才能正确的调用、加载。其实你完全可以手动编写启动命令，启动器只是帮你减少工作量，不然玩游戏还得看懂运行原理是不是太掉价了。但作为开发者，掌握这些至关重要。
### 启动器的附加功能
如果单纯只是作为启动游戏来用的话不如用官方启动器，所以很多启动器作者为了让自己的启动器在市场抢占中占据优势，往往会选择给启动器加入各种附加功能。现在我们看到的第三方游戏启动器，最次的也会带一个Forge安装功能，像PCL等启动器还会加入诸如联机、MOD管理、CurseForge资源下载等等。启动器也渐渐变的不止“启动”了。其实在我们小时候的印象里，多玩MC盒子这类的MC盒子就已经有了这些概念（包括现在的网易中国版启动器），他们集成了游戏启动、好友联机、模组下载等等，构建了一个社区整体。只不过因为网易的铁拳，所以这些盒子的时代落下帷幕，取而代之的就是各类启动器的附加功能，这些附加功能往往不是直接由启动器作者运营，这样就不构成一个社区，网易也就没办法，所以我们还是管这些叫启动器，因为构成社区的就叫游戏盒子了。掌握这些附加功能会使你开发的启动器在市场中有优点。
## 正式动手吧
### 概念
启动器的本质在上面已经说过了，那么我知道了运行原理，我该怎么实现呢？实际上就是解析字符串然后拼接成启动指令最后执行而已，就这么简单，在每个游戏的版本文件夹下面我们都会看到一个 `版本号.json` 文件，这里面提示了我们所需要加载的依赖库、JVM参数、和必须传递的其他参数，我们要做的，就是解析他们，然后拼合成启动指令。
### 正式制作
#### 游戏下载
Minecraft的游戏下载你可以使用MCBBS源或者其他的下载源，我这里以官方下载源为例，其他的下载源请参考其文档，如果只是单纯的镜像站那代码部分可以通用。
我们通过访问 [version_manifest.json](https://piston-meta.mojang.com/mc/game/version_manifest.json) （本文主要使用v1版文件，也是比较通用的文件，其实你也可以自己对v2版进行解析 [version_manifest_v2.json](https://piston-meta.mojang.com/mc/game/version_manifest_v2.json) v2版新增了 sha1 和 complianceLevel 键值，具体作用可以自行 [MinecraftWiki](https://minecraft.fandom.com/wiki/Version_manifest.json)）可以看到一大长串的字符，不过不需要害怕，我们不需要一定看懂这些字符，只需要知道他是一个Json文件，并且我们只需要解析键值即可。下面我列出了一个经过我简化的Json文件，各位可以参照着用。
```json
{
    "latest": {
        "release": "1.19.3",
        "snapshot": "1.19.3"
    },
    "versions": [
        {
            "id": "1.19.3",
            "type": "release",
            "url": "https://piston-meta.mojang.com/v1/packages/6607feafdb2f96baad9314f207277730421a8e76/1.19.3.json",
            "time": "2022-12-07T08:58:43+00:00",
            "releaseTime": "2022-12-07T08:17:18+00:00"
        },
        {
            "id": "22w43a",
            "type": "snapshot",
            "url": "https://piston-meta.mojang.com/v1/packages/0fca847cf28d5caf18864ab7a31718094800abff/22w43a.json",
            "time": "2022-12-07T09:22:42+00:00",
            "releaseTime": "2022-10-26T11:55:59+00:00"
        },
        {
            "id": "1.18.2",
            "type": "release",
            "url": "https://piston-meta.mojang.com/v1/packages/23b6e0e0d0f87da36075a4290cd98df1a76e2415/1.18.2.json",
            "time": "2022-12-07T08:58:43+00:00",
            "releaseTime": "2022-12-05T13:21:34+00:00"
        }
    ]
}
```
上面就是我截取的json文件选段，怎么样，是不是看的清晰多了。我们现在一个个分析。在第一层下有 `"latest"` 和 `"versions"` 键， `"latest"` 下面主要存放的就是最新版本的版本号，其中 `"release"` 是指发行版最新版本号， `"snapshot"` 是指先行版发行的版本号。这个键值可以告诉启动器，究竟哪个版本是最新版本，让启动器实现下载最新版本的简化功能（防止有人无脑算法比版本大小，况且 `22w43a` 和 `1.19.3` 这样的版本号如果算法不够强劲多少会出错），其实你不加也可以，但是用户肯定不会满意。我们现在视线转移到最重要的 `"versions"` 键下，可以看到他以中括号开始，证明他是一个数组，数组里面存放的值就是每个独立的版本。我们单独取出一个值，比方说以 `versions[0]`（就是第一个值）来举例， `"id"` 键下面就是他的版本号，里面可以存放 "x.x.x" 这样的版本格式，也可以存放 "22w43a" 这样的格式， `"type"` 键值写了这个版本的类型，标有 "release" 说明他是一个正式版（发行版），标有 "snapshot" 说明他是一个测试版（快照版）。我们的启动器可以根据这个键值识别各个版本的类型，让用户下对他们想要的类型，一个大型发行版本下往往跟着诸多个快照版。 `"url"` 键值下包含了这个版本的索引文件地址，我们下载一个游戏，所需要的最根本的东西就是这个索引文件，人和下载都是建立在索引文件的基础上的，我们后续会继续讲解他的用法。 `"time"` 和 `"releaseTime"` 分别代表了这个版本的最后更新时间和发行时间，时间格式遵从 ISO 8601。
我们的启动器通过解析 `"version_manifest.json"` 列出了所有游戏版本，但当我们的用户点击下载的时候，我们的启动器又运行了什么呢。
启动器会先下载一个 `"版本号.json"` 文件（就是上文 `"url"` 键值下的那个文件），我后续说的各种文件的下载位置都是不固定的，包括这个，你可以把他下载到任何位置，只要你下次需要用的时候能解析到。不过为了通用，我们规定（或者官方启动器是这样做的）在启动器目录下新建一个 "./mincraft" 文件夹，文件夹里面有一个 "versions" 文件夹，用来装载游戏版本，至于 "versions" 文件夹里面需不需要再细分不同版本的不同文件夹要看用户需求，很多启动器提供的版本隔离，就是将不同的版本在 "versions" 文件夹下再新建一个以游戏名命名的文件夹，用来装载版本内容。在启动器下载完 `"版本号.json"` 文件，我们就可以通过解析这个本地文件来继续下载其他的需要用到的文件了。
我们任意打开一个 `"版本号.json"` 文件会看到以下内容。（下面的json是简化过的，并不是真正存在的，实际参数会比这个长很多，每个版本的json文件顺序都不同，但内容都是一样的）
```json
{
    "arguments": {
        "game": [
            "--username",
            "${auth_player_name}",
            "--version",
            "${version_name}",
            "--gameDir",
            "${game_directory}",
            "--assetsDir",
            "${assets_root}",
            "--assetIndex",
            "${assets_index_name}",
            "--uuid",
            "${auth_uuid}",
            "--accessToken",
            "${auth_access_token}",
            "--clientId",
            "${clientid}",
            "--xuid",
            "${auth_xuid}",
            "--userType",
            "${user_type}",
            "--versionType",
            "${version_type}",
            {
                "rules": [
                    {
                        "action": "allow",
                        "features": {
                            "is_demo_user": true
                        }
                    }
                ],
                "value": "--demo"
            },
            {
                "rules": [
                    {
                        "action": "allow",
                        "features": {
                            "has_custom_resolution": true
                        }
                    }
                ],
                "value": [
                    "--width",
                    "${resolution_width}",
                    "--height",
                    "${resolution_height}"
                ]
            }
        ],
        "jvm": [
            {
                "rules": [
                    {
                        "action": "allow",
                        "os": {
                            "name": "osx"
                        }
                    }
                ],
                "value": [
                    "-XstartOnFirstThread"
                ]
            },
            "-Djava.library.path=${natives_directory}",
            "-Dminecraft.launcher.brand=${launcher_name}",
            "-Dminecraft.launcher.version=${launcher_version}",
            "-cp",
            "${classpath}"
        ]
    },
    "assetIndex": {
        "id": "2",
        "sha1": "c492375ded5da34b646b8c5c0842a0028bc69cec",
        "size": 390746,
        "totalSize": 548701912,
        "url": "https://piston-meta.mojang.com/v1/packages/c492375ded5da34b646b8c5c0842a0028bc69cec/2.json"
    },
    "assets": "2",
    "complianceLevel": 1,
    "downloads": {
        "client": {
            "sha1": "977727ec9ab8b4631e5c12839f064092f17663f8",
            "size": 22704188,
            "url": "https://piston-data.mojang.com/v1/objects/977727ec9ab8b4631e5c12839f064092f17663f8/client.jar"
        },
        "client_mappings": {
            "sha1": "42366909cc612e76208d34bf1356f05a88e08a1d",
            "size": 7574170,
            "url": "https://piston-data.mojang.com/v1/objects/42366909cc612e76208d34bf1356f05a88e08a1d/client.txt"
        },
        "server": {
            "sha1": "c9df48efed58511cdd0213c56b9013a7b5c9ac1f",
            "size": 47162712,
            "url": "https://piston-data.mojang.com/v1/objects/c9df48efed58511cdd0213c56b9013a7b5c9ac1f/server.jar"
        },
        "server_mappings": {
            "sha1": "bc44f6dd84cd2f3ad8c0caad850eaca9e82067e3",
            "size": 5852819,
            "url": "https://piston-data.mojang.com/v1/objects/bc44f6dd84cd2f3ad8c0caad850eaca9e82067e3/server.txt"
        }
    },
    "id": "1.19.3",
    "javaVersion": {
        "component": "java-runtime-gamma",
        "majorVersion": 17
    },
    "libraries": [
        {
            "downloads": {
                "artifact": {
                    "path": "ca/weblite/java-objc-bridge/1.1/java-objc-bridge-1.1.jar",
                    "sha1": "1227f9e0666314f9de41477e3ec277e542ed7f7b",
                    "size": 1330045,
                    "url": "https://libraries.minecraft.net/ca/weblite/java-objc-bridge/1.1/java-objc-bridge-1.1.jar"
                }
            },
            "name": "ca.weblite:java-objc-bridge:1.1",
            "rules": [
                {
                    "action": "allow",
                    "os": {
                        "name": "osx"
                    }
                }
            ]
        },
        {
            "downloads": {
                "artifact": {
                    "path": "com/github/oshi/oshi-core/6.2.2/oshi-core-6.2.2.jar",
                    "sha1": "54f5efc19bca95d709d9a37d19ffcbba3d21c1a6",
                    "size": 947865,
                    "url": "https://libraries.minecraft.net/com/github/oshi/oshi-core/6.2.2/oshi-core-6.2.2.jar"
                }
            },
            "name": "com.github.oshi:oshi-core:6.2.2"
        }
    ],
    "logging": {
        "client": {
            "argument": "-Dlog4j.configurationFile=${path}",
            "file": {
                "id": "client-1.12.xml",
                "sha1": "bd65e7d2e3c237be76cfbef4c2405033d7f91521",
                "size": 888,
                "url": "https://launcher.mojang.com/v1/objects/bd65e7d2e3c237be76cfbef4c2405033d7f91521/client-1.12.xml"
            },
            "type": "log4j2-xml"
        }
    },
    "mainClass": "net.minecraft.client.main.Main",
    "minimumLauncherVersion": 21,
    "releaseTime": "2022-12-07T08:17:18+00:00",
    "time": "2022-12-07T08:17:18+00:00",
    "type": "release"
}
```
我们现在来看这几个关键值， `"arguments"` 代表了这个版本的游戏启动时需要用到的参数，键下面还包含了两个值，分别是 `"game"` ， `"jvm"` 分别代表了游戏所需要的参数和游戏虚拟机环境下所需要的参数。这两个键都是数组类型，数组的每个值都是单独的一个参数。我们先看 `"game"` 数组的 0 , 1 这两个索引位置的值，分别为 `"--username"` ，和 `"${auth_player_name}"` ，我们可以给他理解为一个传入用户名称的参数， `"--username"` 是参数名， `"${auth_player_name}"` 是值，其中用 `"${}"` 这样格式包被的是变量，也就是你可以自由替换这其中的内容，数组中的参数一个逗号就是一个空格，也就是我们上面这0,1索引的两个值变成启动参数会写成这样 `"--username XXX"` （XXX是用户名），在这个键下面我们还能发现 `"rules"` 键，这就是有条件的参数。比方说这段
```json
            {
                "rules": [
                    {
                        "action": "allow",
                        "features": {
                            "is_demo_user": true
                        }
                    }
                ],
                "value": "--demo"
            }
```
其中的 `"action"` 代表这个条件是否允许使用，值一般为 `"allow"` ， `"features"` 是指成立的条件，我们现在看到的这段写的是 `"is_demo_user"` 为 `"true"` 意思就是玩家为Demo玩家，如果这个条件为真，则向启动指令中加入 `"value"` 后面的值 （ `"--demo"` ）。
接下来我们来看 `"jvm"` 下面的值。 `"rules"` 同上，不过就是判断条件变成了系统判断。 `"osx"` 代表系统是 osx 系统 （苹果电脑的系统）, `"windows"` 代表是windows系统，`"linux"` 代表是linux系统。（这些条件都需要启动器自己解析自己判断）
接下来看 `"assetIndex"` 键，其中的 `"id"` 代表资源版本， `"url"` 代表资源索引文件位置，我们需要下载他并且储存在任意你可以调用到的位置（名字可以不改，也可以改成 游戏大版本号.json 比方说 1.19.json）。
`"assets"` 和 上面 `"assetIndex"` 中的 `"id"` 一样。
`"complianceLevel"` 代表这个版本是否为支持版本，1代表这个版本是支持版本，0代表不是，具体含义可以自行wiki。主要区分在mojang新加入的账户系统上面。
`"downloads"` 下面包含的就是我们游戏的本体，他包括客户端（`"client"`）和服务端（`"server"`）两部分，其中客户端或服务端不仅包含一个本体的下载地址，还包含一个Mapping，这是函数的映射表，可以自行理解，与我们这篇教程教的深度关联不大，但是如果你能理解映射表，相信你的启动器一定能锦上添花。
你完全可以给你的启动器加一个开服功能，这些都是你自己的选择，我们这里重点讲如何启动游戏，所以我们的目光要放在 `"client"` 键下面，这个键下面最重要的是 `"url"` 键，它的值包含了客户端jar文件的下载地址，我们现在要做的就是下载客户端jar文件，把他保存到一个你能调用到的位置（还是上面那句话目录你可以自己选，只要你自己能调用得到就行，但是如果作为一个分发软件，你至少要遵守业界公用的目录规范）。
`"id"` 键包含了这个游戏的版本号。
`"javaVersion"` 键下面有一个 `"majorVersion"` 比较重要，它规范了这个游戏版本启动所需的java版本，如果你不想让用户启动不了游戏的话最好给你的启动器加入这个版本判断，这样如果他们用低版本启动游戏你可以进行提示。
`"libraries"` 是较为重要，同时也是最麻烦的一个键。它包含了游戏所需的运行库名称，下载版本以及条件。我们现在截取一段一点点看。
```json
        {
            "downloads": {
                "artifact": {
                    "path": "ca/weblite/java-objc-bridge/1.1/java-objc-bridge-1.1.jar",
                    "sha1": "1227f9e0666314f9de41477e3ec277e542ed7f7b",
                    "size": 1330045,
                    "url": "https://libraries.minecraft.net/ca/weblite/java-objc-bridge/1.1/java-objc-bridge-1.1.jar"
                }
            },
            "name": "ca.weblite:java-objc-bridge:1.1",
            "rules": [
                {
                    "action": "allow",
                    "os": {
                        "name": "osx"
                    }
                }
            ]
        }
```
`"downloads"` 包含他的下载信息， `"artifact"`是文件信息，mojang这样设计json格式我并不明白为什么，可能下一阶段要在 `"downloads"` 下面加入其他的信息和 `"artifact"` 隔离开 ，`"path"` 记住这个值，我建议你按照他给你的位置进行存储，这样会减轻很多工作量（可以按照 `"libraries/${path}"`这样的格式储存），很重要，后续启动游戏也需要他。 `"url"` 包含了这个库的下载位置，我们要做的就是下载他，然后保存到你可以调用得到的位置。但使用他是有前提的，条件就是 `"rules"` 下面的，使用这个库需要用户系统为 `"osx"` 也就是苹果系统（如果一个集合下面不包含 `"rules"` 就代表他是通用的库，所有系统都需要下载他）。我这给的示例只列出了几段运行库，实际上要比这多得多，各位可以取其中的几个，然后写成程序让程序自动下载。
`"logging"` 键是给 log4j 报告生成使用的，我们只需要取其中的 `"argument"` 并把它加入参数就可以了。`"mainClass"` 是游戏的主类入口，后续我们会讲解用法。`"minimumLauncherVersion"` 是最小启动这个版本游戏的启动器版本，意义不明，好像是给官方启动器用的，我们不需要管他就可以了。剩下的键值意思我都解释过，不做过多赘述（所有我没标明意思的键值都代表和这篇教程无关，如果你想让启动器更加优秀你可以去wiki上面自行查找、研究）。
#### 游戏启动
当我们下载好游戏后，终于到了激动人心的启动时刻，上文我讲解了各个值的意思，接下来就该用他们了，将这些键值拼接起来，我们先来看一段启动命令。搭配着这些启动命令来对照着 版本.json 看，相信你们能更好理解。
```
G:\Minecraft\JDK17\bin\java.exe -Dminecraft.client.jar=.minecraft\versions\LauncherTest-Vanilla-1.19.2\LauncherTest-Vanilla-1.19.2.jar -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=16M -XX:-UseAdaptiveSizePolicy -XX:-OmitStackTraceInFastThrow -Xmn128m -Xmx4096m -Dfml.ignoreInvalidMinecraftCertificates=true -Dfml.ignorePatchDiscrepancies=true -XX:HeapDumpPath=MojangTricksIntelDriversForPerformance_javaw.exe_minecraft.exe.heapdump -Djava.library.path=G:\Minecraft\.minecraft\versions\LauncherTest-Vanilla-1.19.2\natives -Dminecraft.launcher.brand=test -Dminecraft.launcher.version=0.1.1 -cp G:\Minecraft\.minecraft\libraries\com\mojang\logging\1.0.0\logging-1.0.0.jar;G:\Minecraft\.minecraft\libraries\com\mojang\blocklist\1.0.10\blocklist-1.0.10.jar;G:\Minecraft\.minecraft\libraries\com\mojang\patchy\2.2.10\patchy-2.2.10.jar;G:\Minecraft\.minecraft\libraries\com\github\oshi\oshi-core\5.8.5\oshi-core-5.8.5.jar;G:\Minecraft\.minecraft\libraries\net\java\dev\jna\jna\5.10.0\jna-5.10.0.jar;G:\Minecraft\.minecraft\libraries\net\java\dev\jna\jna-platform\5.10.0\jna-platform-5.10.0.jar;G:\Minecraft\.minecraft\libraries\org\slf4j\slf4j-api\1.8.0-beta4\slf4j-api-1.8.0-beta4.jar;G:\Minecraft\.minecraft\libraries\org\apache\logging\log4j\log4j-slf4j18-impl\2.17.0\log4j-slf4j18-impl-2.17.0.jar;G:\Minecraft\.minecraft\libraries\com\ibm\icu\icu4j\70.1\icu4j-70.1.jar;G:\Minecraft\.minecraft\libraries\com\mojang\javabridge\1.2.24\javabridge-1.2.24.jar;G:\Minecraft\.minecraft\libraries\net\sf\jopt-simple\jopt-simple\5.0.4\jopt-simple-5.0.4.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-common\4.1.77.Final\netty-common-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-buffer\4.1.77.Final\netty-buffer-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-codec\4.1.77.Final\netty-codec-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-handler\4.1.77.Final\netty-handler-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-resolver\4.1.77.Final\netty-resolver-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-transport\4.1.77.Final\netty-transport-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-transport-native-unix-common\4.1.77.Final\netty-transport-native-unix-common-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\io\netty\netty-transport-classes-epoll\4.1.77.Final\netty-transport-classes-epoll-4.1.77.Final.jar;G:\Minecraft\.minecraft\libraries\com\google\guava\failureaccess\1.0.1\failureaccess-1.0.1.jar;G:\Minecraft\.minecraft\libraries\com\google\guava\guava\31.0.1-jre\guava-31.0.1-jre.jar;G:\Minecraft\.minecraft\libraries\org\apache\commons\commons-lang3\3.12.0\commons-lang3-3.12.0.jar;G:\Minecraft\.minecraft\libraries\commons-io\commons-io\2.11.0\commons-io-2.11.0.jar;G:\Minecraft\.minecraft\libraries\commons-codec\commons-codec\1.15\commons-codec-1.15.jar;G:\Minecraft\.minecraft\libraries\com\mojang\brigadier\1.0.18\brigadier-1.0.18.jar;G:\Minecraft\.minecraft\libraries\com\mojang\datafixerupper\5.0.28\datafixerupper-5.0.28.jar;G:\Minecraft\.minecraft\libraries\com\google\code\gson\gson\2.8.9\gson-2.8.9.jar;G:\Minecraft\.minecraft\libraries\com\mojang\authlib\3.11.49\authlib-3.11.49.jar;G:\Minecraft\.minecraft\libraries\org\apache\commons\commons-compress\1.21\commons-compress-1.21.jar;G:\Minecraft\.minecraft\libraries\org\apache\httpcomponents\httpclient\4.5.13\httpclient-4.5.13.jar;G:\Minecraft\.minecraft\libraries\commons-logging\commons-logging\1.2\commons-logging-1.2.jar;G:\Minecraft\.minecraft\libraries\org\apache\httpcomponents\httpcore\4.4.14\httpcore-4.4.14.jar;G:\Minecraft\.minecraft\libraries\it\unimi\dsi\fastutil\8.5.6\fastutil-8.5.6.jar;G:\Minecraft\.minecraft\libraries\org\apache\logging\log4j\log4j-api\2.17.0\log4j-api-2.17.0.jar;G:\Minecraft\.minecraft\libraries\org\apache\logging\log4j\log4j-core\2.17.0\log4j-core-2.17.0.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl\3.3.1\lwjgl-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl\3.3.1\lwjgl-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl\3.3.1\lwjgl-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-jemalloc\3.3.1\lwjgl-jemalloc-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-jemalloc\3.3.1\lwjgl-jemalloc-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-jemalloc\3.3.1\lwjgl-jemalloc-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-openal\3.3.1\lwjgl-openal-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-openal\3.3.1\lwjgl-openal-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-openal\3.3.1\lwjgl-openal-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-opengl\3.3.1\lwjgl-opengl-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-opengl\3.3.1\lwjgl-opengl-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-opengl\3.3.1\lwjgl-opengl-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-glfw\3.3.1\lwjgl-glfw-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-glfw\3.3.1\lwjgl-glfw-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-glfw\3.3.1\lwjgl-glfw-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-stb\3.3.1\lwjgl-stb-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-stb\3.3.1\lwjgl-stb-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-stb\3.3.1\lwjgl-stb-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-tinyfd\3.3.1\lwjgl-tinyfd-3.3.1.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-tinyfd\3.3.1\lwjgl-tinyfd-3.3.1-natives-windows.jar;G:\Minecraft\.minecraft\libraries\org\lwjgl\lwjgl-tinyfd\3.3.1\lwjgl-tinyfd-3.3.1-natives-windows-x86.jar;G:\Minecraft\.minecraft\libraries\com\mojang\text2speech\1.13.9\text2speech-1.13.9.jar;G:\Minecraft\.minecraft\libraries\com\mojang\text2speech\1.13.9\text2speech-1.13.9-natives-windows.jar;G:\Minecraft\.minecraft\versions\LauncherTest-Vanilla-1.19.2\LauncherTest-Vanilla-1.19.2.jar net.minecraft.client.main.Main --username test --version "test" --gameDir G:\Minecraft\.minecraft --assetsDir G:\Minecraft\.minecraft\assets --assetIndex 1.19 --uuid ${uuid} --accessToken ${accessToken} --clientId ${clientid} --xuid ${auth_xuid} --userType mojang --versionType "test" --width 854 --height 480
```
是不是很长，没关系，这个命令是靠程序自动生成的，并不会让你手写，不过我们还是要理解他。接下来我们一起一点点看他是怎么运作的。
`"G:\Minecraft\JDK17\bin\java.exe"` 代表了Java程序位置，你可以像这样的写法，如果电脑将java加入了系统环境变量，并且java版本正确能启动游戏，也可以直接写成 java 而不是这个绝对路径，但我还是劝你写出java程序所在的绝对路径，对后续版本管理很有用。
`"-Dminecraft.client.jar=${}"` 指定了游戏客户端的jar文件所在位置，也就是上文让你下载的 client.jar 位置。
`"-XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=16M -XX:-UseAdaptiveSizePolicy -XX:-OmitStackTraceInFastThrow"` 就是通过解析 版本.json 文件的 `"jvm"` 键获取到的参数了，jvm参数的意思以及GC调优各位可以自行百度。
`"-Xmn128m -Xmx4096m"` 是指这个jvm虚拟机的最小最大内存，你可以不加，也可以加上让用户自己选择分配内存。其中 "128","4096"就是内存大小，你只需要修改他们并且拼接字符串即可。
`"-Dfml.ignoreInvalidMinecraftCertificates=true -Dfml.ignorePatchDiscrepancies=true -XX:HeapDumpPath=MojangTricksIntelDriversForPerformance_javaw.exe_minecraft.exe.heapdump"`是通过解析 `"game"` 键下面的值而传入的游戏参数，注意，"-XX:HeapDumpPath"是jvm参数但是后面的值是通过解析 版本.json 文件获取到的，具体意思各位可以自行百度。
接下来就是重头戏了 `"-Djava.library.path="` 参数后面传入运行库位置（绝对路径）。
`"-Dminecraft.launcher.brand=test -Dminecraft.launcher.version=0.1.1"`分别传入启动器名称和启动器版本，根据你自己启动器的实况来，具体体现在游戏内 F3 状态面板的左下角。
最重要的 `"-cp"` 参数后面传入各个运行库位置（绝对路径，注意这里说的是各个，意思是你要把每个运行库jar文件传入，而不是大文件夹），每个运行库用 ";" 隔开。也就是上文解析 版本.json 文件并下载的运行库，到现在终于要用上了（注意上文说的 `"rules"` 中的条件，如果条件不正确加载不会成功，也就不会成功启动游戏），在所有运行库都传入完成后，后面跟着一个空格 `"net.minecraft.client.main.Main"` （这个值也是通过解析 版本.json 文件获取到的主类入口位置，如果mojang哪天抽风，可能主类入口就会改变，所以这个值是个变量，而不是常量）。
主类位置后面紧跟着就要传入游戏参数了（注意顺序，一定要先传入jvm参数，然后声明运行库和主类，最后传入游戏参数，不然会报错找不到参数）。这里的参数就是我们解析 版本.json 文件后获取到的 `"arguments"` 下面的 `"game"` 中的值。
`"--username test --version "test" --gameDir G:\Minecraft\.minecraft --assetsDir G:\Minecraft\.minecraft\assets --assetIndex 1.19 --uuid ${uuid} --accessToken ${accessToken} --clientId ${clientid} --xuid ${auth_xuid} --userType mojang --versionType "test""`分别传入了用户昵称、游戏版本（你可以自定义，也可以写正确的版本号，具体体现在游戏窗口标题上）、游戏目录（根据实际情况来，统一规范是要写在 .minecraft 下面），游戏资源目录（同上），游戏资源索引版本（每个大版本号是一个索引，储存在 assets/indexes 下面，同时也是你上文索引文件的下载地址）、uuid、accessToken、clientId、xuid都是正版登录要用到的，通过请求mojang正版验证服务器获取到、用户类型（微软或mojang）、版本类型。
`"--width 854 --height 480"` 就是上文说到的根据条件判断来加入的，他们代表了游戏窗口的大小。宽854，高480。
至此，游戏就可以正确启动了，如果你遇到了一些问题，请接着往下看。
#### 错误解决
游戏启动但是没有声音，可能是资源索引配置错误。
指令提示找不到参数"xxxxx"，检查启动指令顺序，游戏参数要传在主类后面。
### 一些小提示
如果你的针对用户是windows人群，你想要获取到电脑上的所有Java位置，然后通过 java -version 指令检查版本。你可以调用windows的索引功能并搜索java.exe，详情请见微软api文档。或者使用where java 指令（但是只能找到在C://Programe）下面的程序。
如果你想给游戏加入联机功能，你完全可以搭建一个基于frp的内网穿透服务器，然后将用户的服务器端口穿透出去（只是提议，但是要处理好安全问题）。
如果你想让用户能通过启动器下载到mods，你完全可以爬取curseforge上的内容（但是总感觉不厚道）。