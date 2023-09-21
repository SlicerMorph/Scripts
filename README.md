# SlicerMorph Example Scripts
This repository show some sample python scripts for some convenience functions such as exporting to LMs to obscure format. You can use these examples to modify develop your own scripts for specific functions.

### 1. Export a folder of landmarks as a text file: 
```python
import os, glob, numpy

def convertMorphoJLM(inputFileDirectory, outputLandmarkFile)
  extensionInput = ".mrk.json"
  fileList = glob.glob(inputDirectory+"*.json")
  success, currentLMNode = slicer.util.loadMarkupsFiducialList(fileList[0])
  subjectNumber = len(fileList)
  landmarkNumber = currentLMNode.GetNumberOfFiducials()
  LMArray=numpy.zeros(shape=(subjectNumber,landmarkNumber,3))
  fileIndex=0
  fileStems=[];
  headerLM = ["Subject"]
  for pointIndex in range(currentLMNode.GetNumberOfMarkups()):     
    headerLM.append(currentLMNode.GetNthControlPointLabel(pointIndex)+'_X')
    headerLM.append(currentLMNode.GetNthControlPointLabel(pointIndex)+'_Y')
    headerLM.append(currentLMNode.GetNthControlPointLabel(pointIndex)+'_Z')

  LPS = [-1,-1,1]
  point = [0,0,0]
  for file in fileList:
    if fileIndex>0:
      success, currentLMNode = slicer.util.loadMarkupsFiducialList(file)     
    fileStems.append(os.path.basename(file.split(extensionInput)[0]))
    for pointIndex in range(currentLMNode.GetNumberOfMarkups()):
      point=currentLMNode.GetNthControlPointPositionVector(pointIndex)
      point.Set(point[0]*LPS[0],point[1]*LPS[1],point[2]*LPS[2])
      LMArray[fileIndex, pointIndex,:]=point
    slicer.mrmlScene.RemoveNode(currentLMNode)
    fileIndex+=1

  temp = numpy.column_stack((numpy.array(fileStems), LMArray.reshape(subjectNumber, int(3 * landmarkNumber))))
  temp = numpy.vstack((numpy.array(headerLM), temp))
  numpy.savetxt(outputLandmarkFile, temp, fmt="%s", delimiter=",")
```
### 2. Run an image processing pipeline on a folder of volumes: 
This example shows how to interact with a variety of Slicer modules via Python to create an automated imaging pipeline. The function takes a folder of volumes (NRRD or NII.GZ) extension) and for each volume will:
- Auto-threshold (OTSU)
- Run connected components (selecting the largest component)
- Create a model from the segmentation
- Use surface toolbox to decimate the model 80%
- Save the resultant model in OBJ format to the input folder
```python

def slicerScriptingExampleFunction(inputPath):
  import os
  # Setup the SurfaceToolboxWidget to access logic functions
  try:
    slicer.modules.SurfaceToolboxWidget
  except:
    currentModule = slicer.util.selectedModule()
    slicer.util.selectModule('SurfaceToolbox')
    slicer.util.selectModule(currentModule)
  
  # Create segment editor to get access to effects
  segmentEditorWidget = slicer.qMRMLSegmentEditorWidget()
  segmentEditorWidget.setMRMLScene(slicer.mrmlScene)
  segmentEditorNode = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLSegmentEditorNode")
  segmentEditorWidget.setMRMLSegmentEditorNode(segmentEditorNode)
  
  # Loop over and process each file in the inputPath
  for inputFilename in os.listdir(inputPath):
    # Check for valid extensions
    fn, ext = os.path.splitext(inputFilename)
    if (ext.lower() != '.nrrd'):
      fn2, ext2 = os.path.splitext(fn)
      if (ext2+ext.lower() != '.nii.gz'):
        print("skipping ", inputFilename)
        continue
    
    # Load volume and set up segmentation
    inputFilePath = os.path.join(inputPath, inputFilename)
    volumeNode = slicer.util.loadVolume(inputFilePath)
    segNode = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLSegmentationNode")
    segNode.SetReferenceImageGeometryParameterFromVolumeNode(volumeNode)
    addedSegmentID = segNode.GetSegmentation().AddEmptySegment("total")
    segmentEditorWidget.setSegmentationNode(segNode)
    segmentEditorWidget.setMasterVolumeNode(volumeNode)
    
    # Compute OTSU threshold
    import vtkITK
    thresholdCalculator = vtkITK.vtkITKImageThresholdCalculator()
    thresholdCalculator.SetInputData(volumeNode.GetImageData())
    thresholdCalculator.SetMethodToOtsu()
    thresholdCalculator.Update()
    OTSUThresholdValue = thresholdCalculator.GetThreshold()
    volumeScalarRange = volumeNode.GetImageData().GetScalarRange()
    
    # Thresholding with OTSU level
    segmentEditorWidget.setActiveEffectByName("Threshold")
    effect = segmentEditorWidget.activeEffect()
    effect.setParameter("MinimumThreshold", str(OTSUThresholdValue))
    effect.setParameter("MaximumThreshold",str(volumeScalarRange[1]))
    effect.self().onApply()
    
    # Connected components to extract largest component
    segmentEditorWidget.setActiveEffectByName("Islands")
    effect = segmentEditorWidget.activeEffect()
    effect.setParameter("Operation", "KEEP_LARGEST_ISLAND")
    effect.self().onApply()
    
    # Export segment to a model
    shNode = slicer.mrmlScene.GetSubjectHierarchyNode()
    exportFolderItemId = shNode.CreateFolderItem(shNode.GetSceneItemID(), "Segments")
    slicer.modules.segmentations.logic().ExportAllSegmentsToModels(segNode, exportFolderItemId)
    itemList=[]
    shNode.GetItemChildren(exportFolderItemId,itemList)
    if itemList is []:
      print("Error: no model node exported for ", volumeNode.GetName())
      continue
    modelNode = shNode.GetItemDataNode(itemList[0])
    
    # Decimate the model 80%
    slicer.modules.SurfaceToolboxWidget.logic.decimate(modelNode, modelNode, 0.8)
    
    # Save the resultant model as OBJ to the folder
    outputModelName = os.path.join(inputPath, volumeNode.GetName() + ".obj")
    slicer.util.saveNode(modelNode, outputModelName)
    
    # Clean up
    slicer.mrmlScene.RemoveNode(volumeNode)
    slicer.mrmlScene.RemoveNode(segNode)
    slicer.mrmlScene.RemoveNode(modelNode)
    shNode.RemoveItem(exportFolderItemId)
```
After defining this function, run it on a folder containing volumes using: 
```python

slicerScriptingExampleFunction('/path/to/my/data')
```

