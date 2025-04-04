---

# Chapter 3

# Fundamentals of Computational Neuroscience and Single Neurons
-----

*This chapter marks a crucial transition from the biological substrate detailed in Chapter 2 towards the principles and tools required for computational modeling, specifically within the context of Organoid Computing. We lay the essential groundwork of computational neuroscience, establishing the framework for understanding how neural elements process information. We begin by discussing the various levels of abstraction at which neural systems can be modeled, exploring the trade-offs involved and framing the individual neuron as a fundamental, albeit simplified, computational unit while acknowledging other levels of processing. The chapter then introduces key mathematical models used to describe the electrical behavior of single neurons, starting with a more detailed conceptual overview of the biophysically rich Hodgkin-Huxley model, its experimental basis, and its significance. We then delve into mathematically simpler yet powerful abstractions designed for computational efficiency, with a particular focus on the derivation and application of the widely used Leaky Integrate-and-Fire (LIF) model. We will also introduce the equations and qualitative behavior of other influential simplified models like the Quadratic Integrate-and-Fire (QIF), Exponential Integrate-and-Fire (EIF), the Adaptive Exponential Integrate-and-Fire (AdEx) capable of adaptation and bursting, and the versatile Izhikevich model, highlighting their respective strengths, limitations, and suitability for different modeling goals, accompanied by conceptual Brian2 syntax examples. Following the discussion of single neuron dynamics, we will cover the basics of synaptic transmission, the mechanism by which neurons communicate, explaining the generation of excitatory and inhibitory postsynaptic potentials (EPSPs and IPSPs) resulting from neurotransmitter-receptor interactions, and contrasting the implementation and biophysical implications of current-based versus conductance-based synaptic models. We will also explore fundamental concepts in neural coding in greater depth, examining how information might be represented or encoded in the patterns of neuronal firing, including the assumptions and evidence for rate codes, temporal codes, and population codes. Critically, this chapter introduces the first practical, hands-on simulation example using the Brian2 simulator. We will demonstrate step-by-step how to implement and simulate a single LIF neuron receiving external input in Brian2, introducing core concepts like defining model equations with units, creating neuron groups, running simulations, monitoring neuron states (`StateMonitor`) and spikes (`SpikeMonitor`), and visualizing the results, thereby providing the foundational practical skills needed for the more complex network simulations presented in subsequent chapters.*

-----

**3.1 Levels of Modeling. The Neuron as a Computational Unit**

The brain, with its staggering complexity spanning molecules to behavior, presents an immense challenge to scientific understanding. Brain organoids, while simpler, still exhibit intricate multi-scale organization and dynamics. Computational neuroscience endeavors to make sense of this complexity by developing mathematical and computational models that capture essential aspects of neural function at different **levels of abstraction**. This hierarchical approach, famously echoed in David Marr's levels of analysis (computational theory, representation and algorithm, hardware implementation), recognizes that different scientific questions are best addressed at different granularities. Choosing the appropriate modeling level involves a crucial trade-off between biological realism, analytical tractability, and computational feasibility. At the most fundamental physical level lies **molecular modeling**. Here, the focus is on the atomistic or coarse-grained simulation of individual biomolecules crucial for neural function: the intricate conformational changes of voltage-gated ion channels as they open and close, the kinetics of neurotransmitter molecules binding to receptor proteins, the complex chain reactions within intracellular signaling cascades triggered by synaptic or hormonal signals, or the mechanics of vesicle fusion during neurotransmitter release. Models at this level provide the ultimate biophysical grounding, linking structure directly to function, but simulating even a single synapse or a small patch of membrane containing multiple channels at this detail is computationally extraordinarily expensive, limiting its application primarily to understanding specific molecular mechanisms in isolation rather than network behavior.

Moving up one significant step in scale, we reach **cellular modeling**, which focuses on the behavior of an entire single neuron (or sometimes a glial cell like an astrocyte) as an integrated unit. This level is central to much of computational neuroscience and to this chapter. Models at this level aim to describe how the neuron transforms its incoming synaptic inputs into an output signal, typically a sequence of action potentials (spikes). These models can vary greatly in their own internal complexity. Highly detailed **multi-compartment models**, often built using simulation environments like NEURON or GENESIS, represent the neuron's complex morphology by dividing its dendrites, soma, and axon into numerous small, interconnected compartments, each with its own membrane properties and potentially different densities of various ion channels. Such models allow for detailed investigation of dendritic integration, synaptic plasticity localization, and the influence of morphology on firing patterns. However, they are computationally intensive and require detailed anatomical and biophysical data that is often lacking, especially for neurons within developing organoids. At the simpler end of cellular modeling are **single-compartment** or **point-neuron models**, which treat the neuron as a single isopotential entity (or focus only on spike generation at the axon hillock), described by a small number of ordinary differential equations (ODEs). Models like Hodgkin-Huxley (Section 3.2) provide high biophysical detail within a single compartment, while simplified models like LIF, AdEx, or Izhikevich (Section 3.3) abstract away much of the ionic detail to focus on the core input-output transformation with greater computational efficiency.

Ascending further in scale, **network modeling** (often referred to as microcircuit modeling when focusing on local circuits) examines the collective dynamics and computational properties that emerge from the interactions of multiple, synaptically connected neurons. This is the level where phenomena directly relevant to information processing in circuits become apparent: synchronized oscillations in various frequency bands, propagation of activity patterns, attractor dynamics (stable states representing memories or decisions), information routing, and computations based on population activity. Models at this level typically utilize simplified neuron models (like LIF or AdEx) to make simulations of networks containing thousands or millions of neurons computationally feasible. Understanding how network structure (connectivity patterns) and synaptic properties interact with single-neuron dynamics to produce collective behavior is a primary goal of network modeling and central to the aims of this book in the context of organoid computation. Finally, at the highest level, **systems-level modeling** aims to understand the function of large-scale brain systems, involving interactions between multiple distinct brain areas (e.g., cortex, hippocampus, basal ganglia) to explain complex cognitive functions (like perception, decision-making, learning, language) or behavior. Models at this level often employ even more abstract representations, such as rate-based units representing the average activity of entire neural populations, or mathematical frameworks from control theory or machine learning, focusing on information flow and functional roles rather than detailed biophysical mechanisms.

`[Conceptual Figure 3.1: Levels of Modeling in Neuroscience. A hierarchical diagram showing different levels: Bottom - Molecular (ion channel dynamics, receptor binding). Middle-Low - Cellular (multi-compartment dendritic integration, single-compartment spike generation - LIF, HH). Middle-High - Network/Microcircuit (E/I populations, oscillations, synchrony). Top - Systems (interacting brain regions, cognitive function, behavior). Arrows indicate increasing spatial/temporal scale and abstraction upwards. Annotations mention trade-offs: Detail vs. Computational Cost.]`

