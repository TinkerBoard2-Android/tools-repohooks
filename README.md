# AOSP Preupload Hooks

This repo holds hooks that get run by repo during the upload phase.  They
perform various checks automatically such as running linters on your code.

Note: Currently all hooks are disabled by default.  Each repo must explicitly
turn on any hook it wishes to enforce.

[TOC]

## Usage

Normally these execute automatically when you run `repo upload`.  If you want to
run them by hand, you can execute `pre-upload.py` directly.  By default, that
will scan the active repo and process all commits that haven't yet been merged.
See its help for more info.

### Bypassing

Sometimes you might want to bypass the upload checks.  While this is **strongly
discouraged** (often failures you add will affect others and block them too),
sometimes there are valid reasons for this.  You can simply use the option
`--no-verify` when running `repo upload` to skip all upload checks.  This will
skip **all** checks and not just specific ones.  It should be used only after
having run & evaluated the upload output previously.

# Config Files

There are two types of config files:
* Repo project-wide settings (e.g. all of AOSP).  These set up defaults for all
  projects that are checked out via a single manifest.
* Project-local settings (e.g. a single .git repo).  These control settings for
  the local project you're working on.

The merging of these config files control the hooks/checks that get run when
running `repo upload`.

## GLOBAL-PREUPLOAD.cfg

These are the manifest-wide defaults and can be located in two places:
* `.repo/manifests/GLOBAL-PREUPLOAD.cfg`: The manifest git repo.
  Simply check this in to the manifest git repo and you're done.
* `GLOBAL-PREUPLOAD.cfg`: The top level of the repo checkout.
  For manifests that don't have a project checked out at the top level,
  you can use repo's `<copyfile>` directive.

These config files will be loaded first before stacking `PREUPLOAD.cfg`
settings on top.

## PREUPLOAD.cfg

This file is checked in the top of a specific git repository.  Stacking them
in subdirectories (to try and override parent settings) is not supported.

## Example

```
# Per-project `repo upload` hook settings.
# https://android.googlesource.com/platform/tools/repohooks

[Options]
ignore_merged_commits = true

[Hook Scripts]
name = script --with args ${PREUPLOAD_FILES}

[Builtin Hooks]
cpplint = true

[Builtin Hooks Options]
cpplint = --filter=-x ${PREUPLOAD_FILES}

[Tool Paths]
clang-format = /usr/bin/clang-format
```

## Environment

Hooks are executed in the top directory of the git repository.  All paths should
generally be relative to that point.

A few environment variables are set so scripts don't need to discover things.

* `REPO_PROJECT`: The name of the project.
   e.g. `platform/tools/repohooks`
* `REPO_PATH`: The path to the project relative to the root.
   e.g. `tools/repohooks`
* `REPO_REMOTE`: The name of the git remote.
   e.g. `aosp`.
* `REPO_LREV`: The name of the remote revision, translated to a local tracking
   branch. This is typically latest commit in the remote-tracking branch.
   e.g. `ec044d3e9b608ce275f02092f86810a3ba13834e`
* `REPO_RREV`: The remote revision.
   e.g. `master`
* `PREUPLOAD_COMMIT`: The commit that is currently being checked.
   e.g. `1f89dce0468448fa36f632d2fc52175cd6940a91`

## Placeholders

A few keywords are recognized to pass down settings.  These are **not**
environment variables, but are expanded inline.  Files with whitespace and
such will be expanded correctly via argument positions, so do not try to
force your own quote handling.

* `${PREUPLOAD_FILES}`: List of files to operate on.
* `${PREUPLOAD_FILES_PREFIXED}`: A list of files to operate on.
   Any string preceding/attached to the keyword ${PREUPLOAD_FILES_PREFIXED}
   will be repeated for each file automatically. If no string is preceding/attached
   to the keyword, the previous argument will be repeated before each file.
* `${PREUPLOAD_COMMIT}`: Commit hash.
* `${PREUPLOAD_COMMIT_MESSAGE}`: Commit message.

Some variables are available to make it easier to handle OS differences.  These
are automatically expanded for you:

* `${REPO_ROOT}`: The absolute path of the root of the repo checkout.
* `${BUILD_OS}`: The string `darwin-x86` for macOS and the string `linux-x86`
  for Linux/x86.

