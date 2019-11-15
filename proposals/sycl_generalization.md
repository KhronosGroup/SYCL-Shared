# SYCL Generalization

|                  |                                     |
| ---------------- | ----------------------------------- |
| Name             | SYCL Generalization                 |
| Date of creation | 3rd June 2019                       |
| Last updated     | 15th June 2019                      |
| Current revision | 2                                   |
| Dependencies     | None                                |
| Available        | N/A                                 |
| Reply-to         | Ruyman Reyes <ruyman@codeplay.com>  |
| Original author  | Ruyman Reyes <ruyman@codeplay.com>  |
| Contributors     | Romain Biessy <romain.biessy@codeplay.com>, Gordon Brown <gordon@codeplay.com>, Ronan Keryell] <ronan.keryell@xilinx.com>, Morris Hafner <morris@codeplay.com>, Toomas Remmelg <toomas.remmelg@codeplay.com> |

**The contents of this proposal are to encourage early feedback and are not ratified by the Khronos Group so do not constitute a formal specification.**

## Overview

The current SYCL 1.2.1 specification defines SYCL as a "high level model over OpenCL".
The aim for SYCL Next is to make SYCL a generic high level model that can serve as a
general interface to program multiple heterogeneous programming models.

The objectives of this proposal are:

* To steer the direction of the SYCL Next specification towards a general C++ single-source programming model that implementors can adapt to different vendor API
* To detach SYCL from the OpenCL specification, but keeping SYCL aligned with OpenCL. OpenCL should be the "reference vendor API" for all future design in SYCL
* To adapt the conformance process, and the conformance test suite, so that vendors can pass SYCL conformance without relying on OpenCL - but guaranteeing a certain level of usability and features
* To provide a mechanism of extending the general SYCL Next specification in two axes: Vendor-specific (with specific vendor-backend operations) and SYCL general (with Khronos-endorsed extensions that will work across all backends when supported via the underlying hardware)

The objectives of this proposal are NOT:

* To replace other Khronos APIs, such as OpenCL or Vulkan. SYCL is merely a high-level layer that requires other, closer-to-metal, APIs to be implemented
* To add any new features. This proposal merely addresses what is needed to convert SYCL 1.2.1 into a generic specification, but the feature-set will be largely the same as exposed by OpenCL 1.2. Other proposals, based on the new document layout and spec design, will add new features.
* Fundamentally change the structure or the sections of the specification. Although there are good reasons for doing that, this proposal aims to be as close to the SYCL 1.2.1 as possible to minimize the differences and facilitate comparison and reviewing of the *detachment* from OpenCL.

## Revisions

v0.2:
* Minor changes for publication
  
v0.1:
* Initial version

## The backend model

This proposal to detach SYCL from OpenCL is based on a *layered API model*, where the SYCL programming model offers a single-source programming API with a consistent data-model that enables developers to share data across different APIs. 

![Backends](sycl_backends.svg)

A backend is considered a valid backend for SYCL if it can implement all generic SYCL behavior. This can be verified by passing conformance on the SYCL Next CTS.

The SYCL CTS will only actively check conformance of non-backend specific behavior. In particular, interoperability with backends is not tested on the SYCL Next CTS.

The existing SYCL host device is replaced by a host backend, with similar functionality but well defined behavior. 
OpenCL becomes one of the potential backends which a SYCL application can be targeted.

The SYCL group can create **backend specification documents**, defining how a SYCL backend for a given programming model will work across different implementations.
This ensures that different SYCL implementations follow the same interface and offer the same expected behavior for specific backends.
The generic SYCL behavior should be the same across all backends, except for interoperability which is custom per backend by necessity.

In particular, the SYCL group will maintain the *OpenCL backend for SYCL* document, where the implementation of the SYCL program in terms of OpenCL is defined and the interoperability interface is specified. 
The SYCL group will maintain a conformance test suite for the OpenCL backend, based on the existing OpenCL-interoperability tests. 

There are two main aims for the *OpenCL backend for SYCL* document: (1) To ensure SYCL remains aligned and compatible with OpenCL and (2) to serve as a template for specifying other backends for SYCL.
The work to maintain different backend specification documents could also be addressed by sub-groups of the main SYCL WG (e.g. Vulkan backend)