Within this hierarchical framework, the **single neuron** has traditionally been elevated to the status of the fundamental **computational unit** of the nervous system. This influential perspective, dating back to McCulloch and Pitts' abstract model of a neuron as a threshold logic unit, provides a powerful conceptual lens for analyzing brain function. In this view, the neuron acts as a sophisticated information processing device: its extensive dendritic tree collects and integrates thousands of incoming signals (EPSPs and IPSPs) from other neurons; these signals are summed non-linearly over space and time; if the integrated signal depolarizes the membrane potential at the spike initiation zone (usually the axon initial segment) beyond a critical **threshold**, the neuron generates a stereotyped, "all-or-none" digital output pulse—the **action potential** or **spike**; this output spike then propagates along the axon to transmit information to downstream target neurons. This input-integration-thresholding-output scheme forms the basis for most spiking neural network models. However, it's crucial to acknowledge that this is a simplification. Significant computation also occurs *within* neurons, particularly through complex non-linear interactions in the dendrites (dendritic computation). Furthermore, computation is fundamentally a network phenomenon, emerging from the interactions between neurons, and glial cells also play active roles in modulating neuronal activity and synaptic plasticity. Nonetheless, understanding the mathematical rules governing the basic input-output transformation performed by the "average" neuron—how it converts its synaptic input patterns into output spike trains—remains a cornerstone of computational neuroscience. The various mathematical models described in this chapter (HH, LIF, etc.) represent different attempts to capture this transformation with varying degrees of biophysical detail and computational efficiency, providing the essential building blocks for constructing models of neural networks, including those inspired by the developing circuits within brain organoids.

**3.2 Biophysical Models (Hodgkin-Huxley - Overview)**

The groundbreaking work of Alan Hodgkin and Andrew Huxley in the late 1940s and early 1950s represents a watershed moment in neuroscience, marking the transition from qualitative description to quantitative, mechanistic understanding of neuronal excitability. Using the **voltage-clamp technique**—an ingenious experimental method allowing them to hold the membrane potential ($V_m$) of the squid giant axon at a fixed level while measuring the transmembrane currents—they systematically dissected the ionic currents responsible for the action potential. By selectively changing the ionic composition of the bathing solution (e.g., replacing sodium with an impermeable ion) or using pharmacological blockers (though these were less developed at the time), they identified distinct inward (depolarizing) and outward (repolarizing) currents with different time courses and voltage dependencies. They deduced that the early inward current was primarily carried by sodium ions ($Na^+$) and the later outward current primarily by potassium ions ($K^+$), in addition to a passive "leakage" current ($I_{\text{Leak}}$) carried by chloride and other ions.

Based on these meticulous measurements, Hodgkin and Huxley formulated a mathematical model—now universally known as the **Hodgkin-Huxley (HH) model**—that provided a quantitative description of these currents and their dependence on voltage and time. Their model brilliantly conceptualized the neuron membrane as an electrical circuit. The lipid bilayer acts as a capacitor ($C_m$), storing electrical charge. Embedded within this bilayer are pathways for ions to cross, represented as variable conductances ($g_{\text{ion}}$). Each major ion species ($Na^+$, $K^+$, Leak) has its own conductance pathway, associated with an equilibrium or **reversal potential** ($E_{\text{ion}}$) determined by the Nernst equation (e.g., $E_{Na} \approx +55mV$, $E_K \approx -75mV$, $E_{Leak} \approx -65mV$ in their preparation). The current carried by each ion species ($I_{\text{ion}}$) is then given by the product of its conductance and the driving force: $I_{\text{ion}} = g_{\text{ion}}(V_m, t) (V_m - E_{\text{ion}})$.

The true genius of the HH model lies in its description of how the conductances for sodium ($g_{Na}$) and potassium ($g_{K}$) change dynamically in response to changes in membrane voltage. They hypothesized that these conductances are controlled by hypothetical charged "gating particles" within the membrane that move in response to the electric field, opening or closing the ion channels. They proposed that the sodium channel requires three independent activation particles (which they called 'm' particles) to be in the permissive state and one inactivation particle ('h' particle) to be in the permissive state for the channel to conduct. The potassium channel required four activation particles ('n' particles) to be in the permissive state. The probability of each type of particle being in its permissive state ($m$, $h$, and $n$ respectively, ranging from 0 to 1) was modeled as obeying first-order kinetics, governed by voltage-dependent rate constants ($\alpha_x(V_m)$ for transitions to the permissive state and $\beta_x(V_m)$ for transitions away from it).

```latex
% Equation for gating variable dynamics
 \frac{dx}{dt} = \alpha_x(V_m) (1 - x) - \beta_x(V_m) x \quad \text{where } x \in \{m, h, n\}
 \tag{3.1a}
 ```
Hodgkin and Huxley empirically determined complex mathematical functions for these $\alpha$ and $\beta$ rate constants by fitting them to their voltage-clamp data. The total conductance for each ion was then expressed as the product of the maximum possible conductance ($\bar{g}_{\text{ion}}$) and the appropriate probabilities of the gates being open: $g_{Na} = \bar{g}_{Na} m^3 h$ and $g_{K} = \bar{g}_{K} n^4$. Finally, the rate of change of the membrane potential itself ($dV_m/dt$) was described by the membrane equation, based on Kirchhoff's current law stating that the capacitive current must balance the sum of all ionic currents and any externally applied current ($I_{\text{ext}}$):

```latex
\begin{align*}
 C_m \frac{dV_m}{dt} &= I_{\text{ext}} - I_{\text{Na}} - I_{\text{K}} - I_{\text{Leak}} \\
 &= I_{\text{ext}} - \bar{g}_{\text{Na}} m^3 h (V_m - E_{\text{Na}}) - \bar{g}_{\text{K}} n^4 (V_m - E_{\text{K}}) - g_{\text{Leak}} (V_m - E_{\text{Leak}})
 \tag{3.1b}
 \end{align*}
 ```
This complete system of four coupled ordinary differential equations (one for $V_m$, one for $m$, one for $h$, one for $n$) constitutes the full Hodgkin-Huxley model.

`[Conceptual Figure 3.2: Hodgkin-Huxley Membrane Circuit Diagram and Gating Variables. Panel (a): Electrical circuit diagram as before (Cm, gNa, gK, gLeak, ENa, EK, ELeak, Iext). Panel (b): Graphs showing the steady-state activation/inactivation curves (m_inf, h_inf, n_inf) and time constants (tau_m, tau_h, tau_n) as functions of membrane voltage Vm, illustrating the voltage-dependence derived by HH.]`

The HH model was a triumph of quantitative biology. When simulated numerically, it astonishingly reproduced virtually all the key features of the action potential observed in the squid giant axon: the sharp threshold for initiation, the rapid upstroke (driven by $Na^+$ influx through opening 'm' gates), the subsequent repolarization (driven by $Na^+$ channel inactivation via 'h' gates and delayed $K^+$ efflux through opening 'n' gates), the hyperpolarizing afterpotential, and the absolute and relative refractory periods. It provided a concrete, biophysically grounded mechanism for neuronal excitability based on the differential voltage- and time-dependent properties of sodium and potassium conductances. Although originally developed for the squid axon, the HH formalism proved remarkably general. By identifying and incorporating additional types of voltage-gated and calcium-gated ion channels discovered later in mammalian and other neurons (e.g., different subtypes of $K^+$ channels like A-type, M-type, SK/BK channels; various $Ca^{2+}$ channels like L-type, T-type; hyperpolarization-activated channels like $I_h$), researchers have created highly detailed and specific **biophysical models** for myriad neuron types across the brain. These models remain the gold standard for biological realism, allowing exploration of how specific ion channel properties ("channelopathies" in disease) or neuromodulatory effects on channels influence neuronal firing patterns and function.