### Examples

Here are some examples of using the placeholders.
Consider this sample config file.
```
[Hook Scripts]
lister = ls ${PREUPLOAD_FILES}
checker prefix = check --file=${PREUPLOAD_FILES_PREFIXED}
checker flag = check --file ${PREUPLOAD_FILES_PREFIXED}
```
With a commit that changes `path1/file1` and `path2/file2`, then this will run
programs with the arguments:
* ['ls', 'path1/file1', 'path2/file2']
* ['check', '--file=path1/file1', '--file=path2/file2']
* ['check', '--file', 'path1/file1', '--file', 'path2/file2']

## [Options]

This section allows for setting options that affect the overall behavior of the
pre-upload checks.  The following options are recognized:

* `ignore_merged_commits`: If set to `true`, the hooks will not run on commits
  that are merged.  Hooks will still run on the merge commit itself.

## [Hook Scripts]

This section allows for completely arbitrary hooks to run on a per-repo basis.

The key can be any name (as long as the syntax is valid), as can the program
that is executed. The key is used as the name of the hook for reporting purposes,
so it should be at least somewhat descriptive.

Whitespace in the key name is OK!

The keys must be unique as duplicates will silently clobber earlier values.

You do not need to send stderr to stdout.  The tooling will take care of
merging them together for you automatically.

```
[Hook Scripts]
my first hook = program --gogog ${PREUPLOAD_FILES}
another hook = funtimes --i-need "some space" ${PREUPLOAD_FILES}
some fish = linter --ate-a-cat ${PREUPLOAD_FILES}
some cat = formatter --cat-commit ${PREUPLOAD_COMMIT}
some dog = tool --no-cat-in-commit-message ${PREUPLOAD_COMMIT_MESSAGE}
```

## [Builtin Hooks]

This section allows for turning on common/builtin hooks.  There are a bunch of
canned hooks already included geared towards AOSP style guidelines.

* `bpfmt`: Run Blueprint files (.bp) through `bpfmt`.
* `checkpatch`: Run commits through the Linux kernel's `checkpatch.pl` script.
* `clang_format`: Run git-clang-format against the commit. The default style is
  `file`.
* `commit_msg_bug_field`: Require a valid `Bug:` line.
* `commit_msg_changeid_field`: Require a valid `Change-Id:` Gerrit line.
* `commit_msg_prebuilt_apk_fields`: Require badging and build information for
  prebuilt APKs.
* `commit_msg_relnote_field_format`: Check for possible misspellings of the
  `Relnote:` field and that multiline release notes are properly formatted with
  quotes.
* `commit_msg_relnote_for_current_txt`: Check that CLs with changes to
  current.txt or public_plus_experimental_current.txt also contain a
  `Relnote:` field in the commit message.
