= omegapython2 Internal Architecture =

== Overview ==

omegapython2 is a Python binding for omega/torus components that provides a bridge between Python code and native omega/torus C++ libraries. The architecture consists of multiple layers that handle library loading, dynamic code generation, type conversion, and memory management.

== Architecture Layers ==

<pre>
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
</pre>

== Component Details ==

=== Python Layer ===

The Python layer provides the user-facing API and handles dynamic module construction.

==== Library Loading Process ====

<syntaxhighlight lang="python">
import omega
omega.core.library_paths({"omega.core": OC_DLL_PATH})
</syntaxhighlight>

When <code>library_paths()</code> is called, the following steps occur:

# '''Create Invoker''': Calls <code>_omega_interface.create_invoker()</code> to load the DLL and receive an invoker object
# '''API Discovery''': Calls <code>describe_api()</code> library function that returns metadata about:
#* All functions in the library
#* Types (enums, unions, interfaces)
#* Classes and their properties
# '''Module Population''': 
#* Populates internal lists while handling namespaces
#* Updates <code>__all__</code> metadata for IntelliSense support
#* Adds common types to the module
# '''Lazy Generation''': Functions and classes are not immediately added to the module; they are generated on-demand through <code>exec()</code> when first accessed

==== Supported Components ====

omegapython2 supports multiple predefined components including:
* <code>omega.core</code>
* <code>torus.data</code>
* And other omega/torus components

=== Packer/Unpacker Layer ===

The packer/unpacker layer handles type conversion between Python objects and omega variants.

==== Example Function Call Flow ====

For an IDL function defined as <code>string func(string s)</code>:

<syntaxhighlight lang="python">
# Generated code (simplified)
entry_point = omega.core._invoker.get_entrypoint("func")

def func(s: str) -> str:
    packed = omega.core._invoker.pack_str(s)
    res = omega.core._invoker.call_library(entry_point, packed)
    return omega.core._invoker.unpack_str(res)
</syntaxhighlight>

==== Type Mapping ====

{| class="wikitable"
! Omega Type !! Python Type !! API Object Type !! Pack/Unpack
|-
| binary || PyBytes || Reference || pack_binary / unpack_binary
|-
| bool || PyBool || Value || pack_bool / unpack_bool
|-
| double || PyFloat || Value || pack_double / unpack_double
|-
| int || PyLong || Value || pack_int / unpack_int
|-
| string || PyUnicode || Reference || pack_str / unpack_str
|-
| void || Py_None || Value || pack_void / unpack_void
|-
| date || datetime || Value (OADate) || pack_date / unpack_date
|}

=== API Objects (C++ Extension) ===

The C++ extension module (<code>_omega_interface</code>) defines wrapper structures around omega variants.

==== Object Hierarchy ====

<syntaxhighlight lang="cpp">
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
</syntaxhighlight>

* '''ApiValueObject''': Wraps omega value types (int, double, bool, void, date)
* '''ApiReferenceObject''': Wraps omega reference types (string, binary) with reference counting
* '''ApiContainerObject''': Wraps omega container types (arrays, dictionaries) with element type information

=== Type System Implementation ===

==== Primitive Types ====

Value types and reference types are handled through pack/unpack functions in the C++ extension:
* '''pack_*''': Takes Python object → creates omega variant → wraps in API object → returns
* '''unpack_*''': Takes API object → extracts omega variant → converts to Python object → returns

==== Key Type ====

Keys are cached strings in the omega framework. The Python layer provides a <code>Key</code> class that wraps the underlying API object.

==== Unions ====

Unions support multiple types and automatically resolve to the appropriate type on return.

'''IDL Example:'''
<syntaxhighlight lang="cpp">
union test_union {
    int;
    string;
}

test_union testing_union(test_union input);
</syntaxhighlight>

'''Generated Python Code:'''
<syntaxhighlight lang="python">
def testing_union(input: test_union):
    arg0 = omega.core.test_union(input)._api_object
    res_type, res_value = omega.core._invoker.call_library(testing_union_symbol, arg0)
    # res_type determines which unpacker to use
    return unpacker[res_type](res_value)
</syntaxhighlight>

When calling <code>x = testing_union("abc")</code>, the function returns a string (not a union), making union handling transparent and convenient.

==== Dictionary ====

The <code>Dictionary</code> class wraps omega dictionary variants and provides convenience functions for working with Python's native dict objects.

==== Arrays ====

The <code>Array</code> class wraps omega array variants. Arrays are typed based on dimension and element type.

'''Array Creation Methods:'''

;From Python Lists/Tuples
* <code>_invoker.array_create_from_list()</code> creates omega array variants from Python sequences
* Implementation: Creates <code>std::vector</code>, populates by casting elements, passes memory to omega variant

