# Upload Files In Queue

1. **Adjust PHP and Server Configurations**
   PHP Configuration:
   Ensure that your php.ini settings are configured to handle large uploads:

        upload_max_filesize = 256M
        post_max_size = 256M
        max_execution_time = 300

Nginx/Apache Configuration:
For Nginx, you need to set client_max_body_size:

    http {
        client_max_body_size 256M;
    }

For Apache, ensure the LimitRequestBody directive is set appropriately:

    <Directory "/path/to/your/app">
        LimitRequestBody 268435456
    </Directory>

2. Implementing Queued Uploads in Laravel
   Using Laravel Queues:
   Laravel Queues allow you to defer the processing of a time-consuming task. For file uploads, you can upload the file immediately but process it in the background.

Create a Job for Processing:
Generate a new job using the Artisan command:

    php artisan make:job ProcessLargeFile

Inside `ProcessLargeFile.php`, implement the logic to handle the file:

    namespace App\Jobs;
    
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Support\Facades\Storage;
    use Illuminate\Support\Facades\Log;

    class ProcessLargeFile implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
        protected $filePath;
        protected $fileName;
    
        /**
         * Create a new job instance.
         *
         * @return void
         */
        public function __construct($filePath, $fileName)
        {
            $this->filePath = $filePath;
            $this->fileName = $fileName;
        }
    
        /**
         * Execute the job.
         *
         * @return void
         */
        public function handle()
        {
            // Define the permanent storage path
            $destinationPath = 'uploads/permanent/' . $this->fileName;
    
            try {
                // Move the file to the permanent storage location
                if (Storage::exists($this->filePath)) {
                    Storage::move($this->filePath, $destinationPath);
                    Log::info("File moved successfully: {$this->filePath} to {$destinationPath}");
                } else {
                    Log::error("File not found: {$this->filePath}");
                }
            } catch (\Exception $e) {
                Log::error("Failed to move file: {$this->filePath}. Error: " . $e->getMessage());
            }
        }
    }

2. Controller Implementation
   Next, update your controller to handle file uploads and dispatch the job.

`FileUploadController.php`

        namespace App\Http\Controllers;
        
        use Illuminate\Http\Request;
        use App\Jobs\ProcessLargeFile;
        use Illuminate\Support\Facades\Storage;
        use Illuminate\Support\Str;
        
        class FileUploadController extends Controller
        {
            public function upload(Request $request)
            {
                $request->validate([
                'file' => 'required|file|max:256000', // 250MB in KB
                ]);
        
                $file = $request->file('file');
                $fileName = Str::uuid() . '.' . $file->getClientOriginalExtension();
                $filePath = $file->storeAs('uploads/temp', $fileName);
        
                // Dispatch the job
                ProcessLargeFile::dispatch($filePath, $fileName);
        
                return response()->json(['message' => 'File uploaded successfully! Processing in background.']);
            }
        }


3. Queue Configuration
   Ensure your queue system is configured. For local development, you might use the sync driver, but for production, consider using a more robust driver like redis, database, or sqs.

Queue Configuration:
Configure your `.env` file:

    `QUEUE_CONNECTION=database`

Run the migrations to create the necessary database tables:

    php artisan queue:table
    php artisan migrate

Start the queue worker to process jobs in the background:

    php artisan queue:work

