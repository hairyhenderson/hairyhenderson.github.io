+++
date = "2015-06-07T21:05:51-04:00"
draft = true
title = "sensor madness"
categories = [ "electronics" ]

+++

So, I decided I wanted to play with some of these ESP8266 boards
I'd been hearing so much about.

- bought three Adafruit ESP8266 Huzzah! boards and three DHT22 modules
- had a terrible time configuring the Arduino IDE for the board - everything's probably just too new
- finally figured out the key - gotta set some value to `26`:
```cpp
// Need to change cycle count threshold since the ESP8266 works at 80MHz
// 26 seems to put it into a state that agrees with my house's thermostat...
DHT dht(DHTPIN, DHTTYPE, 26);
```

Now I've got a little sketch that:

1. reads WiFi SSID/pass from EEPROM
2. connects to WiFi, and registers `esp8266.local` with mDNS
3. responds to HTTP `GET`s by reading from the DHT22 and sending back some JSON:
```console
$ curl esp8266.local
{"temperature":28.00,"humidity":36.00,"heatIndex":27.41}
```

Still to do:

- come up with a name for the thing that doesn't sound weird, or get used to the weird name I came up with (_roomfly_)
- figure out batteries, and figure out power saving measures (deep sleep?)
- get rid of the HTTP API and get it to talk to an MQTT server instead ([PubSubClient](https://github.com/knolleary/pubsubclient) seems good)
- figure out which MQTT server to get it to talk to ;)
- build an enclosure for it (actually, build 3 enclosures)

woot.
