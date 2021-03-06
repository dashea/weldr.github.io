---
layout: post
title: Finding Weldr On IRC
author: dcantrell
---

The Weldr project uses [IRC](https://en.wikipedia.org/wiki/Internet_Relay_Chat) to communicate with each other around the world.  We have team members in multiple countries and timezones and being available on a chat system is helpful.  We are on the [FreeNode](http://freenode.net/) network in the **#weldr** channel.

One of the limiting aspects of IRC is the difficulty to stay in the chat system while moving among computer systems and shutting systems down or rebooting them.  A helpful way to solve this is to run an IRC proxy or bouncer to keep you always connected.  You then configure your client to connect to your proxy rather than directly to FreeNode.  If this sounds useful to you, the steps below can help you get started.

**Requirements:**
1. [FreeNode nickname registered](http://freenode.net/kb/answer/registration)
2. A reliable always-on/always-connected Linux system to run the proxy on.
3. Patience

## Install Proxy

There are a handful of these available.  My host is a virtual server currently
hosted by [Linode](https://www.linode.com/) and running [CentOS](http://www.centos.org).  I will be installing the proxy server here.

For the proxy server I am using **bip**.  Bip seems to have fallen out of favor to things like [ZNC](https://wiki.znc.in/ZNC).  I couldn't get ZNC to work on CentOS, so I went with bip.  Instructions for ZNC are welcome.

Make sure you have [EPEL](https://fedoraproject.org/wiki/EPEL) enabled on the CentOS system and install bip:

``` terminal
yum install bip
```

Now hopefully that installed.  If it didn't, well, you might be in for a fun treat figuring out what's up.  Bip should install fine on CentOS, RHEL, and Fedora versions of recent release numbers.

## Configure Proxy

Now we get down to business and make the proxy actually do something.  The file of note is `/etc/bip.conf`.  It can be a little confusing because you will be setting up a bip user locally and also configuring it to connect to FreeNode with your nickname credentials.  The bip-user is purely local to this bip installation and is used by you to connect to the proxy.

Bip has a lot of options as well, but what I like to do is have it run and cache all channel information in a backlog and then the next time I connect push all of that to me.  This allows me to catch up where I left off and not miss anything.  I also like to log IRC traffic on my workstation for later grepping so this ensures the backlog on the proxy server is really just temporary.

Here is a condensed configuration file for my bip setup:

``` conf
ip = "0.0.0.0";
port = 7778;
client_side_ssl = true;
client_side_ssl_pem = "/srv/bip/bip.pem";
pid_file="/var/run/bip/bip.pid";
log_level = 3;
log_root = "/srv/log/bip";
log_format = "%u/%n/%Y-%m/%c.%d.log";

network {
    name = "freenode";
    server { host = "chat.freenode.net"; port = 6667; };
};

user {
    name = "bipdcantrell";
    password = "BLAHBLAHBLAHBLAH";
    admin = true;
    ssl_check_mode = "none";

    default_nick = "dcantrell";
    default_user = "dcantrell";
    default_realname = "dcantrell";

    backlog = true;
    backlog_lines = 0;
    backlog_always = true;
    backlog_reset_on_talk = true;
    backlog_reset_connection = true;
    backlog_msg_only = true;

    connection {
        name = "freenode";
        network = "freenode";
        nick = "dcantrell";
        user = "dcantrell";
        away_nick = "dcantrell_away";
        no_client_away_msg = "currently disconnected";
        ignore_first_nick = true;

        on_connect_send = "/msg nickserv identify BLAHBLAHBLAHBLAH";

        channel { name = "#weldr"; };
    };
```

OK, let's take a walk through this file.  I have condensed my example above to uncommented lines in my file.  As you edit the default file, you'll see a lot of this stuff explained in more detailed.

The first thing I do is set this thing to listen on port 7778 on all configured interfaces.  Next up is defining the _freenode_ connection and what server and port to connect to.

The business end of the config file is the _user_ block where I define my bip user account and what I want it to do.  Most of these things are easy to figure out by also looking at the example configuration file.

The password is generated by running the **bipmkpw** command.  The string it outputs needs to be pasted in to the password value.  Even though this is local to bip only, I recommend adjusting permissions on the configuration file so prying eyes have a little more difficulty peeking.

The _backlog_ settings are the ones that cache channel activity and private messages while I am disconnected.  By setting the lines to 0, that means log everything.  This could, in theory, fill up a filesystem.  But, well, who cares?  I always want it to backlog and then once I connect and bip sends my client the backlog, I want it reset.  If you don't do this, you'll always get the same backlog each time you reconnect (actually, it'll have more and more in it since it's the backlog since bip started).  The msg only setting avoids logging joins, parts, quites, and other diagnostic type messages in the backlog.  Some people like that stuff, I don't.

Next we have the _connection_ setting within the _user_ block to connect to FreeNode and put me in **#weldr**.  Match the _nick_ and _user_ to your registered nickname setting.  Modify the on connect send string to send your password with the nickserv identify command.  That is, replace the BLAHBLAHBLAHBLAH part with your plain text FreeNode nickserv password.

Lastly we have the _channel_ block to join **#weldr**.  If you are in other channels on FreeNode, you can add them here separate by commas with no spaces.  Easy.

Save the configuration file and exit.  Time to start the service.

## Start the Proxy

I'm just an unfrozen caveman, so on my fairly old CentOS system, the commands I issue to enable and start the proxy server are:

``` terminal
chkconfig bip on
service bip start
```

The syntax for CentOS 7.x and higher involves systemctl but the commands above will work and scream the new sanctioned command at you so you don't make the same mistake again.  Other systems: don't know.

## Punch a Hole in the Firewall

Right, so you need to allow inbound connections on port 7778.  I did it with this command:

``` terminal
iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 7778 -j ACCEPT
```

You probably want to get that saved in the iptables ruleset so it is active if the host reboots.  I'll leave that to you and Google to figure out.

## Connect to the Proxy

Back over on your workstation, fire up your IRC client.  This is where it's sort of weird.  For the server to connect to, you need to specify the hostname of the server running bip.  The port is 7778 and you need to enable SSL for the connection.

There is a password, which is:  ```USERNAME:PASSWORD:CONNECTION```

Let's break this down.  Bip works by encoding everything it needs in the IRC password field and then picking that apart to figure out what you want from the configuration file.  Neat.

USERNAME is the _name_ value from the _user_ block.  This is where I recommend giving it a unique name to remind you this is the bip user.  My name is David, so I might use **bipdavid**.

PASSWORD is the plain text password that you gave to the **bipmkpw** command earlier.

CONNECTION is the name of the connection you want to use from your configuration.  In this case, we want **freenode**.

Let's say my password is ThunderC4tz.  I would set ```bipdavid:ThunderC4tz:freenode``` as my password for this connection in my client.

If you did everything right and there are absolutely no other problems anywhere, you should connect like any other IRC service and see yourself in **#weldr**.

## Conclusion

Didn't work?  Don't know how to configure an IRC client?  Is LILO stuck at LI and doing nothing?  Feel free to contact me and I'll try to help out.  If I understood how to expose my author contact information via Jekyll, I would do that right now.  However that's going to have to be another change on the site because I'm still learning Jekyll.  I am in the Fedora community and I imagine you can probably find me that way or through github.
