# Half Moon Bay Pumpkin Festival traffic on Saturday 15th October 2022

A scraper that records estimated driving times according to the Google Maps [Directions API](https://developers.google.com/maps/documentation/directions/overview) between the towns of El Granada and Half Moon Bay during the annual Half Moon Bay Art & Pumpkin Festival.

The scraper runs [this script](.github/workflows/scrape.yml) every five minutes and records the output of the JSON API in these files:

- [one.json](one.json) records directions from El Granada, CA, USA to Half Moon Bay, CA, USA.
- [two.json](two.json) records directions from Half Moon Bay, CA, USA to El Granada, CA, USA.

The actual points used are these:

- Half Moon Bay: https://www.google.com/maps/search/FH78%2BQJ+Half+Moon+Bay,+CA,+USA
- El Granada: https://www.google.com/maps/search/GG49%2BCH+El+Granada+CA,+USA

## Analyzing the results

Use [git-history](https://github.com/simonw/git-history) to load the resulting commits into SQLite like this:

```
git-history file hmb.db one.json \
--convert '
try:
    duration_in_traffic = json.loads(content)["routes"][0]["legs"][0]["duration_in_traffic"]["value"]
    return [{"id": "one", "duration_in_traffic": duration_in_traffic}]
except Exception as ex:
    return []
' \
  --full-versions \
  --id id

git-history file hmb.db two.json \
--convert '
try:
    duration_in_traffic = json.loads(content)["routes"][0]["legs"][0]["duration_in_traffic"]["value"]
    return [{"id": "two", "duration_in_traffic": duration_in_traffic}]
except Exception as ex:
    return []
' \
  --full-versions \
  --id id --namespace item2
```
This SQL query then joins the data together to provide time of day and time in minutes in both directions:
```sql
with item1 as (
  select
    time(datetime(commits.commit_at, '-7 hours')) as t,
    duration_in_traffic / 60 as mins_in_traffic
  from
    item_version
    join commits on item_version._commit = commits.id
  order by
    commits.commit_at
),
item2 as (
  select
    time(datetime(commits.commit_at, '-7 hours')) as t,
    duration_in_traffic / 60 as mins_in_traffic
  from
    item2_version
    join commits on item2_version._commit = commits.id
  order by
    commits.commit_at
)
select
  item1.*,
  item2.mins_in_traffic as mins_in_traffic_other_way
from
  item1
  join item2 on item1.t = item2.t
```
The `-7 hours` bit is needed because the original commit dates are recorded as UTC, but I need to display them in local Pacific time.

We pasted the results into [this Google Sheet](https://docs.google.com/spreadsheets/d/1JOimtkugZBF_YQxqn0Gn6NiIhNz-OMH2rpOZtmECAY4/edit) and plotted this chart:

<img width="925" alt="A chart showing the two lines over time" src="https://user-images.githubusercontent.com/9599/196016852-22193d18-7935-4941-9921-8c4deb591da5.png">
