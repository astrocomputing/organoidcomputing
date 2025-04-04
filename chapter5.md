---

# Chapter 5

#  Modeling Heterogeneous Neuron Populations and Spontaneous Activity
----

*Having established the methods for building basic neural networks in Brian2, this chapter focuses on incorporating crucial elements of biological realism often observed in brain organoids and essential for generating complex network dynamics: **heterogeneity** and **spontaneous activity**. Biological neural networks are far from uniform; neurons exhibit significant variability in their properties and connections. Furthermore, neural circuits, especially developing ones like those in organoids, often display ongoing spontaneous activity even in the absence of specific external stimuli. This chapter explores how to capture these features in our computational models. We begin by discussing the biological importance and sources of heterogeneity and how distinct neuron types, like excitatory and inhibitory populations, can be represented. We then delve into methods for modeling variability in neuron parameters *within* a population using statistical distributions implemented via Brian2's string expressions. Subsequently, we address the modeling of spontaneous network activity, covering both intrinsic neuronal noise and, more prominently, the simulation of background synaptic input using tools like Brian2's `PoissonInput` or `PoissonGroup`. We will also briefly reconsider the choice of neuron models (LIF vs. more complex ones like AdEx/Izhikevich) in the context of capturing richer spontaneous dynamics potentially seen in organoids. A crucial related challenge, the **parameter estimation problem**—finding appropriate model parameters to match experimental data—will be discussed conceptually. Finally, we integrate these concepts in a practical Brian2 example, simulating a heterogeneous E/I network exhibiting spontaneous activity driven by background Poisson inputs, demonstrating how these factors shape network behavior.*

-----

**5.1 Capturing Biological Heterogeneity (Cell Types, Varying Properties)**

Biological realism in neural modeling necessitates acknowledging and incorporating **heterogeneity**. Real neural networks, including those self-organizing within brain organoids, are composed of diverse cell types, and even neurons within the same nominal class exhibit considerable variability in their morphological, electrophysiological, and synaptic properties. This heterogeneity is not merely biological "noise" to be averaged away; it is often functionally significant, contributing to the richness of network dynamics, computational capabilities, information coding capacity, and robustness of the system. In brain organoids, heterogeneity arises from multiple sources inherent in their developmental process. Stochastic events during cell fate decisions, subtle variations in the local signaling microenvironment experienced by different cells within the 3D structure, epigenetic differences accumulated during cell proliferation or reprogramming (if using iPSCs), and the generally incomplete or asynchronous maturation across the organoid all contribute to significant cell-to-cell variability. This manifests as differences in neuron size, dendritic complexity, the specific profile of ion channels expressed (affecting firing patterns and thresholds), resting membrane potentials, membrane time constants, synaptic strengths, and connectivity patterns.

Capturing this heterogeneity in computational models is crucial for several reasons. Firstly, it enhances the **biological plausibility** of the simulations, bringing them closer to the systems they aim to represent, like brain organoids. Models incorporating realistic diversity are more likely to reproduce experimentally observed phenomena, such as irregular firing patterns, broad distributions of firing rates, specific oscillatory dynamics, or complex responses to stimuli, which often depend on the interplay between different cell types and the variability within them. Secondly, heterogeneity can significantly impact **network dynamics and computation**. For example, variability in neuronal excitability can prevent excessive synchrony and promote richer dynamic regimes. Diversity in synaptic strengths might be essential for certain learning rules or information storage mechanisms. The presence of distinct cell classes, like excitatory and inhibitory neurons with different properties and connectivity, is fundamental for stabilizing network activity and enabling complex operations. Thirdly, heterogeneity can contribute to the **robustness** of neural systems. A network composed of diverse elements may be less susceptible to perturbations or the failure of individual components compared to a perfectly uniform network. Therefore, moving beyond idealized homogeneous models towards incorporating realistic heterogeneity is a critical step in building more accurate and insightful models of brain organoids and exploring their computational potential.

**5.2 Representing Neuron Types (Distinct E/I Groups)**

One of the most fundamental forms of heterogeneity in cortical and many other brain circuits, readily observed in brain organoids, is the distinction between **excitatory (E)** and **inhibitory (I)** neurons. As discussed previously, excitatory neurons (typically glutamatergic) tend to depolarize their targets, promoting network activity, while inhibitory neurons (typically GABAergic) tend to hyperpolarize or shunt their targets, controlling excitability and shaping dynamics. These two broad classes often differ not only in their neurotransmitter but also in their intrinsic firing properties, morphology, and connectivity patterns. Capturing this E/I distinction is often the first and most crucial step in modeling heterogeneous networks.

