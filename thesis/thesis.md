---
author: |
    | William Findlay
title: |
    | Host-Based Anomaly Detection
    | with
    | Extended BPF
subtitle: |
    | COMP4906 Honours Thesis
date: \today
subparagraph: yes
documentclass: findlay
header-includes:
    - \addbibresource{../bib/thesis.bib}
    #- \makeatletter
    #- \def\@maketitle{
    #- \begin{center}
    #- {\Huge \@title \par}
    #- \vskip 1.5em
    #- by
    #- \vskip 1.5em
    #- {\large \bfseries\@author}
    #- \vskip 0.5em
    #- {\large \itshape \@date}
    #- \end{center}
    #- }
    #- \makeatother

classoption: 12pt
numbersections: true
---

<!-- Setup -->
\pagestyle{fancy}
\counterwithin{lstlisting}{section}

<!-- Title page -->
\maketitle
\thispagestyle{empty}

\begin{center}
    \large
    \vfill
    \includegraphics[width=0.5\textwidth]{logo/CarletonWide_K_186}
    \vfill
    Under the supervision of Dr.\ Anil Somayaji\\
    Carleton University
    \vfill
\end{center}

\onehalfspacing

\clearpage
\pagenumbering{roman}
\section*{Abstract}
\markboth{Abstract}{}

System introspection is becoming an increasingly attractive option
for maintaining operating system stability and security. This is primarily
due to the many recent advances in system introspection technology; in particular,
the 2014 introduction of *Extended Berkeley Packet Filter* (*eBPF*)
into the Linux Kernel [@starovoitov13; @starovoitov14] along with the recent
development of more usable interfaces such as the *BPF Compiler Collection* (*bcc*)
[@bcc] have resulted in a rich, performant, and (perhaps most importantly)
safe subsystem for both kernel and userland instrumentation.

The scope, safety, and performance of eBPF system introspection has potentially powerful applications
in the domain of computer security. In order to demonstrate this, I present
*ebpH*, an eBPF implementation of Somayaji's [@soma02] *Process Homeostasis* (*pH*).
ebpH is an intrusion detection system (IDS) that uses eBPF programs to instrument system calls
and establish normal behavior for processes, building a profile for each executable on the system;
subsequently, ebpH can warn the user when it detects process behavior that violates the established
profiles. Experimental results show that ebpH can detect anomalies in process behavior with negligible
overhead. Furthermore, ebpH's anomaly detection comes with minimal risk to the system thanks to the safety
guarantees of eBPF, rendering it an ideal solution for monitoring production systems.

This thesis will discuss the design and implementation of ebpH along with the technical
challenges which occurred along the way. It will then present experimental data and performance benchmarks
that demonstrate ebpH's ability to monitor process behavior with minimal overhead. Finally, it will conclude
with a discussion on the merits of eBPF IDS implementations and potential avenues for future work.

ebpH is licensed under GPLv2 and full source code is available at [https://github.com/willfindlay/ebph](https://github.com/willfindlay/ebph).

<!--
This thesis seeks to test the limits of what eBPF programs are capable of
with respect to the domain of computer security; specifically, I present *ebpH*,
an eBPF-based intrusion detection system based on Anil Somayaji's
[@soma02] *pH* (*Process Homeostasis*). Preliminary testing
has shown that ebpH is able to detect anomalies in process behavior by instrumenting
system call tracepoints with negligible overhead. Future work will involve testing and
iterating on the ebpH prototype, in order to extend its functionality beyond that
of the current prototype system.
-->

<!--
\noindent
\begin{tabular}{l l}
\textbf{Keywords} & eBPF, intrusion detection, system calls, Linux Kernel introspection,\\
& Process Homeostasis, ebpH
\end{tabular}
-->

\newpage
\section*{Acknowledgments}
\markboth{Acknowledgments}{}

First and foremost, I would like to thank my advisor, Anil Somayaji, for his
tireless efforts to ensure the success of this project as well as for providing
the original design for pH along with invaluable advice and ideas. Implementing ebpH
and writing this thesis has been a long process and not without its challenges.
Dr. Somayaji's support and guidance have been critical to the success of this
undertaking.

I would also like to thank the members and contributors of the *Iovisor Project*,
especially Yonghong Song
and Teng Qin
for their guidance and willingness to respond to issues and questions related to the *bcc* project.
Brendan Gregg's
writings and talks have been a great source of inspiration, especially with respect to background research.
Sasha Goldshtein's *syscount.py* was an invaluable basis
for my earliest proof-of-concept experimentation, although none of that original code has made it into this iteration of ebpH.

For their love and tremendous support of my education, I would like to thank my parents,
Mark and Terri-Lyn. Without them, I am certain that none of this would have been possible.
I would additionally like to thank my mother for suffering through the first draft of this thesis
and finding the many errors that come with writing a paper this large in Vim with no grammar checker.

Finally, I want to thank my dear friend, Amanda, for all the support she has provided me throughout
my university career. I couldn't have made it this far without you.

\singlespacing

<!-- Table of contents -->
\newpage
\begingroup
\hypersetup{linkcolor=black}
\tableofcontents

<!-- List of figs, tables, listings -->
\newpage
\listoffigures
\newpage
\listoftables
\newpage
\lstlistoflistings
\endgroup

\onehalfspacing

<!-- Setup the rest of the document -->
\newpage
\pagenumbering{arabic}
\setcounter{page}{1}

# Introduction

As our computer systems grow increasingly complex, so too does it
become more difficult to gauge precisely what they are doing
at any given moment. Modern computers are often running hundreds, if not thousands
of processes at any given time, the vast majority of which are running silently in
the background. As a result, users often have a very limited notion of what exactly
is happening on their systems, especially beyond that which they can actually see
on their screens. An unfortunate corollary to this observation is that
users *also* have no way of knowing whether their system may be *misbehaving*
at a given moment, whether due to a malicious actor, buggy software, or simply
some unfortunate combination of circumstances [@soma02].

In order to solve this problem of limited visibility, we must turn to
system introspection to provide the necessary tools to gain more
information about system state. While traditional systems introspection
techniques are either too slow, too unsafe, or too limited in scope to be
used effectively in production, a number of recent advances in this field
have presented increasingly attractive options. The introduction
of Dtrace [@cantrill04] in 2004 by Cantrill et al. was an early example
of a robust monitoring framework that could be used in production, although
it came with its own drawbacks, such as an inability to specify complex tracing
programs, as it defined only a high level scripting language for doing so.
In 2014, eBPF (*Extended Berkeley Packet Filter*) [@starovoitov13; @starovoitov14]
was introduced to the Linux kernel. eBPF revolutionized Linux tracing by providing
a safe means of injecting custom bytecode into the kernel from userspace.
These programs could be relatively complex compared to early solutions such as Dtrace,
and subsequent improvements to the eBPF paradigm [@starovoitov19] have seen that complexity
increase significantly.

With the advent eBPF, it is now possible define complex programs to trace nearly all aspects
of a running Linux safely and efficiently. Previously, this would only have been possible
via direct modification of the kernel, either through direct patches or loadable kernel modules,
which both lack the safety guarantees of eBPF. *Process Homeostasis* (pH) [@soma02] is an early
anomaly detection system, implemented as a patch for the Linux 2.2 kernel. In this thesis,
I will present *ebpH* (a portmanteau of eBPF and pH), which provides an eBPF re-implementation
of the original pH system. ebpH provides production safety and forward compatibility guarantees
that were not possible in the original implementation, which will greatly improve its adoptability.
Experimental results show that ebpH is capable of monitoring production systems under moderate to heavy
workloads with negligible performance overhead --- in some cases, even showing performance *improvements* over the
original system. Further, thanks to the safety guarantees of eBPF, zero kernel panics occurred
during its development and testing, a feat which would not have been possible with a kernel-based
implementation.

The rest of this thesis will cover the necessary background material required to understand ebpH,
its design and implementation, experimental performance results, and plans for future work
and iteration on the current prototype.

# Background

In the following sections, I will provide the necessary background information needed
to understand ebpH; this includes an overview of system introspection and tracing techniques
on Linux including eBPF itself, and some background on system calls and intrusion detection.

While my work is primarily focused on the use of eBPF for maintaining
system security and stability, the working prototype for ebpH borrows
heavily from Anil Somayaji's *pH* or *Process Homeostasis* [@soma02],
an anomaly-based intrusion detection and response system written as a patch for Linux Kernel 2.2.
As such, I will also provide some background on the original pH system and many of the design choices therein.

## An Overview of the Linux Tracing Landscape

\label{tracing-section}

System introspection is hardly a novel concept; for years, developers
have been thinking about the best way to solve this problem and have come up
with several unique solutions, each with a variety of benefits and drawbacks.
\autoref{introspection-summary} presents an overview of some prominent examples
relevant to GNU/Linux systems.

\begin{table}[p]
\caption{A summary of various system introspection technologies available for GNU/Linux systems.}
\label{introspection-summary}
\begin{center}
\begin{tabular}{>{\ttfamily}lp{3.8in}l}
    \toprule
\multicolumn{1}{l}{name} & Interface and Implementation & Citations\\
    \midrule
strace & Uses the \texttt{ptrace} system call to trace an invidivual userspace process & \cite{strace, manstrace}  \\
ltrace & Uses the \texttt{ptrace} system call to trace library calls in an individual userland process & \cite{rubirabranco07, manltrace}  \\
SystemTap & Dynamically generates loadable kernel modules for instrumentation; newer versions can optionally use eBPF as a backend instead &  \cite{systemtap, merey17}\\
ftrace & Sysfs pseudo filesystem for tracepoint instrumentation located at \texttt{/sys/kernel/debug/tracing} & \cite{ftrace}\\
perf\_events & Linux subsystem that collects performance events and returns them to userspace & \cite{manperfeventopen}  \\
LTTng & Loadable kernel modules, userland libraries & \cite{lttng}\\
dtrace4linux & A Linux port of DTrace via a loadable kernel module & \cite{dtrace4linux} \\
sysdig & Loadable kernel modules for system monitoring; native support for containers & \cite{sysdig} \\
eBPF & In-kernel execution of pre-verified, JIT-compiled bytecode & \cite{bcc, goldstein16, starovoitov13, starovoitov14} \\
    \bottomrule
\end{tabular}
\end{center}
\end{table}

These technologies can, in general, be classified into a few broad categories (\autoref{instr-cmp}),
albeit with potential overlap depending on the tool:

1) `ptrace`-based instrumentation;
1) Userland libraries;
1) Loadable kernel modules;
1) Kernel subsystems.

\begin{figure}[p]
\begin{center}
\includegraphics[keepaspectratio, height=3.2in]{../figures/instr-cmp.png}
\end{center}
\caption[A high level overview of the broad categories of Linux instrumentation]{
A high level overview of the broad categories of Linux instrumentation.
This does not represent a complete picture of all available tools and interfaces,
but instead presents many of the most popular ones. Note how eBPF covers every presented use case.
}
\label{instr-cmp}
\end{figure}

\FloatBarrier

Applications such as strace [@strace; @manstrace] or gdb [@gdb20] which make use of the ptrace system call are certainly
a viable option for limited system introspection with respect to specific processes.
However, this does not represent a complete solution, as the user is limited to monitoring
the system calls made by a process to communicate with the kernel, its memory, and the state of its registers,
rather than the underlying kernel functions themselves [@manptrace]. The scope of ptrace-based solutions is
also limited by ptrace's lack of scalability; ptrace's API is conducive to tracing single processes at a time rather
than tracing processes system wide. Its limited scale becomes even more obvious when considering the high
amount of context-switching between kernel space and user space required when tracing multiple processes or threads,
especially when these processes and threads make many hundreds of system calls per second [@keniston07].

Although library call instrumentation through software such as ltrace [@rubirabranco07; @manltrace]
does not necessarily suffer from the
same performance issues as described above, it still constitutes a suboptimal solution for many
use cases due to its limited scope. In order to be effective and provide a complete picture of what
exactly is going on during a given process' execution, library tracing needs to be combined with other solutions.
In fact, ltrace does exactly this; when the user specifies the \texttt{-S} flag, ltrace uses the ptrace system call
to provide strace-like system call tracing functionality.

Out of tree LKM-based implementations such as sysdig [@sysdig] and SystemTap [@systemtap] offer an extremely
deep and powerful tracing solution given their ability to instrument the entire system, including
the kernel itself. Their primary detriment is a lack of safety guarantees with respect to the modules
themselves. No matter how vetted or credible a piece of software might be, running it natively in
the kernel always comports with an inherent level of risk; buggy code might cause system failure,
loss of data, or other unintended and potentially catastrophic consequences. Additionally,
such kernel-module-based solutions are highly reliant on specific versions of the Linux kernel;
changes to Linux's API may cause them to break, which in turn requires updates; these updates
then increase the risk of introducing bugs into the codebase, which may in turn lead to the
aforementioned consequences of code failure in kernelspace.

Custom tracing solutions through kernel modules carry essentially the same risks.
No sane manager would consent to running untrusted, unvetted code natively in the kernel
of a production system; the risks are simply too great and far outweigh the benefits.
Instead, such code must be carefully examined, reviewed, and tested, a process which
can potentially take months. What's more, even allowing for a careful testing and vetting
process, there is always some probability that a bug can slip through the cracks, resulting
in catastrophic consequences.

Kernel subsystems such as eBPF [@starovoitov13; @starovoitov14],
ftrace [@ftrace], and perf\_events [@manperfeventopen] seem to be the most desirable choice
of any of the presented solutions. ftrace is a virtual filesystem located at `/sys/kernel/debug/tracing`
which is used for the instrumentation of tracepoints and perf\_events is used to instrument hardware performance
counters. While these two subsystems are useful in their own right, they suffer from poor documentation and
limited user interfaces; eBPF eclipses their functionality entirely and presents a much more compelling interface
for the development of complex applications.

### Comparing eBPF and Dtrace

It is worth spending a bit more time comparing eBPF with Dtrace, as both
APIs are quite full-featured and designed with similar functionality in mind.
The original Dtrace [@cantrill04] was designed in 2004 for Solaris and lives on to this
day in several operating systems, including Solaris, FreeBSD, MacOS X [@gregg14], and
Linux [@dtrace4linux] (the dtrace4linux implementation will be examined with
more scrutiny later in this section).

In general, the original Dtrace and the current version of eBPF share much of the same
functionality and cover similar use cases [@cantrill04; @starovoitov13; @starovoitov14].
This includes perhaps most notably dynamic instrumentation
in both userspace and kernelspace, arbitrary context instrumentation (i.e.\ the ability
to instrument essentially any aspect of the system), and guarantees of safety
and data integrity. The difference between the two systems generally boils
down to the following points [@gregg14; @gregg19bpf]:

1) eBPF supports a superset of Dtrace's functionality;
1) Dtrace provides only a high level interface, while eBPF provides both
low level and high level interfaces (see \autoref{bpf-dtrace-api});
1) Dtrace is useful for writing one-liner scripts, but not for writing more complex programs;
1) eBPF is natively supported in Linux, while Dtrace ports are purely
LKM-based and out of tree.

\begin{figure}
\includegraphics[width=0.5\textwidth]{../figures/bpf-dtrace-api.png}
\caption[Comparing Dtrace and eBPF functionality with respect to API design]
{
Comparing Dtrace and eBPF functionality with respect to API design (adapted from \cite{gregg18}).
Note that eBPF covers more complex use cases and supports both low level
and high level APIs. Dtrace needs to be used in tandem with shell
scripting to support more complex use cases.
}
\label{bpf-dtrace-api}
\end{figure}

dtrace4linux [@dtrace4linux] is a free and open source port of Sun's Dtrace [@cantrill04] for the
Linux Kernel, implemented as a loadable kernel module (LKM). While Dtrace offers
a powerful API for full-system tracing, its usefulness is, in general, eclipsed by
that of eBPF [@gregg18] and requires extensive shell scripting for use cases beyond
one-line tracing scripts. In contrast, with the help of powerful and easy to use
front ends like bcc [@bcc], developing complex eBPF programs for a wide variety of use cases
is becoming an increasingly painless process.

Not only does eBPF cover more complex use cases than Dtrace, but it also
provides support for simple one-line programs through tools like bpftrace [@gregg18; @bpftrace]
which has been designed to provide a high-level Dtrace-like tracing language
for Linux using eBPF as a backend. Although bpftrace only provides a subset of
Dtrace's functionality [@gregg18], its feature set has been carefully curated in order
to cater to the most common use cases and more functionality is being added
on an as-needed basis.

Additional work is being done to fully reimplement Dtrace as a new BPF program type
[@vanhees19] which will further augment eBPF's breadth and provide full
backward compatibility for existing Dtrace scripts to work with eBPF.
This seems to be by far the most promising avenue for Linux Dtrace support thus far,
as it seeks to combine the higher level advantages of Dtrace with
the existing eBPF virtual machine.

\FloatBarrier

## Classic BPF

In 1992, McCanne and Jacobson [@bpf] introduced the original BPF\footnote{Hereafter,
I will refer to the original BPF as {\itshape Classic BPF} to avoid confusion with eBPF and the BPF programming paradigm.}
or *Berkeley Packet Filter* as a mechanism for capturing,
monitoring, and filtering network traffic in the BSD kernel.
Classic BPF's primary insights were two-fold:

1) Network traffic events are *frequent* and *fast*, and therefore an efficient filtering mechanism was needed;
1) A limited, register-based bytecode being run in an in-kernel virtual machine provides precisely the mechanism described in point (1).

