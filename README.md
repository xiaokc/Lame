# Lame
android studio 编译 mp3lame，生成.so库
android 项目中需要使用mp3lame native方法，需要自己编译.so库，在android studio中新建一个ndk项目。


##步骤如下：
### 安装NDK
- 点菜单栏的File->ProjectStructure…->在打开的窗口中左侧选中SDKLocation->在右侧Android NDK Location中填入NDK目录所在路径，如下图所示：
- 配置ndk环境变量：在~/.bash_profile或者/etc/.bashrc中添加

```
export ANDROID_NDK_HOME=/Users/xiaokecong/Android/android-sdk-macosx/ndk-bundle
export PATH=$PATH:$ANDROID_NDK_HOME
```
### 项目中配置jni
- 项目切换至project模式
- 在app/src/main/下新建目录jni
- 在app module的build.gradle中 android field添加：
```
 sourceSets {
        main {
            jni.srcDirs = []
            jniLibs.srcDir 'src/main/libs'
        }
    }
```
### 下载mp3lame源码：
http://lame.sourceforge.net/download.php

http://lame.sourceforge.net/download.php
- 解压mp3lame.tar.gz  ->  lame-3.99.5
- 在jni目录下新建目录：lame-3.99.5_libmp3lame
- 将解压出来lame-3.99.5/libmp3lame/下面的所有.c和.h拷贝到项目jni/lame-3.99.5_libmp3lame 目录下,同时将lame-3.99.5/include/下的lame.h也拷贝到jni/lame-3.99.5_libmp3lame 目录下

### 修改.c和.h代码
- 删除fft.c文件47行的"include "vector/lame_intrin.h""
- 删除set_get.h文件的24行的"#include <lame.h>"
- 将util.h文件的574行的"extern ieee754_float32_t fast_log2(ieee754_float32_t x);" 替换为 "extern float fast_log2(float x);"

###在jni目录下新建Application.mk和Android.mk
- Application.mk中：
```
APP_ABI := armeabi armeabi-v7a arm64-v8a x86 x86_64
APP_MODULES := mp3lame
APP_CFLAGS += -DSTDC_HEADERS
#APP_ABI:=x86_64
#APP_PLATFORM := android-21
```
注意：
如果是x86_64的话需要在Application.mk中加上
APP_CFLAGS += -DSTDC_HEADERS

- Android.mk中：
```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LAME_LIBMP3_DIR := lame-3.99.5_libmp3lame


LOCAL_MODULE    := mp3lame

LOCAL_SRC_FILES :=\
$(LAME_LIBMP3_DIR)/bitstream.c \
$(LAME_LIBMP3_DIR)/fft.c \
$(LAME_LIBMP3_DIR)/id3tag.c \
$(LAME_LIBMP3_DIR)/mpglib_interface.c \
$(LAME_LIBMP3_DIR)/presets.c \
$(LAME_LIBMP3_DIR)/quantize.c \
$(LAME_LIBMP3_DIR)/reservoir.c \
$(LAME_LIBMP3_DIR)/tables.c  \
$(LAME_LIBMP3_DIR)/util.c \
$(LAME_LIBMP3_DIR)/VbrTag.c \
$(LAME_LIBMP3_DIR)/encoder.c \
$(LAME_LIBMP3_DIR)/gain_analysis.c \
$(LAME_LIBMP3_DIR)/lame.c \
$(LAME_LIBMP3_DIR)/newmdct.c \
$(LAME_LIBMP3_DIR)/psymodel.c \
$(LAME_LIBMP3_DIR)/quantize_pvt.c \
$(LAME_LIBMP3_DIR)/set_get.c \
$(LAME_LIBMP3_DIR)/takehiro.c \
$(LAME_LIBMP3_DIR)/vbrquantize.c \
$(LAME_LIBMP3_DIR)/version.c \
LameUtil.c

include $(BUILD_SHARED_LIBRARY)
```

