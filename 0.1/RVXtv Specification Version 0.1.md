# RVXtv Specification Version 0.1
Copyright © 2026 RVXtv

Licensed under the Creative Commons Attribution International License (CC BY 4.0)

## 0. Version 0.1 Note

### 0.1. Breaking changes

The goal of this version of this specification is to function as a base to build the rest of the RVXtv extension. Breaking changes should be expected.

## 1. Introduction

### 1.1. Purpose

This document outlines the specification of the RISC-V Ternary Vector extension (RVXtv). This specification is meant to define an ISA for ternary vector arithmetic that is compatible with RISC-V. 

RVXtv can be used to accelerate neural networks, high-precision ternary arithmetic, and any applications that may require ternary computation, but are hosted on binary hardware (RISC-V).

It is important to note that this extension is unofficial and is not created or endorsed by RISC-V.

### 1.2. Goals

The primary goals for the creation of RVXtv are as follows:

1. Creating a standardized, practical ISA for ternary vector operations in RISC-V

2. Creating a computation standard that is capable of computing ternary vectors faster than virtual binary calculation methods

The following are not goals for the RVXtv project:

1. Replacing ternary-native computing architectures (architectures capable of representing ternary values physically)

2. Being ratified as an official RISC-V extension

### 1.3. Non-ratified stance

The **X** in **RVXtv** indicates that this extension is non-standard and should not be interpreted as a ratified RISC-V standard or as a standard that is in progress to be ratified. This extension is designed to function as a stand-alone specification that is compatible with the RISC-V Integer ISA.

This is because RISC-V is a binary architecture, and RVXtv serves as a bridge between ternary and binary. Using this standard may bottleneck the performance and throughput that would otherwise be gained from utilising a custom architecture and hardware. For theoretical future production use, using entirely ternary-native hardware instead of building on top of RISC-V is suggested. 

That said, RVXtv is intended to be significantly faster than the software emulation of ternary computation in conventional binary RISC-V, and, in lieu of practical ternary-native hardware, may be used to implement production systems that require ternary computation\*.

\* This statement is based on one of the primary stated goals of RVXtv and should not be assumed until Version 1.0

## 2. Memory

### 2.1. Primitives
The primitive of the RVXtv extension is the ternary digit, or "trit". The ternary digit has 3 possible values instead of the binary 2. Because RISC-V is binary, the ternary digits must be expressed in binary.

RVXtv uses one primary mapping* for ternary digits (2 bits encoding 1 trit):

```
00 -> 0
01 -> -1
10 -> 1
11 -> 0
```
This encoding is used so that a binary NOT applied to a ternary value would return a correct ternary NOT. (1->-1, -1->1, and 0->0).

`11` as a representation for 0 should only be used in the context of input, no function should return `11` for zero. `00` is preferred.

\* The current mapping is "balanced", `{-1, 0, 1}` but there are other mappings (`{0, 1, 2}`, `{0, 1/2, 1}`). Notice that this encoding loses one combination per trit, and can be addressed using compression or ternary-native systems.

### 2.2. Registers

RVXtv uses `32` `162 bit` registers* each holding 9 `9-trit` integers (`168 bits` in memory, for byte addressability with 6 extra reserved bits), labeled `tv0-tv31`, encoded instructionally using `5 unsigned bits` (2^5 = 32). Each trit is `2 bits` thus the overall capacity of any given register is 9 elements of 9 trits giving `9*9*2 = 162 bits` per register.

For example: the first element of `tv0`, a `162 bit` value:

```
element 0: tv0[17:0]

    trit 0: tv0[1:0]
    trit 1: tv0[3:2]
    trit 2: tv0[5:4]
    trit 3: tv0[7:6]
    trit 4: tv0[9:8]
    trit 5: tv0[11:10]
    trit 6: tv0[13:12]
    trit 7: tv0[15:14]
    trit 8: tv0[17:16]

element 1: tv0[35:18]

element 2: tv0[53:36]

element 3: tv0[71:54]

element 4: tv0[89:72]

element 5: tv0[107:90]

element 6: tv0[125:108]

element 7: tv0[143:126]

element 8: tv0[161:144]
```

```
Memory:

    Functional Data: tv0[161:0]
    Reserved Space: tv0[167:162]
```