The virtual machine described above was used to implement the *filter* component of BPF, while in-kernel network function
tracepoints implemented the *tap* component. The tap forwarded packet events to the filter, which then decided what to do
with the packets according to a user-defined BPF program. McCanne and Jacobson showed that their approach was much faster
than other contemporary packet filtering techniques, namely NIT [@nit] and CSPF [@mogul87].

While Classic BPF is certainly a powerful technique for filtering packets, Starovoitov
[@starovoitov13; @starovoitov14] realized that its tap and filter mechanism represented a desirable
approach for general system introspection. Therefore, in 2013, he proposed
*Extended BPF* (eBPF), a superset of Classic BPF, which vastly increased the
capabilities of the original BPF virtual machine.

<!--
Since its original introduction, eBPF has offered a consistent, powerful, and production-safe
mechanism for general Linux introspection and continues to improve rapidly over time.
eBPF is discussed in more detail in the following section.
-->

## eBPF: Linux Tracing Superpowers

\label{ebpf-superpowers}

In 2016, eBPF was described by Brendan Gregg [@gregg16]
as nothing short of *Linux tracing superpowers*.
I echo that sentiment here, as it summarizes eBPF's capabilities perfectly.
Through eBPF programs, one can simultaneously trace userland symbols and library
calls, kernel functions and data structures, and hardware performance. What's more,
through an even newer subset of eBPF, known as *XDP* or *Express Data Path* [@hoiland18], one
can inspect, modify, redirect, and even drop packets entirely before they even reach the main kernel network stack.
\autoref{ebpf-use-cases} provides a high level overview of these use cases and the corresponding
eBPF instrumentation required.

\begin{figure}
\includegraphics{../figures/eBPF-use-cases.png}
\caption[A high level overview of various eBPF use cases]{
A high level overview of various eBPF use cases.
Note the high level of flexibility that eBPF provides
with respect to system tracing.}
\label{ebpf-use-cases}
\end{figure}

The advantages of eBPF extend far beyond scope of traceability; eBPF
is also extremely performant, and runs with guaranteed safety.
In practice, this means that eBPF is an ideal tool for use in
production environments and at scale.

Safety is guaranteed with the help of an in-kernel verifier that
checks all submitted bytecode before its insertion into the BPF virtual machine.
While the verifier does limit what is possible (eBPF in its current state is *not* Turing complete), it is constantly being
improved; for example, a recent patch [@starovoitov19] that was mainlined in the Linux 5.3 kernel
added support for verified bounded loops, which greatly increases the computational possibilities of eBPF.
The verifier will be discussed in further detail in \autoref{verifier-section}.

eBPF's superior performance can be attributed to several factors.
On supported architectures,\footnote{x86-64, SPARC, PowerPC, ARM, arm64, MIPS, and s390 \cite{fleming17}}
eBPF bytecode is compiled into machine code using a *just-in-time* (*JIT*)
compiler; this both saves memory and reduces the amount of time it takes
to insert an eBPF program into the kernel. Additionally, since eBPF runs in-kernel
and communicates with userland via direct map access and perf events, the number of context switches
required between userland and the kernel is greatly diminished, especially compared to approaches
such as the ptrace system call.

### How eBPF Works at a High Level

From the perspective of a user, the eBPF workflow is surprisingly simple.
Users can elect to write eBPF bytecode directly (not recommended) or use one
of many front ends to write in higher level languages that are then used to
generate the respective bytecode. IOVisor's bcc [@bcc] offers bindings for several languages including
Python, Go, and C++; users write eBPF programs in C and interact with bcc's API
in order to generate eBPF bytecode and submit it to the kernel.

\autoref{ebpf-topology} presents an overview of eBPF's architecture and dataflow,
including the interaction between userspace programs, eBPF programs in kernelspace,
and the rest of the kernel. This interaction occurs via the `bpf(2)` system call [@man-bpf]
which is used to load and verify BPF programs, issue commands to BPF programs, and
interact with eBPF maps. These maps are the mechanism for sending data between BPF
programs and other BPF programs or BPF programs and userspace.

\begin{figure}[p]
    \includegraphics[width=0.8\textwidth]{../figures/ebpf-arch.png}
\caption[Basic topology of eBPF with respect to userland and the kernel]{
Basic topology of eBPF with respect to userland and the kernel.
Note the bidirectional nature of dataflow between userspace and kernelspace
using maps.}
\label{ebpf-topology}
\end{figure}

<!--
Considering bcc's Python front end as an example:
the user writes their BPF program in C and a user interface in Python. Using a provided
BPF class, the C code is used to generate bytecode which is then submitted
to the verifier to be checked for safety. Assuming the BPF program passes
all required checks, it is then loaded into an in-kernel virtual machine. From there,
we are able to attach onto various probes and tracepoints, both in the kernel and in userland.
-->

\begin{table}[p]
    \caption[Various map types available in eBPF programs, as of Linux 5.5]{
        Various map types \cite{bcc, gregg19bpf} available in eBPF programs, as of Linux 5.5.
    }
    \label{ebpf-maps}
    \resizebox{\textwidth}{.2\textheight}{
    \begin{tabular}{>{\ttfamily}lp{5in}}
    \toprule
    \multicolumn{1}{l}{Map Type} & Description\\
    \midrule
    HASH & A hashtable of key-value pairs\\
    ARRAY & An array indexed by integers; members are zero-initialized\\
    PROG\_ARRAY & A specialized array to hold file descriptors to other BPF programs; used for tail calls\\
    PERF\_EVENT\_ARRAY & Holds perf event counters for hardware monitoring\\
    PERCPU\_HASH & Like \texttt{HASH} but stores a different copy for each CPU context\\
    PERCPU\_ARRAY & Like \texttt{ARRAY} but stores a different copy for each CPU context\\
    STACK\_TRACE & Stores stack traces for userspace or kernerlspace functions\\
    CGROUP\_ARRAY & Stores pointers to cgroups\\
    LRU\_HASH & Like a \texttt{HASH} except least recently used values are removed to make space\\
    LRU\_PERCPU\_HASH & Like \texttt{LRU\_HASH} but stores a different copy for each CPU context\\
    LPM\_TRIE & A "Longest Prefix Matching" trie optimized for efficient traversal\\
    ARRAY\_OF\_MAPS & An \texttt{ARRAY} of file descriptors into other maps\\
    HASH\_OF\_MAPS & A \texttt{HASH} of file descriptors into other maps\\
    DEVMAP & Maps the \texttt{ifindex} of various network devices; used in XDP programs\\
    SOCKMAP & Holds references to \texttt{sock} structs; used for socket redirection\\
    CPUMAP & Allows for redirection of packets to remote CPUs; used in XDP programs\\
    \bottomrule
\end{tabular}
}
\end{table}

There are several map types available in eBPF which cover a wide variety of use cases.
These map types along with a brief description are provided in \autoref{ebpf-maps}.
Thanks to this wide arsenal of maps, eBPF developers have a powerful set of both general-purpose
and specialized data structures at their disposal; as shown in coming sections,
many of these maps are quite versatile and have use cases beyond what might initially
seem pertinent. For example, the `ARRAY` map type may be used to initialize large data structures
to be copied into a general purpose `HASH` (refer to \autoref{appendix-bigdata} in \autoref{ebpf-design-patterns}).
This can be effectively used to bypass the verifier's stack space limitations, which are discussed in detail
in \autoref{verifier-section}.

### Tracepoints, Kprobes, and Uprobes

As shown previously in \autoref{ebpf-use-cases} on page \pageref{ebpf-use-cases}, eBPF supports
a number of distinct program types which may be used to instrument and interact with various
aspects of system functionality. ebpH's functionality mainly relies on three specific program
types: the *tracepoint*, the *kprobe*, and the *uprobe* [@gregg19bpf; @bcc]. Here, I will describe
what each program type does and how they work at a high level. These concepts
will be revisited frequently when discussing ebpH's implementation in \autoref{sec:impl}.

#### Tracepoints

Tracepoints [@gregg19bpf] define the stable kernel tracing API of eBPF; at a high level, they are predefined
sections in kernel code that trap to eBPF handlers when these handlers are defined.
Tracepoints are stable in the sense that the information exposed to eBPF by a tracepoint
will not be likely to change between kernel versions, which means that they are ideal for
use in production where forward compatibility with new versions of Linux is a desirable property.
Although using tracepoints is ideal when possible, they have a few caveats; in particular,
a limited number of tracepoints are defined by the kernel, and they do not cover an exhaustive
list of kernel functionality. Linux 5.5 defines 1,872 tracepoints in total.

ebpH uses tracepoints to implement the bulk of its kernelspace functionality
(c.f. \autoref{ebph_profiles} and \autoref{ebph_processes}). System call tracepoints
are used to track system call sequences for its anomaly detection functionality; scheduler tracepoints
are used to maintain the set of traced processes and to associate these processes with the correct
profiles.

#### Kprobes

Whereas tracepoints define a stable kernel tracing API, kprobes [@gregg19bpf] can be though of as
their dynamic counterparts. Although kprobes may be used to trace any exported kernel function
(that is, any kernel function that is not in-lined by the compiler), they are not considered
stable as the API of a kprobe changes whenever the corresponding kernel function changes.
Thus, tracepoints are preferred when possible and kprobes are generally used as a last resort.
Kprobes work by inserting a breakpoint at a specific address in kernel memory; when this breakpoint
is hit, the kernel traps to the associated BPF handler. Kprobes can also be used to
instrument function returns, in which case they are known as kretprobes.

ebpH uses only one kretprobe, and does so to keep track of when processes receive
a signal that would trap to a signal handler (c.f. \autoref{non_determinism}).

#### Uprobes

Uprobes and uretprobes work in a similar fashion to kprobes and kretprobes; the primary difference
here is that we are now instrumenting *userspace* rather than kernelspace. A breakpoint is inserted
at the target userspace address and this breakpoint traps to the appropriate BPF handler.

ebpH uses uprobes to implement sending complex commands to the BPF program from a userspace library
(c.f. \autoref{ebph_admin_sec}).

### The Verifier: The Good, the Bad, and the Ugly
\label{verifier-section}

The verifier is responsible for eBPF's unprecedented safety given its scope, one of its
most attractive qualities with respect to system tracing. While this verifier
is quintessential to the safety of eBPF,
it is not without its drawbacks. In this section, I describe how the verifier works,
its nuances and drawbacks, and recent work that has been done to improve
the verifier's support for increasingly complex eBPF programs.

Proving the safety of arbitrary code is by definition a difficult problem.
This is thanks in part to theoretical limitations on what is actually provable;
a famous example is the halting problem described by Turing circa 1937 [@turing37].
This difficulty is further compounded by stricter requirements for safety in the context
of an eBPF program; in particular, developers don't want BPF programs to crash or otherwise
damage the kernel [@fleming17].

To illustrate the importance of this problem of safety with respect to eBPF,
let us consider a simple example. We will again consider the halting problem
described above. Suppose we have two eBPF programs, program $A$ and program $B$,
that each hook onto a mission-critical kernel function (`schedule()`, for example).
The only difference between these two programs is that program $A$ always terminates,
while program $B$ runs forever without stopping. What this means in practice is that
the call to `schedule()` will never succeed, and program $B$ effectively constitutes a
denial of service attack [@hussain03] on our system, intentional or otherwise; allowing untrusted
users to load this code into our kernel spells immediate disaster for our system.

By the same token, unbounded memory access attempts within a BPF program may permit
buffer overflows, which may in turn be manipulated to gain arbitrary code execution in kernelspace [@chen11]
(the kind that actually *can* damage the system). In order to aid static analysis of memory access, the verifier
prohibits memory access using registers with unbounded values. For example, accessing an array with induction
variable $i$ in a `for` loop would be prohibited unless it could be shown that this variable's set of possible
values exists within a memory-safe range.

While I have established that verifying the safety of eBPF programs is an important problem to solve,
the question remains as to whether it is *possible* to solve.
For the reasons outlined above, this problem should intuitively seem impossible,
or at least far more difficult than should be feasible. So, what can the verifier do? The answer
is to *change the rules* to make it easier. In particular, while it is difficult to prove
that the set of all possible eBPF programs are safe, it is much easier\footnote{\emph{Easier}
here means \emph{computationally easier}, certainly not trivial.}
to prove this property for a subset of all eBPF programs. \autoref{valid-ebpf} depicts
the relationship between potentially valid eBPF code and verifiably valid eBPF code.

\begin{figure}[p]
    \caption[The set participation of valid C and eBPF programs.]{
        The set participation of valid C and eBPF programs.
        Valid eBPF programs written in C constitute a small subset of all valid C programs.
        Verifiably valid eBPF programs constitute an even smaller subset therein.
    }
    \label{valid-ebpf}
    \includegraphics[height=.4\textheight]{../figures/valid-ebpf.png}
\end{figure}

\begin{figure}[p]
    \caption[Complexity and verifiability of eBPF programs.]{
        Complexity and verifiability of eBPF programs.
        Safety guarantees for eBPF programs rely on certain compromises.
        Ideally the relationship would be as shown on the bottom;
        in practice, the relationship is getting closer over time, but is still
        far from the ideal.
    }
    \label{complexity-verifiability}
    \includegraphics[height=.4\textheight]{../figures/complexity-verifiability.png}
\end{figure}

The immediate exclusion of eBPF programs meeting certain criteria is the crux
of eBPF's safety guarantees. Unfortunately, it also rather intuitively
limits what developers are actually able to do with eBPF programs. In particular,
eBPF is not a Turing complete language; it prohibits arbitrary jump instructions,
cycles in execution graphs, and unverified memory access. Further, eBPF limits
stack allocations to only 512 bytes [@gregg19bpf] --- far too small for many practical use cases.
From a security perspective, these limitations are a *good thing*, because they allow us to immediately exclude eBPF
programs with unverifiable safety; but from a usability standpoint, particularly that of a new eBPF
developer, the trade-off is not without its drawbacks.

Fortunately, the eBPF verifier is getting better over time (\autoref{complexity-verifiability}).
When I say *better*, what I mean is that it is able to prove the safety of increasingly complex programs.
Perhaps the best example of this steady improvement is a recent kernel patch [@starovoitov19] that added
support for bounded loops in eBPF programs. With this patch, the set of viable
eBPF programs was *greatly* increased; in fact, ebpH in its current incarnation relies
heavily on bounded loop support. Prior to bounded loops, eBPF programs relied on *unrolling*
loops at compile time, a technique that was both slow and highly limited. This is just one example
of the critical work that is being done to improve the verifier and thus improve eBPF as a whole.

## System Calls

ebpH (and the original pH system upon which it is based) works by instrumenting
*system calls* in order to establish behavioral patterns for all binaries
running on the system. Understanding pH and ebpH requires a reliable mental model
of what a system call is and how programs use them to communicate with the kernel.

At the time of writing this thesis, the Linux Kernel [@unistd] supports
an impressive 439 distinct system calls, and this number generally grows
with subsequent releases. In general, userspace libraries such as the C standard library
implement a subset of these
system calls, with the exact specifications varying depending on architecture.
These system calls are used to request services from the operating system kernel;
for example, a program that needs to write to a file would make an `open` call
to receive a file descriptor into that file, followed by one or more `write` calls
to write the necessary data, and finally a `close` call to clean up the file descriptor.
These system calls form the basis for much of our process behavior, from I/O as seen above,
to process management, memory management, and even the execution of binaries themselves.

Critically, from a security perspective, system calls provide the interface to the kernel's
*reference monitor* [@jaeger08; @vanoorschot19; @anderson72], an abstraction that refers to
the kernel's facilities for mediating access by subjects (i.e. users and their processes)
onto system objects (i.e. security-sensitive resources). This means that system calls
provide a highly representative picture of a given process' attempts to access resources
that we care about --- whether this access is valid or otherwise.

Through the instrumentation of system calls, we can establish
a clear outline of exactly how a process is behaving, the critical
operations it needs to perform, and how these operations interact with
one another. In fact, system call-based instrumentation forms a primary use
case for several of the tracing technologies previously discussed in \autoref{tracing-section},
perhaps most notably strace. We will discuss the behavioral implications of system calls
further in \autoref{ph-lap}.

## Intrusion Detection

The concept of intrusion detection has seen prevalent attention in academic work since
the early 1980's [@anderson80; @denning85; @denning87].
At a high level, intrusion detection systems (IDS) strive to monitor
systems at a particular level and use observed data to make decisions
about the legitimacy of these observations [@kemmerer02].
Intrusion detection systems can be broadly classified into several categories based on data collection, detection
technique(s), and response. \autoref{ids-overview} presents a broad and incomplete
overview of these categories.

\begin{figure}[p]
    \caption[An overview of the basic categories of IDS]{
        A broad overview of the basic categories of IDS.
        The current version of ebpH can be classified according to the
        categories that have been \underline{underlined}. Note that intrusion detection
        system classification can often be more nuanced than the basic overview presented
        here. However, this should present a good enough overview to understand IDSes in
        the context of ebpH.
    }
    \label{ids-overview}
    \includegraphics{../figures/ids-overview.png}
\end{figure}

In general, intrusion detection systems can either attempt to detect
anomalies (i.e. mismatches in behavior when compared to normally observed patterns)
or misuse-based, which generally refers to matching known attack patterns to observed data [@kemmerer02; @vanoorschot19].
In general, anomaly-based approaches cover a wider variety of attacks while
misuse-based approaches tend to yield fewer false positives. Misuse-based approaches
can also be further broken down into specification-based and signature-based, which
deal in behavioral blacklists and whitelists respectively.
A hybrid approach between any of these techniques is also possible.

Data collection is generally either host-based or network based.
Network-based IDSes examine network traffic and analyze it to detect
attacks or anomalies. In contrast, host-based IDSes analyze the state
of the local system [@kemmerer02; @soma02].

