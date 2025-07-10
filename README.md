# Butler - Background Task Management Framework

Butler is a .NET framework for building, executing, and managing background tasks in a distributed environment. It provides a clean, typed API for defining tasks and supports multiple storage backends including PostgreSQL and in-memory storage.

## Key Features

- **Type-safe task definition** - Define tasks with strongly-typed parameters using `ITask<TRequest>`
- **Flexible metadata** - Optional name/description with tag-based categorization and filtering
- **Multiple storage backends** - PostgreSQL, in-memory, and extensible storage providers
- **Progress tracking** - Real-time progress updates with percentage and messages
- **Task scheduling** - Execute tasks immediately or schedule for future execution
- **Recurring jobs** - Advanced scheduling with cron expressions, intervals, and complex patterns
- **Error handling** - Comprehensive error tracking with retry support
- **Statistics** - Built-in execution statistics and monitoring
- **Health monitoring** - Automatic health checks and alerting for job failures
- **Cancellation support** - Graceful task cancellation with `CancellationToken`

## Quick Start

### 1. Install Packages

```bash
dotnet add package Butler.Core
dotnet add package Butler.Execution
dotnet add package Butler.AspNetCore        # For recurring jobs and API
dotnet add package Butler.Storage.PostgreSQL
```

### 2. Define a Task

```csharp
using Butler.Core;

public class SendEmailRequest
{
    public string To { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
}

public class SendEmailTask : TaskBase<SendEmailRequest>
{
    // Name is optional - defaults to class name "SendEmailTask"
    public override string? Name => "Send Email";
    
    // Description is optional
    public override string? Description => "Sends an email to the specified recipient";
    
    // Use tags for categorization instead of Category
    public override string[] Tags => new[] { "communication", "email", "notification" };

    protected override async Task<TaskResult> ExecuteInternalAsync(
        SendEmailRequest request, 
        ITaskContext context, 
        CancellationToken cancellationToken)
    {
        // Report progress
        await context.ReportProgressAsync(25, "Validating email address...");
        
        // Your email sending logic here
        await Task.Delay(1000, cancellationToken);
        
        await context.ReportProgressAsync(100, "Email sent successfully");
        
        return TaskResult.Success();
    }
}
```

### 3. Configure Services

```csharp
using Butler.Execution;
using Butler.Storage.PostgreSQL;
using Butler.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

// Add Butler services with recurring jobs
builder.Services.AddButlerRecurringJobs(options =>
{
    options.Enabled = true;
    options.PollingIntervalSeconds = 30;
});

// Add PostgreSQL storage
builder.Services.AddPostgreSQLTaskStorage(connectionString);

// Register your tasks
builder.Services.AddScoped<SendEmailTask>();

var app = builder.Build();

// Add recurring jobs API endpoints
app.MapControllers();
```

### 4. Execute Tasks

```csharp
public class EmailController : ControllerBase
{
    private readonly ITaskExecutor _taskExecutor;
    private readonly IRecurringJobService _recurringJobService;
    
    public EmailController(ITaskExecutor taskExecutor, IRecurringJobService recurringJobService)
    {
        _taskExecutor = taskExecutor;
        _recurringJobService = recurringJobService;
    }
    
    [HttpPost("send")]
    public async Task<IActionResult> SendEmail([FromBody] SendEmailRequest request)
    {
        // Execute task immediately
        var executionId = await _taskExecutor.ExecuteTaskAsync(
            "Send Email", 
            request, 
            userId: "current-user-id");
        
        return Ok(new { ExecutionId = executionId });
    }
    
    [HttpPost("setup-daily-emails")]
    public async Task<IActionResult> SetupDailyEmails([FromBody] SendEmailRequest request)
    {
        // Create recurring job for daily emails
        var job = new RecurringJobDefinition
        {
            Name = "Daily Email Notifications",
            TaskName = "Send Email",
            Schedule = new JobSchedule
            {
                Type = JobScheduleType.Cron,
                CronExpression = "0 9 * * *", // Daily at 9 AM
                TimeZone = "UTC"
            },
            ParametersJson = JsonSerializer.Serialize(request),
            IsEnabled = true
        };
        
        var createdJob = await _recurringJobService.CreateRecurringJobAsync(job);
        
        return Ok(new { JobId = createdJob.Id, Message = "Daily email job created" });
    }
}
```

## Architecture Overview

### Core Components

- **`ITask<TRequest>`** - Main interface for defining tasks with typed parameters
  - `Name` - Optional, defaults to class name
  - `Description` - Optional description  
  - `Tags` - Array of tags for categorization and filtering
- **`TaskBase<TRequest>`** - Base class providing common functionality and error handling
- **`ITaskExecutor`** - Service for executing tasks (immediate or scheduled)
- **`IRecurringJobService`** - Service for managing recurring jobs and schedules
- **`ITaskStorage`** - Storage abstraction for task definitions and executions
- **`ITaskContext`** - Execution context providing progress reporting and cancellation

### Storage Providers

- **PostgreSQL** - Full-featured storage with persistence and statistics
- **Memory** - In-memory storage for development and testing

### Task Lifecycle

1. **Definition** - Tasks are registered and their metadata is stored
2. **Request** - A task execution is requested with parameters (immediate or recurring)
3. **Scheduling** - Recurring jobs are scheduled based on cron expressions or intervals
4. **Execution** - The task is executed with progress tracking
5. **Completion** - Results are stored and statistics are updated
6. **Monitoring** - Health checks and performance metrics are collected

## Storage Configuration

### PostgreSQL Storage

