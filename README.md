# PepperTea

PepperTea is an attempt at standardising application logs and metrics with a single, general purpose and descriptive format.

Both logs and metrics means that a PepperTea entity can be represented both as a metric (think StatsD) or a log (think JavaScript `console.log`).

The transition between metric or log *should* be invisible to the user, unless **the user wishes** to specifically mark an entity as either a log or a metric. This is left as an implementation detail rather than part of the entity.

By default, the system should decide based on priority if the entity should be a log, metric, or both, and deliver it to the appropriate destinations (syslog, application logger, StatsD, etc.).

## Entity

Below is a representation of the PepperTea entity in JSON format. Everything is required apart from the `detail` attribute.

```json
{
  "routing": {
    "time": "2014-12-02T14:55:23+0200",
    "priority": "5",
    "project": "PepperTea",
    "environment": "development",
    "source": "i-3f4g7e8c"
  },
  "state": {
    "context": ["http", "api", "server"],
    "transition": "restarted"
  },
  "detail": {
    "issuer": {"name": "Andrei Serdeliuc", "uid": "501"},
    "restart_duration": "500"
  }
}
```

The routing attribute *should* be determined by the system, not the user. The user **must** have the ability to override this.

The priority attribute is left vague because PepperTea will most probably be an extension of existing tools (e.g. python `logging `), and each one defines it’s own priorities.

As a suggestion, use [the syslog severity levels](http://en.wikipedia.org/wiki/Syslog#Severity_levels)

## Single line log format

Parts enclosed in `<>` are required, `[]` are optional; these symbols themselves should not be contained in log messages. Any optional segment can be followed by the same kind of segment, for example the first context level is required, but can be followed by many other contexts: `documentation.markdown.example`, or `api.http.auth.attempt`.

Parts enclosed in `{SECTION:}` are descriptive (for the purposes of this document only) and must not appear in actual entries.

Everything else is literal. Each entry must be on a single line, terminated by newline (`\n`). Each section separated by a single space.

```
{HEADER:}<time> <priority>
{ENVIRONMENT:}<project>.<environment>[.source]
{CONTEXT:}<context[.context]>:<state-transition>
{META:}[<name[.sublevel]>="<arg>"]
```

Example log output:

```
2014-12-02T14:55:23+0200 INFO PepperTea.development.i-3f4g7e8c http.api.server.restart:restarted issuer.name="Andrei Serdeliuc" issuer.uid="501" restart_duration="500"
```

The last `context` should describe the action being taken and the `state-transition` should be a verb describing its outcome:

```
http.api.server.start:started port="8080"
```

or

```
http.api.server.start:failed port="8080" reason="port in use"
```

### Stack traces

In a non local environment, ideally stack traces should be pushed to something like Rollbar or Sentry. Once pushed, the resulting log message should contain a `detail` attribute who's value represents the id of the message (Sentry id for example): `sentry.id="ae1d688fe60abb726cb0b10e58b23ca4"`

When this is not possible, the single line log should end in `\t\\\n`; each line of the stack trace should be preceded by `\t` and end in `\t\\\\n`, with the exception of the last line of the trace which should be terminated by `\n`.

Example log output:

```
2014-12-02T14:55:23+0200 INFO PepperTea.development.i-3f4g7e8c http.api.server.restart:failed issuer.name="Andrei Serdeliuc" issuer.uid="501" restart_duration="500"  \
  Traceback (most recent call last):  \
    File "test.py", line 1, in <module>   \
      python  \
  NameError: name 'python' is not defined

```

Reasoning behind both a prefix (`\t`) and a suffix (`\t\\n`) is that during migration periods,
a system will use PepperTea interchangeably with an existing logging system, which might
already use `\t` as a prefix.

## Metric format
Even though the `source` attribute might be available for an entity, its use in a metric must be determined by the priority of the entity, and this must be configurable by the user. For example a high priority entity would generate both a metric and a log, whereas a low priority entity might generate either or none.

### StatsD

`<project>.<environment>.<context>[.context].<state-transition>[.<source>]`

Example:

`PepperTea.development.http.api.server.restart.failed.i-3f4g7e8c`

### OpenTSDB
OpenTSDB supports tags, which allows us to use much more of the entity data than StatsD does.

TODO

# Implementation example

```python
# Initialise PepperTea
# TODO

# Create an event
user_dict = {"name": "Andrei Serdeliuc", "uid": 501}

pepper.info(["http", "api", "server", "restart"], "restarted", {"issuer": user_dict})
```

To make it more friendly, the above function call could be defined as a function itself:

```python
def http_api_server_restarted_by_user(user_dict):
  pepper.info(["http", "api", "server", "restart"], "restarted", {"issuer": user_dict})

http_api_server_restarted_by_user({"name": "Andrei Serdeliuc", "uid": 501})
```

The above should be taken as an example, since it relies on global state (the `pepper` object).
