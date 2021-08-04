# Finding Fraudulent reviews
Some clues that might identify a review as fake/fraudulent:
* Reviewer has posted reviews in multiple states
* Reviewer has reviewed several businesses in the same niche
For example, if a profile reviews five garage door companies in three different states, there's a very high chance that this person is engaging in fraudulent activity.

Another article by the [consumerist website](https://consumerist.com/2010/04/14/how-you-spot-fake-online-reviews/) lists 30 ways to spot fake online reviews. Among the 30, these might be suitable for detection using a graph database:
| Method | Suitability for graph database |
|:---|:---|
| Reviewers have no other reviews on the site	| This looks like it will be easier to query for in a normal SQL database. No advantage in graph database	|
| Multiple reviews that are exactly the same	| This looks like it will be easier to query for in a normal SQL database. No advantage in graph database	|
| Beware the "polyamorous" reviewer, every product gets a glowing and unvarnished review...	| More suitable for text/sentiment analysis	|
| And the "monogamous" reviewer, whose only reviews are for products by one manufacturer. All praise, natch.	| Doable in graph query, equally doable in SQL	|
| All of the reviewers have accounts created around the same time, usually around the time the domain nace was registered	| Doable in graph query, equally doable in SQL	|
| Higher proportion of 1 and 5 stars review versus 2-4 stars?	| Doable in graph query, equally doable in SQL	|


## Hypothesis and result set
Hypothesis to narrow down dataset for analysis:
* Users that have 2 or less reviews.
* These reviews are written on the same day that they joined yelp.
* These reviews are 1 or 5 stars (Note: There have been research which shows that people are more likely to leave a review when their experience is extremely good or bad. For example, roughly 50% of products on Amazon have a bimodal distribution of ratings).

We get around 224,145 which satisfies the above criteria:
<pre>MATCH (u:User)-[w:WROTE]->(r:Review)
with u, count(r) AS numReview
WHERE numReview <=2  
MATCH (u)-[w:WROTE]->(r:Review)-[re:REVIEWS]->(b:Business)
WHERE 
    apoc.date.convertFormat(r.date, "yyyy-MM-dd HH:mm:SS", 'iso_date') = apoc.date.convertFormat(u.yelping_since, "yyyy-MM-dd HH:mm:SS", 'iso_date') 
    AND (r.stars = "1.0" OR r.stars = "5.0")
RETURN count(r)</pre>

Still too many to go through unless we want to introduce a text analyser.  
Let's narrow down to "Restaurants" Category in the same city, we want to find users who have exactly 2 reviews, one 5 star and the other 1 star, and the user joined yelp on the same day as both reviews were posted.
<pre>MATCH (u:User)-[w:WROTE]->(r:Review)
with u, count(r) AS numReview
WHERE numReview = 2
MATCH (u)-[w5:WROTE]->(r5:Review {stars:'5.0'})-[re5:REVIEWS]->(b5:Business)-[ic5:IN_CATEGORY]->(cat51:Category {category_id:'Restaurants'})<-[ic1:IN_CATEGORY]-(b1:Business)<-[re1:REVIEWS]-(r1:Review {stars:'1.0'})<-[w1:WROTE]-(u)
MATCH (b5)-[:IN_CITY]->(city51:City)<-[:IN_CITY]-(b1)
WHERE apoc.date.convertFormat(r5.date, "yyyy-MM-dd HH:mm:SS", 'iso_date') = apoc.date.convertFormat(r1.date, "yyyy-MM-dd HH:mm:SS", 'iso_date') AND apoc.date.convertFormat(r5.date, "yyyy-MM-dd HH:mm:SS", 'iso_date')=apoc.date.convertFormat(u.yelping_since, "yyyy-MM-dd HH:mm:SS", 'iso_date')
RETURN  u.yelping_since, r5.date,r1.date, city51.city_id, cat51.category_id , u.user_id, r5.stars, r5.text, b5.name, r1.stars, r1.text, b1.name</pre>
  
A very manageable 390 records fulfilling the criteria was found. Listed some of the more suspicious records:
| u.yelping_since | r5.date | r1.date | city51.city_id | cat51.category_id | u.user_id | r5.stars | r5.text | b5.name | r1.stars | r1.text | b1.name |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
| "2010-04-17 20:16:08"	 | "2010-04-17 20:26:43" | "2010-04-17 20:22:56" | "Cambridge-Massachusetts-US" | "Restaurants" | "u-XUXO7UNkHKXvN-kcvcRG7w" | "5.0" | "Excellent food. They have great Sushi and noodles and curries. But they have a delivery charge of 2 $." | "Spice & Rice" | "1.0" | "They have a nice menu, but the food tastes bad. We ordered Masala dosa and Mysore masala dosa and it is the worst dosa I have ever eaten. Even the sambar tasted like water, no flavor, no salt. They add capsicum to the dosa potato which is simply wrong and should not be allowed. Overall the food tastes a few days old." | "Dosa Factory" |
| "2017-04-15 23:07:39" | "2017-04-15 23:07:45" | "2017-04-15 23:09:44" | "West End-British Columbia-CA" | "Restaurants" | "u-x4RvI2njJD1awbyFvVIYGA" | "5.0" | "The only place I eat sushi Downtown! Quality is consistent, portions are huge, price is great, and the service is exceptional! Highly recommend!" | "Samurai Japanese Restaurant" | "1.0" | "$30 gets you barely enough, poorly constructed, low quality food for a small child." | "ShuRaku Sake Bar & Bistro" |
|"2018-09-12 12:03:24" | "2018-09-12 12:07:33" | "2018-09-12 12:10:44" | "Woburn-Massachusetts-US" | "Restaurants" | "u-Yzm8CeUCbCq0GbYZjPP6mg" | "5.0" | "As a vegetarian, I sometimes feel a bit limited here but every time I go, everything I have is delicious and the service is always so wonderful. The waiters are so warm and welcoming, and they are great with kids! Make sure to make a reservation as space is limited and it is always packed." | "Pintxo Pincho Tapas Bar" | "1.0" | "They have no vegetation options which is extremely disappointing. Had t leave as soon as I got there." | "Gene's Chinese Flatbread Cafe"|
|"2017-03-29 23:01:42" | "2017-03-29 23:52:29" | "2017-03-29 23:04:36" | "Belle Isle-Florida-US" | "Restaurants" | "u-_g57VcGyuAp8kNpKcun2jw" | "5.0" | "I would recommend this restaurant to all. I sat at the bar and watched what I believe was the manager look over all food that went out. My food was fantastic, service was great also my server / bartender Susan was a great" | "Cask & Larder" | "1.0" | "Bartender served me a drink with a ROACH in it the his manager still wanted me to pay for it" | "Johnny Rivers' Grill & Market"|
|"2018-07-14 21:56:04" | "2018-07-14 21:59:41" | "2018-07-14 22:02:03" | "Durham-Oregon-US" | "Restaurants" | "u-mlvlSWZ_iJRNeMqYDKlIXQ" | "5.0" | "This restaurant has the freshest oysters in the Portland Metro area and if you would like a great meal you have to try this place. The staff is very friendly and takes great care of its customers. Enjoy." | "Ways & Means Oyster House" | "1.0" | "Do you not ever go to this bar and Grill they have the worst customer service I have ever had. The food is bad the waitstaff is even worse. The bartender has the very worst attitude I've ever seen from a bartender you don't want to go here ever." | "Speakeasy Bar & Grill"|
|"2015-01-24 05:11:44" | "2015-01-24 05:17:23" | "2015-01-24 05:11:58" | "North Atlanta-Georgia-US" | "Restaurants" | "u-avPrwHu9K0gXzawrWSWr6A" | "5.0" | "Great choice for excellent and fresh sushi! The atmosphere is very casual and the food is amazing! This is definitely my go to sushi spot in Atlanta. Plus the plum wine is insatiable!" | "Sushi Bar Yu-ka" | "1.0" | "Horrible horrible experience. Food was awful and the memu is so confusing. The manager has a very bad attitude, and overall the place was just not clean. I would never ever suggest rusans to anybody! One of the worst experiences ever!!!! The only thing good about my experience with the waiter he was actually very nice." | "Ru San's"|
|"2016-03-12 03:24:08" | "2016-03-12 03:42:52" | "2016-03-12 03:24:10" | "Worthington-Ohio-US" | "Restaurants" | "u-Ms3eYVwPpFkBIdOQX790tg" | "5.0" | "I love Dante's pizza. I no longer live very close but I work downtown and will sometimes pick-up a pizza on the way home. Tonight we dined-in and had pizza hot out of the oven. The staff are friendly, the pizza is amazing, they're a neighborhood favorite and I highly recommend Dante's!!!" | "Dante's Pizza" | "1.0" | "I had purchased a Groupon to try Ange's and when I tried to redeem it I was told that they were under new management and that they wouldn't honor it. Now they may be under new management but what really disappointed me was that the new management made NO effort to see if they could convince me to stay and still try their pizza. They just had this, "sorry about your luck" attitude and let us walk out without any attempt to see if they could attract us as a customer. It appears this place needs new management again! What was also surprising was that the place was empty, and this was a Friday night. The new management needs a lesson in marketing and building a customer base. I'll address my Groupon refund with Groupon but I wouldn't give this new management my business, they don't seem to be concerned with attracting customers. We drove to Dante's and had amazing pizza, go there instead." | "Ange's Pizza"|
|"2015-07-26 00:41:39" | "2015-07-26 00:58:18" | "2015-07-26 00:55:25" | "Tangelo Park-Florida-US" | "Restaurants" | "u-9YyRq53_UidiIzmkkjljEw" | "5.0" | "Best sushi I have ever had in orlando. The spice tuna bowl is to die for. The rice is just right, and no other place can match it. The fish taste like they killed it right before they served it to me. I would give this more stars if I could. One word.....Perfection." | "Sushi Tomi" | "1.0" | "This was the worst sushi I have ever had. I had my girlfriend go to pick me up sushi. She ordered twice. The first order came back to the table poorly prepared and had a weird taste. She returned it for a different order came. It still did not look right. She brought it home and I ate it. I can tell you this that if you are looking to lose weight then Aki is for you. Just know that I lost 3 hrs of my life to my bathroom. Vaseline was my savior." | "Aki Restaurant"|
