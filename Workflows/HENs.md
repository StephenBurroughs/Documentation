---
id: HENsReworks
title: Beat the Heat (Exchanger Networks)
desc: ""
updated: 1765764519981
created: 1654223767390
nav_exclude: false
---

Current platform workflows support a one directional flow of information from flowsheets to stream data projects, which are then used to perform pinch analysis and generate grid diagrams. This is fine for producing insights, but is limited in that it does not allow the user to compare or optimise different HEN designs and integrate them back into the broader flowsheet structure. The flowsheet itself is also limited in that the full flowsheet view is suboptimal for interpreting details of the HEN without supurflous intermediate operations. 

## Utilities in the flowsheet
In our current flowsheet implementation, we have heaters, coolers, heat exchangers, and headers. Essentially, we have the bones for everything that is required to show utility systems, but the way in which we do this is suboptiomal. Firstly, heaters and coolers do not actually exist. These operations in practice are heat exchangers, being supplied by a steam/water utility (usually through a header or ring main) to achieve heating/cooling behaviours on a given process stream. Secondly, displaying headers and their associated connections on the general flowsheet is horrible and results in a disgusting pile of spaghetti that even the laziest Italian would be ashamed of.

So, how can we address this? **Separate flowsheets**.

Previously, we have had several long discussions around how to view and interpret steam systems. I think it's time that we implement some of these ideas.
- We should have two flowsheets, one for the flowsheet and one for utilities.
- We should be able to create supply/demand links between the two and sync data across each.

How does this look in practice?

- We have a seperate flowsheet view in which we have our steam system, defined using headers. 
    - Each header represents one steam system utility. headers have defined inlet properties which sets their utility values (eg temperature and pressure), but have their inlet flow constrained by their outlet demand. By default, headers have **ONLY** a condensate and vent outlet. 
    
- Thermo operations should appear in each flowsheet view, as should mixers/splitters. 
    - Other superflous operations are not shown on the utility system flowsheet.
    - Streams also require some changes here. We don't want to show intermediate operations (such as pressure changes), but such operations naturally have an impact on stream conditions. There are a couple of options for addressing this:
        - We show a single stream between the operations, but this stream only has a limited set of properties displayed, such as enthalpy, temperature, flow rate.
        - We don't show streams at all, just connection lines. The lefthand summary panel for operations then holds information about the inlet and outlet conditions for the object in question.
- Heaters and coolers should be modified. First, heaters and coolers should have an option to define exchanger properties, being area and HTC. They should also have an optional connection field of "supplier". This should be a dropdown field that has headers as the entry options. Upon selecting a header, several things happen:
    - A new outlet is created for the header in question, with its outlet flow constrained by the energy demand of the heater/cooler.
    - On the utility flowsheet, the heater/cooler in question should be displayed as a heat exchanger connected to this outlet.
    - A new connection is drawn on the main flowsheet to the operation in question. This connection should be clearly distinct in style, similar to that of logic blocks. 
    - It should be possible to navigate between flowsheets from the unit operation summary, or possibly from the flowsheet itself somehow.
- Packing is synced across both flowsheets. 

For heaters/coolers where a supplier is not selected, this is solved as usual and the heat flow property is treated as an external energy demand. On the utility flowsheet, these are displayed as heaters and coolers rather than exchangers.

Optionally, this could be extended somewhat...
- Rather than just focussing on separate flowsheet views, headers could also have two graphic objects, one each for utility and process flowsheets. This means that we can show both abstractions as outlined previously, but also gives us the option to show a unified view, wherein we highlight hot/cold streams and their steam system supply, while all intermediate objects are present but greyed out (or alternatively all header interactions are greyed out and the main processes are interactive). This means that the user can view just utilities, just processes, or a combination of the two with a greter level 

### What are the benefits to this?
First, it makes it much easier to deal with utility systems within the flowsheet. It is also more realistic than our current approach of heaters and coolers as standalone operations. The greatest strength however, is that it provides us with some very useful data and interfaces for dealing with other areas of HENS.

## Stream data and grid diagrams.
Current implementations for grid diagram HEN representations are also somewhat limited in that they are static in nature - ie, streams and exchanger pairs are extracted from stream data and cannot be modified in situ. The result of this is that our HEN functionality under the status quo is limited to solely representing the network as it stands and fails to account for any modifications that the user may wish to explore.

### What can we do, currently?
Currently, we can generate a HEN grid diagram from stream data. Users can interact with this in the following ways:
- Users can see a summary of the network and its constituent streams in the lefthand summary panel.
- Users can rearrange streams.
- Users can expand and collapse streams:
    - Expanded streams show substreams within a given higher level process, as well as linearised segments (a function to smooth the heat transfer across phase change regions) and their respective exchangers/heaters/coolers.
    - Collapsed streams show a single high level stream with all heaters/coolers/exchangers.
- Users can see the pinch point on the GD, as well as temperature changes.
- Upon hovering, users can see further information:
    - streams/heaters/coolers show temperature and enthalpy changes.
    - heat exchangers also show area.
- Upon clicking on a heater/cooler/exchanger, users are given a summary of this information in the lefthand summary panel.

