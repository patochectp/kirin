# Kirin

![Continuous Integration Status](https://img.shields.io/github/workflow/status/CanalTP/kirin/Continuous%20Integration)
![Code Coverage](https://img.shields.io/codecov/c/gh/CanalTP/kirin)

```py
                                                        /
                                                      .7
                                           \       , //
                                           |\.--._/|//
                                          /\ ) ) ).'/
                                         /(  \  // /
                                        /(   J`((_/ \
                                       / ) | _\     /
                                      /|)  \  eJ    L
                                     |  \ L \   L   L
                                    /  \  J  `. J   L
                                    |  )   L   \/   \
                                   /  \    J   (\   /
                 _....___         |  \      \   \```
          ,.._.-'        '''--...-||\     -. \   \
        .'.=.'                    `         `.\ [ Y
       /   /                                  \]  J
      Y / Y                                    Y   L
      | | |          \                         |   L
      | | |           Y                        A  J
      |   I           |                       /I\ /
      |    \          I             \        ( |]/|
      J     \         /._           /        -tI/ |
       L     )       /   /'-------'J           `'-:.
       J   .'      ,'  ,' ,     \   `'-.__          \
        \ T      ,'  ,'   )\    /|        ';'---7   /
         \|    ,'L  Y...-' / _.' /         \   /   /
          J   Y  |  J    .'-'   /         ,--.(   /
           L  |  J   L -'     .'         /  |    /\
           |  J.  L  J     .-;.-/       |    \ .' /
           J   L`-J   L____,.-'`        |  _.-'   |
            L  J   L  J                  ``  J    |
            J   L  |   L                     J    |
             L  J  L    \                    L    \
             |   L  ) _.'\                    ) _.'\
             L    \('`    \                  ('`    \
              ) _.'\`-....'                   `-....'
             ('`    \
              `-.___/
```

Kirin deals with real-time updates for navitia.
When feeds are provided to Kirin by a client, it requests navitia to find the corresponding vehicle journey and apply the update, that is then posted in a queue for navitia to pick.

The feeds can be of the following type:

- COTS : Also a proprietary realtime information feed for SNCF. JSON files are posted to the Kirin web service
  (example of such feed
  [here](https://github.com/CanalTP/kirin/blob/master/tests/fixtures/cots_train_96231_delayed.json)).
  A cause message subservice is also requested during the processing of this feed.
- GTFS-RT : A realtime information format that comes with the GTFS format (base-schedule informations).
  Documentation is available [here](https://developers.google.com/transit/gtfs-realtime/?hl=en).
  Typically, a transport authority will provide a server where GTFS-RT protobuf files can be consumed and
  regularly polled.

## Setup

This describes a "classic" setup.

A "dockerized" local setup is also demonstrated in fab_kirin, see
[instructions](https://github.com/CanalTP/fab_kirin/demo/README.md).

Kirin supports Python 2.7.

- Install dependencies with

    ```sh
    pip install -r requirements_dev.txt
    ```

    (virtualenv is strongly advised)

- You also need a redis-server to use cache on some requests and a rabbitmq-server to post updated data in the queue.
    It can be installed with :

    ```sh
    sudo apt-get install redis-server rabbitmq-server protobuf-compiler
    ```

- Setup the Kirin database (postgresql >= 9.1 is required):

    ```sh
    sudo -i -u postgres
    # Create a user
    createuser -P navitia (password "navitia")

    # Create database
    createdb -O navitia kirin

    # Create database for tests
    createdb -O navitia chaos_testing
    ctrl + d
    ```

- Create a configuration file by copying and editing ```kirin/default_settings.py```

    ```sh
    NAVITIA_URL = '<url of the navitia server>' # ex: 'http://localhost:5000/'
    DEBUG = True
    log_formatter = 'json'
    ```

- Create a file ```.env``` with the path to you configuration file:

    ```sh
    KIRIN_CONFIG_FILE=default_settings.py
    KIRIN_LOG_FORMATTER='json'  # If you wish to have logs formated as json (more details)
    ```

- Build the protocol buffer files

    ```sh
    git submodule init
    git submodule update
    ./setup.py build_pbf
    ```

- Build the version file:

    ```sh
    ./setup.py build_version
    ```

- Update the database schema (requires honcho):

    ```sh
    pip install honcho
    honcho run ./manage.py db upgrade
    ```

- Run the development server:

    ```sh
    honcho start
    ```

    This command runs several processes :

    - a server to listen to incoming requests
    - a scheduler and its worker to perform tasks scheduled in KIRIN_CONFIG_FILE
      Note: one of the tasks scheduled is a poller to retrieve GTFS-RT files, only useful when there's a feed provider URL defined.
      If not needed, this specific task can be disabled in KIRIN_CONFIG_FILE by removing the 'poller' task in the 'CELERYBEAT_SCHEDULE' section. This will avoid having logs and errors about GTFS-RT.
    - a job to read the info already available in Kirin database. Note that this step of data reloading at boot is mandatory for Kirin to be able to process future real-time feeds.

- Enjoy: you can now request the Kirin API

### Docker

A docker image of Kirin can be built using the Dockerfile:
`docker build -t kirin .`
When running this image, the Kirin web server is launched and an optional *port* can be given to expose the API.
`docker run -p <port>:9090 kirin`

Note: a Kirin database is needed on localhost for the requests to be done successfully.

## API

Kirin API provides several endpoints (that can be requested through port 5000 by default, or
port 54746 if using honcho).\
To list all available endpoints:

```sh
curl 'http://localhost:5000/'
```

### Contributors

#### GET

Returns a list of contributors present in the database

```sh
curl -X GET 'http://localhost:5000/contributors'
```

```json
{
  "contributors": [
    {
      "navitia_coverage": "sncf",
      "is_active": true,
      "navitia_token": "9489dd5f-46b4hhhhhhhhhhhhhhhh",
      "connector_type": "cots",
      "id": "realtime.cots",
      "feed_url": "*not_used_for_cots*",
      "retrieval_interval": "*not_used_for_cots*",
      "broker_url": "*not_used_for_cots*",
      "exchange_name": "*not_used_for_cots*",
      "queue_name": "*not_used_for_cots*"
    },
    {
      "navitia_coverage": "ca-qc-sherbrooke",
      "is_active": true,
      "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a",
      "connector_type": "gtfs-rt",
      "id": "realtime.sherbrooke",
      "feed_url": "http://0.0.0.0./civilia/TTT/pb/tripUpdates.pb",
      "retrieval_interval": 10,
      "broker_url": "*not_used_for_gtfsrt*",
      "exchange_name": "*not_used_for_gtfsrt*",
      "queue_name": "*not_used_for_gtfsrt*"
    },
    {
      "navitia_coverage": "sncf",
      "is_active": true,
      "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a",
      "connector_type": "piv",
      "id": "realtime.piv",
      "feed_url": "*not_used_for_piv*",
      "retrieval_interval": "*not_used_for_piv*",
      "broker_url": "pyamqp://guest:guest@localhost:5672//?heartbeat=60",
      "exchange_name": "piv",
      "queue_name": "piv-kirin"
    }
  ]
}
```

Returns a specific contributor present in the database

```sh
curl -X GET 'http://localhost:5000/contributors/realtime.sherbrooke'
```

```json
{
  "contributors": [
    {
      "navitia_coverage": "ca-qc-sherbrooke",
      "is_active": true,
      "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a",
      "connector_type": "gtfs-rt",
      "id": "realtime.sherbrooke",
      "feed_url": "http://0.0.0.0./civilia/TTT/pb/tripUpdates.pb",
      "retrieval_interval": 10,
      "broker_url": "*not_used_for_gtfsrt*",
      "exchange_name": "*not_used_for_gtfsrt*",
      "queue_name": "*not_used_for_gtfsrt*"
    }
  ]
}
```

#### POST

To create a new contributor, parameters need to be given in json sent in the body as described [here](documentation/internal_format.md).\
Please be careful with format of booleans and numbers :wink:

```sh
curl -X POST 'http://localhost:5000/contributors/' -d'{"feed_url": "http://0.0.0.0./civilia/TTT/pb/tripUpdates.pb", "retrieval_interval": 10, "navitia_coverage": "ca-qc-sherbrooke", "is_active": true, "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a", "connector_type": "gtfs-rt", "id": "realtime.sherbrooke"}' -H'content-type: application/json'
```

``` json
{
  "contributor": {
    "feed_url": "http://0.0.0.0./civilia/TTT/pb/tripUpdates.pb",
    "retrieval_interval": 10,
    "navitia_coverage": "ca-qc-sherbrooke",
    "is_active": true,
    "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a",
    "connector_type": "gtfs-rt",
    "id": "realtime.sherbrooke"
  }
}
```

#### PUT

Modifies a contributor and has same parameters as POST.\
Please be careful with format of booleans and numbers :wink:

When a parameter is missing, it is not changed.

When PUTing the exact result of a GET nothing is changed.

```sh
curl -X PUT 'http://localhost:5000/contributors/' -d'{"feed_url": "http://0.0.0.0./civilia/TTT/pb/tripUpdates.pb", "retrieval_interval": 10, "navitia_coverage": "ca-qc-sherbrooke", "is_active": true, "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a", "connector_type": "gtfs-rt", "id": "realtime.sherbrooke"}' -H'content-type: application/json'
```

```json
{
  "contributor": {
    "feed_url": "http://0.0.0.0./civilia/TTT/pb/tripUpdates.pb",
    "retrieval_interval": 10,
    "navitia_coverage": "ca-qc-sherbrooke",
    "is_active": true,
    "navitia_token": "9489dd5f-46b4-mmmmmmmmmm3bfba0c71e8a",
    "connector_type": "gtfs-rt",
    "id": "realtime.sherbrooke"
  }
}
```

#### DELETE

Deletes a specific contributor

```sh
curl -X DELETE 'http://localhost:5000/contributors/realtime.toto'
```

### Health (GET)

Returns info about the health of Kirin **webservice** (whether it can be used or not).

```sh
curl 'http://localhost:5000/health'
```

Returns message "OK" (HTTP status `200`) if Kirin webservice is healthy, "KO" (HTTP status `503`) otherwise.
This endpoint checks that the webservice runs and responds (obviously).

It also checks:

- the connection to postgresql database (ability to process RT feeds and store initial feed and the result)
- the connection to Navitia (ability to process RT feeds)

This does not check:

- the connection to redis (ability to cache and use circuit-breaker)
- the connection to rabbitmq (ability to send immediately result to Navitia for integration)

For details on errors or statuses... check `/status` :-)

### Status (GET)

Returns info about the Kirin and the previous jobs performed

```sh
curl 'http://localhost:5000/status'
```

In the response received:

- last_update: last time Kirin received a file (or pulled it, depending the client) in order to update navitia data.
- last_valid_update: last time Kirin received a file that was valid and managed to update navitia data properly.
- last_update_error: information about error from the last time Kirin processed a file and a problem occurred. It can either be a problem about the file or the data update. The field will be empty if last_update = last_valid_update.
- navitia_url: root url of the navitia server used to consolidate real-time information received by Kirin.
- db_connection: state of the connection to postgresql database (condition for /health to be "OK")
- navitia_connection: state of the connection to navitia (condition for /health to be "OK")
Other info are available about Kirin ("version"), the database ("db_version", "db_pool_status") and the rabbitmq ("rabbitmq_info").

### SNCF's realtime feeds

For the SNCF's realtime feeds to be taken into account by navitia, some parameters need to be set
for both Kirin and Kraken (the navitia core calculator).

- In Kirin:

    - Add a contributor 'cots' using the end point `/contributors`

- In Kraken:

    - kraken.ini:

    ```ini
    [GENERAL] # The following parameters need to be added to the already existing ones in the GENERAL section
    is_realtime_enabled = true
    kirin_timeout = 180000 # in ms (optional)

    [BROKER] # It represents the rabbitmq-server, fill the following parameters according to your settings
    host = localhost
    port = 5672
    username = guest
    password = guest
    exchange = navitia
    ```

#### Cots (POST)

Post a COTS update file with modifications about a vehicle journey (delay, disruption, deletion, ...)
that will be modified and posted in the rabbitmq queue.

```sh
curl -X POST 'http://localhost:5000/cots' -H 'Content-Type: application/json' -d @<PATH/TO/my_cots.json>
```

For the COTS to be taken into account by navitia, please add the common SNCF's parameters above, plus:

- In Kirin:

    - KIRIN_CONFIG_FILE:

    ```py
    # Parameters for COTS cause message subservice (ParIV)
    COTS_PAR_IV_API_KEY = '<COTS ParIV API key>'
    COTS_PAR_IV_MOTIF_RESOURCE_SERVER = '<URL of COTS ParIV-Motif cause endpoint>'
    COTS_PAR_IV_TOKEN_SERVER = '<URL of COTS ParIV oauth2 token endpoint>'
    COTS_PAR_IV_CLIENT_ID = '<COTS ParIV username>'
    COTS_PAR_IV_CLIENT_SECRET = '<COTS ParIV password>'
    ```

- In Kraken:

    - kraken.ini:

    ```ini
    [BROKER] # in the BROKER section existing from common part
    rt_topics = realtime.cots  # it's possible to add multiple topics simultaneously
    ```

If the COTS was successfully sent and processed by Kirin, the http response 200 will have a message "OK".

## Maintenance

### Definitely remove a contributor from configuration

A command to clean inactive contributors is available, to be triggered manually:

```bash
python ./manage.py purge_contributor <contributor_id>
```

If Kirin is deployed inside docker containers on a platform, you may locate the docker-compose file defining a `kirin`
webservice that was used to launch containers.\
Then for example, if the file is `docker-compose_kirin.yml`, you can launch:

```bash
docker-compose -f docker-compose_kirin.yml run --rm --no-deps kirin ./manage.py purge_contributor <contributor_id>
```

This will check that for the given contributor, `is_active=false` and that
no more object (TripUpdate or RealTimeUpdate) is linked to that contributor in Kirin database.

Those objects are progressively purged by automatic jobs configured in the settings file.

## Development

If you want to develop in Kirin, run tests or read more about technical details please refer to
[CONTRIBUTING.md](CONTRIBUTING.md)
