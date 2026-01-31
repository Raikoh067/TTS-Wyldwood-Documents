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
- **Spam Protection** — 30-minute cooldown per SteamID prevents duplicate/spam submissions

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
While they aren't useful to read, they are used to store the raw data used to calculate totals. 

| Sheet | Purpose |
|-------|---------|
| **DATADeckCards** | Raw card picks (used only for Card Totals) |
| **DATAChampions** | Raw champion stats (used only for Champion Totals) |

---

## Data Format

### Match Results (2 rows per game)
| GameID | Timestamp | Faction | Player Name | SteamID | Round Ended | Win/Lose | Final Score |
|--------|-----------|---------|-------------|---------|-------------|----------|-------------|
| 00001 | 01-31-26 23:59 | Leafsong Nomads | Raikoh | 76561198012345678 | 6 | LOSE | 9 |
| 00001 | 01-31-26 23:59 | Boulderbreaker Clans | Brian | 76561198087654321 | 6 | WIN | 11 |

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

Create these 6 sheet tabs in this order (the order doesn't affect the script, it's just for clarity):

1. **MatchResults** — Rename "Sheet1" to this
2. **FactionTotals** — Click + to add
3. **ChampionTotals** — Click + to add
4. **CardTotals** — Click + to add
5. **DATADeckCards** — Click + to add
6. **DATAChampions** — Click + to add

### Step 3: Add Column Headers

#### MatchResults Sheet
| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| gameId | timestamp | faction | playerName | steamId | roundEnded | winLose | finalScore |

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
 * Includes spam protection via 30-minute cooldown per SteamID
 * Cooldown is checked against existing MatchResults - no separate sheet needed
 */

const CONFIG = {
  MATCH_RESULTS_SHEET: 'MatchResults',
  DECK_CARDS_SHEET: 'DATADeckCards',
  CHAMPIONS_SHEET: 'DATAChampions',
  FACTION_TOTALS_SHEET: 'FactionTotals',
  CHAMPION_TOTALS_SHEET: 'ChampionTotals',
  CARD_TOTALS_SHEET: 'CardTotals',
  
  // Cooldown duration in minutes
  COOLDOWN_MINUTES: 30
};

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    console.log('Received playtest data for game: ' + data.gameId);
    
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    
    // Check cooldowns by looking at MatchResults for recent submissions by these SteamIDs
    const cooldownCheck = checkCooldowns(ss, data.steamIds || []);
    if (cooldownCheck.blocked) {
      console.log('Submission blocked due to cooldown. Minutes remaining: ' + cooldownCheck.minutesRemaining);
      return ContentService
        .createTextOutput(JSON.stringify({
          status: 'cooldown',
          message: 'Please wait before submitting again',
          minutesRemaining: cooldownCheck.minutesRemaining,
          blockedSteamId: cooldownCheck.blockedSteamId
        }))
        .setMimeType(ContentService.MimeType.JSON);
    }
    
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
// COOLDOWN / SPAM PROTECTION FUNCTIONS
// =====================================

/**
 * Check if any of the provided SteamIDs have submitted within the cooldown period
 * by looking at the MatchResults sheet timestamps
 * 
 * @param {Spreadsheet} ss - The active spreadsheet
 * @param {string[]} steamIds - Array of SteamIDs to check
 * @returns {Object} - {blocked: boolean, minutesRemaining: number, blockedSteamId: string}
 */
function checkCooldowns(ss, steamIds) {
  if (!steamIds || steamIds.length === 0) {
    // No SteamIDs provided, allow submission (might be single player or test)
    return { blocked: false };
  }
  
  // Filter out "Unknown" SteamIDs
  const validSteamIds = steamIds.filter(id => id && id !== 'Unknown');
  if (validSteamIds.length === 0) {
    return { blocked: false };
  }
  
  const sheet = ss.getSheetByName(CONFIG.MATCH_RESULTS_SHEET);
  if (!sheet) {
    console.log('MatchResults sheet not found, skipping cooldown check');
    return { blocked: false };
  }
  
  const data = sheet.getDataRange().getValues();
  if (data.length <= 1) {
    // Only header row, no previous submissions
    return { blocked: false };
  }
  
  const now = new Date();
  const cooldownMs = CONFIG.COOLDOWN_MINUTES * 60 * 1000;
  
  // Find the most recent submission timestamp for each SteamID we're checking
  // MatchResults format: gameId | timestamp | faction | playerName | steamId | roundEnded | winLose | finalScore
  // Columns:               0    |     1     |    2    |     3      |    4    |     5      |    6    |     7
  
  for (let i = 1; i < data.length; i++) {
    const rowSteamId = String(data[i][4]); // Column E = steamId
    const rowTimestamp = data[i][1];       // Column B = timestamp
    
    // Check if this row's SteamID matches any we're checking
    if (validSteamIds.includes(rowSteamId)) {
      // Parse the timestamp (format: "MM-DD-YY HH:MM")
      const submissionTime = parseTimestamp(rowTimestamp);
      
      if (submissionTime) {
        const timeSinceSubmission = now - submissionTime;
        
        if (timeSinceSubmission < cooldownMs) {
          // This SteamID submitted within the cooldown period
          const minutesRemaining = Math.ceil((cooldownMs - timeSinceSubmission) / (1000 * 60));
          return {
            blocked: true,
            minutesRemaining: minutesRemaining,
            blockedSteamId: rowSteamId
          };
        }
      }
    }
  }
  
  return { blocked: false };
}

