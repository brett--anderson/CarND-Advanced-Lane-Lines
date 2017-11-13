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

[image1]: ./output_images/undistort_comparison.png "Undistorted Chessboard"
[image2]: ./output_images/distort_corrected.png "Undistored Test Image"
[image3]: ./output_images/binary.png "Binary Example"
[image4]: ./output_images/perspective.png "Warp Example"
[image5]: ./output_images/poly_lines.png "Fit Sliding Window"
[image6]: ./output_images/poly_lines_margin.png "Fit Margin"
[image7]: ./output_images/real_perspective.png "Detected Lines on Original Image"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the third code cell of the IPython notebook located in "./P3.ipynb"

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![undistored chessboard image][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

After determining the object points and image points I'm able to calibrate the camera, returning a set of parameters that allow me to perform corrections to the camera's biases in other images. I've applied the correction to a test image shown below and dispayed the corrected image as well below that.

![undistorted test image][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used the HLS colorspace as I found it very effective during the first project, basic lane finding. HLS isolates the Hue component of the image so it's much easier to mask the lane lines by their yellow and white hues while still having some invariance to shadows that have more of an impact on the lightness and saturation channels. Once in the HLS space I mask only pixels inside the narrow Hue and board Lightness and Saturation thresholds. This does a very good job of isolating the lane lines.

Once the lane line pixels are mostly isolated I applied the sobel gradient filter to the image and found I had the best result highlighting a pixel in the binary image if that pixel appear in both the X and Y gradient filters or the magnitude and direction gradient filters. I've illustrated each of these filters very clearly in "./P3.ipynb", the following is a final binary image:

![Binary Example][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I used the following points to perform the perspective warp on my test image

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 575, 464      | 350, 0        | 
| 707, 464      | w-350, 0      |
| 258, 682      | 350, h        |
| 1049, 682     | w-350, h      |

I calculated the Source points by drawing a trapazoid from the hood of the car to a point on the horizon with a decent lane line thickness, with the sides tracing the lane lines. I chose the destination points to effectively exclude the lines from the other lanes, much like a region of interest mask would do.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I followed the code from the quizes to implement an algorithm for fighting a polynomial to the detected lane pixels. This algorithm looks at the histogram for the bottom of the image and finds two largest peaks that indicated the greatest concentrations of pixels, i.e. the lane lines. Once I have the center of these peaks as a starting point I search a smaller window in the region above and repeat the histogram process to find the lane pixels for that segment of the image. If the pixels have shifted to the left or right I move the window to center around the pixels so that the next window can start above that point and be most likely to discover new lane lines.

![Fit Sliding Window][image5]

Once I have the pixels identified for the lane lines I can fit a second order polynomial that minimises the distance from the line to each of the points. Once this line is determined I can create a region on eitherside of the line in which to search for the next line, which will be faster and potentially more acurate than the sliding window. In practise I will do this in each successive video frame, unless a good line was not found in which case I will fall back to the sliding window approach. Below is an image of the search margin for successive video frames.

![Fit Margin][image6]


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I followed the code provided in the quizes to calculate radius and the distance of the vehicle from centre. The distance is simply the difference between the center of the image, where the center of the car / camera must be and the midpoint between where the two lane lines intercept the bottom of the image. A formula for calculating the radius was provided in the quizes which I used as well. I also had to map from pixel space to real world space by mapping the USA standard distance between lane lines and the number of pixels that separates them in the image

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The following is an example of the identified lane lines and road region being projected back into the original perspective.

![Detected Lines on Original Image][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I tried my pipeline on the challenge videos and it failed pretty badly. Unfortunately I'm out of time to try and perfect the process but it's something I hope to return to at the end of this course. I think I could experiment more with the sobel filter values to potentially get better lane detection in shaded regions. What would probably be more useful though is including some common sense restraints to the possible lane lines that I'm considering. I.e. I could make sure both lanes are always within a specific distance of each other. I could also use the better detected lane line to have a stronger influence on the other lane line and the radius of curvature calculation.