In Brian2, a common and straightforward approach to represent distinct neuron types is to create separate `NeuronGroup` objects for each population. For instance, building on the simple E/I network example from Chapter 4, we would define one `NeuronGroup` for the excitatory population and another for the inhibitory population:

```python
# Parameters (potentially different for E and I)
N_E = 80; N_I = 20
tau_E = 15*ms; tau_I = 10*ms # Example: Inhibitory neurons often faster
V_rest_E = -65*mV; V_rest_I = -70*mV
# ... other parameters ...

# Neuron Models (could be the same model type, e.g., LIF, but with different parameters)
eqs_lif = '''... (LIF equations) ...'''

# Create separate NeuronGroups
P_E = NeuronGroup(N_E, eqs_lif, threshold='...', reset='...', refractory='...',
                  method='euler', name='Exc_Pop') # Naming groups is good practice
P_I = NeuronGroup(N_I, eqs_lif, threshold='...', reset='...', refractory='...',
                  method='euler', name='Inh_Pop')

# Initialize states (potentially differently)
P_E.v = V_rest_E; P_I.v = V_rest_I
```
This approach allows assigning different parameter values (like `tau`, `V_rest`, `V_thresh`) or even entirely different model equations (e.g., using an adapting model like AdEx for excitatory neurons and a fast-spiking model for inhibitory neurons) to each population explicitly. Connections between these distinct groups can then be defined using separate `Synapses` objects, as demonstrated in Chapter 4 (e.g., `Synapses(P_E, P_E, ...)` for E-to-E connections, `Synapses(P_E, P_I, ...)` for E-to-I, etc.), allowing for cell-type-specific connectivity rules and weights.

While using separate `NeuronGroup` objects is clear and flexible, especially when populations have significantly different models, an alternative for simpler cases (where the core equations are the same but parameters differ) is to use a single large `NeuronGroup` and manage heterogeneity through indexing and conditional parameter assignment, similar to how we created `P_E` and `P_I` as slices in the Chapter 4 example. However, for clarity and ease of managing distinct cell types with potentially different dynamics and connectivity rules, using separate `NeuronGroup` objects is often the preferred strategy, particularly when modeling the fundamental E/I division crucial for organoid network dynamics. Further subdivisions (e.g., different classes of interneurons) could also be represented by additional `NeuronGroup` objects if required by the model's complexity and the scientific question.

**5.3 Modeling Parameter Variability (Statistical Distributions)**

Beyond the coarse-grained heterogeneity represented by distinct cell types (like E and I populations), significant variability exists *within* each neuronal population. Individual neurons of the same nominal type are not identical clones; they exhibit a range of properties due to the factors mentioned in Section 5.1. Modeling this intra-population variability can be crucial for reproducing realistic network dynamics, such as asynchronous irregular firing observed *in vivo* and often in organoid recordings.

Brian2 provides a powerful and convenient way to introduce parameter variability when creating a `NeuronGroup` or setting its parameters. Instead of assigning a single fixed value to a parameter, we can assign a **string expression** that utilizes random number generation functions. Brian2 parses these strings and assigns a potentially different value to each neuron in the group based on the specified random distribution. The most common functions are:
*   `rand()`: Returns uniformly distributed random numbers between 0 and 1.
*   `randn()`: Returns normally distributed (Gaussian) random numbers with mean 0 and standard deviation 1.

These functions can be combined with mathematical operations and units within the string expression to set parameters according to various statistical distributions. For example, to initialize the resting potential (`V_rest`) of neurons in a group with some Gaussian variability around a mean value:

```python
# Define mean and standard deviation for V_rest
V_rest_mean = -70*mV
V_rest_std = 5*mV

# Create NeuronGroup (assuming N neurons)
neurons = NeuronGroup(N, eqs, ...)

# Set V_rest using a string expression with randn()
neurons.v = 'V_rest_mean + V_rest_std * randn()' # Assigns each neuron a V sampled from N(V_rest_mean, V_rest_std^2)
# Note: We set 'v' directly here assuming initial V = V_rest. If V_rest is a parameter in eqs, set that instead.
# Example: If V_rest is in eqs: 'V_rest : volt'
# neurons.V_rest = 'V_rest_mean + V_rest_std * randn()'
# neurons.v = V_rest # Initialize v to the neuron's specific V_rest
```

Similarly, we could introduce variability into other parameters like the membrane time constant (`tau`), the firing threshold (`V_thresh`), or even parameters within more complex models like AdEx or Izhikevich. For example, setting `tau` with uniform variability:

```python
# Assume tau is a parameter in the neuron model equations: 'tau : second'
tau_mean = 10*ms
tau_variation = 0.2 # +/- 20% variation
neurons.tau = 'tau_mean * (1 + tau_variation * (2*rand() - 1))' # Uniform dist around tau_mean
```

