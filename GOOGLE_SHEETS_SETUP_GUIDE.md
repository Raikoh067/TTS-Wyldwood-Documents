# Wyldwood Playtest Data Collection
## Google Sheets Setup Guide

---

## What This System Does

This playtest data collection system automatically captures detailed game statistics from your Wyldwood sessions and sends them to a Google Sheets database. This is focused on Totals - providing summary statistics at a glance.

### Key Features:
- **Faction Totals** — Win rates, average scores per faction
- **Champion Totals** — Average health, woodwisps held, death rates per champion
- **Card Totals** — How many times each card has been taken across all games
- **Match Results** — Complete log of every match played

---

## Sheet Structure Overview

### Main Sheets (Auto-Updated Summary Stats)
These sheets show running totals that update automatically after each match submission.

| Sheet | Purpose |
|-------|---------|
| **MatchResults** | Two rows per match (one per faction), the main match log |
| **FactionTotals** | Win %, average score per faction |
| **ChampionTotals** | Average wisps, average health, death % per champion |
| **CardTotals** | Times taken per card per faction |

### Raw Data Sheets (Individual Match Data)
While they aren't usefull to read, they are used to store the raw data used to calculate totals. 

| Sheet | Purpose |
|-------|---------|
| **DATADeckCards** | Raw card picks (used only for Card Totals) |
| **DATAChampions** | Raw champion stats (used only for Champion Totals) |

---

## Data Format

### Match Results (2 rows per game)
| GameID | Timestamp | Faction | Player Name | Round Ended | Win/Lose | Final Score |
|--------|-----------|---------|-------------|-------------|----------|-------------|
| 00001 | 01-31-26 23:59 | Leafsong Nomads | Raikoh | 6 | LOSE | 9 |
| 00001 | 01-31-26 23:59 | Boulderbreaker Clans | Brian | 6 | WIN | 11 |

### Faction Totals
| Faction | Win % | Average Score | Games Played |
|---------|-------|---------------|--------------|
| Leafsong Nomads | 52% | 9.3 | 25 |
| Boulderbreaker Clans | 48% | 9.1 | 25 |

### Champion Totals
| Faction | Champion Name | Average Wisps Held | Average Health | Death % |
|---------|---------------|--------------------| ---------------|---------|
| Leafsong Nomads | Loresinger | 1.8 | 6.5 | 19% |

### Card Totals
| Faction | Card Name | Times Taken |
|---------|-----------|-------------|
| Leafsong Nomads | Prancing Stride | 98 |

---

## Setup Instructions

### Step 1: Create Your Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Click the **+ Blank** button to create a new spreadsheet
3. Name it **"Wyldwood Playtest Data"**

### Step 2: Create the Required Sheets

Create these 6 sheet tabs in this order (the order doesn’t affect the script, it’s just for clarity):

1. **MatchResults** — Rename "Sheet1" to this
2. **FactionTotals** — Click + to add
3. **ChampionTotals** — Click + to add
4. **CardTotals** — Click + to add
5. **DATADeckCards** — Click + to add
6. **DATAChampions** — Click + to add

### Step 3: Add Column Headers

#### MatchResults Sheet
| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| gameId | timestamp | faction | playerName | roundEnded | winLose | finalScore |

#### DATADeckCards Sheet
| A | B | C | D |
|---|---|---|---|
| gameId | timestamp | faction | cardName |

#### DATAChampions Sheet
| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| gameId | timestamp | faction | championName | finalHealth | maxHealth | woodwisps | isDead |

#### FactionTotals Sheet
| A | B | C | D |
|---|---|---|---|
| faction | winPercent | avgScore | gamesPlayed |

#### ChampionTotals Sheet
| A | B | C | D | E |
|---|---|---|---|---|
| faction | championName | avgWisps | avgHealth | deathPercent |

#### CardTotals Sheet
| A | B | C |
|---|---|---|
| faction | cardName | timesTaken |

### Step 4: Open Google Apps Script

1. In your Google Sheet, click **Extensions** → **Apps Script**
2. **Delete** everything in the editor
3. Paste the script below

### Step 5: Paste the Webhook Script

