---
title: "Lab 1: Passing Pollinators"
toc: true
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README).

<!--importin the data :)-->

```js
const data = await FileAttachment("data/pollinator_activity_data.csv").csv({typed: true});
```

```js
import * as Plot from "npm:@observablehq/plot";
```
<!--body mass chart-->

```js
Plot.plot({
  title: "Average Body Mass by Pollinator Species",
  x: {label: "Pollinator Species"},
  y: {label: "Average Body Mass (g)"},
  marks: [
    Plot.barY(data, {
      x: "pollinator_species",
      y: "avg_body_mass_g",
      fill: "pollinator_species"
      
    }),
    Plot.ruleY([0])
  ]
})
```
Carpenter bees tend to have the highest body mass, followed by bumblebees, while honeybees are the smallest. This shows a clear difference in size across species.


<!--wingspan chart <o> -->

```js
Plot.plot({
  title: "Average Wing Span by Pollinator Species",
  x: {label: "Pollinator Species"},
  y: {label: "Average Wing Span (mm)"},
  marks: [
    Plot.barY(data, {
      x: "pollinator_species",
      y: "avg_wing_span_mm",
      fill: "pollinator_species"
    }),
    Plot.ruleY([0])
  ]
})
```
Wing span follows a similar pattern, with carpenter bees having the largest wings, bumblebees slightly smaller, and honeybees the smallest.


<!--temp chart-->

```js
Plot.plot({
  title: "Temperature vs Pollinator Visits",
  x: {label: "Temperature (°C)"},
  y: {label: "Visit Count"},
  marks: [
    Plot.dot(data, {
      x: "temperature",
      y: "visit_count"
      
    })
  ]
})
```
Pollinator visits increase as temperature rises, suggesting warmer conditions are more favorable for activity. However, there is still a baseline level of visits even at lower temperatures.

```js
Plot.plot({
  title: "Average Nectar Production by Flower Species",
  x: {label: "Flower Species"},
  y: {label: "Average Nectar Production (μL)"},
  marks: [
    Plot.barY(data, {
      x: "flower_species",
      y: "nectar_production",
      fill: "flower_species"
    }),
    Plot.ruleY([0])
  ]
})
```
Nectar production wise, Sunflower takes the lead. Coneflower and lavender compete for second, with the former taking the lead by about 10 units. 
