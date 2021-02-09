# ePNS Build Guide

# Introduction

---

Welcome to the ePNS build guide. You will learn how to create a mesh network over WIFI using Raspberry Pi's. After creating the mesh you will allow non-mesh devices to connect to the mesh, then use the mesh to host the ePNS website. 

### Project Aim

---

Create a mesh network of raspberry pi’s that can be deployed (via drones or people) in a disaster area to establish communication between the people and the authorities. Each raspberry pi node will broadcast a WIFI network that the user can connect to via their mobile device. Upon connecting to the WIFI network the user will be able to access a portal in which they may alert authorities of their current situation (e.g., if they need first aid, food and find the latest information. The authorities can then view this data and use it to formulate an action plan and locate people they otherwise may be unaware of.

### Network Definitions

---

- Mesh [node](https://en.wikipedia.org/wiki/Node_(networking)): Any network device that is connected to the mesh network and that helps routing data to (and from) mesh clients.
- [Bridge](https://en.wikipedia.org/wiki/Bridging_(networking)): A network device that joins any two or more network interfaces (e.g., LAN ethernet and wireless) into a single network.
- [DNS](https://en.wikipedia.org/wiki/Domain_Name_System): In brief, a system for translating domain names (e.g., `cgomesu.com`) into IP addresses (`185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`).
- [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol): An IP management system that dynamically assigns layer-3 addresses for devices connected to a network. For instance, it might dynamically assign IPs between `192.168.1.0` and `192.168.1.255` (i.e., `192.168.1.0/24`) to any devices connected to LAN.

### Network Topologies

---

In a bus topology, all nodes in the network are connected directly to a central cable that runs up and down the network - this cable is known as the backbone. Data is sent up and down the backbone until it reaches the correct node.

In a ring topology, the nodes create a circular data path. Each node is connected to two other nodes, like points on a circle.

In a star topology all the nodes are connected to a single hub through a cable. This hub is the central node and all others nodes are connected to the central node. This is the most common type of network. 

### How does our project work?

---

There are many ways of routing packets in a mesh network. A few notable ones are the Optimized Link State Routing (OLSR) and the Hybrid Wireless Mesh Protocol (HWMP). We have chosen to use the routing protocol: *Better Approach to Mobile Adhoc Networking Advanced.* It has long been incorporated into the linux kernel and therefore will work perfectly fine on the Raspberry Pi. Here is a description of B*etter Approach to Mobile Adhoc Networking Advanced* from the [keneral.org](https://www.kernel.org/doc/html/latest/networking/batman-adv.html) website. 

> batman-advanced operates on ISO/OSI Layer 2 only and uses and routes (or better: bridges) Ethernet Frames. It emulates a virtual network switch of all nodes participating. Therefore all nodes appear to be link local, thus all higher operating protocols won’t be affected by any changes within the network. You can run almost any protocol above batman advanced, prominent examples are: IPv4, IPv6, DHCP, IPX

Each unit contains two Raspberry Pi 0's configured with batman-adv. One of the Pi 0's will be a mesh node and therefore will connect to other nodes on the mesh network. The other Pi 0 will be a bridge node which will bridge the mesh network over to a WIFI enabled access point. 

The user can then connect (via their devices WIFI menu) to the access point to gain access to the mesh network and in turn the ePNS services. 

In order to access the ePNS services after connecting, the users will download either the ePNS Portal app (for the end user) or the ePNS Responder app (for first responders). Upon opening either of the apps a warning message will be displayed. The user can then acknowledge the warning (by pressing confirm or login) and will be sent to the corresponding page on the ePNS website. 

The ePNS website is split into two parts: ePNS Portal (for the end user) and ePNS Responder (for first responder). 

ePNS Portal contains two pages, a SOS form and a information page. The user can use the SOS form to send details about their emergency to first responders and the information page contains all the latest government information. 

ePNS Responder only contains a single page for first responders to view SOS form responses. The information will be displayed in a table. To prevent unauthorized access the page will be locked behind a login screen - each responder will have their own set of credentials, which will be made in advanced, and given out by a designated dispatcher. 

### Outcomes

---

After completing this build guide you should be able to:

- create a mesh network using Raspberry Pi's
- enable non-mesh devices to connect to the mesh
- create a HTML form which will save the result into a mySQL database using PHP
- create basic HTML page with cards
- use HTML and PHP to create a table to read data from a mySQL database
- create a login system using HTML and PHP
- create an android app for end users

### Required Hardware

---

To complete this build guide you need the following hardware:

- A minimum of 2 Raspberry Pi's (They must be WIFI enabled)
- USB WIFI Dongles (one for each pi). I used [this](https://www.argos.co.uk/product/1420579?) dongle, costing £9.99

The total cost for this project, for me, was £79.99 including the Pi's and £9.99 without the Pi's. 

### Required Software

---

To complete this build guide you need the following software:

- B.A.T.M.A.N. advanced - a implementation of the B.A.T.M.A.N. routing protocol in form of a linux kernel module operating on layer 2
- batctl - a program to control and configure B.A.T.M.A.N
- dnsmasq - a program which acts as a DNS forwarder and DHCP server
- bridge-utils - a program to create and manage bridge devices
- hostapd - a program to convert a network interface into a access point
- apache - a HTTP web server
- PHP - a general-purpose scripting language
- MariaDB - a mySQL databse engine
- Android Studio - a IDE for developing android apps.
- A text editor.

### Outline

---

This build guide will be split into 3 parts: 

- Part 1 will setup everything related to the mesh network.
- Part 2 will setup apache2, mySQL and create the ePNS website.
- Part 3 will create the Android apps.

# Part 1 - Mesh Network

---

In the first part of this build guide you will create a mesh network and allow non-mesh devices to connect to the mesh. 

### Creating the Mesh Network

---

To create our mesh network on the Raspberry Pi, we will use batman-adv which is part of the standard linux kernel. We will configure batman-adv to create a mesh network over WIFI using the WIFI interface wlan0. Batman-adv will then create a new interface bat0, which allows the Raspberry Pi's to send data across the network. 

Download the latest version of Raspbian from [here](https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit). Download the "Raspberry Pi OS with desktop and recommended software" version. 

Flash the image onto your Raspberry Pi's SD card. You can find instructions for this [here](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up).

Boot up your Raspberry Pi with a keyboard, mouse, monitor and a ethernet connection.

After it has booted, follow the setup wizard on-screen. 

Open up a terminal window (CTRL + ALT + T). 

Now we are going to install `batctl` to do this paste the following command into the terminal: `sudo apt-get install -y batctl`.

Next we are going to create a `.sh` script to configure our mesh network. Open up a terminal window (CTRL + ALT + T), then type: `sudo nano ~/start-batman-adv.sh` to create a new `.sh` file in our root directory. Now add the following content to the file: 

```
#!/bin/bash
# Tell batman-adv which interface to use
sudo batctl if add wlan0
# Set the maximum transmission unit for interface bat0
sudo ifconfig bat0 mtu 1468

# Tell batman-adv this is a gateway client
sudo batctl gw_mode client

# Activates batman-adv interfaces
sudo ifconfig wlan0 up
sudo ifconfig bat0 up
```

Press CTRL + X to save the file and exit the text editor. 

Make the `start-batman-adv.sh` file executable with command :  `chmod +x ~/start-batman-adv.sh`

We need to now create a network interface definition for the wlan0 interface by creating a file. You can do this by opening a terminal window (CTRL + ALT + T) and pasting the following command: `sudo nano /etc/network/interfaces.d/wlan0`. In the the text editor add the following content: 

```
# Activate the wlan0 interface 
auto wlan0
iface wlan0 inet manual
# Define the variables the wlan0 interface should use. 
    wireless-channel 1
    wireless-essid raspi-mesh
    wireless-mode ad-hoc
```

You can replace:

- the channel number with a [valid 2.4 GHz WiFi channel number for your region](https://en.wikipedia.org/wiki/List_of_WLAN_channels).

However, these values must be the same on ALL devices that will form your mesh network.

Now we need to make sure the batman-adv kernel module is loaded at boot time by pasting the following command into a terminal window : `echo 'batman-adv' | sudo tee --append /etc/modules`

Stop the DHCP process from trying to manage the wlan0 interface (as we need it for our mesh network) by pasting the following command into a terminal window : `echo 'denyinterfaces wlan0' | sudo tee --append /etc/dhcpcd.conf`

Finally, we need to make sure that `start-batman-adv.sh` runs at start up. We can do this by editing  `[rc.local](http://rc.local)` with this command `sudo nano /etc/rc.local` in a terminal window and adding this line `/home/pi/start-batman-adv.sh &` before the last line `exit 0`.

Repeat this process on ALL of your Raspberry Pi's that you would like to use. Continue to the next section to create the access point Pi. 

### Creating an access point.

---

In this section, you will convert a mesh Raspberry Pi into an access point and a DHCP server. The access point will connect the USB adapter network interface to the mesh network interface. 

DHCP is the service that provides network configuration to the devices on a network. For example IP addresses, DNS servers etc. We will configure this access point to be a DHCP server for any device that wants to connect to it.

Open up a terminal window and paste in the following commands. 

Install the DHCP software with the command : `sudo apt-get install -y dnsmasq`

Configure the DHCP server by editing the `dnsmasq.conf`  with this command : `sudo nano /etc/dnsmasq.conf`. 

Add the following to the file:

```
interface=br0
dhcp-range=192.168.199.2,192.168.199.99,255.255.255.0,12h
```

Install the bridge utilities using command : `sudo apt-get install -y bridge-utils`

Install the access point software using command :  `sudo apt-get install -y hostapd`

Create a network interface definition for `wlan1` using this command: `sudo nano /etc/hostapd/wlan1.conf`. 

Add the following to the file: 

```
interface=wlan1
bridge=br0
ssid=ePNS-Portal
hw_mode=g
channel=7
auth_algs=1
wmm_enabled=0
```

Edit the file `/etc/default/hostapd` with the command:  `sudo nano /etc/default/hostapd`

Add the following to the file: 

```
# Defaults for hostapd initscript
#
# See /usr/share/doc/hostapd/README.Debian for information about alternative
# methods of managing hostapd.
#
# Uncomment and set DAEMON_CONF to the absolute path of a hostapd configuration
# file and hostapd will be started during system boot. An example configuration
# file can be found at /usr/share/doc/hostapd/examples/hostapd.conf.gz
#
DAEMON_CONF=""

# Additional daemon options to be appended to hostapd command:-
#   -d   show more debug messages (-dd for even more)
#   -K   include key data in debug messages
#   -t   include timestamps in some debug messages
#
# Note that -B (daemon mode) and -P (pidfile) options are automatically
# configured by the init.d script and must not be added to DAEMON_OPTS.
#
DAEMON_OPTS="-B"
```

Edit the file `/etc/dhcpcd.conf` with the command :  `sudo nano /etc/dhcpcd.conf`

Add the following to the file: 

```
denyinterfaces wlan0 eth0 bat0 wlan1
interface br0 
    static ip_address=192.168.199.1/24
```

Edit the file `start-batman-adv.sh` with the command : `sudo nano ~/start-batman-adv.sh`

```
#!/bin/bash

# Tell batman-adv which interface to use
sudo batctl if add wlan0
sudo ifconfig bat0 mtu 1468

sudo brctl addbr br0
sudo brctl addif br0 bat0 eth0

# Tell batman-adv this is a gateway client
sudo batctl gw_mode client

# Activates the interfaces for batman-adv
sudo ifconfig wlan0 up
sudo ifconfig bat0 up

# Restart DHCP now bridge and mesh network are up
sudo dhclient -r br0
sudo dhclient br0
```

Finally, enable the access point service on wlan1 interface using command :  `sudo systemctl enable hostapd@wlan1.service`

Boot up all your Pi's and your mesh network is ready to go!

# Part 2 - Website

---

In this section you will setup the required software, setup the mySQL database and create the ePNS portal and responder websites. The portal website will be for end users and the responder page for first responders. 

### Installation

---

Open up a terminal window and paste in the following commands. 

Install the apache webserver with this command: `sudo apt-get install apache2 -y`

Install the PHP package with this command : `sudo apt-get install php -y`

Install the MariaDB Server and PHP-MySQL packages with this command : `sudo apt-get install mariadb-server php-mysql -y`

### Gathering Files

---

You will need to download a few files before continuing. 

Download the [build guide folder](https://github.com/bluescreened803/ePNS/raw/main/Build%20Guide%20Files.zip) from our repository and paste the contents of the folder into the folder `/var/www/html`

Download [material design bootstrap](https://mdbootstrap.com/download/mdb-ui-kit/free/300fEbeeREjvwYseGfe/MDB-UI-KIT-Free-3.0.0.zip) and paste the `css, js, and src` folders into the folder `/var/www/html`

Copy the `font` folder and `font.css` from the `build guide folder` into the `css` folder. 

### Database Setup

---

Open up a terminal window and paste in the following commands:

Access the `mySQL` prompt with this command: `sudo mysql -u root -p`

We now want to create a new user to access the mysql server. To do this run the following command at the `mysql` prompt: `CREATE USER 'user'@'%' IDENTIFIED BY 'password';`

Grant the user we just created full access with this command: GRANT ALL PRIVILEGES ON *.* TO 'user'@'%';

Finally, we can reload the changes with this command: `FLUSH PRIVILEGES;`

Now we need to create the database, we are going to name it form, we do it with this command: `create database form;`

To select the database we just made, use the command : `use form;`

We are now going to create two tables, one for storing the SOS form responses and the other for users. 

For the table users we want the following information:

- id : a number which increments for every account (eg account 1, account 2)
- username : the username of the account
- password : the password of the account
- created_at : the date it was created.

The command for creating the table is: 

```
CREATE TABLE users (
    id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

For the table responses we want the following information: 

- id : a number which increments for every SOS (eg SOS 1, SOS 2)
- name
- mobile
- email
- location
- emergency
- need
- peoplenum
- message

The command for creating this table is: 

```
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    mobile VARCHAR(255),
    email VARCHAR(255),
    location VARCHAR(255),
    emergency VARCHAR(255),
    need VARCHAR(255),
    peoplenum VARCHAR(255),
    message VARCHAR(255),
);
```

Our database is now setup. 

### Config.php

---

We are going to code the file `config.php` , this will contain a set of universal variables, for connecting to the database, that will be present throughout the project. 

Open up`config.php` using a text editor of your choice. 

A PHP script starts with `<?php` and ends with `?>` so in our file add the following: 

```php
<?php 

?>
```

Now lets create the variables that we need. We want to create the following variables: 

- DB_SERVER : where the server is located, in this case `localhost`
- DB_USERNAME : username for the server (we created one earlier)
- DB_PASSWORD : password for the server (we created one earlier)
- DB_NAME : the name of your database (we named it earlier)

In our file add the following to create variables:

```php
define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'root');
define('DB_PASSWORD', '');
define('DB_NAME', 'form');
```

We now need to create the variable that handles the connecting. It will incorporate all the variables we just made. 

In our file add the following. 

```php
$link = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
```

Finally we need to make sure that if the server is unavailable the user gets a error message. We can do this with a conditional statement. 

In our file add the following: 

```php
if($link === false){
    die("ERROR: Could not connect. " . mysqli_connect_error());
}
```

Here is our finished code: 

```php
<?php

define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'root');
define('DB_PASSWORD', '');
define('DB_NAME', 'form');
 

$link = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD, DB_NAME);
 

if($link === false){
    die("ERROR: Could not connect. " . mysqli_connect_error());
}
?>
```

### Licenses Page

---

This page will contain links to all the corresponding licenses for the website. This page will be linked in a footer on every page. 

Create a title with the `<h1> </h1>` tags. 

```html
<h1>Licences</h1>
```

Create a link to the licenses. 

```html
<p><a href="MDB.pdf" class="text">Material Design Bootstrap License</a></p>
<p><a href="FONT AWESOME.txt" class="text">Font Awesome License</a></p>
<p><a href="APACHE-2.0.txt" class="text">Apache 2.0 License</a></p>
<p><a href="GNU 2.0.txt" class="text">GNU 2.0 License</a></p>
<p><a href="PHP.txt" class="text">PHP</a></p>
```

### HTML Form

---

We are now going to create the HTML SOS form. 

Open up the file `form.html` in your favourite text editor. 

We are going to start by creating a header for this page. This will contain a logo, a title, a subheading and a button which links to another page. 

Start by opening and closing the `<header> </header>` tags and our `<div> </div>` tags.

```html
<header> 
<div class="p-5 text-center bg-light">

</div>
</header>
```

Lets add in the logo and resize it to 128x128. 

```html
<img src="img/logo.png" alt="Italian Trulli" width=128 height=128>
```

Now lets create our heading and subheading. 

```html
<h1 class="mb-3">ePNS Portal</h1>
<h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Portal. Please fill out the form below to send a SOS to first responders.</h4>
```

Finally lets add the button linking to another page. 

```html
<a class="btn btn-primary" href="information.html" role="button">Latest Updates</a>
```

Here is our final code for the header: 

```html
<header>
      <div class="p-5 text-center bg-light">
        <img src="img/logo.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Portal</h1>
        <h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Portal. Please fill out the form below to send a SOS to first responders.</h4>
        <a class="btn btn-primary" href="information.html" role="button">Latest Updates</a>
      </div>
</header>
```

We are now going to create the main part of this page, the SOS form. 

Start by opening and closing our `<form> </form>` tags. We will also tell the form what to do with data. In this case, we want to execute `insert.php` and use the `POST` method of form submission. There are two methods to handle data in a HTML form, `POST` and `GET`. The `POST` method is used to send data to a server. The `GET` method is used to request data from a specified resource.  

```html
<form action="insert.php" method="post"> 

</form>
```

We want to add the following questions to our form: 

- What is your name?
- What is your mobile number?
- Where are you?
- What is your emergency?
- What do you need in the next 24 hours?
- How many of you are there?
- Optional Message

Lets start by creating the first question, "what is your name?". We will start by defining the styling for the text, using the `<div> </div>` tags. 

```html
<div class="form-outline mb-4">

</div>
```

The `<input>` element is the most used form element. An `<input>` element can be displayed in many ways. In this case we use the `text` element in the `type` attribute. We will also give the question a `placeholder` of "Sean Lin". Finally, we need a way to identify the question so we will use the `id`and `name` attributes. 

Note: `id` and `name` must be the same.

```html
<div class="form-outline mb-4">
        <input type="text" id="name" class="form-control" name="name" placeholder="Sean Lin"/>
        <label class="form-label" for="name">What is your name?</label>
</div>
```

The process is the same for all the other questions. Just make sure you give them all separate `id` and `name` attributes, as you will need them later.  

Here is the final code for the form questions: 

```html
<div class="form-outline mb-4">
        <input type="text" id="name" class="form-control" name="name" placeholder="Sean Lin"/>
        <label class="form-label" for="name">What is your name?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="mobile" class="form-control" name="mobile" placeholder="123-45-678"/>
        <label class="form-label" for="mobile">What is your mobile number?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="location" class="form-control" name="location" placeholder="Closest address, building, description"/>
        <label class="form-label" for="location">Where are you?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="emergency" class="form-control" name="emergency" placeholder="Injury/Medical, Fire/Explosion, Violence, Death, Trapped, Other"/>
        <label class="form-label" for="emergency">What is your emergency?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="need" class="form-control" name="need" placeholder="First Aid, Shelter, Basic Food, Hygiene, Clothing, Evacuation, Other"/>
        <label class="form-label" for="need">What do you need in the next 24 hours?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="peoplenum" class="form-control" name="peoplenum" placeholder="10"/>
        <label class="form-label" for="peoplenum">How many of you are there?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="message" class="form-control" name="message" placeholder="Max 255 Words"/>
        <label class="form-label" for="message">Optional Message</label>
</div>
```

The last part of the form is the `submit` button. First we need to define our `<div> </div>` tags and to make the button centred we will add some `<br> </br>` tags. 

```html
<br><div class = "text-center">
        <input type="submit" class="btn btn-primary" name="submit" value="Submit">
</div></form></br>
```

All we need to do know for the button, is define the `<input>` tags, the type will be `submit,` the class `btn btn-primary`, a value of `submit` and a name of `submit`.

Here is the final code for the submit button: 

```html
<br><div class = "text-center">
        <input type="submit" class="btn btn-primary" name="submit" value="Submit">
</div></form></br>
```

The last element for this page is a footer which contains the text "Powered by a Raspberry Pi" and a button which links to the page`lisences.html`. 

Start by defining the footer with the `<footer> </footer>` tags. Then we use the `<div> </div>` to define some styling for the footer, the `class text-center p-3`  and the style `background-color: rgba(0, 0, 0, 0.2)`. 

```html
<footer class="bg-light text-center text-lg-start">
      <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">

</div>
```

Then inside the tags, we have the text "Powered by a Raspberry Pi" and a link to the licences page. 

```html
Powered by a Raspberry Pi.
<a href="Licences.html" class="text-dark">View licences here.</a>
```

Here is the final code for the footer: 

```html
<footer class="bg-light text-center text-lg-start">
      <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
        Powered by a Raspberry Pi.
        <a href="Licences.html" class="text-dark">View licences here.</a>
      </div>
</footer>
```

You have now successfully created the HTML form. Here is the final code for the HTML form: 

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no" />
    <meta http-equiv="x-ua-compatible" content="ie=edge" />
    <title>ePNS Portal</title>
    <link rel="icon" href="img/logo.png" type="image/x-icon" />
    <link rel="stylesheet" href="css/all.css" />
    <link
      rel="stylesheet"
      href="css/font.css"
    />
    <link rel="stylesheet" href="css/mdb.min.css" />
  </head>
  <body>
    <header>
      <div class="p-5 text-center bg-light">
        <img src="img/logo.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Portal</h1>
        <h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Portal. Please fill out the form below to send a SOS to first responders.</h4>
        <a class="btn btn-primary" href="information.html" role="button">Latest Updates</a>
      </div>
    </header>
    <form action="insert.php" method="post">
      <div class="form-outline mb-4">
        <input type="text" id="name" class="form-control" name="name" placeholder="Sean Lin"/>
        <label class="form-label" for="name">What is your name?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="mobile" class="form-control" name="mobile" placeholder="123-45-678"/>
        <label class="form-label" for="mobile">What is your mobile number?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="location" class="form-control" name="location" placeholder="Closest address, building, description"/>
        <label class="form-label" for="location">Where are you?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="emergency" class="form-control" name="emergency" placeholder="Injury/Medical, Fire/Explosion, Violence, Death, Trapped, Other"/>
        <label class="form-label" for="emergency">What is your emergency?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="need" class="form-control" name="need" placeholder="First Aid, Shelter, Basic Food, Hygiene, Clothing, Evacuation, Other"/>
        <label class="form-label" for="need">What do you need in the next 24 hours?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="peoplenum" class="form-control" name="peoplenum" placeholder="10"/>
        <label class="form-label" for="peoplenum">How many of you are there?</label>
      </div>
      <div class="form-outline mb-4">
        <input type="text" id="message" class="form-control" name="message" placeholder="Max 255 Words"/>
        <label class="form-label" for="message">Optional Message</label>
      </div>
      <br><div class = "text-center">
        <input type="submit" class="btn btn-primary" name="submit" value="Submit">
      </div></form></br>
    </form>
    <footer class="bg-light text-center text-lg-start">
      <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
        Powered by a Raspberry Pi.
        <a href="Licences.html" class="text-dark">View licences here.</a>
      </div>
    </footer>
  </body>
  <script type="text/javascript" src="js/mdb.min.js"></script>
  <script type="text/javascript"></script>
</html>
```

### Insert.php

---

This PHP script will handle the data that passes through the HTML form. We want the script to insert the data into a mySQL server under the correct headings. We also want it to give the user confirmation that the data has been saved, or an error if not .

Open `insert.php` in your favourite text editor. 

Lets start by opening the PHP tags.

```php
<?php

?>
```

Load in the `config.php` we made earlier. 

```php
include_once 'config.php';
```

Check whether the form has been completed or not, if so continue on with the script. We will use a IF statement. 

```php
if(isset($_POST['submit']))
{

}
```

Define the variables with the data collected from the form. You will need to use the `id` and `name` you set in the HTML form. 

```php
$name = $_POST['name'];
$mobile = $_POST['mobile'];
$location= $_POST['location'];
$emergency= $_POST['emergency'];
$need= $_POST['need'];
$peoplenum = $_POST['peoplenum'];
$message= $_POST['message'];
```

Create a variable which will insert the form values into the database table `responses`. 

```php
$sql = "INSERT INTO responses (name,mobile,location,emergency,need,peoplenum,message)
VALUES ('$name','$mobile','$location','$emergency', '$need','$peoplenum','$message');
```

Insert the data into the database, and provide either a success or a error message. We will use a IF statement for this. 

```php
if (mysqli_query($link, $sql)) {
```

To include HTML in a PHP script, you use the `echo` command. We will be doing this to provide the user with a success message. 

Create a title: 

```php
echo "<h2><center>The response has been recorded.</center></h2>";
```

Create a centred button using the `<div> </div` tags:

```php
echo "<div class='text-center'>";
echo "<a class='btn btn-primary' href='form.html' role='button'>SOS Form</a>";
echo "</div>";
```

Create a detailed error message using the ELSE statement: 

```php
} else {
echo "Error: " . $sql . ":-" . mysqli_error($link);
}
```

Close the connection

```php
mysqli_close($link);
```

Here is the final code for `insert.php:` 

```php
<?php
include_once 'config.php';
if(isset($_POST['submit']))
{    
     $name = $_POST['name'];
     $mobile = $_POST['mobile'];
     $location= $_POST['location'];
     $emergency= $_POST['emergency'];
     $need= $_POST['need'];
     $peoplenum = $_POST['peoplenum'];
     $message= $_POST['message'];
     $sql = "INSERT INTO responses (name,mobile,location,emergency,need,peoplenum,message)
     VALUES ('$name','$mobile','$location','$emergency', '$need','$peoplenum','$message');
     if (mysqli_query($link, $sql)) {
        echo "<h2><center>The response has been recorded.</center></h2>";
        echo "<div class='text-center'>";
        echo "<a class='btn btn-primary' href='form.html' role='button'>SOS Form</a>";
        echo "</div>";

     } else {
        echo "Error: " . $sql . ":-" . mysqli_error($link);
     }
     mysqli_close($link);
}
?>
```

### HTML Information Page

---

This page will contain the latest government information for the end user. 

Open up `information.html`, with your favourite text editor. 

Start by creating a header, the process is the same as in the HTML form. 

```html
<header>
      <div class="p-5 text-center bg-light">
        <img src="img/logo.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Portal</h1>
        <h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Portal. Here you can find up to date information about what is happenning.</h4>
        <a class="btn btn-primary" href="form.html" role="button">SOS Form</a>
      </div>
</header>
```

This page is going to contain cards, which have information on them. They can be updated at any time by editing the HTML. Lets create an example card with the following information: 

- Footer: 10 minutes ago
- Header: Government Update
- Title: Free emergency shelters
- Text: Head to your local community centre for emergency centres.

Start the card by using the `<div> </div>` tags. The class will be `card`. All of the code for the card will be in these tags. 

```html
<div class="card "> 

</div>
```

Create the header by using the `<div> </div>` tags. The class will be `card-header` . 

```html
<div class="card-header">Government Update</div>
```

Create the body of the card by using the `<div> </div>` tags. The class will be `card-body` . The code for the body will be in these tags. 

```html
<div class="card-body">

</div>
```

Create the cards title using the `<h5> </h5>` tags. The class will be `card-text`. 

```html
<h5 class="card-title">Free Emergency Shelters</h5>
```

Create the cards text with the `<p> </p>` tags. The class will be `card text.` 

```html
<p class="card-text">
          Head to your local community centre for emergency centers. 
</p>
```

Finally, create the footer with the `<div> </div>` tags. 

```html
<div class="card-footer">1o minutes ago</div>
```

Here is the final code for our card:

```html
<div class="card  ">
            <div class="card-header">Government Update</div>
            <div class="card-body">
              <h5 class="card-title">Free Emergency Shelters</h5>
              <p class="card-text">
                   Head to your local community centre for emergency centers. 
              </p>
            </div>
            <div class="card-footer">1 hour ago</div>
</div>
```

Here is an example of another card: 

```html
<div class="card  ">
            <div class="card-header">Goverment Advice</div>
            <div class="card-body">
              <h5 class="card-title">Dealing with the immediate aftermath</h5>
              <p class="card-text">
                <ul class="list-unstyled">
                  <ul>
                    <li>Make sure all household members are accounted for.</li>
                    <li>Attend to physical injuries or emotional distress. In cases of serious injury, summon professional help.</li>
                    <li>Be aware of any new safety issues created by the disaster, such as damaged roads/bridges, chemical spills, downed power lines, and washed-out roads.</li>
                    <li>Stay tuned with the information page on this website for information.</li>
                  </ul>                 
              </p>
            </div>
            <div class="card-footer">10 minutes ago</div>
</div>
```

To finish this page, add the same footer you made before: 

```html
<footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
          Powered by a Raspberry Pi.
          <a href="Licences.html" class="text-dark">View licences here.</a>
        </div>
</footer>
```

Here is the final code for `information.html` .

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no" />
    <meta http-equiv="x-ua-compatible" content="ie=edge" />
    <title>ePNS Portal</title>
    <link rel="icon" href="img/logo.png" type="image/x-icon" />
    <link rel="stylesheet" href="css/all.css" />
    <link
      rel="stylesheet"
      href="css/font.css"
    />
    <link rel="stylesheet" href="css/mdb.min.css" />
  </head>
  <body>
    <header>
      <div class="p-5 text-center bg-light">
        <img src="img/logo.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Portal</h1>
        <h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Portal. Here you can find up to date information about what is happenning.</h4>
        <a class="btn btn-primary" href="form.html" role="button">SOS Form</a>
      </div>
    </header>
    <p class="note note-danger">
        <strong>Notice:</strong> This is for demonstration purposes only.
      </p>
      <div class="d-grid gap-3">
          <div class="card  ">
            <div class="card-header">Goverment Advice</div>
            <div class="card-body">
              <h5 class="card-title">Dealing with the immediate aftermath</h5>
              <p class="card-text">
                <ul class="list-unstyled">
                  <ul>
                    <li>Make sure all household members are accounted for.</li>
                    <li>Attend to physical injuries or emotional distress. In cases of serious injury, summon professional help.</li>
                    <li>Be aware of any new safety issues created by the disaster, such as damaged roads/bridges, chemical spills, downed power lines, and washed-out roads.</li>
                    <li>Stay tuned with the information page on this website for information.</li>
                  </ul>                 
              </p>
            </div>
            <div class="card-footer">10 minutes ago</div>
          </div>
          <div class="card  ">
            <div class="card-header">Government Update</div>
            <div class="card-body">
              <h5 class="card-title">Free Emergency Shelters</h5>
              <p class="card-text">
                   Head to your local community centre for emergency centers. 
              </p>
            </div>
            <div class="card-footer">1 hour ago</div>
          </div>
          <div class="card  ">
            <div class="card-header">Charity Information</div>
            <div class="card-body">
              <h5 class="card-title">Financial Support Available</h5>
              <p class="card-text">
                  Financial support is available for those who need it. Find a representative for more information.
              </p>
            </div>
            <div class="card-footer">2 hour ago</div>
          </div>
      </div>
      <footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
          Powered by a Raspberry Pi.
          <a href="Licences.html" class="text-dark">View licences here.</a>
        </div>
      </footer>
  </body>
  <script type="text/javascript" src="js/mdb.min.js"></script>
  <script type="text/javascript"></script>
</html>
```

### Admin.php

---

This page is for first responders to view form responses. 

Open up the file `admin.php` in your favourite text editor. 

Start by checking if the user is logged in with a PHP script. First we will either create a login session or resume a current login session, if the user is not logged in then it will send the user to the login page. Utilising an IF statement.  

```php
<?php

session_start();
 

if(!isset($_SESSION["loggedin"]) || $_SESSION["loggedin"] !== true){
    header("location: login.php");
    exit;
}
?>
```

Create a header similar to the one we created in the HTML form page, except changing a few details to be more relevant. We will add in a subheading to greet the user with their username. We will also add in a logout button, which will execute `logout.php`

```php
<header>
<div class="p-5 text-center bg-light">
        <img src="img/logo2.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Responder</h1>
        <h4 class="mb-3">Hi, <?php echo htmlspecialchars($_SESSION["username"]); ?>. You are now connected to the ePNS (Emergency-Pi-Network-System) Responder. Here you can view the latest SOS responses.</h4>
        <a href="logout.php" class="btn btn-danger" style="background-color:#ff0000">Sign Out of Your Account</a>
      </div>
</header>
```

Now we are going to create a table which displays the SOS form results. Start by doing some formatting and declaring some styling. 

```php
<div class="bs-example">
<div class="container">
<div class="row">
<div class="col-md-12">
<div class="page-header clearfix">
</div>
```

Now lets select all the current responses with a PHP script. 

```php
<?php
include_once 'config.php';
$result = mysqli_query($link,"SELECT * FROM responses");
?>
```

Now lets check the number of rows in the table we fetched from the variable result. 

```php
<?php
if (mysqli_num_rows($result) > 0) {
?>
```

Lets create our table with the following headings: 

- Name
- Mobile
- Location
- Emergency
- Needs
- Number of people
- Message

We will use the `<tr> </tr>` and `<table>` tags. 

```html
<table class='table'>
<tr>
<td>Name</td>
<td>Mobile</td>
<td>Location</td>
<td>Emergency</td>
<td>Needs</td>
<td>Number of people</td>
<td>Message</td>
</tr>

```

Now lets convert the fetched data into an array which PHP can understand. 

```php
<?php
$i=0;
while($row = mysqli_fetch_array($result)) {
?>
```

Now that the data is in the correct format, we can put it into the table using HTML. 

```php
<tr>
<td><?php echo $row["name"]; ?></td>
<td><?php echo $row["mobile"]; ?></td>
<td><?php echo $row["location"]; ?></td>
<td><?php echo $row["emergency"]; ?></td>
<td><?php echo $row["need"]; ?></td>
<td><?php echo $row["peoplenum"]; ?></td>
<td><?php echo $row["message"]; ?></td>
</tr>
```

Create an error message if no results are found and lets do some cleaning up. 

```php
<?php
$i++;
}
?>
</table>
<?php
}
else{
echo "No result found";
}
?>
```

Add the footer you made earlier on the HTML form page. 

```html
<footer class="bg-light text-center text-lg-start">
  <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
    Powered by a Raspberry Pi.
  <a href="Licences.html" class="text-dark">View licences here.</a>
  </div>
</footer>
```

Here is the final code for `admin.php` . 

```php
<?php
session_start();
 
if(!isset($_SESSION["loggedin"]) || $_SESSION["loggedin"] !== true){
    header("location: login.php");
    exit;
}
?>
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no" />
    <meta http-equiv="x-ua-compatible" content="ie=edge" />
    <title>ePNS Responder</title>
    <link rel="icon" href="img/logo2.png" type="image/x-icon" />
    <link rel="stylesheet" href="css/all.css" />
    <link
      rel="stylesheet"
      href="css/font.css"
    />
    <link rel="stylesheet" href="css/mdb.min.css" />
  </head>
  <body>
      <div class="p-5 text-center bg-light">
        <img src="img/logo2.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Responder</h1>
        <h4 class="mb-3">Hi, <?php echo htmlspecialchars($_SESSION["username"]); ?>. You are now connected to the ePNS (Emergency-Pi-Network-System) Responder. Here you can view the latest SOS responses.</h4>
        <a href="logout.php" class="btn btn-danger" style="background-color:#ff0000">Sign Out of Your Account</a>
      </div>
    </header>
    <div class="bs-example">
<div class="container">
<div class="row">
<div class="col-md-12">
<div class="page-header clearfix">
</div>
<?php
include_once 'config.php';
$result = mysqli_query($link,"SELECT * FROM responses");
?>
<?php
if (mysqli_num_rows($result) > 0) {
?>
<table class='table'>
<tr>
<td>Name</td>
<td>Mobile</td>
<td>Location</td>
<td>Emergency</td>
<td>Needs</td>
<td>Number of people</td>
<td>Message</td>
</tr>
<?php
$i=0;
while($row = mysqli_fetch_array($result)) {
?>
<tr>
<td><?php echo $row["name"]; ?></td>
<td><?php echo $row["mobile"]; ?></td>
<td><?php echo $row["location"]; ?></td>
<td><?php echo $row["emergency"]; ?></td>
<td><?php echo $row["need"]; ?></td>
<td><?php echo $row["peoplenum"]; ?></td>
<td><?php echo $row["message"]; ?></td>
</tr>
<?php
$i++;
}
?>
</table>
<?php
}
else{
echo "No result found";
}
?>
</div>
</div>        
</div>
</div>
<footer class="bg-light text-center text-lg-start">
  <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
    Powered by a Raspberry Pi.
  <a href="Licences.html" class="text-dark">View licences here.</a>
  </div>
</footer>
  </body>
  <script type="text/javascript" src="js/mdb.min.js"></script>
  <script type="text/javascript"></script>
</html>
```

### Login.php

---

This script handles the login process. 

Open up the file `login.php` in your favourite text editor. 

Open the `<?php ?>`  tags and start the session. 

```php
<?php
session_start();

?>
```

Check if the user is already logged in, if yes then redirect them to welcome page.

```php
if(isset($_SESSION["loggedin"]) && $_SESSION["loggedin"] === true){
    header("location: welcome.php");
    exit;
}
```

Include config file we made earlier. 

```php
require_once "config.php";
```

Define our username and password variables, for now they will be empty. 

```php
$username = $password = "";
$username_err = $password_err = "";
```

Start the processing of the login data. 

```php
if($_SERVER["REQUEST_METHOD"] == "POST"){
```

Check if username and password are empty. 

```php
if(empty(trim($_POST["username"]))){
        $username_err = "Please enter username.";
    } else{
        $username = trim($_POST["username"]);
    }
    
    if(empty(trim($_POST["password"]))){
        $password_err = "Please enter your password.";
    } else{
        $password = trim($_POST["password"]);
    }
```

Start the validation process, to see if the imputed credentials against the one in the database. 

```php
if(empty($username_err) && empty($password_err)){
```

Create a SQL select statement to grab the username and password. 

```php
$sql = "SELECT id, username, password FROM users WHERE username = ?";
if($stmt = mysqli_prepare($link, $sql)){
```

Place some variables into a statement. 

```php
mysqli_stmt_bind_param($stmt, "s", $param_username);
```

Set the variable you placed into the statement.

```php
$param_username = $username;
```

Start the execution of the statement. 

```php
if(mysqli_stmt_execute($stmt)){
```

Store a variable. 

```php
mysqli_stmt_store_result($stmt);
```

Check if the username exists. 

```php
if(mysqli_stmt_num_rows($stmt) == 1){
```

Store the results in a variable and then check if password exists. 

```php
mysqli_stmt_bind_result($stmt, $id, $username, $hashed_password);
if(mysqli_stmt_fetch($stmt)){
          if(password_verify($password, $hashed_password)){
```

If everything is good, start the session and store the session variables. 

```php
session_start();
    $_SESSION["loggedin"] = true;
    $_SESSION["id"] = $id;
    $_SESSION["username"] = $username;
```

Send the user to the admin page or give an error if non of that worked out. 

```php
header("location: admin.php");
          } else{
                 $password_err = "The password you entered was not valid.";
```

Now we need to create the login form in HTML. 

Lets start by creating a header, we can use the one we made in the HTML form with the relevant changes. 

```jsx
<div class="p-5 text-center bg-light">
        <img src="img/logo2.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Responder Login</h1>
        <h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Responder login. Please enter your credentials.</h4>
      </div>
</header>
```

Start the creation of the form with the `<form></form>` tags. We will tell the form to execute a PHP script when the form is completed and it will use the `POST` method. 

```html
<form action="<?php echo htmlspecialchars($_SERVER["PHP_SELF"]); ?>" method="post">
```

We will be using a similar structure as the form we made earlier. The class will be set to `form-group`           

also we will be using a PHP script to check if the username and password is empty. 

```html
<div class="form-group <?php echo (!empty($username_err)) ? 'has-error' : ''; ?>">
                <label>Username</label>
                <input type="text" name="username" class="form-control" value="<?php echo $username; ?>">
                <span class="help-block"><?php echo $username_err; ?></span>
</div>

<div class="form-group <?php echo (!empty($password_err)) ? 'has-error' : ''; ?>">
                <label>Password</label>
                <input type="password" name="password" class="form-control">
                <span class="help-block"><?php echo $password_err; ?></span>
</div>
```

To finish the form, we need to add a centred submit button. Using the `<br> </br>` tags, the `<div> </div>` and the `<input>` tags. 

```html
<br>
<div class="form-group" class = "text-center">
<input type="submit" class="btn btn-primary" value="Login">
</div>
</br>
```

To finish the page, add a footer. 

```html
<footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
          Powered by a Raspberry Pi.
          <a href="Licences.html" class="text-dark">View licences here.</a>
        </div>
</footer>
```

Here is the final code for `login.php`. 

```php
<?php

session_start();
 

if(isset($_SESSION["loggedin"]) && $_SESSION["loggedin"] === true){
  header("location: admin.php");
  exit;
}
 

require_once "config.php";
 

$username = $password = "";
$username_err = $password_err = "";
 

if($_SERVER["REQUEST_METHOD"] == "POST"){

    if(empty(trim($_POST["username"]))){
        $username_err = "Please enter username.";
    } else{
        $username = trim($_POST["username"]);
    }
    
   
    if(empty(trim($_POST["password"]))){
        $password_err = "Please enter your password.";
    } else{
        $password = trim($_POST["password"]);
    }
    
   
    if(empty($username_err) && empty($password_err)){
        
        $sql = "SELECT id, username, password FROM users WHERE username = ?";
        
        if($stmt = mysqli_prepare($link, $sql)){
           
            mysqli_stmt_bind_param($stmt, "s", $param_username);
            
            
            $param_username = $username;
            
            
            if(mysqli_stmt_execute($stmt)){
               
                mysqli_stmt_store_result($stmt);
                
               
                if(mysqli_stmt_num_rows($stmt) == 1){                    
                  
                    mysqli_stmt_bind_result($stmt, $id, $username, $hashed_password);
                    if(mysqli_stmt_fetch($stmt)){
                        if(password_verify($password, $hashed_password)){
                         
                            session_start();
                            
                            
                            $_SESSION["loggedin"] = true;
                            $_SESSION["id"] = $id;
                            $_SESSION["username"] = $username;                            
                            
                        
                            header("location: admin.php");
                        } else{
                            
                            $password_err = "The password you entered was not valid.";
                        }
                    }
                } else{
                   
                    $username_err = "No account found with that username.";
                }
            } else{
                echo "Oops! Something went wrong. Please try again later.";
            }

           
            mysqli_stmt_close($stmt);
        }
    }
    
  
    mysqli_close($link);
}
?>

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no" />
    <meta http-equiv="x-ua-compatible" content="ie=edge" />
    <title>ePNS Responder Login</title>
   
    <link rel="icon" href="img/logo2.png" type="image/x-icon" />
  
    <link rel="stylesheet" href="css/all.css" />
   
    <link
      rel="stylesheet"
      href="css/font.css"
    />
   
    <link rel="stylesheet" href="css/mdb.min.css" />
  </head>
  <body>
  
    <div class="p-5 text-center bg-light">
        <img src="img/logo2.png" alt="Italian Trulli" width=128 height=128>
        <h1 class="mb-3">ePNS Responder Login</h1>
        <h4 class="mb-3">You are now connected to the ePNS (Emergency-Pi-Network-System) Responder login. Please enter your credentials.</h4>
      </div>
    </header>
        <form action="<?php echo htmlspecialchars($_SERVER["PHP_SELF"]); ?>" method="post">
            <div class="form-group <?php echo (!empty($username_err)) ? 'has-error' : ''; ?>">
                <label>Username</label>
                <input type="text" name="username" class="form-control" value="<?php echo $username; ?>">
                <span class="help-block"><?php echo $username_err; ?></span>
            </div>    
            <div class="form-group <?php echo (!empty($password_err)) ? 'has-error' : ''; ?>">
                <label>Password</label>
                <input type="password" name="password" class="form-control">
                <span class="help-block"><?php echo $password_err; ?></span>
            </div>
            <br>
            <div class="form-group" class = "text-center">
                <input type="submit" class="btn btn-primary" value="Login">
            </div>
            </br>
        </form>
        <footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
          Powered by a Raspberry Pi.
          <a href="Licences.html" class="text-dark">View licences here.</a>
        </div>
        </footer>
  </body>
  <script type="text/javascript" src="js/mdb.min.js"></script>
  <script type="text/javascript"></script>
</html>
```

### Logout.php

---

This script will handle the process. 

Open up the file `logout.php` in your favourite text editor. 

Start by opening the `<?php ?>` tags.  

```php
<?php

?>
```

Initialize the session

```php
session_start();
```

Reset all of the current session variables

```php
$_SESSION = array();
```

Destroy the current session. 

```php
session_destroy();
```

Redirect the user to the login page and exit the script. 

```php
header("location: login.php");
exit;
```

Here is the final code for logout.php

```php
<?php

session_start();
 

$_SESSION = array();
 

session_destroy();
 

header("location: login.php");
exit;
?>
```

### Registration.php

---

This script will create the account required for `admin.php`. 

Open up the file `registration.php` in your favourite text editor. 

Start by opening and closing the `<?php ?>` tags and including the config file. 

```php
<?php
require_once "config.php";

?>
```

Define the username and password with empty values.  

```php
$username = $password = $confirm_password = "";
$username_err = $password_err = $confirm_password_err = "";
```

Start the processing of the data when a form is submitted, with an IF statement. 

```php
if($_SERVER["REQUEST_METHOD"] == "POST"){
```

Start the validation of the username and prepare the SQL select statement. 

```php
if(empty(trim($_POST["username"]))){
        $username_err = "Please enter a username.";
    } else{
        $sql = "SELECT id FROM users WHERE username = ?";
        if($stmt = mysqli_prepare($link, $sql)){
```

Add the variables into the statement. 

```php
mysqli_stmt_bind_param($stmt, "s", $param_username);
```

Define those variables with the username. 

```php
$param_username = trim($_POST["username"]);
```

Execute the statement. 

```php
if(mysqli_stmt_execute($stmt)){
```

Store the result. 

```php
mysqli_stmt_store_result($stmt);
```

Display an error if it did not work and close the statement. 

```php
if(mysqli_stmt_num_rows($stmt) == 1){
                    $username_err = "This username is already taken.";
                } else{
                    $username = trim($_POST["username"]);
                }
            } else{
                echo "Oops! Something went wrong. Please try again later.";
            }

            
            mysqli_stmt_close($stmt);
        }
    }
```

Validate the password .

```php
if(empty(trim($_POST["password"]))){
        $password_err = "Please enter a password.";     
    } elseif(strlen(trim($_POST["password"])) < 6){
        $password_err = "Password must have atleast 6 characters.";
    } else{
        $password = trim($_POST["password"]);
    }
```

Validate the confirm password .

```php
if(empty(trim($_POST["confirm_password"]))){
        $confirm_password_err = "Please confirm password.";     
    } else{
        $confirm_password = trim($_POST["confirm_password"]);
        if(empty($password_err) && ($password != $confirm_password)){
            $confirm_password_err = "Password did not match.";
        }
    }
```

Check for input errors before inserting into the database. 

```php
if(empty($username_err) && empty($password_err) && empty($confirm_password_err)){
```

Prepare an SQL statement. 

```php
$sql = "INSERT INTO users (username, password) VALUES (?, ?)";
if($stmt = mysqli_prepare($link, $sql)){
```

Add the variables into the SQL statement. 

```php
mysqli_stmt_bind_param($stmt, "ss", $param_username, $param_password);
```

Define some variables for the statement and create a password hash. 

```php
$param_username = $username;
$param_password = password_hash($password, PASSWORD_DEFAULT);
```

Execute the statement, display an error if needed, redirect to the login page, and close the statement. 

```php
if(mysqli_stmt_execute($stmt)){
               
                header("location: login.php");
            } else{
                echo "Something went wrong. Please try again later.";
            }

            
            mysqli_stmt_close($stmt);
        }
    }
    
   
    mysqli_close($link);
}
```

We now need to create the registration form. This will be in a made in a similar way as the login form. 

Start the creation of the form with the `<form></form>` tags. We will tell the form to execute a PHP script when the form is completed and it will use the `POST` method. 

```html
<form action="<?php echo htmlspecialchars($_SERVER["PHP_SELF"]); ?>" method="post">
</form>
```

We will be using a similar structure as the form we made earlier. The class will be set to `form-group`           

also we will be using a PHP script to check if the username and password is empty. 

```html
<div class="form-group <?php echo (!empty($username_err)) ? 'has-error' : ''; ?>">
                <label>Username</label>
                <input type="text" name="username" class="form-control" value="<?php echo $username; ?>">
                <span class="help-block"><?php echo $username_err; ?></span>
            </div>    
            <div class="form-group <?php echo (!empty($password_err)) ? 'has-error' : ''; ?>">
                <label>Password</label>
                <input type="password" name="password" class="form-control" value="<?php echo $password; ?>">
                <span class="help-block"><?php echo $password_err; ?></span>
            </div>
            <div class="form-group <?php echo (!empty($confirm_password_err)) ? 'has-error' : ''; ?>">
                <label>Confirm Password</label>
                <input type="password" name="confirm_password" class="form-control" value="<?php echo $confirm_password; ?>">
                <span class="help-block"><?php echo $confirm_password_err; ?></span>
</div>
```

Finally we need to create a submit button 

```html
<div class="form-group">
                <input type="submit" class="btn btn-primary" value="Submit">
                <input type="reset" class="btn btn-default" value="Reset">
</div>
```

Here is the final code for `register.php` . 

```php
<?php

require_once "config.php";
 

$username = $password = $confirm_password = "";
$username_err = $password_err = $confirm_password_err = "";

if($_SERVER["REQUEST_METHOD"] == "POST"){
 

    if(empty(trim($_POST["username"]))){
        $username_err = "Please enter a username.";
    } else{
        
        $sql = "SELECT id FROM users WHERE username = ?";
        
        if($stmt = mysqli_prepare($link, $sql)){
           
            mysqli_stmt_bind_param($stmt, "s", $param_username);
            
            
            $param_username = trim($_POST["username"]);
            
            
            if(mysqli_stmt_execute($stmt)){
                
                mysqli_stmt_store_result($stmt);
                
                if(mysqli_stmt_num_rows($stmt) == 1){
                    $username_err = "This username is already taken.";
                } else{
                    $username = trim($_POST["username"]);
                }
            } else{
                echo "Oops! Something went wrong. Please try again later.";
            }

            
            mysqli_stmt_close($stmt);
        }
    }
    
    
    if(empty(trim($_POST["password"]))){
        $password_err = "Please enter a password.";     
    } elseif(strlen(trim($_POST["password"])) < 6){
        $password_err = "Password must have atleast 6 characters.";
    } else{
        $password = trim($_POST["password"]);
    }
    
   
    if(empty(trim($_POST["confirm_password"]))){
        $confirm_password_err = "Please confirm password.";     
    } else{
        $confirm_password = trim($_POST["confirm_password"]);
        if(empty($password_err) && ($password != $confirm_password)){
            $confirm_password_err = "Password did not match.";
        }
    }
    
    
    if(empty($username_err) && empty($password_err) && empty($confirm_password_err)){
        
      
        $sql = "INSERT INTO users (username, password) VALUES (?, ?)";
         
        if($stmt = mysqli_prepare($link, $sql)){
         
            mysqli_stmt_bind_param($stmt, "ss", $param_username, $param_password);
            
            
            $param_username = $username;
            $param_password = password_hash($password, PASSWORD_DEFAULT); 
            
            if(mysqli_stmt_execute($stmt)){
                
                header("location: login.php");
            } else{
                echo "Something went wrong. Please try again later.";
            }

            
            mysqli_stmt_close($stmt);
        }
    }
    
    
    mysqli_close($link);
}
?>
 
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Sign Up</title>
</head>
<body>
    <div class="wrapper">
        <h2>Sign Up</h2>
        <form action="<?php echo htmlspecialchars($_SERVER["PHP_SELF"]); ?>" method="post">
            <div class="form-group <?php echo (!empty($username_err)) ? 'has-error' : ''; ?>">
                <label>Username</label>
                <input type="text" name="username" class="form-control" value="<?php echo $username; ?>">
                <span class="help-block"><?php echo $username_err; ?></span>
            </div>    
            <div class="form-group <?php echo (!empty($password_err)) ? 'has-error' : ''; ?>">
                <label>Password</label>
                <input type="password" name="password" class="form-control" value="<?php echo $password; ?>">
                <span class="help-block"><?php echo $password_err; ?></span>
            </div>
            <div class="form-group <?php echo (!empty($confirm_password_err)) ? 'has-error' : ''; ?>">
                <label>Confirm Password</label>
                <input type="password" name="confirm_password" class="form-control" value="<?php echo $confirm_password; ?>">
                <span class="help-block"><?php echo $confirm_password_err; ?></span>
            </div>
            <div class="form-group">
                <input type="submit" class="btn btn-primary" value="Submit">
                <input type="reset" class="btn btn-default" value="Reset">
            </div>
        </form>
    </div>    
</body>
</html>
```

# Part 3 - Android App

---

Right, the time has come. You are eager to test this out, before we do, there is one last step. In this section you will create two Android apps, ePNS Portal (for the end user) and ePNS Responder (for first responders). The app will display a warning message before then proceeding to the correct part of the ePNS website.

Here you will need to move onto a Windows PC or Macintosh. We recommend using Windows 10, macOS 10.14 or higher as your operating experience, to provide the best experience and compatibility with Android Studio.

### Installing Android Studio

---

[Here](https://developer.android.com/studio#downloads) you can download the packages for the installation of Android Studio, depending on your operating system. This is around a 1GB in download size, and you will require around 3 to 4 GB to complete the installation. This includes the Simulator files to test and debug the apps. Note for Windows users, we recommend using the .exe installer to install Android Studio.

Read the software license and press agree.

The package will begin downloading, this may take a while, depending on your internet connection.

For Windows users, run the .exe installer and follow the steps to install Android Studio

For Mac users, mount the .dmg image by double clicking it. Then proceed the drag the application to the application folder, indicated within the window after you have mounted the image.

### Creating a new project

---

First of all, open Android Studio on your computer.

Android Studio shows the Welcome screen, where you can create a new project by clicking **Start a new Android Studio project**.

If you do have a project opened, you start creating a new project by selecting **File > New > New Project** from the main menu.

You are now shown to select your project template. For this Android app, we will be using Basic Activity template. Press Next.

Now we can specify a name for our project, choose its save location and select the coding language. Firstly we will be building the ePNS Portal app. The ePNS Responder uses the exactly the same code, however we alter the URL to redirect the user to the Responder section of the ePNS website instead. You can also alter colour theming of the app, comparable to our design, however today, we will focus on the real code behind this app.

For our name, type in `ePNS Portal` and you can specify your own save location, however the default selected location is usually fine for most people and the project can be acessible from the welcome screen when you open Android Studio, or selecting File > Open Recent.

The language for this app will be written in Java. Press Next and you will be loaded into the IDE interface.

### Initial Setup

---

Allow Android Studio to finish indexing the library and the gradle for your project. The time needed for indexing to complete will vary on your hardware.

On the left hand side, you will see the folder and file structure for your project. We need to amend this in order to create another additional page, to have two in total. One for the warning and connection screen and one for the WebView.

In the root `java` folder, there will be a secondary folder named `com.example.epnsportal`. Right click on this folder, select New > Activity > Basic Activity.

Now, we just need to rename this new activity to `ePNSWebPortal`. Press Next.

The gradle will again resync.

### Building the First Page UI

---

We will be working on `activity_main.xml`. If this is not on the file access toolbar, go onto the left file browser, res > layout, and double click this to open the file.

In Android Studio, .xml files provide to build the user interface. Here we will add text and buttons to help our user navigate the app.

First, make sure you are on the Code or Split view, which you can change in the top right hand corner.

Now, we will need to remove some code Google has provided for us; this is not needed. Remove this code:

```jsx
<com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/Theme.MyApplication.AppBarOverlay">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:popupTheme="@style/Theme.MyApplication.PopupOverlay" />

    </com.google.android.material.appbar.AppBarLayout>

    <include layout="@layout/content_main" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|end"
        android:layout_margin="@dimen/fab_margin"
        app:srcCompat="@android:drawable/ic_dialog_email" />
```

Now we can add add UI elements replacing the code we just removed. The structure of each elements containing a block defining the UI element, and parameters we can set to customise them. For some, we will need to set an id (identifier) for that element, so constraints can be set referring to these identifiers. This allow the app to adapt for all screen sizes. In the same location, add in the UI elements below.

```jsx
<TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="CAUTION: Do not attempt to operate this app in an unsafe environment."
        android:textColor="#d50000"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="0dp"
        android:padding="50px"
        android:id="@+id/notice"/>
<TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Connecting to ePNS Portal"
        android:textAppearance="@style/TextAppearance.AppCompat.Medium"
        android:layout_below="@id/notice"
        android:id="@+id/headline"
        android:layout_marginLeft="50px"/>
<TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="1. Open your device's settings app\n \n2. Open the WiFi connection menu\n \n3. A list of discoverable networks is shown. Find and connect to 'ePNS-Portal'\n \n4. Once connected, return to the ePNS Portal app and press CONFIRM\n \n5. You are now connected to the ePNS Portal. Submit SOS forms and keep updated with governmemt advice."
        android:layout_centerVertical="false"
        android:layout_centerHorizontal="false"
        android:layout_below="@id/headline"
        android:padding="50px"
        android:id="@+id/textView"/>
<TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="192.168.199.1/form.html"
        android:layout_above="@id/confirmbutton"
        android:layout_centerHorizontal="true"
        android:padding="25px"
        android:textAppearance="@style/TextAppearance.AppCompat.Caption"/>
<Button
        android:id="@+id/confirmbutton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="CONFIRM"
        android:backgroundTint="@color/purple_500"
        android:layout_marginLeft="50px"
        android:layout_marginRight="50px"
        android:layout_marginBottom="50px"
        android:layout_alignParentBottom="true"/>
```

You can see a preview of your app beginning to build up. The full code for `activity_main.xml` is below:

```jsx
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="CAUTION: Do not attempt to operate this app in an unsafe environment."
        android:textColor="#d50000"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="0dp"
        android:padding="50px"
        android:id="@+id/notice"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Connecting to ePNS Portal"
        android:textAppearance="@style/TextAppearance.AppCompat.Medium"
        android:layout_below="@id/notice"
        android:id="@+id/headline"
        android:layout_marginLeft="50px"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="1. Open your device's settings app\n \n2. Open the WiFi connection menu\n \n3. A list of discoverable networks is shown. Find and connect to 'ePNS-Portal'\n \n4. Once connected, return to the ePNS Portal app and press CONFIRM\n \n5. You are now connected to the ePNS Portal. Submit SOS forms and keep updated with governmemt advice."
        android:layout_centerVertical="false"
        android:layout_centerHorizontal="false"
        android:layout_below="@id/headline"
        android:padding="50px"
        android:id="@+id/textView"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="192.168.199.1/form.html"
        android:layout_above="@id/confirmbutton"
        android:layout_centerHorizontal="true"
        android:padding="25px"
        android:textAppearance="@style/TextAppearance.AppCompat.Caption"/>
    <Button
        android:id="@+id/confirmbutton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="CONFIRM"
        android:backgroundTint="@color/purple_500"
        android:layout_marginLeft="50px"
        android:layout_marginRight="50px"
        android:layout_marginBottom="50px"
        android:layout_alignParentBottom="true"/>
</RelativeLayout>
```

### Coding the First Page

---

Now move over to the `MainActivity.java` file.

Now from now on, Android Studio will prompt you with errors, but do not worry. Once this is over, all the variables and functions will be declared and used and remove these errors.

Now, coding in Java becomes harder when defining functions. Below is the code for `MainActivity.java`. Essentially this is using the identifier `confirmbutton`, which is the button on the first page to set an action of redirecting the second page (ePNSWebPortal), which we will get onto later:

```jsx
package com.example.epnsportal;

import androidx.appcompat.app.AppCompatActivity;

import android.content.Intent;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;

public class MainActivity extends AppCompatActivity {
    private Button confirmbutton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        confirmbutton = (Button) findViewById(R.id.confirmbutton);
        confirmbutton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                openActivity2();

            }
        });
    }

    public void openActivity2() {
        Intent intent = new Intent(this, ePNSWebPortal.class);
        startActivity(intent);
    }
}
```

### Building the UI for ePNSWebPortal

---

Now we move over to `activity_e_p_n_s_web_portal.xml`. Here we need to create a WebView and accompanying parameters for the floating action button. However, again, we need to remove some code:

```jsx
<com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/Theme.MyApplication.AppBarOverlay">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:popupTheme="@style/Theme.MyApplication.PopupOverlay" />

</com.google.android.material.appbar.AppBarLayout>
```

Once removed, we need add in the WebView, replacing the code above.

```jsx
<WebView
        android:layout_width="match_parent"
        android:laout_height="match_parent"
        android:id="@+id/webView"
        />
```

Now we add in additional parameters for the floating action button. We remove the existing `app:srcCompat` , replacing this with 3 lines of code:

```jsx
app:srcCompat="@android:drawable/ic_menu_help"
app:backgroundTint="@color/purple_500"
app:tint="@color/white"/>
```

The whole of `activity_e_p_n_s_web_portal.xml` looks like this:

```jsx
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ePNSWebPortal">

    <WebView
        android:layout_width="match_parent"
        android:laout_height="match_parent"
        android:id="@+id/webView"
        />

    <include layout="@layout/content_e_p_n_s_web_portal" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|end"
        android:layout_margin="@dimen/fab_margin"
        app:srcCompat="@android:drawable/ic_menu_help"
        app:backgroundTint="@color/purple_500"
        app:tint="@color/white"/>

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

### Coding the Second Page

---

Now we move onto `ePNSWebPortal.java`. Here, we link what we specified in `[MainActivity.java](http://mainactivity.java)` back here and add the same actions for the floating action button, to redirect the user back to the first page. Here is the code for `ePNSWebPortal.java;`

```jsx
package com.example.epnsportal;

import android.content.Intent;
import android.os.Bundle;

import com.google.android.material.floatingactionbutton.FloatingActionButton;
import com.google.android.material.snackbar.Snackbar;

import androidx.appcompat.app.AppCompatActivity;
import androidx.appcompat.widget.Toolbar;

import android.view.View;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import android.widget.Button;

public class ePNSWebPortal extends AppCompatActivity {
    private com.google.android.material.floatingactionbutton.FloatingActionButton fab;
    WebView webView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_e_p_n_s_web_portal);

        fab = findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                openActivity1();

            }
        });

        webView = findViewById(R.id.webView);
        webView.setWebViewClient(new WebViewClient());
        webView.loadUrl("http://192.168.199.1/form.html");

        WebSettings webSettings = webView.getSettings();
        webSettings.setJavaScriptEnabled(true);

    }

    @Override
    public void onBackPressed() {
        if(webView.canGoBack()) {
            webView.goBack();
        }else{
            super.onBackPressed();
        }
    }

    public void openActivity1() {
        Intent intent = new Intent(this, MainActivity.class);
        startActivity(intent);
    }
}
```

### Changing Internet Permissions

---

Finally, we need to gain permission to use internet, in order to access the ePNS-Portal. Head over to  `AndroidManifest.xml`. Above `<application`, add this line of code in:

```jsx
<uses-permission android:name="android.permission.INTERNET" />
```

We also need to add compatibility for fonts used on the website. Add this line above `<activity:`

```jsx
android:usesCleartextTraffic="true">
```

The code for `AndroidMainifest.xml` is below:

```jsx
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.epnsportal">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.EPNSPortal"
        android:usesCleartextTraffic="true">
        <activity
            android:name=".ePNSWebPortal"
            android:label="@string/title_activity_e_p_n_s_web_portal"
            android:theme="@style/Theme.EPNSPortal.NoActionBar"></activity>
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

### ePNS Responder

---

Yay, hopefully you have have a functioning ePNS Portal Android app. Now for ePNS Responder, we will create a new project, and repeat exactly the same steps. However the code above includes code identifiers specific to that project, so below is the list of code for the next project, assuming you have correctly named the project `ePNS Responder` and the new activity is called `ePNSWebResponder`. The new code already includes the new URL to point users to the ePNS Responder website.

`activity_main.xml`

```jsx
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="CAUTION: Do not attempt to operate this app in an unsafe environment.\n \nNote this app 'ePNS Responder' is only to be used by responders. You will require a login from your designated dispatcher."
        android:textColor="#d50000"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="0dp"
        android:padding="50px"
        android:id="@+id/notice"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Connecting to ePNS Responder"
        android:textAppearance="@style/TextAppearance.AppCompat.Medium"
        android:layout_below="@id/notice"
        android:textColor="@color/black"
        android:id="@+id/headline"
        android:layout_marginLeft="50px"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="1. Open your device's settings app\n \n2. Open the WiFi connection menu\n \n3. A list of discoverable networks is shown. Find and connect to 'ePNS-Portal'\n \n4. Once connected, return to the ePNS Responder app and press LOGIN\n \n5. You are now connected to the ePNS Responder."
        android:layout_centerVertical="false"
        android:layout_centerHorizontal="false"
        android:layout_below="@id/headline"
        android:padding="50px"
        android:id="@+id/textView"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="192.168.199.1/admin.php"
        android:layout_above="@id/confirmbutton"
        android:layout_centerHorizontal="true"
        android:padding="25px"
        android:textAppearance="@style/TextAppearance.AppCompat.Caption"/>
    <Button
        android:id="@+id/confirmbutton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="LOGIN"
        android:backgroundTint="@color/purple_500"
        android:layout_marginLeft="50px"
        android:layout_marginRight="50px"
        android:layout_marginBottom="50px"
        android:layout_alignParentBottom="true"/>
</RelativeLayout>
```

`MainActivity.java`

```jsx
package com.example.epnsresponder;

import androidx.appcompat.app.AppCompatActivity;

import android.content.Intent;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;

public class MainActivity extends AppCompatActivity {
    private Button confirmbutton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        confirmbutton = (Button) findViewById(R.id.confirmbutton);
        confirmbutton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                openActivity2();

            }
        });
    }

    public void openActivity2() {
        Intent intent = new Intent(this, ePNSWebResponder.class);
        startActivity(intent);
    }
}
```

`activity_e_p_n_s_web_responder.xml`

```jsx
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ePNSWebPortal">

    <WebView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/webView"
        />

    <include layout="@layout/content_e_p_n_s_web_responder" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|end"
        android:layout_margin="@dimen/fab_margin"
        app:srcCompat="@android:drawable/ic_menu_help"
        app:backgroundTint="@color/purple_500"
        app:tint="@color/white"/>

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

`ePNSWebResponder.java`

```jsx
package com.example.epnsresponder;

import android.content.Intent;
import android.os.Bundle;

import com.google.android.material.floatingactionbutton.FloatingActionButton;
import com.google.android.material.snackbar.Snackbar;

import androidx.appcompat.app.AppCompatActivity;
import androidx.appcompat.widget.Toolbar;

import android.view.View;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import android.widget.Button;

public class ePNSWebResponder extends AppCompatActivity {
    private com.google.android.material.floatingactionbutton.FloatingActionButton fab;
    WebView webView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_e_p_n_s_web_responder);

        fab = findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                openActivity1();

            }
        });

        webView = findViewById(R.id.webView);
        webView.setWebViewClient(new WebViewClient());
        webView.loadUrl("http://192.168.199.1/admin.php");

        WebSettings webSettings = webView.getSettings();
        webSettings.setJavaScriptEnabled(true);

    }

    @Override
    public void onBackPressed() {
        if(webView.canGoBack()) {
            webView.goBack();
        }else{
            super.onBackPressed();
        }
    }

    public void openActivity1() {
        Intent intent = new Intent(this, MainActivity.class);
        startActivity(intent);
    }
}
```

`AndroidManifest.xml`

```jsx
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.epnsresponder">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.EPNSResponder"
        android:usesCleartextTraffic="true">
        <activity
            android:name=".ePNSWebResponder"
            android:label="@string/title_activity_e_p_n_s_web_responder"
            android:theme="@style/Theme.EPNSResponder.NoActionBar"></activity>
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

### Additional Guidance

---

You can find all code hosted on our GitHub repository.

[bluescreened803/ePNS](https://github.com/bluescreened803/ePNS)