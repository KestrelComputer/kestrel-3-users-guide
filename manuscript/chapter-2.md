# Getting to Know Forth

In this chapter, you'll have an understanding of how to write software in Forth for the Kestrel-3, *on* the Kestrel-3.  Unlike most programming tutorials, I'm just going to dive right into a non-trivial application: a simple font editor.  It's not terribly complicated as applications go; but, it will touch many of the facilities of the Kestrel-3 on different levels of abstraction, and will introduce everything from the most basic to some moderately advanced Forth programming idioms.  The program is presented in a manner that allows you to type in the program step by step and to follow along.  It's perfectly OK if you don't get things the first time around; feel free to revisit this chapter again later.  As always, if you have questions which I'm not answering, or if you feel this chapter can be improved, feel free to open a Github issue at [https://github.com/kestrelcomputer/kestrel/issues](https://github.com/kestrelcomputer/kestrel/issues).

Before continuing, you will need to insert an SD card into the Kestrel-3's SD card slot.  If you're running the emulator, you can "insert" a card simply by creating a file named `sdcard.sdc` in the same directory that you're running the emulator from.  On Ubuntu Linux, you can do this with the command `touch sdcard.sdc` *before* launching the emulator, like so:

    touch sdcard.sdc
    bin/e romfile roms/forth

## Block Storage

You may be familiar with other programming environments, either as simple as BASIC, or as sophisticated as C, C#, C++, etc.  In any of these languages, and in many in between, you tend to enter your program using some kind of editor, typically as one or more modules, which you then save with a single action.  Today, even many dialects of Forth prefer this style of programming, since it lets the compiler take advantage of the tooling and filing systems available to the host operating system.

Since the Kestrel-3 lacks a sophisticated operating system in its system ROM[^define_rom], at least as of this publication, an alternative method of storing Forth applications is required.  We do this using *block storage*.  Blocks are also known as *screens*, particularly when used to store Forth programs.

[^define_rom]: Read-Only Memory.  This is the kind of memory which the Kestrel-3 can rely on retaining its contents even when power is disconnected from the computer.

A single block stores exactly 1024 bytes, or 1 *kilobyte* of memory (in our case, to the mounted SD card).  They are identified by number, from 1 to however many may fit on your particular SD card.  The programmer typically maintains an awareness of which ranges of blocks are used by which programs or which data files.

I'm going to assume you have a fresh SD card inserted into the slot, or an empty `sdcard.sdc` file created for the Kestrel-3 emulator to work with.  To prepare to work on our font editor, let's type the following commands:

    100 CLEAN
    100 LIST

You should see Forth respond with `ok` after each command.  The first command tells Forth to reset block 100 to all spaces, preparing it to contain Forth source code.  The second lists the block just to make sure it is clean.  Your screen should look like this:

![Preparing for your first screen of source code.](images/ch2.100clean.png)

Let's provide this block with a title (useful for when we run `INDEX`, which I'll explain later), and a command to load another block.  Do not worry about how I chose this other block; honestly, it's arbitrary.  It also, at this point at least, doesn't matter that we haven't provided any source code for this other block yet, because we'll `CLEAN` it.

    0 SET ( Chapter 2 Font Editor   saf2 2016apr17 )
    2 SET MARKER empty
    3 SET 110 LOAD

Now, if you type `100 LIST` again, you should see the program you entered so far.

## Zooming In on a Glyph

A font is, in its most basic form, a collection of glyphs.  Each glyph describes how the character it represents appears to the human viewer.  The Kestrel's system font consists of 256 fixed, eight-pixel by eight-pixel matrices of on or off bits.

You might have heard the term "byte" before, but never really understood what it meant on a concrete level.  Here's where you'll find out: a byte is simply eight bits.  A bit, then, is an on/off value, stored in numerical form as either a 1 or a 0.  This is important to us, because of how we exploit this fact to render text on the screen.  Below, you'll find a blown-up example of what I mean.  It's the letter A itself, represented both in terms of on/off pixels, and the corresponding hexadecimal and decimal representation.

![The glyph for the letter A.](images/ch2.glyphA.png)