Interoperability *across backends* is guaranteed by the specification only at the level of memory objects.
SYCL memory objects, such as `buffer` and `image` objects are able to store the data in multiple SYCL contexts, which can come from different platforms.
A SYCL runtime is expected to transfer data transparently across SYCL contexts using the most suitable mechanism for the system where the SYCL application is being run. 
Command groups enqueued in queues may have data dependencies on data stored in context(s) from different backends. 
It is the responsability of the SYCL runtime to guarantee that the data will be available on the device where the command group is executed irrespectively of its origin.
Note that a valid implementation of transferring data across two backends is simply to go through the host main memory.

### Terms

* SYCL backend: An implementation of the SYCL programming model using an heterogeneous programming API.  A SYCL backend exposes one or multiple SYCL platforms. For example, the OpenCL backend, via the ICD loader, can expose multiple OpenCL platforms.
* Native backend object/type: An opaque object defined by a specific backend that represents a high-level SYCL object on said backend. There is no guarantee of having native backend objects for all SYCL types.
* Interoperability API: SYCL API entries that enable direct interoperability with the underlying SYCL backend. 
* Active backend(s): The backend(s) used to build the current SYCL application 
* Available backend(s): The intersection of those backends *active at compilation time* and *available in the system at runtime*.

### The SYCL Platform model

SYCL specification will be written in terms of a generic *Backend* instead of *OpenCL API*. 
The SYCL architecture will continue to be based on the OpenCL *platform* model, where a host is connected to one or more SYCL devices.

SYCL devices are accessed from a SYCL runtime via a *platform*.
SYCL *platforms* represent entry points for SYCL backends to expose their devices. Each SYCL backend will expose one or more platform entries, and each of those will contain one or more SYCL devices.

A SYCL application can only see platforms exposed by SYCL backends active at compilation time.
Not all backends active at compilation time may be available at runtime.

In the example below, the user builds a SYCL application with two targets, 
one is spir64 for OpenCL and another is for a custom vendor device that
is exposed via its own backend (call it *Custom*).

The user invokes the compiler as follows:

```sh
$ clang++ -fsycl -fsycl-targets=spir64,custom-device app.cpp -o app.exe
```

There are two active backends at compile time *OpenCL* and *Custom*.
The user executes the application, exporting only the *libCustom.so* 
library to the application on a system where there is no OpenCL implementations
installed:

```sh
$ LD_LIBRARY_PATH=/opt/lib/libCustom.so ./app.exe
```

In this case, although both *OpenCL* and *Custom* were *active at compilation time*, only *Custom* is *available at runtime*.


### The SYCL Execution model

The SYCL runtime in the SYCL application manages all resources from all the active backends. 
The default device provided by an implementation may come from any active backend.

### The SYCL programming model

The `cl::sycl::` namespace is reserved for SYCL 1.2.1.
SYCL Next API will be exposed on the `sycl::` namespace.
The macro `SYCL_VERSION` will be set to `YYYY`.
The header `SYCL/sycl.hpp` will provide all the generic SYCL Next API.
SYCL users should not include any specific backend headers if they use only generic SYCL API.
Backend specific API (e.g. interoperability API) is available in `SYCL/backend/backend_name.hpp`, e.g. OpenCL backend headers will be in `SYCL/backend/opencl.hpp`.
Vendors can provide their own backends.
The interoperability API is defined in `SYCL/vendor/backend/backend_name/backend_header.hpp`
SYCL vendor backends are namespaced as `sycl::vendor::backend::vendor_name::backend_name`.

Both SYCL Next general and vendor-specific extensions can still be defined independently of backend specific functionality.
The path `SYCL/vendor/vendor_name/extension.hpp`, and the namespace `sycl::vendor::vendor_name` are reserved for vendor-specific extensions.
Generic SYCL extensions are exposed via individual headers in `SYCL/ext/ext_name.hpp`.

