# mqtt-launcher

_mqtt-launcher_ is a Python program which subscribes to a set of [MQTT] topics
and executes processes on the host it's running on. Launchable processes are
configured on a per/wildcard basis, and they can be constrained to run only if
a particular text payload is contained in the message.

For example, I can publish a message to my MQTT broker requesting _mqtt-launcher_
create a particular semaphore file for me:

```
mosquitto_pub -t sys/file -m create
```

The configuration file(s) must be valid Python. Configuration is loaded once and contains
the topic / process associations.

```python
# topic         payload value           program & arguments
"sys/file"  :   {
                    'create'        :   [ '/usr/bin/touch', '/tmp/file.one' ],
                    'false'         :   [ '/bin/rm', '-f', '/tmp/file.one'    ],
                    'info'          :   [ '/bin/ls', '-l', '/tmp/file.one' ],
                },
```

Above snippet instructs _mqtt-launcher_ to:

* subscribe to the [MQTT] topic `sys/file`
* look up the payload string and launch the associated programs:
  * if the payload is `create`, then _touch_ a file
  * if the payload is the string `false`, remove a file
  * if the payload is `info`, return information on the file

The payload value may be `None` in which case the eacho of the list elements
defining the program and arguments are checked for the magic string `@!@` which
is replaced by the payload contents. (See example published at `dev/2`, `dev/3` and `dev/4` below.)

_mqtt-launcher_ publishes _stdout_ and _stderr_ of the launched program
to the configured topic with `/report` added to it. So, in the example
above, a non-retained message will be published to `sys/file/report`.
(Note that this message contains whatever the command outputs; trailing
white space is truncated.)

## Screenshot

Here's the obligatory "screenshot".

```
Publishes					Subscribes
-----------------------		------------------------------------------------------------------
						$ mosquitto_sub -v -t 'dev/#' -t 'sys/file/#' -t 'prog/#'


mosquitto_pub -t prog/pwd -n
						prog/pwd (null)
						prog/pwd/report /private/tmp

mosquitto_pub -t sys/file -m create
						sys/file create
						sys/file/report (null)	# command has no output

mosquitto_pub -t sys/file -m info
						sys/file info
						sys/file/report -rw-r--r--  1 jpm  wheel  0 Jan 22 16:10 /tmp/file.one

mosquitto_pub -t sys/file -m remove
						sys/file remove
						# report not published: subcommand ('remove') doesn't exist
						# log file says:
						2014-01-22 16:11:30,393 No matching param (remove) for sys/file

mosquitto_pub -t dev/1 -m hi
						dev/1 hi
						dev/1/report total 16231
						drwxrwxr-x+ 157 root  admin     5338 Jan 20 10:48 Applications
						drwxrwxr-x@   8 root  admin      272 Jan 25  2013 Developer
						drwxr-xr-x+  72 root  wheel     2448 Oct 14 10:54 Library
						...
mosquitto_pub -t dev/2 -m 'Hi Jane!'
						dev/2 Hi Jane!
						dev/2/report 111 * Hi Jane! 222 Hi Jane! 333

mosquitto_pub -t dev/3 -m 'foo-bar'
						dev/3 foo-bar
						dev/3/report foo-bar

mosquitto_pub -t dev/4 -m 'foo/bar'
						dev/4 foo/bar
						dev/4/report var1=foo var2=bar
```

## Configuration

_mqtt-launcher_ loads a Python configuration from the path contained in
the environment variable `$MQTTLAUNCHERCONFIG`; if unset, the path
defaults to `launcher.conf`. See the provided `launcher.conf.example`.

Additional configuration files may be placed in a subdirectory of the
main configuration file path. These are read in lexical order and merged
into the final configuration, which allows for per-machine overrides and
separate storage of settings like the broker password. The subdirectory
must be named the same as the configuration file with a `.d` suffix. Example:
```
├── launcher.conf
└── launcher.conf.d
    ├── 10-password
    └── 20-topics
```

## Logging

_mqtt-launcher_ logs its operation in the file configured as `logfile`.

## Requirements

* Python
* [paho-mqtt](https://pypi.python.org/pypi/paho-mqtt/1.3.1)

## Credits

This program was inspired by two related tools:
* Peter van Dijk's [mqtt-spawn](https://github.com/PowerDNS/mqtt-spawn)
* Dennis Schulte's [mqtt-exec](https://github.com/denschu/mqtt-exec). (I'm not terribly comfortable running NodeJS programs, so I implemented the idea in Python.)

 [MQTT]: http://mqtt.org
