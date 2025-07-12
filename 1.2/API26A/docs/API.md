---

# KVmos API26A Documentation

---

## Overview

**KVmos** is a virtual machine operating system designed to run on embedded microcontrollers such as ATMEGA2560 and ATMEGA328P, simulating a Zinc Virtual Machine (ZVM) with a Kernel (KKN). It features a custom 16-bit CPU architecture with 6 registers, a segmented memory space, and supports multitasking via system calls.

---

## System Configuration

| Macro         | Description                         |
| ------------- | ----------------------------------- |
| `DEBUG_BUILD` | Enables debug mode (verbose output) |
| `ATMEGA2560`  | Target MCU with 6 KB RAM (default)  |
| `ATMEGA328P`  | Target MCU with 1 KB RAM            |

---

## Memory and Registers

* **Memory:**

  * `MEMORY[MAX_MEM]`
  * `MAX_MEM = 6*1024` for ATMEGA2560, or 1024 for ATMEGA328P
  * Memory is byte-addressable

* **Registers:**

  * `REGISTER[MAX_REG]` where `MAX_REG = 6`
  * General Purpose Registers: `GPR0` to `GPR3`
  * Special Registers:

    * `PC` — Program Counter
    * `SP` — Stack Pointer

| Register | Index |
| -------- | ----- |
| GPR0     | 0     |
| GPR1     | 1     |
| GPR2     | 2     |
| GPR3     | 3     |
| PC       | 4     |
| SP       | 5     |

* **Flags (`FL`):** 8-bit flag register with bits used as follows:

| Bit | Flag         | Description                           |
| --- | ------------ | ------------------------------------- |
| 0   | Z (Zero)     | Set if compare result is zero         |
| 1   | N (Negative) | Set if compare result is negative     |
| 5   | SEGFAULT     | Set if memory access violation occurs |
| 6   | RUNNING      | VM running flag                       |
| 7   | EXECUTE      | VM executing flag                     |

---

## Instructions (Opcodes)

The ZVM CPU supports 16 basic instructions, encoded in the high nibble (4 bits) of the instruction byte.

| Opcode | Mnemonic | Description                                             |
| ------ | -------- | ------------------------------------------------------- |
| 0x0    | `MVI`    | Move immediate value to register (8 or 16-bit variants) |
| 0x1    | `MOV`    | Move value from one register to another                 |
| 0x2    | `ADD`    | Add one register to another                             |
| 0x3    | `SUB`    | Subtract one register from another                      |
| 0x4    | `AND`    | Bitwise AND between registers                           |
| 0x5    | `OR`     | Bitwise OR between registers                            |
| 0x6    | `XOR`    | Bitwise XOR between registers                           |
| 0x7    | `CMP`    | Compare two registers and set flags Z and N             |
| 0x8    | `LD`     | Load from memory to register                            |
| 0x9    | `ST`     | Store from register to memory                           |
| 0xA    | `JMP`    | Jump (unconditional, relative, or indirect)             |
| 0xB    | `BRN`    | Branch on flag (Zero or Negative)                       |
| 0xC    | `SHL`    | Shift left register by value                            |
| 0xD    | `SHR`    | Shift right register by value                           |
| 0xE    | `SCB`    | Set or clear a bit in a register                        |
| 0xF    | `SYS`    | System call (invoke kernel syscall)                     |

---

## Instruction Details

### MVI (Move Immediate)

* Load immediate 8 or 16-bit value into a register.
* Instruction format varies based on bits set in instruction byte.

### MOV

* Copy value from source register to destination register.

### ADD / SUB / AND / OR / XOR

* Arithmetic and bitwise operations between two registers, storing result in destination.

### CMP

* Compare two registers, set `Z` (zero) if equal, `N` (negative) if first < second.

### LD / ST

* Load from or store to memory with addressing modes (direct, indirect, or stack-relative).
* Memory bounds checked to avoid segmentation faults.

### JMP / BRN

* Jump instructions supporting absolute, relative, and conditional branches based on flags.

### SHL / SHR

* Bitwise shifts on registers.

### SCB

* Toggle bit in a register based on value in another register.

### SYS (System Calls)

* Trigger kernel functions using the lower 4 bits as syscall index.