When a SYCL implementation is compiling a SYCL source, it must define a pre-processor macro for each active backend.
The host backend is always active, but for symmetry the pre-processor macro `SYCL_BACKEND_HOST` is always defined.
When the OpenCL backend is active, the macro `SYCL_BACKEND_OPENCL` is defined.
When a vendor-backend is active, it must define a macro in the form of `SYCL_BACKEND_XXX_YYY`,
where *XXX* and *YYY* are the vendor name and the backend name respectively.

### Querying backends at compile time

SYCL implementations must expose the SYCL backends available at the time of compilation in the enum class `sycl::backend`. 
At least the `host` backend must be listed.

The enum is defined as follows:

```cpp
namespace sycl {
enum class backend {
    host,
    opencl, // When implementation provides OpenCL backend
    vendor_backend_name // When a vendor specific backend is added 
}

}  // namespace sycl
```

The numerical values of each member of the enum class are implementation defined.

Those that have been enabled for the application being built, the *active backends*, must be registered with the `sycl::is_active` trait,
defined as follows:

```cpp
namespace sycl {

// Generic SYCL-defined interface
template <backend valueT>
struct is_active : std::false_type {};

}  // namespace sycl
```

An example of registering the OpenCL backend as an *active backend* is shown below:

```cpp
namespace sycl {

#ifdef SYCL_BACKEND_OPENCL
template<>
struct is_active<backend::opencl> : std::true_type {};
#endif  // SYCL_BACKEND_OPENCL

}  // namespace sycl
```


When a SYCL implementation has OpenCL as an available backend, the macro
`SYCL_BACKEND_OPENCL` is defined and the `backend::opencl` will be defined such
that `is_active<backend::opencl>::value` is well-formed.  
When the program is built with OpenCL as an active backend, the value of the
`is_active` the trait will be true such that
`is_active<backend::opencl>::value` == true.

Checking the value of `is_active` is always well-formed for those backends
provided by the SYCL implementation.

### Querying backends at runtime

Once a SYCL application is built with a set of *active backends*, those will be available for selection at runtime. 
It is implementation dependent whether certain backends require third-party libraries to be available in the system. 
Building with only the *host* as an *active backend* guarantees the binary will be executed on any platform without third-party libraries. 
Failure to have all dependencies required for **all** *active backends* at runtime will cause the SYCL application to not run.

Once the application is running, users can query what SYCL platforms are available. SYCL implementations will expose the devices provided by each backend grouped into platforms. A backend can expose as many platforms as it may see fit. 
In particular, the *OpenCL* backend will expose all *OpenCL* platforms as SYCL platforms.

#### SYCL platform API

The `platform` API  changes to reflect the new backend functionality are as follows:

```cpp
namespace sycl {
class platform {
public:

// Constructs a SYCL platform from the given backend, defaults to host
explicit platform(backend user_selected_backend = backend::host);

// Constructs a SYCL platform from a device selected by the device
//  selector. The platform will contain at least the device that
//  is returned by the device selector.
explicit platform(const device_selector &deviceSelector);

/* -- common interface members -- */
// Return all SYCL devices available on the platform
vector_class<device> get_devices(
info::device_type = info::device_type::all) const;

template <info::platform param>
typename info::param_traits<info::platform, param>::return_type get_info() const;

bool has_extension(const string_class &extension) const;

// Returns all available SYCL platforms, irrespectively of the backend
static vector_class<platform> get_platforms();

// Returns all available SYCL platforms for the given backend.
static vector_class<platform> get_platforms_from_backend(backend user_selected);

// Returns the backend to which this platform belongs
backend get_backend() const noexcept;

};
} // namespace sycl
} // namespace cl

```

### General interoperability 

SYCL applications can be written against the SYCL application Model without requiring knowledge of the backend they are targeting. 
However, sometimes SYCL developers want to interoperate with the underlying backends to access additional functionality or interact with existing libraries.

The general SYCL interoperability API exposes backend-specific functionality that allows developers to access data types and API functions specific for a given backend.

The general interoperability API is composed of three types of functions:
* Factory functions to construct SYCL objects using backend-specific mechanism
* Get functions to obtain the native type of a given SYCL object.
* A template type trait that provides the type equivalence between SYCL objects and native-backend objects.