Responses can vary significantly based on the system, but can be classified into
two main categories: alerts and counter-attacks. Systems can either choose to alert
an administrator about the potential issue, or choose to mount counter-measures
to defeat or mitigate the perceived attack [@kemmerer02]. Naturally, systems
also have the option to take a hybrid approach here.

Using the above metrics, ebpH can be broadly classified as a host-based
anomaly detection system that responds to anomalies by issuing alerts.
This is generally quite similar to the original pH (\autoref{process-homeostasis})
with one major exception:
As we will see, the original pH also responds to anomalies by delaying
system calls outright and preventing anomalous `execve(2)` calls [@soma02].
Implementing this functionality in ebpH is a topic for future work (c.f. \autoref{response_automation}),
and currently ebpH only supports the anomaly detection aspect of its predecessor.

### A Survey of Data Collection in Intrusion Detection Systems

\label{data_collection_survey}

We have presented the general classification of intrusion detection systems
through the establishment of three core elements of an IDS and several
categories therein. As it relates to eBPF, the *data collection* component of
intrusion detection systems is of particular interest; what is especially
exciting about eBPF is its impressive scope, safety, and performance with respect
to general system introspection; this presents a perfect trifecta of traits for
collecting system data. As such, it is worth examining data collection
techniques from various intrusion detection systems in more detail.

We have established that data collection in intrusion detection systems
can primarily be separated into two relatively broad categories:

1) host-based data collection which involves collecting data
about the use and behavior of a local machine;
1) network-based data collection which involves monitoring network traffic
and looking for established patterns.

While the above two categories are generally sufficient for understanding intrusion
detection at a high level, there are in fact several distinct subcategories therein.
\autoref{data-collection-categories} presents an overview of the most common data
collection subcategories in IDSes.

\begin{figure}[p]
    \caption[An overview of the most common data collection categories and subcategories in IDS]{
        An overview of the most common data collection categories and subcategories in IDS,
        as well as a potentially new and promising category, {\itshape general system introspection}, thanks to eBPF.
        This figure primarily synthesizes the technologies presented in \cite{spafford02, stallings07}.
    }
    \label{data-collection-categories}
    \includegraphics{../figures/data-collection-categories.png}
\end{figure}

#### Internal and External Sensors

Kerschbaum et al.\ [@spafford02] introduce the concept of *internal* sensors
for intrusion detection and contrast them with the far more popular *external* sensors.
An internal sensor by definition is included in the source code of a
monitored component, while an external sensor is implemented outside
of the monitored component. These two categories of sensors each present unique
advantages and disadvantages [@spafford02]. In particular, external sensors are easily modifiable
and extensible, although they introduce potential delays, and are generally weaker
to tampering by intruders; internal sensors minimize overhead (assuming correct implementation)
and are much more resistant to intruder tampering,
but suffer from reduced portability, difficulty in implementation,
and may incur severe performance penalties if implemented incorrectly.

eBPF would fall under the internal sensor classification [@spafford02] due to its implementation within the Linux Kernel;
however, eBPF presents a rather unique case, as it overcomes many of the disadvantages proposed by Kerschbaum et al.
while maintaining the advantages. Specifically, eBPF is completely application transparent, portable to any modern
version of Linux\footnote{Although eBPF is available on all modern kernels, some of its features are specific to the
very newest versions. In particular, recent verifier updates which allow for increased complexity have only been
available since version 5.3. See \autoref{ebpf-superpowers} and \autoref{verifier-section} for more details.},
easy to update and modify, and has guaranteed performance and safety
due to the verifier.

#### Internal Host-Based Approaches

System call pattern analysis was examined in detail by Forrest et al.\ [@forrest96]
and culminated in the development of the original pH system [@soma02] on which ebpH
is based. Somayaji and Inoue [@soma07] compared two methods of organizing system call data for
intrusion detection (full sequences and lookahead pairs), which we will discuss further in \autoref{ph-lap}.

Kerschbaum et al.\ also describe a generic method of application-specific internal sensors
through the addition of audit checks in program source code [@spafford02]. However,
the primary caveat here is that such checks need to be integrated into a specific application
early enough such that refactoring is minimized [@peng95; @spafford02].
This approach is also far less generic than other internal sensor approaches described here.

Another potential internal source for data is through host-based network stack
introspection. Classic BPF [@bpf] and eBPF/XDP [@starovoitov13; @starovoitov14; @bcc; @hoiland18] are quite excellent at this.
Host-based network introspection allows the analysis of network traffic at various points in the kernel's
networking stack, and XDP packet redirection [@hoiland18] allows fast detection and response before a packet even reaches
the main networking stack.

ebpH itself constitutes an internal host-based approach; that is, it uses
eBPF for in-kernel instrumentation of system calls (internal) on a given host (host-based).
As we discuss in \autoref{general_introspection}, a potential avenue for future research
in ebpH is moving beyond system call monitoring to *general system introspection* (c.f.\ \autoref{data-collection-categories}).
This is specifically a possibility due to eBPF's unique classification as an internal
sensor capable of monitoring the entire system dynamically, safely, and with minimal overhead.

#### External Host-Based Approaches

External host-based data collection is very popular in intrusion detection.
This can be primarily attributed due to the advantages described by Kerschbaum et al.\ [@spafford02],
particularly with respect to ease of implementation and portability.

