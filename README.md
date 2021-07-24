# neo4j-yelp Data Modelling and Processing
Use Neo4J (a labelled graph database) to explore yelp data to see if any insights can be found.

Neo4j lists, as best practice, the following steps to designing the initial data model, highlighting that it is an iterative process.
<ol>
<li>Understand the domain.</li>
<li>Create high-level sample data.</li>
<li>Define specific questions for the application.</li>
<li>Identify entities.</li>
<li>Identify connections between entities.</li>
<li>Test the questions against the model.</li>
<li>Test scalability.</li>
</ol>

## Setting up files for local processing
yelp's dataset comprises of 5 json files:
<ol>
<li> yelp_academic_dataset_business.json</li>
<li> yelp_academic_dataset_checkin.json</li>
<li> yelp_academic_dataset_review.json</li>
<li> yelp_academic_dataset_tip.json</li>
<li> yelp_academic_dataset_user.json</li>
</ol>

To make processing of the files easier, I have downloaded them from [yelp](https://www.yelp.com/dataset/download) to my local file system.

In order to use the files from Neo4j desktop, they need to be placed in the imports folder. To find out where the import folder on your local drive is, go to Manage->Open Folder->Import
<br>
![image](https://user-images.githubusercontent.com/830693/126421255-a390bef3-481e-4fc8-b6c5-c22a97c3e75d.png)

This will open the import folder in a new window. To access the files from Neo4j desktop command line, you will have to enable local file import in the neo4j.conf file.  
Similarly, to access this file, go to Manage->Open Folder-> Configurations.  
Open the "neo4j.conf" file in your favorite editor and add the following configuration line:
<pre>apoc.import.file.enabled=true</pre>
<br>
After the configuration takes effect, the file can be accessed from Neo4j's command line. The path is relative to the "import" folder's path, hence you would define the path as
<pre> :param yelp_business_file => '/yelp_dataset/yelp_academic_dataset_business.json'</pre>
<br> if you have unzipped the files into a "yelp_dataset" folder under "import" folder. This command sets up the path string as a param which can be access using $yelp_business_file in other commands.

##Understand the domain
yelp is a online business directory that is used world-wide for discovering local businesses ranging from bars, restaurants, and cafes to hairdressers, spas, and gas stations. Listings are sorted by business types and results are filtered by geographical location, price, and unique features like outdoor seating, delivery service, or a the ability to accept reservations.
It publishes crowd-sourced reviews about these businesses.  
  
Some questions that came to mind:
* Do restaurants with high ratings (> 4 stars) congregate together?
* Do reviewers tend to give altogether high/low ratings? Do the majority (> 75%) of their ratings trend high/low?
* If person A gives a high rating (> 4 stars) for a restaurant, does his/her friends also gives a high rating to the same restaurant? Also low ratings.

Now that the files and Neo4j browser is set up, we can view the contents of the files on Neo4j browser.  
To view the first 10 items of the file, run the following command:
<pre>CALL apoc.load.json($yelp_business_file) YIELD value
RETURN value LIMIT 10</pre>  

For a start, I am interested in the following fields:  
1.	Business
* business_id: string, 22 character unique string business id
* name: string, the business's name
* latitude: float, latitude
* longitude: float, longitude
* stars: float, star rating, rounded to half-stars
* review_count: integer, number of reviews
* categories: an array of strings of business categories
2. User
* user_id: string, 22 character unique user id, maps to the user in user.json
* name: string, the user's first name
* review_count: integer, the number of reviews they've written
* friends: // array of strings, an array of the user's friend as user_ids
* average_stars: float, average rating of all reviews

### Importing data, creating relationships
A first pass of the fields available and using the arrows.app tool to create our first graph model.
![image](https://user-images.githubusercontent.com/830693/126449058-4722a6da-3431-4902-b483-1c157f7a0b7d.png)

Looking at this graph, I think some of the following questions can be answered (if not, we might need to refactor the graph model in a later iteration):
* How many restaurants are there?
* What is the average number of reviews that a restaurant has?
* What is the average number of reviews that each user gave?
The preceding questions looks like they would have been better answered using a traditional database, or even another NoSQL database.
The power of a labelled property graph database lies in traversing the relationships of the graph, and some of the following questions might take advantage of that:
* On average, how many friends does each user have?
* Do restaurants tend to get a higher/lower review from people who are friends?
	
Let's try to import "Business" and "User" datasets into our database.  
Let's see how many records there are in the datasets:
<pre>CALL apoc.load.json($yelp_business_file) YIELD value
RETURN count(value);
CALL apoc.load.json($yelp_user_file) YIELD value
RETURN count(value)</pre>
User: 2,189,457
Business: 160,585

Both apoc.load.json and "LOAD CSV" are single-threaded commands, so the loading might be able to be sped up using parallel processing with *apoc.periodis.iterate*.  
The general construct of the command is:  
<pre>WITH 'call apoc.load.json("/yelp/yelp_dataset/yelp_academic_dataset_user.json") YIELD value RETURN value' AS load_json
CALL apoc.periodic.iterate(load_json, 'WITH value 
CREATE (u:User) SET
    u.user_id= value.user_id,
	u.name = coalesce(value.name),
	u.review_count = value.review_count,
	u.average_stars = value.average_stars',
{batchSize: 5000, parallel: True, iterateList: True, retries:3}) 
YIELD batches, total
RETURN *
</pre>
The above command took 48709ms (around 49seconds) to run on my desktop with exactly 2,189,457 nodes created:
![image](https://user-images.githubusercontent.com/830693/126462080-da955ee7-d904-4f45-b513-44a7b1bd4ec3.png)

NOTE: Initially the command just ended without any error message but no nodes were created. To investigate further, look at the Neo4j's log files located at (you guessed it...) Manage->Open Folder->Logs.  
I found the problem under neo4j.log. There was extra curly brackets around {value} which were not required (cryptic error message was: 
<pre>The old parameter syntax `{param}` is no longer supported. Please use `$param` instead (line 1, column 59 (offset: 58))
"UNWIND $_batch AS _batch WITH _batch.value AS value  WITH {value}"</pre>

We will have to go through the user file for a second time to create the FRIEND_OF relationship between all the user nodes.
<pre>WITH 'call apoc.load.json("/yelp/yelp_dataset/yelp_academic_dataset_user.json") YIELD value RETURN value' AS load_json
CALL apoc.periodic.iterate(load_json, 'WITH value 
MATCH (u:User {user_id:value.user_id}) 
WITH u,value.friends as friends
UNWIND friends as friend
MERGE (u1:User{user_id:friend})
MERGE (u)-[:FRIEND_OF]-(u1)',
{batchSize: 1000, parallel: True, iterateList: True, retries:3}) 
YIELD batches, total
RETURN *

</pre>
NOTE: The above had trouble running to completion, after running for > 3hrs, only 18,000 relationships had been created and it was all 1-1 relationships, i.e., no single node had more than one friend as the following return 0 rows:
<pre>MATCH (p1:User)-[r:FRIEND_OF]-(p2:User)
WITH p1, count(r) as num_friends
WHERE num_friends > 1
RETURN p1,p2
</pre>

<pre>
CALL apoc.periodic.iterate("
CALL apoc.load.json('/yelp/yelp_dataset/yelp_academic_dataset_user.json')
YIELD value RETURN value
","
MERGE (u:User{id:value.user_id})
SET u += apoc.map.clean(value, ['friends','user_id'],[0]),u.name = coalesce(value.name),
	u.review_count = value.review_count,
	u.average_stars = value.average_stars
WITH u,value.friends as friends
UNWIND friends as friend
MERGE (u1:User{id:friend})
MERGE (u)-[:FRIEND_OF]-(u1)
",{batchSize: 100, iterateList: true});
</pre>

![image](https://user-images.githubusercontent.com/830693/126593085-1ac75a36-69ac-4e20-a526-25bbeb045680.png)
![image](https://user-images.githubusercontent.com/830693/126593106-1250e32d-dd9c-4e19-ae3f-3c32a074c2dd.png)

![image](https://user-images.githubusercontent.com/830693/126593762-35bbeba9-830d-449c-b7b9-6f0d7d6d832a.png)