RVXtv also makes use of 32 `18 bit` scalar registers (`24 bits` for byte addressability, 6 extra reserved bits), labeled `tx0-tx31`, they each encode one `9 trit` integer.

Each trit can either be `{-1, 0, 1}`. Each integer in each vector can represent any integer value from -9841 to 9841.

\* RISC-V "V" Vector has support for multiple register lengths, a similar `VLEN` feature may be implemented in the future. 

## 3. Instructions 

### 3.1. Instruction Template

An implementation can use any of the custom reserved opcodes (`CUSTOM-0`, `CUSTOM-1`, `CUSTOM-2`, `CUSTOM-3`) for compatibility, the opcode itself does not encode any information so it is interchangeable. This specification uses `CUSTOM-1` or `0101011`.

Arithmetic instruction template for RVXtv:


| 31-28    | 27-23           | 22-18              | 17-13            | 12-7        | 6-0                | 
| -------- | --------------- | ------------------ | ---------------- | ----------- | ------------------ |
| Reserved | Output Register | Secondary Register | Primary Register | Instruction | Opcode (`0101011`) |

Memory instruction template for RVXtv:


| 31-23    | 22-18           | 17-13            | 12-7        | 6-0                | 
| -------- | --------------- | ---------------- | ----------- | ------------------ |
| Reserved | Output Register | Primary Register | Instruction | Opcode (`0101011`) |

\* \*\* Likely to be changed in Version 1.0.

#### 3.2. Definitions

| Variable  | Value                              | Example |
| --------- | ---------------------------------- | ------- |
| tvd       | Destination Vector Register        | tv8     |
| tvs1      | Primary Vector                     | tv8     |
| tvs2      | Secondary Vector                   | tv8     |
| txs1      | Primary Ternary Scalar Register    | tx8     |
| tv0-tv31  | Vector Registers                   | tv8     |
| tx0-tx31  | Scalar Registers                   | tx8     |
| rs1       | Primary Scalar Register            | x8      |
| rs2       | Secondary Scalar Register          | x9      |
| tvval     | Placeholder for a Vector Element   |         |

`||` is concatenation (e.g. `10 || 01 = 1001`)

For every operation, the secondary vector is being applied to the primary vector, for example in subtraction: `tvs1 - tvs2`, and because subtraction is not commutative, the vectors are evaluated in the order provided.

Every ternary number is encoded using trits:

```
decimal value of a trit number = sum(trit_i * 3^i for i in integer)

For example: 

01 01 01 01 01 01 01 01 01 = (-1)(3^0) + (-1)(3^1) + (-1)(3^2) + (-1)(3^3) + (-1)(3^4) +(-1)(3^5) + (-1)(3^6) + (-1)(3^7) + (-1)(3^8) = -9841
```


### 3.3. Memory instructions

#### 3.3.0. Endianness

The endianness of RVXtv is little-endian.

If 2 bytes of data are to be loaded from address `0x1`, the data is to be loaded from the smallest to the largest, with the first byte, as the least significant, for example:

```
0x1: 0x01 (00000001)
0x2: 0x02 (00000010)
```

The value would be:

```
0x0201 (0000001000000001)
```

#### 3.3.1. Vector Load

Instruction: `tvl`, `000001`, `1`.

```
tvl tvd, rs1
```

```
Example: 
tvl tv1, x1
000001 00001, 00001

Whole instruction: 
0101011 000001 00001 00001 000000000
```

Addressing uses RV32I/RV64I scalar values.

Implementation (RV32):

```
tvd = MEM[rs1:rs1+20];
```

Implementation (RV64):

```
tvd = MEM[rs1:rs1+20];
```

`MEM[address]` is byte-addressed. That is to say, each address points to an individual byte.

Memory loading includes the byte at rs1 and continues 20 bytes forward,  the final 6 bits are reserved and are not part of the functional data (21 bytes total).

Example (RV64):
```
x1 = 0000000000000000000000000000000100000000000000000000000000000001

Memory position at (hexadecimal: 0000000100000001): 1111...1111 (162 bits) || 000000 (for byte-addressability, these bits are discarded)

> tvl 00001(tv1) 00001(x1) 0 

tvd = [0, 0, 0, 0, 0, 0, 0, 0, 0]
or [111111111111111111] * 9
```

