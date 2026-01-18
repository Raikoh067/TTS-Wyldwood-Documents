# Wyldwood Playtest Data Collection
## Google Sheets Setup Guide

---

## What This System Does

This playtest data collection system automatically captures detailed game statistics from your Wyldwood sessions and sends them to a Google Sheets database. This allows you to:

- **Track balance over time** — See which factions win more often and by how much
- **Identify problematic cards** — Find cards that appear in winning decks too frequently (or never)
- **Analyze champion performance** — Track which champions survive, die, or accumulate woodwisps
- **Collect player feedback** — Gather qualitative feedback alongside quantitative data
- **Generate reports and charts** — Use Google Sheets' built-in tools to visualize trends

---

## Data Points Collected

Each time a playtester clicks "Submit Playtest Data," the system captures:

### Per Match (Games Sheet)
| Data Point | Description |
|------------|-------------|
| **Game ID** | Unique identifier linking all data from one session |
| **Timestamp** | Date and time the game ended |
| **Round Ended** | Which round the game concluded on |
| **Winner** | Which faction won (auto-calculated from scores) |
| **Feedback** | Card and general feedback from playtesters |

### Per Faction (FactionResults Sheet)
This is the **primary sheet for faction analysis** — each game creates **one row per faction** so you can easily filter by faction name!

| Data Point | Description |
|------------|-------------|
| **Game ID** | Links back to the match |
| **Faction** | The faction name (e.g., "Leafsong Nomads") |
| **Player Name** | The Steam name of the player controlling this faction |
| **Score** | Final victory points for this faction |
| **Deck Size** | Total cards in this faction's deck |
| **Is Winner** | TRUE/FALSE — did this faction win? |
| **Opponent** | Which faction they played against |
| **Opponent Score** | The opposing faction's final score |

### Per Card (DeckCards Sheet)
Tracks which cards players choose for their decks. The **faction** column lets you filter by faction to answer questions like *"What cards do Leafsong players pick most often?"*

| Data Point | Description |
|------------|-------------|
| **Game ID** | Links back to the match |
| **Card Name** | Name of each card in the deck |
| **Faction** | Which faction's deck this card belongs to (for filtering) |

**Example analysis:**
- Filter by `faction = "Leafsong Nomads"` → see all cards Leafsong players have used
- Create a pivot table (cardName × count) filtered by faction → most popular cards for that faction
- Join with FactionResults using gameId to see cards in *winning* decks

### Per Champion
| Data Point | Description |
|------------|-------------|
| **Champion Name** | e.g., Loresinger, UrscarKing |
| **Faction** | Which faction this champion belongs to |
| **Final Health** | Health remaining at game end |
| **Max Health** | Champion's starting health |
| **Woodwisps** | Number of woodwisps on this champion |
| **Is Dead** | TRUE/FALSE — was this champion eliminated? |

### Current Champion Faction Reference

| Leafsong Nomads | Boulderbreaker Clans |
|-----------------|---------------------|
| Loresinger | UrscarKing |
| HighSpiritseer | FellcastHunter |
| BonebladeAlpha | MakaraElder |
| Wyldspeaker | OutcastKingsguard |
| WyrdbowSentinel | Painseeker |

*More factions coming soon!*

### Feedback (Optional)
| Data Point | Description |
|------------|-------------|
| **Card Feedback** | Cards that felt too strong/weak |
| **General Feedback** | Any other comments from playtesters |
 
---

## Setup Instructions

### Step 1: Create Your Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Click the **+ Blank** button to create a new spreadsheet
3. Name it **"Wyldwood Playtest Data"** (click "Untitled spreadsheet" at top-left)

### Step 2: Create the Four Data Sheets

You need four sheets (tabs) to store different types of data.

1. Look at the bottom of the screen — you'll see a tab called "Sheet1"
2. **Right-click** the "Sheet1" tab and select **Rename** → type **Games**
3. Click the **+** button next to the tabs to add a new sheet → name it **FactionResults**
4. Click the **+** button again → name it **DeckCards**
5. Click the **+** button again → name it **Champions**

You should now have four tabs: `Games`, `FactionResults`, `DeckCards`, `Champions`

### Step 3: Add Column Headers

