# chip-8

[click here](./chip-8.md) to learn about chip-8

- ## Class Members

class data could look like this

```cpp
class Chip8
{
public:
	uint8_t registers[16]{};
	uint8_t memory[4096]{};
	uint16_t index{};
	uint16_t pc{};
	uint16_t stack[16]{};
	uint8_t sp{};
	uint8_t delayTimer{};
	uint8_t soundTimer{};
	uint8_t keypad[16]{};
	uint32_t video[64 * 32]{};
	uint16_t opcode;
};
```

- ## ROM LOADING

we need to have those instructions in memory, so we’ll need a function that loads the contents of a ROM file.

```cpp
#include <fstream>

const unsigned int START_ADDRESS = 0x200;

void Chip8::LoadROM(char const* filename)
{
	// Open the file as a stream of binary and move the file pointer to the end
	std::ifstream file(filename, std::ios::binary | std::ios::ate);
    //std::ios::ate: This flag causes the file pointer to be moved to the end of the file immediately after opening it. This is used to get the size of the file.

	if (file.is_open())
	{
		// Get size of file and allocate a buffer to hold the contents
		std::streampos size = file.tellg();
        //tellg() returns the current position of the file pointer. Since the file pointer is at the end of the file (due to std::ios::ate), this value represents the size of the file in bytes
        //std::streampos is the type used to represent this position/size.
		char* buffer = new char[size];

		// Go back to the beginning of the file and fill the buffer
		file.seekg(0, std::ios::beg);
        //file.seekg(0, std::ios::beg); moves the file pointer back to the beginning of the file (because tellg() moved it to the end earlier).
		file.read(buffer, size);
        //file.read(buffer, size); reads the entire content of the file into the buffer. The size variable tells read() how many bytes to read.
		file.close();

		// Load the ROM contents into the Chip8's memory, starting at 0x200
		for (long i = 0; i < size; ++i)
		{
			memory[START_ADDRESS + i] = buffer[i];
		}

		// Free the buffer
		delete[] buffer;
	}
}
```

Chip8’s memory from 0x000 to 0x1FF is reserved, so the ROM instructions must start at 0x200.

We also want to initially set the PC to 0x200 in the constructor because that will be the first instruction executed.

```cpp
Chip8::Chip8()
{
	// Initialize PC
	pc = START_ADDRESS;   //program counter
}

```

- ## Fonts LOADING

There are sixteen characters that ROMs expected at a certain location so they can write characters to the screen, so we need to put those characters into memory.

The characters are examples of sprites, which we’ll see more of later. Each character sprite is five bytes.

The character F, for example, is 0xF0, 0x80, 0xF0, 0x80, 0x80. Take a look at the binary representation:

```bash
11110000
10000000
11110000
10000000
10000000
```

Can you see it? Each bit represents a pixel, where a 1 means the pixel is on and a 0 means the pixel is off.
if not lets exclude the 0's

```bash
1111
1
1111
1
1
```

now its look more like F

We need an array of these bytes to load into memory. There are 16 characters at 5 bytes each, so we need an array of 80 bytes.
so these 16 characters (hexadecimal digits 0-9 and A-F) in CHIP-8 can indeed be used to create basic text or graphical content, though with some limitations.

```cpp
const unsigned int FONTSET_SIZE = 80;

uint8_t fontset[FONTSET_SIZE] =
{
	0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
	0x20, 0x60, 0x20, 0x20, 0x70, // 1
	0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
	0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
	0x90, 0x90, 0xF0, 0x10, 0x10, // 4
	0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
	0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
	0xF0, 0x10, 0x20, 0x40, 0x40, // 7
	0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
	0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
	0xF0, 0x90, 0xF0, 0x90, 0x90, // A
	0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
	0xF0, 0x80, 0x80, 0x80, 0xF0, // C
	0xE0, 0x90, 0x90, 0x90, 0xE0, // D
	0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
	0xF0, 0x80, 0xF0, 0x80, 0x80  // F
};

```

here each line is an sprite

lets see

```bash
	0xF0, 0x90, 0x90, 0x90, 0xF0, # 0
```

```bash
     c1 c2 c3 c4 c5 c6 c7 c8
R1 :  1  1  1  1  0  0  0  0
R2 :  1  0  0  1  0  0  0  0
R3 :  1  0  0  1  0  0  0  0
R4 :  1  0  0  1  0  0  0  0
R5 :  1  1  1  1  0  0  0  0
```

The 5-byte size for each sprite is because each character is represented as a 5x5 grid of pixels.

