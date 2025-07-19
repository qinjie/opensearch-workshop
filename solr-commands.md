
## Apache Solr to Amazon Opensearch

### Step 0 - Install SolrCloud on EC2

Create 3 X EC2 instances in the same subnet.

Install Apache Solr on them.

```
sudo yum install java-11-amazon-corretto -y
java -version

sudo su
cd ~
wget http://archive.apache.org/dist/lucene/solr/8.6.2/solr-8.6.2.tgz
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.5.7/apache-zookeeper-3.5.7-bin.tar.gz

tar xfvz solr*.tgz
tar xfvz apache-zookeeper-*.tar.gz

cd solr-*/bin
./install_solr_service.sh /root/solr*.tgz -n

sed -i "s|#SOLR_HOST.*|SOLR_HOST=`curl -s http://169.254.169.254/latest/meta-data/public-hostname`|g" /etc/default/solr.in.sh

# Replace with your EC2 instances public DNS
sed -i 's|#ZK_HOST.*|ZK_HOST="ec2-18-212-174-25.compute-1.amazonaws.com:2181,ec2-54-91-128-241.compute-1.amazonaws.com:2181,ec2-54-81-141-240.compute-1.amazonaws.com:2181"|g' /etc/default/solr.in.sh


mv /root/apache-zookeeper-*-bin /opt/zookeeper
cd /opt/zookeeper
mkdir /opt/zookeeper/data

# Change to 2 and 3 for solr-node-2 and solr-node-3 respectively
echo "1" > /opt/zookeeper/data/myid

cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg
sed -i 's|#autopurge|autopurge|g' /opt/zookeeper/conf/zoo.cfg
sed -i 's|dataDir.*|dataDir=/opt/zookeeper/data|g' /opt/zookeeper/conf/zoo.cfg
echo "server.1=ec2-18-212-174-25.compute-1.amazonaws.com:2888:3888" >> /opt/zookeeper/conf/zoo.cfg
echo "server.2=ec2-54-91-128-241.compute-1.amazonaws.com:2888:3888" >> /opt/zookeeper/conf/zoo.cfg
echo "server.3=ec2-54-81-141-240.compute-1.amazonaws.com:2888:3888" >> /opt/zookeeper/conf/zoo.cfg

cd /opt/zookeeper/bin
bash zkServer.sh start

service solr start
```

Configure the secrity group of all 3 instances to 
1/ open all ports to each other.
2/ open port 8983 to computer which will access Solr Dashboard.


### Step 1 - Setup Solr Server and Client

1. Login to EC2 instance “oss-elasticsearch-client-host“
2. Install required dependencies

```
pip3 install pysolr

```

3. Create a Solr core on one of the Solr EC2 instances

```
sudo su
/opt/solr-8.6.2/bin/solr delete -c data_sentiment
/opt/solr-8.6.2/bin/solr create -c data_sentiment -force
/opt/solr-8.6.2/bin/solr stop -all
service solr start

```

4. Go to Solr UI and select the core that was created. Select schema. We are going to modify the managed schema and add twitter columns using “Add Field”. Add 5 following fields with field type. Leave other stuff defaulted.

```
    tweet_tstamp: string
    user_name: string
    polarity: pdouble
    subjectivity: pdouble
    sentiment: string

```

5. Execute Solr client
```
python3 solr.py

```

6. Query from the UI to verify that the records are getting added.

7. Get count of records in the core we are writing to using below command. We will use this count to verify that our migration was successful.

```
sudo yum install -y jq
curl -s --negotiate -u: 'ec2-18-212-174-25.compute-1.amazonaws.com:8983/solr/data_sentiment/query?q=*:*&rows=0' | jq '.response | .numFound'
```

8. To delete all records in a core

```
# For reference. Do not run this.
{'delete': {'query': '*:*'}}
```

### Step 2 - Start migration process to Amazon Opensearch using Apache Hive (distributed)

1. Launch an EMR 6.6 cluster with 3 x r4.2xlarge nodes and Hive installed

2. Copy required JARs in Hive auxlib path

```
aws s3 cp s3://vasveena-scrape/jars/solr-hive-serde-4.0.0.7.2.15.0-147.jar .
aws s3 cp s3://vasveena-scrape/jars/elasticsearch-hadoop-7.10.3-SNAPSHOT.jar .

sudo cp solr-hive-serde-4.0.0.7.2.15.0-147.jar /usr/lib/hive/auxlib
sudo cp elasticsearch-hadoop-7.10.3-SNAPSHOT.jar /usr/lib/hive/auxlib
sudo cp /usr/share/aws/emr/instance-controller/lib/commons-httpclient-3.0.jar /usr/lib/hive/auxlib

