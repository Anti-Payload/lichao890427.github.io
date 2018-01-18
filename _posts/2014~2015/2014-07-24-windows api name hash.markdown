---
layout: post
title: 自动生成系统关键api的hash函数
categories: Algorithm
description: 自动生成系统关键api的hash函数
keywords: Windows
---

## 背景
&emsp;&emsp;《0day安全 软件漏洞分析技术》第二版一书中，有一节动态定位API地址的shellcode，一年前看到该篇文章时，第一次发现hash函数的妙用，不过该文章的缺憾是需要对用到的api，自己分析出不产生碰撞的hash函数，如果api数比较多，用文中使用的hash函数就可能碰撞，因此有必要设计出一种自动适应算法自动生成散列函数。  
&emsp;&emsp;文中是这么说的：shellcode最重要放到缓冲区，为了让shellcode更加通用能被大多数缓冲区容纳，总希望shellcode尽可能短。因此在函数名导出表中搜索函数名时，一般不会用"MessageBoxA"这么长的字符串直接比较。通常情况下我们会对所需的api函数进行hash运算，在搜索导出表是对当前遇到的函数进行同样的hash，这样只要比对hash所得的摘要就能判定是不是我们所需的api了，虽然这种搜索算法需要引入额外的hash算法但是可以节省出存储函数名字符串的代码。  
&emsp;&emsp;现在考虑，如果想构造出hash表达式可以对任意api都生成不碰撞摘要，应该怎么做呢？基于算法复杂度和计算时间的考虑，应该采用多元一次表达式，设函数名字符串数组为name[64]，则hash应为k0*name[0]+k1*name[1]+k2*name[2]+....+k63*name[63];现在就是选择一种算法计算出k0~k63。巧妇难为无米之炊，所以我们先要生成api列表，如何获取这个列表呢?列表中应该有哪些api呢？我们仅考虑系统关键函数kernel32.dll user32.dll gdi32.dll ntdll.dll中的导出函数(如需其它函数，做法类似)。参照我之前的一篇文章：http://www.0xaa55.com/thread-312-1-1.html

## 探索
* 1.首先保证dumpbin.exe在搜索路径中
* 2.cd c:\windows\system32
* 3.(for %i in (dir /s /b kernel32.dll gdi32.dll user32.dll ntdll.dll) do dumpbin -exports %i) >out.txt
* 4.findstr /X ".*[0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F].*[a-zA-Z_]$" out.txt > out1.txt
* 5.findstr /V "@  $ \. : characteristics" out1.txt > out2.txt
* 6.notepad++正则表达式  将out2.txt用^.*[0-9A-F]{8} ([a-zA-Z_0-9]+)$替换为($1)
* 7.用perl脚本排序，结果发现out2.txt共4700+个win7 api函数

```Perl
open RESULT,'out1.txt';
my $hash=();
foreach $key (<RESULT>)
{
	chomp $key;
	$hash{$key}=length $key;
}
close RESULT;
open RESULT,'>out5.txt';
select RESULT;

#for $i (sort keys %hash)按字母排序
for $i (sort sort_by_len keys %hash)
{
	print $i,"\n";
}
close RESULT;
sub sort_by_len
{
	if(length($a)<length($b)){-1}
	elsif(length($a)>length($b)){1}
	else{0}
}
```

&emsp;&emsp;统计出来的api最大长度为65，最小为3，win7有不到5000个API。下面开始编程，由于hash要消耗很大内存空间所以先将编译器栈区大小设置为0x400000以上的，我们会生成0x100000元素的大数组，利用率=5000/0x100000=4.7‰。算稀疏矩阵了，碰撞率相对较低(PS：很多hash算法都是0x100000000的，自然碰撞率更低，我曾尝试过0xFFFF的数组，利用率=5000/65536=76.3‰，但是计算太慢了，不知道会不会有解)  
&emsp;&emsp;设coef为hash系数矩阵，apiname为函数名，hash=apiname[0]*coef[0]+apiname[1]*coef[1]+....  

&emsp;&emsp;我在解决这个问题时遇到的问题有：  
* 1.如何设计算法在短时间内(<1min)计算出系数矩阵，且生成的hash摘要尽可能短
* 2.遇到2个api名碰撞以后，如何调整系数矩阵coef使这2个api不碰撞

