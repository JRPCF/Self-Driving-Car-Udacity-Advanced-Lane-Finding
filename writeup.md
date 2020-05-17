## Advanced Lane Finding - Udacity Project

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

[image1]: ./test_images/straight_lines2.jpg "Original Image"
[image2]: ./camera_cal/calibration1.jpg "Original Calibration Image"
[image3]: ./output_images/undistorted_calibration1.jpg "Undistorted Calibration Image"
[image4]: ./output_images/undistorted_straight_lines2.jpg "Undistorted Image"
[image5]: ./output_images/thresholded_binary_straight_lines2.jpg "Binary Thresholded Image"
[image6]: ./output_images/birds_eye_view_straight_lines2.jpg "Birds Eye View Image"
[image7]: ./output_images/polynomial_fitted_straight_lines2.png "Polynomial Fitted Image"
[image8]: ./output_images/final_straight_lines2.jpg "Final Image"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the method `get_distortion_parameters()` in the second code cell of the IPython notebook located in "./Advanced-Lane_Finding_Pipeline.ipynb". 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  

Here is the original image:
![alt text][image1]

Here is the a calibration image accompanied by the same image undistorted by the `cv2.undistort()` function:
![alt text][image2]
![alt text][image3]

### Pipeline

I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

On the 7th code cell I define the `get_binary_threshold(img,mtx,dist,thresh)` where I pass `dist` and `mtx` acquired from the `get_distortion_parameters()` function and pass the threshold of `thresh= (15, 200)` and the original image.
In it I first copy the image and then undistort it using `cv2.undistort()`.
Then I convert the RGB undistorted image to a HLS image using `cv2.cvtColor()` and extract the S channel to create a new image.
I then take the Sobel operator on the x-axis using `cv2.Sobel()`, take the absolute value of the output image, and finally scale them by putting them such that the max value in the sobel image corresponds to 255 pixel value and the minimum to 0. Finally I select the pixels between the thresholds and set them to 1 and set all other pixels to 0 such that I get a binary thresholded version of the previous image. I then return that output.

Here is an example of the previous image after this step in the pipeline
![alt text][image5]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I then wrote a method `get_birds_eye_view(img)` that takes an input image and transforms it.
To write this method i took a copy of a picture where a car was in the center of a straight road and drew a trapezoid that I knew should correspond to a lane. I then used that as the src array. 

My src array had the following points (where h is the height of the image, w is the width of the image, and s is the y coordinate of the horizon): (w/2-50,s),(w/2+50,s),(w/2-450,h),(w/2+450,h)

My dst array has the following points: (w/2-300,0),(w/2+300,0),(w/2-300,h),(w/2+300,h)

I then get the transformation matrix and its transpose using the `cv2.getPerspectiveTransform()` function.

Finally I transform the image using `cv2.warpPerspective()` and return the image and the transpose matrix

Here is an example of the previous image transformed to a birds eye view

![alt text][image6]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I then, used a Line class, and 3 functions to get the polynomial for the left and right lane.

The `sliding_window(img, nwindows = 9, margin = 100, minpix = 50)` function is very similar to the one in the udacity quiz and uses a histogram to divide the left and right lanes and then uses a sliding window to incrementally get the pixels contained in each line. These points are returned in four arrays `leftx, lefty, rightx, righty` which contains the x coordinates and y coordinates of all points on the left lane and on the right lane.

The `from_prior(img,left,right,margin=100)` function is also very similar to the one in the udacity quiz and it takes the polynomials for the left and right lane discovered in the previous image and searches for the points in the lanes in areas around the polynomial. Once again the function returns four arrays `leftx, lefty, rightx, righty` which contains the x coordinates and y coordinates of all points on the left lane and on the right lane.

The next function is `fit_polynomial(img,left,right,ym_per_pix = 30/720, xm_per_pix = 3.7/700)` this function accepts an image, a Line object representing the left line and a Line object representing the right line. If the Line objects both previously detected polynomials, the method uses `from_prior()` to find points. Else, the method uses `sliding_window()` to find points. It then tried to fit a second-degree polynomial to each line using `np.polyfit()`. If both polynomials are found the Line objects parameters are updated to include the data. If not, the Line objects are updated to show that. Although the objects are changed in place. The method returns references to both objects.

Here is an example of the previous image with the polynomials overlaid.
![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in the `fit_polynomial()` function.
To get the curvature I used the equation (assuming polymial of for Y = Axˆ2 + Bx + C and ym is the ratio of pixel to meters)

curvature  = ((1 + (2 A Y ym + B)ˆ2)ˆ1.5) / |2A|

To get the distance I used the equation (assuming polymial of for Y = Axˆ2 + Bx + C and xm is the ratio of pixel to meters and X is the first x (where y = height))

distance to lane = (AX^2 + BX+ C)*xm

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I then wrote the method `warp_to_original(img,left,right,M)` where M is the matrix transpose of the original birds_eye_view matrix and left and right are Line objects.

I then generate an image where i fill the polynomial bounded by the polynomials in left and right and the top and bottom borders of the image size. I then warp this image using M and overlay it with reduced opacity onto the original image and return it.

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I found that the methodology to get the birds eye view is very dependent on having a flat plane ahead and is very crude, therefore, I searched for another methodology but could not find it. 

I think the binary thresholding in my pipeline is weak and this was confirmed in the challenge video. If I had also used a Black and White image combined with the S channel of the HLS and tuned my parameters better I would probably have a better result. The combination of a mask and different parameters might also be helpful.

This issue is evident in the output video where, on the side of the dashed lane, the binary thresholding has difficulty recognizing the lane closer to the horizon leading to wobbles in the video.
