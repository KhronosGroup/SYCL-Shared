# SYCL Modules

|                  |                                     |
| ---------------- | ----------------------------------- |
| Name             | SYCL Generalization                 |
| Date of creation | 3rd Jun 2019                        |
| Last updated     | 12th Dec  2019                      |
| Status           | Approved by SYCL working group      |
| Current revision | 9                                   |
| Dependencies     | SYCL Generalization                 |
| Available        | N/A                                 |
| Reply-to         | Ruyman Reyes <ruyman@codeplay.com>  |
| Original author  | Ruyman Reyes <ruyman@codeplay.com>  |
| Contributors     | Aksel Alpay <aksel.alpay@uni-heidelberg.de>, Victor Lomuller <victor@codeplay.com>, Toomas Remmelg <Toomas.remmelg@codeplay.com>, Morris Hafner <morris.hafner@codeplay.com> |

**The contents of this proposal are to encourage early feedback and are not ratified by the Khronos Group so do not constitute a formal specification.**

## Overview

While working on the proposal for *Generalization of the SYCL Next API*, 
a number of issues with the current SYCL 1.2.1 `program` class were found.
The current design of the class is too close to OpenCL and prevents
other backends that have different online compilation models to be
implemented reasonably.

## Motivation

The current SYCL program class has been designed purely for the OpenCL 1.2 
compilation model. 
The SYCL program class, like a `cl_program` object, has state. 
This makes the object not thread safe, being the only SYCL class with this
problem.
Error reporting via exception causes different exceptions being thrown depending
on the state of the class, which makes error recovering difficult.

The implementation of the `program` class, and how this maps to the internals
of the different SYCL implementations, is purely implementation defined. 

This causes problems for users trying understanding its functionality of some of
its functionality. A good example is the method `build_from_kernel_type`, which
is defined as the following  according to the SYCL 1.2.1 specification:

> Builds the SYCL kernel function defined
> by the type `kernelT` into the encapsulated `cl_program` with the compiler options
> specified by `buildOptions`, if this SYCL program is an OpenCL program.  Sets the
> state of this SYCL program to `program_state::linked`. Must throw an
> `invalid_object_error` SYCL exception if this program was not in the
> `program_state` ::none state when called. Must return a `compile_program_error`
> SYCL exception if the compilation fails.

User feedback and implementation experience have highlighted various issues,
some of them listed below:

1. Depending on the SYCL implementation, this method may build one or multiple
kernels into the same `cl_program`. SYCL implementations may create independent
compilation modules for each kernel or group them in the same one. The user can only check after the fact what has been built by querying using the function `has_kernel`.
2. Some users don't understand the difference between a SYCL program and a SYCL
application. The fact that you can build a SYCL program inside a SYCL 
application appears confusing to them, since, unless you come from an OpenCL
background, both are the same. One of the most frequent questions we have
for the program class is whether a SYCL application can have more than one
SYCL program object or only one.
3. The interface to pass compilation or link options to the `program` class
only works for SYCL implementations on top of backends that use compiler
flags as user-provided options. Other backends directly would not support
building programs at runtime at all (e.g. *Vulkan* or *hipSYCL*) which would
make the program class unimplementable on those platforms and cause them not to
be conformant.
4. Building kernels ahead of executing them using the type is confusing for users,
see example below:

```cpp
class myKernel;
int main() {
  ....
  cl::sycl::context ctxt(someDeviceSelector);
  cl::sycl::queue q(ctxt);
  cl::sycl::program foo(ctxt);
  
  foo.buld_with_kernel_type<myKernel>();  // A: What is this building?

  // B: If myKernel is a C++ type, why is get_kernel not noexcept?
  auto kernFoo = foo.get_kernel<myKernel>(); 

  q.submit([&](handler& h) {
    auto accA = bufA.get_access<access::mode::read>(h);
    // C: What is the relationship between kernFoo and the lambda?
    h.parallel_for<myKernel>(kernFoo, [=](item<> id) { acc[id] = id; });
  });
}

```

Although the user is writing code to build a kernel in point A, 
the kernel (in the form of the lambda) is not visible until point C.
It is also unclear from the signature why `get_kernel` can have side effects,
i.e. why is not a `const noexcept` method if the only parameter is known at
compile time.

The aim of this proposal is to address the aforementioned problems at the same
time that we generalize the SYCL specification to target multiple backends.

Although probably related to the design of the new program class, it is *not*
in the scope of the current wording for this proposal to address the
*lambda kernel naming issue*.