Click on each sheet tab and add these headers in **Row 1**:

#### Games Sheet (Match Metadata)
Type these in cells A1 through F1:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| gameId | timestamp | roundEnded | winner | cardFeedback | generalFeedback |

#### FactionResults Sheet (Faction Stats — Primary Analysis Sheet!)
Type these in cells A1 through I1:

| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| gameId | timestamp | faction | playerName | score | deckSize | isWinner | opponent | opponentScore |

> **Why this design?** Each game creates TWO rows here (one per faction). This makes it trivial to filter by faction name and see all Leafsong Nomads stats, for example. New factions automatically work — they just add more rows, not more columns!

#### DeckCards Sheet
Type these in cells A1 through D1:

| A | B | C | D |
|---|---|---|---|
| gameId | timestamp | faction | cardName |

#### Champions Sheet
Type these in cells A1 through H1:

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| gameId | timestamp | faction | championName | finalHealth | maxHealth | woodwisps | isDead |

### Step 4: Open Google Apps Script

1. In your Google Sheet, click **Extensions** in the menu bar
2. Select **Apps Script**
3. A new tab will open with a code editor
4. **Delete** everything in the editor (select all, delete)

### Step 5: Paste the Webhook Script

Copy the entire script below and paste it into the Apps Script editor:

```javascript
/**
 * Wyldwood Playtest Data Receiver
 * Receives game data from Tabletop Simulator and writes to Google Sheets
 */

const CONFIG = {
  GAMES_SHEET: 'Games',
  FACTION_RESULTS_SHEET: 'FactionResults',
  DECK_CARDS_SHEET: 'DeckCards',
  CHAMPIONS_SHEET: 'Champions'
};

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    console.log('Received playtest data for game: ' + data.gameId);
    
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    
    writeGameData(ss, data.game);
    writeFactionResultsData(ss, data.factionResults);
    writeDeckCardsData(ss, data.deckCards);
    writeChampionsData(ss, data.champions);
    
    return ContentService
      .createTextOutput(JSON.stringify({
        status: 'success',
        message: 'Data recorded successfully',
        gameId: data.gameId
      }))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch (error) {
    console.error('Error: ' + error.toString());
    return ContentService
      .createTextOutput(JSON.stringify({
        status: 'error',
        message: error.toString()
      }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  return ContentService
    .createTextOutput(JSON.stringify({
      status: 'ok',
      message: 'Wyldwood Playtest Data Receiver is running'
    }))
    .setMimeType(ContentService.MimeType.JSON);
}

function writeGameData(ss, gameData) {
  const sheet = ss.getSheetByName(CONFIG.GAMES_SHEET);
  if (!sheet) throw new Error('Games sheet not found');
  
  sheet.appendRow([
    gameData.gameId,
    gameData.timestamp,
    gameData.roundEnded,
    gameData.winner || 'Tie',
    gameData.cardFeedback || '',
    gameData.generalFeedback || ''
  ]);
}

function writeFactionResultsData(ss, factionResults) {
  const sheet = ss.getSheetByName(CONFIG.FACTION_RESULTS_SHEET);
  if (!sheet) throw new Error('FactionResults sheet not found');
  
  const rows = factionResults.map(fr => [
    fr.gameId,
    fr.timestamp,
    fr.faction,
    fr.playerName,
    fr.score,
    fr.deckSize,
    fr.isWinner,
    fr.opponent,
    fr.opponentScore
  ]);
  
  if (rows.length > 0) {
    const startRow = sheet.getLastRow() + 1;
    sheet.getRange(startRow, 1, rows.length, rows[0].length).setValues(rows);
  }
}

function writeDeckCardsData(ss, deckCards) {
  const sheet = ss.getSheetByName(CONFIG.DECK_CARDS_SHEET);
  if (!sheet) throw new Error('DeckCards sheet not found');
  
  const rows = deckCards.map(card => [
    card.gameId,
    card.timestamp,
    card.faction,
    card.cardName
  ]);
  
  if (rows.length > 0) {
    const startRow = sheet.getLastRow() + 1;
    sheet.getRange(startRow, 1, rows.length, rows[0].length).setValues(rows);
  }
}

function writeChampionsData(ss, champions) {
  const sheet = ss.getSheetByName(CONFIG.CHAMPIONS_SHEET);
  if (!sheet) throw new Error('Champions sheet not found');
  
  const rows = champions.map(champ => [
    champ.gameId,
    champ.timestamp,
    champ.faction,
    champ.championName,
    champ.finalHealth,
    champ.maxHealth,
    champ.woodwisps,
    champ.isDead
  ]);
  
  if (rows.length > 0) {
    const startRow = sheet.getLastRow() + 1;
    sheet.getRange(startRow, 1, rows.length, rows[0].length).setValues(rows);
  }
}

function testScript() {
  const gameId = 'TEST_' + new Date().getTime();
  const timestamp = new Date().toISOString();
  
  const testData = {
    gameId: gameId,
    game: {
      gameId: gameId,
      timestamp: timestamp,
      roundEnded: 5,
      winner: 'Leafsong Nomads',
      cardFeedback: 'Test feedback',
      generalFeedback: 'Test general feedback'
    },
    factionResults: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', playerName: 'TestPlayer1', score: 8, deckSize: 30, isWinner: true, opponent: 'Boulderbreaker Clans', opponentScore: 6 },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', playerName: 'TestPlayer2', score: 6, deckSize: 30, isWinner: false, opponent: 'Leafsong Nomads', opponentScore: 8 }
    ],
    deckCards: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', cardName: 'Savage Rend' },
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', cardName: 'Savage Rend' },
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', cardName: 'Savage Rend' },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', cardName: 'Boulder Smash' }
    ],
    champions: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', championName: 'Loresinger', finalHealth: 5, maxHealth: 10, woodwisps: 2, isDead: false },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', championName: 'UrscarKing', finalHealth: 0, maxHealth: 21, woodwisps: 0, isDead: true }
    ]
  };
  
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  writeGameData(ss, testData.game);
  writeFactionResultsData(ss, testData.factionResults);
  writeDeckCardsData(ss, testData.deckCards);
  writeChampionsData(ss, testData.champions);
  
  console.log('Test data written successfully!');
}
```

