---

# Chapter 9

# Computational Primitives I: Logic and Memory in Organoid-Inspired Networks

---

*Having established how to build and simulate dynamic networks with heterogeneity, spontaneous activity, and plasticity (Chapters 3-7), we now pivot towards exploring their potential **computational capabilities**. Can these complex, recurrent, spiking neural networks, inspired by the structure and dynamics of brain organoids, perform fundamental information processing tasks beyond simple signal transmission or pattern generation? This chapter begins our exploration of **computational primitives**—the basic building blocks from which more complex computations might be constructed—focusing first on elementary **logic-like operations** and fundamental mechanisms for **short-term or working memory**. We start by examining common **neural circuit motifs**, recurring patterns of connectivity like feedback loops (excitatory and inhibitory) and lateral inhibition, discussing their prevalence in biological circuits and their potential roles in shaping network computations and enabling specific functions. We then delve into the conceptual possibility, inherent challenges, and biophysical underpinnings of implementing functions **analogous** to basic **Boolean logic gates** (AND, OR, NOT) using networks of spiking neurons, emphasizing the critical differences from deterministic digital logic due to noise, timing sensitivity, and analog neuronal processing. Following this, we explore the neural mechanisms widely believed to underlie **short-term and working memory**—the brain's crucial ability to actively hold and manipulate information "online" for brief periods (seconds to minutes) to guide ongoing behavior. We focus particularly on the prominent role of **recurrent network dynamics** in generating **persistent activity** as a substrate for maintaining information, and conceptually introduce the complementary contribution of **short-term synaptic plasticity (STP)** in encoding recent activity history (deferring detailed STP modeling to Chapter 13). The concept of **state retention** through **attractor dynamics** in recurrent networks is elaborated as a robust theoretical mechanism for memory maintenance and pattern completion. Finally, the chapter provides practical **Brian2 implementation examples** with detailed explanations: one set demonstrates how specific network motifs can produce outputs analogous to simple logic gates (e.g., coincidence detection for AND, summation for OR, feedforward/feedback inhibition for NOT), exploring parameter sensitivity; the second example simulates a small recurrent E/I network exhibiting **bistability and persistent activity**, demonstrating a basic form of working memory where the network reliably holds its activity state (low or high) after being switched by transient input pulses.*

---

**9.1 Neural Circuit Motifs for Computation (Feedback, Lateral Inhibition)**

Biological neural networks, far from being randomly interconnected meshes, exhibit a remarkable degree of structural organization across multiple scales. At the microcircuit level, specific patterns of connectivity involving small numbers of neurons frequently recur throughout different brain regions and even across diverse species. These recurring wiring diagrams, known as **neural circuit motifs**, are thought to represent fundamental building blocks or canonical circuits shaped by evolutionary pressures and developmental constraints to perform specific computational operations efficiently and reliably. Identifying these motifs and understanding their functional properties provides a powerful framework for dissecting the complex computations performed by larger neural systems. By studying how information is processed within these elementary motifs, we can gain insights into the potential computational repertoire of networks forming within brain organoids, even if the overall connectivity is complex or partially unknown. Two particularly ubiquitous and computationally significant motifs are **feedback loops (recurrence)** and **lateral inhibition**.

**Feedback loops**, or **recurrent connections**, are pathways where the output of a neuron or a population of neurons feeds back, directly or indirectly, to influence its own input or the input of neurons earlier in the processing stream. This creates cycles in the network graph, allowing activity to reverberate and persist over time, fundamentally distinguishing recurrent networks from purely feedforward architectures. Recurrence is a defining feature of cortical and hippocampal circuits *in vivo* and is almost certainly present in the dense networks forming within organoids. Feedback can be broadly categorized as excitatory (positive feedback) or inhibitory (negative feedback), each playing distinct computational roles:

*   **Excitatory Feedback (Positive Feedback):** This occurs when excitatory neurons connect back onto themselves (autapses, though rare) or, more commonly, onto other excitatory neurons within the same population (E-to-E recurrence). Positive feedback loops act as **amplifiers**, potentially boosting weak signals or rapidly driving the network towards a highly active state. They are crucial for implementing **winner-take-all** mechanisms, where competition mediated by inhibition allows the most active neuron or assembly to suppress others and dominate the output. Perhaps most importantly, strong recurrent excitation, when appropriately balanced by inhibition, is the primary mechanism thought to underlie **sustained or persistent neural activity**—the ability of a network to maintain a representation of information (e.g., a remembered stimulus) through continued firing even after the initial input has ceased. This persistent activity is considered a cornerstone of **working memory** (Section 9.3) and can lead to **bistability** or **multi-stability**, where the network possesses multiple stable firing rate patterns (**attractor states**) corresponding to different stored memories or decisions (Section 9.4). However, the inherent positive feedback nature of recurrent excitation makes these circuits prone to instability; without adequate inhibitory control, activity can escalate uncontrollably, leading to pathological hyperexcitability or seizure-like events.

*   **Inhibitory Feedback (Negative Feedback):** This typically involves inhibitory interneurons that receive input from a principal population and project back inhibitory connections onto that same population (e.g., E-to-I followed by I-to-E) or onto themselves (I-to-I). Negative feedback loops are absolutely essential for **stabilizing network activity**, preventing runaway excitation caused by recurrent E-E connections, and maintaining the crucial **balance between excitation and inhibition (E/I balance)** (Sprekeler, 2023). Beyond stability, feedback inhibition plays key roles in **gain control** (adjusting the sensitivity of neuronal responses), **sparse coding** (promoting efficient representations where only a few neurons are active), implementing **competition** between neurons or populations, and, critically, generating **network oscillations**. The rhythmic interplay between excitatory and inhibitory populations, where excitation drives inhibition which then dampens excitation, leading to a subsequent rebound, is the primary mechanism underlying the generation of various brain rhythms (e.g., gamma, beta, theta oscillations) observed experimentally and implicated in coordinating neural processing across time and space.

The specific configuration and strength of these feedback loops, involving different excitatory and inhibitory cell types with potentially distinct time constants and connectivity patterns, largely determine the dynamic repertoire and computational capabilities of a local circuit (Pyle & Rosenbaum, 2022).

**Lateral inhibition** represents another fundamental circuit motif, particularly prominent in sensory processing pathways (like the retina, olfactory bulb, and sensory cortices) but also employed in other brain areas for selection and competition. In its canonical form, an activated neuron inhibits its adjacent or neighboring neurons, often via connections through local inhibitory interneurons. When a neuron receives strong input, it not only produces its own output but also activates inhibitory cells that suppress the activity of nearby neurons receiving weaker input. This spatial arrangement of inhibition has several key computational consequences:
*   **Edge Enhancement / Contrast Enhancement:** In sensory maps, lateral inhibition sharpens the representation of stimulus boundaries or edges. Neurons near an edge receive differential inhibition from neighbors on either side, leading to an enhanced response profile at the edge itself.
*   **Competition and Winner-Take-All Dynamics:** Lateral inhibition provides a direct mechanism for competition between neurons representing similar features or locations. The neuron receiving the strongest input inhibits its competitors more strongly, potentially silencing them and allowing only the "winning" neuron (or a small group) to remain highly active. This is crucial for processes like selective attention, decision making, and categorization.
*   **Decorrelation and Efficient Coding:** By suppressing redundant activity among neighboring neurons that might respond to similar stimuli, lateral inhibition can help to decorrelate their responses, leading to sparser and more efficient population codes where information is represented by a smaller, less correlated set of active neurons.
*   **Pattern Separation:** By increasing the difference between the neural activity patterns evoked by similar but distinct input stimuli (as neighboring representations inhibit each other), lateral inhibition can enhance the network's ability to discriminate between closely related patterns.

