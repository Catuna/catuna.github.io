---
layout: post
title:  "Hacking in Android headset button support for Windows"
date:   2018-07-02 02:23:46 +0200
categories: blog programming
---
I listen to music on my smartphone almost every day, and whenever I do I readily make use of the media control buttons on my headset. What always bothers me however is when I get home, plug my headset into my desktop computer and continue listening, I suddenly lose the ability to utilize these buttons.

Of course I eventually googled for solutions to this problem, but unfortunetely on Windows there seems to be little support for this great feature: A couple minutes of googling leading only to obscure Stack Overflow posts about soundcards and some people saying their Laptops supporting this out-of-the-box.

Never deterred I instead decided to look at this as an interesting challenge: Can I create some piece of software that enables the media control buttons even when you have no hardware support for them at all? The answer is yes, and here is how you can do it in an afternoon.

# How Android Headset buttons work
The first thing we need to understand is how the Headset buttons on your Headset work. A bit of googling lead me to [this spec](https://source.android.com/devices/accessories/headset/plug-headset-spec) in the Android documentation, which includes the following diagram.

![Circuit diagram for Android Microphone]({{ "/images/headset-circuit2.png" | absolute_url }})

As we can see what happens when one of the Headset buttons are pressed is that we are closing the circuit on one of the different resistors which causes current to flow through it. Of special note is Button A (Play/Pause/Hook) which has a resistance of 0 Ohms, ie. it short circuits the microphone. If we can detect if the Microphone is short-circuited we will thereby have a method of detecting when the play/pause button is being pressed.

# Testing the hypothesis
Before we start programming anything we want to quickly check if our reasoning so far has been good. Ie. we need to see if we can tell when the play/pause button is being pressed by looking at the signal coming from the Microphone. Luckily this is as simple as recording the audio on our computer and looking at the waveform. I did this with Audacity and by pressing the play/pause button while recording we get the following waveform.

![Recording button press]({{ "/images/press_signal.png" | absolute_url }})

*Bingo*

As we can see from the waveform pressing the play/pause button has a very appearent signature on the waveform: A sudden drop down to -1 followed by a sudden jump to 1 after which it gradually decreases until reaching 0. Intuitively based on the spec we say earlier I would have though it would jump to 1 then stay there until the button was released, which as we see is not the case. However the signature we see will still be easy to detect if we can capture a stream of samples from the Microphone.

# Audio capturing with Python
Now that we know how we will detect button presses we can think about how we will reach our goal of controlling the music player on a desktop using the Headset buttons.

The first step in implementing this will be to detect when the button has been pressed. To do this we need to capture a stream of samples from the Microphone and detect the distinct signature we saw earlier. For simplicity we will implement our solution in Python. After a bit more googling I found a package called sounddevice which can abstract away the worst part of actually capturing samples from the microphone.

A bit of tinkering later we get this:

{% highlight python %}
import sounddevice as sd

SAMPLE_RATE = 1000 # Sample rate for our input stream
BLOCK_SIZE = 100 # Number of samples before we trigger a processing callback

class HeadsetButtonController:
    def process_frames(self, indata, frames, time, status):
        mean = sum([y for x in indata[:] for y in x])/len(indata[:])

        print(mean)

    def __init__(self):
        self.stream = sd.InputStream(
            samplerate=SAMPLE_RATE,
            blocksize=BLOCK_SIZE,
            channels=1,
            callback=self.process_frames
        )
        self.stream.start()

if __name__ == '__main__':
    controller = HeadsetButtonController()

    while True:
        pass
{% endhighlight %}

Running this continuously prints the mean of each block of samples we are processing as a batch. As you can see we define a sample rate of 1000 which we set awfully low for audio processing (44100 being a common frequency), however we don't really need the resolution for our task. The block size defines how many samples we should buffer before triggering a callback, again we set this very low: A block size of 100 and sample rate of 1000 effectively means we are triggering 10 callbacks a second, each processing only 100 samples.

# Detecting button presses: The probably too simple way
Now that we can capture button presses we can start working on actually detecting button presses. Recall that we saw that the waveform amplitude jumps to 1 whenever we press the button, this gives us a simple way of detecting button presses: If we detect values close to 1 in our stream over a set period of time then we have a button press.

While there are probably many fancy techniques from the field of signal analysis we could use here, we are going to go with the simplest possible: If the mean of the samples in *N* consecutive blocks is above 0.9 then the button is pressed.

Implementing this into our block processing function we get the following:

{% highlight python %}
import sounddevice as sd

SAMPLE_RATE = 1000 # Sample rate for our input stream
BLOCK_SIZE = 100 # Number of samples before we trigger a processing callback
PRESS_SECONDS = 0.2 # Number of seconds button should be held to register press
PRESS_SAMPLE_THRESHOLD = 0.9 # Signal amplitude to register as a button press

BLOCKS_TO_PRESS = (SAMPLE_RATE/BLOCK_SIZE) * PRESS_SECONDS

...

def process_frames(self, indata, frames, time, status):
    mean = sum([y for x in indata[:] for y in x])/len(indata[:])

    if mean < PRESS_SAMPLE_THRESHOLD:
        self.times_pressed += 1

        if self.times_pressed > BLOCKS_TO_PRESS and not self.is_held:
            # The button was pressed!
            self.is_held = True
    else:
        self.is_held = False
        self.times_pressed = 0

...
{% endhighlight %}

Basically we keep an internal counter of how many blocks of samples we have processed that met the threshold requirement, here we simply set it to 0.9 to deal with the inevitable noisy sample. If a block does not meet the requirment the counter is reset and we start over again. We also keep track of whether we have already registered a button press with the is_held variable. This keeps us from registring the press many times even though we never let go of the button.

# Controlling media playback on Windows
Now all that remains is to replace the "The button was pressed!" comment with the actual code for making Windows toggle media playback. Again we turn to Google to understand how we can make this happen: Turns out we can control media playback by simulating a key press with the correct [virtual-key code](https://msdn.microsoft.com/en-us/library/dd375731%28v=VS.85%29.aspx).

Simulating key presses turns out to be fairly simple using the [pywin32](https://github.com/mhammond/pywin32) package which is just a Python wrapper for the Windows API. Putting all of this together we can create the following function:

{% highlight python %}
import win32api
import win32con

VK_MEDIA_PLAY_PAUSE = 0xB3

def toggle_play():
    win32api.keybd_event(VK_MEDIA_PLAY_PAUSE, 0, 0, 0)
{% endhighlight %}

And there we have it! By triggering the toggle_play function in the spot we had "The button was pressed!" commented earlier gives is a program which allows us to control any media player in Windows using the buttons on a Android headset.

Testing the solution shows that it works surprisingly well, the only difference between the functionality on Android vs Windows is that there is a slight delay before the pause/play kicks in, however this is something I can live with.

![And there we have it]({{ "/images/export.gif" | absolute_url }})

*And there we are*

A Python script of 51 lines that allow us to utilize Android headset media controls in Windows. The final source code of this project can be found on [Github](https://github.com/Catuna/AndroidMediaControlsWindows).

# But wait, there's more!
After using the program happily for a couple of hours I noticed a major issue:

![Too much CPU!1!11]({{ "/images/cpu_usage.png" | absolute_url }})

My program is using almost 30% CPU! Obviously this is unacceptable for long-term use, and we need to fix it. After looking at my code I realized that our main thread is buzy waiting in the main loop even though nothing happens in it. A simple solution to this is just to make the thread sleep forever: Since the callback is triggered automatically we don't really need the loop anyway.

{% highlight python %}
from time import sleep

if __name__ == '__main__':
    controller = HeadsetButtonController()

    while True:
        sleep(10)
{% endhighlight %}

![Good enough]({{ "/images/cpu_usage2.png" | absolute_url }})

I also did not want to run a Python script whenever I start my computer manually. Thankfully Python on Windows ships with the useful pythonw.exe binary which effectively starts a "daemon" process without an attached terminal. By placing a shorcut to this process in the "Microsoft\Windows\Start Menu\Programs\Startup" directory with our script as the first argument the application automatically launches itself and runs silently in the background.
