# Detection Rules for Lightning Framework (and How to Make Them With Osquery)

> Osquery queries for detecting Lightning Framework on Linux, covering its file artifacts, persistence and rootkits, packaged as a ready-to-use query pack.

- Source: https://research.intezer.com/blog/2022/08/lightning-framework-linux-detection-rules-osquery/
- Published: 2022-08-03
- Authors: Ryan Robinson
- Tags: linux, framework, tool
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


On 21 July, 2022, we released a blog post about a new malware called
[Lightning Framework](../../../2022/07/lightning-framework-new-linux-threat).
Lightning is a modular malware framework targeting Linux. At the time of the
publication, the Core module had one suspicious detection and the Downloader
module was not detected by any scanning engines on VirusTotal. Due to this, we
have again decided to release a followup blog posting showing how the
information we released can be used to investigate whether you have been
affected.

There are many tools available to gather system information and to perform
threat hunting within an organization. One of our favorites is
[osquery](https://osquery.io). We demonstrated in a previous blog how detection
rules created with osquery can be used to discover the multi-platform malware
[Sysjoker](../../../2022/01/new-backdoor-sysjoker).

This tutorial will show how we use gathered information from the reverse
engineering process to utilize in detection techniques that can serve as an
indication of compromise for the Lighting Framework. There are pros and cons to
this approach over just using file hashes. It allows us to potentially find
other samples related to the Lightning Framework that have a different file
hash but it can also turn up matches that are not malware. For example the
malleable C2 profile is encoded on disk and will have a unique hash each time,
therefore it is important to use other means of detection that are not hash
based.

## Information Gathering with Osquery

Osquery runs as an agent on machines and is designed to provide endpoint
visibility in a performant way. The data is collected by the agent and exposed
via SQL queries.

There are two ways of running osquery, as a daemon via osqueryd or
interactively via osqueryi. The daemon takes a configuration file where the
queries are defined and executes them at a given interval. Data is continuously
collected by the daemon and the difference since the last execution is logged
to a result log. The interactive binary allows for ad-hoc queries and we will
use it in this tutorial to construct queries that can be added to the daemon's
configuration. The queries will be added to a
[query-pack](https://osquery.readthedocs.io/en/stable/deployment/configuration/#query-packs)
to make it easier to manage.

## Detecting File Artifacts from Lightning Framework

Many files are created in a working directory in the Lightning Framework. We
can use these to write threat hunting or detection queries.

If we look at the report, we can see that most of the files are created under a
folder named `/usr/lib64/seahorses/`. This is a folder trying to masquerade as
the legitimate password and key manager
[Seahorse](https://gitlab.gnome.org/GNOME/seahorse). We can use the osquery
table file to monitor for Lightning Framework files under this folder.
According to the [schema](https://osquery.io/schema/5.1.0/#file) for this
table, we can get which user created it, when the file was created, modified,
and last accessed. To enrich the data, we join the uid with the
[users](https://osquery.io/schema/5.1.0/#users) table and the gid with the
[groups](https://osquery.io/schema/5.1.0/#groups) table. This allows us to get
the username and the group name instead of just the ID numbers.

Another thing that is useful is the file hash for the Lightning files. This can
be obtained by joining with the [hash](https://osquery.io/schema/5.1.0/#hash)
table on the path. This allow us to construct a query like this:

```sql
SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
  	  JOIN users using (uid)
  	  JOIN hash using (path)
  	  JOIN groups using (gid)
  	  WHERE file.directory in ('/usr/lib64/seahorses/');
```

An example of a query that shows the malware artifacts is shown below:

![](images/fig1.png "Successfully finding Lightning Framework files using osquery.")

## Detecting Persistence of Lightning Framework

If we look at the persistence documented in the report, we can see that the
framework maintains persistence by creating a startup script for the Downloader
module. We can find this by querying the
[startup_items table](https://osquery.io/schema/5.1.0/#startup_items) and
looking for the misspelled term `elastisearch` (As opposed to the correct
application it is trying to mimic
"[Elasticsearch](https://www.elastic.co/elasticsearch/)").

```sql
SELECT * FROM startup_items WHERE name = 'elastisearch';
```

## Detecting RootKits Installed by Lightning Framework

Lightning Framework is able to install a Linux kernel module (LKM) rootkit. The
rootkit is used to hide process IDs and ports related to running processes of
the framework. Similarly to the startup item. The kernel module also has the
same misspelled `elastisearch` name. To check whether it is installed on your
machine, run the following:

```sql
SELECT * FROM kernel_modules WHERE name = 'elastisearch';
```

## Next Steps for Your Lightning Detection Rules

After this we can tell the osquery daemon to load the query pack. This can be
done by adding a pack entry to the daemon's configuration file:

```json
"packs": {
  	"lightning_framework": "/path/to/pack/lightning.json"
  }
```

To get alerting for this, it is best to forward the logs to your SIEM and
create alerts there. We're curious to see more samples of Lightning Framework –
let us know if you uncover more samples or are able to take the research on it
further!

## Appendix A – lightning.json Pack

```json
{
    "queries": {
        "linux-artifacts": {
            "query": "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
  	JOIN users using (uid) 
  	JOIN hash using (path) 
  	JOIN groups using (gid) 
  	WHERE file.directory in ('/usr/lib64/seahorses/');",
            "interval": "60",
            "version": "1.0.0",
            "description": "https://intezer.com/blog/research/lightning-framework-new-linux-threat/",
            "platform": "linux",
            "value": "Possible Lightning Framework files"
        },
        "linux-startup-item": {
            "query": "SELECT * FROM startup_items WHERE name = 'elastisearch';",
            "interval": "60",
            "version": "1.0.0",
            "description": "https://intezer.com/blog/research/lightning-framework-new-linux-threat/",
            "platform": "linux",
            "value": "Possible Lightning Framework startup item"
        },
        "linux-kernel-module": {
            "query": "SELECT * FROM kernel_modules WHERE name = 'elastisearch';",
            "interval": "60",
            "version": "1.0.0",
            "description": "https://intezer.com/blog/research/lightning-framework-new-linux-threat/",
            "platform": "darwin",
            "value": "Possible Lightning Framework rootkit"
        }
    }
}
```
