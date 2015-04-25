# Introduction

Thank you for your interest in the Kestrel-3 computer.  I built the Kestrel-3 to be completely open-source, from the microprocessor's micro-architecture all the way up to the system software.  What does this mean for you, exactly?  It means that you have the ability to inspect and learn from every component making up the computer and its system software.  It also means that you have the ability to propose changes for future generations of the computer, provided you have the technical skills to implement those changes.  If you *don't* have the technical skills, well, I can't think of a better platform to develop them with.

While I cannot teach you all the tools and skills you'll need in a single, introductory document such as this, I do aim to provide a foundation of knowledge you'll need to be able to use the Kestrel-3 productively, write software for it if you're so inclined, and to have enough background to competently ask for help or alterations to be made to the hardware or system software.

After reading this chapter, you should be able to:

- Set up your Kestrel-3, either real hardware or its emulator.


## Installing the Kestrel-3 Computer Hardware

At the time of this publication, the Kestrel-3 does not exist in physical hardware form.  I will work on this for a future release of the Kestrel project.

## Installing the Kestrel-3 Emulator

The Kestrel-3 computer remains under development, and at the time of this writing, is not suitable for general purpose use.  However, if you're interested in contributing to the development of the Kestrel-3 software, particularly its emulator, development toolchain, or operating system, these instructions should be able to get you on your way.

My development environments include MacOS X 10.10, Linux Mint, and Ubuntu Linux (64-bit).  Below, I describe how to configure a bare-bones Ubuntu 14.04 distribution to support elementary Kestrel-3 development, and should be sufficient to let you follow along with the examples in the remainder of this book.

### Installing SwiftForth

I used SwiftForth to write the RISC-V assembler.  This assembler will eventually be used to construct the system firmware.

    wget http://www.forth.com/downloads/SwiftForth-linux-osx-eval.tgz
    tar xvzf SwiftForth-linux-osx-eval.tgz
    sudo ln -s /home/sfalvo/SwiftForth/bin/linux/sf /bin/sf

You should try to invoke the `sf` command now to see if it works.  It will be a 32-bit binary, so while it'll run on all 32-bit Linux distributions, you have about 50% chance that it will not run correctly on a 64-bit system.  This is because not all 64-bit Linux distributions come equipped with 32-bit compatibility turned on by default.

    $ sf
    SwiftForth i386-Mac OS X 3.5.6 26-Oct-2014
    bye
    $ _

If `sf` can be run successfully, you'll see the SwiftForth banner, and it will wait for your input.  Type `bye` to exit the interpreter.

If you do not see the banner, or you get an error message, chances are your distribution does not have 32-bit compatibility enabled.  If you're running **Ubuntu 14 or earlier**, the following instructions should be invoked at this time to turn it on.  These instructions *will not* work for Ubuntu 15 or later.  As soon as I learn more about how to enable support for 32-bit software in these later versions of Ubuntu, I'll list the steps here.

    sudo bash
    dpkg --add-architecture i386
    echo "foreign-architecture i386" > /etc/dpkg/dpkg.cfg.d/multiarch
    apt-get update
    apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
    exit

This will let your Linux distribution interoperate with the 32-bit SwiftForth binary.  You should be able to run the `sf` command again at this time as described above.


### Installing GCC

If you haven't done so already on your distribution, you'll need to install the GCC package.  This package provides a C compiler, which is what the emulator is written in.  If you already have GCC installed, it's safe to skip this step.

    sudo apt-get install gcc

### Installing Git

We use Git to manage the source code repository for the Kestrel-3 design.  You may skip this step as well if you already have Git installed.

    sudo apt-get install git

### Installing Redo

I chose Redo to provide build automation and dependency analysis.  For my needs, Redo has several advantages over Make:

* The `.do` files describing target rules appear where the target will appear in the filesystem.
* The `.do` files are simple shell scripts, directly executed by the `redo` command(s).

I find this helps keep my build descriptions focused and modular, albeit at the cost of losing the one-stop-shop benefit of having everything in a single Makefile.

    git clone https://github.com/apenwarr/redo
    cd redo
    ./redo
    sudo ./redo install

### Installing and Building the Kestrel Toolchain

At last, we have what it takes to put everything together and build a Kestrel emulator along with its firmware.

    git clone https://github.com/sam-falvo/kestrel
    cd kestrel/3
    redo

### Verifying Everything Works

You should now be able to invoke the emulator and see reasonable output:

    bin/e romfile roms/mlm

This should bring you to the following prompt:

    This is e, the Polaris 64-bit RISC-V architecture emulator.
    Version 0.1.0.
    MLM/K3 V1
    * _

If this message, or a reasonable facsimile thereof, doesn't appear on your computer, something went wrong; try looking for error messages and fixing them, then try again.

