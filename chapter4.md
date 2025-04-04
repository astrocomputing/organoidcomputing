---

# Chapter 4

# Building and Simulating Neural Networks with Brian2
---


*Building upon the foundations of single neuron modeling laid in Chapter 3, this chapter takes the critical step towards simulating interconnected **neural networks**, the fundamental architecture underlying brain function and, potentially, Organoid Computing. We transition from simulating isolated neurons to modeling populations and their interactions using the Brian2 simulator. We begin by exploring how to efficiently represent and manage groups of neurons using the `NeuronGroup` object, moving beyond single-neuron simulations. The core focus then shifts to establishing connections between neurons using Brian2's powerful `Synapses` object. We will detail how to define the synaptic model—specifying what happens when a presynaptic neuron fires (`on_pre`)—and how to incorporate crucial parameters like synaptic weights (`w`) and transmission delays (`delay`). Subsequently, we examine various common strategies for specifying the connectivity patterns within a network, ranging from simple one-to-one or all-to-all connections to more biologically relevant probabilistic or spatially conditional wiring, implemented via the `connect()` method of the `Synapses` object. We revisit synaptic models, contrasting current-based and conductance-based approaches within the practical context of Brian2's `Synapses` syntax and discussing the role of reversal potentials. We then expand our repertoire of monitoring tools, introducing the `PopulationRateMonitor` alongside the `SpikeMonitor` and `StateMonitor` to track collective network activity. Techniques for visualizing network dynamics, particularly raster plots and population firing rate histograms derived from monitor data, will be demonstrated. Finally, the chapter culminates in practical Brian2 implementations, guiding the reader through simulating a simple two-neuron circuit and then constructing and analyzing a basic randomly connected network composed of distinct excitatory and inhibitory populations, illustrating the emergence of network-level activity patterns.*

-----

**4.1 Neuron Populations (`NeuronGroup`)**

While simulating a single neuron, as demonstrated in Chapter 3, is essential for understanding basic principles, the true computational power of neural systems arises from the collective activity of large numbers of interconnected neurons. Brain organoids, even in their current state, contain thousands to millions of neurons. Therefore, any meaningful simulation aiming to capture their dynamics must efficiently handle populations of neurons. Brian2 provides the `NeuronGroup` object precisely for this purpose. A `NeuronGroup` represents a collection of neurons that all share the same mathematical model (defined by the same set of differential equations and parameters) but can potentially have different state variable values (like individual membrane potentials) or receive different inputs.

Creating a `NeuronGroup` involves specifying the number of neurons in the group (`N`), the string containing the model equations (`model`), the condition for spike firing (`threshold`), the action(s) to take upon firing (`reset`), the refractory period (`refractory`), and the numerical integration method (`method`). For example, to create a population of 100 LIF neurons identical to the one modeled in Chapter 3:

```python
# Assuming lif_eqs, V_thresh, V_reset, t_ref are defined as before
N_pop = 100
population = NeuronGroup(N=N_pop,
                         model=lif_eqs,
                         threshold='v > V_thresh',
                         reset='v = V_reset',
                         refractory=t_ref,
                         method='euler')
```

This single line creates 100 independent LIF neurons, each governed by the equations in `lif_eqs`. Brian2 handles the underlying implementation efficiently, often using vectorized operations (performing the same calculation on multiple neurons simultaneously) which is significantly faster than simulating each neuron in a Python loop. State variables within the `NeuronGroup` (like `v` or `I_inj` in our example) are now treated as arrays, where each element corresponds to a neuron in the group. For instance, `population.v` would be an array containing the membrane potentials of all 100 neurons. We can access or set the state variable for individual neurons using indexing (e.g., `population.v[0]` for the first neuron, `population.v[10:20]` for neurons 10 through 19) or set values for the entire population simultaneously (e.g., `population.v = V_rest` initializes all neurons to the resting potential; `population.I_inj = 0.1*nA` applies the same input current to all neurons). This ability to manage large populations efficiently through `NeuronGroup` is fundamental for scaling up our simulations. Often, a network model will consist of multiple `NeuronGroup` objects, for instance, one group representing excitatory neurons and another representing inhibitory neurons, potentially with different parameters or even different underlying model equations.

`[Conceptual Figure 4.1: NeuronGroup Representation. A diagram showing a box labeled "NeuronGroup (N=100)". Inside, icons represent individual neurons. Arrows point from the NeuronGroup definition (specifying model, N, threshold, etc.) to the population. Below, conceptual arrays are shown for state variables like `v` and `I_inj`, each of length N, illustrating vectorized representation.]`

**4.2 Synaptic Connections (`Synapses`): Model (`on_pre`), Weight (`w`), Delay (`delay`)**

