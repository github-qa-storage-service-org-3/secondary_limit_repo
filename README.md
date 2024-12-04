# R-Cloud Module

This project contains a Python skeleton implementation of an R-Cloud module for backup/restore.

This _README_ consists of two main chapters - _Tutorials_ and _General Concepts_:

- The purpose of the _Tutorials_ chapter is to help the developer with the initial steps - build procedure, adding a new
  module as a source to _HYCU R-Cloud_, defining the module's capabilities and authentication settings, and syncing the 
  backupable resources with _HYCU R-Cloud_.
- The purpose of the _General Concepts_ chapter is to explain the concepts that are specific to the Python 
  implementation of R-Cloud module/R-Cloud SDK.


> ***Note***: This README is not intended to be a standalone documentation. Its objective is to explain the specifics of 
> the Python implementation details of an R-Cloud module and offer examples to streamline the initial stages of development.
>
> While implementing an R-Cloud module, it is crucial to refer to the _HYCU R-Cloud Developer Guide_ as the primary 
> documentation source. _HYCU R-Cloud Developer Guide_ provides a runtime-independent extensive coverage of all 
> concepts, ensuring a thorough understanding.

---

## Table of Contents

- [Tutorials](#tutorials)
    - [Tutorial 1: First Steps](#tutorial-1-first-steps)
    - [Tutorial 2: Sync SaaS resources](#tutorial-2-sync-saas-resources)
- [General Concepts](#general-concepts)
    - [Project Layout](#project-layout)
    - [SDK Dependency](#sdk-dependency)
    - [Local Testing](#local-testing)
    - [Error Handling](#error-handling)
    - [Environment Variables](#environment-variables-)
    - [Bucket Storage Access](#bucket-storage-access)
        - [Storage Service Client](#storage-service-client)
        - ~~[DEPRECATED: Storage Client](#deprecated-storage-client-)~~
    - [Module User Guide](#module-user-guide)

---
# Tutorials

## _Tutorial 1:_ First Steps
The purpose of the _First Steps_ tutorial is to help the module developer:

- Build the Python virtual environment with all the necessary dependencies.
- Build and deploy the initial version of the module.
- Add the module as a source to _HYCU R-Cloud_ and sync the test resources.

### _Step 1:_ Python virtual environment

Create and activate a Python virtual environment for development. This script downloads all the dependencies, including _"Python SDK for R-Cloud Plugins"_.

- For Linux
   ```shell
   $ ./make-dev-venv.sh
   $ . $(cat .venv)/bin/activate
   ```
- For Windows (PowerShell)
  ```powershell
  PS> .\make-dev-venv.ps1
  PS> Invoke-Expression -Command "$(Get-Content -Path .\.venv-win)\Scripts\Activate.ps1"
  ```
  
### _Step 2:_ Test the build process

Commit some minor changes (e.g. change the description of a root resource - see `_list_root_resources()` in `plugin/impl/roots.py`) and 
push them to the `master` branch. 

Wait for a few moments for the build to finish. You may see the build progress and build logs in _HYCU R-Cloud_.

### _Step 3:_ Add source to HYCU and sync SaaS sources

#### Add a SaaS source 
1. Log in to _HYCU R-Cloud_.
1. Add a new SaaS source.
1. Select the _Display name_ and _Protection set_ of the SaaS source.
1. Set values to _URL_, _User_ and _Password_ fields. The example module you just deployed is simulating a custom authentication type - for testing purposes, you may put in any values.
1. Click _Save_ to confirm the configuration. 

#### Sync test resources
1. Go to _SaaS_ panel in the _HYCU R-Cloud_.
1. Click the _Refresh_ button to trigger the sync of SaaS sources.
1. When the sync is finished, SaaS root resources `Root 0`, `Root 1` and `Root 2` should be visible in the SaaS panel.

---

## _Tutorial 2:_ Sync SaaS resources

After having successfully finished the _[First Steps](#tutorial-1-first-steps)_ guide, the test SaaS sources `Root 0`, 
`Root 1` and `Root 2` should be visible in _HYCU R-Cloud_.

The aim of this guide is to successfully sync the actual SaaS sources that need to be backed up.

### _Step 1:_ Current implementation

Resource sync should be implemented in `handle_roots()` handler (see `plugin/impl/roots.py`). 

The current implementation of `handle_roots()` calls the `_list_root_resources()` function, which provides the 3 dummy sources visible in _HYCU R-Cloud_ (`Root 0`, 
`Root 1` and `Root 2`).
The goal of this guide is to replace the dummy function `_list_root_resources()` with actual API requests to the SaaS, in order to get the actual resources.

Before the implementation of `handle_roots()` handler, a few other things need to be configured - module authentication ([Step 2](#step-2-define-authentication-context)) and module capabilities ([Step 3](#step-3-declare-modules-capabilities)).


### _Step 2:_ Define authentication context

Import or define an authentication context of the module in `plugin/impl/schema.py` (see `FIXME` comments). The authentication context defined here must be in
sync with what is set in `Capabilities.auth` (defined in Step 3).

There are a few templates for authentication already provided in form of (commented) imports in `plugin/impl/schema.py`.


```python
"""
Code snippet from `plugin/impl/schema.py` - in your code, use *one* of the imports provided below.
"""
# from rcsdk.schema import GenericAuthInfo as AuthContext
# from rcsdk.schema import AwsAuthContext as AuthContext
# from rcsdk.schema import GcAuthContext as AuthContext
# from rcsdk.schema import AuthInfo as AuthContext
```

See the 3 authentication options described below to use the right import and correctly configure the chosen authentication type:

#### _Step 2, Option 1:_ Module requires one authentication type (GC, AWS) or no authentication

If the module requires a single auth type (GC access token, AWS session token, or no authentication), use one of the 3 imports provided in `plugin/impl/schema.py`:

  - `AwsAuthContext`: Module supports AWS session token authentication.
  - `GcAuthContext`: Module supports GC access token authentication.
  - `GenericAuthInfo`: Module does not require authentication.

> ***Note***: If one of the above-mentioned imports is used (`AwsAuthContext`, `GcAuthContext` or `GenericAuthInfo`), then the `AuthContext` class **must be removed**
> from `plugin/impl/schema.py`.

#### _Step 2, Option 2:_ Module requires a custom authentication type

If the module requires a single, but custom authentication type, then provide the needed parameters for authentication in `AuthContext` class.
```python
""" 
EXAMPLE - AuthContext (see `plugin/impl/schema.py`)

AuthContext should only be used if custom authentication type is needed. 
Otherwise, *REMOVE* this class from `plugin/impl/schema.py`.

The parameters below (instanceUrl, instanceUser, instancePassword_secret) are provided just as an example.
Change/modify them to match your individual authentication requirements.
"""
class AuthContext:
    instanceUrl: str
    instanceUser: str
    instancePassword_secret: str
```

#### _Step 2, Option 3:_ Module supports multiple authentication types

If the module requires two or more authentication types, use the import below:

- `AuthInfo`: Module is supporting at least two auth types (GC access token, AWS session token, or custom auth type).

> ***Note***: If `AuthInfo` is used as authentication type, then the `AuthContext` class **must be removed**
> from `plugin/impl/schema.py`.


### _Step 3:_ Declare module's capabilities

The module must also declare its capabilities in order to receive correctly formed API requests. The module's capabilities are declared 
in `Capabilities` in `plugin/main.py`.

```python
"""
CODE SNIPPET FROM `plugin/main.py`
"""

from rcsdk.schema import Capabilities
from rcsdk.constants import AuthType, FeatureType, StorageType

"""
EXAMPLE - see `plugin/main.py`, where a pre-filled example with 'FIXME' comments is provided.
"""
Capabilities(
    auth=[],                    # supported authentication types (see `AuthType` enum in `rcsdk/constants.py`)
    features=[],                # supported features (see `FeatureType` enum in `rcsdk/constants.py`)
    supportedStorageTypes=[]    # supported storage types (see `StorageType` enum in `rcsdk/constants.py`)
)
```

#### _Step 3.1:_ Capabilities.auth

> **_Note_**: `auth` settings in the module's `Capabilities` must be in sync with what was set in the [Authentication Context](#step-2-define-authentication-context) step.

`Capabilities.auth`: Set one or more authentication types (see `AuthType` enum in `rcsdk/constants.py`):

- `AUTH_AWS_SESSION_TOKEN`: Module requires AWS session token authentication.
- `AUTH_GC_ACCESS_TOKEN`: Module requires GC access token authentication.
- `AUTH_SETTINGS`: Module requires custom authentication schema, see `/api/schemas/auth`.
- If the module doesn't require authentication, then the `auth` list should be empty.

#### _Step 3.2:_ Capabilities.features
`Capabilities.features`: Set one or more module's features (see `FeatureType` enum in `rcsdk/constants.py`):

- `REMOTE_STORAGE_ONLY`: Module uses only storage provided by the SaaS, not cloud storage.
- `CROSS_SOURCE_RESTORE`: Module supports restore to a different source.
- `ANOMALY_DETECTION`: Module supports anomaly detection features.
- `BULK_OPERATIONS`: Module operates in a stateful mode.
- `EXPORT`: Module supports export capability (only supported for modules that utilize `StorageType.STORAGE_SERVICE`).

#### _Step 3.3:_ Capabilities.supportedStorageTypes
> ***Note***: `Capabilities.supportedStorageTypes` only need to be set if the module is using bucket storage. 
> If `FeatureType` is set to `REMOTE_STORAGE_ONLY`, then `supportedStorageTypes` parameter should be omitted.

`Capabilities.supportedStorageTypes`: Set the storage types that the module supports (see `StorageType` enum in `rcsdk/constants.py`).

- `STORAGE_SERVICE`: Module is using *Storage Service* for cloud storage access (see [Storage Service Client](#storage-service-client) chapter below)
- ~~`STORAGE_GC`: (**DEPRECATED**) Module requires GC-specific info for cloud storage access.~~
- ~~`STORAGE_AWS`: (**DEPRECATED**) Module requires AWS-specific info for cloud storage access.~~


### _Step 4:_ Authentication

The first step of syncing the resources is to correctly configure the `handle_schemas_auth()` handler (see `plugin/impl/auth_settings.py`). 
Currently, the `handle_schemas_auth()` handler is implemented for a custom authentication type, which requires 3 test inputs - _URL_, _User_ and _Password_.

> **_Note_**: The option you choose here must be in sync with what was set in [Authentication Context](#step-2-define-authentication-context) and [Module Capabilities](#step-3-declare-modules-capabilities) steps.

Choose the correct option below:

#### _Step 4, Option 1:_ Module requires one authentication type (GC, AWS) or no authentication

See this example, if the module requires one authentication type to authenticate with the SaaS - `AUTH_GC_ACCESS_TOKEN`, `AUTH_AWS_SESSION_TOKEN` or no authentication.

In this case, `plugin/impl/auth_settings.py` should be entirely removed. _HYCU R-Cloud_ will automatically provide the correct form for authentication, based on the
authentication type defined in [Module Capabilities](#step-3-declare-modules-capabilities).

After `handle_schemas_auth()` is removed, then you must also remove the routing for `api_schemas_auth_handler`.

```python
"""
Code snippet from `plugin/main.py`
"""

rcsdk.skeleton.config.configure(
    ...
    # *REMOVE* this routing, if 'AUTH_AWS_SESSION_TOKEN' or 'AUTH_GC_ACCESS_TOKEN' is set in Capabilities.auth, and 'handle_schemas_auth()' handler was removed
    api_schemas_auth_handler=handle_schemas_auth,
    ...    
```

#### _Step 4, Option 2:_ Module requires a custom authentication type

See this example, if the module requires a custom authentication type.

In this case the developer should prepare the correct authentication input form.

Modify the provided example from `handle_schemas_auth()` (see `plugin/impl/auth_settings.py`) to prepare the form with all authentication parameters required by the SaaS.

#### _Step 4, Option 3:_ Module supports both AWS and GC authentication types

See this example if the module requires two auth types - `AUTH_AWS_SESSION_TOKEN` and `AUTH_GC_ACCESS_TOKEN`.

In this case, you do not need to specify a separate handler with the input form - _HYCU R-Cloud_ will automatically provide the required form to set AWS and GC credentials.

Remove the `handle_schemas_auth()` handler (in `plugin/impl/auth_settings.py`), and remove the routing for `api_schemas_auth_handler` in `plugin/main.py` (see the example below).

```python
"""
Code snippet from `plugin/main.py`
"""

rcsdk.skeleton.config.configure(
    ...
    # *REMOVE* this routing, if 'AUTH_AWS_SESSION_TOKEN' or 'AUTH_GC_ACCESS_TOKEN' is set in Capabilities.auth, and 'handle_schemas_auth()' handler was removed
    api_schemas_auth_handler=handle_schemas_auth,
    ...
)
```

#### _Step 4, Option 4:_ Module supports multiple authentication types

See this example, if custom authentication type (`AUTH_SETTINGS`) is required, in combination with `AUTH_AWS_SESSION_TOKEN` and/or `AUTH_GC_ACCESS_TOKEN`.

In this case, you only need to manually define the input form for the custom authentication types (`AUTH_SETTINGS`). _HYCU R-Cloud_ will automatically provide the form for `AUTH_AWS_SESSION_TOKEN` and/or `AUTH_GC_ACCESS_TOKEN`.

`AuthInfo` object describes the authentication info in requests - `settings` serve as a template for the custom authentication type.

```python
class AuthInfo:
    """Authentication info in requests."""

    #: data for authentication type AUTH_SETTINGS
    settings: Optional[JsonObject] = field(default=None, metadata=OPTIONAL_OBJECT)

    #: data for authentication type AUTH_AWS_SESSION_TOKEN
    awsSessionToken: Optional[AwsAuthContext] = field(default=None, metadata=OPTIONAL_OBJECT)

    #: data for authentication type AUTH_GC_ACCESS_TOKEN
    gcAccessToken: Optional[GcAuthContext] = field(default=None, metadata=OPTIONAL_OBJECT)
```

Modify the provided example from `handle_schemas_auth()` (`plugin/impl/auth_settings.py`) to prepare the form with all authentication parameters required by the custom authentication type.

### _Step 5:_ Implement handle_roots() handler

Now everything is ready to implement the `handle_roots()` handler. The purpose of this handler is to return a list of 
top-level resources, which are then visible in _HYCU R-Cloud_.

Implement the API calls/use the client to sync the required resources, and provide a correctly formed `RootsResponse`. 
You should refer to _HYCU R-Cloud Developer Guide_ to correctly form the `RootsResponse`.

When using the client/making API calls, the authentication info from `auth_ctx` should be used (if needed). 

```python
"""
Code snippet from `plugin/impl/roots.py`
"""

from typing import Optional
from rcsdk.schema import RootsRequest, RootsResponse
from .schema import AuthContext


def handle_roots(req: RootsRequest, auth_ctx: Optional[AuthContext]) -> RootsResponse:
  """Return a list of top-level resources."""

  # Use `auth_ctx` to get authentication info, used for authentication on the SaaS side (if needed)
  # Implement the API calls/use the client to get the resources from the SaaS
  # Generate a list of `RootResource` objects from the response received from SaaS

  # Return `RootsResponse` (a list of `RootResource` objects, optionally add a list of `Error` objects (if any) and
  #  pagination context (if needed))
  # Assume `list_of_root_resources` is a list of correctly formed `RootResource` objects
  return RootsResponse(resources=list_of_root_resources)
```

### _Step 6:_ Testing the settings

Run the test script to check for errors, and fix the potential errors in your configuration.

- For Linux
   ```shell
   $ ./check.sh
   ```
- For Windows (PowerShell)
  ```powershell
  PS> .\check.ps1
  ```

### _Step 7:_ Push to master

After running the `check.sh` script (and fixing any potential errors), commit and push the changes to master branch. This 
will trigger the build and deploy the new version of the module.

You may see the build progress and build logs in _HYCU R-Cloud_.

### _Step 8:_ Re-add source

In the [Step 3](#step-3-add-source-to-hycu-and-sync-saas-sources) of the _First Steps_ guide you added a source to _HYCU R-Cloud_.

In case you changed the authentication context while following this guide, you should modify the SaaS source credentials.

In case the authentication context didn't change, skip the _"Modify SaaS source credentials"_ part, and go directly to _"Sync test resources"_.

#### Modify SaaS source credentials 
1. Log in to _HYCU R-Cloud_.
1. Open the _Sources_ dialog and go to _SAAS_ tab.
1. Before the change, there were _URL_, _User_ and _Password_ fields to set in the dialog. If the [Step 4: Authentication](#step-4-authentication)
   from this guide was done correctly, you'll now notice that inputs here are customized to your configuration.
1. Provide the required authentication info based on your individual settings.
1. Click _Save_ to update the authentication configuration. 


#### Sync test resources
1. Go to _SaaS_ panel in the _HYCU R-Cloud_.
1. Click the _Refresh_ button to trigger the sync of SaaS sources.
1. When the sync is finished, the dummy SaaS root resources (`Root 0`, `Root 1` and `Root 2`) should be replaced with 
   the actual root resources from `handle_api_roots()` handler (implemented in [Step 5](#step-5-implement-handleroots-handler) of this guide).

---

# General concepts

## Project Layout

### Root Directory

| Name             | Description                                                 |
|------------------|-------------------------------------------------------------|
| `pyproject.toml` | Project configuration file.                                 |
| `manifest.json`  | The file containing the plugin manifest.                    |
| `Dockerfile`     | Configuration for building the Docker image for the plugin. |
| `plugin/`        | Plugin implementation directory.                            |
| `tests/`         | Unit tests and manual test scripts.                         |
| `*.sh`           | Helper shell scripts; see below.                            |
| `*.ps1`          | Helper PowerShell scripts; see below.                       |
| `develop.py`     | Development plugin server; do not run directly, see below.  |


### Plugin Implementation and Tests

| Name                                  | Description                                              |
|---------------------------------------|----------------------------------------------------------|
| `plugin/main.py`                      | Plugin entry point, with plugin configuration.           |
| `plugin/impl/schema.py`               | Context definitions for requests and responses.          |
| `plugin/impl/version.py`              | Handler for `/api/version`; returns the plugin manifest. |
| `plugin/impl/auth_settings.py`        | Handler for `/api/schemas/auth`.                         |
| `plugin/impl/auth_schema_validate.py` | Handler for `/api/schemas/auth/validate`.                |
| `plugin/impl/roots.py`                | Handler for `/api/roots`.                                |
| `plugin/impl/resources.py`            | Handler for `/api/resources`.                            |
| `plugin/impl/config_backup.py`        | Handler for `/api/config/backup`.                        |
| `plugin/impl/backup.py`               | Handler for `/api/backup`.                               |
| `plugin/impl/scenarios_restore.py`    | Handler for `/api/scenarios/restore`.                    |
| `plugin/impl/config_restore.py`       | Handler for `/api/config/restore`.                       |
| `plugin/impl/restore.py`              | Handler for `/api/restore`.                              |
| `plugin/impl/cleanup.py`              | Handler for `/api/cleanup`.                              |
| `plugin/impl/status.py`               | Handler(s) for `/api/status`.                            |
| `test/manual/flow.py`                 | Manual flow test (see below).                            |

---

## SDK Dependency

In the `pyproject.toml` file, the dependency on the SDK is declared here:
```toml
dependencies = [
    'plugin-python-sdk @ git+https://x-token-auth:<PERMANENT_TOKEN>.../plugin_python_sdk.git@<VERSION_TAG>',
    'gunicorn == 20.1.0',
]
```
This will install a generic development kit when the build is triggered. 

The version of the SDK is defined by the `<VERSION_TAG>` (see above). 
When the SDK is updated, and you wish to use the latest SDK version in your project, just bump the version in the 
`<VERSION_TAG>` and trigger a new build.

---

## Local Testing

### Local environment

> ***Note***: This step is already described in the [First steps](#tutorial-1-first-steps) guide. If you already completed these steps, and your venv is activated, skip this chapter and proceed to [Testing](#testing) 

Create and activate a Python virtual environment for development. This script downloads all the dependencies, including _"Python SDK for R-Cloud Plugins"_.

- For Linux
   ```shell
   $ ./make-dev-venv.sh
   $ . $(cat .venv)/bin/activate
   ```
- For Windows (PowerShell)
  ```powershell
  PS> .\make-dev-venv.ps1
  PS> Invoke-Expression -Command "$(Get-Content -Path .\.venv-win)\Scripts\Activate.ps1"
  ```

### Testing

> ***Note***: The following examples assume that the development virtual environment has been activated.

To run the tests you may have added:
```
$ python -m pytest
```
Test scripts in `tests/manual` will be ignored.

To run a development version of the plugin through the Flask web server on port 8080:
- For Linux
    ```shell
    $ ./run-local-develop.sh
    ```
- For Windows (PowerShell)
    ```powershell
    PS> .\run-local-develop.ps1
    ```

To run the plugin through gunicorn on port 8080:
```shell
$ ./run-local-gunicorn.sh
```

To run a simple list/backup/restore test with a local plugin server:
```shell
$ python tests/manual/flow.py
```
This runs the server as with `run-local-develop.sh` and performs the test. To
test against a gunicorn-based plugin, run:
```shell
$ python tests/manual/flow.py --skip-server
```

---

## Error Handling

The SDK supports handling of 2 types of errors - _fatal_ and _non-fatal_ errors (see `backup.py` for examples).

From the HTTP API perspective, non-fatal and fatal errors are just different fields in a JSON:
- Non-fatal errors are returned as a list in responses (see `errors` optional field in `RootsResponse`, 
  `ResourcesResponse`, `BackupResponse`, `RestoreResponse`, etc.)
- Fatal errors are returned as an `error` within `ErrorResponse`.

From the plugin perspective, the errors should be handled as follows:
- Non-fatal errors should be added to the `errors` list in responses (see `RootsResponse`, `ResourcesResponse`, 
  `BackupResponse`, `RestoreResponse`, etc.)
- For fatal errors the plugin should raise an `Exception`. If a plugin raises an `Exception`, the SDK internally 
  handles the fatal exception and the API returns correctly formatted `ErrorResponse`.
    - The use of `APIError` class is encouraged when raising fatal exceptions. `APIError` contains fields that match 
        HTTP fatal error.
        - If `APIError` class is used, all the data can be passed (code and payload).
        - If an arbitrary exception class is used, then only the description field inferred from the exception 
          message is passed.

---

## Environment Variables 

- `PORT`: All published R-Cloud modules are expected to honor the `PORT` environment variable for configuring the 
  listening port. `ENV` variable `PORT` must also be present in `Dockerfile`, so the module always has `PORT` 
  environment variable available.

> ***Note***: For **local** testing, modules may use a custom listening port that can be set in `develop.py`.

---

## Bucket Storage Access

### Storage Service Client

_Storage Service_ provides an abstraction layer for cloud storage operations - uploading and downloading objects. 
It is recommended that modules use `StorageServiceClient` (`rcsdk.utils.storage_service_client`) for all 
interactions with the cloud storage. 

`StorageServiceClient` is replacing the deprecated `StorageClient` (see below).  

_Storage Service_ provides a few additional features compared to `StorageClient` (or directly implementing 
_Google Storage_/_S3 Storage_ access within the module):

- Compression of all data uploaded to the target.
- Storage isolation per backup task, so the use of _Staging Target_ is no longer required.
- Authentication with the cloud storage is done by the platform (not by the module itself).

#### Fundamentals

To use `StorageServiceClient`, module must advertise `StorageType.STORAGE_SERVICE` in `Capabilities` (see `./plugin/impl/main.py`)

_Storage Service_ enables storage access through an instance of `StorageServiceClient`, initialized with the 
storage `StorageInfo` configuration. `StorageInfo` configuration can be obtained from `BackupRequest`, 
`BulkBackupRequest`, `RestoreRequest`, `BulkRestoreRequest`, `ExportRequest`, `BulkExportRequest` or `StatusRequest` 
(_see the example below_).

```python
from typing import Optional

from plugin.impl.schema import AuthContext  # AuthContext must be defined on the plugin side (it's not a part of SDK)

from rcsdk.schema import RestoreRequest, StorageInfo, RestoreResponse
from rcsdk.utils.storage_service_client import StorageServiceClient
from .schema import RootContext, ResourceContext  # Schema must be defined on the plugin side (it's not a part of SDK)

# Assume the plugin announces StorageType.STORAGE_SERVICE
def handle_restore(
    req: RestoreRequest,
    ctx: RootContext | ResourceContext,
    auth_ctx: Optional[AuthContext],
) -> RestoreResponse:
    assert req.storage is not None  # typecheck only
    storage_info: StorageInfo = req.storage
    storage_client = StorageServiceClient(storage_info)
    ...
```

`StorageServiceClient` supports operations with `JSONObject` and `IOBase` objects.
It implements 3 methods:
- `put_object(self, path: str, data: JsonObject | io.IOBase | IO, retry: bool = False) -> None:`
    - Stores the given IOBase object or JSON object to the storage.
    - **Note**: The specified `path` represents a relative path. The `path` is self-contained by the plugin.
- `get_object(self, path: str, object_type: ObjectType) -> JsonObject | io.IOBase:`
    - Returns an object according to specified object_type.
    - **Note**: The specified `path` represents a relative path. The `path` is self-contained by the plugin.
- `export_object(self, source_path: str, destination_path: str) -> None:`
  - Writes data from the source path to the destination path
  - **Note**: The specified `path` represents a relative path. The `path` is self-contained by the plugin.
> **Note**: The module should _never_ modify the `StorageInfo` object. All operation-specific details should be set 
through `put_object`, `get_object` and `export_object` parameters.

#### Example 1: StorageServiceClient - *put_object()*
**Note**: _In some examples below, binary files (`.bin`) are used. You are not in any way limited to this file type, 
you may use any file type you applicable in your case._ 

```python
import io
from typing import Optional

from plugin.impl.schema import AuthContext  # AuthContext must be defined on the plugin side (it's not a part of SDK)

from rcsdk.schema import BackupRequest, BackupResponse, StorageInfo
from rcsdk.typing import JsonObject
from rcsdk.utils.storage_service_client import StorageServiceClient
from .schema import RootContext, ResourceContext  # Schema must be defined on the plugin side (it's not a part of SDK)

# Assume the plugin announces StorageType.STORAGE_SERVICE
def handle_backup(
    req: BackupRequest,
    ctx: RootContext | ResourceContext,
    auth_ctx: Optional[AuthContext],
) -> BackupResponse:
    # Handle backup 
    ...
    storage_info: StorageInfo = req.storage
    storage_client = StorageServiceClient(storage_info)
    
    ''' EXAMPLE 1 - JSON '''
    # Upload backup data (in JSON format) to the storage specified in BackupRequest     
    data_json: JsonObject = {"example_key": "example_value"}
    path = "example.json"
    storage_client.put_object(data=data_json, path=path)
    
    ''' EXAMPLE 2 - JSON with retried uploads'''
    # Setting retry to True allows the storage service to retry uploads of objects with the same name
    data_json: JsonObject = {"example_key": "example_value"}
    path = "example.json"
    storage_client.put_object(data=data_json, path=path, retry=True)

    ''' EXAMPLE 3 - IOBase '''
    # Upload IOBase data
    # Assume "content" is the data that needs to be uploaded to the bucket
    data_io_base: io.IOBase = io.BytesIO(content)
    storage_client.put_object(data=data_io_base, path=path)
        
    ''' EXAMPLE 4 - IOBase. Streaming from local file to cloud storage '''
    # Stream data from an open file to the storage
    # Assume example_large_file.bin is a file that is a part of a backup and needs to be uploaded to the bucket
    path = "example_large_file.bin"
    with open("example_large_file.bin", "rb") as file:
        storage_client.put_object(data=file, path=path)
        
    # Handle the rest of the backup task and return the BackupResponse (see ./plugin/impl/backup.py)
    ...
```

#### Example 2: StorageServiceClient - *get_object()*
**Note**: _In some examples below, binary files (`.bin`) are used. You are not in any way limited to this file type, 
you may use any file type you applicable in your case._  

```python
import io
from typing import Optional

from plugin.impl.schema import AuthContext  # AuthContext must be defined on the plugin side (it's not a part of SDK)

from rcsdk.schema import ObjectType, RestoreRequest, StorageInfo, RestoreResponse
from rcsdk.typing import JsonObject
from rcsdk.utils.storage_service_client import StorageServiceClient
from .schema import RootContext, ResourceContext  # Schema must be defined on the plugin side (it's not a part of SDK)

CHUNK_SIZE = 512 * 1024     # 512 kB is the recommended general purpose chunk size

# Assume the plugin announces StorageType.STORAGE_SERVICE
def handle_restore(
    req: RestoreRequest,
    ctx: RootContext | ResourceContext,
    auth_ctx: Optional[AuthContext],
) -> RestoreResponse:
    # Handle any necessary steps for restore...
    ...
    
    storage_info: StorageInfo = req.storage
    storage_client = StorageServiceClient(storage_info)

    ''' EXAMPLE 1 - JSON '''
    # Get the JSON data needed for restore
    path = "example.json"
    data_json: JsonObject = storage_client.get_object(object_type=ObjectType.JSON, path=path)

    ''' EXAMPLE 2 - IOBase, load entire file into memory (applicable only for small files) '''
    path = "example_io.bin"
    storage_file_handle = storage_client.get_object(object_type=ObjectType.IO_BASE, path=path)
    # Load the entire file into memory
    bucket_file_content = io.BytesIO(storage_file_handle.read())

    ''' EXAMPLE 3 - IOBase, Streaming - read a file from a bucket chunk by chunk '''
    path = "example_large_file_on_storage.bin"
    bucket_file_handle = storage_client.get_object(object_type=ObjectType.IO_BASE, path=path)

    # A loop that reads the file from bucket chunk by chunk
    while True:
        chunk = bucket_file_handle.read(CHUNK_SIZE)  # 512 kB is the recommended general purpose chunk size    
        if not chunk:
            break
        # Do any kind of processing of the chunk data, save to file, upload to another location,...
            
    ''' EXAMPLE 4 - IOBase, Streaming. Save a file from a bucket into a local file '''
    path = "example_large_file_on_storage.txt"
    with open("example_large_file_local.txt", "wb") as local_file:
        bucket_file_handle = storage_client.get_object(object_type=ObjectType.IO_BASE, path=path)

        # A loop that reads the file from bucket chunk by chunk
        while True:
            chunk = bucket_file_handle.read(CHUNK_SIZE)  # 512 kB is the recommended general purpose chunk size    
            if not chunk:
                break
            local_file.write(chunk)
            
    # Handle the rest of the restore task and return the RestoreResponse (see ./plugin/impl/restore.py)
    ...
```

#### Example 3: StorageServiceClient - *export_object()*

```python
from typing import Optional

from rcsdk.schema import ExportRequest, ExportResponse, StorageInfo
from rcsdk.utils.storage_service_client import StorageServiceClient

from .schema import AuthContext, ResourceContext, RootContext # Schema must be defined on the plugin side (it's not a part of SDK)

# Assume the plugin announces StorageType.STORAGE_SERVICE and FeatureType.EXPORT
def handle_export(
    req: ExportRequest,
    ctx: RootContext | ResourceContext,
    _: Optional[AuthContext],  # NOTE: AuthContext is included within the function scope to support generalization
) -> ExportResponse:
    
    # Handle any necessary steps for export...
    ...
        
    storage_info: StorageInfo = req.storage
    storage_client = StorageServiceClient(storage_info)
    
    # Since the paths are relative, the source and destination paths can be the same.
    source_relative_path = "/path/example/example.txt"
    destination_relative_path = "/path/example/exported/example.txt"
    
    storage_client.export_object(source_relative_path, destination_relative_path)
    
     # Handle the rest of the export task and return the ExportResponse (see ./plugin/impl/export.py)
    ...
```

### _[Deprecated]_ Storage Client 

> **Note:** `StorageClient` was deprecated with SDK version `v1.3.0`. Please use `StorageServiceClient` instead, see the 
> chapter above for documentation.

`StorageClient` (`rcsdk.utils.storage_client`) provides an abstraction layer for accessing *Google Storage* or *S3 
Storage*. It enables bucket storage access through a uniform interface regardless of whether *Google Storage* or *S3 
Storage* is used.

#### Fundamentals

To use `StorageClient`, module must advertise `StorageType.STORAGE_GC` or `StorageType.STORAGE_AWS` in `Capabilities` (see `./plugin/impl/main.py`)

`StorageClient()` supports operations with `JSONObject` and `IOBase` objects. It implements 4 methods:

- `put_object(self, storage_info: StorageInfo, data: JsonObject | io.IOBase | IO) -> None`
    - Stores the given `IOBase` object or `JSONObject` to bucket.
    - **Note**: This method automatically detects the type of the `data` parameter (either `JSONObject` or `IOBase`), 
      so no additional parameter needs to be set.
- `get_object(self, storage_info: StorageInfo, object_type: ObjectType = ObjectType.JSON) -> JsonObject | io.IOBase`
    - Returns the object identified by `storage_info` from the bucket.
    - **Note**: To get `IOBase` objects, the `object_type` parameter must be specified. If omitted, the `object_type` 
      parameter defaults to `ObjectType.JSON`.
- `object_exists(self, storage_info: StorageInfo) -> bool`
    - Checks if the object exists in bucket.
- `get_object_metadata(self, storage_info: StorageInfo) -> JsonObject`
    - Returns the metadata of the object identified by `storage_info`.

#### Example 1: StorageClient - *put_object()*
_Note: In some examples below, binary files (`.bin`) are used. You are not in any way limited to this file type, 
you may use any file type you applicable in your case._ 

```python
import copy
import io
from typing import Optional

from plugin.impl.schema import AuthContext  # AuthContext must be defined on the plugin side (it's not a part of SDK)

from rcsdk.schema import BackupRequest, BackupResponse
from rcsdk.typing import JsonObject
from rcsdk.utils.storage_client import StorageClient
from .schema import RootContext, ResourceContext  # Schema must be defined on the plugin side (it's not a part of SDK)


# Assume the plugin announces StorageType.STORAGE_GC or StorageType.STORAGE_S3 
# For backup example, see ./plugin/impl/backup.py (in the plugin project)
def handle_backup(
    req: BackupRequest,
    ctx: RootContext | ResourceContext,
    auth_ctx: Optional[AuthContext],
) -> BackupResponse:
    # Handle backup 
    ...
    
    # Create an instance of StorageClient
    storage_client = StorageClient()
    
    ''' EXAMPLE 1 - JSON '''
    # Upload backup data (in JSON format) to the bucket specified in BackupRequest     
    data_json: JsonObject = {"example_key": "example_value"}
    # Note: 'req.storage.url' parameter needs to be modified to put the JSON file to the correct location
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example.json"
    storage_client.put_object(storage_info=storage_tmp_copy, data=data_json)
    

    ''' EXAMPLE 2 - IOBase. '''
    # Assume "content" is the data that needs to be uploaded to the bucket.
    # Note: This is only applicable for small files, since it loads the entire "content" into the memory 
    data_io_base: io.IOBase = io.BytesIO(content)
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example_io.bin"
    storage_client.put_object(storage_info=storage_tmp_copy, data=data_io_base)
    

    ''' EXAMPLE 3 - IOBase. Streaming from a local file to bucket '''
    # Stream data from an open file to the storage
    # Assume example_large_file.bin is a file that is a part of a backup and needs to be uploaded to the bucket
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example_large_file.bin"
    with open("example_large_file.bin", "rb") as file:
        storage_client.put_object(storage_info=storage_tmp_copy, data=file)

    # Handle the rest of the backup task and return the BackupResponse (see ./plugin/impl/backup.py)
    ...
```

#### Example 2: StorageClient - *get_object()*
_Note: In some examples below, binary files (`.bin`) are used. You are not in any way limited to this file type, 
you may use any file type you applicable in your case._ 

```python
import copy
import io
from typing import Optional

from plugin.impl.schema import AuthContext  # AuthContext must be defined on the plugin side (it's not a part of SDK)

from rcsdk.schema import RestoreRequest, RestoreResponse, ObjectType
from rcsdk.typing import JsonObject
from rcsdk.utils.storage_client import StorageClient
from .schema import RootContext, ResourceContext  # Schema must be defined on the plugin side (it's not a part of SDK)

CHUNK_SIZE = 512 * 1024     # 512 kB is the recommended general purpose chunk size

# Assume the plugin announces StorageType.STORAGE_GC or StorageType.STORAGE_S3
# For restore example, see ./plugin/impl/restore.py (in the plugin project)
def handle_restore(
    req: RestoreRequest,
    ctx: RootContext | ResourceContext,
    auth_ctx: Optional[AuthContext],
) -> RestoreResponse:
    # Handle any necessary steps for restore...
    ...
    
    # Create an instance of StorageClient
    storage_client = StorageClient()

    ''' EXAMPLE 1 - JSON '''
    # Get the JSON data needed for restore
    # Note: 'req.storage.url' parameter needs to be modified to get the desired JSON file
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example.json"
    data_json: JsonObject = storage_client.get_object(storage_info=storage_tmp_copy, object_type=ObjectType.JSON)
    

    ''' EXAMPLE 2 - IOBase, load the entire file into memory (applicable only for small files) '''
    # Get the IOBase data needed for restore
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example_io.bin"
    bucket_file_handle = storage_client.get_object(
        storage_info=storage_tmp_copy,
        object_type=ObjectType.IO_BASE
    )
    # Note: .read() loads the entire file into memory, so this is applicable only for small files
    bucket_file_content = io.BytesIO(bucket_file_handle.read())
    

    ''' EXAMPLE 3 - IOBase, Streaming - read a file from a bucket chunk by chunk '''
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example_large_file_on_bucket.bin"
    bucket_file_handle = storage_client.get_object(
        storage_info=storage_tmp_copy,
        object_type=ObjectType.IO_BASE
    )

    # A loop that reads the file from bucket chunk by chunk
    while True:
        chunk = bucket_file_handle.read(CHUNK_SIZE)  # 512 kB is the recommended general purpose chunk size    
        if not chunk:
            break
        # Do any kind of processing of the chunk data, save to file, upload to another location,...
        

    ''' EXAMPLE 4 - IOBase, Streaming. Save a file from a bucket into a local file '''
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"/{req.taskUuid}/example_large_file_on_bucket.bin"

    with open("example_large_file_local.bin", "wb") as local_file:
        bucket_file_handle = storage_client.get_object(storage_info=storage_tmp_copy, object_type=ObjectType.IO_BASE)

        # A loop that reads the file from bucket chunk by chunk
        while True:
            chunk = bucket_file_handle.read(CHUNK_SIZE)  # 512 kB is the recommended general purpose chunk size    
            if not chunk:
                break
            local_file.write(chunk)
            

    ''' EXAMPLE 5 - Read using iter and a helper function. Save a file from bucket into a local file '''
    storage_tmp_copy = copy.deepcopy(req.storage)
    storage_tmp_copy.url += f"{req.taskUuid}/example_large_file_on_bucket.bin"

    bucket_file_handle = storage_client.get_object(storage_info=storage_tmp_copy, object_type=ObjectType.IO_BASE)

    def read512k():
        return bucket_file_handle.read(CHUNK_SIZE)

    with open("example_large_file_local.bin", "wb") as local_file:
        for chunk in iter(read512k, b''):
            local_file.write(chunk)

    # Handle the rest of the restore task and return the RestoreResponse (see ./plugin/impl/restore.py)
    ...
```

---

## Module User Guide

- Markdown source file should be placed in repository as `userdoc/help.md`.
- Any images should be placed in repository under `userdoc/images/`.
- Use the following steps to build docs for your module:
  - `Git clone` [HYCU R-Cloud module build script repository](https://bitbucket.org/hycuinc/rcloud-module-scripts/src/master/).
  - Follow the steps in the `README.md` file of the cloned repository.