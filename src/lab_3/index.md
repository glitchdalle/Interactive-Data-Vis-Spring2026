---
title: "Lab 3: Mayoral Mystery"
toc: true
---

<style>
main {
  font-family: Arial, Helvetica, sans-serif;
}
</style>

# mayoral mystery

The campaign lost overall, but the data suggests that support was not evenly distributed across New York City. This dashboard looks at where the candidate performed best, how performance varied by income level, whether campaign effort was connected to turnout, and which issues voters responded to most strongly.

```js
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";
import * as topojson from "npm:topojson-client";

const nyc = await FileAttachment("data/nyc.json").json();
const results = await FileAttachment("data/election_results.csv").csv({typed: true});
const survey = await FileAttachment("data/survey_responses.csv").csv({typed: true});

const editorial = {
  ink: "#1f1f1f",
  paper: "#f7f4ee",
  blue: "#335c81",
  blueSoft: "#6c8fb3",
  red: "#8f3b3b"
};

const chartStyle = {
  background: editorial.paper,
  color: editorial.ink,
  fontFamily: "Arial, Helvetica, sans-serif"
};
```

```js
const districts = topojson.feature(nyc, nyc.objects.districts);

const resultsWithShare = results.map(d => ({
  ...d,
  boro_cd: String(d.boro_cd),
  candidate_share: d.votes_candidate / (d.votes_candidate + d.votes_opponent)
}));

const resultByDistrict = new Map(resultsWithShare.map(d => [String(d.boro_cd), d]));

function districtCode(feature) {
  const p = feature.properties;
  return String(
    p.boro_cd ??
    p.BoroCD ??
    p.borocd ??
    p.BORO_CD ??
    p.cd ??
    p.CD
  );
}

function districtNameFromCode(code) {
  const value = String(code);
  const boroughDigit = Number(value[0]);
  const districtNumber = Number(value.slice(1));
  const borough =
    boroughDigit === 1 ? "manhattan" :
    boroughDigit === 2 ? "bronx" :
    boroughDigit === 3 ? "brooklyn" :
    boroughDigit === 4 ? "queens" :
    boroughDigit === 5 ? "staten island" :
    "unknown";
  return `${borough} community district ${districtNumber}`;
}

const topDistrict = resultsWithShare.reduce((best, current) =>
  !best || current.candidate_share > best.candidate_share ? current : best, null
);
const topDistrictFeature = districts.features.find(d => districtCode(d) === topDistrict?.boro_cd);
const topDistrictCenter = topDistrictFeature ? d3.geoCentroid(topDistrictFeature) : null;
const topDistrictCallout = topDistrictCenter ? [{
  x1: topDistrictCenter[0],
  y1: topDistrictCenter[1],
  x2: topDistrictCenter[0] - 0.30,
  y2: topDistrictCenter[1],
  label: `top district:\n${districtNameFromCode(topDistrict.boro_cd)} (${topDistrict.boro_cd})`
}] : [];
```

## 1. Where Support Was Strongest

```js
Plot.plot({
  title: "vote share by district",
  style: chartStyle,
  marginTop: 40,
  marginRight: 30,
  marginBottom: 40,
  marginLeft: 30,
  projection: {
    domain: districts,
    type: "mercator"
  },
  color: {
    label: "Candidate vote share",
    scheme: "Greys",
    percent: true,
    legend: true
  },
  marks: [
    Plot.geo(districts, {
      fill: d => resultByDistrict.get(districtCode(d))?.candidate_share,
      stroke: "white",
      strokeWidth: 0.6,
      title: d => {
        const r = resultByDistrict.get(districtCode(d));
        return r ? `district ${r.boro_cd.toLowerCase()}` : "no data";
      },
      tip: true
    }),
    Plot.link(topDistrictCallout, {
      x1: "x1",
      y1: "y1",
      x2: "x2",
      y2: "y2",
      stroke: editorial.ink,
      strokeWidth: 1.1
    }),
    Plot.text(topDistrictCallout, {
      x: "x2",
      y: "y2",
      text: "label",
      textAnchor: "start",
      dx: 6,
      dy: -16,
      fontFamily: "Arial, Helvetica, sans-serif",
      fontSize: 12,
      fontWeight: 700
    })
  ]
})
```

The map shows that support was concentrated in Manhattan, with Manhattan Community District 10 (also known as district code 110) achieving the highest vote share of any district. Darker districts, mostly in Manhattan and parts of Brooklyn, reflect stronger candidate performance. Lighter districts, particularly in Staten Island and outer Queens, are where the campaign struggled. The geographic pattern is a useful starting point: Manhattan's strong showing suggests the core message already resonated there, while the lighter outer-borough districts point to places where the volume of outreach, or the framing of the message, may need to change.

