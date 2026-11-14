# Day 32: Cameras

*100 Days of Robotics Engineering, Module 4: Sensors and Actuators*

**Time:** about 4.5 hours
**Level:** intermediate. You need Day 30 (range sensing and its field of view), Day 29 (frames and the idea of a reference measurement) and basic Python. Today a camera stops being a picture and becomes a measuring instrument.

**By the end of this day, you will have:**

- A calibrated intrinsic matrix for one camera, with its reprojection error stated in pixels
- A measured frame rate and exposure setting for the camera under two lighting conditions
- A distance estimate to a target of known size, checked against a tape measure at four distances
- A written camera frame convention that links the image axes to the robot's base frame

**What you need:**

- A Raspberry Pi 4 or Raspberry Pi 5 with the Raspberry Pi Camera Module 3, or a USB webcam with at least 640 by 480 resolution. The Camera Module 3 costs about $25 and the Raspberry Pi board about $35 to $80 depending on the model and memory. A USB webcam costs $15 to $60, and a 1080p model is more useful for this lesson than a 480p one. Check current prices.
- Python 3.12 with OpenCV (the opencv-python package, version 4.10 or newer), NumPy 2.1 or newer and Matplotlib 3.9 or newer. OpenCV is free and open source under the Apache 2.0 licence.
- A printed checkerboard calibration target with a known square size. Print it on rigid card, measure the square size with calipers, and keep it flat. A 9 by 6 internal-corner pattern with 25 mm squares is a common choice.
- A tape measure, a ruler and a target object of known height, such as an A4 sheet, which is 297 mm tall.
- A light meter app on a phone, free, to record the scene brightness, or a simple lux meter at about $10.
- Optional: an Intel RealSense depth camera for the depth section of the project. RealSense availability and product lines have changed in recent years, so check the current maker's site before you buy.

**Prerequisites:**

