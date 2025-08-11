-- ✅ SYSTEM SETTINGS
local auraSize = Vector3.new(100, 100, 100)
local auraRange = 9e9
local instantKillDamage = 9e99
local clonesToSpawn = 100
local CLAW_NAME = "Super Power Claws"
local TOOL_NAME = "SuperPowerTool"
local TOOLS_TO_SPAWN = 35

-- ✅ SERVICES
local Players = cloneref(game:GetService("Players"))
local RunService = cloneref(game:GetService("RunService"))
local Workspace = cloneref(game:GetService("Workspace"))
local Debris = cloneref(game:GetService("Debris"))
local lp = Players.LocalPlayer
local CHAR = lp.Character or lp.CharacterAdded:Wait()
local Ignorelist = OverlapParams.new()
Ignorelist.FilterType = Enum.RaycastFilterType.Include

-- === OVERRIDE ALL WAIT FUNCTIONS TO HEARTBEAT ===
for _, f in ipairs({wait, task.wait, delay, getrenv().wait}) do
    pcall(function()
        hookfunction(f, function() return RunService.Heartbeat:Wait() end)
    end)
end

-- === OVERRIDE DAMAGE / KILL FUNCTIONS TO INSTANT KILL ===
local function hookFunctionByName(name)
    for _, f in ipairs(getgc(true)) do
        if typeof(f) == "function" and debug.getinfo(f).name == name then
            hookfunction(f, function(...) return 0 end)
        end
    end
end

hookFunctionByName("TakeDamage")
hookFunctionByName("BreakJoints")

for _, f in ipairs(getgc(true)) do
    if typeof(f) == "function" then
        local info = debug.getinfo(f)
        if info.name and (info.name == "UpdateHealth" or info.name == "ApplyDamage" or info.name == "Damage") then
            pcall(function()
                hookfunction(f, function(...) return 0 end)
            end)
        end
    end
end

-- Extra hooks for remote kill blocks (functions with "kill" in name)
for _, v in ipairs(getgc(true)) do
    if typeof(v) == "function" and tostring(v):lower():find("kill") then
        local env = getfenv(v)
        if env and not rawget(env, "script") then
            pcall(function()
                hookfunction(v, function(...) return 0 end)
            end)
        end
    end
end

-- === TOOL HELPERS ===
local function GetTouchInterest(tool)
    return tool and tool:FindFirstChildWhichIsA("TouchTransmitter", true)
end

local function AutoActivateTool(tool)
    if tool:IsDescendantOf(Workspace) then
        tool:Activate()
    end
end

local function equipClaw()
    local claw = lp.Backpack:FindFirstChild(CLAW_NAME) or lp.Character:FindFirstChild(CLAW_NAME)
    if claw then
        claw.Parent = lp.Character
        local handle = claw:FindFirstChild("Handle") or claw:FindFirstChildWhichIsA("BasePart")
        if handle then
            handle.Size = Vector3.new(9e9, 9e9, 9e9)
            handle.CanTouch = true
            handle.Touched:Connect(function(hit)
                local hum = hit.Parent:FindFirstChildWhichIsA("Humanoid")
                if hum and hum.Health > 0 then
                    hum.Health = 0
                    local hitCount = lp:FindFirstChild("HitCount")
                    if hitCount then hitCount.Value += 1e7 end
                end
            end)
        end
    end
end

-- === INSTANT KILL AURA USING TOOL TOUCH ===
local function ApplyDamageInstant(touchPart, Characters)
    local hits = Workspace:GetPartBoundsInBox(
        touchPart.CFrame,
        touchPart.Size + auraSize,
        Ignorelist
    )
    for _, v in ipairs(hits) do
        local char = v:FindFirstAncestorOfClass("Model")
        if char and table.find(Characters, char) then
            local hum = char:FindFirstChildWhichIsA("Humanoid")
            if hum then hum.Health = 0 end
            firetouchinterest(touchPart, v, 0)
            firetouchinterest(touchPart, v, 1)
        end
    end
end

