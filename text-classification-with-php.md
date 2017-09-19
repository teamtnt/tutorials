# Text Classifiction with PHP

It's hard to ignore the fact that machine learning is becoming ubiquitous.
Everybody is talking about it and everybody seems to be using it.
You'll start asking yourself if PHP is suitable to get into the race. 
It's true that machine learning tasks are CPU intensive, but speed of the
language isn't everything.

With the [TNTSearch](https://github.com/teamtnt/tntsearch) project, we try to bring to you state of the art algorithms
implemented in pure PHP. Those algorithms give first class results and are meant
to be used in real app scenarios. This tutorial will cover text classification.

Text classification can be used for many applications. To name just a few:

* spam detection
* text categorization
* sentiment analysis

A state of the art algorithm for text classification is Multinomial Naive Bayes.
This is a probabilistic learning method which calculates the probability of a document
being in a category. We won't bore you with the Bayes theorem and formulas, there are
plenty resources on the web to satisfy your curiosity. What you probably are waiting
for is how to get your hands dirty and get into the code.

Our main goal was to make it simple to use and make it fast. If you read some tutorials
on classification before, you were only thought how to train your model and to apply
it right away. In practice, this is not the case, you need to save the trained model
somewhere so you can reuse it.

Let's say you're running a blog and want to classify your comments. We want to see
which comments are SPAM and which are not. The ones that are not spam, we'll call
HAM. Ok, enough talking let's dig into the code. First, we need to install TNTSearch,
which is, by the way, a search engine entirely written in PHP but also has some cool stuff
like classification, which is part of information retrieval, and this is of course also
machine learning. So, installing

`composer require teamtnt/tntsearch`

In the root of your project create a `classify.php` file. This is of course only a suggestion,
you'll probably be using a framework like laravel and you are most likely to create an
artisan command for this. But here, we'll keep it simple.

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use TeamTNT\TNTSearch\Classifier\TNTClassifier;

$comments = loadCSV('/load/comments/from/disk.csv');

$classifier = new TNTClassifier();

foreach ($comments as $comment) {
    $classifier->learn($comment->text, $comment->category)
}

$classifier->save('./path/to/comments.cls');
```

What this piece of code does is it simply takes the .csv file containing the comments
together with their category which can be either SPAM or HAM and teaches the classifier.
After that, the trained model is saved to disk to ./path/to/comments.cls
That's it, this is how you train your model. You only need to do the training once. The
larger the dataset the better because the predictions will be more accurate.
Now that we have a trained model, predicting if a comment is SPAM or not is easy. In your
guess.php file, you'll have the following:

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use TeamTNT\TNTSearch\Classifier\TNTClassifier;

$classifier = new TNTClassifier();
$classifier->load('./path/to/comments.cls');

$guess = $classfier->predict('This is a nice post');

echo $guess['label'];
```

The above code is very simple. We load the model from disk and ask the classifier to tell
us what the prediction might be.

We did some tests with large datasets and the performance and accuracy are astonishing. The
SPAM/HAM SMS classification test has a score of `98.4753%
