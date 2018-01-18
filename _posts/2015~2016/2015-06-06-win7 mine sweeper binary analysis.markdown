---
layout: post
title: 关于Win7扫雷逆向分析及外挂编写 
categories: Reverse_Engineering
description: 关于Win7扫雷逆向分析及外挂编写 
keywords: 
---

## 入口

&emsp;&emsp;我系统Win8.1，直接把Win7 MineSweeper那个文件夹弄过来，，是无法运行扫雷的。由于是系统组件，ida载入可以得到符号，先研究为何无法运行！从用户入口WinMain开始跟：

```C++
int __stdcall WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nShowCmd)
{
  DWORD v4; // ecx@1
  int result; // eax@2
  HMODULE v6; // eax@3
  void *v7; // eax@6
  void *v8; // esi@7
  WCHAR Buffer; // [sp+4h] [bp-24h]@3
  char v10; // [sp+6h] [bp-22h]@3
  __int16 v11; // [sp+22h] [bp-6h]@3

  HeapSetInformation(0, HeapEnableTerminationOnCorruption, 0, 0);
  if ( IsGamePlayable(v4, L"Shell-InBoxGames-Minesweeper-EnableGame") )
  {
    g_hInstance = hInstance;
    Buffer = 0;
    memset(&v10, 0, 0x1Cu);
    v11 = 0;
    v6 = GetModuleHandleW(0);
    if ( LoadStringW(v6, 0xFBCu, &Buffer, 16) && !lstrcmpW(&Buffer, L"0") )
      g_Flowerbed = 1;
    v7 = operator new(4u);
    if ( v7 )
    {
      *(_DWORD *)v7 = EngineHandler::`vftable';
      v8 = v7;
    }
    else
    {
      v8 = 0;
    }
    RegisterLogNameResolver((const unsigned __int16 *(__stdcall *)(unsigned int))MsLog::MsLogResolver);
    if ( InitializeEngine((struct IEngineInterface *)v8, 0) )
    {
      if ( v8 )
        IEngineInterface::`scalar deleting destructor'(v8, 1);
      result = 0;
    }
    else
    {
      if ( v8 )
        IEngineInterface::`scalar deleting destructor'(v8, 1);
      result = 1;
    }
  }
  else
  {
    result = 1;
  }
  return result;
}
```

&emsp;&emsp;发现IsGamePlayable处返回失败，在此需要将返回值改为TRUE，接着调试，发现InitializeEngine执行后直接进程结束，因此跟入该函数：

```C++
bool __stdcall InitializeEngine(struct IEngineInterface *a1, struct IControllerInterface *a2)
{
  v71 = a1;
  v72 = a2;
  v2 = (DialogHelper *)GetModuleHandleW(0);
  DialogHelper::Init(v2, 0, 0, nSize);
  v4 = (*(int (__thiscall **)(struct IEngineInterface *, DWORD))(*(_DWORD *)a1 + 16))(a1, v3);//EngineHandler::GetProjectName
  v5 = v4 + 2;
  do
  {
    v6 = *(_WORD *)v4;
    v4 += 2;
  }
  while ( v6 );
  v7 = ((v4 - v5) >> 1) + 24;
  v8 = (wchar_t *)operator new[](2 * v7);
  Dst = v8;
  if ( v8 )
  {
    wcscpy_s(v8, v7, L"Local\\Oberon_");
    v10 = (const wchar_t *)(*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)a1 + 16))(a1);//EngineHandler::GetProjectName
    wcscat_s(Dst, v7, v10);
    wcscat_s(Dst, v7, L"_Singleton");
    v11 = CreateMutexW(0, 1, Dst);
    operator delete(Dst);
    if ( v11 )
    {
      if ( GetLastError() != 183 )
      {
        v15 = GetCommandLineW();
        pNumArgs = 0;
        v16 = CommandLineToArgvW(v15, &pNumArgs);
        v17 = 1;
        pv = v16;
        v73 = 1;
        if ( pNumArgs > 1 )
        {
          do
          {
            if ( !wcscmp(*((const unsigned __int16 **)pv + v17), L"-mce") )
              g_bMediaCenter = 1;
            v17 = v73++ + 1;
          }
          while ( (signed int)v73 < pNumArgs );
        }
        v18 = GetCommandLineW();
        RegisterApplicationRestart(v18, 0);
        CoInitialize(0);
        v76 = 0;
        Dst = 0;
        v79 = 0;
        if ( CoCreateInstance(
               &_GUID_9a5ea990_3034_4d6f_9128_01f3c61022bc,
               0,
               1u,
               &_GUID_e7b2fb72_d728_49b3_a5f2_18ebf5f1349e,
               (LPVOID *)&Dst) >= 0
          && SHGetFolderPathEx(&FOLDERID_ProgramFiles, 0, 0, &psz, 260) >= 0 )
        {
          v19 = (const unsigned __int16 *)(*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)a1 + 84))(a1);//EngineHandler::GetGdfPath
          if ( StringCchCatW(&psz, 0x104u, v19) >= 0 )
          {
            v20 = SysAllocString(&psz);
            g_bstrGDFPath = v20;
            if ( v20 )
            {
              if ( SysStringLen(v20)
                && (*(int (__stdcall **)(wchar_t *, unsigned __int16 *, void **))(*(_DWORD *)Dst + 24))(//CGameExplorer::VerifyAccess 
                     Dst,
                     g_bstrGDFPath,
                     &pv) >= 0
                && pv )
                v76 = 1;
            }
          }
        }
        v79 = -1;
        if ( Dst )
          (*(void (__stdcall **)(wchar_t *))(*(_DWORD *)Dst + 8))(Dst);
        if ( !v76 )
          CleanupEngine(5);
        v21 = operator new(4u);
        if ( v21 )
        {
          *(_DWORD *)v21 = 0;
          g_pSecondTimerCallback = v21;
        }
        else
        {
          g_pSecondTimerCallback = 0;
        }
        _set_new_handler(HandleNewFail);
        _set_new_mode(1);
        g_pInterface = (void *)a1;
        v22 = (wchar_t *)(*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)a1 + 16))(a1);
        g_wszProjectName = LocalizeMessage(v22);
        v23 = GetModuleFileNameW(0, Filename, 0x200u);
        for ( i = v23 - 1; ; --i )
        {
          if ( i > v23 )
            goto LABEL_38;
          v25 = Filename[i];
          if ( v25 == 47 || v25 == 92 )
            break;
        }
        Filename[i] = 0;
