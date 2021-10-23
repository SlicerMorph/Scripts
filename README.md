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
