# rrapper

## 1.0 Overview

This repository contains several scripts that work in complement with the [modified rr](https://github.com/pkmoore/rr), as part of the CrashSimulator project.
It comprises of several important components that make work together to ensure successful record-replay and injection.

## 2.0 Requirements

* Linux-based Operating System using a ≥ 3.11.x kernel

```
$ uname --kernel-release
4.15.0-23-generic
```

* Disable ASLR

```
$ echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

and set

```
$ echo kernel.randomize_va_space = 0 | sudo tee /etc/sysctl.d/01-disable-aslr.conf
```

* Disable ptrace

```
$ echo kernel.yama.ptrace_scope = 0 | sudo tee /etc/sysctl.d/10-ptrace.conf
```

* Python 2.7.x
* libpython2-dev
* zlib

## 3.0 Installation

### Automated Installation _(with Docker)_

We use Docker in order to deploy the entire CrashSimulator codebase into an isolated container. The
`--cap-add=SYS_PTRACE` and `--security-opt seccomp=unconfined` flags are required to allow the usage of
the `ptrace`, `perf_event_open`, and `process_vm_writev` system calls by rr within the container.

```
$ docker build -t crashsimulator .
$ docker run --cap-add=SYS_PTRACE --security-opt seccomp=unconfined -it crashsimulator
```

This starts an interactive session within the Ubuntu image in the container with all the necessary components installed.

### Automated Installation _(with Vagrant)_

As an alternative, Vagrant can provide virtualization and isolation through libvirt while also automating the build process. This is good for macOS users, who want to bootstrap and deploy CrashSimulator as soon as possible.

```
$ vagrant plugin install vagrant-libvirt
$ vagrant up --provider=libvirt
$ vagrant ssh
```

### Manual Installation

After cloning this repository a few dependencies must be put in place:

First, initialize a `virtualenv`:

```
$ pip install virtualenv
$ virtualenv crashsim
$ source crashsim/bin/activate
```

Install CrashSimulator's dependencies

```
$ python setup.py install
$ pip install -r requirements.txt
```

## 4.0 Components

This repository comprises of several modules that work together sequentially in order to produce a trace,
and perform record-replay execution.

### 4.1 `rrinit`

`rrinit` executes initialization routines for the creation of a CrashSimulator
environment.

Before the user can create any tests and perform a record-replay
execution, the testing environment must be optimal for such use. This
application-level script checks for an optimal microarchitecture,
update necessary proc entries, creates a path for storing tests and
configurations, then generates the necessary system call definition file.

By running `rrinit`, a lot of the work can be taken out of `rrtest`
and `rreplay`, and users are able to catch environmental anomalies
before execution. This is also optimal when automating the creation
of images and containers.

```
$ rrinit    # ... is all you need!
```

### 4.2 `rrtest`

`rrtest` generates and bootstraps CrashSimulator-compliant tests.

This application-level script enables the user to perform a preemptive record and
replay execution, such that a strace segment is produced. This strace can then
be compared against a ReplaySession debug log in order to determine corresponding
events and trace lines, such that a final test can be produced and stored. Once
complete, a user can utilize `rreplay` in order to execute another replay execution.


### 4.3 `rreplay`

`rreplay` replays the recorded test generated by `rrtest` by attaching the
CrashSimulator injector within the `rr` process.

By doing this, the user can supply their own checkers and mutators that can alter the trace, modifying
the execution behavior of the trace. By looking at how the `rr`-replayed process is responding to the
introduced anomaly, the user is able to determine whether or not the recorded application needs to
be optimized.

## 5.0 Usage

The use of this codebase is to record traces of program executions using `rr` and having the ability to
communicate these events and inject handles during replay execution.

Automatic trace recording is provided by the `rrtest` module.

```
# first, you create a new test
$ rrtest create --name mytest --comand "./myapplication"
# ... from the generated strace output, select the line of system call
# you are interested in (i.e 100).

# then, configure it accordingly
$ rrtest configure --name mytest --traceline 100
# ... from there, you have the ability now to select the proper event
# that corresponds with that strace line.

# complete! Now I can replay.
$ rreplay mytest
```

### 5.1 Tips for Configuration

When complete the test (stored within `~/.crashsim/mytest/` will have a `config.ini` that is recognizable by `rreplay`. The comments in the below code snippet describe required elements.

```
# Top level section specifies global stuff. Right now, the only element in use
# defines the directory containing the rr recording to be used.
[rr_recording]
rr_dir = test/flask-1/
# Each subsequent section describes an rr event during which we want to spin off
# processes and attach an injector.  Each element below is required to tie all
# these pieces together correctly.
[request_handling_process]
# The rr event during which to perform spin off and injection
event = 16154
# The pid amongst the group spun off to which we want to attach the injector.
# This is the pid of the process as recoreded by rr.  The actual pid the spun
# off processes gets will differ.  Mapping between these two is handled
# automatically by rrapper.py
pid = 14350
# The file that contains the strace segment that describes application behavior
# AFTER it has been spun off
trace_file = test/flask_request_snip.strace
# Defines the line that the injector should start with when it takes over for rr
# and begins its work
trace_start = 0
# Defines the last line that the injector should be concerned with.  Once this
# strace line has been handled by the injector, the injector considers its work
# done and the processes associated with this injection are killed cleaned up.
trace_end = 13
```

Here are some tips to remember if you choose to go back and manually configure the test.

__1. Identifying the interesting rr event__

```
RR_LOG=ReplaySession rr replay -a (optional_trace_name)
```

Setting the RR_LOG environment variable allows you to enable and disable logging
for different parts of rr.  Conveniently, logging only ReplaySession outputs a
listing of system calls as they are handled.  You can use this output to pick
out the system call you are interested (its associated event is listed).


__2. Generating an appropriate trace segment__

```
strace -f -s 65535 -vvvvv -o <filename> <command>
```

For now, the fastest way to get a strace segment to use in driving the injector
is to re-record the application with `strace`.  Strace segments must have certain
information in order to be used by the system.  The above flags configure strace
to output the current process's pid (`-f`), don't limit the length of strings (`-s
65535`) and be as verbose as possible with structures (`-vvvvv`).  This can result
in huge traces so you should chop out any lines that are not relevant to your
replay efforts.  Because the injector's handling of a process can be rough in
places, you should ask it to handle as small of a segment as is required to
exercise your test case.

### 5.2 Checkers and Mutators

#### Checkers

CrashSimulator supports user supplied checkers implemented in Python.  These
checkers consist of Python classes that support a specific set of methods as
described below.  Checkers receive the sequence of system calls as they are
handled by CrashSimulator.  The checker is responsible for examining these system
calls and making a decision as to whether the application took a specific action
or not.  Built in checkers use a form of state machine to make this decision but
users are free to track state using whatever model they prefer.

```
init(self, <additional parameters>)
```

The checker class's `init` method can be used to configure things like file names,
file descriptors, or parameter values to watch for.  The values for these
parameters are passed in when the checker is configured in a .ini file.


```
transition(self, syscall_object)
```

This method is called for every system call entrance and exit handled by
CrashSimulator.  The supplied `syscall_object` is a posix-omni-parser--like object
meaning things like parameters and return values can be examined.


```
is_accepting(self)
```

This method must return a truth-y for false-y value that is used to
CrashSimulator to report whether or not the checker accepted the handled system
call sequence.

#### Mutators

Mutators are also implemented as a python class that implements a specific set
of methods.  The mutator operates on the text content of the configured strace
segment.

```
init(self, <additional parameters>
```

The mutator's `init` method is used to supply values needed during the mutating
process. Values for the additional parameters are supplied through the `config.ini`
 file.

```
mutate_trace(self, trace)
```

__Note:  The below behavior is expected to change.  More of the boilerplate will
be handled by CrashSimulator__

This call is made prior to the CrashSimulator kicking off the system call
handling process.  trace the filename of the strace segment that will be used
during system call handling.  During this method call the mutator is expected to
open, mutate, and write the mutated trace out to a file.  This method must
return the filename of the mutated trace.  CrashSimulator will use this filename
to drive testing rather than the unmodified file.