```C++
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <stdlib.h>
#include <time.h>
using namespace std;
typedef unsigned char ubyte;
typedef unsigned short uword;
int main(int argc, char* argv[])
{
	string filename;//存储api字符串的txt
	vector<string> apinamearray;//存储所有api字符串
	bool find=false;//是否找到正确hash函数
	int maxsize=0;//获取最长api长度
	bool flag[0x100000];//hash表
	int index[0x100000];//用于存储已经占用hash表的api在apinamearray的索引
	//0x100000意味着生成的hash值在0~0x100000，占用20位，比"AA"的长度还小，很节省空间
	//下面2各变量用于定位当前搜索进程
	char curchar='A';
	int curindx=0;
	cout<<"input filename"<<endl;
	cin>>filename;
	ifstream file(filename.c_str());
	string temp;
	while(getline(file,temp))
	{
		if(temp.size() > maxsize)
			maxsize=temp.size();//寻找最长api以设置hash表达式系数数组长度
		apinamearray.push_back(temp);//添加所有api
	}
	time_t begin=time(NULL);
	//hash=apiname[0]*coef[0]+apiname[1]*coef[1]+....
	ubyte* coef=new ubyte[maxsize];//使coef元素不会超过65535
	for(int i=0;i<maxsize;i++)
	{
		coef[i]=0;//初始化系数
	}
	while(!find)
	{
		memset(flag,0,sizeof(flag));
		bool duplicate=false;
		vector<string>::const_iterator itor=apinamearray.begin();
		int size=0;
		int indx=0;
		while(itor != apinamearray.end())
		{
			temp=*itor;
			int sum=1;
			//核心算法
			for(int i=0;i<temp.size();i++)
			{
				sum += temp.at(i)*coef[i];
				sum &= 0xFFFFF;
			}
			if(flag[sum])//如果已存在，则调整hash表达式coef系数
			{
				string other=apinamearray.at(index[sum]);
				printf("%s duplicate with %s\n",temp.c_str(),other.c_str());//输出碰撞函数，便于调整算法
				int i;
				{//显示当前系数矩阵
					for(i=0;i<maxsize;i++)
					{
						if(i%16 == 0)
							printf("\n");
						printf("%d ",coef[i]);
					}
				}*/
							//遇到碰撞调整系数
				int size1=temp.size(),size2=other.size();
				//核心算法：对这2个字符串选最后一位不同字符进行修改，这样就可以使下次计算出的结果不同
				//PS：我先前的方法是吧所有不同位存储起来，随机取一位进行设置，不过效果没有这个好
				if(size1<size2)
				{
					coef[size2-1]++;
				}
				else if(size1>size2)
				{
					coef[size1-1]++;
				}
				else
				{
					for(i=size1-1;i>=0;i--)
					{
						if(temp.at(i) != other.at(i))
						{
							coef[i]++;
							break;
						}
					}
				}
				duplicate=true;
				if(indx>curindx)
				{//显示进度
					curindx=indx;
					printf("%d:%d\n",indx,apinamearray.size());
				}
				if(temp.at(0) > curchar)
				{//显示进度
					curchar=temp.at(0);
					printf("%c %d:%d\n",curchar,indx,apinamearray.size());
				}
				break;//已经碰撞了，所以修正系数后进行下次检测
			}
			else
			{
				flag[sum]=true;
				index[sum]=indx;
			}
			itor++;
			indx++;
		}
		if(duplicate)
		{
		}
		else
		{
			printf("最佳hash系数：");
			for(int i=0;i<maxsize;i++)
			{
				if(i%30 == 0)
					printf("\n");
				printf("%d ",coef[i]);
			}
			printf("\n");
			find=true;//找到hash表达式解
		}
	}
	printf("\ntime used:%d\n",time(NULL)-begin);
	delete []coef;
	return 0;
}
```

## 确定最佳系数
&emsp;&emsp;最佳hash系数：byte hash[50]=
1 1 1 1 2 30 94 117 223 144 78 181 229 235 183 179 112 129 210 209 233 153 137 112 125 149 196 197 187 252
115 40 168 252 58 189 253 201 228 250 192 202 164 101 137 0 44 0 0 119  
time used:22

