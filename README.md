Designing Link Shortening Services

Link shortener:Link Shortener will be used to create a short alias URL for a corresponding long URL, so when a user visits the short URL, he/she will be redirected to the original URL.

Example: Let the original URL be:
Long URL: https://www.google.com
Short URL: https://<your-url>/xCd5a
Whenever you will click the second short url, you will automatically
be redirected to the page referred by the long URL (https://www.google.com).

Design Goals & APIS
The Link Shortening Service will have API endpoints for the following features:
Endpoint: /urls
Create shortened URL
Retrieve shortened URL
Update shortened URL
Delete shortened URL 2. Stats of each shortened URL, Endpoint: /stats/<url or id>
a. Total number of clicks
b. Number of clicks by day
c. Total number of people who clicked links.
Total number of people who clicked by day 3. We will require to have an API for user sign-up. Upon sig-up, we return a token.
Endpoint: /signUp
  
Constraints & Capacity Estimation
Constraints:
The link shortening service is projected to have 1,000 daily active users creating 5 to 10 short URLs per day. The short URLs are used in social media and other applications with a predicted daily volume of 100,000 daily visitors clicking the short

Traffic: Considering we have 1000 active uses creating 10 short URLs/day,
i.e, 10 _ 1000 _ 30 = 300,000 short URL/months, If we are going to keep the service for 5 years, our service will generate about 300,000 *12 *5 = 18M records.
Also, we will have 100,000 daily visitors clicking the short URLs, which makes the system 10x Read heavy than Write operation (~1,000/day).

URL Length: Shortened URL can be a combination of characters (a-z, A-Z) and numbers (0-9) of length 6 (excluding domain).

Database Design:
Each customer will be given an API token for use with the service. Also, we need to keep track of the usage of each shortened URL. Based on that below will be our DB tables/collection and relationship:

    User Schema:

id: Default id
name: User’s name
email: User’s email
creationDate: User creation datetime
Url Schema:
userId: Reference to user’s ID
longUrl: The long URL user wants to shortened
shortUrl: Shortened URL
clicks: Number of clicks on shortened URL

Relationship & Constraint:
To keep track of usage of each shortened URL, we will have a reference of User in url schema.
We have a constraint, Duplicate shortened URLs for each customer are not allowed. So we need to make the longUrl and userId field unique together

Data Storage modeling:
Let’s consider the average data size for the user and url schema is about ~2KB/record
So, we will have, (2 _ 2 _ 18M = 72MB) of data in 5 years.

Observations:
We are storing about 2 million records over the few years.
As discussed earlier, our service is going to be read-heavy.
We have to create 2 models, and there is going to exist ONLY one relationship between them.

Choosing Database:
Based on these observations, I strongly believe that a NoSQL database (MongoDB) will be a good option for the service. NoSQL is highly available and easily scalable.
Usually, RDBMS is used where we have, multiple relationships among the tables and we need to write complex queries using JOINs, which we don’t have in our service. Also, It’s hard to scale RDBMS in comparison to the NoSQL database.

Choosing Language: My first preference is Python with Django & Django REST framework. The problem here is, Django & DRF doesn’t have great support for MongoDB. So for this project, I have two choices

Shortening Algorithm:
Techniques to store Tiny URL:
We have a constraint here: Shortened URLs must be unique for each customer. If two different customers create a short URL to the same destination (customer A and customer B both create short URLs to https://www.google.com), each customer is given a unique shortened URL.

    MD5 Hashing: Write few lines about MD5

MD5 Approach: Hash the long URL using the MD5 and take only the first 5 characters to generate a short URL.

Problem: MD5 will return the same hash value for the same URL, even for different users, which is not acceptable. One approach to resolve this could be, we can add the user token to the input long URL and then hash it. This will ensure that we are getting unique short URL for different users.

Another problem with this approach is that the first 5 characters of the hashed value could be the same for different input URLs. So, we need to check if the short URL
(5 character hash value) already present every time in the DB for saving it. If it is present then create a new one.
Though this approach makes our service a bit slow, I think we can stick with this approach, because we anyway need to query the DB every time to check if the user has already created a short URL for a given long URL. if the longUrl already exists, we gonna return the corresponding short URL to the user, instead of creating a new one. And if the longUrl doesn’t exist we create a new hash value for the input URL and check the uniqueness and save it in DB.
Also, we will first convert the input URL to UTF-8 format before saving it to DB. This will take care of any URL-encoded input from the user.

Counter Approach:
This approach ensures that we don’t need to verify the uniqueness of the short URL, but this approach is complex in comparison to the MD5 approach.
Using a counter could be a better solution because counters always get incremented so we can get a new value for every new request. With this approach, we don’t need to check the uniqueness of the short URL.
Single server approach:  
A single (counter host) server will be responsible for maintaining the counter.
When the application server receives a request it connects to the counter host, which returns a unique number and increments the counter. When the next request comes the counter host again returns the unique number and this goes on.
Every worker host gets a unique number that is used to generate a short URL.
Problem:
The counter host will be a single point of failure. If the counter host goes down for some time then it will create a problem, also if the number of requests will be high then the counter host might not be able to handle the load.
Also, if we scale the application server and have multiple servers, it will be quite difficult for the counter host to synchronize and send unique value to each application server.

Solution: To solve this problem we can use a distributed service Zookeeper, to manage all these tedious tasks and to solve the various challenges of a distributed system like a race condition, deadlock, or particle failure of data.
Zookeeper is basically a distributed coordination service that manages a large set of hosts. It keeps track of all the things such as the naming of the servers, active servers, dead servers, configuration information of all the hosts. It provides coordination and maintains the synchronization between the multiple servers.
Before I discuss the implementation, we need to understand, how many combinations of short URLs can we make. For our service, we are considering 5 characters of short URLs made of 62 chars (a-z, A-Z, 0-9). So there will be 62 \*\* 5 = 916,132,832 ~ 100M combinations.

From 100M combinations take 1st million combinations.
In Zookeeper maintain the range and divide the 1st million into 100 ranges of 1 million each i.e. range 1->(1 – 1,000,000), range 2->(1,000,001 – 2,000,000 and so on.
When servers will be added these servers will ask for the unused range from Zookeepers. Suppose the s1 server is assigned range 1, now s1 will generate the tiny URL incrementing the counter and using the encoding technique. Every time it will be a unique number so there is no possibility of collision and also there is no need to keep checking the DB to ensure that if the URL already exists or not. We can directly insert the mapping of a long URL and short URL into the DB.
In the worst case, if one of the servers goes down then we will only lose a million combinations in Zookeeper (which will be unused and we can’t reuse it as well) but since we have 100 million combinations we should not worry about losing this combination.
If one of the servers will reach its maximum range or limit then again it can take a new fresh range from Zookeeper.
The Addition of a new server is also easy. Zookeeper will assign an unused counter range to this new server.
We will take the 2nd million when the 1st million is exhausted to continue the process.
Speeding up the Read operation
As mentioned earlier, our database is going to be Read-heavy, so we need to find a way to speed up the reading process

Caching is one of the best and quite popular solution for Read-heavy operations. We can cache the URLs which are accessed frequently. For caching we can use in-memory caching like Redis.

There are few points we need to consider when we add caching:
When the cache is full, we need to replace the old URLs with new ones.
Synchronizing the cache with original data, if the user updates or deletes the long URL, the corresponding change should be updated in the cache too.

The above approach will speed up our read operation.

Load Balancing
As we have 100,000 read operations every day, there could be a situation where our application server faces high traffic, and it may lead to a single point of failure. To overcome this, we can use Load Balancer.
The load balance will scale out the application service and divides the traffic into groups of servers. This will ensure the high availability and fast response of our URL shortener service.