AAFID [@spafford00] uses a *combined* internal/external approach based on separate autonomous
agents running continuously on multiple hosts. These agents make use of various
data sources, such as external programs (i.e.\ `ps`, `netstat`, and `df`),
file system state, and network interface packet capture (i.e.\ hooking into the host's networking stack).
Agents supplement collected data by analyzing audit logs generated by the system [@spafford02].

In 1999, Kuperman and Spafford [@kuperman99] proposed the use of library interpolation
for intrusion detection in dynamically linked binaries. Library interpolation is
a method of interposing a custom library implementation between a dynamically
linked executable and its shared objects. This effectively allows the generation of
custom audit data on each library call that a process makes.

#### Internal and External Network-Based Approaches

Network-based approaches [@stallings07] to intrusion detection involve the inspection of network traffic
en route to its destination. This typically comes in the form of inspecting packets headers, payloads, and frequency
to establish patterns for analysis. Generally, network-based approaches have a choice between using
either inline (internal) sensors, or passive (external) sensors for data collection [@stallings07].
An inline sensor either hooks into a network device, or is built into specialized hardware;
traffic passes directly through the sensor and is analyzed directly.

In contrast, passive sensors create copies of network traffic for analysis. This approach is
typically favored since it does not introduce delays into the traffic itself, instead
sacrificing the ability to respond to threats before they reach their destination [@stallings07].
This result is consistent with Kerschbaum et al.'s observation that external sensor approaches tend to be
favored over their internal counterparts [@spafford02].

### eBPF and XDP for Network Intrusion Detection

Most work in eBPF-based intrusion detection leverages its network monitoring capabilities.
Rather than a host-based approach to data collection, these solutions generally fall within
the network-based category; for instance, an eBPF/XDP filter may be set up at a strategic point
within a network to analyze incoming traffic. ntopng [@deri19] uses BPF tracepoints and kprobes to analyze
incoming TCP traffic, and error injection to reject bad connections. Cloudflare has built
DDoS mitigation systems [@bertin17; @fabre18] which use XDP [@hoiland18] to enforce automatically generated
policies before packets enter the main kernel networking stack. Suricata [@suricata18; @yates17] provides
optional support for eBPF and XDP to optimize its network intrusion detection stack performance and to enable
it to drop packets earlier than otherwise would have been possible.
While these solutions have been shown to be efficient and effective against network-based attacks,
none of them focuses on protecting the host itself. Since eBPF can be used to monitor all
aspects of system behavior, this represents a small subset of eBPF's potential use cases
in the field of intrusion detection. ebpH has been designed to rectify this gap in the research.

## Process Homeostasis

Anil Somayaji's *Process Homeostasis* [@soma02], styled as *pH*, forms the basis for
ebpH's core design; as such, it is worth exploring the original implementation,
design choices, and rationale therein. Using the same IDS categories from the previous
section, we can classify pH as a host-based anomaly detection system that responds
by both issuing alerts *and* mounting countermeasures to reduce the impact of anomalies;
in particular pH responds to anomalies by injecting delays into a process' system calls
proportionally to the number of recent anomalies that have been observed [@soma02].
It is in this way that pH lives up to its name: these delays make process behavior
*homeostatic*.

### Anomaly Detection Through Lookahead Pairs

\label{ph-lap}

pH uses a technique known as *lookahead pairs* [@soma02; @soma07] for detecting anomalies
in system call data. This is in stark contrast to other anomaly detection
systems at the time that primarily relied on *full sequence analysis*.
Here we describe lookahead pairs, their use for anomaly detection, and offer
a comparison with the more widely-known full sequence analysis.

In order to identify normal process behavior, profiles are constructed for
each executable on the system. On calls to `execve`, pH associates the correct profile
with a process and begins monitoring its system calls, modifying the lookahead pairs
associated with the testing data of a profile. Once enough normal samples have been gathered
and the profile has reached a specified maturity date, the process is then placed into training
mode wherein sequences of system calls are compared with the existing lookahead
pairs for a given profile.

Somayaji and Inoue [@soma07] contrasted full sequence analysis with lookahead pairs
and found that lookahead pairs produce fewer false positives than full sequences
and maintain this property even with very long window lengths.
This comes at the expense of potentially reduced sensitivity to some attacks
as well as more vulnerable to mimicry attacks. However, as part of their work,
Somayaji and Inoue showed that longer sequences can help mitigate these shortcomings
in practice [@soma07].

Both pH and ebpH use lookahead pair window sizes of 9, which has been shown to be
effective at both minimizing false positive rates and mitigating potential
mimicry attacks [@soma02]. This window size also carries the advantage that
lookahead pairs can be expressed with exactly 8 bits of information (one bit for every
previous position $i \in \{1..9\}$).

### Homeostasis Through System Call Delays

Perhaps the most unique aspect of pH's approach is the means by which it achieves the
eponymous concept of *process homeostasis*: system call delays.
Inspired by the biological process of the same name, pH attempts to maintain
homeostatic process behavior by injecting delays into system calls that are detected
as being anomalous [@soma02].

By scaling this response in proportion to the number of recent anomalies detected in a profile,
pH is able to effectively mitigate attacks while minimizing the impact of occasional false positives.
For example, a process that triggers several dozen anomalies will be slowed down to the point of
effectively stopping, while a process that triggers only one or two might only be delayed by a few seconds.
Admittedly, this relies upon the assumption of low burstiness for false positives. While this assumption generally
holds, Somayaji acknowledges in his dissertation [@soma02] that the possibility of attackers purposely provoking
pH into causing denial of service attacks is a potential problem. Additionally, users may become frustrated
with pH's refusal to allow otherwise legitimate behavior simply due to the fact that it has not yet been
observed.

In its current incarnation, ebpH does not yet delay system calls like its predecessor.
The primary reason for this gap in functionality is that a solution still needs to be
developed that works well with the eBPF paradigm; in particular, injecting delays via
eBPF tracepoints or probes seems untenable due to the verifier's refusal to accommodate
the code required for such an implementation. The addition of system call delays into
ebpH is currently a topic for future work (c.f. \autoref{response_automation}).

# Implementing ebpH

\label{sec:impl}

<!-- TODO: revise, starting here -->

At a high level, ebpH is an anomaly detection system that profiles executable behavior
by sequencing the system calls that processes make to the kernel; this essentially
serves as a reimplementation of the original pH system by Somayaji [@soma02].
What makes ebpH unique is its use of BPF programs for system call instrumentation
and profiling (in contrast to the original pH which was implemented as a Linux 2.2 kernel patch).
In this section, I will present the design and implementation of ebpH, with a particular
emphasis on the both challenges and benefits associated with an eBPF implementation and
ebpH's parallels with the original pH system.

## Why an eBPF Implementation?

In light of the various approaches presented in \autoref{background}, it is worth
comparing the approach taken by the original pH [@soma02] system with the new ebpH
prototype. In doing so, I will attempt to justify why an eBPF implementation of a system
like pH makes sense, and why such an implementation carries key advantages that would not
otherwise be tenable through traditional kernel-based implementations. To begin with,
let us compare the rough features of pH with ebpH; \autoref{ebph_comparison} provides
a rough framework for doing so.

\begin{table}
    \caption[Comparing the current prototype of ebpH with the original pH system]{
        Comparing the current prototype of ebpH with the original pH system.
    }
    \label{ebph_comparison}
    %\resizebox{\columnwidth}{!}{
    \begin{tabular}{>{\ttfamily}llcccccc}
        \toprule
        \multicolumn{1}{l}{\bfseries System} & {\bfseries Implementation} &
            \rotatebox{90}{Portable} & \rotatebox{90}{\parbox{2cm}{Production\\Safe}} &
            \rotatebox{90}{\parbox{2cm}{Low Mem.\\Overhead}} &
            \rotatebox{90}{\parbox{2cm}{Low Perf.\\Overhead}} &
            \rotatebox{90}{Detection} & \rotatebox{90}{Response} \\
        \midrule
        pH \cite{soma02} & Kernel Patch
            & \xmark & \xmark & \cmark & \cmark & \cmark & \cmark\\
        ebpH             & eBPF + Userspace Daemon
            & \cmark & \cmark & \xmark & \cmark & \cmark & \xmark \\
        \bottomrule
    \end{tabular}
    %}
\end{table}

As discussed in previous sections (see \autoref{ebpf-superpowers} and \autoref{data_collection_survey}),
eBPF offers several unique advantages over traditional solutions, particularly with respect to
intrusion detection data collection. eBPF can match the scope [@bcc; @gregg19bpf] and speed [@gebai18]
of kernel-based implementations while providing safety guarantees that previously would not have been possible.
Whereas before there existed an implicit trade-off between production safety on one hand and
scope and efficiency on the other, now eBPF marries these three properties under one paradigm.
Furthermore, eBPF's forward-compatibility ensures that new versions of the kernel will not break old code,
and that in general it is not necessary to upgrade to a new kernel version once one has access
to the minimum set of features required to compile and run a given BPF program. For instance,
since ebpH depends on Linux 5.3, all Linux kernel versions $\ge$ 5.3 will be able to support
ebpH's current set of features. This ensures perfect forward compatibility with production systems
and minimizes the impact of integrating ebpH into a production security stack.

The primary disadvantage of using eBPF is that BPF programs are necessarily
more limited in scope than kernel modules. That is not to say that BPF programs
cannot be complex, but rather that constructs that work well in kernel implementations
often need to be reworked for use in eBPF. A good example of this is the inability for
ebpH to issue system call delays in the same manner as the original pH. This is something
that remains a topic for future work, but there are alternative ways that it can be done,
for example the method using `bpf_signal` described in \autoref{response_automation}.

Since eBPF disallows global variables in the traditional sense, data storage and communication
between BPF programs needs to occur through the variety of maps. Further, limitations on memory
allocation and access restrict the dynamic allocation of data. To cope with these restrictions,
the current version of ebpH takes a less memory-efficient approach than its predecessor;
in particular, the sparse array of lookahead pairs is not dynamically allocated
--- instead, profiles themselves are dynamically allocated at runtime via a special hashmap. There are
plans to rework the way ebpH stores profile data to move it into separate *map-in-map* structures
that should allow a more memory-efficient approach to lookahead pair storage. This is outlined in
more detail in \autoref{lru_section}.

In summary, despite a few shortcomings of an eBPF implementation compared to a kernel implementation,
the benefits of portability and production safety in general outweigh these detractors. Additionally,
the problems with the current ebpH implementation *are* solvable in eBPF, and future versions of the
system should be significantly more memory efficient and offer the capability to respond
to attacks in real time, just as in the original pH [@soma02].

## Architectural Overview

ebpH can be thought of as a combination of several distinct components, functioning in both
userspace and kernelspace. In particular it includes a daemon and several CLI programs
(described in \autoref{userspace-components}) in userspace as well as several BPF programs in kernelspace
(described in \autoref{ebph-profiles} and onwards).
The architecture of ebpH is depicted in \autoref{ebph-dataflow}.

\begin{figure}
    \caption[The architecture of ebpH]{
        The architecture of ebpH. Note how the interaction between userspace programs
        and BPF programs is centered around the ebpH daemon. \code{ebph-admin} and \code{ebph-ps}
        are used to issue commands to and query info from the daemon, which interacts with the BPF
        programs on their behalf. The BPF programs instrument various kernel functions which are triggered
        by system calls from userspace.
    }
    \label{ebph-dataflow}
    \includegraphics[height=.4\textheight]{../figures/ebph-dataflow.png}
\end{figure}

ebpH's CLI programs interact with the daemon through a UNIX stream socket
which connects to the daemon's API. The daemon manages the BPF programs and
interacts with them through a combination of direct map access, perf event buffers,
and library calls, which are instrumented by uprobes in kernelspace. This combination
of techniques allows the daemon to lookup and modify data, poll for events, and
issue complex commands to ebpH's BPF programs. The BPF programs themselves are
used to instrument system calls along with a few other aspects of
the system such as signals and scheduler events. Subsequent sections
will cover these aspects of ebpH's design in further detail.

\FloatBarrier

## Userspace Components

The userspace components of ebpH are comprised of several distinct and related programs.
In particular, these programs can be divided into two sets: the ebpH daemon (`ebphd`) and several
CLI (command line interface) programs used to interact with it. The daemon is responsible for submitting BPF programs
to the kernel, managing their state, and providing an API to other userspace programs.
The CLI programs used to interact with the daemon include `ebph-ps`, used to list actively traced processes, threads, and profiles, providing
information about each, and `ebph-admin`, used to issue commands to the daemon and to check the status of the BPF program.
In order to issue more complex commands to the BPF program, `ebphd` leverages a userspace shared library, `libebph.so`
which provides functions that can be connected to arbitrary BPF programs via `uprobes`.
Earlier versions of ebpH also included a GUI, however the GUI needs to be refactored in order to work with ebpH's new
architecture and this is currently a topic for future work.

### The ebpH Daemon

The ebpH Daemon is implemented as a Python3 script that runs as a daemonized background process.
When started, the daemon uses bcc's Python front end [@bcc] to generate the BPF bytecode responsible for
tracing system calls, building profiles, and detecting anomalous behavior. It then submits this bytecode
to the verifier and JIT compiler for insertion into the eBPF virtual machine.

Once the eBPF program is running in the kernel, the daemon continuously polls a set of specialized BPF maps
called perf buffers which are updated on the occurrence of specific events.
\autoref{bpf-events} presents an overview of the most important events that ebpH cares about.
As events are consumed, they are handled by the daemon and removed from the buffer to make room for new events.
These buffers offer a lightweight and efficient method to transfer data from the eBPF program to userspace,
particularly since buffering data in this way significantly reduces the number of required context switches between
kernelspace and userspace.

In addition to perf buffers, the daemon is also able to communicate with the eBPF program through direct
access to its maps. We use this direct access to issue commands to the eBPF program, check program state,
and gather several statistics, such as profile count, anomaly count, and system call count. At the core of ebpH's
design philosophy is the combination of system visibility and security, and so providing as much information
as possible about system state is of paramount importance.

The daemon also uses direct map access to save and load profiles to and from the disk.
Profiles are saved automatically at regular intervals, configurable by the user,
as well as any time ebpH stops monitoring the system.
These profiles are automatically loaded every time ebpH starts.

<!-- FIXME later: this table footnote is showing up PAGES later... WTF?! -->
\begingroup
\small
\begin{longtable}{>{\ttfamily}lp{2.6in}l}
    \caption{Main perf event categories in ebpH.}
    \label{bpf-events}\\
    \toprule
    \multicolumn{1}{l}{Event} & Description & Memory Overhead\footnotemark{ }\\
    \midrule
    \endfirsthead
    \endhead
    ebpH\_on\_executable\_processed & Reports when a profile has been created & $2^{8} \text{ pages}$\\
    ebpH\_on\_anomaly & Reports anomalies in specific processes and which profile they were associated with & $2^{8} \text{ pages}$\\
    ebpH\_on\_anomaly\_limit & Reports when a profile hits its anomaly limit & $2^{8} \text{ pages}$\\
    ebpH\_on\_tolerize\_limit & Reports when a process hits its tolerize limit & $2^{8} \text{ pages}$\\
    ebpH\_on\_start\_normal & Reports when a profile starts normal monitoring & $2^{8} \text{ pages}$\\
    ebpH\_on\_new\_sequence & Reports new sequences for logging (when enabled) & $2^{8} \text{ pages}$\\
    ebpH\_warning & Reports generic warnings & $2^{2} \text{ pages}$\\
    ebpH\_error & Reports generic errors & $2^{2} \text{ pages}$\\
    \bottomrule
\end{longtable}
\endgroup
\footnotetext{
    The majority of these values are subject to significant
    optimization in future iterations of ebpH. The $2^8$ value is a sensible default chosen by bcc. In practice, many of these events
    are infrequent enough that smaller buffer sizes would be sufficient.
}

\FloatBarrier

In order to facilitate communication with the daemon, `ebphd` exposes a UNIX domain stream socket
at `/var/run/ebph.sock`. This socket is owned by the superuser, `root`, and has permissions `600`
in order to prevent unauthorized processes from attempting to issue commands to the daemon.
The CLI applications, `ebph-ps` (c.f. \autoref{ebph_ps_sec}) and `ebph-admin` (c.f. \autoref{ebph_admin_sec}),
use this socket to send commands to and receive replies from the daemon.

### `ebph-ps`

\label{ebph_ps_sec}

`ebph-ps` is the most common tool that a system administrator will use to get a quick overview of
process state on their system with respect to ebpH profiles. When run with its default settings,
`ebph-ps` lists all currently monitored processes on the system with their PID, comm, current status
(e.g. training, frozen, or normal), total system call count, system calls since last modification,
anomaly count, and normal time. When the user invokes `ebph-ps`, it sends a JSON-encoded request
to the daemon via the `ebphd`'s UNIX domain stream socket. The daemon replies on that same
socket with a JSON-encoded list of processes or profiles. Users who are acquainted with the popular
`ps` command line utility will find `ebph-ps`'s interface quite familiar. \autoref{ebph_ps_out}
shows sample output from `ebph-ps` running on a system.

\lil[
    numbers=none,
    label={ebph_ps_out},
    caption={[Sample output from \texttt{ebph-ps}]
    Sample output from \texttt{ebph-ps}.},
    basicstyle={\ttfamily\scriptsize}]
{../code/sample_outputs/ebph-ps.out}

In addition to listing information per-process, `ebph-ps` can also show information
per-thread using an optional `-t` flag. This can be used to get an idea of the number
of tasks that ebpH is *actually* monitoring (since ebpH's view of a "process" is actually
an individual thread rather than the entire thread group).
Further, the `-p` flag can be specified to list all profiles on the system
instead of processes. This can be used to find duplicate profiles for pruning, find the key
associated with a given profile, or get an idea of the overall behavior of all
binaries on the system. \autoref{ebph_ps_p_out} shows a truncated example of listing
all profiles on a system using the `-p` flag.

\lil[
    numbers=none,
    label={ebph_ps_p_out},
    caption={[Sample output from \texttt{ebph-ps -p}]
    Sample output from \texttt{ebph-ps -p}.
    Note how the \texttt{PID} column has been replaced with the profile \texttt{KEY} and
    \texttt{ebph-ps} now lists each profile exactly once, regardless of whether
    the profile is currently running.},
    basicstyle={\ttfamily\scriptsize}]
{../code/sample_outputs/ebph-ps-p.out}

### `ebph-admin`

\label{ebph_admin_sec}

`ebph-admin` is responsible for issuing more complex commands to ebpH, as well as making
generic queries about ebpH's status. Status queries include information about whether ebpH
is currently monitoring, how many system calls it has observed, how many process and threads
are currently being monitored, and how many profiles are loaded in memory.

Complex commands are issued via `libebph.so`, a dynamic library written in C whose job it
is to expose functions that are then attached to the BPF program via uprobes. These uprobes
are restricted to only trace calls that originate from the daemon's own PID, which prevents
another binary from simply loading that library code and issuing unauthorized commands to the
BPF program. \autoref{ebph_admin_request} depicts the process of using `ebph-admin` to make
a request to the daemon. <!--\autoref{commands_section} discusses the various underlying
mechanisms used to support commands in further detail.-->

\begin{figure}
    \caption[Dataflow of a request from \texttt{ebph-admin}]{
        Dataflow of a request from \texttt{ebph-admin}.
        The program issues requests to the daemon, which then either
        directly accesses a map or triggers the execution of a uprobe
        BPF program with \texttt{libebph.so}, depending on the complexity
        of the request. Note that malicious applications cannot abuse \texttt{libebph.so}
        to issue their own commands --- only the daemon can do this.
    }
    \label{ebph_admin_request}
    \includegraphics[width=.8\textwidth]{../figures/ebph-admin.png}
\end{figure}

### ebpH Logs

The current iteration of ebpH uses log data to communicate events to the user.
The daemon logs all important events to log files located at `/var/log/ebpH` by default,
including detected anomalies with corresponding sequences, profile creation, and process normalization.
Event logging categories roughly correspond to the perf event buffers depicted in
\autoref{bpf-events} on page \pageref{bpf-events}, since the daemon writes a log message whenever
one of these events is observed.

Since the current version of ebpH does not include a GUI (although there are plans to reintroduce a GUI
in the future), logs must be frequently checked to keep track of system behavior and any detected
anomalies. As a temporary stopgap, scripts can watch the logfile on behalf of the user and
send more conspicuous notifications to the user. A few such scripts are included with ebpH;
for instance, `watch-anom.sh` watches for anomaly events in the logfile with `tail` and sends a push notification
to the user via the `notify-send` command.

Using a combination of scripts and manual log analysis, the user can gain a clear picture of how their
system is behaving and whether any anomalies have occurred in said behavior. When the ebpH GUI is
reintroduced in future iterations, it will be much easier to observe system behavior in detail.
Refer to \autoref{gui_section} for a description of future work with respect to ebpH's GUI.

## ebpH Profiles

\label{ebph_profiles}

In order to monitor process behavior, ebpH keeps track of a unique
profile (\autoref{ebph-profile-struct}) for each executable on the system.
It does this by maintaining a hashmap of profiles, hashed by a unique per-executable ID;
this ID is a 64-bit unsigned integer which is calculated as a unique combination
of filesystem device number and inode number:
\begin{align*}
\texttt{key} &= (\texttt{device number} << 32) + \texttt{inode number}
\end{align*}
where $<<$ is the left bitshift operation. In other words, ebpH takes the filesystem's
device ID in the upper 32 bits of our key, and the inode number in the lower 32 bits.
This method provides a simple and efficient way to uniquely map keys to profiles.

\begin{lstlisting}[language=c, label={ebph-profile-struct}, caption={A simplified
definition of the ebpH profile struct.}]
struct ebpH_profile_data
{
    u8 flags[SYSCALLS][SYSCALLS]; /* System call lookahead pairs */
    u64 last_mod_count; /* Syscalls since profile was last modified */
    u64 train_count;    /* Syscalls seen during training */
};

struct ebpH_profile
{
    struct ebpH_profile_data train; /* Training data */
    struct ebpH_profile_data test;  /* Testing data */
    u8 frozen;          /* Is the profile frozen? */
    u8 normal;          /* Is the profile normal? */
    u64 normal_time;    /* Minimum system time required for normalcy */
    u64 anomalies;      /* Number of anomalies in the profile */
    char comm[128];     /* Name of the executable file */
};
\end{lstlisting}

The profile itself is a C data structure that keeps track of information about the
executable as well as two copies of profile data, one for training, and one for
testing. This profile data consists of a sparse two dimensional array of lookahead pairs
[@soma02; @soma07] used to keep track of observed system call patterns. Each entry in this array consists of an
8-bit integer, with the $i^\text{th}$ bit corresponding to a previously observed
distance $i$ between the two calls. When ebpH observes this distance, it sets the corresponding bit
to `1` using a bitwise `OR` operation. Otherwise, it remains `0`.
Each profile maintains lookahead pairs for each possible pair of system calls, and these lookahead
pairs are checked against new sequences when a profile becomes normal.
\autoref{lookahead-ls} presents a sample (`read`, `close`) lookahead pair for the `ls` binary.

\FloatBarrier

\begin{figure}
    \caption[A sample (\lstinline{read}, \lstinline{close}) lookahead pair in the ebpH profile for \lstinline{ls}.]{
        A sample (\lstinline{read}, \lstinline{close}) lookahead pair in the ebpH profile for \lstinline{ls}.
        (a) shows the lookahead pair and (b) shows two relevant system call sequences, separated by several omitted calls.
        Note that the first three system calls in both the first and second sequence are responsible for
        the two least significant bits of the lookahead pair.
    }
    \label{lookahead-ls}
    \includegraphics[height=0.4\textheight]{../figures/lookahead-ls.png}
\end{figure}

Each process (c.f. \autoref{tracing-processes}) is associated with
exactly one profile at a time. Profile association is updated whenever
ebpH observes a process making a call to `execve`. Whenever a process makes
a system call, ebpH looks up its associated profile, and sets the appropriate
lookahead pairs according to the process' most recent system calls. This forms
the crux of how ebpH is able to monitor process behavior.

Just like in the original pH [@soma02], profile state is tracked using the `frozen` and `normal` fields.
When a profile's behavior has stabilized, it is marked frozen. If a profile
has been frozen for one week (i.e. system time has reached `normal_time`),
the profile is then marked normal. Profiles are unfrozen when new behavior is
observed and anomalies are only flagged in normal profiles.

### Writing Profiles to Disk and Reading Profiles from Disk

\label{writing_to_disk}

In order to allow profile data to persist across machine reboots, ebpH periodically
writes profile data to disk, at an interval configurable the user, as well as when
the BPF program is unloaded by the user. Profiles are read from disk when ebpH first loads.

In the original pH, profile data was saved and loaded in kernelspace [@soma02] which meant that
it required kernelspace file I/O, which is often regarded as an unsafe practice. ebpH solves
this problem by moving all file I/O operations into userspace. This is made possible due to the
bidirectional nature of dataflow with respect to eBPF maps. Specifically, when writing to disk, the daemon
queries profile data from each entry in the profile map and writes that data to a file (`/var/lib/ebpH/<profile_key>`).
When reading from disk, the daemon reads profile data from the appropriate files (`/var/lib/ebpH/<profile_key>`)
and associates that data with keys in the newly created profile map.

## Tracing Processes

\label{ebph_processes}

Like profiles, process information is also tracked through a global hashmap of process
structs. The process struct's primary purpose is to maintain the association between
a process and its profile, maintain a sequence of system calls, and keep track of various metadata.
See \autoref{ebph-process-struct} for a simplified definition of the ebpH process struct.

\begin{lstlisting}[language=c, label={ebph-process-struct}, caption={A simplified
definition of the ebpH process struct.}]
struct ebpH_sequence
{
    long seq[9];      /* Remember 9 most recent system calls in order */
    u8 count;         /* How many system calls are in our sequence? */
};

struct ebpH_sequence_stack
{
    ebpH_sequence[3]; /* Keep track of up to 3 sequences at a time */
    int top;          /* Top of the sequence stack, values from 0-2 */
    int should_pop;   /* Pop from the stack on next system call */
};

struct ebpH_process
{
    struct ebpH_sequence_stack;
    u32 pid;          /* Kernel tgid */
    u32 tid;          /* Kernel pid */
    u64 profile_key;  /* Associated profile key */
    u8 in_execve;     /* Are we in the middle of an execve? */
};
\end{lstlisting}

ebpH monitors process behavior by instrumenting tracepoints all system calls.
On every system call return, ebpH adds the corresponding system call number to the process'
current sequence (ebpH actually maintains a *stack* of sequences in order to handle non-deterministic
behavior; this will be covered in more detail shortly). This sequence is subsequently used to index
into the corresponding profile's lookahead pairs and flip the correct bits. If the process' profile is normal,
new sequences will trigger ebpH's anomaly detection mechanism and a warning will be sent to userspace.

In addition to the system call tracepoints described above, ebpH also instruments a few other tracepoints to
keep track of profile creation, process creation and deletion, and profile association on `exec`-family system calls.
The `sched` class of tracepoints exposes hooks on the necessary system functionality in order to do this. Additionally, ebpH defines
one kprobe in order to detect when a process invokes its signal handler. \autoref{ebph-tracepoints} summarizes the
tracepoints and kprobes used by ebpH along with their side effects on ebpH's state.

\begin{table}
    \caption{eBPF tracepoints and kprobes used in ebpH.}
    \label{ebph-tracepoints}
    \begin{tabular}{>{\ttfamily}lp{2.4in}p{2.4in}}
        \toprule
        \multicolumn{1}{l}{Tracepoint/Kprobe} & Description & ebpH Side Effect\\
        \midrule
        sys\_enter & Tracepoint invoked just after system call entry &
            Check for return from a signal handler and pop from sequence stack if necessary\\
        sys\_exit & Tracepoint invoked just before system call return &
            Operate on per-process system call sequences and per-profile lookahead pairs\\
        sched\_process\_fork & Tracepoint invoked just after a call to \texttt{fork(2)},
            \texttt{vfork(2)}, or \texttt{clone(2)} &
            Create a new process struct and add it to the hashmap\\
        sched\_process\_exec & Tracepoint invoked just after an \texttt{exec}-family system call&
            Create a profile if necessary, adding it to the hashmap, and associate it with a process\\
        sched\_process\_exit & Tracepoint invoked just after a thread exits &
            Delete a process struct from the hashmap\\
        get\_signal & Kretprobe invoked when a process' signal handler is about to be called &
            Push a new frame onto the process' sequence stack\\
        \bottomrule
    \end{tabular}
\end{table}

### Profile Creation and Association

There are several important considerations here. First, ebpH needs a way to assign profiles
to processes, which is done by instrumenting the result of an `execve(2)` system call using the
`sched_process_exec` tracepoint. This tracepoint allows us to access information provide by the
`linux_bin_prm` struct, which is used to store information about the executable or interpreted script
that `execve(2)` has loaded. In particular, the executables inode and filesystem device number are used
in combination to compute a key that uniquely maps to an individual executable
on disk. Without this, ebpH would be unable to differentiate between two paths that
resolve to a binary with the same name, for example `/usr/bin/ls` and `./ls`; this is due to
an unfortunate nuance in `execve(2)`'s treatment of pathnames (i.e. it only considers relative paths
when provided in order to save on memory).

In addition to associating a process with the correct profile, ebpH also wipes
the process' current sequence of system calls, to ensure that there is no carryover
between two unrelated profiles when constructing their lookahead pairs. This is important
in order to prevent `execve(2)` calls from being used to construct artificially good sequences
in a profile which may be later used to mask malicious behavior [@soma02].

### Profile Association and Sequence Duplication

