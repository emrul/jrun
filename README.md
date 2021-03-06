# jrun: painless Java component execution

## Summary

[Maven](https://maven.apache.org/) is amazing. It manages dependencies so that
Java projects become reusable "building blocks" in a much more robust way than
many other languages offer. And the
[Maven Central repository](https://search.maven.org/) contains a tremendous
wealth of code, ripe for reuse in your own projects.

But shockingly, Maven provides no easy way to actually __launch code__ from the
beautifully managed dependencies stored so lovingly into `~/.m2/repository`.

This project fills that gap: `jrun` launches Java code. You do not need to
download or install any JARs; you just specify an "endpoint" consisting of a
[Maven artifact](http://stackoverflow.com/a/2487511/1207769) identifier, plus
a main class if needed/desired, and `jrun` uses Maven to obtain and run it.

## Installation

Just clone this repo, and symlink `jrun` into your favorite `bin` directory.

The script uses some common utilities (e.g., `cat`) as well as `mvn` and `java`
for the heavy lifting. If you are missing anything, the script will tell you.

## Usage

```
Usage: jrun [-v] [-u] [-U] <jvm-args> <endpoint> <main-args>

  -v          : verbose mode flag
  -u          : update/regenerate cached environment
  -U          : force update from remote Maven repositories (implies -u)
  <jvm-args>  : any list of arguments to the JVM
  <endpoint>  : the artifact(s) + main class to execute
  <main-args> : any list of arguments to the main class

The endpoint should have one of the following formats:

- groupId:artifactId
- groupId:artifactId:version
- groupId:artifactId:mainClass
- groupId:artifactId:version:mainClass
- groupId:artifactId:version:classifier:mainClass

If version is omitted, then RELEASE is used.
If mainClass is omitted, it is auto-detected.
You can also write part of a class beginning with an @ sign,
and it will be auto-completed.

Multiple artifacts can be concatenated with pluses,
and all of them will be included on the classpath.
However, you should not specify multiple main classes.
```

### Examples

| Program                      | Command                                                                              |
|:----------------------------:|:------------------------------------------------------------------------------------:|
| Jython REPL                  | `jrun org.python:jython-standalone`                                                  |
| JRuby eval                   | `echo "puts 'Hello Ruby'" \| jrun org.jruby:jruby-complete:@jruby.Main`              |
| Groovy REPL                  | `jrun org.codehaus.groovy:groovy-groovysh:@shell.Main+commons-cli:commons-cli:1.3.1` |
| SciJava REPL with JRuby      | `jrun org.scijava:scijava-common:@ScriptREPL+org.scijava:scripting-jruby`            |
| SciJava REPL with Jython     | `jrun org.scijava:scijava-common:@ScriptREPL+org.scijava:scripting-jython`           |
| SciJava REPL with Groovy     | `jrun org.scijava:scijava-common:@ScriptREPL+org.scijava:scripting-groovy`           |
| SciJava REPL with Clojure    | `jrun org.scijava:scijava-common:@ScriptREPL+org.scijava:scripting-clojure`          |
| SciJava REPL with JavaScript | `jrun org.scijava:scijava-common:@ScriptREPL+org.scijava:scripting-javascript`       |

Note the usage of the `+` syntax as needed to append elements to the classpath.

### FAQ

* __Is it fast?__
  Endpoints are synthesized in a local cache under `~/.jrun`.
  So invoking the same endpoint a second time is really quick.
* __What does "no installation" mean?__
  Classpath elements are [hard-linked](https://en.wikipedia.org/wiki/Hard_link)
  into `~/.jrun` from `~/.m2/repository` rather than copied, so the `~/.jrun`
  folder has a tiny footprint even if you execute lots of different endpoints.
* __What if an endpoint has a new version?__
  Pass the `-U` flag to `jrun` to rebuild the endpoint.
  Note that unlike `mvn`, though, `jrun` does not check for updates otherwise.
* __Does it work on Windows?__
  It works with [Cygwin](https://www.cygwin.com/). It is not yet tested with
  [MinGW](http://www.mingw.org/) (included with
  [Git for Windows](https://git-for-windows.github.io/)) or Microsoft's
  [Windows Subsystem for Linux](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide).
  Reports of success or failure here are welcome; see
  [issue #7](https://github.com/ctrueden/jrun/issues/7).

### Configuration

You can configure the behavior of `jrun` using the `$HOME/.jrunrc` file.

#### Repositories

You can define additional remote Maven repositories,
from which artifacts will be retrieved. E.g.:

```ini
[repositories]
imagej.public = https://maven.imagej.net/content/groups/public
```

If you need more control over where artifacts come from—for example, if you
want to use your own remote Maven repository as a mirror of Maven Central—you
can do it using Maven's usual `~/.m2/settings.xml`; see [Using Mirrors for
Repositories](https://maven.apache.org/guides/mini/guide-mirror-settings.html).

#### Shortcuts

You can define shortcuts for launching commonly used programs:

```ini
[shortcuts]
repl = imagej:org.scijava.script.ScriptREPL
imagej = net.imagej:imagej
fiji = sc.fiji:fiji:LATEST
scifio = io.scif:scifio-cli
```

Shortcuts are substituted verbatim from the beginning of the endpoint,
single-pass in the order they are defined. So e.g. now you can run:
```
jrun repl
```
Note that with the `repl` shortcut above, the main class
(`org.scijava.script.ScriptREPL`) comes from a _different_ artifact than
the toplevel artifact (`net.imagej:imagej`). This is intentional, so that
all of [ImageJ](https://imagej.net/), including all of the various SciJava
`scripting-<foo>` plugins, is included in the classpath of the REPL.

#### Settings

There are a few configurable settings:

```ini
[settings]
m2Repo = /path/to/.m2Repo (default ~/.m2/repository)
cacheDir = /path/to/.jrun (default ~/.jrun)
links = soft (options: hard, soft, none; default hard)
```

## Alternatives

There is [JPM4J](http://jpm4j.org/), but it did not work too well for me:

* It wants to maintain its _own_ local repository of JARs outside of Maven—why?
  Everyone should use Maven repositories, a thoroughly established standard.

* For each artifact, you have to choose a single main class as its sole command
  which gets linked into a shell command that runs it.

* It does not seem well-synced with Maven Central, and/or does not
  seem to deal with dependencies in the expected way; e.g.:

    ```
    $ jpm install -l -f -m org.scijava.script.ScriptREPL org.scijava:scijava-common
    Errors
      0. Target specifies Class-Path in JAR but the indicated file .../repo/scijava-expression-parser-3.0.0.jar is not found
      1. Target specifies Class-Path in JAR but the indicated file .../repo/gentyref-1.1.0.jar is not found
      2. Target specifies Class-Path in JAR but the indicated file .../repo/eventbus-1.4.jar is not found
    ```

* The source is, oddly, part of [bnd](https://github.com/bndtools/bnd)
  rather than in the [jpm4j organization](https://github.com/jpm4j) anywhere,
  which is not a good sign, modularity-wise.