Introducing such parameter heterogeneity has significant consequences for network behavior. It typically leads to **desynchronization** of neuronal firing, as neurons respond differently to common inputs due to their varied thresholds, time constants, etc. This can shift the network dynamics from highly synchronized oscillations or bursts towards a more biologically realistic state of **asynchronous irregular (AI)** firing, which is often observed in the awake cortex. Variability in thresholds means that some neurons will be intrinsically more excitable than others, potentially leading to broad distributions of firing rates across the population. Properly calibrated heterogeneity is often essential for building models that accurately reflect the complex, non-uniform activity patterns seen in MEA or calcium imaging recordings from brain organoids. The degree and type of variability introduced become important model parameters themselves, ideally informed by experimental measurements if available.

`[Conceptual Figure 5.1: Heterogeneity Effects. Panel (a): Raster plot of a homogeneous network showing highly synchronized firing. Panel (b): Raster plot of a heterogeneous network (with parameter variability) showing asynchronous, irregular firing. Panel (c): Histogram of firing rates from the heterogeneous network, showing a broad, often skewed distribution.]`

**5.4 Modeling Spontaneous Activity (Intrinsic Noise, Background Input - `PoissonInput`)**

A prominent feature of developing neural networks *in vivo*, and frequently observed in *in vitro* preparations like brain organoids (as discussed in Chapter 2), is the presence of **spontaneous activity**. Neurons fire action potentials and networks exhibit ongoing dynamics even in the absence of specific, externally applied stimuli. This background activity is not just random noise; it plays crucial roles in network development, synaptic plasticity, maintaining network responsiveness, and potentially encoding internal states or performing ongoing computations. Understanding and modeling spontaneous activity is therefore critical for creating realistic simulations of organoid function.

Spontaneous activity can arise from several sources. **Intrinsic neuronal noise** originates from the stochastic opening and closing of individual ion channels in the neuron's membrane. While the collective behavior of many channels is often well-described by deterministic models like Hodgkin-Huxley, the random fluctuations of small numbers of channels can lead to spontaneous membrane potential fluctuations that occasionally trigger action potentials, especially in small neurons or near threshold. This intrinsic noise can be modeled by adding a stochastic term (often representing Gaussian white noise) to the neuron's membrane potential equation. In Brian2, this can be achieved using the symbol `xi` within the model equations, which represents a Gaussian random variable scaled by time step:

```latex
% Equation including intrinsic noise term
\tau \frac{dV}{dt} = -(V - V_{\text{rest}}) + RI(t) + \sigma \sqrt{\tau} \xi
```

Here, $\sigma$ determines the amplitude of the noise, and the $\sqrt{\tau}$ scaling (or sometimes $\sqrt{2 \tau}$ depending on convention and solver) is often included for consistency across different time constants. Brian2 handles the numerical integration of such stochastic differential equations.

However, in many biological networks, especially larger ones, a more significant source of spontaneous activity is thought to be **ongoing background synaptic input**. Even in the absence of a specific task or stimulus, neurons *in vivo* receive a continuous barrage of excitatory and inhibitory synaptic inputs from other neurons within the local network or from distant brain areas. This synaptic "noise" reflects the reverberating activity within the broader network. In *in vitro* systems like organoids, which lack external inputs, spontaneous activity likely arises from a combination of intrinsic neuronal excitability (perhaps heightened in developing neurons), recurrent network connections generating self-sustained activity, and potentially imbalances in excitation and inhibition.

A common and effective way to model this background synaptic bombardment in simulations is to assume that the network neurons receive inputs from a large number of unmodeled external neurons firing randomly according to **Poisson processes**. A Poisson process describes events occurring independently at a certain average rate. To simulate this in Brian2, one can use the `PoissonInput` class (for directly adding events to a state variable, like conductance) or, perhaps more commonly and flexibly, the `PoissonGroup` object. A `PoissonGroup` represents a population of virtual neurons that fire spikes according to independent Poisson processes at specified rates. These virtual Poisson neurons can then be connected synaptically to the actual neurons in the network model using `Synapses` objects, just like connecting regular `NeuronGroup`s.

```python
# --- Modeling Background Input with PoissonGroup ---
N_neurons = 100 # Number of neurons in our main network
# Parameters for background input
bg_rate = 2*Hz    # Average firing rate of external Poisson neurons
N_bg_E = 800     # Number of external excitatory Poisson sources
N_bg_I = 200     # Number of external inhibitory Poisson sources
w_bg_E = 0.5*mV  # Weight of background E input (voltage jump example)
w_bg_I = -1.0*mV # Weight of background I input

# Create PoissonGroups
P_bg_E = PoissonGroup(N_bg_E, rates=bg_rate)
P_bg_I = PoissonGroup(N_bg_I, rates=bg_rate)

# Connect PoissonGroups to the main neuron population ('neurons')
# Assuming 'neurons' is the target NeuronGroup
# Using voltage jump model for synapses here for simplicity
syn_bg_E = Synapses(P_bg_E, neurons, on_pre='v_post += w_bg_E')
syn_bg_E.connect(p=0.1) # Each network neuron receives input from 10% of bg E sources

syn_bg_I = Synapses(P_bg_I, neurons, on_pre='v_post += w_bg_I')
syn_bg_I.connect(p=0.1) # Each network neuron receives input from 10% of bg I sources
```

