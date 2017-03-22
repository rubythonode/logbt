logbt
-----

[![Build Status](https://travis-ci.org/mapbox/logbt.svg?branch=master)](https://travis-ci.org/mapbox/logbt)

Short for "Log Backtrace", this is a bash wrapper for displaying a backtrace when a program crashes.

The `logbt` command can also:

  - Respond to the `USR1` signal and generate a backtrace of a healthy program (which will continue to run)
  - Act as init process (aka "PID1") for docker containers (receives signals and reaps child processes)
  - Automatically clean up coredumps on the system (to avoid your disk filling up)
  - Work on sudo-enabled travis-ci.org machines

### Upgrading

If upgrading from a previous logbt version see [Upgrading.md](UPGRADING.md) for details on how to adapt your code.

### Supports

 - Linux and OS X
 - Docker (see [Docker](#Docker-considerations) below)

### Depends

The [`logbt` run command](#run-logbt) requires `gdb` on linux and `lldb` on OS X.

Recommended install of gdb on linux:

```
mkdir mason && curl -sSfL https://github.com/mapbox/mason/archive/v0.6.0.tar.gz | tar --gunzip --extract --strip-components=1 --directory=./mason
./mason/mason install gdb 7.12
export PATH=$(./mason/mason prefix gdb 7.12)/bin:${PATH}
which gdb
```

Recommended install of lldb on OS X is to get latest XCode.

### Installing

To install `logbt` to `/usr/local/bin`:

```sh
curl -sSfL https://github.com/mapbox/logbt/archive/v2.0.1.tar.gz | tar --gunzip --extract --strip-components=1 --exclude="*md" --exclude="test*" --directory=/usr/local
which logbt
/usr/local/bin/logbt
logbt --version
```

Locally (perhaps if your user cannot write to `/usr/local`):

```sh
curl -sSfL https://github.com/mapbox/logbt/archive/v2.0.1.tar.gz | tar --gunzip --extract --strip-components=2 --exclude="*md" --exclude="test*" --directory=.
./logbt --version
```

### Usage

There are two main modes to using `logbt`. First you run `logbt --setup` and second you run `logbt -- <your program>` to launch your program with it.

#### Setup logbt

```bash
sudo logbt --setup
```

This command sets the system `core_pattern` to ensure it is ready for `logbt` to use.

This is required on Linux (modifies `/proc/sys/kernel/core_pattern`). Running `logbt --setup` is optional on OS X if the system default for `kern.corefile` is intact (This means on OS X that `$(sysctl -n kern.corefile) == '/cores/core.%P'`)

Common default values for `core_pattern` on linux (which do not work with `logbt`) are:

  - `|/usr/libexec/abrt-hook-ccpp %s %c %p %u %g %t e` Seen on Centos 6
  - `|/usr/share/apport/apport %p %s %c` Seen on Ubuntu Precise
  - `|/usr/share/apport/apport %p %s %c %P` Seen on Ubuntu Trusty

#### Run logbt

All commands passed to `logbt` after `--` are interpreted as the program to run and any arguments to pass to that program.

This is known as the `run` command.

Therefore, to launch your program with `logbt` run:

```bash
logbt -- <your program> <your program args>
````

Then logbt will run as long as your program runs. If `logbt` your program will be killed with `SIGTERM`. If your program exits then `logbt` will exit with the same exit code. If your program crashes then `logbt` will display a backtrace and exit with the crashing exit code.

#### Additional options

 - `logbt --test`: tests that `logbt` is functioning correctly. Should be run after `logbt --setup`
 - `logbt --current-pattern`: displays the current `core_pattern` value on the system (`/proc/sys/kernel/core_pattern` on linux and `sysctl -n kern.corefile` on OS X)
 - `logbt --target-pattern`: displays the target `core_pattern` value that `logbt --setup` will apply to the system which is `/tmp/logbt-coredumps/core.%p.%e` on linux and `/tmp/logbt-coredumps/core.%P` on OS X)
 - `logbt --version`: Prints the `logbt` version
 - `logbt --help`: Prints the `logbt` usage help

### Snapshotting

A experimental feature of `logbt` >= 2.x is the ability to send a `USR1` signal and to generate backtrace of the healthy child program.

```bash
kill -USR1 <pid of logbt>
```

Or, if running `logbt` in a docker container, you can send this via the host like:

```
# see ./test/docker-snapshotting.sh for a full example
docker kill --signal="SIGUSR1" <container id>
```

When `USR1` is received by `logbt` the child program is paused (`SIGSTOP`), a backtrace is generated, and then the child program is resumed (`SIGCONT`).

There are several limitations to consider before using this feature. Future `logbt` versions will likely change this interface to use [bcc tools](https://iovisor.github.io/bcc/) for snapshotting due to limitations 1/2 below.

1) Not recommended for production

This is useful for checking on what the child program is doing for debugging, but should not be used in production systems because the child program is stopped for potentially >= several seconds.

2) ptrace support

`ptrace` support is needed to allow `gdb` on linux to attach to the child process. To enable `ptrace` in a docker container you must run with `--cap-add SYS_PTRACE` or `--privileged`. And `ptrace_scope` scope in the kernel likely will need to be set to zero like: `sudo bash -c "echo 0 > /proc/sys/kernel/yama/ptrace_scope"`. Another final limitation of ptrace is that only one tool may be attached at one time.

3) child limitations

The `child` must be the program you want snapshotted. While `logbt` supports tracking crashes of any children or grandchildren of the program run by `logbt` the snapshotting will only be done on the direct child.


### Docker considerations

Docker linux containers inherit their kernel settings from the linux host. Because the `core_pattern` modified by [`logbt --setup`](#Logbt-setup) is kernel-level the `--setup` command must be either be run as root on the linux host (recommended) or within a container run with the `--privileged` flag.

If you try to run `logbt --setup` in a container without the `--privileged` flag you will see an error like: `/proc/sys/kernel/core_pattern: Read-only file system`

:warning: Running `logbt --setup` in a `privileged` container will change the `core_pattern` value for the host. On OS X (with docker for mac) the `core_pattern` value will also be changed in the underlying linux host run by the hypervisor. You can see this by logging into the linux vm with `screen` by doing `screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty` and then `cat /proc/sys/kernel/core_pattern`. (`ctrl a \` to exit). By default it will be `cores` and after running a docker container that runs `logbt --setup` with `docker run --privileged` it will be equal to the `logbt` internal value for linux of `/tmp/logbt-coredumps/core.%p.%E`. This will be inherited for all other docker containers you run on OS X.

The `--privileged` only applies to `docker run` and not `docker build` (refs https://github.com/docker/docker/issues/1916)

With AWS, the ECS [container definition](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definition_security) is how you ask for `privileged` runs.

The `logbt --setup` command does not work on some CI systems like https://circleci.com (neither on the host or in the container). This is because on circleci `/proc/sys/kernel/core_pattern` is read-only on the host (perhaps because the host machine is actually a docker container). It appears that to run in `--privileged` you need to be an [enterprise customer](https://github.com/circleci/image-builder/#ubuntu-1404-xxl-enterprise).

Running `logbt` on travis works great with ["sudo-enabled" machines](https://docs.travis-ci.com/user/ci-environment/#Virtualization-environments). These machines allow you to run `sudo logbt --setup` on the host or run `logbt --setup` in a `privileged` container. However similar to circleci the `logbt --setup` command does not work on https://travis-ci.org "container-based" (aka `sudo:false`) machines.

One other alternative to running `--privileged` is mounting a writable /proc directory like:

    docker run --volume /proc:/writable-proc <image name> bash

Then within that container you can write to `writable-proc` and it will be reflected in `/proc`:

    cat $(./bin/logbt --target-pattern) > /writable-proc/sys/kernel/core_pattern

After that command both `/writable-proc/sys/kernel/core_pattern` and `/proc/sys/kernel/core_pattern` will be equivalent and `logbt --test` should work.

But surprisingly this still modifies the host `core_pattern` so there is no major advantage to this method over running `logbt --setup` directly on the host.


### FAQ

#### Q: core file may not match

I'm seeing a warning in the backtrace that says "core file may not match specified executable file". Why is that happening and is it a problem?

**Answer:**

This is normal and harmless if the program you launched with `logbt` has customized its process "title". For example, with `node` you can do:

```js
process.title = 'custom-name';
```

When `gdb` prints `core file may not match specified executable file` it is saying that it noticed your modification of the process title.

#### Q: non-tracked corefiles

I'm seeing a message from logbt like `[logbt] No corefile found at /tmp/logbt-coredumps/core.641.*`. Is that a problem or indication that backtraces are not working?

**Answer:**

If you also see a message following it like:

```
[logbt] Found corefile (non-tracked) at /tmp/logbt-coredumps/core.642.!root!.nvm!versions!node!v4.7.2!bin!node
[logbt] Processing cores...
```

Then everything is okay. What is happening is that `logbt` uses the `<pid>` (process id) of the program it launches to look for a corefile when that program crashes. Let's call that program the `parent` process. In this case the `parent` did not crash (pid of 641) but a program it launched (a child) crashed (pid of 642). Because the parent correctly reported the crash to `logbt` (via returning an exit code indicating a crash) then `logbt` knows to look harder for corefiles that may have been created from a crashing child. In this case `logbt` prints a message that it found a `(non-tracked)` corefile to indicate a child crashed rather than a parent.
