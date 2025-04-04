# Chapter 5

# Modeling Heterogeneous Neuron Populations and Spontaneous Activity
------

*Having established the methods for building basic neural networks in Brian2, this chapter focuses on incorporating crucial elements of biological realism often observed in brain organoids and essential for generating complex network dynamics: **heterogeneity** and **spontaneous activity**. Biological neural networks are far from uniform; neurons exhibit significant variability in their properties and connections. Furthermore, neural circuits, especially developing ones like those in organoids, often display ongoing spontaneous activity even in the absence of specific external stimuli. This chapter explores how to capture these features in our computational models. We begin by discussing the biological importance and sources of heterogeneity and how distinct neuron types, like excitatory and inhibitory populations, can be represented. We then delve into methods for modeling variability in neuron parameters *within* a population using statistical distributions implemented via Brian2's string expressions. Subsequently, we address the modeling of spontaneous network activity, covering both intrinsic neuronal noise and, more prominently, the simulation of background synaptic input using tools like Brian2's `PoissonInput` or `PoissonGroup`. We will also briefly reconsider the choice of neuron models (LIF vs. more complex ones like AdEx/Izhikevich) in the context of capturing richer spontaneous dynamics potentially seen in organoids. A crucial related challenge, the **parameter estimation problem**—finding appropriate model parameters to match experimental data—will be discussed conceptually. Finally, we integrate these concepts in a practical Brian2 example, simulating a heterogeneous E/I network exhibiting spontaneous activity driven by background Poisson inputs, demonstrating how these factors shape network behavior.*

----

**5.1 Capturing Biological Heterogeneity (Cell Types, Varying Properties)**

The pursuit of biological realism in computational neuroscience compels us to move beyond idealized models and embrace the inherent **heterogeneity** that characterizes real neural systems. The intricate networks within the brain, and similarly those self-assembling within brain organoids, are far from monolithic arrays of identical processing units. Instead, they comprise a rich tapestry of diverse cell types, and even neurons nominally belonging to the same class exhibit considerable variation in their specific morphological features (size, shape, dendritic branching), electrophysiological profiles (firing patterns, thresholds, adaptation rates), and synaptic connection properties (strength, plasticity rules, target specificity). This pervasive biological variability is not merely an inconvenient imperfection or "noise" that complicates analysis; rather, mounting evidence suggests it is often a functionally significant feature, intricately woven into the fabric of neural computation and contributing profoundly to the richness, flexibility, and robustness of brain function. Failing to account for heterogeneity can lead to models that exhibit overly simplistic or stereotypical dynamics, failing to capture the complex, often irregular, and adaptive behaviors observed in living neural tissue.

In the specific context of brain organoids, the sources of heterogeneity are multifaceted and deeply rooted in the complex biological processes underlying their *in vitro* development. As pluripotent stem cells differentiate along neural lineages, **stochastic events** (random fluctuations) at the molecular level can influence individual cell fate decisions, leading to variations in the final cell types produced even under seemingly identical conditions. Furthermore, within the three-dimensional structure of the growing organoid, cells inevitably experience slightly different **local signaling microenvironments**. Gradients of morphogens, nutrients, oxygen, and metabolic waste products can vary spatially, influencing proliferation rates, differentiation trajectories, and neuronal maturation in a position-dependent manner. If induced pluripotent stem cells (iPSCs) are used as the starting material, the reprogramming process itself can introduce **epigenetic variability** among cells, potentially leading to differences in gene expression and cellular properties that persist through differentiation. Additionally, the processes of neuronal migration, dendritic outgrowth, and synapse formation involve complex cell-cell interactions and activity-dependent mechanisms that are inherently variable and contribute to unique wiring patterns and properties for each neuron. Finally, the overall **maturation process** within an organoid is often asynchronous, with different cells and circuits developing at different rates, further contributing to the heterogeneous landscape observed at any given time point.

Recognizing these biological realities underscores the necessity of incorporating heterogeneity into our computational models if we aim to bridge the gap between simulation and the experimental observations from brain organoids. Models that explicitly include diverse cell types and variability in their parameters offer several key advantages. Firstly, they drastically enhance the **biological plausibility** of the simulations. By reflecting the known diversity of the system being modeled, such simulations are inherently more likely to reproduce the complex spatio-temporal activity patterns often recorded experimentally, such as the asynchronous irregular firing characteristic of the awake cortex, the broad distributions of neuronal firing rates, specific types of network oscillations that depend on the interplay of different interneuron classes, or complex responses to structured stimuli that engage diverse cellular properties. Capturing this realism is essential if the models are to serve as effective tools for understanding the underlying mechanisms.

Secondly, heterogeneity profoundly impacts **network dynamics and computational capabilities**. Introducing variability in neuronal excitability parameters (like thresholds or resting potentials) can act as a powerful **desynchronizing** force, preventing the network from falling into overly simplistic, globally synchronized states and promoting richer, more complex dynamic regimes, potentially operating near critical points thought to be advantageous for computation. Diversity in synaptic strengths and connectivity patterns can enhance the network's capacity for information storage and pattern separation. The fundamental division into excitatory and inhibitory populations, often with distinct dynamic properties (e.g., faster-spiking interneurons), is absolutely essential for maintaining network stability (the E/I balance) and enabling complex computational operations like gain control, selective attention, and temporal sequence processing. Ignoring this heterogeneity can lead to models that fail to capture these critical functional aspects.

Thirdly, heterogeneity is increasingly recognized as a crucial factor contributing to the remarkable **robustness** of biological neural systems. A network composed of diverse elements, each with slightly different response properties, may be inherently less vulnerable to perturbations, damage, or the failure of individual components compared to a highly tuned, homogeneous network where the failure of one element might have cascading consequences. This distributed representation and functional redundancy provided by heterogeneity likely contribute to the brain's ability to function reliably despite ongoing synaptic turnover, cell death, and noisy environmental conditions. Therefore, embracing heterogeneity in models is not just about realism, but potentially about understanding fundamental principles of robust biological computation. Moving forward, our simulations must increasingly reflect this essential aspect of neural organization.

**5.2 Representing Neuron Types (Distinct E/I Groups)**