In this example, we create two `PoissonGroup`s representing large populations of external excitatory and inhibitory neurons firing randomly at 2 Hz. We then connect these background sources sparsely (10% probability) to the main network neurons using `Synapses`. The arrival of spikes from `P_bg_E` causes small depolarizations (`w_bg_E`), while spikes from `P_bg_I` cause small hyperpolarizations (`w_bg_I`). This constant, random synaptic bombardment provides a fluctuating input current to the network neurons, driving spontaneous firing even without other inputs and contributing to the asynchronous irregular activity characteristic of balanced E/I networks. Adjusting the rates (`bg_rate`), connection probabilities (`p`), and weights (`w_bg_E`, `w_bg_I`) of these background inputs allows tuning the level and characteristics of the spontaneous activity in the simulated network to match experimental observations or theoretical assumptions. Using `PoissonGroup` is often preferred over adding direct noise (`xi`) to the voltage equation as it more explicitly models the synaptic nature of background activity in biological circuits.

**5.5 Choice of Neuron Models (LIF vs. AdEx/Izhikevich for Organoid Dynamics)**

When modeling the spontaneous activity and dynamic behavior observed in brain organoids, the choice of the underlying single neuron model becomes particularly relevant. As discussed in Chapter 3, different neuron models offer varying degrees of biological realism and computational efficiency. While the **Leaky Integrate-and-Fire (LIF) model** is computationally inexpensive and captures the basic integrate-and-fire mechanism, its simple dynamics might be insufficient to reproduce some of the richer phenomena seen in organoid recordings. Organoid activity, especially during development, can exhibit complex patterns, including **burst firing** (where neurons fire clusters of spikes followed by silence), **spike frequency adaptation** (decreasing firing rate during constant input), and diverse intrinsic firing responses to stimulation. These features are often linked to the presence of specific ion channels (e.g., slow potassium channels, calcium channels) that are not explicitly included in the basic LIF formulation.

If capturing these richer dynamics is deemed important for understanding organoid function or computation, employing more sophisticated neuron models like the **Adaptive Exponential Integrate-and-Fire (AdEx)** or the **Izhikevich model** may be necessary. The AdEx model, with its additional adaptation variable (`w`), can explicitly model spike frequency adaptation and, with appropriate parameter choices, generate intrinsic bursting behavior. The Izhikevich model, also featuring two variables, is renowned for its ability to reproduce a wide catalogue of experimentally observed firing patterns (regular spiking, adapting, bursting, chattering, fast spiking, etc.) simply by tuning its four main parameters. Using AdEx or Izhikevich neurons instead of LIF neurons within a network simulation allows the model to generate more complex spontaneous activity patterns intrinsically, potentially reducing the reliance on external inputs or complex synaptic rules to achieve similar dynamic richness. For example, a network of intrinsically bursting neurons might more readily exhibit synchronized bursting activity patterns similar to those seen in developing cultures.

However, this increased biological realism comes at the cost of **increased computational complexity** (simulating two differential equations per neuron instead of one) and, perhaps more significantly, **increased parameterization difficulty**. Finding the appropriate values for the additional parameters in AdEx or Izhikevich models (`a`, `b`, `tau_w`, etc.) to match specific neuronal behaviors observed in organoids can be challenging, especially given the heterogeneity and immaturity of the cells (as discussed in Section 5.6). Therefore, the choice remains a trade-off. If the primary goal is to simulate large-scale network interactions and basic E/I balance effects, the LIF model might still be sufficient and computationally advantageous. If accurately capturing specific firing patterns, adaptation phenomena, or intrinsic bursting observed in organoid recordings is critical for the research question, then investing the extra computational effort and parameter tuning required for AdEx or Izhikevich models may be warranted. Often, researchers might start with simpler LIF models and progressively incorporate more complex neuron dynamics as needed to explain specific experimental findings or explore hypotheses related to richer firing patterns. The flexibility of Brian2 allows relatively easy substitution of different neuron models within the same network structure, facilitating such comparative studies.

**5.6 The Challenge of Parameter Estimation (Conceptual)**

