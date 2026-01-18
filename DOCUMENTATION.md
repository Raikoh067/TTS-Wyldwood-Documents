# Tabletop Simulator Mod Documentation

This document provides a detailed overview of the scripted objects in the mod, their functions, interactions, and configuration options.

## 1. Overview

This mod uses a **Blue-Seat-Centric** setup system. This means:
*   **Blue Player's Seat** is the "anchor" for all object positions.
*   Objects for other players (Red, Purple, Green) are automatically calculated based on Blue's position, rotated around the table center.
*   **Global Script** (`Global.-1.lua`) manages the initial setup and the card display UI.

---

## 2. Core Systems

### Round Tracker
**Script:** `RoundTracker.9aa9a2.lua`
**Object Name:** `RoundTracker`

The central "brain" of the game. It manages the flow of rounds, turn order, and game state.

**Functionality:**
*   **Game Flow**: Manages the sequence of play (Start Game -> Rounds 1-6 -> End Game).
*   **Turn Order**: Tracks whose turn it is (Blue vs. Red) and alternates first player each round.
*   **Omen Cards**: Draws Omen cards from the `OmenDeck` and triggers their effects (spawning Woodwisps).
*   **Woodwisps**: Spawns Woodwisp tokens at specific coordinates based on the active Omen card.
*   **Fate Points**: Resets Fate counters at the start of each round.
*   **Win Conditions**: Checks for 10 VP win condition or end of Round 6.

**Interactions:**
*   **Communicates with:**
    *   `BlueFateCounter` / `RedFateCounter`: Sets fate points at the start of rounds.
    *   `BlueActivationDone` / `RedActivationDone`: Updates the state of activation buttons (Self vs. Other).
    *   `BluePlayerScoreTracker` / `RedPlayerScoreTracker`: Checks VP totals for win conditions.
    *   `Global`: Calls `displayOmenCard` to show the current Omen on screen.
    *   `GameTracker`: Shows/hides the "Submit Playtest Data" button at game end.
*   **Controls:**
    *   `OmenDeck`: Shuffles and draws cards.
    *   `Woodwisp Bag`: Spawns tokens from this container.

**Configuration (`CONFIG` table):**
*   `WOODWISP_SPAWN_LOCATIONS`: Coordinates for the 9 possible Woodwisp spawn points.
*   `OMEN_WOODWISP_LOCATIONS`: Maps Omen Card names to spawn location indices.
*   `BUTTON_POSITIONS`: Adjust position/rotation/size of UI buttons (Start, Reset, End Round).
*   `CLEANUP_AREA`: Defines the zone to clear when starting a new game.

---

### Global Script
**Script:** `Global.-1.lua`

The manager of the physical table state and the on-screen UI.

**Functionality:**
*   **Table Setup**: Defines the "Master" layout based on the Blue seat. Automatically calculates positions for Red (180°), Purple (90°), and Green (270°) seats.
*   **UI Manager**: Handles the XML UI for displaying cards on screen.
    *   `displayOmenCard(card)`: Shows the current Omen card.
    *   `displayActiveCard(card, color)`: Shows a player's active card.
*   **Logging**: Centralized logging system (`logAction`) for debugging and game tracking.

**Interactions:**
*   **Called by:** `RoundTracker`, `ActiveCardZone`, `FactionPicker`.

**Configuration:**
*   `PLAYER_OBJECT_SETUP`: Defines which objects go where for the Blue player.
*   `SCRIPT_ZONE_SETUP`: Defines where script zones (Deck, Discard) are created.

---

## 3. Player Objects

### Activation Token
**Script:** `BlueActivationDone.4fe029.lua` / `RedActivationDone.5d9172.lua`
**Object Name:** `BlueActivationDone` / `RedActivationDone`

A toggle button that indicates whose turn it is.

**Functionality:**
*   **Visual Indicator**: Changes color and text based on turn state.
    *   **Blue's Turn**: Blue button says "End Activation". Red button says "Blue's Activation".
    *   **Red's Turn**: Red button says "End Activation". Blue button says "Red's Activation".
*   **Click Action**: Clicking "End Activation" passes the turn to the other player.

**Interactions:**
*   **Communicates with:** `RoundTracker`.
    *   Calls `RoundTracker.call("toggleActivation")` when clicked.
    *   Receives `refreshButton` calls from `RoundTracker` to update its state.

---

