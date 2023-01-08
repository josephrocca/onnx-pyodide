# ONNX Pyodide
The `onnx` Python library (not `onnxruntime`, to be clear) running in Pyodide.

Demo: https://josephrocca.github.io/onnx-pyodide/demo/v3

# Build Instructions

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

# Build wasm version of protobuf
git clone https://github.com/protocolbuffers/protobuf.git protobuf-wasm
cd protobuf-wasm
git checkout v3.20.2
git submodule update --init --recursive
mkdir build_source
emcmake cmake cmake -B=build_source -Dprotobuf_BUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_SYSCONFDIR=/etc -DCMAKE_POSITION_INDEPENDENT_CODE=ON -Dprotobuf_BUILD_TESTS=OFF -DCMAKE_BUILD_TYPE=Release
cd build_source
emmake make
sudo env "PATH=$PATH" emmake make install
cd ../../
# cp "$(pwd)/protobuf-wasm/build_source/protoc.js-3.20.2.wasm" "$(pwd)/protobuf-wasm/build_source/libprotobuf.a"


# build "real" version of protobuf because we need the protoc binary in PATH
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
-DProtobuf_INCLUDE_DIR=$(pwd)/protobuf-wasm/src \
-DProtobuf_LIBRARIES=$(pwd)/protobuf-wasm/build_source/libprotobuf.a \
-Dpybind11_DIR=$(python -c 'import pybind11 as _; print(_.__path__[0])')/share/cmake/pybind11 \
-DPYTHON_INCLUDE_DIR=$(python -c "import sysconfig; print(sysconfig.get_path('include'))") \
-DPYTHON_LIBRARY=$(python -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))") \
-DCMAKE_PROJECT_INCLUDE=$(pwd)/overwriteProp.cmake \
-DCMAKE_C_FLAGS=\"-fPIC\" \
-DCMAKE_CXX_FLAGS=\"-fPIC\""

# build
pyodide build --exports whole_archive
```

