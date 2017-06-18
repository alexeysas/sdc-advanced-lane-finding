## Advanced Lane Finding Project

### Goal
The goal of the project is to detect current car lane, detecting both left and right line.

### Steps
Following steps were applied:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./images/distortion.png "Distortion"
[image2]: ./images/road_normal.png "Road"
[image3]: ./images/road_undistorted.png "Road undistorted"

[image4]: ./images/initital_debug.png "Initial"
[image5]: ./images/Undistorted_debug.png "Undistorted"
[image6]: ./images/SobelX_debug.png "Sobel X applied"
[image7]: ./images/S_channel_selection_debug.png "S channel selected"
[image8]: ./images/SobelX_for_S_channel_debug.png "S channel + Sobel X"
[image9]: ./images/Combined_debug.png "Combined"

[image10]: ./images/road_region.png "Region"
[image11]: ./images/road_perspective.png "Perspective"
[image12]: ./images/Bird_View_debug.png "Bird view"
[image13]: ./images/hist.png "Histogram"
[image14]: ./images/sliding.png "Histogram"



[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

### Camera Calibration

First we need to transform image to remove [a camera distortion effect](https://en.wikipedia.org/wiki/Distortion_(optics)) which might affect finding correct lines curvature.

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Object points were set as 0:8, 0:6 grid matrix. Thry are the same for every successfully detected chessboard image from the callibration folder. Additionaly, we set `image points` to the (x, y) coordinates of the successfully detected corners of the calibration chessboard image.

Then we are combined all "object points" and "image points" of each individual chessboard image into set of arrays and use open cv (`cv2.calibrateCamera()` function) to compute the camera calibration and distortion coefficients. After calibration is done I tested distortion correction on the test chessboard image using the `cv2.undistort()` function and obtained good result: 

![alt text][image1]

The code for this step is contained in cells 2-6 of the [a IPython notebook](project.ipynb)

### Pipeline (single images)

#### 1. Correct image distortion

First we need to remove distortion from the test image using matrix found during camera calibration.

After I applied distortion corection to the test image of the road I get pretty resonable results:

Normal image:

![alt text][image2]

Undistorted image:

![alt text][image3]

The code for this step is contained in cell 6 of the [a IPython notebook](project.ipynb)

#### 2. For the second step we need to pre-process image to extract line pixels.
Following techniques were considered:
* Using [a SobelX and SobelY operators](https://en.wikipedia.org/wiki/Sobel_operator)
* Using gradient magnitude
* Using direction gradient
* Using HSL color space convertion and gradients based on S channel.

The code for the methods above can be found in the cell 9 of the [a IPython notebook](project.ipynb)

I've as a result I've come up with following pre-processing pipiline:

* Load initial image
![alt text][image4]

* Undistort
![alt text][image5]

* Use SobelX operator on the source image 
![alt text][image6]

* Use S Channel threashhold for HSL color space
![alt text][image7]

* Use SobelX operator on the S Channel image
![alt text][image8]

* Combine SobelX for RGB and SobelX for S Channel image.
![alt text][image9]

The code for the pipline can be found in the cell 65 of the [a IPython notebook](project.ipynb)

#### 3. Perspective transform

The next step is to transform region of interest of the pre-processed image to the bird-view of the road lane to correctly detect line and measure curvature. 

First we need to identify projection transform matrix.  To achive this we need four points on the road which we know cplaces in a way so they determine rectangle.

I choose image with the straight falt line and determine rectangilar region below: 

![alt text][image10]

After that I determined mapped destination points on the bird-view image:

src = np.float32([(275,670), (1030,670), 
                   (766,500), (521,500)])

dst = np.float32([[250, 680], [1030, 680],
                    [1030, 300], [250, 300]])

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 275,670       | 250, 680      | 
| 1030,670      | 1030, 680     |
| 766,500       | 1030, 300     |
| 521,500       | 250, 300      |

I've used open CV getPerspectiveTransform function to calculate transform matrix and warpPerspective to perform perspective transform of the image.

This resulted image is following:

![alt text][image11]

The perspective transform worked as expected as road lines are vertical and parallel.

#### 4. Detect lines pixels

After the perspective transform is applied to the image from the pipeline above we are ready to find lines and lines curvature:

![alt text][image12]

First we need to find starting points - for this purpose we buit historgram to find left and right peaks with maximum number of detected points:

![alt text][image13]

After that we can use found X coordinates as a starting points to search both lines. In the loop we are sliding windows centered in the found x coourdinates up. When we found enough line pixels inside the window we recentered sliding window based on the mean value of these pixels.



The code to find line based on the bird-eye view image  can be found in the cells 50 of the [a IPython notebook](project.ipynb)

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
