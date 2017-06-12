# Snap - How to extend API v2

## [Snap daemon side](https://github.com/intelsdi-x/snap)

### API v2 routes
All API v2 routes are available in file `./mgmt/rest/v2/api.go`. This file contains API method call routes depending on the HTTP request. There is Swagger ([swagger.io](http://swagger.io/)) metadata above each route, for generating OpenAPI ([www.openapis.org](https://www.openapis.org/)) Specification file used to autogenerate client.

### API v2 route Swagger template:
```go
// swagger:route <REQUEST_METHOD> <REQUEST_URL> <CATEGORY> <FUNCTION_NAME>
//
// <SUMMARY>
//
// <DESCRIPTION>
//
// Produces:
// <PRODUCED_DATA_TYPE>
//
// Schemes: http, https
//
// Responses:
// <RESPONSE_CODE>: <RESPONSE_ID>
// <RESPONSE_CODE>: <RESPONSE_ID>
// ...
api.Route{Method: "<REQUEST_METHOD>", Path: prefix + "<REQUEST_URL>", Handle: s.<FUNCTION_NAME>}
```

- `REQUEST_METHOD` - One of HTTP request methods, for example `GET`, `POST`, `PUT`, `DELETE`.
- `REQUEST_URL` - URL request part with parameter names surrounded with `{}`, for example `/plugins/{ptype}/{pname}/{pversion}/config`. These names will be passed by client. If request contains parameters, they should be defined in a struct, described with `swagger:parameters <FUNCTION_NAME>`, struct name doesn't matter, for example (more info about OpenAPI *Parameter Objects* [here](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#parameter-object)):

	```go
	// PluginConfigDeleteParams defines parameters for deleting a config.
	//
	// swagger:parameters deletePluginConfigItem
	type PluginConfigDeleteParams struct {
		// required: true
		// in: path
		PName string `json:"pname"`
		// required: true
		// in: path
		PVersion int `json:"pversion"`
		// required: true
		// in: path
		// enum: collector, processor, publisher
		PType string `json:"ptype"`
		// in: body
		// required: true
		Config []string `json:"config"`
	}
	```

- `CATEGORY` - Just category of API call used by Swagger for client code organization, each category will be separate package in generated client.

- `FUNCTION_NAME` - Name of function called by API request. Example function for getting plugin config by plugin type, name and version (you can use this function as template):
	```go
	func (s *apiV2) getPluginConfigItem(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
		var err error
		styp := p.ByName("type")
		if styp == "" {
			cfg := s.configManager.GetPluginConfigDataNodeAll()
			item := &PluginConfigItem{cfg}
			Write(200, item, w)
			return
		}

		typ, err := core.GetPluginType(styp)
		if err != nil {
			Write(400, FromError(err), w)
			return
		}

		name := p.ByName("name")
		sver := p.ByName("version")
		iver := -2
		if sver != "" {
			if iver, err = strconv.Atoi(sver); err != nil {
				Write(400, FromError(err), w)
				return
			}
		}

		cfg := s.configManager.GetPluginConfigDataNode(typ, name, iver)
		item := &PluginConfigItem{cfg}
		Write(200, item, w)
	}
	```
- `SUMMARY` - Short summary of API request, for example "Create Task"
- `DESCRIPTION` - Description of API request
- `PRODUCED_DATA_TYPE` - Data type of response. For Snap, it will be always `application/json`.
- `RESPONSE_CODE` - HTTP response code (number - 200, 404, 500 etc.)
- `RESPONSE_NAME` - Swagger response name for specific response code, for example `MetricResponse`. It should be a struct tagged by `swagger:response <RESPONSE_NAME>`, struct name doesn't matter, for example (more info about OpenAPI *Response Objects* [here](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#response-object)):

	```go
	// MetricsResponse is the representation of metric operation response.
	//
	// swagger:response MetricsResponse
	type MetricsResp struct {
		// in: body
		Body struct {
			Metrics []Metric `json:"metrics,omitempty"`
		}
	}
	```

After implementing new API call, just rebuild Snap with `make`, launch daemon and check responses using curl:

```sh
$ curl -X <REQUEST_METHOD> http://127.0.0.1:8181/v2/<REQUEST_URL>
```

for example:

```sh
$ curl -X GET 127.0.0.1:8181/v2/plugins
{
  "plugins": [
    {
      "name": "vsphere-vsan",
      "version": 1,
      "type": "collector",
      "signed": false,
      "status": "loaded",
      "loaded_timestamp": 1496827474,
      "href": "http://127.0.0.1:8181/v2/plugins/collector/vsphere-vsan/1"
    },
    {
      "name": "file",
      "version": 2,
      "type": "publisher",
      "signed": false,
      "status": "loaded",
      "loaded_timestamp": 1496841155,
      "href": "http://127.0.0.1:8181/v2/plugins/publisher/file/2"
    }
  ]
}
```

It is important for Swagger API specification to be correct, otherwise generating client will be impossible. OpenAPI Specification file is validated after every build, so - for example, if you see something like that in your Snap build log, probably you are missing *Parameter Object* for your API call:
```
"/home/user/workspace/go/src/github.com/intelsdi-x/snap/scripts/../swagger.json" is invalid against swagger specification 2.0. see errors :
- path param "{id}" has no parameter definition
```

If you see this message, everything is OK:
```
The swagger spec at "/home/user/workspace/go/src/github.com/intelsdi-x/snap/scripts/../swagger.json" is valid against swagger specification 2.0
```

#### What is API specification file?

API specification file for Snap is called `swagger.json` and can be found in root of snap repository. It contains JSON-formatted data about all structures needed for client to communicate. It is detailed Snap protocol specification in machine-readable format.

## [Snap client side](https://github.com/intelsdi-x/snap-client-go)

You need to have your Snap daemon API changes merged to official repo before making changes in client. Then `make` command will update client code automatically to latest API changes.

If you want to start working on client code before merging Snap daemon changes:

- Run `make deps`
- Replace `./vendor/github.com/intelsdi-x/snap/swagger.json` with new one
- Update client code with `make swagger`

## [Snap CLI side](https://github.com/intelsdi-x/snap-cli)

You need to have your updated Snap client merged to official repo before making changes in CLI.

If you want to modify CLI code before Snap client got merged:
- Run `make deps`
- Replace `./vendor/github.com/intelsdi-x/snap-client-go with your updated version
- Modify CLI code and use `make all` instead of `make` as build command to prevent restoring deps

In `./snaptel/commands.go` you need to add new command to public `Commands` and `Subcommands` slices. For example, for "snaptel metric list":
```go
{
	Name: "metric",
	Subcommands: []cli.Command{
		{
			Name:   "list",
			Usage:  "list",
			Action: listMetrics,
			Flags: []cli.Flag{
				flMetricVersion,
				flMetricNamespace,
				flVerbose,
			},
		},
		...
	},
}
```

Each command has list of flags, you define them in `./snaptel/flags.go`, for above example:

```go
// metric
flMetricVersion = cli.IntFlag{
	Name:  "metric-version, v",
	Usage: "The metric version. 0 (default) returns all. -1 returns latest.",
}
flMetricNamespace = cli.StringFlag{
	Name:  "metric-namespace, m",
	Usage: "A metric namespace",
}

// general
flVerbose = cli.BoolFlag{
	Name:  "verbose",
	Usage: "Verbose output",
}
```

Action is function assigned to specific command. Function definitions are inside `./snaptel/<command_category>.go`, for example `./snaptel/plugin.go` or `./snaptel/task.go`. Example:

```go
func someAction(ctx *cli.Context) error {
	// Getting command argument values
	boolParam := ctx.Bool("verbose")
	stringParam := ctx.String("metric-namespace")
	intParam := ctx.Int("metric-version")

	// Calling some client auto-generated call

	// Generate Parameters Object
	metricsParams := plugins.NewGetMetricsParamsWithTimeout(FlTimeout.Value)

	// Set up query parameters
	metricsParams.SetNs(&stringParam)
	metricsParams.SetVer(&intParam)

	// Send query using client
	resp, err := client.Plugins.GetMetrics(metricsParams)

	// Print results, etc.
}
```

And that's all, enjoy your newly implemented Snap API functionalities!