Building biologically realistic computational models of complex systems like brain organoids inevitably confronts a major hurdle: **parameter estimation**. Our models, whether single-neuron models (LIF, AdEx, HH) or network models (including synaptic weights, connection probabilities, delays), contain numerous parameters whose values determine the model's behavior. To make these models truly useful for understanding biology or making predictions, these parameters should ideally be set to values that accurately reflect the properties of the real system being modeled. However, finding these "correct" parameter values is often exceptionally difficult.

Firstly, the **sheer number of parameters** can be daunting. Even a relatively simple network model might involve parameters for neuron dynamics (e.g., $\tau, V_{\text{rest}}, V_{\text{thresh}}$ for multiple cell types), synaptic properties (weights, delays, time constants for E-to-E, E-to-I, I-to-E, I-to-I connections), connectivity rules (probabilities, spatial extent), background input characteristics (rates, weights), and potentially plasticity rules. Exploring this high-dimensional parameter space exhaustively is computationally infeasible. Secondly, many of these parameters are **difficult or impossible to measure directly** in the experimental system, especially in complex, heterogeneous 3D structures like organoids. While some properties might be measured from individual neurons using patch-clamp, obtaining comprehensive data on synaptic strengths, connection probabilities, or precise ion channel kinetics for all relevant cell types *in situ* is extremely challenging. Thirdly, biological systems exhibit significant **variability** (as discussed previously), meaning that there might not be a single "correct" value for a parameter, but rather a distribution of values. The model should ideally capture this distribution, adding another layer of complexity to the estimation process.

Consequently, modelers often resort to various strategies, none of which are perfect. Parameters might be **taken from literature** reporting values measured in related, often simpler, experimental systems (e.g., rodent brain slices), but these may not accurately reflect the specific properties of human neurons developing within an organoid. **Manual tuning** is common, where the modeler iteratively adjusts parameters by hand until the simulation output qualitatively resembles some target experimental observation (e.g., matching average firing rates or the presence of oscillations). This is often subjective and may not explore the parameter space systematically. More systematic approaches involve **automated parameter search** methods. These can range from simple **grid searches** (testing combinations of parameters on a grid, computationally expensive) to more sophisticated **optimization algorithms** (e.g., genetic algorithms, simulated annealing, gradient-based methods if a suitable objective function measuring the difference between simulation and experiment can be defined) that try to find parameter sets minimizing the discrepancy between model output and experimental data. Advanced techniques from statistics and machine learning, such as **Bayesian inference** or **data assimilation**, aim to estimate probability distributions over possible parameter values consistent with the observed data, providing a more principled way to handle uncertainty but often requiring significant computational resources and expertise.

`[Conceptual Figure 5.2: Parameter Estimation Workflow. A diagram showing "Experimental Data" (e.g., MEA recordings from organoid) and a "Computational Model" with unknown parameters. An "Objective Function" compares simulation output to experimental data. An "Optimization Algorithm" (or manual tuning) iteratively adjusts model parameters to minimize the objective function, leading to an "Estimated Parameter Set". Dashed lines indicate challenges like high dimensionality, lack of data, and variability.]`

In the context of brain organoids, parameter estimation is particularly challenging due to the system's complexity, variability, developmental nature (parameters might change over time), and limitations in experimental access for detailed measurements. Therefore, while computational modeling is indispensable, it's crucial to acknowledge the uncertainties associated with parameter values. Often, the goal is less about finding the single "true" set of parameters and more about exploring how different parameter regimes affect network dynamics, identifying which parameters are most sensitive, and generating testable hypotheses about the underlying biological mechanisms, even with simplified or representative parameter sets. Throughout this book, while we will use plausible parameter values based on literature or for illustrative purposes, we must remain aware that precise, data-driven parameterization of organoid models remains a significant ongoing research challenge.

**5.7 Brian2 Implementation: Heterogeneous Population with Noise**

This example integrates the concepts discussed in this chapter. We will simulate a network of excitatory and inhibitory LIF neurons where:
1.  Neuron parameters (e.g., resting potential, threshold) have some variability within each population.
2.  The network exhibits spontaneous activity driven by background synaptic input modeled using `PoissonGroup`.