### 3. turn landmarks into 3D sphere models: 
This can be useful to display pointLists not as Markups, but as models. It can also used to create masks to train NNs models for LM detection. Model names are obtained from the label of the corresponding LM.  
Paste function into the python console, and specify a radius for the sphere model:

```
def markup2Spheres(markupNode, radius):
  for i in range (0, markupNode.GetNumberOfControlPoints()):
    # create a sphere centered at the current point
    sphereSource = vtk.vtkSphereSource()
    sphereSource.SetCenter(markupNode.GetNthControlPointPosition(i))
    sphereSource.SetRadius(radius)
    # set resolution of phi and theta to get a smooth surface
    sphereSource.SetPhiResolution(100)
    sphereSource.SetThetaResolution(100)
    sphereSource.Update()
    # create model and display nodes in the slicer scene to visualize the sphere
    modelNode = slicer.vtkMRMLModelNode()
    modelNode.SetAndObservePolyData(sphereSource.GetOutput())
    modelNode.SetName(markupNode.GetNthControlPointLabel(i))
    slicer.mrmlScene.AddNode(modelNode)
    modelDisplayNode = slicer.mrmlScene.AddNewNodeByClass('vtkMRMLModelDisplayNode')
    modelNode.SetAndObserveDisplayNodeID(modelDisplayNode.GetID())
```

If your pointList object called `F`, execute the function like this
```
n= getNode("F")
markup2Spheres(n, 8)
```

### 4. Creating a custom layout for complex scenes: 
This example shows how to generate a custom layout that allows more interaction with the scene. This example generates a row of three (3) 3D viewers on top, a row of three (3) Red, Green and Yellow slice views, and finally third row that contains a table and plot windows. The heights of each row is user adjustable. You can modify this example to create a layout that you prefer to use, and make it permanent by editing this .slicerrc.py start up script. 

