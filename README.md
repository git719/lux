# My SCM 
A personal [Source Control Management (SCM)](https://en.wikipedia.org/wiki/Version_control) system, like [Github](https://github.com) or [BitBucket](https://bitbucket.org), using [Gitea](https://gitea.io/en-us/), [MySQL](https://www.mysql.com/), and fronted by [Traefik](https://containo.us/traefik/). This particular Gitea implementation uses a [Docker](https://en.wikipedia.org/wiki/Docker_(software)) Compose multi-container setup on Ubuntu Linux (`docker-compose.yml`). This is essentially an extended version of the intructions at https://docs.gitea.io/en-us/install-with-docker/.

This uses and creates the following:

* DNS domains hosted at [DigitalOcean](https://www.digitalocean.com/)
* TLS certificates using [LetsEncrypt](https://letsencrypt.org/), with DNS challenge
* Traefik API and Dashboard with basic HTTP auth login
* Gitea self-hosted git service
* MySQL database for Gitea
* Docker images:
  * traefik = <https://hub.docker.com/_/traefik?tab=tags>
  * gitea = <https://hub.docker.com/r/gitea/gitea/tags>
  * mysql = <https://hub.docker.com/_/mysql?tab=tags>

## Getting started
* Set up a Linux Ubuntu server, and install Docker and Docker Compose
* Checkout this repo: `git clone REPO_URL myscm`
* We'll check it out to the `myscm` directory, but you can use any other name
* Switch to the `myscm` directory
* Create an `.env` file and populate the parameters _to your requirements_, with more **secured passwords**:
```
# .env
VER_TRAEFIK="v2.6.0" # https://hub.docker.com/_/traefik?tab=tags
VER_GITEA="1.15.10"  # https://hub.docker.com/r/gitea/gitea/tags
VER_MYSQL="8.0.28"   # https://hub.docker.com/_/mysql?tab=tags
#
DO_AUTH_TOKEN=d31fdc49_Use-Your-Own-Digital-Ocean-Token!_fd31fdc3
LE_EMAIL=info@myscm.com
SITE_DOMAIN=myscm.com
IPADDRESS=10.10.5.2
TRAEFIK_DOMAIN=traefik.myscm.com
TRAEFIK_CREDS="traefik:$apr1$TUWPhgKR$PSMWDX.ReIJPP4eQF4raH1"
# Generate with: echo $(htpasswd -nb username Password)
# Note that '$' don't need to be escaped, since string from .env file is treated as literal by docker-compose
DB_NAME=gitea
DB_USER=gitea
DB_PASSWD=gitea
MYSQL_ROOT_PASSWORD=gitea
# Below values must coincide with above DB_* ones
MYSQL_DATABASE=gitea
MYSQL_USER=gitea
MYSQL_PASSWORD=gitea
```
* Create an empty `acme.json` file, and give it the required Traefik permissions
```
touch acme.json
chmod 600 acme.json
```
* Note that for security reasons files `.env` and `acme.json` are obviously not kept in this repo!
* Next, create the external `myscm_net_web` network:
```
docker network create --gateway 10.10.5.1 --subnet 10.10.5.0/24 myscm_net_web
```
Of course, this network can be named `myscm_net_web` or whatever else you want. And the CIDR block can also be anything you want, as long as it is routable back to the Linux host. Just make sure that `myscm.com` and `traefik.myscm.com` point to 10.10.5.2, or whatever the first IP address is for the block you're using.

* Bring up the system: `docker-compose --verbose up -d`

## Configuration
Once the system is running, go to your domain http://myscm.com, click _Sign In_ and do the final system configuration. Also, the Traefik Dashboard should be available at http://traefik.myscm.com. Both links should switch from HTTP to HTTPS using SSL certs provided by LetsEncrypt.

## SSH CLI Access
The system is accessible via a browser, but you can also access it via SSH using user `git`, on your domain `myscm.com`, and over port `22`. To simplify this access, it's easier to update your `$HOME/.ssh/config` file with a stanza such as this:
```
Host                     myscm.com
  User                   git
  IdentityFile           /home/user/.ssh/id_ed25519
  StrictHostKeyChecking  no
  UserKnownHostsFile     /dev/null
```

Of course, you'll need to generate your public key and add it to your user profile by:
* clicking on your user's avatar and selecting _Settings_
* click on _SSH/GPG Keys_ => _Manage SSH Keys_ => _Add Key_

## Backup and Restore
To backup a running system, run the `./backup` script. Note that the user context must have sudo privilege on the Linux Docker host, in order to do a raw tar backup of the volumes under `/var/lib/docker/volumes/`.

Note, this is not a real-time backup process, so the service _will be stopped_ temporarily during this backup.

The resulting backup will produce a `backup.tgz` file made up of:
```
/var/lib/docker/volumes/db/*
/var/lib/docker/volumes/gitea/*
./.env
./acme.json
```

To restore a system, make sure you have a `backup.tgz` file, then run the `./restore` script. Again, the user needs to have sudo privilege.

For a full recovery from a dump file, you will need to create the external `myscm_net_web` network beforehand (see command above)

## Post-installation Hardening and Tweaks
After initial setup and login, do the following for better security:
```
docker-compose stop
sudo -i
vi /var/lib/docker/volumes/gitea/_data/gitea/conf/app.ini
```
* Under the `[service]` section update the following:
```
DISABLE_REGISTRATION              = true
REQUIRE_SIGNIN_VIEW               = true
```
* Under section `[openid]` update below:
```
ENABLE_OPENID_SIGNIN = false
ENABLE_OPENID_SIGNUP = false
```
* At this point you may also want to update the `[repository]` section with the following:
```
DEFAULT_BRANCH = main
```
Finally, exit the sudo session and restart the system:
```
exit
docker-compose start
```
Then confirm all above settings are active by login in via the web UI and:
* clicking on your user's avatar and selecting _Site Administration_
* then under the _Configuration_ tab review the _Service Configuration_ section

## Upgrades
To upgrade your installation to the latest release(s):
* Edit each of the `ENV_*` variables in the `.env` file
* Tear everything down: `docker-compose down`
* Then start all afresh: `docker-compose --verbose up -d`

## References
1. https://docs.gitea.io/en-us/install-with-docker/
2. https://www.howtoforge.com/tutorial/install-gitea-using-docker-on-ubuntu/
3. https://docs.traefik.io/routing/routers/#certresolver
4. https://stackoverflow.com/questions/61774228/traefik-cant-connect-to-server-with-docker-compose
5. https://www.reddit.com/r/Traefik/comments/e9ubk2/le_wildcard_certificates_on_traefik_v2/
6. https://containo.us/blog/traefik-2-0-docker-101-fc2893944b9d/
7. https://www.smarthomebeginner.com/traefik-2-docker-tutorial/