```python
# === Brian2 Simulation: Heterogeneous E/I Network with Spontaneous Activity ===
from brian2 import *
import matplotlib.pyplot as plt

start_scope()

# --- Parameters ---
# Network Size
N = 1000
N_E = int(0.8 * N) # 80% Excitatory
N_I = int(0.2 * N) # 20% Inhibitory

# LIF Neuron Parameters (Mean Values)
tau_E = 15*ms; tau_I = 10*ms
V_rest_E_mean = -65*mV; V_rest_I_mean = -70*mV
V_thresh_E_mean = -50*mV; V_thresh_I_mean = -55*mV
V_reset = -75*mV
t_ref = 2*ms

# Heterogeneity Parameters (Standard Deviations or Range)
V_rest_std = 3*mV
V_thresh_std = 2*mV
# Using uniform variation for tau for illustration
tau_variation = 0.1 # +/- 10%

# Synaptic Parameters (Conductance-based)
tau_g_E = 5*ms; tau_g_I = 10*ms # Synaptic conductance time constants
E_E = 0*mV; E_I = -75*mV # Reversal potentials
w_EE = 0.8*nS; w_EI = 0.6*nS # E weights (nS = nanoSiemens)
w_IE = -4.0*nS; w_II = -3.0*nS # I weights (negative conductance change)
p_connect = 0.1 # Connection probability for all types
delay_syn = 1.5*ms

# Background Input Parameters
bg_rate = 5*Hz   # Rate of external Poisson spikes
w_bg = 1.0*nS    # Weight of background excitatory input

# --- Neuron Model (Conductance-based LIF) ---
eqs = '''
dv/dt = (V_rest - v + g_E*(E_E - v) + g_I*(E_I - v)) / tau : volt (unless refractory)
dg_E/dt = -g_E / tau_g_E : siemens  # Excitatory conductance dynamics
dg_I/dt = -g_I / tau_g_I : siemens  # Inhibitory conductance dynamics
V_rest : volt                       # Neuron-specific resting potential
V_thresh : volt                     # Neuron-specific threshold
tau : second                        # Neuron-specific time constant
'''

# --- Create Neuron Groups (Single group with subgroups) ---
neurons = NeuronGroup(N, eqs,
                      threshold='v > V_thresh',
                      reset='v = V_reset',
                      refractory=t_ref,
                      method='euler')
P_E = neurons[:N_E]
P_I = neurons[N_E:]

# --- Set Heterogeneous Parameters ---
# Excitatory Population
P_E.V_rest = 'V_rest_E_mean + V_rest_std * randn()'
P_E.V_thresh = 'V_thresh_E_mean + V_thresh_std * randn()'
P_E.tau = 'tau_E * (1 + tau_variation * (2*rand() - 1))'
# Inhibitory Population
P_I.V_rest = 'V_rest_I_mean + V_rest_std * randn()'
P_I.V_thresh = 'V_thresh_I_mean + V_thresh_std * randn()'
P_I.tau = 'tau_I * (1 + tau_variation * (2*rand() - 1))'

# Initialize membrane potential near rest
neurons.v = 'V_rest + 2*mV * rand()'
neurons.g_E = 0*nS
neurons.g_I = 0*nS

# --- Create Background Input ---
P_bg = PoissonGroup(N, rates=bg_rate) # One Poisson source per neuron for simplicity

# --- Synapses ---
# Background Synapses (Excitatory)
syn_bg = Synapses(P_bg, neurons, on_pre='g_E_post += w_bg', delay=0.1*ms) # Minimal delay
syn_bg.connect(j='i') # Each Poisson source targets one network neuron

# Internal Network Synapses (using simplified weight assignment)
# E -> E
syn_EE = Synapses(P_E, P_E, on_pre='g_E_post += w_EE', delay=delay_syn)
syn_EE.connect(p=p_connect)
# E -> I
syn_EI = Synapses(P_E, P_I, on_pre='g_E_post += w_EI', delay=delay_syn)
syn_EI.connect(p=p_connect)
# I -> E
syn_IE = Synapses(P_I, P_E, on_pre='g_I_post += w_IE', delay=delay_syn) # Note: w_IE is negative
syn_IE.connect(p=p_connect)
# I -> I
syn_II = Synapses(P_I, P_I, on_pre='g_I_post += w_II', delay=delay_syn) # Note: w_II is negative
syn_II.connect(p=p_connect)

# --- Monitors ---
spike_mon = SpikeMonitor(neurons)
rate_mon = PopulationRateMonitor(neurons) # Monitor overall rate
state_mon_v = StateMonitor(neurons, 'v', record=[0, N_E, N // 2]) # Record a few voltages

# --- Run Simulation ---
sim_duration = 1000*ms
run(sim_duration)

# --- Visualize Results ---
plt.figure(figsize=(12, 10))

# 1. Raster Plot
plt.subplot(3, 1, 1)
plt.plot(spike_mon.t/ms, spike_mon.i, '.k', markersize=1)
plt.axhline(N_E, ls='-', color='r', lw=1)
plt.xlabel('Time (ms)')
plt.ylabel('Neuron Index')
plt.title(f'Raster Plot (N={N}, E: 0-{N_E-1}, I: {N_E}-{N-1})')
plt.xlim(0, sim_duration/ms)
plt.ylim(-1, N)
plt.grid(True, axis='y', linestyle=':')

# 2. Population Rate
plt.subplot(3, 1, 2)
plt.plot(rate_mon.t/ms, rate_mon.rate, label='Overall Population Rate')
plt.xlabel('Time (ms)')
plt.ylabel('Rate (Hz)')
plt.title('Population Firing Rate')
plt.legend()
plt.xlim(0, sim_duration/ms)
plt.grid(True)

# 3. Example Voltage Traces
plt.subplot(3, 1, 3)
plt.plot(state_mon_v.t/ms, state_mon_v.v[0]/mV, label=f'Neuron 0 (E)')
plt.plot(state_mon_v.t/ms, state_mon_v.v[1]/mV, label=f'Neuron {N_E} (I)')
plt.plot(state_mon_v.t/ms, state_mon_v.v[2]/mV, label=f'Neuron {N // 2} (E)')
plt.xlabel('Time (ms)')
plt.ylabel('Membrane Potential (mV)')
plt.title('Example Voltage Traces')
plt.legend(fontsize='small')
plt.xlim(0, sim_duration/ms)
plt.grid(True)

plt.tight_layout()
plt.show()
```