LABEL_38:
        SetCurrentDirectoryW(Filename);
        XmlNode::SetNodeName((XmlNode *)&ErrorXml, L"ErrorLog");
        v26 = operator new(0x10u);
        pv = v26;
        v79 = 1;
        if ( v26 )
          v27 = (XmlManager *)XmlManager::XmlManager((XmlManager *)v26);
        else
          v27 = 0;
        v79 = -1;
        g_pXmlManager = v27;
        CheckAllocation((const void *)v27);
        ResourceBase::SetNameForDPI(v28, (bool)g_wszProjectName);
        Log(0x10u, 1129, "Engine.cpp", L"Initializing Virtual FS");
        if ( ResourceWMA::LoadAsNeeded(v29) )
        {
          Log(0x10u, 1138, "Engine.cpp", L"Project name localized to: %s", g_wszProjectName);
          OutputDebugStringW(Filename);
          Log(0x10u, 1142, "Engine.cpp", L"Using working directory: %s", Filename);
          Log(0x10u, 1146, "Engine.cpp", L"Initializing GDI+");
          v56 = 1;
          v58 = 1;
          v57 = 0;
          v59 = 0;
          GdiplusStartup(&g_GdiPlusToken, &v56, &v60);
          v63 = 0;
          v62 = 0;
          LoadWindowPrefs(&v63, &v62);
          pv = operator new(0x1440u);
          v79 = 2;
          if ( pv )
            v30 = (DllFileMgr *)DllFileMgr::DllFileMgr(0);
          else
            v30 = 0;
          v79 = -1;
          ::pv = v30;
          v31 = (wchar_t *)(**(int (***)(void))v71)();
          Str::Str(v31);
          v79 = 3;
          v32 = DllFileMgr::Open(::pv, (const struct Str *)&pRawInputDevices, g_bDebugEnabled, L"input\\");
          v79 = -1;
          v76 = v32 == 0;
          Str::~Str((Str *)&pRawInputDevices);
          if ( v76 )
          {
            D3DXTex::TF_Row::TF_Row((D3DXTex::TF_Row *)&pRawInputDevices);
            v79 = 4;
            v33 = *(wchar_t **)(Str::Str(0x37Cu) + 8);
            v34 = *(int (***)(void))v71;
            LOBYTE(v79) = 5;
            v35 = (*v34)();
            Str::Format((Str *)&pRawInputDevices, v33, v35);
            LOBYTE(v79) = 4;
            Str::~Str((Str *)&v54);
            DialogHelper::ShowMessageBox(
              0x385u,
              0,
              1u,
              0xFFFEu,
              0,
              (const unsigned __int16 *)pRawInputDevices.hwndTarget,
              (const unsigned __int16 *)1,
              (bool)v52);
          }
          else if ( CreateGameWindow() )
          {
            if ( g_bMediaCenter )
            {
              Log(0x10u, 1211, "Engine.cpp", L"Initializing MCE Dialog");
              DialogHelper::InitMCE(0, v52);
              if ( g_bMediaCenter )
              {
                Log(0x10u, 1220, "Engine.cpp", L"Registering for raw input, for remote control");
                pRawInputDevices.usUsagePage = 12;
                pRawInputDevices.usUsage = 1;
                pRawInputDevices.hwndTarget = g_hWnd;
                pRawInputDevices.dwFlags = 0;
                if ( !RegisterRawInputDevices(&pRawInputDevices, 1u, 0xCu) )
                {
                  v36 = GetLastError();
                  Log(0x10u, 1231, "Engine.cpp", L"Register failed, winerror %d", v36);
                }
              }
            }
            Log(0x10u, 1237, "Engine.cpp", L"Adding system events");
            Event::RegisterEventType(2, Event_MouseEnter::Create);
            Event::RegisterEventType(4, Event_MouseDown::Create);
            Event::RegisterEventType(5, Event_MouseDoubleClick::Create);
            Event::RegisterEventType(3, Event_MouseLeave::Create);
            Event::RegisterEventType(6, Event_MouseRelease::Create);
            Event::RegisterEventType(7, Event_MouseReleaseOut::Create);
            Event::RegisterEventType(8, Event_MouseStatusBarClick::Create);
            Event::RegisterEventType(9, Event_MouseOuterDown::Create);
            Event::RegisterEventType(10, Event_MouseOuterRelease::Create);
            Event::RegisterEventType(11, Event_MouseGlobalRelease::Create);
            Event::RegisterEventType(1, Event_ButtonClick::Create);
            Event::RegisterEventType(13, Event_AnimationComplete::Create);
            Event::RegisterEventType(12, Event_AnimationFrame::Create);
            Event::RegisterEventType(14, Event_AnimationInt::Create);
            Event::RegisterEventType(15, Event_Timer::Create);
            Event::RegisterEventType(16, Event_FocusCheck::Create);
            Event::RegisterEventType(17, Event_FocusGot::Create);
            Event::RegisterEventType(18, Event_FocusLost::Create);
            Event::RegisterEventType(19, Event_FocusSelect::Create);
            Event::RegisterEventType(20, Event_FocusUnSelect::Create);
            Event::RegisterEventType(21, Event_FocusGetNodeAtDir::Create);
            Event::RegisterEventType(22, Event_FocusGetNextTabNode::Create);
            Event::RegisterEventType(23, Event_InitComplete::Create);
            Event::RegisterEventType(24, Event_DoDefaultAction::Create);
            Event::RegisterEventType(25, Event_DragCheck::Create);
            Event::RegisterEventType(26, Event_DragStart::Create);
            Event::RegisterEventType(27, Event_DragMove::Create);
            Event::RegisterEventType(28, Event_DragEnd::Create);
            Event::RegisterEventType(29, Event_KeyDown::Create);
            Event::RegisterEventType(30, Event_KeyUp::Create);
            Event::RegisterEventType(31, Event_AccessGetParent::Create);
            Event::RegisterEventType(32, Event_AccessGetChildCount::Create);
            Event::RegisterEventType(33, Event_AccessGetChild::Create);
            Event::RegisterEventType(34, Event_TipClose::Create);
            Event::RegisterEventType(37, Event_DpiChange::Create);
            Event::RegisterEventType(35, Event_ControllerKey::Create);
            Log(0x10u, 1280, "Engine.cpp", L"Registering Node Types");
            NodeButton::Register();
            NodeLabel::Register();
            NodeBase::Register();
            NodeSprite::Register();
            NodeEmitter::Register();
            NodeShot::Register();
            NodeNumber::Register();
            Log(0x10u, 1291, "Engine.cpp", L"Initializing Timekeeping");
            if ( Timekeeping::InitializeTimekeeping((Timekeeping *)v52) )
            {
              DialogHelper::SetDialogShutdownCallback((DialogHelper *)Engine_ResetTimer, v53);
              Log(0x10u, 1304, "Engine.cpp", L"Initializing ResourceManager");
              v37 = operator new(0x54u);
              pv = v37;
              v79 = 6;
              if ( v37 )
                v38 = (ResourceManager *)ResourceManager::ResourceManager(v37);
              else
                v38 = 0;
              v79 = -1;
              g_pResourceManager = v38;
              CheckAllocation((const void *)v38);
              if ( ResourceManager::Initialize(g_pResourceManager) )
              {
                v65 = 0;
                v66 = 0;
                v68 = 0;
                v64 = g_hRenderWindow;
                LOWORD(v68) = 0;
                v67 = 32;
                (*(void (__stdcall **)(int *, int *))(*(_DWORD *)g_pInterface + 12))(&v65, &v66);
                if ( g_bDoubleDPI )
                {
                  v65 *= 2;
                  v66 *= 2;
                }
                Log(0x10u, 1331, "Engine.cpp", L"Initializing RenderManager");
                v39 = operator new(0x90u);
                pv = v39;
                v79 = 7;
                if ( v39 )
                  v40 = (RenderManager *)RenderManager::RenderManager(v39);
                else
                  v40 = 0;
                v79 = -1;
                g_pRenderManager = v40;
                CheckAllocation((const void *)v40);
                if ( RenderManager::Initialize(g_pRenderManager, (const struct RenderInitializeOptions *)&v64) )
                {
                  Log(0x10u, 1342, "Engine.cpp", L"Initializing Audio");
                  v41 = operator new(0x28u);
                  pv = v41;
                  v79 = 8;
                  if ( v41 )
                    v42 = (Audio *)Audio::Audio(v41);
                  else
                    v42 = 0;
                  v79 = -1;
                  g_pAudio = v42;
                  CheckAllocation((const void *)v42);
                  if ( Audio::Initialize(g_pAudio) )
                  {
                    Log(0x10u, 1354, "Engine.cpp", L"Initializing XNA Common Controller");
                    pv = operator new(0x80u);
                    v79 = 9;
                    if ( pv )
                      v43 = (CommonControllerThread *)CommonControllerThread::CommonControllerThread(
                                                        (int)g_hWnd,
                                                        g_bMediaCenter,
                                                        (int)v72);
                    else
                      v43 = 0;
                    v79 = -1;
                    g_pCommonController = v43;
                    Thread::Begin(v43);
                    Log(0x10u, 1361, "Engine.cpp", L"Initializing Timer");
                    v44 = operator new(0x34u);
                    if ( v44 )
                      v45 = (ResourceWMA *)Timer::Timer(v44);
                    else
                      v45 = 0;
                    g_pTimer = v45;
                    CheckAllocation((const void *)v45);
                    if ( ResourceWMA::LoadAsNeeded(g_pTimer) )
                    {
                      Log(0x10u, 1372, "Engine.cpp", L"Loading Fonts");
                      v46 = XmlManager::GetXml(g_pXmlManager, L"xml\\Fonts.xml");
                      if ( v46 )
                      {
                        pv = XmlNode::XPathElementSearch(v46, L"/Font", &v73);
                        Log(0x10u, 1383, "Engine.cpp", L"%d Fonts Found", v73);
                        for ( Dst = 0; (unsigned int)Dst < v73; Dst = (wchar_t *)((char *)Dst + 1) )
                        {
                          v47 = XmlNode::GetNodeValue(*((XmlNode **)pv + (_DWORD)Dst));
                          Log(0x10u, 1388, "Engine.cpp", L"Loading Font: %s", v47);
                          if ( AddFontResourceW(v47) )
                          {
                            Array<NodeBase *>::Add(v47);
                          }
                          else
                          {
                            Log(0x10u, 1412, "Engine.cpp", L"Couldn't add font: %s", v47);
                            operator delete(v47);
                          }
                        }
                        operator delete(pv);
                      }
                      else
                      {
                        Log(0x10u, 1426, "Engine.cpp", L"No Font Xml Found");
                      }
                      Log(0x10u, 1431, "Engine.cpp", L"Registering for session notification");
                      WTSRegisterSessionNotification(g_hWnd, 0);
                      v72 = XmlManager::GetXml(g_pXmlManager, L"engine.xml");
                      g_bInInitializer = 1;
                      Log(0x10u, 1439, "Engine.cpp", L"Engine Initialization Complete: Initializing Game Code.");
                      v48 = v71;
                      if ( !(unsigned __int8)(*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)v71 + 48))(v71) )
                      {
                        Log(0x10u, 1442, "Engine.cpp", L"Game code initialization failed.");
                        CleanupEngine(0);
                      }
                      g_bInInitializer = 0;
                      if ( v72 )
                        g_bFocusPause = XmlNode::GetXmlInt(v72, L"/PauseOnLostFocus", -1) > 0;
                      Engine_LoadingComplete();
                      g_Accelerator = 0;
                      if ( (*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)v48 + 40))(v48) )
                      {
                        v49 = (*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)v48 + 40))(v48);
                        v50 = GetModuleHandleW(0);
                        g_Accelerator = LoadAcceleratorsW(v50, (LPCWSTR)v49);
                      }
                      g_bInitializing = 0;
                      v60(&g_GdiPlusToken);
                      while ( 1 )
                      {
                        while ( !PeekMessageW(&Msg, 0, 0, 0, 0) )
                          RunEngine();
                        if ( !GetMessageW(&Msg, 0, 0, 0) )
                          break;
                        if ( !g_Accelerator || !TranslateAcceleratorW(g_hWnd, g_Accelerator, &Msg) )
                        {
                          TranslateMessage(&Msg);
                          DispatchMessageW(&Msg);
                        }
                      }
                      v61(g_GdiPlusToken);
                      (*(void (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)v48 + 72))(v48);
                      CleanupEngine(0);
                    }
                    Log(0x10u, 1365, "Engine.cpp", L"Failed to initialize Timer");
                  }
                  else
                  {
                    Log(0x10u, 1346, "Engine.cpp", L"Failed to initialize Audio");
                  }
                }
                else
                {
                  Log(0x10u, 1335, "Engine.cpp", L"Failed to initialize RenderManager");
                }
              }
              else
              {
                Log(0x10u, 1308, "Engine.cpp", L"Failed to initialize ResourceManager");
              }
            }
            else
            {
              Log(0x10u, 1294, "Engine.cpp", L"Failed to initialize timekeeping");
            }
          }
          else
          {
            Log(0x10u, 1198, "Engine.cpp", L"Window Creation Failed");
          }
        }
        else
        {
          Log(0x10u, 1132, "Engine.cpp", L"InitOberVFS() Failed");
        }
        CleanupEngine(0);
      }
      v12 = (wchar_t *)(*(int (__thiscall **)(struct IEngineInterface *))(*(_DWORD *)a1 + 16))(a1);
      v13 = LocalizeMessage(v12);
      v14 = FindWindowW(v13, 0);
      LocalFree((HLOCAL)v13);
      if ( v14 )
      {
        if ( IsIconic(v14) )
          ShowWindow(v14, 10);
        BringWindowToTop(v14);
        SetForegroundWindow(v14);
      }
    }
    ExitProcess(0);
  }
  return 0;
}
```

&emsp;&emsp;可以发现，初始化了一个com组件，GUID=e7b2fb72_d728_49b3_a5f2_18ebf5f1349e（感兴趣可以在注册表CLSID中查，是个CGameExplorer类），可以看到最后一个参数获得了一个com对象，调用序列：

```Txt
EngineHandler::GetProjectName  return "MineSweeper.dll"
CoCreateInstance(CGameExplorer)
EngineHandler::GetGdfPath return "\\Microsoft Games\\Minesweeper\\Minesweeper.exe"
CGameExplorer::VerifyAccess               这里会返回空，也是检测什么东西的，需要改返回值为0
```

&emsp;&emsp;改了这2处，就可以运行了！改法有很多，不说了。再来看结构，扫雷用C++写的，用到了XNA，可以找到棋牌类：

```C++
void *__thiscall Board::Board(void *this, int a2, int a3, int a4, int a5, int a6, int a7, int a8, char a9)
{
  void *v9; // esi@1
  unsigned int v10; // eax@1
  int v12; // [sp-14h] [bp-30h]@7
  int v13; // [sp-10h] [bp-2Ch]@7
  unsigned int v14; // [sp-Ch] [bp-28h]@3
  signed int v15; // [sp-8h] [bp-24h]@1
  void *v16; // [sp+Ch] [bp-10h]@1
  int v17; // [sp+18h] [bp-4h]@1

  v9 = this;
  v16 = this;
  v15 = 16;
  *(_DWORD *)this = &Board::`vftable';
  Array<UITile *>::Array<UITile *>(v15);
  *((float *)v9 + 7) = 0.0;
  *((_DWORD *)v9 + 8) = a2;
  *((_DWORD *)v9 + 1) = a5;
  *((_DWORD *)v9 + 2) = a4;
  *((_DWORD *)v9 + 3) = a3;
  *((_DWORD *)v9 + 11) = a6;
  *((_DWORD *)v9 + 9) = a7;
  v17 = 0;
  *((_DWORD *)v9 + 4) = 0;
  *((_DWORD *)v9 + 5) = 0;
  *((_DWORD *)v9 + 6) = 0;
  *((_DWORD *)v9 + 16) = 0;
  *((_DWORD *)v9 + 17) = 0;
  *((_DWORD *)v9 + 10) = a8;
  Board::initTiles((Board *)v9);
  v10 = *((_DWORD *)v9 + 9);
  if ( v10 != -1 && *((_DWORD *)v9 + 10) != -1 )
  {
    v15 = *((_DWORD *)v9 + 10);
    v14 = v10;
    if ( a9 )
    {
      Board::placeMines((Board *)v9, v14, v15);
      ++*((_DWORD *)v9 + 6);
    }
    else
    {
      Board::AttemptReveal((Board *)v9, v14, v15);
    }
  }
  if ( *((_DWORD *)v9 + 1) > *((_DWORD *)v9 + 3) * *((_DWORD *)v9 + 2) - 9 )
  {
    v15 = 1;
    Str::Str(L"Too many mines for tile count");
    StrErr(v12, v13, v14, v15);
    *((_DWORD *)v9 + 1) = *((_DWORD *)v9 + 3) * *((_DWORD *)v9 + 2) - 9;
  }
  return v9;
}
```

继续向上看，来到Game构造函数：

```C++
void *__thiscall Game::Game(void *this)
{
  void *v1; // esi@1
  int v2; // edi@1
  int v3; // eax@1
  int v4; // ST08_4@1
  signed int v5; // eax@1
  int v6; // ST08_4@1

  v1 = this;
  *(_DWORD *)this = &Game::`vftable';
  Array<UITile *>::Array<UITile *>(16);
  Array<UITile *>::Array<UITile *>(16);
  Array<UITile *>::Array<UITile *>(16);
  Array<UITile *>::Array<UITile *>(16);
  Array<UITile *>::Array<UITile *>(16);
  Array<UITile *>::Array<UITile *>(16);
  v2 = (int)((char *)v1 + 152);
  *(_DWORD *)v2 = 0;
  *(_DWORD *)(v2 + 8) = 0;
  *(_DWORD *)(v2 + 12) = 0;
  GameStats::GameStats((char *)v1 + 168);
  Game::G = (Game *)v1;
  *((_DWORD *)v1 + 8) = -1;
  *((_DWORD *)v1 + 9) = -1;
  *((_DWORD *)v1 + 4) = 0;
  *((_DWORD *)v1 + 3) = 0;
  *((_DWORD *)v1 + 2) = 0;
  *((_BYTE *)v1 + 218) = 0;
  *((_DWORD *)v1 + 10) = 2;
  *((_DWORD *)v1 + 12) = 0;
  *((_DWORD *)v1 + 13) = 0;
  *((_BYTE *)v1 + 197) = 0;
  *((_BYTE *)v1 + 195) = 1;
  *((_BYTE *)v1 + 193) = 0;
  *((_BYTE *)v1 + 194) = 0;
  *((_BYTE *)v1 + 196) = 0;
  *((_BYTE *)v1 + 192) = 0;
  *((_DWORD *)v1 + 47) = 0;
  *((_BYTE *)v1 + 24) = 1;
  *((_BYTE *)v1 + 25) = 1;
  *((_BYTE *)v1 + 26) = 1;
  *((_BYTE *)v1 + 27) = 1;
  *((_BYTE *)v1 + 28) = 1;
  *((_BYTE *)v1 + 29) = 1;
  *((_BYTE *)v1 + 30) = 0;
  *((_DWORD *)v1 + 50) = 1;
  v3 = EDifficultyToWidth(1);
  v4 = *((_DWORD *)v1 + 50);
  *((_DWORD *)v1 + 51) = v3;
  v5 = EDifficultyToHeight(v4);
  v6 = *((_DWORD *)v1 + 50);
  *((_DWORD *)v1 + 52) = v5;
  *((_DWORD *)v1 + 53) = EDifficultyToMineCount(v6);
  Game::RandomizeSeedOnTime((Game *)v1);
  CSQMTimeRecorder::SetDataId((CSQMTimeRecorder *)((char *)v1 + 152), 0x17C7u);
  Game::Reset((Game *)v1, 0, 0, 0);
  Game::RequestSetState(v1, 0);
  *((_BYTE *)v1 + 216) = 0;
  *((_BYTE *)v1 + 217) = 0;
  return v1;
}
```

&emsp;&emsp;EDifficultyToWidth和EDifficultyToHeight，分析可知是将默认难度级别转换为雷区阵列的高和宽，同理EDifficultyToMineCount是雷数，那么Game类结构：

```C++
+200	Difficulty	1	2	3
+204	Width 		9	16	16
+208	Height		9	16	30
+212	MineCount	10	40	99
```

再往上层看调用者：

```C++
struct Game *__stdcall Game::SafeGetSingleton()
{
  void *v0; // ecx@2
  Game *v1; // eax@3
  int v3; // [sp-10h] [bp-2Ch]@6
  int v4; // [sp-Ch] [bp-28h]@6
  int v5; // [sp-8h] [bp-24h]@6
  void *v6; // [sp+Ch] [bp-10h]@2
  int v7; // [sp+18h] [bp-4h]@2

