# Google CTF 2021 - cpp writeup
Originally from my Google Gist Writeup submission here: https://gist.github.com/Diesel-Hadez/a10e67d04c88ca005890b6f85c992fb3
From the description:-

```
We have this program's source code, but it uses a strange DRM solution. Can you crack it?
```

Attached was a c file, which can be found [here](https://github.com/google/google-ctf/blob/master/2021/quals/rev-cpp/attachments/cpp.c).

Looking into the file, we can see it is almost entirely made up of pre-processor commands, that is, everything that begins with a \#.

Funnily enough, despite the name of the challenge being "cpp", the code appears to be closer to C. My theory is that the "pp" in the 
"cpp" title _doesn't_ stand for "plus plus", but rather, "pre-processor".

## Initial Recon.

The code seems to be split into an _Intialisation section_ and a _Recursive section_. The initialisation section sets up various things,
such as the flag input, character encodings, the ROM, etc. The recursive section is done recursively by including the file over and over again
until it reaches a base condition. The Initialisation section only occurs in the first iteration of the source code being interpreted.
I will talk more in-depth about these later.

### Initialisation Section

This isn't too hard. The first thing we see upon opening the file, besides the copyright/license, is the following.

    // Please type the flag:
    #define FLAG_0 CHAR_C
    #define FLAG_1 CHAR_T
    #define FLAG_2 CHAR_F
    #define FLAG_3 CHAR_LBRACE
    #define FLAG_4 CHAR_w
    #define FLAG_5 CHAR_r
    #define FLAG_6 CHAR_i
    #define FLAG_7 CHAR_t


For those unfamiliar with C/C++, the \#define command acts as sort of a replacement done prior to compilation. For example,

    #define MAX_PEOPLE 100
    //...
    if (numPeople+1 >= MAX_PEOPLE){
        return 0;
    }
    
would be converted to the following

    //...
    if (numPeople+1 >= 100){
        return 0;
    }

Of course, you have to be very careful with doing this. It is very easy to make a mistake, and
the compiler doesn't have as much as an idea of _what_ you are trying to accomplish. It will not for example,
detect a syntax error of `numPeople+1 >= MAX_PEOPLE0` and will happily convert that to 1000. 
There are plenty of other gotchas (and benefits!), but this is getting out-of-topic.

It seems very clear that what is needed to be done is to replace the CHAR\_<Character\> with the correct flag.
The "CHAR\_C" and other CHAR\_<Characters\> are defined further down in the file.

    // No need to change anything below this line
    #define CHAR_a 97
    #define CHAR_b 98
    #define CHAR_c 99
    //...

Nothing out-of-the-ordinary. It seems to be simply defining it to be it's corresponding ASCII value.  The letter 'A' for example corresponds to ASCII
value 0x61, which is 97 in decimal, and 'B' corresponds to 98, and so forth. 

At the time of the CTF, I did consider that there might be a _nasty_ trick, where the first 10 or so characters were correctly defined,
but later on some values would be purposefully assigned the wrong value, as a sort of cheap way to trick the reader. I just put 
that as a possibility in my notes to come back to later, if I ever needed to diagnose an issue. Thankfully this turned out not to be the case.

I also found the "No need to change anything below this line" comment quite funny. I would _definitely_ end up changing stuff below that line,
as a form of debugging the program. But I suppose that was just a warning that I might risk breaking the code and getting an invalid checker.

Every so often, you would see a pre-processor command, '\#warning'. This is meant for programmers to issue warnings into the terminal
when compiling certain code. For example, there might be an optional '\#define EXPERIMENTAL_MODE' that if done by the user of a library,
would enable an unstable API that is prone to crashing, so the author of the library could put this in to warn and remind the programmer about it.
Here however, it was simply a creative way to write output to the terminal using preprocessor commands.

There is a `#define S 0`. We will talk about this later. The more pressing thing right now is the code immediately following that.

    #define ROM_00000000_0 1
    #define ROM_00000000_1 1
    #define ROM_00000000_2 0
    #define ROM_00000000_3 1
    #define ROM_00000000_4 1
    #define ROM_00000000_5 1
    #define ROM_00000000_6 0
    #define ROM_00000000_7 1
    #define ROM_00000001_0 1
    #define ROM_00000001_1 0
    #define ROM_00000001_2 1
    #define ROM_00000001_3 0
    #define ROM_00000001_4 1
    #define ROM_00000001_5 0
    #define ROM_00000001_6 1
    #define ROM_00000001_7 0
    //...

The author of the code was kind enough to provide a descriptive name. It followed the convention ROM\_X\_0, ROM\_X\_1, ROM\_X\_2, ROM\_X\_3, ROM\_X\_4, ROM\_X\_5, ROM\_X\_6, ROM\_X\_7, then ROM\_(X+1)\_0, ROM\_(X+1)\_1, etc. (where X is in binary) all the way up to ROM\_01011010\_7. Going by the name "ROM" alone, I assumed it was some sort of [Read-Only Memory](https://en.wikipedia.org/wiki/Read-only_memory). And because
there was a ROM embedded into the code, I assumed there would later on be an interpreter that _interprets_ the ROM to do some calculations, similar to what a real machine does.  I would soon find out that my hunch was right, but for now, let's focus on extracting the ROM to a nicer format.

They are all being assigned values of either 1 or 0, so it is trivial to see that those are bits. The right-most numbers are 0 to 7, before going to the next byte in the ROM. 
(I had assumed X was the ROM address), so I assumed that ROM\_X\_Y meant _bit Y in address X_ of the ROM. 

With some VIM Macros, I was able to only get the numbers on the right.

    1
    1
    0
    1
    1
    1
    0
    1
    1
    0
    1
    //...

And then add an additional newline every 8th line


    1
    1
    0
    1
    1
    1
    0
    1
    
    1
    0
    1
    0
    1
    0
    1
    0
    //...
    
And finally, get rid of single newlines.

    11011101
    10101010
    //...
    
Additionally, I wanted to work with this in python, so I added the necessary code (the 0b prefix and commas) for that with yet another vim macro.
    
    rom = [
    0b11011101,
    0b10101010,
    #...
    ]


Of course, this would mean the least significant bit is the leftmost bit, which isn't usually how it is expressed. To get around this, I would just wrote a simple
function to reverse the bits. For cleanliness, I also printed it out onto my terminal afterwards, and re-coded my rom variable to use a list of those instead.
For further compactness, you might want to use a list of hex digits or even load it from a binary file.

    rom = [
        0b10111011,
        0b01010101
        #...
        ]
        
(Note: I also outputted it into a binary file, and looked through it with a hex editor, unfortunately I didn't see anything obvious that would help me).

There was also a section where it loaded in the flag into the ROM starting at address 0b10000000.

    #if FLAG_0 & (1<<0)
    #define ROM_10000000_0 1
    #else
    #define ROM_10000000_0 0
    #endif
    #if FLAG_0 & (1<<1)
    #define ROM_10000000_1 1
    #else
    #define ROM_10000000_1 0
    #endif

Yes, The C Preprocessor has conditionals!

The value `(1 << X)` outputs to a number with only the Xth bit set, so it should be easy to see how this is just a primitive copy.

Finally, there was a section for some helper macros (think of them like functions).

    #define _LD(x, y) ROM_ ## x ## _ ## y
    #define LD(x, y) _LD(x, y)
    #define _MA(l0, l1, l2, l3, l4, l5, l6, l7) l7 ## l6 ## l5 ## l4 ## l3 ## l2 ## l1 ## l0
    #define MA(l0, l1, l2, l3, l4, l5, l6, l7) _MA(l0, l1, l2, l3, l4, l5, l6, l7)
    #define l MA(l0, l1, l2, l3, l4, l5, l6, l7)

The neat thing about this is that I could copy-paste these snippets into another C program, and `printf()` the output to see if I understood it correctly.
As it stands, The LD function returns (i.e: the code `LD(a,b)` is replaced by the return value) a value ROM\_<X\>\_<Y\>, which _itself_ returns the value in memory
in that particular location (and which bit). It seems to be a primitive for _loading_ a value from the ROM. 

The MA does something similar to what we discussed earlier about reversing the bits. Think of l0, l1, l2, l3, ... l7 as bits 0, 1, 2, 3, ..., 7. Although it doesn't
seem very clear, these are also variables in the preprocessor, as later on they get defined values. Every instance of `l` would then be replace with the address based on the bits of lX, where X is the Xth bit.

For example, to get the value at address 3, first convert 3 to binary and pad it to 8 bits. 00000011. Now set those bits in lX (0 and 1). Then call l

    #define l0 1
    #define l1 1
    #define l2 0
    #define l3 0
    #define l4 0
    #define l5 0
    #define l6 0
    #define l7 0
    #warning l // This is 00000011

Note that, for example, `l0` is NOT expanded to `l 0`, so the MA is not called when it sees `l0`, only when `l` is by itself.
### Recursive section
Curiously, this section of the code also only occurs if the Include Level (How many times it has included itself) of 12. Not sure why that specific value was chosen. Anything other than `S` being 0 would have been fine.

The rest of the code appeared to be a giant switch case for the `S` variable (remember that from earlier?), with each case doing something different. This section takes up the majority of the code, with none of the other sections even coming close. 

Skipping past this for a moment,
I went to the bottom of the code. 

    #if S != -1
    #include "cpp.c"
    #endif
    
This is the recursive part where it includes itself. This also tells me that when S=-1, the "program" terminates. This is the base-case. 

Note that when experimenting and copying and renaming the file, one has to be careful to also rename this portion.
As otherwise it would just include the original file instead, as I have fallen into that trap when going through this. Also note that for some reason, this code was included
twice, which I'm still unsure of what the reasoning for that is.

The actual C code of the program is not very interesting.

    #include <stdio.h>
    int main() {
    printf("Key valid. Enjoy your program!\n");
    printf("2+2 = %d\n", 2+2);
    }
    
## Clean-up
In C, at least in code I'm familiar with, preprocessor commands aren't usually indented like in python, even
when there are `#ifdef`s and `#endifs`. So, I simply ran `clang-format cpp.c > cleaner.c` to get some slightly nicer to read code.
For instance, now the following:-

    #if FLAG_0 & (1<<0)
    #define ROM_10000000_0 1
    #else
    #define ROM_10000000_0 0
    #endif
    
is this!

    #if FLAG_0 & (1 << 0)
        #define ROM_10000000_0 1
    #else
        #define ROM_10000000_0 0
    #endif

Ah! Much better. It especially nicer with the longer code segments.

## Analysis
I found a pre-processor command `#pragma message` to be quite helpful. It just prints out things in the terminal, and also allows me to inspect 
the values of other "variables" (defined statements). Here's an example:-

    #if S == 0
        #pragma message("S = " STRING(S))
        #undef S
        #define S 1
        #undef S
        #define S 24
    #endif

This "print-style debugging" was useful for checking if values are what I thought they would be. Usually this catched a few bugs in my program.
It would have been a lot more frustrating to manually step through it in my head. I suppose an actual debugger that allows you to "step through" it
would be even more helpful, but I am not aware of such a tool. C/C++ debuggers AFAIK don't have this functionality.

Now, judging by the S switch case, I figured this could be some sort of [finite-state machine](https://en.wikipedia.org/wiki/Finite-state_machine) with a LOT of different states, which I interpreted as meaning different
instructions like in an instruction set of a particular computer architecture.

Upon further reflection, I realised it was more analogous to a Program Counter (Or Instruction Pointer), as S was looping back to various locations in the code.
Let's look at `S=0`,


    #if S == 0
        #undef S
        #define S 1
        #undef S
        #define S 24
    #endif

Like I said, note how the "Program Counter `S`" is re-assigned the value 24 to jump to that "location" (Note that this is DIFFERENT than the ROM location!).
Breaking it down into a more conventional assembly syntax, this can be seen as "JMP 24".

I'm not sure why it is redundantly assigned the value of 1 before that, but in most instructions here, the next instruction is the immediately following instructions.
I take it that this C pre-processor code was generated by another higher-level interface, and it was decided to just have every instruction have this "jump to next instruction"
code since that happens in most instructions, and in those that do other branches, it would simply just reassign S.

I decided to make my own interpreter for this language (so that I could more easily debug it) , and rewrote this in python.

        if globals['S'] == 0:
            debug_print('Opcode: ' + str(globals['S']))
            asm_print("JMP 24")
            globals['S'] = 24

Now to just do this 57 more times! Unfortunately, they weren't all as easy as this. Some definitely were as easy as this,
(e.g: defining other globals like `A0` for A's 0th bit, `A1` for A's 1st bit,  some instructions just negated existing global variables, and some instructions
were _exact_ copies of other instructions!) and so I started implementing those first in python.

But some instructions left me scratching my head.
Take instruction 46 for example. The assembly is rather long, so here's a direct translation to python.

    globals['S'] = 47
    for i in range(8):
        if in_globals('C' + str(i)):
            if in_globals('A' + str(i)):
                globals.pop('A' + str(i), None)
            else:
                globals['A' + str(i)] = 'DEFINED'

The `in_globals` function is simply for the `#ifdef` statement, think of it as checking if it is in a defined or undefined state yet.
The `i` is the `i`th bit. (e.g: in_globals('C' + str(i)) is True if bit i of C is 1, False if 0)

The `pop` sets the `i`th bit to 0. The ` = 'DEFINED'` sets the `i`th value to 1. Not the most elegant code, but I was in a hurry.

As for the for loop; in the original assembly code was just unrolled. A repeating mess of the same code-- save for `i`, over and _over_ again.

Now, I couldn't figure out what this did. A better Maths student than me might be able to immediately spot it, but I am a mere human.
Instead, I enclosed this section of code into a function, and tried calling it with various different parameters of A and C to see if I could spot any connection.
Through trial and error, I eventually figured out this was an Exclusive OR function, `A = A XOR C`. 
As soon as I found this, I probably should have just paid attention to where this function was called and logged the parameters, 
as XOR is a common method of encrypting a value in a CTF, so as to try get the correct flag that way. But at this point, I was heavily invested in disassembling the program.

I repeated this process of "reimplementation and experimenting with parameters" for many other instructions until I figured out what it does. 
Some of them were additions, subtractions, multiplications, ORs, etc.

Just for for fun, here's another instruction that I recoded to python directly to figure out what it does. Can you guess what it does?

    def mystery(x: int):
        x &= (1 << 32)-1
        c = False

        # Find the first 0
        # unset it.
        for i in range(8):
            if not bit_set(x, i):
                if c:
                    x = set_bit(x,i)
                    c = False
            else:
                if not c:
                    x = unset_bit(x,i)
                    c = True
        return x & 0xFF

`bit_set(x,i)` justs tells you whether the `i`th bit of `x` is set.

`set_bit(x,i)` returns `x` but with the `i`th bit set as well.

The 0xFF masking is probably not needed, I did it since we were working with 8 bit values and sometimes it wouldn't work without it.

SPOILERS: The mystery function returns x multiplied by 2. Yes, a simple single shift left would have sufficed, 
but I suppose that would have been _too easy_ to see, so they went with this implementation.

What was really annoying is that some instructions were duplicated, and some had very subtle changes in them. It made it so that I had to look carefully to be sure that it
_really_ was the same instruction. This could be the difference between an "addition" instruction and a "subtraction" instruction, which would obviously break my entire interpreter/disassembler.

Eventually, I managed to decode all the "instructions", and made a corresponding pseudo-assembly code. (If not otherwise noted with a `JMP` or an `S` assignment, the next instruction is the one immediately following it)

    "-1: ERROR",
    "0: JMP 24",
    "1: R = ~R",
    "2: Z = 1",
    "3: R += Z",
    "4: R += Z",
    "5: IF R=0: S=38",
    "6: R += Z",
    "7: IF R=0: S=59",
    "8: R += Z",
    "9: IF R=0: S=59",
    "10: BUG; S = 11",
    "11: ERR",
    "12: X = 1",
    "13: Y = 0",
    "14: IF X=0: S=22",
    "15: Z = X",
    "16: Z = Z & B",
    "17: IF Z=0: S=19",
    "18: Y = A + Y",
    "19: X = X * 2",
    "20: A = A * 2",
    "21: JMP 14",
    "22: A = Y",
    "23: JMP 1",
    "24: I = 0",
    "25: M = 0",
    "26: N = 1",
    "27: P = 0",
    "28: Q = 0",
    "29: B = 0b11101001 (aka 233 or 0xe9)",
    "30: B = B + I",
    "31: IF B=0: S=56",
    "32: B = 0b10000000 ( aka 128 or 0x80 )",
    "33: B = B + I",
    "34: A = rom[B] (B is reversed);",
    "35: B = rom[I] (I is reversed);",
    "36: R = 1",
    "37: JMP 12",
    "38: O = M",
    "39: O = O + N",
    "40: M = N",
    "41: N = O",
    "42: A = A + M",
    "43: B = 0b00100000 ( aka 32 or 0x20 )",
    "44: B = B + I",
    "45: C = rom[B] (B is reversed);",
    "46: A = A ^ C (XOR)",
    "47: P = P + A",
    "48: B = 0b01000000 aka 64 aka 0x40",
    "49: B = B + I",
    "50: A = rom[B] (B is reversed);",
    "51: A = A ^ P (XOR)",
    "52: Q = Q | A;",
    "53: A = 1;",
    "54: I = I + A;",
    "55: JMP 29",
    "56: IF Q=0: S=58 (FLAGCHECK)",
    "57: INVALID_FLAG_PRINT",
    "58: CORRECT_FLAG",

Note how some instructions, like instruction 34 and 35, did in fact make use of the ROM, in case you were wondering at this point what was the use of that.

Notice the FlagChecking instruction (56), "IF Q=0: S=58", when S is 58, it prints out that it has the correct flag. The only instruction using Q (besides the initialisation to 0) is instruction 52,
`Q = Q | A;`. This would mean that `A` would always have to be 0 in order for the flag to be correct, and lo and behold, on lines 50 and 51, A is loaded with a value in the ROM,
and XORed with C. So my hunch of a XOR cipher was indeed likely correct, and I could have gotten the flag this way by logging the parameters when XORing.

Not wanting to take things easy, I decided to decompile the entire program.

    // I might have gotten some things wrong here
    std::uint8_t index = 0;
    std::uint8_t M = 0;
    std::uint8_t N = 1;
    std::uint8_t xor_me = 0;
    std::uint8_t dont_zero_me = 0;

    constexpr std::uint8_t FLAG_START = 0x80;

    while (true) {
        flag_ptr = 233;
        flag_ptr = flag_ptr+index;

        // 23 loops until it overflows to 0
        if (flag_ptr != 0) {
            flag_ptr = FLAG_START;
            flag_ptr += index;
            
            mod_cur_flag_char = rom[flag_ptr];
            
            encrypted_flag_char = rom[index];
            R = 1;
            
            X = 1;
            Y = 0;
            // This while loop isn't infinite because the 8-bit X variable will eventually overflow and become 0
                // The values of X
                // 0b00000001
                // 0b00000010
                // 0b00000100
                // 0b00001000
                // 0b00010000
                // 0b00100000
                // 0b01000000
                // 0b10000000
            
            // mod_cur_flag_char is modified after this.
            while (X != 0) {
                // If the Xth bit of encrypted_flag_char is set
                if ((X & encrypted_flag_char) != 0) {
                    Y += mod_cur_flag_char;
                }
                // X <<= 1
                X *= 2;
                mod_cur_flag_char *= 2;
            }
            
            mod_cur_flag_char = Y;
            // I think this is just to check that your emulator handles overflows correctly
            R = ~R; // 0b11111110
            Z = 1;  // 0b00000001
            R += Z; // 0b11111111
            R += Z; // 0b00000000 (carry bit is ignored)
            if (R != 0) {
                R += Z; // 0b00000001
                if (R == 0) {
                    // BAD exit
                    return 59;
                }
                R += Z; // 0b00000002
                // Should never happen
                if (R == 0) {
                    // BAD exit
                    return 59;
                }
                std::printf("BUG, shouldn't reach this point");
            }
            
            O = M + N;
            M = N;
            N = O;
            
            mod_cur_flag_char += M;
    
            adjusted_index = 32+index;
            rom_32 = rom[adjusted_index];
            mod_cur_flag_char = mod_cur_flag_char ^ rom_32;

            xor_me += mod_cur_flag_char;
            
            adjusted_index = 64+index;
            rom_64 = rom[adjusted_index];
            rom_64 = rom_64 ^ xor_me;
            
            // rom_64 has to be 0 ALL the time
            dont_zero_me = dont_zero_me | rom_64;
            index += 1;
        }
        else {
            break;
        }
    }

    if (dont_zero_me == 0) {
        std::printf("Correct Flag!");
        return 0;
    }


And then I turned it to python and brute-forced the flag.
    
    import string
    
    rom = [
    # <snip>
    ]

    # ROM seems to go up to address 155, but I allocated some extra headroom in case I was wrong.
    ROM_LENGTH = 0xFF+1
    rom = rom[:] + [0x00]*(ROM_LENGTH-len(rom))
    
    def gety(e, p):
        x = 1
        ret = 0
        while x != 0:
            if (x & e) != 0:
                ret = (ret + p) & 0xFF
            x = (x * 2) & 0xFF
            p = (p * 2) & 0xFF
        return ret

    xor_me = 0
    O = 0
    M = 0
    N = 1 # I didn't see this the first time and was the cause of hours of frustration

    real_flag = ''
    FLAG_START = 0x80
    for i in range(27):
        O = M + N
        M = N
        N = O
        
        y = M
        rom32 = rom[32+i]
        rom64 = rom[64+i]
        
        # Let's explore the possibilities
        for c in string.printable:
            rom[FLAG_START+i] = ord(c)
            local_y = y + gety(rom[0+i], rom[FLAG_START+i])

            local_y = local_y ^ rom32

            local_xor_me = xor_me + local_y
            local_xor_me &= 0xFF
            
            if (rom64 ^ local_xor_me) == 0:
                y += gety(rom[0+i], rom[FLAG_START+i])
                y = y ^ rom32
                xor_me = xor_me + y
                xor_me &= 0xFF
                real_flag += c
                break

    print('Flag:  ' + real_flag)

You could not imagine my delight upon seeing the flag pop up on my screen. `CTF{pr3pr0cess0r_pr0fe5sor}`. I think it was in the wee hours of the morning,
and I had not slept, having spent the entire day on it. Ironically, the next challenge I would solve would be done not even a couple hours after I solved this,
before I even went to sleep.

## Closing thoughts
I do feel like I should have focused on getting the results without caring about the process. At one point, I figured out I could brute-force the flag by trying character-by-character and seeing how early the program terminates. But there was a side of me that _really_ wanted to get a "nice" solution, in which I understood everything inside out.
On the other hand, it definitely would have saved me time and left me more time to deal with other challenges. I suppose what I'm trying to say is that I really enjoyed this challenge, so much so that I didn't want to "move on" or "take the easy way out" when doing it. Props to the author of this challenge!
