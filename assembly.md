# x86_64 Assembly Handbook for your JPL compiler

You will be using an assembler called NASM. We picked this assembler
because it is user-friendly (compared to other assemblers) and
cross-platform. It has [nice tutorials][tutorial] and a [comprehensive
manual][manual].

[tutorial]: https://cs.lmu.edu/~ray/notes/nasmtutorial/
[manual]: https://nasm.us/doc/nasmdoc0.html

You can assemble some assembly code like this:

    nasm -fXXX code.s

Here, `XXX` is the name of your object file format. On Linux, that'll
be `elf64`, on macOS `macho64`, and on windows `win64`. Don't forget
the "64". Assembling produces an object file with a `.o` extension.
You then need to link it, like this:

    ld code.o runtime.o

The `runtime.o` in this case is the object file containing the
provided functions. This will produce a file called `a.out`, short for
"assembler output". (Note that it is not actually output by your
assembler...)

You can run that file:

    ./a.out

It should run! If it doesn't, you'll need to debug. That will be hard.

## Syntax of Assembly

Lines of assembly generally look like this:

    LABEL: INSTRUCTION WIDTH OPERAND, OPERAND ; COMMENT
    
Labels are optional and name locations in the binary code. The width
is usually inferred from the operands, but you can explicitly specify
it as `qword` for 64 bits when you need to. Different instructions
have different numbers of operands. The *destination* register comes
first. Semicolons start a line comment.

Your assembly file should start with the the following text:

    global _main

This tells NASM that the assembly code defines a function named
`_main` and that that function name should be externally available (to
the runtime).

Next, your assembly file will have the line:

    section .data

Following that line, you'll define a bunch of constants, and after
that you'll have:

    section .text

after which will come all of your assembly instructions.

Here are some short, quick guides to x86 assembly:

- https://khoury.neu.edu/home/ntuck/courses/2018/09/cs3650/amd64_asm.html
- http://www.cs.cmu.edu/afs/cs/academic/class/15213-s20/www/recitations/x86-cheat-sheet.pdf

These guides are a bit longer and more detailed:

- https://software.intel.com/content/dam/develop/external/us/en/documents/introduction-to-x64-assembly-181178.pdf
- https://www.cs.tufts.edu/comp/40/docs/x64_cheatsheet.pdf

And finally, this is the definitive guide:

- https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html

## The x86 Registers and Memory Layout

Modern x86 CPUs (technically, x86_64 CPUs) have 16 integer registers
and 8 floating point registers that you can access directly. All are
64 bits wide. We will be using them like this:

- `rsp`: The stack pointer, which we'll use to pass complex arguments
  (like tuples and arrays) to functions
- `rbp`: The base pointer, which we'll use to refer to positions in
  the stack frame
- `rax`: The register containing integer return values
- `rbx` and `r10`: Registers that we'll use as intermediate
  locations in a lot of assembly snippets
- `ebx`: The bottom 32 bits of the `rbx` register, which we'll
  sometimes use to do 32-bit loads and stores.
- `xmm0` through `xmm7`: Floating-point registers

Generally speaking, x86 programs have a _stack_ and a _heap_. The
stack is for storing local variables; the heap is for storing
in-memory data that persists across function calls.

