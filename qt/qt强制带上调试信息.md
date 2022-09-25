[qt release版本带上调试信息](https://stackoverflow.com/questions/6993061/build-qt-in-release-with-debug-info-mode)

### 强制带上调试信息

[https://stackoverflow.com/posts/35704181/timeline](https://stackoverflow.com/posts/35704181/timeline)[]
[https://forum.qt.io/topic/90561/generating-debug-symbols-pdb-for-release](https://forum.qt.io/topic/90561/generating-debug-symbols-pdb-for-release)

Old question, I know. But nowadays, you can simply use

```cpp
CONFIG += force_debug_info
```

to get debug symbols even in release mode. When you use `QMake` via the command line, I usually do this to get a release build with debug info:

```cpp
qmake CONFIG+=release CONFIG+=force_debug_info path/to/sources
```

this will enable below conditions of `Qt5/mkspecs/features/`[default_post.prf](https://github.com/qt/qtbase/blob/96efc38f100686a8183f45367e54bf6cb670bdba/mkspecs/features/default_post.prf#L35):

```cpp
force_debug_info|debug: CONFIG += debug_info
force_debug_info {
    QMAKE_CFLAGS_RELEASE = $$QMAKE_CFLAGS_RELEASE_WITH_DEBUGINFO
    QMAKE_CXXFLAGS_RELEASE = $$QMAKE_CXXFLAGS_RELEASE_WITH_DEBUGINFO
    QMAKE_LFLAGS_RELEASE = $$QMAKE_LFLAGS_RELEASE_WITH_DEBUGINFO
}
```

which would even work for `Qt 4.x` but we would need to manually append above conditions into `default_post.prf` for `Qt 4.x`

### 调试文件独立出去

[](https://stackoverflow.com/posts/51556261/timeline)

Just select Profile build in the projects tab of Qt Creator rather than the debug or release builds. It will add a lot of arguments to the qmake call.

```cpp
qmake.exe someproject.pro -spec win32-msvc "CONFIG+=qml_debug" 
"CONFIG+=qtquickcompiler" "CONFIG+=force_debug_info" "CONFIG+=separate_debug_info"
```

