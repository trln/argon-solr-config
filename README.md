# TRLN Discovery Solr Configs

This directory contains Solr ConfigSets for the `trlnbib` index used by TRLN
Discovery.  The document schema was derived originally from
https://github.com/billdueber/solr6_test_conf and has added features to enhance
rollup functionality and CJK handling (via https://github.com/sul-dlss/CJKFilterUtils).

Each ConfigSet describes a collection, including the index schema and various
'handlers' that affect the behavior of the collection when queried.  In
general, changes to a ConfigSet should be tested somewhere, then committed to a
shared Git repository, before they are sent to production Solr.

## Managing ConfigSets via ZooKeeper

Since we run Solr in "SolrCloud" mode, the configuration of each collection is
managed through ZooKeeper.  Solr ships with an 'embedded' ZooKeeper instance
that includes a `zkcli.sh` script for communicating with ZooKeeper; such a
script is also available from the standalone ZooKeeper distribution, but the
one in a Solr distribution seems to work best. 

Solr-specific documentation is kept in the "Solr Reference Guide" which changes
from version to version, here is a link to the v7.2 reference guide:

https://lucene.apache.org/solr/guide/7_2/command-line-utilities.html

### A note about updating configsets

If a configuration contains 'overlay' files (e.g. `configoverlay.json` that
store the same values as ones that are also found in, e.g. `solrconfig.xml`,
then those values will override whatever is stored in `solrconfig.xml`.  In
general we are trying to avoid doing this, as JSON has formatting requirements
(no multiline strings) that make it hard to maintain configurations this way.
Note that using Solr's "web" API to manage the configuration *will*
automatically create `configoverlay.json` so please don't mix and match that
API with direct ZK management of the configuration. 

If you do, please note that Zookeeper's `upconfig` command copies new files and
updates ones previously stored, but does *not* _delete_ any files that you have
removed from the directory you upload.  If you do run into a situation where
the configoverlay.json (or other file) is present, and you want to delete it,
look at the `clear` command offered by the ZooKeeper client; it will delete
individual named files as well as directories.

### `zkcli.sh` examples at a glance.

For a running Solr instance, it will typically have a `SOLR_HOME` which will
usually be something like `/opt/solr/server`, and you can usually find
`zkcli.sh` under `$SOLR_HOME/scripts/cloud-scripts`.  Your setup may vary.

### Downloading the current configuration for a collection:

You need to be able to 'talk to' the ZooKeeper instance Solr is using; this
will usually require that you be "inside the firewall", e.g. in AWS that you
run these *at least* from a host in the same VPC.  Additionally, your host must
be allowed to talk to the ZooKeeper nodes over their management port (default
is 2181).

You might need to do the above if the configuration in this repository somehow
gets out of synch with the configuration in ZooKeeper.

#### 'embedded' Solr on the same machine, running standalone SolrCloud

    $ $SOLR_HOME/scripts/cloud-scripts/zkcli.sh -zkhost localhost:9983 -cmd downconfig -confname trlnbib -confdir trlnbib

Notes:

  * `localhost:9983` is where ZooKeeper 'lives' when you run it in embedded mode (`bin/solr -c start`)
  * `zkhost` is a comma-separated list hostname:port pairs of *all* the zookeeper nodes in the cluster; the example has a single ZK node
  * `-cmd downconfig` says you want to copy the configuration to your machine
  * `-confname` is the name of the configuration 
  * `-confdir` is the path *on the machine where you are running the command*
    to which the configuration will be downloaded.  May be absolute or relative
    to the current working directory.

#### Remote SolrCloud with standalone ZooKeeper cluster from a management host

It's probably a good idea, if you are running SolrCloud, to have a single host
from which you manage all your ZooKeeper and Solr instances.  Put some
reasonably high security on it, and make sure that ZooKeeper and Solr instance
will only allow your management node (and, in the case of the Solr nodes, your
Blacklight applications) to reach them.

This assumes we have multiple ZK hosts; uploading the configset to any of the Solr nodes attached to the same ZK cluster
will propagate the changes across the cluster.

    $ $SOLR_HOME/scripts/cloud-scripts/zkcli.sh -zkhost zk1:2181,zk2:2181,zk3:2181 -cmd downconfig -confname trlnbib -confdir trlnbib

Note that the values of zkhost here have to be resolvable hostnames, so you should have already set up your `/etc/hosts` file for this to work.

#### Uploading a new config

See above, but swap `-cmd downconfig` for `-cmd upconfig`

Note that you will have to reload the collections using your configset in order
to 'pick up' the new configuration.  *This may result in downtime* as your Solr
nodes readjust, but whether this happens and for how long depends on what kind
of changes you make. You may also need to reindex your data after reloading in
order to pick up the benefits of any changes you have made.

You reload a collection via:

    $ curl https://my.solr.node:8983/solr/admin/collections?action=RELOAD&name=trlnbib

Where 'trlnbib' stands in for the name of the collection with the updated
configuration.

### ConfigSets 

  * `trlnbib`: the 'main' TRLN shared bibliographic record index
