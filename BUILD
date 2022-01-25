load("@rules_cc//cc:defs.bzl", "objc_library")
load("@build_bazel_rules_swift//swift:swift.bzl", "swift_library")

DEFINES_1 = ["FOO=A B C"]

DEFINES_2 = ["\'FOO=X Y Z\'"]

objc_library(
    name = "foo_objc_1",
    srcs = [
        "foo.m",
    ],
    defines = DEFINES_1,
    deps = [":foo_swift_1"],
)

swift_library(
    name = "foo_swift_1",
    srcs = [
        "foo.swift",
    ],
    defines = DEFINES_1,
)

objc_library(
    name = "foo_objc_2",
    srcs = [
        "foo.m",
    ],
    defines = DEFINES_2,
    deps = [":foo_swift_2"],
)

swift_library(
    name = "foo_swift_2",
    srcs = [
        "foo.swift",
    ],
    defines = DEFINES_2,
)