### 书写jni下的LameUtil.c和LameUtil.h
- LameUtil.c
```
#include "lame-3.99.5_libmp3lame/lame.h"
#include "LameUtil.h"

static lame_global_flags *glf = NULL;

JNIEXPORT void JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_init(
        JNIEnv *env, jclass cls, jint inSamplerate, jint inChannel,
        jint outSamplerate, jint outBitrate, jint quality) {
    if (glf != NULL) {
        lame_close(glf);
        glf = NULL;
    }
    glf = lame_init();
    lame_set_in_samplerate(glf, inSamplerate);
    lame_set_num_channels(glf, inChannel);
    lame_set_out_samplerate(glf, outSamplerate);
    lame_set_brate(glf, outBitrate);
    lame_set_quality(glf, quality);
    lame_init_params(glf);
}

JNIEXPORT jint JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_encode(
        JNIEnv *env, jclass cls, jshortArray buffer_l, jshortArray buffer_r,
        jint samples, jbyteArray mp3buf) {
    jshort* j_buffer_l = (*env)->GetShortArrayElements(env, buffer_l, NULL);

    jshort* j_buffer_r = (*env)->GetShortArrayElements(env, buffer_r, NULL);

    const jsize mp3buf_size = (*env)->GetArrayLength(env, mp3buf);
    jbyte* j_mp3buf = (*env)->GetByteArrayElements(env, mp3buf, NULL);

    int result = lame_encode_buffer(glf, j_buffer_l, j_buffer_r,
            samples, j_mp3buf, mp3buf_size);

    (*env)->ReleaseShortArrayElements(env, buffer_l, j_buffer_l, 0);
    (*env)->ReleaseShortArrayElements(env, buffer_r, j_buffer_r, 0);
    (*env)->ReleaseByteArrayElements(env, mp3buf, j_mp3buf, 0);

    return result;
}

JNIEXPORT jint JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_flush(
        JNIEnv *env, jclass cls, jbyteArray mp3buf) {
    const jsize mp3buf_size = (*env)->GetArrayLength(env, mp3buf);
    jbyte* j_mp3buf = (*env)->GetByteArrayElements(env, mp3buf, NULL);

    int result = lame_encode_flush(glf, j_mp3buf, mp3buf_size);

    (*env)->ReleaseByteArrayElements(env, mp3buf, j_mp3buf, 0);

    return result;
}

JNIEXPORT void JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_close(
        JNIEnv *env, jclass cls) {
    lame_close(glf);
    glf = NULL;
}
```

- LameUtil.h
```
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_android_cong_mediaeditdemo_audiomux_LameUtil */

#ifndef _Included_LameUtil
#define _Included_LameUtil
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_android_cong_mediaeditdemo_audiomux_LameUtil
 * Method:    init
 * Signature: (IIIII)V
 */
JNIEXPORT void JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_init
  (JNIEnv *, jclass, jint, jint, jint, jint, jint);

/*
 * Class:     com_android_cong_mediaeditdemo_audiomux_LameUtil
 * Method:    encode
 * Signature: ([S[SI[B)I
 */
JNIEXPORT jint JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_encode
  (JNIEnv *, jclass, jshortArray, jshortArray, jint, jbyteArray);

/*
 * Class:     com_android_cong_mediaeditdemo_audiomux_LameUtil
 * Method:    flush
 * Signature: ([B)I
 */
JNIEXPORT jint JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_flush
  (JNIEnv *, jclass, jbyteArray);

/*
 * Class:     com_android_cong_mediaeditdemo_audiomux_LameUtil
 * Method:    close
 * Signature: ()V
 */
JNIEXPORT void JNICALL Java_com_android_cong_mediaeditdemo_audiomux_LameUtil_close
  (JNIEnv *, jclass);

#ifdef __cplusplus
}
#endif
#endif
```

### 编译.so
- 终端进入jni目录，使用ndk-build命令进行编译
- 编译成功：

![](https://github.com/xiaokc/Lame/tree/master/app/src/main/res/drawable/so_in_libs.jpeg)

### .so库的使用
- 将编译生成的全部.so库拷贝到需要的项目module文件下的jniLibs，在LameUtil.java类中加载.so库：
```
static{
		System.loadLibrary("mp3lame");
	}
```
- 声明native方法：init(...),encode(...),flush(...),close(...).