```csharp
// Basic configuration
services.AddPostgreSQLTaskStorage(connectionString);

// With options
services.AddPostgreSQLTaskStorage(options =>
{
    options.ConnectionString = connectionString;
    options.CommandTimeoutSeconds = 30;
    options.RetentionDays = 30;
    options.MaxProgressUpdatesPerExecution = 100;
});

// With automatic migration
services.AddPostgreSQLTaskStorageWithMigration(connectionString);
```

### Memory Storage

```csharp
services.AddMemoryTaskStorage();
```

## Task Development Patterns

### Using TaskBase<TRequest>

```csharp
public class DataProcessingTask : TaskBase<DataProcessingRequest>
{
    // Name defaults to "DataProcessingTask" if not specified
    public override string? Name => "Process Data";
    public override string? Description => "Processes large datasets";
    public override string[] Tags => new[] { "data", "processing", "batch" };

    protected override async Task<TaskResult> ExecuteInternalAsync(
        DataProcessingRequest request, 
        ITaskContext context, 
        CancellationToken cancellationToken)
    {
        var totalItems = request.Items.Count;
        
        for (int i = 0; i < totalItems; i++)
        {
            // Check for cancellation
            cancellationToken.ThrowIfCancellationRequested();
            
            // Process item
            await ProcessItem(request.Items[i]);
            
            // Report progress
            var progress = (i + 1) * 100 / totalItems;
            await context.ReportProgressAsync(progress, $"Processed {i + 1}/{totalItems} items");
        }
        
        return TaskResult.Success(new { ProcessedCount = totalItems });
    }
}
```

### Implementing ITask<TRequest> Directly

```csharp
public class CustomTask : ITask<CustomRequest>
{
    // Name defaults to "CustomTask" if null
    public string? Name => null; 
    public string? Description => null;
    public string[] Tags => new[] { "custom", "example" };

    public async Task<TaskResult> ExecuteAsync(
        CustomRequest request, 
        ITaskContext context, 
        CancellationToken cancellationToken)
    {
        try
        {
            // Your task logic here
            return TaskResult.Success();
        }
        catch (Exception ex)
        {
            return TaskResult.Failure(ex.Message);
        }
    }
}
```

### Minimal Task Example

```csharp
// Most minimal task - uses class name and no description
public class SimpleTask : TaskBase<string>
{
    // All properties optional - will use class name "SimpleTask"
    public override string[] Tags => new[] { "simple" };

    protected override async Task<TaskResult> ExecuteInternalAsync(
        string request, 
        ITaskContext context, 
        CancellationToken cancellationToken)
    {
        await Task.Delay(100, cancellationToken);
        return TaskResult.Success($"Processed: {request}");
    }
}
```

## Project Structure

```
src/Infrastructure/Butler/
├── src/
│   ├── Butler.Core/              # Core interfaces and models
│   │   ├── ITask.cs             # Main task interface
│   │   ├── TaskBase.cs          # Base task implementation
│   │   ├── Models/              # Task models (TaskDefinition, TaskExecution, etc.)
│   │   └── Storage/             # Storage abstractions
│   ├── Butler.Execution/        # Task execution engine
│   │   ├── ITaskExecutor.cs     # Task execution interface
│   │   └── TaskExecutor.cs      # Task execution implementation
│   ├── Butler.Storage.PostgreSQL/ # PostgreSQL storage provider
│   └── Butler.Storage.Memory/   # In-memory storage provider
└── README.md
```

## Recurring Jobs

Butler provides powerful recurring job capabilities for enterprise-grade background task scheduling:

### Quick Recurring Job Example

```csharp
// Create a daily backup job
var backupJob = new RecurringJobDefinition
{
    Name = "Daily Database Backup",
    TaskName = "Data Backup",
    Schedule = new JobSchedule
    {
        Type = JobScheduleType.Cron,
        CronExpression = "0 2 * * *", // Daily at 2 AM
        TimeZone = "UTC"
    },
    ParametersJson = JsonSerializer.Serialize(new DataBackupRequest
    {
        BackupPath = "/backups/daily",
        IncludeUserData = true,
        IncludeSystemData = true
    }),
    IsEnabled = true,
    MaxRetryAttempts = 3
};

await recurringJobService.CreateRecurringJobAsync(backupJob);
```

### Key Recurring Jobs Features

- **Flexible Scheduling**: Cron expressions, intervals, and one-time schedules
- **Time Zone Support**: Schedule jobs across different time zones
- **Advanced Configuration**: Retry policies, timeouts, and concurrency control
- **Health Monitoring**: Automatic failure detection and alerting
- **Comprehensive API**: Full REST API for job management
- **Statistics**: Detailed performance metrics and reporting

## Next Steps

### Documentation
- **[Recurring Jobs Guide](docs/recurring-jobs-README.md)** - Complete guide to recurring jobs
- **[API Reference](docs/api-reference.md)** - Complete API documentation
- **[Configuration Guide](docs/configuration-guide.md)** - Advanced configuration options
- **[Migration Guide](docs/migration-guide.md)** - Migrate from other job systems

### Examples
- **[Basic Recurring Jobs](examples/09-BasicRecurringJobs/)** - Simple recurring job examples
- **[Advanced Scheduling](examples/10-AdvancedScheduling/)** - Complex cron patterns and time zones
- **[Job Management](examples/11-JobManagement/)** - Lifecycle management and bulk operations
- **[Monitoring & Statistics](examples/12-MonitoringAndStatistics/)** - Performance tracking and health monitoring
- **[All Examples](examples/)** - Complete collection of working examples
