# ONNX Pyodide
The `onnx` Python library (not `onnxruntime`, to be clear) running in Pyodide - WIP


# Build Instructions:

```bash
git clone --single-branch v21.12 https://github.com/onnx/onnx
cd onnx
git submodule update --init --recursive

# pyodide setup:
pip install pyodide-build

# install emscripten
git clone https://github.com/emscripten-core/emsdk
cd emsdk
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

pip install "pybind11[global]"
export PATH="$(pwd)/protobuf/build_source:$PATH"
export CMAKE_ARGS="-DONNX_USE_LITE_PROTO=ON -DProtobuf_INCLUDE_DIR=$(pwd)/protobuf/src -DProtobuf_LIBRARIES=$(pwd)/protobuf/build_source -DProtobuf_LITE_LIBRARY=$(pwd)/protobuf/build_source/libprotobuf-lite.a -Dpybind11_DIR=$(python -c 'import pybind11 as _; print(_.__path__[0])')/share/cmake/pybind11"
pyodide build
```