However, this high fidelity comes at a steep price: **computational complexity**. Each HH-style neuron requires solving a system of multiple (often 5-20 or more) coupled, non-linear ODEs. Simulating large networks of such detailed neurons rapidly becomes computationally prohibitive, requiring significant supercomputing resources. Moreover, the **parameterization challenge** is immense. Accurately measuring the densities, kinetics ($\alpha$'s and $\beta$'s), and voltage dependencies of all relevant ion channels for a specific neuron type, especially within a complex and variable tissue like a brain organoid, is experimentally extremely demanding. Often, parameters are borrowed or estimated, introducing uncertainty. Therefore, while biophysical models like HH are invaluable for understanding fundamental mechanisms at the single-cell level, their direct application to large-scale network simulations aimed at uncovering emergent computational principles or modeling systems like organoids often necessitates the use of mathematically **simplified neuron models** that capture the essential spiking behavior more efficiently.

**3.3 Simplified Models: LIF (Detailed), QIF, EIF, AdEx, Izhikevich (Overview & Suitability)**

The significant computational burden and parameterization difficulties associated with detailed biophysical models like Hodgkin-Huxley spurred the development of a diverse family of **simplified neuron models**. The core philosophy behind these models is to abstract away the intricate details of individual ion channel dynamics while retaining the essential functional properties required for spiking network behavior: integration of inputs over time, a threshold mechanism for spike initiation, the generation of a spike event, and a subsequent refractory period. By reducing the mathematical complexity, typically down to just one or two differential equations per neuron, these models achieve substantial gains in computational efficiency, making it feasible to simulate networks containing millions of neurons or to run simulations over behaviorally relevant timescales. The trade-off, of course, is a loss of biophysical detail and potentially a reduced repertoire of dynamic behaviors compared to HH-style models. The art of computational neuroscience often lies in choosing the simplest model that still captures the phenomena relevant to the specific research question being addressed.

The archetypal and most widely employed simplified spiking model is the **Leaky Integrate-and-Fire (LIF) model**. Its origins trace back to the work of Louis Lapicque in the early 20th century. The LIF model conceptualizes the neuron's membrane as a simple electrical circuit consisting of a resistor ($R$) representing ion channels open at rest (the "leak") and a capacitor ($C$) representing the membrane's ability to store charge, driven by an input current $I(t)$. Applying Kirchhoff's current law ($I_{\text{total}} = I_C + I_R$) gives $I(t) = C \frac{dV}{dt} + \frac{V - V_{\text{rest}}}{R}$, where $V_{\text{rest}}$ is the voltage across the resistor when $I(t)=0$. Rearranging this yields the standard LIF differential equation for the subthreshold membrane potential $V$:

```latex
 C \frac{dV}{dt} = -(V - V_{\text{rest}})/R + I(t)
 \quad \text{or} \quad
 \tau \frac{dV}{dt} = -(V - V_{\text{rest}}) + R I(t)
 \tag{3.2}
 ```

Here, $\tau = RC$ is the crucial **membrane time constant**, which characterizes how quickly the membrane potential $V$ responds to changes in the input current $I(t)$. The term $-(V - V_{\text{rest}})$ represents the "leak" current driving the potential back towards rest, while $RI(t)$ represents the voltage change driven by the input current. The model "integrates" the input current $I(t)$ over time, filtered by the membrane leak. The second key component is the **firing mechanism**: if the membrane potential $V$ reaches or exceeds a predefined **threshold voltage** ($V_{\text{thresh}}$), the LIF model stipulates that a spike is generated. The shape of the spike itself is not explicitly modeled; its occurrence is simply registered as an event at that time. Immediately following the spike event, the membrane potential $V$ is instantaneously **reset** to a specific value $V_{\text{reset}}$ (usually equal to or slightly below $V_{\text{rest}}$), mimicking the repolarization and hyperpolarization phases after a real action potential. Additionally, the model typically includes an **absolute refractory period** ($t_{\text{ref}}$) after the reset, during which $V$ is clamped (or the integration is halted) preventing the neuron from firing again immediately, capturing the physiological inability of neurons to fire during the spike's falling phase. This combination of linear subthreshold integration, a sharp threshold, instantaneous reset, and refractoriness provides a remarkably effective, albeit highly simplified, caricature of a spiking neuron using just a single ODE, making the LIF model exceptionally computationally efficient.

`[Conceptual Figure 3.3: LIF Neuron Dynamics. Panel (a): RC circuit diagram for LIF (Resistor R and Capacitor C in parallel, driven by current I(t)). Panel (b): Graph showing membrane potential V(t) over time. Input current I(t) steps up. V(t) rises exponentially towards a steady state R*I(t) + V_rest with time constant tau. When V(t) crosses V_thresh, a spike is registered, V(t) is immediately reset to V_reset, and integration pauses for t_ref before resuming.]`

Despite its widespread use and efficiency, the standard LIF model possesses certain limitations regarding biological realism. One notable issue is the linearity of its subthreshold dynamics and the artificial sharpness of the firing threshold. Real neurons often exhibit non-linear integration effects in their dendrites and a smoother, more dynamic spike initiation process near threshold due to the rapid activation of sodium channels. Several alternative "integrate-and-fire" type models have been proposed to address these aspects while aiming to retain computational simplicity. The **Quadratic Integrate-and-Fire (QIF)** model, mathematically related to the normal form of a Type I neuron bifurcation (where firing rate can start from zero), introduces a quadratic term in the voltage equation: $\tau \frac{dV}{dt} = (V - V_{\text{rest}})(V - V_{\text{critical}}) + RI(t)$. This non-linearity causes the voltage to diverge towards infinity in finite time once it crosses a certain range, providing a more dynamically realistic (though still abstract) representation of spike initiation compared to the hard threshold of LIF. The **Exponential Integrate-and-Fire (EIF)** model offers perhaps a better balance, explicitly incorporating an exponential term that mimics the sharp activation of sodium channels near threshold:
```latex
% EIF Equation
 \tau \frac{dV}{dt} = -(V - V_{\text{rest}}) + \Delta_T \exp\left(\frac{V - V_T}{\Delta_T}\right) + RI(t)
 \tag{3.3a}
 ```
Here, $V_T$ acts as an effective threshold, and the slope factor $\Delta_T$ controls how sharply the voltage increases as it approaches $V_T$. The EIF model provides a good fit to the spike initiation dynamics observed in cortical neurons with relatively little added computational cost over LIF.

A major limitation of LIF, QIF, and EIF models is their inability to intrinsically capture **spike frequency adaptation** or **bursting** behaviors, which are common properties of many real neurons mediated by slower ion currents. To address this, models incorporating a second dynamic variable were developed. The **Adaptive Exponential Integrate-and-Fire (AdEx)** model builds upon the EIF framework by adding a second differential equation for an adaptation variable $w$, which typically represents a slow adaptation current (like a $Ca^{2+}$-activated $K^+$ current or $Na^+$ channel inactivation):
```latex
% AdEx Equations
\begin{align*}
 \tau \frac{dV}{dt} &= -(V - V_{\text{rest}}) + \Delta_T \exp\left(\frac{V - V_T}{\Delta_T}\right) - w + RI(t) \\
 \tau_w \frac{dw}{dt} &= a(V - V_{\text{rest}}) - w
 \tag{3.3b}
 \end{align*}
 ```
