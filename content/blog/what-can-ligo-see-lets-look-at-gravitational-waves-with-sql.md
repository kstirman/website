+++
author = "Nathan Griffith"
categories = ["drill", "sql", "science", "gravity waves", "ligo", "physics", "astrophysics"]
type = "blog"
date = "2016-02-11"
title = "What can LIGO see? Let's look at gravitational waves with SQL"
+++

<script type="text/javascript"
 src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

It's difficult to overstate how thrilling today's news about gravity waves is. The scientific community has been waiting
a *long* time for this, and verification of the phenomenon has wide reaching implications in the fields of both
astrophysics and particle physics. Gravity is, after all, the biggest thorn in the side of modern theoretical particle
physics.

From a particle-centric standpoint gravity wave detection is identical to 'graviton' detection. This is important
because gravitons amount to the 'missing link' between the currently disconnected realms of the very big (dictated by
Einstein's general relativity) and the very small (governed by quantum mechanics). The observation of gravitational
waves may help to constrain the current multitude of competing quantum gravity theories, leading us closer what's
frequently called the Holy Grail of physics: a Theory of Everything.

Because the LIGO gravitational wave observatory is awesome, they've made some of the data relevant to their gravity wave
event public (go check out [this site](https://losc.ligo.org/events/GW150914/)). As you may have guessed, I'm going to
use [Apache Drill](https://drill.apache.org/docs/drill-in-10-minutes/) to say something about the data! In particular
I'll be investigating just how sensitive their equipment is, and what else they may be able to detect.

To do this analysis, I'll be looking at the data for the 'H1' LIGO detector shown in the leftmost plots of the top and
third rows of [Figure 1](https://losc.ligo.org/events/GW150914/). These files are called `fig1-observed-H.txt` and
`fig1-residual-H.txt`, and they contain the signal-with-background and background-only time series data. To use these
with Drill, open them up and remove the first line of the file, which is a comment. Then go edit the 'dfs' plugin JSON
(go to http://localhost:8047/storage/dfs after starting `drill-embdedded`) so that you have an entry in `"formats"` that
looks like:

```
    "txt": {
      "type": "text",
      "extensions": [
        "txt"
      ],
      "delimiter": " "
    },
```

This ensures that Drill knows that to do with a file that ends in '.txt', instructing it to treat spaces as the column
delimiter. With this done, let's write a SQL query to find the standard deviation of the background noise:

```
> SELECT STDDEV(CAST(columns[1] AS FLOAT)) FROM dfs.`/path/to/fig1-residual-H.txt`;
+----------------------+
|        EXPR$0        |
+----------------------+
| 0.16534411608717056  |
+----------------------+
```

So now let's ask: On average, how many standard deviations away from the noise are the 100 biggest signal data points?
First, we'll make a view (remember `USE dfs.tmp;`):

```
CREATE VIEW bigsignal AS
     SELECT ABS(CAST(columns[1] AS FLOAT)/0.165) signal
       FROM dfs.`/Users/ngriffith/Downloads/LIGO/fig1-observed-H.txt`
   ORDER BY signal DESC
      LIMIT 100;
```

and then we can take the average:

```
> SELECT AVG(signal) FROM bigsignal;
+--------------------+
|       EXPR$0       |
+--------------------+
| 5.884780800703798  |
+--------------------+
```

Not too far off from the event significance given in [the paper](https://dcc.ligo.org/P150914/public) of 5.3 sigma!
(Although their methodology is admittedly wildly different as well as much more rigorous and subtle.)

But what exactly *is* this signal? Well a LIGO detector works by precisely measuring two identical kilometer-scale
distances with lasers. The measured distances are are arranged in a cross-pattern, so that any passing gravitational
waves will cause these lengths to contract or expand relative to one another by different (*very tiny!*) amounts. The
'signal' numbers that we've been looking at are the differences between these two lengths. One source of gravity waves
(and the one that the people who run LIGO indicate in their announcement) is two massive bodies, such as black holes,
orbiting one another.

Now let's have a little more fun. The length difference from the data should be directly proportional to the amplitude
\\( A \\) of the gravity wave, which in this situation expresses the proportionality ([according to
Wikipedia](https://en.wikipedia.org/wiki/Gravitational_wave#Power_radiated_by_orbiting_bodies)) of:

$$ A \propto  \frac{m_1 m_2}{R} $$

where \\( m_1 \\) and \\( m_2 \\) are the masses of the orbiting bodies, and \\( R \\) is the distance of the observer
from the center of mass of the two-body system.

Decreasing the observed signal waveform of about 5.88 sigma by half would leave itself us comfortably near 3 sigma
territory, which is still a very strong indicator for significance (randomly achieving 3 sigma result is about a
one-in-ten-thousand event). Admittedly from a visual standpoint this wouldn't leave a very strong looking signal, but
statistical analysis in conjunction with confirmation from another type of observatory (such as one looking at radio or
gamma rays) may yield useful astrophysical data.

In the discovery paper the authors list two colliding black holes with similar masses (around 30 times the Sun) as a
likely source of the event that they observed. They also place the event at a distance of about 1.2 billion light-years
from Earth. If we can manage to notice gravity wave signals with half the strength, then LIGO would be able to detect
similar events twice as far away, or with 71% the constituent mass.

The gravitational wave astronomy revolution has just started, and I'm extremely excited to see where it leads us!