now we have to load these chracters in the memory(ROM)
`0x050-0x0A0`: Storage space for the 16 built-in characters (0 through F), which we will need to manually put into our memory because ROMs will be looking for those characters.

```cpp
const unsigned int FONTSET_START_ADDRESS = 0x50;

Chip8::Chip8()
{

	// Load fonts into memory
	for (unsigned int i = 0; i < FONTSET_SIZE; ++i)
	{
		memory[FONTSET_START_ADDRESS + i] = fontset[i];
	}
}
```

- ## Random Number Generator

There is an instruction which places a random number into a register. With physical hardware this could be achieved by, reading the value from a noisy disconnected pin or using a dedicated RNG chip, but we’ll just use C++’s built in random facilities.
The random number generator in the Chip-8 emulator simulates the hardware's ability to generate random numbers. By using C++'s built-in random functions and seeding the RNG with the system clock, the emulator can generate unpredictable values, just like the original hardware did. This randomness is essential for gameplay in many Chip-8 games.

```cpp
std::default_random_engine randGen; // seed of random byte
std::uniform_int_distribution<uint8_t> randByte; // uint8 ensures values store in a byte
```

The constructor will look like

```cpp
Chip8()
		: randGen(std::chrono::system_clock::now().time_since_epoch().count())
	{
		// Initialize RNG
		randByte = std::uniform_int_distribution<uint8_t>(0, 255U);
	}

```

- ## Instructions

The CHIP-8 has 36 instructions that we need to emulate.
CHIP-8 programs are strictly hexadecimal based. This means that the format of a CHIP-8 program bears little resemblance to the text-based formats of higher level languages. Each CHIP-8 instruction is two bytes in length and is represented using four hexadecimal digits. For example, one common instruction is the `00E0` instruction, which is used to clear the screen of all graphics data.

Certain CHIP-8 instructions accept ‘arguments’ to specify the values which should be read or modified by a given instruction when encountered by the interpreter. An argument is passed to an instruction of this type also as a hexadecimal digit.

Here is the list of **Chip-8 instructions**:

---

# `00E0 - CLS`

Clear the display.

We can simply set the entire video buffer to zeroes.

```cpp
void Chip8::OP_00E0()
{
	memset(video, 0, sizeof(video));
}
```

---

# `00EE - RET`

Return from a subroutine.

The top of the stack has the address of one instruction past the one that called the subroutine, so we can put that back into the PC. Note that this overwrites our preemptive pc += 2 earlier.

```cpp
void Chip8::OP_00EE()
{
	--sp;
	pc = stack[sp];
}
```

# `1nnn - JP addr`

Jump to location nnn.
The interpreter sets the program counter to nnn.
A jump doesn’t remember its origin, so no stack interaction required.

```cpp
void Chip8::OP_1nnn()
{
	uint16_t address = opcode & 0x0FFFu;

	pc = address;
}
```

lets take an example If the opcode is `1ABC`
opcode & 0x0FFF will result in 0x0ABC (the address part).
by performing and operator this will cause the 4 most significant bits to become 0 where the rest will remain same
The program counter pc will be set to 0xABC, and the program will jump to that location.

---

# `2nnn - CALL addr`

Call subroutine at nnn.

When we call a subroutine, we want to return eventually, so we put the current PC onto the top of the stack. Remember that we did pc += 2 in Cycle(), so the current PC holds the next instruction after this CALL, which is correct. We don’t want to return to the CALL instruction because it would be an infinite loop of CALLs and RETs.

```cpp
void Chip8::OP_2nnn()
{
	uint16_t address = opcode & 0x0FFFu;

	stack[sp] = pc;
	++sp;
	pc = address;
}
```

# `3xkk - SE Vx, byte`

The 3xkk instruction in Chip-8 is SE Vx, byte, which means "Skip next instruction if Vx equals kk." This is a comparison instruction that allows the program to conditionally skip the next instruction based on the value stored in a register.
Skip next instruction if Vx = kk.
Since our PC has already been incremented by 2 in Cycle(), we can just increment by 2 again to skip the next instruction.

This instruction is commonly used in control flow operations where you want to execute some code conditionally. Specifically, it's used to make decisions or skip over certain parts of code when a specific condition is met.

```cpp
void Chip8::OP_3xkk()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t byte = opcode & 0x00FFu;

	if (registers[Vx] == byte)
	{
		pc += 2;
	}
}
```

The first nibble (3) represents the instruction type (in this case, a comparison).
The second nibble (x) represents the register index (which can range from V0 to VF).
The last byte (kk) is the value to compare against.

