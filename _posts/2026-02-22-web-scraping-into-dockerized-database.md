## On Scraping Weather Prediction Data

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

Now for the fun part: start writing some python code. I put this in a script called temp_sc

```python
#!/home/cjh1/Documents/database_access_venv/bin/python3

### Import necessary libraries
import niquests as niq
# import requests as rq
import pandas as pd
import json
from datetime import datetime, date, timedelta
from dateutil import parser
from pprint import pprint

if __name__ == '__main__':
        ### Construct the request (can also be done with the requests library, but niquests is the newer, drop-in iteration)
        params = {'latitude': 32.9343,
                'longitude': -97.0781,
                'hourly': 'temperature_2m'}
        url = 'https://api.open-meteo.com/v1/forecast'
        resp = niq.get(url, params=params)

        ### Extract the key columns from the returned payload
        times = list(pd.json_normalize(json.loads(resp.text))['hourly.time'][0])
        temps = list(pd.json_normalize(json.loads(resp.text))['hourly.temperature_2m'][0])

        ### Compile the data frame
        eDate = date.today()
        times2 = [parser.isoparse(x).strftime("%m/%d/%Y %H:%M:%S") for x in times]
        temps2 = [round((x*1.8)+32,3) for x in temps]
        df = pd.DataFrame({'Time':times2, 'Temp':temps2, 'eDate':eDate})

        pprint(df)
```
