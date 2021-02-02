---
layout: post
title: Local Configuration for osquery on Windows
comments: true
---

I pretty commonly get asked by folks for a generic Windows configuration for
osquery, as the example configuration pack in the osquery repository favors
posix systems a bit ([Something we're hoping to make better](https://github.com/facebook/osquery/issues/3725)).

While the configuration is a core component to what queries one is interested
in for their enterprise, we typically perform most of the daemon configuration
through the `--flagsfile`. Below is the flags file I typically use with the
following config.

```text
--config_plugin=filesystem
--config_path=C:\ProgramData\osquery\osquery.conf
--enable_monitor
--events_expiry=300
--logger_plugin=filesystem
--logger_path=C:\ProgramData\osquery\log
--database_path=C:\ProgramData\osquery\osquery.db
--pidfile=C:\ProgramData\osquery\osquery.pid
--disable_watchdog=false
--disable_events=false
--windows_event_channels=System,Application,Setup,Security
--verbose
```

And here's the corresponding `osquery.conf

```json
{
  "schedule": {
    "heartbeat": {
      "query": "select si.hostname, si.uuid, si.computer_name, up.total_seconds as uptime from system_info si, uptime up;",
      "interval": 900,
      "snapshot": "true"
    },
    "windows_events": {
      "query": "select * from windows_events;",
      "interval": 180
    },
    "interace_addr_mac": {
      "query": "select ia.address, ia.mask, id.mac from interface_addresses ia, interface_details id where ia.interface = id.interface;",
      "interval": 300,
      "snapshot": "true"
    },
    "logged_in_users": {
      "query": "select user, time, pid from logged_in_users where type='active';",
      "interval": 300,
      "snapshot": "true"
    },
    "listening_ports": {
      "query": "select * from listening_ports;",
      "interval": 300,
      "snapshot": "true"
    },
    "processes": {
      "query": "select * from processes;",
      "interval": 300,
      "snapshot": "true"
    },
    "scheduled_tasks": {
      "query": "select * from scheduled_tasks;",
      "interval": 3600
    },
    "startup_items": {
      "query": "select * from startup_items;",
      "interval": 3600,
      "snapshot": "true"
    },
    "drivers": {
      "query": "select * from drivers;",
      "interval": 86400,
      "snapshot": "false"
    },
    "services": {
      "query": "select * from services;",
      "interval": 86400,
      "snapshot": "false"
    },
    "etc_hosts": {
      "query": "select * from etc_hosts;",
      "interval": 86400,
      "snapshot": "false"
    },
    "windows_patches": {
      "query": "select * from patches;",
      "interval": 86400,
      "snapshot": "false"
    },
    "system_users": {
      "query": "select * from users;",
      "interval": 86400,
      "snapshot": "false"
    }
  },
  "decorators": {
    "load": [
      "SELECT uuid AS host_uuid FROM system_info;",
      "SELECT user AS username FROM logged_in_users ORDER BY time DESC LIMIT 1;"
    ]
  },
  // Fill this out with custom packs, or the example packs from the repo
  "packs": { }
}
```
