# Integrating MySQL and PostgreSQL in Laravel with Filament and Leaflet.js



1. Configuring PostgreSQL in Laravel Without Affecting MySQL 

.env File 

```
# MySQL (default, no changes made)
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=carbonfarming
DB_USERNAME=root
DB_PASSWORD=

# PostgreSQL (for GIS only)
DB_PG_CONNECTION=pgsql
DB_PG_HOST=127.0.0.1
DB_PG_PORT=5432
DB_PG_DATABASE=gis_database
DB_PG_USERNAME=postgres
DB_PG_PASSWORD=
```

config/database.php File

```
return [
    'default' => env('DB_CONNECTION', 'mysql'), 

    'connections' => [
        'mysql' => [
            'driver' => 'mysql',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'forge'),
            'username' => env('DB_USERNAME', 'forge'),
            'password' => env('DB_PASSWORD', ''),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix' => '',
            'strict' => true,
            'engine' => null,
        ],

        'pgsql' => [
            'driver' => 'pgsql',
            'host' => env('DB_PG_HOST', '127.0.0.1'),
            'port' => env('DB_PG_PORT', '5432'),
            'database' => env('DB_PG_DATABASE', 'forge'),
            'username' => env('DB_PG_USERNAME', 'forge'),
            'password' => env('DB_PG_PASSWORD', ''),
            'charset' => 'utf8',
            'prefix' => '',
            'schema' => 'public',
            'postgis' => true, // Enable PostGIS
        ],
    ],
];
```

2. Installing Necessary Dependencies 
   Installing Leaflet.js 
   To use Leaflet.js in your Laravel project, install it via npm:
   
   ```
   npm install leaflet
   ```

Installing Laravel Package for Spatial

Install the grimzy/laravel-mysql-spatial package to work with spatial data in Laravel: 

```
composer require grimzy/laravel-mysql-spatial
```

3. Laravel Models for MySQL and PostgreSQL 
   Model Land.php (PostGIS, PostgreSQL) 
   
   ```
   use Grimzy\LaravelMysqlSpatial\Eloquent\SpatialTrait;
   use Illuminate\Database\Eloquent\Model;
   
   class Land extends Model
   {
       use SpatialTrait;
   
       protected $connection = 'pgsql'; // Connects to PostgreSQL
       protected $fillable = ['name'];
       protected $spatialFields = ['area']; // Define the spatial field
   }
   ```

Model Farmer.php (MySQL, remains unchanged)

```
use Illuminate\Database\Eloquent\Model;

class Farmer extends Model
{
    protected $fillable = ['name', 'email']; // Uses MySQL by default
}
```

4. Separate Migration for PostgreSQL
   Creating the Migration for PostgreSQL 

```
Schema::connection('pgsql')->create('lands', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->polygon('area'); // PostGIS field
    $table->timestamps();
});
```

Command to Migrate PostgreSQL

```
php artisan migrate --database=pgsql
```

5. Controller for Data from MySQL and PostgreSQL 
    Controller DashboardController.php 
   
   ```
   use App\Models\Land;
   use App\Models\Farmer;
   
   public function index()
   {
       // Data from MySQL
       $farmers = Farmer::all();
   
       // Data from PostgreSQL
       $lands = Land::all();
   
       return view('dashboard', compact('farmers', 'lands'));
   }
   ```

6. Integrating PostgreSQL in Filament 
    Creating Filament Resource for Land 

```
php artisan make:filament-resource Land
```

Class LandResource.php

```
use Filament\Resources\Resource;
use Filament\Resources\Forms;
use Filament\Resources\Tables;
use App\Models\Land;

class LandResource extends Resource
{
    protected static ?string $model = Land::class;

    public static function form(Forms\Form $form): Forms\Form
    {
        return $form->schema([
            Forms\Components\TextInput::make('name')->required(),
        ]);
    }

    public static function table(Tables\Table $table): Tables\Table
    {
        return $table->columns([
            Tables\Columns\TextColumn::make('name'),
        ]);
    }
}
```

7. Integrating Leaflet.js for PostgreSQL 
   Controller MapController.php 
   
   ```
   use App\Models\Land;
   
   public function showMap()
   {
       $lands = Land::all();
       return view('map', compact('lands'));
   }
   View map.blade.php
   <div id="map" style="height: 500px;"></div>
   
   <script>
       var map = L.map('map').setView([45.0, 25.0], 10);
       L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
   
       var lands = @json($lands);
   
       lands.forEach(function(land) {
           var polygon = L.polygon(land.area.coordinates).addTo(map);
           polygon.bindPopup(land.name);
       });
   </script>
   ```