On x86 the top of the stack is pointed to by `rsp`. Generally speaking
`rbp` is used to point to the other end of the current stack frame
(the part of the stack that's for the currently executing function).
Usually local variables are referenced relative to `rbp`. The stack
grows *down*, so `rsp <= rbp` and you *subtract* from `rsp` to add
more to the stack.

## Debugging Assembly

You can use `gdb` or `lldb` to debug a program generated from
assembly. Here are some things to keep in mind:

- Both tools use a different assembly syntax. You'll figure most of
  the differences out on your own, but you may be surprised to learn
  that they swap the arguments, with the destination register coming
  last.

Here are some common error messages:

- `stack_not_16_byte_aligned_error`: Your stack size must be a
  multiple of 16 bytes. Make it a little bigger to make it a multiple
  of 16 bytes.
- `impossible combination of address sizes`: You forgot the "64" at
  the end of your NASM format flag. It should be, for example, `elf64`
  not `elf`.

Something weird that could happen:

- If you find that you're only printing part of long strings, or
  printing some garbage before those strings, check that you're not
  overwriting part of the string pointer. This might be from using an
  `rbx` register where you should be using `ebx` with booleans, for example.

Some tutorials on GDB:

Here is the [gdb documentation][gdb-docs]. Various GUI front-ends for
gdb are available if you prefer a non-command-line debugging
environment. Here are some quick tutorials on using the
assembly-relevant parts of gdb:

[gdb-docs]: https://www.gnu.org/software/gdb/documentation

- https://www.cs.umb.edu/~cheungr/cs341/Using_gdb_for_Assembly.pdf
- https://www.cs.swarthmore.edu/~newhall/cs31/resources/ia32_gdb.php
- https://web.cecs.pdx.edu/~apt/cs491/gdb.pdf

Alternatively, you can use `lldb`, which is similar but has somewhat
different command names.

## Defining Constants

Integer, floating point, and string constants are defined differently
in assembly. Here's a quick example:

    four: dq 4
    pi:   dq 3.1415926535897932384626433
    hewo: db `Hello, World!`, 0

Define integer constants like this:

    NAME: dq VALUE

Define floating point constants like so:

    NAME: dq VALUE

Define strings like this:

    NAME: db `VALUE`, 0

Here `NAME` is the name of the constant and `VALUE` is the original
value being stored. Note that for strings, you *must* include the
comma and the zero, which adds the null byte. Also note that you use
backticks, not double quotes. Double quotes don't allow
backslash-escapes, while backticks do.

## Function prologue and epilogue

The function prologue is the following sequence of assembly commands:

	push    rbp
	mov     rbp, rsp
	sub     rsp, XXX

where `XXX` is the total size of the stack frame. The function
epilogue is the following sequence of assembly commands:

	add     rsp, XXX
	pop     rbp
	ret

Here `XXX` is again the total size of the stack frame. The function
prologue and epilogue are perfect mirrors and must match.

## Compiling statements

A `let` statement with an integer or floating point constant
corresponds to the following assembly:

    mov     rbx, [rel NAME]
	mov     [rbp - XXX], rbx

Here `NAME` is the constant's name and `XXX` is the location
corresponding to the variable on the left hand side of the `let`.

A `let` statement with a reference to an integer of floating argument
corresponds to the following assembly:

    mov		rbx, [rbp - XXX]
    mov		[rbp - YYY], rbx

Here `XXX` is the location of the variable on the right hand side and
`YYY` is the location of the variable on the left hand side.

An `assert` statement with a variable reference corresponds to the
following assembly:

	cmp qword	[rbp - XXX], 0
    jne		.SKIP
    mov		rdi, [rel NAME]
    call	_abort
    .SKIP:

Here, `XXX` is the location of the variable in the assertion, `SKIP`
is a unique name for the assertions, and `NAME` is the name for the
string constant in the assertion. Note that `SKIP` is preceded by a
dot.

A `return` statement with a variable reference corresponds to the
following assembly:

	movq	-XXX(%rbp), %rax

Here `XXX` is the location of the variable.

All other commands call a function (either a named one like `blur` and
`sepia` or another provided function like `time` or `read_image`).

## Calling functions

A function call has some number of arguments and one output. But how
exactly it is called depends on the details.

Integer and boolean arguments are passed in the registers `rdi`,
`rsi`, `rdx`, `rcx`, `r8`, and `r9`, in that order. Integer and
boolean return values are located in `rax`.

Float arguments are passed in `xmm0`, `xmm1`, `xmm2`, `xmm3`, `xmm4`,
`xmm5`, `xmm6`, and `xmm7`. Float return values are located in `xmm0`.

`pict` arguments are passed on the stack, so they do not require any
registers at all. `pict` return values, however, mean an extra integer
passed as the first argument, in `rdi` (which contains a pointer to
where that `pict` must be written).

Calling a function requires setting up each argument; calling the
function; and saving the return value.

`int` arguments set with this bit of assembly for variable arguments:

    mov      REG, [rbp - XXX]
             
where `XXX` is the location of that integer argument on the stack and
`REG` is the destination register

For constant integer arguments use this bit of assembly instead:

    mov      REG, [rel NAME]
             
where `NAME` is the name of the constant.

Finally, if the function returns a `pict` value, it that actually
takes an extra first argument. Don't get this wrong. Set that first
argument like so before calling the function:

	lea      rdi, [rbp - XXX]

Since it's a first argument, it always takes up the `rdi` register.
This uses the `lea` command instead of the `mov` command to load the
address, not its contents.

`bool` arguments are moved to their destination register like so:

    mov      ebx, [rbp - XXX]
    mov      REG, rbx

This uses two `mov` commands to convert 32-bit to 64-bit values.

`float` arguments are moved to their destination register like so:

    movsd	REG, [rbp - XXX]

For constant integer arguments use this bit of assembly instead:

    mov      REG, [rel NAME]
    
where `NAME` is the name of the constant.

String arguments are moved to their destination register like so:

    lea     REG, [rel NAME]

Here `NAME` is the name of that string constant. Note that we use
`lea` instead of `mov` since we want to store a pointer in the
register, instead of storing the value at the pointer.

To push a `pict` on the stack, do this:

    add     rsp, 24
	lea     rbx, [rbp - XXX]
	mov     r10, [rbx]
	mov     [rsp], r10
	mov     r10, [rbx + 8]
	mov     [rsp + 8], r10
	mov     r10, [rbx + 16]
	mov     [rsp + 16], r10

Here `XXX` is the location of the `pict` argument. Note that there is
no `REG` in this block. `pict` arguments do not take up a register
because they are passed on the stack.

If you know a bit of assembly, you can see that this copies 24 bytes
from the location of the `pict` argument to the top of the stack. Note
that you can't move from memory to memory. The data has to stop over
in a register.

On macOS, where the stack has to be 16-byte aligned, you will want to
add 32 bytes instead of 24. Make sure to make the corresponding change
after the call instruction too.

To call the function, use the assembly:

    call	_FNNAME
    
Here, `FNNAME` is the name of the function. Note the underscore.

After the function returns, if it took any `pict` arguments, you have
to remove them from the stack, like this:

    sub rsp, 24

If the function returns an `int`, you can move it to its location
using the following assembly:

    mov     [rbp - XXX], rax

If it returns a `bool`, move it like this instead:

    mov     [rbp - XXX], eax

If it returns a `float`, move it to its location like so:

    movsd	[rbp - XXX], xmm0

If it returns a `pict`, the function you call has already written it
to its final location, so there's nothing to do.