## Revisions

v 0.9:

* Added wording to clarify `device_image` relationship with `module_state`
* Using `module_state` consistently in the text
* Clarified behavior of `this_module`.

v 0.8:
* Minor editorial fixes for publication
* **sycl::invalid_object** thrown if joining empty vector of modules
* Clarifying wording about SYCL modules vs `this_module`
* Renamed `has_module` to `has_module_in`

v 0.7:
* Added `has_modules` and `has_any_module` functions to check whether
there are modules for a given context in the current translation unit.
* Acknowledge that two modules on two different contexts can contain images
for the same kernel.
* Added implementation note re: ISA device images and specialization constants
* `get_kernel_names()` and `has_kernels` are no longer noexcept 

v 0.6:
* Rename *source* modules to *input*

v 0.5:
* Further editorial fixes from internal feedback
* Added a definition of an empty module
* Definition of equality between two modules
* Compilation and link options are now on an "options" namespace

v 0.4:
* Various editorial fixes from internal feedback
* Added a "sycl::join" free function that joins multiple modules into one

v 0.3:this_module
* Various editorial fixes from internal feedback
* Added a new state for the module, *source*
* Improved wording for `this_module`, and defined interface for it
* Improvements to specialization constants interface

v 0.2: 
* Clarifications on module concepts
* Using templated `module` class for the different states rather than separate object types
* Introduce device images and device-image selector on dispatch
* Introducing the `use_module` method for the `handler`
* Introducing the `sycl::this_module` namespace its members `get()`, and `get_kernel`.
* Introduction of *specialization constants* as part of the *module* features.

## Summary of proposed changes

This proposal aims to entirely revamp the SYCL 1.2.1 program class and replace
it by type-safe, more generic *SYCL module* objects.
This proposal presents three types of *SYCL module* objects, depending on 
their contents: *input*, *object* or *executable*.
Factory functions in the *sycl* namespace provide constructors for each type
of module. 

A SYCL *module* is a high-level representation of a set of functions,
symbols and metadata that is meant for execution on one or multiple *SYCL
devices* grouped in a *SYCL context*.
A SYCL *module* contains a number of *device images*. Each *device image*
contains a representation of the same set of functions, symbols and metadata,
but potentially represented in different formats. 
The *device images* stored in a given *SYCL module* are implementation-defined,
and typically depend on the compilation options provided by the user.
For example, a *SYCL module* **foo** contains the function **bar**.
The *SYCL module* for an implementation that supports OpenCL could contain two 
*device images*, one with an SPIR-V IR and another with an ISA for a specific
device. Both *device images* contain the function **foo**, represented in 
different formats.
Note that the type of *device images* contained in a module is implementation defined,
and it does not affect the `state` of the SYCL module.
The SYCL implementation will retrieve those `device images` that are in the requested `state`, 
but it is implementation defined whether they are generated on request or not.
In particular, if a SYCL context has *online compilation* capabilities, 
the implementation can perform the desired transformations on the fly.
If there is no *online compilation*, this is not expected.

An *empty SYCL module* contains no *device images*.
Two *SYCL module* objects are said to be equal if and only if the following
conditions are true:
* Both *SYCL module* objects are bound to the same *SYCL context*
* Both *SYCL module* objects are on the same state
* The *device images* contained in both modules are the same.

There are three types of *SYCL module* objects, depending on the state.

A *input* module, `sycl::module<sycl::module_state::input>` contains
*device images* in raw format from either the result of the SYCL device
compiler output or some backend-specific factory function.
*input* modules cannot be dispatched or linked directly.
A *SYCL object module* needs to be constructed from a *input* module, which
can then be linked to create an executable module.
Functions exist to skip the intermediate step and directly provide a 
*SYCL executable module* from a *input* module whenever possible.

An *object module* `sycl::module<sycl::module_state::object>` may contain also 
*kernels*, symbols or device functions.
*object module* objects cannot be dispatched to a device, and require 
*linking* to ensure all symbols are resolved. 
Performing a *linking* operation on an *object module* generates an 
*executable module*.
Linking two modules with specialization constants preserves the union all
specialization constants in all modules, only if its intersection is empty.
Otherwise, the values for the specialization constant are undefined.

A SYCL *module* or *executable module* 
(`sycl::module<sycl::module_state::executable>`) represents a *SYCL module* 
that contains a number of images and metadata ready to be used on a 
device of the context associated with the *SYCL module*. 
This implies there is at least one *SYCL kernel* that can be obtained from
the module and be dispatched to a device in the associated context.
All *kernels* contained in a SYCL *module* must be able to be dispatched directly
to a *SYCL device* from the associated context without further compilation 
or linking stages.

