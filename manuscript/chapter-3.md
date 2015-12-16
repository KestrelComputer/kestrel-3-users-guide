# STS Operating System

NOTE: As of this writing, STS remains incomplete.  However, sufficient amounts of code exists to give you a crude flavor of what STS will be like in the future.  This chapter discusses STS V1.5 as of the time this chapter was published to LeanPub.

STS is the Kestrel-3's native operating system.  It is an acronym, standing for Single-Tasking System.  Feature-wise, this places STS in competition with CP/M or MS-DOS.  It's intended to be easy to port to new architectures, even if you have to rewrite it from scratch.

Four versions of STS exist: versions 1.0 through 1.2 ran on the Kestrel-2 computer, while version 1.5 is the first version to run on the Kestrel-3.  This chapter discusses STS version 1.5.

## Building STS

When you download and install the Kestrel-3 emulator, it does not (yet) build STS by default, since it's still in active development.  To build it, you'll need to take additional steps:

    cd git/kestrel/3
    redo
    cd src/stsv1
    redo stsv1.rom

The first call to `redo` will provide you with some instructions concerning your `PATH` environment variable.  You'll need to set your path environment, so that the development tools used to make STS can be found.  You should just be able to type (or copy and paste) the example given verbatim and have it work.

## Running STS

Once your path is established, the second call to `redo` actually builds the ROM image.  This will produce a roughly 1MB file which you can pass to the emulator:

    e romfile stsv1.rom

This will kick off the emulator and drop you into an STS interactive shell environment.  On my computer, it shows this output:

    kc5tja@enif:~/git/kestrel/3$ redo
    redo  all
    redo    bin/e
    ... output elided for brevity ...
    redo        src/sdtest/sdtest.asm
    redo          bin/bspl.fs
    Be sure to add bin/ to your $PATH if you haven't already.
      export PATH=$PATH:/home/kc5tja/git/kestrel/3/bin
    kc5tja@enif:~/git/kestrel/3$ export PATH=$PATH:/home/kc5tja/git/kestrel/3/bin
    kc5tja@enif:~/git/kestrel/3$ cd src/stsv1
    kc5tja@enif:~/git/kestrel/3/src/stsv1$ redo stsv1.rom
    redo  stsv1.rom
    redo    stsv1.asm
    kc5tja@enif:~/git/kestrel/3/src/stsv1$ e romfile stsv1.rom
    This is e, the Polaris 64-bit RISC-V architecture emulator.
    Version 0.2.4.

    Booting STS V1.5
    ... built-in self-tests elided for brevity ...

        #####     ######    #####  
       ##   ##      ##     ##   ## 
      ##           ##     ##      
       #####       ##      #####  
          ##      ##          ## 
     ##   ##      ##     ##   ## 
     #####       ##      #####  
                                
              STS V1.5

    Copyright 2014-2015 Samuel A. Falvo II, et. al.

    This software is subject to the terms of the Mozilla Public License, v. 2.0.
    If a copy of the MPL was not distributed with this file, you can obtain one at
    https://mozilla.org/MPL/2.0/ .
    > _

As with the machine language monitor, you can exit the emulator at any time by pressing CTRL-C on the console.

## Editing Commands

You can type commands into the STS shell simply by typing characters on the keyboard.  If you make a mistake, you can use the backspace, DEL, or CTRL-H keys to backspace, depending on how your operating system's keyboard mapping is configured.  (For example, under Ubuntu Linux 15.04 with its default configuration, I have to use CTRL-H, because both backspace and DEL produce incorrect control codes.  This can, of course, be fixed; however, it might result in unusual keyboard behavior for other applications that ship with or is made for that operating system.)

For example, let's suppose you want to run the `hello` program, to print a greeting to the screen.  You can do this by typing the following text:

    /rom/hello

and pressing ENTER.  But, if you make a mistake,

    /rom/hlelo

you'll need to backspace to the first `l`, and retype the remainder of the command line.

NOTE: You'll notice a lack of more sophisticated editing commands.  E.g., CTRL-A, CTRL-E, CTRL-F, the up- and down-arrow keys, etc. do not work.  Not only that, but typing them may well cause the shell to act strangely.  For example, on my development computer, typing `/rom//`, tapping the backspace key, then continuing with `hello` will cause STS to report that `/rom/hello` is not found, even though that apparently is the exact same command I had you type above.  This is more to do with the incompleteness of STS itself rather than my ambitions for the OS.  At some point in the future, after higher priority tasks are resolved, I will revisit the user interface and add more convenience features.

