# Complete Guide: Auto-Trainer Creation Without Cheat Engine GUI

## Part 1: Manual Text Editor Method

### Step 1: Understanding CT File Structure & Safe Deletions

A typical CT file has this XML structure:
```xml
<CheatTable>
  <Forms>...</Forms>                    <!-- SAFE TO DELETE -->
  <CheatEntries>...</CheatEntries>      <!-- KEEP - Contains actual cheats -->
  <UserdefinedSymbols>...</UserdefinedSymbols>  <!-- KEEP - Memory addresses -->
  <LuaScript>...</LuaScript>           <!-- REPLACE - Contains UI logic -->
</CheatTable>
```

#### Safe to Delete Sections:

1. **`<Forms>` section** - Contains GUI form data, completely unnecessary for auto-trainers
2. **`<Hotkeys>` blocks** - Found inside `<CheatEntry>` elements, remove all of these:
   ```xml
   <Hotkeys>
     <Hotkey>...</Hotkey>
   </Hotkeys>
   ```
3. **Visual options** - Remove these attributes from `<CheatEntry>` tags:
   - `<Options moHideChildren="1"/>`
   - Any UI-related metadata

#### What to Keep:

1. **`<CheatEntries>`** - The actual cheat code and assembly scripts
2. **`<UserdefinedSymbols>`** - Memory addresses and symbols
3. **`<AssemblerScript>`** - The actual cheat functionality
4. **`<ID>`** and `<Description>`** - Needed for identification

### Step 2: Manual Conversion Process

#### 2.1 Clean the File
1. **Open your .CT file** in a text editor
2. **Delete the entire `<Forms>` section** (from `<Forms>` to `</Forms>`)
3. **Remove all `<Hotkeys>` sections** throughout the file
4. **Remove `<Options>` lines** that contain UI directives

#### 2.2 Extract Cheat IDs
Look through your `<CheatEntries>` and note down all the `<ID>` values:
```xml
<CheatEntry>
  <ID>10070</ID>  <!-- Note this number -->
  <Description>"Unlimited Health"</Description>
  <!-- Keep the AssemblerScript section -->
</CheatEntry>
```

#### 2.3 Replace the LuaScript Section
Find the `<LuaScript>` section and replace its entire contents with:

```lua
<LuaScript>-- Auto-enable all cheats trainer
-- Generated auto-trainer

getAutoAttachList().add("GAME_PROCESS_NAME.exe")
hideAllCEWindows()

RequiredCEVersion=7.4
if (getCEVersion==nil) or (getCEVersion()&lt;RequiredCEVersion) then
  messageDialog('Please install Cheat Engine '..RequiredCEVersion, mtError, mbOK)
  closeCE()
end

function onAttach()
  local addresslist = getAddressList()
  sleep(3000)
  
  -- Auto-enable all cheats
  local cheats = {
    -- ADD YOUR CHEAT IDs HERE
    10070,
    10071,
    10072
  }
  
  for i, cheatId in ipairs(cheats) do
    local memrec = addresslist.getMemoryRecordByID(cheatId)
    if memrec then
      memrec.Active = true
      sleep(100)
    end
  end
end

local al = getAutoAttachList()
al.OnAttach = onAttach

local form = createForm()
form.Width = 0
form.Height = 0
form.WindowState = wsMinimized
form.ShowInTaskbar = false
form.Visible = false
form.FormStyle = fsStayOnTop

createTimer(nil, function()
  if not isProcessActive("GAME_PROCESS_NAME.exe") then
    closeCE()
  end
end, 5000)
</LuaScript>
```

#### 2.4 Customize for Your Game
1. **Replace `GAME_PROCESS_NAME.exe`** with your game's executable
2. **Update the cheats array** with your collected cheat IDs
3. **Save as .CETRAINER** extension

---

## Part 2: NodeJS Automation Script

Here's a complete NodeJS script that automatically converts any CT file to an auto-trainer:

### Installation & Setup

```bash
npm init -y
npm install xml2js
```

### The Converter Script

Save this as `ct-to-trainer.js`:

