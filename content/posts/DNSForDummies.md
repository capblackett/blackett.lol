+++
title = 'DNS For Dummies'
date = 2024-05-14T11:22:26Z
draft = true
+++

## About {#About}

A (somewhat) brief overview of the domain name system. It's worth knowing on a deeper level just how we can get to `www.example.com` without knowing the IP address of the web server delivering the page.

## Contents {#contents}
- [1 - The Domain Name System](#dns1)
- [2 - Querying A DNS Server](#dns2)
- [3 - Types of DNS Server](##dns3)
- [4 - DNS Record Propagation](#dns4)
- [5 - Querying A DNS Server (Again)](#dns5)
- [6 - Types Of DNS Record](#dns6)
- [7 - Closing Thoughts](#dns7)
- [8 - Advanced Reading](#dns8)

---

## DNS Overview {#dns1}

### The Domain Name System

As you may already know, computers communicate over networks using IP addresses to identify the specific device they are sending traffic to. In the case of loading a web page your computer will send a request to a web server for the HTML document. But rather than typing out the IP address we use a human friendly domain name instead. But how does your computer know that `www.example.com` corresponds to 1.2.3.4? This is what the domain name system (DNS) is for.

DNS servers operate similarly to a phone book by mapping web addresses to IP addresses so we don't have to remember an endless number of IPs. When you type a website into your browser's address bar your computer will send a query to the DNS server it has been configured to use. Do you remember configuring a DNS server for your computer? Probably not, this is part of the information that is given to a device via DHCP.

[_^top_](#About)

---

### Querying A DNS Server {#dns2}

When I said your computer sends a request to a DNS server that's not quite correct. The first thing your browser does is check its own cache of DNS mappings. If we had to run a query every time we went to www.google.com we'd be sending a _lot_ of unnecessary traffic. It's highly unlikely that Google's servers will change IP between us clicking on "maps" and "images", so our computers can remember that google.com is on 1.2.3.4. This is called caching. So the first step is to see if we 'remember' the IP address by checking the DNS cache. If there's no match then we can start sending DNS queries.



Chances are your computer does not ask a DNS server directly for domain to IP mappings, especially if it's been configured via DHCP. In most home networks the devices are set with the home router as the DNS server. Much like your computer, the router doesn't want to keep sending unnecessary traffic so it has its own little cache of DNS records. If no matching record is found then the router will forward the query on your computer's behalf. Typically a home router will pass the query to a DNS server at your ISP, though you are often able to configure which server it will query. And, if you hadn't already guessed it, the DNS server at the ISP will have a cache of records too.

[_^top_](#About)

---

### Types of DNS Server {#dns3}

The types of 'servers' mentioned so far (home router, ISP DNS server) are called _recursive_ DNS servers, sometimes called _resolver_ DNS servers. They don't have any records of their own, but they can hold cached copies of records they get from other DNS servers. If a query comes in for a record they don't have they can look it up to cache it and return the desired IP address. To keep the cache updated the recursive DNS server only holds the cached records for a limited time. This is called the _time to live_ (TTL). The TTL is how long the server stores the cached record and acts like a countdown. Once it has expired the server will send out a query for the record again to 'top it up'. If no records are found then the cached entry is removed.

But if recursive DNS servers don't have their own records then where do DNS records come from?

The other type of DNS server is an _authoritative_ DNS server. These are the big boys of the internet who have the final authority over domains. When you buy a domain name from a registrar you will need to set up DNS records with an authoritative DNS server in order to map your domain to an IP address. It's from these authoritative servers that the recursive servers get answers to the queries they are sent.

[_^top_](#About)

---

### DNS Record Propagation {#dns4}

Once a record has been entered into an authoritative DNS server it takes a little time to get passed onto the "next step down" recursive DNS servers. This process of spreading the word about the new DNS records is called DNS propagation and can take up to 48 hours. My personal experience has been that it's never taken more than 45 minutes. With modern DNS services like Amazon's Route 53 their "DNS Server" is actually a series of servers all around the world in data centres connected with high speed lines. Cloudflare and Google will use something similar.

[_^top_](#About)

---

### Querying A DNS Server (Again) {#dns5}

So now the DNS records have been filed with an authoritative server and propagated around the world we can go through the querying process again:

 - You type www.example.com into your browser
 - Your browser checks its cache, no record found
 - Your computer sends the query to your home router
 - Your home router checks its cache, no record found
 - Your home router sends the query to the ISP's DNS server
 - The ISP's DNS server checks its cache, no record found
 - The ISP's DNS server sends a query either to an authoritative DNS server or another recursive server, which will check its cache and if nothing is found send a query either to an authoritative DNS server or another recursive server, which will check _its_ cache etc.
 - Eventually either a recursive server has a cached record or an authoritative server replies with a record
 - The record is passed back down the chain to your computer
 - The computer sends a request to the IP address to ask for an HTML file
 - The web server can't find the HTML file request and replies with a 404 error message. That's just a joke, don't worry. Your Roblox fanfic website will still be accessible.

So now you should have a rough idea of how a computer can find the IP address for a web server it's never spoken to before. Phew, that's a lot of steps! No wonder everyone is caching records to save time. This is why web pages can sometimes take a little bit longer to load up when you first go to them. All of that querying happens in the space of a few seconds, which absolutely blows my mind.

[_^top_](#About)

---

### Types of DNS Record {#dns6}

So what are DNS records? These are the entries in a DNS server that are used to send responses when queries are received. So far we've only talked about mapping IP addresses to domain names - this is an **A** record. Let's take a look at some other record types and examples of each.

#### A Record
Maps a domain name to an IPv4 address. Query comes in with a domain name, reply is sent with an IPv4 address.

```dns
example.com A 1.2.3.4
```

#### AAAA Record
Maps a domain name to an IPv6 address. As above.

```dns
example.com AAAA 2001:db8::1
```

#### CNAME Record
Maps one domain name (alias) to another (canonical name). This is useful when running many services from the same IP address (eg, web, FTP, SFTP). So you could have ftp.example.com and www.example.com mapped to example.com and this in turn could have an A record mapping it to your server's IP address. If this IP address ever changes only one record needs updated (the A record for example.com).

```dns
www.example.com CNAME example.com
```
Here we have `www.example.com` mapped to `example.com`, which in turn will have an A record to point to a web server IP address. Both URLs point to the same IP in this case.

```dns
oldcompany.com CNAME newcompany.com
```
In this example a company has changed name so users going to the old web address will be sent to the new web address (which in turn will have an A record for the IP address of the web server)

#### MX Record
Used to specify a mail server is at a domain (MX = mail exchange). This is used to send/receive e-mails. The record points to a domain name which in turn will have an A record to point to the IP address of the mail server. As there can be multiple mail servers a priority system is used with a higher number indicating higher priority and vice versa.

```dns
example.com MX 10 mail.example.com
```
`mail.example.com` would have an A record for the IP address of the mail server

#### TXT Record
Used to hold text and associate it with a domain, as the name suggests. This can be used to verify that you own a domain with a third party application. For example, an e-mail marketing platform could give you a verification code to enter into a TXT record on an authoritative DNS server.

```dns
example.com TXT "website-verification=ABCD1234XYZ987"
```
In this example the web service will run a DNS query to check what TXT records are associated with the domain name. If it sees a matching code then it knows you are the owner of this domain.

#### NS Record
Specifies the name server responsible for the domain. This is the authoritative DNS server that DNS queries for a given domain will be sent to.

```dns
example.com NS nameserver.com
```
When a DNS server that holds this record receives a query for example.com it will respond telling the device doing the querying to send it to nameserver.com

[_^top_](#About)

---

### Closing Thoughts {#dns7}

Hopefully now you've got a better idea of what DNS is. It's used _everywhere_ in tech. For example, at my place of work we rarely use the IP addresses of our 900+ servers. Trying to remember this many IP addresses would drive a man insane. Instead we have our own DNS server set up to map IP addresses in the private 10.0.0.0/8 range to human readable names.

You have no idea what 10.1.2.3 would be if you'd never seen it before but you can tell at a glance that SITE1-DOMAINCONTROLLER.business.co.uk is going to be a domain controller at site 1 of our campus. Whilst that looks like a website address it's only going to return a website if SITE1-DOMAINCONTROLLER is running a web server. I can tell you with authority that none of our domain controllers run web server software. That would be mad. It's also not accessible outside of our network as the IP address it maps to is not routable on the public internet.

If you have any questions about DNS or you'd like to issue a correction just head on over to [NetworkChuck's community Discord server](http://bit.ly/nc-discord ) and ask away.

[_^top_](#About)

---

## Advanced Reading {#dns8}

### DNS Hierarchy

The way DNS resolves a fully qualified domain name (eg, `blog.blackett.lol`) is a little more nuanced that simply finding a match in a server's cached records. The lookup process starts from the rightmost part of the FQDN and works to the left. So the first part of the FQDN would be the `.lol`, which is called a _top-level domain_ (TLD). TLD servers are the authoritative servers that are responsible for managing the domain name space under the TLD they govern.

So our query begins by having a resolver contact a root DNS server which will tell us the IP address for the TLD server that manages `.lol` domains. From here, the TLD server will tell us the address for the authoritative name server that has records for `blackett.lol`. We then query this authoritative name server for the `blackett.lol` records (eg, an A record) and it tells us the IP address of the web server that has the HTML document we want.

Phew, it's an even more complicated process than I explained earlier. Fortunately for us there is a lot of caching at all steps along the way so we don't have to go through the entire process every single time we want to watch basket weaving tutorials on YouTube.

[_^top_](#About)

---

### DNS In The Cloud

The abstraction DNS provides is utilised within cloud providers like AWS as a foundational component of their internal networking. This can be a little tricky to grasp unless you already have a solid understanding of DNS and how it's used. The gist of it is that services call upon domain names for interactivity instead of IP addresses as this allows for scaling and availability without the worry of low level back-end networking configuration.

The advantage of cloud platform as a service offerings is that the work we do is abstracted above the OS, computer, networking etc. resources. By using API endpoints for communication between services all of that back end stuff can be chopped and changed for scaling and availability purposes without impacting our applications. We don't need to specify the IP addresses of new servers in our availability pool, we just specify the fully qualified domain name of the availability pool itself.

[_^top_](#About)

---

### DNS Security 

As the DNS system relies on trusting the authoritative, root, and TLD servers we need to take measures to protect the DNS infrastructure from spoofing, cache poisoning, DDoS attacks, and other malicious activity. One method for combatting harm is DNSSEC - Domain Name System Security Extensions. DNSSEC is a suite of extension to DNS that adds a variety of features to address vulnerabilities in traditional DNS infrastructure.

For example, DNSSEC uses cryptographic keys and digital signatures to sign DNS records, allowing a zone's owner to 'stamp' their private key on a record to prove ownership. A corresponding public key is published in a DNSKEY record, this is used by DNS resolvers to verify the authenticity of the signed records.

[_^top_](#About)

