# welleng
[![Open Source Love svg2](https://badges.frapsoft.com/os/v2/open-source.svg?v=103)](https://github.com/pro-well-plan/pwptemp/blob/master/LICENSE.md)
[![PyPI version](https://badge.fury.io/py/welleng.svg)](https://badge.fury.io/py/welleng)
[![Downloads](https://static.pepy.tech/personalized-badge/welleng?period=total&units=international_system&left_color=grey&right_color=orange&left_text=Downloads)](https://pepy.tech/project/welleng)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

[welleng] aspires to be a collection of useful tools for Wells/Drilling Engineers, kicking off with a range of well trajectory analysis tools.

  - Generate survey listings and interpolation with minimum curvature
  - Calculate well bore uncertainty data (currently utilizing the [ISCWSA] MWD Rev4 model)
  - Calculate well bore clearance and Separation Factors (SF)
    - standard [ISCWSA] method
    - new mesh based method using the [Flexible Collision Library]
## New Features!
  - **Import from Landmark .wbp files:** using the `exchange.wbp` module it's now possible to import .wbp files exported from Landmark's COMPASS or DecisionSpace software.
  ```
import welleng as we

wp = we.exchange.wbp.load(demo.wbp) # import file
survey = we.exchange.wbp.wbp_to_survey(wp, step=30) # convert to survey
mesh = we.mesh.WellMesh(survey, method='circle') # convert to mesh
we.visual.plot(m.mesh) # plot the mesh
  ```
  - **Export to .wbp files *(experiemental)*:** using the `exchange.wbp` module, it's possible to convert a planned survey file into a list of turn points that can be exported to a .wbp file.
  ```
import welleng as we

wp = we.exchange.wbp.WellPlan(survey) # convert Survey to WellPlan object
doc = we.exchange.wbp.export(wp) # create a .wbp document
we.exchange.wbp.save_to_file(doc, f"demo.wbp") # save the document to file
  ```
  - **Well Path Creation:** the addition of the `connector` module enables drilling well paths to be created simply by providing start and end locations (with some vector data like inclination and azimuth). No need to indicate *how* to connect the points, the module will figure that out itself.
  - **Fast visualization of well trajectory meshes:** addition of the `visual` module for quick and simple viewing and QAQC of well meshes.
  - **Mesh Based Collision Detection:** the current method for determining the Separation Factor between wells is constrained by the frequency and location of survey stations or necessitates interpolation of survey stations in order to determine if Anti-Collision Rules have been violated. Meshing the well bore interpolates between survey stations and as such is a more reliable method for identifying potential well bore collisions, especially wth more sparse data sets.
  - More coming soon!
## Tech
[welleng] uses a number of open source projects to work properly:

* [trimesh] - awesome library for loading and using triangular meshes
* [numpy] - the fundamental package for scientific computing with Python
* [scipy] - a Python-based ecosystem of open-source software for mathematics, science, and engineering
* [vedo] - a python module for scientific visualization, analysis of 3D objects and point clouds based on VTK.
## Installation
[welleng] requires [trimesh], [numpy] and [scipy] to run. Other libraries are optional depending on usage and to get [python-fcl] running on which [trimesh] is built may require some additional installations. Other than that, it should be an easy pip install to get up and running with welleng and the minimum dependencies.

```
pip install welleng
```
For developers, the repository can be cloned and locally installed in your GitHub directory via your preferred Python env.
```
git clone https://github.com/jonnymaserati/welleng.git
cd welleng
pip install -e .
```
Make sure you include that `.` in the final line (it's not a typo) as this ensures that any changes to your development version are immediately implemented on save.
## Quick Start
Here's an example using `welleng` to construct a couple of simple well trajectories with the `connector` module, creating survey listings for the wells with well bore uncertainty data, using these surveys to create well bore meshes and finally printing the results and plotting the meshes with the closest lines and SF data.

```
import welleng as we
import numpy as np
from tabulate import tabulate

# construct simple well paths
print("Constructing wells...")
connector_reference = we.connector.Connector(
    pos1=[0,0,0],
    inc1=0,
    azi1=0,
    pos2=[-100,0,2000.],
    inc2=90,
    azi2=60,
).survey(step=50)

connector_offset = we.connector.Connector(
    pos1=[0,0,0],
    inc1=0,
    azi1=225,
    pos2=[-280,-600,2000],
    inc2=90,
    azi2=270,
).survey(step=50)

# make a survey objects and calculate the uncertainty covariances
print("Making surveys...")
survey_reference = we.survey.Survey(
    md=connector_reference.md,
    inc=connector_reference.inc_deg,
    azi=connector_reference.azi_deg,
    error_model='ISCWSA_MWD'
)

survey_offset = we.survey.Survey(
    md=connector_offset.md,
    inc=connector_offset.inc_deg,
    azi=connector_offset.azi_deg,
    start_nev=[100,200,0],
    error_model='ISCWSA_MWD'
)

# generate mesh objects of the well paths
print("Generating well meshes...")
mesh_reference = we.mesh.WellMesh(
    survey_reference
)
mesh_offset = we.mesh.WellMesh(
    survey_offset
)

# determine clearances
print("Setting up clearance models...")
c = we.clearance.Clearance(
    survey_reference,
    survey_offset
)

print("Calculating ISCWSA clearance...")
clearance_ISCWSA = we.clearance.ISCWSA(c)

print("Calculating mesh clearance...")
clearance_mesh = we.clearance.MeshClearance(c, sigma=2.445)

# tabulate the Separation Factor results and print them
results = [
    [md, sf0, sf1]
    for md, sf0, sf1
    in zip(c.reference.md, clearance_ISCWSA.SF, clearance_mesh.SF)
]

print("RESULTS\n-------")
print(tabulate(results, headers=['md', 'SF_ISCWSA', 'SF_MESH']))

# get closest lines between wells
lines = we.visual.get_lines(clearance_mesh)

# plot the result
we.visual.plot(
    [mesh_reference.mesh, mesh_offset.mesh], # list of meshes
    names=['reference', 'offset'], # list of names
    colors=['red', 'blue'], # list of colors
    lines=lines
)

print("Done!")
```
This results in a quick, interactive visualization of the well meshes that's great for QAQC. What's interesting about these results is that the ISCWSA method does not explicitly detect a collision in this scenario wheras the mesh method does.

![image](https://user-images.githubusercontent.com/41046859/102106351-c0dd1a00-3e30-11eb-82f0-a0454dfce1c6.png)

For more examples, including how to build a well trajectory by joining up a series of sections created with the `welleng.connector` module (see pic below), check out the [examples].

![image](https://user-images.githubusercontent.com/41046859/102206410-d56ef000-3ecc-11eb-9f1a-b2a6b45fe479.png)

Well trajectory generated by [build_a_well_from_sections.py]

## Todos
 - Add a Target class to see what you're aiming for - **in progress**
 - Export to Landmark's .wbp format so survey listings can be modified in COMPASS - **in progress**
 - Documentation
 - Generate a scene of offset wells to enable fast screening of collision risks (e.g. hundreds of wells in seconds)
 - Well trajectory planning - construct your own trajectories using a range of methods (and of course, including some novel ones) **- DONE!**
 - More error models
 - WebApp for those that just want answers
 - Viewer - a 3D viewer to quickly visualize the data and calculated results **- DONE!**

It's possible to generate data for visualizing well trajectories with [welleng], as can be seen with the rendered scenes below.
![image](https://user-images.githubusercontent.com/41046859/97724026-b78c2e00-1acc-11eb-845d-1220219843a5.png)
ISCWSA Standard Set of Well Paths

![image](https://media-exp1.licdn.com/dms/image/C5612AQEBKagFH_qlqQ/article-inline_image-shrink_1500_2232/0?e=1609977600&v=beta&t=S3C3C_frvUCgKm46Gtat2-Lor7ELGRALcyXbkwZyldM)
Equinor's Volve Wells

The ISCWSA standard set of well paths for evaluating clearance scenarios and Equinor's [volve] wells have been rendered in [blender] above. See the [examples] for the code used to generate the [volve] scene, extracting the data from the [volve] EDM.xml file.

License
----

LGPL v3

Please note the terms of the license. Although this software endeavors to be accurate, it should not be used as is for real wells. If you want a production version or wish to develop this software for a particular application, then please get in touch with [jonnycorcutt], but the intent of this library is to assist development.

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [jonnycorcutt]: <mailto:jonnycorcutt@gmail.com>
   [welleng]: <https://github.com/jonnymaserati/welleng>
   [Flexible Collision Library]: <https://github.com/flexible-collision-library/fcl>
   [trimesh]: <https://github.com/mikedh/trimesh>
   [python-fcl]: <https://github.com/BerkeleyAutomation/python-fcl>
   [vedo]: <https://github.com/marcomusy/vedo>
   [numpy]: <https://numpy.org/>
   [scipy]: <https://www.scipy.org/>
   [examples]: <https://github.com/jonnymaserati/welleng/tree/main/examples>
   [blender]: <https://www.blender.org/>
   [volve]: <https://www.equinor.com/en/how-and-why/digitalisation-in-our-dna/volve-field-data-village-download.html>
   [ISCWSA]: <https://www.iscwsa.net/>
   [build_a_well_from_sections.py]: <https://github.com/jonnymaserati/welleng/tree/main/examples/build_a_well_from_sections.py>
