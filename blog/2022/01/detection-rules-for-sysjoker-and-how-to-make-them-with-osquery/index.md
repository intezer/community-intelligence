# Detection Rules for Sysjoker (and How to Make Them With Osquery)

> Osquery queries for hunting SysJoker on Linux, macOS and Windows, built from its file artifacts and persistence and packaged as a ready-to-use query pack.

- Source: https://research.intezer.com/blog/2022/01/detection-rules-for-sysjoker-and-how-to-make-them-with-osquery/
- Published: 2022-01-14
- Authors: Joakim Kennedy
- Tags: backdoor, multiplatform, tool
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


On January 11, 2022, we released a blog post on a new malware called [SysJoker](../../../2022/01/new-backdoor-sysjoker). SysJoker is a malware targeting Windows, macOS, and Linux. At the time of the publication, the Linux and macOS versions were not detected by any scanning engines on VirusTotal. As a consequence to this, we decided to release a followup blog posting showing how the information we released can be used to investigate whether you have been affected.

There are many tools available to gather system information and to perform threat hunting within an organization. We decided to use [osquery](https://osquery.io) in this blog post. The reason for choosing osquery is that it is supported for all the affected operating systems that SysJoker targets and it is open source allowing essentially anyone to follow along in this tutorial.

## Information Gathering with Osquery

Osquery runs as an agent on machines and is designed to provide endpoint visibility in a performant way. The data is collected by the agent and exposed via SQL queries.

There are two ways of running osquery, as a daemon via osqueryd or interactively via osqueryi. The daemon takes a configuration file where the queries are defined and executes them at a given interval. Data is continuously collected by the daemon and the difference since the last execution is logged to a result log. The interactive binary allows for ad-hoc queries and we will use it in this tutorial to construct queries that can be added to the daemon's configuration. The queries will be added to a [query-pack](https://osquery.readthedocs.io/en/stable/deployment/configuration/#query-packs) to make it easier to manage.

### File Artifacts

The report highlights files and folders created by the [SysJoker malware](../../../2022/01/new-backdoor-sysjoker). We can use these to write hunting or detection queries. Since the Windows version is well detected by antivirus engines and Patrick Wardle released a [great blog post](https://objective-see.com/blog/blog_0x6C.html) on how his free macOS security tools can detect the macOS variant, we will put most focus on the Linux variant. Since osquery abstracts the differences between all the operating systems into SQL, the queries we produce for Linux can easily be translated to the other operating systems.

If we look at the report, we can see that all of the files are created under a folder named `/.Library`. This is a very unusual folder that is not expected to exist on a normal Linux install. We can use the osquery table file to monitor for files under this folder. According to the schema for this table, we can get which user created it, when the file was created, modified, and last accessed. To enrich the data, we join the uid with the [users](https://osquery.io/schema/5.1.0/#users) table and the gid with the groups table. This allows us to get the username and the group name instead of just the id numbers.

Another thing that is useful is the file hash for the files. This can be obtained by joining with the hash table on the path. This allow us to construct a query like this:

```sql
SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
        JOIN users using (uid)
        JOIN hash using (path)
        JOIN groups using (gid)
        WHERE file.directory in ('/.Library');
```

For macOS we can use the same logic by just changing the directory to the one from the report: `/Library/MacOsServices`. This also is not a standard folder.

On Windows, the malware uses two folders under the hidden ProgramData folder. We can search for files under these two folders. With this information we can create the following queries for our pack. In the pack we are using an interval of 60 seconds.

```json
    "linux-artifacts": {
      "query" : "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
      JOIN users using (uid) 
      JOIN hash using (path) 
      JOIN groups using (gid) 
      WHERE file.directory in ('/.Library');",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "linux",
      "value" : "Possible sysjoker Linux files"
    },
    "macos-artifacts": {
      "query" : "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
      JOIN users using (uid) 
      JOIN hash using (path) 
      JOIN groups using (gid) 
      WHERE file.directory in ('/Library/MacOsServices');",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "darwin",
      "value" : "Possible sysjoker macOS files"
    },
    "windows-artifacts": {
      "query" : "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
      JOIN users using (uid) 
      JOIN hash using (path) 
      JOIN groups using (gid) 
      WHERE file.directory in ('C:\ProgramData\RecoverySystem', 'C:\ProgramData\SystemData');",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "windows",
      "value" : "Possible sysjoker Windows files"
    }
```

### Persistence

If we look at the persistence documented in the report, we can see that it uses different techniques for the different operating systems: cron job on Linux, launchd entry on macOS, and registry run key on Windows. While they are different, luckily osquery has tables for these that can be used.

The cron job is executed on reboot. This allow us to query the cron table for job that runs at a reboot event with a command including the name of the malware file:

```sql
SELECT * FROM crontab WHERE event LIKE '@reboot' AND command LIKE '%/updateSystem);
```

On macOS we can use the launchd table to query for entries with a similar name to the name used by SysJoker:

```sql
SELECT * FROM launchd WHERE path LIKE '%com.apple.update.plist;
```

Finally on Windows we can query the registry table for keys similar to the registry key in the report:

```sql
SELECT * FROM registry WHERE key LIKE 'HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run' AND name = 'igfxCUIService';
```

## Next Steps

The next step is to tell the osquery daemon to load the query pack we created. This can be done by adding a pack entry to the daemon's configuration file:

```json
"packs": {
      "sysjoker": "/path/to/pack/sysjoker.json"
  }
```

Once the pack has been loaded the daemon will log result to the result log. To get alerts when matches have been found, the logs can be forward to a SIEM. (The process of configuring log forwarding to a SIEM is out of scope of this and will not be covered.)

When using packs the name of the query will be prepended by the pack name, making it easy to create an alert. For example in this tutorial, all queries with a result will have a name starting with `pack_sysjoker`. If you don't want to create a pack from scratch, you can use the pack provided in the appendix below.

---

## Appendix A – sysjoker.json Pack

```json
{
  "queries": {
    "linux-artifacts": {
      "query" : "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
      JOIN users using (uid) 
      JOIN hash using (path) 
      JOIN groups using (gid) 
      WHERE file.directory in ('/.Library');",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "linux",
      "value" : "Possible SysJoker Linux files"
    },
    "macos-artifacts": {
      "query" : "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
      JOIN users using (uid) 
      JOIN hash using (path) 
      JOIN groups using (gid) 
      WHERE file.directory in ('/Library/MacOsServices');",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "darwin",
      "value" : "Possible SysJoker macOS files"
    },
    "windows-artifacts": {
      "query" : "SELECT path, filename, username, groupname, mtime, ctime, atime, md5, sha1, sha256 FROM file 
      JOIN users using (uid) 
      JOIN hash using (path) 
      JOIN groups using (gid) 
      WHERE file.directory in ('C:\ProgramData\RecoverySystem', 'C:\ProgramData\SystemData');",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "windows",
      "value" : "Possible SysJoker Windows files"
    },
    "linux-cron": {
      "query" : "SELECT * FROM crontab 
      WHERE command LIKE '@reboot%' 
      AND command LIKE '%updateSystem)';",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "linux",
      "value" : "Possible SysJoker crontab entry"
    },
    "macos-plist": {
      "query" : "select * from launchd where path like '%com.apple.update.plist';",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "darwin",
      "value" : "Possible SysJoker macOS plist entry"
    },
    "win-registry": {
      "query" : "SELECT * FROM registry WHERE key LIKE 'HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run' AND name = 'igfxCUIService';",
      "interval" : "60",
      "version": "1.0.0",
      "description" : "https://intezer.com/blog/malware-analysis/new-backdoor-sysjoker/",
      "platform": "windows",
      "value" : "Possible SysJoker Windows registry entry"
    }
  }
}
```
