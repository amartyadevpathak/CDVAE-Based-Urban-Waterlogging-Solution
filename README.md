# Generative Discovery of High-Strain-Tolerant Poroelastic Binders for Urban Waterlogging Alleviation

*An independent computational materials-science project by Amartya Dev Pathak*

We started off with a problem that is very much in evidence in Ahmedabad: during periods of heavy monsoon rain the roads will be waterlogged for long periods. It's not always feasible to go out and improve the drainage infrastructure everywhere so we wanted to look at a different approach. Could we design a road which would allow more water to pass through it without in any way compromising the structural strength that you need to support the traffic?

The fundamental issue is a trade-off. More empty space in the road means better water permeability, but as the area of contact between the aggregates decreases the stress on the small binder bridges increases. The binder therefore has to take up the extra stress created by the increased porosity of the road whilst still permitting water to pass through the interconnected pores.

My approach is to do it at two scales. First translate the rainfall and traffic requirements into the mechanical properties that the binder must have. Then use generative machine learning to search for a crystalline material that would satisfy those requirements.

The central idea is that the water is going to pass through the pores that are connected in the concrete rather than through the atomic structure of the binder crystal. The crystal is being looked at as the load bearing element in a more porous road structure.

## What I set out to do

My main objective was the computational discovery of a poroelastic road-binder matrix which would be able to support a higher level of road porosity and water permeability whilst still having adequate stress resistance for the traffic loads.

A second objective was to create a continuous, high-dimensional latent space representation of inorganic crystals using generative machine learning and to explore the physical and thermodynamic plausibility of high strength crystal structures. This was to act as a proof of principle for future work which would incorporate commercial viability and elemental abundance.

This is a computational proof of concept. It is not a claim that a finished road material has in fact been synthesised or deployed.

## How the project works

The pipeline links together two different scales of the problem.

### 1. Macro-scale feasibility: translating a road problem into material requirements

Before we even got to the actual crystal generation we had to work out what the road itself would require. That part of the project looked at the relationship between rainfall, hydraulic permeability, porosity, aggregate size, sand content and mechanical strength.

