#### 1. Launch a VM in GCE and SSH into it
We will use that VM to simulate a MySQL DB using a Docker containers, and we will run our binglog listner there

#### 2. Install dependencies packages and Docker
```
sudo apt-get install python-pip 
pip install google-cloud-pubsub mysql-replication google.cloud.bigquery
sudo apt-get install -y docker.io
```

#### 3. Configure MySQL DB 
```
mkdir /tmp/mysql_conf/
```
```
cat << EOF > /tmp/mysql_conf/my_sql.cnf
[mysqld]
log_bin=/tmp/binlog
binlog_format=ROW
server-id=100
EOF
```

#### 4. Start MySQL DB
```
docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=my-secret-pw -d -v /tmp/mysql_conf/:/etc/mysql/conf.d -p 3307:3306 mysql:5.7
```


#### 5. Verify it works 
Create a DB 
```
docker run -it --link mysql-db:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" -e"create database demo;"'
```
Display DBs 
```
docker run -it --link mysql-db:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"show databases;"'
```
Delete a DB 
```
docker run -it --link mysql-db:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"drop database demo;"'
```
#### 7. Create a PubSub Topic 
Create a PubSub topic in which we'll write any MySQL changes 
```
gcloud beta pubsub topics create sql-topic
```

#### 8. Create a BigQuery Dataset 
Create a BigQuery Dataset to receive the MySQL data 

Create the schema
```
bq_schemafile = open("bq_schema.json","w")
bq_schemafile.write("""
[
  {
    "name": "schema",
    "type": "STRING"
  },
  {
    "name": "pos",
    "type": "INTEGER"
  },
  {
    "name": "table",
    "type": "STRING"
  },
  {
    "name": "type",
    "type": "STRING"
  },
  {
    "fields": [
      {
        "fields": [
          {
            "name": "id",
            "type": "INTEGER"
          },
          {
            "name": "name",
            "type": "STRING"
          }
        ],
        "name": "values",
        "type": "RECORD"
      },
      {
        "fields": [
          {
            "name": "id",
            "type": "INTEGER"
          },
          {
            "name": "name",
            "type": "STRING"
          }
        ],
        "name": "before_values",
        "type": "RECORD"
      },
      {
        "fields": [
          {
            "name": "id",
            "type": "INTEGER"
          },
          {
            "name": "name",
            "type": "STRING"
          }
        ],
        "name": "after_values",
        "type": "RECORD"
      }
    ],
    "name": "row",
    "type": "RECORD"
  }
]
""")
bq_schemafile.close()
```
Create the dataset 
```
bq mk --schema bq_schema.json demo.mysql_replication_table 
```

#### 9. Create a Dataflow Pipeline 
Launch a Dataflow stream pipeline that will read from PubSub any BinLog events, and will write this changes to BigQuery 
```
gcloud dataflow jobs run pubsub-to-bq-pipeline-demo \
    --gcs-location gs://dataflow-templates/latest/PubSub_to_BigQuery \
    --parameters \
inputTopic=projects/sql-to-bq-demo/topics/sql-topic,\
outputTableSpec=sql-to-bq-demo:demo.mysql_replication_table
```

#### 10. Configure the listener 
This is a short Python code that listens to the MySQL BinLog, picks up any changes and write them to a PubSub topic. 
```
cat << EOF > binlog-listener.py
import json
import os
from google.cloud import pubsub
from pymysqlreplication import BinLogStreamReader
from pymysqlreplication.row_event import (
  DeleteRowsEvent,
  UpdateRowsEvent,
  WriteRowsEvent,
)

stream = BinLogStreamReader(
    connection_settings= {
      "host": "localhost",
      "port": 3307,
      "user": "root",
      "passwd": "my-secret-pw"},
    server_id=100,
    blocking=True,
    resume_stream=True,
    only_events=[DeleteRowsEvent, WriteRowsEvent, UpdateRowsEvent])

publisher = pubsub.PublisherClient()
topic_name = 'projects/{project_id}/topics/{topic}'.format(
    project_id='sql-to-bq-demo',
    topic='sql-to-bq-topic',  # Set this to something appropriate.
)

for binlogevent in stream:
    ##binlogevent.dump()
    for row in binlogevent.rows:
      event = {"schema": binlogevent.schema,
      "table": binlogevent.table,
      "type": type(binlogevent).__name__,
      "row": row,
      "pos": stream.log_pos,
      ##"time": binlogevent.date
      }
    event_json = json.dumps(event)
    print(event_json)
    publisher.publish(topic_name, event_json.encode())
EOF
```

Run the listner
```
python binlog-listener.py
```

#### 11. Generate Data in MySQL DB 
Create a DB
```
docker run -it --link mysql-db:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" -e"create database demo;"'
```

Delete a DB
```
docker run -it --link mysql-db:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"drop database demo;"'
```

Create a table
```
docker run -it --link some-mysql:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"use demo ; create table demo.test (id int,name varchar(40), PRIMARY KEY(id));"'
```

Insert a row
```
docker run -it --link some-mysql:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"insert into demo.test (id,name) values (1,\"value1\");"'
```
```
docker run -it --link some-mysql:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"insert into demo.test (id,name) values (2,\"value2\");"'
```
```
docker run -it --link some-mysql:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"insert into demo.test (id,name) values (3,\"value3\");"'
```
Update a row 
```
docker run -it --link some-mysql:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"update demo.test SET name=\"value12345\" WHERE id=1;"'
```

Review what is in the MySQL DB 
```
docker run -it --link some-mysql:mysql --rm mysql sh -c \
    'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" \
    -e"SELECT * FROM demo.test;"' 
```

#### 12. Look at BigQuery 


