# Packaging PL/Java for a software distribution

If you are responsible for creating or maintaining a PL/Java package
for a particular software distribution, thank you. PL/Java reaches a
larger community of potential users thanks to your efforts. To minimize
frustration for your users and yourself, please consider these notes
when building your package.

## What is the default `pljava.libjvm_location`?

Users of a PL/Java source build nearly always have to set the PostgreSQL
variable `pljava.libjvm_location` before the extension will work, because
there is too much variation in where Java gets installed across systems
for PL/Java to supply a useful default.

When you package for a particular platform, you may have the advantage of
knowing the conventional location for Java on that platform, and you can
improve the PL/Java setup experience for users of your package by adding
`-Dpljava.libjvmdefault=...` on the `mvn` command line when building,
where the `...` is the path to the JVM library shared object where it
would be by default on your target platform. See [here][locatejvm] to find
the exact file this should refer to.

[locatejvm]: ../install/locatejvm.html

Although PL/Java is currently built with a JDK no later than Java 8, it can
then run in the database with a newer JVM, and support application code that
uses the newer Java features. While using Java 8 to build a package, you are
encouraged to set the default `pljava.libjvm_location` to the library of a
later JRE that is expected to be present on your platform.

## What kind of a package is this?

Your package may be for a distribution that has formal guidelines for how
to package software in certain categories, such as "Java applications",
"Java libraries", or "PostgreSQL extensions". That may force a judgment
as to which of those categories PL/Java falls in.

### If possible: it's a PostgreSQL extension

PL/Java has the most in common with other PostgreSQL extensions (even though
it happens to involve Java). It has nearly nothing in common with "Java
applications" or "Java libraries" as those are commonly understood. It is
neither something that can run on its own as an application, nor a library
that would be placed on the classpath in the usual fashion for other Java code
to use. It is only usable within PostgreSQL under its own distinctive rules.

### Not recommended: Java application or library guidelines

Formal guidelines developed for packaging Java applications or libraries
are likely to impose requirements that have no value or are inappropriate
in PL/Java's case. The necessary locations for PL/Java's components are
determined by the rules of the PostgreSQL extension mechanism, not other
platform rules that may apply to conventional Java libraries, for example.

A packaging system's built-in treatment for Java libraries may even actively
break PL/Java. One packaging system apparently unpacks and repacks
jar files in a way that adds spurious entries. It has that "feature" to
address an obscure issue involving
[multilib conflicts for packages that use GCJ][repack], which doesn't apply
to PL/Java at all, and when the repacking silently added spurious entries
to PL/Java's self-installer jar, it took time to track down why unexpected
things were getting installed.

If you are using that packaging system, please be sure to follow the step
shown in that link to disable the repacking of jars.

[repack]: https://www.redhat.com/archives/fedora-devel-java-list/2008-September/msg00040.html

### An exception: `pljava-api`

The one part of PL/Java that could, if desired, be handled in the manner of
Java libraries is `pljava-api`. This single jar file is needed on the classpath
when compiling Java code that will be loaded into PL/Java in the database.
It is _not_ needed at the time that code will _run_. That means it could be
appropriate to treat `pljava-api` as a separate `-devel` package, if your
packaging guidelines encourage such a distinction. In that case, you would
exclude the `pljava-api` jar file from the main package, and produce a `-devel`
package that provides it.

A `-devel` package providing `pljava-api` might appropriately follow
java library packaging guidelines to ensure it appears on a developer's
classpath when compiling code to run in PL/Java. Ideally, such a package
would also place the `pljava-api` artifact into the local Maven repository,
if any. (PL/Java's [hello world example](../use/hello.html) illustrates using
Maven to build code for use in PL/Java, which assumes the local Maven repo
contains `pljava-api`.)

To build `pljava-api` in isolation. simply change into the `pljava-api`
subdirectory and run `mvn clean install`. It builds quickly and independently
of the rest of the project, with fewer build dependencies than the project
as a whole.

