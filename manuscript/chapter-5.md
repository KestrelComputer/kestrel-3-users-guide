# STS Application Binary Interface

NOTE: As of this writing, STS remains incomplete.  However, sufficient amounts of code exists to give you a crude flavor of what STS will be like in the future.  This chapter discusses STS V1.5 as of the time this chapter was published to LeanPub.

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

The following list of STS system calls covers everything from version 1.0 through to 1.5.  It's primarily of interest to people who studied earlier STS implementations and wish to compare Kestrel-2 versus Kestrel-3 runtime environments.

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
