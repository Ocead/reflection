# ![](./res/ocead.svg) Ocead's Reflection Library
A single-include library for reflection in C++ programs

## Example
<table>
<tr>
<th>With reflection</th>
<th>Without reflection</th>
</tr>
<tr>
<td>

```c++
#include <string>
#include <ocead/reflection.hpp>

class example : public ocead::reflection {
    REFLECTIVE(example)(
      PROP(int, 1000, number);
      PROP(public:, std::string, 1001, text) = "Gnampf";
      double non_reflective_0;
      
      FUNC(public:, void, 2000, add, (int summand)) {
          number += summand;
      }
      
      FUNC(std::string const &, 2001, get_text, () const) {
          return text;
      }
      
      void non_reflective_1();
      
      CTOR(example, 3000, ()) = default;
      
      CTOR(example, 3001, (int n, std::string && t))
          : number(n),
            text(t) { }
      
      DTOR(example, 4000) = default;
    );
    
    private:
    double non_reflective_2 = 6.;
};
```

</td>
<td>

```c++
#include <string>


class example {
    
    private: int number;
    public: std::string text = "Gnampf";
    double non_reflective_0;
    
    public: void add(int summand) {
      number += summand;
    }
    
    public: std::string const & get_text() const {
      return text;
    }
    
    void non_reflective_1();
    
    example() = default;
    
    example(int n, std::string && t)
      : number(n),
        text(t) { }
    
    ~example() = default;
    
    
    private:
    double non_reflective_2 = 6.;
};
```

</td>
</tr>
</table>

## Requirements
- GNU GCC or Clang supporting C++17
- RTTI is **not** required

## Usage
(The following examples are based on the `example` class from the example above)

### Writing reflective classes
With the macros described below, class-wide unique IDs are added to the classes symbols.
With these IDs, the reflective symbols can be accessed dynamically at runtime.
By default the type of these IDs is `std::size_t`,
but you can replace it with any type that produces compile time constant expressions.

> ℹ I recommend enum classes as a type for class-wide unique IDs.

#### Reflective blocks
Each class definition may contain a reflective block. This block is either delimited by the
```c++
BEGIN_REFLECTIVE(example);
...
END_REFLECTIVE();
```
macros, or contained within the
```c++
REFLECTIVE(example)(
    ...
)
```
macro, here with `example` as the reflective classes name. All reflective symbols must appear in these blocks.

> ℹ Reflective blocks may also contain non-reflective symbols. The other way round is non possible.

#### Properties
Reflective properties can be declared with the `PROP` macro, like so:
```c++
PROP(public:, std::string, 1001, text) = "Gnampf";
```
Here:
- `public:` is the access specifier for the reflective property (can be omitted)
- `std::string` the type of the reflective property
- `1001` the class-wide unique ID for the reflective symbol
- `text` the name of the reflective property
- `= "Gnammpf"` the optional default initializer for the reflective symbol

Without access specifier the same declaration would look like this:
```c++
PROP(std::string, 1001, text) = "Gnampf";
```
In this case, the reflective property is `private` by default.

#### Functions
Reflective functions can be declared with the `FUNC` macro, like so:
```c++
FUNC(public: void, 2000, add, (int summand)) {
    number += summand;
}
```
Here:
- `public:` is the access specifier for the reflective function (can be omitted)
- `void` the return type of the reflective function
- `2000` the class-wide unique ID for the reflective symbol
- `add` the name of the reflective function
- `(int summand)` the parameter list of the function and optional const qualifier
- `{ number += summand; }` the optional function body following the declaration

If the access specifier is omitted, it defaults to `public`.

#### Constructors
Reflective constructors can be declared almost like functions, but with the `CTOR` macro instead:
```c++
CTOR(public: explicit, example, 3001, (int n, std::string && t))
    : number(n),
      text(t) { }
```
Here:
- `public: explicit` is the access specifier for the reflective constructor
  and optional explicit specifier (can be omitted)
- `example` the class name
- `3001` the class-wide unique ID for the reflective symbol
- `(int n, std::string && t)` the parameter list of the constructor
- `: number(n), text(t) { }` the optional initializer list and constructor body

