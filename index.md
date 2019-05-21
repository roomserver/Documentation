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

##### stillpic.py

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

The picture taken is preprocessed and the digits are detected.

[Github link](https://github.com/shwetha1607/Server-temp/blob/Version-1.1/temp_detect2%20(1).py).



### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/roomserver/Documentation/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
