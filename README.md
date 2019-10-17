## README

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.
* Implement plausibilization/tracking mechanisms for continuous lane detection and processing based on the video
* Feed the project video to the developed pipeline and visualize the estimated lane as well as the estimated curvatures (left and right) and the vehicle's offset from the lane center

[//]: # (Image References)

[image1]: ./camera_cal/calibration1.jpg "Distorted image"
[image2]: ./output_images/calibration1.jpg "Undistorted image"
[image3]: ./test_images/test7.jpg "Distorted image"
[image4]: ./output_images/test_2undist.jpg "Undistorted image"

[image5]: ./test_images/test1.jpg "Original image"
[image6]: ./output_images/test2.jpg "Binary image"

[image7]: ./test_images/test8.jpg "Original image"
[image8]: ./output_images/warped6.jpg "Binary warped image"

[image9]: ./output_images/test_all4.jpg "Warped image, sliding window search, estimated polynoms and lane projection onto the original image"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

Writeup provided.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

First, object points are extracted. The object world coordinates are simulated in form of a 9x6 mesh grid. In order to find the image points, a corner detection function cv2.findChessboardCorners() was applied to one of the given images. For each calibration image with detected corners a copy of synthetic object array will be appended to the original object points' array 'objpoints' and detected corners will be appended to the 'imgpoints' array. 'objpoints' and 'imgpoints' arrays will then be fed to the 'cv2.calibrateCamera()' function in order to calculate the camera matrix 'mtx' and the distortion coefficients 'dist'. With known lens distortion, the original image was then undistorted by using the 'cv2.undistort()' function (see images below).

![alt text1][image1]![alt-text2][image2]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

![alt text1][image3]![alt-text2][image4]

Effects of the original distortion and its correction can be seen on the change of the car hood's shape.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

In order to obtain a binarized image with distinctive lane markings, I decided to try both color and gradient thresholding. In order to have a flexible way of tuning, I took use of the iPython widgets, specifically, the 'interact' functionality: https://ipywidgets.readthedocs.io/en/latest/examples/Using%20Interact.html - see the notebook for results.

Using gradient thresholding seemed not effective enough as many damaged/dirty road surface caused too many artefacts to come through the thresholding, which would then have a negative impact on the polynom fitting. In order to detect yellow lines, the b-channel of the LAB color scheme was used. For the white line detection, the lightness channel of the hls color palette was used. The color filtering functionality is implemented in the `color_grad_thresholding()` function (starting with the code fragment at line 61).

![alt text1][image5]![alt text2][image6]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The perspective transformation is done by using the function `cv2.getPerspectiveTransform(src, dst)`. To do so, a set of source and destination points was hardcoded (this combination allowed for a transformation with the line parallelism being preserved):

```python
src_bottom_left = [200,720]
src_bottom_right = [1200, 720]
src_top_left = [565, 470]
src_top_right = [740, 470]

dst_bottom_left = [300,720]
dst_bottom_right = [1000, 720]
dst_top_left = [300, 1]
dst_top_right = [1000, 1]
```

I am not a fan of hardcoding, but here I was at least happy with the result.

The result of the perspective transformation on the binary image can be seen here:

![alt text1][image7]![alt-text2][image8]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Finding the outstanding pixels has never been easier thanks to the binary character of the image! Here, the most likely location of a lane marking is found by computing a histogram over the y-dimension of the image. The x region with the maximum gradient value represents the potential lane marking candidate. This procedure is done for both left and right halves of the image - this way, left and right lane markings can be located. Pixel extraction is then done by a sliding-window approach - the "good" pixels are searched for within windows located along the y-axis; after that, the next window is relocated according to the mean x-position of the pixels detected within the previos window. This procedure is implemented within the `find_lane_pixels()` function. After the initial detection was successful, it makes sense to start looking for the line candidate in the next frame within the area of the previosly detected line. This is implemented in the `search_around_poly()` function.

![alt text][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of the curvature was calculated within the function `meas_curv_real`. This was done by transforming the pixel y-coordinates into the real world by using a constant factor - it is assumed that 720 pixels along the y-axis correspond to the real-world distance of 30 meters and 700 pixels along the x-axis correspond to the real-world distance of 3.7 meters. After transforming the line coordinates from the pixel to the meter notation, a polynom will be fitted to the available points. The radius is calculated by using the formula presented in the lesson:
```python
left_curverad = ((1 + (2*left_fit_cr[0]*y_eval + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
right_curverad = ((1 + (2*right_fit_cr[0]*y_eval + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.
In order to project the detected lane onto the image, the area between the left and right lines was filled using the `cv2.fillPoly` function. Then the lane area was unwarped into the original image by using an inverse perspective transformation.

```python
lines_area_image, lane_area_image, left_fit_new, right_fit_new, ploty, left_curverad, right_curverad, lane_offset = search_around_poly(warped, left_fit, right_fit)
unwarped = cv2.warpPerspective(lane_area_image, Minv, (img.shape[1], img.shape[0]))
colored_lane = cv2.addWeighted(img, 1, unwarped, 0.5, 0)
```

![alt text][image9]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./output_video/project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The biggest issue so far was related bad fitting due to a strong influence of outliers, especially on the damaged road as gradient thresholding was used along with color thresholding. Later, I decided to find a way to ensure "cleaner" images after thresholding - for this purpose I used the b channel of the LAB color scheme for a better detection of yellow lines and the l channel of the HLS scheme for detection of white lines. Even though, white lines are still detected in a reduced range. Here it might make more sense to threshold the far and near range separately for each other and with different threshold values.

The proposed algorithm might be still vulnerable to outliers. To tackle this, robust fit approaches can be tried (e.g. RANSAC). Outlier analysis can generally be a good approach here. Tracking some further parameters (curvature, lane width, slope) as well mirroring a stable line onto the opposite site in case if the other line was not detected/was implausible could add more to the robustness of this approach. Nevertheless, applying machine and deep learning to the problem of lane detection gains more and more attention due to the scalability of the ML approaches. Selected papers on lane detection with ML can be found here: https://paperswithcode.com/task/lane-detection/codeless.

