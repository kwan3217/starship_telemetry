# Starship telemetry

Scripts for downloading, extracting and processing telemetry data from the
Starship–Superheavy integrated flight test on Apr 20 2023. Will likely update with
telemetry from subsequent launches.

I'm sharing these scripts here, but they were written to run on my Linux system and I
have not made any effort to make them portable or consider how you might run any scripts
other than `plot.py` on other systems.

The Python script `plot.py` should work on other systems as long as you install the
dependencies: `numpy`, `scipy`, `pandas`, `matplotlib`, `Pillow`, and `pyatmos`. The
telemetry data extracted using the other two scripts is saved as a `.csv` file in this
repository, so `plot.py` has the data it needs even if you can't run the other scripts
on your system.

* `download_and_crop.sh` is a bash script that calls `yt-dlp` to download SpaceX's
  YouTube video of the launch, then passes the video stream to `ffmpeg`, which crops the
  regions of the image containing telemetry data and saves the resulting images to file
  once per second of video.

* `extract.py` uses `pytesseract` to call `tesseract` and perform optical character
  recognition on these images, and saves the resulting data to `telemetry.csv`. This
  script just keeps running waiting for frames, you can run it at the same time as
  `download_and_crop.sh` to process data as a pipeline. You'll need to stop it manually
  with ctrl-C. This pipelining is because I was considering making some kind of
  live-updating dashboard during the next launch attempt, but this will be tricky and
  time-consuming to get right and so I probably won't end up doing that.

* `plot.py` processes the telemetry data and attempts to extract vertical and downrange
  position, velocity, and acceleration. It also uses a model of atmospheric density and
  speed of sound with altitude to compute the dynamic pressure (q) and Mach number.
  Assuming that the velocity telemetry from SpaceX is in the non-inertial, rotating
  frame of the Earth, this script also uses the approximate planned bearing of the
  vehicle to compute its position above the Earth over its flight, and corrects for the
  rotation speed of Earth at that point to obtain its velocity in the non-rotating frame
  of the Earth. This is then used to calculate orbital parameters: apogee, perigee, and
  semimajor axis. All this data is plotted.

Obviously the vehicle didn't get very far from the launch site in its first test flight,
so this frame change and orbital parameter calculation will be more relevant for future
flights which hopefully cover more distance!

# Updates for Flight 6 (by kwan3217)
I was inspired by this project but wasn't able to get it to work.
I ended up doing it almost entirely with the command line. I do
use the `combine.py` and modified `plot.py` scripts. The following
is an approximation of the process for flight 6.

This whole process takes several hours and hundreds of gigabytes of space.

## Make sure `yt-dlp` is up-to-date.
``` bash
snap install yt-dlp
```
This has to be a relatively recent to get past the current adblockers.

## Download the video
``` bash
mkdir starship_telemetry/youtube
cd starship_telemetry/youtube
yt-dlp https://https://www.youtube.com/watch?v=ZW_vcTfKFk8
cd ..
```
This gets the file `FULL SpaceX Starship Flight 6 Broadcast re-upload [ZW_vcTfKFk8].webm`.
This video is in 1080p and in the form of a `.webm` file which ffmpeg can handle just fine.

## Trim the video
This makes it so that frame 1 is at telemetry T-00:00:04, and continues past splashdown.
``` bash
ffmpeg -i youtube/FULL*.webm -ss 00:40:00 -t 01:06:00 youtube/timecrop.webm
```

## Extract the video frames
There are over 118,000 frames. This causes problems in such programs as geeqie
which just get slower and slower the more frames they have to deal with. As a consequence, we
move the frames into sub-folders with 1000 frames each (except the last).

```
cd starship_telemetry/
mkdir frames
ffmpeg -i youtube/timecrop.webm frames/%06d.png
for i in `seq -w 0 118`
do
  mkdir -v frames/$i
  mv -v frames/$i*.png frames/$i
done
```

## Subset the frames
We use ImageMagick `convert` to do all of the following on each frame:
* Crop out a region of interest
* Decompose the image by RGB and take the red channel, since that has the best color contrast.
* Threshold the image so pixel values >200 DN are white and less are black
* Invert the image so we have white background and black text, which Tesseract reads much better
* Save the image as a png in the right folder