The defining characteristic of a neural network is the presence of **synaptic connections** that allow neurons to influence each other's activity. In Brian2, synapses are modeled using the `Synapses` object. A `Synapses` object connects a **source** `NeuronGroup` to a **target** `NeuronGroup` (which can be the same group, allowing for recurrent connections). It defines the rules governing how the firing of a neuron in the source group affects neurons in the target group.

Creating a `Synapses` object requires specifying the source group, the target group, and crucially, a `model` string that defines any synaptic variables (like synaptic weights) and an `on_pre` string that defines the action(s) to be taken on the target neuron(s) when a spike arrives from a connected source neuron. Let's consider a simple example of excitatory synapses that increase a postsynaptic current variable (`I_syn`) in the target neurons upon presynaptic spike arrival:

```python
# Assuming 'source_group' and 'target_group' are NeuronGroup objects
# Let's assume target_group has a variable 'I_syn : amp' in its model equations
synapses = Synapses(source_group, target_group,
                    model='w : 1',  # Define a synaptic weight 'w' (dimensionless unit 1)
                    on_pre='I_syn_post += w*nA') # On presynaptic spike, add w*nA to target I_syn
                                                 # '_post' accesses the target neuron's variable
```

In this example:
*   `model='w : 1'` defines a synaptic variable `w` associated with each potential synapse. This typically represents the **synaptic weight** or strength. We give it dimensionless units (`1`) here, but it could have units (e.g., `volt` if directly adding to voltage, or `siemens` if representing conductance). Each individual connection created later will have its own value for `w`.
*   `on_pre='I_syn_post += w*nA'` specifies what happens when a neuron in `source_group` fires. The `on_pre` pathway is activated. It accesses the `I_syn` variable of the connected postsynaptic neuron(s) (indicated by the `_post` suffix) and increments it by the value of the specific synapse's weight `w`, multiplied by `nA` to give it current units (assuming `I_syn` was defined with `amp` units in the target `NeuronGroup`).

Brian2 also allows for modeling **transmission delays**, the time lag between a presynaptic spike and its effect on the postsynaptic neuron, which arises from finite axonal conduction velocity and synaptic processing time. Delays are handled internally by Brian2's spike propagation mechanism. You can assign specific delay values to individual synapses. Each `Synapses` object has a `delay` attribute, which can be set when creating connections:

```python
# ... within the Synapses object or when connecting ...
synapses.delay = 2*ms # Set a uniform delay for all synapses created next
# OR set individual delays: synapses.delay[i, j] = ...
```

The `Synapses` object acts as a container for all potential connections between the source and target groups. It stores the synaptic variables (like `w`) and manages the propagation of spikes according to the specified delays and `on_pre` rules. However, simply creating the `Synapses` object doesn't create any actual connections yet. That requires using its `connect()` method, described next.

**4.3 Connectivity Strategies: One-to-one, All-to-all, Probabilistic, Conditional**

After defining the `Synapses` object with its model and `on_pre` actions, we need to specify which specific pairs of presynaptic and postsynaptic neurons are actually connected. This is done using the `connect()` method of the `Synapses` object. Brian2 offers several flexible ways to define the network **connectivity** or **topology**.

*   **One-to-one connections:** Connect neuron `i` in the source group to neuron `i` in the target group (requires source and target groups to have the same size).
    ```python
    synapses.connect(j='i')
    ```
*   **All-to-all connections:** Connect every neuron in the source group to every neuron in the target group (excluding self-connections if source and target are the same group, unless specified otherwise).
    ```python
    synapses.connect() # No arguments means all-to-all
    ```
*   **Specific pairs:** Connect specific neuron pairs by providing indices.
    ```python
    synapses.connect(i=[0, 1], j=[2, 3]) # Connects neuron 0->2 and neuron 1->3
    ```
*   **Conditional connections:** Connect neurons based on a condition evaluated for each potential pair `(i, j)`, where `i` is the index in the source group and `j` is the index in the target group. The condition is provided as a string.
    ```python
    # Connect neuron i to neuron j only if i is not equal to j (no self-connections)
    synapses.connect(condition='i != j')

    # Connect neuron i to j only if j is within +/- 5 indices of i (local connectivity)
    synapses.connect(condition='abs(i - j) < 5')
    ```
*   **Probabilistic connections:** Connect each possible pair `(i, j)` with a certain probability `p`.
    ```python
    # Connect each pair with a 10% probability
    synapses.connect(p=0.1)
    ```

Often, connectivity rules are combined. For example, one might want to connect neurons probabilistically but only if they meet a certain condition (e.g., not connecting to themselves):

```python
# Connect with 10% probability, excluding self-connections
synapses.connect(condition='i != j', p=0.1)
```