## 2. Support by Income

```js
const incomePerformance = d3.rollups(
  resultsWithShare,
  v => d3.mean(v, d => d.candidate_share) * 100,
  d => d.income_category
).map(([income_category, avg_share_percent]) => ({income_category, avg_share_percent}));
```

```js
Plot.plot({
  title: "vote share by income",
  style: chartStyle,
  marginTop: 40,
  marginRight: 30,
  marginBottom: 50,
  marginLeft: 55,
  x: {label: "Income"},
  y: {label: "Vote share (%)", domain: [0, 70]},
  marks: [
    Plot.barY(incomePerformance, {
      x: "income_category",
      y: "avg_share_percent",
      fill: editorial.blue,
      stroke: "white",
      strokeWidth: 0.8,
      title: d => `${d.income_category} income districts\nAverage vote share: ${d.avg_share_percent.toFixed(1)}%`
    }),
    Plot.ruleY([0], {stroke: editorial.ink, strokeOpacity: 0.5})
  ]
})
```

<div style="font-family: sans-serif; font-weight: 700; text-transform: lowercase; margin-top: 0.25rem;">income ─────→</div>

The income breakdown tells a clear story. The candidate performed best in low-income districts, with roughly 57% vote share, followed by middle-income at around 48%, and weakest in high-income districts at around 28%. The message landed most effectively in communities where housing affordability and public transit are not abstract policy questions but daily realities. High-income districts represent the largest gap, and may require a different approach entirely.

## 3. Get Out the Vote vs Turnout

```js
Plot.plot({
  title: "doors knocked vs turnout",
  style: chartStyle,
  marginTop: 40,
  marginRight: 30,
  marginBottom: 50,
  marginLeft: 55,
  x: {label: "Doors knocked"},
  y: {label: "Turnout (%)"},
  marks: [
    Plot.dot(resultsWithShare, {
      x: "gotv_doors_knocked",
      y: "turnout_rate",
      fill: editorial.blueSoft,
      r: 4,
      opacity: 0.85,
      title: d => `district ${d.boro_cd.toLowerCase()}`,
      tip: true
    })
  ]
})
```

The scatter plot does not show a consistent relationship between doors knocked and turnout. Districts where the campaign knocked more doors did not reliably turn out more voters. The dots are spread wide, with no clear upward pattern. Before committing the same resources to door-knocking in the next cycle, the campaign should look carefully at where turnout actually improved and whether canvassing effort had anything to do with it.

## 4. Issue Alignment

```js
const surveyIssues = survey.flatMap(d => [
  {issue: "housing", alignment: d.affordable_housing_alignment},
  {issue: "transit", alignment: d.public_transit_alignment},
  {issue: "childcare", alignment: d.childcare_support_alignment},
  {issue: "tax", alignment: d.small_business_tax_alignment},
  {issue: "policing", alignment: d.police_reform_alignment}
]);
```

```js
const issueAlignment = d3.rollups(
  surveyIssues,
  v => d3.mean(v, d => d.alignment),
  d => d.issue
).map(([issue, average]) => ({issue, average}))
 .sort((a, b) => d3.descending(a.average, b.average));
```

```js
Plot.plot({
  title: "issue alignment (1-5)",
  style: chartStyle,
  x: {label: "Issue"},
  y: {label: "Score", domain: [1, 5]},
  marginTop: 40,
  marginRight: 30,
  marginBottom: 55,
  marginLeft: 55,
  marks: [
    Plot.barY(issueAlignment, {
      x: "issue",
      y: "average",
      fill: editorial.red
    }),
    Plot.text(issueAlignment, {
      x: "issue",
      y: "average",
      dy: -8,
      text: d => d.issue,
      fontWeight: "bold",
      fontFamily: "sans-serif",
      textAnchor: "middle"
    }),
    Plot.ruleY([1], {stroke: editorial.ink, strokeOpacity: 0.5})
  ]
})
```

Housing and transit each scored around 4.0 out of 5, the strongest alignment of any issues surveyed. Childcare landed in the middle. Small business tax policy scored lower. Police reform was the outlier, coming in near 1.5, meaning voters were significantly out of step with the candidate on that issue. Housing and transit are the natural pillars for the next campaign's message. The police reform position is worth revisiting before the next run.

## Recommendation


Three things stand out. First, the geographic base is real and worth building from. Manhattan showed up, and lower-income outer-borough districts share enough of the same issue profile to be winnable with the right investment. Second, the message already has its strongest pillars: housing and transit. Lead with those. The police reform alignment gap is the single clearest signal in the survey data, and it is worth taking seriously. Third, the GOTV strategy needs a second look. More doors knocked did not reliably mean more votes. The next run calls for targeted outreach in the right districts, not blanket volume everywhere.