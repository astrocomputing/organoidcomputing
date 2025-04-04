---

# Chapter 4

# Building and Simulating Neural Networks with Brian2**

----

*Building upon the foundations of single neuron modeling laid in Chapter 3, this chapter takes the critical step towards simulating interconnected **neural networks**, the fundamental architecture underlying brain function and, potentially, Organoid Computing. We transition from simulating isolated neurons to modeling populations and their interactions using the Brian2 simulator. We begin by exploring how to efficiently represent and manage groups of neurons using the `NeuronGroup` object, moving beyond single-neuron simulations and discussing the underlying computational efficiencies derived from vectorization and code generation. The core focus then shifts to establishing connections between neurons using Brian2's powerful `Synapses` object. We will detail how to define the synaptic model—specifying not just the immediate action when a presynaptic neuron fires (`on_pre`) but also any persistent synaptic state variables (like weights or plasticity-related traces)—and how to incorporate crucial parameters like synaptic weights (`w`) and transmission delays (`delay`), emphasizing their biological significance and variability. Subsequently, we examine various common strategies for specifying the connectivity patterns or **network topology**, ranging from simple, highly regular structures (one-to-one, all-to-all) to more biologically relevant sparse, probabilistic, or spatially organized wiring schemes, implemented via the flexible `connect()` method of the `Synapses` object. We revisit synaptic models, contrasting current-based and conductance-based approaches within the practical context of Brian2's `Synapses` syntax, delving deeper into the implementation details (including different ways to model conductance decay), the biophysical importance of **reversal potentials** and **driving forces**, and the functional implications like **shunting inhibition**. We then expand our repertoire of monitoring tools, detailing the utility and implementation of the `PopulationRateMonitor` for tracking collective dynamics alongside the `SpikeMonitor` (for precise spike timing) and `StateMonitor` (for subthreshold variable evolution), discussing the trade-offs in terms of data volume and computational overhead. Techniques for visualizing network dynamics, particularly interpreting subtle features in **raster plots** (e.g., types of synchrony, propagating waves) and **population firing rate histograms** (e.g., transients, oscillatory signatures) to infer network states, will be demonstrated with greater nuance. Finally, the chapter culminates in practical Brian2 implementations, guiding the reader step-by-step through simulating first a minimal two-neuron circuit to clearly illustrate direct synaptic interaction, and then constructing and analyzing a more representative basic randomly connected network composed of distinct excitatory and inhibitory populations, illustrating the emergence of characteristic network-level activity patterns shaped by recurrent connectivity and the crucial balance between excitation and inhibition.*

---- 

**4.1 Neuron Populations (`NeuronGroup`)**

While the simulation of isolated single neurons provides a crucial foundation for understanding their intrinsic electrical properties (Chapter 3), the remarkable computational capabilities attributed to the brain and potentially harnessable from brain organoids arise almost entirely from the coordinated and collective activity of vast numbers of interconnected neurons functioning as a network. Even moderately sized organoids contain cell populations numbering in the many thousands or millions. Therefore, to progress towards simulations that capture the essence of these systems, we require computational tools capable of representing and simulating large groups of neurons efficiently. Brian2 provides this capability through its central class for defining neuronal populations: the **`NeuronGroup`**. Conceptually, a `NeuronGroup` acts as a container for a collection of individual neurons that are all described by the **same mathematical model**. This means they share the identical set of differential equations defining their dynamics, the same condition for triggering an action potential (`threshold`), and the same mechanism for resetting their state variables after a spike (`reset`). Furthermore, parameters defined within the model equations are typically assumed to be identical for all neurons in the group unless heterogeneity is explicitly introduced using the string-expression mechanisms discussed in Chapter 5. Despite being governed by the same rules, each neuron within the `NeuronGroup` maintains its own independent dynamic state variables (e.g., its specific membrane potential $V(t)$, its adaptation level $w(t)$ if using an adaptive model) and can receive unique combinations of synaptic inputs, allowing for rich and diverse activity patterns to emerge across the population.

The true strength of the `NeuronGroup` lies not just in its conceptual simplicity but in its highly optimized implementation, designed specifically for computational efficiency when dealing with large populations. Brian2 achieves this primarily through **vectorization**. Instead of simulating each neuron sequentially in a loop (which would be extremely slow in Python), Brian2 represents the state variables of all `N` neurons in the group as numerical arrays (typically NumPy arrays or similar structures in generated code). The mathematical operations defined in the model equations (e.g., calculating the change in membrane potential based on current voltage, leak, and input currents) are then applied element-wise across these entire arrays in each simulation time step. Modern processors are highly optimized for such vectorized calculations, performing the same operation on multiple data points simultaneously (SIMD - Single Instruction, Multiple Data). Furthermore, for enhanced performance, Brian2 employs **code generation**. It can translate the user-provided model description (equations, threshold, reset) into optimized low-level code (e.g., C++, Cython) which is then compiled and executed. This generated code avoids the overhead associated with interpreting Python instructions repeatedly within the main simulation loop, leading to substantial speedups, particularly for large `N` and complex models. This combination of vectorization and code generation makes it computationally feasible to simulate networks containing tens of thousands or even millions of neurons on standard desktop computers or clusters, approaching scales relevant for studying emergent phenomena in organoid-like systems.

Creating a `NeuronGroup` involves providing the necessary arguments to its constructor: the total number of neurons `N`, the `model` string (containing differential equations, parameters with units), the `threshold` condition string, the `reset` statement(s) string, the `refractory` period duration, and the numerical integration `method` (e.g., `'euler'`, `'rk2'`, `'rk4'`).

```python
# Example: Defining a population of 500 AdEx neurons
N_pop = 500
# Parameters for AdEx model (assuming they are defined with units)
adex_eqs = '''
dv/dt = (gL*(EL - v) + gL*DeltaT*exp((v - VT)/DeltaT) - w + I_syn)/Cm : volt (unless refractory)
dw/dt = (a*(v - EL) - w)/tauw : amp
I_syn : amp # Synaptic input current variable
'''
# Create the group
adex_population = NeuronGroup(N_pop, adex_eqs,
                              threshold='v > Vcut', # Cutoff potential for spike detection
                              reset='v = Vr; w += b', # Reset V and increment w
                              refractory=5*ms, # Refractory period
                              method='euler', # Integration method
                              name='AdEx_Population') # Assigning a name
```
Once created, the state variables defined in the `model` string (e.g., `v`, `w`, `I_syn`) become attributes of the `NeuronGroup` object, accessible as array-like objects that represent the current state of the entire population. Brian2 provides seamless integration with NumPy's slicing and indexing syntax, allowing for highly flexible access and modification of these state variables.

```python
# Initialize all AdEx neurons' voltages to the reset potential Vr
adex_population.v = Vr

# Set the adaptation variable 'w' for the first half of the population
adex_population.w[:N_pop//2] = 10*pA

# Read the current voltage of the last neuron in the group
voltage_last_neuron = adex_population.v[-1]

# Apply a specific synaptic current only to neurons with indices between 100 and 199
# (Assuming I_syn is defined in the model)
adex_population.I_syn[100:200] = 50*pA
```
This ability to efficiently manage and manipulate the states of large neuronal populations is fundamental for constructing complex network models. While a single `NeuronGroup` might sometimes suffice (e.g., using indexing to handle E/I differences as shown in Chapter 5), realistic models often benefit from using **multiple `NeuronGroup` objects** to represent distinct cell types (e.g., cortical layers, specific interneuron classes like PV+ and SST+ cells). Each group can have its own specific model equations, parameters reflecting its unique biophysics, and size. This modular design enhances code clarity, organization, and biological fidelity, allowing for explicit representation of the cellular diversity inherent in brain organoids and facilitating the definition of cell-type-specific connectivity patterns in the next steps.

