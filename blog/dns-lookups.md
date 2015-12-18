# A developer's guide to DNS queries with dig

<?image of phonebook?>
![alt text](/images/chrome_dev_session.png)

The DNS protocol can be tricky to understand. Because of its recursive nature and its heavy reliance on caching, it can be difficult to understand what is happening behind the scenes. Updating a DNS record can leave you scratching your head when the name isn't resolving in the way you think it should. This post should help clarify with some background and give you the necessary tools to diagnose yourself.

Dig stands for domain information groper. Seriously, I'm not making it up.

## A little background

DNS' job is to resolve names into ip addresses. It exists because a it's tough for humans to memorize something like "170.149.172.130", but easy to remember the domain name that it is associated with the address (in this case, nytimes.com). A common analogy used to describe DNS is that it is like a phonebook for the internet. But unlike the the phonebooks of yore, this one can be updated by the individual owners without requiring the entire book to be reprinted. Let's dive into how this works...

## DNS Architecture

DNS can be thought of as a hierarchical collection of zones. Let's take yahoo's fantasy basketball address as an example:

basketball.fantasysports.yahoo.com

In order for DNS to figure out what the ulitmate ip is for that domain name, it will need to recurse down the zones until it finds an authoritative answer. In this example, we need to go through 4 zones before we get to the ultimate answer:

```
. <-- root zone, all addresses start here
.com <-- the .com zone
.yahoo <-- yahoo's zone record
.fantasysports <-- yahoo has chosen to have a sub-domain with its own zone record for fantasy sports
```

