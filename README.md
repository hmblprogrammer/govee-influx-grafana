# govee-influx-grafana
Temperature monitoring with Govee devices, InfluxData Telegraf, InfluxDB and Grafana

This project documents how to configure [InfluxData's Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) data collection agent to collect temperature and humidity data from the [Govee series of Thermo-Hygrometers](https://us.govee.com/collections/thermo-hydrometer), producing dashboards with rich graphs using [Grafana](https://grafana.com).

Govee data is collected from [GoveeBTTempLogger](https://github.com/wcbonner/GoveeBTTempLogger) which runs on a Linux
machine or Raspberry Pi and collects data from the Govee devices using low energy Bluetooth advertisements.

## Prerequisites:

In order to get going, you will need:

1. **Some Govee Thermo-Hygrometers** (at least one) which can be purchased through the Govee store or amazon.com.
   Any of the Govee Thermo-Hygrometers should work, including the indoor and outdoor models, and models with an LCD display.
   
   *Note:* You might want to **start with the devices powered off**, otherwise it might be difficult to identify which device is which. My methodology here was to _leave all devices powered off_, and power them on _one at a time_, allowing them to be discovered and adding them to Telegraf one at a time, so I was sure each device's location in my home was properly tagged.
   
1. **A host to run [GoveeBTTempLogger](https://github.com/wcbonner/GoveeBTTempLogger) and [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) on**.
   
   Personally, I am using a Raspberry Pi Series 4.
   
1. **An InfluxDB Server and Grafana installation** to publish metrics to and display graphs. You can 
    [use InfluxDB Cloud for free](https://www.influxdata.com/products/influxdb-cloud/), and [Grafana has documentation on connecting to an InfluxDB server](https://grafana.com/docs/grafana/latest/getting-started/getting-started-influxdb/)

## Setup Instructions

The following steps describe how to get up and running and have temperature and humidity graphs of your environment:

### Building GoveeBTTempLogger

**Note:** If you have multiple Govee devices with similar temperatures, you may want to ensure they are all powered off before you begin. This helps you identify and tag each device's location when publishing data to InfluxDB.

Detailed instructions for building GoveeBTTempLogger are available on [the GoveeBTTempLogger GitHub site](https://github.com/wcbonner/GoveeBTTempLogger); Here is an abridged version:

1. Check out the source code of [GoveeBTTempLogger](https://github.com/wcbonner/GoveeBTTempLogger) to the host which will be collecting data (E.G. a Raspberry Pi) using git (`git clone https://github.com/wcbonner/GoveeBTTempLogger.git` or
   `git clone git@github.com:wcbonner/GoveeBTTempLogger.git`)
1. Install prerequisite build software with:
    ```sudo apt-get install bluetooth bluez libbluetooth-dev```
   (for Ubuntu/Raspbian; commands for other distros will vary)
1. Build GoveeBTTempLogger using:
    ```make deb```
1. Install using:
    ```sudo dpkg -i GoveeBTTempLogger.deb```

### Configuring GoveeBTTempLogger

By default, GoveeBTTempLogger generates SVG files which we don't need (we have Grafana graphs!) so we can disable the built-in SVG graphs. You may want to also adjust verbosity, or other configuration of GoveeBTTempLogger.

1. To start creating a user override of the systemd unit file for GoveeBTTempLogger, run:
     ```sudo systemctl edit goveebttemplogger.service``` 
1. This will open your preferred `$EDITOR`. In the editor, enter the following:
    ```
    [Service]
    Environment="SVGARGS="
    Environment="VERBOSITY=0"
    ```
    The `VERBOSITY` line is *optional* and can be adjusted to your preference. You may want a number higher than `1` when starting, to be able to see logs with E.G. `sudo journalctl -eu goveebttemplogger.service`, and reduce verbosity when you know everything is working.
1. Run:
     ```sudo systemctl daemon-reload```
     to ensure that systemd is up-to-date
1. Run: 
    ```sudo systemctl restart goveebttemplogger.service``` 
    to restart the GoveeBTTempLogger service and apply the changes
1. Check the service status with:
     ```sudo systemctl status goveebttemplogger.service```
    make sure that it says: `Active: active (running)`

GoveeBTTempLogger should now be listening for Bluetooth Low Energy Advertisements from your Govee devices. If you haven't removed the tabs from the battery compartments of the Govee devices, you should do so now... _probably just starting with just one_, so you can identify its ID.

You can see if GoveeBTTempLogger has found devices and is writing data by checking the `/var/log/goveebttemplogger` directory. If everything is working properly, you should see a text file in that directory for every Govee device that GoveeBTTempLogger has found.

### Initial Telegraf Setup

This section could be expanded with more details; for now, 
[the InfluxDB documentation](https://docs.influxdata.com/influxdb/v2.0/write-data/no-code/use-telegraf/) is your best resource.

1. Download and install Telegraf from [https://portal.influxdata.com/downloads/](https://portal.influxdata.com/downloads/)
1. Configure Telegraf to write data to your InfluxDB server. One way to do this is [Using InfluxDB to generate a telegraf config](https://docs.influxdata.com/influxdb/v2.0/write-data/no-code/use-telegraf/auto-config/)
1. Ensure telegraf is running and enabled with `sudo systemctl enable --now telegraf`


### Configuring Telegraf to read Govee data

We're ready to actually start ingesting the data from GoveeBTTempLogger into Telegraf. Here's where having the GoveeBTTempLogger devices powered off and adding one at a time comes into play. The following process should be repeated for every Govee device you have:

1. If you haven't done so yet, power on the Govee device and ensure that GoveeBTTempLogger can see it.
   Look in the `/var/log/goveebttemplogger` and identify the text file for the Govee device you just added.
   One way to do this is using `ls -lhS /var/log/goveebttemplogger` and looking for the last (smallest) file. 
   If you _just_ added a new Govee device, it will be the smallest text file in that directory.
   
   The filename will have the format: `gvhXXXX_SSSSSSSSSSSS-YYYY-MM.txt`, where `XXXX` is a model number, and `SSSSSSSSSSSS` is a serial number, and `YYYY-MM` is the current year and month.
1. Add a telegraf configuration for this Govee device:
    1. Edit (creating, if this is the first device) the file `/etc/telegraf/telegraf.d/govee.conf`
    1. Add an `inputs.tail` input to the ned of this file read the Govee text file step #1, as follows:
       ```
        [[inputs.tail]]
        alias = "govee"
        name_override = "govee"
        files = ["/var/log/goveebttemplogger/gvhXXXX_SSSSSSSSSSSS-*.txt"]
        data_format = "csv"
        csv_header_row_count = 0
        csv_column_names = ["time", "temperature", "humidity", "battery_level"]
        csv_column_types = ["string", "float", "float", "float"]
        csv_delimiter = "\t"
        csv_timestamp_column = "time"
        csv_timestamp_format = "2006-01-02 15:04:05"
        [inputs.tail.tags]
            location = "Add some location here..."
            sublocation = "Add an optional secondary location here"
            id = "SSSSSSSSSSSS"
       ```
       **Note:** You need to edit the above config to match your actual values:
           * Replace `gvhXXXX_SSSSSSSSSSSS` with the actual filename _without the date part_ from the first step
               * Change the `location` tag to something meaningful, like "outside", or "fridge"
               * Optionally use the `sublocation` tag to further clarify the location, like "porch" or "vegetable drawer"
               * Change the `id` tag to the serial number of the Govee (the `SSSSSSSSSSSS` part of the filename. Or include the
                 `XXXX` part. or omit this altogether, it's just for your reference)
1. Restart Telegraf using `sudo systemctl restart telegraf`
1. Optionally check to ensure the data is in Grafana before proceeding
1. Repeat for every Govee device you have

### Create a Grafana dashboard

Create a Grafana dashboard to visualize your new temperature and humidity data however you like!

In your Grafana dashboard, the `measurement` will be `govee`, and the fields will be: `temperature`, `humidity` and `battery_level`

You can use `GROUP BY` to separate temperatures by location and/or sub_location. For example: `GROUP BY time($__interval), "location", "sublocation", "id"` to group by both `location` and `sublocation`.

I have personally decided to have a `location` and `sublocation` tag to be able to get broader or finer granularity of a given location in my graphs. For example, I might want to have two thermometers in my refrigerator, or multiple thermometers outside. If I select to `GROUP BY tag(sublocation)` then I can see each part of my refrigerator or outside temperature separately; if I do not group by sublocation then I get an average for that location.

(In the case of my refrigerator, I find that it cools pretty unevenly; in the case of outdoor temperatures, I am concerned with how proximity to my home affects the accuracy of the actual ambient air temperature measurement. You may not have as many devices as I do nor may be as concerned about aggregation to this degree, and if so can omit the `sublocation` tag)

The Grafana dashboard I created is available to be imported from the `grafana/` directory. Download this dashboard and import into your Grafana install to start with what I use as a reasonable default.

## Next Steps

Things to consider after you have a working setup include:

* Add Pushover alarms in case the temperature of your refrigerator freezer gets out of range
* Look at dat-over-day or week-over-week temperature graphs of your outside temperatures to track weather patterns
* Integrate temperature graphs with a smart dehumidifier or HVAC system to keep humidity levels within safe levels
* Look at graphs of your environment and amaze your friend with what a dork you are and how much you ❤️ data! 😂

## Notes

* This is a very rough, initial documentation of my initial effort. This may be incomplete, confusing or downright wrong!
  Please supply feedback so I can improve it.
* `GoveeBTTempLogger` writes all temperature values in Celsius/centigrade. I have crafted my own Grafana dashboard to convert to Fahrenheit using: `val * 1.8) + 32`. Remove this if you're in a region which uses Celsius.
* Please feel free to submit PRs against my documentation, or against [GoveeBTTempLogger](https://github.com/wcbonner/GoveeBTTempLogger) (for which I am not the original author)