I used the fluid flow and porous materials relationships (Darcy's law and the Kozeny–Carman equation) to link the required drainage behaviour to the pore structure. The concrete side model uses porosity, sand rate and voids in mineral aggregate (VMA) together with some experimental pervious concrete data to establish the required permeability and strength.

I initially thought about an XGBoost model but the experimental dataset we had was pretty small and I was worried about overfitting. Went down the route of polynomial ridge regression instead, using least squares fitting and feature scaling to model the measured relationships. Also constrained the optimisation to the range of the experimental data rather than letting it go and extrapolate freely beyond that.

The output from this stage is a set of mechanical requirements for the binder. In effect it's giving the crystal generation stage a target to aim at rather than asking a generative model to work out what a road needs.

### 2. Generative crystal discovery: searching for a stronger binder

For the crystal generation stage I used a Crystal Diffusion Variational Autoencoder (CDVAE) with the MP-20 crystal dataset from the Materials Project.

I also worked on a property prediction component using an MLP (Multi-Layer Perceptron) oracle. The project notes describe the combining of the bulk and shear moduli into a single strength index so that the model could learn a mechanical strength objective. The idea was to use the gradients to search the CDVAE's latent space for regions associated with stronger crystal structures and then decode the selected latent coordinates to produce candidate crystal structures.

This wasn't just about generating structures. I had to go through the entire model configuration, training behaviour, checkpointing, property scaling and the gradient flow through the optimisation code. One issue we came across was that an oracle configured with just one output could predict the strength index but was completely unable to predict the formation energy. So a high strength index on its own wasn't sufficient to establish that a given structure was chemically and thermodynamically viable.

### 3. Crystal verification: checking what the generator produces

A gen model can produce structures that look very promising in terms of its objective function, but which are not necessarily physically meaningful at all. I therefore treated verification as a separate and absolutely vital stage in the process.

The validation work includes:

- Geometric checks: screening the generated structures for unreasonably short interatomic distances and other such basic structural issues.

- Chemical checks: oxidation state and electroneutrality checks (including SMACT based checks informed by Pauling's rules).

- Structural relaxation and thermodynamic screening: using atomistic machine learning tools and energy above hull comparisons to determine whether the candidate structures are viable or extremely unstable.

- Mechanical evaluation: finite strain analysis, derive an elastic stiffness tensor from the energy changes, and use Voigt–Reuss–Hill (VRH) averages to get the polycrystalline bulk and shear moduli.

The VRH calculation is important because a real binder wouldn't be a single perfectly aligned single crystal. It would be a polycrystal with grains of various orientations. The relevant property is therefore the average response over all directions rather than the stiffness along a single crystallographic axis.

## Computational results

The project synopsis reports a computationally selected candidate with the following properties:

| Property | Reported value |
|---|---:|
| Energy above the convex hull | 72 meV/atom |
| Shear modulus | 70.5 GPa |
| Bulk modulus | 194.28 GPa |
| Longitudinal elastic modulus (VRH-based estimate) | 288.28 GPa |

These are computational results not measurements from a physically synthesised sample. The EAH value was interpreted in the project as being in a metastability window which was worth looking at further; it should by no means be taken as evidence of experimental synthesizability.

The wider design target that's been explored in the synopsis is a high porosity concrete architecture (including the 35% porosity case). The candidate binder is therefore a theoretical material design direction rather than a product that's ready for the road. You'd need physical synthesis, your own calculations and your own mechanical testing before you could make any claims about its actual performance.

## What I learned while building it

A big part of this project was working out why an approach that looked perfectly sensible on paper wasn't in fact working as expected in the code.

We had all sorts of package and environment problems, memory limits, training got interrupted, there were model instability issues and bugs in the latent space optimisation path. We had to question results that looked rather too good. At one point the predicted stiffness didn't change at all during optimisation and tracing the computation graph revealed a gradient flow issue. In another area of the process we were doing stiffness optimisation and we ended up with candidates that had very good predicted moduli but were very poor in terms of their chemical/thermodynamic properties.

That completely changed the way I did the project. I stopped thinking of the generative model's score as the be-all and end-all. You had to go through an independent set of geometric, chemical, thermodynamic and mechanical checks on any candidate. That project really taught me that just because something looks and seems wrong doesn't mean it is. I'd stopped the CDVAE training when I saw a big change in the loss, but looking at the reconstruction and KL divergence behaviour told me the model was actually exploring its latent space. At the same time you can't just take a result that looks absolutely fantastic on the face of it as being correct. You have to understand what the model is doing before you can accept its output.

## Limitations and next steps

There are important gaps between a computational candidate and a usable construction material:

- The crystal structures generated have not in fact been made or tested in the lab.

- The macro-scale regression is based on a very limited experimental design. The early porosity data range that I have in my notes is 14–26%, so a 35% porosity design target cannot be considered validated by that dataset and requires further experimental backing.

- The first strength prediction setup used a single output oracle. The strength score on its own wasn't sufficient to establish formation energy or thermodynamic stability. That has to be borne in mind when interpreting any multi-objective optimisation and independent verification that might be performed.

- elemental abundance, cost, scalable synthesis, long term durability and real world road performance testing are all still open questions.

These are limitations of the project, not things I'm trying to hide. They're the very things that would need to be addressed before the work could go from computational discovery to practical materials engineering.

## Explore the research and implementation

I'd recommend reading the research paper first for the full mathematical formulation, the assumptions, the datasets, the model design, the equations, the validation procedure and the detailed results. This README is just an introduction and doesn't in any way show all the decisions or the technical details behind the work.

- Read the full [Research Paper](./RESEARCH_PAPER_AMARTYA.docx).
- Explore the source code in this repository — the implementation is the best way to follow how the macro-scale feasibility model, CDVAE workflow, latent-space optimisation and crystal-verification steps fit together.
- A useful reading order for the code is: start with the macro-scale feasibility model, move to the CDVAE training and property-prediction components, then examine the latent-space optimiser and the crystal-verification scripts.

## A note on tools and prior work
This project builds on existing datasets, open-source models and scientific software. It does not claim to have created CDVAE, the Materials Project, Matbench, CHGNet, MEGNet or SMACT. The work here is in bringing these components together for this particular road-material problem, building the problem-specific modelling and optimisation workflow, and examining the generated candidates through additional physical and chemical constraints. The research paper should be consulted for the full references and acknowledgements.

This project is an ongoing computational investigation. The purpose of publishing the code and research is to make the approach, assumptions, results and limitations available for others to experiment and research, to provide practical solutions by huilding on this project to the global waterlogging problem.