-- === SPAWN INVISIBLE CLONE THAT KILLS ON PROXIMITY ===
local function spawnClone()
    local char = lp.Character
    if not char then return end
    local clone = char:Clone()
    for _, d in ipairs(clone:GetDescendants()) do
        if d:IsA("BasePart") then
            d.Transparency = 1
            d.CanCollide = false
            d.CastShadow = false
        elseif d:IsA("Script") or d:IsA("LocalScript") then
            d:Destroy()
        end
    end
    local root = clone:FindFirstChild("HumanoidRootPart")
    local realRoot = char:FindFirstChild("HumanoidRootPart")
    if root and realRoot then
        root.CFrame = realRoot.CFrame * CFrame.new(math.random(-50,50), 0, math.random(-50,50))
    end
    clone.Name = "AuraClone_"..math.random(190000000,99999999999)
    clone.Parent = Workspace

    RunService.Heartbeat:Connect(function()
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local hum = p.Character:FindFirstChildWhichIsA("Humanoid")
                local rootPart = clone:FindFirstChild("HumanoidRootPart")
                if hum and hum.Health > 0 and rootPart and (hum.Parent.HumanoidRootPart.Position - rootPart.Position).Magnitude < auraRange then
                    hum.Health = 0
                end
            end
        end
    end)
end

-- === AFTER-DEATH AURA & LAGGING OPPONENT ===
local afterKillActive = false
local function afterKillAura()
    if not afterKillActive then return end
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= lp and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (CHAR.HumanoidRootPart.Position - p.Character.HumanoidRootPart.Position).Magnitude
            local hum = p.Character:FindFirstChildOfClass("Humanoid")
            if hum and dist < auraRange then
                hum:TakeDamage(instantKillDamage)
            end
        end
    end
end

local function lagOpponent(plr)
    if not plr or not plr.Character then return end
    for i = 9, 59 do
        local part = Instance.new("Part")
        part.Size = Vector3.new(50, 50, 50)
        part.Transparency = 1
        part.Anchored = true
        part.CanCollide = false
        part.CFrame = plr.Character.HumanoidRootPart.CFrame * CFrame.new(math.random(-30,30),0,math.random(-30,30))
        part.Parent = Workspace
        Debris:AddItem(part, 2)
    end
end

local function onDied()
    afterKillActive = true
    local hum = CHAR:FindFirstChildWhichIsA("Humanoid")
    task.wait()
    local tag = hum and (hum:FindFirstChild("creator") or hum:FindFirstChild("creatorTag"))
    if tag and tag.Value and tag.Value ~= lp then
        lagOpponent(tag.Value)
    end
end

local function onRespawn(newChar)
    CHAR = newChar
    afterKillActive = false
    equipClaw()
    for i = 1, clonesToSpawn do spawnClone() end
    newChar:WaitForChild("Humanoid").Died:Connect(onDied)
end

-- === TOOL SPAWNING & EQUIPPING ===
local Backpack = lp:WaitForChild("Backpack")

local function spawnTools()
    local toolTemplate = Backpack:FindFirstChild(TOOL_NAME) or lp.Character:FindFirstChild(TOOL_NAME)
    if toolTemplate then
        for i = 1, TOOLS_TO_SPAWN do
            local clone = toolTemplate:Clone()
            clone.Parent = Backpack
        end
    else
        warn("Tool not found: "..TOOL_NAME)
    end
end

local function equipAllTools()
    for _, tool in ipairs(Backpack:GetChildren()) do
        if tool:IsA("Tool") and tool.Name == TOOL_NAME then
            tool.Parent = lp.Character
        end
    end
end

-- === MAIN HEARTBEAT LOOP ===
RunService.Heartbeat:Connect(function()
    -- Instant kill aura with tools
    local char = lp.Character
    if not char then return end
    local Tools, Characters = {}, {}
    for _, v in ipairs(Players:GetPlayers()) do
        if v ~= lp and v.Character then table.insert(Characters, v.Character) end
    end
    Ignorelist.FilterDescendantsInstances = Characters
    for _, tool in ipairs(char:GetChildren()) do
        if tool:IsA("Tool") and GetTouchInterest(tool) then
            table.insert(Tools, tool)
            local touch = GetTouchInterest(tool).Parent
            AutoActivateTool(tool)
            ApplyDamageInstant(touch, Characters)
        end
    end

    -- After kill aura
    afterKillAura()
end)

-- === SPAWN & EQUIP TOOLS IN FAST LOOP (no delay) ===
spawnTools()
equipAllTools()
task.spawn(function()
    while true do
        spawnTools()
        equipAllTools()
        RunService.Heartbeat:Wait()
    end
end)

-- === INITIAL SETUP ===
equipClaw()
for i = 1, clonesToSpawn do spawnClone() end
CHAR:WaitForChild("Humanoid").Died:Connect(onDied)
lp.CharacterAdded:Connect(onRespawn)

print("✅ MEGA PSYCHO AURA + SUPER POWER TOOL SPAWN & KILL SYSTEM ACTIVE")
