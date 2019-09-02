---
title: "Release of range-tagger v0.0.1"
date: 2019-08-22T11:41:13-04:00
draft: true
---

*tl;dr: range-tagger is an application to help manually label segments of long videos to create training datasets for use in machine learning.  You can download it [here](https://github.com/jswilson/range-tagger/releases/tag/v0.0.1) (OSX only for now) and watch a demo video [here](https://youtu.be/sZvp8YXCoto)*

# Motivation
I was recently working on a machine learning project that required classifying every frame in a set of 4-hour videos.  To simplify describing the project, you can think of it like trying to classify television broadcast footage into “commercial” and “show” labels; we want to accurately label every frame in a broadcast as being an advertisement or part of the actual television show.

We had a collection of videos, but unfortunately none of it was labeled, so we had to create our own training data by manually labeling a portion of the videos.  What's the best way to go about doing this?  We didn't want to manually tag every single frame in the videos.  *"Here is frame 4789, is it a 'commercial' or 'show'?  Now here's frame 4790..."*  This is a television broadcast, so you would never have a commercial frame followed by a television frame, followed by a commercial frame.

Since we don't want to actually look at and classify every frame, it's easier to think of this problem as we are searching for the boundaries between parts of the video:

![](/identify.png "Identify video boundaries")

We want to end up with *ranges of frames*, each with a certain tag.  The end result should be:

| Start Frame | End Frame | Tag |
| ----------- | ----------- | ----------- |
| 0 | 456 | Commercial |
| 457 | 1193 | Show |
| 1194 | 2035 | Commercial |
| ... | ... | ... |

This is distinct from a more common video use case, object detection.  One example of object detection might be *“In this video, label all of the cars.”*  Every frame doesn’t necessarily contain a car, or the opposite might be true depending on your source and every frame contains dozens of cars.  In our use case, we don’t need to examine every single frame like with object detection; we just need to find the “edges” of sections of the video.

There are standalone applications to help with object detection in videos (Microsoft has a good open source offering [here](https://github.com/microsoft/VoTT)), but I was unable to find anything catering to my specific use case.

# Initial Approach

Before creating range-tagger, I initially was solving the problem with a manual process.  I used freely available video players to view and navigate through the video.  I started at the beginning of the video, noted what was displayed, and then used the player’s navigation tools to move forward in the video until I found the frame at which it changed into another section.  I then recorded the frame numbers and the appropriate tag for that range in a spreadsheet.  But I ran into some problems:

### 1. The process was error prone
These are long video files (4+ hours), so the frame numbers quickly became 6 digits.  It was easy to make mistakes when recording the frame numbers.  Since the purpose of this process was to generate an accurate training dataset, this was a fairly significant problem.

### 2. The freely available video players had flaws that made them poor fits for my use case

For instance, [VLC](https://www.videolan.org/vlc/index.html) is phenomenal for watching and consuming media, but it isn’t what I would call “frame-first.”  That is, you can't say "go to this specific frame number" or "go forward 12 frames."  Traditional media players are designed for a human being consuming media, not a computer program reading specific frames.  That isn't the way modern video codecs are designed to work, so video players don’t provide those features.  But I *do* need to know which frame I’m on, and I do need to navigate frame-by-frame through videos.  While VLC does generally provide good coarse and fine navigation abilities, you can't go backwards one frame.  This lack of “frame-first” navigation made VLC a less than ideal solution.

I also tried an open source video editor called [OpenShot](https://www.openshot.org/), which is a great piece of software.  It is much more of a “frame-first” video editor; it displays frame numbers and allows you to navigate frame-by-frame, which is what I need for my use case.  But, its navigation features weren’t as fully featured as VLC’s.  There are *only* frame-forward and frame-backward keys; you can hold them down to make larger frame jumps, but it doesn’t provide coarse enough navigation abilities to quickly move through the 4+ hour videos I’m dealing with.

Between the process being very error prone, and no freely available solutions being “frame-first” and easy to navigate, I decided to create an application customized to my workflow.

# Enter range-tagger

range-tagger is an OSX, python, Qt-based application I created that is designed to solve these problems.  range-tagger provides:

- Simple tools for the user to mark segments of the video without typing frame numbers
- Coarse and fine video navigation that work at the frame-by-frame level
- A “frame-first” video system that knows which frame of video you are currently seeing

![](/range-tagger.jpg "range-tagger screenshot")

I won't delve too much into how it works here, but you can check out a demo/how-to video at [this link](https://youtu.be/sZvp8YXCoto).

## Technical Design Decisions
There were a few big technical decisions that shaped how range-tagger works; I thought it would be useful to document the motivations behind them.

### Web vs. Desktop

This was really the first big decision: should range-tagger be a web app or a desktop app?  A web app would generally be preferred, since it allows anyone to get started without installation, but I ran into issues trying to identify a suitable video player library.  While there are a variety of video players available for use on the web, I was unable to find a free solution that provided frame-by-frame access and navigation.  There are paid solutions available, but one of the subgoals of this project was to offer an open source tool to the community.  Given the difficulty in solving the frame-level video problem on the web, I decided to go with a desktop application.  Sidenote: an open source, frame-by-frame web-based video player appears to be a hole in available offerings, if anyone is looking for a project.

### Choice of Video Library

Once the decision was made to make this a desktop app, I still had to choose the right library to handle video playing and navigation.  This is the API range-tagger uses internally to interface with the video library:

```
    def get_total_frames(self):
        # return the total number of frames in the video

    def get_current_frame_number(self):
        # return the frame number we are currently viewing

    def get_frame(self, frame_number):
        # return the image at the given frame_number
```

This is a very simple API, which is nice because it means we can easily swap out video libraries to test them, but it was actually a little tricky to find a library that could meet it.

I evaluated a few different video libraries:

### ❌ libVLC
The VLC developers also provide a library that performs all of the functions of VLC.  You could actually create an entire clone of the VLC application using just libVLC.  However, libVLC has the same problem as VLC: it is not “frame-first,” so you don’t know what frame you’re on, and you can’t access specific frames.

### ❌ opencv
Typically used for computer vision, opencv does provide mechanisms for video I/O.  I had a promising start with opencv, since the API does allow direct access to frames.  Unfortunately, the API provides [incorrect results](https://github.com/opencv/opencv/issues/9053#issuecomment-503788660); that is, you can use the API to ask for frame 6418, but you might not get frame 6418...you might get a different frame.  opencv is certainly a great library, but they should probably remove the API if it can’t/won’t provide the correct results.

### ✅ libopenshot
I actually hadn’t previously heard of [libopenshot](https://github.com/OpenShot/libopenshot) until developing this application.  Much like VLC is based on libVLC, the OpenShot video editor is based entirely on libopenshot.  Their API provides frame-by-frame access to video files (and it works), so it can fulfill the required API.  I was pretty excited to stumble upon libopenshot, actually; I had been getting dismayed at the prospects of finding a workable library, but it ticks all the boxes, so this was what I chose.

### OSX Only

The only downside of libopenshot is it doesn’t appear to be too widely used, and as a result it doesn’t have prebuilt binaries available for all platforms.  Getting it to compile for my platform (OSX) was a fairly significant task, so I haven’t yet built it for Linux and Windows platforms.  This is the only dependency tying range-tagger to a single platform; I’d be happy to help anyone make a build for Linux or Windows if they have a need for it.

# Results
With those decisions made, I was able to put together range-tagger in fairly short order.  Upon completion of the first version, I was able to label about 11 hours worth of video in under 2 hours with great accuracy and little trouble.  While range-tagger remains in alpha, it is still ready to be used for any similar workloads you might have with tagging video.

# Future work
There are a couple of areas I’d like to explore in the future:

### Faster video library
libopenshot is a great library, and it works especially well on lower resolution video, but there can be a noticeable delay when accessing frames completely randomly in HD video, a delay that isn’t present in commercial video editors.  This is a tough problem to solve (again, modern video codecs aren’t designed for efficient random access), but I’d like to see what can be done to make the ```get_frame``` operation as fast as possible, even when randomly accessing frames in high resolution video.

### New user interface methods for finding video edges
I started range-tagger with the simplest method: allow the user to efficiently navigate the video both coarsely and finely.  But are there other methods that could be even better?  Range-tagger can be a place to experiment with new methods of video navigation for this purpose.

# Conclusion
Feel free to reach out on github if you use range-tagger, or if you have any interest in getting it built for Linux or Windows.  Again, you can download the first release of range tagger [here](https://github.com/jswilson/range-tagger/releases/tag/v0.0.1).  Enjoy!