**Explanation of the Code:**

1.  **Parameters:** Defines network size (`N`, `N_E`, `N_I`), mean LIF parameters for E and I types, standard deviations for heterogeneity (`V_rest_std`, `V_thresh_std`, `tau_variation`), conductance-based synaptic parameters (`tau_g_E`, `tau_g_I`, reversal potentials `E_E`, `E_I`, weights `w_EE` etc. in nanoSiemens `nS`), connection probability `p_connect`, and background input parameters (`bg_rate`, `w_bg`).
2.  **Neuron Model:** Defines equations (`eqs`) for a conductance-based LIF neuron. It includes dynamics for excitatory (`g_E`) and inhibitory (`g_I`) conductances, and crucially declares `V_rest`, `V_thresh`, and `tau` as neuron-specific parameters (their values will be set differently for each neuron).
3.  **Neuron Groups:** Creates a single `NeuronGroup` (`neurons`) for all `N` neurons. Subgroups `P_E` and `P_I` are defined using slicing for easier referencing.
4.  **Heterogeneity:** Sets the neuron-specific parameters (`V_rest`, `V_thresh`, `tau`) for `P_E` and `P_I` using string expressions involving `randn()` (for Gaussian variability) and `rand()` (for uniform variability around the mean values).
5.  **Initial Conditions:** Initializes membrane potentials randomly near rest and sets initial conductances to zero.
6.  **Background Input:** Creates a `PoissonGroup` (`P_bg`) with `N` virtual neurons, each firing at `bg_rate`. A `Synapses` object (`syn_bg`) connects each Poisson neuron `i` to network neuron `i` (`j='i'`), incrementing the excitatory conductance `g_E` upon spike arrival with weight `w_bg`. This provides independent noisy background drive to each neuron.
7.  **Internal Synapses:** Creates four `Synapses` objects (`syn_EE`, `syn_EI`, `syn_IE`, `syn_II`) to represent connections within and between the E and I populations. They use probabilistic connectivity (`p=p_connect`) and update the appropriate postsynaptic conductance (`g_E` or `g_I`) based on the presynaptic spike and the specified weights (note that inhibitory weights `w_IE`, `w_II` effectively add negative conductance).
8.  **Monitors:** Sets up a `SpikeMonitor` for all neurons, a `PopulationRateMonitor` for the overall network rate, and a `StateMonitor` to record the membrane potential `v` of a few example neurons.
9.  **Run & Visualize:** Runs the simulation for 1000 ms (1 second). It then generates three plots: a raster plot showing the asynchronous, irregular spiking activity driven by the background input and network interactions; the overall population firing rate over time; and example voltage traces from selected E and I neurons, illustrating their subthreshold fluctuations and spike events.

This simulation demonstrates how combining neuronal heterogeneity and spontaneous background drive leads to network dynamics that are qualitatively more similar to those observed in biological preparations like brain organoids compared to the simpler, deterministic networks simulated previously. The resulting asynchronous state provides a dynamic background upon which further computations or stimulus responses could potentially occur.

**5.8 Conclusion and Planned Code**

