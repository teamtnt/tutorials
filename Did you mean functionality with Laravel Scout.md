![Imgur](http://i.imgur.com/aEmSYNZ.jpg)

In this tutorial, we'll show you how to create "Did you mean" functionality fast and easy.
Data we'll use will be a large city database consisting of more than 3 million cities. The idea is to show the correct city name in case your users misspell them. 
The list of cities comes from MaxMind, Inc and you can find the list
[here](https://www.maxmind.com/en/free-world-cities-database). 
To understand what we are building take a look at the [demo page](http://cities.tnt.studio/).
Setting up Laravel isn't covered, but you can find plenty of tutorials on how to do this. 
Our project will depend on Laravel Scout and TNTSearch so lets install those dependencies:

`composer require teamtnt/laravel-scout-tntsearch-driver`

Add this to your providers array in `config\app.php`

```php
'providers' => [
    /*
     * Package Service Providers...
     */
    Laravel\Scout\ScoutServiceProvider::class,
    TeamTNT\Scout\TNTSearchScoutServiceProvider::class,
]
```

After that publish the Scout config file like:

`php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"`

And in your `.env, set TNTSearch as the default driver:

`SCOUT_DRIVER=tntsearch`

In `config/scout.php` set the `storage_path`

```php
'tntsearch' => [
    'storage'  => storage_path(),
],
```

Make sure this directory is writable.
Lets start with a basic command that will download and import the list of cities to our database. 
The database migration and the model look as follows:

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

Add `public $timestamps = false;` to your `City` model since we don't need timestamps.
Notice the column `n_grams`, this will help us later to achieve the did you mean functionality.
Now, lets populate the table. Our dataset is a regular file where each line represents a city.
A simple command will do the job:

`php artisan make:command ImportCities`

```php
<?php
namespace App\Console\Commands;
use App\City;
use Illuminate\Console\Command;
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
        $bar        = $this->output->createProgressBar(3173959);
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
        //we enter only cities wich have a population set
        if ($cityArray[4] < 1) {
            return;
        }
        $city             = new City;
        $city->country    = $cityArray[0];
        $city->city       = utf8_encode($cityArray[2]);
        $city->region     = $cityArray[3];
        $city->population = $cityArray[4];
        $city->latitude   = trim($cityArray[5]);
        $city->longitude  = trim($cityArray[6]);
        $city->n_grams    = $this->createNGrams($city->city);
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
```

Don't forget to register the command in `app\Console\Kernel.php`

```php
<?php

    protected $commands = [
        \App\Console\Commands\ImportCities::class
    ];
 ```

The command will automatically download the file from Maxmind, unzip it and import the
cities to our database. 
We'll only import cities that have a population greater than 0.
Once we have the cities in our database, let's create the inverted index that scout will
consume.

`php artisan tntsearch:import App\City`

Ok, now we can do a basic search for a city:

`App\City::serach('Berlin')->get()`

Although this works fine, it's still not able to fetch a city if you make a typo.
Let's take care of this. 

`php artisan make:command CreateCityTrigrams`

```php
<?php
namespace App\Console\Commands;
use Illuminate\Console\Command;
use TeamTNT\TNTSearch\TNTSearch;
class CreateCityTrigrams extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'city:trigrams';
    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Creates an index of city trigrams';
    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }
    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->info("Creating index of city trigrams");
        $tnt = new TNTSearch;
        $driver = config('database.default');
        $config = config('scout.tntsearch') + config("database.connections.$driver");
        $tnt->loadConfig($config);
        $tnt->setDatabaseHandle(app('db')->connection()->getPdo());
        $indexer = $tnt->createIndex('cityngrams.index');
        $indexer->query('SELECT id, n_grams FROM cities;');
        $indexer->setLanguage('no');
        $indexer->run();
    }
}
```

This creates the index of trigrams that we'll query if we don't find anything in our
`cities.index`

Our `CityController.php` looks like:

```php
<?php
namespace App\Http\Controllers;
use App\City;
use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use TeamTNT\TNTSearch\Indexer\TNTIndexer;
use TeamTNT\TNTSearch\TNTSearch;
class CityController extends Controller
{
    public function search(Request $request)
    {
        $res = City::search($request->get('city'))->get();
        if (isset($res[0])) {
            return [
                'didyoumean' => false,
                'data'       => $res[0]
            ];
        }
        //if we don't find anything we'll try to guess
        $TNTIndexer = new TNTIndexer;
        $trigrams   = $TNTIndexer->buildTrigrams($request->get('city'));
        $tnt = new TNTSearch;
        $driver = config('database.default');
        $config = config('scout.tntsearch') + config("database.connections.$driver");
        $tnt->loadConfig($config);
        $tnt->setDatabaseHandle(app('db')->connection()->getPdo());
        $tnt->selectIndex("cityngrams.index");
        $res  = $tnt->search($trigrams, 10);
        $keys = collect($res['ids'])->values()->all();
        $suggestions = City::whereIn('id', $keys)->get();
        $suggestions->map(function ($city) use ($request) {
            $city->distance = levenshtein($request->get('city'), $city->city);
        });
        $sorted = $suggestions->sort(function ($a, $b) {
            if ($a->distance === $b->distance) {
                if ($a->population === $b->population) {
                    return 0;
                }
                return $a->population > $b->population ? -1 : 1;
            }
            return $a->distance < $b->distance ? -1 : 1;
        });
        return [
            'didyoumean' => true,
            'data'       => $sorted->values()->all()
        ];
    }
}
```
## How does it work

We threw a lot of code at you above. Although this will work perfectly
it's also important that you understand why it's working and what the
logic behind is. Lets, for example, take the city "Berlin". Now, if
we break it into trigrams we become the following:

`__b _be ber erl rli lin in_ n__`

Let's assume you make a typo and instead of Berlin you type "Berln".
The trigrams now look like:

`__b _be ber erl rln ln_ n__`

As you can see, 5 trigrams match the trigrams from the correct form
and only 2 don't match.

![Imgur](http://i.imgur.com/E9JLkwk.jpg)

Each trigram set of the correct form is stored in `cityngrams.index`.
If we query the index with a mistyped  word "berln" well get pretty
accurate results since 5 matches were found. It will also return some
other results that match the trigrams so we'll have to do one more step 
to get the best suggestions possible. The levenshtein distance between
two words will simply tell us how many characters we need to change to 
get the same word. So the levenshtein distance between "Berlin" and "Berln" 
is 1 because we only need to add an "i" for the words to match. So we'll sort
our result from our `cityngrams.index` by levensthein distance and return
the suggestions to the user. Pretty simple but powerfull, isn't it?

The front end of the [demo page](http://cities.tnt.studio/) is build with [react](https://facebook.github.io/react/).
In the upcoming months will create an on-line course on how to get started with React and Laravel
so make sure to subscribe to our newsletter below. Also, follow us on twitter
[@nticaric](https://twitter.com/nticaric) and [@sasatokic](https://twitter.com/sasatokic)
