# Echo: The Sound of Silence — Full Roblox System Blueprint

This document contains the complete layout, gameplay breakdown, and functional Luau scripts for your Echo Location Roblox game. You can copy and paste this directly into your GitHub repository `README.md` or split the code blocks into your project files.

---

## 🎮 Game Overview & Lore

* **Concept:** A multiplayer psychological horror/stealth game where the world is pitch black.
* **The Gimmick:** Players cannot see anything unless they make noise (walking, jumping, clapping, or talking into proximity voice chat). Sound sends out a visual radar ripple that temporarily outlines the environment in glowing lines.
* **The Threat:** A blind AI monster hunts the players. It cannot see light, but it has perfect hearing. Every time a player makes a sound to "see," they risk drawing the creature straight to them.

---

## 🗺️ Map Layout & Environmental Design

### Zone 1: The Maintenance Tunnels (The Spawn)
* **Aesthetic:** Tight, claustrophobic corridors with exposed metal piping.
* **Gameplay:** Introduces players to the echo mechanic. Sound bounces cleanly off walls, but narrow corridors leave very few places to hide if the monster enters the same hallway.
* **Hazard:** Burst Steam Pipes — Randomly hiss loudly. They light up the area completely but draw the monster's attention immediately.

### Zone 2: The Flooded Generator Rooms (The Fluid Hazard)
* **Aesthetic:** Massive industrial halls filled with deactivated machinery and 6 inches of standing water.
* **Gameplay:** Walking on the ground creates unavoidable splashing noises. Players must parkour across raised metal catwalks and floating debris to remain silent.

### Zone 3: The Echo Chambers (The High-Risk Vaults)
* **Aesthetic:** High-ceiling concrete storage vaults.
* **Gameplay:** Sounds made here trigger a Natural Echo Delay (the ripple bounces twice). It provides excellent visibility of the giant room but keeps the sound echoing for a long time, making it highly dangerous.

### Zone 4: The Blast Door Exit (The Escape Finale)
* **Aesthetic:** A long, straight concrete corridor terminating at a massive reinforced steel vault door.
* **Gameplay:** To escape, players must activate a terminal. The vault door takes 30 seconds to grind open, making deafening mechanical noises. Players must hide perfectly still in the dark while the monster aggressively searches the corridor.

---

## 🔧 Complete Roblox System Code

### 1. Server Script (Noise Framework & Jump Detection)
**Location:** `ServerScriptService -> Script` (Name: `EchoServerManager`)

```lua
-- ServerScriptService -> EchoServerManager
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

-- Create the communication bridge if it doesn't exist
local EchoEvent = ReplicatedStorage:FindFirstChild("CreateEcho")
if not EchoEvent then
    EchoEvent = Instance.new("RemoteEvent")
    EchoEvent.Name = "CreateEcho"
    EchoEvent.Parent = ReplicatedStorage
end

-- Global function to register a noise event
_G.TriggerNoise = function(position, volume, isContinuous)
    -- Tell all clients to render the visual ripple
    EchoEvent:FireAllClients(position, volume)
    
    -- Fire a global event for the Monster AI to listen to
    local monsterSignal = ReplicatedStorage:FindFirstChild("MonsterHearingSignal")
    if monsterSignal then
        monsterSignal:Fire(position, volume, isContinuous)
    end
end

-- Detect when players make noise via movement (Jumping/Landing)
Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function(character)
        local humanoid = character:WaitForChild("Humanoid")
        local rootPart = character:WaitForChild("HumanoidRootPart")
        
        humanoid.StateChanged:Connect(function(oldState, newState)
            if newState == Enum.HumanoidStateType.Landed then
                -- Player hit the ground; trigger a medium-sized noise ripple
                _G.TriggerNoise(rootPart.Position, 25, false)
            elseif newState == Enum.HumanoidStateType.Jumping then
                -- Player jumped; trigger a smaller noise ripple
                _G.TriggerNoise(rootPart.Position, 10, false)
            end
        end)
    end)
end)
```

### 2. Client LocalScript (Raycast Echo Visualizer)
**Location:** `StarterPlayer -> StarterPlayerScripts -> LocalScript` (Name: `EchoClientRenderer`)

