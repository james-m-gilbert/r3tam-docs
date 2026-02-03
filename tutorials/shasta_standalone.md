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

# Shasta Reservoir: Standalone Simulation - Part One

For this first tutorial, we will work towards simulating water temperature in Shasta Reservoir as a single model component - that is, without interacting with any other downstream information or conditions.
Later tutorials will cover a Keswick Reservoir component and methods for coupling these components together.

Because Shasta Reservoir simulations are a bit more involved (more inputs, more parameters, selective withdrawal functions), this tutorial is broken into two parts. 
This first part covers the inputs to the model via a sequential walkthrough of the main input specification file.
The second covers running the model under different configurations and reviewing basic results.

## A brief introduction to Shasta Reservoir Temperature Simulation

There are many great resources for information on the history and technical details of water temperature management at Shasta Dam and Reservoir -- and how it is commonly simulated in computer models -- so I won't go into much depth here.
For now, the main point to consider is that the temperature of water stored in and released from Shasta Reservoir is affected by several key factors or processes:

  1. The temperature and volume of water flowing into the reservoir
  2. The mixing and stratification (distinctly warmer water separating from cooler water below) of water within the reservoir volume
  3. Heat exchange of water from the area around the reservoir (mainly the surface), and
  4. Selective withdrawal that releases water from different points in the water column to achieve a desired downstream release temperature.

In real life we can see from records of temperature profiles in Shasta Reservoir that each of these factors can have different effects at different times of the year or when the reservoir is empty vs full. 
One key characteristic of Shasta Reservoir is that it statifies seasonally - meaning that a volume of water in the upper part of the vertical profile warms more than water at depth, creating a distinct boundary between two temperature zones. 
The water in this cooler, lower zone is often what is desired for coldwater temperature management for salmon during the hot summer months.

In this version of R3TAM, we use a vertically discretized one-dimensional model we call *ResTemp* (for "Reservoir Temperature") to represent those four factors and their interaction in a reservoir like Shasta.
These factors are addressed by a set of calculations that occur at least once each time step - which for the Shasta *ResTemp* model is one day.
At each time step, the model integrates information read in from user-provided input files, performs calculations, then updates the water temperature and water density conditions (we sometimes refer to these as the system "states").
The output from each time step is a record of those system states as well as other outcomes of interest - primarily the temperature of water released from Shasta Dam (e.g. via penstocks or river outlets) to the tailwater below. 

You can read more about the specifics of the process represenations in the reference sections (*currently under development*) or the Technical Memorandum document.

The construction of the *ResTemp* code - including how the main factors mentioned above are translated to computational terms -- determines the type and form of information and data that need to be provided by a modeler when performing simulations.
We provide a description of these inputs in the sections below on our way toward running a standalone Shasta temperature simulation.

## Model Inputs