---

# `4xkk** - SNE Vx, byte`

Skip next instruction if Vx != kk.
Since our PC has already been incremented by 2 in Cycle(), we can just increment by 2 again to skip the next instruction.

```cpp
void Chip8::OP_4xkk()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t byte = opcode & 0x00FFu;

	if (registers[Vx] != byte)
	{
		pc += 2;
	}
}
```

# `5xy0** - SE Vx, Vy`

Skip next instruction if Vx = Vy.
The opcode 5xy0 corresponds to the instruction SE Vx, Vy, which means "skip the next instruction if the value in register Vx equals the value in register Vy."
Since our PC has already been incremented by 2 in Cycle(), we can just increment by 2 again to skip the next instruction.

```cpp
void Chip8::OP_5xy0()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	if (registers[Vx] == registers[Vy])
	{
		pc += 2;
	}
}
```

# `6xkk - LD Vx, byte`

Set Vx = kk.
LD Vx, kk means "load the value kk into register Vx".
This instruction simply takes the lower 8 bits of the opcode and stores that value into the specified register Vx.

```cpp
void Chip8::OP_6xkk()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t byte = opcode & 0x00FFu;

	registers[Vx] = byte;
}
```

# `7xkk - ADD Vx, byte`

Set Vx = Vx + kk.

```cpp
void Chip8::OP_7xkk()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t byte = opcode & 0x00FFu;

	registers[Vx] += byte;
}
```

# `8xy0 - LD Vx, Vy`

    Set Vx = Vy.

```cpp
void Chip8::OP_8xy0()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	registers[Vx] = registers[Vy];
}

```

# `8xy1 - OR Vx, Vy`

Set Vx = Vx OR Vy.

```cpp
void Chip8::OP_8xy1()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	registers[Vx] |= registers[Vy];
}
```

# `8xy2 - AND Vx, Vy`

Set Vx = Vx AND Vy.

```cpp
void Chip8::OP_8xy2()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	registers[Vx] &= registers[Vy];
}
```

14. **8xy3** - XOR Vx, Vy
    Set Vx = Vx XOR Vy.

```cpp
void Chip8::OP_8xy3()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	registers[Vx] ^= registers[Vy];
}
```

# `8xy4 - ADD Vx, Vy`

Set Vx = Vx + Vy, set VF = carry.
The values of Vx and Vy are added together. If the result is greater than 8 bits (i.e., > 255,) VF is set to 1, otherwise 0. Only the lowest 8 bits of the result are kept, and stored in Vx.

This is an ADD with an overflow flag. If the sum is greater than what can fit into a byte (255), register VF will be set to 1 as a flag.

```cpp
void Chip8::OP_8xy4()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	uint16_t sum = registers[Vx] + registers[Vy];

	if (sum > 255U)
	{
		registers[0xF] = 1;
	}
	else
	{
		registers[0xF] = 0;
	}

	registers[Vx] = sum & 0xFFu;
}
```

# `8xy5 - SUB Vx, Vy`

Set Vx = Vx - Vy, set VF = NOT borrow.
If Vx > Vy, then VF is set to 1, otherwise 0. Then Vy is subtracted from Vx, and the results stored in Vx.

```cpp
void Chip8::OP_8xy5()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	if (registers[Vx] > registers[Vy])
	{
		registers[0xF] = 1;
	}
	else
	{
		registers[0xF] = 0;
	}

	registers[Vx] -= registers[Vy];
}

```

Chip-8 works with unsigned 8-bit registers, so a borrow doesn't produce a negative result directly. Instead, it causes a wraparound, and the borrow is tracked by setting or clearing the VF flag.

If no borrow (i.e., Vx >= Vy), VF is set to 1 (no borrow).
If there is a borrow (i.e., Vx < Vy), VF is set to 0 (borrow occurred).

---

# `8xy6 - SHR Vx {, Vy}`

Set Vx = Vx SHR 1.
If the least-significant bit of Vx is 1, then VF is set to 1, otherwise 0. Then Vx is divided by 2.
A right shift is performed (division by 2), and the least significant bit is saved in Register VF.

```cpp
void Chip8::OP_8xy6()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	// Save LSB in VF
	registers[0xF] = (registers[Vx] & 0x1u);    //if ht elsb is 0 it will store 0 and if 1 then 1 will be stored

	registers[Vx] >>= 1;
}

```

---

# `8xy7 - SUBN Vx, Vy`

## reverse subtraction

