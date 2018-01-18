---
layout: post
title: xp扫雷程序的部分分析
categories: Reverse_Engineering
description: xp扫雷程序的部分分析
keywords: Reverse_Engineering
---

## 简介
&emsp;&emsp;xp扫雷很受欢迎，那么里面有什么古怪呢？让我们进行逆向分析。初学ollydbg，便出此文。 

## 分析布雷过程
&emsp;&emsp;使用调试器附加扫雷程序，选中一块雷区点击，ollydbg便停在入口处(由于程序内部释放焦点，因此我们由winmine切入到ollydbg窗口时)

```Asm
01001BC9  /.  55            PUSH EBP
01001BCA  |.  8BEC          MOV EBP,ESP
01001BCC  |.  83EC 40       SUB ESP,40
01001BCF  |.  8B55 0C       MOV EDX,DWORD PTR SS:[ARG.2]
01001BD2  |.  8B4D 14       MOV ECX,DWORD PTR SS:[ARG.4]
01001BD5  |.  53            PUSH EBX
01001BD6  |.  56            PUSH ESI
01001BD7  |.  33DB          XOR EBX,EBX
01001BD9  |.  57            PUSH EDI
01001BDA  |.  BE 00020000   MOV ESI,200
01001BDF  |.  43            INC EBX
01001BE0  |.  33FF          XOR EDI,EDI
01001BE2  |.  3BD6          CMP EDX,ESI
01001BE4  |.  0F87 75030000 JA 01001F5F


01001F5F  |> \8D82 FFFDFFFF LEA EAX,[EDX-201]                        ; Switch (messages 201..212, 7 exits)
01001F65  |.  83F8 11       CMP EAX,11
01001F68  |.  0F87 3B020000 JA 010021A9
01001F6E  |.  0FB680 DE2100 MOVZX EAX,BYTE PTR DS:[EAX+10021DE]
01001F75  |.  FF2485 C22100 JMP DWORD PTR DS:[EAX*4+10021C2]
```

跳几次，就到WM_LBUTTONUP的位置了
```Asm
01001FDF  |> \33FF          XOR EDI,EDI                              ; Cases 202 (WM_LBUTTONUP), 205 (WM_RBUTTONUP), 208 (WM_MBUTTONUP) of switch winmine.1001F5F
01001FE1  |.  393D 40510001 CMP DWORD PTR DS:[1005140],EDI
01001FE7  |.  0F84 BC010000 JE 010021A9

01001FED  |> /893D 40510001 MOV DWORD PTR DS:[1005140],EDI
01001FF3  |. |FF15 D8100001 CALL DWORD PTR DS:[<&USER32.ReleaseCaptu ; [USER32.ReleaseCapture
01001FF9  |. |841D 00500001 TEST BYTE PTR DS:[1005000],BL
01001FFF  |. |0F84 B6000000 JZ 010020BB
01002005  |. |E8 D7170000   CALL 010037E1                            ; [winmine.010037E1
0100200A  |. |E9 9A010000   JMP 010021A9
0100200F  |> |393D 48510001 CMP DWORD PTR DS:[1005148],EDI           ; Case 204 (WM_RBUTTONDOWN) of switch winmine.1001F5F
```

&emsp;&emsp;由于下面已经是WM_RBUTTONDOWN的判断了，因此CALL 010021A9是处理WM_LBUTTONUP的函数，我们F7跟进