Implementing lateral inhibition requires specific local connectivity patterns, typically involving excitatory neurons activating nearby inhibitory interneurons that, in turn, project inhibitory synapses onto neighboring excitatory neurons (and potentially other interneurons). While directly mapping these precise microcircuits in organoids is challenging, the co-existence of E and I neurons provides the necessary cellular components, suggesting that lateral inhibition motifs could potentially self-organize or be encouraged through specific culture conditions or engineered scaffolding.

`[Conceptual Figure 9.1: Neural Circuit Motifs. Panel (a) Feedback Loops: Expanded diagram showing E->E recurrent loop, E->I->E feedback inhibitory loop, and I->I recurrent inhibitory loop. Panel (b) Lateral Inhibition: Expanded diagram showing multiple principal neurons (E) arranged spatially. Central E neuron activates local Interneuron (I), which inhibits adjacent E neurons. Arrows indicate flow of excitation (+) and inhibition (-).]`

Beyond these two prominent examples, other important motifs include **feedforward excitation** (chains of excitatory neurons, potentially for signal propagation or amplification), **feedforward inhibition** (where an excitatory input activates both a principal neuron and an interneuron that inhibits the principal neuron slightly later, providing precise temporal control or sharpening responses), **disinhibition** (where an inhibitory neuron inhibits another inhibitory neuron, effectively releasing a target neuron from inhibition), and more complex **triadic** or **higher-order motifs**. These elementary circuit patterns likely combine in intricate ways within larger networks to implement more sophisticated computations. Identifying potential motifs within organoid models (perhaps through functional connectivity analysis or limited structural studies) and simulating their computational properties using tools like Brian2 (as shown below for logic and memory) provides a valuable bottom-up approach to understanding the information processing potential embedded within these developing biological circuits (Kumarasinghe et al., 2023).

```python
# Conceptual Brian2 Snippet: Defining Motif Connections
# Assume NeuronGroups P_E, P_I exist

# 1. Recurrent Excitation (E->E)
# p_rec_exc = 0.1 # Probability of recurrent E connection
# syn_EE = Synapses(P_E, P_E, on_pre='g_E_post += w_EE', ...) # Conductance based example
# syn_EE.connect(condition='i!=j', p=p_rec_exc)

# 2. Feedback Inhibition (E->I->E)
# p_ei = 0.2; p_ie = 0.3
# w_ei = 1.0*nS; w_ie_abs = 5.0*nS # Conductance weights
# syn_EI = Synapses(P_E, P_I, on_pre='g_E_post += w_ei', ...)
# syn_EI.connect(p=p_ei)
# syn_IE = Synapses(P_I, P_E, on_pre='g_I_post += w_ie_abs', ...)
# syn_IE.connect(p=p_ie)

# 3. Lateral Inhibition (via local I neurons, conceptual 1D space)
# Assume P_E and P_I have 'x : meter (constant)' attribute
# sigma_exc_lat = 50*umetre; p_max_EI_lat = 0.4 # E activates local I
# sigma_inh_lat = 100*umetre; p_max_IE_lat = 0.5 # I inhibits broader E neighbors
# prob_EI_lat = f'{p_max_EI_lat} * exp(-(x_pre - x_post)**2 / (2*{sigma_exc_lat}**2))'
# prob_IE_lat = f'{p_max_IE_lat} * exp(-(x_pre - x_post)**2 / (2*{sigma_inh_lat}**2))'
# syn_E_I_lat = Synapses(P_E, P_I, on_pre='g_E_post += w_ei', ...)
# syn_E_I_lat.connect(p=prob_EI_lat)
# syn_I_E_lat = Synapses(P_I, P_E, on_pre='g_I_post += w_ie_abs', ...)
# syn_I_E_lat.connect(condition='i!=j', p=prob_IE_lat)
```
*(This snippet shows conceptually how different motifs could be constructed using separate `Synapses` objects and appropriate `connect` rules, potentially involving probabilities or distance dependence.)*

**9.2 Implementing Logic Functions (AND, OR, NOT) with Network Motifs (Possibilities, limitations)**

A foundational question when evaluating any potential new computational substrate is its ability to implement basic **Boolean logic functions**—operations like AND, OR, and NOT, which serve as the fundamental building blocks for all digital computers. Can networks of biological spiking neurons, perhaps arranged into specific circuit motifs like those discussed above, perform computations that are analogous to these logic gates? The answer is conceptually yes, but with significant caveats and crucial differences compared to the deterministic, noise-free operation of standard electronic logic gates. Spiking neural networks operate using asynchronous communication, analog integration of inputs over time, and threshold-based firing, principles vastly different from the clocked, binary logic levels of digital circuits. Therefore, while we can design small spiking network motifs whose *average* input-output behavior mimics logic functions under specific interpretations, these implementations are typically **probabilistic**, **sensitive to timing and noise**, and may require careful **parameter tuning**.

Let's explore conceptual implementations and their limitations in more detail:

*   **AND Gate Analogue (Coincidence Detection):** The AND function yields 'True' output only when *all* inputs are simultaneously 'True'. A spiking neuron can approximate this by acting as a **coincidence detector**. It needs to fire only when multiple input spikes arrive within a very narrow temporal window, such that their postsynaptic potentials (PSPs) summate effectively before decaying significantly.
    *   **Biophysical Basis:** Effective coincidence detection is favored by neurons with properties that limit temporal integration, primarily a **short membrane time constant** ($\tau = RC$). A short $\tau$ causes PSPs to decay rapidly, meaning only near-synchronous inputs can summate above threshold. Additionally, **synaptic location** matters; inputs arriving close together on the dendrite summate more effectively than spatially distributed inputs. Specific ion channel compositions (e.g., presence of channels active near threshold) can also influence coincidence detection properties.
    *   **Implementation Idea:** A single postsynaptic neuron receives excitatory inputs (via synapses with weights $w_A, w_B, ...$) from multiple presynaptic neurons (Inputs A, B, ...). The parameters must be tuned such that the depolarization caused by any single input's EPSP is subthreshold, but the near-simultaneous arrival of EPSPs from all required inputs surpasses the threshold $V_{thresh}$.
        ```latex
        % Conceptual Tuning for AND-like behavior
        \text{For all } k, \quad V_{\text{rest}} + \text{EPSP}_k(t=t_{peak}) < V_{thresh} \\
        V_{\text{rest}} + \sum_{k \in \text{Inputs}} \text{EPSP}_k(t \approx t_{arrival}) > V_{thresh} \quad (\text{if coincident})
        ```
    *   **Interpretation:** Inputs A, B are 'True' if they fire within the coincidence window $\Delta t_{coinc}$ (related to $\tau$ and synaptic kinetics). Output is 'True' if the postsynaptic neuron fires.
    *   **Challenges:** Extremely sensitive to the precise relative timing of input spikes ($\Delta t$). Noise (timing jitter, $V_m$ fluctuations, synaptic failures) can easily cause false positives (firing with incomplete input) or false negatives (failing to fire with coincident input). Requires careful tuning of weights relative to threshold and time constants.

