--[[
    ========================================================================
                           TED HUB - BLOX FRUITS AUTO FARM
    ========================================================================
    100% Native Roblox GUI. No external libraries. Works on all executors.
    ========================================================================
]]

-- 1. AGUARDA O JOGO CARREGAR TOTALMENTE
if not game:IsLoaded() then
    game.Loaded:Wait()
end

local Players = game:GetService("Players")
local plr     = Players.LocalPlayer

-- 2. AGUARDA O PERSONAGEM ESTAR DISPONÍVEL NA MEMÓRIA
while not plr do
    task.wait(0.1)
    plr = Players.LocalPlayer
end

-- 3. AGUARDA OS DADOS DE NÍVEL E PONTOS DO PERSONAGEM
local data = plr:WaitForChild("Data", 30)
local level = data and data:WaitForChild("Level", 30)
local points = data and data:WaitForChild("Points", 30)

if not data or not level or not points then
    warn("[Ted Hub] Falha Crítica: Dados de nível do jogador não encontrados.")
    return
end

local TweenService    = game:GetService("TweenService")
local RunService      = game:GetService("RunService")
local VirtualUser     = game:GetService("VirtualUser")
local VirtualInput    = game:GetService("VirtualInputManager")
local UserInputService= game:GetService("UserInputService")
local replicated      = game:GetService("ReplicatedStorage")
local remote          = nil

-- Carrega a conexão com o servidor de forma assíncrona
task.spawn(function()
    local remotes = replicated:WaitForChild("Remotes", 20)
    if remotes then
        remote = remotes:WaitForChild("CommF_", 20)
    end
end)

-- ════════════════════════════════════════════════════════════
--   BYPASS DE LAG / ERRO DMGDEBUG (CORREÇÃO DO CONSOLE RED)
-- ════════════════════════════════════════════════════════════
task.spawn(function()
    local remotes = replicated:WaitForChild("Remotes", 10)
    if remotes then
        local dmgDebug = remotes:WaitForChild("DMGDEBUG", 10)
        if dmgDebug and dmgDebug:IsA("RemoteEvent") then
            pcall(function()
                dmgDebug.OnClientEvent:Connect(function() end)
            end)
        end
    end
end)

_G.AutoFarm      = false
_G.AutoHaki      = true      -- Haki ativado por padrão
_G.AutoStats     = false
_G.LowCpu        = false
_G.SelectedWeapon= "Melee"   -- "Melee" | "Sword" | "Blox Fruit" | "Gun"
_G.StatTarget    = "Melee"   -- "Melee" | "Defense" | "Sword" | "Gun" | "Demon Fruit"
_G.TweenSpeed    = 250       -- Velocidade de Voo configurada para 250
_G.BringMobs     = true

local placeId = game.PlaceId
local W1 = (placeId == 2753915549 or placeId == 85211729168715)
local W2 = (placeId == 4442272183 or placeId == 79091703265657)
local W3 = (placeId == 7449423635 or placeId == 100117331123089)

-- Detecção de mar inteligente caso a ID do lugar falhe
if not W1 and not W2 and not W3 then
    local map = workspace:FindFirstChild("Map")
    if map then
        if map:FindFirstChild("Port Town") or map:FindFirstChild("Hydra Island") then
            W3 = true
        elseif map:FindFirstChild("Kingdom of Rose") or map:FindFirstChild("Green Zone") then
            W2 = true
        else
            W1 = true
        end
    else
        local lv = level.Value
        if lv < 700 then
            W1 = true
        elseif lv < 1500 then
            W2 = true
        else
            W3 = true
        end
    end
end

local SeaName = W1 and "First Sea" or W2 and "Second Sea" or W3 and "Third Sea" or "Unknown"

local InfiniteJump = false
local customSpeed = 16
local customJump = 50
local useSpeed = false
local useJump = false

-- Anti-Idle
task.spawn(function()
    plr.Idled:Connect(function()
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new(0, 0))
    end)
end)

-- Escolha automática de time
task.spawn(function()
    task.wait(2)
    pcall(function() 
        if remote then
            remote:InvokeServer("SetTeam", "Pirates") 
        end
    end)
end)

-- Pulo Infinito
UserInputService.JumpRequest:Connect(function()
    if InfiniteJump then
        local char = plr.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum:ChangeState("Jumping")
        end
    end
end)

-- Loop de velocidade e pulo customizados
task.spawn(function()
    while task.wait(0.2) do
        pcall(function()
            local char = plr.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum then
                if useSpeed then
                    hum.WalkSpeed = customSpeed
                end
                if useJump then
                    hum.JumpPower = customJump
                end
            end
        end)
    end
end)

-- ════════════════════════════════════════════════════════════
--   PHYSICAL MOVER-BASED FLIGHT ENGINE
-- ════════════════════════════════════════════════════════════
local shouldTween = false

local function CleanPhysics()
    local char = plr.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if hrp then
        local bv = hrp:FindFirstChild("TedBV")
        if bv then bv:Destroy() end
        local bg = hrp:FindFirstChild("TedBG")
        if bg then bg:Destroy() end
        for _, v in pairs(char:GetDescendants()) do
            if v:IsA("BasePart") then
                v.CanCollide = true
            end
        end
    end
end

local function HoldAt(cf)
    local char = plr.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local bv = hrp:FindFirstChild("TedBV") or Instance.new("BodyVelocity")
    bv.Name = "TedBV"
    bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    bv.Velocity = Vector3.new(0, 0, 0)
    bv.Parent = hrp

    local bg = hrp:FindFirstChild("TedBG") or Instance.new("BodyGyro")
    bg.Name = "TedBG"
    bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bg.CFrame = cf
    bg.Parent = hrp

    for _, v in pairs(char:GetDescendants()) do
        if v:IsA("BasePart") then
            v.CanCollide = false
        end
    end
end

