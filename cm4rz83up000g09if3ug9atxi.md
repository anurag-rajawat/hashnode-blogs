---
title: "Building an Envoy Wasm Plugin in Rust"
seoTitle: "Create Envoy Wasm Plugin Using Rust"
seoDescription: "Learn to build a Wasm plugin in Rust for Envoy, enabling passive observation of RESTful and gRPC API calls"
datePublished: Tue Dec 17 2024 04:41:31 GMT+0000 (Coordinated Universal Time)
cuid: cm4rz83up000g09if3ug9atxi
slug: building-an-envoy-wasm-plugin-in-rust
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1734409180971/ea073759-96be-43f3-8782-28fc158ce6e8.png
tags: rust, wasm, envoy, service-mesh

---

# Introduction

In this hands-on tutorial, we'll delve into creating a WebAssembly (WASM) HTTP plugin for Envoy Proxy. This plugin will enable us to passively observe the RESTful and gRPC API call traffic without interfering with them.

# Prerequisites

* **Rust Programming Language**
    
* **Envoy Proxy:** A high-level understanding of Envoy Proxy's architecture and functionality.
    
* **A Curious Mind:** A willingness to explore new concepts and technologies.
    

Let's dive in!

# When Envoy meets Wasm

[Envoy](https://www.envoyproxy.io) is an open-source service proxy designed for cloud-native applications. It offers a wide range of features, including connection pooling, retry mechanisms, TLS management, and rate limiting.

## Extending Envoy

What if you need a feature that Envoy doesn't offer out of the box?

### Traditional Way

Traditionally, adding custom logic to Envoy involved modifying its core code, recompiling, and redeploying. This process is not only time-consuming but also risky.

### Modern Way

A Flexible Approach Using Plugins.

Plugins provide a more elegant solution. You can extend Envoy's capabilities by writing custom plugins without altering its core functionality.

One of the most exciting developments in the plugin ecosystem is the support for [WebAssembly](https://webassembly.org) (Wasm). Wasm allows you to write plugins in languages like Rust, C++, and more, and compile them into a portable bytecode format. This enables high-performance, secure, and language-agnostic plugins.

We’ll write our Wasm plugin in Rust.

Read more about Wasm in Envoy [here](https://github.com/proxy-wasm/spec/blob/main/docs/WebAssembly-in-Envoy.md) and Proxy Wasm specification [here](https://github.com/proxy-wasm/spec/blob/main/abi-versions/v0.2.1/README.md).

# Implementation

## High Level Workflow

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734361430239/05d26ccd-18b4-4375-b81f-e36270a21c8e.png align="center")

## Envoy Wasm Filter

Let’s initialize a new rust library.

```bash
cargo new --lib --edition 2021 --vcs git envoy-wasm-filter
```

You should have a similar directory tree:

```bash
$ tree envoy-wasm-filter
envoy-wasm-filter
├── Cargo.toml
└── src
    └── lib.rs

2 directories, 2 files
```

Update `Cargo.toml` file to tell Rust that this is a library:

```ini
[package]
name = "envoy-wasm-filter"
version = "0.1.0"
edition = "2021"
authors = ["Anurag Rajawat", "anuragsinghrajawat22@gmail.com"]

[lib]
name = "httpfilter"
path = "src/lib.rs"
crate-type = ["cdylib"] # Tell `rustc` to build a dynamic linked library

[profile.release]
# Tell `rustc` to optimize for small code size.
opt-level = "s"
```

Install the rust wasm toolchain:

```bash
rustup target add wasm32-unknown-unknown
```

Let’s write some code, open the `src/lib.rs` file in your favourite code editor.

```rust
use std::collections::HashMap;

/// Metadata about an API event, including context ID, timestamp, and information about the traffic source and node.
#[derive(Default)]
struct Metadata {
    context_id: u32,
    timestamp: u64,

    /// Name of the traffic source being used (e.g., Envoy, Istio, Consul).
    traffic_source_name: String,

    /// Version of the traffic source being used (e.g., 1.31, 1.24).
    traffic_source_version: String,

    /// The name of the Kubernetes node where the workload is running. If the workload
    /// is not running in a Kubernetes environment, this field is empty.
    node_name: String,
}

/// Represents an incoming HTTP request, including headers and body.
#[derive(Default)]
struct Reqquest {
    headers: HashMap<String, String>,
    body: String,
}

/// Represents an outgoing HTTP response, including headers and body.
#[derive(Default)]
struct Ressponse {
    headers: HashMap<String, String>,
    body: String,
}

/// Represents a generic workload, which can be a Kubernetes or non-Kubernetes resource.
/// It serves as a source or destination for access within a system.
#[derive(Default)]
struct Workload {
    /// Name of the workload.
    name: String,

    /// The namespace in which the workload is deployed. This field is only applicable
    /// for Kubernetes workloads.
    namespace: String,

    /// IP address of the workload.
    ip: String,

    /// Port number used by the workload.
    port: u16,
}

/// Represents an API event, encapsulating metadata, request, response, source, and destination workloads.
#[derive(Default)]
struct APIEvent {
    /// Metadata about the API event.
    metadata: Metadata,

    /// Incoming HTTP request.
    request: Reqquest,

    /// Outgoing HTTP response.
    response: Ressponse,

    /// Source workload of the API call.
    source: Workload,

    /// Destination workload of the API call.
    destination: Workload,

    protocol: String,
}

/// Configuration for the plugin.
#[derive(Default)]
struct PluginConfig {
    /// Name of the upstream service.
    upstream_name: String,

    /// Path to the upstream service.
    path: String,

    /// Authority of the upstream service.
    authority: String,
}

/// Represents a plugin instance, holding configuration and API event information.
#[derive(Default)]
struct Plugin {
    /// Context ID for the plugin instance.
    _context_id: u32,

    /// Configuration for the plugin.
    config: PluginConfig,

    /// API event being processed by the plugin.
    api_event: APIEvent,
}

/// Maximum allowed size for request and response bodies.
const MAX_BODY_SIZE: usize = 1_000_000; // 1MB
```

These type definitions are self-explanatory. Note that we've implemented the `Default` trait on all types to provide their default values.

Now let’s add the proxy-wasm dependency:

```bash
cargo add proxy-wasm
```

Now let’s write our business logic: observe all the incoming/outgoing RESTful and gRPC API calls.

```rust
use proxy_wasm::traits::{Context, RootContext};
use proxy_wasm::types::LogLevel;

// ...
// Existing code
// ...


impl Context for Plugin {}

impl RootContext for Plugin {}

/// This is the entry point for the Wasm module and is part of ABI specification.
fn _start() {
    proxy_wasm::main! {{
        // Set the Log level to `warning`
        proxy_wasm::set_log_level(LogLevel::Warn);

        // Set the root context of the filter to a new instance of the Plugin type with its default value.
        proxy_wasm::set_root_context(|_| -> Box<dyn RootContext> {Box::new(Plugin::default())});
    }}
}
```

Here we’ve defined the `_start` function that gets called when a proxy wasm module gets first loaded. Since we’re creating the `Plugin` object implementing the `RootContext` trait, so we’ve to implement the `RootContext` trait and its super trait which is `Context`.

According to Proxy Wasm ABI specification:

> *Plugin Context* (also known as *Root Context*) is a context in which operations not related to any specific request or stream are executed.

If you're wondering why we need this, it's because we have a `PluginConfig` type that holds the configuration for our plugin. We need to read this configuration, and the root context allows us to do that.

Let’s implement the `RootContext` `on_configure` method accordingly:

```rust
use log::error;

// ...
// Existing code
// ...

impl RootContext for Plugin {
    fn on_configure(&mut self, _plugin_configuration_size: usize) -> bool {
        if let Some(config_bytes) = self.get_plugin_configuration() {
            if let Ok(config) = serde_json::from_slice::<PluginConfig>(&config_bytes) {
                self.config = config;
            } else {
                error!("failed to parse plugin config: {}", String::from_utf8_lossy(&config_bytes));
            }
        } else {
            error!("no plugin config found");
        }
        true
    }

    fn create_http_context(&self, _context_id: u32) -> Option<Box<dyn HttpContext>> {
        Some(Box::new(Plugin {
            _context_id,
            config: self.config.clone(),
            api_event: Default::default(),
        }))
    }

    fn get_type(&self) -> Option<ContextType> {
        Some(ContextType::HttpContext)
    }
}

// ...
// Existing code
// ...
```

Here, we've implemented the `on_configure` method to read the user-provided configuration when the module starts, allowing us to use it later. We need to unmarshal the provided JSON config into our `PluginConfig` type, so we're using the [serde-rs JSON](https://github.com/serde-rs/json) library.

**Explain** `create_http_context` **and** `get_type` **methods**

So first let’s add that dependency:

```bash
cargo add serde-json
```

and derive the required crates on `PluginConfig` type as follows, otherwise the code would not compile:

```rust
use serde::Deserialize;

#[derive(Deserialize, Clone, Default)]
struct PluginConfig {
    ...
    ...
}
```

So far, so good. Now, let's dive into the interesting part: observing API calls. To achieve this, we'll implement the `HttpContext` trait to build our filter that operates at the `HTTP` layer, aka the Application or L7 layer.

Edit the `src/lib.rs` file as follows:

```rust
use proxy_wasm::types::{Action, ContextType, LogLevel};

// ...
// Exisiting code
// ...

impl HttpContext for Plugin {
    fn on_http_request_headers(&mut self, _num_headers: usize, _end_of_stream: bool) -> Action {
        let (src_ip, src_port) = get_url_and_port(
            String::from_utf8(
                self.get_property(vec!["source", "address"])
                    .unwrap_or_default(),
            )
            .unwrap_or_default(),
        );

        let req_headers = self.get_http_request_headers();
        let mut headers: HashMap<String, String> = HashMap::with_capacity(req_headers.len());
        for header in req_headers {
            // Don't include Envoy's pseudo headers
            // https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#id13
            if !header.0.starts_with("x-envoy") {
                headers.insert(header.0, header.1);
            }
        }

        self.api_event.metadata.timestamp = self
            .get_current_time()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();
        self.api_event.metadata.context_id = self._context_id;
        self.api_event.request.headers = headers;

        let protocol = String::from_utf8(
            self.get_property(vec!["request", "protocol"])
                .unwrap_or_default(),
        )
        .unwrap_or_default();
        self.api_event.protocol = protocol;

        self.api_event.source.ip = src_ip;
        self.api_event.source.port = src_port;
        self.api_event.source.name = String::from_utf8(
            self.get_property(vec!["node", "metadata", "NAME"])
                .unwrap_or_default(),
        )
        .unwrap_or_default();
        self.api_event.source.namespace = String::from_utf8(
            self.get_property(vec!["node", "metadata", "NAMESPACE"])
                .unwrap_or_default(),
        )
        .unwrap_or_default();

        Action::Continue
    }

    fn on_http_request_body(&mut self, _body_size: usize, _end_of_stream: bool) -> Action {
        let body = String::from_utf8(
            self.get_http_request_body(0, _body_size)
                .unwrap_or_default(),
        )
        .unwrap_or_default();
    
        if !body.is_empty() && body.len() <= MAX_BODY_SIZE {
            self.api_event.request.body = body;
        }
        Action::Continue
    }
}

fn get_url_and_port(address: String) -> (String, u16) {
    let parts: Vec<&str> = address.split(':').collect();

    let mut url = "".to_string();
    let mut port = 0;

    if parts.len() == 2 {
        url = parts[0].parse().unwrap_or_default();
        port = parts[1].parse().unwrap_or_default();
    } else {
        error!("invalid address");
    }

    (url, port)
}
```

`HTTPContext` trait provides several methods to deal with different stages of HTTP traffic.

We’ve added our business logic to handle `HTTP Request` by using the provided trait methods. Let’s understand what these trait methods are:

* The `on_http_request_headers` method is invoked as soon as the HTTP request headers are received from the downstream client.
    
* The `on_http_request_body` method is invoked for each chunk of the HTTP request body, even if the processing is temporarily paused.
    

Our goal is to passively observe the traffic without interfering with the API calls. To achieve this, we utilize the `Action::Continue` action within our implementations. This ensures that the request processing continues uninterrupted.

To gain insights into the various aspects of the HTTP request/response, we rely on the attributes provided by Envoy. Envoy attributes can be found [here](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes).

Similarly, let’s add methods to handle `HTTP Response`:

```rust
impl HttpContext for Plugin {
    fn on_http_response_headers(&mut self, _num_headers: usize, _end_of_stream: bool) -> Action {
        let (dest_ip, dest_port) = get_url_and_port(
            String::from_utf8(
                self.get_property(vec!["destination", "address"])
                    .unwrap_or_default(),
            )
            .unwrap_or_default(),
        );

        let res_headers = self.get_http_response_headers();
        let mut headers: HashMap<String, String> = HashMap::with_capacity(res_headers.len());
        for res_header in res_headers {
            // Don't include Envoy's pseudo headers
            // https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#id13
            if !res_header.0.starts_with("x-envoy") {
                headers.insert(res_header.0, res_header.1);
            }
        }

        self.api_event.response.headers = headers;
        self.api_event.destination.ip = dest_ip;
        self.api_event.destination.port = dest_port;

        Action::Continue
    }

    fn on_http_response_body(&mut self, _body_size: usize, _end_of_stream: bool) -> Action {
        let body = String::from_utf8(
            self.get_http_response_body(0, _body_size)
                .unwrap_or_default(),
        )
        .unwrap_or_default();
        if !body.is_empty() && body.len() <= MAX_BODY_SIZE {
            self.api_event.response.body = body;
        }
        Action::Continue
    }
}
```

Here is the complete implementation for the required methods of `HTTPContext` trait:

```rust
impl HttpContext for Plugin {
    fn on_http_request_headers(&mut self, _num_headers: usize, _end_of_stream: bool) -> Action {
        let (src_ip, src_port) = get_url_and_port(
            String::from_utf8(
                self.get_property(vec!["source", "address"])
                    .unwrap_or_default(),
            )
            .unwrap_or_default(),
        );

        let req_headers = self.get_http_request_headers();
        let mut headers: HashMap<String, String> = HashMap::with_capacity(req_headers.len());
        for header in req_headers {
            // Don't include Envoy's pseudo headers
            // https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#id13
            if !header.0.starts_with("x-envoy") {
                headers.insert(header.0, header.1);
            }
        }

        self.api_event.metadata.timestamp = self
            .get_current_time()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();
        self.api_event.metadata.context_id = self._context_id;
        self.api_event.request.headers = headers;

        let protocol = String::from_utf8(
            self.get_property(vec!["request", "protocol"])
                .unwrap_or_default(),
        )
        .unwrap_or_default();
        self.api_event.protocol = protocol;

        self.api_event.source.ip = src_ip;
        self.api_event.source.port = src_port;
        self.api_event.source.name = String::from_utf8(
            self.get_property(vec!["node", "metadata", "NAME"])
                .unwrap_or_default(),
        )
        .unwrap_or_default();
        self.api_event.source.namespace = String::from_utf8(
            self.get_property(vec!["node", "metadata", "NAMESPACE"])
                .unwrap_or_default(),
        )
        .unwrap_or_default();

        Action::Continue
    }

    fn on_http_request_body(&mut self, _body_size: usize, _end_of_stream: bool) -> Action {
        let body = String::from_utf8(
            self.get_http_request_body(0, _body_size)
                .unwrap_or_default(),
        )
        .unwrap_or_default();

        if !body.is_empty() && body.len() <= MAX_BODY_SIZE {
            self.api_event.request.body = body;
        }
        Action::Continue
    }

    fn on_http_response_headers(&mut self, _num_headers: usize, _end_of_stream: bool) -> Action {
        let (dest_ip, dest_port) = get_url_and_port(
            String::from_utf8(
                self.get_property(vec!["destination", "address"])
                    .unwrap_or_default(),
            )
            .unwrap_or_default(),
        );

        let res_headers = self.get_http_response_headers();
        let mut headers: HashMap<String, String> = HashMap::with_capacity(res_headers.len());
        for res_header in res_headers {
            // Don't include Envoy's pseudo headers
            // https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#id13
            if !res_header.0.starts_with("x-envoy") {
                headers.insert(res_header.0, res_header.1);
            }
        }

        self.api_event.response.headers = headers;
        self.api_event.destination.ip = dest_ip;
        self.api_event.destination.port = dest_port;

        Action::Continue
    }

    fn on_http_response_body(&mut self, _body_size: usize, _end_of_stream: bool) -> Action {
        let body = String::from_utf8(
            self.get_http_response_body(0, _body_size)
                .unwrap_or_default(),
        )
        .unwrap_or_default();
        if !body.is_empty() && body.len() <= MAX_BODY_SIZE {
            self.api_event.response.body = body;
        }
        Action::Continue
    }
}
```

We've implemented our business logic to monitor API calls passively. This involves intercepting request and response data, including headers and bodies, to fulfil our specific business requirements.

Let’s give it a try.

* Compile the module:
    

```bash
cargo build --target wasm32-unknown-unknown --release
```

* Create a docker-compose file as follows:
    

```yaml
services:
  envoy:
    image: envoyproxy/envoy:v1.31-latest
    hostname: envoy
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
      - ./target/wasm32-unknown-unknown/release:/etc/envoy/proxy-wasm-plugins
    networks:
      - envoymesh
    ports:
      - "10000:10000"
    environment:
      ENVOY_UID: 0

networks:
  envoymesh: { }
```

* Create the Envoy config `envoy.yaml` file as follows:
    

```yaml
  static_resources:
    listeners:
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 10000
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                codec_type: AUTO
                route_config:
                  name: local_routes
                  virtual_hosts:
                    - name: local_service
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          direct_response:
                            status: 200
                            body:
                              inline_string: "Namaste world!"
                http_filters:
                  - name: envoy.filters.http.wasm
                    typed_config:
                      "@type": type.googleapis.com/udpa.type.v1.TypedStruct
                      type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
                      value:
                        config:
                          name: "httpfilter"
                          root_id: "httpfilter"
                          vm_config:
                            runtime: "envoy.wasm.runtime.v8"
                            code:
                              local:
                                filename: "/etc/envoy/proxy-wasm-plugins/httpfilter.wasm"
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

* Start the services:
    

```bash
$ docker compose up
...
...
envoy | [2024-12-08 14:33:48.085][1][info][main] [source/server/server.cc:958] all clusters initialized. initializing init manager
envoy | [2024-12-08 14:33:48.085][1][info][config] [source/common/listener_manager/listener_manager_impl.cc:930] all dependencies initialized. starting workers
envoy | [2024-12-08 14:33:48.089][1][info][main] [source/server/server.cc:978] starting main dispatch loop
```

* Send a request
    

```bash
$ curl localhost:10000
Namaste world!
```

As you can see, we’re getting the expected response.

I know you’re wondering about the specifics of these API calls.

The thing is, we've already gathered all the necessary information from the Envoy proxy. However, we still need to forward this data to our ingestion service for further processing.

Just for verification, let’s print out the information about API calls on `STDOUT`. Then we’ll write our ingestion service.

```rust
use log::{error, warn};

impl Context for Plugin {
    fn on_done(&mut self) -> bool {
        let event_json = serde_json::to_string(&self.api_event).unwrap_or_default();
        warn!("{}", event_json);
        true
    }
}
```

The `on_done` method is invoked once the host has completed its processing of the context (identified by `context_id`). By returning `true`, we indicate to the host that it can proceed to finalize and delete the context.

Derive the `Serialize` trait to marshal `APIEvent` type so that it can be marshalled into JSON, otherwise the code won’t compile:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Default)]
struct Metadata {
    ...
    ...
}

#[derive(Serialize, Default)]
struct Reqquest {
    ...
    ...
}

#[derive(Serialize, Default)]
struct Ressponse {
    ...
    ...
}

#[derive(Serialize, Default)]
struct Workload {
    ...
    ...
}

#[derive(Serialize, Default)]
struct APIEvent {
    ...
    ...
}
```

Re-compile the module:

```bash
cargo build --target wasm32-unknown-unknown --release
```

Start the services:

```bash
$ docker compose up
...
...
envoy | [2024-12-08 14:56:13.624][1][info][main] [source/server/server.cc:958] all clusters initialized. initializing init manager
envoy | [2024-12-08 14:56:13.624][1][info][config] [source/common/listener_manager/listener_manager_impl.cc:930] all dependencies initialized. starting workers
envoy | [2024-12-08 14:56:13.629][1][info][main] [source/server/server.cc:978] starting main dispatch loop
```

Once again send the request:

```bash
$ curl localhost:10000
Namaste world!
```

If you check the logs of the compose terminal, you will see something similar as follows:

```bash
...
...
envoy-1  | [2024-12-15 06:24:57.672][25][warning][wasm] [source/extensions/common/wasm/context.cc:1198] wasm log httpfilter httpfilter: {"metadata":{"context_id":2,"timestamp":1734243897,"traffic_source_name":"","traffic_source_version":"","node_name":""},"request":{"headers":{":path":"/","x-request-id":"0f1aad54-23bd-4c1d-91ef-53a3f477976b",":method":"GET",":scheme":"http","user-agent":"HTTPie/3.2.4","x-forwarded-proto":"http",":authority":"localhost:10000","accept-encoding":"gzip, deflate","accept":"*/*"},"body":""},"response":{"headers":{":status":"200","content-type":"text/plain","content-length":"14"},"body":"Namaste world!"},"source":{"name":"","namespace":"","ip":"172.19.0.1","port":60806},"destination":{"name":"","namespace":"","ip":"172.19.0.2","port":10000},"protocol":"HTTP/1.1"}
```

This means we’re able to successfully intercept the traffic.

## Ingestion Service

Let's build a basic HTTP web service to receive API call telemetry from our Envoy Wasm plugin. We'll use Go for this example, but you can choose your preferred language.

Let’s initialize a go module for our service:

```bash
go mod init github.com/anurag-rajawat/tutorials/ingestion-service
```

Create a `main.go` file as follows:

```go
package main

import (
	"errors"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
)

func main() {
	log.Println("starting ingestion service")
	args := os.Args
	if len(args) < 2 {
		log.Fatal("no port specified.")
	}
	port := args[1]

	mux := http.NewServeMux()
	mux.HandleFunc("/api/v1/events", apiEventsHandler)

	srv := &http.Server{
		Addr:    fmt.Sprintf(":%v", port),
		Handler: mux,
	}

	log.Printf("ingestion service is listening on port: %v", port)
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatalf("failed to start server, error: %v", err)
	}
}

func apiEventsHandler(writer http.ResponseWriter, request *http.Request) {
	if request.Method != http.MethodPost {
		writer.WriteHeader(http.StatusMethodNotAllowed)
		return
	}

	if request.Body == nil {
		log.Print("body is nil")
		writer.WriteHeader(http.StatusBadRequest)
		return
	}

	apiEvent, err := io.ReadAll(request.Body)
	if err != nil {
		log.Printf("failed to read request body, error: %v", err)
		http.Error(writer, "failed to read request body", http.StatusInternalServerError)
		return
	}

	log.Printf("API Event received\n%v", string(apiEvent))
}
```

We've created a basic HTTP web server that exposes a single POST endpoint (`/api/v1/events`) for our plugin to send API call data. You can customize it as needed.

Now that we have a service ready to receive data, we can configure our plugin to send API call data to this service.

Update the `on_done` method in `src/lib.rs` file of the plugin as follows:

```rust
impl Context for Plugin {
    fn on_done(&mut self) -> bool {
        dispatch_http_call_to_upstream(self);
        true
    }
}


fn dispatch_http_call_to_upstream(obj: &mut Plugin) {
    update_metadata(obj);
    let telemetry_json = serde_json::to_string(&obj.api_event).unwrap_or_default();

    let headers = vec![
        (":method", "POST"),
        (":authority", &obj.config.authority),
        (":path", &obj.config.path),
        ("accept", "*/*"),
        ("Content-Type", "application/json"),
    ];

    let http_call_res = obj.dispatch_http_call(
        &obj.config.upstream_name,
        headers,
        Some(telemetry_json.as_bytes()),
        vec![],
        Duration::from_secs(1),
    );

    if http_call_res.is_err() {
        error!(
            "failed to dispatch HTTP call, to '{}' status: {http_call_res:#?}",
            &obj.config.upstream_name,
        );
    }
}

fn update_metadata(obj: &mut Plugin) {
    obj.api_event.metadata.node_name = String::from_utf8(
        obj.get_property(vec!["node", "metadata", "NODE_NAME"])
            .unwrap_or_default(),
    )
    .unwrap_or_default();
    obj.api_event.metadata.traffic_source_name = "Envoy".to_string();
}
```

Here, we send the collected data to our service after processing is complete, allowing the host to finalize and delete the context.

Let's run both and check if they work as expected.

* Open a new terminal and start the ingestion service:
    

```bash
$ go run main.go 8888
2024/12/15 10:31:34 starting ingestion service
2024/12/15 10:31:34 ingestion service is listening on port: 8888
```

* To allow the plugin to send data, update the service address within the `envoy.yaml` file.
    

```yaml
static_resources:
  listeners:
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
      - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              codec_type: AUTO
              route_config:
                name: local_routes
                virtual_hosts:
                  - name: local_service
                    domains:
                      - "*"
                    routes:
                      - match:
                          prefix: "/"
                        direct_response:
                          status: 200
                          body:
                            inline_string: "Namaste world!"
              http_filters:
                - name: envoy.filters.http.wasm
                  typed_config:
                    "@type": type.googleapis.com/udpa.type.v1.TypedStruct
                    type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
                    value:
                      config:
                        name: "httpfilter"
                        root_id: "httpfilter"
                        configuration:
                          "@type": "type.googleapis.com/google.protobuf.StringValue"
                          value: |
                            {
                                "upstream_name": "ingestion-service",
                                "authority": "ingestion-service",
                                "path": "/api/v1/events"
                            }
                        vm_config:
                          runtime: "envoy.wasm.runtime.v8"
                          code:
                            local:
                              filename: "/etc/envoy/proxy-wasm-plugins/httpfilter.wasm"
                - name: envoy.filters.http.router
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
    - name: "ingestion-service"
      connect_timeout: 1s
      type: STRICT_DNS
      load_assignment:
        cluster_name: "ingestion-service"
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.29.251 # Replace this with your machine IP if you're running it locally via docker
                      port_value: 8888 # Replace this with your ingestion service port
```

* Re-compile the plugin module:
    

```bash
cargo build --target wasm32-unknown-unknown --release
```

* Start the Envoy proxy:
    

```bash
$ docker compose up
```

* Send a request to Envoy proxy:
    

```bash
$ curl localhost:10000
Namaste world!
```

* Check the logs of the ingestion service:
    

```bash
...
...
2024/12/15 10:37:44 API Event received
{"metadata":{"context_id":2,"timestamp":1734239264,"traffic_source_name":"Envoy","traffic_source_version":"","node_name":""},"request":{"headers":{":path":"/",":method":"GET",":scheme":"http","user-agent":"curl/8.7.1","x-forwarded-proto":"http",":authority":"localhost:10000","accept":"*/*","x-request-id":"1f01e5de-4fa2-422b-b0e0-094342147124"},"body":""},"response":{"headers":{":status":"200","content-type":"text/plain","content-length":"14"},"body":"Namaste world!"},"source":{"name":"","namespace":"","ip":"172.19.0.1","port":62442},"destination":{"name":"","namespace":"","ip":"172.19.0.2","port":10000},"protocol":"HTTP/1.1"}
```

As you can see, we're receiving all the API call details such as source and destination IPs and ports, request and response body, method, and the protocol used for this API call. There is much more you can do with this information.

# Real World use

I know you’re wondering about its real-world use cases.

In the real world, many API observability and security solutions use these plugins or modules. They help understand the user environment by showing which service is calling another service, on which endpoint, and with which method, among other details.

You can even extend service meshes like Istio and Consul, which use Envoy proxy.

That’s it for now. You can find the complete code [**here**.](https://github.com/anurag-rajawat/tutorials)

Please feel free to comment or criticize :)

# Summary

> This blog post describes how to create a WebAssembly (WASM) plugin for Envoy Proxy to passively monitor incoming and outgoing RESTful and gRPC API calls. The plugin intercepts the request and response data, including headers and bodies, without interfering with the actual calls.

# References

* [Envoy proxy attributes](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes)
    
* [Envoy Wasm Plugin](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/wasm/v3/wasm.proto)
    
* [Proxy Wasm Rust SDK](https://github.com/proxy-wasm/proxy-wasm-rust-sdk)