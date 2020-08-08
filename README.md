# IS590 Programmming for Analytics - Summer - Final Project

### Land Use Analysis Leveraging Numpy & Point Cloud Generated Scans
**Author - Jeremy Carnahan**
 
 
## Background on Land Use Analytics
Aerial imagery has been used for many years, [predating even the airplane](https://en.wikipedia.org/wiki/Pigeon_photography), as people have used aerial perspectives for many industries.  One usecase in particular is for quickly accounting for how land is partitioned for farming, forestry, urbanization, roadways, etc. Flood planning utilizes this distribution of land use to determine rainfall absorption and runoff for hydrology models.

For almost a century, measurement of land use has utilized a process called photogrammetry to extract surface areas of land use categories by measuring aerial photos with trigonometry.  This process is still quite effective, but is dependent on knowing the precise altitude above ground and camera intrinsics.  In the past few decades a new technology called LiDAR (Light Detection And Ranging) leverages numerous laser pulses to scan terrain from the air, and returns very accurate representation of features on the ground.  The data this produces is called a **pointcloud**, which is a 3-dimensional array of points in a Cartesian grid.  Further, this pointcloud undergoes a classification process using a standard called [American Society for Photogrammetry & Remote Sensing v1.4](http://www.asprs.org/wp-content/uploads/2019/03/LAS_1_4_r14.pdf).  This standard is used to delineate land use categories below, either through manual categorization or machine learning, to about 20 categories.  In the flood planning use case, typically the three categories used for hydrology modeling are: vegetation, road surfaces, and ground. 


## Hypothesis
_Pointclouds are classified 3-dimensional XYZ points using ASCII format, and are generally several gigabytes to terrabytes in size.  My hypothesis is that we can use Numpy to analyze these points to more quickly determine land use surface area, as Numpy is designed for n-dimension vectorization, and efficiently leverages memory for quick analysis.  If this approach is successful, it could substitute the numerous hours of manual measurement of aerial photography to determine vegetation and road surface area for flood modeling._


##### Import Modules
We will need to pull in the modules we'll use to extract the pointcloud (.las file) into a proper Numpy ndArray using 'laspy'. 
From 'scipy' we'll use functions to calculate the area of classified points. 
'ipyleaflet' is used to get a course understanding of the area of interest by embedding a map we can calculate total area from.
'skimage.measure' is a module we can use to decimate our pointcloud for better visualization performance.


##### Area of Interest and Map Extent
In order to download the LiDAR data from the [National Oceanic & Atmospheric Administration](https://coast.noaa.gov/dataviewer/#/lidar/search/), we need to determine the extent (bounding box) of the area of interest.  ipyleaflet, which uses an open source tile mapping service using a product called Leaflet, can be embedded in Jupyter to determine overall surface area for the area of interest.  

![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/map.JPG "Area of Interest")


##### Importing the Pointcloud .las file
Now with our area of interest defined, and LiDAR data downloaded from NOAA, we can import the pointcloud using laspy to convert the .las ASCII file into a Numpy 3D array. 

We'll also do a little data exploration to determine the ASPRS standard version used and point metadata.

![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/all_classes_canted.JPG "All Feature Classes")


##### Data Size & Shape
Now that we've seen the metadata, we'll start to transform the raw Numpy extract from the .las in a more usable format.  We'll also take a look at the data's shape and size.

laspy.py documentation recommends extracting out each dimension from the .las file first, stacking, and transposing.  This is due to the way that the LiDAR systems organize the XYZ points in an orientation that differs from Numpy's XYZ/3D array. 

Recommendation found on laspy.py getting started page here: https://pythonhosted.org/laspy/tut_part_1.html


##### Data Transformation
As noted in the background section above, we are only interested in vegetation and road surfaces when determining land use for precipitation absorption and runoff (note that other hydrology methods to determine water flow can be derived from a pointcloud, but is beyond the scope of this analysis).  

Our first step is to extract just the classification we are interested in, and we'll save them out to files and load the points back in, which helps us manage our memory more effectively.  


##### Data Visualization
Here we start to run into some issues.  While so far the analysis of the pointclouds in numpy ndarrays has been smooth, with ample memory, as soon as we begin to try to visualize the data in Jupyter the system begins to stagger.  Additionally, the limited screen space limits any detailed view we may want to have to examine the pointcloud.

Below was an attempt to visualize the data in Matplotlib.  
![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/matplotlib_roads.JPG "Matplotlib plot of Roads class") 


Ultimately I chose to use an external tool called 'CloudCompare' to view the results as you can see in the following 4 images.  Note: hover your mouse over the images to see what land use feature classes have been extracted. 

![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/all_classes.JPG "All Feature Classes")  
![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/ground.JPG "Ground") 

![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/vegetation.JPG "Vegetation")  
![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/roads.JPG "Roads")

##### Pointcloud Decimation (Reduction)
Numpy arrays make it fairly easy to perform a reduction in the granularity of the pointcloud. While we wouldn't want to do this if we want to maintain accuracy/precision, it may be a useful step for reducing the size for visualization. I used a skimage.measure method called 'block_reduce' that systematically reduces the size (and shape if desired) using an average of points every 100 points.  With this method I was able to reduce the roads pointcloud from 197,241 points to 1974 points.  
  


##### Land Use Surface Area (and some bad news...)
Now that we've extracted out just the land use points/elements that we want, we now need to perform our area calculations.  'scipy.spatial' has a method called ConvexHull, which we've seen previously in IS-590; however, this is a little different.  Even though the method is called ConvexHull, when given a 3D numpy array, the method actually performs what is called a Delaunay triangulation invoked from Qhull class parent.  Effectively it connects the points with edges to create small triangle facets, and then calculates the area of each small triangle.  This has the added benefit over a simple aerial overlay in that Delaunay will also include additional area as it considers the z-axis in addition to just the x & y axis.  But there is bad news...

Unfortunately, it seems the scipy.spatial.ConvexHull method is unable to determine when it should not connect points that are not near to it.  A function that is used in pointcloud utilities (but not present in scipy) is to restrict the Delaunay to a preset threshold of "nearest neighbors", which results in **MANY** large faceted areas to be calculated and inflating our area results.  As we see in the ConvexHull results below, the area calculated is many times larger than our starting surface area.
![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/roads_intensity.JPG "Road Pointcloud")
![alt text](https://github.com/flyboy1378/Final_Projects_Su2020/blob/master/Data/Screenshots/roads_mesh.JPG "Roads with Delaunay Triangulation")


## Learnings

* The process to convert a .las file, manipulate it, and extract labeled features is straight forward through the use of Laspy.
* The the Numpy process performance is acceptable, even on large .las files up to 3GB, on a modern personal computing system.
* Point cloud visualization in Matplotlib and Ipyvolume were not acceptable on a modern personal computer, and it is recommended that an alternative resource, such as CloudCompare, is used for pointcloud visualization. 
* While not trivial, I believe there could be further steps taken to overcome the inaccuracies from the area calculations with more research into Delaunay triangulation that controls which facet edges are created based on a nearest neighbor threshold.





## References:

[Pigeon Photography](https://en.wikipedia.org/wiki/Pigeon_photography)

[NOAA Data Access Viewer](https://coast.noaa.gov/dataviewer/#/lidar/search/)

[NOAA Metadata for sample pointcloud used in project](https://github.com/flyboy1378/Final_Projects_Su2020/tree/master/Data/Metadata)

[ASPRS LAS Specification 1.4 - R14](http://www.asprs.org/wp-content/uploads/2019/03/LAS_1_4_r14.pdf)

[Leaflet Opensource Tile Mapping for Python](https://ipyleaflet.readthedocs.io/en/latest/api_reference/map.html#usage)

[Laspy .las Editor for Python](https://pythonhosted.org/laspy/tut_part_1.html)

[Pointcloud Classification Parsing](https://gis.stackexchange.com/questions/255833/classifying-lidar-ground-points-using-laspy)

[block_reduce from scikit-learn](https://scikit-image.org/docs/dev/api/skimage.measure.html#skimage.measure.block_reduce)

[ConvexHull/Delaunay from scipy.org docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.ConvexHull.html)

[scipy.spatial.ConvexHull Pull Request documentation](https://github.com/scipy/scipy/issues/12290)

[Github Markdown Cheatsheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)
