
# How to apply GIMP effects (filters) to video


Table of contents
* [Versions used](#Versions-used)
* [Summary](#Summary)
* [Description](#Description)
    * [1. Save each frame of the video as an image](#1-Save-each-frame-of-the-video-as-an-image)
    * [2. Apply the effects to each image](#2-Apply-the-effects-to-each-image)
    * [3. Convert the images back to a video](#3-Convert-the-images-back-to-a-video)




#### Versions used:
OS: Ubuntu 21.04\
kernel: 5.11.0-22-generic\
GIMP: 2.10.22\
bash: 5.1.4\
GNOME Terminal: 3.38.1\
ffmpeg: 4.3.2-0\
ffprobe: 4.3.2-0


<br/><br/>
## Summary:
GNU Image Manipulation Program (GIMP) accepts only images, not videos. This tutorial describes how to do the following:

1. Save each frame of the video as an image
2. Apply a GIMP effect to each image
3. Convert the images back to a video

### This walkthrough will be using the bash CLI on Linux



<br/><br/>
## Description:

### 1. Save each frame of the video as an image
The first thing we need to do is determine how much zero padding in the filenames we need so they are sorted correctly. You can estimate how much zero padding you need by multiplying the frames per second by the video duration.
    Example: 30FPS x 15 seconds = 450 frames. Three digit number, so we need zero padding of at least 2.\
`zero_pads=2`

Or you could just use an arbitrarily high value, like 8, which is enough for about a billion frames.\
`zero_pads=8`

**Optional:**\
For fun, here is a programmatic method for determining how much zero padding you need for a file. It uses ffprobe to get the number of frames. Take the log of that number and divide by the log of 10. Basically, count the number of digits.
```
file_path=/home/user/Videos/f20653824_ftyp.mov
zero_pads=$(echo "l($(ffprobe -i $file_path -v error -select_streams v:0 -show_entries stream=nb_frames -of default=nokey=1:noprint_wrappers=1))/l(10)" | bc -l | cut -d \. -f1)
```

<br/><br/>
Create a directory to put the images in and cd to it.
```
mkdir all_frames
cd all_frames
```
<br/><br/>
Save each frame of the video as an image:\
`ffmpeg -hide_banner -i /home/user/Videos/f20653824_ftyp.mov temp_%0"$zero_pads"d.png`


<br/><br/><br/>
### 2. Apply the effects to each image
Now we have a directory full of images which we want to apply a GIMP effect to. Let's create another directory to save the output of GIMP. I'll name my directory "new".\
`mkdir new`


<br/><br/><br/>
**===  IMPORTANT!  ===**\
Become familar with the Script-Fu Console.\
Open the GIMP (GUI) app. In the Menubar go to Filters > Script-Fu > Console > Browse.\
This is where you find the names of all the GIMP commands, and their options, and their explanations.
<br/><br/><br/>

### Here is the basic syntax for GIMP CLI:
```
gimp -i filename.png \
-b '(some-gimp-command 0 1 2 3 4 5)' \
-b '(file-png-save-defaults 1 1 2 "output.png" "raw")' \
-b '(gimp-quit 0)'
```

Open a file with GIMP using "-i".\
The trailing backslash is to indicate that this code is mulitple lines.\
The "-b" argument is used to run multiple commands in a batch...\
followed by the command you want to use. The command is usually followed by some numerical options.\
Then save the file.\
Finally, close GIMP.

<br/><br/><br/>
Now let's do a real example.

We are going to loop through all the images in this directory.\
`for i in $(find . -maxdepth 1 -type f | sort -V); do`

Open a file with GIMP\
`gimp -i $filename \`

Optional: You can specify the number of threads to utilize. I'm not sure if this works.\
`gimp --gegl-threads=4 -i "$i" \`


The first batch command we want to run is to make a selection. That way we apply the effect to only a specific portion of the image.\
`-b '(gimp-image-select-rectangle 1 0 20 15 800 700)' \`

Here we are using rectangular select. There are other selection methods such as ellipse, rounded rectangle, etc.
To understand the number options, look up the command in the Script-Fu Console, as I stated above. I'll explain only this command.
The last 4 numbers are most important in this example. They are: x coordinate of upper-left corner of rectangle, y coordinate of upper-left corner of rectangle, The width of the rectangle, The height of the rectangle. So this example we are starting the rectangular selection at position 20, 15 and it is 800 pixels wide by 700 pixels high.


With the next batch command we are going to apply the effect (filter) called "wind" to the selection. Again, you can find the name of other effects in the Script-Fu Console. They start with "plug-in-".\
`-b '(plug-in-wind 1 1 2 10 2 60 0 0)' \`\
Notice that the third numerical option is "2". This refers to the selection we made in the previous step.\
GIMP CLI uses the term "Drawable" instead of "Selection".

You can apply more batch commands if you wish.

<br/><br/>
Now that we are done, let's save the result. We are using the directory named "new" and "$i" is the variable we assigned during the FOR loop. I don't know why the raw filename and double quotes are needed, but they are.\
`-b "(file-png-save-defaults 1 1 2 \"new/$i\" \"raw\")" \`

Close GIMP\
`-b '(gimp-quit 0)'`


<br/><br/>
### All the bash GIMP code together (with extras):
```
for i in $(find . -maxdepth 1 -type f | sort -V)
    do echo -e \\n Begin: "$i"

    gimp --gegl-threads=4 -i "$i" \
    -b '(gimp-image-select-rectangle 1 0 20 15 800 700)' \
    -b '(plug-in-wind 1 1 2 10 2 60 0 0)' \
    -b "(file-png-save-defaults 1 1 2 \"new/$i\" \"raw\")" \
    -b '(gimp-quit 0)'

    echo Done: "$i"
done
```

<br/><br/>
I get the following error:\
gimp: GEGL-WARNING: (../gegl/buffer/gegl-tile-handler-cache.c:1076):gegl_tile_cache_destroy: runtime check failed: (g_queue_is_empty (&cache_queue))
EEEEeEeek! 5 GeglBuffers leaked

but it still works, so I ignore it.
<br/><br/>
You can probably make this much faster if you can find a way to keep one GIMP instance open the entire time, rather than open and close for each file. Let me know if you figure it out.


<br/><br/>
### 3. Convert the images back to a video
Now we have a directory full of images with the effects applied. Use ffmpeg to convert them back into a video.
```
cd new
ffmpeg -hide_banner -framerate 29.97 -pattern_type glob -i '*.png' -c:v libx264 -preset slow -crf 19 output.mkv
```
<br/><br/>
The audio will have been removed. Here's how you can extract the audio stream from the orignal video:\
`ffmpeg -hide_banner -i /home/user/Videos/f20653824_ftyp.mov -vn -acodec copy output.aac`

<br/><br/>
And here's how to reapply the audio to the video after the new effects.\
`ffmpeg -hide_banner -i output.mkv -i output.aac -map 0:v -map 1:a -c:v copy combined_output.mkv`