- Day 30: the field of view of a range sensor, the beam width at a distance and the sign of an error
- Day 29: the difference between a measured quantity and its estimate, and repeated runs
- Day 31: the idea of a sensor frame and its orientation relative to the robot
- Python: running a script, reading an image array and writing a CSV file

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [How cameras developed](#how-cameras-developed)
- [Glossary](#glossary)
- [The pinhole camera model](#the-pinhole-camera-model)
- [Resolution and field of view](#resolution-and-field-of-view)
- [Exposure, frame rate and motion](#exposure-frame-rate-and-motion)
- [Depth from stereo](#depth-from-stereo)
- [Calibration](#calibration)
- [Camera frames](#camera-frames)
- [Errors and limits](#errors-and-limits)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a calibrated camera with a measured distance check](#project-a-calibrated-camera-with-a-measured-distance-check)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A camera is the richest sensor on most robots. A single image holds millions of measurements of light, and the robot's software can find lanes, people, shelves, markers and the edges of a doorway from it. The same image can mislead the robot badly. A camera with the wrong exposure sees a dark wall as black, a camera that is out of calibration measures a distance wrong by several percent, and a camera with a slow frame rate reports the world a fraction of a second late.

Today you treat the camera as a measuring device. You learn the pinhole model that turns a pixel into a direction, the calibration that finds the model's numbers for your lens, and the exposure and frame-rate settings that decide whether the image is useful at speed. Then you check the camera's distance estimate with a tape measure. The skills carry over to every vision system in a robot, from a line follower to a warehouse picker.

---

## How cameras developed

| Period | What happened | Why it matters for robots today |
|---|---|---|
| Around 1550 to 1600 | The camera obscura, a darkened room or box with a small hole, is described and used by artists and natural philosophers. | The pinhole model you use today is the geometry of this device. |
| 1826 to 1830s | Nicéphore Niépce and then Louis Daguerre produce the first permanent photographs on chemically coated plates. | The idea of capturing light as a stored image started here. |
| 1880s to 1900s | Roll film and the Kodak camera make photography common, and lenses are designed to correct distortion. | Lens distortion models come from optical design of this period. |
| 1960s to 1970s | Charge-coupled device (CCD) image sensors are invented, and they become the standard for digital cameras. | The digital sensor replaced film and made computer vision possible. |
| 1990s to today | CMOS image sensors become cheap and fast, and computer vision libraries such as OpenCV spread to robotics and phones. | A $25 camera module now runs a full vision pipeline. |

The optical geometry is almost unchanged since the camera obscura. What changed is the sensor, which now records a number for each pixel, and the software, which reads those numbers.

---

## Glossary

- **Pixel:** the smallest element of a digital image, which records the light level in one small area of the sensor.
- **Focal length (f):** the distance from the lens to the image plane when the camera focuses on distant objects. In the model, it is measured in pixels for a digital camera.
- **Principal point:** the point where the optical axis meets the image plane, usually near the centre of the image.
- **Intrinsic matrix (K):** a 3 by 3 matrix that holds the focal lengths and principal point of a camera. It maps camera-frame coordinates to pixel coordinates.
- **Extrinsic parameters:** the rotation and translation that place the camera in the world or relative to another frame.
- **Distortion:** the bending of straight lines in the image by the lens. Radial distortion makes lines curve outward or inward near the edges.
- **Exposure:** the amount of light the sensor collects, set by the exposure time, the gain and the lens aperture.
- **Frame rate (FPS):** the number of images captured per second.
- **Rolling shutter:** a sensor read-out that scans the image row by row, so a fast-moving object appears skewed.
- **Global shutter:** a sensor that captures the whole image at once, with no skew.
- **Stereo baseline:** the distance between the optical centres of two cameras in a stereo pair.
- **Disparity:** the horizontal shift of the same point between the left and right images of a stereo pair, in pixels.
- **Reprojection error:** the average distance, in pixels, between a calibration point's measured position and its predicted position from the model.
- **Checkerboard:** a flat pattern of black and white squares with a known size, used as the reference for calibration.

---

## The pinhole camera model

The pinhole model describes how a point in the world lands on the image. A ray from a point in front of the camera passes through the optical centre and hits the image plane. The point's position on the image depends on its position in the camera's frame and on the focal length.

The first principle is similar triangles. A point at position (X, Y, Z) in the camera frame, with Z pointing forward, projects to the image at u = f times X / Z plus the principal point's x coordinate, and v = f times Y / Z plus the principal point's y coordinate. The factor 1 / Z is why distant objects appear smaller and why the image loses depth. Written as a matrix, the mapping is K times the point, followed by division by Z, where K holds fx, fy, cx and cy.

The worked example uses a 640 by 480 camera with a horizontal field of view of 60 degrees. The focal length in pixels is fx = (640 / 2) divided by tan(60 / 2) = 320 / tan(30 degrees) = 320 / 0.5774 = 554 pixels. The principal point is near (320, 240). A point 1 m in front of the camera and 0.1 m to the right appears at u = 554 times 0.1 / 1.0 + 320 = 375 pixels. A point at the same X but 2 m away appears at u = 554 times 0.1 / 2.0 + 320 = 347.7 pixels. The point moves toward the centre as it recedes, which is the perspective effect the robot must invert to get a distance.

The common mistake is to treat the focal length in the datasheet as the focal length in pixels. The datasheet gives the focal length in millimetres, and the pixel focal length is that value divided by the pixel size, which is in millimetres per pixel. A lens of 3.6 mm on a sensor with 1.4 micrometre pixels has a focal length of 3.6 / 0.0014 = 2571 pixels, which is far from a typical value for a wide-angle webcam. Always compute the pixel focal length from the field of view or from calibration, not from the lens label alone.

The pinhole model has no distortion, so a real lens needs a correction. Calibration finds the distortion coefficients with the intrinsic matrix, and the correction is applied to each frame before measurements are made.

On the job, the intrinsic matrix is stored with every camera in a robot, and the calibration file is versioned with the robot's software. A camera replaced on a robot needs a new calibration, or its measurements shift.

---

## Resolution and field of view

Resolution is the number of pixels in the image, and field of view is the angle the image covers. Together they set how many pixels an object of a given size occupies at a given distance, which decides whether the robot can see it.

The first principle is the angular size of a pixel. A camera with a horizontal field of view of 60 degrees and 640 pixels across has about 60 / 640 = 0.094 degrees per pixel at the image centre. An object of width w at distance Z spans about w / Z radians, which in pixels is w times fx / Z. A 5 cm ball at 1 m, with fx = 554, spans 0.05 times 554 / 1.0 = 27.7 pixels. At 4 m it spans 6.9 pixels, which may be too few for a reliable detection.

The worked example is a detection rule. A robot needs at least 20 pixels across a target to detect it reliably with a common classical detector. The 5 cm ball at 1 m gives 28 pixels, so it is detectable. At 2 m it gives 13.9 pixels, which falls below the rule. The camera's limit is therefore 1.4 m for this ball, not the 4 m that the sensor's field of view might suggest.

The common mistake is to choose the highest resolution for every task. A 4K camera gives four times as many pixels as a 1080p camera, but it also needs four times the processing power and a higher data rate. A robot that only needs to detect a large object at short range is better off with a lower resolution and a higher frame rate.

Field of view also changes with the lens. A wide-angle lens sees more, but its pixels cover less of the scene at a given distance, so small objects are harder to detect. A telephoto lens sees less, with larger pixels on the target, but it has a narrower view, and the robot may miss an object that is close to it. Choose the lens from the detection rule and from the robot's field of view requirement, not from the lens name.

On the job, resolution and field of view are set in the design document of the perception system, and the minimum number of pixels per object is one of the requirements.

---

## Exposure, frame rate and motion

Exposure sets how long the sensor collects light for each frame. It decides whether the image is too dark or too bright, and it trades brightness against motion blur and frame rate. A longer exposure collects more light and blurs moving objects, and a shorter exposure freezes motion and needs more light.

The first principle is the relation between exposure time and motion. An object moving at speed v across the image at distance Z moves v times t / Z radians during an exposure of t seconds. In pixels, the blur is v times t times fx / Z. A robot moving at 1 m/s past an object 2 m away, with fx = 554 and an exposure of 10 ms, gets a blur of 1 times 0.01 times 554 / 2 = 2.8 pixels. That is visible, and it affects edge detection and precise measurement.

The worked example sets the frame rate. The maximum frame rate at a given exposure is limited by the sensor's read-out time, and a camera at 30 frames per second has a frame period of 33 ms. An exposure of 10 ms fits within that period, but an exposure of 40 ms does not, so the camera limits the exposure to the frame period. A camera set to 60 frames per second has a frame period of 16.7 ms, which means the exposure must be less than 16.7 ms.

The common mistake is to leave the camera in automatic exposure. The auto mode changes the exposure with the scene, and a robot that moves from a dark corridor to a bright window sees its images change brightness in a few frames. A detector tuned on one brightness fails on the other. Use a fixed exposure and gain for measurement, and check the image's mean brightness before each test.

Rolling shutters add a second effect. A rolling shutter reads the sensor row by row, so a fast-moving object is skewed. A robot that moves quickly past a pole sees the pole lean. A global-shutter sensor avoids this, at a higher price and often a lower resolution, and the choice matters for fast robots.

In the market, the frame rate and the exposure control are the main specifications for a machine-vision camera, and they are matched to the speed of the conveyor, the robot or the vehicle.

---

## Depth from stereo

A single camera measures direction but not distance. Two cameras separated by a known baseline see the same point at slightly different positions, and the shift between the two images, the disparity, gives the distance. This is stereo vision, and it is the principle behind human depth perception.

The first principle is triangulation by disparity. For two parallel cameras with focal length f in pixels and baseline B in metres, a point at depth Z appears with a disparity d = f times B / Z. Solving for depth gives Z = f times B / d. The depth falls as the disparity rises, so near objects have large disparities and distant objects have small ones.

The worked example uses the numbers from the pinhole section, with a baseline of 6 cm. A point with a disparity of 20 pixels is at Z = 554 times 0.06 / 20 = 1.66 m. The depth resolution at that distance, which is the change in depth for a one-pixel change in disparity, is dZ = Z squared divided by (f times B), which is 1.66 squared divided by (554 times 0.06), or 0.083 m. A one-pixel error in disparity at 1.66 m gives an 8 cm depth error. At 3 m, the same one-pixel error gives a depth error of about 27 cm. The depth error grows with the square of the distance, which is why stereo depth is accurate only at short range.

The common mistake is to use a wide baseline for a short-range robot and a narrow baseline for a long-range one, without checking the result. A narrow baseline gives low disparity and poor depth at range. A wide baseline gives good depth at range but a minimum distance below which the two views no longer overlap. Calculate the baseline from the working range before you buy or mount the cameras.

Stereo also fails on textureless surfaces, such as a white wall or a glossy floor, because the matcher cannot find the same point in both images. It fails on repeated patterns, such as a fence or a tiled floor, because the matcher finds the wrong match. Depth sensors that project their own pattern, such as an infrared dot projector in some RealSense models, avoid the textureless problem, at a cost in sunlight and range.

In the market, stereo cameras are used in outdoor robots and in driver-assistance systems, and depth cameras are used in indoor service robots and in picking systems.

---

## Calibration

Calibration finds the intrinsic matrix and the distortion coefficients of a camera. It works by photographing a flat pattern with known geometry from several angles, finding the corners in each image, and fitting the model's numbers to the observed corners. The fit gives the focal lengths, the principal point and the distortion, and its residual is the reprojection error.

The first principle is that every corner of the checkerboard has a known position on the flat pattern. The camera sees the corner at a pixel position, and the model predicts where the corner should appear for a given pose of the pattern. The calibration adjusts the model's numbers and each pose to minimize the difference between predicted and observed corners. The residual, in pixels, is the reprojection error. A value below about 0.5 pixels is a good calibration for most vision tasks, and a value above 1 pixel means something is wrong, such as a blurred image or a bent pattern.

The worked example is the calibration with a 9 by 6 pattern of 25 mm squares, photographed at about 15 poses. The OpenCV function findChessboardCorners finds the inner corners in each image, and calibrateCamera returns the camera matrix, the distortion coefficients, the rotation and translation for each pose, and the overall reprojection error. The Python code below is the core of the process:

```python
import glob
import cv2
import numpy as np

PATTERN = (9, 6)          # inner corners per row and column
SQUARE_M = 0.025          # square size in metres, measured with calipers

objp = np.zeros((PATTERN[0] * PATTERN[1], 3), np.float32)
objp[:, :2] = np.mgrid[0:PATTERN[0], 0:PATTERN[1]].T.reshape(-1, 2) * SQUARE_M

obj_points, img_points = [], []
size = None
for path in sorted(glob.glob("calibration/*.jpg")):
    img = cv2.imread(path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    size = gray.shape[::-1]
    found, corners = cv2.findChessboardCorners(gray, PATTERN, None)
    if found:
        corners = cv2.cornerSubPix(gray, corners, (11, 11), (-1, -1),
                                   (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 30, 0.001))
        obj_points.append(objp)
        img_points.append(corners)

rms, K, dist, _, _ = cv2.calibrateCamera(obj_points, img_points, size, None, None)
print("RMS reprojection error (px):", rms)
print("K =\n", K)
print("dist =", dist.ravel())
np.savez("camera_calibration.npz", K=K, dist=dist)
```

The `rms` value that calibrateCamera returns is the root-mean-square reprojection error, and it is the number to report. The folder of images must include poses with the pattern tilted, rotated and placed in different parts of the image, because a set of poses that all face the camera does not constrain the distortion well.

The common mistake is to calibrate at one resolution and use the camera at another. The calibration matrix is tied to the image size, and a calibration at 1280 by 720 applied to a 640 by 480 image gives wrong measurements. Calibrate at the resolution you use, and store the size with the file. Another mistake is to use a blurred or a poorly printed pattern. A pattern printed on paper that curls or that has a square size of 24.6 mm instead of 25 mm gives a scale error in every measurement.

In the market, camera calibration is part of the factory test and of every robot's startup check. A camera that has been bumped, refocused or replaced needs a new calibration.

---

## Camera frames

A camera has its own coordinate frame, and the robot has another. The camera's frame is attached to the lens, and its axes are defined by the image. A point in the camera frame must be transformed into the robot's base frame before the robot can act on it, and the transform depends on the camera's mounting.

The first principle is the optical frame convention. The camera's optical frame has Z pointing forward along the optical axis, X pointing right in the image and Y pointing down. This is the frame the pinhole model uses. The robot's base frame, from the ROS REP-103 convention, has X pointing forward, Y to the left and Z up. The two frames differ by a fixed rotation, and the rotation must be applied before the camera's point is used by the rest of the robot.

The worked example is a camera mounted 10 cm above the base, tilted down by 20 degrees, looking forward. A point 1 m in front of the camera, along its optical axis, is at position (0, 0, 1) in the optical frame. To express it in the base frame, the code applies the rotation that converts the optical axes to the base axes, followed by the rotation for the 20-degree tilt, and the translation for the mounting height. The point's height in the base frame is then 0.10 minus sin(20 degrees) times 1, which is about minus 0.24 m, below the camera. The point lies on the floor if the floor is at that height, which is the test a robot uses to find the ground plane.

The common mistake is to confuse the optical frame with the base frame. A robot that uses the optical axes directly for motion drives in the wrong direction, because the optical Z axis points forward, and the base X axis points forward, so the mapping is a permutation of axes with a sign change. Write the transform as a named matrix, and test it with a marker at a known position.

Camera frames are also a source of latency. An image captured at time t may reach the robot's software at t plus 50 ms, and the robot must use the timestamp of the capture, not the time it processes the frame. A robot that uses the wrong time moves to a position that has already changed.

On the job, the camera's extrinsic transform is measured with a calibration target and stored in the robot's transform tree, so that every other sensor and actuator uses the same frame definitions.

---

## Errors and limits

A camera system has several kinds of error, and each one has a clear sign in the data. The three that matter most for a robot are the calibration error, the exposure error and the depth error.

The first principle is that the calibration error is a scale and a distortion error. A wrong focal length scales every measured distance by the same factor, which shows up in the distance check as an error that grows with distance. A wrong distortion term bends the image's straight lines, which shows up at the edges of the frame as a shift that depends on the pixel's position. The distance check and a straight-line test are therefore different tests for different errors.

The worked example is the distance check with a known target. An A4 sheet, 297 mm tall, is placed at 1 m, and the camera measures its height in pixels, say 180 pixels, with the sheet's top and bottom edges detected. The distance estimate is Z = f times H / h, which gives 554 times 0.297 / 180 = 0.914 m. The tape measure says 1.000 m, so the estimate is 8.6 percent short. The error may come from the focal length, which the calibration would correct, or from the sheet's height in the image being detected a few pixels short. A repeat at four distances separates the two: a constant percentage error points to focal length, and an error that changes with distance points to the detector.

The second error is exposure. A dark image gives noisy edges, and a bright image saturates the edges. The camera's histogram shows the problem: a histogram with many values at the top or bottom is clipped. Correct the exposure before you measure, not after.

The third error is depth. A stereo depth estimate has an error that grows with the square of the distance, as the worked example showed. A depth figure with no distance stated is not useful. State the error at each distance in the README.

Camera limits also include latency, motion blur and lighting. A camera at 30 frames per second adds about 33 ms of frame period plus processing time. A camera in direct sunlight or in a dark room is outside its design range, and no algorithm recovers a usable image from a saturated or black frame.

In the market, vision-based robots are tested under the lighting and motion they will meet, and the test report gives the detection rate and the distance error at each condition.

---

## Hands-on exercises

Use the camera on a stable mount, and keep it away from heat sources and from direct sunlight unless the test requires it. A Raspberry Pi and its camera can get warm under a long test, so let the board cool between runs. Cameras and single-board computers are not hazardous at these voltages, but a loose cable can short the Pi's 5 V pin, so check each connection before you power the board.

1. **Capture and check the frame.** Capture 20 frames with the camera at 640 by 480 and print the image size, the mean brightness and the frame rate. Save one frame and open it. Note whether the image is clipped at the top or bottom of the histogram.

2. **Calibrate the camera.** Capture 15 images of the checkerboard at different poses, including tilted and near-edge poses. Run the calibration script and record the RMS reprojection error, the focal lengths fx and fy, the principal point and the distortion coefficients. Repeat with 5 images and compare the result.

3. **Undistort a test image.** Use cv2.undistort with the saved matrix and the distortion coefficients on a photo of a straight-edged object, such as a door frame or a ruler laid straight across the frame. Compare the edges before and after. Measure the bend in pixels at the edge.

4. **Frame rate and exposure.** Set the camera to a fixed exposure and a fixed gain. Record the frame rate and the mean brightness at three exposure times. Repeat under a lamp and under daylight from a window, and record the difference.

5. **Distance check.** Place the A4 sheet at 0.5 m, 1.0 m, 1.5 m and 2.0 m from the camera, and measure the sheet's height in pixels at each distance. Compute the distance from the pinhole formula and compare it with the tape measure. Record the error at each distance.

6. **Frame transform.** Mount the camera at a known height and tilt, and place a marker on the floor at a measured distance in front of it. Compute the marker's position in the base frame from the marker's pixel location and the camera's transform. Compare with the tape measure.

---

## Project: a calibrated camera with a measured distance check

This project builds a calibrated camera pipeline that measures distance to a known target and reports the error against a tape measure. It produces the calibration file, the frame transform and a distance table, which are the three things a robot's perception system needs before it can act on a camera.

**Goal:** calibrate one camera at its working resolution, measure the frame rate and the exposure at two lighting conditions, and produce a distance estimate for a target of known size, with the error measured at four distances.

**Repo layout:**

```
100-days-robotics-engineering-day-32-cameras/
├── README.md
├── calibration/
│   ├── images/                  # 15 checkerboard photos
│   └── calibrate_camera.py
├── analysis/
│   ├── distance_check.py
│   └── requirements.txt
├── data/
│   ├── camera_calibration.npz
│   ├── exposure_runs.csv
│   └── distance_check.csv
├── calculations/
│   ├── pixel_focal_length.md
│   ├── frame_transform.md
│   └── depth_resolution.md
├── plots/
│   └── distance_error.png
└── images/
    └── camera_mount.jpg
```

**Starter code:** `analysis/distance_check.py`

```python
"""Estimate distance to a target of known height from its pixel height.

Pinhole model: Z = fy * H / h, where H is the target height in metres,
h is its height in pixels and fy is the focal length in pixels from calibration.
"""
import csv
from pathlib import Path

import numpy as np

CAL = np.load(Path(__file__).resolve().parent.parent / "data" / "camera_calibration.npz")
K = CAL["K"]
FY = K[1, 1]
TARGET_HEIGHT_M = 0.297        # A4 sheet, long edge, measured

def distance_from_pixels(h_px):
    return FY * TARGET_HEIGHT_M / h_px

if __name__ == "__main__":
    # Each row: true_distance_m, measured_height_px
    rows = [
        (0.5, 0),   # replace with your measured pixel heights
        (1.0, 0),
        (1.5, 0),
        (2.0, 0),
    ]
    print("true_m, height_px, estimated_m, error_percent")
    for true_m, h_px in rows:
        if h_px <= 0:
            continue
        est = distance_from_pixels(h_px)
        err = 100.0 * (est - true_m) / true_m
        print(f"{true_m:.2f}, {h_px}, {est:.3f}, {err:+.1f}")
```

The starter code loads the calibration file, takes the vertical focal length from the intrinsic matrix, and converts a measured pixel height into a distance. It reports the error against the tape-measure value. The pixel heights in the list are placeholders, and the script skips any row with a zero height, so the output shows only the rows you have measured. Replace them with the heights from your own images.

The calibration step is in `calibration/calibrate_camera.py`, which is the code from the lesson, saved as a file. Write it as part of the project, and save the output to `data/camera_calibration.npz`. The calibration must use the same resolution as the distance check.

**Step-by-step tasks:**

1. Create the repo and write the README with the goal, the camera model, the resolution, the lens and the mount, and the wiring diagram.
2. Capture 15 checkerboard images at the working resolution and run the calibration. Record the RMS reprojection error and the K matrix in `calculations/pixel_focal_length.md`, and compare fx with the value from the field of view calculation in the lesson.
3. Measure the frame rate and the mean brightness at two exposure times under two lighting conditions, and record them in `data/exposure_runs.csv`.
4. Place the A4 sheet at 0.5, 1.0, 1.5 and 2.0 m, measure its pixel height at each distance, and record the rows in `data/distance_check.csv`.
5. Complete `analysis/distance_check.py`, run it, and produce `plots/distance_error.png` with the error against distance.
6. Write `calculations/frame_transform.md` with the camera's mounting transform, and `calculations/depth_resolution.md` with the depth error per pixel of disparity at 1 m and 2 m for a stereo pair of your choice, or a short note explaining why you did not use stereo.

**README checklist:**

- [ ] Goal, camera model, resolution, lens and mount
- [ ] Calibration result with the RMS reprojection error and the K matrix
- [ ] Exposure and frame-rate table for two lighting conditions
- [ ] Distance check table with the error at four distances
- [ ] Frame transform from the camera to the robot base
- [ ] Depth resolution note
- [ ] One limit of the camera, stated with a number from your data

**Done when:**

- The RMS reprojection error is below 0.5 pixels, or the README explains the cause of a higher value and the steps taken.
- The distance error at each of the four distances is within 5 percent of the tape measure, or the README explains the source of the larger error.
- The calibration file is saved at the working resolution, and the README states that resolution.
- The frame transform is tested with a marker at a known position and the error is reported in millimetres.

**Evidence to commit to GitHub:** the calibration script, the calibration file, the exposure and distance data, the analysis script, the plot, the calculations, the mount photo and the README. A reviewer should be able to rerun the analysis and see the same errors.

The camera itself is safe. The hazard is the single-board computer and the mount, so keep the mount secure, keep the cable clear of moving parts and do not power the board from a source with no current limit.

---

## Check your understanding

1. A camera has a horizontal field of view of 70 degrees and 1280 pixels across. Calculate the pixel focal length. Then calculate the pixel height of a 15 cm object at 1.5 m, if its height is measured in the same direction as the focal length.
2. A stereo pair has a baseline of 8 cm and a focal length of 700 pixels. A point has a disparity of 35 pixels. Calculate the depth, and the depth error for a one-pixel error in disparity.
3. A camera is set to 30 frames per second with an exposure of 25 ms. Decide whether the setting is possible, and explain your reasoning. Then calculate the motion blur, in pixels, for a robot moving at 0.5 m/s past an object 3 m away with fx = 600 and the 25 ms exposure.
4. A calibration gives an RMS reprojection error of 1.8 pixels. List two likely causes and one test that would confirm the cause.
5. A distance check reads 4 percent short at 0.5 m, 4 percent short at 1 m and 4 percent short at 2 m. Name the most likely cause and the correction that fixes it.
6. A robot's camera frame has its optical Z axis pointing forward. Explain why a robot that uses the optical axes directly for a forward move drives the wrong way, and state the axis that corresponds to the robot's forward direction in the base frame.

---

### Answers

1. The pixel focal length is (1280 / 2) / tan(35 degrees) = 640 / 0.7002 = 914 pixels. The object's pixel height is 914 times 0.15 / 1.5 = 91.4 pixels.
2. The depth is Z = 700 times 0.08 / 35 = 56 / 35 = 1.6 m. The depth error for a one-pixel disparity error is dZ = Z squared divided by (f times B), which is 1.6 squared divided by (700 times 0.08), or 2.56 / 56 = 0.046 m, about 4.6 cm.
3. The setting is possible, because the frame period at 30 frames per second is 33.3 ms and a 25 ms exposure fits inside it, provided the sensor's read-out time also fits in the rest of the period. The motion blur is v times t times fx / Z = 0.5 times 0.025 times 600 / 3 = 2.5 pixels.
4. Two likely causes are a blurred or a poorly printed checkerboard, and a set of poses that all face the camera or do not cover the image. A third is a resolution mismatch between the calibration and the use. A test is to recapture the images with a sharper pattern, a wider range of poses and the working resolution, and to compare the RMS value.
5. A constant percentage error at every distance points to the focal length or to the target height used in the calculation. The most likely cause is a wrong focal length from calibration or a target height that is 4 percent larger than the value used. The correction is to re-check the target height, and if it is correct, to re-run the calibration at the working resolution.
6. The optical Z axis points forward in the camera frame, but the robot's base frame has X pointing forward. A forward command in the optical frame sends the robot along the base frame's Z axis, which is up, or the wrong axis, so the robot moves in the wrong direction or does not move as planned. The base forward direction corresponds to the optical Z axis after the fixed rotation, and the transform must be applied before the command is sent.

---

## Sources

*Read*

- [Wikipedia: Pinhole camera model](https://en.wikipedia.org/wiki/Pinhole_camera_model) for the geometry of the camera model and the intrinsic matrix.
- [Wikipedia: Camera resectioning](https://en.wikipedia.org/wiki/Camera_resectioning) for the matrix form of the projection and the calibration.
- [Wikipedia: Stereo camera](https://en.wikipedia.org/wiki/Stereo_camera) for the stereo principle and the baseline.
- [Wikipedia: Epipolar geometry](https://en.wikipedia.org/wiki/Epipolar_geometry) for the constraint that makes stereo matching work.
- [Wikipedia: Rolling shutter](https://en.wikipedia.org/wiki/Rolling_shutter) for the skew effect and its causes.
- The OpenCV documentation at docs.opencv.org, the camera calibration tutorial and the reference for cv2.calibrateCamera and cv2.undistort. Search "OpenCV camera calibration tutorial".
- The Raspberry Pi Camera Module 3 documentation at raspberrypi.com/documentation. Search "Raspberry Pi Camera Module 3 documentation".

*Tutorials*

- Adafruit Learning System, the guides on cameras for Raspberry Pi and on computer vision with OpenCV. Search "Adafruit Raspberry Pi camera guide".
- SparkFun Learn, the tutorials on machine vision and on camera basics. Search "SparkFun Learn machine vision".
- GeeksforGeeks, the article on the pinhole camera model and on stereo vision. Search "GeeksforGeeks stereo vision depth".

*Watch*

- Cyrill Stachniss, the lecture series on camera models and computer vision for robotics on YouTube. Search "Stachniss camera model".
- MIT OpenCourseWare, 6.801 Machine Vision, the lectures on camera geometry and calibration. Search the course title on ocw.mit.edu.

*Practice*

- OpenCV's Python tutorials, which include a camera calibration sample you can run on the checkerboard images before using your own camera. Search "OpenCV Python tutorials camera calibration".

---

## Tomorrow

Tomorrow you bring the whole module together. Day 33 is the sensor integration project: you collect data from the encoders, the IMU, a distance sensor and the servo command at the same time, timestamp each stream, log it, and check the timing between a commanded motion and the sensors' response. The camera from today can be added as a later stream, and the calibration habits carry over to every sensor in the module.

**Next:** [Day 33: Sensor Integration Project](Robotics-Day-33-Sensor-Integration-Project.md)