* `commit_msg_test_field`: Require a `Test:` line.
* `cpplint`: Run through the cpplint tool (for C++ code).
* `gofmt`: Run Go code through `gofmt`.
* `google_java_format`: Run Java code through
  [`google-java-format`](https://github.com/google/google-java-format)
* `jsonlint`: Verify JSON code is sane.
* `pylint`: Alias of `pylint2`.  Will change to `pylint3` by end of 2019.
* `pylint2`: Run Python code through `pylint` using Python 2.
* `pylint3`: Run Python code through `pylint` using Python 3.
* `rustfmt`: Run Rust code through `rustfmt`.
* `xmllint`: Run XML code through `xmllint`.
* `android_test_mapping_format`: Validate TEST_MAPPING files in Android source
  code. Refer to go/test-mapping for more details.

Note: Builtin hooks tend to match specific filenames (e.g. `.json`).  If no
files match in a specific commit, then the hook will be skipped for that commit.

```
[Builtin Hooks]
# Turn on cpplint checking.
cpplint = true
# Turn off gofmt checking.
gofmt = false
```

## [Builtin Hooks Options]

Used to customize the behavior of specific `[Builtin Hooks]`.  Any arguments set
here will be passed directly to the linter in question.  This will completely
override any existing default options, so be sure to include everything you need
(especially `${PREUPLOAD_FILES}` -- see below).

Quoting is handled naturally.  i.e. use `"a b c"` to pass an argument with
whitespace.

See [Placeholders](#Placeholders) for variables you can expand automatically.

```
[Builtin Hooks Options]
# Pass more filter args to cpplint.
cpplint = --filter=-x ${PREUPLOAD_FILES}
```

## [Builtin Hooks Exclude Paths]

*** note
This section can only be added to the repo project-wide settings
[GLOBAL-PREUPLOAD.cfg].
***

Used to explicitly exclude some projects when processing a hook. With this
section, it is possible to define a hook that should apply to the majority of
projects except a few.

An entry must completely match the project's `REPO_PATH`. The paths can use the
[shell-style wildcards](https://docs.python.org/library/fnmatch.html) and
quotes. For advanced cases, it is possible to use a [regular
expression](https://docs.python.org/howto/regex.html) by using the `^` prefix.

```
[Builtin Hooks Exclude Paths]
# Run cpplint on all projects except ones under external/ and vendor/.
# The "external" and "vendor" projects, if they exist, will still run cpplint.
cpplint = external/* vendor/*

# Run rustfmt on all projects except ones under external/.  All projects under
# hardware/ will be excluded except for ones starting with hardware/google (due to
# the negative regex match).
rustfmt = external/ ^hardware/(!?google)
```

## [Tool Paths]

Some builtin hooks need to call external executables to work correctly.  By
default it will call those tools from the user's `$PATH`, but the paths of those
executables can be overridden through `[Tool Paths]`.  This is helpful to
provide consistent behavior for developers across different OS and Linux
distros/versions.  The following tools are recognized:

* `bpfmt`: used for the `bpfmt` builtin hook.
* `clang-format`: used for the `clang_format` builtin hook.
* `cpplint`: used for the `cpplint` builtin hook.
* `git-clang-format`: used for the `clang_format` builtin hook.
* `gofmt`: used for the `gofmt` builtin hook.
* `google-java-format`: used for the `google_java_format` builtin hook.
* `google-java-format-diff`: used for the `google_java_format` builtin hook.
* `pylint`: used for the `pylint` builtin hook.
* `rustfmt`: used for the `rustfmt` builtin hook.
* `android-test-mapping-format`: used for the `android_test_mapping_format`
  builtin hook.

See [Placeholders](#Placeholders) for variables you can expand automatically.

```
[Tool Paths]
# Pass absolute paths.
clang-format = /usr/bin/clang-format
# Or paths relative to the top of the git project.
clang-format = prebuilts/bin/clang-format
# Or paths relative to the repo root.
clang-format = ${REPO_ROOT}/prebuilts/clang/host/${BUILD_OS}/clang-stable/bin/clang-format
```

# Hook Developers

These are notes for people updating the `pre-upload.py` hook itself:

* Don't worry about namespace collisions.  The `pre-upload.py` script is loaded
  and exec-ed in its own context.  The only entry-point that matters is `main`.
* New hooks can be added in `rh/hooks.py`.  Be sure to keep the list up-to-date
  with the documentation in this file.

## Warnings

If the return code of a hook is 77, then it is assumed to be a warning.  The
output will be printed to the terminal, but uploading will still be allowed
without a bypass being required.

# TODO/Limitations

* `pylint` should support per-directory pylintrc files.
* Some checkers operate on the files as they exist in the filesystem.  This is
  not easy to fix because the linters require not just the modified file but the
  entire repo in order to perform full checks.  e.g. `pylint` needs to know what
  other modules exist locally to verify their API.  We can support this case by
  doing a full checkout of the repo in a temp dir, but this can slow things down
  a lot.  Will need to consider a `PREUPLOAD.cfg` knob.
* We need to add `pylint` tool to the AOSP manifest and use that local copy
  instead of relying on the version that is in $PATH.
* Should make file extension filters configurable.  All hooks currently declare
  their own list of files like `.cc` and `.py` and `.xml`.
* Add more checkers.
  * `clang-check`: Runs static analyzers against code.
  * License checking (like require AOSP header).
  * Whitespace checking (trailing/tab mixing/etc...).
  * Long line checking.
  * Commit message checks (correct format/BUG/TEST/SOB tags/etc...).
  * Markdown (gitiles) validator.
  * Spell checker.