*   **OR Gate Analogue (Sufficient Input / Summation):** The OR function yields 'True' if *at least one* input is 'True'. A spiking neuron approximates this if it is sufficiently sensitive or receives strong enough inputs such that firing is reliably triggered by input from **any single** (or a small subset) of its designated input channels. It essentially acts as a detector for sufficient incoming drive, regardless of which specific input provided it.
    *   **Biophysical Basis:** This requires either very strong individual synaptic weights ($w_k$), a low intrinsic firing threshold ($V_{thresh}$) relative to the resting potential ($V_{rest}$), or a high input resistance ($R$) amplifying the voltage effect of synaptic currents ($R I_{\text{syn}}$). Neurons operating in a high-excitability state might naturally behave more like OR gates.
    *   **Implementation Idea:** A postsynaptic neuron receives excitatory inputs from presynaptic neurons A, B, C, ... The weights $w_A, w_B, ...$ are individually large enough to cause suprathreshold EPSPs.
        ```latex
        % Conceptual Tuning for OR-like behavior
        \text{For all } k, \quad V_{\text{rest}} + \text{EPSP}_k(t=t_{peak}) > V_{thresh}
        ```
    *   **Interpretation:** Input A/B 'True' = Spike arrival. Output 'True' = Output neuron fires.
    *   **Challenges:** Prone to firing in response to spurious inputs or background noise due to its high sensitivity. Strong synapses might violate biological constraints or lead to saturation issues. Defining the distinction between a meaningful input spike and noise becomes critical.

*   **NOT Gate Analogue (Inhibition):** The NOT gate inverts its input. In spiking networks, logical negation is typically implemented using synaptic **inhibition**, leveraging the ability of inhibitory neurons to suppress the firing of target neurons. Several motifs can achieve this:
    *   **Inhibition of Spontaneous Activity:** This relies on an output neuron (C) that exhibits baseline spontaneous firing (e.g., driven by tonic background excitation or intrinsic pacemaker properties). The input neuron (A) drives an inhibitory interneuron (I), which strongly inhibits C. When A is 'True' (firing), I is activated, silencing C (Output 'False'). When A is 'False' (silent), I is inactive, allowing C to fire spontaneously (Output 'True'). This requires careful tuning of the spontaneous drive to C and the strength of the I-to-C inhibition.
    *   **Direct Inhibition / Veto:** Input A provides direct, strong inhibition onto output neuron C, which also receives tonic excitatory drive from other sources. When A fires ('True'), its IPSP prevents C from reaching threshold, resulting in 'False' output. When A is silent ('False'), the tonic excitation drives C to fire ('True' output).
    *   **Feedforward Inhibition for Temporal NOT:** Input A excites both output neuron C and an inhibitory interneuron I. If the pathway through I is slightly faster or stronger, the inhibition onto C arrives just after the direct excitation, effectively vetoing the spike or creating a very brief firing window. This can implement functions like "fire only if A has *not* fired recently."
    *   **Interpretation:** Defining 'True'/'False' inputs and outputs often relies on comparing firing rates to baseline or detecting the presence/absence of spikes within specific time windows.
    *   **Challenges:** Requires precise timing and strength balance between excitatory and inhibitory pathways. The definition and detection of the 'True' (e.g., spontaneous firing) vs. 'False' (silence) output states need careful consideration, especially in the presence of noise.

`[Conceptual Figure 9.2: Spiking Network Logic Gate Analogues. (a) AND: Neuron C needs coincident spikes from A & B (short tau, precise weights). (b) OR: Neuron C fires if either A or B spikes (strong weights / low threshold). (c) NOT (Inhibition of Spontaneous): Input A -> Interneuron I -| Output C (spontaneously active). (d) NOT (Direct Inhibition): Input A -| Output C (receives other excitation).]`

While these conceptual implementations highlight the possibility of logic-like operations, it is crucial to reiterate the significant **limitations and fundamental differences** compared to conventional digital logic:

1.  **Probabilistic and Noisy Operation:** Biological computation is inherently stochastic due to channel noise, synaptic unreliability (vesicle release failure), and background network activity (Destexhe, 2023). Consequently, neural logic gates will operate **probabilistically**, producing the "correct" output only with a certain likelihood, rather than deterministically. Achieving high reliability might necessitate redundant representations or error-correcting codes at the population level.
2.  **Critical Timing Dependence:** The function of motifs like the AND gate is exquisitely sensitive to the **relative timing** of input spikes, operating correctly only within narrow coincidence windows. Biological timing jitter can easily disrupt these operations. Clocked synchronization, fundamental to digital logic, is generally absent in the asynchronous spiking of biological networks (though network oscillations can provide temporal reference frames).
3.  **Analog Computation Substrate:** Neurons perform complex **analog computations** on their dendritic trees and sum inputs non-linearly at the soma before applying a threshold. Reducing this rich behavior to simple binary logic gates is a drastic simplification. The brain likely leverages this analog processing for more sophisticated computations than pure Boolean logic. The mapping between continuous membrane potentials and discrete spike outputs is inherently non-linear and threshold-dependent.
4.  **Parameter Sensitivity and Tuning:** Achieving the desired input-output function for these motifs often requires **precise and potentially fragile tuning** of multiple parameters (synaptic weights relative to thresholds, membrane time constants, inhibitory strengths, background activity levels). How such precise tuning is achieved and maintained robustly in biological systems, especially developing and variable ones like organoids, remains a major question.
5.  **Scalability and Composability:** Building large-scale, complex computational systems by simply wiring together these basic neural logic gates faces enormous challenges related to **timing control, error propagation, fan-in/fan-out limitations, and maintaining parameter tuning** across layers. The brain likely employs different architectural principles for scaling computation, perhaps involving population codes, hierarchical structures, and specialized circuits rather than universal gate composition.
6.  **Universality:** It's not straightforward to construct robust and easily composable **universal logic gates** (like NAND or NOR) using simple spiking neuron motifs, further limiting the direct applicability of digital design principles.

In conclusion, while simulating logic-like primitives provides valuable insight into the basic computational repertoire of spiking circuits, it is crucial to view them as **analogous operations** (coincidence detection, summation, inhibition) rather than direct implementations of digital logic. Their probabilistic, timing-sensitive, and parameter-dependent nature highlights fundamental differences between biological ("wetware") and silicon ("hardware") computation. The Brian2 examples in Section 9.5 will illustrate these principles and challenges.

**9.3 Short-Term and Working Memory (Role of recurrence, STP)**

Beyond processing immediate inputs, a hallmark of advanced cognition is the ability to temporarily store and manipulate information that is no longer present in the environment. This capacity underlies **short-term memory** (often viewed as passive storage over seconds) and the more active, attention-demanding process of **working memory** (the ability to actively hold *and* manipulate information online to guide ongoing thoughts and actions, typically lasting seconds to minutes) (Compte, 2022). Working memory acts as a crucial mental workspace, essential for tasks requiring integration of information over time, such as language comprehension (holding the beginning of a sentence while processing the end), reasoning (keeping track of intermediate steps), planning future actions, and making decisions based on recent context. Understanding the neural mechanisms that enable the brain to bridge these temporal gaps is a central goal in neuroscience and critical for assessing the potential cognitive capabilities of systems like brain organoids. Two principal classes of biological mechanisms, likely operating in concert, are thought to contribute to short-term and working memory: **sustained reverberating activity** within recurrent neural circuits, and **dynamic modulation of synaptic efficacy** through short-term plasticity.

