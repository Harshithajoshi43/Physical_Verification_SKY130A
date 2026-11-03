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


5 . ENVIRONMENT SETUP
```bash
lsb_release -a
uname -a
free -h
df -h /
nproc
```

Update packages and install Git:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git -y
git --version
```
<img width="1080" height="617" alt="image" src="https://github.com/user-attachments/assets/92c4f3de-fa77-4c14-90c7-65f3445c2b6d" />

## 3. Tool Installation

### Magic

```bash
git clone git://opencircuitdesign.com/magic
cd magic
sudo apt install tcl-dev tk-dev -y
./configure
make
sudo make install
```
<img width="1080" height="602" alt="image" src="https://github.com/user-attachments/assets/1e929542-7732-46ab-9983-4a9ce37efab8" />

