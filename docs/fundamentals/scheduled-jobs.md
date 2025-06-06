---
sidebar_label: Scheduled Jobs
---

# Scheduled Jobs

Many enterprise applications require that certain actions are performed in the background and triggered on a set schedule (e.g. every night, every week, or even every few minutes). We call these **Scheduled Jobs** and build on top of Shesha's support for [Background Jobs](/docs/fundamentals/background-jobs). Such operations may include:

- Sending of bulk notifications such as reminders
- Data update and clean-up after a certain timeframe (e.g. looking for expired shopping carts, data synchronisation)
- Monitoring (e.g. monitoring system availability or appearance of a file) 

# Important Design Principles
To make your solution robust it is important that any Scheduled Jobs follow the following design principles:

- **Make your job fault-tolerant** - The best way to do this is to implement your job logic as an <a href="https://stackoverflow.com/questions/1077412/what-is-an-idempotent-operation" target="_blank">idempotent operation</a>, which means that it should be possible to run your job multiple times without leaving the system in an inconsistent state. For example, if your job failed to run, or was interrupted (e.g. due to hardware or network failure), the admin should be able to simply rerun your job without fear of corrupting system data or performing duplicate operations. 
- **Commit your changes regularly and atomically** - Changes to data that your job may perform should be committed to the database <a href="https://en.wikipedia.org/wiki/Atomicity_(database_systems)" target="_blank">atomically</a> and regularly. This is important especially for long-running jobs, so that, should the job get interupted and get restarted, it does not have to restart all operations from scratch. Obviously, any updates should be made at logical points where it would not leave the system in an inconsistent state.
- **Log progress regularly, but not too often** - It is important to provide visibility of the progress of background jobs, especially for long-running ones. Use the `Log` method to provide feedback on the progress of your job. Note that this does add overhead and may impact performance so you may want to batch operations before logging feedback (e.g. log progress every 100 notifications, rather than on every notification).
- **Think of performance** - Many scheduled jobs are used to perform bulk operations and go through large amounts of data. NHibernate adds a lot of overhead when processing large number of records and may introduce an unacceptable overhead. In such cases consider <a href="https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/retrieving-and-modifying-data" target="_blank">using ADO.NET directly</a> rather than going through NHibernate as performance will be much better. 

# Implementing a Scheduled Job

To implement a Scheduled Job simply create a new class that inherits from `ScheduledJobBase` and override the `DoExecuteAsync` method.

* There is no need to wrap the `DoTask` logic within a Try Catch statement as the base class already implements this and any unhandled exceptions will be caught and logged.
* The schedule on which the ScheduledJob is to be executed is specified via the `cronString` property of the class `ScheduledJob` attribute. To help create a valid cronstring you can use a <a href="http://www.cronmaker.com" target="_blank">cronstring editor such as this one</a>.

## Bulk Notifications Background Jobs Sample
One of the most types of background jobs are those to send bulk notifications of some sort.

### Example
```cs
/// <summary>
/// Sends reminders of appointments the day before the appointments.
/// </summary>
[ScheduledJob("9bc54591-55fb-4e2e-91b6-199cb9c187d0",
    startupMode: StartUpMode.Automatic, 
    cronString: "0 2 * * *",  // Executes everyday at 2am
    description: "Sends reminders of appointments the day before the appointments.")]
public class SendAppointmentReminderNotificationJob : ScheduledJobBase, ITransientDependency
{
    // Sql to retrieve the list of valid Appointments scheduled for tomorrow for which a Reminder notification has not yet been sent according to the audit log in AbpTenantNotifications
    private const string SQL_SELECT_REMINDERS_TO_SEND = @"SELECT Id FROM Health_Appointments WHERE 
                                            IsDeleted = 0
                                        AND StatusLkp = 3 /*Booked*/
                                        AND [AppointmentTime] >= @fromDate AND [AppointmentTime] < @toDate
                                        AND Id NOT IN (SELECT EntityId FROM AbpTenantNotifications WHERE NotificationName = @templateId)
                                ";

    public SendAppointmentReminderNotificationJob()
    {

    }

    /// <summary>
    /// Implements the logic to be executed on specified schedule.
    /// </summary>
    public override async Task DoExecuteAsync(CancellationToken cancellationToken)
    {
        var bookingNotificationSender = IocManager.Resolve<IBookingNotificationSender>();
        var appointmentsRepo = IocManager.Resolve<IRepository<CdmAppointment, Guid>>();

        Log.Info("Started...");
       
        var numProcessed = 0;
        using (var session = IocManager.Resolve<ISessionFactory>().OpenSession())
        {
            // Retrieving list of notifications to be sent as a ADO.NET DataReader for improved performance as volumes may be significant
            DbDataReader reader = GetReaderForAllAppointmentsStillToNotify(session, DateTime.Now.Date.AddDays(1), DateTime.Now.Date.AddDays(2), NotificationTemplateIds.AppointmentReminder);

            while (reader.Read())
            {
                // Reads through the full results set record by record
                var appointmentId = reader.GetGuid(0);

                try
                {
                    var appointment = appointmentsRepo.Get(appointmentId);
                    await bookingNotificationSender.NotifyAppointmentReminderAsync(appointment);
                    JobStatistics.NumSucceeded++;
                }
                catch (Exception ex)
                {
                    Log.Error($"Failed to send message for appointment Id:{appointmentId}", ex);
                    JobStatistics.NumErrors++;
                }
                finally
                {
                    numProcessed++;
                    if (numProcessed % 100 == 0)    // Only logging every 100 messages to reduce overhead
                        Log.Info($"{numProcessed} appointments have been processed.");
                }
            }
        }

        Log.Info($"All appointments have been processed - Sent: {JobStatistics.NumSucceeded} | Failed: {JobStatistics.NumErrors} | Skipped: {JobStatistics.NumSkipped}");
    }


    private static DbDataReader GetReaderForAllAppointmentsStillToNotify(ISession session, DateTime appointmentsFrom, DateTime appointmentsTo, Guid templateId)
    {
        var connection = session.Connection;
        var command = connection.CreateCommand();
        command.CommandText = SQL_SELECT_REMINDERS_TO_SEND;
        AddParameter(command, "@fromDate", appointmentsFrom);
        AddParameter(command, "@toDate", appointmentsTo);
        AddParameter(command, "@templateId", templateId);

        var reader = command.ExecuteReader();
        return reader;
    }

    private static void AddParameter(DbCommand command, string paramName, object paramValue)
    {
        var parameter = command.CreateParameter();
        parameter.ParameterName = paramName;
        parameter.Value = paramValue;
        command.Parameters.Add(parameter);
    }

}
```

