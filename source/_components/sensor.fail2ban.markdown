---
layout: page
title: "Fail2Ban Sensor"
description: "Instructions on how to integrate a fail2ban sensor into Home Assistant."
date: 2017-10-19 10:30
sidebar: true
comments: false
sharing: true
footer: true
ha_category: Sensor
ha_iot_class: "Local Polling"
logo: fail2ban.png
ha_release: 0.57
---


The `fail2ban` sensor allows for IPs banned by [fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page) to be displayed in the Home Assistant frontend.

<p class='note'>
Your system must have `fail2ban` installed and correctly configured for this sensor to work. In addition, Home Assistant must be able to read the `fail2ban` log file.
</p>

To enable this sensor, add the following lines to your `configuration.yaml`:

```yaml
# Example configuration.yaml entry
sensor:
  - platform: fail2ban
    jails:
      - ssh
      - hass-iptables
```

{% configuration %}
jails:
  description: List of configured jails you want to display.
  required: true
  type: list
name:
  description: Name of the sensor.
  required: false
  type: string
  default: fail2ban
file_path:
  description: Path to the fail2ban log.
  required: false
  type: string
  default: /var/log/fail2ban.log
{% endconfiguration %}

### {% linkable_title Set up Fail2Ban %}

For most setups, you can follow [this tutorial](/cookbook/fail2ban/) to set up `fail2ban` on your system. It will walk you through creating jails and filters, allowing you to monitor IP addresses that have been banned for too many failed SSH login attempts, as well as too many failed Home Assistant login attempts.

### {% linkable_title Fail2Ban with Docker %}

<p class='note'>
These steps assume you already have the Home Assistant docker running behind NGINX and that it is externally accessible. It also assumes the docker is running with the `--net='host'` flag.
</p>

For those of us using Docker, the above tutorial may not be sufficient. The following steps specifically outline how to set up `fail2ban` and Home Assistant when running Home Assistant within a Docker behind NGINX. The setup this was tested on was an unRAID server using the [let's encrypt docker](https://github.com/linuxserver/docker-letsencrypt) from linuxserver.io.

#### {% linkable_title Set http logger %}

In your `configuration.yaml` file, add the following to the `logger` component to ensure that Home Assistant prints failed login attempts to the log.

```yaml
logger:
  logs:
    homeassistant.components.http.ban: warning
```

#### {% linkable_title Edit the `jail.local` file %}

Next, we need to edit the `jail.local` file that is included with the Let's Encrypt docker linked above.  Note, for this tutorial, we'll only be implementing the `[hass-iptables]` jail from the [previously linked tutorial](/cookbook/fail2ban/).

Edit `/mnt/user/appdata/letsencrypt/fail2ban/jail.local` and append the following to the end of the file:

```
[hass-iptables]
enabled = true
filter = hass
action = iptables-allports[name=HASS]
logpath = /hass/home-assistant.log
maxretry = 5
```

#### {% linkable_title Create a filter for the Home Assistant jail %}

Now we need to create a filter for `fail2ban` so that it can properly parse the log.  This is done with a `failregex`.  Create a file called `hass.local` within the `filter.d` directory in `/mnt/user/appdata/letsencrypt/fail2ban` and add the following:

```
[INCLUDES]
before = common.conf

[Definition]
failregex = ^%(__prefix_line)s.*Login attempt or request with invalid authentication from <HOST>.*$

ignoreregex =

[Init]
datepattern = ^%%Y-%%m-%%d %%H:%%M:%%S
```

#### {% linkable_title Map log file directories %}

First, we need to make sure that fail2ban log can be passed to Home Assistant and that the Home Assistant log can be passed to fail2ban.  When starting the Let's Encrypt docker, you need to add the following argument (adjust paths based on your setup):

```
/mnt/user/appdata/home-assistant:/hass
```

This will map the Home Assistant configuration directory to the Let's Encrypt docker, allowing `fail2ban` to parse the log for failed login attempts.

Now do the same for the Home Assistant docker, but this time we'll be mapping the `fail2ban` log directory to Home Assistant so that the fail2ban sensor is able to read that log:

```
/mnt/user/appdata/letsencrypt/log/fail2ban:/fail2ban
```


#### {% linkable_title Send client IP to Home Assistant %}

By default, the IP address that Home Assistant sees will be that of the container (something like `172.17.0.16`).  What this means is that for any failed login attempt, assuming you have correctly configured `fail2ban`, the Docker IP will be logged as banned, but the originating IP is still allowed to make attempts.  We need `fail2ban` to recognize the originating IP to properly ban it.

First, we have to add the following to the nginx configuration file located in `/mnt/user/appdata/letsencrypt/nginx/site-confs/default`.

```bash
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

This snippet should be added within your Home Assistant server config, so you have something like the following:

```bash
server {
    ...
    location / {
        proxy_pass http://192.168.0.100:8123;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api/websocket {
        proxy_pass http://192.168.0.100:8123/api/websocket;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    ...
}
```

Once that's added to the nginx configuration, we need to modify the Home Assistant `configuration.yaml` such that the `X-Forwarded-For` header can be parsed.  This is done by adding the following to the `http` component:

```yaml
http:
  use_x_forwarded_for: True
```

At this point, once the Let's Encrypt and Home Assistant dockers are restarted, Home Assistant should be correctly logging the originating IP of any failed login attempt.  Once that's done and verified, we can move onto the final step.

#### {% linkable_title Add the fail2ban sensor %}

Now that we've correctly set everything up for Docker, we can add our sensors to `configuration.yaml` with the following:

```yaml
sensor:
  - platform: fail2ban
    jails:
      - hass-iptables
    file_path: /fail2ban/fail2ban.log
```

Assuming you've followed all of the steps, you should have one fail2ban sensor, `sensor.fail2ban_hassiptables`, within your front-end.

### {% linkable_title Other debug tips %}

If, after following these steps, you're unable to get the `fail2ban` sensor working, here are some other steps you can take that may help:

- Add `logencoding = utf-8` to the `[hass-iptables]` entry
- Ensure the `failregex` you added to `filter.d/hass.local` matches the output within `home-assistant.log`
- Try changing the datepattern in `filter.d/hass/local` by adding the following entry (change the datepattern to fit your needs). [source](https://github.com/fail2ban/fail2ban/issues/174)
    ```
    [Init]
    datepattern = ^%%Y-%%m-%%d %%H:%%M:%%S
    ```
