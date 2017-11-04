![Searching for Bobby Fisher with Laravcel 5](http://i.imgur.com/BUvPWvr.jpg)

In modern web applications, a common requirement is a search feature. 
Clients are spoiled by Google and other search engines and expect a 
powerful search experience in their own products. In this tutorial, 
we'll cover how to search your user base and sequentially make yourself 
or your clients happy.

If you worked on a medium sized or larger application, assuming you have 
a typical "users" table, you came up with a solution yourself like:

    public function scopeSearchByKeyword($query, $keyword)
    {
        if ($keyword!='') {
            $query->where(function ($query) use ($keyword) {
                $query->where("firstname", "LIKE","%$keyword%")
                    ->orWhere("lastname", "LIKE", "%$keyword%")
                    ->orWhere("email", "LIKE", "%$keyword%")
                    ->orWhere("phone", "LIKE", "%$keyword%");
            });
        }
        return $query;
    }

This will certainly work for one keyword, but you'll face problems when you try
to search by multiple keywords like `Bobby Fischer`. So you dig deeper into your
toolbox and write something like:

    $users = User::where(function ($q) use ($query) {
        $q->where(DB::raw('CONCAT( firstname, " ", lastname)'), 'like', '%' . $query . '%')
        ->orWhere(DB::raw('CONCAT( lastname, " ", firstname)'), 'like', '%' . $query . '%')
        ->orWhere('email', 'like', '%' . $query . '%')
    });

As you see, this will cover the case `Bobby Fischer` but will have several downsides. This approach doesn't scale very well. If the wildcard `%` operator is both on the left and right side of the query `%query%`, the internal index of the database cannot be leveraged, which
means that the DB engine needs to go through every single row to see if there's a match, and 
that's bad... not to mention slow!

If you introduce more fields, you'll have even more permutations in your code so you'll end up 
with a nonperformat and nonreadable query.

The solution to this problem is a [package](https://github.com/teamtnt/tntsearch) written in pure PHP 
that deals with this stuff and lets you do some cool things. Laravel also introduced a driver based solution
for fulltext search which works nicely with TNTSearch.

The installation process is easy and can be done with the following command

`composer require teamtnt/laravel-scout-tntsearch-driver`

After that, add the service provider to `app/config/app.php`:

// config/app.php
'providers' => [
    // ...
    Laravel\Scout\ScoutServiceProvider::class,
    TeamTNT\Scout\TNTSearchScoutServiceProvider::class,
],


Add  `SCOUT_DRIVER=tntsearch` to your `.env` file

Then you should publish `scout.php` configuration file to your config directory

```bash
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

In your `config/scout.php` add:

```php

'tntsearch' => [
    'storage'  => storage_path(), //place where the index files will be stored
    'fuzziness' => env('TNTSEARCH_FUZZINESS', false),
    'fuzzy' => [
        'prefix_length' => 2,
        'max_expansions' => 50,
        'distance' => 2
    ],
    'asYouType' => false,
    'searchBoolean' => env('TNTSEARCH_BOOLEAN', false),
],
```

The first thing is creating the index and the second thing is answering the search 
queries using the index we created. 

We'll create a laravel command that will do the indexing for us:

    namespace App\Console\Commands;

    use Config;
    use Illuminate\Console\Command;
    use TNTSearch;

    class IndexUsers extends Command
    {
        /**
         * The name and signature of the console command.
         *
         * @var string
         */
        protected $signature = 'index:users';

        /**
         * The console command description.
         *
         * @var string
         */
        protected $description = 'Index the users table';

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

            $indexer = TNTSearch::createIndex('users.index');
            $indexer->query('SELECT id, firstname, lastname, email, phone, bio FROM users;');
            $indexer->run();
        }
    }

Once we have this command we can run `php artisan index:users`. This will create a file in your
storage folder called `users.index`. The index is now complete and we can do queries against it.

Another approach to achieve the same thing is to add a searchable trait to your model:


```php
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class User extends Model
{
    use Searchable;

    public $asYouType = true;
    
    /**
     * Get the indexable data array for the model.
     *
     * @return array
     */
    public function toSearchableArray()
    {
        $array = $this->toArray();

        // Customize array...

        return $array;
    }
}
```

And then run:

`php artisan scout:import App\\Post`

Thats it. Doing the search is simple. We'll do this in our `UserController`:

    public function index(Request $request)
    {
        $users = User::search($request->input('q'))->get();

        return view('users.index', compact('users'));
    }

This controller assumes that you have a simple search form which submits a query in 
a field called `q`.


The `searchBoolean` method is a very powerful feature and if you are familiar with
some basic boolean algebra you'll understand right away how it works. 

Every space represents an `AND` operator so when you type `Bobby Fisher` you are actually 
asking for every record that contains the words `Bobby` and the word `Fisher`. It translates to
`Bobby AND Fisher`. 

If you want results that contain either Bobby or Fisher you would write `Bobby or Fisher`. 

What about negation, ie. if you want to get all Fishers that aren't Bobbies? Simple, `Bobby -Fisher`. 

The dash `-` represents the negation.

You can also type part of an email like `gmail.com` which will return all users that have a Gmail address. I'm sure you can think of many more examples.

Updating the index manually isn't neccesarry beacuse scout will do it automatically.

What about scaling you may ask? Will it handle more than 10k users?
Don't worry, it will scale fine even if you have millions of users.

This is a simple use tutorial to get you started with TNTSearch [package](https://github.com/teamtnt/tntsearch) which is
often more than enough to satisfy your or your client needs for searching. 

In the upcoming versions of the package, you can expect new features like "weighting schemes", "fuzzy searching" and similar advanced features. 

If you like the [package](https://github.com/teamtnt/tntsearch) please star it on Github, it means a lot to have support
from the community and if you have suggestions please let us know via Github or in comments.

For other cool tutorials subscribe to our newsletter bellow.
