# Install Pi OS

Install Raspbian Lite (headless), using Raspberry Pi Imager, selecting option to access SSH from getgo

Update everything

```
sudo apt update
sudo apt upgrade -y
```

To access SSH via VS Code, you may want to delete old entries (same ip) editing `~/.ssh/known_hosts`


# NUT UPS install & config

## NUT Server

1. Check USB port

    ```
    $lsusb

    Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 001 Device 003: ID 0463:ffff MGE UPS Systems UPS
    Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

    ```

2. Install


    ```
    sudo apt install nut
    ```
    This will install packages `nut-client` and `nut-server`


3. Scan devices

    ```
    $sudo nut-scanner -U

    Scanning USB bus.
    [nutdev1]
      driver = "usbhid-ups"
      port = "auto"
      vendorid = "0463"
      productid = "FFFF"
      product = "Eaton 5P"
      serial = "G113R09093"
      vendor = "EATON"
      bus = "001"
    ```

4. Save old config, link to convenient dir

    ```
    sudo cp /etc/nut/ups.conf /etc/nut/ups.example.conf  

    sudo ln -sv /etc/nut/nut.conf /home/akira/docker/nut/nut.conf
    sudo ln -sv /etc/nut/ups.conf /home/akira/docker/nut/ups.conf
    sudo ln -sv /etc/nut/upsd.conf /home/akira/docker/nut/upsd.conf
    sudo ln -sv /etc/nut/upsd.users /home/akira/docker/nut/upsd.users
    sudo ln -sv /etc/nut/upsmon.conf /home/akira/docker/nut/upsmon.conf
    sudo ln -sv /etc/nut/upssched.conf /home/akira/docker/nut/upssched.conf

    sudo chmod 660 /etc/nut/*
    ```

5. Add user to nut group

    ```
    sudo usermod -a -G nut $USER
    ```

    Logout & Login so user update groups (check with `groups`)


6. Edit `ups.conf` with scanned datas

    ```
    pollinterval = 3

    # Set maxretry to 3 by default, this should mitigate race with slow devices:
    maxretry = 3

    [eaton5p1550i]
        driver = usbhid-ups
        port = auto
        desc = "Eaton 5P1550I"
        vendorid = 0463
        productid = ffff
        serial = G113R09093


    # Dummy UPS as qnap require this name
    [qnapups]
      driver = dummy-ups
      port = eaton5p1550i@localhost
      #mode = dummy-loop
      desc = "Eaton UPS for QNAP"
    ```

7. Save &  link to convenient dir (refer to 4)

8. Edit `upsmon.conf`

    ```
    RUN_AS_USER root
    
    MONITOR eaton5p1550i@localhost 1 ups_admin p@ssw0rd primary

    # Use upssched to trigger scripts
    NOTIFYCMD /usr/sbin/upssched

    SHUTDOWNCMD "/sbin/shutdown -h now"
    FINALDELAY 10 #delay prior to shutdown this server after notify shutdown
    # File flag that will be created if system goes down
    POWERDOWNFLAG /etc/killpower 

    # What will trigger the scripts
    NOTIFYFLAG ONLINE SYSLOG+EXEC+WALL
    NOTIFYFLAG ONBATT SYSLOG+EXEC+WALL
    NOTIFYFLAG LOWBATT SYSLOG+EXEC+WALL
    NOTIFYFLAG FSD SYSLOG+EXEC+WALL
    NOTIFYFLAG COMMOK SYSLOG+EXEC
    NOTIFYFLAG COMMBAD SYSLOG+EXEC
    NOTIFYFLAG NOCOMM SYSLOG+EXEC
    NOTIFYFLAG SHUTDOWN SYSLOG+EXEC+WALL
    NOTIFYFLAG REPLBATT SYSLOG+EXE

    # No Comm Warn Time
    NOCOMMWARNTIME 640
    ```