---

## System Calls API

System calls are invoked via the `SYS` opcode. They receive a 16-bit argument pointer (`argptr`) which indexes into memory.

| Syscall Index | Function          | Description                                  |
| ------------- | ----------------- | -------------------------------------------- |
| 0             | `SysWriteIO`      | Write a byte to IO address                   |
| 1             | `SysReadIO`       | Read a byte from IO address into register 3  |
| 2             | `SysScheduleTask` | Schedule a new task                          |
| 3             | `SysKillTask`     | Kill a specified task                        |
| 4             | `SysMalloc`       | Allocate memory (stub)                       |
| 5             | `SysFree`         | Free memory (stub)                           |
| 6             | `SysGetPC`        | Get current PC into register 3               |
| 7             | `SysHalt`         | Halt VM execution                            |
| 8             | `SysKill`         | Kill VM process                              |
| 9             | `SysOpen`         | Open a file by name, returning file handle   |
| 10            | `SysRead`         | Read byte from memory address                |
| 11            | `SysExec`         | Execute program at memory address (schedule) |

---

## System Call Usage

### SysWriteIO

```cpp
void SysWriteIO(uint16_t argptr);
// Writes MEMORY[argptr+2] byte to IO address stored at MEMORY[argptr] and MEMORY[argptr+1].
```

### SysReadIO

```cpp
void SysReadIO(uint16_t argptr);
// Reads byte from IO address stored at MEMORY[argptr] and MEMORY[argptr+1] into REGISTER[3].
```

### SysScheduleTask

```cpp
void SysScheduleTask(uint16_t argptr);
// Attempts to schedule task located at MEMORY[argptr], returns task ID in REGISTER[3] or 0xFF if full.
```

### SysKillTask

```cpp
void SysKillTask(uint16_t argptr);
// Kills the task specified by MEMORY[argptr].
```

### SysMalloc / SysFree

* Currently stubs; memory allocation not implemented.

### SysGetPC

```cpp
void SysGetPC(uint16_t argptr);
// Copies current PC to REGISTER[3].
```

### SysHalt

```cpp
void SysHalt(uint16_t argptr);
// Clears VM running flag, halting execution.
```

### SysKill

```cpp
void SysKill(uint16_t argptr);
// Clears VM execution flag, effectively killing VM.
```

### SysOpen

```cpp
void SysOpen(uint16_t argptr);
// Opens file by name (null-terminated string in MEMORY[argptr]), returns handle or 0xFF if not found.
```

### SysRead

```cpp
void SysRead(uint16_t argptr);
// Reads byte from address MEMORY[argptr..argptr+1]+MEMORY[argptr+2] into REGISTER[3].
```

### SysExec

```cpp
void SysExec(uint16_t argptr);
// Schedules and executes program starting at address MEMORY[argptr..argptr+1].
```

---

## Multitasking

* KVmos supports up to 16 tasks (indexed 0 to 15).
* Each task has its own register set and stack saved in memory.
* `storeTask()` and `loadTask()` save/load the CPU state to/from memory.
* Task switching occurs every `TIME_SLICE` milliseconds (default 10 ms).
* The scheduler switches the current task cyclically.

---

## VM Lifecycle Functions

### VMsetup()

Sets running and execute flags to start VM:

```cpp
#define VMsetup() (FL |= 0b11000000)
```

### VMloop()

Runs the virtual machine main loop, decoding and executing instructions, and handling task switches.

---

## Debug Mode

* Enabled with `DEBUG_BUILD`.
* Prints registers, flags, PC, and memory dumps over serial.
* Supports step-by-step execution controlled via `allowStep`.

---

## Example Usage

```cpp
void setup() {
  MEMORY[0xF0] = 0x10;           // Load program start
  SysScheduleTask(0xF0);         // Schedule task at 0xF0
  MEMORY[0xF0] = 0x00;
  VMsetup();                    // Start VM
}

void loop() {
  VMloop();                    // Run VM
```


instructions
}

```

---

## Summary

KVmos API26A exposes a simple but flexible 16-bit CPU architecture and system call interface for embedded virtual machines. It allows running and multitasking small programs on microcontrollers via a virtual machine abstraction.

---
