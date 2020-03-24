---
layout: post
title:  "Quickly edit sections out of compressed videos using FFmpeg"
date:   2020-03-24 00:00:00 +0530
author: Ganesh
categories: technology
comments: true
---
Videos using popular compression formats such as MP4 consist of many **keyframes** or "intra-frames" which store the complete images in the data stream. For the rest of the frames, the changes occurring from one frame to the next are the only changes stored in the data stream in order to greatly reduce the amount of information (that's compression, in essence). If we decide to chop out a section of a video with the end points at non keyframe locations, the newly created second section of the video starts with orphaned frames as their keyframe is chopped off! When the video is played and as a player parses the first few orphaned frames, we can see jarring artefacts until a keyframe is reached.

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/video-frames.svg" caption="Figure 1. A compressed video containing keyframes (in red) and other frames (in white)" width=500 align="center" %}

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/chopped-frames.svg" caption="Figure 2. Some frames become orphans when they lose their keyframe to the chopped off section" width=500 align="center" %}

In the example below, you can see the glitches as the orphaned frames are being parsed and the player recovering the video after it sees a keyframe.

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/video-artefacts.png" caption="Artefacts can be seen when orphaned frames are being parsed by a video player. Courtesy: A still from the film 'Spectre'. Copyrights belong to the respective owners including Columbia Pictures Industries, Inc, Metro-Goldwyn-Mayer Studios Inc, Eon Productions Ltd and Danjaq, LLC" width=900 align="center" %}

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/actual-shot.png" caption="The video player recovering from the artefacts after finding a key frame. Courtesy: A still from the film 'Spectre'. Copyrights belong to the respective owners including Columbia Pictures Industries, Inc, Metro-Goldwyn-Mayer Studios Inc, Eon Productions Ltd and Danjaq, LLC" width=900 align="center" %}

Such artefacts can be avoided by using techniques such as **edit lists** or **transcoding**. Instead of leaving frames orphaned, an edit list ensures their corresponding keyframe gets copied even if the keyframe is in the section that gets chopped off. If the video was cut at a non-keyframe location, the copied over keyframe is now guaranteed to be positioned at an incorrect location in the video sequence. To avoid the perception of sudden back and forth jumps in the video due to incorrect placement of the keyframe, the edit list needs to specifically tell the player to use this special keyframe only to parse the subsequent frames correctly. Even though edit lists seem to be a convenient approach to deal with keyframe artefacts, many players simply avoid using the information provided by the edit lists, by default, or do not support them at all.

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/editlist-frames.svg" caption="Figure 3. Copying over the keyframe from the chopped section to save the orphaned frames" width=500 align="center" %}

Transcoding, on the other hand, reconstructs keyframes and the subsequent frames for the entire video. As a side effect of the process, the chopping glitches are removed as no frame is left orphaned. Transcoding involves two steps: the first part of the process is **decoding**. This is whereby the original data is transferred to an uncompressed format. The second part of the process is the **re-encoding** whereby the data will now be transferred to the newly desired format. Performing loseless transcoding on videos which use popular video compression formats such as MP4 can cause compression artefacts to blow up in size to result in insanely large file sizes! Additionally, you'll need to know the transcoding settings that were used to produce the original video beforehand to get back the edited video at the original quality.

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/transcoding-frames.svg" caption="Figure 4. Transcoding results in reconstruction of all the frames. Hence, the glitch vanishes." width=400 align="center" %}

Chopping out sections of videos often involves costly transcoding times and use of costly softwares such as Adobe After Effects or Adobe Premium Pro to get back the videos without any loss of quality. FFmpeg is a free and open source cross platform solution to record, convert and stream audio and video. The program is designed for command-line-based processing and widely used for format transcoding, basic editing (trimming and concatenation) and video post-production effects. While edit lists are not popularly used and transcoding is time consuming, there is one other technique that can chop videos for you quickly without losing quality, although, the flexibility of choosing the end points for chopping is reduced. The steps involved in the technique are discussed below with an additional bonus step desciribing the process of burning subtitles into videos using FFmpeg: 

**Step 1 (Optional)**: Burn the SRT (text based) subtitles file into a video. Optionally, at first, the SRT file can be cleaned out by getting it edited online at https://www.nikse.dk/subtitleedit/online# or using any other SRT editor. Since this is an optional step, just note that the subtitle burning process requires encoding. However, this particular type of encoding is quite quick with FFmpeg's `subtitles` filter. A static build of FFmpeg may be required (atleast at the time of writing this blog) to use the `subtitles` filter. Static builds can be downloaded off the main ffmpeg website.

```bash
$ ffmpeg -i input.mp4 -filter:v subtitles=subtitles.srt output.mp4
```

**Step 2**: Break the video into segments with each clip starting with a keyframe. As we are avoiding encoding, not cutting on keyframes leads to artefacts in the video as we have seen before. Spliting the video on keyframes allows us to later concatenate them in any way we want. By doing this step, you will be creating multiple short clips each starting with a keyframe.

{% include image.html url="/assets/blog-data/2020-03-24-quickly-edit-out-videos-using-ffmpeg/keyframe-method.svg" caption="Figure 5. Splitting the entire video on keyframes into tiny little clips" width=600 align="center" %}

```bash
$ ffmpeg -i INPUT.mp4 -acodec copy -f segment -vcodec copy -reset_timestamps 1 -map 0 OUTPUT%d.mp4
```

**Step 3**: Create a file called `mylist.txt` using the simple Python script provided below. The text file should contain the paths to the required clips in the format `file '/path/to/clip/file.mp4'`. Since the script simply prints the formatted strings, use the shell's redirection operator to write the output to a text file (not shown here as it is straight forward). To remove a part of the original video that you don't want, just exclude them to avoid them in the final video. In the example below, the clips numbered 133, 761, 894, ... so on are excluded.

```python
# Specify the starting and ending + 1 number of clips
# generated after spliting the main video on
# keyframes
for i in range(0, 2000):
    # Exclude the clips to be omitted from the final video
    if i in [133, 761, 894, 895, 896, 897, 1002, 1340, 1341, 1342, 1343, 1344, 1345]:
        continue
    else:
        print "file '/path/to/clip/OUTPUT%s.mp4'" % i
```

**Step 4**: Concatenate multiple videos: the step clearly doesn't require any encoding. The final `output.mp4` should not contain the clips you excluded in the previous step.

```bash
$ cat mylist.txt
file '/path/to/clip/OUTPUT1.mp4'
file '/path/to/clip/OUTPUT2.mp4'
file '/path/to/clip/OUTPUT3.mp4'

$ ffmpeg -f concat -safe 0 -i mylist.txt -c copy output.mp4
```

In conclusion, this technique noticeably reduces your ability to select end points for a cut in a video but allows you to make really quick edits without compromising on the video quality. Although, it is important to note that in order to achieve maximum efficiency in compression, formats like MP4 rely on placing keyframes at scene cuts, location changes or at the start of a motion of a person or objects in the video. Statistically, these locations in the video are exactly where someone would love to have their end points for a cut as these points in the video act as scene transitions.