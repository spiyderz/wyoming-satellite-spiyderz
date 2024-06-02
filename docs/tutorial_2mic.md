# This is a fork of the original git here: 

https://github.com/rhasspy/wyoming-satellite/tree/master

All cdredit goes to the original developer(s). All I did was fork their repo, clean up the 2_mic tutitorial to make it a bit easier, fixed one line of code in the openwakeword repo to fix the install failing and added the command for openwakeword to start at boot that was missing in the original tutitorial so that the system would survive a reboot or coming back from power loss. I also removed the LED portion of the noriginal tut because I couldnt get it to work. 

## Hardware links for what I have in my setup...

Disclaimer: These are NOT Amazon affiliate links, they are just links for the hardware i used. I purchased on amazon because I didnt want to wait forever for things to come from overseas. I make NO money from these links and give no endorsement other than they worked for me.

Pi Zreo 2 W: https://www.amazon.com/dp/B09LTDQY2Z?psc=1 Its more expensive but I wanted one with headers already installed

2 Mic HAT: https://www.amazon.com/dp/B098R7TTM4?psc=1 This is a knock off respeaker HAT but it works great and comes with some pretty terrific speakers

I can comfirm this works with a Pi4 (and probably a Pi3?) if you have one lying around.

# Tutorial with 2mic HAT

Create a voice satellite using a [Raspberry Pi Zero 2 W](https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/) and a [ReSpeaker 2Mic HAT](https://wiki.keyestudio.com/Ks0314_keyestudio_ReSpeaker_2-Mic_Pi_HAT_V1.0).

This tutorial should work for almost any Raspberry Pi and USB microphone. Audio enhancements and local wake word detection may require a 64-bit operating system, however.

## Install OS

Follow instructions to [install Raspberry Pi OS](https://www.raspberrypi.com/software/). Under "Choose OS", pick "Raspberry Pi OS (other)" and "Raspberry Pi OS (Legacy, **64-bit**) Lite".

When asking if you'd like to apply customization settings, choose "Edit Settings" and:

* Set a username/password
* Configure the wireless LAN
* Under the Services tab, enable SSH and use password authentication

## Install Software

After flashing and booting the satellite, connect to it over SSH using the username/password you configured during flashing.

**On the satellite**, make sure system dependencies are installed:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends  \
  git \
  python3-venv
```

Clone the `wyoming-satellite` repository:

```sh
git clone https://github.com/spiyderz/wyoming-satellite.git
```

Install the ReSpeaker 2Mic or 4Mic HAT Drivers, recompile and install the drivers (this will take really long time):

```sh
cd wyoming-satellite/
sudo bash etc/install-respeaker-drivers.sh
```

After install the drivers, you must reboot the satellite:

```sh
sudo reboot
```

Once the satellite has rebooted, reconnect over SSH and continue the installation:

```sh
cd wyoming-satellite/
python3 -m venv .venv
.venv/bin/pip3 install --upgrade pip
.venv/bin/pip3 install --upgrade wheel setuptools
.venv/bin/pip3 install \
  -f 'https://synesthesiam.github.io/prebuilt-apps/' \
  -r requirements.txt \
  -r requirements_audio_enhancement.txt \
  -r requirements_vad.txt
```

If the installation was successful, you should be able to run help and get ba help list:
```sh
script/run --help
```

## Determine Audio Devices

Picking the correct microphone/speaker devices is critical for the satellite to work. We'll do a test recording and playback in this section.

List your available microphones with:

```sh
arecord -L
```

Record a 5 second sample from your chosen microphone:

```sh
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav
```

Say something while `arecord` is running.

Play back your recorded sample WAV:

```sh
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav
```

You should hear your recorded sample.


## Create Services

You can run wyoming-satellite as a systemd service by first creating a service file:

``` sh
sudo systemctl edit --force --full wyoming-satellite.service
```

Paste in the following template, and change both `/home/pi` and the `script/run` arguments to match your set up:

```text
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target
Requires=wyoming-openwakeword

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run --name 'my satellite' --uri 'tcp://0.0.0.0:10700' --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' --snd-volume-multiplier 1.0 --mic-auto-gain 5 --mic-noise-suppression 1 --done-wav sounds/done.wav --awake-wav sounds/awake.wav
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Save the file and exit your editor.


## Local Wake Word Detection

Install the necessary system dependencies:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends  \
  libopenblas-dev
```

From your home directory, install the openWakeWord Wyoming service:

```sh
git clone https://github.com/spiyderz/wyoming-openwakeword.git
cd wyoming-openwakeword
script/setup
```

Create a systemd service for it:

``` sh
sudo systemctl edit --force --full wyoming-openwakeword.service
```

Paste in the following template, and change both `/home/pi` and the `script/run` arguments to match your set up:

```text
[Unit]
Description=Wyoming openWakeWord

[Service]
Type=simple
ExecStart=/home/pi/wyoming-openwakeword/script/run --uri 'tcp://127.0.0.1:10400'  -threshold 0.90
WorkingDirectory=/home/pi/wyoming-openwakeword
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Save the file and exit your editor.

Enable openwakeword and wyoming services, reload and restart the satellite service:

``` sh
sudo systemctl enable --now wyoming-openwakeword.service
sudo systemctl enable --now wyoming-satellite.service
```

You should see the wake service get automatically loaded:

``` sh
sudo systemctl status wyoming-satellite.service wyoming-openwakeword.service
```

They should all be "active (running)" and green.

Go to your home assistant page and you should see the discovered wyoming satellite. Add it.


Test out your satellite by saying "ok, nabu" and a voice command. Use `journalctl` to check the logs of services for errors:

``` sh
journalctl -u wyoming-openwakeword.service -f
```

Make sure to run `sudo systemctl daemon-reload` every time you make changes to either service file.


## ChatGPT through OPENAI Extended Conversation

You can install OpenAI Extended Conversation using this tutitorial on Github

https://github.com/jekalmin/extended_openai_conversation


## Exposing devices to Assist

Make sure that the entites you want to control are exposed to assist.

