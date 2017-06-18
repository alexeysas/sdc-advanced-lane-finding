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
[image14]: ./images/sliding.png "Lines"
[image15]: ./images/result_sample1.png "Result"

[video1]: ./project_video_updated.mp4 "Video"
[video2]: ./challenge_video_updated.mp4 "Video"

### Camera Calibration

First we need to transform image to remove [a camera distortion effect](https://en.wikipedia.org/wiki/Distortion_(optics)) which might affect finding correct lines curvature.

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Object points were set as 0:8, 0:6 grid matrix. They are the same for every successfully detected chessboard image from the calibration folder. Additionally, we set `image points` to the (x, y) coordinates of the successfully detected corners of the calibration chessboard image.

Then we are combined all "object points" and "image points" of each individual chessboard image into set of arrays and use open cv (`cv2.calibrateCamera()` function) to compute the camera calibration and distortion coefficients. After calibration is done I tested distortion correction on the test chessboard image using the `cv2.undistort()` function and obtained good result: 

![alt text][image1]

The code for this step is contained in cells 2-6 of the [a IPython notebook](project.ipynb)

### Pipeline (single images)

#### 1. Correct image distortion

First, we need to remove distortion from the test image using matrix found during camera calibration.

After I applied distortion correction to the test image of the road I get pretty reasonable results:

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
* Using HSL color space conversion and gradients based on S channel.

The code for the methods above can be found in the cell 9 of the [a IPython notebook](project.ipynb)

As a result I've come up with following pre-processing pipeline:

* Load initial image
![alt text][image4]

* Undistort
![alt text][image5]

* Use SobelX operator on the source image 
![alt text][image6]

* Use S Channel threshold for HSL color space
![alt text][image7]

* Use SobelX operator on the S Channel image
![alt text][image8]

* Combine SobelX for RGB and SobelX for S Channel image.
![alt text][image9]

The code for the pipeline can be found in the cell 65 of the [a IPython notebook](project.ipynb)

#### 3. Perspective transform

The next step is to transform region of interest of the pre-processed image to the bird-view of the road lane to correctly detect line and measure curvature. 

First, we need to identify projection transform matrix.  To achieve this, we need four points on the road which we know placed in a way so they determine rectangle.

I selected image with the straight flat line and determine rectangular region below: 

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

First, we need to find starting points - for this purpose we built histogram to find left and right peaks with maximum number of detected points:

![alt text][image13]

After that we can use X coordinates we found as a starting point to search both lines. In the loop, we are sliding windows centered in the found x coordinates up. When we found enough line pixels inside the window we re-centered sliding window based on the mean value of these pixels.

After pixels are determined for the left and right lines - we can fit second order polynomial the pixels using numpy.polyfit function.

Here is sliding window and polynomial fit processing results:

![alt text][image14]

The code to find line based on the bird-eye view image can be found in the cells 50 of the [a IPython notebook](project.ipynb)

#### 5. Find Lines curvature and distance to the car center

To find line curvature we can use formula described  [a there](http://www.intmath.com/applications-differentiation/8-radius-curvature.php) 

As we have second order polynomial curve we need to find derivatives and apply to the formula specified above.

Three python functions were created for this and be found in the cell 14 of the [a IPython notebook](project.ipynb)

Using these functions and car position (center bottom of the image) we can find radius of curvature in pixels.  To find real-world radius of curvature I've hardcoded conversions coefficients according to my perspective projection and USA standards for the lane width

 ym_per_pix = 30.0 / 720 
 xm_per_pix = 3.4 / 700
 
 After that I've calculated real world curvature in meters for both current fit and average fit for the 5 previous frames (for the video only)
 
To calculate car position, I've used the fact that camera is mounted on the car center. Additionally, we need to find warped car center on the projection image. I've used Open CV perspectiveTransform function for that.
 
 Code can be found in the cell 8 of the [a IPython notebook](project.ipynb)
 
The final step is to measure distance by finding difference car center coordinate and bottom line's pixels, and convert distance to the real world measures using conversions coefficients above.

Code for the calculating lines curvature and car distance to the lines can be found in the calculate_lines_parameters function
in the cell 12 of the [a IPython notebook](project.ipynb)
 
 
#### 6. Plot detected lane and car information

The final step is to plot detected lane, curvature radius and car position together with debug information on the image.

Result can be found below:

![alt text][image15]

Drawing code can be found in the cell 14 of the [a IPython notebook](project.ipynb)

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_updated.mp4)

### Discussion

#### 1. Issues and future updates

One of the challenging tasks to make lane finding algorithm work correctly is many hyper parameters to tune including:

* Sliding windows size and count
* Gradients and color selection thresholds for the processing pipelines
* Pipeline configuration
* Vertical distance to analyze for the perspective transform

Additionally, algorithm does not work so well when there are complex shadows on the road, road is not clear and lines are old and not bright.

Also, algorithm will likely fail then there are "fake" lines on the road and road boarders like the lane lines.  Second video illustrates these issues well.

Also, algorithm is pretty slow. It can not be used for real time processing. So we need to work to improve perfomance by removing non-optimal code or probably consider different image resolution s as well.

For the future improvement, we can thing about following ways:

* Use information about standard lane width to filter such artifacts as "fake" lines.
* Implement more robust algorithm to reject outliers. For example, we can use GPS and map information to find real road curvature and compare with detected. 
* Even if I used information from the previous frame as a starting point to detect lines - this still can be improved by using combined information from the n previous frames - to achieve more stable result.
* Additionally, we can think about a ways of adjusting hyperparameters according to the lightning conditions and road state. As some parameters works pretty good on the shadowed clean road but not so good for road of different color.

