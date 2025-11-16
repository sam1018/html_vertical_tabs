# omegapython2 Internal Architecture

## Overview

omegapython2 is a Python binding for omega/torus components that provides a bridge between Python code and native omega/torus C++ libraries. The architecture consists of multiple layers that handle library loading, dynamic code generation, type conversion, and memory management.

## Architecture Layers

```
┌─────────────────────────────────────────────┐
│         Python User Code                    │
│  (import omega; omega.core.func())          │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│         Python Layer                        │
│  - Lazy function/class generation           │
│  - Namespace handling                       │
│  - Type wrappers (Array, Dictionary, etc.)  │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│    Packer/Unpacker Layer                    │
│  - Type conversion (Python ↔ omega)         │
│  - pack_* / unpack_* functions              │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│    C++ Extension (_omega_interface)         │
│  - Invoker object                           │
│  - API Objects (Value/Reference/Container)  │
│  - Memory management                        │
│  - GIL handling                             │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│    omega/torus Native Libraries (DLLs)      │
└─────────────────────────────────────────────┘
```

## Component Details

### 1. Python Layer

The Python layer provides the user-facing API and handles dynamic module construction.

#### Library Loading Process

```python
import omega
omega.core.library_paths({"omega.core": OC_DLL_PATH})
```

When `library_paths()` is called, the following steps occur:

1. **Create Invoker**: Calls `_omega_interface.create_invoker()` to load the DLL and receive an invoker object
2. **API Discovery**: Calls `describe_api()` library function that returns metadata about:
   - All functions in the library
   - Types (enums, unions, interfaces)
   - Classes and their properties
3. **Module Population**: 
   - Populates internal lists while handling namespaces
   - Updates `__all__` metadata for IntelliSense support
   - Adds common types to the module
4. **Lazy Generation**: Functions and classes are not immediately added to the module; they are generated on-demand through `exec()` when first accessed

#### Supported Components

omegapython2 supports multiple predefined components including:
- `omega.core`
- `torus.data`
- And other omega/torus components

### 2. Packer/Unpacker Layer

The packer/unpacker layer handles type conversion between Python objects and omega variants.

#### Example Function Call Flow

For an IDL function defined as `string func(string s)`:

```python
# Generated code (simplified)
entry_point = omega.core._invoker.get_entrypoint("func")

def func(s: str) -> str:
    packed = omega.core._invoker.pack_str(s)
    res = omega.core._invoker.call_library(entry_point, packed)
    return omega.core._invoker.unpack_str(res)
```

#### Type Mapping

| Omega Type | Python Type | API Object Type | Pack/Unpack |
|------------|-------------|-----------------|-------------|
| binary     | PyBytes     | Reference       | pack_binary / unpack_binary |
| bool       | PyBool      | Value           | pack_bool / unpack_bool |
| double     | PyFloat     | Value           | pack_double / unpack_double |
| int        | PyLong      | Value           | pack_int / unpack_int |
| string     | PyUnicode   | Reference       | pack_str / unpack_str |
| void       | Py_None     | Value           | pack_void / unpack_void |
| date       | datetime    | Value (OADate)  | pack_date / unpack_date |

### 3. API Objects (C++ Extension)

The C++ extension module (`_omega_interface`) defines wrapper structures around omega variants.

#### Object Hierarchy

```cpp
struct ApiValueObject {
    PyObject_HEAD;
    omega::core::api::variant object;
};

struct ApiReferenceObject {
    ApiValueObject underlying;
    omega::core::api::ref_count_t deallocator;
};

struct ApiContainerObject {
    ApiReferenceObject underlying;
    omega::core::api::type_id element_type;
    size_t rank;
    size_t length;
};
```

- **ApiValueObject**: Wraps omega value types (int, double, bool, void, date)
- **ApiReferenceObject**: Wraps omega reference types (string, binary) with reference counting
- **ApiContainerObject**: Wraps omega container types (arrays, dictionaries) with element type information

### 4. Type System Implementation

#### Primitive Types

Value types and reference types are handled through pack/unpack functions in the C++ extension:
- **pack_***: Takes Python object → creates omega variant → wraps in API object → returns
- **unpack_***: Takes API object → extracts omega variant → converts to Python object → returns

#### Key Type

Keys are cached strings in the omega framework. The Python layer provides a `Key` class that wraps the underlying API object.

#### Unions

Unions support multiple types and automatically resolve to the appropriate type on return.

**IDL Example:**
```cpp
union test_union {
    int;
    string;
}

test_union testing_union(test_union input);
```

**Generated Python Code:**
```python
def testing_union(input: test_union):
    arg0 = omega.core.test_union(input)._api_object
    res_type, res_value = omega.core._invoker.call_library(testing_union_symbol, arg0)
    # res_type determines which unpacker to use
    return unpacker[res_type](res_value)
```