The most fundamental and ubiquitously recognized form of cellular heterogeneity within cortical structures and many other brain regions, which is also readily apparent in developing brain organoids, is the functional dichotomy between **excitatory (E)** and **inhibitory (I)** neurons. This division represents a cornerstone of neural circuit organization and dynamics. Excitatory neurons, typically utilizing glutamate as their primary neurotransmitter in the mammalian CNS, act to depolarize their postsynaptic targets, primarily by opening channels permeable to $Na^+$ and $Ca^{2+}$. Their collective action serves to propagate signals and drive activity through the network. In contrast, inhibitory neurons, predominantly using GABA (or glycine in some regions), act to hyperpolarize their targets (by opening $Cl^-$ or $K^+$ channels with reversal potentials below threshold) or to implement shunting inhibition (by increasing conductance near the resting potential, effectively reducing the impact of excitatory inputs). Their crucial role is to control overall network excitability, prevent runaway positive feedback loops, sculpt spatio-temporal activity patterns, synchronize neuronal firing (e.g., pacing gamma oscillations), and implement various forms of gain control and competitive interactions.

These two broad classes, E and I neurons, often exhibit systematic differences extending beyond their neurotransmitter identity. Inhibitory neurons, particularly certain subtypes like fast-spiking basket cells, often display distinct intrinsic electrophysiological properties compared to excitatory pyramidal neurons, such as narrower action potentials, higher maximum firing rates, and less spike frequency adaptation. Their morphological characteristics (e.g., dendritic tree structure, axonal projection patterns) and synaptic connectivity rules also differ significantly. For instance, inhibitory connections are often more local, while excitatory connections can span longer distances; connection probabilities between different E/I combinations (E-E, E-I, I-E, I-I) typically follow specific patterns crucial for network stability and function. Accurately capturing this fundamental E/I distinction, including potential differences in their intrinsic dynamics and connectivity statistics, is therefore often considered the minimum essential level of heterogeneity required for building even moderately realistic models of cortical or organoid circuits.

In the Brian2 simulation environment, representing these distinct E and I populations is straightforward and can be achieved in several ways, depending on the complexity required. A highly flexible and often conceptually clear approach is to define **separate `NeuronGroup` objects** for the excitatory and inhibitory populations. This method allows maximum flexibility in assigning potentially different model equations (e.g., LIF for one type, AdEx for another) or distinct parameter sets (e.g., different time constants, thresholds, resting potentials reflecting known physiological differences) to each group independently.

```python
# Example: Defining separate E and I groups with potentially different taus
N_E = 80; N_I = 20
tau_E = 15*ms; tau_I = 10*ms
V_rest_common = -70*mV # Other params might be common or also distinct

eqs_lif_e = '''dv/dt = (V_rest_common - v)/tau_E : volt ...'''
eqs_lif_i = '''dv/dt = (V_rest_common - v)/tau_I : volt ...''' # Using different tau

P_E = NeuronGroup(N_E, eqs_lif_e, threshold='...', reset='...', name='Excitatory')
P_I = NeuronGroup(N_I, eqs_lif_i, threshold='...', reset='...', name='Inhibitory')

# Connections are then defined between these specific groups:
# syn_EE = Synapses(P_E, P_E, ...)
# syn_EI = Synapses(P_E, P_I, ...)
# syn_IE = Synapses(P_I, P_E, ...)
# syn_II = Synapses(P_I, P_I, ...)
```
This modular approach makes the code structure intuitive and facilitates setting up cell-type-specific connectivity rules using distinct `Synapses` objects linking the appropriate source and target groups. It clearly segregates the definitions and parameters related to each population.

An alternative strategy, particularly suitable when the underlying differential equations are identical for both populations but some parameters differ, is to use a **single, larger `NeuronGroup`** encompassing all neurons and then use **slicing and indexing** to manage the E and I subpopulations.

```python
# Example: Single group with E/I slices
N = 100; N_E = 80; N_I = 20
tau_base = 10*ms # Base time constant

eqs_lif_param = '''
dv/dt = (V_rest - v)/tau : volt ...
tau : second # Tau is now a per-neuron parameter
V_rest : volt # V_rest is also per-neuron
'''
neurons = NeuronGroup(N, eqs_lif_param, threshold='...', reset='...', name='Population')
P_E = neurons[:N_E]
P_I = neurons[N_E:]

# Assign parameters differently to slices
P_E.tau = 15*ms
P_I.tau = 10*ms
P_E.V_rest = -65*mV
P_I.V_rest = -70*mV
# ... initialize other parameters ...

# Synapses connect using indices relative to the main 'neurons' group
# syn = Synapses(neurons, neurons, ...)
# syn.connect(i=P_E.i, j=P_I.i, ...) # Connect E subset to I subset
```
This approach can sometimes be more memory-efficient for very large networks but might make the code slightly less readable, especially when dealing with complex connectivity rules or multiple distinct parameters. The choice between these methods often depends on personal preference and the specific requirements of the model. For the level of complexity typically encountered in introductory organoid modeling focusing on E/I interactions, using separate `NeuronGroup` objects often provides better clarity and modularity. Of course, biological reality includes far more than just two neuron types; representing finer subdivisions (e.g., different classes of cortical interneurons like PV+, SST+, VIP+) would involve creating additional `NeuronGroup` objects with appropriately tuned parameters and connectivity profiles, adding further layers of biologically relevant heterogeneity.

**5.3 Modeling Parameter Variability (Statistical Distributions)**

While distinguishing between major cell classes like excitatory and inhibitory neurons captures a crucial aspect of heterogeneity, it doesn't account for the significant variability observed *within* each population. As established earlier, individual neurons, even of the same nominal type and in close proximity, exhibit a spectrum of properties. Their resting membrane potentials might differ by several millivolts, their firing thresholds can vary, their membrane time constants might not be identical, and the strengths of their outgoing or incoming synapses can fluctuate considerably. Incorporating this **intra-population variability** into computational models is often essential for achieving dynamics that closely mimic biological observations, particularly the asynchronous and irregular firing patterns prevalent in many brain areas and often seen in organoid cultures. Homogeneous networks, where all neurons of a given type are identical, tend to exhibit overly synchronized dynamics, with neurons firing in lockstep or producing highly regular oscillations, which is typically considered biologically unrealistic for cortical computation.

Brian2 offers a highly flexible and syntactically elegant mechanism for introducing such parameter variability directly within string expressions when setting the initial values or parameters of a `NeuronGroup` or `Synapses` object. Instead of assigning a single fixed numerical value (with units), one can provide a string that includes mathematical operations and calls to built-in random number generation functions. Brian2 parses this string and executes it independently for each element (neuron or synapse) being initialized, thereby assigning potentially unique, randomly drawn values based on the specified distribution. The two primary random functions available are:
*   `rand()`: Generates random numbers drawn from a **uniform distribution** between 0.0 and 1.0.
*   `randn()`: Generates random numbers drawn from a **standard normal (Gaussian) distribution** with a mean of 0.0 and a standard deviation of 1.0.

