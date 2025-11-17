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
│  - Omega variant wrappers (Array, Dictionary, etc.)  │
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
│    omega/torus Native Libraries       │
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
#* All functions with their argument and return types
#* Types (enums, unions, interfaces)
#* Classes and their properties
# '''Module Population''': 
#* Populates internal lists while handling namespaces
#* Updates <code>__all__</code> metadata for IntelliSense support
#* Adds common types to the module
# '''Lazy Generation''': Functions and interfaces are not immediately added to the module; they are generated on-demand when first accessed

==== Supported Components ====

omegapython2 supports multiple predefined components. Each component corresponds to an omega/torus library that can be loaded independently.

You can see the full list of supported components here: //p4/depot/omegapython2/components.json

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
|}

'''Complex Types:''' For date, dictionary, and array types, omegapython2 provides wrapper classes (<code>OADate</code>, <code>Dictionary</code>, <code>Array</code>) that encapsulate omega variants and provide Pythonic interfaces.

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

* '''ApiValueObject''': Wraps omega value types (int, double, bool, void, date). These objects contain the variant directly.
* '''ApiReferenceObject''': Wraps omega reference types (string, binary, dictionaries) with reference counting. Contains an <code>ApiValueObject</code> plus a deallocator.
* '''ApiContainerObject''': Wraps omega array types with element type and dimension information. Contains an <code>ApiReferenceObject</code> plus metadata about the array structure.

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
* Uses NumPy's <code>ascontiguousarray()</code> to obtain a contiguous memory pointer
* <code>_invoker.array_create_from_pointer()</code> copies the memory contents
* The copied memory is passed to the omega variant, which takes ownership

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

The <code>OADate</code> type wraps omega date variants. On the Python side, <code>OADate</code> provides conversion methods to and from <code>datetime.datetime</code> objects, enabling seamless integration with Python's standard datetime library.

=== Memory Management ===

Memory safety in omegapython2 is addressed at two levels: omega variant memory and Python object memory.

==== Python GC Integration ====

API objects are Python objects managed by Python's garbage collector. Deallocation hooks are implemented for each API object type:

* '''ApiValueObject dealloc''': Destroys the wrapped omega variant and deallocates the Python object
* '''ApiReferenceObject dealloc''': Decrements the omega variant's reference count (allowing omega framework to manage cleanup) and deallocates the Python object
* '''ApiContainerObject dealloc''': Inherits reference object deallocation behavior

==== Omega Reference Counting ====

The omega framework provides reference counting for all omega objects. The Python binding leverages this system:
* Reference objects maintain a <code>ref_count_t deallocator</code>
* When Python GC deallocates a reference object, the reference count is decremented
* Omega framework manages the actual memory lifecycle

==== Python Object Memory Safety ====

To prevent memory leaks from Python object reference counting errors in the C++ extension, omegapython2 uses the <code>py_obj_ref_t</code> wrapper class:

* '''RAII Pattern''': <code>py_obj_ref_t</code> provides automatic resource management for Python objects (PyObject*) using RAII (Resource Acquisition Is Initialization)
* '''Automatic Reference Management''': When <code>py_obj_ref_t</code> goes out of scope, it automatically decrements the Python object's reference count
* '''Convenience Functions''':
** Create from new reference: When receiving a new reference from Python C API (e.g., from <code>PyLong_FromLong()</code>)
** Create from borrowed reference: When receiving a borrowed reference that needs to be retained (increments reference count)

This ensures proper Python reference counting throughout the C++ extension code, preventing memory leaks on the Python side.

==== Memory Leak Detection ====

'''Omega Side:''' Python tests validate memory integrity by checking omega object counts before and after each test execution. This ensures no memory leaks occur from omega variants across the Python-C++ boundary.

<syntaxhighlight lang="python">
ref_count_before = omega.core.ref_count()
run_test()
ref_count_after = omega.core.ref_count()
assert ref_count_before == ref_count_after
</syntaxhighlight>

'''Note:''' The reference count checker runs in debug builds only.

'''Python Side:''' The <code>py_obj_ref_t</code> wrapper provides automated memory safety through RAII, preventing Python object leaks in the C++ extension.

'''Future Improvements:''' Integration with Valgrind for comprehensive memory leak detection across both Python and native code boundaries.

=== Concurrency - GIL Handling ===

The Global Interpreter Lock (GIL) is released before making omega library calls, allowing:
* Better performance for CPU-bound omega operations
* Concurrent execution of multiple omega library calls from different Python threads
* Prevention of Python interpreter blocking during long-running omega operations

Implementation pattern in C++ extension:
<syntaxhighlight lang="cpp">
// Extract omega variants from API objects (with GIL held)
omega::core::api::variant arg = extract_variant(api_object);

// Release GIL before calling omega library
Py_BEGIN_ALLOW_THREADS
omega::core::api::variant result = omega_library_call(arg);
Py_END_ALLOW_THREADS

// Wrap result in API object (with GIL held)
return create_api_object(result);
</syntaxhighlight>

== Data Flow Example ==

=== Complete Call Flow for <code>omega.core.func("hello")</code> ===

# '''User calls function''' in Python: <code>result = omega.core.func("hello")</code>
# '''Python layer''': Lazy generation executes (if first call) to create the function
# '''Packer''': <code>pack_str("hello")</code> is called
#* Creates <code>ApiReferenceObject</code> wrapping omega string variant
#* Returns PyObject*
# '''C++ Extension''': <code>call_library(entry_point, packed_arg)</code>
#* Extracts omega variant from API object (with GIL held)
#* Releases GIL
#* Calls native omega library function
#* Reacquires GIL
#* Wraps the returned omega variant in an API object
# '''Unpacker''': <code>unpack_str(result)</code> is called
#* Extracts the omega variant from the API object
#* Converts it to a PyUnicode object
#* Returns the Python string
# '''User receives''' Python string result

'''Note:''' Both packer and unpacker functions are implemented in the C++ extension module (<code>_omega_interface</code>).
