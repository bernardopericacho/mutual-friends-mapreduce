#!/usr/bin/env python
#
# Copyright 2011 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

""" This is a sample application that tests the MapReduce API.

It does so by allowing users to upload a zip file containing plaintext files
and perform some kind of analysis upon it. Currently three types of MapReduce
jobs can be run over user-supplied input data: a WordCount MR that reports the
number of occurrences of each word, an Index MR that reports which file(s) each
word in the input corpus comes from, and a Phrase MR that finds statistically
improbably phrases for a given input file (this requires many input files in the
zip file to attain higher accuracies)."""

__author__ = """aizatsky@google.com (Mike Aizatsky), cbunch@google.com (Chris
Bunch)"""

import datetime
import jinja2
import logging
import re
import urllib
import webapp2
import urllib2

from google.appengine.ext import blobstore
from google.appengine.ext import db

from google.appengine.ext.webapp import blobstore_handlers

from mapreduce.lib import files
from google.appengine.api import taskqueue
#from google.appengine.api import users

from mapreduce import base_handler
from mapreduce import mapreduce_pipeline
from mapreduce import operation as op
from mapreduce import shuffler

class User(db.Model):
  name = db.StringProperty()

class MutualFriends(db.Model):
  userA = db.StringProperty()
  userB = db.StringProperty()
  mutual = db.StringProperty()

class FileMetadata(db.Model):
  """A helper class that will hold metadata for the user's blobs.

  Specifially, we want to keep track of who uploaded it, where they uploaded it
  from (right now they can only upload from their computer, but in the future
  urlfetch would be nice to add), and links to the results of their MR jobs. To
  enable our querying to scan over our input data, we store keys in the form
  'user/date/blob_key', where 'user' is the given user's e-mail address, 'date'
  is the date and time that they uploaded the item on, and 'blob_key'
  indicates the location in the Blobstore that the item can be found at. '/'
  is not the actual separator between these values - we use '..' since it is
  an illegal set of characters for an e-mail address to contain.
  """

  __SEP = ".."
  __NEXT = "./"

  owner = db.StringProperty()
  filename = db.StringProperty()
  uploadedOn = db.DateTimeProperty()
  source = db.StringProperty()
  blobkey = db.StringProperty()
  findingfriends_link = db.StringProperty()

  @staticmethod
  def getFirstKeyForUser(username):
    """Helper function that returns the first possible key a user could own.

    This is useful for table scanning, in conjunction with getLastKeyForUser.

    Args:
      username: The given user's e-mail address.
    Returns:
      The internal key representing the earliest possible key that a user could
      own (although the value of this key is not able to be used for actual
      user data).
    """

    return db.Key.from_path("FileMetadata", username + FileMetadata.__SEP)

  @staticmethod
  def getLastKeyForUser(username):
    """Helper function that returns the last possible key a user could own.

    This is useful for table scanning, in conjunction with getFirstKeyForUser.

    Args:
      username: The given user's e-mail address.
    Returns:
      The internal key representing the last possible key that a user could
      own (although the value of this key is not able to be used for actual
      user data).
    """

    return db.Key.from_path("FileMetadata", username + FileMetadata.__NEXT)

  @staticmethod
  def getKeyName(username, date, blob_key):
    """Returns the internal key for a particular item in the database.

    Our items are stored with keys of the form 'user/date/blob_key' ('/' is
    not the real separator, but __SEP is).

    Args:
      username: The given user's e-mail address.
      date: A datetime object representing the date and time that an input
        file was uploaded to this app.
      blob_key: The blob key corresponding to the location of the input file
        in the Blobstore.
    Returns:
      The internal key for the item specified by (username, date, blob_key).
    """

    sep = FileMetadata.__SEP
    return str(username + sep + str(date) + sep + blob_key)



"""**************************************** WEB ************************** """


class WaitHandler(webapp2.RequestHandler):
  def get(self):
    pipeline_id = self.request.get('pipeline')
    pipeline = mapreduce_pipeline.MapreducePipeline.from_id(pipeline_id)
    if pipeline.has_finalized:
	self.redirect("/")
    else:
        logging.error("Running")

