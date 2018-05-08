---
title: iOS crash 解析定位，shell脚本查找crash
date: 2016-06-25
updated: 2016-06-25
---

iOS开发中，对于线上版本或公测版本产生的crash，我们可以通过结合.app ，.dSYM 及 crash log 三个文件来进行解析定位。

***最新更新:***

**最近对查找线上Crash做了整理，写成[CrashScript.sh](https://github.com/lele8446/CrashScript/tree/master)，详情见下面**`查找Crash脚本`

<!-- more -->

***

* 获取iOS设备上的 crash log

 1. 将iOS设备连接到电脑上，打开 Xcode -> Organizer -> Devices,找到该台设备，在 Device logs 中找到 crash log（后缀为 .crash 的 log 文件），将其导出即可。

 2. 如果你的应用已经上架App Store，那么开发者可以通过iTunes Connect（Manage Your Applications - View Details - Crash Reports）获取用户的crash log。不过这并不是100%有效的，而且大多数开发者并不依赖于此，因为这需要用户设备同意上传相关信息，详情可参见iOS: Providing Apple with diagnostics and usage information摘要。

 3. 第三方crash收集系统，甚至还带了符号化crash日志的功能。比较常用的有[Crashlytics](http://www.cnblogs.com/www.crashlytics.com)，[Flurry](http://www.flurry.com/)等。
 


* 确保.app  .dSYM和crash log的uuid相同

	以上三者的uuid必须都一样才能进行解析，查看三个文件的 uuid ：

 1. 查看xx.app文件的uuid的方法,在终端输入：
 > $ dwarfdump --uuid xxx.app/xxx (xxx工程名)

 2. 查看xx.app.dSYM文件的uuid的方法，在命令行输入：
 >$ dwarfdump --uuid xxx.app.dSYM (xxx工程名)

 3. 查看 crash log 文件的 uuid的方法：<br>
 在 crash log 文件中，找到 Binary Images: 项目名后面第一个尖括号中的一串码就是改 crash log 文件的 uuid。
如下，70464c7fc4df37f38f81eeaf88a0713d就是uuid。
>Binary Images:
0x10000c000 - 0x100cf7fff xxx (xxx工程名) arm64  <70464c7fc4df37f38f81eeaf88a0713d>

* 显示.dYSM包内容，进入/Contents/Resources/DWARF路径下，执行命令：
> atos -arch armv7 -o XXX(项目名称) 0x17D580(16进制crash奔溃地址)

 显示结果，可以看到是PCBabyMyGroupReplyTVCell类第66行出错。
 
      	-[PCBabyMyGroupReplyTVCell setCellInfo:] (in BabyBook2) (PCBabyMyGroupReplyTVCell.m:66)
      
	如果定位不对，可能涉及到地址偏移的计算。首先查看起始地址，即使每次iOS app启动都会加载(main module)主模块在不同的内存地址，但是dSYM文件总是假设你的main module加载在地址0x100000000(64位)，或者0x4000(32位)。
	
	查看偏移，显示.dYSM包内容，进入/Contents/Resources/DWARF路径下，执行命令：
	
	> otool -arch arm64 -l XXX(XXX为项目名) | grep -B 1 -A 10 "LC_SEGM" | grep -B 3 -A 8 "__TEXT"

 显示结果：
 
		      Load command 1
		                cmd LC_SEGMENT_64
		            cmdsize 1032
		            segname __TEXT
		             vmaddr 0x0000000100000000
		             vmsize 0x0000000000620000
		            fileoff 0
		           filesize 6422528
		            maxprot 0x00000005
		           initprot 0x00000005
		             nsects 12
		              flags 0x0
看到vmaddr显示为0x0000000100000000，所以偏移地址计算为：0x17D580＋0x0000000100000000＝0x10017D580。
再次执行终端命令：
> atos -arch armv7 -o XXX(项目名称) 0x10017D580

	附：32位及64位设备的执行命令
	
	32位： xcrun atos -arch armv7 -o xxx(应用名称)  xxx(偏移地址)<br>
	64位： xcrun atos -arch arm64 -o xxx(应用名称)  xxx(偏移地址)
	
***
### 查找Crash脚本
***注意！！!Shell脚本默认无法处理带空格的文件路径，请确保`.xcarchive` 文件以及`CrashScript.sh`脚本所在路径名不存在空格。（Xcode Archive生成的`.xcarchive`文件名默认带空格，请去除）***

获取打包生成的 `.xcarchive` 文件，将其和`CrashScript.sh`脚本放到同一级目录下，运行脚本:

    ./CrashScript.sh [-u] [-t <Device type>] -a <Code address> 
参数说明：`[]表示可选参数，<>表示必填参数`

    -u 				         是否查看UUID
    -t <Device type>			发生crash的设备类型，有两种值：32和64，默认64
    -a <Code address>	       10进制的出错地址
脚本完整[代码](https://github.com/lele8446/CrashScript/tree/master)

    #!/bin/bash

	#--------------------------------------------------------------------------------
	# 脚本说明：
	#
	# 1、实现功能：
	#     1）、查看 .xcarchive 文件的UUID
	#     2）、查看10进制的出错地址对应的代码
	#
	# 2、使用方式：
	#     1）、将CrashScript.sh脚本和 .xcarchive 文件放到同一级文件夹下
	#     2）、运行脚本：
	#            ./CrashScript.sh [-u] [-t <Device type>] -a <Code address> 
	#         参数说明：
	#                 -u 				     是否查看UUID
	#                 -t <Device type>		发生crash的设备类型，有两种值：32和64，默认64
	#                 -a <Code address>	   10进制的出错地址
	#--------------------------------------------------------------------------------

	# 脚本文件所在根目录
	Release_path=$(pwd)
	xcarchive_path=""
	cd ${Release_path}
	Valid_dic=false
	for i in `ls`;
	do 
		#获取文件后缀名
		extension=${i##*.}
		if [[ ${extension} == "xcarchive" ]]; then
			Valid_dic=true
			xcarchive_path="${Release_path}/${i}"
		fi
	done

	if [[ ${Valid_dic} == false ]]; then
		echo -e "\033[31mCrashScript.sh脚本所在路径不存在.xcarchive文件，请检查！！\033[0m"
		exit 2
	fi

	Check_UUID="NO"
	Device_type="64"
	Have_code_address="NO"
	Code_address=""

	# 参数处理
	param_pattern=":ut:a:"
	OPTIND=1
	while getopts $param_pattern optname
	  do
	    case "$optname" in
		  "u")        
			Check_UUID="YES"	
	        ;;
	      "t")
			tmp_optind=$OPTIND
			tmp_optname=$optname
			tmp_optarg=$OPTARG

			OPTIND=$OPTIND-1
			if getopts $param_pattern optname ;then
				echo  -e "\033[31m选项参数错误 $tmp_optname\033[0m"
				exit 2
			fi
			OPTIND=$tmp_optind
			Device_type=$tmp_optarg
			if [[ ${Device_type} != "32" && ${Device_type} != "64" ]]; then
				echo  -e "\033[31m选项$tmp_optname 参数错误 $Device_type\033[0m"
				exit 2
			fi
	        ;;
	      "a")        
			tmp_optind=$OPTIND
			tmp_optname=$optname
			tmp_optarg=$OPTARG
			Have_code_address="YES"

			OPTIND=$OPTIND-1
			if getopts $param_pattern optname ;then
				echo  -e "\033[31m选项参数错误 $tmp_optname\033[0m"
				exit 2
			fi
			OPTIND=$tmp_optind
			Code_address=$tmp_optarg
	        ;;
	      "?")
	        echo -e "\033[31m选项错误: $OPTARG\033[0m"
			exit 2
	        ;;
	      ":")
	        echo -e "\033[31m选项 $OPTARG 必须带参数\033[0m"
			exit 2
	        ;;
	      *)
	        echo -e "\033[31m参数错误\033[0m"
			exit 2
	        ;;
	    esac
	  done

	dSYMs_path="${xcarchive_path}/dSYMs"
	dSYM_file=""
	cd ${dSYMs_path}

	# if [ ! -d "${dSYMs_path}" ];then
	if [ $? != 0 ]; then
		echo -e "\033[31m*************  dSYMs路径不存在  **************\033[0m"
		echo -e "\033[31mdSYMs_path = ${dSYMs_path}\033[0m"
		exit 2
	fi

	for file in `ls`;
	do 
		#获取文件后缀名
		extension=${file##*.}
		if [[ ${extension} == "dSYM" ]]; then
			dSYM_file=${file}
		fi
	done

	if [[ ${Check_UUID} == "YES" ]]; then
		echo -e "\033[32m*************** 获取  UUID ***************\033[0m"
		echo -e "\033[36mdSYM文件 : ${dSYM_file}\033[0m";
		echo ''
		# 获取UUID
		dwarfdump --uuid ${dSYM_file}
		echo ''

		if [ $? != 0 ]; then
		echo -e "\033[31m*************  获取UUID出错  **************\033[0m"
		exit 2
	fi
	fi

	if [[ ${Have_code_address} == "NO" ]]; then
		echo -e "\033[31m请输入出错的10进制代码地址！！\033[0m"
		exit 2
	fi

	DWARF_path="${dSYMs_path}/${dSYM_file}/Contents/Resources/DWARF"
	echo -e "\033[32m*************** 获取  DWARF ***************\033[0m"
	# echo -e "\033[36mDWARF_path : ${DWARF_path}\033[0m";

	cd ${DWARF_path}
	for file in `ls`;
	do
		echo -e "\033[36mDWARF文件 : ${file}\033[0m"

		echo ''
		echo -e "\033[32m*************** 获取出错代码 ***************\033[0m"

		if [[ ${Device_type} == "32" ]]; then
			# 搜索地址＝偏移地址转换为16进制后加0x4000;
			searchAddress="0x"$(echo "ibase=10;obase=16;${Code_address}+16384"|bc);
			xcrun atos -arch armv7 -o ${file} ${searchAddress}
		elif [[ ${Device_type} == "64" ]]; then
			#搜索地址＝偏移地址转换为16进制后，64位系统加100000000
			searchAddress="0x"$(echo "ibase=10;obase=16;${Code_address}+4294967296"|bc);
			xcrun atos -arch arm64 -o ${file} ${searchAddress}
		fi
		
	done