8. Viewing Land Areas 
    View surfaces.blade.php 
   
   ```
   <div class="container mx-auto">
       <h1 class="text-2xl font-bold">Land Areas</h1>
   
       @if($surfaces->count() > 0)
           <table class="table-auto w-full">
               <thead>
                   <tr>
                       <th class="px-4 py-2">Name</th>
                       <th class="px-4 py-2">Coordinates</th>
                       <th class="px-4 py-2">Area (sqm)</th>
                   </tr>
               </thead>
               <tbody>
                   @foreach($surfaces as $surface)
                       <tr>
                           <td class="border px-4 py-2">{{ $surface->name }}</td>
                           <td class="border px-4 py-2">{{ json_encode($surface->coordinates) }}</td>
                           <td class="border px-4 py-2">{{ $surface->area }}</td>
                       </tr>
                   @endforeach
               </tbody>
           </table>
       @else
           <p>No land areas available.</p>
       @endif
   </div>
   ```

9. Converting KML Files to GeoJSON or CSV 
   Converting KML to GeoJSON 
   
   To convert KML files to GeoJSON, you can use the kml-to-geojson package available on npm. Install it like this:
   
   ```
   npm install kml-to-geojson
   ```

Example Usage

```
const kml = require('kml-to-geojson');
const fs = require('fs');

fs.readFile('path/to/file.kml', (err, data) => {
    if (err) throw err;

    kml(data.toString(), (geojson) => {
        console.log(geojson); // You can save geojson to a file or use it directly
    });
});
```

Converting KML to CSV

To convert KML files to CSV, you can use a custom script or online tools. A simple approach could be to parse KML files using an XML parser and extract the desired data.

10. Integrating QGIS with Laravel 
     Exporting Data from QGIS 
    
    To integrate QGIS with Laravel, you can export data from QGIS in GeoJSON or CSV format, then import it into your Laravel application's database. 
    Open QGIS and load the desired layer. 
    Export the data: 
    Right-click on the desired layer > Export > Export As... 
    Choose GeoJSON or CSV format. 
    Save the file to disk. 
    Importing Data into Laravel 
    
    To import data into Laravel, you can use a migration or a custom script that reads GeoJSON or CSV files and inserts them into the database.
    
    ```
    use Illuminate\Support\Facades\DB;
    
    public function importGeoJSON($filePath)
    {
        $data = json_decode(file_get_contents($filePath), true);
        foreach ($data['features'] as $feature) {
            DB::table('lands')->insert([
                'name' => $feature['properties']['name'],
                'area' => $feature['geometry']['coordinates'], // Ensure the correct format
            ]);
        }
    }
    ```

11. Integrating Google Earth Engine with Laravel 
     Configuring Google Earth Engine 
    
    Create a Google Cloud account and enable the Google Earth Engine API. 
    Obtain the API key from the Google Cloud Console. 
    Using the Google Earth Engine API 
    
    To use Google Earth Engine from Laravel, install the Google API client package:
    
    ```
    composer require google/apiclient
    ```
    
    Example Usage
    
    ```
    use Google\Client;
    
    public function getDataFromGEE()
    {
        $client = new Client();
        $client->setApplicationName('My Laravel App');
        $client->setScopes(['https://www.googleapis.com/auth/earthengine.readonly']);
        $client->setAuthConfig('/path/to/your/credentials.json');
    
        $service = new \Google\Service\Earthengine($client);
    
        // Example request to retrieve data
        $result = $service->projects->get('projects/my_project/datasets/my_dataset');
    
        return $result;
    }
    ```

Creating a Controller to Interact with GEE 
Create a controller that interacts with Google Earth Engine to retrieve and process the desired data: 

```
php artisan make:controller GEEController
```

Method in the Controller

```
use Google\Client;

public function fetchData()
{
    // Authentication and setup
    $client = new Client();
    $client->setApplicationName('My Laravel App');
    $client->setScopes(['https://www.googleapis.com/auth/earthengine.readonly']);
    $client->setAuthConfig('/path/to/your/credentials.json');

    $service = new \Google\Service\Earthengine($client);

    // Extracting data
    $data = $service->projects->listProjects();

    return view('geo_data', ['data' => $data]);
}
```

This documentation covers the essential steps for integrating MySQL and PostgreSQL in Laravel, using Filament for the interface and Leaflet.js for visualizing spatial data. Feel free to add additional details or customize the documentation according to your specific needs!