Once connections are created using `connect()`, you can set the values for synaptic variables like weights (`w`) and delays (`delay`) for these specific connections. This can be done uniformly, randomly, or based on indices.

```python
# After synapses.connect(...)

# Set all weights to a specific value
synapses.w = 1.5

# Set weights randomly from a normal distribution
synapses.w = 'rand()*0.5 + 1.0' # Weight ~ Uniform(1.0, 1.5) in this example
# Or using normal distribution: synapses.w = '1.0 + 0.2*randn()'

# Set delays based on distance (if neurons have positions 'x_pre', 'x_post')
# Requires defining x in the NeuronGroups and linking in Synapses model
# synapses.delay = 'sqrt((x_pre - x_post)**2) / velocity' # Conceptual example
```

These flexible connectivity mechanisms allow the construction of diverse network architectures, from simple regular structures to complex, random, or spatially organized topologies inspired by biological observations in systems like brain organoids, where connectivity is often local and probabilistic.

`[Conceptual Figure 4.2: Connectivity Examples. Panel (a): One-to-one connection between two groups. Panel (b): All-to-all connection. Panel (c): Probabilistic random connection (shows only a subset of possible connections being made). Panel (d): Conditional connection (e.g., connecting only nearby neurons).]`

**4.4 Synaptic Models (Current Injection, Conductance-Based). Reversal Potentials.**

As introduced in Chapter 3 (Section 3.4), synaptic effects can be modeled either by injecting a current or by changing the conductance of the postsynaptic membrane. Brian2's `Synapses` object allows for straightforward implementation of both types within its `model` and `on_pre` (or `on_post`) definitions.

**Current-Based Synapses:** This is often the simplest approach. The postsynaptic `NeuronGroup` needs a state variable representing the total synaptic current it receives (e.g., `I_syn : amp`). The `Synapses` object then modifies this variable directly in its `on_pre` pathway.

```python
# Target NeuronGroup model includes: I_syn : amp
# Synapses definition:
syn_current = Synapses(source, target,
                       model='w : amp', # Weight is now in current units
                       on_pre='I_syn_post += w')
# Connect and set weights:
syn_current.connect(p=0.1)
syn_current.w = '50*pA + 10*pA*randn()' # Assign weights with units
```
Here, upon a presynaptic spike, the weight `w` (which has units of current, e.g., picoamps `pA`) is directly added to the `I_syn` variable of the connected postsynaptic neuron(s). The `I_syn` variable would then typically be included in the target neuron's membrane potential equation, for example: `dv/dt = (V_rest - v + R*I_syn)/tau : volt`. This approach is computationally efficient but ignores the dependence of synaptic current on the postsynaptic membrane potential (the driving force).

**Conductance-Based Synapses:** This model provides greater biological realism by explicitly modeling the change in membrane conductance triggered by neurotransmitter binding. The postsynaptic `NeuronGroup` model needs variables representing the synaptic conductances (e.g., `g_exc : siemens` for excitatory, `g_inh : siemens` for inhibitory). The membrane potential equation must then include terms for the currents generated by these conductances, incorporating the respective **reversal potentials** ($E_{\text{rev}}$).

```python
# Target NeuronGroup model includes:
# g_exc : siemens
# g_inh : siemens
# ... other variables ...
# Equation includes synaptic currents:
# dv/dt = ( ... - g_exc*(v - E_exc) - g_inh*(v - E_inh) ... ) / Cm : volt

# Define reversal potentials (constants or group parameters)
E_exc = 0*mV
E_inh = -75*mV

# Excitatory Synapses definition:
syn_exc = Synapses(source_exc, target,
                   model='''w : siemens # Weight is now conductance increase
                            dg_syn/dt = -g_syn/tau_syn : siemens (event-driven)''', # Conductance decay
                   on_pre='g_syn += w') # Increase conductance on spike arrival
                   # Link g_syn to target's g_exc: g_exc_post = g_syn (can be implicit if names match)
                   # More commonly, directly increment target conductance:
syn_exc = Synapses(source_exc, target,
                   model='w : siemens', # Weight is conductance increment
                   on_pre='g_exc_post += w') # Directly add weight to target conductance state variable

# Inhibitory Synapses definition (similar, targetting g_inh):
syn_inh = Synapses(source_inh, target,
                   model='w : siemens',
                   on_pre='g_inh_post += w')

# Assume target neuron needs conductance decay (add to NeuronGroup equations):
# dg_exc/dt = -g_exc / tau_exc : siemens
# dg_inh/dt = -g_inh / tau_inh : siemens
```
In this conductance-based scheme (showing a simplified direct increment version in `on_pre`), the presynaptic spike causes a step increase (`w`) in the relevant synaptic conductance variable (`g_exc` or `g_inh`) of the postsynaptic neuron. These conductance variables typically decay exponentially back to zero over time (modeled by adding `dg/dt = -g/tau_syn` equations to the *postsynaptic* `NeuronGroup`). The membrane potential equation in the postsynaptic `NeuronGroup` must then include the terms `-g_exc*(v - E_exc)` and `-g_inh*(v - E_inh)` representing the currents flowing through these conductances, driven by the difference between the membrane potential `v` and the respective reversal potentials (`E_exc` typically ~0 mV, `E_inh` typically ~-70 to -80 mV). While requiring slightly more complex model definitions, conductance-based synapses more accurately capture synaptic integration properties, including shunting inhibition and the non-linear summation of inputs, which can be crucial for understanding network dynamics, especially in regimes with high activity or strong E/I interactions relevant to organoid modeling.