```javascript
const fs = require('fs');
const xml2js = require('xml2js');
const path = require('path');

class CTToTrainerConverter {
  constructor() {
    this.parser = new xml2js.Parser();
    this.builder = new xml2js.Builder({
      xmldec: { version: '1.0', encoding: 'utf-8' }
    });
  }

  async convertFile(inputPath, outputPath, gameProcessName) {
    try {
      const xmlContent = fs.readFileSync(inputPath, 'utf-8');
      const result = await this.parser.parseStringPromise(xmlContent);
      
      const cleanedResult = this.cleanCheatTable(result);
      const cheatIds = this.extractCheatIds(cleanedResult);
      cleanedResult.CheatTable.LuaScript = [this.generateLuaScript(gameProcessName, cheatIds)];
      
      const xml = this.builder.buildObject(cleanedResult);
      fs.writeFileSync(outputPath, xml);
      
      console.log(`✅ Successfully converted ${inputPath} to ${outputPath}`);
      console.log(`📋 Found ${cheatIds.length} cheats: ${cheatIds.join(', ')}`);
      
    } catch (error) {
      console.error('❌ Error converting file:', error.message);
    }
  }

  cleanCheatTable(cheatTable) {
    // Remove Forms section
    if (cheatTable.CheatTable.Forms) {
      delete cheatTable.CheatTable.Forms;
    }

    // Remove hotkeys from all cheat entries
    if (cheatTable.CheatTable.CheatEntries) {
      this.removeHotkeysRecursive(cheatTable.CheatTable.CheatEntries);
    }

    return cheatTable;
  }

  removeHotkeysRecursive(entries) {
    if (!Array.isArray(entries)) return;

    entries.forEach(entryContainer => {
      if (entryContainer.CheatEntry) {
        entryContainer.CheatEntry.forEach(entry => {
          // Remove hotkeys
          if (entry.Hotkeys) {
            delete entry.Hotkeys;
          }
          
          // Remove UI options
          if (entry.Options) {
            delete entry.Options;
          }

          // Recursively process nested entries
          if (entry.CheatEntries) {
            this.removeHotkeysRecursive(entry.CheatEntries);
          }
        });
      }
    });
  }

  extractCheatIds(cheatTable) {
    const ids = [];
    
    if (cheatTable.CheatTable.CheatEntries) {
      this.extractIdsRecursive(cheatTable.CheatTable.CheatEntries, ids);
    }
    
    return ids;
  }

  extractIdsRecursive(entries, ids) {
    if (!Array.isArray(entries)) return;

    entries.forEach(entryContainer => {
      if (entryContainer.CheatEntry) {
        entryContainer.CheatEntry.forEach(entry => {
          if (entry.ID && entry.ID[0]) {
            ids.push(parseInt(entry.ID[0]));
          }

          // Recursively process nested entries
          if (entry.CheatEntries) {
            this.extractIdsRecursive(entry.CheatEntries, ids);
          }
        });
      }
    });
  }

  generateLuaScript(gameProcessName, cheatIds) {
    const cheatIdsString = cheatIds.map(id => `    ${id}`).join(',\n');
    
    return `-- Auto-enable all cheats trainer
-- Generated automatically from CT file

getAutoAttachList().add("${gameProcessName}")
hideAllCEWindows()

RequiredCEVersion=7.4
if (getCEVersion==nil) or (getCEVersion()<RequiredCEVersion) then
  messageDialog('Please install Cheat Engine '..RequiredCEVersion, mtError, mbOK)
  closeCE()
end

function onAttach()
  local addresslist = getAddressList()
  sleep(3000)
  
  -- Auto-enable all cheats
  local cheats = {
${cheatIdsString}
  }
  
  for i, cheatId in ipairs(cheats) do
    local memrec = addresslist.getMemoryRecordByID(cheatId)
    if memrec then
      memrec.Active = true
      sleep(100)
    end
  end
end

local al = getAutoAttachList()
al.OnAttach = onAttach

local form = createForm()
form.Width = 0
form.Height = 0
form.WindowState = wsMinimized
form.ShowInTaskbar = false
form.Visible = false
form.FormStyle = fsStayOnTop

createTimer(nil, function()
  if not isProcessActive("${gameProcessName}") then
    closeCE()
  end
end, 5000)`;
  }
}

// Command line usage
if (require.main === module) {
  const args = process.argv.slice(2);
  
  if (args.length < 3) {
    console.log('Usage: node ct-to-trainer.js <input.CT> <output.CETRAINER> <game.exe>');
    console.log('Example: node ct-to-trainer.js witcher3.CT witcher3_auto.CETRAINER witcher3.exe');
    process.exit(1);
  }

  const [inputPath, outputPath, gameProcess] = args;
  const converter = new CTToTrainerConverter();
  converter.convertFile(inputPath, outputPath, gameProcess);
}

module.exports = CTToTrainerConverter;
```

