# Azure IoT Edge Hands On Labs - Module 1

Created and maintained by the Microsoft Azure IoT Global Black Belts

## Clone the lab materials locally

The first step is to clone the lab materials locally (you'll need a few components of module2 locally to run).

```cmd
cd \
git clone https://github.com/azureiotgbb/azure-iot-edge-hol
```

## Create an IoT Hub and an "Edge Device"

For the lab exercises, we need an IoT Hub created in an Azure Subscription for which you have administrative access.

Create an IoT Hub in your subscription by following the instructions [here](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-through-portal)

While you are in the Azure portal, let's go ahead and grab a couple of important connection parameters and create an IoT Edge Device

In the IoT Hub blade of the Azure portal for your created IoT Hub, do the following:
* In the left-hand nav bar, click on "Shared Access Policies" and then click on "iothubowner", copy the "Connection String - Primary Key" string and paste it into Notepad.  We'll need it later.  This is your "IoTHub Owner Connection string".  Close the "Shared Access Policy" blade
* In the left-hand nav bar, click on "IoT Edge Devices (preview)"
* click "Add Edge Device"
* Give your IoT Edge Device a name and click "Create"
* once created, find the IoT Edge Device connection string (primary key) and copy/paste this into Notepad.  This is the "IoT Edge Device" connection string

## Create docker hub account

IoT Edge modules are pulled by the Edge runtime from a docker containder image repository.  You can host one locally in your own network/infrastrcuture if you choose, Azure offers a [container service](https://azure.microsoft.com/en-us/services/container-service/)  and of course, Docker themselves offer a respository (docker hub).  For simplicity, we will run the labs based off of hosting images in docker hub.  If you feel confident in doing so, feel free to leverage other docker image responsitories instead of docker hub if you wish.

For Docker Hub, you need a Docker ID.  Create one by visting www.docker.com and clicking on "Create Docker ID" and following the instructions.  Remember the docker ID you create, as we'll use it later.  Generally, docker images are referred to in a three part name:  \<respository>/image:tag where "respository", if using Docker Hub, is just your Docker ID,  image is your image name, and tag is an optional "tag" you can use to have multiple images with the same name (often used for versioning)

## Install Prerequisites

In order to execute the hands-on labs, there are a number of pre-requisites that need to be installed and configured.  Unless otherwise noted, the default installation of the items below are fine

__** Note - for in-person deliveries by the IoT GBBs, some of this may have been done for you.  Please check with your instructor **__

* Windows 10 Fall Creators Update (build 16299)
* [Docker for Windows](https://docs.docker.com/docker-for-windows/install/)   ** the "community edition" is fine.  A reboot may be required to enable Hyper-V
* [Python 2.7 for Windows](https://www.python.org/downloads/)
    * during setup, check the box to "add python 2.7 folder to the path"
* [Visual Studio Code](https://code.visualstudio.com/)
* [.NET Core SDK](https://www.microsoft.com/net/core#windowscmd)
* [Arduino IDE](http://www.arduino.cc/)
* [Open SSL](https://sourceforge.net/projects/openssl/)
    * for the lab instructions later, create a c:\utils folder and unzip the downloaded OpenSSL zip to c:\utils\ 
* clone the Azure IoT C sdk.  We need this to get the certificate generation scripts.  Also, while Edge is in public preview, we need the 'modules-preview' branch of the SDK.  Run the following command from the root of the "C" drive
    * git clone -b modules-preview http://github.com/azure/azure-iot-sdk-c


## Additional miscellaneous setup

There are a few final steps needed to set up our specific lab scenario.  We are using our Edge device "as a gateway*, so we need a) our IoT Device to be able to find it and b) to have valid certificates so the IoT Device will open a successful TLS connection to the Edge

* Add a host file entry for our Edge device -- this will let out "IoT Device" resolve and find our Edge gateway.  To do this:
    * Open a command prompt __*as an Administrator*__
    * open (with notepad) c:\windows\system32\drivers\etc\hosts
        * notepad.exe c:\windows\system32\drivers\etc\hosts
    * add a row at the bottom with the following
        * 127.0.0.1  mygateway.local
    * save and close the file
    * confirm you can successfully "ping mygateway.local"

* During the public preview, Edge only works with RSA certs.  Edit the following PowerShell script   C:\azure-iot-sdk-c\tools\CACertificates\ca-certs.ps1
    * on line 26, change the $useECC from $true to $false
    * save and close the script
* Open a PowerShell session __*as an Adminstrator*__
    * make an \edge folder   (mkdir c:\edge)
    * cd to the \edge folder (cd \edge)
    * Run "Set-ExecutionPolicy Unrestricted"
    * Run the following commands to set up our use of OpenSSL
        * $ENV:PATH += ";c:\utils\OpenSSL\bin"
        * $ENV:OPENSSL_CONF="c:\utils\OpenSSL\bin\openssl.cnf"
    * import the ca-certs script with the following command (note the leading dot)
        * . \azure-iot-sdk-c\tools\CACertificates\ca-certs.ps1
    * Run Test-CACertsPrerequisites and make sure it returns the result "SUCCESS"
    * DO NOT CLOSE THE POWERSHELL session yet
        * (if you do, just reopen it and re-add the environment variables above)

* We are now ready to generate the TLS certificates for our Edge device
    * make sure you are still in the c:\edge folder in your PowerShell session
    * run "New-CACertsCertChain" to generate our test certs  (in production, you would use a real CA for this...)
    * in the azure portal, navigate back to your IoT Hub and click on "Certificates" on the left-nav and click "+Add".  Give your certificate a name, and upload the c:\edge\RootCA.cer" file

## Install IoT Edge configuration tool

Microsoft provides a python-based, cross-platform configuration and setup tool for IoT Edge.  To install the tool, open an administrator command prompt and run:

```
pip install -U azure-iot-edge-runtime-ctl
```

## Configure and start IoT Edge

Now that we have all the pieces in place, we are ready to start up our IoT Edge device.  We will start it by specifying the IoT Edge Device connection string capture above, as well as specifying the certificates we generated to allow downstream devices to establish valid TLS sessions with our Edge gateway.

To setup and configure our IoT Edge device, run the following command.

```

iotedgectl setup --connection-string "<Iot Edge Device connection string>" --edge-hostname "mygateway.local" --device-ca-cert-file c:\edge\myGateway-public.pem --device-ca-chain-cert-file c:\edge\myGateway-all.pem --device-ca-private-key-file c:\edge\myGateway-private.pem --owner-ca-cert-file c:\edge\RootCA.pem
    
```
Replace *IoT Edge Device connection string* with the Edge device connection string you captured above

We're ready now to start our IoT Edge device

```
iotedgectl start
```

You can see the status of the docker images by running 

```
docker ps
```

at this point (because we haven't added any modules to our Edge device yet), you should only see one container/module running called 'edgeAgent'

If you want to see if the edge Agent successfully started, run

```
docker logs -f edgeAgent
```

CTRL-C to exit the logs when you are ready

__**Congratulations -- you now have an IoT Edge device up and running and ready to use**__

