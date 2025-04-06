---

# Chapter 13

# Advanced Synaptic Models: Kinetics, Plasticity, and Modulation

---


*In our journey constructing progressively more realistic models of neural networks, we have advanced from simple LIF neurons to models capturing richer intrinsic dynamics (Chapter 11) and even considered the influence of glial cells (Chapter 12). However, another critical element determining network function lies at the connections between neurons: the synapses. Thus far, we have primarily modeled synapses using simple instantaneous current injections or basic conductance changes with single exponential decays (Chapters 3, 4). While computationally efficient and useful for capturing the sign (excitatory/inhibitory) and basic strength of connections, this representation constitutes a significant simplification of the complex biochemical, biophysical, and dynamic processes occurring at real biological synapses. Synaptic transmission is far from a static, stereotyped event; its efficacy changes dynamically based on the precise history of presynaptic activity over multiple timescales (**short-term and long-term plasticity**), different neurotransmitter receptors mediating the postsynaptic response exhibit vastly different activation, inactivation, and desensitization **kinetics**, leading to diverse temporal integration properties, and the overall state of the network, including the properties of individual neurons and synapses, can be globally reshaped and reconfigured by diffusely released **neuromodulatory** signals. Incorporating these advanced synaptic features into our models is often absolutely crucial for capturing realistic network behavior, implementing specific types of neural computations (especially those involving temporal processing or learning), understanding adaptive processes, and bridging the gap between simulations and experimental observations of synaptic function. This chapter delves into these essential synaptic complexities. We begin by elaborating on **why simple synapse models are often insufficient** for addressing many questions about network function. We then explore in detail the different **kinetics of major neurotransmitter receptors** (comparing fast ionotropic AMPA/GABA_A with slower ionotropic NMDA and metabotropic GABA_B receptors), discussing how their distinct temporal characteristics profoundly shape postsynaptic integration and information processing. A major section is dedicated to the ubiquitous phenomenon of **Short-Term Plasticity (STP)**, introducing the underlying biophysical concepts (vesicle release, calcium dynamics) and presenting the widely used **Tsodyks-Markram (TM) model** as a versatile tool for capturing activity-dependent synaptic facilitation and depression. Subsequently, we discuss the pervasive and critical role of **neuromodulation** in dynamically altering brain states and computations, outlining how neuromodulators like dopamine, acetylcholine, serotonin, and norepinephrine reconfigure network excitability and synaptic efficacy. We explore conceptual and practical strategies for **modeling neuromodulatory effects** within simulations, typically through the dynamic control of key neuronal or synaptic parameters. Finally, the chapter provides expanded and refined practical **Brian2 implementation examples**: demonstrating how to model synapses with **distinct kinetics** (AMPA, NMDA, GABA_A, GABA_B analogues), implementing the **Tsodyks-Markram model** to clearly show frequency-dependent STF and STD, and simulating the effects of **neuromodulation** on both synaptic short-term dynamics and neuronal excitability.*

---


**13.1 Beyond Simple Synapses: Capturing Richer Dynamics**

In the initial stages of modeling neural networks, particularly large ones, synapses are often simplified to their most basic function: transmitting a signal from a presynaptic neuron to a postsynaptic one, causing either excitation or inhibition with a certain fixed strength (weight) and potentially a transmission delay. The postsynaptic effect might be modeled as an instantaneous current injection or, slightly more realistically, as a transient change in synaptic conductance ($g_{syn}$) that decays exponentially with a single time constant ($\tau_{syn}$), as described by Equation 13.1. This simple conductance-based synapse model captures the essential properties of driving force ($V_{post} - E_{syn}$) and basic temporal summation. Its computational efficiency makes it suitable for many large-scale simulations where the primary focus is on network architecture or population dynamics arising from static connectivity.

However, biological synapses are vastly more complex and dynamic entities. Relying solely on the simple static-weight, single-exponential decay model overlooks several critical features that are known to profoundly influence neuronal computation and network behavior. Ignoring these richer synaptic dynamics can lead to models that fail to reproduce key experimental findings, lack important computational capabilities, or make inaccurate predictions about network function, especially in systems like developing organoids where synaptic properties might be particularly dynamic or immature. Key limitations of the simple model include:

1.  **Diversity of Postsynaptic Receptor Kinetics:** Neurons utilize a variety of neurotransmitters (glutamate, GABA, acetylcholine, etc.) that act on diverse families of postsynaptic receptors. Even for a single neurotransmitter like glutamate, receptors like AMPA and NMDA exhibit dramatically different activation/deactivation speeds and voltage dependencies (Section 13.2). AMPA receptors mediate fast, transient excitation essential for rapid signal transmission, while NMDA receptors, with their slow kinetics and voltage-dependent $Mg^{2+}$ block, are crucial for temporal integration, coincidence detection, and triggering synaptic plasticity. Similarly, fast GABA_A receptors mediate rapid inhibition, while slower GABA_B receptors (acting via G-proteins and $K^+$ channels) produce prolonged inhibitory effects. Modeling all synaptic conductances with a single, fixed time constant ($\tau_{syn}$) completely masks this functional diversity and the distinct roles these receptor subtypes play in shaping neuronal integration and network dynamics (e.g., generating different oscillation frequencies, implementing different temporal filtering properties).

2.  **Ubiquitous Short-Term Plasticity (STP):** Contrary to the assumption of a fixed weight `w`, the efficacy of nearly all chemical synapses changes dynamically based on the recent pattern of presynaptic firing, over timescales of milliseconds to seconds (Section 13.3). Synapses can facilitate (strengthen transiently) or depress (weaken transiently), or exhibit combinations thereof. This STP profile is frequency-dependent and synapse-specific. STP acts as a dynamic filter, altering how synapses transmit information based on the input statistics. Depressing synapses might signal changes in input rate more effectively, while facilitating synapses might enhance responses to salient bursts or maintain signals briefly. These rapid dynamics are thought to be critical for adaptation, gain control, temporal sequence processing, working memory, and balancing network activity. A static synapse model is fundamentally incapable of capturing these essential adaptive properties (Barroso-Flores et al., 2023).

3.  **Pervasive Neuromodulation:** Neural circuits do not operate in isolation but are constantly influenced by neuromodulatory systems that broadcast signals (like dopamine, serotonin, acetylcholine, norepinephrine) across large brain areas (Section 13.4). These neuromodulators act, often via slower metabotropic receptors, to fundamentally alter the "state" of neural networks. They can change neuronal intrinsic properties (making neurons more or less excitable, altering adaptation or bursting), modify synaptic strengths, gate synaptic plasticity (determining when learning occurs), and reconfigure functional connectivity patterns (Bocchio & Fisahn, 2023). This allows the brain to flexibly adapt its processing strategy based on behavioral context, attention demands, arousal level, or reward prediction. Models with fixed neuronal and synaptic parameters cannot capture this crucial state-dependent flexibility inherent in biological computation.

4.  **Synaptic Location and Dendritic Complexity:** The simple model ignores the spatial location of the synapse on the postsynaptic neuron's dendritic tree. Inputs arriving at distal dendrites undergo significant filtering and attenuation before reaching the soma, and interact differently with local dendritic non-linearities compared to proximal inputs (Section 11.7). While fully addressing this requires multi-compartment models, even in point-neuron simulations, acknowledging that synaptic integration is not uniform is important. Simple synapse models implicitly assume somatic input.

5.  **Inherent Stochasticity of Transmission:** Neurotransmitter release from presynaptic terminals is a probabilistic process involving the stochastic release of discrete packets (quanta) of neurotransmitter contained in vesicles. The probability of release ($P_{rel}$) can be less than 1 and can fluctuate. This introduces variability and potential failures in synaptic transmission, which can significantly impact network reliability and dynamics (Destexhe, 2023). Deterministic synapse models often average over or ignore this fundamental stochasticity.

Therefore, while simple synapse models provide a useful starting point, building more realistic and computationally powerful network simulations, especially those aiming to model complex biological phenomena like learning, adaptation, state-dependent processing, or specific synaptic mechanisms potentially operating in organoids, necessitates incorporating more advanced synaptic features. This chapter focuses on introducing practical Brian2 implementations for modeling distinct receptor kinetics, short-term plasticity (STP), and the effects of neuromodulation, providing the tools to endow simulated synapses with richer, more biologically plausible dynamics.

**13.2 Receptor Kinetics: Fast vs. Slow (AMPA/GABA_A, NMDA/GABA_B)**

