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


  
	
