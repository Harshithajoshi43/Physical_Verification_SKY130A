# Physical_Verification_SKY130A
PHYSICAL VERIFICATION WITH SKY130A: STUDY NOTES

1. WHAT THIS IS ABOUT
Physical verification asks one question: does the mask layout I drew actually
implement the circuit I intended, and does it obey the foundry's manufacturing
rules? Functional verification checks behaviour (voltages, timing). Physical
verification checks the geometry itself.

Two checks do most of the work:
 - DRC (Design Rule Check): are the shapes legal for this process?
 - LVS (Layout vs Schematic): is the drawn circuit electrically identical to
   the intended one?

The workflow comes from the VSD five-day physical verification workshop. It
covers where these checks sit in an RTL-to-GDSII flow and how to find and fix
violations before tape-out.


2. THE PDK AND WHAT EACH PIECE FEEDS
SKY130A is an open 130 nm CMOS process kit. Different parts of it serve
different tools:
 - SPICE models      -> NGSpice (simulation)
 - Tech file         -> Magic (layers, layout rules, DRC)
 - LVS setup script  -> Netgen (device matching rules)
 - Cell libraries    -> reusable device and standard-cell layouts

open_pdks is the installer that reformats the SkyWater files for the open-source
tools. It places everything under a shared pdk directory (for example
/usr/share/pdk/sky130A/ or /usr/local/share/pdk/sky130A/), with a libs.tech
folder for tool-specific files and a libs.ref folder for cell libraries.

Process stack, bottom to top:
 substrate / N-well -> diffusion + poly (transistors) -> local interconnect (LI)
 -> licon / mcon contacts -> Metal1 ... Metal5, joined by Via1 ... Via4
 -> dielectric between conductive levels.


3. TOOLCHAIN AND ROLES
 Xschem   - draw the schematic, export a SPICE netlist
 NGSpice  - simulate before and after layout
 Magic    - draw layout, run DRC, extract the netlist
 Netgen   - compare layout netlist against schematic netlist (LVS)

Order of operations:
 schematic -> pre-layout sim -> layout -> DRC -> extraction -> LVS
 -> post-layout sim


4. SETUP NOTES
 - Environment used: Ubuntu 24.04 LTS in a VirtualBox VM on Windows 11.
   A newer Ubuntu release broke the tool builds, so rolling back was the fix.
 - Magic and Netgen are built from source (opencircuitdesign.com).
   Xschem is built from its GitHub repo. NGSpice comes from apt.
 - Magic build error: "too many arguments to function" (WindAddCommand,
   WindMove, WindSetWindowAreas). Cause: the newer default GCC C standard
   doesn't suit Magic's older code. Fix: make clean, then
   ./configure CFLAGS="-std=gnu17", then make.
 - Install the PDK with: ./configure --enable-sky130-pdk, make, sudo make
   install.
 - Sanity check: launch Magic with the sky130A tech file and run "tech name".
   It should print sky130A.

Per-project folder layout (from the workshop): make mag/, xschem/ and netgen/
subfolders, then copy in the PDK startup files:
 - xschemrc            -> xschem/
 - spinit (renamed .spiceinit) -> xschem/
 - sky130A.magicrc (renamed .magicrc) -> mag/
 - sky130A_setup.tcl   -> netgen/


5. THE DESIGN: A CMOS INVERTER
 Device   Model                    W      L       Fingers  Body
 NMOS     sky130_fd_pr__nfet_01v8  4.5 um 0.18 um 3        VSS
 PMOS     sky130_fd_pr__pfet_01v8  3 um   0.18 um 3        VDD

Subcircuit pin order: vdd, in, out, vss. A custom symbol was made so the
inverter can be dropped into a testbench.


6. STEP-BY-STEP FLOW
Step 1: Schematic and pre-layout simulation
 - Build the inverter and testbench in Xschem (1.8 V supply, pulse input).
 - Run the transient simulation. Output should be the inverse of the input.
   This gives a reference result before any layout exists.