#### 3.3.2. Vector Store

Instruction: `tvs`, `000010`, `2`.

```
tvs tvs1, rs1
```

```
Example: 
tvs tv1, x1
000010 00001, 00001

Whole instruction: 
0101011 000010 00001 00001 000000000
```

Addressing uses RV32I/RV64I scalar values.

Memory storage includes the byte at rs1 and continues 20 bytes forward,  the final 6 bits are reserved and are not part of the functional data (21 bytes total).

Implementation (RV32):

```
MEM[rs1:rs1+20] = tvs1;
```

Implementation (RV64):

```
MEM[rs1:rs1+20] = tvs1;
```

`MEM[address]` is byte-addressed. 

Example (RV64):
```
x1 = 0000000000000000000000000000000100000000000000000000000000000001

Value of tvd: 1111...1111 (162 bits)

> tvs 00001(tv1) 00001(x1) 0 

MEM[0x0000000100000015:0x0000000100000001] = 1111...1111 (162 bits)  || 000000 (for byte-addressability, these bits are discarded)

(0x0000000100000015 is 0x0000000100000001 + 20)
```

### 3.4. Integer Instructions

#### 3.4.1. Vector Addition

Instruction: `tvadd.vv`, `000011`, `3`.

```
tvadd.vv tvd, tvs1, tvs2

Example: 
tvadd.vv tv1, tv2, tv3
000011 00001, 00010, 00011

whole instruction
0101011 000011 00001 00010 00011 0000
```

Implementation:

```
tvadd.vv: reg[tvd] = reg[tvs1] + reg[tvs2] (elementwise)
```

Overflow is wrapped, thus every result is `mod 3^9`.

For example, with single-number vectors:

```
tv2 = 10 10 10 10 10 10 10 10 10 (9841)
tv3 = 00 00 00 00 00 00 00 00 10 (1)

> tvadd.vv tv1, tv2, tv3

tv1 = 01 01 01 01 01 01 01 01 01 (9842, calculated to modulo 3^9, giving -9841)
```

#### 3.4.2. Scalar Addition

Instruction: `tvadd.vs`, `000100`, `4`.

```
tvadd.vs tvd, tvs1, txs1

Example: 
tvadd.vs tv1, tv2, tx1
000100 00001, 00010, 00001

whole instruction
0101011 000100 00001 00010 00001 0000
```

Implementation:

```
tvadd.vs: reg[tvd] = reg[tvval] + reg[tx1] for tvval in reg[tvs1] 
```

Overflow is wrapped, thus every result is `mod 3^9`, because 9 trits can represent any integer value from -9841 to 9841.

For example, with a single-number vector and a scalar:

```
tv2 = 10 10 10 10 10 10 10 10 10 (9841)
tx1 = 00 00 00 00 00 00 00 00 10 (1)

> tvadd.vs tv1, tv2, tx1

tv1 = 01 01 01 01 01 01 01 01 01 (9842, calculated to modulo 3^9, giving -9841)
```

#### 3.4.3. Vector Subtraction

Instruction: `tvsub.vv`, `000101`, `5`.

```
tvsub.vv tvd, tvs1, tvs2

Example: 
tvsub.vv tv1, tv2, tv3
000101 00001, 00010, 00011

whole instruction
0101011 000101 00001 00010 00011 0000
```

Implementation:

```
tvsub.vv: reg[tvd] = reg[tvs1] - reg[tvs2] (elementwise)
```

Like addition, underflow is wrapped. Making every result `mod 3^9`.

#### 3.4.4. Scalar Subtraction

Instruction: `tvsub.vs`, `000110`, `6`.

```
tvsub.vs tvd, tvs1, txs1

Example: 
tvsub.vs tv1, tv2, tx1
000110 00001, 00010, 00001

whole instruction
0101011 000110 00001 00010 00001 0000
```

Implementation:

```
tvsub.vs: reg[tvd] = reg[tvval] - reg[tx1] for tvval in reg[tvs1] 
```

Like addition, underflow is wrapped. Making every result `mod 3^9`.

#### 3.4.5. Vector Multiplication

Instruction: `tvmul.vv`, `000111`, `7`.

```
tvmul.vv tvd, tvs1, tvs2

Example: 
tvmul.vv tv1, tv2, tv3
000111 00001, 00010, 00011

whole instruction
0101011 000111 00001 00010 00011 0000
```

