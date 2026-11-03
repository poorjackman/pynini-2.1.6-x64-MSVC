# pynini-2.1.6-x64-MSVC

Windows x64（MSVC）版 **Pynini 2.1.6.post1** 的构建记录与预编译 wheel。

- 关联项目：[OpenMOSS/MOSS-TTS-Nano](https://github.com/OpenMOSS/MOSS-TTS-Nano)（SSH：`git@github.com:OpenMOSS/MOSS-TTS-Nano.git`）
- Pynini 源码：`2.1.6.post1`（Apache-2.0）
- 适用平台：Windows x64 + CPython 3.12（wheel 标签：`cp312-cp312-win_amd64`）
- 编译工具：MSVC（Visual Studio 2026 / MSVC 14.51）+ OpenFST 1.8.3（conda-forge win-64）
- 标签：`pynini`、`moss-tts-nano`

本项目提供可直接用于 Windows 原生环境的 Pynini wheel 与完整编译记录，适用于不想使用 WSL 或 conda 安装 Pynini 的 MOSS-TTS-Nano 用户。官方源码在 MSVC 下会因硬编码 GCC/Clang 参数报 `D8021`，本仓库记录并修复了该问题。

---

问题,原项目的一个依赖没有适配Windows，详见：[MOSS-TTS-Nano-#issue#6](https://github.com/OpenMOSS/MOSS-TTS-Nano/issues/6)

以下给是本地编译的过程(DeepSeek),和不使用conda配置原TTS项目的过程(Copilot)。

本项目已经修改好，成品whl文件从release获取。

# whl 文件使用

## 适用环境

- Windows x64
- CPython 3.12（此 wheel 为 `cp312-cp312-win_amd64`，其他 Python 版本不能使用）
- 已安装 `uv` 或 `pip`

## 安装步骤

从 [Releases](https://github.com/poorjackman/pynini-2.1.6-x64-MSVC/releases) 下载两个文件：

- `pynini-2.1.6.post1-cp312-cp312-win_amd64.whl`
- `openfst-dlls-win64.zip`

安装 wheel：

```bat
uv pip install pynini-2.1.6.post1-cp312-cp312-win_amd64.whl
:: 或者
pip install pynini-2.1.6.post1-cp312-cp312-win_amd64.whl
```

将 DLL 解压到当前环境的 `site-packages`（必须与 `_pynini.cp312-win_amd64.pyd` 同目录）：

```bat
tar -xf openfst-dlls-win64.zip -C D:\path\to\.venv\Lib\site-packages
```

> 注意：只把 DLL 加入 `PATH` 是不够的。Python 3.8+ 加载扩展模块时不再通过 `PATH` 搜索依赖 DLL，必须放在 `.pyd` 同目录，或通过 `os.add_dll_directory()` 注册。
>
> wheel 内不包含 DLL；OpenFST 与 dlfcn-win32 的 DLL 单独发布，便于明确许可证边界。

验证：

```bat
python -c "import pynini; print(pynini.__version__)"
```

预期输出：

```text
2.1.6.post1
```

快速功能验证：

```bat
python -c "import pynini; print(pynini.shortestpath(pynini.concat(pynini.accep('a'), pynini.accep('b'))).string())"
```

预期输出 `ab`。

## 文件校验

- wheel SHA-256：`02a8dd942e0d69e0cd49e8b9d54104452d8901fbc403ad8f1aefbb4a7af1e3fa`
- DLL 包 SHA-256：`36b28b899f274c005002c95d63bda2af47ef1258b6fe104199e81b03e235dc35`

## 卸载

```bat
uv pip uninstall pynini
```

然后删除 `site-packages` 下的 `fst*.dll`、`dl.dll`、`LICENSES`（如不再需要）。


# pynini 本地编译（MSVC）

先清除旧构建产物，再在 `pynini-2.1.6.post1` 源码目录执行：

```bat
rmdir /s /q build
rmdir /s /q pynini.egg-info
uv pip install --no-build-isolation --python D:\projectRepo\MOSS-TTS-Nano\.venv\Scripts\python.exe .
```

a. 修改的编译参数

问题出在 `setup.py`：它无条件把 GCC/Clang 参数传给编译器，MSVC 会直接报 `D8021: 无效的数值参数`。修改内容：

- 保留非 MSVC 环境原有的 `COMPILE_ARGS`，新增 `MSVC_COMPILE_ARGS` 做等价替换：

| GCC/Clang                      | MSVC         | 说明                   |
| ------------------------------ | ------------ | ---------------------- |
| `-std=c++17`                   | `/std:c++17` | C++ 标准               |
| `-funsigned-char`              | `/J`         | 让 `char` 默认为无符号 |
| `-Wno-deprecated-declarations` | `/wd4996`    | 关闭弃用警告           |
| `-Wno-unused-function`         | `/wd4505`    | 关闭未使用函数警告     |
| `-Wno-register`                | 无           | MSVC 无此警告，省略    |
| `-Wno-unused-local-typedefs`   | 无           | MSVC 无此警告，省略    |

- 通过自定义 `BuildExt(build_ext)`，在 `build_extensions()` 中按 `self.compiler.compiler_type == "msvc"` 切换参数，并注册 `cmdclass={"build_ext": BuildExt}`。按实际编译器类型判断（而不是 `sys.platform`），clang-cl 也会走 MSVC 分支。
- MSVC 下从 `libraries` 中移除 `m`、`dl`：这两个是 POSIX 系统库，Windows 上没有对应的 `m.lib`/`dl.lib`，链接会失败。
- 追加两个 Windows 必需宏：
  - `/DFST_NO_DYNAMIC_LINKING`：Windows 没有 `<dlfcn.h>`，需与 OpenFST 的 Windows 构建保持一致。
  - `/DNOMINMAX`：阻止 `windows.h` 定义 `min`/`max` 宏，否则 OpenFST 头文件和 pyx 生成代码中的 `std::max` 会被破坏。
- `extensions/cdrewrite.h`：Windows SDK 把 `OPTIONAL` 定义成空宏，`CDRewriteMode::OPTIONAL` 会被展开成 `CDRewriteMode::`。在枚举定义前加入：

```cpp
#ifdef OPTIONAL
#undef OPTIONAL
#endif
```

- `extensions/cpynini.pxd`：修正枚举 C++ 名拼写 `OPTOINAL` -> `OPTIONAL`。

b. 环境补齐的原因和方法

原因：pynini 源码本身不包含 OpenFST，编译必须有与 MSVC 兼容的 OpenFST 1.8.3 头文件和 `.lib`。本机原先没有安装 OpenFST，直接编译会在 `fst/util.h` 处报 `C1083: 无法打开包括文件`。Linux 版或非 Windows 的 conda 包也不能用，必须使用 MSVC 构建的 Windows 包。conda-forge 提供了 `openfst 1.8.3 win-64`，正好对应 pynini 2.1.6.post1 的兼容版本；它的 `fst.dll` 还依赖 `dl.dll`，因此要同时补 `dlfcn-win32`。

方法：

1. 下载并解包（以下命令在 `D:\projectRepo\MOSS-TTS-Nano\.deps` 目录执行；Windows 自带的 `tar` 支持 `.conda` 和内部的 `.tar.zst`）：

```bat
curl -L -o openfst-1.8.3-hc790b64_3.conda https://conda.anaconda.org/conda-forge/win-64/openfst-1.8.3-hc790b64_3.conda
mkdir D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64
tar -xf openfst-1.8.3-hc790b64_3.conda -C D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64
tar -xf D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\pkg-openfst-1.8.3-hc790b64_3.tar.zst -C D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64

curl -L -o dlfcn-win32-1.4.1-h63175ca_0.conda https://conda.anaconda.org/conda-forge/win-64/dlfcn-win32-1.4.1-h63175ca_0.conda
mkdir D:\projectRepo\MOSS-TTS-Nano\.deps\dlfcn-win32
tar -xf dlfcn-win32-1.4.1-h63175ca_0.conda -C D:\projectRepo\MOSS-TTS-Nano\.deps\dlfcn-win32
tar -xf D:\projectRepo\MOSS-TTS-Nano\.deps\dlfcn-win32\pkg-dlfcn-win32-1.4.1-h63175ca_0.tar.zst -C D:\projectRepo\MOSS-TTS-Nano\.deps\dlfcn-win32
```

2. 编译时让 MSVC 找到 OpenFST 的头文件和导入库（每次重新编译都要设置）：

```bat
set INCLUDE=D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\include
set LIB=D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\lib
uv pip install --no-build-isolation --python D:\projectRepo\MOSS-TTS-Nano\.venv\Scripts\python.exe .
```

3. 运行时补齐 DLL。Python 3.8 以后不再通过 `PATH` 搜索扩展模块的依赖 DLL，必须放到 `.pyd` 同目录（或调用 `os.add_dll_directory`）：

```bat
copy D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\bin\fst.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
copy D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\bin\fstscript.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
copy D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\bin\fstfar.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
copy D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\bin\fstfarscript.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
copy D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\bin\fstpdtscript.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
copy D:\projectRepo\MOSS-TTS-Nano\.deps\openfst-win64\Library\bin\fstmpdtscript.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
copy D:\projectRepo\MOSS-TTS-Nano\.deps\dlfcn-win32\Library\bin\dl.dll D:\projectRepo\MOSS-TTS-Nano\.venv\Lib\site-packages\
```

验证：

```bat
D:\projectRepo\MOSS-TTS-Nano\.venv\Scripts\python.exe -c "import pynini, pywrapfst; print(pynini.__version__)"
```

输出 `2.1.6.post1` 即成功；已进一步验证 `cdrewrite(..., mode="opt")`、`pynini.concat` 和 `shortestpath` 均正常。

注意：如果没有设置 `INCLUDE`/`LIB` 就直接重编译，会停在 `fst/util.h` 找不到，这属于 OpenFST 路径未配置，不是编译参数问题；如果没有复制 DLL，则导入时报 `DLL load failed while importing _pynini`。



# MOSS-TTS-Nano 配置 不用 conda

如果电脑磁盘空间充足，安装官方介绍，使用 conda 完成配置更方便（conda 配置python 包更全，占用空间更多，uv 配置 python 更快）。



### 1. 克隆项目、配置环境

```shell
git clone https://github.com/OpenMOSS/MOSS-TTS-Nano.git
```

创建、激活虚拟环境:

```shell
uv venv --python 3.12 --seed .venv
.\.venv\Scripts\Activate.ps1
```

安装 pip / setuptools / wheel

```shell
uv pip install --python .\.venv\Scripts\python.exe --upgrade pip setuptools wheel
```



### 2. 依赖配置

```shell
uv pip install "pynini==2.1.6.post1"
```

```shell
uv pip install "git+https://github.com/WhizZest/WeTextProcessing.git"
```

Q. pynini 编译报错怎么办？

```
   self._finalize_license_expression()

      cl: ...... error D8021
      ......
      ......
      ......,-std=c++17', '-Wno-register',

      '-Wno-deprecated-declarations', '-Wno-unused-function', '-Wno-unused-local-typedefs', '-funsigned-char']'

      returned non-zero exit status 2.
```

原因：pynini==2.1.6.post1 的源码中硬编码传递了 GCC/Clang 专用的编译器参数,如 -std=c++17、-Wno-register,MSVC 编译器（cl.exe），这些 Linux 风格的参数，从而导致编译失败。

解决方法有如下(GPT)：

1. 升级 pynini 到更新的版本。（2.1.7 出同样的问题）
2. 下载已经编译好的对应 Python 3.12 的 .whl 文件。（没找到）
3. 换用Windows的GCC/Clang 编译器。（CPython 的解释器不支持）

> conda-forge 社区维护了译的底层科学计算/NLP 库（包含 pynini、pywrapfst 等的预编译二进制包）。
> 直接下载：conda install -c conda-forge pynini

4. 修改源码，自己编译，参考如上。



### 3. 安装项目依赖

```shell
uv pip install -r requirements.txt
pip install -e .

```



### 4. 验证

```shell
python -c "import torch; print('torch 配置成功')"

```



### 5. 启动

#### 命令行执行

```powershell
python infer_onnx.py `
   --prompt-audio-path assets/audio/zh_1.wav `
   --text "请在此输入文本，使用本地 CPU 版本 TTS 测试，生成的音频保存在本项目output目录下。"
```



#### 在浏览器启动

CPU版（适合无显卡的电脑）：

```shell
python app_onnx.py

```

GPU版，更快：
```shell
python app.py
```
