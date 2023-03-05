# IoT-Microcontroller-to-the-Cloud-
Transfer remote sensor data to AWS Cloud using Linux and RIOT OS
# Send data to AWS by the MQTT protocol using [RIOT-OS](https://www.riot-os.org/)
Send the data by the MQTT protocol and its constrained variant, called MQTT-SN, using RIOT OS on three A8-M3 nodes in FIT/IoT-LAB.

## Abstract

MQTT (MQ Telemetry Transport) is an IoT connectivity protocol designed as an extremely lightweight publish/subscribe protocol to transport messages between devices.

Using RIOT OS on three A8-M3 nodes in FIT/IoT-LAB, we will develop the MQTT protocol and its constrained variant, MQTT-SN. The MQTT-SN client running on a RIOT OS node will display MQTT messages published to the 'test/riot' topic from the Grenoble SSH frontend host. Into the FIT/IoT-LAB we will create 3 nodes, 

* The first node will be used as a border router for propagating an IPv6 prefix.
* The second node will be used as an MQTT broker
* The third node will run an MQTT-SN client to connect with the broker.

After successully created MQTT brocker, we will create a instance into the AWS Cloud and here we will take the AWS EC2 service. In AWS we will create a brocker then we will make a bridge between two brocker.  



<p align="center">
  <img src="https://github.com/PinkuChanda/Send-data-to-AWS-by-the-MQTT-protocol-using-RIOT-OS/blob/main/assets/Send-data-to-AWS-by-the-MQTT-protocol-using-RIOT-OS.jpg">
</p>



## Local Implementation using RIOT-OS

1. Connect to the SSH frontend of the FIT/IoT-LAB Grenoble site with the 'username' you created when we registered for the testbed:

    ```
	my_computer:~$ ssh <username>@grenoble.iot-lab.info
	```
2. Authenticate with the testbed and submit whatever interval experiment we require using three A8-M3 type nodes before waiting for the experiment to begin:

    ```
	username@grenoble:~$ iotlab-auth -u <username>
	username@grenoble:~$ iotlab-experiment submit -n riot_mqtt -d 60 -l 3,archi=a8:at86rf231+site=grenoble
	username@grenoble:~$ iotlab-experiment wait
	```
    Make a note of the displayed experiment ID, then use the commands below to retrieve additional information about the experiment once the experiment is running:

    ```
	username@grenoble:~$ iotlab-experiment get -i <experiment_ID> -s
	username@grenoble:~$ iotlab-experiment get -i <experiment_ID> -r
	```

3. The first node (a8-1) will act as a border router by utilizing the gnrc border router. 

    * Clone the RIOT OS from GitHub:
        ```
		username@grenoble:~$ git clone https://github.com/RIOT-OS/RIOT.git
		```
    * Build the `gnrc_border_router` A8-M3 nodes, which is 500000:  
        ```
		username@grenoble:~$ source /opt/riot.source
		username@grenoble:~$ cd RIOT
		username@grenoble:~/RIOT/$ make ETHOS_BAUDRATE=500000 BOARD=iotlab-a8-m3 clean all -C examples/gnrc_border_router
		```  
    *  Copy and flash the compiled firmware to the first experiment node by running the following commands:    
        ```
		username@grenoble:~/RIOT/$ scp examples/gnrc_border_router/bin/iotlab-a8-m3/gnrc_border_router.elf root@node-a8-1:
		username@grenoble:~/RIOT/$ ssh root@node-a8-1
		root@node-a8-1:~# flash_a8_m3 gnrc_border_router.elf
		```

4. The we will configure the network settings of the border router.  

    * In the first node to compile too named `uhcpd`
        ```
		root@node-a8-1:~# cd ~/A8/riot/RIOT/dist/tools/uhcpd
		root@node-a8-1:~/A8/riot/RIOT/dist/tools/uhcpd# make clean all
		```

    * Compile the ethos tool as well and configuire it on the node's public IPv6 network:

        ```
		root@node-a8-1:~/A8/riot/RIOT/dist/tools/uhcpd# cd ../ethos
		root@node-a8-1:~/A8/riot/RIOT/dist/tools/ethos# make clean all
		```

    * Then we will run `printenv` for getting prefix directly on the A8 node.

        ```
		root@node-a8-1:~# printenv
		```
        
    * On the border router, the network can finally be configured automatically using the following commands:

        ```
		root@node-a8-1:~/A8/riot/RIOT/dist/tools/ethos# ./start_network.sh /dev/ttyA8_M3 tap0 IPv6_prifix 500000
		```

    * Then we will get output similar to that shown below will be displayed.

        ```
		net.ipv6.conf.tap0.forwarding = 1
		net.ipv6.conf.tap0.accept_ra = 0
		----> ethos: sending hello.
		----> ethos: activating serial pass through.
		----> ethos: hello reply received
		```

