# The Machine Language Monitor

The Machine Language Monitor, or MLM, serves as a software equivalent to a computer's "front panel".  It lets you inspect and alter memory, allowing you to write programs for the computer using the computer itself.  It also lets you inspect and alter the microprocessor's available registers, which is often required when trying to solve problems with erroneously entered software (a process known as *debugging*).

The MLM is a very low-level interface to the computer; literally, just one step above actually flipping physical switches to operate the computer.  While operating systems and other kinds of system software exist at some level to abstract the computer away from the user, the MLM fully exposes the critical features of the computer instead.  It truly is the simplest possible system software that succeeds in letting the user manage and operate the computer.

The MLM that comes with the Kestrel-3 exposes the following resources to the user:

- Two kinds of memory: so-called RAM and ROM.
- CPU Registers.

The following sections explain what these resources are and how best to understand them.

After reading this chapter, you should be able to:

- Understand what computer memory is, and how it's roughly organized in the Kestrel-3.

- Understand the difference between bits, nybbles, bytes, half-words, words, and double-words.

- Understand some of the differences between a real, hardware Kestrel-3 and the standard Kestrel-3 emulator.

- Understand how to use the Kestrel-3's Machine Language Monitor to enter and run programs.

What this chapter will *not* cover is how to translate RISC-V instructions into the numbers you'd use to plug into the MLM.  By its nature, learning how to write machine language programs requires longer exposition and more detailed explanations.  Therefore, I'll defer this topic for another chapter.

## Memory

Computer memory consists of lots of tiny cells, which are called *bits*.  Each bit can hold one of two values: 0 or 1.  Think of a bit as a square on a chess board.  It either holds a piece (1) or it doesn't (0).

A bit can be valuable for telling a program whether or not something has happened, or if something exists or not.  Much beyond that, however, individual bits don't hold a lot of valuable information.  It's only when we use collections of them that we may begin to encode more useful, richer kinds of information.

Unlike a chess board, however, a computer's memory is laid out sequentially, like so:

    0100100001001001 ... etc

As you can see from the example above, a bunch of 0s and 1s put together doesn't really convey much value on its own.  You and the computer both need some meaningful way of identifying where useful information begins and ends.

### Introducing Bytes, Nybbles, and Hexadecimal Numbers