What we want our program to do, ultimately, is render a character in "fat bits", approximating the graph on the left-hand side above, where it's easier for us to see and later manipulate.  We don't really care about the corresponding hexadecimal or even their decimal values.  That's something for the computer to worry about.

So, let's create a program to do just that.  

The most fundamental thing we have is "the bit."  It's either on (1) or off (0).  On bits render as white on the screen, so we're going to recreate that in the fat-bits magnification as well.  So, let's write a simple program to render a bit that is "on".  I'm going to zoom in by a factor of 16; meaning, each pixel in a character will now occupy a 16 pixel rectangle on the screen.  Since rectangles aren't fundamental objects, let's break this down even further.

Our rectangle will consist of 16 rows of 16 pixels.  As it happens, Kestrel Forth has a word `H!` which lets us set 16-bits of memory at a time (*half-word store*, in case you were wondering what `H!` stood for).  So, storing a number with all bits turned on into the video frame buffer should yield a white line:

    -1 $FF0000 H!

and, it does.  What we want next is a program to repeat the above simple program enough times to draw a solid rectangle, without having to manually specify addresses everywhere.  So, let's just specify the address once:

    $FF0000

and see if we can use the computer itself to remember the address for us.  While we're at it, let's make sure the computer in fact *does* remember our address after we're done:

    -1 OVER H! .S

This does the same thing, but as you can see in the output, we have a number left on the stack --- that's our memory address.  We just need to increment it to the next row of pixels:

    -1 OVER H!  80 +  .S

Type that line several times in a row to confirm that it actually works as intended.  You should see a white rectangle start to appear in the upper-lefthand corner of the screen.

All that typing is getting laborious though, so let's tell Forth to remember what we mean whenever we say, oh, say, *row*:

    : row ( a - a' )  -1 OVER H!  80 + ;

The `:` symbol tells Forth that we're interested in creating a new word.  The name of the word follows immediately; in this case `row`, for row-of-pixels.

The stuff between the parentheses is a *comment*; its purpose is to inform the programmer of some relevant information about the word; it has zero effect on the meaning of the Forth program itself.  In this case, we're illustrating a "stack effect diagram," or more simply, "stack effect."  This is saying, in a symbolic way, that we're accepting an **a**ddress, and returning another **a**ddress**'** which (for our purposes) is greater than **a**.  We know that `a'` must be larger than `a` because of the *context* in which `row` executes (that of drawing something to the screen).  We'll talk more about stack effects in a later chapter; for now, though, just keep typing and learning.

The `OVER` word tells Forth to take the value "over" the immediate top of stack.  So if we have a stack with the values `1 2 3` and we invoke `OVER` on this, we would expect to see `1 2 3 2` as the result.  So in our case above, `OVER` is what allows us to reuse the address provided as input.

`;` tells the compiler when to *stop* the compilation, and identifies the official end of the definition.

So now, you should be able to just type `row` several times and get the same effect.

When we've convinced ourselves that we can draw individual rows of pixels on the screen, let's bundle this into yet another word that draws a properly proportioned rectangle on the screen, and test it.  Note how we use `PAGE` to clear the screen; it also means we have to use `AT-XY` to place the cursor underneath where we expect the rectangle to go, so the `ok` prompt doesn't overwrite what we just drew.

    : on ( - ) $FF0000 15 FOR row NEXT DROP ; 
    PAGE 0 4 AT-XY
    on

You should see a big white rectangle in the upper-lefthand corner of the screen.  This proves to us that the code works as intended; of course, it's fixed to that corner, so we'll need to change this code later.  But, for now, let's commit this to block 110 before we lose it.

    110 CLEAN
    0 SET ( Fatbits rendering   saf2 2016apr17 )
    2 SET : row ( a - a' ) -1 OVER H! 80 + ;
    3 SET : on ( - ) $FF0000 15 FOR row NEXT DROP ;

To test what we have so far, we make sure that all buffers are actually saved to storage:

    FLUSH

and then we restart the Forth environment:

    BYE

At this point, we should have a clean slate.  Let's make sure our code works:

    100 LOAD
    on
    .S

We should, again, see a white rectangle in the upper-lefthand corner of the screen.  We should also see that our data stack is empty.

![Drawing a white fat-bit.](images/ch2.fatbit.white.png)