The variable $w$ integrates the subthreshold membrane potential $V$ (controlled by parameter $a$) and decays with time constant $\tau_w$. Crucially, $w$ acts as a hyperpolarizing current in the voltage equation. Additionally, the reset mechanism for AdEx often includes an increment to $w$ upon spiking ($w \rightarrow w + b$), further enhancing adaptation. By tuning the parameters $a$ (subthreshold adaptation coupling), $\tau_w$ (adaptation time constant), and $b$ (spike-triggered adaptation increment), the AdEx model can reproduce a variety of firing patterns including regular spiking, strong adaptation, initial bursting, and regular bursting, all with just two ODEs.

Another highly influential two-variable model is the **Izhikevich model**, developed by Eugene Izhikevich specifically for computational efficiency combined with dynamic richness. It uses a mathematically clever formulation:
```latex
% Izhikevich Equations
\begin{align*}
 \frac{dV}{dt} &= 0.04 V^2 + 5 V + 140 - u + I(t) \\
 \frac{du}{dt} &= a(bV - u)
 \tag{3.3c}
 \end{align*}
 ```
with an auxiliary after-spike resetting rule: if $V \ge +30 \text{ mV}$, then $V \leftarrow c$ and $u \leftarrow u + d$. (Note: Units are often implicitly mV, ms, nA in the original formulation). The variable $u$ represents a recovery variable providing negative feedback. By adjusting just the four dimensionless parameters $a, b, c, d$, this simple-looking system can generate a remarkable diversity of firing patterns observed across different neuronal types (e.g., regular spiking, intrinsically bursting, chattering, fast spiking, low-threshold spiking, resonator). Its computational efficiency is comparable to AdEx, making it very popular for large-scale simulations where diverse neuron types need to be included.

`[Conceptual Table 3.1: Comparison of Simplified Neuron Models. Rows: LIF, QIF, EIF, AdEx, Izhikevich. Columns: No. of Variables (1, 1, 1, 2, 2), Key Equation Feature (Linear, Quadratic, Exponential, Exp.+Adaptation, Quadratic+Recovery), Spike Initiation Realism (Low, Medium, Good, Good, Medium), Adaptation/Bursting (No, No, No, Yes, Yes), Parameter Count (Few, Few, Few, Moderate, Few(4)), Firing Pattern Diversity (Low, Low, Low, Moderate, High), Computational Cost (Very Low, Low, Low, Medium, Medium).]`

The **suitability** of each model depends heavily on the specific goals and constraints of the simulation. For exploring fundamental network phenomena in very large systems where computational speed is paramount and detailed single-neuron dynamics are less critical, **LIF** remains a valuable workhorse. If accurate spike initiation shape is important (e.g., for certain STDP studies), **EIF** offers a good improvement with minimal overhead. If the biological system being modeled (e.g., specific neurons in an organoid culture) exhibits significant adaptation or intrinsic bursting, and these features are hypothesized to be functionally relevant, then **AdEx** or **Izhikevich** models become necessary choices, despite the added computational cost and parameterization challenges. Brian2's equation-oriented syntax makes defining these models relatively straightforward. Here are conceptual snippets of how their core equations might look in Brian2 `model` strings:

```python
# Conceptual Brian2 syntax for different models (parameters assumed defined)

# EIF
eif_eqs = '''
dv/dt = (-(v - V_rest) + DeltaT*exp((v - VT)/DeltaT) + R*I)/tau : volt (unless refractory)
I : amp
'''

# AdEx
adex_eqs = '''
dv/dt = (-(v - V_rest) + DeltaT*exp((v - VT)/DeltaT) - w + R*I)/tau : volt (unless refractory)
dw/dt = (a*(v - V_rest) - w)/tau_w : amp
I : amp
'''
# AdEx reset would be: 'v = Vr; w += b'

# Izhikevich (using typical dimensionless units for V, u, I)
# Need to define parameters a, b, c, d and handle units carefully if mixing
izh_eqs = '''
dv/dt = (0.04*v**2 + 5*v + 140 - u + I)/ms : 1 # Voltage 'v' is dimensionless here
du/dt = a*(b*v - u)/ms : 1 # Recovery 'u' is dimensionless
I : 1 # Input current 'I' also needs consistent units
a : 1; b : 1 # Parameters a, b
'''
# Izhikevich reset: 'v = c; u += d' (where c, d are parameters)
```
Throughout this book, we will primarily utilize the LIF model for introducing concepts related to network connectivity, dynamics, and basic computation due to its simplicity. However, we will explicitly return to models like AdEx in Chapter 11 when discussing advanced neuron modeling and their potential relevance for capturing richer dynamics observed in organoid systems.

**3.4 Basic Synaptic Transmission (EPSPs, IPSPs, Current vs. Conductance Models)**

The ability of neurons to form networks and process information hinges entirely on their capacity to communicate with each other through specialized junctions known as **synapses**. In the vast majority of synapses in the mammalian central nervous system, communication is chemical: the arrival of an action potential at the axon terminal of the **presynaptic** neuron triggers a complex cascade of events leading to the release of **neurotransmitters** into the **synaptic cleft**, a narrow gap (typically 20-40 nanometers) separating it from the **postsynaptic** neuron. These released neurotransmitter molecules rapidly diffuse across the cleft and bind to specific **neurotransmitter receptors** embedded in the postsynaptic membrane. This binding event is the key step that transduces the chemical signal back into an electrical or biochemical response in the postsynaptic neuron, thereby transmitting information across the synapse.

Neurotransmitters and their receptors come in many varieties, leading to diverse synaptic effects. However, a fundamental distinction is between **excitatory** and **inhibitory** transmission. **Glutamate** is the principal excitatory neurotransmitter in the CNS. When released, it typically binds to ionotropic receptors like **AMPA receptors** and **NMDA receptors** on the postsynaptic membrane. Activation of AMPA receptors leads to the rapid opening of channels permeable primarily to $Na^+$ (and some $K^+$), resulting in a net influx of positive charge and a rapid depolarization of the postsynaptic membrane. This transient depolarization is termed an **Excitatory Postsynaptic Potential (EPSP)**. NMDA receptors have more complex properties: they are also permeable to $Ca^{2+}$ (making them important for plasticity) and their channel is blocked by $Mg^{2+}$ at resting membrane potentials, requiring prior depolarization (often provided by AMPA receptor activation) to become fully active. Multiple EPSPs arriving close together in time (**temporal summation**) or at nearby locations on the dendrite (**spatial summation**) can add up. If the summed depolarization reaches the neuron's firing threshold at the axon initial segment, it triggers a postsynaptic action potential.

Conversely, **gamma-aminobutyric acid (GABA)** is the primary inhibitory neurotransmitter in the brain. GABA typically binds to **GABA$_A$ receptors** (ionotropic) or **GABA$_B$ receptors** (metabotropic). GABA$_A$ receptors are ligand-gated ion channels primarily permeable to chloride ions ($Cl^-$). The effect of opening GABA$_A$ channels depends on the relationship between the postsynaptic membrane potential ($V_m$) and the reversal potential for chloride ($E_{Cl}$), which itself depends on the intracellular chloride concentration maintained by chloride transporters. In many mature neurons, $E_{Cl}$ is close to or slightly below the resting potential ($V_{\text{rest}}$). In this case, GABA$_A$ activation leads to $Cl^-$ influx, causing a small hyperpolarization or stabilization of $V_m$ near $V_{\text{rest}}$. This transient potential change is called an **Inhibitory Postsynaptic Potential (IPSP)**. Even if the IPSP doesn't cause significant hyperpolarization (i.e., if $E_{Cl} \approx V_{\text{rest}}$), the opening of chloride channels increases the membrane conductance, leading to **shunting inhibition**, which makes it harder for concurrent EPSPs to depolarize the neuron to threshold. GABA$_B$ receptors are G protein-coupled receptors that typically activate inwardly rectifying potassium ($K^+$) channels, leading to $K^+$ efflux and a slower, longer-lasting hyperpolarizing IPSP (as $E_K$ is usually very negative, around -80 to -90 mV). The constant interplay between incoming streams of EPSPs and IPSPs dynamically shapes the neuron's membrane potential trajectory and ultimately determines its output spiking pattern.

