### Documentation is included in the Documentation folder ###

[REFrameWork Documentation](https://github.com/UiPath/ReFrameWork/blob/master/Documentation/REFramework%20documentation.pdf)

### BCHouse_Queue Template ###
**Modified Robotic Enterprise Framework**

* Built on top of *Transactional Business Process* template
* Uses *State Machine* layout for the phases of automation project
* Offers high level logging, exception handling and recovery
* Keeps external settings in *Config.xlsx* file and Orchestrator assets
* Pulls credentials from Orchestrator assets and *Windows Credential Manager*
* Gets transaction data from Orchestrator queue and updates back status
* Takes screenshots in case of system exceptions

### Modification by BCHouse ###

+ Main - Add variable DateTime Start to receive StartPoint
+ Main - Add Argumdent testMode to Main and hand over in Process.xaml to different between ProdMode and Testing- or DevMode
+ Main - Add Argument MaximumTransactionToProcessed to stop the bot after an specific number of transactions
+ Main - Add InitRetry Transitions to Retry Init after an Exception occured in Init State, all Apps will be close for Retry. Add also a ConfigValue to determine RetryNumber of Init
+ Main - Add Workflow RemoveExceptionScreenshots to clean up screenshotFolder. Add also a ConfigValue to determine duration of storage a screenshot
+ Main - Move SetTransactionStatus to Finally of ProcessState
+ Main - Add Workflow QueueReport to get a datatable of relevant TransactionItems that was edited. Add also ConfigValues to receive relevant information
+ Main - Add Workflow CreateReport to save datatable to a new excel file or append to an existing excel file. Add also ConfigValues to dertermine reportFilePath
+ Main - Add Workflow GetQueueMaxRetries to receive to maximum RetryNumber of Queue to retry TransactionItems. This Number will overwrite ConfigValue
 + ./Framework/*InitiAllSettings* - Add Logs when ConfigValues will be overwritten
 + ./Framework/*KillAllProcesses* - Add Retry to kill processes of current User. Add also a ConfigValue to determine application that will be kill
 + ./Framework/*GetTransactionData* - Rename in GetTransactionItem
 + ./Framework/*GetTransactionData* - Add in Initializing. By default a DataTable will build. Can be changed to read an excel file or other transactionTypes
+ Project.json - Disable ModernDesign

### How It Works ###

1. **INITIALIZE PROCESS**
 + ./Framework/*InitiAllSettings* - Load configuration data from Config.xlsx file and from assets
 + ./Framework/*GetAppCredential* - Retrieve credentials from Orchestrator assets or local Windows Credential Manager
 + ./Framework/*InitiAllApplications* - Open and login to applications used throughout the process

2. **GET TRANSACTION DATA**
 + ./Framework/*GetTransactionData* - Fetches transactions from an Orchestrator queue defined by Config("OrchestratorQueueName") or any other configured data source

3. **PROCESS TRANSACTION**
 + *Process* - Process trasaction and invoke other workflows related to the process being automated 
 + ./Framework/*SetTransactionStatus* - Updates the status of the processed transaction (Orchestrator transactions by default): Success, Business Rule Exception or System Exception

4. **END PROCESS**
 + ./Framework/*CloseAllApplications* - Logs out and closes applications used throughout the process


### For New Project ###

1. Check the Config.xlsx file and add/customize any required fields and values
2. Implement InitiAllApplications.xaml and CloseAllApplicatoins.xaml workflows, linking them in the Config.xlsx fields
3. Implement GetTransactionData.xaml and SetTransactionStatus.xaml according to the transaction type being used (Orchestrator queues by default)
4. Implement Process.xaml workflow and invoke other workflows related to the process being automated