That `mvn clean install` also puts the `pljava-api` artifact into the local
Maven repository on the build host. A `-devel` package will ideally put the
same artifact into the local Maven repository of the installation target.
(While the other subprojects in a PL/Java full build also create artifacts
in the build host's local Maven repository, they can be ignored; `pljava-api`
is the useful one to have in an installation target host's repository.)

### PL/Java API javadocs

The PL/Java build does not automatically build javadocs. Those that go with
`pljava-api` can be easily generated by changing into the `pljava-api`
subdirectory and running `mvn javadoc:javadoc` to build them, then collecting
the `apidocs` subtree from `target/site`. They can be included in the same
package as `pljava-api` or in a separate javadoc package, as your guidelines
may require.

### An `examples` package?

A full PL/Java build also builds `pljava-examples`, which typically will also
be installed into PostgreSQL's _SHAREDIR_`/pljava` directory. If the packaging
guidelines encourage placing examples into a separate package, this jar file
can be excluded from the main package and delivered in a separate one.
The examples can be built in isolation by changing into the `pljava-examples`
subdirectory and running `mvn clean package`, as long as the `pljava-api` has
been built first and installed into the build host's local Maven repository.

Note that many of the examples do double duty as tests, as described in
_confirming the build_ below.

## Scripting the build

Options on the `mvn` command line may be useful in the scripted build for
the package.

`-Dpljava.libjvmdefault=`_path/to/jvm-shared-object_
: As suggested earlier, please use this option to build a useful default
into PL/Java for the `pljava.libjvm_location` PostgreSQL variable, users
of your package will not need to set that variable before
`CREATE EXTENSION pljava` works.

`-Dpgsql.pgconfig=`_path/to/pg\_config_
: If the build host may have more than one PostgreSQL version installed,
a package specific to one version can be built by using this option to point
to the `pg_config` command in the `bin` directory of the needed PostgreSQL
version. (The same effect was always possible by making sure that `bin`
directory was at the front of the `PATH` when invoking `mvn`, but this option
on the `mvn` command makes it more explicit.)

`-Pwnosign`
: If `gcc` is used in the build, adding this option on the `mvn` command line
will suppress a large number of nuisance warnings `gcc` would otherwise
produce, keeping the build log manageable in case any actual problem needs to
be found. (The nuisance warnings can be traced upstream to PostgreSQL headers,
and have been [deemed not pressing to fix there][notgoingthere], as upstream
also builds with those warnings suppressed.)

`-Dnar.cores=1`
: This option also helps to make the build log more intelligible in case any
problem needs to be found, by avoiding the interleaving of messages from
simultaneous compilations. PL/Java builds quickly enough that there is little
cost in wall time to do the compiles on one core.

[notgoingthere]: https://www.postgresql.org/message-id/19039.1496353285%40sss.pgh.pa.us

## Patching PL/Java

If your packaging project requires patches to PL/Java, and not simply the
passing of options at build or packaging time as described on this page,
please [open an issue][issue] so that the possibility of addressing your
need without patching can be discussed.

[issue]: https://github.com/tada/pljava/issues

## Confirming the build

A full build also produces a `pljava-examples` jar, containing many examples
that double as tests. Many of these are run from the deployment descriptor
if the PL/Java extension is created in a PostgreSQL instance and then the
examples jar is loaded with `install_jar` passing `deploy => true`, which
should complete with no warnings.

Some tests involving Unicode are skipped if the `server_encoding` is not
`utf-8`, so it is best to run them in a server instance created with that
encoding.

## Packaging the built items

The end product of a full PL/Java source build is a jar file that functions as
a self-extracting installer when run by `java -jar`. It contains the files that
are necessary on a target system to use PL/Java with PostgreSQL, including
those needed to support `ALTER EXTENSION UPGRADE`.

It also contains the `pljava-api` jar, needed for developing Java code to use
in a database with PL/Java, and the `pljava-examples` jar. As discussed above,
these may be omitted from a base package and supplied separately, if packaging
guidelines require.

The self-extracting jar consults `pg_config` at the time of extraction to
determine where the files should be installed.

Given this jar as the result of the build, there are three broad approaches
to constructing a package:

| Approach | Pro | Con |
----|----|----|
Capture self-extracting jar in package, deliver to target system and run it as a post-install action | Simple, closest to a vanilla PL/Java build. | May not integrate well into package manager for querying, uninstalling, or verifying installed files; probably leaves the self-installing jar on the target system, where it serves no further purpose. |
Run self-extracting jar at packaging time, and package the files it installs | Still simple, captures the knowledge embedded in the installer jar; integrates better with package managers needing the list of files installed. | Slightly less space-efficient? |
Ignore the self-extracting jar and hardcode a list of the individual files resulting from the build to be captured in the package | ? | Brittle, must reverse-engineer what pljava-packaging and installer jar are doing, from release to release. Possible to miss things. |

The sweet spot seems to be the middle approach.

When running the self-extractor, its output can be captured for a list of the
files installed. (As always, parsing that output can get complicated if the
pathnames have newlines or other tricky characters. The names of PL/Java-related
files in the jar do not, so there is no problem as long as no tricky characters
are in the PostgreSQL installation directory names reported by `pg_config`.)

A package specific to a PostgreSQL version can pass
`-Dpgconfig=`_path/to/pg\_config_ to Java when running the self-extractor,
to ensure the locations are obtained from the desired version's `pg_config`.
(This is the extraction-time analog of the `-Dpgsql.pgconfig` that can be
passed to `mvn` at build time.)

If necessary to satisfy some packaging guideline, individual locations
obtained from `pg_config` can be overridden with more specific options
such as `-Dpgconfig.sharedir=...` as described in the [install][] guide.
Or, the packaging script might simply move files, or edit the paths they
will have on the target system.

In addition to the files named in the self-extractor's output, additional
files could be included in the package (if guidelines require the README
or COPYRIGHT, for example). As discussed above, the `pljava-api` jar could
be filtered from the list if it will be delivered in a separate `-devel`
package, and the same could be done for `pljava-examples`.

[install]: ../install/install.html

## Late-breaking packaging news and tips

A [Packaging Tips][tips] page on the PL/Java wiki will be created for
information on packaging issues that may be reported and resolved between
released updates to this documentation. Please be sure to check there for
any packaging issue not covered here.

[tips]: https://github.com/tada/pljava/wiki/Packaging-tips