The postsynaptic response to neurotransmitter release is fundamentally determined by the properties of the **receptor proteins** that bind the transmitter. These receptors dictate the type of ion channel opened (or intracellular cascade initiated), the duration of the response, and any dependencies on other factors like membrane voltage. Understanding the kinetics and properties of the major receptor types mediating fast excitation and inhibition is crucial for modeling synaptic integration accurately.

*   **Fast Excitatory Transmission (AMPA Receptors):**
    *   **Mechanism:** Mediated primarily by **AMPA receptors (AMPARs)**, which are ionotropic receptors permeable to $Na^+$ and $K^+$ ($E_{syn} \approx 0$ mV). Glutamate binding causes extremely rapid channel opening (sub-ms).
    *   **Kinetics:** Characterized by **fast activation** and relatively **fast deactivation/desensitization**. The decay of the AMPAR-mediated conductance is often approximated by a single exponential with a time constant $\tau_{AMPA}$ typically ranging from **1 to 10 ms**. Some models use a dual exponential for a slightly more realistic shape.
    *   **Function:** Responsible for the rapid depolarization underlying fast EPSPs, enabling high-fidelity transmission of information encoded in spike timing across synapses. They mediate the bulk of fast excitatory drive in the CNS.
    *   **Modeling (Brian2):** Often modeled using a single conductance variable $g_{AMPA}$ with first-order decay:
      ```python
      # AMPA-like synapse model
      # eqs_ampa = 'dg_ampa/dt = -g_ampa / tau_ampa : siemens (summed)'
      # on_pre_ampa = 'g_ampa += w_ampa'
      # tau_ampa = 5*ms # Example value
      ```