```lua
-- StarterPlayerScripts -> EchoClientRenderer
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")

local EchoEvent = ReplicatedStorage:WaitForChild("CreateEcho")

-- Configuration for visual styling
local RAY_COUNT = 72 -- Number of beams sent out in a 360-degree circle
local RIPPLE_COLOR = Color3.fromRGB(0, 255, 255) -- Cyan/Neon Blue radar line
local FADE_TIME = 1.2 -- How long the visible wall markers last

-- Function to create a glowing marker at a contact point
local function spawnVisualMarker(position, normal)
    local part = Instance.new("Part")
    part.Shape = Enum.PartType.Ball
    part.Size = Vector3.new(0.5, 0.5, 0.5)
    part.Position = position
    part.Color = RIPPLE_COLOR
    part.Material = Enum.Material.Neon
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.Parent = Workspace:WaitForChild("Terrain") -- Keeps hierarchy clean

    -- Create fade away tween animation
    local tweenInfo = TweenInfo.new(FADE_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    local tweenProperties = {
        Size = Vector3.new(0.1, 0.1, 0.1),
        Transparency = 1
    }
    
    local tween = TweenService:Create(part, tweenInfo, tweenProperties)
    tween:Play()
    
    -- Clean up memory
    tween.Completed:Connect(function()
        part:Destroy()
    end)
end

-- Process the incoming echo request
EchoEvent.OnClientEvent:Connect(function(originPosition, volume)
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    
    -- Exclude characters from blocking the radar lines
    local filterList = {}
    for _, player in ipairs(game.Players:GetPlayers()) do
        if player.Character then table.insert(filterList, player.Character) end
    end
    raycastParams.FilterDescendantsInstances = filterList

    -- Math for 360-degree horizontal raycasting ring
    for i = 1, RAY_COUNT do
        local angle = math.rad((i / RAY_COUNT) * 360)
        local direction = Vector3.new(math.cos(angle), 0, math.sin(angle)) * volume
        
        local raycastResult = Workspace:Raycast(originPosition, direction, raycastParams)
        
        if raycastResult then
            spawnVisualMarker(raycastResult.Position, raycastResult.Normal)
        end
    end
end)
```

### 3. Sonar Clicker Tool Script (Player Interaction Item)
**Location:** Create a `Tool` in `StarterPack` named `SonarClicker`. Inside the Tool, create a `Script` named `ClickerHandler`.

```lua
-- Tool (StarterPack.SonarClicker) -> ClickerHandler
local Tool = script.Parent
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Cooldown = false
local CLICK_COOLDOWN = 0.8
local SONAR_VOLUME = 35 -- Distance the click echo travels

Tool.Activated:Connect(function()
    if Cooldown then return end
    Cooldown = true
    
    local character = Tool.Parent
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    
    if rootPart then
        -- Play local audio cue if you have an ID
        local sound = Tool:FindFirstChildOfClass("Sound")
        if sound then sound:Play() end
        
        -- Call global server noise generator
        if _G.TriggerNoise then
            _G.TriggerNoise(rootPart.Position, SONAR_VOLUME, false)
        end
    end
    
    task.wait(CLICK_COOLDOWN)
    Cooldown = false
end)
```

---

## 🧩 GitHub Repository Setup Instructions
1. Initialize a new repo on GitHub or your system (`git init`).
2. Create a file named `README.md`.
3. Copy the entire contents of this text block and paste it inside.
4. Push to your repository to preserve your project design outline and foundation scripts.

---

## 💡 Notes for Expansion

This blueprint is a strong starting point for a playable prototype. You can extend it with:

- a monster AI that chases sound sources
- stealth/hiding systems
- objective markers and escape logic
- ambient noise and player voice-based echolocation
- dynamic lighting toggles for darkness and player visibility
- zone-specific sound profiles and hazard mechanics

If you want, I can also turn this into a more production-ready Roblox project structure with separate scripts for:
- monster AI
- noise system
- escape terminal
- player movement stealth tuning
- lobby and round management