local function flyTo(targetCFrame, speed)
    local char = plr.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    shouldTween = true

    local bv = hrp:FindFirstChild("TedBV") or Instance.new("BodyVelocity")
    bv.Name = "TedBV"
    bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    bv.Parent = hrp

    local bg = hrp:FindFirstChild("TedBG") or Instance.new("BodyGyro")
    bg.Name = "TedBG"
    bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bg.Parent = hrp

    local targetPos = targetCFrame.Position
    local startPos = hrp.Position
    local distance = (startPos - targetPos).Magnitude

    local skyY = math.max(startPos.Y, targetPos.Y) + 200
    local points = {}
    if distance > 1500 then
        table.insert(points, Vector3.new(startPos.X, skyY, startPos.Z))
        table.insert(points, Vector3.new(targetPos.X, skyY, targetPos.Z))
    end
    table.insert(points, targetPos)

    for _, currentTarget in ipairs(points) do
        while shouldTween do
            local pos = hrp.Position
            local direction = (currentTarget - pos)
            local d = direction.Magnitude
            if d < 5 then break end

            bv.Velocity = direction.Unit * speed
            bg.CFrame = CFrame.new(pos, currentTarget)

            for _, v in pairs(char:GetDescendants()) do
                if v:IsA("BasePart") then
                    v.CanCollide = false
                end
            end
            task.wait()
        end
    end

    bv.Velocity = Vector3.new(0, 0, 0)
    shouldTween = false
end

local function Teleport(cf)
    local char = plr.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    shouldTween = false
    task.wait(0.1)
    
    local dist = (hrp.Position - cf.Position).Magnitude
    if dist > 350 then
        task.spawn(function()
            flyTo(cf, _G.TweenSpeed or 250)
        end)
    else
        hrp.CFrame = cf
    end
end

-- ════════════════════════════════════════════════════════════
--   WEAPON EQUIP
-- ════════════════════════════════════════════════════════════
local function EquipWeapon()
    local char = plr.Character
    if not char then return end
    local equipped = char:FindFirstChildOfClass("Tool")
    if equipped then
        local tip = equipped.ToolTip or ""
        if string.find(string.lower(tip), string.lower(_G.SelectedWeapon)) then
            return
        end
    end
    local bp = plr.Backpack
    for _, tool in pairs(bp:GetChildren()) do
        if tool:IsA("Tool") then
            local tip = tool.ToolTip or ""
            if string.find(string.lower(tip), string.lower(_G.SelectedWeapon)) then
                char.Humanoid:EquipTool(tool)
                return
            end
        end
    end
    local fallbackTool = bp:FindFirstChildOfClass("Tool")
    if fallbackTool then
        char.Humanoid:EquipTool(fallbackTool)
    end
end

local function HasQuest()
    local main = plr.PlayerGui:FindFirstChild("Main")
    if main then
        local q = main:FindFirstChild("Quest")
        return q and q.Visible
    end
    return false
end

-- ════════════════════════════════════════════════════════════
--   ARMAMENT HAKI ACTIVATOR (COM DEBOUNCE ANTI-LAG E ANTI-BUG)
-- ════════════════════════════════════════════════════════════
local lastHakiTime = 0
local function ActivateHaki()
    local char = plr.Character
    if char and remote then
        if not char:FindFirstChild("HasBuso") then
            local now = tick()
            if now - lastHakiTime > 4 then
                lastHakiTime = now
                pcall(function()
                    remote:InvokeServer("Buso")
                end)
            end
        end
    end
end

local function GetActiveQuestMob()
    local main = plr.PlayerGui:FindFirstChild("Main")
    if main then
        local q = main:FindFirstChild("Quest")
        if q and q.Visible then
            for _, v in pairs(q:GetDescendants()) do
                if v:IsA("TextLabel") and v.Text ~= "" then
                    local textLower = string.lower(v.Text)
                    for _, entry in ipairs(QuestDB) do
                        local mName = entry[5]
                        if string.find(textLower, string.lower(mName)) then
                            return entry
                        end
                    end
                end
            end
        end
    end
    return nil
end

local function FindMob(name)
    local Enemies = workspace:FindFirstChild("Enemies")
    if Enemies then
        for _, v in pairs(Enemies:GetChildren()) do
            if string.find(string.lower(v.Name), string.lower(name))
                and v:FindFirstChild("Humanoid")
                and v.Humanoid.Health > 0
                and v:FindFirstChild("HumanoidRootPart") then
                return v
            end
        end
    end
    for _, v in pairs(workspace:GetChildren()) do
        if string.find(string.lower(v.Name), string.lower(name))
            and v:FindFirstChild("Humanoid")
            and v.Humanoid.Health > 0
            and v:FindFirstChild("HumanoidRootPart") then
            return v
        end
    end
    return nil
end

local function ClumpMobs(name, targetPos)
    if not _G.BringMobs then return end
    local char = plr.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local Enemies = workspace:FindFirstChild("Enemies")
    if not Enemies then return end

    local stackPos = hrp.Position + Vector3.new(0, -4, 0)

    for _, v in pairs(Enemies:GetChildren()) do
        if string.find(string.lower(v.Name), string.lower(name))
            and v:FindFirstChild("Humanoid")
            and v.Humanoid.Health > 0
            and v:FindFirstChild("HumanoidRootPart") then
            local mobHrp = v:FindFirstChild("HumanoidRootPart")
            if mobHrp and (mobHrp.Position - targetPos).Magnitude < 3000 then
                mobHrp.CanCollide = false
                mobHrp.CFrame     = CFrame.new(stackPos)
                v.Humanoid.WalkSpeed   = 0
                v.Humanoid.JumpPower   = 0
                if v.Humanoid:FindFirstChild("Animator") then
                    pcall(function() v.Humanoid.Animator:Destroy() end)
                end
                plr.SimulationRadius = math.huge
            end
        end
    end
end

local QuestDB = {}

