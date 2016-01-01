
# Anatomy of `CALIEN.COM`

## Abstract

`CALIEN.COM` is a cheat program for Alien Breed, an awesome (and for me, a bit
difficult) game created by Team17.

This cheat program allowed you to have infinite lives. You were still damaged
by aliens, but when you died, you didn't loose any life.

## Tools

* DOSBox 0.74
* GW-Basic
* MASM Toolkit

## The `BASIC` program

When MicroMania released their cheat programs, they didn't release it with any
digital support, so the only thing they could do is give the source code of a
program that would create the cheat program.

I used GW-Basic to execute the basic file. 

What does it do?

* Save the cheat program into disk with the name `calien.com`.
* Perform a checksum to verify that the cheat code has been copied correctly.

The checksum is done by adding each byte from `A$` (which contains the string
stored in `DATA` statements from `140` to `190`) to the variable `SUM`, and
then is compared to the expected value: `8388`.

This can be easily tested with this sample program written in Python:

```python
# Transformed data statements into a nice list of string bytes
data = ['EB','34','55','8B','EC','1E','8E','5E','04','81','3E','15','3A','FE',
 '4C','75','0B','C7','06','15','3A','90','90','C6','06','17','3A','90','1F',
 '5D','EA','00','00','00','00','43','61','72','67','61','20','41','4C','49',
 '45','4E','20','42','52','45','45','44','2E','24','FA','B8','00','00','8E',
 'D8','8B','1E','84','00','8B','0E','86','00','B8','02','01','A3','84','00',
 '8C','0E','86','00','0E','1F','89','1E','1F','01','89','0E','21','01','FB',
 'B4','09','BA','23','01','CD','21','BA','23','01','CD','27','00','00','00',
 '00','00','00','00','00','00']

# Create a new list by converting the list data into a list of integers
data_int = [ int(x, 16) for x in data ]

# Sum all values and compare it to the expected value
reduce(lambda x, y: x + y, data_int) == 8388
```

Also, this is pretty convenient to check if the `DATA` statements have been
copied correctly, because it is a serious PITA to check which byte is wrong.
The monospaced font used by MicroMania is seriously one of the worst I've ever
seen, because sometimes is pretty difficult to tell the difference between and
`8` and a `B`.

## The cheat program

Once `calien.com` is generated, you just have to run it, then run the game,
and let it do its magic. But what's doing behind the scene?

The `com` file is a program of less than 64KiB that is stored at a specific
location designated by MS-DOS. Using the MASM toolkit, there's a program called
`DEBUG.EXE` that allows to debug `com` executables.

This program does not have any boundaries between code and data. Both are mixed
and it is up to the programmer to use it wisely.

Within DOSBox, you can execute the next statement:

```
C:\DEBUG CALIEN.COM
```

The debugger loads the program, and you are able to step through the loaded
program, watch the CPU register values, or disassemble machine instructions.

There's more information available at [The Starman's MS-DOS DEBUG Guide]. It
offers a much more detailed documentation about available commands in `DEBUG`.

For now, let's just peek into the memory of the loaded process by writing in
the debugger `-U`:

![First 32 bytes in memory from segment 0760][debug_calien_00]

By DOS standard, a `com` executable is loaded at `CS:0100`, and by default,
`DEBUG` will try to disassemble the first 32 bytes from that direction. Mind 
that it can disassemble from any direction you ask it.

Let's take a look at the current state of the CPU registers by inputting `-R`:

![Current CPU registers][debug_calien_01]

As you can see, the code segment (`CS`) is pointing at `0x0760`, and the
instruction pointer (`IP`) is at `0x0100`. The instruction to be executed is
`JMP 0136`, so the first thing it will do is a relative jump to `0760:0136`.

So let's see what instructions are available at `CS:0136` by executing
`-U 136`:

![Unassembled instructions available at 0760:0136][debug_calien_02]

Like before, it tried to unassemble 32 bytes from `0136`. Let's just go a bit
farder by unassembling from `CS:0154` with `-U 154`:

![Unassembled instructions available at 0760:0154][debug_calien_03]

Now, take a look at both images with the unassembled code. The second column
indicates the hexadecimal representation of the instructions to be executed by
the CPU.