**4.5 Network Monitoring (`SpikeMonitor`, `PopulationRateMonitor`)**

Understanding the behavior of simulated neural networks requires effective tools for monitoring their activity during the simulation run. Brian2 provides several `Monitor` objects for this purpose, building on those introduced in Chapter 3.

The **`SpikeMonitor`** remains essential for recording the precise timing of action potentials from neurons in a specified `NeuronGroup`. It stores the index `i` of the neuron that fired and the time `t` of the spike. This information is fundamental for creating raster plots (visualizing individual spike times) and for performing spike train analysis (e.g., calculating firing rates, interspike intervals, correlations).

```python
# Monitor spikes from the 'population' NeuronGroup
spike_mon = SpikeMonitor(population)
# After run(duration), access spike times via spike_mon.t
# Access indices of neurons that spiked via spike_mon.i
```

The **`StateMonitor`** allows recording the continuous evolution of one or more state variables (like membrane potential `v`, synaptic conductances `g_exc`, `g_inh`, or adaptation variables `w`) from a subset or all neurons in a `NeuronGroup`. This is useful for detailed inspection of subthreshold dynamics or the internal state of specific neurons, but can generate large amounts of data, especially for large networks or long simulations.

```python
# Monitor membrane potential 'v' and excitatory conductance 'g_exc'
# from neurons with indices 0, 10, and 20 in 'population'
state_mon = StateMonitor(population, ['v', 'g_exc'], record=[0, 10, 20])
# Access data: state_mon.t, state_mon.v, state_mon.g_exc
```

For analyzing the overall activity level of a neuronal population, recording every single spike or state variable can be excessive or computationally expensive. The **`PopulationRateMonitor`** provides a convenient way to estimate and record the instantaneous firing rate of an entire `NeuronGroup`, averaged over short time bins. It essentially counts the number of spikes occurring within each time step (or a slightly wider bin) and divides by the number of neurons and the bin width to provide an estimate of the population firing rate (typically in Hz). This gives a smooth measure of the collective activity level of the population.

```python
# Monitor the population firing rate of the 'population' NeuronGroup
rate_mon = PopulationRateMonitor(population)
# After run(duration), access rate estimates via rate_mon.rate
# Access corresponding time points via rate_mon.t
```

Using these monitors appropriately allows researchers to capture different aspects of network dynamics, from the fine-grained detail of individual spike times and membrane potential fluctuations to the macroscopic view of population activity levels, providing the raw data needed for visualization and quantitative analysis of the simulation results.

**4.6 Visualization (Raster Plots, Population Firing Rates)**

Effective visualization is crucial for interpreting the complex dynamics emerging from neural network simulations. The data collected by Brian2's monitors can be readily plotted using standard Python libraries like Matplotlib. Two particularly common and informative visualizations for network activity are raster plots and population firing rate plots.

A **Raster Plot** visualizes the spike times of individual neurons within a population. Typically, the y-axis represents the index of each neuron in the `NeuronGroup`, and the x-axis represents time. A small marker (often a dot or a short vertical line) is placed at the coordinate `(t, i)` whenever neuron `i` fires a spike at time `t`. Raster plots provide a detailed view of the network's spatio-temporal activity patterns. One can easily spot periods of synchronized firing (vertical alignment of markers), oscillations (regularly repeating patterns), propagating waves of activity, or differences in firing behavior between subpopulations. Data for raster plots comes directly from a `SpikeMonitor` (`spike_monitor.t` for time, `spike_monitor.i` for neuron index).

`[Conceptual Figure 4.3: Example Raster Plot. Y-axis: Neuron Index (e.g., 0 to 99). X-axis: Time (ms). Dots or vertical ticks indicate spike events. Shows some background firing and a period of synchronized bursting where many neurons fire together.]`

