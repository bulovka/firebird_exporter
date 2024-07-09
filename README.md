# Firebird exporter & alert rules for Prometheus and Grafana dashboard

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://www.gnu.org/licenses/old-licenses/mit.en.html)
![Maintainer](https://img.shields.io/badge/maintainer-ldrahnik-blue)
[![Ask Me Anything !](https://img.shields.io/badge/Ask%20about-anything-1abc9c.svg)](https://github.com/asus-linux-drivers/asus-numberpad-driver/issues/new/choose)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)
[![Hits](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fgithub.com%2Fldrahnik%2Ffirebird_exporter&count_bg=%2379C83D&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=hits&edge_flat=false)](https://hits.seeyoufarm.com)

The Firebird exporter for Prometheus is built by a simple script written in Python with the help of [script_exporter](https://github.com/ricoberger/script_exporter) and [filestat_exporter](https://github.com/michael-doubez/filestat_exporter).

The script and Grafana dashboard have been tested with `firebird2.5.2`, `firebird2.5.7`, `firebird3.0.7` and `firebird4.0.4`.

- Metrics of database

![Screenshot from 2024-06-23 13 39 12](https://github.com/ldrahnik/firebird_exporter/assets/3233644/7f98ab59-26be-4fd0-8151-879ad5f8f0e5)

- Metrics of backup `gbak` logs

![Screenshot from 2024-06-23 14 31 32](https://github.com/ldrahnik/firebird_exporter/assets/3233644/14dec613-5dc3-455b-a4a8-79f3c6556f67)

- Custom grafana dashboard

![Screenshot from 2024-06-23 13 48 35](https://github.com/ldrahnik/firebird_exporter/assets/3233644/522fb732-1cc8-41dc-a1b4-1f2b165c2540)
![Screenshot from 2024-06-23 13 49 37](https://github.com/ldrahnik/firebird_exporter/assets/3233644/8d88980c-110a-4b7d-b777-0855a3e273a8)
![Screenshot from 2024-06-23 13 59 08](https://github.com/ldrahnik/firebird_exporter/assets/3233644/57c6a318-70b8-4e7d-a4ab-bfa5afa3e93f)
![Screenshot from 2024-06-23 13 41 02](https://github.com/ldrahnik/firebird_exporter/assets/3233644/c7d24172-423c-49ea-a511-f4ec2f0325a8)
![Screenshot from 2024-06-23 13 55 34](https://github.com/ldrahnik/firebird_exporter/assets/3233644/8cf610a1-8984-4a05-b533-e451263fdfd3)
![Screenshot from 2024-06-23 14 45 39](https://github.com/ldrahnik/firebird_exporter/assets/3233644/06f7c8a9-8433-4e78-b56f-acc7ba13218f)

# Setup

## 1. Monitoring probe (db metrics)

```
├── docker-compose.yml
└── script_exporter
    ├── scripts
    │   ├── dbs.csv
    │   └── firebird_exporter.py
    ├── ...
    ├── script_exporter.yml
    └── Dockerfile
```

Uses [script-exporter](https://github.com/ricoberger/script_exporter) built from the source code by enhancing the original [Dockerfile](https://github.com/ricoberger/script_exporter/blob/main/Dockerfile). The environment that includes firebird lib is ensured by docker image [jacobalberty/firebird:3.0](https://hub.docker.com/r/jacobalberty/firebird). Script exporting the metrics is written in python and uses [firebird-driver](https://pypi.org/project/firebird-driver/) installed with `pip`.

**./docker-compose.yml**
```
...
  script_exporter:
    build:
      context: ./script_exporter
      dockerfile: Dockerfile
    container_name: script_exporter
    restart: unless-stopped
    volumes:
     - ./script_exporter/script_exporter.yaml:/script_exporter.yaml
     - ./script_exporter/scripts/:/scripts/:ro
    ports:
      - 9469:9469
    networks:
      - default
...
```

**./script_exporter/** (git clone https://github.com/ricoberger/script_exporter)

**./script_exporter/Dockerfile**
```
FROM golang:1.22.0-alpine3.18 as build
RUN apk update && apk add git make
RUN mkdir /build
WORKDIR /build
COPY . .
RUN export CGO_ENABLED=0 && make build

FROM jacobalberty/firebird:3.0
COPY --from=build /build/bin/script_exporter /bin/script_exporter
RUN apt-get update && apt-get install -y software-properties-common python3-dev python3-pip
RUN pip install firebird-driver
ENV TINI_VERSION v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini
EXPOSE 9469
ENTRYPOINT ["/tini", "--"]

CMD ["/bin/script_exporter", "-config.file", "/script_exporter.yaml"]
```

**./script_exporter/script_exporter.yaml**
```
scripts:
  - name: firebird
    script: python3 scripts/firebird_exporter.py
```

**./script_exporter/scripts/firebird_exporter.py**

```
#!/usr/bin/env python

__author__ = "Lukáš Drahník"
__credits__ = ["Lukáš Drahník", "Marek Křiklán"]
__license__ = "MIT"
__version__ = "0.0.1"
__maintainer__ = "Lukáš Drahník"
__email__ = "ldrahnik@gmail.com"
__status__ = "Production"

import os
import sys
import subprocess
from firebird.driver import driver_config, connect

__location__ = os.path.realpath(
    os.path.join(os.getcwd(), os.path.dirname(__file__)))

prefix = 'fb_'

db_host = ''
db_name = ''
db_user = ''
db_password = ''

db_uni_man_file = 'dbs.csv'

def find_db_user_and_pass(db_host_and_name, db_file):

   db_user = None
   db_pass = None

   with open(os.path.join(__location__, db_file)) as f:

      k = f.read()
      proc_grep = subprocess.run(
         ['grep', db_host_and_name + ","],
         input=k,
         stdout=subprocess.PIPE,
         encoding='ascii'
      )

      if proc_grep.returncode == 0:

         proc_user = subprocess.run(
            ['cut', '-f2', '-d', ','],
            input=proc_grep.stdout,
            stdout=subprocess.PIPE,
            encoding='ascii'
         )
         db_user = proc_user.stdout.strip()

         proc_pass = subprocess.run(
            ['cut', '-f3', '-d', ','],
            input=proc_grep.stdout,
            stdout=subprocess.PIPE,
            encoding='ascii'
         )
         db_pass = proc_pass.stdout.strip()

   return [db_user, db_pass]


if len(sys.argv) > 1:

   db_host_and_name = sys.argv[1]

   db_host = db_host_and_name.split(':')[0]
   db_name = db_host_and_name.split(':')[1]

   [db_user, db_password] = find_db_user_and_pass(db_host_and_name, db_uni_man_file)

else:
   print("First argument has to be in format <db_host:db_name> (e.g. fbserver.company.com:db_name).")
   exit(1)

def print_counter(cursor):
   global prefix

   for row in cursor.fetchall():
      print(prefix + row[0] + str(row[1]))

def print_vector(cursor, value = 1):
   global prefix, db_host, db_name

   for row in cursor.fetchall():

      vector_names_and_values = [field[0].lower() + '="' + str(row[i]).replace('\r\n', '').replace('\n', '').replace('"', '\\"') + '"' for i, field in enumerate(cursor.description) if i > 0 and not field[0].lower().startswith('as_value')]
      vector_as_value = [str(row[i]) for i, field in enumerate(cursor.description) if i > 0 and field[0].lower().startswith('as_value')]

      if len(vector_as_value):
         value = vector_as_value[0]

      print(prefix + row[0] + "{" + ','.join(vector_names_and_values) + "} " + str(value))

driver_config.server_defaults.host.value = db_host

#print(db_host, db_name, db_user, db_password) # testing purpose only
with connect(db_name, user=db_user, password=db_password) as con:

   cursor = con.cursor()
   try:
      cursor.execute("""SELECT 'active_transactions' as counter_name, count(*) as cntr_value
            FROM MON$TRANSACTIONS
            WHERE MON$STATE = 1
            UNION
            SELECT 'open_connections' as counter_name, count(*) as cntr_value
            FROM MON$ATTACHMENTS
            UNION
            SELECT 'idle_connections' as counter_name, count(*) as cntr_value
            FROM MON$ATTACHMENTS
            WHERE MON$STATE = 0
            UNION
            SELECT 'database_state' as counter_name, MON$SHUTDOWN_MODE as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'pages_allocated_externally' as counter_name, MON$PAGES as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'page_fetches' as counter_name, MON$PAGE_FETCHES as cntr_value
            FROM MON$IO_STATS
            WHERE MON$STAT_GROUP = 0
            UNION
            SELECT 'page_writes' as counter_name, MON$PAGE_WRITES as cntr_value
            FROM MON$IO_STATS
            WHERE MON$STAT_GROUP = 0
            UNION
            SELECT 'database_memory_used' as counter_name, MON$MEMORY_USED as cntr_value
            FROM MON$MEMORY_USAGE
            WHERE MON$STAT_GROUP = 0
            UNION
            SELECT 'database_page_size' as counter_name, MON$PAGE_SIZE as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'database_allocated_pages' as counter_name, MON$PAGE_BUFFERS as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'stalled_statements' as counter_name, count(*) as cntr_value
            FROM MON$STATEMENTS
            WHERE MON$STATE = 2
            UNION
            SELECT 'running_statements' as counter_name, count(*) as cntr_value
            FROM MON$STATEMENTS
            WHERE MON$STATE = 1 AND MON$ATTACHMENT_ID <> CURRENT_CONNECTION
            UNION
            SELECT 'oldest_active' as counter_name, MON$OLDEST_ACTIVE as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'oldest_snapshot' as counter_name, MON$OLDEST_SNAPSHOT as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'next_transaction' as counter_name, MON$NEXT_TRANSACTION as cntr_value
            FROM MON$DATABASE""")
   except:

      # old fb version without table MON$MEMORY_USAGE
      cursor.execute("""SELECT 'active_transactions' as counter_name, count(*) as cntr_value
            FROM MON$TRANSACTIONS
            WHERE MON$STATE = 1
            UNION
            SELECT 'open_connections' as counter_name, count(*) as cntr_value
            FROM MON$ATTACHMENTS
            UNION
            SELECT 'idle_connections' as counter_name, count(*) as cntr_value
            FROM MON$ATTACHMENTS
            WHERE MON$STATE = 0
            UNION
            SELECT 'database_state' as counter_name, MON$SHUTDOWN_MODE as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'pages_allocated_externally' as counter_name, MON$PAGES as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'page_fetches' as counter_name, MON$PAGE_FETCHES as cntr_value
            FROM MON$IO_STATS
            WHERE MON$STAT_GROUP = 0
            UNION
            SELECT 'page_writes' as counter_name, MON$PAGE_WRITES as cntr_value
            FROM MON$IO_STATS
            WHERE MON$STAT_GROUP = 0
            UNION
            SELECT 'database_page_size' as counter_name, MON$PAGE_SIZE as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'database_allocated_pages' as counter_name, MON$PAGE_BUFFERS as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'stalled_statements' as counter_name, count(*) as cntr_value
            FROM MON$STATEMENTS
            WHERE MON$STATE = 2
            UNION
            SELECT 'running_statements' as counter_name, count(*) as cntr_value
            FROM MON$STATEMENTS
            WHERE MON$STATE = 1 AND MON$ATTACHMENT_ID <> CURRENT_CONNECTION
            UNION
            SELECT 'oldest_active' as counter_name, MON$OLDEST_ACTIVE as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'oldest_snapshot' as counter_name, MON$OLDEST_SNAPSHOT as cntr_value
            FROM MON$DATABASE
            UNION
            SELECT 'next_transaction' as counter_name, MON$NEXT_TRANSACTION as cntr_value
            FROM MON$DATABASE""")

   print_counter(cursor)

   # vectors
   cursor.execute("""SELECT 'transaction_duration_seconds' AS vector_name, mt.MON$TRANSACTION_ID AS transaction_id,CAST(mt.MON$TIMESTAMP as TIMESTAMP) as transaction_timestamp, DATEDIFF(second, mt.MON$TIMESTAMP, current_timestamp) AS as_value, ma.MON$REMOTE_ADDRESS as REMOTE_ADDRESS, ma.MON$REMOTE_PROCESS as PROCESS, s.MON$SQL_TEXT as SQL_TEXT FROM MON$ATTACHMENTS ma JOIN MON$TRANSACTIONS mt ON ma.MON$ATTACHMENT_ID = mt.MON$ATTACHMENT_ID LEFT JOIN MON$STATEMENTS s ON ma.MON$ATTACHMENT_ID  = s.MON$ATTACHMENT_ID WHERE ma.MON$STATE = 1 AND ma.MON$ATTACHMENT_ID <> CURRENT_CONNECTION AND DATEDIFF(second, mt.MON$TIMESTAMP, current_timestamp) > 180""")
   print("# transactions are displayed only with duration > 3 mins")
   print_vector(cursor)

   cursor.execute("""SELECT 'info' as vector_name, RDB$GET_CONTEXT('SYSTEM', 'ENGINE_VERSION') as version FROM RDB$DATABASE""")
   print_vector(cursor)
```

**./script_exporter/scripts/dbs.csv**

```
...
# example 1
fbserver.company.com:/data/vol2/data/db_name.gdb,sysdba,pass
# example 2
fbserver.company.com:db_name,sysdba,pass
...
```

## 2. Monitoring probe (backup metrics)

```
├── filestat.yaml
├── web-config.yml
└── filestat_exporter.exe
```

**filestat.yaml**
```
exporter:

  working_directory: "\\\\xyz\\backups\\"
  enable_crc32_metric: true

  files:
      - patterns: ['*.log']
```

**web-config.yml**
```
basic_auth_users:
  admin: <bcrypt-hash-of-pass>
```


## 3. Prometheus

**prometheus.yml**
```
...
  - job_name: filestat_exporter
    metrics_path: /metrics
    static_configs:
      - targets: [ 'xyz:9943' ]
    basic_auth:
      username: 'admin'
      password: 'pass'

  - job_name: 'script_exporter'
    metrics_path: /probe
    params:
      script: [firebird]
      params: [target]
    static_configs:
      - targets:
         - fbserver.company.com:/data/vol2/data/db_name.gdb
         - fbserver.company.com:db_name
    basic_auth:
      username: 'admin'
      password: 'pass'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: xyz:9469
      - source_labels: [__param_target]
        target_label: target
      - source_labels: [__param_target]
        target_label: instance
```
**alert_rules.yml**
```
...
  - alert: FirebirdBackupCheck
    expr: changes(file_content_hash_crc32{path=~".log"}[25h]) == 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: Firebird backup was not created for more than 24h (instance {{ $labels.instance }})
      description: ""

  - alert: FirebirdOldestActiveNextTransactionCheck
    expr: fb_next_transaction - fb_oldest_active > 20000000
    for: 1h
    labels:
      severity: critical
    annotations:
      summary: Firebird next transaction is not equal to the oldest active (instance {{ $labels.instance }})
      description: ""

  - alert: FirebirdOldestActiveOldestSnapshotEqualityCheck
    expr: fb_oldest_active - fb_oldest_snapshot > 1000000000000000 # only testing, has to be set up zero
    for: 1h
    labels:
      severity: critical
    annotations:
      summary: Firebird oldest active is not equal to the oldest snapshot (instance {{ $labels.instance }})
      description: ""

  - alert: FirebirdTransactionDuration
    expr: fb_transaction_duration_seconds > 1440000
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: Firebird transaction (transaction {{ $labels.transaction_id }}, instance {{ $labels.instance }})
      description: "The transaction duration > 4h"
...
```


## 4. Grafana dashboard

[dashboard.json](https://github.com/ldrahnik/firebird_exporter/blob/main/dashboard.json)
