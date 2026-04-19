# Tetris Roadmap: Zero to Running

## Target

Get the Game Boy game `Tetris (USA) (SGB Enhanced).gb` running on the emulator.

## Source ROM

`Tetris (USA) (SGB Enhanced).gb` — MBC5+RAM+BATTERY, 1 MiB ROM, 8 KiB SRAM.
This is the SGB-enhanced version; the base game works on DMG too.

## Scope: What Tetris Needs

Tetris uses:
- **CPU** (full SM83 instruction set)
- **MBC5** cartridge banking (not MBC1)
- **PPU** — background tilemap rendering, no sprites needed (Tetris uses the background layer only)
- **VBlank interrupt** — frame timing
- **Timer** — game clock / lock delay
- **Joypad** — D-pad + A/B buttons
- **Cartridge header** — ROM size, MBC type, checksum

Tetris does NOT use:
- Audio (channels 1-4)
- Serial / link cable
- Objects/sprites (Tetris is all background tilemap)
- CGB-specific features (SGB enhancement is a bonus, not required)

## Implementation Phases

### Phase 0: CPU Core

Implement the SM83 instruction set (~150+ instructions).

**Key facts from PanDocs:**
- 8 registers: A, F (flags), B, C, D, E, H, L
- 16-bit registers: SP, PC
- Flags: Z (zero), N (subtraction), H (half-carry), C (carry)
- Variable-length instructions (1-3 bytes)
- Opcodes grouped into 4 blocks (top 2 bits), plus $CB-prefixed block
- Invalid opcodes ($D3, $DB, $DD, $E3, $E4, $EB, $EC, $ED, $F4, $FC, $FD) hard-lock the CPU

**Milestones:**
1. CPU struct with registers, flags, fetch/decode/execute loop
2. Implement all Block 0 instructions (NOP, LD r16/imm16, LD [r16mem], A, INC/DEC r16/r8, LD r8/imm8, RLCA/RRCA/RLA/RRa, DAA, CPL, SCF, CCF, JR, JR cond, STOP)
3. Implement Block 1 (8-bit register-to-register loads, HALT)
4. Implement Block 2 (8-bit arithmetic: ADD, ADC, SUB, SBC, AND, XOR, OR, CP)
5. Implement Block 3 (ADD/ADC/SUB/SBC/AND/XOR/OR/CP with imm8, conditional/unconditional RET, RETI, JP cond/imm16, JP HL, CALL cond/imm16, RST, POP/PUSH r16stk, prefix byte)
6. Implement $CB-prefixed block (RLC/RL/SLA/SRA/SWAP/SRL r8, BIT/RES/SET b3, r8)
7. Implement LDH instructions (ldh [c],a / ldh [imm8],a / ld a,[c] / ldh a,[imm8])
8. Implement special instructions (DI, EI, LD HL,SP+imm8, ADD SP,imm8, LD SP,HL)
9. Verify with a self-test ROM that exercises all instruction families

### Phase 1: Memory & Cartridge Loading

**Key facts from PanDocs memory map:**
- 0000-3FFF: 16 KiB ROM bank 00 (fixed)
- 4000-7FFF: 16 KiB ROM bank 01-N (switchable via MBC)
- 8000-9FFF: 8 KiB VRAM
- A000-BFFF: 8 KiB external RAM (switchable via MBC)
- C000-CFFF: 4 KiB WRAM
- D000-DFFF: 4 KiB WRAM (bank-switchable on CGB)
- E000-FDFF: Echo RAM (mirror of C000-DDFF)
- FE00-FE9F: OAM
- FEA0-FEFF: Not usable
- FF00-FF7F: I/O registers
- FF80-FFFE: HRAM
- FFFF: IE (interrupt enable)