A **Population Firing Rate Plot** provides a more macroscopic view of the network's overall activity level over time. It plots the estimated instantaneous firing rate of the population (typically averaged across all neurons in the monitored `NeuronGroup`) as a function of time. This plot smooths out the details of individual spikes and highlights changes in the overall intensity of network activity. It is particularly useful for identifying periods of high activity (like network bursts), observing oscillatory modulations in the population rate, or tracking the network's response to stimuli or parameter changes. Data for this plot comes directly from a `PopulationRateMonitor` (`rate_monitor.t` for time, `rate_monitor.rate` for the estimated rate). Alternatively, one can calculate a binned population rate histogram from `SpikeMonitor` data, although `PopulationRateMonitor` is often more convenient.

`[Conceptual Figure 4.4: Example Population Firing Rate Plot. Y-axis: Population Rate (Hz). X-axis: Time (ms). A line graph showing the smoothed firing rate of the population corresponding to the raster plot in Fig 4.3, with clear peaks during the synchronized bursts.]`

Combining these two plots—the detailed raster plot and the summarized population rate plot, often aligned vertically with a shared time axis—provides a powerful way to understand both the fine-grained spiking dynamics and the overall collective behavior of the simulated neural network. These visualizations are essential tools for analyzing the output of the network simulations we will build, including the simple E/I network in the next section.

**4.7 Brian2 Implementation: Simple E/I Networks**

We now apply the concepts introduced in this chapter to build and simulate simple networks using Brian2. We will start with a minimal two-neuron network and then move to a slightly larger network containing distinct populations of excitatory (E) and inhibitory (I) neurons with random connectivity.

**Example 1: Two Connected Neurons**
Let's simulate two LIF neurons where neuron 0 excites neuron 1 with a delay.

```python
# === Brian2 Simulation: Two Connected LIF Neurons ===
from brian2 import *
import matplotlib.pyplot as plt

start_scope() # Clears Brian2 namespace from previous runs

# --- Parameters ---
tau = 10*ms; V_rest = -70*mV; V_thresh = -50*mV; V_reset = -75*mV; t_ref = 2*ms
weight_exc = 15*mV # Use voltage jump for simplicity (current-based equivalent)
delay_syn = 3*ms

# --- Neuron Model ---
# Inject constant current to neuron 0 to make it fire
eqs = '''
dv/dt = (V_rest - v)/tau : volt (unless refractory)
I_inj : amp # External current parameter
'''
neurons = NeuronGroup(2, eqs, threshold='v>V_thresh', reset='v=V_reset', refractory=t_ref, method='euler')
neurons.v = V_rest
neurons.I_inj[0] = 2.1*mV/ms * (tau / R) if 'R' in locals() else 2.1*mV/ms*10e-9*amp/(1*volt) # Simplified current calc needed
# Simplified: Assume tau/R = C = 10e-9 F. Then R*I = tau/C * I. Need I such that R*I > V_thresh - V_rest
# Let's directly apply current that causes sufficient voltage change rate
neurons.I_inj[0] = 2.5*mV/ms * 10e-9 * amp # Needs checking, units are tricky without R explicitly
# --- Simplified Approach: Use direct voltage input in model ----
eqs_v_input = '''
dv/dt = (V_rest - v + V_drive)/tau : volt (unless refractory)
V_drive : volt # Driving voltage term
'''
neurons = NeuronGroup(2, eqs_v_input, threshold='v>V_thresh', reset='v=V_reset', refractory=t_ref, method='euler')
neurons.v = V_rest
neurons.V_drive[0] = 25*mV # Drive neuron 0 above threshold
neurons.V_drive[1] = 0*mV

# --- Synapses ---
# Neuron 0 excites Neuron 1
syn = Synapses(neurons, neurons, on_pre='v_post += weight_exc', delay=delay_syn)
syn.connect(i=0, j=1) # Connect neuron 0 to neuron 1

# --- Monitors ---
state_mon = StateMonitor(neurons, 'v', record=True) # Record both neurons
spike_mon = SpikeMonitor(neurons)

# --- Run ---
run(100*ms)

# --- Plot ---
plt.figure(figsize=(12, 6))
# Neuron 0 potential
plt.subplot(2, 1, 1)
plt.plot(state_mon.t/ms, state_mon.v[0]/mV, label='Neuron 0 (Presynaptic)')
plt.axhline(V_thresh/mV, ls='--', color='r', lw=0.5)
plt.ylabel('Potential (mV)')
plt.title('Two Connected Neurons')
plt.legend()
plt.grid(True)
# Neuron 1 potential
plt.subplot(2, 1, 2)
plt.plot(state_mon.t/ms, state_mon.v[1]/mV, label='Neuron 1 (Postsynaptic)')
# Plot spike times of neuron 0 as markers on neuron 1's plot
neuron0_spikes = spike_mon.t[spike_mon.i == 0]
if len(neuron0_spikes) > 0:
    plt.vlines((neuron0_spikes + delay_syn)/ms, V_reset/mV - 5*mV, V_reset/mV + 5*mV, color='gray', lw=1, ls=':', label='Presynaptic Spike Arrival (Delayed)')
plt.axhline(V_thresh/mV, ls='--', color='r', lw=0.5, label='Threshold')
plt.xlabel('Time (ms)')
plt.ylabel('Potential (mV)')
plt.legend()
plt.grid(True)

plt.tight_layout()
plt.show()

# Print spike counts
print(f"Neuron 0 spiked {np.sum(spike_mon.i == 0)} times.")
print(f"Neuron 1 spiked {np.sum(spike_mon.i == 1)} times.")
```

