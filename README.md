# Calculating PF Vulnerability Indices

## Introduction
This repository contains the code used to calculated the Voltage Vulnerability Index \(VVI\) and the Power Vulnerability Index \(PVI\) described in the paper "Algorithmic Approaches to Characterizing Power Flow Cyber-Attack Vulnerabilities" published at ISGT 2019.

Running this code requires the [Matpower](https://matpower.org/download/all-releases/) library. With the exception of the `runVVI` and `runPVI` functions, this code was written with Matpower6.0 and may not be compatible with different versions. The `runVVI` and `runPVI` functions should be compatible with both Matpower6.0 and Matpower7.0. The Matlab path file should be augmented to include the folder containing these functions, as well as the Matpower folder its extra subfolders.

## Usage Instructions
### Voltage Vulnerability Index
The recommended method of finding the VVI is as follows:
```
write = true
mpc = loadcase(casedata)
VVI = runVVI(mpc, write)
```
Where casedata is the name of the power grid to be analyzed \(e.g. 'case9'\). This will return a column vector of voltage failure points of each bus in the power grid. It will also save the results to a file titled "runVVI-*n*bus.csv" where *n* is the number of buses in the system. Each row in the file represents a different bus and the columns contain the following information from the point of failure:
1) Bus index
2) Bus active power demand
3) Bus active power generation
4) Bus reactive power demand
5) Bus reactive power generation
6) Maximum voltage magnitude to cause a PF convergence failure

### Power Vulnerability Index
The method for finding the PVI is slightly more complicated than the VVI:
```
write = true
mpc = loadcase(casedata)
[S, theta, busIdx, success] = runPVI(mpc, tau, numAngles, Pmax, write)
```
Where the parameters are such:
- *casedata* is the name of the power grid intended to be analyzed
- *tau* is the radial resolution of the search. A smaller *tau* will find more precisely the minimum power needed to cause a convergence failure, but will increase the run-time logarithmically.
- *numAngles* is the number of power angles to be swept at each apparent power magnitude. By default this value is 12, meaning it will increment the power angle in steps of 30 degrees at each magnitude.
- *Pmax* is the maximum apparent power to be checked before moving on to the next bus. This is important so the algorithm does not get stuck on an unsensitive node. 
- *write* determines whether or not the function will automatically write the results to a file. By default this is 0. 

Because the PVI measures only the vulnerability of PQ buses, the output vectors will not be the length of the number of buses in the grid. The function call will return the following information:
- *S* : Column vector of the magnitude of the minimum apparent power injections \(how much is added, not total bus power\) causing a power flow convergence failure at each PQ bus
- *theta* : The angle of the apparent power injection at each bus
- *busIdx* : The corresponding bus index of each PQ bus analyzed
- *success* : A column vector showing whether each bus was successful at causing a failure

If `write = true`, the function will store the results in a file titled "runPVI-*n*bus.csv" where *n* is the number of buses in the system. If a bus is not successful at causing a failure at a given Pmax, success will be set to 0 and both S and theta will be set to infinite and should be disregarded.

## Functional Descriptions
Functions other than the two described above are also included in the repository and will be described below.

---

##### `findPQFailureFast`
This function finds the PVI of one bus of a power grid and is used in runPVI by calling it once for every PQ bus in the grid. The inputs and outputs are very similar to those of runPVI, although you must also specify a bus index to be analyzed. 

**Inputs:**
- *busIdx* : The index of the bus to be analyzed
- *tau*: The radial resolution of the search. A smaller *tau* will find more precisely the minimum power needed to cause a convergence failure, but will increase the run-time logarithmically.
- *numAngles* : The number of power angles to be swept at each apparent power magnitude. By default this value is 12, meaning it will increment the power angle in steps of 30 degrees at each magnitude.
- *Pmax* : The maximum apparent power to be checked before moving on to the next bus. This is important so the algorithm does not get stuck on an unsensitive node. 

**Outputs:**
- *S*: The minimum apparent power magnitude to cause a failure.
- *angle*: The angle of the apparent power at the lowest failure.
- *success*: Whether the bus successfully caused a power flow failure.
- *out* : a row vector containing the aggregated information about the failure point.

---

##### `findVoltageFailure`
Performs a binary search to find the minimum voltage magnitude that causes a convergence failure at a given bus index in a system. This function is called by runVVI *n* times to provide the failure points of each bus. 

**Inputs:**
- *busIdx* : The index of the bus to be analyzed.
- *mpc* : The case data of the power grid \(e.g. `mpc = loadcase('case9')`\)
- *slack* : A row vector containing the bus index of the slack bus, the generator index of the slack generator, and the real power generated by the slack generator under normal conditions.

**Outputs:**
- *Vlow* : The maximum voltage magnitude that causes a convergence failure.
- *out* : aggregated information about the failure \(bus Index, real power demand and generation, reactive power demand and generation\)
- *failure* : whether the bus was successful at causing a convergence failure at a V > 0.

---

##### `load_shed_PQ_sweep`
Performs single-bus load shedding to restore convergence after a failure caused by a power injection. This function requires a data file on the system's PVI results, which should be generated using the function `test_convergence(casedata)`.

**Inputs:**
- *casedata* : The case data for the power grid
- *PQ_sweep_data* : The csv file containing the PVI data

**Outputs:** This function creates a .csv file containing the following columns:
1) Bus index of failure
2) Bus type (will always be PQ)
3) Index of smallest bus that restores convergence when shed
4) Smallest apparent power S that can be shed to restore convergence
5) Angle of smallest apparent power
6) Number of different possible buses that can restore convergence when shed
7) Number of attempted load sheds (number of buses with nonzero S)

---

##### `my_fdi`
This is a helper function for `test_convergence_se` that changes measurement values input to the state estimation algorithm.

---

##### `slacktest_full`
This function runs the VVI calculation for an entire system and outputs a file containing the results. This function can be used in place of runVVI, but is not recommended as it is slower and may not be compatible for Matpower7.0. The primary use for this function is to create the data file, which is necessary to run the function `slacktest_load_shed`.

The recommended call for this function is `filename = slacktest_full(casedata)`, where filename stores the name of the data file and can then be input to other functions for analysis. 

---

##### `slacktest_load_shed`
Performs single-bus load shedding to restore convergence after a failure caused by a voltage drop. This function requires a data file generated by the `slacktest_full` function. 

**Outputs:** This function creates a .csv file containing the following columns:
1) Bus index of failure
2) Bus type
3) Index of smallest bus that restores convergence when shed
4) Smallest power that can be shed to restore convergence
5) Number of different buses that can be shed restore convergence
6) number of attempted load sheds (number of buses with nonzero S)

---

##### `test_convergence`
This function runs the PVI calculation for an entire system and outputs a file containing the results. This function can be run in place of runPVI, but this is not recommended as it is slower and may not be compatible with Matpower7.0. The primary use for this function is to create the data file necessary to run the function `load_shed_PQ_sweep`.

**Inputs:**
- *case_data* : The file name of the system you want to analyze
- *max_power* : The maximum apparent power the function checks before moving on to the next node
- *angular_res* : The number of angles the function sweeps at each apparent power step
- *radial_step* : The step size of the apparent power increments. Decreasing *radial_step* will improve the resolution but increase the run time linearly.

---

##### `test_convergence_se`
This function applies both the PVI and VVI calculations to the state estimation algorithm and runs them on the given system. The only input arguments are the name of the system to be analyzed and the resolution of PVI calculation. The function creates two files, one storing the VVI results for state estimation, and one storing the PVI results.  
