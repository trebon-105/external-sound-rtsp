Mixing an external analog audio to a RTSP stream from a security camera.

## Hardware

This guide assumes you are using

- Raspberry Pi 4 Model B
- HiFiBerry DAC+ ADC HAT for stereo analog audio input

## Setup

1. Install fresh [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems) via [Raspberry Pi imager](https://www.raspberrypi.com/software) onto a SD card; don't forget to [set a password](https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/) and enable SSH in settings

2. SSH to the Raspberry using the username and password you set

   ```shell
   ssh pi@192.168.105.123
   ```

3. Update & upgrade

   ```shell
   sudo apt-get update && sudo apt-get upgrade -y
   ```

4. Update hostname & set timezone

   ```shell
   sudo raspi-config nonint do_hostname raspi-rtsp
   sudo timedatectl set-timezone Europe/Prague
   ```

5. [Configure HiFiBerry](https://www.hifiberry.com/docs/software/configuring-linux-3-18-x): comment out line with `dtparam=audio=on` in `/boot/config.txt` and add the following below:

   ```
   # Enable HiFiBerry DAC+ ADC
   dtoverlay=hifiberry-dacplusadc
   ```
   
   and also replace the line
   
   ```
   dtoverlay=vc4-kms-v3d
   ```
   
   with 
   
   ```
   dtoverlay=vc4-kms-v3d,audio=off
   ```

6. Reboot and verify the sound card is recognized

   ```shell
   pi@raspi-rtsp:~ $ arecord -L
   null
       Discard all samples (playback) or generate zero samples (capture)
   default
       Default Audio Device
   sysdefault
       Default Audio Device
   hw:CARD=sndrpihifiberry,DEV=0
       snd_rpi_hifiberry_dacplusadc, HiFiBerry DAC+ADC HiFi multicodec-0
       Direct hardware device without any conversions
   plughw:CARD=sndrpihifiberry,DEV=0
       snd_rpi_hifiberry_dacplusadc, HiFiBerry DAC+ADC HiFi multicodec-0
       Hardware device with all software conversions
   default:CARD=sndrpihifiberry
       snd_rpi_hifiberry_dacplusadc, HiFiBerry DAC+ADC HiFi multicodec-0
       Default Audio Device
   sysdefault:CARD=sndrpihifiberry
       snd_rpi_hifiberry_dacplusadc, HiFiBerry DAC+ADC HiFi multicodec-0
       Default Audio Device
   dsnoop:CARD=sndrpihifiberry,DEV=0
       snd_rpi_hifiberry_dacplusadc, HiFiBerry DAC+ADC HiFi multicodec-0
       Direct sample snooping device
   ```

7. Create `/etc/asound.conf` file with the following content (the card name has to match the one in the command output above) and reboot again

    ```shell
    sudo tee /etc/asound.conf >/dev/null << EOF
    pcm.!default {
      type hw card "CARD=sndrpihifiberry,DEV=0"
    }
    ctl.!default {
      type hw card "CARD=sndrpihifiberry,DEV=0"
    }
    EOF
    ```

8. Install ffmpeg

    ```shell
    sudo apt install ffmpeg -y
    ```

9. Download the [latest version](https://github.com/aler9/rtsp-simple-server/releases) of [rtsp-simple-server](https://github.com/aler9/rtsp-simple-server) for ARMv7
    
    ```shell
    wget https://github.com/aler9/rtsp-simple-server/releases/download/v0.19.1/rtsp-simple-server_v0.19.1_linux_armv7.tar.gz
    tar -xvzf rtsp-simple-server_v0.19.1_linux_armv7.tar.gz
    rm rtsp-simple-server_v0.19.1_linux_armv7.tar.gz
    sudo mv rtsp-simple-server /usr/local/bin/
    sudo mv rtsp-simple-server.yml /usr/local/etc/
    ```

10. Fill in RTSP stream URL in the following and add it under the `paths` key at the end of the `/usr/local/etc/rtsp-simple-server.yml` file (see [FFmpeg command explained below](#ffmpeg-command))

    ```yaml
      stage:
        runOnInit: >
          ffmpeg
            -itsoffset 0
            -thread_queue_size 8
            -rtsp_transport tcp
            -i rtsp://unifi-nvr:7447/DSer53oue3awK9ft 
            -f alsa
            -itsoffset 0
            -thread_queue_size 8
            -ar 44100
            -i hw:CARD=sndrpihifiberry
            -c:v copy
            -map 0:v:0
            -map 1:a:0
            -f rtsp
            -rtsp_transport tcp
            rtsp://localhost:$RTSP_PORT/$RTSP_PATH
        runOnInitRestart: yes
     ```

11. Create the service to start rtsp-simple-server on boot

    ```shell
    sudo tee /etc/systemd/system/rtsp-simple-server.service >/dev/null << EOF
    [Unit]
    After=network.target
    [Service]
    ExecStart=/usr/local/bin/rtsp-simple-server /usr/local/etc/rtsp-simple-server.yml
    [Install]
    WantedBy=multi-user.target
    EOF
    ```

12. Enable and start the service

    ```shell
    sudo systemctl enable rtsp-simple-server
    sudo systemctl start rtsp-simple-server
    ```

## FFmpeg command

Achieving low-latency RTSP restreaming with FFmpeg is a bit tricky. Here is the FFmpeg command with comments:

```bash
ffmpeg \
  -itsoffset 0 \ # video delay to sync with audio
  -thread_queue_size 8 \ # try using as low value as possible
  -rtsp_transport tcp \ # use TCP for RTSP
  -i rtsp://unifi-nvr:7447/DSer53oue3awK9ft \ # RTSP URL of the security camera (see below)
  -f alsa \ # audio input from ALSA
  -itsoffset 0 \ # audio delay to sync with video
  -thread_queue_size 8 \ # try using as low value as possible
  -ar 44100 \ # audio 44 100 Hz sampling rate
  -i hw:CARD=sndrpihifiberry \ # capture sound from HiFiBerry
  -c:v copy \ # copy video codec
  -map 0:v:0 \ # map video input stream #0 to output #0
  -map 1:a:0 \ # map audio input stream #1 (sound card) to output #0
  -f rtsp \ # use RTSP for output
  -rtsp_transport tcp \ # use TCP for RTSP
  rtsp://localhost:$RTSP_PORT/$RTSP_PATH # use the default port and path defined as YAML key
```

### UniFi Protect RTSP stream

You can turn on RTSP in UniFi Protect (go to camera, settings & advanced). You'll see the following URL:

```
rtsps://{ UniFi NVR IP address }:7441/{ secret token }?enableSrtp
```

We don't want to use RTSP over SSL, so just pick the secret token and use the URL with port `7447`. Also you can set the NVR hostname instead of IP address in case it changes.

```
rtsp://{ UniFi NVR hostname }:7447/{ secret token }
```
