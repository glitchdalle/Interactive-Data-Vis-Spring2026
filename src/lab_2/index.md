---
title: "Lab 2: Subway Staffing"
toc: true
---

# subway staffing dashboard

Looking at ridership, incident response times, and where staffing might be an issue for summer 2026.

```js
import * as d3 from "npm:d3";
import * as Plot from "npm:@observablehq/plot";

const incidents = await FileAttachment("data/incidents.csv").csv({typed: true});
const local_events = await FileAttachment("data/local_events.csv").csv({typed: true});
const upcoming_events = await FileAttachment("data/upcoming_events.csv").csv({typed: true});
const ridership = await FileAttachment("data/ridership.csv").csv({typed: true});
```

```js
const currentStaffing = {
  "Times Sq-42 St": 19,
  "Grand Central-42 St": 18,
  "34 St-Penn Station": 15,
  "14 St-Union Sq": 4,
  "Fulton St": 17,
  "42 St-Port Authority": 14,
  "Herald Sq-34 St": 15,
  "Canal St": 4,
  "59 St-Columbus Circle": 6,
  "125 St": 7,
  "96 St": 19,
  "86 St": 19,
  "72 St": 10,
  "66 St-Lincoln Center": 15,
  "50 St": 20,
  "28 St": 13,
  "23 St": 8,
  "Christopher St": 15,
  "Houston St": 18,
  "Spring St": 12,
  "Chambers St": 18,
  "Wall St": 9,
  "Bowling Green": 6,
  "West 4 St-Wash Sq": 4,
  "Astor Pl": 7
};
```

## 1. Events, Ridership, and Fare Increase

```js
const ridershipByDate = d3.rollups(
  ridership,
  v => d3.sum(v, d => d.entrances + d.exits),
  d => d.date
).map(([date, total]) => ({date, total}));

const eventsByDate = d3.rollups(
  local_events,
  v => d3.sum(v, d => d.estimated_attendance),
  d => d.date
).map(([date, total]) => ({date, total}));

const fareDate = new Date("2025-07-15");
```

```js
Plot.plot({
  title: "Ridership Over Time with Events and Fare Increase",
  x: {
    label: null
  },
  y: {
    label: "Daily entries + exits"
  },
  marginTop: 40,
  marks: [
    Plot.lineY(ridershipByDate, {
      x: "date",
      y: "total",
      stroke: "#374151",
      strokeWidth: 2.5
    }),

    Plot.dot(eventsByDate, {
      x: "date",
      y: d3.min(ridershipByDate, d => d.total) - 2000,
      r: 3,
      fill: "#9CA3AF",
    }),

    Plot.ruleX([fareDate], {
      stroke: "#DC2626",
      strokeWidth: 1.5
    }),

    Plot.text([{date: fareDate, total: d3.max(ridershipByDate, d => d.total)}], {
      x: "date",
      y: "total",
      text: ["Fare increase"],
      dy: -10,
      dx: 35,
      fill: "#DC2626",
      fontWeight: "bold",
      fontSize: 12
    })
  ]
})
```

Ridership tends to increase around local events, so event traffic looks like a stronger driver than the July fare increase. There isn’t a clear drop or shift right after the fare hike.

## 2. Response Time by Station

```js
const responseByStation = d3.rollups(
  incidents,
  v => d3.mean(v, d => d.response_time_minutes),
  d => d.station
).map(([station, avg]) => ({station, avg}))
 .sort((a, b) => d3.descending(a.avg, b.avg));
 const slowest3 = responseByStation.slice(0, 3);
```

```js

Plot.plot({
  title: "Average Response Time by Station",
  x: {label: "Minutes"},
  y: {label: null},
  marginLeft: 140,
  marks: [
    Plot.barX(responseByStation, {
      x: "avg",
      y: "station",
      sort: {y: "-x"},
      fill: d => slowest3.some(s => s.station === d.station) ? "#7C3AED" : "#D1D5DB"
    }),
    Plot.ruleX([0]),

    Plot.text(responseByStation, {
      x: "avg",
      y: "station",
      text: d => d.avg.toFixed(1),
      dx: 8,
      textAnchor: "start",
      fontSize: 11,
      fill: "#374151"
    })
  ]
})
```

Response times vary a lot by station, with a few locations clearly slower than the rest. The stations at the top of the chart stand out enough that they’re worth a closer look, since longer response times often point to staffing issues or more complex station layouts.

## 3. Staffing Needs for Summer 2026

```js
const upcomingByStation = d3.rollups(
  upcoming_events,
  v => d3.sum(v, d => d.expected_attendance),
  d => d.nearby_station
).map(([station, total]) => ({
  station,
  total,
  staff: currentStaffing[station],
  pressure: total / currentStaffing[station]
}))
.sort((a, b) => d3.descending(a.pressure, b.pressure));

const top3 = upcomingByStation.slice(0, 3);
```

```js
Plot.plot({
  x: {
    label: "Projected attendees per current staff member",
    domain: [0, 8000]
  },
  title: "Projected Staffing Pressure for Summer 2026",
  y: { label: null },
  marginLeft: 140,
  marginRight: 80,
  marks: [
    Plot.barX(upcomingByStation, {
      x: "pressure",
      y: "station",
      sort: { y: "-x" },
      fill: d => top3.some(t => t.station === d.station) ? "#B91C1C" : "#D1D5DB"
    }),
    Plot.ruleX([0]),
    Plot.text(upcomingByStation, {
      x: "pressure",
      y: "station",
      text: d => Math.round(d.pressure).toLocaleString(),
      dx: 8,
      textAnchor: "start",
      fontSize: 11,
      fill: "#374151"
    })
  ]
})
```

The three red bars are the stations that look most understaffed going into summer. Canal St is way out ahead of everything else -- it only has 4 staff but a lot of projected attendance, so that ratio is going to be a problem.
