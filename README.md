# vioup
Ansible Playbook utilizing the VIO collections upgradevio module to upgrade a VIO from v3 to v4

Various notes and observations while working on this:


•	All you need from the VIO installation ISO, is the mksysb file located in /usr/sys/inst.images. You can use the viosupgrade -I command to extract it or just mount the iso and extract it from there. I did not try to use a customized mksysb image. Let me know if that’s worth testing. Not sure it would be a supported approach though.

•	I ran the playbook without a login user override, so it ran as root on the VIO servers, using ssh keys from my Ansible master.

•	Initially, to make things faster, I used the Ansible copy module to copy the mksysb to the VIO server. Even though I specified to store it in /home/padmin, during the transfer it temporarily used space in / since that’s where root’s home directory is located. Had to increase /’s size to make that work.

•	In later runs, I used a NFS mount which negated the increase of /. Not sure what method you would be using.

•	After the upgrade to v 4, there are password rules applied that didn’t exist in v3. So in essence, I was locked out of the padmin account with a 3004-327 Your password has been expired for too long error. So just be aware of the new password rules applied to v4.

•	I had issues with booting the VIO servers with a 554 service code, which pointed to possible issues with reserve policies and path algorithms, however I think that’s entirely due to the P9 system being directly connected to the FlashSystem without a SAN switch in the DSS lab. Just bringing this up in case you see it in testing too. 

•	With the mksysb copied to the VIO server, the upgrade took about 15-30 minutes and included 3.5 reboots.  

•	The module does have a handy parameter, the wait_reboot=true which doesn’t allow the playbook to continue until the upgrade has completed and the VIO server is fully functional. After the upgrade, I did not have any issues with any of the client LPAR’s (mostly Red Hat instances running an OpenShift environment).

•	After completion, the module does return some handy information, including versions before and after and the old rootvg disk name for any clean up calls.

•	Since this is basically a new-and-complete install of the VIO software, followed by a restore of the configuration, the use of the filename parameter to provide a file of a list of files to be copied over is a must. Things like /etc/motd, /etc/ntpd.conf, etc. I didn’t see an IBM recommended list of files to include, so hunted around for some recommendations. I can share if you want to see what I came up with.

•	The use of the filename parameter to save and restore certain files during the upgrade has a caveat. In my testing, if the file listed did not exist on the source v3, the upgrade would fail.  Hence in my playbook I actually parse the file, validated if a file exists, if it does, adds that to a new file and use that new file as the parameter for the viosupgrade module.

•	Since I was running the test numerous times, I would just change the bootlist on v4 to point back at the v3 disk and reboot. 

•	Once back on v3, the altinst_rootvg would be visible. Trying to use the ibm.power_aix.lvg module to remove it failed, since it showed in use. Even when running exportvg altinst_rootvg from the CLI before restarting the upgrade would cause an error, since the disk still showed in use. For the lspv -free (as padmin) command to show the disk available, I had to run chpv -C on the hdisk to show it as free and for the viosupgrade to allow me to use it. Hence, my playbook just ran these commands via the shell module. Identify the disk if any, validate it’s varied off, export the VG and run chpv -C on they hdisk. This might only be relevant for testing, but a necessity if re-running a failed upgrade for example.

•	After the upgrade, because of the underlying AIX upgrade, the python3 executable moved.  So, the playbook starts with ansible_python_interpreter=/opt/freeware/bin/python3 and after the upgrade ansible_python_interpreter=/usr/bin/python3. 


Disclaimer, I hardly call myself an Ansible expert, so not saying that my approaches are always the best, just where my testing took me. 

