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
            $table->string('accent_city');
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

Notice the column `n_grams`, this will help us later to achieve the did you mean functionality.

