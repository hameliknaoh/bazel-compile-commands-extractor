<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Compile Commands Extractor for Bazel](#compile-commands-extractor-for-bazel)
  - [Quick Example](#quick-example)
  - [Usage](#usage)
    - [First, add this tool to your Bazel setup.](#first-add-this-tool-to-your-bazel-setup)
    - [Second, get the extractor running.](#second-get-the-extractor-running)
      - [There are four common paths:](#there-are-four-common-paths)
        - [1. Have a relatively simple codebase, where every target builds without needing any additional configuration or flags beyond what's in .bazelrc?](#1-have-a-relatively-simple-codebase-where-every-target-builds-without-needing-any-additional-configuration-or-flags-beyond-whats-in-bazelrc)
        - [2. Are there Bazel flags, e.g., `--config=my_important_flags_or_toolchains --compilation_mode=dbg`, that you apply manually apply to all your builds while developing?](#2-are-there-bazel-flags-eg---configmy_important_flags_or_toolchains---compilation_modedbg-that-you-apply-manually-apply-to-all-your-builds-while-developing)
        - [3. Often, though, you'll want to specify the top-level, output targets you care about and/or what flags they individually need. This avoids issues where some targets can't be built on their own; they need configuration on the command line or by a parent rule. An example of the latter is an android_library, which probably cannot be built independently of the android_binary that configures it.](#3-often-though-youll-want-to-specify-the-top-level-output-targets-you-care-about-andor-what-flags-they-individually-need-this-avoids-issues-where-some-targets-cant-be-built-on-their-own-they-need-configuration-on-the-command-line-or-by-a-parent-rule-an-example-of-the-latter-is-an-android_library-which-probably-cannot-be-built-independently-of-the-android_binary-that-configures-it)
        - [4. Using `ccls` or another tool that, unlike `clangd`, doesn't want or need headers in `compile_commands.json`?](#4-using-ccls-or-another-tool-that-unlike-clangd-doesnt-want-or-need-headers-in-compile_commandsjson)
    - [If you've got a very large project and `compile_commands.json` is taking a while to generate:](#if-youve-got-a-very-large-project-and-compile_commandsjson-is-taking-a-while-to-generate)
  - [Editor Setup — for autocomplete based on `compile_commands.json`](#editor-setup--for-autocomplete-based-on-compile_commandsjson)
    - [VSCode](#vscode)
      - [If you work on your repository with others...](#if-you-work-on-your-repository-with-others)
    - [Other Editors](#other-editors)
  - ["Smooth Edges" — what we've enjoyed using this for](#smooth-edges--what-weve-enjoyed-using-this-for)
    - [Here's what you should be expecting, based on our experience:](#heres-what-you-should-be-expecting-based-on-our-experience)
  - [Rough Edges](#rough-edges)
  - [Other Projects Likely Of Interest](#other-projects-likely-of-interest)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Compile Commands Extractor for Bazel

**What is this project trying to do for me?**

First, provide Bazel users cross-platform autocomplete for the C language family (C++, C, Objective-C, Objective-C++, and CUDA), and thereby make development more efficient and fun!

More generally, export Bazel build actions into the `compile_commands.json` format that enables great tooling decoupled from Bazel.

## Quick Example

Let's extract the [`compile_commands.json` database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) to enrich the C/C++ LSP server in your editor.

1. Update `MODULE.bazel`: Add `hedron_compile_commands` bazel module

```starlark
# MODULE.bazel

bazel_dep(name = "hedron_compile_commands", version="0.1")

git_override(
    module_name = "hedron_compile_commands",
    remote = "https://github.com/hameliknaoh/bazel-compile-commands-extractor.git",
    branch = "main",
)
```

2. Update `BUILD` and add targets of interests e.g. `//:main`:

```starlark
load("@hedron_compile_commands//:refresh_compile_commands.bzl", "refresh_compile_commands")

refresh_compile_commands(
    name = "refresh_compile_commands",

    targets = {
      "//:main":"",
    },
)

# ...

cc_binary(
    name = "main",
    srcs = ["main.cpp"],
    # deps = [...],
)
```

3. Build the target e.g. `main` target, and when successful, run `bazelisk run :refresh_compile_commands` to generate `compile_commands.json`:

```bash
# Build main target
» bazelisk build //:main
INFO: Analyzed target //:main (0 packages loaded, 2571 targets configured).
INFO: Found 1 target...
Target //:main up-to-date:
  bazel-bin/main
INFO: Elapsed time: 1.786s, Critical Path: 0.04s
INFO: 1 process: 848 action cache hit, 1 internal.
INFO: Build completed successfully, 1 total action

# Generate compile_commands.json
» bazelisk run :refresh_compile_commands
INFO: Analyzed target //:refresh_compile_commands (0 packages loaded, 2369 targets configured).
INFO: Found 1 target...
Target //:refresh_compile_commands up-to-date:
  bazel-bin/refresh_compile_commands
  bazel-bin/refresh_compile_commands.check_python_version.py
  bazel-bin/refresh_compile_commands.py
INFO: Elapsed time: 0.556s, Critical Path: 0.02s
INFO: 1 process: 9 action cache hit, 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/refresh_compile_commands
>>> Analyzing commands used in //:main
>>> Finished extracting commands for //:main

# List compile_commands.json
» ls -al compile_commands.json
-rw-rw-r-- 1 test test 7719283 Feb 14 19:50 compile_commands.json
```

## Usage

> Basic Setup Time: 10m

Howdy, Bazel user 🤠. Let's get you set up fast with some awesome tooling for the C language family.

There's a bunch of text here but only because we're trying to spell things out and make them easy. If you have issues, let us know; we'd love your help making things even better and more complete—and we'd love to help you!

### First, add this tool to your Bazel setup.

Copy this into your `MODULE.bazel`:

```starlark
# MODULE.bazel

bazel_dep(name = "hedron_compile_commands", version="0.1")

git_override(
    module_name = "hedron_compile_commands",
    remote = "https://github.com/hameliknaoh/bazel-compile-commands-extractor.git",
    branch = "main",
)
```

### Second, get the extractor running.

We'll generate a `compile_commands.json` file in the root of the Bazel workspace.

That file describes how Bazel is compiling all the (Objective-)C(++) or CUDA files. With the compile commands in a common format, build-system-independent tooling (e.g. `clangd` autocomplete, `clang-tidy` linting etc.), can get to work.

We'll get it running and then move onto the next section while it whirrs away. But in the future, every time you want tooling (like autocomplete) to see new `BUILD`-file changes, rerun the command you chose below! Clangd will automatically pick up the changes.

#### There are four common paths:

##### 1. Have a relatively simple codebase, where every target builds without needing any additional configuration or flags beyond what's in .bazelrc?

In that case, just `bazel run @hedron_compile_commands//:refresh_all`

Note: you have to `bazel run` this tool, not just `bazel build` it.

##### 2. Are there Bazel flags, e.g., `--config=my_important_flags_or_toolchains --compilation_mode=dbg`, that you apply manually apply to all your builds while developing?

It's fairly important that you supply those same Bazel flags when running this tool, too, so we can accurately understand the build, where files are being generated, etc.

Append, e.g. `-- --config=my_important_flags_or_toolchains --compilation_mode=dbg` to the above, or whatever flags you normally build with while developing.

Note: the extra `--` is not a typo, and functions to pass the flags to this tool when it runs rather than when it builds. Your command should look like:

`bazel run @hedron_compile_commands//:refresh_all -- --config=my_important_flags_or_toolchains --compilation_mode=dbg`

##### 3. Often, though, you'll want to specify the top-level, output targets you care about and/or what flags they individually need. This avoids issues where some targets can't be built on their own; they need configuration on the command line or by a parent rule. An example of the latter is an android_library, which probably cannot be built independently of the android_binary that configures it.

In that case, you can easily specify the top-level, output targets you're working on and the flags needed to build them.

Open a `BUILD` file—we'd recommend using (or creating) `//BUILD`—and add something like:

```Starlark

# BUILD

load("@hedron_compile_commands//:refresh_compile_commands.bzl", "refresh_compile_commands")

refresh_compile_commands(
    name = "refresh_compile_commands",

    # Specify the targets of interest.
    # For example, specify a dict of targets and any flags required to build.
    targets = {
      "//:my_output_1": "--important_flag1 --important_flag2=true",
      "//:my_output_2": "",
    },
    # No need to add flags already in .bazelrc. They're automatically picked up.
    # If you don't need flags, a list of targets is also okay, as is a single target string.
    # Wildcard patterns, like //... for everything, *are* allowed here, just like a build.
      # As are additional targets (+) and subtractions (-), like in bazel query https://docs.bazel.build/versions/main/query.html#expressions
    # And if you're working on a header-only library, specify a test or binary target that compiles it.
)
```

(For more details on `refresh_compile_commands`, look at the docs at the top of [`refresh_compile_commands.bzl`](./refresh_compile_commands.bzl)).

Finally, you'll need to `bazel run :refresh_compile_commands`

##### 4. Using `ccls` or another tool that, unlike `clangd`, doesn't want or need headers in `compile_commands.json`?

Similar to the above, we'll use `refresh_compile_commands` for configuration, but instead of setting `targets`, set `exclude_headers = "all"`.

### If you've got a very large project and `compile_commands.json` is taking a while to generate:

Adding `exclude_external_sources = True` and `exclude_headers = "external"` can help, with some tradeoffs.

For now, we'd suggest continuing on to set up `clangd` (below). Thereafter, if you your project proves to be large enough that it stretches the capacity of `clangd` and/or this tool to index quickly, take a look at the docs at the top of [`refresh_compile_commands.bzl`](./refresh_compile_commands.bzl) for instructions on how to tune those flags and others.

## Editor Setup — for autocomplete based on `compile_commands.json`

### VSCode

Let's get `clangd`'s extension installed and configured.

```Shell
code --install-extension llvm-vs-code-extensions.vscode-clangd
# We also need make sure that Microsoft's C++ extension is not involved and interfering.
code --uninstall-extension ms-vscode.cpptools
```

Then, open VSCode _user_ settings, so things will be automatically set up for all projects you open.

Search for "clangd".

Add the following three separate entries to `"clangd.arguments"`:

```Shell
--header-insertion=never
--compile-commands-dir=${workspaceFolder}/
--query-driver=**
```

(Just copy each as written; VSCode will correctly expand `${workspaceFolder}` for each workspace.)

- They get rid of (overzealous) header insertion; locate the compile commands correctly, even when browsing system headers outside the source tree; and cause `clangd` to interrogate Bazel's compiler wrappers to figure out which system headers are included by default.
- If your Bazel `WORKSPACE` is a subdirectory of your project, change `--compile-commands-dir` to point into that subdirectory by overriding the flags in your _workspace_ settings. You'll need to re-specify all the flags when you override, because the workspace settings replace all the flags in the user settings.

<!-- At least until https://github.com/clangd/vscode-clangd/issues/138 is resolved. -->

Turn on: Clangd: Check Updates

- You always want the latest! New great features and fixes are always getting added to clangd.
- We'll assume you always have the latest and aren't using an old version nor Apple's `clangd` intended for Xcode. While we can and do make great efforts to workaround issues in the current version of `clangd`, we remove those workarounds when `clangd` fixes them upstream. This keeps the code simple and development velocity fast!

If turning on automatic updates doesn't prompt you to download the actual `clangd` server binary, hit (CMD/CTRL+SHIFT+P)->Download language Server.

You may need to subsequently reload VSCode [(CMD/CTRL+SHIFT+P)->reload] for the plugin to load. The `clangd` download should prompt you to do so when it completes.

#### If you work on your repository with others...

... and would like these settings to be automatically applied for your teammates, also add the settings to the VSCode _workspace_ settings and then check `.vscode/settings.json` into source control.

### Other Editors

If you're using another editor, you'll need to follow the same rough steps as above: [get the latest version of clangd set up to extend the editor](https://clangd.llvm.org/installation.html#editor-plugins) and then supply the same flags as VSCode. We know people have had an easy time setting up this tool with other editors, like Emacs and Vim+YouCompleteMe(YCM), for example.

Once you've succeeded in setting up another editor—or set up `clang-tidy`, or otherwise seen anything that might improve this readme—we'd love it if you'd give back and contribute what you know! Just edit this `README.md` on GitHub and file a PR :)

## "Smooth Edges" — what we've enjoyed using this for

You should now be all set to go! Way to make it through setup.

There should be a `compile_commands.json` file in the root of your workspace, enabling your editor to provide great, clang-based autocomplete. And you should know what target to `bazel run` to refresh that autocomplete, when you make `BUILD`-file changes big enough to require a refresh.

Behind the scenes, that `compile_commands.json` file contains entries describing all the commands used to build every source file in your project. And, for now, there's also one entry per header, describing one way it is compiled. (This gets you great autocomplete in header files, too, so you don't have to think about [`clangd`'s biggest rough edge](https://github.com/clangd/clangd/issues/123)). Crucially, all these commands have been sufficiently de-Bazeled for clang tooling (or you!) to understand them.

### Here's what you should be expecting, based on our experience:

We use this tool every day to develop a cross-platform library for iOS and Android on macOS. Expect Android completion in Android source, macOS in macOS, iOS in iOS, etc. People use it on Linux/Ubuntu and Windows, too.

All the usual clangd features should work. CMD/CTRL+click navigation (or option if you've changed keybindings), smart rename, autocomplete, highlighting etc. Everything you expect in an IDE should be there (because most good IDEs are backed by `clangd`). As a general principle: If you're choosing tooling that needs to understand a programming language, you want it to be based on a compiler frontend for that language, which clangd does as part of the LLVM/clang project.

Everything should also work for generated files, though you may have to run a build for the generated file to exist. If you're using this with remote execution or cache, you'll likely have to use `--remote_download_regex` to pull down the header and source files and to avoid errors in-editor, now that build without the bytes (`--remote_download_toplevel`) is the Bazel default. If you work through this, we'd love it if you'd give back and file a PR adding good instructions for everyone else --or at least share what you've learned in an issue. You'll also want to pull down \*.d dependency files on non-Windows; they let us find headers much faster when they're available as a cache. We'd appreciate if you'd also check to make sure that they're pulled down even without (`--noexperimental_inmemory_dotd_files`). Thanks for helping!

## Rough Edges

Otherwise, we've self-filed issues for the rough edges we know about and are tracking. We'd love to hear from you there about what you're seeing, good and bad. Please add things if you find more rough edges, and let us know if you need help or more features.

On the other hand, if you've set things up and they're working well, we'd still love to hear from you. Please file a "non-issue" in the issues tab describing your success! We'd love to hear what you're working on, what platforms you're using, and what you're finding most useful. And maybe also toss a star our way so we know it was helpful to you.

We'd also love to work with you on contributions and improvements, of course! Development setup is easy, not onerous; we've got [a great doc to guide you quickly into being able to make the changes you need.](./ImplementationReadme.md) The codebase is super clean and friendly. Stepping into the code is a fun and efficient way to get the improvements you want.

---

## Other Projects Likely Of Interest

If you're using Bazel for the C language family, you'll likely also want some of our other tooling, like...

1. A good way of making secure network requests: [hedronvision/bazel-make-cc-https-easy](https://github.com/hedronvision/bazel-make-cc-https-easy)
2. A way to use std::filesystem across platforms: [hedronvision/bazel-cc-filesystem-backport](https://github.com/hedronvision/bazel-cc-filesystem-backport)

---

_Looking for implementation details instead? Want to dive into the codebase?_
See [ImplementationReadme.md](./ImplementationReadme.md).

_Bazel/Blaze maintainer reading this?_ If you'd be interested in integrating this into official Bazel tools, let us know in an issue or email, and let's talk! We love getting to use Bazel and would love to help.
