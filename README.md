# Intel-7260-Wifi-Fix
A fix for Intel 7260 WIFI PCI cards, which intermittently and unpredictably stop working on Linux.
(With a little bit of know how, this might easily be adapted to support other chipsets.)

The Intel 7260 WIFI PCI cards have fantasitic wifi capabilities, but are notorious for intermittently and unpredictably shutting down and becoming completely non-responsive, with no way of restarting the card except for rebooting the system.

After a LOT of searching I found a couple of scripts which could be run that would restart the card. While that was nice, the card would definitely still go down from time to time, and then require the user to manually run the script. This was an improvement, but not very convenient, and I wanted a way to automate the process so I could simply forget about it and have it just work.

I took the script and modified it just a bit, and also added a few checks at the beginning of the script, which would check in various ways whether the the wifi card was working or not. (At first, the only checks I had were based upon `nmcli` and `ifconfig` but it seemed that were failures which these would not catch. I then added another check based on the output of `lshw`, because while debugging and suffering with this problem I had noticed different outputs of `lshw` depending upon whether the card was working or not; specifically, when the card was working I would see that "bus_master" was listed under the capabilities for the device, but that this would go missing when it had failed, or even was just starting to fail.) Anyway, once these checks were in place, once the script was run, the following would happen:

  (1) If the wifi was found to be WORKING, the the script would simply exit.
  (2) If the wifi was found to be NOT WORKING, then the script would continue and perform the wifi reset.
  
I then set up some cron jobs which would run my modified script every 20 seconds. Once I had this all set up, my wifi problems were over!

# How to set this up

First, you need to have `ifconfig` installed on your system. I think it would be relatively easy to modify the script to use `ip` instead, or even to detect which of these were available on your system, but I have not implemented that yet. Anyway, as it is now, you want to make sure you have `ifconfig` installed so first just run:
`sudo apt install net-tools`
Then create a file called `.fixwifi` in your home directory. (I prepended this file with a dot so that it would act as a hidden file.)
`touch ~/.fixwifi`
If you want to make sure that worked, run the following to list the files in your home directory. You should see the newly created `.fixwifi` listed last.
`ls -t -r -a`
Now you want to make the file executable:
`chmod +x ~/.fixwifi`
Now you have to add the contents to this file. I prefer to use the nano editor, so for me the command would be:
`sudo nano ~/.fixwifi`
Then you have to copy/paste the follwing into your empty .fixwifi file:

`#!/bin/bash

#############################################################################################################
#                                                                                                           #
# This script requires "ifconfig" to be installed. If not installed, then do so now!                        #
# sudo apt install net-tools                                                                                #
# The interface name is set below as "wlp3s0". If yours is different seach through and replace with yours.  #
#                                                                                                           #
#############################################################################################################

# You need to know the designations for your wifi interface.
# You can find these out by running: sudo lshw -C network
# Look through the output, and replace the information between the quotes, as necessary, in the following settings.

# product
wirelessPCI=$(lspci |grep "Wireless 7260")

# logical name
interface="wlp3s0"

# Intel voodoo. The setting below is known to work with the Wireless 7260. If we knew what this value should be
# for other Intel chipsets, this script should work for them as well. Maybe it's the same for multiple chipsets?
voodoo="0x50.B=0x40"

# Don't change anything below this line.
###########################################################################################################
# If this script works, then do "sudo crontab -e" and # add the following, without the initial hash (#) in each line. 
#* * * * * /home/kflynn/.fixwifi
#* * * * * sleep 20; /home/kflynn/.fixwifi
#* * * * * sleep 40; /home/kflynn/.fixwifi
###########################################################################################################

#---------------------------------------------------------------------------------------
# Assume wifi okay at first. If there are any problems during checks, this gets changed.
wifiOK=true

# This will refresh the list of networks in NetworkManager. Comment out if not needed. 
nmcli device wifi list

# Check if wifi is okay using: sudo lshw -C network | grep "pciexpress"
# if "bus_master" is not in the output, this indicates a problem
capabilities=$(sudo lshw -C network | grep "pciexpress")
    if [[ $capabilities == *"bus_master"* ]]; then
        echo "OKAY - bus_master found in: $capabilities"
    else
        wifiOK=false
        echo "OOPS - bus_master missing in: $capabilities"
    fi

# Check if wifi is okay using: nmcli networking connectivity 
# "unknown" and "none" indicate a problem.
connectivity=$(nmcli networking connectivity)
echo "CONNECTIVITY: $connectivity"
    if [ $connectivity = "unknown" ]; then
        wifiOK=false
        echo "OOPS - nmcli networking connectivity: unknown"
    fi
    if [ $connectivity = "none" ]; then
        wifiOK=false
        echo "OOPS - nmcli networking connectivity: none"
    fi
    
# Check if wifi is okay using: ifconfig $interface up
# anything other than "0" indicates a problem.
sudo ifconfig $interface up
initialExitCode=$?
    if [ $initialExitCode -eq 0 ]; then
        echo "OKAY - iconfig $interface up (should be 0): $initialExitCode"
    else
        wifiOK=false
        echo "OOPS - iconfig $interface up (should be 0): $initialExitCode"
    fi

# If wifi is okay, then say so and return; otherwise the script will continue.
if $wifiOK;then
    echo "WIFI OKAY, RETURNING"
    exit 1
fi
#---------------------------------------------------------------------------------------

# If we got to this point then we have detected a problem with wifi (wifiOK=false).
# The rest of this script will get it back up and running!
    
# Figure out what pci slot Linux has assigned the Network controller: Intel Corporation Wireless 7260
pci=$(echo ${wirelessPCI} | awk '{ print $1 }')
devicePath="/sys/bus/pci/devices/0000:$pci/remove"

# Not the best solution as this script can hang. 
# But since if this script fails the ONLY way to revive the wifi anyway is a reboot...
# Feel free to improve the script if you have the scriptfu ninja skills to do so.
while true; do

    # Tell Linux to remove the wifi card from the PCI device list only if it exists in the first place.
    if [ -f $devicePath ]; then
        echo '----removing device'
        echo 1 | sudo tee $devicePath > /dev/null
        sleep 1
    fi

    # Reprobe the driver modules in case we have removed them in a failed attempt to wake the network card.
    echo '----reprobing drivers'
    sudo modprobe iwlmvm
    sudo modprobe iwlwifi
    
    # Try to have Linux bring the network card back online as a PCI device. 
    echo '----pci rescan'
    echo 1 | sudo tee /sys/bus/pci/rescan > /dev/null
    sleep 1

    # Check if Linux managed to bring the network card back online as a PCI device.
    if [ -f $devicePath ]; then
        echo '----device is back'

        # Looks like we are back in business. 
        # So we try to set the PCI slot with some voodoo I don't understand that the Intel devs told me to try.
        # https://bugzilla.kernel.org/show_bug.cgi?id=191601
        sudo setpci -s $pci $voodoo

        sleep 1
        wifiId=$(rfkill list |grep Wireless |awk -F: '{ print $1 }')
        echo "----rfkill unblock wireless device: $wifiId"
        sudo rfkill unblock $wifiId

        sleep 1
        # Bring the wireless network interface up.
        sudo ifconfig $interface up

        # Did the wifi interface actually go live?
        exitCode=$?
        echo "----device UP status $exitCode"
        if [ $exitCode -eq 0 ];then

            # This should be the default for wireless devices as it is well documented that enabling power management causes problems.
            sudo iwconfig $interface power off

            # The exit code will be the exit code of our attempt at turning power management off for the interface.
            break
        fi
    else
        # The restart attempt failed, so we need to remove the the wifi driver modules and loop back in another attempt to revive the wifi.
        echo "----WIFI RESTART FAILED - ATTEMPTING AGAIN"
        sudo modprobe -r iwlmvm
        sudo modprobe -r iwlwifi
    fi
done

echo "DONE - WIFI SHOULD RESTART NOW."`

Lines 45, 48, and 52, for "product" "logical name" and "Intel voodoo," must reflect your own setup. The settings I have work for me. Here's how you can find your settings: 
`sudo lshw -C network`
In the listing for your wireless network, put whatever it says for "product" between the quotes in line 45.
Put whatever it says for "logical name" between the quotes in line 48.
For line 52, I have no idea of how to find your setting. I just know that this is the setting which works for the Intel 7260.

You can then run the file, just to see that it works. Note that if you run it when your wifi is working, it will give a few messages, the last of which will be, "WIFI OKAY - RETURNING." If you run it when your wifi is NOT working, you will also see a number of messages, the last of which will be "DONE - WIFI SHOULD RESTART NOW." And then, wonderfully, your wifi should just restart! To run the file:
`~/.fixwifi`
You now have a very easy way to restart your wifi, without needing to restart your computer!

Once all the above is working, you'll want to set up a cron job to run this in the background, so you never have to think about it again!
If this is the first time you are using cron, the following makes sure it can run in the background:

`sudo systemctl enable cron`

Now it's time to go ahead and set up a crontab as root:

`sudo crontab -e`

It will ask what editor you want to use. Pick the one you want. (As the prompt will tell you, nano is the simplest.)

Now you need to add the following three lines, replacing the path with the actual path to your .fixwifi file:

`* * * * * /path/to/.fixwifi`
`* * * * * sleep 20; /path/to/.fixwifi`
`* * * * * sleep 40; /path/to/.fixwifi`

When you have added these three lines, modified to reflect the actual path, save the file and you're done! (If you chose nano, press Ctrl-X to finish editing and then press "y" in response to "Save modified buffer?" and then just press "Enter" to accept the name of the file you want to send it to.)

That's it!