```Asm
010037E1  /$  A1 18510001   MOV EAX,DWORD PTR DS:[1005118]           ; winmine.010037E1(guessed void)
010037E6  |.  85C0          TEST EAX,EAX
010037E8  |.  0F8E C8000000 JLE 010038B6
010037EE  |.  8B0D 1C510001 MOV ECX,DWORD PTR DS:[100511C]
010037F4  |.  85C9          TEST ECX,ECX
010037F6  |.  0F8E BA000000 JLE 010038B6
010037FC  |.  3B05 34530001 CMP EAX,DWORD PTR DS:[1005334]
01003802  |.  0F8F AE000000 JG 010038B6
01003808  |.  3B0D 38530001 CMP ECX,DWORD PTR DS:[1005338]
0100380E  |.  0F8F A2000000 JG 010038B6
01003814  |.  53            PUSH EBX
01003815  |.  33DB          XOR EBX,EBX
01003817  |.  43            INC EBX
01003818  |.  833D A4570001 CMP DWORD PTR DS:[10057A4],0             ; 次数？
0100381F  |.  75 4A         JNE SHORT 0100386B
01003821  |.  833D 9C570001 CMP DWORD PTR DS:[100579C],0             ; 时间？
01003828  |.  75 41         JNE SHORT 0100386B
0100382A  |.  53            PUSH EBX                                 ; /Arg1 => 1
0100382B  |.  E8 BD000000   CALL 010038ED                            ; \winmine.010038ED
01003830  |.  FF05 9C570001 INC DWORD PTR DS:[100579C]
01003836  |.  E8 7AF0FFFF   CALL 010028B5                            ; [winmine.010028B5
0100383B  |.  6A 00         PUSH 0                                   ; /TimerFunc = 00000000
0100383D  |.  68 E8030000   PUSH 3E8                                 ; |Timeout = 1000. ms
01003842  |.  53            PUSH EBX                                 ; |TimerID
01003843  |.  FF35 245B0001 PUSH DWORD PTR DS:[1005B24]              ; |hWnd = 000B047C, class = 扫雷, text = 扫雷
01003849  |.  891D 64510001 MOV DWORD PTR DS:[1005164],EBX           ; |
0100384F  |.  FF15 B4100001 CALL DWORD PTR DS:[<&USER32.SetTimer>]   ; \USER32.SetTimer
01003855  |.  85C0          TEST EAX,EAX
01003857  |.  75 07         JNZ SHORT 01003860
01003859  |.  6A 04         PUSH 4                                   ; /Arg1 = 4
0100385B  |.  E8 F0000000   CALL 01003950                            ; \winmine.01003950
01003860  |>  A1 18510001   MOV EAX,DWORD PTR DS:[1005118]
01003865  |.  8B0D 1C510001 MOV ECX,DWORD PTR DS:[100511C]
0100386B  |>  841D 00500001 TEST BYTE PTR DS:[1005000],BL
01003871  |.  5B            POP EBX
01003872  |.  75 10         JNZ SHORT 01003884
01003874  |.  6A FE         PUSH -2
01003876  |.  59            POP ECX
01003877  |.  8BC1          MOV EAX,ECX
01003879  |.  890D 1C510001 MOV DWORD PTR DS:[100511C],ECX
0100387F  |.  A3 18510001   MOV DWORD PTR DS:[1005118],EAX
01003884  |>  833D 44510001 CMP DWORD PTR DS:[1005144],0
0100388B  |.  74 09         JE SHORT 01003896
0100388D  |.  51            PUSH ECX                                 ; /Arg2
0100388E  |.  50            PUSH EAX                                 ; |Arg1
0100388F  |.  E8 23FDFFFF   CALL 010035B7                            ; \winmine.010035B7
01003894  |.  EB 20         JMP SHORT 010038B6
01003896  |>  8BD1          MOV EDX,ECX
01003898  |.  C1E2 05       SHL EDX,5
0100389B  |.  8A9402 405300 MOV DL,BYTE PTR DS:[EAX+EDX+1005340]
010038A2  |.  F6C2 40       TEST DL,40
010038A5  |.  75 0F         JNZ SHORT 010038B6
010038A7  |.  80E2 1F       AND DL,1F
010038AA  |.  80FA 0E       CMP DL,0E
010038AD  |.  74 07         JE SHORT 010038B6
010038AF  |.  51            PUSH ECX                                 ; /Arg2
010038B0  |.  50            PUSH EAX                                 ; |Arg1
010038B1  |.  E8 5CFCFFFF   CALL 01003512                            ; \winmine.01003512
010038B6  |>  FF35 60510001 PUSH DWORD PTR DS:[1005160]              ; /Arg1 = 0
010038BC  |.  E8 52F0FFFF   CALL 01002913                            ; \winmine.01002913
010038C1  \.  C3            RETN
```

&emsp;&emsp;由于雷区数据不应该随函数的退出而消失，可以判断数据存放在静态或者全局变量区，所以着重关注DS:[100??]这种地方，多试几次可知
10057A4记录了用户挖雷次数，而100579C处记录着已用时间，真正的布雷函数显然在那几个call中。你在这些汇编代码中发现了SetTimer这个函数，你在游戏时可能已经发现，在用户挖第一个雷时才会计数，因此call 010038ED和call 010028B5处一定有猫腻，第一个点进去以后是这样：

