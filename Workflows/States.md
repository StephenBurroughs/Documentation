---
id: FSStateRework
title: States
desc: ''
updated: 1774828088765
created: 1774826755789
nav_order: 3
nav_exclude: false
---
It is not currently possible to sync values within the flowsheet to states across optimisation or multisteady state solves. This issue is slightly different for each of these cases. For multisteady state, it is possible to view comparative values as a graph or table view, but it is not possible to select a state and synchronise the property values in the flowsheet. For optimisation, solving the model changes the state of the flowsheet itself and it is not possible to visually compare the presolve/post solve state at all.

## What's the desired outcome here?

In an ideal world, the user should be able to run an optimisation and compare the results to the previous state of the flowsheet. The user should also be able to revert to this previous state. If we take this to it's natural conclusion, the user should be able to make modifications to bounds or goal functions within an optimisation scenario, comparing the performance of some given metric across each and selecting the state to view as the flowsheet itself. For multisteady state, the user should be able to sync the state of the flowsheet with a given row in the table of states.

## How can this be achieved?

Basically, we need a way to store property values across solves and associate them with a state. When getting results from MSS, all properties for the given state should be updated and stored. When solving an optimisation, a new state should be generated and the solution should be stored. This means that we need a way to visually track solution states within the optimisation workflow, as well as probably a higher level workflow to see states of the flowsheet across all scenarios. 

## What are some challenges here?
How do we handle situations where the flowsheet has changed structurally? Also, how do we deal with identifying and modifying the "base state"?

Let's say that we have a flowsheet with one base state, S1. If we make modifications, by default things in S1 get updated. Now, this makes sense as we don't want to be storing every single state upon a user making some change, but there are cases where a user may want to revert to some previous state and making flowsheet copies isn't really an ideal workflow. We should thus give the user the ability to manually tag and save a state within the flowsheet itself.

The case for structural changes is similar. Basically, we should let users save layouts. If a user has saved states and makes a change to the base layout, L1, then we should think about how we deal with this. If they've made major modifications, then it doesn't really make sense to keep any previous states as they're no longer relevant. On the other hand, if they've just added in a new process somewhere and it is independent of the previously defined structure, then maybe it makes sense to sync the first solved version of this new process across all other states? We need a way to identify which of these approaches is appropriate.

Now, one approach to this would be to simply save the full state of the flowsheet rather than indexing property values. If we do this, then structural changes don't matter as we can just go back to what it looked like before. This is pretty horrible though as we're storing far more information than is really necessary in most cases, plus it's pretty bad to be storing every multi steady state solve across every version of the flowsheet structure if the old ones aren't really needed anymore. A better option is probably to be able to track versions of the flowsheet structure instead (eg multiple flowsheets for a given project). This means that a project would have a set of flowsheets (created manually by the user), with each flowsheet then having a set of states. This could be extended to allow the user to actually just define different flowsheets entirely under one project (eg rather than having multiple abstracted processes in one flowsheet, they could be separate flowsheets and the user could defined expressions/constraints between them and solve them as one) but this could get pretty messy, so maybe it's better to just have it as independent flowsheets under one given project.

## So, where to from here?
Well, there are a couple of things to consider. First, are these outlined approaches a good way to move forward? Secondly, where should these workflows be held and what should they look like? Thirdly, what does this mean for P-Graph?