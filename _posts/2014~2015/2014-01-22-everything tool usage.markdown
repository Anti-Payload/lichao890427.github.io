---
layout: post
title: Everything工具使用
categories: CommonTools
description: Everything工具使用
keywords: 
---

## 一．简介
&emsp;&emsp;Everything是一款免费快速的文件搜索引擎，用于快速搜索特定名称的文件和文件夹，在你输入以后，瞬间会找到并显示匹配列表，是超越win自带搜索几光年的神器。它的特点是：安装文件体积小，用户界面简洁，快速文件索引及搜索，实时捕获文件系统改变，支持正则表达式，系统资源消耗低，自动版本检测及更新，还可以通过http ftp etp共享文件。支持windows 2000，xp， 2003， vista， 2008， windows 7， windows 8。官网www.voidtools.com 现在的版本是1.3，支持64位系统，官网有他的命令行程序和用于第三方开发的api。目前Everything还不能搜索文件内容。  
&emsp;&emsp;安装everything，第一次使用时会建立数据库，每次启动时会检查文件系统有没有改变，首先为了方便使用，对everything作如下配置：  
&emsp;&emsp;工具->选项 打开Everything选项卡,常规中选择语言，必要的时候会提示下载语言包。下面选择“集成到资源管理器”  
&emsp;&emsp;常规->界面        选中“允许多个窗口”和“从系统托盘图标创建窗口”  
&emsp;&emsp;&emsp;&emsp;索引  数据库路径选择一个路径，比如C:\Users\Administrator  
&emsp;&emsp;&emsp;&emsp;接下来，对每个磁盘做如下操作：  
&emsp;&emsp;快捷键：ctrl+s   保存列表结果  
&emsp;&emsp;在栏目上右键可以选择属性，单击排序  

## 二.内部语法
&emsp;&emsp;先介绍搜索专用的特殊字符，这些字符大多是文件名中不能出现的特殊字符。  

|符号|解释                     |例子         |解释                             |
|----|------------------------ |-------------|---------------------------------|
|空格|逻辑与                   |li chao      |文件\(夹\)名中既含li又含chao     |
|\|   |逻辑或                  |1.txt \| 2.txt|文件名含1或2的txt文件            |
|!   |逻辑非                   |\*.txt !b     |文件名不含b的txt文件             |
|< > |提高优先级,类似于数学的()|file:<1 \| 2 >|文件名含1或2的文件(夹)(参见file:)|
|""  |特殊字符串               |"foo bar"    |如果没有""会认为是逻辑与         |

通配符：  
* \*  匹配0-∞个任意字符
* a*.txt  匹配形如”ab.txt” “abbb.txt”
* ?   匹配1个任意字符  
* a??.txt 匹配形如”abc.txt” “aaa.txt”

修饰符：  
* case: 匹配大小写
* file:只匹配文件
* folder:只匹配文件夹
* path:匹配路径和文件名
* regex:正则表达式
* ww:  wholeword:全字匹配

函数： 
* attrib:<属性>      搜索特定属性的目标                             *.txt attrib:a所有存档属性的txt文件
* attributes:<属性> 同上
* datecreated:<date>     搜索特定创建日期的目标
* * *.txt datecreated:lastyear 去年创建的txt文件
* * *.txt datecreated:2010-2012 
* datemodified:<date>   搜索特定修改日期的目标
* dc:<date>   搜索特定创建日期的目标
* dm:<date>搜索特定修改日期的目标
* dupe:          搜索重复目标                               
* empty:       搜索空文件夹
* ext:<list>    搜索指定后缀的目标  用分号分隔
* * file:<ext:bmp;txt>  bmp和txt文件
* len:<length>        筛选出特定长度的目标名
* * *.txt len:5-10   文件名长5至10的txt文件
* parents:<count>  Search for files and folders with the specified number of parent folders.
* size:<size> 搜索特定大小的文件
* * *.txt size:large    1MB-16 MB的txt文件   
* * *.txt size:7mb-8mb

函数语法：  

|function:value     |  等于value   |function:<=value  |小于等于value  |
|-------------------|--------------|------------------|---------------|
|function:<value    | 小于value    |function:=value   |等于value      |
|function:>value    | 大于value    |function:>=value  |大于或等于value|
|function:start..end|范围start到end|function:start-end|范围start到end |

大小语法：  
&emsp;&emsp;size[kb|mb|gb]

大小常数：  

|empty|0KB      |tiny    |0-10 KB    |
|-----|---------|--------|-----------|
|small|10-100 KB|medium  |100KB-1 MB |
|large|1MB-16 MB|gigantic|16MB-128 MB|

日期常数：  
&emsp;&emsp;Today                   yesterday            <last|past|prev|current|this ><week|month|year>