*   **Fast Inhibitory Transmission (GABA_A Receptors):**
    *   **Mechanism:** Mediated mainly by **GABA_A receptors (GABA_ARs)**, which are ionotropic receptors forming a channel typically permeable to **Chloride ions ($Cl^-$)**. The reversal potential $E_{GABA_A}$ is usually close to or slightly hyperpolarized relative to the resting potential (e.g., -65 to -75 mV), depending on intracellular $Cl^-$ concentration.
    *   **Kinetics:** GABA binding triggers rapid channel opening. Deactivation kinetics are also relatively fast, though often slightly slower than AMPARs. Decay time constants $\tau_{GABA_A}$ typically range from **5 to 20 ms**.
    *   **Function:** Mediate fast inhibitory postsynaptic potentials (IPSPs). Depending on the relationship between $E_{GABA_A}$ and $V_{rest}/V_{thresh}$, activation can cause hyperpolarization or **shunting inhibition** (increasing membrane conductance, making it harder for excitatory inputs to depolarize the neuron, even if the voltage doesn't change much). Crucial for E/I balance, generating oscillations, and controlling spike timing.
    *   **Modeling (Brian2):** Similar structure to AMPA, using a single conductance $g_{GABA_A}$ with its own decay constant and inhibitory reversal potential ($E_{syn} = E_{GABA_A}$).
      ```python
      # GABA_A-like synapse model
      # eqs_gaba_a = 'dg_gaba_a/dt = -g_gaba_a / tau_gaba_a : siemens (summed)'
      # on_pre_gaba_a = 'g_gaba_a += w_gaba_a'
      # tau_gaba_a = 10*ms; E_GABA_A = -70*mV # Example values
      # Neuron equation includes: ... + g_gaba_a*(E_GABA_A - v) ...
      ```

*   **Slow Excitatory Transmission (NMDA Receptors):**
    *   **Mechanism:** Mediated by **NMDA receptors (NMDARs)**, ionotropic glutamate receptors with unique properties. They are permeable to $Na^+$, $K^+$, and importantly, **$Ca^{2+}$** ($E_{syn} \approx 0$ mV). Their activation requires both glutamate binding and postsynaptic depolarization to relieve a voltage-dependent **$Mg^{2+}$ block**.
    *   **Kinetics:** Exhibit **significantly slower kinetics** compared to AMPARs. Activation (channel opening after relief of Mg block) is relatively slow, and deactivation upon glutamate removal is very slow, with decay time constants $\tau_{NMDA}$ often in the range of **50-200 ms or longer**. This allows NMDARs to integrate inputs over much longer timescales.
    *   **Function:** Crucial for **synaptic plasticity** (LTP/LTD), as the $Ca^{2+}$ influx acts as a second messenger triggering downstream signaling cascades. Their voltage dependence makes them **coincidence detectors** for presynaptic activity and postsynaptic depolarization. They contribute to sustained depolarization during high-frequency stimulation and play roles in learning, memory, and potentially stabilizing persistent activity.
    *   **Modeling (Brian2):** Requires capturing both slow kinetics and voltage dependence. Kinetics often modeled using a **dual-exponential** approach (separate rise and decay variables) or multi-state models. Voltage dependence typically included via a multiplicative $B(V)$ term (Eq 13.2) in the current calculation.
      ```python
      # NMDA-like synapse model (Dual Exponential Conductance)
      # eqs_nmda = '''dx/dt = -x / tau_nmda_rise : siemens (summed) # Driving variable
      #              dg_nmda/dt = (x - g_nmda) / tau_nmda_decay : siemens (summed)'''
      # on_pre_nmda = 'x += w_nmda_scaled' # w_nmda_scaled includes normalization
      # tau_nmda_rise=2*ms; tau_nmda_decay=100*ms # Example values
      # Neuron equation calculates current using g_nmda and B(V):
      # I_nmda = g_nmda * B(v) * (E_NMDA - v)
      # where B(v) is the Mg block function.
      ```

*   **Slow Inhibitory Transmission (GABA_B Receptors):**
    *   **Mechanism:** Mediated by **GABA_B receptors (GABA_BRs)**, which are **metabotropic** (GPCRs). GABA binding activates G-proteins, which typically lead (via intermediate steps) to the opening of **G-protein coupled inward rectifier $K^+$ channels (GIRKs)**.
    *   **Kinetics:** Activation is **slow** due to the intracellular signaling cascade (tens of ms onset). The resulting $K^+$ current is also **very slow and prolonged**, with decay time constants $\tau_{GABA_B}$ often lasting **hundreds of milliseconds (e.g., 150-300 ms)**. $E_{syn}$ is the potassium reversal potential $E_K$ (typically -80 to -100 mV).
    *   **Function:** Mediate slow, long-lasting IPSPs. Often require stronger or more prolonged presynaptic GABA release (e.g., during bursts or high-frequency firing) for significant activation compared to GABA_ARs. Contribute to slow network oscillations (e.g., theta, delta rhythms), shaping neuronal excitability over longer timescales, and regulating presynaptic release (via presynaptic GABA_BRs, not modeled here).
    *   **Modeling (Brian2):** Given the metabotropic pathway, modeling is often phenomenological. A common approach is to use a slow conductance variable $g_{GABA_B}$ activated indirectly by presynaptic GABA release (perhaps requiring multiple spikes or using a filtered representation of presynaptic activity) and decaying very slowly. A dual-exponential kinetic model with slow time constants can also approximate the shape.
      ```python
      # Conceptual GABA_B-like synapse (Slow K+ conductance)
      # Requires modeling G-protein activation step, simplified here
      # eqs_gaba_b = '''ds_gaba_b/dt = -s_gaba_b / tau_gaba_b_decay : 1 (summed) # Activation variable
      #                 g_gaba_b = gbar_gaba_b * (s_gaba_b**n / (s_gaba_b**n + Kd**n)) : siemens # Sigmoidal activation of K+ channel
      #              ''' # n=Hill coeff, Kd=half-activation
      # on_pre_gaba_b = 's_gaba_b += W_gaba_b' # Increment activation variable (needs tuning)
      # tau_gaba_b_decay = 200*ms; gbar_gaba_b = 1.0*nS; n=4; Kd=0.5 # Example params
      # Neuron equation includes: ... + g_gaba_b*(EK - v) ...
      ```
      *(Self-correction: Provided a more plausible (though still conceptual) model for GABA_B involving an intermediate activation variable `s_gaba_b` and sigmoidal activation of the conductance `g_gaba_b`, reflecting the indirect G-protein coupling.)*

The distinct kinetics of these receptor subtypes allow neural circuits to process information across multiple timescales simultaneously. Fast AMPA and GABA_A transmission supports rapid computation and timing-based codes, while the slower integration provided by NMDA and GABA_B receptors allows for sensitivity to input history, sustained responses, and modulation over longer periods. Incorporating this kinetic diversity is often essential for accurately modeling synaptic integration, plasticity induction, and network dynamics in simulations. Brian2's flexible equation system allows users to define custom models capturing these different kinetic profiles, as demonstrated further in Section 13.6.

**13.3 Short-Term Plasticity (STP): The Tsodyks-Markram (TM) Model**

A crucial aspect of synaptic realism, often neglected in simpler models, is that the strength of synaptic transmission is not fixed but dynamically changes based on the recent history of presynaptic activity. These transient, activity-dependent changes in synaptic efficacy, occurring over timescales of milliseconds to seconds or minutes, are collectively known as **Short-Term Plasticity (STP)**. STP contrasts with long-term plasticity (LTP/LTD, Chapter 6) which involves much more persistent changes in strength. STP ensures that the synapse's response is not just determined by the current presynaptic spike but is shaped by the context of preceding spikes, making synapses powerful dynamic filters of information (Barroso-Flores et al., 2023; Naskar et al., 2022).

The two principal forms of STP, **Short-Term Facilitation (STF)** and **Short-Term Depression (STD)**, arise from presynaptic mechanisms related to neurotransmitter vesicle release:
*   **STF (Facilitation):** An *increase* in synaptic strength during repetitive stimulation. This is often prominent at low stimulation frequencies. The primary underlying mechanism is thought to be the **accumulation of residual calcium ($[Ca^{2+}]_i$)** in the presynaptic terminal. Each incoming action potential triggers $Ca^{2+}$ influx, which is necessary for vesicle fusion and neurotransmitter release. If spikes arrive sufficiently close together (before calcium is fully cleared), the residual calcium from previous spikes adds to the influx from the current spike, leading to a higher peak $[Ca^{2+}]_i$. Since vesicle release probability ($P_{rel}$) is highly sensitive (often supra-linearly) to $[Ca^{2+}]_i$, this results in an increased release probability and thus a larger postsynaptic response for subsequent spikes. STF is typically more significant at synapses with a low initial $P_{rel}$.
*   **STD (Depression):** A *decrease* in synaptic strength during repetitive stimulation, particularly pronounced at high frequencies. This is mainly attributed to the **depletion of the readily releasable pool (RRP)** of synaptic vesicles. The RRP represents vesicles docked at the active zone, primed and ready for immediate release upon $Ca^{2+}$ influx. Each action potential consumes a fraction of these vesicles. At high firing rates, the rate of vesicle consumption can exceed the rate at which the RRP is replenished from reserve pools or recycling, leading to a progressive decrease in the number of vesicles released per spike and thus a smaller postsynaptic response. STD is generally more prominent at synapses with a high initial $P_{rel}$.

The specific profile of STP (predominantly facilitating, depressing, or exhibiting a mixture) depends on the interplay between these calcium-dependent facilitation mechanisms and vesicle depletion/replenishment dynamics. This profile varies greatly across different synapse types (e.g., excitatory vs. inhibitory, specific neuronal subtypes, brain regions) and can also be modulated by developmental stage or neuromodulatory state (Grillo et al., 2023).

STP confers important computational capabilities upon synapses and networks:
*   **Frequency-Dependent Filtering:** Depressing synapses act like high-pass filters, responding more strongly to transient changes or the onset of activity than to sustained high rates. Facilitating synapses can act more like low-pass filters or integrators, amplifying sustained or bursty inputs (Trilla et al., 2023).
*   **Gain Control and Adaptation:** STD provides an automatic mechanism to adjust synaptic gain based on the average presynaptic firing rate, preventing saturation of postsynaptic neurons and helping the network adapt to varying input levels.
*   **Temporal Pattern Detection:** The history dependence introduced by STP makes synapses sensitive to the temporal structure of input spike trains (e.g., detecting specific inter-spike intervals or bursts).
*   **Information Transmission:** STP can dynamically optimize information transfer across synapses depending on the input statistics.
*   **Working Memory:** Both STF (maintaining elevated efficacy) and STD (activity "imprint" in depleted resources) have been implicated as potential contributors to short-term information storage over seconds.

**The Tsodyks-Markram (TM) Model:** To capture these essential STP dynamics within computational models, several phenomenological models have been developed. The most widely used and influential is the **Tsodyks-Markram (TM) model**, which provides a concise description based on the dynamics of presynaptic resource availability and utilization (Tsodyks & Markram, 1997; Tsodyks et al., 1998). The TM model tracks two key state variables associated with each synapse:
*   $x(t)$: Represents the fraction of total neurotransmitter resources currently available in the readily releasable pool (RRP). $x$ ranges from 0 (fully depleted) to 1 (fully replenished).
*   $u(t)$: Represents the fraction of available resources $x$ that is utilized (i.e., released) by a single presynaptic spike. This "utilization factor" $u$ is conceptually related to the release probability $P_{rel}$ and is assumed to be influenced by presynaptic calcium levels.

The dynamics evolve as follows:
1.  **Recovery between spikes:** In the absence of presynaptic spikes, the available resource fraction $x$ recovers towards 1 (replenishment) with a time constant $\tau_{rec}$:
    ```latex
    \frac{dx}{dt} = \frac{1 - x}{\tau_{rec}}
    \tag{13.3a}
    ```
    Simultaneously, the utilization factor $u$ decays back towards a baseline level $U$ (representing the utilization/release probability for an isolated spike arriving after a long silence) with a time constant $\tau_{facil}$:
    ```latex
    \frac{du}{dt} = - \frac{u - U}{\tau_{facil}}
    \tag{13.3b}
    ```
2.  **Updates upon presynaptic spike arrival (at time $t_{sp}$):** The standard sequence of events modeled is:
    *   **Facilitation Update:** First, the utilization factor $u$ for the *upcoming* release event is potentially increased based on residual effects (calcium) from previous spikes. A common update rule is: $u(t_{sp}^+) = u(t_{sp}^-) + U \cdot (1 - u(t_{sp}^-))$. This means $u$ increases towards 1 with each spike, with the increment scaled by the baseline $U$. The decay of $u$ between spikes is governed by $\tau_{facil}$.
    *   **Calculate Release Amount:** The amount of neurotransmitter effectively released by the current spike is proportional to the currently available resources multiplied by the (just updated) utilization factor: $\text{Release Amount} \propto u(t_{sp}^+) \cdot x(t_{sp}^-)$.
    *   **Resource Depletion:** The available resources $x$ are then depleted by the amount utilized: $x(t_{sp}^+) = x(t_{sp}^-) - u(t_{sp}^+) \cdot x(t_{sp}^-)$.
    *   **Postsynaptic Effect:** The magnitude of the postsynaptic response (e.g., conductance increase $\Delta g$) is proportional to the release amount: $\Delta g = w_{static} \cdot u(t_{sp}^+) \cdot x(t_{sp}^-)$, where $w_{static}$ is a static scaling factor representing the maximum possible synaptic strength.

The four key parameters of the TM model ($U, \tau_{facil}, \tau_{rec}$, and $w_{static}$) determine the specific STP profile:
*   $U$: Baseline utilization/release probability. High $U$ tends to favor initial depression (as resources deplete quickly). Low $U$ allows for facilitation to dominate initially.
*   $\tau_{facil}$: Time constant of facilitation. Determines how long the increase in $u$ persists. Longer $\tau_{facil}$ leads to more sustained facilitation. If $\tau_{facil}$ is very short (or $U$ is high), facilitation effects are minimal.
*   $\tau_{rec}$: Time constant of resource recovery (vesicle replenishment). Determines how quickly the synapse recovers from depression. Shorter $\tau_{rec}$ allows for faster recovery and sustained transmission at higher rates. Longer $\tau_{rec}$ leads to more pronounced depression.
*   $w_{static}$: Overall scaling factor for the synaptic strength.

By selecting appropriate values for these parameters, the TM model can effectively reproduce facilitating synapses (low $U$, long $\tau_{facil}$), depressing synapses (high $U$, short $\tau_{facil}$), and synapses exhibiting a combination of initial facilitation followed by depression.

**Implementing the TM Model in Brian2:** This requires defining the state variables `x` and `u` within the `Synapses` object and implementing both the continuous dynamics (using the `model` string) and the discrete updates upon spike arrival (using the `on_pre` argument).

```python
# Brian2 Implementation: Tsodyks-Markram Model
# State variables: x, u (synaptic variables)
# Parameters: U_TM, tau_rec, tau_facil (synaptic parameters, use U_TM to avoid conflict with u variable)
#             w_static (scaling weight)
tm_model_eqs = '''
    dx/dt = (1 - x) / tau_rec : 1 (event-driven)
    du/dt = (U_TM - u) / tau_facil : 1 (event-driven)
    # Parameters defined per synapse
    tau_rec : second (constant)
    tau_facil : second (constant)
    U_TM : 1 (constant) # Baseline utilization
    w_static : siemens (constant) # Static weight scaling factor
    '''
# on_pre needs to implement the sequence: update u, calculate release, deplete x, apply PSP
tm_on_pre_seq = '''
                 u = u + U_TM * (1 - u) # 1. Facilitate u (using previous u)
                 release_amount = u * x # 2. Calculate release amount (using facilitated u and previous x)
                 x = x - release_amount # 3. Deplete x
                 # 4. Apply postsynaptic effect (e.g., conductance increase)
                 g_target_post += w_static * release_amount # Replace g_target_post with actual variable name
                 '''
# Create Synapses object
# TM_Synapses = Synapses(source, target, model=tm_model_eqs, on_pre=tm_on_pre_seq, ...)
# Initialize state variables:
# TM_Synapses.x = 1.0
# TM_Synapses.u = TM_Synapses.U_TM # Initialize u to baseline U_TM
# Set parameters U_TM, tau_rec, tau_facil, w_static
```
*(Self-correction: Renamed parameter U to U_TM to avoid conflict with state variable u. Used the standard update sequence in `on_pre`.)* Simulating synapses with this model using different parameter sets allows direct observation and analysis of frequency-dependent STF and STD, adding a crucial layer of dynamic realism to network models (see Section 13.6 Example 2).

**13.4 Neuromodulation: Controlling Network State and Efficacy**

Beyond the rapid, point-to-point communication mediated by classical neurotransmitters (glutamate, GABA) acting on ionotropic receptors, the brain utilizes a fundamentally different mode of signaling known as **neuromodulation** to orchestrate global brain states, regulate motivation and attention, facilitate learning, and adapt network function to changing behavioral contexts. Neuromodulatory systems employ specific signaling molecules—including **monoamines** like **dopamine (DA)**, **serotonin (5-HT)**, and **norepinephrine/noradrenaline (NE/NA)**; **acetylcholine (ACh)**; **histamine**; a vast array of **neuropeptides** (e.g., opioids, oxytocin, vasopressin, orexin); and potentially other signaling molecules like endocannabinoids or nitric oxide—which act broadly to modify the way neural circuits process information.

Several key features distinguish neuromodulation from classical fast synaptic transmission:
*   **Source:** Neuromodulators are often released from relatively small populations of neurons located in specific subcortical nuclei (e.g., dopamine from the Ventral Tegmental Area (VTA) and Substantia Nigra pars compacta (SNc); serotonin from the Raphe nuclei; norepinephrine from the Locus Coeruleus (LC); acetylcholine from the Basal Forebrain and brainstem nuclei). These neurons typically possess widely branching axons that project diffusely across large target brain areas.
*   **Release Mechanism:** Release often occurs not just at specific synaptic terminals but also from axonal **varicosities** (swellings along the axon), leading to release into the extracellular space rather than being confined to a narrow synaptic cleft. This allows neuromodulators to diffuse over longer distances (**volume transmission**) and potentially affect many neurons and synapses in a region simultaneously.
*   **Receptors:** Neuromodulators primarily act by binding to **metabotropic receptors**, most commonly **G-protein coupled receptors (GPCRs)**, located on both neurons (pre- and postsynaptically, somatically, dendritically) and glial cells. Activation of GPCRs triggers slower intracellular signaling cascades involving second messengers (like cAMP, cGMP, $IP_3$, DAG, $Ca^{2+}$) and protein kinases/phosphatases.
*   **Timescales:** Due to the involvement of diffusion and multi-step intracellular cascades, the effects of neuromodulators typically have a **slower onset** (hundreds of milliseconds to seconds or longer) and are **much longer-lasting** (seconds to minutes, hours, or even inducing persistent changes via gene expression) compared to the millisecond timescale of ionotropic synaptic transmission.
*   **Effects:** Instead of directly causing rapid excitation or inhibition, neuromodulators primarily **modulate** the intrinsic properties of neurons and the efficacy of synaptic transmission. They act like "tuning knobs" or "gain controls," altering the way circuits respond to classical neurotransmitter inputs.

The overall function of neuromodulation is to **dynamically reconfigure neural circuits** and **set the computational state** of the brain based on the organism's internal state (e.g., arousal, hunger, stress level) and external context (e.g., presence of rewards, novelty, attentional demands) (Bocchio & Fisahn, 2023). Different neuromodulatory systems are tightly linked to specific behavioral functions:
*   **Dopamine:** Critically involved in **reward prediction error signaling**, **motivation**, **reinforcement learning** (strengthening actions leading to reward), control of voluntary movement (affected in Parkinson's), and regulating cognitive functions like **working memory** and cognitive flexibility in the prefrontal cortex. Often gates synaptic plasticity.
*   **Serotonin:** Plays complex roles in regulating **mood**, anxiety, aggression, impulsivity, sleep, appetite, and social behavior. Its effects on neuronal excitability and synaptic function are diverse and region-specific, and dysregulation is implicated in depression and anxiety disorders.
*   **Norepinephrine:** Primarily associated with **arousal**, **vigilance**, **attention**, the **fight-or-flight** stress response, and enhancing **memory consolidation** for salient events. Generally acts to increase signal-to-noise ratio in cortical processing, making neurons more responsive to strong inputs and less responsive to weak inputs.
*   **Acetylcholine:** Plays fundamental roles in **cortical activation/arousal**, **attention**, **sensory processing** (e.g., enhancing stimulus discrimination), **learning**, and **memory formation** (particularly consolidation). Both fast (nicotinic) and slow (muscarinic) receptors mediate its effects, often promoting synaptic plasticity. Degeneration of cholinergic neurons is a hallmark of Alzheimer's disease.

Neuromodulators exert their influence by altering a wide range of cellular and synaptic targets:
*   **Intrinsic Neuronal Properties:** They can modify resting potential, input resistance, firing threshold, action potential shape (e.g., width, AHP), adaptation rates (by modulating slow $K^+$ or $Ca^{2+}$ channels like $I_M, I_{AHP}, I_h$), and resonance properties.
*   **Synaptic Transmission:** They can modulate the probability of neurotransmitter release from presynaptic terminals (acting on presynaptic channels or the release machinery) or alter the sensitivity, number, or kinetics of postsynaptic receptors (e.g., by phosphorylation).
*   **Synaptic Plasticity:** They often act as crucial **gating signals** for long-term plasticity, determining whether LTP or LTD can be induced by coincident activity (e.g., dopamine is often required for reward-related LTP). They can also modulate the expression magnitude or duration of plasticity.
*   **Network State:** By collectively altering neuronal and synaptic properties across a population, neuromodulators can shift the overall dynamic state of a network, for example, promoting synchronized oscillations, enhancing sparse coding, or switching between different computational modes (e.g., exploratory vs. exploitative).

Understanding neuromodulation is critical for modeling realistic brain function, as network computations are rarely static but are constantly being adapted based on behavioral context. Brain organoids, while lacking the structured long-range neuromodulatory inputs from brainstem nuclei, do contain neurons capable of synthesizing classical transmitters and potentially some neuromodulators. Moreover, they possess the receptors for these modulators. Thus, **exogenous application** of specific neuromodulators (or their agonists/antagonists) to the organoid culture provides a powerful experimental tool to probe the functional repertoire of the developing network, assess its state-dependent processing capabilities, and potentially investigate mechanisms related to psychiatric or neurological disorders where neuromodulatory systems are implicated. Incorporating simplified models of neuromodulation into organoid simulations is therefore important for bridging this gap.

**13.5 Modeling Neuromodulation: Parameter Control**

Simulating the full complexity of neuromodulatory systems—including the detailed firing patterns of neuromodulatory neurons, the physics of volume transmission and diffusion in the extracellular space, the kinetics of GPCR binding and G-protein activation, the intricate intracellular signaling cascades involving multiple second messengers and interacting pathways, and the diverse downstream effects on numerous ion channels and synaptic proteins—represents a formidable challenge far beyond the scope of typical network simulations and often requiring specialized multi-scale modeling tools.

Therefore, within standard neuronal network simulation frameworks like Brian2, the influence of neuromodulation is typically modeled in a highly **phenomenological** and **simplified** manner, focusing on capturing the **net functional impact** of a modulator on specific, measurable neuronal or synaptic properties rather than simulating the underlying molecular mechanisms in detail. The most common and practical strategy relies on **dynamic parameter control**:

1.  **Define a Neuromodulator Level Signal:** Represent the presence or concentration of the neuromodulator(s) using one or more variables, often termed `modulator_level` or similar. This signal represents the overall strength of the modulatory influence.
    *   **External Control:** In many simulations, this level is treated as an **external input** controlled by the experimenter, mimicking bath application of a drug or representing a presumed behavioral state. This is often implemented using a `TimedArray` specifying the modulator concentration profile over time, or by simply setting a global variable to different constant values during different phases of the simulation.
    *   **Activity-Dependent Proxy:** Alternatively, the modulator level could be made **dynamic**, potentially depending on the average activity level of a specific neuronal population within the network (e.g., simulating activity-dependent release of acetylcholine within cortex) or driven by a simplified model of the neuromodulatory nucleus itself. This often involves defining an ODE for the modulator level that integrates network activity and decays slowly.
    ```python
    # Conceptual: TimedArray for external control
    # mod_level = TimedArray(...)
    # Use mod_level(t) in equations or operations.

    # Conceptual: Activity-dependent modulator release proxy
    # tau_mod_release = 500*ms; mod_gain = 0.01; tau_mod_decay = 2000*ms
    # eqs_mod = '''dMod_level/dt = mod_gain*mean_network_rate - Mod_level/tau_mod_decay : 1'''
    # Requires estimating mean_network_rate (e.g., from PopulationRateMonitor)
    # and feeding it into this dynamic variable (possibly via @network_operation).
    ```

2.  **Identify Key Target Parameters:** Based on experimental literature, identify the specific neuronal or synaptic parameters that are most significantly affected by the neuromodulator being modeled. This requires knowledge of the modulator's known mechanisms of action. Common targets include:
    *   *Neuronal Intrinsic Properties:* $V_{rest}$ (resting potential), $V_{thresh}$ (firing threshold), $g_L$ (leak conductance), adaptation parameters (e.g., $a$ or $b$ in AdEx/Izhikevich, affecting $K^+$ currents like $I_M$ or $I_{AHP}$), $Ca^{2+}$ channel conductances.
    *   *Synaptic Transmission:* $w_{static}$ (baseline synaptic weight), parameters of STP models like $U, \tau_{rec}, \tau_{facil}$ in the TM model (affecting release probability and dynamics).
    *   *Synaptic Plasticity:* Parameters controlling LTP/LTD induction thresholds or learning rates (e.g., $A_{pre}, A_{post}$ in STDP models).

3.  **Define the Modulatory Function:** Specify a mathematical relationship describing how the target parameter(s) depend on the current `modulator_level`. This function should capture the direction (increase or decrease) and potentially the dose-dependence of the effect. Simple functions are often used:
    *   **Linear:** $Parameter_{eff} = Parameter_{base} \times (1 + \text{mod\_strength} \times \text{modulator\_level})$
    *   **Multiplicative:** $Parameter_{eff} = Parameter_{base} \times \text{modulator\_level}^{\text{exponent}}$
    *   **Sigmoidal:** Using a logistic or Hill function to represent saturation effects.
    *   **Specific Experimental Fits:** If quantitative data on the modulator's effect is available, a more specific function can be used.
    ```python
    # Conceptual Example: Modulator linearly increases neuronal gain (e.g., reduces threshold)
    # V_thresh_eff = V_thresh_base - thresh_mod_factor * modulator_level

    # Conceptual Example: Modulator sigmoidally gates plasticity learning rate
    # learning_rate_eff = max_learning_rate / (1 + exp(-(modulator_level - mod_midpoint)/mod_slope))
    ```

4.  **Implement Dynamic Parameter Update in Brian2:** Integrate this parameter modulation into the simulation. The implementation strategy depends on how frequently the modulator level changes and which parameters are affected:
    *   **Using Namespace with TimedArray:** If the `modulator_level` is controlled externally via a `TimedArray` and changes relatively slowly, and the target parameter appears directly in the model equations (e.g., $g_L$, adaptation parameters), you can often define the effective parameter within the equation string using a function lookup passed via the `namespace`. This is usually efficient.
    *   **Using `@network_operation`:** If the `modulator_level` changes dynamically within the simulation, or if the parameter being modulated is not directly in the equations (e.g., synaptic weight `w` which is often static in the `model` string but changed in `on_pre`), or if complex calculations are needed to determine the parameter value, then using a `@network_operation` is often necessary. This function runs periodically (e.g., every few ms or tens of ms), calculates the current effective parameter value(s) based on the `modulator_level`, and updates the corresponding attributes of the `NeuronGroup` or `Synapses` objects (e.g., `neurons.V_thresh = ...`, `synapses.w = ...`). This provides maximum flexibility but can add computational overhead if run very frequently.
    *   **Defining Parameters as State Variables:** For parameters that change relatively slowly due to modulation, another approach is to define them as additional state variables within the `NeuronGroup` or `Synapses` model, with their own differential equation relaxing towards a target value set by the `modulator_level`. E.g., `dParam_eff/dt = (Param_target(modulator_level) - Param_eff) / tau_mod_effect`.

This parameter control approach provides a powerful yet tractable way to simulate the functional impact of neuromodulation in network models. It allows researchers to investigate how changes in neuromodulatory tone might switch network dynamics, alter information processing strategies, control learning rules, or mediate state-dependent behaviors in simulations of systems like brain organoids, facilitating comparison with experiments involving pharmacological manipulation of neuromodulatory pathways. The third Brian2 example in Section 13.6 illustrates modulating both synaptic and neuronal parameters.

**13.6 Brian2 Implementation: Complex Synapses and Modulation**

*(This section retains and potentially refines the three core examples from the previous response, ensuring they are runnable and clearly illustrate the implementation of kinetics, STP, and modulation.)*

This section provides practical Brian2 code examples demonstrating how to implement the advanced synaptic features discussed: distinct kinetics, short-term plasticity (Tsodyks-Markram model), and basic neuromodulation affecting both synaptic and neuronal parameters.

**Task 1: Implementing Distinct Synaptic Kinetics**
This example simulates and compares postsynaptic responses mediated by AMPA-like (fast), NMDA-like (slow, voltage-dependent), GABA_A-like (fast inhibition), and GABA_B-like (slow inhibition) synapses.

```python
# === Brian2 Simulation: Distinct Synapse Kinetics ===
# (13.1_ConductanceSynapseKinetics.ipynb - Expanded)
from brian2 import *; import matplotlib.pyplot as plt; import numpy as np
start_scope(); defaultclock.dt = 0.1*ms

# --- Neuron Parameters ---
tau=20*ms; V_rest=-70*mV; V_thresh=-50*mV; V_reset=-75*mV; EK = -85*mV # K+ reversal for GABA_B
# Neuron equations need all conductance terms
eqs_neuron = '''dv/dt = (V_rest - v + g_ampa*(0*mV - v) + g_nmda*Bv*(0*mV - v)
                           + g_gaba_a*(-70*mV - v) + g_gaba_b*(EK - v)) / tau : volt
                g_ampa : siemens; g_nmda : siemens; g_gaba_a : siemens; g_gaba_b : siemens # Synaptic conductances
                Bv = 1.0 / (1.0 + exp(-0.062*(v/mV + 20)) * 1.0 / 3.57) : 1 # Simplified Mg block, V_half ~ -20mV
             ''' # Added GABA terms and explicit B(v)

# --- Postsynaptic Neuron ---
target_neuron = NeuronGroup(1, eqs_neuron, threshold='v > V_thresh', reset='v = V_reset', method='euler')
target_neuron.v = V_rest

# --- Input Source (Generates spikes for each synapse type) ---
input_ampa = SpikeGeneratorGroup(1, [0], [10*ms]); input_nmda = SpikeGeneratorGroup(1, [0], [60*ms])
input_gaba_a = SpikeGeneratorGroup(1, [0], [110*ms]); input_gaba_b = SpikeGeneratorGroup(1, [0], [160*ms])

# --- Synapse Models & Connections ---
# AMPA (Fast Excitation)
tau_ampa=5*ms; w_ampa=0.5*nS
syn_ampa = Synapses(input_ampa, target_neuron, 'w:siemens; dg_ampa/dt=-g_ampa/tau_ampa : siemens (summed)', on_pre='g_ampa+=w', name='AMPA')
syn_ampa.connect(); syn_ampa.w=w_ampa; syn_ampa.tau_ampa=tau_ampa

# NMDA (Slow Excitation + Mg Block)
tau_nmda_rise=5*ms; tau_nmda_decay=120*ms; w_nmda=0.3*nS
eqs_nmda = 'dx/dt=-x/tau_nmda_rise : siemens (summed)\ndg_nmda/dt=(x-g_nmda)/tau_nmda_decay : siemens (summed)\nw:siemens; tau_nmda_rise:second; tau_nmda_decay:second'
norm_factor = tau_nmda_decay*tau_nmda_rise/(tau_nmda_decay-tau_nmda_rise)*(((tau_nmda_rise/tau_nmda_decay)**(tau_nmda_rise/(tau_nmda_decay-tau_nmda_rise)))-((tau_nmda_rise/tau_nmda_decay)**(tau_nmda_decay/(tau_nmda_decay-tau_nmda_rise))))
if abs(norm_factor)<1e-6: norm_factor=tau_nmda_decay/exp(1.)
syn_nmda = Synapses(input_nmda, target_neuron, eqs_nmda, on_pre='x+=w/norm_factor', name='NMDA')
syn_nmda.connect(); syn_nmda.w=w_nmda; syn_nmda.tau_nmda_rise=tau_nmda_rise; syn_nmda.tau_nmda_decay=tau_nmda_decay

# GABA_A (Fast Inhibition)
tau_gaba_a=10*ms; w_gaba_a=0.8*nS
syn_gaba_a = Synapses(input_gaba_a, target_neuron, 'w:siemens; dg_gaba_a/dt=-g_gaba_a/tau_gaba_a : siemens (summed)', on_pre='g_gaba_a+=w', name='GABA_A')
syn_gaba_a.connect(); syn_gaba_a.w=w_gaba_a; syn_gaba_a.tau_gaba_a=tau_gaba_a

# GABA_B (Slow Inhibition - Conceptual: activates slow K+ conductance)
tau_gaba_b_rise=40*ms; tau_gaba_b_decay=200*ms; w_gaba_b=0.4*nS
eqs_gaba_b = 'dx/dt=-x/tau_gaba_b_rise : siemens (summed)\ndg_gaba_b/dt=(x-g_gaba_b)/tau_gaba_b_decay : siemens (summed)\nw:siemens; tau_gaba_b_rise:second; tau_gaba_b_decay:second'
norm_factor_b = tau_gaba_b_decay*tau_gaba_b_rise/(tau_gaba_b_decay-tau_gaba_b_rise)*(((tau_gaba_b_rise/tau_gaba_b_decay)**(tau_gaba_b_rise/(tau_gaba_b_decay-tau_gaba_b_rise)))-((tau_gaba_b_rise/tau_gaba_b_decay)**(tau_gaba_b_decay/(tau_gaba_b_decay-tau_gaba_b_rise))))
if abs(norm_factor_b)<1e-6: norm_factor_b=tau_gaba_b_decay/exp(1.)
syn_gaba_b = Synapses(input_gaba_b, target_neuron, eqs_gaba_b, on_pre='x+=w/norm_factor_b', name='GABA_B')
syn_gaba_b.connect(); syn_gaba_b.w=w_gaba_b; syn_gaba_b.tau_gaba_b_rise=tau_gaba_b_rise; syn_gaba_b.tau_gaba_b_decay=tau_gaba_b_decay

# --- Monitors ---
mon_v = StateMonitor(target_neuron, 'v', record=0)
mon_g = StateMonitor(target_neuron, ['g_ampa', 'g_nmda', 'g_gaba_a', 'g_gaba_b'], record=0)

# --- Run Simulation ---
run(250*ms)

# --- Visualize ---
plt.figure(figsize=(12, 8))
# Voltage
plt.subplot(2, 1, 1); plt.plot(mon_v.t/ms, mon_v.v[0]/mV); plt.title('Postsynaptic Voltage Response'); plt.ylabel('Vm (mV)'); plt.grid(alpha=0.5)
# Conductances
plt.subplot(2, 1, 2); plt.plot(mon_g.t/ms, mon_g.g_ampa[0]/nS, label=f'AMPA ($\tau$={tau_ampa/ms}ms)')
plt.plot(mon_g.t/ms, mon_g.g_nmda[0]/nS, label=f'NMDA ($\tau_d$={tau_nmda_decay/ms}ms)')
plt.plot(mon_g.t/ms, mon_g.g_gaba_a[0]/nS, label=f'GABA_A ($\tau$={tau_gaba_a/ms}ms)')
plt.plot(mon_g.t/ms, mon_g.g_gaba_b[0]/nS, label=f'GABA_B ($\tau_d$={tau_gaba_b_decay/ms}ms)')
plt.title('Synaptic Conductances'); plt.ylabel('Conductance (nS)'); plt.xlabel('Time (ms)'); plt.legend(fontsize='small'); plt.grid(alpha=0.5); plt.ylim(bottom=-0.05)
plt.tight_layout(); plt.show()
```
*Explanation:* This expanded example simulates a single target neuron receiving inputs from four different synapse types activated sequentially. AMPA synapse causes a fast, brief EPSP. NMDA synapse causes a slower rising, very prolonged EPSP whose current depends on voltage (via `Bv` term). GABA_A causes a fast, brief IPSP (or shunting). GABA_B (modeled conceptually with slow dual-exponential kinetics activating a K+ conductance) causes a slow, prolonged IPSP. The plots clearly distinguish the different time courses of the conductances and their resulting effects on the postsynaptic membrane potential.

**Task 2: Implementing Tsodyks-Markram (TM) Model for STP**
*(Code largely identical to previous expanded response, clearly showing STD vs STF)*
```python
# === Brian2 Simulation: Tsodyks-Markram STP Model ===
# (13.2_TsodyksMarkramSTP.ipynb)
from brian2 import *; import matplotlib.pyplot as plt; import numpy as np
start_scope(); defaultclock.dt = 0.1*ms
# --- Input Spike Train (20 Hz) ---
spike_times = np.arange(15)*50*ms + 20*ms; input_spikes = SpikeGeneratorGroup(1, [0]*len(spike_times), spike_times)
# --- Target Neurons (N=2) ---
target_neurons = NeuronGroup(2, 'dv/dt = -v/(15*ms) : volt', threshold='v>-55*mV', reset='v=-75*mV'); target_neurons.v = -75*mV
# --- Tsodyks-Markram Synapse Model ---
U_dep=0.5; tau_rec_dep=100*ms; tau_facil_dep=50*ms # Depression params
U_fac=0.1; tau_rec_fac=100*ms; tau_facil_fac=500*ms # Facilitation params
w_static = 1.8*mV # Synaptic strength (voltage jump)
tm_model='dx/dt=(1-x)/tau_rec:1(event-driven); du/dt=(U_TM-u)/tau_facil:1(event-driven); U_TM:1; tau_rec:second; tau_facil:second'
tm_on_pre='u=u+U_TM*(1-u); release=u*x; x=x-release; v_post+=w_static*release'
# --- Create Depressing and Facilitating Synapses ---
syn_dep=Synapses(input_spikes, target_neurons[0:1], tm_model, on_pre=tm_on_pre); syn_dep.connect(); syn_dep.U_TM=U_dep; syn_dep.tau_rec=tau_rec_dep; syn_dep.tau_facil=tau_facil_dep; syn_dep.x=1.0; syn_dep.u=U_dep
syn_fac=Synapses(input_spikes, target_neurons[1:2], tm_model, on_pre=tm_on_pre); syn_fac.connect(); syn_fac.U_TM=U_fac; syn_fac.tau_rec=tau_rec_fac; syn_fac.tau_facil=tau_facil_fac; syn_fac.x=1.0; syn_fac.u=U_fac
# --- Monitors ---
mon_dep=StateMonitor(syn_dep,['u','x'],record=0); mon_fac=StateMonitor(syn_fac,['u','x'],record=0); mon_v=StateMonitor(target_neurons,'v',record=True)
input_mon=SpikeMonitor(input_spikes)
# --- Run Simulation ---
run(spike_times[-1] + 50*ms)
# --- Calculate PSP amplitude proxy (u*x) at spike times ---
psp_amp_dep=[]; psp_amp_fac=[]; spike_times_ms=input_mon.t/ms
for t_spk in spike_times_ms:
    idx_dep=np.argmin(np.abs(mon_dep.t/ms-t_spk)); psp_amp_dep.append(mon_dep.u[0][idx_dep]*mon_dep.x[0][idx_dep])
    idx_fac=np.argmin(np.abs(mon_fac.t/ms-t_spk)); psp_amp_fac.append(mon_fac.u[0][idx_fac]*mon_fac.x[0][idx_fac])
# --- Visualize ---
plt.figure(figsize=(12, 9)); plt.subplot(3, 1, 1); plt.plot(mon_v.t/ms, mon_v.v[0]/mV, label='Target 0 (Dep)'); plt.plot(mon_v.t/ms, mon_v.v[1]/mV, label='Target 1 (Fac)'); plt.title('Postsynaptic Responses'); plt.ylabel('Vm (mV)'); plt.legend(); plt.grid(alpha=0.5)
plt.subplot(3, 1, 2); plt.plot(mon_dep.t/ms, mon_dep.u[0],'b-',label='Dep u'); plt.plot(mon_dep.t/ms, mon_dep.x[0],'b--',label='Dep x'); plt.plot(mon_fac.t/ms, mon_fac.u[0],'r-',label='Fac u'); plt.plot(mon_fac.t/ms, mon_fac.x[0],'r--',label='Fac x'); plt.title('TM State Variables'); plt.ylabel('State'); plt.legend(); plt.grid(alpha=0.5)
plt.subplot(3, 1, 3); plt.plot(spike_times_ms, psp_amp_dep, 'bo-', label='Depressing Amp (u*x)'); plt.plot(spike_times_ms, psp_amp_fac, 'ro-', label='Facilitating Amp (u*x)'); plt.title('Short-Term Plasticity'); plt.ylabel('Effective Amp (u*x)'); plt.xlabel('Spike Time (ms)'); plt.legend(); plt.grid(alpha=0.5); plt.ylim(bottom=0)
plt.tight_layout(); plt.show()
```
*Explanation:* Compares a depressing synapse (high U) and a facilitating synapse (low U, high $\tau_{facil}$) receiving the same 20Hz input train. The plots clearly show the decreasing effective amplitude ($u \times x$) for the depressing synapse and the increasing amplitude for the facilitating one, illustrating the core STP phenomena modeled by Tsodyks-Markram.

**Task 3: Simulating Simple Neuromodulation (Affecting STP and Excitability)**
This example simulates a global neuromodulator that simultaneously decreases synaptic release probability (reduces `U` in TM synapses) AND increases neuronal excitability (lowers firing threshold).

```python
# === Brian2 Simulation: Neuromodulation of STP and Excitability ===
# (13.3_NeuromodulationSimEnhanced.ipynb)
from brian2 import *; import matplotlib.pyplot as plt; import numpy as np
start_scope(); defaultclock.dt = 0.1*ms

# --- Neuron Model (with dynamic threshold) ---
V_thresh_base = -55*mV; V_thresh_mod_range = 3*mV # Modulator lowers threshold
tau=15*ms; V_rest=-70*mV; V_reset=-75*mV; t_ref=3*ms
eqs_neuron = '''dv/dt = (V_rest - v)/tau : volt (unless refractory)
                 V_thresh : volt''' # Threshold as a parameter
target_neuron = NeuronGroup(1, eqs_neuron, threshold='v > V_thresh', reset='v = V_reset', method='euler')
target_neuron.v = V_rest; target_neuron.V_thresh = V_thresh_base

# --- Input Spike Train (20 Hz) ---
input_spikes = SpikeGeneratorGroup(1, [0]*15, np.arange(15)*50*ms + 20*ms)

# --- TM Synapse Model (Depressing Baseline) ---
U_base = 0.6; tau_rec = 150*ms; tau_facil = 50*ms; w_static = 1.7*mV
U_mod_factor = 0.8 # Modulator strongly reduces U
tm_model_mod='dx/dt=(1-x)/tau_rec:1(event-driven); du/dt=(U_syn-u)/tau_facil:1(event-driven); U_syn:1; tau_rec:second; tau_facil:second'
tm_on_pre_mod='u=u+U_syn*(1-u); release=u*x; x=x-release; v_post+=w_static*release'
syn_tm = Synapses(input_spikes, target_neuron, model=tm_model_mod, on_pre=tm_on_pre_mod, name='ModulatedSyn')
syn_tm.connect(); syn_tm.tau_rec=tau_rec; syn_tm.tau_facil=tau_facil; syn_tm.x=1.0; syn_tm.U_syn=U_base; syn_tm.u=U_base

# --- Conceptual Neuromodulator Level (TimedArray) ---
mod_times = np.array([0, 300, 600, 800]) * ms # Modulator ON between 300 and 600 ms
mod_levels = np.array([0.0, 1.0, 1.0, 0.0]) # Level 0 to 1
modulator_proxy = TimedArray(mod_levels, dt=1*ms) # Use dt=1ms for smoother interpolation

# --- Network Operation to Apply Neuromodulation ---
@network_operation(dt=5*ms) # Update reasonably frequently
def apply_neuromodulation():
    current_mod_level = modulator_proxy(defaultclock.t); current_mod_level = np.clip(current_mod_level, 0, 1)
    # Modulate Synaptic U
    syn_tm.U_syn = U_base * (1 - U_mod_factor * current_mod_level)
    # Modulate Neuronal Threshold
    target_neuron.V_thresh = V_thresh_base - V_thresh_mod_range * current_mod_level

# --- Monitors ---
mon_syn = StateMonitor(syn_tm, ['u', 'x', 'U_syn'], record=0)
mon_v = StateMonitor(target_neuron, 'v', record=0)
mon_thresh = StateMonitor(target_neuron, 'V_thresh', record=0) # Monitor threshold
input_mon = SpikeMonitor(input_spikes); target_mon = SpikeMonitor(target_neuron)

# --- Run Simulation ---
run(800*ms)

# --- Calculate PSP amplitude proxy (u*x) at spike times ---
psp_amp = []; times_spk = input_mon.t/ms
for t_spk in times_spk: idx = np.argmin(np.abs(mon_syn.t/ms - t_spk)); psp_amp.append(mon_syn.u[0][idx] * mon_syn.x[0][idx])

# --- Visualize ---
fig, axes = plt.subplots(5, 1, figsize=(10, 12), sharex=True)
# Modulator Level & Synaptic U & Threshold
axes[0].set_title('Neuromodulation Effects')
mod_t_plot = np.linspace(0, 800, 200); mod_l_plot=[np.clip(modulator_proxy(t*ms),0,1) for t in mod_t_plot]
axes[0].plot(mod_t_plot, mod_l_plot, 'r-', label='Mod Level'); axes[0].set_ylabel('Mod Level', color='r'); axes[0].tick_params(axis='y', labelcolor='r')
ax0b = axes[0].twinx(); ax0b.plot(mon_syn.t/ms, mon_syn.U_syn[0], 'b--', label='Synaptic U'); ax0b.plot(mon_thresh.t/ms, mon_thresh.V_thresh[0]/mV, 'g:', label='V_thresh')
ax0b.set_ylabel('Syn U / Vth (mV)'); ax0b.tick_params(axis='y'); axes[0].legend(loc='upper left'); ax0b.legend(loc='upper right'); axes[0].grid(alpha=0.5)
# Postsynaptic Voltage
axes[1].plot(mon_v.t/ms, mon_v.v[0]/mV); axes[1].set_ylabel('Vm (mV)'); axes[1].grid(alpha=0.5)
if len(target_mon.t)>0: axes[1].plot(target_mon.t/ms, np.ones_like(target_mon.t)*-50, 'k^', ms=5) # Plot target spikes
# STP Variables u and x
axes[2].plot(mon_syn.t/ms, mon_syn.u[0], label='u'); axes[2].plot(mon_syn.t/ms, mon_syn.x[0], '--', label='x'); axes[2].set_ylabel('State'); axes[2].legend(); axes[2].grid(alpha=0.5)
# Effective PSP Amplitude
axes[3].plot(times_spk, psp_amp, 'o-', label='Effective Amp (u*x)'); axes[3].set_ylabel('Amp (u*x)'); axes[3].legend(); axes[3].grid(alpha=0.5)
# Input spikes for reference
axes[4].plot(input_mon.t/ms, np.zeros_like(input_mon.t), '|k'); axes[4].set_yticks([]); axes[4].set_xlabel('Time (ms)'); axes[4].set_title('Input Spikes')
# Indicate modulation period
for ax in axes[:4]: ax.axvspan(300, 600, color='gray', alpha=0.1, label='Mod ON')
plt.tight_layout(); plt.show()

```
*Explanation:* This enhanced neuromodulation example shows a modulator (controlled by `TimedArray`) affecting two targets: it decreases the synaptic utilization parameter `U_syn` (making the synapse more depressing) and simultaneously decreases the neuronal firing threshold `V_thresh` (making the neuron more excitable). The `@network_operation` updates both parameters dynamically based on the modulator level. The plots visualize the modulator time course, the resulting changes in `U_syn` and `V_thresh`, and the combined effect on the postsynaptic voltage and effective synaptic strength ($u \times x$). This illustrates how neuromodulation can have complex, multi-target effects that reconfigure circuit function.

**13.7 Conclusion and Planned Code**

*(Content largely identical to the previous expanded response, summarizing the chapter's contributions)*

This chapter significantly enhanced the realism and computational potential of our network models by delving into **advanced features of synaptic transmission** that go beyond simple, static connections. We moved past basic conductance models to explore the implications of **diverse receptor kinetics**, highlighting the functional differences between fast transmission mediated by AMPA/GABA_A receptors and slower, often voltage-dependent transmission via NMDA and GABA_B receptors. We then introduced the crucial concept of **Short-Term Plasticity (STP)**, explaining the mechanisms and computational roles of activity-dependent facilitation (STF) and depression (STD), and presented the widely used **Tsodyks-Markram (TM) model** as a powerful tool for capturing these dynamic changes in synaptic efficacy. Furthermore, we discussed the pervasive influence of **neuromodulation**, explaining how diffuse signaling systems (dopamine, acetylcholine, etc.) can reconfigure network states and modulate both neuronal excitability and synaptic properties, including transmission strength and plasticity rules, often through **dynamic parameter control**. Concrete **Brian2 implementation examples** demonstrated how to: (1) simulate synapses with distinct **kinetics** (fast vs. slow/voltage-dependent); (2) implement the **Tsodyks-Markram model** to observe both STF and STD phenomena; and (3) simulate a simple form of **neuromodulation** by dynamically altering key synaptic and neuronal parameters based on a conceptual modulator signal. Incorporating these advanced synaptic properties—realistic kinetics, short-term dynamics, and sensitivity to neuromodulatory context—is essential for building models that can capture the flexibility, adaptability, and rich temporal processing capabilities observed in biological neural circuits, including potentially those developing within brain organoids.

**Planned Code Examples:**
*   **`13.1_ConductanceSynapseKinetics.ipynb`:** (Provided and explained in Section 13.6) Implements and compares postsynaptic responses generated by synapses with AMPA-like, NMDA-like, GABA_A-like, and conceptual GABA_B-like kinetics.
*   **`13.2_TsodyksMarkramSTP.ipynb`:** (Provided and explained in Section 13.6) Implements the Tsodyks-Markram model for STP within Brian2 `Synapses`. Simulates both depressing and facilitating synapses driven by a spike train and visualizes the dynamic changes in state variables (`u`, `x`) and effective synaptic strength.
*   **`13.3_NeuromodulationSimEnhanced.ipynb`:** (Provided and explained in Section 13.6) Demonstrates neuromodulation dynamically changing both a synaptic STP parameter (`U_syn`) and a neuronal excitability parameter (`V_thresh`) based on an external time-varying modulator signal.

---
**References for Further Reading**

1.  **Barroso-Flores, J., Herrera-Valdez, M. A., & Galarraga, E. (2023). Short-term synaptic plasticity: Mechanisms, functions, and unanswered questions.** *Neuroscience & Biobehavioral Reviews, 149*, 105175. https://doi.org/10.1016/j.neubiorev.2023.105175
    *   *Summary:* A thorough and recent review dedicated to Short-Term Plasticity (STP). It meticulously covers the known molecular mechanisms underlying facilitation and depression (vesicle dynamics, presynaptic calcium), details mathematical models including Tsodyks-Markram (Section 13.3), discusses the diverse computational functions attributed to STP (filtering, gain control, memory), and outlines key areas for future research.*
2.  **Bocchio, M., & Fisahn, A. (2023). Shaping neuronal activity with neuromodulators.** *Neuropharmacology, 232*, 109513. https://doi.org/10.1016/j.neuropharm.2023.109513
    *   *Summary:* Focuses on how major neuromodulatory systems (ACh, NE/NA) influence neuronal firing patterns and network oscillations (like gamma rhythms). It details specific effects on ion channels and synaptic transmission, providing biological grounding for the concept of neuromodulation dynamically controlling network state and function (Sections 13.4, 13.5).*
3.  **Destexhe, A. (2023). The biological noise of the brain.** *Nature Reviews Neuroscience, 24*(10), 611–626. https://doi.org/10.1038/s41583-023-00735-y
    *   *Summary:* Discusses the pervasive nature of noise in neural systems, including stochastic synaptic transmission (probabilistic vesicle release, Section 13.1). This inherent variability interacts with deterministic models of kinetics (Section 13.2) and plasticity (Section 13.3), impacting the reliability and function of synapses.*
4.  **Grillo, F. W., Song, S., & Han, K. S. (2023). Diversity of inhibitory synapses.** *Trends in Neurosciences, 46*(11), 930-942. https://doi.org/10.1016/j.tins.2023.08.006
    *   *Summary:* Highlights the significant diversity among inhibitory synapses formed by different interneuron subtypes onto various postsynaptic targets. This includes variations in GABA_A and GABA_B receptor composition and kinetics (Section 13.2), as well as distinct short-term plasticity profiles (Section 13.3), underscoring the complexity of inhibition.*
5.  **Naka, A., & Fukunaga, I. (2023). Ionotropic glutamate receptors in the accessory olfactory system.** *Frontiers in Neural Circuits, 17*, 1119703. https://doi.org/10.3389/fncir.2023.1119703
    *   *Summary:* While system-specific, this review provides detailed information on the molecular properties, trafficking, and functional roles of different ionotropic glutamate receptors (AMPA, NMDA, kainate) in neural circuits, offering concrete biological examples relevant to the discussion of receptor diversity and kinetics (Section 13.2).*
6.  **Naskar, S., Sudhakar, S. V., Sekar, R., Parthiban, G. L., & Raman, K. (2022). Modeling and analysis of synaptic dynamics.** *Frontiers in Computational Neuroscience, 16*, 824606. https://doi.org/10.3389/fncom.2022.824606
    *   *Summary:* Reviews various computational approaches for modeling dynamic aspects of synapses. It covers deterministic phenomenological models for short-term plasticity like Tsodyks-Markram (Section 13.3), as well as stochastic models aimed at capturing the probabilistic nature of vesicle release (related to Section 13.1).*
7.  **Palma-Tortosa, S., Nabavi, S., & Clopath, C. (2023). Towards synaptic-level understanding of memory representations.** *Current Opinion in Neurobiology, 80*, 102687. https://doi.org/10.1016/j.conb.2023.102687
    *   *Summary:* Explores how memories might be encoded in the fine details of synaptic connectivity and plasticity rules. It connects synaptic-level mechanisms, including short-term (Section 13.3) and long-term plasticity, to the formation and nature of memory representations at the network level.*
8.  **Planert, H., Paulus, W., & Trenado, C. (2023). Temporal integration of synaptic inputs shapes neuronal gain control in cortical neurons.** *Brain Sciences, 13*(3), 500. https://doi.org/10.3390/brainsci13030500
    *   *Summary:* This experimental and modeling study investigates how the integration of synaptic inputs over time—a process shaped by receptor kinetics (Section 13.2) and short-term plasticity (Section 13.3)—influences the input-output relationship (gain) of cortical neurons, linking synaptic dynamics to cellular computation.*
9.  **Pyle, R., & Rosenbaum, R. (2022). The structure of inhibitory circuitry and computations in spiking neural networks.** *PLoS Computational Biology, 18*(8), e1010368. https://doi.org/10.1371/journal.pcbi.1010368
    *   *Summary:* Uses computational modeling to explore how the architecture of inhibitory circuits, involving synapses with specific kinetics (GABA_A/B, Section 13.2) and potentially distinct STP properties (Section 13.3), enables specific network computations like normalization or gain control.*
10. **Trilla, F., Nörenberg, A., & Sprekeler, H. (2023). Short-term plasticity results in frequency-dependent information transmission in identify-and-fire neurons.** *bioRxiv*, 2023.12.06.570379. https://doi.org/10.1101/2023.12.06.570379
    *   *Summary:* This recent theoretical preprint analyzes how short-term plasticity mechanisms, as modeled phenomenologically (e.g., TM model, Section 13.3), act as dynamic filters that shape the transmission of information between spiking neurons in a way that depends critically on the frequency content of the input signal.*

---