## Structure of a Command

If you've used MS-DOS, Tripos, Unix, or other command-line environments before, you'll probably be reasonably familiar with the structure of a command in STS.  For those who might not be, this is how a command typically breaks down:

    /rom/m2   /rom/m2.slides
    =======   ==============
      |         |
      |         +-- Information to be passed to the command.
      |             These are called parameters, since they parameterize
      |             the behavior of the command.
      |
      +-- This is the program to invoke on your behalf.


Each command is responsible for parsing out its own parameters as needed, so not all commands accept the same kinds of parameters, or even if they do, in the same order.  You'll need to consult either the STS command reference or the reference material for your 3rd-party software to learn the proper incantation necessary to call upon its services.

## Volumes, Directories, and Files

A book contains a number of chapters.  These chapters typically will contain information that you can read and enjoy.  This book might, then, be part of a related series of books; think of an encyclopaedia for an example.  In this case, how do you identify the appropriate book out of the collection?  We speak of such texts as *volumes*.  Programs, their data files, and configuration settings need to be stored somewhere, as well.  Since electronic storage media store a plurality of these things, by way of analogy to books, these media are also called *volumes*.

Each volume contains a table, called a *directory* or, synonymously, a *volume table of contents* (VTOC), that tells STS what information is available on the volume.  Each unit of information is given a name, and is associated with a location in the storage medium's addressible space where to find the information itself.  These units of information are known as *files*.

    +-----------------+
    | Volume: myDisk  |
    |=================|
    | dir             |
    |  +-----------+  |
    |  | program   |  |
    |  | to list a |  |
    |  | directory |  |
    |  +-----------+  |
    |                 |
    | copy            |
    |  +-----------+  |
    |  | program   |  |
    |  | to copy   |  |
    |  | files     |  |
    |  +-----------+  |
    |                 |
    | ...etc...       |
    +-----------------+

## Filenames

To identify a file, you must provide its name in a format that STS can understand.  A filename has the following structure:

    /rom/m2.slides
     --- ---------
      |     |
      |     +-- The name of the file sitting on the volume named "rom".
      |
      +-- The volume containing the file we want ("rom" in this case).    

To access the file named above, STS would need to first isolate the `rom` volume from the others that may be in the system.  This step tells STS *how* to access the storage medium, as different volumes may be stored on different devices, each with their own rules for access.  It also disambiguates one device of the same kind from another, should more than one exist.  Once knowledge of how to access the volume is known to STS, it uses this information to try and locate the file named `m2.slides`.

If neither `rom` nor the file `m2.slides` is accessible on a volume named `rom`, then an error will be shown to the user.For example, if you execute the command:

    /rom/m2 /rom/aldhalsjkdfh

or some other random filename unlikely to be present, you'll receive the following error:

    File open failed.

## Qualifiers

STS implements only a single directory level.  This might become a problem if you have a lot of related files.  For example, suppose you have three meetings to go to, each needing their own slide deck.  How do you name these slide decks?  One approach is to use *qualifiers*, common prefixes that, among other uses, identify the kind of file you're working with.  For example:

    m2.slides
    m2.slides.bizdev
    m2.slides.sales
    m2.slides.kestrel3

The first component, `m2`, is known as the *high-level qualifier*, often abbreviated to HLQ.  In this case, we contextually know that these files are intended to be used by the Milestone-2 application.  The last component is often called the *low-level qualifier*, or LLQ.  This is what tells you which specific kind of file it is.  

I should take this moment to point out that STS itself does not have any awareness of qualifiers or their semantic significance.  Qualifiers are strictly intended for human consumption, a tool to help you organize your files.  However, that said, *individual programs* that run under STS may assign significance to some qualifiers.  For example, the command shell in STS V1.0 through V1.2 required all programs to have the `PRG.` prefix, even though the STS kernel couldn't care less.

## Command Reference

The following sections provide a synopsis of the commands that currently ship with the STS environment.

### /rom/hello

    /rom/hello

A simple program that displays a greeting to the user.  This program is intended to make sure basic console output works as expected.

### /rom/m2

    /rom/m2 deckfile

Displays the slide presentation `deckfile`.  This program is interactive, and will continue running until you leave the program.  To leave the program, type `0` (zero) on the console.  This will quit the program and return you to the STS shell.

When the program starts, the first slide of the deck will be shown.  To advance to the next slide, press `3`.  This may continue until you reach the last slide.  To retreat to the previous slide, press `2`.  To quickly jump to the first or last slide, use `1` (first) or `4` (last) on the console.