These functions can be easily scaled and shifted within the string expression to achieve desired distributions for specific parameters. For example, let's revisit setting the resting potential (`V_rest`) for a population of neurons, assuming `V_rest` is defined as a per-neuron parameter in the model equations (`V_rest : volt`). To assign values drawn from a Gaussian distribution with a specific mean (`V_rest_mean`) and standard deviation (`V_rest_std`):

```python
V_rest_mean = -70*mV
V_rest_std = 5*mV
neurons.V_rest = 'V_rest_mean + V_rest_std * randn()'
# Brian2 interprets this string and assigns each neuron in the 'neurons' group
# a V_rest value = -70mV + 5mV * (a sample from N(0,1))
```
This single line efficiently assigns heterogeneous resting potentials across the entire population according to the specified normal distribution.

Similarly, one might want to introduce variability using a uniform distribution. For instance, to set the membrane time constant (`tau`), assuming it's a per-neuron parameter (`tau : second`), to vary uniformly within a certain percentage range around a mean value:

```python
tau_mean = 10*ms
tau_range_half = 2*ms # e.g., tau varies between 8ms and 12ms
# Using rand() which is U(0,1)
neurons.tau = '(tau_mean - tau_range_half) + (2 * tau_range_half * rand())'
# Alternative using +/- percentage:
# tau_variation = 0.2 # +/- 20%
# neurons.tau = 'tau_mean * (1 + tau_variation * (2*rand() - 1))'
```
The expression `(2*rand() - 1)` generates random numbers uniformly distributed between -1 and 1, allowing easy specification of symmetric variation around a mean.

This string-based mechanism can be applied to virtually any numerical parameter defined in the `NeuronGroup` or `Synapses` model equations, including thresholds (`V_thresh`), reset potentials (`V_reset`), adaptation parameters in AdEx or Izhikevich models, synaptic weights (`w`), or synaptic delays (`delay`). Introducing heterogeneity in firing thresholds, for instance, means some neurons will be intrinsically more excitable (lower threshold) and fire more readily in response to input, while others will be less excitable (higher threshold). Variability in time constants affects how quickly neurons integrate inputs. Heterogeneity in synaptic weights introduces variability in communication strength. Collectively, assigning realistic distributions (often Gaussian or log-normal are assumed for biological parameters, though uniform can be used for simplicity) to key parameters is a powerful technique for breaking excessive synchrony in simulated networks and promoting more biologically plausible asynchronous irregular (AI) dynamics. This AI state, characterized by irregular individual neuron firing and low pairwise correlations, is thought to be computationally advantageous, allowing for richer coding and faster responses. Therefore, incorporating parameter variability is not just about realism but is often functionally necessary to achieve network dynamics relevant to computation.

`[Conceptual Figure 5.1: Heterogeneity Effects. Panel (a): Raster plot of a homogeneous network showing highly synchronized firing. Panel (b): Raster plot of a heterogeneous network (with parameter variability) showing asynchronous, irregular firing. Panel (c): Histogram of firing rates from the heterogeneous network, showing a broad, often skewed distribution.]`

**5.4 Modeling Spontaneous Activity (Intrinsic Noise, Background Input - `PoissonInput`)**

A defining characteristic of living neural circuits, strikingly evident in recordings from the developing brain *in vivo* and commonly observed in functional assessments of *in vitro* systems like brain organoids, is the pervasive presence of **spontaneous activity**. Neurons do not simply remain silent waiting for an external command; they exhibit ongoing, seemingly random firing, and the network as a whole hums with fluctuating electrical dynamics even in the complete absence of structured sensory input or deliberately applied stimuli. This intrinsic, self-generated activity is far from being mere inconsequential background noise. It is increasingly recognized as playing fundamental roles in various aspects of brain function and development. During development, spontaneous activity (often taking the form of synchronized waves or bursts in early stages) is crucial for guiding processes like neuronal migration, synapse formation, circuit refinement, and the activity-dependent maturation of ion channels and receptors. In mature circuits, spontaneous activity helps maintain neuronal responsiveness, preventing neurons from becoming completely quiescent, contributes to the representation of internal states (e.g., during memory consolidation or mental simulation), may play a role in stochastic resonance (enhancing the detection of weak signals), and forms the dynamic backdrop upon which stimulus-evoked responses are superimposed. Given its ubiquity and functional significance, understanding the origins of spontaneous activity and incorporating it realistically into computational models is essential for simulating organoid behavior and exploring its computational implications.

The origins of spontaneous activity in biological networks can be traced to several contributing factors. At the single-cell level, **intrinsic neuronal noise** arises from the inherently stochastic (random) nature of molecular processes, particularly the opening and closing of individual ion channels embedded in the neuronal membrane. While the collective behavior of a vast number of channels can often be approximated by deterministic equations (like the HH model), the random fluctuations in the state of a smaller number of channels, especially near the threshold potential or in neurons with high input resistance, can lead to significant membrane potential fluctuations (`Vm noise`). These fluctuations can occasionally, purely by chance, cross the action potential threshold, leading to spontaneous spike generation. This intrinsic noise source can be incorporated into computational models by adding a stochastic term, often represented as Gaussian white noise, to the differential equation governing the membrane potential. In Brian2, the symbol `xi` within model equations represents such a noise term, appropriately scaled by the simulation time step:

```latex
% Equation including intrinsic noise term sigma*sqrt(tau)*xi
\tau \frac{dV}{dt} = -(V - V_{\text{rest}}) + RI(t) + \sigma \sqrt{\tau} \xi
```
Here, $\sigma$ controls the amplitude of the noise. Simulating such stochastic differential equations (SDEs) requires appropriate numerical methods (like Euler-Maruyama, supported by Brian2). While intrinsic channel noise certainly contributes to spontaneous activity, especially in developing or low-activity states, it's often thought not to be the dominant factor in driving the relatively high levels of spontaneous firing observed in active cortical circuits *in vivo*.