Another special consideration is with respect to `fork(2)` and `clone(2)` family system calls.
A forked process should begin with the same state as its parent and should (at least initially)
be associated with the same profile as its parent. A subsequent `execve(2)` (i.e. the `fork`-`execve` pattern)
would then overwrite this association.
In order to accomplish this, ebpH instruments tracepoints for the `fork(2)`, `vfork(2)`, and `clone(2)`
system calls, ensuring that it associates the child process with the parent's profile, if such a profile
exists. If ebpH detects an `execve(2)` as outlined above, it will simply overwrite the initial
profile association provided by the fork. The parent's current system call sequence
is also copied to the child to prevent forks from being used to break sequences.

### Dealing with Signal Handlers and Non-Determinism

\label{non_determinism}

As an anomaly-based intrusion detection system, it is critical that ebpH be able to establish
normal profiles of program behavior in a timely manner. As presented in previous sections,
establishing the normalcy of a profile requires that the it has been active for at least
a week and that the ratio of total system calls seen during training to system calls the last
time the profile was modified be sufficiently large. As a corollary, every time a process makes
a system call that results in a previously unobserved sequence, this ratio becomes increasingly
difficult to achieve. In practice, this means that it is much harder to normalize profiles that
exhibit less deterministic behavior. As a practical example, consider the time required to stabilize
a relatively simple binary, such as `ls` versus a complex web browser like `firefox`; not only does `firefox`
make significantly more system calls during an average run, but it is also far more likely to produce
a previously unseen sequence at any given time.

This problem of normalizing profiles is compounded by the non-deterministic behavior introduced by
signals and signal handlers. This phenomenon was first noted by Amai et al. [@amai05] in a 2005 technical
paper on the original pH system. In particular, they noted that signal handlers were a significant
source of non-deterministic behavior in processes that ultimately led to significantly longer
wait times until profile normalcy. This effect is not difficult to see in practice, especially in the context
of complex programs that run for extended periods of time, such as the above `firefox` example.
Suppose that we have some sequence of system calls $(A, B, C, D, E)$ and a signal handler that
invokes system calls $(F, G, H)$; depending on when this signal is caught during the initial sequence,
the resulting sequence can vary significantly. For example, we might see $(A, F, G, H, B, C, D, E)$
in one instance and $(A, B, C, D, F, G, H, E)$ in another. This results in a significant deterioration in profile
stability, and subsequently in profile normalcy times.

ebpH deals with the problem of signal handlers in the same manner proposed by Amai et al. [@amai05].
Specifically, it maintains a stack of system call sequences in each process struct; each time a traced process receives
a signal, ebpH pushes a frame onto this stack, and when the process exits from its signal handler, ebpH pops the frame.
This has the effect of temporarily wiping ebpH's memory of a process' current system call sequence
whenever it enters a signal handler, allowing subsequent lookahead pairs to be unaffected by
the execution context prior to the handler's invocation and vice versa. In order to decide when to push, ebpH
instruments a kprobe on the kernel's `get_signal` implementation; this allows it to detect when a process
receives a signal that will be handled. Subsequently, ebpH detects a return from a signal handler by checking for
the `rt_sigreturn` system call; when ebpH detects such a return, it pops the top frame from the sequence stack.

### Reaping Processes

ebpH reaps tasks from its process map whenever detects that they have exited.
By reaping process structs from our map as ebpH is finished with them, it is able to ensure that the map
neither fills up, nor does it consume more memory than necessary. In order to detect when a task exits,
ebpH instruments the `sched_process_exit` tracepoint provided by the kernel's trace API.
This tracepoint is triggered whenever the scheduler handles the termination of a task.
Within the BPF program associated with the tracepoint, ebpH simply determines the task's
PID and deletes that key from the process map.

## Training, Testing, and Anomaly Detection

ebpH profiles are tracked in two phases, *training mode* and *testing mode*.
Profile data is considered training data until the profile becomes normal (as described in \autoref{ebph-profiles}).
Once a profile is in testing mode, the lookahead pairs generated by its associated processes
are compared with existing data. When mismatches occur, they are flagged as anomalies which are reported
to userspace via a perf event buffer. The detection of an anomaly also prompts ebpH
to remove the profile's normal flag and return it to training mode.

### A Simple Example of ebpH Anomaly Detection

As an example, consider the simple program shown in \autoref{anomaly.c}.
This program's normal behavior is to simply print a message to the terminal.
However, when issued an extra argument (in practice, this could be a secret keyword
for activating a backdoor), it prints one extra message. This will cause a noticeable
change in the lookahead pairs associated with the program's profile, and this will be flagged
by ebpH if the profile has been previously marked normal.

\protect\enlargethispage*{\baselineskip}

\lil[language=c, label={anomaly.c}, caption={\texttt{anomaly.c}, a simple program to demonstrate anomaly detection in ebpH.}]{../code/anomaly.c}

In order to test this, I artificially lower ebpH's normal time to three seconds instead of one week.
Then, I run the above test program several times with no arguments to establish normal behavior. Once the profile
has been marked as normal, I then run the same test program with an argument to produce the anomaly.
ebpH immediately detects the anomalous system calls and flags them.
These anomalies are then reported to userspace via a perf buffer as shown in \autoref{anomaly-flag}.

\begin{lstlisting}[label={anomaly-flag}, numbers=none, caption={[The flagged anomaly in the \texttt{anomaly}
binary as shown in the ebpH logs]The flagged anomaly in the \texttt{anomaly}
binary as shown in the ebpH logs. Note that ebpH also logs the offending sequence, reordering it
so that most recent system calls appear on the right.}, language=none]
WARNING: Anomalies in PID 11162 (anomaly 38803844):
    MPROTECT, MPROTECT, MPROTECT, MUNMAP, FSTAT, BRK, BRK, WRITE, WRITE
\end{lstlisting}

From here, one can figure out exactly what went wrong by inspecting the system call sequences produced by the `anomaly`
program, in both cases and comparing them with their respective lookahead pair patterns.
\autoref{anomaly-lookahead-comp} provides an example of this comparison.

\begin{figure}[p]
\includegraphics{../figures/lookahead-anomaly.png}
\caption[Two sample (\lstinline{write}, \lstinline{write}) lookahead pairs in the ebpH profile for \texttt{anomaly.c}]{
    Two sample (\lstinline{write}, \lstinline{write}) lookahead pairs in the ebpH profile for \texttt{anomaly.c}.
    (a) shows the lookahead pair and (b) shows two relevant system call sequences. The left hand side depicts normal program
    behavior, while the right hand side depicts our artificially generated anomaly.
    There are several other anomalous lookahead pairs which result from this extra write call, but we focus
    on (\lstinline{write}, \lstinline{write}) for simplicity.
}
\label{anomaly-lookahead-comp}
\end{figure}

While this contrived example is useful for demonstrating ebpH's anomaly detection,
process behavior in practice is often more nuanced. ebpH collects at least a week's worth of
data about a process' system calls before marking it normal, which often corresponds with several branches of execution.
In a real example, the multiple consecutive write calls might be a perfectly normal execution path for this process;
by ensuring that ebpH takes its time before deciding whether a process' profile has reached acceptable maturity
for testing, it decreases the probability of any false positives.

<!-- ## The `ebphd` Commands API -->

<!-- \label{commands_section} -->

<!-- ### Setting Runtime Parameters -->

<!-- ### Examining Profiles and Processes -->

<!-- ### Issuing More Complex Commands -->

## Soothing the Verifier

The development of ebpH elicited many challenges with respect to the eBPF verifier.
As seen in \autoref{verifier-section}, eBPF programs become more difficult to verify
as they increase in complexity; as a corollary, when developing large and complex
eBPF programs, a great deal of care and attention must be paid to ensure that the verifier
will not reject the code.

The problem of dealing with the eBPF verifier can be expressed in
the form of several subproblems as follows:

1) Many kernel functions and programming constructs are prohibited in eBPF;
2) eBPF programs come with a hard stack space limit of 512 bytes;
3) Traditional C-style dynamic memory allocation is prohibited;
4) Support for bounded loops is in its infancy and such loops must be carefully constructed to avoid verifier issues;
5) The verifier tends to err on the side of caution and will produce false positives with non-negligible frequency.

Subproblem (1) poses a particular challenge for a few aspects of ebpH's design: namely, profile keying and storage,
`execve(2)` abortion, and issuing system call delays. This is due to the fact that eBPF programs do not have
access to many of the helper functions available for traditional kernel development. As such, profile keying
and storage have been fundamentally changed in ebpH compared to its predecessor, and `execve(2)` abortion and
system call delays have been left as topics for future work (see \autoref{response_automation}). The original
pH [@soma02] stored profiles as a linked list and indexed them using executable pathnames. Unfortunately,
the kernel helper required to build pathnames from a `dentry` is not available in eBPF and, while a partial
solution has been submitted for review in the kernel upstream [@zhang19], this will likely not be merged into the mainline
until a much later version of Linux. As befits the BPF paradigm, ebpH stores its profiles in a global
hashmap instead of a linked list and indexes this hashmap by a uniquely computed integer based on executable metadata,
namely its inode and filesystem device number. Profiles are then augmented with the executables *filename*\footnote{This is
not the same thing as a \emph{pathname}.} for usability purposes.

From subproblems (2) and (3), one immediate issue arises: with no means of explicit dynamic memory allocation and
a stack space limit of 512 bytes, ebpH needs an alternative method of instantiating the relatively large data
structures described in \autoref{ebph_profiles} and \autoref{ebph_processes}, as both the `ebpH_profile` and
`ebpH_process` structs are larger than would be allowed in the eBPF stack. Fortunately, a creative solution exists
this problem which leverages the `BPF_ARRAY`'s unique property of zero-initializing elements on creation. What this means
is that ebpH can maintain a size `1` array for each data structure it requires; when it needs to instantiate a struct,
all it needs to do is look up this value from the array, and copy it into the corresponding global hashmap.
Fortunately, we can creatively solve this problem by using a `BPF_ARRAY` for initialization. This technique constitutes the
design pattern outlined in \autoref{appendix-bigdata} of \autoref{ebpf-design-patterns}.

In order to prevent ebpH's maps from consuming all of the system's memory, they are flagged with
`BPF_F_NO_PREALLOC` which notifies the kernel that these maps should be dynamically allocated
at runtime as opposed to statically allocated at load time. While this lessens the burden on the system,
it is not an ideal solution. There are known issues with dynamically allocated maps [@starovoitov16prealloc]
which may cause deadlocks when used in certain high-volume tracing events such as kernel spinlock counters.
For ebpH's prototype, this trade-off in reliability is acceptable, but future versions will be refactored
to make use of a combination of `LRU_HASH` for low memory overhead and `HASH_OF_MAPS` for lookahead pair storage.
This will provide a more reliable and more memory efficient approach than the current dynamic hashmap allocation.
\autoref{lru_section} discusses this future refactor in more detail.

From subproblem (4), the obvious issue arises that loops need to be "simple" enough for the eBPF
verifier to reason about them. For example, loops that have entrypoints in the middle of
iteration will potentially be flagged if the verifier is unable
to correctly identify the loop structure [@corbet18]. Since the verifier relies
on pattern matching in order to identify induction variables, LLVM optimizations
to eBPF bytecode introduce an element of fragility to loop verification [@corbet18].
Bounded loops that perform memory access using the induction variable are also quite finicky at best;
the verifier must be able to show that memory access is safe in all possible states --- this precludes induction
variables from having an unsafe lower *or* upper bound when they are used to index into a buffer.
These limitations affect ebpH and its design in non-trivial ways; for example, ebpH requires specially crafted
helper functions to perform simple operations such as indexing into the array of lookahead pairs or per-process
system call sequences. These helpers perform extra checks on the bounds of the variable being used
to index into the array and are designed to handle failure gracefully. Additionally, special compiler macros
are employed to ensure that LLVM optimizations do not affect their integrity.

Subproblem (5) is perhaps the most difficult to reckon with, but is quite understandable when
considering the gravity and difficulty of the problem that the verifier is trying to solve.
As shown in \autoref{verifier-section}, guaranteeing the safety of arbitrary untrusted programs
is a difficult problem and concessions need to be made in order for such guarantees
to be tenable. False positives are unfortunately one of those concessions. When the verifier rejects
code due to a false positive, there is simply no better solution than to try a different approach.
ebpH triggered many false positives during its development which required significant refactoring of
otherwise reasonable code. While these verifier false positives are unfortunate, they are a far cry
from the vexing kernel panics, data corruptions, and other crashes that so often occur during ordinary
kernel development --- ebpH crashed the system precisely *zero* times during testing *and* development.
This extraordinary feat is made possible by eBPF's safety guarantees.

## Dealing a Lack of Concurrency Control Mechanisms

\label{impl_no_conc}

Due to a lack of sufficient preemption checks in the verifier [@verifier_git; @bpf_h_git], tracing
programs are currently forbidden from using the `bpf_spin_lock` primitive included in kernel 5.1.
This means that ebpH has no means of managing concurrency within its data structures in the traditional sense,
and there no immediately obvious way of guaranteeing that modifications to profiles are consistent.
Notably, the map used to keep track of processes does not suffer from this problem, as each `ebpH_process`
data structure keeps track of its own thread.

Since profiles may be modified by multiple processes at once, it is important that these
modifications be kept relatively synchronized to avoid mismatches. One useful aspect of ebpH's
design is that entries within lookahead pairs are tracked with bits, which means that each entry is
either `1` or `0` at a given time, and that entries are set using a bitwise `OR` operation.
Since a `x | 1` is always equal to `1`, the operation of setting a lookahead pair is actually immune to
concurrency-related issues. Similarly, lookahead pairs are only checked for anomalies once a profile has been frozen,
and this check is done on a separate copy of the training data from the one being modified. This means that
the operation of checking a lookahead pair is also safe.

On the other hand, ebpH profiles have several flags and counters that need to be kept
synchronized in order for them to work properly. Although locking in the traditional sense is impossible,
eBPF *does* have atomic add and subtract instructions [@bpf_h_git] that combine the operation of checking a value
with the operation of incrementing or decrementing it. ebpH uses these to keep its flags and counters
at least semi-consistent with what is expected. Although this is not a perfect solution, it is sufficient
as a temporary stopgap until a better alternative is made available. \autoref{no_concurrency} further
discusses the need for concurrency control mechanisms in eBPF.

# Measuring ebpH's Overhead

\label{measuring_overhead_section}

One of the primary advantages of eBPF is its relatively low overhead [@gregg19bpf; @starovoitov13; @starovoitov14]
compared to many other system introspection solutions (c.f. \autoref{tracing-section} and \autoref{ebpf-superpowers}).
In order to justify this claim in the context of an eBPF intrusion detection system,
it is necessary to ascertain the overhead associated with running ebpH on a variety of
systems under a variety of workloads (artificial and otherwise). Here I
describe the tests that were conducted in order to determine this overhead.
\autoref{methodology-section} outlines the systems and tools used for testing and
provides an overview of the collected datasets.
The specifics of each benchmarking test along with the results are provided in \autoref{results-section}.

## Methodology

\label{methodology-section}

The experimental methodology used to determine ebpH's performance overhead includes both
macro and micro-benchmarks, to establish both real-world behavior and highly controlled experimental
results respectively. Benchmarks were primarily concerned with ebpH's overhead on system calls, although
other factors were considered in the micro-benchmark tests, such as signal handler overhead, IPC (interprocess-communication),
and process creation latency. Macro-benchmarking data was collected on various
systems under various workloads, including: a server used in production;
a personal computer; and a CCSL (Carleton Computer Security Lab) workstation. Micro-benchmarking
data was collected on the CCSL workstation only under an otherwise idle workload, in order to prevent
corruption of results by outside factors.
\autoref{systems} summarizes each of the systems used for the collection of benchmarking data,
including relevant hardware specifications.

\begin{table}
    \caption[Systems used for the collection of ebpH benchmarking data]
    {
        Systems used for the collection of ebpH benchmarking data.
    }
    \label{systems}
    \begin{tabular}{>{\ttfamily}llll}
        \toprule
        \multicolumn{1}{l}{System} & Description & \multicolumn{2}{l}{Specifications}\\
        \midrule
        \multirow{4}{*}{arch} & \multirow{4}{*}{Personal workstation}
            & Kernel & 5.5.10-arch1-1\\
            & & CPU  & Intel i7-7700K (8) @ 4.500GHz\\
            & & GPU  & NVIDIA GeForce GTX 1070\\
            & & RAM  & 16GB DDR4 3000MT/s\\
            & & Disk & 1TB Samsung NVMe M.2 SSD\\
        \hline
        \multirow{4}{*}{bronte} & \multirow{4}{*}{CCSL workstation}
            & Kernel & 5.3.0-42-generic\\
            & & CPU  & AMD Ryzen 7 1700 (16) @ 3.000GHz\\
            & & GPU  & AMD Radeon RX\\
            & & RAM  & 32GB DDR4 1200MT/s\\
            & & Disk & 250GB Samsung SATA SSD 850\\
        \hline
        \multirow{4}{*}{homeostasis} & \multirow{4}{*}{Mediawiki server}
            & Kernel & 5.3.0-42-generic\\
            & & CPU  & Intel i7-3615QM (8) @ 2.300GHz\\
            & & GPU  & Integrated\\
            & & RAM  & 16GB DDR3 1600MT/s\\
            & & Disk & 500GB Crucial CT525MX3\\
        \bottomrule
    \end{tabular}
\end{table}

### `lmbench` Micro-Benchmark

<!--
In addition to system call overhead, we are also interested in how ebpH affects overall system performance.
In particular, ebpH should have negligible impact on normal system use. In order to ascertain this, micro-benchmarking
data was collected for a variety of test cases. \autoref{micro-datasets} provides a description of each micro-benchmark dataset,
including the system and the workload tested. Additional details of each micro-benchmark test are provided in their respective
results sections.
-->

