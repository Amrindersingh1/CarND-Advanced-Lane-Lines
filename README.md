# Advanced Lane Detection
## Overview
Detect lanes using computer vision techniques. This project is part of the [Udacity Self-Driving Car Nanodegree](https://www.udacity.com/drive), and much of the code is leveraged from the lecture notes.

The following steps were performed for lane detection:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.


## How to run
* Run `AdvancedLaneFindingProject.ipynb`
* Individual operations can be viewed inline in jupyter notebook
* project video will be generated at end and is apended as html video to view in jupyter notebook itself

To run the lane detector on arbitrary video files, update the last few lines of 'line_fit_video.py'.

## Camera calibration
The camera calibration was done using the chessboard images in './camera_cal/calibration*.jpg'. 
The following steps were performed for each calibration image:

* Read the image
* Convert image to grayscale
* Find chessboard corners with OpenCV's `cv2.findChessboardCorners`
* Draw chessboard corners with OpenCV's `cv2.drawChessboardCorners`
* objectpoints and imagepoints were stored for future reference
* Board was assumed to be 9*6

![](output_images/chesscorners.PNG)

After the above steps were executed for all calibration images: 
* `undistort()` method is used that lies in  `AdvancedLaneFindingProject.ipynb`
* distortion matrices were calculated using `cv2.calibrateCamera`
* undistort image using `cv1.undistort`

![temp](output_images/undistorted.PNG)

Here is the actual image undistorted via camera calibration:
![](output_images/undistortedlane.PNG)

### Prespective transform
* `prespectiveTransform()` method is used that lies in  `AdvancedLaneFindingProject.ipynb`
* Given the thresholded binary image, the next step is to perform a perspective transform. The goal is to transform the image such that we get a "bird's eye view" of the lane, which enables us to fit a curved line to the lane lines (e.g. polynomial fit). Another thing this accomplishes is to "crop" an area of the original image that is most likely to have the lane line pixels.

* To accomplish the perspective transform, I use OpenCV's `getPerspectiveTransform()` and `warpPerspective()` functions

Here is the example image, after applying perspective transform:

![](output_images/prespective.PNG)

### Sobel Gradient Magnitude Threshold
* Next step was to calculate sobel gradient magnitude threshhold
* `mag_thresh()` method is used that lies in  `AdvancedLaneFindingProject.ipynb`
* A function that applies Sobel x and y, then computes the magnitude of the gradient and applies a threshold
* sobel_kernel was defaulted to 3 and mag_thresh to (20,255)
* `cv2.sobel()` was used to calculate sobel gradient

Here is the output from above process:
![](output_images/sobelmagnitude.PNG)


### Sobel Gradient Direction Threshold
* Next step was to calculate sobel gradient direction threshhold
* `dir_threshold()` method is used that lies in  `AdvancedLaneFindingProject.ipynb`
* A function that applies Sobel x and y, then computes the direction of the gradient and applies a threshold
* sobel_kernel was defaulted to 15 and thresh to (0,0.1)
* `cv2.sobel()` was used to calculate sobel gradient
* `np.arctan2(abs_sobely, abs_sobelx)` is used to calculate the direction of the gradient 

Here is the output from above process:
![](output_images/sobeldirection.PNG)

### combined sobel output
![](output_images/sobelboth.PNG)

### color threshholds
* I ran a image through various threshold color values and found that
* LAB B-channel and HLS S-channel was best fit
* added two methods `lab_bthresh()` and `hls_lthresh()` to do same

### Threshold binary image
* The next step was to connect all the previous operations to create a threshold binary image
* following steps were performed for pipeline:
  * undistort()
  * prespectiveTransform()
  * mag_thresh()
  * dir_threshold()
  * img_lthresh()
  * img_bthresh()
  * combined the sobel and color threshold to generate final binary image
  * All the operation performed are already explained above
* Below is the ouput of pipeline

![](output_images/pipeline.PNG)

### Polynomial fit
Given the warped binary image from the previous step, I now fit a 2nd order polynomial to both left and right lane lines. In particular, I perform the following:

* Calculate a histogram of the bottom half of the image
* Partition the image into 9 horizontal slices
* Starting from the bottom slice, enclose a 200 pixel wide window around the left peak and right peak of the histogram (split the histogram in half vertically)
* Go up the horizontal window slices to find pixels that are likely to be part of the left and right lanes, recentering the sliding windows opportunistically
* Given 2 groups of pixels (left and right lane line candidate pixels), fit a 2nd order polynomial to each group, which represents the estimated left and right lane lines

The code to perform the above is in the `line_fit()` function of 'line_fit.py'.

Since our goal is to find lane lines from a video stream, we can take advantage of the temporal correlation between video frames.

Given the polynomial fit calculated from the previous video frame, one performance enhancement I implemented is to search +/- 100 pixels horizontally from the previously predicted lane lines. Then we simply perform a 2nd order polynomial fit to those pixels found from our quick search. In case we don't find enough pixels, we can return an error (e.g. `return None`), and the function's caller would ignore the current frame (i.e. keep the lane lines the same) and be sure to perform a full search on the next frame. Overall, this will improve the speed of the lane detector, useful if we were to use this detector in a production self-driving car. The code to perform an abbreviated search is in the `tune_fit()` function of 'line_fit.py'.

Another enhancement to exploit the temporal correlation is to smooth-out the polynomial fit parameters. The benefit to doing so would be to make the detector more robust to noisy input. I used a simple moving average of the polynomial coefficients (3 values per lane line) for the most recent 5 video frames. The code to perform this smoothing is in the function `add_fit()` of the class `Line` in the file 'Line.py'. The `Line` class was used as a helper for this smoothing function specifically, and `Line` instances are global objects in 'line_fit.py'.

Below is an illustration of the output of the polynomial fit, for our original example image. For all images in 'test_images/\*.jpg', the polynomial-fit-annotated version of that image is saved in 'output_images/polyfit_\*.png'.

![polyfit](output_images/polyfit_test2.png)

### Radius of curvature
Given the polynomial fit for the left and right lane lines, I calculated the radius of curvature for each line according to formulas presented [here](http://www.intmath.com/applications-differentiation/8-radius-curvature.php). I also converted the distance units from pixels to meters, assuming 30 meters per 720 pixels in the vertical direction, and 3.7 meters per 700 pixels in the horizontal direction.

Finally, I averaged the radius of curvature for the left and right lane lines, and reported this value in the final video's annotation.

The code to calculate the radius of curvature is in the function `calc_curve()` in 'line_fit.py'.

### Vehicle offset from lane center
Given the polynomial fit for the left and right lane lines, I calculated the vehicle's offset from the lane center. The vehicle's offset from the center is annotated in the final video. I made the same assumptions as before when converting from pixels to meters.

To calculate the vehicle's offset from the center of the lane line, I assumed the vehicle's center is the center of the image. I calculated the lane's center as the mean x value of the bottom x value of the left lane line, and bottom x value of the right lane line. The offset is simply the vehicle's center x value (i.e. center x value of the image) minus the lane's center x value.

The code to calculate the vehicle's lane offset is in the function `calc_vehicle_offset()` in 'line_fit.py'.

### Annotate original image with lane area
Given all the above, we can annotate the original image with the lane area, and information about the lane curvature and vehicle offset. Below are the steps to do so:

* Create a blank image, and draw our polyfit lines (estimated left and right lane lines)
* Fill the area between the lines (with green color)
* Use the inverse warp matrix calculated from the perspective transform, to "unwarp" the above such that it is aligned with the original image's perspective
* Overlay the above annotation on the original image
* Add text to the original image to display lane curvature and vehicle offset

The code to perform the above is in the function `final_viz()` in 'line_fit.py'.

Below is the final annotated version of our original image. For all images in 'test_images/\*.jpg', the final annotated version of that image is saved in 'output_images/annotated_\*.png'.

![annotated](output_images/annotated_test2.png)

## Discussion
This is an initial version of advanced computer-vision-based lane finding. There are multiple scenarios where this lane finder would not work. For example, the Udacity challenge video includes roads with cracks which could be mistaken as lane lines (see 'challenge_video.mp4'). Also, it is possible that other vehicles in front would trick the lane finder into thinking it was part of the lane. More work can be done to make the lane detector more robust, e.g. [deep-learning-based semantic segmentation](https://arxiv.org/pdf/1605.06211.pdf) to find pixels that are likely to be lane markers (then performing polyfit on only those pixels).