Step 2: Layout in Magic
 - NMOS near the VSS rail at the bottom, PMOS near the VDD rail at the top.
 - Both devices use three fingers.
 - A Metal2 jumper is needed to get past a routing conflict (see section 8).
 - readspice only reads a netlist. It does NOT build a routed layout for you.

Step 3: DRC
 - Run: drc check, drc count, drc why.
 - "drc why" explains which rule a flagged error violates.
 - Target: 0 errors.

Step 4: Extraction
 - Run: extract all, ext2spice lvs, ext2spice.
 - The "lvs" option sets up the netlist style that Netgen expects.
 - Workshop extras: ext2spice cthresh 0.01 (ignore tiny capacitances) and
   ext2spice extresist on (include extracted resistance).

Step 5: LVS in Netgen
 - Compare the Magic-extracted netlist to the Xschem netlist, using
   sky130A_setup.tcl as the rule file.
 - Success message: "Circuits match uniquely."
 - Expected for this design: 2 devices and 4 nets on both sides (the three
   fingers per transistor merge into one device each).

Step 6: Post-layout simulation
 - Simulate the extracted netlist with a 10 fF load capacitor, 1.8 V supply
   and a 10 ns period pulse input.
 - The logic is still correct. The difference from pre-layout is that the
   extracted version carries real parasitics.


7. RESULTS SUMMARY
 - Pre-layout sim: output is the complement of the input.
 - DRC: 0 violations.
 - LVS: match, 2 devices, 4 nets.
 - Post-layout sim: still a working inverter, now including parasitics.


8. LVS DEBUGGING: THE MAIN LESSON
Symptom: the first LVS run showed 8 nets in layout vs 4 in the schematic, and
4 devices vs 2.

Why: a 3-finger transistor has four diffusion strips in the order
drain, source, drain, source. If the two inner source strips are not wired to
the supply rail, Magic treats each as its own floating net and does not merge
the fingers into one device.

Fixes:
 - NMOS: add Metal1 contacts on the inner source strips and tie them to VSS.
 - PMOS: same idea with VDD, but going straight up with Metal1 would run into
   the drain strip tied to "out" and short out to vdd. The workaround is a
   jumper: Metal1 -> Via1 -> Metal2 -> Via1 -> Metal1, which hops over the
   drain.

Progress of the fixes (NMOS / PMOS device counts, nets layout vs schematic):
 start: 3->2 / 3->2, 8 vs 4 nets
 NMOS fixed: 3->1 / 3->2, 5 vs 4
 partial PMOS fix: no improvement
 PMOS jumper done: 3->1 / 3->1, 4 vs 4 -> match


9. KEY TAKEAWAYS
 1. In multi-finger devices, every diffusion strip must be wired explicitly.
 2. When Metal1 is blocked, hop over with Via1 -> Metal2 -> Via1.
 3. LVS reports are diagnostic. Net and device counts tell you exactly what is
    unconnected or shorted.
 4. DRC-clean does not mean LVS-correct, and LVS-correct does not mean
    DRC-clean. Both checks are needed.
 5. Tool versions and OS choice matter for open-source EDA builds.


10. EXTRA WORKSHOP COMMANDS (from the yathAg snippet)
 - Run a standard DRC on a library cell with the run_standard_drc.py script
   in libs.tech/magic.
 - Check metal density on a GDS with check_density.py.
 - Generate fill patterns with generate_fill.py (load the cell in Magic and
   set "box values 0 0 0 0" to view fill).
 - Workshop topics also included GDSII data formats, extraction styles,
   ports and port indexes, abstract views, and XOR verification.


11. SOURCES
 - SKY130A PDK docs (skywater-pdk.readthedocs.io)
 - Magic and Netgen docs (opencircuitdesign.com)
 - VSD Physical Verification workshop (vlsisystemdesign.com)
