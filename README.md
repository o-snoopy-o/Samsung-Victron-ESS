# Samsung-Victron ESS Project

## Introduction

I managed to win some very well-priced Samsung SDI lithium batteries in a local auction. Initially, I knew I would be taking a bit of a risk but took comfort in the fact that I could strip the cells from the housing and use them regardless of my success or otherwise using them as intended.

I set about researching the best way to interface with these batteries and found some information on this Reddit. The posts gave me confidence that I could in fact interface directly with the existing BMS without having to strip them and use a different BMS.

Without the manual posted in the Reddit, I would have found it extremely difficult to proceed, so thank you to the poster.

Initial attempts at RS485 communication proved to be difficult, and I swapped between RS485 and CAN-BUS a few times before I managed to get some meaningful data over CAN-BUS.

I am now powering my workshop fully using the system, with plans to integrate it into my home wiring once I have some more time available. 48.4kWh of storage at a fraction of the cost of a Tesla Powerwall :p

I was not planning on releasing any information at this time, but due to several personal messages via Reddit, it seems many people in possession of these same batteries, having taken advantage of the liquidation of SENEC Australia, require some guidance on how best to utilize them.

I do not claim to be an expert in any way and caution anyone following my lead to do so with their own due diligence, but I thought it appropriate to share my experiences for the good of the community.

## Scope

