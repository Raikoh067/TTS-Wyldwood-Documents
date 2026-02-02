# Google Sheets Playtest Data Collection Setup Guide

This guide explains how to set up Google Sheets to receive playtest data from Tabletop Simulator.

## Overview

The system works as follows:
1. TTS sends playtest data (including player SteamIDs) from the GameTracker.5f1e44.lua object via HTTP POST to a Google Apps Script web app
2. The script checks if any player has submitted data in the last 30 minutes (spam protection)
3. If cooldown check passes, data is written to the appropriate sheets
4. The script returns success/failure status back to TTS

## Sheet Structure

Your Google Sheet needs **5 sheets** with these exact names:
1. **MatchResults** - Raw match data (timestamp, gameId, winner, loser, factions, steamIds)
2. **FactionTotals** - Aggregated win/loss stats per faction
3. **ChampionTotals** - Aggregated win/loss stats per champion
4. **CardTotals** - Aggregated card usage statistics
5. **Feedback** - Player feedback after each match

## Setup Instructions

### Step 1: Create the Google Sheet

1. Go to [Google Sheets](https://sheets.google.com)
2. Create a new blank spreadsheet
3. Name it something like "Playtest Data"

### Step 2: Create the Required Sheets

Create 5 sheets with these exact names and column headers:

#### Sheet 1: MatchResults
Column headers (Row 1):
```
timestamp | gameId | winningFaction | losingFaction | winningPlayer | losingPlayer | winningSteamId | losingSteamId
```

#### Sheet 2: FactionTotals
Column headers (Row 1):
```
faction | wins | losses | winRate (displays as percentage)
```

#### Sheet 3: ChampionTotals
Column headers (Row 1):
```
champion | faction | deaths | appearances | deathRate (displays as percentage)
```

#### Sheet 4: CardTotals
Column headers (Row 1):
```
cardName | faction | timesTaken
```

#### Sheet 5: Feedback
Column headers (Row 1):
```
gameId | winningFaction | winningPlayerName | losingPlayerName | cardFeedback | gameFeedback
```

### Step 3: Create the Google Apps Script

1. In your Google Sheet, go to **Extensions > Apps Script**
2. Delete any existing code in the editor
3. Paste the following code:

```javascript
// ============================================
// CONFIGURATION
// ============================================
const CONFIG = {
  MATCH_RESULTS_SHEET: 'MatchResults',
  FACTION_TOTALS_SHEET: 'FactionTotals',
  CHAMPION_TOTALS_SHEET: 'ChampionTotals',
  CARD_TOTALS_SHEET: 'CardTotals',
  FEEDBACK_SHEET: 'Feedback',
  COOLDOWN_MINUTES: 30,
  ENABLE_STEAMID_CHECK: true // Set to false to disable cooldown checks
};

// ============================================
// MAIN ENTRY POINT
// ============================================
function doPost(e) {
  try {
    // Check if postData exists
    if (!e || !e.postData || !e.postData.contents) {
       return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: 'no_data',
        message: 'No POST data received'
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    const data = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    
    // Check cooldown for all submitted SteamIDs
    const cooldownResult = checkCooldowns(ss, data.steamIds || []);
    if (!cooldownResult.allowed) {
      return ContentService.createTextOutput(JSON.stringify({
        success: false,
        error: 'cooldown',
        message: cooldownResult.message,
        remainingMinutes: cooldownResult.remainingMinutes
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    // Write the data to sheets
    writeMatchResults(ss, data);
    writeFeedback(ss, data);
    
    // Update aggregated totals
    updateFactionTotals(ss, data);
    updateChampionTotals(ss, data);
    updateCardTotals(ss, data);
    
    return ContentService.createTextOutput(JSON.stringify({
      success: true,
      message: 'Data recorded successfully'
    })).setMimeType(ContentService.MimeType.JSON);
    
  } catch (error) {
    // Log error for debugging
    Logger.log('Error in doPost: ' + error.toString());
    if (error.stack) Logger.log(error.stack);
    
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: 'server_error',
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

// ============================================
// COOLDOWN FUNCTIONS
// ============================================
function checkCooldowns(ss, steamIds) {
  // Return early if check is disabled
  if (!CONFIG.ENABLE_STEAMID_CHECK) {
    Logger.log('SteamID check disabled in CONFIG');
    return { allowed: true };
  }

  if (!steamIds || steamIds.length === 0) {
    // No SteamIDs provided - allow submission but log warning
    Logger.log('Warning: No SteamIDs provided for cooldown check');
    return { allowed: true };
  }
  
  const sheet = ss.getSheetByName(CONFIG.MATCH_RESULTS_SHEET);
  if (!sheet) {
    Logger.log('Warning: MatchResults sheet not found');
    return { allowed: true };
  }
  
  const now = new Date();
  const cooldownMs = CONFIG.COOLDOWN_MINUTES * 60 * 1000;
  const cutoffTime = new Date(now.getTime() - cooldownMs);
  
  // Get all data from MatchResults
  const data = sheet.getDataRange().getValues();
  if (data.length <= 1) {
    // Only header row or empty - no previous submissions
    return { allowed: true };
  }
  
  // Find column indices (assuming headers in row 1)
  const headers = data[0];
  const timestampCol = headers.indexOf('timestamp');
  const winningSteamIdCol = headers.indexOf('winningSteamId');
  const losingSteamIdCol = headers.indexOf('losingSteamId');
  
  if (timestampCol === -1 || winningSteamIdCol === -1 || losingSteamIdCol === -1) {
    Logger.log('Warning: Required columns not found in MatchResults');
    return { allowed: true };
  }
  
  // Check each row for matching SteamIDs within cooldown period
  for (let i = data.length - 1; i >= 1; i--) {
    const row = data[i];
    const rowTimestamp = parseTimestamp(row[timestampCol]);
    
    if (!rowTimestamp || rowTimestamp < cutoffTime) {
      // This row is outside cooldown period, and since we're going backwards
      // through time, all remaining rows will also be outside cooldown
      break;
    }
    
    const rowWinnerSteamId = String(row[winningSteamIdCol]);
    const rowLoserSteamId = String(row[losingSteamIdCol]);
    
    // Check if any submitted SteamID matches this row
    for (const steamId of steamIds) {
      if (String(steamId) === rowWinnerSteamId || String(steamId) === rowLoserSteamId) {
        const elapsedMs = now.getTime() - rowTimestamp.getTime();
        const remainingMs = cooldownMs - elapsedMs;
        const remainingMinutes = Math.ceil(remainingMs / 60000);
        
        return {
          allowed: false,
          message: `SteamID ${steamId} submitted data ${Math.floor(elapsedMs / 60000)} minutes ago. Please wait ${remainingMinutes} more minute(s).`,
          remainingMinutes: remainingMinutes
        };
      }
    }
  }
  
  return { allowed: true };
}

function parseTimestamp(value) {
  if (!value) return null;
  
  // If it's already a Date object
  if (value instanceof Date) {
    return value;
  }
  
  // If it's a string, try to parse it
  if (typeof value === 'string') {
    const parsed = new Date(value);
    if (!isNaN(parsed.getTime())) {
      return parsed;
    }
  }
  
  // If it's a number (Excel serial date)
  if (typeof value === 'number') {
    // Google Sheets uses 1899-12-30 as epoch
    const excelEpoch = new Date(1899, 11, 30);
    return new Date(excelEpoch.getTime() + value * 24 * 60 * 60 * 1000);
  }
  
  return null;
}

// ============================================
// DATA WRITING FUNCTIONS
// ============================================
function writeMatchResults(ss, data) {
  const sheet = ss.getSheetByName(CONFIG.MATCH_RESULTS_SHEET);
  if (!sheet || !data.matchResults) return;
  
  const mr = data.matchResults;
  
  // Extract SteamIDs from the steamIds array
  // Format: [winningSteamId, losingSteamId]
  const winningSteamId = data.steamIds && data.steamIds.length > 0 ? data.steamIds[0] : '';
  const losingSteamId = data.steamIds && data.steamIds.length > 1 ? data.steamIds[1] : '';
  
  const row = [
    new Date(),                    // timestamp
    data.gameId || '',             // gameId
    mr.winningFaction || '',       // winningFaction
    mr.losingFaction || '',        // losingFaction
    mr.winningPlayer || '',        // winningPlayer
    mr.losingPlayer || '',         // losingPlayer
    winningSteamId,                // winningSteamId
    losingSteamId                  // losingSteamId
  ];
  
  sheet.appendRow(row);
}

function writeFeedback(ss, data) {
  const sheet = ss.getSheetByName(CONFIG.FEEDBACK_SHEET);
  if (!sheet) {
    Logger.log('Warning: Feedback sheet not found');
    return;
  }
  
  // Only write if there's actual feedback content
  const cardFeedback = data.feedback?.cardFeedback || '';
  const gameFeedback = data.feedback?.gameFeedback || '';
  
  if (!cardFeedback && !gameFeedback) {
    Logger.log('No feedback provided - skipping feedback write');
    return;
  }
  
  const mr = data.matchResults || {};
  
  const row = [
    data.gameId || '',             // gameId
    mr.winningFaction || '',       // winningFaction
    mr.winningPlayer || '',        // winningPlayerName
    mr.losingPlayer || '',         // losingPlayerName
    cardFeedback,                  // cardFeedback
    gameFeedback                   // gameFeedback
  ];
  
  sheet.appendRow(row);
  Logger.log('Feedback written: ' + JSON.stringify(row));
}

// ============================================
// AGGREGATION FUNCTIONS
// ============================================
function updateFactionTotals(ss, data) {
  if (!data.matchResults) return;
  
  const sheet = ss.getSheetByName(CONFIG.FACTION_TOTALS_SHEET);
  if (!sheet) return;
  
  const mr = data.matchResults;
  const winningFaction = mr.winningFaction;
  const losingFaction = mr.losingFaction;
  
  if (!winningFaction || !losingFaction) return;
  
  // Get existing data
  const dataRange = sheet.getDataRange();
  const values = dataRange.getValues();
  
  // Find or create rows for each faction
  let winnerRow = -1;
  let loserRow = -1;
  
  for (let i = 1; i < values.length; i++) {
    if (values[i][0] === winningFaction) winnerRow = i + 1;
    if (values[i][0] === losingFaction) loserRow = i + 1;
  }
  
  // Update winner
  if (winnerRow === -1) {
    const newRow = sheet.getLastRow() + 1;
    sheet.appendRow([winningFaction, 1, 0, '=TEXT(B' + newRow + '/(B' + newRow + '+C' + newRow + '),"0%")']);
  } else {
    const currentWins = sheet.getRange(winnerRow, 2).getValue() || 0;
    sheet.getRange(winnerRow, 2).setValue(currentWins + 1);
  }
  
  // Update loser
  if (loserRow === -1) {
    const newRow = sheet.getLastRow() + 1;
    sheet.appendRow([losingFaction, 0, 1, '=TEXT(B' + newRow + '/(B' + newRow + '+C' + newRow + '),"0%")']);
  } else {
    const currentLosses = sheet.getRange(loserRow, 3).getValue() || 0;
    sheet.getRange(loserRow, 3).setValue(currentLosses + 1);
  }
}

function updateChampionTotals(ss, data) {
  // Champions are tracked globally (not per-player).
  // Each champion entry should include current HP so death can be inferred.
  if (!data.champions || !Array.isArray(data.champions)) return;
  
  const sheet = ss.getSheetByName(CONFIG.CHAMPION_TOTALS_SHEET);
  if (!sheet) return;
  
  // Process each champion's death status
  for (const champ of data.champions) {
    if (champ.name) {
      const died = getChampionDiedFlag(champ);
      updateChampionRow(sheet, champ.name, champ.faction || '', died);
    }
  }
}

function getChampionDiedFlag(champ) {
  // Prefer HP-based death detection if provided
  if (typeof champ.health === 'number') {
    return champ.health <= 0;
  }
  // Fallback to explicit died flag if present
  if (typeof champ.died === 'boolean') {
    return champ.died;
  }
  return false;
}

function updateChampionRow(sheet, champion, faction, died) {
  const dataRange = sheet.getDataRange();
  const values = dataRange.getValues();
  
  let championRow = -1;
  for (let i = 1; i < values.length; i++) {
    if (values[i][0] === champion) {
      championRow = i + 1;
      break;
    }
  }
  
  if (championRow === -1) {
    // Add new champion: champion | faction | deaths | appearances | deathRate
    const newRow = sheet.getLastRow() + 1;
    sheet.appendRow([
      champion, 
      faction, 
      died ? 1 : 0,  // deaths
      1,              // appearances
      '=TEXT(C' + newRow + '/D' + newRow + ',"0%")'  // deathRate as percentage
    ]);
  } else {
    // Update existing
    if (died) {
      const currentDeaths = sheet.getRange(championRow, 3).getValue() || 0;
      sheet.getRange(championRow, 3).setValue(currentDeaths + 1);
    }
    const currentAppearances = sheet.getRange(championRow, 4).getValue() || 0;
    sheet.getRange(championRow, 4).setValue(currentAppearances + 1);
  }
}

function updateCardTotals(ss, data) {
  if (!data.deckCards) return;
  
  const sheet = ss.getSheetByName(CONFIG.CARD_TOTALS_SHEET);
  if (!sheet) return;
  
  // Process all deck cards (winner and loser combined)
  const allCards = [];
  if (data.deckCards.winner) {
    for (const card of data.deckCards.winner) {
      allCards.push(card);
    }
  }
  if (data.deckCards.loser) {
    for (const card of data.deckCards.loser) {
      allCards.push(card);
    }
  }
  
  for (const card of allCards) {
    updateCardRow(sheet, card.name, card.faction || '');
  }
}

function updateCardRow(sheet, cardName, faction) {
  const dataRange = sheet.getDataRange();
  const values = dataRange.getValues();
  
  let cardRow = -1;
  for (let i = 1; i < values.length; i++) {
    if (values[i][0] === cardName) {
      cardRow = i + 1;
      break;
    }
  }
  
  if (cardRow === -1) {
    // Add new card: cardName | faction | timesTaken
    sheet.appendRow([cardName, faction, 1]);
  } else {
    // Update existing - increment timesTaken (column 3)
    const currentCount = sheet.getRange(cardRow, 3).getValue() || 0;
    sheet.getRange(cardRow, 3).setValue(currentCount + 1);
  }
}

// ============================================
// TESTING FUNCTION
// ============================================
function testScript() {
  try {
    Logger.log('Starting testScript execution...');
    
    // Test with sample data
    const testData = {
      gameId: 'TEST-' + Date.now(),
      steamIds: ['76561198012345678', '76561198087654321'],
      matchResults: {
        winningFaction: 'Leafsong Nomads',
        losingFaction: 'Boulderbreaker Clans',
        winningPlayer: 'TestPlayer1',
        losingPlayer: 'TestPlayer2'
      },
      deckCards: {
        winner: [
          { name: 'Nature\'s Wrath', faction: 'Leafsong Nomads' },
          { name: 'Vine Whip', faction: 'Leafsong Nomads' }
        ],
        loser: [
          { name: 'Rock Slam', faction: 'Boulderbreaker Clans' },
          { name: 'Mountain Shield', faction: 'Boulderbreaker Clans' }
        ]
      },
      champions: [
        { name: 'HighSpiritseer', faction: 'Leafsong Nomads', health: 5, maxHealth: 8 },
        { name: 'Loresinger', faction: 'Leafsong Nomads', health: 0, maxHealth: 7 },
        { name: 'UrscarKing', faction: 'Boulderbreaker Clans', health: 0, maxHealth: 10 },
        { name: 'MakaraElder', faction: 'Boulderbreaker Clans', health: 3, maxHealth: 6 }
      ],
      feedback: {
        cardFeedback: 'Test card feedback - some cards feel overpowered',
        gameFeedback: 'Test game feedback - really enjoyed the match!'
      }
    };
    
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    if (!ss) {
      throw new Error('SpreadsheetApp.getActiveSpreadsheet() returned null. Make sure the script is container-bound to a Google Sheet.');
    }
    
    // Test cooldown check
    Logger.log('Testing cooldown check...');
    const cooldownResult = checkCooldowns(ss, testData.steamIds);
    Logger.log('Cooldown result: ' + JSON.stringify(cooldownResult));
    
    if (cooldownResult.allowed) {
      // Write test data
      Logger.log('Writing test match results...');
      writeMatchResults(ss, testData);
      
      Logger.log('Writing test feedback...');
      writeFeedback(ss, testData);
      
      Logger.log('Updating faction totals...');
      updateFactionTotals(ss, testData);
      
      Logger.log('Updating champion totals...');
      updateChampionTotals(ss, testData);
      
      Logger.log('Updating card totals...');
      updateCardTotals(ss, testData);
      
      Logger.log('Test completed successfully!');
    } else {
      Logger.log('Cooldown active - skipping write test');
    }
    
  } catch (error) {
    Logger.log('FATAL ERROR in testScript: ' + error.toString());
    if (error.stack) {
      Logger.log('Stack trace: ' + error.stack);
    }
    throw error;
  }
}
```

### Step 4: Deploy as Web App

1. Click **Deploy > New deployment**
2. Click the gear icon next to "Select type" and choose **Web app**
3. Configure:
   - **Description**: "Playtest Data Collector"
   - **Execute as**: "Me"
   - **Who has access**: "Anyone"
4. Click **Deploy**
5. **Authorize** the app when prompted
6. **Copy the Web App URL** - you'll need this for the Lua script in TTS 

### Step 5: Configure TTS

Update the `GOOGLE_SHEETS_URL` in your TTS script (GameTracker.lua) with the Web App URL you copied. It looks like this in the TTS script:

```lua
local GOOGLE_SHEETS_URL = "https://script.google.com/macros/s/YOUR_DEPLOYMENT_ID/exec"
```

## Testing

### Test from Google Sheets

1. In Apps Script, select `testScript` from the function dropdown
2. Click **Run**
3. Check the **Execution log** for results
4. Verify data appears in your sheets (MatchResults, Feedback, FactionTotals, etc.)

### Test from TTS

1. Play a complete game in TTS
2. Submit the playtest data
3. Check your Google Sheet for the new entry
4. Try submitting again within 30 minutes - you should see the cooldown message

## Troubleshooting

### "Cooldown active" when it shouldn't be
- Check the MatchResults sheet for unexpected entries
- Verify the timestamp column is formatted correctly
- Check that SteamIDs are being recorded properly

### Data not appearing
- Check the Apps Script execution logs for errors
- Verify the Web App URL is correct in TTS
- Make sure all sheet names match exactly (case-sensitive)

### Feedback not recording
- Ensure the "Feedback" sheet exists with exact name
- Check that feedback fields are not empty when submitting
- Review Apps Script logs for any errors

### Authorization errors
- Re-deploy the web app
- Make sure "Anyone" has access
- Try re-authorizing the script

## Data Flow Summary

```
TTS Game Complete
       ↓
Collect Data (including SteamIDs)
       ↓
POST to Google Apps Script
       ↓
Check Cooldowns (scan MatchResults for recent SteamID submissions)
       ↓
   [Cooldown Active?]
       ↓ No              ↓ Yes
Write to Sheets    Return Error to TTS
       ↓
Return Success to TTS
```

## Payload Format Reference

The TTS script sends data in this format:

```json
{
  "gameId": "unique-game-id",
  "steamIds": ["winningSteamId", "losingSteamId"],
  "matchResults": {
    "winningFaction": "Faction Name",
    "losingFaction": "Faction Name",
    "winningPlayer": "Player Name",
    "losingPlayer": "Player Name"
  },
  "deckCards": {
    "winner": [{"name": "Card Name", "faction": "Faction"}],
    "loser": [{"name": "Card Name", "faction": "Faction"}]
  },
  "champions": [
    {"name": "Champion Name", "faction": "Faction", "health": 0, "maxHealth": 10},
    {"name": "Champion Name", "faction": "Faction", "health": 6, "maxHealth": 10}
  ],
  "feedback": {
    "cardFeedback": "Player's card feedback text",
    "gameFeedback": "Player's general game feedback text"
  }
}
```

**Note:** Only the host can submit playtest data. The submit button and feedback UI are only visible to the host.
```