This is where the notion of a *byte* comes in.  Instead of inventing some ultra-clever way for a computer to say, "OK, here's where the start of my information begins," we arbitrarily *declare* that useful information will be *structured* in groups of, say, eight bits.  (To be fair, this ultra-clever technology *does* exist; however, it's mostly used for telecommunications and not for online processing of data.)

If we take a look at our previous example above, except grouping them into bytes, we still cannot divine much information about the blob of data, but we *can* see an inkling of a pattern emerging.

    01001000 01001001 ... etc

Indeed, we can separate the data down further, into groups of 4 bits, like so:

    0100 1000 0100 1001 ... etc

As it happens, working with 4-bit quantities like this, each one called a *nybble* by the way, proves especially convenient for human beings, but not necessarily as 1s and 0.  Instead, we assign a number or letter to each of these groupings, according to the table below.

    What's really       How we write it
    in the computer     for convenience
        0000                    0
        0001                    1
        0010                    2
        0011                    3
        0100                    4
        0101                    5
        0110                    6
        0111                    7
        1000                    8
        1001                    9
        1010                    A
        1011                    B
        1100                    C
        1101                    D
        1110                    E
        1111                    F

*Note:* There is a reason behind this ordering, and it has to do with something called base-2 arithmetic.  I'll just skip the explanation of what this means for now.  You *will* eventually need to learn it in order to write software for *any* computer, and the Kestrel-3 is no different; however, we can cross that bridge when it's actually important to have that knowledge.

Going back to our previous example, then, we can write each nybble more conveniently and compactly by replacing each group of four bits with its equivalent symbol from the table above:

    4 8 4 9 ... etc

Each of these digits is called a *hex digit*, short for *hexadecimal* digit, or base-16 digit.  That's a big word that basically says your digits use a set of 16 possible symbols, instead of our usual set of 10 (what is known as base-10, or just decimal).

You should by now understand an important insight at this point in time: computers don't care what the bits represent.  Human beings do.  All computers do is manipulate those bits in ways *we*, as people, think will be useful.  Back in the 1980s when I first started learning this stuff, there was an initialism in common use called GIGO, short for Garbage In, Garbage Out.  It's a nice sound-bite, but it doesn't fully emphasize the importance that humans have on the *context* required to interpret the information you see above.  Is it an address?  Is it text?  Is it weather-related?  We can't tell by looking at the blob of data, no matter how it's represented to the reader.  Only when the reader has enough context can he or she make sense of those numbers.

Until now, I haven't given you the context for this data.  However, let's do that now.  Suppose we have some a priori knowledge that tells us we're looking at *text*.  If we know that this represents text, probability is good that you're looking at something compatible with ASCII information.  We can use an ASCII chart (see appendix) to decode what each byte means.  

Let's go back to looking at the data as a set of bytes, but this time keeping the hexadecimal form:

    48 49 ... etc

As it turns out by reading the ASCII chart, these two bytes form the letters H and I, in that order.

Hi!

The act of reading bytes and interpreting them is what programmers call *reading a memory dump* (sometimes also referred to as a "core dump", largely for historical reasons), where you combine raw hexadecimal numbers written to your display with your contextual knowledge to make sense of what the computer's trying to tell you its memory contains.

But, what about larger numbers than eight bits?  There's no fundamental reason why we couldn't use 16-, 24-, 32-, or even larger numbers.  In fact, some sizes occur so frequently that we assign them names.  A *word* is defined to be 32-bits on any RISC-V compatible processor, in part because that's how big a RISC-V instruction happens to be.  However, it's equally adept at working with 16-bit quantities (*half-words*), and with 64-bit quantities (*double-words*).  We'll often use abbreviations: hwords, words, and dwords, respectively.  So, for example, since a memory address is expressed with 16 nybbles (64 bits), we can fit that address into a dword-sized region of memory.

Hopefully, you know what bits, nybbles, bytes, and the various kinds of words are.  Next, you'll need to know the two different kinds of memory that the Kestrel-3 exposes to a programmer.  This is important because if you attempt to store a program into the wrong kind of memory, you'll be unpleasantly disappointed with the results.


### RAM and ROM

Before learning how to interact with the MLM, you need to know about the resources it gives you access to.  The Kestrel-3 contains many different kinds of memories; however, only two are of concern to us right now: RAM and ROM.

RAM is an acronym standing for Random-Access Memory.  It holds both the instructions and the data needed to implement a computer program, or it can serve as a temporary storage depot for other kinds of data that a program works with.  For example, the description of what you see on a video display will be stored in RAM.

ROM is an acronym standing for Read-Only Memory.  It also holds both the instructions and the data needed to implement a computer program.  However, if I may be permitted to over-generalize here, it differs from RAM in that its contents remain fixed; it cannot change, even when the power is turned off.  For this reason, we store a computer's system software in ROM, so that it's there when we turn the power to the computer on.

RAM differs from ROM in that you can alter its contents; but, of course, it loses its contents completely when power is turned off.  Also, its name is an historical accident, for ROM also supports random access.



### What's Special About Random Access?

Random access means that the microprocessor is free to read or alter any byte from memory that it desires, whenever it desires.  For example, when the computer is turned on for the first time, the Kestrel-3 will start reading instructions from location $0000000000002000 (or, in decimal, 8192).  It did not have to read locations 0 through 8191 first to do so.  This saves a great deal of time, as you can imagine, as the computer often must process information from widely separated locations in memory.

Sequential access, on the other hand, is what it reads on the tin.  To access a byte of memory at some arbitrary location, you must first read (and, usually, discard) all preceding bytes.  Sequential accesses typically happen when communicating with I/O devices.  I won't discuss it furher here, but be aware that various sequential access techniques exist and are useful in appropriate contexts.



### Memory Addressing and the Kestrel-3 Memory Map

How do you know if you're working with RAM or ROM?  For that matter, given the `48 49` sequence above, where do those numbers actually *come from?*

Remember our chess board approximation from before?  Remember how I'd said that they're laid out two-dimensionally?  If you notice, along one edge of the chess board, you'll find the letters A, B, C, D, E, F, G, and H.  Along the other edge, numbers from 1 through 8.  A player can specify any location on the board by just giving one letter, and one number.  In fact, if you ever played chess on a computer before, you probably used this technique to indicate the pieces you wanted to move.  This number and letter pair are said to be *coordinates*, but for our purposes, we call them *addresses*; with that number, we can "address" any element on the chess board.

Computer memory works the same way, but they use a single (potentially very large) number.

We call the microprocessor in the Kestrel-3 *byte-addressible.*  This means that the microprocessor can, with a suitably large enough number, identify where information is stored down to individual bytes.  Since the Kestrel-3 is a 64-bit machine, an address can be written with up to 16 nybbles.  This represents an enormous amount of space to put things.  Go ahead and calculate it out for yourself!  You have sixteen symbols, each of which can be any one of those 16 hexadecimal digits.  What number do you get?  That's how many different bytes of memory the Kestrel-3 is capable of addressing.  It's almost certainly far more than you'll ever need in a computer like the Kestrel-3.  Indeed, even enterprise-grade, big-iron computers, such as mainframes and distributed clusters, aren't expected to top out their 64-bits of space until after the year 2030!

When the computer powers on for the first time, the microprocessor needs to start executing software from a well-known location in memory.  The precise details of how this works falls outside the scope of this chapter; however, I will say that RISC-V compatible processors begin their search for software at address $0000000000002000.  Since ROM memory holds its information even when power is off, it makes sense to place the computer's system software at this location.  This explains why, if you read something called a *memory map*, which is something that tells you what kind of memory or hardware I/O devices can be found at which addresses, ROM appears starting at address $0000000000000000, and is large enough to cover up to address $0000000000002000.

I reproduce the Kestrel-3 emulator's memory map below:

    Starting from       You'll find this    and ending here,
    ----------------    -----------------   ----------------
    0000000000000000    ROM                 00000000000FFFFF
    0100000000000000    RAM                 0100000000FFFFFF
    0200000000000000    Unused as of V0.1   0EFFFFFFFFFFFFFF
    0F00000000000000    Debugger Port       0F00000000000001
    1000000000000000    Unused              FFFFFFFFFFFFFFFF
    ----------------    -----------------   ----------------

Contrast this against a proposed configuration for the Digilent Nexys2 version of the Kestrel-3 hardware:

    0000000000000000    ROM                 00000000000FFFFF
    0100000000000000    RAM                 0100000000FFFFFF
    0200000000000000    Keyboard and Mouse  020000000000001F
    0300000000000000    Video Controller    03000000000000FF
    0400000000000000    SD Card, GPIO       040000000000000F
    0500000000000000    Unassigned          0FFFFFFFFFFFFFFF
    1000000000000000    Expansion           FFFFFFFFFFFFFFFF

You'll find more detailed memory maps in the appendices.

Note that the emulator models a reasonably close subset of one particular hardware configuration for the Kestrel-3, but it is not a perfect match.  It's conceivable that future hardware configurations will have different memory maps still.  Since an address selects which hardware component(s) the CPU will talk to, it's easy to see that a memory map for one computer may well differ from that of another, even if they're part of the same family.  For this reason, some types of system software, such as operating systems, hardware abstraction layers (HALs), and Basic Input/Output Systems (BIOS), all exist to not only enable the user to interact with the computer, but also to help application software run without concern for a specific computer's hardware configuration.

### Dumping Memory At Last!

You now have the required background to dump the contents of memory to the screen and, with some practice, make some reasonable sense of the results.  To look at a particular byte in memory, you need only type its address, and use the `@` command, like so:

    * 0000000000002000 @

What comes back to you may be different, but it should look something like this response:

    0000000000002000:03 .  .  .  .  .  .  .  .  .  .  .  .  .  .  .
    * _

You may continue to look at subsequent addresses by simply repeating the process:

    * 0000000000002000 @
    0000000000002000:03  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .
    * 0000000000002001 @
    0000000000002000: . 13  .  .  .  .  .  .  .  .  .  .  .  .  .  .
    * 0000000000002002 @
    0000000000002000: .  . 80  .  .  .  .  .  .  .  .  .  .  .  .  .

As you can imagine, it's *really* inconvenient to inspect memory one byte at a time like this.  You can use `.` to tell the MLM to inspect a range of memory:

    * 0000000000002000.0000000000002003 @
    0000000000002000:03 13 80 00  .  .  .  .  .  .  .  .  .  .  .  .
    * _

You can look at any contiguous region of memory using this same approach.  If you want to display more than 16 bytes at a time, it will break the read-out into 16-byte chunks for easier reading, like so:

    (REPLACE ME WITH REAL DATA)
    * 0000000000002000.000000000000203F @
    0000000000002000:03 13 80 00 03 13 80 00 03 13 80 00 03 13 80 00
    0000000000002010:03 13 80 00 03 13 80 00 03 13 80 00 03 13 80 00
    0000000000002020:03 13 80 00 03 13 80 00 03 13 80 00 03 13 80 00
    0000000000002030:03 13 80 00 03 13 80 00 03 13 80 00 03 13 80 00
    * _

If we want to look at the contents of RAM instead of ROM, simply change the address (range) used.

    * 0100000000000000.010000000000003F @

You'll probably find that when looking at memory layouts like this, you come to rely on the relative position of information in the display.  To make it easier to read memory dumps with this technique, the MLM will pad a dump with periods as appropriate:

    * 0000000000002000.0000000000002003 @
    0000000000002000:03 13 80 00  .  .  .  .  .  .  .  .  .  .  .  . 
    * 0000000000002004.0000000000002007 @
    0000000000002000: .  .  .  . 03 13 80 00  .  .  .  .  .  .  .  . 
    * 000000000000200E.0000000000002013 @
    0000000000002000: .  .  .  .  .  .  .  .  .  .  .  .  .  . 80 00
    0000000000002010:03 13 80 00  .  .  .  .  .  .  .  .  .  .  .  .
    * _

This keeps the data you're interested in a consistent location on the screen at all times.

### Altering Memory

The `,` command changes a single byte of memory.  For example, if we wanted to store the text string `HI` into RAM, we could use a command like the following:

    * 0100000000010000.48,49,
    * _

You can verify the information is stored by dumping that region of memory again:

    * 0100000000010000.0100000000010001@
    0100000000010000:48 49  .  .  .  .  .  .  .  .  .  .  .  .  .  .
    * _

To prove to yourself that it works, try repeating the above exercise, except using memory starting at address $0100000000010002 and $0100000000010003.

Note that you can confirm to yourself that ROM is, truly, read-only by attempting to store data into low memory:

    * 80.8F@
    0000000000000080:CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC
    * 80.11,22,33,44,
    * 80.8F@
    0000000000000080:CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC CC
    * _

Pro-tip: If you know you're wanting to store a number larger than a byte, then you can type the number naturally, provided it fits inside of 64 bits, and use as many commas as required to store the whole number.  For example, a 16-bit number could be written as `4948,,`, a 32-bit number as `DEADBEEF,,,,`, and so on.  Unfortunately, no corresponding method exists for reading memory out as 16-bit or larger quantities.


### CPU Registers

At this point, you know enough to plug a program into memory by typing in the hexadecimal codes that comprise it, but we still don't know how to really communicate with such a program without actually altering the program itself, or without altering an otherwise reusable piece of code into something more tightly bound to its parameters.  Writing software much beyond simple toys will require us to write more general-purpose programs.  Doing that requires we use a different technique to "talk" to the program.

The CPU, the device responsible for interpreting the program you store in memory, has a special kind of memory of its own, called *registers*.  Each register can hold 64 bits worth of information, and it has 32 of them.  One, called the program counter (PC), tells the CPU where the next instruction to perform is coming from.  The reamining 31 registers are called *general purpose* registers.  Some of these registers are used for accounting purposes, like keeping track of subroutine linkage, while others are freely available for use to pass parameters into or out of a subroutine.  Exact details will depend on the program being run.

These registers are addressed by name, ranging from X1 through X31.  Note that a pseudo-register X0 does exist, but it's hardwired to the constant zero; you can never change it, a feature more useful than you might initially think.

When a program invokes the MLM, either through an SBREAK instruction or by using a JAL or JALR instruction to invoke the MLM directly, it *saves* the previously running program's registers in RAM for inspection.  You can use the `X` command to address this special location, and `:` to inspect it.  For example, to dump all 32 general purpose registers, you'd write:

    * 0X.1FX:

Here, instead of using `@` to inspect memory, we use `:`.  This is a special version of the memory dump command which knows how to compute the appropriate addresses used by the MLM to represent the interrupted program's registers.  Indeed, when the result appears on the screen, it looks like you had used the `@` command all along.

    * 0X:
    0100000000000000:00 00 00 00 00 00 00 00  .  .  .  .  .  .  .  . 
    * 3X:
    0100000000000010: .  .  .  .  .  .  .  . 00 00 00 00 00 00 00 00
    * _

As you might imagine, since the `X` command merely computes a memory address to work with, we can use it with the `,` command to effect changes to the interrupted program's registers as well.  For example, if we want to set X1 to the value $1111, and X2 to $2222, we would write:

    * 1X.1111,,
    * 2X.2222,,
    * _

Be careful not to double-up your Xs.  It will result in an incorrect address and can lead to memory corruption if you're not careful.


## Your First Program

Now that you know how to write program bytes into RAM, and how to pass that program information it needs to run via CPU registers, let's now program the Kestrel-3 to add two numbers.  Without going into the process of how I arrived at these numbers, you can enter the following command:

    * 0100000000010000.B3,81,20,00,13,62,10,40,13,12,32,00,67,00,02,00,
    * _

This program takes two numbers, in th X1 and X2 registers, and returns the result in X3.  So, let's plug in some values to add:

    * 1X.1234,,
    * 2X.3210,,
    * 3X.0000,,
    * _

This places the value $1234 in register X1, and $3210 in register X2, and finally, zero in X3.  We can confirm the registers are set accordingly like so:

    * 1X.3X:
    0100000000000000: .  .  .  .  .  .  .  . 34 12 00 00 00 00 00 00
    0100000000000010:10 32 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    * _

We execute the program by telling the monitor where to go:

    * 0100000000010000G
    MLM/K3 V1
    * _

We should be able to inspect the registers at this point:

    * 3X:
    0100000000000010: .  .  .  .  .  .  .  . 44 44 00 00 00 00 00 00
    * _

If the result isn't $4444, something went wrong.  Double-check that the numbers you typed above for the program are correct and try again.  If it matches the result give here, congradulations!  You've just written your very first Kestrel-3 program!


