# fdpCloud

General-purpose Linked Data server, named for Fair Data Points.

# Tools

## Apache2
Standard Apache2 server with these additional modules enabled with e.g. `a2enmod proxy_http`
* proxy
* proxy_http
* ssl
* maybe socache_shmcb.load ? don't know how that got there.

The `000-default` config in /etc/apache/sites-enabled is removed; replaced with:
`fdpcloud.org.conf -> ../sites-available/fdpcloud.org.conf`

If you modify configs or modules run this as root: `apachectl graceful`

## webhook
Debian package to handle webhooks; runs on localhost:9000. An apache reverse proxy shuttles requests to it.

## Certbot
Needs attention every 3 months. This will prompt you to write to the DNS record:
``` bash
sudo certbot certonly --manual \
 --preferred-challenges=dns \
 --email webmaster@fdpcloud.org \
 --server https://acme-v02.api.letsencrypt.org/directory
 --agree-tos \
 -d fdpcloud.org -d *.fdpcloud.org
```

## screen
General-purpose screen multiplexer. The standard config set up below under [#reboot/start] sets up consoles for maintanance and debugging.
Here's how to invoke screen:
| Description 				| Command 				|
|---------------------------------------|---------------------------------------|
| Start a new session with session name | `screen -US <session_name>`		|
| List running sessions / screens	      | `screen -ls`				|
| Attach to a running session		        | `screen -x`				|

The (default) attention key is control a (`^a` below). Here are the commands used below:

| Description				| Command								|
|---------------------------------------|-----------------------------------------------------------------------|
| Create new window 			| `^a c`								|
| Change to window by number 		| `Ctrl-a <number>` (only for windows 0 to 9)				|
| See window list 			| `^a "` (allows you to select a window to change to)		|
| Show window bar 			| `^a w` (if you don't have window bar)				|

`^a "` is very cool; try it.

See [Screen Quck Reference](https://gist.github.com/jctosta/af918e1618682638aa82) for more tips.

# Debian packages

| name                        | version           | architecture | description                                           |
|-----------------------------|-------------------|--------------|-------------------------------------------------------|
| apache2                     | 2.4.52-1ubuntu4.6 | amd64        | Apache HTTP Server                                    |
| apache2-bin                 | 2.4.52-1ubuntu4.6 | amd64        | Apache HTTP Server (modules and other binary files)   |
| apache2-data                | 2.4.52-1ubuntu4.6 | all          | Apache HTTP Server (common files)                     |
| apache2-utils               | 2.4.52-1ubuntu4.6 | amd64        | Apache HTTP Server (utility programs for web servers) |
| python3-certbot-apache      | 1.21.0-1          | all          | Apache plugin for Certbot                             |
| webhook                     | 2.8.0-3           | amd64        | Small server for creating HTTP endpoints (hooks)      |

# Tasks

## reboot/start
``` bash
$ restart
```
... a browser reload will tell you when Apache has restarted ...
``` bash
ssh fdpcloud.org
sudo su -
  screen -US console              # screen window 0 watches webhook
    ^aA ^h^h^h^hwebhook monitor
    sudo su - fdpCloud
      webhook -debug hooks.json
  ^ac                             # screen window 1 is for adding repos
    ^aA ^h^h^h^hadd repos
    cd sites/
  ^ac # screen window 2 is for editing Apache conf
    ^aA ^h^h^h^hApache.conf
    emacs /etc/apache2/sites-enabled/fdpcloud.org.conf
  ^ac # screen window 3 is for restarting Apache
    ^aA ^h^h^h^hApache control
```

## add repo
- create repo (possibly under StaticFDP)
- copy HTTPS URL from Code button but chop skip the trailing `.git` (e.g. `https://github.com/StaticFDP/FlashCard1`)
``` bash
ssh fdpcloud.org
sudo su -
screen -x console
^a 1 # screen window 1
  (cd github/ && git clone https://github.com/StaticFDP/wikidata StaticFDP/FlashCard1)
^a 2
```
Here we cound on your emacs-fu to scroll down to the Subdomains sections in both HTTP and HTTPS and copy e.g. the wikidata entry to make your new subdomain. It should look a bit like:
```
<VirtualHost *:80>
  ServerName &lt;lower-case domain name, e.g. FlashCard1&gt;.fdpcloud.org
  DocumentRoot /home/fdpCloud/sites/github/StaticFDP/FlashCard1
  <Directory /home/fdpCloud/sites/github/StaticFDP/FlashCard1>
    include ./sites-available/fdpcloud-dirConfig
  </Directory>
  include ./sites-available/fdpcloud-common
</VirtualHost>
```
- Now back to screen:
``` bash
^a 3
  apachectl graceful
```
- Go back to repo and create a webhook (Settings Webhhoks)
  - Payload URL: `https://fdpCloud.org/HOOKS/updateFromGithub`
  - Content type: `application/json`
  - Secret: `yJJON3k6tT` - - - yes, same for everyone. could change to "NoDOSPlease"
  - ☑ Enable SSL verification
  - ☑ just the `push` event
  - ☑ Active
  - [Update Webhook]

This should immediate fire a push webhook. you can see the results in the webhook monitor screen.