if W1 then
    QuestDB = {
        {0,9,"BanditQuest1",1,"Bandit","Starter Island",CFrame.new(1045.96,27.00,1560.82),CFrame.new(1045.96,27.00,1560.82)},
        {10,14,"JungleQuest",1,"Monkey","Jungle Island",CFrame.new(-1598.09,35.55,153.38),CFrame.new(-1448.52,67.85,11.47)},
        {15,29,"JungleQuest",2,"Gorilla","Jungle Island",CFrame.new(-1598.09,35.55,153.38),CFrame.new(-1129.88,40.46,-525.42)},
        {30,39,"BuggyQuest1",1,"Pirate","Pirate Village",CFrame.new(-1141.07,4.10,3831.55),CFrame.new(-1103.51,13.75,3896.09)},
        {40,59,"BuggyQuest1",2,"Brute","Pirate Village",CFrame.new(-1141.07,4.10,3831.55),CFrame.new(-1140.08,14.81,4322.92)},
        {60,74,"DesertQuest",1,"Desert Bandit","Desert Island",CFrame.new(894.49,5.14,4392.43),CFrame.new(924.80,6.45,4481.59)},
        {75,89,"DesertQuest",2,"Desert Officer","Desert Island",CFrame.new(894.49,5.14,4392.43),CFrame.new(1608.28,8.61,4371.01)},
        {90,99,"SnowQuest",1,"Snow Bandit","Frozen Village",CFrame.new(1389.74,88.15,-1298.91),CFrame.new(1354.35,87.27,-1393.95)},
        {100,119,"SnowQuest",2,"Snowman","Frozen Village",CFrame.new(1389.74,88.15,-1298.91),CFrame.new(6241.99,51.52,-1243.98)},
        {120,149,"MarineQuest2",1,"Chief Petty Officer","Marine Fortress",CFrame.new(-5039.59,27.35,4324.68),CFrame.new(-4881.23,22.65,4273.75)},
        {150,174,"SkyQuest",1,"Sky Bandit","Skypiea",CFrame.new(-4839.53,716.37,-2619.44),CFrame.new(-4953.21,295.74,-2899.23)},
        {175,189,"SkyQuest",2,"Dark Master","Skypiea",CFrame.new(-4839.53,716.37,-2619.44),CFrame.new(-5259.84,391.40,-2229.04)},
        {190,209,"PrisonerQuest",1,"Prisoner","Prison",CFrame.new(5308.93,1.66,475.12),CFrame.new(5098.97,-0.32,474.24)},
        {210,249,"PrisonerQuest",2,"Dangerous Prisoner","Prison",CFrame.new(5308.93,1.66,475.12),CFrame.new(5654.56,15.63,866.30)},
        {250,274,"ColosseumQuest",1,"Toga Warrior","Colosseum",CFrame.new(-1580.05,6.35,-2986.48),CFrame.new(-1820.21,51.68,-2740.67)},
        {275,299,"ColosseumQuest",2,"Gladiator","Colosseum",CFrame.new(-1580.05,6.35,-2986.48),CFrame.new(-1292.84,56.38,-3339.03)},
        {300,324,"MagmaQuest",1,"Military Soldier","Magma Village",CFrame.new(-5313.37,10.95,8515.29),CFrame.new(-5411.16,11.08,8454.29)},
        {325,374,"MagmaQuest",2,"Military Spy","Magma Village",CFrame.new(-5313.37,10.95,8515.29),CFrame.new(-5802.87,86.26,8828.86)},
        {375,399,"FishmanQuest",1,"Fishman Warrior","Underwater City",CFrame.new(61122.65,18.50,1569.40),CFrame.new(60878.30,18.48,1543.76)},
        {400,449,"FishmanQuest",2,"Fishman Commando","Underwater City",CFrame.new(61122.65,18.50,1569.40),CFrame.new(61922.63,18.48,1493.93)},
        {450,474,"SkyExp1Quest",1,"God's Guard","Upper Skypiea",CFrame.new(-4721.89,843.87,-1949.97),CFrame.new(-4710.04,845.28,-1927.31)},
        {475,524,"SkyExp1Quest",2,"Shanda","Upper Skypiea",CFrame.new(-7859.10,5544.19,-381.48),CFrame.new(-7678.49,5566.40,-497.22)},
        {525,549,"SkyExp2Quest",1,"Royal Squad","Enel's Palace",CFrame.new(-7906.82,5634.66,-1411.99),CFrame.new(-7624.25,5658.13,-1467.35)},
        {550,624,"SkyExp2Quest",2,"Royal Soldier","Enel's Palace",CFrame.new(-7906.82,5634.66,-1411.99),CFrame.new(-7836.75,5645.66,-1790.62)},
        {625,649,"FountainQuest",1,"Galley Pirate","Fountain City",CFrame.new(5259.82,37.35,4050.03),CFrame.new(5551.02,78.90,3930.41)},
        {650,699,"FountainQuest",2,"Galley Captain","Fountain City",CFrame.new(5259.82,37.35,4050.03),CFrame.new(5441.95,42.50,4950.09)},
    }
elseif W2 then
    QuestDB = {
        {700,724,"Area1Quest",1,"Raider","Kingdom of Rose",CFrame.new(-429.54,71.77,1836.18),CFrame.new(-728.33,52.78,2345.77)},
        {725,774,"Area1Quest",2,"Mercenary","Kingdom of Rose",CFrame.new(-429.54,71.77,1836.18),CFrame.new(-1004.32,80.16,1424.62)},
        {775,799,"Area2Quest",1,"Swan Pirate","Dark Arena",CFrame.new(638.44,71.77,918.28),CFrame.new(1068.66,137.61,1322.11)},
        {800,874,"Area2Quest",2,"Factory Staff","Magma Village (Sea 2)",CFrame.new(632.70,73.11,918.67),CFrame.new(73.08,81.86,-27.47)},
        {875,899,"MarineQuest3",1,"Marine Lieutenant","Green Zone",CFrame.new(-2440.80,71.71,-3216.07),CFrame.new(-2821.37,75.90,-3070.09)},
        {900,949,"MarineQuest3",2,"Marine Captain","Green Zone",CFrame.new(-2440.80,71.71,-3216.07),CFrame.new(-1861.23,80.18,-3254.70)},
        {950,974,"ZombieQuest",1,"Zombie","Thriller Bark",CFrame.new(-5497.06,47.59,-795.24),CFrame.new(-5657.78,78.97,-928.69)},
        {975,999,"ZombieQuest",2,"Vampire","Thriller Bark",CFrame.new(-5497.06,47.59,-795.24),CFrame.new(-6037.67,32.18,-1340.66)},
        {1000,1049,"SnowMountainQuest",1,"Snow Trooper","Snow Mountain",CFrame.new(609.86,400.12,-5372.26),CFrame.new(549.15,427.39,-5563.70)},
        {1050,1099,"SnowMountainQuest",2,"Winter Warrior","Snow Mountain",CFrame.new(609.86,400.12,-5372.26),CFrame.new(1142.75,475.64,-5199.42)},
        {1100,1124,"IceSideQuest",1,"Lab Subordinate","Ice Castle",CFrame.new(-5429.05,15.98,-5297.96),CFrame.new(-5275.20,20.76,-5260.67)},
        {1125,1174,"IceSideQuest",2,"Government Spy","Ice Castle",CFrame.new(-5429.05,15.98,-5297.96),CFrame.new(-5600,22,-5500)},
        {1175,1224,"ForgottenQuest",1,"Pirate Millionaire","Forgotten Island",CFrame.new(-3053.98,237.19,-10145.04),CFrame.new(-712.83,98.58,5711.95)},
        {1225,1274,"FrostQuest",1,"Dragon Crew Warrior","Ice Admiral Island",CFrame.new(5668.98,28.52,-6483.35),CFrame.new(7021.50,55.76,-730.13)},
        {1275,1574,"FrostQuest",2,"Dragon Crew Archer","Ice Admiral Island",CFrame.new(5668.98,28.52,-6483.35),CFrame.new(6625,378,244)},
    }
