![GitHub all releases](https://img.shields.io/github/downloads/leninkelvin/VanillaAMBER/total?style=for-the-badge&logo=appveyor)
![GitHub forks](https://img.shields.io/github/forks/leninkelvin/VanillaAMBER?style=for-the-badge&logo=appveyor)
![GitHub Repo stars](https://img.shields.io/github/stars/leninkelvin/VanillaAMBER?style=for-the-badge&logo=appveyor)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.7008274.svg)](https://doi.org/10.5281/zenodo.7008274)

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
trajin 5R8T_wat_md3.mdcrd # read in the trajectory to analyse

strip :WAT # removing WAT molecules
strip :Na+ # removing sodium atoms
strip :Cl- # removing chlorine atoms
center :1-304 mass origin
image origin center
autoimage         # all of this to center the solute
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
RUn the command:

```
cpptraj -p 5R8T_wat.prmtop -i analysis1.in
```

And you should have new files in your directory. If, something went wrong, you would have to troubleshoot. Read the errors since they are informative and the output is verbose. For visualization [grace](https://plasma-gate.weizmann.ac.il/Grace/) is the best. You can install it in linux like this:
```
sudo apt install grace
```

If it is already installed:
```
xmgrace rmsA.dat &
```
The result should look like this, X-axis is frames not time, and Y-axis is angstroms.

![RMSD](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/rmsA.png)

Now, RMSF. Here X-axis is residue and Y-axis is displacement (check the manual). The image is presented as grace displays it. It, of course, has to be modified and adjusted to be published. 

```
xmgrace rmsfA.dat &
```

Notice the last residues are really moving. It is worthwhile to check if an atom (or residue) not attached to the solute was taken into consideration in the calculation. 
 
![RMSF](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/rmsfA.png)

To visualize hydrogen binds, run the command:

```
xmgrace -nxy DAnhb.dat ADnhb.dat
```
Even thou I opened two files **DAnhb.dat** and **ADnhb.dat** we can only see one line. It is because the results are the same result.

![NHB](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/ADnhb.png)

We can visualize secondary structure in two ways, both calculated above. [Gnuplot](http://www.gnuplot.info) can be installed with:

```
sudo apt install gnuplot
```

First, with gnuplot:
```
gnuplot dssp.gnu
```
![DSSP](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/dssp.png)
```
This format gives a view of the secondary structure per residue vs time. It will allow us to percieve changes in initial secondary structure. In this case, it is difficult to identify specific regions because residue labels are overlapping.
Another visualization can be achieved using grace:
xmgrace -nxy dssp.gnu.sum &
```
![DSSPSUM](https://github.com/leninkelvin/VanillaAMBER/blob/main/material/dssppr.png)
Here we see total of secondary structure (Y-axis) vs residue (X-axis). It allows for the observation of regions fluctuating between different structures. 

These are only a few of the analyses that can be carried out with amber. Hopefully, this will help you start off on your own.

# Créditos

Author [Lenin Domínguez-Ramírez](https://github.com/leninkelvin)

# Data

If you want the output data to compare there is a copy at figshare:

https://doi.org/10.6084/m9.figshare.20514864.v1