&emsp;&emsp;确定了hash表达式系数以后，用hash表达式计算出需要的api的hash值存于shellcode中即可，可以发现这样做很节省空间  
&emsp;&emsp;对任意api生成的摘要为0~0x100000的不重复数，如果你有n个api，则shellcode占用空间为：20n+400 位，而直接存储api字符串，即使假设平均长度10字节(如果统计一下，会发现平均为18)，也需要80n 位，当然节省了空间是用时间换取的，计算hash会稍微损耗一点时间，不过影响不大。


## 优化
&emsp;&emsp;对任意api生成的摘要为0~0x40000的不重复数，如果你有n个api，则shellcode占用空间为：18n+800 位，而直接存储api字符串，假设平均长度10字节，同样需要80n 位，如果n较大，这种方式更合适，下面是这个问题的回溯解法，这种方式适合文本按长度排列的情况：

```C++
#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <stdlib.h>
#include <time.h>
using namespace std;
typedef unsigned char ubyte;
vector<string> apinamearray;//存储所有api字符串
bool flag[0x10000];//hash表
int index[0x10000];//用于存储已经占用hash表的api在apinamearray的索引
bool valid(ubyte* coef,int curdepth)
{
	memset(flag,0,sizeof(flag));
	vector<string>::const_iterator itor=apinamearray.begin();
	int indx=0;
	while(itor != apinamearray.end() && curdepth+1 >= (*itor).size())
	{
		int sum=0;
		//核心算法
		int size=(*itor).size();
		for(int i=0;i<size;i++)
		{
			sum += (*itor).at(i)*coef[i];
			sum &= 0xFFFF;
		}   
		if(flag[sum])
		{
			string other=apinamearray.at(index[sum]);
			printf("%s duplicate with %s\n",(*itor).c_str(),other.c_str());
			return false;
		}
		else
		{
			flag[sum]=true;
			index[sum]=indx;
		}
		itor++;
		indx++;
	}
	return true;
}
void showhash(ubyte* coef,int maxsize)
{
	vector<string>::const_iterator itor=apinamearray.begin();
	while(itor != apinamearray.end())
	{
		int sum=0;
		//核心算法
		int size=(*itor).size();
		for(int i=0;i<size;i++)
		{
			sum += (*itor).at(i)*coef[i];
			sum &= 0xFFFF;
		}
		printf("%s->%d ",(*itor).c_str(),sum);
		itor++;
	}
}
int main(int argc, char* argv[])
{
	int maxcoef=64;//参数界限
	string filename;//存储api字符串的txt
	int minsize=INT_MAX;//获取最短api长度
	int maxsize=0;//获取最长api长度
	//下面变量用于定位当前搜索进程
	int curmaxdepth=0;//搜索到的最大深度
	time_t begin=time(NULL);
	cout<<"input filename"<<endl;
	cin>>filename;
	ifstream file(filename.c_str());
	string temp;
	while(getline(file,temp))
	{
		if(temp.size() > maxsize)
			maxsize=temp.size();//寻找最长api以设置hash表达式系数数组长度
		if(temp.size() < minsize)
			minsize=temp.size();//寻找最短api长度
		apinamearray.push_back(temp);
	}
	ubyte* coef=new ubyte[maxsize+1];//使coef元素不会超过65535
	for(int i=0;i<maxsize;i++)
	{
		coef[i]=0;//初始化系数
	}
	int layer=0;
	while(layer >= 0)
	{
		if(curmaxdepth<layer)
		{
			curmaxdepth=layer;
			printf("max:%d:%d\n",layer,maxsize);
		}
		else if(layer < 2)
		{
			printf("%d:%d/%d\n",layer,coef[layer],maxcoef);
		}
		while(coef[layer]<maxcoef && !valid(coef,layer))
		{
			coef[layer]++;
		}
		if(coef[layer]<maxcoef)
		{
			if(layer == maxsize)
			{
				//显示当前系数矩阵
				for(i=0;i<maxsize;i++)
				{
					if(i%16 == 0)
						printf("\n");
					printf("%d ",coef[i]);
					showhash(coef,maxsize);
				}
				printf("\n");
				layer--;
				coef[layer]++;
			}
			else
			{
				layer++;
				coef[layer]=0;
			}
		}
		else
		{
			layer--;
			if(layer>=0)
				coef[layer]++;
		}
	}
	printf("\ntime used:%d\n",time(NULL)-begin);
	delete []coef;
	return 0;
}
```