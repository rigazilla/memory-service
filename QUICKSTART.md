# Quickstart: Memory Service + OneRing demo

This guide starts the Memory Service stack with the demo configuration from
[`compose.override.yaml`](compose.override.yaml) and connects OneRing to it.

## Prerequisites

- Docker with Docker Compose v2
- An OpenAI API key (used for embeddings and, optionally, the cognition processor)
- Access to the OneRing repository on github.ibm.com and to IBM Artifactory (for the build)
- A JDK and Maven (`mvn`) for building OneRing

## 1. Start the Memory Service

From the root of this repository:

```bash
export OPENAI_API_KEY=sk-...
docker compose --profile cognition up -d --build
```

`docker compose` automatically merges [`compose.yaml`](compose.yaml) with
[`compose.override.yaml`](compose.override.yaml). The override applies the demo setup:

| Setting | Value |
| --- | --- |
| `MEMORY_SERVICE_API_KEYS_ONERING` | `one_ring_api_key` (override with the env var of the same name) |
| `MEMORY_SERVICE_TRUSTED_USER_ID_CLIENTS` | `onering,cognition_processor,developer_frontend` |
| `MEMORY_SERVICE_EMBEDDING_OPENAI_API_KEY` | `${OPENAI_API_KEY}` |
| `COGNITION_OPENAI_MODEL_NAME` | `gpt-4o-mini` |
| `MEMORY_SERVICE_RATE_LIMIT_MODE` | `off` |

The API key registers a client with the id `onering`. Because that client is listed in
`MEMORY_SERVICE_TRUSTED_USER_ID_CLIENTS`, OneRing can act on behalf of a user by passing the user id
itself, without an OIDC token.

Omit `--profile cognition` if you don't need the cognition processor.

Wait until `memory-service` reports `healthy`:

```bash
docker compose ps memory-service
```

### Endpoints

| Service | URL |
| --- | --- |
| Memory Service (REST + gRPC) | http://localhost:8082 |
| Developer frontend | http://localhost:8082/developer/ |
| Keycloak | http://localhost:8081 |
| Cognition processor | http://localhost:8090 |
| Grafana | http://localhost:3001 |

## 2. Build and start OneRing

### Clone

```bash
git clone --branch memory-service-plugin-demo --single-branch \
  https://github.ibm.com/project-prince/OneRing.git
cd OneRing
```

### Configure Artifactory

The OneRing build resolves dependencies from Artifactory. Export your credentials and make sure a
`settings.xml` that uses them is available in the OneRing directory:

```bash
export W3_USERNAME=<your w3 username>
export ARTIFACTORY_ID_TOKEN=<your Artifactory identity token>
```

<!-- TODO: document where to get the settings.xml (or include its content) and how it uses
     W3_USERNAME / ARTIFACTORY_ID_TOKEN. -->

### Build

```bash
mvn -s settings.xml clean verify
```

To skip the tests, use:

```bash
mvn -s settings.xml clean install -DskipTests
```

### Start the OneRing server

Run this from the OneRing directory. It points OneRing at the Memory Service started in step 1:

```bash
java -jar onering-quarkus-app/target/quarkus-app/quarkus-run.jar serve \
  --agent-dir my-agent \
  --store-type memory-service \
  --store-url http://localhost:8082 \
  --store-api-key one_ring_api_key \
  --state-dir .onering
```

| Option | Value |
| --- | --- |
| `--agent-dir` | Directory containing the agent definition (`my-agent`) |
| `--store-type` | `memory-service` stores conversations in the Memory Service |
| `--store-url` | `http://localhost:8082` |
| `--store-api-key` | `one_ring_api_key`, the key configured in `compose.override.yaml` |
| `--state-dir` | Local OneRing state directory (`.onering`) |

### Start the chat UI

In a second terminal, in the OneRing directory:

```bash
java -jar onering-quarkus-app/target/quarkus-app/quarkus-run.jar dev \
  --agent-dir my-agent \
  --no-start
```

`--no-start` makes the chat UI attach to the server you already started instead of launching its own.

<!-- TODO: add the chat UI URL printed by the `dev` command. -->

## 3. Verify

1. Use the OneRing chat UI to send a message.
2. Open the Memory Service developer frontend:
   - Conversations are listed at http://localhost:8082/developer/conversations.
   - Memories are visible at http://localhost:8082/developer/memories.

## Stop and clean up

1. Exit the OneRing chat UI by pressing `Ctrl-D` twice.
2. Stop the OneRing server with `Ctrl-C`.
3. Stop the Memory Service stack from this repository's root:

```bash
docker compose --profile cognition down      # keep data volumes
docker compose --profile cognition down -v   # also remove data
```

## Troubleshooting

- **401 from the Memory Service**: the API key OneRing sends doesn't match
  `MEMORY_SERVICE_API_KEYS_ONERING`. If you set that variable, use the same value in OneRing.
- **Embedding or cognition errors**: `OPENAI_API_KEY` wasn't exported in the shell that ran
  `docker compose up`. Export it and recreate the containers.
- **Port conflicts**: the stack uses 8080, 8081, 8082, 8090, 5432, 6333, 6379, 9000 and 3001. Stop
  anything already bound to them.
