---
date: 2023-07-19
title: Gravitational Waves optical counterpart (Part 2)
cardimage: fink_mm.png
---

We are pleased to introduce Fink_MM, which enables real-time analysis of any events sent by the General Coordinates Network (GCN).
<!--more-->


In the [previous post](https://fink-broker.org/2023-07-12_gw), we described a new service in Fink to inspect optical counterparts to gravitational wave events. This service is highly flexible as you can input any sky maps previously emitted by gravitational wave interferometers, but to find counterparts it relies on the alert data stored in the Fink database, that is updated once a day. Hence it is not designed for real-time analysis.

We are now pleased to introduce the second part of this service, which enables real-time analysis of any events sent by the General Coordinates Network (GCN).

## Pipeline Overview

Since now a year, a new service has been developed inside the fink-broker to create a real-time multi-messenger analysis pipeline.

![Fink_MM](images/Fink_MM_pipeline.png)
[Fink-MM](https://github.com/FusRoman/Fink_MM) extends the existing Fink pipeline and it performs a cross-match between the ZTF alerts processed by Fink and the General Coordinates Network (GCN) circulars. The purpose is to find the optical counterparts of astronomical events detected by the observatories sending circulars in the GCN as soon as possible. A daemon is always listening to the GCN circulars and storing them in a database. These circulars are then cross-matched with the ZTF alerts, and the results are stored in the Fink database. Simultaneously, the output is filtered and sent to the Fink community.

### Current listening observatories

Here is the list of observatories that we actively listen:

- Gamma-Ray: [Fermi](https://fermi.gsfc.nasa.gov/), [Swift](https://swift.gsfc.nasa.gov/about_swift/), [Integral](https://www.cosmos.esa.int/web/integral/home)
- X-Ray: Swift
- Optical: Swift
- Neutrino: [IceCube](https://icecube.wisc.edu/)
- Gravitational Waves: [LIGO](https://www.ligo.caltech.edu/), [VIRGO](https://www.virgo-gw.eu/), [KAGRA](https://gwcenter.icrr.u-tokyo.ac.jp/en/)

### Two workflows: online and offline

Fink-MM triggers two tasks each day if ZTF alerts are available. The first one is the online mode. It is a streaming job connected to the GCN daemon and the Fink pipeline and it joins both streams. The join condition looks at all the ZTF alerts emitted after the circulars trigger time and located inside the error region. The online mode also looks for the circulars emitted the day before. The typical latency for the online is:
- ZTF alerts latency: 15 minutes
- GCN circulars latency: few seconds, up to 1 hour
- Fink computation pipeline: 1 minute

On the other hand, the offline mode runs once at the end of a ZTF night and looks for all the circulars and ZTF alerts within a given time window (typically ten days) in case we missed events outside the observing period of ZTF.

## Fink-MM output

The current output schema for Fink-MM is the following:

{{% table %}}
| Field | Type | Contents |
| --- | ---| ---|
| `objectId` | string | Unique ZTF identifier of the object |
| `candid` | int | Unique ZTF identifier of the alert |
| `ztf_ra` | float | ZTF right ascension, in degree |
| `ztf_dec` | float | ZTF declination, in degree |
| `fid` | int | filter id (ZTF, g=1, r=2) |
| `jdstarthist` | float | first variation time at three sigma of the object, in JD |
| `rb`  | float | real bogus score |
| `jd`  | float | alert emission time |
| `instrument` | String | Triggering instruments (GBM, XRT, ...) |
| `observatory` | string | Triggering observatory (Fermi, Swift, IceCube, ...) |
| `triggerId` | string | Unique GCN identifier ([GCN viewer](https://heasarc.gsfc.nasa.gov/wsgi-scripts/tach/gcn_v2/tach.wsgi/) to retrieve quickly the notice) |
| `gcn_ra` | float | GCN Event right ascension, in degree |
| `gcn_dec` | float | GCN Event declination, in degree |
| `gcn_loc_error` | float | GCN error location, in arcminute |
| `triggerTimeUTC` | string | GCN TriggerTime in UTC |
| `p_assoc` | float | Serendipitous probability of associating the alerts with the GCN event |
| `fink_class` | string | [Fink Classification](https://fink-broker.readthedocs.io/en/latest/science/classification/) |
{{% /table %}}

In addition, there is also all the added value by Fink and the [science module](https://fink-broker.readthedocs.io/en/latest/science/added_values/), and for the online mode, we added some extra fields to help taking a decision:

{{% table %}}
| Field | Type | Contents |
| --- | --- | ---:|
| `delta_mag` | float | Difference of magnitude between the alert and the previous one |
| `rate` | float | Magnitude rate (magnitude/day) |
| `from_upper` | bool | If true, the last alert used for the difference magnitude is an upper limit |
| `start_vartime` | float | first variation time at five sigma of the object (in JD, only valid for 30 days) |
| `diff_vartime` | float | difference between `start_vartime` and `jd` (if above 30, `start_vartime` is not valid) |
{{% /table %}}

## Accessing the data in real-time

We already deployed several real-time topics focusing on GRB and GW optical counterparts. We list the available topic names below, and they are ready to be used.

### Available GRB topics

{{% table %}}
| Topic Name | Description |
| --- | ---|
| `fink_grb_bronze` | Alerts with a real bogus (rb) above 0.7, classified by Fink as an extragalactic event within the error location of a GRB event. |
| `fink_grb_silver` | Alerts satisfying the bronze filter with a `grb_proba` above 5 sigma. |
| `fink_grb_gold` | Alerts satisfying the silver filter with a `mag_rate` above 0.3 mag/day and a `rb` above 0.9. |
{{% /table %}}

### Available GW topics

{{% table %}}
| Topic Name | Description |
| --- | --- |
| `fink_gw_bronze` | Alerts with a real bogus (`rb`) above 0.7, classified by Fink as an extragalactic event within the error location of a GW event. |
{{% /table %}}

### How to listen to the topics?

First, register to get an account on the [livestream service](https://fink-broker.readthedocs.io/en/latest/services/livestream/), or update your existing topics using the [fink-client](https://github.com/astrolabsoftware/fink-client) (>=6.0):

```bash
fink_client_register \
    -username <USERNAME> \ # given privately
    -group_id <GROUP_ID> \ # given privately
    -mytopics <topic1 topic2 etc> \ # see above or https://fink-broker.readthedocs.io/en/latest/science/filters/topics/
    -servers <SERVER> \ # given privately, comma separated if several
    -maxtimeout 10 \ # in seconds
     --verbose
```

Finally poll alerts to get matching alerts in real-time:

```bash
# Get first matches for example on the `fink_gw_bronze` topic, and save them on disk
fink_consumer --display -limit 10 --save -outdir MMA_DB
+--------------+----------------+----------------+----------------------+-------------+------------+
|   ObjectId   | Classification |     Topic      |    Rate (mag/day)    | Observatory | Trigger ID |
+--------------+----------------+----------------+----------------------+-------------+------------+
| ZTF23aasnpzx |    Unknown     | fink_gw_bronze | -0.09833341715458073 |     LVK     |  S230718h  |
+--------------+----------------+----------------+----------------------+-------------+------------+
+--------------+----------------+----------------+---------------------+-------------+------------+
|   ObjectId   | Classification |     Topic      |   Rate (mag/day)    | Observatory | Trigger ID |
+--------------+----------------+----------------+---------------------+-------------+------------+
| ZTF23aasnirt |    Unknown     | fink_gw_bronze | -0.9822790407862867 |     LVK     |  S230718h  |
+--------------+----------------+----------------+---------------------+-------------+------------+
...
```

Some information will be displayed on the screen, and alerts will be saved on disk (Apache Avro format). You can easily inspect alerts using Fink-provided tools, e.g. in Python

```python
from fink_client.avroUtils import AlertReader

# load all alerts
r = AlertReader('MMA_DB')

# format into a Pandas DataFrame
pdf = r.to_pandas()

print(pdf[['objectId', 'fink_class', 'triggerId']])
        objectId    fink_class triggerId
0   ZTF23aasohuq       Unknown  S230718h
1   ZTF23aasnuqf       Unknown  S230718d
2   ZTF23aasnvfa       Unknown  S230718d
3   ZTF23aasogwq       Unknown  S230718q
4   ZTF23aasohuv       Unknown  S230718q
5   ZTF23aasogwq       Unknown  S230718h
6   ZTF23aasogwz       Unknown  S230718h
7   ZTF23aasoski       Unknown  S230718q
8   ZTF23aasnvze       Unknown  S230718d
9   ZTF23aasnoek  SN candidate  S230718d
10  ZTF23aasnvey       Unknown  S230718d
```

## What is next?

This is the first building block for real-time multi-messenger analysis in Fink. As time goes, we will deploy more filters, and associated topics. If you are using this service, your feedback is welcome! Do not hesitate to reach us to build your own filter based on Fink-MM output, or suggest us other incoming streams to correlate with the ZTF alert stream.