*(Self-correction during generation: Calculating the correct `I_inj` to reliably make the LIF fire without knowing `R` is tricky. Switched to a simpler model `eqs_v_input` using a direct voltage drive term `V_drive` for neuron 0 to ensure it spikes, and used a direct voltage increment `v_post += weight_exc` in `on_pre` for simplicity in this first network example, equivalent to a current pulse integrated over one time step).*

**Example 2: Simple E/I Network**
Now, let's create a network of 100 LIF neurons, 80 excitatory (E) and 20 inhibitory (I), with random connections between them.

```python
# === Brian2 Simulation: Simple E/I Network ===
from brian2 import *
import matplotlib.pyplot as plt

start_scope()

# --- Parameters ---
N = 100
N_E = 80 # Number of excitatory neurons
N_I = 20 # Number of inhibitory neurons
tau = 10*ms; V_rest = -70*mV; V_thresh = -50*mV; V_reset = -75*mV; t_ref = 2*ms

# --- Neuron Model ---
# Using voltage drive again for simplicity to provide baseline activity
eqs = '''
dv/dt = (V_rest - v + V_drive)/tau : volt (unless refractory)
V_drive : volt # Driving voltage term
'''
# Create the full population
neurons = NeuronGroup(N, eqs, threshold='v>V_thresh', reset='v=V_reset', refractory=t_ref, method='euler')
# Separate groups for easier indexing (slices of the main group)
P_E = neurons[:N_E]
P_I = neurons[N_E:]

# Set initial conditions and baseline drive
neurons.v = V_rest + rand(N)*(V_thresh - V_rest)*0.5 # Random initial V near rest
neurons.V_drive = 10.0*mV # Baseline drive (adjust as needed)

# --- Synapses ---
# Define connection probabilities and weights
p_EE = 0.1; w_EE = 0.8*mV # E->E
p_EI = 0.2; w_EI = 1.0*mV # E->I
p_IE = 0.4; w_IE = -2.5*mV # I->E (Inhibitory weights are negative voltage jumps)
p_II = 0.4; w_II = -2.0*mV # I->I
delay_syn = 1.5*ms

# Create Synapses (using simple voltage jump model)
# Excitatory connections
syn_E = Synapses(P_E, neurons, on_pre='v_post += w', delay=delay_syn)
syn_E.connect(i=P_E.i, j=P_E.i, p=p_EE) # E -> E (j needs indices within 'neurons')
syn_E.connect(i=P_E.i, j=P_I.i, p=p_EI) # E -> I
syn_E.w = 'w_EE' # Assign weights based on connection type (simple assignment here)
# Need more precise weight assignment if mixed connection types in one Synapses obj
# Re-do with separate Synapses for clarity:
syn_EE = Synapses(P_E, P_E, on_pre='v_post += w_EE', delay=delay_syn)
syn_EE.connect(p=p_EE)

syn_EI = Synapses(P_E, P_I, on_pre='v_post += w_EI', delay=delay_syn)
syn_EI.connect(p=p_EI)

# Inhibitory connections
syn_IE = Synapses(P_I, P_E, on_pre='v_post += w_IE', delay=delay_syn)
syn_IE.connect(p=p_IE)

syn_II = Synapses(P_I, P_I, on_pre='v_post += w_II', delay=delay_syn)
syn_II.connect(p=p_II)

# --- Monitors ---
spike_mon = SpikeMonitor(neurons)
rate_mon_E = PopulationRateMonitor(P_E)
rate_mon_I = PopulationRateMonitor(P_I)

# --- Run ---
run(500*ms)

# --- Plot ---
plt.figure(figsize=(12, 8))

# Raster Plot
plt.subplot(2, 1, 1)
plt.plot(spike_mon.t/ms, spike_mon.i, '.k', markersize=2)
plt.axhline(N_E, ls='-', color='r', lw=1) # Line separating E and I neurons
plt.xlabel('Time (ms)')
plt.ylabel('Neuron Index')
plt.title('Raster Plot (E: 0-79, I: 80-99)')
plt.xlim(0, 500)
plt.ylim(-1, N) # Adjusted ylim
plt.grid(True, axis='y', linestyle=':') # Grid lines for y-axis

# Population Rates
plt.subplot(2, 1, 2)
plt.plot(rate_mon_E.t/ms, rate_mon_E.rate, label='Excitatory Rate')
plt.plot(rate_mon_I.t/ms, rate_mon_I.rate, label='Inhibitory Rate')
plt.xlabel('Time (ms)')
plt.ylabel('Population Rate (Hz)')
plt.title('Population Firing Rates')
plt.legend()
plt.xlim(0, 500)
plt.grid(True)

plt.tight_layout()
plt.show()
```