**Implementation note**: The aim of the above paragraph is to provide 
users of the SYCL API a mechanism to ensure zero-overhead from 
device image building at the time of kernel dispatch.

A *SYCL kernel* remains the same as it is in the current SYCL specification:
An object that contains a function callable from the host application 
using a backend-specific mechanism that can execute on devices from the 
associated context. 
The kernel can be later dispatched using the usual SYCL dispatch API calls, or
the native type obtained for interoperability with backend objects.

The methods for building, compiling and linking a *module* become free functions
in the SYCL namespace. 
Generic functions to *compile*, *link* and *build* module objects exist.
Online compilation of sources is a backend-specific feature.
In particular, being able to online-compile an OpenCL C source becomes an
OpenCL specific feature, described on the OpenCL C backend document.

SYCL is a single-source programming model, hence, for each translation unit, the SYCL
*device compiler* will produce at least one SYCL *module* in a particular *state*,
storing *device_images* for all *kernels* and associated functions and symbols for each translation unit.
This module can be accessed through the `this_module` interface, described below.
Users can create their own modules by using interoperability functionality, e.g, loading
an OpenCL program or a SPIR-V binary.

**Implementation Note** Some implementations in some cases may decide to split all kernels in a single translation unit into different compilation modules. This compilation modules are implementation-defined and not visible to SYCL users. To a SYCL user, SYCL *module* objects stores different *device images* for a set of kernels, irrespectively of the internal implementation-layout.

One (or more) *object modules* can be linked to produce an *executable module*
irrespectively of how the kernels in those modules are organized internally into compilation
modules, as long as all the references between all the linked modules are solved.
If the linking of two (or more) *object modules* does not produce an *executable module*, 
where all contained images are ready for dispatch and without undefined references, the 
the linking must fail and throw an exception.

**Implementation note**: Implementations may chose to store each device kernel
into its own compilation module with its own dependencies. Linking together two
SYCL modules must not cause an invalid redefinition of symbols.

Modules with different images for a given context can be joined together into
a new one that contains a copy all device images using the "sycl::join" 
operation. This enables users to combine different versions of the same code
under the same module, and use that for dispatch later on.

The module associated with the current translation unit, and used by default
for kernel dispatch, can be obtained with the free function 
*`sycl::this_module::get(context)`*  

The instance of the *SYCL module* retrieved is bound to the given *SYCL context* object.
The *SYCL module* will contain *device images* that can be executed by the
devices in the *SYCL context*.
For example, if the *SYCL context* contains OpenCL devices, the *device images*
could be *SPIR-V* and native ISA binaries.
Note that, different module objects for different *SYCL context* can contain
device images with the same kernels.

When dispatching kernels from within *SYCL command groups*, the *SYCL runtime*
will use the module returned by *this_module* to find the desired kernel.
The default process used to obtain the kernel image is as follows:
1. If `this_module::get<module_state::executable>()` returns a module, find 
the kernel there, jump to step 5. Otherwise, next step.
2. If `this_module::get<module_state::object>()` returns a module, attempt
to link the module. If successful, find the kernel there, jump to step 5. 
Otherwise, next step.
3. If `this_module::get<module_state::input>` returns a module,
attempt to build the module. If successful, find the kernel there, jump to step
5. Otherwise, next step.
4. If no module has been found that contains the kernel, throw 
`sycl::kernel_not_found` exception.
5. Dispatch the kernel.

Users can override which module is used by passing an *executable module* to 
the *`use_module`* method of the *SYCL handler* object.
This module will be used to search for kernels instead of the default for the
current translation unit.

Further customization can be achieved by providing a *device image selector* 
function, which receives the *device image* begin and end and returns an image
to be used for dispatch. The contents of this selector are inherently
implementation-defined.

A *SYCL module* objects may expose *specialization constants*. 
Specialization is intended for constant objects that will not have known constant values 
until after initial generation of a module in an intermediate representation format (e.g. SPIR-V).
When a module contains *specialization constants*, the method 
*`sycl::module<sycl::module_state::input>::has_spec_constant()`* returns `true`.
Note that, although a module may expose *specialization constants*, not all *device images* inside the module may support them.
When *specialization constants* are available but the user doesn't set values on them, the default values are used.
The default values follow C++ initialization rules whenever not specified by 
users.
When *specialization constants* are not available in some images in a module,
but the user sets a value to them, they are set as an additional kernel
argument.

