---
name: football-image-collector
description: After a football analysis post is written, searches for relevant images from stats sites and official sources, downloads them, and saves them in a dated folder on the desktop. Trigger when the user says "find images", "get images for this post", "collect images", or automatically after any football post is drafted if the user has indicated they want images. Also trigger when the user says "save images" or "image folder".
---

# Football Image Collector

After a football analysis post has been drafted, your job is to:
1. Identify exactly what images are needed for that post
2. Search for them on appropriate sources
3. Download them to a named folder on the desktop
4. Report what was saved and what needs manual saving

---

## Step 1 ??Identify Image Needs

Read the post and extract:
- **Match**: which teams, which competition, which date
- **Key stats mentioned**: xG, possession %, specific player numbers, aerial duels, etc.
- **Key players mentioned**: any player whose stats or profile might be useful
- **Tactical concept**: if the post explains a concept (low block, high press, etc.)

Based on this, decide which of the following image types are needed:

| Image type | When needed | Best source |
|---|---|---|
| Match stats graphic | Every match-based post | SofaScore, WhoScored |
| xG chart / shot map | When xG is discussed | Understat, FBref |
| Player season stats | Player Story format | FBref, WhoScored |
| League table / standings | Context-setting posts | WhoScored |
| Match action photo | Every post | Official club Twitter/Instagram |
| Heatmap / touch map | Tactical posts | SofaScore |

---

## Step 2 ??Search for Images

For each image type needed, search using WebSearch with specific queries:

**Match stats (SofaScore):**
- Query: `site:sofascore.com [Team A] [Team B] [date or competition]`
- Or: `SofaScore [Team A] vs [Team B] [month year] stats`

**xG / shot map (Understat):**
- Query: `understat [Team A] [Team B] xG [season]`
- Navigate to: `understat.com/match/[id]` if match ID is known

**Player stats (FBref):**
- Query: `fbref [player name] [season] stats`

**Match action photo:**
- Query: `[Team A] vs [Team B] [date] official [club name] twitter`
- Check the club's official Twitter/X and Instagram for post-match images they shared publicly

**WhoScored match report:**
- Query: `whoscored [Team A] [Team B] [competition] [season]`

---

## Step 3 ??Download Images

For each image found, download it to the desktop folder using PowerShell.

**Create the folder first:**
```powershell
$date = Get-Date -Format "yyyy-MM-dd"
$topic = "[short topic slug from post title]"
$folder = "C:\Users\User\Desktop\$date - $topic"
New-Item -ItemType Directory -Path $folder -Force
```

**Download each image:**
```powershell
$url = "[image URL]"
$filename = "01_match_stats.png"  # use numbered prefix for ordering
Invoke-WebRequest -Uri $url -OutFile "$folder\$filename"
```

**Filename convention:**
- `01_match_stats.png` ??SofaScore/WhoScored match overview
- `02_xg_chart.png` ??xG or shot map
- `03_player_stats.png` ??player stat card
- `04_heatmap.png` ??heatmap or touch map
- `05_match_action.jpg` ??official match photo

**If a direct image URL cannot be found or the download is blocked:**
- Save a text file `image_links.txt` in the same folder with the URL and description
- The user can open the link and save manually

```powershell
$links = @"
[Image description]: [URL]
[Image description]: [URL]
"@
$links | Out-File "$folder\image_links.txt" -Encoding UTF8
```

---

## Step 4 ??Report to User

After completing, report:

```
?? Folder created: Desktop/[date] - [topic]

??Downloaded:
- 01_match_stats.png (SofaScore)
- 02_xg_chart.png (Understat)

?? Manual save needed (links saved in image_links.txt):
- Match action photo: [URL] ??official Newcastle Twitter post-match
- Player heatmap: [URL] ??SofaScore (login required)

? Tip: [Any note about image quality or alternatives]
```

---

## Source Reference

| Site | What it's good for | Notes |
|---|---|---|
| SofaScore | Match stats, heatmaps, ratings | Some images require login |
| WhoScored | Match overview, player ratings, charts | Good for season stats |
| FBref | Deep player/team stats, progressive passes, etc. | Screenshot-friendly tables |
| Understat | xG charts, shot maps, rolling xG | Free, no login |
| Opta / StatsBomb | Published infographics on their social feeds | Search their Twitter posts |
| Official club Twitter/X | Post-match action photos | Publicly shared, safe to use with credit |
| UEFA official media | Champions League / Europa match photos | Press-release quality |
| Premier League official | EPL match photos and graphics | Publicly shared |

---

## Important Notes

- Only download images that are publicly accessible (no login wall)
- For images behind login (SofaScore heatmaps, etc.) ??save the URL in `image_links.txt`
- Always note the source so the user can credit correctly when posting
- Do not download Getty / AFP / Reuters watermarked images
- Official club and league social media posts are the safest source for match action