  if ( !Game::G )
  {
    v0 = operator new(0xE0u);
    v6 = v0;
    v7 = 0;
    if ( v0 )
      v1 = (Game *)Game::Game(v0);
    else
      v1 = 0;
    v7 = -1;
    Game::G = v1;
    if ( !v1 || (v6 = &v3, Str::Str(dword_10868FC), !(unsigned __int8)Game::initLogic(Game::G, v3, v4, v5)) )
      Game::PrintFatalErrorAndQuit(103);
  }
  return Game::G;
}
```

&emsp;&emsp;可见，Game是个单例类，构造的类指针存在了全局变量Game::G中，因此我们只要读取内存PE获取该结构相对.data段偏移，即可获取全部所需数据（或者调用SafeGetSingleton，只不过要定位.text段基址）。再来看 Game::Reset

```C++
void __thiscall Game::Reset(Game *this, bool a2, bool a3, bool a4)
{
  v4 = this;
  Game::SetTimerEnabled(this, 0);
  v27 = 0;
  if ( !a3 )
  {
    v5 = *((_DWORD *)v4 + 4);
    if ( v5 )
    {
      if ( *(_DWORD *)(v5 + 24) > 0 && *((_DWORD *)Game::G + 10) == 1 )
      {
        v6 = *(_DWORD *)(v5 + 32);
        if ( v6 != 4 )
        {
          v7 = *(float *)(v5 + 28);
          floorf(*(float *)(v5 + 28));
          GameStats::AddNewScore((char *)v4 + 168, v6, (signed int)v7, 0);
          Game::Save(Game::G, 0, 0);
        }
      }
    }
  }
  v8 = *((_DWORD *)v4 + 50);
  *((float *)v4 + 11) = 1.0;
  v23 = v8;
  if ( a3 && (v9 = *((_DWORD *)v4 + 4)) != 0 )
  {
    v23 = *(_DWORD *)(v9 + 32);
    v24 = *(_DWORD *)(v9 + 12);
    v10 = *(_DWORD *)(v9 + 8);
    v11 = *(_DWORD *)(v9 + 4);
    v25 = v10;
  }
  else if ( v8 == 4 )
  {
    v24 = *((_DWORD *)v4 + 51);
    v25 = *((_DWORD *)v4 + 52);
    v11 = *((_DWORD *)v4 + 53);
  }
  else
  {
    v24 = EDifficultyToWidth(v8);
    v25 = EDifficultyToHeight(v8);
    v11 = EDifficultyToMineCount(v8);
  }
  v26 = v11;
  if ( a3 && (v12 = *((_DWORD *)v4 + 4)) != 0 )
  {
    v13 = *(_DWORD *)(v12 + 36);
    v22 = *(_DWORD *)(v12 + 40);
    v20 = *(_DWORD *)(v12 + 44);
    Board::`scalar deleting destructor'((void *)v12, 1);
    *((_DWORD *)v4 + 4) = 0;
    v14 = operator new(0x48u);
    if ( v14 )
      v15 = Board::Board(v14, v23, v24, v25, v26, v20, v13, v22, 1);
    else
      v15 = 0;
    *((_DWORD *)v4 + 4) = v15;
    *((_DWORD *)v15 + 6) = 0;
    Game::RandomizeSeedOnTime(v4);
    v27 = 1;
    v16 = 1;
  }
  else
  {
    v17 = (void *)*((_DWORD *)v4 + 4);
    v16 = 1;
    if ( v17 )
    {
      Board::`scalar deleting destructor'(v17, 1);
      *((_DWORD *)v4 + 4) = 0;
    }
    v21 = operator new(0x48u);
    if ( v21 )
    {
      v18 = _time(0);
      v19 = Board::Board(v21, v23, v24, v25, v26, v18, -1, -1, 0);
    }
    else
    {
      v19 = 0;
    }
    *((_DWORD *)v4 + 4) = v19;
  }
  Game::SaveGameExplorerStatistics(v4);
  *((_BYTE *)v4 + 217) = 0;
  if ( !*((_BYTE *)v4 + 193) || !Game::RandomizeArt(v4, v16) )
  {
    if ( !a2 )
      goto LABEL_33;
    if ( *((_BYTE *)v4 + 27) )
      UIBoardCanvas::SetAllTilesTopAlpha(*((UIBoardCanvas **)v4 + 3), 0);
    Game::ResetCanvas(v4);
  }
  if ( a2 )
  {
    Game::RequestSetState(v4, 1);
    UIBoardCanvas::Refresh(*((UIBoardCanvas **)v4 + 3), 1);
    Game::DoNewBoardAnimation(v4);
  }
