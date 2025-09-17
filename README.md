# Sample Grist self-hosted configuration

This is a fairly complete sample Docker Compose configuration
synthesised from these two existing examples:

- [Traefik basic auth](https://github.com/gristlabs/grist-core/tree/main/docker-compose-examples/grist-traefik-basic-auth)
- [Keycloak, Postgres, Redis, Minio](https://github.com/gristlabs/grist-core/tree/main/docker-compose-examples/grist-with-keycloak-postgres-redis-minio)

# Instructions

Here is how to get a self-hosted Grist instance using this example.

## Get an Ubuntu server somewhere with a domain name

For this example, we'll be using a t2.small EC2 instance on AWS at my
personal domain `gristcon.jordigh.com`.

Use `ssh` to log into your server. We'll be doing most of the
configuration over the terminal.

## Clone this repo

```sh
git clone https://github.com/jordigh/GristCon2025.git
cd GristCon2025
```

## Install Docker

For convenience, there is an included install script.

```sh
sudo ./install-docker
```

This will install Docker and grant the necessary permissions to use it
without sudo.

After installation, log out and log back in with `ssh`.

## Fill in env-example

There is a template for environment variables used to configure this:

```sh
cp env-example .env
vim .env
```

The values should be as follows:

- `GRIST_DOMAIN`: The domain name where Grist will be hosted (in this
  example, `gristcon.jordigh.com`
- `GRIST_DEFAULT_EMAIL`: The email of the admin user (in this example,
  `jordi@getgrist.com`)
- `PERSIST_DIR`: Persistent storage for all of the Docker containers.
  This includes Keycloak's database as well as Grist's documents. The
  default value is fine for now.
- `MINIO_PASSWORD`: A random-looking string. It has to be long and unguesable.
- `DATABASE_PASSwORD`: Same as above. The same password will be used
  for Grist's home database as well as Keycloak's database.
- `OIDC_CLIENT_SECRET`: Leave it blank for now. We will fill this in
  later once we've setup keycloak.

## Start up Compose

If everything has been set up as above, the following should start up
most services:

```sh
docker compose up
```

You should see minio, Redis, two Postgres instances, KeyCloak start
up. Finally, Grist should fail to start with the error message `Realm
does not exist`. This is expected, because KeyCloak needs to be
configured via its web interface.

## Configure logins

We will do this in two phases. During the first startup, everything
except Grist should start. In particular, Keycloak will be available
at `https://$GRIST_DOMAIN/grist` (in this example, at
https://gristcon.jordigh.com/keycloak ). Once Keycloak is configured,
we will go back and define `OIDC_CLIENT_SECRET` for Grist so that it
can connect to Keycloak and use it for authentication.


### Configure Keycloak

We need to create a realm, user, and an OIDC client

#### Create a realm and user

For Keycloak, we need to create a new realm named `myrealm` (this name
matches a name provided in the `compose.yaml` file), and a user in
this realm. The instructions to do that are
[here](https://www.keycloak.org/getting-started/getting-started-docker).

**IMPORTANT**: The user must be given the same email as the one
defined in `GRIST_DEFAULT_EMAIL`. This will make the first user an
administrator of the Grist instance.

#### Create a Grist OIDC client

The next step is to create a client. The instructions are provided
[here](https://support.getgrist.com/install/oidc/#example-keycloak).
Use the following settings:

1. **`clientid`**: `gristclient` (click Next)
2. **Client authentication**: On
3. **Standard flow**: On (click Next)
4. **Root URL**: `https://GRIST_DOMAIN` (in this example,
   `https://gristcon.jordigh.com` )
5. **Valid redirect URIs**: `/oauth2/callback`
6. **Valid post logout URIs**: `/*`(click Save)

Once the client is created, click on the **Credentials** tab and copy
the **ClientSecret** value.

### Finish configuring Grist

Stop the Compose service (Ctrl+C). Go back into the `.env` file and
fill in the `OIDC_CLIENT_SECRET` variable with the value copied above.

Save and exit run `docker compose up` again.

This time all services should start, including Grist.

## Log in to Grist

Visiting the Grist domain at `https://$GRIST_DOMAIN` (in this example,
https://gristcon.jordigh.com ), should now redirect to Keycloak.
Logging in with the credentials we used before should work, as well as
logging out.

We can verify in the admin panel if everything works well.

## Adding other login methods

One of the advantages of Keycloak is that it can be used to add other
identity providers such as Google, Microsoft, or Github. These can be
used as alternative methods for logging into Grist.

We will do as an example a Github login.

### Create a new OAuth app

Navigate to [github's developer
settings](https://github.com/settings/developers), and create a new
OAuth app. We will register our Keycloak as an OAuth app on Github.

Use the following settings:

1. **Application name**: Grist
2. **Homepage URL**: `https://$GRIST_DOMAIN/keycloak` (in this example,
   `https://gristcon.jordigh.com/keycloak` )
3. **Authorization callback URL**:
   `https://$GRIST_DOMAIN/keycloak/realms/myrealm/broker/github/endpoint`
   (in this example
   `https://gristcon.jordigh.com/keycloak/realms/myrealm/broker/github/endpoint` )

This should now give you a client ID. Click on **Generate new client
secret** to also obtain a client secret.

### Add Github as an identity provider on Keycloak

Log back into the Keycloak admin console at
`https://$GRIST_DOMAIN/keycloak` (in this example,
https://gristcon.jordigh.com/keycloak ), and make sure you're in the
`myrealm` realm.

1. Click on **Identity Providers** in the left side bar.
2. Click on the Github icon
3. Paste the **Client ID** and **Client Secret** obtained from the
   OAuth application in Github as above.
4. Scroll down a bit to **Trust Email** and set it to On
5. Click **Add**.

### Test github login

That should now work!