If the access specifier is omitted, it defaults to `public`.

#### Destructors
Reflection destructor can be declared almost like constructors, but with the `DTOR` macro instead:
```c++
DTOR(public:, example, 4000) = default;
```
Here:
- `public:` is the access specifier for the reflective destructor (can be omitted)
- `example` the class name
- `4000` the class-wide unique ID for the reflective symbol
- `= default` the implementation of the destructor

Again, if the access specifier is omitted, it defaults to `public`.

### Obtaining class descriptors
Class descriptors provide you with reflective information at runtime.
To obtain a class descriptor of a reflective class, use the `ocead::reflect` template function,
like so:
```c++
const auto & refl = ocead::reflect<example>();
```

If you have RTTI enabled for your program, you can additionally obtain a classes class descriptor
by calling the overridden `ocead::reflective::reflect` function, like so:
```c++
example instance;
const auto & refl = instancel.reflect();
```

These descriptors map the class-wide unique IDs to symbol descriptors that contain information such as name, type
and, depending on the kind of symbol, various function pointers.

### Accessing member fields

#### Typed / Untyped
The difference in calling typed and untyped accessors is where the class-wide unique ID goes:
- For typed accessors the ID becomes a template parameter
  ```c++
  auto & number = refl.access<1000>(instance);
  ```
- For untyped accessors the ID is a regular parameter
  ```c++
  auto & number = *static_cast<int &>(refl.access(&instance, 1000));
  ```

For each `access` function a `const_access` function for accessing const objects exists.
#### Access by function / offset
Normally, reflective properties are accesses via generated functions that return the property.
For faster access you may use access by offset.
Only reflective properties that aren't references can also be accessed by offset from a pointer to an object.
> ⚠ Accessing members of non standard-layout types is only conditionally supported.
> Check if your compiler supports offset accessing before using this feature!

The functions for access by offset are `offset_access` and `const_offset_access`.
They can be called just like their non-`offset_` counterparts.

### Invoking member functions
Reflective functions can be invoked with the `invoke` functions in class and symbol descriptors.
The call depends on what you know at compile time:

- If you know the ID at compile time:
  ```c++
  auto sum = refl.invoke<int, 2000>(&instance, 5);
  ```
- If you don't know the ID at compile time:
  ```c++
  auto sum = refl.invoke_return<int>(&instance, 2000, 5);
  ```
  For the ID-known-at-compile-time variant you only need the `invoke function`.
  For the ID-known-at-run-time variant you have to discern whether to use the invoked functions
  return value, in which case you have to call `invoke_return` instead, or not. Then it is:
  ```c++
  refl.invoke(&instance, 2000, 5);
    ```

In any case, you pass a pointer to the object on which the reflective function shall be invoked on,
then the optional id, followed by the arguments with which to call the function.
Pass the arguments just as you would when calling `std::invoke`.

> Feedback and suggestions in regard to resolving the ambiguity between invocations a very welcome.

### Constructing instances
Constructing reflective objects is very straightforward.
You can construct an instance of a reflective class in one of three ways:

- new
  ```c++
  void * instance = refl.construct(3001, 6, std::string(""));
  ```
  This constructs the instance using `new`.
- placement new
   ```c++
  alignas(example) std::byte buffer[sizeof(example)];
  void * instance = refl.placement_construct(buffer, 3001, 6, std::string(""));
  ```
  This constructs the instance using placement `new`.
- shared new
  ```c++
  auto instance_ptr = refl.shared_construct(buffer, 3001, 6, std::string(""));
  ``` 
  This constructs the instance using `new` and returns a smart pointer to the instance.

As with invocations, if you know the class-wide unique ID at compile time, you may pass is as a
template parameter.
### Destroying instances
Destroying reflective objects is even simpler:

- Calling destructors
  ```c++
  refl.destroy(4000, &instance);
  ```
  This calls the destructor in instance.
- Deallocating instances
  ```c++
  refl.dealloc_destroy(4000, &instance);
  ```
  This calls the destructor in instance and deallocates the occupied memory.
  Use it just as you would use `delete`.
  