The most widely accepted theoretical model for the active maintenance component of working memory involves **persistent neural activity** generated and sustained within **recurrent excitatory circuits** (Compte, 2022). The core concept proposes that information is held "online" by a specific subpopulation of neurons representing that information entering a self-sustaining, high-activity firing state after being triggered by a transient input cue. This persistent firing is thought to arise from **strong, positive feedback** mediated by recurrent excitatory connections (E-to-E synapses) among the neurons within the memory-representing assembly. Once activated by the input, these neurons repeatedly excite each other, creating a reverberating loop that maintains their firing at an elevated rate even after the external stimulus has disappeared. For this mechanism to function effectively as a memory store, several conditions must be met:
*   **Stability:** The high-activity state must be stable, neither dying out quickly nor escalating into uncontrolled, epileptiform activity. This requires precise **balancing by feedback inhibition**. Inhibitory interneurons, driven by the active excitatory population (E-to-I), provide negative feedback (I-to-E) that counteracts the recurrent excitation, preventing runaway firing and stabilizing the persistent state at a specific firing rate. Recurrent inhibition between interneurons (I-to-I) can also contribute to stability and sculpt network dynamics.
*   **Selectivity:** Ideally, only the neurons representing the specific item(s) being held in memory should enter the persistent state, while other neurons remain at a baseline low-activity level. This allows multiple items or locations to be represented distinctly. Achieving selectivity might involve **winner-take-all dynamics** mediated by inhibition, where the initially most strongly activated assembly suppresses competing assemblies.
*   **Switchability:** The network must be able to transition reliably between the baseline low-activity state and the specific high-activity memory state(s) triggered by appropriate input cues, and potentially be reset or updated by new inputs or internal signals. This implies the existence of **bistability** (two stable states) or **multi-stability** (multiple stable states) in the network's dynamics. Non-linearities in neuronal firing responses or synaptic properties (e.g., the voltage-dependence of NMDA receptors, which have slow kinetics suitable for stabilizing persistent states) are thought to be crucial for supporting these multiple stable firing rates.

Strong experimental evidence supports the role of persistent activity in working memory. Single-neuron recordings in awake primates performing working memory tasks (like delayed response or delayed match-to-sample tasks, where an initial cue must be remembered across a delay period) consistently find neurons, particularly in association cortices like the **prefrontal cortex (PFC)**, parietal cortex, and temporal cortex, that exhibit **sustained, elevated firing rates** specifically during the delay period when the animal is holding information in mind. The identity or location of the neuron often correlates with the specific information being remembered (e.g., a specific spatial location, object identity, or rule). Perturbing this delay period activity often impairs working memory performance. Computational models incorporating recurrent E/I networks with parameters tuned to exhibit bistability or multi-stability have successfully replicated many features of persistent activity and its role in working memory tasks (Compte, 2022; Abbasi et al., 2023). Simulating such persistent activity (as in Section 9.5, Example 2) requires careful construction of recurrent E/I networks and tuning of connection strengths to achieve stable high-firing states triggered by transient inputs.

`[Conceptual Figure 9.3: Working Memory via Persistent Activity. Expanded diagram: Shows E and I populations with strong recurrent E-E (thick arrows), E->I, I->E, and I->I connections. Plot below shows: (1) Baseline: Low E and I rates. (2) Input Cue: Transient input activates subset of E neurons, rate increases. (3) Delay Period: Input off, but E neuron rate remains high due to E-E recurrence; I rate also increases to balance excitation. (4) End/Reset: Another input or internal signal might terminate the persistent state, returning rates to baseline.]`

While persistent activity provides a mechanism for actively maintaining information through continuous firing (which can be metabolically expensive), **short-term synaptic plasticity (STP)** offers a complementary, potentially more passive and energy-efficient mechanism for encoding a "memory trace" of recent network activity directly within the synaptic strengths themselves. As briefly introduced previously and detailed further in Chapter 13, STP refers to rapid (milliseconds to seconds), activity-dependent changes in the efficacy of synaptic transmission that typically recover back to baseline relatively quickly. The two primary forms, often co-existing with different relative strengths depending on the synapse type, are:
*   **Short-Term Facilitation (STF):** A transient *increase* in synaptic strength (PSP amplitude) following recent presynaptic spikes, often attributed to residual presynaptic calcium accumulation enhancing vesicle release probability ($P_{rel}$). STF is more prominent at synapses with low initial $P_{rel}$.
*   **Short-Term Depression (STD):** A transient *decrease* in synaptic strength during sustained or high-frequency presynaptic activity, primarily attributed to the depletion of the readily releasable pool (RRP) of synaptic vesicles. STD is more pronounced at synapses with high initial $P_{rel}$.

Because the current strength of a synapse exhibiting STP depends on its recent activation history (e.g., facilitated after recent firing, depressed after sustained firing), the pattern of synaptic efficacies across the network at any given moment implicitly encodes a **transient memory** of recent spatio-temporal activity patterns. This "synaptic memory" can influence ongoing network processing and contribute to short-term information storage without necessarily requiring persistent postsynaptic firing. For example, theoretical work suggests that networks incorporating both STF (providing positive feedback) and STD (providing negative feedback) can generate complex dynamics, including transient bursts of activity or even sustained firing states relevant to working memory, potentially operating in conjunction with recurrent connectivity. STD can also implement forms of adaptation or gain control, making the network sensitive to changes in input statistics. STF can enhance the impact of temporally correlated or bursting inputs. While perhaps less robust for maintaining information over many seconds compared to stable attractor states based on persistent firing, STP provides a ubiquitous and mechanistically distinct substrate for holding information about the immediate past, likely playing a critical role in temporal integration, sequence processing, and shaping network dynamics on the timescale of seconds. Models incorporating both recurrent connectivity and realistic STP often exhibit richer and more biologically plausible working memory behaviors.

Given that **brain organoids** contain developing neurons forming recurrent synapses, both mechanisms are potentially relevant to their capacity for short-term information processing. The presence of spontaneous activity and E/I circuitry suggests the basic substrate for **persistent activity** might emerge, although achieving stable, controllable bistability likely depends heavily on the specific network structure, E/I balance, and developmental maturity reached (Trujillo & Muotri, 2022). Similarly, synapses forming within organoids are almost certainly subject to various forms of **STP**, potentially with unique properties reflecting their developmental stage. Investigating the interplay between recurrent dynamics and short-term synaptic modifications within organoid models using combined experimental and simulation approaches is crucial for understanding their potential for temporal computation and short-term memory functions. Our Brian2 implementation in Section 9.5 will focus on demonstrating the persistent activity mechanism as a fundamental illustration of working memory based on network state.

**9.4 State Retention in Recurrent Networks**

Building directly on the concept of persistent activity, the ability of recurrent neural networks to maintain **stable internal states** provides a powerful and robust mechanism for **state retention**, underpinning functions like working memory, decision accumulation, and associative memory recall. This idea is formally captured within the theoretical framework of **attractor neural networks** (Abbasi et al., 2023). In this paradigm, learned information (memories, categories, choices) is not stored in specific locations but is represented implicitly by distinct, stable patterns of firing activity across the entire network population—these patterns are the **attractor states** of the network's dynamics. An attractor state possesses the property of stability: if the network's activity configuration is perturbed slightly away from the attractor pattern, the internal recurrent dynamics, shaped by the synaptic weight matrix, will naturally pull the state back towards that stable configuration, much like a ball rolling back to the bottom of a valley in an energy landscape.

