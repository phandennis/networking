# activities
## activity 1

---

1. go to r1 and enable a new network adapter
    * select r1 (do not start it yet)
    * slece settings
    * go to network tab in setting
    * select an empty netowrk adapter
    * attach to internal network (name it grp1_64)
    * make sure cable is connected check box is checked
2. make new clones called test1 and test2 `(link clone not full clone)`
    * enable thier network adapters as well
    * generate new MAC addresses.
    * you can give them MAC addresses that are not in use on the network. It
      will make it easier for you to make the dhcpd.conf file later on r1. Forexample
      test1 `020000000090`
3. Start r1. You should be able to see the new network adapter in the nmtui menu.
Edit that menu to have a IPv4 range that will be given to you. 
    * for our activity, we will use `10.0.45.64/26`
    * this will give is a range of usable IP addresses from `10.0.45.65 - 10.0.45.126`
    * you can find this from a website or the script that he gave you. 
    * his advice is to assign the max address to the interface so that we can
      remember it easily.`(best practice to assing the max address to the router)`
    * in this case, we will use `10.0.45.126/26`
    * `sudo nmtui` to activate the connection.

4. Now we will go to the r1 dhcpd.conf file and edit it.
    * `sudo vim /etc/dhcp/dhcpd.conf`
    * add the following lines to the file
        ```
        subnet 10.0.45.64 netmask 255.255.255.192 {
            option routers 10.0.45.126;
            range 10.0.45.65 10.0.45.75;
            host test1 {
                    hardware ethernet 02:00:00:00:00:90;
                    fixed-address 10.0.45.126;
            }

            host test2 {
                    hardware ethernet 02:00:00:00:00:A0;
                    fixed-address 10.0.45.125;
            }
        }
        ```
    * after this you can run `sudo dhcpd -t` to test the file and make sure
      are no errors.

5. Now we will configure the bird.conf file on r1
    - `sudo vim /etc/bird.conf`
    - add the following lines to the file
        ```
        log syslog all;             # Log all messages

        router id 10.20.30.100;     # use your routers enp0s3 IP as its ID

        protocol device {           # the device "protocol" needs to be included to
                                    # activate all of the interfaces
        }

        protocol kernel {
            ipv4 {                  # export all routes learned by bird to the kernel
                export all;       # routing table
            };
        }

        protocol ospf {              # Activate OSPF
            area 0 {
                interface "enp0s3" { # Configure the enp0s3 connected network to be
                };                   # advertised to other routers. Also send and receive
                                    # link state advertisements on this interface

                interface "br0" {    # Configure the enp0s8 connected network to be
                    stub;            # to be advertised to other routers. Don't send or
                };                   # receive link stat advertisements on this interface
                                    # accomplished by the "stub" directive
                interface "enp0s10" {
                    stub;
                };
            };
        }
        ```
        * then you have to restart bird with `sudo systemctl restart bird`

6. after this you should be able to ping this new device from anywhere in the
   network. (after disabling the nftables firewall)
    - `sudo systemctl stop nftables`
    - `ping 10.0.45.125`


## activity 2

---

1. make a new router (clone it from the base image) and give it the name `grp1_rtr`.
2. you may give them custom MAC addresses if you want, but not required for
   this activity.
    * enable this adapter attach to `Host-only Adapter` and select the same network on as r1 and r2 on adapter 1.
    * enable another adapter on the internal network and call it `grp_128`.
3. make two clones and make them connected to the `internal network` name `grp_128` call your new cloned vm test3 and test4.
4. move on to the router`(grp1_rtr)`, all the work will be done on this router now.
5. `sudo nmtui` and edit enp0s3's ipv4 address to be `10.20.30.10/24`, and save
   this. 
6. on the second adapter on nmtui, we will give the address from the given range.
    - `10.0.45.190/26` 
7. Now you can ssh into this router. 
8. ssh and edit the dhcpd.conf file:
    - `sudo vim /etc/dhcp/dhcpd.conf`
    - add the following lines to the file
        ```
        subnet 10.0.45.128 netmask 255.255.255.192 {
            option routers 10.0.45.190;
            range 10.0.45.129 10.0.45.139;

            host test3 {
                    hardware ethernet 02:00:00:00:00:12;
                    fixed-address 10.0.45.189;
            }

            host test4 {
                    hardware ethernet 02:00:00:00:00:13;
                    fixed-address 10.0.45.188;
                    }
        }
        ```
    importent note: without fixed-address, your host will still connect. The dhcp server will ramdonly assign the address from the range you have given it. (in this case  10.45.129 to 10.45.139) to the host. To find the address, you can use the command "ip -a" on the host device.  
    `*host device here means test3 and test4`
    - then restart the service
    - then enable ipforwarding on the router
        - `sudo vim /etc/sysctl.conf`
        - add the following line to the file
            ```
            net.ipv4.ip_forward=1
            ```
        - then run `sudo sysctl --system` to apply the changes
9. Now we will configure  the bird.conf file (dynammic routing) on the new router. We can just
   copy it from r1 through scp:
   ```
   sudo scp admin@10.20.30.100;/etc/bird.conf /etc/bird.conf
   ```
    - then change the file to have only the "enp0s3" and "enp0s8" interfaces.
        ```
        log syslog all;             # Log all messages

        router id 10.20.30.10;     # use your routers enp0s3 IP as its ID

        protocol device {           # the device "protocol" needs to be included to
                                    # activate all of the interfaces
        }

        protocol kernel {
            ipv4 {                  # export all routes learned by bird to the kernel
                export all;       # routing table
            };
        }

        protocol ospf {              # Activate OSPF
            area 0 {
                interface "enp0s3" { # Configure the enp0s3 connected network to be
                };                   # advertised to other routers. Also send and receive
                                    # link state advertisements on this interface
                # receive link stat advertisements on this interface
                                    # accomplished by the "stub" directive
                interface "enp0s8" {
                    stub;
                };
            };
        }
        ```

    - check for errors `sudo bird -p`
    - restart the service `sudo systemctl restart bird` and enable it
      `sudo systemctl enable bird`.