```javascript
/**
 * Wyldwood Playtest Data Receiver
 * Receives game data from Tabletop Simulator and writes to Google Sheets
 */

const CONFIG = {
  MATCH_RESULTS_SHEET: 'MatchResults',
  DECK_CARDS_SHEET: 'DATADeckCards',
  CHAMPIONS_SHEET: 'DATAChampions',
  FACTION_TOTALS_SHEET: 'FactionTotals',
  CHAMPION_TOTALS_SHEET: 'ChampionTotals',
  CARD_TOTALS_SHEET: 'CardTotals'
};

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    console.log('Received playtest data for game: ' + data.gameId);
    
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    
    // Write raw data
    writeMatchResults(ss, data.matchResults, data.feedback);
    writeDeckCards(ss, data.deckCards);
    writeChampions(ss, data.champions);
    
    // Update totals
    updateFactionTotals(ss);
    updateChampionTotals(ss);
    updateCardTotals(ss);
    
    return ContentService
      .createTextOutput(JSON.stringify({
        status: 'success',
        message: 'Data recorded and totals updated',
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

// =====================================
// RAW DATA WRITING FUNCTIONS
// =====================================

function writeMatchResults(ss, matchResults, feedback) {
  const sheet = ss.getSheetByName(CONFIG.MATCH_RESULTS_SHEET);
  if (!sheet) throw new Error('MatchResults sheet not found');
  
  const rows = matchResults.map(mr => [
    mr.gameId,
    mr.timestamp,
    mr.faction,
    mr.playerName,
    mr.roundEnded,
    mr.winLose,
    mr.finalScore
  ]);
  
  if (rows.length > 0) {
    const startRow = sheet.getLastRow() + 1;
    sheet.getRange(startRow, 1, rows.length, rows[0].length).setValues(rows);
  }
  
  // Store feedback in the first row of each match (column H onwards if desired)
  // For now, feedback is logged but not stored separately
  if (feedback && (feedback.cardFeedback || feedback.generalFeedback)) {
    console.log('Card Feedback: ' + (feedback.cardFeedback || 'None'));
    console.log('General Feedback: ' + (feedback.generalFeedback || 'None'));
  }
}

function writeDeckCards(ss, deckCards) {
  const sheet = ss.getSheetByName(CONFIG.DECK_CARDS_SHEET);
  if (!sheet) throw new Error('DATADeckCards sheet not found');
  
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

function writeChampions(ss, champions) {
  const sheet = ss.getSheetByName(CONFIG.CHAMPIONS_SHEET);
  if (!sheet) throw new Error('DATAChampions sheet not found');
  
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

// =====================================
// TOTALS UPDATE FUNCTIONS
// =====================================

function updateFactionTotals(ss) {
  const matchSheet = ss.getSheetByName(CONFIG.MATCH_RESULTS_SHEET);
  const totalsSheet = ss.getSheetByName(CONFIG.FACTION_TOTALS_SHEET);
  if (!matchSheet || !totalsSheet) return;
  
  const data = matchSheet.getDataRange().getValues();
  if (data.length <= 1) return; // Only header row
  
  // Skip header, aggregate by faction
  const factionStats = {};
  for (let i = 1; i < data.length; i++) {
    const faction = data[i][2]; // Column C = faction
    const winLose = data[i][5]; // Column F = winLose
    const score = data[i][6];   // Column G = finalScore
    
    if (!faction) continue;
    
    if (!factionStats[faction]) {
      factionStats[faction] = { wins: 0, games: 0, totalScore: 0 };
    }
    
    factionStats[faction].games++;
    factionStats[faction].totalScore += (score || 0);
    if (winLose === 'WIN') {
      factionStats[faction].wins++;
    }
  }
  
  // Build output rows
  const outputRows = [['faction', 'winPercent', 'avgScore', 'gamesPlayed']];
  for (const faction in factionStats) {
    const stats = factionStats[faction];
    const winPercent = stats.games > 0 ? Math.round((stats.wins / stats.games) * 100) + '%' : '0%';
    const avgScore = stats.games > 0 ? (stats.totalScore / stats.games).toFixed(1) : '0';
    outputRows.push([faction, winPercent, avgScore, stats.games]);
  }
  
  // Clear and write
  totalsSheet.clear();
  totalsSheet.getRange(1, 1, outputRows.length, outputRows[0].length).setValues(outputRows);
}

function updateChampionTotals(ss) {
  const champSheet = ss.getSheetByName(CONFIG.CHAMPIONS_SHEET);
  const totalsSheet = ss.getSheetByName(CONFIG.CHAMPION_TOTALS_SHEET);
  if (!champSheet || !totalsSheet) return;
  
  const data = champSheet.getDataRange().getValues();
  if (data.length <= 1) return;
  
  // Aggregate by faction + champion
  const champStats = {};
  for (let i = 1; i < data.length; i++) {
    const faction = data[i][2];      // Column C
    const champName = data[i][3];    // Column D
    const health = data[i][4] || 0;  // Column E
    const wisps = data[i][6] || 0;   // Column G
    const isDead = data[i][7];       // Column H
    
    if (!faction || !champName) continue;
    
    const key = faction + '|' + champName;
    if (!champStats[key]) {
      champStats[key] = { faction, champName, totalHealth: 0, totalWisps: 0, deaths: 0, count: 0 };
    }
    
    champStats[key].totalHealth += health;
    champStats[key].totalWisps += wisps;
    champStats[key].count++;
    if (isDead === true || isDead === 'TRUE' || isDead === 'true') {
      champStats[key].deaths++;
    }
  }
  
  // Build output
  const outputRows = [['faction', 'championName', 'avgWisps', 'avgHealth', 'deathPercent']];
  for (const key in champStats) {
    const s = champStats[key];
    const avgWisps = (s.totalWisps / s.count).toFixed(1);
    const avgHealth = (s.totalHealth / s.count).toFixed(1);
    const deathPercent = Math.round((s.deaths / s.count) * 100) + '%';
    outputRows.push([s.faction, s.champName, avgWisps, avgHealth, deathPercent]);
  }
  
  totalsSheet.clear();
  totalsSheet.getRange(1, 1, outputRows.length, outputRows[0].length).setValues(outputRows);
}

function updateCardTotals(ss) {
  const cardSheet = ss.getSheetByName(CONFIG.DECK_CARDS_SHEET);
  const totalsSheet = ss.getSheetByName(CONFIG.CARD_TOTALS_SHEET);
  if (!cardSheet || !totalsSheet) return;
  
  const data = cardSheet.getDataRange().getValues();
  if (data.length <= 1) return;
  
  // Aggregate by faction + card
  const cardStats = {};
  for (let i = 1; i < data.length; i++) {
    const faction = data[i][2];   // Column C
    const cardName = data[i][3];  // Column D
    
    if (!faction || !cardName) continue;
    
    const key = faction + '|' + cardName;
    if (!cardStats[key]) {
      cardStats[key] = { faction, cardName, count: 0 };
    }
    cardStats[key].count++;
  }
  
  // Build output sorted by faction, then by count descending
  const outputRows = [['faction', 'cardName', 'timesTaken']];
  const sortedKeys = Object.keys(cardStats).sort((a, b) => {
    const statsA = cardStats[a];
    const statsB = cardStats[b];
    if (statsA.faction !== statsB.faction) {
      return statsA.faction.localeCompare(statsB.faction);
    }
    return statsB.count - statsA.count; // Higher count first
  });
  
  for (const key of sortedKeys) {
    const s = cardStats[key];
    outputRows.push([s.faction, s.cardName, s.count]);
  }
  
  totalsSheet.clear();
  totalsSheet.getRange(1, 1, outputRows.length, outputRows[0].length).setValues(outputRows);
}

// =====================================
// TEST FUNCTION
// =====================================

function testScript() {
  const gameId = '00001';
  const timestamp = '01-30-26 15:30';
  
  const testData = {
    gameId: gameId,
    matchResults: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', playerName: 'TestPlayer1', roundEnded: 6, winLose: 'WIN', finalScore: 11 },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', playerName: 'TestPlayer2', roundEnded: 6, winLose: 'LOSE', finalScore: 9 }
    ],
    deckCards: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', cardName: 'Savage Rend' },
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', cardName: 'Savage Rend' },
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', cardName: 'Prancing Stride' },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', cardName: 'Boulder Smash' },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', cardName: 'Boulder Smash' }
    ],
    champions: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', championName: 'Loresinger', finalHealth: 5, maxHealth: 10, woodwisps: 2, isDead: false },
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', championName: 'Wyldspeaker', finalHealth: 0, maxHealth: 8, woodwisps: 0, isDead: true },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', championName: 'UrscarKing', finalHealth: 8, maxHealth: 21, woodwisps: 1, isDead: false }
    ],
    feedback: {
      cardFeedback: 'Savage Rend feels too strong',
      generalFeedback: 'Great game!'
    }
  };
  
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  writeMatchResults(ss, testData.matchResults, testData.feedback);
  writeDeckCards(ss, testData.deckCards);
  writeChampions(ss, testData.champions);
  updateFactionTotals(ss);
  updateChampionTotals(ss);
  updateCardTotals(ss);
  
  console.log('Test data written successfully!');
}

// Manually recalculate all totals (run this if totals seem off)
function recalculateAllTotals() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  updateFactionTotals(ss);
  updateChampionTotals(ss);
  updateCardTotals(ss);
  console.log('All totals recalculated!');
}
```

