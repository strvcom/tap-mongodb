# tap-mongodb

This is a [Singer](https://singer.io) tap that produces JSON-formatted data following the [Singer spec](https://github.com/singer-io/getting-started/blob/master/SPEC.md) from a MongoDB source. 
This plugin extends the original [implementation](https://github.com/singer-io/tap-mongodb) by introducing support for **srv** mode.

## Set up Virtual Environment
```
python3 -m venv ~/.virtualenvs/tap-mongodb
source ~/.virtualenvs/tap-mongodb/bin/activate
```

## Install tap
```
pip install -U pip setuptools
pip install tap-mongodb
```

## Set up Config file
Create json file called `config.json`, with the following contents:
```
{
  "password": "<password>",
  "user": "<username>",
  "host": "<host ip address>"
}
```
The folowing parameters are optional for your config file:

| Name | Type | Description |
| -----|------|------------ |
| `port` | integer | port number (default is 27017) |
| `database` | string | name of the database |
|`ssl` | Boolean | can be set to true to connect using ssl |
|`srv` | Boolean | use DNS Seed List connection string - [link](https://docs.mongodb.com/manual/reference/connection-string/) |
| `include_schema_in_destination_stream_name` | Boolean | forces the stream names to take the form `<database_name>_<collection_name>` instead of `<collection_name>`|

All of the above attributes are required by the tap to connect to your mongo instance. 

## Run in discovery mode
Run the following command and redirect the output into the catalog file
```
tap-mongodb --config ~/config.json --discover > ~/catalog.json
```

Your catalog file should now look like this:
```
{
  "streams": [
    {
      "table_name": "<table name>",
      "tap_stream_id": "<tap_stream_id>",
      "metadata": [
        {
          "breadcrumb": [],
          "metadata": {
            "row-count":<int>,
            "is-view": <bool>,
            "database-name": "<database name>",
            "table-key-properties": [
              "_id"
            ],
            "valid-replication-keys": [
              "_id"
            ]
          }
        }
      ],
      "stream": "<stream name>",
      "schema": {
        "type": "object"
      }
    }
  ]
}
```

## Edit Catalog file
### Using valid json, edit the config.json file
To select a stream, enter the following to the stream's metadata:
```
"selected": true,
"replication-method": <replication method>,
```

`<replication-method>` must be either `FULL_TABLE` or `LOG_BASED`

To add a projection to a stream, add the following to the stream's metadata field:
```
"tap-mongodb.projection": <projection>
```

For example, if you were to edit the example stream to select the stream as well as add a projection, config.json should look this:
```
{
  "streams": [
    {
      "table_name": "<table name>",
      "tap_stream_id": "<tap_stream_id>",
      "metadata": [
        {
          "breadcrumb": [],
          "metadata": {
            "row-count": <int>,
            "is-view": <bool>,
            "database-name": "<database name>",
            "table-key-properties": [
              "_id"
            ],
            "valid-replication-keys": [
              "_id"
            ],
            "selected": true,
            "replication-method": "<replication method>",
            "tap-mongodb.projection": "<projection>"
          }
        }
      ],
      "stream": "<stream name>",
      "schema": {
        "type": "object"
      }
    }
  ]
}

```
## Run in sync mode:
`tap-mongodb --config ~/config.json --catalog ~/catalog.json`

The tap will write bookmarks to stdout which can be captured and passed as an optional `--state state.json` parameter to the tap for the next sync.

## Supplemental MongoDB Info

### Local MongoDB Setup
If you haven't yet set up a local mongodb client, follow [these instructions](https://github.com/singer-io/tap-mongodb/blob/master/spikes/local_mongo_setup.md)

## Use as a Meltano plugin
To use this tap in Meltano, the configuration `meltano.yml` file needs to be extended with a definition of a **custom plugin** as follows:

```
plugins:
  extractors:
  - name: tap-mongodb
    label: MongoDB
    description: General purpose, document-based, distributed database
    namespace: tap_mongodb
    variants:
      - name: strv
        repo: https://github.com/strvcom/tap-mongodb
        pip_url: git+https://github.com/strvcom/tap-mongodb.git
        executable: tap-mongodb
        capabilities:
          - catalog
          - discover
          - state
        settings_group_validation:
          - [ 'host', 'user', 'password']
        settings:
          - name: host
            label: Host URL
            value: localhost
          - name: port
            kind: integer
            value: 27017
          - name: user
          - name: password
            kind: password
          - name: database
            label: Database Name
          - name: replica_set
          - name: ssl
            kind: boolean
            value: false
            value_post_processor: stringify
            label: SSL
          - name: verify_mode
            kind: boolean
            value: true
            value_post_processor: stringify
            description: SSL Verify Mode
          - name: srv
            kind: boolean
            value: true
            description: Use DNS Seed List connection string
          - name: include_schemas_in_destination_stream_name
            kind: boolean
            value: false
            description: Forces the stream names to take the form `<database_name>_<collection_name>` instead of `<collection_name>`
```

To configure the plugin, there is an [official guide](https://hub.meltano.com/extractors/mongodb.html) to follow. 
The official guide & plugin does not support **srv** mode, to configure this plugin to run in this mode, extend the configuration file with the following:

```
plugins:
  extractors:
  - name: tap-mongodb
    ...
    config:
      host: <host ip address>
      user: <username>
      srv: true
```

The passwords should be set as described in the original guide.

**Note**: An extractor configuration needs to be defined for this plugin to work. The configuration is described in the [official guide](https://meltano.com/docs/getting-started.html#add-an-extractor-to-pull-data-from-a-source).
A simple example of additional configuration is the following extension of configuration file:

```
plugins:
  extractors:
  - name: tap-mongodb
    ...
    config:
      ...
    metadata:
      '*':
        replication-method: FULL_TABLE
```

---

Copyright &copy; 2019 Stitch
