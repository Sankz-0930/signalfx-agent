# Writing a Monitor

Monitors are what go out to the environment around the agent and collect
metrics about running services or platforms.  Adding a new monitor is
relatively simple.

We considered using the new Go 1.8+ plugin architecture to implement plugins
but have currently decided against it due to various issues still outstanding
with the plugin framework that make the benefits relatively minimal for a great
deal of added complexity and artifact size.  Therefore right now, new monitors
must be compiled into the agent binary.

First, create a new package within the `github.com/signalfx/neo-agent/monitors`
package (or inside the `monitors/collectd` package if creating a collectd
wrapper monitor).  Inside that package create a single module named whatever
you like that will hold the monitor code.  If your monitor gets complicated,
you can of course split it up into multiple modules or even packages as
desired.

Here is a minimalistic example of a monitor:

```go
	package mymonitor

	import (
		"time"
		"github.com/signalfx/golib/datapoint"
		"github.com/signalfx/neo-agent/core/config"
		"github.com/signalfx/neo-agent/monitors"
		"github.com/signalfx/neo-agent/monitors/types"
		"github.com/signalfx/neo-agent/utils"
		log "github.com/sirupsen/logrus"
	)

	func init() {
		monitors.Register("my-monitor",
			func() interface{} { return &Monitor{} },
			&Config{})
	}

	// Config for monitor
	type Config struct {
		config.MonitorConfig `yaml:",inline" acceptsEndpoints:"true"`

		# Required for monitors that accept auto-discovered endpoints
		Host string `yaml:"host"`
		Port uint16 `yaml:"port"`
		Name string `yaml:"name"`

		MyVar string `yaml:"myVar"`
	}

	// Validate will check the config for correctness.
	func (c *Config) Validate() error {
		if c.MyVar == "" {
			return errors.New("myVar is required")
		}
		return nil
	}

	// Monitor that collectd metrics.
	type Monitor struct {
		// This will be automatically injected to the monitor instance before
		// Configure is called.
		Output types.Output
		stop func()
	}

	// Configure and kick off internal metric collection
	func (m *Monitor) Configure(conf *Config) error {
		// Start the metric gathering process here.
		m.stop = utils.RunOnInterval(func() {

			# This would be a more complicated in a real monitor, but this
			# shows the basic idea of using the Output interface to send
			# datapoints.
		    m.Output.SendDatapoint(datapoint.New("my-monitor.requests",
			    map[string]string{"env": "test"}, 100, datapoint.Gauge, time.Now())

		}, time.Duration(conf.IntervalSeconds)*time.Second)

		return nil
	}

	// Shutdown the monitor
	func (m *Monitor) Shutdown() {
		// Stop any long-running go routines here
		if m.stop != nil {
			m.stop()
		}
	}
```

There are two data types that are essential to a monitor: the configuration and the
monitor itself.  By convention these are called `Config` and `Monitor` but the
names don't matter and can be anything you like.

## Configuration