This project interfaces the Samsung ELPM482-00005 battery with the Victron ecosystem using the dbus-mqtt-battery driver by [mr-manuel](https://github.com/mr-manuel/venus-os_dbus-mqtt-battery).

I am using 10x batteries currently, and the code supports 10x batteries.

As an intermediary, the Arduino accepts the CAN-BUS data, performs translation, and sends it to the driver via MQTT. The Arduino code aims to eliminate null values and average data where required so that the Victron ecosystem does not react so quickly when invalid data is presented.

The Victron ecosystem may consist of a Cerbo-GX (in my case) or another 'Venus OS' device that accepts MQTT data and presents it on the Victron dbus.

I am using a **Multiplus-II 48/10000/140-100/100** as an inverter/charger.

## Documented

I will share my known running code for all elements of the system. The code is a work in progress, and there are no guarantees expressed or implied. The code works for *my* needs, which may differ from yours. I will be updating the code as I progress through this project, but it will be slow and may not suit your needs. **Use at your own risk.**

## Design

The batteries communicate over CAN-BUS directly to the CAN-BUS shield on the Arduino UNO R4 WiFi. The RJ45 connector must be wired specifically to ensure the CAN high and low are connected to the DB9 connector on the shield. The adapter shown was the cleanest way to enable this wire matching.

![Wiring and Adapter - Coming Soon](IMAGE_LINK)

The batteries can be interlinked with RJ45 Ethernet cables, with special attention paid to the correct ports and termination resistors.

![TR](https://github.com/o-snoopy-o/Samsung-Victron-ESS/blob/main/images/Screenshot%202024-10-24%20204348.png)
![CAN-BUS](https://github.com/o-snoopy-o/Samsung-Victron-ESS/blob/main/images/Screenshot%202024-10-24%20204236.png)

The Arduino can be configured using Arduino IDE. Modify the provided code appropriately to define your MQTT broker. In my case, it was the Cerbo GX on `192.168.1.200/24`.

Further, ensure that all the desired registers are accounted for and that the JSON data packets match the format given for dbus-mqtt-battery.

Follow the guide at dbus-mqtt-battery to enable root access to 'Venus OS' and install the driver.

You can directly copy my configuration for dbus-mqtt-battery if you’d like. I find **WinSCP** to be the most convenient way of modifying files in Windows for the remote systems.

## Result

After wiring the connections, uploading the sketch to the Arduino, and configuring the ‘Venus OS’, you should now see battery parameters available in the ‘Venus OS’ console. If you haven’t already done so, you can configure ESS assistants for your Multiplus-II.

![VRM Example](https://github.com/o-snoopy-o/Samsung-Victron-ESS/blob/main/images/Screenshot%202024-10-23%20131711.png)

## Further

I chose to take it a little further and try to implement some system monitoring. The main reason for this was to enable further logging than what is available in the Victron VRM portal.

I used **Telegraf** to collect the data from the Cerbo as a broker. This gives both Victron data and battery data. I also used Telegraf to collect Modbus data from my Fronius inverter and took all that data into **InfluxDB2**. Once the data was being stored in the appropriate buckets, I built some Grafana dashboards to visualize the data.

All 3 services are running on a **Debian 6.1.106-3** instance within Docker containers. The 3 containers have persistent data volumes. The server is internally hosted with plans to transition to a Docker swarm once time allows.

The dashboards are always a work in progress and may never be complete. I would also like to implement a **PowerBI streaming dataset**, but I’m still trying to get my head around how it works. It seems that the data must be manipulated and structured before pushing to the PowerBI service. This is on the long-term roadmap.

## Step by Step Guide
But first a warning!
Do not connect batteries of differing State of Charge. If you have a bank of more than 1 battery, they should be sitting at a very similar voltage otherwise HUGE currents can flow between batteries in parallel causing the 63amp fuse to blow or worse, causing the battery BMS to enter a fault state that may not be recoverable. Ensure that the batteries are close in voltage (within 0.01v) and hook them all up together or charge them all to the same voltage before connecting in parallel. You may also connect them via a suitable resistor to limit the inrush current, but this can take a LOOONG time to equalise, try and avoid this where possible. You can use the direct terminals that bypass the BMS to check the voltage as the output terminals will not be live with the BMS off.

Resources: ([Blog?](https://elpm482-00005.blogspot.com/2024/06/samsung-elpm482-00005.html))

## Parts
- Nx Samsung ELPM482-00005 Batteries
- 1x Keyestudio KS0411 CAN-BUS Shield ([Wiki](https://wiki.keyestudio.com))
- 1x Arduino UNO R4 WiFi ([Documentation](https://www.arduino.cc/))
- 1x DB9 Female To RJ45 Modular Adapter ([Altronics](https://www.altronics.com.au/))
- 1x RJ45 Ethernet Cable (Straight Through)


### Part 1 – Battery and Arduino

1. Wire your DB9 to RJ45 adapter to match the image.
2. Connect the CAN-BUS on Battery 1 to the DB9 Adapter. Ensure the cable is inserted into the correct port on the battery. Termination resistors don’t really matter at this point, but it’s worth setting anyway.
3. Upload the customized sketch to the Arduino. Ensure you set the Cerbo-GX broker IP Address or hostname correctly. Input your SSID and key. If you’re using an Ethernet shield, this is out of scope, and you need to modify the code appropriately.
4. Ensure the serial monitor is set to the correct Baud Rate and watch for feedback.
5. Power on the battery by holding the power button for about 3 seconds.
6. Observe the serial monitor in Arduino IDE for activity. If you are getting positive feedback, move on to the next step. Leave everything as is.

### Part 2 – Configure the ‘Venus OS’

1. Set your Cerbo-GX or other compatible device to have a static IP address or make the DHCP lease static for the device in your DHCP server.
2. Enable SSH to Cerbo GX, [Venus OS: Root Access](https://www.victronenergy.com/live/venus-os:root_access).
3. SSH into the Cerbo GX with Putty.
4. Install the dbus-mqtt-battery driver into ‘Venus OS’ by following [this guide](https://github.com/mr-manuel/venus-os_dbus-mqtt-battery).
5. Use WinSCP to access the dbus-mqtt-battery config file and edit as required, or use a suitable editor like **nano** through Putty.

The bare minimum configuration is:

```json
{
    "Dc": {
        "Power": 321.6,
        "Voltage": 52.7
    },
    "Soc": 63,
    "Info": {
        "MaxChargeVoltage": 55.2,
        "MaxChargeCurrent": 80.0,
        "MaxDischargeCurrent": 120.0
    }
}
```

At this point, the Victron ecosystem should identify the battery and respond to the charge voltage limit and charge current limit, likewise for the discharge current limit.

![Parameters](https://github.com/o-snoopy-o/Samsung-Victron-ESS/blob/main/images/Screenshot%202024-10-23%20132123.png)

If you have more than one battery and the interface does not show all batteries, check you have daisy-chained the CAN-BUS properly and that the termination resistors are set appropriately.

![Modules](https://github.com/o-snoopy-o/Samsung-Victron-ESS/blob/main/images/Screenshot%202024-10-23%20132032.png)

### Part 3 – Observe for Correct Data

The battery supports a maximum charge current of **47 Amps** per module at **58.1VDC**. The maximum discharge current is also **47 Amps** per module. These specifications change with cell temperature, so derate your system accordingly to protect the batteries.





## Future Plans

I will monitor the system for cell imbalance and battery balancing. The Grafana dashboard displays the cell imbalance or disparity, and I will be keeping an eye on this more closely in the coming weeks.
![Grafana](https://github.com/o-snoopy-o/Samsung-Victron-ESS/blob/main/images/Screenshot%202024-10-23%20131531.jpg)

---
