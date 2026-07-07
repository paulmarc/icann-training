
<img src="https://github.com/yakanho/training/assets/54844453/321060e5-fc84-40f7-8caa-846d0a68494b" alt="ICANN" style="zoom:25%;" />

------

# Lab configure TSIG

```
Created by: Yazid AKANHO
Modified by: Yazid AKANHO
Current version: 2024102900
Previous version: 2024020800
```
------
# Introduction
Tsig KEY Base security

## Goals
Instead of using IP addresses, we'll now be using cryptographic keys to authenticate zone transfer – this uses TSIG, a mechanism by which the communication between the primary and secondary server will be authenticated using this key.

**==Note:==**
> Commands preceded with $ imply that you should execute the command as a general user - not as root.
Commands preceded with "#" imply that you should be working as root.
Commands with more specific command lines (e.g. “rtr7>" or “mysql>") imply that you are executing commands on remote equipment.


## On your primary server (SOA server)

Generate the tsig key 

```
$ sudo tsig-keygen -a hmac-sha256 grp7-key > /tmp/grp7-key.txt
```

Check the content of the file. Should look similar to this:

```
key "grp7-key" {
	algorithm hmac-sha256;
	secret "THIS_IS_MY_KEY";
}; 
```

Add the tsig key at the bottom of **named.conf.options** config file.

```
key "grp7-key" {
algorithm hmac-sha256;
        secret "THIS_IS_MY_KEY";
};
server 100.100.7.130 {
     keys {grp7-key ; };
};
server 100.100.7.131 {
     keys {grp7-key ; };
};
```

Don't forget to replace 7 in "**grp7-key**" ....!

Then in your zone, **change allow-transfer line** as shown below

```
zone "grp7.<lab_domain>.te-labs.training" {                                                                               
        type primary;                                                                                                  
        file "/var/lib/bind/zones/db.grp7";                                                                                     
        allow-transfer { key grp7-key; };
        also-notify { 100.100.7.130; 100.100.7.131; };                                                                                      
};
```

As you can see above, we've changed “allow-transfer” statement allowing transfer of the zone for holders of the "tsig-key".

Restart *named* service

```
$ sudo named-checkconf
$ sudo rndc reconfig
$ sudo systemctl status bind9
```

## On NS1 server

Test that zone transfer has stopped working.
```
$ dig @100.100.7.66 axfr grp7.<lab_domain>.te-labs.training

...
; Transfer failed.
```

A look into the SOA server logs should show something like:

```
$ tail /var/log/syslog

24-May-2022 10:03:29.433 client @0x7f185c006920 100.100.1.130#38993 (grp1.<lab_domain>.te-labs.training): zone transfer 'grp1.<lab_domain>.te-labs.training/A7FR/IN' denied
```

**We need the key!**

You can also test manually as follows:

```
$ dig @100.100.7.66 -y hmac-sha256:grp7-key:THIS_IS_MY_KEY axfr grp7.<lab_domain>.te-labs.training
```


### Add the TSIG key to your NS1 configuration

In **/etc/bind/named.conf.options**, add the tsig key, and a statement to tell which key to use when talking to “100.100.7.66;” (the soa server ):

```
key "grp7-key" {
        algorithm hmac-sha256;
        secret "THIS_IS_MY_KEY";
};

server 100.100.7.66 {		// here you put the IP of YOUR primary server (SOA)
        keys { grp7-key; };
};
```

Save, exit and **restart bind9**.

### Testing the configuration

On SOA server **increase the serial** and **reload BIND9 service**. 

In ns1, go to logs and validate that the transfer was successful.

```
$ tail /var/log/syslog

zone grp2.<lab_domain>.te-labs.training/IN: Transfer started.
transfer of 'grp2.<lab_domain>.te-labs.training/IN' from 100.100.2.66#53: connected using 100.100.2.13>
zone grp2.<lab_domain>.te-labs.training/IN: transferred serial 2022052401: TSIG 'grp2-key'
transfer of 'grp2.<lab_domain>.te-labs.training/IN' from 100.100.2.66#53: Transfer status: success
transfer of 'grp2.<lab_domain>.te-labs.training/IN' from 100.100.2.66#53: Transfer completed: 1 messag>
zone grp2.<lab_domain>.te-labs.training/IN: sending notifies (serial 2022052401)
managed-keys-zone: Key 20326 for zone . is now trusted (acceptance timer complete)
resolver priming query complete
```

## On NS2 server

Edit **/etc/nsd/nsd.conf** file, create key section and add the tsig key grp7-key

```
key:
        name: "grp7-key"
        algorithm: hmac-sha256
        secret: "THIS_IS_MY_KEY"
```
and Fix pattern:

change these lines

```
allow-notify: 100.100.7.66 NOKEY
request-xfr: A7FR 100.100.7.66 NOKEY
```
with this:

```
allow-notify: 100.100.7.66 grp7-key
request-xfr: A7FR 100.100.7.66 grp7-key
```


Save, exit, verify and **restart NSD service**.

```
$ nsd-checkconf /etc/nsd/nsd.conf
$ sudo nsd-control reconfig
$ sudo nsd-control reload grp7.<lab_domain>.te-labs.training
```

Check the logs on NS2 and on SOA.