McVoy's `lmbench` [@lmbench; @lmbenchgit] is a Linux micro-benchmarking suite that has seen
prominent use in academia [@lmbenchex1; @lmbenchex2; @lmbenchex3; @lmbenchex4] for establishing
various performance metrics of UNIX-like systems. The *OS-category* benchmarks in `lmbench` are most
relevant to ebpH's overhead.This category provides performance metrics such as:

- Simple system call latency (c.f. \autoref{bronte_lmbench_syscall} and \autoref{bronte_lmbench_syscall_graph});
- `select(2)` latency on various file types (c.f. \autoref{bronte_lmbench_select} and \autoref{bronte_lmbench_select_graph});
- Signal handler latency (c.f. \autoref{bronte_lmbench_signal} and \autoref{bronte_lmbench_signal_graph});
- Dynamic process creation latency (c.f. \autoref{bronte_lmbench_process} and \autoref{bronte_lmbench_process_graph});
- IPC (inter-process communication) latency for pipes and UNIX stream sockets (c.f. \autoref{bronte_lmbench_ipc}
and \autoref{bronte_lmbench_ipc_graph}).

Simple system call and `select(2)` latency will give an idea of how ebpH affects system call overhead directly, while
signal handler latency will show the overhead caused by both ebpH's treatment of the underlying system calls
as well as the signal-aware stack discussed in \autoref{non_determinism}. Finally, the process creation and IPC
latency metrics will provide a better picture of ebpH's overhead in a more practical context.

<!--
\begin{table}
    \caption[ebpH micro-benchmarking datasets]{
        ebpH micro-benchmarking datasets.
    }
    \label{micro-datasets}
    \begin{tabular}{>{\ttfamily}l>{\ttfamily}llp{3in}}
    \toprule
    \multicolumn{1}{l}{Dataset} & \multicolumn{1}{l}{System} & Workload & Description \\
    \midrule
        bronte-lmbench & bronte & Artificial & OS micro-benchmarks from the
            \texttt{lmbench} \cite{lmbench, lmbenchgit} suite, with and without ebpH \\
        arch-x11perf & arch & Artificial & \texttt{x11perf} \cite{x11perf, x11perfcomp} full benchmarking suite, with and without ebpH \\
    \bottomrule
    \end{tabular}
\end{table}
-->

### Kernel Compilation Micro-Benchmark

While the contrived tests presented by `lmbench` provide a reliable and widely accepted
overview of performance characteristics, they are not necessarily representative of
ebpH's impact in practice. In order to get an idea of ebpH's impact on resource-intensive
computational operations, I elected to include a kernel compilation performance micro-benchmark.
This also has the nice side effect of mirroring a similar test conducted on the original pH system,
which will aid in later comparison of the two (c.f. \autoref{sec:compare}).

This benchmark consisted of timing the compilation of the Linux 5.6 kernel on `bronte`, using
all 16 of its logical cores. Five trials were run with ebpH enabled and five trials were run without.
Times were measured using `bash`'s `time` command and aggregated with `awk`. The full shell script
used to run these tests is available in \autoref{appendix:kernel}.

### `bpfbench` Macro-Benchmarks

