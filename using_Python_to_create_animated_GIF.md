I had a map of Oregon in which I followed some John Nelson tutorials to have each county filled with a pastel along with a darker vignette along the borders and an inner glow.



For a Python exercise, I was curious for the steps to start with a blank map of Oregon counties, and then add three random colored counties in a cumulative fashion until the 

blank map of Oregon was full of the pastel colors. And then reverse this process so that three colored counties would be taken away at random, so that the colored map reverts

to the original blank state.



In other words, start from this:



!\[Frame 001](assets/images/frame\_001.png)



to this:



!\[Frame 013](assets/images/frame\_013.png)



I wanted to ensure that no same three counties were batched together when either being added or removed. I also wanted the add / remove animation to run five times, and then have a GIF created in my output path folder.



After a day of debugging, below is the final script to use Python in ArcGIS Pro create an animated GIF.



\# ===========================================================

\# START

\# ===========================================================





import arcpy, os, random

from PIL import Image

import glob



\# ===========================================================

\# 1. ENVIRONMENT SETUP

\# ===========================================================

aprx = arcpy.mp.ArcGISProject("CURRENT")

mp = aprx.listMaps()\[0]



blank\_layer\_name = "orcntypoly"

colored\_layer\_name = "County Layer for GIF"



blank\_layer = mp.listLayers(blank\_layer\_name)\[0]

colored\_layer = mp.listLayers(colored\_layer\_name)\[0]



\# If it's a group layer, drill down

if colored\_layer.isGroupLayer:

&#x20;   colored\_layer = colored\_layer.listLayers()\[0]



\# Clear filters and selections

colored\_layer.definitionQuery = ""

arcpy.management.SelectLayerByAttribute(colored\_layer, "CLEAR\_SELECTION")



\# Layout + map frame

layout = aprx.listLayouts()\[0]

mapframe = layout.listElements("MAPFRAME\_ELEMENT")\[0]



\# Output folder

output\_folder = r"D:\\GIS\\Colorful Oregon Counties GIF\\Frames"

os.makedirs(output\_folder, exist\_ok=True)



print("Environment ready.")

print("Blank layer:", blank\_layer.name)

print("Colored layer:", colored\_layer.name)



\# ===========================================================

\# 2. NORMALIZATION + UNIQUE COUNTY LIST

\# ===========================================================

def normalize(code):

&#x20;   return str(code).zfill(3)



county\_field = "COBCODE"



all\_counties = sorted({

&#x20;   normalize(row\[0].replace("OR", ""))

&#x20;   for row in arcpy.da.SearchCursor(colored\_layer, \[county\_field])

})



print("Unique counties:", len(all\_counties))

print(all\_counties)



\# ===========================================================

\# 3. UNIQUE BATCH PICKER

\# ===========================================================

def pick\_unique\_batch(remaining, used\_batches):

&#x20;   attempts = 0

&#x20;   while True:

&#x20;       attempts += 1



&#x20;       if len(remaining) <= 3:

&#x20;           batch = remaining\[:]

&#x20;       else:

&#x20;           batch = random.sample(remaining, 3)



&#x20;       batch = \[normalize(c) for c in batch]

&#x20;       key = tuple(sorted(batch))



&#x20;       if key not in used\_batches:

&#x20;           used\_batches.add(key)

&#x20;           return batch



&#x20;       if attempts > 1000:

&#x20;           raise RuntimeError("Unable to find unused batch — too many cycles requested.")



\# ===========================================================

\# 4. FORWARD ANIMATION

\# ===========================================================

def forward\_cycle(start\_frame, used\_batches):

&#x20;   remaining = all\_counties.copy()

&#x20;   colored\_so\_far = \[]

&#x20;   frame = start\_frame



&#x20;   colored\_layer.definitionQuery = "1 = 0"

&#x20;   layout.exportToPNG(os.path.join(output\_folder, f"frame\_{frame:03d}.png"), resolution=200)

&#x20;   frame += 1



&#x20;   while remaining:

&#x20;       batch = pick\_unique\_batch(remaining, used\_batches)



&#x20;       for c in batch:

&#x20;           if c not in colored\_so\_far:

&#x20;               colored\_so\_far.append(c)



&#x20;       sql\_values = ",".join(\[f"'OR{c}'" for c in colored\_so\_far])

&#x20;       sql = f"{county\_field} IN ({sql\_values})"

&#x20;       colored\_layer.definitionQuery = sql



&#x20;       layout.exportToPNG(os.path.join(output\_folder, f"frame\_{frame:03d}.png"), resolution=200)

&#x20;       frame += 1



&#x20;       for c in batch:

&#x20;           if c in remaining:

&#x20;               remaining.remove(c)



&#x20;   return frame, colored\_so\_far.copy()



\# ===========================================================

\# 5. REVERSE ANIMATION

\# ===========================================================

def reverse\_cycle(start\_frame, full\_list, used\_batches):

&#x20;   remaining = full\_list.copy()

&#x20;   colored\_so\_far = full\_list.copy()

&#x20;   frame = start\_frame



&#x20;   sql\_values = ",".join(\[f"'OR{c}'" for c in colored\_so\_far])

&#x20;   sql = f"{county\_field} IN ({sql\_values})"

&#x20;   colored\_layer.definitionQuery = sql

&#x20;   layout.exportToPNG(os.path.join(output\_folder, f"frame\_{frame:03d}.png"), resolution=200)

&#x20;   frame += 1



&#x20;   while remaining:

&#x20;       batch = pick\_unique\_batch(remaining, used\_batches)



&#x20;       for c in batch:

&#x20;           if c in colored\_so\_far:

&#x20;               colored\_so\_far.remove(c)



&#x20;       if colored\_so\_far:

&#x20;           sql\_values = ",".join(\[f"'OR{c}'" for c in colored\_so\_far])

&#x20;           sql = f"{county\_field} IN ({sql\_values})"

&#x20;       else:

&#x20;           sql = "1 = 0"



&#x20;       colored\_layer.definitionQuery = sql



&#x20;       layout.exportToPNG(os.path.join(output\_folder, f"frame\_{frame:03d}.png"), resolution=200)

&#x20;       frame += 1



&#x20;       for c in batch:

&#x20;           if c in remaining:

&#x20;               remaining.remove(c)



&#x20;   return frame



\# ===========================================================

\# 6. RUN 5 FULL CYCLES

\# ===========================================================

frame\_counter = 1



for cycle in range(5):

&#x20;   print(f"=== Starting cycle {cycle+1} ===")



&#x20;   used\_batches = set()



&#x20;   frame\_counter, full\_list = forward\_cycle(frame\_counter, used\_batches)

&#x20;   frame\_counter = reverse\_cycle(frame\_counter, full\_list, used\_batches)



print("All 5 cycles complete.")



\# ===========================================================

\# 7. CREATE GIF FROM FRAMES

\# ===========================================================

print("Building GIF...")



frames = \[]

for file in sorted(glob.glob(os.path.join(output\_folder, "frame\_\*.png"))):

&#x20;   frames.append(Image.open(file))



gif\_path = os.path.join(output\_folder, "oregon\_animation.gif")



frames\[0].save(

&#x20;   gif\_path,

&#x20;   save\_all=True,

&#x20;   append\_images=frames\[1:],

&#x20;   duration=120,   # ms per frame (adjust speed here)

&#x20;   loop=0

)



print("GIF created:", gif\_path)



\# ===========================================================

\# END

\# ===========================================================



The final result:



!\[Animation](assets/images/Oregon\_animation.gif)