The following sections refer to files that support a Shasta Reservoir simulation using the ResTemp model.
You can find these files in the `shasta` folder within the examples provided [on GitHub](https://github.com/james-m-gilbert/r3tam/tree/test-release/examples) or in a zipped folder [here](files/examples.zip).

### Run Specification File

A ResTemp simulation requires a range of different information and data, including:

  - Reservoir volume, area, elevation relationships
  - Vertical discretization
  - Parameter values
  - Input time series data
  - Release options and characteristics
  - Time period to simulate

This information is provided as input to the model via a in [YAML](https://yaml.org/about/) specification file.
A screenshot of an example YAML specification file for a ResTemp Shasta Reservoir model is shown in {numref}`fig-yamlspec-sha`.
The specification file is structured around *key-value* entries where the *key* is a text entry followed by a colon. 
The *value* entry follows the colon and can take the form of a single value (numeric, text, date, etc) or a path to a separate file that contains other types of information. 

The specification file `shasta_standalone.yaml` provided with the Shasta Reservoir example simulation has all the required information already set, but we provide a brief overview of the types and formats of information below.

```{figure} ./figs/restemp_shasta_specyaml_example.png
:name: fig-yamlspec-sha

Screenshot of top portion of an example ResTemp model specification file in YAML format.
```

#### General Information

The first few entries in the specification file provide general information that applies to the entire simulation and model files.
These are described in {numref}`tbl-restemp-geninput` below:

```{table} ResTemp Model Specification - General Inputs
:name: tbl-restemp-geninput
:align: left

| Entry  | Description        | 
| ---------  | -----------        |
| **RunName**   | Name to be assigned to the model run - output files will use this name, so avoid spaces for better filesystem usability (*required*)|
| **Description** | A longer description of the simulation, helpful to provide notes to yourself about important characteristics of a specific model run (*optional*) |
| **FacilityName** | Name of the reservoir being simulated (*optional*) |
| **SurfaceMethod** | Sets the surface energy exchange method. Currently this should be set to `simple`, but a full energy balance option is being develped (*required - set to `simple`*) |
| **SimulationUnits** | Sets the unit system in which to simulate. Options are `afdegF` for acre-feet and degrees Fahrenheit and `m3degC` for cubic meters and degrees Celsius. For the early release version, the `afdegF` setting is recommended. (*required*) | 
| **Location** | Description, latitute, and longitude provide information about the reservoir. (*optional*)  | 

```

#### Parameters

The second section in the specification file sets the parameter values used by the `ResTemp` simulation components.
The first five (C1-C5) are the primary parameters, and the remaining set of parameters are used to set different options for the simulation.

The parameters and the calibrated values provided as part of the Shasta Reservoir example model are listed in {numref}`tbl-restemp-params`.


```{table} ResTemp Model Specification - Parameters - Values shown are unitless unless otherwise noted
:name: tbl-restemp-params
:align: left

| Parameter | Value | Description |
| --- | --- | --- |
| **C1** | 0.225 | Affects rate of air-water heat exchange at the water surface |
| **C2** | 0.375 | Affects efficiency of heating from absorbed incoming shortwave solar radiation |
| **C3** | 0.980 | Affects efficiency of latent heat exchange due to evaporation |
| **C4** | 0.0525 |Affects rate of heat exchange from inflowing water |
| **C5** | 0.004 | Affects rate of diffusive mixing among layers |
| **DiffuseDist** | 15 (feet) | Distance over which to calculate interlayer diffusion (units set in `DiffuseDistUnits`) |
| **CriticalDepth** | 35 (feet) | Depth from surface at which surface heat exchange goes to zero (units set in `CritcalDepthUnits`) |
| **MeanAlbedo**  | 0.08 | Controls the balance of reflection andabsorption of solar radiation at the water surface |
| **AlbedoAmplitude** | 0.025  | Controls seasonal variability of albedo; if 0, then `MeanAlbedo` applies uniformly for each time step |
| **ExtinctionCoeff** | 0.45 | |
| **SeasSolRad** | 1.0 | An option for modifying solar radiation to account for seasonality; value of 1 means no modification |
| **WindMethod** | 'w2' | Set method by which wind speed is converted to information needed for wind-driven mixing and evaporation; 'w2' uses the same method as in CE-QUAL-W2 |
| **WindAlpha** | 9.45 | Parameter needed for the 'w2' `WindMethod` option |
| **WindBeta** | 0.46 | Parameter needed for the 'w2' `WindMethod` option |
| **WindGamma** | 2.05 | Parameter needed for the 'w2' `WindMethod` option |
| **WindHeight_m** | 10.0 | The height at which input wind speed is measured (assumed in units of meters). This is needed for potential adjustment to 2-m height assumed for wind mixing and evaporation calculations |
| **BlendingMethod** | 0.0 | Option for optimized blending among point sinks [0 = Blending using only hydraulic approximations; 1 = linear program optimized blending to match specified target temperature] | 

```

#### Simulation Properties

The specification file also includes entries to set required configuration information like timing, initial conditions, and simulation mode options.
A screenshot of these entries from the example Shasta specification file is shown in {numref}`fig-timingmode-sha` and explained in more detail below.


 ```{figure} ./figs/restemp_shasta_timingmodeinit_example.png
:name: fig-timingmode-sha

Screenshot of top portion of an example ResTemp model specification file in YAML format.
``` 

##### Timing 

The `*Timing*` entry includes two sub-entries `StartDate` and `EndDate`. 
The input for each should follow a *"month/day/year"* format. 
As an example, to start the simulation on January 1, 2000, one would enter '1/1/2000' as the value to the `StartDate` key.

The input datasets gathered for the Shasta model cover the period from January 1, 2000 through September 30, 2022. 
The input specification file for the first example sets the simulation period to a smaller window - January 1, 2014 through December 31, 2015. 

```{admonition} Some notes on simulation timing
:class: note
:class: dropdown

Check to make sure that the `EndDate` entry is later than the `StartDate`. 
The simulation will not run if this is not the case.

Another thing to keep in mind is that the simulation timing tells the ***ResTemp*** model what time series data to look for as it advances through each timestep. Make sure that input time series provide data for the time range set by the timing.

The ResTemp model assumes a daily time step. The `TimeStepDays` key that is included in example specification files should be set to 1.0 for now. The option for sub-daily time steps is currently under consideration for future development as it requires some substantial changes to the model functions.

```

(sec-initcond)=
##### Initial condition 

The *Intial Condition* entry is used to provide input to the model regarding how to intialize the reservoir states (storage volume and temperature). 
Current functionality is limited to reading in and assigning these states from observational data - this is is set by assigning the `Method` sub-entry as *'from_obs'*.
See the (sec-obsprof) for more information about the observation data options and requirements. 

```{admonition} Some notes on simulation timing
:class: note
:class: dropdown

The simulation start date set via `Timing > StartDate` should correspond closely with a date of available observation data - that is, reservoir storage and temperature profiles. In the common case where the desired simulation start date and the available profile data do not exactly align, the model will automatically find the closest date to the set start date and assign the profile data accordingly. This is important to keep in mind when evaluating simulated temperature profiles around these initialization dates.

```

(sec-simmodes)=
##### Simulation modes

There are several user-selectable options that affect how a simulation proceeds or the types of information required as input. 
For the most part, these options are set under the `Simulation Mode` key in the specification file. 
The options used for the standalone Shasta Reservoir simulation are described here.

  - **Hindcast**: Set to `True` to indicate that gate operations are prescribed; if set to `False`, and the *SimGateOps* is set to `True`, then ***ResTemp*** will attempt to automatically operate gates to meet a prescribed temperature target. This is set to `True` for this example because we are simulating over a historical period where we have observed inputs and actual gate operations.
  - **SimGateOps**: Set to `False` if  **Hindcast = True**; set to `True` if you want to dynamically operate temperature control gate options to meet a downstream temperature target
  - **GateChgFreq**: An integer number of days that sets the shortest interval over which a gate change can be made when run in **Hindcast = False** and/or **SimGateOps = True** modes. Ignored if **Hindcast = True**.
  - **SimReleaseSchedule**: [*Not currently functional*] - An option to dynamically set a reservoir release schedule based on flood rule requirements and other delivery rules
  - **CalcEvap**: Sets whether surface evaporation is dynamically calculated. If `True`, then dew point data needs to be provided as an input time series. If `False`, the model will expect a prescribed evaporation time series input. For this example we set it to `True` to demonstrate the functionality. It can be set to `False` and use the USBR-reported reservoir evaporation provided in the input files.
  - **ReinitMonth**: If value is between 1 and 12, the model will attempt to reinitialize the reservoir profile on the specified month (e.g. 1 = January, 6 = June, 12 = Decmeber) if the simulation period covers multiple years. Specified in conjunction with **ReinitDay**. If set to 99, no reinitialization will occur.
  - **ReinitDay**: Integer to indicate day of the month (as set in **ReinitMonth**) to reinitialize the temperature profile in the reservoir. If **ReinitMonth** is set to 99, this value is ignored.
  - **Debug**: The flags for **Release** and **General** accept integer values and, when set to values greater than 0, will print out to the console window and log files various diagnostic information that can be helpful in debugging errors or understanding reasons for certain results. *Note: Increasing the **Debug** flags from 0 will slow down the simulation time.*

You can see how these options are set in the specification file for the Shasta reservoir example in {numref}`fig-timingmode-sha` above.


### Reservoir Outlets

A key part of simulating a reservoir with selective withdrawal capabilities is defining the characteristics of the different release or withdrawal options. 
The ***ResTemp*** model includes three categories of release options:

  1. Flood release at the dam crest or spillway elevation
  2. Penstock release with (or without) selective withdrawal and leakage
  3. River release

The options and configuration information related to each of these, and how those details are set for the Shasta example, are explained below.

#### Flood Release

The ***ResTemp*** model assumes that any time reservoir storage exceeds the maximum volume (as prescribed in the elevation-area-volume characteristics - see {numref}`sec-resspec`), and other outlets are at capacity or otherwise not able to evacuate the necessary volume of water, the excess storage above the dam crest or spillway elevation is released as a flood spill.
Beyond the definition of the elevation-volume relationship for a reservoir and dam, no other information is required as input.

#### Penstocks and Selective Withdrawal

The penstock release option is meant to represent the release of water through power-generating penstocks at a single set elevation.
The penstock release can be augmented with additional options or constraints with selective withdrawal units (more on that below) and leakage units (again, more on that below).
This approach was informed by the needs of simulating the Temperature Control Device (TCD) at Shasta Reservoir and follows, to the extent possible, the methods and techniques implemented in CE-QUAL-W2 models of Shasta Reservoir {cite:p}`deas2020b,daniels2018`.

(penstockspec)=
##### Penstock Specifications

```{margin}
Recall that as a 1-D model, ***ResTemp*** cannot simulate lateral variation in temperatures
```  
Penstock releases are assumed to occur through one or more outlets at a single uniform elevation that can be represented with a single total capacity (i.e. as the sum of capacities of individual penstock pipes). Based on this, the key information that should be provided under the `Penstocks` entry are penstock elevation and penstock capacity (release limit).

For the Shasta reservoir model, the penstock information is shown below.

% :name: yamlspec-penstock
% :caption: Penstock specifications for Shasta Reservoir

```yaml

Penstocks:
   PenstockElevation: 819.5
   PenstockElevUnits: 'ft'
   PenstockReleaseLimit: 18750.
   PenstockReleaseLimit_Units: 'cfs'

```

(swdspec)=
##### Selective Withdrawal Specifications

While the [penstock information](penstockspec) is relatively simple, the information required to define selective withdrawal configuration is a bit more involved.
The options and information required by ***ResTemp*** for selective withdrawal are a result of the model being originally developed to capture the particular functionality (and idiosyncracies) of the Shasta TCD. 
Applying the model to other reservoirs may not require the full set of information described in this sub-section

The `Selective Withdrawal` entry represents a section with multiple sub-entries. The first sub-entry (`CheckMinHead`) should indicate whether minimum head constraints are checked for each selective withdrawal element.
The remaining entries set the characteristics and parameters of one or more selective withdrawal elements. 
These elements represent one or more gates that can be operated (opened and closed) independently but at a uniform elevation. 
For the Shasta TCD, these elements represent the four gate levels  - the upper, middle, lower (pressure relief), and side gate levels.

The following snippet shows the section of the example Shasta input specification file that defines the four selective withdrawal elements used to represent the Shasta TCD.

% :name: yamlspec-shaswd
% :caption: Selective withdrawal element (gate) specifications for Shasta Reservoir

```yaml
Selective Withdrawal:
  CheckMinHead: True
  3:
    Name: 'UpperGate'
    TopElev: 1045.
    TopElevUnits: 'ft'
    BottomElev: 1000.
    BottomElevUnits: 'ft'
    NumberOfGates: 5
    MinHead: 20.
    MinHeadUnits: 'ft'
    A1GT: 1 
    B1GT: 1.33 
    G1GT: 1.3 
    NumPointSinks: 3
  2:
    Name: 'MiddleGate'
    TopElev: 945.
    TopElevUnits: 'ft'
    BottomElev: 900.
    BottomElevUnits: 'ft'
    NumberOfGates: 5
    MinHead: 35.
    MinHeadUnits: 'ft'
    A1GT: 1.
    B1GT: 1.33 
    G1GT: 1.4 
    NumPointSinks: 3
  1:
    Name: 'PressureReliefGate'
    TopElev: 831.
    TopElevUnits: 'ft'
    BottomElev: 804.
    BottomElevUnits: 'ft'
    NumberOfGates: 5
    MinHead: 35.
    MinHeadUnits: 'ft'
    A1GT: 1.0
    B1GT: 1.33 
    G1GT: 1.45 
    NumPointSinks: 3
  0:
    Name: 'SideGate'
    TopElev: 800. 
    TopElevUnits: 'ft'
    BottomElev: 720.
    BottomElevUnits: 'ft'
    NumberOfGates: 2
    MinHead: 35.
    MinHeadUnits: 'ft'
    A1GT: 2.5
    B1GT: 1.1 
    G1GT: 1.0 
    NumPointSinks: 5

```

Each element or gate is set by assigning an integer number, ordered from lowest elevation to highest. 
For the Shasta TCD, that means that the side gate is assigned the number 0 and the upper-most gate is assigned the number 3.

Each gate can have a descriptive name assigned via the `Name` entry - this isn't used by the model but is a helpful reminder of which gate is which. 
The `TopElev` and `BottomElev` entries set the top and bottom elevations of the gates that make up that element or gate level. 
The corresponding units entries designate whether the elevations are in feet (`ft`) or meters (`m`) - the example Shasta model uses feet.

Each gate level can have one or more operable gates or openings through which water is withdrawn. This is set using the `NumberOfGates` entry (5 for the upper, middle, and lower gate levels and 2 for the side gate on the TCD). 
When providing specified gate operations as inputs, the model will not recognize values that exceed the `NumberOfGates` setting for a gate level. Similarly, the model will open and close gates on a level up to the number set by `NumberOfGates` when dynamic gate operations are activated.

The determination of how much water flows through a specific gate level when one or more gates on that level are open is a complicated and somewhat uncertain process. The ***ResTemp*** model employs the hydraulic approximations for flow through submerged large orifices as documented in CE-QUAL-W2. The parameters `A1GT`, `B1GT`, & `G1GT` are used in these hydraulic approximations and help define the maximum flow through an opening with the specified height (from the top and bottom elevations) and hydraulic head (determined from the reservoir water surface elevation). These hydraulic calculations are used only to determine the relative amount of flow through gate levels with one or more gates open. If only a single gate level has open gates, then it is assumed that the full penstock release goes through that gate level.

The `MinHead` entry sets the minimum head requirement, relative to that level's bottom elevation, for that gate level to be operated in isolation. This constraint is in effect only if the `CheckMinHead` entry is set to `True`. Using the *UpperGate* entry as an example, for a gate on that level to be opened if all lower gate levels are closed, the reservoir surface elevation must be at least 1020 feet (`BottomElev` [1000 ft] + `MinHead` [20 ft]). 

Finally, the `NumPointSinks` sets the number of discrete point locations across the vertical span of an open gate level at which withdrawal envelopes are calculated (this affects the aggregate TCD release temperature and the removal of water from within the water column). 
The large gate openings on selective withdrawal structures such as the Shasta TCD cannot be well represented by a single calculation point and so are approximated using multiple point sinks that are distributed across the vertical span of the open gate. Increasing the number of point sinks may improve resolution of release temperatures and/or affects to the temperature profile in the reservoir, but may also increase simulation time. 
The number of point sinks set in the example Shasta specification file were found to be a good approximation.

(lkgspec)=
##### Leakage

One factor that complicates the representation of the Shasta TCD is that the operation of the gates alone cannot account for the temperature of the water released through the penstocks. 
Incorporating the effects of leakage through non-operable surfaces of the TCD structure improves the performance of water temperature simulations, so we do the same with ***ResTemp***. 
The approximation of leakage phenomena implemented here follows the approach described by {cite:p}`deas2020b` and consists of eight (8) leakage zones at various locations on the TCD.

Each leakage zone is characterized by a top and bottom elevation (`TopElev` and `BottomElev` as in the [Selective Withdrawal section](swdspec)) that determine where in the water column the leakage water will be drawn from. 
The other entries in the specification file (`LeakageFactor` and `OutletExclude`) only apply if an older leakage representation is used (see {cite:p}`daniels2018` for more details). 

Because the leakage forumulation is specific to the Shasta TCD, the remaining parameters used to implement the leakage calculations are hard-coded as a special function in ***ResTemp***. 
The interested modeler can review the specifics in the function *calc_leakageMDWE* in the `outflow.py` file in the [R3TAM code repository](https://github.com/james-m-gilbert/r3tam/blob/main/src/r3tam/outflow.py). 

(rivoutspec)=
#### River Outlets

The final option for releases is through river outlets. ***ResTemp*** river outlets can be assigned at different elevations but, like with penstocks and selective withdrawal elements, cannot capture lateral variation and must represent a total capacity for all outlets at a given elevation.
In contrast to the selective withdrawal elements, the river outlets are assumed to be small enough that a single point sink for each outlet element is appropriate.

The Shasta Reservoir example simulation uses three (3) river outlet elements to represent river outlet structures at three levels on the dam face. 
The entries for these elements are shown in the snippet below.
The elevations (`TopElev` and `BottomElev` as in other release structure specifications) and total capacity are set for each river outlet element. The capacities represent the sum of capacities from individual river outlets (e.g. if there are three river outlets on a level with a capacity of 10,000 cfs each, the listed capacity for that element in the specification file would be 30,000 cfs).

```yaml

RiverOutlets:
  3:
    TopElev:  946.25 
    TopElevUnits: 'ft'
    BottomElev: 937.75 
    BottomElevUnits: 'ft'
    Capacity: 28800. 
    CapacityUnits: 'cfs'
  2:
    TopElev: 837.75 + (102/12.)
    TopElevUnits: 'ft'
    BottomElev: 837.75
    BottomElevUnits: 'ft'
    Capacity: 30000.
    CapacityUnits: 'cfs'
  1:
    TopElev: 737.75 + (102/12.)
    TopElevUnits: 'ft'
    BottomElev: 737.75
    BottomElevUnits: 'ft'
    Capacity: 20000.
    CapacityUnits: 'cfs'

```

(sec-resspec)=
### Reservoir Specifications

The *ResTemp* formulation treats a reservoir as a single volume of water that can vary vertically  but is assumed to be uniform laterally. 
To distinguish this vertical variation in temperature the reservoir volume is assigned a certain number of layers by the modeler. 
The number and thickness (they don't all have to be the same size) of the layers determines how finely or coarsely the resulting temperature simulation will be, and it can affect the accuracy of the simulation.

#### Layer Elevations

The discretization of the reservoir volume into layers is set via the `LayerElevations` entry in the specification file (shown below).
The `Values` sub-entry below `LayerElevations` can take a direct listing of elevation values or a path to a text file that includes such a listing. 
This file is provided for the Shasta example model in the `inputs` folder and is titled `layers42.txt`, as shown in the input specification file.

The file contains a simple listing of index and elevation values (in units of feet) ordered sequentially from low to high. 
Each elevation value sets the bounds of a layer - there are 43 values listed to define the top and bottom elevation of 42 layers.
Note that the layer definitions encompass the elevation range of a full Shasta reservoir - that is, up to 1067 feet above mean sea level. 
This sets the maximum elevation of the reservoir, but the actual volume at any given time step will depend on the mass balance of flows in an out of the reservoir.

```yaml
LayerElevations:
  Units: 'ft'
  Values: ['inputs/layers42.txt']
```

(elevareavolspec)=
#### Elevation-Volume and Elevation Area Characteristics

The ResTemp model tracks water volume, temperature, and heat exchange at each layer. 
This requires that the volume and area of each layer be defined. 
This is done through two more look-up tables that are set through the `ElevAreaTable` and `ElevStorTable` entries in the input specification file (below).

For the Shasta Reservoir example, these files are provided in the `inputs` folder:

  - **ShastaElevationAreaTable.txt**: Relates the surface area of each layer top/bottom to the elevation
  - **ShastaElevationCapacityTable.txt**: Sets the total volume of the reservoir when water is filled to that elevation

The ResTemp code reads in these three files (layer specification: `layers42.txt`, elevation-area: `ShastaElevationAreaTable.txt`, & elevation-volume: `ShastaElevationCapacityTable.txt` ) as part of the model initialization and automatically assigns incremental volumes and elevations to each layer that comprises the model. 

``` yaml
ElevAreaTable:
  Units: 'ft'
  Values: ['inputs/ShastaElevationAreaTable.txt']
ElevStorTable:
  Units: 'ft'
  Values: ['inputs/ShastaElevationCapacityTable.txt']

```

### Time Series & Observation Inputs

The input specifications described so far have either been simple key-value pairs or paths to basic lookup tables. 
The remaining inputs described in this section encompass time series data used as inputs to drive the model (e.g. inflows and meteorology) and observational data used to evaluate or initialize the model (e.g. temperature profiles).

The following sections describe the different input types and specific files used for the example Shasta Reservoir model.
Within the input specification file, the input time series (inflows, releases, and meteorology) are set under the `TimeSeriesInputs` main entry and observation data (measured temperature profiles, records of outflow temperatures) are set under the `Observations` main entry.

(note-tsinputs)=
```{admonition} A few notes on time series input files
:class: note

With the exception of water temperature profile data, the time series inputs should be provided in a consistent comma-separated value (CSV) format for each of the required input types. 
This format includes the following requirements:
  1. A non-blank name for each column of data (preferably without spaces)
  2. The first column consists of a continuous (no gaps) and sequential (in chronological order) index of valid date values
  3. Each column of data should be filled (no blanks) and include only values that are valid for the input type; examples of invalid values might be a negative inflow value or a value of 354 degrees for air temperature.

General categories of time series (Inflow, Outflow, Meteorology) can be specified from a single file or multiple files, but all data of a particular category must come from the same file.

Files can contain additional columns that are not used by the model. ***ResTemp*** will only read in and use the columns of data listed in the specification file.

```

(sec-inflows)=
#### Reservoir Inflows

Current ***ResTemp*** models use only a single time series to set inflows to the reservoir. 
For reservoirs fed by multiple tributaries, like Shasta, these individual tributary flows should be accumulated into a single time series.
This inflow time series is specified through the `Inflow` entry (under the main `TimeSeriesInputs` entry, as shown in the snippet below).
The first part of the `Inflow` entry is the `Path` - this is where the path to a comma-separated value (csv) formatted time series file is provided (see [the above note on time series inputs](note-tsinputs)).

The `Header` entry for inflows identifies the column names in the input CSV file that correspond to the inflow volumes (`inflow`) and inflow water temperature (`inflowTemp`).
In the snippet below you can see that the second `inflow` line has been commented out using a *#* character. Doing this allows you to quickly switch between different time series if you have multiple versions in the input file that you'd like to test out. 

``` yaml
TimeSeriesInputs:
  Inflow:
    Path: inputs/ShastaInputs_2000to2022v20250225.csv
    Header:
      inflow: 'Inflow_minus_precip_AF' # use precip as part of surface heat exchange
      #inflow: 'Inflow_Total_AF'      # lump precip in with inflow
      inflowTemp: 'InflowTemp_degF'

```

(sec-outflows)=
#### Reservoir Outflows

The inputs in `Outflow` section (under the main `TimeSeries` entry, as with `Inflow`) set the releases and gate operations for the reservoir. 

The `Path` entry sets the location of a CSV file that contains the required time series for total outflows, selective withdrawal gate operations, and river outlet operations.

The `Header` section contains entries that set the column names for the following specific outflow components:

  - `outflow`: The total volume of water released through the penstocks (does not include river outlet releases) for each time step; *Note - this should be the cumulative volume for the time step (e.g. one day), not a flow rate*
  - `gates`: Sets the gate operations for the selective withdrawal elements as defined in the [selective withdrawal section][swdspec]; provided as a dictionary (bounded by braces) where the keys are integer indicators of the selective withdrawal elements and the values are text strings corresponding to the column name in the input file with the number of gates open on that level for each day
  - `tempTarg`: The name of the column that sets the temperature target immediately downstream of the dam; this is only used if dynamic gate operations are activated
  - `rivOutFlow`: The total volume of water released through all river outlets for each time step; *Note - this should be the cumulative volume for the time step (e.g. one day), not a flow rate*
  - `rivOutlets`: Sets the river outlet operations for the available river outlets; provided as a dictionary (bounded by braces) where the keys are integer indicators of the river outle elements and the values are text strings corresponding to the column name in the input file with the number of outlets open on that level for each day
  - `storage`: *No longer used - was originally provided as a way to determine outflow from prescribed storage if no outflow is available*

The file and column names for the outflow time series used with the Shasta Reservoir example are shown in the snippet below:

```yaml

  Outflow:
    Path: inputs/ShastaInputs_2000to2022v20250225.csv
    Header:
      outflow: 'Total_Outflow_AF'
      gates: {3: 'Gate_Upp', 2: 'Gate_Mid', 1: 'Gate_PRG', 0: 'Gate_SDG'}
      tempTarg: 'Tw_target_degF' 
      rivOutlets: {3: 'Riv_Upp', 2: 'Riv_Mid', 1: 'Riv_Bot'}
      rivOutFlow: 'River_Outlet_Release_AF'
      storage: 'Storage_AF'

```

(sec-meteo)=
#### Meteorology

The calculation of heat exchange at the water surface requires meteorological information for each time step. 
This data is specified under the `Meterology` entry within the `TimeSeries` section of the input specification file.

As with the other `TimeSeries` categories, the `Path` entry sets the location of the CSV file with meteorological time series data.
The `Header` section contains entries that set the column names for the following specific meteorological variables:

  - `airTemp`: Name for the column containing the air temperature at or near the reservoir; the column name should be appended with an underscore characer ("_") and a units indicator ('degF' if the provided data are in degrees Fahrenheit and 'degC' if data are in degrees Celsius)
  - `solRad`: Name for the column containing the incoming shortwave solar radiation data, in units of $W m^-2$
  - `wind`: Name for the column containing the windspeed data, in units of $m s^-2$
  - `precip`: Name for the column containing precipitation data, as a depth in inches, that falls on the reservoir surface. This is needed if the affect of rainfall on both the reservoir volume and surface water temperatures are to be included in the simulation. If the temperature affect of precipitation is to be ignored, the modeler should adjust inflows or otherwise ensure that the reservoir mass balance is not substantively affected by doing so.
  - `evap`: Column that contains the time series of reservoir evaporation. The column name should end with an underscore followed by a units indicator (allowable options are "_af", "acre-feet", or "_cfs") so that the values are applied correctly. The values provided in this entry are not used if evaporation is calculated as part of the simulation (see the `CalcEvap` option in the [Simulation Modes section](sec-simmodes)).
  - `dewpt`: Name for the column containing dew point data. The column name should end with an underscore and units indicator (options are "_degC" or "_degF"). Dew point data are only used if the surface evaporation is calculated as part of the simulation
  - `tke`: Name for the column containing data for the wind-driven turbulent kinetic energy at the water surface. This data is used to more accurately represent wind mixing on daily time steps.

The entries for the `Meteorology` section of the example Shasta Reservoir model specification file are show below:

``` yaml
  Meteorology:
    Path: inputs/ShastaInputs_2000to2022v20250225.csv
    Header:
      airTemp: 'AirTemp_degF'
      solRad: 'SolRad_gridMet_Wm2' #or SolRad_Wm2 for Redding station data
      wind: 'Wind_msec'
      precip: 'Precip_in'  # use as part of surface heat exchange
      #precip: 'Precip_in_allzeros' # lump precip in with inflow
      evap: 'Evap_cfs'
      dewpt: 'Tdew_degC'
      tke: 'Wind_TKE_Term'

```

#### Observations

Observation data are used for initializing ***ResTemp*** models and for comparing to simulated results at the end of a model run.
Entries for vertical temperature profiles are the only observation data that are strictly required for a simulation, but the specifications for both release temperatures and profiles are described below.

##### Observed Outflow

The `Outflow` entry under the `Observations` section in the input specification file sets the file location and column names for data relevant to the reservoir releases and related outcomes.

The pattern for assigning the data is the same as for the [inflowa](sec-inflows), [outflows](sec-outflows), and [meteorology](sec-meteo): a `Path` entry with the location of the CSV time series file and a `Header` section with sub-entries where the column name of each data variable can be assigned.
The types of data that can be provided in this section include:

  - `flow`: Volume of penstock outflow for each time step (note this is volume and not flow rate)
  - `temperature`: Temperature of water released from the reservoir or measured at a tailbay location
  - `temperature_other`: If there is a second water temperature record, it can be provided here. For the Shasta Reservoir, there are two release temperature records - one at a sensor just downstream of the dam and one calculated from penstock temperatures
  - `storage`: Reservoir storage; this is helpful in comparing simulated storage to the observed record to ensure accurate mass balance

The observed outflow entries as provided in the example Shasta reservoir input specification file are provided here:

``` yaml
Observations:
  Outflow:
    File:
      Path: obs/ShastaOutflowStorageAreaObs_2000to2022v20230429.csv
      Header:
        flow: 'Total_Outflow_AF'
        temperature: 'Tw_Temp_degF'
        temperature_other: 'TCD_WtAvgTemp_degF'
        storage: 'Storage_AF'

```

(sec-obsprof)=
##### Temperature Profiles

The final input dataset in the specification file is for vertical water temperature profiles. These data are required for initialization (see [](sec-initcond)) and are helpful for comparing to simulated reservoir temperature profiles to evaluate model performance.

Because temperature profiles span multiple elevations for a given date, a simple time series format as used for other inputs is insufficient. Instead, the `Header` information provided in the `Profiles` input specification section (under the main `Observations` section) includes the following entries:

  - `index_cols`: A bracket-bound ("[]") list of 2 integers that designate the 0-indexed columns in the csv file that contain, in order, the *date* and *elevation* of temperature values provided in a third column; for example, if *[0,1]* is provided, the first (0 = first in a zero-based index) column should contain date information and the second (1 = second in a zero-based index) column should contain elevation data
  - `elevation`: Name of the column that contains the elevation data; this should correspond to the same column as the second integer provided to `index_cols`
  - `temperature`: Name of the column that contains the water temperature values; the column name should end with an underscore character followed by a units indicator (options are "_degC" or "_degF").

The observed profile entries as provided in the example Shasta reservoir input specification file are provided here:

``` yaml

  Profiles:
    File:
      Path: obs/shasta_temp_profiles_00toSep22.csv
      Header:
        index_cols: [0,1]
        elevation: 'Elev_ft'
        temperature: 'Temp_degF'
```



## References

```{bibliography}
:style: plain
```