### Step 6: Save the Script

1. Click the **floppy disk icon** (or press **Ctrl+S** / **Cmd+S**)
2. If prompted, name the project **"Wyldwood Data Receiver"**

### Step 7: Deploy as a Web App

1. Click the blue **Deploy** button (top-right)
2. Select **New deployment**
3. Click the **gear icon** next to "Select type"
4. Choose **Web app**
5. Fill in the settings:
   - **Description:** `Wyldwood Playtest Receiver v1`
   - **Execute as:** `Me (your email)`
   - **Who has access:** `Anyone`
6. Click **Deploy**
7. Click **Authorize access** when prompted
8. Sign in with your Google account
9. If you see "Google hasn't verified this app," click **Advanced** → **Go to Wyldwood Data Receiver (unsafe)**
10. Click **Allow**

### Step 8: Copy Your Web App URL

After deployment, you'll see a URL that looks like:
```
https://script.google.com/macros/s/AKfycby...very-long-string.../exec
```

**Copy this entire URL** — you'll need it for the next step.

### Step 9: Update the TTS Script

1. In Tabletop Simulator, find the **GameTracker** object
2. Right-click it and select **Scripting** → **Lua Script**
3. Near the top, find this line:
   ```lua
   WEBHOOK_URL = "https://script.google.com/macros/s/YOUR_SCRIPT_ID_HERE/exec",
   ```
4. Replace `YOUR_SCRIPT_ID_HERE` with your actual Web App URL
5. Save the script (**Save & Play** or **Ctrl+S**)

---

## Testing Your Setup

### Test 1: Script Test (Apps Script)

1. Go back to your Apps Script tab
2. In the function dropdown (next to the play button), select **testScript**
3. Click the **Run** button (▶)
4. Go to your Google Sheet and check all four tabs — you should see test data

### Test 2: Full Test (Tabletop Simulator)

1. Load Wyldwood in TTS
2. Play a quick game or set up a test scenario
3. Click the **Submit Playtest Data** button on the GameTracker
4. Fill in some feedback and click **Submit**
5. Check your Google Sheet — new data should appear in all four tabs