sudo chmod 777 /usr/lib/hive/auxlib/solr-hive-serde-4.0.0.7.2.15.0-147.jar
sudo chmod 777 /usr/lib/hive/auxlib/elasticsearch-hadoop-7.10.3-SNAPSHOT.jar
sudo chmod 777 /usr/lib/hive/auxlib/commons-httpclient-3.0.jar

ls -l /usr/lib/hive/auxlib/

```

### 3. Login to Hive CLI and create a Hive table for Solr core data_sentiment

```
CREATE EXTERNAL TABLE data_sentiment_solr (`id` string,
`user_name` string,
`tweet_tstamp` string,
`message` string,
`polarity` double,
`subjectivity` double,
`sentiment` string
 )
ROW FORMAT DELIMITED lines terminated by '\n'
STORED BY 'com.lucidworks.hadoop.hive.LWStorageHandler'
TBLPROPERTIES('solr.server.url' = 'http://ec2-18-212-174-25.compute-1.amazonaws.com:8983/solr',
              'solr.collection' = 'data_sentiment',
              'solr.query' = '*:*');
```

### 4. Make sure you are able to query the table

```
select * from data_sentiment_solr limit 5;
```

### 5. Create an index in Amazon Opensearch called “data_sentiment_solr” with below schema. You can add extra fields or remove any fields you want.

```
curl -XDELETE -u 'admin:Test123$' 'https://vpc-opensearch-domain-66-2cojbcwgonrbsgwadq524hrgoi.us-east-1.es.amazonaws.com:443/data_sentiment_solr'

curl -XPUT -u 'admin:Test123$' 'https://vpc-opensearch-domain-66-2cojbcwgonrbsgwadq524hrgoi.us-east-1.es.amazonaws.com:443/data_sentiment_solr' -k -H 'Content-Type: application/json' -d '
        {
          "settings": {
            "number_of_shards": 2,
            "number_of_replicas": 2
          },
          "mappings" : {
           "properties" : {
             "tweet_id": {"type": "text"},
             "user" : {"type": "text"},
             "tweet_tstamp" : { "type" : "date", "format" :  "yyyy-MM-dd HH:mm:ss"},
             "tweet" : {"type": "text"},
             "polarity" : {"type" : "float"},
             "subjectivity" : {"type" : "float"},
             "sentiment" : {"type": "text"},
             "modified_tstamp" : { "type" : "date", "format" :  "yyyy-MM-dd HH:mm:ss"}
            }
           }
         }
       '
```

### 6. Create a Hive table for Amazon Opensearch index we just created

```
CREATE EXTERNAL TABLE data_sentiment_os (`tweet_id` string,
`user_name` string,
`tweet_tstamp` string,
`message` string,
`polarity` float,
`subjectivity` float,
`sentiment` string,
`modified_tstamp` string
 )
STORED BY 'org.elasticsearch.hadoop.hive.EsStorageHandler'
TBLPROPERTIES(
    'es.nodes' = 'https://vpc-opensearch-domain-66-2cojbcwgonrbsgwadq524hrgoi.us-east-1.es.amazonaws.com',
    'es.port' = '443',
    'es.net.http.auth.user' = 'admin',
    'es.net.http.auth.pass' = 'Test123$',
    'es.net.ssl' = 'true',
    'es.nodes.wan.only' = 'true',
    'es.nodes.discovery' = 'false',
    'es.resource' = 'data_sentiment_solr/_doc'
);
```

### 7. Make sure you are able to query the table without any errors. 

Table is empty at this point but if you get any errors when you do a select *, check /var/log/hive/user/hadoop/hive.log

### 8. Try to insert test data  (optional)

```
insert into table data_sentiment_os values ('0','test','2022-06-28 01:04:55','test from hive','0.0','0.0','neutral','2022-06-28 01:04:55')
```

### 9. Now perform insert from Solr Hive table into Amazon Opensearch Hive table

```

insert into table data_sentiment_os (tweet_id, tweet_tstamp, user_name, message, polarity, subjectivity, sentiment, modified_tstamp) select id, tweet_tstamp, user_name, message, polarity, subjectivity, sentiment, from_unixtime(unix_timestamp()) from data_sentiment_solr;

```

### 10. Once job finishes, verify the count between Solr source and Amazon Opensearch destination

```
sar -n DEV 1 3

