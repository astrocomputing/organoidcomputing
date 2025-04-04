
---

# Chapter 6

# Synaptic Plasticity and Learning in Organoid Models

----

*Chapters 3 through 5 established the foundations for modeling neuronal populations and their baseline activity, including heterogeneity and spontaneous dynamics. However, a crucial element of biological neural systems, particularly relevant for both development and computation, is their remarkable capacity for **adaptation and learning**, mediated primarily by **synaptic plasticity**. Unlike the static connections assumed in our previous simple network models, biological synapses are dynamic entities whose strength or efficacy can change over time based on the patterns of neural activity they experience. This ability to modify connections is widely believed to be the fundamental cellular mechanism underlying learning, memory formation, and the refinement of neural circuits during development. Brain organoids, as developing neural systems containing neurons capable of forming synapses, are presumed to possess intrinsic mechanisms for plasticity. Understanding and modeling these mechanisms is therefore critical, not only for creating more biologically realistic simulations of organoid development and function but also for exploring their potential computational capabilities related to learning and adaptation. This chapter delves into the core concepts of synaptic plasticity relevant to organoid modeling. We begin by reviewing the **biological basis** of prominent forms of synaptic modification, including Long-Term Potentiation (LTP) and Long-Term Depression (LTD), and discuss the significance of **developmental plasticity**. We then focus on key computational models of plasticity, particularly **Hebbian learning** and its more precise formulation, **Spike-Timing-Dependent Plasticity (STDP)**, exploring their principles and implementation. Recognizing that purely Hebbian rules can lead to network instability, we then introduce crucial **homeostatic plasticity** mechanisms, such as synaptic scaling and intrinsic plasticity, that help maintain stable network function. We conceptually explore the **potential for inducing and observing learning** phenomena within brain organoid systems, acknowledging the significant experimental challenges. Finally, we provide practical **Brian2 implementations**, demonstrating how to simulate basic STDP between two neurons and how to incorporate a simple homeostatic mechanism within a small network model, illustrating the functional consequences of these adaptive processes.*

----

**6.1 Biological Basis: LTP/LTD, Developmental Plasticity, Relevance to Organoids**

The concept that the strength of connections between neurons is not fixed but can change dynamically based on experience is central to modern neuroscience. This phenomenon, broadly termed **synaptic plasticity**, encompasses a diverse range of mechanisms operating over various timescales, from milliseconds (short-term plasticity, see Chapter 13) to potentially lifelong changes (long-term plasticity). Long-term plasticity, involving persistent alterations in synaptic efficacy lasting hours, days, or longer, is widely considered the primary cellular substrate for learning and memory storage in the brain. The two most extensively studied forms of long-term plasticity at excitatory synapses in the hippocampus and neocortex are **Long-Term Potentiation (LTP)** and **Long-Term Depression (LTD)**.

**Long-Term Potentiation (LTP)** refers to a long-lasting enhancement in the strength of synaptic transmission following a brief period of high-frequency stimulation or correlated pre- and postsynaptic activity. First discovered in the rabbit hippocampus by Terje Lømo and Timothy Bliss in the 1970s, LTP has become the dominant experimental model for studying synaptic strengthening related to learning. A typical experimental protocol to induce LTP involves delivering a short burst of high-frequency electrical stimulation (e.g., 100 Hz for 1 second, often called a tetanus) to presynaptic axons while recording the resulting EPSPs in postsynaptic neurons. Following this intense stimulation, the amplitude of subsequent EPSPs evoked by single test pulses is significantly and persistently increased. The underlying cellular mechanisms are complex and involve multiple signaling cascades, but a key player at many glutamatergic synapses is the **NMDA receptor (NMDAR)**. NMDARs are unique because they are both ligand-gated (activated by glutamate) and voltage-gated (blocked by $Mg^{2+}$ ions at resting potential). Strong depolarization of the postsynaptic neuron (caused by the high-frequency stimulation activating AMPA receptors) relieves this $Mg^{2+}$ block, allowing $Ca^{2+}$ ions to flow into the postsynaptic cell through the NMDAR channel. This influx of $Ca^{2+}$ acts as a crucial second messenger, activating various downstream signaling pathways, including Calcium/calmodulin-dependent protein kinase II (**CaMKII**). CaMKII activation leads to phosphorylation of existing AMPA receptors (increasing their conductance) and, crucially, promotes the insertion of **additional AMPA receptors** into the postsynaptic membrane from intracellular stores. This increase in the number of functional AMPA receptors is a primary mechanism underlying the enhanced synaptic strength observed in LTP. LTP itself can be further divided into early-phase LTP (lasting 1-3 hours, dependent on phosphorylation and receptor trafficking) and late-phase LTP (lasting many hours or days, requiring new protein synthesis and potentially structural changes at the synapse).

**Long-Term Depression (LTD)** represents the functional opposite of LTP: a long-lasting decrease in synaptic efficacy. LTD can typically be induced experimentally by prolonged periods of **low-frequency stimulation** (LFS) of presynaptic axons (e.g., 1-5 Hz stimulation for several minutes). Following LFS, the amplitude of subsequent EPSPs is persistently reduced. Like LTP, the mechanisms underlying LTD are diverse and depend on the specific synapse and brain region, but often involve NMDARs or metabotropic glutamate receptors (mGluRs). A key factor distinguishing LTD from LTP induction is often the magnitude and duration of the postsynaptic $Ca^{2+}$ increase. While the large, rapid $Ca^{2+}$ influx via NMDARs during high-frequency stimulation triggers kinases like CaMKII leading to LTP, the smaller, more prolonged $Ca^{2+}$ rise during LFS is thought to preferentially activate **protein phosphatases** (like calcineurin/PP1 and PP2B). These phosphatases act to dephosphorylate target proteins, including AMPA receptors, and promote the **removal (internalization) of AMPA receptors** from the postsynaptic membrane via endocytosis. This reduction in the number of postsynaptic AMPA receptors leads to the observed decrease in synaptic strength. LTD is thought to be equally important as LTP for learning and memory, potentially mediating the weakening or elimination of incorrect associations, allowing for flexibility in learning, and preventing synaptic saturation.

`[Conceptual Figure 6.1: LTP and LTD Mechanisms. Panel (a) LTP Induction: High-frequency presynaptic activity + strong postsynaptic depolarization -> Glutamate release -> AMPA R activation + NMDAR Mg2+ block relieved -> Large Ca2+ influx via NMDAR -> CaMKII activation -> AMPA R phosphorylation & Insertion -> Increased synaptic strength (larger EPSP). Panel (b) LTD Induction: Low-frequency presynaptic activity + modest postsynaptic depolarization -> Glutamate release -> Weaker NMDAR activation -> Smaller, prolonged Ca2+ influx -> Phosphatase activation (PP1/PP2B) -> AMPA R dephosphorylation & Internalization -> Decreased synaptic strength (smaller EPSP).]`

