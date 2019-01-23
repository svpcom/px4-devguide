# Long-distance Video Streaming and telemetry via raw WiFi radio

This page shows how to set up a companion computer with a camera (Logitech C920 or RaspberryPi camera) such that the video stream is transferred from the UAV to a ground computer and displayed in *QGroundControl*. Also it provides support for bidirectional telemetry like SiK radio does. This setup uses WiFi in unconnected (broadcast) mode and software from the [Wifibroadcast project](https://github.com/svpcom/wifibroadcast/wiki).


## Wifibroadcast Overview

The *Wifibroadcast project* provides video and telemetry transport which use low-level WiFi packets to avoid distance and latency limitations of ordinary ieee80211 stack.

> **Note** Before using *Wifibroadcast* check regulators allow this kind of WiFi use in your country. 

The high level benefits of *Wifibroadcast* include:

 - 1:1 map RTP to IEEE80211 packets for minimum latency (doesn't serialize to byte steam)
 - Smart FEC support (immediately yeild packet to video decoder if FEC pipeline without gaps)
 - [Bidirectional mavlink telemetry](https://github.com/svpcom/wifibroadcast/wiki/Setup-HOWTO). You can use it for mavlink up/down and video down link.
 - Automatic TX diversity (select TX card based on RX RSSI)
 - Stream encryption and authentication ([libsodium](https://download.libsodium.org/doc/))
 - Distributed operation. It can gather data from cards on different hosts. So you don't limited to bandwidth of single USB bus.
 - Aggreagation of mavlink packets. Doesn't send wifi packet for every mavlink packet.
 - Enhanced [OSD](https://github.com/svpcom/wifibroadcast_osd) for Raspberry PI (consume 10% CPU on PI Zero)
   Compatible with any screen resolution. Supports aspect correction for PAL to HD scaling.
   
Additional information is provided in the [FAQ](#faq) below.


## Hardware Setup

The hardware setup consists of the following parts:

On TX (UAV) side:
* [NanoPI NEO2](http://www.friendlyarm.com/index.php?route=product/product&product_id=180) (and/or Raspberry Pi if use Pi camera).
* [Logitech camera C920](https://www.logitech.com/en-us/product/hd-pro-webcam-c920?crid=34) or [Raspberry Pi camera](https://www.raspberrypi.org/products/camera-module-v2/).
* WiFi module  [ALPHA AWU036ACH](https://www.alfa.com.tw/WiFi%20USB%20Antenna.html).

On RX (ground station side):
* Any computer with Linux (tested on Fedora 25 x86-64).
* WiFi module  [ALPHA AWU036ACH](https://www.alfa.com.tw/WiFi%20USB%20Antenna.html). See [wifibroadcast wiki > WiFi hardware](https://github.com/svpcom/wifibroadcast/wiki/WiFi-hardware) for more information on supported modules.

If you don't need high-power cards, you can use any card with **rtl8812au** chipset.

## Hardware Modification

Alpha AWU036ACH is a high power card that uses too much current while transmitting. If you power it from USB it will reset the port on most ARM boards.
So you need to connect it to 5V BEC directly. You can do this two ways:

1. Make a custom USB cable. [You need to cut ``+5V`` wire from USB plug and connect it to BEC](https://electronics.stackexchange.com/questions/218500/usb-charge-and-data-separate-cables)
2. Cut a ``+5V`` wire on PCB near USB port and wire it to BEC. Don't do this if doubt. Use custom cable instead!
   Also I suggest to add 470uF low ESR capacitor (like ESC has) between power and ground to filter voltage spikes. Be aware of [ground loop](https://en.wikipedia.org/wiki/Ground_loop_%28electricity%29) when using several ground wires.

   
## Software Setup

1. Install **libpcap** and **libsodium** development libs and install **python2.7** + **python-twisted** packages.
2. Download [wifibroadcast sources](https://github.com/svpcom/wifibroadcast).
3. See [Setup HOWTO](https://github.com/svpcom/wifibroadcast/wiki/Setup-HOWTO) how to build debian, rpm or tar.gz package and configure it.

### UAV Setup

1. Setup camera to output RTP stream:

   a. Logitech camera C920 camera:
      ```
      gst-launch-1.0 uvch264src device=/dev/video0 initial-bitrate=4000000 average-bitrate=4000000 iframe-period=3000 name=src auto-start=true \
               src.vidsrc ! queue ! video/x-h264,width=1280,height=720,framerate=30/1 ! h264parse ! rtph264pay ! udpsink host=localhost port=5602
      ```
   b. RaspberryPi camera:
      ```
      raspivid --nopreview --awb auto -ih -t 0 -w 1280 -h 720 -fps 49 -b 4000000 -g 147 -pf high -o - | gst-launch-1.0 fdsrc ! h264parse !  rtph264pay !  udpsink host=127.0.0.1 port=5602
      ```
 2. Configure WFB for drone as described in [Setup HOWTO](https://github.com/svpcom/wifibroadcast/wiki/Setup-HOWTO) 
 3. Configure autopilot (px4 stack) to output telemetry stream at 1500kbps (other UART speeds doesn't match well to NEO2 frequency dividers). Setup ``malink-router`` to route mavlink packets to/from WFB:
    ```
    [UdpEndpoint wifibroadcast]
    Mode = Normal
    Address = 127.0.0.1
    Port = 14550
    ```
 
### Ground Station Setup

1. Run qgroundcontrol or
   ```
   gst-launch-1.0 udpsrc port=5600 caps='application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264' \
             ! rtph264depay ! avdec_h264 ! clockoverlay valignment=bottom ! autovideosink fps-update-interval=1000 sync=false
   ```
   to decode video.
2. Configure WFB for ground station as described in [Setup HOWTO](https://github.com/svpcom/wifibroadcast/wiki/Setup-HOWTO)

## Tuning radio settings
With default settings WFB use radio channel 165 (5825 MHz), width 20MHz, MCS #1 (QPSK 1/2) with long GI. This provides ~7 mbit/s of **effective** speed (i.e. usable speed after FEC and packet encoding) for **both directions** in sum, because WiFi is half-duplex. So it is sutable for video down stream 720p@49fps (4 mbit/s) + two full-speed telemetry streams (uplink and downlink). If you need a higher bandwidth you can use other MCS index (for example 2 or greater) and/or 40MHz channel. 

## Antennas and diversity
For simple cases you can use omnidirectional antennas with linear (that bundled with wifi cards) or circular (cleaver leaf) polarization. If you want to setup long distance link you can use multipe wifi adapters with directional and omnidirectional antennas. TX/RX diversity for multiple adapters supported out of box (just add multiple NICs to ``/etc/default/wifibroadcast``). If your WiFi adapter has two antennas (like Alfa AWU036ACH) TX diversity is implemented via [STBC](https://en.wikipedia.org/wiki/Space%E2%80%93time_block_code). Cards with 4 ports (like Alfa AWUS1900) are currently not supported for TX diversity (only RX is supported).

## FAQ
Q: What is a difference from original wifibroadcast?

A: Original version of wifibroadcast use a byte-stream as input and splits it to packets of fixed size (1024 by default). If radio packet was lost and this is not corrected by FEC you'll got a hole at random (unexpected) place of stream. This is especially bad if data protocol is not resistent to (was not desired for) such random erasures. So i've rewrite it to use UDP as data source and pack one source UDP packet into one radio packet. Radio packets now have variable size depends on payload size. This is reduces a video latency a lot.

Q: What type of data can be transmitted using wifibroadcast?

A: Any UDP with packet size <= 1466. For example x264 inside RTP or Mavlink.

Q: What are transmission guarancies?

A: Wifibrodcast use FEC (forward error correction) which can recover 4 lost packets from 12 packets block with default settings. You can tune it (both TX and RX simultaniuosly!) to fit your needs.

> **Caution** Don't use band that the RC TX operates on! Or setup RTL properly to avoid model loss.

Q: Is only Raspberry PI supported?

A: Wifibroadcast is not tied to any GPU - it operates with UDP packets. But to get RTP stream you need a video encoder (with encode raw data from camera to x264 stream). In my case RPI is only used for video encoding (becase RPI Zero is too slow to do anything else) and all other tasks (including wifibroadcast) are done by other board (NanoPI NEO2).

## Theory
Wifibroadcast puts the wifi cards into monitor mode. This mode allows to send and receive arbitrary packets without association and waiting for ACK packets.
[Analysis of Injection Capabilities and Media Access of IEEE 802.11 Hardware in Monitor Mode](https://github.com/svpcom/wifibroadcast/blob/master/patches/Analysis%20of%20Injection%20Capabilities%20and%20Media%20Access%20of%20IEEE%20802.11%20Hardware%20in%20Monitor%20Mode.pdf)
[802.11 timings](https://github.com/ewa/802.11-data)


#### What ARM Boards are recommended for the UAV?

Board | Pros  | Cons
--- | --- | ---
[Raspberry Pi Zero](https://www.raspberrypi.org/products/raspberry-pi-zero/) | - Huge community<br>- Camera support<br>- HW video encoder/decoder with OMX API. | - Hard to buy outside US (shipping costs >> its price)<br>- Slow CPU<br>- Only one USB bus<br>- 512MB SDRAM
[Odroid C0](http://www.hardkernel.com/main/products/prdt_info.php?g_code=G145326484280) | - Fast CPU<br>- EMMC<br>- 1GB SDRAM | - Very sensitive to radio interference<br>- Doesn't supported by mainline kernel<br>- High cost<br>- HW video encoder is broken<br>- Bad PCB quality (too thin, ground pins without [thermal relief](https://en.wikipedia.org/wiki/Thermal_relief))
[NanoPI NEO2](http://www.friendlyarm.com/index.php?route=product/product&product_id=180) | - ARM 64-bit CPU<br>- Very cheap<br>- Supported by mainline kernel<br>- 3 independent USB busses<br>- 1Gbps Ethernet port<br>- 3 UARTs<br>- Very small form-factor<br>- Resistant to radio interference | - Small community<br>- 512MB SDRAM<br>- No camera interface

This article chose to use Pi Zero as camera board (encode video) and NEO2 as main UAV board (wifibroadcast, MAVLink telemetry, etc.)


## TODO

1. Make prebuilt images. Pull requests are welcome.
2. Do a flight test with different cards/antennas.