The config struct is where any configuration of your monitor will go.  It must
embed the `github.com/signalfx/neo-agent/config.MonitorConfig` struct, which
includes generic configuration common to all monitors.  Configuration of the
agent (and also monitors) is driven by YAML and it is best practice to
explicitly state the yaml key for your config values instead of letting the
YAML interpreter derive it by default.  See [the golang YAML
docs](https://godoc.org/gopkg.in/yaml.v2) for more information.

Configuration fields can be of any type so long as it can be deserialized from
YAML.

By default, the agent will ensure any provided configuration matches the types
specified in your config struct.  If you want more advanced validation, you can
implement the `Validate() error` method on your config type.  This will be
called by the agent with the config struct fully populated with the provided
config before calling the `Configure` method of your monitor.  If it returns a
non-nil error, the error will be logged and the `Configure` method will not be
called.

The embedded `MonitorConfig` struct contains a field called `IntervalSeconds`.
Your monitor must make a good effort to send metrics at this interval, but
nothing is enforcing it so you have total freedom to follow the interval
however you like.  There is nothing in the agent that calls your monitor on a
regular interval.

Monitor config is considered immutable once configured.  That means that your
monitor's `Configure` method will never be called more than once for a given
monitor instance.  You are free to mutate the config instance within the
monitor code however, if desired.

### Auto Discovery

If your monitor is watching service endpoints that are appropriate for auto
discovery (e.g. a web service), you have to tell the agent this by specifying
the `acceptsEndpoints:"true"` tag on the embedded `MonitorConfig` struct in
your config struct type.  See the example above for what this looks like.
Then, you must specify three YAML fields in your config struct that all
discovered service endpoints provide in their configuration data:

- `host` (string): The hostname or IP address of the discovered service
- `port` (uint16): The port number of the service (can be TCP or UDP)
- `name` (string): A human-friendly name for the service as determined by the
observer that generated the endpoint.

These are normally called `Host`, `Port` and `Name`, but you can call them
whatever you like as long as the YAML name is correct.

You should also specify the `yaml:",inline"` tag of the embedded
`MonitorConfig` field so that observers that create endpoints with config that
overrides fields in that embedded struct can be correctly merged into the
config struct (e.g. the Kubernetes observer can set the interval via
annotations).

When an endpoint is discovered by an observer, the observer sets configuration
on the endpoint that then gets merged into your monitor's config before
`Configure` is called.

## Monitor Struct

Every monitor must have a struct type that defines it.  This is what gets
instantiated by the agent and what has the `Configure` method that gets called
after being instantiated.  A new instance of your monitor struct will be
created for each distinct configuration, so `Configure()` will only be called
once per monitor instance.

A monitor's interface is simple: there must be a `Configure` method and there
can optionally be a `Shutdown` method.  The `Configure` method must take a
pointer to the same config struct type registered for the monitor (see below
for registration).

There is a special field that can be specified by the monitor struct that will
be automatically populated by the agent:

- `Output "github.com/signalfx/neo-agent/monitors/types".Output`: This is what
	is used to send data from the monitor back to the agent, and then on to
	SignalFx.  This value has three methods:

	- `SendDatapoints([]*"github.com/signalfx/golib/datapoint".Datapoint)`:
		Sends datapoints, appending any extra dimensions specified in the
		configuration or by the service endpoint associated with the monitor.

	- `SendEvents([]*"github.com/signalfx/golib/event".Event)`: Sends events.

	- `SendDimensionProps(*"github.com/signalfx/neo-agent/monitors/types".DimProperties)`:
		Sends property updates for a specific dimension key/value pair.

The name and type of the struct field must be exactly as specified or else it
will not be injected.

## Registration

The [init function](https://golang.org/doc/effective_go.html#init) of your
package must register your monitor with the agent core.  This is done by
calling the `Register` function in the `monitors` package.  This function takes
three arguments:

1) The type of the monitor.  This is a string that should be dash delimited.
You will use this type in the agent configuration to identify the monitor.

2) A niladic factory function that returns a new uninitialized instance of your
monitor.

3) A reference to an uninitialized instance of your monitor's config struct.
This is used to perform config validation in the agent core, as well as to pass
the right type to the Configure method of the monitor.

The `Configure` method will receive a reference to the config struct that you
registered with the agent.  It is guaranteed to have passed its `Validate`
method, if provided.

## Create Dependency From Agent

To force the agent to compile and statically link in your new monitor code in
the binary, you must include the package in the
`github.com/signalfx/neo-agent/core/modules.go` module.

## Shutdown

Most monitors will need to do some kind of shutdown logic to avoid leaking
memory/goroutines.  This should be done in the `Shutdown()` method if your
monitor.  The agent will call this method if provided when the monitor is no
longer needed.  It should not block.

The `Shutdown()` method will not be called more than once.

If your monitor's configuration is changed in the agent, the agent will
shutdown existing monitors dependent on that config and recreate them with the
new config.

## Best Practices

 - It is best to send metrics immediately upon a monitor being configured and
   then at the specified interval so that metrics start coming out of the
   agent as soon as possible.  This will help minimize the chance of metric
   gaps.