## Playing with Dependencies
Oct 27, 2026

---

Ive always wondered what's the deal with dependencies in C/C++ and why was it so hard to integrate code to an application compared to applications developed in other languages that usually are built using a package manager.

After playing for a bit with and with some confidence lets try to clarify some ideas so at least the cornerstone is set

From the perspective of creating a dependency, lets say raylib.
The two options to create a library are runtime and static.

A runtime library allows your application to access code that doesnt come from itself, you only have to permanently add to your code the import library to resolve runtime symbols.


##### As a lib provider

How to create a runtime library using the compiler and linker from Microsoft:
```
cl /LD "src/test_lib.cpp" /link /OUT:"target/MT/test_lib.dll" /IMPLIB:"target/MT/test_libdll.lib"
```


How to create a static library using the compiler and linker from Microsoft:
```
cl /c "src/test_lib.cpp"
lib /OUT:"target/test_lib.lib" "test_lib.obj"
```
Here we need another step, you can imagine the lib command like an aggregator of all the object files made by the compiler.

##### As a lib user

To link your application with any library wou would need one of the two, the static library or the header library.

Also make sure to have the .h header files so you have the api available

How to integrate a static library to your application:
```
cl /Iinclude "src/main.cpp" /link "test_lib.lib"
```
You can think of this step to be like, just add all the object files from the .lib file to my application, this is important because at the time of deploying my application, it wont need anything else, as a result the weight of the application will be bigger.

How to integrate a runtime library to your application:
```
cl /Iinclude "src/main.cpp" /link "/test_libdll.lib""
```
Here the step is the same, but for your application to be able to run remember it need the .dll file available at runtime. So the distribution of the application is different


##### Here we got the full example

###### 1) Creating runtime dependency linked against static crt

```
cl /nologo /std:c++17 /EHsc /LD /MT /Iinclude "src/test_lib.cpp" /Fo:"target/MT/" ^
   /link /OUT:"target/MT/test_lib.dll" /IMPLIB:"target/MT/test_libdll.lib"

Dump of file .\target\MT\test_lib.dll

File Type: DLL

  Image has the following dependencies:

    KERNEL32.dll
```

Linking against that runtime dependency from the application.

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
Here you can see that in order to run the application test_lib.dll and KERNERL32 (inherited from test_lib) need to be accessible at runtime.

###### 2) Creating runtime dependency linked against runtime crt

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
Here you can see the whole crt now is needed

Linking against that runtime dependency from the application.

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

Remember that the library and the application need to be linked the same way, you cant link your application with the runtime crt and the lib being linked with static crt, that would generate a conflict

###### 3) Creating static dependency linked against runtime crt

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

Here you can see it has no dependencies.

```
cl /nologo /std:c++17 /EHsc /MT /Iinclude "src/main.cpp" /Fo:"target/" /Fe:"target/main.exe" /link "target/test_lib.lib"

Dump of file .\target\main.exe

File Type: EXECUTABLE IMAGE

  Image has the following dependencies:

    KERNEL32.dll
```

---
##### Conclusion

Now its clear why it is a pain to distribute application and libraries without a package manager. You have to make sure you are using the correct version of the c runtime (glibc, muslc, wincrt) and also the user has to know what you are providing him with and how to link his application with.
And linking your application with the dependency is not the end of the story, you will also have problems with naming conflicts, how the dependency is managing memory etc.
Thats why language programing standards and good practices, even tho they add more work, end up generating lest headache.
One of the advantages modern languages have over legacy ones for sure.
Also this problem is completely managable once understood.