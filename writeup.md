## Writeup

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

[image1]: ./output_images/calib_img.jpg "Calibration image"
[image2]: ./output_images/test_undist_img.jpg "Undistorted calibration image"
[image3]: ./test_images/test4.jpg "Image"
[image4]: ./output_images/undist6.jpg "Undistorted image"
[image5]: ./output_images/source_points.png "Source points"
[image6]: ./output_images/destination_points.png "Destination points"
[image7]: ./output_images/binary6.jpg "Binary example"
[image8]: ./output_images/binary_warped6.jpg "Binary warped example"
[image9]: ./output_images/poly6.jpg "Fit Visual"
[image10]: ./output_images/final_output6.jpg "Output example"
[video1]: ./output_videos/output_project_video.mp4 "Project video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./find\_lanes.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `object_points` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `image_points` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `object_points` and `image_points` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]
*Calibration image*

![alt text][image2]
*Undistorted calibration image*

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Distortion correction could be applied to the road images the same way it was tested previously on one of the calibration images: using the `cv2.undistort()` function. Here is the outcome of undistortion on one of the road test images:

![alt text][image3]
*Test image*

![alt text][image4]
*Undistorted test image*

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines 67 through 115 in the 5th cell of `find_lanes.ipynb`).  Here's an example of my output for this step. 

![alt text][image7]
*Thresholded image*

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in lines 117 through 118 in the fifth cell of `find_lanes.ipynb`.  The `warp()` function takes as inputs an image (`img`), as well as the inversion matrix (`M`), which is calculated in the third cell of the file. The inversion matrix (`Minv`), which is used for inverting the perspective transform with the `unwarp()` function, is also calculated in that cell. I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[img_shape[0]-568,img_shape[1]/2+100], # top right
    [img_shape[0]-100,img_shape[1]], # bottom right
    [100,img_shape[1]], # bottom left
    [568,img_shape[1]/2+100]]) # top left
dst = np.float32(
    [[img_shape[0]/2+400,0], # top right
    [img_shape[0]/2+400,img_shape[1]], # bottom right
    [img_shape[0]/2-400,img_shape[1]], # bottom left
    [img_shape[0]/2-400,0]]) # top left
```

This resulted in the following source and destination points:

| Source        | Destination   | Orientation   |
|:-------------:|:-------------:|:-------------:|
| 712, 460      | 940, 0        | top right     |
| 1180, 720     | 940, 720      | bottom right  |
| 100, 720      | 340, 720      | bottom left   |
| 568, 460      | 340, 0        | top left      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image5]
*Source points*

![alt text][image6]
*Destination points*

After this step, perspective transform could be applied to the binary images:

![alt text][image8]
*Warped binary image*

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial.

After these two steps I identified the lane pixels by using the sliding window search with 9 windows and a histogram of the wapred binary image, but with a few changes when compared to the exapmle provided in the lessons. First, if the nonzero points in a window are highly deviated, only the more centered ones are taken into consideration. Then, if there are enough found pixels, the next window is centered to the most frequent value of the pixels, not the mean value. Finally, if the image is a part of a video stream, lane pixels are detected in previous frames, and the intercept value of the model doesn't deviate too much from the previous one, the next lane pixels are simply gotten around the previously fitted polynomial.

After the left and right lane pixels are detected, I fit a 2nd order polynomial to both of them, like so:

![alt text][image9]
*Fitted polynomials*

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I used a helper function `curverad()` (lines 234-235 in the fifth cell if `find_lanes.ipynb`) to calculate the curvature of both of the lanes (lines 349-351 for a single image, lines 404-406 for a stream of images, both in the fifth cell of the file). I then used a weighted average to calculate the actual curvature of the road (lines 353-356 for a single image, lines 408-414 for a stream of images, again in the fifth cell of the file). The offset calculation for both single images and a stream of images was done in lines 416-432 of the fifth cell.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

When I had the fitted lines, I created a blank images where I filled the lane area with green, drew the left lane with red, and the right with blue, combined and unwarped these, and merged them with the original (undistorted) image. Finally, I added text about the radius of curvature and the offset of the vehicle to the top of the image. Here is an example of the final output:

![alt text][image10]
*Output image*

---

### Pipeline (video)

#### 1. Provide a link to your final video output. Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

I created a `Line()` class in order to make the lane finding algorithm more robust and smooth for a video stream (lines 256-311 in the fifth cell of `find_lanes.ipynb`). This class keeps track of important information about the lanes, such as the 10 most recent fits, used for calculating the running average of the fits.

Here's a [link to my video result](./output_videos/output_project_video.mp4).

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

My pipeline performs poorly when the edge of sidewall's shadow is close to either side of the lane, in extreme brightness, and when the road is not flat enough, as seen from the output videos `output_challenge_video.mp4` and `output_harder_challenge_video.mp4` located in the folder "./output\_videos/". Also, if the first fit in the video stream is not very good, then the pipeline has trouble correcting its mistake.

When it comes to the shadow and the extreme brightness problems, fine tuning and clever combining of the parameters in the color and gradient thresholding phase would most likely vastly improve the pipelines performance. This would also decrease the likelyhood of a bad fit in the beginning of a video stream. AS for the (un)flatness problem, this would most likely require changing the perspective transform phase, so that the source and destination points and therefore the transform matrices are calculated individually for each of the images. This would require some very clever selection of the source and destination points, which is probably very difficult.

Other changes to improve the performance of the pipeline include checking for robustness of the fits before the pixel detection phase, more clever lane pixel detection, and experiments with general parameter tuning and combining in the thresholding phase.
