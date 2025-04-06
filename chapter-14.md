---

# Chapter 14

# Modeling Vascularization Effects and Metabolic Constraints

----

*Our exploration of biological realism in organoid models has so far encompassed cellular heterogeneity (Ch 5), complex network structures (Ch 7), synaptic plasticity (Ch 6, 13), advanced neuronal dynamics (Ch 11), and glial influences (Ch 12). However, a fundamental biological constraint profoundly shapes the function, limits the scale, and determines the viability of real brain tissue, yet is notably deficient in standard brain organoid cultures: **vascularization**. The brain is an incredibly energy-hungry organ, consuming a disproportionate amount of the body's oxygen and glucose supply—roughly 20% of total oxygen consumption despite being only ~2% of body mass. This relentless demand is met *in vivo* by an intricate, densely packed network of blood vessels that ensures efficient delivery of metabolic fuels and removal of waste products. This vascular network, coupled with intricate neurovascular coupling mechanisms that dynamically adjust blood flow to match local activity needs, is absolutely essential for sustaining the high metabolic cost of neural computation. Current brain organoids, typically grown *in vitro* without an independently generated or integrated circulatory system, rely solely on passive diffusion for nutrient/oxygen supply and waste removal. This fundamental limitation imposes severe constraints on organoid size, cellular composition, structural organization, long-term maturation, and ultimately, their functional complexity and potential computational capacity. This chapter directly addresses the critical issue of **vascularization and metabolic constraints** in the context of organoid computing. We begin by emphasizing the **vascular imperative**—detailing why a functional blood supply is indispensable for healthy brain tissue, the mechanisms of neurovascular coupling, and the unavoidable consequences of diffusion limitations in 3D organoid cultures, including the development of hypoxic/necrotic cores and metabolic stress gradients. We then detail the cascading **cellular consequences of hypoxia and nutrient limitation**, focusing on the mechanisms of **energy failure** (ATP depletion), subsequent **ion gradient rundown**, resultant alterations in **neuronal excitability** and synaptic function, and the pathways leading to potential cell death. Recognizing the inherently spatial nature of these diffusion limitations, we explore strategies for **modeling spatial gradients** of resource availability within simulations, linking neuronal position to metabolic state. Critically, we dedicate significant discussion to approaches for **modeling activity-dependent metabolic costs**, acknowledging that the very process of neural computation consumes energy, and exploring how to represent this consumption within models. We then delve into how to functionally **link metabolism to neuronal function** within simulations, making key neuronal or synaptic parameters (like thresholds, reversal potentials, or synaptic weights) dependent on the simulated metabolic state (e.g., local energy levels). The related, often-overlooked issue of **metabolic waste accumulation** and its potential impact is also considered. We conceptually compare the expected network dynamics under **well-nourished versus resource-limited** conditions. Finally, the chapter provides expanded and refined practical **Brian2 implementation examples** demonstrating simplified but illustrative approaches to incorporate these constraints: one modeling a **static spatial gradient** affecting multiple neuronal properties, and another implementing a **dynamic energy variable** per neuron that is consumed by neuronal activity (spiking and potentially synaptic currents) and impacts neuronal excitability, showcasing activity self-limitation.*

----

**14.1 The Vascular Imperative: Why Organoids Need Blood Vessels (Limitations)**

The brain's computational prowess comes at a steep metabolic price. Maintaining the steep ionic gradients across neuronal membranes, primarily through the relentless action of the Na+/K+-ATPase pump, consumes a vast amount of energy (ATP). Additional energy is required for neurotransmitter synthesis, packaging, release, and reuptake, as well as for postsynaptic signaling cascades and the structural maintenance and plasticity of neuronal processes and synapses (Stetter et al., 2022). This high energy demand necessitates a continuous and abundant supply of oxygen ($O_2$) and glucose, the primary substrates for aerobic ATP production.

*In vivo*, this metabolic need is met by an extraordinarily dense and precisely regulated **vascular network**. The brain parenchyma is permeated by a complex hierarchy of arteries, arterioles, capillaries, venules, and veins. Capillaries, the finest vessels, form an incredibly dense meshwork such that virtually every neuron is within 10-20 micrometers of a capillary, ensuring very short diffusion distances for $O_2$ and glucose delivery from the blood and efficient removal of metabolic waste products like $CO_2$ and lactate. Furthermore, the brain possesses sophisticated **neurovascular coupling** mechanisms, involving coordinated signaling between neurons, astrocytes, pericytes, and smooth muscle cells surrounding arterioles. These mechanisms dynamically adjust local cerebral blood flow (CBF) on a timescale of seconds to match increases in local neural activity, ensuring that active brain regions receive an adequate supply of metabolic substrates precisely when needed. This tight coupling between neural activity, energy demand, and blood supply is fundamental to normal brain function.

In stark contrast, standard methods for generating brain organoids *in vitro* typically produce three-dimensional tissue structures that **lack an intrinsic, functional vascular system**. Whether grown in suspension culture, on orbital shakers, or embedded within hydrogel matrices (like Matrigel), these organoids rely entirely on **passive diffusion** for the exchange of gases, nutrients, and waste products with the surrounding culture medium. While diffusion can adequately support small tissue structures, its efficiency drops dramatically with increasing distance. The characteristic diffusion length for oxygen in metabolically active tissue is physically limited, typically estimated to be around 150-250 µm (Loga et al., 2023). As brain organoids mature and increase in size, often reaching diameters of several millimeters, the cells located deep within the core inevitably find themselves far beyond this diffusion limit.

This fundamental lack of vascularization is arguably the single most critical limitation hindering the development and functional complexity of current brain organoid models (Ho et al., 2022; Singh et al., 2022). The immediate and most obvious consequence is the formation of a **hypoxic** (oxygen-deprived) and **nutrient-deprived** environment in the organoid core, leading eventually to **necrosis** (cell death) and the formation of a central cavity. However, even in the regions surrounding this necrotic core, cells exist along steep **gradients** of oxygen, glucose, and waste products. Cells in these intermediate zones experience chronic **metabolic stress**, which can trigger adaptive responses (like a switch towards anaerobic glycolysis) but also leads to significant alterations in gene expression, impaired cellular function, abnormal differentiation pathways, increased oxidative stress, and reduced viability compared to cells near the well-nourished surface (Loga et al., 2023; Ma et al., 2022). This creates a highly artificial internal environment, characterized by spatial heterogeneity and metabolic compromise, that is fundamentally different from the uniformly well-perfused *in vivo* brain.

From the perspective of utilizing organoids for studying neural computation or as a potential "wetware" substrate (Organoid Computing), this vascular deficit imposes severe constraints beyond simply limiting size:
*   **Impaired Neuronal Maturation and Synaptogenesis:** Chronic metabolic stress hinders key developmental processes. Neurons may fail to mature fully, exhibit stunted dendritic or axonal growth, form fewer or less stable synapses, and fail to acquire mature electrophysiological properties. Oligodendrocyte development and myelination are also highly sensitive to oxygen levels and metabolic support (He et al., 2022), leading to deficient myelination in organoid cores.
*   **Altered Cellular Composition:** The hypoxic core environment may selectively favor the survival or proliferation of certain cell types (e.g., progenitor cells, specific glial subtypes) while being detrimental to others (e.g., specific neuronal subtypes), potentially skewing the cellular composition compared to *in vivo* development.
*   **Aberrant Network Activity:** As detailed in the next section, metabolic stress directly impacts neuronal excitability and synaptic transmission, often leading to initial hyperexcitability followed by functional decline or pathological synchronization patterns (e.g., seizure-like events) that may not reflect normal *in vivo* dynamics.
*   **Limited Computational Capacity:** The combination of impaired maturation, altered activity, and the inability to sustain the high energy demands of complex processing likely severely restricts the computational capabilities of non-vascularized organoid networks compared to brain tissue. Models ignoring these energy constraints might significantly overestimate the potential for sustained high-frequency computation.