## Records
You can generate arbitrary record classes from reflective classes,
that contain only the member fields you need, like so:
```c++
auto rec = ocead::make_record<1000, 1001>(instance);
```
The template parameters are the IDs of the properties you want to include into the record.
You can access the records properties with `std::get`:

```c++
int number = std::get<1000>(rec);
```

## Customization
You may customise this library by overriding different macros.

These macros control the types employed by the library:

|Macro|Default type|Description|
|-----|------------|-----------|
|`OCEAD_REFLECTION_COUNTER_TYPE`|`std::size_t`|Used for globally distinguishing reflective symbols|
|`OCEAD_REFLECTION_ID_TYPE`|`std::size_t`|Used as unique identifier by `OCEAD_REFLECTION_CLASS_DESCRIPTOR_TYPE`|
|`OCEAD_REFLECTION_TYPEID_TYPE`|`char const *`|Used for differentiating (return) types of reflective symbols|
|`OCEAD_REFLECTION_NAME_TYPE`|`char const *`|Used for recording names of reflective symbols|
|`OCEAD_REFLECTION_CONTAINER_TEMPLATE`|`ocead::reflection::type_identity`|May be used to transform the apparent type of a reflective property into a more complex type|
|`OCEAD_REFLECTION_ACCESSOR_TYPE`|`ocead::accessor_t`|Function type for non-const accessors|
|`OCEAD_REFLECTION_CONST_ACCESSOR_TYPE`|`ocead::const_accessor_t`|Function type for const accessors|
|`OCEAD_REFLECTION_INVOKER_TYPE`|`ocead::invoker_t`|Function type for non-const invokers|
|`OCEAD_REFLECTION_CONST_INVOKER_TYPE`|`ocead::const_invoker_t`|Function type for const invokers|
|`OCEAD_REFLECTION_SHARED_PTR_TYPE`|`std::shared_ptr<void>`|Smart pointer type that can hold deleter functions|
|`OCEAD_REFLECTION_SYMBOL_DESCRIPTOR_TYPE`|`ocead::reflection::symbol_descriptor`|Type used for describing single reflective symbols|
|`OCEAD_REFLECTION_REFLECTOR_MAP_TYPE`|`std::map`|Type used for organizing `OCEAD_REFLECTION_SYMBOL_DESCRIPTOR_TYPE`|
|`OCEAD_REFLECTION_CLASS_DESCRIPTOR_TYPE`|`ocead::reflection::class_descriptor`|Type used for describing reflective classes|

These macros control how other macros in this library are expanded:

|Macro|Default expansion|Description|
|-----|-----------------|-----------|
|`OCEAD_REFLECTION_IDENTIFIER_MACRO(type)`|`ocead::reflection::typename_hash(#type)`|Produces `OCEAD_REFLECTION_TYPEID_TYPE` from the name of the type|
|`OCEAD_REFLECTION_ADD_PREFIX(symbol)`|`_reflection_##symbol`|Macro for modifying the names of additionally generated symbols|
|`OCEAD_REFLECTION_DEFAULT_PROPERTY_VISIBILITY`|`private:`|The default visibility of reflective properties|
|`OCEAD_REFLECTION_DEFAULT_FUNCTION_VISIBILITY`|`public:`|The default visibility of reflective functions|
|`OCEAD_REFLECTION_DEFAULT_CONSTRUCTOR_VISIBILITY`|`public:`|The default visibility of reflective constructors|
|`OCEAD_REFLECTION_DEFAULT_DESTRUCTOR_VISIBILITY`|`public:`|The default visibility of reflective destructors|

You can disable reflection by defining the
```
OCEAD_NO_REFLECTION
```
macro.
If defined, the macros for reflective symbols expand only to the declaration of the symbols.

## Roadmap
- [x] Non-static member fields
- [x] Non-static member functions
- [x] Const-qualified members
- [x] Constructors
- [x] Destructors
- [x] Reference fields
- [x] Functions returning references
- [x] Records
- [x] Functions without return value
- [x] Accessing member fields via offset
- [ ] Static member fields
- [ ] Static member functions

## License
The code in this repository is distributed under the Boost Software License Version 1.0.

See the [code file](include/ocead/reflection.hpp) or the [LICENSE](LICENSE) file for the full license text.