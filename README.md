## **Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/undistorted.png "Undistorted"
[image2]: ./output_images/undistorted_lane.png "Road Transformed"
[image3]: ./output_images/thresholded.png "Binary Example"
[image4]: ./output_images/warped.png  "Warp Example"
[image5]: ./output_images/sliding_window.png "Sliding Window"
[image6]: ./output_images/histogram.png "Histogram"
[image7]: ./output_images/prev_frame.png "Prev Frame"
[image8]: ./output_images/colored.png "Output"
[video1]: ./project_video_out.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "Advanced-Lane-Finding.ipynb"  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

This is an example of applying distortion correction to a dashcam image. The correction is subtle, you can perceive it on the hood of the car.
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image.  Here's an example of my output for this step.

![alt text][image3]

I experimented with different thresholding and finally settled on these steps.
1. Apply a 3x3 sobel kernel along `x` and threshold the magnitude between 10 and 200
2. Apply a 3x3 sobel kernel, calculate the direction of gradient and threshold between 30 deg and 90 deg.
3. Convert to HLS color space and threshold S channel between 100 and 255 to detect white and yellow and L channel between 100 and 255 to avoid dark regions.

Final mask was based on combining steps 1 and 2 with *OR* and step 3 with *AND*

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in the section *Perspective Transform*. I chose the hardcoded source and destination points as below:

| Source        | Destination   |
|:-------------:|:-------------:|
| 550, 480      | 250, 0        |
| 190, 720      | 250, 720      |
| 1120, 720     | 1000, 720      |
| 690, 450      | 1000, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The functions `get_hist_and_xbase`, `sliding_window` and `detect_line_from_previous_frame`, which identify lane lines and fit a second order polynomial to both right and left lane lines, are in the sections *Histogram*, *Sliding Window* and *Use detected lines for future frames*. At first, this computes a histogram of the bottom half of the image and finds the local maxima of the left and right halves of the histogram. The function then identifies ten windows from which to identify lane pixels, each one centered on the midpoint of the pixels from the window below and follows the lane lines up to the top of the binary image. This speeds processing by only searching a small portion of the image. Pixels belonging to each lane line are identified and the Numpy polyfit() method fits a second order polynomial to each set of pixels. The image below demonstrates how this process works:

![alt text][image5]

This is the histogram calculated before the sliding window starts from the base.

![alt text][image6]

The method `detect_line_from_previous_frame` bypasses the sliding window search and uses the lanes detected in the previous frame and only searches for pixels within a certain range of it. The image below demonstrates this - the green shaded area is the range from the previous fit, and the yellow lines and red and blue pixels are from the current image:

![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature is based upon this [tutorial](http://www.intmath.com/applications-differentiation/8-radius-curvature.php) and calculated in the code cell with the method `radius_and_offset`:

curve_radius = `((1 + (2*fit[0]*y_eval*y_meters_per_pixel + fit[1])**2)**1.5) / np.absolute(2*fit[0])`
Here, fit is the coefficients in y. y_eval is the y position where curvature is based (the bottom, position of the car in the image). y_meters_per_pixel converts from pixels to meters.

The position of the vehicle with respect to the center of the lane is:

lane_center_position = `(r_fit_x_bottom + l_fit_x_bottom) /2`
center_dist = `(car_position - lane_center_position) * x_meters_per_pix`
r_fit_x_bottom and l_fit_x_bottom are the x of the right and left fits at the bottom of the image. The car position is the difference between these intercept points and the image midpoint.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in methods `colorLane` and `overlayText`.  Here is an example of my result on a test image:

![alt text][image8]
---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_out.mp4)
![Project Video][video1]
---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The problems I faced were with lighting condition (too bright, too dark), shadows and other markings on the road that can be confused with the lanes. I had to experiment with different thresholds for the L channel and the gradient magnitude. Ligting conditions (snow, shadows) and bright white/yellow cars next to lanes can cause this pipeline to fail. Smoothing the fits from last `n` fits helped.

A better thresholding scheme and a better way of rejecting fits that deviate too much from the previous fits will be some areas of improvement for this pipeline.