class IndexHandler(webapp2.RequestHandler):
  """The main page that users will interact with, which presents users with
  the ability to upload new data or run MapReduce jobs on their existing data.
  """
  template_env = jinja2.Environment(loader=jinja2.FileSystemLoader("templates"),
                                    autoescape=True)
  def get(self):
    
    #user = users.get_current_user()
    #username = user.nickname()
    username = "Alberto"
    arch = db.GqlQuery("SELECT * FROM FileMetadata")
    archivo = len([result for result in arch])
    logging.error("Archivos %s ", archivo)

    resul = db.GqlQuery("SELECT * FROM MutualFriends")
    res = len([s for s in resul])
    logging.error("Mutual Friends %s ", res)

    mutual = db.GqlQuery("SELECT userB,mutual FROM MutualFriends WHERE userA = :1", username)

    tot = db.GqlQuery("SELECT name FROM User WHERE name != :1", username)
    rest = [t for t in tot]
    anyone = len(rest)
    logging.error("Users %s ", anyone+1)

    dev=[]
    for x in rest:
	none=[]
	for y in mutual:
		if x.name==y.userB:
			none.append(y.mutual)
	dev.append([x.name,len(none),none])

    common = db.GqlQuery("SELECT userB FROM MutualFriends WHERE userA = :1", username)
    com = len([t for t in common])
    calcular = 0
    subir = 0
    cambiar = 0
    if (archivo==0 and res==0):
	subir = 1
    if (archivo==1 and res==0):
	calcular = 1
    if (archivo==1 and res>0):
        cambiar = 1
    
    if len(rest)==0:
	fill_users()
	self.redirect("/")
	
    if username:
	mate = db.GqlQuery("SELECT * FROM User WHERE name = :1 ",username)
	
    if not mate:
        u = User(name=username)
        u.put()

    q = FileMetadata.all()
    results = q.fetch(10)

    items = [result for result in results]
    length = len(items)

    upload_url = blobstore.create_upload_url("/upload")
	
    url = ""
    url_linktext=""
    """if user:
	url = users.create_logout_url(self.request.uri)
        url_linktext = 'Logout'
	
    else:
	url = users.create_login_url(self.request.uri)
        url_linktext = 'Login'"""
     
    self.response.out.write(self.template_env.get_template("index.html").render(
        {"username": username,
         "items": items,
         "length": length,
         "upload_url": upload_url,
	 "url": url,
         "url_linktext": url_linktext,
	 "rest": rest,
         "arch": arch,
	 "subir": subir,
         "cambiar": cambiar,		
         "calcular": calcular,
         "com": com,
	 "dev": dev,
	 "common": common}))


  def post(self):
    filekey = self.request.get("filekey")
    blob_key = self.request.get("blobkey")

    if self.request.get("finding_friends"):
      pipeline = FindingFriendsPipeline(filekey, blob_key)
      pipeline.start()
      logging.error("PIPELINE: %s", pipeline.pipeline_id)
      self.redirect('/')
    elif self.request.get("change"):
      q = db.GqlQuery("SELECT * FROM FileMetadata") 
      db.delete(q)
      q = db.GqlQuery("SELECT * FROM MutualFriends") 
      db.delete(q)
      self.redirect("/")    

def fill_users():
  s = ["Alberto","Borja","Carlos","Damian","Eustaquio"]
  r = [["Borja","Carlos","Damian"],["Alberto","Carlos","Damian","Eustaquio"],["Alberto","Borja","Damian","Eustaquio"],["Alberto","Borja","Carlos","Eustaquio"],["Borja","Carlos","Damian"]]

  for i,user in enumerate(s):
  	u = User(name=user)
        u.put()
        """friends = r[i]
	for x in friends:
		f = Friend(name=user,friend=x)
		f.put()"""

def split_into_sentences(s):
  """Split text into list of sentences."""
  s = re.sub(r"\s+", " ", s)
  s = re.sub(r"[\\.\\?\\!]", "\n", s)
  return s.split('\n',)

def split_into_words(s):
  """Split a sentence into list of words."""
  s = re.sub(r"\W+", " ", s)
  s = re.sub(r"[_0-9]+", " ", s)
  return s.split()

def split_into_people(s):
  """Split a sentence into list of words."""
  s = re.sub(r" - ", "", s)
  s = re.sub(r"\]|\[", "", s)
  return s.split(',',)

def split_lines(s):
  """Split text into friends"""
  return s.split('\n',)

def intersection(a):
	result = set(a[1])
	for element in a:
		result = result & set(element) 
	return list(result)

