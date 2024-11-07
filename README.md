# 2024-11-07

## This documents is for fixing the issue of laravel file upload to fintel backend that is hindered and limited by Cloudflare limit of 100s timeouts.

### The solution is to intergrate the file upload with MQTT with laravel queue and mqtt.

### Code 

1 - [Laravel Part](#laravel-part)
2 - [Simulation using docker compose and AWS lambda function - lambda will simulate the ML processing](#simulation-using-docker-compose-and-aws-lambda-function---lambda-will-simulate-the-ml-processing)

```php
// Controller implementation
namespace App\Http\Controllers;

use App\Services\FileUploadService;
use Illuminate\Http\Request;

class FileUploadController extends Controller
{
    private FileUploadService $uploadService;

    public function __construct(FileUploadService $uploadService)
    {
        $this->uploadService = $uploadService;
    }

    public function upload(Request $request)
    {
        $request->validate([
            'file' => 'required|file|max:' . config('upload.max_size')
        ]);

        try {
            $result = $this->uploadService->handleUpload($request->file('file'));
            return response()->json($result);
        } catch (Exception $e) {
            return response()->json([
                'status' => 'error',
                'message' => $e->getMessage()
            ], 400);
        }
    }
}

// ML Processing Job
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessMLJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    private string $filePath;

    public function __construct(string $filePath)
    {
        $this->filePath = $filePath;
    }

    public function handle(): void
    {
        // Implement ML processing logic here
        // This will be processed by your queue worker
    }
}
```

### Explanation of the code

1. **FileUploadService**: This service handles the file upload process end-to-end. It validates the file, generates a unique filename, uploads the file to S3 with proper headers, publishes a message to MQTT, and queues the ML processing job.

2. **FileUploadController**: This controller handles the incoming file upload request from the SPA. It validates the request, calls the `handleUpload` method of the `FileUploadService`, and returns the result as a JSON response.

3. **ProcessMLJob**: This job implements the ML processing logic. It will be processed by the queue worker.

### Simulation using docker compose and AWS lambda function - lambda will simulate the ML processing

#### Lambda function code

```ts
import { APIGatewayProxyHandler } from "aws-lambda";

export const handler: APIGatewayProxyHandler = async (event, context) => {
  // Simulate a delay of 100 seconds
  await new Promise((resolve) => setTimeout(resolve, 100000));

  // Return a 200 OK response
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: "Success",
    }),
  };
};
```

#### Docker compose file
To set up a complete environment using Docker Compose for a Laravel application integrated with an MQTT broker and an AWS Lambda function simulation, follow the steps below. This guide will include a docker-compose.yml file and instructions for setting up each component.
Docker Compose File
Here's a basic docker-compose.yml file to set up a Laravel application, an MQTT broker, and a local simulation of the AWS Lambda function:

```yaml
version: '3.8'

services:
  laravel-app:
    image: laravel:latest
    container_name: laravel_app
    ports:
      - "8000:80"
    volumes:
      - .:/var/www/html
    environment:
      - APP_ENV=local
      - APP_DEBUG=true
      - APP_KEY=base64:your_app_key_here
      - AWS_ACCESS_KEY_ID=minioadmin
      - AWS_SECRET_ACCESS_KEY=minioadmin
      - AWS_DEFAULT_REGION=us-east-1
      - AWS_BUCKET=laravel-bucket
      - AWS_ENDPOINT=http://minio:9000
    depends_on:
      - mqtt-broker
      - minio

  mqtt-broker:
    image: eclipse-mosquitto:latest
    container_name: mqtt_broker
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log

  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - ./minio/data:/data
    command: server /data --console-address ":9001"

  lambda-simulator:
    image: node:14
    container_name: lambda_simulator
    volumes:
      - ./lambda:/usr/src/app
    working_dir: /usr/src/app
    command: ["node", "index.js"]
    depends_on:
      - laravel-app

```

### MQTT Broker Setup

Configuration: You can customize the Mosquitto MQTT broker by adding configuration files in the ./mosquitto/config directory. The default configuration should work for basic setups.

### AWS Lambda Function Simulation
1. Create a Directory for Lambda: Create a directory named lambda in the same location as your docker-compose.yml.

2. Add Lambda Function Code: Inside the lambda directory, create an index.js file with the following content:

```ts
   const http = require('http');

   const server = http.createServer((req, res) => {
     if (req.method === 'POST') {
       let body = '';
       req.on('data', chunk => {
         body += chunk.toString();
       });
       req.on('end', () => {
         console.log('Received:', body);
         setTimeout(() => {
           res.writeHead(200, { 'Content-Type': 'application/json' });
           res.end(JSON.stringify({ message: 'Success' }));
         }, 100000); // Simulate 100 seconds delay
       });
     } else {
       res.writeHead(405, { 'Content-Type': 'text/plain' });
       res.end('Method Not Allowed');
     }
   });

   server.listen(3000, () => {
     console.log('Lambda simulator running on port 3000');
   });
```

3. **Run the Docker Compose**: 
Start your services using Docker Compose. 
```bash 
docker-compose up
```

### Additional Notes

- **Network Configuration**: Ensure that your Laravel application can communicate with the MQTT broker and the Lambda simulator. Docker Compose should handle this by default, but you may need to adjust network settings if you encounter issues.
- **AWS Lambda**: This setup simulates an AWS Lambda function locally. For production, you would deploy your function to AWS and configure your Laravel application to invoke it via AWS SDK or API Gateway.

This setup provides a local development environment to test the integration of Laravel with MQTT and a simulated AWS Lambda function. Adjust configurations as needed for your specific use case.

# Laravel Part
### Laravel Application Setup

1. Create a Laravel Project: If you haven't already, create a Laravel project in the directory where your docker-compose.yml is located.

```bash
composer create-project laravel/laravel laravel-app
``` 

2. **Configure Environment Variables**: Update the `.env` file in your Laravel project to match the environment variables specified in the `docker-compose.yml`.


3. **Install MQTT Client**: Add the MQTT client package to your Laravel project.

   ```bash
   composer require php-mqtt/client
   ```

Set up the mqtt configuration in the .env file as follows:
since we are using mosquitto as the mqtt broker, the host is localhost, port is 1883 and client id is laravel-app.
```
MQTT_HOST=localhost
MQTT_PORT=1883
MQTT_CLIENT_ID=laravel-app
```

4. Set Up S3 and MQTT Configurations: Ensure your config/filesystems.php and config/mqtt.php are properly configured with your S3 and MQTT broker details.
 -1 Laravel controller for file upload
 -2 Laravel service for file upload
 -3 Laravel job for ML processing


#### Laravel Code controller for file upload

```php
<?php 

namespace App\Http\Controllers;

class FileUploadController extends Controller
{
    private FileUploadService $uploadService;

    public function __construct(FileUploadService $uploadService)
    {
        $this->uploadService = $uploadService;
    }

    public function upload(Request $request)
    {
        try {
            $request->validate([
                'file' => 'required|file|max:' . config('upload.max_size')
        ]);

        // Call the handleUpload method of the FileUploadService
        $result = $this->uploadService->handleUpload($request->file('file'));

            return response()->json($result);
        } catch (Exception $e) {
            return response()->json([
                'status' => 'error',
                'message' => 'Error uploading file: ' . $e->getMessage()
            ], 500);
        }
    }
}
```
### Example of S3 file handler 

The code below is complete example of how to use file handler in laravel to upload, download, delete and list files in s3 bucket. To complete the file handling we will record the file path, file mime type, timestamp, file name , file size and uploader in the database and s3 metadata.
##### Migration for file database table

```php 
// database/migrations/xxxx_xx_xx_xxxxxx_create_files_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateFilesTable extends Migration
{
    public function up()
    {
        Schema::create('files', function (Blueprint $table) {
            $table->id();
            $table->string('path')->unique();
            $table->string('mime_type');
            $table->string('file_name');
            $table->unsignedBigInteger('file_size');
            $table->foreignId('uploader_id')->constrained('users');
            $table->timestamp('uploaded_at');
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('files');
    }
}
``` 

##### File handler service in laravel
```php
namespace App\Services;

use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Storage;
use App\Models\File;
use App\Interfaces\FileHandlerInterface;
use Illuminate\Http\UploadedFile;
use App\Jobs\ProcessMLJob;
use App\Services\MqttService;

class FileHandlerService implements FileHandlerInterface
{
    protected MqttService $mqttService;
    private array $allowedMimes = ['image/jpeg', 'image/png', 'application/pdf'];
    private string $bucket;

    public function __construct(MqttService $mqttService)
    {
        $this->mqttService = $mqttService;
        $this->bucket = config('filesystems.disks.s3.bucket');
    }

    public function uploadFile(UploadedFile $file): string
    {
        $this->validateFile($file);
        $filePath = $this->generateUniqueFilePath($file);

        // Upload file to S3
        Storage::disk('s3')->put($filePath, file_get_contents($file), [
            'visibility' => 'public',
            'Metadata' => $this->getMetadata($file)
        ]);

        // Record metadata in database
        $this->recordFileMetadata($file, $filePath);

       // Dispatch MQTT publish job
        PublishMqttMessage::dispatch('file-uploads/new', [
            'file_path' => $filePath,
            'mime_type' => $file->getMimeType(),
            'timestamp' => now()->timestamp
        ])->onQueue('mqtt-processing');

        // Dispatch ML processing job
        ProcessMLJob::dispatch($filePath)->onQueue('ml-processing')->delay(now()->addSeconds(5));

        return $filePath;
    }

    public function downloadFile(string $filePath): string
    {
        return Storage::disk('s3')->temporaryUrl($filePath, now()->addMinutes(30));
    }

    public function deleteFile(string $filePath): void
    {
        Storage::disk('s3')->delete($filePath);
        File::where('path', $filePath)->delete();
    }

    public function listFiles(): Collection
    {
        $files = Storage::disk('s3')->allFiles('files');
        return collect($files)->map(fn($filePath) => File::where('path', $filePath)->first());
    }

    private function recordFileMetadata(UploadedFile $file, string $filePath): void
    {
        File::create([
            'path' => $filePath,
            'mime_type' => $file->getClientMimeType(),
            'file_name' => $file->getClientOriginalName(),
            'file_size' => $file->getSize(),
            'uploader_id' => auth()->id(),
            'uploaded_at' => now(),
        ]);
    }

    private function getMetadata(UploadedFile $file): array
    {
        return [
            'mime_type' => $file->getClientMimeType(),
            'file_size' => $file->getSize(),
            'file_name' => $file->getClientOriginalName(),
            'uploaded_at' => now()->toDateTimeString(),
        ];
    }

    private function validateFile(UploadedFile $file): void
    {
        if (!in_array($file->getMimeType(), $this->allowedMimes)) {
            throw new \Exception('Invalid file type');
        }

        if ($file->getSize() > config('upload.max_size')) {
            throw new \Exception('File too large');
        }
    }

    private function generateUniqueFilePath(UploadedFile $file): string
    {
        return 'files/' . time() . '_' . uniqid() . '.' . $file->getClientOriginalExtension();
    }
}


```

##### MQTT Service in laravel

```php
namespace App\Services;

use PhpMqtt\Client\MqttClient;
use PhpMqtt\Client\ConnectionSettings;
use Illuminate\Support\Facades\Log;

class MqttService
{
    private MqttClient $client;

    public function __construct()
    {
        $this->client = new MqttClient(
            config('mqtt.host'),
            config('mqtt.port'),
            config('mqtt.client_id')
        );
    }

    public function publish(string $topic, array $data): void
    {
        try {
            $this->connect();
            $this->client->publish($topic, json_encode($data), 1);
            $this->client->disconnect();
        } catch (\Exception $e) {
            Log::error('MQTT publish failed: ' . $e->getMessage());
        }
    }

    public function listen(string $topic, callable $callback): void
    {
        try {
            $this->connect();
            $this->client->subscribe($topic, $callback, 1);
            $this->client->loop(true);
        } catch (\Exception $e) {
            Log::error('MQTT listener failed: ' . $e->getMessage());
        }
    }

    private function connect(): void
    {
        $settings = (new ConnectionSettings)
            ->setKeepAlive(60)
            ->setConnectTimeout(5)
            ->setReconnectPeriod(1000);

        $this->client->connect($settings);
    }
}

```


##### File handler interface in laravel
The file handler interface is used to define the methods that the file handler service must implement and also structure of metadata that will be recorded in the database.
```php

namespace App\Interfaces;

interface FileHandlerInterface
{
    public function uploadFile(UploadedFile $file): string;
    public function downloadFile(string $filePath): string;
    public function deleteFile(string $filePath): void;
    public function listFiles(): Collection;

    private function recordFileMetadata(UploadedFile $file, string $filePath): void;

    // metadata structure
    private function getMetadata(UploadedFile $file): array;
}
```

####  Create a new job that handles the publishing of MQTT messages for ML processing:

```bash
php artisan make:job PublishToMQTT
```

#### Code for the job

```php

// app/Jobs/PublishMqttMessage.php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Services\MqttService;

class PublishMqttMessage implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected string $topic;
    protected array $messageData;

    /**
     * Create a new job instance.
     *
     * @param string $topic
     * @param array $messageData
     */
    public function __construct(string $topic, array $messageData)
    {
        $this->topic = $topic;
        $this->messageData = $messageData;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle(MqttService $mqttService)
    {
        // Publish the message to the specified MQTT topic
        $mqttService->publish($this->topic, $this->messageData);
    }
}

```

#### Code for ML processing job using queue

```bash
php artisan make:job ProcessMLJob
```

```php
// app/Jobs/ProcessMLJob.php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class ProcessMLJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected string $filePath;

    /**
     * Create a new job instance.
     *
     * @param string $filePath
     */
    public function __construct(string $filePath)
    {
        $this->filePath = $filePath;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        Log::info('Processing file for ML: ' . $this->filePath);
        // Implement ML processing logic here
    }
}

```
Run the queue worker to pick up jobs from both mqtt-processing and ml-processing queues: 

```bash
php artisan queue:work --queue=ml-processing,mqtt-processing
```

