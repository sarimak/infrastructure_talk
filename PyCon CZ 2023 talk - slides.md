---
title: Let's not bother our researchers by the infrastructure
author: Jiří Bajer
format: revealjs
theme: league
highlight-style: base16-tomorrow
defaultTiming: 140
width: 1920
height: 1080
---

<style>
.reveal pre.code {max-width: 60%;  margin-left: auto; margin-right: auto; max-height: 80%}
.reveal pre.code-wrapper {max-width: 60%;  margin-left: auto; margin-right: auto; max-height: 80%}
</style>

## Let's not bother our researchers by the infrastructure

<!-- .slide: data-background-image="pyconcz23.svg" data-background-size=contain data-background-opacity=0.2 -->

Jiří Bajer

[rossum.ai](https://rossum.ai)

---

## 🥺️ MVP Code

- Infrastructure and ML algorithms mixed

- Copy-pasted but slightly different boilerplate

- Use cases buried deep in implementation details

note: Researchers have to mess with infrastructure, integration tests for everything. Hard to troubleshoot. Hard to onboard.

---

## 🪠 Cleanup

- Split the infrastructure from domain logic

- Unify the boilerplate

- Parametrize and inject the infrastructure

- Remove decisions from top level of abstraction

note: What kinds of services do we have, what do they have in common and where is the domain--infrastructure boundary?

---

## 🔎 Anatomy of ML Service

- Single process (CPU-bound)

- Infinite loop until shutdown

- Poll I/O source for a message (dataclass)

- Process the message (callback per message type)

- Infrastructure clients (managed lifecycle)

- Acknowledge/send a response

note: Asynchronous messaging consumer or synchronous web service. Actor model: Multi-core and more RAM for the ML algorithm to finish sooner, more replicas for processing more at once. All state backed by persistent storage, no cross-service shared state.

---

## 💌️ Messaging Service

```Python {data-line-numbers}
while not shutdown_requested:
    payload, metadata = consumer.poll_topic(timeout)
    message = deserialize(payload, metadata, AVAILABLE_MESSAGES)

    if message:
        with consumer.autoacknowledge():
            process(message)
```

note: Works for Kafka, Pulsar and RabbitMQ. Message is a dataclass, its type is taken from metadata. De-/eserializable from/to JSON.

---

## 🌍️ Web Service

```Python {data-line-numbers}
while not shutdown_requested:
    payload, headers = http_server.poll_for_request(timeout)
    metadata = to_metadata(headers)
    message = deserialize(payload, metadata, AVAILABLE_MESSAGES)

    if message:
        response_message = process(message)
        http_server.send_response(
            payload=response_message.json_payload,
            headers=to_headers(response_message.metadata),
            code=response_message.code,
        )
```

note: Headers vs. metadata, response message and code. Pre-forked Gunicorn but no shared state. RPC-style/POST JSON.

---

## 🤲 Service Types Compared

```Python {data-line-numbers="2,6"}
while not shutdown_requested:
    payload, metadata = consumer.consume_from_topic(timeout)
    message = deserialize(payload, metadata, AVAILABLE_MESSAGES)

    if message:
        with consumer.autoacknowledge():
            process(message)
```

```Python {data-line-numbers="2-3,8-12"}
while not shutdown_requested:
    payload, headers = http_server.poll_for_request(timeout)
    metadata = to_metadata(headers)
    message = deserialize(payload, metadata, AVAILABLE_MESSAGES)

    if message:
        response_message = process(message)
        http_server.send_response(
            payload=response_message.json_payload,
            headers=to_headers(response_message.metadata),
            code=response_message.code,
        )
```

note: Similar/easily replaceable. Domain-agnostic/logic encapsulated in processing. Processing done by callbacks for given message type.

---

## 🛠️ Concrete Service

```Python {data-line-numbers}
@dataclasses.dataclass
class PredictionRequest(Message):  # Command or Event
    MESSAGE_TYPE = "prediction.request"

    data_id: str
```

```Python {data-line-numbers}
class Predictor(MessagingService):
    def __init__(self, config):  # No I/O yet
        super().__init__(config)
        self.message_handlers = {
            PredictionRequest: self.handle_prediction_request
        }  # All use cases provided by the service

        self.model = OurDeepLearningPredictor(...)
        self.dao = PredictionDAO(self.db_client)
        self.dao.register_tables()
```

note: Parent classes are domain agnostic and hide the repeating bits.

---

## ⚡️ Infrastructure Clients

```Python {data-line-numbers}
class Predictor(MessagingService):
    ...

    def register_resources(self):
        self.queue = self.register_client(KafkaClient)
        self.consumer = self.queue.consumer(topic="")
        self.producer = self.queue.producer(topic="")

        self.s3_client = self.register_client(S3Client)
        self.http_client = self.register_client(HTTPClient)

        self.metrics = self.register_client(PrometheusClient)
        self.prediction_duration = self.metrics.histogram(name="")

        self.db_client = self.register_client(SQLAlchemyClient)
```

note: Registered and configured. Connectivity checks upon start, graceful shutdown.  Metric exposition and liveness/readiness endpoints in background thread.

---

## ⏳️ Message Handler

```Python {data-line-numbers}
class Predictor(MessagingService):
    ...

    def handle_prediction_request(self, message: PredictionRequest):
        heavy_data = self.s3_client.get("bucket", message.data_id)
        foreign_data = self.http_client.get(self.config.external_url)

        with self.metrics_client.elapsed_time(
            metric=self.prediction_duration
        ):  # Duration of whole handler is measured automatically
            prediction = self.model.predict(
                heavy_data, foreign_data
            )
            self.dao.save_prediction(prediction)
            self.producer.push(PredictionResult(**prediction))
```

note: Handler parametrizes the infrastructure clients by domain-specific bits. Clients hide the domain-agnostic code for retries and giveups, timeouts, connectivity checks, transaction rollbacks, schema migrations upon service start. Transaction boundaries and updating of metrics are domain-specific concerns. Cyclomatic complexity is one. All decisions are hidden in self.model (with no I/O). Very similar to ports and adapters architecture.

---

## 🗃️ Database Access

```Python {data-line-numbers}
class PredictionDAO:  # Simple domain, many rows
    def register_tables(self):
        self.predictions = sqlalchemy.table(
            "predictions",
            self.db_client.metadata,
            sqlalchemy.Column(...),
        )

    def save_prediction(self, prediction):
        with self.db_client.engine.begin() as connection:
            connection.execute(
                sqlalchemy.insert(self.predictions).values(
                    **prediction
                )
            )
```

note: Simple replacement by noSQL. Encapsulates backward compatibility tricks.

---

## 🌫️ Hidden Infrastructure

- ♻️ **Lifecycle Management**

  Liveness/readiness endpoints, before ready, tracing

- 🎚️ **Configuration**

  Dict or environment/config maps, type coercion

- 🪵 **Logging**

  Service/client lifecycle, client config, processed message

- 📉 **Metrics**

  Central registration, exposition, response times, multi-process

- 🩺 **Testing**

  IPython CLI, fixture with mocked queue/object storage

note: Web server in background thread, automatic logging/metrics. DB migrations or model initialization in before_ready hook (alive but not ready).

---

## 🚧️ Domain boundary

💼️ **Domain-specific code**

- Concrete message dataclasses
- Concrete services
- Message handlers
- Registration and use of infrastructure client

🛠️ **Domain-agnostic boilerplate** <!-- .element: class="fragment" data-fragment-index="1" -->

- `Service`, `MessagingService`, `WebService` <!-- .element: class="fragment" data-fragment-index="1" -->
- Message and its deserialization <!-- .element: class="fragment" data-fragment-index="1" -->
- Message handler selection and calling <!-- .element: class="fragment" data-fragment-index="1" -->
- Infrastructure clients <!-- .element: class="fragment" data-fragment-index="1" -->

note: Engineers: reusable boilerplate, researchers: domain logic.

---

## 💪️ Framework wrestling

- 🌍 **WSGIServer** & **ThreadingMixIn**

  Shutdown hook, request size limit, Werkzeug

- 🦄 **Gunicorn**

  Import & call + hooks, multi-process metrics

- 🔥 **Prometheus client**

  HTTP server in child thread, copy of exposition

- 🦷 **Alembic**

  Import & call from before_ready

- 😵‍💫 **gRPC**

  Thread-pool, generated code, stub and servicer

note: Need to control client's lifecycle (configure, connect and disconnect) and the infinite loop. Requires bending the frameworks to behave like libraries (imported and called). Clients cannot make I/O in their constructor.

--

## HTTP Server in a Thread

```Python {data-line-numbers}
class ThreadedWSGIServer(
    socketserver.ThreadingMixIn,
    wsgiref.simple_server.WSGIServer,  # Request max 64KB
):
    daemon_threads = True  # New thread for each request
```

```Python {data-line-numbers}
class WSGIApplication:
    def __call__(self, environ, start_response):
        request = werkzeug.wrappers.Request(environ)
        response_message, http_code = self.process(...)
        response = werkzeug.wrappers.Response(
            response_message.get_payload(),
            http_code,
            response_message.get_headers(),
        )
        return response(environ, start_response)
```

```Python {data-line-numbers}
class HTTPServer:
    def start(self):
        self.wsgi_server = ThreadedWSGIServer(
            (host, port),
            wsgiref.simple_server.WSGIRequestHandler,
            bind_and_activate=True,
        )
        self.wsgi_server.set_app(WSGIApplication(...))
        self.wsgi_server.timeout = ...
        self.thread = threading.Thread(
            target=self.wsgi_server.serve_forever,
            daemon=True,
        )
        self.thread.start()

    def stop(self):
        self.wsgi_server.server_close()
        self.thread.join()
```

note: Liveness/readiness probes, Prometheus metrics

--

## Web Service (Gunicorn)

```Python {data-line-numbers}
class GunicornApplication(gunicorn.app.base.BaseApplication):
    def __call__(self, environ, start_response):
        return self.wsgi_application(environ, start_response)

    def load_config(self):
        self.cfg.set("bind", "0.0.0.0:80")
        self.cfg.set(
            "post_worker_init",
            lambda w: w.app.callable.web_service.start(),
        )
        self.cfg.set(
            "worker_exit",
            lambda _, w: w.app.callable.web_service.stop()
        )  # + shut down the service

    def load(self):
        self.wsgi_application = WSGIApplication(
            lambda: os.kill(os.getppid(), signal.SIGTERM),
        )
        return self
```

```Python {data-line-numbers}
class WebService(Service):
    def run_forever(self):
        GunicornApplication(self).run()
```

note: Gunicorn overwrites the signal handlers

--

## Prometheus client

```Python {data-line-numbers}
self.registry = prometheus_client.registry.CollectorRegistry(
    auto_describe=True
)  # New instance in Gunicorn's post_worker_init
prometheus_client.exposition.generate_latest(self.registry)
content_type = prometheus_client.exposition.CONTENT_TYPE_LATEST
# Remove PID of Gunicorn worker at child_exit
```

note: Web services must use disk-based multi-process of the Prometheus client. Each worker needs own CollectorRegistry created at post_worker_init and its PID needs to be removed at child_exit.

--

## Alembic Migrations on start

```Python {data-line-numbers}
def start():
    self.alembic_config = alembic.config.Config()
    self.alembic_config.set_main_option("script_location", ...)
    self.script_directory = alembic.script.ScriptDirectory.from_config(
        self.alembic_config
    )
    self.environment_context = alembic.runtime.environment.EnvironmentContext(
        config=self.alembic_config,
        script=self.script_directory,
    )

def upgrade(self, db_client, target_revision):
    def _do_upgrade(revision, _context):
        return self.script_directory._upgrade_revs(
            target_revision, revision
        )

    with db_client.engine.begin() as connection:
        self.environment_context.configure(
            connection=connection,
            target_metadata=db_client.metadata,
            fn=_do_upgrade,
        )
        self.environment_context.run_migrations()

def get_current(self, db_client):
    # DB query to alembic_version.c.version_num

def get_head(self):
    return self.script_directory.get_current_head()
```

note: And exclusive lock (PostgreSQL: LOCK TABLE alembic_version IN ACCESS EXCLUSIVE MODE).

--

## Alembic CLI

env.py:

```Python {data-line-numbers}
if alembic.context.is_offline_mode():
    alembic.context.configure(
        url=self.db_client.connection_string,
        target_metadata=self.db_client.metadata,
        literal_binds=True,
        dialect_opts={...},
    )
    with alembic.context.begin_transaction():
        alembic.context.run_migrations()
else:
    with self.db_client.engine.begin() as connection:
        alembic.context.configure(
            connection=connection,
            target_metadata=self.db_client.metadata,
        )
        alembic.context.run_migrations()
```

note: To support `alembic revision`, `alembic upgrade` from CLI etc.

--

## gRPC Client

- Create an insecure channel and wait for `grpc.channel_ready_future`
- Create own `grpc.<method_type>MultiCallable` stub using the channel with string de-/serializers for generated protobuf request/response class and method type
- Call the `stub.with_call` (or `stub` for streaming) with a serialized request and deserialize the response
- Cancel the `channel_ready_future` and close the channel upon stop

note: Generated boilerplate from protobuf definitions is partially unused.

--

## gRPC Server

- Run part of code from `grpc._server._Server.start` in own thread
- Delegate to grpc.server with a thread-pool with 1 worker and our service handler
- Unregister atexit hook of concurrent futures, so stuck thread won't block shutdown
- Run part of code from `grpc._server._start` after add_insecure_port
- Implement service handler compatible with `grpc.GenericRpcHandler` which gets method and invocation metadata from the `grpc.HandlerCallDetails` and submits a newly created instance of `grpc.<method type>_rpc_method_handler` to the single-thread pool, with string de-/serializers for generated protobuf request/response class and our own behavior (handler)
- Run part of `grpc._server._serve` which polls the completion_queue in infinite loop and waits until the thread pool shuts down upon shutdown
- The main loop of the service runs a HTTP server with liveness/readiness probes and metrics

note: Again, skips part of the generated boilerplate. Overall, gRPC is not an easy to integrate but it is possible. I would avoid gRPC in general if I could.

---

## 🙇️ Thank you

[linkedin.com/jiribajer](https://linkedin.com/jiribajer)

Slides: [github.com/sarimak/infrastructure_talk](https://github.com/sarimak/infrastructure_talk)

---

## 📖 Further reading

[sandimetz.com](https://sandimetz.com)

[cosmicpython.com](https://cosmicpython.com)

[Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

[Boundaries](https://www.destroyallsoftware.com/talks/boundaries)

[12 Factor App](https://12factor.net/)
