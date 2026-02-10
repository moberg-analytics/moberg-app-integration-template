# Moberg Clinical Platform (MCP) App Integration Template

### Clients must fork this repository and **must not** create a pull request.

This repository is a **template codebase for third-party integrations with Moberg Analytics**.
It demonstrates how to build, test, and package a Python extension module as a **compiled shared object (`.so`)** for delivery to Moberg Analytics.

Please feel free to modify this README to reflect the specifics of your module.

## Repo Structure

```
tests/                 --> Input and output test cases (pytest)
main.py               --> Example script showing how to run the module
module_name.so        --> Compiled shared object (Python extension module)
LICENSE               --> Update according to your licensing requirements
test_file.csv         --> Example CSV input for testing and demo use cases
```

> **Note:** The `.so` file is the artifact that will ultimately be delivered to Moberg Analytics.

## Introduction

This module integrates with the **Moberg Clinical Platform (MCP)** by exposing a compiled Python extension that can be imported directly into the MCP runtime environment.

The goal of this repository is to:

* Provide a reference structure for third-party analytics modules
* Demonstrate how to compile Python code into a `.so` file
* Ensure reproducibility and compatibility with MCP's deployment environment

## Setup & Installation

### Requirements

Ensure your environment matches the versions below to avoid compatibility issues:

```
Python 3.8.19
numpy==1.20.3
pandas==2.0.2
scipy==1.10.0
pytest==8.1.1
cython==0.29.21
```

> ⚠️ **Python version note**
> Moberg Analytics is **currently migrating from Python 3.8 to Python 3.12**.
> This means:
>
> * Newer releases may require updated dependency versions
> * Compiled modules (`.so` files) **must be rebuilt** for Python 3.12
> * Package constraints may change in the near future to maintain compatibility
>
> We recommend avoiding hard-coded assumptions about Python internals and periodically validating your build against newer Python versions.

### Installation Steps

1. **Clone the repository**

```bash
git clone https://github.com/your-repo/moberg-analytics-integration.git
cd moberg-analytics-integration
```

2. **Install dependencies**

```bash
pip install -r requirements.txt
```

## Building the Module (`.so` File)

Moberg Analytics expects modules to be delivered as **compiled Python extension files** (`.so`), for example:

```
sd_module.cpython-38-x86_64-linux-gnu.so
```

## ⚠️ Multiprocessing Compatibility (Important)

Moberg Analytics uses **multiprocessing** within its runtime environment.

Because of this:

> **Third-party compiled modules (`.so` files MUST NOT include their own multiprocessing, threading pools, or process forking logic.**

Including multiprocessing inside your compiled module will **conflict with MCP’s execution model** 

Do not include the following:
* `multiprocessing`
* `joblib`
* `concurrent.futures.ProcessPoolExecutor`
* native thread pools in C/C++
* OpenMP / MKL parallelism
* Other multiprocessing libraries/modules

Including anything from the list above may result in the following issues:
* Deadlocks
* Hung workers
* Crashed containers
* Silent performance degradation
* Non-deterministic behavior

### Required Guidelines

When building your module:

* ❌ Do NOT use `multiprocessing` or spawn processes internally
* ❌ Do NOT manage worker pools inside the module
* ❌ Do NOT rely on implicit parallelism from native libraries

Instead:

* ✅ Assume your code is already running inside a distributed environment
* ✅ Write modules as **single-process, single-threaded functions**
* ✅ Allow MCP to handle all parallel execution

If your algorithm is computationally intensive, it should be structured so that:

* Individual calls operate on **one unit of work**
* Parallelism is handled externally by Moberg’s orchestration layer

### Native Library Note

Some scientific libraries (NumPy, SciPy, BLAS/MKL) may enable multithreading by default.

If used, please explicitly limit threads during build or runtime, for example:

```bash
export OMP_NUM_THREADS=1
export MKL_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
```

This ensures compatibility with MCP’s distributed scheduler.

### Recommended Build Approach (Cython)

1. **Create a Cython entry point**
   Example: `sd_module.pyx`

```cython
# sd_module.pyx
def run(input_path):
    return f"Processing file: {input_path}"
```

2. **Create a `setup.py` file**

```python
from setuptools import setup
from Cython.Build import cythonize

setup(
    name="sd_module",
    ext_modules=cythonize(
        "sd_module.pyx",
        language_level="3"
    ),
)
```

3. **Build the shared object**

```bash
python setup.py build_ext --inplace
```

4. **Verify output**
   After building, you should see a file similar to:

```
sd_module.cpython-38-x86_64-linux-gnu.so
```

This `.so` file is the **deliverable artifact** to Moberg Analytics.

> 🔐 **Important**
>
> * The `.so` file is **Python-version and architecture specific**
> * Builds must be performed on a compatible Linux environment
> * Python 3.8 and Python 3.12 builds are **not interchangeable**

## Usage

### Importing and Running the Module

Once compiled, the module can be imported like any standard Python package:

```python
import sd_module

result = sd_module.run("test_file.csv")
print(result)
```

## Testing

We use **pytest** for testing the integration.

Run all tests with:

```bash
pytest tests/
```

Tests should:

* Validate expected outputs
* Use the provided `test_file.csv`
* Confirm behavior matches documented assumptions

## Demo Input File

Please provide a representative demo input file (`test_file.csv`) that:

* Matches expected input format
* Covers common use cases
* Clearly documents any required or optional columns

## Delay in Response

If the module performs event detection or time-based analysis:

* Document any **expected delays**
* Specify whether outputs are real-time, near-real-time, or retrospective

This helps Moberg Analytics integrate the module appropriately within clinical and research workflows.

## Changelog

Document notable changes here, including:

* Version bumps
* Dependency updates
* Python compatibility changes (e.g., 3.8 → 3.12)
* Behavioral or output differences

## Contact

Please provide technical and support contact information here (email, organization, or repository link).
