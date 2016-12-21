This tutorial will teach you how to build a "Did you mean functinality" and guide you through
the process step by step. For our example we'll use a large world city database consisting of
more than 3 million cities. The list of cities is provided by MaxMind, Inc and can be downloaded
[here](https://www.maxmind.com/en/free-world-cities-database).
Setting up laravel won't be covered here, but you can find plenty tutorials out there on how to
do this.

Lets start with a basic command that will import the list of cities to our database. The database
migration looks as follows:

`php artisan make:model City --migration`
```php
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateCitiesTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('cities', function (Blueprint $table) {
            $table->increments('id');
            $table->string('country');
            $table->string('city');
            $table->string('region');
            $table->float('population');
            $table->double('latitude', 15, 8);
            $table->double('longitude', 15, 8);
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('cities');
    }
}
```  
Add `public $timestamps = false;` to your `City`model since we don't need timestamps.

Notice the column `n_grams`, this will help us later to achieve the did you mean functionality.

Now, lets prepopulate the table. Our dataset is a regular file where each line represents a city.
A simple command will do the job:

`php artisan make:command ImportCities`
```php

<?php

namespace App\Console\Commands;

use App\City;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use TeamTNT\TNTSearch\Indexer\TNTIndexer;

class ImportCities extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'import:cities';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Imports cities from MaxMind';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->tnt = new TNTIndexer;
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->info("Downloading worldcitiespop.txt.gz from MaxMind");

        $gzipedFile  = storage_path().'/worldcitiespop.txt.gz';
        $unZipedFile = storage_path().'/worldcitiespop.txt';

        if (!file_exists($gzipedFile)) {
            file_put_contents($gzipedFile, fopen("http://download.maxmind.com/download/worldcities/worldcitiespop.txt.gz", 'r'));
        }

        $this->info("Unziping worldcitiespop.txt.gz to worldcitiespop.txt");

        $this->line("\n\nInserting cities to database");

        if (!file_exists($unZipedFile)) {
            $this->unzipFile($gzipedFile);
        }

        $cities = fopen(storage_path().'/worldcitiespop.txt', "r");

        $lineNumber = 0;
        $bar = $this->output->createProgressBar(3173959);

        if ($cities) {
            while (!feof($cities)) {
                $line = fgets($cities, 4096);

                if ($lineNumber == 0) {
                    $lineNumber++;
                    continue;
                }
                //we skip the first line since it's the header
                $line = explode(',', $line);
                $this->insertCity($line);
                $lineNumber++;
                $bar->advance();
            }
            fclose($cities);
        }
        $bar->finish();
    }

    public function insertCity($cityArray)
    {
        $city             = new City;
        $city->country    = $cityArray[0];
        $city->city       = utf8_encode($cityArray[2]);
        $city->region     = $cityArray[3];
        $city->population = 0;
        if ($cityArray[4] != "") {
            $city->population = $cityArray[4];
        }
        $city->latitude  = trim($cityArray[5]);
        $city->longitude = trim($cityArray[6]);
        $city->n_grams   = $this->createNGrams($city->city);
        $city->save();

    }

    public function unzipFile($from)
    {

        // Raising this value may increase performance
        $buffer_size   = 4096; // read 4kb at a time
        $out_file_name = str_replace('.gz', '', $from);

        // Open our files (in binary mode)
        $file     = gzopen($from, 'rb');
        $out_file = fopen($out_file_name, 'wb');

        // Keep repeating until the end of the input file
        while (!gzeof($file)) {
            // Read buffer-size bytes
            // Both fwrite and gzread and binary-safe
            fwrite($out_file, gzread($file, $buffer_size));
        }

        // Files are done, close files
        fclose($out_file);
        gzclose($file);
    }

    public function createNGrams($word) 
    {
        return utf8_encode($this->tnt->buildTrigrams($word));
    }
}


Don't forget to register the command in `app\Console\Kernel.php`
```php
<?php

    protected $commands = [
        \App\Console\Commands\ImportCities::class
    ];
 ```

The createNGrams method requires TNTSearch, so before running the command run:

`composer require teamtnt/tntsearch`
     * Run the migrations.
