# HW: Search Engine

In this assignment you will create a highly scalable web search engine.

**Due Date:** Sunday, 9 May

**Learning Objectives:**
1. Learn to work with a moderate large software project
1. Learn to parallelize data analysis work off the database
1. Learn to work with WARC files and the multi-petabyte common crawl dataset
1. Increase familiarity with indexes and rollup tables for speeding up queries

## Task 0: project setup

1. Fork this github repo, and clone your fork onto the lambda server

1. Ensure that you'll have enough free disk space by:
    1. bring down any running docker containers
    1. run the command
       ```
       $ docker system prune
       ```

## Task 1: getting the system running

In this first task, you will bring up all the docker containers and verify that everything works.

There are three docker-compose files in this repo:
1. `docker-compose.yml` defines the database and pg_bouncer services
1. `docker-compose.override.yml` defines the development flask web app
1. `docker-compose.prod.yml` defines the production flask web app served by nginx

Your tasks are to:

1. Modify the `docker-compose.override.yml` file so that the port exposed by the flask service is different.

1. Run the script `scripts/create_passwords.sh` to generate a new production password for the database.

1. Build and bring up the docker containers.

1. Enable ssh port forwarding so that your local computer can connect to the running flask app.

1. Use firefox on your local computer to connect to the running flask webpage.
   If you've done the previous steps correctly,
   all the buttons on the webpage should work without giving you any error messages,
   but there won't be any data displayed when you search.

1. Run the script
   ```
   $ sh scripts/check_web_endpoints.sh
   ```
   to perform automated checks that the system is running correctly.
   All tests should report `[pass]`.

## Task 2: loading data

There are two services for loading data:
1. `downloader_warc` loads an entire WARC file into the database; typically, this will be about 100,000 urls from many different hosts. 
1. `downloader_host` searches the all WARC entries in either the common crawl or internet archive that match a particular pattern, and adds all of them into the database

### Task 2a

We'll start with the `downloader_warc` service.
There are two important files in this service:
1. `services/downloader_warc/downloader_warc.py` contains the python code that actually does the insertion
1. `downloader_warc.sh` is a bash script that starts up a new docker container connected to the database, then runs the `downloader_warc.py` file inside that container

Next follow these steps:
1. Visit https://commoncrawl.org/the-data/get-started/
1. Find the url of a WARC file.
   On the common crawl website, the paths to WARC files are referenced from the Amazon S3 bucket.
   In order to get a valid HTTP url, you'll need to prepend `https://commoncrawl.s3.amazonaws.com/` to the front of the path.
1. Then, run the command
   ```
   $ ./download_warc.sh $URL
   ```
   where `$URL` is the url to your selected WARC file.
1. Run the command
   ```
   $ docker ps
   ```
   to verify that the docker container is running.
1. Repeat these steps to download at least 5 different WARC files, each from different years.
   Each of these downloads will spawn its own docker container and can happen in parallel.

You can verify that your system is working with the following tasks.
(Note that they are listed in order of how soon you will start seeing results for them.)
1. Running `docker logs` on your `download_warc` containers.
1. Run the query
   ```
   SELECT count(*) FROM metahtml;
   ```
   in psql.
1. Visit your webpage in firefox and verify that search terms are now getting returned.

### Task 2b

The `download_warc` service above downloads many urls quickly, but they are mostly low-quality urls.
For example, most URLs do not include the date they were published, and so their contents will not be reflected in the ngrams graph.
In this task, you will implement and run the `download_host` service for downloading high quality urls.

