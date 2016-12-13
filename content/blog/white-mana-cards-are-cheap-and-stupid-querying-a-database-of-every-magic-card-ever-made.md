+++
date = "2015-12-03"
title = "White mana cards are cheap and stupid: Querying a database of every Magic card ever made"
type = "blog"
author = "Nathan Griffith"
categories = ["drill", "sql", "json", "magic", "games", "data"]

+++

With over 10,000 unique cards and a decade of success behind it, *Magic: The Gathering* is still as relevant as ever to 
fans of the collectible card game genre. But how well has *Magic* managed to stick to its core design philosophy over
the years? Today we'll examine this question by analyzing a JSON dump of the text found on every card ever produced.

For the uninitiated (and those who haven't a touched a card since their days on the middle school cafeteria circuit),
actions in a *Magic* game are enabled by a resource called "mana" that comes in five fundamental varieties: White, Blue,
Black, Red, and Green. Each of these colors has a different personality and play style. I'm not going to go into extreme
detail here (believe me, you can find that somewhere else on the web if you need it), but some of the characteristics
are:

+ White: Likes to fight using small creatures. Has a focus on life and healing.
+ Blue: Values control and intellectual capacity.
+ Black: Uses cards with themes of death and disease. Not afraid of hurting the player to the gain the advantage.
+ Red: Emphasis on creatures that pack a lot of offensive punch and spells that do damage.
+ Green: Employs large powerful creatures and nature/life forces.

<br>
<br>
<p style="text-align: center;">
<img style="max-width: 100%;" src="/img/mtg_mana.jpg">
</p>
<p style="text-align: center; font-style: italic;">The mana symbols of *Magic: The Gathering*.<br>Source: http://archive.wizards.com/Magic/magazine/article.aspx?x=mtg/daily/arcana/719</p>
<br>
<br>

We'll proceed by using Apache Drill to query a folder full of JSON documents (one for each *Magic* set), which is
available via [this site](http://mtgjson.com) as a zipped file. As an example, here's the JSON for a card called
"Abyssal Hunter" which is found in the "cards" list of the object/file corresponding to the "Mirage" set:

```
{
  "artist":"Steve Luke",
  "cmc":4,
  "colors":["Black"],
  "flavor":"Some smiles show cheer; some merely show teeth.",
  "id":"61beaf3de61dc097e0070b3f13b8de84da3c2761",
  "imageName":"abyssal hunter",
  "layout":"normal",
  "manaCost":"{3}{B}",
  "multiverseid":3272,"name":"Abyssal Hunter",
  "power":"1",
  "rarity":"Rare",
  "subtypes":["Human","Assassin"],
  "text":"{B}, {T}: Tap target creature. Abyssal Hunter deals damage equal to Abyssal Hunter's power to that creature.",
  "toughness":"1",
  "type":"Creature — Human Assassin",
  "types":["Creature"]
}
```

Before we begin our analysis, let's do a little prep work and set up a "view" in Drill called `card_data` that we can
query against. The following command makes a view that pulls data about the name, mana cost, text, mana color, and the
toughness/power of a card if it's a 'creature':

```sql
CREATE VIEW card_data AS select DISTINCT t.cards.name Name, t.cards.cmc CMC, t.cards.text Text, t.cards.colors[0] Color,
            t.cards.types[0] Type, t.cards.toughness Toughness, t.cards.`power` `Power`
       FROM (SELECT FLATTEN(t_raw.cards) cards
               FROM dfs.`/path/to/AllSetFiles` t_raw) t
       WHERE t.cards.colors[0] IS NOT NULL
         AND t.cards.colors[1] IS NULL
         AND t.cards.types[0] IS NOT NULL
         AND t.cards.types[1] IS NULL;
```

To simplify things, I'm using the last four lines to specify cards that have only one color requirement for mana (e.g.
'Black'), and are classified as belonging to only one type (e.g. 'Creature').

First, let's see if Black's supposed fascination with morbidity holds up by searching the card texts for instances of
the word 'die.'

```
> SELECT Color, COUNT(Name) `Num Cards` FROM card_data WHERE LOWER(Text) LIKE '%die%' GROUP BY Color;
+--------+------------+
| Color  | Num Cards  |
+--------+------------+
| White  | 45         |
| Black  | 74         |
| Red    | 48         |
| Green  | 49         |
| Blue   | 13         |
+--------+------------+
```

Ok, so far so good. Now what about the word 'life'? Seems like this one should show up a lot in White and maybe Green
cards.

```
> SELECT Color, COUNT(Name) `Num Cards` FROM card_data WHERE LOWER(Text) LIKE '%life%' GROUP BY Color;
+--------+------------+
| Color  | Num Cards  |
+--------+------------+
| White  | 189        |
| Black  | 243        |
| Green  | 80         |
| Blue   | 9          |
| Red    | 20         |
+--------+------------+
```

Wow&mdash;Black is once again in the lead! I think that makes sense though; a lot of Black cards probably
have to do with manipulating the player's life points. But it's still a bit odd that Black comes off as the most
explicitly 'life'-obsessed.

Next let's see if the creature cards from each color of mana fall in line with their characterizations from the list
above. We can ask for the average 'power' and 'toughness' of creatures with non-contextual stats (i.e., ones that don't
have an asterisk in these values) with this query:

```
> SELECT Color, AVG(CAST(`Power` AS FLOAT)) `Avg Power`, AVG(CAST(Toughness AS FLOAT)) `Avg Toughness` FROM card_data t WHERE Type = 'Creature' AND `Power` NOT LIKE '%*%' AND `Toughness` NOT LIKE '%*%' GROUP BY Color;
+--------+---------------------+---------------------+
| Color  |      Avg Power      |    Avg Toughness    |
+--------+---------------------+---------------------+
| White  | 2.026235741444867   | 2.555513307984791   |
| Blue   | 2.2026678141135974  | 2.6351118760757317  |
| Black  | 2.5331278890600926  | 2.450308166409861   |
| Red    | 2.6187548039969255  | 2.4584934665641813  |
| Green  | 2.6882978723404256  | 2.8177304964539007  |
+--------+---------------------+---------------------+
```

Yup, Green definitely has the most 'power'-ful offensive creatures on average, with Red falling just behind it. But
what's interesting about the results of this query is how close the average Black creature falls to the average Red
creature&mdash;especially when you take into account their defensive 'toughness' stat. Black's creature characterization
isn't emphasized in descriptions of the mana colors, but this table indicates that Black creatures are "basically Red"
(at least when it comes to their fundamental stats).

Finally, let's check out the average converted mana cost and average text length of mono-colored, mono-typed cards.
Converted mana cost (CMC) is the total amount of mana resources (color-specific plus any-colored) that a card requires
when it's put in play, and the text of a card tells you how to use it in a game.

```
> SELECT Color, AVG(CMC) `Avg CMC`, AVG(LENGTH(Text)) `Avg Text Length` FROM card_data t GROUP BY Color;
+--------+---------------------+---------------------+
| Color  |       Avg CMC       |   Avg Text Length   |
+--------+---------------------+---------------------+
| White  | 3.154371002132196   | 132.53899480069325  |
| Blue   | 3.3505647263249347  | 143.49057430951336  |
| Black  | 3.4389721627408996  | 141.2681660899654   |
| Red    | 3.3674957118353346  | 138.5313862249346   |
| Green  | 3.3970149253731345  | 139.6011309264898   |
+--------+---------------------+---------------------+
```

Remember how I said White liked to play small creatures? Well it looks like all those small-cost cards are pretty
evident in White's average mana cost. So: check. White is behaving as expected.

It's worth noting, though, that Black clearly has the highest average mana cost of the five colors. I feel like that's
pretty notable! Maybe that fact should be included in more intro summaries.

We haven't talked about Blue much yet, but remember: It's the mana color that most likes to show off its knack for
strategy and fancy book learnin'. If we can take the average length of a card's instruction text to be a good indicator
of complexity, then this table shows that Blue definitely lives up to its reputation.

So if Blue is 'smart,' then who is 'stupid'? Well, I guess that would be White. However, I can see why no one describes
it that way: "Play White: It's cheap and stupid!" probably isn't the best way create new fans of White *Magic* cards.

*Mana symbols and card text are copyright Wizards of the Coast LLC.*
