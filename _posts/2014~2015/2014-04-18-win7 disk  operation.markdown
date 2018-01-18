---
layout: post
title: win7磁盘读写操作
categories: C/C++
description: win7磁盘读写操作
keywords: 
---

## 直接放代码
&emsp;&emsp;正常情况下，Win7系统16扇区以后写操作是无效的，如果要破解这个通常会在驱动级实现，然而这里提供一种应用层实现方式：
```C++
HANDLE handle=CreateFile("\\\\.\\c:",GENERIC_ALL,FILE_SHARE_READ|FILE_SHARE_WRITE,NULL,OPEN_EXISTING,FILE_ATTRIBUTE_NORMAL,NULL);
//GENERIC_ALL有时不好使，不过权限最大，如果用GENERIC_ALL返回INVALID_HANDLE则改成GENERIC_READ|GENERIC_WRITE
BOOL ret;
ret=SetFilePointer(handle,0x0000CC00,NULL,FILE_BEGIN);
BYTE buffer[512];
memset(buffer,1,512);
DWORD num;
ret=DeviceIoControl(handle,FSCTL_DISMOUNT_VOLUME,NULL,0,NULL,0,&num,NULL);//注意FSCTL_LOCK_VOLUME无效
//下面要写数据了，注意别把自己的数据写了，自己修改上面的偏移
ret=WriteFile(handle,buffer,512,&num,NULL);//必须是扇区大小的倍数，否则报参数错
CloseHandle(handle);

&emsp;&emsp;当非Administrator和Administrators组的用户访问CreateFile时会失败，这时需要用到提权函数，下面是逆向出的unlocker的一部分代码用于提权

```C++
bool SetACL(LPWSTR OjbectName)
{
    PSID pEveryoneSID=NULL,pAdminSID=NULL;
    SID_IDENTIFIER_AUTHORITY SIDAuthWorld=SECURITY_WORLD_SID_AUTHORITY;
    if(!AllocateAndInitializeSid(&SIDAuthWorld,1,SECURITY_WORLD_RID,0,0,0,0,0,0,0,&pEveryoneSID))
    {//ERROR
        if(pEveryoneSID) FreeSid(pEveryoneSID);
        return FALSE;
    }
    SID_IDENTIFIER_AUTHORITY SIDAuthNT=SECURITY_NT_AUTHORITY;
    if(!AllocateAndInitializeSid(&SIDAuthNT,2,SECURITY_BUILTIN_DOMAIN_RID,DOMAIN_ALIAS_RID_ADMINS,
        0,0,0,0,0,0,&pAdminSID))
    {
        if(pAdminSID) FreeSid(pAdminSID);
        if(pEveryoneSID) FreeSid(pEveryoneSID);
        return FALSE;
    }
	
	EXPLICIT_ACCESS ea[2];
	ZeroMemory(ea,2*sizeof(EXPLICIT_ACCESS));
	ea[0].grfAccessPermissions=GENERIC_ALL;
	ea[0].grfAccessMode=SET_ACCESS;
	ea[0].grfInheritance=NO_INHERITANCE;
	ea[0].Trustee.TrusteeForm=TRUSTEE_IS_SID;
	ea[0].Trustee.TrusteeType=TRUSTEE_IS_WELL_KNOWN_GROUP;
	ea[0].Trustee.ptstrName=(LPSTR)pEveryoneSID;
	ea[1].grfAccessPermissions=GENERIC_ALL;
	ea[1].grfAccessMode=SET_ACCESS;
	ea[1].grfInheritance=NO_INHERITANCE;
	ea[1].Trustee.TrusteeForm=TRUSTEE_IS_SID;
	ea[1].Trustee.TrusteeType=TRUSTEE_IS_GROUP;
	ea[1].Trustee.ptstrName=(LPSTR)pAdminSID;
	
	PACL pACL=NULL;
	if(!SetEntriesInAcl(2,ea,NULL,&pACL))
	{
		DWORD nRet=SetNamedSecurityInfoW(OjbectName,SE_FILE_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pACL,NULL);
		if(nRet == ERROR_ACCESS_DENIED)
		{
			SetNamedSecurityInfoW(OjbectName,SE_FILE_OBJECT,OWNER_SECURITY_INFORMATION,pAdminSID,NULL,NULL,NULL);
			SetNamedSecurityInfoW(OjbectName,SE_FILE_OBJECT,DACL_SECURITY_INFORMATION,NULL,NULL,pACL,NULL);
		}
	}
	if(pACL) LocalFree(pACL);
	if(pAdminSID) FreeSid(pAdminSID);
	if(pEveryoneSID) FreeSid(pEveryoneSID);
	return 0;
}
```

## 注意事项
&emsp;&emsp;read和write的时候一般是512的倍数，读写大小不能超过系统最大粒度，通常是64M，MSDN上未提示这个