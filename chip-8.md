# CHIP-8

**CHIP-8** is a simple and minimalist virtual machine and programming language that was designed in the 1970s to run on early microcomputers, primarily for educational purposes. It allows programmers to create basic games and applications in a time when resources (memory, processing power, etc.) were limited.

CHIP-8 was not tied to any specific hardware but was intended to run on various machines, and because it was so simple, it became a popular choice for developers and educators. Let’s dive into its details:

---

### 1. **Background and Purpose**:

- **Developed by**: Joseph Weisbecker in 1977.
- **Original Purpose**: It was designed to allow programming for early home computers like the **COSMAC VIP** and **Telmac 1800**, which used RCA's 1802 microprocessor. CHIP-8 was made to give these machines basic graphics and game capabilities.
- **Target Platform**: CHIP-8 was used on microcomputers with very limited memory, typically 4 KB of RAM or less.

Despite being simple, CHIP-8 provided a functional environment for creating graphical applications like simple games.

---

### 2. **Structure of CHIP-8**:

The CHIP-8 system consists of:

- **Interpreter**: A program running on a real machine (like a modern computer or an emulator) that simulates the CHIP-8 virtual machine. This interpreter decodes and executes CHIP-8 instructions.
- **Instruction Set**: A set of machine-level instructions that the virtual machine understands. These instructions are used to manipulate memory, perform arithmetic, and control graphics and sound.
- **Memory**: CHIP-8 provides a small, fixed memory space (typically 4 KB), which holds both program code and data.

---

### 3. **Key Components**:

#### a. **Registers**:

CHIP-8 has a set of 16 **8-bit registers** labeled **V0 to VF**.Each register is able to hold any value from 0x00 to 0xFF. These registers are used to store temporary data during the execution of a program. One of these registers (**VF**) is often used as a "carry" flag or used during certain arithmetic operations.

- **V0 to VE**: General-purpose registers used for various operations.
- **VF**: Often used as a flag (like the carry flag in arithmetic operations).

#### b. **Memory**:

- **4 KB of Memory**: The CHIP-8 has 4096 bytes of memory, meaning the address space is from `0x000` to `0xFFF`.
  The address space is segmented into three sections:

- `0x000-0x1FF`: Originally reserved for the CHIP-8 interpreter, but in our modern emulator we will just never write to or read from that area. Except for…

- `0x050-0x0A0`: Storage space for the 16 built-in characters (0 through F), which we will need to manually put into our memory because ROMs will be looking for those characters.

- `0x200-0xFFF`: Instructions from the ROM will be stored starting at 0x200, and anything left after the ROM’s space is free to use.


Most Chip-8 programs start at location 0x200 (512), but some begin at 0x600 (1536). Programs beginning at 0x600 are intended for the ETI 660 computer.

```bash
+---------------+= 0xFFF (4095) End of Chip-8 RAM
|               |
|               |
|               |
|               |
|               |
| 0x200 to 0xFFF|
|     Chip-8    |
| Program / Data|
|     Space     |
|               |
|               |
|               |
+- - - - - - - -+= 0x600 (1536) Start of ETI 660 Chip-8 programs
|               |
|               |
|               |
+---------------+= 0x200 (512) Start of most Chip-8 programs
| 0x000 to 0x1FF|
| Reserved for  |
|  interpreter  |
+---------------+= 0x000 (0) Start of Chip-8 RAM
```


The ETI 660 is an early microcomputer that is historically significant for being one of the platforms for which Chip-8 was created. It was a relatively simple computer used primarily in educational settings and by hobbyists in the 1970s.




#### c. **16-bit Program Counter (PC)**:

The actual program instructions are stored in memory starting at address 0x200. The CPU needs a way of keeping track of which instruction to execute next.

The Program Counter (PC) is a special register that holds the address of the next instruction to execute. Again, it’s 16 bits because it has to be able to hold the maximum memory address (0xFFF).

An instruction is two bytes but memory is addressed as a single byte, so when we fetch an instruction from memory we need to fetch a byte from PC and a byte from PC+1 and connect them into a single value. We then increment the PC by 2 because We have to increment the PC before we execute any instructions because some instructions will manipulate the PC to control program flow. Some will add to the PC, some will subtract from it, and some will change it completely.

#### d. **16-bit Index Register**:

The Index Register is a special register used to store memory addresses for use in operations. It’s a 16-bit register because the maximum memory address (0xFFF) is too big for an 8-bit register.

#### e. **Stack**:

16-level Stack
A stack is a way for a CPU to keep track of the order of execution when it calls into functions. There is an instruction (CALL) that will cause the CPU to begin executing instructions in a different region of the program. When the program reaches another instruction (RET), it must be able to go back to where it was when it hit the CALL instruction. The stack holds the PC value when the CALL instruction was executed, and the RETURN statement pulls that address from the stack and puts it back into the PC so the CPU will execute it on the next cycle.

The CHIP-8 has 16 levels of stack, meaning it can hold 16 different PCs. Multiple levels allow for one function to call another function and so on, until they all return to the original caller site.

Putting a PC onto the stack is called pushing and pulling a PC off of the stack is called popping.

#### f. **8-bit Stack Pointer**:

Similar to how the PC is used to keep track of where in memory the CPU is executing, we need a Stack Pointer (SP) to tell us where in the 16-levels of stack our most recent value was placed (i.e, the top).

We only need 8 bits for our stack pointer because the stack will be represented as an array, so our stack pointer can just be an index into that array. We only need sixteen indices then, which a single byte can manage.

When we pop a value off the stack, we won’t actually delete it from the array but instead just copy the value and decrement the SP so it “points” to the previous value.

With each CALL, the current PC (which was previously incremented to point to the next instruction) is placed where the SP was pointing, and the SP is incremented.

With each RET, the stack pointer is decremented by one and the address that it’s pointing to is put into the PC for execution.

#### g. **Delay and Sound Timers**:

- 8-bit Delay Timer :
  The CHIP-8 has a simple timer used for timing. If the timer value is zero, it stays zero. If it is loaded with a value, it will decrement at a rate of 60Hz.

Rather than making sure that the delay timer actually decrements at a rate of 60Hz, I just decrement it at whatever rate we have the cycle clock set to (discussed later) which has worked fine for all the games I’ve tested.

- 8-bit Sound Timer : 
  The CHIP-8 also has another simple timer used for sound. Its behavior is the same (decrementing at 60Hz if non-zero), but a single tone will buzz when it’s non-zero. Programmers used this for simple sound emission.


#### h. **Key Pads**:

The CHIP-8 has 16 input keys that match the first 16 hex values: 0 through F. Each key is either pressed or not pressed.
It does not really matter how you implement the key mapping, but I suggest something as on the right side.

```bash
Keypad       Keyboard
+-+-+-+-+    +-+-+-+-+
|1|2|3|C|    |1|2|3|4|
+-+-+-+-+    +-+-+-+-+
|4|5|6|D|    |Q|W|E|R|
+-+-+-+-+ => +-+-+-+-+
|7|8|9|E|    |A|S|D|F|
+-+-+-+-+    +-+-+-+-+
|A|0|B|F|    |Z|X|C|V|
+-+-+-+-+    +-+-+-+-+
```


#### i. **64x32 Monochrome Display Memory**:


The CHIP-8 has an additional memory buffer used for storing the graphics to display. It is 64 pixels wide and 32 pixels high. Each pixel is either on or off, so only two colors can be represented.

Understanding and then emulating its operation is probably the most challenging part of the entire project

 As mentioned, a pixel can be either on or off. In our case, we’ll use a uint32 for each pixel to make it easy to use with SDL, so on is 0xFFFFFFFF and off is 0x00000000.


 `sprite` : In CHIP-8, a sprite is a small, graphical object or image that is drawn on the screen, typically used for creating visual elements in games (such as characters, obstacles, or other objects). Sprites are essentially binary data that represent a set of pixels in a small rectangular shape, and they are drawn using a specific CHIP-8 instruction.

Sprites in CHIP-8 are typically 8 pixels wide and n pixels tall (usually 1 to several rows of 8 pixels).
The width is always 8 pixels because each byte of memory can represent a row of 8 pixels (since 1 byte = 8 bits, and each bit represents a single pixel).


The draw instruction iterates over each pixel in a sprite and XORs the sprite pixel with the display pixel.

- Sprite Pixel Off XOR Display Pixel Off = Display Pixel Off
- Sprite Pixel Off XOR Display Pixel On = Display Pixel On
- Sprite Pixel On XOR Display Pixel Off = Display Pixel On
- Sprite Pixel On XOR Display Pixel On = Display Pixel Off


In other words, a display pixel can be set or unset with a sprite. This is often done to only update a specific part of the screen. If you knew you had drawn a sprite at (X,Y) and you now want to draw it at (X+1,Y+1), you could first issue a draw command again at (X,Y) which would erase the sprite, and then you could issue another draw command at (X+1,Y+1) to draw it in the new location. This is why moving objects in CHIP-8 games flicker.