Beyond these canonical forms studied primarily in adult or adolescent brain slices, **developmental plasticity** encompasses a range of activity-dependent modifications crucial for shaping neural circuits during early life, particularly within defined **critical periods**. During development, initial patterns of connectivity are often exuberant and imprecise. Spontaneous neural activity, as well as activity driven by early sensory experience, plays a vital instructive role in refining these connections. Processes like LTP and LTD are thought to operate vigorously during development, strengthening synapses that participate in correlated activity patterns relevant to function and weakening those that do not. This activity-dependent competition leads to the **stabilization** of appropriate connections and the **elimination (pruning)** of inappropriate or unused ones. Classic examples include the refinement of ocular dominance columns in the visual cortex (where inputs from the two eyes compete for cortical territory based on correlated activity) and the tuning of receptive fields in sensory cortices. Developmental plasticity ensures that the mature circuit architecture is optimally adapted to the organism's environment and functional requirements. This often involves different molecular players or regulatory mechanisms compared to plasticity in the adult brain.

The fundamental mechanisms of synaptic plasticity, both LTP/LTD-like processes and developmental refinement, are highly **relevant to brain organoids**. Given that organoids are intrinsically developing systems recapitulating aspects of early neurogenesis, migration, and synaptogenesis, it is reasonable to hypothesize that they possess the molecular machinery for, and indeed exhibit, various forms of synaptic plasticity. The expression of key molecules like NMDARs, AMPARs, CaMKII, and phosphatases has been documented in organoids. Furthermore, the spontaneous network activity observed in maturing organoids (bursts, oscillations) provides the very patterns of correlated and uncorrelated firing that are thought to drive plasticity *in vivo*. Therefore, activity-dependent synaptic modification is likely playing a crucial role in shaping the emergent connectivity and functional properties of the organoid network over its extended culture period. Understanding and modeling this intrinsic plasticity is essential not only for accurately simulating organoid development but also for assessing their potential for learning and computation. If Organoid Computing aims to leverage learning capabilities, it must engage with these endogenous plasticity mechanisms.

However, directly **studying synaptic plasticity experimentally within brain organoids presents significant challenges**. The 3D structure and optical opacity make traditional electrophysiological techniques like paired patch-clamp recordings (needed to precisely control pre- and postsynaptic activity and measure synaptic strength) extremely difficult, especially for deep neurons. Inducing LTP or LTD with patterned electrical stimulation via MEAs lacks cellular specificity. Optical methods using optogenetics combined with calcium or voltage imaging offer better specificity but face limitations in resolution, throughput, and long-term stability. Furthermore, the inherent variability and incomplete maturity of organoids can complicate the interpretation of plasticity experiments. Despite these challenges, some studies are beginning to provide evidence for plasticity in organoid systems, for example, by showing changes in network responses following periods of patterned stimulation or pharmacological manipulation of plasticity-related pathways. Nonetheless, our understanding of the specific forms, rules, and prevalence of synaptic plasticity within the unique *in vitro* environment of a developing human brain organoid remains rudimentary compared to knowledge from rodent slice physiology. Computational modeling therefore plays a particularly vital role here, allowing us to explore the *potential consequences* of incorporating known plasticity rules (derived from other systems) into organoid-like network models, generating hypotheses about their role in shaping organoid function and guiding future experimental investigations.

**6.2 Hebbian Learning and STDP: Principles and Timing Windows**

Among the most influential theoretical concepts regarding how synaptic plasticity might support learning is the **Hebbian learning principle**, famously summarized by Donald Hebb in his 1949 book "The Organization of Behavior." Often paraphrased as **"neurons that fire together, wire together,"** Hebb's postulate proposed that if a presynaptic neuron (A) repeatedly or persistently takes part in firing a postsynaptic neuron (B), then the efficacy of the synapse from A to B should increase. Conversely, synapses from presynaptic neurons that are consistently inactive when the postsynaptic neuron fires might weaken. This simple yet powerful idea suggests that synaptic strengths are adjusted based on the **correlation** between pre- and postsynaptic activity. Hebbian learning provides an elegant mechanism for **associative memory**—neurons representing features that frequently co-occur would develop strong mutual connections, forming cell assemblies that represent learned concepts or memories. Activation of part of the assembly (a partial cue) could then lead to the activation of the entire assembly (pattern completion). Hebbian-type rules, in various mathematical forms, have become a cornerstone of theoretical neuroscience and connectionist AI models for unsupervised learning, feature extraction, and memory formation.

While Hebb's original postulate was conceptual and focused on correlated firing rates, experimental discoveries, particularly concerning LTP and LTD, provided biological grounding. The requirement for postsynaptic depolarization (often leading to NMDAR activation) coincident with presynaptic activity for LTP induction aligns well with the Hebbian concept. However, further experimental work revealed that the **precise timing** of pre- and postsynaptic spikes, on the scale of milliseconds, plays a critical role in determining the sign and magnitude of synaptic change, leading to the refinement of Hebbian ideas into the framework of **Spike-Timing-Dependent Plasticity (STDP)**. Pioneering experiments in the late 1990s (e.g., by Bi & Poo, Markram et al.) using paired recordings and precisely timed spike induction demonstrated that if a presynaptic spike consistently arrives **shortly before** a postsynaptic spike (typically within a window of ~10-40 milliseconds), the synapse tends to undergo **LTP** (potentiation). Conversely, if the presynaptic spike consistently arrives **shortly after** the postsynaptic spike (again within a similar time window), the synapse tends to undergo **LTD** (depression). Spikes arriving further apart in time typically induce little or no change.

This temporally asymmetric rule, where **pre-before-post leads to LTP** and **post-before-pre leads to LTD**, is the canonical form of STDP. The relationship between the timing difference ($\Delta t = t_{\text{post}} - t_{\text{pre}}$) and the change in synaptic weight ($\Delta w$) is often characterized by an **STDP time window** or **learning window**. The typical shape involves two exponential decay curves: one for potentiation ($\Delta t > 0$) and one for depression ($\Delta t < 0$), crossing zero at $\Delta t = 0$.