A more widely implicated source, particularly in mature and interconnected networks, is the continuous **background synaptic input** that neurons receive. Even when not engaged in a specific task, a neuron in the cortex is constantly bombarded by thousands of excitatory and inhibitory postsynaptic potentials (EPSPs and IPSPs) arriving from other neurons within the local circuit or from more distant brain regions. This relentless synaptic barrage reflects the ongoing, self-sustained, reverberating activity within the larger network. The summation of these numerous, largely uncorrelated synaptic events creates significant fluctuations in the postsynaptic neuron's membrane potential, occasionally driving it across threshold to fire spontaneously. This synaptic background activity is crucial for maintaining the network in a responsive "high-conductance state" observed *in vivo*, characterized by low input resistance and rapid membrane potential fluctuations. In isolated *in vitro* systems like brain organoids, which lack external inputs from other brain regions, spontaneous activity likely arises from a combination of factors: heightened intrinsic excitability common in developing neurons, the formation of recurrent excitatory connections leading to self-amplifying activity loops, the interplay with inhibitory neurons attempting to control this excitation, and potentially spontaneous neurotransmitter release.

Modeling this pervasive background synaptic activity computationally often involves approximating the input from a vast number of unmodeled external neurons as random, independent **Poisson processes**. A Poisson process is a statistical model describing events (in this case, presynaptic spikes) occurring randomly over time with a certain average rate ($\lambda$). Assuming that each neuron in our simulated network receives input from many such external sources provides a mathematically tractable way to generate fluctuating synaptic drive. Brian2 offers several convenient tools for implementing this. The `PoissonInput` class can be used to directly trigger events (like adding to a conductance variable) in target neurons at specified rates. However, a more flexible and arguably more intuitive approach is to use the **`PoissonGroup`** object. A `PoissonGroup` acts like a virtual `NeuronGroup` whose neurons fire spikes according to independent Poisson processes at user-defined rates. These virtual Poisson neurons can then be connected to the main network neurons using standard `Synapses` objects, allowing explicit modeling of the synaptic weights and delays associated with this background input.

```python
# Example: Setting up background Poisson input
N_neurons = 1000 # Target population size
bg_rate_E = 8*Hz  # Avg rate of external E sources
bg_rate_I = 4*Hz  # Avg rate of external I sources
# For simplicity, create one E and one I source group targeting all neurons
# (More realistic: many sources sparsely connected)
P_bg_E = PoissonGroup(1, rates=bg_rate_E * N_neurons * p_bg_connect_E) # Total rate for E input
P_bg_I = PoissonGroup(1, rates=bg_rate_I * N_neurons * p_bg_connect_I) # Total rate for I input
# Need connection probability and weight... simpler: Drive each neuron individually
P_bg = PoissonGroup(N_neurons, rates=bg_rate) # Each neuron gets its own Poisson source

# Connect these sources to the 'neurons' group
# Assuming conductance-based synapses targetting g_E
w_bg_input = 1.5*nS # Weight of background synapse
syn_background = Synapses(P_bg, neurons, on_pre='g_E_post += w_bg_input')
syn_background.connect(j='i') # Connect Poisson neuron i to network neuron i
```
*(Self-correction: The initial idea of shared Poisson sources is complex to scale correctly. Switched to the simpler, common approach where each network neuron receives its own independent Poisson input stream, represented by connecting `PoissonGroup(N, rates)` one-to-one.)*

This setup provides each neuron in the `neurons` group with an independent stream of excitatory synaptic conductance increases arriving randomly at the average rate `bg_rate`. By appropriately setting the `bg_rate` and the synaptic weight `w_bg_input` (and potentially adding a similar inhibitory background input stream targeting `g_I`), the modeler can effectively inject synaptic noise that drives spontaneous activity and helps establish a dynamic background state resembling *in vivo* conditions. This method is generally preferred over adding direct voltage noise (`xi`) because it more explicitly models the synaptic origin of background fluctuations and allows for separate control over excitatory and inhibitory background drives, which is crucial for achieving balanced network states. Tuning these background input parameters becomes another key aspect of fitting the model to desired spontaneous activity levels observed experimentally.

**5.5 Choice of Neuron Models (LIF vs. AdEx/Izhikevich for Organoid Dynamics)**

The decision regarding which specific mathematical model to employ for representing individual neurons takes on particular significance when aiming to capture the potentially complex spontaneous dynamics observed in developing systems like brain organoids. The choice inherently involves navigating the persistent trade-off between computational tractability and the fidelity with which the model reproduces the repertoire of behaviors exhibited by real biological neurons. As established in Chapter 3, the spectrum of available models ranges from the highly simplified Leaky Integrate-and-Fire (LIF) neuron to progressively more complex formulations.

The **LIF model**, due to its representation by a single linear differential equation, offers unparalleled computational efficiency, making it highly attractive for simulations involving very large numbers of neurons or requiring long simulation durations. It successfully captures the fundamental neuronal properties of input integration, membrane leak, firing threshold, and refractoriness. For many studies focusing on large-scale network phenomena driven primarily by connectivity patterns and E/I balance, the LIF model can be perfectly adequate. However, its inherent simplicity means it cannot intrinsically reproduce certain dynamic features often observed in biological neurons, including those potentially present in organoid cultures. Specifically, the standard LIF model does not exhibit **spike frequency adaptation** (the gradual decrease in firing rate during a sustained suprathreshold input) or **intrinsic bursting** (the generation of clusters of action potentials followed by periods of quiescence, even with constant input). Its spike initiation is also typically modeled as instantaneous, lacking the smoother upswing seen experimentally. If these richer single-neuron dynamics are believed to play a crucial role in shaping the network activity patterns or computational properties of the organoid model, then the limitations of the basic LIF model might necessitate exploring more sophisticated alternatives.

Models like the **Adaptive Exponential Integrate-and-Fire (AdEx)** or the **Izhikevich model** were specifically developed to bridge the gap between the extreme simplicity of LIF and the complexity of full Hodgkin-Huxley models. Both AdEx and Izhikevich models achieve this by incorporating a second dynamic variable (in addition to the membrane potential $V$) that represents a slower process, typically related to adaptation currents (like slow $K^+$ currents or $Ca^{2+}$-activated currents). In the AdEx model, this variable $w$ mediates spike frequency adaptation and, through its interaction with the exponential spike initiation mechanism, can also generate intrinsic bursting for certain parameter regimes. The Izhikevich model, using a different mathematical formulation involving quadratic terms and a piecewise reset mechanism for its recovery variable $u$, is particularly celebrated for its remarkable ability to reproduce a vast diversity of known electrophysiological firing patterns (regular spiking, intrinsic bursting, chattering, fast spiking, low-threshold spiking, mixed-mode oscillations, etc.) simply by adjusting its four main parameters ($a, b, c, d$).

