set SSID password
```
# /etc/wpa_supplicant.conf
ctrl_interface=/var/run/wpa_supplicant
update_config=1
country=PT

network={
    ssid="YourSSID"
    psk="YourWifiPassword"
}
```

On this NXP stack, `wfd0` is the Wi-Fi Direct interface, while normal station/client bring-up uses `mlan0`. Start `mlan0`:
```sh
wpa_supplicant -B -i mlan0 -c /etc/wpa_supplicant.conf
iw dev mlan0 link
```

Obtain IP address:
```sh
udhcpc -i mlan0
ip addr show dev mlan0
ip route
```

test:
```sh
ping 1.1.1.1
```
