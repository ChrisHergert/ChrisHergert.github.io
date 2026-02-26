---
layout: post
title: "Scraping Weather Prediction Data - Part I"
description: "Walking through setting up the data lake table and scraping script for weather prediction data"
tags:
  - Blogging
---

## Scraping Weather Prediction Data - Part I

So, now that I've got a MySQL instance running locally on my homelab, I've started scraping raw weather data into the database. This is based on the Open-Meteo weather prediction engine, mostly because they make their web API available for public consumption.

This API is super convenient to hit with a basic HTTP query out of the Niquests module in Python (a personal favorite, created as a drop-in replacement for Requests due to all the drama around the Rq development that seems to have stalled out, although I'd be lying if I said that [HTTPX](https://www.python-httpx.org/) wasn't also a strong contender), so my general plan is to pull down the next week's hourly temperature forecasts and logging those. The goal is to, eventually, do an analysis of what how the prediction error decreases as time to the actual hour/date decreases. Eventually, I'll likely factor in other variables if I want to turn this into a more predictive model. Maybe after I collect a full year's data, I'll do some kind of paired binary classifier model that anticipates the probability that a given day will be above or below the day-before temp, based on a series of factors like daytime/nighttime and the predictions for that hour from the prior seven days.

---

####  Setting up the tables

Because this is event data, it's going to eventually be a fact table; the task today is just to get the data into our lake-level tables so that we can eventually batch-transform these data up into the warehouse level tables.

So we start with this query to set up the table:

```SQL
# Creating the basics for a raw fact table
CREATE TABLE coredb.WeatherPredictions (event_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY, Prediction_Time DATETIME, oDate DATE, Temp DOUBLE, Location VARCHAR(30) );
```

Since we're doing this in a Dockerize MySQL instance, you can either do this via the above command in a console, or you can simply wrap it into a MySQL feed command using the MySQL CLI:

```Powershell
docker exec my-mysql mysql -u root -p<password> -e 'CREATE TABLE coredb.WeatherPredictions (event_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY, Prediction_Time DATETIME, oDate DATE, Temp DOUBLE, Location VARCHAR(30) );'
```

It's not ideal to be using the password in the docker call, but it'll do for the time being.

---

#### Setting up the scraper

Next up, we've got to write a Python script that will harvest data from the Open-Meteo web API, parse it into a DataFrame, and then load it to the database.

First up, let's create a Venv for this project. Don't muddy your core Python installation's tool chain, it's just going to hurt you down the road. Been there, done that.

```powershell
# Create your venv
python -m venv database_accces_venv

# Next, add the venv's Python installation to the top of your machine's PATH variable so that you're installing packages to this VENV rather than to your core Python installation. If you're on a Windows machine, your call will use the "call" command rather than "source", which is for anyone running *NIX
source database_access_venv/bin/activate

# Next up, install some libraries into that venv.
pip install niquests pandas pprint
```


Now for the fun part: start writing some python code. I put this in a script called temp_scraper.py, and you can see from the shebang that I'm calling that new venv if I want to run this script as an executable. Spoiler alert: that's what we're going to do later. 

You can see that the Open-Meteo API takes location-specifying longitude and latitude parameters. I use the params for where I live, in Dallas, and then I add the actual location name as a column. We'll turn this into a location key in the next level, the data warehouse. Data Lakes will naturally tend to have a lot of text values that haven't been cleaned, parsed, or converted to keys yet.

```python
#!/<absolute file path to venv>/database_access_venv/bin/python3

### Import necessary libraries
import niquests as niq
# import requests as rq
import pandas as pd
import json
from datetime import datetime, date, timedelta
from dateutil import parser
from pprint import pprint
from sqlalchemy import create_engine
import pymysql # This enables us to use the mysql+pymysql construction in the connection string.


if __name__ == '__main__':
        ### Construct the request (can also be done with the requests library, but niquests is the newer, drop-in iteration)
        params = {'latitude': <latitude in decimal format; note that S is negative in this representation>,
                'longitude': <longitude in decimal format, same negative format as for longitude, but here it's W that's negative, e.g. Dallas' 96.7970 W. becomes -96.7970>
                'hourly': 'temperature_2m'}
        url = 'https://api.open-meteo.com/v1/forecast'
        resp = niq.get(url, params=params)

        ### Extract the key columns from the returned payload
        times = list(pd.json_normalize(json.loads(resp.text))['hourly.time'][0])
        temps = list(pd.json_normalize(json.loads(resp.text))['hourly.temperature_2m'][0])

        ### Compile the data frame
        oDate = date.today()
        times2 = [parser.isoparse(x).strftime("%Y/%m/%d %H:%M:%S") for x in times]
        temps2 = [round((x*1.8)+32,3) for x in temps]
        df = pd.DataFrame({'Prediction_time':times2, 'Temp':temps2, 'oDate':oDate, 'Location': 'Dallas'})

        ### Initialize the params for the database connection.
        USR = '<username>'
        PWD = '<user password>'
        HOST = '<hostname or service IP>' # I use the service-specific IP address here, which is a terrible idea, but if you're using a docker-compose then you can use the service name from your docke>
        PORT = 3306 #If you assigned a different listening port to map to 3306 in your docker-run for setting up your MySQL service, use that other listening port instead.
        DB_NAME = 'coredb'
        TBL_NAME = 'WeatherPredictions'

        ### Create the connection and attempt to insert data
        engine = create_engine(f'mysql+pymysql://{USR}:{PWD}@{HOST}:{PORT}/{DB_NAME}')

        try:
                df.to_sql(name='WeatherPredictions', con=engine, if_exists='append', index=False) ### We include index=False to keep Python from trying to write the index as a column.
                print("Successful write.")
        except Exception as e:
                print(f"An error occurred: {e}") 
```

And voila, run that and you'll load the next seven days' hourly info into your database table.
A few things to note: your date formatting (for the times2 variable) have to match what MySQL expects for a datetime; some people may be accustomed to databases that will do a little more pattern-recognition to deduce what your date *appears* that it should be, but MySQL's not interested in doing that. You either feed it the datetime formate it expects, or you get to try again. I was originally using the more readable "%m/%d/%Y %H:%M:%S" format because I planned to feed these into a string column and do the text-to-datetime processing in the transform step up to the warehouse layer, but this ended up just being easier to bring them in right away as datetimes.

--- 
Scheduling Your Job

This is just going to be a quick Cron job, since I don't have a full orchestrator set up. Someday I might set up Airflow, since that's one of my favorite tools I've used in my career, but I've never used it without the GUI, and the goal is to do all of this homelabbing without a GUI. 
Anyway, on to Cron. If you've never set up a Cron job before, run `cron -e` and select your IDE of choice (I prefer Nano, since I'm not a masochist, but Vim is also an option for those of you who are). You'll be presented with this:

```powershell
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command

```

I know Cron can be intimidating to look at, but that howto right there at the top tells you everything, and this [nice reference](https://adminschoice.com/crontab-quick-reference/) will explain things even more if you need it. To that end, I want my job to run every night at midnight, with no limitations on the DoM, DoW, or the particular month, so my cron schedule for this job looks like this:

```powershell
# m h  dom mon dow   command
0 0 * * * /home/cjh1/Documents/database_access_venv/project_files/temp_scraper.py
```

And that's about the sum of it. Save the crontab, and your job will now be in production on your homelab, scraping weather data for your location of choice!