When calling `x = testing_union("abc")`, the function returns a string (not a union), making union handling transparent and convenient.

#### Dictionary

The `Dictionary` class wraps omega dictionary variants and provides convenience functions for working with Python's native dict objects.

#### Arrays

The `Array` class wraps omega array variants. Arrays are typed based on dimension and element type.

**Array Creation Methods:**

a. **From Python Lists/Tuples**: 
   - `_invoker.array_create_from_list()` creates omega array variants from Python sequences
   - Implementation: Creates `std::vector`, populates by casting elements, passes memory to omega variant

b. **From NumPy Arrays**:
   - Uses NumPy's `ascontiguousarray()` to get memory pointer
   - `_invoker.array_create_from_pointer()` copies memory into omega variant
   - Omega variant takes ownership of the memory

#### Classes and Interfaces

For IDL definitions like:

```cpp
interface test_object {}

class test_obj_f : test_object {
    double double_val;
}
```

A builder function is lazily generated:

```python
def TestObjF(DoubleVal: float) -> omega.core.test_object:
    res = omega.core._invoker.class_create({
        "class": test_obj_f_symbol,
        "DoubleVal": omega.core._invoker.pack_double(DoubleVal)
    })
    return omega.core.test_object(res)
```

#### OADate

The `OADate` type wraps omega date variants. On the Python side, it appears as `datetime.datetime` objects, while in the C++ extension it's stored as `ApiValueObject` containing a float.

### 5. Memory Management

#### Python GC Integration

API objects are Python objects managed by Python's garbage collector. Deallocation hooks are implemented for each API object type:

- **ApiValueObject dealloc**: Destroys the wrapped omega variant
- **ApiReferenceObject dealloc**: Decrements the omega framework's reference count
- **ApiContainerObject dealloc**: Inherits reference object deallocation

#### Omega Reference Counting

The omega framework provides reference counting for all omega objects. The Python binding leverages this system:
- Reference objects maintain a `ref_count_t deallocator`
- When Python GC deallocates a reference object, the reference count is decremented
- Omega framework manages the actual memory lifecycle

#### Memory Leak Detection

Python tests validate memory integrity by checking omega object counts before and after each test execution. This ensures no memory leaks occur across the Python-C++ boundary.

### 6. Concurrency - GIL Handling

The Global Interpreter Lock (GIL) is released before making omega library calls, allowing:
- Better performance for CPU-bound omega operations
- Concurrent execution of multiple omega library calls from different Python threads
- Prevention of Python interpreter blocking during long-running omega operations

Implementation pattern:
```cpp
// In C++ extension
Py_BEGIN_ALLOW_THREADS
// Call omega library function
result = omega_library_call(args);
Py_END_ALLOW_THREADS
```

## Data Flow Example

### Complete Call Flow for `omega.core.func("hello")`

1. **User calls function** in Python: `result = omega.core.func("hello")`

2. **Python layer**: Lazy generation executes (if first call) to create the function

3. **Packer**: `pack_str("hello")` is called
   - Creates `ApiReferenceObject` wrapping omega string variant
   - Returns PyObject*

4. **C++ Extension**: `call_library(entry_point, packed_arg)`
   - Releases GIL
   - Extracts omega variant from API object
   - Calls native omega library function
   - Reacquires GIL
   - Returns result as API object

5. **Unpacker**: `unpack_str(result)` is called
   - Extracts omega variant from API object
   - Converts to Python string
   - Returns PyUnicode object

6. **User receives** Python string result

## Key Design Principles

1. **Lazy Evaluation**: Functions and classes are generated only when first accessed, improving load times
2. **Type Safety**: Strong type mapping between Python and omega ensures correct data conversion
3. **Memory Safety**: Dual memory management (Python GC + omega ref counting) prevents leaks
4. **Performance**: GIL release during library calls enables better concurrency
5. **Convenience**: Union type resolution and array creation from Python/NumPy structures provide ergonomic APIs
6. **Transparency**: API discovery via `describe_api()` keeps bindings in sync with library changes

## Extension Points

The architecture supports extension through:
- Adding new component modules (following the `omega.core` pattern)
- Implementing additional type wrappers (similar to `Dictionary`, `Array`, `OADate`)
- Custom packer/unpacker pairs for specialized types
- Enhanced memory tracking and debugging hooks

## Summary

omegapython2's architecture provides a robust, efficient bridge between Python and omega/torus native libraries. The layered design separates concerns clearly: the Python layer handles user interaction and lazy generation, the packer/unpacker layer manages type conversion, the C++ extension provides performance-critical operations and memory management, and the GIL handling ensures good concurrency characteristics. This design enables Python developers to work with omega/torus components using Pythonic idioms while maintaining the performance and type safety of the underlying C++ libraries.
