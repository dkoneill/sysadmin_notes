# Monitor influxdb data flows and restart telegraf

Telegraf configures MQTT sources to accept data, transform, and insert
it into influxdb. Sometimes remote MQTT sources such as ttn become
disconnected and telegraf doesn't have the inate ability to monitor
and restart those connections.

This document describes a monit configuration that looks at the delay
in data records in influxdb and based on that delay restarts telegraf
which fixes the problem. This is not necessarily definitive because
some remote sensors don't report on a high frequency basis, therefore,
the check is done every 40 minutes or so rather than the default 120
second monit cycle.

### 1. `/usr/local/bin/check_influx_data_flow.py`

Create this script and make it executable. Insert the proper
information for the databases and queries.
```

#!/usr/bin/env python3
import subprocess
import sys
from datetime import datetime, timedelta

import argparse

# CLI argument parsing
parser = argparse.ArgumentParser(description="InfluxDB check script for Monit")
parser.add_argument("--debug", action="store_true", help="Print debug information")
args = parser.parse_args()
debug = args.debug

# Define your databases and multiple queries per DB
queries = {
    "sensors": [
        "SELECT timestamp, uplink_message_f_cnt FROM \"mqtt\" WHERE \"topic\" =~ /(?i).*rsf1-lht65.*/ ORDER BY time DESC LIMIT 1",
        "SELECT timestamp, uplink_message_f_cnt FROM \"mqtt\" WHERE \"topic\" =~ /(?i).*ls-11x.*/ ORDER BY time DESC LIMIT 1",
        "SELECT timestamp, uplink_message_f_cnt FROM \"mqtt\" WHERE \"topic\" =~ /(?i).*lht52-a.*/ ORDER BY time DESC LIMIT 1"
    ]
}

MAX_AGE_MINUTES = 30
now = datetime.utcnow()
status = 0

def log(msg):
    if debug:
        print(msg)

def parse_influx_time(nanotime):
    """Convert Influx nanosecond timestamp to datetime object (UTC)."""
    try:
        timestamp = int(nanotime)
        dt = datetime.utcfromtimestamp(timestamp / 1e9)
        return dt
    except ValueError:
        return None

for db, db_queries in queries.items():
    for query in db_queries:
        try:
            cmd = [
                "influx", "-database", db,
                "-execute", query,
                "-format", "csv"
            ]
            output = subprocess.check_output(cmd, stderr=subprocess.DEVNULL).decode("utf-8").strip().splitlines()

            if len(output) < 2:
                log(f"ERROR: No data returned for {db} query: {query}")
                status = 1
                continue

            header = output[0].split(",")
            row = output[1].split(",")

            # extract timestamp and check age
            time_idx = header.index("time")
            influx_time = row[time_idx]
            record_time = parse_influx_time(influx_time)

            if not record_time:
                log(f"ERROR: Invalid timestamp for {db} query: {query}")
                status = 1
                continue

            age_minutes = (now - record_time).total_seconds() / 60
            if age_minutes > MAX_AGE_MINUTES:
                log(f"ERROR: Stale data for {db} query (age: {age_minutes:.1f} minutes): {query}")
                status = 1
            else:
                log(f"OK: {db} result = {row} (age: {age_minutes:.1f} minutes)")

        except Exception as e:
            log(f"ERROR: Query failed for {db}: {e}")
            status = 1
```

### 2. `/etc/monit/conf-available/influx-data-monitor`

```
CHECK PROGRAM influxdb_data_flow WITH PATH "/usr/local/bin/check_influx_data_flow.py"
    EVERY 20 CYCLES
    IF STATUS != 0 THEN RESTART
    START PROGRAM = "/bin/systemctl start telegraf.service"
    STOP PROGRAM  = "/bin/systemctl stop  telegraf.service"
```
Link to make it active and restart monit
```
cd /etc/monit/conf-enabled
ln -s ../conf-available/influx-data-monitor .

systemctl stop monit.service
systemctl start monit.service
```
