# Vanilla AMBER
This is a guide to use AMBER MD. It will borrow material from the [Amber Tutorials](http://ambermd.org/tutorials/) tutorials as well as experience and more resources. 

We will use a SARS-CoV-2 main protease, PDB ID: [5R8T](https://www.rcsb.org/structure/5R8T). 

We will prepare the files for simulation, run the simulations, and analyse a few basic parameters.

Most of these steps can be completed using CPUs, but it is **highly recommended that you run these steps on a GPU** so you can complete them promptly and practice. 

If you are ready, let's rock!

# Using pdb4amber

Make sure that, after installing amber22 and ambertools, your system loads amber (mine is installed in /opt/):
```
source /opt/amber22/amber.sh
```
Create a folder or otherwise place the files in a place you can reach in the command line. You don't need to download it since it is included with this repo. However, here is the command:
```
wget https://files.rcsb.org/download/5R8T.pdb
```
Now, we will use pdb4amber. For details on the commands used please see [AMBER manual](http://ambermd.org/doc12/Amber22.pdf).
```
pdb4amber -i 5R8T.pdb -o 5R8T_amber.pdb -d --most-populous
```
Below is the output:
```
==================================================
Summary of pdb4amber for: 5R8T.pdb
===================================================

----------Chains
The following (original) chains have been found:
A

---------- Alternate Locations (Original Residues!))

The following residues had alternate locations:
VAL_73
ASP_216
ASN_221
-----------Non-standard-resnames
DMS

---------- Missing heavy atom(s)

None
The alternate coordinates have been discarded.
Only the highest occupancy for each atom was kept.
```
Briefly, one chain was found (A). Three residues had alternate locations and DMS was found as heteroatoms. We need to remove these.
```
grep ATOM 5R8T_amber.pdb > 5R8T_final.pdb
```
Now, we have a suitable PDB file for a simulation. 
If you needed to use a ligand there are aditional steps to performed. 
For more information go to [Using pdb4amber](http://ambermd.org/tutorials/basic/tutorial9/index.php).

# tleap

Tleap is the program that we need to use next to prepare our files. Tons of details can be found in the Amber manual as well as in the [fundamentals of LEaP](http://ambermd.org/tutorials/pengfei/index.php).

Tleap can be use directly on the command line or through scripts. In this repo, I have included a [tleap.in](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/tleap.in).
```
tleap -f tleap.in
```
The final line on the output should read:
```
Exiting LEaP: Errors = 0; Warnings = 3; Notes = 1.
```
Warnings and notes are ok; errors would mean tleap found something that prevented the correct creation of the parameters and topology files. Possible causes are heteroatoms, unusual residues and others. 

And with this, we can run the simulations.

# pmemd

For the simulation we will need four files to start an energy minimization of the solvent, an energy minimization of the solute, temperature and pressure equilibration. These files are [min1.in](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/tleap.in), [min2.in](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/min2.in), [md1.in](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/md1.in), and [md2.in](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/md2.in). 

More details can be found in [running MD with pmemd](http://ambermd.org/tutorials/basic/tutorial14/index.php)

First energy minimization:
```
pmemd.cuda -O -i min1.in -o 5R8T_wat_min.out -p 5R8T_wat.prmtop -c 5R8T_wat.inpcrd -r 5R8T_wat_min1.rst -ref 5R8T_wat.inpcrd
```
The output of this minimization (5R8T_wat_min1.rst) will be used in the next step. 
Second energy minimization:
```
pmemd.cuda -O -i min2.in -o 5R8T_wat_min2.out -p 5R8T_wat.prmtop -c 5R8T_wat_min1.rst -r 5R8T_wat_min2.rst
```
The output of the second minimization (5R8T_wat_min2.rst) will be used in the next step. 
Temperature equilibration:
```
pmemd.cuda -O -i md1.in -o 5R8T_wat_md1.out -p 5R8T_wat.prmtop -c 5R8T_wat_min2.rst -r 5R8T_wat_md1.rst -x 5R8T_wat_md1.mdcrd -ref 5R8T_wat_min2.rst
```
The output of the temperature equilibration (5R8T_wat_md1.rst) will be used in the next step. 
Pressure equilibration:
```
pmemd.cuda  -O -i md2.in -o 5R8T_wat_md2.out -p 5R8T_wat.prmtop -c 5R8T_wat_md1.rst -r 5R8T_wat_md2.rst -x 5R8T_wat_md2.mdcrd
```
The output of the pressure equilibration (5R8T_wat_md2.rst) will be used in the next step. This next step is the **production** part of the MD. The md3.in files is set for 10 ns which will take about 3 hours on an RTX 3060.

```
pmemd.cuda  -O -i md3.in -o 5R8T_md3.out -p 5R8T.prmtop -c 5R8T_md2.rst -r 5R8T_md3.rst -x 5R8T_md3.mdcrd
```
After this step we should have material for an analysis.

# cpptraj

Among the files in this repo we have [analysis1.in](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/analysis1.in). 
While the content of all of the files included is important (read the manual) we will only delve into this last one:

```
trajin 5R8T_wat_md3.mdcrd #

strip :WAT
strip :Na+
strip :Cl-
center :1-304 mass origin
image origin center
autoimage
rms first :1-304

rms first :1-304 out rmsA.dat

atomicfluct out rmsfA :1-304 byres bfactor 

hbond donormask :1-304 acceptormask :1-304 out DAnhb.dat avgout DAavghb.dat 

hbond acceptormask :1-304 donormask :1-304 out ADnhb.dat avgout ADavghb.dat 

secstruct :1-304 out dssp.gnu 

go
```

The first line loads the trajectory from the **production** step. 

The following three lines remove water, sodium and chlorine atoms.

Then, four lines to make sure that our protein is centered and the frames (steps) of the simulation are aligned before the analysis.

Now, the next lines will calculate RMSD, RMSF, hydrogen bonds and secondary structure. 


