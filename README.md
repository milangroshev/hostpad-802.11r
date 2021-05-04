# Hostpad 802.11r roaming with FT
This repo contains Configuration of virtual AP network based on hostapd with 802.11r to allow testing fast transition (FT) clients.

A WiFi client is usually connected to an AccessPoint for network access. When the AccessPoint becomes unreachable for example as the client moved, the client needs to switch over to an other AccessPoint providing the same extended service set (ESSID). This is called roaming. Using IEEE 802.11r the client can roam between access points in almost real-time. This repo contains all the configuration files to set up a radius server with two virtual AP and a client that is able to perform fast transition between the APs. This scenario has been tested with remotly controlled mobile robot (Turtlebot 2) and the robot was able to perform roaming without the interuption of service. 

The normal roaming involves the following steps for the client:
 - Scanes for other reachable AP
 - Disconnects from the old AP
 - Changing the Wi-Fi channel
 - Autentication adn Association
 - WPA-EAP-Handshake ( for 802.1X)
 - 4-way WPA handshake
 - Connection is ready

Hoever, this romaing requires for the client to disconnect and this means usually interuption of service. With, PSK-Authentification, 802.11i and 802.11r things can be spped up and the down time can be significanlty decreased (to 40 ms). This means:
- PSK-Authentification does not need the EAP Handshake.
- 802.11i pre-authentication allows the client to perform authentication with the new AP while still connected to the old AP
- 802.11r (fast transition) over-air performs the 4-way WPA handshake piggy-backed on the Authentication and Association frames and skips the EAP-Handshake.
- 802.11r (fast transition) over-ds does the same as over-air but has the Authentication handshake (not association) performed while still connected to the old AccessPoint.

Our scenario involves the following steps:
- Radius server
- 2 hostapd based vAP with configured 802.11i and 802.11r
- Client

## Requirements
### Hardware
- 2 nodes (AP1 and AP2) with Wi-Fi interfaces (they can be also Wi-Fi USB Dongles ) that support 802.11r and 802.11i that will be used as access points.
- 1 node (Client) with Wi-Fi interface (they can be also Wi-Fi USB Dongles ) that support 802.11r and 802.11i that will be used as client.
### Software
- on AP1 and AP2 install hostapd 2.9. For debian based systems: `sudo apt get install hostapd`
- on AP1 install radius server and mysql. For debian based systems: 
  - `sudo apt get install mysql-server`
  - `sudo apt get install freeradius freeradius-mysql`

## Configuration
### Mysql (note: the MySql configuration is performed on AP1)
On AP1 where we installed radius and mysql we need to create the Database that freeradius will use. Execute the following sequnce of commands in AP1:
- `mysql -uroot -p`
- `CREATE DATABASE radius;`
- `GRANT ALL ON radius.* TO radius@localhost IDENTIFIED BY "radpass";`
- `exit`

Import the freeraduis mysql schema files in the database you created:
- `mysql -u root -p radius < /etc/freeradius/sql/mysql/schema.sql`
Note: Ubuntu schema files are not called mysql.sql, but schema.sql.

Next, you need to configure freeradius to use the mysql database. Edit the /etc/freeradius/radiusd.conf file (example [here](./freeradius/radiusd.conf)):
- uncomment `$INCLUDE  sql.conf`
Edit also the /etc/freeradius/sql.conf file (example [here](./freeradius/sql.conf)):
- set database = `"mysql"`


### Freeradius
On AP1 where freeradius is installed edit the /etc/freeradius/radiusd.conf file and after line 355 add (example [here](./freeradius/radiusd.conf)):
- radius server listening auth ip address and interface (in our example ip: 192.168.100.1 and interface: eno1)
```yaml
listen {
        
        type = auth
        ipaddr = 192.168.100.1   
        port = 0
        interface = eno1	

}
```
- radius server listening accounting ip address and interface (in our example ip: 192.168.100.1 and interface: eno1)
```yaml
listen {
        
        type = acct
        ipaddr = 192.168.100.1  
        port = 0
        interface = eno1	

}
```
- nesure that `$INCLUDE  sql.conf` is uncommented

On AP1 edit the /etc/freeradius/clients.conf file and add (example [here](./freeradius/clients.conf)):
- clients ip address pool and secret (in our example ip: 192.168.0.0/24 and secret: robotdemo)
```yaml
client 192.168.0.0/24 {

        secret      = robotdemo
        require_message_authenticator = no
        nastype     =
}
```

On AP1 edit the /etc/freeradius/users file and add (example [here](./freeradius/users)):
- The username and password of the client that will roam (this info will be later also configured in the wpa_supplicant)
  - `robot01 Cleartext-Password := "robotdemo"`