Since ebpH's kernelspace functionality resides in system call hooks, much of its imposed overhead on the system
can be established by running macro-benchmarks on the time required to make system calls.
The data collected here will augment the selected system call data from the `lmbench` micro-benchmarks
by increasing its scope and placing it within the context of realistic system workloads.
Initially, I planned to use `syscount` [@syscount] from `bcc-tools` for this purpose,
however this tool currently has a race condition that may affect results due to its use of
`BPF_HASH` rather than `BPF_PERCPU_ARRAY` for data storage (c.f. \autoref{ebpf-maps} on page \pageref{ebpf-maps}).
Instead, an ad-hoc benchmarking tool, `bpfbench`\footnote{Full source code available at
\url{https://github.com/willfindlay/bpfbench}.}, was written in eBPF for this purpose. Like \texttt{syscount},
`bpfbench` measures system call overhead by taking the difference of `ktime` (in nanoseconds)
between system call entry and return; this difference along with the number of calls
observed is stored in an eBPF map for later analysis. Unlike `syscount`, `bpfbench`
stores this data in a `PERCPU_ARRAY`, aggregating data at the end when necessary; this means
that neither the system call count nor the system call overhead is subject to race conditions
like its predecessor. See \autoref{bpfbench} for the BPF portion of `bpfbench`'s source code.

Tests were run under a variety of workloads and benchmarking data was collected using `bpfbench`.
For each dataset, the same test was conducted on the system twice: once with ebpH running, and once without.
All ebpH data was collected while ebpH was monitoring the entire system (i.e. started immediately on
boot via a `systemd` unit) and running with normal parameters and logging settings.
\autoref{macro-datasets} provides a description of each dataset, including the system and the workload
tested.

\begin{table}
    \caption[ebpH macro-benchmarking datasets]{
        ebpH macro-benchmarking datasets.
    }
    \label{macro-datasets}
    \begin{tabular}{>{\ttfamily}l>{\ttfamily}llp{2.3in}}
        \toprule
        \multicolumn{1}{l}{Dataset} & \multicolumn{1}{l}{System} & Workload & Description \\
        \midrule
        bronte-7day & bronte & Idle & \texttt{bpfbench}, 7 days with ebpH and 7 days without \\
        homeostasis-3day & homeostasis & Production & \texttt{bpfbench}, 3 days with ebpH and 3 days without \\
        arch-3day & arch & Normal use & \texttt{bpfbench}, 3 days with ebpH and 3 days without \\
        \bottomrule
    \end{tabular}
\end{table}

\FloatBarrier

After benchmarking data was collected, overhead was calculated according to the following equation:

\begin{align*}
    \text{Overhead}_\text{syscall} &= \frac{T_{\text{ebph}_{\text{syscall}}}
    - T_{\text{base}_{\text{syscall}}}} {T_{\text{base}_{\text{syscall}}}}
    \intertext{where,}
    T_{\text{syscall}} &= \frac{\text{Total time}}{\text{Number of occurrences}}
\end{align*}
as measured by \texttt{bpfbench}.

<!--
Since system call runtime is not necessarily deterministic depending on implementation,
some of the results from the macro-benchmarks were pathological. I define a pathological result
as any result that showed a significant performance improvement when running ebpH over
running the system normally. There can be many causes for such disparity; for instance
system calls with a high variance in the amount of "thinking" required in kernelspace
based on input, or system calls that block until a certain condition is true on a given
system resource.
-->

Many system calls in Linux are designed to wait and return when some property becomes true on a system resource;
such calls are referred to as *blocking system calls*. Since many blocking system calls introduce a high
degree of variance in results, they have been pruned from the results presented here. In particular, any
system call with a standard deviation in runtime higher than 10 microseconds has been removed from the presented
results. This is an effective heuristic for weeding out blocking system calls that could impact the integrity of the
dataset while preserving those that have acceptable impact on results. Full, unadulterated results
are provided in \autoref{appendix_datasets}.

## Results

\label{results-section}

This section presents the results of all benchmarks. Micro-benchmarking results will be presented
first in order to provide a more statistically significant depiction of ebpH's overhead, followed by macro-benchmarking
data collected with `bpfbench` to cover ebpH's behavior in production environments. Macro-benchmark results
have been trimmed for brevity, and pruned for outliers and results with unacceptably high variance. As mentioned,
full datasets are available in \autoref{appendix_datasets}.

### `bronte-lmbench` System Latency Micro-Benchmark

\label{bronte_lmbench}

1000 ebpH and 1000 non-ebpH OS-category trials were run on `bronte`, a workstation in the CCSL (Carleton Computer Security Lab)
at Carleton University. The results were then averaged and compared to determine overhead.

<!-- syscalls -->
\begin{table}
    \caption[Results of the system call benchmarks from the \code{bronte-lmbench} dataset]{
        Results of the system call benchmarks from the \code{bronte-lmbench} dataset.
        Standard deviations are given in parentheses and smaller overhead is better. Note that the \code{open/close}
        benchmark shows the times of \emph{both} system calls taken together, which explains why the difference
        between base and ebpH times is doubled. This was an unfortunate design choice by the developers of \code{lmbench}.
    }
    \label{bronte_lmbench_syscall}
    \input{../data/bench/bronte-lmbench/syscall_results.tex}
\end{table}

<!-- syscalls -->
\begin{figure}
    \caption[Mean system call times from the \code{bronte-lmbench} dataset]{
        Mean system call times from the \code{bronte-lmbench} dataset.
        Standard error is given as error bars.
        Smaller difference in times is better.
    }
    \label{bronte_lmbench_syscall_graph}
    \includegraphics[width=.6\textwidth]{../data/bench/bronte-lmbench/syscall_times.png}
\end{figure}

As shown in \autoref{bronte_lmbench_syscall}, ebpH adds non-negligible overhead to simple system calls.
However, this result is misleading, as the actual difference between base and ebpH times is less than
a microsecond (about one third of a microsecond to be more precise). As soon as base times for system calls
approach one microsecond, (e.g. in the case of `stat(2)`), overhead drops significantly. For extremely
short calls like `getppid(2)`, the overhead is just over $614\%$, which is representative of the
worst case, but longer system calls like `stat(2)`, overhead drops to about $66\%$. This overhead is
more representative of the general case.

<!-- select -->
\begin{table}[b!]
    \caption[Results of the \code{select(2)} benchmarks from the \code{bronte-lmbench} dataset]{
        Results of the \code{select(2)} benchmarks from the \code{bronte-lmbench} dataset.
        Standard deviations are given in parentheses and smaller overhead is better.
    }
    \label{bronte_lmbench_select}
    \input{../data/bench/bronte-lmbench/select_results.tex}
\end{table}

The `select(2)` system call benchmarks provide an idea of the overhead imposed on a blocking
system call in a controlled environment; the more time the kernel spends blocking, the smaller effect ebpH's
overhead has on system call runtime. `select(2)` [@man_select] is used to wait until one or more
file descriptors become available for a given operation; the `select(2)` benchmarks from `lmbench`
invoke this system call on predefined sets of file descriptors, shown in \autoref{bronte_lmbench_select}.
The results here demonstrate that the overhead imposed by ebpH rapidly diminishes as the duration spent blocking
increases, and in some cases drops below the standard third of a microsecond that was observed previously;
the likely explanation here is that the overhead incurred by ebpH is occurring during time that would otherwise
be spent blocking.

<!-- select -->
\begin{figure}
    \caption[Mean \code{select(2)} times from the \code{bronte-lmbench} dataset]{
        Mean \code{select(2)} times from the \code{bronte-lmbench} dataset.
        Standard error is given as error bars.
        Smaller difference in times is better.
    }
    \label{bronte_lmbench_select_graph}
    \includegraphics[width=.6\textwidth]{../data/bench/bronte-lmbench/select_times.png}
\end{figure}

\FloatBarrier

As discussed in \autoref{non_determinism}, ebpH makes use of special logic to separate the non-deterministic
behavior caused by signal handlers from other observed process behavior. \autoref{bronte_lmbench_signal}
shows that the overhead imposed on the execution of simple signal handlers is relatively low, around
39\%. This result is especially impressive considering that it includes the standard per-system-call overhead
(c.f. \autoref{bronte_lmbench_syscall}) imposed on `rt_sigreturn(2)` [@man_sigreturn],
which is invoked upon return from a signal handler.

<!-- signal -->
\begin{table}
    \caption[Results of the signal handler benchmarks from the \code{bronte-lmbench} dataset]{
        Results of the signal handler benchmarks from the \code{bronte-lmbench} dataset.
        "Installation" represents the registration of a signal handler with \code{rt_sigaction(2)} and
        "Handler" represents the time taken to complete a simple signal handler.
        Standard deviations are given in parentheses and smaller overhead is better.
    }
    \label{bronte_lmbench_signal}
    \input{../data/bench/bronte-lmbench/signal_results.tex}
\end{table}

<!-- signal -->
\begin{figure}
    \caption[Mean signal handler times from the \code{bronte-lmbench} dataset]{
        Mean signal handler times from the \code{bronte-lmbench} dataset.
        "Installation" represents the registration of a signal handler with \code{rt_sigaction(2)} and
        "Handler" represents the time taken to complete a simple signal handler.
        Standard error is given as error bars.
        Smaller difference in times is better.
    }
    \label{bronte_lmbench_signal_graph}
    \includegraphics[width=.6\textwidth]{../data/bench/bronte-lmbench/signal_times.png}
\end{figure}

\FloatBarrier

While the aforementioned benchmarking results have been informative with respect to the per-system-call
and per-signal overhead of ebpH, they neglect to provide an accurate depiction of what this overhead
might look like in practice. To that end, the dynamic process creation and IPC benchmarks offered
by `lmbench` present a much clearer picture of ebpH's practical overhead. \autoref{bronte_lmbench_process}
presents the overhead of running three distinct process creation C programs as follows:

- `fork+exit` forks\footnote{The current C standard library implementation of \code{fork(3)}
actually produces the \code{clone(2)} system call rather than \code{fork(2)}.} itself and the child immediately exits;
- `fork+execve` forks itself and immediately executes a simple "hello world" program in the child;
- `fork+/bin/sh -c` forks itself and spawns a shell which then invokes the same "hello world" program described above.
This roughly corresponds to the implementation of the C standard library's `system(3)` [@man_system] interface.

The above three methods of process creation each involve increasing degrees of complexity with respect
to their system calls and, as a corollary, the overhead caused by ebpH increases for each one. Stating with
`fork+exit`, \autoref{bronte_lmbench_process} shows that ebpH imposes very little overhead on basic process
creation, on the order of 5 microseconds, or about 2.7\%.

The `fork+execve` case introduces more overhead, due to the special operations that ebpH must perform
when a process first executes, such as looking up a binary's inode information and associating it with a profile
(creating this profile if it does not yet exist). While this operation is not free, it is inexpensive relative
to the existing overhead of an `execve(2)` system call and imposes a total performance overhead of just 8\%.

Finally, `fork+/bin/sh -c` imposes the most overhead of all three methods; this makes sense as it involves
*two* `execve(2)` calls, one for `/bin/sh` and one for the "hello world" program, as well as the additional
per-system-call overhead from `/bin/sh` itself. Still, the overhead for this method is only about 10\%,
which is acceptable in practice.

<!-- processes -->
\begin{table}
    \caption[Results of the process creation benchmarks from the \code{bronte-lmbench} dataset]{
        Results of the process creation benchmarks from the \code{bronte-lmbench} dataset.
        Standard deviations are given in parentheses and smaller overhead is better.
    }
    \label{bronte_lmbench_process}
    \input{../data/bench/bronte-lmbench/process_results.tex}
\end{table}

<!-- processes -->
\begin{figure}
    \caption[Mean process creation times from the \code{bronte-lmbench} dataset]{
        Mean process creation times from the \code{bronte-lmbench} dataset.
        Standard error is given as error bars.
        Smaller difference in times is better.
    }
    \label{bronte_lmbench_process_graph}
    \includegraphics[width=.6\textwidth]{../data/bench/bronte-lmbench/process_times.png}
\end{figure}

\clearpage

\autoref{bronte_lmbench_ipc} shows the overhead caused by ebpH on two methods of IPC, pipes and
Unix domain stream sockets. UNIX stream socket IPC, ebpH imposes an overhead of 1.7 microseconds,
or about 18\%. For pipes, it imposes an overhead of 1.25 microseconds, or about 28\%.
While these results are significant, they shouldn't pose much of a problem
for modern applications.

<!-- IPC -->
\begin{table}
    \caption[Results of the IPC benchmarks from the \code{bronte-lmbench} dataset]{
        Results of the IPC benchmarks from the \code{bronte-lmbench} dataset.
        Standard deviations are given in parentheses and smaller overhead is better.
    }
    \label{bronte_lmbench_ipc}
    \input{../data/bench/bronte-lmbench/ipc_results.tex}
\end{table}

<!-- IPC -->
\begin{figure}
    \caption[Mean IPC times from the \code{bronte-lmbench} dataset]{
        Mean IPC times from the \code{bronte-lmbench} dataset.
        Standard error is given as error bars.
        Smaller difference in times is better.
    }
    \label{bronte_lmbench_ipc_graph}
    \includegraphics[width=.6\textwidth]{../data/bench/bronte-lmbench/ipc_times.png}
\end{figure}

\FloatBarrier

### `bronte-kernel` Kernel Compilation Micro-Benchmark

\label{bronte_kernel}

While `lmbench` provides a good representation of the overhead associated
with system simple system calls and various simple operations, it is not necessarily
indicative of performance impact as a whole. In order to ascertain how resource-intensive
operations are affected by ebpH, I ran a benchmark of Linux 5.6 kernel compilation times with and without ebpH.
Five trials were run without ebpH running and five more trials were run with ebpH running.
\autoref{tab:bronte_kernel} shows the results of the benchmark.

\begin{table}
    \caption[Kernel compilation times from the \code{bronte-kernel} dataset]{
        Kernel compilation times from the \code{bronte-kernel} dataset.
        \code{System} represents CPU time spent in kernelspace, \code{User}
        represents CPU time spent in userspace, and \code{Elapsed} represents real
        time elapsed. Note that the test was run using all 16 of \code{bronte}'s logical cores,
        therefore true elapsed time is significantly shorter than system and user CPU times.
        Standard deviations are given in parentheses and smaller overhead is better.
    }
    \label{tab:bronte_kernel}
    \input{../data/bench/bronte-kernel/bronte_kernel_results.tex}
\end{table}

According to \autoref{tab:bronte_kernel}, ebpH has relatively
small impact on the overhead of kernelspace operations during compilation,
with a `System` overhead of only $10.6\%$. This makes sense, as most of the system calls
being made during kernel compilation are relatively long to begin with, such as `execve(2)`.
Longer system calls will have a higher base time and thus ebpH's sub-microsecond runtime
has limited impact on total overhead. As expected, ebpH has negligible impact on the `User`
time, well within the margin of error. The total impact that ebpH had on compilation times
is reflected by the `Elapsed` time, which shows that ebpH only had a $1\%$ performance impact overall.

### `bronte-7day`

\label{bronte_7day}

<!-- FIXME: Redo results once bronte benchmark is over -->
<!-- FIXME: this section needs to be revised when the new 7-day macro-benchmark is done -->

The `bronte-7day` macro-benchmark was collected using `bpfbench` over a period of
14 days in total: seven days with ebpH and seven without. `bronte` is a workstation
in the CCSL lab at Carleton University. Tests were run under an idle workload.
\autoref{tab:bronte_7day} and \autoref{fig:bronte_7day} show the top 20
system calls by count (after removing outliers and high variance blocking system calls)
over the 14 day period along with associated overheads for the base and ebpH tests.

\begin{table}
    \caption[Top 20 system call overheads by count in the \code{bronte-7day} dataset]{
        Top 20 system call overheads by count, with standard deviations of less than 10 microseconds,
        in the \code{bronte-7day} dataset.
        Standard deviations are given in parentheses.
    }
    \label{tab:bronte_7day}
    \resizebox{\columnwidth}{!}{
    \input{../data/bench/bronte-7day/bronte_7day_results.tex}
    }
\end{table}

\begin{figure}
    \caption[Top 20 system call overheads by count in the \code{bronte-7day} dataset]{
        Top 20 system call overheads by count, with standard deviations of less than 10 microseconds,
        in the \code{bronte-7day} dataset. Time scale is logarithmic. Standard error is given as error bars.
    }
    \label{fig:bronte_7day}
    \includegraphics[width=.8\textwidth]{../data/bench/bronte-7day/bronte_7day_times.png}
\end{figure}

The data in \autoref{tab:bronte_7day} show that ebpH imposes anywhere from relatively moderate to
severe overhead on the most frequency executed system calls in `bronte-7day`. A few results show
slight performance improvements under ebpH, but these are anomalous. Such anomalous results are
likely due to ambient system factors such as caching, availability of resources, or highly variable behavior
based on flags, such as in the case of `ioctl(2)` whose runtime depends on implementation details within various
character devices. Besides the aforementioned anomalies, these results are
mostly indicative of the overhead that ebpH imposes on frequent system calls; however,
the next two sections will present the same benchmark run under production and ordinary use workloads,
which will be more representative of ebpH's overhead in practice.

\FloatBarrier

### `homeostasis-3day`

\label{homeostasis_3day}

The `homeostasis-3day` macro-benchmark was collected using `bpfbench` over a period of
six days in total: three days with ebpH and three without. `homeostasis` is a Mediawiki
server used to host the COMP3000 course wiki at Carleton University. Tests were
run under the normal workload associated with running the webserver and SQL database.
\autoref{tab:homeostasis_3day} and \autoref{fig:homeostasis_3day} show the top 20
system calls by count (after removing outliers and high variance blocking system calls)
over the six day period along with associated overheads for the base and ebpH tests.

\begin{table}
    \caption[Top 20 system call overheads by count in the \code{homeostasis-3day} dataset]{
        Top 20 system call overheads by count, with standard deviations of less than 10 microseconds,
        in the \code{homeostasis-3day} dataset.
        Standard deviations are given in parentheses.
    }
    \label{tab:homeostasis_3day}
    \resizebox{\columnwidth}{!}{
    \input{../data/bench/homeostasis-3day/homeostasis_3day_results.tex}
    }
\end{table}

\begin{figure}
    \caption[Top 20 system call overheads by count in the \code{homeostasis-3day} dataset]{
        Top 20 system call overheads by count, with standard deviations of less than 10 microseconds,
        in the \code{homeostasis-3day} dataset. Time scale is logarithmic. Standard error is given as error bars.
    }
    \label{fig:homeostasis_3day}
    \includegraphics[width=.8\textwidth]{../data/bench/homeostasis-3day/homeostasis_3day_times.png}
\end{figure}

As with the previous marco-benchmark, \autoref{tab:homeostasis_3day} shows that
ebpH has moderate to significant impact on the runtime overhead of the most frequently executed system calls. Of the
five most frequent system calls, all present with an overhead of less than 20\%,
and `read(2)` in particular shows a slight performance improvement under ebpH.
As in the previous section, this result is clearly pathological and is likely a result
of ambient factors such as caching and availability of resources. The overheads
of ordinary, non-blocking system calls such as `getpid(2)` and `lseek(2)` are consistent
with previously observed results. In general, the overheads presented here
are unlikely to have significant impact on performance of modern applications.

### `arch-3day`

\label{arch_3day}

Similar to the `homeostasis` tests, the `arch-3day` macro-benchmark was collected
over a period of six days on `arch`, my personal desktop computer; the idea was to see
what sort of overhead ebpH caused during the everyday use of a personal workstation.
While these results certainly have a higher variance than previous results due to
inconsistent usage and workload, it is important to see how ebpH behaves on a variety of
systems under a variety of use cases. \autoref{tab:arch_3day} and \autoref{fig:arch_3day}
show the top 20 system calls by count (after removing outliers and high variance blocking system calls)
over the six day period along with associated overheads for the base and ebpH tests.

\begin{table}
    \caption[Top 20 system call overheads by count in the \code{arch-3day} dataset]{
        Top 20 system call overheads by count, with standard deviations of less than 10 microseconds,
        in the \code{arch-3day} dataset.
        Standard deviations are given in parentheses.
    }
    \label{tab:arch_3day}
    \resizebox{\columnwidth}{!}{
    \input{../data/bench/arch-3day/arch_3day_results.tex}
    }
\end{table}

\begin{figure}
    \caption[Top 20 system call overheads by count in the \code{arch-3day} dataset]{
        Top 20 system call overheads by count, with standard deviations of less than 10 microseconds,
        in the \code{arch-3day} dataset.
        Time scale is logarithmic. Standard error is given as error bars.
    }
    \label{fig:arch_3day}
    \includegraphics[width=.8\textwidth]{../data/bench/arch-3day/arch_3day_times.png}
\end{figure}

The results shown in \autoref{tab:arch_3day}, are roughly consistent with previous
macro-benchmarks. In this dataset, the five most frequent calls present with an overhead
of less than approximately 10\%, an even better even better than previous trials.
A few system calls show moderate performance improvement, but as before, this is
likely explained by ambient factors in the system.

### Summary

\autoref{bronte_lmbench} shows that ebpH has significant impact on the overheads
of short system calls and that this impact diminishes as kernel runtime increases.
These findings extend to other aspects of system performance, such as process creation,
interprocess communication, and signal handling.
\autoref{bronte_kernel} demonstrates that ebpH imposes negligible overhead on
kernel compilation, a task that is highly intensive in CPU usage and involves the
creation of many processes. This further reinforces the notion that
ebpH's overhead is acceptable in practice, its relatively high impact
on short system call runtimes.
The results from the `bpfbench` macro-benchmarks (\autoref{bronte_7day}, \autoref{homeostasis_3day},
\autoref{arch_3day}) show that while ebpH
has significant impact on the overhead of short system calls, this impact
diminishes significantly as base system call runtime increases. Further, the
majority of frequent system calls on a variety of different work loads tend
to have a high enough base runtime that ebpH has minimal impact in practice.
<!--
\autoref{fig:macro_summary} shows system call overheads from the three macro-benchmark
datasets plotted against base runtime. This further indicates
-->

## Comparing Results with the Original pH

\label{sec:compare}

In Somayaji's dissertation [@soma02], he provides performance metrics on selected system calls
as well as kernel compilation benchmarks and X11 performance statistics. Some of the methodology
I have used for measuring ebpH's performance directly mirrors this approach in order to facilitate
easy comparison between the two systems. In particular, the `lmbench` micro-benchmarks and the
`kernel-build` micro-benchmark will be informative in this regard.

The `bronte-lmbench` system call results in \autoref{bronte_lmbench_syscall} on page \pageref{bronte_lmbench_syscall}
show that ebpH consistently adds just over a third of a microsecond of runtime to system calls on `bronte`.
Depending on the call in question, this can result in anything from minor to significant overhead -- different system calls
require different amounts of processing in kernelspace, depending on their design and implementation.
In the pH dissertation [@soma02], Somayaji presents a small variety of system call overheads with varying
base times and shows that pH adds approximately 1.9 microseconds of runtime. Although ebpH adds only
about one sixth of this overhead, this result is misleading due to the difference in hardware specifications
between `lydia`, the system which pH was tested on in 2002, and `bronte`, the system that ebpH was tested on;
in particular, `bronte` is a *significantly* faster and more powerful machine, which means that base runtime
will not be directly comparable between the two systems. The percent overhead statistic may be slightly more
informative here. For null system calls (that is, system calls which require next to no thinking on the part
of the kernel), such as `getpid` or `getppid`, ebpH adds $614\%$ overhead, which may seem quite significant.
In contrast, pH adds only about $165\%$. However, if pH were tested on `bronte` today, this overhead would
likely be much larger, as the base execution time for system calls would be significantly smaller.
As the complexity of calls increases, the percent overheads expressed by pH and ebpH approach
each other. For instance, `write(2)` has an overhead of approximately $133\%$ in pH and about $321\%$
in ebpH. `sigaction(2)` can also be compared with the signal handler install results from \autoref{bronte_lmbench_signal}
on page \pageref{bronte_lmbench_signal} (since that is in essence just a call to `rt_sigaction(2)`);
pH achieves an overhead of about $75\%$ while ebpH adds about $175\%$.

The dynamic process creation latency results from `bronte-lmbench` will also be quite informative
for establishing a comparison between pH and ebpH. In the pH dissertation [@soma02], Somayaji
presents the overheads of three distinct process creation benchmarks, exactly the same ones
that I have used here to test ebpH. For the `fork+exit` test, pH achieves $3.3\%$ overhead, while ebpH
achieves $2.7\%$; in this case, ebpH actually begins to outperform the original pH. These results
are also reflected in the next two tests, `fork+execve` and `fork+/bin/sh -c`. For `fork+execve`, ebpH
performs astonishingly well compared to its predecessor, with an overhead of $8.1\%$ compared to
pH's $273.6\%$. This result, however, is slightly misleading as pH loads profiles from disk into kernel
memory on every `execve(2)` call, whereas ebpH maintains them in a map. Thus, ebpH's overhead does not
include the overhead required to load a profile into memory. Similarly, ebpH's results in the `fork+/bin/sh -c`
test show an overhead of about $10\%$, while pH's overhead is closer to $29\%$. The impact of the differences in
handling of profiles is more diminished here, although it is still a factor. Regardless, these results show that
ebpH is consistently able to either outperform or keep up with pH in real applications.

Finally, the kernel compilation benchmarks presented in \autoref{bronte_kernel} show improvement over
the original pH results [@soma02]. In particular, ebpH only adds about $10\%$ overhead to `System` time, compared to
pH's $38\%$; however, this large improvement is most likely due to ebpH's reduced overhead on `execve(2)` calls,
which make up a large portion of kernelspace overhead for compilation tasks. Even so, the end result is a $1\%$
total performance overhead for ebpH, compared to $3\%$ for the original pH, which shows the ebpH can keep
up with pH in practice.

# Discussion

Previous sections have presented the design, implementation, and testing of ebpH, and offered
a comparison between ebpH and its predecessor, pH, in light of design and implementation
differences between the two. Past sections have shown that ebpH supports many of the same features
as the original pH while offering significantly higher portability and adaptability.
Experimental results presented in the previous section have shown that its performance
overhead can compete with the original version. This section will discuss further the viability
of eBPF-based anomaly detection in light of the results, and present topics for future work
to improve and extend future versions of ebpH.
\autoref{ebpf_shortcomings} discusses the shortcomings of eBPF that I believe need to be resolved
in order to permit more complex intrusion detection software within this paradigm, while
\autoref{future-work} presents topics for future work in development, testing, and design of
future iterations of ebpH.

## Shortcomings of eBPF

\label{ebpf_shortcomings}

In previous sections, I have highlighted the important factors that make the eBPF
paradigm an excellent choice for the development and deployment of host-base intrusion
detection systems. While the experimental results in \autoref{measuring_overhead_section}
have shown that eBPF can be as efficient as kernel-based implementations and
\autoref{implementing-ebph} has described how eBPF can be used to implement many of the same
features as kernel-based implementations, I have not yet touched on many of the shortcomings
of the technology. This section will attempt to rectify this gap in light of empirical
observations from the development of ebpH.

### Lack of Concurrency Control Mechanisms in Tracing Programs

\label{no_concurrency}

As discussed in previous sections (\autoref{impl_no_conc}), the lack of concurrency control mechanisms in
eBPF tracing programs [@verifier_git; @bpf_h_git] is detrimental to the use of eBPF for the creation of complex,
security-sensitive applications. While the risks associated with non-deterministic data
are fine for simple tracing programs designed for use cases such as performance analysis,
this assumption quickly breaks down for more complex applications that rely on accurate results.
The current version of ebpH mostly gets away with this due to the way it handles lookahead pairs
combined with the use of atomic add and subtract operations for profile flags. However, future iterations
of ebpH may depend on more complex behavioral tracking and analysis which is currently not possible
in eBPF to an acceptable degree of certainty.

eBPF *does* have restricted concurrency primitives, such as `bpf_spin_lock` [@bpf_h_git], but these are limited
to non-tracing (and non-socket) programs due to insufficient checks by the verifier. This is to prevent
buggy BPF programs from causing kernel functions to timeout, which could potentially crash the system.
Resolving this problem would require updates to the verifier to ensure that it can properly
check preemptions related to spin locks in tracing programs. While this functionality is currently not
available the BPF maintainers have indicated that they are planning to support locking in tracing programs
in the future [@bpf_h_git].

### Limited Support for Necessary Kernel Helpers

One of the major improvements of extended BPF over classic BPF is the introduction of
the `bpf_call` instruction to its bytecode [@starovoitov13; @starovoitov14].
In particular, eBPF programs can invoke a predetermined set of helper functions
provided by the kernel [@gregg19bpf; @man_bpf_helpers]. Due to verifiability requirements on BPF programs,
the set of kernel functions that can be invoked is *highly* limited in scope.
As of Linux 5.5, eBPF supports 117 distinct helper functions [@bpf_h_git]. However,
many of these helpers relate specifically to operations on eBPF maps and lookups on
architecture-specific kernel data structures and more still are limited to specific
niche program types, such as XDP, socket filter, or traffic classifier programs.

One particular pain-point that I encountered during the development of ebpH is
the lack of a reliable means of constructing pathnames in eBPF. The kernel provides
helpers for doing so, but these are not available for BPF programs. This means that
ebpH is unable to support hashing profiles by pathname, and instead must rely
on the computation of a unique key from filesystem metadata. While this solution is
pragmatically the same, ebpH's usability suffers as a result. In the original pH,
profiles were stored on disk in subdirectories that mirrored their pathname in the original filesystem;
in ebpH however, they are simply stored with the same filename as the corresponding profile key.
This makes it difficult for users to interact with ebpH without using `ebph-ps` to figure out
the key of the profile they want first. While there are potential workarounds for this, including the
determination of pathnames in userspace, these are not ideal in terms of performance or reliability.
It is worth noting, however, that a patch is currently under review to remedy this gap in eBPF's
functionality [@zhang19].

### Verifier Bugs

Although the verifier provides critical safety guarantees to eBPF programs,
it suffers from a few bugs that, in the best case, make it difficult to work with.
In particular, the verifier can be inconsistent when performing static analysis
on large and complex programs, such as the BPF programs employed by ebpH.
To illustrate this complexity empirically, consider \autoref{fig:sys_exit} which
depicts the instruction flow of ebpH's `sys_exit` tracepoint program.

<!-- FIXME: make this not be a draft image for final version -->
\begin{figure}
    \caption[The instruction flow of ebpH's \code{sys_exit} tracepoint program]{
        The instruction flow of ebpH's \code{sys_exit} tracepoint program.
        Note the complexity of the BPF program. This figure was generated using
        \code{bpftool} and \texttt{graphviz}'s \code{osage} tool.
    }
    \label{fig:sys_exit}
    \includegraphics[width=.6\textwidth]{../figures/progviz/raw_syscalls_sys_exit.png}
\end{figure}

The verifier itself is a rather complex program; as of Linux 5.5, it consists of over 10,000
lines of C code [@verifier_git]. As a consequence of this complexity, the probability of bugs
increases significantly. If the verifier fails at any stage in the verification process, it
errs on the side of caution and rejects the program. Unfortunately, this behavior does impact
ebpH to an extent. Due to a presently unknown bug in the verifier, it occasionally rejects
the `sys_exit` program depicted in \autoref{fig:sys_exit}; this issue can be resolved by restarting
the system. While this behavior is certainly annoying, it is an acceptable trade-off for the
safety guarantees that the verifier provides when it is working properly.

One argument that may arise from this notion of inconsistent verifier behavior is whether
it truly protects the system at all from buggy BPF programs. After all, one of the primary
advantages cited for eBPF programs over kernel-based implementations is the ability to
guarantee production safety despite BPF code running in ring 0 with full access to the kernel.
A counter-point to this argument is that a 99\% probability of guaranteeing safety is better
than a 0\% probability --- that is to say, having a verifier that works almost all of the time
is better than not having one at all.

### Dropped Perf Buffer Submissions

Another primary advantage for using eBPF over traditional kernel-based
implementations is the ability to easily buffer communication with userspace.
Context switches between userspace and kernelspace are expensive [@gebai18];
eBPF largely mitigates this by allowing userspace programs to buffer map access,
which in turn allows for variable granularity in the amount of context switches
required per event. For instance, ebpH reads events from its BPF programs
using the perf event buffer interface provided by eBPF; to do this, it
simply polls each map every second via an event loop.

While perf buffers do reduce program overhead in practice, they have caveats of their
own that need to be addressed. For instance, BPF programs may outright refuse to
submit some perf events (this behavior was encountered during the development of ebpH)
and events that occur too frequently may fill the buffer completely, which in turn causes
events to be dropped. Although these caveats do pose significant challenges to the development
of reliable BPF programs, it is possible to circumvent them with careful design choices.
For instance, submission failure can be checked within the BPF program and backup mechanisms can
then be employed to be sure that the data makes it to userspace; dropped submissions that occur
due to high frequency events may be solved by tuning the size of the buffer or adjusting the frequency
at which the BPF program polls it.

## Future Work

This thesis was primarily focused on three important points:

1) Establishing the viability of eBPF as a method for host-based anomaly detection;
1) Showcasing and describing ebpH, a partial reimplementation of Somayaji's pH [@soma02] in eBPF;
1) Determining the experimental and practical overhead of ebpH on system performance.

