# Getting to Know Forth

# TODO: Screenshots for cursor code.

In this chapter, you'll have an understanding of how one typically writes software in Forth for the Kestrel-3, *on* the Kestrel-3.  This chapter is not directly intended to teach you how to program the Kestrel-3; rather, it's an illustration of what it's like to write software in a native Forth environment.  Unlike most programming tutorials, I'm just going to dive right into a non-trivial application: a simple font editor that you can use to make custom character sets.

It's not terribly complicated as applications go; but, it will touch many of the facilities of the Kestrel-3 on different levels of abstraction, and will introduce everything from the most basic to some moderately advanced Forth programming idioms that you might not have thought possible from reading other chapters of this book.

The program is presented in a manner that allows you to type in the program step by step and to follow along.  It's perfectly OK if you don't get things the first time around; feel free to revisit this chapter again later.  As always, if you have questions which I'm not answering, or if you feel this chapter can be improved, feel free to open a Github issue at [https://github.com/kestrelcomputer/kestrel/issues](https://github.com/kestrelcomputer/kestrel/issues).

If you intend on following along, it's best if you insert a *blank* SD card into the Kestrel-3's SD card slot.  This way, you won't have to worry about following the instructions and risking ruining data on an existing SD card.  If you're running the emulator, you can "insert" a card simply by creating a file named `sdcard.sdc` in the same directory that you're running the emulator from.  On Ubuntu Linux, you can do this with the command `touch sdcard.sdc` *before* launching the emulator, like so:

    rm -f sdcard.sdc
    dd if=/dev/zero of=sdcard.sdc bs=1024 count=200
    bin/e romfile roms/forth

## Block Storage

You may be familiar with other programming environments in older home computers, such as their many dialects of BASIC, or you may be familiar with programming on today's enterprise-quality environments, using languages as sophisticated as C#, C++, Java, etc.  The one thing these languages all have in common is the concept of the *file*, an arbitrarily long sequence of secondary storage in which you store a program.

In these environments, you tend to enter your program using some kind of *editor*, typically as one or more modules, which you then save with a single action (e.g., selecting **Save** from a menu, or typing `SAVE "MY PROGRAM",8` or the like).  Today, even many dialects of Forth prefer this style of programming, since it lets the compiler take advantage of the tooling and filing systems available to the host operating system.

Since the Kestrel-3 currently lacks a file-capable operating system in its system ROM[^define_rom], an alternative method of storing Forth applications is required.  We do this using something called *block storage*.  Blocks are also known as *screens*, particularly when used to store Forth programs.

[^define_rom]: Read-Only Memory.  This is the kind of memory which the Kestrel-3 can rely on retaining its contents even when power is disconnected from the computer.

A single block stores exactly 1024 bytes, or 1 *kilobyte* of memory (in our case, to the mounted SD card).  They are identified by number, from 1 to however many may fit on your particular SD card.  The programmer typically maintains an awareness of which ranges of blocks are used by which programs or which data files.

I'm going to assume you have a fresh SD card inserted into the slot, or an empty `sdcard.sdc` file created for the Kestrel-3 emulator to work with.  To prepare to work on our font editor, let's type the following commands:

    100 CLEAN
    100 LIST

You should see Forth respond with `ok` after it finishes each command.  The first command tells Forth to reset block 100 to all spaces, preparing it to contain Forth source code.  The second lists the block just to make sure it has been cleared.  Your screen should look like this:

![Preparing for your first screen of source code.](images/ch2.100clean.png)

Let's provide this block with a title (useful for when we run `INDEX`, which I'll explain later), and a command to load another block.  Do not worry about how I chose this other block; honestly, it's arbitrary.  It also, at this point at least, doesn't matter that we haven't provided any source code for this other block yet, because we'll just `CLEAN` it later first.

    0 SET ( Chapter 2 Font Editor   saf2 2016apr17 )
    2 SET MARKER empty
    3 SET 110 LOAD

Now, if you type `100 LIST` again, you should see the program you entered so far.

## Zooming In on a Glyph

A font is, in its most basic form, a collection of glyphs.  Each glyph describes how the character it represents appears to the human viewer.  The Kestrel's system font consists of 256 fixed, eight-pixel by eight-pixel matrices of on or off bits.

You might have heard the term "byte" before, but never really understood what it meant on a concrete level.  A byte is simply eight bits.  A bit, then, is an on/off value, stored in numerical form as either a 1 or a 0.  This combination of bits allows us to represent numbers, text, etc.  The interpretation, really, is entirely context dependent.  This is important to us because of how we exploit this fact to render text on the screen.  Below, you'll find a blown-up example of what I mean.  It's the letter A itself, represented both in terms of on/off pixels, and the corresponding hexadecimal and decimal representation.

{id="ch2_img_glyphA"}
![The glyph for the letter A.](images/ch2.glyphA.png)

What we want our font editor to do, ultimately, is render a character in "fat bits", approximating the graph on the left-hand side above, where it's easier for us to see and later manipulate.  We don't really care about the corresponding hexadecimal or even their decimal values.  That's something for the computer to worry about.

### White Fat Bits

The most fundamental thing we have is "the bit."  It's either on (1) or off (0).  On bits render as white on the screen, so we're going to recreate that in the fat-bits magnification as well.  So, let's write a simple program to render a bit that is "on".  I'm going to zoom in by a factor of 16; meaning, each pixel in a character will now occupy a 16 pixel rectangle on the screen.  Since rectangles aren't fundamental objects, let's break this down even further.

Our rectangle will consist of 16 rows of 16 pixels.  As it happens, Kestrel Forth has a word `H!` which lets us set 16-bits of memory at a time (*half-word store*, in case you were wondering what `H!` stood for).  So, storing a number with all bits turned on into the video frame buffer should yield a small white line in the upper left-hand corner of the screen:

    -1 $FF0000 H!

