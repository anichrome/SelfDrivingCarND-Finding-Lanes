# **Finding Lane Lines on the Road** 

<img src="examples/laneLines_thirdPass.jpg" width="480" alt="Combined Image" />

Overview
---

When we drive, we use our eyes to decide where to go.  The lines on the road that show us where the lanes are act as our constant reference for where to steer the vehicle.  Naturally, one of the first things we would like to do in developing a self-driving car is to automatically detect lane lines using an algorithm.

The goals / steps of this project are the following:
* Build a pipeline to find lane lines on the road in images
* Use the pipeline to find lane lines on the road in video

[//]: # (Image References)

[im01]: ./test_images/solidWhiteCurve.jpg "Solid White Curve"
[im02]: ./test_images/solidWhiteRight.jpg "Solid White Right"
[im03]: ./test_images/solidYellowCurve.jpg "Solid Yellow Curve"
[im04]: ./test_images/solidYellowCurve2.jpg "Solid Yellow Curve2"
[im05]: ./test_images/solidYellowLeft.jpg "Solid Yellow Left"
[im06]: ./test_images/whiteCarLaneSwitch.jpg "White Car Lane Switch"

[im07]: ./test_images_output/solidWhiteCurveRoi.png "Solid White Curve"
[im08]: ./test_images_output/solidWhiteRightRoi.png "Solid White Right"
[im09]: ./test_images_output/solidYellowCurveRoi.png "Solid Yellow Curve"
[im10]: ./test_images_output/solidYellowCurve2Roi.png "Solid Yellow Curve2"
[im11]: ./test_images_output/solidYellowLeftRoi.png "Solid Yellow Left"
[im12]: ./test_images_output/whiteCarLaneSwitchRoi.png "White Car Lane Switch"

[im13]: ./test_images_output/solidWhiteCurve.png "Solid White Curve"
[im14]: ./test_images_output/solidWhiteRight.png "Solid White Right"
[im15]: ./test_images_output/solidYellowCurve.png "Solid Yellow Curve"
[im16]: ./test_images_output/solidYellowCurve2.png "Solid Yellow Curve2"
[im17]: ./test_images_output/solidYellowLeft.png "Solid Yellow Left"
[im18]: ./test_images_output/whiteCarLaneSwitch.png "White Car Lane Switch"

## 1. Describing the pipeline

The pipeline to find lane lines consists of following steps:

### Finding prominent line segments

The first step in the pipeline is to remove unnecessary details in the image and to extract the line segments which would potentially represent the lanes. In this work, I implemented two different ways of extracting the line segments.

#### Extracting the line segments with edge detection

In this approach, the image is first converted to grayscale using

   ```
   grayscale = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
   
   ```

The original RGB image is converted to grayscale to reduce the color channels. It is easier to process an image this way.


2. Convert the grayscale image to blurred image

```
    blurredImage = gaussian_blur(grayImage, kernel_size)
```
Use the gaussian blur function to blur the grascale image. The gaussian blur further reduces the noise in input grayscale image.


3. Find the edges

```
    edgeImage = canny(blurredImage, low_threshold, high_threshold)
```

A canny edge detector is used to detect strong edges in the blurred image which could potentially describe the lane lines.

Using this approach, I could farely identify the lines. But, once the lighting conditions and road curvature gets difficult, this approach fails to perform well. I was unable to find lines effectively which resulted in empty lines. 

To improve my implementation, I used another approach,


#### Extracting the line segments using Lightness and Saturation channel

in this approach, I converted the image to HSV image from which I could separate lightness and saturation channels of the image. The lightness channel was found to identify white lines and the saturation channel was found to identify yellow lines better.

Then, I converted the two channels to a binary image with a fixed threshold. Then, I combined the two binary images to get a combined thresholded binary image which identified white and yellow lines more effectively. The code can be found in code block 3.

```
def getBinaryThresholdL(lChannel, thresh=(200, 255)):
    binaryImage = np.zeros_like(lChannel)
    binaryImage[(lChannel > thresh[0]) & (lChannel <= thresh[1])] = 1
    return binaryImage    

def getBinaryThresholdS(sChannel, thresh=(160, 255)):

    binaryImage = np.zeros_like(sChannel)
    binaryImage[(sChannel > thresh[0]) & (sChannel <= thresh[1])] = 1
    return binaryImage

def getCombinedBinaryImage(lChannel, sChannel):
    # Combine L and S channels
    combined = np.zeros_like(lChannel)
    combined[(lChannel == 1) | (sChannel == 1)] = 1  
    return combined
    
```    

### Region of interest


A region of interest is identified in the image where it is probable of finding the lanes. This reduces the computations required on the whole image. All the pixels which do not belong in this region of interest is removed. This is called masked image.

    #imshape = image.shape
    #vertices = np.array([[(0,imshape[0]),(500, 270), (imshape[1], imshape[0]), (imshape[1],imshape[0])]], dtype=np.int32)
    #maskedImage = region_of_interest(combinedBinaryImage, vertices)


### Hough Transformation


From the masked image, hough transformation is used to find all the line segments in the image. The function returns a blank image(black background) with lanes drawn on it.

    rho = 2
    theta = np.pi/180.0
    threshold = 65
    min_line_len = 110
    max_line_gap = 350
    #lineImage = hough_lines(maskedImage, rho, theta, threshold, min_line_len, max_line_gap) 



### Extending the draw_line function

In order to use the lines returned by the hough transform and draw them on the original image, several modifications were done to the draw_line function which are described below:


The lines from the hough transform are seperated as belonging to left and right lanes by their slope. Negative slopes belong to left lane and positive slopes belong to right lane.

            if slope < 0:
                left_lines_slope.append(slope)
                left_lines_intercept.append(intercept)
                
            else:
                right_lines_slope.append(slope)
                right_lines_intercept.append(intercept)


In the next step, I removed some outliers in the slope and intercepts to eliminate the jitter and improve the accuracy.

```
    #remove outliers from the list of slopes
    mean_left = np.mean(numpy_left_slopes, axis=0)
    sd_left = np.std(numpy_left_slopes, axis=0)    
    final_list_left = [x for x in left_lines_slope if (x > mean_left - 1 * sd_left)]
    final_list_left = [x for x in final_list_left if (x < mean_left + 1 * sd_left)] 
```
In the next step, I calculated the mean of slope and intercepts.

```
    meanSlopeLeft = sum(final_list_left) / len(final_list_left)
    meanSlopeRight = sum(final_list_right) / len(final_list_right)
    
    meanInterceptLeft = sum(final_list_left_intercept) / len(final_list_left_intercept)
    meanInterceptRight = sum(final_list_right_intercept) / len(final_list_right_intercept)
```

With the help of individual line segments, an extrapolated single line is calculated for each side of the lane. The co-ordinates of such an extrapolated line is calculated as shown above. The line starts from bottom of the image and ends at the pixel defined by yMax.

```
    x1Left = (yMax - meanInterceptLeft)/meanSlopeLeft
    y1Left = yMax
    x2Left = (yMin - meanInterceptLeft)/meanSlopeLeft)
    y2Left = yMin
    x1Right = (yMax - meanInterceptRight)/meanSlopeRight
    y1Right = yMax
    x2Right = (yMin - meanInterceptRight)/meanSlopeRight
    y2Right = yMin
```        


### Draw lines on the original RGB Image

The last step is to draw the extrapolated lanes on the original RGB image. The code for this stage can be found in code block 5.

The figures below shows original image, the ROI image and the image with lane lines drawn for the test images provided.

|Original Image | ROI Image | Lane Image |
|:---:|:---:|:---:|
| ![alt text][im01] | ![alt text][im07] | ![alt text][im13] |
| ![alt text][im02] | ![alt text][im08] | ![alt text][im14] |
| ![alt text][im03] | ![alt text][im09] | ![alt text][im15] |
| ![alt text][im04] | ![alt text][im10] | ![alt text][im16] |
| ![alt text][im05] | ![alt text][im11] | ![alt text][im17] |
| ![alt text][im06] | ![alt text][im12] | ![alt text][im18] |

### Video Pipeline

For the videos provided, I had to make more changes to the draw line function in order to process the video and output the data without too much jitter. Here are the changes I made.

#### Introducing Lines Class

I implemented a class to hold certain properties of the hough lines. The class looks as shown.

```
class Lines():
    def __init__(self):
        
        self.best_coordinates_exist = False
        
        self.best_mean_slope_left = 0 
        self.best_mean_slope_right = 0
        
        self.best_mean_intercept_left = 0
        self.best_mean_intercept_right = 0
        
        self.previous_lines_slope_left = []   
        self.previous_lines_slope_right = []
        
        self.previous_lines_intercept_left = []   
        self.previous_lines_intercept_right = []
```        

#### Introducing memory

In order for the video to be stable, I did not calculate the hough transform for every frame. I drew the line with the first set of lines and stored the co-ordinates. Only if a large deviation in the slope was found in the successive frames, I recalculated the hough transform and updated the co-ordinates the new line. 

#### Averaging slope and intercept over 'n' frames

In order to make the calculated slope and intercept robust, I stored the slopes and intercept for last n frames and averaged them for the result. This resulted in better and more robust line co-ordinates.

```
#store upto 70 previous lines
if len(self.previous_lines_slope_left) > 70:
    self.previous_lines_slope_left = self.previous_lines_slope_left[len(self.previous_lines_slope_left) - 70:]

if len(self.previous_lines_slope_right) > 70:    
    self.previous_lines_slope_right = self.previous_lines_slope_right[len(self.previous_lines_slope_right) - 70:]

```


### 2. Identify potential shortcomings with your current pipeline


One of the potential shortcoming would be when the lighting conditions in the image changes. This makes it difficult for the lightness and saturation channels to extract the lines effectively.

During my observation in the video, sometimes the line drawn has a tendancy to deviate too much from the mean. This could be because of the sudden and short change in the conditions which is not currently handled.

Changing lanes, left and right turns are not handled.

different weather conditions may change the perspective of the input image which is not handled right now.

Another shortcoming is that, if a vehicle is present in the region of interest the lanes are drawn on them too


### 3. Suggest possible improvements to your pipeline

A possible improvement would be to optimize the calculation of slopes, intercepts and co-ordinates. This would result in faster processing of the frame.

Improve the tracking of lines by a more sophisticated approach.

Give a confidence measure to the detection.

Use other color filters like HLS as an initial filter along with Region of Interest.



