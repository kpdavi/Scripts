# NR427 Short Project - Automated Mapping Using the ArcPy Mapping Module

For my short project, I used the ArcPy Mapping module to create
a map of potentially related USGS Breeding Bird Survey routes
within each of 46 states (the lower continental US excepting
California and Texas). 

First, I created a new project in ArcGIS Pro with a blank map. 

![project](blank_Pro_project.png)

I inserted a Layout into the project and added a mapframe to the layout. I also added a
north arrow, scale bar, blank legend, and text box to the layout:

![layout](layout_Pro_project.png)

I manually changed the background color of the legend to white so that the symbols in the
legend would be visible (rather than blending in with the basemap).

## Workflow For Creating a Single Map

First, I added the focal layer to the map. The focal layer on which I tested the code was
a shapefile of the Breeding Bird Survey routes in Colorado that were likely related 
based on a test distance of 10 miles.

```python
import arcpy.mp as mp  # import the ArcPy Mapping Module

# Create the project object
aprx = mp.ArcGISProject("CURRENT")

# Create a variable to represent the path to the data
datapath = r"E:\Programming in GIS\NR 426 - Programming in GIS I\Final project\Data\States\17\BBS_routes_1966_2017_17_likely_related_10_miles_no_duplicates.shp"

# Reference the current map, then add data to it
map = aprx.listMaps("Map")[0]

map.addDataFromPath(datapath)
```

I then activated the mapframe in the layout, and within the activated mapframe, I 
manually zoomed to the extent of the focal layer (i.e., in ArcGIS Pro, I right-clicked 
on the focal layer in the Contents pane, then selected "Zoom to Layer"). I found I had 
to do this, otherwise the maps for subsequent states wouldn't be zoomed to the 
extent I wanted them to be. I also used the code below to ensure each map's extent
represented the extent I wanted (although I found this code was not effective without
manually zooming to the layer the first time the mapframe was set up, as I describe above.)

```python
# Assign the first (and only, in this project) layout to an object
lyt = aprx.listLayouts("Layout")[0]

# Assign the layers of the map to an object
lyrs = map.listLayers()

# Assign the focal layer to a new object
zoomExtent = lyrs[0]

# Assign the mapframe to an object
mf = lyt.listElements("MAPFRAME_ELEMENT")[0]

# Use the focal layer as the extent for panning
ext = mf.getLayerExtent(zoomExtent,True)
mf.panToExtent(ext)
```

I wanted to symbolize potentially related routes by the two-digit portion of the
route number (the tens and hundreds digits; represented by the 'Pair_rts' field) 
that they share in common. So, for example, if there were multiple routes ending in 
08, I wanted those to be one color, and if there were routes ending in 10, I wanted 
those to be a different color, etc. I also wanted to make sure the size of the symbols 
were large. I changed a SimpleRenderer to a UniqueValueRenderer, set the renderer to 
color the symbols based on the 'Pair_rts' field, and changed the size of each symbol
in the renderer:

```python
# Change the layer so that routes are displayed with a different color for each Pair_rts value
lyrsym = zoomExtent.symbology # first assign the symbology of the trail layer to a new object

# Check the renderer type of the trails layer
lyrsym.renderer.type
# 'SimpleRenderer'

# Change the renderer to a UniqueValuesRenderer to get routes that are displayed with a different color for
# each Pair_rts value
lyrsym.updateRenderer('UniqueValueRenderer')

# Now tell the renderer to use the Unit field for the unique values
lyrsym.renderer.fields = ['Pair_rts']

# First make the symbols of the layer bigger
for grp in lyrsym.renderer.groups:
    for itm in grp.items:
        itm.symbol.size = 12

# Update the symbology to save the changes
zoomExtent.symbology = lyrsym
```

Then, I changed the blank text box so that it was a title for the map, and repositioned 
it so that it was centered at the top of the page:

```python
# Change the title of the map and reposition it on the page:
# First assign the Text element of the layout to its own object and use this to change the text in the textbox to
# give the layout a more informative title
title = lyt.listElements('TEXT_ELEMENT', "Text")[0]
title.text = "Likely Related Routes in State {}".format(state)  # you can change the font, size, properties, etc.
title.textSize = 30  # change the size of the text
        
# Reposition the title textbox so that it is located at the top center of the page
upperX = 1.20
upperY = 10
title.elementPositionX = upperX
title.elementPositionY = upperY
```

Now that I had all the information I wanted displayed how I wanted in the map, 
I exported the map as a PDF:
```python
# Export the layout to a PDF
outFile = r"E:\Programming in GIS\NR 426 - Programming in GIS I\Final project\Data\States\17\Likely_related_routes_17_map.pdf"
lyt.exportToPDF(outFile, image_quality="BEST", embed_fonts=True)
```

