---
layout: post
title: "FOSSEE Fellowship Report - Building an Event Logging System"
author: "Krithik Vaidya"
tags: [fossee, spoken-tutorial, open-source, fellowship]
image: FOSSEE-Fellowship-Report/logo.png
---

## Summer Fellowship Report

## Spoken Tutorial Event Logs and Analytics System

### Introduction

This is the project report for my work done as a FOSSEE Summer Fellow under the Spoken Tutorials project. FOSSEE is an initiative by IIT-Bombay to promote the development of open-source applications, especially in the field of education/academia. My work involved creating pipelines for extracting user activity logs (referred to as _Event Logging_) as users browse and interact with the website (which is built using the Django framework), and storing them into a MongoDB database, thus enabling data analysis and visualization. The analytics system for analyzing and visualizing these logs was also parallelly created by my teammate.

The Event Logging system consists of two main parts:-

*   Saving of user activity logs as they browse and visit different parts of the website (henceforth referred to as _Website Logs_).
*   Saving and display of course completion and tutorial progress statistics for users (referred to as _Tutorial Progress Logs_)

The open-source code for the project can be found on GitHub - 

[spoken-website (branch: logs-krithik)](https://github.com/abhijitbonik/spoken-website/tree/logs-krithik)

[spoken-website (branch: logs-krithik-js)](https://github.com/abhijitbonik/spoken-website/tree/logs-krithik-js)

[Spoken-Analytics-System (branch: master)](https://github.com/Spoken-tutorial/Spoken-Analytics-System/)

[Presentation Video](https://drive.google.com/file/d/1MxgjGecnsIjRe7RS8GWwE61BneNbGoqU/view?usp=sharing)

### I. Website Logs

Our first goal was the saving of user activity logs for all the pages on the website. For each visit to a page on the Spoken Tutorials website, we are currently logging information such as:

1. Username of visitor
2. Path of web page visited
3. Browser info, Operating System info, and Device info
4. IP address
5. Date and Time of visit (in UTC timezone)
6. Whether the visit was first time or returning
7. Latitude and Longitude of user
8. Country, Region, and City from which the request originated, if available
9. Request method (GET, POST, etc.)
10. Referring link
11. Page title
12. Arguments and Keyword Arguments sent to the view in the request, if any.
13. POST request data, if any.

Two different architectures have been created for extracting and saving Website Logs. Both architectures are soon talked about in detail:-

*   Django Middleware-based approach (server-side)
*   Javascript-based approach (client-side)

Both these systems have their own pros and cons, which are detailed at a later point below.

Next, the final architectures of both the above approaches will be discussed, with the reasonings for the chosen architectures talked about after.


### Final architecture, Middleware-based Approach

This approach firstly consists of extracting all the log data in the Django middleware, except for the location data. After extracting the data in the middleware, we make an asynchronous call to an API function, created in the [Spoken-Analytics-System](https://github.com/Spoken-tutorial/Spoken-Analytics-System/tree/krithik) (in *logs_api/views.py*, the *save_middleware_log()* function view). This function will get the location data, and then enqueue the log in a Redis queue named *middleware_log*.

For obtaining the location information, we will perform IP-based Geolocation - i.e., we will map the IP address of the visit to location data using the [GeoIP2](https://pypi.org/project/geoip2/) Geolocation service.

A separate monitoring program has been written (*monitor_queue.py*) to monitor the Redis queue. When the number of items in the queue reaches ≥ n (where n is defined by the MONGO_BULK_INSERT_COUNT variable in [settings.py](https://github.com/Spoken-tutorial/Spoken-Analytics-System/blob/krithik/analytics_system/settings.py#L263)), the first n items will be extracted from the queue. According to the value of SAVE_LOGS_WITH_CELERY (True/False, defined in [settings.py](https://github.com/Spoken-tutorial/Spoken-Analytics-System/blob/491257fde3f2c1f8cf09cb3c915e28fca419aeed/analytics_system/settings.py#L261)), the extracted logs will either be saved in bulk directly, by the *monitor_queue()* function itself, or sent as a task to Celery to be asynchronously executed.

The saving of the logs in MongoDB is done using the [pymongo](https://pymongo.readthedocs.io/en/stable/) library. The extraction of Brower info, OS info, and Device info in the Django middleware is done by using the [django-user-agents](https://pypi.org/project/django-user-agents/) library.

To record first-time visits, we set the option SESSION_EXPIRE_AT_BROWSER_CLOSE in Django settings to False, and then use the following snippet:


<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/1.png" alt=""></figure>


The HTML title of the webpage is not available on the server-side, but is required for visualization purposes. To overcome this, we have a separate _event_name_ field in the logs. The event name is determined by Regex matching the URL to a predefined list of patterns of URLs defined by us. After the match is found, it will use the corresponding event name. The EVENT_NAME_DICT used for matching can be found in the spoken-website repo (see below for the screenshot), _logs/urls_to_events.py_ file. The drawback is that for every new URL added to the spoken website that does not match a previously defined pattern, the dict needs to be updated too. Else the logs for visits to those URL patterns won't be recorded.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/2.png" alt=""></figure>

The logs stored in MongoDB can then be used for performing analysis and creating visualizations.

The branch containing the code for this approach can be found [here](https://github.com/abhisgithub/spoken-website/tree/logs-krithik) (the branch also contains the implementation of the Tutorial Progress Logging system). The code written for the API can be found in the [Spoken-Analytics-System](https://github.com/Spoken-tutorial/Spoken-Analytics-System/tree/krithik/logs_api) repository.


### **Explanation of choices in the architecture**


#### **Usage of API for saving logs, instead of doing it locally**

Initially, the middleware asynchronously called a local function (i.e. running on the same system as the server) to do the Geolocation and the pushing of logs to Redis. The Redis monitoring consumer function was also running locally, in the same system as the spoken-website. However, we decide to scrap this approach and transfer the function to a separate API in the Spoken-Analytics-System because:



*   It is easier to perform an asynchronous HTTP call, rather than calling a local function asynchronously. Calling a local function asynchronously complicates the code by introducing asyncio and multithreading.
*   In case of high loads on the spoken-website server, the system and server would be slowed down.
*   Since there are a fewer changes to the main spoken website repository, it would be quicker and easier to review and deploy to production.
*   It makes sense to have the Event Logging and Analytics related code under a separate subsystem.


#### **Asynchronous saving of logs in middleware implementation**

We could have directly done Geolocation and saved the log into MongoDB in the middleware itself, without needing to involve Redis queues and separate APIs. However, these are blocking operations that would increase the page load times for the user especially in case of high load scenarios. It also has the drawbacks mentioned in the previous section, related to saving logs locally instead of API based approach. Also, reliable bulk saving of logs would have been difficult to achieve using a purely middleware-based approach.

**Giving the option of using Celery for bulk saving of logs**

Celery is a full-featured task processing software for Python web applications used to asynchronously execute work outside the HTTP request-response cycle. It provides features such as automatic scaling of the number of workers, multiprocessing to perform concurrent execution of tasks, etc. There are applications like [Flower](https://flower.readthedocs.io/en/latest/) to monitor Celery workers and the task queues. A Celery system can consist of multiple workers and brokers, giving way to high availability and horizontal scaling.

Hence Celery is also a good option for running tasks asynchronously.


#### **Usage of pymongo for saving logs into MongoDB**

To save the logs in MongoDB from a Python function, we started by using [pymongo](https://pymongo.readthedocs.io/en/stable/). Pymongo simply takes a Python dictionary and saves it as a document in the specified MongoDB database and collection. Since there was no validation being done, we decided to add validation to the MongoDB database. This is the current validation schema, after multiple iterations of refinements:

[https://gist.github.com/krithikvaidya/18bcaba3d9a87d8a40e92c4cdd1bac95](https://gist.github.com/krithikvaidya/18bcaba3d9a87d8a40e92c4cdd1bac95)

The above is the MongoDB [validation schema](https://docs.mongodb.com/manual/core/schema-validation/) for the website event logs collection. This is to be entered in the mongo shell, after selecting the appropriate database. The above deals with validating the data just before the document enters the database. This is not necessary to do, but will be helpful in ensuring consistency.

For tighter coupling between our Django application and the MongoDB database, we decided to try replacing the use of pymongo with [Djongo](https://pypi.org/project/djongo/). Djongo is a SQL to MongoDB query transpiler. It translates a SQL query string into a MongoDB query document. As a result, all the Django ORM features, work as-is. It would also let us interact with the Djongo model through the Django admin page.

Hence, using Djongo would add another layer of validation and consistency to the log data in the MongoDB database. Using Djongo allows you to interact with MongoDB exactly as you’d interact with SQL databases using models, with some additional MongoDB specific features.

However, the load testing (talked about below) showed that pymongo performed many orders of magnitude faster than djongo.


## **Choosing the right architecture by performing** **Load Testing**

(This load testing was done before the API based implementation of Middleware-based logs).

Till now, a single async function task/celery task dealt with the storing of a single log in the database. To possibly speed up the saving of logs, we considered bulk saving of logs in a single task (for e.g. 1000 logs in 1 task) instead of a single log per task.

To determine the best setup amongst the different setup described previously, I proceeded to do extensive load testing - using different combinations of



*   Celery/our local async function
*   Djongo/pymongo
*   Single log per task/multiple logs per task.

**Testing method**: Queue with 10000 tasks for each of the (2 * 2 * 2) = 8 setups.

**Testing environment** - my local machine.

According to the load testing observations,



*   The setups using pymongo performed many orders of magnitude better than those using Djongo. This is probably because of the extra Djongo model layer between Django and MongoDB. For every log that needed to be saved, an object of the model had to be created, and the validations, etc. brought in a large overhead. The feature benefits provided by Djongo over pymongo was heavily outweighed by its lower performance and the additional complexity of usage.
*   The setups implementing Bulk saving of logs performed much better than single saving. This is because the time required for inserting a log is determined by the following factors, where the numbers indicate approximate proportions:
    *   Connecting: (3)
    *   Sending query to server: (2)
    *   Parsing query: (2)
    *   Inserting logs: (size of log)
    *   Closing: (1)

    Obviously, bulk saving of logs will be much faster since only a single connection is opened for multiple logs.

*   In this environment, our own consumer function’s performance was better than Celery’s. But, Celery is a much more full-featured task processing software, as talked about previously.

So, considering all the factors, we decided to choose the setup with



*   an option to either use Celery or saving the logs in monitor_queue.py itself,
*   pymongo for saving of logs into MongoDB, and
*   bulk saving of logs per task.

There is a SAVE_LOGS_WITH_CELERY setting in the [settings.py](https://github.com/Spoken-tutorial/Spoken-Analytics-System/blob/krithik/analytics_system/settings.py#L261) file. Setting it to true will use Celery as the task processing queue to save the logs, and setting it to false with use the separate queue monitoring Python function (defined in monitor_queue.py) as the task processing queue. The MONGO_BULK_INSERT_COUNT value controls how many logs are inserted, per bulk insert operation.


### **The monitor_queue.py script**

The _monitor_queue.py_ script (found in the Spoken-Analytics-System repo) monitors the Redis queue, whose name is decided by the value of the USE_MIDDLEWARE_LOGS setting. The number of logs it extracts from the Redis queue per bulk insertion is determined by the MONGO_BULK_INSERT_COUNT settings variable. Many precautions have been taken to avoid crashes. It uses Python's in-built logging system to periodically print out informative log messages. A sample run of the script is as below:

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/3.png" alt=""></figure>

(Each of the iterations producing a line of output in the above has a 5-second delay. This delay can be changed by tweaking the value of the MONITOR_QUEUE_ITERATION_DELAY setting).

The code in the script has been sufficiently commented so as to be understandable.

**The end result - a log stored in MongoDB:**

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/4.png" alt=""></figure>

(In the middleware implementation, it is not possible to get the page title. However, it has been kept in the logs as an empty field for consistency)


## Final Architecture, JS-based Approach

This approach consists of extracting all the log data in the client-side, using Javascript. Once the page DOM Content Loading completes, we run a function to extract all the information. To get accurate location data, we use the HTML5 [Geolocation API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API). If the user accepts, the accurate latitude and longitude data is extracted. Else if the user declines, we query the [freegeoip.app](https://freegeoip.app/json/) API, to map the user IP address to a location.

To get the browser and OS info, we use the [browser-report](https://github.com/keithws/browser-report) JS library. To detect the device type, we use the [device-detector](https://github.com/PoeHaH/devicedetector) JS library.

Once all the data is extracted, a procedure similar to that in the middleware-based architecture is used. We make an AJAX call to another API function created in the Spoken-Analytics-System (in _logs_api/views.py, the_ _save_js_log()_ view function). This view enqueues the log in a Redis queue named _js_log_. The name of the Redis queue monitored by _monitor_queue.py_ (middleware_log or js_log) is decided by the value of USE_MIDDLEWARE_LOGS in [settings.py](https://github.com/Spoken-tutorial/Spoken-Analytics-System/blob/krithik/analytics_system/settings.py) file of Spoken-Analytics-System. The rest of the procedure is the same as in the middleware-based approach, involving bulk inserting logs into the DB in the monitor_queue function itself, or sending it as a task to be processed by Celery.

In the JS implementation, the exit link activity can also be recorded (explained below)

The JS implementation is simpler to deploy to production, as there is comparatively little extra code on the server-side. In the JS implementation, the _logs_ app is only used for defining a context processor (for accessing IP address and Logs API URL) and a template tag. The middleware, views, etc are not used.

The branch containing the code for this approach can be found [here](https://github.com/abhisgithub/spoken-website/tree/logs-krithik) (the branch also contains the implementation of the Tutorial Progress Logging system), as well as in the [Spoken-Analytics-System](https://github.com/Spoken-tutorial/Spoken-Analytics-System/tree/krithik/logs_api) repository.


### **Getting the location data in client-side JS**

We can leverage the HTML5 Geolocation API to get the accurate location data of the client. However, this is subject to the user agreeing to provide their location information. The browser will put up a prompt like this when the page loads:

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/5.png" alt=""></figure>

If the user chooses to accept, the accurate coordinates can be obtained. However, to map the coordinates to Country/Region/City data (called reverse geocoding), we will need to make an API call. There are two free APIs available for this purpose:



1. [OpenStreetMap (Nominatim)](https://nominatim.org/release-docs/develop/api/Reverse/)
2. [Big Data Cloud API](https://www.bigdatacloud.com/geocoding-apis/free-reverse-geocode-to-city-api)

Although both APIs work well, they do not return data in a consistent manner (may not return region/city data, or may return it but label the fields with some other name). Hence they cannot be used in our code. Another option is to do the reverse geocoding on the server-side using [](https://pypi.org/project/reverse_geocoder/)the [reverse_geocoder](https://pypi.org/project/reverse_geocoder/) library. However, it is somewhat slow and may bottleneck the system in case of a large load of requests.

In case the user does not accept the browser's request for location, then the [freegeoip.app](https://freegeoip.app/json/) API will be used for the purpose of performing geolocation (which uses the IP address). This is, however, less accurate.

Side note: Modern browsers do not let user’s location data be accessed using the HTML5 Geolocation API over HTTP. It is allowed only over HTTPS. In a local development environment, since the server runs on HTTP, we will need to add the IP address of our local development server to the chrome flag chrome://flags/#unsafely-treat-insecure-origin-as-secure

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/6.png" alt=""></figure>

### **Exit Link Activity**

Exit links are links on the website which point to URLs having a different hostname than spoken-tutorial.org. To record exit link activity, we add an onclick event listener to all the links on a page. When a link is clicked, the Javascript checks if it has a different hostname. If it does, an AJAX call is made with the data containing exit link, page on which the link was clicked and datetime of the click, to an API on the server-side. The API saves the log into a MongoDB collection named _exit_link_logs._


## Comparison of the two Architectures

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/7.png" alt=""></figure>

All of the above dealt with saving the website user event logs. The second part of the project deals with Course and Tutorial Progress logging.


# Course and Tutorial Progress Logging

Here, we aim to log tutorial-related data while users watch the Spoken Tutorials. Using these logs, we can accomplish a lot - the user can see their course progress, see which tutorials they have completed and which they have yet to complete, continue watching at the timestamp they stopped watching the tutorial at, etc. For the analytics side, we can see which course is more popular, which tutorial is rewatched the most number of times, the average number of visits a user makes to a tutorial, etc.


## Saving video logs in real-time

On the tutorial watch pages (for e.g., [this one](https://spoken-tutorial.org/watch/BASH/Introduction+to+BASH+Shell+Scripting/English/)), the videos are played using the Video.js library. Video.js provides functionality to execute some logic every time the video's timestamp is updated. Using this feature, every time the minute of the video changes from the previously saved log’s minute, we can make an asynchronous AJAX call to an API defined by us, to save the tutorial progress. This API will collect the data sent by the AJAX call, such as:



*   The username of the tutorial watcher
*   The current video time
*   The current datetime
*   The FOSS, the language and the tutorial,
*   The number of times the user has visited that tutorial in that language
*   The total length of the video.

The called API function then does some calculations to check if the video has been completed (if 80% of the minutes of the video has been crossed), etc. and updates the MongoDB document for that user.

Once this entire API call finishes successful execution, another AJAX call is made to an API to check if the tutorial should be marked as complete (if it is not already marked as complete). According to the response, the relevant parts of the DOM are updated to display the completion status.

Buttons have been provided for users to mark the tutorial as completed or incomplete. Clicking the button makes another AJAX call.

For non-authenticated users, the website does not display any progress/completion data, and does not make any of these AJAX calls.

The setup accounts for saving logs in the scenarios of skipping ahead, fast-forwarding, going back, etc.

We can choose to update the tutorial progress at time intervals smaller than a minute, however, this will increase the load on the server.

Please check the screenshots below (the formatting of the pages may look off in some places since the static files (images, thumbnails, etc) are not present on my local setup):

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/8.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/9.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/10.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/11.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/12.png" alt=""></figure>

Once 80% of the minutes of the video are completed, the tutorial is automatically marked as completed (if it wasn’t already marked) in the database, and the webpage is automatically updated to reflect the new completion status.


## Tutorial Search page

On the tutorial search pages, such as [this one](https://spoken-tutorial.org/tutorial-search/?search_foss=BASH&search_language=English), when the user is authenticated and has chosen a FOSS and a language, the course completion percentage and the option to continue watching where they stopped watching is given, as shown below.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/13.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/14.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/15.png" alt=""></figure>

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/16.png" alt=""></figure>

## Schema for saving tutorial progress logs

Currently, the tutorial progress logs are being saved in the same database as the website logs (_logs_ database), but in a different collection (_tutorial_progress_logs_ collection).

The below describes an example of one document in the collection:

[https://gist.github.com/krithikvaidya/25662dca3e80e0d7e59a253de71ba1a3](https://gist.github.com/krithikvaidya/25662dca3e80e0d7e59a253de71ba1a3)

The logged data can also be used for statistical calculations and visualizations. 


# Other Explorations


## Visit Duration Logging

To log the visit duration along with all the other visit data in a single log, we need to change the structure of the Javascript code. Instead of extracting the data and saving the logs on DOMContentLoaded, we will only extract the data on DOMContentLoaded, and save the logs before the page unloads, so that the duration of the visit can also be calculated. To do this, we can use [window.unbeforeunload](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/onbeforeunload) or jQuery’s beforeunload functionality. For mobile users, when the page focus is lost (tab changed, browser closed, etc.), marks the end of the visit.

However, when tested with different browsers, these features were found to behave inconsistently and unreliably. Another option is the <code>[navigator.sendBeacon](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon)</code> method which allows us to asynchronously send a small amount of data over HTTP to a web server. However, older browsers do not support this feature, and [this](https://volument.com/blog/sendbeacon-is-broken) extensive study about the feature concluded that navigator.sendBeacon is broken, and should not be used in production. 

The code for this attempted approach can be found [here](https://github.com/abhisgithub/spoken-website/tree/logs-krithik-js-visitd).


## Redis Persistence

In order to make sure that Redis queue data isn't lost during restarts/crashes of the system. There are two ways of doing this:

**The Redis Database Backup**

The Redis Database Backup, or RDB, files are snapshots that are taken at predetermined frequencies, to be used as a backup in a point-in-time recovery in the event of a data storage failure.

**The Append-Only File**

The append-only file, or AOF, is a mode of data persistence where Redis persist the dataset by taking a snapshot and then appending the snapshot with changes as those changes take place.

These methods can be used together, separately, or not at all in some circumstances.

For our use case, the AOF persistence method is more appropriate.

In basic terms, append-only log files keep a record of data changes that occur by writing

each change to the end of the file. In doing this, anyone could recover the entire

dataset by replaying the append-only log from the beginning to the end.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/FOSSEE-Fellowship-Report/17.png" alt=""></figure>

<span style="text-decoration:underline;">Table 4.1 - Sync options to use with appendfsync</span>


<table>
  <tr>
   <td><span style="text-decoration:underline;">Option</span>
   </td>
   <td><span style="text-decoration:underline;">How often syncing will occur</span>
   </td>
  </tr>
  <tr>
   <td>always
   </td>
   <td>Every write command to Redis results in a write to disk. This slows Redis down substantially if used.
   </td>
  </tr>
  <tr>
   <td>everysec
   </td>
   <td>Once per second, explicitly syncs write commands to disk.
   </td>
  </tr>
  <tr>
   <td>no
   </td>
   <td>Lets the operating system control syncing to disk.
   </td>
  </tr>
</table>


In our case, using appendfsync always would be the most ideal. However, if we were to set appendfsync always, every write to Redis would result in a write to disk, and we can ensure minimal data loss if Redis were to crash. Unfortunately, because we’re writing to disk with every write to Redis, we’re limited by disk performance, which is roughly 200 writes/second for a spinning disk, and maybe a few tens of thousands for an SSD (a solid-state drive).

As a reasonable compromise between keeping data safe and keeping our write performance high, we can also set appendfsync everysec.

A sample Redis configuration file with appendfsync can be found in the repo [here](https://github.com/Spoken-tutorial/Spoken-Analytics-System/blob/krithik/Misc/redis.conf).

[Further Reading](https://redislabs.com/ebook/part-2-core-concepts/chapter-4-keeping-data-safe-and-ensuring-performance/4-1-persistence-options/4-1-2-append-only-file-persistence/)


## MongoDB-Elasticsearch Connection and Sync

For improved visualizations and performance of statistics calculations, moving the logs from MongoDB to Elasticsearch was briefly explored. To sync a MongoDB collection to an Elasticsearch collection, we explored the following options:

**Initial choices:**



*   [Mongo Connector](https://github.com/yougov/mongo-connector/wiki)
*   [Mongoosastic](https://github.com/mongoosastic/mongoosastic)
*   [Mongo Stream](https://github.com/electionsexperts/mongo-stream)
*   [Monstache](https://github.com/rwynn/monstache-site)
*   [Logstash Input MongoDB](https://github.com/phutchins/logstash-input-mongodb)
*   [MongoDB input driver plugin for logstash](https://dbschema.com/jdbc-driver/MongoDb.html)

After considering various factors like appropriateness for our use case, frequency of updates, presence of support, ease of use, support for newer versions of Elasticsearch etc., we narrowed it down to two choices:



1. [MongoDB input driver plugin for logstash](https://dbschema.com/jdbc-driver/MongoDb.html)

This uses the Logstash part of the Elastic stack to sync the two databases. Since Logstash does not have an official MongoDB input plugin, we have to use a 3rd party plugin. The logstash configuration file for this purpose can be found in the Spoken-Analytics-System repository [here](https://github.com/Spoken-tutorial/Spoken-Analytics-System/tree/krithik/Misc/mongo-es-logstash.confhere).

To understand the use of the driver, please refer to the following links:

*   [Link 1](https://stackoverflow.com/questions/58342818/sync-mongodb-to-elasticsearch)
*   [Link 2](https://stackoverflow.com/questions/47956088/mongodb-driver-class-for-logstash-input-plugin-jdbc)
*   [Link 3](https://discuss.elastic.co/t/mongodb-logstash-integration-solved/122299)
*   [Link 4](https://bitbucket.org/dbschema/mongodb-jdbc-driver/src/master/)
*   [Link 5](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html#plugins-inputs-jdbc-jdbc_driver_class)

2\.   [Monstache](https://github.com/rwynn/monstache-site)

Monstache is a go daemon that syncs MongoDB to Elasticsearch in real-time. Probably the easiest to setup.


## Saving of Offline Tutorial-Related Logs

A good portion of Spoken Tutorials users hail from rural-type areas, where internet connectivity is limited. In these cases, the cdcontent download feature is used to download the Spoken Tutorials and view them locally, in a web browser. To record these logs, we can use the HTML5 Web Storage API to save the required logs locally. When the user comes online, the browser can automatically send this data to a Spoken Tutorials API.


# Conclusion

This project dealt with creating robust pipelines for the saving of user activity logs, such as website browsing logs and course/tutorial progress logs. This data can be then used for generating insights, analytics, and visualizations of the user activities on the website. The presence of tutorial progress logs and their associated features contribute toward improving the user experience. Further explorations had been done with respect to Redis data backup, visit duration logging, and pipelining data between MongoDB and Elasticsearch.

For future improvements, we can extend the events tracking to other subdomains under spoken-tutorials.org (such as [forums.spoken-tutorial.org](forums.spoken-tutorial.org), [process.spoken-tutorial.org](process.spoken-tutorial.org), etc.) and improve the cdcontent download logging. We can also log the user activity paths, similar to how it is shown by StatCounter. The idea of syncing MongoDB and Elasticsearch can be further explored to improve the performance of statistical calculations and visualizations.

In the end, I would like to thank FOSSEE and the Spoken Tutorials project for providing me the opportunity to work on this project. I would like to thank my mentor, Sir Abhijit Bonik, for providing guidance and allowing the freedom to explore different approaches. I would also like to thank Ma’am Nancy Varkey and Ma’am Kirti Ambrey for periodically reviewing my work.
