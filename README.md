Title: Twitter Civil Unrest Analysis with Apache Spark
Date: 2015-04-29
Category: Data Science
tags: python, spark
Author: Will Farmer
Summary: Using the Streaming Twitter API to analyze tweets as an indicator of civil unrest.

#Introduction

On Monday I attended the CU Boulder Computer Science Senior Projects Expo, and there was one project
in particular that I thought was pretty neat: determining areas of civil unrest through Twitter
post analysis. They had some pretty cool visuals and used Apache Spark, so I figured I'd try to
recreate it on my own. What follows is an initial prototype attempt as a proof of concept. I'll go
through each step included here,
[but I've also included the code on my github profile](https://github.com/willzfarmer/TwitterPanic),
so feel free to clone and run it.

#PySpark Streaming

One initial design choice that I made was to create a streaming program, that constantly ran and
produced results. This meant that my best bet was to use Spark's streaming API. As with any other
Spark program we first create our `SparkContext` object, and since I'm set up on my laptop (with 4
workers) we designate our program to use the local Spark instance. After we instantiate our
`SparkContext` we create our `StreamingContext` and establish a batch interval, which is simply the
interval at which the streaming API will update.

    :::python
    sc  = SparkContext('local[4]', 'Social Panic Analysis')
    ssc = StreamingContext(sc, BATCH_INTERVAL)

We can now create our stream, which is done by setting a default "blank" RDD, and creating a new
`queueStream` who's default contents is the blank RDD from before.

    :::python
    rdd = ssc.sparkContext.parallelize([0])
    stream = ssc.queueStream([], default=rdd)

At this point our stream consists of a single entry, the RDD containing `[0]`. We need to convert that to
usable output, which we do by applying a flat map over every element in the stream. This mapping
process (in my case) converts each element currently in the stream to some new RDD that has some set
data. (Note, you could absolutely have a ton of elements in your blank RDD to start, which would
mean that you're pulling in data over each one of those elements. I did not do that due to [Twitter's
Rate Limiting](https://dev.twitter.com/rest/public/rate-limiting), which I'll go over more in depth
later.)

Now we perform our analysis on the streaming object. (I'll cover this more in depth later)

After the analysis is done we output the data. (Again, I'll go over this later)

Since Spark is lazily evaluated, nothing has been done yet. All we've done is establish the fact
that we have some data stream and that we wish to perform some series of steps on it in our
analysis. The final step is to start our stream and basically just wait for something to stop it.

    :::python
    ssc.start()
    ssc.awaitTermination()


#Appendix: Code

    :::python
    """
    Twitter Panic!

    Real time monitoring of civil disturbance through the Twitter API

    NOTE, ONLY PYTHON 2.7 IS SUPPORTED
    """
    from __future__ import print_function

    import sys
    import ast
    import json
    from pyspark import SparkContext
    from pyspark.streaming import StreamingContext
    from pyspark.streaming.kafka import KafkaUtils
    import requests

    import numpy as np
    import matplotlib.pyplot as plt
    import threading
    import Queue
    import time

    import cartopy.crs as ccrs

    # File with OAuth keys
    # Corresponding format is the following
    # auth = requests_oauthlib.OAuth1(key1, key2, key3, key4)
    # url = 'https://stream.twitter.com/1.1/statuses/filter.json'

    import config


    BATCH_INTERVAL = 60  # How frequently to update (seconds)
    BLOCKSIZE = 50  # How many tweets per update


    def main():
        threads = []
        q = Queue.Queue()
        # Set up spark objects and run
        sc  = SparkContext('local[4]', 'Social Panic Analysis')
        ssc = StreamingContext(sc, BATCH_INTERVAL)
        threads.append(threading.Thread(target=spark_stream, args=(sc, ssc, q)))
        threads.append(threading.Thread(target=data_plotting, args=(q,)))
        [t.start() for t in threads]


    def data_plotting(q):
        plt.ion() # Interactive mode
        fig = plt.figure(figsize=(30, 30))
        ax = plt.axes(projection=ccrs.PlateCarree())
        ax.set_extent([-130, -60, 20, 50])
        ax.coastlines()
        while True:
            if q.empty():
                time.sleep(5)
            else:
                data = np.array(q.get())
                try:
                    ax.scatter(data[:, 0], data[:, 1], transform=ccrs.PlateCarree())
                    plt.draw()
                except IndexError: # Empty array
                    pass


    def spark_stream(sc, ssc, q):
        """
        Establish queued spark stream.

        For a **rough** tutorial of what I'm doing here, check this unit test
        https://github.com/databricks/spark-perf/blob/master/pyspark-tests/streaming_tests.py

        * Essentially this establishes an empty RDD object filled with integers [0, BLOCKSIZE).
        * We then set up our DStream object to have the default RDD be our empty RDD.
        * Finally, we transform our DStream by applying a map to each element (remember these
            were integers) and setting the next element to be the next element from the Twitter
            stream.

        * Afterwards we perform the analysis
            1. Convert each string to a literal python object
            2. Filter by keyword association (sentiment analysis)
            3. Convert each object to just the coordinate tuple

        :param sc: SparkContext
        :param ssc: StreamingContext
        """
        # Setup Stream
        rdd = ssc.sparkContext.parallelize([0])
        stream = ssc.queueStream([], default=rdd)

        stream = stream.transform(tfunc)

        # Analysis
        coord_stream = stream.map(lambda line: ast.literal_eval(line)) \
                            .filter(filter_posts) \
                            .map(get_coord)

        # Convert to something usable....
        coord_stream.foreachRDD(lambda t, rdd: q.put(rdd.collect()))

        # Run!
        ssc.start()
        ssc.awaitTermination()


    def stream_twitter_data():
        """
        Only pull in tweets with location information

        :param response: requests response object
            This is the returned response from the GET request on the twitter endpoint
        """
        data      = [('language', 'en'), ('locations', '-130,20,-60,50')]
        query_url = config.url + '?' + '&'.join([str(t[0]) + '=' + str(t[1]) for t in data])
        response  = requests.get(query_url, auth=config.auth, stream=True)
        print(query_url, response) # 200 <OK>
        count = 0
        for line in response.iter_lines():  # Iterate over streaming tweets
            try:
                if count > BLOCKSIZE:
                    break
                post     = json.loads(line.decode('utf-8'))
                contents = [post['text'], post['coordinates'], post['place']]
                count   += 1
                yield str(contents)
            except:
                print(line)


    def tfunc(t, rdd):
        """
        Transforming function. Converts our blank RDD to something usable

        :param t: datetime
        :param rdd: rdd
            Current rdd we're mapping to
        """
        return rdd.flatMap(lambda x: stream_twitter_data())


    def filter_posts(line):
        """
        Perform sentiment analysis and identify tweets that indicate violence

        :param line: list
            List from dataset
        """
        keywords  = ['riot', 'protest', 'violence', 'angry', 'sad', 'mourn', 'http']
        for k in keywords:
            if k in line[0]:
                return True


    def get_coord(line):
        """
        Converts each object into /just/ the associated coordinates

        :param line: list
            List from dataset
        """
        coord = tuple()
        try:
            if line[1] == None:
                coord = line[2]['bounding_box']['coordinates']
                coord = reduce(lambda agg, nxt: [agg[0] + nxt[0], agg[1] + nxt[1]], coord[0])
                coord = tuple(map(lambda t: t / 4.0, coord))
            else:
                coord = tuple(line[1]['coordinates'])
        except TypeError:
            print(line)
        return coord


    if __name__ == '__main__':
        sys.exit(main())
