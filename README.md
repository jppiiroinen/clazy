# Source Code

You can get clazy from:

- https://github.com/KDE/clazy
- git@git.kde.org:clazy
- http://anongit.kde.org/clazy

--------------------------------------------------------------------------------
# Build Instructions (Linux)

##### Install Dependencies:
- OpenSUSE tumbleweed: zypper install cmake git-core llvm llvm-devel llvm-clang llvm-clang-devel
- Ubuntu-16.04: apt-get install g++ cmake clang llvm git-core libclang-3.8-dev qtbase5-dev
- Archlinux: pacman -S make llvm clang python2 cmake qt5-base git gcc
- Other distros: Check llvm/clang build docs.

##### Build and install clang >= 3.6 if your distro doesn't provide it:
```
  $ git clone https://github.com/llvm-mirror/llvm.git <some_directory>
  $ cd <some_directory>/tools/ && git clone https://github.com/llvm-mirror/clang.git
  $ cd <some_directory>/projects && git clone https://github.com/llvm-mirror/compiler-rt.git
  $ mkdir <some_directory>/build && cd <some_directory>/build
  $ cmake -DCMAKE_INSTALL_PREFIX=<prefix> -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_BUILD_TYPE=Release ..
  $ make -jX && make install
```

##### Build the clazy plugin:
```
  $ cd clazy/
  $ cmake -DCMAKE_INSTALL_PREFIX=<prefix> -DCMAKE_BUILD_TYPE=Release
  $ make && make install
```

See troubleshooting section if you have problems.

--------------------------------------------------------------------------------
# Build Instructions (Windows)

The instructions assume your terminal is suitable for development (msvc2013).
jom, nmake, git, cmake and cl should be in your PATH.

##### Build and install llvm and clang from master (3.9):
```
  > git clone https://github.com/llvm-mirror/llvm.git <some_directory>
  > cd <some_directory>\tools\ && git clone https://github.com/llvm-mirror/clang.git
  > cd clang
  > git cherry-pick ae1cd1e7c301954bab703e9116a30b330902d43a
  > git cherry-pick bce41007c954eafd1d2fdcecbf4cc007697901e7
  > mkdir <some_directory>\build && cd <some_directory>\build
  > cmake -DCMAKE_INSTALL_PREFIX=c:\my_install_folder\llvm\ -DLLVM_TARGETS_TO_BUILD="X86" -DCMAKE_BUILD_TYPE=Release -G "NMake Makefiles JOM" ..
  > jom
  > nmake install
  > Add c:\my_install_folder\llvm\ to PATH
```

##### Build the clazy plugin:
```
  > cd clazy\
  > cmake -DCMAKE_INSTALL_PREFIX=c:\my_install_folder\llvm\ -DCMAKE_BUILD_TYPE=Release -G "NMake Makefiles JOM" -DCLAZY_ON_WINDOWS_HACK=ON
  > jom && nmake install
  ```
--------------------------------------------------------------------------------
# Setting up your project to build with clazy

You should now have the clazy command available to you, in <prefix>/bin/.
Compile your programs with it instead of clang++/g++.

Note that this command is just a convenience wrapper which calls:
`clang++ -Xclang -load -Xclang ClangLazy.so -Xclang -add-plugin -Xclang clang-lazy`

If you have multiple versions of clang installed (say clang++-3.5 and clang++-3.6)
you can choose which one to use by setting the CLANGXX environment variable, like so:
`export CLANGXX=clang++-3.6; clazy`

To build a CMake project use:
 `cmake . -DCMAKE_CXX_COMPILER=clazy`
and rebuild.

To make it the compiler for qmake projects, just run qmake like:
`qmake -spec linux-clang QMAKE_CXX="clazy"`

Alternatively, if you want to use clang directly, without the wrapper:
`qmake -spec linux-clang QMAKE_CXXFLAGS="-Xclang -load -Xclang ClangLazy.so -Xclang -add-plugin -Xclang clang-lazy"`

You can also edit mkspecs/common/clang.conf and change QMAKE_CXX to clazy instead of clang++ and run qmake -spec linux-clang

You're all set, clazy will now run some checks on your project, but not all of them.
Read on if you want to enable/disable which checks are run.

--------------------------------------------------------------------------------
# Selecting which checks to enable

