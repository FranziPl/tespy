.. _using_tespy_networks_label:

TESPy networks
==============

The tespy.networks.network class handles preprocessing, solving and postprocessing. We will walk you through all the important steps.

Setup
-----

Network container
^^^^^^^^^^^^^^^^^

The TESPy network contains all data of your plant, which in terms of the calculation is represented by a nonlinear system of equations. The system variables of your TESPy network are:

 * mass flow,
 * pressure,
 * enthalpy and
 * the mass fractions of the network's fluids.

The solver will solve for these variables. As stated in the introduction the list of fluids is passed to your network on creation.
If your **system includes fluid mixtures**, you should **always make use of the value ranges** for the system variables. This improves the stability of the algorithm. Try to fit the boundaries as tight as possible,
for instance, if you kwow that the maximum pressure in the system will be at 10 bar, use it as upper boundary.

.. note::

	Value ranges for pure fluids are not required as these are dealt with automatically.

.. code-block:: python

    from tespy import nwk

	fluid_list = ['CO2', 'H2O', 'N2', 'O2', 'Ar']
	my_plant = nwk.network(fluids=fluid_list)
	my_plant.set_attr(p_unit='bar', h_unit='kJ / kg')
	my_plant.set_attr(p_range=[0.05, 10], h_range=[15, 2000])
	
.. _printout_logging_label:

Printouts and logging
+++++++++++++++++++++

TESPy comes with an inbuilt logger. If you want to keep track of debugging-messages, general information, warnings or errors you should enable the logger. At the beginning of your python script e. g. add the following lines:

.. code-block:: python

	from tespy.tools import logger
	import logging
	logger.define_logging(
		log_path=True, log_version=True,
		screen_level=logging.INFO, file_level=logging.DEBUG
	)
	
The log-file will be saved to :code:`~/.tespy/log_files/` by default. All available options are documented in the :py:func:`API <tespy.tools.logger.define_logging>`.

Prior to solving the network there are options regarding the **console printouts for the calculation progress** using the :py:meth:`set_printoptions method <tespy.networks.network.set_printoptions>`.
You can choose the print_level (info or none). Check out the :py:meth:`API-documentation <tespy.networks.network.set_printoptions>` for more information.

.. code-block:: python

	myplant.set_printoptions(print_level='none') # disabling iteration information printout

Adding connections
++++++++++++++++++

As seen in the introduction, you will have to create your networks from the components and the connections between them.
You can add connections directly or via subsystems and networks holding them by using the appropriate methods:

.. code-block:: python

	myplant.add_conns()
	myplant.add_subsys()
	myplant.add_nwks()

.. note::

	You do not need to add the components to the network, as they are inherited via the added connections.
	After having set up your network and added all required elements, you can start the calculation.

Busses: power connections
+++++++++++++++++++++++++

Another type of connection is the bus: Busses are power connections for e. g. turbomachines or heat exchangers. They can be used to model motors or generators, too. Add them to your network with the following method:

.. code-block:: python

	myplant.add_busses()
	
You will learn more about busses and how they work in :ref:`this part<tespy_busses_label>`.

Start calculation
^^^^^^^^^^^^^^^^^

You can start the solution process with the following line:

.. code-block:: python

	myplant.solve(mode='design')

This starts the initialisation of your network and proceeds to its calculation. The specification of the calculation mode is mandatory, see the list of available keywords:

 * :code:`mode` is the calculation mode (design-calculation or offdesign-calculation),
 * :code:`init_path` is the path to the network folder you want to use for initialisation,
 * :code:`design_path` is the path to the network folder which holds the information of your plants design point,
 * :code:`max_iter` is the maximum amount of iterations performed by the solver,
 * :code:`init_only` stop after initialisation (True/False).

There are two calculation modes available (:code:`'design'` and :code:`'offdesign'`), which are explained in the subsections below.
If you choose :code:`offdesign` as calculation mode the specification of a :code:`design_path` is mandatory.

