<!--

@license Apache-2.0

Copyright (c) 2025 The Stdlib Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

-->

# Package Migration

> This document is intended to provide a step-by-step guide for moving a package from one part of the codebase to another part of the codebase. This guide also applies when renaming a package.

The following outlines a sequence of steps to follow in order to migrate a single strided API package which is in `stats/base` to `stats/strided`.

At a high level, the process is as follows:

1.  Copy the existing package to a new location.
2.  Update all import paths which point to the original package to the new copy.
3.  Remove the original package.

A more detailed sequence follows. It is **extremely important** to closely follow each step in order to avoid inadvertently breaking downstream packages within the project.

## Steps

### 0. Establish a clean working repository

You should avoid performing a migration on a repository branch which has unstaged changes. Your repository should be in a clean and pristine state prior to performing a package migration. Because you will often be making changes which are randomly spread out across the entire codebase, you want to avoid a scenario where you unintentionally commit changes unrelated to the migration.

Ensure that you create a new branch from the latest upstream `develop`. Assuming that your `upstream` is the main stdlib development repository (i.e., `https://github.com/stdlib-js/stdlib.git`),

```bash
git checkout develop
git pull upstream develop
git checkout -b <branch_name>
```

where `<branch_name>` is a placeholder for the name of a branch on which you'll perform the migration (e.g., `migrate-foo-bar`).

### 1. Copy package

Copy the existing package to the desired location. Assuming you are in the top-level root project directory,

```bash
cp -R lib/node_modules/@stdlib/path/to/existing/package lib/node_modules/@stdlib/path/to/new/package
```

For example,

```bash
cp -R lib/node_modules/@stdlib/stats/base/dmax lib/node_modules/@stdlib/stats/strided/dmax
```

### 2. Update package contents

Next, update the contents of the new package, which is likely to include the following:

-   The package name (`package.json`, `lib/index.js`, `README.md`, and any other files which include the name of the original package.

-   If a package contains `src` and `include` directories,

    -   update the directory tree according to the path of the new package (e.g., `include/stdlib/stats/base/dmax` would become `include/stdlib/stats/strided/dmax`).
    -   update header guards within header files in the `include` directory (e.g., `STDLIB_STATS_BASE_DMAX` would become `STDLIB_STATS_STRIDED_DMAX`).

There may be other contents needing updating, so be sure to carefully inspect package contents.

### 3. Compile any native files

If a package contains a `src/addon.c` file, ensure that the package successfully compiles by running the following command

```bash
make install-node-addons NODE_ADDONS_PATTERN="path/to/new/package"
```

For example,

```bash
make install-node-addons NODE_ADDONS_PATTERN="stats/strided/dmax"
```

### 4. Run package unit tests and quality control commands

To ensure that the new package works as intended, run the package's unit tests and other quality control commands.

#### Unit tests

```bash
make test TESTS_FILTER=".*/path/to/new/package/.*"
```

For example,

```bash
make test TESTS_FILTER=".*/stats/strided/dmax/.*"
```

#### Examples

```bash
make examples EXAMPLES_FILTER=".*/path/to/new/package/.*"
```

For example,

```bash
make examples EXAMPLES_FILTER=".*/stats/strided/dmax/.*"
```

If a package contains C examples, compile and run the C examples.

```bash
make examples-c EXAMPLES_FILTER=".*/path/to/new/package/.*"
```

For example,

```bash
make examples-c EXAMPLES_FILTER=".*/stats/strided/dmax/.*"
```

#### Benchmarks

```bash
make benchmark BENCHMARKS_FILTER=".*/path/to/new/package/.*"
```

For example,

```bash
make benchmark BENCHMARKS_FILTER=".*/stats/strided/dmax/.*"
```

If a package contains C benchmarks, compile and run the C benchmarks.

```bash
make benchmark-c BENCHMARKS_FILTER=".*/path/to/new/package/.*"
```

For example,

```bash
make benchmark-c BENCHMARKS_FILTER=".*/stats/strided/dmax/.*"
```

### 5. Commit new package

Provided all tests pass and source files successfully compile, commit the new package to your migration branch.

```bash
git add lib/node_modules/@stdlib/path/to/new/package && git commit
```

For example,

```bash
git add lib/node_modules/@stdlib/stats/strided/dmax && git commit
```

When writing the commit message, you should include the name of the package being added and, if there is a public issue related to this particular package migration, include a Git trailer which references that issue.

For example,

```text
feat: add `stats/strided/dmax`

Ref: https://github.com/stdlib-js/stdlib/issues/4797
```

### 6. Remove the export of the original package from its parent namespace

Next, open the `lib/index.js` file found in the parent namespace of the original package (e.g., `lib/node_modules/@stdlib/stats/base/lib/index.js`.

If that file includes an exported symbol from the original package, remove it. For example,

```diff
-
- /**
- * @name dmax
- * @memberof ns
- * @readonly
- * @type {Function}
- * @see {@link module:@stdlib/stats/base/dmax}
- */
- setReadOnly( ns, 'dmax', require( '@stdlib/stats/base/dmax' ) );
```

### 7. Commit the changes to the parent namespace

If you removed an exported symbol from the parent namespace, commit those changes, making note that this is a breaking change.

```bash
git add lib/node_modules/@stdlib/path/to/original/parent/namespace && git commit
```

For example

```bash
git add lib/node_modules/@stdlib/stats/base && git commit
```

In your commit message, you should include a `BREAKING_CHANGE`, migration instructions, and a reference to any related public issues. For example,

```text
remove: remove `dmax` from namespace

This commit removes the `dmax` symbol from the `stats/base`
namespace due to a package migration.

BREAKING CHANGE: remove `dmax`

To migrate, users should access the same symbol via the
`stats/strided` namespace.
```

### 8. Update paths using a global find-and-replace

Next, perform a global find-and-replace to migrate all `require` (and `import`) paths which currently reference the original package to reference the new package.

For example, the following `require` statement

```text
var dmax = require( 'stats/base/dmax' );
```

should become

```text
var dmax = require( 'stats/strided/dmax' );
```

A couple of very important notes to keep in mind when performing a global find-and-replace.

-   Be **very careful** to avoid erroneously updating the paths of packages whose names have a common prefix (e.g., `stats/base/dmaxabs`, `stats/base/dmaxsorted`, `stats/base/dmaxabssorted`). Those packages should **not** be inadvertently updated.
-   Additionally, ensure that, for packages having C implementations, if a package basename (e.g., `dmax`) has a hyphen, then downstream include paths also need to be updated. E.g., for package `stats/base/foo-bar` with include file `stats/base/foo_bar`, all downstream packages which include the previous header file need to be updated accordingly (e.g., `stats/strided/foo_bar`).

### 9. Avoid updating original package and error database

There are three packages where we do **not** want to update `require` paths.

-   The original package. The original package should remain working and keeps its original paths.
-   The global error database. The global error database is an append-only log. We need to avoid invalidating any existing references.
-   The REPL databases. Given the high velocity of stdlib development, updating these databases will create merge conflicts, which do not need to be immediately resolved. We can avoid the hassle of needing to rectify these conflicts by deferring to stdlib's daily cron job which automatically maintains and updates these databases.

To dismiss any changes made to the above, run the following command

```bash
git checkout -- ./lib/node_modules/@stdlib/path/to/original/package && git checkout -- ./lib/node_modules/@stdlib/error && git checkout -- ./lib/node_modules/@stdlib/repl && git status
```

For example,

```bash
git checkout -- ./lib/node_modules/@stdlib/stats/base/dmax && git checkout -- ./lib/node_modules/@stdlib/error && git checkout -- ./lib/node_modules/@stdlib/repl && git status
```

After running the above, double-check the results of `git status` to check that the list of changed files matches expectation.

### 10. Commit changes

Now that you've cleaned up the path updates, commit the changes to your branch.

```bash
git add . && git commit
```

As updating paths should not affect the behavior of downstream packages, your commit message should be a `refactor`, and you should include a reference to any public issue related to the migration. For example,

```text
refactor: update paths

Ref: https://github.com/stdlib-js/stdlib/issues/4797
```

### 11. Remove original package

At this point, now that all downstream packages use the new package, we should be able to remove the original package from the project.


```bash
rm -rf lib/node_modules/@stdlib/path/to/original/package
```

For example,


```bash
rm -rf lib/node_modules/@stdlib/stats/base/dmax
```

### 12. Commit changes

Commit the changes to your branch.

```bash
git add lib/node_modules/@stdlib/path/to/original/package && git commit
```

For example,

```bash
git add lib/node_modules/@stdlib/stats/base/dmax && git commit
```

In your commit message, you should include a `BREAKING_CHANGE`, migration instructions, and a reference to any related public issues. For example,

```text
remove: remove `stats/base/dmax`

This commit removes `@stdlib/stats/base/dmax` in favor of
`@stdlib/stats/strided/dmax`.

BREAKING CHANGE: remove `stats/base/dmax`

To migrate, users should update their require/import paths to use
`@stdlib/stats/strided/dmax` which provides the same API and implementation.

Ref: https://github.com/stdlib-js/stdlib/issues/4797
```

### 13. Push changes to remote repository

At this point, you've completed a package migration. You should attempt to push your changes to your remote repository.

```bash
git push origin <branch_name>
```

where `<branch_name>` is the name of the branch on which you've been working.

If you made these changes on a fork, you should open a pull request against the `develop` branch on the main project [repository][stdlib-github].

* * *

## Notes

-   A pull request should **only** migrate a single package. Please do **not** open pull request which attempts to migrate multiple packages at the same time.
-   Notice that every commit includes a `Ref:` link back to the RFC issue on the main project repository. This is useful for providing additional context regarding changes, especially those involving deprecations.
-   Provided you have properly setup your local repository (see the [contributing][stdlib-contributing] and [development][stdlib-development] guides), linting will be performed after every commit, and, prior to pushing changes to a remote repository, affected unit tests, examples, and benchmarks should automatically run. Depending on how widely used the original package was throughout stdlib, these quality control steps may take considerable time, and it is possible that unrelated lint errors may be flagged. If possible, address any failures, restage the changes, and attempt to commit or push again.
-   As mentioned above, be **very careful** when performing global find-and-replace. It can be easy to mistakenly update non-applicable paths, thus breaking packages and all downstream dependents. You've been warned.

* * *

## Checklist



* * *

## Reviewers

Typically, we prefer to squash commits to a single commit when merging pull requests in stdlib. However, when performing package migrations, we need to preserve the exact sequence of migration commits in order to ensure proper changelog generation and automatic package versioning. Accordingly, the procedure for reviewing and merging migration pull requests diverges from normal protocol and requires additional vigilance to ensure that a migration does not break downstream usage.

When reviewing a pull request involving a package migration, one should do the following:

1.  Verify that the pull request includes at most `4` and only `4` commits (i.e., `feat`, `remove`, `refactor`, and `remove`, in that order). It is possible that the namespace of the original package did not export an associated symbol; in which case, the first `remove` may not be present, and the sequence should be `feat`, `refactor`, and `remove`, in that order.

2.  Verify that all CI checks pass. If not, investigate and determine whether to block the pull request from being merged.

3.  Inspect each commit for the following:

    -   `feat`: should only add the new package. Ensure that all `require` paths have been updated and correctly refer to the new package. Ensure that any `include` directories have been renamed. Ensure that header guards have been updated.
    -   `remove`: should only remove the symbol from the original package.
    -   `refactor`: check as many files as possible, ensuring a wide cross-section of affected files, to ensure that the updated paths correctly point to the new package.
    -   `remove`: should only remove the original package.

    If there exists a public issue associated with the migration, ensure that each commit refers to that public issue with a `ref:` Git trailer.

    Ensure that each `remove` commit message body includes a `BREAKING CHANGE` section, along with migration steps.

4.  If the pull request has merge conflicts, inspect the conflicts and determine whether you can easily resolve. If the conflicts cannot be resolved, you'll likely need to block the pull request from being merged. If they can be resolve, go ahead and resolve them, creating a merge commit.

5.  If everything looks okay, approve the pull request changes.

6.  If the pull request includes a merge commit, do the following:

    -   Enable "Allow merge commits" (with the default message) on GitHub in repository settings.
    -   Refresh the pull request page.
    -   Select "Create a merge commit".
    -   In the commit body, add a `ref:` trailer which points to the PR URL and add a `reviewed-by:` trailer with your info.
    -   Merge.
    -   Disable "Allow merge commits" on GitHub in repository settings. Our default merge setting should be "Squash and merge", and we do not want to mistakenly perform merge commits in future PRs.
    -   Finished.

7.  Otherwise, do the following:

    -   Enable "Allow rebase merging" on GitHub in repository settings.
    -   Refresh the pull request page.
    -   Select "Rebase and merge".
    -   Confirm that you wish to perform the operation.
    -   Merge.
    -   Finished.

<section class="links">

[stdlib-github]: https://github.com/stdlib-js/stdlib

[stdlib-contributing]: https://github.com/stdlib-js/stdlib/blob/develop/CONTRIBUTING.md

[stdlib-development]: https://github.com/stdlib-js/stdlib/blob/develop/docs/contributing/development.md

</section>

<!-- /.links -->