elseif W3 then
    QuestDB = {
        {1500,1524,"PiratePortQuest",1,"Pirate Millionaire","Port Town",CFrame.new(-289.77,43.82,5579.94),CFrame.new(-871,91,5556)},
        {1525,1574,"PiratePortQuest",2,"Pistol Billionaire","Port Town",CFrame.new(-289.77,43.82,5579.94),CFrame.new(-1027.65,92.40,6578.85)},
        {1575,1599,"FemaleQuest",1,"Dragon Crew Warrior","Hydra Island",CFrame.new(-13232.68,332.40,-7626.01),CFrame.new(-13446,413,-7760)},
        {1600,1699,"FemaleQuest",2,"Dragon Crew Archer","Hydra Island",CFrame.new(-13232.68,332.40,-7626.01),CFrame.new(-11778,426,-10592)},
        {1700,1724,"MarineTreeQuest",1,"Marine Commodore","Great Tree",CFrame.new(2179.30,28.73,-6739.97),CFrame.new(2401,123,-7589)},
        {1725,1774,"MarineTreeQuest",2,"Marine Rear Admiral","Great Tree",CFrame.new(2179.30,28.73,-6739.97),CFrame.new(3588,229,-7085)},
        {1775,1799,"FloatingTurtleQuest1",1,"Fishman Raider","Floating Turtle",CFrame.new(-10452.92,332.18,-8511.45),CFrame.new(-10941,332,-8760)},
        {1800,1824,"FloatingTurtleQuest1",2,"Fishman Captain","Floating Turtle",CFrame.new(-10452.92,332.18,-8511.45),CFrame.new(-11035,332,-9087)},
        {1825,1849,"FloatingTurtleQuest2",1,"Forest Pirate","Floating Turtle",CFrame.new(-9324,114,-10214),CFrame.new(-10450,114,-10170)},
        {1850,1899,"FloatingTurtleQuest2",2,"Mythical Pirate","Floating Turtle",CFrame.new(-9324,114,-10214),CFrame.new(-9255,114,-9435)},
        {1900,1924,"JunglePirateQuest",1,"Jungle Pirate","Floating Turtle",CFrame.new(-9324,114,-10214),CFrame.new(-11778,426,-10592)},
        {1925,1974,"MusketeerQuest",1,"Musketeer Pirate","Floating Turtle",CFrame.new(-9324,114,-10214),CFrame.new(-9255,114,-9435)},
        {1975,1999,"HauntedQuest1",1,"Reborn Skeleton","Haunted Castle",CFrame.new(-9515.6,172.1,6078.6),CFrame.new(-8750,142,6040)},
        {2000,2024,"HauntedQuest1",2,"Living Zombie","Haunted Castle",CFrame.new(-9515.6,172.1,6078.6),CFrame.new(-8740,142,6220)},
        {2025,2049,"HauntedQuest2",1,"Demonic Soul","Haunted Castle",CFrame.new(-9515.6,172.1,6078.6),CFrame.new(-9500,162,5550)},
        {2050,2074,"HauntedQuest2",2,"Possessed Mummy","Haunted Castle",CFrame.new(-9515.6,172.1,6078.6),CFrame.new(-9650,142,5700)},
        {2075,2099,"TreatsQuest1",1,"Peanut Scout","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-1993,187,-10103)},
        {2100,2124,"TreatsQuest1",2,"Peanut Warrior","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-1993,187,-10103)},
        {2125,2149,"TreatsQuest2",1,"Ice Cream Chef","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-877,118,-11032)},
        {2150,2199,"TreatsQuest2",2,"Ice Cream Commander","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-877,118,-11032)},
        {2200,2224,"CakeQuest1",1,"Cookie Crafter","Sea of Treats",CFrame.new(-1975,48,-12028),CFrame.new(-2021,38,-12028)},
        {2225,2249,"CakeQuest1",2,"Cake Guard","Sea of Treats",CFrame.new(-1975,48,-12028),CFrame.new(-2021,38,-12028)},
        {2250,2299,"CakeQuest2",1,"Cake Assassin","Sea of Treats",CFrame.new(-1975,48,-12028),CFrame.new(-2021,38,-12028)},
        {2300,2324,"CakeQuest2",2,"Cake Captain","Sea of Treats",CFrame.new(-1975,48,-12028),CFrame.new(-2021,38,-12028)},
        {2325,2349,"CandyQuest1",1,"Candy Rebel","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-1993,187,-10103)},
        {2350,2399,"CandyQuest1",2,"Sweet Thief","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-1993,187,-10103)},
        {2400,2449,"CandyQuest2",1,"Cocoa Warrior","Sea of Treats",CFrame.new(-819.38,64.93,-10967.28),CFrame.new(-1993,187,-10103)},
        {2450,2474,"TikiQuest1",1,"Sunfire Captain","Tiki Outpost",CFrame.new(-16235,12,365),CFrame.new(-16235,12,365)},
        {2475,2499,"TikiQuest1",2,"Island Outlaw","Tiki Outpost",CFrame.new(-16235,12,365),CFrame.new(-16235,12,365)},
        {2500,9999,"TikiQuest2",1,"Tiki Commando","Tiki Outpost",CFrame.new(-16235,12,365),CFrame.new(-16235,12,365)},
    }
end

local function GetQuestEntry()
    local lv = level.Value
    local best = nil
    for _, q in ipairs(QuestDB) do
        if lv >= q[1] and lv <= q[2] then best = q end
    end
    if not best and #QuestDB > 0 then best = QuestDB[#QuestDB] end
    return best
end

local function TryEnterFishman()
    pcall(function() remote:InvokeServer("requestEntrance", Vector3.new(61163.85, 11.68, 1819.78)) end)
end

-- Loop de Farm principal
local function AutoFarmLoop()
    while _G.AutoFarm do
        task.wait(0.15)

        local char = plr.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        
        if hrp and hum and hum.Health > 0 then
            local q = GetQuestEntry()
            if q then
                local qName   = q[3]
                local qIndex  = q[4]
                local mobName = q[5]
                local NPC_CF  = q[7]
                local mobCF   = q[8]

                if not HasQuest() then
                    if string.find(qName, "Fishman") then
                        TryEnterFishman()
                        task.wait(0.5)
                    end
                    
                    local targetNPC = NPC_CF * CFrame.new(0, 3, 0)
                    local dist = (hrp.Position - targetNPC.Position).Magnitude
                    if dist > 150 then
                        flyTo(targetNPC, _G.TweenSpeed)
                    end
                    HoldAt(targetNPC)
                    task.wait(0.5)
                    
                    local npcName = nil
                    pcall(function()
                        local folders = {workspace}
                        local npcFolder = workspace:FindFirstChild("NPCs")
                        if npcFolder then table.insert(folders, npcFolder) end
                        
                        for _, folder in ipairs(folders) do
                            for _, v in ipairs(folder:GetChildren()) do
                                if v:IsA("Model") and v:FindFirstChild("Humanoid") and not Players:GetPlayerFromCharacter(v) then
                                    local root = v:FindFirstChild("HumanoidRootPart") or v:FindFirstChild("PrimaryPart")
                                    if root and (root.Position - targetNPC.Position).Magnitude < 30 then
                                        npcName = v.Name
                                        break
                                    end
                                end
                            end
                            if npcName then break end
                        end
                    end)
                    
                    if npcName then
                        pcall(function()
                            if remote then
                                remote:InvokeServer("Description", npcName)
                            end
                        end)
                        task.wait(0.3)
                    end

                    pcall(function()
                        if remote then
                            remote:InvokeServer("StartQuest", qName, qIndex)
                        end
                    end)
                    task.wait(0.6)
                else
                    local activeQuest = GetActiveQuestMob()
                    if activeQuest then
                        if level.Value > activeQuest[2] then
                            pcall(function() 
                                if remote then
                                    remote:InvokeServer("AbandonQuest") 
                                end
                            end)
                            task.wait(0.5)
                        else
                            mobName = activeQuest[5]
                            mobCF   = activeQuest[8]
                        end
                    end

                    if HasQuest() then
                        local mob = FindMob(mobName)

                        if mob and mob:FindFirstChild("HumanoidRootPart") then
                            local mobPos = mob.HumanoidRootPart.Position
                            local farmCF = CFrame.new(mobPos.X, mobPos.Y + 3, mobPos.Z)

                            local dist = (hrp.Position - farmCF.Position).Magnitude
                            if dist > 150 then
                                flyTo(farmCF, _G.TweenSpeed)
                            end

                            HoldAt(farmCF)
                            ClumpMobs(mobName, mobPos)
                            
                            -- Ativa Haki apenas se a configuração global estiver habilitada
                            if _G.AutoHaki then
                                ActivateHaki()
                            end
                            
                            EquipWeapon()

                            -- Ataque ativo
                            pcall(function()
                                local tool = char:FindFirstChildOfClass("Tool")
                                if tool then
                                    tool:Activate()
                                end
                            end)
                            pcall(function()
                                local combat = replicated:FindFirstChild("RigControllerWorkspaceServer") and replicated.RigControllerWorkspaceServer:FindFirstChild("Combat")
                                if combat then
                                    combat:FireServer()
                                end
                            end)
                            pcall(function()
                                VirtualUser:CaptureController()
                                VirtualUser:ClickButton1(Vector2.new(9e9, 9e9))
                                VirtualUser:Button1Down(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
                                VirtualUser:Button1Up(Vector2.new(0, 0), workspace.CurrentCamera.CFrame)
                            end)
                        else
                            local targetSpawn = mobCF * CFrame.new(0, 3, 0)
                            local dist = (hrp.Position - targetSpawn.Position).Magnitude
                            if dist > 150 then
                                flyTo(targetSpawn, _G.TweenSpeed)
                            end
                            HoldAt(targetSpawn)
                            task.wait(1)
                        end
                    end
                end
            else
                task.wait(1)
            end
        else
            task.wait(1)
        end
    end
end

local function AutoStatsLoop()
    while _G.AutoStats do
        task.wait(1.5)
        pcall(function()
            local pts = points.Value
            if pts > 0 and remote then
                remote:InvokeServer("AddPoint", _G.StatTarget, pts)
            end
        end)
    end
end

local function ApplyLowCpu()
    pcall(function()
        local l = game:GetService("Lighting")
        local t = workspace.Terrain
        t.WaterWaveSize   = 0
        t.WaterWaveSpeed  = 0
        t.WaterReflectance= 0
        t.WaterTransparency = 0
        l.GlobalShadows   = false
        l.FogEnd          = 9e9
        l.Brightness      = 0
        settings().Rendering.QualityLevel = "Level01"
        for _, v in pairs(game:GetDescendants()) do
            if not _G.LowCpu then break end
            if v:IsA("ParticleEmitter") or v:IsA("Trail") then
                v.Lifetime = NumberRange.new(0)
            elseif v:IsA("Fire") or v:IsA("Smoke") or v:IsA("Sparkles") or v:IsA("SpotLight") then
                v.Enabled = false
            elseif v:IsA("BlurEffect") or v:IsA("SunRaysEffect") or v:IsA("DepthOfFieldEffect") or v:IsA("BloomEffect") then
                v.Enabled = false
            end
        end
    end)
end

-- GUI (🐻 Brown Bear Theme)
local sg = Instance.new("ScreenGui")
sg.Name = "TedHub"
sg.ResetOnSpawn = false
sg.IgnoreGuiInset = true

-- Exclui qualquer GUI antiga para evitar sobreposição e bugs
local oldGui
pcall(function()
    oldGui = game:GetService("CoreGui"):FindFirstChild("TedHub")
end)
if not oldGui then
    pcall(function()
        local playerGui = plr:FindFirstChildOfClass("PlayerGui")
        if playerGui then
            oldGui = playerGui:FindFirstChild("TedHub")
        end
    end)
end
if oldGui then
    pcall(function() oldGui:Destroy() end)
end

-- Coloca na CoreGui de forma segura, se falhar cai na PlayerGui
local function ParentGui()
    local success = pcall(function()
        sg.Parent = game:GetService("CoreGui")
    end)
    if not success or not sg.Parent then
        local playerGui = plr:FindFirstChildOfClass("PlayerGui")
        while not playerGui do
            task.wait(0.5)
            playerGui = plr:FindFirstChildOfClass("PlayerGui")
        end
        sg.Parent = playerGui
    end
end
ParentGui()

local C_BG      = Color3.fromRGB(10,  7,  5)
local C_TOPBAR  = Color3.fromRGB(22, 14,  8)
local C_SIDEBAR = Color3.fromRGB(18, 11,  7)
local C_CARD    = Color3.fromRGB(24, 15,  9)
local C_ACCENT  = Color3.fromRGB(165, 90, 30)
local C_ACCENT2 = Color3.fromRGB(200,120, 45)
local C_TAB_ACT = Color3.fromRGB(140, 75, 22)
local C_TAB_OFF = Color3.fromRGB(28,  18, 11)
local C_TEXT    = Color3.fromRGB(235,215,190)
local C_SUBTEXT = Color3.fromRGB(160,130,100)
local C_ON      = Color3.fromRGB(155, 85, 25)
local C_OFF     = Color3.fromRGB(45,  30, 18)
local C_CLOSE   = Color3.fromRGB(160, 50, 30)

-- Instanciação seguindo a diretriz oficial (propriedades primeiro, parent por último)
local main = Instance.new("Frame")
main.Name = "Main"
main.Size = UDim2.new(0, 490, 0, 330)
main.Position = UDim2.new(0.5, -245, 0.5, -165)
main.BackgroundColor3 = C_BG
main.BorderSizePixel = 0
main.Active = true
main.Parent = sg

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 12)
corner.Parent = main

local stroke = Instance.new("UIStroke")
stroke.Thickness = 1.5
stroke.Color = C_ACCENT
stroke.Parent = main

local topBar = Instance.new("Frame")
topBar.Size = UDim2.new(1, 0, 0, 44)
topBar.BackgroundColor3 = C_TOPBAR
topBar.BorderSizePixel = 0
topBar.Parent = main

local topCorner = Instance.new("UICorner")
topCorner.CornerRadius = UDim.new(0, 12)
topCorner.Parent = topBar

local fix = Instance.new("Frame")
fix.Size = UDim2.new(1, 0, 0, 12)
fix.Position = UDim2.new(0, 0, 1, -12)
fix.BackgroundColor3 = C_TOPBAR
fix.BorderSizePixel = 0
fix.Parent = topBar

local titleLbl = Instance.new("TextLabel")
titleLbl.Size = UDim2.new(1, -60, 1, 0)
titleLbl.Position = UDim2.new(0, 14, 0, 0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text = "🐻 TED HUB  •  " .. SeaName
titleLbl.TextColor3 = C_TEXT
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.Font = Enum.Font.GothamBold
titleLbl.TextSize = 14
titleLbl.Parent = topBar

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 28, 0, 28)
closeBtn.Position = UDim2.new(1, -38, 0.5, -14)
closeBtn.BackgroundColor3 = C_CLOSE
closeBtn.Text = "✕"
closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 12
closeBtn.Parent = topBar

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 6)
closeCorner.Parent = closeBtn

closeBtn.MouseButton1Click:Connect(function()
    _G.AutoFarm = false
    _G.AutoStats = false
    shouldTween = false
    sg:Destroy()
end)

local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.new(0, 125, 1, -44)
sidebar.Position = UDim2.new(0, 0, 0, 44)
sidebar.BackgroundColor3 = C_SIDEBAR
sidebar.BorderSizePixel = 0
sidebar.Parent = main

local sideList = Instance.new("UIListLayout")
sideList.Padding = UDim.new(0, 5)
sideList.HorizontalAlignment = Enum.HorizontalAlignment.Center
sideList.VerticalAlignment = Enum.VerticalAlignment.Top
sideList.Parent = sidebar

local sidePad = Instance.new("UIPadding")
sidePad.PaddingTop = UDim.new(0, 10)
sidePad.Parent = sidebar

local contentArea = Instance.new("Frame")
contentArea.Size = UDim2.new(1, -125, 1, -44)
contentArea.Position = UDim2.new(0, 125, 0, 44)
contentArea.BackgroundTransparency = 1
contentArea.Parent = main

-- Função premium de arrasto da GUI (Mobile e PC)
local function MakeDraggable(frame)
    local dragToggle = nil
    local dragSpeed = 0.15
    local dragStart = nil
    local startPos = nil

    local function updateInput(input)
        local delta = input.Position - dragStart
        local position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        TweenService:Create(frame, TweenInfo.new(dragSpeed), {Position = position}):Play()
    end

    frame.InputBegan:Connect(function(input)
        if (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) and UserInputService:GetFocusedTextBox() == nil then
            dragToggle = true
            dragStart = input.Position
            startPos = frame.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragToggle = false
                end
            end)
        end
    end)

    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            if dragToggle then
                updateInput(input)
            end
        end
    end)
end
MakeDraggable(main)

local tabs, pages = {}, {}
local function MakePage()
    local sf = Instance.new("ScrollingFrame")
    sf.Size = UDim2.new(1, -14, 1, -14)
    sf.Position = UDim2.new(0, 7, 0, 7)
    sf.BackgroundTransparency = 1
    sf.ScrollBarThickness = 3
    sf.ScrollBarImageColor3 = C_ACCENT
    sf.CanvasSize = UDim2.new(0, 0, 0, 0)
    sf.Visible = false
    sf.Parent = contentArea

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 7)
    layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    layout.Parent = sf

    layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        sf.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 14)
    end)
    return sf