curl -s --negotiate -u: 'ec2-18-212-174-25.compute-1.amazonaws.com:8983/solr/data_sentiment/query?q=*:*&rows=0' | jq '.response | .numFound'
390

curl -X GET -u 'admin:Test123$' 'https://vpc-opensearch-domain-66-2cojbcwgonrbsgwadq524hrgoi.us-east-1.es.amazonaws.com:443/data_sentiment_solr/_count?pretty' -k
{"count":390,"_shards":{"total":2,"successful":2,"skipped":0,"failed":0}}
```

### Final Step - Upgrade Clients

Write client -> Apache Solr (sync) -> Amazon Kinesis Firehose (async) -> Amazon OpenSearch

```
import os
import json
import boto3
import tweepy
import pytz
from tweepy import OAuthHandler
from tweepy import StreamingClient
from textblob import TextBlob
from datetime import datetime
import requests
from requests_aws4auth import AWS4Auth
import pysolr

class process_tweet(tweepy.StreamingClient):
  print("in process_tweet")
  def on_data(self, data):
    print("on_data")
    # print(data)
    # decode json
    dict_data = json.loads(data)

    # pass tweet into TextBlob
    tweet_id = TextBlob(dict_data['data']['id'])
    timestamp = datetime.now().astimezone(est).strftime("%Y-%m-%d %H:%M:%S")
    tweet = TextBlob(dict_data['data']['text'])
    tweetstr = str(tweet)
    s = tweetstr.split(':')[0].strip()
    p = s.partition("RT @")
    message = dict_data["data"]["text"].strip('\n').strip()
    #stripped = ''.join(e for e in message if e.isalnum())
    if len(p) >= 2:
        if p[1] == "RT @":
            user_name = p[2]
        else:
            user_name = "none"
    else:
        user_name = "none"

    # determine if sentiment is positive, negative, or neutral
    if tweet.sentiment.polarity < 0:
        sentiment = "negative"
    elif tweet.sentiment.polarity == 0:
        sentiment = "neutral"
    else:
        sentiment = "positive"

    # output values
    print("tweet_id: "+ str(tweet_id))
    print("timestamp: " +timestamp)
    print("user_name: " + user_name)
    print("tweet_polarity: " + str(tweet.sentiment.polarity))
    print("tweet_subjectivity: " + str(tweet.sentiment.subjectivity))
    print("sentiment: " + sentiment)

    final_solr_data = {}
    final_solr_data['id'] = dict_data['data']['id']
    final_solr_data['user_name'] = user_name
    final_solr_data['tweet_tstamp'] = timestamp
    final_solr_data['message'] = message
    final_solr_data['polarity'] = str(tweet.sentiment.polarity)
    final_solr_data['subjectivity'] = str(tweet.sentiment.subjectivity)
    final_solr_data['sentiment'] = sentiment

    final_fh_data = {}
    final_fh_data['tweet_id'] = str(tweet_id)
    final_fh_data['timestamp'] = timestamp
    final_fh_data['user_name'] = user_name
    final_fh_data['tweet_polarity'] = str(tweet.sentiment.polarity)
    final_fh_data['tweet_subjectivity'] = str(tweet.sentiment.subjectivity)
    final_fh_data['sentiment'] = sentiment

    json_solr_data = json.dumps(final_solr_data)
    json_fh_data = json.dumps(final_fh_data)

    # index values into Solr
    solr.add(final_solr_data)

    # Write to Kinesis Firehose with delivery stream set to Amazon OpenSearch
    fh.put_record(
            DeliveryStreamName='PUT-OPS-lVe9k',
            Record={'Data': json_fh_data},
        )

  def on_error(self, status):
    print(status)

if __name__ == '__main__':
#def handler(event, context):
    print("in handler")

    # Create Solr client
    print("create Solr client")
    solr = pysolr.Solr('http://ec2-18-212-174-25.compute-1.amazonaws.com:8983/solr/data_sentiment', always_commit=True)

    # Create client for Kinesis Firehose with delivery stream set to Amazon OpenSearch
    print("create client instance for firehose")
    fh = boto3.client('firehose', region_name='us-east-1',aws_access_key_id=os.environ.get("AWS_ACCESS_KEY_ID"),aws_secret_access_key=os.environ.get("AWS_SECRET_ACCESS_KEY"))

    print("opensearch instance created")
    est = pytz.timezone('US/Eastern')

    bt=os.environ.get("TWITTER_API_TOKEN")
    printer = process_tweet(bt)
    printer.sample()

```

Repeat for all Solr cores you want to migrate.
