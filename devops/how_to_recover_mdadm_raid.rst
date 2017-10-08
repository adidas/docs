How-To: Recovering a mdadm RAID array with a failed drive
=========================================================

Created: 2017-10-08

Hard drives can be fickel things. I needed to upgrade my server’s OS from Ubuntu 12.10 to 16.04, but I forgot what
drives it had. I wasn’t sure if I had a separate drive for the OS or not, and I am very green when it comes to this type
of linux work. To aid in investigating this I opened up the case and looked. Yes, I had three. When I plugged in
everything I found one of the drives died during my intrusion.

Obviously I was nervous. I was running only two drives in my raid array. They were purchased at the same time and used
for the exact same purposes. There was a good chance that if one died the other could be next soon.

**When faced with this type of drive failure your first step needs to be a backup immediately.**

You may also want to take note of various system config files such as
* /etc/samba/smb.config
* /etc/network/interfaces
* /etc/fstab

I ordered an external drive immediately and left the machine untouched until it could be backed up. I was feeling much
better once I knew the data was on another drive. Now I had to address the degraded RAID array issue. The machine was
failing to boot, but this was safety feature of the OS. Ubuntu was reporting “the system may have suffered a hardware
fault, such as a disk drive failure…”.  I was also unable to get the system to get past this by instructing it to start
with the degraded RAID. I think the OS at this level was not detecting my USB keyboard. However, I was able to get past
this by restarting, pressing “e” at the grub menu to edit the boot commands, adding “bootdegraded=true” to the end of
the line that started with “linux”, and pressing F10 to boot the machine.

Once I was in the machine I was able to query the raid array a few ways. One was to use mdadm

.. code:: bash

    sudo mdadm --detail /dev/md0

The other was

.. code:: bash

    cat /proc/mdstat

The first showed me which drive was degraded, but the second gave me a quick way to see what was going on.

However, I needed to determine which physical drive was associated with /dev/sda. As a quick reminder the drives are
assigned in the order they are detected. This will mean the PC will scan the SATA ports in order and the OS will assign
them in that order. It’s then obvious that sda is the hard drive connected to the lowest number SATA connector. To be on
the safe side I disconnected the drive with the issue to confirm that everything was at it seemed.

Once I was sure I knew what was what I started getting the server back up. I replaced both the system (optional) and
failed drive with new drives. **Make sure you replace the failed RAID drive with one of equal or larger size. Otherwise
the rebuild will not work.** I installed Ubuntu 16.04 as this was my original goal. To get the raid array working, all I
had to do was run the following command to copy the still-good RAID drive’s partition to the the new drive

.. code:: bash

    sudo sfdisk -d /dev/sda | sudo sfdisk --force /dev/sdb.

Note that my source was sda. I plugged in the new drive in a SATA socket that numbered higher than the old drive.
Eitherway, this process was quick. The final step was to add the drive to the array. This was done with

.. code:: bash

    sudo mdadm --manage /dev/md127 --add /dev/sdb1

This part took the longest, about 2 hours. mdadm automatically started to rebuild the array once the drive was added.
``cat /prod/mdstat`` will allow you to monitor the process. You also may find that the RAID array might change where it’s
located. Originally, it was at md0 on the old OS, but then moved to md127 on 16.04, but ended up back on md0 in the end.

In conclusion, I found the whole process to be stressful as I was dealing with data I didn’t want to lose, but in
hindsight it was very easy. The hardest part was relearning what I did 4 years ago to set up the array in the first
place (documentation!, documentation!, documentation!) Hopefully, this can help someone else who finds him/herself in
the same predicament. 

