# Intel-7260-Wifi-Fix

(Please leave a comment - and maybe an upvote - to let me know how this worked for you: https://askubuntu.com/questions/1262557/cannot-get-intel-7260-to-work-properly-on-ubuntu-20-04-disconnects-intermittent)

A fix for Intel 7260 WIFI PCI cards, which intermittently and unpredictably stop working on Linux.
(With a little bit of know how, this might easily be adapted to support other chipsets.)

The Intel 7260 WIFI PCI cards have fantasitic wifi capabilities, but are notorious for intermittently and unpredictably shutting down and becoming completely non-responsive, with no way of restarting the card except for rebooting the system.

After a LOT of searching I found a couple of scripts which could be run that would restart the card. While that was nice, the card would definitely still go down from time to time, and then require the user to manually run the script. This was an improvement, but not very convenient, and I wanted a way to automate the process so I could simply forget about it and have it just work.

I took the script and modified it just a bit, and also added a few checks at the beginning of the script, which would check in various ways whether the the wifi card was working or not. (At first, the only checks I had were based upon `nmcli` and `ifconfig` but it seemed that were failures which these would not catch. I then added another check based on the output of `lshw`, because while debugging and suffering with this problem I had noticed different outputs of `lshw` depending upon whether the card was working or not; specifically, when the card was working I would see that "bus_master" was listed under the capabilities for the device, but that this would go missing when it had failed, or even was just starting to fail.) Anyway, once these checks were in place, once the script was run, the following would happen:

  (1) If the wifi was found to be WORKING, the the script would simply exit.
  (2) If the wifi was found to be NOT WORKING, then the script would continue and perform the wifi reset.
  
I then set up some cron jobs which would run my modified script every 20 seconds. Once I had this all set up, my wifi problems were over!

# How to set this up

The setup does take a few minutes and some preparation, but it it WELL worth it, and I will take you through step-by-step!

First, you need to have `ifconfig` installed on your system. I think it would be relatively easy to modify the script to use `ip` instead, or even to detect which of these were available on your system, but I have not implemented that yet. Anyway, as it is now, you want to make sure you have `ifconfig` installed so first just run:
`sudo apt install net-tools`
Now that you hve `ifconfig` installed, you can now proceed to download these two files into your home directory:

`https://raw.githubusercontent.com/John-Vision/Intel-7260-Wifi-Fix/master/fixwifi`

`https://raw.githubusercontent.com/John-Vision/Intel-7260-Wifi-Fix/master/fixwifi-force
`

To download them from within the terminal copy/paste/run the following lines

`cd ~`

`wget https://raw.githubusercontent.com/John-Vision/Intel-7260-Wifi-Fix/master/fixwifi`

`wget https://raw.githubusercontent.com/John-Vision/Intel-7260-Wifi-Fix/master/fixwifi-force`

Now that you have these two file in your home directory, you need to make them executable.

`chmod +x ~/fixwifi`

`chmod +x ~/fixwifi-force`

These two files are essentially the same, but with one difference: `fixwifi` first checks to see if you wifi is up and running; if it is then it just exits, but if not then it goes ahead and reset your wifi. 'fixwifi-force', on the other hand, does not bother to perform any check, and will reset your wifi whether it's already running or not.

Both of these files have some settings which you can change manually. Assuming that you have the Intel 7260 (which is what this is all about!) you shouldn't have to change anything, except *possibly* the line in each file (about line 19 in each) which says `interface="wlp3s0"`. Your interface may be different: typical values are things like, wlan0, wlp2s0, and the like. You can check your interface by executing `sudo lshw -C network | grep "logical name: w"`, as long as you run this while your wifi is working. So, if needed, just change the interface setting to whatever is appropriate for you, in each of these two files.

Once you have all this done, try `~/fixwifi-force`. If everything worked you should see your wifi get disconnected (if it was already connected) and then come back online. If this did not happen, then you need to check the output, and see if there are any errors. The most common (and easy to fix) error would be having the interface set wrong. (See paragraph above.) Another possibility is that you don't have an Intel 7260, in which case you would also have to change the part between the quotes in the setting for "wirelessPCI," and probably also the setting for "voodoo". (I have no idea of how to help you with the voodoo setting. This part is pretty much a mystery to me.)

Once you have `~/fixwifi-force` up and running, you are really in luck! Just make sure you have the same settings in `fixwifi` that worked for you in `fixwifi-force`. Now all you need to do is set up some cron jobs to run `fixwifi` periodically in the background, so you never have to think about it again!

If this is the first time you are using cron, the following makes sure it can run in the background:

`sudo systemctl enable cron`

Now it's time to go ahead and set up a crontab as root:

`sudo crontab -e`

It will ask what editor you want to use. Pick the one you want. (As the prompt will tell you, nano is the simplest.)

Now you need to add the following three lines, replacing the path with the actual path to your .fixwifi file. (Don't enter the path as a shortcut like "~/fixwifi" but actually go ahead and type out the full path.)

`* * * * * /path/to/.fixwifi`

`* * * * * sleep 20; /path/to/.fixwifi`

`* * * * * sleep 40; /path/to/.fixwifi`

When you have added these three lines, modified to reflect the actual path, save the file and you're done! (If you chose nano, press Ctrl-X to finish editing and then press "y" in response to "Save modified buffer?" and then just press "Enter" to accept the name of the file you want to send it to.)

That's it! Enjoy your new, stress free Intel 7260 Wifi!

(Please leave a comment - and maybe an upvote - to let me know how this worked for you: https://askubuntu.com/questions/1262557/cannot-get-intel-7260-to-work-properly-on-ubuntu-20-04-disconnects-intermittent)
