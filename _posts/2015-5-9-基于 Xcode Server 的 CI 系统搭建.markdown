---
layout: post
category: CI
---

## Xcode Server based CI

`CI` 就不需要我来解释了，Wiki 在这儿[Continuous integration](http://en.wikipedia.org/wiki/Continuous_integration)。 其实我个人没有深刻的了解起作用，目前的对其还只是不知名的崇拜，因为每个企业都有 CI 系统并且有团队维护保证，对于原来公司，我印象最深可的就是 SIT(System Intergation Testing) 阶段，我们移动端的开发，进入此阶段后就不应该再做功能开发，而是需要在测试人员测试过程中，提供 bug 修复服务，代码版本控制系统也会切出一个专门的分支来做这个事情，大家都会将 SIT 上的改动提交到这个分支，而 SIT package 的打包工作也会变得越来越频繁，直到最终出包。

其实原公司有很多问题，比如基本没有单元测试和整合测试阶段，当然，没有也是因为开发的责任。在公司中比较重要的测试阶段就是系统测试和回归测试，基本上原公司最重视的阶段，但是这两个工程中，基本都是靠庞大的测试团队人工测试，而且大部分也只是界面测试，不同模块集成过程中的内部逻辑基本无法无法理论覆盖，如果内部逻辑有比较明显的外在表现还好说，一旦没有外在表现，就很容易隐藏很多问题。实际上针对 UI 界面的测试本来也是移动开发的一个难点，这个暂且不论，但是我个人觉得，公司还是可以再很多细节方面做得更好，提供良好可用的测试框架，培训 TDD 或者 BDD 测试思想和最佳实践，提供给开发人员良好的反馈机制反向激励开发人员对测试感兴趣。当然，原公司其实并非 严格的敏捷流程，对于代码的更改并没有培训时所说的那么热情 积极主动的拥抱，所以可能 CI 并不是真正的核心需求。

因为目前的情况是没有测试人员，所以需要自己来测试保证软件质量，而又因为自己想走敏捷开发，想拥抱改变，所以自然而然的就需要一套 CI 来保证软件质量，做到每一个提交都能够做到自动化分析测试打包并分发到 beta 测试用户手上。

## Server.app

 好吧，虽然我不愿意承认，但是我还是想说这套系统的核心其实不用我操心太多，因为有这么一个 Apple 提供的 APP 已经完成了绝大部分的功能，你需要的是付钱就好。
 这中间遇到的比较复杂的问题就是 cocoapods 的集成以及在解决问题的工程中对于 Server 的工作流程有一个大致的了解

### Cocoapods

因为以前使用过 github 的 travis CI，对于 CI 系统集成 Cocopods 的思路应该还是很简单的，其实就是 github 接受到推送的时候，会触发 travis CI 的事件，拉取代码，然后进入项目主目录，执行

     pod install

 安装完毕后，针对 `workspace` 文件通过 `xctool` 来执行编译测试的命令。
 因为 Xcode Server 没有官方支持 Cocoapods, 所以这条命令需要一个时机来执行，通过翻看 Cocoapods 社区有关的[讨论](https://github.com/CocoaPods/blog.cocoapods.org/issues/21)，其中还有大量的用户提出了宝贵的意见这里再放一个[总结帖](https://github.com/CocoaPods/blog.cocoapods.org/blob/master/_drafts/CocoaPods-Bots.markdown)，大致得到了以下的信息

 + 你需要在 `scheme` 中的 `pre-action` 中加入一个执行脚本，然后运行相关命令

 + 在 Server 中，实际上是一个名为 `teamsserver` 的用户在帮你做这些工作，你需要为其设置合适的用户权限

 看到这我想到了以下几个问题
 
 + PATH 的问题，因为真在工作的用户是一个 `teamserver` 用户，那他的 PATH 会不会和我目前不一样，通过 `brew` 安装的 bin 都放在了 `/usr/local/bin` 里，假如 `teamserver` 的 PATH 不包括的话，我想通过 `proxychains4` 来排除功夫网的干扰也成问题。
 + `scheme` 中的 `pre-action` 是什么鬼，实际上我在设置 `bot` 的时候我看到了一个更适合的名为 `trigger pre run script` 的入口

 好了，想完了， 先查了下 `scheme` 中的 `pre-action`，文档点击[这里](https://developer.apple.com/library/ios/recipes/xcode_help-scheme_editor/Articles/SchemeDialog.html)，很好理解，他有一个好处是可以使用 [Build Settings](https://developer.apple.com/library/mac/documentation/DeveloperTools/Reference/XcodeBuildSettingRef/1-Build_Setting_Reference/build_setting_ref.html#//apple_ref/doc/uid/TP40003931-CH3-SW38)，而 Xcode Server 提供的 `trigger pre run script` 不行，他只能用当前的环境变量

 为了安全起见，我在 `trigger pre run script` 先执行了一个 `set` 命令看看结果，如下:

    Apple_PubSub_Socket_Render=/private/tmp/com.apple.launchd.am5GKKgJiV/Render
    BASH=/bin/bash
    BASH_ARGC=()
    BASH_ARGV=()
    BASH_LINENO=([0]="0")
    BASH_SOURCE=([0]="/var/folders/q6/js_jd2h13bvd752sb4gp4r54000086/T/550619E3-120B-48BA-8907-B81FF93B4533-26190-00001AA45D72F3CE")
    BASH_VERSINFO=([0]="3" [1]="2" [2]="57" [3]="1" [4]="release" [5]="x86_64-apple-darwin14")
    BASH_VERSION='3.2.57(1)-release'
    DIRSTACK=()
    EUID=262
    GROUPS=()
    HOME=/var/_xcsbuildd
    HOSTNAME=david-macbook-pro.local
    HOSTTYPE=x86_64
    IFS=$' \t\n'
    LC_ALL=en_US.UTF-8
    LOGNAME=_xcsbuildd
    MACHTYPE=x86_64-apple-darwin14
    OPTERR=1
    OPTIND=1
    OSTYPE=darwin14
    PATH=/Applications/Xcode.app/Contents/Developer/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin
    PIPESTATUS=([0]="0")
    PPID=26190
    PS4='+ '
    PWD=/Library/Developer/XcodeServer/Integrations/Caches/816770395ec7710c2659022ce303f039/Source
    SHELL=/bin/false
    SHELLOPTS=braceexpand:hashall:interactive-comments
    SHLVL=1
    SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.0xRJk8052W/Listeners
    TERM=dumb
    TMPDIR=/var/folders/q6/js_jd2h13bvd752sb4gp4r54000086/T/
    UID=262
    USER=_xcsbuildd
    XCS=1
    XCS_BOT_ID=816770395ec7710c2659022ce303f039
    XCS_BOT_NAME='MetaReader Bot'
    XCS_BOT_TINY_ID=096E9B8
    XCS_INTEGRATION_ID=816770395ec7710c2659022ce30c1c55
    XCS_INTEGRATION_NUMBER=11
    XCS_INTEGRATION_RESULT=unknown
    XCS_INTEGRATION_TINY_ID=938EE36
    XCS_OUTPUT_DIR=/Library/Developer/XcodeServer/Integrations/Integration-816770395ec7710c2659022ce30c1c55
    XCS_SOURCE_DIR=/Library/Developer/XcodeServer/Integrations/Caches/816770395ec7710c2659022ce303f039/Source
    XPC_FLAGS=0x0
    XPC_SERVICE_NAME=com.apple.xcsbuildd
    _=LC_ALL
    __CF_USER_TEXT_ENCODING=0x106:0x0:0x0
    /Library/Developer/XcodeServer/Integrations/Caches/816770395ec7710c2659022ce303f039/Source

 好了，当我看到 `USER` 和 `HOME` 的参数的时候，我发现当前运行的用户不是 `teamsserver` 而是 `_xcsbuildd`。再看 `PATH` 果然没有我想要的 `/usr/local/bin` 目录，再看 `PWD` 发现这个脚本执行所在的目录并非项目根目录，而是所有源代码存放的目录，也就是说想执行 `pod install` 还需要 `cd` 相应目录才可以。

 先跑一次验证一下失败结果:

    Bot Issue: error. Shell Script Invocation Error.
    Issue: Diff: /../Podfile.lock: No such file or directory.
    
    Bot Issue: error. Shell Script Invocation Error.
    Issue: Diff: /Manifest.lock: No such file or directory.

    Bot Issue: error. Shell Script Invocation Error.
    Issue: The sandbox is not in sync with the Podfile.lock. Run 'pod install' or update your CocoaPods installation..

然后带着此前的结论修改之后脚本之后。再跑一次，通过！