```xml
# Define custom layouts for GPA modules in slicer global namespace
myCustomLayout = """
<layout type=\"vertical\">
 <item>
   <layout type=\"horizontal\">
     <item>
      <view class=\"vtkMRMLViewNode\" singletontag=\"1\">
       <property name=\"viewlabel\" action=\"default\">1</property>
      </view>
     </item>
     <item>
      <view class=\"vtkMRMLViewNode\" singletontag=\"2\" type=\"secondary\">"
       <property name=\"viewlabel\" action=\"default\">2</property>"
      </view>
    </item>
    <item>
      <view class=\"vtkMRMLViewNode\" singletontag=\"3\" type=\"secondary\">"
       <property name=\"viewlabel\" action=\"default\">3</property>"
      </view>
    </item>
   </layout>
 </item>
 <item>
  <layout type=\"horizontal\">
   <item>
    <view class=\"vtkMRMLSliceNode\" singletontag=\"Red\">
     <property name=\"orientation\" action=\"default\">Axial</property>
     <property name=\"viewlabel\" action=\"default\">R</property>
     <property name=\"viewcolor\" action=\"default\">#F34A33</property>
    </view>
   </item>
     <item>
      <view class=\"vtkMRMLSliceNode\" singletontag=\"Green\">
       <property name=\"orientation\" action=\"default\">Coronal</property>
       <property name=\"viewlabel\" action=\"default\">G</property>
       <property name=\"viewcolor\" action=\"default\">#F34A33</property>
      </view>
     </item>
   <item>
    <view class=\"vtkMRMLSliceNode\" singletontag=\"Yellow\">"
       <property name=\"orientation\" action=\"default\">Sagittal</property>
       <property name=\"viewlabel\" action=\"default\">Y</property>
       <property name=\"viewcolor\" action=\"default\">#F34A33</property>
    </view>"
   </item>"
  </layout>
 </item>
 <item>
  <layout type=\"horizontal\">
     <item>
      <view class=\"vtkMRMLPlotViewNode\" singletontag=\"PlotViewerWindow_1\">
       <property name=\"viewlabel\" action=\"default\">1</property>
      </view>
     </item>
   <item>
    <view class=\"vtkMRMLTableViewNode\" singletontag=\"TableViewerWindow_1\">"
     <property name=\"viewlabel\" action=\"default\">T</property>"
    </view>"
   </item>"
  </layout>
 </item>
</layout>
"""

#assign arbitrary number to custom layout (must be unique)
customLayoutId=707
layoutManager = slicer.app.layoutManager()
layoutManager.layoutLogic().GetLayoutNode().AddLayoutDescription(customLayoutId, myCustomLayout)
layoutManager.setLayout(customLayoutId)

```

### 5. Read an NRRD volume from the disk, and resample it to a specified voxel size.

```
#where to read the input file and save the reduced file
filePath="/Users/amaga/Library/Caches/slicer.org/Slicer/SlicerIO/MR-head.nrrd"
outPath="/Users/amaga/Desktop/Test.nrrd"

# read file into Slicer
slicer.util.loadVolume(filePath)

# find the files NodeID
volumeNode = getNode('MR-head')

#create a blank Markup ROI
roiNode = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLMarkupsROINode")

#set the new markup ROI to the dimensions of the volume holaded
cropVolumeParameters = slicer.mrmlScene.AddNewNodeByClass("vtkMRMLCropVolumeParametersNode")
cropVolumeParameters.SetInputVolumeNodeID(volumeNode.GetID())
cropVolumeParameters.SetROINodeID(roiNode.GetID())
slicer.modules.cropvolume.logic().SnapROIToVoxelGrid(cropVolumeParameters)  # optional (rotates the ROI to match the volume axis directions)
slicer.modules.cropvolume.logic().FitROIToInputVolume(cropVolumeParameters)
slicer.mrmlScene.RemoveNode(cropVolumeParameters)

#set the cropping parameters
cropVolumeLogic = slicer.modules.cropvolume.logic()
cropVolumeParameterNode = slicer.vtkMRMLCropVolumeParametersNode()
cropVolumeParameterNode.SetIsotropicResampling(True)

#set the output resolution to 2 millimeters. units in slicer is always in mm. 
cropVolumeParameterNode.SetSpacingScalingConst(2)
cropVolumeParameterNode.SetROINodeID(roiNode.GetID())
cropVolumeParameterNode.SetInputVolumeNodeID(volumeNode.GetID())

#do the cropping
cropVolumeLogic.Apply(cropVolumeParameterNode)

#obtain the nodeID of the cropped volume 
croppedVolume = slicer.mrmlScene.GetNodeByID(cropVolumeParameterNode.GetOutputVolumeNodeID())

#write the cropped volume to file
slicer.util.exportNode(croppedVolume, outPath, {"useCompression": 0})
```
