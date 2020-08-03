
This ia a modified version from https://github.com/s60sc/ESP32-CAM_MJPEG2SD
Added functionality 
* Url encoded requests to handle ssids with spaces
* Added upload move function
* Added footer information
* Improved select sd stored file for playback 
* Stream and playback from access point conections via 
  SmartPhone or Tablet with no internet at all
* Synchronize clock from browser if no internet (Access Point connections)
* Save and load Camera settings from nvs
* Modified user interface

![image1](extras/screenshot.png)

# ESP32-CAM_MJPEG2SD
ESP32 Camera extension to record JPEGs to SD card as MJPEG files and playback to browser. 

Files uploaded by FTP are optionally converted to AVI format to allow recordings to replay at correct frame rate on media players.

## Purpose
The MJPEG format contains the original JPEG images but displays them as a video. MJPEG playback is not inherently rate controlled, but the app attempts to play back at the MJPEG recording rate. MJPEG files can also be played on video apps or converted into rate controlled AVI or MKV files etc.

Saving a set of JPEGs as a single file is faster than as individual files and is easier to manage, particularly for small image sizes. Actual rate depends on quality and size of SD card and complexity and quality of images. A no-name 4GB SDHC labelled as Class 6 was 3 times slower than a genuine Sandisk 4GB SDHC Class 2. The following recording rates were achieved on a freshly formatted Sandisk 4GB SDHC Class 2 using SD_MMC 1 line mode on a AI Thinker OV2640 board, set to maximum JPEG quality and the configuration given in the __To maximise rate__ section below.

Frame Size | OV2640 camera max fps | mjpeg2sd max fps | Detection time ms
------------ | ------------- | ------------- | -------------
QQVGA | 50 | 35 |  20
HQVGA | 50 | 30 |  40
QVGA | 50 | 25 |  70
CIF | 50 | 20 | 110
VGA | 25 | 15 |  80
SVGA | 25 | 10 | 120
XGA | 6.25 | 6 | 180
SXGA | 6.25 | 4 | 300
UXGA | 6.25 | 2 | 450

## Design

The ESP32 Cam module has 4MB of pSRAM which is used to buffer the camera frames and the construction of the MJPEG file to minimise the number of SD file writes, and optimise the writes by aligning them with the SD card cluster size. For playback the MJPEG is read from SD into a cluster sized buffer, and sent to the browser as timed individual frames.

The SD card can be used in either __MMC 1 line__ mode (default) or __MMC 4 line__ mode. The __MMC 1 line__ mode is practically as fast as __MMC 4 line__ and frees up pin 4 (connected to onboard Lamp), and pin 12 which can be used for eg a PIR.  

The MJPEG files are named using a date time format __YYYYMMDD_HHMMSS__, with added frame size, recording rate, duration and frame count, eg __20200130_201015_VGA_15_60_900.mjpeg__, and stored in a per day folder __YYYYMMDD__.  
The ESP32 time is set from an NTP server. Define a different timezone as appropriate in`mjpeg2sd.cpp`.


## Installation and Use

Download files into the Arduino IDE sketch location, removing `-master` from the folder name.  
The included sketch `ESP32-CAM_MJPEG2SD.ino` is derived from the `CameraWebServer.ino` example sketch included in the Arduino ESP32 library. 
Additional code has been added to the original file `app_httpd.cpp` to handle the extra browser options, and an additional file`mjpeg2sd.cpp` contains the SD handling code. The web page content in `camera_index.h` has been updated to include additional functions. 
The face detection code has been removed to reduce the sketch size to allow OTA updates.

To set the recording parameters, additional options are provided on the camera index page, where:
* `Frame Rate` is the required frames per second
* `Min Frames` is the minimum number of frames to be captured or the file is deleted
* `Verbose` if checked outputs additional logging to the serial monitor

An MJPEG recording is generated by holding a given pin high (kept low by internal pulldown when released).  
The pin to use is:
* pin 12 when in 1 line mode
* pin 33 when in 4 line mode

An MJPEG recording can also be generated by the camera itself detecting motion as given in the __Motion detection by Camera__ section below.

If recording occurs whilst also live streaming to browser, the frame rate will be slower. 

To play back a recording, select the file using __Select folder / file__ on the browser to select the day folder then the required MJPEG file.
After selecting the MJPEG file, press __Start Stream__ button to playback the recording. 
The recorded playback rate can be changed during replay by changing the __FPS__ value. 
After playback finished, press __Stop Stream__ button. 
If a recording is started during a playback, playback will stop.

The following functions are provided by [@gemi254](https://github.com/gemi254).

* Entire folders or files within folders can be deleted by selecting the required file or folder from the drop down list then pressing the __Delete__ button and confirming.

* Entire folders or files within folders can be uploaded to a remote server via FTP by selecting the required file or folder from the drop down list then pressing the __FTP Upload__ button.

* The FTP, Wifi, and other parameters ned to be defined in file `myConfig.h`, and can also be modified via the browser under __Other Settings__.

Browser functions only tested on Chrome.


## To maximise rate

To get the maximum frame rate on OV2640, in `ESP32-CAM_MJPEG2SD.ino`:
* `config.xclk_freq_hz = 10000000;` This is faster than the default `20000000` 
* `config.fb_count = 8` to provide sufficient buffering between SD writes for smaller frame sizes 

In `mjpeg2sd.cpp` change `#define CLUSTERSIZE 32768` if the SD card cluster size is not 32kB.

## Motion detection by Camera

An MJPEG recording can also be generated by the camera itself detecting motion using the `motionDetect.cpp` file.  
JPEG images of any size are retrieved from the camera and 1 in N images are sampled on the fly for movement by decoding them to very small grayscale bitmap images which are compared to the previous sample. The small sizes provide smoothing to remove artefacts and reduce processing time.

For movement detection a high sample rate of 1 in 2 is used. When movement has been detected, the rate for checking for movement stop is reduced to 1 in 10 so that the JPEGs can be captured with only a small overhead. The __Detection time ms__ table shows typical time in millis to decode and analyse a frame retrieved from the OV2640 camera.

To enable motion detection by camera, in `mjpeg2sd.cpp` set `#define USE_MOTION true`

Additional options are provided on the camera index page, where:
* `Motion Sensitivity` sets a threshold for movement detection, higher is more sensitive.
* `Show Motion` if enabled and the __Start Stream__ button pressed, shows images of how movement is detected for calibration purposes. Gray pixels show movement, which turn to white if the motion threshold is reached.

![image1](extras/motion.png)

The `motionDetect.cpp` file contains additional documented monitoring parameters that can be modified. 
