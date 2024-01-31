load(":refresh_compile_commands.bzl", "refresh_compile_commands")

# See README.md for interface.

# But if you aren't doing any cross-compiling for other platforms, the following can be a good default:
# bazel run @hedron_compile_commands//:refresh_all
refresh_compile_commands(
    name = "refresh_all",
)


# Stardoc users only: Depend on "@hedron_compile_commands//:bzl_srcs_for_stardoc" as needed.
# Why? Stardoc requires all loaded files to be listed as deps; without this we'd prevent users from running Stardoc on their code when they load from this tool in, e.g., their own workspace.bzl or wrapping macros.
filegroup(
    name = "bzl_srcs_for_stardoc",
    visibility = ["//visibility:public"],
    srcs = glob(["**/*.bzl"]) + [
        "@bazel_tools//tools:bzl_srcs",
        "@hedron_compile_commands_pip//:requirements.bzl",
        "@python_3_11//:defs.bzl",
        "@rules_python//:bzl",
    ],
)



########################################
# Implementation:
# If you are looking into the implementation, start with the overview in ImplementationReadme.md.

exports_files(["refresh.template.py"])  # For implicit use by the refresh_compile_commands macro, not direct use.

cc_binary(
    name = "print_args",
    srcs = ["print_args.cpp"],
    visibility = ["//visibility:public"],
)

# Quick test for https://github.com/bazelbuild/rules_python/issues/1732#issuecomment-1918268343. Delete when resolved.
load("@python_3_11//:defs.bzl", "py_binary")
py_binary(
    name = "nvcc_clang_diff",
    srcs = ["nvcc_clang_diff.py"],
)
