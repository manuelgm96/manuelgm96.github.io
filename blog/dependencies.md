# Playing with Dependencies

Oct 27, 2026

---

I've always wondered what's the deal with dependencies in C/C++ and why it's so hard to integrate code into an application compared to applications developed in other languages, which are usually built using a package manager.

After playing around with this for a bit and gaining some confidence, let's try to clarify a few ideas so that, at least, we can lay the groundwork.

From the perspective of creating a dependency, let's say raylib, there are two options for creating a library: runtime (dynamic) and static.

A runtime library allows your application to access code that doesn't come from the application itself. You only have to link your application against the import library to resolve the runtime symbols.

### As a library provider

How to create a runtime library using Microsoft's compiler and linker:

```
cl /LD "src/test_lib.cpp" /link /OUT:"target/MT/test_lib.dll" /IMPLIB:"target/MT/test_libdll.lib"
```

How to create a static library using Microsoft's compiler and linker:

```
cl /c "src/test_lib.cpp"
lib /OUT:"target/test_lib.lib" "test_lib.obj"
```

Here, we need another step. You can think of the `lib` command as an aggregator of all the object files produced by the compiler.

### As a library user

To link your application against a library, you need one of two things: a static library or an import library.

Also, make sure you have the `.h` header files so that the API is available to your application.

How to integrate a static library into your application:

```
cl /Iinclude "src/main.cpp" /link "test_lib.lib"
```

You can think of this step as saying, "Just add all the object code from the `.lib` file to my application."

This is important because, when deploying the application, it won't need the library separately at runtime. As a result, the executable will generally be larger.

How to integrate a runtime library into your application:

```
cl /Iinclude "src/main.cpp" /link "target/MT/test_libdll.lib"
```

Here, the linking step is similar, but for your application to run, remember that the `.dll` file must be available at runtime. So, the distribution of the application is different.

### Here, we've got the full example

#### 1) Creating a runtime dependency linked against the static CRT

```
cl /nologo /std:c++17 /EHsc /LD /MT /Iinclude "src/test_lib.cpp" /Fo:"target/MT/" ^
   /link /OUT:"target/MT/test_lib.dll" /IMPLIB:"target/MT/test_libdll.lib"

Dump of file .\target\MT\test_lib.dll

File Type: DLL

  Image has the following dependencies:

    KERNEL32.dll
```

Linking against that runtime dependency from the application:

```
cl /nologo /std:c++17 /EHsc /MT /Iinclude "src/main.cpp" /Fo:"target/MT/" /Fe:"target/MT/main.exe" /link "target/MT/test_libdll.lib"

Dump of file .\target\MT\test_lib.dll

File Type: DLL

Dump of file .\target\MT\main.exe

File Type: EXECUTABLE IMAGE

  Image has the following dependencies:

    test_lib.dll
    KERNEL32.dll
```

Here, you can see that, for the application to run, `test_lib.dll` needs to be accessible at runtime. `KERNEL32.dll` is also a dependency of the application.

#### 2) Creating a runtime dependency linked against the runtime CRT

```
cl /nologo /std:c++17 /EHsc /LD /MD /Iinclude "src/test_lib.cpp" /Fo:"target/MD/" ^
   /link /OUT:"target/MD/test_lib.dll" /IMPLIB:"target/MD/test_libdll.lib"

Dump of file .\target\MD\test_lib.dll

File Type: DLL

  Image has the following dependencies:

    MSVCP140.dll
    KERNEL32.dll
    VCRUNTIME140_1.dll
    VCRUNTIME140.dll
    api-ms-win-crt-string-l1-1-0.dll
    api-ms-win-crt-runtime-l1-1-0.dll
```

Here, you can see that the DLL now depends on the runtime C/C++ libraries.

Linking against that runtime dependency from the application:

```
cl /nologo /std:c++17 /EHsc /MD /Iinclude "src/main.cpp" /Fo:"target/MD/" /Fe:"target/MD/main.exe" /link "target/MD/test_libdll.lib"

Dump of file .\target\MD\main.exe

File Type: EXECUTABLE IMAGE

  Image has the following dependencies:

    test_lib.dll
    KERNEL32.dll
    VCRUNTIME140.dll
    api-ms-win-crt-runtime-l1-1-0.dll
    api-ms-win-crt-math-l1-1-0.dll
    api-ms-win-crt-stdio-l1-1-0.dll
    api-ms-win-crt-locale-l1-1-0.dll
    api-ms-win-crt-heap-l1-1-0.dll
```

Remember that the library and the application should use compatible CRT configurations, especially when sharing memory, allocating or freeing resources, or passing CRT objects across module boundaries. Mixing CRT configurations can cause conflicts and undefined behavior.

#### 3) Creating a static dependency linked against the static CRT

```
cl /nologo /std:c++17 /EHsc /c /MT /Iinclude "src/test_lib.cpp" /Fo:"target/"
lib /nologo /OUT:"target/test_lib.lib" "target/test_lib.obj"

Dump of file .\target\test_lib.lib

File Type: LIBRARY

   Linker Directives
   -----------------
   /FAILIFMISMATCH:_MSC_VER=1900
   /FAILIFMISMATCH:_ITERATOR_DEBUG_LEVEL=0
   /FAILIFMISMATCH:RuntimeLibrary=MT_StaticRelease
   /DEFAULTLIB:libcpmt
   /FAILIFMISMATCH:_CRT_STDIO_ISO_WIDE_SPECIFIERS=0
   /alternatename:_Avx2WmemEnabled=_Avx2WmemEnabledWeakValue
   /FAILIFMISMATCH:annotate_string=0
   /FAILIFMISMATCH:annotate_vector=0
   /FAILIFMISMATCH:annotate_optional=0
   /DEFAULTLIB:LIBCMT
   /DEFAULTLIB:OLDNAMES
   /EXPORT:add
```

Here, you can see that the library doesn't have any runtime DLL dependencies listed. Instead, its linker directives specify the libraries needed to link the code, including the static CRT.

```
cl /nologo /std:c++17 /EHsc /MT /Iinclude "src/main.cpp" /Fo:"target/" /Fe:"target/main.exe" /link "target/test_lib.lib"

Dump of file .\target\main.exe

File Type: EXECUTABLE IMAGE

  Image has the following dependencies:

    KERNEL32.dll
```

---

### Conclusion

Now it's clear why distributing applications and libraries without a package manager can be a pain. You have to make sure you're using the correct version of the C runtime (glibc, musl libc, or the Windows CRT), and the user has to know what you're providing and how to link it with their application.

And linking your application against a dependency isn't the end of the story. You'll also have to deal with naming conflicts, how the dependency manages memory, and other issues.

That's why programming language standards and good practices, even though they add more work, end up saving you a lot of headaches.

This is one of the advantages modern languages have over legacy ones, for sure.

Also, this problem is completely manageable once you understand it.