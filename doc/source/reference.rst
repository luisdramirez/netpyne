.. _package_reference:

Package Reference
=======================================

Model components and structure
-------------------------------

Creating a network model requires:

* A dictionary ``netParams`` with the network parameters.

* A dictionary ``simConfig`` with the simulation configuration options.

* A call to the method(s) to create and run the network model, passing as arguments the above dictionaries, e.g. ``createAndSimulate(netParams, simConfig)``.

These components can be included in a single or multiple python files. This section comprehensively describes how to define the network parameters and simulation configuration options, as well as the methods available to create and run the network model.


Network parameters
-------------------------

The ``netParams`` dictionary includes all the information necessary to define your network. It is composed of the following 3 lists:

* **popParams** - list of populations in the network

* **cellParams** - list of cell property rules (e.g. cell geometry)

* **connParams** - list of network connectivity rules


The ``netParams`` organization is consistent with the standard sequence of events that the framework executes internally:

* creates a ``Network`` object and adding inside a set of ``Population`` and ``Cell`` objects based on ``popParams``

* sets the cell properties based on ``cellParams`` (checking which cells match the conditions of each rule)

* creates a set of connections based on ``connParams`` (checking which presynpatic and postsynaptic cells match the connectivity rule conditions). 

The image below illustrates this process:

.. image:: figs/process.png
	:width: 50%
	:align: center


Additionally, ``netParams`` may contain the following single-valued params:

* **scale**: Scale factor multiplier for number of cells (default: 1)

* **sizeX**: x-dimension (horizontal length) network size in um (default: 100)

* **sizeY**: y-dimension (vertical height or cortical depth) network size in um (default: 100)

* **sizeZ**: z-dimension (horizontal depth) network size in um (default: 100)

* **defaultWeight**: Default connection weight, in ms (default: 1)

* **defaultDelay**: Default connection delay, in ms (default: 1)

* **propVelocity**: Conduction velocity in um/ms (e.g. 500 um/ms = 0.5 m/s) (default: 500)

* **scaleConnWeight**: Connection weight scale factor (default: 1)

* **popTagsCopiedToCells**: List of tags that will be copied from the population to the cells (default: ['popLabel', 'cellModel', 'cellType'])

Other arbitrary entries to the ``netParams`` dict can be added and used in the custom defined functions for connectivity parameters (see :ref:`function_string`). 

Population parameters 
^^^^^^^^^^^^^^^^^^^^^^^^^^

Each item of the ``popParams`` list consists of a dictionary that defines the properties of a network population. It includes the following fields:

* **popLabel** - An arbitrary label for this population assigned to all cells; can be used to as condition to apply specific connectivtiy rules.

* **cellType** - Arbitrary cell type attribute/tag assigned to all cells in this population; can be used as condition to apply specific cell properties. 
	e.g. 'Pyr' (for pyramidal neurons) or 'FS' (for fast-spiking interneurons)

* **numCells** or **density** - The total number of cells in this population or the density in neurons/mm3 (one or the other is required). 
	The volume occupied by each population can be customized (see ``xRange``, ``yRange`` and ``zRange``); otherwise the full network volume will be used (defined in ``netParams``: ``sizeX``, ``sizeY``, ``sizeZ``).
	
	``density`` can be expressed as a function of normalized location (``xnorm``, ``ynorm`` or ``znorm``), by providing a string with the variable and any common Python mathematical operators/functions. e.g. ``'1e5 * exp(-ynorm/2)'``.

* **cellModel** - Arbitrary cell model attribute/tag assigned to all cells in this population; can be used as condition to apply specific cell properties. 
	e.g. 'HH' (standard Hodkgin-Huxley type cell model) or 'Izhi2007' (Izhikevich 2007 point neuron model). 

* **xRange** or **xnormRange** - Range of neuron positions in x-axis (horizontal length), specified 2-element list [min, max]. 
	``xRange`` for absolute value in um (e.g. [100,200]), or ``xnormRange`` for normalized value between 0 and 1 as fraction of ``sizeX`` (e.g. [0.1,0.2]).

* **yRange** or **ynormRange** - Range of neuron positions in y-axis (vertical height=cortical depth), specified 2-element list [min, max]. 
	``yRange`` for absolute value in um (e.g. [100,200]), or ``ynormRange`` for normalized value between 0 and 1 as fraction of ``sizeY`` (e.g. [0.1,0.2]).

