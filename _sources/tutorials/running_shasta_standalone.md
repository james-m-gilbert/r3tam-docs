---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
bibliography:
  - ../references.bib
---

# Shasta Reservoir: Standalone Simulation - Part Two

If you finished the [first tutorial on the Shasta ResTemp model](./shasta_standalone.md), you now have some familiarity with the types of information required as input and some of the options available for running. 
In this part of the tutorial we'll go through the simpler process of using that information and options to actually run some simulations.

## Setting up a simulation

The yaml specification file we covered in the first part of this tutorial contains all the input information, but to make a simulation we need to connect that information to the ***R3TAM*** code that does the calculations and advances through our prescribed simulation period.
We do this through a relatively Python script. In fact, the rest of this tutorial will be describing the contents of just such a Python file that can be used to run a Shasta model.

To make things easier, I've provided the files needed to run the example simulations in a zipped folder here. 
If you haven't already done so, download these files and place them in a directory where you'll want to do your test runs.
Once you've done that, navigate to the *shasta* folder (`examples/shasta`) and note the arrangement of files and folders.
You should see a folder named `inputs`, one named `obs`, the YAML run specification file `shasta_standalone.yaml`, and several Python (*\*.py*) files.

You may recognize the `inputs` folder as the location of several input files (layers, area-elevation-volume curves, time series inputs) that are set in the specification file. 
Similarly the `obs` folder contains the observation files for temperature profiles and release temperatures.

Once you've gotten an overview of the main shasta run folder, open the `shasta_stanadlone_example01.py` file in Spyder, VSCode, or your Python IDE of choice. We'll go over the parts of this script and how they're used to run the Shasta example simulation.
For each part you should be able to run the relevant code in your Python interpreter/IDE - remember to switch to the new virtual environment to which you installed the ***R3TAM*** package if you haven't already done so.

## Running a *ResTemp* Simulation for Shasta Reservoir

Python scripts for running *ResTemp* or ***R3TAM*** models can become rather complicated (e.g. for coupled multi-model simulations or ensemble forecasting), but for a basic standalone simulation they are straightforward.
The scripts generally include four important components:
  1. Package/module imports
  2. Initialization
  3. Advancing through the simulation
  4. Finalizing and reviewing results

The following sections review how the contents of `shasta_stanadlone_example01.py` address each of these.

### Package imports

At the top of the Python file `shasta_stanadlone_example01.py` you'll see some notes describing the purpose of this script and a number of import statements. 
The first set of imports - for `pandas` and `numpy`, shown in the snipped below - are not strictly necessary for running the model, but they are commonly used and can be helpful if you want to explore datasets produced by the simulation. 

``` python
# import some standard libraries
import sys, os
import pandas as pd
import numpy as np

```

Below that are several lines that import the ***R3TAM*** modules needed for the simulation. 
Because this is a standalone (i.e. no other models are coupled) simulation, the only part of ***R3TAM*** we need is the *ResTemp* module. 
This line is active while the other imports are commented out. Importing the other modules would not cause a problem, but for the sake of this example we leave them commented out.
Note on line 28 that we do import a module called `make_plots` - this is a set of plotting functions that works directly with the results of a *ResTemp* simulation.
These plots can be convenient for reviewing model results after a simulation. We'll discuss that further below.

``` python
# import the R3TAM libraries - since this is just a Shasta reservoir 
# simulation, we really only need the ResTemp module
from r3tam import restemp as rt
#from r3tam import longtemp as lt
#from r3tam.river import River
#from r3tam import coupling

# the `make_plots` library has helpful plots for viewing results
from r3tam import make_plots as mkp

```

Note that we import `restemp` and rename it as `rt`. This is mainly for brevity - you could import it without renaming if you prefer.
Many of the key simulation functions for the *ResTemp* model can be accessed through this `rt` object.

The final import for the `time` package is simply for measuring how long the simulations take. 
***R3TAM*** is supposed to be *rapid* - might as well check to know for sure.

Run this import section in your Python interpreter according to the methods of your IDE. 
In IDEs that recognize the `#%%` cell demarcation, you should be able to simply run each cell (e.g with `Ctrl+Enter`).
Running the import section should produce no errors or console messages.

### Initialization

In the initialization step, all of the information in the YAML specification file is read in and used to set up the model object and the parameters of the simulation. 
The key thing we need is the location of the specification file. 
We set this path to the variable `config_fp` as shown in the example on line 37.
You'll want to change the file path here to match the location of the YAML specification file on your own computer.

We can now use that `config_fp` variable as an argument to the `initialize_model` function:

``` python

config_fp = r'D:\02_Projects\SacTemp\SimTemp\R3TAM\examples\shasta\shasta_standalone.yaml'

shasta = rt.Res.initialize_model(config_fp, profile_temp_units='degF')

```

