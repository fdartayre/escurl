# Description

`escurl` is a Bash script acting as a command line wrapper around `curl` to interact with the REST APIs of Elasticsearch and Kibana. All `curl` options are supported but calling multiple APIs on the same command line is not supported.

# Configuration

## Profiles

Profiles are plaintext files stored in the home directory of `escurl` (`~/.escurl` by default). Each profile contains the parameters to connect to Elasticsearch and/or Kibana. By default `escurl` reads a configuration file named `default`.

### Environment variables

The location of the home directory can be changed via the `ES_CURL_HOME` environment variable and the configuration file can be changed via `ES_CURL_PROFILE`. The profile can also be overwritten with the `--profile <name>` command line option.

### Profile content

Profiles contain settings in the form of `setting=value`. Recognized settings are:

- `elasticsearch`:  No default value
- `kibana`:  No default value
- `user`: No default value
- `password`: No default value
- `options`: No default value

All the settings are optional. Here's an example of a configuration file:
```
elasticsearch=https://localhost:9200
kibana=https://localhost:5601
user=elastic
password=changeme
options="-k"
```

# Usage
```
escurl [options...] [API|URL]
```

## API, URL and options

The `escurl` command line options all start with `-` and the list of recognized options can be found below. Other parameters are either interpreted as an API, a URL, a `curl` parameter or a JSON / NDJSON payload.

- A command line parameter that starts with `http[s]://` or contains a `<hostname>:<port>` is interpreted as a URL. As in `curl`, you can specify multiple parts of URLs by writing part sets within braces as in `http://site.{one,two,three}.com`. Multiple URLs are not supported though, and a URL cannot start with `{`.
- A command line parameter that starts with `-` is either a `escurl` parameter if it is listed below or interpreted as a `curl` parameter otherwise. All curl parameters are supported.
- A command line parameter that starts with `{` is interpreted as a JSON (or NDJSON) payload to be sent to the server. Similar to data payload in `curl`, if it starts with the letter `@`, the rest should be a filename. If the first character in the file is `{`, the file content is also interpreted as JSON (or NDJSON) payload.
- Otherwise, the command line parameter is interpreted as an API to be called on Elasticsearch or Kibana (if the API starts with `kbn:` or if `--kibana` was specified).

### Options

`--config`

&nbsp;&nbsp;
Print the current configuration. Can be combined with `--profiles` to show the content of all the profiles.

`-h, --help`

&nbsp;&nbsp;
Usage help.

`--kibana`

&nbsp;&nbsp;
Use Kibana endoint instead of Elasticsearch. Same as prefixing the API with `kbn:`.

`--ndjson`

&nbsp;&nbsp;
Use NDJSON content type instead of JSON. It only is effective when a JSON payload has been detected.

`--no-pretty`

&nbsp;&nbsp;
Disable pretty results. By default pretty results are enabled for requests sent to Elasticsearch.

`--print`

&nbsp;&nbsp;
Print the `curl` command instead of executing it.

`--profile <name>`

&nbsp;&nbsp;
Overwrite the configured/default profile.

`--profiles`

&nbsp;&nbsp;
Print the list of profiles in `escurl` home directory. This can be combined with `--config` to display the content of the profiles as well (password will be printed too).

`--show-pwd`

&nbsp;&nbsp;
By default `--config` and `--print` options don't show the password configured in the profiles. This option prints the actual configured password.

# Workflow examples

Profiles offer a convenient way to store the credentials of deployments. It can be useful to adopt a naming convention and prefix the profiles with `ess_`, `eck_` or `local_` followed by the name of the deployment.

The following command lists the profiles in the `escurl` home directory:
```bash
$ escurl --profiles
default
default-apikey
eck_simple
ess_apm
ess_security
ess_version_8
local_ssl
```
## Elasticsearch 
To work with a specific cluster, say `ess_security`, we can export `ES_CURL_PROFILE`:
```bash
$ export ES_CURL_PROFILE=ess_security
$ escurl --config
--
ES_CURL_HOME=
ES_CURL_PROFILE=ess_security
--
path=/Users/fred/.escurl/ess_security
--
elasticsearch=https://security-45dbac.es.us-central1.gcp.cloud.es.io:9243
kibana=https://security-45dbac.kb.us-central1.gcp.cloud.es.io:9243
user=fred
password=<secret>
options=
```
Further `escurl` commands are directly issued to this cluster:
```bash
$ escurl
{
  "name" : "instance-0000000005",
  "cluster_name" : "a9e5c54f660248e1a65eaffb8fce8ff6",
  "cluster_uuid" : "d3fLC3UFSZuSf0lE9dPyEQ",
  "version" : {
    "number" : "8.1.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "39afaa3c0fe7db4869a161985e240bd7182d7a07",
    "build_date" : "2022-04-19T08:13:25.444693396Z",
    "build_snapshot" : false,
    "lucene_version" : "9.0.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
```bash
$ escurl _cluster/health
{
  "cluster_name" : "a9e5c54f660248e1a65eaffb8fce8ff6",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 570,
  "active_shards" : 1140,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
To print the `curl` command instead of executing it, add `--print`:
```bash
$ escurl _cluster/health --print
curl -u'fred:<secret>' "https://security-45dbac.es.us-central1.gcp.cloud.es.io:9243/_cluster/health?pretty"
```
To illustrate how to send a JSON payload, let's create a user:
```
$ escurl -XPOST /_security/user/jacknich/ '{
  "password" : "l0ng-r4nd0m-p@ssw0rd",
  "roles" : [ "admin", "other_role1" ],
  "full_name" : "Jack Nicholson",
  "email" : "jacknich@example.com",
  "metadata" : {
    "intelligence" : 7
  }
}'
```
The generated `curl`command is:
```bash
curl -X'POST' -H'Content-Type: application/json' -d '{
  "password" : "l0ng-r4nd0m-p@ssw0rd",
  "roles" : [ "admin", "other_role1" ],
  "full_name" : "Jack Nicholson",
  "email" : "jacknich@example.com",
  "metadata" : {
    "intelligence" : 7
  }
}' -u'fred:<secret>' "https://security-45dbac.es.us-central1.gcp.cloud.es.io:9243/_security/user/jacknich?pretty"
```
Sending bulk requests from a file named `requests` will also be quite shorter than the corresponding curl command:
```bash
$ escurl -XPOST my_index/_bulk @requests --ndjson
```
The generated `curl` command is:
```
curl -X'POST' -H'Content-Type: application/x-ndjson' --data-binary '@requests' -u'fred:<secret>' "https://security-45dbac.es.us-central1.gcp.cloud.es.io:9243/my_index/_bulk?pretty"
```
## Kibana
To send requests to Kibana instead of Elasticsearch we can either use the `--kibana` option or prefix the API with `kbn:`:
```
escurl kbn:/api/index_management/indices
```
Executes this command:
```bash
curl -H'kbn-xsrf: true' -u'fred:l0ng-r4nd0m-p@ssw0rd' "https://security-45dbac.kb.us-central1.gcp.cloud.es.io:9243/api/index_management/indices"
```