* **zRange** or **znormRange** - Range of neuron positions in z-axis (horizontal depth), specified 2-elemnt list [min, max]. 
	``zRange`` for absolute value in um (e.g. [100,200]), or ``znormRange`` for normalized value between 0 and 1 as fraction of ``sizeZ`` (e.g. [0.1,0.2]).

Examples of standard population::

	netParams['popParams'].append({'popLabel': 'Sensory',  'cellType': 'PYR', 'cellModel': 'HH', 'ynormRange':[0.2, 0.5], 'density': 50000})


It is also possible to create a special type of population consisting of NetStims (NEURON's artificial spike generator), which can be used to provide background inputs or artificial stimulation to cells. The actual NetStim objects will only be created if the population is connected to some cells, in which case, one NetStim will be created per postsynaptic cell. is The NetStim population contains the following fields:

* **popLabel** - An arbitrary label for this population assigned to all cells; can be used to as condition to apply specific connectivtiy rules. (e.g. 'background')

* **cellModel** - Needs to be set to ``NetStim``.

* **rate** - Firing rate in Hz (note this is the inverse of the NetStim interval property).

* **noise** - Fraction of noise in NetStim (0 = deterministic; 1 = completely random).

* **number** - Max number of spikes generated (default = 1e12)

* **source** - Source of noise (optional; currently set to ``random`` by default, which is the only option implemented)

Example of NetStim population::
	
	netParams['popParams'].append({'popLabel': 'background', 'cellModel': 'NetStim', 'rate': 100, 'noise': 0.5})  # background inputs

Finally, it is possible to define a population composed of individually-defined cells by including the list of cells in the ``cellsList`` dictionary field. Each element of the list of cells will in turn be a dictionary containing any set of cell properties such as ``cellLabel`` or location (e.g. ``x`` or ``ynorm``). An example is shown below::

	cellsList = [] 
	cellsList.append({'cellLabel':'gs15', 'x': 1, 'ynorm': 0.4 , 'z': 2})
	cellsList.append({'cellLabel':'gs21', 'x': 2, 'ynorm': 0.5 , 'z': 3})
	netParams['popParams'].append({'popLabel': 'IT_cells', 'cellModel':'Izhi2007b', 'cellType':'IT', 'cellsList': cellsList}) #  IT individual cells



Cell property rules
^^^^^^^^^^^^^^^^^^^^^^^^

The rationale for using cell property rules is that you can apply cell properties to subsets of neurons that match certain criteria, e.g. only those neurons of a given cell type, and/or of a given population, and/or within a certain range of locations. 

Each item of the ``cellParams`` list contains a dictionary that defines a cell property rule, containing the following fields:

* **label** - Arbitrary name which identifies this rule.

* **conditions** - Set of conditions required to apply the properties to a cell. 
	Defined as a dictionary with the attributes/tags of the cell and the required values, e.g. {'cellType': 'PYR', 'cellModel': 'HH'}. 

* **sections** - Dictionary containing the sections of the cell, each in turn containing the following fields (can omit those that are empty):

	* **geom**: Dictionary with geometry properties, such as ``diam``, ``L`` or ``Ra``. 
		Can optionally include a field ``pt3d`` with a list of 3D points, each defined as a tuple of the form ``(x,y,z,diam)``

	* **topol**: Dictionary with topology properties.
		Includes ``parentSec`` (label of parent section), ``parentX`` (parent location where to make connection) and ``childX`` (current section --child-- location where to make connection).
	
	* **mechs**: Dictionary of density/distributed mechanisms.
		The key contains the name of the mechanism (e.g. ``hh`` or ``pas``)
		The value contains a dictionary with the properties of the mechanism (e.g. ``{'g': 0.003, 'e': -70}``).
	
	* **syns**: Dictionary of synapses (point processes). 
		The key contains an arbitrary label for the synapse (e.g. 'NMDA').
		The value contains a dictionary with the synapse properties (e.g. ``{'_type': 'Exp2Syn', '_loc': 1.0, 'tau1': 0.1, 'tau2': 1, 'e': 0}``). 
		
		Note that properties that are not internal variables of the point process are denoted with an underscore:

		* ``_type``, the name of the NEURON mechanism, e.g. ``'Exp2Syn'``.
		* ``_loc``, section location where to place synapse, e.g. 1.0, default=0.5.
	
	* **pointps**: Dictionary of point processes (excluding synapses). 
		The key contains an arbitrary label (e.g. 'Izhi')
		The value contains a dictionary with the point process properties (e.g. ``{'_type':'Izhi2007a', 'a':0.03, 'b':-2, 'c':-50, 'd':100, 'celltype':1})`. 
		
		Note that properties that are not internal variables of the point process are denoted with an underscore: 

		* ``_type``,the name of the NEURON mechanism, e.g. ``'Izhi2007a'``
		* ``_loc``, section location where to place synapse, e.g. ``1.0``, default=0.5.
		* ``_vref`` (optional), internal mechanism variable containing the cell membrane voltage, e.g. ``'V'``.
		* ``_synList`` (optional), list of internal mechanism synapse labels, e.g. ['AMPA', 'NMDA', 'GABAB']

* **vinit** - (optional) Initial membrane voltage (in mV) of the section (default: -65)
	e.g. ``cellRule['sections']['soma']['vinit'] = -72``

* **spikeGenLoc** - (optional) Indicates that this section is responsible for spike generation (instead of the default 'soma'), and provides the location (segment) where spikes are generated.
	e.g. ``cellRule['sections']['axon']['spikeGenLoc'] = 1.0``

Example of two cell property rules::

	## PYR cell properties (HH)
	cellRule = {'label': 'PYR_HH', 'conditions': {'cellType': 'PYR', 'cellModel': 'HH'},  'sections': {}}

	soma = {'geom': {}, 'topol': {}, 'mechs': {}, 'syns': {}}  # soma properties
	soma['geom'] = {'diam': 18.8, 'L': 18.8, 'Ra': 123.0, 'pt3d': []}
	soma['geom']['pt3d'].append((0, 0, 0, 20))
	soma['geom']['pt3d'].append((0, 0, 20, 20))
	soma['mechs']['hh'] = {'gnabar': 0.12, 'gkbar': 0.036, 'gl': 0.003, 'el': -70} 
	soma['syns']['NMDA'] = {'_type': 'ExpSyn', '_loc': 0.5, 'tau': 0.1, 'e': 0}

	dend = {'geom': {}, 'topol': {}, 'mechs': {}, 'syns': {}}  # dend properties
	dend['geom'] = {'diam': 5.0, 'L': 150.0, 'Ra': 150.0, 'cm': 1}
	dend['topol'] = {'parentSec': 'soma', 'parentX': 1.0, 'childX': 0}
	dend['mechs']['pas'] = {'g': 0.0000357, 'e': -70} 
	dend['syns']['NMDA'] = {'_type': 'Exp2Syn', '_loc': 1.0, 'tau1': 0.1, 'tau2': 1, 'e': 0}

	cellRule['sections'] = {'soma': soma, 'dend': dend}  # add sections to dict
	netParams['cellParams'].append(cellRule)  # add rule dict to list of cell property rules


	## PYR cell properties (Izhi)
	cellRule = {'label': 'PYR_Izhi', 'conditions': {'cellType': 'PYR', 'cellModel': 'Izhi2007'},  'sections': {}}

	soma = {'geom': {}, 'pointps':{}, 'syns': {}}  # soma properties
	soma['geom'] = {'diam': 18.8, 'L': 18.8, 'Ra': 123.0}
	soma['pointps']['Izhi'] = {'_type':'Izhi2007a', '_vref':'V', 'a':0.03, 'b':-2, 'c':-50, 'd':100, 'celltype':1}
	soma['syns']['NMDA'] = {'_type': 'ExpSyn', '_loc': 0.5, 'tau': 0.1, 'e': 0}

	cellRule['sections'] = {'soma': soma}  # add sections to dict
	netParams['cellParams'].append(cellRule)  # add rule to list of cell property rules


.. note:: As in the example above, you can use temporary variables/structures (e.g. ``soma`` or ``cellRule``) to facilitate the creation of the final dictionary ``netParams['cellParams']``.

.. ​note:: Several cell properties may be applied to the same cell if the conditions match. The latest cell properties will overwrite previous ones if there is an overlap.

.. seealso:: Cell properties can be imported from an external file. See :ref:`importing_cells` for details and examples.


Connectivity rules
^^^^^^^^^^^^^^^^^^^^^^^^

The rationale for using connectivity rules is that you can create connections between subsets of neurons that match certain criteria, e.g. only presynaptic neurons of a given cell type, and postsynaptic neurons of a given population, and/or within a certain range of locations. 

Each item of the ``connParams`` list contains a dictionary that defines a connectivity rule, containing the following fields:

* **preTags** - Set of conditions for the presynaptic cells. 
	Defined as a dictionary with the attributes/tags of the presynaptic cell and the required values e.g. ``{'cellType': 'PYR'}``. 

	Values can be lists, e.g. ``{'popLabel': ['Exc1', 'Exc2']}``. For location properties, the list values correspond to the min and max values, e.g. ``{'ynorm': [0.1, 0.6]}``

* **postTags** - Set of conditions for the postynaptic cells. 
	Same format as ``preTags`` (above).

* **sec** (optional) - Name of target section on the postsynaptic neuron (e.g. ''`soma'``). 
	If omitted, defaults to 'soma' if exists, otherwise to first section in the cell sections list. 

* **synReceptor** (optional) - Label of target synapse on the postsynaptic neuron (e.g. ``'AMPA'``). 
	If omitted employs first synapse in the cell synapses list.
	
* **weight** (optional) - Strength of synaptic connection (e.g. ``0.01``). 
	Associated to a change in conductance, but has different meaning and scale depending on the synapse and cell model. 

	Can be defined as a function (see :ref:`function_string`).

	If omitted, defaults to ``netParams['defaultWeight'] = 1``.

* **delay** (optional) - Time (in ms) for the presynaptic spike to reach the postsynaptic neuron.
	Can be defined as a function (see :ref:`function_string`).

	If omitted, defaults to ``netParams['defaultDelay'] = 1``

* **probability** (optional) - Probability of connection between each pre- and postsynaptic cell (0 to 1).

	Can be defined as a function (see :ref:`function_string`).

	Sets ``connFunc`` to ``probConn`` (internal probabilistic connectivity function).

	Overrides the ``convergence`` and ``divergence`` parameters.

* **convergence** (optional) - Number of pre-synaptic cells connected to each post-synaptic cell.

	Can be defined as a function (see :ref:`function_string`).

	Sets ``connFunc`` to ``convConn`` (internal convergence connectivity function).

	Overrides the ``divergence`` parameter; has no effect if the ``probability`` parameters is included.

* **divergence** (optional) - Number of post-synaptic cells connected to each pre-synaptic cell.

	Can be defined as a function (see :ref:`function_string`).
	
	Sets ``connFunc`` to ``divConn`` (internal divergence connectivity function).

	Has no effect if the ``probability`` or ``convergence`` parameters are included.

* **connFunc** (optional) - Internal connectivity function to use. 
	Its automatically set to ``probConn``, ``convConn`` or ``divConn``, when the ``probability``, ``convergence`` and ``divergence`` parameters are included, respectively. Otherwise defaults to ``fullConn``, ie. all-to-all connectivity.

	User-defined connectivity functions can be added.

Example of connectivity rules:

.. code-block:: python

	## Cell connectivity rules
	netParams['connParams'] = [] 

	netParams['connParams'].append({
		'preTags': {'popLabel': 'S'}, 
		'postTags': {'popLabel': 'M'},  #  S -> M
		'sec': 'dend',					# target postsyn section
		'syn': 'NMDA',					# target synapse
		'weight': 0.01, 				# synaptic weight 
		'delay': 5,					# transmission delay (ms) 
		'probability': 0.5})				# probability of connection		

	netParams['connParams'].append(
		{'preTags': {'popLabel': 'background'}, 
		'postTags': {'cellType': ['S','M'], 'ynorm': [0.1,0.6]}, # background -> S,M with ynrom in range 0.1 to 0.6
		'synReceptor': 'NMDA',					# target synapse 
		'weight': 0.01, 					# synaptic weight 
		'delay': 5}						# transmission delay (ms) 


.. note:: NetStim populations can only serve as presynaptic source of a connection. Additionally, only the ``fullConn`` (default) and ``probConn`` (using ``probability`` parameter) connectivity functions can be used to connect NetStims. NetStims are created *on the fly* during the implementation of the connectivity rules, instantiating one NetStim per postsynaptic cell.

.. _function_string:

Functions as strings
^^^^^^^^^^^^^^^^^^^^^^^

Some of the parameters (``weight``, ``delay``, ``probability``, ``convergence`` and ``divergence``) can be provided using a string that contains a function. The string will be interpreted internally by NetPyNE and converted to the appropriate lambda function. This string may contain the following elements:

* Numerical values, e.g. '3.56'

* All Python mathematical operators: '+', '-', '*', '/', '%', '**' (exponent), etc.

* Python mathematical functions: 'sin', 'cos', 'tan', 'exp', 'sqrt', 'mean', 'inf'

* Python random number generation functions: 'random', 'randint', 'sample', 'uniform', 'triangular', 'gauss', 'betavariate', 'expovariate', 'gammavariate' (see https://docs.python.org/2/library/math.html for details)

* Cell location variables:
	* 'pre_x', 'pre_y', 'pre_z': post-synaptic cell x, y or z location.

	* 'pre_normx', 'pre_normy', 'pre_normz': normalized pre-synaptic cell x, y or z location.
	
	* 'post_x', 'post_y', 'post_z': post-synaptic cell x, y or z location.
	
	* 'post_normx', 'post_normy', 'post_normz': normalized post-synaptic cell x, y or z location.
	
	* 'dist_x', 'dist_y', 'dist_z': absolute Euclidean distance between pre- and postsynaptic cell x, y or z locations.
	
	* 'dist_normx', 'dist_normy', 'dist_normz': absolute Euclidean distance between normalized pre- and postsynaptic cell x, y or z locations.
	
	* 'dist_2D', 'dist_3D': absolute Euclidean 2D (x and z) or 3D (x, y and z) distance between pre- and postsynaptic cells.

	* 'dist_norm2D', 'dist_norm3D': absolute Euclidean 2D (x and z) or 3D (x, y and z) distance between normalized pre- and postsynaptic cells.

	
* Single-valued numerical network parameters defined in the ``netParams`` dictionary. Existing ones can be customized, and new arbitrary ones can be added. The following parameters are available by default:
	* 'sizeX', 'sizeY', 'sizeZ': network size in um (default: 100)

	* 'defaultWeight': Default connection weight, in ms (default: 1)

	* 'defaultDelay': Default connection delay, in ms (default: 1)

	* 'propVelocity': Conduction velocity in um/ms (default: 500)


String-based functions add great flexibility and power to NetPyNE connectivity rules. They enable the user to define a wide variety of connectivity features, such as cortical-depth dependent probability of connection, or distance-dependent connection weights. Below are some illustrative examples:

* Convergence (num presyn cells targeting postsyn) uniformly distributed between 1 and 15:

	.. code-block:: python

		netParams['connParams'].append(
			'convergence': 'uniform(1,15)',
		# ... 

* Connection delay set to minimum value of 0.2 plus a gaussian distributed value with mean 13.0 and variance 1.4:
	
	.. code-block:: python

		netParams['connParams'].append(
			'delay': '0.2 + gauss(13.0,1.4)',
		# ...

* Same as above but using variables defined in the ``netParams`` dict:

	.. code-block:: python

		netParams['delayMin'] = 0.2
		netParams['delayMean'] = 13.0
		netParams['delayVar'] = 1.4

		# ...

		netParams['connParams'].append(
			'delay': 'delayMin + gauss(delayMean, delayVar)',
		# ...

* Connection delay set to minimum ``defaultDelay`` value plus 3D distance-dependent delay based on propagation velocity (``propVelocity``):

	.. code-block:: python

		netParams['connParams'].append(
			'delay': 'defaultDelay + dist_3D/propVelocity',
		# ...

* Probability of connection dependent on cortical depth of postsynaptic neuron:

	.. code-block:: python

		netParams['connParams'].append(
			'probability': '0.1+0.2*post_y', 
		# ...

* Probability of connection decaying exponentially as a function of 2D distance, with length constant (``lengthConst``) defined in network parameters:

	.. code-block:: python

		netParams['lengthConst'] = 200

		# ...

		netParams['connParams'].append(
			'probability': 'exp(-dist_2D/lengthConst)', 
		# ...


Simulation configuration
--------------------------

.. - Want to have more control, customize sequence -- sim module related to sim; net module related to net
.. - Other structures are possible (flexibiliyty) - e.g. can read simCfg or netparams from disk file; can load existing net etc

Below is a list of all simulation configuration options by categories:

Related to the simulation and netpyne framework:

* **duration** - Duration of the simulation, in ms (default: 1000)
* **dt** - Internal integration timestep to use (default: 0.025)
* **randseed** - Random seed to use (default: 1)
* **createNEURONObj** - Create HOC objects when instantiating network (default: True)
* **createPyStruct** - Create Python structure (simulator-independent) when instantiating network (default: True)
* **verbose** - Show detailed messages (default: False)

Related to recording:

* **recordCells** - List of cells from which to record traces. Can include cell gids (eg. 5), population labels (eg. 'S' to record from one cell of the 'S' population), or 'all', to record from all cells. NOTE: All items in ``plotCells`` are automatically included in ``recordCells``. (default: [])
* **recordTraces** - Dict of traces to record (default: {} ; example: {'V_soma':{'sec':'soma','pos':0.5,'var':'v'}})
* **recordStim** - Record spikes of cell stims (default: False)
* **recordStep** - Step size in ms for data recording (e.g. 1)

Related to file saving:

* **filename** - Name of file to save model output (default: 'model_output')
* **timestampFilename**  - Add timestamp to filename to avoid overwriting (default: False)
* **savePickle** - Save data to pickle file (default: False)
* **saveJson** - Save dat to json file (default: False)
* **saveMat** - Save data to mat file (default: False)
* **saveTxt** - Save data to txt file (default: False)
* **saveDpk** - Save data to .dpk pickled file (default: False)
* **saveHDF5** - Save data to save to HDF5 file (default: False)


Related to plotting and analysis:

* **plotRaster** - Whether or not to plot a raster (default: True)
* **maxspikestoplot** - Maximum number of spikes to plot (default: 3e8)
* **orderRasterYfrac** - Order cells in raster by yfrac (default is by pop and cell id) (default: False)
* **plotCells** - Plot recorded traces for this list of cells. Can include cell gids (eg. 5), population labels (eg. 'S' to record from one cell of the 'S' population), or 'all', to record from all cells. NOTE: All items in ``plotCells`` are automatically included in ``recordCells``. (default: [] ; example: [5,10,'PYR'])
* **plotLFPSpectrum** - Plot power spectral density (PSD) of LFP (default: False) (not yet implemented)
* **plotConn** - Plot connectivity matrix (default: False) (not yet implemented)
* **plotWeightChanges** - Plot weight changes (default: False) (not yet implemented)
* **plot3dArch** - plot 3d architecture of network (default: False) (not yet implemented)


Structure of data and code
---------------------------

* Framework module 
* Network, Population and Cell classes
* Simulation and analysis modules

Network, Population and Cell classes
-------------------------------------

* Network
	* net.setParams()
	* net.createPops()
	* net.createCells()
	* net.connectCells()
	* net.fullConn()
	* net.probConn()
	* net.convConn()
	* net.divConn()

* Population
	* pop.createCells()
	* pop.createCellsFixedNum()
	* pop.createCellsDensity()
	* pop.createCellsList()

* Cell
	* cell.create()
	* cell.createPyStruct()
	* cell.createNEURONObj()
	* cell.associateGid()
	* cell.addConn()
	* cell.addStim()
	* cell.recordTraces()
	* cell.recordStimSpikes()


Package methods
----------------

Wrapper (init) methods 
^^^^^^^^^^^^^^^^^^^^^^^^

* init.createAndSimulate()

Simulation-related methods
^^^^^^^^^^^^^^^^^^^^^^^^^^

* sim.setNet()
* sim.setNetParams()
* sim.setSimCfg()
* sim.loadSimCfg()
* sim.loadSimParams()
* sim.createParallelContext()
* sim.setupRecording()
* sim.runSim()
* sim.gatherData()
* sim.saveData()


Analysis-related methods
^^^^^^^^^^^^^^^^^^^^^^^^^^

* analysis.plotRaster()
* analysis.plotTraces()


Structure of saved data
------------------------

* simConfig
* netParams
* net
* simData