---
layout: post
title: 开发你自己的su
categories: Android
description: 开发你自己的su
keywords: android
---

# 开发你自己的su

## 背景

&emsp;&emsp;市面上常见的Android Root工具，原理就是利用系统漏洞将su文件下载到可执行目录。这样第三方进程可以执行su提升子进程的权限（记住Root提权不是提升当前进程权限，而是给子进程提权）。比如在app中读取文件可以用命令，`su -c cat /data/data/com.example/xxx`，当前app进程并未提权，而是cat子进程提权，所以读取到了其他App-com.example的文件。   
&emsp;&emsp;上面介绍了Root原理，那么很多Root工具除了su，还会有对应的授权管理app存在，它们记录一个白名单，决定哪个app拥有权限，一般有3种权限：允许/询问/拒绝。询问的时候，第三方app调用su后，su会和授权管理app通信，导致系统弹窗询问用户是否允许。这个在有时候是很讨厌的，那么怎么去除这个弹窗呢，这就必须我们自己编译一个su，并设置SUID位。SUID位确保su执行的子进程以Root权限运行  
&emsp;&emsp;这里介绍的是绝对可行的root方式，对于android模拟器，无论x86 x64 ，无论4.0 5.0 6.0都适用。真机的话5.0和6.0，root成功率比较低，因此不适合在上面做root方面的测试，这时只能考虑虚拟机，因为虚拟机adb有root权限，利用adb root权限便可以提升su权限。网上盛传的方法是http://androidsu.com/superuser https://www.0xaa55.com/forum.php ... tid=1648&extra=
需要superuser.apk，我分析了该网站的su，发现除了传统su的流程(setuid setgid)外，还加入了和superuser.apk通信的这一步。superuser.apk用于单个app权限设置。遗憾的是该作者并没完成mips架构，和5.0以上的su适配  
&emsp;&emsp;默认的su，是只允许root权限和shell权限来执行的，因此一般的app并不能使用，因此这也是我们修改的重点，同时还要给su加上必要的环境变量，我这种方式，去除了权限判断部分，因此减少了某些情况下由于这部分带来的root权限未成功获取的bug。这里提供一种方式，可以支持全平台

## 代码

```C++
#include <errno.h>
#include <getopt.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pwd.h>

#define AID_ROOT             0  /* traditional unix root user */
#define AID_SYSTEM        1000  /* system server */
#define AID_SHELL         2000  /* adb and debug shell user */
#define AID_NOBODY        9999
#define AID_APP          10000  /* first app user */
#define AID_USER        100000  /* offset for uid ranges for each user */


void pwtoid(const char* tok, uid_t* uid, gid_t* gid) {
    struct passwd* pw = getpwnam(tok);
    if (pw)
    {
        if (uid) *uid = pw->pw_uid;
        if (gid) *gid = pw->pw_gid;
    }
    else
    {
        char* end;
        errno = 0;
        uid_t tmpid = strtoul(tok, &end, 10);
        if (errno != 0 || end == tok)
                printf("invalid uid/gid '%s'", tok);
        if (uid) *uid = tmpid;
        if (gid) *gid = tmpid;
    }
}

void extract_uidgids(const char* uidgids, uid_t* uid, gid_t* gid, gid_t* gids, int* gids_count) {
    char *clobberablegids;
    char *nexttok;
    char *tok;
    int gids_found;

    if (!uidgids || !*uidgids)
    {
        *gid = *uid = 0;
        *gids_count = 0;
        return;
    }

    clobberablegids = strdup(uidgids);
    strcpy(clobberablegids, uidgids);
    nexttok = clobberablegids;
    tok = strsep(&nexttok, ",");
    pwtoid(tok, uid, gid);
    tok = strsep(&nexttok, ",");
    if (!tok)
    {
        /* gid is already set above */
        *gids_count = 0;
        free(clobberablegids);
        return;
    }
    pwtoid(tok, NULL, gid);
    gids_found = 0;
    while ((gids_found < *gids_count) && (tok = strsep(&nexttok, ",")))
    {
        pwtoid(tok, NULL, gids);
        gids_found++;
        gids++;
    }
    if (nexttok && gids_found == *gids_count)
    {
        fprintf(stderr, "too many group ids\n");
    }
    *gids_count = gids_found;
    free(clobberablegids);
}

int main(int argc, char** argv)
{
    uid_t current_uid = getuid();

    // Handle -h and --help.
    ++argv;
    if (*argv && (strcmp(*argv, "--help") == 0 || strcmp(*argv, "-h") == 0))
    {
        fprintf(stderr,
                "usage: su [UID[,GID[,GID2]...]] [COMMAND [ARG...]]\n"
                "\n"
                "Switch to WHO (default 'root') and run the given command (default sh).\n"
                "\n"
                "where WHO is a comma-separated list of user, group,\n"
                "and supplementary groups in that order.\n"
                "\n");
        return 0;
    }

    uid_t uid = 0;
    gid_t gid = 0;

    if (*argv)
    {
        gid_t gids[10];
        int gids_count = sizeof(gids)/sizeof(gids[0]);
        extract_uidgids(*argv, &uid, &gid, gids, &gids_count);
        if (gids_count) {
            if (setgroups(gids_count, gids))
            {
                printf("setgroups failed");
            }
        }
        ++argv;
    }

    if (setgid(gid))
            printf("setgid failed");
    if (setuid(uid))
            printf("setuid failed");

    //设置环境变量
    setenv("PATH", "/sbin:/vendor/bin:/system/sbin:/system/bin:/system/xbin", 1);
    setenv("LD_LIBRARY_PATH", "/vendor/lib:/system/lib", 1);
    setenv("ANDROID_BOOTLOGO", "1", 1);
    setenv("ANDROID_ROOT", "/system", 1);
    setenv("ANDROID_DATA", "/data", 1);
    setenv("ANDROID_ASSETS", "/system/app", 1);
    setenv("EXTERNAL_STORAGE", "/sdcard", 1);
    setenv("ASEC_MOUNTPOINT", "/mnt/asec", 1);
    setenv("LOOP_MOUNTPOINT", "/mnt/obb", 1);
    char* exec_args[argc + 1];
    size_t i = 0;
    for (; *argv != NULL; ++i)
    {
      exec_args[i] = *argv++;
    }

    if (i == 0) exec_args[i++] = "/system/bin/sh";
    exec_args[i] = NULL;

    execvp(exec_args[0], exec_args);
    printf("failed to exec %s", exec_args[0]);
}
```

相关代码我的github上也有

