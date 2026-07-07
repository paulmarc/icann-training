
<img src="https://github.com/yakanho/training/assets/54844453/321060e5-fc84-40f7-8caa-846d0a68494b" alt="ICANN" style="zoom:25%;" />

------

# Lab configure your reverse DNS zone

```
Created by: Yazid AKANHO
Modified by: -
Current version: 2024020800
Previous version:-
```
------

<img src="https://github.com/yakanho/training/assets/54844453/d794aab6-720a-4802-86b5-6afea3032957" alt="lab_topology" style="zoom:50%;" />

**<u>Lab Topology (Group 7) </u>**

------


During this practice we are only going to access the following equipment:

* **grp7-cli** : client
* **grp7-soa** : hidden authoritative servers (primary)
* **grp7-ns1** & **grp7-ns2** : secondary authoritative servers

> [!WARNING]
>
> In all this lab, be carefull to always replace ***7*** by your Group number in IP addresses, server name and any other place where required. Same for <*dnsme*> to be replace by the domain name registered for the class.



## Configure the primary authoritative server (SOA)

To map your IP address to your domain name, we’ll need to setup a reverse zone. We are going to configure a hidden authoritative server for your reverse zone and create the authoritative zone reverse\_*grp7*.<*dnsme*>.te-labs.training.

```
$ sudo nano /var/lib/bind/zones/reverse_grp7.dnsme.te-labs.training
```


```
$TTL    300
@		IN		SOA		soa.grp7.dnsme.te-labs.training. dnsadmin.dnsme.te-labs.training. (                                            
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;

                IN              NS              ns1.grp7.dnsme.te-labs.training. ; your name server
                IN              NS              ns2.grp7.dnsme.te-labs.training. ; your name server
66		IN		PTR		soa.grp7.dnsme.te-labs.training.
67		IN		PTR		resolv1.grp7.dnsme.te-labs.training.
68		IN		PTR		resolv2.grp7.dnsme.te-labs.training.
130		IN		PTR		ns1.grp7.dnsme.te-labs.training.
131		IN		PTR		ns2.grp7.dnsme.te-labs.training.
```

Save and exit.

Run the following command to check for any errors in your setup:

```
$ named-checkzone 7.100.100.in-addr.arpa /var/lib/bind/zones/reverse_grp7.dnsme.te-labs.training
```

Next, edit the /etc/bind/named.conf.local file and add the following lines:

```
zone "7.100.100.in-addr.arpa" {
  type primary;
  file "/var/lib/bind/zones/reverse_grp7.dnsme.te-labs.training";
  allow-transfer { any; };
  also-notify {100.100.7.130; 100.100.7.131; };
};
```

Save and exit.

Run the following command to check for any errors in your setup:

```
# named-checkconf
```

Restart Bind 9 

```
$ sudo rndc reload
$ sudo systemctl status bind9
```

Test your reverse DNS using dig
```
$ dig -x 100.100.7.66 @localhost
```

Or

```
$ dig 66.7.100.100.in-addr.arpa. PTR @localhost
```


Question: Do you get a DNS response with the PTR record in the answer section?



## Configure the secondary authoritative servers (ns1 and ns2) 

These servers are the ones that expose our (reverse) zone publicly (so they will be open-to-all servers). You should now know how to configure the secondary NS. If you forgot, go back to the lab where you created the forward zone for your grp7 and follow the instructions to do this new configuration for your reverse zone.

Once you are done with configuration, test your reverse zone propagation.

## Test your zone configuration and propagation.
Use *dig* tool to verify your zone configuration and propagation, then do the same for one or two other groups in the class and share comments. From your client, run the following dig queries. All should return answer otherwise you should review your configurations before continiuing:

1. dig -x 100.100.7.66 @100.100.7.66
2. dig -x 100.100.7.66 @100.100.7.130
3. dig -x 100.100.7.66 @100.100.7.131
4. dig -x 100.100.7.67 @100.100.7.130
5. dig -x 100.100.7.68 @100.100.7.130