The usage of an initialisation path is always optional but highly recommended, as the convergence of the solution process will be improved, if you provide good starting values.
If do not specify an :code:`init_path`, the initialisation from priorly saved results will be skipped.
:code:`init_only=True` usually is used for debugging. Or, you could use this feature to export a not solved network, if you want to do the parametrisation in .csv-files rather than your python script.

Design mode
+++++++++++

The design mode is used to design your system and is always the first calculation of your plant. **The offdesign calculation is always based on a design calculation!**.
Obviously as you are designing the plant the way you want, you are flexible to choose the parameters to specify.
However, you can't specify parameters that are based on a design case, as for example the isentropic efficiency characteristic function of a turbine or a pump. Specifying a value for the efficiency is of course possible.

Offdesign mode
++++++++++++++

The offdesign mode is used to **calulate the performance of your plant, if parameters deviate from the plant's design point**. This can be partload operation, operation at different temperature or pressure levels etc..
Thus, before starting an offdesing calculation you have to design your plant first. By stating :code:`'offdesign'` as calculation mode, **components and connections will auto-switch to the offdesign mode.**
For components, this means that all parameters provided in :code:`component.design` will be unset and instead all parameters provided in :code:`component.offdesign` will be set.
This applies to connections analogously. **The value of the newly set parameter is always equal to the value from the design case (or based on it for characteristics).**

.. code-block:: python

	myplant.solve(mode='offdesign', design_path='mynetwork')

.. note::

	Since version 0.1.0 there are no default design and offdesign parameters! All design and offdesign have to be specified manually as in the example below.

You can specify design and offdesign parameters for components and connections. For example, for a condenser you would usually design it to a maximum terminal temperature difference, in offdesign the heat transfer coefficient
is selected. The heat transfer coefficient is calculated in the preprocessing of the offdesign case based on the results of the design-case. Of course, this applies to all other parameters in the same way.
Also, the pressure drop is a result of the geometry for the offdesign case, thus we swap the pressure ratios with zeta values.

.. code-block:: python

	heat_ex.set_attr(design=['ttd_u', 'pr1', 'pr2'], offdesign=['kA', 'zeta1', 'zeta2'])
	
.. note::

	Some parameters come with characteristic functions based on the design case properties. This means, that e. g. the isentropic efficiency of a turbine is calculated as function of the actual mass flow to design mass flow ratio.
	You can provide your own (measured) data or use the already existing data from TESPy. All standard characteristic functions are available at :py:class:`tespy.components.characteristics.characteristics`. How to specify own data and all available characteristic functions are provided in :ref:`this section <component_characteristics_label>`.

If you want to **prevent the autoswitch from design to offdesign mode** for specific components, use :code:`heat_ex.set_attr(mode='man')`.

For connections it works in the same way, e. g. write

.. code-block:: python

	connection.set_attr(design=['h'], offdesign=['T'])

if you want to replace the enthalpy with the temperature for your offdesign. **The temperature is a result of the design calculation and that value is then used for the offdesign calculation in this example.**

Solving
-------

A TESPy network can be represented as a linear system of nonlinear equations, consequently the solution is obtained with numerical methods.
TESPy uses the n-dimensional Newton–Raphson method to find the systems solution, which may only be found, if the network is parameterized correctly.
**The number of variables n** is :math:`n = num_{conn} \cdot (3 + num_{fluids})`.

The algorithm requires starting values for all variables of the system, thus an initialisation of the system is runned prior to calculating the solution.
**High quality initial values are crutial for convergence speed and stability**, bad starting values might lead to instabilty and diverging calculation can be the result.
Thus there are different levels for the initialisation.

Initialisation
^^^^^^^^^^^^^^

The initialisation is performed in the following steps.

**General preprocessing:**

 * check network consistency and initialise components (if network topology is changed to a prior calculation only),
 * perform design/offdesign switch (for offdesign calculations only)