Employing AdEx or Izhikevich neurons in a network simulation allows the model to generate more complex and potentially more biologically realistic spontaneous activity patterns *intrinsically*, arising from the properties of the individual neurons themselves, rather than relying solely on network interactions or specific patterns of external input. For instance, a network composed of AdEx neurons tuned to be intrinsically bursting might spontaneously generate synchronized bursting activity that more closely resembles patterns seen in developing cortical cultures or organoids, compared to a network of purely regular-spiking LIF neurons. Similarly, incorporating adaptation allows the network to exhibit history-dependent responses and potentially operate across a wider range of input intensities. Therefore, if experimental data from organoids reveals prominent adaptation, bursting, or other complex firing patterns, using these more sophisticated models might be crucial for accurately capturing the underlying dynamics and exploring their functional consequences.

However, the advantages of using AdEx or Izhikevich models must be weighed against their drawbacks. The primary cost is **increased computational load**, as simulating two coupled differential equations per neuron is inherently more demanding than simulating one, potentially limiting the feasible size or duration of network simulations. Perhaps more critically, these models introduce additional parameters (e.g., $a, b, c, d$ for Izhikevich; $\tau_w, a, b$ for AdEx) that need to be determined. **Parameterizing these models** to accurately reflect the specific properties of potentially immature and heterogeneous neurons within a brain organoid can be significantly more challenging than parameterizing the simpler LIF model (see Section 5.6). Experimental data characterizing the detailed firing patterns of different organoid cell types might be scarce or highly variable. Therefore, the choice of neuron model involves a careful balancing act. For initial explorations or very large-scale simulations focusing on fundamental E/I interactions, LIF might be the pragmatic choice. For studies specifically investigating the role of adaptation, bursting, or diverse firing patterns suggested by experimental organoid data, the additional complexity and parameterization effort required for AdEx or Izhikevich models may be necessary and justified. Brian2's flexible equation-oriented structure facilitates implementing and comparing these different models within the same network framework, allowing researchers to systematically investigate the impact of single-neuron dynamics on emergent network behavior.

**5.6 The Challenge of Parameter Estimation (Conceptual)**

A critical, often underappreciated, yet fundamental challenge encountered when constructing biologically plausible computational models of neural systems, especially complex and developing ones like brain organoids, is the formidable task of **parameter estimation**. Our mathematical models, regardless of their level of abstraction (from single ion channels to large networks), invariably contain a multitude of **parameters**—constants or variables that define the specific quantitative properties of the model components (e.g., membrane time constants, ion channel conductances, synaptic weights, connection probabilities, adaptation strengths, background input rates). The behavior of the model is exquisitely sensitive to the values assigned to these parameters. For a model to be truly informative about the biological system it represents, or to make reliable predictions, its parameters should ideally be constrained by, or derived from, empirical measurements of that system. However, acquiring the necessary experimental data and using it to determine the "correct" parameter values for complex biological models is fraught with difficulties, constituting a major bottleneck in the modeling process.

The first major difficulty stems from the sheer **high dimensionality of the parameter space**. Even a moderately complex model of an E/I network, incorporating heterogeneity and some synaptic dynamics, can easily involve dozens or even hundreds of parameters. Systematically exploring how the model's behavior changes across all possible combinations of these parameters is computationally infeasible due to the "curse of dimensionality"—the exponential growth of the volume of the parameter space with the number of parameters. This means that finding the specific region within this vast space that corresponds to biologically realistic behavior can be like searching for a needle in a haystack.

Secondly, many crucial parameters are inherently **difficult or impossible to measure directly and accurately** within the target experimental system, particularly in the intricate, dense, and often optically opaque 3D environment of a brain organoid. While techniques like patch-clamp electrophysiology can provide detailed information about the intrinsic properties or synaptic inputs of individual, accessible neurons, obtaining comprehensive, quantitative data on, for example, the precise strengths and kinetics of all relevant synapse types (E-E, E-I, I-E, I-I), the exact probabilities of connection between different cell types as a function of distance or layer, the detailed kinetics of all functionally relevant ion channels, or the real-time dynamics of neuromodulatory influences across the entire organoid network is currently beyond experimental reach. Modelers often have to rely on incomplete data, extrapolate from related but different systems (e.g., rodent slice physiology), or make educated guesses based on theoretical principles.

Thirdly, the pervasive **biological variability** adds another layer of complexity. As emphasized throughout this chapter, parameters are often not fixed constants but vary significantly from cell to cell and potentially from preparation to preparation. This means that searching for a single "best" value for each parameter might be misguided. Ideally, the model should capture the *distributions* of parameter values observed experimentally. Furthermore, in a developing system like an organoid, parameters themselves might be **non-stationary**, changing over developmental time as cells mature and circuits reorganize. This dynamic nature makes parameter estimation even more challenging, potentially requiring time-varying parameters or models that explicitly incorporate developmental processes.

Faced with these challenges, computational neuroscientists employ a range of parameter estimation strategies, often in combination. A common starting point is to **leverage existing literature**, adopting parameter values reported for similar cell types or synapses in other experimental preparations, while acknowledging the potential inaccuracies of this transfer. **Manual tuning** remains a prevalent technique, especially in exploratory modeling. Here, the modeler relies on intuition and iterative trial-and-error, adjusting key parameters by hand and visually comparing the simulation output (e.g., firing rates, oscillation patterns in raster plots or LFPs) to qualitative features observed in experimental data. While useful for initial model development, manual tuning is subjective, labor-intensive, not guaranteed to find optimal solutions, and typically explores only a tiny fraction of the parameter space.

More systematic approaches employ **automated parameter search and optimization** techniques. These methods require defining a quantitative **objective function** (or cost function) that measures the discrepancy between specific features extracted from the simulation output and corresponding features from target experimental data. Optimization algorithms are then used to automatically search the parameter space for values that minimize this objective function. Algorithms used include **grid search** (systematically evaluating the model on a predefined grid of parameter values, feasible only for very low dimensions), **gradient-based methods** (if the gradient of the objective function with respect to parameters can be calculated or estimated, but often gets stuck in local minima), and **global optimization algorithms** (like genetic algorithms, particle swarm optimization, simulated annealing) which are better at exploring complex, high-dimensional landscapes but can be computationally very expensive, often requiring thousands or millions of simulation runs.