;From NumPy Arrays
* Uses NumPy's <code>ascontiguousarray()</code> to get memory pointer
* <code>_invoker.array_create_from_pointer()</code> copies memory into omega variant
* Omega variant takes ownership of the memory

==== Classes and Interfaces ====

For IDL definitions like:

<syntaxhighlight lang="cpp">
interface test_object {}

class test_obj_f : test_object {
    double double_val;
}
</syntaxhighlight>

A builder function is lazily generated:

<syntaxhighlight lang="python">
def TestObjF(DoubleVal: float) -> omega.core.test_object:
    res = omega.core._invoker.class_create({
        "class": test_obj_f_symbol,
        "DoubleVal": omega.core._invoker.pack_double(DoubleVal)
    })
    return omega.core.test_object(res)
</syntaxhighlight>

==== OADate ====

The <code>OADate</code> type wraps omega date variants. On the Python side, it appears as <code>datetime.datetime</code> objects, while in the C++ extension it's stored as <code>ApiValueObject</code> containing a float.

=== Memory Management ===

==== Python GC Integration ====

API objects are Python objects managed by Python's garbage collector. Deallocation hooks are implemented for each API object type:

* '''ApiValueObject dealloc''': Destroys the wrapped omega variant
* '''ApiReferenceObject dealloc''': Decrements the omega framework's reference count
* '''ApiContainerObject dealloc''': Inherits reference object deallocation

==== Omega Reference Counting ====

The omega framework provides reference counting for all omega objects. The Python binding leverages this system:
* Reference objects maintain a <code>ref_count_t deallocator</code>
* When Python GC deallocates a reference object, the reference count is decremented
* Omega framework manages the actual memory lifecycle

==== Memory Leak Detection ====

Python tests validate memory integrity by checking omega object counts before and after each test execution. This ensures no memory leaks occur across the Python-C++ boundary.

=== Concurrency - GIL Handling ===

The Global Interpreter Lock (GIL) is released before making omega library calls, allowing:
* Better performance for CPU-bound omega operations
* Concurrent execution of multiple omega library calls from different Python threads
* Prevention of Python interpreter blocking during long-running omega operations

Implementation pattern:
<syntaxhighlight lang="cpp">
// In C++ extension
Py_BEGIN_ALLOW_THREADS
// Call omega library function
result = omega_library_call(args);
Py_END_ALLOW_THREADS
</syntaxhighlight>

== Data Flow Example ==

=== Complete Call Flow for <code>omega.core.func("hello")</code> ===

# '''User calls function''' in Python: <code>result = omega.core.func("hello")</code>
# '''Python layer''': Lazy generation executes (if first call) to create the function
# '''Packer''': <code>pack_str("hello")</code> is called
#* Creates <code>ApiReferenceObject</code> wrapping omega string variant
#* Returns PyObject*
# '''C++ Extension''': <code>call_library(entry_point, packed_arg)</code>
#* Releases GIL
#* Extracts omega variant from API object
#* Calls native omega library function
#* Reacquires GIL
#* Returns result as API object
# '''Unpacker''': <code>unpack_str(result)</code> is called
#* Extracts omega variant from API object
#* Converts to Python string
#* Returns PyUnicode object
# '''User receives''' Python string result

== Key Design Principles ==

# '''Lazy Evaluation''': Functions and classes are generated only when first accessed, improving load times
# '''Type Safety''': Strong type mapping between Python and omega ensures correct data conversion
# '''Memory Safety''': Dual memory management (Python GC + omega ref counting) prevents leaks
# '''Performance''': GIL release during library calls enables better concurrency
# '''Convenience''': Union type resolution and array creation from Python/NumPy structures provide ergonomic APIs
# '''Transparency''': API discovery via <code>describe_api()</code> keeps bindings in sync with library changes

== Extension Points ==

The architecture supports extension through:
* Adding new component modules (following the <code>omega.core</code> pattern)
* Implementing additional type wrappers (similar to <code>Dictionary</code>, <code>Array</code>, <code>OADate</code>)
* Custom packer/unpacker pairs for specialized types
* Enhanced memory tracking and debugging hooks

== Summary ==

omegapython2's architecture provides a robust, efficient bridge between Python and omega/torus native libraries. The layered design separates concerns clearly: the Python layer handles user interaction and lazy generation, the packer/unpacker layer manages type conversion, the C++ extension provides performance-critical operations and memory management, and the GIL handling ensures good concurrency characteristics. This design enables Python developers to work with omega/torus components using Pythonic idioms while maintaining the performance and type safety of the underlying C++ libraries.