As you may have seen, those bytes are the ones from the cheat program, and we
can see that by opening our `com` file with a hex editor:

![CALIEN.COM][hexfiend_calien_01]

Keeping that in mind, we can understand a bit more how `calien.com` is
structured.

There's a first code segment of 32 bytes from `0760:0100` to `0760:0122`:

| Address     | Instruction                 |
| -----------:| --------------------------- |
| `0760:0100` | `JMP  0136`                 |
| `0760:0102` | `PUSH BP`                   |
| `0760:0103` | `MOV  BP,SP`                |
| `0760:0105` | `PUSH DS`                   |
| `0760:0106` | `MOV  DS,[BP+04]`           |
| `0760:0109` | `CMP  WORD PTR [3A15],4CFE` |
| `0760:010F` | `JNZ  011C`                 |
| `0760:0111` | `MOV  WORD PTR [3A15],9090` |
| `0760:0117` | `MOV  BYTE PTR [3A17],90`   |
| `0760:011C` | `POP  DS`                   |
| `0760:011D` | `POP  BP`                   |
| `0760:011E` | `JMP  0000:0000`            |

And then, there's another second code fragment of 56 bytes from `0760:0136` to
`0760:016E`

| Address     | Instruction       |
| -----------:| ----------------- |
| `0760:0136` | `CLI`             |
| `0760:0137` | `MOV  AX,0000`    |
| `0760:013A` | `MOV  DS,AX`      |
| `0760:013C` | `MOV  BX,[0084]`  |
| `0760:0140` | `MOV  CX,[0086]`  |
| `0760:0144` | `MOV  AX,0102`    |
| `0760:0147` | `MOV  [0084],AX`  |
| `0760:014A` | `MOV  [0086],CS`  |
| `0760:014E` | `PUSH CS`         |
| `0760:014F` | `POP  DS`         |
| `0760:0150` | `MOV  [011F],BX`  |
| `0760:0154` | `MOV  [0121],CX`  |
| `0760:0158` | `STI`             |
| `0760:0159` | `MOV  AH,09`      |
| `0760:015B` | `MOV  DX,0123`    |
| `0760:015E` | `INT  21`         |
| `0760:0160` | `MOV  DX,0123`    |
| `0760:0163` | `INT  27`         |
| `0760:0165` | `ADD  [BX+SI],AL` |
| `0760:0167` | `ADD  [BX+SI],AL` |
| `0760:0169` | `ADD  [BX+SI],AL` |
| `0760:016B` | `ADD  [BX+SI],AL` |

Though I tried to step through the program to see how it behaved, I don't know
why, but when trying to execute `0760:0147` from the debugger, it crashes, and
I cannot execute again the debugger until I restart DOSBox, so this part for
now will be a bit vague until I understand a bit more the innards of DOS.

But, I think it is pretty intuitive to understand what is happening in several
parts of the code. Let's start with the first 8 instructions:

```asm
CLI
MOV  AX,0000
MOV  DS,AX
MOV  BX,[0084]
MOV  CX,[0086]
MOV  AX,0102
MOV  [0084],AX
MOV  [0086],CS
```

The first instruction clears the interrupt flags. But the next instructions,
what I think they are doing is to set in some part of the memory in the data
segment `0000` that something has to be executed from `0760:0102`, instead of
what originally was intended to be executed, that now is available at `CX:BX`.

In DOSBox, `BX` has the value `14A0` and `CX` has the value `F000`. I haven't
tested this in a virtual machine, like VirtualBox, so I don't know if this
values would remain the same.

Let's dig up some more:

```asm
PUSH CS
POP  DS
MOV  [011F],BX
MOV  [0121],CX
STI
```

With the first 2 instructions, it resets the data segment to the original one.
Then, with the next two instructions, the cheat program edits its own
instruction at `0760:011E`, changing the code from `JMP 0000:0000` to
`JMP CX:BX`. Thus allowing the cheat program to continue with the original
execution flow, but executing the instructions from `0760:0102` before anything
else.

Lastly, it enables again interruptions. Let's see some more instructions.

```asm
MOV  AH,09
MOV  DX,0123
INT  21
```

This requires to take a look at the chapter thirteen of [The Art of Assembly],
specifically to the section **13.3.2 MS-DOS Character Oriented Functions**.
There, you can understand how to operate with the DOS interrupt oriented API.