`[Conceptual Figure 3.4: EPSP and IPSP. Panel (a): Schematic of an excitatory synapse (glutamate -> AMPA/NMDA R) leading to Na+/Ca2+ influx and an EPSP (depolarizing voltage trace). Panel (b): Schematic of an inhibitory synapse (GABA -> GABA_A R) leading to Cl- influx and an IPSP (hyperpolarizing or shunting voltage trace). Panel (c): Schematic showing temporal and spatial summation of EPSPs leading to threshold crossing.]`

To incorporate these crucial interactions into computational models, we need mathematical descriptions of synaptic transmission. As introduced briefly before, two main formalisms are commonly used: **current-based** and **conductance-based** models.

The **current-based synapse model** offers simplicity and computational speed. It approximates the effect of neurotransmitter binding as a direct injection of a synaptic current pulse, $I_{\text{syn}}(t)$, into the postsynaptic neuron. The shape of this current pulse is often modeled as an instantaneous jump followed by an exponential decay, or using an alpha function ($I_{\text{syn}}(t) \propto t \exp(-t/\tau_{\text{syn}})$) which has a smoother rise and fall. The amplitude (or total charge) of this current pulse is determined by the synaptic weight, $w$. This $I_{\text{syn}}(t)$ is then simply added to the other currents in the postsynaptic neuron's membrane equation:
```latex
% LIF with current-based synapse
 \tau \frac{dV}{dt} = -(V - V_{\text{rest}}) + R (I_{\text{ext}}(t) + I_{\text{syn}}(t))
```
In Brian2 `on_pre` syntax, this often translates to directly incrementing a postsynaptic current variable: `on_pre='I_syn_post += w'`, where `w` has units of current. The decay dynamics of `I_syn_post` are then handled within the `NeuronGroup`'s equations. The primary drawback of this model is its biophysical inaccuracy: it completely neglects the dependence of synaptic current on the postsynaptic membrane potential ($V_m$) and the driving force ($V_m - E_{\text{rev}}$). This can lead to unrealistic behavior, such as EPSPs causing arbitrarily large depolarizations even when $V_m$ is already near the excitatory reversal potential.

The **conductance-based synapse model** provides a more biophysically faithful description by explicitly modeling the change in membrane **conductance** ($g_{\text{syn}}(t)$) caused by the opening of postsynaptic ion channels. The synaptic current is then calculated dynamically at each time step using Ohm's law with the appropriate driving force:
```latex
% Conductance-based synaptic current
 I_{\text{syn}}(t) = g_{\text{syn}}(t) (V(t) - E_{\text{rev}})
 \tag{3.4}
```
Here, $g_{\text{syn}}(t)$ is the time-varying synaptic conductance (e.g., often modeled as $g_{\text{syn}}(t) \propto \exp(-t/\tau_{\text{syn}})$ or an alpha/bi-exponential function triggered by the presynaptic spike), $V(t)$ is the instantaneous postsynaptic membrane potential, and $E_{\text{rev}}$ is the specific reversal potential for the type of synapse ($E_E \approx 0$ mV for excitation, $E_I \approx -75$ mV for GABA$_A$ inhibition). In Brian2, this usually involves defining conductance variables (`g_E`, `g_I`) and their decay dynamics in the `NeuronGroup`, and using `on_pre` in the `Synapses` object to increment the appropriate conductance by a weight `w` (which now has units of conductance, e.g., Siemens): `on_pre='g_E_post += w'`. The membrane equation in the `NeuronGroup` must then include the conductance terms:
```latex
% LIF with conductance-based synapses
 C_m \frac{dV}{dt} = (V_{\text{rest}} - V)/R + g_E(t)(E_E - V) + g_I(t)(E_I - V) + I_{\text{ext}}
```
This model correctly captures the saturation of synaptic currents as $V$ approaches $E_{\text{rev}}$ and accurately represents shunting inhibition. Because of its greater realism, especially in modeling the interplay of excitation and inhibition in recurrent networks operating in fluctuating regimes (like the high-conductance state often seen *in vivo* and potentially in active organoids), the conductance-based formalism is generally preferred for detailed network modeling, despite being slightly more computationally demanding than the current-based approach. Understanding the concepts of reversal potentials and driving forces is crucial for correctly implementing and interpreting conductance-based synaptic models.

**3.5 Neural Coding (Rate, Temporal, Population)**

Beyond understanding how individual neurons fire and communicate, a central challenge in neuroscience is deciphering the **neural code**: the set of rules or conventions by which the nervous system represents information about the external world, internal states, motor commands, or abstract concepts using the electrical activity of neurons. How are features like the intensity of a light, the identity of an object, the direction of movement, or the memory of an event encoded in the patterns of action potentials (spikes) generated by neural populations? Elucidating the neural code is fundamental not only for understanding brain function but also for developing effective strategies to interface with neural systems, whether for therapeutic purposes (e.g., brain-computer interfaces) or for potentially harnessing biological computation, as envisioned in Organoid Computing. If we aim to interpret the activity of an organoid network or input information into it meaningfully, we need hypotheses about how information might be encoded in its spiking patterns. Several major coding schemes have been proposed, and it's likely that the brain employs a flexible combination depending on the specific context, brain area, and timescale involved.

The historically dominant and perhaps most intuitive hypothesis is **rate coding**. This scheme proposes that information is primarily encoded in the **average number of spikes** a neuron fires within a given time window, i.e., its **firing rate**. A neuron might respond to a preferred stimulus (e.g., a vertical bar of light for a visual neuron, a specific frequency for an auditory neuron) with a high firing rate, and to non-preferred stimuli with a low firing rate, or even silence. The intensity of a stimulus is often encoded by modulating this firing rate—a stronger stimulus evokes a higher rate. This relationship between stimulus intensity and firing rate is often characterized by the neuron's **tuning curve** or **F-I curve** (Frequency-Input curve). Rate coding assumes that the precise timing of individual spikes within the measurement window is largely irrelevant or averages out, with the mean rate being the critical variable carrying information. Evidence supporting rate coding comes from numerous sensory and motor systems where firing rates clearly correlate with stimulus parameters or muscle forces. Computationally, rate codes are appealing due to their conceptual simplicity and relative robustness to noise (averaging over time smooths out random fluctuations). However, a significant potential drawback is the inherent **slowness** of rate coding; accurately estimating a firing rate requires observing spikes over a potentially long time window (e.g., tens or hundreds of milliseconds), which seems inconsistent with the brain's ability to perform rapid computations and react quickly to changing environments.