LABEL_33:
  if ( v27 )
    UIBoardCanvas::ShowTipMessage(*((UIBoardCanvas **)v4 + 3), L"Restart");
  if ( a4 )
  {
    UserInterface::ProcessMouseMove(g_pUserInterface, v16);
    Engine_ResetTimer();
  }
  if ( a3 )
    *(_DWORD *)(*((_DWORD *)v4 + 4) + 24) = 1;
  *((_BYTE *)v4 + 197) = 0;
}
```

## 初步分析结果

```Txt
Game:sizeof=224
+8		DWORD
+12		UIBoardCanvas *
+16		Board* 
+24		bool
+25		bool
+26		bool
+27		bool
+28		bool
+29		bool
+30		bool
+32		int
+36		int
+40		int state
+48		DWORD
+52		DWORD
+56		Array<UITile *>*
+72		Array<UITile *>*
+88		Array<UITile *>*
+104	Array<UITile *>*
+120	Array<UITile *>*
+136	Array<UITile *>*
+152	CSQMTimeRecorder*
+188	DWORD
+192	bool
+193	bool
+194	bool
+195	bool
+196	bool
+197	bool

+200	Difficulty	
+204	Width 		
+208	Height		
+212	MineCount	
+216	bool
+217	bool
+218	bool IsTimerEnabled

