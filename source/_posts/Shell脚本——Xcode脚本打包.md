---
title: Shell脚本——Xcode脚本打包
date: 2016-06-20
updated: 2016-06-21
---


最近有好一段时间没有整理博客了，也不是因为忙，就是太懒。或者说程序员都自带懒属性😅，表现在对于一些机械性重复性的工作，宁愿一次性多花些精力去封装、去写出各种各样的工具来替自己干活，也不愿意多次重复劳动（就算那是很简单的工作）。这不，我结合最近的工作，研究着写了两份Shell脚本，一是关于[Xcode脚本打包](https://github.com/lele8446/ReleaseDir)，还有一个是关于[crash log解析](http://lele8446.top/2016/06/25/Shell脚本查找iOS线上Crash/)（这个将在下一篇文章说明）。

<!-- more -->

****
### Shell打包脚本实现了如下功能
1. 指定打包项目的Build号，Version版本号——Version号可选择自增或自定义（在测试阶段打包时，每次手动改变Version号很是麻烦，而且不方便管理；而如果能打包时通过脚本参数控制，so easy～）
2. 导出xcarchive文件，通过xcarchive文件得到dSYM文件，这是很重要的。查找线上crash时，缺它不可！[crash log解析](http://www.jianshu.com/p/392d801062d1)请看这里。
3. 打包生成ipa文件，这个就是最终需要的渠道包了。

另外打包脚本可以根据项目配置，每一次成功打包后，都会生成对应目录，并生成记录本次打包的log日志文件。
***
### 打包脚本

    #!/bin/bash

    #--------------------------------------------------------------------------------
	# 脚本说明：
	# 1、实现功能：
	#     1）、指定打包项目的Build号，Version版本号（Version号可选择自增或自定义）
	#     2）、导出xcarchive文件
	#     3）、打包生成ipa文件
	# 2、使用方式：
	#     1）、将ReleaseDir文件夹，放到跟所要打包的项目的根目录（TestDemo）同级别的目录下
	#     2）、cd至ReleaseDir，运行脚本./Release.sh TestDemo，并选择输入相关参数，开始打包
	#     3）、完成打包后，生成的目标文件在如下目录：
	#          /ReleaseDir/ArchivePath/TestDemo/1.0.0/2017_03_03_10:37:34
	#         (/ReleaseDir/ArchivePath/打包项目名称/打包版本号/打包时间)
	#--------------------------------------------------------------------------------
	#
	#--------------------------------------------------------------------------------
	# 打包project
	#        ./Release.sh  <Project directory name> [-s <Name>] [-e] [-d] [-a] [-b <Build number>]
	# 打包workspace
	#        ./Release.sh  <Project directory name> [-w] [-s <Name>] [-e] [-d] [-b <Build number>] [-v <Version number>]
	# 参数说明：
	#     <Project directory name>    第一个参数：所要打包的项目的根目录文件夹名称         
	#     -w 				         workspace打包，不传默认为project打包
	#     -s <Name>			       对应workspace下需要编译的scheme（不传默认取xcodeproj根目录文件名）
	#     -e 	        	         打包前是否先编译工程（不传默认不编译）
	#     -d 		                 工程的configuration为 Debug 模式，不传默认为Release
	#     -a 			             打包，Version版本号自动＋1（针对多次打测试包时的版本号修改）
	#	  -b <Build Num>              Build版本号，指定项目Build号
	#	  -v <Version Num>            Version版本号，指定项目Version号
	#  注意，参数-a 与 -v 互斥，只能选择传其中一种参数！！
	#--------------------------------------------------------------------------------
	#
	#--------------------------------------------------------------------------------
	# ReleaseDir文件目录说明：
	#
	# |___TestDemo                                所要打包的项目的根目录
	# |___ReleaseDir                              打包相关资源的根目录
	#    |___Release.sh                           Release.sh打包脚本
	#    |___ExportOptions.plist                  -exportOptionsPlist 配置文件
	#    |___ArchivePath                          打包文件输出路径
	#       |___TestDemo                          打包项目名称
	#          |___1.0.0                          打包版本号
	#             |___2017_03_02_16/23/28         打包时间（格式：年_月_日_时/分/秒）
	#                |___TestDemo_1.0.0.xcarchive 导出的.xcarchive文件
	#                |___TestDemo.ipa             导出的.ipa文件
	#                |___LogPath                  打包日志
	#
	#--------------------------------------------------------------------------------

    #计时
    SECONDS=0
    #--------------------------------------------
    # 参数判断
    #--------------------------------------------
    # 如果参数个数少于1
    if [ $# -lt 1 ];then
	    echo -e  "\033[31m第一个参数请输入 Xcode project 文件所在的根目录文件夹名称！！\033[0m"
	    exit 2
    fi

    # 脚本文件所在根目录
    Release_path=$(pwd)
    Project_dir=$1
    Project_path=""

    if [ "`pwd`" != "/" ]; then
	    cd ..
	    Root_path=$(pwd)
	    Project_path="${Root_path}/${Project_dir}"
    else
	    echo -e "\033[31m脚本路径错误\033[0m"
	    exit 1;
    fi

    if [ ! -d "${Project_path}" ];then
	    echo -e "\033[31m首参数必须是有效的文件夹名称！！\033[0m"
	    exit 2
    fi

    #Xcode project 文件路径
    cd ${Project_path}
    Valid_dic=false
    for i in `ls`;
    do 
	    #获取文件后缀名
	    extension=${i##*.}
	    if [[ ${extension} == "xcodeproj" ]]; then
		    Valid_dic=true
	    elif [[ ${extension} == "xcworkspace" ]]; then
		    Valid_dic=true
	    fi
    done

    if [[ ${Valid_dic} == false ]]; then
	    echo -e "\033[31m请输入包含xcodeproj或xcworkspace的文件夹名称！！\033[0m"
	    exit 2
    fi

    Archive_type="project"
    Scheme_name="default"
    Edit="NO"
    Configuration="Release"
    Auto_increment_version_num="NO"
    Build_str="custom"
    Version_str="custom"

    # 参数处理
    param_pattern=":ws:edab:v:"
    OPTIND=2
    while getopts $param_pattern optname
      do
        case "$optname" in
	      "w")        
		    Archive_type="workspace"	
            ;;
          "s")
		    tmp_optind=$OPTIND
		    tmp_optname=$optname
		    tmp_optarg=$OPTARG

		    OPTIND=$OPTIND-1
		    if getopts $param_pattern optname ;then
			    echo  -e "\033[31m选项参数错误 $tmp_optname\033[0m"
			    exit 2
		    fi
		    OPTIND=$tmp_optind
		    Scheme_name=$tmp_optarg
            ;;
          "e")        
		    Edit="YES"	
            ;;     
	      "d")        
		    Configuration="Debug"	
            ;;
          "a")
		    Auto_increment_version_num="YES"
		    ;;
          "b")
		    tmp_optind=$OPTIND
		    tmp_optname=$optname
		    tmp_optarg=$OPTARG

		    OPTIND=$OPTIND-1
		    if getopts $param_pattern optname ;then
			    echo  -e "\033[31m选项参数错误 $tmp_optname\033[0m"
			    exit 2
		    fi
		    OPTIND=$tmp_optind
		    Build_str=$tmp_optarg
            ;;
          "v")        
		    tmp_optind=$OPTIND
		    tmp_optname=$optname
		    tmp_optarg=$OPTARG

		    OPTIND=$OPTIND-1
		    if getopts $param_pattern optname ;then
			    echo  -e "\033[31m选项参数错误 $tmp_optname\033[0m"
			    exit 2
		    fi
		    OPTIND=$tmp_optind
		    Version_str=$tmp_optarg
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

    # echo "Project_path = ${Project_path}"
    #app文件名称
    appname=$(basename ./*.xcodeproj)
    #通过app文件名获得工程target名字
    target_name=$(echo $appname | awk -F. '{print $1}')
    #app文件中Info.plist文件路径
    App_infoplist_path=${Project_path}/${target_name}/Info.plist
    # 获取version值
    Version_initial=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" ${App_infoplist_path})
    # 获取build值
    Build_initial=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" ${App_infoplist_path})

    if [[ ${Scheme_name} == "default" ]]; then
	    Scheme_name=${target_name}
    fi

    echo "*************************************************"
    echo -e "\033[36m* 打包项目： ${target_name}\033[0m"
    echo "*************************************************"
    echo ''

    # 将数字123转换为1.2.3的格式输出
    Num_str=""
    caculate(){
	    Parameter_num=$1;
	    if [[ ${Parameter_num} < 10 ]]; then
		    if [[ ${Num_str} == "" ]]; then
			    Num_str="${Parameter_num}"
		    else
			    Num_str="${Parameter_num}.${Num_str}"
		    fi
	    else
		    #商
		    quotient="$[${Parameter_num}/10]"
		    #余数
		    remainder="$[${Parameter_num}%10]"
		    if [[ ${Num_str} == "" ]]; then
			    Num_str="${remainder}"
		    else
			    Num_str="${remainder}.${Num_str}"
		    fi
		    if [[ ${quotient} > 0 ]]; then
			    caculate ${quotient}
		    fi
	    fi
    }

    #--------------------------------------------
    # 处理Version版本号
    #--------------------------------------------
    # Version版本号自增
    if [[ ${Auto_increment_version_num} == "YES" ]]; then
	    if [[ ${Version_str} != "custom" ]]; then
		    echo  -e "\033[31m选项-a 与 -v 不能同时存在\033[0m"
		    exit 2
	    else
		    #取Version版本号
		    Version_str=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" ${App_infoplist_path})
		    #把Version_str字符串根据.分割为数组
		    Array=($(echo $Version_str | tr '.' ' ' | tr -s ' '))
		    #数组长度
		    Array_length=$(echo ${#Array[@]})
		    # 获取版本号数字
		    Version_num=0
		    for (( i = 0; i < ${Array_length}; i++ )); do
			    index_num=${Array[$i]}
			    sum_num="$[10**(${Array_length}-${i}-1)]"
			    index_num="$[${index_num}*${sum_num}]"
			    Version_num="$[${Version_num}+${index_num}]"
		    done

		    Version_num="$[$Version_num+1]"
		    Num_str=""
		    caculate ${Version_num}
		    Version_str=${Num_str}
	    fi
    fi

    #--------------------------------------------
    # 修改Version版本号以及Build号
    #--------------------------------------------
    if [[ ${Version_str} != "custom" ]]; then
	    echo "初始Version = ${Version_initial}"
	    /usr/libexec/Plistbuddy -c "Set CFBundleShortVersionString $Version_str" "${App_infoplist_path}"
	    Version_result=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" ${App_infoplist_path})
	    echo "Version改变后 = ${Version_result}"
    fi
    if [[ ${Build_str} != "custom" ]]; then
	    echo "初始Build = ${Build_initial}"
	    /usr/libexec/Plistbuddy -c "Set CFBundleVersion $Build_str" "${App_infoplist_path}"
	    Build_result=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" ${App_infoplist_path})
	    echo "Build改变后 = ${Build_result}"
    fi

    if [ $? != 0 ]; then
	    /usr/libexec/Plistbuddy -c "Set CFBundleShortVersionString $Version_initial" "${App_infoplist_path}"
	    /usr/libexec/Plistbuddy -c "Set CFBundleVersion $Build_initial" "${App_infoplist_path}"
	    echo -e "\033[31m*************     修改版本号出错     **************\033[0m"
	    exit 2
    fi

    #--------------------------------------------
    # 准备打包
    #--------------------------------------------
    ExportOptionsPlist="${Release_path}/ExportOptions.plist"
    #取当前时间字符串
    Now=$(date +"%Y_%m_%d_%H:%M:%S")
    Project_version=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" ${App_infoplist_path})
    Version_output_path="${Release_path}/ArchivePath/${target_name}/${Project_version}"
    IPA_path="${Version_output_path}/${Now}"
    Archive_file_name="${target_name}_${Project_version}"
    Archive_path="${IPA_path}/${Archive_file_name}.xcarchive"

    mkdir -p "${IPA_path}"
    Log_path="${IPA_path}/LogPath"

    echo "************* xcodebuild clean 项目 **************"

    # 清除项目
    # xcodebuild clean -configuration ${Configuration} &>/dev/null
    xcodebuild clean -configuration ${Configuration} >> $Log_path
    # if [[ ${Edit} == "YES" ]]; then
	    # cd "~/Library/Developer/Xcode"
	    # user=$USER
	    # Xcode_path="/Users/${user}/Library/Developer/Xcode"
	    # cd ${Xcode_path}
	    # for i in `ls`;
	    # do 
	    # 	if [[ ${i} == "DerivedData" ]]; then
	    # 		if [ -d "${i}" ];then
	    # 			# 删除~/Library/Developer/Xcode/DerivedData文件夹
	    # 			rm -rf ${i}
	    # 		fi
	    # 	fi
	    # done
    # fi

    revertVersionNum(){
	    /usr/libexec/Plistbuddy -c "Set CFBundleShortVersionString $Version_initial" "${App_infoplist_path}"
	    /usr/libexec/Plistbuddy -c "Set CFBundleVersion $Build_initial" "${App_infoplist_path}"
	    rm -rf "${IPA_path}"
	    files=`ls ${Version_output_path}`
	    if [ -z "$files" ]; then
		    # 该版本号下没打过包，把该版本号文件夹删除 
	        rm -rf "${Version_output_path}"
	    fi
    } 

    if [ $? = 0 ]; then
	    echo -e "\033[32m************* xcodebuild clean 完成 **************\033[0m"
	    echo ''
	    cd ${Project_path}
    else
	    revertVersionNum;
	    echo -e "\033[31m************* xcodebuild clean 失败 **************\033[0m"
	    echo ''
	    exit 1;
    fi

    Workspace_name="${Project_path}/${target_name}.xcworkspace"
    Project_name="${Project_path}/${target_name}.xcodeproj"
    if [[ ${Edit} == "YES" ]]; then
	    if [[ ${Archive_type} == "workspace" ]];then
		    #编译workspace
		    xcodebuild -workspace "${Workspace_name}" -scheme "${Scheme_name}" -configuration "${Configuration}" >> $Log_path
	    else
		    #编译project
		    xcodebuild -configuration "${Configuration}" >> $Log_path
	    fi

	    if [ $? = 0 ]; then
		    echo -e "\033[32m************* xcodebuild 编译 完成 **************\033[0m"
		    echo ''
	    else
		    revertVersionNum;
		    echo -e "\033[31m************* xcodebuild 编译 失败 **************\033[0m"
		    echo ''
		    exit 1;
	    fi
    fi

    echo -e "\033[36m*************    打包 版本号 ${Project_version}    **************\033[0m"
    echo "************* 开始导出xcarchive文件  *************"
    if [[ ${Archive_type} == "workspace" ]];then
	    #打包workspace
	    xcodebuild archive -workspace "${Workspace_name}" -scheme "${Scheme_name}" -configuration "${Configuration}" -archivePath "${Archive_path}" >> $Log_path
    else
	    #打包project
	    xcodebuild archive -project "${Project_name}" -scheme "${Scheme_name}" -configuration "${Configuration}" -archivePath "${Archive_path}" >> $Log_path
    fi
    if [ $? = 0 ]; then
	    echo -e "\033[32m************* 导出xcarchive文件 完成 **************\033[0m"
	    echo ''
    else
	    revertVersionNum;
	    echo -e "\033[31m************* 导出xcarchive文件 失败 **************\033[0m"
	    echo ''
	    exit 1;
    fi

    echo "*************   开始导出 ipa 文件    **************"
    # xcodebuild -exportArchive -archivePath "${Archive_path}" -exportPath "${IPA_path}" -exportOptionsPlist "${ExportOptionsPlist}" &>/dev/null
    xcodebuild -exportArchive -archivePath "${Archive_path}" -exportPath "${IPA_path}" -exportOptionsPlist "${ExportOptionsPlist}" >> $Log_path
    if [ $? = 0 ]; then
	    echo -e "\033[32m*************    导出 ipa 包完成     **************\033[0m"
	    echo ''
    else
	    revertVersionNum;
	    echo -e "\033[31m*************    导出 ipa 包失败     **************\033[0m"
	    echo ''
	    exit 1;
    fi
    #输出总用时
    echo -e "\033[32m*************  打包完成. 耗时: ${SECONDS}s   **************\033[0m"
    echo ''

***
###脚本详解

-  开始部分，对输入参数进行了判断，脚本命令：`[]表示可选参数，<>表示必填参数`

       ./Release.sh  <Project directory name> [-w] [-s <Name>] [-e] [-d] [-a] [-b <Build number>] [-v <Version number>]

- 第二步，判断并修改Build号以及Version号，`-b <Build number>  -v <Version number>`，分别取对应的值；如果是`-a`则Version号自增，在旧版本号的基础上＋1，其中的主版本号、副版本号、发布号默认采用的是10进制规则，举例
> `1.0.0` 执行`-a`后为 `1.0.1`<br>
> `1.1.9` 执行`-a`后为 `1.2.0`

- 导出xcarchive文件，这里要区分是project项目，还是workspace项目，如果你使用了CocoPads管理项目，那么打包的就是workspace项目

	      if [[ ${Archive_type} == "workspace" ]];then
		      #打包workspace
		      xcodebuild archive -workspace "${Workspace_name}" -scheme "${Scheme_name}" -configuration "${Configuration}" -archivePath "${Archive_path}" >> $Log_path
	      else
		      #打包project
		      xcodebuild archive -project "${Project_name}" -scheme "${Scheme_name}" -configuration "${Configuration}" -archivePath "${Archive_path}" >> $Log_path
	      fi
      
	`${Workspace_name}`是.xcworkspace文件的完整路径（`${Project_name} `一样），`${Scheme_name}`表示项目Scheme的名称，`${Configuration}`有两个值：Debug和Release，`${Archive_path}`是生成的xcarchive文件的导出路径，`>> $Log_path`表示将log日志输出到Log_path文件

- 生成ipa文件

      	xcodebuild -exportArchive -archivePath "${Archive_path}" -exportPath "${IPA_path}" -exportOptionsPlist "${ExportOptionsPlist}" >> $Log_path
      
	`${ExportOptionsPlist}`指向ReleaseDir文件夹下的ExportOptions.plist文件，可以在文件内填写跟打包相关的配置
![QQ20170303-150057@2x.png](http://upload-images.jianshu.io/upload_images/1429982-a749b05d47be9dd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	1. compileBitcode：不上架App Store，Xcode是否启用Bitcode重新编译，默认为YES。
	2. method：归档类型，包括app-store、ad-hoc、
package、enterprise、development以及developer-id。
	3. uploadBitcode：上线App Store是否开启Bitcode，默认为YES。
	4. uploadSymbols：上线App Store，是否开启符号序列化，这是与查crash相关的，默认为YES。

关于更多的xcodebuild指令，可以通过`xcodebuild -help`查看。

***
最后附上两张打包成功的图片

![QQ20170303-151520@2x.png](http://upload-images.jianshu.io/upload_images/1429982-731170b26679c196.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![QQ20170303-151638@2x.png](http://upload-images.jianshu.io/upload_images/1429982-98a992b7ffba67d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
[脚本资源](https://github.com/lele8446/ReleaseDir)👆看这里。另外补充一点，脚本打包前请先在Xcode，General—Targets—Signing 中选择好对应的证书。