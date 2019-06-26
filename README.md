# GPU accelerated video processing on OpenCV with Python

This repository describes a solution for processing video files with GPU code using OpenCV in Python. I present the class that handles the video reading and present example on how to use it in examples that run on GPU-cores where available. I give my reasoning and why it mattered to me.

This solution builds upon the [fine article](https://www.pyimagesearch.com/2015/02/02/just-open-sourced-personal-imutils-package-series-opencv-convenience-functions/) by **Adrian Rosenbock** at PyImageSearch. He wrote a good solution that utilizes threading to allow for faster processing of videos. I built upon that and decided to write this text to support my solution as well as share it. This code is licensed under the same MIT license his code is.

With this library I was able to speed up my video processing time for my 50 minute videos from 4:15 hours to just under an hour! Read on how I got there.

1) [Usage Example](#usage-example)
1) [Tutorial](#tutorial)
1) [Reasoning](#reasoning)
1) [Understanding OpenCV](#understanding-opencv)
1) [Threading](#threading)
1) [Performance Testing](#performance-testing)

## Usage example

**Installation**: Copy the class code to your project and import it in your Python code.

```python
import cv2
from UMatFileVideoStream import UMatFileVideoStream

video = UMatFileVideoStream(files[0], selectionRate).start()
rgb = cv2.UMat(self.height, self.width, cv2.CV_8UC3)
while not video.stopped:
    cv2.cvtColor(video.read(), cv2.COLOR_BGR2RGB, hsv, 0)
    # more of processing before fetching the images
    cv2.cvtColor(hsv, cv2.COLOR_HSV2RGB, hsv, 0)
    img = hsv.get()   # image is now a numpy array
```

The example above runs all the OpenCV methods in a dedicated GPU if one is available. Any methods that are called similarly with the matrices as source and target specified as ```UMat``` types then OpenCV will be able to utilize GPU for them. This is a bare example only showing simple operations on the images.

User should call the ```read()``` method only once for the read and then use the reference further from that call. The reference ```UMat``` will be overwritten immediately after so it should in fact be part of a call to read the image data to another ```UMat``` image preferably. The bytes are in GPU-memory already so any copy operations will be fast.

When you need to fetch the image from the GPU-memory back to CPU for i.e. serialization you may use the ```get()``` method on the ```UMat``` instance. That returns the numpy type array you can use for example in writing the image out. OpenCV also provides mechanisms to write out images to files directly from ```UMat``` so use those if that is what you need.

## Tutorial

### *Reasoning*

I had a use case to process the video faster than that and I had to run image operations on all the frames and I wanted to do that in the GPU for speed purposes. I had previously used ffmpeg with NVENC extensions very successfully to run encoding on GPU. I was able to achieve speeds in excess of 8.0x speeds to the input video in encoding, so you should understand I was frustrated at the appalling 0.2x speeds I was getting with my first code on the algorithm I needed to execute.

I would need speed gains from somewhere as this situation was forcing my computer to be very busy for the 5 hours it took to process the 1 hour video pieces. Something had to be done.

### *Understanding OpenCV*

A broad generalization of OpenCV is to say that it is a math library operating on matrices. OpenCV works so that the functions defined in the different namespaces are in the general parts able to operate on any ```InputArray``` types. The actual implementation chooses internally which path to take to for the interface implementation, so that the code may be CPU-bound or GPU-bound where applicable. As a rule of thumb one can say that matrices of ```UMat``` types are GPU-bound whereas ```Mat``` types are CPU-bound. This is a feature I wanted to utilize.

The example I give above is what I needed to achieve and I had to find out ways to call OpenCV properly to have it call the necessary functions on the proper matrices. In this case the ```UMat```s I needed to use. The trick for me with OpenCV was in realising that the functions in the Python interface were possible to be called on any ```InputArray``` types and achieve the GPU processing I was after.

### *Threading*

The **imutils** library already defined a good case for the threading model that I wanted to re-use. So my alteration for the class was to specialize it for ```UMat```s and have the buffering essentially write directly to GPU-memory as that is where the ```UMat```s are used. I think it is actually shared memory it uses there and this abstraction is hidden from the user for the most part.

Modifications to the original include initializing the ```UMat```s as called for in the queuesize and then pushing the frame numbers to queue instead of the actual images. This way developer is able to read the ```UMat``` type frames by calling ```read()``` method on the instance and not have to worry about the queue that handles the internals of the class.

I also added a destructor for the class to clean up the code afterwards. Currently very undocumented feature is that the ```read()``` method sleeps until there is actually frames on the queue before moving on. This is a deadlock situation that requires attention in future.

### *Performance Testing*

The performance numbers were compared against the input video runtimes that were usually closer to 1h for me. This is a practice FFMPEG demonstrates when encoding a video, so it tells me that what is the current speed of the processing compared to the normal run speed of the video. Any significant multiplier that took much longer than that was a hindrance to me.

So the code started out from the CPU-only code that was able to do the algorithm I needed on the frames and the speed was appalling compared to encoding only in GPU. The speed was at 0.2x of the normal runtime of the input video and my processing was taking 4:20 h in total for the input video of 0:50 h. Ouch.

I figured it would be good to translate the code to using GPU-code and I was successful at pushing the images to GPU to be handled and I was still short of proper performance. I was able to reach 0.3x speed with this step and it saved me about 1 h in the processing already and it was still near 3 hours.

I thought back to the article I'd read ages ago on PyImageSearch. Threading was a good thing to improve the code. As I ran my tests for the threaded video decoder I was able to raise the performance to another level very nicely. Still using my existing code underneath I managed 0.52x of the normal runtime speed with this and my processing time was at 1:40 h. I knew I could still improve.

The final solution came from modifying the library and seeing that I can recycle the matrices in the shared memory where necessary. Some might argue against this but I needed more speed in the now. After the necessary modifications I tested and saw that I was able to achieve near realtime at 0.87x speed of the original runtime of the video. My processing time for the 0:50 h video was now 0:57 h. Mission accomplished.

-- Mikael Lavi