

# Logging

## Log Levels
### TRACE
For "tracing" the code i.e locating part of a function specifically, showing which branches of an if statement are hit or inputs into a function.

*Alerting:* None
*Environments:* Dev & Staging

### DEBUG

For information that is diagnostically helpful and gives a more low level overview of how the operations/requests are working. These logs should be seen as only available when needed and are by default disabled in pre-production and production environments unless requested otherwise.

Note: We should make extra effort to ensure that we still only add Debug logs *only if they are necessary*, otherwise it is very easy to fall into a situation where there are so many useless debug logs that it becomes almost impossible to query the logs.

*Alerting:* None
*Environments:* Dev & Staging - Prod for specific cases.

### INFO

For generally useful information that gives a high level overview of how the operations/request are working. Such as config settings, persisting new data, adding data to a queue, offloading a request to a different service or if the operations/requests are changing state.

*Alerting:* None
*Environments:* All

### ERROR
For errors that are fatal to the current operation/request, but not to the wider system. Although this log level may not trigger an alert directly, it may have a condition attached to it such as `if > 10 within the last 24 hrs: trigger alert`.

*Alerting:* Conditional
*Environments:* All

### FATAL
For errors that cause a critical issue to the system and may prevent further operations/requests from being handled correctly, or cause for the system to fall into an unrecoverable state, or may cause for data to be lost. Using this log level will trigger an alert and probably wake people up in the middle of the night, so use with caution.

*Alerting:* Always
*Environments:* All

### SILENT
Used for testing or CI only. Setting the environment variable log level to `SILENT` should stop any logs from being written.

#### Example pseudo code

```ts
const handleCreateAccountRequest = (req) => {
  const timeStart = Date.now();
  logger.trace({ req }, `handleCreateAccountRequest started ${timeStart}`);
  logger.debug({ body: req.body }, `processing create account request host=[${req.host}] query=[${req.query}] url=[${req.url}]`)
  const account = createAccount(data);
  logger.trace({ body: req.body }, `handleCreateAccountRequest finished in ${Date.now() - timeStart}ms`);
}

const createAccount = (data) => {
  logger.info(data, `persisting new account: userId=[${data.userId}]`);
  try {
    databaseCreateAccount(data);
  } catch (error: any) {
    if (isUserDoesNotExistError(error)) {
      logger.warn({ error }, `failed to persist account - user does not exist`)
    } else if (isUnableToConnectToDatabaseError(error)) {
      logger.fatal({ error }, `failed to persist account - unable to connect to database`)
    } else {
      logger.error({ error }, `failed to persist account - generic error`)
    }
  }
}
```

## Log Syntax & Querying

Using AWS CloudWatch insights and other log services, we can construct complex queries that scan through persisted log data in order to:
- Extract metrics
  - Number of accounts created
  - Volume of transactions from account
- Trace requests
  - See all logs from a request that passed through multiple micro-services
  - Build a high level diagram of communication between services
  - Identify all logs related to failed operation/request
- Trigger alerts
  - Alert slack if there are more than 10 errors within the last X amount of time
  - Alert slack if we have not received updates from a 3rd party service for 24 hours

### Keywords

In order to help us achieve the above, we need to construct our logs in a way that is consistent across the entire organization and try to use specific keywords that help us tie each log to a type of action.

- `persisting` - attempting to store data in a persistent storage such as a database, local directory or cache.
- `resolving` - attempting to reach out to a different service to carry out a request
- `resolved` - finished resolving a request
- `processing` - attempting to process an operation
- `processed` - finished processing an operation
- `failed to` - failed to perform an operation

Example:

```ts
logger.info('persisting account to postgres')
```

### Indexing

Cloudwatch and other log services will index parts of a log entry provided that the log entry conforms to a certain format. With cloudwatch, which is our current log service, it indexes any JSON objects specified at the beginning of a log.

For example:

```ts
logger.info({ amount: tx.amount, currency: tx.currency }, 'processed inbound transaction')
```

Now we could use the following query in the Cloudwatch Log Insights platform to extract transaction volume:

```sql
fields @timestamp, @message, amount, currency
| filter @message like /processed inbound transaction/
| filter currency='USD'
| stats avg(amount) * count(*) as volume, count(*) as transactionVolume
```

All logs containing data should make heavy use of this functionality.

### Correlating

When creating log entries, we can specify a correlation id to be indexed by Cloudwatch to help us locate all logs related to a specific operation/request. To do this, at the beginning of each request or operation, generate a random id and then add it the indexed data of *all logs downstream*. When making requests to other micro-services, add the id into the header and extract it when received.

```ts
logger.info({ 'x-correlation-id': "5a9b1b6c" }, 'processed event')
```

Now we can use the following query to the full history of logs related to the request '5a9b1b6c' and even track it across services.

```sql
fields @timestamp, @message
| filter `x-correlation-id`="5a9b1b6c"
| sort @timestamp desc
```

All log entries should have a correlation id and make use of an `orgId` as much as possible. We can use [pino](https://getpino.io/#/) middleware to automatically add this info (see the [cldsvc-sdk pino middleware](https://github.com/IoFinnet/io-core-cldsvc-sdk/blob/691b97e6d598293460f75a4433cb6edd3bb6c66d/src/utils/initLogger.ts#L21).