Implementation:

```
tvmul.vv: reg[tvd] = reg[tvs1] * reg[tvs2] (elementwise)
```

As with addition, overflow is wrapped (`mod 3^9`).

#### 3.4.6. Scalar Multiplication

Instruction: `tvmul.vs`, `001000`, `8`.

```
tvmul.vs tvd, tvs1, txs1

Example: 
tvmul.vs tv1, tv2, tx1
001000 00001, 00010, 00001

whole instruction
0101011 001000 00001 00010 00001 0000
```

Implementation:

```
tvmul.vs: reg[tvd] = reg[tvval] * reg[tx1] for tvval in reg[tvs1] 
```

As with addition, overflow is wrapped (`mod 3^9`).

#### 3.4.7. Vector Division

Instruction: `tvdiv.vv`, `001001`, `9`.

```
tvdiv.vv tvd, tvs1, tvs2

Example: 
tvdiv.vv tv1, tv2, tv3
001001 00001, 00010, 00011

whole instruction
0101011 001001 00001 00010 00011 0000
```

Implementation:

```
tvdiv.vv: reg[tvd] = floor(reg[tvs1] / reg[tvs2]) (elementwise)
```

For division by zero, the maximum value is given.

```
tv2 = 00 00 00 00 00 00 00 00 10 (1)
tv3 = 00 00 00 00 00 00 00 00 00 (0)

> tvdiv.vv tv1, tv2, tv3

tv1 = 10 10 10 10 10 10 10 10 10 (9841)
```

#### 3.4.8. Scalar Division

Instruction: `tvdiv.vs`, `001010`, `10`.

```
tvdiv.vs tvd, tvs1, txs1

Example: 
tvdiv.vs tv1, tv2, tx1
001010 00001, 00010, 00001

whole instruction
0101011 001010 00001 00010 00001 0000
```

Implementation:

```
tvdiv.vs: reg[tvd] = floor(reg[tvval] / reg[tx1]) for tvval in reg[tvs1] 
```

Just like addition, overflow is wrapped (`mod 3^9`).

For division by zero, the maximum value is given.

```
tv2 = 00 00 00 00 00 00 00 00 10 (1)
tx1 = 00 00 00 00 00 00 00 00 00 (0)

> tvdiv.vs tv1, tv2, tx1

tv1 = 10 10 10 10 10 10 10 10 10 (9841)
```

#### 3.4.9. Vector Remainder

Instruction: `tvrem.vv`, `001011`, `11`.

```
tvrem.vv tvd, tvs1, tvs2

Example: 
tvrem.vv tv1, tv2, tv3
001011 00001, 00010, 00011

whole instruction
0101011 001011 00001 00010 00011 0000
```

Implementation:

```
tvrem.vv: reg[tvd] = reg[tvs1] mod reg[tvs2] (elementwise)
```

For division by zero, the dividend is returned.

```
tv2 = 00 00 00 00 00 00 00 00 10 (1)
tv3 = 00 00 00 00 00 00 00 00 00 (0)

> tvrem.vv tv1, tv2, tv3

tv1 = 00 00 00 00 00 00 00 00 10 (1)
```

#### 3.4.10. Scalar Remainder

Instruction: `tvrem.vs`, `001100`, `12`.

```
tvrem.vs tvd, tvs1, txs1

Example: 
tvrem.vs tv1, tv2, tx1
001100 00001, 00010, 00001

whole instruction
0101011 001100 00001 00010 00001 0000
```

Implementation:

```
tvrem.vs: reg[tvd] = reg[tvval] mod reg[tx1] for tvval in reg[tvs1] 
```

Just like addition, overflow is wrapped (`mod 3^9`).

For division by zero, the dividend is returned.

```
tv2 = 00 00 00 00 00 00 00 00 10 (1)
tx1 = 00 00 00 00 00 00 00 00 00 (0)

> tvrem.vs tv1, tv2, tx1

tv1 = 00 00 00 00 00 00 00 00 10 (1)
```

## 4. Citations

The RISC-V Instruction Set Manual, Volume I: User-Level ISA, Document Version 20260120

The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Document Version 20260120

RISC-V "V" Vector Extension Version 1.0