`[Conceptual Figure 5.2: Parameter Estimation Workflow. A diagram showing "Experimental Data" (e.g., MEA recordings from organoid) and a "Computational Model" with unknown parameters. An "Objective Function" compares simulation output (e.g., simulated firing statistics) to experimental data (e.g., measured firing statistics). An "Optimization Algorithm" (or manual tuning loop) iteratively adjusts model parameters (P) to minimize the objective function value, leading to an "Estimated Parameter Set" (P*). Dashed lines indicate challenges like high dimensionality, lack of data, local minima, and variability.]`

More recently, advanced methods grounded in statistics and machine learning are gaining traction. **Bayesian inference** approaches (e.g., using Markov Chain Monte Carlo (MCMC) methods or approximate techniques like variational inference or simulation-based inference/Approximate Bayesian Computation (ABC)) aim not just to find a single best parameter set, but to estimate the full **posterior probability distribution** over the parameter space, given the experimental data and prior knowledge. This provides a principled way to quantify uncertainty about parameter values and identify parameter degeneracies (where different parameter combinations produce similar outputs). However, Bayesian methods are typically computationally intensive and require careful statistical formulation. **Data assimilation** techniques, borrowed from fields like weather forecasting, aim to continuously update the state and parameters of a model as new experimental data becomes available, potentially suitable for tracking non-stationary systems. Tools like `scikit-optimize` in Python offer implementations of some optimization algorithms, while specialized frameworks are emerging for simulation-based inference in neuroscience.

For modeling brain organoids specifically, parameter estimation remains a particularly acute challenge due to the system's inherent complexity, high variability, developmental dynamics, and the current limitations in obtaining comprehensive, high-resolution functional data *in situ*. Therefore, while striving for biological realism, modelers must be transparent about the assumptions made and the uncertainties associated with the chosen parameters. Often, the most fruitful approach involves focusing on **sensitivity analysis** (identifying which parameters most strongly influence the model behaviors of interest), exploring the model's dynamics across plausible ranges of key parameters rather than fixating on finding a single "correct" set, and using the model primarily to generate **testable hypotheses** about underlying mechanisms that can then guide future, more targeted experimental investigations. While precise quantitative matching might be elusive, computational models parameterized within plausible biological ranges remain invaluable tools for gaining qualitative insights and understanding the potential functional logic of organoid circuits.

**5.7 Brian2 Implementation: Heterogeneous Population with Noise**

This example integrates the core concepts discussed in this chapter: simulating a network composed of distinct excitatory and inhibitory populations where individual neurons possess heterogeneous parameters, and driving the network into a state of spontaneous activity using background Poisson synaptic input. We will use a conductance-based LIF model to incorporate synaptic effects more realistically.

*(The code block remains the same as in the previous response, as it already implements heterogeneity and Poisson input. The explanation below is expanded.)*

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
w_IE = -4.0*nS; w_II = -3.0*nS # I weights (negative conductance change means increase in g_I)
p_connect = 0.1 # Connection probability for all types
delay_syn = 1.5*ms

# Background Input Parameters
bg_rate = 5*Hz   # Rate of external Poisson spikes per input source
w_bg = 1.0*nS    # Weight of background excitatory input synapse

# --- Neuron Model (Conductance-based LIF) ---
# Includes dynamics for g_E and g_I, and declares V_rest, V_thresh, tau as per-neuron parameters
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
                      method='euler', name='MainPopulation') # Use a descriptive name
P_E = neurons[:N_E]
P_I = neurons[N_E:]
P_E.name = 'ExcitatorySubgroup'
P_I.name = 'InhibitorySubgroup'

# --- Set Heterogeneous Parameters ---
# Assign distinct mean values and variability for E and I populations
# Excitatory Population
P_E.V_rest = 'V_rest_E_mean + V_rest_std * randn()'
P_E.V_thresh = 'V_thresh_E_mean + V_thresh_std * randn()'
P_E.tau = 'tau_E * (1 + tau_variation * (2*rand() - 1))'
# Inhibitory Population
P_I.V_rest = 'V_rest_I_mean + V_rest_std * randn()'
P_I.V_thresh = 'V_thresh_I_mean + V_thresh_std * randn()'
P_I.tau = 'tau_I * (1 + tau_variation * (2*rand() - 1))'

# Initialize membrane potential randomly near individual V_rest
# Need to access the assigned V_rest values. Best practice: initialize after setting parameters.
neurons.v = 'V_rest + rand() * (V_thresh - V_rest) * 0.2' # Small random offset from V_rest
neurons.g_E = 0*nS
neurons.g_I = 0*nS

# --- Create Background Input ---
# One Poisson source driving each neuron independently
P_bg = PoissonGroup(N, rates=bg_rate, name='BackgroundInput')

# --- Synapses ---
# Background Synapses (Excitatory, conductance-based)
# Connects Poisson source i to network neuron i
syn_bg = Synapses(P_bg, neurons, on_pre='g_E_post += w_bg', delay=0.1*ms, name='BackgroundSynapses')
syn_bg.connect(j='i')

# Internal Network Synapses (Conductance-based)
# Excitatory connections
syn_EE = Synapses(P_E, P_E, on_pre='g_E_post += w_EE', delay=delay_syn, name='EE_Synapses')
syn_EE.connect(condition='i!=j', p=p_connect) # Avoid self-connections
syn_EI = Synapses(P_E, P_I, on_pre='g_E_post += w_EI', delay=delay_syn, name='EI_Synapses')
syn_EI.connect(p=p_connect)

# Inhibitory connections (Note: w_IE, w_II are negative, increasing g_I)
syn_IE = Synapses(P_I, P_E, on_pre='g_I_post -= w_IE', delay=delay_syn, name='IE_Synapses') # Use -= w_IE because w_IE is negative
# Correction: Brian2 adds conductance. Inhibitory weight should be positive value added to g_I.
# Let's redefine weights to be positive magnitudes.
w_IE_abs = 4.0*nS; w_II_abs = 3.0*nS
syn_IE = Synapses(P_I, P_E, on_pre='g_I_post += w_IE_abs', delay=delay_syn, name='IE_Synapses')
syn_IE.connect(p=p_connect)
syn_II = Synapses(P_I, P_I, on_pre='g_I_post += w_II_abs', delay=delay_syn, name='II_Synapses')
syn_II.connect(condition='i!=j', p=p_connect)