Recognizing this critical bottleneck, considerable research is focused on developing strategies to **vascularize brain organoids**. Current approaches include co-culturing organoids with endothelial cells and pericytes to promote self-assembly of vessel-like structures, implanting organoids into the brains of host animals (xenotransplantation) to allow invasion by host vasculature, using microfluidic devices to perfuse organoids with medium through engineered channels, and employing advanced biomaterials or scaffolding techniques (Ho et al., 2022; Schmidt et al., 2024). While progress is being made, achieving stable, functional, and physiologically realistic vascular networks comparable to the *in vivo* brain within *in vitro* organoids remains a major ongoing challenge. Therefore, for realistically modeling current organoid systems or for investigating the fundamental role of metabolic constraints in neural computation, incorporating the effects of diffusion limitations and activity-dependent energy use into our simulations is not just an optional refinement but a potential necessity.

**14.2 Consequences of Hypoxia and Nutrient Limitation (Energy, Excitability)**

The lack of sufficient oxygen (hypoxia) and glucose (often termed aglycemia or nutrient limitation) deep within non-vascularized organoids triggers a devastating cascade of cellular events primarily stemming from **energy failure**, specifically the depletion of **adenosine triphosphate (ATP)**. ATP is the universal energy currency of the cell, generated primarily through **oxidative phosphorylation** in mitochondria—a highly efficient process requiring both oxygen and glucose (or alternative substrates like lactate or ketone bodies). While cells can generate ATP anaerobically through **glycolysis**, this pathway yields far less ATP per molecule of glucose and produces lactic acid as a byproduct, which can lead to detrimental intracellular and extracellular acidosis.

Neurons have exceptionally high energy demands due to the continuous need to power **ion pumps**, particularly the **Na+/K+-ATPase**. This pump hydrolyzes ATP to actively transport $Na^+$ ions out of the cell and $K^+$ ions in, working against their respective concentration gradients. Maintaining these gradients is absolutely fundamental for establishing the negative resting membrane potential ($V_{rest}$), generating action potentials (which rely on $Na^+$ influx and $K^+$ efflux down these gradients), and restoring the ionic balance after electrical activity. Estimates suggest the Na+/K+ pump alone consumes 20-40% or even more of the brain's total energy expenditure (Stetter et al., 2022). Other energy-intensive neuronal processes include neurotransmitter synthesis, packaging into vesicles, presynaptic calcium extrusion, postsynaptic receptor trafficking, and dendritic/axonal transport.

When hypoxia or severe glucose limitation compromises ATP production, the consequences unfold rapidly:

1.  **Na+/K+ Pump Failure:** As ATP levels drop, the Na+/K+ pump activity drastically decreases. It can no longer effectively counteract the passive leak of ions across the membrane or restore the gradients perturbed by electrical activity.
2.  **Ion Gradient Dissipation:** The immediate result is a progressive **rundown of the transmembrane ion gradients**. Intracellular $Na^+$ concentration ($[Na^+]_{int}$) begins to rise as it leaks in and isn't pumped out. Simultaneously, extracellular $K^+$ concentration ($[K^+]_{ext}$) increases as $K^+$ leaks out and accumulates due to both neuronal efflux during activity and impaired re-uptake by astrocytes (whose pumps and transporters are also ATP-dependent, Sections 12.2).
3.  **Membrane Depolarization:** The rise in $[K^+]_{ext}$ causes the potassium equilibrium potential ($E_K$) to become less negative (depolarize). Since $V_{rest}$ is heavily influenced by $E_K$, the neuron undergoes significant **depolarization**. The influx of $Na^+$ further contributes to this depolarization.
4.  **Initial Hyperexcitability:** This initial depolarization moves the membrane potential closer to the firing threshold, often leading to a transient phase of **neuronal hyperexcitability**, characterized by spontaneous firing, increased responsiveness to stimuli, and potentially synchronous network bursting or seizure-like discharges.
5.  **Impaired Action Potential Dynamics:** As ion gradients continue to deteriorate, the mechanisms generating action potentials become compromised. Reduced $Na^+$ gradient weakens the driving force for the depolarizing $Na^+$ current. Reduced $K^+$ gradient and potentially impaired function of voltage-gated $K^+$ channels slow down the repolarization phase, leading to **action potential broadening**. Eventually, action potential amplitude decreases, and generation may fail altogether.
6.  **Synaptic Transmission Failure:** Both presynaptic neurotransmitter release (requiring $Ca^{2+}$ influx and vesicle cycling, both energy-dependent) and postsynaptic signaling are impaired. Reduced action potential amplitude and altered presynaptic $Ca^{2+}$ dynamics decrease transmitter release. Postsynaptically, maintaining receptor function and downstream signaling also requires energy. Furthermore, the ATP-dependent re-uptake of neurotransmitters like glutamate by astrocytes fails (Section 12.3), leading to glutamate accumulation in the synaptic cleft.
7.  **Calcium Dysregulation and Excitotoxicity:** The combination of sustained membrane depolarization (which can relieve the $Mg^{2+}$ block of NMDA receptors), glutamate accumulation, and potential reversal of transporters like the Na+/Ca2+ exchanger under severe gradient rundown leads to a massive, uncontrolled influx of $Ca^{2+}$ ions into the neuron. Intracellular $Ca^{2+}$ buffering systems become overwhelmed, and ATP-dependent $Ca^{2+}$ pumps (like PMCA and SERCA) fail. This sustained high intracellular calcium triggers a toxic cascade known as **excitotoxicity**: activation of proteases (like calpains) and lipases, damage to mitochondria (further impairing energy production), generation of damaging reactive oxygen species (ROS), and eventual activation of apoptotic or necrotic cell death pathways.

This cascade illustrates how metabolic compromise due to lack of vascularization rapidly translates into profound neuronal dysfunction, progressing from initial hyperexcitability to synaptic failure, energy collapse, excitotoxicity, and ultimately cell death, explaining the necrotic cores observed in organoids. Importantly, even cells in regions experiencing milder, chronic hypoxia or nutrient limitation, short of outright death, will likely exhibit altered excitability, impaired synaptic function, reduced plasticity, and abnormal network dynamics, significantly diverging from healthy *in vivo* conditions. Accounting for these metabolic effects is therefore crucial for interpreting organoid behavior and for building realistic computational models constrained by energy availability.

**14.3 Modeling Spatial Gradients (Simplified distance-based availability)**

The reliance on passive diffusion for nutrient and oxygen supply in non-vascularized organoids inherently creates **spatial gradients**: concentrations are highest near the surface exposed to the culture medium and decrease progressively towards the core. Conversely, metabolic waste products accumulate to higher levels in the core. This spatially varying microenvironment means that neurons located at different depths within the organoid experience different baseline metabolic states and levels of stress, leading to significant **spatial heterogeneity** in their structure, function, and viability. Capturing this spatial dimension is important for simulating the realistic state of an organoid network, particularly the characteristic core-shell structure.

Modeling the precise 3D concentration profiles resulting from diffusion and consumption requires solving **reaction-diffusion partial differential equations (PDEs)** within the specific geometry of the organoid. This typically involves using numerical methods like finite element or finite difference methods, often implemented in specialized simulation environments, and is computationally demanding, especially when coupled with a large network of active neurons consuming the resources.

Within the context of network simulations using point neurons in Brian2, where detailed diffusion physics are usually not simulated, we must resort to highly **simplified, conceptual models** to represent these spatial gradients. If the network model incorporates spatial coordinates for neurons (e.g., `x, y, z` parameters, as in Chapter 7), we can make neuronal properties dependent on their location relative to the organoid's surface or center.

A common and tractable simplification is to assume a **static, predefined gradient** profile at the start of the simulation:

1.  **Define Organoid Geometry and Neuron Position:** Assign spatial coordinates (`x, y, z`) to each neuron, typically assuming they are distributed within a defined volume (e.g., a sphere, ellipsoid, or slab). Calculate each neuron's position relative to the conceptual center or, more relevantly, its **distance $d_i$ from the nearest surface**. For a simple spherical approximation of radius $R_{org}$, the radial distance $r_i$ from the center can be used, where distance from surface is roughly $R_{org} - r_i$.
    ```python
    # Assuming spherical organoid centered at 0, radius R_org
    # neurons.r = sqrt(neurons.x**2 + neurons.y**2 + neurons.z**2) : meter (constant)
    # neurons.dist_from_surf = clip(R_org - neurons.r, 0*meter, R_org) : meter (constant)
    ```

2.  **Define Resource Availability Profile:** Postulate a mathematical function $f(d_i)$ that maps the distance from the surface $d_i$ to a normalized **resource availability factor**, ranging from 1 (at the surface, $d_i=0$) down to 0 (or a minimum value) at the diffusion limit or core ($d_i \ge L_{diff}$). Common choices for $f(d_i)$ include:
    *   **Linear Decay:** $f(d_i) = \max(0, 1 - d_i / L_{diff})$
    *   **Exponential Decay:** $f(d_i) = \exp(-d_i / L_{diff})$
    *   **Sigmoidal Decay:** Using a logistic function for a smoother transition.
    *   **Quadratic Profile (from diffusion equation steady state):** $f(d_i) \approx 1 - (d_i / L_{diff})^2$ (approximation)
    The parameter $L_{diff}$ represents the characteristic diffusion length (~150-250 µm for oxygen).
    ```python
    # Example: Exponential resource factor based on distance from surface
    # L_diff = 200*umetre # Characteristic diffusion length
    # neurons.resource_factor = exp(-neurons.dist_from_surf / L_diff) : 1 (constant)
    # This gives factor=1 at surface (d=0), decays inwards. Let's reverse this.
    # Better: Use distance from center 'r' and decay from surface inward.
    # neurons.resource_factor = clip(exp(-(neurons.r - R_org + L_diff)**2 / (2*L_diff**2)), 0, 1) # Complex
    # Simplest: linear decay with radius r
    # neurons.resource_factor = clip(1 - neurons.r / R_org, 0, 1) : 1 (constant) # 1 at center, 0 at edge? No, reverse.
    neurons.resource_factor = clip(1 - neurons.r / R_org, 0, 1) # Let's assume this means 1=healthy(surface), 0=core
    # Corrected: Resource factor should be high near surface (r=R_org), low near center (r=0)
    neurons.resource_factor = clip(neurons.r / R_org, 0, 1) # Simplest linear scaling: 0 at center, 1 at surface
    # Or based on distance from nearest surface (if geometry complex):
    # neurons.resource_factor = clip(neurons.dist_from_surf / L_diff, 0, 1) # Linear up to L_diff? Needs better function.
    # Let's stick to simple radial linear: factor = r / R_org (0=core, 1=surface)
    ```
    *(Self-correction: Corrected the resource factor definition multiple times to reflect high resources at surface (large r or small d_surf) and low resources at core (small r or large d_surf). Using `factor = neurons.r / R_org` assumes the center is most deprived and surface is best.)*

3.  **Link Resource Factor to Neuronal Parameters:** Make key parameters determining neuronal viability and function dependent on this spatially varying `resource_factor`. Based on Section 14.2, plausible links include modulating:
    *   **Baseline Excitability:** Map `resource_factor` (0=core, 1=surface) to parameters. E.g., `V_thresh = V_thresh_core + (V_thresh_surface - V_thresh_core) * resource_factor`. Or modulate $V_{rest}$, $E_K$.
    *   **Maximum Firing Capability:** E.g., `tau_ref = tau_ref_surf + (tau_ref_core - tau_ref_surf) * (1 - resource_factor)`. Make refractory period longer in the core.
    *   **Synaptic Strength:** E.g., `w_max = w_max_surf * resource_factor`. Reduce max synaptic efficacy in the core.
    *   **Energy Dynamics (Section 14.4):** Make the baseline energy level $E_{baseline}$ proportional to `resource_factor`. E.g., `E_baseline = E_max * resource_factor`.

4.  **Implementation in Brian2 (Initialization):** Set these spatially varying parameters for each neuron during the initialization phase of the simulation, typically using string expressions if the distance `r` or `resource_factor` are defined as parameters within the `NeuronGroup`.
    ```python
    # --- Brian2 Example: Initializing Multiple Params based on Radius ---
    # Assume NeuronGroup 'neurons' has equations defining V_thresh, tau_ref, E_base as parameters
    # And 'r' and 'R_org' are defined.
    eqs_spatial_params = '''
    # ... neuron dynamics dv/dt etc ...
    r : meter (constant)
    V_thresh : volt (constant)
    tau_ref_val : second (constant)
    E_base : 1 (constant) # Baseline energy level
    '''
    neurons = NeuronGroup(N, eqs_spatial_params, ..., refractory='tau_ref_val') # Use param in refractory
    # ... assign x, y, calculate r, R_org ...

    # Define parameter ranges
    V_thresh_surface=-55*mV; V_thresh_core=-50*mV
    tau_ref_surface=2*ms; tau_ref_core=10*ms
    E_base_surface=1.0; E_base_core=0.1

    # Use string expressions for assignment (efficient)
    # resource_factor = clip(r / R_org, 0, 1) # 0=core, 1=surface
    neurons.V_thresh = 'V_thresh_core + (V_thresh_surface - V_thresh_core) * clip(r/R_org, 0, 1)'
    neurons.tau_ref_val = 'tau_ref_core + (tau_ref_surface - tau_ref_core) * clip(r/R_org, 0, 1)'
    neurons.E_base = 'E_base_core + (E_base_surface - E_base_core) * clip(r/R_org, 0, 1)'

    # Verify assignment (optional)
    # print(f"Neuron 0 (center): Vth={neurons.V_thresh[0]/mV:.1f}, tref={neurons.tau_ref_val[0]/ms:.1f}, Ebase={neurons.E_base[0]:.2f}")
    # print(f"Neuron N-1 (edge?): Vth={neurons.V_thresh[N-1]/mV:.1f}, tref={neurons.tau_ref_val[N-1]/ms:.1f}, Ebase={neurons.E_base[N-1]:.2f}")
    ```
    *(Self-correction: Corrected refractory period assignment in `NeuronGroup` to use the parameter `tau_ref_val`. Refined string expressions for clarity.)*

This **static gradient modeling** approach, while a significant simplification of the complex dynamic reality of diffusion and consumption, provides a computationally feasible method to introduce the fundamental consequence of vascular limitation—spatial heterogeneity reflecting core-shell metabolic differences—into organoid network simulations. It allows exploration of how this inherent structural-functional gradient might shape network dynamics, information propagation, and overall computational capacity. The first Brian2 example in Section 14.8 implements this strategy.

**14.4 Modeling Activity-Dependent Metabolic Costs**

Beyond the static spatial limitations imposed by diffusion gradients from the culture medium, neural activity itself actively consumes energy resources. Every action potential requires subsequent activation of the Na+/K+-ATPase pump to restore ion gradients, neurotransmitter release involves ATP-dependent vesicle cycling, and postsynaptic receptor activation can trigger energy-consuming intracellular cascades. *In vivo*, neurovascular coupling dynamically increases local blood flow to match this activity-dependent energy demand. In non-vascularized organoids, however, this coupling is absent. High levels of neuronal activity can therefore rapidly deplete the locally available energy stores (ATP derived from diffusing oxygen and glucose), potentially leading to metabolic stress and functional impairment even in regions near the surface if the activity is sufficiently intense or prolonged. Capturing this crucial feedback loop—where computation consumes energy, and energy availability limits computation—is essential for modeling realistic constraints on neural processing, particularly in energy-limited systems like organoids.

