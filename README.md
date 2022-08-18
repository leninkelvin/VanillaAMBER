# Vanilla AMBER
This is a guide to use AMBER MD. It will borrow material from the [Amber Tutorials](http://ambermd.org/tutorials/) tutorials as well as experience and more resources. 

We will use a SARS-CoV-2 main protease, PDB ID: [5R8T](https://www.rcsb.org/structure/5R8T). 

We will prepare the files for simulation, run the simulations, and analyse a few basic parameters.

Most of these steps can be completed using CPUs, but it is **highly recommended that you run these steps on a GPU** so you can complete them promptly and practice. 

If you are ready, let's rock!

# Using pdb4amber

Make sure that, after installing amber22 and ambertools, your system loads amber:

```
wget https://files.rcsb.org/download/5R8T.pdb
```

