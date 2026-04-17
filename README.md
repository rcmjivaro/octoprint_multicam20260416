# octoprint_multicam20260416
configure the multicam plugin to work properly
#
Before you begin, read all of this to the end before doing anything. If you are unsure about some of it, fear not. When you're doing it and you get to that point it might be easier than it seemed during the first run through.
I had nothing but problems with multicam. Camera 2 not showing up, showing the image from the primary camera, working now but not later. That quallifies as non-functional in my book. In looking through the code I decided that they had done half the work to make multiple cameras function if you have their knowledge and are sitting in a lab and don't reboot too many times. Not good enough.

I added a variable to the *.txt files that is the entire or at least partial name given to the target camera in /dev/v4l/by-id and then 3 lines of code that extract the target /dev/video[0-9] name associated with that target camera name (the content of /dev/v4l/by-id is links that redirect to a /dev/video[0-9] device file). Now, each time the system boots, and starts the cameras, it detects and uses the current device name (which can and does change) for the target camera and it just works. If you unplug/replug/plug-in one of your cameras just run systemctl start(or restart) webcam[0-9]d and voila! multicams! 

I thought about scripting this setup but I'm retired now. I got what I need out of it and with the lack of functional info out there this should be good help and will also familiarize you with the system architecture. It would also require a number of bash tricks I barely remember how to do and some fancy commands with sed. It will be a lot less work to paste my notes for ye and go sip some irish whiskey. I think I'll just do that.
Here you go.

##################

multi-webcam setup in octoprint:

# make the directory for addional camera option files
mkdir /boot/firmware/octopi.conf.d

### octopi.txt ###
# the primary camera option file
add to /boot/firmware/octopi.txt:
camera="usb"
enable_network_monitor=0
destination_host=192.168.1.1
camera_http_webroot="./www-octopi"
camera_usb_options="-r 640x480 -f 10"
camera_http_options="-p 8080"
target_camera="usb-Sonix" # (enough text from 'ls /dev/v4l/by-id/ |grep index0' to identify the target camera)

### webcam2.txt ###
# camera option file number 2
cp /boot/firmware/octopi.txt /boot/firmware/octopi.conf.d/webcam2.txt
edit webcam2.txt to read:
camera="usb"
enable_network_monitor=0
destination_host=192.168.1.1
camera_http_webroot="./www-octopi2"
camera_http_options="-p 8081"
target_camera="usb-Bartcam" # (enough text from 'ls /dev/v4l/by-id/ |grep index0' to identify the target camera)


### webcamd ###
# the bash script that starts the primary camera
# cd to the directory that holds the camera start scripts
cd /root/bin
# edit webcamd as follows:
# at the end of the array declarations (line 46), add this line:
array_target_camera=()

# The next array section pulls in values from the config file(line 95). Add this line:
array_target_camera+=("${target_camera}")

# this uses the camera name to aquire its target device name which can change at boot. It has to be detected dynamically.
# Under the comment that says "# starts up the USB webcam", find the line (line 188) that says "device=`basename "$input"`". Insert these 3 lines after it:
    vlnk=$(ls /dev/v4l/by-id | grep ${target_camera} |grep index0)
    camdev=$(readlink -f /dev/v4l/by-id/${vlnk})
    device=$(basename ${camdev})


### webcam2d ###
# copy the primary camera file to another file to minimize changes required for the first additional camera
cp /root/bin/webcamd /root/bin/webcam2d
# update the CONFIG_FILE variable with the webcam2d path/filename (line 18)
# add error handling after the CONFIG_FILE line:
if [ ! -f "${CONFIG_FILE}" ]; then
        echo -e "Config file ${CONFIG_FILE}) cannot be found."
        exit 1
fi
#

### systemd ###

cp /etc/systemd/system/webcamd.service /etc/systemd/system/webcam2d.service
edit the Exec line in webcam2d.service to read:
ExecStart=/root/bin/webcam2d

Enable the service
systemctl enable webcam2d
#

# This code won't be reached but it's really not desirable for any of the additional cameras.
comment out the if statement under "# Fallback for older images"

#
### haproxy.cfg ###
# This takes the stream from the cameras and makes it accessible for use.
# edit /etc/haproxy/haproxy.cfg as follows:
under heading 'frontend public' copy the line 'use_backend webcam if { path_beg /webcam/ }' to a new line.
edit that new line to change both appearances of webcam to webcam2
toward end of file, copy the 'backend webcam' section.
edit the new section by changing header to read 'backend webcam2', change /webcam/ on the reqrep line to /webcam2/ and the port number on the server line to 8081

### mjpg-streamer ###
# The index file just provides a framework but I went with total isolation of cameras.
cp -r /opt/mjpg-streamer/www-octopi /opt/mjpg-streamer/www-octopi2


### COLD BOOT ###
Shutdown the system using the menu on the web page, then remove power, wait 10 seconds and apply power again.


### web interface ###
# In the web UI, disable the 'Classic Webcam' plugin, then add these plugins
multicam
webcam extras
camera settings


### multicam ###
# For the primary camera
webcam URL /webcam/?action=stream
snapshot URL http://127.0.0.1/?action=snapshot

# For the first secondary camera
webcam2 URL /webcam2/?action=stream
snapshot URL http://127.0.0.1/?action=snapshot

####
### REBOOT ###
Perform a warm or cold boot. After the system is fully back up and the web page appears again, have the browser refresh the page so it isn't using previously cached information. The easy way is to press F5.

The end.

Nowthen. 
The USB bus on a raspberry pi has limited bandwidth. Be sure to not push it or it can affect your prints.
The pi can supply limited power to the USB ports. It would be best to use a powered USB hub for your cameras.
If you have more cameras to configure, follow all of the above direcctions that apply to webcam2 but change the 2 to a 3 and on and on.

### Troubleshooting ###
/var/log/webcamd.log is key.
Unfortunately it doesn't have timestamps so it's not possible to tell when the current enteries were recorded so to make it clear do this:
echo '####################' >> /var/log/webcamd.log
echo '####################' >> /var/log/webcamd.log
echo '####################' >> /var/log/webcamd.log
echo '####################' >> /var/log/webcamd.log
reboot and what comes after those lines is current.
There are 2 things I found most important. Is there an entry for each configured camera, and is the mjpeg-streamer line correct. If camera enteries are missing, there is an error in the webcam[0-9]d script file, and if the mjpg-streamer file has the wrong device filename it is likely that the target-camera variable value is wrong. Keep in mind that camera number 2 should be webcam2 and that number should increment for each additional camera. Also, the port number for webcam2 should be 8081 and that number should also increment for each addional camera.
If the script is right and the variable is right then you need to refresh the page (F5) or reboot the system.

