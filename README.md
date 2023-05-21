KEES - The Coloclue Network Automation Toolchain
================================================

![alt text](https://raw.githubusercontent.com/coloclue/kees/master/kees_roof.jpg "'Kees' text on roof of a farm")

This code in this repository is used for the following tasks:

    - use the database at https://github.com/coloclue/peering to generate all
      IXP peering
    - generate IRR filters for each peer
    - generate RPKI based filters for each peer
    - generate remaining BIRD configuration
    - push the BIRD configs to all boxes

KEES IS AUTHORITATIVE:
--------------------

    The 'kees' repository is the authoritative source for BIRD configurations in
    the coloclue network. Any changes to the BIRD configuration on the routers
    will be overwritten by the scripts in this repository.

    Some parts of the BIRD configurations are generated by Python scripts, others
    are manually composed and stored in the 'blobs' directory.  The 'blobs'
    directory contains a directory for each BIRD router. Later on the peering
    and rpki files are copied over the copy of the 'blobs' to augment it with
    the 'dynamic' (read IRR or RPKI) portions of the configuration. The goal is
    to have a configuration that is as dynamic as possible, meaning that the
    static router configs are completely empty.

Repository layout:
------------------

	$ tree
	.
	├── LICENSE
	├── README
	├── blobs
	│   ├── dcg-1.router.nl.coloclue.net
	│   │   ├── bird.conf                  <- Static config for bird (IPv4 definition on the routers)
	│   │   ├── bird6.conf                 <- Static config for bird6 (IPv6 definition on the routers)
	│   │   ├── peerings                   <- Filled by peering_filters 
	│   │   └── rpki                       <- Filled by rtrsub
	│   ├── dcg-2.router.nl.coloclue.net
	│   │   ├── bird.conf
	│   │   ├── bird6.conf
	│   │   ├── peerings
	│   │   └── rpki
	│   ├── eunetworks-2.router.nl.coloclue.net
	│   │   ├── bird.conf
	│   │   ├── bird6.conf
	│   │   ├── blackholes                 <- Specific annoucements to mitigate DDoSses is placed here automatically
	│   │   ├── peerings
	│   │   └── rpki
	│   └── eunetworks-3.router.nl.coloclue.net
	│       ├── bird.conf
	│       ├── bird6.conf
	│       ├── peerings
	│       └── rpki
	├── generate-peer-config.sh            <- Generates all the filters for the peers
	├── gentool                            <- A YAML to Jinja2 template generator
	├── peering_filters                    <- creates IRR filters & IXP peering configs
	├── templates
	│   ├── afi_specific_filters.j2        <- Filters where the AFI is relevant
	│   ├── ebgp_state.j2                  <- State of the eBGP sessions, toggled by the maintenance-mode setting
	│   ├── envvars.j2                     <- UID and GID to run the Bird process as
	│   ├── filter.j2                      <- Filter for peers
	│   ├── generic_filters.j2             <- Global filters where the AFI is irrelevant
	│   ├── header.j2                      <- File where all the generated files are included
	│   ├── ibgp.j2                        <- Template for the iBGP sessions
	│   ├── interfaces.j2                  <- Interfaces used by Bird (for instance to announce OSPF hello's on)
	│   ├── members_bgp.j2                 <- template for BGP sessions to members
	│   ├── ospf.j2                        <- OSPF definition
	│   ├── peer.j2                        <- Template for the BGP sessions definition to peers
	│   ├── rpkiwhitelist.j2               <- Template for RPKI whitelisted prefixes
	│   ├── static_routes.j2               <- Template for static routes (we use this for static routes towards members)
	│   └── transit.j2                     <- Template in which the BGP sessions towards our transits are defined
	├── update-routers.sh
	└── vars
	    ├── dcg-1.router.nl.coloclue.net.yml
	    ├── dcg-2.router.nl.coloclue.net.yml
	    ├── eunetworks-2.router.nl.coloclue.net.yml
	    ├── generic.yml                    <- generic KEES settings and inventory for routers and IXPs
	    ├── members_bgp.yml                <- ips & prefixes of members with BGP
	    ├── statics-dcg.yml                <- static routes NorthC (f.k.a. DCG)
	    ├── statics-eunetworks.yml         <- static routes euNetworks
	    ├── scrubbers.yml                  <- DDoS scrubber neighbours
	    └── transit.yml                    <- transit eBGP neighbours

Dependencies:
-------------

    bgpq3 - http://snar.spb.ru/prog/bgpq3/
    BIRD 2 - https://bird.network.cz/ # minimum version is 2.0.8
    OpenSSH client - https://www.openssh.com/
    PHP - https://www.php.net/  # TODO: what's the minimum supported version?
    Python 3 - https://www.python.org/  # TODO: what's the minimum supported version?
    rsync - https://rsync.samba.org/

You will also need the Python packages in `requirements.txt`.

Most dependencies can be installed on Debian with `# apt install bgpq3 bird2
openssh-client php python3-jinja2 python3-numpy python3-requests python3-yaml
rsync`. Some of the Python dependencies can't be found in the Debian
repository. You can use [pip](https://pip.pypa.io/en/stable/) to install them:
`$ pip3 install -r requirements.txt`.

BIRD 2 has IPv4 and IPv6 support in the same process, although Kees isn't built for this.
It is required to use a second process for IPv6, which uses the bird6.conf config file.

Usage:
------

Install the above dependencies first.

Note that KEES currently requires the ability to write to the
`/opt/routefilters/` and `/opt/router-staging/` directories. This is a bug, we
are tracking it in [#15](https://github.com/coloclue/kees/issues/15).

    $ ./generate-peer-config.sh - Generate new filters for peers, you need to run this before ./update-routers.sh push
    $ ./update-routers.sh push  - To build a new config and push it to the routers, uses the filters generated by ./generate-peer-config.sh
    $ ./update-routers.sh check - To build a new config and validate it, but don't push it to the routers

The `update-routers.sh` script expects to find `bird` on your PATH. Non-root
users most likely want to run it like this:  
`$ PATH="$PATH:/usr/sbin" ./update-routers.sh check`.

To prepare a Debian machine to build Kees:

```
sudo apt install python3-pip bgpq3 python3-jinja2
sudo pip3 install rtrsub
sudo pip3 install numpy
sudo pip3 install ipaddr
sudo pip3 install pyyaml
sudo pip install jinja2
sudo pip install hiyapyco

sudo mkdir -p /opt/routefilters/
sudo mkdir -p /opt/router-staging/
sudo chown -R ${USER} /opt/routefilters/ /opt/router-staging/
```

And symlink in your `vars/` and `blobs/` directories.

Authors:
-------

Copyright (c) 2014-2017, Job Snijders <job@instituut.net>  
Copyright (c) 2017-2022, Network committee Coloclue <routers@coloclue.net>  
Copyright (c) 2020, Imre Jonk <imre@imrejonk.nl>