The three mechanism must be implemented following this API specification for a backend to be a valid SYCL backend.
Not all equivalences are necessary.
Backend implementors can provide additional, backend-specific, functionality for interoperability.

#### Template trait to map SYCL types and Native types

SYCL backends that expose an interoperability interface must provide a function that enables user to query which SYCL object has an equivalent Native type.
Backends must define, for each SYCL type they expect to have a Native type equivalent, an specialization of the `struct interop` as shown below:

```cpp
namespace sycl {
namespace backend_name {
    // SYCL OpenCL backend headers, generic interop interface definitions
    template<backend name, typename SYCLObjectT>
    struct interop;
}  // backend_name
}  // sycl
```

An example of interoperability mapping for OpenCL can be seen below, where the SYCL context object is mapped to an underlying `cl_context` object type.

```cpp
namespace sycl {

  // Mapping defined by SYCL the backend document
  template<>
    struct interop<backend::opencl, sycl::context> {
      using type = opencl::cl_context;
    };

}  // sycl
```


#### Creation of SYCL objects from backend-specific features

When a SYCL backend wants to expose functionality to create SYCL objects from a Native Object, the backend must expose a *factory function*.
These *factory function*s are only available when the backend-specific header is included by the user.

The factory function looks like the following:

```cpp
namespace sycl {
namespace backend_name {

// Construction of device, platform or context objects
template<typename SyclObjectT>
SyclObjectT make(interop<backend backend_name, SyclObjecT>::type interop);

// Construction of all other objects
template<typename SyclObjectT>
SyclObjectT make(sycl::context context, typename interop<backend backend_name, SyclObjecT>::type interop);

// Construction of all other objects
template<typename SyclObjectT, typename... Args>
SyclObjectT make(sycl::context context, typename interop<backend backend_name, SyclObjecT>::type interop, Args &&... additional_args);

}  // backend_name
}  // sycl
```

Note that SYCL objects that are associated with a context require a SYCL context to be passed on construction. This guarantees that ```get_context()``` returns the same underlying SYCL object.
Backends can add as many additional arguments as they require to build a valid SYCL object.
Arguments can also be in the form of SYCL property objects.

### Obtaining a NativeType from a SYCL Object

SYCL backends that expose interoperability must declare ```get_native``` functions that convert a SYCL oject to the NativeType counterpart.

The general SYCL API defines the free-function `get_native` as follows:
```cpp
namespace sycl {

template<backend backendName, typename SYCLObjectT>
auto get_native(SYCLObjectT obj) -> typename interop<backendName, SYCLObjectT>::type;


}  // namespace sycl
```

For convenience, each backend defines a function that predefines the backend name.
These `get_native` functions are only available when the backend-specific header is included by the user.

The interface is defined below: 

```cpp
namespace sycl {
namespace backend_name {

template<typename SYCLObjectT>
auto get_native(SYCLObjectT obj) -> typename interop<backend_name, SYCLObjectT>::type;

}  // backend_name
}  // sycl
```

Note in this case the ```get_native``` function cannot take any other parameters.

In particular, for OpenCL:

```cpp
namespace sycl {
namespace opencl {

template<typename SYCLObjectT>
auto get_native(SYCLObjectT obj) -> typename interop<backend::opencl, SYCLObjectT>::type;

}  // opencl
}  // sycl
```

All SYCL objects with interoperability will expose a member `get_native`,
defined as follows:

```cpp
class sycl_object {
 public:
 ...
 template<backend backendName>
 auto get_native() -> typename interop<backendName, sycl_object>::type;
};
```
Each backend can specify an implementation of the method for their backend,
even if is only as a forwarding for the generic one.

The `get()` method from SYCL 1.2.1 is removed from all classes.
The retain and release behaviour for OpenCL interoperability is moved
to the OpenCL backend-specific document.

## Other changes

* All SYCL constructors taking OpenCL objects are removed from the SYCL
  specification. The functionality they expose is presented via factory 
  functions in the OpenCL-specific backend.


## Related proposals

* Program class refactor (In progress)
* Image class redesign (In progress)

## Open questions

* What do we do with info parameters? Are they backend specific? Do we define some to be generic, others backend-specific?