```latex
% Canonical STDP Window Equation (Conceptual)
\Delta w (\Delta t) =
\begin{cases}
    A_+ \exp(-\Delta t / \tau_+) & \text{if } \Delta t > 0 \quad (\text{LTP}) \\
    -A_- \exp(\Delta t / \tau_-) & \text{if } \Delta t < 0 \quad (\text{LTD}) \\
    0 & \text{if } \Delta t = 0
\end{cases}
\tag{6.1}
```
Here, $A_+$ and $A_-$ determine the maximum potentiation and depression amplitudes, while $\tau_+$ and $\tau_-$ represent the time constants of the potentiation and depression windows, respectively (typically in the range of 10-50 ms).

`[Conceptual Figure 6.2: Canonical STDP Window. X-axis: Time difference (t_post - t_pre) in ms. Y-axis: Change in synaptic weight (% change or arbitrary units). Shows an asymmetric curve: positive peak for small positive delta_t (pre-before-post, LTP), decaying exponentially; negative peak for small negative delta_t (post-before-pre, LTD), decaying exponentially towards zero for larger time differences. Key parameters A+, A-, tau+, tau- indicated.]`

It is crucial to note, however, that this canonical window shape is a simplification. Experimental studies have revealed considerable **diversity in STDP rules** across different synapse types, brain regions, developmental stages, and dependence on factors like postsynaptic voltage, neuromodulatory state (e.g., dopamine, acetylcholine), and recent activity history. Some synapses exhibit symmetric windows, or windows where only potentiation or only depression occurs. The dependence on spike timing can be more complex than simple pairs, leading to models involving **spike triplets or quadruplets**. Furthermore, the precise molecular mechanisms linking spike timing to LTP/LTD induction are still being fully elucidated but are thought to involve the differential activation of postsynaptic signaling pathways (like CaMKII vs. calcineurin) based on the precise timing, magnitude, and duration of the postsynaptic $Ca^{2+}$ transients triggered by coincident pre- and postsynaptic activity.

From a **computational perspective**, STDP provides a powerful mechanism for circuits to learn temporal relationships and refine connectivity based on causal interactions. The potentiation of pre-before-post sequences allows networks to learn **temporal predictions** and **sequences**. The depression of post-before-pre sequences can help break spurious correlations and promote **competition** between inputs. STDP can contribute to the development of **feature selectivity**, receptive field refinement, and the synchronization or desynchronization of neuronal assemblies based on input patterns. Its sensitivity to precise spike timing makes it particularly relevant for models involving **temporal coding**. Due to its biological grounding and computational power, STDP has become one of the most widely implemented plasticity rules in spiking neural network simulations, including those aiming to model learning or development in organoid-like systems. Implementing STDP typically involves adding state variables to the synapse model to track the recent timing of pre- and postsynaptic spikes (often called "synaptic traces").

**6.3 Homeostatic Plasticity: Network Stability (Synaptic Scaling, Intrinsic Plasticity)**

While Hebbian learning rules like STDP provide compelling mechanisms for activity-dependent strengthening and weakening of synapses based on correlation and timing, potentially underlying learning and memory, they suffer from inherent **instability issues** when implemented in recurrent networks. Purely Hebbian mechanisms based on positive feedback (correlated activity strengthens connections, which leads to more correlated activity) can easily lead to **runaway dynamics**. If potentiation dominates, widespread strengthening of synapses can cause network activity to escalate uncontrollably, leading to epileptiform hyperexcitability and saturation of synaptic weights. Conversely, if depression dominates or activity levels fall too low, synapses might weaken excessively, potentially silencing the entire network. To counteract these instabilities and maintain network activity within a stable, functional dynamic range over long periods, neural circuits employ various forms of **homeostatic plasticity**. These are slower-acting regulatory mechanisms that sense deviations in overall activity levels from a desired set-point and adjust neuronal or synaptic parameters to restore stability. Homeostatic plasticity acts globally or locally to ensure that networks remain responsive and capable of further learning, preventing saturation or silencing caused by runaway Hebbian processes.

One of the best-characterized forms of homeostatic plasticity is **synaptic scaling**. This mechanism operates by **multiplicatively scaling the strengths (weights) of *all* excitatory synapses** impinging onto a given postsynaptic neuron, either upwards or downwards, to compensate for prolonged changes in that neuron's average firing rate. If a neuron's activity level remains chronically low (e.g., due to sensory deprivation or blockade of network activity), synaptic scaling mechanisms trigger a slow, multiplicative increase in the strength of all its excitatory inputs, making the neuron more sensitive and restoring its firing rate towards a target baseline. Conversely, if a neuron's activity becomes chronically high (e.g., due to excessive stimulation or blockade of inhibition), synaptic scaling induces a multiplicative decrease in its excitatory synaptic strengths, reducing its sensitivity and preventing runaway firing. Crucially, synaptic scaling is thought to be **multiplicative** and **input-specific** (affecting synapses onto a neuron, not emanating from it), meaning it preserves the *relative* strengths of different synapses established by Hebbian learning, while adjusting the overall gain. This allows the neuron to maintain its learned synaptic weight patterns while adapting its overall responsiveness. Experimental evidence for synaptic scaling comes from studies where chronic manipulation of network activity (e.g., using TTX to block spikes or bicuculline to block GABA$_A$ receptors) leads to compensatory changes in the amplitudes of miniature EPSCs (mEPSCs, reflecting postsynaptic receptor sensitivity). Proposed molecular mechanisms involve sensing average intracellular calcium levels or activity-related signaling molecules (like TNF-$\alpha$ or Arc) that regulate the trafficking and surface expression of AMPA receptors across the entire dendritic tree.

`[Conceptual Figure 6.3: Synaptic Scaling. Panel (a) Control: Shows a neuron receiving synapses with different initial weights (represented by size). Panel (b) Low Activity: Depicts chronic low firing rate detected by the neuron. Arrows indicate upregulation of AMPA receptors. Panel (c) Scaled Up: Shows all synaptic weights multiplicatively increased, restoring firing rate while preserving relative differences. Panel (d) High Activity: Depicts chronic high firing rate. Arrows indicate downregulation of AMPA receptors. Panel (e) Scaled Down: Shows all synaptic weights multiplicatively decreased.]`

