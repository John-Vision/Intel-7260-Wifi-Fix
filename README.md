# Intel-7260-Wifi-Fix
A fix for Intel 7260 WIFI PCI cards, which intermittently and unpredictably stop working on Linux.
(With a little bit of know how, this might easily be adapted to support other chipsets.)

The Intel 7260 WIFI PCI cards have fantasitic wifi capabilities, but are notorious for intermittently and unpredictably shutting down and becoming completely non-responsive, with no way of restarting the card except for rebooting the system.

After a LOT of searching I found a couple of scripts which could be run that would restart the card. While that was nice, the card would definitely still go down from time to time, and then require the user to manually run the script. This was an improvement, but not very convenient, and I wanted a way to automate the process so I could simply forget about it and have it just work.

I took the script and modified it just a bit, and also added some checks at the beginning of the script, so that it could now determine for itself whether the wifi card was working or no, so:
  (1) If the wifi IS found to be working, the the script simply exits.
  (2) If the wifi IS NOT found to be working, then the script continues and performs the wifi reset.
  
I then set up some cron jobs which would run my modified script every 20 seconds. Once I had this all set up, my wifi problems were over!
