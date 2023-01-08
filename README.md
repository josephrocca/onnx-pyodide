# ONNX Pyodide
The `onnx` Python library (not `onnxruntime`, to be clear) running in Pyodide - WIP

# Progress:

## Version 1

Demo: https://josephrocca.github.io/onnx-pyodide/demo/v1

Shows this warning:

```
Failed to load dynlib /lib/python3.10/site-packages/onnx/onnx_cpp2py_export.cpython-310-wasm32-emscripten.so. We probably just tried to load a linux .so file or something.
```

And then crashes with this error:

<details>
  <summary><b>Click for error logs</b></summary>
  
```
Uncaught PythonError: Traceback (most recent call last):
  File "/lib/python3.10/asyncio/futures.py", line 201, in result
    raise self._exception
  File "/lib/python3.10/asyncio/tasks.py", line 232, in __step
    result = coro.send(None)
  File "/lib/python3.10/_pyodide/_base.py", line 531, in eval_code_async
    await CodeRunner(
  File "/lib/python3.10/_pyodide/_base.py", line 359, in run_async
    await coroutine
  File "<exec>", line 9, in <module>
  File "/lib/python3.10/site-packages/onnx/__init__.py", line 6, in <module>
    from .onnx_cpp2py_export import ONNX_ML  # noqa
ImportError: dynamic module does not define module export function (PyInit_onnx_cpp2py_export)

    at new_error (pyodide.asm.js:10:179954)
    at pyodide.asm.wasm:0xe78a8
    at pyodide.asm.wasm:0xee978
    at method_call_trampoline (pyodide.asm.js:10:229349)
    at pyodide.asm.wasm:0x1313a1
    at pyodide.asm.wasm:0x202469
    at pyodide.asm.wasm:0x16ca9e
    at pyodide.asm.wasm:0x1318b5
    at pyodide.asm.wasm:0x1319af
    at pyodide.asm.wasm:0x131a52
    at pyodide.asm.wasm:0x1eb770
    at pyodide.asm.wasm:0x1e579f
    at pyodide.asm.wasm:0x131a95
    at pyodide.asm.wasm:0x1ed552
    at pyodide.asm.wasm:0x1eb1b2
    at pyodide.asm.wasm:0x1e579f
    at pyodide.asm.wasm:0x131a95
    at pyodide.asm.wasm:0xee1af
    at pyodide.asm.wasm:0xee050
    at Module.callPyObjectKwargs (pyodide.asm.js:10:123403)
    at Module.callPyObject (pyodide.asm.js:10:123781)
    at wrapper (pyodide.asm.js:10:219389)
```
</details>
  
## Version 2
  
Demo: ...

<details>
  <summary><b>Click for build instructions</b></summary>
 
```bash
git clone --branch v21.12 https://github.com/onnx/onnx
cd onnx
git submodule update --init --recursive

pip install pyodide-build
pip install "pybind11[global]"

# install emscripten
git clone https://github.com/emscripten-core/emsdk
cd emsdk
pyodide config get emscripten_version  # this line is needed due to: https://github.com/pyodide/pyodide/issues/3430
PYODIDE_EMSCRIPTEN_VERSION=$(pyodide config get emscripten_version)
./emsdk install ${PYODIDE_EMSCRIPTEN_VERSION}
./emsdk activate ${PYODIDE_EMSCRIPTEN_VERSION}
source emsdk_env.sh
cd ../

# install protobuf (not just binary)
git clone https://github.com/protocolbuffers/protobuf.git
cd protobuf
git checkout v3.20.2
git submodule update --init --recursive
mkdir build_source && cd build_source
cmake ../cmake -Dprotobuf_BUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_SYSCONFDIR=/etc -DCMAKE_POSITION_INDEPENDENT_CODE=ON -Dprotobuf_BUILD_TESTS=OFF -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
cd ../../
export PATH="$(pwd)/protobuf/build_source:$PATH"

# add some extra cmake variables like `set_property(GLOBAL PROPERTY TARGET_SUPPORTS_SHARED_LIBS TRUE)` - see the file for the rest, including references to explanations
curl https://gist.githubusercontent.com/josephrocca/9740493cd72e5be587177b31b40ed8f5/raw/509fab5dc03bec8aa598e7fbce16330de94893ca/overwriteProp.cmake > overwriteProp.cmake

# currently the only edit needed for CMakeLists.txt is to remove `,--exclude-libs,ALL` from line 492 - see explanation here: https://github.com/pyodide/pyodide/issues/3427#issuecomment-1374422693
curl https://gist.githubusercontent.com/josephrocca/9740493cd72e5be587177b31b40ed8f5/raw/ef697fb45c5a00523c266b5265abb11cef2810e7/CMakeLists.txt > CMakeLists.txt

export CMAKE_ARGS="-DONNX_USE_PROTOBUF_SHARED_LIBS=OFF \
-DONNX_USE_LITE_PROTO=OFF \
-DProtobuf_INCLUDE_DIR=$(pwd)/protobuf/src \
-DProtobuf_LIBRARIES=$(pwd)/protobuf/build_source/libprotobuf.a \
-Dpybind11_DIR=$(python -c 'import pybind11 as _; print(_.__path__[0])')/share/cmake/pybind11 \
-DPYTHON_INCLUDE_DIR=$(python -c "import sysconfig; print(sysconfig.get_path('include'))") \
-DPYTHON_LIBRARY=$(python -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))") \
-DCMAKE_PROJECT_INCLUDE=$(pwd)/overwriteProp.cmake \
-DCMAKE_C_FLAGS=\"-fPIC\" \
-DCMAKE_CXX_FLAGS=\"-fPIC\""

# build
pyodide build --exports whole_archive


##########################
#  OPTIONAL STUFF BELOW  #
##########################

# install wabt to inspect generated .so (wasm) file
sudo apt update
sudo apt install ninja-build
git clone --recursive https://github.com/WebAssembly/wabt
cd wabt
git submodule update --init
make

cd dist
unzip /workspaces/onnx/dist/onnx-1.13.0-cp310-cp310-emscripten_3_1_27_wasm32.whl
cd ../

/workspaces/onnx/wabt/bin/wasm-objdump --full-contents /workspaces/onnx/dist/onnx/onnx_cpp2py_export.cpython-310-wasm32-emscripten.so > wabt-full-contents.txt
/workspaces/onnx/wabt/bin/wasm-objdump --details /workspaces/onnx/dist/onnx/onnx_cpp2py_export.cpython-310-wasm32-emscripten.so > wabt-details.txt
/workspaces/onnx/wabt/bin/wasm-objdump --disassemble /workspaces/onnx/dist/onnx/onnx_cpp2py_export.cpython-310-wasm32-emscripten.so > wabt-disassemble.txt
/workspaces/onnx/wabt/bin/wasm-objdump --headers /workspaces/onnx/dist/onnx/onnx_cpp2py_export.cpython-310-wasm32-emscripten.so > wabt-headers.txt

```
</details>
  

<details>
  <summary><b>...</b></summary>
  
```

```
</details>

