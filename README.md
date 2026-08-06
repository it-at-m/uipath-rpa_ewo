### Documentation

General documentation is included in the `Documentation` folder.

[REFrameWork Documentation](https://github.com/UiPath/ReFrameWork/blob/master/Documentation/REFramework%20documentation.pdf)

## BCHouse Queue Template

**Modified Robotic Enterprise Framework**

* Built on top of the *Transactional Business Process* template
* Uses a state machine layout for the different phases of the automation
* Provides centralized logging, exception handling and recovery
* Keeps external settings in `Data/Config.xlsx` and Orchestrator assets
* Retrieves credentials from Orchestrator assets or the Windows Credential Manager
* Retrieves transaction data from an Orchestrator queue and updates the transaction status after processing
* Takes screenshots when system exceptions occur
* Supports queue reports, configurable retries and automatic cleanup of old exception screenshots

## Modifications by BCHouse

* `Main.xaml`

  * Added a `DateTime` start variable to record the process start time
  * Added the `testMode` argument and passes it to `Process.xaml` to distinguish between production, test and development execution
  * Added the `MaximumTransactionToProcessed` argument to stop the robot after a configured number of transactions
  * Added retry transitions for the initialization state. Before an initialization retry, all applications are closed
  * Added the `RemoveExceptionScreenshot` workflow and a configuration value for the screenshot retention period
  * Moved `SetTransactionStatus.xaml` to the `Finally` section of the process state
  * Added `QueueReport.xaml` to retrieve the relevant queue items processed during the current run
  * Added `CreateReport.xaml` to create a new Excel report or append data to an existing report
  * Added `GetQueueMaxRetryNumber.xaml` to retrieve the maximum retry count configured for the Orchestrator queue. This value overwrites the corresponding value from `Config.xlsx`
* `Framework/InitAllSettings.xaml`

  * Added log messages when configuration values are overwritten
* `Framework/KillAllProcesses.xaml`

  * Added retry handling when terminating processes belonging to the current user
  * Added a configuration value defining which application processes should be terminated
* `Framework/GetTransactionItem.xaml`

  * Renamed from `GetTransactionData.xaml`
  * Uses an Orchestrator queue by default but can be adapted for Excel files, data tables or other transaction sources
* `project.json`

  * Disabled Modern Design

## Project Purpose

The project `EWO.Rechte.und.neue.Nutzer.prod` automates the processing of ServiceNow catalog tasks for EWO/SYNERGO user administration.

For every transaction, the robot retrieves the corresponding ServiceNow ticket, identifies the requested user and the requested groups, checks whether the user already exists in EWO and then updates the user's permissions.

The process can:

* Retrieve a ServiceNow task using the task `sys_id`
* Read the LDAP user ID from the ticket
* Read the requested groups from the ticket variable `Benötigte Rollen / Gruppen`
* Check whether the user already exists in EWO/SYNERGO
* Remove existing groups and roles from an existing user
* Import a user from LDAP or an external system if the user does not yet exist
* Assign the requested groups to the user
* Write progress information and processing results back to the ServiceNow ticket
* Set the ServiceNow task state depending on whether the transaction was successful or failed

> **Current implementation note:** The workflow for adding roles is currently commented out in `Framework/Process.xaml`. Existing roles may be removed during permission cleanup but only the requested groups are actively assigned afterward.

## Process Architecture

### Main workflows

| Workflow                              | Purpose                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------ |
| `Main.xaml`                           | Controls the REFramework state machine and the overall process lifecycle |
| `Framework/InitAllSettings.xaml`      | Loads settings, constants and Orchestrator assets                        |
| `Framework/InitAllApplications.xaml`  | Retrieves the EWO credential, starts the production client and logs in   |
| `Framework/GetTransactionItem.xaml`   | Retrieves the next transaction from the Orchestrator queue               |
| `Framework/Process.xaml`              | Performs the ServiceNow and EWO user-administration steps                |
| `Framework/SetTransactionStatus.xaml` | Updates the queue item and sends the final result to ServiceNow          |
| `Framework/CloseAllApplications.xaml` | Performs the configured application shutdown steps                       |
| `Process/GET Ticket Infos.xaml`       | Retrieves ticket data from ServiceNow                                    |
| `Process/SNOW Response.xaml`          | Adds worknotes to a ServiceNow ticket and optionally changes its state   |

## How It Works

### 1. Initialize Process

* `Framework/InitAllSettings.xaml`

  * Loads configuration values from `Data/Config.xlsx`
  * Loads the configured ServiceNow endpoint values from Orchestrator assets
* `Framework/InitAllApplications.xaml`

  * Retrieves the EWO application credential from the Orchestrator credential asset `rpa006_EWO`
  * Uses the Orchestrator folder `Prod/KVR/rpa006_EWO`
  * Starts the productive EWO client with:

```text
C:\okprg\okewo\icon\AWR-P.bat
```

* Logs in to the EWO/SYNERGO application

### 2. Get Transaction Data

`Framework/GetTransactionItem.xaml` retrieves the next item from the Orchestrator queue configured in `Data/Config.xlsx`.

Current queue configuration:

```text
ProcessEWOQueue
```

The queue item `Reference` must contain the ServiceNow task `sys_id`. This value is used for all subsequent ServiceNow GET and POST requests.

### 3. Retrieve the ServiceNow Ticket

`Process/GET Ticket Infos.xaml` performs the following steps:

1. Retrieves the OAuth client credentials from the credential asset:

```text
crd_snow_clientid_secret
```

2. Requests an OAuth access token using the `client_credentials` grant
3. Sends a GET request to the configured ServiceNow ticket endpoint
4. Appends the queue item reference, the ServiceNow `sys_id`, to the request URL
5. Deserializes the returned ticket information into a JSON object

The process uses the following data from the response:

```text
requested_for.userId
```

This value becomes the LDAP user ID used in EWO.

The requested permissions are read from the ServiceNow variable:

```text
Benötigte Rollen / Gruppen
```

The value is split by commas and each entry is trimmed before processing.

After the ticket has been read, the process writes the following worknote to ServiceNow:

```text
Ticket eingelesen
```

### 4. Check or Import the User in EWO

The robot opens the EWO user administration and searches for the LDAP user ID.

#### Existing user

When the user already exists, the robot:

* Opens the user's permissions
* Removes all currently assigned groups
* Removes all currently assigned roles
* Continues with the assignment of the groups requested in ServiceNow

#### New user

When the user does not yet exist, the robot:

* Opens the tenant and external-system user administration
* Searches for the LDAP user
* Imports the user into EWO
* Writes the following worknote to ServiceNow:

```text
Nutzer wurde aus LDAP übernommen
```

### 5. Assign Requested Groups

For every value read from `Benötigte Rollen / Gruppen`, the robot:

1. Opens the group selection
2. Searches for the requested group
3. Selects the matching result
4. Writes a ServiceNow worknote in the following format:

```text
Gruppe zugeteilt: <group name>
```

5. Applies the complete group selection in EWO

The separate workflow for assigning roles is currently disabled through a `Comment Out` activity.

### 6. Update ServiceNow

`Process/SNOW Response.xaml` is used for intermediate worknotes and final status changes.

The workflow:

1. Retrieves `crd_snow_clientid_secret`
2. Requests a new OAuth bearer token
3. Creates a JSON request body
4. Sends a POST request to:

```text
<Str_SNOW_url_close_comment_ticket><sys_id>/close
```

When no task state is supplied, only a worknote is sent:

```json
{
  "worknote": "..."
}
```

When a task state is supplied, the worknote and state are sent together:

```json
{
  "state": "...",
  "worknote": "..."
}
```

ServiceNow calls are retried up to three times in the main process, with a 30-second interval between attempts.

### 7. Set the Final Transaction Status

`Framework/SetTransactionStatus.xaml` updates both the Orchestrator queue item and the ServiceNow task.

| Result                                 | ServiceNow worknote                                                      | ServiceNow state |
| -------------------------------------- | ------------------------------------------------------------------------ | ---------------: |
| Successful                             | `Bot erfolgreich abgeschlossen`                                          |              `3` |
| Business Rule Exception                | `Business Exception: Bitte Auftrag neu anlegen`                          |              `4` |
| System Exception after maximum retries | `System Exception nach maximalen Retries: Fall bitte manuell bearbeiten` |              `7` |

The corresponding Orchestrator queue item is set to `Successful`, `Failed` with a Business Rule Exception or `Failed` with a System Exception.

### 8. End Process

* `Framework/CloseAllApplications.xaml` performs the configured shutdown steps
* `Framework/KillAllProcesses.xaml` can terminate configured application processes if required
* The current configuration identifies `javaw` as the relevant application process
* Queue reporting and screenshot cleanup are performed according to the values in `Data/Config.xlsx`

## Configuration

### Orchestrator Queue

| Config key                | Current value     | Purpose                                                 |
| ------------------------- | ----------------- | ------------------------------------------------------- |
| `OrchestratorQueueName`   | `ProcessEWOQueue` | Queue containing the ServiceNow task references         |
| `OrchestratorQueueFolder` | Empty             | Uses the current or classic Orchestrator folder context |

### ServiceNow Assets

The actual ServiceNow URLs are not stored in the repository. They are loaded at runtime from Orchestrator assets.

| Config key                          | Orchestrator asset                  | Purpose                                               |
| ----------------------------------- | ----------------------------------- | ----------------------------------------------------- |
| `Str_SNOW_get_oauth_token`          | `Str_SNOW_get_oauth_token`          | OAuth token endpoint                                  |
| `Str_SNOW_request_call`             | `Str_SNOW_request_call`             | Base URL used to retrieve ticket information          |
| `Str_SNOW_url_close_comment_ticket` | `Str_SNOW_url_close_comment_ticket` | Base URL used to add worknotes and change task states |
| ServiceNow credential               | `crd_snow_clientid_secret`          | OAuth client ID and client secret                     |

The `OrchestratorAssetFolder` column is currently empty for the ServiceNow URL assets. Therefore, asset resolution depends on the Orchestrator folder context in which the process is executed.

### Environment Information

The EWO application configuration is explicitly set to production:

* Project name: `EWO.Rechte.und.neue.Nutzer.prod`
* EWO credential folder: `Prod/KVR/rpa006_EWO`
* Startup file: `AWR-P.bat`

The ServiceNow environment cannot be determined from the repository alone because the actual endpoint values are stored in Orchestrator assets. Before deployment, verify that all three ServiceNow URL assets point to the intended production environment and that `crd_snow_clientid_secret` is linked to the correct production credential.

> Some annotations in the workflows still mention that `crd_snow_clientid_secret` must be linked from a test folder. These annotations do not control execution but should be updated or removed after the production asset configuration has been verified.

## Technical Notes

* The ServiceNow integration uses OAuth 2.0 with the `client_credentials` grant
* HTTP requests use a timeout of 6,000 milliseconds
* SSL certificate verification is currently disabled for the ServiceNow HTTP activities through `EnableSSLVerification="False"`
* The process is designed as an unattended UiPath process but requires user interaction with the EWO desktop application
* The application process configured for cleanup is `javaw`
* Requested group values must match searchable EWO group names
* The ServiceNow ticket response must contain `requested_for.userId` and the variable `Benötigte Rollen / Gruppen`

## Requirements Before Deployment

1. Create or verify the Orchestrator queue `ProcessEWOQueue`
2. Ensure that the queue item reference contains the ServiceNow task `sys_id`
3. Verify the EWO credential asset `rpa006_EWO` in `Prod/KVR/rpa006_EWO`
4. Verify the ServiceNow credential asset `crd_snow_clientid_secret`
5. Verify the values and environment of all three ServiceNow URL assets
6. Confirm that `AWR-P.bat` exists on the robot machine
7. Confirm that the robot can access and interact with the EWO/SYNERGO client
8. Confirm that the required ServiceNow fields and variables are returned by the API
9. Review whether SSL certificate verification can be enabled for production use
10. Decide whether role assignment should remain disabled or be reactivated and tested

## For a New Project Based on This Template

1. Review `Data/Config.xlsx` and add or customize the required settings, constants and assets
2. Implement `Framework/InitAllApplications.xaml` and `Framework/CloseAllApplications.xaml`
3. Configure `Framework/GetTransactionItem.xaml` and `Framework/SetTransactionStatus.xaml` for the required transaction type
4. Implement the business logic in `Framework/Process.xaml`
5. Store credentials and environment-specific endpoints in Orchestrator assets rather than directly in the workflows
6. Validate queue references, exception handling, reporting and application cleanup before production deployment
