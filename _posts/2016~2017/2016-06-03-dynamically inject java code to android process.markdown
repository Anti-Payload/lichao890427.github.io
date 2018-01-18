---
layout: post
title: 动态注入并执行Android代码到进程
categories: Android
description: 动态注入并执行Android代码到进程
keywords: Inject
---

# 动态注入并执行Android代码到进程

## 背景

&emsp;&emsp;网上可以见到很多帖子是如何注入进程并钩取c层api的例子，而动态的让目标进程加载一个dex并执行这个dex却没看到，c层的注入很简单，可以参考adbi，ptrace进去修改当前指令指针寄存器，构造一个dlopen的指令并调用，类似于win上getcontext修改eip， java层稍复杂一些，加载dex->解析dex->查找目标类->调用函数,下面是测试用例,我们的目标是打log

## 代码

```C++
import android.util.Log;
public class injectjava {
    static void initialize(){
        Log.i("INJECTJAVA","injectava called");
    }
}

#define LOGI(...)  __android_log_print(ANDROID_LOG_ERROR, "INJECTJAVA", __VA_ARGS__)
#define DEXNAME "/data/local/tmp/inject.dex"

typedef void Object;
typedef void ClassObject;
typedef void Thread;
typedef uint8_t u1;
typedef uint32_t u4;
struct JNIEnvExt
{
    const struct JNINativeInterface* funcTable;
    const struct JNINativeInterface* baseFuncTable;
    u4      envThreadId;
    Thread* self;
};

typedef void DexOptHeader;
typedef void DexStringId;
typedef void DexTypeId;
typedef void DexFieldId;
typedef void DexMethodId;
typedef void DexProtoId;
typedef void DexLink;
typedef void DexClassLookup;

struct DexHeader
{
    u1  magic[8];           /* includes version number */
    u4  checksum;           /* adler32 checksum */
    u1  signature[20];       /* SHA-1 hash */
    u4  fileSize;           /* length of entire file */
    u4  headerSize;         /* offset to start of next section */
    u4  endianTag;
    u4  linkSize;
    u4  linkOff;
    u4  mapOff;
    u4  stringIdsSize;
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize;
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};

struct DexClassDef
{
    u4  classIdx;           /* index into typeIds for this class */
    u4  accessFlags;
    u4  superclassIdx;      /* index into typeIds for superclass */
    u4  interfacesOff;      /* file offset to DexTypeList */
    u4  sourceFileIdx;      /* index into stringIds for source file name */
    u4  annotationsOff;     /* file offset to annotations_directory_item */
    u4  classDataOff;       /* file offset to class_data_item */
    u4  staticValuesOff;    /* file offset to DexEncodedArray */
};

struct DexFile
{
    DexOptHeader *pOptHeader;
    DexHeader *pHeader;
    DexStringId *pStringIds;
    DexTypeId *pTypeIds;
    DexFieldId *pFieldIds;
    DexMethodId *pMethodIds;
    DexProtoId *pProtoIds;
    DexClassDef *pClassDefs;
    DexLink *pLinkData;
    DexClassLookup *pClassLookup;
};

struct DvmDex
{
    DexFile*            pDexFile;
};

int (*dvmDexFileOpenPartial)(const void* addr, int len, DvmDex** ppDvmDex)=0;
int (*dexSwapAndVerify)(void* addr, int len)=0;
JNIEnv* (*dvmGetJNIEnvForThread)()=0;
Object* (*dvmDecodeIndirectRef)(JNIEnv* env, jobject jobj)=0;
Object* (*dvmDecodeIndirectRef_)(Thread* self, jobject jobj)=0;
ClassObject* (*dvmDefineClass)(DvmDex* pDvmDex, const char* descriptor, Object* classLoader)=0;
DexClassLookup* (*dexCreateClassLookup)(DexFile* pDexFile)=0;

void inject()
{
    /*
     * C层hook
     */
    /*
     * JAVA层hook
     */
    //获取关键函数指针
    do
    {
        void* hdvm = dlopen("libdvm.so", RTLD_LAZY | RTLD_GLOBAL);
        if(!hdvm)
        {
            LOGI("libdvm not exist!");
            break;
        }
        dvmDexFileOpenPartial = (typeof(dvmDexFileOpenPartial))dlsym(hdvm, "dvmDexFileOpenPartial");
        if(!dvmDexFileOpenPartial)
        {
            dvmDexFileOpenPartial = (typeof(dvmDexFileOpenPartial))dlsym(hdvm, "_Z21dvmDexFileOpenPartialPKviPP6DvmDex");
            if(!dvmDexFileOpenPartial)
                LOGI("dvmDexFileOpenPartial not exist!");
        }
        dexSwapAndVerify = (typeof(dexSwapAndVerify))dlsym(hdvm, "dexSwapAndVerify");
        if(!dexSwapAndVerify)
        {
            dexSwapAndVerify = (typeof(dexSwapAndVerify))dlsym(hdvm, "_Z16dexSwapAndVerifyPhi");
            if(!dexSwapAndVerify)
                LOGI("dexSwapAndVerify not exist!");
        }
        dvmGetJNIEnvForThread = (typeof(dvmGetJNIEnvForThread))dlsym(hdvm, "dvmGetJNIEnvForThread");
        if(!dvmGetJNIEnvForThread)
        {
            dvmGetJNIEnvForThread = (typeof(dvmGetJNIEnvForThread))dlsym(hdvm, "_Z21dvmGetJNIEnvForThreadv");
            if(!dvmGetJNIEnvForThread)
                LOGI("dvmGetJNIEnvForThread not exist!");
        }
        dvmDecodeIndirectRef = (typeof(dvmDecodeIndirectRef))dlsym(hdvm, "dvmDecodeIndirectRef");
        if(!dvmDecodeIndirectRef)
        {
            dvmDecodeIndirectRef = (typeof(dvmDecodeIndirectRef))dlsym(hdvm, "_Z20dvmDecodeIndirectRefP7_JNIEnvP8_jobject");
            if(!dvmDecodeIndirectRef)
            {
                dvmDecodeIndirectRef_ = (typeof(dvmDecodeIndirectRef_))dlsym(hdvm, "_Z20dvmDecodeIndirectRefP6ThreadP8_jobject");
                if(!dvmDecodeIndirectRef_)
                    LOGI("dvmDecodeIndirectRef_ not exist!");
            }
        }
        dvmDefineClass = (typeof(dvmDefineClass))dlsym(hdvm, "dvmDefineClass");
        if(!dvmDefineClass)
        {
            dvmDefineClass = (typeof(dvmDefineClass))dlsym(hdvm, "_Z14dvmDefineClassP6DvmDexPKcP6Object");
            if(!dvmDefineClass)
                LOGI("dvmDefineClass not exist!");
        }
        dexCreateClassLookup = (typeof(dexCreateClassLookup))dlsym(hdvm, "dexCreateClassLookup");
        if(!dexCreateClassLookup)
        {
            dexCreateClassLookup = (typeof(dexCreateClassLookup))dlsym(hdvm, "_Z20dexCreateClassLookupP7DexFile");
            if(!dexCreateClassLookup)
                LOGI("dexCreateClassLookup not exist!");
        }
        if(!dvmDexFileOpenPartial || !dexSwapAndVerify || !dvmGetJNIEnvForThread || !dvmDefineClass || !dexCreateClassLookup || (!dvmDecodeIndirectRef && !dvmDecodeIndirectRef_))
            break;
        LOGI("InitOk");
        //读取dex以便动态加载
        char* dexdata = 0;
        int dexlen;
        int dexfd = open(DEXNAME, O_RDONLY);
        if(dexfd == -1)
        {
            LOGI("can't open inject.dex!");
            break;
        }
        struct stat st = {0};
        fstat(dexfd , &st);
        if(st.st_size)
        {
            dexlen = st.st_size;
            dexdata = (char*)malloc(st.st_size);
            if(dexdata)
                read(dexfd, dexdata, st.st_size);
        }
        close(dexfd);
        if(!dexdata)
        {
            LOGI("read inject.dex fail!");
            break;
        }
        int result;
        DvmDex* dvmDex = 0;
        JNIEnv* env =0;
        jclass java_lang_Class =0;
        jmethodID java_lang_Class_forName =0;
        jclass java_lang_ClassLoader =0;
        jmethodID java_lang_ClassLoader_getSystemClassLoader =0;
        jobject jSystemClassLoader =0;
        Object* SystemClassLoader =0;
        jstring com_injectjava_str =0;
        jclass com_injectjava =0;
        jmethodID com_injectjava_initialize =0;
        DexFile* pDexFile = 0;
        int csize,i;
        dexSwapAndVerify(dexdata, dexlen);
        result = dvmDexFileOpenPartial(dexdata, dexlen, &dvmDex);
        if(result != 0)
        {
            LOGI("dvmDexFileOpenPartial fail!");
            goto END1;
        }
        env = dvmGetJNIEnvForThread();
        if(!env)
        {
            LOGI("dvmGetJNIEnvForThread fail!");
            goto END1;
        }
        java_lang_Class = env->FindClass("java/lang/Class");
        if(!java_lang_Class)
        {
            LOGI("java_lang_Class null!");
            goto END1;
        }
        java_lang_Class_forName = env->GetStaticMethodID(java_lang_Class, "forName", "(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;");
        if(!java_lang_Class_forName)
        {
            LOGI("java_lang_Class_forName null!");
            goto END1;
        }
        java_lang_ClassLoader = env->FindClass("java/lang/ClassLoader");
        if(!java_lang_ClassLoader)
        {
            LOGI("java_lang_ClassLoader null!");
            goto END1;
        }
        java_lang_ClassLoader_getSystemClassLoader = env->GetStaticMethodID(java_lang_ClassLoader, "getSystemClassLoader", "()Ljava/lang/ClassLoader;");
        if(!java_lang_ClassLoader_getSystemClassLoader)
        {
            LOGI("java_lang_ClassLoader_getSystemClassLoader null!");
            goto END1;
        }
        jSystemClassLoader = env->CallStaticObjectMethod(java_lang_ClassLoader, java_lang_ClassLoader_getSystemClassLoader, "()Ljava/lang/ClassLoader;");
        if(!jSystemClassLoader)
        {
            LOGI("jSystemClassLoader null!");
            goto END1;
        }
        LOGI("InitOk1");
        if(dvmDecodeIndirectRef)
            SystemClassLoader = dvmDecodeIndirectRef(env, jSystemClassLoader);
        else
            SystemClassLoader = dvmDecodeIndirectRef_(((JNIEnvExt*)env)->self, jSystemClassLoader);
        if(!SystemClassLoader)
        {
            LOGI("SystemClassLoader null!");
            goto END1;
        }
        pDexFile = dvmDex->pDexFile;
        pDexFile->pClassLookup = dexCreateClassLookup(dvmDex->pDexFile);
        csize = pDexFile->pHeader->classDefsSize;
        for(i=0;i<csize;i++)
        {
            pDexFile->pClassDefs[i].accessFlags | 0x10000;
        }
        dvmDefineClass(dvmDex, "Lcom/injectjava;", SystemClassLoader);
        com_injectjava_str = env->NewStringUTF("com.injectjava");
        if(!com_injectjava_str)
        {
            LOGI("com_injectjava null!");
            goto END1;
        }
        com_injectjava = (jclass)env->CallStaticObjectMethod(java_lang_Class, java_lang_Class_forName, com_injectjava_str, true, jSystemClassLoader);
        if(!com_injectjava)
        {
            LOGI("com_injectjava null!");
            goto END1;
        }
        com_injectjava_initialize = env->GetStaticMethodID(com_injectjava, "initialize", "()V");
        if(!com_injectjava_initialize)
        {
            LOGI("com_injectjava_initialize null!");
            goto END1;
        }
        env->CallStaticVoidMethod(com_injectjava, com_injectjava_initialize);
        LOGI("InitOk2");
        END1:
        free(dexdata);
    }
    while(false);
}
```

## 结论

&emsp;&emsp;将要执行的dex放到/data/local/tmp/inject.dex即可

```Txt
06-03 15:47:34.984 16197-16197/com.example.hellojni E/INJECTJAVA: InitOk
06-03 15:47:35.074 16197-16197/com.example.hellojni E/INJECTJAVA: InitOk1
06-03 15:54:08.304 16197-16197/com.example.hellojni I/INJECTJAVA: injectava called
```

&emsp;&emsp;***这里只是做实验验证Java代码的注入原理分析，Frida注入框架是这类动态注入的最佳实践框架***