The `rt` (*ResTemp* module) contains a number of objects and functions. The one we access here, `Res`, is for a reservoir.
We can access the `initialize_model` from this reservoir object. 
This initialization function takes one required argument - the path to a YAML specification file - and has several optional keyword arguments. 
One of these keyword arguments is `profile_temp_units` - we set this to `degF` as the temperature profile data provided in the input files are in degrees Fahrenheit.

Run the two commands (or the cell). You should see some information printed to the console screen - consistent with the listing shown below.
This readout should confirm much of the information provided as input and can be helpful in debugging or tracing unexpected results.

The first part confirms the location of files and the model directory - if this isn't where you intend to run the model, now you can correct it.
The second part confirms the units for calculations followed by a listing of penstock configurations. 
The final portion of the listing displays the assigned initial temperature profile, going from lowest layers to higher until the initial volume is reached. 
Depending on the initialization volume, some layers will show `nan` for `Temp` and `TotE` as these layers are empty - this is as expected and is not an error.


``` 

Loading configuration file: D:\02_Projects\SacTemp\SimTemp\R3TAM\examples\shasta\shasta_standalone.yaml
working directory set as: D:\02_Projects\SacTemp\SimTemp\R3TAM\examples\shasta
Assigning Debug levels

-------------------------------------------
Model calculations will be done in units of:
	ACRE-FEET, ACRES, and DEG F
-------------------------------------------
Unknown units for river outlet capacity - assuming CFS
Unknown units for river outlet capacity - assuming CFS
Unknown units for river outlet capacity - assuming CFS


-----------------------------------------------
   Reading in PENSTOCK configuration       
		Variable:   PenstockElevation
		Variable:   PenstockElevUnits
		Variable:   PenstockReleaseLimit
			Setting penstock release limit to 18750.0
		Variable:   PenstockReleaseLimit_Units
------------------DONE---------------------------

assigning outflow data

*Adjusting windspeed to equivalent 2-m height from 10.0 m

              DateTime      Elev_ft  Temp_degF
0  2000-01-21 00:00:00  1031.096834      51.44
1  2000-01-21 00:00:00  1031.044342      51.44
2  2000-01-21 00:00:00  1031.021376      51.44
3  2000-01-21 00:00:00  1031.027938      51.44
4  2000-01-21 00:00:00  1031.034499      51.44
assigning profile assuming 'tidy' format
  Layer ---- Temp  ----- TotE ------ Vol ------ MaxVol 
=====================================
  0       45.3920     22469.0      495.0       495.0
  1       45.4064     495338.7      10909.0       10909.0
  2       45.5720     505985.9      11103.0       11103.0
  3       45.7160     560432.4      12259.0       12259.0
  4       45.7340     747110.6      16336.0       16336.0
  5       45.8891     615831.2      13420.0       13420.0
  6       45.9267     724907.6      15784.0       15784.0
  7       45.9517     903456.1      19661.0       19661.0
  8       46.0501     971749.0      21102.0       21102.0
  9       46.1554     1186931.8      25716.0       25716.0
  10       46.4109     1371906.1      29560.0       29560.0
  11       46.6830     1577977.2      33802.0       33802.0
  12       47.1389     1808577.3      38367.0       38367.0
  13       47.5503     2041285.3      42929.0       42929.0
  14       48.0396     2284713.3      47559.0       47559.0
  15       48.7852     2572640.8      52734.0       52734.0
  16       50.0020     2931617.8      58630.0       58630.0
  17       50.4860     3290526.0      65177.0       65177.0
  18       50.4860     3615504.4      71614.0       71614.0
  19       50.4860     3955780.0      78354.0       78354.0
  20       50.4860     4317360.8      85516.0       85516.0
  21       50.4860     4673085.1      92562.0       92562.0
  22       50.4922     5066382.8      100340.0       100340.0
  23       50.5040     5443523.1      107784.0       107784.0
  24       50.5040     5867302.2      116175.0       116175.0
  25       50.5040     6260778.9      123966.0       123966.0
  26       50.5040     6713294.7      132926.0       132926.0
  27       50.5220     7146741.1      141458.0       141458.0
  28       50.5400     5144466.6      101790.0       151017.0
  29       nan     nan      0.0       159741.0
  30       nan     nan      0.0       169833.0
  31       nan     nan      0.0       179264.0
  32       nan     nan      0.0       189926.0
  33       nan     nan      0.0       200603.0
  34       nan     nan      0.0       211922.0
  35       nan     nan      0.0       223206.0
  36       nan     nan      0.0       235179.0
  37       nan     nan      0.0       246549.0
  38       nan     nan      0.0       259112.0
  39       nan     nan      0.0       270518.0
  40       nan     nan      0.0       283609.0
  41       nan     nan      0.0       205373.0

```

If you got something similar to the console listing shown above, your Shasta *ResTemp* model is initialized and ready to run.
We've assigned the model to the variable name `shasta`, so now we can access the reservoir functions in combination with the Shasta-specific data through this object.

### Run the model