A *SYCL module* created from a *SYCL context* that runs on the *Host backend* contains
no *device images* and no *specialization constants*. 

## API definitions

### Module class

```cpp
namespace sycl {

enum class module_state {
  input,
  object,
  executable
};

template<module_state state>
class module {
  module() = delete;
  module(/* Implementation defined constructor */);

  public:

  using device_image = /* implementation defined */;
  using device_image_iterator = /* implementation defined */;

  context get_context() const noexcept;

  // Available if state is module_state::executable
  template <typename KernelNameT>
  kernel get_kernel() const;
  kernel get_kernel(std::string kernelName) const;

  // Available if  state is module_state::executable
  template <typename T>
  bool has_kernel() const noexcept;
  bool has_kernel(std::string kernelName) const;

  // Available if state is module_state::executable
  /* get_kernel_names.
   * Returns an std::vector where each entry is a kernel name
   * provided by the module.
   * Only names of kernels known at compile-time are guaranteed
   * to be listed.
   */
  std::vector<std::string> get_kernel_names() const;

  /* begin.
   * Returns an iterator to the first device image contained in the module 
   */
  device_image_iterator begin() const;

  /* end.
   * Returns an iterator to the last device image contained in the module.
   */
  device_image_iterator end() const;

  /**
   * Returns true if the current module uses specialization constants.
   *
   */
  bool has_spec_constant() const noexcept;

  // Available if module_state::input
  /* set_spec_constant.
  * If ID is a spec constant residing in the current module, returns a
  * spec_constant immutable object to represent the binding of the value
  * to the given module.
  */
  template <typename ID, typename T>
  spec_constant<T, ID> set_spec_constant(T cst);
};

}  // namespace sycl
```

Constructors to the `module` object are private and implementation defined.
Only *factory functions* (See below) can be used to create a `module` object.

`module` objects follow *Common reference semantics*.

### Free functions

The functionality that currently resides on methods of the program class
is moved to independent free functions in the `sycl` namespace.
Note this is just a user-facing API, implementations are free to implement
the functionality as implementation-defined methods of the `module` class.
Users can provide *SYCL properties* to customize the compilation, but no compilation flags.

The following free functions are defined in the general SYCL Next specification:

```cpp
namespace sycl {

namespace property {
  namespace module {
   namespace options {
    /** debug.
     * The module is build with debug symbols */
    class debug {
      public:
      debug() = default;
    };

    /** log.
     * Requests log information being redirected to the user-provided
     * ostream
    */
    class log {
      public:
      log(std::ostream& output);

      ostream& get_log_stream() const;
    };
   }
  }
}

/** build
 * Compiles and link the given input module and returns an executable module.
 * Options are defined in sycl::property::module::options, 
 * or by vendor extensions.
 * Throws:
 *  sycl::compile_program_error if the input cannot be linked
 *  sycl::link_program_error if the input cannot be linked
 */
template<typename... Options>
module<module_state::executable> build(module<module_state::input> input, Options&&... ops);

/** compile.
 * Compiles the given input object into an object module.
 * Options are defined in sycl::property::module::options, 
 * or by vendor extensions.
 * Throws:
 *  sycl::compile_program_error if there are compilation errors
 */
template<typename... Options>
module<module_state::object> compile(module<module_state::input> input, Options&&... ops);

/** link.
 * Links the given unlinked module objects into a single module.
 * All unlinked module objects must be associated with the same SYCL context.
 * Options are defined in sycl::property::module::options, 
 * or by vendor extensions.
 * Throws:
 *  sycl::link_program_error if the object modules cannot be linked 
 *  sycl::invalid_object if objects.empty() == true
 */
template<typename... Options>
module<module_state::executable> link(std::vector<module<module_state::object>> objects, Options&&... ops);


/** join.
 * Copies all device images from the given modules into a new module.
 * All device images must represent the same SYCL device code and contain
 * the same kernels and device symbols.
 * Throws:
 *  sycl::invalid_object : If modules are incompatible with each other or if modules.empty()
*/
template<module_state state>
module<state> join(std::vector<module<state>> modules);

}  // namespace sycl
```

### OpenCL `backend-specific` functionality

The OpenCL C specific API that exists currently on the SYCL 1.2.1 program class is
replaced by the following free functions on the OpenCL backend.
Note that the OpenCL C backend supports traditional compilation flags 
in the form of `std::string` objects.