9. Edit `upsd.conf`

    ```
    # Network UPS Tools: Listen to all addresses
    #
    LISTEN 0.0.0.0 3493
    LISTEN :: 3493

    # If no Client (ie server is the client) only local:
    # LISTEN 127.0.0.1 3493 

    ```

10. Edit `nut.conf`

    ```
    MODE=netserver
    ```

11. Edit `upsd.users`

    ```
    [ups_admin]
      password = p@ssw0rd
      #Allow changing values of certain variables in the UPS
      actions = SET
      #Allow setting the "Forced Shutdown" flag in the UPS
      actions = FSD
      # Allow all instant commands
      instcmds = ALL
      upsmon primary 

    [ups_user]
      password = theUPSpwd
      upsmon secondary

    #User for qnap
    [admin]
      password = 123456
      instcmds = ALL
      upsmon slaveups
    ```

12. reboot or 

    ```
    sudo service nut-client restart
    sudo systemctl restart nut-monitor
    sudo upsdrvctl stop
    sudo upsdrvctl start
    ```

    Enable at boot

    ```
    sudo systemctl enable nut-client
    sudo systemctl enable nut-monitor

    ```


    try

    ```
    upsc eaton5p1550i@localhost
    ```
    
    run driver in debug mode
    
    ```
    /usr/lib/nut/usbhid-ups -a {{UPS_NAME}} -DDD -d1
    
    or
    
    upsdrvctl -DD -d start eaton5p1550i
    ```

    Change UPS variable

    ```
    upsrw eaton5p1550i


    upsrw -s outlet.1.delay.shutdown=180 eaton5p1550i
    upsrw -s outlet.1.autoswitch.charge.low=20 eaton5p1550i
    upsrw -s outlet.2.autoswitch.charge.low=20 eaton5p1550i
    upsrw -s battery.charge.restart=25 eaton5p1550i
    ```

    UPS commands

    ```
    upscmd -l eaton5p1550i

    upscmd eaton5p1550i beeper.disable
    ```


## NUT Client

1. for new client install via (for the server its already done)

    ```
    sudo apt install nut-client
    ```

2. Verify connection to server: 

    ```
    upsc eaton5p1550i@pihole.local
    ```

3. edit `upsmon.conf`

    ```
    RUN_AS_USER root
    
    MONITOR eaton5p1550i@localhost 1 ups_admin p@ssw0rd primary

    # Use upssched to trigger scripts
    NOTIFYCMD /usr/sbin/upssched

    SHUTDOWNCMD "/sbin/shutdown -h now"
    FINALDELAY 10 #delay prior to shutdown this server after notify shutdown
    # File flag that will be created if system goes down
    POWERDOWNFLAG /etc/killpower 

    # What will trigger the scripts
    NOTIFYFLAG ONLINE SYSLOG+EXEC+WALL
    NOTIFYFLAG ONBATT SYSLOG+EXEC+WALL
    NOTIFYFLAG LOWBATT SYSLOG+EXEC+WALL
    NOTIFYFLAG FSD SYSLOG+EXEC+WALL
    NOTIFYFLAG COMMOK SYSLOG+EXEC
    NOTIFYFLAG COMMBAD SYSLOG+EXEC
    NOTIFYFLAG NOCOMM SYSLOG+EXEC
    NOTIFYFLAG SHUTDOWN SYSLOG+EXEC+WALL
    NOTIFYFLAG REPLBATT SYSLOG+EXE

    # No Comm Warn Time
    NOCOMMWARNTIME 640
    ```

4. edit `nut.conf`

    ```
    MODE=netclient
    ```

5. Check logs via

    ```
    journalctl -f
    ```

    disable driver if required

    ```
    systemctl disable --now nut-driver@ups
    ```


6. **Configuration**, the global scheme is:

  - Notify flags declared in  `upsmon.conf` 
  - `NOTIFYCMD /usr/sbin/upssched` declared in `upsmon.conf` to work with upssched
  - `upssched.conf` that capture the event and link to macro name (ex : `onbatt`)
  - `CMDSCRIPT /home/akira/docker/nut/ups-cmd` in `upssched.conf` is the script in where action will be implemented
    using the argument to determine which macro will be triggered