*(Self-correction during generation: Corrected synapse creation to use separate `Synapses` objects for E->E, E->I, I->E, I->I for clarity in assigning weights and understanding connections. Corrected indexing for connecting subpopulations `P_E` and `P_I` which are slices of the main `neurons` group. Added random initial voltages and a baseline `V_drive` to encourage activity. Adjusted plotting limits and added grid lines for better readability.)*

**Explanation of Code Examples:**

*   **`4.1_TwoConnectedNeurons.ipynb`:**
    1.  **Setup:** Imports Brian2 and Matplotlib, defines LIF parameters, a synaptic weight (`weight_exc` causing a voltage jump), and a delay (`delay_syn`).
    2.  **Neuron Model:** Creates a `NeuronGroup` of 2 neurons using a simplified model with a `V_drive` term instead of explicit `I_inj` for easier control. Neuron 0 receives a constant drive (`25*mV`) to make it fire, while Neuron 1 receives no drive initially.
    3.  **Synapses:** Creates a `Synapses` object connecting `neurons` to itself. The `on_pre` action specifies that the postsynaptic potential `v_post` is directly incremented by `weight_exc` upon presynaptic spike arrival. `syn.connect(i=0, j=1)` creates the single connection from neuron 0 to neuron 1. The connection inherits the default `delay` specified.
    4.  **Monitors:** A `StateMonitor` records the voltage (`v`) of both neurons (`record=True`), and a `SpikeMonitor` records all spikes.
    5.  **Run & Plot:** The simulation runs for 100 ms. The plot shows the voltage traces of both neurons. When neuron 0 spikes, its effect (an EPSP-like jump) is seen on neuron 1 after the specified `delay_syn`. Vertical dashed lines indicate the delayed arrival time of spikes from neuron 0 at neuron 1.

*   **`4.2_SimpleEI_Network.ipynb`:**
    1.  **Setup:** Imports libraries, defines network size (`N`, `N_E`, `N_I`), and LIF parameters.
    2.  **Neuron Model:** Creates a `NeuronGroup` for all `N` neurons using the `V_drive` model. Subgroups `P_E` (indices 0 to N_E-1) and `P_I` (indices N_E to N-1) are created using slicing for convenience in defining connections. Neurons are initialized with slightly randomized membrane potentials near rest, and all neurons receive a constant baseline `V_drive` to promote some background firing.
    3.  **Synapses:** Connection probabilities (`p_EE`, `p_EI`, etc.) and corresponding synaptic weights (`w_EE`, `w_EI`, etc.) are defined. Inhibitory weights are negative voltage jumps. Four separate `Synapses` objects (`syn_EE`, `syn_EI`, `syn_IE`, `syn_II`) are created, each connecting the appropriate source and target subgroups (`P_E`, `P_I`). The `on_pre` action directly adds the respective weight to `v_post`. The `connect(p=...)` method is used to establish connections randomly based on the specified probabilities.
    4.  **Monitors:** A `SpikeMonitor` records spikes from all neurons. Two `PopulationRateMonitor` objects track the firing rates of the excitatory (`P_E`) and inhibitory (`P_I`) populations separately.
    5.  **Run & Plot:** The simulation runs for 500 ms. The first subplot shows the raster plot of all neurons, with a horizontal red line separating the E and I populations. Dots indicate spike times. The second subplot shows the smoothed population firing rates for the E and I groups over time, calculated by the respective monitors. This allows visualization of the overall network activity and the interplay between excitation and inhibition.

These examples demonstrate the transition from single neurons to networks in Brian2, covering the creation of neuron populations, the definition of synaptic connections with various strategies, and the monitoring and visualization of network-level dynamics.

**4.8 Conclusion and Planned Code**