### Step 6: Save and Deploy

1. Click the **floppy disk icon** to save
2. Name the project **"Wyldwood Data Receiver"**
3. Click **Deploy** → **New deployment**
4. Click the **gear icon** → select **Web app**
5. Set:
  - **Description:** `Wyldwood`
   - **Execute as:** `Me`
   - **Who has access:** `Anyone`
6. Click **Deploy**
7. **Authorize** when prompted (click Advanced → Go to Wyldwood... if needed)
8. **Copy the Web App URL**

### Step 7: Update the TTS Script

In Tabletop Simulator:
1. Find the **GameTracker** object
2. Open its Lua script
3. Find the `WEBHOOK_URL` line near the top
4. Replace it with your new Web App URL
5. Save

---

## Testing Your Setup

### Test the Google Script

1. In Apps Script, select **testScript** from the function dropdown
2. Click **Run** (▶)
3. Check your sheets - you should see test data appear
4. The Totals sheets should auto-populate

### Test from TTS

1. Play a quick game in Tabletop Simulator
2. Click **Submit Playtest Data** when the game ends
3. Check Google Sheets for the new data

---

## Understanding the Totals

### Faction Totals
Shows overall balance between factions:
- **Win %** — Higher = faction might be too strong
- **Avg Score** — Compare across factions for balance
- **Games Played** — Confidence in the data