Board:sizeof=72
+4		MineCount
+8		Height
+12		Width
+24		是否挖过雷？
+28		float 
+32		Difficulty	
+36		HitX
+40		HitY
+44
+68		Array<Array<BYTE>>* MineArray 

Array<UITile *>:sizeof=16
+0		MineCount
+4
+8
+12		DWORD[]	MineIndexArray//存储雷位置索引

UITile:
+36		state
```

继续分析：

```C++
void __thiscall Board::placeMines(Board *this, int a2, int a3)
{//布置雷区
  Board *v3; // esi@1
  int v4; // eax@1
  int i; // edi@4
  signed int v6; // ecx@5
  int v7; // eax@9
  int v8; // edi@10
  unsigned int v9; // ebx@13
  unsigned int v10; // ecx@15
  signed int v11; // ebx@16
  int v12; // eax@16
  int v13; // edx@16
  unsigned int Seed; // [sp+Ch] [bp-8h]@1
  int pv; // [sp+10h] [bp-4h]@2

  v3 = this;
  Seed = GetRandomSeed();
  SetRandomSeed(*((_DWORD *)v3 + 11));
  v4 = (int)operator new(0x10u);
  if ( v4 )
    pv = Array<UITile *>::Array<UITile *>(v4, 16);
  else
    pv = 0;
  for ( i = 0; i < *((_DWORD *)v3 + 3) * *((_DWORD *)v3 + 2); ++i )
  {
    v6 = *((_DWORD *)v3 + 3);
    if ( (i % v6 - a2) * (2 * (i % v6 - a2 >= 0) - 1) > 1 || (i / v6 - a3) * (2 * (i / v6 - a3 >= 0) - 1) > 1 )//取第一次点击位置以外的雷区位置，存到数组array1里，这样设计就不会导致第一次就点到雷
      Array<NodeBase *>::Add(pv, i);
  }
  v7 = (int)operator new(0x10u);
  if ( v7 )
    v8 = Array<UITile *>::Array<UITile *>(v7, *((_DWORD *)v3 + 1));
  else
    v8 = 0;
  while ( *(_DWORD *)v8 != *((_DWORD *)v3 + 1) && *(_DWORD *)pv )
  {//若雷数不够且array1没被用完就一直布雷
    v9 = GetRandom(0, *(_DWORD *)pv - 1);//rand(0,array1.size()-1)
    Array<NodeBase *>::Add(v8, *(_DWORD *)(*(_DWORD *)(pv + 12) + 4 * v9));//在array2中增加该位置
    Array<NodeBase *>::Remove(pv, v9);//从array1中去除该位置
  }
  v10 = 0;
  if ( *(_DWORD *)v8 )
  {
    do
    {
      v11 = *((_DWORD *)v3 + 3);
      v12 = *(_DWORD *)(*(_DWORD *)(v8 + 12) + 4 * v10) / v11;
      v13 = *(_DWORD *)(*(_DWORD *)(v8 + 12) + 4 * v10++) % v11;//分别还原横纵坐标
      *(_BYTE *)(v12 + *(_DWORD *)(*(_DWORD *)(*(_DWORD *)(*((_DWORD *)v3 + 17) + 12) + 4 * v13) + 12)) = 1;//可以推测出将布雷坐标数组存到Array<Array<BYTE>>二维数组里
    }
    while ( v10 < *(_DWORD *)v8 );
  }
  Array<int>::`scalar deleting destructor'((void *)v8, 1);
  if ( pv )
    Array<int>::`scalar deleting destructor'((void *)pv, 1);
  SetRandomSeed(Seed);
}