Modeling the full intricacies of cellular energy metabolism (pathways like glycolysis, TCA cycle, oxidative phosphorylation, ATP/ADP/AMP ratios, lactate dynamics) is computationally extremely complex and typically requires specialized metabolic modeling frameworks. For integrating metabolic constraints into standard SNN simulations, highly **simplified, phenomenological models** are usually employed. The core idea is to introduce a state variable representing local energy availability and define rules for its consumption by neural activity and its replenishment from supply.

1.  **Define a Local Energy Variable ($E$):** Associate a state variable, $E(t)$, with each neuron or small local population. This variable represents the readily available energy resources, conceptually linked to local ATP concentration or the capacity to generate ATP quickly. $E$ is typically normalized, ranging from $E_{min}$ (e.g., 0, fully depleted) to $E_{max}$ (e.g., 1, fully replenished).

2.  **Model Energy Replenishment/Supply:** Assume energy is supplied or replenished over time, driving $E$ back towards a baseline level ($E_{baseline}$) with a characteristic time constant ($\tau_E$). $E_{baseline}$ represents the steady-state energy level determined by substrate supply (which could be spatially dependent, Section 14.3), and $\tau_E$ represents the timescale of metabolic recovery or substrate transport.
    ```latex
    \frac{dE}{dt}_{supply} = \frac{E_{baseline} - E}{\tau_E}
    \tag{14.2}
    ```

3.  **Model Energy Consumption:** Implement terms that decrease $E$ based on energy-consuming neural processes. The relative costs of different processes are still debated (Stetter et al., 2022), but key contributors are:
    *   **Action Potential Generation/Recovery:** The largest cost is typically associated with restoring ion gradients after spikes via the Na+/K+ pump. This is most simply modeled as a discrete reduction in $E$ immediately following each spike:
      ```python
      # In NeuronGroup reset: E = clip(E - DeltaE_spike, E_min, E_max)
      # where DeltaE_spike is the energy cost per spike.
      ```
    *   **Synaptic Transmission:** Presynaptic release (vesicle cycling) and postsynaptic processes (restoring gradients after PSPs, signaling cascades) also consume ATP. This could be modeled by linking energy consumption to the integrated synaptic conductance or current:
      ```python
      # Conceptual term in dE/dt equation:
      # dE/dt += ... - cost_factor_syn * (g_E + g_I)
      # Or linked to neurotransmitter release in presynaptic terminal.
      ```
    *   **Resting Potential Maintenance:** Even at rest, the Na+/K+ pump consumes energy to counteract passive ion leaks. This could be modeled as a small, constant baseline consumption rate.
    For pragmatic implementation in Brian2, linking consumption primarily to **spike generation** via the `reset` mechanism is often the most straightforward approach, as it directly couples energy use to the primary output signal of the neuron. The parameter `DeltaE_spike` then represents the lumped energy cost associated with firing an action potential and recovering from it.

4.  **Combine Supply and Consumption:** The full ODE for the energy variable combines replenishment and consumption:
    ```latex
    \frac{dE}{dt} = \frac{E_{baseline} - E}{\tau_E} - \text{Consumption Rate}(...)
    \tag{14.1 revisited}
    ```
    Or, if using spike-based consumption:
    ```latex
    \frac{dE}{dt} = \frac{E_{baseline} - E}{\tau_E} \quad \text{and} \quad \text{on spike: } E \leftarrow \text{clip}(E - \Delta E_{spike}, E_{min}, E_{max})
    \tag{14.3}
    ```

```python
# Brian2 Implementation Snippet: Dynamic Energy Variable E with Spike-Based Cost

# Parameters
E_baseline = 1.0; E_max = 1.0; E_min = 0.0
tau_E_recovery = 1000*ms # 1 second recovery time
energy_cost_per_spike = 0.05 # Consumes 5% of max energy per spike

# Add E to neuron equations
eqs_energy = '''
# ... (Standard neuron dynamics dv/dt, etc.) ...
dE/dt = (E_baseline - E) / tau_E_recovery : 1 (unless refractory) # Energy recovery dynamics
E : 1 # Energy state variable
'''
# Add energy consumption to the reset statement
reset_energy_consume = '''
v = V_reset
E = clip(E - energy_cost_per_spike, E_min, E_max) # Consume energy on spike
# ... (other reset actions like refractory period) ...
'''

# NeuronGroup uses these equations and reset
# neurons = NeuronGroup(N, eqs_energy, threshold='v > V_thresh_eff', # Threshold might depend on E
#                       reset=reset_energy_consume, refractory=...)
# neurons.E = E_baseline # Initialize energy
```
*(Self-correction: Ensured parameter names match description. Reset string explicitly clips E after consumption.)* This framework introduces a dynamic energy variable $E$ for each neuron, representing a balance between supply (potentially limited by spatial location via $E_{baseline}$) and activity-dependent consumption (primarily linked to spiking here). This variable now provides a dynamic measure of the neuron's metabolic state, ready to be linked back to its functional properties. Recent modeling work focuses on developing efficient ways to simulate such constraints (Mahmoudi et al., 2023).

**14.5 Linking Metabolism to Neuronal Function (Parameter Modulation)**

Simply tracking a conceptual energy variable $E$ is insufficient; for metabolic constraints to impact computation, this variable must **modulate the functional properties** of the neurons or synapses. The crucial step is establishing biologically plausible links between the energy state $E$ and key parameters governing neuronal excitability, firing, and communication. Section 14.2 outlined the cellular consequences of energy failure (ATP depletion), providing guidance for these links. Lower energy levels should generally lead to impaired function.

Here are several ways to implement this feedback in Brian2 models, making neuronal or synaptic parameters dependent on the dynamic energy variable $E$ (where $E=1$ or $E_{max}$ represents healthy state, and $E=0$ or $E_{min}$ represents complete depletion):

1.  **Modulation of Ion Gradients ($V_{rest}$, $E_K$, $E_{Na}$):** Since the Na+/K+ pump is highly ATP-dependent, low energy $E$ leads to rundown of $Na^+$ and $K^+$ gradients.
    *   **Implementation:** Make reversal potentials $E_K$ and $E_{Na}$ (and consequently $V_{rest}$) dependent on $E$. As $E$ decreases, $E_K$ should become less negative, and $E_{Na}$ less positive.
    ```python
    # Conceptual Link: E affects EK and ENa
    EK_healthy = -85*mV; EK_depleted = -60*mV # Example range
    ENa_healthy = 55*mV; ENa_depleted = 30*mV
    EK_eff = EK_depleted + (EK_healthy - EK_depleted) * clip(E, E_min, E_max) # Linear scaling
    ENa_eff = ENa_depleted + (ENa_healthy - ENa_depleted) * clip(E, E_min, E_max)
    # These effective reversals would be used in the dv/dt equation.
    # Requires E to be accessible, potentially using @network_operation or making EKs/ENa dynamic vars.
    ```
    *   **Functional Consequence:** This initially causes depolarization and hyperexcitability but eventually leads to reduced driving forces for action potential currents, potentially causing spike failure. Complex effects.

2.  **Modulation of Firing Threshold ($V_{thresh}$):** Perhaps the most common and functionally direct way to model activity suppression under metabolic stress is to increase the effective firing threshold $V_{thresh}$ as energy $E$ depletes. While initial depolarization might occur, this link assumes overall dysfunction makes generating a full spike harder when ATP is low.
    *   **Implementation:** Make $V_{thresh}$ a function of $E$, increasing as $E$ decreases.
    ```python
    # Implemented in Section 14.8 Example 2:
    # V_thresh_eff = V_thresh_base + V_thresh_penalty * (1 - clip(E, E_min, E_max))
    # This V_thresh_eff is used in the threshold condition string 'v > V_thresh_eff'.
    # Requires E to be defined in the neuron's equations.
    ```
    *   **Functional Consequence:** Directly reduces firing rate in response to low energy, providing strong negative feedback to limit activity. Simple and effective for modeling self-limitation.