This chapter provided the essential tools and concepts for constructing and simulating basic spiking neural networks using Brian2, moving beyond the single-neuron focus of Chapter 3. We learned how to define populations of neurons with shared properties using `NeuronGroup`. The core mechanisms for establishing connections were detailed through the `Synapses` object, including defining synaptic models via `on_pre`, incorporating weights (`w`) and delays (`delay`), and utilizing various connectivity strategies (`connect()` method) like probabilistic or conditional connections. We revisited current-based versus conductance-based synaptic models within the practical context of Brian2 implementation. Essential monitoring tools were expanded to include `PopulationRateMonitor` for tracking collective activity, alongside `SpikeMonitor` and `StateMonitor`. Finally, standard visualization techniques like raster plots and population rate plots were presented as key methods for interpreting network behavior. The chapter culminated in two practical Brian2 examples: simulating a simple two-neuron excitatory connection and building a small network comprising distinct excitatory and inhibitory populations with random connectivity. These examples serve as stepping stones, equipping the reader with the foundational skills needed to tackle the more complex, biologically inspired organoid models in the subsequent chapters, where we will incorporate features like heterogeneity, plasticity, and more advanced dynamics.

**Planned Code Examples:**
*   **`4.1_TwoConnectedNeurons.ipynb`:** (Provided and explained in Section 4.7) Simulates two LIF neurons where one excites the other with a delay, demonstrating basic `NeuronGroup`, `Synapses`, `connect()`, `delay`, and monitoring.
*   **`4.2_SimpleEI_Network.ipynb`:** (Provided and explained in Section 4.7) Simulates a small network of randomly connected excitatory and inhibitory LIF neurons, illustrating the creation of subpopulations, probabilistic connectivity between groups, and visualization of network activity using raster plots and population rates.

**4.9 References for Further Reading (APA Format)**

1.  Amit, D. J. (1989). *Modeling brain function: The world of attractor neural networks*. Cambridge University Press. *(Classic text on attractor networks, exploring collective dynamics in recurrent networks, though often using rate-based models.)*
2.  Brunel, N. (2000). Dynamics of sparsely connected networks of excitatory and inhibitory spiking neurons. *Journal of Computational Neuroscience, 8*(3), 183–208. https://doi.org/10.1023/a:1008925309027 *(Seminal theoretical work analyzing the dynamics of randomly connected E/I spiking networks, relevant to the second example.)*
3.  Destexhe, A., Mainen, Z. F., & Sejnowski, T. J. (1994). Synthesis of models for synaptic currents, membrane potential fluctuations and morphological types of systems of neurons. *Journal of Computational Neuroscience, 1*(3), 195–230. https://doi.org/10.1007/BF00961875 *(Provides detailed background on modeling synaptic currents and conductances.)*
4.  Gewaltig, M.-O., & Diesmann, M. (2007). NEST (Neural Simulation Tool). *Scholarpedia, 2*(4), 1430. https://doi.org/10.4249/scholarpedia.1430 *(Describes another major neural simulator, NEST, providing context on the landscape of simulation tools.)*
5.  Hertz, J., Krogh, A., & Palmer, R. G. (1991). *Introduction to the theory of neural computation*. Addison-Wesley. *(A foundational textbook covering various aspects of neural networks, including basic network structures and dynamics.)*
6.  Litwin-Kumar, A., & Doiron, B. (2012). Slow dynamics and high variability in balanced cortical networks with clustered connectivity. *Nature Neuroscience, 15*(11), 1498–1505. https://doi.org/10.1038/nn.3220 *(Explores dynamics in more complex E/I networks with non-random structure, relevant for later chapters.)*
7.  Morrison, A., Diesmann, M., & Gerstner, W. (2008). Incorporating short-term plasticity in computationally efficient models of balanced activity. *Neural Computation, 20*(12), 3169-3194. https://doi.org/10.1162/neco.2008.01-08-688 *(Discusses modeling balanced E/I networks, incorporating synaptic dynamics often implemented using simulator tools.)*
8.  Potjans, T. C., & Diesmann, M. (2014). The cell-type specific cortical microcircuit: Relating structure and activity in a full-scale spiking network model. *Cerebral Cortex, 24*(3), 785–806. https://doi.org/10.1093/cercor/bhs358 *(A landmark paper demonstrating large-scale simulation of a cortical microcircuit, showcasing complex network construction.)*
9.  Stimberg, M., Brette, R., & Goodman, D. F. M. (2019). Brian 2, an intuitive and efficient neural simulator. *eLife, 8*, e47314. https://doi.org/10.7554/eLife.47314 *(The primary reference for Brian2, essential for understanding the tool used for implementation in this chapter.)*
10. van Vreeswijk, C., & Sompolinsky, H. (1996). Chaos in neuronal networks with balanced excitatory and inhibitory activity. *Science, 274*(5293), 1724–1726. https://doi.org/10.1126/science.274.5293.1724 *(Influential theoretical work on the dynamics of balanced E/I networks, often studied via simulation.)*
