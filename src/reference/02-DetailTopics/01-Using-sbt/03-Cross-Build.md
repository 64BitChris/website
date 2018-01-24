---
out: Cross-Build.html
---

  [Command-Line-Reference]: Command-Line-Reference.html
  [Publishing]: Publishing.html
  [Cross-Build-Plugins]: Cross-Build-Plugins.html

Cross-building
--------------

### Introduction

Different versions of Scala can be binary incompatible, despite
maintaining source compatibility. This page describes how to use `sbt`
to build and publish your project against multiple versions of Scala and
how to use libraries that have done the same.

### Publishing Conventions

The underlying mechanism used to indicate which version of Scala a
library was compiled against is to append `_<scala-version>` to the
library's name. For Scala 2.10.0 and later, the binary version is used.
For example, `dispatch-core_2.10` when compiled against
2.10.0, 2.10.1 or any 2.10.x version. This fairly simple approach
allows interoperability with users of Maven, Ant and other build tools.

The rest of this page describes how `sbt` handles this for you as part
of cross-building.

### Using Cross-Built Libraries

To use a library built against multiple versions of Scala, double the
first `%` in an inline dependency to be `%%`. This tells `sbt` that it
should append the current version of Scala being used to build the
library to the dependency's name. For example:

```scala
libraryDependencies += "net.databinder.dispatch" %% "dispatch-core" % "0.13.3"
```
A nearly equivalent, manual alternative for a fixed version of Scala is:

```scala
libraryDependencies += "net.databinder.dispatch" % "dispatch-core_2.12" % "0.13.3"
```

### Cross-Building a Project

Define the versions of Scala to build against in the
`crossScalaVersions` setting. Versions of Scala 2.10.2 or later are
allowed. For example, in a `.sbt` build definition:

```scala
crossScalaVersions := Seq("2.11.11", "2.12.2")
```

To build against all versions listed in `crossScalaVersions`, prefix
the action to run with `+`. For example:

```
> + package
```

A typical way to use this feature is to do development on a single Scala
version (no `+` prefix) and then cross-build (using `+`) occasionally
and when releasing.

You can use `++ <version>` to temporarily switch the Scala version currently
being used to build.
For example:

```
> ++ 2.12.2
[info] Setting version to 2.12.2
> ++ 2.11.11
[info] Setting version to 2.11.11
> compile
```
`<version>` should be either a version for Scala published to a repository or
the path to a Scala home directory, as in `++ /path/to/scala/home`.
See [Command Line Reference][Command-Line-Reference] for details.

The ultimate purpose of `+` is to cross-publish your
project. That is, by doing:

```
> + publish
```

you make your project available to users for different versions of
Scala. See [Publishing][Publishing] for more details on publishing your project.

In order to make this process as quick as possible, different output and
managed dependency directories are used for different versions of Scala.
For example, when building against Scala 2.12.4,

-   `./target/` becomes `./target/scala_2.12/`
-   `./lib_managed/` becomes `./lib_managed/scala_2.12/`

Packaged jars, wars, and other artifacts have `_<scala-version>`
appended to the normal artifact ID as mentioned in the Publishing
Conventions section above.

This means that the outputs of each build against each version of Scala
are independent of the others. `sbt` will resolve your dependencies for
each version separately. This way, for example, you get the version of
Dispatch compiled against 2.11 for your 2.11.x build, the version
compiled against 2.12 for your 2.12.x builds, and so on. You can have
fine-grained control over the behavior for different Scala versions
by using the `cross` method on `ModuleID` These are equivalent:

```scala
"a" % "b" % "1.0"
"a" % "b" % "1.0" cross CrossVersion.Disabled
```

These are equivalent:

```scala
"a" %% "b" % "1.0"
"a" % "b" % "1.0" cross CrossVersion.binary
```

This overrides the defaults to always use the full Scala version instead
of the binary Scala version:

```scala
"a" % "b" % "1.0" cross CrossVersion.full
```

`CrossVersion.patch` sits between `CrossVersion.binary` and `CrossVersion.full`
in that it strips off any trailing `-bin-...` suffix which is used to
distinguish varaint but binary compatible Scala toolchain builds.

```scala
"a" % "b" % "1.0" cross CrossVersion.patch
```

This uses a custom function to determine the Scala version to use based
on the binary Scala version:

```scala
"a" % "b" % "1.0" cross CrossVersion.binaryMapped {
  case "2.9.1" => "2.9.0" // remember that pre-2.10, binary=full
  case "2.10" => "2.10.0" // useful if a%b was released with the old style
  case x => x
}
```

This uses a custom function to determine the Scala version to use based
on the full Scala version:

```scala
"a" % "b" % "1.0" cross CrossVersion.fullMapped {
  case "2.9.1" => "2.9.0"
  case x => x
}
```

A custom function is mainly used when cross-building and a dependency
isn't available for all Scala versions or it uses a different convention
than the default.