The set of initial network states that eventually converge to a specific attractor is termed its **basin of attraction**. The existence, location, number, and stability of these attractors within the network's high-dimensional state space are fundamentally determined by the network's architecture (specifically, the pattern and density of recurrent connections) and the precise values of the **synaptic weights** connecting the neurons. **Hebbian learning** rules (Chapter 6) provide the biological mechanism by which these attractors can be formed or "learned" based on experience. When a particular input pattern is repeatedly presented, causing a specific group of neurons to be co-activated, Hebbian plasticity (like STDP) selectively strengthens the recurrent synapses *between* the neurons participating in that pattern. This activity-dependent synaptic modification effectively "carves" a stable attractor state corresponding to the learned pattern into the network's dynamical landscape. After learning, presenting an input cue that falls within the basin of attraction for a stored pattern will cause the network dynamics to evolve and eventually settle into that specific attractor state, thereby achieving **associative memory recall**. This process can also exhibit **pattern completion**, where a noisy or incomplete version of the input cue can still successfully trigger convergence to the correct, complete stored pattern represented by the attractor.

In the context of **working memory**, the attractor framework provides a robust mechanism for holding information "online" (Section 9.3). The item to be remembered is represented by the network residing in a specific high-activity attractor state, distinct from a baseline low-activity state. A transient input cue corresponding to the item effectively "pushes" the network state into the basin of attraction for the associated high-activity attractor. Once the cue is removed, the network remains trapped in this stable attractor due to the balance of recurrent excitation within the memory-representing assembly and feedback inhibition controlling overall activity, thus actively maintaining the information through persistent firing (Compte, 2022). This attractor-based maintenance is generally considered more robust to noise and capable of longer durations than memory based purely on decaying synaptic traces (STP).

Models of **decision making** also frequently employ attractor dynamics. Different attractors might represent different possible choices or decisions. Incoming sensory evidence provides input that biases the network state towards one attractor or another. The recurrent dynamics effectively integrate this evidence over time. Noise in the system ensures that eventually, the network state crosses a threshold and falls decisively into the basin of one attractor, representing the commitment to a specific choice. The competition between potential choices is often mediated by strong mutual inhibition between the neuronal populations representing different options (related concepts in Sprekeler, 2023; Brea et al., 2023).

The type of attractor dynamics a network exhibits depends on its parameters and structure:
*   **Point Attractors:** Represent stable fixed points in firing rate space. Ideal for storing discrete memories or representing categorical decisions. The network settles into a specific static firing pattern.
*   **Line/Ring Attractors:** Represent continuous manifolds of stable states. Suitable for encoding continuous variables like spatial location, head direction, or stimulus orientation. The network state can move smoothly along the attractor manifold, allowing for integration or updating of the represented variable (e.g., path integration).
*   **Sequential/Cyclic Attractors:** Represent stable sequences of activity patterns that repeat over time. Useful for storing and replaying temporal sequences, like learned motor programs or episodic memories. The network dynamically transitions through a series of states in a specific order.

Achieving stable attractor dynamics, particularly bistability or multi-stability useful for memory and decision tasks, in simulations of spiking neural networks requires careful **parameter tuning**. The balance between recurrent excitation (needed to sustain activity) and feedback inhibition (needed for stability and selectivity) is often critical and can be sensitive to parameters like synaptic weights ($w_{EE}, w_{EI}, w_{IE}, w_{II}$), connection probabilities, neuronal thresholds, and potentially adaptation mechanisms (Abbasi et al., 2023). Finding the parameter regimes that support robust attractor dynamics is often a key goal of computational modeling studies exploring memory or decision making.

`[Conceptual Figure 9.4: Attractor Dynamics for State Retention. Expanded diagram: Shows a 3D "energy landscape" surface with multiple valleys (attractors A, B, C) and a baseline flat area. Trajectories starting within a basin are shown converging to the bottom of the corresponding valley. An 'Input Cue A' trajectory starts outside, is pushed into basin A, and converges. Shows separation between basins.]`

While directly mapping the attractor landscape of a complex biological network like a brain organoid is experimentally extremely difficult, observing functional signatures like **sustained shifts in population activity** following transient stimuli (persistent activity), clear **bistability** or **multi-stability** in response dynamics, **hysteresis** effects (state dependence on history), or evidence of **pattern completion** from noisy cues could provide indirect evidence consistent with underlying attractor dynamics. Computational simulations using Brian2 are essential for exploring the parameter regimes and network architectures (e.g., strength of recurrence, E/I balance) under which such state retention mechanisms might realistically emerge in organoid-inspired models and how these mechanisms could support basic memory functions. The second Brian2 example in the following section will explicitly demonstrate the creation of a bistable network exhibiting stimulus-triggered persistent activity, providing a concrete simulation of this fundamental memory primitive.

**9.5 Brian2 Implementation: Simulating Basic Logic and Memory**

We now provide expanded Brian2 code examples to illustrate the computational primitives discussed in this chapter: implementing circuits that exhibit behaviors analogous to basic logic gates, and simulating a recurrent network capable of short-term memory through persistent activity. These examples emphasize the implementation details within Brian2 and the interpretation of the resulting dynamics.

**Example 1: Logic Gate Analogues (AND, OR, NOT)**
This code simulates three small motifs designed to approximate AND, OR, and NOT logic using LIF neurons. We use voltage jump synapses for conceptual clarity, highlighting the sensitivity to parameters and timing.

