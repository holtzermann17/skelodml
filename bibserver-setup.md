# Installation

The "[simple setup](http://bibserver.readthedocs.org/en/latest/install.html)" instructions for installing BibServer leave out some important pieces, but the "[deployment](http://bibserver.readthedocs.org/en/latest/deploy.html)" instructions could mostly be followed.  To install a basic setup for hacking on a Macbook Pro, here's what I needed:

- java
- elasticsearch
- git
- python2.7
- pip
- virtualenv
- gunicorn

(Further instructions for a serious deployment will follow in due course.)

## Ingredients

1. Install Java 1.8 from Oracle (http://www.java.com/download).  
2. Download ElasticSearch 1.2.1 ([tar.gz](https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.2.1.tar.gz)) and unpack that.
3. I had git installed, but otherwise run `port install git-core`
4. I had to do some fiddling with MacPorts make sure that `python` is Python 2.7.7.
5. `port install py-pip`
6. `port install py-virtualenv`
7. Turn on Apache via `sudo apachectl start`

In order to be sure you're using the right Java version, you may need to add this to your `~/.profile`:

```sh
export JAVA_HOME=`/usr/libexec/java_home -v 1.8`
```

Then, run `source ~/.profile`.  To make sure that you're using the correct Python version, run `python --version` and `which python` - if it's not 2.7.7, follow the tips [here](http://stackoverflow.com/questions/8201760/how-to-macports-select-python).

## Recipe

Set up a "virtualenv" environment:

1. `mkdir ~/env/`
2. `virtualenv ~/env/testdir`
3. `cd ~/env/testdir`                                 
4. `source bin/activate`

Add the required pieces to this environment:

1. `git clone https://github.com/okfn/bibserver`
2. `cd bibserver`                               
3. `pip install -e .`
4. `pip install gunicorn`

Due to some changes to the way one of the Python dependencies works, you'll need to patch a file in order to get things running.  (This will hopefully be integrated into the mainline code shortly.)

```diff
diff --git a/bibserver/view/account.py b/bibserver/view/account.py
index 9f938f7..08d188b 100644
--- a/bibserver/view/account.py
+++ b/bibserver/view/account.py
@@ -3,4 +3,5 @@ import uuid
 from flask import Blueprint, request, url_for, flash, redirect
 from flask import render_template
 from flask.ext.login import login_user, logout_user
-from flask.ext.wtf import Form, TextField, TextAreaField, PasswordField, validators, ValidationError
+from flask.ext.wtf import Form
+from wtforms import TextField, TextAreaField, PasswordField, validators, ValidationError
+from wtforms.validators import Required

 from bibserver.config import config
 import bibserver.dao as dao
```

## Running

First: 

1. You need to have ElasticSearch running in order to do anything.  Start a separate terminal and run `bin/elasticsearch` from inside the directory where it was unpacked.  It will typically runs on port 9200 -- doublecheck the output on your terminal and if that looks right, browse to **localhost:9200**.

Now, from within the bibserver virtualenv that you've set up and activity (working directory is `~/env/testdir/bibserver/`):

1. `python bibserver/web.py`
2. Browse to **localhost:5000**.

If all of that works, then you'll need to do a few next steps in the web browser to give yourself something to work with.  Browse to **localhost:5000/account/register** to create an account.  You'll be logged in automatically and an email will not be sent.  Browse to **localhost:5000/upload** and click "upload from your PC" to select a suitable file.  BibTeX formats is recognized automatically -- here's some [sample data](http://metameso.org/~joe/corneli.bib) that you can use if you like.

Once something is uploaded you can browse it at **localhost:5000/collections**.  From **localhost:5000/USERNAME/COLLECTION**, you can then click "Download as BibJSON" and inspect the results.

# Querying ElasticSearch

Expanding on the hints on [this page](http://exploringelasticsearch.com/searching_data.html), I see that you can find the schema (in JSON format) via

```sh
curl -X GET localhost:9200/bibserver/_mapping
```

And, then, you can get specific results back (using my sample data) in the following way:

```sh 
curl -X GET localhost:9200/bibserver/record/_search -d \
 '{"size": 3,"query": {"match": {"author.name": "McLuhan, Marshall"}}}'
```

Returns:

```javascript
{
   "took":4,
   "timed_out":false,
   "_shards":{
      "total":5,
      "successful":5,
      "failed":0
   },
   "hits":{
      "total":2,
      "max_score":4.564443,
      "hits":[
         {
            "_index":"bibserver",
            "_type":"record",
            "_id":"ef13eae6742ea8aa6ecaa950d42152d2",
            "_score":4.564443,
            "_source":{
               "publisher":"Sphere Books",
               "_last_modified":"20140614213719",
               "title":"Understanding media: The extensions of man",
               "url":"http://bibsoup.net/joe/joe_citations/mcluhan1994understanding",
               "author":[
                  {
                     "name":"McLuhan, Marshall",
                     "id":"McLuhanMarshall"
                  }
               ],
               "collection":"joe_citations",
               "year":"1964",
               "owner":"joe",
               "_created":"20140614213719",
               "_id":"ef13eae6742ea8aa6ecaa950d42152d2",
               "type":"book",
               "id":"mcluhan1994understanding"
            }
         },
         {
            "_index":"bibserver",
            "_type":"record",
            "_id":"95d501e85f24ec287fea08df419a9e43",
            "_score":3.19511,
            "_source":{
               "publisher":"Oxford University Press",
               "_last_modified":"20140614213719",
               "author":[
                  {
                     "name":"McLuhan, Marshall",
                     "id":"McLuhanMarshall"
                  },
                  {
                     "name":"Powers, Bruce R",
                     "id":"PowersBruceR"
                  }
               ],
               "url":"http://bibsoup.net/joe/joe_citations/global-village",
               "title":"The Global Village",
               "collection":"joe_citations",
               "year":"1989",
               "owner":"joe",
               "_created":"20140614213719",
               "_id":"95d501e85f24ec287fea08df419a9e43",
               "type":"book",
               "id":"global-village"
            }
         }
      ]
   }
}
```
