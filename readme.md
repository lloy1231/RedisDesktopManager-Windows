## 前言

因为作者在 0.9.4 之后选择对所有的安装包收费，不再提供安装包下载，但是源码依旧公开，于是本人选择了自己编译 Windows 版本的道路，同时无偿提供给大家。

下载地址：[release](https://github.com/necan/RedisDesktopManager-Windows/releases)

## 编译过程

网络上 关于 Redis Desktop Manager 的编译教程都是 Linux 下的，没有任何参考价值。而官方文档提供的步骤过于简洁，以至于根本无法操作。本人摸索了几个小时，终于完整编译并打包。具体操作如下：

### 安装工具

#### 安装 VSCode 2015

到 [http://blog.postcha.com/read/66](http://blog.postcha.com/read/66) 下载 VSCode 专业版，自定义安装，一定要勾选 VC ++，然后一直下一步，大约两个小时装完。

#### 安装 Qt 5.9

到 [http://mirrors.ustc.edu.cn/qtproject/archive/qt/5.9/](http://mirrors.ustc.edu.cn/qtproject/archive/qt/5.9/) 下载最新到 Qt 5.9 版本，一直下一步就行

#### 安装 CMake

到 [https://cmake.org/download/](https://cmake.org/download/) 下载 32 位的版本，安装时注意勾选添加到 PATH

#### 安装 NSIS

安装打包工具 [http://nsis.sourceforge.net/Download](http://nsis.sourceforge.net/Download)

#### 安装 Python 2

右键解压，不要安装，因为我用到 Python 3，不想冲突

#### 安装 OpenSSL

下载并安装 [Win 32 OpenSSL 1.0.x](https://slproweb.com/products/Win32OpenSSL.html) 

### 编译 Redis Desktop Manager

打开 “VS2015 x86 本机工具命令提示符”

#### 获取源码

```powershell
git clone -q --depth=5 --branch=0.9 https://github.com/uglide/RedisDesktopManager.git D:\redisdesktopmanager
cd D:\redisdesktopmanager
git submodule update --init --recursive
```

#### 编译 libssh2

```powershell
cd ./3rdparty/qredisclient/3rdparty/qsshclient/3rdparty/libssh2
cmake -G "Visual Studio 14 2015" -DCRYPTO_BACKEND=OpenSSL -DBUILD_EXAMPLES=off -DBUILD_TESTING=off -H. -Bbuild
cmake --build build --config "Release"
```

#### 设置版本号

版本号自己到 Github 找

```powershell
cd D:\redisdesktopmanager
set APPVEYOR_BUILD_VERSION=0.9.4.1055
"D:\Program Files\Python\python27\python.exe" ./build/utils/set_version.py %APPVEYOR_BUILD_VERSION% > ./src/version.h
"D:\Program Files\Python\python27\python.exe" ./build/utils/set_version.py %APPVEYOR_BUILD_VERSION% > ./3rdparty/crashreporter/src/version.h
```

#### 编译 crashreporter

```powershell
cd ./3rdparty/crashreporter
"D:\Qt\Qt5.9.6\5.9.6\msvc2015\bin\qmake.exe" CONFIG+=release DESTDIR=D:\redisdesktopmanager\bin\windows\release
powershell -Command "(Get-Content Makefile.Release).replace('DEFINES       =','DEFINES       = -DAPP_NAME=\\\"RedisDesktopManager\\\" -DAPP_VERSION=\\\""%APPVEYOR_BUILD_VERSION%"\\\" -DCRASH_SERVER_URL=\\\"https://oops.redisdesktop.com/crash-report\\\"')" > Makefile.Release2
nmake -f Makefile.Release2
```

#### Qt 编译

打开 Qt Creator，打开 `./src/rdm.pro`

选择 “Deaktop Qt 5.9.6 MSVC2015 32bit”，构建选择 release，点击构建项目。

### 打包

```powershell
cd D:\redisdesktopmanager
copy /y .\bin\windows\release\rdm.exe .\build\windows\installer\resources\rdm.exe
copy /y .\bin\windows\release\rdm.pdb .\build\windows\installer\resources\rdm.pdb
D:\redisdesktopmanager\3rdparty\gbreakpad\src\tools\windows\binaries\dump_syms .\bin\windows\release\rdm.pdb  > .\build\windows\installer\resources\rdm.sym
cd build/windows/installer/resources/
D:\Qt\Qt5.9.6\5.9.6\msvc2015\bin\windeployqt --no-angle --no-opengl-sw --no-compiler-runtime --no-translations --release --force --qmldir D:\redisdesktopmanager\src\qml rdm.exe
rmdir /S /Q .\platforminputcontexts
rmdir /S /Q .\qmltooling
rmdir /S /Q .\QtGraphicalEffects
del /Q  .\imageformats\qtiff.dll
del /Q  .\imageformats\qwebp.dll
cd D:\redisdesktopmanager
call "C:\\Program Files (x86)\\NSIS\\makensis.exe" /V1 /DVERSION=0.9.4.1055  ./build/windows/installer/installer.nsi
```

打包后的文件：`D:\redisdesktopmanager\build\windows\installer\redis-desktop-manager-0.9.4.1055.exe`