```python
# Conceptual Python snippet to calculate average firing rate from spike times

import numpy as np

# Assume spike_times_ms is a list or array of spike times in milliseconds
# Assume duration_ms is the total observation duration in milliseconds

# Example data
spike_times_ms = np.array([15.2, 35.8, 51.3, 78.0, 95.1])
duration_ms = 100.0

# Calculate firing rate in Hz (spikes per second)
num_spikes = len(spike_times_ms)
duration_s = duration_ms / 1000.0
if duration_s > 0:
    firing_rate_hz = num_spikes / duration_s
else:
    firing_rate_hz = 0.0

# print(f"Average firing rate: {firing_rate_hz:.2f} Hz")
# Output: Average firing rate: 50.00 Hz
```

Challenging the rate-coding dogma, the concept of **temporal coding** emphasizes the potential information carried by the **precise timing** of individual spikes, or by the fine temporal structure within spike patterns (like bursts or pauses), often considered relative to an external event (like stimulus onset) or internal network dynamics (like ongoing oscillations). In this view, *when* a spike occurs carries information, not just *how many* spikes occur. Compelling evidence for temporal coding exists in specific systems, such as the remarkably precise timing (sub-millisecond) used by neurons in the auditory brainstem to encode sound localization based on interaural time differences. Other proposed temporal codes involve information encoded in the **first spike latency** after a stimulus, the relative timing of spikes across different neurons (**synchrony** or **asynchrony**), specific **sequences** of spikes, or the phase relationship between spikes and background network oscillations (e.g., theta or gamma rhythms). Temporal codes offer the potential for much **higher information capacity** (as each spike's timing adds information) and **faster processing** (information can potentially be conveyed by just one or a few precisely timed spikes) compared to rate codes. However, implementing and reliably decoding precise spike timing in the face of biological noise (synaptic jitter, channel stochasticity) presents significant challenges, and the metabolic cost of maintaining such precision might be high. The extent to which precise temporal codes are used broadly across the cortex remains an active area of investigation and debate.

Neither rate nor temporal coding considers individual neurons in isolation. **Population coding** recognizes that information in the brain is typically represented not by single neurons, but by the **distributed pattern of activity across large populations** of neurons. Individual neurons often exhibit broad tuning curves, responding to a range of stimuli around their preferred stimulus, and their responses can be variable (noisy). However, by combining the simultaneous activity of many such broadly tuned and noisy neurons, the nervous system can achieve remarkably precise and reliable representations. A classic example is the encoding of arm movement direction in the motor cortex. Individual neurons fire most vigorously for movements in their specific "preferred direction," but also fire at lower rates for nearby directions. The actual intended direction of movement can be accurately predicted by calculating a **population vector**: a weighted average of the preferred directions of all active neurons, where the weight for each neuron is determined by its current firing rate relative to its maximum rate. Population coding offers several advantages: **robustness** (the representation degrades gracefully if some neurons fail), **high fidelity** (averaging across many neurons reduces noise), and the ability to represent **high-dimensional information** (different combinations of neuronal activity can represent different points in a multi-dimensional feature space). Many complex brain functions, from sensory perception (e.g., representing object identity by the pattern of activity across visual cortex neurons) to decision-making and motor control, are thought to rely heavily on population-level representations.

`[Conceptual Figure 3.5: Neural Coding Schemes. Panel (a) Rate Coding: Shows spike trains for weak vs. strong stimulus, differing in spike density. Panel (b) Temporal Coding: Shows spike trains with similar rates but different precise timings (e.g., relative to stimulus onset or an oscillation). Panel (c) Population Coding: Shows a population of neurons with tuning curves (e.g., for direction). For a given stimulus direction, the pattern of firing rates across the population encodes the direction, illustrated conceptually with a population vector.]`

It is increasingly apparent that these coding schemes are not mutually exclusive. The brain likely employs a **multiplexed code**, utilizing different strategies in parallel or dynamically switching between them depending on the task demands and behavioral context. Information might be encoded simultaneously in the firing rates of neurons, their precise spike timing relative to network oscillations, and the pattern of activity across the population. Understanding this complex interplay is crucial for interpreting neural data, including the potentially rich but complex activity patterns generated by brain organoids. When simulating organoid models, we need to consider which types of coding might be plausible given the model's components (neuron types, synapse types, connectivity) and how we might design simulations or analyses to probe these potential coding strategies.

**3.6 Brian2 Implementation: Simulating a Single LIF Neuron**

Having established the theoretical foundations of single neuron modeling and neural coding, we now embark on our first practical implementation using the **Brian2 simulator**. This example serves as a hands-on tutorial, demonstrating the fundamental workflow for defining, simulating, and analyzing a simple neuronal model in Brian2. We will focus on the **Leaky Integrate-and-Fire (LIF) neuron**, chosen for its simplicity and widespread use, making it an ideal starting point. Our objective is to:
1.  Define the parameters and differential equation for the LIF model using Brian2's syntax, critically incorporating physical units.
2.  Create a `NeuronGroup` representing a single instance of this LIF neuron.
3.  Apply an external stimulus in the form of a constant step current injected into the neuron.
4.  Use `StateMonitor` and `SpikeMonitor` to record the neuron's membrane potential trajectory and spike times during the simulation.
5.  Execute the simulation using the `run()` command.
6.  Visualize the results by plotting the membrane potential over time and indicating the spike events.

This concrete example will illustrate key Brian2 concepts and provide the foundational code structure upon which we will build more complex simulations in subsequent chapters. We will follow the logical steps outlined in the previous version's description but provide the code first, followed by a detailed line-by-line explanation.

```python
# === Brian2 Simulation: Single LIF Neuron ===
# (3.1_SingleLIFNeuron.ipynb)

# Import necessary Brian2 library components and plotting tools
from brian2 import *
#%matplotlib inline # Use this in Jupyter notebooks for inline plots
import matplotlib.pyplot as plt

# Use start_scope() at the beginning of scripts or notebooks using Brian2
# to clear any Brian2 objects created in previous runs.
start_scope()

# --- 1. Define Model Parameters with Units ---
# Using Brian2's unit system (ms, mV, Mohm, nA, pA, nS etc.) is crucial
# for ensuring physical consistency and preventing errors.
tau = 10*ms        # Membrane time constant (e.g., 10 milliseconds)
V_rest = -70*mV    # Resting membrane potential (e.g., -70 millivolts)
V_thresh = -50*mV  # Spike threshold (e.g., -50 millivolts)
V_reset = -75*mV   # Reset potential after a spike (e.g., -75 millivolts)
R = 100*Mohm       # Membrane resistance (e.g., 100 Megaohms)
# Membrane capacitance C can be calculated C = tau/R, or defined independently
# C = 100*pF
# tau = R*C # Check consistency if defining C
t_ref = 2*ms       # Absolute refractory period (e.g., 2 milliseconds)

# Define the input current stimulus
input_current = 0.25*nA # Constant current applied during part of the simulation

# --- 2. Define Model Equations ---
# The model is defined using a multi-line string.
# Equations are written in standard mathematical notation.
# State variables (like 'v') are defined by their differential equations.
# Parameters used in equations (like 'V_rest', 'tau', 'R', 'I_inj') must be
# defined either globally (as above) or as per-neuron parameters.
# Units are specified after a colon for state variables and parameters.
# The '(unless refractory)' flag prevents integration during the refractory period.

lif_eqs = '''
dv/dt = (V_rest - v + R*I_inj)/tau : volt (unless refractory)
I_inj : amp (constant) # Define I_inj as a constant parameter (can be changed externally)
'''

# --- 3. Create Neuron Group ---
# Create a NeuronGroup containing N=1 neuron.
# Arguments: number of neurons, model equations string, threshold condition string,
# reset statement string, refractory period duration, integration method.
neuron = NeuronGroup(N=1,
                     model=lif_eqs,
                     threshold='v > V_thresh',
                     reset='v = V_reset',
                     refractory=t_ref,
                     method='euler', # Using simple Euler method for integration
                     name='LIF_Neuron_0') # Assigning a name is good practice

# --- 4. Set Initial Conditions and Parameters ---
# Initialize the state variable 'v' (membrane potential) for the neuron.
neuron.v = V_rest
# Set the value for the parameter 'I_inj' (initially zero).
neuron.I_inj = 0*nA

# --- 5. Define Monitors ---
# StateMonitor records the values of specified state variables over time.
# Arguments: source group, variable(s) to record (list or string), indices to record from.
# 'record=0' means record from the neuron at index 0.
state_monitor = StateMonitor(neuron, 'v', record=0, name='Voltage_Monitor')

# SpikeMonitor records the times and indices of spikes from a group.
# Arguments: source group.
spike_monitor = SpikeMonitor(neuron, name='Spike_Monitor')

# --- 6. Run Simulation in Stages ---
# It's often useful to run the simulation in segments, especially when changing stimuli.
# store() saves the current network state, allowing restore() later if needed.
store()

# Run for an initial period with no input current
print(f"Running initial phase (no current)...")
run(20*ms)

# Apply the step current stimulus
print(f"Applying input current: {input_current}...")
neuron.I_inj = input_current
# Run simulation while the current is on
run(80*ms)

# Turn off the current
print(f"Turning off current...")
neuron.I_inj = 0*nA
# Run for a final period to see relaxation
run(50*ms)
print("Simulation finished.")

# --- 7. Analyze and Visualize Results ---
print("Plotting results...")
plt.figure(figsize=(12, 5)) # Create a figure

# Plot the membrane potential trace recorded by the StateMonitor
# Access time points: state_monitor.t
# Access voltage values for the first recorded neuron (index 0): state_monitor.v[0]
# Divide by units (ms, mV) for plotting numerical values on axes
plt.plot(state_monitor.t/ms, state_monitor.v[0]/mV, label='Membrane Potential (Vm)', color='navy')

# Add a horizontal line indicating the firing threshold
plt.axhline(V_thresh/mV, ls='--', color='red', lw=1.5, label='Threshold (V_thresh)')

# Add a horizontal line indicating the resting potential
plt.axhline(V_rest/mV, ls=':', color='gray', lw=1, label='Resting Potential (V_rest)')

# Indicate spike times recorded by the SpikeMonitor on the voltage plot
# Access spike times: spike_monitor.t
# Ensure we only plot spikes that occurred during the monitored period
valid_spike_times = spike_monitor.t[(spike_monitor.t >= state_monitor.t[0]) & (spike_monitor.t <= state_monitor.t[-1])]
if len(valid_spike_times) > 0:
    # Plot markers slightly above threshold for visibility
    plt.plot(valid_spike_times/ms, np.ones(len(valid_spike_times)) * (V_thresh/mV + 5), 'r^', markersize=8, label='Spikes')

# Add labels, title, legend, and grid
plt.xlabel('Time (ms)')
plt.ylabel('Membrane Potential (mV)')
plt.title('Simulation of a Single Leaky Integrate-and-Fire (LIF) Neuron')
plt.legend(loc='lower right')
plt.grid(True, linestyle='--', alpha=0.6)
plt.ylim(V_reset/mV - 10, V_thresh/mV + 15) # Adjust y-axis limits for better view

plt.tight_layout() # Adjust layout
plt.show() # Display the plot

# Optional: Print summary information
total_spikes = spike_monitor.num_spikes
print(f"\nSimulation Summary:")
print(f" - Total duration: {(state_monitor.t[-1] - state_monitor.t[0])/ms:.1f} ms")
print(f" - Input current applied: {input_current} from 20 ms to 100 ms")
print(f" - Total number of spikes: {total_spikes}")
if total_spikes > 0:
    avg_firing_rate_during_stim = total_spikes / (80*ms) # Calculate rate during 80ms stimulus
    print(f" - Average firing rate during stimulus: {avg_firing_rate_during_stim:.2f}")
    print(f" - Spike times (ms): {np.round(spike_monitor.t/ms, 2)}")

```

**Explanation of the Code (`3.1_SingleLIFNeuron.ipynb`):**

1.  **Import and Scope:** We import the necessary Brian2 functions (`from brian2 import *`) and Matplotlib for plotting. `start_scope()` is used to ensure a clean environment for Brian2 objects, especially important when running code repeatedly in notebooks or scripts.
2.  **Parameters:** All model parameters are defined explicitly with their physical **units** using Brian2's built-in unit system (`ms`, `mV`, `Mohm`, `nA`, `pA`, etc.). This is a key feature of Brian2 that enhances code clarity and helps prevent unit conversion errors. We define parameters for the LIF dynamics (`tau`, `V_rest`, `V_thresh`, `V_reset`, `R`, `t_ref`) and the external stimulus (`input_current`).
3.  **Model Equations:** The core dynamics of the LIF neuron are defined in the `lif_eqs` multi-line string. It contains the differential equation for the membrane potential `v` (following Equation 3.2). The equation specifies the unit of the state variable (`volt`). The flag `(unless refractory)` tells Brian2 to pause this integration during the refractory period. We also declare `I_inj` as a constant parameter of type `amp` within the model; marking it `(constant)` signifies that its value is the same for all neurons (only one here) but can be changed externally between `run()` calls.
4.  **NeuronGroup Creation:** A `NeuronGroup` named `neuron` is instantiated to represent our single (`N=1`) LIF neuron. We pass the `model` equations string, the `threshold` condition (`'v > V_thresh'`) which triggers a spike when true, the `reset` statement (`'v = V_reset'`) which sets the voltage back after a spike, the duration of the `refractory` period, and the numerical `method` for solving the ODE (`'euler'`, a simple first-order method). Assigning a `name` helps identify objects in more complex simulations.
5.  **Initial Conditions/Parameters:** Before starting the simulation, we set the initial state of the neuron. `neuron.v = V_rest` sets the membrane potential to the resting potential. `neuron.I_inj = 0*nA` sets the initial value of the injected current parameter to zero.
6.  **Monitors:** We set up monitors to record the simulation output. `StateMonitor(neuron, 'v', record=0)` creates a monitor that will record the value of the state variable `'v'` from the neuron at index `0` at each time step. `SpikeMonitor(neuron)` creates a monitor that will record the time `t` and index `i` (which will always be 0 here) whenever the neuron fires a spike. Names are assigned for clarity.
7.  **Simulation Execution:** The simulation is run in three distinct stages using the `run(duration)` command. This allows us to apply the stimulus partway through. `store()` saves the initial state before the first run. We run for 20 ms with `I_inj = 0*nA`. Then, we change the neuron's parameter `neuron.I_inj = input_current` and run for 80 ms. Finally, we set `neuron.I_inj = 0*nA` again and run for 50 ms. Brian2 handles the numerical integration based on the defined equations and the specified `dt` (time step, determined automatically or set manually).
8.  **Analysis and Visualization:** After the simulation finishes, the recorded data is available in the monitor objects. `state_monitor.t` holds the array of time points, and `state_monitor.v[0]` holds the array of corresponding voltage values for the recorded neuron. `spike_monitor.t` holds the array of spike times. We use Matplotlib to create a plot of voltage vs. time. We divide the monitor data by the appropriate Brian2 units (`ms`, `mV`) to get dimensionless numbers suitable for plotting. Horizontal lines are added for `V_thresh` and `V_rest` for visual reference. Spike times from `spike_monitor` are plotted as markers (red triangles '^') on the voltage trace slightly above threshold for visibility. Labels, title, legend, and grid enhance readability. `plt.show()` displays the generated figure. Finally, optional print statements output summary statistics like the total number of spikes and their precise timings.

Executing this code will generate a plot vividly illustrating the characteristic behavior of an LIF neuron: integrating the input current (causing the voltage ramp), firing action potentials upon reaching threshold, resetting, exhibiting refractoriness, and relaxing back to rest when the stimulus ceases. This fundamental example demonstrates the core workflow of Brian2—defining models with units, creating objects, running simulations, and accessing/visualizing results—providing the essential practical skills needed to tackle the more complex network simulations presented in the following chapters.

**3.7 Conclusion and Planned Code**

This chapter has laid the crucial theoretical and practical groundwork for the computational modeling aspects of this book, focusing on the fundamental unit of neural computation: the single neuron. We established the context by discussing the **levels of modeling** in neuroscience, acknowledging the trade-offs between detail and computational feasibility, and framing the neuron as a primary, albeit simplified, processing element. We reviewed the principles and historical significance of the biophysically detailed **Hodgkin-Huxley model** before concentrating on the derivation, utility, and limitations of **simplified spiking models**. The **Leaky Integrate-and-Fire (LIF)** model was examined in detail, while other important models like **QIF, EIF, AdEx, and Izhikevich** were introduced with their core equations and dynamic capabilities (adaptation, bursting), along with conceptual Brian2 syntax. The fundamental mechanisms of **synaptic transmission** were explained, covering **EPSPs and IPSPs**, the role of key neurotransmitters and receptors, and the crucial distinction between **current-based** and **conductance-based** synaptic models, highlighting the importance of driving forces and reversal potentials for the latter. We also explored the primary hypotheses regarding the **neural code**, discussing how information might be represented through **rate, temporal, and population** activity patterns. Critically, this chapter provided the first concrete, hands-on introduction to simulation using the **Brian2** environment. Through the detailed implementation and explanation of simulating a single LIF neuron (`3.1_SingleLIFNeuron.ipynb`), we illustrated the core workflow: defining models with physical **units**, creating `NeuronGroup` objects, setting parameters and initial conditions, utilizing `StateMonitor` and `SpikeMonitor` for data recording, executing simulations with `run()`, and visualizing results with Matplotlib. The conceptual understanding and practical Brian2 skills acquired in this chapter are indispensable prerequisites for the subsequent chapters, where we will leverage these foundations to construct and analyze interconnected neural networks (Chapter 4), incorporate biological realism like heterogeneity and spontaneous activity (Chapter 5), introduce synaptic plasticity (Chapter 6), and ultimately explore the computational potential of these organoid-inspired network models.

**Planned Code Example:**
*   **`3.1_SingleLIFNeuron.ipynb`:** (Provided and explained in Section 3.6) Simulates a single Leaky Integrate-and-Fire neuron receiving a step current input, demonstrating basic Brian2 usage, model definition with units, monitoring state variables and spikes, and plotting membrane potential trace with spike indication. (Jupyter Notebook format `.ipynb` recommended for interactive execution and visualization).

**3.8 References for Further Reading (APA Format)**

1.  Abbott, L. F. (1999). Lapicque's introduction of the integrate-and-fire model neuron (1907). *Brain Research Bulletin, 50*(5-6), 303–304. https://doi.org/10.1016/s0361-9230(99)00161-6 *(Provides valuable historical context on the origins of the integrate-and-fire concept, tracing it back to Lapicque.)*
2.  Dayan, P., & Abbott, L. F. (2001). *Theoretical neuroscience: Computational and mathematical modeling of neural systems*. MIT Press. *(A comprehensive and highly influential textbook covering a vast range of topics in computational neuroscience, including detailed treatments of single neuron models, synaptic plasticity, network dynamics, and neural coding.)*
3.  Gerstner, W., Kistler, W. M., Naud, R., & Paninski, L. (2014). *Neuronal dynamics: From single neurons to networks and models of cognition*. Cambridge University Press. *(Another excellent and modern textbook, offering rigorous mathematical descriptions and analysis of various neuron models (LIF, GLM, AdEx, HH), synaptic mechanisms, plasticity rules, and network behaviors, including code examples.)*
4.  Hodgkin, A. L., & Huxley, A. F. (1952). A quantitative description of membrane current and its application to conduction and excitation in nerve. *The Journal of Physiology, 117*(4), 500–544. https://doi.org/10.1113/jphysiol.1952.sp004764 *(The original, groundbreaking paper describing the Hodgkin-Huxley model. Essential reading for understanding the foundations of biophysical modeling.)*
5.  Izhikevich, E. M. (2003). Simple model of spiking neurons. *IEEE Transactions on Neural Networks, 14*(6), 1569–1572. https://doi.org/10.1109/TNN.2003.820440 *(The concise paper introducing the computationally efficient Izhikevich model and demonstrating its ability to reproduce diverse firing patterns.)*
6.  Izhikevich, E. M. (2004). Which model to use for cortical spiking neurons? *IEEE Transactions on Neural Networks, 15*(5), 1063–1070. https://doi.org/10.1109/TNN.2004.832719 *(Provides a practical comparison of different simplified neuron models (LIF, QIF, Izhikevich, etc.) discussing their trade-offs in capturing cortical neuron dynamics versus computational cost.)*
7.  Koch, C. (1999). *Biophysics of computation: Information processing in single neurons*. Oxford University Press. *(A classic and detailed text focusing on how the biophysical properties of dendrites and single neurons contribute to information processing, complementing the point-neuron focus of many simplified models.)*
8.  Brette, R., & Gerstner, W. (2005). Adaptive exponential integrate-and-fire model as an effective description of neuronal activity. *Journal of Neurophysiology, 94*(5), 3637–3642. https://doi.org/10.1152/jn.00686.2005 *(The paper introducing the AdEx model, detailing its formulation and its ability to capture adaptation and bursting with only two variables.)*
9.  Rieke, F., Warland, D., de Ruyter van Steveninck, R., & Bialek, W. (1997). *Spikes: Exploring the neural code*. MIT Press. *(A seminal book applying principles from information theory to analyze neural spike trains, offering deep insights into the concepts and challenges of understanding the neural code.)*
10. Stimberg, M., Brette, R., & Goodman, D. F. M. (2019). Brian 2, an intuitive and efficient neural simulator. *eLife, 8*, e47314. https://doi.org/10.7554/eLife.47314 *(The primary reference paper for the Brian2 simulator. Essential for understanding its design philosophy, features (like the unit system and equation-oriented approach), and capabilities, as used throughout this book.)*
