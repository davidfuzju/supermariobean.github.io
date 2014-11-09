---
layout: post
category: iOS
---

##问题由来##

在CTVoip项目中，生产环境中的sip服务器被强制只支持amr语音编解码，也就是说sip客户端在进行语音的编解码时只能通过amr编解码方式来完成，才能成功实现语音通讯。

+ iOS sdk4.0以后就不再支持这种格式的文件，所以需要使用第三方的编解码库来弥补缺失

+ 查询后得到找到一个第三方的开源amr编解码库，opencore-amr。其官方主页点击[这里](http://sourceforge.net/projects/opencore-amr/files/?source=navbar)

##库信息##

+ pjpeoject

	版本

		pjproject-2.1.0

	内容

	pjproject 提供的开发包需要通过编译来产生相应的静态库
	主要产生以下几个库文件夹：

		pjlib
		pjlib-util
		pjmedia
		pjnath
		pjsip
		third_party

	其中third_party中包含了一些语音编解码库(g7221,gsm,ilbc,speex)、网络鉴权算法库(milenage)、音频采样库(resample)、安全实时传输协议库(srtp)。

	具体有关milenage的介绍点击[这里](x-devonthink-item://3AEFDDC9-2025-4FDA-9F24-90526DCBCC6D)

	**实际上，一般情况下，对于iOS版本进行pjproject编译的时候，是默认使用g7221和ilbc等语音编码库。**

	**其他的库一般是必须的，而语音编码库则根据实际需要更改，这里的话，只需要opencore-amr的语音编解码，其他都应该disable**

+ opencore-amr

	版本

		opencore-amr-0.1.3

	内容

	包含了amrnb的enc和dec以及amrwb的dec，但是不包含enc，amrwb的enc部分被单独提出来作为一个独立的库名为

		vo-amrwbenc

+ vo-amrwbenc

	版本

		vo-amrwbenc-0.1.2

	内容

	包含了amrwb的enc部分。

##问题解决过程##

###编译得到opencore-amr for iphone的库###

***实际上，一般库的iphone版本都是基于arm cpu架构上的，但实际上为了某些情况下可以在模拟器上调试，也需要i386 cpu架构的库版本***

+ 无论如何，你首先需要知道你的Developer路径，这对于iOS开发者来说非常重要，很多交叉编译器都在这个目录之下

		DEVELOPER="/Applications/Xcode.app/Contents/Developer"
		SDKVERSION="blablabla"
		DEST="blablabla"

	另外，现在常见的iOS设备的arm cpu类型主要还是armv7和armv7s，所以在家i386的cpu类型，一共是三中需要讨论的情况。

+ 当arch = "armv7" || "armv7s"时

		PLATFORM="iPhoneOS"
		PATH="${DEVELOPER}/Platforms/${PLATFORM}.platform/Developer/usr/bin:$PATH"
		SDK="${DEVELOPER}/Platforms/${PLATFORM}.platform/Developer/SDKs/${PLATFORM}${SDKVERSION}.sdk"
		CC="gcc -arch $arch --sysroot=$SDK" CXX="g++ -arch $arch --sysroot=$SDK" LDFLAGS="-Wl,-syslibroot,$SDK" ./configure --host=arm-apple-darwin --prefix=$DEST/$arch --disable-shared

	armv7和armv7s的编译是差不多的。**至少我现在还没遇到不一样的情况，可能有。**下面着重介绍一下我在这个编译阶段遇到的问题。

	+ PATH

		将iPhoneOS.platform中的/usr/bin加入环境变量中，是为了在make阶段使用合乎-arch参数的交叉编译器，在这里应该就是在新增加到PATH变量中的那个地址中的arm版的gcc

			arm-apple-darwin10-llvm-g++-4.2
			arm-apple-darwin10-llvm-gcc-4.2

		只有使用正确版本的交叉编译器，才能编译出合乎cpu架构的库

	+ SDK

			选择正确的版本就可以了。

	+ ./configure

		在之前我曾误认为将CC，CXX，LDFLAGS与./configure分行是可行，但是后来事实证明这是非常错误的。这是一条命令，CC，CXX，LDFLAGS中的参数应该与./configure一同传入。

		CC:用来指定c编译器

		CXX:用来指定cxx编译器

		**这两个参数中都指定了--sysroot为SDK的路径，指向系统头文件**

		LDFLAGS:则指定了系统库文件

		--host=arm-apple-darwin:指定了目标机器，而make也会根据这个参数来判断PATH中所提供的那个交叉编译器是否是所需要的编译器

 		--prefix=$DEST/$arch:产生的库文件和头文件的目标路径
 		--disable-shared:**可能与生成静态库或者动态库有关**

+ 当arch = "i386"时

		PLATFORM="iPhoneSimulator"
    	PATH="${DEVELOPER}/Platforms/${PLATFORM}.platform/Developer/usr/bin:$PATH"
    	SDK="${DEVELOPER}/Platforms/${PLATFORM}.platform/Developer/SDKs/${PLATFORM}${SDKVERSION}.sdk"
    	CC="gcc -arch $arch" CXX="g++ -arch $arch" ./configure --prefix=$DEST/$arch --disable-shared

	i386的情况相对要简单一些，无需指定--sysroot和--host参数

+ 最后进行并库安装和清理

    make
    make install  
    make clean

	通过上述步骤就可以获得以上提到的opencore-amr for iphone 以及 vo-amrwbenc for iphone的库和头文件

有关linux套件安装过程中configure,make,make install的作用可以点击[这里](x-devonthink-item://B60570FF-10A9-45D8-B663-59C6E843650E)

###编译获得pjproject for iphone的库###

+ 基础编译方法（不含将opencore-amr相关库编译进入pjproject的方法）

	实际上pjproject的给出的说明非常的清晰，这里不再累述，但是一些区别需要澄清

	+ armv7*

			echo "<======== Building PjProject for iPhone $arch"  
			ARCH="-arch $arch" \
			./configure-iphone

	+ i386

			echo "<======== Building PjProject for iPhoneSimulator $arch"
			DEVPATH="/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer"
			IPHONESDK="iPhoneSimulator6.1.sdk"
			CC="/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/usr/bin/gcc-4.2" \
			CFLAGS="-O2 -m32 -miphoneos-version-min=4.0 -g -ggdb -g3 -DNDEBUG" \
			LDFLAGS="-O2 -m32" \
			./configure-iphone

	两者差异很大，和之前所说到的编译过程不一样，关键的一点事arm版本的没有PATH的声明，难道就不需要寻找对应的cpu版本的交叉编译器了么？实际上这与你具体执行的configure文件有关，你可以发现，在此处，arm版本中出现了configure-iphone而i386出现了，这就是问题的关键所在，所以我们查阅一下代码，实际也确实是这样的，这个configure-iphone在执行最终的aconfigure之前进行了诸多默认判断，已经获取了这些参数设定。

	但是i386就没有那，虽然也是用了configure-iphone，但是configure-iphone的代码中似乎没有提供platform为iphonesimulator版本的路径，也就是这个configure-iphone文件只考虑了arm版本的默认情况，当你通过ARCH参数输入i386是，是没有默认的devpath，随意这就造成了后面需要手动设定路径。

	**另外可以发现，i386中并没有使用PATH环境变量的方法来获取正确版本的gcc，而是直接通过绝对路径告诉configure-iphone用这个版本的gcc**

+ 带有opencore-amr库的编译方法

	经过千辛万苦，终于将opencore-amrnb for iphone成功编译进入pjsip for iphone。

	大致说一下过程。其实说起来很简单，就是将编译好的opencore-amrnb for iphone在pjsip for iphone的编译阶段连接进来。

	+ 这里要先说明的是一个网上的官方解答

		实际上，pjsip的官方网站上给出了支持opencore-amr的平台，主要由两个，Makefile based (Linux, MacOS X, etc.)和Windows，其他平台暂不支持。点击[这里](http://trac.pjsip.org/repos/ticket/1388)查看详细。

		在mac环境下，首先安装opencore-amr，这就会在目录下

			/usr/local/lib
			/usr/local/include

		下生成相应的静态库文件和头文件。此后重新

			./configure

		pjproject，就会在原本的Log中显示disable的opencore-amr的地方显示检查opencore的安装。

		实际上，在此过程中，gcc通过寻找默认的库和头文件目录是，获知libopencore-amrnb静态库的存在，同时获得头文件，在编译阶段将其连接进入pjproject库。

		而在编译arm版本也就是iphone版本的时候，就发生了问题。

	基于上面的基础编译方法，在研究了文件aconfigure文件后，发现与opencore-amr有关的configure参数

		--with-opencore-amr=$DEST/$arch \
 		--with-opencore-amrwbenc=$DEST/$arch \
		--disable-g7221-codec \
		--disable-speex-codec \
		--disable-gsm-codec \
		--disable-ilbc-codec \

	后面的参数主要是disable掉了很多不需要的语音编解码。

	**在disable语音编解码的过程中，有这么一个冲突，就是在pjproject的源码中,目录**

		pjlib/include/pj/config_site.h

	**的文件中有专门为iphone所设置的一系列参数。在configure阶段，会有响应的宏定义被直接插入到这个文件中来，所以在disable掉一些编码的同时，是需要将这个头文件中的一些宏定义给注释掉，以免产生redefine的错误**

	前两个为pjproject支持opencore-amrnb,opencore-amrwb两种语音编解码提供了参数选项，路径就是此前我们生成的for iphone的库和头文件所存在单位。

	**在实际的编译过程中，如果仅仅有opencore-amr的库和头文件存在，只能使pjproject支持opencore-amrnb，必须在同时提供opencore-amr和vo-amrwbenc的时候，才能使pjproject支持opencore-amrwb**

	**最初开始安装amrwb编解码的时候是因为发生了生产环境中发送dtmf时发生了偶尔无法响应的情况，在无法确定服务端问题的情况下，我自己猜测是因为wb主管宽频语音信号编解码，而nb主管窄频语音信号编解码，而dtmf是通过宽频信号和窄频信号的组合来实现字符发送的。基于此我才安装wb，但是结果表明，是我乱想了**

	此外，再设置具体的头文件和库文件的位置就可以了（实际上感觉已经不需要了，很可能这个参数就已经包含了这个功能）

		CFLAGS="-O2 -Wno-unused-label -I $DEST/$arch/include:$DEST/$arch/include" \
		LDFLAGS="-L $DEST/$arch/lib:$DEST/$arch/lib" \

	至此，就可以重新编译pjproject，并且提供对opencore-amrnb和amrwb的支持了。

	该问题的解决最大归功于这一篇问答，可以查看点击[这里](x-devonthink-item://25F41272-DFCE-4E3E-AF80-C4DFA1FF016E)


##后续

***对于iphone simulator也就是对于cpu架构为i386编译方法中，需要注意如下内容***

其中

	CC="/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/usr/bin/gcc-4.2"

应该修正为

	CC="/Applications/Xcode.app/Contents/Developer/usr/bin/gcc"

就可以通过编译。这里就不知道水果公司到底是什么用意了。