int __thiscall Board::AttemptReveal(Board *this, unsigned int a2, unsigned int a3)
{
  Board *v3; // esi@1
  int v4; // eax@1
  int v5; // eax@6
  unsigned int *v7; // [sp+0h] [bp-10h]@0
  int v8; // [sp+Ch] [bp-4h]@1

  v3 = this;
  v4 = *(_DWORD *)(*(_DWORD *)(*(_DWORD *)(*(_DWORD *)(*((_DWORD *)this + 16) + 12) + 4 * a2) + 12) + 4 * a3);
  v8 = 0;
  if ( v4 == 9 || v4 == 11 )
  {
    if ( *((_DWORD *)this + 6) )
    {
      if ( *(_BYTE *)(a3 + *(_DWORD *)(*(_DWORD *)(*(_DWORD *)(*((_DWORD *)this + 17) + 12) + 4 * a2) + 12)) )
      {
        v8 = 0;
LABEL_11:
        ++*((_DWORD *)v3 + 6);
        goto LABEL_12;
      }
      v5 = Board::revealAt(this, a2, a3, 0, a2, a3, 0);
    }
    else
    {
      Board::placeMines(this, a2, a3);//如果是首次挖雷，则布雷
      v5 = Board::revealAt(v3, a2, a3, 0, a2, a3, 0);
      *((_DWORD *)v3 + 9) = a2;
      *((_DWORD *)v3 + 10) = a3;//存储点击位置
    }
    v8 = v5;
    goto LABEL_11;
  }
  if ( *((_BYTE *)Game::G + 24) )
    GameAudio::PlaySoundProto(0, 0, 0, v7);
LABEL_12:
  if ( *((_DWORD *)v3 + 12) <= 0u )
    UIBoardCanvas::Refresh(*((UIBoardCanvas **)Game::G + 3), 1);
  return v8;
}
```

&emsp;&emsp;至此，想得到的结果已经都得到。这里只研究了很普通的一些属性，然而代码中却有很多可以挖掘的地方，请自行研究！

## 外挂开发

```C++
template<typename T>
struct Array
{
	DWORD MineCount;
	DWORD unused1;
	DWORD unused2;
	T* data;
};

