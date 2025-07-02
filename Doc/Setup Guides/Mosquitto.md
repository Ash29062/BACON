# Mosquitto Installation guide
## Introduction
Mosquitto is an open source message broker that implements the MQTT protocol. Mosquitto is lightweight and is suitable for use on all devices from low power single board computers to full servers. The Mosquitto project also provides a C library for implementing MQTT clients, and the very popular mosquitto_pub and mosquitto_sub command line MQTT clients.
## Installation
### Windows
Click on the link to initiate the download of the installer for windows  
https://mosquitto.org/files/binary/win64/mosquitto-2.0.21a-install-windows-x64.exe

Run the installer once downloaded, continue with the installation as is, and make note of the install location.

Once the installation is done, add your install location to PATH  
To do that:
<ol>
<li>Open "Edit the system environment variables"
<li>Select "Environment variables" on the bottom right
<li>Select Path in the System Variables Section and Click edit.

![image](./Pictures/Screenshot%202025-07-01%20195905.png)
<li>Click on New and paste the path to your install location (for example C:\Program Files\mosquitto)

![image](./Pictures/Edit%20Environment%20Variable.png)
</ol>

## Verification
Now you have successfully installed mosquitto. You can verify your installation by the following steps
<ol>
<li> Open Terminal and run the following command

`mosquitto_sub -t /home/test `
<li> Open another Terminal and run the following command

`mosquitto_pub -t /home/test -m hello`
<li> You should now see the text "hello" in your first terminal. If yes, then your mosquitto is working fine.