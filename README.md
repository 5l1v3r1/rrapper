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

* Allow access to kernel perf events

```
$ echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid


```

* Hardware Performance Counters

```
For VMWare Fusion:
Virtual Machine Menu -> Settings -> Processors and Memory -> Advanced Options
-> tick "Enable Code Profiling Applications in this Virtual Machine"

```

```
Other Setups:
You must configure your system or virtualization setup to allow access to hardware performance counters.
KVM allows this for Linux Hosts
Virtual Box DOES NOT support virtualized performance counters.
```

**Note that some of these settings can be reverted upon system restart.**

* An installed and working copy of our modified version of rr located [here](https://github.com/pkmoore/rr/tree/spin-off).
Note that you must install the ```spin-off``` branch from this repository __NOT__ the master branch.

**Very Important Note: You must pull and use the spin-off branch from the repository containing our modified version of rr.  The master branch tracks the unmodified version of rr provided by mozilla/rr**

Other dependencies that may need to be installed using your Linux distribution's package manager:

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

### Ensuring Your Installation Works

There are a few actions you can take to ensure you have installed the tool
correctly.

First, run ```rrinit``` to check your environment.  This will ensure the options
mentioned at the top of this file have been configured correctly.

Second, run ```nosetests``` to run the unit tests that ship with rrapper.


## 4.0 Usage
__This interface is under heavy development.  Actual output may differ from below__
Basic Usage:
```
crashsim <testname> --command="<command or executable>" --mutator="<mutator constructor>"
```
e.g.

```
crashsim testmyapp --command="myapp -h -t -c100" --mutator="UnusualFiletypeMutator(S_IFCHR)"
```
Expected output:
```
UnusualFiletypeMutator for event 140() completed without deltas
UnusualFiletypeMutator for event 146() completed without deltas
UnusualFiletypeMutator for event 151() completed without deltas
ReplayDeltaError open system call is not expected from trace (expected write)
```

The first three lines indicate the application did not recognize that we changed the file being
examined at the specific event from a regular file (S_IFREG) to a character device (S_IFCHR).

The 4th line indicates the application recognized that the file being examined was different than expecteed
and diverted off the previous recording we took.  For more information see the anomaly specific documentaton in the anomalies folder.


### 4.1 Tips for Configuration

When complete the test (stored within `~/.crashsim/mytest/` will have a `config.ini`.  This file can be modified for more advanced testing.  Updated configs can be used via ```rreplay <test name>``` The comments in the below code snippet describe required elements.

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


### 4.2 Checkers and Mutators

#### Mutators

Mutators are also implemented as a python class that implements a specific set
of methods.  The mutator operates on the text content of the configured strace
segment.

```
init(self, <additional parameters>
```

The mutator's `init` method is used to supply values needed during the mutating
process. Values for the additional parameters are supplied through the `config.ini`
file or on the command line via the --mutator switch.
 
```
identify_lines(self, syscalls)
```
This function receives syscalls, an iterator, which yeilds posix-omni-parser syscall objects for the entire recording.
Syscalls should be consumed using a for i in syscalls: pattern to examine each syscall as it is yeilded.  This function should identify which syscall object indexes are valid opportunities during which the mutators anomaly can be simulated.  This function should return a list of these indexes after it has examined all of the system call objects.

```
mutate_syscalls(self, syscalls)
```
This function receives syscalls, a list containing a subset of posix-omni-parser system call objects starting from an index identified by identify_lines() and ending some fixed number of system calls later (5 by default).  This function should modify the contents of these objects in such a way that the mutator's anomaly is present.  These modified system calls are used to influence an applications execution.

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

## 5.0 Old Interface

This repository comprises of several modules that work together sequentially in order to produce a trace,
and perform record-replay execution.

### 5.1 `rrinit`

`rrinit` makes an effort to configure your virtual machine with many of
the requirements mentioned above.  There is a possibility it will will
not be able to do so.

This command also checks for the presence of a supported microarchitecture,
creates a path for storing tests and configurations, 
then generates the necessary system call definition file.

```
$ rrinit    # ... is all you need!
```

### 5.2 `rrtest`

`rrtest` generates and bootstraps CrashSimulator-compliant tests.

This application-level script enables the user to perform an initial recording
in order to produce the rr recording and strace output required used for
testing.  The produced strace can then be compared against a ReplaySession debug
log in order to determine corresponding events and trace lines, in order to put
together a test configuration.  Once complete, a user can utilize `rreplay` in
order to execute another replay execution.


### 5.3 `rreplay`

`rreplay` replays the recorded test generated by `rrtest` by attaching the
CrashSimulator injector and executing a test as described in the test's
configuration file.
