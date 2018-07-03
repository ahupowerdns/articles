---
title: "Kolmo"
date: 2017-10-24T10:24:57+02:00
draft: false
---

## Welcome to Kolmo
There are three avenues to learn about Kolmo
([GitHub](https://github.com/ahuPowerDNS/kolmo)).  There are two Youtube videos,
one from the most excellent [NLNOG
2017](https://www.youtube.com/watch?v=pb4X-Jjd1yw), the other from the equally
excellent [UKNOF 38](https://www.youtube.com/watch?v=Hy8nmyBXmyc).  These
videos explain a lot of the the history and the 'why' behind Kolmo.

There is also the [kolmo.org](https://kolmo.org) website, which is
'self-hosted' by the `ws` Kolmo-powered webserver.

Finally, there is this post, in which I focus on what Kolmo actually does,
without spending much time on the 'why' or the history.

## The state of the art
Most software has a configuration file, perhaps in ".ini" format or JSON,
XML or something custom.  Many items in the configuration file also have
defaults, described in two places: the source code and the actual product
technical documentation.  These frequently agree.  Defaults in the source
code change however, and the documentation does not always keep up.

In addition, frequently the commandline can overrule the configuration file
in some aspects.

Finally, parts of the configuration can typically be changed at runtime, but
only some parts. 

And here is the thing.  Every software project has to build all this from
scratch.  Sure there are configuration file parsers and command line
parsers.  You could even just load JSON or XML files.  But it remains a ton
of work.  In fact, with the exception of the configuration file parser,
there is (almost) nothing to help you get this done.

This also means that luxurious things, like building an API for the
configuration file, adding configuration file validation or automated
constraints are a lot of work.

## Kolmo features
What Kolmo offers is:

 * A configuration file definition language
   * Self-documenting, with constraints, units, and metadata
   * 'Typesafe', so knows about IP addresses, port numbers, strings, integers
 * Tool that turns this configuration schema into Markdown-based documentation
 * A standalone parser for configuration files
   * Test for validity, consistency
 * Runtime library for parsing configuration file & getting data from it
 * Standalone tooling to interrogate and manipulate the configuration
 * A runtime loadable webserver that allows manipulation of running
 configuration (within constraints)
 * Every configuration change is stored and can be rolled back
 * Ability to dump, at runtime:
   * Running configuration
   * Delta of configuration against default ('minimal configuration')
   * Delta of running configuration versus startup configuration

In effect, a Kolmo enabled piece of software gets a documented configuration
file that can be modified safely and programmatically, offline, on the same
machine or at runtime, with a full audit trail, including rollback
possibility.

An important goal of Kolmo is to ease the work of automation: with Kolmo, it
should be possible to easily change and serialize the state of a well
working configuration, and get it in an readibly deployable state.

### The configuration file schema
`ws` is a simple webserver which actually powers
[kolmo.org](https://kolmo.org). 
A snippet from the [configuration file
schema](https://github.com/ahupowerdns/kolmo/blob/master/ws-schema.lua) of `ws`:

```
main:registerVariable("verbose", "bool", { 
   default="true", runtime="true", cmdline="-v", 
   description="Perform verbose logging"})

main:registerVariable("server-name", "string",  { 
   default="", runtime="true", 
   description="Name this server reports as by default"})

main:registerVariable("client-timeout", "integer", {
   default="5000",
   runtime="true",
   unit="milliseconds",
   description="Timeout before client gets disconnected",
   check=
   'if(x < 1) then error("Timeout must be at least one millisecond") end'
})
```

This defines three top-level variables called `verbose`, `server-name` and
`client-timeout`. 

With the following command, we can turn this into documentation:
```
$ kolctl --schema ws-schema.lua markdown > ws-schema.md.html
```
Through some Javascript magic and the use of Markdeep, this renders as both
Markdown and HTML. 

Every configuration item has a type (for example `bool`), but also a
default. In addition, `runtime` indicates if the program can accept updates
to this setting at runtime. Finally the `description` is used in the
documentation.

More interesting is the definition of new `classes`:
```
site=createClass("site", "A site we serve")
site:registerVariable("name", "string", { 
    runtime="false", description="Hostname of this website" 
})
site:registerVariable("enabled", "bool", { 
  runtime="false",   default="true",   
  description="If this site is enabled"
})
site:registerVariable("path", "string", { 
  runtime="true",   description="Path on fs where content is"
})
  
site:registerVariable("listen", "struct", { 
  member_type="ipendpoint", runtime="false", 
  description="IP endpoints we listen on"})
  
site:registerVariable("redirect-to-https", "bool", { 
   default="false", runtime="true", 
   description="If all http requests should be redirected to https"
})

sites=main:registerVariable("sites", "struct", {
  member_type="site", runtime="false", description="Sites we serve"
})
```

This defines a new class called a `site`, and several variables of such a
site, like its name, `enabled` status, `path` of the content and the
`ipendpoint`s (IP address + port) on which the site listens. Finally it
defines a `bool` which determines if all http requests should be forwarded
to https.

The last line actually hooks up the `site` to a top-level variable called
`sites`.

### The configuration file
A typical configuration file then looks like this:
```
{
    "listeners": {
        "[::]:443": {
            "cert-file": "/etc/letsencrypt/live/kolmo.org/fullchain.pem",
            "key-file": "/etc/letsencrypt/live/kolmo.org/privkey.pem",
            "tls": true
        }
    },
    "server-name": "kolmo.org",
    "sites": {
        "kolmo": {
            "listen": {
                "0": "[::]:443"
            },
            "name": "kolmo.org",
            "path": "/var/www/kolmo.org",
            "redirect-to-https": true
        }
    },
    "verbose": false
}
```
Note how `sites` as defined above is filled with one item called
`kolmo`. In addition, we can see that the `server-name` is set. More
interesting, a `listener` is defined with a certificate and a private key.

The configuration file shown is stored in `ws.conf`, and looks exactly as
outlined. However, we can also run `kolmoctl --schema ws-schema.lua --config
ws.conf full-config`, which gives us a configuration file with all the
defaults from the schema in there. This contains everything in ws.conf,
augmented with all the omitted defaults:

```
{
    "carbon-server": "",
    "client-timeout": 5000,
    "hide-server-type": false,
    "hide-server-version": false,
...
    "loggers": {
        "messages": {
            "log-errors": true,
            "log-file": "",
            "log-hits": false,
            "log-warning": true,
            "syslog": true,
            "syslog-facility": "daemon"
        }
    }
}
```

### Exploring the configuration
When exploring a new piece of software, we can read the documentation.. but
what is more fun than interrogating the configuration options directly?

```
$ alias wsctl="kolctl --schema=ws-schema.lua --config=ws.json"
$ wsctl ls
carbon-server                                Send performance metrics to this IP address
client-timeout           5000                Timeout before client gets disconnected
hide-server-type         false               If we should hide server type
hide-server-version      false               If we should hide server version number
kolmo-servers            {struct}            Kolmo servers on which to provide Kolmo service
listeners                {struct}            Optional configurations per IP address listener
loggers                  {struct}            Loggers that log events and hits
max-connections          200                 Maximum number of connections
server-name                                  Name this server reports as by default
sites                    {struct}            Sites we serve
verbose                  true                Perform verbose logging

$ wsctl ls sites
ds9a.nl                  {struct}            A site we serve
kolmo                    {struct}            A site we serve

$ wsctl ls sites/kolmo
enabled                  true                If this site is enabled
listen                   {struct}            IP endpoints we listen on
name                     kolmo.org           Hostname of this website
path                     /var/www/kolmo.org  Path on fs where content is
redirect-to-https        true                If all http requests should be redirected to https
```

Alternately, use the 'kolctl markdown' command to get a pretty HTML rendering of
all configuration items, classes and their defaults.

### Changing the configuration
Note that common configuration file parsers can parse your configuration
just fine, but they can't write out a copy of the running configuration.

Let's make some changes:

```
$ wsctl set client-timeout=1000
$ wsctl minimal-config | grep client-timeout
    "client-timeout": 1000,

```

Or more interestingly, let's try to configure the carbon-server as something
that is an invalid IP address/port combination, and then do it right:

```
$ wsctl set carbon-server=1.2.3.4:90000
Fatal error: Unable to convert presentation address '1.2.3.4:90000'
$ wsctl set carbon-server=1.2.3.4:9000
$ wsctl ls carbon-server
carbon-server            1.2.3.4:9000        Send performance metrics to this IP address
```

Or let's exercise the constraint defined in the schema `client-timeout`:
```
$ wsctl set client-timeout=0
error: Timeout must be at least one millisecond
```

### Audit trail
Whenever we made a change as above, this left a copy of the minimal
configuration file with a timestamp:

```
$ ls -l ws.json*
lrwxrwxrwx 1 ahu ahu  23 okt 23 20:02 ws.json -> ws.json.20171023-200247
-rw-rw-r-- 1 ahu ahu 281 okt 23 20:01 ws.json.20171023-200129
-rw-rw-r-- 1 ahu ahu 318 okt 23 20:02 ws.json.20171023-200234
-rw-rw-r-- 1 ahu ahu 318 okt 23 20:02 ws.json.20171023-200247
```
Reverting to an old configuration is as easy as moving the symlink. The
files can also easily be compared to see what changed.

### Runtime changes
You can't randomly change configuration settings and hope the program
supports such changes at runtime. So specific settings that support runtime
changing are marked in the schema file. To make a runtime change, the
`kolctl` program connects to your running software over http.

First let us define a kolmo-server, and then connect to it:
```
$ wsctl show kolmo
listen-address                               IP address on which to run a Kolmo server
readonly-password                            Password that grants RO access to this server
readwrite-password                           Password that grants RW access to this server

$ wsctl add kolmo-servers 0 '{"listen-address": "127.0.0.1:2001"}'
$ ws &
$ alias wsctl='kolctl --remote=http://127.0.0.1:2001'
$ wsctl set verbose=true
{"result":"ok"}
```

We are now interrogating the configuration not via the files on disk
directly, but through the running daemon. In the example above, we changed
the `verbose` setting to `true`. Let's check:

```
$ wsctl delta-config
{
    "verbose": true
}
```

This shows us the difference between the configuration we were started with
and the one we are running now.

If we undo the change, it looks like this:

```
$ wsctl set verbose=false
{"result":"ok"}
$ wsctl delta-config
{}

```

If we want to store the currently runtime configuration, we can extract it
easily enough with 'wsctl minimal-config', as above. 

Note that behind the scenes, `wsctl` (or actually `kolctl`) is only making
calls to a RESTful API running within the running program.  In other words,
if you speak 'REST' you can modify a Kolmo enabled configuration at ease.

Finally, `kolctl --webserver` can be used to provide a RESTful API on a
configuration file without the actual program being present, which is useful
to manipulate configuration files at any time, or from any place.

## Applicability to automation
Currently, to automate the deployment of pieces of software requires either
custom modules, typically third party, that "know" how to configure a piece
of software. Alternatively, more generic 'lineinfile' modules can be used to
(bluntly) edit configuration files by ensuring the presence or absence of
certain lines. There are also '.ini' modules.

With Kolmo, it is possible to programmatically change a configuration file,
either on the system doing the automation (locally) or on the destination
host (remotely). In both cases 'kolctl' is used to modify a configuration
file until it has the desired state.

## The programming language side of things
As noted, default values for settings frequently live in the actual code of
software projects. This is hard enough to find, but also makes it very easy
for documentation and reality to get out of sync.

A prime goal for Kolmo is therefore to move defaults to only one place: the
configuration schema definition. It turns out that this not only unifies
documentation and reality, it also shortens the actual code a lot.

This is from the `ws` sample webserver:
```
  KolmoConf kc;
  kc.initSchemaFromFile("ws-schema.lua");

  kc.initConfigFromJSON("ws.json");
  kc.declareRuntime();
  kc.initConfigFromCmdline(argc, argv);

  kc.d_main.tieBool("verbose", &g_verbose);
```

This 'ties' the configuration file `verbose` setting to the variable
`g_verbose`.  If the runtime verbose setting is changed, so does
`g_verbose`.

Note that if the configuration file is not compliant with the schema, an
exception is thrown. The benefit of this is that we can rely on the
configuration being valid at all times. 

Getting the actual `site` objects from the configuration:

```
  auto sites = kc.d_main.getStruct("sites"); 

  for(const auto& m : sites->getAll()) {
    auto site = dynamic_cast<const KolmoStruct*>(m.second); // this will get simpler
    cout<<"["<<m.first<<"] We run a website called "<<site->getString("name")<<endl;
    if(!site->getBool("enabled")) {
      cout<<"However, site is not enabled, skipping"<<endl;
    }
    else {
      cout<<"The site enable status: "<<site->getBool("enabled")<<endl;
      cout<<"We serve from path: "<<site->getString("path")<<endl;
      cout<<"We serve on addresses: ";

      auto listeners = site->getStruct("listen");
      for(const auto& i : listeners->getMembers()) {
        ComboAddress ca=listeners->getIPEndpoint(i); // typesafe
        cout<<ca.toStringWithPort()<<endl;
        listenAddresses.insert(ca);
      }
    }
    cout<<endl;
  }

```

Worthy of note is the line marked 'typesafe'.  We get actual IP address &
port numbers from the configuration parser, not just a string.  There's no
need for any conversion, the IP address is already in native form.

## Immediate future
Based on the infrastructure present, it is straightforward to teach `kolctl
--webserver` (and the process runtime kolmo webserver thread) to not only
provide a RESTful API, but also to host a web-based control panel of the
configuration.

This would not only be pretty, but also enforce all the constrains and
type-safety embedded in the configuration file schema. It would also
helpfully display the documentation and units of all settings.

## Practical matters
Kolmo is very fresh. It is MIT licensed, so easy to embed and use. Right
now, there is only a C++ library. To be fair, Kolmo is more of a 'proof of
concept' than anything you'd want to use.

It is my hope however that "what has been seen can not be unseen". In other
words, for your next project, you'll ponder why you are writing all this
configuration validation code, and why you can't serialized your
configuration in minimal form compared to the defaults.


