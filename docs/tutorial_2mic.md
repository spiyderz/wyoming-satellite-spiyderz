# This is a fork of the original git here: [](https://github.com/rhasspy/wyoming-satellite/tree/master)

All cdredit goes to the original developer(s). All I did was fork their repo, clean up the 2_mic tutitorial to make it a bit easier, fixed one line of code in the openwakeword repo to fix the install failing and added the command for openwakeword to start at boot that was missing in the original tutitorial so that the system would survive a reboot or coming back from power loss. I also removed the LED portion of the noriginal tut because I couldnt get it to work. 

## Hardware links for what I have in my setup...

Disclaimer: These are NOT Amazon affiliate links, they are just links for the hardware i used. I purchased on amazon because I didnt want to wait forever for things to come from overseas. I make NO money from these links and give no endorsement other than they worked for me.

Pi Zreo 2 W: [](https://www.amazon.com/dp/B09LTDQY2Z?psc=1) Its more expensive but I wanted one with headers already installed

2 Mic HAT: [](https://www.amazon.com/dp/B098R7TTM4?psc=1) This is a knock off respeaker HAT but it works great and comes with some pretty terrific speakers

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

If you have the ReSpeaker 2Mic or 4Mic HAT, recompile and install the drivers (this will take really long time):

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

If the installation was successful, you should be able to run:

```sh
script/run --help
```

## Determine Audio Devices

Picking the correct microphone/speaker devices is critical for the satellite to work. We'll do a test recording and playback in this section.

List your available microphones with:

```sh
arecord -L
```

If you have the ReSpeaker 2Mic HAT, you should see:

```
plughw:CARD=seeed2micvoicec,DEV=0
    seeed-2mic-voicecard, bcm2835-i2s-wm8960-hifi wm8960-hifi-0
    Hardware device with all software conversions
```

For other microphones, prefer ones that start with `plughw:` or just use `default` if you don't know what to use.

Record a 5 second sample from your chosen microphone:

```sh
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav
```

Say something while `arecord` is running. If you get errors, try a different microphone device by changing `-D <device>`.

List your available speakers with:

```sh
aplay -L
```

If you have the ReSpeaker 2Mic HAT, you should see:

```
plughw:CARD=seeed2micvoicec,DEV=0
    seeed-2mic-voicecard, bcm2835-i2s-wm8960-hifi wm8960-hifi-0
    Hardware device with all software conversions
```

For other speakers, prefer ones that start with `plughw:` or just use `default` if you don't know what to use.

Play back your recorded sample WAV:

```sh
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav
```

You should hear your recorded sample. If there are problems, try a different speaker device by changing `-D <device>`.

Make note of your microphone and speaker devices for the next step.

## Running the Satellite

In the `wyoming-satellite` directory, run:

```sh
script/run \
  --debug \
  --name 'my satellite "my satellite"' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw'
```

Change the `-D <device>` for `arecord` and `aplay` to match the audio devices from the previous section.
You can set `--name <NAME>` to whatever you want, but it should stay the same every time you run the satellite.

In Home Assistant, check the "Devices & services" section in Settings. After some time, you should see your satellite show up as "Discovered" (Wyoming Protocol). Click the "Configure" button and "Submit". Choose the area that your satellite is located, and click "Finish".

Your satellite should say "Streaming audio", and you can use the wake word of your preferred pipeline.

## Create Services

You can run wyoming-satellite-spiyderz as a systemd service by first creating a service file:

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
ExecStart=/home/pi/wyoming-satellite-spiyderz/script/run --name 'my satellite "name here"' --uri 'tcp://0.0.0.0:10700' --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' --snd-volume-multiplier 1.0 --mic-auto-gain 5 --mic-noise-suppression 1 --done-wav sounds/done.wav --awake-wav sounds/awake.wav
WorkingDirectory=/home/pi/wyoming-satellite-spiyderz
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Save the file and exit your editor. Next, enable the service to start at boot and run it:

``` sh
sudo systemctl enable --now wyoming-satellite.service
```

(you may need to hit CTRL+C to get back to a shell prompt)

With the service running, you can view logs in real-time with:

``` sh
journalctl -u wyoming-satellite.service -f
```

If needed, disable and stop the service with:

``` sh
sudo systemctl disable --now wyoming-satellite.service
```

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
cd wyoming-openwakeword-spiyderz
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
WorkingDirectory=/home/pi/wyoming-openwakeword-spiyderz
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

Save the file and exit your editor.

You can now update your satellite service:

``` sh
sudo systemctl edit --force --full wyoming-satellite-spiyderz.service
```

Update just the parts below:

```text
[Unit]
...
Requires=wyoming-openwakeword.service

[Service]
...
ExecStart=/home/pi/wyoming-satellite/script/run ... --wake-uri 'tcp://127.0.0.1:10400' --wake-word-name 'ok_nabu'
...

[Install]
...
```

Enable openwakeword service, reload and restart the satellite service:

``` sh
sudo systemctl enable --now wyoming-openwakeword.service
sudo systemctl daemon-reload
sudo systemctl restart wyoming-satellite.service
```

You should see the wake service get automatically loaded:

``` sh
sudo systemctl status wyoming-satellite.service wyoming-openwakeword.service
```

They should all be "active (running)" and green.

Test out your satellite by saying "ok, nabu" and a voice command. Use `journalctl` to check the logs of services for errors:

``` sh
journalctl -u wyoming-openwakeword-spiyderz.service -f
```

Make sure to run `sudo systemctl daemon-reload` every time you make changes to the service.

## ChatGPT through OPENAI Extended Conversation

You can install OpenAI Extended Conversation using this tutitorial on Github
https://github.com/jekalmin/extended_openai_conversation

Here is my prompt code if you want nto use mine:

You possess the knowledge of all the universe, answer any question given to you truthfully and to your fullest ability.  
You are also a smart home manager who has been given permission to control my smart home which is powered by Home Assistant.
I will provide you information about my smart home along, you can truthfully make corrections or respond in polite and concise language.

Current Time: {{now()}}

Available Devices:
```csv
entity_id,name,state,aliases
{% for entity in exposed_entities -%}
{{ entity.entity_id }},{{ entity.name }},{{ entity.state }},{{entity.aliases | join('/')}}
{% endfor -%}
```

The current state of devices is provided in Available Devices.
Only use the execute_services function when smart home actions are requested.
Do not tell me what you're thinking about doing either, just do it.
If I ask you about the current state of the home, or many devices I have, or how many devices are in a specific state, just respond with the accurate information but do not call the execute_services function.
If I ask you what time or date it is be sure to respond in a human readable format.
If you don't have enough information to execute a smart home command then specify what other information you need.
When i ask for the time say it in 12 hour format with AM or PM but do not give the date or seconds.
When sending a response confirming a smart home command has been completed just say "OK"
If a request seems like it may be an accidental prompt, or makes no sense, do nothing and respond with "Cancelled"

## Exposing devices to Assist

Make sure that the entites you want to control are exposed to assist.