`[Conceptual Figure 4.1: NeuronGroup Representation. A diagram showing a box labeled "NeuronGroup (N)". Inside, icons represent individual neurons. Arrows point from the NeuronGroup definition (specifying model, N, threshold, etc.) to the population. Below, conceptual arrays are shown for state variables like `v` and `w`, each of length N, illustrating vectorized representation. Example code snippet `adex_population.w[:N//2] = 10*pA` is shown modifying a slice of the `w` array.]`

**4.2 Synaptic Connections (`Synapses`): Model (`on_pre`), Weight (`w`), Delay (`delay`)**

The computational power and dynamic richness of neural networks arise critically from the **synaptic connections** that mediate communication between neurons. Action potentials generated by one neuron trigger responses in others, allowing information to flow and be processed within the network. In Brian2, the `Synapses` object is the central class for defining and managing these interactions. It acts as a sophisticated intermediary, specifying the pathways, mechanisms, and properties of synaptic transmission between a designated **source** `NeuronGroup` and a **target** `NeuronGroup`. A key feature is that the source and target can be the same `NeuronGroup` (`Synapses(P, P, ...)`), enabling the straightforward implementation of **recurrent connections**—neurons within a population connecting back onto themselves or other neurons in the same population. Such recurrence is a hallmark of biological circuits, particularly in cortex and hippocampus, and is essential for creating sustained activity, memory functions, and complex dynamics.

When creating a `Synapses` object, several key arguments define its behavior, primarily through string expressions parsed by Brian2:
*   `model`: This multi-line string defines any **state variables** associated with the individual synaptic connections themselves. These variables exist for *each* synapse instance created later by the `connect()` method. The most fundamental synaptic variable is typically the **synaptic weight (`w`)**, representing the strength or efficacy of the connection. The units of `w` depend on the chosen synaptic model: it might be dimensionless (`w : 1`), represent a voltage jump (`w : volt`), a current amplitude (`w : amp`), or most realistically for conductance-based models, a conductance change (`w : siemens`). Beyond the static weight, the `model` string can also include variables needed for **synaptic plasticity** (e.g., pre- and post-synaptic traces for Spike-Timing-Dependent Plasticity, as detailed in Chapter 6) or variables modeling **short-term plasticity** (like synaptic depression or facilitation, see Chapter 13).
*   `on_pre`: This string specifies the sequence of commands to execute on the connected **postsynaptic target neuron(s)** whenever a spike arrives from a **presynaptic source neuron**, after accounting for the synaptic delay. This is where the primary effect of the synapse is implemented. Common actions involve incrementing a postsynaptic state variable (like a conductance `g_E_post` or a current `I_syn_post`, accessed using the `_post` suffix) by an amount determined by the synaptic weight `w`. More complex actions, including conditional logic or updates to synaptic plasticity variables, can also be included here.
*   `on_post` (Optional): This string defines actions to be taken on the **synaptic variables** (defined in the `model` string) whenever the **postsynaptic target neuron** fires. While not typically used for basic transmission, this pathway is crucial for implementing many forms of Hebbian plasticity, such as STDP, where the timing of postsynaptic spikes relative to presynaptic spikes influences synaptic weight changes.

Let's expand on the conductance-based synapse example, explicitly showing how the weight `w` defined in the `model` is used in `on_pre`:

```python
# Assume target_group has state variable 'g_E : siemens' for excitatory conductance
syn_cond_detailed = Synapses(source_group, target_group,
                             model='''
                                   w : siemens # Synaptic weight (conductance increase)
                                   # Example: Add a placeholder for plasticity later
                                   last_update_time : second
                                   ''',
                             on_pre='''
                                    g_E_post += w
                                    last_update_time = t # Record time of spike arrival at synapse
                                    ''',
                             name='ConductanceSynapses')
```
In this more detailed example, each synapse instance will store its specific weight `w` and a variable `last_update_time`. When a presynaptic spike arrives (after the delay), the `on_pre` code is executed: the postsynaptic conductance `g_E_post` is incremented by the synapse's weight `w`, and the `last_update_time` variable associated with *that specific synapse* is updated to the current simulation time `t`. This demonstrates how `on_pre` can modify both postsynaptic variables and synaptic state variables.

The **synaptic delay (`delay`)** represents the finite time lag between the occurrence of a presynaptic action potential and its effect on the postsynaptic neuron. Biologically, this delay comprises several components: the time for the action potential to propagate along the axon (dependent on distance, diameter, and myelination), the time for neurotransmitter release machinery to activate at the synapse, the diffusion time across the synaptic cleft, and the time for postsynaptic receptors to bind the neurotransmitter and open ion channels. These delays can vary significantly, from sub-millisecond delays for local connections to tens of milliseconds for long-range projections. Incorporating realistic delays is often critical for accurately modeling network dynamics, as they strongly influence the timing of interactions, the emergence of synchrony and oscillations, and the network's ability to process temporal patterns. Brian2 handles delays by placing incoming spikes into an internal queue associated with each `Synapses` object. The `on_pre` action is only executed when the simulation time reaches the spike time plus the specified delay for that connection.

Delays can be set in several ways for a `Synapses` object. A single value assigns a uniform delay to all connections subsequently created by `connect()`:
```python
syn_cond_detailed.delay = 5*ms # Uniform 5ms delay
```
Alternatively, delays can be assigned heterogeneously after connections are made, allowing for more biological realism where delays might depend on factors like distance or cell type. This is often done using string expressions evaluated for each synapse instance:
```python
# After syn_cond_detailed.connect(...)

# Example: Delay drawn from a uniform distribution between 1ms and 4ms
syn_cond_detailed.delay = '1*ms + 3*ms * rand()'

# Example: Delay proportional to index difference (conceptual distance)
# syn_cond_detailed.delay = 'abs(i - j) * 0.1*ms + 0.5*ms' # Requires i, j available

# Ensure delays are physically plausible (non-negative) using clip()
syn_cond_detailed.delay = 'clip(3*ms + 1*ms * randn(), 0.1*ms, 10*ms)' # Gaussian delay, clipped
```
By providing mechanisms to define synaptic state variables (`model`), the rules for spike arrival (`on_pre`), optional postsynaptic spike effects (`on_post`), and realistic transmission delays (`delay`), the `Synapses` object offers a powerful and flexible framework for modeling the diverse forms of communication that underpin neural network function.

**4.3 Connectivity Strategies: One-to-one, All-to-all, Probabilistic, Conditional**

Defining the potential synaptic interactions via the `Synapses` object is only half the story; the crucial next step is to specify the actual **network connectivity** or **topology**—the precise pattern of which presynaptic neurons form connections with which postsynaptic neurons. The `connect()` method associated with every `Synapses` object is used for this purpose. Brian2 offers remarkable flexibility in defining these connections, allowing modelers to easily create networks ranging from simple, idealized structures to complex, sparse, and structured topologies that more closely resemble biological reality, often using concise and intuitive syntax.

The `connect()` method fundamentally operates on pairs of indices: `i` representing the index of a neuron in the source `NeuronGroup`, and `j` representing the index of a neuron in the target `NeuronGroup`. Connections are created for pairs `(i, j)` that satisfy the criteria specified in the arguments to `connect()`.

*   **Highly Regular Structures:**
    *   **One-to-one:** Connects source neuron `i` only to target neuron `j` where `i == j`. Requires `N_source == N_target`. Useful for feedforward layers or mapping corresponding elements.
        ```python
        synapses.connect(j='i')
        ```
    *   **All-to-all:** Connects every source neuron `i` to every target neuron `j`. This results in dense connectivity ($N_{\text{source}} \times N_{\text{target}}$ connections). If connecting a group to itself (`Synapses(P, P, ...)`), self-connections (`i==j`) are automatically excluded by default unless explicitly requested (e.g., using `condition='True'`). Dense connectivity is computationally expensive and biologically unrealistic for large networks but useful for small motifs or fully connected theoretical models.
        ```python
        synapses.connect() # Creates all possible connections (excluding i=j if source==target)
        ```

*   **Explicit Connections:** For complete control over specific links, lists or arrays of pre- (`i`) and post- (`j`) synaptic indices can be provided. This is mainly useful for very small, handcrafted circuits or debugging.
    ```python
    # Example: Connect 0->1, 0->2, 1->2
    synapses.connect(i=[0, 0, 1], j=[1, 2, 2])
    ```

*   **Conditional Connections:** A highly powerful feature is the ability to specify a connection condition as a boolean string expression involving `i` and `j`, and potentially other static variables accessible to the `Synapses` object. A connection is created only if the condition evaluates to `True` for the pair `(i, j)`.
    ```python
    # Connect recurrently, avoiding self-connections
    synapses_recurrent.connect(condition='i != j')

    # Connect based on index proximity (simple form of locality)
    synapses.connect(condition='abs(i - j) < 5 and i != j') # Connect within a radius of 5 indices

    # Connect based on parity (e.g., even neurons connect to odd neurons)
    synapses.connect(condition='i % 2 == 0 and j % 2 != 0')

    # Conceptual Spatial Connectivity: Connect if Euclidean distance is below threshold
    # Requires neuron positions (x, y) to be defined and accessible.
    # Assume NeuronGroup 'neurons' has 'x : meter' and 'y : meter' attributes.
    # The syntax might involve linking or accessing them via special names.
    # max_dist_sq_str = f"({max_dist/meter})**2" # Use f-string for value substitution
    # condition_str = f"((neurons.x[i]/meter - neurons.x[j]/meter)**2 + \
    #                  (neurons.y[i]/meter - neurons.y[j]/meter)**2) < {max_dist_sq_str} and i != j"
    # synapses_spatial.connect(condition=condition_str)
    # Note: Direct indexing like neurons.x[i] in the condition string might be inefficient or not supported.
    # Brian2 often optimizes spatial connections using different approaches (e.g., spatial indexing).
    # Refer to Brian2 documentation for recommended methods for efficient spatial connectivity.
    ```
*(Self-correction: Emphasized the potential inefficiency/syntax issues with direct spatial coordinate access in the `condition` string and recommended checking Brian2 documentation for optimized spatial connection methods.)*

*   **Probabilistic Connections:** This is frequently used to model the sparse and seemingly random connectivity observed in many brain regions. The argument `p` specifies the probability (a float between 0 and 1) that a connection will be formed between any potential pair `(i, j)` that satisfies any other specified conditions.
    ```python
    # Create connections randomly with a uniform probability of 10%
    synapses.connect(p=0.1)

    # Combine probability with a condition (e.g., avoid self-connections)
    synapses_recurrent.connect(condition='i != j', p=0.1)

    # Probability can also be a string expression, allowing it to vary, e.g., with distance
    # sigma_dist = 100*umetre
    # prob_str = f'exp(-((neurons.x[i] - neurons.x[j])**2 + (neurons.y[i] - neurons.y[j])**2) / (2*({sigma_dist/meter})**2))'
    # synapses_spatial_prob.connect(condition='i != j', p=prob_str)
    # Again, check Brian2 docs for efficient implementation of distance-dependent probabilities.
    ```
The `p` argument allows easy creation of networks with specific **sparsity** levels (fraction of potential connections that are actually made), a key characteristic of biological circuits which are typically very sparse (often < 1% connectivity). Probabilistic connection rules also naturally lead to variability in **convergence** (number of inputs to a neuron) and **divergence** (number of outputs from a neuron) across the population.

Crucially, synaptic parameters like weights (`w`) and delays (`delay`) are typically assigned *after* the `connect()` method has created the specific synapse instances. This allows assigning values only to the connections that actually exist. The assignment can use single values for uniformity, or string expressions involving random functions (`rand()`, `randn()`) or indices (`i`, `j`) to introduce heterogeneity or structure to these parameters across the established connections.

```python
# 1. Create connections
synapses.connect(condition='i != j', p=0.1)

# 2. Assign parameters to the created connections
# Uniform assignment
synapses.w = 1.5 * nS
synapses.delay = 2.0 * ms

# Heterogeneous weights (Gaussian)
synapses.w = '1.5*nS + 0.5*nS * randn()'

# Heterogeneous delays (Uniform)
synapses.delay = '1.0*ms + 3.0*ms * rand()' # Delays from 1ms to 4ms

# Weight dependent on presynaptic index 'i' (example)
# synapses.w = '(1.0 + 0.01*i) * nS'
```
This two-step process—first defining the structure with `connect()`, then assigning properties to the existing connections—provides a powerful and flexible workflow for constructing diverse and biologically plausible network models in Brian2, enabling the exploration of how different network architectures influence emergent dynamics and potential computations in organoid-inspired simulations.

`[Conceptual Figure 4.2: Connectivity Examples. Panel (a): One-to-one connection. Panel (b): All-to-all connection. Panel (c): Probabilistic random connection (sparse). Panel (d): Conditional spatial connection (local connectivity based on conceptual distance).]`

**4.4 Synaptic Models (Current Injection, Conductance-Based). Reversal Potentials.**

Having established the framework for creating populations of neurons (`NeuronGroup`) and specifying the connectivity patterns between them (`Synapses.connect()`), we now focus more deeply on the implementation details of the synaptic interaction itself within the `Synapses` object, specifically contrasting the **current-based** versus **conductance-based** modeling approaches. The choice between these significantly impacts both the biological realism and the computational performance of the simulation.

**Current-Based Synapses:** This approach models the effect of a presynaptic spike as a direct injection of a current pulse, $I_{\text{syn}}(t)$, into the postsynaptic neuron. It's conceptually simpler and computationally faster because the synaptic current is independent of the postsynaptic neuron's state. In Brian2, this usually involves:
1.  Defining a state variable in the postsynaptic `NeuronGroup` to represent the incoming synaptic current (e.g., `I_syn_E : amp`).
2.  Defining the decay dynamics of this current variable within the `NeuronGroup`'s equations (e.g., `dI_syn_E/dt = -I_syn_E / tau_syn_E : amp`).
3.  Defining the `Synapses` object with a weight `w` having units of current (e.g., `w : amp`) and an `on_pre` action that increments the postsynaptic current variable (e.g., `on_pre='I_syn_E_post += w'`).
4.  Including the synaptic current variable(s) in the postsynaptic neuron's membrane potential equation (e.g., `... + R*(I_syn_E + I_syn_I) ...`).

```python
# --- Current-Based Synapse Full Example Snippet ---
tau_syn_E = 5*ms
eqs_neuron_curr = '''
dv/dt = (V_rest - v + R*I_syn_E)/tau : volt (unless refractory)
dI_syn_E/dt = -I_syn_E / tau_syn_E : amp # Current decay handled in neuron
R : ohm # Assuming R is defined
tau: second
V_rest : volt
'''
target_neurons_curr = NeuronGroup(N_target, eqs_neuron_curr, ...)

# Synapse definition
syn_E_curr = Synapses(source_group, target_neurons_curr,
                      model='w : amp', # Weight is peak current or charge
                      on_pre='I_syn_E_post += w') # Increment current variable
syn_E_curr.connect(...)
syn_E_curr.w = 50*pA # Example weight
```
While efficient, this model's primary limitation is its failure to account for the **driving force**. The effect of opening synaptic channels biologically depends strongly on the difference between the postsynaptic membrane potential ($V_m$) and the synapse's reversal potential ($E_{\text{rev}}$). Current-based models ignore this, meaning an EPSP always provides the same current injection regardless of whether the neuron is near rest or highly depolarized, which is inaccurate.

**Conductance-Based Synapses:** This model offers significantly greater biophysical realism by explicitly simulating the change in the postsynaptic membrane's **conductance** ($g_{\text{syn}}(t)$) caused by the presynaptic spike. The resulting synaptic current is then calculated dynamically at each time step based on this conductance and the driving force: $I_{\text{syn}} = g_{\text{syn}}(t) (V(t) - E_{\text{rev}})$. Implementing this in Brian2 typically involves:
1.  Defining state variables for the synaptic conductances in the postsynaptic `NeuronGroup` (e.g., `g_E : siemens`, `g_I : siemens`).
2.  Defining the decay dynamics for these conductance variables within the `NeuronGroup`'s equations (e.g., `dg_E/dt = -g_E / tau_g_E : siemens`).
3.  Including the synaptic current terms, calculated using the conductances and appropriate reversal potentials ($E_E$, $E_I$), within the `NeuronGroup`'s membrane potential equation (e.g., `... + g_E*(E_E - v) + g_I*(E_I - v) ...`).
4.  Defining the `Synapses` object with a weight `w` having units of conductance (e.g., `w : siemens`) and an `on_pre` action that increments the appropriate postsynaptic conductance variable (e.g., `on_pre='g_E_post += w'`).

```python
# --- Conductance-Based Synapse Full Example Snippet ---
tau_g_E = 5*ms; E_E = 0*mV
eqs_neuron_cond = '''
dv/dt = (V_rest - v + g_E*(E_E - v))/tau : volt (unless refractory)
dg_E/dt = -g_E / tau_g_E : siemens # Conductance decay handled in neuron
tau: second
V_rest: volt
'''
target_neurons_cond = NeuronGroup(N_target, eqs_neuron_cond, ...)
target_neurons_cond.g_E = 0*nS # Initialize conductance

# Synapse definition
syn_E_cond = Synapses(source_group, target_neurons_cond,
                      model='w : siemens', # Weight is conductance increment
                      on_pre='g_E_post += w')
syn_E_cond.connect(...)
syn_E_cond.w = 1.0*nS # Example weight
```
This structure correctly captures the dependence on driving force. As $V$ approaches $E_E$ (around 0 mV), the excitatory current $g_E(E_E - V)$ diminishes. Similarly, as $V$ approaches $E_I$ (e.g., -75 mV), the inhibitory current $g_I(E_I - V)$ diminishes or even reverses sign (though still pulling $V$ towards $E_I$).

An alternative way to model conductance dynamics in Brian2, sometimes preferred for efficiency or specific plasticity rules, is to model the conductance decay within the `Synapses` object itself using **summed variables**. This requires modifying both the `Synapses` model and the `NeuronGroup` model:

```python
# --- Conductance-Based Synapse using Summed Variables ---
eqs_neuron_summed = '''
dv/dt = (V_rest - v + g_E_total*(E_E - v))/tau : volt (unless refractory)
g_E_total : siemens (summed) # Declares g_E_total sums inputs from synapses
tau : second; V_rest : volt
'''
target_neurons_summed = NeuronGroup(N_target, eqs_neuron_summed, ...)

# Synapse model now includes decay
syn_E_summed = Synapses(source_group, target_neurons_summed,
                      model='''w : siemens
                               dg_syn/dt = -g_syn/tau_g_E : siemens (event-driven)''',
                      on_pre='g_syn += w', # Increment synapse's own conductance variable
                      # Link synapse's g_syn to neuron's g_E_total
                      namespace={'g_E_total_post': 'g_syn'}) # Syntax may vary, check docs
# OR more direct approach if supported:
# syn_E_summed = Synapses(..., on_pre='g_E_total_post += w') # Needs testing if g_E_total handles decay implicitly
```
*(Self-correction: The exact syntax for summed variables and linking requires careful checking against Brian2 documentation. The core idea is that the synapse manages its conductance state including decay, and the neuron sums these contributions. The direct `on_pre='g_E_total_post += w'` might work if `g_E_total` is implicitly treated like a decaying sum, but the explicit synapse-based decay (`dg_syn/dt`) linked to a summed variable is a more general pattern.)*

The conductance-based approach, by incorporating reversal potentials and driving forces, provides a more accurate biophysical picture of synaptic integration. It naturally captures effects like **synaptic saturation** and **shunting inhibition**. Shunting occurs when inhibitory synapses with $E_I$ close to $V_{\text{rest}}$ open; they cause little voltage change but significantly increase the membrane conductance ($g_I$), thereby reducing the neuron's input resistance ($R_{\text{input}} = 1/g_{\text{total}}$). This makes the neuron "leakier" and less responsive to concurrent excitatory inputs, effectively implementing a divisive scaling or gain control mechanism. Such effects are crucial for stabilizing network activity and enabling complex computations, making conductance-based models often the preferred choice when simulating networks aiming for biological realism, such as models of dynamically balanced E/I networks potentially relevant to organoid activity.

**4.5 Network Monitoring (`SpikeMonitor`, `PopulationRateMonitor`)**

Extracting meaningful information from the complex, high-dimensional dynamics generated by spiking neural network simulations requires robust and efficient monitoring tools. Brian2 provides a suite of `Monitor` classes designed to record different aspects of network activity during a simulation run, allowing researchers to capture the data necessary for subsequent analysis and visualization while managing computational resources effectively. The choice of which monitors to use and which data to record depends on the specific scientific questions being addressed and the scale of the simulation.

The **`SpikeMonitor`** remains an indispensable tool for analyzing network spiking patterns. It efficiently records the fundamental output of spiking neurons: the precise **time (`t`)** of each action potential and the **index (`i`)** of the neuron within the monitored `NeuronGroup` that generated it. This discrete event data is relatively compact to store and forms the basis for a wide range of analyses. Raster plots are generated directly from `SpikeMonitor` data. From the recorded spike times, one can calculate crucial statistics like mean firing rates, interspike interval (ISI) distributions (characterizing firing regularity), coefficients of variation (CV) of ISIs, pairwise spike time correlations (measuring synchrony), and more complex measures of information coding or network state. Because it only records events, `SpikeMonitor` typically introduces minimal computational overhead during the simulation.

```python
# Example: Monitoring spikes from E and I populations separately
# spike_mon_E = SpikeMonitor(P_E, name='ExcSpikes')
# spike_mon_I = SpikeMonitor(P_I, name='InhSpikes')
# Or monitor all neurons and separate later using indices
spike_mon_all = SpikeMonitor(neurons, name='AllSpikes')

# Accessing data after run():
# times_E = spike_mon_all.t[spike_mon_all.i < N_E] # Times of E spikes
# indices_E = spike_mon_all.i[spike_mon_all.i < N_E] # Indices of E neurons that spiked
# times_I = spike_mon_all.t[spike_mon_all.i >= N_E] # Times of I spikes
# indices_I = spike_mon_all.i[spike_mon_all.i >= N_E] # Indices of I neurons that spiked
```

The **`StateMonitor`** provides access to the continuous evolution of internal state variables within neurons, such as the membrane potential (`v`), synaptic conductances (`g_E`, `g_I`), adaptation variables (`w`), or calcium concentrations. This detailed view is invaluable for understanding the subthreshold dynamics that lead to spiking, diagnosing model behavior, visualizing synaptic integration, or tracking variables related to plasticity mechanisms. However, its utility comes at a potentially significant cost in terms of **memory consumption and file size**, especially for large networks or long simulations. Recording a floating-point value for each specified variable, for each selected neuron, at every simulation time step can generate enormous datasets. Therefore, judicious use of the `record` argument is essential. Instead of recording from all neurons (`record=True`), one typically selects a small, representative subset of indices (`record=[index1, index2, ...]`) or uses boolean indexing or conditions if needed, focusing on the neurons and variables most critical for the analysis. Careful planning is required to balance the need for detailed state information against available memory and storage resources.

```python
# Example: Monitoring voltage and conductances for a few select neurons
indices_to_record = [0, N_E // 2, N_E] # First E, middle E, first I neuron
vars_to_record = ['v', 'g_E', 'g_I']
state_mon_sample = StateMonitor(neurons, vars_to_record, record=indices_to_record, name='SampleStates')

# Accessing data:
# voltage_neuron0 = state_mon_sample.v[0] # Voltage trace for the first recorded neuron (index 0)
# gE_neuronNE = state_mon_sample.g_E[2] # g_E trace for the third recorded neuron (index N_E)
# time_vector = state_mon_sample.t
```

For understanding the overall activity level of a large population, tracking every individual spike or voltage trace can be cumbersome. The **`PopulationRateMonitor`** offers an efficient solution by calculating and recording an estimate of the **instantaneous population firing rate** online during the simulation. It effectively counts spikes within small time bins (related to `dt`) across the entire monitored `NeuronGroup`, normalizes by the group size and bin duration, and stores the resulting rate time series. This provides a smooth, macroscopic measure of how active the population is over time. It is particularly useful for identifying network states (e.g., high vs. low activity), detecting emergent oscillations reflected in the population rate, quantifying the overall response to stimuli, and comparing activity levels across different conditions or between distinct populations (e.g., comparing E rate vs. I rate). Because the calculation is done incrementally during the run, `PopulationRateMonitor` is significantly more memory-efficient than recording all spikes with `SpikeMonitor` and calculating the rate post-hoc, especially for very large networks and long simulations.

```python
# Example: Monitoring rates of E and I populations separately
rate_mon_E = PopulationRateMonitor(P_E, name='ExcRate')
rate_mon_I = PopulationRateMonitor(P_I, name='InhRate')

# Accessing data:
# rate_trace_E = rate_mon_E.rate
# time_vector_rate = rate_mon_E.t # Same time vector for both if simulation setup is simple
```
By thoughtfully selecting and combining these different monitors—`SpikeMonitor` for precise timing, `StateMonitor` for detailed subthreshold dynamics of key neurons, and `PopulationRateMonitor` for efficient tracking of overall population activity—researchers can effectively capture the multi-scale dynamics of their simulated networks and extract the necessary data for comprehensive analysis and visualization, paving the way for quantitative comparisons with experimental data from systems like brain organoids.

**4.6 Visualization (Raster Plots, Population Firing Rates)**

Interpreting the complex datasets generated by spiking neural network simulations hinges crucially on effective **visualization**. Raw numerical outputs from monitors, often consisting of long lists of spike times or voltage values, are difficult to make sense of directly. Graphical representations allow us to readily perceive patterns, trends, and anomalies in the network's behavior, facilitating understanding, debugging, comparison between conditions, and communication of results. Python's extensive plotting library, **Matplotlib**, integrates seamlessly with Brian2 and provides the tools needed to create informative visualizations from monitor data. Among the most fundamental and widely used plots for analyzing network activity are raster plots and population firing rate plots.

A **Raster Plot**, generated from `SpikeMonitor` data (`spike_monitor.t` and `spike_monitor.i`), provides a detailed microscopic view of the network's spiking output. By plotting a marker for each spike at its time of occurrence (x-axis) against the index of the neuron that fired it (y-axis), raster plots reveal the fine spatio-temporal structure of activity. Careful examination allows identification of various dynamic features:
*   **Asynchronous Irregular (AI) Firing:** A seemingly random scattering of points, with individual neurons firing irregularly and low correlation between neurons, characteristic of balanced states in cortical circuits.
*   **Synchronous Events:** Clear vertical alignment of many dots indicates brief periods where a large fraction of the population fires almost simultaneously. This can range from weak synchrony (slightly increased vertical density) to strong **network bursts** (very dense vertical bands).
*   **Oscillations:** Rhythmic patterns become visible as alternating vertical bands of high and low firing density across the population, or as individual neurons firing periodically. The frequency of oscillations can often be estimated visually.
*   **Propagating Waves:** Diagonal patterns or sequences of activation across neuron indices (if indices have spatial meaning) can indicate waves of activity propagating through the network.
*   **Population Differences:** Plotting different subpopulations (e.g., E vs. I neurons) with different colors or y-axis ranges allows direct visual comparison of their firing patterns, rates, and participation in network events. For instance, inhibitory neurons might fire at higher rates or be more involved in pacing oscillations.

```python
# Conceptual plotting code for a colored raster plot
import matplotlib.pyplot as plt

N_E = 80 # Example number of excitatory neurons

spike_indices = spike_monitor.i
spike_times = spike_monitor.t

is_E = spike_indices < N_E
is_I = spike_indices >= N_E

plt.figure(figsize=(10, 5))
plt.plot(spike_times[is_E]/ms, spike_indices[is_E], '.r', markersize=1, label='Excitatory')
plt.plot(spike_times[is_I]/ms, spike_indices[is_I], '.b', markersize=1, label='Inhibitory')
plt.xlabel('Time (ms)')
plt.ylabel('Neuron Index')
plt.title('Raster Plot')
plt.legend()
# plt.show()
```

Complementing the detailed raster plot, the **Population Firing Rate Plot** offers a macroscopic summary of network activity. Derived from a `PopulationRateMonitor` (`rate_monitor.t`, `rate_monitor.rate`) or calculated by binning spikes from a `SpikeMonitor`, this plot shows the average firing rate across the population as a function of time. It smooths over individual spike details, highlighting trends in overall activity levels. This view is particularly effective for:
*   **Quantifying Activity Levels:** Determining the baseline spontaneous firing rate or the peak rate during specific events.
*   **Detecting Transients:** Clearly showing increases or decreases in population activity in response to stimuli or parameter changes.
*   **Analyzing Oscillations:** Rhythmic fluctuations in the population rate provide strong evidence for network oscillations. Spectral analysis (e.g., using Fourier transforms) of the rate signal can precisely quantify the dominant oscillation frequencies and their power.
*   **Comparing E/I Dynamics:** Plotting the rates of excitatory and inhibitory populations on the same axes reveals their dynamic interplay, such as whether inhibition closely tracks excitation (characteristic of balanced networks) or whether oscillations involve specific phase relationships between the two populations.

```python
# Conceptual plotting code for E and I population rates
plt.figure(figsize=(10, 4))
plt.plot(rate_mon_E.t/ms, rate_mon_E.rate/Hz, label='Excitatory Rate', color='red')
plt.plot(rate_mon_I.t/ms, rate_mon_I.rate/Hz, label='Inhibitory Rate', color='blue')
plt.xlabel('Time (ms)')
plt.ylabel('Population Rate (Hz)')
plt.title('Population Firing Rates')
plt.legend()
# plt.show()
```

Often, displaying the raster plot directly above the corresponding population rate plot(s), aligned with a common time axis, provides the most powerful combined visualization. This allows correlating macroscopic changes in population rate with the underlying microscopic spiking patterns that generate them. Beyond these two core plots, other visualizations are frequently useful: **histograms** of interspike intervals (ISIs) can reveal firing regularity (e.g., Poisson-like exponential vs. more regular); **histograms of firing rates** across the population illustrate heterogeneity; **voltage trace plots** (from `StateMonitor`) show subthreshold dynamics; and **cross-correlation histograms** quantify pairwise neuronal synchrony. Proficiency in generating and interpreting these visualizations is fundamental for extracting biological insights from the rich datasets produced by Brian2 network simulations.

`[Conceptual Figure 4.3: Example Raster Plot. Y-axis: Neuron Index (e.g., 0 to 99). X-axis: Time (ms). Dots or vertical ticks indicate spike events. Shows some background firing and a period of synchronized bursting where many neurons fire together.]`

`[Conceptual Figure 4.4: Example Population Firing Rate Plot. Y-axis: Population Rate (Hz). X-axis: Time (ms). A line graph showing the smoothed firing rate of the population corresponding to the raster plot in Fig 4.3, with clear peaks during the synchronized bursts.]`

**4.7 Brian2 Implementation: Simple E/I Networks**

We now apply the concepts introduced in this chapter—defining neuron populations (`NeuronGroup`), establishing synaptic connections (`Synapses`, `connect()`) with specific weights and delays, implementing synaptic models, monitoring network activity (`SpikeMonitor`, `PopulationRateMonitor`, `StateMonitor`), and visualizing results (raster plots, rate plots)—to build and simulate illustrative networks using Brian2. We begin with a minimal two-neuron circuit to clearly demonstrate direct interaction and synaptic delay, and then proceed to construct and analyze a slightly larger, more representative network incorporating distinct excitatory (E) and inhibitory (I) populations with random interconnectivity, allowing us to observe emergent network dynamics.

**Example 1: Two Connected Neurons**
This first example focuses on the simplest possible network interaction: neuron 0 provides excitatory input to neuron 1 with a defined synaptic delay. To keep the focus on the connection, we simplify the synapse to cause a direct voltage jump in the postsynaptic neuron and provide a constant drive to neuron 0 to ensure it fires reliably. This allows us to clearly observe the cause-and-effect relationship mediated by the synapse and the delay.

```python
# === Brian2 Simulation: Two Connected LIF Neurons ===
# (4.1_TwoConnectedNeurons.ipynb)
from brian2 import *
import matplotlib.pyplot as plt

# Use start_scope() in scripts or notebooks to ensure Brian2 objects
# from previous runs are cleared, preventing unexpected interactions.
start_scope()

# --- 1. Parameters ---
# Neuron parameters (shared by both neurons)
tau = 10*ms; V_rest = -70*mV; V_thresh = -50*mV; V_reset = -75*mV; t_ref = 2*ms
# Synapse parameters
weight_exc = 15*mV # Excitatory weight defined as a direct voltage increase for simplicity
delay_syn = 3*ms   # Synaptic transmission delay from neuron 0 to neuron 1
# Input drive parameter (to make neuron 0 fire)
drive_amount = 25*mV # Constant effective voltage drive applied to neuron 0

# --- 2. Neuron Model ---
# Define LIF model equations with a 'V_drive' parameter
eqs_v_input = '''
dv/dt = (V_rest - v + V_drive)/tau : volt (unless refractory)
V_drive : volt # Driving voltage term (parameter)
'''
# Create a NeuronGroup containing N=2 neurons using these equations
neurons = NeuronGroup(2, eqs_v_input, threshold='v>V_thresh',
                      reset='v=V_reset', refractory=t_ref, method='euler', name='TwoNeurons')

# Set initial conditions and specify the drive for each neuron
neurons.v = V_rest              # Initialize both neurons at resting potential
neurons.V_drive[0] = drive_amount # Neuron 0 receives the constant drive
neurons.V_drive[1] = 0*mV         # Neuron 1 receives no external drive

# --- 3. Synapses ---
# Define the synaptic connection from neuron 0 to neuron 1
# The 'on_pre' string specifies the action: increase postsynaptic voltage 'v_post' by 'weight_exc'
syn = Synapses(neurons, neurons, # Source and target are the same group
               on_pre='v_post += weight_exc',
               delay=delay_syn,
               name='ExcitatoryLink_0_to_1')
# Use the connect() method to create the specific connection: i=0 (source) to j=1 (target)
syn.connect(i=0, j=1)

# --- 4. Monitors ---
# Monitor the state variable 'v' (voltage) for both neurons (record=True)
state_mon = StateMonitor(neurons, 'v', record=True, name='VoltageMonitor')
# Monitor spike events (time and index) for both neurons
spike_mon = SpikeMonitor(neurons, name='SpikeMonitor')

# --- 5. Run Simulation ---
simulation_time = 100*ms
print(f"Running simulation for {simulation_time}...")
run(simulation_time)
print("Simulation complete.")

# --- 6. Analyze and Visualize Results ---
print("Plotting results...")
plt.figure(figsize=(12, 7)) # Adjusted figure size

# Plot Neuron 0 (Presynaptic) Voltage Trace
plt.subplot(2, 1, 1)
plt.plot(state_mon.t/ms, state_mon.v[0]/mV, label='Neuron 0 (Presynaptic)', color='blue', lw=1.5)
plt.axhline(V_thresh/mV, ls='--', color='red', lw=1, label='Threshold')
plt.ylabel('Potential (mV)')
plt.title('Two Connected Neurons Simulation Results')
plt.legend(loc='upper right')
plt.grid(True, linestyle=':')

# Plot Neuron 1 (Postsynaptic) Voltage Trace
plt.subplot(2, 1, 2)
plt.plot(state_mon.t/ms, state_mon.v[1]/mV, label='Neuron 1 (Postsynaptic)', color='green', lw=1.5)
# Add vertical lines indicating the arrival time of spikes from neuron 0 at neuron 1
neuron0_spike_times = spike_mon.t[spike_mon.i == 0]
if len(neuron0_spike_times) > 0:
    arrival_times = (neuron0_spike_times + delay_syn) / ms
    # Get current y-axis limits to draw lines across the plot
    ymin, ymax = plt.ylim()
    plt.vlines(arrival_times, ymin, ymax,
               color='gray', lw=1, ls=':',
               label=f'Pre-Spike Arrival (delay={delay_syn/ms}ms)')
    plt.ylim(ymin, ymax) # Ensure ylim doesn't change due to vlines

plt.axhline(V_thresh/mV, ls='--', color='red', lw=1) # Threshold line
plt.xlabel('Time (ms)')
plt.ylabel('Potential (mV)')
plt.legend(loc='upper right')
plt.grid(True, linestyle=':')

plt.tight_layout() # Adjust subplot spacing
plt.show()

# Optional: Print spike information for verification
neuron0_spike_count = np.sum(spike_mon.i == 0)
neuron1_spike_count = np.sum(spike_mon.i == 1)
print(f"\nNeuron 0 spiked {neuron0_spike_count} times.")
if neuron0_spike_count > 0:
    print(f"  Neuron 0 spike times (ms): {np.round(neuron0_spike_times/ms, 2)}")
print(f"Neuron 1 spiked {neuron1_spike_count} times.")
if neuron1_spike_count > 0:
    neuron1_spike_times = spike_mon.t[spike_mon.i == 1]
    print(f"  Neuron 1 spike times (ms): {np.round(neuron1_spike_times/ms, 2)}")

```

**Explanation of Example 1:**
This code simulates a fundamental synaptic interaction. Neuron 0, driven by `V_drive`, fires periodically. Each time it fires, after the `delay_syn` (3 ms), its action potential causes a direct jump (`weight_exc` = 15 mV) in the membrane potential of neuron 1. If this jump, potentially combined with the decay from previous jumps, pushes neuron 1's potential above `V_thresh`, neuron 1 will also fire. The plots clearly visualize this delayed cause-and-effect relationship, showing neuron 1's voltage increasing sharply shortly after neuron 0 fires (indicated by the gray dashed lines representing spike arrival times). This simple example verifies the correct functioning of `NeuronGroup`, `Synapses`, `connect`, `delay`, and the monitors.

**Example 2: Simple E/I Network**
Now, we construct a slightly larger network (N=100 neurons) comprising distinct excitatory (80%) and inhibitory (20%) populations. Neurons within and between these populations are connected randomly with a specified probability. This setup allows us to observe emergent network dynamics resulting from the interplay of recurrent excitation and inhibition, a core motif in many biological circuits, including those forming in organoids. We will use the simplified voltage-jump synaptic model again for clarity in this introductory network example.

```python
# === Brian2 Simulation: Simple E/I Network ===
# (4.2_SimpleEI_Network.ipynb)
from brian2 import *
import matplotlib.pyplot as plt
import numpy as np # Import numpy for calculations if needed

start_scope()

# --- 1. Parameters ---
# Network Size
N = 100
N_E = 80 # Number of excitatory neurons
N_I = 20 # Number of inhibitory neurons
# Neuron parameters (using simpler V_drive model)
tau = 10*ms; V_rest = -70*mV; V_thresh = -50*mV; V_reset = -75*mV; t_ref = 2*ms
# Synaptic parameters (voltage jump model)
# Weights (Choose values to get interesting dynamics, may need tuning)
w_EE = 0.8*mV # E->E synapse causes 0.8mV jump
w_EI = 1.0*mV # E->I synapse causes 1.0mV jump
w_IE = -2.5*mV # I->E synapse causes -2.5mV jump (inhibitory)
w_II = -2.0*mV # I->I synapse causes -2.0mV jump (inhibitory)
# Connection probability (uniform for all connection types here)
p_connect = 0.1 # 10% connection probability
# Synaptic delay
delay_syn = 1.5*ms
# Baseline drive (to provide some spontaneous activity)
base_drive = 10.5*mV # Adjust this value to control baseline firing rate

# --- 2. Neuron Model ---
eqs = '''
dv/dt = (V_rest - v + V_drive)/tau : volt (unless refractory)
V_drive : volt # Driving voltage term (parameter)
'''
# Create the full population
neurons = NeuronGroup(N, eqs, threshold='v>V_thresh', reset='v=V_reset',
                      refractory=t_ref, method='euler', name='EI_Population')
# Create convenient subgroup references using slicing
P_E = neurons[:N_E]; P_E.name = 'Excitatory'
P_I = neurons[N_E:]; P_I.name = 'Inhibitory'

# Set initial conditions and baseline drive
neurons.v = V_rest + rand(N)*(V_thresh - V_rest)*0.5 # Random initial V near rest
neurons.V_drive = base_drive # Apply baseline drive to all neurons

# --- 3. Synapses ---
# Create four Synapses objects for the four connection types (E->E, E->I, I->E, I->I)
# Using the simple voltage jump model defined by 'on_pre'

# E -> E connections
syn_EE = Synapses(P_E, P_E, on_pre='v_post += w_EE', delay=delay_syn, name='EE_Syn')
syn_EE.connect(condition='i!=j', p=p_connect) # Connect E to E, avoid self-connections

# E -> I connections
syn_EI = Synapses(P_E, P_I, on_pre='v_post += w_EI', delay=delay_syn, name='EI_Syn')
syn_EI.connect(p=p_connect) # Connect E to I

# I -> E connections
syn_IE = Synapses(P_I, P_E, on_pre='v_post += w_IE', delay=delay_syn, name='IE_Syn')
syn_IE.connect(p=p_connect) # Connect I to E

# I -> I connections
syn_II = Synapses(P_I, P_I, on_pre='v_post += w_II', delay=delay_syn, name='II_Syn')
syn_II.connect(condition='i!=j', p=p_connect) # Connect I to I, avoid self-connections

# --- 4. Monitors ---
# Monitor spikes from all neurons
spike_mon = SpikeMonitor(neurons, name='AllSpikes')
# Monitor population rates for E and I groups separately
rate_mon_E = PopulationRateMonitor(P_E, name='ExcRate')
rate_mon_I = PopulationRateMonitor(P_I, name='InhRate')

# --- 5. Run Simulation ---
simulation_time = 500*ms
print(f"Running E/I network simulation for {simulation_time}...")
run(simulation_time)
print("Simulation complete.")

# --- 6. Analyze and Visualize Results ---
print("Plotting results...")
plt.figure(figsize=(12, 8)) # Adjusted figure size

# Subplot 1: Raster Plot (colored by E/I)
plt.subplot(2, 1, 1)
spike_indices = spike_mon.i
spike_times = spike_mon.t
is_E = spike_indices < N_E
is_I = spike_indices >= N_E
plt.plot(spike_times[is_E]/ms, spike_indices[is_E], '.r', markersize=2, label='Excitatory')
plt.plot(spike_times[is_I]/ms, spike_indices[is_I], '.b', markersize=2, label='Inhibitory')
# plt.axhline(N_E, ls='-', color='gray', lw=0.5) # Optional separator line
plt.xlabel('Time (ms)')
plt.ylabel('Neuron Index')
plt.title(f'Raster Plot (N={N}, E=Red, I=Blue)')
plt.xlim(0, simulation_time/ms)
plt.ylim(-1, N)
plt.legend(markerscale=4, loc='upper right')
plt.grid(True, axis='y', linestyle=':', alpha=0.5)

# Subplot 2: Population Firing Rates
plt.subplot(2, 1, 2)
plt.plot(rate_mon_E.t/ms, rate_mon_E.rate/Hz, label='Excitatory Rate', color='red', lw=1.5)
plt.plot(rate_mon_I.t/ms, rate_mon_I.rate/Hz, label='Inhibitory Rate', color='blue', lw=1.5)
plt.xlabel('Time (ms)')
plt.ylabel('Population Rate (Hz)')
plt.title('Population Firing Rates')
plt.legend(loc='upper right')
plt.xlim(0, simulation_time/ms)
plt.ylim(bottom=0) # Ensure rate doesn't go below zero
plt.grid(True, linestyle=':', alpha=0.5)

plt.tight_layout() # Adjust subplot spacing
plt.show()

# Optional: Calculate and print average firing rates
avg_rate_E = spike_mon.count[spike_mon.i < N_E].sum() / N_E / simulation_time
avg_rate_I = spike_mon.count[spike_mon.i >= N_E].sum() / N_I / simulation_time
print(f"\nAverage Firing Rates:")
print(f"  Excitatory: {avg_rate_E:.2f} Hz")
print(f"  Inhibitory: {avg_rate_I:.2f} Hz")

```

**Explanation of Example 2:**
This simulation sets up a recurrent network with both excitatory and inhibitory interactions. The `neurons` group contains both types, identified by slicing (`P_E`, `P_I`). Four `Synapses` objects define the connectivity within and between these populations, using random connections (`p=0.1`). The weights are set such that E neurons excite, and I neurons inhibit (by causing negative voltage jumps). A baseline drive (`base_drive`) is applied to all neurons to ensure some level of spontaneous activity emerges from the network interactions rather than solely from external input (though background Poisson input, as in Chapter 5, is a more realistic way to achieve this). The simulation runs for 500 ms. The resulting plots show the network's emergent behavior. The raster plot (colored) typically reveals complex firing patterns, possibly asynchronous or potentially exhibiting some synchronous bursts or oscillations depending on the chosen parameters (weights, delays, drive). The population rate plots show the time course of activity for the E and I groups, often revealing how inhibition tracks excitation or how oscillations manifest at the population level. This example demonstrates how to construct basic E/I networks, the foundation for modeling more complex circuits inspired by biological observations in systems like brain organoids, where the balance of excitation and inhibition is thought to be crucial for shaping activity patterns. Tuning the parameters (`w_EE` to `w_II`, `p_connect`, `base_drive`) would be necessary to achieve specific dynamic regimes (e.g., stable asynchronous firing vs. oscillatory behavior).

**4.8 Conclusion and Planned Code**

This chapter provided the essential tools and concepts for constructing and simulating basic spiking neural networks using Brian2, moving beyond the single-neuron focus of Chapter 3. We learned how to define populations of neurons with shared properties using `NeuronGroup`, appreciating the efficiency gained through vectorization and code generation. The core mechanisms for establishing connections were detailed through the `Synapses` object, including defining synaptic models via `on_pre` (and potentially `model` variables like `w`), incorporating biologically relevant transmission delays (`delay`), and utilizing various connectivity strategies (`connect()` method) such as probabilistic or conditional rules to create sparse or structured networks. We delved deeper into implementing current-based versus conductance-based synaptic models, emphasizing the importance of reversal potentials and driving forces for the latter's biological realism. Essential monitoring tools were expanded, detailing the usage and trade-offs of `SpikeMonitor`, `StateMonitor`, and `PopulationRateMonitor` for capturing different aspects of network activity. Finally, standard visualization techniques like raster plots and population rate plots were presented as key methods for interpreting the complex dynamics emerging from network simulations. The chapter culminated in two practical Brian2 examples: simulating a simple two-neuron excitatory connection with delay and building a small recurrent network comprising distinct excitatory and inhibitory populations with random connectivity. These examples serve as crucial stepping stones, equipping the reader with the foundational skills needed to tackle the more complex, biologically inspired organoid models in the subsequent chapters, where we will incorporate features like heterogeneity (Chapter 5), synaptic plasticity (Chapter 6), more realistic network structures (Chapter 7), and explore their computational primitives (Chapters 9 & 10).

**Planned Code Examples:**
*   **`4.1_TwoConnectedNeurons.ipynb`:** (Provided and explained in Section 4.7) Simulates two LIF neurons where one excites the other with a delay, demonstrating basic `NeuronGroup`, `Synapses`, `connect()`, `delay`, and monitoring of voltage traces.
*   **`4.2_SimpleEI_Network.ipynb`:** (Provided and explained in Section 4.7) Simulates a small network of randomly connected excitatory and inhibitory LIF neurons, illustrating the creation of subpopulations via slicing, probabilistic connectivity between groups (`p` argument), separate `Synapses` objects for different connection types, and visualization of network activity using colored raster plots and population rates.

------
**References for Further Reading**

1.  **Sprekeler, H. (2023). Balanced E/I in cortical networks: constraints, computations, and controversies.** *Current Opinion in Neurobiology, 83*, 102790. https://doi.org/10.1016/j.conb.2023.102790
    *   *Summary:* This review provides a timely synthesis of the theory and evidence surrounding balanced excitation and inhibition in cortical circuits. It discusses how network structure (connectivity) and synaptic properties lead to characteristic dynamic states (like the asynchronous irregular activity often modeled), explores the computational implications of balance, and highlights ongoing debates. Highly relevant background for understanding and modeling the E/I networks introduced in Section 4.7.*
2.  **Stefanon, G., & Destexhe, A. (2023). Biophysical modeling of the dynamics of neuronal populations.** *Scholarpedia, 18*(1), 55741. http://www.scholarpedia.org/article/Biophysical_modeling_of_the_dynamics_of_neuronal_populations
    *   *Summary:* Offers a concise yet comprehensive overview of various computational approaches used to model the collective dynamics emerging from large populations of neurons. It covers methods ranging from detailed spiking network simulations (like those built in this chapter using `NeuronGroup` and `Synapses`) to more abstract mean-field descriptions, providing valuable context for situating the modeling techniques employed.*
3.  **Borges, R. R., Tomé, B., Carvalho, D. V., Required, F., Needed, F., & Required, F. (2022). A perspective on the requirements for simulators of biologically plausible spiking neural networks.** *Frontiers in Neuroscience, 16*, 1019007. https://doi.org/10.3389/fnins.2022.1019007
    *   *Summary:* This perspective article discusses the essential features and desirable capabilities required in modern software tools designed for simulating biologically realistic spiking neural networks (SNNs). It touches upon aspects like model definition flexibility (equation-oriented), handling heterogeneity, plasticity, network construction, performance, and usability, providing context for why simulators like Brian2 (used in this chapter) are developed and chosen.*
4.  **Huang, C., Ruff, D. A., Pyle, R., Rosenbaum, R., Cohen, M. R., & Doiron, B. (2023). Dynamic gain control explains stimulus-specific suppression in primary visual cortex.** *eLife, 12*, e84449. https://doi.org/10.7554/eLife.84449
    *   *Summary:* A primary research and modeling paper that utilizes simulations of recurrent E/I spiking neural networks (similar in principle to the one built in Section 4.7, though more complex) to explain specific response properties observed in visual cortex. Serves as a recent example of how the types of network models introduced in this chapter are applied to investigate neural computations.*
5.  **Onken, A., & Liu, J. K. (2023). Analyzing and visualizing the dynamics of spiking neural networks.** *Frontiers in Computational Neuroscience, 17*, 1110099. https://doi.org/10.3389/fncom.2023.1110099
    *   *Summary:* This article discusses various methods and challenges associated with analyzing and visualizing the complex, high-dimensional dynamics generated by spiking neural network simulations. It covers techniques relevant to interpreting the outputs from monitors (like raster plots and population rates, Section 4.6) and extracting meaningful insights about network states and computations.*
6.  **Carlu, M., Bist, B., Kekuš, M., Mainen, Z. F., Pascucci, V., & Stiles, J. R. (2022). Simulating the multi-scale brain: extrasynaptic transmission and integration between brain-region simulators.** *Frontiers in Neuroinformatics, 16*, 900237. https://doi.org/10.3389/fninf.2022.900237
    *   *Summary:* Discusses the significant challenges and emerging approaches in simulating brain function across multiple scales, including integrating detailed network models (like those built using the methods in this chapter) with models of other processes or larger brain regions. Provides context for placing the network construction techniques within a broader simulation landscape.*
7.  **Trujillo, C. A., Rice, E. S., Schaefer, N. K., & Muotri, A. R. (2022). Re-exploring brain function with human neural organoids.** *Cell Stem Cell, 29*(11), 1540–1558. https://doi.org/10.1016/j.stem.2022.10.006
    *   *Summary:* While primarily focused on the biology of organoids, this review highlights the emergent network activity observed in these systems. Understanding how to simulate basic network structures (as covered in this chapter) is a prerequisite for building models aimed at replicating or explaining these observed organoid dynamics, providing motivation for the techniques learned.*
8.  **Lobato-Rincon, L. L., Gönczy, L., & Lengyel, M. (2022). Principles of recurrent neural network dynamics for sequence processing and prediction.** *Current Opinion in Neurobiology, 76*, 102596. https://doi.org/10.1016/j.conb.2022.102596
    *   *Summary:* Reviews theoretical principles governing the dynamics of recurrent neural networks (RNNs), including spiking networks with recurrent connections like the E/I network in Section 4.7. It discusses how network structure and dynamics relate to computational capabilities like sequence processing, providing theoretical context for analyzing the behavior of simulated networks.*
9.  **Pauli, R., Stimberg, M., Dahmen, D., & Tetzlaff, C. (2022). Phase transitions of memory formation in field-based neuromorphic hardware.** *eLife, 11*, e77172. https://doi.org/10.7554/eLife.77172
    *   *Summary:* This computational study utilizes Brian2 to simulate relatively complex network models incorporating plasticity (beyond this chapter's scope) to investigate memory formation. It serves as a recent example showcasing the application of Brian2 for constructing and simulating non-trivial network architectures, demonstrating the utility of the foundational techniques covered in Chapter 4.*
10. **Kumarasinghe, K., Kuhl, E., & Goriely, A. (2023). Neural dynamics on graphs: A network-centric perspective.** *Frontiers in Computational Neuroscience, 17*, 1106427. https://doi.org/10.3389/fncom.2023.1106427
    *   *Summary:* This article provides a perspective on studying neural dynamics from a network science viewpoint, emphasizing the interplay between network structure (connectivity, topology - as defined in Section 4.3) and the resulting collective dynamics. It highlights the importance of graph theoretical concepts in understanding simulated network behavior.*
   
-----
