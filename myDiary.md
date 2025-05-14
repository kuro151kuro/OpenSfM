# 構築日記
- Windows 11
# Installing dependencies on Windows

<img src="./md_image/Installing dependencies on Windows.png" alt="" width="300">


## python3.9をインストール
- 仮想環境を構築
### PSで git コマンドを実行するためGitをインストール

## cmakeのインストール

```powershell
bootstrap -vcpkg.bat
```
でVisual StudioのC++ビルド環境が必要といわれる
- [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/ja/visual-cpp-build-tools/) 
の選択項目からCmakeをインストール

```bash
.\OpenSfM\vcpkg\downloads\tools\cmake-3.30.1-windows\cmake-3.30.1-windows-i386\bin
```
に cmake-gui.exe があるのでPathを通す

# Building the library
## sphinx がないといわれるのでインストール
(requirements.txtにあるはずだけどインストールされてない)
```powershell
pip install "Sphinx==4.2.0"
```

## wheel がないといわれるのでインストール
(requirements.txtにあるはずだけどインストールされてない)
```powershell
pip install wheel
```

## vcpkgのライブラリが見つからないと言われる
- BLAS / LAPACK
    → 数値計算ライブラリ。ceres や suitesparse に必須
- SuiteSparse
    → 構造化線形代数の主要ライブラリ群（OpenSfM に必要）
- Eigen3
    → 行列演算ライブラリ。Eigen3Config.cmake が見つからないと出ている

vcpkg 経由でライブラリをインストール
すでに vcpkg がある場所に移動して、以下を実行：

```powershell
cd .\vcpkg
.\vcpkg install eigen3 suitesparse lapack blas ceres --triplet x64-windows
```


## SuiteSparse のヘッダーが見つからない
```
Failed to find SuiteSparse - Did not find AMD header # など 
```
🔎 原因
vcpkg が *.lib ファイル（ライブラリ）はインストールしているが、対応する *.h（ヘッダー）ファイルが見つかっていない。

よくあるのは CMake がインクルードパスを探せていないケース
SuiteSparseConfig.cmakeの修正を求められたが存在しない

❌ SuiteSparseConfig.cmake が存在しない理由
- vcpkg の suitesparse パッケージは CMake 用の「Config ファイル (SuiteSparseConfig.cmake) を提供しない
- 代わりに 個別ライブラリ（CHOLMOD, AMD, etc.）ごとの .cmake ファイル が生成され、プロジェクト側で必要に応じて手動で find_library() や include_directories() を使って探す形式

✅ 修正ポイント

".\opensfm\src\CMakeLists.txt" 内でこの1行を コメントアウト
```cmake
find_package(SuiteSparse)
```

そのすぐ後に以下の内容を 追加：

```cmake
# SuiteSparse のライブラリとヘッダを明示的に指定（vcpkg用）
include_directories("C:/Users/9614k/Documents/GitHub/OpenSfM/vcpkg/installed/x64-windows/include")
link_directories("C:/Users/9614k/Documents/GitHub/OpenSfM/vcpkg/installed/x64-windows/lib")

set(SUITESPARSE_LIBS
  amd
  camd
  colamd
  ccolamd
  cholmod
  suitesparseconfig
  cxsparse
)

foreach(lib ${SUITESPARSE_LIBS})
  find_library(${lib}_LIBRARY ${lib} PATHS "C:/Users/9614k/Documents/GitHub/OpenSfM/vcpkg/installed/x64-windows/lib")
endforeach()
```

## OpenCV が見つからない
```bash
Could not find a package configuration file provided by "OpenCV" # など
```

🔎 原因
vcpkg install opencv4 をしていても、OpenCV_DIR が CMake に伝わっていない

✅ 対応策

vcpkgコマンド実行のため、vcpkg.exe のPATHを環境変数に入力
```bash
C:\Users\9614k\Documents\GitHub\OpenSfM\vcpkg
```

OpenCV を vcpkg でインストール
```powershell
.\vcpkg install opencv4 --triplet x64-windows
```
その後、setup.py 内で OpenCV_DIR 環境変数を指定：

```python
def configure_c_extension():
    """Configure cmake project to C extension."""
    print(
        f"Configuring for python {sys.version_info.major}.{sys.version_info.minor}..."
    )
    os.makedirs("cmake_build", exist_ok=True)
    cmake_command = [
        "cmake",
        "../opensfm/src",
        "-DPYTHON_EXECUTABLE=" + sys.executable,
    ]
    if sys.platform == "win32":
        cmake_command += [
            "-DVCPKG_TARGET_TRIPLET=x64-windows",
            "-DCMAKE_TOOLCHAIN_FILE=../vcpkg/scripts/buildsystems/vcpkg.cmake",
            "-DOpenCV_DIR=../vcpkg/installed/x64-windows/share/opencv4", # OpenCVConfig.cmake へのファイルパスを追加
        ]
    subprocess.check_call(cmake_command, cwd="cmake_build")
```

```
# glog に対して定数 GLOG_NO_ABBREVIATED_SEVERITIES
add_definitions(-DGLOG_NO_ABBREVIATED_SEVERITIES)
```


## CMakeLists.txt の if 文で構文エラー
```javascript
CMake Error at CMakeLists.txt:76 (if):
if given arguments: "LESS" "3.0"
Unknown arguments specified
```
🔎 原因
if(LESS 3.0) は CMake にとって無効な文です。

✅ 対応策
OpenSfM/CMakeLists.txt を開いて、

76行目あたりの if 文 を確認：

```cmake
if(LESS 3.0)   # ❌ 間違い
```
これを正しい形式に修正：

```cmake
if(${CMAKE_VERSION} VERSION_LESS "3.0")  # ✅ 正しい
```
VERSION_LESS で比較するのが CMake の正式な構文です。