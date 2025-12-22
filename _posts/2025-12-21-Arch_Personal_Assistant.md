---
layout: post
title: "Arch Linux and a Personal Assistant"
date: 2025-12-21 09:00:00 -0500
categories: [Linux, Arch, A.I.]
tags: [Arch, AI]
author: Th3_4ngl3r
toc: true
comments: false
---

## Building a Voice Assistant in Arch Linux at 3 AM

It’s 3 a.m. on a Tuesday, and my nightly shenanigans have finally wound down. Like many Linux users, I have a deep, unwavering disdain for the mouse. It slows me down, breaks my flow, and generally feels like an unnecessary relic of the Windows world I left behind. One of the biggest reasons so many of us jump ship to Linux is the freedom it gives us—flexibility, customization, and an open‑source community that encourages tinkering at every level.

So there I am, half‑delirious and fully caffeinated, when a thought hits me: Could I build a voice assistant into Arch? I’m pretty sure I can. The real question is—what would I make it do?

With how much I’ve been using AI lately, I’m typing constantly. And honestly, I’m lazy. That’s why I avoid the mouse in the first place. If I can automate something, I will. So naturally, the next step is figuring out how to map a keybinding that records my voice and pipes it straight into an AI model.

And once that’s possible… well, the fun begins.

Imagine saying “Destroy everything” and having your hard drive immediately zeroed out. (Terrifying? Yes. Hilarious? Also yes.) Or saying “Browser” and watching your browser pop open—though to be fair, I already have a keybinding for that.

Today, I’m going to walk through how I set all of this up. The possibilities are probably endless. Sure, something more polished or more efficient might already exist—but it’s 3 a.m., I’m bored, and this one is mine.

If you have ideas for features I should add, drop them in the comments. I’m genuinely curious what wild or clever uses other people might come up with.

## Prerequisites

The first thing I needed was a simple workflow—nothing fancy, just the core loop:

Press a button → Record my voice → Send the audio to Gemini.

After digging around for a bit, I landed on Whisper, OpenAI’s speech‑to‑text model, to handle the transcription side of things. It’s lightweight, accurate, and easy to integrate into a custom setup.  Also, free.

Link to Whisper: <https://github.com/openai/whisper>

* Install required packages

```bash
sudo pacman -S \
  dunst \ 
  jq \
  curl \
  sox \
  python \
  python-pip \
  alsa-utils \
  ffmpeg
```

* Install Whisper

```bash
pip install -U openai-whisper
```

## Obtain Gemini API Key

* Log into the Gemini API Studio <https://aistudio.google.com/>
* Click **Create API Key**
* Secure the key or email it to me! **export GEMINI_API_KEY="Your API key here"**.
  * Optional but recommended:  Store this in either ~/.profile or ~/.bashrc

## Creating a Voice Assistant Script

I ran into a few issues when I first started writing the script. Every time arecord kicked off, it acted like it was recording—but the resulting file was completely silent. My initial command looked something like:

```bash
arecord -q -f cd -t wav $audio
```

After some digging, I discovered the culprit: PulseAudio and PipeWire can lock the microphone, preventing arecord from accessing it. The workaround was to bypass them entirely and talk directly to the hardware.

To figure out which device to target, I ran **arecord -l** which generated the following output:

```bash
**** List of CAPTURE Hardware Devices ****
card 2: Generic_1 [HD-Audio Generic], device 0: ALC285 Analog [ALC285 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

Once you identify the device you want, you can pass it to arecord using the -D (or --device) flag. I chose to reference the hardware directly using its numeric address. The format is where X is the card number and Y is the device number. This gives arecord direct access to the microphone hardware and avoids any interference from PulseAudio or PipeWire.

```bash
hw:X,Y
```

Create the script ~/.local/bin/voice-gemini.sh

```bash
#!/usr/bin/env bash

AUDIO="/tmp/voice_prompt.wav"
TEXT=/tmp/voice_prompt.txt 

# Record audio (push to talk style)
dunstify "Recording..."
arecord -D hw:2,0 -d 10 -f cd "$AUDIO" # Specifies hardware and max recording length 10 seconds

# Transcribe using Whisper
/home/th34ngl3r/venvs/whisper/bin/whisper "$AUDIO" \
 --model small \
 --language en \
 --output_format txt \
 --output_dir /tmp 

PROMPT=$(cat /tmp/voice_prompt.txt)

# Display the voice_prompt with dunst
ANSWER=$(cat $TEXT) && dunstify -u normal "Looking up:" "$ANSWER"
# Send to Gemini
RESPONSE=$(curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent" \
  -H 'Content-Type: application/json' \
  -H 'X-goog-api-key: $GEMINI_API_KEY' \
  -X POST \
  -d '{
    "contents": [
      {
        "parts": [
          {
            "text": "$PROMPT"
          }
        ]
      }
    ]
  }')

# Check if RESPONSE is empty or null
if [ -z "$RESPONSE" ] || [ "$RESPONSE" = "null" ]; then
    dunstify -u normal "Gemini" "❌ Gemini API unavailable or invalid key"
else
    dunstify -u normal "Gemini" "$RESPONSE"
fi


# Display response with Dunst

dunstify -u normal "Gemini" "$RESPONSE"

###
# Future add something here to save a txt copy or question / responses in ~/Documents/ai_responses
###

###
# Future Add something here to cleanup all voice files in /tmp
###
```

Make the script executable.

```bash
chmod +x ~/.local/bin/voice-gemini.sh
```

The script above depicts sending the query to to pro version of Gemini.  You can replace
<https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent>

with

<https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent>

## Create the keybinding

Add the following to your Hyprland key binds:

```bash
bind = SUPER, G, exec, ~/.local/bin/voice-gemini.sh
```

Now, when you press SUPER and G you will start a 10 second recording that will ship off to Gemini and receive a prompt via Dunst with the reply.  This could greatly me expanded to take you anywhere!

I will be posting this script to my Github page and update it as this grows.

## Conclusion

Building this little 3 a.m. voice assistant started as a half‑awake experiment, but it quickly turned into one of those classic Arch Linux rabbit holes—equal parts frustrating, enlightening, and ridiculously fun. What I ended up with isn’t just a neat trick tied to a keybinding; it’s a foundation for something far more flexible. Whether it evolves into a full‑blown personal assistant, a system automation tool, or just another quirky project I tinker with at odd hours, the beauty is that it’s entirely mine to shape. If you decide to try this yourself or have ideas that could push it even further, I’d love to hear them. After all, the best part of Linux is building things that make your workflow feel a little more like magic.
