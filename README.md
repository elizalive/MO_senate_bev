MO_senate_bev
=============

Intro
-----

This is a web scraping and data presentation project for one of my journalism classes. The goal is to create a bird's-eye-view of the Missouri State Senate. So far, what I am able to do is create, populate and continually update a database of all legislation that passes through the state senate for a given general assembly (two years), including the description, committee, sponsors and co-sponsors, actions and topics for each piece of legislation.

Dependencies
------------

* Python 2.x
* sqlite3
* [BeautifulSoup4](http://www.crummy.com/software/BeautifulSoup/ "BeautifulSoup4")
* [lxml parser](http://www.crummy.com/software/BeautifulSoup/bs4/doc/#installing-a-parser "lxml parser")
* [Requests](http://docs.python-requests.org/en/latest/ "Requests")

Scrapers
--------

There are two primary scripts -- db_seed.py and db_catchup.py -- which share a module of functions I wrote specifically for scraping the [Missouri State Senate website](http://www.senate.mo.gov/14info/BTS_Web/BillList.aspx?SessionType=R "Missouri State Senate website") (that's what scrapers.py is).

The db_seed.py is a Python script that:
* Creates the MO_senate sqlite database (and if it's already in your current directory, it creates an archive of the current one)
* Populates this database with all of the current Senate legislation, grabbing most the informaiton from [this list page](http://www.senate.mo.gov/14info/BTS_Web/BillList.aspx?SessionType=R "this list page") and [this details page](http://www.senate.mo.gov/14info/BTS_Web/Bill.aspx?SessionType=R&BillID=27723560 "this details page").

Note that, in even years, the db_seed.py will also gather this info for the previous year, so the list of bills could be over a thousand. And since we have to send as many as three requests to get all of the info about each bill, and since we want to be kind to other people's web servers and only make a request every few seconds, db_seed.py could take several hours to run. 

Which is why I'll try to keep a pretty current version of the sqlite db in this repo. So you should only have to run db_seed.py if something really wrong happens with the sqlite database and has to be completely re-built.

That's also why I wrote db_catchup.py, which only gets you the information missing in your local copy of the database, with fewer requests and much less time. Even if you're local copy of the data is a few weeks behind, it should only need to run for a few minutes.

Whenever you run db_catchup.py:
* It first checks the list of bills on the Senate website for any new ones that aren't currently in your database, and then pulls these down. 
* If it does find new bills, it also re-builds the topics table, the information for which is conveniently all available at [one page for each year](http://www.senate.mo.gov/14info/BTS_Web/Keywords.aspx?SessionType=R "one page per year"), requring only one or two requests. 
* It then checks for any new actions on the bills that were already in your database, and pulls all these down. The way it does this is by getting the most recent action date found in your current local copy of the data, and then checking [this page](http://www.senate.mo.gov/14info/BTS_Web/ActionDates.aspx?SessionType=R "this page") to see if there are any more recent dates of action.

There's a key assumption here: That for each date in which bill actions were previously collected, all of the actions were indeed collected into the database. To put it another way: if there's even one bill action for April 9, 2014 in your database, then db_catchup.py assumes that I have all the actions for that date. So if there any doubt, it's worth it to just delete the bill action records for any date that might be missing actions (and any date after that) and then run db_catchup.py

In order to verify that db_catchup.py is working as described above, I wrote another script, test_db_catchup.py, which basically just deletes the three most recent bills (and all of their associated records) and any actions for the most recent three days). It will tell you which bills were deleted and how many bill actions so that, after re-running db_catchup.py, you can verify that this information shows back up in the database. Incidentally, this test helped me catch a few bugs with how I was doing date filtering in sqlite.

One thing I want to be sure to have represented in this database is the most recent action date and description for each bill. I started out collecting this, but realized it would be a pan to try to keep up-to-date. So instead I figured out a SQL query that will return that using the records in the bills_actions table:

```sql
select a.bill_year, a.bill_type, a.bill_number, a.action_date, a.action_desc
from bills_actions as a
join 
(select bill_year, bill_type, bill_number, max(rowid) as last
from bills_actions
group by bill_year, bill_type, bill_number) as b
on a.rowid = b.last
```

Note this query assumes that the order in which the actions were inserted is roughly earliest to most recent (at least for any given bill), which is exactly how both of the scripts are working.