```cpp
namespace sycl {
namespace opencl {

/** binary_blob_t
 * Binary blob, which starts at pointer pointed by `first` and has size
 * indicated by `second`.
 */
using binary_blob_t = std::pair<const char*, size_t>;

/** create_with_source
 * Creates a *source module* from the given OpenCL C string source
 */
module<module_state::input> create_with_source (context, std::string source);

/** create_program_with_il
 * Creates a *source module* from the given IL.
 */
module<module_state::input> create_program_with_il (context, binary_blob_t il);

/** create_program_with_binary
 * Creates a *source module* from the given device binaries.
 */
module<module_state::input> create_program_with_binary(context, binary_blob_t deviceBinary);

/** create_program_with_builtin_kernels
 * Creates a *source module* from the list of kernel names.
 * The kernel names must match OpenCL built-in kernels available in the OpenCL 
 * context represented by the given SYCL context.
 */
module<module_state::input> create_program_with_builtin_kernels (context, std::vector<std::string> kernelNames)

}  // namespace opencl 
}  // namespace sycl
```


#### SYCL object interoperability


In the OpenCL backend, all types of *SYCL module* objects map to
different states of a `cl_program` object.
A SYCL `kernel` object maps to a `cl_kernel` object in OpenCL.

```cpp
namespace sycl {

struct interop<backend::opencl, module<module_state::input>> {
  native_type = opencl::cl_program;
};

struct interop<backend::opencl, module<module_state::object>> {
  native_type = opencl::cl_program;
};

struct interop<backend::opencl, module<module_state::executable>> {
  native_type = opencl::cl_program;
}; 

struct interop<backend::opencl, kernel> {
  native_type = opencl::kernel;
};
 
}  // namespace sycl
```

The `cl_program` returned from interoperability with a `module<module_state::input>`
is suitable to be used on the calls to `clCompileProgram` and `clBuildProgram`.
The `cl_program` returned from interoperability with an `module<module_state::object>`
is suitable to be used on the call `clLinkProgram`.
The `cl_program` returned from interoperability with a `module` is not suitable
for any OpenCL runtime compilation APIs (i.e. clBuildProgram, clCompile or clLink).

The following additional free functions are defined for interoperability:

```cpp
namespace sycl {
namespace opencl {


/** import.
 * Returns the unlinked module object that encapsulates the given clProgram object.
 * The reference count of the clProgram is increased.
 * The state must match that of the clProgram object:
 *   module_state::input -> CL_PROGRAM_SOURCE or CL_PROGRAM_IL must not be null
 *   module_state::object -> CL_PROGRAM_BINARIES must not be null, 
					cl_program is the outcome of a call to `clCompileProgram`
 *   module_state::executable -> CL_PROGRAM_BINARIES must not be null,
 *        cl_program is the outcome of a call to `clLinkProgram` or 
 *        `clBuildProgram`and CL_PROGRAM_NUM_KERNELS must be at least 1.
 */
template<module_state state>
module<state> import(sycl::context, cl_program clProgram);

}
}
```

## Context class

The *SYCL context* class gains a method to retrieve the *SYCL modules* associated with it.

```cpp
namespace sycl {
class context {
  public:
    /* .... existing members .... */

    /**
     * Returns a vector containing the module objects associated with this context
     * instance with the requested state.
     * The modules returned may be a super-set of any user-created modules,
     * as implementations are free to create *SYCL module* instances of their own.
     */
    template<module::module_state state>
    std::vector<module<state>> get_modules() const noexcept;
};
}  // sycl
```

## Dispatching kernels associated with a module

The API to dispatch a parallel-for from a pre-compiled binary remains the same, 
a sycl::kernel object can be passed to the `parallel_for` or `single_task` to use as a pre-compiled image
of the user-given functor.

A new method, `use_module(sycl::module<module_state::executable> moduleWithKernels)`, is added to the handler.
The method associates a SYCL *module* object to the command group.
All kernels used on the command group will be obtained only from the given SYCL *module*.
If the SYCL *module* does not contain any of the requested kernel, `sycl::kernel_not_found` is thrown synchronously.
If the user passes a *SYCL kernel* object that does not belong to the *SYCL module* passed to the `use_module` method,
`sycl:invalid_kernel` is thrown synchronously.

Additionally, a new method `use_module(sycl::module<module_state::executable> moduleWithKernels, DeviceImageSelectorT selector)` is also added
to the handler. In addition to use the given module to obtain all kernels used in the command group, 
the `selector` function will be used to help the runtime select a particular *device image* from the module. 
The signature of the *device image selector* is as follows:

