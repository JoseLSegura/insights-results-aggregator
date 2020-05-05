# Insights Results Aggregator

[![GoDoc](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator?status.svg)](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator)
[![Go Report Card](https://goreportcard.com/badge/github.com/RedHatInsights/insights-results-aggregator)](https://goreportcard.com/report/github.com/RedHatInsights/insights-results-aggregator)
[![Build Status](https://travis-ci.org/RedHatInsights/insights-results-aggregator.svg?branch=master)](https://travis-ci.org/RedHatInsights/insights-results-aggregator)
[![codecov](https://codecov.io/gh/RedHatInsights/insights-results-aggregator/branch/master/graph/badge.svg)](https://codecov.io/gh/RedHatInsights/insights-results-aggregator)

Aggregator service for insights results

## Description

Insights Results Aggregator is a service that provides Insight OCP data that are being consumed by OpenShift Cluster Manager. That data contain information about clusters status (especially health, security, performance) based on results generated by Insights rules engine. Insights OCP data are consumed from selected broker, stored in a storage (that basically works as a cache) and exposed via REST API endpoints.

## Architecture

Aggregator service consists of three main parts:

1. Consumer that reads (consumes) Insights OCP messages from specified message broker. Usually Kafka broker is used but it might be possible to develop a interface for different broker. Insights OCP messages are basically encoded in JSON and contain results generated by rule engine.

1. HTTP or HTTPS server that exposes REST API endpoints that can be used to read list of organizations, list of clusters, read rules results for selected cluster etc. Additionally, basic metrics are exposed as well. Those metrics is configured to be consumed by Prometheus and visualized by Grafana.

1. Storage backend which is some instance of SQL database. Currently SQLite3 and PostgreSQL are fully supported, but more SQL databases might be added later.

### Whole data flow

![data_flow](./doc/customer-facing-services-architecture.png)

1. Event about new data from insights operator is consumed from Kafka. That event contains (among other things) URL to S3 Bucket
2. Insights operator data is read from S3 Bucket and insigts rules are applied to that data
3. Results (basically organization ID + cluster name + insights results JSON) are stored back into Kafka, but into different topic
4. That results are consumed by Insights rules aggregator service that caches them
5. The service provides such data via REST API to other tools, like OpenShift Cluster Manager web UI, OpenShift console, etc.

Optionally, an organization whitelist can be enabled by the configuration variable `enable_org_whitelist`, which enables processing of a .csv file containing organization IDs (path specified by the config variable `org_whitelist`) and allows report processing only for these organizations. This feature is disbabled by default, and might be removed altogether in the near future.

### DB structure

#### Table report

This table is used as a cache for reports consumed from broker. Size of this
table (i.e. number of records) scales linearly with the number of clusters,
because only latest report for given cluster is stored (it is guarantied by DB
constraints). That table has defined compound key `org_id+cluster`,
additionally `cluster` name needs to be unique across all organizations.

```sql
CREATE TABLE report (
    org_id          INTEGER NOT NULL,
    cluster         VARCHAR NOT NULL UNIQUE,
    report          VARCHAR NOT NULL,
    reported_at     TIMESTAMP,
    last_checked_at TIMESTAMP,
    PRIMARY KEY(org_id, cluster)
)
```

### Tables rule and rule_error_key

These tables represent the content for Insights rules to be displayed by OCM.
The table `rule` represents more general information about the rule, whereas the `rule_error_key`
contains information about the specific type of error which occurred. The combination of these two create a unique rule.
Very trivialized example could be:

* rule "REQUIREMENTS_CHECK"
  * error_key "REQUIREMENTS_CHECK_LOW_MEMORY"
  * error_key "REQUIREMENTS_CHECK_MISSING_SYSTEM_PACKAGE"

```sql
CREATE TABLE rule (
    module        VARCHAR PRIMARY KEY,
    name          VARCHAR NOT NULL,
    summary       VARCHAR NOT NULL,
    reason        VARCHAR NOT NULL,
    resolution    VARCHAR NOT NULL,
    more_info     VARCHAR NOT NULL
)
```

```sql
CREATE TABLE rule_error_key (
    error_key     VARCHAR NOT NULL,
    rule_module   VARCHAR NOT NULL REFERENCES rule(module),
    condition     VARCHAR NOT NULL,
    description   VARCHAR NOT NULL,
    impact        INTEGER NOT NULL,
    likelihood    INTEGER NOT NULL,
    publish_date  TIMESTAMP NOT NULL,
    active        BOOLEAN NOT NULL,
    generic       VARCHAR NOT NULL,
    PRIMARY KEY(error_key, rule_module)
)
```

#### Table cluster_rule_user_feedback

```sql
-- user_vote is user's vote,
-- 0 is none,
-- 1 is like,
-- -1 is dislike
CREATE TABLE cluster_rule_user_feedback (
    cluster_id VARCHAR NOT NULL,
    rule_id VARCHAR NOT NULL,
    user_id VARCHAR NOT NULL,
    message VARCHAR NOT NULL,
    user_vote SMALLINT NOT NULL,
    added_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    PRIMARY KEY(cluster_id, rule_id, user_id),
    FOREIGN KEY (cluster_id)
        REFERENCES report(cluster)
        ON DELETE CASCADE,
    FOREIGN KEY (rule_id)
        REFERENCES rule(module)
        ON DELETE CASCADE
)
```

#### Table cluster_rule_toggle

```sql
CREATE TABLE cluster_rule_toggle (
    cluster_id VARCHAR NOT NULL,
    rule_id VARCHAR NOT NULL,
    user_id VARCHAR NOT NULL,
    disabled SMALLINT NOT NULL,
    disabled_at TIMESTAMP NULL,
    enabled_at TIMESTAMP NULL,
    updated_at TIMESTAMP NOT NULL,
    
    CHECK (disabled >= 0 AND disabled <= 1),

    PRIMARY KEY(cluster_id, rule_id, user_id)
)
```

#### Table consumer_error

Errors that happen while processing a message consumed from Kafka are logged into this table. This allows easier debugging of various issues, especially those related to unexpected input data format.

```sql
CREATE TABLE consumer_error (
    topic           VARCHAR NOT NULL,
    partition       INTEGER NOT NULL,
    topic_offset    INTEGER NOT NULL,
    key             VARCHAR,
    produced_at     TIMESTAMP NOT NULL,
    consumed_at     TIMESTAMP NOT NULL,
    message         VARCHAR,
    error           VARCHAR NOT NULL,

    PRIMARY KEY(topic, partition, topic_offset)
)
```

## Documentation for developers

All packages developed in this project have documentation available on [GoDoc server](https://godoc.org/):

* [entry point to the service](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator)
* [package `broker`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/broker)
* [package `consumer`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/consumer)
* [package `content`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/content)
* [package `metrics`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/metrics)
* [package `migration`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/migration)
* [package `producer`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/producer)
* [package `server`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/server)
* [package `storage`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/storage)
* [package `types`](https://godoc.org/github.com/RedHatInsights/insights-results-aggregator/types)

## Configuration

Configuration is done by toml config, default one is `config.toml` in working directory,
but it can be overwritten by `INSIGHTS_RESULTS_AGGREGATOR_CONFIG_FILE` env var.

Also each key in config can be overwritten by corresponding env var. For example if you have config

```toml
[storage]
db_driver = "sqlite3"
sqlite_datasource = "./aggregator.db"
pg_username = "user"
pg_password = "password"
pg_host = "localhost"
pg_port = 5432
pg_db_name = "aggregator"
pg_params = ""
```

and environment variables

```shell
INSIGHTS_RESULTS_AGGREGATOR__STORAGE__DB_DRIVER="postgres"
INSIGHTS_RESULTS_AGGREGATOR__STORAGE__PG_PASSWORD="your secret password"
```

the actual driver will be postgres with password "your secret password"

It's very useful for deploying docker containers and keeping some of your configuration
outside of main config file(like passwords).

## Broker configuration

Broker configuration is in section `[broker]` in config file

```toml
[broker]
address = "localhost:9092"
topic = "topic"
publish_topic = "topic"
group = "aggregator"
enabled = true
save_offset = true
```

* `address` is an address of kafka broker (DEFAULT: "")
* `topic` is a topic to consume messages from (DEFAULT: "")
* `publish_topic` is a topic to publish messages to(see package producer) (DEFAULT: "")
* `group` is a kafka group (DEFAULT: "")
* `enabled` is an option to turn broker on (DEFAULT: false)
* `save_offset` is an option to turn on saving offset of successfully consumed messages.
The offset is stored in the same kafka broker. If it turned off,
consuming will be started from the most recent message (DEFAULT: false)

Option names in env configuration:

* `address` - INSIGHTS_RESULTS_AGGREGATOR__BROKER__ADDRESS
* `topic` - INSIGHTS_RESULTS_AGGREGATOR__BROKER__TOPIC
* `publish_topic` - INSIGHTS_RESULTS_AGGREGATOR__BROKER__PUBLISH_TOPIC
* `group` - INSIGHTS_RESULTS_AGGREGATOR__BROKER__GROUP
* `enabled` - INSIGHTS_RESULTS_AGGREGATOR__BROKER__ENABLED
* `save_offset` - INSIGHTS_RESULTS_AGGREGATOR__BROKER__SAVE_OFFSET

## Server configuration

Server configuration is in section `[server]` in config file.

```toml
[server]
address = ":8080"
api_prefix = "/api/v1/"
api_spec_file = "openapi.json"
debug = true
auth = true
auth_type = "xrh"
use_https = true
enable_cors = true
```

* `address` is host and port which server should listen to
* `api_prefix` is prefix for RestAPI path
* `api_spec_file` is the location of a required OpenAPI specifications file
* `debug` is developer mode that enables some special API endpoints not used on production
* `auth` turns on or turns authentication
* `auth_type` set type of auth, it means which header to use for auth `x-rh-identity` or `Authorization`. Can be used only with `auth = true`. Possible options: `jwt`, `xrh`
* `use_https` is option to turn on TLS server
* `enable_cors` is option to turn on CORS header, that allows to connect from different hosts (**don't use it in production**)

Please note that if `auth` configuration option is turned off, not all REST API endpoints will be usable. Whole REST API schema is satisfied only for `auth = true`.

## Local setup

There is a `docker-compose` configuration that provisions a minimal stack of Insight Platform and
a postgres database.
You can download it here <https://gitlab.cee.redhat.com/insights-qe/iqe-ccx-plugin/blob/master/docker-compose.yml>

### Prerequisites

* minio requires `../minio/data/` and `../minio/config` directories to be created
* edit localhost line in your `/etc/hosts`:  `127.0.0.1       localhost kafka minio`
* `ingress` image should present on your machine. You can build it locally from this repo <https://github.com/RedHatInsights/insights-ingress-go>

### Usage

1. Start the stack `podman-compose up` or `docker-compose up`
2. Wait until kafka will be up.
3. Start `ccx-data-pipeline`: `python3 -m insights_messaging config-devel.yaml`
4. Build `insights-results-aggregator`: `make build`
5. Start `insights-results-aggregator`: `INSIGHTS_RESULTS_AGGREGATOR_CONFIG_FILE=config-devel.toml ./insights-results-aggregator`

Stop Minimal Insights Platform stack `podman-compose down` or `docker-compose down`

In order to upload an insights archive, you can use `curl`:

```shell
curl -k -vvvv -F "upload=@/path/to/your/archive.zip;type=application/vnd.redhat.testareno.archive+zip" http://localhost:3000/api/ingress/v1/upload -H "x-rh-identity: eyJpZGVudGl0eSI6IHsiYWNjb3VudF9udW1iZXIiOiAiMDAwMDAwMSIsICJpbnRlcm5hbCI6IHsib3JnX2lkIjogIjEifX19Cg=="
```

or you can use integration tests suite. More details are [here](https://gitlab.cee.redhat.com/insights-qe/iqe-ccx-plugin).

### Kafka producer

It is possible to use the script `produce_insights_results` from `utils` to produce several Insights results into Kafka topic. Its dependency is Kafkacat that needs to be installed on the same machine. You can find installation instructions [on this page](https://github.com/edenhill/kafkacat).

### Rules content

The generated cluster reports from Insights results contain three lists of rules that were either __skipped__ (because of missing requirements, etc.), __passed__ (the rule got executed but no issue was found), or __hit__ (the rule got executed and found the issue it was looking for) by the cluster, where each rule is represented as a dictionary containing identifying information about the rule itself.

The __hit__ rules are the rules that the customer is interested in and therefore the information about them (their __content__) needs to be displayed in OCM. The content for the rules is present in another repository, alongside the actual rule implementations, which is primarily maintained by the support-facing team.
For that reason, the `insights-results-aggregator` is setup in a way, that the content we're interested in is copied to the Docker image from the repository during image build and we have a push webhook on master branch set up in that repository, signalling our app to be rebuilt.

This content is then processed upon application start-up, correctly parsed by the package `content` and saved into the database (see DB structure).

When a request for a cluster report comes from OCM, the report is parsed (TODO: parse reports only once when consuming them) and content for all the hit rules is returned.

#### Local environment with rules content

The rules content parser is configured by default to expect the content in a root directory `/rules-content`.
This can be changed either by an environment variable `INSIGHTS_RESULTS_AGGREGATOR__CONTENT__PATH` or by modifying the config file entry:

```toml
[content]
path = "/rules-content"
```

To get the latest rules content locally, you can `make rules_content`, which just runs the script `update_rules_content.sh` mimicking the Dockerfile behavior (NOTE: you need to be in RH VPN to be able to access that repository, but it is not private). The script copies the content into a .gitignored folder `rules-content`, so all that's necessary is to change the expected path.

## Database

Aggregator is configured to use SQLite3 DB by default, but it also supports PostgreSQL.
In CI and QA environments, the configuration is overridden by environment variables to use PostgreSQL.

To establish connection to the PostgreSQL instance provided by the minimal stack in `docker-compose.yml` for local setup, the following configuration options need to be changed in `storage` section of `config.toml`:

```toml
[storage]
db_driver = "postgres"
pg_username = "user"
pg_password = "password"
pg_host = "localhost"
pg_port = 55432
pg_db_name = "aggregator"
pg_params = "sslmode=disable"
```

### Migration mechanism

This service contains an implementation of a simple database migration mechanism that allows semi-automatic transitions between various database versions as well as building the latest version of the database from scratch.

Before using the migration mechanism, it is first necessary to initialize the migration information table `migration_info`. This can be done using the `migration.InitInfoTable(*sql.DB)` function. Any attempt to get or set the database version without initializing this table first will result in a `no such table: migration_info` error from the SQL driver.

New migrations must be added manually into the code, because it was decided that modifying the list of migrations at runtime is undesirable.

To migrate the database to a certain version, in either direction (both upgrade and downgrade), use the `migration.SetDBVersion(*sql.DB, migration.Version)` function.

**To upgrade the database to the highest available version, use `migration.SetDBVersion(db, migration.GetMaxVersion())`.** This will automatically perform all the necessary steps to migrate the database from its current version to the highest defined version.

See `/migration/migration.go` documentation for an overview of all available DB migration functionality.

## REST API schema based on OpenAPI 3.0

Aggregator service provides information about its REST API schema via endpoint `api/v1/openapi.json`. OpenAPI 3.0 is used to describe the schema; it can be read by human and consumed by computers.

For example, if aggregator is started locally, it is possible to read schema based on OpenAPI 3.0 specification by using the following command:

```shell
curl localhost:8080/api/v1/openapi.json
```

Please note that OpenAPI schema is accessible w/o the need to provide authorization tokens.

## Prometheus API

It is possible to use `/api/v1/metrics` REST API endpoint to read all metrics exposed to Prometheus or to any tool that is compatible with it.
Currently, the following metrics are exposed:

1. `api_endpoints_requests` the total number of requests per endpoint
1. `api_endpoints_response_time` API endpoints response time
1. `consumed_messages` the total number of messages consumed from Kafka
1. `feedback_on_rules` the total number of left feedback
1. `produced_messages` the total number of produced messages
1. `written_reports` the total number of reports written to the storage

Additionally it is possible to consume all metrics provided by Go runtime. There metrics start with `go_` and `process_` prefixes.

## pprof interface in debug mode

In debug mode, standard Golang pprof interface is available at `/debug/pprof/`

Common usage (for using pprof against local instance):

```
go tool pprof localhost:8080/debug/pprof/profile
```

A practical example is available here:
https://medium.com/@paulborile/profiling-a-golang-rest-api-server-635fa0ed45f3



## Authentication

Authentication is working through `x-rh-identity` token which is provided by 3scale. `x-rh-identity` is base64 encoded JSON, that includes data about user and organization, like:

```JSON
{
  "identity": {
    "account_number": "0369233",
    "type": "User",
    "user": {
      "username": "jdoe",
      "email": "jdoe@acme.com",
      "first_name": "John",
      "last_name": "Doe",
      "is_active": true,
      "is_org_admin": false,
      "is_internal": false,
      "locale": "en_US"
    },
    "internal": {
      "org_id": "3340851",
      "auth_type": "basic-auth",
      "auth_time": 6300
    }
  }
}
```

If aggregator didn't get identity token or got invalid one, then it returns error with status code `403` - Forbidden.

## Contribution

Please look into document [CONTRIBUTING.md](CONTRIBUTING.md) that contains all information about how to contribute to this project.

Please look also at [Definitiot of Done](DoD.md) document with further informations.

## Testing

tl;dr: `make before_commit` will run most of the checks by magic

The following tests can be run to test your code in `insights-results-aggregator`.
Detailed information about each type of test is included in the corresponding subsection:

1. Unit tests: checks behaviour of all units in source code (methods, functions)
1. REST API Tests: test the real REST API of locally deployed application with database initialized with test data only
1. Integration tests: the integration tests for `insights-results-aggregator` service
1. Metrics tests: test whether Prometheus metrics are exposed as expected

### Unit tests

Set of unit tests checks all units of source code. Additionally the code coverage is computed and displayed.
Code coverage is stored in a file `coverage.out` and can be checked by a script named `check_coverage.sh`.

To run unit tests use the following command:

`make test`

If you have postgres running on port from `./config-devel.toml` file it will also run tests against it

### All integration tests

`make integration_tests`

#### Only REST API tests

Set of tests to check REST API of locally deployed application with database initialized with test data only.

To run REST API tests use the following command:

`make rest_api_tests`

By default all logs from the application aren't shown, if you want to see them, run:

`./test.sh rest_api --verbose`

## CI

[Travis CI](https://travis-ci.com/) is configured for this repository. Several tests and checks are started for all pull requests:

* Unit tests that use the standard tool `go test`.
* `go fmt` tool to check code formatting. That tool is run with `-s` flag to perform [following transformations](https://golang.org/cmd/gofmt/#hdr-The_simplify_command)
* `go vet` to report likely mistakes in source code, for example suspicious constructs, such as Printf calls whose arguments do not align with the format string.
* `golint` as a linter for all Go sources stored in this repository
* `gocyclo` to report all functions and methods with too high cyclomatic complexity. The cyclomatic complexity of a function is calculated according to the following rules: 1 is the base complexity of a function +1 for each 'if', 'for', 'case', '&&' or '||' Go Report Card warns on functions with cyclomatic complexity > 9
* `goconst` to find repeated strings that could be replaced by a constant
* `gosec` to inspect source code for security problems by scanning the Go AST
* `ineffassign` to detect and print all ineffectual assignments in Go code
* `errcheck` for checking for all unchecked errors in go programs
* `shellcheck` to perform static analysis for all shell scripts used in this repository
* `abcgo` to measure ABC metrics for Go source code and check if the metrics does not exceed specified threshold

Please note that all checks mentioned above have to pass for the change to be merged into master branch.

History of checks performed by CI is available at [RedHatInsights / insights-results-aggregator](https://travis-ci.org/RedHatInsights/insights-results-aggregator).

## Rules

The user has the ability to disable a rule/health check recommendation that they're not interested in to stop it from showing in OCM. The user also has the ability to re-enable the rule, in case they later become interested in it, or in the case of an accidental disable, for exapmle.

This is made possible by using these two endpoints:
`clusters/{cluster}/rules/{rule_id}/disable`
`clusters/{cluster}/rules/{rule_id}/enable`

### Tutorial rule

Directory `rules/tutorial/` contains tutorial rule that is 'hit' by any cluster.

## Mock data for aggregator

Data to be consumed by aggregator through Kafka broker is prepared in `utils/produce_insights_results/` subdirectory.
Several types of data are available there:

* `r_[0-9]*.json` - real data analyzed from test clusters
* `r_tutorial_[0-9]*.json` - real data analyzed from test clusters with added tutorial rule result
* `result*.json` - artifically created data
* `big_resuts.json` - file with most reports created by joining several real data (no cluster is in the state when all rules fail)
* `big_results_tutorial.json` - the same, but with tutorial rule result
* `big_results_no_skips.json` - the same, but no skipped rules are stored
* `big_results_no_skips_tutorial.json` - the same, but with tutorial rule result
* `no_hits.json` - data with no rule hits (ie. the cluster is healthy)
* `no_hits_no_skips.json` - data with no rule hits and no skips (ie. there's no health check performed)
* `tutorial_only.json` - report with only tutorial rule hit

## Utilitites

Utilities are stored in `utils` subdirectory.

### `anonymize.py`

Anonymize input data produced by OCP rules engine.

All input files that ends with '.json' are read by this script and
if they contain 'info' key, the value stored under this key is
replaced by empty list, because these informations might contain
sensitive data. Output file names are in format 's_number.json', ie.
the original file name is not preserved as it also might contain
sensitive data.

### `2report.py`

Converts outputs from OCP rule engine into proper reports.

All input files that with filename 's_\*.json' (usually anonymized
outputs from OCP rule engine' are converted into proper 'report'
that can be:

1. Published into Kafka topic
1. Stored directly into aggregator database

It is done by inserting organization ID, clusterName and lastChecked
attributes and by rearanging output structure. Output files will
have following names: 'r_\*.json'.

### `fill_in_results.sh`

This script can be used to fill in the aggregator database in the selected pipeline with data taken from test clusters.
It performs several operations:

1. Decompress input data generated by Insights operator and stored in Ceph/AWS bucket, update directory structure accordingly
1. Run Insights OCP rules against all input data
1. Anonymize OCP rules results
1. Convert OCP rules results into a form compatible with aggregator. These results (JSONs) can be published into Kafka using `produce.sh` (several times if needed)

#### Usage

```shell
./fill_in_results.sh archive.tar.bz org_id cluster_name
```

#### A real example

```shell
./fill_in_results.sh external-rules-archives-2020-03-31.tar 11789772 5d5892d3-1f74-4ccf-91af-548dfc9767aa
```

### `stat.py`

This script can be used to display statistic about rules that really 'hit' problems on clusters. Can be used against test data or production data if needed.

### `gen_broken_messages.py`

This script read input message (that should be correct) and generates bunch of new messages. Each generated message is broken in some way so it is possible to use such messages to test how broken messages are handled on aggregator (ie. consumer) side.

Types of input message mutation:
* any item (identified by its key) can be removed
* new items with random key and content can be added
* any item can be replaced by new random content

### `affected_clusters.py`

This script can be used to analyze data exported from `report` table by
the following command typed into PSQL console:

    \copy report to 'reports.csv csv

Script displays two tables:
    1. org id + cluster name (list of affected clusters)
    2. org id + number of affected clusters (usually the only information reguired by management)

### `json_check.py`

Simple checker if all JSONs have the correct syntax (not scheme).

Usage:

```
usage: json_check.py [-h] [-v]

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  make it verbose
```