**Milestones:**
1. `Memory` struct implementing the 64KB address space with read/write
2. ROM file loader — read .gb/.gbc file into memory
3. Cartridge header parser (bytes $0100-$014F) — extract cartridge type, ROM size, RAM size, title, checksum
4. Header checksum validation (subtraction checksum at $014D, global sum at $014E-$014F)
5. Entry point at $0100 — jump to $0150 (where most games' actual code lives)

### Phase 2: MBC1 Memory Bank Controller

**Key facts:**
- MBC1 controls ROM banking (banks 1-N in 4000-7FFF) and RAM banking (A000-BFFF)
- Two separate bank select registers: one for ROM (0400-7FFF), one for RAM (A000-BFFF)
- ROM banking: bits 0-4 of select register → bank number (0 = bank 0, must not be used)
- RAM banking: bit 0 of RAM select register → RAM bank number
- Memory mode select (bits 3-4 of ROM select register at 2000-3FFF):
  - 00 = 2-bank mode (banks 0-3)
  - 01 = 4-bank mode (banks 0-3)
  - 1x = 16-bank mode (banks 0-15)
- Writing to 2000-3FFF selects ROM bank; writing to A000-BFFF selects RAM bank

**Milestones:**
1. MBC1 struct implementing bank switching logic
2. ROM bank switching in 4000-7FFF range
3. RAM bank switching in A000-BFFF range
4. Memory mode select (2-bank / 4-bank / 16-bank)
5. Verify with a ROM that uses MBC1

### Phase 3: PPU — Background Rendering

The PPU is the most complex subsystem. Tetris only needs the background layer.

**Key facts from PanDocs rendering:**
- 154 scanlines per frame (144 visible + 10 VBlank)
- 4 PPU modes: Mode 2 (search OAM), Mode 3 (send pixels to LCD), Mode 0 (end of scanline), Mode 1 (VBlank)
- Mode 3: 172-289+ dots (varies based on scrolling, window, OBJ penalties)
- 1 dot = 1/2^22 seconds (~238ns) at 4.194304 MHz
- Background: 32x32 tile map, 8x8 pixel tiles, 2bpp
- VRAM layout: 384 tiles (16 bytes each) + 2 tile maps (1024 bytes each) per bank
- Registers: LCDC ($FF40), SCY ($FF42), SCX ($FF43), WX ($FF4A), WY ($FF4B)
- LCD control bits: BG Window Enable (bit 0), Window/Tile Select (bit 1), Window Enable (bit 2), BG Enable (bit 3), OBJ Enable (bit 4), OBJ Size (bit 5), BG Window Tile Data Select (bit 6), LCD Enable (bit 7)

**Milestones:**
1. PPU struct with mode machine (modes 0-1-2-3-0-1 cycle)
2. OAM reading (FE00-FE9F) — 40 entries of 4 bytes each (Y, X, tile, attributes)
3. Background tile map rendering (scrolling, window)
4. Tile data fetching from VRAM (2bpp → 4 color indices)
5. Palette application (BGP register $FF47)
6. VBlank interrupt at end of scanline 144
7. LCD interrupt via STAT register ($FF41)
8. Output a 160x144 frame to a display buffer
9. Verify with a simple ROM that draws a background

### Phase 4: Interrupts & Timer

**Key facts from PanDocs:**
- IE register ($FFFF): VBlank(0), LCD(1), Timer(2), Serial(3), Joypad(4)
- IF register ($FF0F): same bits, set when interrupt requested
- IME flag: internal CPU flag, set by EI, cleared by DI and interrupt entry
- Interrupt priorities: VBlank > LCD > Timer > Serial > Joypad
- ISR takes 5 M-cycles (2 wait + 2 push PC + 1 jump)
- RST vectors: 0000, 0008, 0010, 0018, 0020, 0028, 0030, 0038
- Interrupt service addresses: 0040, 0048, 0050, 0058, 0060

**Timer:**
- DIV ($FF04): Divider register, increments at 16384 Hz, write any value resets to $00
- TIMA ($FF05): Timer, increments at frequency selected by TAC ($FF07)
- TAC ($FF07): Timer allow — select clock source (4 kHz, 262144 Hz, 65536 Hz, 16384 Hz)
- TMA/TIMA overflow triggers Timer interrupt

**Milestones:**
1. Interrupt controller (IME, IE, IF registers)
2. VBlank interrupt handling (at end of scanline 144)
3. LCD/STAT interrupt (triggered on mode changes)
4. Timer with DIV, TIMA, TAC registers
5. Timer overflow interrupt
6. Joypad interrupt (bit transition on P1 register)
7. RST instruction support (8 jump vectors)
8. Verify with a ROM that uses VBlank and Timer interrupts

### Phase 5: Joypad Input

**Key facts:**
- P1/JOYP register ($FF00): bit 4 selects d-pad, bit 5 selects action buttons
- When selected, read bits 0-3 (pressed = 0, released = 1)
- Most programs read this register multiple times (debounce delay)

**Milestones:**
1. Joypad register implementation ($FF00)
2. Keyboard input mapping (arrow keys → d-pad, Z/X → A/B, Enter → Start, Shift → Select)
3. Joypad interrupt on key press/release

### Phase 6: Display & Game Loop

**Milestones:**
1. Integrate with a rendering backend (winit + wgpu, or sdl2)
2. 160x144 display buffer → window
3. Main emulation loop at ~60 Hz (actual: 59.73 Hz)
4. Cycle-accurate timing: 4.194304 MHz master clock = 1,048,576 cycles/sec
5. Frame timing: ~69891 cycles per frame (154 scanlines × 456 dots × 4 cycles/dot)
6. Pause/resume functionality
7. ROM file argument loading

### Phase 7: Tetris-Specific Verification

1. Load `Tetris (USA) (SGB Enhanced).gb`
2. Verify Nintendo logo appears on boot
3. Verify title screen renders correctly
4. Verify D-pad moves pieces, A/B places/rotates, Start pauses
5. Verify game loop runs (pieces fall, lines clear, score updates)
6. Compare output screenshots with reference emulators (mGBA, BGB)

## Optional / Later Phases

- **MBC5** — Tetris uses MBC5, not MBC1. Implement after MBC1 is verified.
- **Audio** — 4 channels (CH1 square, CH2 square, CH3 wave, CH4 noise)
- **SRAM save/load** — Battery-backed RAM for save states
- **Save states** — Full state snapshot and restore
- **Other MBCs** — MBC2, MBC3, MBC5, HuC1
- **Serial/link cable** — Not needed for single-player Tetris
- **CGB mode** — Color support, double-speed CPU
- **SGB commands** — Super Game Boy enhancements (border, tile remap)
- **Boot ROM** — Nintendo boot ROM (optional, can skip by patching ROMs)
- **Debugging tools** — CPU trace, memory watch, register display

## Reference Documents

All authoritative references are in `ref-docs/pandocs/src/`:
- `CPU_Instruction_Set.md` — Full instruction set with opcode encoding
- `CPU_Registers_and_Flags.md` — Register layout and flag behavior
- `Memory_Map.md` — Complete 64KB address space layout
- `Graphics.md` — Tiles, palettes, layers overview
- `Rendering.md` — PPU mode machine, scanline timing, dot timing
- `Interrupts.md` — IE/IF registers, IME, priorities, ISR timing
- `Audio.md` — APU architecture, 4 channels
- `Joypad_Input.md` — P1/JOYP register, button matrix
- `The_Cartridge_Header.md` — ROM header layout, checksums
- `MBCs.md` — Memory bank controllers overview
- `MBC1.md` — MBC1 specifics
- `MBC5.md` — MBC5 specifics (Tetris uses this)