end

local function MakeTab(label, icon)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.9, 0, 0, 34)
    btn.BackgroundColor3 = C_TAB_OFF
    btn.Text = icon .. "  " .. label
    btn.TextColor3 = C_SUBTEXT
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 11
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.BorderSizePixel = 0
    btn.Parent = sidebar

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = btn

    local page = MakePage()
    table.insert(tabs, btn)
    table.insert(pages, page)

    btn.MouseButton1Click:Connect(function()
        for _, b in pairs(tabs) do b.BackgroundColor3 = C_TAB_OFF; b.TextColor3 = C_SUBTEXT end
        for _, p in pairs(pages) do p.Visible = false end
        btn.BackgroundColor3 = C_TAB_ACT; btn.TextColor3 = C_TEXT; page.Visible = true
    end)
    return page, btn
end

local function Card(parent, h)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(0.97, 0, 0, h or 46)
    f.BackgroundColor3 = C_CARD
    f.BorderSizePixel = 0
    f.Parent = parent

    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(0, 8)
    fCorner.Parent = f

    return f
end

local function InfoCard(parent, top, bot)
    local f = Card(parent, 52)
    
    local t1 = Instance.new("TextLabel")
    t1.Size = UDim2.new(1, -16, 0.45, 0)
    t1.Position = UDim2.new(0, 10, 0, 5)
    t1.BackgroundTransparency = 1
    t1.Text = top
    t1.TextColor3 = C_SUBTEXT
    t1.Font = Enum.Font.GothamSemibold
    t1.TextSize = 10
    t1.TextXAlignment = Enum.TextXAlignment.Left
    t1.Parent = f

    local t2 = Instance.new("TextLabel")
    t2.Size = UDim2.new(1, -16, 0.5, 0)
    t2.Position = UDim2.new(0, 10, 0.45, 0)
    t2.BackgroundTransparency = 1
    t2.Text = bot
    t2.TextColor3 = C_TEXT
    t2.Font = Enum.Font.Gotham
    t2.TextSize = 12
    t2.TextXAlignment = Enum.TextXAlignment.Left
    t2.Parent = f

    return {Set = function(s) t2.Text = s end}