Set Vx = Vy - Vx, set VF = NOT borrow.
If Vy > Vx, then VF is set to 1, otherwise 0. Then Vx is subtracted from Vy, and the results stored in Vx.

```cpp
void Chip8::OP_8xy7()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	if (registers[Vy] > registers[Vx])
	{
		registers[0xF] = 1;
	}
	else
	{
		registers[0xF] = 0;
	}

	registers[Vx] = registers[Vy] - registers[Vx];
}

```

---

# `8xyE - SHL Vx {, Vy}`

Set Vx = Vx SHL 1.
If the most-significant bit of Vx is 1, then VF is set to 1, otherwise to 0. Then Vx is multiplied by 2.
A left shift is performed (multiplication by 2), and the most significant bit is saved in Register VF.

```cpp
void Chip8::OP_8xyE()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	// Save MSB in VF
	registers[0xF] = (registers[Vx] & 0x80u) >> 7u;   //0x80u (binary 10000000 in little endian) is used to isolate the MSB.
    //>> 7u right shifts the result so that only the MSB remains (either 0 or 1).

	registers[Vx] <<= 1;
}
```

The VF flag is used to store the most-significant bit of Vx before the shift, indicating whether the shifted-out bit was 1 or 0

---

# `9xy0 - SNE Vx, Vy`

Skip next instruction if Vx != Vy.
Since our PC has already been incremented by 2 in Cycle(), we can just increment by 2 again to skip the next instruction.

```cpp
void Chip8::OP_9xy0()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;

	if (registers[Vx] != registers[Vy])
	{
		pc += 2;
	}
}
```

---

# `Annn - LD I, addr`

Set I = nnn.

The operation Annn - LD I, addr in the Chip-8 instruction set is a simple instruction that sets the index register (I) to a specific memory address, nnn, which is the 12-bit address specified by the opcode. This operation essentially loads the value nnn into the index register (I).

The index register I is a special 16-bit register in the Chip-8 system that is used to hold memory addresses for various purposes, such as for storing data, pointing to sprites, or pointing to the memory where the next instruction will be executed.

```cpp
void Chip8::OP_Annn()
{
	uint16_t address = opcode & 0x0FFFu;

	index = address;
}
```

---

# `Bnnn - JP V0, addr`

Jump to location nnn + V0.

```cpp
void Chip8::OP_Bnnn()
{
	uint16_t address = opcode & 0x0FFFu;

	pc = registers[0] + address;
}
```

Adding V0 to nnn provides flexibility for relative jumps. It lets the program's control flow adapt dynamically based on the contents of V0, allowing for more sophisticated behavior like conditional jumps, loops, and event-driven flow changes. Without adding V0, the jump would be fixed to a specific address, reducing flexibility.

---

# `Cxkk - RND Vx, byte`

Set Vx = random byte AND kk.

The Cxkk - RND Vx, byte instruction in the Chip-8 instruction set is used to set a register Vx to a random byte that is then bitwise ANDed with a byte (kk) specified in the opcode.

This instruction is typically used for generating random numbers with a specific range by limiting the output using a mask (kk), making it useful in certain games or applications where random events or actions need to occur.

```cpp
void Chip8::OP_Cxkk()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t byte = opcode & 0x00FFu;

	registers[Vx] = randByte(randGen) & byte;
    // randByte(randGen) generates a random byte (an 8-bit number, i.e., a value between 0 and 255).
}
```

So, in summary, the Cxkk instruction is primarily used for creating masked random bytes to introduce randomness into your program

---

# `Dxyn - DRW Vx, Vy, nibble`

Display n-byte sprite starting at memory location I at (Vx, Vy), set VF = collision.

We iterate over the sprite, row by row and column by column. We know there are eight columns because a sprite is guaranteed to be eight pixels wide.

If a sprite pixel is on then there may be a collision with what’s already being displayed, so we check if our screen pixel in the same location is set. If so we must set the VF register to express collision.

Then we can just XOR the screen pixel with 0xFFFFFFFF to essentially XOR it with the sprite pixel (which we now know is on). We can’t XOR directly because the sprite pixel is either 1 or 0 while our video pixel is either 0x00000000 or 0xFFFFFFFF.

The OP_Dxyn function is a Chip-8 operation that handles the drawing of a sprite on the screen. It takes in the values of Vx, Vy, and the height of the sprite (number of rows) from the opcode, then renders the sprite starting at the memory location I to the screen at the position specified by Vx and Vy. Additionally, it handles collision detection, setting the VF register if a collision occurs.

