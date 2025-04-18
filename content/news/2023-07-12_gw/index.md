---
date: 2023-07-11
title: Gravitational Waves optical counterpart (Part 1)
cardimage: gw.png
---

The fourth observing run of the gravitational wave detectors have already begun, and you might wonder if there are optical counterparts from Fink/ZTF to their detection? We are happy to introduce a new service to quickly explore potential matches.
<!--more-->


### Portal search

The fourth observing run of the gravitational wave detectors have already begun, and you might wonder if there are optical counterparts from Fink/ZTF to their detection? We are happy to introduce a new service to quickly explore potential matches: [https://fink-portal.org/gw](https://fink-portal.org/gw).

The idea is simple: you enter a superevent name on the left as well as a credible level, and hit the button. The alerts falling into the sky map will be shown in the table, and displayed on the skymap. Note that we consider only alerts that started varying within a time window of -1 to +6 days after the gravitational wave detectors trigger.

The table contains a few columns to guide the search, but you can easily access individual objects by clicking on their name if you need further inspection. Regarding the skymap, you can open the top left menu, and (de)select classes of your choice. The classes are provided by Fink, using all the added values from our classifiers and catalog crossmatches. Note that you can directly run a query from the URL, e.g. [https://fink-portal.org/gw?credible_level=0.2&event_name=S230709bi](https://fink-portal.org/gw?credible_level=0.2&event_name=S230709bi).

![gw](images/gw_portal_fink.png)

### Gravitational wave API

The API will let you do the same, but you have more freedom in the input skymap. You can send any skymap from the Gravitational-Wave Candidate Event Database (GraceDB), but also any custom skymap that will use the same format. Here is an example for a O4 candidate:

```python
import io
import requests
import pandas as pd

# GW credible region threshold to look for. Note that the values in the resulting
# credible level map vary inversely with probability density: the most probable pixel is
# assigned to the credible level 0.0, and the least likely pixel is assigned the credible level 1.0.
# Area of the 20% Credible Region:
credible_level = 0.2

# Query Fink
r = requests.post(
    'https://fink-portal.org/api/v1/bayestar',
    json={
        'event_name': 'S230709bi',
        'credible_level': credible_level,
        'output-format': 'json'
    }
)

pdf = pd.read_json(io.BytesIO(r.content))
```

Fink returns all alerts emitted within `[-1 day, +6 day]` from the GW event inside the chosen credible region.
Alternatively, you can input a sky map retrieved from the GraceDB event page, or distributed via GCN.
Concretely on [S230709bi](https://gracedb.ligo.org/superevents/S230709bi/view/):

```python
import io
import requests
import pandas as pd

# LIGO/Virgo probability sky maps, as gzipped FITS (bayestar.fits.gz)
data = requests.get('https://gracedb.ligo.org/api/superevents/S230709bi/files/bayestar.fits.gz')

# Query Fink
r = requests.post(
    'https://fink-portal.org/api/v1/bayestar',
    json={
        'bayestar': str(data.content),
        'credible_level': 0.2,
        'output-format': 'json'
    }
)

pdf = pd.read_json(io.BytesIO(r.content))
```

You can then quickly inspect what kind of match is available:

```python
pdf.groupby('d:classification').size()
d:classification
SN candidate               2
Solar System MPC           2
Solar System candidate    42
Unknown                   35
```

More information at [https://fink-portal.org/api](https://fink-portal.org/api).

### Schema change

Note that the schema of the data returned by the `api/v1/bayestar` endpoint changed. It is lightweight, and more focused on the GW search. The type and description of columns can be found at https://fink-portal.org/api/v1/columns, plus two additional columns:

| Name | type | Description |
|--|--|--|
| 'd:classification' | str | classification infered from Fink |
| 'v:jdstartgw' | double | Trigger time (Julian Date) for the gravitational wave detectors |


### What is next?

This service relies on the alert data stored in the Fink database, that is updated once a day. Hence it is not designed for real-time analysis. But... wait for the Part 2 of this post ;-)
