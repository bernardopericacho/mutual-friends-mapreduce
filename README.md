Mutual Friends - mapreduce
==========================

Mutual Friends is a python-based application that uses Google MapReduce API. Social networks have huge amounts of users and calculating mutual friends it is a very expensive-time operation, very common and usually consulted. 
Using MapReduce API it will be another method to calculate them.

This application needs a .zip file with the users relations .txt files splitted to use the whole map-reduce capacity.
A .txt file may contain relations like:

Alberto - [Borja,Carlos,Damian]

Borja - [Alberto,Carlos,Damian,Eustaquio]

Carlos - [Alberto,Borja,Damian,Eustaquio]

Damian - [Alberto,Borja,Carlos,Eustaquio]

Eustaquio - [Borja,Carlos,Damian]

where the first name is the user and the list on its right are his friends. 

***Note that to use the whole map-reduce capacity it would be better to split all the relations in several files.***

Two diffent zip files are provided to use the web-based application.

This app has been uploaded to Google App Engine.

application link: http://map-reduce-friends.appspot.com/

*******************************************
Instructions to upload to Google App Engine
*******************************************
https://developers.google.com/appengine/docs/python/gettingstartedpython27/uploading