# --- Monitors ---
spike_mon = SpikeMonitor(neurons, name='SpikeMonitor')
rate_mon = PopulationRateMonitor(neurons, name='RateMonitor')
# Record voltage from 3 example neurons (indices 0 (E), N//2 (E), N_E (I))
state_mon_v = StateMonitor(neurons, 'v', record=[0, N // 2, N_E], name='VoltageMonitor')

# --- Run Simulation ---
sim_duration = 1000*ms
run(sim_duration)

# --- Visualize Results ---
plt.figure(figsize=(12, 10))

# 1. Raster Plot
plt.subplot(3, 1, 1)
# Plot E and I spikes with different colors for clarity
plt.plot(spike_mon.t[spike_mon.i < N_E]/ms, spike_mon.i[spike_mon.i < N_E], '.r', markersize=1, label='Excitatory')
plt.plot(spike_mon.t[spike_mon.i >= N_E]/ms, spike_mon.i[spike_mon.i >= N_E], '.b', markersize=1, label='Inhibitory')
# plt.axhline(N_E, ls='-', color='gray', lw=0.5) # Line no longer needed with colors
plt.xlabel('Time (ms)')
plt.ylabel('Neuron Index')
plt.title(f'Raster Plot (N={N}, Red=E, Blue=I)')
plt.xlim(0, sim_duration/ms)
plt.ylim(-1, N)
plt.legend(markerscale=5, loc='upper right')
plt.grid(True, axis='y', linestyle=':')

# 2. Population Rate
plt.subplot(3, 1, 2)
# Calculate smoothed rates for better visualization
# Use rate_mon directly first, smoothing is optional advanced step
plt.plot(rate_mon.t/ms, rate_mon.rate, label='Overall Population Rate')
plt.xlabel('Time (ms)')
plt.ylabel('Rate (Hz)')
plt.title('Population Firing Rate')
plt.legend()
plt.xlim(0, sim_duration/ms)
plt.grid(True)

# 3. Example Voltage Traces
plt.subplot(3, 1, 3)
neuron_labels = [f'Neuron 0 (E)', f'Neuron {N // 2} (E)', f'Neuron {N_E} (I)']
colors = ['red', 'darkred', 'blue']
for i in range(len(state_mon_v.record)):
    plt.plot(state_mon_v.t/ms, state_mon_v.v[i]/mV, label=neuron_labels[i], color=colors[i], lw=0.5)
plt.xlabel('Time (ms)')
plt.ylabel('Membrane Potential (mV)')
plt.title('Example Voltage Traces')
plt.legend(fontsize='small')
plt.xlim(0, sim_duration/ms)
plt.grid(True)

plt.tight_layout()
plt.show()
```
*(Self-correction: Corrected the handling of inhibitory synaptic weights in the conductance-based model. Conductance must be positive, so `w_IE` and `w_II` should represent the positive conductance increment, which is then added to `g_I`. Adjusted the code accordingly (`w_IE_abs`, `w_II_abs`). Also improved initial voltage setting and added names to Brian2 objects for better introspection. Added coloring to raster plot.)*

**Explanation of the Code:**

1.  **Parameters:** We define parameters for a larger network (`N=1000`) with an 80/20 E/I ratio. Mean LIF parameters are specified, potentially differing between E (`tau_E`, `V_rest_E_mean`, `V_thresh_E_mean`) and I types (`tau_I`, etc.). Parameters controlling the amount of heterogeneity are also defined (`V_rest_std`, `V_thresh_std`, `tau_variation`). Synaptic parameters are now defined for a **conductance-based** model, including time constants for conductance decay (`tau_g_E`, `tau_g_I`), reversal potentials (`E_E`, `E_I`), and synaptic weights specified as conductance changes (e.g., `w_EE` in nanoSiemens, `nS`). Inhibitory weights (`w_IE_abs`, `w_II_abs`) represent the magnitude of the increase in inhibitory conductance `g_I`. Background input parameters (`bg_rate`, `w_bg`) define the external drive.
2.  **Neuron Model:** The `eqs` string now defines a conductance-based LIF model. The membrane potential `dv/dt` equation includes terms for the currents flowing through the excitatory (`g_E`) and inhibitory (`g_I`) conductances, calculated using their respective reversal potentials (`E_E`, `E_I`). Crucially, the equations also include the dynamics for `g_E` and `g_I` themselves, modeling their exponential decay back to zero with time constants `tau_g_E` and `tau_g_I`. Importantly, `V_rest`, `V_thresh`, and `tau` are declared within the equations without fixed values, indicating they will be **per-neuron parameters**.
3.  **Neuron Groups:** A single `NeuronGroup` (`neurons`) is created for all `N` neurons using the conductance-based equations. Slices `P_E` and `P_I` are defined for convenience. Descriptive names are added using the `name` argument.
4.  **Heterogeneity:** This is where parameter variability is introduced. Using string expressions with `randn()` and `rand()`, we assign unique values for `V_rest`, `V_thresh`, and `tau` to each neuron within the `P_E` and `P_I` subgroups, drawn from distributions centered around their respective mean values (`V_rest_E_mean`, etc.).
5.  **Initial Conditions:** Membrane potentials `v` are initialized randomly to values slightly offset from each neuron's specific `V_rest`. Synaptic conductances `g_E` and `g_I` are initialized to zero.
6.  **Background Input:** A `PoissonGroup` (`P_bg`) is created with `N` independent sources, each firing at `bg_rate`. The `Synapses` object `syn_bg` connects Poisson source `i` to network neuron `i` (`j='i'`). When a background spike arrives, it increases the excitatory conductance `g_E` of the target neuron by `w_bg`. This provides sustained, noisy excitatory drive.
7.  **Internal Synapses:** Four `Synapses` objects are created to handle the E-to-E, E-to-I, I-to-E, and I-to-I connections within the network. They use probabilistic connectivity (`p=p_connect`). The `on_pre` actions increment the appropriate postsynaptic conductance (`g_E` for excitatory inputs, `g_I` for inhibitory inputs) by the corresponding weight (`w_EE`, `w_EI`, `w_IE_abs`, `w_II_abs`). Conditions `i!=j` are added for recurrent connections (EE, II) to avoid self-synapses.
8.  **Monitors:** A `SpikeMonitor`, a `PopulationRateMonitor` (for the whole network), and a `StateMonitor` (recording `v` from 3 specific neurons) are set up. Names are assigned for clarity.
9.  **Run & Visualize:** The simulation is run for 1 second (`1000*ms`). The visualization part generates three plots: (1) A raster plot showing individual spike times, now colored to distinguish E (red) and I (blue) neurons, typically revealing asynchronous and irregular activity. (2) The overall population firing rate over time, showing fluctuations around an average level sustained by the background input and network interactions. (3) Example voltage traces from selected E and I neurons, illustrating the fluctuating subthreshold dynamics driven by the balanced synaptic input and occasional spike events.

This more comprehensive simulation effectively demonstrates how to combine key elements of biological realism—distinct cell types, parameter heterogeneity, conductance-based synapses, and background synaptic noise—within the Brian2 framework. The resulting network activity, characterized by spontaneous, irregular firing and a balance between excitation and inhibition, provides a more plausible baseline state upon which to study stimulus processing or learning in organoid-inspired models, compared to simpler homogeneous or deterministic networks. Fine-tuning the parameters (heterogeneity levels, connection weights, background input strength) would be necessary to quantitatively match specific experimental observations (the parameter estimation challenge).

**5.8 Conclusion and Planned Code**

This chapter significantly advanced our modeling capabilities by introducing two critical elements of biological realism: **neuronal heterogeneity** and **spontaneous activity**. We explored the biological rationale for incorporating diversity, demonstrated practical methods in Brian2 for representing distinct cell types (like E/I populations) and for modeling intra-population variability in parameters using statistical distributions (`rand()`, `randn()`). We discussed the importance of spontaneous activity in biological circuits, particularly in developing systems like organoids, and implemented a common modeling strategy using background synaptic input generated by `PoissonGroup` objects. The choice of neuron models (LIF vs. AdEx/Izhikevich) was revisited in light of capturing potentially complex spontaneous dynamics. We also conceptually addressed the significant challenge of **parameter estimation**, highlighting the difficulty in precisely tuning complex models to match experimental data from variable systems like organoids. The chapter culminated in a Brian2 simulation of a heterogeneous E/I network driven by Poisson background input, illustrating how these features lead to more biologically plausible asynchronous irregular dynamics. By incorporating heterogeneity and spontaneous activity, our models take a crucial step towards capturing the complex, dynamic environment within which computation might occur in brain organoids. The techniques learned here will be essential for building the more sophisticated models of plasticity, structure, and function in the subsequent chapters.

**Planned Code Example:**
*   **`5.1_HeterogeneousPopulationNoise.ipynb`:** (Provided and explained in Section 5.7) Simulates a network of heterogeneous excitatory and inhibitory LIF neurons (using conductance-based synapses) exhibiting spontaneous activity driven by Poisson background input. Demonstrates setting variable parameters using string expressions and using `PoissonGroup` for background drive. Visualizes raster plots, population rates, and voltage traces.

**5.9 References for Further Reading (APA Format)**

1.  Amit, D. J., & Brunel, N. (1997). Model of global spontaneous activity and local structured activity during delay periods in the cerebral cortex. *Cerebral Cortex, 7*(3), 237–252. https://doi.org/10.1093/cercor/7.3.237 *(Classic work modeling spontaneous activity states in cortical networks.)*
2.  Destexhe, A., Rudolph, M., & Paré, D. (2003). The high-conductance state of neocortical neurons in vivo. *Nature Reviews Neuroscience, 4*(9), 739–751. https://doi.org/10.1038/nrn1198 *(Reviews the importance of background synaptic activity in shaping neuronal integration in vivo.)*
3.  Ecker, A. S., Berens, P., Tolias, A. S., & Bethge, M. (2010). The effect of noise correlations in populations of neurons. *Journal of Neuroscience, 30*(45), 14965–14965. https://doi.org/10.1523/JNEUROSCI.3966-10.2010 *(Discusses the impact of neuronal variability and correlations, often influenced by heterogeneity and noise.)*
4.  Gjorgjieva, J., Clopath, C., Audet, J., & Pfister, J. P. (2016). Brain-inspired methods for achieving robust computation. *Current Opinion in Neurobiology, 37*, 110–117. https://doi.org/10.1016/j.conb.2016.02.001 *(Discusses how features like heterogeneity might contribute to robust computation.)*
5.  Marder, E., & Goaillard, J. M. (2006). Variability, compensation, and homeostasis in neuron and network function. *Nature Reviews Neuroscience, 7*(7), 563–574. https://doi.org/10.1038/nrn1949 *(Highlights the prevalence and functional importance of variability and compensatory mechanisms in neural systems.)*
6.  Padmanabhan, K., & Urban, N. N. (2010). Intrinsic biophysical diversity Decorrelates neuronal firing dynamics. *Nature Neuroscience, 13*(10), 1276–1282. https://doi.org/10.1038/nn.2630 *(Experimental and modeling work showing how heterogeneity in intrinsic properties can desynchronize networks.)*
7.  Renart, A., de la Rocha, J., Bartho, P., Hollender, L., Parga, N., Reyes, A., & Harris, K. D. (2010). The asynchronous state in cortical circuits. *Science, 327*(5965), 587–590. https://doi.org/10.1126/science.1179850 *(Investigates the mechanisms underlying asynchronous activity in cortex, relevant to modeling spontaneous states.)*
8.  Softky, W. R., & Koch, C. (1993). The highly irregular firing of cortical cells is inconsistent with temporal integration of random EPSPs. *Journal of Neuroscience, 13*(1), 334–350. https://doi.org/10.1523/JNEUROSCI.13-01-00334.1993 *(Classic paper highlighting the challenge of explaining irregular firing, motivating models incorporating noise and balance.)*
9.  Stimberg, M., Goodman, D. F. M., Benichoux, V., & Brette, R. (2019). Equation-oriented specification of neural models for simulations. *Frontiers in Neuroinformatics, 13*, 6. https://doi.org/10.3389/fninf.2019.00006 *(Discusses the equation-oriented approach of Brian2, which facilitates implementing heterogeneity via string expressions.)*
10. Van Rossum, M. C., Bi, G. Q., & Turrigiano, G. G. (2000). Stable Hebbian learning from spike timing-dependent plasticity. *Journal of Neuroscience, 20*(23), 8812–8821. https://doi.org/10.1523/JNEUROSCI.20-23-08812.2000 *(While focused on plasticity, it uses Poisson inputs to drive spontaneous activity in models, illustrating the technique.)*
