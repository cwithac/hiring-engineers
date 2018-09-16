![datadog logo](https://i.imgur.com/4a0vj3W.png)

- [Prerequisites - Setup The Environment](https://github.com/cwithac/hiring-engineers/blob/CWRIGHT_xenial64/answers.md#prerequisites---setup-the-environment)
- [Collecting Metrics](https://github.com/cwithac/hiring-engineers/blob/CWRIGHT_xenial64/answers.md#collecting-metrics)
- [Visualizing Data](https://github.com/cwithac/hiring-engineers/blob/CWRIGHT_xenial64/answers.md#visualizing-data)
- [Collecting APM Data](https://github.com/cwithac/hiring-engineers/blob/CWRIGHT_xenial64/answers.md#collecting-apm-data)

### Prerequisites - Setup The Environment

##### Vagrant

```shell
  $ vagrant init ubuntu/xenial64
  $ vagrant up
  $ vagrant ssh
```

##### Datadog and Agent Reporting Metrics

```shell
$ DD_API_KEY=<YOUR_API_KEY> bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"

$ sudo service datadog-agent restart
```

<hr>

### Collecting Metrics

> Add tags in the Agent config file and show us a screenshot of your host and its tags on the Host Map page in Datadog.

```shell
  $ sudo nano /etc/datadog-agent/datadog.yaml
  $ sudo service datadog-agent restart
```

```yaml
# Set the host's tags (optional)
tags: os:high_sierra, year:2013
```

> Install a database on your machine (MongoDB, MySQL, or PostgreSQL) and then install the respective Datadog integration for that database.

```shell
$ sudo apt-get update
$ sudo apt-get install mysql-server
$ sudo ufw allow mysql
$ sudo systemctl start mysql
$ sudo systemctl enable mysql
$ /usr/bin/mysql -u root -p
```
```sql
CREATE USER 'datadog'@'localhost' IDENTIFIED BY '<PASSWORD>';
GRANT REPLICATION CLIENT ON *.* TO 'datadog'@'localhost' WITH MAX_USER_CONNECTIONS 5;
GRANT PROCESS ON *.* TO 'datadog'@'localhost';
GRANT SELECT ON performance_schema.* TO 'datadog'@'localhost';
```

```shell
$ mysql -u datadog --password='<PASSWORD>' -e "show status" | \
grep Uptime && echo -e "\033[0;32mMySQL user - OK\033[0m" || \
echo -e "\033[0;31mCannot connect to MySQL\033[0m"
mysql -u datadog --password='<PASSWORD>' -e "show slave status" && \
echo -e "\033[0;32mMySQL grant - OK\033[0m" || \
echo -e "\033[0;31mMissing REPLICATION CLIENT grant\033[0m"

MySQL PROCESS grant - OK

$ mysql -u datadog --password='<PASSWORD>' -e "SELECT * FROM performance_schema.threads" && \
echo -e "\033[0;32mMySQL SELECT grant - OK\033[0m" || \
echo -e "\033[0;31mMissing SELECT grant\033[0m"
mysql -u datadog --password='<PASSWORD>' -e "SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST" && \
echo -e "\033[0;32mMySQL PROCESS grant - OK\033[0m" || \
echo -e "\033[0;31mMissing PROCESS grant\033[0m"

MySQL PROCESS grant - OK

$ sudo mv /etc/datadog-agent/conf.d/mysql.d/conf.yaml.example /etc/datadog-agent/conf.d/mysql.d/conf.yaml
$ sudo nano /etc/datadog-agent/conf.d/mysql.d/conf.yaml
```

```yaml
init_config:

instances:
  - server: localhost
    user: datadog
    pass: <PASSWORD>
    tags:
        - testing_db
        - testing_mysql
    options:
        replication: 0
        galera_cluster: 1
```

```shell
$ sudo service datadog-agent restart

$ sudo datadog-agent status

=========
Collector
=========

  Running Checks
  ==============

    mysql (1.3.0)
    -------------
      Total Runs: 2
      [...]
```

> Create a custom Agent check that submits a metric named my_metric with a random value between 0 and 1000.

```shell
$ sudo touch /etc/datadog-agent/conf.d/my_metric.yaml
$ sudo nano /etc/datadog-agent/conf.d/my_metric.yaml
```

```yaml
init_config:

instances:
  [{}]
```

```shell
$ sudo touch /etc/datadog-agent/checks.d/my_metric.py
$ sudo nano /etc/datadog-agent/checks.d/my_metric.py
```

```python
from checks import AgentCheck
import random

class MyMetric(AgentCheck):
  def check(self, instance):
    self.gauge('my_metric', random.randint(0,1000))
```

```shell
$ sudo service datadog-agent restart
```

> Change your check's collection interval so that it only submits the metric once every 45 seconds.

```shell
$ sudo nano /etc/datadog-agent/conf.d/my_metric.yaml
```

```yaml
init_config:

instances:
    -   min_collection_interval: 45
```

```shell
$ sudo service datadog-agent restart
```

<hr>

### Visualizing Data

> Utilize the Datadog API to create a Timeboard that contains:
> - Your custom metric scoped over your host.
> - Any metric from the Integration on your Database with the anomaly function applied.
> - Your custom metric with the rollup function applied to sum up all the points for the past hour into one bucket

```shell
$ curl "https://api.datadoghq.com/api/v1/validate?api_key=<YOUR_API_KEY>"

{"valid":true}

$ api_key=<YOUR_API_KEY>
$ app_key=<YOUR_APPLICATION_KEY>

$ curl  -X POST -H "Content-type: application/json" \
-d '{
      "graphs" : [
      {
          "title": "scoped over host",
          "definition": {
              "events": [],
              "requests": [
                {"q": "avg:my_metric{*}"}
              ]
          },
          "viz": "timeseries"
      },
      {
          "title": "integration of database with anomaly function applied",
          "definition": {
              "events": [],
              "requests": [
                {"q": "avg:mysql.performance.bytes_sent{*}"}
              ]
          },
          "viz": "timeseries"
      },
      {
          "title": "rollup function applied",
          "definition": {
              "events": [],
              "requests": [
                {"q": "avg:my_metric{*}.rollup(sum, 3600)"}
              ]
          },
          "viz": "timeseries"
      }
      ],
      "title" : "My_Metric Timeboard 2.0",
      "description" : "A dashboard with my_metric custom agent.",
      "template_variables": [{
          "name": "host1",
          "prefix": "host",
          "default": "host:my-host"
      }],
      "read_only": "True"
}' \
"https://api.datadoghq.com/api/v1/dash?api_key=${api_key}&application_key=${app_key}"

{"dash":{"read_only":true,"graphs":[{"definition":{"requests":[{"q":"avg:my_metric{*}"}],"events":[]},"title":"scoped over host"},{"definition":{"requests":[{"q":"avg:mysql.performance.bytes_sent{*}"}],"events":[]},"title":"integration of database with anomaly function applied"},{"definition":{"requests":[{"q":"avg:my_metric{*}.rollup(sum, 3600)"}],"events":[]},"title":"rollup function applied"}],"template_variables":[{"default":"host:my-host","prefix":"host","name":"host1"}],"description":"A dashboard with my_metric custom agent.","title":"My_Metric Timeboard 2.0","created":"<DATETIME REDACTED>","id":910540,"created_by":{"disabled":false,"handle":"cathleenmwright@gmail.com","name":"Cathleen Wright","is_admin":true,"role":"Administrator","access_role":"adm","verified":true,"email":"<REDACTED>","icon":"<REDACTED>"},"modified":"<DATETIME REDACTED>"},"url":"/dash/910540/mymetric-timeboard-20","resource":"/api/v1/dash/910540"}
```

```json
{"errors": ["Error parsing query: unable to parse anomalies(avg:mysql.performance.bytes_sent{*}, basic, 2): Rule 'scope_expr' didn't match at ', 2)' (line 1, column 53)."]}

"requests": [
    {"q": "anomalies(avg:mysql.performance.bytes_sent{*}, 'basic', 2)"}
  ]
```

<hr>

### Collecting APM Data

Given the following Flask app (or any Python/Ruby/Go app of your choice) instrument this using Datadog’s APM solution:

```python
# /etc/datadog-agent/my_app.py

from flask import Flask
import logging
import sys

# Have flask use stdout as the logger
main_logger = logging.getLogger()
main_logger.setLevel(logging.DEBUG)
c = logging.StreamHandler(sys.stdout)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
c.setFormatter(formatter)
main_logger.addHandler(c)

app = Flask(__name__)

@app.route('/')
def api_entry():
    return 'Entrypoint to the Application'

@app.route('/api/apm')
def apm_endpoint():
    return 'Getting APM Started'

@app.route('/api/trace')
def trace_endpoint():
    return 'Posting Traces'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port='5050')
```

```yaml
# /etc/datadog-agent/datadog.yaml

apm_config:
 enabled: true
```

```shell
$ sudo apt-get update
$ sudo apt install python3-pip
$ pip3 install ddtrace
$ pip3 install flask
$ sudo service datadog-agent restart
$ ddtrace-run python3 my_app.py

DEBUG:ddtrace.contrib.flask.middleware:flask: initializing trace middleware
<DATETIME REDACTED> - ddtrace.contrib.flask.middleware - DEBUG - flask: initializing trace middleware
DEBUG:ddtrace.writer:resetting queues. pids(old:None new:2310)
<DATETIME REDACTED> - ddtrace.writer - DEBUG - resetting queues. pids(old:None new:2310)
DEBUG:ddtrace.writer:starting flush thread
<DATETIME REDACTED> - ddtrace.writer - DEBUG - starting flush thread
 * Serving Flask app "my_app" (lazy loading)
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
INFO:werkzeug: * Running on http://0.0.0.0:5050/ (Press CTRL+C to quit)
<DATETIME REDACTED> - werkzeug - INFO -  * Running on http://0.0.0.0:5050/ (Press CTRL+C to quit)
DEBUG:ddtrace.api:reported 1 services
<DATETIME REDACTED> - ddtrace.api - DEBUG - reported 1 services
[...]

$ curl http://127.0.0.1:5050/
Entrypoint to the Application
$ curl http://127.0.0.1:5050/api/apm
Getting APM Started
$ curl http://127.0.0.1:5050/api/trace
Posting Traces

INFO:werkzeug:127.0.0.1 - - [<DATETIME REDACTED>] "GET / HTTP/1.1" 200 -
<DATETIME REDACTED> - werkzeug - INFO - 127.0.0.1 - - [<DATETIME REDACTED>] "GET / HTTP/1.1" 200 -
DEBUG:ddtrace.api:reported 1 traces in 0.00134s
<DATETIME REDACTED> - ddtrace.api - DEBUG - reported 1 traces in 0.00134s
INFO:werkzeug:127.0.0.1 - - [<DATETIME REDACTED>] "GET /api/apm HTTP/1.1" 200 -
<DATETIME REDACTED> - werkzeug - INFO - 127.0.0.1 - - [<DATETIME REDACTED>] "GET /api/apm HTTP/1.1" 200 -
DEBUG:ddtrace.api:reported 1 traces in 0.00137s
<DATETIME REDACTED> - ddtrace.api - DEBUG - reported 1 traces in 0.00137s
INFO:werkzeug:127.0.0.1 - - [<DATETIME REDACTED>] "GET /api/trace HTTP/1.1" 200 -
<DATETIME REDACTED> - werkzeug - INFO - 127.0.0.1 - - [<DATETIME REDACTED>] "GET /api/trace HTTP/1.1" 200 -
DEBUG:ddtrace.api:reported 1 traces in 0.00130s
<DATETIME REDACTED> - ddtrace.api - DEBUG - reported 1 traces in 0.00130s
```

```yaml
# /etc/datadog-agent/datadog.yaml

apm_config:
 enabled: true
 analyzed_spans:
    flask|flask.request: 1
```

```shell
$ sudo service datadog-agent restart
```

![APM](https://i.imgur.com/pKqiuyx.png?1)

- [Tracing Python Applications](https://docs.datadoghq.com/tracing/setup/python/)
- [Flask](http://pypi.datadoghq.com/trace/docs/web_integrations.html#flask)
- [Datadog Python Trace Client](http://pypi.datadoghq.com/trace/docs/#module-ddtrace.contrib.flask)
- [APM Setup](https://docs.datadoghq.com/tracing/setup/)