```cpp
using exe_mod = sycl::module<module_state::executable>;

exe_mod::device_image selector(exe_mod::device_image_iterator start, exe_mod::device_image_iterator end);
```

This allows implementations to offer a fine-grain selection of *device images* inside modules for advanced users.
Since *device_images* are implementation-specific, the `DeviceImageSelector` implementation must rely on implementation-specific
behavior.


```cpp
class handler {
  public:

  ....
  
  /** use_module.
   * Uses the given module as input to obtain the kernel objects
   * for all kernels in the command group.
   */
  void use_module(sycl::module<module_state::executable> moduleWithKernels);

  /* use_module.
   * Uses the given module as origin to obtain the kernel objects
   * for all kernels in the command group.
   * DeviceImageSelectorT
   */
  template <typename DeviceImageSelectorT>
  void use_module(sycl::module<module_state::executable> moduleWithKernels, DeviceImageSelectorT selector);

}
```

## Module for the current translation unit

The namespace `this_module` is defined on the SYCL headers and contains the following interface:

```cpp
namespace sycl {
namespace this_module {
  template<module_state state>
  module<state> get(context ctxt);

  bool has_any_module(context ctxt);

  template<module_state state>
  bool has_module_in(context ctxt);
}  // this_module
}  // sycl
```

The function `has_any_module(context ctxt)` returns *true* if there are modules available
in any state for the given SYCL context.

The function `has_module_in(context ctxt)` returns *true* if there is a module available
for the given SYCL context and current translation unit in the *state* requested, *false* otherwise.

The module object returned by `get(context ctxt)` contains all available images in 
the state requested by the user.

If `has_module_in` returns *true* for a particular value of `module_state` 
and context, the `get` must return one module for the same values.
If `has_any_module` returns true, `get` must return one module 
for at least one value of `module_state`.

**Implementation note**: Implementations can trigger compilation and linking
operations when user request for an *executable* or *object* module if
the original images are *input*, but it is not expected for the user to
always have *executable* modules ready and it is valid to return an empty
module.

## Specialization constants

Proposal [CP015](https://github.com/codeplaysoftware/standards-proposals/blob/master/spec-constant/index.md)
introduced usage of SPIR-V specialization constants in SYCL.
Specialization constants are associated with the SYCL 1.2.1 program class, 
and named using a C++ type like kernels.
The program class gains a `set_spec_constant` method, that sets a runtime
value to the specialization constant.
The program can then be build with `build_with_kernel_type` to create a 
specialized kernel.

On the new, type-safe, module approach, specialization constants are 
associated with the OpenCL backend. 
Other backends (e.g. Vulkan) may offer the same functionality, so the
specialization constant type is defined as an (experimental) general 
SYCL type as below:

```cpp
namespace sycl {
namespace experimental {

template <typename T, typename ID = T>
class spec_constant {
private:
  // Implementation defined constructor.
  spec_constant(/* Implementation defined */);
public:
  spec_constant();

  T get() const; // explicit access.
  operator T() const;  // implicit conversion.
};

}  // namespace experimental
}  // namespace sycl
```

`spec_constant` is a *placeholder type* used by the *SYCL implementation* to
inject the real value of the specialization constant at runtime.
It also allows implementations that don't support specialization constant
to hide the value as a parameter, keeping the lambda-capture the same
irrespectively of the support of the feature.

The interface to set values to the specialization constant
in the module is backend specific:

```cpp
namespace sycl {
namespace opencl {
  template<typename ID, typename T>
  spec_constant<ID, T> set_spec_constant(module<module_state::executable> syclModule, T cst) const;
}  // namespace opencl
}  // namespace sycl
```

For simplicity, the spec constant placeholder type can also be obtained from
the command group handler:

```cpp
namespace sycl {
 class handler {
// ...
public:
 template <typename ID, typename T>
 spec_constant<T, ID> get_spec_constant();
// ...
};

}  // namespace sycl
```

Note the value of the specialization constant depends on the module that is used,
not on the placeholder object.

*Implementation note*: Although specialization constants are meant for situations
where there is a final online compilation stage, if the implementation supports
compiling directly on the backend for an ISA, specialization constants can still
be supported via two mechanism: (1) Having different specialized images for known
values on the specialization constant or (2) having the value of the specialization
constant as a normal kernel parameter. If choosing (1), the original SPIR-V
image should be part of the SYCL module so that the module can be specialized
for non-known values.