def finding_friends_map(data):
  """Finding Friends map function."""
  (entry, text_fn) = data
  text = text_fn()
  logging.debug("Got %s", entry.filename)
  for s in split_lines(text):
	if len(s) != 0:
		owner = re.sub(r' -.*$', "", s)
		others = re.sub(owner, "", s)
		l = split_into_people(others)
		lcopy = l[:]
		for name in l:	
			if owner > name:
				key = [name,owner]
			else:
				key = [owner,name]
			yield (key, lcopy)
		
	
def finding_friends_reduce(key, values):
  """Word count reduce function."""
  logging.error("KEY %s VALUES %s ", key, values)
  l1 = split_into_people(values[0])
  l2 = split_into_people(values[1])
  result = []
  for x in l1:
	item1 = re.sub(r' ', "", x)
	for y in l2:
		item2 = re.sub(r' ', "", y)
		if item1==item2:
			result.append(item1)
  #logging.error("KEY %s RESULT %s ", key, result)
  yield "%s: %s\n" % (key,result)

"""**************************************** FINDING FRIENDS CLASS ************************** """

class FindingFriendsPipeline(base_handler.PipelineBase):
  """Finding Friends

  Args:
    blobkey: blobkey to process as string. Should be a zip archive with
      text files inside.
  """

  def run(self, filekey, blobkey):
    logging.debug("filename is %s" % filekey)
    output = yield mapreduce_pipeline.MapreducePipeline(
        "finding_friends",
        "main.finding_friends_map",
        "main.finding_friends_reduce",
        "mapreduce.input_readers.BlobstoreZipInputReader",
        "mapreduce.output_writers.BlobstoreOutputWriter",
        mapper_params={
            "blob_key": blobkey,
        },
        reducer_params={
            "mime_type": "text/plain",
        },
        shards=16)
    yield StoreOutput("FindingFriends", filekey, output)

"""******************************************  CLASS *********************************** """

class StoreOutput(base_handler.PipelineBase):
  """A pipeline to store the result of the MapReduce job in the database.

  Args:
    mr_type: the type of mapreduce job run (e.g., WordCount, Index)
    encoded_key: the DB key corresponding to the metadata of this job
    output: the blobstore location where the output of the job is stored
  """

  def run(self, mr_type, encoded_key, output):
    key = db.Key(encoded=encoded_key)
    m = FileMetadata.get(key)
    if mr_type == "FindingFriends":
      m.findingfriends_link = output[0]
      f = MutualFriends(userA="A",userB="B",mutual="")
      f.put()
      s = re.sub(r'/blobstore/', "", output[0])
      bob = str(urllib.unquote(s)).strip()
      blob_info = blobstore.BlobInfo.get(bob) 
      blob_info.open()
      string = blob_info.open().read()

      for line in split_lines(string):
         if len(line)!=0:
         	s = re.sub(r" |\'", "",line)
         	r = re.sub(r'"',"",s)
		data = s.split(':',)
		users = re.sub(r"\[|\]", "",data[0])
		resto = re.sub(r'"|\[|\]', "",data[1])
		separados = users.split(',',)
		f = MutualFriends(userA=separados[0],userB=separados[1],mutual=resto)
         	f.put()
		f = MutualFriends(userA=separados[1],userB=separados[0],mutual=resto)
         	f.put()
    m.put()

class UploadHandler(blobstore_handlers.BlobstoreUploadHandler):
  """Handler to upload data to blobstore."""

  def post(self):
    source = "uploaded by user"
    upload_files = self.get_uploads("file")
    blob_key = upload_files[0].key()
    name = self.request.get("name")

    #user = users.get_current_user()

    #username = user.nickname()
    date = datetime.datetime.now()
    str_blob_key = str(blob_key)
    key = FileMetadata.getKeyName("Alberto", date, str_blob_key)

    m = FileMetadata(key_name = key)
    m.owner = "Alberto"
    m.filename = name
    m.uploadedOn = date
    m.source = source
    m.blobkey = str_blob_key
    m.put()
    self.redirect("/")


class DownloadHandler(blobstore_handlers.BlobstoreDownloadHandler):
  """Handler to download blob by blobkey."""

  def get(self, key):
    logging.error("key is %s" % key)
    key = str(urllib.unquote(key)).strip()
    logging.error("key is %s" % key)
    blob_info = blobstore.BlobInfo.get(key)
    self.send_blob(blob_info)

"""**********************************************************************************"""

app = webapp2.WSGIApplication(
    [
        ('/', IndexHandler),
        ('/upload', UploadHandler),
        (r'/blobstore/(.*)', DownloadHandler),
    ],
    debug=True)