```Asm
010038ED  /$  833D B8560001 CMP DWORD PTR DS:[10056B8],3             ; winmine.010038ED(guessed Arg1)
010038F4  |.  75 47         JNE SHORT 0100393D
010038F6  |.  8B4424 04     MOV EAX,DWORD PTR SS:[ARG.1]
010038FA  |.  48            DEC EAX                                  ; Switch (cases 1..3, 4 exits)
010038FB  |.  74 2A         JZ SHORT 01003927
010038FD  |.  48            DEC EAX
010038FE  |.  74 15         JZ SHORT 01003915
01003900  |.  48            DEC EAX
01003901  |.  75 3A         JNZ SHORT 0100393D
01003903  |.  68 05000400   PUSH 40005                               ; Case 3 of switch winmine.10038FA
01003908  |.  FF35 305B0001 PUSH DWORD PTR DS:[1005B30]
0100390E  |.  68 B2010000   PUSH 1B2
01003913  |.  EB 22         JMP SHORT 01003937
01003915  |>  68 05000400   PUSH 40005                               ; Case 2 of switch winmine.10038FA
0100391A  |.  FF35 305B0001 PUSH DWORD PTR DS:[1005B30]
01003920  |.  68 B1010000   PUSH 1B1
01003925  |.  EB 10         JMP SHORT 01003937
01003927  |>  68 05000400   PUSH 40005                               ; Case 1 of switch winmine.10038FA
0100392C  |.  FF35 305B0001 PUSH DWORD PTR DS:[1005B30]
01003932  |.  68 B0010000   PUSH 1B0
01003937  |>  FF15 68110001 CALL DWORD PTR DS:[<&WINMM.PlaySoundW>]  ; \WINMM.PlaySoundW
0100393D  \>  C2 0400       RETN 4                                   ; Default case of switch winmine.10038FA
```

&emsp;&emsp;可以看出这个函数就是在你点击了雷以后发出声音的，没什么价值，但是此时你注意到下面的一个函数用到了rand，对啊，这个随机数在初始化雷阵的时候肯定用得到啊，先下个断点再说。开始新游戏，然后扫雷就断在rand函数处：

```Asm
01003940  /$  FF15 B0110001 CALL DWORD PTR DS:[<&msvcrt.rand>]       ; [MSVCRT.rand
01003946  |.  99            CDQ
01003947  |.  F77C24 04     IDIV DWORD PTR SS:[ARG.1]
0100394B  |.  8BC2          MOV EAX,EDX
0100394D  \.  C2 0400       RETN 4
```

&emsp;&emsp;这个函数显然是用来取rand()%N的，运行到winmine代码中，会发现一个循环，很重要：

```Asm
010036C7  |> /FF35 34530001 /PUSH DWORD PTR DS:[1005334]             ; /Arg1 = 9
010036CD  |. |E8 6E020000   |CALL 01003940                           ; \winmine.01003940
010036D2  |. |FF35 38530001 |PUSH DWORD PTR DS:[1005338]             ; /Arg1 = 9
010036D8  |. |8BF0          |MOV ESI,EAX                             ; |
010036DA  |. |46            |INC ESI                                 ; |
010036DB  |. |E8 60020000   |CALL 01003940                           ; \winmine.01003940
010036E0  |. |40            |INC EAX
010036E1  |. |8BC8          |MOV ECX,EAX
010036E3  |. |C1E1 05       |SHL ECX,5
010036E6  |. |F68431 405300 |TEST BYTE PTR DS:[ESI+ECX+1005340],80
010036EE  |.^\75 D7         |JNZ SHORT 010036C7
010036F0  |. |C1E0 05       |SHL EAX,5
010036F3  |. |8D8430 405300 |LEA EAX,[ESI+EAX+1005340]
010036FA  |. |8008 80       |OR BYTE PTR DS:[EAX],80
010036FD  |. |FF0D 30530001 |DEC DWORD PTR DS:[1005330]
01003703  |.^\75 C2         \JNZ SHORT 010036C7
```

&emsp;&emsp;这段代码经过逆向分析可以得到伪代码：
* [1005338]处存放棋盘大小  SIZE
* [1005330]处存放余下要摆放的雷数  MineToSet
* [1005340]存放棋盘数据  BYTE DATA[32][32]   我们初级模式9*9只用到[1][1]-[9][9]的数据

```C++
while(MineToSet)
{
  int line=rand%SIZE+1,column=rand%SIZE+1;
  if((DATA[line][column]&0x80) == 0)
  {//如果不是雷
    DATA[line][column] |= 0x80;//设置为雷
    MineToSet--;
  }
}
```