Another major category of homeostatic regulation operates not at the synapse, but within the neuron itself, known as **intrinsic plasticity**. This involves activity-dependent modifications of a neuron's **intrinsic excitability**—its inherent tendency to fire action potentials in response to a given input current—independent of changes in synaptic strength. Neurons can achieve this by altering the density, distribution, or kinetic properties of various **voltage-gated ion channels** in their membrane (soma, dendrites, or axon initial segment). For example, if a neuron experiences prolonged periods of high activity, it might upregulate the expression of certain potassium ($K^+$) channels (like slow $K^+$ channels contributing to adaptation, or leak $K^+$ channels increasing resting conductance) or downregulate sodium ($Na^+$) or calcium ($Ca^{2+}$) channels. These changes would make the neuron less excitable, requiring stronger input to reach threshold, thereby counteracting the excessive activity. Conversely, prolonged inactivity might trigger downregulation of $K^+$ channels or upregulation of $Na^+$/$Ca^{2+}$ channels, increasing intrinsic excitability and making the neuron more responsive to inputs. Intrinsic plasticity acts as another crucial feedback loop, sensing average firing rates or internal calcium levels and adjusting the neuron's input-output function to maintain activity homeostasis. It often operates on timescales similar to or slower than synaptic scaling (hours to days) and likely interacts synergistically with synaptic homeostatic mechanisms to ensure overall network stability. Modeling intrinsic plasticity typically involves making parameters like firing threshold ($V_{thresh}$), resting potential ($V_{rest}$), specific ion channel conductances (if using HH-style models), or adaptation parameters (in AdEx/Izhikevich models) dependent on the neuron's recent activity history.

Beyond synaptic scaling and intrinsic plasticity, other homeostatic and regulatory mechanisms likely contribute to network stability. These include **plasticity of inhibitory synapses** (adjusting the strength of inhibition to balance changes in excitation), **metaplasticity** (activity-dependent changes in the *rules* or thresholds for inducing LTP/LTD themselves, making subsequent plasticity harder or easier depending on prior history), and regulation of neurotransmitter release probability (**short-term plasticity** can also have homeostatic effects). The overarching principle is that neural networks possess a rich repertoire of negative feedback mechanisms operating at multiple levels (synaptic, cellular, network) and timescales to counteract the potentially destabilizing effects of purely Hebbian positive feedback. Incorporating both Hebbian (for learning associations) and homeostatic (for stability) plasticity rules into computational models is therefore often essential for achieving simulations that exhibit both stable ongoing activity and the capacity for meaningful, long-term learning and adaptation, qualities that are likely critical for developing functional circuits in brain organoids.

**6.4 Potential for Inducing Learning in Organoids (Conceptual)**

Given that brain organoids contain populations of interconnected neurons and glia, express molecules known to be involved in synaptic plasticity (like NMDARs), and exhibit complex spontaneous activity patterns, a tantalizing question arises: **can these *in vitro* systems be induced to learn?** Can we, through carefully designed experimental interventions, guide the intrinsic plasticity mechanisms within an organoid to modify its network structure or function in a way that reflects adaptation to specific inputs or achieves a desired computational outcome? This prospect represents a major frontier in both organoid research and the pursuit of Organoid Computing, moving beyond passive observation towards active functional manipulation and engineering. While definitive demonstrations of complex learning remain elusive, conceptualizing how learning *might* be induced provides a framework for designing future experiments and simulations.

The core hypothesis is that by applying specific patterns of **spatio-temporal stimulation** to the organoid, potentially combined with **feedback mechanisms**, one could selectively activate Hebbian plasticity rules like STDP or related mechanisms to modify synaptic weights in a targeted manner. Conceptual approaches include:
*   **Patterned Electrical Stimulation via MEAs:** Delivering specific sequences or correlated patterns of electrical pulses through multiple electrodes on an MEA could aim to induce LTP or LTD at targeted synaptic pathways within the network, potentially encoding simple associations or stimulus representations.
*   **Patterned Optogenetic Stimulation:** Using light to control genetically modified neurons expressing opsins (like ChR2 for excitation, NpHR for inhibition) offers much higher spatial and temporal precision. One could potentially activate specific pre- and postsynaptic neurons with controlled timing to directly drive STDP, or activate larger ensembles in specific patterns to shape network connectivity.
*   **Closed-Loop Systems:** Implementing a system where the recorded activity of the organoid (e.g., from an MEA or calcium imaging) is analyzed in real-time, and this information is used to determine subsequent stimulation patterns. This creates a feedback loop. If the stimulation is designed to act as a "reward" or "error" signal contingent on the organoid producing a desired activity pattern, it might be possible to implement principles analogous to **reinforcement learning**, gradually shaping the network's dynamics towards a target behavior or computation. This approach was conceptually demonstrated in the DishBrain experiment with 2D cultures playing Pong.
*   **"Environmental Enrichment" Analogues:** For more complex organoid or assembloid systems cultured in sophisticated microfluidic devices ("organ-on-a-chip"), one might envision providing more structured "sensory" inputs (e.g., patterns of chemical gradients, flow changes, or even simple visual/auditory stimuli if appropriate sensors/transducers are integrated) and observing if the network adapts its structure or function over time in response to this enriched environment, akin to developmental learning *in vivo*.

However, attempting to induce and verify learning in brain organoids faces **immense practical and conceptual challenges**:
*   **Interfacing Limitations:** As repeatedly emphasized, reliably delivering precise, patterned stimuli to specific neurons deep within the 3D tissue and simultaneously reading out the network's response with sufficient resolution remains a major technological hurdle (the I/O problem).
*   **Biological Constraints:** The **incomplete maturity** of neurons and synapses in current organoids might limit their capacity for robust plasticity or complex learning. The lack of certain cell types (e.g., specific interneurons, microglia involved in pruning) or structures (e.g., myelination, distinct layers) could also hinder learning processes that rely on them *in vivo*.
*   **Variability and Reproducibility:** The inherent **heterogeneity** between organoids makes it difficult to design stimulation protocols that work reliably across different samples or to definitively attribute observed changes to learning rather than random fluctuations or developmental drift.
*   **Measuring Learning:** Defining and **measuring "learning"** in an isolated *in vitro* system lacking natural behavior is conceptually challenging. How do we assess if the organoid has learned a task? Changes in evoked responses, alterations in spontaneous activity patterns (e.g., emergence of new oscillations or sequences), modifications in functional connectivity metrics, or successful performance in a simple closed-loop task might serve as proxies, but interpreting these changes as genuine learning requires careful controls and validation. Directly measuring changes in synaptic weights *in situ* across the network is currently infeasible.
*   **Defining Tasks and Feedback:** Designing meaningful "computational tasks" suitable for the limited capabilities and I/O of current organoids, and devising effective "reward" or "feedback" signals that can be delivered *in vitro* to guide plasticity appropriately, are significant conceptual challenges borrowing from machine learning and control theory but adapted for biological wetware.
*   **Ethical Considerations:** As organoid models become more complex and potentially capable of more sophisticated forms of learning or adaptation, the ethical considerations surrounding their use (discussed in Sections 2.6 and 1.4) become increasingly pertinent, requiring careful oversight and public dialogue.