1. The file `services/downloader_host/downloader_host.py` has 3 `FIXME` statements.
   You will have to complete the code in these statements to make the python script correctly insert WARC records into the database.

   HINT:
   The code will require that you use functions from the cdx_toolkit library.
   You can find the documentation [here](https://pypi.org/project/cdx-toolkit/).
   You can also reference the `download_warc` service for hints,
   since this service accomplishes a similar task.

1. Run the query
   ```
   SELECT * FROM metahtml_test_summary_host;
   ```
   to display all of the hosts for which the metahtml library has test cases proving it is able to extract publication dates.
   Note that the command above lists the hosts in key syntax form, and you'll have to convert the host into standard form.
1. Select 5 hostnames from the list above, then run the command
   ```
   $ ./downloader_host.sh "$HOST/*"
   ```
   to insert the urls from these 5 hostnames.

## ~~Task 3: speeding up the webpage~~

Since everyone seems pretty overworked right now,
I've done this step for you.

There are two steps:
1. create indexes for the fast text search
1. create materialized views for the `count(*)` queries

## Submission

1. Edit this README file with the results of the following queries in psql.
   The results of these queries will be used to determine if you've completed the previous steps correctly.

    1. This query shows the total number of webpages loaded:
       ```
       select count(*) from metahtml;
        ```
        ```
         count
        -------
         257833
        (1 row)
       ```

    1. This query shows the number of webpages loaded / hour:
       ```
       select * from metahtml_rollup_insert order by insert_hour desc limit 100;
       ```
       
       ```
         hll_count |  url   | hostpathquery | hostpath |  host  |      insert_hour       
        -----------+--------+---------------+----------+--------+------------------------
                 2 | 58028  |         56442 |    54139 |      2 | 2021-05-06 03:00:00+00
                 4 | 76332  |         77912 |    74490 |      4 | 2021-05-06 02:00:00+00
                 3 | 34355  |         36341 |    33251 |  13119 | 2021-05-06 01:00:00+00
                 3 | 25801  |         27334 |    24044 |  45070 | 2021-05-06 00:00:00+00
                 3 | 33614  |         34159 |    30133 |  46134 | 2021-05-05 23:00:00+00
                 1 | 24405  |         26328 |    23912 |  26014 | 2021-05-05 22:00:00+00
         (6 rows)
       ```

    1. This query shows the hostnames that you have downloaded the most webpages from:
       ```
       select * from metahtml_rollup_host order by hostpath desc limit 100;
       ```
       ```
         url | hostpathquery | hostpath |          host                  
          ---+---------------+----------+----------------------------
       13442 |         13283 |    13438 | com,pinterest)
         650 |           672 |      641 | com,nytimes)
         359 |         13283 |    13438 | com,bing)
          71 |            71 |       71 | com,theguardian)
          54 |            54 |       54 | com,fivethirtyeight)
          23 |            23 |       23 | com,google,drive)
          23 |            23 |       23 | com,tripadvisor)
          22 |            22 |       22 | fr,tripadvisor)
          22 |            22 |       22 | com,cnet)
          22 |            22 |       22 | com,washingtontimes)
          21 |            21 |       21 | net,slideshare)
          20 |            20 |       20 | com,coursehero)
          19 |            19 |       19 | com,funnyjunk)
          18 |            18 |       18 | com,euronews)
          18 |            18 |       18 | com,fanatics)
          18 |            18 |       18 | com,dpreview)
          17 |            17 |       17 | com,reddit)
          17 |            17 |       17 | com,weather)
          17 |            17 |       17 | com,weatherbug,weather)
          17 |            17 |       17 | id,co,tripadvisor)
          16 |            16 |       16 | com,wsj,online)
          16 |            16 |       16 | nl,tripadvisor)
          15 |            15 |       15 | se,tripadvisor)
          14 |            14 |       14 | com,pocketmags)
          13 |            13 |       13 | com,veteranstoday)
          13 |            13 |       13 | au,com,sbs)
          16 |            13 |       13 | jp,co,yahoo,gyao)
          13 |            13 |       13 | com,usatoday)
          13 |            13 |       13 | com,tasnimnews)
          12 |            12 |       12 | com,reuters)
          12 |            12 |       12 | jp,osaka-info)
          12 |            12 |       12 | kr,co,koreatimes)
          13 |            12 |       12 | org,usni,blog)
          13 |            12 |       12 | au,net,abc)
          12 |            12 |       12 | uk,co,telegraph)
          11 |            11 |       11 | com,redbubble)
          14 |            11 |       11 | com,freerepublic)
          11 |            11 |       11 | com,ussanews)
          11 |            11 |       11 | uk,co,bbc)
          11 |            11 |       11 | de,spiegel)
          11 |            11 |       11 | com,theatlantic)
          11 |            11 |       11 | com,krdo)
          11 |            11 |       11 | de,imi-online)
          10 |            10 |       10 | com,reuters,uk)
          10 |            10 |       10 | int,who)
          10 |            10 |       10 | com,townhall)
          10 |            10 |       10 | jp,go,niid)
          10 |            10 |       10 | uk,co,independent)
          10 |            10 |       10 | gov,treasury)
          10 |            10 |       10 | il,org,terrorism-info)
          10 |            10 |       10 | com,nydailynews)
          10 |            10 |       10 | org,imf)
          11 |            10 |       10 | com,straitstimes)
          10 |            10 |       10 | com,upi)
          10 |            10 |       10 | net,eurotopics)
          10 |            10 |       10 | com,localnews8)
          10 |            10 |       10 | com,indiatimes,economictimes)
          10 |            10 |       10 | fr,conseil-constitutionnel)
          16 |            10 |       10 | org,resumenlatinoamericano)
           9 |             9 |        9 | com,wordpress,natethayer)
           9 |             9 |        9 | com,radiosantafe)
           9 |             9 |        9 | com,fortunascorner)
           9 |             9 |        9 | com,patterico)
           9 |             9 |        9 | com,dailysignal)
           9 |             9 |        9 | co,edu,uexternado)
          11 |             9 |        9 | es,sport)
           9 |             9 |        9 | eu,tripadvisor)
           9 |             9 |        9 | com,theintercept)
           9 |             9 |        9 | de,juraforum)
           9 |             9 |        9 | com,thedefensepost)
           9 |             9 |        9 | com,nationalpost)
           9 |             9 |        9 | de,hart-brasilientexte)
           9 |             9 |        9 | com,mashable)
           9 |             9 |        9 | com,thehindu)
           9 |             9 |        9 | ru,infox)
           9 |             9 |        9 | com,sputniknews)
           9 |             9 |        9 | com,dnyuz)
           9 |             9 |        9 | com,latimes)
           9 |             9 |        9 | org,journal-neo)
           9 |             9 |        9 | com,sinonk)
           9 |             9 |        9 | com,cnbc)
           9 |             9 |        9 | com,sigforum)
           9 |             9 |        9 | com,yahoo,news,ca)
           9 |             9 |        9 | org,journal-neo,ru)
           9 |             9 |        9 | org,fao)
          13 |             9 |        9 | com,sedaily)
          11 |             9 |        9 | ac,lgbt)
           8 |             8 |        8 | com,headtopics)
           8 |             8 |        8 | info,fnad)
           8 |             8 |        8 | com,guardianlv)
           8 |             8 |        8 | com,gatofuns)
           8 |             8 |        8 | com,wordpress,shuzheng)
           8 |             8 |        8 | info,itbnews)
           8 |             8 |        8 | com,france24)
           8 |             8 |        8 | com,vietnamartnews)
           8 |             8 |        8 | com,onenewspage)
           8 |             8 |        8 | fr,lemonde)
           8 |             8 |        8 | es,20minutos)
           8 |             8 |        8 | com,thestar)
          11 |            10 |        8 | com,timeanddate)
        (100 rows)
       ```

1. Take a screenshot of an interesting search result.
   Add the screenshot to your git repo, and modify the `<img>` tag below to point to the screenshot.

   <img src='screenshot.png' />

1. Commit and push your changes to github.

1. Submit the link to your github repo in sakai.
