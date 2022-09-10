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

### 3. Creating a custom layout for complex scenes: 
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