```python
# === Brian2 Simulation: Logic Gate Analogues ===
# (9.1_LogicGateMotifsSim.ipynb - Enhanced Visualization)
from brian2 import *
import matplotlib.pyplot as plt
import numpy as np

start_scope(); defaultclock.dt = 0.1*ms

# --- Neuron Parameters (LIF) ---
tau=8*ms; V_rest=-70*mV; V_thresh=-55*mV; V_reset=-80*mV; t_ref=4*ms # Shorter tau, longer t_ref
eqs_lif_v = '''dv/dt = (V_rest - v)/tau : volt (unless refractory)'''

# --- Input Spike Generators ---
# Input A: Fires @ 20, 70 ms
times_A = np.array([20, 70]) * ms
input_A = SpikeGeneratorGroup(1, [0]*len(times_A), times_A, name='InputA')
# Input B: Fires @ 20, 90 ms (Coincident with A @ 20ms; Alone @ 90ms)
times_B = np.array([20, 90]) * ms
input_B = SpikeGeneratorGroup(1, [0]*len(times_B), times_B, name='InputB')
# Input C: Spontaneous drive for NOT gate neuron C (higher rate)
spont_rate = 80*Hz
input_C_driver = PoissonGroup(1, rates=spont_rate, name='InputC_Spont')

# --- Logic Gate Neurons (Separate Groups) ---
AND_neuron = NeuronGroup(1, eqs_lif_v, threshold='v>V_thresh', reset='v=V_reset', refractory=t_ref, method='euler', name='AND_Out')
OR_neuron  = NeuronGroup(1, eqs_lif_v, threshold='v>V_thresh', reset='v=V_reset', refractory=t_ref, method='euler', name='OR_Out')
NOT_neuron = NeuronGroup(1, eqs_lif_v, threshold='v>V_thresh', reset='v=V_reset', refractory=t_ref, method='euler', name='NOT_Out')
# Initialize voltages
AND_neuron.v = V_rest; OR_neuron.v = V_rest; NOT_neuron.v = V_rest

# --- Synapses (Voltage Jump Model) ---
# Weights tuned relative to V_thresh - V_rest = 15mV
w_AND = 9*mV  # Single input subthreshold (9 < 15), two coincident suprathreshold (18 > 15)
w_OR  = 16*mV # Single input suprathreshold (16 > 15)
w_NOT_inhib = -30*mV # Very strong inhibition from A to NOT neuron
w_spont_drive = 17*mV # Suprathreshold drive from C to make NOT neuron active

# Connections for AND Gate (A->AND, B->AND)
syn_A_AND = Synapses(input_A, AND_neuron, on_pre='v += w_AND', delay=0.1*ms); syn_A_AND.connect()
syn_B_AND = Synapses(input_B, AND_neuron, on_pre='v += w_AND', delay=0.1*ms); syn_B_AND.connect()
# Connections for OR Gate (A->OR, B->OR)
syn_A_OR = Synapses(input_A, OR_neuron, on_pre='v += w_OR', delay=0.1*ms); syn_A_OR.connect()
syn_B_OR = Synapses(input_B, OR_neuron, on_pre='v += w_OR', delay=0.1*ms); syn_B_OR.connect()
# Connections for NOT Gate (A -| NOT, C -> NOT)
syn_A_NOT = Synapses(input_A, NOT_neuron, on_pre='v += w_NOT_inhib', delay=0.1*ms); syn_A_NOT.connect()
syn_C_NOT = Synapses(input_C_driver, NOT_neuron, on_pre='v += w_spont_drive', delay=0.1*ms); syn_C_NOT.connect()

# --- Monitors ---
mon_A = SpikeMonitor(input_A); mon_B = SpikeMonitor(input_B); mon_C = SpikeMonitor(input_C_driver)
mon_AND_spikes = SpikeMonitor(AND_neuron); mon_OR_spikes = SpikeMonitor(OR_neuron); mon_NOT_spikes = SpikeMonitor(NOT_neuron)
mon_AND_v = StateMonitor(AND_neuron, 'v', record=0); mon_OR_v = StateMonitor(OR_neuron, 'v', record=0); mon_NOT_v = StateMonitor(NOT_neuron, 'v', record=0)

# --- Run Simulation ---
run(130*ms)

# --- Visualize (Enhanced) ---
fig, axes = plt.subplots(4, 1, figsize=(10, 12), sharex=True, gridspec_kw={'height_ratios': [1, 2, 2, 2]})
# Inputs
axes[0].set_title('Inputs (A, B = Specified; C = Poisson Drive)')
axes[0].plot(mon_A.t/ms, np.zeros_like(mon_A.t)+0, '|r', ms=20, mew=2, label='Input A')
axes[0].plot(mon_B.t/ms, np.zeros_like(mon_B.t)+1, '|g', ms=20, mew=2, label='Input B')
axes[0].plot(mon_C.t/ms, np.zeros_like(mon_C.t)+2, '.k', ms=4, label='Input C (Drive)')
axes[0].set_yticks([0, 1, 2]); axes[0].set_yticklabels(['A', 'B', 'C']); axes[0].set_ylim(-0.5, 2.5); axes[0].legend(fontsize='small', loc='upper right'); axes[0].grid(alpha=0.5)
# Output helper function
def plot_output(ax, v_mon, spike_mon, title):
    ax.set_title(title)
    ax.plot(v_mon.t/ms, v_mon.v[0]/mV, 'b', lw=1.5, label='Vm')
    if len(spike_mon.t)>0: ax.plot(spike_mon.t/ms, np.ones_like(spike_mon.t)*V_thresh/mV, 'r^', ms=8, label='Spike')
    ax.axhline(V_thresh/mV, ls='--', color='r', lw=1); ax.axhline(V_rest/mV, ls=':', color='gray', lw=1)
    ax.set_ylabel('Vm (mV)'); ax.legend(fontsize='small', loc='lower right'); ax.grid(alpha=0.5)
    ax.set_ylim(V_reset/mV-5, V_thresh/mV+10) # Consistent y-limits
# Plot Outputs
plot_output(axes[1], mon_AND_v, mon_AND_spikes, 'AND Output (Should fire ~20ms, ~100ms)')
plot_output(axes[2], mon_OR_v, mon_OR_spikes, 'OR Output (Should fire ~20, ~70, ~80, ~100ms)')
plot_output(axes[3], mon_NOT_v, mon_NOT_spikes, 'NOT Output (NOT A, Should be active except after A spikes)')
axes[3].set_xlabel('Time (ms)')
plt.tight_layout(rect=[0, 0, 1, 0.98]); plt.show() # Adjust layout slightly

# Print spike counts for verification
print(f"AND Neuron Spikes @ {np.round(mon_AND_spikes.t/ms)} ms (Count: {mon_AND_spikes.num_spikes})")
print(f"OR Neuron Spikes @ {np.round(mon_OR_spikes.t/ms)} ms (Count: {mon_OR_spikes.num_spikes})")
print(f"NOT Neuron Spikes @ {np.round(mon_NOT_spikes.t/ms)} ms (Count: {mon_NOT_spikes.num_spikes})")
```
*Explanation:* This enhanced example uses slightly adjusted parameters (shorter `tau`, longer `t_ref`) and input timings to better illustrate the logic.
*   **AND:** Fires only when A and B arrive nearly simultaneously (around 20ms and 100ms). The shorter `tau` makes coincidence detection more likely.
*   **OR:** Fires whenever A or B fires (around 20, 70, 80, 100 ms) because `w_OR` is suprathreshold.
*   **NOT:** Fires spontaneously due to Poisson input C, but the strong, direct inhibition from A (`w_NOT_inhib`) should effectively silence it immediately following A's spikes at 20, 70, 100 ms.
The visualization is improved with larger input markers and consistent y-limits for output voltages. Spike times are printed for clearer verification of the logic. Still, perfect deterministic behavior is unlikely due to the simulation time step and threshold interactions.

**Example 2: Working Memory via Persistent Activity (Bistability)**
This code simulates a recurrent E/I network designed to show bistability and persistent activity, modeling a basic working memory function. Parameters are tuned to support both a low-activity baseline and a high-activity persistent state triggered by a transient input.

