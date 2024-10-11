# TVM.CMakeExtend

A modular approach to extend TVM without modifying its core code.

## Introduction

`TVM.CMakeExtend` is a template project that demonstrates how to extend [Apache TVM](https://github.com/apache/tvm) by adding custom passes, code generators, or other functionalities without directly modifying the core TVM source code. This approach leverages CMake's modular build system to separate your custom extensions from TVM's core, facilitating easier maintenance, updates, and collaboration.

This project serves as a reference for developers who are building projects based on TVM and wish to avoid the complexities and pitfalls of forking and modifying TVM directly.

## Background

When building projects on top of TVM, it's common to encounter scenarios where the current capabilities of TVM are insufficient, necessitating modifications or extensions to its core functionalities. Traditionally, developers have forked TVM and directly altered its source code, adding new passes, custom nodes, or code generators. While this method works, it introduces several challenges:

- **Maintenance Overhead**: Keeping the forked version up-to-date with the upstream TVM changes becomes cumbersome.
- **Collaboration Difficulties**: Developers contributing to your project also need to navigate your modified version of TVM.
- **Upstream Integration**: Changes are often too specific or hacky to be merged back into the main TVM repository.
- **Dependency Conflicts**: Direct modifications can lead to conflicts with other projects or dependencies that rely on the standard TVM.

To address these issues, `TVM.CMakeExtend` proposes a solution that maintains a clean separation between TVM's core and your custom extensions.

## Solution Overview

This project demonstrates how to:

- **Keep TVM as an Independent Module**: Treat TVM as an external dependency, either as a submodule or by linking to a prebuilt version.
- **Use CMake for Modular Builds**: Utilize CMake to build your custom code separately, linking against the TVM libraries without integrating your code into TVM's source tree.
- **Avoid Code Duplication and Conflicts**: By not modifying TVM directly, you avoid merge conflicts and can benefit from the latest updates in TVM without additional overhead.
- **Facilitate Collaboration**: Other developers can contribute to your project without needing to navigate a custom version of TVM.

## Repository Structure

```
TVM.CMakeExtend/
├── 3rdparty/
│   └── tvm/                 # Submodule pointing to TVM
├── build/                   # Build directory
├── include/                 # Custom header files
├── src/                     # Custom source files (passes, codegens, etc.)
├── python/
│   └── your_project/        # Python bindings and extensions
├── CMakeLists.txt           # Main CMake configuration
└── README.md                # This README file
```

## Getting Started

### Prerequisites

- CMake (version 3.13 or higher)
- A C++17 compatible compiler
- Optional: CUDA toolkit if working with GPU code
- Python 3.6 or higher (for Python bindings)

### Clone the Repository

```bash
git clone --recursive https://github.com/LeiWang1999/TVM.CMakeExtend.git
```

### Building the Project

You have two options:

1. **Use a Prebuilt TVM**

   If you already have TVM installed or built elsewhere (e.g., via `pip install apache-tvm`), you can link against it.

   ```bash
   mkdir build && cd build
   cmake .. -DTVM_PREBUILD_PATH=/path/to/tvm/build
   make -j$(nproc)
   ```

   Replace `/path/to/tvm/build` with the actual path to your TVM build directory containing `libtvm.so` and `libtvm_runtime.so`.

2. **Build TVM from Source**

   If you prefer to build TVM from source along with your project:

   ```bash
   mkdir build && cd build
   cp ../3rdparty/tvm/cmake/config.cmake .
   # Edit config.cmake to enable desired features
   echo "set(USE_LLVM ON)" >> config.cmake
   echo "set(USE_CUDA ON)" >> config.cmake
   cmake ..
   make -j$(nproc)
   ```

   This will build both TVM and your custom extensions.

### Environment Setup

Ensure that the built libraries are discoverable at runtime:

- **Library Paths**: Add the paths to `libtvm.so`, `libtvm_runtime.so`, and your custom shared libraries to your `LD_LIBRARY_PATH` (or equivalent) environment variable.
- **Python Paths**: Update `PYTHONPATH` to include TVM's and your project's Python packages.

### Using Your Custom Extensions in Python

Your custom passes or functionalities can be registered and accessed from Python.

For example, in your C++ code:

```cpp
#include <tvm/tir/transform.h>

tvm::transform::Pass MyCustomPass() {
    // Your pass implementation
}

TVM_REGISTER_GLOBAL("my_project.transform.MyCustomPass")
.set_body_typed(MyCustomPass);
```

In Python:

```python
import tvm
import your_project

mod = ...  # your IRModule
mod = tvm.transform.Sequential([
    your_project.transform.MyCustomPass(),
])(mod)
```

## Detailed Explanation

### CMake Modular Build

The key to this approach is the CMake configuration that allows you to build your project separately while linking against TVM.

**Using Prebuilt TVM Libraries**

```cmake
if (DEFINED TVM_PREBUILD_PATH)
    message(STATUS "Using prebuilt TVM from ${TVM_PREBUILD_PATH}")
    add_library(tvm SHARED IMPORTED)
    set_target_properties(tvm PROPERTIES
        IMPORTED_LOCATION "${TVM_PREBUILD_PATH}/libtvm.so"
        INTERFACE_INCLUDE_DIRECTORIES "${TVM_PREBUILD_PATH}/../include"
    )
    add_library(tvm_runtime SHARED IMPORTED)
    set_target_properties(tvm_runtime PROPERTIES
        IMPORTED_LOCATION "${TVM_PREBUILD_PATH}/libtvm_runtime.so"
        INTERFACE_INCLUDE_DIRECTORIES "${TVM_PREBUILD_PATH}/../include"
    )
else()
    message(STATUS "Building TVM from source")
    add_subdirectory(${TVM_SOURCE_DIR} tvm EXCLUDE_FROM_ALL)
endif()
```

This configuration checks if `TVM_PREBUILD_PATH` is defined:

- If it is, it treats TVM as a prebuilt library and links against it.
- If not, it adds TVM as a subdirectory to build it from source.

**Building Your Custom Extensions**

```cmake
file(GLOB_RECURSE CUSTOM_SRCS src/*.cc)
add_library(custom_objs OBJECT ${CUSTOM_SRCS})
set(CUSTOM_INCLUDES
    ${TVM_SOURCE_DIR}/include
    ${TVM_SOURCE_DIR}/src
    ${TVM_SOURCE_DIR}/3rdparty/dlpack/include
    ${TVM_SOURCE_DIR}/3rdparty/dmlc-core/include
)
target_include_directories(custom_objs PRIVATE ${CUSTOM_INCLUDES})
```

This sets up your custom source files and includes TVM's headers for compilation.

### Handling Python Bindings

To ensure that your custom extensions are properly registered with TVM's Python API:

- **Load Custom Libraries Before Importing TVM**: Load your custom shared libraries before importing `tvm` in Python to ensure that the global functions are registered.

- **Modify `__init__.py`**: In your Python package's `__init__.py`, handle environment variables and library loading:

  ```python
  import os
  import sys

  # Set up environment variables
  os.environ['TVM_LIBRARY_PATH'] = '/path/to/your/libs'

  # Load custom libraries
  from .libinfo import find_lib_path
  _LIBS = find_lib_path()
  for lib in _LIBS:
      tvm.lib.load_library(lib)

  import tvm
  ```

### Custom Library Loader (`libinfo.py`)

Implement a custom library finder that locates your shared libraries at runtime.

```python
import os

def find_lib_path():
    curr_path = os.path.dirname(os.path.abspath(__file__))
    lib_path = []
    for lib in ['libyour_project.so', 'libyour_project.dylib', 'your_project.dll']:
        full_path = os.path.join(curr_path, lib)
        if os.path.exists(full_path):
            lib_path.append(full_path)
    if not lib_path:
        raise RuntimeError("Cannot find your_project library")
    return lib_path
```

## Examples

### Adding a Custom Pass

**C++ Implementation (`src/my_pass.cc`):**

```cpp
#include <tvm/tir/transform.h>

namespace tvm {
namespace tir {
namespace transform {

tvm::transform::Pass MyCustomPass() {
    auto pass_func = [](PrimFunc f, IRModule m, PassContext ctx) {
        // Implement your pass logic here
        return f;
    };
    return tvm::transform::CreatePrimFuncPass(pass_func, 0, "MyCustomPass", {});
}

TVM_REGISTER_GLOBAL("my_project.transform.MyCustomPass")
.set_body_typed(MyCustomPass);

}  // namespace transform
}  // namespace tir
}  // namespace tvm
```

**Python Usage:**

```python
import tvm
import your_project.transform

mod = ...  # your IRModule
mod = your_project.transform.MyCustomPass()(mod)
```

### Adding a Custom Code Generator

Similar to adding a pass, you can register custom code generators and use them in your compilation flow.

## Benefits of This Approach

- **Upstream Compatibility**: Easily update TVM to the latest version without merging conflicts.
- **Ease of Collaboration**: Contributors can work on your project without dealing with a modified TVM.
- **Modularity**: Separate concerns between your extensions and TVM's core functionalities.
- **Reduced Maintenance**: Focus on your project's logic without worrying about TVM's internals.
- **Community Integration**: Facilitate sharing and potentially upstreaming your extensions.


## Acknowledgments

This project is inspired by the modular build approach used in [MLC-LLM](https://github.com/mlc-ai/mlc-llm). Special thanks to the Apache TVM community for providing a powerful and flexible deep learning compiler infrastructure.
