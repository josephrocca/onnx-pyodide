# ONNX Pyodide
The `onnx` Python library (not `onnxruntime`, to be clear) running in Pyodide - WIP

# Demo:

Here's the current demo: https://josephrocca.github.io/onnx-pyodide/demo/v1

It's currently crashing with this error:

<details>
  <summary>Click for error logs</summary>
  
```
Uncaught PythonError: Traceback (most recent call last):
  File "/lib/python3.10/asyncio/futures.py", line 201, in result
    raise self._exception
  File "/lib/python3.10/asyncio/tasks.py", line 234, in __step
    result = coro.throw(exc)
  File "/lib/python3.10/_pyodide/_base.py", line 531, in eval_code_async
    await CodeRunner(
  File "/lib/python3.10/_pyodide/_base.py", line 359, in run_async
    await coroutine
  File "<exec>", line 7, in <module>
  File "/lib/python3.10/site-packages/micropip/_micropip.py", line 600, in install
    await gather(*wheel_promises)
  File "/lib/python3.10/asyncio/futures.py", line 284, in __await__
    yield self  # This tells Task to wait for completion.
  File "/lib/python3.10/asyncio/tasks.py", line 304, in __wakeup
    future.result()
  File "/lib/python3.10/asyncio/futures.py", line 201, in result
    raise self._exception
  File "/lib/python3.10/asyncio/tasks.py", line 234, in __step
    result = coro.throw(exc)
  File "/lib/python3.10/site-packages/micropip/_micropip.py", line 247, in install
    await self.load_libraries(target)
  File "/lib/python3.10/site-packages/micropip/_micropip.py", line 238, in load_libraries
    await gather(*map(lambda dynlib: loadDynlib(dynlib, False), dynlibs))
  File "/lib/python3.10/asyncio/futures.py", line 284, in __await__
    yield self  # This tells Task to wait for completion.
  File "/lib/python3.10/asyncio/tasks.py", line 304, in __wakeup
    future.result()
  File "/lib/python3.10/asyncio/futures.py", line 201, in result
    raise self._exception
pyodide.JsException: TypeError: Cannot read properties of undefined (reading 'apply')

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

# Build Instructions:

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

curl https://gist.githubusercontent.com/josephrocca/9740493cd72e5be587177b31b40ed8f5/raw/509fab5dc03bec8aa598e7fbce16330de94893ca/overwriteProp.cmake > overwriteProp.cmake
curl https://gist.githubusercontent.com/josephrocca/9740493cd72e5be587177b31b40ed8f5/raw/ef697fb45c5a00523c266b5265abb11cef2810e7/CMakeLists.txt > CMakeLists.txt  # currently the only edit is to remove `,--exclude-libs,ALL` from line 492 - see explanation here: https://github.com/pyodide/pyodide/issues/3427#issuecomment-1374422693

export CMAKE_ARGS="-DONNX_USE_LITE_PROTO=OFF -DProtobuf_INCLUDE_DIR=$(pwd)/protobuf/src -DProtobuf_LIBRARIES=$(pwd)/protobuf/build_source -Dpybind11_DIR=$(python -c 'import pybind11 as _; print(_.__path__[0])')/share/cmake/pybind11 -DPYTHON_INCLUDE_DIR=$(python -c "import sysconfig; print(sysconfig.get_path('include'))") -DPYTHON_LIBRARY=$(python -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))") -DCMAKE_PROJECT_INCLUDE=$(pwd)/overwriteProp.cmake -DCMAKE_C_FLAGS=\"-fPIC\" -DCMAKE_CXX_FLAGS=\"-fPIC\""

pyodide build --exports whole_archive
```