---

## Analyzing Your Data

### Quick Stats (Games Sheet)

**Total Matches Played:**
```
=COUNTA(A2:A)
```

**Average Game Length (rounds):**
```
=AVERAGE(C2:C)
```

### Faction Stats (FactionResults Sheet — The Main Analysis Sheet!)

This is where the magic happens. Each game creates two rows here (one per faction), making it trivial to analyze any faction:

**Games Played by Faction:**
```
=COUNTIF(C:C,"Leafsong Nomads")
=COUNTIF(C:C,"Boulderbreaker Clans")
```

**Win Rate for Any Faction:**
```
=COUNTIFS(C:C,"Leafsong Nomads",G:G,TRUE)/COUNTIF(C:C,"Leafsong Nomads")
```

**Average Score for a Faction:**
```
=AVERAGEIF(C:C,"Leafsong Nomads",E:E)
```

**Average Score Differential (Win Margin):**
```
=AVERAGEIFS(E:E,C:C,"Leafsong Nomads",G:G,TRUE) - AVERAGEIFS(I:I,C:C,"Leafsong Nomads",G:G,TRUE)
```

**Filter All Games for One Faction:**
Use Filter: `Data → Create a filter` then filter column C (faction) by name

**All Matches Where a Specific Faction Won (from Games sheet):**
Use Filter: `Data → Create a filter` then filter column D (winner) by faction name

> **Pro tip:** For detailed faction analysis, use the **FactionResults** sheet instead — it's designed for faction-centric queries!

### Card Analysis (DeckCards Sheet)

**Most Popular Cards (Create a Pivot Table):**
1. Select all data (Ctrl+A)
2. Go to **Insert** → **Pivot table**
3. Set **Rows** to `cardName`
4. Set **Values** to `cardName` (summarize by COUNTA)
5. Sort by count descending

**Cards Used by Faction:**
```
=COUNTIFS(C:C,"Leafsong Nomads",D:D,"Card Name Here")
```

**Most Popular Cards for Any Faction (Pivot Table):**
1. Create pivot table
2. Add filter on `faction`
3. Rows: cardName, Values: COUNTA
4. Filter by whichever faction you want to analyze

### Champion Analysis (Champions Sheet)

**Champion Survival Rate:**
```
=1 - (COUNTIF(H2:H,TRUE)/COUNTA(H2:H))
```

**Average Health Remaining by Champion:**
```
=AVERAGEIF(D:D,"Loresinger",E:E)
```

**Total Woodwisps by Faction:**
```
=SUMIF(C:C,"Leafsong Nomads",G:G)
```

---

## Creating Charts

### Card Popularity Bar Chart
1. Create a Pivot Table of DeckCards (cardName → count)
2. Select the results
3. **Insert** → **Chart** → **Bar chart**

### Faction Win Rate Pie Chart
1. In a new area, calculate wins per faction using COUNTIF on the winner column
2. Select the data
3. **Insert** → **Chart** → **Pie chart**

### Champion Death Rate by Faction
1. Create a Pivot Table: Rows = faction, Values = COUNT of isDead where TRUE
2. Insert a bar chart

---

## Troubleshooting

### "Error submitting data" appears in TTS
- Double-check your Web App URL is correct (no extra spaces)
- Make sure the script is deployed with **"Anyone"** access
- Check Apps Script logs: **View** → **Executions**

### Data not appearing in sheets
- Verify sheet names are exactly: `Games`, `FactionResults`, `DeckCards`, `Champions`
- Check that column headers match the expected names
- Run `testScript()` in Apps Script to verify it works

### Permission errors
- Re-deploy the web app (Deploy → Manage deployments → Edit → New version)
- Make sure you authorized the script when prompted

---

## Updating the Script Later

If you need to make changes:

1. Edit the code in Apps Script
2. Click **Deploy** → **Manage deployments**
3. Click the **pencil icon** to edit
4. Change **Version** to **"New version"**
5. Click **Deploy**

The URL stays the same — no changes needed in TTS!

---

## Questions?

If you run into issues, check:
1. Apps Script execution logs (**View** → **Executions**)
2. Run `testScript()` to verify the Google side works
