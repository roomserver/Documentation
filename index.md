## Getting Started

Features:

* Taking a picture using the PiCamera
* Detecting the temperature from the picture taken of the display
* Sending out mail/SMS alerts
* Provision to suspend/resume receiving alerts
* Getting information about users registered to receive alerts


### Taking a picture using the PiCamera

A picture of the temperature display is taken and the saved in the directory /home/pi/Pictures/server under the name image.jpg.
The picture is saved only when the lights aren't switched on in the server room.
[Github link](https://github.com/shwetha1607/Server-temp/blob/Version-1.1/stillpic.py).

###### stillpic.py

```python
from picamera import PiCamera
from picamera.array import PiRGBArray
import time
import cv2

camera= PiCamera()
camera.resolution = (640, 480)
rawCapture = PiRGBArray(camera, size=(640,480))

time.sleep(5)
camera.capture(rawCapture,format='bgr')
image = rawCapture.array

gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
blur = cv2.blur(gray, (5, 5))

if cv2.mean(blur)[0] < 3.0:
	cv2.imwrite('/home/pi/Pictures/server/image.jpg', image)
else:
	#print('lights on')
	pass
```

### Detection of temperature using Computer Vision

The picture taken is preprocessed and the digits are detected. Here is the breakdown of the process:

###### temp_detect.py

 * Import the packages
 ```python
import cv2
import numpy as np
import imutils
from imutils import contours 
```
* Initialize a dictionary to map the seven segment digit to its value.
```python
DIGITS_LOOKUP = {
        (1, 1, 1, 0, 1, 1, 1): 0,
        (0, 0, 1, 0, 0, 1, 0): 1,
        (1, 0, 1, 1, 1, 0, 1): 2,
        (1, 0, 1, 1, 0, 1, 1): 3,
        (0, 1, 1, 1, 0, 1, 0): 4,
        (1, 1, 0, 1, 0, 1, 1): 5,
        (1, 1, 0, 1, 1, 1, 1): 6,
        (1, 0, 1, 0, 0, 1, 0): 7,
        (1, 1, 1, 1, 1, 1, 1): 8,
        (1, 1, 1, 1, 0, 1, 1): 9,
    }
```
* Create a class called Temp_detect with the image path initialized.
```python
class TempDetect:

    def __init__(self, img_path):

        # Load the image
        self.image = cv2.imread(img_path)
```
* Preprocess the image
```python
    def pre_processing(self, iteration_val):

        # Crop the image
        (h, w) = self.image.shape[:2]
        center = (w / 2, h / 2)
        print(center)
        self.image = self.image[40:400, 150:600]
        self.image = imutils.resize(self.image, width = 950)

        # Convert to grayscale
        gray = cv2.cvtColor(self.image, cv2.COLOR_BGR2GRAY)

        # Convert pixels >110 to white
        thresh = cv2.threshold(gray, 100, 255, cv2.THRESH_BINARY)[1]

        thresh = cv2.dilate(thresh, None, iterations=iteration_val)
        thresh = cv2.erode(thresh, None, iterations=1)

        blurred = cv2.GaussianBlur(thresh, (11, 11), 0)

        thresh = cv2.threshold(blurred, 100, 255, cv2.THRESH_BINARY)[1]

        #cv2.imshow('thresh', thresh)
        #cv2.waitKey(0)

        return thresh
```

* Finding contours in the pre processed image. A contour is simply a curve joining all the continuous points (along the boundary), having same color or intensity.
```python
    def find_contours(self, thresh):

        cnts = cv2.findContours(thresh.copy(), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        cnts = imutils.grab_contours(cnts)
	digitCnts = []
	
        for c in cnts:
            # compute the bounding box of the contour
            (x, y, w, h) = cv2.boundingRect(c)
            # if the contour is sufficiently large, it must be a digit
            # first condition for digit 1
            if (4 <= w <= 14) and (20 <= h <= 55):
                digitCnts.append(c)
            if (15 <= w <= 45) and (20 <= h <= 55):
                digitCnts.append(c)

        digitCnts = contours.sort_contours(digitCnts, method="left-to-right")[0]
        print('len: ', len(digitCnts))

        return digitCnts
```

* Each digit is saved as a contour in digitCnts. Each segment of the digit contour is iterated over to check if that segment is active (>50% of the area has white pixels). This builds a tuple with 1 for active segments and 0 for inactive segments. This tuple acts as the key and matched for its value from the DIGITS_LOOKUP dictionary.
```python
    def detect_digit(self, thresh, digitCnts):

        digits = []
        for c in digitCnts:
            (x, y, w, h) = cv2.boundingRect(c)            
            if (4 <= w <= 14) and (20 <= h <= 55):
                on = (0, 0, 1, 0, 0, 1, 0)
            else:
                roi = thresh[y:y+h, x:x+w]
                (roiH, roiW) = roi.shape               
                (dW, dH) = (int(roiW * 0.3), int(roiH * 0.15))
                dHC = int(roiH * 0.05)
                dBR = int(roiW * 0.5)

                segments = [
                    ((dH, 0), (w-dH, dH)),  # top
                    ((0, dH), (dW, h // 2 - dHC)),  # top-left
                    ((w - dW, dH), (w, h // 2 - dHC)),  # top-right
                    ((dH, (h // 2)-dHC ), (w-dH, (h // 2)+dHC)),  # center
                    ((0, h // 2 + dHC), (dW, h - dH)),  # bottom-left
                    ((w - dBR, h // 2 + dHC), (w, h - dH)),  # bottom-right
                    ((dH, h - dH), (w-dH, h))  # bottom
                ]

                on = [0] * len(segments)

                for (i, ((xs, ys), (xf, yf))) in enumerate(segments):
                    segRoi = roi[ys:yf, xs:xf]
                    #cv2.imshow('segment', segRoi)
                    #cv2.waitKey(0)
                    no_of_pixels = cv2.countNonZero(segRoi)
                    area = (xf - xs) * (yf - ys)

                    if no_of_pixels / float(area) >= 0.5:
                        on[i] = 1

                    print(on)

            digit = DIGITS_LOOKUP.get(tuple(on), -1)
            digits.append(digit)

        return digits
```

* A method that makes the calls of each of the above mentioned functions.
```python
    def final_call(self, iterval):
        thresh = self.pre_processing(iteration_val=iterval)
        digitCnts = self.find_contours(thresh)
        digits = self.detect_digit(thresh, digitCnts)
        print(digits)
        cv2.destroyAllWindows()
        return digits
```

The entire code on github can be found [here](https://github.com/shwetha1607/Server-temp/blob/Version-1.1/temp_detect2%20(1).py).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/roomserver/Documentation/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
