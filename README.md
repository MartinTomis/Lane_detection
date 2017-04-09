# Lane_detection


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

[//]: # (Image References)

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

I first calibrated the camera using 20 chessboard images for calibration. It turned out that not all images had 9x6 corners (because some of the images are cut-off close to the last corner). So the final calibration matrix is computed using 17 images.

Using information about these images, cv2.calibrateCamera() is used to calculate distortion coefficients and some other outputs required to calculate undistorted images. I do this in my "perspective_transform" function, using cv2.undistort() function.



The code for this step is contained in the first code cell of the IPython notebook located in "./examples/example.ipynb" (or in lines # through # of the file called `some_file.py`).  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
Here is an example of an original image (straight_lines2.jpg) and the same image after applying "undistorting".
![alt tag](https://github.com/MartinTomis/Lane_detection/blob/master/original_vs_undistorted.png)


####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.


Note: I do this step on a "warped" image, so it does not follow immediately the preceding step.

I use mine function gradient_detection() to create a binary image. 
The function first converts images to the HLS color space and then uses both gradient thresholding (using only x-derivative) and color-thresholding (Saturation-channel). Initially I used only gradient-thresholding, but this did not work well for the part of the road with lighter track, where there is lower contrast between the line and the road.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Applying the perspective transform requires identifying 4 points (if cv2.getPerspectiveTransform() is used) in the original image and 4 new points, forming a rectangle, so that the original points are essentially "stretched" to allign with these new points. All other points in lying in between the 4 corner points of the original image, will also be "stretched". The entire transformation changes dimensions while maintaining that straight lines remain straight.

To identify the relevant region, I used defined function region_of_interest() which takes image and 4 vertices as an input and produces the same image with the are outside of the relevant region shaded. The same vertices will be later used to create the perspective transformation, so I then played a bit with the setting of the vertices, to ensure that the results are good. 

Here is the final setting: 
![alt tag](https://github.com/MartinTomis/Lane_detection/blob/master/region_of_interest.png)
The code for my perspective transform includes a function called `warper()`, which appears in lines 1 through 8 in the file `example.py` (output_images/examples/example.py) (or, for example, in the 3rd code cell of the IPython notebook).  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?
After the bird's eye view transformation, the I apply mine function gradient_detection to create a binary image. 
The function uses both gradient thresholding (using only x-derivative) and color-thresholding. After the warped binary picture is created, lines are identified in it. The treatment is different for the first frame and for remaining frames.


For the first frame of a video, a separate treatment is applied. Function "sliding_window", based on code in lecture videos, initially creates a histogram showing the concentration of pixels of the binary image for different values of x. If, as we do in the project video, we start at a reasonably straight segment of a road and hence identify 2 lines, the function identifies 2 clusters of pixels in the data and then slides along the y-axis, from bottom to top. With each shift of the sliding window, though, the function may adjust even the x position, if a new large cluster is identified. Within each window, "hot" pixels are identified and their x ans y indices are appended to create 2 vectors used to fit a 2nd-order polynomial. As some of the lines are nearly vertical, x is out dependent variable.

![alt tag](https://github.com/MartinTomis/Lane_detection/blob/master/sliding%20window%20output.png)

Three coefficients for each of the 2 lines are fitted (square term, linear term and an intercept) and passed to mine function find_lines(), applied to all but the first image. The function is again heavily based on the code shown in lectures.

This time, it is not necessary to use histogram to identify "x-coordinates" where the line pixels should lie. Instead, the starting pixels whose coordinates are regressed are identified in the area around fitted curves - using "past" coefficients.

I tried multiple options of what should be the "past" coefficients:
* Coefficients from the 1st frame - disadvantage is that is once the line in the current frame is very curved, the pixels may not lie in the block centered around the fitted line. As a result, the predictions are quite unstable. Advantage is that there is always a reasonable starting point (provided the first frame is "nice").
* Smoothed/weighted coefficients from the immediately preceding frame - this really smooths resuls, however the disadvantage is that if in few preceding frames there is "irregularity" and the fitted curve by mistake changes shape dramatically, the line may essentially dissapear - this issue can be somwhow mitigated by assigning less weight to the most recent observations - with alpha=0.5 I observed a serious problem, for alpha=0.1, there was no such problem. I in the end go for this option in the submitted video, but I would probably prefer the 1st option in a more general setting.






code i
Then I did sdentifies ome other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image5]

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

To calculate the curvature, I use the formula shown in lecture. The result is very sensitive to changes in the estimated parameters. To decrease this sensitivity, I again apply smoothing to the results - this time by calculating exponentially weighted average of the curvature radius. I apply more smoothing than for the line plotting, as the curvature formula features higher powers and a ration - both decreasing stability of the estimate. I hence use alpha of 0.1.

To calculate the distance from the center, I calculate the x-value for both projected lines, at the maximum value of y (bottom of the image). I denote these xl and xr, for the left and right line, respectively. The idea then is simple. Since the image width is 640 pixels, the car is to the right of the center if the mid-point between xl and xr is smaller than 640. To get distance in meters, I rescale the distance between the mid-point and 640 by 3.7/700 (3.7 is approximately the true distance between lines, 700 is the same distance in pixels).



    ym_per_pix=30/720
    xm_per_pix=3.7/700
    left_fit_cr = np.polyfit(lefty*ym_per_pix, leftx*xm_per_pix, 2)
    right_fit_cr = np.polyfit(righty*ym_per_pix, rightx*xm_per_pix, 2)

    #print(xl*xm_per_pix, xr*xm_per_pix)
    off_center=(((xr + xl) / 2.0) - 640) * xm_per_pix
I did this in lines # through # in my code in `my_other_file.py`

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.



I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

