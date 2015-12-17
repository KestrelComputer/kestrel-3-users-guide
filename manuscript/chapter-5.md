# STS Application Binary Interface

NOTE: As of this writing, STS remains incomplete.  However, sufficient amounts of code exists to give you a crude flavor of what STS will be like in the future.  This chapter discusses STS V1.5 as of the time this chapter was published to LeanPub.

This document attempts to establish the operating environment
used by a program running under the STS operating system.

* Status: **DRAFT**
* Date: Fri Nov 20 17:40:11 PST 2015

## Program Format

An executable file consists of an 8-byte header,
followed by the actual software.

                dword   $0000000000008067
        start:  ; code follows.

An assembler which produces only raw binary output
is sufficient to generate the header, as above,
or, it may generate it through the following sequence:

                jalr    x0, 0(x1)
                word    0
        start:  ; code follows

## Code, Data, and BSS Segments

This executable format
does not provide
for initialized or uninitialized
data segments as distinct entities.
The entire contents of the file,
including the header,
is treated as a single, opaque block of code.

Initialized data typically appears
colocated with executable code
in the file.
While not an optimal solution,
particularly for systems employing memory management hardware,
it significantly simplies the implementation
of STS itself.

Uninitialized data
must be explicitly allocated
by the program itself.

This implies that,
except under highly specialized situations,
STS' executables are not re-entrant:
the code images cannot be shared
across different processes
without employing some form of copy-on-write solution.

This also implies
that the code image in memory
MUST
be writable,
so that the application may save critical state,
such as the base address to its self-allocated, uninitialized data segment.

STS executables can be loaded at any dword-aligned location in memory.
RISC-V instructions are position-independent by intention,
so no need exists to relocate the loaded image.

## Initial Conditions

When a program is launched under STS, the registers are set
as follows, according to the ABI established by the current
BSPL compiler.

        PC              Base address of the header, plus 8.
        X1              Return Address to whatever launched your program
        X2              Data Stack Pointer
        X3              Return Stack Pointer
        X4              Global Variables Pointer
        X5..X15         Undefined; best not to use them without saving first.
        X16..X31        Undefined; these needn't be saved.

### PC

Execution of an STS binary commences
exactly 8 bytes following the `JALR` header.
Given that the header is kept as part of the loaded image,
and if we see a module is loaded at address `$123400`,
then the initial PC register will be `$123408`.

### Return Address (X1)

Per current BSPL compiler implementation details,
the X1 register conventionally holds the return address of a subroutine.
Since the caller invokes a binary as a subroutine,
this implies that X1 contains the return address.
The caller may or may not be STS itself.

The contents of this register MUST be saved
prior to use by the application.

### Data Stack Pointer (X2)

The X2 register will point
to the top of the caller's data stack.
The caller is typically, but need not be, STS V1.5 itself.

The stack will contain a parameter block,
providing the called program with enough information
to productively use the STS operating system services.

Note that the size of the caller's data stack isn't known
by the called program.
The caller MUST arrange at least 64 bytes for the called program's use.
If more than 64 bytes will be needed,
it's in the program's best interest
to allocate its own data stack.

Before returning to the binary's caller,
X2 will need to be restored with this initial value.


### Data Stack Contents

The calling program MUST place some parameters on the data stack.
This information provides the called application
with the system-level linkage it needs to
follow the user's specific instructions through command parameterization,
and to actually invoke STS system calls.

The linkage structure MUST be laid out as follows on caller's data stack:

        DSP+0           Pointer to STS system call jump table.
        DSP+8           Return code (defaults to 0).
        DSP+16          Undefined; reserved for the caller's use.
        DSP+24          Length of the command name.
        DSP+32          Address of the command name.
        DSP+40          Length of the command tail.
        DSP+48          Address of the command tail.

#### STS System Call Jump Table

This field points to a simple array of `JAL` instructions,
each corresponding to a STS system call entrypoint.
Each endpoint is named for the appropriate system call,
prefixed with `STS_`, and rendered in all upper-case letters.
For example, the offset for `getmem` is written `STS_GETMEM`.
The system call reference follows later in this chapter.