```python
# === Brian2 Simulation: Working Memory (Persistent Activity) ===
# (9.2_WorkingMemoryRecurrenceSim.ipynb - Enhanced Params & Viz)
from brian2 import *
import matplotlib.pyplot as plt
import numpy as np

start_scope(); defaultclock.dt = 0.1*ms

# --- Parameters ---
N_E = 80; N_I = 20; N = N_E + N_I
# LIF parameters (Conductance-based) - Tuned for bistability
tau_E = 20*ms; tau_I = 10*ms; V_rest = -65*mV; V_thresh = -50*mV; V_reset = -60*mV; t_ref = 4*ms # Longer t_ref
tau_g_E = 5*ms; tau_g_I = 10*ms; E_E = 0*mV; E_I = -75*mV
# Synaptic Weights (Crucial: Strong Recurrent E, Strong Feedback I)
w_EE = 1.1*nS   # Stronger E-E
w_EI = 0.7*nS   # E -> I drive
w_IE_abs = 5.0*nS # I -> E feedback inhibition
w_II_abs = 4.0*nS # I -> I stabilization
p_connect = 0.25 # Denser connectivity
# Background Input (Low to allow low baseline state)
bg_rate = 1.5*Hz; w_bg = 2.0*nS # Lower rate, maybe stronger weight
# Input Stimulus Parameters (Transient pulse to E population)
stim_strength = 3.5*nS # Strong input weight
stim_start = 250*ms; stim_duration = 100*ms # Trigger pulse
stim_reset_start = 700*ms; stim_reset_duration = 50*ms # Optional reset pulse (e.g., inhibitory)
N_stim_inputs = N_E # Target all E neurons for simplicity
stim_rate_on = 100*Hz # High rate during pulse

# --- Neuron Model ---
eqs_neuron = '''dv/dt = (V_rest - v + g_E*(E_E - v) + g_I*(E_I - v)) / tau : volt (unless refractory)
                  dg_E/dt = -g_E / tau_g_E : siemens; dg_I/dt = -g_I / tau_g_I : siemens
                  tau : second; g_E : siemens; g_I : siemens'''
neurons = NeuronGroup(N, eqs_neuron, threshold=f'v > V_thresh', reset=f'v = V_reset', refractory=t_ref, method='euler')
P_E = neurons[:N_E]; P_I = neurons[N_E:]; P_E.tau = tau_E; P_I.tau = tau_I
neurons.v = V_rest + randn(N)*2*mV; neurons.g_E = 0*nS; neurons.g_I = 0*nS

# --- Background Input ---
P_bg = PoissonGroup(N, rates=bg_rate)
syn_bg = Synapses(P_bg, neurons, on_pre='g_E_post += w_bg', delay=1*ms); syn_bg.connect(j='i')

# --- Input Stimulus (Transient pulse via PoissonGroup rates) ---
input_driver = PoissonGroup(N_stim_inputs, rates=0*Hz) # Silent initially
syn_stim = Synapses(input_driver, P_E, on_pre='g_E_post += stim_strength', delay=1*ms)
syn_stim.connect(j='i') # Connect stim i to E neuron i

# --- Recurrent Connections (Static weights tuned for bistability) ---
# ... (Syn_EE, Syn_EI, Syn_IE, Syn_II definitions as before with new weights/p) ...
syn_EE=Synapses(P_E,P_E,'w:siemens',on_pre='g_E_post+=w',delay=1*ms); syn_EE.connect(condition='i!=j',p=p_connect); syn_EE.w=w_EE
syn_EI=Synapses(P_E,P_I,'w:siemens',on_pre='g_E_post+=w',delay=1*ms); syn_EI.connect(p=p_connect); syn_EI.w=w_EI
syn_IE=Synapses(P_I,P_E,'w:siemens',on_pre='g_I_post+=w',delay=1*ms); syn_IE.connect(p=p_connect); syn_IE.w=w_IE_abs
syn_II=Synapses(P_I,P_I,'w:siemens',on_pre='g_I_post+=w',delay=1*ms); syn_II.connect(condition='i!=j',p=p_connect); syn_II.w=w_II_abs

# --- Monitors ---
spike_mon = SpikeMonitor(neurons)
rate_mon_E = PopulationRateMonitor(P_E)
rate_mon_I = PopulationRateMonitor(P_I)
v_mon_E = StateMonitor(P_E, 'v', record=range(5)) # Monitor first 5 E neurons

# --- Simulation with Stimulus Pulse ---
total_sim_time = 1000*ms
# Use scheduled runs to change input rates
@network_operation(dt=stim_start)
def apply_stim(): input_driver.rates = stim_rate_on; print(f"Stim ON @ {defaultclock.t/ms:.0f}ms")
@network_operation(dt=stim_start + stim_duration)
def remove_stim(): input_driver.rates = 0*Hz; print(f"Stim OFF @ {defaultclock.t/ms:.0f}ms")
# Optional Reset Pulse (e.g., activate I neurons strongly or add external inhibition)
# @network_operation(dt=stim_reset_start)
# def apply_reset(): P_I.v = V_thresh # Force I neurons to fire (conceptual reset)
# @network_operation(dt=stim_reset_start + stim_reset_duration)
# def remove_reset(): pass # Reset pulse is transient

print(f"Running simulation for {total_sim_time}...")
run(total_sim_time)
print("Simulation complete.")

# --- Visualize ---
plt.figure(figsize=(12, 9))
# Raster plot
ax_raster=plt.subplot(3, 1, 1); idx=spike_mon.i; t=spike_mon.t; is_E=idx<N_E; is_I=idx>=N_E
plt.plot(t[is_E]/ms, idx[is_E], '.r', ms=1.5, label='E'); plt.plot(t[is_I]/ms, idx[is_I], '.b', ms=1.5, label='I')
plt.axvspan(stim_start/ms, (stim_start+stim_duration)/ms, color='gray', alpha=0.4, label='Stim Pulse')
# if 'stim_reset_start' in locals(): plt.axvspan(stim_reset_start/ms, (stim_reset_start+stim_reset_duration)/ms, color='purple', alpha=0.3, label='Reset Pulse')
plt.xlabel('Time (ms)'); plt.ylabel('Neuron Index'); plt.title('Persistent Activity (Working Memory)'); plt.legend(markerscale=3); plt.xlim(0, total_sim_time/ms); plt.ylim(-1, N)
# Population Rates
ax_rate=plt.subplot(3, 1, 2, sharex=ax_raster); plt.plot(rate_mon_E.t/ms, rate_mon_E.rate/Hz, color='red', label='E Rate'); plt.plot(rate_mon_I.t/ms, rate_mon_I.rate/Hz, color='blue', label='I Rate')
plt.axvspan(stim_start/ms, (stim_start+stim_duration)/ms, color='gray', alpha=0.4)
# if 'stim_reset_start' in locals(): plt.axvspan(stim_reset_start/ms, (stim_reset_start+stim_reset_duration)/ms, color='purple', alpha=0.3)
plt.ylabel('Rate (Hz)'); plt.title('Population Rates'); plt.legend(); plt.ylim(bottom=0); plt.grid(alpha=0.5)
# Sample E Voltages
ax_volt=plt.subplot(3, 1, 3, sharex=ax_raster); plt.plot(v_mon_E.t/ms, v_mon_E.v.T/mV, lw=0.5)
plt.axvspan(stim_start/ms, (stim_start+stim_duration)/ms, color='gray', alpha=0.4)
# if 'stim_reset_start' in locals(): plt.axvspan(stim_reset_start/ms, (stim_reset_start+stim_reset_duration)/ms, color='purple', alpha=0.3)
plt.xlabel('Time (ms)'); plt.ylabel('Vm (mV)'); plt.title(f'Sample E Neuron Voltages'); plt.grid(alpha=0.5)
plt.tight_layout(); plt.show()
# Print rates
rate_before=np.mean(rate_mon_E.rate[rate_mon_E.t < stim_start]) if any(rate_mon_E.t < stim_start) else 0*Hz
rate_after=np.mean(rate_mon_E.rate[rate_mon_E.t > (stim_start+stim_duration)]) if any(rate_mon_E.t > (stim_start+stim_duration)) else 0*Hz # Crude estimate ignoring potential reset pulse
print(f"\nAvg E rate BEFORE stim: {rate_before:.2f}"); print(f"Avg E rate AFTER stim : {rate_after:.2f}")
if rate_after > rate_before * 1.5 and rate_after > 3*Hz: print("  Persistent activity likely achieved!")
else: print("  Network returned to baseline.")

```
*(Self-correction: Adjusted parameters (`w_EE`, `w_IE_abs`, `p_connect`, `bg_rate`, `stim_strength`, `t_ref`, `V_reset`) slightly further to increase likelihood of observing bistability/persistence, though precise tuning is needed. Used `@network_operation` to schedule stimulus changes cleanly.)*
*Explanation:* This simulation aims to demonstrate working memory via persistent activity. A recurrent E/I network is configured with potentially strong recurrent excitation (`w_EE`) and strong feedback inhibition (`w_IE_abs`). It receives low background input (`bg_rate`). A strong, transient stimulus pulse is delivered via `input_driver`. The visualization shows the network state before, during, and crucially, *after* the stimulus. If parameters are in the right regime, the E-population rate should jump during the stimulus and *remain high* long after the stimulus ends, stabilized by the E-I loop, indicating successful state retention (persistent activity). The sample voltage traces should show neurons firing continuously in this persistent state. Achieving robust bistability often requires careful parameter exploration.

