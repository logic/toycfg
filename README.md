# toycfg

`toycfg` is a simple configuration management system. It is completely
inappropriate for any sort of production use, and was written on a lark;
someone suggested to me that writing a bare-bones configuration management
system would make a good take-home interview question, and I decided to test
that idea.

It turns out, yeah, that's actually a pretty interesting problem. But don't
use this for anything, it's mostly just here so I can point at it and say
"see! this is terrible! stop writing terrible shell scripts to manage your
infrastructure!"

`toycfg` has packages, files, services, and modules (meta-collections of
packages, files, and services) as first-class resources, and aside from
services, each resource has an associated pre- and post-hook that can be
executed prior to and after work on the resource has been completed.

There is no state carried between instantiation of resources; that is to say,
if you're wanting to do someting like puppet's "notify" to let a service know
that it needs to restart, well, you should probably use puppet. Remember what
I said about not actually using this?

If you use this on anything but Ubuntu, at a minimum you'll need to change the
`apt-get` invocations into something more appropriate for your distribution,
and the logic may have to change a bit.

The `example` directory shows a very basic example that you can use with, say,
a docker container with a recent ubuntu image. Something like this worked for
me on a recent Fedora release:

    sudo docker run -v `pwd`/example:/etc/toycfg:ro,Z \
      -v `pwd`/toycfg:/usr/bin/toycfg:ro,Z -i -t --rm ubuntu:latest

And then just run `toycfg base` and watch the... er... "magic" happen; once
it's finished, `curl -v http://localhost/` should tell you hello. Run `toycfg`
a second time, and you'll note that no state changes should be logged. You'll
also note that executions still happen, because there is no gating logic on
those; make sure your pre and post executables are idempotent!

`toycfg` is free software; please see the file `COPYING` for licensing details.

## Usage

`toycfg` takes a single command line argument, the name of the module to
execute.

`toycfg`'s advanced behavior is configured via environment variables:

* `TOYCFG_MODULES`
  * the path containing any modules; defaults to `/etc/toycfg`

`toycfg` will process packages first, then files, then services; the
assumption here is that you'll want your externally-sourced mutations
(packages) instantiated first, so you can overwrite any state that they
leave on disk, before finally starting up any services you need started.

## Module Configuration

### Packages

The `packages` directory contains individual files for each package to be
manipulated. The contents of the file are either "absent" (to remove the
package), "latest" (to ensure the latest version is installed), or a specific
text string indicating the version.

The `packages` directory contains a directory for each package that needs to
be manipulated on the system. This directory must contain a `version` file,
which contains either "absent" (to remove the package), "latest" (to install
the latest version of the package), or a text string indicating a specific
version of the package to install.

If the directory contains an executable file named `pre`, it is executed as a
script before any changes are applied. A non-zero exit code will cause the
resource to be skipped.

If the directory contains an executable file named `post`, it is executed after
any changes have been applied.

### Files

The `files` directory contains a filesystem tree rooted at `/`.

Every directory in the tree containing a file named `type` is treated as a
resource to be managed. If `type` is missing, the directory is considered a
placeholder, and is left completely unmanaged.

If `type` contains "file", then the full path to that resource is instantiated
as a plain file, with the content specified in the file `content`.

If `type` contains "template, the contents of the file (read from the file
`content`) are interpreted as a bash here-document/here-string, allowing for
the use of variable expansion and command substitution.

If `type` contains "link", a symlink is created at the full path to the
resource, linking to the file specified in the file `content`.

If `type` contains "directory", a directory is created at the full path to the
resource. If the directory contains a file named `purge`, any existing
contents of the directory are removed (aside from what is explicitly managed).

If `type` contains "absent", the file or directory is removed, if it exists.

If the directory contains a file named `owner`, the owner is set to the content
of that file. Default is the user the system runs as (root).

If the directory contains a file named `group`, the group is set to the content
of that file. Default is the user the system runs as (root).

If the directory contains a file named `mode`, the mode is set to the content
of that file. Default is determined by the umask of the running user.

If the directory contains an executable file named `pre`, it is executed as a
script before any changes are applied. A non-zero exit code will cause the
resource to be skipped.

If the directory contains an executable file named `post`, it is executed after
all changes have been applied.

### Services

The `services` directory contains individual files named for each service to
be managed. The contents of the file are either "running" (to ensure the
service is up and running) or "stopped" (to ensure the service is not running).

### Requires and Next

A file named `requires` contains a list of module names which will be executed
(in listed order) before the contents of this module are executed.

A file named `next` contains a list of module names which will be executed (in
listed order) after the contents of this module are executed.

Good advice would be to use one or the other throughout as a general pattern,
but not both; while the main host module may need both, for simplicity's
sake, most modules should only use `requires`.

### Pre/Post

A file named `pre`, if it exists, will be executed before anything else in
the module is executed. A non-zero exit code will cause the rest of the module
to be skipped.

A file named `post`, if it exists, will be executed after all resources for
the module (including sub-modules) have been executed.
