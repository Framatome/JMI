# JMI
**_JNI Modern Interface in C++_**

### Features

- Support both In & Out parameters for JNI methods
- Per class jclass cache, per method jmethodID cache, per field jfieldID cache
- The same C++/Java linkage: a static java member maps to a static member in C++
- getEnv() at any thread without caring about when to detach
- Signature is generated by compiler only once
- Supports JNI arithmetic types, JObject, string and array of these types as method parameter type, return type and field type
- Easy to use

### Example:
- Setup java vm in `JNI_OnLoad`: `jmi::javaVM(vm);`

- Create a SurfaceTexture: 
```
    // define SurfaceTexture tag class in any scope visibile by jmi::JObject<SurfaceTexture>
    struct SurfaceTexture : jmi::ClassTag { static std::string name() {return "android/graphics/SurfaceTexture";}};
    ...
    GLuint tex = ...
    ...
    jmi::JObject<SurfaceTexture> texture;
    if (!texture.create(tex)) {
        // texture.error() ...
    }
```

- Create Surface from SurfaceTexture:
```
    struct Surface : jmi::ClassTag { static std::string name() {return "android.view.Surface";}}; // '.' or '/'
    ...
    jmi::JObject<Surface> surface;
    surface.create(texture);
```

- Call void method:
```
    texture.call("updateTexImage");
```

or

```
    texture.call<void>("updateTexImage");
```

- Call method with output parameters:
```
    float mat4[16]; // or std::array<float, 16>, valarray<float>
    texture.call("getTransformMatrix", std::ref(mat4)); // use std::ref() if parameter should be modified by jni method
```

If out parameter is of type `JObject<...>` or it's subclass, `std::ref()` is not required because the object does not change, only some fields may be changed.

```
    MediaCodec::BufferInfo bi;
    bi.create();
    codec.dequeueOutputBuffer(bi, timeout);
```

- Call method with a return type:
```
    jlong t = texture.call<jlong>("getTimestamp");
```

## jmethodID Cache

 `GetMethodID/GetStaticMethodID()` is always called in `call/callStatic("methodName", ....)` every time, while it's called only once in overload one `call/callStatic<...MTag>(...)`, where `MTag` is a subclass of `jmi:MethodTag` implementing `static const char* name() { return "methodName";}`.

```
    // GetMethodID() will be invoked only once for each method in the scope of MethodTag subclass
    struct UpdateTexImage : jmi::MethodTag { static const char* name() {return "updateTexImage";}};
    struct GetTimestamp : jmi::MethodTag { static const char* name() {return "getTimestamp";}};
    struct GetTransformMatrix : jmi::MethodTag { static const char* name() {return "getTransformMatrix";}};
    ...
    texture.call<UpdateTexImage>(); // or texture.call<void,UpdateTexImage>();
    jlong t = texture.call<jlong, GetTimestamp>();
    texture.call<GetTransformMatrix>(std::ref(mat4)); // use std::ref() if parameter should be modified by jni method
```

### Field API

Field api supports cacheable and uncacheable jfieldID. Field object can be JNI basic types, string, JObject and array of these types.

Cacheable jfieldID through FieldTag

```
    JObject<MyClassTag> obj;
    ...
    struct MyIntField : FieldTag { static const char* name() {return "myIntFieldName";} };
    auto ifield = obj.field<int, MyIntField>();
    jfieldID ifid = ifield; // or ifield.id()
    ifield.set(1234);
    int ivalue = ifield; // or ifield.get();

    // static field is the same except using the static function JObject::staticField
    struct MyStrFieldS : FieldTag { static const char* name() {return "myStaticStrFieldName";} };
    auto& ifields = JObject<MyClassTag>::staticField<std::string, MyIntFieldS>(); // it can be an ref
    jfieldID ifids = ifields; // or ifield.id()
    ifields.set("JMI static field test");
    ifields = "assign";
    std::string ivalues = ifields; // or ifield.get();
```

Uncacheable jfieldID using field name string directly

```
    auto ifield = obj.field<int>("myIntFieldName");
    ...
```

### Writting a C++ Class for a Java Class

Create a class inherits JObject<YouClassTag> or stores it as a member, or use CRTP JObject<YouClass>. Each method implementation is usually less then 2 lines of code. See [JMITest](test/JMITest.h)

### Using Signatures Generated by Compiler

The function template `string signature_of(T)` returns the signature of type T. T can be jni fundamental types (except jobject types), several c++ types such as const char*, string, vector, array, valarray, reference_wrapper, void, and function types whose return type and parameter types are of above types.

`string signature_of(T)` result string has static storage duration, so it's safe to use `signature_of(...).data()`.

example:

```
    void native_test_impl(JNIEnv *env , jobject thiz, ...) {}

    staitc const JNINativeMethod gMethods[] = {
        {"native_test", signature_of(native_test_impl).data(), native_test_impl},
        ...
    };
```

You may find that a macro can simplify above example:

```
    #define DEFINE_METHOD(M) {#M, signature_of(M##_impl).data(), M##_impl}
    staitc const JNINativeMethod gMethods[] = {
        DEFINE_METHOD(native_test),
        ...
    }
```

### Convenience Functions
- to_string(std::string)
- from_string(jstring)
- getEnv()
- android::application()

### Known Issues

- If return type and first n arguments of call/call_static are the same, explicitly specifying return type and n arguments type is required

### Why JObject is a Template?
- To support per class jclass, per method jmethodID, per field jfieldID cache

#### Tested Compilers

g++ >= 4.9, clang >= 3.4.2

### TODO
- some C++ classes for frequently used java classes
- modern C++ class generator script

#### MIT License
>Copyright (c) 2016-2018 WangBin