属性常数：  
&emsp;&emsp;R 只读文件  H 隐藏文件  S 系统文件  D 文件夹  A 存档文件  N 普通文件

## 三．正则表达式 regular expression  
正则表达式：(觉得难的跳过，高级话题，这里简单介绍)  
&emsp;&emsp;开启正则表达式：Everything选项卡->常规->Home      Match regex:选择Enabled，新开窗口就可以使用正则表达式了。  
&emsp;&emsp;一般匹配搜索有三种方式：
* 常规搜索：你输入什么搜索什么
* 通配符：使用* ?等符号
* 正则表达式：最复杂也最万能的搜索匹配法
注意，正则表达式内部不能出现多余空格

## 四. 搜索实例

|                  目标                                 |                   语法                    |
|-------------------------------------------------------|-------------------------------------------|
|找到所有c:\windows目录及其下任意子目录的txt文件           | c:\windows\*.txt                          |
|找出所有bmp和jpg文件                                    | *.bmp \| *.jpg                            |
|找出所有名为download文件夹下的所有avi文件                | download\ .avi                            |
|找出所有名字中含.tx的文件夹                              | folder:.tx                                |
|搜索空txt文件                                           | *.txt file:size:0                         |
|搜索所有大于1MB的常见图像文件                            | <*.bmp\|*.jpg\|*.png\|*.tga> size:>1mb    |
|找到所有c:\windows目录下的txt文件                        | regex:c:\\windows\\[^\]*.txt              |
|列出所有c:\windows的N级子目录                            | regex:c:\\windows\\[^\]*(\\[^\]*){N}$     |
|列出所有c:\windows的N级子目录下的txt文件                 | regex:c:\\windows\\[^\]*(\\[^\]*){N}\.txt$|
|查找所有全字匹配1.txt的文件                              | ww:1.txt                                  |
|查找wi开头的h文件和cpp文件                               | file:<wi*.h\|wi*.cpp> or wi* <ext:h\|cpp> |
|XXX第N集.rmvb”，XXX是电视剧名，N是数字                   | regex:.*第[0-9]+集                        |
|连续的RAR压缩包 XXXX.partN.rar，XXXX是压缩包名，N是数字   | regex:.*part[0-9]+.rar                    |
|连续的ZIP压缩包 XXXX.zN                                 | regex:.*\.z[0-9]+                         |
|搜索所有纯中文目标                                       | regex:^[^0-9a-z]*$                        |
|搜索带中文字符的目标                                     | regex:^.*[^!-~]+.*$                       |


## 五．命令行
&emsp;&emsp;Everything的命令行选项：everything提供的命令提供了更多选项，用于配置设置和搜索，窗口的功能大部分都可以通过命令实现，此外还提供编辑搜索列表、全屏显示结果、调试everything、设置数据库等很多小功能。  
&emsp;&emsp;命令行界面的everything：如前所述，官网提供了命令行界面的everything，直接下载就可以用。由于该软件强大的搜索功能，因此用于二次开发也不为过(这就是下面要说的Everything-SDK.zip)，官网也提供了该程序源码，解压后是个es-src文件夹，我采用vc6编译之，为了成功编译，需要建立一个控制台程序，然后在everything_ipc.h里定义typedef unsigned long ULONG_PTR;，之后把工程改成UNICODE的，就成功了。顺便看了他的实现代码，它是通过命令行接受用户的搜索字符串，通过发送一个WM_COPYDATA消息吧这些数据发给后台everything.exe处理，处理完成后会发送回WM_COPYDATA消息给命令行程序，命令行接收搜索结果并显示。  
&emsp;&emsp;Everything-sdk则是一个更加专业的接口可以用c/c++调用，源码提供了封装成dll调用的方法。和命令行界面的everything相比，这个代码更加专业、安全、稳定，除了WM_COPYDATA它还提供了第二种方法实现进程通信，那就是在后台创建一个everything线程，传递参数，目标程序会发送WM_COPYDATA回来，这样就可以接收到搜索结果。  
&emsp;&emsp;注意：上述几种方式都要求后台everything程序在运行且数据库处理完毕。  

## 题外话  
&emsp;&emsp;有人说了，everything是很强大，那么搜索文件内容怎么办呢，我推荐你使用notepad++，同样，这也是一款神器，支持正则表达式，不支持通配符。他的文件查找有一项“文件查找“可以指定单个目录，筛选特定文件类型的文件进行内容查找。曾经有人让我找所有形如E*G*D*I的单词，我下了一个牛津高阶词典，然后使用正则表达式：[a-z]*e[a-z]*g[a-z]*d[a-z]*i[a-z]*  
&emsp;&emsp;然后就开始搜索吧，enjoy it!!!  
