---
title: Model Registry
sidebar_position: 4
---

# Model Registry

:::caution
This module is still in active development and should be considered a prerelease version.
:::

Cogment Model Registy is designed to store, version and make available AI models to be used by Cogment actors.

The Model Registry stores model versions in two ways:

-   **Transient** (non-archived) model versions can be used to broadcast an updated version of a model to its users, e.g. during training. Transient model versions are stored in memory and can be evicted, in particular once the memory limit is reached.
-   **Archived** model versions are stored on the filesystem, they are slower to create and to retrieve and should be used for long-term storage of specific versions, e.g. for validation or deployment purposes.

## Command line

The Model Registry is simply called this way

```bash
$ cogment services model_registry --port=9000 --archive_dir=./models/
```

## Configuration

The Model Registry configuration can be specified either through the command line or environment variables.

### `port`

The TCP port where to serve the [The Model Registry API](../grpc.md#model-registry-api).

Can be specified as:

-   a command line option, e.g. `--port=12000`,
-   an environment variable, e.g. `COGMENT_MODEL_REGISTRY_PORT=12000`,
-   its default value is 9002.

### `grpc_reflection`

Whether or not to enable [gRPC reflection](https://github.com/grpc/grpc/blob/master/doc/server-reflection.md) on the served endpoints.

Can be specified as:

-   a command line flag, e.g. `--grpc_reflection`,
-   an environment variable, e.g. `COGMENT_MODEL_REGISTRY_GRPC_REFLECTION=1`,
-   by default, it is disabled.

### `archive_dir`

Path to the directory to store archived model versions and model metadata.

Can be specified as:

-   a command line option, e.g. `--archive_dir=./path/to/models/`,
-   an environment variable, e.g. `COGMENT_MODEL_REGISTRY_ARCHIVE_DIR=./path/to/models/`,
-   its default value is `.cogment/model_registry`

### `cache_max_items`

This defines the maximum number of model versions that can be stored in the transient cache.

Can be specified as:

-   a command line option, e.g. `--cache_max_items=500`,
-   an environment variable, e.g. `COGMENT_MODEL_REGISTRY_VERSION_CACHE_MAX_ITEMS=500`,
-   its default value is 100.

### `sent_version_chunk_size`

The maximum size for model version data chunk sent by the server. It is defined in bytes.

Can be specified as:

-   a command line option, e.g. `--sent_version_chunk_size=10000000`,
-   an environment variable, e.g. `COGMENT_MODEL_REGISTRY_SENT_MODEL_VERSION_DATA_CHUNK_SIZE=10000000`,
-   its default value is 5242880 Bytes, i.e. 5MB.

### `log_level`

Set to define the minimum level for logging. Possible values are: `off`, `error`, `warning`, `info`, `debug`, `trace`. Note however that all trace and most debug level logs will only output if running the debug compiled version of the Orchestrator.

Can be specified as:

-   a command line option, e.g. `--log_level=debug`,
-   an environment variable, e.g. `COGMENT_LOG_LEVEL=5`,
-   default value is info.

### `log_file`

Base file name for daily log output. The name will be suffixed with the date and a new file will be made every day. If not provided the logs go to stdout.

Can be specified as:

-   a command line option, e.g. `--log_file=./path/to/cogment.log`,
-   an environment variable, e.g. `COGMENT_LOG_FILE=./path/to/cogment.log`,
-   default value is info.

## API usage examples

::: tip

The following examples require `COGMENT_MODEL_REGISTRY_GRPC_REFLECTION` to be enabled as well as [grpcurl](https://github.com/fullstorydev/grpcurl)\_

:::

### Create or update a model - `cogmentAPI.ModelRegistrySP/CreateOrUpdateModel( .cogmentAPI.CreateOrUpdateModelRequest ) returns ( .cogmentAPI.CreateOrUpdateModelReply );`

```console
$ echo "{\"model_info\":{\"model_id\":\"my_model\",\"user_data\":{\"type\":\"my_model_type\"}}}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/CreateOrUpdateModel
{

}
```

### Delete a model - `cogmentAPI.ModelRegistrySP/DeleteModel( .cogmentAPI.DeleteModelRequest ) returns ( .cogmentAPI.DeleteModelReply );`

```console
$ echo "{\"model_id\":\"my_model\"}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/DeleteModel
{

}
```

### Retrieve models - `cogmentAPI.ModelRegistrySP/RetrieveModels( .cogmentAPI.RetrieveModelsRequest ) returns ( .cogmentAPI.RetrieveModelsReply );`

_These examples require `COGMENT_MODEL_REGISTRY_GRPC_REFLECTION` to be enabled as well as [grpcurl](https://github.com/fullstorydev/grpcurl)_

#### List the models

```console
$ echo "{}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/RetrieveModels
{
  "modelInfos": [
    {
      "modelId": "my_model",
      "userData": {
        "type": "my_model_type"
      }
    },
    {
      "modelId": "my_other_model",
      "userData": {
        "type": "my_model_type"
      }
    }
  ],
  "nextModelHandle": "2"
}
```

#### Retrieve specific model(s)

```console
$ echo "{\"model_ids\":[\"my_other_model\"]}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/RetrieveModels
{
  "modelInfos": [
    {
      "modelId": "my_other_model",
      "userData": {
        "type": "my_model_type"
      }
    }
  ],
  "nextModelHandle": "1"
}
```

### Create a model version - `cogmentAPI.ModelRegistrySP/CreateVersion( stream .cogmentAPI.CreateVersionRequestChunk ) returns ( .cogmentAPI.CreateVersionReply );`

```console
$ echo "{\"header\":{\"version_info\":{
    \"model_id\":\"my_model\",\
    \"archived\":true,\
    \"data_size\":$(printf chunk_1chunk_2 | wc -c)\
  }}}\
  {\"body\":{\
    \"data_chunk\":\"$(printf chunk_1 | base64)\"\
  }}\
  {\"body\":{\
    \"data_chunk\":\"$(printf chunk_2 | base64)\"\
  }}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/CreateVersion
{
  "versionInfo": {
    "modelId": "my_model",
    "versionNumber": 2,
    "creationTimestamp": "907957639",
    "archived": true,
    "dataHash": "jY0g3VkUK62ILPr2JuaW5g7uQi0EcJVZJu8IYp3yfhI=",
    "dataSize": "14"
  }
}
```

### Retrieve model versions infos - `cogmentAPI.ModelRegistrySP/RetrieveVersionInfos ( .cogmentAPI.RetrieveVersionInfosRequest ) returns ( .cogmentAPI.RetrieveVersionInfosReply );`

#### List the versions of a model

```console
$ echo "{\"model_id\":\"my_model\"}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/RetrieveVersionInfos
{
  "versionInfos": [
    {
      "modelId": "my_model",
      "versionNumber": 1,
      "creationTimestamp": "1633119005107454620",
      "archived": true,
      "dataHash": "jY0g3VkUK62ILPr2JuaW5g7uQi0EcJVZJu8IYp3yfhI=",
      "dataSize": "14"
    },
    {
      "modelId": "my_model",
      "versionNumber": 2,
      "creationTimestamp": "1633119625907957639",
      "archived": true,
      "dataHash": "jY0g3VkUK62ILPr2JuaW5g7uQi0EcJVZJu8IYp3yfhI=",
      "dataSize": "14"
    }
  ],
  "nextVersionHandle": "3"
}
```

#### Retrieve specific versions of a model

```console
$ echo "{\"model_id\":\"my_model\", \"version_numbers\":[1]}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/RetrieveVersionInfos
{
  "versionInfos": [
    {
      "modelId": "my_model",
      "versionNumber": 1,
      "creationTimestamp": "1633119005107454620",
      "archived": true,
      "dataHash": "jY0g3VkUK62ILPr2JuaW5g7uQi0EcJVZJu8IYp3yfhI=",
      "dataSize": "14"
    }
  ],
  "nextVersionHandle": "2"
}
```

#### Retrieve the n-th to last version of a model

```console
$ echo "{\"model_id\":\"my_model\", \"version_numbers\":[-2]}" | grpcurl -plaintext -d @ localhost:9000 cogmentAPI.ModelRegistrySP/RetrieveVersionInfos
{
  "versionInfos": [
    {
      "modelId": "my_model",
      "versionNumber": 1,
      "creationTimestamp": "1633119005107454620",
      "archived": true,
      "dataHash": "jY0g3VkUK62ILPr2JuaW5g7uQi0EcJVZJu8IYp3yfhI=",
      "dataSize": "14"
    }
  ],
  "nextVersionHandle": "2"
}
```

### Retrieve given version data - `cogmentAPI.ModelRegistrySP/RetrieveVersionData ( .cogmentAPI.RetrieveVersionDataRequest ) returns ( stream .cogmentAPI.RetrieveVersionDataReplyChunk );`

```console
$ echo "{\"model_id\":\"my_model\", \"version_number\":1}" | grpcurl -plaintext -d @ localhost:9000 cogment.ModelRegistrySP/RetrieveVersionData
{
  "dataChunk": "Y2h1bmtfMWNodW5rXzI="
}
```

To retrieve the n-th to last version, use `version_number:-n` (e.g. `-1` for the latest, `-2` for the 2nd to last).