**Finding starting values:**

 * fluid propagation,
 * fluid property initialisation,
 * initialisation from .csv (preprocessing with :code:`design_path` for offdesign case and setting starting values with :code:`init_path`).

The network check is used to find errors in the network topology, the calulation can not start without a successful check. The component initialisation is important for components using characteristics and the combustion chamber,
a preprocessing of some parameters is required. The preprocessing for the components is performed in the :code:`comp_init` method of the components.
You will find the methods in the :py:mod:`components module <tespy.components.components>`. The design/offdesign switch is described in the network setup section.

**The fluid propagation is a very important step in the initialisation:** Often, you will specify the fluid at one point of the network only, thus all other connections are missing an initial information on the fluid vector,
if you are not using an :code:`init_path`. Also, you do not need to state a starting value for the fluid vector at every point of the network. The fluid propagation will push/pull the specified fluid through the network.
If you are using combustion chambers these will be starting points and a generic flue gas composition will be calculated prior to the propagation.

.. note::
	If the fluid propagation fails, you often experience an error, where the fluid property database can not find a value, because the fluid is 'nan'. Providing starting values manually can fix this problem.

The fluid property initialisation takes the user specified starting values if available and otherwise uses generic starting values on the bases of to which components the connection is linked to.

Last step is the initialisation from :code:`init_path`: For offdesign cases a preprocessing based on the :code:`design_path` in order to recreate the design case and set parameters based on the design case is performed.
If you specified an :code:`init_path` TESPy searches through the connections file for the network topology and if the corresponding connection is found, the starting values for the system variables are extracted from the connections file.
**The files do not need to contain all connections of your network, thus you can build up your network bit by bit and initialise the existing parts of your network from the path.**
**Be aware that a change within the fluid vector does not allow this practice.** Thus, if you plan to use additional fluids in parts of the network you have not touched until now, you will need to state all fluids from the beginning.

.. note::

	Initialisation from a converged calculation usually yields the best performance and is highly receommended.
	In order to initialise your calculation from a path, you need to provide the path to the saved/exported network. If you saved your calculation restults you will find the results in the specified base path './savename/'.


Algorithm
^^^^^^^^^

In this section we will give you an introduction to the implemented solution algorithm.

Newton–Raphson method
+++++++++++++++++++++

The Newton–Raphson method requires the calculation of residual values for the equations and of the partial derivatives to all system variables (jacobian matrix).
In the next step the matrix is inverted and multiplied with the residual vector to calculate the increment for the system variables.
This process is repeated until every equation's result in the system is "correct", thus the residual values are smaller than a specified error tolerance. All equations are of the same structure:

.. math::

	0 = \text{expression}

calculate the residuals

.. math::

	f(\vec{x}_i)

jacobian matrix J

.. math::
	J(\vec{x})=\left(\begin{array}{cccc}
	\frac{\partial f_1}{\partial x_1} & \frac{\partial f_1}{\partial x_2} & \cdots & \frac{\partial f_1}{\partial x_n} \\
	\frac{\partial f_2}{\partial x_1} & \frac{\partial f_2}{\partial x_2} & \cdots & \frac{\partial f_2}{\partial x_n} \\
	\vdots & \vdots & \ddots & \vdots \\
	\frac{\partial f_n}{\partial x_1} & \frac{\partial f_n}{\partial x_2} & \cdots & \frac{\partial f_n}{\partial x_n}
	\end{array}\right)

derive the increment

.. math::
	\vec{x}_{i+1}=\vec{x}_i-J(\vec{x}_i)^{-1}\cdot f(\vec{x}_i)

while

.. math::
	||f(\vec{x}_i)|| > \epsilon

.. note::

	You have to provide the exact amount of required parameters (neither less nor more) and the parametrisation must not lead to linear dependencies.
	Each parameter you set for a connection and each energy flow you specify for a bus will add one equation to your system.
	On top, each component provides a different amount of basic equations plus the equations provided by your component specification.
	For example, setting the power of a pump results in an additional equation compared to a pump without specified power:

