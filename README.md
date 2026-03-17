# Mosquitto MQTT Example on Clever Cloud
[![Clever Cloud - PaaS](https://img.shields.io/badge/Clever%20Cloud-PaaS-orange)](https://clever-cloud.com)

This example shows how to deploy [Eclipse Mosquitto](https://mosquitto.org/) on Clever Cloud using the [Linux native runtime](https://www.clever.cloud/developers/doc/applications/linux/) and [Mise](https://mise.jdx.dev/) task runner.

## About the Application

Mosquitto is a lightweight, open-source MQTT message broker that implements the MQTT protocol. This example builds Mosquitto from source on Clever Cloud using the Linux native runtime, with password-based authentication and persistent message storage via an FS Bucket.

The `mise.toml` file defines two tasks:

- **build**: Downloads and compiles libmicrohttpd, cJSON, and Mosquitto from source, then sets up password authentication
- **run**: Starts the Mosquitto broker (which serves both MQTT and the HTTP API)

## Prerequisites

- A [Clever Cloud account](https://console.clever-cloud.com/)
- [Clever Tools CLI](https://www.clever-cloud.com/developers/doc/clever-tools/getting_started/) installed

## Deploying on Clever Cloud

You have two options to deploy your application on Clever Cloud: using the Web Console or using the Clever Tools CLI.

### Option 1: Deploy using the Web Console

#### 1. Create an account on Clever Cloud

If you don't already have an account, go to the [Clever Cloud console](https://console.clever-cloud.com/) and follow the registration instructions.

#### 2. Set up your application on Clever Cloud

1. Log in to the [Clever Cloud console](https://console.clever-cloud.com/)
2. Click on "Create" and select "An application"
3. Choose **Linux** as the runtime environment
4. Configure your application settings (name, region, etc.). We recommend at least a **Medium** build instance since Mosquitto is compiled from source along with its dependencies

#### 3. Create an FS Bucket add-on

Mosquitto needs persistent storage for message data. Create an FS Bucket add-on and link it to your application. Then configure the mount point (see environment variables below).

#### 4. Configure Environment Variables

In your application's settings, add the following environment variables:

| Variable | Required | Description |
|---|---|---|
| `MOSQUITTO_VERSION` | Yes | Mosquitto version to build (e.g., `2.1.2`). Find versions on the [Mosquitto download page](https://mosquitto.org/download/) |
| `CJSON_VERSION` | Yes | cJSON version (e.g., `1.7.19`). Required build dependency |
| `LIBMICROHTTPD_VERSION` | Yes | libmicrohttpd version (e.g., `1.0.2`). Required for the HTTP API. Find versions on the [GNU FTP](https://ftp.gnu.org/gnu/libmicrohttpd/) |
| `MQTT_USER` | Yes | Username for MQTT authentication |
| `MQTT_PASSWORD` | Yes | Password for MQTT authentication |
| `CC_FS_BUCKET` | Yes | FS Bucket mount point in the format `/storage:<bucket-host>` |

#### 5. Set up TCP Redirection

Since MQTT is a TCP protocol (not HTTP), you need a TCP redirection to expose the broker port. Configure this in the **Domain names** section of your application's settings in the Web Console.

#### 6. Deploy Your Application

You can deploy your application using Git:

```bash
# Add Clever Cloud as a remote repository
git remote add clever git+ssh://git@push-par-clevercloud-customers.services.clever-cloud.com/app_<your-app-id>.git

# Push your code to deploy
git push clever master
```

### Option 2: Deploy using Clever Tools CLI

#### 1. Install Clever Tools

Install the Clever Tools CLI following the [official documentation](https://www.clever-cloud.com/developers/doc/clever-tools/getting_started/):

```bash
# Using npm
npm install -g clever-tools

# Or using Homebrew (macOS)
brew install clever-tools
```

#### 2. Log in to your Clever Cloud account

```bash
clever login
```

#### 3. Create a new application

```bash
# Initialize the current directory as a Clever Cloud Linux application
clever create --type linux <YOUR_APP_NAME>
```

#### 4. Use a larger instance for builds

Since Mosquitto is compiled from source along with its dependencies, we recommend at least a Medium build instance to speed up compilation:

```bash
clever scale --build-flavor M
```

#### 5. Create an FS Bucket for persistent storage

```bash
clever addon create fs-bucket <YOUR_FS_BUCKET_NAME> --link <YOUR_APP_NAME>
clever env set CC_FS_BUCKET "/storage:$(clever env | awk -F = '/BUCKET_HOST/ { gsub(/"/, "", $2); print $2}')"
```

#### 6. Configure environment variables

```bash
# Required: set Mosquitto and cJSON versions
clever env set MOSQUITTO_VERSION "2.1.2"
clever env set CJSON_VERSION "1.7.19"
clever env set LIBMICROHTTPD_VERSION "1.0.2"

# Required: set MQTT credentials
clever env set MQTT_USER "user_name"
clever env set MQTT_PASSWORD "a_good_password"
```

#### 7. Set up TCP redirection

Since MQTT is a TCP protocol (not HTTP), you need a TCP redirection to expose the broker port. An external port will be attributed to your application — use it to connect to your MQTT broker.

If you use a `cleverapps.io` domain (for testing purposes only):

```bash
clever domain  # to show the domain name of your app
clever tcp-redirs add --namespace cleverapps
```

If you've set up a custom domain:

```bash
clever domain add your_domain.com
clever tcp-redirs add --namespace default
```

#### 8. Deploy your application

```bash
clever deploy
```

#### 9. Open your application in a browser

```bash
clever open
```

## Architecture

Mosquitto runs a single process with two listeners:

- **MQTT listener** on port `4040` — handles MQTT client connections, exposed externally via TCP redirection
- **HTTP API listener** on port `8080` — serves the Mosquitto control API and static files from `www/`, used by Clever Cloud for health checks

Clever Cloud's reverse proxy routes HTTP traffic to the HTTP API on port `8080`, while MQTT clients connect through the TCP redirection (an external port assigned by Clever Cloud mapped to the internal port `4040`).

## Mosquitto Configuration

The broker is configured via `config/mosquitto.conf`:

| Setting | Value | Description |
|---|---|---|
| `password_file` | `passwdfile` | Password file generated during build |
| `listener` | `4040 0.0.0.0` | MQTT listener port and bind address |
| `listener` | `8080 0.0.0.0` | HTTP API listener for health checks and control API |
| `protocol` | `http_api` | Protocol for the 8080 listener |
| `http_dir` | `www` | Directory for serving static HTTP files |
| `enable_control_api` | `true` | Enable the `$CONTROL/broker/v1` API |
| `persistence` | `true` | Enable message persistence |
| `persistence_location` | `storage` | Persistence directory (mapped to FS Bucket) |
| `autosave_interval` | `60` | Save persistent data every 60 seconds |

You can customize Mosquitto further by editing `config/mosquitto.conf`. See the [Mosquitto documentation](https://mosquitto.org/man/mosquitto-conf-5.html) for all available options.

## Connecting to the Broker

First, retrieve the external port assigned by the TCP redirection:

```bash
export TCP_PORT=$(clever tcp-redirs | awk '/cleverapps/ {print $1}')
```

Then connect to your Mosquitto broker using any MQTT client, using the external port from the TCP redirection:

```bash
# Subscribe to a topic
mosquitto_sub -h <your-app-domain> -p $TCP_PORT -u <MQTT_USER> -P <MQTT_PASSWORD> -t "test/topic"

# Publish a message
mosquitto_pub -h <your-app-domain> -p $TCP_PORT -u <MQTT_USER> -P <MQTT_PASSWORD> -t "test/topic" -m "Hello MQTT"
```

## Monitoring Your Application

Once deployed, you can monitor your application through:

- **Web Console**: The Clever Cloud console provides logs, metrics, and other tools to help you manage your application.
- **CLI**: Use `clever logs` to view application logs and `clever status` to check the status of your application.

## Additional Resources

- [Mosquitto Documentation](https://mosquitto.org/documentation/)
- [MQTT Protocol Specification](https://mqtt.org/mqtt-specification/)
- [Clever Cloud Linux Runtime Documentation](https://www.clever.cloud/developers/doc/applications/linux/)
- [Clever Cloud Documentation](https://www.clever-cloud.com/doc/)