```cpp
void Chip8::OP_Dxyn()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t Vy = (opcode & 0x00F0u) >> 4u;
	uint8_t height = opcode & 0x000Fu;

	// Wrap if going beyond screen boundaries
	uint8_t xPos = registers[Vx] % VIDEO_WIDTH;
	uint8_t yPos = registers[Vy] % VIDEO_HEIGHT;

	registers[0xF] = 0;    //Resetting the Collision Flag

	for (unsigned int row = 0; row < height; ++row)
	{
		uint8_t spriteByte = memory[index + row];

		for (unsigned int col = 0; col < 8; ++col)
		{
			uint8_t spritePixel = spriteByte & (0x80u >> col);
			uint32_t* screenPixel = &video[(yPos + row) * VIDEO_WIDTH + (xPos + col)];

			// Each pixel is stored as a 32-bit unsigned integer (uint32_t), which allows each pixel to store color information or other relevant data. (For simplicity, each uint32_t is often used to represent whether the pixel is on or off, typically 0xFFFFFFFF for on and 0x00000000 for off).

			// The screen is arranged as a 2D grid, but video[] is a 1D array. To access the correct index in this 1D array, we need to convert the 2D coordinates (x, y) into a single index.

            // To calculate the index of a pixel on a 1D array (the video[] array), the formula is:
            // index = y * VIDEO_WIDTH + x

            // y is the row number (vertical position).
            // x is the column number (horizontal position).
            // VIDEO_WIDTH is the width of the screen in pixels.

			// The screen has multiple rows of pixels. Each row contains exactly VIDEO_WIDTH pixels.
            // To determine which row you are on, you multiply the row number (yPos + row) by VIDEO_WIDTH. This gives you how many pixels have been passed before reaching the desired row.

            // Once you know which row you're in, you add the column number (xPos + col) to find the exact position in that row.



			//is used to calculate the address of a pixel on the screen based on its position (x, y), and it stores the address in the screenPixel pointer.

			// Sprite pixel is on
			if (spritePixel)
			{
				// Screen pixel also on - collision
				if (*screenPixel == 0xFFFFFFFF)
				{
					registers[0xF] = 1;
				}

				// Effectively XOR with the sprite pixel
				*screenPixel ^= 0xFFFFFFFF;
			}
		}
	}
}

```

lets have an example in which we have an 1d array of

```bash
0 1 2
3 4 5
6 7 8
```

now i want to access 4 so what i will do is use 
`index = y * VIDEO_WIDTH + x`
this formulla it will more likly look like this

```bash
index = rowno * row_size + colno
```
so
rowno = 1
row_size = 2
colno = 2


index = 1 * 2 + 2 = 4 

so that is how we can access 1d array as an 2d array


---

# `Ex9E - SKP Vx`

Skip next instruction if key with the value of Vx is pressed.

Since our PC has already been incremented by 2 in Cycle(), we can just increment by 2 again to skip the next instruction.

```cpp
void Chip8::OP_Ex9E()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	uint8_t key = registers[Vx];

	if (keypad[key])
	{
		pc += 2;
	}
}
```

---

# `ExA1 - SKNP Vx`

Skip next instruction if key with the value of Vx is not pressed.

Since our PC has already been incremented by 2 in Cycle(), we can just increment by 2 again to skip the next instruction.

```cpp
void Chip8::OP_ExA1()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	uint8_t key = registers[Vx];

	if (!keypad[key])
	{
		pc += 2;
	}
}
```

---

# `Fx07 - LD Vx, DT`

Set Vx = delay timer value.

```cpp
void Chip8::OP_Fx07()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	registers[Vx] = delayTimer;
}

```

---

# `Fx0A - LD Vx, K`

Wait for a key press, store the value of the key in Vx.
The easiest way to “wait” is to decrement the PC by 2 whenever a keypad value is not detected. This has the effect of running the same instruction repeatedly.

The Fx0A opcode in the Chip8 architecture is responsible for waiting for a keypress from the user and storing the value of the key (which is pressed) into a register Vx. The instruction essentially blocks the execution of the program until a key is pressed. Once a key is pressed, the value corresponding to the key (a number from 0 to 15) is stored in the register Vx.