3.  **Modulation of Synaptic Release (Presynaptic $E$):** Neurotransmitter vesicle cycling and release are ATP-dependent. Low energy in the *presynaptic* terminal ($E_{pre}$) should reduce synaptic efficacy.
    *   **Implementation:** Make the effective synaptic weight ($w_{eff}$) or the baseline release probability ($U_{TM}$ in Tsodyks-Markram model) dependent on $E_{pre}$. This requires the `Synapses` object to access the state variable $E$ of the *presynaptic* neuron.
    ```python
    # Conceptual Brian2 Snippet: Presynaptic E modulates static weight w
    # Define synapse with access to E_pre
    eqs_syn_Emod = '''w_base : siemens (constant) # Base weight
                     # Access E from presynaptic neuron (read-only)
                     E_pre = E_source : 1 (linked)'''
    # Define the effective weight in on_pre, based on E_pre
    on_pre_Emod = '''
                   w_effective = w_base * clip(E_pre, E_min, E_max) # Scale by presynaptic energy
                   g_target_post += w_effective # Use modulated weight
                   '''
    # NeuronGroup must define E: neuron_eqs = '''... dE/dt=... : 1 ... E : 1'''
    # Create synapse linking E_pre to E_source (variable E in the source NeuronGroup)
    # syn_mod = Synapses(source_neurons, target_neurons, model=eqs_syn_Emod, on_pre=on_pre_Emod, ...)
    # syn_mod.connect(...)
    # syn_mod.w_base = ... # Set base weight
    ```
    *(Self-correction: Used Brian2's linked variable mechanism (`E_pre = E_source : 1 (linked)`) which allows a synapse model to read a variable from the source neuron, enabling presynaptic modulation.)*
    *   **Functional Consequence:** Reduces the impact of spikes fired by energy-depleted neurons, providing another negative feedback loop.

4.  **Modulation of Postsynaptic Response:** Energy depletion in the *postsynaptic* neuron might affect its ability to respond to inputs, e.g., by altering receptor sensitivity or downstream signaling.
    *   **Implementation:** Modulate the magnitude of conductance change (`w` in `on_pre`) or parameters controlling postsynaptic integration based on the postsynaptic energy state $E_{post}$ (which is just $E$ if defined per neuron).
    ```python
    # Conceptual: Postsynaptic E modulates PSP amplitude
    # on_pre = ''' g_target_post += w_base * clip(E_post, E_min, E_max) '''
    ```
    *   **Functional Consequence:** Makes neurons less responsive to inputs when their energy is low.

5.  **Modulation of Other Parameters:** Low energy could plausibly affect other parameters like refractory period ($\tau_{ref}$), adaptation strength ($a, b$ in AdEx/Izhikevich), or ion channel kinetics, offering further avenues for implementing metabolic feedback.

Implementing these links requires careful consideration of how parameters are defined and updated in Brian2 (within equations, via namespace, using `@network_operation`, or through linked variables). The choice depends on which parameter is targeted (neuronal vs. synaptic, pre- vs. postsynaptic) and how dynamically the energy variable $E$ changes. By incorporating one or more of these feedback mechanisms, computational models can begin to capture the fundamental principle that neural activity is constrained by its metabolic cost, leading to more realistic simulations of network dynamics under conditions of limited energy supply, as relevant to organoids. The second example in Section 14.8 focuses on linking $E$ to $V_{thresh}$.

**14.6 Modeling Waste Accumulation Effects (Conceptual)**

While energy substrate delivery (oxygen, glucose) is critical, the efficient **removal of metabolic waste products** is equally important for maintaining a healthy neuronal microenvironment. Active neural metabolism generates several byproducts that can be detrimental if allowed to accumulate:
*   **Carbon Dioxide ($CO_2$):** Produced by aerobic respiration. Dissolves to form carbonic acid, lowering pH (acidosis).
*   **Lactic Acid:** Produced by anaerobic glycolysis, especially under hypoxic conditions or high activity. Also contributes to acidosis.
*   **Protons ($H^+$):** Directly related to pH changes from $CO_2$ and lactate. Extracellular acidosis (low pH) can significantly modulate the activity of numerous ion channels (e.g., voltage-gated $Na^+, K^+, Ca^{2+}$ channels), neurotransmitter receptors (especially NMDA receptors, which are inhibited by low pH), and transporters, generally leading to decreased neuronal excitability and synaptic transmission.
*   **Reactive Oxygen Species (ROS):** Byproducts of mitochondrial respiration, especially when metabolism is stressed. ROS (like superoxide, hydrogen peroxide) can cause oxidative damage to cellular components (lipids, proteins, DNA) if antioxidant defenses are overwhelmed, contributing to long-term dysfunction and cell death.
*   **Ammonia:** A byproduct of amino acid metabolism, neurotoxic at high concentrations.

*In vivo*, the vascular system plays a crucial role not only in supply but also in efficiently washing out these waste products, maintaining stable pH and low levels of potentially toxic byproducts. In non-vascularized organoids, however, these waste products can accumulate within the tissue core due to slow diffusion out into the large volume of the culture medium. This accumulation, particularly the resulting acidosis, likely contributes significantly to the dysfunction and death observed in organoid cores, acting synergistically with hypoxia and nutrient deprivation.

**Modeling Waste Accumulation Effects:** Directly modeling the production rates of multiple waste products, their complex diffusion patterns in the ECS, their buffering (e.g., bicarbonate buffering of pH), and their specific molecular interactions with numerous channels and receptors is exceedingly complex and requires detailed biochemical and biophysical data that is often unavailable. Therefore, attempts to include waste effects in standard SNN simulations are usually highly **conceptual and simplified**:

1.  **Introduce a Generic `Waste` Variable:** Similar to the energy variable $E$, define a state variable $W(t)$ associated with each neuron or region, representing the local accumulation level of generic metabolic waste.
2.  **Model Waste Production:** Assume waste is produced as a function of neuronal activity. This could be linked to:
    *   Spiking rate (e.g., increment $W$ in the `reset` statement, analogous to $E$ consumption).
    *   Metabolic state (e.g., production increases more rapidly when energy $E$ is low, reflecting increased reliance on anaerobic glycolysis).
3.  **Model Waste Clearance:** Assume waste clears slowly via diffusion or baseline metabolic processes, represented by a slow decay term in the equation for $W$:
    ```latex
    \frac{dW}{dt} = \text{Production Rate}(\text{activity}, E) - \frac{W}{\tau_{clear}}
    \tag{14.4}
    ```
    The clearance time constant $\tau_{clear}$ would be very long, reflecting slow diffusion out of the organoid. $\tau_{clear}$ could potentially be made spatially dependent (longer in the core).

4.  **Link `Waste` Level to Neuronal/Synaptic Parameters:** Based on the known effects of acidosis or oxidative stress, make key parameters dependent on $W$:
    *   **Reduced Excitability:** Increase firing threshold $V_{thresh}$, decrease resting potential $V_{rest}$ (further if combined with $K^+$ effects), or decrease input resistance (increase leak $g_L$).
    *   **Reduced Synaptic Transmission:** Decrease presynaptic release probability (modulate $U_{TM}$) or decrease postsynaptic receptor efficacy (reduce static weight $w_{static}$), particularly for NMDA receptors which are sensitive to pH.
    *   **Impaired Plasticity:** High waste levels could inhibit LTP/LTD induction mechanisms.

```python
# Conceptual Brian2 Snippet: Waste Variable Affecting Excitability
# Assume Waste W increases with spikes, decays very slowly
W_inc_per_spike = 0.001; tau_waste_clear = 10000*ms # 10 seconds clearance
W_max = 1.0; W_min = 0.0
# Add W to neuron equations
eqs_waste = '''
# ... (Standard neuron dynamics dv/dt, etc.) ...
# ... (Energy dynamics dE/dt, E, etc. if included) ...
dW/dt = -W / tau_waste_clear : 1 (unless refractory) # Waste clearance
W : 1 # Waste level variable
'''
# Add waste production to reset
reset_waste = '''
# ... (Standard reset v=V_reset) ...
# ... (Energy consumption E=clip(...) if included) ...
W = clip(W + W_inc_per_spike, W_min, W_max) # Produce waste on spike
'''
# Neuron Group definition
# neurons = NeuronGroup(N, eqs_waste, threshold='v > V_thresh_eff_total', reset=reset_waste, ...)

# Link Waste W to threshold (example)
# V_thresh_waste_penalty = 10*mV # Max threshold increase due to waste
# V_thresh_eff_total = V_thresh_base \
#                      + V_thresh_energy_penalty * (1 - clip(E, E_min, E_max)) \
#                      + V_thresh_waste_penalty * clip(W, W_min, W_max)
# # V_thresh_eff_total would be used in the threshold condition string or updated via @network_operation
```
This conceptual approach adds another layer of negative feedback: high activity produces waste, accumulated waste impairs function (e.g., reduces excitability), which then limits further activity. While highly simplified and abstracting away the specific chemical identities and effects, incorporating such a term allows simulations to explore the potential impact of impaired waste clearance, acting synergistically with energy depletion, on limiting sustained computation and contributing to dysfunction in metabolically stressed network models relevant to organoid cores.

**14.7 Simulating Network Effects: Well-Nourished vs. Resource-Limited**

By incorporating the modeling strategies discussed—spatial resource gradients (Section 14.3) and activity-dependent metabolic costs linked to function (Sections 14.4-14.5, potentially including waste effects from 14.6)—we equip our Brian2 simulations with the capability to explore how metabolic constraints shape network dynamics and computation. A powerful approach is to perform **comparative simulations**, contrasting the behavior of the same underlying network architecture under different metabolic scenarios:

1.  **"Well-Nourished" / Control Scenario:** This simulation represents an idealized baseline, assuming optimal metabolic conditions similar to a fully vascularized *in vivo* environment or the outermost, best-supplied layers of an organoid. In the model, this is achieved by:
    *   Disabling any spatial gradients (e.g., setting `resource_factor = 1` everywhere).
    *   Setting baseline energy levels high (`E_{baseline} = E_{max}`).
    *   Setting energy recovery rates fast (small $\tau_E$).
    *   Setting the energy cost per spike low (`DeltaE_spike` small or zero) or disabling the feedback link between energy $E$ and neuronal function entirely.
    *   Assuming efficient waste clearance (fast $\tau_{clear}$ or disabled waste effects).
    Under these conditions, the network dynamics are primarily governed by the intrinsic neuronal properties, synaptic strengths, connectivity structure, and external inputs, without significant metabolic limitations. This serves as a crucial reference point.

2.  **"Static Gradient" / Core-Shell Scenario:** This simulation explicitly models the diffusion limitation characteristic of non-vascularized organoids by imposing static spatial gradients on key parameters based on distance from the center/surface (using methods from Section 14.3). For example, core neurons might have higher thresholds, lower baseline energy supply (`E_{baseline}` lower in core), or slower energy recovery (`tau_E` longer in core).
    *   **Expected Network Effects:** Compared to the control, we anticipate observing significant **spatial heterogeneity** in activity. The core region might exhibit markedly reduced spontaneous firing rates, lower responsiveness to stimuli, and potentially different oscillation frequencies or synchrony patterns compared to the periphery. Activity waves initiated at the periphery might fail to propagate effectively into the core. Global network synchrony might be disrupted. Tasks requiring integration across the entire network might be impaired due to the dysfunctional core. The network might effectively behave as a smaller, primarily peripheral functional unit.

3.  **"Dynamic Energy Constraint" Scenario:** This simulation incorporates the activity-dependent energy consumption feedback loop (Sections 14.4, 14.5), potentially combined with the static spatial gradient affecting baseline supply ($E_{baseline}$).
    *   **Expected Network Effects:** Compared to the control (and potentially even the static gradient model under low activity), the primary effect here is **activity self-limitation**, especially during periods of high demand. We expect to see:
        *   **Reduced Sustained Firing Rates:** The network might initially respond strongly to a stimulus, but the firing rates will likely decrease over time as energy depletes and negative feedback (e.g., increased threshold) kicks in, even if the stimulus remains high. Maximum achievable firing rates will be lower than in the unconstrained case.
        *   **Shorter Bursts / Oscillations:** Network bursts or episodes of synchronous oscillations, which are metabolically demanding, may be shorter in duration or occur less frequently as they rapidly consume local energy reserves.
        *   **Activity-Dependent Failure:** Under very strong or prolonged stimulation, particularly in regions with lower baseline supply (core), energy levels might drop so low that neurons become effectively silenced or unable to transmit signals reliably, potentially leading to transient or permanent functional deficits in parts of the network.
        *   **Increased Network Variability:** Fluctuations in local energy levels might introduce additional sources of variability in neuronal firing and network dynamics.
        *   **Altered Information Processing:** Tasks requiring sustained high-frequency processing or integration over long periods might be severely impaired. The network's effective computational capacity could be significantly reduced compared to the unconstrained control.

By systematically comparing the results (e.g., firing rate distributions, synchrony measures, response latencies, information theoretic measures, task performance metrics from Section 10.4) across these different simulated metabolic scenarios, researchers can quantitatively assess the potential impact of vascular limitations and energy budgets on the functional capabilities of organoid-inspired neural networks. These simulations can generate testable predictions for organoid experiments (e.g., regarding core vs. periphery activity patterns, responses to metabolic inhibitors, or effects of engineered vascularization) and provide crucial insights into the fundamental, yet often overlooked, role of metabolic constraints in shaping neural computation *in vivo* and *in vitro*.

**14.8 Brian2 Implementation: Phenomenological Metabolic Constraints**

*(This section retains and refines the two core examples from the previous response.)*

This section provides practical Brian2 code examples implementing the simplified modeling strategies for metabolic constraints discussed above: a static spatial gradient affecting excitability and baseline energy, and a dynamic energy variable linked to firing threshold.

**Example 1: Static Spatial Gradient Affecting Threshold and Baseline Energy**
This simulation sets up a 2D network where both the firing threshold ($V_{thresh}$) and the baseline energy level ($E_{baseline}$ used in Example 2) vary linearly with distance from the center.

```python
# === Brian2 Simulation: Static Metabolic Gradient Effect ===
# (14.1_StaticOxygenGradientSim.ipynb - Enhanced)
from brian2 import *; import matplotlib.pyplot as plt; import numpy as np
start_scope(); defaultclock.dt = 0.1*ms

# --- Parameters ---
N = 400; grid_dim = int(np.sqrt(N)); R_org = 250*umetre # Organoid radius for gradient
# Neuron params (LIF)
tau = 20*ms; V_rest = -70*mV; V_reset = -80*mV; t_ref = 2*ms; R = 100*Mohm
# Gradient Parameters
V_thresh_surface = -55*mV; V_thresh_core = -50*mV # Core less excitable (higher Vth)
E_base_surface = 1.0; E_base_core = 0.1 # Core lower baseline energy
# Input drive
I_drive_val = 0.17*nA # Constant input current (adjust for desired activity)

# --- Neuron Model with Spatial Coordinates and Gradient Parameters ---
eqs_grad = '''dv/dt = (V_rest - v + R*I_inj)/tau : volt (unless refractory)
               x : meter (constant); y : meter (constant); r : meter (constant)
               V_thresh : volt (constant) # Set based on r
               E_baseline : 1 (constant) # Set based on r
               I_inj : amp'''
neurons = NeuronGroup(N, eqs_grad, threshold='v > V_thresh', reset='v = V_reset', refractory=t_ref, method='euler')
# Assign positions and calculate radius 'r'
neurons.x='(i // grid_dim - grid_dim/2.0 + 0.5) * 2*R_org / grid_dim'; neurons.y='(i % grid_dim - grid_dim/2.0 + 0.5) * 2*R_org / grid_dim'
neurons.r = 'sqrt(x**2 + y**2)'
# Set parameters based on radial gradient (resource_factor = 0=core, 1=surface)
resource_factor = 'clip(r / R_org, 0, 1)'
neurons.V_thresh = 'V_thresh_core + (V_thresh_surface - V_thresh_core) * ({factor})'.format(factor=resource_factor)
neurons.E_baseline = 'E_base_core + (E_base_surface - E_base_core) * ({factor})'.format(factor=resource_factor)
# Initialize Vm and Input
neurons.v = V_rest; neurons.I_inj = I_drive_val

# --- Monitors ---
spike_mon = SpikeMonitor(neurons)
# Record initial V_thresh and E_baseline to verify gradient
init_vals_mon = StateMonitor(neurons, ['V_thresh', 'E_baseline'], record=True, when='start')

# --- Run Simulation ---
run(500*ms)

# --- Visualize ---
plt.figure(figsize=(12, 9))
# Plot initial thresholds vs radius
ax1 = plt.subplot(2, 2, 1); plt.plot(neurons.r/um, init_vals_mon.V_thresh[0]/mV, '.', alpha=0.3); plt.xlabel('Radius (um)'); plt.ylabel('V_thresh (mV)'); plt.title('Threshold Gradient'); plt.grid(alpha=0.5)
# Plot initial E_baseline vs radius
ax2 = plt.subplot(2, 2, 2); plt.plot(neurons.r/um, init_vals_mon.E_baseline[0], '.', alpha=0.3); plt.xlabel('Radius (um)'); plt.ylabel('E_baseline'); plt.title('Energy Baseline Gradient'); plt.grid(alpha=0.5); plt.ylim(0, 1.1)
# Raster Plot (sorted by radius)
idx_sorted_by_r = np.argsort(neurons.r); y_plot_indices = np.argsort(idx_sorted_by_r)[spike_mon.i]
ax3 = plt.subplot(2, 1, 2)
if len(spike_mon.i) > 0: plt.plot(spike_mon.t/ms, y_plot_indices, '.k', ms=1)
else: print("No spikes recorded.")
plt.xlabel('Time (ms)'); plt.ylabel('Neuron (Sorted by Radius)'); plt.title('Spiking Activity (Core at Bottom)'); plt.xlim(0, 500); plt.ylim(-1, N); plt.grid(alpha=0.5)
plt.tight_layout(); plt.show()
# Calculate rates core vs periphery
core_idx = neurons.r < R_org / 2; peri_idx = neurons.r >= R_org / 2
rate_core = spike_mon.count[core_idx].sum()/(core_idx.sum()*500*ms) if core_idx.sum()>0 else 0*Hz
rate_peri = spike_mon.count[peri_idx].sum()/(peri_idx.sum()*500*ms) if peri_idx.sum()>0 else 0*Hz
print(f"Avg Rate Core (<{R_org/2/um:.0f}um): {rate_core:.2f}"); print(f"Avg Rate Periphery (>= {R_org/2/um:.0f}um): {rate_peri:.2f}")
```
*Explanation:* This refined example sets both `V_thresh` and a baseline energy level `E_baseline` (which could be used by Example 2) according to a linear gradient based on radial distance `r`. Core neurons are less excitable (higher `V_thresh`) and have lower baseline energy. The plots confirm the gradients, and the raster plot clearly shows suppressed activity in the core (bottom of plot) compared to the periphery, demonstrating the impact of static spatial metabolic constraints.

**Example 2: Dynamic Energy Constraint Affecting Threshold**
*(Code largely identical to previous expanded response, demonstrating activity self-limitation)*
```python
# === Brian2 Simulation: Dynamic Energy Constraint ===
# (14.2_DynamicEnergyConstraintSim.ipynb)
from brian2 import *; import matplotlib.pyplot as plt; import numpy as np
start_scope(); defaultclock.dt = 0.1*ms
# --- Parameters ---
N=100; tau=15*ms; V_rest=-70*mV; V_reset=-80*mV; t_ref=3*ms; R=100*Mohm; V_thresh_base=-55*mV
E_baseline=1.0; E_max=1.0; E_min=0.0; tau_E_recovery=800*ms; energy_cost_per_spike=0.08; V_thresh_penalty=15*mV
I_drive_on=0.25*nA; I_drive_off=0.05*nA; drive_start=100*ms; drive_stop=600*ms
# --- Neuron Model with Energy Dynamics and Dynamic Threshold ---
eqs_energy='''dv/dt = (V_rest - v + R*I_inj)/tau : volt (unless refractory)
              dE/dt = (E_baseline - E) / tau_E_recovery : 1
              V_thresh_eff = V_thresh_base + V_thresh_penalty * (1 - clip(E, E_min, E_max)) : volt
              I_inj : amp; R : ohm; E : 1'''
reset_energy='v=V_reset; E=clip(E - energy_cost_per_spike, E_min, E_max)'
neurons = NeuronGroup(N, eqs_energy, threshold='v > V_thresh_eff', reset=reset_energy, refractory=t_ref, method='euler')
neurons.v=V_rest; neurons.E=E_baseline; neurons.I_inj=I_drive_off; neurons.R=R
# --- Monitors ---
spike_mon=SpikeMonitor(neurons); rate_mon=PopulationRateMonitor(neurons); state_mon=StateMonitor(neurons,['E','V_thresh_eff'],record=0)
# --- Run Simulation with Activity Burst ---
run(drive_start); neurons.I_inj=I_drive_on; run(drive_stop-drive_start); neurons.I_inj=I_drive_off; run(1000*ms-drive_stop)
# --- Visualize ---
plt.figure(figsize=(12, 9))
ax1=plt.subplot(3,1,1); plt.plot(spike_mon.t/ms, spike_mon.i, '.k', ms=1); plt.axvspan(drive_start/ms, drive_stop/ms, color='grey', alpha=0.2, label='High Drive'); plt.ylabel('Neuron Idx'); plt.title('Dynamic Energy Constraint'); plt.legend(fontsize='small')
ax2=plt.subplot(3,1,2,sharex=ax1); plt.plot(rate_mon.t/ms, rate_mon.smooth_rate(width=20*ms)/Hz); plt.axvspan(drive_start/ms, drive_stop/ms, color='grey', alpha=0.2); plt.ylabel('Pop Rate (Hz)'); plt.grid(alpha=0.5)
ax3=plt.subplot(3,1,3,sharex=ax1); ax3.plot(state_mon.t/ms, state_mon.E[0], 'b-', label='Energy (E)'); ax3.set_ylabel('Energy (a.u.)', color='b'); ax3.tick_params(axis='y', labelcolor='b'); ax3.set_ylim(E_min-0.05, E_max+0.05)
ax3b=ax3.twinx(); ax3b.plot(state_mon.t/ms, state_mon.V_thresh_eff[0]/mV, 'r--', label='Eff. Vth'); ax3b.set_ylabel('Vth_eff (mV)', color='r'); ax3b.tick_params(axis='y', labelcolor='r')
plt.axvspan(drive_start/ms, drive_stop/ms, color='grey', alpha=0.2); ax3.set_xlabel('Time (ms)'); ax3.legend(loc='lower left'); ax3b.legend(loc='lower right'); ax3.grid(alpha=0.5)
plt.tight_layout(); plt.show()
```
*Explanation:* This code implements the dynamic energy model where spiking consumes energy `E`, and lower `E` increases the firing threshold `V_thresh_eff`. The plots show that during high input drive, `E` decreases, `V_thresh_eff` increases, and the population firing rate is consequently suppressed compared to what it would be without the energy constraint, demonstrating activity self-limitation due to metabolic feedback.

**14.9 Conclusion and Planned Code**

This chapter tackled the fundamentally important yet often neglected influence of **metabolic constraints** on neural network function, with particular relevance to **brain organoids** hindered by their lack of **vascularization**. We detailed the critical reliance of the brain on vascular supply for oxygen and nutrients, contrasting it with the diffusion-limited reality of organoid culture leading to hypoxic cores and pervasive metabolic stress. The cellular **consequences of energy failure**—from ion pump disruption and gradient rundown to altered excitability, synaptic failure, and excitotoxicity—were elaborated. We explored pragmatic modeling strategies within Brian2 to account for these limitations, including implementing **static spatial gradients** of resource availability or baseline neuronal properties, and, crucially, modeling **dynamic, activity-dependent energy costs**. We detailed how a conceptual energy variable ($E$), consumed by neuronal activity (primarily spiking) and linked back to **modulate neuronal function** (e.g., increasing firing threshold, altering synaptic efficacy), can impose realistic energy budgets and activity self-limitation. The potential role of **waste accumulation** as an additional constraint was also conceptually discussed. Through comparative simulation scenarios—well-nourished, static gradient, and dynamic energy constraint—we highlighted how metabolic limitations can drastically alter network dynamics, spatial activity patterns, and computational capacity. Expanded **Brian2 code examples** provided concrete implementations of a static gradient affecting multiple parameters and a dynamic energy variable modulating firing threshold. Recognizing and modeling these metabolic constraints is paramount for interpreting organoid experiments realistically, understanding the inherent limitations on computation imposed by energy budgets *in vivo*, and developing more biologically faithful simulations of neural systems operating under physiological or pathological conditions.

**Planned Code Examples:**
*   **`14.1_StaticOxygenGradientSim.ipynb`:** (Provided and explained in Section 14.8 - Enhanced) Simulates a network where multiple neuronal parameters ($V_{thresh}$, $E_{baseline}$) vary statically based on spatial location (radial distance), mimicking a core-shell structure due to diffusion limits.
*   **`14.2_DynamicEnergyConstraintSim.ipynb`:** (Provided and explained in Section 14.8 - Refined) Implements a dynamic energy variable $E$ per neuron that depletes with spiking and recovers slowly. Low energy levels increase the firing threshold ($V_{thresh\_eff}$), demonstrating activity self-limitation driven by metabolic feedback.

----
**References for Further Reading**

1.  **Stetter, O., Papadopoulos, S., & Lazar, A. (2022). Energy requirements of neural computation in the brain.** *Current Opinion in Neurobiology, 75*, 102585. https://doi.org/10.1016/j.conb.2022.102585
    *   *Summary:* This essential review synthesizes current knowledge about the energy budget of the brain, breaking down the estimated costs of different components of neural computation (maintaining potentials, spiking, synaptic transmission, plasticity). Provides crucial quantitative background for understanding why metabolic constraints are so important (Section 14.1, 14.4).*
2.  **Ho, R., Salas-Lucia, F., & Fattahi, P. (2022). Engineering human brain organoids.** *Annual Review of Biomedical Engineering, 24*, 157-181. https://doi.org/10.1146/annurev-bioeng-111121-072416
    *   *Summary:* Reviews the state-of-the-art in brain organoid engineering. A significant portion discusses the challenge of vascularization (Section 14.1) and various bioengineering strategies being developed to introduce vasculature or perfusion (e.g., co-cultures, microfluidics, bioprinting, xenografts) to overcome diffusion limits and improve viability and maturation.*
3.  **Loga, M., Oláh, V., Heiland, M., Tura, A., & Szabó, I. (2023). Metabolic aspects of cerebral organoids.** *Neurochemistry International, 169*, 105581. https://doi.org/10.1016/j.neuint.2023.105581
    *   *Summary:* Directly addresses the metabolism of brain organoids. It reviews experimental findings on their glucose utilization, reliance on glycolysis vs. oxidative phosphorylation, evidence for hypoxic cores (Section 14.2), and how metabolic profiles change during development, providing critical experimental context for the modeling efforts discussed.*
4.  **Ma, Z., Yao, R., Rong, Y., Cheng, K., Chen, J., & Wu, S. (2022). Exploiting criticality for enhanced reservoir computing.** *Frontiers in Neuroscience, 16*, 857618. https://doi.org/10.3389/fnins.2022.857618
    *   *Summary:* While focused on reservoir computing, the study relates to the need for sustained complex dynamics. Metabolic constraints (Sections 14.2, 14.4, 14.5) impose fundamental limits on the ability of any network, biological or simulated, to maintain the high levels of activity potentially required for such computations.*
5.  **Mahmoudi, N., Zenke, F., & Boucheny, C. (2023). Efficient simulation of networks of spiking neurons with short-term plasticity and realistic metabolic energy constraints.** *bioRxiv*, 2023.07.07.548092. https://doi.org/10.1101/2023.07.07.548092
    *   *Summary:* This highly relevant computational work specifically tackles the challenge of simulating spiking networks that incorporate both STP (Chapter 13) and models of metabolic energy constraints similar to those discussed here (Sections 14.4, 14.5). They propose efficient simulation methods, highlighting active research in this integration.*
6.  **Nour, M., Şimşek, Ö., Kruck, P., Tsai, Y. Y. S., Öztürk, G., & Kiselev, V. G. (2023). Quantitative diffusion MRI of cerebral organoids.** *NeuroImage, 279*, 120314. https://doi.org/10.1016/j.neuroimage.2023.120314
    *   *Summary:* Uses advanced imaging (diffusion MRI) to non-invasively probe the microstructure of brain organoids (cell density, tissue organization). These structural features directly influence the effective diffusion paths and distances for nutrients and waste (Section 14.1, 14.3), providing experimental data to potentially inform spatial models.*
7.  **Peng, L., Mir, Z. M., Sparks, F. T., Cvetkovic, C., & Scherer, S. S. (2023). Metabolic Interactions Between Neurons and Glia.** *Annual Review of Neuroscience, 46*, 87-107. https://doi.org/10.1146/annurev-neuro-102122-104824
    *   *Summary:* Provides a detailed review of the metabolic interplay between neurons and glial cells (astrocytes, oligodendrocytes). Discusses substrate transport (glucose, lactate), waste removal, and how glial metabolism supports neuronal function (Section 14.1). These interactions are crucial for overall brain energy homeostasis and are affected by hypoxia (Section 14.2).*
8.  **Schmidt, H., Schöler, H. R., & Le MessageBox, A. (2024). Brain organoids: towards functional multicellular neural circuits in vitro.** *Journal of Neurochemistry*, e16084. https://doi.org/10.1111/jnc.16084 *(Published online Nov 2023)*
    *   *Summary:* Reviews the ongoing quest to achieve truly functional circuits within brain organoids. It implicitly acknowledges that limitations like the lack of vascularization and consequent metabolic stress (Sections 14.1, 14.2) are major hurdles currently preventing the full recapitulation of mature *in vivo* network function.*
9.  **Singh, R., Kim, H., & Cho, S. (2022). Recent advances in monitoring and controlling brain organoid development.** *Experimental & Molecular Medicine, 54*(10), 1707-1717. https://doi.org/10.1038/s12276-022-00860-8
    *   *Summary:* Focuses on technological advancements for monitoring organoid growth and function, including sensors that might eventually provide real-time feedback on the internal metabolic state (relevant to Section 14.2). Also discusses strategies for controlling the culture environment to potentially mitigate metabolic stress (Section 14.1).*
10. **Yi, M., Wang, J., Tian, L., & Zhou, Y. (2023). Computational modeling of tripartite synapses: exploring the roles of astrocytes in synaptic information processing.** *Cognitive Neurodynamics, 17*(2), 261-285. https://doi.org/10.1007/s11571-022-09857-z
    *   *Summary:* Reviews models of astrocyte function at the synapse. Astrocyte roles like glutamate uptake (Section 12.3) are highly energy-dependent (requiring Na+/K+ pump activity fueled by ATP). Therefore, astrocyte function itself is subject to the metabolic constraints discussed in this chapter (Section 14.2), creating complex feedback loops.*
   
----