I then removed the focal layer, so that the map
was blank for the process to start again for the next focal layer:
```python
# Now remove the focal layer so that the next focal layer and map can be made
map.removeLayer(zoomExtent)
```
## Workflow for Creating Multiple Maps

After getting the code working for a single state (Colorado; see code above), I needed 
to create a similar map for each of the three additional test distances I used for 
the rest of the 45 states for which I created these files (note: the input shapefiles
were created by expanding my code for my final project for NR426). In other words, 
I needed to create 180 maps of potentially related Breeding Bird Survey routes 
(4 test distances per state x 45 focal states). 

To this, I made a list of the folders that contain the data I needed to make
the maps:
```python
import os

# Set the root path; all the data I need are contained in folders within this folder
rootpath = r'E:\Programming in GIS\NR 426 - Programming in GIS I\Final project\Data\States'

# Get a list of the folders in the 'States' folder; all the folders in the 'States' 
# folder contain the data I need to create maps
subfolders = [f.name for f in os.scandir(rootpath) if f.is_dir()]  

# Change the string list of folders to a list of integers and assign to a new object; 
# used this list for looping later in the code
statenums = list(set(map(int, subfolders)))        
```

I also set up additional variables I needed to run my script:
```python
dist = "10 Miles"  # this variable was a test distance used to determine whether potentially related routes were actually
                  # likely related
dist_cap = dist.capitalize()  # puts the dist object in sentence capitalization for file naming purposes
split_dist_cap = dist_cap.split(" ")  # splits the dist_cap object using a space; for use later in the code for file
                                      # naming purposes
```

Then I iterated through each state to produce maps for a single test distance 
for each state. I didn't iterate over test distances (i.e., in addition to iterating
over states) because I preferred to have manual control over producing the maps for 
each test distance. 
```python
try:
    # Iterate through the states and create a separate map for each showing the potentially related routes that are
    # symbolized by the last two shared digits of their route number
    for state in statenums:
        # Create the project object
        aprx = mp.ArcGISProject("CURRENT")

        # Create a variable to represent the path to the data
        datapath = r"E:\Programming in GIS\NR 426 - Programming in GIS I\Final project\Data\States\{}\BBS_routes_1966_2017_{}_likely_related_{}_{}_no_duplicates.shp".format(state,state,split_dist_cap[0],split_dist_cap[1])

        # Reference the current map, then add data to it
        map = aprx.listMaps("Map")[0]

        map.addDataFromPath(datapath)

        # Assign the first (and only, in this project) layout to an object
        lyt = aprx.listLayouts("Layout")[0]

        # Assign the layers of the map to an object
        lyrs = map.listLayers()

        # Assign the focal layer to a new object
        zoomExtent = lyrs[0]

        # Assign the mapframe to an object
        mf = lyt.listElements("MAPFRAME_ELEMENT")[0]

        # Use the focal layer as the extent for panning
        ext = mf.getLayerExtent(zoomExtent,True)
        mf.panToExtent(ext)

        # Change the layer so that routes are displayed with a different color for each Pair_rts value
        lyrsym = zoomExtent.symbology # first assign the symbology of the trail layer to a new object

        # Check the renderer type of the trails layer
        lyrsym.renderer.type
        # 'SimpleRenderer'

        # Change the renderer to a UniqueValuesRenderer to get routes that are displayed with a different color for
        # each Pair_rts value
        lyrsym.updateRenderer('UniqueValueRenderer')

        # Now tell the renderer to use the Unit field for the unique values
        lyrsym.renderer.fields = ['Pair_rts']

        # First make the symbols of the layer bigger
        for grp in lyrsym.renderer.groups:
            for itm in grp.items:
                itm.symbol.size = 12

        # Update the symbology to save the changes
        zoomExtent.symbology = lyrsym

        # Change the title of the map and reposition it on the page:
        # First assign the Text element of the layout to its own object and use this to change the text in the textbox to
        # give the layout a more informative title
        title = lyt.listElements('TEXT_ELEMENT', "Text")[0]
        title.text = "Likely Related Routes in State {}".format(state)  # you can change the font, size, properties, etc.
        title.textSize = 30  # change the size of the text

        # Reposition the title textbox so that it is located at the top center of the page
        upperX = 1.20
        upperY = 10
        title.elementPositionX = upperX
        title.elementPositionY = upperY

        # Export the layout to a PDF
        outFile = r"E:\Programming in GIS\NR 426 - Programming in GIS I\Final project\Data\States\{}\Likely_related_routes_{}_{}_{}_map.pdf".format(state,state,split_dist_cap[0],split_dist_cap[1])
        lyt.exportToPDF(outFile, image_quality="BEST", embed_fonts=True)

        # Now remove the focal layer so that the next focal layer and map can be made
        map.removeLayer(zoomExtent)

except Exception as e:
    print("Error: " + e.args[0])
```