Each region has its own crop geometry. I actually ran each block below in a separate terminal to use all the cores
on my machine. The subframes are sorted into subfolders of 1000 just like the frames. 

```bash
# Booster spd
export x0=354
export y0=908
export x1=445
export y1=946
export fb0=0
export fb1=12
export region='booster_spd'
```

```bash
# Booster alt
export x0=354
export y0=945
export x1=445
export y1=978
export fb0=0
export fb1=12
export region='booster_alt'
```

```bash
# Ship spd
export x0=1525
export y0=908
export x1=1624
export y1=946
export fb0=5
export fb1=118
export region='ship_spd'
```

```bash
# Ship alt
export x0=1525
export y0=945
export x1=1624
export y1=978
export fb0=5
export fb1=118
export region='ship_alt'
```

```bash
# Timestamp
export x0=852
export y0=942
export x1=1065
export y1=993
export fb0=0
export fb1=118
export region='timestamp'
```

```bash
# common code
cd starship_telemetry/frames
mkdir ${region}_frames
for i in `seq -w $fb0 $fb1`
do 
  mkdir ${region}_frames/$i
  for j in $i/*.png
  do 
    echo $j
    convert $j -crop $((x1-x0))x$((y1-y0))+${x0}+${y0} \
               -colorspace RGB \
               -channel R \
               -separate \
               +channel \
               -threshold 78% \
               -negate \
               ${region}_frames/$j
  done
done
```

## Run tesseract on the frames to extract the data
All of these extractions are similar, but the booster ones can be stopped
after about frame 14000 and the ship ones don't start until about frame 5000.
Also we use a different whitelist for `tesseract` for the timestamp and the speeds/alts.

Tesseract runs slower than imagemagic, so I ran each of these blocks
in a separate terminal and started each block shortly after the corresponding
imagemagick cropper was running.
```bash
# Timestamp
cd frames/timestamp_frames
(for j in [0-9]*; 
 do 
   for i in $j/*; 
   do 
     echo `basename $i .png`, `tesseract $i stdout --oem 3 --psm 7 -c "tessedit_char_whitelist=T:+-0123456789" 2> /dev/null`;
   done
 done) | 
sed 's/[^[:print:]]//g' | 
tee ../../timestamp.csv
```

```bash
# Booster speed
cd frames/booster_spd_frames
(for j in 0`seq -w 0 14`; 
 do 
   for i in $j/*; 
   do 
     echo `basename $i .png`, `tesseract $i stdout --oem 3 --psm 7 -c "tessedit_char_whitelist=0123456789" 2> /dev/null`;
   done
 done) | 
sed 's/[^[:print:]]//g' | 
tee ../../booster_spd.csv
```

```bash
# Booster alt
cd frames/booster_alt_frames
(for j in 0`seq -w 0 14`; 
 do 
   for i in $j/*; 
   do 
     echo `basename $i .png`, `tesseract $i stdout --oem 3 --psm 7 -c "tessedit_char_whitelist=0123456789" 2> /dev/null`;
   done
 done) | 
sed 's/[^[:print:]]//g' | 
tee ../../booster_alt.csv
```

```bash
# Ship speed
cd frames/ship_spd_frames
(for j in `seq -w 5 118`; 
 do 
   for i in $j/*; 
   do 
     echo `basename $i .png`, `tesseract $i stdout --oem 3 --psm 7 -c "tessedit_char_whitelist=0123456789" 2> /dev/null`;
   done
 done) | 
sed 's/[^[:print:]]//g' | 
tee ../../ship_spd.csv
```

```bash
cd frames/ship_alt
# Ship altitude
(for j in `seq -w 5 118`; 
 do 
   for i in $j/*; 
   do 
     echo `basename $i .png`, `tesseract $i stdout --oem 3 --psm 7 -c "tessedit_char_whitelist=0123456789" 2> /dev/null`;
   done
 done) | 
sed 's/[^[:print:]]//g' | 
tee ../../ship_alt.csv
```

# Combine the CSVs
This is the new `combine.py` which merges all of the csvs generated above into
a single `telemetry.csv`.