You may want to choose which checks to enable before starting to compile.
There are many checks and they are divided in levels:
```
Checks from level0:
    container-anti-pattern
    wrong-qglobalstatic
    writing-to-temporary
    qstring-insensitive-allocation
    qvariant-template-instantiation
    qstring-ref    (fix-missing-qstringref)
    temporary-iterator
    qstring-arg
    qmap-with-pointer-key
    qdatetime-utc    (fix-qdatetime-utc)
    qgetenv    (fix-qgetenv)
    qfileinfo-exists

Checks from level1:
    range-loop
    non-pod-global-static
    missing-qobject-macro
    qdeleteall
    inefficient-qlist-soft
    foreach
    detaching-temporary
    auto-unexpected-qstringbuilder    (fix-auto-unexpected-qstringbuilder)
    qstring-left

Checks from level2:
    qstring-allocations    (fix-qlatin1string-allocations,fix-fromLatin1_fromUtf8-allocations,fix-fromCharPtrAllocations)
    rule-of-two-soft
    rule-of-three
    reserve-candidates
    virtual-call-ctor
    old-style-connect    (fix-old-style-connect)
    function-args-by-ref
    implicit-casts
    global-const-char-pointer
    container-inside-loop

Checks from level3:
    bogus-dynamic-cast
    detaching-member
    copyable-polymorphic
    assert-with-side-effects
```
#### Description of each level
- level0: Very stable checks, 99.99% safe, no false-positives
- level1: Similar to level0, but sometimes (rarely) there might be some false-positives
- level2: Sometimes has false-positives (20-30%).
- level3: Not always correct, possibly very noisy, might require a knowledgeable developer to review, might have a very big rate of false-positives, might have bugs.

If you don't specify anything then all checks from level0 and level1 will run.
To specify a list of checks to run, or to choose a level, you can use the `CLAZY_CHECKS` env variable or pass arguments to the compiler.

##### Example via env variable
```
export CLAZY_CHECKS="bogus-dynamic-cast,qmap-with-key-pointer,virtual-call-ctor" # Enables only these 3 checks
export CLAZY_CHECKS="level0" # Enables all checks from level0
export CLAZY_CHECKS="level0,detaching-temporary" # Enables all from level0 and also detaching-temporary
```
##### Example via compiler argument
`clazy -Xclang -plugin-arg-clang-lazy -Xclang level0,detaching-temporary`
Don't forget to re-run cmake/qmake/etc if you altered the c++ flags to specify flags.

--------------------------------------------------------------------------------
# Enabling Fixits

Some checks support fixits, in which clazy will re-write your source files whenever it can fix something.
You can enable a fixit through the env variable, for example:
`export CLAZY_FIXIT="fix-qlatin1string-allocations"`

Only one fixit can be enabled each time.
**WARNING**: Backup your code, don't blame me if a fixit is not applied correctly.
For better results don't use parallel builds, otherwise a fixit being applied in an header file might be done twice.

--------------------------------------------------------------------------------
# Troubleshooting

- clang: symbol lookup error: /usr/lib/x86_64-linux-gnu/ClangLazy.so: undefined symbol: _ZNK5clang15DeclarationName11getAsStringEv
  This is due to mixing ABIs. Your clang/llvm was compiled with the new gcc c++ ABI [1] but you compiled the clazy plugin with clang (which uses old ABI [2]).

  The solution is to build the clazy plugin with gcc or use a distro which hasn't migrated to gcc5 ABI yet, such as archlinux.

  [1] https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html
  [2] https://llvm.org/bugs/show_bug.cgi?id=23529

- [Fedora 23] cmake can't find LLVM ?
  Try installing the llvm-static package. If that doesn't work you'll have to build llvm/clang yourself
  (There are reports that /usr/share/llvm/cmake/LLVM-Config.cmake is buggy).

- Some checks are misteriously not producing warnings or not applying fixits ?
  Check if you have ccache interfering and turn it off.

--------------------------------------------------------------------------------
# Reducing warning noise

- If you think you found a false-positive, file a bug report.
- If you want to suppress warnings from headers of Qt or 3rd party code, include them with -isystem instead of -I.

- You can also suppress individual warnings by file or by line by inserting comments:

  To disable clazy in a specific source file, insert this comment, anywhere in the file:
`// clazy:skip`

  To disable specific checks in a source file, insert a comment such as
`// clazy:excludeall=check1,check2`

  To disable specific checks in specific source lines, insert a comment in the same line as the warning:
`(...) // clazy:exclude=check1,check2`

  Don't include the "clazy-" prefix. If, for example, you want to disable qstring-allocations you would write:
`// clazy:exclude=qstring-allocations` not clazy-qstring-allocations.

--------------------------------------------------------------------------------
# Reporting bugs and wishes

- bug tracker: https://bugs.kde.org/enter_bug.cgi?product=clazy
- IRC: #kde-clazy (freenode)
- E-mail: smartins@kde.org

--------------------------------------------------------------------------------
# Contributing patches
https://git.reviewboard.kde.org

--------------------------------------------------------------------------------
