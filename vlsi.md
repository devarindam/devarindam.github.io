---
title: Projects
layout: page
---

## 🔲 4-Bit Comparator with Cascadable Comparator Cells

> **EEE458 — VLSI II | Group 11**  
> Arindam Dev (1106040) & Md. Shamim Hussain (1106032)  
> *Department of EEE, Bangladesh University of Engineering and Technology (BUET)*  
> Supervised by **Dr. A.B.M. Harun-ur Rashid** & **Tariqul Islam**

---

## 📌 Overview

A full top-down VLSI design of a **4-bit magnitude comparator** using cascadable 1-bit comparator cells, verified through behavioral simulation, gate-level simulation, transistor-level schematic, layout, DRC/LVS checking, and tapeout. The design follows a complementary-cell architecture (COMPA / COMPB) to minimize logic complexity and propagation delay.

---

## 🎯 Design Specifications

| Parameter | Value |
|---|---|
| Function | Compare two 4-bit numbers A and B |
| Outputs | COUT (A > B), DOUT (A < B); both 0 means A = B |
| Cell architecture | Cascadable 1-bit comparator cells (COMPA + COMPB) |
| Total transistors | 96 (4 cells × 24 transistors each) |
| Technology | gpdk090 (90nm CMOS) |
| Propagation delay | 197 ps (TT corner) |
| Chip size | 129.37 µm × 80.9 µm |

---

## 🧠 Logic Design

Each 1-bit comparator cell takes 4 inputs — `Ai`, `Bi`, `Ci+1` (greater-from-previous), `Di+1` (less-from-previous) — and produces 2 outputs:

```
Ci  = 1  if  Ai > Bi  (or if already Ci+1 = 1)
Di  = 1  if  Ai < Bi  (or if already Di+1 = 1)
Ci = Di = 0  if  Ai = Bi  and  Ci+1 = Di+1 = 0
```

Two complementary cell types alternate across stages to reduce gate count:

**COMPA** (true inputs → inverted outputs):
```
C̄ᵢ = NOR(Ci+1, Di+1, Āᵢ, Bᵢ)
D̄ᵢ = NOR(Di+1, Ci+1, Aᵢ, B̄ᵢ)
```

**COMPB** (inverted inputs → true outputs):
```
Ci-1 = NAND(C̄ᵢ, D̄ᵢ, Aᵢ₋₁, B̄ᵢ₋₁)
Di-1 = NAND(D̄ᵢ, C̄ᵢ, Āᵢ₋₁, Bᵢ₋₁)
```

**4-bit cascade arrangement:**
```
A3,B3 → [COMPB] → [COMPA] → [COMPB] → [COMPA] → COUT, DOUT
              stage3    stage2    stage1    stage0
```

---

## ⚙️ Design Flow (Top-Down Hierarchy)

```
1. Behavioral (Verilog HDL)
        │  Quartus II functional simulation ✓
        ▼
2. Gate Level (Orcad + PSpice)
        │  Transient simulation with 74xx gates ✓
        ▼
3. Transistor Level (Cadence Schematic L)
        │  Logical effort sizing, TT/FF/SS/FS corners ✓
        ▼
4. Cell Layout (Cadence Layout XL)
        │  COMPA: 16.345µm × 8.005µm
        │  COMPB: 10.5µm × 8.045µm
        │  DRC: No errors ✓ | LVS: Match ✓
        ▼
5. Parasitic Extraction (Cadence ASSURA)
        │  Post-layout simulation ✓
        ▼
6. Full Chip + I/O Pads → Tapeout (GDS)
```

---

## 💻 Verilog Implementation

```verilog
// Top-level 4-bit comparator
module comp4(Cin, Din, A, B, Cout, Dout);
    input  Cin, Din;
    output Cout, Dout;
    input  [3:0] A, B;
    wire   [2:0] C, D;

    compb stage3(Cin,  Din,  A[3], B[3], C[2], D[2]);
    compa stage2(C[2], D[2], A[2], B[2], C[1], D[1]);
    compb stage1(C[1], D[1], A[1], B[1], C[0], D[0]);
    compa stage0(C[0], D[0], A[0], B[0], Cout, Dout);
endmodule

// COMPA cell — true inputs, inverted outputs
module compa(Cc, Dc, A, B, C, D);
    input  Cc, Dc, A, B;
    output C, D;
    assign C = ~(Cc & ~(Dc & A & ~B));
    assign D = ~(Dc & ~(Cc & ~A & B));
endmodule

// COMPB cell — inverted inputs, true outputs
module compb(C, D, A, B, Cc, Dc);
    input  C, D, A, B;
    output Cc, Dc;
    assign Cc = ~(C | ~(D | ~A | B));
    assign Dc = ~(D | ~(C | A | ~B));
endmodule
```

---

## 📊 Simulation Results

### Timing Performance (Transistor Level, TT corner)

| Parameter | Value |
|---|---|
| Rise time — output C | 91.73 ps |
| Fall time — output C | 91.73 ps |
| Rise time — output D | 39.2 ps |
| Fall time — output D | 39.2 ps |
| Propagation delay | **197 ps** |

### Process Corner Simulations

| Corner | Status |
|---|---|
| TT (Typical-Typical) | ✅ Pass |
| FF (Fast-Fast) | ✅ Pass |
| FS (Fast-Slow) | ✅ Pass |
| SS (Slow-Slow) | ✅ Pass |

---

## 🏗️ Layout Details

### Cell Sizes

| Cell | Width | Height | Transistors |
|---|---|---|---|
| COMPA | 16.345 µm | 8.005 µm | 24 (12P + 12N) |
| COMPB | 10.5 µm | 8.045 µm | 24 (12P + 12N) |
| **Full chip** | **129.37 µm** | **80.9 µm** | **96 total** |

### Verification Checks

| Check | Tool | Result |
|---|---|---|
| DRC | Cadence Assura | ✅ No errors |
| LVS | Cadence Assura | ✅ Schematic & layout match |
| Parasitic extraction | ASSURA av_extracted | ✅ Completed |
| Tapeout (GDS stream out) | Virtuoso | ✅ 0 errors, 0 warnings |

### Floorplan Strategy

- Inputs `Aᵢ` and `Bᵢ` enter from top and bottom of each cell
- `Cᵢ` and `Dᵢ` propagate **horizontally** between cells
- VDD/GND rails run horizontally, distributed at right angles within cells
- Cell height is fixed; chip width scales linearly with bit-width → easily extensible to N-bit comparator

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| Quartus II | Verilog compilation & functional simulation |
| Orcad + PSpice | Gate-level schematic & transient simulation |
| Cadence Schematic L | Transistor-level schematic entry |
| Cadence Layout XL | Cell and chip layout |
| Cadence Assura | DRC, LVS, parasitic extraction |
| Cadence Virtuoso | Full chip integration & GDS stream out |

---

## 📄 References

1. Pucknell, D. A., & Eshraghian, K. *Basic VLSI Design*, 3rd ed. Prentice Hall, 1994.
2. Weste, N. H. E., Harris, D., & Banarjee, A. *CMOS VLSI Design: A Circuits and Systems Perspective*, 3rd ed. Pearson Education, 2008.
3. [Wikipedia — Integrated Circuit Layout](http://en.wikipedia.org/wiki/Integrated_circuit_layout)
