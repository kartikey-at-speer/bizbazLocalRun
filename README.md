---
title: BizBaz optimizations
tags: []
---

# BizBaz


## Database Administration: Review and suggestions
 
A code snippet shared between all the scrapers is:
```
        client = MongoClient('MongoString')
        db = client["heatmap"]
        collection = db["scraperName"]

```
This creates an individual connection for every scraper which is a poor practice. The process of a database connection involves:
- DNS resolution
- TCP connection
- Handshake
- Authentication
- Finally, the actual Query

Which means you're paying for the overhead of creating and maintaing these connections everytime while all previously made connections are sitting idle. 

Further, the database connection should always be stored as a server level variable then than hardcoding it as a string. As the project gets scaled and the db calls become more frequent, it becomes more logical to run your own custom mongo server with replicas running. In such an instance, the developer would be needed to visit every file and update the values with the new database connection strings. This will create further problems if a few files are missed from the refactor. 

### Indexing 
Indexing is the primary solution to improve the performance of a database. MongoDB by design, is a much slower database than MySQL or Postgres. Due to this reason, it is essential to have well maintained indexes to have a quick turnaround in db read queries.

By default, Mongo adds an index on the id field. Currently, the project has no other indexes in the ** Heatmap DB**. This would have a ripple effect on all read queries of the DB and will lead to poor performance on the user side of the app. 

![](ScreenshotBizBaz.png)

### Normalization

All scraper tables have data which is common among them, eg, all tables have title, keywords, industry_keywords, content etc. Since the Data is of a similar nature, it is recommended to keep these in the same table and the aspects/fields which vary between tables should be added as "extensions" on this data. This allows for more efficient queries and all differenct sources can still be identified using the "source" field present in the data. 

### Timestamps

Mongo also offers create and update timestamps which allow us to add easy logging of document versions and updates.

# Code Enhancements



From my first review of the platform, my primary concern is the sequentiality of it. All scrapers are executed in a serial "one by one" manner whereas there is no sharing of data/context variables betweem them and so they can run just as easily in parallel to provide much faster results. Equipped with multithreading, the platform ran almost 12x faster on my local machine, and much less powerful than a production server. 

There are a lot of code level optimization that we can make. I'm adding a few here: 

Inside scraper, once the "homesoup" object is created, we can further add enhancements to process different kinds of objects in parallel. 

An example:
```
news_content = homesoup.find('div', class_ = 'news_content').find('ul').find_all('a')

        for article in news_content:
            href = article['href']
            if utils.is_local_link(href):
                link = home_url + href[2:]
                if utils.check_duplicate(collection, link):
                    continue
                try:
                    scraped = self.scrape_article(link)
                except Exception as e:
                    print('Unable to scrape article at ' + link)
                    print('Encountered an exception:' , e)
                    continue
                scraped['section'] = 'News Content'
                scraped['source'] = self.source
                collection.insert_one(scraped)
            
        intl_coop = homesoup.find('ul', class_ = 'part2_news').find_all('a')

        for article in intl_coop:
            href = article['href']
            if utils.is_local_link(href):
                link = home_url + href[2:]
                if utils.check_duplicate(collection, link):
                    continue
                try:
                    scraped = self.scrape_article(link)
                except Exception as e:
                    print('Unable to scrape article at ' + link)
                    print('Encountered an exception:' , e)
                    continue
                scraped['section'] = 'International Cooperation'
                scraped['source'] = self.source
                collection.insert_one(scraped)
```

As we can see here, both objects (news_content and intl_copp) have the exact same processing flow with the exception of "Section". These two flows can also be executed parallel. Another code level optimization that can be done here is creating a function to handle both these objects in the following manner:
```
    def process(object, section):
        for article in object:
            href = article['href']
            if utils.is_local_link(href):
                link = home_url + href[2:]
                if utils.check_duplicate(collection, link):
                    continue
                try:
                    scraped = self.scrape_article(link)
                except Exception as e:
                    print('Unable to scrape article at ' + link)
                    print('Encountered an exception:' , e)
                    continue
                scraped['section'] = section
                scraped['source'] = self.source
                collection.insert_one(scraped)
```

the objects can then be simply processed by calling them as:
```
process(int_scoop, "International")
```
This makes the code more readable and easier to maintain as well as modify. Another advantage of breaking these out as functions is we can share them across all scrapers instead of limiting this to Ministry Of Agriculture scraper. 

An example of this, is the function named 
"__is_p_or_block" which is already shared between CDT and xinhua scrapers. There are multiple examples of such functions which are duplicated in the code.

### Code cleanup and refactoring

All programs should follow the five primary principles of good software design:
- The Single-responsibility principle: "There should never be more than one reason for a class to change."
- The Open–closed principle: "Software entities should be open for extension, but closed for modification."
- The Liskov substitution principle: "Functions that use pointers or references to base classes must be able to use objects of derived classes without knowing it."
- The Interface segregation principle: "Many client-specific interfaces are better than one general-purpose interface."
- The Dependency inversion principle: "Depend upon abstractions, [not] concretions."

The above 5 rules, although cannot be enforced always, generally outline a great framework for software development. BizBas in it's current state, does not follow some of these principles.

There is a lot of room for granular improvements as well. These Atomic improvements cumulatively have a huge impact on the code quality.


Example:  
```
if not pagesoup.find(id='Content'):
            pagesoup = main.find('div', class_='picshow')
            picshow = True
        else:
            picshow = False
        if picshow:
            info = main.find('span', class_='info_l').text
            #article['title'] = main.find('h1').text.strip()
        else:
            info = pagesoup.find('span', class_='info_l').text
            #article['title'] = pagesoup.find('h1').text.strip()
Note: This is the last documented usage of picshow
```
can be simplified to:
```
        if not pagesoup.find(id='Content'):
            pagesoup = main.find('div', class_='picshow')
            info = main.find('span', class_='info_l').text
        else:
            info = pagesoup.find('span', class_='info_l').text
```

# Overall review

In terms of the Python and Flask programs, the BizBaz platform has a significant scope for improvement in terms of code quality and system design principles. With these changes in place, I expect the platform to be able to be able to:
- Scrape significantly more data
- Store data in a more a more optimized manner
- Make the process of insight derivation from data easier and more profound
- Operate much faster 
- Become open to extension in terms of data sources as well as feature richness 

-----