What it is doing is, when the interruption `021h` is called with the
instruction `INT 21`, DOS will print on the terminal the string terminated with
the character `$` available at `DS:DX`. In our case, `0760:0123`.

When executing this three instructions, on terminal we will see the string
`Carga Alien Breed.`.

Now, let's go for the last two relevant instructions:

```asm
MOV  DX,0123
INT  27
```

Again, we have to take another look at `The Book`, but this time we have to check
the section **[13.3.7.2 Terminate, but Stay Resident]**:

> This function also terminates program execution, but upon returning to DOS,
> the memory in use by the process is not returned to the DOS free memory pool.
> Essentially, the program remains in memory. Programs which remain resident in
> memory after returning to DOS are often called TSRs (terminate and stay
> resident programs).

Essentially, what this is doing is saving `0x0123` paragraphs (or `291 * 16`
bytes) from the current process to be in memory, from `0760:0000` to the
quantity stated.

The other instructions left won't be executed, but I think they were left so
that the basic `DATA` statements would be aligned nicely.

Now that `calien.com` has been executed, we have our cheat program available 
in memory, and it has edited what I think it is a system call to execute the
first code segment of the cheat program.

So, now let's take a look at what will be executed in some time, with the
changes done by the cheat program:

| Address     | Instruction                 |
| -----------:| --------------------------- |
| `0760:0102` | `PUSH BP`                   |
| `0760:0103` | `MOV  BP,SP`                |
| `0760:0105` | `PUSH DS`                   |
| `0760:0106` | `MOV  DS,[BP+04]`           |
| `0760:0109` | `CMP  WORD PTR [3A15],4CFE` |
| `0760:010F` | `JNZ  011C`                 |
| `0760:0111` | `MOV  WORD PTR [3A15],9090` |
| `0760:0117` | `MOV  BYTE PTR [3A17],90`   |
| `0760:011C` | `POP  DS`                   |
| `0760:011D` | `POP  BP`                   |
| `0760:011E` | `JMP  CX:BX`                |

And now, let's look at the first four instructions:

```asm
PUSH BP
MOV  BP,SP
PUSH DS
MOV  DS,[BP+04]
```

It is saving the current base pointer and data segment into the stack, and
then, setting the data segment to an address available at the stack that I 
believe it points to the data segment of the Alien Breed's process.

```asm
CMP  WORD PTR [3A15],4CFE
JNZ  011C
MOV  WORD PTR [3A15],9090
MOV  BYTE PTR [3A17],90
```

Now that the cheat program is in the game's data segment, it will verify the
number of lives available at `DS:3A15`. If it's not zero, it will simply
restore the data segment and base pointer to its original values. Otherwise,
it will edit the value at that memory address.

Finally, the magic has been done. Infinite lives, at the expense of modifying
whatever system call might be. Just for the sake of one more live.

You're a cheater.


[The Starman's MS-DOS DEBUG]: http://thestarman.pcministry.com/asm/debug/debug.htm
[The Art of Assembly]: http://web.archive.org/web/20120414215452id_/http://www.arl.wustl.edu/~lockwood/class/cs306/books/artofasm/Chapter_13/CH13-4.html#HEADING4-9
[13.3.7.2 Terminate, but Stay Resident]: http://web.archive.org/web/20120414215013id_/http://www.arl.wustl.edu/~lockwood/class/cs306/books/artofasm/Chapter_13/CH13-5.html#HEADING5-50

[debug_calien_00]: https://dl.dropboxusercontent.com/u/9123154/cdn/cargadores/debug_calien_00.png
[debug_calien_01]: https://dl.dropboxusercontent.com/u/9123154/cdn/cargadores/debug_calien_01.png
[debug_calien_02]: https://dl.dropboxusercontent.com/u/9123154/cdn/cargadores/debug_calien_02.png
[debug_calien_03]: https://dl.dropboxusercontent.com/u/9123154/cdn/cargadores/debug_calien_03.png
[hexfiend_calien_00]: https://dl.dropboxusercontent.com/u/9123154/cdn/cargadores/hexfiend_calien_00.png
[hexfiend_calien_01]: https://dl.dropboxusercontent.com/u/9123154/cdn/cargadores/hexfiend_calien_01.png