### Usage Examples

#### Command Line:
```bash
node ct-to-trainer.js input.CT output.CETRAINER witcher3.exe
```

#### Batch Processing Script:
Save as `batch-convert.js`:

```javascript
const CTToTrainerConverter = require('./ct-to-trainer.js');
const fs = require('fs');
const path = require('path');

const games = [
  { ct: 'witcher3.CT', trainer: 'witcher3_auto.CETRAINER', process: 'witcher3.exe' },
  { ct: 'eldenring.CT', trainer: 'eldenring_auto.CETRAINER', process: 'eldenring.exe' },
  // Add more games here
];

const converter = new CTToTrainerConverter();

async function convertAll() {
  for (const game of games) {
    if (fs.existsSync(game.ct)) {
      await converter.convertFile(game.ct, game.trainer, game.process);
    } else {
      console.log(`⚠️  File not found: ${game.ct}`);
    }
  }
}

convertAll();
```

---

## Part 3: Playnite Integration

### Pre-Game PowerShell Script:
```powershell
# Auto-start trainer before game
$trainerPath = "D:\Cheats\{GameName}_auto.CETRAINER"
$cheatEngine = "C:\Program Files\Cheat Engine 7.4\cheatengine-x86_64-SSE4-AVX2.exe"

# Kill any existing CE processes
Get-Process -Name "cheatengine-x86_64-SSE4-AVX2" -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 1

# Start trainer
Start-Process -FilePath $cheatEngine -ArgumentList "`"$trainerPath`"" -WindowStyle Hidden
Start-Sleep -Seconds 2
```

### Post-Game PowerShell Script:
```powershell
# Clean up after game
Get-Process -Name "cheatengine-x86_64-SSE4-AVX2" -ErrorAction SilentlyContinue | Stop-Process -Force
```

---

## Part 4: Advanced Features & Troubleshooting

### Adding Custom Delays
For games that need longer initialization:

```lua
function onAttach()
  local addresslist = getAddressList()
  
  -- Wait for game to be fully loaded
  sleep(5000)  -- Increase this for slower games
  
  -- Enable main script first
  local mainScript = addresslist.getMemoryRecordByID(10070)
  if mainScript then
    mainScript.Active = true
    sleep(1000)  -- Wait for main script to initialize
  end
  
  -- Then enable other cheats
  local cheats = {10071, 10072, 10073}
  for i, cheatId in ipairs(cheats) do
    local memrec = addresslist.getMemoryRecordByID(cheatId)
    if memrec then
      memrec.Active = true
      sleep(200)  -- Delay between each cheat
    end
  end
end
```

### Error Handling & Logging
```lua
function onAttach()
  local addresslist = getAddressList()
  sleep(3000)
  
  local cheats = {10070, 10071, 10072}
  local activated = 0
  
  for i, cheatId in ipairs(cheats) do
    local memrec = addresslist.getMemoryRecordByID(cheatId)
    if memrec then
      local success = pcall(function()
        memrec.Active = true
      end)
      if success then
        activated = activated + 1
      end
      sleep(100)
    end
  end
  
  -- Optional: Show success message (remove for completely silent operation)
  -- messageDialog('Activated '..activated..' cheats', mtInformation, mbOK)
end
```

## Quick Reference

### File Extensions:
- `.CT` = Cheat Engine Table (input)
- `.CETRAINER` = Cheat Engine Trainer (output)

### Key Sections to Remember:
- Remove: `<Forms>`, `<Hotkeys>`, `<Options>`
- Keep: `<CheatEntries>`, `<UserdefinedSymbols>`, `<AssemblerScript>`
- Replace: `<LuaScript>` with auto-activation code

### Common Game Processes:
- Witcher 3: `witcher3.exe`
- Elden Ring: `eldenring.exe`
- Cyberpunk 2077: `Cyberpunk2077.exe`
- Dark Souls 3: `DarkSoulsIII.exe`

This guide provides both manual and automated approaches to convert any CT file into a fully automated, invisible trainer that integrates seamlessly with Playnite!
