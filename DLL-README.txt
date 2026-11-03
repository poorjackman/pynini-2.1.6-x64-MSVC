OpenFST 1.8.3 + dlfcn-win32 1.4.1 runtime DLLs
for pynini 2.1.6.post1 (CPython 3.12, Windows x64)

Extract this archive into the Python environment's site-packages directory:

  tar -xf openfst-dlls-win64.zip -C D:\path\to\.venv\Lib\site-packages

All DLLs must sit in the same directory as _pynini.cp312-win_amd64.pyd.
Adding them to PATH alone is not sufficient for Python 3.8+.

Licenses:
  LICENSES/OpenFST-COPYING          Apache License 2.0
  LICENSES/dlfcn-win32-COPYING      MIT License
