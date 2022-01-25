#### Issues with passing same list of defines to both `objc_library` and `swift_library`

To follow what is detailed below check the `BUILD` file in this repo and note two `objc_library` targets (`foo_objc_1`, `foo_objc_2`) each depending on an `swift_library` target (`foo_swift_1`, `foo_swift_2`). With this setup and passing the same list of preprocessor macros to both libraries in the `defines` attribute undesired behavior is triggered.

ps: (1) In the build invocations below one can pass `--action_env=$(date +%S)` to force invalidation and avoid hitting cached artifacts in order to build the same target multiple times (2) The `--subcommands` flag is used to expose the compiler invocations.

#### Duplicated values and undesired split by space due to Bourne shell tokenization

The list of defines below is passed to both `foo_objc_1` and `foo_swift_1`
```python
DEFINES_1 = ["FOO=A B C"]
```
from here one can build `foo_objc_1`
```sh
bazel build --subcommands :foo_objc_1
```
and even though it compiles note the underlying clang invocation
```sh
external/local_config_cc/wrapped_clang -arch x86_64 '-D_FORTIFY_SOURCE=1' -fstack-protector -fcolor-diagnostics -Wall -Wthread-safety -Wself-assign -fno-omit-frame-pointer -O0 -DDEBUG 'DEBUG_PREFIX_MAP_PWD=.' -Wshorten-64-to-32 -Wbool-conversion -Wconstant-conversion -Wduplicate-method-match -Wempty-body -Wenum-conversion -Wint-conversion -Wunreachable-code -Wmismatched-return-types -Wundeclared-selector -Wuninitialized -Wunused-function -Wunused-variable -iquote . -iquote bazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin -Ibazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin -MD -MF bazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin/_objs/foo_objc_1/arc/foo.d '-DFOO=A B C' '-DFOO=A' -DB -DC -DOS_MACOSX -fno-autolink -isysroot __BAZEL_XCODE_SDKROOT__ -F__BAZEL_XCODE_SDKROOT__/System/Library/Frameworks -F__BAZEL_XCODE_DEVELOPER_DIR__/Platforms/MacOSX.platform/Developer/Library/Frameworks -fobjc-arc '-mmacosx-version-min=11.3' -O0 '-DDEBUG=1' -c foo.m -o bazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin/_objs/foo_objc_1/arc/foo.o
```
in particular
```sh
'-DFOO=A B C' '-DFOO=A' -DB -DC
```

Note how `FOO` is duplicated (which is expected since we're passing the same list to both). The `'-DFOO=A B C'` occurence comes from the fact that `swift_library` propagates `defines` to [all targets that depend on it](https://github.com/bazelbuild/rules_swift/blob/master/doc/rules.md#swift_library), important to note that `KEY=VALUE` pairs shouldn't be passed to `swift_library` in the first place but it's being done here to simulate a scenario where one would pass the same list of defines to both libraries. 

The `'-DFOO=A' -DB -DC` occurence comes from the `objc_library` and the split is due to [Bourne shell tokenization](https://docs.bazel.build/versions/0.29.0/be/common-definitions.html#sh-tokenization) which is undesired. One could decide to then wrap the macro in quotes, e.g. `\'FOO=X Y Z\'` but this would cause a different issue (see section below).

Note the warnings triggered by this:
```sh
INFO: From Compiling foo.m:
In file included from <built-in>:379:
<command line>:4:9: warning: 'FOO' macro redefined [-Wmacro-redefined]
#define FOO A
        ^
<command line>:3:9: note: previous definition is here
#define FOO A B C
        ^
1 warning generated.
Target //:foo_objc_1 up-to-date:
```

#### Compilation error if consumer decides to wrap value in single quote

The list of defines below is passed to both `foo_objc_2` and `foo_swift_2`
```python
DEFINES_2 = ["\'FOO=X Y Z\'"]
```
one could do that to prevent the issue described in the previous section, this build will fail though
```sh
bazel build --subcommands :foo_objc_2
```

Note the clang invocation in this case
```sh
external/local_config_cc/wrapped_clang -arch x86_64 '-D_FORTIFY_SOURCE=1' -fstack-protector -fcolor-diagnostics -Wall -Wthread-safety -Wself-assign -fno-omit-frame-pointer -O0 -DDEBUG 'DEBUG_PREFIX_MAP_PWD=.' -Wshorten-64-to-32 -Wbool-conversion -Wconstant-conversion -Wduplicate-method-match -Wempty-body -Wenum-conversion -Wint-conversion -Wunreachable-code -Wmismatched-return-types -Wundeclared-selector -Wuninitialized -Wunused-function -Wunused-variable -iquote . -iquote bazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin -Ibazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin -MD -MF bazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin/_objs/foo_objc_2/arc/foo.d '-D'\''FOO=X Y Z'\''' '-DFOO=X Y Z' -DOS_MACOSX -fno-autolink -isysroot __BAZEL_XCODE_SDKROOT__ -F__BAZEL_XCODE_SDKROOT__/System/Library/Frameworks -F__BAZEL_XCODE_DEVELOPER_DIR__/Platforms/MacOSX.platform/Developer/Library/Frameworks -fobjc-arc '-mmacosx-version-min=11.3' -O0 '-DDEBUG=1' -c foo.m -o bazel-out/applebin_macos-darwin_x86_64-fastbuild-ST-cd2b3b8d4835/bin/_objs/foo_objc_2/arc/foo.o
```
in particular
```sh
'-D'\''FOO=X Y Z'\''' '-DFOO=X Y Z'
```

The `'-D'\''FOO=X Y Z'\'''` occurence comes from `swift_library` trying to wrap in quotes an already wrapped value and the second occurence `'-DFOO=X Y Z'` comes from `objc_library` receiving the correct value for that macro.

The build will fail due to how `swift_library` propagated the macro to its depender
```sh
<command line>:3:9: error: macro name must be an identifier
#define 'FOO X Y Z'
        ^
1 error generated.
Error in child process '/usr/bin/xcrun'. 1
Target //:foo_objc_2 failed to build
```

#### Takeaways

* Consumers should not pass the same list of defines to both `objc_library` and `swift_library`, instead explicitly pass expected values for each.
* `objc_library` preprocessor macros should be always wrapped in single quotes to prevent similar issues as described here due to Bourne shell tokenization.