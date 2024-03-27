# Increase Apache TCP Throughput

The default configuration of Apache/Nginx and server network TCP settings do not allow high throughput causing dropped network packets under heavy load. These dropped packets correspond to a 'Connection timed out' and/or 'Connection reset by peer' error on clients. The following is a guide of how to diagnose this issue and increase TCP throughput using Apache as the example web server.

## Diagnosis

Assuming a 'Connection timed out' and/or 'Connection reset by peer' error on a client, the following can be used to diagnose whether the server is under-configured to handle heavy loads. All commands are done on the server (not client).

Run this command `watch -n 2 -t -d 'pgrep apache2 | wc -l';`, if the number is 151, this means that the one parent process and the default 150 Apache worker processes are maxed out. This means that current inbound requests on port 80 are probably getting backlogged. 

Run this command to see how many port 80 LISTEN connections are getting backlogged (queued, unacknowledged) `watch -n 2 -t -d "ss -ltni '( sport = :80 )'";`. If the 'Recv-Q' number ever exceeds the 'Send-Q' number, the queue is completely full and packets are getting dropped, this would cause a 'Connection timed out' and/or 'Connection reset by peer' error on the client. Such requests die silently and are not very helpful to the client.

Run this command to see how many times this LISTEN queue overflowed since last server reboot `watch -n 2 -t -d "netstat -st | grep -A 1 'times the listen queue of a socket overflowed'";`. Using the server's up time since last reboot can help you get an average of how many requests are getting dropped on a daily basis. With proper configuration changes, this number should not increase dramatically if at all, and a well configured server will have a 0.

## Tuning

If using Apache, verify which mpm is being used via `sudo apachectl -V;`. Configure the appropriate Apache file, in the case of the 'prefork' mpm `sudo nano /etc/apache2/mods-available/mpm_prefork.conf;`. The variables of interest are 'MaxRequestWorkers', 'ServerLimit', and 'ListenBacklog'. When exceeding certain values for 'MaxRequestWorkers', the 'ServerLimit' must be raised accordingly, and the 'ListenBacklog' will increase the size of the LISTEN queue to reduce dropped packets under heavy load or during Apache restarts. Be aware of available server resources when making any changes, always test the changes before making them in a production environment. The following is an example for a server requiring high throughput:

```
<IfModule mpm_prefork_module>
        StartServers             5
        MinSpareServers          5
        MaxSpareServers          10
        MaxRequestWorkers        1000
        MaxConnectionsPerChild   0
        ServerLimit              1000
        ListenBacklog            9000
</IfModule>
```

These changes will not automatically apply to the TCP configuration of a server, so the following must also happen. Create a new sysctl config file and increase the appropriate parameters `sudo nano /etc/sysctl.d/highthroughput.conf;`. Research/test all of the parameters before making any changes, the following is an example for high throughput:

```
# Increase network values to give higher concurrent TCP throughput
# Default 128
net.core.somaxconn = 10240
# Default 1000
net.core.netdev_max_backlog = 10240
# Default 1024
net.ipv4.tcp_max_syn_backlog = 65536
```

Then run `sudo sysctl -p /etc/sysctl.d/highthroughput.conf;` to apply these changes without a server reboot. Now restart Apache for all the Apache changes to take effect, it is preferable to perform this when server requests are low, `sudo service apache2 restart;`. Monitor the system for any performance issues and tweak the values if necessary. Decreasing any of the Apache values going forward should only require a `sudo service apache2 reload`.

## Post-diagnosis
If everything was done right, the command  `watch -n 2 -t -d 'pgrep apache2 | wc -l';` should now reflect a number up to 'MaxRequestWorkers' + 1 (parent process), but ideally all/most concurrent requests can be handled without exceeding this value. 

Under extreme cases the command `watch -n 2 -t -d "ss -ltni '( sport = :80 )'";` should reveal that the backlog queue will be utilized, but never exceeded. 

And the command `watch -n 2 -t -d "netstat -st | grep -A 1 'times the listen queue of a socket overflowed'";` should reveal that this number does not increase, meaning no new TCP requests on port 80 overflowed, thereby eliminating the connection errors clients used to get. Congratulations, let the requests flow!

## Helpful Links

https://wiki.archlinux.org/title/sysctl

https://engineering.chartbeat.com/2014/01/02/part-1-lessons-learned-tuning-tcp-and-nginx-in-ec2/

https://linux.die.net/man/7/tcp