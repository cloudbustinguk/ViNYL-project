# McStasScript integration into harmonized API

This document contains a few ideas for how McStasScript could be integrated into a SIMEX like python user interface called the backenginge. In SIMEX, simulation is performed by calculators objects that contain simulation code and takes input from parameter objects that specialized for each calculator.
This user interface of McStasScript is heavily influenced by the necessity to input a lot of information into one calculator, where SIMEX is better suited for running a simulation based on a series of calculators. Reconciling the two will then most likely require concessions somewhere.

The backengine is specified in another document, but here the most important part is how calculators and parameters are instantiated:
```
calculator = CalculatorName(parameters=parameters, inpath="/path/to/input/data", outpath="/path/to/output/data")
parameters = CalculatorParameters(param1=value1*unit1, param2=value2*unit2, ...)
```
The parameter class should not contain simulation code, but only parameters with consistency checks.

## Instrument object from parameter class

The calculator could take a McStasScript instrument object as “parameters”, inpath rarely necessary, could dump data in outpath. Would violate design principles on parameters, as the parameter object would contain methods
for performing the simulation.

```
# my parameters is McStasScript instrument object
my_parameters = McStasScript(name=“ODIN”, author=“Mads”)
 
# Now that my_parameters is an instrument object, information can slowly be added
src = my_parameters.add_component(“source”, “Source_simple”)
src.xwidth = 0.1
src.yheight = 0.1
src.E0 = 10
src.dE = 2
 
my_parameters.add_parameter(“theta”, value=10)
 
mono = my_parameters.add_component(“mono”, “Monochromator_flat”)
mono.xwidth = 0.03
mono.yheight = 0.05
mono.Q = 1.22
mono.set_AT([0, 0, 1], RELATIVE=“source”)
mono.set_ROTATED([0, “theta”, 0], RELATIVE=“source”)
 
my_parameters.add_declare_var(“two_theta”)
my_parameters.append_initialize(“two_theta = 2.0*theta;”)
 
beam_dir = my_parameters.add_component(“beam_dir”, “Arm”)
beam_dir.set_AT([0,0,0], RELATIVE=“mono”)
beam_dir.set_ROTATED([0,”two_theta”,0], RELATIVE=“source”)
 
# Could have necessary datafiles and similar in inpath
# Results written to outpath
my_calculator = McStas(parameters=my_parameters, inpath=“/path/to/input/data”, outpath=“/path/to/output/data”)

# Need to supply instrument parameters to calculator, here through input dict
my_calculator.run(input={"theta": 25}, ncount=1E7)

data = my_calculator.data
```
Issues with this approach:
* Parameter class contains simulation method
* May require input of settings at calculator.run(), but could be carried in parameters.

Benefits with this approach:
* Can use most of the McStasScript interface
* Follows large part of the specifications

## Instrument object from calculator

Calculator could also be used to instantiate an instrument object, the only
problem is that it does not make much sense to insert parameters
immediately. Would need to insert parameters in run call. Also since
most information is passed after the creation of the instrument object,
there is very little information in the associated parameter class.
Internal checking is however done whenever information is added to the
instrument object regardless, so the input sanitation duty is being
performed somewhere.

```
# my parameters is small and almost trivel
my_parameters = McStasParameters(name=“ODIN”, author=“Mads”)
 
my_calculator = McStasScript(parameters=my_parameters, inpath=“/path/to/input/data”, outpath=“/path/to/output/data”)
 
src = my_calculator.add_component(“source”, “source_simple”)
src.xwidth = 0.1
src.yheight = 0.1
src.E0 = 10
src.dE = 2
 
my_calculator.add_parameter(“theta”, value=10)
 
mono = my_calculator.add_component(“mono”, “Monochromator_flat”)
mono.xwidth = 0.03
mono.yheight = 0.05
mono.Q = 1.22
mono.set_AT([0, 0, 1], RELATIVE=“source”)
mono.set_ROTATED([0, “theta”, 0], RELATIVE=“source”)
 
my_calculator.add_declare_var(“two_theta”)
my_calculator.append_initialize(“two_theta = 2.0*theta;”)
 
beam_dir = my_calculator.add_component(“beam_dir”, “Arm”)
beam_dir.set_AT([0,0,0], RELATIVE=“mono”)
beam_dir.set_ROTATED([0,”two_theta”,0], RELATIVE=“source”)
 
my_calculator.run(input = {“theta”: 25}, ncount=1E7)
 
data = my_calculator.data
```
Issues with this approach:
* May require input of some parameters at calculator.run(), but could be carried in parameters
* Parameters do no carry much information and does not get a chance to do much input sanitation

Benefits with this approach:
* Can use most of the McStasScript interface
* McStasScript completely integrated, and almost fits within specifications

## Directly supply a McStasScript object to parameters
It would also be possible to let parameters contain an McStasScript instrument object, along with other requirements for the simulation. It is for example necessary to request a certain number of simulated rays (if a source is simulated), and this information is hard to fit into the preceding schemes, but in this case would fit nicely into parameters.

```
Instr = McStasScript.McStas_instr(name=“ODIN”, author=“Mads”)

src = Instr.add_component(“source”, “source_simple”)
src.xwidth = 0.1
src.yheight = 0.1
src.E0 = 10
src.dE = 2
 
Instr.add_parameter(“theta”, value=10)
 
mono = Instr.add_component(“mono”, “Monochromator_flat”)
mono.xwidth = 0.03
mono.yheight = 0.05
mono.Q = 1.22
mono.set_AT([0, 0, 1], RELATIVE=“source”)
mono.set_ROTATED([0, “theta”, 0], RELATIVE=“source”)
 
Instr.add_declare_var(“two_theta”)
Instr.append_initialize(“two_theta = 2.0*theta;”)
 
beam_dir = Instr.add_component(“beam_dir”, “Arm”)
beam_dir.set_AT([0,0,0], RELATIVE=“mono”)
beam_dir.set_ROTATED([0,”two_theta”,0], RELATIVE=“source”)

# Create parameter object
my_parameters = McStasParameters(instr=Instr, ncount=1E7, gravity=True, input= {"theta" : 25})

my_calculator = McStasScript(parameters=my_parameters, inpath=“/path/to/input/data”, outpath=“/path/to/output/data”)
 
my_calculator.run()
 
data = my_calculator.data
```
Issues with this approach:
* Not really harmonized, as McStasScript is just used next to the API

Benefits with this approach:
* Can use most of the McStasScript interface
* Actually fits with specifications, but McStasScript more seperated from backengine that others