### Active Card Zone
**Script:** `BlueActiveCardZone.f7ea26.lua` / `RedActiveCardZone.549d7c.lua`
**Object Name:** `BlueActiveCardZone` / `RedActiveCardZone`

A tile that detects when a card is played.

**Functionality:**
*   **Card Detection**: Uses collision detection (`onCollisionEnter`) to detect when a card is placed on top.
*   **UI Display**: Triggers the Global UI to show the card on screen for all players.
*   **Discard Button**: Creates a physical "Discard" button above the tile. Clicking it moves the card to the player's discard pile.

**Interactions:**
*   **Communicates with:**
    *   `Global`: Calls `displayActiveCard` and `hideActiveCard`.
    *   `BluePlayerDiscardZone` (Scripting Zone): Finds this zone to know where to discard cards.

---

### Fate Counter
**Script:** `BlueFateCounter.86710c.lua` / `RedFateCounter.f9c783.lua`
**Object Name:** `BlueFateCounter` / `RedFateCounter`

Tracks a player's Fate Points.

**Functionality:**
*   **Counter**: Simple +/- buttons to track a number.
*   **Persistence**: Saves its value between game loads.

**Interactions:**
*   **Controlled by:** `RoundTracker` (resets to 0 or sets to round number).
*   **Logging**: Reports changes to `Global` for logging.

**Configuration:**
*   `CONFIG.COLORS`: Customize button colors.

---

### Score Tracker
**Script:** `BluePlayerScoreTracker.8a8f5f.lua` / `RedPlayerScoreTracker.99ec09.lua`
**Object Name:** `BluePlayerScoreTracker` / `RedPlayerScoreTracker`

A comprehensive dashboard for player scoring.

**Functionality:**
*   **VP Tracking**: Automatically calculates total Victory Points based on inputs.
    *   **Woodwisps**: +1 VP each.
    *   **Champions**: +3 VP each (checkboxes).
    *   **Leader**: +5 VP (checkbox).
    *   **Other**: +1 VP (generic counter).
*   **UI**: Displays the total score prominently.

**Interactions:**
*   **Read by:** `RoundTracker` (to check for 10 VP win condition).
*   **Read by:** `GameTracker` (for data collection).

**Configuration:**
*   `CONFIG.VP_VALUES`: Change how many points each item is worth.
*   `CONFIG.POSITIONS`: Move UI elements around the board.
*   `CONFIG.MAX_COUNTS`: Change the number of checkboxes available.

---

### Faction Picker
**Script:** `LeafsongFactionPicker.9be4cd.lua` / `BoulderbreakerFactionPicker.8d94fe.lua`

A setup tool for spawning player factions.

**Functionality:**
*   **Object Storage**: Stores JSON data of faction objects (Champions, Tokens) directly in the script.
*   **Spawning**: When "Select Faction" is clicked, it spawns the stored objects.
*   **Rotation**: Automatically rotates spawned objects based on the player's seat (Blue 0°, Red 180°, etc.).
*   **Score Tracker Image**: Updates the player's Score Tracker with their faction's art.

**Interactions:**
*   **Communicates with:** `BluePlayerScoreTracker` (to update the custom image).

**Configuration:**
*   `CONFIG.FACTION`: Set the faction name and Score Tracker image URL.
*   `CONFIG.SEAT_ROTATIONS`: Define rotation angles for each seat.
*   **Setup Mode**: Use the "Setup" button to save new objects into the script memory.

---

## 4. Utility Objects

### Game Tracker
**Script:** `GameTracker.5f1e44.lua`
**Object Name:** `GameTracker`

Collects playtest data and submits it to Google Sheets.

**Functionality:**
*   **Data Collection**: Scrapes data from the table at game end:
    *   **Round Count**: From `RoundTracker`.
    *   **Scores**: From `ScoreTrackers`.
    *   **Factions**: Detects factions by looking for specific Champion GUIDs on the table.
    *   **Decks**: Scans deck zones to list cards remaining/discarded.
*   **Feedback Form**: Provides a UI for players to enter text feedback.
*   **Submission**: Sends a JSON payload to a Google Apps Script Webhook.

**Interactions:**
*   **Reads from:** `RoundTracker`, `ScoreTrackers`, Deck Zones.
*   **Communicates with:** External Webhook (Google Sheets).

**Configuration:**
*   `CONFIG.WEBHOOK_URL`: The URL of your Google Apps Script.
*   `CONFIG.FACTIONS` / `CONFIG.CHAMPIONS`: Registry of Factions and their Champion GUIDs (used for detection).