#include <TlHelp32.h>
void CWin7MineSweeperHackerDlg::OnBnClickedOk()
{
	// TODO: 在此添加控件通知处理程序代码
	UpdateData(TRUE);
	m_output="";
	PROCESSENTRY32 pe;
	pe.dwSize=sizeof(PROCESSENTRY32);
	HANDLE hProcess=NULL;
	HANDLE hSnapshot=CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS,0);
	if(hSnapshot == INVALID_HANDLE_VALUE || !Process32First(hSnapshot,&pe))
	{
		AfxMessageBox("ERROR!");
		return;
	}
	do
	{
		if(StrStr(pe.szExeFile,"MineSweeper.exe"))
		{
			hProcess=OpenProcess(PROCESS_ALL_ACCESS,FALSE,pe.th32ProcessID);
			break;
		}
	}
	while(Process32Next(hSnapshot,&pe));
	CloseHandle(hSnapshot);
	if(hProcess == NULL)
	{
		AfxMessageBox("找不到进程！");
		return;
	}

	MODULEENTRY32 me;
	me.dwSize=sizeof(MODULEENTRY32);
	LPVOID peBase=NULL;
	DWORD peSize;
	hSnapshot=CreateToolhelp32Snapshot(TH32CS_SNAPMODULE,pe.th32ProcessID);
	if(hSnapshot == NULL || !Module32First(hSnapshot,&me))
	{
		AfxMessageBox("ERROR");
		return;
	}

	do 
	{
		if(StrStrI(me.szModule,"mine"))
		{
			peBase=(LPVOID)me.modBaseAddr;
			peSize=me.modBaseSize;
			break;
		}
	} while (Module32Next(hSnapshot,&me));
	CloseHandle(hSnapshot);

	if(peBase == NULL)
	{		
		AfxMessageBox("ERROR");
		return;
	}

	BYTE* buffer=new BYTE[peSize];
	DWORD readnum;
	ReadProcessMemory(hProcess,peBase,buffer,peSize,&readnum);
	IMAGE_DOS_HEADER* dosheader=(IMAGE_DOS_HEADER*)buffer;
	IMAGE_NT_HEADERS* ntheader=(IMAGE_NT_HEADERS*)(buffer+dosheader->e_lfanew);
	IMAGE_SECTION_HEADER* cur=(IMAGE_SECTION_HEADER*)(ntheader+1);
	for(int i=0;i<ntheader->FileHeader.NumberOfSections;i++)
	{
		if(StrStrI((CHAR*)cur->Name,".data"))
		{
			break;
		}
		cur++;
	}
	struct Board;
	struct Game
	{
		DWORD unused[4];
		struct Board* board;
	};
	struct Board
	{
		DWORD unused1;
		DWORD MineCount;
		DWORD Height;
		DWORD Width;
		DWORD unused2[4];
		DWORD Difficulty;
		DWORD HitX;
		DWORD HitY;
		DWORD unused3[6];
		Array< Array<BYTE> >* MineArray;
	};
	
	BYTE* GameInThatProc = (BYTE*)*(Game**)(buffer + cur->VirtualAddress + 0x88B4);
	//换算成本地地址
	Game G;
	if (GameInThatProc == NULL)
	{
		AfxMessageBox("数据有误");
		goto end1;
	}
	ReadProcessMemory(hProcess, GameInThatProc, &G, sizeof(G), &readnum);
	//换算成本地地址
	Board B;
	if (G.board == NULL)
	{
		AfxMessageBox("数据有误");
		return;
	}
	ReadProcessMemory(hProcess, G.board, &B, sizeof(B), &readnum);
	Array< Array<BYTE> > MA;
	if (B.MineArray == NULL)
	{
		AfxMessageBox("数据有误");
		goto end1;
	}
	ReadProcessMemory(hProcess, B.MineArray, &MA, sizeof(MA), &readnum);
	typedef Array<BYTE>* pMAsub;//先取指针数组
	pMAsub* data1 = new pMAsub[B.Width];
	if (MA.data == NULL)
	{
		AfxMessageBox("数据有误");
		goto end1;
	}
	ReadProcessMemory(hProcess, MA.data, data1, sizeof(Array<BYTE>*) * B.Width, &readnum);

	for(int i=0;i<B.Width;i++)
	{
		Array<BYTE> data2;//对指针数组每个指针找到对象内存
		ReadProcessMemory(hProcess, data1[i], &data2, sizeof(data2), &readnum);
		if (data2.data == NULL)
		{
			AfxMessageBox("数据有误");
			goto end1;
		}
		BYTE* data3 = new BYTE[B.Height];
		ReadProcessMemory(hProcess, data2.data, data3, sizeof(BYTE) * B.Height, &readnum);

		for(int j=0;j<B.Height;j++)
		{
			if(data3[j] == 1)//雷
			{
				m_output += "雷";
			}
			else
			{
				m_output += "空";
			}
		}
		delete[]data3;
		m_output += "\r\n";
	}

	delete[]data1;

end1:
	delete []buffer;
	CloseHandle(hProcess);
	UpdateData(FALSE);
}
```

## 结论

```Txt
新开一局扫雷，可以发现程序获取结果：
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空
 空空空空空空空空空

 可见，没点击前，是不布雷的，和之前分析的一样
 点击以后：
 空雷空空空空空空空
 空雷空空空空空空空
 空空空空空空空空空
 空空雷空雷空空空空
 空空雷空空空空空空
 空空空空空空空空空
 雷雷空空空空空空空
 空空雷空空空空空空
 雷空空空雷空空空空
```

 特别注意，，由于程序中使用的数据结构原因，得到的结果，和扫雷屏幕上显示，关于y=x对称！！！