.. math::
	\forall i \in \mathrm{network.fluids} \, &0 = fluid_{i,in} - fluid_{i,out}\\
											 &0 = \dot{m}_{in} - \dot{m}_{out}\\
					 \mathrm{additional:} \, &0 = 1000 - \dot{m}_{in} (\cdot {h_{out} - h_{in}})

.. _using_tespy_convergence_check_label:

Convergence stability
+++++++++++++++++++++

One of the main downsides of the Newton–Raphson method is that the initial stepwidth is very large and that it does not know physical boundaries,
for example mass fractions smaller than 0 and larger than 1 or negative pressure. Also, the large stepwidth can adjust enthalpy or pressure to quantities that are not covered by the fluid property databases.
This would cause an inability e. g. to calculate a temperature from pressure and enthalpy in the next iteration of the algorithm. In order to improve convergence stability, we have added a convergence check.

**The convergence check manipulates the system variables after the increment has been added** (if the system variable's value is not user specified). This manipulation has four steps, the first two are always applied:

 * cutting off mass fractions smaller than 0 and larger than 1: This way a mass fraction of a single fluid components never exceeds these boundaries.
 * check, wheather the fluid properties of pure fluids are within the available ranges of CoolProp and readjust the values if not.

The next two steps are applied, if the user did not specify an init_file and the iteration count is lower than 3, thus in the first three iteration steps of the algorithm only. In other cases this convergence check is skipped.

 * Fox mixtures: check, if the fluid properties (pressure, enthalpy and temperature) are within the user specified boundaries (:code:`p_range, h_range, T_range`) and if not, cut off higher/lower values.
 * Check the fluid properties of the connections based on the components they are connecting. E. g. check if the pressure at the outlet of a turbine is lower than the pressure at the inlet or if the flue gas composition at a combustion chamber's
   outlet is within the range of a "typical" flue gas composition. If there are any violations, the corresponding variables are manipulated. If you want to look up, what exactly the convergence check for a specific component does,
   look out for the :code:`convergence_check` methods in the :py:mod:`tespy.components.components module <tespy.components.components>`.

In a lot of different tests the algorithm has found a near enough solution after the third iteration, further checks are usually not required.

**Improve the convergence stability with the :code:`state` keyword for connections:**

It is possible to improve the convergence stability manually when using pure fluids. If you know the fluid's state is liquid or gaseous prior to the calculation, you may provide the according value for the keyword e. g. :code:`myconn.set_attr(state='l')`.
The convergence check manipulates the enthalpy values so that the fluid is always in the desired state at that point. For an example see :ref:`the release information of version 0.1.1 <whats_new_011_example_label>`

Troubleshooting
+++++++++++++++

In this section we show you how you can troubleshoot your calculation and list up common mistakes. If you want to debug your code, make sure to enable tespy.logger and have a look at the log-file at :code:`~/.tespy/` (or at your specified location).

First of all, make sure your network topology is set up correctly, TESPy will prompt an Error, if not.
Also, TESPy will prompt an error, if you did not provide enough or if you provide too many parameters for your calculation, but you will not be given an information which specific parameters are under- or overdetermined.

.. note::
	Always keep in mind, that the system has to find a value for mass flow, pressure, enthalpy and the fluid mass fractions. Try to build up your network step by step and have in mind, what parameters will be determined
	by adding an additional component without any parametrisation. This way, you can easily determine, which parameters are still to be specified.

When using multiple fluids in your network, e. g. :code:`fluids=['water', 'air', 'methane']` and at some point you want to have water only, you still need to specify the mass fractions for both air and methane (although beeing zero) at that point :code:`fluid={'water': 1, 'air': 0, 'methane': 0}`.
Also, setting :code:`fluid={water: 1}, fluid_balance=True` will still not be sufficent, as the fluid_balance parameter adds only one equation to your system.

If you are modeling a cycle, e. g. the clausius rankine cylce, you need to make a cut in the cycle using a sink and a source not to overdetermine the system. Have a look in the :ref:`heat pump tutorial <heat_pump_tutorial_label>`
to understand why this is important.

If you have provided the correct number of parameters in your system and the calculations stops after or even before the first iteration, there are four frequent reasons for that:

 * Sometimes, the fluid property database does not find a specific fluid property in the initialisation process, have you specified the values in the correct unit?
 * Also, fluid property calculation might fail, if the fluid propagation failed. Provide starting values for the fluid composition, especially, if you are using drums, merges and splitters.
 * A linear dependency in the jacobian matrix due to bad parameter settings stops the calculation (overdetermining one variable, while missing out on another).
 * A linear dependency in the jacobian matrix due to bad starting values stops the calculation.

The first reason can be eleminated by carefully choosing the parametrisation. **A linear dependendy due to bad starting values is often more difficult to resolve and it may require some experience.**
In many cases, the linear dependency is caused by equations, that require the **calculation of a temperature**, e. g. specifying a temperature at some point of the network, terminal temperature differences at heat exchangers, etc..
In this case, **the starting enthalpy and pressure should be adjusted in a way, that the fluid state is not within the two-phase region:** The specification of temperature and pressure in a two-phase region does not yield a distict value for the enthalpy.
Even if this specific case appears after some iterations, better starting values often do the trick. Also consider reading :ref:`this <using_tespy_convergence_check_label>`.

Another frequent error is that fluid properties move out of the bounds given by the fluid property database. The calculation will stop immediately. **Adjusting pressure and enthalpy ranges for the convergence check** might help in this case.

.. note::

	If you experience slow convergence or instability within the convergence process, it is sometimes helpful to have a look at the iterinformation. This is printed by default and provides
	information on the residuals of your systems' equations and on the increments of the systems' variables. Maybe it is only one variable causing the instability, thus its increment is much larger
	than the incerement of the other variables.

Did you experience other errors frequently and have a workaround/tips for resolving them? You are very welcome to contact us and share your experience for other users!

Postprocessing
--------------

A postprocessing is performed automatically after the calculation finished. You have two further options:

 * print the results to prompt (:code:`nw.print_results()`) and
 * save the results in a .csv-file (:code:`nw.save('savename')`).

You can print the components and its properties to the prompt and the connections and its properties as well. If you choose to save your results the specified folder will be created containing the information about the network, all connections, busses, components and characteristics.

In order to perform calculations based on your results, you can access all components' and connections' parameters:

For the components this is the way to go

.. code:: python

	eff = mycomp.eta_s.val # isentropic efficiency of mycomp
	s_irr = mycomp.Sirr.val # entropy production of mycomp due to irreveribility

Use this code for connection parameters:

.. code:: python

	mass_flow = myconn.m.val # value in specified network unit
	mass_flow_SI = myconn.m.val_SI # value in SI unit
	mass_fraction_oxy = myconn.fluid.val['O2'] # for the mass fraction of oxygen

TESPy network reader
====================

The network reader is a useful tool to import networks from a datastructure using .csv-files. In order to reimport an exported TESPy network, you must save the network first.

.. code:: python

	nw.save('mynetwork')

This generates a folder structure containing all relevant files defining your network (general network information, components, connections, busses, characteristics) holding the parametrization of that network.
You can reimport the network using following code with the path to the saved documents. The generated network object contains the same information as a TESPy network created by a python script. Thus, it is possible to set your parameters in the .csv-files, too.

.. code:: python

	from tespy import nwkr
	nw = nwkr.load_nwk('path/to/mynetwork')
	
.. note::

	- Imported connections are accessible by the connections' target and target id, e. g.: :code:`nw.imp_conns['condenser:in1']`. 
	- Imported components and busses are accessible by their label, e. g. :code:`nw.imp_comps['condenser']` and :code:`nw.imp_busses['total heat output']` respectively.