/**
 * Parse a timestamp string in format "MM-DD-YY HH:MM" into a Date object
 * @param {string|Date} timestamp - The timestamp to parse
 * @returns {Date|null} - Parsed Date or null if invalid
 */
function parseTimestamp(timestamp) {
  if (!timestamp) return null;
  
  // If it's already a Date object (Google Sheets might auto-convert), use it directly
  if (timestamp instanceof Date) {
    return timestamp;
  }
  
  // Parse string format "MM-DD-YY HH:MM"
  const str = String(timestamp);
  const match = str.match(/^(\d{2})-(\d{2})-(\d{2})\s+(\d{1,2}):(\d{2})$/);
  
  if (match) {
    const month = parseInt(match[1], 10) - 1; // JS months are 0-indexed
    const day = parseInt(match[2], 10);
    let year = parseInt(match[3], 10);
    const hour = parseInt(match[4], 10);
    const minute = parseInt(match[5], 10);
    
    // Convert 2-digit year to 4-digit (assumes 2000s)
    year = year + 2000;
    
    return new Date(year, month, day, hour, minute);
  }
  
  // Try parsing as a generic date string
  const parsed = new Date(str);
  if (!isNaN(parsed.getTime())) {
    return parsed;
  }
  
  return null;
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
    mr.steamId || 'Unknown',
    mr.roundEnded,
    mr.winLose,
    mr.finalScore
  ]);
  
  if (rows.length > 0) {
    const startRow = sheet.getLastRow() + 1;
    sheet.getRange(startRow, 1, rows.length, rows[0].length).setValues(rows);
  }
  
  // Log feedback (could be stored in a separate column or sheet if desired)
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
    const faction = data[i][2];  // Column C = faction
    const winLose = data[i][6];  // Column G = winLose
    const score = data[i][7];    // Column H = finalScore
    
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
// TEST FUNCTIONS
// =====================================

function testScript() {
  const gameId = '00001';
  const timestamp = '01-30-26 15:30';
  
  const testData = {
    gameId: gameId,
    steamIds: ['76561198012345678', '76561198087654321'],
    matchResults: [
      { gameId: gameId, timestamp: timestamp, faction: 'Leafsong Nomads', playerName: 'TestPlayer1', steamId: '76561198012345678', roundEnded: 6, winLose: 'WIN', finalScore: 11 },
      { gameId: gameId, timestamp: timestamp, faction: 'Boulderbreaker Clans', playerName: 'TestPlayer2', steamId: '76561198087654321', roundEnded: 6, winLose: 'LOSE', finalScore: 9 }
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
  
  // Check cooldowns first (like real submission would)
  const cooldownCheck = checkCooldowns(ss, testData.steamIds);
  if (cooldownCheck.blocked) {
    console.log('TEST BLOCKED: SteamID ' + cooldownCheck.blockedSteamId + ' is on cooldown. Minutes remaining: ' + cooldownCheck.minutesRemaining);
    return;
  }
  
  writeMatchResults(ss, testData.matchResults, testData.feedback);
  writeDeckCards(ss, testData.deckCards);
  writeChampions(ss, testData.champions);
  updateFactionTotals(ss);
  updateChampionTotals(ss);
  updateCardTotals(ss);
  
  console.log('Test data written successfully!');
}

// Test the cooldown check without writing data
function testCooldownCheck() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const testSteamIds = ['76561198012345678', '76561198087654321'];
  
  const result = checkCooldowns(ss, testSteamIds);
  console.log('Cooldown check result: ' + JSON.stringify(result));
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

## Spam Protection System

### How It Works

1. **SteamID Collection**: When a game ends, the TTS script collects the unique Steam IDs of both players
2. **Cooldown Check**: Before accepting a submission, the Google script searches the MatchResults sheet for any previous submissions by these SteamIDs within the last 30 minutes
3. **Rejection or Accept**: If a recent submission is found, the new submission is rejected with a message showing minutes remaining. Otherwise, the data is recorded.

### Why This Works

- **Steam IDs are unique and permanent** — Each Steam account has a unique 64-bit ID that cannot be changed
- **Server-side validation** — The cooldown check happens on Google's servers, so it can't be bypassed by modifying the TTS script

### Adjusting the Cooldown Duration

To change the 30-minute cooldown, modify the line in the Google Apps Script that looks like this:

```javascript
COOLDOWN_MINUTES: 30
```

---

## Testing Your Setup

### Test the Google Script

1. In Apps Script, select **testScript** from the function dropdown
2. Click **Run** (▶)
3. Check your sheets - you should see test data appear
4. The Totals sheets should auto-populate
5. Run **testScript** again immediately - it should be blocked by cooldown!

### Test Cooldown Functions

- **testCooldownCheck()** — Check if test SteamIDs are on cooldown based on MatchResults

### Test from TTS

1. Play a quick game in Tabletop Simulator
2. Click **Submit Playtest Data** when the game ends
3. Check Google Sheets for the new data
4. Try submitting again immediately - you should see a cooldown message

---

## Troubleshooting

### Data not appearing
- Check that all 6 sheets exist with correct names
- Check column headers match exactly
- Run `testScript()` to verify the script works

### Totals not updating
- Run `recalculateAllTotals()` manually from Apps Script
- Check for errors in Apps Script logs (View → Executions)

### Cooldown not working
- Make sure the MatchResults sheet has the correct columns including steamId (column E)
- Run `testCooldownCheck()` to debug
- Check that timestamps are in the expected format

### "Unknown" SteamIDs
- This happens when a player seat is empty or when testing in single-player
- Submissions with "Unknown" SteamIDs are still allowed (they don't trigger cooldowns)

---