After demonstrating our intuition is correct, we next want a program to repeat the above simple program enough times to draw a solid rectangle, without having to manually specify addresses everywhere.  Let's just specify the address once:

    PAGE
    $FF0000

and see if we can use the computer itself to remember the address for us.  While we're at it, let's make sure the computer in fact *does* remember our address after we're done:

    -1 OVER H! .S

Why `OVER`?  Because our address was placed on the Forth data stack first, and the value we want to write is underneath it, we need to "reach over" to access the address again for `H!` to work correctly.  The `.S` word will print to the display the current data stack contents without actually clearing the stack.

After proving this does the same thing, we have a number left on the stack -- that's our memory address.  We just need to increment it to the next row of pixels on the screen.  With the Kestrel-3 running at 640x480 resolution with only 2 colors on the screen (black and white), it turns out that each row of pixels corresponds to 80 bytes.  You can figure this out yourself by simply dividing the resolution by the number of pixels stored in a byte (640 / 8 = 80).  So, we use that figure to increment our address:

    -1 OVER H!  80 +  .S

The first time you type the above line, nothing will apparently have happened.  However, as you type that line several times in a row, you should be able to confirm that it actually works as intended.  A slim white rectangle will start to grow downwards in the upper-lefthand corner of the screen.

All that typing is getting laborious though, so let's tell Forth to remember what we mean whenever we say, just to pick an arbitrary name, *row*:

    : row ( a - a' )  -1 OVER H!  80 + ;

There are some more language elements to discuss here.  The `:` symbol tells Forth that we're interested in creating a new word.  The name of the word follows immediately; in this case `row`, as in row of pixels.

The stuff between the parentheses is a *comment*; its purpose is to inform the programmer of some relevant information about the word; it has zero effect on the meaning of the Forth program itself.  In this case, we're illustrating a "stack effect diagram," or more simply, "stack effect."  This is saying to whoever is reading the code, in a symbolic way, that we're accepting an address `a`, and returning another address `a'`.  It's not stated in this comment that `a' > a` or that `a' = a+80`, because it's obvious from context what the result should be.  Reading the line of source, we see that `a'` should be larger than `a` by 80.  Another reason who know this must be true is because of the context in which we intend on using `row`, namely to affect graphics on the screen.  We'll talk more about stack effects elsewhere.

`;` tells the compiler when to *stop* the compilation, and identifies the official end of the definition.

So now, you should be able to just type `row` several times and get the same effect.

When we've convinced ourselves that we can draw individual rows of pixels on the screen, let's bundle this into yet another word that draws a properly proportioned rectangle on the screen, and test it.  Note how we use `PAGE` to clear the screen; it also means we have to use `AT-XY` to place the cursor underneath where we expect the rectangle to go, so the `ok` prompt doesn't overwrite what we just drew.

    : on ( - ) $FF0000 15 FOR row NEXT DROP ; 
    PAGE 0 4 AT-XY
    on

Why only `15` in the program?  That's because `FOR` counts down towards, and including, zero.  If you were to count the numbers out sequentially, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, and 0, you'll notice that there are in fact *16* numbers present.

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

### Black Fat Bits

We turn our attention now to representing those bits which are turned *off*.  Referring back to [the figure of the glyph for the letter A](#ch2_img_glyphA), note how despite the dark squares imply dark dots on the screen, we can still see the grid lines in the figure.  We similarly want some kind of visual cue on the screen when representing fat bits for dark pixels.

What we need to do is render the edges of a simple rectangle, like so:

![What we want to use to represent a black fat-bit.](images/ch2.fatbit.black.desired.png)

We already know how to get the horizontal line:

    PAGE 0 4 AT-XY
    -1 $FF0000 H!

But, how do we get the vertical strip?  We only need the top-most bit of the 16-bit row to be set, so let's give it a shot:

    $8000 $FF0050 H!

Whoops, that's not right.  If we were to continue with this, we'd end up with something that looks like a giant T.  The reason this happens is because the microprocessor in the Kestrel-3 is a *little-endian* processor, while the video circuitry expects data to be stored in *big-endian* format[^mgia_history].  For this reason, we need to swap the upper and lower halves of the word we want to store.  You can confirm this following code words:

    $0080 $FF0050 H!
    $0080 $FF00A0 H!

You should start to see the top-most portion of the rectangle's edges start to appear in the upper left-hand corner of the screen.

[^mgia_history]: The Monochrome Graphics Interface Adapter, or MGIA, core is responsible for taking data from memory and sending it to your monitor.  I originally developed it for the Kestrel-2, which originally was going to use a *word-addressed* processor.  Later in its evolution, it became more advantageous to switch the processor to use bytes as its smallest addressible unit; however, the MGIA remained word-addressed.  The Kestrel-3 inherits the Kestrel-2's MGIA (in the upgraded form of the MGIA-II) core, complete with its word-based addressing.  Fear not, however; the successor to the MGIA-II core will let you adjust its endianness, making writing software for it significantly less burdensome.
 
So, our black fat-bit consists of a row of pixels, followed by a 15-tall column of pixels.  Let's incorporate that definition into our program so far:

    110 LIST
    3 OPEN
    3 SET : col ( a - a' ) $0080 OVER H! 80 + ;
    5 SET : off ( - ) $FF0000 row 14 FOR col NEXT DROP ;
    FLUSH
    100 LOAD
    off .S

The `OPEN` command is used to insert a blank line at the indicated line.  This lets us use `3 SET` subsequently to insert code before the definition of `on` (which is now on line 4).  Be careful when using `OPEN` though; anything stored on line 15 will be discarded.  Thankfully, we're no where close to having to deal with that problem.

Note also that we define `col` above the definition of `on` or `off`.  In this simple example, it's not strictly necessary to do this, so long as you make sure `col` appears before `off`.  However, I find it useful to organize words by their logical relationship to each other.  In this case, `row` and `col` are related in that they're primitives used by `on` and `off`.

Anyway, you should see something like the following:

![Drawing a black fat-bit.](images/ch2.fatbit.black.png)

### Rows of Fat Bits

At this point, we have a means of drawing a single white or black fat bit in the upper left-hand corner of the display.  But, what we'd really like is to draw a complete matrix of them.  A matrix is an array of arrays, of course, so it seems logical that our next step is to simply get an array of fat bits up on the screen first.

I need a plan to go about doing this.  I won't go into the different approaches I've considered here; this chapter is already quite long as it is.  Instead, I'll just invoke the power of successive refinement and iterative programming.

For the moment, let's pretend we have a version of `on` and `off` which renders a white or black fat-bit where we tell it.  That is, instead of hardcoding `$FF0000` as the frame buffer memory address, we accept this value from the stack.  It would be very convenient if we can invoke a sequence like, `on off on off`, to just render fat-bits horizontally.  Since `on` and `off` would accept an address from the stack, it follows that they *must* also leave an address on the stack as well.

In other words, we need `on` and `off` to have the following stack effects:

    : on ( a - a' ) ...
    : off ( a - a' ) ...

But what is `a'` in this case?  Since a byte holds 8 pixels, and our fat-bits are 16-pixels wide, it follows then that `a' = a + 2`.  Provided we don't do this more than 40 times in a row, we should be able to draw a horizontal strip of fat-bits.

The simplest way to achieve this is to change our existing definitions like so:

    110 LIST
    4 SET : on ( a - a') DUP 15 FOR row NEXT DROP 2 + ;
    5 SET : off ( a - a') DUP row 14 FOR col NEXT DROP 2 + ;
    110 LIST  ( to make sure our changes took )
    FLUSH

Now, we need to test them to make sure our assumptions hold.

    BYE
    100 LOAD
    $FF0000 on off on off HEX U. DECIMAL

You should see something like the following:

![An array of fat-bits.](images/ch2.fatbit.array.png)

Notice that the address left on the stack is 8 greater than what we started with, which is what we'd expect after incrementing it by two four times.  Further note the on/off pattern on the top of the display.

But, calling `on` and `off` manually isn't the goal; we want to invoke them in a pattern based on the numbers corresponding to a row in a glyph's font.  The top of the letter A, [if you'll recall](#ch2_img_glyphA), is represented by the number `24`.  How can we convert a number into a set of fat-bits?  Let's make a word which handles this one bit at a time.  Literally.

    : pixel ( n a - n' a' ) OVER $80 AND IF on ELSE off THEN SWAP 2* SWAP ;
    PAGE 0 4 AT-XY
    24 $FF0000
    pixel
    pixel
    pixel
    pixel
    pixel
    pixel
    pixel
    pixel

![The top row of the letter A.](images/ch2.fatbit.A.row.png)

Success!  Now we commit this to our program in block memory.

    110 LIST
    7 SET : pixel ( n a - n' a' ) OVER $80 AND IF on ELSE OFF THEN
    8 SET     SWAP 2* SWAP ;
    9 SET : row ( n a - ) 7 FOR pixel NEXT 2DROP ;
    FLUSH

Whoa, hold up!  I'm redefining `row` on line 9.  How is this supposed to work?  Forth, unlike other programming languages, has what's called a *hyper-static global environment.*  These four fancy words basically means, very simply, if you *redefine* a word, then *prior* uses of the word *do not* change meaning.  Only future uses of the word take on the new semantics.

**This is why you hear the word *context* so frequently in Forth programming.**  It's clear that, by the time line 9 rolls around, the *context* for what a row of pixels *means* has changed.  We simply make this known to Forth in the most natural way possible; we simply *redefine* it as we need it, *when* we need it.

Contrast this with virtually any other language you're likely to come into contact with.  Particularly illustrated by various flavors of Lisp, if you were to change the meaning of `row` like this (in some made-up dialect of Lisp):

    (define (row ...) ...definition 1...)
    (define (on addr) (times 16 (let (addr (row addr)))))
    ... some more code here ...
    (define (row ...) ...definition 2...)

then the meaning of `(on x)` would fundamentally change.  Were you to do this in a more statically controlled language like C, you'll just get a compile-time error for duplicately defined symbols.

Anyway, we can prove that this works easily enough:

    BYE
    100 LOAD
    24 $FF0000 row .S

Note that we use `2DROP` to clean the stack up after we're done rendering the row.  Cleaning up after ourselves is important because it lets us think about words that perform an action (a "procedure") strictly as *verbs*; once they're proven to work, we can treat them axiomatically when debugging more complex programs later.

Now that we can render a row, we need to use `row` to build out our matrix on the screen.  We want each row to appear 16 pixels below its predecessor.  Since each row of pixels requires 80 bytes of frame buffer memory, that corresponds to 1280 bytes.  So, let's make a new word `matrix` to render the complete matrix.

    : matrix ( b a - ) 7 FOR OVER C@ OVER row 1280 + SWAP 1+ SWAP NEXT 2DROP ;

**Note.** I'm no longer taking byte values directly from the stack; now, `b` refers to a byte in memory (`C@` means *character fetch*).  Yes, Kestrel Forth is a 64-bit Forth, which means we can pass 8 bytes directly in if we wanted to.  However, it's more useful to draw data from memory, which we'll see in a later section.

We can test this simply enough by copying the bytes from [the glyph for A](#ch2_img_glyphA) into a 64-bit wide variable, and sending it through `matrix` like so:

    VARIABLE bm ( bit map )
    $183C3C667EC3C300 bm !
    PAGE 0 17 AT-XY
    bm $FF0000 matrix

![MGIA endian differences strikes again!](images/ch2.upsidedown.A.png)

Whoops; I forgot to swap the bytes due to the MGIA's big-endian accesses to memory.  Even so, I think we've adequately proven that `matrix` works as intended.  Let's commit it to block storage now.

    110 LIST
    10 SET : matrix ( b a - ) 7 FOR OVER C@ OVER row
    11 SET     1280 + SWAP 1+ SWAP NEXT 2DROP ;
    FLUSH
    110 LIST

This block is starting to look pretty full, so I think we'll end it there.  Really, the whole purpose of this block is to render the matrix, so let's update the index title accordingly.

    0 SET ( Fatbits: matrix   saf2 2016apr17 )
    FLUSH
    110 LIST

One final test to make sure it works from a clean slate:

    BYE
    100 LOAD
    VARIABLE bm
    $00C3C37E663C3C18 bm !
    bm $FF0000 matrix

We should see a giant version of the letter A.

Since we've accomplished one of our goals, now seems like a good time to update documentation for our software.  I'm going to place our documentation in block 111.

    111 CLEAN
    0 SET \ shadow docs: matrix    saf2 2016apr17
    2 SET row (a - a') draws 16 white pixels at the screen addr
    3 SET     passed in.  Returns the next addr in the column.
    4 SET col (a - a') draws a single pixel, but otherwise
    5 SET     similar to row.
    6 SET on, off (a - a') draws a white or black fat-bit resp.
    7 SET pixel (n a - n' a') draws a single pixel, taken from
    8 SET     bit 7 of n.  a' points to next fat-bit, n' is 2*n.
    9 SET row (n a -) draws a single row of fat-bits, based
    10 SET     on the value of n.
    11 SET matrix (b a -) draws an 8x8 grid of fat-bits. Data
    12 SET     comes from a buffer pointed at by b.
    FLUSH

These kinds of screens are historically known as *shadow blocks*.  They contain synopses which explain the roles of the words defined in the corresponding non-shadow block.  By convention, even-numbered blocks contains code, and odd-numbered blocks contains commentary.  It's a good habit to write shadow comments once a block of code has been composed.  Forth is extremely terse, and it's much easier to forget the purpose of a word sooner than in other, more conventional programming languages.

## Your Move: Prompting the User with a Cursor

We've gotten a reasonable facsimile of the graph to show on the screen.  But, now, let's add a *cursor* so that we can indicate which cell on the fatbit grid to change.

OK, more specifications.  There are several approaches we can take to accomplish this goal, but once again, I'm going to compact my thought process down for the sake of shortening an already lengthy chapter.

I think it'd be nice to have a small, grey dot in the middle of a cell to indicate the current cell.  Of course, as I write this, the Kestrel-3 does not have greyscale capability, so we'll emulate it with a simple dither pattern, like so:

![The desired cursor shape.](images/ch2...)
................  0000  0000
................  0000  0000
................  0000  0000
................  0000  0000
....@.@.@.@.....  0AA0  A00A
.....@.@.@.@....  0550  5005
....@.@.@.@.....  0AA0  A00A
.....@.@.@.@....  0550  5005
....@.@.@.@.....  0AA0  A00A
.....@.@.@.@....  0550  5005
....@.@.@.@.....  0AA0  A00A
.....@.@.@.@....  0550  5005
................  0000  0000
................  0000  0000
................  0000  0000
................  0000  0000


I like this design, because it leaves a large portion of the cell visible to the user, while still providing a visual cue to the user which cell is currently referenced by the program.  Other options include cross-hairs, an arrow shape, etc.  As we'll soon see, this is actually one of the simpler patterns to render.

I've also converted the bitmap pattern into hexadecimal values both for human consumption (middle column), and in the proper format expected by the MGIA hardware (right-most column).  We'll be using the latter column later.

### Rendering the Cursor

For now, let's keep things simple, and just render the cursor on the screen in the upper left-hand corner.  We can expand from there later.

First, we start out with defining our bitmap data.

    CREATE cursorImg
    0 , 0 , 0 , 0 , $A00A , $5005 , $A00A , $5005 ,
    $A00A , $5005 , $A00A , $5005 , 0 , 0 , 0 , 0 ,

Next, I'll make a word that "paints" the cursor, line by line, just like we did with the fatbit.  We know in this case that it must accept two addressses (one for our cursor image, and one for the framebuffer), and it should leave two addresses on the stack (same thing, suitably adjusted) so we can just run the same word again and again until the cursor has been rendered.

Unlike painting the fatbits, though, we want each on-bit in our cursor image to *toggle* the on-screen bit on or off.  Off bits should cause no action to take place.  It turns out there's a logical operator called *exclusive-OR* (`XOR`) which does this for us.  As you might imagine, `H@` is the compliment to `H!`, where it reads a 16-bit quantity from memory (*half-word fetch*).

    : row ( c a - c' a' ) OVER @ OVER H@ XOR OVER H! 80 + SWAP CELL+ SWAP ;

We can test it out like so:

    cursorImg $FF0000
    row row row row
    row row row row
    row row row row
    row row row row

At this point, we should have a small grey dot in the upper lefthand corner of the screen.

![Proof of concept for the cursor.](images/ch2...)

We see that it works, so let's commit this to block storage for future use.

    108 CLEAN
    0 SET ( Cursor Display    saf2 2016apr19)
    2 SET CREATE cursorImg HEX
    3 SET   0000 , 0000 , 0000 , 0000 , A00A , 5005 , A00A , 5005 ,
    4 SET   A00A , 5005 , A00A , 5005 , 0000 , 0000 , 0000 , 0000 ,
    5 SET DECIMAL
    7 SET : row ( c a - c' a' ) OVER @ OVER H@ XOR OVER H!
    8 SET     80 + SWAP CELL+ SWAP ;
    FLUSH

We can make sure everything still works by testing the program so far:

    BYE
    108 LOAD
    cursorImg $FF0000 row row row row row row row row row row row row

That should work well enough to render the cursor onto the sreen.  If so, let's link it into our font editor program.

    100 LIST
    3 SET 110 LOAD ( matrix visuals )
    4 SET 108 LOAD ( cursor visuals )

### Moving the Cursor

The cursor doesn't serve much purpose if it's just stuck in one spot on the screen; there needs to be a way to actually move it around.  So, our next step is to adjust the code we've written so far so that we can place it anywhere on our matrix.

We know the fatbit matrix is 8 wide and 8 tall, so it seems reasonable that we should make some kind of mathematical formula that translates a pair of small integers into a screen address.  `cursor` already accepts a screen address, so there is no need to change `cursor` itself.

We know that each cell is two bytes wide, so it seems reasonable that we multiply `x` by two and add to some base address.  Let's write that first, since that should be pretty simple to get working.

    : addr ( x - a ) 2* $FF0000 + ;

Well, that was pretty easy.  Let's test it.

    HEX
    0 addr U.
    1 addr U.
    2 addr U.
    7 addr U.
    DECIMAL

The first three prints establish the pattern we want to see; with each increasing coordinate, the resulting address goes up by two.  The final test is just for good measure.

![Testing our address computation so far.](images/ch2...)

**Preconditions again.**  We will *never* invoke this function with a `x` coordinate outside the range of 0 to 7.  So, we do not include any explicit error-checking logic here.  In most other programming languages, the very *thought* of coding like this would be anathema.  If you feel safer adding this bounds-checking logic, I'll leave it as an exercise for you to consider.  However, just be aware: our *intention* that `addr` be invoked with an `x` coordinate in the range 0..7 will be documented in the shadow block later, but it will be *enforced* in the *code* elsewhere.  I can get away with this because `addr` is very clearly a function that is of domain-specific, very limited use outside of the font editor.

**If**, however, I were writing a word for general purpose consumption, where I simply *didn't trust* the code invoking my software, then things would be very different.  I would never trust my inputs, and would strongly encourage bounds checking in this case.  However, this is a topic for another time, and is well beyond the scope of this chapter.

Let's try rendering the cursor in the 3rd place over to the right, just to get a feel of how everything is supposed to fit together:

    cursorImg 3 addr row row row row row row row row row row row row

![Cursor appears to the right.](images/ch2...)

It's now time to consider the `y` coordinate, or the vertical axis.  Like the horizontal axis, we are going to constrain it to the values 0 to 7 inclusive elsewhere in the code.  But, our address computation is different.  Each fatbit cell is 16 pixels tall, and as we recall from before, each row on the screen consists of 80 bytes.  So, each row of the fatbit matrix consists of 1280 bytes of memory.  So, let's account for that in our definition of `addr` now:

    : addr ( x y - a ) 1280 * SWAP 2* + $FF0000 + ;

As you can see, the effective address computation is a bit more complex, but not terribly so.  We first account for the `y` coordinate, then tackle the `x` coordinate separately and add them together.  Once we have our relative offset, we add the base address of the screen, and that's our final address for the cursor.  Let's test it.

    PAGE 0 17 AT-XY
    cursorImg 3 4 addr row row row row row row row row row row row row

![The cursor can now be placed vertically too.](images/ch2...)

We see that things work, so let's commit this to block storage.

    108 LIST
    10 SET : addr ( x y - a ) 1280 * SWAP 2* + $FF0000 + ;
    FLUSH

You can probably predict how we're going to write a word that draws the entire 16x16 pixel image for the cursor.  Here's how I did it, and tested it.

    : cursor ( x y - ) cursorImg -ROT addr 15 FOR row NEXT 2DROP ;
    PAGE 0 17 AT-XY
    2 4 cursor 7 5 cursor .S

![We can draw the cursor anywhere we want, and stack looks good.](images/ch2...)

Now that we have this code working, we can commit this as well.

    108 LIST
    12 SET : cursor ( x y - ) cursorImg -ROT addr
    13 SET     15 FOR row NEXT 2DROP ;
    FLUSH

Looks like we have completed our cursor drawing program, so now it's time to update our documentation.

    109 CLEAN
    0 SET \ shadow docs: Cursor rendering
    2 SET cursorImg (-a) answers the address of a 16x16 pixel bit-
    3 SET     map containing the cursor image.
    4 SET row (c a-c' a') toggles bits in the on-screen frame-
    5 SET     buffer if and only if corresponding bits in the
    6 SET     cursor bitmap are set.  Adjusts pointers.
    7 SET addr (x y-a) answers the framebuffer address correspond-
    8 SET     ing to the specified coordinate in the fatbit
    9 SET     matrix.
    10 SET cursor (x y-) draws or undraws (if called again) the
    11 SET     cursor image at the designated place on the fat-
    12 SET     bit matrix.
    FLUSH

### Making Sure It All Works Together

A final test is to make sure that our fatbits matrix code and our cursor rendering code actually works together.  There's no obvious reason why they should interfere; however, computer software has a nasty habit of surprising the author in this respect.

We'd like to create a simple glyph buffer, display it, and present the cursor in its home position (0,0).  That should be enough for now.

    VARIABLE glyph
    $1122334455667788 glyph !
    PAGE 0 17 AT-XY  glyph $FF0000 matrix  0 0 cursor

The result should look somewhat like this:

![Confirming that the fatbit viewing code and the cursor display works together.](images/ch2...)

Once we know that it does, let's commit it to block storage.

    106 CLEAN
    0 SET ( full screen update    saf2 2016apr22 )
    2 SET VARIABLE glyph
    3 SET $1122334455667788 glyph !
    4 SET : everything ( - ) PAGE glyph $FF0000 matrix 0 0 cursor ;
    FLUSH

    100 LIST
    5 SET 106 LOAD ( full screen visualization )
    FLUSH

We should now be able to test that it integrates nicely:

    BYE
    100 LOAD everything

After a few seconds, the screen should look like before, with a simple fatbits matrix and a cursor in position (0,0).  The only difference is the `OK` prompt will partially overwrite the results.

When we're happy with that, let's not forget to update our shadow documentation.

    107 CLEAN
    0 SET \ shadow docs: full screen update
    2 SET glyph (-a) is a 64-bit buffer holds the current image of the
    3 SET   glyph being edited.
    4 SET everything (-) redraws the entire screen from scratch.  This
    5 SET   is a very slow operation since it touches every pixel on
    6 SET   the screen, so use with care.
    FLUSH

## Quitting the Application

Let's switch gears for a moment.  So far, we've been writing code to provide visual output to the user.  However, we also need to accept input as well.  We know we want the font editor to run as an application; however, we need a way to exit the editor so that the user can return to a normal Forth `ok` prompt whenever he or she wants.  I'm going to pick an arbitrary convention for this now.  We can implement this logic simply enough by waiting for a key press of the letter `Q`, like this:

    : quit? ( c - f ) 81 = ;
    : main ( - ) BEGIN KEY quit? UNTIL ;

We can test this now like so:

    main

The computer should appear to lock up, doing absolutely nothing what-so-ever until you type a capital `Q` on the keyboard.

Once we've confirmed this works as expected, let's bundle it into the rest of our application as well.

    100 LIST
    6 SET 104 LOAD ( main event loop )
    104 CLEAN
    0 SET ( main event loop   saf2 2016apr22 )
    2 SET : quit? ( c - f ) 81 = ;
    3 SET : main ( - ) BEGIN KEY quit? UNTIL ;
    FLUSH

And once more, a simple integration test.

    BYE
    100 LOAD main .S

You should be able to mash on the keyboard all you want; it should not return to an `ok` prompt until you type a capital `Q`.  Furthermore, the data stack should report as empty.

Normally, we'd end up writing a shadow block now, but this code is so simple, and obviously incomplete, that I'll defer that action until we flesh out the code a bit more.  That way, we minimize the amount of rework that's involved.

## Where Do We Begin?

Now that we have a way of *quitting* the application, how do we *start* it?  Perhaps a more important question, with which glyph do we begin our editing session with?  So far, on block 106, we maintain a single 8-byte buffer (one variable in Kestrel-3 Forth happens to be eight bytes) to hold our glyph's image in.  We even pre-initialize it with some bogus data just to confirm our visualization code continues to work.  However, we've reached a point now where this is no longer feasible.

At this point, I need to talk more about how Kestrel-3 Forth interprets font data in memory.  A font contains 256 glyphs, whether they're used or not.  Each glyph consists of eight bytes of data.  So, a complete font contains 2048 bytes of memory.  We need to allocate at least this much space as a temporary working font for the font editor to work from.  This buffer will replace the `glyph` variable.

Next, we need to adjust how each row of pixels is accessed.  This will involve changing code in block 110.  This is because the Kestrel-3 treats fonts as a **2048x8** pixel bitmap, instead of as a sequential array of 8x8 bitmaps.  The changes are relatively simple, but we must remember to make them.  So, instead of each row of pixels being at sequential addresses, they now appear at 256-byte offsets from each other.  See the figure below for a visual explanation as to why.

![The font bitmap is now wider than a single glyph, so now each row is spaced at 256 bytes.](images/ch2...)

Finally, we need to provide a communications path from the user to the font editor that lets the user select which glyph to edit when starting the font editor.  We'll re-purpose the variable `glyph` for this purpose.

Unfortunately, these changes are not something which can be easily tested interactively.  So, in the event of a defect being introduced, we'll need to rely on some debugging skills or simple logic to figure things out.  However, it's always best to get into the habit of proving the correctness of your code ahead of time as much as you can.  It may take longer to write your programs this way, but it will save immense amounts of time when debugging later.

Let's begin with creating the 2KiB buffer we'll use for temporary storage.

    106 LIST
    2 OPEN
    2 SET CREATE fontbuf  2048 ALLOT

Next, we're going to alter our `everything` definition to compute the proper glyph address inside this bitmap.

    3 SET : addr ( c - a ) fontbuf + ;
    4 SET
    6 SET
    7 SET : everything ( - ) PAGE glyph @ addr $FF0000 matrix
    8 SET     0 0 cursor ;

That should be all we need to get the fatbit code to start drawing the matrix.  Getting to *keep* drawing the right bits in the matrix is the next thing we need to change.

    110 LIST
    11 SET     1280 + SWAP 256 + SWAP NEXT 2DROP ;

Finally we need to provide a way of "running" the font editor with the provided glyph number.  Let's have it print an error if the glyph number is not valid; otherwise, just set the glyph and invoke `everything` then `main`.

    104 LIST
    5 SET : design ( n - ) glyph ! everything main ;

We should now be in a position to test everything.

    FLUSH
    BYE
    100 LOAD
    2 design

We should finally have a program that draws a fatbit matrix and a cursor, but which does nothing until you press capital `Q`.

## Going Mobile -- Supporting Cursor Movement

Right now, our font editor's utility remains quite limited.  No means exists for moving the cursor about the glyph.  I will add the ability to move the cursor in this section.

Instead of relying on SDL- or PS/2-specific keycodes for the arrow keys found on most keyboards, I'm going to stick with using plain ASCII.  For now, let's use `W`, `S`, `A`, and `D` keys for up, down, left, and right movement.  Implementing cursor movement will involve changing three blocks of code.  First, we need to add the lower-level words that actually perform the visual bookkeeping.  Next, we need to enhance the `main` event handler to support more than just quitting the application.  Finally, we need to provide appropriate bindings between the event handler and our lower-level words.  We'll treat the event dispatcher changes in the next section.

We start by implementing the low-level code.  We must recognize that the first-generation Kestrel-3 computer is not a fast computer; we cannot just refresh the entire display and expect good performance.  Instead, it would be better if we rendered the full display, and then *incrementally* updated the display with each input event.  To see how this works, we first render the entire display:

    cursor- everything 0 17 AT-XY cursor+

The words `cursor-` and `cursor+` hide and reveal the text cursor, respectively.  Without these words, cursor-shaped video artifacts will remain, as the Forth interpreter and our application will compete for access to the same pixels.  Your screen should look like this:

![After drawing the whole screen.](images/ch2...)

Notice that the cursor appears at location (0,0) in the matrix.  If we want to move it to location (1,0), for instance, we first must erase the cursor at (0,0), then redraw it at (1,0).

    0 0 cursor  1 0 cursor

The screen should now look like:

![After moving the cursor.](images/ch2...)

Thus, every cursor movement key must first erase the cursor from where it's currently located, then adjust the coordinates, then redraw.

    VARIABLE cx  VARIABLE cy
    0 cx !  0 cy !
    : toggle ( - ) cx @ cy @ cursor ;
    : goL ( - ) toggle  cx @ IF -1 cx +! THEN  toggle ;
    : goR ( - ) toggle  cx @ 7 < IF 1 cx +! THEN toggle ;

To test what we have so far, let's refresh the display, and try moving the cursor to the right (`goR`) or to the left (`goL`).

    cursor- everything 0 17 AT-XY cursor+
    goR goR goR ( move to the right 3 places )
    goL goL goL ( move back )
    goL goL ( try to go beyond left-hand edge )
    toggle 7 cx ! toggle
    goR goR ( try to go beyond right-hand edge )
    goL

After confirming the above logic works and you still have a functional Forth environment, we now gain the confidence to do the same with the Y coordinate as well.

    : goU ( - ) toggle  cy @ IF -1 cy +! THEN  toggle ;
    : goD ( - ) toggle  cy @ 7 < IF 1 cy +! THEN  toggle ;
    0 cx ! 0 cy !
    cursor- everything 0 17 AT-XY cursor+
    goD goD goD ( move down 3 places )
    goU goU goU ( move back up )
    goU goU ( try to go beyond top edge )
    toggle 7 cy ! toggle
    goD goD ( try to go beyond bottom edge )
    goU


Since the logic is so similar to the horizontal movement words, it should be a fairly rote procedure to confirm that the vertical movement words work as intended.  Once this is done, we commit them to block storage.

    106 LIST
    5 SET VARIABLE cx  VARIABLE cy  VARIABLE glyph
    8 SET     home toggle ;
    7 OPEN
    7 SET : home ( - ) 0 cx ! 0 cy ! ;
    7 OPEN
    7 SET : toggle ( - ) cx @ cy @ cursor ;

    112 CLEAN
    0 SET ( cursor movement   saf2 2016apr22 )
    2 SET : goL ( - ) toggle  cx @ IF -1 cx +! THEN  toggle ;
    3 SET : goR ( - ) toggle  cx @ 7 XOR IF 1 cx +! THEN  toggle ;
    4 SET : goU ( - ) toggle  cy @ IF -1 cy +! THEN  toggle ;
    5 SET : goD ( - ) toggle  cy @ 7 XOR IF 1 cy +! THEN  toggle ;

    100 LIST
    7 SET 112 LOAD ( cursor movement )

    FLUSH

Now we test our integration.

    BYE
    100 LOAD
    cursor- everything 0 17 AT-XY cursor+ home
    goR goR
    goD goD
    goL goL
    goU goU

Everything should look pretty good at this point, which now frees us up to going back to consider how we handle event dispatch.

### Revisiting the Main Loop

In this section, we need to implement an event dispatching mechanism that is somewhat more sophisticated than simply checking to see if we're quitting or not.  My strategy for this is to convert key presses into Forth dictionary lookups.  This maps each key (say `Q`) to a specially formatted Forth word (e.g., `$$Q`).  If the word is discovered, it's executed; otherwise, we just ignore the key press.

First, let's test our intuition about finding a word in the dictionary.  We need a workspace to construct words with:

    VARIABLE ws
    $20242403 ws !

This lays down four bytes into the variable, which form a valid Forth string.  In fact, we should be able to `TYPE` it and get back the following result:

    ws COUNT TYPE CR

![Two dollar signs are expected.](images/ch2...)

Let's define a simple word according to this template:

    : $$A ." Hello world" CR ;

We should be able to locate this word by name:

    : test1 $" $$A" NAME? .S ;
    test1

We should have two values on the stack; their meaning isn't important now, except for the fact that they should both be non-zero.  This is how we know the word was found.  If we try to find a word which we know hasn't yet been defined, we expect the top item on the stack to be zero.

    : test2 $" $$B" NAME? .S ;
    2DROP test2

Locating the word based on a key press should now be fairly obvious.  We simply store the keycode into the appropriate place in the `ws` workspace, and then pass it to `NAME?` to find the word.

    : handler ( c -- xt na | ws 0 ) ws 3 + C! ws NAME? ;
    KEY handler .S

If you type a capital-`A`, you should see that the word is found (non-zero top of stack).  If you press `B`, or some other character, you should find the top of stack zero.

Now that we see how this all works, let's alter our event handling code to make use of the new dispatching logic.  We're just going to rewrite the whole block from scratch, since that takes less effort than incrementally changing things.

    104 CLEAN
    0 SET ( main event loop   saf2 2016apr22 )
    2 SET VARIABLE done?   VARIABLE ws
    3 SET : handler ( c - xt na | ws 0 ) ws 3 + C!  ws NAME? ;
    4 SET : main ( - ) BEGIN KEY handler IF EXECUTE ELSE DROP THEN
    5 SET     done? @ UNTIL ;
    7 SET : design ( n - ) 255 AND glyph !  everything
    8 SET     0 done? !  $20242403 ws !  main ;
    10 SET : $$Q ( - ) -1 done? ! ;
    FLUSH

Now we can test to see if the new event handler works exactly like the old one.

    BYE
    100 LOAD
    2 design

You should see the fatbit matrix and the cursor again, and it should not do anything with any key you type *except* for capital `Q`, which should exit the program.

Now that we have this logic implemented, we can now easily express our keyboard mappings for basic cursor movement.

    114 CLEAN
    0 SET ( keyboard mappings   saf2 2016apr23 )
    2 SET : $$w ( - ) goU ;
    3 SET : $$s ( - ) goD ;
    4 SET : $$a ( - ) goL ;
    5 SET : $$d ( - ) goR ;
    7 SET : $$i ( - ) goU ;
    8 SET : $$k ( - ) goD ;
    9 SET : $$j ( - ) goL ;
    10 SET : $$l ( - ) goR ;

    100 LIST
    8 SET 114 LOAD ( keyboard mappings )

I'm kind of showing off here; I'm mapping `WASD` and `IJKL` to the same cursor movement functions to facilitate left- and right-handed keyboard navigation.  Go ahead and try it out:

    FLUSH
    BYE
    100 LOAD
    2 design

Feel free to play around with the keyboard navigation keys.  Screen update should be quick and snappy.  Press `Q` (capital!) to quit back to Forth.  

## Toggling Bits

Ideally, I'd like to use the spacebar to toggle bits in a glyph.  But, this poses a bit of a problem with our current event dispatcher.  The space is what Forth uses to separate words; ergo, it cannot appear *in* a word.  Several approaches exist to remedy this: we could change our `handler` word to use hexadecimal ASCII codes in the words it looks up instead of the actual characters themselves, or we can add custom logic to detect spaces and dispatch accordingly.

The former approach requires changing a lot of lines of code: everything in block 114, and a few in block 104, and whatever is new for dispatching on the space (I figure, two lines of code).  That's less than 32 lines of code for sure; however, I think I can do it in only 4 to 5 lines of code if I just dispatch on the space explicitly.  For this reason, I choose the latter approach.

    104 LIST
    3 OPEN
    3 SET : -space ( c - c' ) DUP $20 = IF DROP 95 THEN ;
    5 SET : main ( - ) BEGIN KEY -space handler IF EXECUTE ELSE DROP THEN

    114 LIST
    11 SET : $$_ ( - ) 0 17 AT-XY ." Hello world!" CR ;
    FLUSH
    BYE
    100 LOAD
    0 design

At this point, confirm that the cursor keys still work.  Then, press the spacebar.  You should get a worldly greeting!

Now that we have the spacebar functionality activated, let's make it do something useful.  We know that we want to *toggle* a bit, which just screams for using `XOR` again, just like we did when drawing and erasing the cursor.  Thus, if we're given a memory address for a byte in a glyph, we can affect one of its bits like so (don't type this in; I'm merely illustrating here):

    ( addr mask - ) OVER C@ XOR SWAP C!

We need to compute both the `mask` and the `addr`, based on the current values of the cursor coordinates.

I'm going to start with the `mask`.  We know the cursor location is stored in the variables `cx` and `cy`; `cy` selects which byte in the glyph we want to affect, while `cx` determines which bit.  However, the mapping for bits is backwards from what most other software (and even the hardware) expects.  See figure for an illustration of what I mean.

Generating the `mask`, then, involves shifting the value `1` left by 7 positions if `cx` is 0, 6 if it's 1, etc.  In other words:

    ( addr - ) 1 cx @ 7 XOR LSHIFT
    ( addr mask - ) OVER C@ XOR SWAP C!

Calculating the address of the byte is similar to computing the address of a byte in the screen's frame buffer.  A single increment of `cy` implies a 256-byte span, so the byte we're interested in working with is *a* + 256\*`cy`, where *a* represents the *base address* of the glyph (byte 0 of the glyph).

    ( base - ) cy @ 256 * +
    ( addr ) 1 cx @ 7 XOR LSHIFT
    ( addr mask ) OVER C@ XOR SWAP C!

Finally, we can figure out the value of our glyph's base address by simply adding the address of `fontbuf`, which refers to the first byte of the first glyph, and the current character together.

    ( - ) fontbuf glyph @ +
    ( base ) cy @ 256 * +
    ( addr ) 1 cx @ 7 XOR LSHIFT
    ( addr mask ) OVER C@ XOR SWAP C!

OK, we have a strategy for how to change our bit.  Now let's get to really coding this thing.  First, let's make a word that lets us update a single fatbit, as determined by the `cx` and `cy` variables.  We have a word, `addr` on block 108, which calculates the desired screen address from a coordinate pair.  However, it's shadowed by another `addr` on block 106 later on.  After discovering the former is a useful word with reuse potential, let's rename it to something more meaningful than simply "address."

    108 LIST
    10 SET : tile ( x y - a ) 1280 * SWAP 2* + $FF0000 + ;
    12 SET : cursor ( x y - ) cursorImg -ROT tile
    109 LIST
    7 SET tile (x y-a) answers the framebuffer address correspond-
    FLUSH

Be sure to test to make sure everything still works.

    BYE
    100 LOAD
    0 design
    ( cursor around then shift-Q to quit )


![What the computer thinks the bits are, versus what the font editor thinks the bits are.](images/ch2...)

We need to translate the coordinate when figuring out which bit to affect.
