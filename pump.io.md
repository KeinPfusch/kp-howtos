**Hardening of Pump.io**


pump.io is a nice federated social network. Unfortunately, it is developed the works-on-my-laptop way, so that it lacks 
of the very basic security good practices. First, developers are stating you cannot (or should not) put a reverse proxy 
in front of it. this is not good news, because a tree-tier architecture is a very basic good practice for security. Meaning,
no server providing the business logic should be directly exposed to the internet: the business logic should be hosted only in
behind of some separation element, which sanitizes the protocol.

Actually it is not true that pump.io can't work with a reverse proxy in front of it: it is just a matter to configure it properly.
I personally managed to configure caddyserver ( https://caddyserver.com/ ) to make use of ACME support and to sanitize 
the protocol. The problems which I faced were easy to solve, with the following configuration:

```
https://example.com https://www.example.com {
header / -Server
bind 9.8.7.6
	gzip
	tls your.mail@here

proxy / https://1.2.3.4:443 {
		    insecure_skip_verify
		    transparent
		    header_upstream Connection {>Connection}
                    header_upstream Upgrade {>Upgrade} 
		    }


timeouts {
	read   none 
	header none 
	write  none 
	idle   none 
}

```
This is enough to support the way pump.io is managing the websocket. 

*Warning*: for caddy, you need to disable http2 to make it work, invoking the server with *--http2=false* option


Then, now we have a second security bad practice to solve.

Pump.io wants to listen on port 443 for ssl, or will end in a endless loop of redirection in case of a reverse proxy. 
This is very common in software made by "works-on-my-laptop" developers, which are only aware of localhost and have 
no clue of security. Since they use to be the  admin of their laptops, they think it is ok to do the same on servers.
The answer to this concern is that pump.io "_can start as root, and then spawn childs owned by unprivileged users_".

Unfortunately, in this specific case this answer is not acceptable.Pump.io makes use of a jeopardy of JS libraries, each one 
developed by different parties. Each of those libraries may contain 0-day or security issues, which will reflect on the whole
platform. This answer was accepted in the past for code which was written by a single entity to address security issues too. The
heartbleed issue has made clear a single flawed library can impact whatever software using it. So the approach of "i start
as root and then spawn unprivileged processes" is not acceptable anymore.Full stop. It is prone to privilege escalation. I call BS
on that answer.

This approach is acceptable in amateur environments, next-corner-shops and self-hosted machines, and just because they don't 
have liability in case of hacking. But, we are lucky, and Linux offers the way for a process running in non-privileged way
to bind on port 443.

The way is : http://manpages.ubuntu.com/manpages/xenial/en/man1/authbind.1.html

Authbind can make your program to be able to open port 443 and 80 , without being run by root. Meaning, a privilege escalation
will gain no root privilege. 

Install authbind using your favorite package manager.

Configure it to grant access to the relevant ports, e.g. to allow 80 and 443 from all users and groups:

```
sudo touch /etc/authbind/byport/80
sudo touch /etc/authbind/byport/443
sudo chmod 777 /etc/authbind/byport/80
sudo chmod 777 /etc/authbind/byport/443
```

This will make the ports available.
Now execute your command via authbind (optionally specifying --deep or other arguments, see the man page):

```
authbind --deep /path/to/binary command line args
```
In the case of pump.io, on a debian machine

```
authbind --deep  node /somewhere/bin/pump -c /somewhere/pump.json
```

This will run pump.io, as the user which will invoke it, hopefully unprivileged. An attacker which'll be able to escalate 
privileges in some ways, will only get the privileges of the user which invokes "node". Please notice, -deep will extend the 
privileges to any subprocess. 

