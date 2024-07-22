This note include three parts:
1. the definition of Gibbs dividing surface
2. the code for getting profile density from simulation
3. the code for getting the Gibbs dividing surface position by function fitting.
## Definition

>The dividing line is drawn so that the two areas, shaded by the horizontal lines, are equal to each other, on each side of the solvent curve. Thus the surface excess of the solvent is zero. 
>"Surface Chemistry Of Solid and Liquid Interfaces" p179

![[Pasted image 20240720203016.png]]
DOI: 10.1039/b412375a

The profile density can be fitted to the function:
$$
\frac{\rho(z)}{\rho_B }= \frac 12 [1- \tanh (b(z-c))]
$$
$\rho_B$ is the density of bulk, and $b, c$ are parameters. The position of Gibbs dividing surface corresponds to the $c$.

## Get profile density

 It's easy with MDAnalysis to get profile density from MD trajectory. MDAnalysis is a friendly python library, however the manual about profile density has little example. Here is the code I calculate the profile density and save it in NumPy binary format.

- Some notes:
	- The trajectory object must have charge info, here I made a fake one.
	- The output unit is $g/\text{cm}^3$  
``` python 
from MDAnalysis.analysis.rms import rmsd
from MDAnalysis.topology import guessers
from MDAnalysis.topology.guessers import guess_types
from MDAnalysis.topology.guessers import guess_bonds
from MDAnalysis.topology.guessers import guess_atom_mass
from MDAnalysis.analysis import align
import MDAnalysis as mda
from MDAnalysis.analysis.lineardensity import LinearDensity as LD

# matplotlib
import matplotlib.pyplot as plt

import numpy as np
from scipy.optimize import curve_fit
import math

def get_profile_dens(u, resolution = 1, frames = None):
    """get profile density of the trajectiory,
    will report error if the university donot contain charge info.
    Args:
        u (mda Universe): mda trajectory
        resolution (int, optional): resolution of stat density. Defaults to 1.
        frames (int, optional): which frames are in stat. Defaults is None and all frames will be counted.
    """
    ldens = LD(u.atoms, binsize = resolution)
    ldens.run()
    return ldens.results.z.mass_density, ldens.results.z.hist_bin_edges

def plot_dens():
    dens = np.load("./profile_dens.npy")
    edges = np.load("./profile_dens_edges.npy")
    half_step = (edges[1] - edges[0])*.5
    fig, ax = plt.subplots()
    ax.plot( edges[:-1]+ half_step, dens)
    plt.stairs(dens, edges)
    plt.show()
    
def main():
    prefix = "xxxx"
    n_an = 173
    n_wt = 1555
    n_frames = 5000 # 20 fs/(step or frame)
    frames = np.arange(n_frames)
    traj = mda.Universe(f"{prefix}.gro", f"{prefix}.trr" , guess_bonds = True
    
    # add charge info
    charges = np.zeros(len(traj.atoms)) # load fake charges
    print(charges)
    traj.add_TopologyAttr('charges', charges)
    dens, edges = get_profile_dens(traj)
    np.save("./profile_dens.npy", dens)
    np.save("./profile_dens_edges.npy", edges)
```

## Fitting

In practical, we can make $\rho_B$ also a parameter, and get the value of Gibbs dividing surface $c$ by fitting. 
The profile density shape is often like a trapezium in simulation. We can make the function for fitting as this
$$
    {\rho(z)} = \frac {\rho_B }2 \left[1- \tanh \left(b\|z-z_0\|-c\right)\right]
$$
$z_0$ is the position of bulk center, usually close to the box center.
This code use `curve_fit` module in `scipy` for fitting.

``` python
from MDAnalysis.analysis.rms import rmsd
from MDAnalysis.topology import guessers
from MDAnalysis.topology.guessers import guess_types
from MDAnalysis.topology.guessers import guess_bonds
from MDAnalysis.topology.guessers import guess_atom_mass
from MDAnalysis.analysis import align
import MDAnalysis as mda
from MDAnalysis.analysis.lineardensity import LinearDensity as LD

# matplotlib
import matplotlib.pyplot as plt

import numpy as np
from scipy.optimize import curve_fit
import math

def fit_dens(dens, pos, first_guess):
    """Fitting profile density function to got the values of bulk density and Gibbs dividing surface.
    Main function is:
    $$
    \frac{\rho(z)}{\rho_B }= \frac 12 [1- \tanh (b(z-c))]
    $$
    \rho_B: bulk density
    c: position of the Gibbs divdiding surface
    Args:
        dens (np array): profiledensity values
        pos (np array): corresponding depth of profile density
    Returns:
        popt (np array): parameters of rho_B, b, c, z0, z0 is the position where at the center of bulk
    """
    popt, pcov = curve_fit(profile_dens_func, pos, dens, p0= first_guess, bounds=(0, 200))
    return popt
    
def profile_dens_func(z, rho_B, b, c, z0):
    res = 0.5* rho_B * (1- np.tanh(b * (np.abs(z- z0)- c))
    return res  

def check_fitting(dens, pos, paras):
    pred = profile_dens_func(pos, *paras)
    fig, ax = plt.subplots()
    ax.plot( pos, dens)
    ax.plot( pos, pred)
    plt.show()

dens = np.load("./profile_dens.npy")
edges = np.load("./profile_dens_edges.npy")
half_edge = 0.5 * (edges[1] - edges[0])
pos = edges[:-1] + half_edge
print(pos)

paras = fit_dens(dens, pos, first_guess= [1, 1, 30, 70])
print(f"rho_B, b, c, z0: {paras}")

check_fitting(dens, pos, paras)
```

**Results** 
![[Pasted image 20240722010645.png|300x200]]
`rho_B, b, c, z0: [ 0.9545965   0.39721626 21.19331264 70.57164109]`

- If the code raised `OptimizeWarning: Covariance of the parameters could not be estimated warnings.warn('Covariance of the parameters could not be estimated'`, one may change the first guess in the fitting to fix this.
