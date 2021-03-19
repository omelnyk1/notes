# Fluentd agent for GCE virtual machine
## Problem
We have customized MongoDB cluster based on GCE virtual machines (RHEL 8). We need to install fluentd agents on it and publish logs to Google Logging service.

## Steps
* Make sure IAM service account, assigned on VMs, has pernmissions to write logs. For instance, predefined role [roles/logging.logWriter](https://cloud.google.com/iam/docs/understanding-roles).
* Login on virtual machine
* Add Google Fluentd repo:
```sh
$ curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh
$ bash add-logging-agent-repo.sh
```
* Install fluentd agent:
```sh
$ yum install -y google-fluentd-1.8.4
```
* Create folder for configuration files:
```sh
$ mkdir -p /etc/google-fluentd/config.d/
```
* Set configuration file in /etc/google-fluentd/google-fluentd.conf:
```xml
@include config.d/*.conf

 # Do not collect fluentd's own logs to avoid infinite loops.
<match fluent.**>
   @type null
</match>

# Add a unique insertId to each log entry that doesn't already have it.
# This helps guarantee the order and prevent log duplication.
<filter **>
   @type add_insert_ids
</filter>

# Configure all sources to output to Google Cloud Logging
<match **>
    @type google_cloud
    buffer_type file
    buffer_path /var/log/google-fluentd/buffers
    # Set the chunk limit conservatively to avoid exceeding the recommended
    # chunk size of 5MB per write request.
    buffer_chunk_limit 512KB
    # Flush logs every 5 seconds, even if the buffer is not full.
    flush_interval 5s
    # Enforce some limit on the number of retries.
    disable_retry_limit false
    # After 3 retries, a given chunk will be discarded.
    retry_limit 3
    # Wait 10 seconds before the first retry. The wait interval will be doubled on
    # each following retry (20s, 40s...) until it hits the retry limit.
    retry_wait 10
    # Never wait longer than 5 minutes between retries. If the wait interval
    # reaches this limit, the exponentiation stops.
    # Given the default config, this limit should never be reached, but if
    # retry_limit and retry_wait are customized, this limit might take effect.
    max_retry_wait 300
    # Use multiple threads for processing.
    num_threads 8
    # Use the gRPC transport.
    use_grpc true
    # If a request is a mix of valid log entries and invalid ones, ingest the
    # valid ones and drop the invalid ones instead of dropping everything.
    partial_success true
</match>
```
* Set up /etc/google-fluentd/config.d/syslog.conf for parse system logs:
```xml
<source>
  @type tail

  # Parse the timestamp, but still collect the entire line as 'message'
  format /^(?<message>(?<time>[^ ]*\s*[^ ]* [^ ]*) .*)$/

  path /var/log/messages
  pos_file /var/lib/google-fluentd/pos/syslog.pos
  read_from_head true
  tag syslog
</source>
```
* Configure /etc/google-fluentd/config.d/syslog.conf to parse MongoDB logs. We gonna make transformation log keys from [standard format](https://docs.mongodb.com/manual/reference/log-messages/#json-log-output-format) to custom.
For instance, in Google Logging service instead of `"s": "E"` we will see `"severity": "ERROR"`:
```xml
<source>
  @type tail
  format json
  path /var/log/mongodb/*.log
  pos_file /var/lib/google-fluentd/pos/mongodb.pos
  read_from_head true
  tag mongodb
</source>
<filter mongodb>
  @type record_transformer
  enable_ruby
  <record>
    severity ${ if record["s"].match("D") then "DEBUG" elsif record["s"].match("I") then "INFO" elsif record["s"].match("W") then "WARNING" elsif record["s"].match("E") then "ERROR" elsif record["s"].match("F") then "FATAL" end }
    message ${record['msg']}
    component ${record['c']}
    context ${record['ctx']}
  </record>
  remove_keys s,msg,c,ctx
</filter>
```
* Enable and start SystemD unit:
```sh
$ systemctl enable google-fluentd
$ systemctl start google-fluentd
```