5. Connect to the second node ('a8-2') in another terminal to configure the MQTT broker:

    * Connect to the Grenoble SSH frontend before proceeding to the second node.

        ```
		my_computer:~$ ssh <username>@grenoble.iot-lab.info
		username@grenoble:~$ ssh root@node-a8-2
		```
    * Edit the configuration file `config.conf` by using the `vim ` editor as following our `config.cnf` file. This configuration will make the MQTT broker accessible via MQTT-SN on port 1885 from any node behind the border router, and via MQTT on port 1886 from the SSH frontend.

    * Get this node's global IPv6 address, as it will be required later to connect to it in step 6.

        ```
		root@node-a8-2:~# ip -6 -o addr show eth0
		2: eth0    inet6 2001:660:3207:400::66/64 scope global        valid_lft forever preferred_lft forever
		2: eth0    inet6 fe80::fadc:7aff:fe01:98fc/64 scope link      valid_lft forever preferred_lft forever
		```

    * Then we will start the MQTT broker on the second node.
        ```
		root@node-a8-2:~# broker_mqtts config.cnf
		```

        The output will be similler to that shown bellow

        ```
        20230126 153509.370 CWNAN9999I Really Small Message Broker
        20230126 153509.374 CWNAN9998I Part of Project Mosquitto in Eclipse
        (http://projects.eclipse.org/projects/technology.mosquitto)
        20230126 153509.376 CWNAN0049I Configuration file name is config.cnf
        20230126 153509.384 CWNAN0053I Version 1.3.0.2, Dec 20 2020 23:34:33
        20230126 153509.385 CWNAN0054I Features included: bridge MQTTS 
        20230126 153509.386 CWNAN9993I Authors: Ian Craggs (icraggs@uk.ibm.com), Nicholas O'Leary
        20230126 153509.390 CWNAN0300I MQTT-S protocol starting, listening on port 1885
        20230126 153509.392 CWNAN0014I MQTT protocol starting, listening on port 1886
        ```

6. Open a third terminal to prepare the third node (`a8-3`) as an MQTT-SN client:

    *  Log in to the Grenoble SSH frontend, build and flash the RIOT MQTT-SN

        ```
		your_computer$ ssh <username>@grenoble.iot-lab.info
		username@grenoble:~$ source /opt/riot.source
		username@grenoble:~$ cd RIOT
		username@grenoble:~/RIOT$ make BOARD=iotlab-a8-m3 -C examples/emcute_mqttsn
		username@grenoble:~/RIOT/$ scp examples/emcute_mqttsn/bin/iotlab-a8-m3/emcute_mqttsn.elf root@node-a8-3:
		username@grenoble:~/RIOT/$ ssh root@node-a8-3
		root@node-a8-3:~# flash_a8_m3 emcute_mqttsn.elf
		```

    * To access the shell interface of the third node, run the program `miniterm.py`.
        ```
		root@node-a8-3:~# miniterm.py /dev/ttyA8_M3 500000
		```

    * Connect to the MQTT broker using the 'con' command and the global IPv6 address obtained in step 5, then subscribe to the 'test/riot' topic using the'sub' command.

        ```
		> con 2001:660:3207:400::66 1885
		> sub test/riot
		```    

7. Test the publish/subscribe mechanism between the MQTT broker and client nodes using a fourth terminal.

    * Connect to the Grenoble SSH frontend and use the preinstalled mosquitto pub command to send the message "FUAS-RIOT-LAB" to the MQTT broker, referencing the global IPv6 address set in step 5 and the topic name should be test/riot:

        ```
		your_computer$ ssh <username>@grenoble.iot-lab.info
		username@grenoble:~$ mosquitto_pub -h 2001:660:3207:400::66 -p 1886 -t test/riot -m FUAS-RIOT-LAB
		```

    * The MQTT-SN client that has already subscribed to the topic 'test/riot' should produce the following output in the third terminal (used in step 6). 

        ```
		### got publication for topic 'test/riot' [1] ###
		FUAS-RIOT-LAB
		```     


## AWS Implementation         

1. After Successfully created the AWS EC2 instance we will get a IPv4 public IP and will get a range of port as well as configuration key. Then take a another terminal for creating a brocker into the AWS EC2 insentace. 

    ```
    my_computer:~$ sudo ssh -i security_key.pem ubuntu@AWS_Public_IP
    ```

    After successfully login by the IP address, then we will run following commands to install mosquitto:

    ```
    ubuntu@ip-172-31-62-165:~$sudo apt-get update
    ubuntu@ip-172-31-62-165:~$sudo apt-get install mosquitto
    ```

    Then create a new configuration file and set a port number (replace your AWS port), as following our file `aws-config.conf`. 

    ```
    listener 1891 #Replace with your own AWS port
    allow_anonymous true
    ```

    Then run the following command:

    ```
    mosquitto -c aws-config.conf
    ```

2. Open a sencond terminal for AWS, here we will subscribe with AWS public IP as well as the port number.

    ```
    mosquitto_sub -h AWS_Public_IP -p PORT -t test/riot 
    ```

3. Then create a new configuration file and set a port number (replace your port), as following our file `bridge.cnf`.

    ```
    listener 1890 #Replace with your local port

    allow_anonymous true

    connection conn-broker1

    address 52.72.16.87:1891 #Replace with your AWS Public IP and port

    topic # both 0
    topic house/sensor out 0 b1/ ""
    topic house/lamp in 0 ""  b2/

    remote_clientid broker1
    ```

* Then go to the brocker node into the Local Implementation (step 5), for making bridge run this following comand in the right directory:
    ```
    root@node-a8-104:~/A8/riot/RIOT/dist/tools/mosquitto_rsmb# mosquitto -c bridge.cnf
    ```
* After successfully connected the bridge, subscribe form AWS client.
    ```
    mosquitto_sub -h AWS_Public_IP -p PORT -t test/riot
    ```

* Then we will publish from the local client as static data ( from step 7) run the following command with same topic (test/riot).

    ```
    while :; do mosquitto_pub -h 2001:660:5307:3000::68 -p PORT -t test/riot -m temp-12-degree; sleep 1 ;
    ```

Then we will get output same to that shown below will be displayed.

<p align="center">
  <img src="https://github.com/PinkuChanda/Send-data-to-AWS-by-the-MQTT-protocol-using-RIOT-OS/blob/main/assets/temperature.png">
</p>