```cpp
void Chip8::OP_Fx0A()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	if (keypad[0])
	{
		registers[Vx] = 0;
	}
	else if (keypad[1])
	{
		registers[Vx] = 1;
	}
	else if (keypad[2])
	{
		registers[Vx] = 2;
	}
	else if (keypad[3])
	{
		registers[Vx] = 3;
	}
	else if (keypad[4])
	{
		registers[Vx] = 4;
	}
	else if (keypad[5])
	{
		registers[Vx] = 5;
	}
	else if (keypad[6])
	{
		registers[Vx] = 6;
	}
	else if (keypad[7])
	{
		registers[Vx] = 7;
	}
	else if (keypad[8])
	{
		registers[Vx] = 8;
	}
	else if (keypad[9])
	{
		registers[Vx] = 9;
	}
	else if (keypad[10])
	{
		registers[Vx] = 10;
	}
	else if (keypad[11])
	{
		registers[Vx] = 11;
	}
	else if (keypad[12])
	{
		registers[Vx] = 12;
	}
	else if (keypad[13])
	{
		registers[Vx] = 13;
	}
	else if (keypad[14])
	{
		registers[Vx] = 14;
	}
	else if (keypad[15])
	{
		registers[Vx] = 15;
	}
	else
	{
		pc -= 2;  //If no key is pressed (i.e., none of the keypad[] values are 1), the program does not continue to the next instruction.
        //Instead, the Program Counter (PC) is decremented by 2, causing the Fx0A instruction to be re-executed in the next cycle. This has the effect of "waiting" until a key is pressed, effectively halting the program at this point until user input is provided.
	}
}

```

## Keypad Layout and Mapping:

```bash
[1] [2] [3] [C]
[4] [5] [6] [D]
[7] [8] [9] [E]
[A] [0] [B] [F]
```

---

# `Fx15 - LD DT, Vx`

Set delay timer = Vx.

```cpp
void Chip8::OP_Fx15()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	delayTimer = registers[Vx];
}
```

---

# `Fx18 - LD ST, Vx`

Set sound timer = Vx.

```cpp
void Chip8::OP_Fx18()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	soundTimer = registers[Vx];
}
```

---

# `Fx1E - ADD I, Vx`

Set I = I + Vx.

```cpp
void Chip8::OP_Fx1E()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	index += registers[Vx];
}
```

---

# `Fx29 - LD F, Vx`

Set I = location of sprite for digit Vx.

We know the font characters are located at 0x50, and we know they’re five bytes each, so we can get the address of the first byte of any character by taking an offset from the start address.

```cpp
void Chip8::OP_Fx29()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t digit = registers[Vx];

	index = FONTSET_START_ADDRESS + (5 * digit);
}
```

Each character sprite is 5 bytes long, so to calculate the address of the sprite for the digit in digit, the formula FONTSET_START_ADDRESS + (5 \* digit) is used. This formula multiplies the digit by 5 (since each sprite is 5 bytes) and adds the base address of the fontset.

---

# `Fx33 - LD B, Vx`

The Fx33 - LD B, Vx instruction in Chip8 stores the BCD (Binary-Coded Decimal) representation of the value in register Vx into three consecutive memory locations starting from the address stored in the I register.

BCD is a system where each decimal digit (0-9) is represented by its binary equivalent. For instance:

The decimal number 345 would be represented as the individual binary equivalents of 3, 4, and 5.
In BCD, each decimal digit is stored separately as a 4-bit binary value.

Store BCD representation of Vx in memory locations I, I+1, and I+2.
The hundreds place (1 digit)
The tens place (1 digit)
The ones place (1 digit)

The interpreter takes the decimal value of Vx, and places the hundreds digit in memory at location in I, the tens digit at location I+1, and the ones digit at location I+2.

We can use the modulus operator to get the right-most digit of a number, and then do a division to remove that digit. A division by ten will either completely remove the digit (340 / 10 = 34), or result in a float which will be truncated (345 / 10 = 34.5 = 34).

```cpp
void Chip8::OP_Fx33()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;
	uint8_t value = registers[Vx];

	// Ones-place
	memory[index + 2] = value % 10;
	value /= 10;

	// Tens-place
	memory[index + 1] = value % 10;
	value /= 10;

	// Hundreds-place
	memory[index] = value % 10;
}
```

Imagine you want to display the number 345 on the screen:

You can't directly display 345 on Chip8 because the graphics system operates with smaller "sprites" (usually 8x8 pixel blocks).

You use the Fx33 instruction to convert 345 into its BCD representation:

Hundreds place: 3
Tens place: 4
Ones place: 5
The number 345 is then broken down into three separate digits (3, 4, and 5), each stored in consecutive memory locations starting at I.

These digits can then be used as individual sprites (or other representations) on the screen.

Breaking Down the Number:
For Vx = 345, the BCD conversion would:

Store the hundreds digit (3) in memory[I]
Store the tens digit (4) in memory[I+1]
Store the ones digit (5) in memory[I+2]
This allows the Chip8 interpreter or emulator to easily display or manipulate individual digits, even though Chip8's limited memory can only store 8-bit values in its registers.

---

# `Fx55 - LD [I], Vx`

Store registers V0 through Vx in memory starting at location I.

The instruction Fx55 - LD [I], Vx in the Chip8 instruction set is used to store the values of registers V0 to Vx into memory, starting from the address stored in the I register.

```cpp
void Chip8::OP_Fx55()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	for (uint8_t i = 0; i <= Vx; ++i)
	{
		memory[index + i] = registers[i];
	}
}
```

The index register determines the starting memory address, and i is the offset into memory.

---

# `Fx65 - LD Vx, [I]`

Read registers V0 through Vx from memory starting at location I.

```cpp
void Chip8::OP_Fx65()
{
	uint8_t Vx = (opcode & 0x0F00u) >> 8u;

	for (uint8_t i = 0; i <= Vx; ++i)
	{
		registers[i] = memory[index + i];
	}
}
```

The Fx65 - LD Vx, [I] instruction in the Chip8 instruction set reads the values stored in memory (starting at the address stored in the I register) and loads them into registers V0 through Vx.

---

<br>
<br>
<br>
<br>
<br>

# **FUNCTION POINTER TABLE**

The easiest way to decode an opcode is with a switch statement, but it gets messy when you have a lot of instructions. The CHIP-8 isn’t so bad, but we’ll use a different technique that is more scalable and good to know when making more advanced emulators.

Instead we’ll implement an array of function pointers where the opcode is an index into an array of function pointers. The downside to a function pointer table is that we must have an array big enough to account for every opcode because the opcode is an index into the array. Dereferencing a pointer for every instruction may also have problems, but when your emulator is complex it’s probably worth it.

If you look at the list of opcodes, you’ll notice that there are four types:

## The entire opcode is unique:

```bash
$1nnn
$2nnn
$3xkk
$4xkk
$5xy0
$6xkk
$7xkk
$9xy0
$Annn
$Bnnn
$Cxkk
$Dxyn
```

## The first digit repeats but the last digit is unique:

```bash
$8xy0
$8xy1
$8xy2
$8xy3
$8xy4
$8xy5
$8xy6
$8xy7
$8xyE
```

## The first three digits are $00E but the fourth digit is unique:

```bash
$00E0
$00EE
```

## The first digit repeats but the last two digits are unique:

```bash
$ExA1
$Ex9E
$Fx07
$Fx0A
$Fx15
$Fx18
$Fx1E
$Fx29
$Fx33
$Fx55
$Fx65
```

---

The first digits go from $0 to $F so we’ll need an array of function pointers that can be indexed up to $F, which requires $F + 1 elements.

For the opcodes with first digits that repeat ($0, $8, $E, $F), we’ll need secondary tables that can accommodate each of those.

$0 needs an array that can index up to $E+1
$8 needs an array that can index up to $E+1
$E needs an array that can index up to $E+1
$F needs an array that can index up to $65+1

In the master table, we set up a function pointer to a function that then indexes correctly based on the relevant parts of the opcode.

Just in case one of the invalid opcodes is called, we can create a dummy OP_NULL function that does nothing, but will be the default function called if a proper function pointer is not set.

