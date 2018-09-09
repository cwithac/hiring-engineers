![datadog logo](https://i.imgur.com/4a0vj3W.png)

### Prerequisites - Setup The Environment

##### Vagrant

```shell
  $ vagrant init ubuntu/xenial64
  $ vagrant up
  $ vagrant ssh
```

##### Datadog and Agent Reporting Metrics

```shell
$ DD_API_KEY=<YOUR-API-KEY> bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"

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

> Once this is created, access the Dashboard from your Dashboard List in the UI:
> - Set the Timeboard's timeframe to the past 5 minutes

> - Take a snapshot of this graph and use the @ notation to send it to yourself.

> - Bonus Question: What is the Anomaly graph displaying?

<hr>

### Monitoring Data

> Since you’ve already caught your test metric going above 800 once, you don’t want to have to continually watch this dashboard to be alerted when it goes above 800 again. So let’s make life easier by creating a monitor.

> Create a new Metric Monitor that watches the average of your custom metric (my_metric) and will alert if it’s above the following values over the past 5 minutes:

> - Warning threshold of 500
> - Alerting threshold of 800
> - And also ensure that it will notify you if there is No Data for this query over the past 10m.

> Please configure the monitor’s message so that it will:
> - Send you an email whenever the monitor triggers.
> - Create different messages based on whether the monitor is in an Alert, Warning, or No Data state.
> - Include the metric value that caused the monitor to trigger and host ip when the Monitor triggers an Alert state.

> - When this monitor sends you an email notification, take a screenshot of the email that it sends you.

> Bonus Question: Since this monitor is going to alert pretty often, you don’t want to be alerted when you are out of the office. Set up two scheduled downtimes for this monitor:

> - One that silences it from 7pm to 9am daily on M-F,

> - And one that silences it all day on Sat-Sun.

> - Make sure that your email is notified when you schedule the downtime and take a screenshot of that notification.

<hr>

### Collecting APM Data

Given the following Flask app (or any Python/Ruby/Go app of your choice) instrument this using Datadog’s APM solution:

<details><summary>EXPAND</summary>
<p>

```python
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

</p>

> Note: Using both ddtrace-run and manually inserting the Middleware has been known to cause issues. Please only use one or the other.

</details>

> Bonus Question: What is the difference between a Service and a Resource?

> Provide a link and a screenshot of a Dashboard with both APM and Infrastructure Metrics.
> Please include your fully instrumented app in your submission, as well.

<hr>