7. Edit `upssched.conf` would be something like this:

    `*` stand for all UPS connected to the server, could be `eaton5p1550i@localhost`

    ```
    CMDSCRIPT /etc/nut/upssched-cmd
    PIPEFN /run/nut/upssched.pipe
    LOCKFN /run/nut/upssched.lock


    # The UPS is back on line
    # Cancel any running "On Battery" timer, then execute "Online" cmd
    AT ONLINE * EXECUTE online

    # The UPS is on battery 
    # Start a 120s timer then execute the "onbatt" command
    #AT ONBATT eatonups@localhost START-TIMER onbatt 120
    AT ONBATT * EXECUTE onbatt


    # If the power is out for more than 5min , shutown the UPS
    # ignore the LB condition
    #AT ONBATT eaton_ups@localhost START-TIMER shutafter5min 300
    #AT ONLINE * CANCEL-TIMER shutafter5min
    # The UPS battery is low (determined by driver)

    # Execute the "Low Battery" cmd immediately
    AT LOWBATT * EXECUTE lowbatt

    # The UPS has been commanded into the "Forced Shutdown" mode
    # Execute the "Forced Shutdown" cmd immediately
    AT FSD * EXECUTE fsd

    # Communication with the UPS has been established
    # Cancel any running "Communications Lost" timer, 
    # then execute the "Communications Restored" cmd
    AT COMMOK * CANCEL-TIMER commbad commok

    # Communication with the UPS was just lost
    # Start a 30s timer, then execute the "Communications Lost" cmd
    AT COMMBAD * START-TIMER commbad 30

    # The UPS can't be contacted for monitoring
    # Start a 15s timer, then execute the "No Communications" cmd
    AT NOCOMM * START-TIMER nocomm 15
    AT NOCOMM * EXECUTE commbad

    # The local system is being shut down
    # Execute the "Notify Shutdown" cmd immediately
    AT SHUTDOWN * EXECUTE shutdown

    # The UPS needs to have its battery replaced
    # Start a 5min timer, then execute the "Replace Battery" comd
    AT REPLBATT * START-TIMER replbatt 300
    ```

