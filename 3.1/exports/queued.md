# Queued

[[toc]]

In case you are working with a lot of data, it might be wise to queue the entire process. 

Given we have the following export class:

```php
namespace App\Exports;

use App\Invoice;
use Maatwebsite\Excel\Concerns\Exportable;
use Maatwebsite\Excel\Concerns\FromQuery;

class InvoicesExport implements FromQuery
{
    use Exportable;

    public function query()
    {
        return Invoice::query();
    }
}
```

It's as easy as calling `->queue()` now.

```php
(new InvoicesExport)->queue('invoices.xlsx');

return back()->withSuccess('Export started!');
```

Behind the scenes the query will be chunked and multiple jobs will be chained. These jobs will be executed in the correct order,
and will only execute if none of the previous have failed. 

## Implicit Export queueing

You can also mark an export implicitly as a queued export. You can do this by using Laravel's `ShouldQueue` contract.

```php
namespace App\Exports;

use App\Invoice;
use Illuminate\Contracts\Queue\ShouldQueue;
use Maatwebsite\Excel\Concerns\Exportable;
use Maatwebsite\Excel\Concerns\FromQuery;

class InvoicesExport implements FromQuery, ShouldQueue
{
    use Exportable;

    public function query()
    {
        return Invoice::query();
    }
}
```

In your controller you can now call the normal `->store()` method. 
Based on the presence of the `ShouldQueue` contract, the export will be queued.

```php
(new InvoicesExport)->store('invoices.xlsx');
```

## Appending jobs

The `queue()` method returns an instance of Laravel's `PendingDispatch`. This means you can chain extra jobs that will be added to the end of the queue and only executed if all export jobs are correctly executed.

```php
(new InvoicesExport)->queue('invoices.xlsx')->chain([
    new NotifyUserOfCompletedExport(request()->user()),
]);
```

```php
namespace App\Jobs;

use App\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\SerializesModels;

class NotifyUserOfCompletedExport implements ShouldQueue
{
    use Queueable, SerializesModels;
    
    public $user;
    
    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function handle()
    {
        $this->user->notify(new ExportReady());
    }
}
```

## Custom queues

Because `PendingDispatch` is returned, we can also change the queue that should be used.

```php
(new InvoicesExport)->queue('invoices.xlsx')->allOnQueue('exports');
```

## Multi-server setup

If you are dealing with a multi-server setup (using e.g. a loadbalancer), you might want to make sure the temporary file that is used to store each chunk of data on, is the same for each job. You can achieve this by configuring a remote temporary file in the config.

In `config/excel.php`

```php
'temporary_files' => [
    'remote_disk' => 's3',
],
```

## Custom Query Size
Queued exportables are processed in chunks; each chunk being a job pushed to the queue by the `QueuedWriter`.
In case of exportables that implement the [FromQuery](/3.1/exports/from-query.html) concern, the number of jobs is calculated by dividing the `$query->count()` by the chunk size.

### When to use
Depending on the implementation of the `query()` method (e.g. when using a `groupBy` clause), the calculation mentioned before might not be correct.

If this is the case, you should use the `WithCustomQuerySize` concern to provide a custom calculation of the query size.