Although these points are enough to define a significant contribution in the context of
an undergraduate thesis, there remains several aspects of the project that can be improved upon or
more thoroughly analyzed, and used for determining the direction of future iterations or other related
research endeavors. To that end, I propose several topics for future work on ebpH and related projects
in this section. Many of these will be explored in depth as part of my work for my upcoming
Master of Computer Science thesis. In this section, I will be covering the following points:

1) The need to control for further sources of non-determinism (c.f. \autoref{further_non_determinism});
1) Potential avenues for adding automated response to ebpH (\autoref{response_automation});
1) A security analysis of ebpH (c.f. \autoref{security_analysis_section});
1) Refactoring ebpH to use new hashmap types to reduce memory overhead and squash bugs (c.f. \autoref{lru_section});
1) The need for a graphical user interface and subsequent usability study (c.f. \autoref{gui_section});
1) Retrofitting ebpH to make use of other sources of system data, beyond system calls (c.f. \autoref{general_introspection}).

### Controlling for Further Sources of Non-Deterministic Behavior

\label{further_non_determinism}

While simple binaries do normalize relatively quickly in ebpH, the complex
behavior of many modern processes causes some binaries to never normalize.
This is due to sources of *non-determinism* in their behavior. Some examples of
things that might introduce non-determinism include scheduling hints,
complex event-based behavior, and interprocess communication.

Currently, ebpH handles the non-determinism of signal handlers by implementing a stack of
sequences and pushing to and popping from that stack as signal handlers are called and returned.
While this is a start, it does not capture all sources of non-determinism in complex programs
such as a web browser. Instead, ebpH needs to know what system calls are associated with
non-deterministic behavior and treat these calls differently. Depending on the severity
of the issue, this may require a re-design of ebpH's heuristics for analyzing profile
lookahead pairs.

Future work in this area will include determining commonalities between new sequences, especially
false positives in normalized binaries, and analyzing these commonalities to deduce precisely
what these sources of non-deterministic behavior are and how to mitigate related issues therein.
This will be a major focus of improvement for future iterations of ebpH.

### Automating ebpH Response

\label{response_automation}

ebpH's predecessor, pH [@soma02], was capable of responding to attacks by issuing delays
to system calls proportionally to recent anomalous behavior. The current version of ebpH lacks
this functionality due to implementation constraints imposed by eBPF. However, recent additions to
eBPF have made it more conducive to automated response [@gregg19bpf]. In particular, Linux 5.3
introduced two critical helpers [@bcc] for policy enforcement from BPF: `bpf_signal` and `bpf_override_return`.

`bpf_signal` provides the ability for BPF programs to send arbitrary signals to the current task
directly from kernelspace. Since the signal is coming from the kernel, it will be delivered instantly,
without the usual delays associated with sending signals from userspace. By sending a process the signal
`SIGSTOP` [@man_signal], it will be possible to stop its execution in *real time*, during the offending
system call. Subsequently, a `SIGCONT` [@man_signal] can be issued to wake the process once its delay has
been observed. This second signal could either be sent from userspace (since we no longer have the same
sense of urgency associated with the initial response) or issued from some frequently invoked BPF tracepoint,
for example `sched_switch`, after a predetermined amount of time has passed.

`bpf_override_return` could be used to implement the second response category employed in pH [@soma02]:
`execve(2)` abortion, cited by Somayaji's dissertation as being necessary to defeat certain classes of
attacks (e.g. buffer overflows for shell code execution). With `bpf_override_return`, ebpH can issue targeted
error injections one of the helper functions used by `execve(2)`-family calls to load binaries.

By combining the above two techniques, it will be possible to convert ebpH into a fully functional
intrusion prevention system, like its predecessor. Signals can be used to implement process delays
and targeted error injections can be used to implement `execve(2)` abortion. With these two changes,
ebpH's functionality will become a superset of the original pH's, which will facilitate direct
comparison between the two systems when conducting a security analysis (c.f. \autoref{security-analysis}).

### Security Analysis

\label{security_analysis_section}

In order to measure ebpH's effectiveness at detecting and (in future versions) mitigating attacks,
it is necessary to conduct a thorough security analysis of the system. In anomaly detection, there
are a few important heuristics to consider when determining the efficacy of a system:
false positive rate (FPR), false negative rate (FNR), true positive rate (TPR), true
negative rate (TNR), and alarm precision (AP) [@vanoorschot19]. When combined, these
heuristics provide an accurate representation of:

- How often the system flags legitimate behavior as anomalous (FPR);
- How often the system misses anomalous behavior (FNR);
- How often the system detects anomalous behavior (TPR);
- How often the system allows legitimate behavior (TNR);
- The ratio of true positives to total positives (AP).

According to the above definitions, FPR and FNR provide an indication of the *error rate*
of an anomaly detection system, while TPR and TNR provide an indication of the *correctness rate*.
Finally, AP provides an indication of what percentage of all flagged anomalies are correct.
Determining these five heuristics for ebpH will require carefully planned testing strategies
comprised of building known-good profiles, mounting various known attacks, and measuring
rates of flagged events against predetermined values. Additionally, the same known-good profiles
should be tested for extended periods of time under normal system behavior to ensure that the
rate of false positives is acceptable.

As a general-purpose anomaly-based IDS, it is important to show that ebpH
is capable of detecting a wide variety of attacks. The mimicry attacks described in Wagner and Soto's paper [@wagner02]
are particularly interesting, as they were directly designed to defeat the original pH system
(albeit an earlier version with much shorter lookahead pair window length) by constructing attack patterns
that generate false negatives. The results depicted in their paper should be compared against the results from
testing and used to inform later changes to ebpH.

### Refactoring Profile and Process Hashmaps to Other Map Types

\label{lru_section}

As discussed in previous sections, ebpH is not as memory-efficient as its predecessor due
to implementation details with how it stores profile and process data. Currently,
ebpH uses two ordinary hashmaps to store profiles and process information, which are created with
a special flag that signals the kernel to dynamically allocate them rather than preallocate.
While this saves on the overhead of allocating all profiles and processes beforehand (which would
be prohibitively expensive), there are several problems with this design choice.
In particular, known issues with dynamic map allocation may cause deadlocks under conditions with
high event frequency [@starovoitov16prealloc] and the granularity of allocation is too large to
efficiently store the large sparse data structures required by ebpH to manage lookahead pairs
in profile data.

To that end, I plan to make the following changes in a future iteration of ebpH:

1) Refactoring all hashmaps into the `LRU_HASH` type;
1) Refactoring profile data storage to use a `HASH_OF_MAPS`.

#### Refactoring Profiles and Processes to Use `LRU_HASH`

The `LRU_HASH` [@bcc; @gregg19bpf] is an eBPF map type of size $n$ that keeps $n$ entries preallocated
at all times. When a BPF program attempts to add an entry to a full `LRU_HASH`, it discards
the least recently used data from the map to make room for the new entry. While this behavior may not
seem ideal, it serves as a reasonable compromise between the current dynamic map allocation approach
that may cause deadlocks and a preallocation approach that would be infeasible due to memory restrictions.
With an `LRU_HASH`, the size of the preallocated map can be a fraction of the size of the current
maps that ebpH uses. For instance, ebpH's process map has 4,194,304 entries by default, one for each
possible thread ID on the system, to ensure that new processes will always have space in the map.
With an `LRU_HASH`, this would no longer be needed, as adding a new process to the map
would simply cause ebpH to forget about the least recently used process. Even using the generous
default map size of 10,240 entries would represent a 99.7\% reduction in map size, which would
in turn reduce the overhead of preallocating the entire map significantly.

#### Refactoring Profile Data to Use `HASH_OF_MAPS`

`HASH_OF_MAPS` [@bcc; @gregg19bpf] is a type of map-in-map data structure that allows
BPF programs to define and store maps inside of other maps. While support for this
was added in 2017 to Linux 4.12 [@lau17], `HASH_OF_MAPS` and `ARRAY_OF_MAPS` have
only been supported in bcc [@bcc] since a November 2019 patch [@song19].
With map-in-map support, ebpH can redefine the way it stores profile data, significantly
reducing the granularity of profile data allocation. In particular, lookahead pair
data can be stored in a per-profile two-layer hashmap, indexed by two keys: current and
previous system call. This means that ebpH would only need to allocated
the lookahead pairs that are currently in use by a given profile, which would save
significantly on memory overhead compared to the current design.

### Reintroducing the ebpH GUI and Conducting a Usability Study

\label{gui_section}

If ebpH is to be a truly adoptable security solution, it first needs to be a usable one.
In its current incarnation, ebpH is not user friendly; it supports minimal interaction
through a set of simple CLI programs and most user feedback and notification occurs
through log files. In order to ebpH to become usable software, the daemon needs a graphical
user interface to act as a dashboard for user interaction.

I envision the ebpH GUI as a way for the user to get detailed information about the behavior
of their system and make administrative decisions therein. For example, the user could
view a visual representation of recent anomalous behavior, inspect profiles for detailed
information, make modifications to profiles, and perform operations on running processes
such as killing them or blacklisting them from monitoring. This will allow the user
to use ebpH as a tool for visualizing system behavior and make easy modifications to ebpH's
enforcement on the system.

Both during and after the development of the GUI, I plan to conduct usability studies to get
an idea of how users interact with ebpH, what their perception of the system is, and whether
changes need to be made to increase its potential for future adoption.

### General System Introspection: Integrating Multiple Homeostatic Systems into ebpH

\label{general_introspection}

<!-- TODO: write this section -->

One of eBPF's primary strengths is the ability to monitor the *entire* system
at once. BPF programs can be written to instrument system calls, kernel functions,
signals, library calls, memory allocations, keyboard input, incoming network packets ---
the list goes on. What's more, eBPF programs can freely communicate with each other and with userspace
programs through maps. Monitoring system call sequences is a good start for eBPF anomaly detection,
but this can be extended to do so much more. In Somayaji's dissertation [@soma02], he describes future
iterations of pH that consist of multiple homeostatic systems working together, interacting with each other,
and monitoring many aspects of system behavior in a loosely couple manner. This vision fits the eBPF
paradigm perfectly, and I believe that this is something that will be achievable in future versions of ebpH.

After determining what aspects of system behavior it needs to monitor, extending ebpH in such a manner
would be a relatively straightforward process. The daemon's API is already extensible, and adding more
data sources would be as simple as writing BPF programs to instrument them; the BPF programs could then interact
with each other via map access. All that remains is to come up with a new heuristic to integrate the data sources
into a cohesive detection and response mechanism. The original pH dissertation [@soma02] can provide further guidance
in this regard.

Integrating multiple data sources in this way would not only move ebpH towards emulating true homeostasis
in biology, but would also serve as a means of dealing with the non-deterministic behavior described
in previous sections. With multiple homeostatic mechanisms providing feedback to each other, the impact of
false positives in system call sequences will naturally diminish. The current prototype of
ebpH already has an example of this in the way that it handles signals --- ebpH uses the invocation of a signal
handler to inform its decision-making with respect to novel system call sequences, and thus reduce non-determinism
in sequences.

<!-- NOTE: maybe we can re-use some of this?
As an intrusion detection system, ebpH's role is well-defined: monitor the system, detect misbehaving processes,
and report them to the user. However, there is one glaring problem with this approach, particularly as we venture
into the territory of automated responses via system call delays: users do not necessarily *want* a system that
chooses not to perform a requested action; they also do not necessarily *want* a system that harasses them
with warnings about program behavior that they either don't care about or don't necessarily understand.

One potential solution to this problem is providing other benefits to the user through ebpH *in addition
to* intrusion detection and response. For example, future versions of ebpH could include a performance analysis component,
a debugger component, or any number of other metrics for increased system visibility. After all, one of the primary use cases
for system introspection is precisely that: allowing a user to observe their system. By adding this extra functionality,
we can provide complimentary benefits to the user that may incentivize them to run ebpH in the first place.

It should also not be overlooked that, in many cases, increased system state visibility can provide implicit security
benefits to the experienced user. For example, an experienced system administrator could use a future version of ebpH
to find vulnerabilities in their system before an attack even occurs.
-->

# Conclusion

In this thesis, I have presented the design and implementation (c.f. \autoref{sec:impl}) of ebpH,
a host-based intrusion detection system written in eBPF that instruments system calls and
builds per-executable behavioral profiles. Experimental results (c.f. \autoref{measuring_overhead_section})
have shown that ebpH can keep up with the performance of its kernel-based predecessor, pH.
Finally, I presented my plans for the future of ebpH as a system that can integrate many facets of
system behavior by leveraging the eBPF paradigm to take advantage of its multi-faceted capabilities
with respect to system instrumentation; this, coupled with a few improvements made possible by
recent advances to eBPF itself, should position ebpH as a strong contender in the field
of intrusion detection (c.f. \autoref{discussion}). \autoref{tab:new_comparison} presents
a theoretical comparison between a future version of ebpH, the current version of ebpH, and the original pH.

\begin{table}
    \caption[Revisiting the comparison of ebpH and pH in light of topics for future work]{
        Revisiting the comparison of ebpH and pH in light of topics for future work.
        Note that \texttt{ebpH 1.0} represents the current version of ebpH, while
        \texttt{ebpH 2.0} represents the future version of ebpH that was discussed in
        \autoref{future-work}.
    }
    \label{tab:new_comparison}
    \resizebox{\columnwidth}{!}{
    \begin{tabular}{>{\ttfamily}lllcccccc}
        \toprule
        \multicolumn{1}{l}{\bfseries System} & {\bfseries Implementation} & {\bfseries Data Collected} &
            \rotatebox{90}{Portable} & \rotatebox{90}{\parbox{2cm}{Production\\Safe}} &
            \rotatebox{90}{\parbox{2cm}{Low Mem.\\Overhead}} &
            \rotatebox{90}{\parbox{2cm}{Low Perf.\\Overhead}} &
            \rotatebox{90}{Detection} & \rotatebox{90}{Response} \\
        \midrule
        pH \cite{soma02} & Kernel Patch & System call sequences
            & \xmark & \xmark & \cmark & \cmark & \cmark & \cmark\\
        ebpH 1.0         & eBPF + Userspace Daemon & System call sequences
            & \cmark & \cmark & \xmark & \cmark & \cmark & \xmark \\
        ebpH 2.0         & eBPF + Userspace Daemon & Many aspects of system
            & \cmark & \cmark & \cmark & \cmark & \cmark & \cmark \\
        \bottomrule
    \end{tabular}
    }
\end{table}

\clearpage

Ultimately, this work shows that eBPF represents a powerful tool for building versatile, performant,
and production safe intrusion detection systems. While current work in this area is representative of its advantages
in network-based IDS implementations, eBPF has equal merits in host-based implementations.
The current version of ebpH serves as a proof of concept to demonstrate eBPF's value in
this regard, and future iterations on my prototype will hopefully be able to take further advantage of
eBPF's power and versatility to deliver a truly homeostatic intrusion detection system.

<!-- References -->
\clearpage
\addcontentsline{toc}{section}{References}
\printbibliography
\clearpage

\appendix
\appendixpage

# Handling Large Datatypes in eBPF Programs

\label{ebpf-design-patterns}

\lil[language=c, caption={Handling large datatypes in eBPF programs.}, label={appendix-bigdata}]{../code/design_patterns/bigdata.c}

<!-- TODO: Add the following:
            Setting runtime parameters with arrays
            Issuing commands with shared library + uprobes
-->

\FloatBarrier

\clearpage

# `bpfbench` Source Code

\label{bpfbench}

\lil[language=c, caption={The eBPF component of \code{bpfbench}.}]{../code/bpfbench/src/bpf/bpf_program.c}

\FloatBarrier

\clearpage

# Script Used to Run Kernel Compilation Trials

\label{appendix:kernel}

\lil[language=bash, caption={The shell script used to measure times for the kernel compilation benchmark.}]
{../data/bench/bronte-kernel/bench.sh}

\FloatBarrier

\clearpage

# Full Macro-Benchmarking Datasets

\label{appendix_datasets}

<!-- TODO: include these -->
\begingroup
\let\tablesize\scriptsize
\input{../data/bench/bronte-7day/bronte_7day_full_results.tex}
\input{../data/bench/homeostasis-3day/homeostasis_3day_full_results.tex}
\input{../data/bench/arch-3day/arch_3day_full_results.tex}
\endgroup