After traversing through these zones, we can get an authoritative answer that basketball.fantasysports.yahoo.com is actually a [CNAME](http://en.wikipedia.org/wiki/CNAME_record) (for those not familiar, think of it as an alias) for real.fantasysports.a03.yahoodns.net and that server's ip is 216.39.54.78 (as of Feb. 22, 2014). 

<image here of zone hierarchy?>

You can verify this yourself by using the `ping` command from your terminal:

```bash
coldwd@muir:~> ping basketball.fantasysports.yahoo.com
PING real.fantasysports.a03.yahoodns.net (216.39.54.78): 56 data bytes
64 bytes from 216.39.54.78: icmp_seq=0 ttl=51 time=63.516 ms
64 bytes from 216.39.54.78: icmp_seq=1 ttl=51 time=47.648 ms
64 bytes from 216.39.54.78: icmp_seq=2 ttl=51 time=104.893 ms
```

The output from the `ping` command shows the real machine name (real.fantasysports.a03.yahoodns.net) and its corresponding ip address (216.39.54.78). All of the hard work done by DNS was handled for you behind the scenes. This is often the case with DNS, most end-users never need to know of its existence. It just works.

In order to visualize what is going on behind the scenes, we'll need to use a tool called `dig`.

## The dig command

<image of digdug?>

The dig command is part of the BIND domain name server software suite and is automatically installed on Linux and OS X machines. You should be able to type:

```
$ dig basketball.fantasysports.yahoo.com
```

...and get output looking something like:

```
; <<>> DiG 9.8.3-P1 <<>> basketball.fantasysports.yahoo.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16090
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;basketball.fantasysports.yahoo.com. IN	A

;; ANSWER SECTION:
basketball.fantasysports.yahoo.com. 84 IN CNAME	real.fantasysports.yahoo.com.
real.fantasysports.yahoo.com. 95 IN	CNAME	real.fantasysports.a03.yahoodns.net.
real.fantasysports.a03.yahoodns.net. 26	IN A	216.39.54.78

;; Query time: 126 msec
;; SERVER: 192.168.42.1#53(192.168.42.1)
;; WHEN: Sun Feb 23 18:32:14 2014
;; MSG SIZE  rcvd: 136
```

Okay, now were getting a little more context around the how we arrived at the 216.39.54.78 ip address. Let's break down each section and explain what's happening:

<screen shot of each section with annotations?>

You might have noticed that the dig output did not go through all of the zones I had listed above in order to get an answer. Instead, it immediately returned the CNAME record. Why is this? The answer is lies in the way DNS uses caching.

## DNS Caching

In order to improve performance, DNS supports caching at the nameserver level. What's happening with my request for basketball.fantasysports.yahoo.com can be told in that very last section of dig output:

```
;; Query time: 126 msec
;; SERVER: 192.168.42.1#53(192.168.42.1)
;; WHEN: Sun Feb 23 18:32:14 2014
;; MSG SIZE  rcvd: 136
```

The server that was responsible for responding was 192.168.42.1. This is actually my local ISP's very own DNS server. Almost all ISPs do this. They will keep a local cache of all requested DNS records for their customers. This can speed things up considerably. Instead of hitting several servers across all of the zones to piece together the ip, you just hit one server and get a response back almost instantaneously (126ms in our example).

## Time-to-live

What happens when things change? Say yahoo decides to upgrade its webservers for basketball.fantasysports.yahoo.com and the machine has a new ip address. How will the ISP's DNS cache know to update? 

An important concept in DNS is **time-to-live**, back to the example we've been using in our example. Notice that there is a three digit number next to each record in the answer section:

<image annotating the ttl portion?>

That number counts down to zero and once it does, it will make a full request to an authoritative nameserver and cache the result anew. You can see the countdown happen for yourself. Run the same dig query twice after waiting 5 seconds. Here's what my output looks like:

<image annotating the ttl portion?>

My ISP sets the TTL for its records at 300s. That means that a record will take *at most* 5 minutes to be refreshed in the local cache. However, TTLs are in effect at all levels of the DNS zones. So although my local cache checks every 300s for changes, zone records could have longer TTLs so it will take longer to flow down. This is a good time to revisit the zone flow...

## Root Zones

In order to get a more complete picture of what is going on, we'll need to query a nameserver that doesn't already have basketball.fantasysports.yahoo.com cached. dig allows you to directly specify the nameserver you'd like to use for the lookup using the "@" directive. Here's a dummy example:

```
$ dig @my.nameserver.org basketball.fantasysports.yahoo.com
```

Since we want to get a complete view from start to finish, we're going to pass a **root nameserver** to the dig command. A little bit of interesting internet trivia for you to learn. The entire hierarchy of the internet is managed by 13 root nameserver clusters. These have a special designation of using the "root-domain.net" domain. The clusters are managed by the big players of the internet (Verisign, ICANN, DoD, etc.), the first cluster is named a.root-domain.net, the second is b.root-domain.net, and the last one is m.root-domain.net. Each of those clusters have many servers responding to requests and they are spread out across the globe. As of this blog posting, there are 386 total root servers.

So, let's pretend for a second that the cache server didn't have a record of basketball.fantasysports.yahoo.com. How would it figure out what the ip address is? It has to start somewhere, and that would be with one of the root nameservers. The server would ask one of the root nameservers (let's pick g.root-server.net)


## Full picture of zones using +trace

So far we've only used of the simplified version of the dig command. This is fine for finding a quick answer on what ip address an endpoint is using. There are many options that the dig command accepts. It's worth checking the man page quickly to get a quick overview on all the options at your disposal.

An important 

## A note on browsers

why you can't just type the ip address into a browser
host: header

example, attitash / virtualhosts / http://crotchedmountain.com/

chrome://net-internals/#dns
ModHeader chrome extention

## Other notes

53 is the default port for DNS, so when you see: 192.168.19.2#53 it's specifying the port number, confusing, right?

There are 13 root nameserver **clusters**. These are designated as [a-m].root-servers.net. Those 13 clusters are distributed across 386 (as of blog post date) actual servers. They are all over the globe.
http://www.root-servers.org/ is a really good resouce to visualize this


## Additional Resources

* http://en.wikipedia.org/wiki/Domain_Name_System
* http://en.wikipedia.org/wiki/Dig_(command)
* http://computer.howstuffworks.com/dns.htm
* http://en.wikipedia.org/wiki/Virtual_hosting
* http://superuser.com/questions/477314/how-do-dns-servers-work
* http://www.thegeekstuff.com/2012/02/dig-command-examples/
* http://serverfault.com/questions/179630/how-can-i-see-time-to-live-ttl-for-a-dns-record
* http://apple.stackexchange.com/questions/80158/dig-not-returning-authority-section
* 

## Questions for Anki

* What is default port for dns?
* How does TTL work in DNS?
* How many root servers are there?
