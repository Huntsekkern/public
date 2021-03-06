# Exercise 4

*Managed code* is code whose memory is allocated and freed for  you (eg. Java,
C#, ...). Basically code with garbage collection and other goodies.  *Native
code* is code whose memory is not "managed", as in there is no reference
counting, no garbage collection, and memory isn't freed for you (C++' delete and
C's free, for instance).
[source](https://stackoverflow.com/questions/855756/difference-between-native-and-managed-code)

A program is *memory-safe* if it is protected from various software bugs (index
of out bounds exceptions, ...) and security vulnerabilities (buffer overflows,
dangling pointer dereferences, ...).

Managed languages have the advantage that the runtime can perform sanity checks
on the memory-accessing parts of the code to detect programming errors and
report them (at the expense of some performance overhead). Java is an example of
a managed language as its runtime error detection checks array bounds and
pointer dereferences automatically.

Native languages (C, C++, ...) offer performance, but have the drawback that
they don't protect the user against illegal memory accesses. Some of these
memory accesses are caught by the operating system and cause a segmentation
fault, but the operating system cannot catch any illegal memory accesses which
lie within the program's mapped address space.

Through rigorous coding practices, one can write both high-performance and
memory-safe code in native languages, however even the best native code
programmers inevitably write code that violates some security policy. In a
perfect world, programmers would know the exact location of all security
vulnerabilities in their codebase and could just patch the bugs, but we don't
live in a perfect world and are definitely not oracles! What one wants is a tool
that could automatically enforce certain security policies in native code and
flag them as soon as they occur.

*Sanitizers* are a possible solution to this problem as they enforce a security
(or safety) policy and make some form of undefined behavior explicit.  Instead
of silently corrupting program state, the sanitizer detects the violation when
it happens and stops program execution with a detailed error report. Sanitizers
result in high overhead due to the additional metadata they track and therefore
are a development tool, often run with test cases or with fuzzing.

Most sanitizers are implemented in two parts:

1. A compiler pass or dynamic binary translation that
   [instruments](https://en.wikipedia.org/wiki/Instrumentation_(computer_programming))
   existing code, and
2. Runtime checks that enforce the desired security policy.


The goal of this lab is to get familiar with one of the built-in sanitizers from
the LLVM compiler infrastructure, [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) (ASan)

## AddressSanitizer (ASan)

ASan detects spatial and some temporal memory safety violations by instrumenting
object allocations/deallocations and pointer dereferences. ASan keeps metadata
about each allocated memory object, including where the object was allocated and
if the object is still valid.  Each pointer read or write is instrumented with a
fast validity check and violations trigger an exception and a report. You can
compile your code with ASan by using the `-fsanitize=address` flag to the
`clang` compiler. You can learn more about ASan
[here](https://github.com/google/sanitizers/wiki/AddressSanitizer).

## Task 0: Set-up your sanitization environment

The first step is to set up the necessary build environment. You'll need at
least a recent version of `clang` installed (4.0 should suffice). LLVM comes
with a set of sanitizers by default which allow you to compile programs with
additional sanitization policies. Sanitizers are exclusive, so you can only use
one sanitizer at a time. To get a feel for what ASan does, try to test it on the
following program which suffers from a use-after-free memory bug:

```C
#include <stdlib.h>

int main() {
  char *x = (char*) malloc(10 * sizeof(char*));
  free(x);
  return x[5];
}
```

You can do this by compiling and running the program above using

```bash
clang -fsanitize=address use-after-free.c -o use-after-free
./use-after-free
```

Look at the output generated by ASan and try to understand what it is precisely
flagging.

## Task 1: Getting acquainted with ASan
To evaluate ASan's sanitization abilities, we will leverage the ASan sanitizer
on a test suite of synthetic C/C++ programs which contain lots of known flaws.
Using the sanitizer on a codebase with *known* flaws helps understand the
class of memory violations the sanitizer can protect against.

The test suite we will use for this experiment is v1.3 of the [Juliet test
suite](https://www.nist.gov/publications/juliet-11-cc-and-java-test-suite) for
C/C++ programs with 64,099 test programs.

The `setup.sh` script takes care of downloading the Juliet test suite, selects a
subset of the CWEs to run the sanitizer on, then reports summary statistics on
the sanitizer's ability to detect the memory-safety violation.

Note that, under the hood, the `setup.sh` script uses
`create_single_asan_Makefile.py` to construct a custom Makefile that compiles
the classes of CWEs of interest. The assignment uses CWE562 (return of stack
variable address) as a showcase. If you want to try another CWE, you'll need to
modify the following regex in `create_single_asan_Makefile.py`:

```Python
testregex = "CWE(562)+((?!socket).)*(\.c)$"
```

If you run `setup.sh` on CWE562, you should get the following output:

```
Calling CWE562_Return_of_Stack_Variable_Address__return_buf_01_good();
helperGood1 string
CWE562_Return_of_Stack_Variable_Address__return_buf_01_good(); passed
Calling CWE562_Return_of_Stack_Variable_Address__return_pointer_buf_01_good();
elperGood1 string
CWE562_Return_of_Stack_Variable_Address__return_pointer_buf_01_good(); passed
Calling CWE562_Return_of_Stack_Variable_Address__return_buf_01_bad();

CWE562_Return_of_Stack_Variable_Address__return_buf_01_bad(); failed
CWE562_Return_of_Stack_Variable_Address__return_buf_01_bad();
Calling CWE562_Return_of_Stack_Variable_Address__return_pointer_buf_01_bad();
z
CWE562_Return_of_Stack_Variable_Address__return_pointer_buf_01_bad(); failed
CWE562_Return_of_Stack_Variable_Address__return_pointer_buf_01_bad();
Total Bad Tests: 2
Passed: 0
Failed: 2
Total Good Tests: 2
Passed: 2
Failed: 0
Total Bad Time Outs: 0
Total Good Time Outs: 0
Total Bad Tests: 2
Passed: 0
Failed: 2
Total Good Tests: 2
Passed: 2
Failed: 0
Total Bad Time Outs: 0
Total Good Time Outs: 0
```

Notice that the different tests are classified into known good/bad cases, and
that the tool returns a summary count of which bad tests failed (which is what
we want), as well as which good tests passed (which is also what we want). The
quality of a sanitization tool comes from its ability to classify program
behaviors accurately.

We encourage you to check the source code of the examples the sanitizer is
running on to understand what the tool is detecting.

## Task 2: Evaluating ASan

* What CWE classes can the sanitizer theoretically detect? Read the description
  of the sanitizers and discuss which CWE classes are covered.

* Modify the regex in `create_single_asan_Makefile.py` to compile the test
  classes you think the sanitizer should cover.

* Run the test cases and discuss what how many false positives (the sanitizer
  reports a violation even though the program is legal) and false negatives (the
  sanitizer misses a violation) you have. What could be the reason for the false
  positives or false negatives? Discuss an example of a false positive/negative.
