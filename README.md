# Does Height Matter in Professional Tennis?

**Perry Gregg**

[View Full HTML Report](https://perrygregg4.github.io/Tennis-ATP-Scraping/atpscraping.html)


## Project Overview

I got curious about this after watching a match where the announcers kept bringing up how much Reilly Opelka's height helps his serve. It made me wonder whether being tall actually translates into a better ranking, or whether it is more of a talking point than a real advantage. Tennis is not basketball. You need speed, consistency, a solid return game, and a lot of other things that height probably does not help with. So I decided to just look at the numbers.

I scraped the current ATP Top 100 from the tour's own website, pulled heights from ESPN, ran a regression to see if there was any real relationship, and then checked whether media coverage on Google News was more positive toward taller players. That last part was honestly more of a curiosity than a hypothesis I was confident about.



## Scraping ATP Rankings

I pulled the top 100 singles players straight from the ATP Tour rankings page. The table structure was pretty clean so BeautifulSoup handled it without much trouble. The one annoying part was getting country codes. They are stored inside an SVG flag element as part of a long href string, so I had to strip the prefix path to get just the two-letter code. Everything else, rank, name, points, and player URL, came through without any issues.

```{python}
import requests
from bs4 import BeautifulSoup
import pandas as pd

link = "https://www.atptour.com/en/rankings/singles"
link_request = requests.get(link, headers={
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
})
soup = BeautifulSoup(link_request.content, 'html.parser')
rows = soup.select('table tbody tr')

df_out = []
for row, entry in enumerate(rows):
    rank = entry.select_one('td.rank')
    name = entry.select_one('li.name a span.lastName')
    player_link = entry.select_one('li.name a')
    flag = entry.select_one('li.avatar use')
    points = entry.select_one('td.points a')

    country_code = None
    if flag:
        href = flag.get('href', '')
        country_code = href.replace('/assets/atptour/assets/flags.svg#flag-', '').upper()

    out = pd.DataFrame({
        'rank': [rank.text.strip() if rank else None],
        'player': [name.text.strip() if name else None],
        'country': [country_code],
        'points': [points.text.strip() if points else None],
        'link': ['https://www.atptour.com' + player_link.get('href') if player_link else None]
    })
    df_out.append(out)

df_out = pd.concat(df_out, ignore_index=True)
df_out.dropna(subset=['player'], inplace=True)
df_out.reset_index(drop=True, inplace=True)
df_out.head()
```

This gave me 100 rows, Alcaraz at the top with 13,550 points and Luca Van Assche at the bottom with 596.

---

## Collecting Player Heights from ESPN

The ATP site does not have height data in a format that was easy to scrape, so I went to ESPN instead. I grabbed the player profile URLs from the ESPN tennis rankings page and then looped through each one to find the height field in the metadata section. ESPN lists heights in feet and inches like "6-2", so I split the string and converted to centimeters. I added a short sleep between requests to avoid getting rate limited.

```{python}
import time

headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'}
espn_rankings_url = 'https://www.espn.com/tennis/rankings'
espn_r = requests.get(espn_rankings_url, headers=headers)
espn_soup = BeautifulSoup(espn_r.content, 'html.parser')
espn_links = espn_soup.select('a[href*="/tennis/player/_/id/"]')

seen_urls = []
seen_set = {}
for a in espn_links:
    href = a.get('href', '')
    if href not in seen_set:
        seen_set[href] = True
        full_url = href if href.startswith('http') else 'https://www.espn.com' + href
        seen_urls.append(full_url)
    if len(seen_urls) == 100:
        break

espn_players = []
for i, url in enumerate(seen_urls):
    player_r = requests.get(url, headers=headers)
    player_soup = BeautifulSoup(player_r.content, 'html.parser')

    name_tag = player_soup.find('h1')
    player_name = name_tag.get_text(strip=True) if name_tag else None

    height_raw = None
    height_cm = None
    for item in player_soup.select('ul.player-metadata li'):
        label = item.find('span')
        if label and label.get_text(strip=True) == 'Height':
            full_text = item.get_text(strip=True)
            height_raw = full_text.replace(label.get_text(strip=True), '').strip()
            break

    if height_raw:
        parts = height_raw.split('-')
        if len(parts) == 2:
            feet, inches = int(parts[0]), int(parts[1])
            height_cm = round((feet * 12 + inches) * 2.54, 1)

    espn_players.append({
        'player': player_name,
        'height_raw': height_raw,
        'height_cm': height_cm,
        'espn_url': url
    })
    time.sleep(0.3)

df_heights = pd.DataFrame(espn_players)
df_heights.head()
```

Every player had a height listed, which I was not expecting since I thought some of the lower-ranked players might not have full profiles. The range was wider than I thought too. Reilly Opelka at 210 cm and Sebastian Baez at 170 cm are 40 cm apart, and both are in the top 100. The mean came out to 187.4 cm, so about 6 feet 2 inches. Most players are somewhere in the 182 to 195 range.

---

## Does Height Actually Predict Ranking?

I ran a linear regression using scipy with height as the x variable and ATP ranking as y. Since rank 1 is the best, a negative slope would mean taller players tend to rank higher.

```{python}
import numpy as np
from scipy import stats
import matplotlib
import matplotlib.pyplot as plt
matplotlib.rcParams['figure.dpi'] = 100

df_heights['rank'] = range(1, len(df_heights) + 1)
df_reg = df_heights[df_heights['height_cm'].notna()].copy()
df_reg['rank'] = df_reg['rank'].astype(int)

slope, intercept, r_value, p_value, std_err = stats.linregress(
    df_reg['height_cm'], df_reg['rank']
)

print(f"Slope:     {slope:.4f} rank positions per cm")
print(f"R-squared: {r_value**2:.4f}")
print(f"p-value:   {p_value:.4f}")
```

 Metric :  Value 
 Slope:  -1.33 rank positions per cm 
 R squared:  0.095 
 p value:  0.0018 

The p value is 0.0018, so the relationship is significant. The slope says that for each extra centimeter of height you gain about 1.3 ranking spots on average. I was honestly a little surprised the relationship showed up at all given how scattered the scatter plot looked.

But then the R squared kind of puts it in perspective. 0.095 means height is only accounting for about 9.5 percent of what determines someone's ranking. So yes, there is something there, but it is a pretty small piece of the picture. My best guess for why height helps at all is the serve. Taller players get more leverage and a steeper angle into the box, which translates to easier holds. But that does not explain 90 percent of what separates rank 1 from rank 100, which is pretty much everything else about how you play tennis.

```{python}
fig, ax = plt.subplots(figsize=(8, 5))
ax.scatter(df_reg['height_cm'], df_reg['rank'], alpha=0.6, edgecolors='k', linewidths=0.5)
x_line = np.linspace(df_reg['height_cm'].min(), df_reg['height_cm'].max(), 200)
y_line = slope * x_line + intercept
ax.plot(x_line, y_line, color='red', linewidth=1.5,
        label=f'y = {slope:.2f}x + {intercept:.1f}\nR² = {r_value**2:.3f}, p = {p_value:.4f}')
ax.set_xlabel('Height (cm)')
ax.set_ylabel('ATP Ranking')
ax.set_title('ATP Ranking vs. Player Height')
ax.legend()
plt.tight_layout()
plt.show()
plt.close(fig)
```



## Sentiment Analysis Using Google News Headlines

I wanted to see whether media coverage was warmer toward taller players. My first attempt used ESPN player pages to grab headlines, but it turned out 48 of the 100 players did not have enough articles there to score, which made the tall versus short comparison pretty unreliable since the players with more ESPN coverage tended to be the bigger names anyway. I switched to the Google News RSS feed, which you can query by just passing a player's name and "tennis" as search terms. That returned around 100 headlines per player across all 100 players, so the comparison was actually apples to apples.

```{python}
from textblob import TextBlob
from bs4 import XMLParsedAsHTMLWarning
import warnings
warnings.filterwarnings('ignore', category=XMLParsedAsHTMLWarning)

headline_data = []
for i in range(len(df_heights)):
    player_name = df_heights.loc[i, 'player']
    if not player_name:
        headline_data.append({'player': player_name, 'num_headlines': 0, 'avg_sentiment': None})
        continue

    query = str(player_name).replace(' ', '+') + '+tennis'
    url = f'https://news.google.com/rss/search?q={query}&hl=en-US&gl=US&ceid=US:en'
    r = requests.get(url, headers=headers, timeout=10)
    soup = BeautifulSoup(r.content, 'html.parser')

    items = soup.find_all('item')
    titles = [item.find('title').get_text(strip=True) for item in items if item.find('title')]
    sentiments = [TextBlob(t).sentiment.polarity for t in titles]
    avg_sentiment = sum(sentiments) / len(sentiments) if sentiments else None

    headline_data.append({
        'player': player_name,
        'num_headlines': len(titles),
        'avg_sentiment': avg_sentiment
    })
    time.sleep(0.5)

df_sentiment = pd.DataFrame(headline_data)
df_merged = pd.merge(df_heights, df_sentiment, on='player')
mean_height = df_merged['height_cm'].mean()
df_merged['height_group'] = df_merged['height_cm'].apply(
    lambda h: f'Tall (at or above {round(mean_height, 1)} cm)'
    if h >= mean_height else f'Short (below {round(mean_height, 1)} cm)'
)
df_merged[['player', 'height_cm', 'height_group', 'num_headlines', 'avg_sentiment']].head()
```

Group:  Mean Sentiment Score 
 Tall players: (at or above 187.4 cm)  0.0866 
 Short players: (below 187.4 cm)  0.0767 
 Difference: 0.0100 

Both groups are basically sitting at zero. The scale goes from negative one to positive one and both means are under 0.09, so neither group is getting coverage that is noticeably positive or negative. The 0.01 difference between them is about as close to nothing as you can get. I am not totally sure why individual players varied as much as they did within each group, though my guess is it comes down to whether they recently won something or were in the news for a controversy rather than anything to do with their height. The main takeaway is that height does not seem to shape how journalists write about these players.

```{python}
groups = df_merged.groupby('height_group')['avg_sentiment'].apply(list)
labels = list(groups.index)
data = [groups[l] for l in labels]

fig, ax = plt.subplots(figsize=(7, 5))
bp = ax.boxplot(data, labels=labels, patch_artist=True,
                boxprops=dict(facecolor='lightblue', color='navy'),
                medianprops=dict(color='red', linewidth=2))
ax.set_ylabel('Average Sentiment Score (TextBlob)')
ax.set_title('News Sentiment by Player Height Group')
ax.axhline(0, color='gray', linestyle='--', linewidth=0.8)
plt.tight_layout()
plt.show()
plt.close(fig)
```

---

## Results and Conclusion

The regression found a real relationship between height and ranking, and the p value is low enough that it is hard to dismiss. Taller players do tend to rank higher. The serve advantage is probably the main reason. That said, the R squared of 0.095 makes it clear that height is not much of a deciding factor when you compare it to everything else that goes into being a top 100 player.

The sentiment analysis did not find much of anything. With complete coverage of all 100 players and around 100 headlines each, the difference in how the media covers tall versus short players was essentially zero. If you are writing about a tennis player, you are probably writing about their results, not their height.