Despite these hurdles, some preliminary studies have provided intriguing hints. For example, researchers have shown that organoid networks can adapt their spontaneous activity patterns in response to chronic electrical stimulation or pharmacological manipulations known to affect plasticity. Others have demonstrated simple forms of associative conditioning or responsiveness changes using optogenetic stimulation. The aforementioned DishBrain experiment, while using 2D cultures, provided a compelling proof-of-concept for goal-directed learning in a closed-loop environment. Extending these paradigms to the more complex 3D structure of organoids, while technically demanding, represents an exciting future direction. Success in reliably inducing and measuring learning in organoids would not only validate their potential as platforms for studying human learning and memory mechanisms but also be a critical step towards realizing the computational aspirations of Organoid Computing. Computational modeling plays a vital role here, allowing us to simulate different stimulation protocols and learning rules *in silico* to predict which strategies might be most effective before attempting complex and resource-intensive experiments.

**6.5 Brian2 Implementation: STDP and Simple Homeostasis**

We now turn to implementing some of the core plasticity concepts discussed using the Brian2 simulator. We will focus on two key examples: implementing Spike-Timing-Dependent Plasticity (STDP) between a pair of neurons, and simulating a simple form of homeostatic plasticity (synaptic scaling) in a small network to stabilize activity. These examples illustrate how plasticity rules can be incorporated into Brian2's `Synapses` object.

**Example 1: STDP Implementation**
To model STDP, we need to track the timing of recent pre- and postsynaptic spikes. A common approach is to introduce **synaptic trace variables**. Typically, a presynaptic trace (`apre`) increases instantaneously upon presynaptic spike arrival and decays exponentially, while a postsynaptic trace (`apost`) increases upon postsynaptic spike arrival and decays exponentially. The change in synaptic weight (`w`) upon a spike event then depends on the current value of the *other* trace.

The standard additive STDP update rules can be formulated as follows:
*   Upon presynaptic spike arrival (at time $t_{\text{pre}}$):
    *   $a_{\text{pre}} \rightarrow a_{\text{pre}} + 1$ (instantaneous increase)
    *   $w \rightarrow w - \eta \cdot a_{\text{post}} \cdot w_{\text{max}}$ (LTD based on existing postsynaptic trace)
*   Upon postsynaptic spike arrival (at time $t_{\text{post}}$):
    *   $a_{\text{post}} \rightarrow a_{\text{post}} + 1$ (instantaneous increase)
    *   $w \rightarrow w + \eta \cdot a_{\text{pre}} \cdot w_{\text{max}}$ (LTP based on existing presynaptic trace)
And the traces decay exponentially:
```latex
\begin{align*}
 \tau_{\text{pre}} \frac{da_{\text{pre}}}{dt} &= -a_{\text{pre}} \\
 \tau_{\text{post}} \frac{da_{\text{post}}}{dt} &= -a_{\text{post}}
 \tag{6.2}
 \end{align*}
 ```
Here, $\tau_{\text{pre}}$ and $\tau_{\text{post}}$ control the time windows for LTP and LTD, $\eta$ is a learning rate, and $w_{\text{max}}$ (or similar bounds) are often included to keep weights within a plausible range.

Let's implement this in Brian2 for a synapse connecting two LIF neurons. We'll stimulate them with controlled timing to demonstrate LTP and LTD.