```cpp
class Chip8
{
	Chip8()
		: randGen(std::chrono::system_clock::now().time_since_epoch().count())
	{

		// Set up function pointer table
		table[0x0] = &Chip8::Table0;
		table[0x1] = &Chip8::OP_1nnn;
		table[0x2] = &Chip8::OP_2nnn;
		table[0x3] = &Chip8::OP_3xkk;
		table[0x4] = &Chip8::OP_4xkk;
		table[0x5] = &Chip8::OP_5xy0;
		table[0x6] = &Chip8::OP_6xkk;
		table[0x7] = &Chip8::OP_7xkk;
		table[0x8] = &Chip8::Table8;
		table[0x9] = &Chip8::OP_9xy0;
		table[0xA] = &Chip8::OP_Annn;
		table[0xB] = &Chip8::OP_Bnnn;
		table[0xC] = &Chip8::OP_Cxkk;
		table[0xD] = &Chip8::OP_Dxyn;
		table[0xE] = &Chip8::TableE;
		table[0xF] = &Chip8::TableF;

		for (size_t i = 0; i <= 0xE; i++)
		{
			table0[i] = &Chip8::OP_NULL;
			table8[i] = &Chip8::OP_NULL;
			tableE[i] = &Chip8::OP_NULL;
		}

		table0[0x0] = &Chip8::OP_00E0;
		table0[0xE] = &Chip8::OP_00EE;

		table8[0x0] = &Chip8::OP_8xy0;
		table8[0x1] = &Chip8::OP_8xy1;
		table8[0x2] = &Chip8::OP_8xy2;
		table8[0x3] = &Chip8::OP_8xy3;
		table8[0x4] = &Chip8::OP_8xy4;
		table8[0x5] = &Chip8::OP_8xy5;
		table8[0x6] = &Chip8::OP_8xy6;
		table8[0x7] = &Chip8::OP_8xy7;
		table8[0xE] = &Chip8::OP_8xyE;

		tableE[0x1] = &Chip8::OP_ExA1;
		tableE[0xE] = &Chip8::OP_Ex9E;

		for (size_t i = 0; i <= 0x65; i++)
		{
			tableF[i] = &Chip8::OP_NULL;
		}

		tableF[0x07] = &Chip8::OP_Fx07;
		tableF[0x0A] = &Chip8::OP_Fx0A;
		tableF[0x15] = &Chip8::OP_Fx15;
		tableF[0x18] = &Chip8::OP_Fx18;
		tableF[0x1E] = &Chip8::OP_Fx1E;
		tableF[0x29] = &Chip8::OP_Fx29;
		tableF[0x33] = &Chip8::OP_Fx33;
		tableF[0x55] = &Chip8::OP_Fx55;
		tableF[0x65] = &Chip8::OP_Fx65;
	}

// These functions (Table0, Table8, TableE, TableF) are used to handle specific opcode categories in the Chip-8 emulator. The general purpose is to decode and execute the opcode instructions by invoking the appropriate functions that are mapped in the function pointer tables (table0, table8, tableE, tableF).


	void Table0()
	{
		((*this).*(table0[opcode & 0x000Fu]))();
	}

	void Table8()
	{
		((*this).*(table8[opcode & 0x000Fu]))();
	}

	void TableE()
	{
		((*this).*(tableE[opcode & 0x000Fu]))();
	}

	void TableF()
	{
		((*this).*(tableF[opcode & 0x00FFu]))();
	}

	void OP_NULL()
	{}

	typedef void (Chip8::*Chip8Func)();
	Chip8Func table[0xF + 1];
	Chip8Func table0[0xE + 1];
	Chip8Func table8[0xE + 1];
	Chip8Func tableE[0xE + 1];
	Chip8Func tableF[0x65 + 1];
}
```

- For larger emulators:

The function pointer table approach is a great choice because it offers scalability, performance optimization, and modular architecture.

- For smaller systems like CHIP-8:

It could be an over-engineered solution, where simpler alternatives (like a switch statement or a direct function call for each opcode) may save memory and complexity.

The main issue with the function pointer table approach is that it allocates space for all possible opcode values, leading to wasted memory for unused opcodes.

Memory Wastage in tableF:

For the 0xF opcodes, there are only 9 opcodes defined:

```bash
0xF07 = OP_Fx07
0xF0A = OP_Fx0A
0xF15 = OP_Fx15
0xF18 = OP_Fx18
0xF1E = OP_Fx1E
0xF29 = OP_Fx29
0xF33 = OP_Fx33
0xF55 = OP_Fx55
0xF65 = OP_Fx65
```

This means that out of the 1024 possible entries (for the range 0x00-0xFF), only 9 are actually used, which leads to a lot of wasted memory (1015 entries that aren't used).

but because for complex emulator this is the best approch so we will stick to it

---



# Fetch, Decode, Execute

When we talk about one cycle of this primitive CPU that we’re emulating, we’re talking about it doing three things:

Fetch the next instruction in the form of an opcode
Decode the instruction to determine what operation needs to occur
Execute the instruction
The decoding and executing are done with the function pointers we just implemented. We get the first digit of the opcode with a bitmask, shift it over so that it becomes a single digit from $0 to $F, and use that as index into the function pointer array. It’s then further decoded in the Table() method.

```cpp
void Chip8::Cycle()
{
	// Fetch
	opcode = (memory[pc] << 8u) | memory[pc + 1];

	// Increment the PC before we execute anything
	pc += 2;

	// Decode and Execute
	((*this).*(table[(opcode & 0xF000u) >> 12u]))();

	// Decrement the delay timer if it's been set
	if (delayTimer > 0)
	{
		--delayTimer;
	}

	// Decrement the sound timer if it's been set
	if (soundTimer > 0)
	{
		--soundTimer;
	}
}
```