end

local function Toggle(parent, label, default, cb)
    local f = Card(parent, 44)
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.65, 0, 1, 0)
    lbl.Position = UDim2.new(0, 12, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = C_TEXT
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = f

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 56, 0, 24)
    btn.Position = UDim2.new(1, -66, 0.5, -12)
    btn.BackgroundColor3 = default and C_ON or C_OFF
    btn.Text = default and "ON" or "OFF"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 10
    btn.Parent = f

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = btn

    local state = default
    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.BackgroundColor3 = state and C_ON or C_OFF
        btn.Text = state and "ON" or "OFF"
        cb(state)
    end)
end

local function Cycle(parent, label, opts, defIdx, cb)
    local f = Card(parent, 44)
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.5, 0, 1, 0)
    lbl.Position = UDim2.new(0, 12, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = C_TEXT
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = f

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 115, 0, 26)
    btn.Position = UDim2.new(1, -126, 0.5, -13)
    btn.BackgroundColor3 = Color3.fromRGB(28, 18, 10)
    btn.Text = opts[defIdx]
    btn.TextColor3 = C_ACCENT2
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 10
    btn.Parent = f

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = btn

    local idx = defIdx
    btn.MouseButton1Click:Connect(function()
        idx = idx >= #opts and 1 or idx + 1
        btn.Text = opts[idx]
        cb(opts[idx])
    end)