There are some bugs with existing implementations.
- GD streams are labelled (but these labels don't actually show all of the stream segments)
- Hovering over collapsed streams only displays information about the first segment.
- Adding new stream data entries in the stream data table is broken.
- Cooling enthalpies are not propertly captured.

### What **Should** we be able to do?
There are two areas to consider here. Design and retrofit.
From the design perspective, this basically means we want to integrate Open-HENS - Keegan's PhD work - for HEN synthesis. This entails:
- A new factory for Open-HENS
- An Open-HENS service.
- Some revision around how we handle grid diagrams - essentially we want to be able to capture multiple stream data projects (our database model) for a given platform project such that we can present multiple possible HEN structures generated as possible solution from Open-HENs.

Retrofit naturally follows on from this, but requires some further consideration around implementation. There are two general classes of retrofit approach:
- Automatic generation of retrofit HENS. This approach needs to consider:
    - reuse of existing exchangers with defined properties.
    - fixed heat exchanger matches.
    - tainted matches.
    - integration of new exchangers with defined properties.
- Manual modifications to HENs. This approach needs to consider:
    - changes to existing exchanger placement or properties.
    - placement of new exchangers.
    - Removal of existing exchhangers.
- All of this is also reliant on a change to the current stream data project such that we can capture and compare multiple HEN structures.

It's worth noting that tying in the earlier notions of utility system flowsheets is quite useful here. If we have the ability to link in steam and water systems and define specifics around heating/cooling utility heat exchangers, then this gives us a greater degree of information that can be used within HEN visualisation and manipulation. Without this information, we are instead limited to defining heaters and coolers within HENs based on demand, rather than providing more specific information such as sizing or utility supply.

### Backend Requirements.
There are two challenges from the backend perspective for retrofit analysis. In the first instance, modifications to Open-HENs must be made such that the solving algorithm can take in a set of initial conditions that must be maintained. In the second instance, we need a new factory api to build a HEN model and solve it to reflect the modifications made to the system. This factory might create a simplified flowsheet and reuse the existing IDAES solving service, or it may make sense to use a lower fidelity system such as synheat. *A point to consider here is that if we want to use IDAES to capture higher fidelity models (rather than a simple mass energy balance) then it is also necessary to have users specifiy stream composition when adding new streams within the stream data table*.

### Frontend Requirements.
There are several requirements for frontend workflows that stem from these changes. The first is the ability to capture multiple stream data projects. One approach for how we might do this could be to change the functionality of the lefthand sidebar when currently accessing pinch/hen analysis. Rather than opening the pinch tab when clicking the CC icon (as is the status quo behaviour), we could instead trigger the opening of a new side panel that has a list of stream data projects, with a button to create a new project. By default, this list should be empty. Upon creating or opening a project, the currently implemented pinch tab should open with that project. An initial implementation might allow for this behaviour, as well as the copying of existing projects and the exporting of modified projects as new projects. While this workflow is a little irritating when switching between a flowsheet and pinch view, I think that we should encourange a multiple browser tab approach which alleviates this pain point.

The second area of interest here is the GD view. Currently, the buttons to extract stream data, import from OpenPinch, and run PA are all present in each tab of the stream data UI. For all tabs other than stream data, these buttons are redunant. In the HEN Design tab, we should have two buttons: Modify and Optimise.

Upon clicking modify, this button should disappear. A new "Solve" button should appear, as well as some cancle option to go back to the base case. the user should be able to drag exchangers to change their stream pairing. Clicking on exchangers/heaters/coolers should also bring up a writable version of their properties. Additionally, users should be able to add and define new operations. If all objects in the HEN are adequately defined, the solve button should be clickable. If the HEN is in a solved state, the optimise button should be clickable. 

When clicking the optimise button, a workflow should open (possibly in the right hand side). This workflow should have an option to fix initial conditions. If this option is selected, the user should be presented with a list of all existing exchangers and be able to specify which properties should be treated as unchangeable. These properties are: 
- Whether the exchanger must be used.
- Hot and cold side connections
- Heat Exchange Area
- Heat Transfer Coefficient

They should also be given the option to add additional specifications. These include:
- Defined heat exchangers
    - These could have defined areas and HTC, or these could be bounded properties.
- Bounds to the number of new exchangers.
- Required or illegal stream pairs.

Results should be presented in a tabbed fashion, such that a user can compare the resulting generated networks. Each of these resulting networks should also be able to be exported to a new project for further modification.


## Integration with Flowsheeting Capabilities
So far, we have a one directional communication from the flowsheet to stream data. We want to make this bidirectional. **THIS SHOULD NOT BE AUTOMATIC IN EITHER DIRECTION**. To update from a flowsheet, the user should have to press the button to extract new stream data. To update the flowsheet from the GD, we require a new workflow.

Upon modifying and solving/optimising a GD, there should be options to preview the flowsheet and export changes. When previewing, the user should be given a flowsheet view that **only** shows placements of operations (all, not just heaters/coolers/hx) and stream connections. In the case of new operations, placement should be automatic (this doesn't require too much thought right now as it's part of a larger autoarrange challenge). Users should be able to adjust things in this preview, with changes being tracked even when they exit this preview workflow. The user should be able to export from within the preview workflow, or from the GD view. Upon exporting, positions should be updated based on the preview, with properties synced from operations in the GD. It would be very useful to be able to export updated HENs as new flowsheet projects, but this is part of a larger area of discussion around the role of scenarios and projects within the current platform.