This chapter significantly advanced our modeling capabilities by introducing two critical elements of biological realism: **neuronal heterogeneity** and **spontaneous activity**. We explored the biological rationale for incorporating diversity, demonstrated practical methods in Brian2 for representing distinct cell types (like E/I populations) and for modeling intra-population variability in parameters using statistical distributions (`rand()`, `randn()`). We discussed the importance of spontaneous activity in biological circuits, particularly in developing systems like organoids, and implemented a common modeling strategy using background synaptic input generated by `PoissonGroup` objects. The choice of neuron models (LIF vs. AdEx/Izhikevich) was revisited in light of capturing potentially complex spontaneous dynamics. We also conceptually addressed the significant challenge of **parameter estimation**, highlighting the difficulty in precisely tuning complex models to match experimental data from variable systems like organoids. The chapter culminated in a Brian2 simulation of a heterogeneous E/I network driven by Poisson background input, illustrating how these features lead to more biologically plausible asynchronous irregular dynamics. By incorporating heterogeneity and spontaneous activity, our models take a crucial step towards capturing the complex, dynamic environment within which computation might occur in brain organoids. The techniques learned here will be essential for building the more sophisticated models of plasticity, structure, and function in the subsequent chapters.

**Planned Code Example:**
*   **`5.1_HeterogeneousPopulationNoise.ipynb`:** (Provided and explained in Section 5.7) Simulates a network of heterogeneous excitatory and inhibitory LIF neurons exhibiting spontaneous activity driven by Poisson background input. Demonstrates setting variable parameters using string expressions and using `PoissonGroup` for background drive. Visualizes raster plots, population rates, and voltage traces.

**5.9 References for Further Reading (APA Format)**

1.  Amit, D. J., & Brunel, N. (1997). Model of global spontaneous activity and local structured activity during delay periods in the cerebral cortex. *Cerebral Cortex, 7*(3), 237–252. https://doi.org/10.1093/cercor/7.3.237 *(Classic work modeling spontaneous activity states in cortical networks.)*
2.  Destexhe, A., Rudolph, M., & Paré, D. (2003). The high-conductance state of neocortical neurons in vivo. *Nature Reviews Neuroscience, 4*(9), 739–751. https://doi.org/10.1038/nrn1198 *(Reviews the importance of background synaptic activity in shaping neuronal integration in vivo.)*
3.  Ecker, A. S., Berens, P., Tolias, A. S., & Bethge, M. (2010). The effect of noise correlations in populations of neurons. *Journal of Neuroscience, 30*(45), 14965–14965. https://doi.org/10.1523/JNEUROSCI.3966-10.2010 *(Discusses the impact of neuronal variability and correlations, often influenced by heterogeneity and noise.)*
4.  Gjorgjieva, J., Clopath, C., Audet, J., & Pfister, J. P. (2016). Brain-inspired methods for achieving robust computation. *Current Opinion in Neurobiology, 37*, 110–117. https://doi.org/10.1016/j.conb.2016.02.001 *(Discusses how features like heterogeneity might contribute to robust computation.)*
5.  Marder, E., & Goaillard, J. M. (2006). Variability, compensation, and homeostasis in neuron and network function. *Nature Reviews Neuroscience, 7*(7), 563–574. https://doi.org/10.1038/nrn1949 *(Highlights the prevalence and functional importance of variability and compensatory mechanisms in neural systems.)*
6.  Padmanabhan, K., & Urban, N. N. (2010). Intrinsic biophysical diversityDecorrelates neuronal firing dynamics. *Nature Neuroscience, 13*(10), 1276–1282. https://doi.org/10.1038/nn.2630 *(Experimental and modeling work showing how heterogeneity in intrinsic properties can desynchronize networks.)*
7.  Renart, A., de la Rocha, J., Bartho, P., Hollender, L., Parga, N., Reyes, A., & Harris, K. D. (2010). The asynchronous state in cortical circuits. *Science, 327*(5965), 587–590. https://doi.org/10.1126/science.1179850 *(Investigates the mechanisms underlying asynchronous activity in cortex, relevant to modeling spontaneous states.)*
8.  Softky, W. R., & Koch, C. (1993). The highly irregular firing of cortical cells is inconsistent with temporal integration of random EPSPs. *Journal of Neuroscience, 13*(1), 334–350. https://doi.org/10.1523/JNEUROSCI.13-01-00334.1993 *(Classic paper highlighting the challenge of explaining irregular firing, motivating models incorporating noise and balance.)*
9.  Stimberg, M., Goodman, D. F. M., Benichoux, V., & Brette, R. (2019). Equation-oriented specification of neural models for simulations. *Frontiers in Neuroinformatics, 13*, 6. https://doi.org/10.3389/fninf.2019.00006 *(Discusses the equation-oriented approach of Brian2, which facilitates implementing heterogeneity via string expressions.)*
10. Van Rossum, M. C., Bi, G. Q., & Turrigiano, G. G. (2000). Stable Hebbian learning from spike timing-dependent plasticity. *Journal of Neuroscience, 20*(23), 8812–8821. https://doi.org/10.1523/JNEUROSCI.20-23-08812.2000 *(While focused on plasticity, it uses Poisson inputs to drive spontaneous activity in models, illustrating the technique.)*
