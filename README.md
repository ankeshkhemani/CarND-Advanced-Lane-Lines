# CarND-Advanced-Lane-Lines
My submission for the Udacity Self-Driving Car Nanodegree program Project 4 - Advanced Lane Lines Detection

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.
* Run the pipeline on a video with the output superimposed on the video.

[//]: # (Image References)

[im01]: ./output_images/01-calibration.png "Chessboard Calibration"
[im02]: ./output_images/02-undistort_chessboard.png "Undistorted Chessboard"
[im03]: ./output_images/03-undistort.png "Undistorted Dashcam Image"
[im04]: ./output_images/04-unwarp.png "Perspective Transform"
[im05]: ./output_images/05-colorspace_exploration.png "Colorspace Exploration"
[im06]: ./output_images/09-sobel_magnitude_and_direction.png "Sobel Magnitude & Direction"
[im07]: ./output_images/11-hls_l_channel.png "HLS L-Channel"
[im08]: ./output_images/12-lab_b_channel.png "LAB B-Channel"
[im09]: ./output_images/13-pipeline_all_test_images.png "Processing Pipeline for All Test Images"
[im10]: ./output_images/14-sliding_window_polyfit.png "Sliding Window Polyfit"
[im11]: ./output_images/15-sliding_window_histogram.png "Sliding Window Histogram"
[im12]: ./output_images/16-polyfit_from_previous_fit.png "Polyfit Using Previous Fit"
[im13]: ./output_images/17-draw_lane.png "Lane Drawn onto Original Image"
[im14]: ./output_images/18-draw_data.png "Data Drawn onto Original Image"

[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
*Here I will consider the rubric points individually and describe how I addressed each point in my implementation.*


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.
This is the first step of the project under the heading, "Compute the camera calibration using chessboard images"
A number of images of a chessboard taken with the same camera from different angles, is given as input. OpenCV's findChessboardCorners function is used to obtain the internal corners on the input images. These image points along with actual indices of the chessboard's internal corners, are fed to OpenCV's calibrateCamera function which returns camera calibration and distortion coefficients. 
Once calculated, these camera calibration and distortion coefficients can then be used by OpenCV's undistort function to undistort any image taken by the same camera.

The image below is an example of a raw calibration image and its undistorted version:

![alt text][im02]

### Pipeline (test images)

#### 1. Provide an example of a distortion-corrected image.

The image below shows the results of applying distortion correction on one of the test images.

![alt text][im03]


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this can be found after the perspective transform in the project.ipynb
It made more sense to form a binary image after perspective transform.

The below image shows the various channels of three different color spaces for the same image:

![alt text][im05]

The primary project video consists of standard white and yellow lines. As seen from the course lecture, quizes, Udacity forum discussions and in the image above, the HLS L channel does a good job in isolating white lines and the LAB B channel does a good job in isolation yellow lines.
To keep the pipeline simple, I chose to use just these in my code and finetuned the thresholds to work well with all test images. 
The maximum value of each channels were normalized to 255 to work well in different lighting conditions. To get rid of noise, the B channel was not normalized if the maximum value for an image was less than 175, signifying that there was no actual yellow line in the image. 

Below are examples of thresholds in the HLS L channel and the LAB B channel:

![alt text][im07]
![alt text][im08]


Below are the binary images for all the corresponding test images after applying a combination of HLS L Channel and LAB B Channel thresholding:

![alt text][im09]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is titled "Perspective Transform" in the Jupyter notebook.
The `unwarp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  Since the project video is seemingly without elevation, I chose to hardcode the source and destination points in the following manner:

```
src = np.float32([(575,464),
                  (707,464), 
                  (258,682), 
                  (1049,682)])
dst = np.float32([(450,0),
                  (w-450,0),
                  (450,h),
                  (w-450,h)])
```

The image below demonstrates the results of the perspective transform: 

![alt text][im04]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The functions `sliding_window_polyfit` under the heading "Sliding Window Polyfit" and `polyfit_using_prev_fit` under the heading "Polyfit Using Fit from Previous Frame" in the code, identify lane lines and fit a second order polynomial to both right and left lane lines. 

The sliding_window_polyfit computes a histogram of the bottom half of the image and finds the base of the left and right lane lines. The right lane and left lane is identified just left and right of the midpoint. This helped to reject lines from other lanes. 
The function then identifies ten windows from which to identify lane pixels, each one centered on the midpoint of the pixels from the window below. This effectively "follows" the lane lines up to the top of the binary image, and speeds processing by only searching for activated pixels over a small portion of the image. Pixels belonging to each lane line are identified and the Numpy `polyfit()` method fits a second order polynomial to each set of pixels. The image below demonstrates how this process works:
![alt text][im10]

The image below depicts the histogram generated by `sliding_window_polyfit`; the resulting base points for the left and right lanes - the two peaks nearest the center - are clearly visible:

![alt text][im11]

The `polyfit_using_prev_fit` leverages a previous fit (from a previous video frame, for example) and only searches for lane pixels within a certain range of that fit. The image below demonstrates this - the green shaded area is the range from the previous fit, and the yellow lines and red and blue pixels are from the current image:

![alt text][im12]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature is calculated in the code cell titled "Radius of Curvature and Distance from Lane Center Calculation" using this line of code:
```
curve_radius = ((1 + (2*fit[0]*y_0*y_meters_per_pixel + fit[1])**2)**1.5) / np.absolute(2*fit[0])
```
In this example, `fit[0]` is the first coefficient (the y-squared coefficient) of the second order polynomial fit, and `fit[1]` is the second (y) coefficient. `y_0` is the y position within the image upon which the curvature calculation is based (the bottom-most y - the position of the car in the image - was chosen). `y_meters_per_pixel` is the factor used for converting from pixels to meters. This conversion was also used to generate a new fit with coefficients in terms of meters. 

The position of the vehicle with respect to the center of the lane is calculated with the following lines of code:
```
lane_center_position = (r_fit_x_int + l_fit_x_int) /2
center_dist = (car_position - lane_center_position) * x_meters_per_pix
```
`r_fit_x_int` and `l_fit_x_int` are the x-intercepts of the right and left fits, respectively. The car position is the difference between these intercept points and the image midpoint.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

You can find this code in the section titled "Draw the Detected Lane Back onto the Original Image" and "Draw Curvature Radius and Distance from Center Data onto the Original Image" in the project notebook. A polygon is generated based on plots of the left and right fits, warped back to the perspective of the original image using the inverse perspective matrix `Minv` and overlaid onto the original image. The image below is an example of the results of the `draw_lane` function:

![alt text][im13]

Below is an example of the results of the `draw_data` function, which writes text identifying the curvature radius and vehicle position data onto the original image:

![alt text][im14]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I encountered problems isolating the white and yellow lines,specifically with the noise in the binary image. Having the maximum normalized to 255 and cutoff for the LAB B threshold as 175 helped stabilise the outcome for the project video's lighting conditions. However, I am afraid this won't work well on other situations. I also did not use any gradient thresholds or dynamic thresholding in my pipeline, however I hope to revisit this in future so that the pipeline is more robust to different kinds of roads.

For the perspective transform, I used hardcoded metrices for source and destination assuming the road is flat as in the project video. However, in case of elevation, this fails remarkably as tested on the challenge videos. Hence it would be more robust to determine the src and dst metrices dynamically. 
I also think that using more than 10 sliding windows in sliding_window_polyfit function might work better for sharp turns like that of the challenge videos.


For Visualisation tools and tricks, I referred to udacity carnd forums and my cohort's facebook group discussions.