```python
# === Brian2 Simulation: STDP Implementation ===
# (6.1_STDP_Implementation.ipynb)
from brian2 import *
import matplotlib.pyplot as plt

start_scope()

# --- Neuron Parameters ---
tau = 10*ms; V_rest = -70*mV; V_thresh = -50*mV; V_reset = -75*mV; t_ref = 2*ms
eqs_lif = '''dv/dt = (V_rest - v)/tau : volt (unless refractory)'''

# --- STDP Parameters ---
tau_pre = 20*ms; tau_post = 20*ms # STDP time constants
w_max = 0.01 # Maximum synaptic weight (dimensionless here)
A_pre = 0.01 # Pre-before-post amplitude (LTP)
A_post = -0.0105 # Post-before-pre amplitude (LTD, slightly stronger to balance)
learning_rate = 1 # Factor eta included in A_pre/A_post for simplicity

# --- Create Neurons ---
# Using SpikeGeneratorGroup to precisely control spike times
pre_neuron_times = np.arange(10, 200, 25) * ms # Presynaptic spikes
post_neuron_times_ltp = pre_neuron_times + 10*ms # Post spikes 10ms AFTER pre (for LTP)
post_neuron_times_ltd = pre_neuron_times - 10*ms # Post spikes 10ms BEFORE pre (for LTD)

# Select which case to run:
post_case = 'LTP' # or 'LTD'
if post_case == 'LTP':
    post_neuron_times = post_neuron_times_ltp
    title_suffix = '(LTP: pre-before-post)'
else:
    post_neuron_times = post_neuron_times_ltd
    title_suffix = '(LTD: post-before-pre)'

pre_neuron = SpikeGeneratorGroup(1, np.zeros(len(pre_neuron_times)), pre_neuron_times)
post_neuron = SpikeGeneratorGroup(1, np.zeros(len(post_neuron_times)), post_neuron_times)

# --- Create Synapses with STDP ---
# Synaptic model includes weight 'w' and traces 'apre', 'apost'
# Differential equations for traces are included in the model string
# Weight 'w' is also declared as a variable
stdp_eqs = '''
    w : 1 # Synaptic weight (dimensionless)
    dapre/dt = -apre/tau_pre : 1 (event-driven) # Presynaptic trace dynamics
    dapost/dt = -apost/tau_post : 1 (event-driven) # Postsynaptic trace dynamics
    '''
# on_pre updates presynaptic trace and applies LTD part of rule
# on_post updates postsynaptic trace and applies LTP part of rule
syn_stdp = Synapses(pre_neuron, post_neuron,
                    model=stdp_eqs,
                    on_pre='''
                           apre += A_pre
                           w = clip(w + apost, 0, w_max)
                           ''',
                    on_post='''
                            apost += A_post
                            w = clip(w + apre, 0, w_max)
                            ''',
                    name='STDP_Synapse')
syn_stdp.connect(i=0, j=0) # Connect the two neurons
syn_stdp.w = w_max / 2 # Initialize weight in the middle

# --- Monitors ---
# Monitor the synaptic weight 'w' over time
weight_monitor = StateMonitor(syn_stdp, 'w', record=0) # Record synapse 0

# Monitor spikes to verify timing (optional)
spike_mon_pre = SpikeMonitor(pre_neuron)
spike_mon_post = SpikeMonitor(post_neuron)

# --- Run Simulation ---
simulation_time = 210*ms
run(simulation_time)

# --- Visualize Results ---
plt.figure(figsize=(10, 5))
plt.plot(weight_monitor.t/ms, weight_monitor.w[0], label='Synaptic Weight (w)')
# Add lines for initial and max weight
plt.axhline(w_max / 2, ls=':', color='gray', label='Initial Weight')
plt.axhline(w_max, ls='--', color='red', label='Max Weight (w_max)')
plt.axhline(0, ls='--', color='blue', label='Min Weight (0)')

plt.xlabel('Time (ms)')
plt.ylabel('Synaptic Weight')
plt.title(f'STDP Weight Change {title_suffix}')
plt.legend()
plt.grid(True)
plt.ylim(-0.001, w_max * 1.1) # Adjust ylim for visibility
plt.tight_layout()
plt.show()

print(f"Simulation Case: {post_case}")
print(f"Initial weight: {weight_monitor.w[0][0]:.4f}")
print(f"Final weight: {weight_monitor.w[0][-1]:.4f}")

```
**Explanation of STDP Code (`6.1_STDP_Implementation.ipynb`):**
1.  **Setup:** Imports, `start_scope()`. Defines neuron parameters (though LIF neurons aren't simulated here, parameters like `tau` might be used implicitly if needed) and STDP parameters (`tau_pre`, `tau_post`, `w_max`, amplitudes `A_pre`, `A_post`). Note `A_post` is negative for LTD.
2.  **Neuron Control:** Uses `SpikeGeneratorGroup` to create two "virtual" neurons that fire spikes at precisely specified times (`pre_neuron_times`, `post_neuron_times`). We create two scenarios: one where postsynaptic spikes occur 10ms *after* presynaptic spikes (LTP case) and one where they occur 10ms *before* (LTD case).
3.  **STDP Synapse:** Defines the `Synapses` object `syn_stdp`.
    *   `model`: Includes the synaptic weight `w` and the two trace variables `apre` and `apost`. It also includes the differential equations defining their exponential decay (`(event-driven)` indicates these equations are only updated when events occur, making it efficient).
    *   `on_pre`: Defines what happens when `pre_neuron` spikes. The presynaptic trace `apre` is increased by `A_pre`. The weight `w` is updated based on the *current* value of the postsynaptic trace (`apost`). The `clip(value, min, max)` function keeps the weight within bounds [0, `w_max`].
    *   `on_post`: Defines what happens when `post_neuron` spikes. The postsynaptic trace `apost` is increased by `A_post` (which is negative). The weight `w` is updated based on the *current* value of the presynaptic trace (`apre`).
4.  **Connection & Init:** Connects the pre- to the post-neuron and initializes the weight `w` to half its maximum value.
5.  **Monitors:** A `StateMonitor` records the synaptic weight `w` over time. `SpikeMonitor`s are included optionally to verify the input spike timing.
6.  **Run & Visualize:** Runs the simulation. The plot shows the evolution of the synaptic weight `w`. If `post_case` is 'LTP', the weight should increase towards `w_max`. If 'LTD', it should decrease towards 0.

**Example 2: Simple Homeostasis (Synaptic Scaling)**
Now, let's implement a conceptual model of homeostatic synaptic scaling in a small recurrent E/I network. We'll assume excitatory synapses onto excitatory neurons (`EE`) should scale to maintain a target firing rate in the postsynaptic neurons. We model this by adding a slow dynamic variable representing deviation from target rate and letting it modulate the EE weights. *(Note: This is a highly simplified model for illustration).*

```python
# === Brian2 Simulation: Homeostatic Plasticity (Conceptual Scaling) ===
# (6.2_PlasticNetworkActivityHomeostasis.ipynb)
from brian2 import *
import matplotlib.pyplot as plt
import numpy as np

start_scope()

# --- Parameters ---
N = 100; N_E = 80; N_I = 20
# LIF parameters (using conductance-based)
tau_E = 15*ms; tau_I = 10*ms; V_rest_E = -65*mV; V_rest_I = -70*mV
V_thresh = -50*mV; V_reset = -75*mV; t_ref = 2*ms
tau_g_E = 5*ms; tau_g_I = 10*ms; E_E = 0*mV; E_I = -75*mV
# Synaptic Weights (Initial Mean Values)
w_EE_init = 1.0*nS; w_EI = 1.0*nS; w_IE_abs = 4.0*nS; w_II_abs = 4.0*nS
p_connect = 0.1; delay_syn = 1.5*ms
# Background Input
bg_rate = 8*Hz; w_bg = 0.8*nS
# Homeostasis Parameters
target_rate = 5*Hz  # Target firing rate for E neurons
eta_homeo = 1e-4 # Homeostatic learning rate (slow)
tau_rate = 100*ms # Time constant for estimating firing rate

# --- Neuron Model ---
# Conductance-based LIF with rate estimation variable 'r_est'
eqs = '''
dv/dt = (V_rest - v + g_E*(E_E - v) + g_I*(E_I - v)) / tau : volt (unless refractory)
dg_E/dt = -g_E / tau_g_E : siemens
dg_I/dt = -g_I / tau_g_I : siemens
dr_est/dt = -r_est / tau_rate : Hz # Low-pass filtered firing rate estimate
V_rest : volt; tau : second
'''
# Create Neuron Groups
neurons = NeuronGroup(N, eqs, threshold='v>V_thresh',
                      reset='v = V_reset; r_est += (1*Hz*ms)/tau_rate', # Increment rate estimate on spike
                      refractory=t_ref, method='euler')
P_E = neurons[:N_E]; P_I = neurons[N_E:]
# Assign parameters (can add heterogeneity as in Ch 5 if desired)
P_E.V_rest = V_rest_E; P_I.V_rest = V_rest_I; P_E.tau = tau_E; P_I.tau = tau_I
neurons.v = V_rest; neurons.g_E = 0*nS; neurons.g_I = 0*nS; neurons.r_est = target_rate # Start at target rate

# --- Background Input ---
P_bg = PoissonGroup(N, rates=bg_rate)
syn_bg = Synapses(P_bg, neurons, on_pre='g_E_post += w_bg', delay=0.1*ms)
syn_bg.connect(j='i')

# --- Synapses (EE with Homeostasis) ---
# Synaptic model includes weight w and the homeostatic update rule
syn_EE = Synapses(P_E, P_E,
                  model='''dw/dt = eta_homeo * (target_rate - r_est_post) * w : siemens (clock-driven)
                           w : siemens''', # Weight is dynamic
                  on_pre='g_E_post += w',
                  delay=delay_syn, name='EE_HomeoSyn')
syn_EE.connect(condition='i!=j', p=p_connect)
syn_EE.w = w_EE_init # Initialize EE weights

# Other connections (static weights for simplicity)
syn_EI = Synapses(P_E, P_I, model='w:siemens', on_pre='g_E_post += w', delay=delay_syn)
syn_EI.connect(p=p_connect); syn_EI.w = w_EI
syn_IE = Synapses(P_I, P_E, model='w:siemens', on_pre='g_I_post += w', delay=delay_syn)
syn_IE.connect(p=p_connect); syn_IE.w = w_IE_abs
syn_II = Synapses(P_I, P_I, model='w:siemens', on_pre='g_I_post += w', delay=delay_syn)
syn_II.connect(condition='i!=j', p=p_connect); syn_II.w = w_II_abs

# --- Monitors ---
spike_mon_E = SpikeMonitor(P_E)
rate_mon_E = PopulationRateMonitor(P_E)
# Monitor a sample of EE weights
weight_sample_indices = np.random.choice(syn_EE.N, 10, replace=False) # Sample 10 weights
weight_monitor = StateMonitor(syn_EE, 'w', record=weight_sample_indices)

# --- Run Simulation ---
simulation_time = 2000*ms # Run longer to see homeostasis effects
run(simulation_time)

# --- Visualize Results ---
plt.figure(figsize=(12, 10))
# 1. Raster Plot (E neurons only)
plt.subplot(3, 1, 1)
plt.plot(spike_mon_E.t/ms, spike_mon_E.i, '.r', markersize=2)
plt.xlabel('Time (ms)'); plt.ylabel('E Neuron Index'); plt.title('Raster Plot (Excitatory)')
plt.xlim(0, simulation_time/ms); plt.ylim(-1, N_E)

# 2. Population Rate (E neurons)
plt.subplot(3, 1, 2)
plt.plot(rate_mon_E.t/ms, rate_mon_E.rate/Hz, color='red', label='Excitatory Rate')
plt.axhline(target_rate/Hz, ls='--', color='gray', label=f'Target Rate ({target_rate})')
plt.xlabel('Time (ms)'); plt.ylabel('Rate (Hz)'); plt.title('E Population Firing Rate')
plt.legend(); plt.xlim(0, simulation_time/ms); plt.ylim(bottom=0)

# 3. Sample EE Weights Evolution
plt.subplot(3, 1, 3)
for i in range(len(weight_sample_indices)):
    plt.plot(weight_monitor.t/ms, weight_monitor.w[i]/nS, lw=0.5, alpha=0.8)
plt.xlabel('Time (ms)'); plt.ylabel('Weight (nS)'); plt.title('Sample EE Synaptic Weights')
plt.xlim(0, simulation_time/ms); plt.ylim(bottom=0)

plt.tight_layout(); plt.show()

avg_final_rate_E = np.mean(rate_mon_E.rate[-int(100*ms/defaultclock.dt):]) # Avg rate in last 100ms
print(f"Average E rate in last 100ms: {avg_final_rate_E:.2f} Hz (Target: {target_rate})")
avg_final_weight_EE = np.mean(syn_EE.w)
print(f"Average EE weight at end: {avg_final_weight_EE/nS:.3f} nS (Initial: {w_EE_init/nS:.3f} nS)")

```
*(Self-correction: Implemented homeostasis by adding a rate estimation variable `r_est` to the neuron model, incremented on spikes and decaying. Added a `dw/dt` rule to the EE `Synapses` model that adjusts `w` based on the difference between the postsynaptic estimated rate `r_est_post` and the `target_rate`. Made other synapses static for simplicity. Added monitoring of sample weights.)*

**Explanation of Homeostasis Code (`6.2_PlasticNetworkActivityHomeostasis.ipynb`):**
1.  **Setup:** Defines parameters for an E/I network using conductance-based LIF neurons. Includes parameters for the homeostatic mechanism: `target_rate` (desired average firing rate for E neurons), `eta_homeo` (a slow learning rate), and `tau_rate` (time constant for estimating the neuron's firing rate).
2.  **Neuron Model:** The `eqs` string now includes a state variable `r_est : Hz` for each neuron, representing its estimated firing rate. `dr_est/dt = -r_est / tau_rate` implements leaky integration (low-pass filtering) of the firing rate. The `reset` statement is modified: when a neuron fires, `r_est` is incremented by `(1*Hz*ms)/tau_rate` (effectively adding 1 spike averaged over `tau_rate`).
3.  **Synapses:** The crucial part is the `syn_EE` (E-to-E) synapse definition.
    *   `model`: It now includes a differential equation for the weight itself: `dw/dt = eta_homeo * (target_rate - r_est_post) * w : siemens`. This rule slowly changes the weight `w` based on the difference between the postsynaptic neuron's estimated rate (`r_est_post`) and the `target_rate`. If the rate is too high, the weight decreases (scaling down); if too low, it increases (scaling up). The change is multiplicative (`* w`), consistent with synaptic scaling. `(clock-driven)` means this is updated at every simulation time step. The weight `w` is also declared as a state variable.
    *   `on_pre`: Still increments the postsynaptic conductance `g_E_post` by the *current* value of `w`.
4.  **Other Components:** Background input (`PoissonGroup`) drives activity. Other synapses (EI, IE, II) are defined with static weights for simplicity. Monitors track spikes, population rate (E), and a sample of the dynamic EE weights.
5.  **Run & Visualize:** The simulation runs for a longer duration (2000 ms) to allow the slow homeostatic process to take effect. The plots show the E neuron raster, the E population rate converging towards the `target_rate` (indicated by the dashed line), and the evolution of the sampled EE synaptic weights adjusting over time to achieve this rate stabilization.

These examples provide a practical starting point for incorporating fundamental plasticity mechanisms into Brian2 network simulations. STDP allows for activity-dependent learning based on precise spike timing, while homeostatic rules like the conceptual scaling implemented here are essential for maintaining network stability in the face of ongoing plastic changes or varying input levels—both crucial aspects for modeling adaptive biological networks like those potentially forming in brain organoids.

**6.7 Conclusion and Planned Code**

This chapter delved into the critical topic of **synaptic plasticity**, the fundamental biological mechanism enabling learning, memory, and developmental refinement in neural circuits, and explored its relevance and implementation within the context of modeling brain organoids. We reviewed the **biological basis** of long-term plasticity, focusing on the canonical forms of **LTP** and **LTD** and their underlying molecular mechanisms involving NMDA receptors, calcium signaling, and AMPA receptor trafficking. The importance of **developmental plasticity** in shaping circuits during critical periods was highlighted, establishing the strong expectation that plasticity should be active in developing organoid systems. We then examined key computational frameworks for plasticity, starting with the conceptual **Hebbian principle** ("fire together, wire together") and moving to its temporally precise refinement, **Spike-Timing-Dependent Plasticity (STDP)**, detailing its characteristic timing windows and computational implications. Recognizing the stability challenges posed by purely Hebbian rules, we introduced the concept and mechanisms of **homeostatic plasticity**, including **synaptic scaling** and **intrinsic plasticity**, which act to maintain network activity within a stable functional range. We conceptually explored the exciting **potential for inducing learning** in organoids through patterned stimulation or closed-loop paradigms, while candidly acknowledging the significant **experimental challenges** related to interfacing, biological maturity, variability, and measurement. Finally, we provided concrete **Brian2 implementations**, demonstrating step-by-step how to model additive **STDP** using synaptic traces in the `Synapses` object and how to implement a **simple homeostatic synaptic scaling** rule driven by estimated postsynaptic firing rates. These examples illustrated how plasticity rules modify synaptic weights over time, shaping network dynamics and stabilizing activity. Understanding and modeling plasticity is essential for capturing the adaptive nature of biological neural networks and exploring the potential for learning and computation in organoid-based systems. The ability to simulate these adaptive processes provides a powerful tool for investigating mechanisms that are difficult to probe experimentally in organoids.

**Planned Code Examples:**
*   **`6.1_STDP_Implementation.ipynb`:** (Provided and explained in Section 6.5) Simulates STDP between two neurons driven by precisely timed spikes using `SpikeGeneratorGroup`. Demonstrates implementing STDP rules with synaptic traces (`apre`, `apost`) in the `Synapses` model and shows weight changes corresponding to LTP and LTD protocols.
*   **`6.2_PlasticNetworkActivityHomeostasis.ipynb`:** (Provided and explained in Section 6.5) Simulates a small E/I network where E-to-E synapses undergo a simple form of homeostatic synaptic scaling aimed at stabilizing postsynaptic firing rates around a target value. Demonstrates implementing dynamic weight changes (`dw/dt`) in the `Synapses` model based on estimated neuronal activity (`r_est`). Visualizes network activity and weight evolution to show stabilization.

**6.8 References for Further Reading (APA Format)**

1.  Bi, G. Q., & Poo, M. M. (1998). Synaptic modifications in cultured hippocampal neurons: Dependence on spike timing, synaptic strength, and postsynaptic cell type. *Journal of Neuroscience, 18*(24), 10464–10472. https://doi.org/10.1523/JNEUROSCI.18-24-10464.1998 *(Seminal experimental paper establishing the basic rules of STDP.)*
2.  Bliss, T. V., & Lømo, T. (1973). Long‐lasting potentiation of synaptic transmission in the dentate area of the anaesthetized rabbit following stimulation of the perforant path. *The Journal of Physiology, 232*(2), 331–356. https://doi.org/10.1113/jphysiol.1973.sp010273 *(The classic paper first describing Long-Term Potentiation (LTP).)*
3.  Caporale, N., & Dan, Y. (2008). Spike timing-dependent plasticity: A Hebbian learning rule. *Annual Review of Neuroscience, 31*, 25–46. https://doi.org/10.1146/annurev.neuro.31.060407.125639 *(Excellent review covering the experimental phenomenology and theoretical implications of STDP.)*
4.  Davis, G. W. (2006). Homeostatic control of neural activity: From phenomenology to molecular design. *Annual Review of Neuroscience, 29*, 307–323. https://doi.org/10.1146/annurev.neuro.29.051605.112904 *(Reviews mechanisms of homeostatic plasticity, including intrinsic plasticity and synaptic scaling.)*
5.  Gerstner, W., Kempter, R., van Hemmen, J. L., & Wagner, H. (1996). A neuronal learning rule for sub-millisecond temporal coding. *Nature, 383*(6595), 76–78. https://doi.org/10.1038/383076a0 *(Early theoretical work proposing STDP-like rules before extensive experimental data.)*
6.  Hebb, D. O. (1949). *The organization of behavior: A neuropsychological theory*. Wiley. *(The original source of the Hebbian postulate, foundational for learning rules.)*
7.  Lisman, J. E., & Spruston, N. (2005). Postsynaptic factors controlling the generation and propagation of backpropagating action potentials. *Nature Reviews Neuroscience, 6*(11), 855-866. https://doi.org/10.1038/nrn1790 *(Discusses factors like backpropagating action potentials relevant for STDP induction.)*
    *Self-correction: While relevant to mechanisms, maybe a reference more focused on homeostasis modeling or organoid plasticity is better.*
    *Revised Reference 7:* Zenke, F., Agnes, E. J., & Gerstner, W. (2015). Diverse synaptic plasticity mechanisms orchestrated to form and retrieve memories in spiking neural networks. *Nature Communications, 6*, 6922. https://doi.org/10.1038/ncomms7922 *(Computational modeling paper combining different plasticity types (STDP, homeostasis) for memory function in SNNs.)*
8.  Malenka, R. C., & Bear, M. F. (2004). LTP and LTD: An embarrassment of riches. *Neuron, 44*(1), 5–21. https://doi.org/10.1016/j.neuron.2004.09.012 *(Comprehensive review of the molecular mechanisms underlying LTP and LTD.)*
9.  Turrigiano, G. G. (2012). Homeostatic synaptic plasticity: Local and global mechanisms for stabilizing neuronal function. *Cold Spring Harbor Perspectives in Biology, 4*(1), a005736. https://doi.org/10.1101/cshperspect.a005736 *(Authoritative review specifically on homeostatic synaptic plasticity mechanisms like synaptic scaling.)*
10. Turrigiano, G. G., & Nelson, S. B. (2004). Homeostatic plasticity in the developing nervous system. *Nature Reviews Neuroscience, 5*(2), 97–107. https://doi.org/10.1038/nrn1327 *(Focuses on the role of homeostatic plasticity during development, highly relevant to organoids.)*