The calling program **may** provide its own implementation
of the STS system calls.
The called program can never trust the jump table address it receives
to be the one that the STS kernel itself provides.
However, it *can* trust that it offers completely compatible semantics.
This allows some programs to encapsulate STS features for other programs.

For example, a trace utility can launch another STS program,
providing its own system call table to capture and log system calls,
as `strace' does for Linux applications.
Yet another program may provide cooperative multitasking,
by switching tasks on all potentially long-running I/O operations.
Still others may provide sandboxing rules for untrusted programs,
limiting access to filesystem resources
or imposing quotas on memory consumption.
The STS kernel itself need not explicitly support these features.

#### Return Code

If your program wishes
to return a result code
to the calling program,
such as a command shell,
you'll need
to update the value
at DSP+8 with
the appropriate value
prior to returning.
This value will be interpreted by the calling program.
It defaults to zero,
usually taken to mean "success",
so programs don't need to explicitly change it
unless it wants to communicate a failure.

#### Command Name and Tail

The command name address and length fields
lets the command discover the filename used to invoke the program.
This typically has two uses:

1.  If you use symbolic links to create a number of aliases for a single binary, the program may select what function to perform based on how it was invoked.
2.  If the program generates an error, this information is used to provide a customized usage report.

The command tail address and length fields
provides user-supplied arguments to the command.
Neither STS nor its shells interpret the command tail.

For instance, let's suppose you write the following command on the shell:

        copy from /rom/* to /ramdisk all

The program, when run, will have the command name of `"copy"` (sans quotes)
and the tail set to `" from /rom/* to /ramdisk all"`.
Observe the leading whitespace on the tail.
Printing the command name and tail, in that order,
with no intervening prints, will print the command exactly as typed.

Although STS does not interpret command names, tails, or results,
standards surrounding their use is encouraged.
If any such standards exist, they will be documented in separately.

### Return Stack Pointer (X3)
The X3 register will point
to the top of the caller's return stack.
The caller is typically, but need not be, STS V1.5 itself.

Note that the size of the caller's return stack isn't known
by the called program.
The caller MUST arrange at least 64 bytes for the called program's use.
If more than 64 bytes will be needed,
it's in the program's best interest
to allocate its own return stack.

Before returning to the binary's caller,
X3 will need to be restored with this initial value.

### Global Variables Pointer (X4)
The `JALR`-header executable file format described here
provides no support for BSS or uninitialized data segments.
However, such segments are desirable for proper operation.

X4 typically points inside a 4KiB block of memory reserved
for global variables.
BSPL global variables are allocated from this block of memory.
Allocations start at offset -2048 and continue upwards in memory.
The RISC-V indexed addressing mode only supports 12 bits of displacement,
hence the 4KiB limitation.
X4 points in the middle because RISC-V sign-extends all displacements.

It is the responsibility of the called program to create its own
uninitialized data segment.
This implies that this register MUST be saved during program start-up.

This pointer MUST be restored prior to returning to the caller.

### Reserved Registers (X5..X15)

These registers are currently not allocated by the BSPL runtime
or compiler.  They may therefore be used by the application as it sees fit.
However, be aware that sub-programs may also use them.
This includes STS itself.

Therefore, if these registers are going to be used by your application,
make sure you save and restore them appropriately.

### Evaluation Registers (X16..X31)

These registers are used by BSPL to simulate a data stack for expression evaluation purposes.
Therefore, these registers may be used for any purpose by the application.
Unlike X5 through X15, though,
convention dictates that any control flow
invalidates the contents of these registers
(unless you or your compiler can statically determine their stability).

## System Calls

### close    

    \ BPSL

    scb @ >d  close

    ; Assembly Language

        addi    dsp, dsp, -8
        sd      scb, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_CLOSE(x16)

Close releases the resources acquired in the process of opening a file.  The SCB reference must be that which was returned by the corresponding `open` call.

### emit     

    \ BSPL

    character @ >d  emit

    ; Assembly Language

        addi    dsp, dsp, -8
        sb      character, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_EMIT(x16)

Emit sends a single character to the user's display.  The cursor is moved along, as though typing the character on a teletype device.

### filsiz   

    \ BSPL

    scb @ >d  filsiz  d> size !

    ; Assembly Language

        addi    dsp, dsp, -8
        sd      scb, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_FILSIZ(x16)
        ld      size, 0(dsp)
        addi    dsp, dsp, 8

Filsiz returns the size of the open file, in bytes.  The current read/write position is not altered.

### fmtmem   

    \ BSPL

    baseAddr @ >d  regionSize @ >d  fmtmem

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      baseAddr, 8(dsp)
        sd      regionSize, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_FMTMEM(x16)

fmtmem formats a memory pool with the required metadata to support allocation requests.  The `baseAddr` variable must point to the start of the
memory pool, while `regionSize` must contain its size, in bytes.  `fmtmem` does not return any results, and cannot fail.

**Beware**: formatting a pool that's been previously used will, in effect, forcefully deallocate everything from that pool.

**As of STS V1.5, this procedure is internal to STS.**  New code should *not* invoke this procedure.

### fremem   

    \ BSPL

    allocatedMem @ >d  fremem

    ; Assembly Language

        addi    dsp, dsp, -8
        sd      allocatedMem, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_FREMEM(x16)

Fremem releases a block of memory, whose pointer was returned by a previous call to `getmem`.

### getkey   

    \ BSPL

    getkey  d> key !

    ; Assembly Language

    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_GETKEY(x16)
        lbu     character, 0(dsp)
        addi    dsp, dsp, 8

Blocks until a valid ASCII character has been received from the user's keyboard.  Note that getkey does **not** offer a timeout!  If non-blocking operation is preferred, you should use `polkey` to poll for data availability before invoking `getkey`.

Example:

    dword: keepRunning

    : handleKey ( k -- )
        0 d@ h# 51 - 0=if   ( did user press Q? )
            d# 0 keepRunning ! exit
        then
        ... ;

    : doBackgroundProcessing
        d# 1 keepRunning !
        begin keepRunning @ while
            polkey d> if getkey handleKey then
            doBackgroundStuffHere
        again ;

### getmem   

    \ BSPL

    blockSize @ >d  getmem  d> success !  d> allocatedMem !

    ; Assembly Language

        addi    dsp, dsp, -8
        sd      blockSize, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_GETMEM(x16)
        lb      success, 0(dsp)
        ld      allocatedMem, 8(dsp)
        addi    dsp, dsp, 16

Getmem allocates at least `blockSize` bytes of memory.  If successful, it returns the address of the allocated block along with a non-zero flag.  Otherwise, the flag will be zero, and the `allocatedMem` value will be undefined.

### getver   

    \ BSPL

    getver  d> major !  d> minor !  d> patch !

    ; Assembly Language

    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_GETVER(x16)
        ld      major, 0(dsp)
        ld      minor, 8(dsp)
        ld      patch, 16(dsp)
        addi    dsp, dsp, 24

Getver returns the (semantic-compatible) version number of the STS kernel.  This does not necessary correspond to the versions of other components in a running STS system.

### loadseg  

    \ BSPL

    S" /sd0/PRG.helloWorld" >d >d  loadseg  d> errCode !  d> segPtr !

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      filenameAdr, 8(dsp)
        sd      filenameLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_LOADSEG(gp)
        ld      errCode, 0(dsp)
        ld      segPtr, 8(dsp)
        addi    dsp, dsp, 16

Loadseg attempts to load a program into memory, and if required for the executable format, perform appropriate relocations so that the loaded code may be executable in-place.  If successful, a reference to the first (of potentially many) segments will be returned, along with a zero error code.  Otherwise, the segment reference will be undefined, and the error code will be non-zero.

Note that the segment reference returned points to the first actual byte of data or code; segment-specific linkage appears at negative offsets from this reference.  Therefore, segment references **cannot** be passed to `fremem`.  Use `unloadseg` instead.

Note further that `loadseg` does **not** execute the program.  You must arrange for this to happen through other means.

### movmem   

    \ BSPL

    srcBuf @ >d  dstBuf @ >d  bufLen @ >d  movmem

    ; Assembly Language

        addi    dsp, dsp, -24
        sd      srcBuf, 16(dsp)
        sd      dstBuf, 8(dsp)
        sd      bufLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_MOVMEM(gp)

Moves a block of memory from `srcBuf` to `dstBuf`.  The block moved will consist of `bufLen` bytes.  Currently, this procedure does not handle overlapping blocks of memory.  It implements a simple, slow, byte-granular, ascending memory move.  However, future releases of STS may improve on this procedure's implementation.

### open     

    \ BSPL

    S" /sd0/DATA.helloWorld" >d >d  open  d> errCode !  d> scb !

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      filenameAdr, 8(dsp)
        sd      filenameLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_OPEN(gp)
        ld      errCode, 0(dsp)
        ld      scb, 8(dsp)
        addi    dsp, dsp, 16

Open attempts to resolve the given filename to a resource on a volume, and asks the resource to create a new connection to it.  If successful, a zero error code and a stream control block (SCB) are returned to the caller; otherwise, a nil result is returned, along with a non-zero error code.

The provided filename *must* be formatted to include both the volume and the filename (e.g., `/vol/filename.here`).  Anything else will yield an error.

### polkey   

    \ BSPL

    polkey  d> pending !

    ; Assembly Language

    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_POLKEY(x16)
        lb      pending, 0(dsp)
        addi    dsp, dsp, 8

Polkey returns non-zero if `getkey` will *not* block.  Put another way, this procedure returns non-zero if at least one ASCII character is available to be read from the keyboard.

### read     

    \ BSPL

    dstBuf @ >d  bufLen @ >d  scb @ >d  read  d> errCode !  d> actual !

    ; Assembly Language

        addi    dsp, dsp, -24
        sd      dstBuf, 16(dsp)
        sd      bufLen, 8(dsp)
        sd      scb, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_READ(x16)
        ld      errCode, 0(dsp)
        ld      actual, 8(dsp)
        addi    dsp, dsp, 16

Read attempts to read the next sequence of bytes from the referenced stream.  If the resource is not readable, an error will result.  Otherwise, the actual number of bytes read will be returned.  Note that an actual byte count of zero is valid; with a zero error code, this implies you've either reached the end of a file, or for non-file types of resources, that no data is yet available.

### seek     

    \ BSPL

    position @ >d  scb @ >d  seek  d> errCode !  d> oldPos !

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      position, 8(dsp)
        sd      scb, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_SEEK(x16)
        ld      errCode, 0(dsp)
        ld      oldPos, 8(dsp)
        addi    dsp, dsp, 16

Seek attempts to set the read/write pointer for the stream at the indicated position.  If this is not possible, a non-zero error code will be returned.  Otherwise, the error code will be zero, and the previous position is returned.

### setmem   

    \ BSPL

    dstBuf @ >d  bufLen @ >d  character @ >d  setmem

    ; Assembly Language

        addi    dsp, dsp, -24
        sd      dstBuf, 16(dsp)
        sd      bufLen, 8(dsp)
        sb      character, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_SETMEM

Setmem attempts to set a block of memory to an arbitrary byte value.  Every character, starting with the one at `dstBuf` and extending up to `dstBuf+bufLen-1` will be set to `character`.

### strDup   

    \ BSPL

    srcBuf @ >d  bufLen @ >d  strDup  d> success !  d> newString !

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      srcBuf, 8(dsp)
        sd      bufLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_STRDUP

StrDup attempts to duplicate a string in memory.  The only way this can fail is if no memory is available for allocation.  In this case, a zero success code is returned, and the new string pointer will be undefined.  Otherwise, the success flag will be non-zero, and the new string pointer will contain a copy of the provided string.  STS guarantees that `newString` will be at least as long as `srcBuf`, but it *may* produce a larger buffer.

### strEql   

    \ BSPL

    S" Test String" >d >d  srcBuf @ >d  bufLen @ >d  strEql  d> yesNo !

    ; Assembly Language

        addi    dsp, dsp, 32
        sd      aStringPtr, 24(dsp)
        sd      aStringLen, 16(dsp)
        sd      bStringPtr, 8(dsp)
        sd      bStringLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_STREQL
        ld      yesNo, 0(dsp)
        addi    dsp, dsp, 8

StrEql compares two strings for equality.  If they match, byte for byte, true is returned.  Otherwise, false is returned.

### type     

    \ BSPL

    S" Hello World!" >d >d  type

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      srcBuf, 8(dsp)
        sd      bufLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_TYPE

Sends a complete string found at `srcBuf` and of length `bufLen` to the user's console.  Note that this bypasses the STS file I/O mechanism.

### unloadseg

    \ BSPL

    seg @ >d  unloadseg

    ; Assembly Language

        addi    dsp, dsp, -8
        sd      seg, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_UNLOADSEG

Unloadseg will remove a program that was loaded via `loadseg` from memory.  Note that no reference checks are made; this procedure will remove the program unconditionally.  It is the caller's responsibility to ensure that nothing else in the running STS system references the program to be unloaded.

### write    

    \ BSPL

    srcBuf @ >d  bufLen @ >d  scb @ >d  write  d> errCode !  d> actual !

    ; Assembly Language

        addi    dsp, dsp, -24
        sd      srcBuf, 16(dsp)
        sd      bufLen, 8(dsp)
        sd      scb, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_READ(x16)
        ld      errCode, 0(dsp)
        ld      actual, 8(dsp)
        addi    dsp, dsp, 16

Write attempts to write the next sequence of bytes to the referenced stream.  If the resource is not writable, an error will result.  Otherwise, the actual number of bytes written will be returned.  Note that an actual byte count of zero is valid; with a zero error code, this implies that the resource you're writing to is not yet ready for more data, and you should probably try again later.

### zermem   

    \ BSPL

    ; Assembly Language

        addi    dsp, dsp, -16
        sd      dstBuf, 8(dsp)
        sd      bufLen, 0(dsp)
    L:  auipc   gp, 0
        ld      x16, stsBase-L(gp)
        jalr    ra, STS_ZERMEM

Zermem fills a region of memory starting at `dstBuf` and extending to `dstBuf+bufLen-1` with zeros.  This is a convenience API: it's equivalent to `dstBuf @ >d  bufLen @ >d  h# 00  setmem`.