## Responding to ScheduledJob Events
The `ScheduledJobBase` class exposes three events, namely the `OnSuccess`, `OnFail` and `OnLog` events. You can override these events inside the scheduled job, in order to use them. 

```Cs
[ScheduledJob("1FF7882E-0A3B-4F88-A8B3-C3C20AFAFBDE", StartUpMode.Manual)]
public class MyJob: ScheduledJobBase, ITransientDependency
{
    public override async Task DoExecuteAsync(CancellationToken cancellationToken)
    {
        // DO something great here
    }

    public override async Task OnSuccess()
    {
        // Do something when the job has succeeded
        // Maybe send a notification?
    }

    public override async Task OnFail(Exception ex)
    {
        // Do something when the job fails to execute
        // Maybe email the admin user?
    }

    public override void OnLog(object sender, ScheduledJobOnLogEventArgs e)
    {
        // Do something when you log something inside the DoExecuteAsync method
    }
}
```

# Administering ScheduledJobs :TODO

# Viewing Job Progress and Logs

## Job Progress
By default, the `ScheduledJobBase` class inherits from a generic class of the same name, and includes `JobStatistic` property, which is of type `ScheduledJobStatistic`. This property exposes the following properties: `NumSucceeded`, `NumSkipped` and `NumErrors`. 

You can also implement a custom statistic class, but **it should inherit from `ScheduledJobStatistic`**. The following is how you can implement a job with custom statistics:
```Cs
public class MyJobStats: ScheduledJobStatistic
{
     public int TotalProcessedRecords { get; set; }
}


[ScheduledJob("1FF7882E-0A3B-4F88-A8B3-C3C20AFAFBDE", StartUpMode.Manual)]
public class MyJob: ScheduledJobBase<MyJobStats>, ITransientDependency
{
    public override async Task DoExecuteAsync(CancellationToken cancellationToken)
    {
        for (int i=0;i<100;i++)
        { 
           // The JobStatistics property here is of type MyJobStats so it has all the properties defined in the class
           JobStatistics.TotalProcessedRecords++;
        }

    }
}
```
 The Job Statistics can be viewed from the admin portal. After starting the job from the admin portal navigate to the job details page, and click on the recent execution from the data table. You should see the following:

![image](https://user-images.githubusercontent.com/85956374/222988093-0c14e798-c7be-4212-913a-f3619c00d82c.png)

![image](https://user-images.githubusercontent.com/85956374/222988810-8836bb7e-937b-4f50-8a36-4feeb4f9cf7b.png)

## Logging methods
There are two methods which can be used to save log files for any scheduled job. 

1. File system (logs appear under `~/App_Data` folder)
2. Stored files (logs can be configured to persist in Azure storage, or to use `~/App_Data` folder)

To change the logging method, do the following:

```Cs
[ScheduledJob("1FF7882E-0A3B-4F88-A8B3-C3C20AFAFBDE", StartUpMode.Manual, LogMode = LogMode.StoredFile, LogFolder = "ThisJob/Awesome")]
public class TestJob: ScheduledJobBase, ITransientDependency
{

}
```

The `LogMode` property is used to select how the logs are persisted, while the `LogFolder` specifies the folder in which the logs are stored. This could be either inside `App_Data` folder or on Azure ( _Currently not sure about how the permissions on Azure Storage are structured, so not sure if the folder will be created on Azure Storage. **TODO: look into this**_ ).

## Viewing logs
All background jobs should log their progress and any potential errors to a log file using the `Log` method. There are a couple of ways to view Job execution logs as detailed below. 


## Accessing from the UI
### TODO - functionality to allow administrator to view execution logs from the front-end still needs to be built out

## Accessing from the Server
All progress of background jobs gets logged by default to the following location: `\App_Data\jobs\{jobname}\{Executionrun log file}.log`

If you have access to the Azure App Service via the Azure portal you can view the log files using the App Service Editor as illustrated below:

![image](https://user-images.githubusercontent.com/85956374/222988833-c1b6d4a6-6d62-4d46-9143-3fc5c20940cb.png)

Then navigate to the `App_Data\jobs\{Job name}` folder to view the various log files:

![image](https://user-images.githubusercontent.com/85956374/222988841-a457472f-ea61-41d5-9a60-3cb39b38acf5.png)

HangFire dashboard:TODO