8. Edit the script `upssched-cmd` **(which shall be executable)**

    ```
   #!/bin/sh

    # --- Config --------------


    # target hostname or ip of device that will be checked up
    # if target is down on 2nd check after $retry, power cycle UPS
    target=192.168.40.41
    #target=truenas.local

    retry=4m #4min to reboot the server
    minCharge=90 # min UPS %charge to power on device

    ups=eaton5p1550i@localhost
    powerOffCmd=outlet.1.load.off
    powerOnCmd=outlet.1.load.on
    admin=ups_admin
    pwd=p@ssw0rd

    fname_line1=/tmp/oled-display/line1 
    fname_line2=/tmp/oled-display/line2 

    # --- Config --------------
    #mkdir -p /tmp/oled-display #done in the python file


    case $1 in
      onbatt)
        logger -t upssched-cmd "UPS running on battery"
        echo "UPS on batt" > $fname_line1
        echo "timer" > $fname_line2
        ;;
      online)
        logger -t upssched-cmd "Power restored on UPS"
        echo "UPS on line" >  $fname_line1
        # test if target device is alive if not power cycle
        #if ping -c 1 -W 1 "$target"; then
        if curl -I "http://$target"; then
          logger -t upssched-cmd "$target alive"
          echo "$target alive" >  $fname_line2
        else
          logger -t upssched-cmd "$target offline, retrying in $retry"
          echo "Retry in $retry s" >  $fname_line1
          echo "timer" >  $fname_line2
          sleep $retry
          if curl -I "http://$target"; then
            logger -t upssched-cmd "$target is alive"
            echo "" >  $fname_line1
            echo "$target alive" >  $fname_line2
          else
            # $target offline for more than $retry time,
            # wait for ups battery enough to power cycle
            logger -t upssched-cmd "$target offline checkin UPS battery charge"
            echo "Server offline" >  $fname_line1
            echo "checking charge" >  $fname_line1
            while :; do
              clear;
              charge=$(upsc eaton5p1550i@localhost battery.charge > /dev/stdout 2> /dev/null)
              if [ $charge -ge $minCharge ]; then
                # $target offline and UPS charge enough, 
                # power cycle to wake up devices   
                logger -t upssched-cmd "$target offline UPS battery is $charge waking up devices"   
                echo " $charge % => Ok" >  $fname_line1
                echo "Power cycle 1" >  $fname_line2

                upscmd -u $admin -p $pwd $ups $powerOffCmd
                sleep 5s
                upscmd -u $admin -p $pwd $ups $powerOnCmd

                echo "Power cyle done" >  $fname_line1
                echo "" >  $fname_line2
                break
              fi
              logger -t upssched-cmd "$charge < $minCharge retring in 1 min"
              echo "Waiting $minCharge %" >  $fname_line1
              echo "Actual $charge %" >  $fname_line2
              sleep 1m; # check if battery is enough evry min
            done
            
          fi
        fi    
        ;;
      lowbatt)
        logger -t upssched-cmd "Low battery on UPS!"
        echo "UPS low batt" > $fname_line1
        ;;
      shutafter5min)
        logger -t upssched-cmd "Shutdown after 5 minutes"
        #/usr/bin/upsmon -c fsd
        ;;
      fsd)
        logger -t upssched-cmd "Forced Shutdown from UPS!"
        echo "UPS forced shutdown" > $fname_line1
        ;;
      commok)
        logger -t upssched-cmd "Communications restored with UPS"
        echo "UPS Com Ok" > $fname_line1
        echo "" > $fname_line2
        ;;
      commbad)
        logger -t upssched-cmd "Warning: Lost communications with UPS"
        echo "UPS Com lost" > $fname_line1
        echo "timer" > $fname_line2
        ;;
      shutdown)
        logger -t upssched-cmd "System is shutting down now!"
        echo "UPS is shutting down now" > $fname_line1
        ;;
      replbatt)
        logger -t upssched-cmd "Replace battery on UPS"
        echo "UPS replace battery" > $fname_line1
        echo "timer" > $fname_line2
        ;;
      nocomm)
        logger -t upssched-cmd "The UPS can’t be contacted for monitoring!"
        echo "UPS no Com" > $fname_line1
        ;;
      *)
        logger -t upssched-cmd "Unknown event: $1"
        echo "UPS unknown event" > $fname_line1
        ;;
    esac
    ```

10. Configure RAMDISK

**Warning** python script is creating the directories in `/tmp` at boot

```
sudo nano /etc/fstab
```

Add these lines

```
# RAM disk tmp and log in RAM disk
tmpfs /tmp  tmpfs defaults,noatime 0 0
tmpfs /var/log  tmpfs defaults,noatime,size=16m 0 0
```

9. List available commands for UPS and driver:

    ```
    upscmd -l eaton5p1550i@localhost
    upscmd -u <username> -p <password> <system> <command>
    upscmd -u $admin -p $adminpass eaton5p1550i@localhost outlet.1.load.off
    ```






# Installation DOCKER

1. Install docker with command:

    ```
    curl -fsSL https://get.docker.com | sh
    ```

2. Grant user access to docker to avoid sudo

    ```
    sudo usermod -aG docker $USER
    ```

    Logout and login to update user permissions

4. Check if ps work without sudo

    ```
    docker ps
    ```




# OLED display

<https://wiki.52pi.com/index.php?title=DR-0001#How_to_enable_I2C_OLED_display?>

## Enable I2C to access OLED display

Enable I2C by editing the `/boot/firmware/config.txt` file and adding `dtparam=i2c_arm=on` if it's not already present, After making changes, reboot your Pi.


