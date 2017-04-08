Avahi aliases
-------------

On most Linux variants your machine is automatically advertised on the local network in the [`.local`](https://en.wikipedia.org/wiki/.local) domain by [Avahi](https://en.wikipedia.org/wiki/Avahi_(software)) using multicast DNS.

So you or any other machine on the local network can ping your machine like so:

    $ ping myhostname.local

Where `myhostname` is the value returned by:

    $ hostname

In addition to the normal advertising of a machine's hostname it would be nice to advertise CNAMEs, i.e. aliases, for it. E.g. if I was running an [Apache Spark](http://spark.apache.org/) master on my machine I might want to advertise it as `spark-master.local`.

---

If you Google for this you find a fair number of projects - I downloaded the main hits and took a look at all of them.

In the end it turns all of them are just variants on a Python script that was originally published on the Avahi wiki. There was one exception - a Ruby script that's part of the [OpenShift Origin server](https://github.com/openshift/origin-server) project.

---

First off Avahi doesn't seem to support aliases directly through one of their standard commands (someone did submit a patch, see [here](https://lists.freedesktop.org/archives/avahi/2012-July/002173.html) and [here](https://lists.freedesktop.org/archives/avahi/2012-July/002173.html), but it never got accepted) but their website (essentially offline since sometime in 2016) used to host a very short and simple Python script that would do this.

That script is included here as [`avahi-alias`](avahi-alias) and was taken from the October 2015 [archived version](https://web.archive.org/web/20151016190620/http://www.avahi.org/wiki/Examples/PythonPublishAlias) of the original wiki page as found on the Wayback Machine.

Note that the script depends on avahi and dbus imports. Dbus should be installed by default but for avahi you need to do:

    $ sudo apt-get install python-avahi

Then to advertise a CNAMEs simply run the script like so:

    $ ./avahi-alias spark-master.local

You can specify as many names as you want (separated by spaces). The names will be advertised as long as the script is left running, just press control-C to exit.

Note: all names must end in `.local`.

Systemd
-------

If you're running a modern Linux variant it's probably running [systemd](https://en.wikipedia.org/wiki/Systemd). So if you want to install the above as a service that's run at system startup you just need a unit file for systemd.

The necessary unit file is trivial - it's included here as [`avahi-alias.service`](avahi-alias.service). To install the Python script and systemd unit file and start things as a service just do:

    $ sudo cp avahi-alias /usr/local/bin
    $ echo spark-master.local > aliases
    $ sudo mv aliases /etc/avahi
    $ sudo cp avahi-alias.service /etc/systemd/system
    $ sudo systemctl daemon-reload
    $ sudo systemctl start avahi-alias

Check the status and ping the alias:

    $ systemctl status avahi-alias
    $ ping spark-master.local

To change an alias or add additional ones just edit `/etc/avahi/aliases` and then run:

    $ sudo systemctl restart avahi-alias

To enable the service to be started automatically at startup:

    $ sudo systemctl enable avahi-alias

Licenses
--------

The [`avahi-alias.service`](avahi-alias.service) at just ten lines hardly seems to warrant a license, but as I was asked about [this](https://github.com/george-hawkins/avahi-aliases-notes/issues/1) I've included a [`LICENSE`](LICENSE.txt) file here and release the service file under the version 2 Apache license.

The [Python script](avahi-alias), that does the real work, was written by Damjan Georgievski and contributed to the Avahi wiki in 2006. There is no statement there as to any specific license and it has been incorporated into many projects (see later) that have been released under various different licenses. I've emailed Damjan about this.

Alternative projects
--------------------

So as we've seen aliases just require one short Python script (and a systemd unit file it you want to run it as a service). So what are all all the other projects that Google turns up?

Nearly all of them are minor variants on the Avahi wiki script. Here are my notes on what I found.

### Python variants

Github user Gdamjan took the original Avahi wiki script (as you can see from the [commit history](https://gist.github.com/gdamjan/3168336/revisions)) and modified it slightly to create [this gist](https://gist.github.com/gdamjan/3168336). The only changes are that it outputs some helpful usage information if you don't specify any aliases,  uses some more concise Python constructs and automatically adds `.local` if required.

Github user Airtonix took Gdamjan's gist and turned in into [avahi-aliases](https://github.com/airtonix/avahi-aliases) - making it [pip](https://en.wikipedia.org/wiki/Pip_(package_manager)) installable and useable as a daemon (with names stored in `/etc/avahi/aliases.d/default`). A script for use with Upstart (Ubuntu's old init system) is included. The core Avahi code is identical to that in Gdamjan's gist.

Let's demonstrate it quickly by modifying it slightly to run from the command line (rather than as a daemon) and then getting it to advertise `simple-test.local`:

    $ sudo apt-get install python-avahi
    $ curl -O https://github.com/airtonix/avahi-aliases/blob/master/avahi_aliases/bin/avahi-alias
    $ sed -i 's/ALIAS_DEFINITIONS =.*/ALIAS_DEFINITIONS = [ "aliases-list" ]/' avahi-alias
    $ sed -i '/daemon/d' avahi-alias
    $ echo '        AvahiAliases().run()' >> avahi-alias
    $ echo simple-test.local > aliases-list
    $ python avahi-alias

In another terminal window ping `simple-test.local` and it should ping your current machine. To stop the advertising just kill the Python process with control-C.

The last commit to this project was in 2013 - and a number of forks have been made. Looking at the Github fork graph for the most interesting ones (in terms of numbers of commits are) one finds:

* a [systemd version](https://github.com/5sw/avahi-aliases) - this forked from the Airtronix project before it incorporated Gdamjan's minor changes, so the core Avahi code is identical to the original Avahi wiki code. In addition to a systemd unit file, this fork adds command line commands allowing one to add, list and remove aliases on the fly (without having to edit a configuration file). A little oddly this version starts a separate version of the underlying Python script for each name advertised.
* the [till/avahi-aliases](https://github.com/till/avahi-aliases) and [PraxisLabs/avahi-aliases](https://github.com/PraxisLabs/avahi-aliases) versions - despite the apparent number of additional commits, beyond those in the original, neither involves any noteworthy change to the Airtronix setup (the PraxisLabs changes all seem to be related to adding [Aegir](http://www.aegirproject.org/) support).

Separately to `avahi-aliases` and its many forks Github user Jinko created a similar project with an almost identical name [`avahi-alias`](https://github.com/jinnko/avahi-alias). This uses the original wiki Python script unchanged and adds an old style Debian init script and rather than reading names from the command line it reads them from an XML formatted file. Bizarrely despite the complex format of the XML file (described in the README) that makes it look like it supports all kinds of additional DNS-SD features, it actually ignores everything except the simple hostname entries and advertises them just like the original script.

Like the other projects it's long inactive - the last commit was in 2012. Minor further development occurred in the [fork](https://github.com/asharpe/avahi-alias) by user Asharpe that removes the pointless XML configuration, replacing it with a simple text file with a hostname per line (and instead of a Debian init script provides Upstart scripts).

### OpenShift Origin server

All the projects above are just minor variants on the original Avahi wiki script and none are undergoing active development (which isn't necessarily a bad thing - the original is simple and works).

The only Avahi alias code that I found that seems to be part of an actively maintained project is the [Avahi-CNAME-manager](https://github.com/openshift/origin-server/tree/master/extras/avahi-cname-manager) that's part of the OpenShift Origin server.

Note: while the overall server project sees regular commits, the Avahi-CNAME-manager code itself is unchanged since January 2014.

The Avahi-CNAME-manager comes with a systemd unit file and has a built in REST server that allows on-the-fly adding and removing of aliases. Unlike all the other projects the code here has been written from scratch in Ruby. And unlike the Avahi wiki code:

* It does not make use of the [IDN ToASCII algorithm](https://en.wikipedia.org/wiki/Internationalized_domain_name#ToASCII_and_ToUnicode) (which is only relevant if you plain to advertise names that involve non-ASCII characters).
* It can publish aliases for any machine on the local network, not just the machine that it's running on.

Let's demonstrate it quickly by modifying it slightly to run from the command line (rather than as a daemon) and then getting it to advertise `simple-test.local`:

    $ sudo apt-get install ruby-dbus ruby-sinatra ruby-parseconfig
    $ curl -O https://raw.githubusercontent.com/openshift/origin-server/master/extras/avahi-cname-manager/bin/avahi-cname-manager
    $ sed -i -e 's?/var/lib/avahi-cname-manager/??' -e 's?/etc/avahi/??' avahi-cname-manager
    $ echo simple-test.local=$(hostname).local > aliases
    $ printf 'KEY_NAME=alpha\nKEY_VALUE=beta' > cname-manager.conf
    $ ruby avahi-cname-manager

In another terminal window you should then be able to ping `simple-test.local` and you can also add and remove aliases using curl like so:

    $ curl --data "key_name=alpha&key_value=beta&cname=simpletest.local&fqdn=$(hostname).local" http://localhost:8053/add_alias

Important: if either the hostname for the `cname` or the `fqdn` values contains anything other than the letters a to z then it will generate a 400 response and a message like this:

    Invalid CNAME 'another-name.local'

This is odd as it doesn't care about characters like the minus sign if they're used in the `aliases` file. As the host I'm working on does include a minus in its name it's impossible to specify it as the `fqdn` value.

And what's the `alpha` and `beta` thing about? It's a simple form of authentication, you could change `beta` in the file `cname-manager.conf` to e.g. some more obscure passphrase and then only remote parties that know the passphrase could add or delete aliases (I don't know why it's useful to be able to configure both a key name and a key value).
