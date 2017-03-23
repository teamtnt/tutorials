

# Implementing Geo Search
Sooner or later you’ll find yourself building an application that deals with geolocation and if you don’t have a strong understanding of it you’ll have a hard time to do it right. In this tutorial we’ll try to explain the inner workings of a powerful geolocation engine. Later, we’ll show how to simply do it by using TNTSearch, but before that, you need to understand it.

Some basic geographic terminology is required so we’ll start with longitude and latitude. Let’s imagine the world as a ball. If we draw vertical lines that are spaced equally, we’ll get lines of longitude

![globe-longitude](https://cloud.githubusercontent.com/assets/824840/24270000/b1cb6372-1013-11e7-9c94-0839e7856a3f.jpg)

If we do the same but this time drawing lines horizontally, we’ll get lines of latitude.

![globe-latitude](https://cloud.githubusercontent.com/assets/824840/24270007/bb913f8a-1013-11e7-9297-7ed0dfb8732f.jpg)

Any point on our imaginary ball can be represented with longitude and latitude.

![globe-point](https://cloud.githubusercontent.com/assets/824840/24270011/c611b14c-1013-11e7-8573-cdea387174de.jpg)

If we have two points located somewhere on our ball, we can calculate their shortest distance with the haversine formula

<img width="439" alt="haversine-formula" src="https://cloud.githubusercontent.com/assets/824840/24270027/d27a7c2a-1013-11e7-8000-bc85a3f7f9ea.png">

Don’t worry about the math behind for now, just know that this gives you the shortest distance.

So far so good, now that we have some basic understanding, let’s try to tackle a common problem. In our example, we’ll try to find all Apple stores in the radius of 30 km of our current location. You can extend this problem to anything that fits your use case.
A pre-requirements for our made up example is that we have a list of all Apple stores in the world with their respective longitude and latitude.

First, lets try to solve the problem with the most obvious solution. If we name the point of our current location L, and the apple stores with A1, A2, A3 etc.. we can go through each store and calculate the distance between the current store and our current location ( d= haversine(L, An) ). After we sort the list of distances, we return only the once where the distance is less than 30km. 
As you can see, this approach isn’t very effective, it’s a brute force approach and the complexity of this is O(N) where N represents the number of apple stores. So if your N is large, you’ll have some performance issues because you need to calculate the distance between your location and every store.

I think we can do better. If our search radius is 30km, do we really need to check the points that are maybe on the north or south pole? I guess we don’t. Let say we are in New York L(40.712784, -74.005941).
It makes sense to check only the places that have the latitude between 40.712784 + 30km and 40.712784 - 30km, right? Because every thing else will be out of our boundaries. The same goes for longitude, so we need to check the distance between -74.005941 + 30km and -74.005941 - 30km. Ok, so you see that I’m mixing kilometres here with longitude and latitude. Longitude and latitude are in degrees and kilometres are, well, in kilometres, you cannot sum them just like that. What we need to do first, is to calculate how many kilometres is one degree of latitude and how many is one kilometre of longitude. Lets start simple.

![globe-cut](https://cloud.githubusercontent.com/assets/824840/24270042/de313216-1013-11e7-931a-86a252a8e9cb.jpg)

We know that there are 90 degrees from the equator to the north pole and also we know that there are 10000km from equator to the north pole. Doing the calculation is easy. One degree represents 111,111km. Note that the distance of 10000 kilometers was defined when the metric system was introduced. Today we’ know that the number is a little bit different because of the earth curvature so 111.045km per latitude degree is a better fit.
Calculating  longitude km per degree bit different, because the lines of longitude get closer together on north and south pole. So the formula would be 

111.045 * cos(latitude)

Our example of 30 km latitude would be 0,2701607456 degrees. You can get km to longitude degrees by putting the numbers in the above formula.

This approach creates a boundary  rectangle on the globe where the north boundary is 40.712784 + 0,2701607456 and the south boundary is 40.712784 - 0,2701607456. This way we don’t need to check those points that are greater than 40,9829447456 and smaller than 40,4426232544 which is a huge improvement. The same logic is applied for east and west.