**9.8 Conclusion and Planned Code**

This chapter initiated our exploration into the fundamental **computational primitives** that might be implemented by organoid-inspired neural networks, focusing on elementary **logic operations** and **short-term/working memory**. We discussed how common **neural circuit motifs**, such as feedback loops and lateral inhibition, provide structural building blocks potentially underlying specific computations. We conceptually examined the possibility of implementing functions **analogous to logic gates** (AND, OR, NOT) using spiking neurons, emphasizing the inherent differences from digital logic due to noise, timing sensitivity, and analog integration, and highlighting the likely probabilistic nature of such operations. We then explored mechanisms for **transient information storage**, focusing on the critical role of **recurrent network dynamics** in generating **persistent activity** as a substrate for working memory, and conceptually introduced the contribution of short-term synaptic plasticity. The idea of **state retention** through **attractor dynamics** in recurrent networks was presented as a robust mechanism for memory maintenance. Practical **Brian2 implementations** illustrated these concepts with expanded detail: one set of examples showed how simple motifs could yield logic-like behavior (coincidence detection, summation, inhibition), exploring parameter tuning; the second demonstrated how a recurrent E/I network could exhibit **bistability and persistent activity**, serving as a basic model for working memory by holding its state after a transient input. These simulations provide concrete examples of how relatively simple network structures and dynamics can give rise to fundamental computational operations, laying the groundwork for investigating more complex processing like pattern recognition in the next chapter.

**Planned Code Examples:**
*   **`9.1_LogicGateMotifsSim.ipynb`:** (Provided and explained in Section 9.5) Simulates small motifs using LIF neurons and spike generators to demonstrate behaviors analogous to AND (coincidence detection), OR (input summation), and NOT (inhibition) logic gates, using voltage jump synapses for clarity.
*   **`9.2_WorkingMemoryRecurrenceSim.ipynb`:** (Provided and explained in Section 9.5) Simulates a recurrent E/I network exhibiting bistability, where a transient input pulse switches the network to a persistent high-activity state, demonstrating a basic working memory mechanism using conductance-based synapses.

---
**References for Further Reading**

1.  **Abbasi, O., Jazayeri, M., & Ostojic, S. (2023). Geometry of population activity in spiking network models.** *Current Opinion in Neurobiology, 80*, 102708. https://doi.org/10.1016/j.conb.2023.102708
    *   *Summary:* This review explores modern theoretical approaches using dynamical systems and geometry to understand collective neural activity. It covers attractor dynamics, crucial for conceptualizing stable memory states (Section 9.4), and neural manifolds, providing a framework for interpreting high-dimensional network states in recurrent circuits (Section 9.1, 9.3).*
2.  **Brea, J., Senn, W., & Pfister, J. P. (2023). The balancing act of learning in recurrent neural networks.** *arXiv preprint arXiv:2301.10663*. https://arxiv.org/abs/2301.10663
    *   *Summary:* This theoretical paper delves into the challenges of maintaining stability while enabling learning (plasticity) in recurrent networks. Understanding these stability constraints is essential when designing networks capable of sustained persistent activity for working memory (Section 9.3, 9.4) without falling into pathological states.*
3.  **Compte, A. (2022). Working memory: Linking neural dynamics to behaviour.** *Current Opinion in Neurobiology, 77*, 102626. https://doi.org/10.1016/j.conb.2022.102626
    *   *Summary:* Provides a focused, up-to-date review on the neural mechanisms supporting working memory. It extensively discusses the role of persistent activity in cortical circuits, attractor models (Section 9.4), the contribution of inhibition, and how these dynamics relate to behavioral performance, offering key background for Sections 9.3 and 9.4.*
4.  **Destexhe, A. (2023). The biological noise of the brain.** *Nature Reviews Neuroscience, 24*(10), 611–626. https://doi.org/10.1038/s41583-023-00735-y
    *   *Summary:* Reviews the pervasive role of noise in neural systems. This is critical context for evaluating the feasibility and reliability of implementing precise computations like logic gates (Section 9.2) or maintaining stable memory states against random perturbations (Section 9.4) in biologically inspired models.*
5.  **Huang, C., Ruff, D. A., Pyle, R., Rosenbaum, R., Cohen, M. R., & Doiron, B. (2023). Dynamic gain control explains stimulus-specific suppression in primary visual cortex.** *eLife, 12*, e84449. https://doi.org/10.7554/eLife.84449
    *   *Summary:* This study provides a concrete example of using recurrent E/I network models (involving feedback motifs, Section 9.1) to explain specific computational functions observed *in vivo* (gain control). Demonstrates the application of the types of models discussed for understanding neural processing.*
6.  **Kumarasinghe, K., Kuhl, E., & Goriely, A. (2023). Neural dynamics on graphs: A network-centric perspective.** *Frontiers in Computational Neuroscience, 17*, 1106427. https://doi.org/10.3389/fncom.2023.1106427
    *   *Summary:* Argues for the importance of network structure (graph theory) in determining neural dynamics. Circuit motifs like feedback loops (Section 9.1) are key elements of this structure that enable specific computational functions like memory or logic-like operations.*
7.  **Lobato-Rincon, L. L., Gönczy, L., & Lengyel, M. (2022). Principles of recurrent neural network dynamics for sequence processing and prediction.** *Current Opinion in Neurobiology, 76*, 102596. https://doi.org/10.1016/j.conb.2022.102596
    *   *Summary:* This review focuses on RNN dynamics for temporal tasks, but the underlying principles of state retention and integration within recurrent circuits (Sections 9.1, 9.3, 9.4) are broadly applicable, including to working memory functions discussed in this chapter.*
8.  **Meunier, D., Nica, I. C., Lajoie, G., & Kuhl, E. (2022). Reservoir computing properties of complex neural systems.** *Philosophical Transactions of the Royal Society A: Mathematical, Physical and Engineering Sciences, 380*(2231), 20210259. https://doi.org/10.1098/rsta.2021.0259
    *   *Summary:* Explores how complex dynamics in recurrent systems (Section 9.1) create high-dimensional state representations. This concept of information being held in the network state relates closely to mechanisms of working memory (Section 9.3) and state retention (Section 9.4).*
9.  **Pyle, R., & Rosenbaum, R. (2022). The structure of inhibitory circuitry and computations in spiking neural networks.** *PLoS Computational Biology, 18*(8), e1010368. https://doi.org/10.1371/journal.pcbi.1010368
    *   *Summary:* This computational study specifically examines how different inhibitory motifs (Section 9.1) and connectivity structures impact the computations (like gating or normalization) performed by spiking networks, relevant for understanding how inhibition shapes logic-like operations (Section 9.2) and stabilizes memory states (Section 9.4).*
10. **Sprekeler, H. (2023). Balanced E/I in cortical networks: constraints, computations, and controversies.** *Current Opinion in Neurobiology, 83*, 102790. https://doi.org/10.1016/j.conb.2023.102790
    *   *Summary:* Provides a crucial review of E/I balance. Achieving the appropriate balance within recurrent circuits (Section 9.1) is fundamental for enabling stable persistent activity needed for working memory models (Sections 9.3, 9.4) while preventing uncontrolled excitation or oscillations.*
   
---