end

local function Button(parent, label, cb)
    local f = Card(parent, 44)
    
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -24, 1, -14)
    btn.Position = UDim2.new(0, 12, 0, 7)
    btn.BackgroundColor3 = C_ACCENT
    btn.Text = label
    btn.TextColor3 = C_TEXT
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    btn.Parent = f

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = btn

    btn.MouseButton1Click:Connect(cb)
    return btn
end

local function Slider(parent, label, min, max, default, cb)
    local f = Card(parent, 56)
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.6, 0, 0, 24)
    lbl.Position = UDim2.new(0, 12, 0, 4)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = C_TEXT
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = f
    
    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0.3, 0, 0, 24)
    valLbl.Position = UDim2.new(0.7, -12, 0, 4)
    valLbl.BackgroundTransparency = 1
    valLbl.Text = tostring(default)
    valLbl.TextColor3 = C_ACCENT2
    valLbl.Font = Enum.Font.GothamBold
    valLbl.TextSize = 12
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.Parent = f
    
    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1, -24, 0, 6)
    bar.Position = UDim2.new(0, 12, 0.7, -3)
    bar.BackgroundColor3 = C_OFF
    bar.Parent = f

    local barCorner = Instance.new("UICorner")
    barCorner.CornerRadius = UDim.new(0, 3)
    barCorner.Parent = bar
    
    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
    fill.BackgroundColor3 = C_ACCENT
    fill.Parent = bar

    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(0, 3)
    fillCorner.Parent = fill
    
    local trigger = Instance.new("TextButton")
    trigger.Size = UDim2.new(1, 0, 1, 0)
    trigger.BackgroundTransparency = 1
    trigger.Text = ""
    trigger.Parent = bar
    
    local dragging = false
    local function update(input)
        local pos = math.clamp((input.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteWidth, 0, 1)
        fill.Size = UDim2.new(pos, 0, 1, 0)
        local value = math.floor(min + (pos * (max - min)))
        valLbl.Text = tostring(value)
        cb(value)
    end
    
    trigger.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            update(input)
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
    
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

-- ════════════════════════════════════════════════════════════
--   TABS CREATION
-- ════════════════════════════════════════════════════════════
local pgHome,btnHome       = MakeTab("Home","🏠")
local pgFarm,_             = MakeTab("Auto Farm","⚔️")
local pgStats,_            = MakeTab("Stats","📊")
local pgTeleports,_        = MakeTab("Teleports","📍")
local pgMisc,_             = MakeTab("Outros","⚙️")

btnHome.BackgroundColor3=C_TAB_ACT; btnHome.TextColor3=C_TEXT; pgHome.Visible=true

-- HOME TAB CONTENT
local lvInfo  = InfoCard(pgHome,"Player Level",tostring(level.Value))
local ptInfo  = InfoCard(pgHome,"Stat Points", tostring(points.Value))
local seaInfo = InfoCard(pgHome,"Current Sea",SeaName)
Toggle(pgHome,"Anti-Lag / Boost FPS",false,function(on) _G.LowCpu=on; if on then task.spawn(ApplyLowCpu) end end)
Toggle(pgHome,"Bring / Clump Mobs",true,function(on) _G.BringMobs=on end)

task.spawn(function()
    while sg and sg.Parent do
        task.wait(2)
        pcall(function() lvInfo.Set(tostring(level.Value)); ptInfo.Set(tostring(points.Value)) end)
    end
end)

-- FARM TAB CONTENT
local islandInfo = InfoCard(pgFarm,"Target Island","Auto-Detecting...")
local mobInfo    = InfoCard(pgFarm,"Target Mob","Auto-Detecting...")
local questInfo  = InfoCard(pgFarm,"Quest Name","Auto-Detecting...")

task.spawn(function()
    while sg and sg.Parent do
        task.wait(2)
        pcall(function()
            local q=GetQuestEntry()
            if q then
                islandInfo.Set(q[6])
                mobInfo.Set(string.format("%s  (Lv %d - %d)",q[5],q[1],q[2]))
                questInfo.Set(q[3].."  ["..q[4].."]")
            else
                islandInfo.Set("No data for level "..tostring(level.Value))
            end
        end)
    end
end)

Toggle(pgFarm,"Enable Auto Farm",false,function(on)
    _G.AutoFarm = on
    if on then
        pcall(function() 
            if remote then
                remote:InvokeServer("AbandonQuest") 
            end
        end)
        task.wait(0.3)
        task.spawn(AutoFarmLoop)
    else
        shouldTween = false
        CleanPhysics()
    end
end)
Toggle(pgFarm,"Haki de Armamento Automático",true,function(on) _G.AutoHaki=on end)
Cycle(pgFarm,"Weapon Type",{"Melee","Sword","Blox Fruit","Gun"},1,function(opt) _G.SelectedWeapon=opt end)

-- STATS TAB CONTENT
Toggle(pgStats,"Auto Assign Points",false,function(on) _G.AutoStats=on; if on then task.spawn(AutoStatsLoop) end end)
Cycle(pgStats,"Stat Category",{"Melee","Defense","Sword","Gun","Demon Fruit"},1,function(opt) _G.StatTarget=opt end)

-- TELEPORTS TAB CONTENT
local Teleports = {}
if W1 then
    Teleports = {
        {"Starter Island", CFrame.new(1045.96, 27.00, 1560.82)},
        {"Jungle", CFrame.new(-1598.09, 35.55, 153.38)},
        {"Pirate Village", CFrame.new(-1141.07, 4.10, 3831.55)},
        {"Desert", CFrame.new(894.49, 5.14, 4392.43)},
        {"Frozen Village", CFrame.new(1389.74, 88.15, -1298.91)},
        {"Marine Fortress", CFrame.new(-5039.59, 27.35, 4324.68)},
        {"Skypiea (Sky)", CFrame.new(-4839.53, 716.37, -2619.44)},
        {"Prison", CFrame.new(5308.93, 1.66, 475.12)},
        {"Colosseum", CFrame.new(-1580.05, 6.35, -2986.48)},
        {"Magma Village", CFrame.new(-5313.37, 10.95, 8515.29)},
        {"Underwater City", CFrame.new(61122.65, 18.50, 1569.40)},
        {"Fountain City", CFrame.new(5259.82, 37.35, 4050.03)},
    }
elseif W2 then
    Teleports = {
        {"Kingdom of Rose", CFrame.new(-429.54, 71.77, 1836.18)},
        {"Cafe (Zone 1)", CFrame.new(-380.44, 72.99, 290.11)},
        {"Green Zone", CFrame.new(-2440.80, 71.71, -3216.07)},
        {"Dark Arena", CFrame.new(638.44, 71.77, 918.28)},
        {"Thriller Bark", CFrame.new(-5497.06, 47.59, -795.24)},
        {"Snow Mountain", CFrame.new(609.86, 400.12, -5372.26)},
        {"Ice Castle", CFrame.new(-5429.05, 15.98, -5297.96)},
        {"Forgotten Island", CFrame.new(-3053.98, 237.19, -10145.04)},
        {"Hot and Cold", CFrame.new(5668.98, 28.52, -6483.35)},
    }
elseif W3 then
    Teleports = {
        {"Port Town", CFrame.new(-289.77, 43.82, 5579.94)},
        {"Hydra Island", CFrame.new(-13232.68, 332.40, -7626.01)},
        {"Great Tree", CFrame.new(2179.30, 28.73, -6739.97)},
        {"Floating Turtle", CFrame.new(-10452.92, 332.18, -8511.45)},
        {"Haunted Castle", CFrame.new(-9515.6, 172.1, 6078.6)},
        {"Sea of Treats", CFrame.new(-819.38, 64.93, -10967.28)},
        {"Castle on the Sea", CFrame.new(-5015.42, 315.67, -3157.01)},
        {"Tiki Outpost", CFrame.new(-16235.12, 12.35, 365.12)},
    }
end

for _, dest in ipairs(Teleports) do
    Button(pgTeleports, "Ir para: " .. dest[1], function()
        Teleport(dest[2])
    end)
end

-- MISC TAB CONTENT
Toggle(pgMisc, "Pulo Infinito", false, function(on)
    InfiniteJump = on
end)

Toggle(pgMisc, "Habilitar WalkSpeed Customizado", false, function(on)
    useSpeed = on
    if not on then
        pcall(function() plr.Character.Humanoid.WalkSpeed = 16 end)
    end
end)

Slider(pgMisc, "Velocidade", 16, 250, 16, function(val)
    customSpeed = val
end)

Toggle(pgMisc, "Habilitar JumpPower Customizado", false, function(on)
    useJump = on
    if not on then
        pcall(function() plr.Character.Humanoid.JumpPower = 50 end)
    end
end)

Slider(pgMisc, "Poder de Pulo", 50, 300, 50, function(val)
    customJump = val
end)

Slider(pgMisc, "Velocidade do Voo (Tween)", 100, 350, 250, function(val)
    _G.TweenSpeed = val
end)

Button(pgMisc, "Trocar de Servidor (Server Hop)", function()
    local HttpService = game:GetService("HttpService")
    local TeleportService = game:GetService("TeleportService")
    pcall(function()
        local servers = HttpService:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/" .. placeId .. "/servers/Public?sortOrder=Asc&limit=100"))
        for _, s in pairs(servers.data) do
            if s.playing < s.maxPlayers and s.id ~= game.JobId then
                TeleportService:TeleportToPlaceInstance(placeId, s.id, plr)
                break
            end
        end
    end)
end)

Button(pgMisc, "Reconectar ao Servidor (Rejoin)", function()
    pcall(function()
        game:GetService("TeleportService"):Teleport(placeId, plr)
    end)
end)

print("[Ted Hub] Loaded. Sea: "..SeaName)