&emsp;&emsp;我们从ollydbg的内存窗口取得1005340的数据，分析以后可以得到雷区数组，可以得知8F处的为雷，0F处的不是雷  
![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_12.png)


## 问题思考
* 为什么选择WM_LBUTTONUP?  
&emsp;&emsp;这个你可以动手试试就知道了，你在雷区按下(设该点为A)鼠标不放，移动到别处(设该点为B)，若B也在雷区，则B处鼠标释放后程序视为挖雷
而如果在外边则什么效果也没有。相反如果你在雷区之外A处按下鼠标不放，拖到雷区之内B处释放，则B处雷挖开。可见是否挖雷与LBUTTONDOWN的位置无关，而与LBUTTONUP位置有关  
* 为什么初始布置的雷区和实际的不一样？如上图，左上角的雷本是没有的，而[3][4]处本来应该有雷，为什么没有？  
&emsp;&emsp;其实这2处在用户第一次点击雷区时被winmine偷偷替换掉了，也就是说用户第一次就点中了雷!但是这是不对的，第一次就中雷就没意思了。所以winmine釜底抽薪，吧这个雷换到别的地方去了。xp版winmine变态的地方在于，点人脸的时候会布雷，而点第一次的时候你永远不会踩中雷，因为他在第一次的时候偷偷把他换掉。如果你第一次就点中了布好的雷上，他就把这个位置换到别的地方，让他不是雷，如果你没点到雷上自然啥都不做，***所以第一次永远点不到雷上***。。。
&emsp;&emsp;[1][1]-[9][9]之外的，都是0x10    [1][1]-[9][9]初始化为0x0F，然后每次你按人脸的时候，布9科雷 布雷就是发现arr[j]&0x80 == 0的时候进行arr[j] |= 0x80 高位置1，点了第一次，他会判断你是否中了雷，中了，就把雷放到别的地方去，反正总是让你第一次不中雷，中了，他自然再去获取随机数，找到一个没有雷的位置，然后设置为雷，然后把你踩的地方设为非雷

## 扫雷内挂分析
&emsp;&emsp;先介绍一下xp扫雷内挂用法：
* 1.打开扫雷界面
* 2.输入X Y Z Z Y
* 3.按一下右下角的shift键  
 
&emsp;&emsp;这时，鼠标放在雷区活动，你会看到屏幕左上角有个小光点在一闪一闪。（很小很小的，不容易看。最好桌面是深色的，要不看不清），小光点出现，说明鼠标停在的格子不是雷，没有小光点，就是雷区！

```C++
static int index=0;
static WCHAR str[]=L"XYZZY";
static int xCur=-1,yCur=-1;
static xBoxMax,yBoxMax;//雷区大小，初级为9*9，存储在注册表中
static BYTE data[32][32];//雷区数据
switch(uMsg)
{
	case WM_KEYDOWN:
	{
		switch(wParam）
		{
			case VK_SHIFT:
				if(index >= 5)
						index ^= 0x14;//如果前5字符正确，后面只要index>=5即可
				return DefWindowProc();
			break;
			
			case VK_F4:        
			case VK_F5:
			case VK_F6:
			...无关代码
			break;
		
			default:
				if(index < 5)
				{
					if(str[index] == wParam)
							index++;
					else
							index=0;
				}
			}
		}
		break;
	
		case WM_MOUSEMOVE:
		{        
			......
			if(index)
			{
				if((index == 5 && wParam & MK_CONTROL) || index > 5)
				{
					int x=LOWORD(lParam);
					int y=HIWORD(lParam);
					xCur=x/16;//这个16显然可以推测为一个雷格子的大小了。。。
					yCur=(y-39)/16;//39同样可以推测为窗口顶部到雷区上边缘长度
					if(xCur>0 && yCur>0 && xCur<xBoxMax && yCur<yBoxMax)
					{
						HDC hdc=GetDC(NULL);
						//注意下面data[yCur][xCur]是对的，yCur代表行，然而在绘图中却是竖直方向
						if(data[yCur][xCur]&0x80)//如果是雷
								SetPixel(hdc,0,0,RGB(0,0,0_);//显示为黑
						else
								SetPixel(hdc,0,0,RGB(255,255,255);                                        
						ReleaseDC(NULL,hdc);
					}
				}
			}
		}
		break;
	}
	return DefWindowProc();
}
```