## System Call Compatibility Matrix

For those curious of the history of the STS ABI and API, the following compatibility matrix illustrates the evolution over time.  It covers everything from version 1.0 through to 1.5, the latest release as of this publication.  It's primarily of interest to people who studied earlier STS implementations and wish to compare Kestrel-2 versus Kestrel-3 runtime environments.

|System Call|1.0|1.1|1.2|1.5|
|:---------:|:-:|:-:|:-:|:-:|
|getver   |No|No|No|Yes|
|emit     |No|No|No|Yes (2)|
|type     |No|No|No|Yes (2)|
|polkey   |No|No|No|Yes (2)|
|getkey   |No|No|No|Yes (2)|
|fmtmem   |Yes|Yes|Yes|Yes|
|fremem   |Yes|Yes|Yes|Yes|
|getmem   |Yes|Yes|Yes|Yes|
|movmem   |No|No|Yes (1)|Yes|
|setmem   |No|No|No|Yes|
|zermem   |No|No|No|Yes|
|strDup   |No|No|No|Yes|
|strEql   |No|Yes (1)|Yes (1)|Yes|
|open     |Yes|Yes|Yes|Yes|
|close    |Yes|Yes|Yes|Yes|
|read     |Yes|Yes|Yes|Yes|
|write    |No|No|No|Yes (3)|
|filsiz   |No|No|No|Yes|
|seek     |No|No|Yes (1)|Yes|
|unloadseg|Yes|Yes|Yes|Yes|
|loadseg  |Yes|Yes|Yes|Yes|

Notes:

1.  The same capability existed, but went under a different name and/or had a different calling convention.

2.  It remains unclear if these procedures will remain in future STS versions.

3.  Planned for future release.