### Champion Totals
Shows which champions are performing well:
- **Avg Wisps** — Higher = champion is collecting more wisps
- **Avg Health** — Higher = champion survives with more health
- **Death %** — Higher = champion dies more often (might be too weak or focused)

### Card Totals
Shows card popularity by faction:
- **Times Taken** — Higher = card is popular (powerful? necessary? or just fun?)
- Compare similar cards to see which gets picked more

---

## Troubleshooting

### Data not appearing
- Check that all 6 sheets exist with correct names
- Check column headers match exactly
- Run `testScript()` to verify the script works

### Totals not updating
- Run `recalculateAllTotals()` manually from Apps Script
- Check for errors in Apps Script logs (View → Executions)

### "Unknown" faction appearing
- Check that champion GUIDs in the TTS script match your actual champions
- The faction detection relies on finding champions by GUID

### Scores showing as 0
- Make sure the game actually ended (scores should be on the score trackers)
- Check that score trackers have the `getTotalVP` function

---

## Current Champion Reference

| Leafsong Nomads | Boulderbreaker Clans |
|-----------------|---------------------|
| Loresinger | UrscarKing |
| HighSpiritseer | FellcastHunter |
| BonebladeAlpha | MakaraElder |
| Wyldspeaker | OutcastKingsguard |
| WyrdbowSentinel | Painseeker |

*New factions will automatically appear in totals when added to the game!*
