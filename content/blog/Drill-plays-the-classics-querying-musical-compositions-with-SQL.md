+++
author = "Nathan Griffith"
categories = ["drill", "music", "sql", "midi"]
date = "2016-01-21"
title = "Drill plays the classics: Querying musical compositions with SQL"
type = "blog"
+++

Today in 'Wow, I never expected to use SQL for *that!*,' I'm going to show how you can use Drill (along with a simple
command line utility) to analyze musical compositions for sound and style characteristics.

The music we'll be analyzing is a selection of piano pieces written by various classical composers. In particular, we'll
be looking at representations of this music encoded into MIDI files created by one Bernd Krueger, who hosts them on his
site: www.piano-midi.de. A linchpin of this approach will be the super cool command line utility `midicsv` ([available
here](http://www.fourmilab.ch/webtools/midicsv/)), which translates the events of a MIDI file into text in the form of a
CSV file.

After a little prepwork I found myself with a directory called `music_csv` which held the CSV-translated MIDI files in
subdirectories named after each of the composers I had downloaded. As we can see on the midicsv web site, the third
column of each CSV listing contains the type of event. Two of these that might be of particular interest are 'Note_on_c'
and 'Tempo', which respectively begin playing a note and set the current tempo of the piece. For 'Note_on_c' the value
we'll pay attention to (the note being played) is in column five, while for 'Tempo' we'll (unsurprisingly) be looking at
tempo values, which for this event type show up in column four.

In MIDI notes are identified by a number, with '60' being Middle C, and 59 and 61 being the notes just below and just
above that location. So first let's examine the 'average note' for each composer's selection. This might be a good
choice if we're looking for music that sounds somewhat dramatic, due to the presence of more notes from the bottom half
of the keyboard. A Drill query to my `music_csv` directory that gives me each composer's average note looks like:

```
  SELECT dir0 composer, AVG(CAST(TRIM(columns[4]) AS INT)) `average note`
    FROM dfs.`/path/to/music_csv`
   WHERE columns[2] LIKE ' Note_on_c'
GROUP BY dir0
ORDER BY `average note`;
```

Which yields the following:

```
+--------------+---------------------+
|   composer   |    average note     |
+--------------+---------------------+
| borodin      | 61.21679561573178   |
| mendelssohn  | 61.601812135524604  |
| schumann     | 62.13615485564304   |
| bach         | 62.41817789291883   |
| chopin       | 62.59844650639674   |
| beethoven    | 62.94818578301335   |
| granados     | 62.998422159887795  |
| schubert     | 63.03703764725266   |
| mussorgsky   | 63.205213945135924  |
| debussy      | 63.73597678916828   |
| brahms       | 63.85321901437839   |
| tchaikovsky  | 63.93177966101695   |
| grieg        | 65.22842035060975   |
| burgmueller  | 65.4425336466567    |
| balakirew    | 66.0802039293708    |
| albeniz      | 66.68735803242735   |
| mozart       | 67.12492121887229   |
| liszt        | 67.51094939468125   |
| haydn        | 67.87497578301583   |
+--------------+---------------------+
```

So for moodier music, it looks like Borodin is your best bet.

Next we'll try a similar query for the average tempo of a composer's selections. This could be useful if you're looking
for either tranquil or spritely music:

```
  SELECT dir0 composer, AVG(CAST(TRIM(columns[3]) AS INT)) `average tempo`
    FROM dfs.`/Users/ngriffith/Downloads/music_csv`
   WHERE columns[2] LIKE ' Tempo'
GROUP BY dir0
ORDER BY `average tempo`;
```

With:

```
+--------------+---------------------+
|   composer   |    average tempo    |
+--------------+---------------------+
| grieg        | 463950.99792665546  |
| balakirew    | 516662.79188934295  |
| chopin       | 549635.4575186928   |
| schubert     | 555130.6943059019   |
| granados     | 559992.7081807082   |
| beethoven    | 562109.4496243594   |
| mussorgsky   | 572826.2225694832   |
| tchaikovsky  | 582765.8715467677   |
| burgmueller  | 585393.9650706437   |
| borodin      | 587289.7334545455   |
| albeniz      | 588197.7116116117   |
| liszt        | 591821.5634490239   |
| debussy      | 599595.9116922494   |
| haydn        | 606727.8991650156   |
| schumann     | 620904.3406527168   |
| mozart       | 634484.8481915854   |
| bach         | 665263.0298245614   |
| brahms       | 688238.5542258788   |
| mendelssohn  | 695130.138571672    |
+--------------+---------------------+
```

So start your search for slower music with Grieg, and go to Mendelssohn if you'd prefer something faster. (If you're
curious, the unit used for tempo here is the length of a quarter note in microseconds).

Note that both of these examples rely on Drill's ability to query entire directories (and subdirectories!) of files at once.
And I've also used the `dir0` variable to my advantage in order to provide the name of the composer from my folder
hierarchy.

Querying classical music compositions is definitely an unexpected and wonderful way to use Drill!
