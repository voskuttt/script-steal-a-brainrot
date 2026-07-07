if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
UIS = game:GetService("UserInputService")
RunService = game:GetService("RunService")
Stats = game:GetService("Stats")
TweenService = game:GetService("TweenService")
HttpService = game:GetService("HttpService")
ReplicatedStorage = game:GetService("ReplicatedStorage")
Workspace = game:GetService("Workspace")
Lighting = game:GetService("Lighting")
TeleportService = game:GetService("TeleportService")
CoreGui = game:GetService("CoreGui")
VirtualInputManager = game:GetService("VirtualInputManager")

-- Unlimited FPS: remove Roblox's default 240 FPS cap. 0 = uncapped on most executors.
pcall(function() if setfpscap then setfpscap(0) end end)

-- Insta-reset state captured by the FireServer hook below.
-- GUID is the first arg of the balloon payload. Defaults to a randomly-generated
-- UUID so the payload is at least syntactically valid (an empty string was
-- silently rejected by the server -- that's why the reset wasn't firing). If you
-- have a known-good GUID, set `_G.ALPHA_RESET_GUID` before the script runs.
local function _newGUID()
    local hex = "0123456789abcdef"
    local t = {}
    for i = 1, 32 do t[#t+1] = hex:sub(math.random(1,16), math.random(1,16)) end
    return table.concat(t, "", 1, 8) .. "-" .. table.concat(t, "", 9, 12)
        .. "-" .. table.concat(t, "", 13, 16) .. "-" .. table.concat(t, "", 17, 20)
        .. "-" .. table.concat(t, "", 21, 32)
end

local function makeOneWay(plat)
    if not plat then return end
    local rsConn
    local lastY = nil
    rsConn = game:GetService("RunService").Stepped:Connect(function()
        if not plat or not plat.Parent then
            if rsConn then rsConn:Disconnect() end
            return
        end
        local char = game.Players.LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if hrp then
            local currentY = hrp.Position.Y
            if not lastY then lastY = currentY end
            
            local deltaY = currentY - lastY
            local isMovingUp = (hrp.AssemblyLinearVelocity.Y > 1) or (deltaY > 0.01 and deltaY < 5)
            
            if isMovingUp then
                plat.CanCollide = false
            else
                if currentY > plat.Position.Y + 0.1 then
                    plat.CanCollide = true
                else
                    plat.CanCollide = false
                end
            end
            
            lastY = currentY
        end
    end)
end

GUID = GUID or _G.ALPHA_RESET_GUID or _newGUID()
resetRemote = nil
instaResetCooldown = false

local old; old = hookfunction(Instance.new("RemoteEvent").FireServer, function(self, ...)
    local args = {...}
    local arg1 = args[1]

    -- Capture the first RE/* remote the game itself fires -- that's the reset
    -- target for instareset() below.
    if not resetRemote and self.Name:sub(1, 3) == "RE/" then
        resetRemote = self
    end

    if #self.Name == 67 and arg1 and typeof(arg1) == "string" then
        if string.find(arg1, "StopTrying") then
            print("ez bypass")
            return
        end
    end
    return old(self, ...)
end)

-- LPH_NO_VIRTUALIZE: real preprocessor directive under Luraph; harmless passthrough
-- everywhere else. Declared once globally so every heavy hot-path body can wrap
-- itself without each call site re-shimming.
if not LPH_NO_VIRTUALIZE then LPH_NO_VIRTUALIZE = function(fn) return fn end end

-- =====================================================================
-- LAZY-LOAD QUEUE
-- =====================================================================
_G.__SXELazyQ = _G.__SXELazyQ or {}
local function LazyInit(name, fn)
    table.insert(_G.__SXELazyQ, { name = name, fn = LPH_NO_VIRTUALIZE(fn) })
end
task.defer(LPH_NO_VIRTUALIZE(function()
    while true do
        if #_G.__SXELazyQ > 0 then
            local item = table.remove(_G.__SXELazyQ, 1)
            local ok, err = pcall(item.fn)
            if not ok then warn("[ALPHA Lazy]", item.name, err) end
            task.wait(0.08)
        else
            task.wait(0.5)
        end
    end
end))

-- Persistent panel visibility store that survives script re-executes within the same Roblox session
_G._SXEPanelVis = _G._SXEPanelVis or {}

_G.lazyUIs = {}
_G.addLazyUI = function(element, targetVis, isScreenGui, panelName)
    if element then
        if isScreenGui then
            element.Enabled = false
        else
            element.Visible = false
        end
        table.insert(_G.lazyUIs, {element = element, targetVis = targetVis, isScreenGui = isScreenGui, cancelled = false, panelName = panelName})
    end
end
_G.cancelLazyUI = function(element)
    for _, item in ipairs(_G.lazyUIs) do
        if item.element == element then
            item.cancelled = true
            break
        end
    end
end
task.delay(4.0, function()
    for _, item in ipairs(_G.lazyUIs) do
        if item.element and not item.cancelled then
            local vis
            if item.panelName then
                -- Priority: _G store (survives re-exec) > Config.Visibilities > default true
                local fromG = _G._SXEPanelVis[item.panelName]
                if fromG ~= nil then
                    vis = fromG
                elseif Config and Config.Visibilities then
                    local saved = Config.Visibilities[item.panelName]
                    if saved ~= nil then vis = saved else vis = true end
                else
                    vis = item.targetVis
                end
            else
                vis = item.targetVis
            end
            if item.isScreenGui then
                pcall(function() item.element.Enabled = vis end)
            elseif item.element.Parent then
                pcall(function() item.element.Visible = vis end)
            end
        end
    end
    if _G.initRemoteSellLazy then
        pcall(_G.initRemoteSellLazy)
    end
end)

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
pcall(function() Players.RespawnTime = 0 end)

-- GLOBAL SCALE SYSTEM (Mobile Friendly Auto-Scaling)
local GlobalUIScaleVal = 1
local scaledGuis = {}

local function getGlobalScale()
    return GlobalUIScaleVal
end

local function updateAllGuisScale(newScale)
    GlobalUIScaleVal = newScale
    for sg, master in pairs(scaledGuis) do
        pcall(function()
            if sg and sg.Parent and master and master.Parent then
                local scaleObj = master:FindFirstChild("ALPHA_GlobalScale")
                if scaleObj then
                    scaleObj.Scale = newScale
                end
                master.Size = UDim2.new(1 / newScale, 0, 1 / newScale, 0)
            end
        end)
    end
end

local function registerScreenGui(sg)
    local master = sg:FindFirstChild("ALPHA_MasterFrame")
    if not master then
        master = Instance.new("Frame")
        master.Name = "ALPHA_MasterFrame"
        master.BackgroundTransparency = 1
        master.BorderSizePixel = 0
        master.Parent = sg
        
        local scaleObj = Instance.new("UIScale")
        scaleObj.Name = "ALPHA_GlobalScale"
        scaleObj.Parent = master
    end
    scaledGuis[sg] = master
    
    pcall(function()
        local scaleObj = master:FindFirstChild("ALPHA_GlobalScale")
        if scaleObj then
            scaleObj.Scale = GlobalUIScaleVal
        end
        master.Size = UDim2.new(1 / GlobalUIScaleVal, 0, 1 / GlobalUIScaleVal, 0)
    end)
    return master
end

local function recalculateScale()
    local cam = Workspace.CurrentCamera
    if not cam then return end
    local size = cam.ViewportSize
    local h = size.Y
    
    local isMobile = UIS.TouchEnabled
    local newScale = 1
    if isMobile then
        -- Mobile/Tablet scaling: aggressive down-scale to perfectly fit small landscape screens
        newScale = math.clamp(h / 1000, 0.40, 0.58)
    else
        -- PC/Console scaling: standard comfortable scaling
        newScale = math.clamp(h / 800, 0.65, 1.0)
    end
    updateAllGuisScale(newScale)
end

local cameraConn
local function setupCameraListener()
    if cameraConn then pcall(function() cameraConn:Disconnect() end) end
    local cam = Workspace.CurrentCamera
    if cam then
        cameraConn = cam:GetPropertyChangedSignal("ViewportSize"):Connect(recalculateScale)
        recalculateScale()
    end
end
Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(setupCameraListener)
task.spawn(setupCameraListener)

local old = playerGui:FindFirstChild("AlphaHub_V3"); if old then old:Destroy() end
local gui_sg = Instance.new("ScreenGui"); gui_sg.Name = "AlphaHub_V3"; gui_sg.ResetOnSpawn = false; gui_sg.IgnoreGuiInset = true; gui_sg.DisplayOrder = 9999999; gui_sg.Parent = playerGui
local gui = registerScreenGui(gui_sg)

-- SHARED TOGGLE STATE
local ToggleState = {}
local function regToggle(name, default)
    if not ToggleState[name] then ToggleState[name] = {value = default or false, listeners = {}} end
end
local function getToggle(name) return ToggleState[name] and ToggleState[name].value or false end
local function setToggle(name, val, skipNotify)
    regToggle(name)
    ToggleState[name].value = val
    if not skipNotify then
        for _, fn in ipairs(ToggleState[name].listeners) do pcall(fn, val) end
    end
end
local function onToggleChanged(name, fn)
    regToggle(name); table.insert(ToggleState[name].listeners, fn)
end

-- GLOBAL CONFIG FORWARD DECLARATIONS (to avoid undefined/shadowing issues)
local Config
local saveConfig
local loadConfig

-- THEME (ALPHA Red)
Themes = {
    Light = {
        Background=Color3.fromRGB(255,255,255), MainBackground=Color3.fromRGB(255, 240, 240),
        Panel=Color3.fromRGB(255, 245, 245), Row=Color3.fromRGB(255, 250, 250), RowHover=Color3.fromRGB(255, 245, 245),
        Accent=Color3.fromRGB(255, 0, 0), AccentLight=Color3.fromRGB(255, 50, 50),
        Green=Color3.fromRGB(255, 30, 30), Red=Color3.fromRGB(200, 0, 0), Red2=Color3.fromRGB(180, 0, 0),
        Text=Color3.fromRGB(255, 40, 40), Dim=Color3.fromRGB(150, 50, 50), Stroke=Color3.fromRGB(200, 100, 100),
        SoftButton=Color3.fromRGB(255, 248, 248), SoftButtonHover=Color3.fromRGB(255, 245, 245),
        SoftAccent=Color3.fromRGB(255, 242, 242), SoftAccentHover=Color3.fromRGB(255, 240, 240),
        ToggleOff=Color3.fromRGB(255, 245, 245), ToggleOff2=Color3.fromRGB(255, 248, 248),
        InputBg=Color3.fromRGB(255,255,255), SliderBg=Color3.fromRGB(255, 200, 200),
        BlacklistHover=Color3.fromRGB(255, 200, 200), BlacklistLeave=Color3.fromRGB(255, 248, 248),
    },
    Dark = {
        Background=Color3.fromRGB(20,20,20), MainBackground=Color3.fromRGB(15,15,15),
        Panel=Color3.fromRGB(28,25,28), Row=Color3.fromRGB(35,30,35), RowHover=Color3.fromRGB(48,38,48),
        Accent=Color3.fromRGB(255, 0, 0), AccentLight=Color3.fromRGB(255, 50, 50),
        Green=Color3.fromRGB(255, 30, 30), Red=Color3.fromRGB(200, 0, 0), Red2=Color3.fromRGB(180, 0, 0),
        Text=Color3.fromRGB(255,255,255), Dim=Color3.fromRGB(200,200,200), Stroke=Color3.fromRGB(80, 20, 20),
        SoftButton=Color3.fromRGB(35,28,33), SoftButtonHover=Color3.fromRGB(45,35,42),
        SoftAccent=Color3.fromRGB(55,38,48), SoftAccentHover=Color3.fromRGB(65,45,58),
        ToggleOff=Color3.fromRGB(35,28,33), ToggleOff2=Color3.fromRGB(35,28,33),
        InputBg=Color3.fromRGB(30,25,30), SliderBg=Color3.fromRGB(80, 30, 30),
        BlacklistHover=Color3.fromRGB(100, 20, 20), BlacklistLeave=Color3.fromRGB(70, 30, 30),
    }
}
Theme = {}
for k, v in pairs(Themes.Light) do
    Theme[k] = v
end

function applyTheme(themeName)
    local fromTheme = {}
    for k, v in pairs(Theme) do
        fromTheme[k] = v
    end
    
    local toTheme = Themes[themeName] or Themes.Light
    for k, v in pairs(toTheme) do
        Theme[k] = v
    end
    
    -- Helper 1: Explicitly style panels to avoid color collision bugs
    local function updatePanelTheme(f, isMain)
        if not f then return end
        f.BackgroundColor3 = isMain and toTheme.MainBackground or toTheme.Background
        for _, child in ipairs(f:GetChildren()) do
            if child:IsA("UIStroke") then
                child.Color = toTheme.AccentLight
            elseif child:IsA("Frame") then
                if child.Size == UDim2.new(1,-24,0,1) then
                    child.BackgroundColor3 = toTheme.AccentLight
                elseif child.BackgroundTransparency == 1 and child.Size == UDim2.new(1,0,0,42) then
                    for _, sub in ipairs(child:GetChildren()) do
                        if sub:IsA("TextLabel") then
                            if sub.TextSize == 16 or sub.TextSize == 12 then
                                sub.TextColor3 = toTheme.Text
                            elseif sub.TextSize == 10 then
                                sub.TextColor3 = toTheme.Dim
                            end
                        end
                    end
                end
            end
        end
    end

    -- Helper 2: Explicitly style bottom bar
    local function updateBottomBarTheme()
        if not bottomBar then return end
        bottomBar.BackgroundColor3 = toTheme.Background
        for _, child in ipairs(bottomBar:GetChildren()) do
            if child:IsA("UIStroke") then
                child.Color = toTheme.AccentLight
            elseif child:IsA("TextLabel") then
                if child.Text == "ALPHA HUB PRIVAT" or child.Text == "|" or child.Text == "discord.gg/j2pxrygT4" then
                    child.TextColor3 = toTheme.AccentLight
                elseif child.Text == "By:@SE67 and @SXLVATORE" then
                    child.TextColor3 = toTheme.Dim
                end
            elseif child:IsA("Frame") then
                if child.Size == UDim2.new(0,1,0,36) then
                    child.BackgroundColor3 = toTheme.Accent
                end
            end
        end
    end

    -- Helper 3: Explicitly style admin panel rows
    local function updateAdminPanelTheme()
        if not apBG then return end
        apBG.BackgroundColor3 = toTheme.Background
        local idx = 0
        for uid, row in pairs(apRows) do
            if row and row.Parent then
                idx = idx + 1
                local isAlt = (idx % 2 == 0)
                local rowCol = isAlt and toTheme.Row or toTheme.Panel
                row.BackgroundColor3 = rowCol
                local plr = Players:GetPlayerByUserId(uid)
                local isBlacklisted = isPlayerBlacklisted and isPlayerBlacklisted(plr)
                for _, child in ipairs(row:GetChildren()) do
                    if child:IsA("Frame") then
                        if child.Size == UDim2.fromOffset(34,34) then
                            child.BackgroundColor3 = toTheme.InputBg
                            local stroke = child:FindFirstChildOfClass("UIStroke")
                            if stroke then stroke.Color = toTheme.Accent end
                        elseif child.ZIndex == 12 then
                            for _, btn in ipairs(child:GetChildren()) do
                                if btn:IsA("TextButton") then
                                    if btn.Name == "BlacklistBtn" then
                                        btn.BackgroundColor3 = isBlacklisted and Color3.fromRGB(255, 60, 60) or toTheme.BlacklistLeave
                                    else
                                        btn.BackgroundColor3 = toTheme.SoftButton
                                    end
                                end
                            end
                        end
                    elseif child:IsA("TextLabel") then
                        if child.TextSize == 14 then
                            child.TextColor3 = toTheme.Text
                        elseif child.TextSize == 10 then
                            child.TextColor3 = toTheme.Dim
                        elseif child.TextSize == 11 then
                            child.TextColor3 = toTheme.AccentLight
                        end
                    end
                end
            end
        end
    end

    -- Run explicit styling on all static panels
    pcall(function()
        updatePanelTheme(main, true)
        updateBottomBarTheme()
        updateAdminPanelTheme()
        for _, name in ipairs({"Invisible Steal Panel", "Admin Command Panel", "Command Cooldowns", "Actions", "Steal Panel", "Steal Target"}) do
            updatePanelTheme(panels[name], false)
        end
        updatePanelTheme(actionSettingsPanel, false)
        updatePanelTheme(tpSpeedSettingsPanel, false)
    end)
    
    -- Fallback/Recursive styling for other dynamic frames (excluding mainBody descendants!)
    local function updateInstanceColors(inst)
        local shouldStyle = true
        if inst:IsA("GuiObject") or inst:IsA("UIStroke") then
            -- AVOID COLOR COLLISION BUG: Skip styling anything inside the tab mainBody dynamically
            if mainBody and inst:IsDescendantOf(mainBody) then return end
            
            -- Avoid updating main structures explicitly styled above
            if inst == main or inst == bottomBar or inst == apBG then
                shouldStyle = false
            end
            for _, p in ipairs({"Invisible Steal Panel", "Admin Command Panel", "Command Cooldowns", "Actions", "Steal Panel", "Steal Target"}) do
                if inst == panels[p] then
                    shouldStyle = false
                end
            end
            if inst == actionSettingsPanel or inst == tpSpeedSettingsPanel then
                shouldStyle = false
            end

            if shouldStyle then
                local bgKeys = {
                    Background=true, MainBackground=true, Panel=true, Row=true, RowHover=true,
                    SoftButton=true, SoftButtonHover=true, SoftAccent=true, SoftAccentHover=true,
                    ToggleOff=true, ToggleOff2=true, InputBg=true, SliderBg=true,
                    BlacklistHover=true, BlacklistLeave=true
                }
                local textKeys = {
                    Text=true, Dim=true, Accent=true, AccentLight=true,
                    Green=true, Red=true, Red2=true, Stroke=true
                }
                local properties = {
                    BackgroundColor3 = bgKeys,
                    TextColor3 = textKeys,
                    PlaceholderColor3 = textKeys,
                    Color = textKeys
                }
                
                for prop, allowedKeys in pairs(properties) do
                    pcall(function()
                        local current = inst[prop]
                        if typeof(current) == "Color3" then
                            -- PROTECT WHITE TEXT ON SPECIFIC BUTTONS FROM THEME CONVERSION
                            if prop == "TextColor3" and inst.Name == "WhiteTextBtn" then
                                return
                            end
                            if prop == "BackgroundColor3" and inst.Name == "WhiteSliderKnob" then
                                return
                            end
                            for k, val in pairs(fromTheme) do
                                if allowedKeys[k] then
                                    if (current.R - val.R)^2 + (current.G - val.G)^2 + (current.B - val.B)^2 < 0.0001 then
                                        inst[prop] = toTheme[k]
                                        break
                                    end
                                end
                            end
                        end
                    end)
                end
            end
        end
        for _, child in ipairs(inst:GetChildren()) do
            updateInstanceColors(child)
        end
    end
    
    pcall(function()
        local sg = playerGui:FindFirstChild("AlphaHub_V3")
        if sg then updateInstanceColors(sg) end
        if _G.updateLogoImage then
            _G.updateLogoImage(themeName == "Dark")
        end
    end)
    pcall(function()
        local ExploitGui = (gethui and gethui()) or game:GetService("CoreGui")
        local sg = ExploitGui:FindFirstChild("XiPriorityAlertTest")
        if sg then updateInstanceColors(sg) end
    end)
    for _, name in ipairs({"ALPHA_RemoteSell", "ALPHA_StealProgressBar", "XiAdminPanel"}) do
        pcall(function()
            local otherSg = playerGui:FindFirstChild(name)
            if otherSg then updateInstanceColors(otherSg) end
        end)
    end
    
    -- Force specific buttons that must always have white text to keep Color3.new(1,1,1)
    local function forceWhiteText(inst)
        if inst:IsA("TextButton") then
            local txt = inst.Text
            if txt == "ON" or txt == "OFF" or txt == "ADD" or txt == "X" or txt == "▲" or txt == "▼" then
                inst.TextColor3 = Color3.new(1, 1, 1)
            elseif inst.Size == UDim2.new(0, 50, 0, 20) then -- Keybind bind button
                inst.TextColor3 = Color3.new(1, 1, 1)
            end
        elseif inst:IsA("Frame") and inst.Name == "WhiteSliderKnob" then
            inst.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        end
        for _, child in ipairs(inst:GetChildren()) do
            forceWhiteText(child)
        end
    end
    
    pcall(function()
        local sg = playerGui:FindFirstChild("AlphaHub_V3")
        if sg then forceWhiteText(sg) end
    end)
    
    if Config then
        Config.DarkMode = (themeName == "Dark")
        if saveConfig then saveConfig() end
    end
    pcall(function() if rebuildActions then rebuildActions() end end)
    pcall(function() if rebuildActionSettings then rebuildActionSettings() end end)
    pcall(function() if rebuildTpSpeedSettings then rebuildTpSpeedSettings() end end)
    pcall(function() if loadTab then loadTab(UI.CurrentTab) end end)
end
_G.applyTheme = applyTheme

local UI = {Locked=false, OpenMenuKey=Enum.KeyCode.LeftControl, CurrentTab="Auto TP"}

Keybinds = {
    Kick="Y",["Rejoin Job ID"]="J",Clone="F",["Manual TP"]="T",["Invisible Steal"]="U",
    ["Job ID"]="K",Proximity="P",["Carpet Boost"]="Q",["Open Menu"]="LeftControl",
    ["Ragdoll Self"]="R",["Drop Brainrot"]="G",Float="Z",Reset="X",["Auto Buy"]="K",
    ["Click to AP"]="NONE"
}

priorityList = {
    "Strawberry Elephant",
    "Meowl",
    "Skibidi Toilet",
    "Headless Horseman",
    "Dragon Gingerini",
    "Dragon Cannelloni",
    "Ketupat Bros",
    "Hydra Dragon Cannelloni",
    "La Supreme Combinasion",
    "Love Love Bear",
    "Ginger Gerat",
    "Cerberus",
    "Capitano Moby",
    "La Casa Boo",
    "Burguro and Fryuro",
    "Spooky and Pumpky",
    "Cooki and Milki",
    "Rosey and Teddy",
    "Popcuru and Fizzuru",
    "Reinito Sleighito",
    "Fragrama and Chocrama",
    "Garama and Madundung",
    "Ketchuru and Musturu",
    "La Secret Combinasion",
    "Tralaledon",
    "Tictac Sahur",
    "Ketupat Kepat",
    "Tang Tang Keletang",
    "Orcaledon",
    "La Ginger Sekolah",
    "Los Spaghettis",
    "Lavadorito Spinito",
    "Swaggy Bros",
    "La Taco Combinasion",
    "Los Primos",
    "Los Chillis",
    "Chillin Chili",
    "Tuff Toucan",
    "W or L",
    "Chipso and Queso",
    "Signore Carapace",
    "Arcadragon",
    "John Pork",
    "Elefanto Frigo",
    "Antonio",
    "Pancake and Syrup",
    "Griffin",
    "Kalika Bros",
    "Globa Steppa",
    "Fishino Clownino",
    "Rico Dinero",
    "Tirilikalika Tirilikalako",
    "Digi Narwhal",
    "Hydra Bunny",
    "Dug dug dug",
    "Bunny and Eggy",
    "Los Hackers",
    "Duggy Bros",
    "Guest 666",
    "Money Money Reindeer",
    "Foxini Lanternini",
    "Fragola La La La",
    "Quackini Snackini",
    "Los Sekolahs",
    "Los Tacoritas",
    "Los Amigos",
    "Fortunu and Cashuru",
    "Jolly Jolly Sahur",
    "Boppin Bunny",
    "Gym Bros",
    "Los Cupids",
    "Festive 67",
    "Celularcini Viciosini",
    "Cloverat Clapat",
    "La Food Combinasion",
    "Hopilikalika Hopilikalako",
    "Celestial Pegasus",
    "Sammyni Fattini",
    "Money Money Bros",
    "La Spooky Grande",
    "Cash or Card",
    "Swag Soda",
    "Los Planitos",
    "Lovin Rose",
    "Tacorita Bicicleta",
    "Los Jolly Combinasionas",
    "La Romantic Grande",
    "La Easter Grande",
    "Los Hotspotsitos",
    "Rosetti Tualetti",
    "Los Bros",
    "Gobblino Uniciclino",
    "Chicleteira Cupideira",
    "La Extinct Grande",
    "Las Sis",
    "Nacho Spyder",
    "Gold Gold Gold",
    "Los Mariachis",
    "Snailo Clovero",
    "La Jolly Grande",
    "Los Candies",
    "Churrito Bunnito",
    "Bananito",
    "Eviledon",
    "Los 67",
    "Los Sweethearts",
    "Noo my Heart",
    "La Lucky Grande",
    "Ventoliero Pavonero",
    "Baskito",
    "Chimnino",
    "Los Puggies",
    "Camera Ramena",
    "Los 25",
    "Spinny Hammy",
    "Money Money Puggy",
    "Cigno Fulgoro",
    "Los Spooky Combinasionas",
    "Chicleteira Noelteira",
    "Mariachi Corazoni",
    "Tacorillo Crocodillo",
    "Noo my Gold",
    "Los Mobilis",
    "Mieteteira Bicicleteira",
    "DJ Panda",
    "Los Combinasionas",
    "Nuclearo Dinossauro",
    "Bacuru and Egguru",
    "Spaghetti Tualetti",
    "La Grande Combinasion",
    "Esok Sekolah"
}

actionConfig = {
    ["Ragdoll Self (R)"]=true,["Rejoin PS"]=true,["Rejoin Job ID (J)"]=true,
    ["Kick (Y)"]=true,["Kick To Private"]=true,["Reset (X)"]=true,
    ["Anti Ragdoll"]=false,["Infinite Jump"]=false,["Float"]=false,["Carpet Speed"]=false,
}

-- CONFIG (merged)
local CONFIG_FILE = "alpha_hub_v3_config.json"
local PS_CODE_FILE = "alpha_hub_pscode.txt"
PrivateServerCode = ""

local function loadPSCode()
    pcall(function() if typeof(readfile)=="function" and typeof(isfile)=="function" and isfile(PS_CODE_FILE) then PrivateServerCode = readfile(PS_CODE_FILE) end end)
end
local function savePSCode()
    pcall(function() if typeof(writefile)=="function" then writefile(PS_CODE_FILE, PrivateServerCode or "") end end)
end
loadPSCode()

Config = {
    positions={},keybinds={},actions={},locked=false,
    DarkMode=false,
    AntiRagdoll=false,InfiniteJump=false,Float=false,
    AutoResetBalloon=false,AutoKickOnSteal=false,KickToPrivateServer=false,CleanErrorGUIs=false,
    LineToBase=false,LineToBrainrot=false,InvisStealAngle=225,SinkSliderValue=7,
    AutoRecoverLagback=true,
    WalkSpeedEnabled=false, WalkSpeedValue=16,
    AutoTPPriority=true, AutoTPHighestGen=false, AutoTPHighestValue=false, FPSBoost=false, FPSBoostUltra=false, XRay=false, FOV=70,
    BrainrotESP=true, TimerESP=false, SubspaceMineESP=false, PlayerESP=true, BaseOwnerESP=false,
    AutoBuyEnabled=false, AutoBuyRange=17, AutoGrabSpeed=17, AutoBuyKey="K",
    AutoDestroyTurrets=false, AutoUnlockOnSteal=false,
    AutoInvisDuringSteal=false,
    ClickToAP=false, ClickToAPSingleCommand=false,
    ClickToAPRadius=8,
    SpamBaseOwnerCommands={balloon=true, inverse=true, jail=true, jumpscare=true, morph=true, nightvision=true, ragdoll=true, rocket=true, tiny=true},
    SpamBaseOwnerOrder={"balloon", "inverse", "jail", "jumpscare", "morph", "nightvision", "ragdoll", "rocket", "tiny"},
    SpamBaseOwnerSingleCommand=false,
    ProximityAP=false, ShowJobJoiner=true, AntiBeeDisco=false,
    RemoteSellEnabled=false, AdminPanelUI=true,
    StealHighest=true, StealPriority=false, StealNearest=false,
    AutoStealEnabled=true,
    Unwalk=false,
    Visibilities = {
        ["Invisible Steal Panel"] = true,
        ["Admin Command Panel"] = true,
        ["Command Cooldowns"] = true,
        ["Actions"] = true,
        ["Steal Panel"] = true,
        ["Steal Target"] = true,
    },
    TpSettings = {
        Tool="Flying Carpet", TpKey="T", CloneKey="V", CarpetSpeedKey="Q",
        InfiniteJump=false, DelayVal=0.4, CloneDelayVal=0.1,
        RagdollTP=false, FPSWait=false, FlyTP=false, FlyTPSpeed=160, FlyTPCloseSpeed=75,
        GrabbleTP=false, GrabbleTPSpeed=230,
        TpOnLoad=false, MinGenForTp="", MinGenForGrab="",
        BrainrotCarpet=false,
    },
    PriorityList=priorityList,
    RemovedFromPriority={},
}

local suffixes = {
    k = 1e3,
    m = 1e6,
    b = 1e9,
    t = 1e12,
    q = 1e15,
    qi = 1e18,
    qd = 1e18,
    qn = 1e18,
    sx = 1e21,
    sp = 1e24,
    oc = 1e27,
    no = 1e30,
    dc = 1e33,
    ud = 1e36,
    dd = 1e39,
    td = 1e42,
    qad = 1e45,
    qid = 1e48,
    sxd = 1e51,
    spd = 1e54,
    ocd = 1e57,
    nod = 1e60,
    vg = 1e63,
}

local function parseMinGen(str)
    if not str or type(str) ~= "string" then return 0 end
    str = str:gsub("%s", ""):lower():gsub("/s$", "")
    if str == "" then return 0 end
    local numStr, suffix = str:match("^([%d%.]+)(%a*)$")
    if not numStr then return 0 end
    local num = tonumber(numStr)
    if not num or num < 0 then return 0 end
    if suffix ~= "" then
        local mult = suffixes[suffix]
        if mult then
            return num * mult
        end
    end
    return num
end

local function canUseFiles() return typeof(readfile)=="function" and typeof(writefile)=="function" and typeof(isfile)=="function" end
loadConfig = function()
    if not canUseFiles() then return end
    local ok,data=pcall(function() if isfile(CONFIG_FILE) then return HttpService:JSONDecode(readfile(CONFIG_FILE)) end end)
    if ok and type(data)=="table" then
        for k,v in pairs(data) do
            if k == "PriorityList" or k == "RemovedFromPriority" then
                Config[k] = v
            elseif type(v) == "table" and type(Config[k]) == "table" then
                for subk, subv in pairs(v) do
                    Config[k][subk] = subv
                end
            else
                Config[k] = v
            end
        end
        if type(Config.positions)~="table" then Config.positions={} end
        if type(Config.keybinds)~="table" then Config.keybinds={} end
        if type(Config.actions)~="table" then Config.actions={} end
        if type(Config.RemovedFromPriority)~="table" then Config.RemovedFromPriority={} end
        local removedSet = {}
        for _, rn in ipairs(Config.RemovedFromPriority) do removedSet[rn] = true end
        if type(Config.PriorityList)=="table" then
            if #Config.PriorityList == 0 then
                Config.PriorityList = priorityList
            else
                local present = {}
                for _, name in ipairs(Config.PriorityList) do
                    present[name] = true
                end
                for _, name in ipairs(priorityList) do
                    if not present[name] and not removedSet[name] then
                        table.insert(Config.PriorityList, name)
                    end
                end
                priorityList = Config.PriorityList
            end
        else
            Config.PriorityList = priorityList
        end
        -- Deduplicate priority list
        if type(Config.PriorityList) == "table" then
            local seen = {}
            local cleanList = {}
            for _, name in ipairs(Config.PriorityList) do
                if not seen[name] then
                    seen[name] = true
                    table.insert(cleanList, name)
                end
            end
            Config.PriorityList = cleanList
            priorityList = cleanList
        end
    end
end
saveConfig = function()
    if not canUseFiles() then return end
    task.spawn(function() pcall(function() writefile(CONFIG_FILE, HttpService:JSONEncode(Config)) end) end)
end

-- BASE64 ENCODER/DECODER FOR CONFIG SHARING (Wrapped in do-scope to save global registers)
do
    local b64chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    local function base64Encode(data)
        local len = #data
        local t = {}
        for i = 1, len, 3 do
            local a = data:byte(i)
            local b = data:byte(i + 1) or 0
            local c = data:byte(i + 2) or 0
            local n = a * 65536 + b * 256 + c
            t[#t + 1] = b64chars:sub(math.floor(n / 262144) + 1, math.floor(n / 262144) + 1)
            t[#t + 1] = b64chars:sub(math.floor(n / 4096) % 64 + 1, math.floor(n / 4096) % 64 + 1)
            t[#t + 1] = (i + 1 <= len) and b64chars:sub(math.floor(n / 64) % 64 + 1, math.floor(n / 64) % 64 + 1) or "="
            t[#t + 1] = (i + 2 <= len) and b64chars:sub(n % 64 + 1, n % 64 + 1) or "="
        end
        return table.concat(t)
    end

    local function base64Decode(data)
        data = data:gsub('[^' .. b64chars .. '=]', '')
        local len = #data
        local t = {}
        local lookup = {}
        for i = 1, 64 do lookup[b64chars:sub(i, i)] = i - 1 end
        local i = 1
        while i <= len do
            local c1 = lookup[data:sub(i, i)]
            local c2 = lookup[data:sub(i + 1, i + 1)]
            local s3 = data:sub(i + 2, i + 2)
            local s4 = data:sub(i + 3, i + 3)
            local c3 = lookup[s3]
            local c4 = lookup[s4]
            if not c1 or not c2 then break end
            local n = c1 * 262144 + c2 * 4096 + (c3 or 0) * 64 + (c4 or 0)
            t[#t + 1] = string.char(math.floor(n / 65536))
            if s3 ~= "=" then
                t[#t + 1] = string.char(math.floor(n / 256) % 256)
            end
            if s4 ~= "=" then
                t[#t + 1] = string.char(n % 256)
            end
            i = i + 4
        end
        return table.concat(t)
    end

    _G.importConfig = function(str)
        if not str or str == "" then
            ShowNotification("IMPORT ERROR", "Config string is empty")
            return false
        end
        
        local success, parsed = pcall(function()
            return HttpService:JSONDecode(str)
        end)
        
        if not success or type(parsed) ~= "table" then
            local decoded
            pcall(function() decoded = base64Decode(str:gsub("%s", "")) end)
            if decoded then
                success, parsed = pcall(function() return HttpService:JSONDecode(decoded) end)
            end
        end

        if not success or type(parsed) ~= "table" then
            ShowNotification("IMPORT ERROR", "Invalid config data")
            return false
        end

        for k, v in pairs(parsed) do
            Config[k] = v
        end
        
        if type(Config.PriorityList) == "table" then priorityList = Config.PriorityList end
        
        saveConfig()

        for k, v in pairs(Config.keybinds or {}) do
            if Keybinds[k] ~= nil and type(v) == "string" then
                Keybinds[k] = v
                if k == "Open Menu" and Enum.KeyCode[v] then
                    UI.OpenMenuKey = Enum.KeyCode[v]
                end
            end
        end

        for k, v in pairs(Config.actions or {}) do
            if actionConfig[k] ~= nil then
                actionConfig[k] = v and true or false
            end
        end

        if type(Config.locked) == "boolean" then
            UI.Locked = Config.locked
        end

        initToggles()

        if setFPSBoost then pcall(setFPSBoost, Config.FPSBoost) end
        if setFPSBoostUltra then pcall(setFPSBoostUltra, Config.FPSBoostUltra) end
        if setXRay then pcall(setXRay, Config.XRay) end
        if setInfiniteJump then pcall(setInfiniteJump, Config.InfiniteJump) end
        if setFloat then pcall(setFloat, Config.Float) end
        if setCarpetSpeed then pcall(setCarpetSpeed, Config.CarpetSpeed or false) end
        -- ProximityAP always starts OFF, never loaded from config
    Config.ProximityAP = false
        if toggleAutoBuy then pcall(toggleAutoBuy, Config.AutoBuyEnabled) end
        if setStealMode then
            if Config.StealHighest then pcall(setStealMode, "Highest")
            elseif Config.StealPriority then pcall(setStealMode, "Priority")
            elseif Config.StealNearest then pcall(setStealMode, "Nearest")
            end
        end
        if updateMovementPanelLabels then pcall(updateMovementPanelLabels) end

        rebuildActions()
        rebuildActionSettings()
        loadTab(UI.CurrentTab)

        ShowNotification("CONFIG SYSTEM", "Config imported successfully!")
        return true
    end

    _G.exportConfig = function()
        local ok, str = pcall(function()
            return HttpService:JSONEncode(Config)
        end)
        if not ok or not str then
            ShowNotification("EXPORT ERROR", "Failed to encode config")
            return nil
        end
        local cbSuccess = false
        if typeof(setclipboard) == "function" then
            pcall(function() setclipboard(str); cbSuccess = true end)
        elseif typeof(toclipboard) == "function" then
            pcall(function() toclipboard(str); cbSuccess = true end)
        end
        if cbSuccess then
            ShowNotification("CONFIG SYSTEM", "Config copied to clipboard!")
        else
            ShowNotification("CONFIG SYSTEM", "Config generated! Copy from share box.")
        end
        return str
    end
end
local function serializePos(pos) return {xs=pos.X.Scale,xo=pos.X.Offset,ys=pos.Y.Scale,yo=pos.Y.Offset} end
local function rememberPosition(name, frame) if not name or not frame then return end; Config.positions[name]=serializePos(frame.Position); saveConfig() end
local function applySavedPosition(name, frame)
    if not name or not frame then return end; local d=Config.positions and Config.positions[name]
    if d then frame.Position=UDim2.new(d.xs or 0,d.xo or 0,d.ys or 0,d.yo or 0) end
end
local function initToggles()
    setToggle("Anti Ragdoll", Config.AntiRagdoll, true)
    setToggle("Auto Reset Balloon", Config.AutoResetBalloon, true)
    setToggle("Infinite Jump", Config.InfiniteJump, true)
    setToggle("Auto Kick", Config.AutoKickOnSteal, true)
        setToggle("Auto Buy", Config.AutoBuyEnabled, true)
    setToggle("Auto Steal", Config.AutoStealEnabled, true)
    setToggle("Steal Highest", Config.StealHighest, true)
    setToggle("Steal Priority", Config.StealPriority, true)
    setToggle("Steal Nearest", Config.StealNearest, true)
    setToggle("Click to AP", Config.ClickToAP, true)
    setToggle("ClickToAP", Config.ClickToAP, true)
    setToggle("Click AP Single Cmd", Config.ClickToAPSingleCommand, true)
    setToggle("ClickToAPSingle", Config.ClickToAPSingleCommand, true)
    setToggle("FPS Boost (normal)", Config.FPSBoost, true)
    setToggle("FPS Boost (normal)", Config.FPSBoost, true)
    setToggle("FPS Boost Ultra", Config.FPSBoostUltra, true)
    setToggle("FPSBoostUltra", Config.FPSBoostUltra, true)
    setToggle("XRay", Config.XRay, true)
    setToggle("X-Ray", Config.XRay, true)
    setToggle("Xray", Config.XRay, true)
    setToggle("Proximity", Config.ProximityAP, true)

    setToggle("Player ESP", Config.PlayerESP, true)
    setToggle("Brainrot ESP", Config.BrainrotESP, true)
    setToggle("Timer ESP", Config.TimerESP, true)
    setToggle("Subspace Mine ESP", Config.SubspaceMineESP, true)
    setToggle("Base Owner ESP", Config.BaseOwnerESP, true)
    setToggle("Float", Config.Float, true)
    setToggle("Anti-Bee & Anti-Disco", Config.AntiBeeDisco, true)
    setToggle("AntiBeeDisco", Config.AntiBeeDisco, true)
    setToggle("Admin Panel UI", Config.AdminPanelUI, true)
    setToggle("Auto Invis During Steal", Config.AutoInvisDuringSteal, true)
    setToggle("Auto TP Priority Mode", Config.AutoTPPriority, true)
    setToggle("Auto TP Highest Gen", Config.AutoTPHighestGen, true)
    setToggle("Auto TP Highest Value", Config.AutoTPHighestValue, true)
    setToggle("Unwalk", Config.Unwalk, true)
    setToggle("Stealing ESP", Config.StealingESP, true)
    setToggle("WalkSpeed", Config.WalkSpeedEnabled, true)
    setToggle("Dark Mode", Config.DarkMode, true)
    setToggle("DarkMode", Config.DarkMode, true)
    setToggle("Grabble TP", Config.TpSettings.GrabbleTP or false, true)
end

loadConfig()
if Config.DarkMode then
    for k, v in pairs(Themes.Dark) do
        Theme[k] = v
    end
end
for k,v in pairs(Config.keybinds or {}) do if Keybinds[k]~=nil and type(v)=="string" then Keybinds[k]=v; if k=="Open Menu" and Enum.KeyCode[v] then UI.OpenMenuKey=Enum.KeyCode[v] end end end
for k,v in pairs(Config.actions or {}) do if actionConfig[k]~=nil then actionConfig[k]=v and true or false end end
if type(Config.locked)=="boolean" then UI.Locked=Config.locked end
initToggles()

-- SYNCHRONIZER DETECTION BYPASS + STEALTH CHANNEL READER 
pcall(function()
    local getupvalue = getupvalue or debug.getupvalue

    local function getInternalTable()
        local Packages = ReplicatedStorage:FindFirstChild("Packages")
        if not Packages then return nil end
        local SynMod = Packages:FindFirstChild("Synchronizer")
        if not SynMod then return nil end
        local ok, syn = pcall(require, SynMod)
        if not ok or not syn then return nil end
        local Get = syn.Get
        if type(Get) ~= "function" then return nil end
        for i = 1, 5 do
            local s, u = pcall(getupvalue, Get, i)
            if s and type(u) == "table" then
                if u.___private or u.___channels or u.___data then return u end
                for k, v in pairs(u) do
                    if (type(k) == "string" and k:match("^Plot_")) or type(v) == "table" then
                        return u
                    end
                end
            end
        end
        local s, e = pcall(getfenv, Get)
        if s and e and e.self then return e.self end
        return nil
    end

    local SyncInt = {_cache = {}, _data = nil}
    _G.SyncInt = SyncInt

    task.spawn(function()
        for i = 1, 10 do
            SyncInt._data = getInternalTable()
            if SyncInt._data then break end
            task.wait(1)
        end
    end)

    local function myCustomGet(self, prop)
        if self[prop] then return self[prop] end
        for _, sub in ipairs({"CacheTable", "Data", "_data", "state", "values"}) do
            if type(self[sub]) == "table" and self[sub][prop] then
                return self[sub][prop]
            end
        end
        local alts = {
            Owner = {"owner", "Owner", "plotOwner", "PlotOwner"},
            AnimalList = {"animalList", "AnimalList", "animals", "Animals", "pets"},
        }
        if alts[prop] then
            for _, a in ipairs(alts[prop]) do
                if self[a] then return self[a] end
                for _, sub in ipairs({"CacheTable", "Data", "_data", "state", "values"}) do
                    if type(self[sub]) == "table" and self[sub][a] then
                        return self[sub][a]
                    end
                end
            end
        end
        return nil
    end

    function _G.stealthGet(n)
        if not n or type(n) ~= "string" then return nil end
        if SyncInt._cache[n] == false then return nil end
        local res = nil
        if SyncInt._data then
            for _, k in ipairs({n, "Plot_" .. n, "Plot" .. n, n .. "_Channel", "Channel_" .. n}) do
                if SyncInt._data[k] then
                    res = SyncInt._data[k]
                    break
                end
            end
            if not res then
                for k, v in pairs(SyncInt._data) do
                    if type(k) == "string" and (k == n or k:find(n, 1, true)) and type(v) == "table" then
                        res = v
                        break
                    end
                end
            end
        end
        if res and type(res) == "table" then
            if type(res.Get) ~= "function" then
                rawset(res, "Get", myCustomGet)
            end
            SyncInt._cache[n] = res
            return res
        end
        SyncInt._cache[n] = false
        return nil
    end

    function _G.sProp(ch, p)
        if not ch or type(ch) ~= "table" then return nil end
        if ch[p] then return ch[p] end
        for _, sub in ipairs({"CacheTable", "Data", "_data", "state", "values"}) do
            if type(ch[sub]) == "table" and ch[sub][p] then
                return ch[sub][p]
            end
        end
        if type(ch.Get) == "function" and ch.Get ~= myCustomGet then
            local ok, r = pcall(ch.Get, ch, p)
            if ok then return r end
        end
        local alts = {
            Owner = {"owner", "Owner", "plotOwner", "PlotOwner"},
            AnimalList = {"animalList", "AnimalList", "animals", "Animals", "pets"},
        }
        if alts[p] then
            for _, a in ipairs(alts[p]) do
                if ch[a] then return ch[a] end
                for _, sub in ipairs({"CacheTable", "Data", "_data", "state", "values"}) do
                    if type(ch[sub]) == "table" and ch[sub][a] then
                        return ch[sub][a]
                    end
                end
            end
        end
        return nil
    end

    -- Setup value modification bypass
    local Packages = ReplicatedStorage:FindFirstChild("Packages")
    local SynMod = Packages and Packages:FindFirstChild("Synchronizer")
    local okReq, syn = pcall(require, SynMod)
    if okReq and typeof(syn) == "table" then
        local function HasBoolUpvalue(Fn)
            local OkU, Ups = xpcall(debug.getupvalues, function() end, Fn)
            if not OkU then return false end
            for _, V in pairs(Ups) do
                if typeof(V) == "boolean" then return true end
            end
            return false
        end
        for _, Fn in pairs(syn) do
            if typeof(Fn) == "function" and not isexecutorclosure(Fn) then
                local OkU, Ups = xpcall(debug.getupvalues, function() end, Fn)
                if OkU then
                    for Idx, V in pairs(Ups) do
                        if typeof(V) == "function" and not isexecutorclosure(V) and HasBoolUpvalue(V) then
                            pcall(debug.setupvalue, Fn, Idx, newcclosure(function() end))
                        end
                    end
                end
            end
        end

        -- Hook Get method directly to route through stealthGet
        -- local oldGet = syn.Get
        -- if oldGet then
        --     syn.Get = function(self, plotName)
        --         local ch = _G.stealthGet(plotName)
        --         if ch then return ch end
        --         return oldGet(self, plotName)
        --     end
        -- end
    end
end)




-- Decrypted net
local Decrypted=setmetatable({},{__index=function(S,ez)
    local ok,Netty=pcall(function() return ReplicatedStorage.Packages.Net end); if not ok or not Netty then return nil end
    if ez:sub(1,3)~="RE/" and ez:sub(1,3)~="RF/" then return nil end
    local Remote; for i,v in ipairs(Netty:GetChildren()) do if v.Name==ez then local children=Netty:GetChildren(); Remote=children[i+1]; break end end
    if Remote and not rawget(Decrypted,ez) then rawset(Decrypted,ez,Remote) end; return rawget(Decrypted,ez)
end})

-- COOLDOWN TRACKER
ACTION_COOLDOWNS = {ragdoll=30,jail=60,rocket=120,balloon=30,inverse=30,jumpscare=30,tiny=30,morph=30,nightvision=30}
local lastActionUse = {}

-- Read real timer from the game's AdminPanel UI
local function _readRealAdminTimer(cmd)
    local realAdminGui = playerGui:FindFirstChild("AdminPanel")
    if not realAdminGui then return nil end
    local ok, contentScroll = pcall(function() return realAdminGui.AdminPanel.Content.ScrollingFrame end)
    if not ok or not contentScroll then return nil end
    local cmdBtn = contentScroll:FindFirstChild(cmd)
    if not cmdBtn then return nil end
    local timerLabel = cmdBtn:FindFirstChild("Timer")
    if not timerLabel or not timerLabel.Visible then return 0 end
    local num = tonumber(timerLabel.Text:match("%d+"))
    return num or 0
end

local function apIsOnCooldown(cmd)
    local realTime = _readRealAdminTimer(cmd)
    if realTime ~= nil then return realTime > 0 end
    local l=lastActionUse[cmd]; local cd=ACTION_COOLDOWNS[cmd] or 0; return l and cd>0 and (tick()-l)<cd
end
local function apGetRemaining(cmd)
    local realTime = _readRealAdminTimer(cmd)
    if realTime ~= nil then return realTime end
    local l=lastActionUse[cmd]; local cd=ACTION_COOLDOWNS[cmd] or 0; if not l then return 0 end; return math.max(0,cd-(tick()-l))
end
local function apStartCooldown(cmd) lastActionUse[cmd]=tick() end
AP_ALL_COMMANDS={"balloon","inverse","jail","jumpscare","morph","nightvision","ragdoll","rocket","tiny"}
AP_COMMAND_EMOJIS={balloon="🎈",inverse="🔄",jail="🔒",jumpscare="👻",morph="🎭",nightvision="🌙",ragdoll="🌀",rocket="🚀",tiny="🐜"}
if not Config.ClickToAPCommands then
    Config.ClickToAPCommands = {}
    for _, cmd in ipairs(AP_ALL_COMMANDS) do Config.ClickToAPCommands[cmd] = true end
end
if not Config.AdminPanelButtons then
    Config.AdminPanelButtons = {ragdoll=true, jail=true, rocket=true, balloon=true}
end
if not Config.ClickToAPRadius then
    Config.ClickToAPRadius = 8
end
if not Config.SpamBaseOwnerCommands then
    Config.SpamBaseOwnerCommands = {}
    for _, cmd in ipairs(AP_ALL_COMMANDS) do Config.SpamBaseOwnerCommands[cmd] = true end
end
if not Config.SpamBaseOwnerOrder then
    Config.SpamBaseOwnerOrder = {}
    for i, cmd in ipairs(AP_ALL_COMMANDS) do Config.SpamBaseOwnerOrder[i] = cmd end
end
if Config.SpamBaseOwnerSingleCommand == nil then
    Config.SpamBaseOwnerSingleCommand = false
end
setToggle("SpamBaseOwnerSingleCommand", Config.SpamBaseOwnerSingleCommand or false)

_G.apBlacklist = Config.apBlacklist or {}

local function isPlayerBlacklisted(plr)
    if not plr then return false end
    local uid = plr.UserId
    return _G.apBlacklist[uid] == true or _G.apBlacklist[tostring(uid)] == true
end

-- Cached Synchronizer module. Re-requiring + WaitForChild on every call (inside
-- per-player loops) was a major admin-panel lag source; cache it once.
do
    local cached
    function _G.__getSync()
        if cached then return cached end
        local ok, mod = pcall(function()
            local pkgs = ReplicatedStorage:FindFirstChild("Packages")
            return pkgs and require(pkgs:WaitForChild("Synchronizer", 5))
        end)
        if ok then cached = mod end
        return cached
    end
end

-- NOTE: current-base-owner detection lives below as _G.__getCurrentBaseOwnerId,
-- right after getPlayerBaseInfo (it depends on getPlotAtPosition/getPlotOwner,
-- which are declared further down). Every player here owns their own base, so a
-- "who owns ANY base" check would (correctly but uselessly) flag everyone.

local function getPlotOwner(plot)
    if not plot then return nil end
    local Synchronizer = _G.__getSync()
    if Synchronizer then
        local ch = Synchronizer:Get(plot.Name)
        if ch then
            local owner = ch:Get("Owner")
            if owner then
                if typeof(owner) == "Instance" and owner:IsA("Player") then
                    return owner
                elseif type(owner) == "table" and owner.Name then
                    return Players:FindFirstChild(owner.Name)
                elseif type(owner) == "number" then
                    return Players:GetPlayerByUserId(owner)
                end
            end
        end
    end
    local sign = plot:FindFirstChild("PlotSign")
    local textLabel = sign
        and sign:FindFirstChild("SurfaceGui")
        and sign.SurfaceGui:FindFirstChild("Frame")
        and sign.SurfaceGui.Frame:FindFirstChild("TextLabel")
    if textLabel then
        local baseText = textLabel.Text
        local nickname = (baseText and baseText:match("^(.-)'")) or baseText
        if nickname then
            for _, p in ipairs(Players:GetPlayers()) do
                if (p.DisplayName == nickname) or (p.Name == nickname) then
                    return p
                end
            end
        end
    end
    return nil
end

local function getPlotAtPosition(pos)
    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local closestPlot = nil
    local minDistance = math.huge
    for _, plot in ipairs(plots:GetChildren()) do
        local plotPos
        if plot:IsA("Model") then
            plotPos = plot.PrimaryPart and plot.PrimaryPart.Position or plot:GetPivot().Position
        else
            plotPos = plot.Position
        end
        if plotPos then
            local distH = math.sqrt((pos.X - plotPos.X)^2 + (pos.Z - plotPos.Z)^2)
            if distH < minDistance then
                minDistance = distH
                closestPlot = plot
            end
        end
    end
    if closestPlot and minDistance < 72 then
        return closestPlot
    end
    return nil
end

local function getPlayerBaseInfo(plr)
    if not plr or not plr.Character then return nil, nil end
    local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil, nil end
    local plot = getPlotAtPosition(hrp.Position)
    if plot then
        return plot, getPlotOwner(plot)
    end
    return nil, nil
end

-- userId of the owner of the base the LOCAL player is currently standing in
-- (nil if none). Memoized ~0.4s and shared by all admin rows, so only ONE row
-- shows "Base Owner": the owner of the base you're actually in.
do
    local cachedId, lastT = nil, 0
    function _G.__getCurrentBaseOwnerId()
        local now = os.clock()
        if (now - lastT) < 0.4 then return cachedId end
        lastT = now
        cachedId = nil
        local _, owner = getPlayerBaseInfo(LocalPlayer)
        if owner then cachedId = owner.UserId end
        return cachedId
    end
end

local function getStealingInfo(plr)
    if not plr or not plr.Character then return nil, nil end
    local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil, nil end
    
    local isPlayingAnim = false
    local hum = plr.Character:FindFirstChildOfClass("Humanoid")
    local animator = hum and hum:FindFirstChildOfClass("Animator")
    if animator then
        pcall(function()
            for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                local id = track.Animation and track.Animation.AnimationId
                if id and (id:find("18537363391") or id:find("steal") or id:find("grab")) then
                    isPlayingAnim = true
                    break
                end
            end
        end)
    end
    
    local plots = Workspace:FindFirstChild("Plots")
    if plots then
        for _, plot in ipairs(plots:GetChildren()) do
            local owner = getPlotOwner(plot)
            if owner == plr then continue end
            
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, pod in ipairs(podiums:GetChildren()) do
                    local base = pod:FindFirstChild("Base")
                    local spawn = base and base:FindFirstChild("Spawn")
                    if spawn then
                        local dist = (hrp.Position - spawn.Position).Magnitude
                        if (isPlayingAnim and dist < 12) or (dist < 4.5) then
                            local animalName = "Brainrot"
                            pcall(function()
                                local Synchronizer = _G.__getSync()
                                if Synchronizer then
                                    local ch = Synchronizer:Get(plot.Name)
                                    if ch then
                                        local al = ch:Get("AnimalList")
                                        local ad = al and (al[pod.Name] or al[tonumber(pod.Name)])
                                        if ad and type(ad) == "table" then
                                            local ok2, Datas = pcall(function() return ReplicatedStorage:FindFirstChild("Datas") end)
                                            local okA, AnimalsData = pcall(function() return require(Datas:FindFirstChild("Animals")) end)
                                            if okA and AnimalsData and AnimalsData[ad.Index] then
                                                animalName = AnimalsData[ad.Index].DisplayName or ad.Index
                                            else
                                                animalName = ad.Index
                                            end
                                        end
                                    end
                                end
                            end)
                            return owner, animalName
                        end
                    end
                end
            end
        end
    end
    
    local attrStealing = plr:GetAttribute("Stealing")
    if attrStealing then
        return nil, plr:GetAttribute("StealingIndex") or "Brainrot"
    end
    
    return nil, nil
end

-- ADMIN COMMAND BRIDGE
local function fireClick(button)
    if not button then return false end
    local ok=pcall(function()
        if typeof(firesignal)=="function" then
            pcall(firesignal, button.MouseButton1Click)
            pcall(firesignal, button.MouseButton1Down)
            pcall(firesignal, button.MouseButton1Up)
            pcall(firesignal, button.Activated)
        else
            local x=button.AbsolutePosition.X+(button.AbsoluteSize.X/2)
            local y=button.AbsolutePosition.Y+(button.AbsoluteSize.Y/2)+58
            VirtualInputManager:SendMouseButtonEvent(x,y,0,true,game,0)
            VirtualInputManager:SendMouseButtonEvent(x,y,0,false,game,0)
        end
    end); return ok
end
_G.fireClick=fireClick

local function runAdminCommand(targetPlayer, commandName)
    if not targetPlayer or not commandName or commandName=="" then return false end
    if isPlayerBlacklisted(targetPlayer) then 
        ShowNotification("BLOCKED", targetPlayer.DisplayName .. " is blacklisted")
        return false 
    end
    local realAdminGui=playerGui:FindFirstChild("AdminPanel")
    if not realAdminGui then realAdminGui=playerGui:WaitForChild("AdminPanel",3) end
    if not realAdminGui then return false end
    local wasEnabled = realAdminGui.Enabled
    realAdminGui.Enabled = true
    local okC,contentScroll=pcall(function() return realAdminGui.AdminPanel.Content.ScrollingFrame end)
    if not okC or not contentScroll then realAdminGui.Enabled=wasEnabled; return false end
    local cmdBtn=contentScroll:FindFirstChild(commandName); if not cmdBtn then realAdminGui.Enabled=wasEnabled; return false end
    fireClick(cmdBtn)
    task.wait(0.01)
    local okP,profilesScroll=pcall(function() return realAdminGui.AdminPanel.Profiles.ScrollingFrame end)
    if not okP or not profilesScroll then realAdminGui.Enabled=wasEnabled; return false end
    local playerBtn=profilesScroll:FindFirstChild(targetPlayer.Name)
    if not playerBtn then
        task.wait(0.01)
        playerBtn=profilesScroll:FindFirstChild(targetPlayer.Name)
    end
    if not playerBtn then
        for _,child in ipairs(profilesScroll:GetChildren()) do
            if child:IsA("GuiButton") then local nl=child:FindFirstChildWhichIsA("TextLabel")
                if nl and (nl.Text==targetPlayer.Name or nl.Text==targetPlayer.DisplayName) then playerBtn=child; break end
            end
        end
    end
    if not playerBtn then realAdminGui.Enabled=wasEnabled; return false end
    fireClick(playerBtn)
    apStartCooldown(commandName)
    task.delay(0.05, function()
        if realAdminGui and realAdminGui.Parent then realAdminGui.Enabled=wasEnabled end
    end)
    return true
end
_G.runAdminCommand=runAdminCommand


local function runAutoBaseActions()
    task.spawn(function()
        task.wait(1.5)
        local char = player.Character or player.CharacterAdded:Wait()
        local hrp = char:WaitForChild("HumanoidRootPart", 10)
        if not hrp then return end
        
        local nearestPlayer = nil
        local nearestDist = math.huge
        local Plots = Workspace:FindFirstChild("Plots")
        if Plots then
            for _, plot in ipairs(Plots:GetChildren()) do
                local owner = getPlotOwner(plot)
                if owner and owner ~= player then
                    local sign = plot:FindFirstChild("PlotSign")
                    if sign then
                        local signPos = (sign:IsA("BasePart") and sign.Position) or (sign.PrimaryPart and sign.PrimaryPart.Position)
                        if signPos then
                            local dist = (hrp.Position - signPos).Magnitude
                            if dist < nearestDist then
                                nearestDist = dist
                                nearestPlayer = owner
                            end
                        end
                    end
                end
            end
        end
        
        if nearestPlayer then
            -- auto base actions removed
        end
    end)
end
_G.runAutoBaseActions = runAutoBaseActions

local function isMobyUser(p) return p and p.Character and p.Character:FindFirstChild("_moby_highlight")~=nil end
local function isKawaifuUser(p) return p and p.Character and p.Character:FindFirstChild("KaWaifu_NeonHighlight")~=nil end

local function getNextAvailableCommand()
    local priorityCmds={"ragdoll","balloon","rocket","jail"}
    for _,cmd in ipairs(priorityCmds) do if not apIsOnCooldown(cmd) then return cmd end end
    for _,cmd in ipairs(AP_ALL_COMMANDS) do if not apIsOnCooldown(cmd) then return cmd end end
    return nil
end

local function kickPlayer(stolenText)
    local isAutoKickSteal = false
    
    if stolenText and type(stolenText) == "string" then
        isAutoKickSteal = true
    end

    if Config.KickToPrivateServer and PrivateServerCode and PrivateServerCode ~= "" and isAutoKickSteal then
        task.delay(0.2, function()
            pcall(function()
                local ExperienceService = game:GetService("ExperienceService")
                ExperienceService:LaunchExperience({
                    placeId = game.PlaceId,
                    linkCode = PrivateServerCode,
                })
            end)
        end)
        return
    end
    pcall(function() game:Shutdown() end)
    pcall(function() LocalPlayer:Kick("\nALPHA HUB PRIVAT") end)
end

-- SHARED STATE
SharedState = {SelectedPetData=nil, AllAnimalsCache={}, ListNeedsRedraw=true, InitialScanComplete=false, seenUIDs={}, BrainrotNames={}}

-- NOTIFICATION SYSTEM
local function ShowNotification(title, text)
    -- Disabled: all notifications removed by user request
    return
end

-- ============================================================
-- RESET
-- ============================================================
local executeReset

do
--[[
    Roblox Instant Reset & Lagback Detector (Defensive Always-Working Hook)
    Created by Antigravity (DeepMind Coding Assistant)
    
    Fixes:
    - Reverted capture-state resetting. Once TargetRemote is captured, it is kept for the entire session.
    - Fixed the reset-spam issue. Resetting state resets immediately upon character removal so you can spam resets.
--]]

-- SERVICES
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

-- CONFIGURATION DEFAULT
local Config = {
    Enabled = true,
    DetectionThreshold = 6,
    HistoryDuration = 2.0,
    VelocityDotThreshold = -0.3,
    
    ResetKey = "NONE",           -- Managed via main Keybinds (default X)
    FloodCount = 50,             -- Number of junk fires to force server lagback reset
    
    ResetMethods = {
        SpamHeartbeat = true,    -- Spam captured ToolActivationController remote
        HumanoidHealth = true,   -- Fallback: Set health to 0
        BreakJoints = true,      -- Fallback: Break joints
        ChangeStateDead = true,  -- Fallback: Force dead state
        DestroyHumanoid = false, -- Fallback: Destroy humanoid
        FireRemotes = true       -- Fallback: Fire detected generic reset remotes
    },
    
    ScanInterval = 15,
    RemoteKeywords = {"reset", "kill", "die", "suicide", "respawn", "clear", "death", "damage", "reavatar"},
    BlacklistedRemotes = {"AdminReset", "StaffReset", "ModReset"},
    
    Debug = false,
    NotifyOnReset = true
}

-- STATE VARIABLES
local TargetRemote = nil -- Dynamically captured heartbeat remote
local rawFS = nil        -- Original unhooked FireServer function
local resetRemotes = {}
local positionHistory = {}
local lastPosition = nil
local lastVelocity = Vector3.new(0, 0, 0)
local isTeleporting = false
local resetting = false
local scanTimer = 0
local lastHistoryUpdate = 0

-- LOGGING UTILITY
local function log(message, isWarn)
    -- Disabled: do not print or warn anything to dev console
    return
end

-- EXPORTED TELEPORT BYPASS API
_G.NotifyLocalTeleport = function(duration)
    local delayTime = duration or 0.5
    isTeleporting = true
    task.delay(delayTime, function()
        isTeleporting = false
    end)
end
shared.NotifyLocalTeleport = _G.NotifyLocalTeleport

-- OPTIMIZED INCREMENTAL REMOTE EVENT SCANNER (For Generic Fallbacks)
local function scanForRemotes()
    table.clear(resetRemotes)
    
    task.spawn(function()
        local searchAreas = {ReplicatedStorage, Workspace}
        local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
        if playerGui then
            table.insert(searchAreas, playerGui)
        end
        
        local count = 0
        for _, area in ipairs(searchAreas) do
            local queue = {area}
            while #queue > 0 do
                local current = table.remove(queue, 1)
                count = count + 1
                if count % 150 == 0 then
                    task.wait()
                end
                
                pcall(function()
                    for _, child in ipairs(current:GetChildren()) do
                        if child:IsA("RemoteEvent") or child:IsA("RemoteFunction") then
                            local nameLower = child.Name:lower()
                            local isMatch = false
                            for _, keyword in ipairs(Config.RemoteKeywords) do
                                if string.find(nameLower, keyword, 1, true) then
                                    isMatch = true
                                    break
                                end
                            end
                            
                            if isMatch then
                                local isBlacklisted = false
                                for _, black in ipairs(Config.BlacklistedRemotes) do
                                    if string.find(child.Name, black, 1, true) then
                                        isBlacklisted = true
                                        break
                                    end
                                end
                                
                                if not isBlacklisted then
                                    table.insert(resetRemotes, child)
                                end
                            end
                        elseif child:IsA("Folder") or child:IsA("Configuration") or child:IsA("Model") or child:IsA("Tool") then
                            table.insert(queue, child)
                        end
                    end
                end)
            end
        end
    end)
end

-- --- DYNAMIC REMOTE INTERCEPTION (ToolActivationController Heartbeat Hook) ---
local function hookHeartbeatRemote()
    local newcc = newcclosure or function(f) return f end
    
    pcall(function()
        local seen = {}
        local capturing = true
        
        local function handleFire(self, ...)
            if not capturing then return end
            if not self:IsA("RemoteEvent") then return end
            
            local a1 = (select("#", ...) >= 1) and (select(1, ...)) or nil
            if typeof(a1) ~= "string" then return end
            
            local nm = self.Name
            if type(nm) ~= "string" or not nm:match("^RE/%x%x%x%x%x%x%x%x") then return end
            
            local callingScript = nil
            pcall(function() callingScript = getcallingscript() end)
            
            local isScriptMatch = false
            if callingScript then
                if callingScript.Name == "ToolActivationController" or string.find(callingScript.Name, "ToolActivationController") then
                    isScriptMatch = true
                end
            else
                isScriptMatch = true
            end
            
            if isScriptMatch then
                local c = (seen[self] or 0) + 1
                seen[self] = c
                if c >= 2 then
                    TargetRemote = self
                    capturing = false
                    seen = nil
                    log("ToolActivationController heartbeat remote captured: " .. self:GetFullName())
                end
            end
        end

        if hookfunction then
            local dummyFunc = function(self, ...) return self end
            rawFS = dummyFunc
            rawFS = hookfunction(Instance.new("RemoteEvent").FireServer, newcc(function(self, ...)
                pcall(handleFire, self, ...)
                return (rawFS ~= dummyFunc and rawFS or Instance.new("RemoteEvent").FireServer)(self, ...)
            end))
            log("Successfully hooked FireServer via hookfunction.")
        elseif hookmetamethod then
            local oldNamecall
            oldNamecall = hookmetamethod(game, "__namecall", newcc(function(self, ...)
                local method = getnamecallmethod()
                if (method == "FireServer" or method == "fireServer") and self:IsA("RemoteEvent") then
                    pcall(handleFire, self, ...)
                end
                return oldNamecall(self, ...)
            end))
            log("Successfully hooked __namecall via hookmetamethod.")
        else
            log("Warning: Executor does not support hookfunction or hookmetamethod.", true)
        end
    end)
end

-- INSTANT RESET (new.lua method) -- spams the captured RE/* reset remote with a
-- junk payload until the character respawns. This is the method that actually
-- works in THIS game (client Health/BreakJoints is server-authoritative and does
-- nothing). Robustness over new.lua: waits briefly for the remote to be captured
-- and time-caps the loop, so the cooldown can never get stuck = reset always fires.
local function instantReset()
    if instaResetCooldown then return end
    instaResetCooldown = true
    local lp = LocalPlayer
    local oldChar = lp.Character
    task.spawn(function()
        local t0 = os.clock()
        -- if the reset remote hasn't been captured yet, wait briefly for it
        while not resetRemote and (os.clock() - t0) < 3 do task.wait() end
        while resetRemote and lp.Character == oldChar and (os.clock() - t0) < 8 do
            pcall(function() resetRemote:FireServer("randomstring") end)
            task.wait()
        end
        instaResetCooldown = false
    end)
end

-- Native Esc -> Reset button: route it through the same working reset (remote spam)
-- instead of the game's default, so the menu reset also works instantly.
local _alphaResetBindable = Instance.new("BindableEvent")
_alphaResetBindable.Event:Connect(function() pcall(instantReset) end)
task.spawn(function()
    for _ = 1, 12 do
        local ok = pcall(function()
            game:GetService("StarterGui"):SetCore("ResetButtonCallback", _alphaResetBindable)
        end)
        if ok then break end
        task.wait(1)
    end
end)

-- KEYBIND TRIGGER (Disabled local listener to avoid collision with 'Drop Brainrot' keybind G)
-- Reset is managed by the main Keybinds setting in the GUI (defaults to key X)

_G.InstantReset = instantReset -- Global access for other scripts

-- HISTORY CACHE MANAGEMENT
local function updateHistory(pos)
    local now = tick()
    if now - lastHistoryUpdate < 0.05 then return end
    lastHistoryUpdate = now
    
    table.insert(positionHistory, {pos = pos, time = now})
    
    while #positionHistory > 0 and now - positionHistory[1].time > Config.HistoryDuration do
        table.remove(positionHistory, 1)
    end
end

-- --- MAIN MONITORING LOOP ---
local function onHeartbeat(dt)
    if resetting then return end
    
    -- Scan remotes periodically in background
    scanTimer = scanTimer + dt
    if scanTimer >= Config.ScanInterval then
        scanTimer = 0
        scanForRemotes()
    end
end

-- INITIALIZATION
scanForRemotes()
-- hookHeartbeatRemote() removed -- it stacked a second FireServer hook on top of
-- the Vanish bypass hook, which broke the chain after re-injects and stopped
-- in-game tool activation from working. The Vanish FireServer hook at the top of
-- the file is the ONLY hook this script installs now.
RunService.Heartbeat:Connect(onHeartbeat)



-- Compatibility wrapper for executeReset
local lastBalloonResetTime = 0
executeReset = function(isBalloon)
    if isBalloon then
        if tick() - lastBalloonResetTime < 20 then return end
        lastBalloonResetTime = tick()
    end
    instantReset()
end
_G.executeReset = executeReset
end

-- CLONE
local function instantClone()
    local c=player.Character; if not c then return end
    local h=c:FindFirstChildOfClass("Humanoid"); if not h then return end
    local cl=player.Backpack:FindFirstChild("Quantum Cloner") or c:FindFirstChild("Quantum Cloner"); if not cl then return end
    pcall(function() h:UnequipTools() end); task.wait()
    if cl.Parent~=c then h:EquipTool(cl); task.wait() end
    local tf=playerGui:FindFirstChild("ToolsFrames"); local qc=tf and tf:FindFirstChild("QuantumCloner")
    local tb=qc and qc:FindFirstChild("TeleportToClone"); if not tb then return end
    _G.isCloning=true; cl:Activate(); task.wait(0.05); tb.Visible=true
    pcall(function() firesignal(tb.MouseButton1Click) end)
    pcall(function() firesignal(tb.MouseButton1Up) end)
    pcall(function() firesignal(tb.Activated) end)
    task.delay(0.55, function() _G.isCloning=false end)
end

-- DROP BRAINROT
local _wfConns,_wfActive={},false
local function stopWalkFling() _wfActive=false; for _,c in ipairs(_wfConns) do if typeof(c)=="RBXScriptConnection" then c:Disconnect() end end; _wfConns={} end
local function startWalkFling()
    _wfActive=true; local ch=player.Character; if not ch then return end
    local rr=ch:FindFirstChild("HumanoidRootPart")
    for _,o in pairs(Workspace.CurrentCamera:GetChildren()) do if o.Name=="HumanoidRootPart" then rr=o; break end end
    if not rr then return end
    table.insert(_wfConns,RunService.Stepped:Connect(function() if not _wfActive then return end
        for _,p in ipairs(Players:GetPlayers()) do if p~=player and p.Character then for _,pt in ipairs(p.Character:GetChildren()) do if pt:IsA("BasePart") then pt.CanCollide=false end end end end
    end))
    local co=coroutine.create(function()
        if _G.invisibleStealEnabled then rr.CFrame=rr.CFrame*CFrame.new(0,3,0) end
        while _wfActive do RunService.Heartbeat:Wait(); if not rr or not rr.Parent then break end
            local v=rr.Velocity; rr.Velocity=v*10000+Vector3.new(0,10000,0)
            RunService.RenderStepped:Wait(); if rr then rr.Velocity=v end
            RunService.Stepped:Wait(); if rr then rr.Velocity=v+Vector3.new(0,0.1,0) end
        end
    end); coroutine.resume(co); table.insert(_wfConns,co)
end
local function runDropBrainrot() if _wfActive then return end; startWalkFling(); task.delay(0.4,stopWalkFling) end

-- ============================================================
-- INVISIBLE STEAL
-- ============================================================
do
local animPlaying = false
local tracks = {}
local clone, oldRoot, hip, connection
local folderConnections = {}
local serverGhosts = {}
local ghostEnabled = true
local lagbackCallCount = 0
local lagbackWindowStart = 0
local lastLagbackTime = 0
local errorOrbActive = false
local errorOrb = nil
local errorOrbConnection = nil

_G.invisibleStealEnabled = false
_G.InvisStealAngle = Config.InvisStealAngle or 225
_G.SinkSliderValue = Config.SinkSliderValue or 7
_G.AutoRecoverLagback = Config.AutoRecoverLagback ~= nil and Config.AutoRecoverLagback or true
_G.AutoInvisDuringSteal = Config.AutoInvisDuringSteal or false

local function clearErrorOrb()
    if errorOrb and errorOrb.Parent then errorOrb:Destroy() end
    errorOrb = nil; errorOrbActive = false
    if errorOrbConnection then errorOrbConnection:Disconnect(); errorOrbConnection = nil end
end

local function createErrorOrb()
    if errorOrbActive then return end
    errorOrbActive = true
    for _, ghost in pairs(serverGhosts) do if ghost and ghost.Parent then ghost:Destroy() end end
    serverGhosts = {}
    -- Error UI notifications disabled
end

local function createServerGhost(position)
    if not ghostEnabled or errorOrbActive then return end
    local now = tick()
    if now - lastLagbackTime < 0.05 then return end
    lastLagbackTime = now
    if now - lagbackWindowStart > 1 then lagbackCallCount = 0; lagbackWindowStart = now end
    lagbackCallCount = lagbackCallCount + 1
    if lagbackCallCount >= 7 then createErrorOrb(); return end
    for _, g in pairs(serverGhosts) do if g and g.Parent then g:Destroy() end end
    serverGhosts = {}
    local ghost = Instance.new("Part")
    ghost.Name = "LagbackGhost"; ghost.Shape = Enum.PartType.Ball
    ghost.Size = Vector3.new(3, 3, 3); ghost.Color = Color3.fromRGB(255, 0, 0)
    ghost.Material = Enum.Material.Glass; ghost.Transparency = 0.3
    ghost.CanCollide = false; ghost.Anchored = true; ghost.CastShadow = false
    ghost.Position = position + Vector3.new(0, 5, 0); ghost.Parent = Workspace.CurrentCamera
    table.insert(serverGhosts, ghost)
    -- Lagback text notifications & billboard GUIs disabled
end

local function clearAllGhosts()
    for _, ghost in pairs(serverGhosts) do pcall(function() if ghost and ghost.Parent then ghost:Destroy() end end) end
    serverGhosts = {}; clearErrorOrb(); lagbackCallCount = 0; lastLagbackTime = 0
    pcall(function()
        local pg = player:FindFirstChild("PlayerGui")
        if pg then for _, gui in pairs(pg:GetChildren()) do if gui.Name == "LagbackNotification" then gui:Destroy() end end end
    end)
    pcall(function() if Workspace.CurrentCamera then for _, c in pairs(Workspace.CurrentCamera:GetChildren()) do if c.Name == "LagbackGhost" then c:Destroy() end end end end)
    pcall(function() for _, c in pairs(Workspace:GetDescendants()) do if c.Name == "LagbackGhost" then c:Destroy() end end end)
end

local function removeFolders()
    local pf = Workspace:FindFirstChild(player.Name)
    if not pf then return end
    local dr = pf:FindFirstChild("DoubleRig")
    if dr then
        local rr = dr:FindFirstChild("HumanoidRootPart") or dr:FindFirstChildWhichIsA("BasePart")
        if rr and ghostEnabled then createServerGhost(rr.Position) end
        dr:Destroy()
    end
    local cs = pf:FindFirstChild("Constraints")
    if cs then cs:Destroy() end
    local conn = pf.ChildAdded:Connect(function(child)
        if child.Name == "DoubleRig" then
            task.defer(function()
                local rr = child:FindFirstChild("HumanoidRootPart") or child:FindFirstChildWhichIsA("BasePart")
                if rr and ghostEnabled then createServerGhost(rr.Position) end
                child:Destroy()
            end)
        elseif child.Name == "Constraints" then child:Destroy() end
    end)
    table.insert(folderConnections, conn)
end

local function doClone()
    local character = player.Character
    if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
        hip = character.Humanoid.HipHeight
        oldRoot = character:FindFirstChild("HumanoidRootPart")
        if not oldRoot or not oldRoot.Parent then return false end
        for _, c in pairs(oldRoot:GetChildren()) do
            if c:IsA("Attachment") and (c.Name:find("Beam") or c.Name:find("Attach")) then c:Destroy() end
        end
        for _, c in pairs(oldRoot:GetChildren()) do if c:IsA("Beam") then c:Destroy() end end
        local tmp = Instance.new("Model"); tmp.Parent = game
        character.Parent = tmp
        clone = oldRoot:Clone(); clone.Parent = character
        oldRoot.Parent = Workspace.CurrentCamera
        clone.CFrame = oldRoot.CFrame; character.PrimaryPart = clone
        character.Parent = Workspace
        for _, v in pairs(character:GetDescendants()) do
            if v:IsA("Weld") or v:IsA("Motor6D") then
                if v.Part0 == oldRoot then v.Part0 = clone end
                if v.Part1 == oldRoot then v.Part1 = clone end
            end
        end
        tmp:Destroy(); return true
    end
    return false
end

local function revertClone()
    local character = player.Character
    if not oldRoot or not oldRoot:IsDescendantOf(Workspace) or not character or character.Humanoid.Health <= 0 then return end
    local tmp = Instance.new("Model"); tmp.Parent = game
    character.Parent = tmp
    oldRoot.Parent = character; character.PrimaryPart = oldRoot
    character.Parent = Workspace; oldRoot.CanCollide = true
    for _, v in pairs(character:GetDescendants()) do
        if v:IsA("Weld") or v:IsA("Motor6D") then
            if v.Part0 == clone then v.Part0 = oldRoot end
            if v.Part1 == clone then v.Part1 = oldRoot end
        end
    end
    if clone then local p = clone.CFrame; clone:Destroy(); clone = nil; oldRoot.CFrame = p end
    oldRoot = nil
    if character and character.Humanoid then character.Humanoid.HipHeight = hip end
    clearAllGhosts()
end

local function animationTrickery()
    local character = player.Character
    if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
        local anim = Instance.new("Animation")
        anim.AnimationId = "http://www.roblox.com/asset/?id=18537363391"
        local humanoid = character.Humanoid
        local animator = humanoid:FindFirstChild("Animator") or Instance.new("Animator", humanoid)
        local animTrack = animator:LoadAnimation(anim)
        animTrack.Priority = Enum.AnimationPriority.Action4
        animTrack:Play(0, 1, 0); anim:Destroy()
        table.insert(tracks, animTrack)
        animTrack.Stopped:Connect(function() if animPlaying then animationTrickery() end end)
        task.delay(0, function()
            animTrack.TimePosition = 0.7
            task.delay(0.3, function() if animTrack then animTrack:AdjustSpeed(math.huge) end end)
        end)
    end
end

local _invisToggleCooldown = 0

local function invisTurnOff()
    clearAllGhosts()
    if not animPlaying then return end
    local character = player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    animPlaying = false; _G.invisibleStealEnabled = false
    setToggle("Invisible Steal", false)
    for _, t in pairs(tracks) do pcall(function() t:Stop(0) end) end
    tracks = {}
    if connection then connection:Disconnect(); connection = nil end
    for _, c in ipairs(folderConnections) do if c then c:Disconnect() end end
    folderConnections = {}
    revertClone(); clearAllGhosts()
    -- Force reset all animations back to default
    if humanoid then
        pcall(function()
            local animator = humanoid:FindFirstChildOfClass("Animator")
            if animator then
                for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                    if track.Priority == Enum.AnimationPriority.Action4 or track.Priority == Enum.AnimationPriority.Action3 then
                        track:Stop(0)
                    end
                end
            end
            humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
            task.defer(function()
                if humanoid and humanoid.Parent then
                    humanoid:ChangeState(Enum.HumanoidStateType.Running)
                end
            end)
        end)
    end
    if _G.updateMovementPanelInvisVisual then pcall(_G.updateMovementPanelInvisVisual, false) end
    -- Instantly disable walkspeed when invis steal turns off
    if WalkSpeedState and WalkSpeedState.enabled and Config.WalkSpeedEnabled then
        setWalkSpeedEnabled(false)
    end
    _invisToggleCooldown = tick()
end

local function invisTurnOn()
    if animPlaying then return end
    local character = player.Character
    if not character then return end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    animPlaying = true; _G.invisibleStealEnabled = true
    setToggle("Invisible Steal", true)
    if _G.updateMovementPanelInvisVisual then pcall(_G.updateMovementPanelInvisVisual, true) end
    tracks = {}; removeFolders()
    local success = doClone()
    if success then
        task.wait(0.05); animationTrickery()
        task.defer(function()
            if _G.resetBrainrotBeam then pcall(_G.resetBrainrotBeam) end
            if _G.resetPlotBeam then pcall(_G.resetPlotBeam) end
            task.wait(0.1)
            if _G.updateBrainrotBeam then pcall(_G.updateBrainrotBeam) end
            if _G.createPlotBeam then pcall(_G.createPlotBeam) end
        end)
        -- Enable walkspeed 1 second after invis steal activates
        task.delay(1, function()
            if _G.invisibleStealEnabled and not WalkSpeedState.enabled then
                setWalkSpeedEnabled(true)
            end
        end)
        local lastSetPosition = nil; local skipFrames = 5
        connection = RunService.PreSimulation:Connect(function()
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 and oldRoot then
                local root = character.PrimaryPart or character:FindFirstChild("HumanoidRootPart")
                if root then
                    if skipFrames > 0 then skipFrames = skipFrames - 1; lastSetPosition = nil
                    elseif lastSetPosition and ghostEnabled then
                        local currentPos = oldRoot.Position
                        local jumpDist = (currentPos - lastSetPosition).Magnitude
                        if jumpDist > 6 and not _G.RecoveryInProgress and player:GetAttribute("Stealing") then
                            lastSetPosition = nil; createServerGhost(currentPos)
                            if _G.AutoRecoverLagback and _G._forceInvisToggle then
                                _G.RecoveryInProgress = true
                                task.spawn(function()
                                    pcall(_G._forceInvisToggle); task.wait(0.6)
                                    if player:GetAttribute("Stealing") then
                                        pcall(_G._forceInvisToggle)
                                    end
                                    _G.RecoveryInProgress = false
                                end)
                            end
                        end
                    end
                    if clone then clone.CanCollide = true end
                    if oldRoot and oldRoot.Parent then
                        for _, c in pairs(oldRoot:GetChildren()) do
                            if c:IsA("Attachment") or c:IsA("Beam") then c:Destroy() end
                        end
                        local sa = (_G.SinkSliderValue or 7) * 0.5
                        local cf = root.CFrame - Vector3.new(0, sa, 0)
                        oldRoot.CFrame = cf * CFrame.Angles(math.rad(_G.InvisStealAngle or 225), 0, 0)
                        oldRoot.AssemblyLinearVelocity = root.AssemblyLinearVelocity; oldRoot.CanCollide = false
                        lastSetPosition = oldRoot.Position
                    end
                end
            end
        end)
    end
end

_G.toggleInvisibleSteal = function()
    if (tick() - _invisToggleCooldown) < 0.3 then return end
    if animPlaying then invisTurnOff() else invisTurnOn() end
end

-- Force toggle that bypasses debounce (for auto-recover and auto-invis)
_G._forceInvisToggle = function()
    if animPlaying then invisTurnOff() else invisTurnOn() end
end

player.CharacterAdded:Connect(function(newChar)
    task.wait(0.1)
    if Config then Config.ClickToAP = false end
    clearErrorOrb(); clearAllGhosts(); lagbackCallCount = 0
    pcall(function() for _, c in pairs(Workspace.CurrentCamera:GetChildren()) do if c:IsA("BasePart") and c.Name == "HumanoidRootPart" then c:Destroy() end end end)
    if oldRoot then pcall(function() oldRoot:Destroy() end); oldRoot = nil end
    if clone then pcall(function() clone:Destroy() end); clone = nil end
    animPlaying = false; _G.invisibleStealEnabled = false
    setToggle("Invisible Steal", false)
    if _G.updateMovementPanelInvisVisual then pcall(_G.updateMovementPanelInvisVisual, false) end
    task.wait(0.2)
    local camera = Workspace.CurrentCamera
    if camera and newChar then
        local h = newChar:FindFirstChildOfClass("Humanoid")
        if h then camera.CameraSubject = h; camera.CameraType = Enum.CameraType.Custom end
    end
end)

local function setupDeathListener()
    local ch = player.Character
    if ch then
        local h = ch:FindFirstChildOfClass("Humanoid")
        if h then h.Died:Connect(function() clearErrorOrb(); clearAllGhosts(); lagbackCallCount = 0 end) end
    end
end
setupDeathListener()
player.CharacterAdded:Connect(function() task.wait(0.1); setupDeathListener() end)

-- ANTI-DIE
task.spawn(function()
    _G.AntiDieDisabled = false
    local function setupAntiDie()
        if _G.AntiDieDisabled then return end
        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        if _G.AntiDieConnection then pcall(function() _G.AntiDieConnection:Disconnect() end) end
        _G.AntiDieConnection = humanoid:GetPropertyChangedSignal("Health"):Connect(function()
            if _G.AntiDieDisabled then return end
            if humanoid.Health <= 0 then
                humanoid.Health = humanoid.MaxHealth
            end
        end)
    end
    _G.setupAntiDie = setupAntiDie
    setupAntiDie()
    player.CharacterAdded:Connect(function()
        task.wait(0.5)
        if not _G.AntiDieDisabled then
            setupAntiDie()
        end
    end)
end)

-- Automatic Invis During Steal Loop
task.spawn(function()
    local wasStealingForInvis = false
    local autoEnabledInvis = false
    task.wait(1)
    while task.wait(0.15) do
        if Config.AutoInvisDuringSteal == false then
            wasStealingForInvis = false
            autoEnabledInvis = false
        else
            local isStealing = player:GetAttribute("Stealing")
            if isStealing and not wasStealingForInvis then
                if not _G.invisibleStealEnabled and _G._forceInvisToggle then
                    task.defer(function()
                        if player:GetAttribute("Stealing") and not _G.invisibleStealEnabled then
                            pcall(_G._forceInvisToggle)
                            autoEnabledInvis = true
                        end
                    end)
                end
            end
            if not isStealing and autoEnabledInvis and _G.invisibleStealEnabled and _G._forceInvisToggle then
                task.wait(0.3)
                if not player:GetAttribute("Stealing") then
                    pcall(_G._forceInvisToggle)
                    autoEnabledInvis = false
                end
            end
            wasStealingForInvis = isStealing
        end
    end
end)

-- FLOAT
FloatState={active=false,platform=nil,followConn=nil}
local function removeFloatPlatform() if FloatState.followConn then FloatState.followConn:Disconnect(); FloatState.followConn=nil end; if FloatState.platform then FloatState.platform:Destroy(); FloatState.platform=nil end end
local function createFloatPlatform()
    removeFloatPlatform(); local c=player.Character; local hrp=c and c:FindFirstChild("HumanoidRootPart"); if not hrp then return end
    local p=Instance.new("Part"); p.Size=Vector3.new(7,1,7); p.Anchored=true; p.CanCollide=true; p.CanTouch=false; p.CanQuery=false
    p.Transparency=1; p.CastShadow=false; p.CFrame=CFrame.new(hrp.Position-Vector3.new(0,3.35,0)); p.Parent=Workspace; FloatState.platform=p
    FloatState.followConn=RunService.Heartbeat:Connect(function() if not FloatState.active then return end
        local ch=player.Character; local h=ch and ch:FindFirstChild("HumanoidRootPart")
        if h and FloatState.platform then FloatState.platform.CFrame=CFrame.new(h.Position-Vector3.new(0,3.35,0)) end
    end)
end
local function setFloat(on) FloatState.active=on; Config.Float=on; saveConfig(); setToggle("Float",on); if on then createFloatPlatform() else removeFloatPlatform() end end
_G.toggleFloat=function() setFloat(not FloatState.active) end

-- WALKSPEED (CFrame Bypass)
WalkSpeedState = {enabled = false, conn = nil, speed = Config.WalkSpeedValue or 16}
local function setWalkSpeedEnabled(en)
    WalkSpeedState.enabled = en
    Config.WalkSpeedEnabled = en
    setToggle("WalkSpeed", en)
    saveConfig()
    if WalkSpeedState.conn then WalkSpeedState.conn:Disconnect(); WalkSpeedState.conn = nil end
    if not en then return end
    WalkSpeedState.conn = RunService.Heartbeat:Connect(function(dt)
        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if not humanoid or not rootPart or humanoid.Health <= 0 then return end
        if humanoid.MoveDirection.Magnitude > 0 and WalkSpeedState.speed > humanoid.WalkSpeed then
            local extraSpeed = WalkSpeedState.speed - humanoid.WalkSpeed
            rootPart.CFrame = rootPart.CFrame + (humanoid.MoveDirection * extraSpeed * dt)
        end
    end)
end
local function setWalkSpeedValue(v)
    v = math.clamp(math.floor(v + 0.5), 15, 29)
    WalkSpeedState.speed = v
    Config.WalkSpeedValue = v
    saveConfig()
    return v
end
_G.setWalkSpeedEnabled = setWalkSpeedEnabled
_G.setWalkSpeedValue = setWalkSpeedValue

-- CARPET SPEED
CarpetState={enabled=false,conn=nil}
local function setCarpetSpeed(en) CarpetState.enabled=en; setToggle("Carpet Speed",en)
    if CarpetState.conn then CarpetState.conn:Disconnect(); CarpetState.conn=nil end; if not en then return end
    CarpetState.conn=RunService.Heartbeat:Connect(function() local c=player.Character; if not c then return end
        local hum=c:FindFirstChild("Humanoid"); local hrp=c:FindFirstChild("HumanoidRootPart"); if not hum or not hrp then return end
        local tn=Config.TpSettings.Tool or "Flying Carpet"; if not c:FindFirstChild(tn) then local tb=player.Backpack:FindFirstChild(tn); if tb then hum:EquipTool(tb) end end
        if c:FindFirstChild(tn) then local md=hum.MoveDirection; if md.Magnitude>0 then hrp.AssemblyLinearVelocity=Vector3.new(md.X*140,hrp.AssemblyLinearVelocity.Y,md.Z*140) else hrp.AssemblyLinearVelocity=Vector3.new(0,hrp.AssemblyLinearVelocity.Y,0) end end
    end)
end

-- INFINITE JUMP (0.1s cooldown, 55 velocity)
InfJumpState={enabled=false,conn=nil,lastJump=0}
local function setInfiniteJump(en) InfJumpState.enabled=en; Config.InfiniteJump=en; Config.TpSettings.InfiniteJump=en; setToggle("Infinite Jump",en); saveConfig()
    if InfJumpState.conn then InfJumpState.conn:Disconnect(); InfJumpState.conn=nil end; if not en then return end
    InfJumpState.conn=RunService.Heartbeat:Connect(function() if not UIS:IsKeyDown(Enum.KeyCode.Space) then return end
        local now=tick(); if now-InfJumpState.lastJump<0.1 then return end; local c=player.Character; if not c then return end
        local hrp=c:FindFirstChild("HumanoidRootPart"); local hum=c:FindFirstChild("Humanoid"); if not hrp or not hum or hum.Health<=0 then return end
        InfJumpState.lastJump=now; hrp.AssemblyLinearVelocity=Vector3.new(hrp.AssemblyLinearVelocity.X,55,hrp.AssemblyLinearVelocity.Z)
    end)
end

-- ANTI RAGDOLL
-- Anti-Ragdoll System (comprehensive with knockback suppression)
do
    local antiRagdollConnections = {}
    local antiRagdollCharacter, antiRagdollHumanoid, antiRagdollRootPart, antiRagdollAnimator
    local lastVelocity = Vector3.new(0, 0, 0)
    local velocityChangeThreshold = 40
    local velocityMagnitudeThreshold = 25
    local maxVelocity = 15

    local function isFlyingCarpetActive()
        if not antiRagdollCharacter then return false end
        local tool = antiRagdollCharacter:FindFirstChildWhichIsA("Tool")
        if not tool then return false end
        local hrp = antiRagdollCharacter:FindFirstChild("HumanoidRootPart")
        if hrp then
            for _, obj in ipairs(hrp:GetChildren()) do
                if obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then
                    return true
                end
            end
        end
        return false
    end

    local function isRagdolled()
        if not antiRagdollHumanoid then return false end
        local state = antiRagdollHumanoid:GetState()
        return state == Enum.HumanoidStateType.Physics
            or state == Enum.HumanoidStateType.Ragdoll
            or state == Enum.HumanoidStateType.FallingDown
            or state == Enum.HumanoidStateType.GettingUp
    end

    local function enableAntiRagdollControls()
        pcall(function()
            local PlayerModule = player:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 10)
            require(PlayerModule):GetControls():Enable()
        end)
    end

    local function cleanupRagdoll()
        if not antiRagdollCharacter then return end
        local carpetEquipped = isFlyingCarpetActive()

        local function processChildren(parent)
            for _, obj in ipairs(parent:GetChildren()) do
                if obj:IsA("BallSocketConstraint") or obj:IsA("NoCollisionConstraint") or obj:IsA("HingeConstraint")
                    or (obj:IsA("Attachment") and (obj.Name == "A" or obj.Name == "B")) then
                    obj:Destroy()
                elseif obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then
                    if not carpetEquipped then obj:Destroy() end
                elseif obj:IsA("Motor6D") then
                    obj.Enabled = true
                elseif obj:IsA("BasePart") then
                    for _, child in ipairs(obj:GetChildren()) do
                        if child:IsA("BallSocketConstraint") or child:IsA("NoCollisionConstraint") or child:IsA("HingeConstraint") or child:IsA("Motor6D") then
                            if child:IsA("Motor6D") then
                                child.Enabled = true
                            else
                                child:Destroy()
                            end
                        elseif child:IsA("Attachment") and (child.Name == "A" or child.Name == "B") then
                            child:Destroy()
                        end
                    end
                end
            end
        end

        pcall(function() processChildren(antiRagdollCharacter) end)

        if antiRagdollAnimator then
            for _, track in pairs(antiRagdollAnimator:GetPlayingAnimationTracks()) do
                local animName = track.Animation and track.Animation.Name:lower() or ""
                if animName:find("rag") or animName:find("fall") or animName:find("hurt") or animName:find("down") then
                    track:Stop(0)
                end
            end
        end
    end

    local function setupAntiRagdollCharacter(char)
        antiRagdollCharacter = char
        antiRagdollHumanoid = char:WaitForChild("Humanoid", 10)
        antiRagdollRootPart = char:WaitForChild("HumanoidRootPart", 10)
        antiRagdollAnimator = antiRagdollHumanoid and antiRagdollHumanoid:WaitForChild("Animator", 10)
        lastVelocity = Vector3.new(0, 0, 0)
    end

    local function clearAntiRagdollConnections()
        for _, c in pairs(antiRagdollConnections) do
            pcall(function() c:Disconnect() end)
        end
        antiRagdollConnections = {}
    end

    local function setupAntiRagdollConnections()
        clearAntiRagdollConnections()
        if not antiRagdollHumanoid or not antiRagdollRootPart then return end

        table.insert(antiRagdollConnections, antiRagdollHumanoid.StateChanged:Connect(function()
            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                if not isFlyingCarpetActive() then
                    antiRagdollHumanoid:ChangeState(Enum.HumanoidStateType.Running)
                end
                cleanupRagdoll()
                Workspace.CurrentCamera.CameraSubject = antiRagdollHumanoid
                enableAntiRagdollControls()
            end
        end))

        pcall(function()
            local impulsePath = ReplicatedStorage:FindFirstChild("Packages")
            if impulsePath then
                impulsePath = impulsePath:FindFirstChild("Net")
                if impulsePath then
                    impulsePath = impulsePath:FindFirstChild("RE/CombatService/ApplyImpulse")
                    if impulsePath then
                        table.insert(antiRagdollConnections, impulsePath.OnClientEvent:Connect(function()
                            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                                antiRagdollRootPart.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                            end
                        end))
                    end
                end
            end
        end)

        table.insert(antiRagdollConnections, antiRagdollCharacter.DescendantAdded:Connect(function()
            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                cleanupRagdoll()
            end
        end))

        table.insert(antiRagdollConnections, RunService.Heartbeat:Connect(function()
            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                cleanupRagdoll()
                local velocity = antiRagdollRootPart.AssemblyLinearVelocity
                if (velocity - lastVelocity).Magnitude > velocityChangeThreshold
                    and velocity.Magnitude > velocityMagnitudeThreshold then
                    antiRagdollRootPart.AssemblyLinearVelocity = velocity.Unit * math.min(velocity.Magnitude, maxVelocity)
                end
                lastVelocity = velocity
            end
        end))

        enableAntiRagdollControls()
        cleanupRagdoll()
    end

    function startAntiRagdoll()
        _G.AntiRagdollEnabled = true
        _G.antiKnockbackEnabled = true
        Config.AntiRagdoll = true; setToggle("Anti Ragdoll", true); saveConfig()
        if player.Character then
            setupAntiRagdollCharacter(player.Character)
            setupAntiRagdollConnections()
        end
    end

    function stopAntiRagdoll()
        _G.AntiRagdollEnabled = false
        _G.antiKnockbackEnabled = false
        Config.AntiRagdoll = false; setToggle("Anti Ragdoll", false); saveConfig()
        clearAntiRagdollConnections()
    end

    _G.toggleAntiRagdoll = function(enabled)
        if enabled then startAntiRagdoll() else stopAntiRagdoll() end
    end
    _G.enableAntiKnockback = function() startAntiRagdoll() end
    _G.disableAntiKnockback = function() stopAntiRagdoll() end

    player.CharacterAdded:Connect(function(char)
        clearAntiRagdollConnections()
        antiRagdollCharacter = nil; antiRagdollHumanoid = nil; antiRagdollRootPart = nil; antiRagdollAnimator = nil
        local humanoid = char:WaitForChild("Humanoid", 10)
        local rootPart = char:WaitForChild("HumanoidRootPart", 10)
        if not humanoid or not rootPart then return end
        task.wait(0.2)
        setupAntiRagdollCharacter(char)
        if _G.AntiRagdollEnabled or _G.antiKnockbackEnabled then
            setupAntiRagdollConnections()
        end
    end)

    if player.Character then
        setupAntiRagdollCharacter(player.Character)
        if _G.AntiRagdollEnabled or _G.antiKnockbackEnabled then
            setupAntiRagdollConnections()
        end
    end
end

local function applyUnwalk(char, on)
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local animator = hum and hum:FindFirstChildOfClass("Animator")
    local animate = char:FindFirstChild("Animate")
    if on then
        if animate then animate.Disabled = true end
        if animator then
            local ok, tracks = pcall(function() return animator:GetPlayingAnimationTracks() end)
            if ok and tracks then
                for _, t in ipairs(tracks) do pcall(function() t:Stop(0) end) end
            end
        end
    else
        if animate then animate.Disabled = false end
    end
end

local function setUnwalk(on)
    Config.Unwalk = on
    saveConfig()
    setToggle("Unwalk", on)
    if on then
        task.spawn(function()
            local char = player.Character
            for i = 1, 6 do
                if not Config.Unwalk or player.Character ~= char then break end
                applyUnwalk(char, true)
                task.wait(0.3)
            end
        end)
    else
        applyUnwalk(player.Character, false)
    end
end
_G.setUnwalk = setUnwalk

player.CharacterAdded:Connect(function(char)
    task.spawn(function()
        char:WaitForChild("Humanoid", 10)
        task.wait(0.1)
        if Config.Unwalk then
            for i = 1, 6 do
                if not Config.Unwalk or player.Character ~= char then break end
                applyUnwalk(char, true)
                task.wait(0.3)
            end
        end
    end)
end)

local function isMyPlot_Instant(plotName)
    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(plotName)
    if not plot then return false end

    -- Method 1: Synchronizer check 
    local syncSuccess = false
    local isOwner = false
    pcall(function()
        local Packages = ReplicatedStorage:FindFirstChild("Packages")
        local Synchronizer = Packages and require(Packages:FindFirstChild("Synchronizer"))
        if Synchronizer then
            local ch = Synchronizer:Get(plotName)
            if ch then
                local own = _G.sProp(ch, "Owner")
                if own then
                    syncSuccess = true
                    if (typeof(own) == "Instance" and own == player) or
                       (typeof(own) == "table" and own.UserId == player.UserId) or
                       (typeof(own) == "number" and own == player.UserId) or
                       (typeof(own) == "string" and (own:lower() == player.Name:lower() or own:lower() == player.DisplayName:lower())) then
                        isOwner = true
                    end
                end
            end
        end
    end)
    if syncSuccess then return isOwner end

    -- Method 2: SurfaceGui TextLabel check (Display Name or Username)
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local surfaceGui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
        if surfaceGui then
            local label = surfaceGui:FindFirstChildWhichIsA("TextLabel", true)
            if label then
                local text = label.Text:lower()
                if text:find(player.DisplayName:lower(), 1, true) or text:find(player.Name:lower(), 1, true) then
                    return true
                end
            end
        end
    end

    -- Method 3: BillboardGui "YourBase" check
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") and yb.Enabled == true then
            return true
        end
    end

    return false
end
_G.isMyPlot_Instant = isMyPlot_Instant

-- UNLOCK BASE 
local function getUnlockHRP()
    local c = LocalPlayer.Character or player.Character
    if not c then return end
    return c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("UpperTorso")
end

local function smartInteract(number)
    local char = LocalPlayer.Character or player.Character
    local hrp = getUnlockHRP()
    if not char or not hrp then return end

    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return end

    local closestPlot, minDistance = nil, 40

    for _, plot in pairs(plots:GetChildren()) do
        local plotPos
        if plot:IsA("Model") then
            plotPos = plot.PrimaryPart and plot.PrimaryPart.Position or plot:GetPivot().Position
        else
            plotPos = plot.Position
        end

        local dist = (hrp.Position - plotPos).Magnitude
        if dist < minDistance then
            closestPlot = plot
            minDistance = dist
        end
    end

    if closestPlot and closestPlot:FindFirstChild("Unlock") then
        local items = {}

        for _, item in pairs(closestPlot.Unlock:GetChildren()) do
            local pos = item:IsA("Model") and item:GetPivot().Position or item.Position
            table.insert(items, {
                Obj = item,
                Y = pos.Y
            })
        end

        table.sort(items, function(a, b)
            return a.Y < b.Y
        end)

        if items[number] then
            for _, pr in pairs(items[number].Obj:GetDescendants()) do
                if pr:IsA("ProximityPrompt") then
                    pcall(function()
                        fireproximityprompt(pr)
                    end)
                end
            end
        end
    end
end

local function getCurrentUnlockFloor()
    local hrp = getUnlockHRP()
    if not hrp then return 1 end

    local y = hrp.Position.Y
    if y < 12 then
        return 1
    else
        return 2
    end
end

-- PROXIMITY AP
ProximityAPActive=false
proxAPRing = nil

local function createProxAPRing()
    local existing = Workspace:FindFirstChild("XiProxAPRing")
    if existing then existing:Destroy() end
    local r = Instance.new("Part")
    r.Name = "XiProxAPRing"
    r.Shape = Enum.PartType.Cylinder
    r.Anchored = true
    r.CanCollide = false
    r.CanTouch = false
    r.CanQuery = false
    r.CastShadow = false
    r.Material = Enum.Material.Neon
    r.Transparency = 0.8
    r.Color = Color3.fromRGB(232, 111, 177)
    local range = Config.ProximityRange or 15
    r.Size = Vector3.new(0.2, range*2, range*2)
    r.Parent = Workspace
    proxAPRing = r
end

local function destroyProxAPRing()
    if proxAPRing then proxAPRing:Destroy(); proxAPRing = nil end
    local e = Workspace:FindFirstChild("XiProxAPRing")
    if e then e:Destroy() end
end

_proxAPRingFrame=0
RunService.Heartbeat:Connect(function()
    if not ProximityAPActive then return end
    _proxAPRingFrame = _proxAPRingFrame + 1
    if _proxAPRingFrame < 2 then return end
    _proxAPRingFrame = 0
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp or not proxAPRing then return end
    local range = Config.ProximityRange or 15
    proxAPRing.Size = Vector3.new(0.2, range * 2, range * 2)
    proxAPRing.CFrame = (hrp.CFrame * CFrame.Angles(0, 0, math.rad(90))) - Vector3.new(0, 2.8, 0)
end)

local function setProximityAP(on)
    ProximityAPActive = on
    -- Never save ProximityAP to config â€” always starts OFF
    Config.ProximityAP = false
    setToggle("Proximity", on)
    if on then createProxAPRing() else destroyProxAPRing() end
end

onToggleChanged("Proximity", function(on)
    ProximityAPActive = on
    -- Never save ProximityAP to config â€” always starts OFF
    Config.ProximityAP = false
    if on then createProxAPRing() else destroyProxAPRing() end
end)

task.spawn(function()
    while true do
        task.wait(0.2)
        if ProximityAPActive then
            local mc = player.Character
            local mh = mc and mc:FindFirstChild("HumanoidRootPart")
            if mh then
                for _,p in ipairs(Players:GetPlayers()) do
                    if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                        if isPlayerBlacklisted(p) then continue end
                        if (p.Character.HumanoidRootPart.Position - mh.Position).Magnitude <= (Config.ProximityRange or 15) then
                            local activeCmds = {}
                            for _,cmd in ipairs(AP_ALL_COMMANDS) do
                                if not apIsOnCooldown(cmd) then
                                    table.insert(activeCmds, cmd)
                                end
                            end
                            for i, cmd in ipairs(activeCmds) do
                                task.spawn(function()
                                    task.wait((i - 1) * 0.01)
                                    runAdminCommand(p, cmd)
                                end)
                            end
                        end
                    end
                end
            end
        end
    end
end)

-- AUTO RESET ON BALLOON / AUTO KICK ON STEAL / CLEAN ERRORS
task.spawn(function() while true do task.wait(1); if not Config.AutoResetBalloon then continue end
    for _,g in ipairs(playerGui:GetDescendants()) do local txt=(g:IsA("TextLabel") or g:IsA("TextButton")) and g.Text
        if txt and string.find(txt,'ran "balloon" on you') then executeReset(true); break end end end end)

task.spawn(function() local kw="you stole"; local hooked=setmetatable({},{__mode="k"})
    local function hookObj(obj) if hooked[obj] then return end; hooked[obj]=true
        if Config.AutoKickOnSteal and string.find(string.lower(tostring(obj.Text or "")),kw,1,true) then kickPlayer(tostring(obj.Text or "")); return end
        obj:GetPropertyChangedSignal("Text"):Connect(function() if Config.AutoKickOnSteal and string.find(string.lower(tostring(obj.Text or "")),kw,1,true) then kickPlayer(tostring(obj.Text or "")) end end)
    end
    local function watchRoot(root) for _,obj in ipairs(root:GetDescendants()) do if obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox") then hookObj(obj) end end
        root.DescendantAdded:Connect(function(desc) if desc:IsA("TextLabel") or desc:IsA("TextButton") or desc:IsA("TextBox") then hookObj(desc) end end) end
    for _,g in ipairs(playerGui:GetChildren()) do watchRoot(g) end; playerGui.ChildAdded:Connect(function(g) watchRoot(g) end)
end)

task.spawn(function() local GS=pcall(function() return cloneref(game:GetService("GuiService")) end) and cloneref(game:GetService("GuiService")) or game:GetService("GuiService")
    while true do if Config.CleanErrorGUIs then pcall(function() GS:ClearError() end) end; task.wait(0.1) end end)

-- ============================================================
-- AUTO STEAL / GRAB
-- ============================================================
do
local Packages = ReplicatedStorage:WaitForChild("Packages")
local Datas = ReplicatedStorage:WaitForChild("Datas")
local Synchronizer = require(Packages:WaitForChild("Synchronizer"))
local AnimalsData = require(Datas:WaitForChild("Animals"))

autoStealEnabled = Config.AutoStealEnabled
if autoStealEnabled == nil then autoStealEnabled = true end
instantStealEnabled = Config.InstantStealEnabled
if instantStealEnabled == nil then instantStealEnabled = true end
stealHighestEnabled = Config.StealHighest
if stealHighestEnabled == nil then stealHighestEnabled = true end
stealPriorityEnabled = Config.StealPriority
stealNearestEnabled = Config.StealNearest
selectedTargetIndex = 1
selectedTargetUID = nil
manuallySelectedUID = nil
currentStealTargetUID = nil
activeProgressTween = nil
instantStealReady = false
instantStealDidInit = false
INSTANT_STEAL_RADIUS = 60
INSTANT_STEAL_COOLDOWN = 0
lastInstantStealTime = 0
PromptMemoryCache = {}
InternalStealCacheData = {}

-- ============================================================
-- AUTO GRAB INTEGRATION 
-- ============================================================
local CONFIG = {
    AUTO_STEAL = false,
    RADIUS = 60
}

local boxes = {
    {min = Vector3.new(-337.448303, -3.898971, -122.397758), max = Vector3.new(-328.004578, -3.898971, 242.625626)},
    {min = Vector3.new(-327.257660, -3.899109, -122.228622), max = Vector3.new(-320.600891, -3.899109, 242.612259)},
    {min = Vector3.new(-319.783386, -3.898970, -122.227089), max = Vector3.new(-312.908325, -3.898970, 242.585617)},
    {min = Vector3.new(-312.445648, -3.899108, -122.389832), max = Vector3.new(-305.489899, -3.899108, 242.456818)},
    {min = Vector3.new(-305.037048, -3.898970, -122.230743), max = Vector3.new(-293.957489, -3.898970, 242.606873)},
    {min = Vector3.new(-491.448608, -3.898972, -122.253258), max = Vector3.new(-481.811737, -3.898972, 242.615005)},
    {min = Vector3.new(-498.971069, -3.898970, -122.382767), max = Vector3.new(-491.748840, -3.898970, 242.612061)},
    {min = Vector3.new(-506.436737, -3.898972, -122.411476), max = Vector3.new(-499.318542, -3.898972, 242.615982)},
    {min = Vector3.new(-513.783569, -3.898972, -122.223297), max = Vector3.new(-506.801849, -3.898972, 242.627090)},
    {min = Vector3.new(-525.236938, -3.898972, -122.409813), max = Vector3.new(-514.265015, -3.898972, 242.608932)},
}

local trackedPrompts = {}
local lastFire = {}

local SAFE_POLL_RATE = 0.05
local SAFE_POLL_OVERRIDE_UNTIL = 0

function _G.getSafePollRate()
    if os.clock() < SAFE_POLL_OVERRIDE_UNTIL then
        return 0.27
    end
    return SAFE_POLL_RATE
end

function _G.triggerSafePollBoost()
    SAFE_POLL_OVERRIDE_UNTIL = os.clock() + 3
end

local FIRE_DEBOUNCE = 0.12
local FIRE_BURST = 4
local ENABLE_BURST = 35
local ENABLE_DEBOUNCE = 0.00
local ENABLE_COOLDOWN = 0.08
local lastEnableFire = {}

local function getHRP()
    local char = LocalPlayer.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function getBoxIndex(pos)
    for i,b in ipairs(boxes) do
        if pos.X >= math.min(b.min.X,b.max.X) and pos.X <= math.max(b.min.X,b.max.X)
        and pos.Z >= math.min(b.min.Z,b.max.Z) and pos.Z <= math.max(b.min.Z,b.max.Z) then
            return i
        end
    end
end

local function getPromptPosition(prompt)
    local p = prompt.Parent
    if not p then return end
    if p:IsA("Attachment") and p.Parent then p = p.Parent end
    if p:IsA("BasePart") then return p.Position elseif p:IsA("Model") then return p:GetPivot().Position end
end

local function promptMatchesSelectedPet(prompt)
    if not SharedState then return false end
    local selected = SharedState.SelectedPetData
    if not selected then return false end
    local model = prompt:FindFirstAncestorOfClass("Model")
    if not model then return false end
    if selected.slot then
        local slotAncestor = prompt:FindFirstAncestor(selected.slot)
        if slotAncestor or model.Name == selected.slot or (model.Parent and model.Parent.Name == selected.slot) then return true end
    end
    local name = selected.name or selected.petName
    if name then
        local wantedName = string.lower(name)
        local current = model
        while current do
            if current.Name and string.lower(current.Name) == wantedName then return true end
            current = current.Parent
        end
    end
    return false
end

local function isPromptAvailable(prompt, hrpPos)
    if not autoStealEnabled or not instantStealEnabled then return false end
    if not prompt or not prompt.Parent or not prompt.Enabled then return false end
    local pos = getPromptPosition(prompt)
    if not pos then return false end
    local plot = prompt:FindFirstAncestorOfClass("Model")
    if plot then
        local plots = workspace:FindFirstChild("Plots")
        if plots then
            local parentPlot = prompt:FindFirstAncestorWhichIsA("Model")
            while parentPlot and parentPlot.Parent ~= plots do parentPlot = parentPlot.Parent end
            if parentPlot then
                local sign = parentPlot:FindFirstChild("PlotSign")
                if sign then
                    local gui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
                    local label = gui and gui:FindFirstChildWhichIsA("TextLabel", true)
                    if label then
                        local txt = label.Text:lower()
                        if txt:find(game.Players.LocalPlayer.Name:lower(), 1, true) or txt:find(game.Players.LocalPlayer.DisplayName:lower(), 1, true) then return false end
                    end
                end
            end
        end
    end
    if _G.NEAREST_INSTANT_MODE == true then
        -- Box checks removed to allow nearest to work globally based on radius
    end
    if not (_G.NEAREST_INSTANT_MODE == true) then
        if not promptMatchesSelectedPet(prompt) then return false end
    end
    local configuredRadius = Config.AutoGrabRadius or 60
    return (pos - hrpPos).Magnitude <= configuredRadius
end

local function canFire(prompt, debounce)
    local t = os.clock()
    local last = lastFire[prompt]
    if last and (t - last) < debounce then return false end
    lastFire[prompt] = t
    return true
end

local function firePrompt(prompt, burst, debounce)
    if not prompt or not prompt.Parent or not prompt.Enabled or not canFire(prompt, debounce) then return end
    for i = 1, burst do pcall(function() fireproximityprompt(prompt, 0) end) end
end

local function trackPrompt(prompt)
    if trackedPrompts[prompt] then return end
    trackedPrompts[prompt] = true
    local function tryInstantEnableFire()
        if not autoStealEnabled or not instantStealEnabled then return end
        local hrp = getHRP()
        if hrp and isPromptAvailable(prompt, hrp.Position) then
            CONFIG.AUTO_STEAL = true
            local now = os.clock()
            local le = lastEnableFire[prompt]
            if not le or (now - le) >= ENABLE_COOLDOWN then
                lastEnableFire[prompt] = now
                firePrompt(prompt, ENABLE_BURST, ENABLE_DEBOUNCE)
            end
        end
    end
    task.defer(tryInstantEnableFire)
    pcall(function() prompt:GetPropertyChangedSignal("Enabled"):Connect(function() if prompt.Enabled then tryInstantEnableFire() end end) end)
    prompt.AncestryChanged:Connect(function() if not prompt:IsDescendantOf(workspace) then trackedPrompts[prompt] = nil; lastFire[prompt] = nil; lastEnableFire[prompt] = nil end end)
end

local function scanBrainrotPrompts()
    local plots = workspace:FindFirstChild("Plots")
    if plots then
        for _, plot in ipairs(plots:GetChildren()) do
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, obj in ipairs(podiums:GetDescendants()) do
                    if obj:IsA("ProximityPrompt") then trackPrompt(obj) end
                end
            end
        end
    end
end

scanBrainrotPrompts()
workspace.DescendantAdded:Connect(function(obj)
    if obj:IsA("ProximityPrompt") and obj:FindFirstAncestor("AnimalPodiums") then trackPrompt(obj) end
end)

task.spawn(function()
    while task.wait(_G.getSafePollRate()) do
        _G.NEAREST_INSTANT_MODE = (stealNearestEnabled == true)
        if autoStealEnabled and instantStealEnabled then
            local hrp = getHRP()
            if not hrp then CONFIG.AUTO_STEAL = false; continue end
            local myPos = hrp.Position
            local anyAvailable = false
            for prompt in pairs(trackedPrompts) do
                if isPromptAvailable(prompt, myPos) then
                    anyAvailable = true
                    if CONFIG.AUTO_STEAL then firePrompt(prompt, FIRE_BURST, FIRE_DEBOUNCE) end
                end
            end
            CONFIG.AUTO_STEAL = anyAvailable
        else
            CONFIG.AUTO_STEAL = false
        end
    end
end)

local function isMyBaseAnimal(animalData)
    if not animalData or not animalData.plot then return false end
    return isMyPlot_Instant(animalData.plot)
end

function get_all_pets()
    local out = {}
    local cache = SharedState.AllAnimalsCache
    local minGenStr = Config.StealNearest and Config.TpSettings.MinGenForGrab or Config.TpSettings.MinGenForTp
    local minGen = parseMinGen(minGenStr)
    local myName = LocalPlayer.Name
    local myDisplay = LocalPlayer.DisplayName
    local prioSet = {}
    for _, p in ipairs(priorityList) do prioSet[p:lower()] = true end
    if cache and type(cache) == "table" then
        for _, a in ipairs(cache) do
            if a and (a.genValue or 0) >= 1 and a.owner ~= myName and a.owner ~= myDisplay then
                local isBrainrot = ((tonumber(a.genValue) or 0) >= 10000000)
                local isPriority = (a.name and prioSet[a.name:lower()]) or (a.index and prioSet[a.index:lower()])
                if isBrainrot and a.name then SharedState.BrainrotNames[a.name:lower()] = true end
                if minGen > 0 and (tonumber(a.genValue) or 0) < minGen then
                    if not isBrainrot and not isPriority then continue end
                end
                table.insert(out, {
                    name = a.name,
                    petName = a.name,
                    mpsText = a.genText,
                    mpsValue = a.genValue,
                    petValue = a.petValue or 0,
                    owner = a.owner,
                    plot = a.plot,
                    slot = a.slot,
                    uid = a.uid,
                    mutation = a.mutation,
                    animalData = a,
                })
            end
        end
    end
    table.sort(out, function(a, b) return (a.mpsValue or 0) > (b.mpsValue or 0) end)
    return out
end

function get_all_pets_by_value()
    local out = {}
    local cache = SharedState.AllAnimalsCache
    local minGenStr = Config.StealNearest and Config.TpSettings.MinGenForGrab or Config.TpSettings.MinGenForTp
    local minGen = parseMinGen(minGenStr)
    local myName = LocalPlayer.Name
    local myDisplay = LocalPlayer.DisplayName
    if cache and type(cache) == "table" then
        for _, a in ipairs(cache) do
            if a and (a.genValue or 0) >= 1 and a.owner ~= myName and a.owner ~= myDisplay then
                if minGen > 0 and (tonumber(a.genValue) or 0) < minGen then continue end
                table.insert(out, {
                    name = a.name,
                    petName = a.name,
                    mpsText = a.genText,
                    mpsValue = a.genValue,
                    petValue = a.petValue or 0,
                    owner = a.owner,
                    plot = a.plot,
                    slot = a.slot,
                    uid = a.uid,
                    mutation = a.mutation,
                    animalData = a,
                })
            end
        end
    end
    table.sort(out, function(a, b) return (a.petValue or 0) > (b.petValue or 0) end)
    return out
end

function findProximityPromptForAnimal(animalData)
    if not animalData then return nil end
    local cp = PromptMemoryCache[animalData.uid]
    if cp and cp.Parent then return cp end
    local plot = Workspace.Plots:FindFirstChild(animalData.plot)
    if not plot then return nil end
    local podiums = plot:FindFirstChild("AnimalPodiums")
    if not podiums then return nil end
    local ch = Synchronizer:Get(plot.Name)
    if not ch then
        local podium = podiums:FindFirstChild(animalData.slot)
        if podium then
            local base = podium:FindFirstChild("Base")
            local spawn = base and base:FindFirstChild("Spawn")
            if spawn then
                local attach = spawn:FindFirstChild("PromptAttachment")
                if attach then
                    for _, p in ipairs(attach:GetChildren()) do
                        if p:IsA("ProximityPrompt") then
                            PromptMemoryCache[animalData.uid] = p
                            return p
                        end
                    end
                end
            end
        end
        return nil
    end
    local al = ch:Get("AnimalList")
    if not al then return nil end
    local brainrotName = (animalData.name and animalData.name:lower()) or ""
    local targetSlot = animalData.slot
    local foundPodium = nil
    for slot, ad in pairs(al) do
        if (type(ad) == "table") and (tostring(slot) == targetSlot) then
            local aName, aInfo = ad.Index, AnimalsData[ad.Index]
            if aInfo and ((aInfo.DisplayName or aName):lower() == brainrotName) then
                foundPodium = podiums:FindFirstChild(tostring(slot))
                break
            end
        end
    end
    if not foundPodium then foundPodium = podiums:FindFirstChild(animalData.slot) end
    if foundPodium then
        local base = foundPodium:FindFirstChild("Base")
        local spawn = base and base:FindFirstChild("Spawn")
        if spawn then
            local attach = spawn:FindFirstChild("PromptAttachment")
            if attach then
                for _, p in ipairs(attach:GetChildren()) do
                    if p:IsA("ProximityPrompt") and p.Enabled and (p.ActionText:lower():find("steal") or p.ActionText == "") then
                        PromptMemoryCache[animalData.uid] = p
                        return p
                    end
                end
            end
            local startPos = spawn.Position
            local slotX, slotZ = startPos.X, startPos.Z
            local nearestPrompt = nil
            local minDist = math.huge
            for _, desc in pairs(plot:GetDescendants()) do
                if desc:IsA("ProximityPrompt") and desc.Enabled and (desc.ActionText:lower():find("steal") or desc.ActionText == "") then
                    local part = desc.Parent
                    local promptPos = nil
                    if part and part:IsA("BasePart") then
                        promptPos = part.Position
                    elseif part and part:IsA("Attachment") and part.Parent and part.Parent:IsA("BasePart") then
                        promptPos = part.Parent.Position
                    end
                    if promptPos then
                        local checkStartY = startPos.Y
                        if brainrotName:find("la secret combinasion") then checkStartY = startPos.Y - 5 end
                        local horizontalDist = math.sqrt(((promptPos.X - slotX) ^ 2) + ((promptPos.Z - slotZ) ^ 2))
                        if (horizontalDist < 5) and (promptPos.Y > checkStartY) then
                            local yDist = promptPos.Y - checkStartY
                            if yDist < minDist then
                                minDist = yDist
                                nearestPrompt = desc
                            end
                        end
                    end
                end
            end
            if nearestPrompt then
                PromptMemoryCache[animalData.uid] = nearestPrompt
                return nearestPrompt
            end
        end
    end
    return nil
end

-- executeInstantSteal logic removed

function setStealMode(mode)
    manuallySelectedUID = nil
    stealHighestEnabled = (mode == "Highest")
    stealPriorityEnabled = (mode == "Priority")
    stealNearestEnabled = (mode == "Nearest")
    Config.StealHighest = stealHighestEnabled
    Config.StealPriority = stealPriorityEnabled
    Config.StealNearest = stealNearestEnabled
    Config.StealMode = mode
    saveConfig()
    setToggle("Steal Highest", stealHighestEnabled)
    setToggle("Steal Priority", stealPriorityEnabled)
    setToggle("Steal Nearest", stealNearestEnabled)
    -- Bridge to SXE Clone-TP engine
    if _G.SXEStealMode then pcall(_G.SXEStealMode, mode) end
end

-- Create the Bottom Steal HUD matching 
local hudGui = playerGui:FindFirstChild("AutoStealCurrentTargetHUD")
if hudGui then hudGui:Destroy() end
hudGui = Instance.new("ScreenGui")
hudGui.Name = "AutoStealCurrentTargetHUD"
hudGui.ResetOnSpawn = false
hudGui.IgnoreGuiInset = true
hudGui.DisplayOrder = 998
hudGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
hudGui.Parent = playerGui
local STEALBAR = {
    PANEL = Color3.fromRGB(15, 15, 15),
    TEXT = Color3.fromRGB(255, 255, 255),
    STROKE = Color3.fromRGB(60, 40, 55),
    GLOW = Color3.fromRGB(232, 111, 177),
    TRACK = Color3.fromRGB(55, 40, 50),
    TRACK2 = Color3.fromRGB(35, 30, 35),
    FILL1 = Color3.fromRGB(238, 98, 178),
    FILL2 = Color3.fromRGB(232, 111, 177),
}

local mobileScale = UIS.TouchEnabled and 0.6 or 1
local targetHud = Instance.new("Frame", hudGui)
targetHud.Name = "CurrentTargetHUD"
targetHud.AnchorPoint = Vector2.new(0.5, 1)
targetHud.Size = UDim2.new(0, 230 * mobileScale, 0, 46 * mobileScale)
targetHud.Position = UDim2.new(0.5, 0, 1, -135)
targetHud.BackgroundColor3 = STEALBAR.PANEL
targetHud.BackgroundTransparency = 0.02
targetHud.BorderSizePixel = 0
targetHud.ZIndex = 70
Instance.new("UICorner", targetHud).CornerRadius = UDim.new(0, math.floor(12 * mobileScale))

local hudStroke = Instance.new("UIStroke", targetHud)
hudStroke.Color = STEALBAR.STROKE
hudStroke.Thickness = 1
hudStroke.Transparency = 0.35

local hudGlow = Instance.new("UIStroke", targetHud)
hudGlow.Color = STEALBAR.GLOW
hudGlow.Thickness = 3
hudGlow.Transparency = 0.84
hudGlow.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

local hudShadow = Instance.new("ImageLabel", targetHud)
hudShadow.Name = "Shadow"
hudShadow.AnchorPoint = Vector2.new(0.5, 0.5)
hudShadow.Position = UDim2.new(0.5, 0, 0.5, 1)
hudShadow.Size = UDim2.new(1, 20, 1, 20)
hudShadow.BackgroundTransparency = 1
hudShadow.Image = "rbxassetid://6014261993"
hudShadow.ImageColor3 = Color3.new(0, 0, 0)
hudShadow.ImageTransparency = 0.72
hudShadow.ScaleType = Enum.ScaleType.Slice
hudShadow.SliceCenter = Rect.new(49, 49, 450, 450)
hudShadow.ZIndex = 69

local hudName = Instance.new("TextLabel", targetHud)
hudName.Name = "TargetName"
hudName.Size = UDim2.new(1, -12, 0, 13 * mobileScale)
hudName.Position = UDim2.fromOffset(6 * mobileScale, 3 * mobileScale)
hudName.BackgroundTransparency = 1
hudName.Font = Enum.Font.GothamBold
hudName.TextSize = 11 * mobileScale
hudName.TextColor3 = STEALBAR.TEXT
hudName.TextXAlignment = Enum.TextXAlignment.Center
hudName.TextTruncate = Enum.TextTruncate.AtEnd
hudName.ZIndex = 72
hudName.Text = "No target"

local hudProgressBg = Instance.new("Frame", targetHud)
hudProgressBg.Name = "ProgressBg"
hudProgressBg.Size = UDim2.new(1, -10 * mobileScale, 0, 18 * mobileScale)
hudProgressBg.Position = UDim2.fromOffset(5 * mobileScale, 18 * mobileScale)
hudProgressBg.BackgroundColor3 = STEALBAR.TRACK
hudProgressBg.BorderSizePixel = 0
hudProgressBg.ZIndex = 72
Instance.new("UICorner", hudProgressBg).CornerRadius = UDim.new(0, math.floor(8 * mobileScale))

local hudProgressBgStroke = Instance.new("UIStroke", hudProgressBg)
hudProgressBgStroke.Color = STEALBAR.STROKE
hudProgressBgStroke.Thickness = 1
hudProgressBgStroke.Transparency = 0.55

local hudInnerTrack = Instance.new("Frame", hudProgressBg)
hudInnerTrack.Name = "InnerTrack"
hudInnerTrack.Size = UDim2.new(1, -2, 1, -2)
hudInnerTrack.Position = UDim2.fromOffset(1, 1)
hudInnerTrack.BackgroundColor3 = STEALBAR.TRACK2
hudInnerTrack.BackgroundTransparency = 0.15
hudInnerTrack.BorderSizePixel = 0
hudInnerTrack.ZIndex = 72
Instance.new("UICorner", hudInnerTrack).CornerRadius = UDim.new(0, math.floor(7 * mobileScale))

local hudProgressFill = Instance.new("Frame", hudProgressBg)
hudProgressFill.Name = "ProgressFill"
hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
hudProgressFill.BackgroundColor3 = STEALBAR.FILL1
hudProgressFill.BorderSizePixel = 0
hudProgressFill.ZIndex = 73
Instance.new("UICorner", hudProgressFill).CornerRadius = UDim.new(0, math.floor(8 * mobileScale))

local hudProgressFillGradient = Instance.new("UIGradient", hudProgressFill)
hudProgressFillGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, STEALBAR.FILL1),
    ColorSequenceKeypoint.new(1, STEALBAR.FILL2),
})

local hudProgressFillStroke = Instance.new("UIStroke", hudProgressFill)
hudProgressFillStroke.Color = Color3.fromRGB(255, 255, 255)
hudProgressFillStroke.Thickness = 1
hudProgressFillStroke.Transparency = 0.45

local hudPercent = Instance.new("TextLabel", hudProgressBg)
hudPercent.Name = "Percent"
hudPercent.Size = UDim2.new(1, 0, 1, 0)
hudPercent.BackgroundTransparency = 1
hudPercent.Font = Enum.Font.GothamBold
hudPercent.TextSize = 12 * mobileScale
hudPercent.TextColor3 = STEALBAR.TEXT
hudPercent.TextStrokeTransparency = 0.7
hudPercent.TextXAlignment = Enum.TextXAlignment.Center
hudPercent.ZIndex = 74
hudPercent.Text = "0%"

local function applySelection(newIndex, pets)
    if newIndex and (newIndex >= 1) and (newIndex <= #pets) then
        local newUID = pets[newIndex].uid
        if (selectedTargetIndex ~= newIndex) or (selectedTargetUID ~= newUID) or (SharedState.SelectedPetData == nil) then
            selectedTargetIndex = newIndex
            selectedTargetUID = newUID
            SharedState.SelectedPetData = pets[newIndex]
            if SharedState.SelectedPetData then SharedState.LastTargetedPetMpsValue = SharedState.SelectedPetData.mpsValue or 0 end
        end
    end
end

-- Update Selection Thread
task.spawn(function()
    while true do
        task.wait(0.1)
        if autoStealEnabled then
            local pets = get_all_pets()
            if #pets > 0 then
                if manuallySelectedUID then
                    local found = false
                    for i, p in ipairs(pets) do if p.uid == manuallySelectedUID then applySelection(i, pets); found = true; break end end
                    if not found then manuallySelectedUID = nil; selectedTargetUID = nil; SharedState.SelectedPetData = nil end
                elseif stealPriorityEnabled then
                    local foundPrioIndex = nil
                    for _, pName in ipairs(priorityList) do
                        local searchName = pName:lower()
                        for i, p in ipairs(pets) do if (p.petName and p.petName:lower() == searchName) or (p.animalData and p.animalData.index and p.animalData.index:lower() == searchName) then foundPrioIndex = i; break end end
                        if foundPrioIndex then break end
                    end
                    applySelection(foundPrioIndex, pets)
                elseif stealNearestEnabled then
                    local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        local bestIndex, bestDist = nil, math.huge
                        for i, p in ipairs(pets) do
                            local targetPart = p.animalData and findAdorneeGlobal(p.animalData)
                            if targetPart and targetPart:IsA("BasePart") then
                                local d = (hrp.Position - targetPart.Position).Magnitude
                                if d < bestDist then bestDist = d; bestIndex = i end
                            end
                        end
                        if bestIndex then
                            applySelection(bestIndex, pets)
                        else
                            SharedState.SelectedPetData = nil
                        end
                    else applySelection(1, pets) end
                else applySelection(1, pets) end
            else SharedState.SelectedPetData = nil end
        else SharedState.SelectedPetData = nil end
    end
end)

-- Heartbeat loop updating bottom HUD labels & executing steals
local function _pbFmtVal(num)
    if not num or num == 0 then return "0" end
    if num >= 1e9 then return string.format("%.2fb", num / 1e9) end
    if num >= 1e6 then return string.format("%.2fm", num / 1e6) end
    if num >= 1e3 then return string.format("%.1fk", num / 1e3) end
    return tostring(math.floor(num))
end

RunService.RenderStepped:Connect(function()
    if not (Config and Config.AutoStealEnabled) then
        hudName.Text = "Disabled"
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        return
    end

    if LocalPlayer:GetAttribute("Stealing") then
        hudProgressFill.Size = UDim2.new(1, 0, 1, 0)
        hudProgressFill.BackgroundColor3 = Theme.Green or Color3.fromRGB(80, 220, 120)
        hudPercent.Text = "100%"
        hudName.Text = "Carrying Brainrot!"
        return
    end

    local status = _G.ALPHA_StealStatus or {}
    if status.active then
        local p = math.clamp((tick() - (status.start or 0)) / (status.duration or 1.3), 0, 1)
        hudProgressFill.Size = UDim2.new(p, 0, 1, 0)
        hudPercent.Text = math.floor(p * 100) .. "%"
        hudProgressFill.BackgroundColor3 = (p >= 1) and (Theme.Green or Color3.fromRGB(80, 220, 120)) or (Theme.AccentLight or STEALBAR.FILL1)
        hudName.Text = "Stealing..."
    elseif status.target then
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = Theme.AccentLight or STEALBAR.FILL1
        local vs = _pbFmtVal(status.target.mps or status.target.value)
        hudName.Text = (status.target.name or "Brainrot") .. ((vs ~= "0") and (" - $" .. vs) or "")
    else
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = Theme.AccentLight or STEALBAR.FILL1
        hudName.Text = "Searching..."
    end
end)

end


-- ============================================================
-- PRIORITY ALERT (SXE Styled 3D Viewer with Sorting Queue)
-- ============================================================
local function ShowPriorityAlertImpl(brainrotName, genText, mutation, ownerUsername)
    local normalizedMutation = mutation and mutation:gsub("%s+", ""):lower() or ""
    
    local mutationColors = {
        ["gold"] = Color3.fromRGB(255, 222, 89),
        ["diamond"] = Color3.fromRGB(37, 196, 254),
        ["bloodrot"] = Color3.fromRGB(145, 0, 27),
        ["rainbow"] = Color3.fromRGB(255, 0, 0),
        ["candy"] = Color3.fromRGB(255, 0, 0),
        ["lava"] = Color3.fromRGB(255, 149, 0),
        ["galaxy"] = Color3.fromRGB(170, 60, 255),
        ["yinyang"] = Color3.fromRGB(255, 255, 255),
        ["radioactive"] = Color3.fromRGB(104, 245, 0),
        ["cursed"] = Color3.fromRGB(245, 56, 56),
        ["divine"] = Color3.fromRGB(255, 209, 59),
        ["cyber"] = Color3.fromRGB(121, 219, 255)
    }
    
    local ExploitGui = (gethui and gethui()) or game:GetService("CoreGui")
    local strokeColor = mutationColors[normalizedMutation] or Theme.AccentLight
    local txtColor = Theme.Text
    local subColor = mutationColors[normalizedMutation] or Theme.Dim
    
    local subText = ""
    if normalizedMutation ~= "" and normalizedMutation ~= "none" then
        subText = mutation:upper()
    else
        subText = "NORMAL"
    end
    
    local existing = ExploitGui:FindFirstChild("XiPriorityAlertTest")
    if existing then existing:Destroy() end
    
    if Config.PrioritySoundAlert and Config.PrioritySoundID and Config.PrioritySoundID ~= "" then
        pcall(function()
            local sid = Config.PrioritySoundID:match("%d+")
            if sid then
                local snd = Instance.new("Sound")
                snd.SoundId = "rbxassetid://" .. sid
                snd.Volume = 1
                snd.Parent = game:GetService("SoundService")
                snd:Play()
                game:GetService("Debris"):AddItem(snd, 5)
            end
        end)
    end
    
    local alertGui = Instance.new("ScreenGui")
    if _G.addLazyUI then _G.addLazyUI(alertGui, true, true) end
    alertGui.Name = "XiPriorityAlertTest"
    alertGui.ResetOnSpawn = false
    alertGui.DisplayOrder = 999
    alertGui.Parent = ExploitGui
    
    local alertFrame = Instance.new("Frame")
    alertFrame.Size = UDim2.new(0, 360, 0, 70)
    alertFrame.Position = UDim2.new(0.5, 0, 0, -100) -- Start off-screen top
    alertFrame.AnchorPoint = Vector2.new(0.5, 0)
    alertFrame.BackgroundColor3 = Theme.MainBackground
    alertFrame.BackgroundTransparency = 0.08
    alertFrame.BorderSizePixel = 0
    alertFrame.Parent = registerScreenGui(alertGui)
    
    local gradient = Instance.new("UIGradient", alertFrame)
    gradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Theme.MainBackground),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 244, 249))
    })
    gradient.Rotation = 90
    
    local corner = Instance.new("UICorner", alertFrame)
    corner.CornerRadius = UDim.new(0, 10)
    
    local stroke = Instance.new("UIStroke", alertFrame)
    stroke.Color = strokeColor
    stroke.Thickness = 1.5
    stroke.Transparency = 0.12
    
    local bottomAccent = Instance.new("Frame", alertFrame)
    bottomAccent.Size = UDim2.new(1, -24, 0, 3)
    bottomAccent.Position = UDim2.new(0.5, 0, 1, -4)
    bottomAccent.AnchorPoint = Vector2.new(0.5, 1)
    bottomAccent.BackgroundColor3 = strokeColor
    bottomAccent.BorderSizePixel = 0
    local bottomCorner = Instance.new("UICorner", bottomAccent)
    bottomCorner.CornerRadius = UDim.new(1, 0)
    
    -- 3D Model ViewportFrame on the left
    local viewportFrame = Instance.new("ViewportFrame", alertFrame)
    viewportFrame.Size = UDim2.new(0, 56, 0, 56)
    viewportFrame.Position = UDim2.new(0, 12, 0.5, -2)
    viewportFrame.AnchorPoint = Vector2.new(0, 0.5)
    viewportFrame.BackgroundTransparency = 1
    viewportFrame.BorderSizePixel = 0
    viewportFrame.Ambient = Color3.fromRGB(255, 255, 255)
    viewportFrame.LightColor = strokeColor
    
    local worldModel = Instance.new("WorldModel", viewportFrame)
    local camera = Instance.new("Camera", viewportFrame)
    viewportFrame.CurrentCamera = camera
    
    task.spawn(function()
        local modelsFolder = ReplicatedStorage:FindFirstChild("Models") and ReplicatedStorage.Models:FindFirstChild("Animals")
        local animationsFolder = ReplicatedStorage:FindFirstChild("Animations") and ReplicatedStorage.Animations:FindFirstChild("Animals")
        
        if modelsFolder and brainrotName then
            local sourceModel = modelsFolder:FindFirstChild(brainrotName)
            if not sourceModel then
                for _, child in pairs(modelsFolder:GetChildren()) do
                    if child.Name:lower() == brainrotName:lower() then
                        sourceModel = child
                        break
                    end
                end
            end
            
            if sourceModel then
                local modelClone = sourceModel:Clone()
                modelClone.Parent = worldModel
                
                local cframe, size = modelClone:GetBoundingBox()
                local maxAxis = math.max(size.X, size.Y, size.Z)
                
                if modelClone.PrimaryPart then
                    modelClone:SetPrimaryPartCFrame(CFrame.new(0, 0, 0))
                else
                    modelClone:MoveTo(Vector3.new(0, 0, 0))
                end
                
                -- Zoomed in camera offset
                local cameraOffset = Vector3.new(0, size.Y / 2, -(maxAxis * 0.85))
                camera.CFrame = CFrame.lookAt(cameraOffset, Vector3.new(0, 0, 0))
                
                if animationsFolder then
                    local animFolder = animationsFolder:FindFirstChild(sourceModel.Name)
                    local idleAnim = animFolder and animFolder:FindFirstChild("Idle")
                    if idleAnim and idleAnim:IsA("Animation") then
                        local animator = nil
                        local humanoid = modelClone:FindFirstChildOfClass("Humanoid")
                        local animController = modelClone:FindFirstChildOfClass("AnimationController")
 
                        if humanoid then
                            animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)
                        elseif animController then
                            animator = animController:FindFirstChildOfClass("Animator") or Instance.new("Animator", animController)
                        else
                            animController = Instance.new("AnimationController", modelClone)
                            animator = Instance.new("Animator", animController)
                        end
 
                        local animTrack = animator:LoadAnimation(idleAnim)
                        animTrack.Looped = true
                        animTrack:Play(0)
                        task.delay(0.01, function()
                            if animTrack.IsPlaying then
                                animTrack.TimePosition = 0.2
                            end
                        end)
                    end
                end
            end
        end
    end)
    
    local titleLabel = Instance.new("TextLabel", alertFrame)
    titleLabel.Size = UDim2.new(1, -85, 0, 20)
    titleLabel.Position = UDim2.new(0, 76, 0, 12)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = brainrotName .. " • " .. genText
    titleLabel.Font = Enum.Font.GothamBlack
    titleLabel.TextSize = 13
    titleLabel.TextColor3 = txtColor
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local subLabel = Instance.new("TextLabel", alertFrame)
    subLabel.Size = UDim2.new(1, -85, 0, 16)
    subLabel.Position = UDim2.new(0, 76, 0, 32)
    subLabel.BackgroundTransparency = 1
    subLabel.Text = subText .. " (Owner: " .. ownerUsername .. ")"
    subLabel.Font = Enum.Font.GothamBold
    subLabel.TextSize = 10
    subLabel.TextColor3 = subColor
    subLabel.TextXAlignment = Enum.TextXAlignment.Left

    TweenService:Create(alertFrame, TweenInfo.new(0.5, Enum.EasingStyle.Exponential, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 0, 0, 20)
    }):Play()
    
    task.delay(4, function()
        if alertFrame and alertFrame.Parent then
            TweenService:Create(alertFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.In), {
                Position = UDim2.new(0.5, 0, 0, -100)
            }):Play()
            task.wait(0.45)
            if alertGui and alertGui.Parent then alertGui:Destroy() end
        end
    end)
end

local highestAlertedPriorityIndex = 99999

task.spawn(function()
    while true do
        task.wait(1.5)
        pcall(function()
            local cache = SharedState.AllAnimalsCache
            if not cache or #cache == 0 then return end
            
            local bestPet = nil
            local bestPriorityIndex = 99999
            
            for _, pet in ipairs(cache) do
                local priorityIndex = nil
                for idx, pName in ipairs(priorityList) do
                    if pName:lower() == pet.name:lower() then
                        priorityIndex = idx
                        break
                    end
                end
                
                if priorityIndex then
                    if priorityIndex < bestPriorityIndex then
                        bestPriorityIndex = priorityIndex
                        bestPet = pet
                    end
                end
            end
            
            if bestPet and bestPriorityIndex < highestAlertedPriorityIndex then
                local myName = LocalPlayer.Name
                local myDisplay = LocalPlayer.DisplayName
                if bestPet.owner ~= myName and bestPet.owner ~= myDisplay then
                    highestAlertedPriorityIndex = bestPriorityIndex
                    ShowPriorityAlertImpl(bestPet.name, bestPet.genText, bestPet.mutation, bestPet.owner)
                end
            end
        end)
    end
end)

-- ============================================================
-- PRIORITY SCANNER
-- ============================================================
local PRIORITY_LIST = priorityList
task.spawn(function()
    local ok1,Packages=pcall(function() return ReplicatedStorage:WaitForChild("Packages",5) end); if not ok1 or not Packages then return end
    local ok2,Datas=pcall(function() return ReplicatedStorage:WaitForChild("Datas",5) end); if not ok2 or not Datas then return end
    local ok3,Shared=pcall(function() return ReplicatedStorage:WaitForChild("Shared",5) end); if not ok3 or not Shared then return end
    local ok4,Utils=pcall(function() return ReplicatedStorage:WaitForChild("Utils",5) end); if not ok4 or not Utils then return end
    local okS,Synchronizer=pcall(function() return require(Packages:WaitForChild("Synchronizer")) end); if not okS then return end
    local okA,AnimalsData=pcall(function() return require(Datas:WaitForChild("Animals")) end); if not okA then return end
    local okAS,AnimalsShared=pcall(function() return require(Shared:WaitForChild("Animals")) end); if not okAS then return end
    local okN,NumberUtils=pcall(function() return require(Utils:WaitForChild("NumberUtils")) end); if not okN then return end
    local allAnimalsCache={}; local lastAnimalData={}
    local function getAnimalHash(al) if not al then return "" end; local h=""; for slot,d in pairs(al) do if type(d)=="table" then h=h..tostring(slot)..tostring(d.Index)..tostring(d.Mutation) end end; return h end
    local function scanSinglePlot(plot) pcall(function()
        local ch=Synchronizer:Get(plot.Name); if not ch then return end
        local al=ch:Get("AnimalList"); local owner=ch:Get("Owner")
        if not owner or not owner.Name or not Players:FindFirstChild(owner.Name) then
            lastAnimalData[plot.Name]=nil; for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==plot.Name then table.remove(allAnimalsCache,i) end end; return end
        if not al then lastAnimalData[plot.Name]=nil; for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==plot.Name then table.remove(allAnimalsCache,i) end end; return end
        local ownerName=owner.Name; local hash=getAnimalHash(al)
        if lastAnimalData[plot.Name]==hash then return end
        
        for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==plot.Name then table.remove(allAnimalsCache,i) end end
        for slot,ad in pairs(al) do if type(ad)=="table" then
            local aName,aInfo=ad.Index,AnimalsData[ad.Index]; if aInfo then
                local mut=ad.Mutation or "None"; if mut=="Yin Yang" then mut="YinYang" end
                local traits=(ad.Traits and #ad.Traits>0) and table.concat(ad.Traits,", ") or "None"
                local gv=AnimalsShared:GetGeneration(aName,ad.Mutation,ad.Traits,nil)
                local gt="$"..NumberUtils:ToString(gv).."/s"
                local pv=0; pcall(function() pv=AnimalsShared:GetValue(aName,ad.Mutation,ad.Traits,nil) or 0 end)
                if type(pv)~="number" then pv=0 end
                table.insert(allAnimalsCache,{name=aInfo.DisplayName or aName,index=aName,genText=gt,genValue=gv,petValue=pv,mutation=mut,traits=traits,owner=ownerName,plot=plot.Name,slot=tostring(slot),uid=plot.Name.."_"..tostring(slot)})
            end
        end end
        lastAnimalData[plot.Name]=hash
        table.sort(allAnimalsCache,function(a,b) return a.genValue>b.genValue end)
        SharedState.AllAnimalsCache=allAnimalsCache; SharedState.ListNeedsRedraw=true
    end) end
    -- Brainrot scanner (brxken model): NO DescendantAdded/DescendantRemoving
    -- listeners. Those fired thousands of times while every plot's parts streamed
    -- in at startup -> that was the freeze/lag. Each plot instead gets ONE light
    -- 0.5s periodic rescan; scanSinglePlot is hash-gated, so it's a cheap no-op
    -- whenever a plot's AnimalList hasn't changed. retries softened (40 @ 0.07s)
    -- so the channel wait doesn't spin tight during load.
    local function setupPlotListener(plot) local ch; local retries=0
        while not ch and retries<40 do local ok,r=pcall(function() return Synchronizer:Get(plot.Name) end); if ok and r then ch=r; break else retries=retries+1; task.wait(0.07) end end
        if not ch then return end; scanSinglePlot(plot)
        task.spawn(function() while plot.Parent do task.wait(0.5); scanSinglePlot(plot) end end)
    end
    local plots=Workspace:WaitForChild("Plots",8)
    if plots then
        for _,p in ipairs(plots:GetChildren()) do
            task.spawn(setupPlotListener, p)
        end
        SharedState.InitialScanComplete=true
        plots.ChildAdded:Connect(function(p) task.wait(0.5); task.spawn(setupPlotListener, p) end)
        plots.ChildRemoved:Connect(function(p) lastAnimalData[p.Name]=nil; for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==p.Name then table.remove(allAnimalsCache,i) end end; SharedState.ListNeedsRedraw=true end)
    end
    task.spawn(function() while true do SharedState.AllAnimalsCache=allAnimalsCache; task.wait(0.5) end end)
end)

-- Steal attribute listeners
player:GetAttributeChangedSignal("Stealing"):Connect(function()
    local isStealing=(player:GetAttribute("Stealing")==true)
    if FloatState.active and not isStealing then setFloat(false) end
    if _G.AutoInvisDuringSteal then
        if isStealing and not _G.invisibleStealEnabled and _G._forceInvisToggle then task.defer(function() if player:GetAttribute("Stealing") and not _G.invisibleStealEnabled then pcall(_G._forceInvisToggle) end end)
        elseif not isStealing and _G.invisibleStealEnabled and _G._forceInvisToggle then task.wait(0.3); if not player:GetAttribute("Stealing") then pcall(_G._forceInvisToggle) end end
    end
    if isStealing and Config.AutoUnlockOnSteal then
        local hrp=player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local floor = getCurrentUnlockFloor()
            task.spawn(function()
                task.wait(0.1)
                pcall(smartInteract, floor)
            end)
        end
    end
end)

player.CharacterAdded:Connect(function() task.wait(0.5); if FloatState.active then removeFloatPlatform(); createFloatPlatform() end end)

-- ============================================================
-- TP SYSTEM
-- ============================================================
function findAdorneeGlobal(animalData)
    if not animalData then return nil end
    local plot = Workspace:FindFirstChild("Plots") and Workspace.Plots:FindFirstChild(animalData.plot)
    if plot then
        local podiums = plot:FindFirstChild("AnimalPodiums")
        if podiums then
            local podium = podiums:FindFirstChild(animalData.slot)
            if podium then
                local base = podium:FindFirstChild("Base")
                if base then
                    local spawn = base:FindFirstChild("Spawn")
                    if spawn then return spawn end
                    return base:FindFirstChildWhichIsA("BasePart") or base
                end
            end
        end
    end
    return nil
end

function getClosestBaseSign(brainrotPart)
    if not brainrotPart or not brainrotPart:IsA("BasePart") then return nil end
    local closestPart = nil
    local closestDist = math.huge
    for _, label in ipairs(Workspace:GetDescendants()) do
        if label:IsA("TextLabel") then
            local txt = tostring(label.Text or "")
            if (txt ~= "") and txt:lower():find("base", 1, true) then
                local gui = label:FindFirstAncestorWhichIsA("SurfaceGui")
                if gui then
                    local part = gui.Adornee or gui.Parent
                    if part and part:IsA("BasePart") then
                        local dist = (part.Position - brainrotPart.Position).Magnitude
                        if dist < closestDist then
                            closestDist = dist
                            closestPart = part
                        end
                    end
                end
            end
        end
    end
    return closestPart
end

function riseToY(hrp, targetY)
    if not hrp then return end
    local MAX_TIME = 3
    local start = os.clock()
    while hrp.Parent and (hrp.Position.Y < targetY) do
        local dist = targetY - hrp.Position.Y
        local speed = math.clamp(dist * 20, 280, 310)
        hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, speed, hrp.AssemblyLinearVelocity.Z)
        if (os.clock() - start) > MAX_TIME then
            break
        end
        RunService.Heartbeat:Wait()
    end
    hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, 0, hrp.AssemblyLinearVelocity.Z)
end

function equipTpToolAndWait(hum)
    if not hum then return nil end
    local char = hum.Parent
    local toolName = Config.TpSettings.Tool or "Flying Carpet"
    local tool = LocalPlayer.Backpack:FindFirstChild(toolName) or (char and char:FindFirstChild(toolName))
    if tool then
        hum:EquipTool(tool)
        task.wait(0.02)
    end
    return tool
end

function walkForward(seconds)
    local char = LocalPlayer.Character
    local hum = char:FindFirstChild("Humanoid")
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local pm = LocalPlayer:WaitForChild("PlayerScripts")
    local Controls = require(pm:WaitForChild("PlayerModule")):GetControls()
    local lookVector = hrp.CFrame.LookVector
    Controls:Disable()
    local startTime = os.clock()
    local conn
    conn = RunService.RenderStepped:Connect(function()
        if (os.clock() - startTime) >= seconds then
            conn:Disconnect()
            hum:Move(Vector3.zero, false)
            Controls:Enable()
            return
        end
        hum:Move(lookVector, false)
    end)
end

function waitSecondsHeartbeat(sec)
    local t = 0
    while t < sec do
        t += RunService.Heartbeat:Wait()
    end
end

function waitUntilHeartbeat(predicate, timeoutSec)
    local t = 0
    while true do
        if predicate() then
            return true
        end
        local dt = RunService.Heartbeat:Wait()
        t += dt
        if timeoutSec and (t >= timeoutSec) then
            return false
        end
    end
end

-- ============================================================
-- TELEPORT SYSTEM STAGE 2
-- ============================================================
TP_V2_MED_POINTS = {
    { name = "MED1", pos = Vector3.new(-410.65, -5.68, -46.1) },
    { name = "MED2", pos = Vector3.new(-410.91, -5.68, 168.89) },
}

TP_V2_SECOND_FLOOR_POINTS = {
    { name = "TP1", pos = Vector3.new(-488.88, 15, 196.38), facing = "back" },
    { name = "TP2", pos = Vector3.new(-487.79, 15, 138.13), facing = "front" },
    { name = "TP3", pos = Vector3.new(-489.38, 15, 89.23), facing = "back" },
    { name = "TP4", pos = Vector3.new(-489.69, 15, 30.98), facing = "front" },
    { name = "TP5", pos = Vector3.new(-488.75, 15, -17.95), facing = "back" },
    { name = "TP6", pos = Vector3.new(-490, 15, -75.9), facing = "front" },
    { name = "TP7", pos = Vector3.new(-331.75, 15, -75.8), facing = "back" },
    { name = "TP8", pos = Vector3.new(-329.98, 15, -18.16), facing = "front" },
    { name = "TP9", pos = Vector3.new(-330.04, 15, 31.14), facing = "back" },
    { name = "TP10", pos = Vector3.new(-331.28, 15, 88.92), facing = "front" },
    { name = "TP11", pos = Vector3.new(-330.57, 15, 138.1), facing = "back" },
    { name = "TP12", pos = Vector3.new(-330.01, 15, 195.96), facing = "front" },
}

TP_V2_ALLOWED_BY_MED = {
    MED1 = { TP6 = true, TP7 = true, TP8 = true, TP10 = true, TP12 = true, TP5 = true, TP3 = true, TP1 = true },
    MED2 = { TP1 = true, TP2 = true, TP4 = true, TP6 = true, TP7 = true, TP9 = true, TP11 = true, TP12 = true },
}

function flatDistance(a, b)
    return (Vector3.new(a.X, 0, a.Z) - Vector3.new(b.X, 0, b.Z)).Magnitude
end

_G._isTargetPlotUnlocked = function(plotName)
    local ok, res = pcall(function()
        local plots = Workspace:FindFirstChild("Plots")
        if not plots then return false end
        local targetPlot = plots:FindFirstChild(plotName)
        if not targetPlot then return false end
        local unlockFolder = targetPlot:FindFirstChild("Unlock")
        if not unlockFolder then return true end
        local unlockItems = {}
        for _, item in pairs(unlockFolder:GetChildren()) do
            local pos = nil
            if item:IsA("Model") then
                pcall(function() pos = item:GetPivot().Position end)
            elseif item:IsA("BasePart") then
                pos = item.Position
            end
            if pos then
                table.insert(unlockItems, { Object = item, Height = pos.Y })
            end
        end
        table.sort(unlockItems, function(a, b) return a.Height < b.Height end)
        if #unlockItems == 0 then return true end
        local floor1Door = unlockItems[1].Object
        for _, desc in ipairs(floor1Door:GetDescendants()) do
            if desc:IsA("ProximityPrompt") and desc.Enabled then return false end
        end
        for _, child in ipairs(floor1Door:GetChildren()) do
            if child:IsA("ProximityPrompt") and child.Enabled then return false end
        end
        return true
    end)
    return (ok and res) or false
end

function getClosestBaseSignToPosition(worldPos)
    local closestPart = nil
    local closestDist = math.huge
    for _, label in ipairs(Workspace:GetDescendants()) do
        if label:IsA("TextLabel") then
            local txt = tostring(label.Text or "")
            if (txt ~= "") and txt:lower():find("base", 1, true) then
                local gui = label:FindFirstAncestorWhichIsA("SurfaceGui")
                if gui then
                    local part = gui.Adornee or gui.Parent
                    if part and part:IsA("BasePart") then
                        local dist = (part.Position - worldPos).Magnitude
                        if dist < closestDist then
                            closestDist = dist
                            closestPart = part
                        end
                    end
                end
            end
        end
    end
    return closestPart
end

function getNearestTeleportV2MedPoint(fromPos)
    local bestPoint = nil
    local bestDist = math.huge
    for _, entry in ipairs(TP_V2_MED_POINTS) do
        local dist = flatDistance(fromPos, entry.pos)
        if dist < bestDist then
            bestDist = dist
            bestPoint = entry
        end
    end
    return bestPoint, bestDist
end

function getBestTeleportV2SecondFloorPoint(medPoint, brainrotPos)
    local medPos = medPoint and medPoint.pos
    local medName = medPoint and medPoint.name
    if not medPos or not medName then return nil end
    local allowed = TP_V2_ALLOWED_BY_MED[medName]
    if not allowed then return nil end
    local bestPoint = nil
    local bestMedDist = math.huge
    local bestBrainrotDist = math.huge
    for _, entry in ipairs(TP_V2_SECOND_FLOOR_POINTS) do
        if allowed[entry.name] then
            local distToMed = flatDistance(medPos, entry.pos)
            local distToBrainrot = flatDistance(brainrotPos, entry.pos)
            if (distToBrainrot < bestBrainrotDist) or ((math.abs(distToBrainrot - bestBrainrotDist) <= 0.001) and (distToMed < bestMedDist)) then
                bestPoint = entry
                bestMedDist = distToMed
                bestBrainrotDist = distToBrainrot
            end
        end
    end
    return bestPoint, bestMedDist, bestBrainrotDist
end



local FLY_SPEED = 160
local FLY_RISE_SPEED = 200
local FLY_RAY_DIST = 20
local FLY_TIMEOUT = 15
local FLY_ARRIVE_DIST = 4
local flyRayParams = RaycastParams.new()
flyRayParams.FilterType = Enum.RaycastFilterType.Exclude

function flyForwardTo(hrp, tpPos, lookDir, targetY, customSpeed)
    if not hrp or not hrp.Parent then
        return false
    end
    local char = hrp.Parent
    local hum = char:FindFirstChildOfClass("Humanoid")
    local unwalkConn
    if hum then
        local animator = hum:FindFirstChildOfClass("Animator")
        if animator then
            unwalkConn = RunService.Heartbeat:Connect(function()
                pcall(function()
                    for _, tr in ipairs(animator:GetPlayingAnimationTracks()) do
                        tr:Stop()
                    end
                end)
            end)
        end
    end
    flyRayParams.FilterDescendantsInstances = { char }
    local startTime = os.clock()
    while hrp.Parent do
        if (os.clock() - startTime) > FLY_TIMEOUT then
            break
        end
        if LocalPlayer:GetAttribute("Stealing") then
            break
        end
        local myPos = hrp.Position
        local flatDiff = Vector3.new(tpPos.X - myPos.X, 0, tpPos.Z - myPos.Z)
        local flatDist = flatDiff.Magnitude
        if flatDist < FLY_ARRIVE_DIST then
            break
        end
        local moveDir = flatDiff.Unit
        local yVel = 0
        if tpPos.Y <= 10 then
            -- Glue the feet to the floor by offsetting the HumanoidRootPart by +3.5 studs to prevent clipping into the ground and dying
            local targetFloorY = tpPos.Y + 3.5
            if targetY then
                -- Diagonal smooth ascent starting from 60 studs away down to 20 studs away
                if flatDist <= 60 then
                    local t = math.clamp((60 - flatDist) / (60 - 20), 0, 1)
                    targetFloorY = (tpPos.Y + 3.5) + (targetY - tpPos.Y) * t
                end
            end
            -- Multi-Ray Collision Detection (Feet, Torso, and Head levels)
            local heightsToCheck = {-3, 0, 3}
            local isRealObstacle = false
            for _, heightOffset in ipairs(heightsToCheck) do
                local rayOrigin = myPos + Vector3.new(0, heightOffset, 0)
                local rayFwd = Workspace:Raycast(rayOrigin, moveDir * FLY_RAY_DIST, flyRayParams)
                if rayFwd then
                    local part = rayFwd.Instance
                    if part and part.CanCollide then
                        local topY = part.Position.Y + (part.Size.Y / 2)
                        -- Only fly over if the obstacle's top is significantly higher than flat ground (more than 5 studs)
                        if (topY - tpPos.Y) > 5 then
                            isRealObstacle = true
                            break -- Tall obstacle detected, stop checking other heights
                        end
                    end
                end
            end

            if isRealObstacle then
                -- If tall obstacle detected, rise up and fly over it
                yVel = FLY_RISE_SPEED
            else
                -- Check if there is a real tall obstacle underneath us
                local rayDown = Workspace:Raycast(myPos, Vector3.new(0, -15, 0), flyRayParams)
                local obstacleUnderneath = false
                if rayDown then
                    local hitY = rayDown.Position.Y
                    if hitY > (tpPos.Y + 4) then
                        obstacleUnderneath = true
                    end
                end

                if obstacleUnderneath then
                    -- Keep current altitude while flying over the obstacle
                    yVel = 0
                else
                    -- No real obstacle in front or underneath, instantly stick to the ground
                    hrp.CFrame = CFrame.new(hrp.Position.X, targetFloorY, hrp.Position.Z) * (hrp.CFrame - hrp.CFrame.Position)
                    yVel = 0
                end
            end
        else
            -- Normal rise/fall behavior for second floor
            local rayFwd = Workspace:Raycast(myPos, moveDir * FLY_RAY_DIST, flyRayParams)
            if rayFwd then
                yVel = FLY_RISE_SPEED
            end
            local rayDown = Workspace:Raycast(myPos, Vector3.new(0, -12, 0), flyRayParams)
            if rayDown and ((myPos.Y - rayDown.Position.Y) < 5) then
                yVel = math.max(yVel, 100)
            end
            if not rayFwd and (tpPos.Y < (myPos.Y - 3)) then
                yVel = math.clamp((tpPos.Y - myPos.Y) * 3, -120, 0)
            end
        end
        local speed = customSpeed or Config.TpSettings.FlyTPSpeed or FLY_SPEED
        hrp.AssemblyLinearVelocity = (moveDir * speed) + Vector3.new(0, yVel, 0)
        hrp.AssemblyAngularVelocity = Vector3.zero
        RunService.Heartbeat:Wait()
    end
    if unwalkConn then
        unwalkConn:Disconnect()
        unwalkConn = nil
    end
    if not hrp.Parent then
        return false
    end
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.AssemblyAngularVelocity = Vector3.zero
    local finalY = targetY or tpPos.Y
    local finalPos = Vector3.new(tpPos.X, finalY, tpPos.Z)
    hrp.CFrame = CFrame.lookAt(finalPos, finalPos + (lookDir or hrp.CFrame.LookVector))
    hrp.AssemblyLinearVelocity = Vector3.zero
    return true
end

function prepMiniTpTool(hum, hrp)
    if not hum or not hrp then return end
end

local function getTargetPetData()
    local cache = SharedState.AllAnimalsCache
    if not cache or #cache == 0 then
        return nil
    end
    
    local pets = get_all_pets()
    
    -- 1) Manual Override First
    if manuallySelectedUID then
        for _, a in ipairs(cache) do
            if a.uid == manuallySelectedUID and a.owner ~= LocalPlayer.Name then
                return a
            end
        end
    end
    
    -- 1.5) Priority List FIRST (Always target priority pets before anything else)
    for _, pName in ipairs(priorityList) do
        local searchName = pName:lower()
        local bestPet = nil
        local bestDist = math.huge
        for _, a in ipairs(cache) do
            if a and a.name and ((a.name:lower() == searchName) or (a.index and a.index:lower() == searchName)) and (a.owner ~= LocalPlayer.Name) then
                local dist = math.huge
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local plots = Workspace:FindFirstChild("Plots")
                    local plotInst = plots and plots:FindFirstChild(a.plot)
                    if plotInst then
                        local pos = nil
                        pcall(function() pos = plotInst:GetPivot().Position end)
                        if pos then
                            dist = (hrp.Position - pos).Magnitude
                        end
                    end
                end
                
                if not bestPet then
                    bestPet = a
                    bestDist = dist
                else
                    local currentGen = a.genValue or 0
                    local bestGen = bestPet.genValue or 0
                    if currentGen > bestGen then
                        bestPet = a
                        bestDist = dist
                    elseif currentGen == bestGen then
                        if dist < bestDist then
                            bestPet = a
                            bestDist = dist
                        end
                    end
                end
            end
        end
        if bestPet then
            return bestPet
        end
    end

    -- 2) Brainrots Second (target brainrots after priority)
    -- Respect Min-Gen-for-TP here too. Without this, brainrots BELOW the configured
    -- min gen (but still over the 10M "is-a-brainrot" baseline) got teleported to
    -- even though the list/scan correctly excluded them. Priority pets above are
    -- intentionally exempt from min gen; this non-priority section is not.
    local _mgStr = (Config.StealNearest and Config.TpSettings.MinGenForGrab) or Config.TpSettings.MinGenForTp
    local _brainrotFloor = math.max(10000000, parseMinGen(_mgStr) or 0)
    local bestBrainrot = nil
    local bestBrainrotDist = math.huge
    for _, a in ipairs(cache) do
        if a and a.owner ~= LocalPlayer.Name and a.genValue and a.genValue >= _brainrotFloor then
            local dist = math.huge
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                local plots = Workspace:FindFirstChild("Plots")
                local plotInst = plots and plots:FindFirstChild(a.plot)
                if plotInst then
                    local pos = nil
                    pcall(function() pos = plotInst:GetPivot().Position end)
                    if pos then dist = (hrp.Position - pos).Magnitude end
                end
            end
            if dist < bestBrainrotDist then
                bestBrainrotDist = dist
                bestBrainrot = a
            end
        end
    end
    if bestBrainrot then return bestBrainrot end

    -- 3) Auto TP Highest Gen
    if Config.AutoTPHighestGen then
        if pets and #pets > 0 then
            return pets[1].animalData
        end
    end
    
    -- 2b) Auto TP Highest Value (sorted by pet value, not gen)
    if Config.AutoTPHighestValue then
        local valuePets = get_all_pets_by_value()
        if valuePets and #valuePets > 0 then
            return valuePets[1].animalData
        end
    end
    
    -- 3b) Auto TP Priority mode fallback (only reached if no priority pet was found)
    -- If they specifically selected priority mode but no priority pet is on the server, we return the highest gen fallback
    if Config.AutoTPPriority then
        if pets and #pets > 0 then
            return pets[1].animalData
        end
        return nil
    end
    
    -- 4) General Selected Pet
    if SharedState.SelectedPetData then
        return SharedState.SelectedPetData.animalData
    end
    
    return nil
end

local doGrabbleVelocityTP
do
local UPPER = {
    B = {{coord=Vector3.new(-487.921448,16.850713,-75.768013),facing="NORTH"},{coord=Vector3.new(-332.379730,16.850722,-75.762100),facing="NORTH"},{coord=Vector3.new(-487.134918,16.850713,-18.094154),facing="SOUTH"},{coord=Vector3.new(-316.300171,16.850713,-17.845898),facing="SOUTH"}},
    C = {{coord=Vector3.new(-330.765381,16.850713,31.424425),facing="NORTH"},{coord=Vector3.new(-502.989349,16.850713,31.172430),facing="NORTH"},{coord=Vector3.new(-489.077087,16.850713,89.010147),facing="SOUTH"},{coord=Vector3.new(-330.908936,16.850713,88.930145),facing="SOUTH"}},
    D = {{coord=Vector3.new(-331.264893,16.850713,138.209167),facing="NORTH"},{coord=Vector3.new(-487.935181,16.850713,138.026321),facing="NORTH"},{coord=Vector3.new(-487.774933,16.850713,195.882538),facing="SOUTH"},{coord=Vector3.new(-330.799133,16.850575,196.022354),facing="SOUTH"}},
}
local LOWER = {
    B = {{coord=Vector3.new(-335.725586,-3.048217,-74.984589),facing="NORTH"},{coord=Vector3.new(-503.214233,-3.048217,-75.043137),facing="NORTH"},{coord=Vector3.new(-483.619385,-3.718430,-18.844337),facing="SOUTH"},{coord=Vector3.new(-316.147095,-3.048218,-18.818844),facing="SOUTH"}},
    C = {{coord=Vector3.new(-335.985413,-3.048218,32.051426),facing="NORTH"},{coord=Vector3.new(-503.277008,-3.048217,31.956175),facing="NORTH"},{coord=Vector3.new(-483.749390,-3.048218,88.147003),facing="SOUTH"},{coord=Vector3.new(-315.793823,-3.048217,88.163979),facing="SOUTH"}},
    D = {{coord=Vector3.new(-335.476654,-3.048218,139.001083),facing="NORTH"},{coord=Vector3.new(-503.710083,-3.048218,138.989883),facing="NORTH"},{coord=Vector3.new(-315.654938,-3.048218,195.302444),facing="SOUTH"},{coord=Vector3.new(-483.859253,-3.048218,195.269043),facing="SOUTH"}},
}
local UPPER_Y_THRESHOLD = 7
local TALL_PETS = { ["La Secret Combinasion"]=true, ["La Jolly Grande"]=true }
local TALL_OFFSET = 3

local CARPET_SPEED = 230
local INBASE_SPEED = 230
local function getCarpetSpeed()
    return Config.TpSettings.FlyTPSpeed or 230
end
local function getInBaseSpeed()
    return Config.TpSettings.FlyTPCloseSpeed or 230
end
local SKY_CLONE_WAIT = 0.2
local CARPET_NAMES = { "Flying Carpet", "Carpet", "Cloud", "Witch's Broom", "Cupid's Wings", "Santa's Sleigh", "Magic Carpet", "Waverider" }
local GRAPPLE_NAMES = { "Grapple Hook", "Grappling Hook", "Grapple", "Hook", "Web Slinger", "Grapple Gun", "GrappleHook" }

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS        = game:GetService("UserInputService")
local RS         = game:GetService("ReplicatedStorage")
local LP         = Players.LocalPlayer

_G.AntiDieDisabled = false
do
    local _conn, _diedConn, _hbConn
    local _harden = function(hum)
        pcall(function() hum.BreakJointsOnDeath = false end)
        pcall(function() hum.RequiresNeck = false end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false) end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false) end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false) end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Physics, false) end)
    end
    local function _revive(hum)
        pcall(function() hum.Health = hum.MaxHealth end)
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
    end
    local function _bind()
        local char = LP.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return end
        _harden(hum)
        if _conn then pcall(function() _conn:Disconnect() end) end
        if _diedConn then pcall(function() _diedConn:Disconnect() end) end
        if _hbConn then pcall(function() _hbConn:Disconnect() end) end
        _conn = hum:GetPropertyChangedSignal("Health"):Connect(function()
            if _G.AntiDieDisabled then return end
            if hum.Health <= 0 then _revive(hum) end
        end)
        _diedConn = hum.Died:Connect(function()
            if _G.AntiDieDisabled then return end
            _revive(hum)
        end)
        local _lastHarden = 0
        _hbConn = RunService.Heartbeat:Connect(function()
            if _G.AntiDieDisabled or not hum or not hum.Parent then return end
            local now = os.clock()
            if now - _lastHarden >= 0.5 then _lastHarden = now; _harden(hum) end
            if hum.Health <= 0 then _revive(hum) end
            local state = hum:GetState()
            if state == Enum.HumanoidStateType.Dead or state == Enum.HumanoidStateType.Ragdoll
               or state == Enum.HumanoidStateType.FallingDown then
                pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
            end
        end)
    end
    _bind()
    LP.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid", 5)
        if hum then _harden(hum) end
        task.wait(0.1)
        _bind()
    end)
end

-- =====================================================================
-- Synchronizer detection bypass
-- =====================================================================
do
    local okReq, syn = pcall(require, RS:FindFirstChild("Packages"):FindFirstChild("Synchronizer"))
    if okReq and typeof(syn) == "table" then
        local function HasBoolUpvalue(Fn)
            local OkU, Ups = xpcall(debug.getupvalues, function() end, Fn)
            if not OkU then return false end
            for _, V in pairs(Ups) do
                if typeof(V) == "boolean" then return true end
            end
            return false
        end
        for _, Fn in pairs(syn) do
            if typeof(Fn) == "function" and not isexecutorclosure(Fn) then
                local OkU, Ups = xpcall(debug.getupvalues, function() end, Fn)
                if OkU then
                    for Idx, V in pairs(Ups) do
                        if typeof(V) == "function" and not isexecutorclosure(V) and HasBoolUpvalue(V) then
                            pcall(debug.setupvalue, Fn, Idx, newcclosure(function() end))
                        end
                    end
                end
            end
        end
    end
end



-- =====================================================================
-- Module loaders
-- =====================================================================
local Synchronizer, AnimalsData, AnimalsShared, NumberUtils

local function loadModules()
    if Synchronizer then return true end
    local ok = pcall(function()
        local Packages = RS:WaitForChild("Packages", 5)
        local Datas = RS:WaitForChild("Datas", 5)
        local Shared = RS:WaitForChild("Shared", 5)
        local Utils = RS:WaitForChild("Utils", 5)
        Synchronizer = require(Packages:WaitForChild("Synchronizer"))
        AnimalsData = require(Datas:WaitForChild("Animals"))
        AnimalsShared = require(Shared:WaitForChild("Animals"))
        NumberUtils = require(Utils:WaitForChild("NumberUtils"))
    end)
    return ok and Synchronizer ~= nil
end

local NetModule
local function loadNet()
    if NetModule then return true end
    local ok, mod = pcall(function()
        return require(RS:WaitForChild("Packages", 5):WaitForChild("Net", 5):FindFirstChildWhichIsA("ModuleScript", true))
    end)
    if not ok or type(mod) ~= "table" then return false end
    NetModule = mod
    return true
end

local function findTool(name)
    local char = LP.Character
    local bp = LP:FindFirstChild("Backpack")
    return (char and char:FindFirstChild(name)) or (bp and bp:FindFirstChild(name))
end

local function findGrapple()
    for _, n in ipairs(GRAPPLE_NAMES) do
        local t = findTool(n)
        if t and t:IsA("Tool") then return t, n end
    end
    return nil
end

local function equipCarpet()
    local char = LP.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hum then return nil end
    local preferred = Config and Config.TpSettings and Config.TpSettings.Tool
    if preferred then
        local pt = findTool(preferred)
        if pt and pt:IsA("Tool") then
            if pt.Parent ~= char then pcall(function() hum:EquipTool(pt) end) end
            return preferred
        end
    end
    for _, n in ipairs(CARPET_NAMES) do
        local t = findTool(n)
        if t and t:IsA("Tool") then
            if t.Parent ~= char then pcall(function() hum:EquipTool(t) end) end
            return n
        end
    end
    return nil
end

local function fireGrapple()
    if not NetModule then loadNet() end
    if not NetModule then return end
    local char = LP.Character
    if not char then return end
    if not char:FindFirstChild("Grapple Hook") then
        local bp = LP:FindFirstChild("Backpack")
        local tool = bp and bp:FindFirstChild("Grapple Hook")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if tool and hum then pcall(function() hum:EquipTool(tool) end) end
    end
    if not char:FindFirstChild("Grapple Hook") then return end
    pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end)
end

local function carpetEngage()
    if not NetModule then pcall(loadNet) end
    local char = LP.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not char or not hum then return nil end
    local g = findTool("Grapple Hook")
    if g then
        if g.Parent ~= char then 
            pcall(function() hum:EquipTool(g) end)
            task.wait(0.05) -- Let tool equip register
        end
        if NetModule then pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end) end
        task.wait(0.22) -- Let grapple projectile register and pull
    end
    pcall(function() hum:UnequipTools() end)
    task.wait(0.10) -- Wait before putting carpet on
    local cn = equipCarpet()
    _G.TPEngage = "carpet=" .. tostring(cn)
    return cn
end

-- =====================================================================
-- Pet priority data tables
-- =====================================================================
local PET_PRIORITY_TIERS = {
    [1] = { pets = {"Headless Horseman"}, threshold = 0 },
    [2] = { pets = {"Signore Carapace"}, threshold = 0 },
    [3] = { pets = {"John Pork"}, threshold = 0 },
    [4] = { pets = {"Strawberry Elephant"}, threshold = 0 },
    [5] = { pets = {"Arcadragon"}, threshold = 5e9 },
    [6] = { pets = {"Elefanto Frigo"}, threshold = 10e9 },
    [7] = { pets = {"Meowl"}, threshold = 5e9 },
    [8] = { pets = {"Skibidi Toilet"}, threshold = 5e9 },
    [9] = { pets = {"Love Love Bear"}, threshold = 0 },
    [10] = { pets = {"Antonio"}, threshold = 0 },
    [11] = { pets = {"Pancake and Syrup"}, threshold = 0 },
    [12] = { pets = {"Griffin"}, threshold = 0 },
    [13] = { pets = {"Globa Steppa","La Supreme Combinasion","Fishino Clownino","Dragon Gingerini","Tirilikalika Tirilikalako"}, threshold = 5e9 },
    [14] = { pets = {"Ginger Gerat","Pet"}, threshold = 10e9 },
    [15] = { pets = {"Hydra Bunny","Digi Narwhal","Kalika Bros"}, threshold = 3e9 },
    [16] = { pets = {"Hydra Dragon Cannelloni","Dragon Cannelloni","Bunny and Eggy"}, threshold = 3e9 },
    [17] = { pets = {"Ketupat Bros","Rosey and Teddy","La Casa Boo","Fragola la la"}, threshold = 3e9 },
    [18] = { pets = {"Fragola La La La","Cerberus","Guest 666","Los Hackers"}, threshold = 1e9 },
    [19] = { pets = {"Garama and Madundung","Spooky and Pumpky","Reinito Sleighito","Burguro And Fryuro","Cooki and Milki","Fragrama and Chocrama","La Food Combinasion","Los Amigos","Foxini Lanternini","Capitano Moby","Fortunu and Cashuru","Los Sekolahs","Celestial Pegasus"}, threshold = 750e6 },
    [20] = { pets = {"La Secret Combinasion","Sammyni Fattini","Cloverat Clapat","Popcuru and Fizzuru"}, threshold = 1e9 },
}

local TIER_LOOKUP = {}
for tier, data in pairs(PET_PRIORITY_TIERS) do
    for _, name in ipairs(data.pets) do TIER_LOOKUP[name] = tier end
end

local LOCKED_TIERS = { [1]=true, [2]=true, [3]=true, [4]=true }

local DIRECT_THRESHOLDS = {
    [3] = { [4] = 10e9 },
    [4] = {},
    [5] = { [6] = math.huge },
    [6] = { [9] = math.huge, [10] = math.huge, [12] = 15e9 },
    [10] = { [12] = 20e9 },
    [11] = { [12] = 10e9 },
}

local MUTATION_PRIORITY = {
    ["Galaxy"]=1,["Candy"]=1,["Yin Yang"]=1,["YinYang"]=1,["Divine"]=1,
    ["Cursed"]=1,["Lava"]=1,["Radioactive"]=1,["Cyber"]=1,["Rainbow"]=1,["Bloodrot"]=2,
}

local MUTATED_BEATS_GRIFFIN = {
    ["Fishino Clownino"]=true,["Globa Steppa"]=true,
    ["La Supreme Combinasion"]=true,["Tirilikalika Tirilikalako"]=true,
}

local function getMutPrio(m)
    if not m or m == "" or m == "None" then return 0 end
    if MUTATION_PRIORITY[m] then return MUTATION_PRIORITY[m] end
    local n = tostring(m):lower():gsub("[%s%-_]","")
    if n == "bloodrot" then return 2 end
    if n == "yinyang" or n == "galaxy" or n == "candy" or n == "divine"
        or n == "cursed" or n == "lava" or n == "radioactive" or n == "cyber"
        or n == "rainbow" then return 1 end
    return 0
end

local function getCumThreshold(hi, lo)
    if DIRECT_THRESHOLDS[hi] and DIRECT_THRESHOLDS[hi][lo] then return DIRECT_THRESHOLDS[hi][lo] end
    if LOCKED_TIERS[hi] then return math.huge end
    local total = 0
    for t = hi + 1, lo do
        local td = PET_PRIORITY_TIERS[t]
        if td and td.threshold > 0 then total = total + td.threshold end
    end
    return total
end

local function petOutranks(aName, bName, aMut, bMut, aMPS, bMPS)
    if aName == "Strawberry Elephant" and bName == "John Pork" then return true end
    if aName == "John Pork" and bName == "Strawberry Elephant" then return false end

    if MUTATED_BEATS_GRIFFIN[aName] and bName == "Griffin" and getMutPrio(aMut) >= 1 then return true end
    if aName == "Griffin" and MUTATED_BEATS_GRIFFIN[bName] and getMutPrio(bMut) >= 1 then return false end

    if aName == "Antonio" and bName == "Elefanto Frigo" and getMutPrio(aMut) >= 1 then return true end
    if aName == "Elefanto Frigo" and bName == "Antonio" and getMutPrio(bMut) >= 1 then return false end

    local tA = TIER_LOOKUP[aName] or 99
    local tB = TIER_LOOKUP[bName] or 99

    if not (TIER_LOOKUP[aName] and TIER_LOOKUP[bName]) then
        if tA == tB then return (aMPS or 0) > (bMPS or 0) end
        return tA < tB
    end

    if tA == tB then
        local pA, pB = getMutPrio(aMut), getMutPrio(bMut)
        if pA ~= pB then return pA > pB end
        return (aMPS or 0) > (bMPS or 0)
    end

    if tA == 4 and tB == 3 then return true end
    if tA == 3 and tB == 4 then return false end

    local hi = math.min(tA, tB)
    local lo = math.max(tA, tB)
    local hiMPS = tA < tB and aMPS or bMPS
    local loMPS = tA < tB and bMPS or aMPS
    local cum = getCumThreshold(hi, lo)
    if cum > 0 and cum ~= math.huge then
        if (loMPS or 0) - (hiMPS or 0) > cum then return tA > tB end
    end
    return tA < tB
end

-- =====================================================================
-- Plot / channel helpers
-- =====================================================================
local function getPlotChannel(plotName)
    if not Synchronizer then return nil end
    local channel
    pcall(function() channel = Synchronizer:Get(plotName) end)
    if not channel then pcall(function() channel = Synchronizer:Wait(plotName) end) end
    return channel
end

local function channelGet(channel, key)
    return _G.sProp(channel, key)
end

local function isMyPlot(channel)
    if not channel then return false end
    local owner = channelGet(channel, "Owner")
    if not owner then return false end
    local result = false
    pcall(function()
        if typeof(owner) == "Instance" and owner:IsA("Player") then
            result = owner.UserId == LP.UserId
        elseif type(owner) == "table" and owner.UserId then
            result = owner.UserId == LP.UserId
        elseif typeof(owner) == "Instance" then
            result = owner == LP
        elseif type(owner) == "string" then
            result = (owner:lower() == LP.Name:lower() or owner:lower() == LP.DisplayName:lower())
        elseif type(owner) == "number" then
            result = owner == LP.UserId
        end
    end)
    return result
end

local function ownerInGame(channel)
    if not channel then return false end
    local owner = channelGet(channel, "Owner")
    if not owner then return false end
    local inGame = false
    pcall(function()
        if typeof(owner) == "Instance" and owner:IsA("Player") then
            inGame = Players:FindFirstChild(owner.Name) ~= nil
        elseif type(owner) == "number" then
            inGame = Players:GetPlayerByUserId(owner) ~= nil
        elseif type(owner) == "table" and owner.Name then
            inGame = Players:FindFirstChild(tostring(owner.Name)) ~= nil
        elseif type(owner) == "string" then
            inGame = Players:FindFirstChild(owner) ~= nil
        elseif typeof(owner) == "Instance" and owner.Name then
            inGame = Players:FindFirstChild(owner.Name) ~= nil
        end
    end)
    return inGame
end

local function getStealPromptForSlot(plot, slot)
    local podiums = plot and plot:FindFirstChild("AnimalPodiums")
    local podium = podiums and podiums:FindFirstChild(tostring(slot))
    local base = podium and podium:FindFirstChild("Base")
    local spawnp = base and base:FindFirstChild("Spawn")
    local att = spawnp and spawnp:FindFirstChild("PromptAttachment")
    local prompt = att and att:FindFirstChildWhichIsA("ProximityPrompt")
    if prompt and (prompt.ActionText:lower():find("steal") or prompt.ActionText == "") then return prompt end
    return nil
end

local _BLOCKING_MACHINE_TYPES = {
    Fuse     = true,
    Duel     = true,
    Trade    = true,
    Crafting = true,
}
local function _CartisIsFusing(animalData)
    if type(animalData) ~= "table" then return false end
    local m = animalData.Machine
    if type(m) ~= "table" then return false end
    return _BLOCKING_MACHINE_TYPES[m.Type] == true
end

local function scanAllPets()
    local pets = {}
    if not loadModules() then return pets end

    local Plots = Workspace:FindFirstChild("Plots")
    if not Plots then return pets end

    for _, plot in ipairs(Plots:GetChildren()) do
        local channel = getPlotChannel(plot.Name)
        if not channel then continue end
        if isMyPlot(channel) then continue end
        -- if not ownerInGame(channel) then continue end

        local animalList = channelGet(channel, "AnimalList")
        if not animalList then continue end

        for slot, animalData in pairs(animalList) do
            if type(animalData) ~= "table" then continue end
            local animalName = animalData.Index
            if not animalName then continue end
            local animalInfo = AnimalsData and AnimalsData[animalName]
            if not animalInfo then continue end
            if _CartisIsFusing(animalData) then continue end

            local mutation = animalData.Mutation or "None"
            local genValue = 0
            pcall(function()
                genValue = AnimalsShared:GetGeneration(animalName, animalData.Mutation, animalData.Traits, nil)
            end)

            local displayName = (animalInfo and animalInfo.DisplayName) or animalName
            local pos = getPetPosition(plot, slot)

            if pos then
                table.insert(pets, {
                    name = displayName,
                    mps = genValue,
                    mutation = mutation,
                    position = pos,
                    plot = plot.Name,
                    slot = tostring(slot),
                    prompt = getStealPromptForSlot(plot, slot),
                })
            end
        end
    end

    table.sort(pets, function(a, b)
        return petOutranks(a.name, b.name, a.mutation, b.mutation, a.mps, b.mps)
    end)

    return pets
end

local function findClosest(petPos, coordTable, fromPos)
    local all = {}
    for skyKey, coords in pairs(coordTable) do
        for _, data in ipairs(coords) do
            local c = data.coord
            local d = (petPos.X - c.X)^2 + (petPos.Z - c.Z)^2
            all[#all + 1] = { data = data, key = skyKey, petD = d }
        end
    end
    table.sort(all, function(a, b) return a.petD < b.petD end)
    if #all == 0 then return nil, nil end
    if not fromPos then return all[1].data, all[1].key end

    local cands = { all[1] }
    if all[2] and math.abs(all[2].data.coord.X - all[1].data.coord.X) < 40 then
        cands[#cands + 1] = all[2]
    end
    local best, bestKey, bestDist = all[1].data, all[1].key, math.huge
    for _, e in ipairs(cands) do
        local c = e.data.coord
        local d = (fromPos.X - c.X)^2 + (fromPos.Z - c.Z)^2
        if d < bestDist then bestDist = d; best = e.data; bestKey = e.key end
    end
    return best, bestKey
end

-- =====================================================================
-- Path visualization helpers
-- =====================================================================
local _vizParts = {}
local function clearViz()
    for _, p in ipairs(_vizParts) do if p and p.Parent then p:Destroy() end end
    table.clear(_vizParts)
end
local function vizLine(a, b, color)
    local d = b - a
    if d.Magnitude < 0.05 then return end
    local p = Instance.new("Part")
    p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CastShadow = false
    p.Material = Enum.Material.Neon; p.Color = color
    p.Size = Vector3.new(0.4, 0.4, d.Magnitude)
    p.CFrame = CFrame.new((a + b) / 2, b)
    p.Parent = Workspace
    _vizParts[#_vizParts + 1] = p
end
local function vizDot(pos, color, sz)
    local p = Instance.new("Part")
    p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CastShadow = false
    p.Shape = Enum.PartType.Ball; p.Material = Enum.Material.Neon; p.Color = color
    p.Size = Vector3.new(sz, sz, sz)
    p.Position = pos
    p.Parent = Workspace
    _vizParts[#_vizParts + 1] = p
end
local function vizPath(fromPos, waypoints)
    -- Pathfinder lines/dots disabled (user request: no lines).
    return
end

local isGrabbleTeleporting = false
local isTeleporting = false

local function getPlotKey(plotName)
    if not plotName then return nil end
    local first, second = plotName:match("^Plot_([B-D])([1-4])$")
    if first and second then return first end
    local first2, second2 = plotName:match("^Plot([B-D])([1-4])$")
    if first2 and second2 then return first2 end
    local first3 = plotName:match("^([B-D])[1-4]$")
    if first3 then return first3 end
    return nil
end

local SPEED = 200
local ARRIVE = 3

local function vZero(hrp)
    if hrp then hrp.AssemblyLinearVelocity = Vector3.zero; hrp.AssemblyAngularVelocity = Vector3.zero end
end

local MAX_CLIMB = 60

local function velMoveThrough(hrp, waypoints, speedOverride, allowJump, quickStart)
    if not hrp or not hrp.Parent or #waypoints == 0 then return end
    local _runSpeed = speedOverride or Config.TpSettings.GrabbleTPSpeed or (_G.SXECarpetSpeed or CARPET_SPEED or 230)
    vizPath(hrp.Position, waypoints)
    local wpIdx = 1
    local done = false
    local conn
    local function finish()
        if done then return end
        done = true
        if hrp and hrp.Parent then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            local _, y = hrp.CFrame:ToEulerAnglesYXZ()
            hrp.CFrame = CFrame.new(waypoints[#waypoints]) * CFrame.Angles(0, y, 0)
        end
        if conn then conn:Disconnect() end
        clearViz()
    end
    local lastDist, stall = math.huge, 0

    if quickStart then
        local _hp = RaycastParams.new()
        _hp.FilterType = Enum.RaycastFilterType.Exclude
        _hp.IgnoreWater = true
        local _skip = {}
        for _, pl in ipairs(Players:GetPlayers()) do
            if pl.Character then _skip[#_skip + 1] = pl.Character end
        end
        _hp.FilterDescendantsInstances = _skip
        for _ = 1, 3 do
            local target = waypoints[wpIdx]
            if not target then break end
            local flat = Vector3.new(target.X - hrp.Position.X, 0, target.Z - hrp.Position.Z)
            local mag = flat.Magnitude
            if mag < 1 then break end
            local nextPos = hrp.Position + flat.Unit * math.min(20, mag)
            local _hit = Workspace:Raycast(hrp.Position, nextPos - hrp.Position, _hp)
            if _hit and _hit.Instance and _hit.Instance.CanCollide then break end
            hrp.CFrame = (hrp.CFrame - hrp.CFrame.Position) + nextPos
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            RunService.Heartbeat:Wait()
            if not hrp or not hrp.Parent then return end
        end
    end

    conn = RunService.Heartbeat:Connect(function()
        if not hrp or not hrp.Parent or done then
            if conn then conn:Disconnect() end
            return
        end
        equipCarpet()
        local target = waypoints[wpIdx]
        local diff = target - hrp.Position
        local mag = diff.Magnitude
        if mag < ARRIVE then
            wpIdx = wpIdx + 1
            if wpIdx > #waypoints then finish() return end
            lastDist, stall = math.huge, 0
            target = waypoints[wpIdx]
            diff = target - hrp.Position
            mag = diff.Magnitude
        end
        if mag > lastDist - 0.05 then stall = stall + 1 else stall = 0 end
        lastDist = mag
        if stall >= 18 then finish() return end
        if mag >= 0.1 then
            local dir = diff.Unit
            if allowJump and diff.Y > 5 and wpIdx < #waypoints then
                local hum = hrp.Parent and hrp.Parent:FindFirstChildOfClass("Humanoid")
                if hum then
                    local st = hum:GetState()
                    if st ~= Enum.HumanoidStateType.Jumping and st ~= Enum.HumanoidStateType.Freefall then
                        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end)
                        pcall(function() hum.Jump = true end)
                    end
                end
            end
            local _sp = _runSpeed
            local _vy = dir.Y * _sp
            if _vy > MAX_CLIMB then _vy = MAX_CLIMB end
            hrp.Velocity = Vector3.new(dir.X * _sp, _vy, dir.Z * _sp)
        end
    end)
    local totalDist = 0
    local prev = hrp.Position
    for _, wp in ipairs(waypoints) do
        totalDist = totalDist + (prev - wp).Magnitude
        prev = wp
    end
    local timeout = totalDist / math.min(SPEED, _runSpeed) + 2
    local elapsed = 0
    while not done and elapsed < timeout do
        task.wait(0.05)
        elapsed = elapsed + 0.05
    end
    finish()
    vZero(hrp)
end

-- =====================================================================
-- Raycast / route-pulling helpers
-- =====================================================================
local _DIRS = { Vector3.new(1,0,0), Vector3.new(-1,0,0), Vector3.new(0,0,1), Vector3.new(0,0,-1) }
local _STRUCT = { ["structure base home"] = true, ["Wall"] = true, ["Floor"] = true, ["Roof"] = true }
local _SKIP_NAME = { ["DeliveryHitbox"]=true, ["StealHitbox"]=true, ["LaserHitbox"]=true,
    ["AnimalTarget"]=true, ["Multiplier"]=true, ["Laser"]=true, ["Hitbox"]=true,
    ["Spawn"]=true, ["MainRoot"]=true, ["SecondFloor"]=true, ["ThirdFloor"]=true, ["Slope"]=true }

function _blocks(inst)
    if not inst then return false end
    if _SKIP_NAME[inst.Name] then return false end
    if inst.CanCollide then return true end
    if _STRUCT[inst.Name] then return true end
    local s = inst.Size
    if s and math.max(s.X * s.Y, s.X * s.Z, s.Y * s.Z) > 150 then return true end
    return false
end

function _block(origin, target)
    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Exclude
    rp.IgnoreWater = true
    local skip = {}
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl.Character then skip[#skip + 1] = pl.Character end
    end
    local o = origin
    for _ = 1, 16 do
        rp.FilterDescendantsInstances = skip
        local d = target - o
        if d.Magnitude < 0.05 then return nil end
        local res = Workspace:Raycast(o, d, rp)
        if not res then return nil end
        if _blocks(res.Instance) then return res end
        skip[#skip + 1] = res.Instance
        o = res.Position + d.Unit * 0.3
    end
    return nil
end

function _clear(a, b) return _block(a, b) == nil end

function _clearDist(origin, dir, maxD)
    local res = _block(origin, origin + dir.Unit * maxD)
    if not res then return maxD end
    return (res.Position - origin).Magnitude
end

local function _len(pts)
    local s, prev = 0, pts[1]
    for k = 2, #pts do s = s + (pts[k] - prev).Magnitude; prev = pts[k] end
    return s
end

function _pull(pts)
    if #pts <= 2 then return pts end
    local out = { pts[1] }
    local i = 1
    while i < #pts do
        local j = #pts
        while j > i + 1 and not _clear(out[#out], pts[j]) do j = j - 1 end
        out[#out + 1] = pts[j]
        i = j
    end
    return out
end

local function _stages(toPos)
    local st = {}
    for _, dr in ipairs(_DIRS) do
        local cd = _clearDist(toPos, dr, 46)
        if cd >= 12 then st[#st + 1] = toPos + dr * math.min(cd - 5, 38) end
    end
    return st
end

local function _routeClear(pts)
    for i = 1, #pts - 1 do
        if not _clear(pts[i], pts[i + 1]) then return false end
    end
    return true
end

local function _peakY(pts)
    local m = -math.huge
    for _, p in ipairs(pts) do if p.Y > m then m = p.Y end end
    return m
end

local function _starts(fromPos)
    local pts = { fromPos }
    if _block(fromPos, fromPos + Vector3.new(0, 40, 0)) then
        for _, dr in ipairs(_DIRS) do
            local cd = _clearDist(fromPos, dr, 40)
            if cd >= 12 then pts[#pts + 1] = fromPos + dr * math.min(cd - 5, 34) end
        end
    end
    return pts
end

local function _candidates(sp, stage, toPos)
    local list = {}
    local function add(mid)
        if mid then list[#list + 1] = { sp, mid, stage, toPos }
        else list[#list + 1] = { sp, stage, toPos } end
    end
    add(nil)
    add(Vector3.new(stage.X, sp.Y, stage.Z))
    add(Vector3.new(sp.X, stage.Y, sp.Z))
    local dir = Vector3.new(stage.X - sp.X, 0, stage.Z - sp.Z)
    if dir.Magnitude > 0.1 then
        dir = dir.Unit
        local perp = Vector3.new(-dir.Z, 0, dir.X)
        for _, off in ipairs({ 20, -20, 40, -40 }) do
            add(sp + perp * off)
        end
    end
    return list
end

local PathfindingService = game:GetService("PathfindingService")
local _CLEARANCE = 6

local function _clearWide(a, b)
    if not _clear(a, b) then return false end
    local d = Vector3.new(b.X - a.X, 0, b.Z - a.Z)
    if d.Magnitude < 0.1 then return true end
    local p = Vector3.new(-d.Z, 0, d.X).Unit * _CLEARANCE
    return _clear(a + p, b + p) and _clear(a - p, b - p)
end

local function _pullWide(pts)
    if #pts <= 2 then return pts end
    local out = { pts[1] }
    local i = 1
    while i < #pts do
        local j = #pts
        while j > i + 1 and not _clearWide(out[#out], pts[j]) do j = j - 1 end
        out[#out + 1] = pts[j]
        i = j
    end
    return out
end

function computeRoute(fromPos, toPos, facingDir)
    if _clear(fromPos, toPos) then return { toPos } end
    local entry = facingDir and (toPos - facingDir * 14) or toPos
    local groundTo = Vector3.new(entry.X, fromPos.Y, entry.Z)
    local path = PathfindingService:CreatePath({
        AgentRadius = 8, AgentHeight = 5, AgentCanJump = true, AgentJumpHeight = 10, AgentMaxSlope = 89,
    })
    local FLOAT = 3
    local nav = { fromPos }
    local ok = pcall(function()
        path:ComputeAsync(Vector3.new(fromPos.X, fromPos.Y, fromPos.Z), groundTo)
    end)
    if ok and path.Status == Enum.PathStatus.Success then
        local last = fromPos
        for _, wp in ipairs(path:GetWaypoints()) do
            if (wp.Position - last).Magnitude >= 8 then
                nav[#nav + 1] = wp.Position + Vector3.new(0, FLOAT, 0)
                last = wp.Position
            end
        end
    end
    nav[#nav + 1] = entry + Vector3.new(0, FLOAT, 0)
    local route = _pullWide(nav)
    route[#route + 1] = toPos
    return route
end

-- =====================================================================
-- Tool & Clone Helpers
-- =====================================================================

function getPetPosition(plot, slot)
    local podiums = plot:FindFirstChild("AnimalPodiums")
    if not podiums then return nil end
    local podium = podiums:FindFirstChild(tostring(slot))
    if not podium then return nil end

    for _, desc in ipairs(podium:GetDescendants()) do
        if desc:IsA("Model") and desc.Name ~= "Claim" and desc.Name ~= "Base" and desc.Name ~= "Decorations" then
            local hasMesh = false
            for _, c in ipairs(desc:GetDescendants()) do
                if c:IsA("MeshPart") then hasMesh = true; break end
            end
            if hasMesh then
                local ok, cf = pcall(function() return desc:GetBoundingBox() end)
                if ok then return cf.Position end
            end
        end
    end

    local ok, cf = pcall(function() return podium:GetPivot() end)
    if ok then return cf.Position end
    return podium.Position
end

end

local cloneref = cloneref or function(o) return o end
local getinfo = debug.getinfo or getinfo
local lp = cloneref(game:GetService("Players")).LocalPlayer

local function Strip()
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    if not getconnections or not getinfo then return end
    for _, sig in ipairs({"CFrame", "Position"}) do
        local signal = hrp:GetPropertyChangedSignal(sig)
        if signal then
            for _, c in ipairs(getconnections(signal)) do
                local f = c.Function
                if f and c.Enabled and getinfo(f) and getinfo(f).source == "=ReplicatedFirst.test" then
                    pcall(function() c:Disable() end)
                end
            end
        end
    end
end

task.spawn(function()
    while true do
        pcall(Strip)
        task.wait(0.5)
    end
end)

function runAutoSnipe()
    pcall(Strip)
    if _G._isTpMoving then
        return
    end
    if Config.TpSettings and Config.TpSettings.GrabbleTP then
        _G._isTpMoving = true
        local fn = _G.SXEStartSideTP or doGrabbleVelocityTP
        local okGrab, errGrab = pcall(fn)
        _G._isTpMoving = false
        if not okGrab then
            warn("Grabble TP error:", errGrab)
        end
        return
    end
    if CarpetState and CarpetState.enabled then
        setCarpetSpeed(false)
    end
    local targetPetData = getTargetPetData()
    if not targetPetData then
        return
    end
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChild("Humanoid")
    if not hrp or not hum or (hum.Health <= 0) then
        return
    end
    local lastFpsCap = pcall(getfpscap) and getfpscap() or 60
    pcall(setfpscap, 200)
    -- (Old Grabble TP branch removed - now handled by SXE Clone-TP router above.)
    _G._isTpMoving = true
    local ok, err = pcall(function()
        local targetPart = findAdorneeGlobal(targetPetData)
        if not targetPart then
            error("targetPart nil")
        end
        local targetCF
        if targetPart:IsA("Attachment") then
            targetCF = targetPart.WorldCFrame
        elseif targetPart:IsA("BasePart") then
            targetCF = targetPart.CFrame
        else
            local okPivot, pivot = pcall(function()
                return targetPart:GetPivot()
            end)
            if okPivot then
                targetCF = pivot
            end
        end
        if not targetCF then
            error("targetCF nil")
        end
        local exactPos = targetPart.Position
        local carpetName = Config.TpSettings.Tool
        local carpet = LocalPlayer.Backpack:FindFirstChild(carpetName) or char:FindFirstChild(carpetName)
        local cloner = LocalPlayer.Backpack:FindFirstChild("Quantum Cloner") or char:FindFirstChild("Quantum Cloner")
        equipTpToolAndWait(hum)
        local isSecondFloor = exactPos.Y > 10
        local isCloseBase = false
        
        local signPart = getClosestBaseSign(targetPart)
            if not signPart then
                error("signPart nil")
            end
            local signCF = signPart.CFrame
            local RIGHT = signCF.RightVector
            local LEFT = -RIGHT
            local FORWARD = signCF.LookVector
            local BACK = -signCF.LookVector
            local medPoint = getNearestTeleportV2MedPoint(hrp.Position)
            if not medPoint then
                error("normal tp MED point nil")
            end
            hrp.AssemblyLinearVelocity = Vector3.zero
            local tpPos
            if exactPos.Y <= 6.313370704650879 then
                local frontPoint = signPart.Position + (FORWARD * 20)
                local backPoint = signPart.Position + (BACK * 20)
                local myPos = hrp.Position
                local distFront = (Vector3.new(frontPoint.X, 0, frontPoint.Z) - Vector3.new(myPos.X, 0, myPos.Z)).Magnitude
                local distBack = (Vector3.new(backPoint.X, 0, backPoint.Z) - Vector3.new(myPos.X, 0, myPos.Z)).Magnitude
                local chosen = ((distFront < distBack) and frontPoint) or backPoint
                tpPos = Vector3.new(chosen.X, -4.8, chosen.Z)
            else
                local backPoint = signPart.Position + (BACK * 20)
                local frontPoint = signPart.Position + (FORWARD * 20)
                local myPos = hrp.Position
                local myFlat = Vector3.new(myPos.X, 0, myPos.Z)
                local distBack = (Vector3.new(backPoint.X, 0, backPoint.Z) - myFlat).Magnitude
                local distFront = (Vector3.new(frontPoint.X, 0, frontPoint.Z) - myFlat).Magnitude
                local chosen = ((distBack < distFront) and backPoint) or frontPoint
                -- Always ground level Y
                tpPos = Vector3.new(chosen.X, -4.8, chosen.Z)
            end
            local initialDist = (hrp.Position - tpPos).Magnitude
            isCloseBase = (initialDist <= 100) and isSecondFloor
            if Config.TpSettings.FlyTP then
                -- Fly directly to the target base position (tpPos) facing LEFT, with obstacle avoidance
                flyForwardTo(hrp, tpPos, LEFT, isSecondFloor and (signPart.Position.Y + 4) or nil, isCloseBase and (Config.TpSettings.FlyTPCloseSpeed or 75) or nil)
            else
                -- Teleport directly to the target base position (tpPos) facing LEFT
                if hum and hum.Parent then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
                task.wait(0.01)
                hrp.AssemblyLinearVelocity = Vector3.zero
                local targetCF = CFrame.lookAt(tpPos, tpPos + LEFT)
                if isCloseBase then
                    -- Smooth CFrame glide to close bases to bypass anti-cheat flags
                    local startCF = hrp.CFrame
                    local duration = 2.5
                    local start = tick()
                    while tick() - start < duration and hrp.Parent do
                        local t = (tick() - start) / duration
                        hrp.AssemblyLinearVelocity = Vector3.zero
                        hrp.AssemblyAngularVelocity = Vector3.zero
                        hrp.CFrame = startCF:Lerp(targetCF, t)
                        RunService.Heartbeat:Wait()
                    end
                end
                hrp.CFrame = targetCF
                hrp.AssemblyLinearVelocity = Vector3.zero
            end
            waitUntilHeartbeat(function()
                return hrp and hrp.Parent and (flatDistance(hrp.Position, tpPos) <= 5)
            end, 0.5)
            task.wait(0.05)

            if isSecondFloor and hrp.Position.Y < (signPart.Position.Y + 2) then
                -- Smooth and slow fly up at a safe distance (tpPos is 15-20 studs away)
                local startY = hrp.Position.Y
                local targetY = signPart.Position.Y + 4
                local riseTime = isCloseBase and 5.0 or 1.2
                local elapsed = 0
                while elapsed < riseTime and hrp.Parent do
                    elapsed = elapsed + RunService.Heartbeat:Wait()
                    local t = math.min(elapsed / riseTime, 1)
                    local currentY = startY + (targetY - startY) * t
                    hrp.CFrame = CFrame.lookAt(Vector3.new(tpPos.X, currentY, tpPos.Z), Vector3.new(tpPos.X, currentY, tpPos.Z) + LEFT)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                end
                hrp.CFrame = CFrame.lookAt(Vector3.new(tpPos.X, targetY, tpPos.Z), Vector3.new(tpPos.X, targetY, tpPos.Z) + LEFT)
                hrp.AssemblyLinearVelocity = Vector3.zero
            end
            if isSecondFloor or not _G._isTargetPlotUnlocked(targetPetData.plot) then
                prepMiniTpTool(hum, hrp)
                waitSecondsHeartbeat(0.05)
                if isSecondFloor then
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    waitSecondsHeartbeat(0.01)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.CFrame = hrp.CFrame * CFrame.new(0, 0, -2.5)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                else
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    waitSecondsHeartbeat(0.03)
                    walkForward(0.1)
                end
                waitSecondsHeartbeat(Config.TpSettings.CloneDelayVal or 0.1)
                local miniPos = hrp.Position
                waitSecondsHeartbeat(0.01)
                local stillAtMiniPos = waitUntilHeartbeat(function()
                    return hrp and hrp.Parent and ((hrp.Position - miniPos).Magnitude <= 2)
                end, 0.75)
                if not stillAtMiniPos then
                    error("moved away from mini TP position before clone")
                end
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                local cloneSuccess = false
                for attempt = 1, 3 do
                    instantClone()
                    local moved = waitUntilHeartbeat(function()
                        return hrp and hrp.Parent and ((hrp.Position - miniPos).Magnitude >= 0.35)
                    end, 3)
                    if moved then
                        cloneSuccess = true
                        break
                    end
                    while _G.isCloning do
                        task.wait()
                    end
                    task.wait(0.1)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.CFrame = CFrame.new(miniPos)
                end
                if not cloneSuccess then
                    error("clone failed/no push")
                end
                if cloner then
                    hum:EquipTool(cloner)
                    task.wait(0.02)
                    pcall(function()
                        cloner:Activate()
                    end)
                    task.wait(0.05)
                    equipTpToolAndWait(hum)
                end
            end
            
            -- Keep velocity zero and CFrame glued to target animal for steal duration
            local holdEnd = tick() + math.max(0.18, Config.TpSettings.DelayVal or 0.4)
            while tick() < holdEnd do
                if not hrp.Parent then
                    break
                end
                if LocalPlayer:GetAttribute("Stealing") then
                    break
                end
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                pcall(function()
                    hrp.CFrame = CFrame.new(targetPart.Position)
                end)
                RunService.Heartbeat:Wait()
            end
            
            -- Post-steal stabilization / platform fallback
            if isSecondFloor then
                local plat = Instance.new("Part")
                plat.Name = "XiTempPlatform"
                plat.Size = Vector3.new(6, 1.5, 6)
                plat.Position = Vector3.new(hrp.Position.X, hrp.Position.Y - 5.5, hrp.Position.Z)
                plat.Color = Color3.fromRGB(232, 111, 177)
                plat.Material = Enum.Material.Neon
                plat.Anchored = true
                plat.CanCollide = false; pcall(makeOneWay, plat)
                plat.Transparency = 0.3
                plat.Parent = Workspace
                task.spawn(function()
                    local start = tick()
                    while (tick() - start) < 20 do
                        if LocalPlayer:GetAttribute("Stealing") then break end
                        task.wait(0.1)
                    end
                    plat:Destroy()
                end)
            else
                for i = 1, 5 do
                    if LocalPlayer:GetAttribute("Stealing") then break end
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    if (hrp.Position - targetPart.Position).Magnitude > 3 then
                        hrp.CFrame = CFrame.new(targetPart.Position)
                    end
                    task.wait(0.05)
                end
            end
        -- Force equip the configured TP tool (not the cloner)
        pcall(function() hum:UnequipTools() end)
        task.wait(0.05)
        equipTpToolAndWait(hum)
        -- Verify tool is equipped, retry if game re-equipped cloner
        task.defer(function()
            task.wait(0.1)
            local char2 = LocalPlayer.Character
            local hum2 = char2 and char2:FindFirstChildOfClass("Humanoid")
            if not hum2 then return end
            local toolName2 = Config.TpSettings.Tool or "Flying Carpet"
            local equipped = char2:FindFirstChild(toolName2)
            if not equipped then
                pcall(function() hum2:UnequipTools() end)
                task.wait(0.05)
                local t2 = LocalPlayer.Backpack:FindFirstChild(toolName2)
                if t2 then pcall(function() hum2:EquipTool(t2) end) end
            end
        end)
        if isCloseBase then
            task.wait(0.3) -- slightly delay starting carpet speed to brainrot for close bases to avoid anti-cheat flag
        end
        
        -- Walk to brainrot with Carpet Speed instead of direct teleport
        local targetBrainrotPos = targetPart.Position
        local distToBrainrot = (hrp.Position - targetBrainrotPos).Magnitude
        
        if distToBrainrot > 3 then
            -- Equip carpet and walk there with carpet speed
            local toolName = Config.TpSettings.Tool or "Flying Carpet"
            local tool = LocalPlayer.Backpack:FindFirstChild(toolName) or char:FindFirstChild(toolName)
            if tool and tool.Parent ~= char then hum:EquipTool(tool); task.wait(0.02) end
            
            local walkTimeout = os.clock()
            local WALK_SPEED = 190
            
            while hrp.Parent and (os.clock() - walkTimeout) < 8 do
                if LocalPlayer:GetAttribute("Stealing") then break end
                local brPos = targetPart.Position
                local diff = Vector3.new(brPos.X - hrp.Position.X, 0, brPos.Z - hrp.Position.Z)
                local flatDist = diff.Magnitude
                
                local verticalDiff = brPos.Y - hrp.Position.Y
                if flatDist < 3 and math.abs(verticalDiff) < 8 then break end
                
                local moveDir = (flatDist > 0.1) and diff.Unit or Vector3.zero
                local yVel = hrp.AssemblyLinearVelocity.Y
                
                -- Handle vertical: if brainrot is above, rise up smoothly
                if verticalDiff > 2 then
                    yVel = math.clamp(verticalDiff * 7, 30, 60)
                elseif verticalDiff < -2 then
                    yVel = math.clamp(verticalDiff * 4, -80, 0)
                end
                
                local currentWalkSpeed = WALK_SPEED
                if flatDist < 3 then currentWalkSpeed = 0 end
                
                hrp.AssemblyLinearVelocity = Vector3.new(moveDir.X * currentWalkSpeed, yVel, moveDir.Z * currentWalkSpeed)
                RunService.Heartbeat:Wait()
            end
            hrp.AssemblyLinearVelocity = Vector3.zero
        end
        
        -- Final positioning near brainrot
        local verticalDiff = targetPart.Position.Y - hrp.Position.Y
        if verticalDiff > 2 then
            local airPos = Vector3.new(targetPart.Position.X, targetPart.Position.Y - 8, targetPart.Position.Z)
            local plat = Instance.new("Part")
            plat.Name = "XiTempPlatform"
            plat.Size = Vector3.new(6, 1.5, 6)
            plat.Position = airPos - Vector3.new(0, 3, 0)
            plat.Color = Color3.fromRGB(232, 111, 177)
            plat.Material = Enum.Material.Neon
            plat.Anchored = true
            plat.CanCollide = false; pcall(makeOneWay, plat)
            plat.Transparency = 0.3
            plat.Parent = Workspace
            
            -- Smooth CFrame Lerp to prevent anti-cheat triggers and flinging
            local startCF = hrp.CFrame
            local targetCF = CFrame.new(airPos)
            local duration = 0.6
            local start = tick()
            while tick() - start < duration and hrp.Parent do
                if LocalPlayer:GetAttribute("Stealing") then break end
                local t = (tick() - start) / duration
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                hrp.CFrame = startCF:Lerp(targetCF, t)
                RunService.Heartbeat:Wait()
            end
            hrp.CFrame = targetCF
            hrp.AssemblyLinearVelocity = Vector3.zero
            task.spawn(function()
                local start = tick()
                while (tick() - start) < 20 do
                    if LocalPlayer:GetAttribute("Stealing") then break end
                    task.wait(0.1)
                end
                plat:Destroy()
            end)
        else
            for i = 1, 5 do
                if LocalPlayer:GetAttribute("Stealing") then break end
                hrp.AssemblyLinearVelocity = Vector3.zero
                if (hrp.Position - targetPart.Position).Magnitude > 3 then
                    hrp.CFrame = CFrame.new(targetPart.Position)
                end
                task.wait(0.05)
            end
        end
    end)
    _G._isTpMoving = false
    pcall(setfpscap, lastFpsCap)
    if not ok then
        warn("runAutoSnipe error:", err)
    end
end

function tpToBrainrot()
    local targetPetData = getTargetPetData()
    if not targetPetData then
        return
    end
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChild("Humanoid")
    if not hrp or not hum or (hum.Health <= 0) then
        return
    end
    local targetPart = nil
    for _ = 1, 10 do
        targetPart = findAdorneeGlobal(targetPetData)
        if targetPart then
            break
        end
        task.wait(0.05)
    end
    if not targetPart then
        return
    end
    local lastFpsCap = pcall(getfpscap) and getfpscap() or 60
    pcall(setfpscap, 200)
    local ok, err = pcall(function()
        local carpetName = Config.TpSettings.Tool
        local carpet = LocalPlayer.Backpack:FindFirstChild(carpetName) or char:FindFirstChild(carpetName)
        if carpet then
            hum:EquipTool(carpet)
        end
        local targetPos = targetPart.Position
        local verticalDiff = targetPos.Y - hrp.Position.Y
        if verticalDiff > 2 or targetPos.Y > 25 then
            local plat = Instance.new("Part")
            plat.Name = "XiTempPlatform"
            plat.Size = Vector3.new(6, 1.5, 6)
            plat.Position = Vector3.new(targetPos.X, targetPos.Y - 11, targetPos.Z)
            plat.Color = Color3.fromRGB(232, 111, 177)
            plat.Material = Enum.Material.Neon
            plat.Anchored = true
            plat.CanCollide = false; pcall(makeOneWay, plat)
            plat.Transparency = 0.3
            plat.Parent = Workspace
            
            -- Instant teleport instead of lerp
            local targetCF = CFrame.new(targetPos.X, targetPos.Y - 8, targetPos.Z)
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            hrp.CFrame = targetCF
            task.wait(0.05)
            hrp.AssemblyLinearVelocity = Vector3.zero
            
            task.spawn(function()
                local start = tick()
                while (tick() - start) < 20 do
                    if LocalPlayer:GetAttribute("Stealing") then
                        break
                    end
                    task.wait(0.1)
                end
                plat:Destroy()
            end)
        else
            -- Instant teleport to brainrot
            local targetCF = CFrame.new(targetPos)
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            hrp.CFrame = targetCF
            task.wait(0.05)
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            local holdEnd = tick() + math.max(0.18, Config.TpSettings.DelayVal or 0.4)
            while tick() < holdEnd do
                if not hrp.Parent then
                    break
                end
                if LocalPlayer:GetAttribute("Stealing") then
                    break
                end
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                pcall(function() hrp.CFrame = CFrame.new(targetPos) end)
                RunService.Heartbeat:Wait()
            end
        end
    end)
    pcall(setfpscap, lastFpsCap)
    if not ok then
        warn("tpToBrainrot error:", err)
    end
end
_G.runAutoSnipe = runAutoSnipe
_G.tpToBrainrot = tpToBrainrot

-- ============================================================
-- CLICK TO AP
-- ============================================================
local ctapHighlight=Instance.new("Highlight",CoreGui)
ctapHighlight.FillColor=Color3.fromRGB(255,150,200); ctapHighlight.FillTransparency=0.3
ctapHighlight.OutlineColor=Color3.fromRGB(255,150,200); ctapHighlight.OutlineTransparency=0
ctapHighlight.Adornee=nil; ctapHighlight.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop

local function rayToCubeIntersect(rayOrigin,rayDirection,cubeCenter,cubeSize)
    local halfSize=cubeSize/2; local minB=cubeCenter-Vector3.new(halfSize,halfSize,halfSize); local maxB=cubeCenter+Vector3.new(halfSize,halfSize,halfSize)
    local rd=Vector3.new(rayDirection.X==0 and 0.0001 or rayDirection.X, rayDirection.Y==0 and 0.0001 or rayDirection.Y, rayDirection.Z==0 and 0.0001 or rayDirection.Z)
    local tmin,tmax=(minB.X-rayOrigin.X)/rd.X,(maxB.X-rayOrigin.X)/rd.X; if tmin>tmax then tmin,tmax=tmax,tmin end
    local tymin,tymax=(minB.Y-rayOrigin.Y)/rd.Y,(maxB.Y-rayOrigin.Y)/rd.Y; if tymin>tymax then tymin,tymax=tymax,tymin end
    if tmin>tymax or tymin>tmax then return false end; if tymin>tmin then tmin=tymin end; if tymax<tmax then tmax=tymax end
    local tzmin,tzmax=(minB.Z-rayOrigin.Z)/rd.Z,(maxB.Z-rayOrigin.Z)/rd.Z; if tzmin>tzmax then tzmin,tzmax=tzmax,tzmin end
    return not(tmin>tzmax or tzmin>tmax)
end

RunService.RenderStepped:Connect(function()
    if Config.ClickToAP then
        local camera=Workspace.CurrentCamera; local mousePos=UIS:GetMouseLocation()
        local ray=camera:ViewportPointToRay(mousePos.X,mousePos.Y); local bestPlayer,bestDist=nil,math.huge
        for _,p in ipairs(Players:GetPlayers()) do if p~=LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            if rayToCubeIntersect(ray.Origin,ray.Direction,p.Character.HumanoidRootPart.Position,Config.ClickToAPRadius or 8) then
                local dist=(ray.Origin-p.Character.HumanoidRootPart.Position).Magnitude; if dist<bestDist then bestDist=dist; bestPlayer=p end
            end
        end end
        ctapHighlight.Adornee=(bestPlayer and bestPlayer.Character) or nil
    else ctapHighlight.Adornee=nil end
end)

UIS.InputBegan:Connect(function(inp,g)
    if not g and inp.UserInputType==Enum.UserInputType.MouseButton1 and Config.ClickToAP then
        local camera=Workspace.CurrentCamera; local mousePos=UIS:GetMouseLocation()
        local ray=camera:ViewportPointToRay(mousePos.X,mousePos.Y); local bestPlayer,bestDist=nil,math.huge
        for _,p in ipairs(Players:GetPlayers()) do if p~=LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            if rayToCubeIntersect(ray.Origin,ray.Direction,p.Character.HumanoidRootPart.Position,Config.ClickToAPRadius or 8) then
                local dist=(ray.Origin-p.Character.HumanoidRootPart.Position).Magnitude; if dist<bestDist then bestDist=dist; bestPlayer=p end
            end
        end end
        if bestPlayer then
            if isPlayerBlacklisted(bestPlayer) then
                ShowNotification("CLICK AP", bestPlayer.DisplayName .. " is blacklisted")
                return
            end
            local cmdList = Config.ClickToAPOrder or AP_ALL_COMMANDS
            local startIndex = _G.ClickToAPIndex or 1
            if startIndex > #cmdList then startIndex = 1 end
            
            local picked = nil
            local attempts = 0
            while attempts < #cmdList do
                local idx = ((startIndex - 1 + attempts) % #cmdList) + 1
                local cmd = cmdList[idx]
                if Config.ClickToAPCommands and Config.ClickToAPCommands[cmd] and not apIsOnCooldown(cmd) then
                    picked = cmd
                    _G.ClickToAPIndex = idx + 1
                    break
                end
                attempts = attempts + 1
            end

            if not picked then
                ShowNotification("CLICK AP", "No commands enabled or all on cooldown")
                return
            end

            if runAdminCommand(bestPlayer, picked) then
                local emoji = AP_COMMAND_EMOJIS[picked] or "⚡"
                ShowNotification("CLICK AP", emoji .. " " .. picked .. " → " .. bestPlayer.DisplayName)
            end
        end
    end
end)

-- ============================================================
-- ESP
-- ============================================================
-- Player ESP
local playerESPEnabled=Config.PlayerESP; local playerBillboards={}
DANGER_TOOLS={["Boogie Bomb"]=true,["Medusa's Head"]=true,["Body Swap Potion"]=true,["Laser Cape"]=true,["Rainbowrath Sword"]=true,["Gummy Bear"]=true}
local function getHeldTool(p) local c=p.Character; if not c then return nil end; for _,o in ipairs(c:GetChildren()) do if o:IsA("Tool") then return o.Name end end; return nil end
local function makePlayerBillboard(plr)
    local bb=Instance.new("BillboardGui"); bb.Name="PlayerESP_"..tostring(plr.UserId); bb.Size=UDim2.new(0,170,0,34)
    bb.StudsOffsetWorldSpace=Vector3.new(0,2.8,0); bb.AlwaysOnTop=true; bb.LightInfluence=0; bb.ResetOnSpawn=false
    local nameLbl=Instance.new("TextLabel",bb); nameLbl.Size=UDim2.new(1,0,0,18); nameLbl.BackgroundTransparency=1
    nameLbl.Font=Enum.Font.GothamBold; nameLbl.TextSize=14; nameLbl.TextColor3=Color3.fromRGB(255,255,255)
    nameLbl.TextStrokeTransparency=0.4; nameLbl.TextStrokeColor3=Color3.fromRGB(0,0,0); nameLbl.Text=plr.Name
    local toolLbl=Instance.new("TextLabel",bb); toolLbl.Name="ToolLabel"; toolLbl.Size=UDim2.new(1,0,0,13); toolLbl.Position=UDim2.new(0,0,0,18)
    toolLbl.BackgroundTransparency=1; toolLbl.Font=Enum.Font.GothamMedium; toolLbl.TextSize=11; toolLbl.TextColor3=Color3.fromRGB(100,220,255)
    toolLbl.TextStrokeTransparency=0.4; toolLbl.TextStrokeColor3=Color3.fromRGB(0,0,0); toolLbl.Text=getHeldTool(plr) or ""
    return bb,nameLbl
end
local function createOrRefreshPlayerESP(plr)
    if plr==LocalPlayer then return end; local hrp=plr.Character and plr.Character:FindFirstChild("HumanoidRootPart"); if not hrp then return end
    local hum=plr.Character:FindFirstChild("Humanoid"); if hum then hum.DisplayDistanceType=Enum.HumanoidDisplayDistanceType.None end
    local uid=plr.UserId; local entry=playerBillboards[uid]
    if not entry or not entry.bb or not entry.bb.Parent then
        if entry and entry.bb then pcall(function() entry.bb:Destroy() end) end
        local bb,nameLbl=makePlayerBillboard(plr); bb.Adornee=hrp; bb.Parent=hrp; playerBillboards[uid]={bb=bb,nameLbl=nameLbl,player=plr}
    elseif entry.bb.Adornee~=hrp then entry.bb.Adornee=hrp; entry.bb.Parent=hrp end
end
local function clearPlayerESP()
    for uid,entry in pairs(playerBillboards) do if entry.bb then pcall(entry.bb.Destroy,entry.bb) end; playerBillboards[uid]=nil end
end
task.spawn(function() while true do task.wait(0.5)
    if playerESPEnabled then
        for _,plr in ipairs(Players:GetPlayers()) do if plr~=LocalPlayer then pcall(createOrRefreshPlayerESP,plr) end end
        for uid,entry in pairs(playerBillboards) do if entry.bb and entry.bb.Parent then
            pcall(function() local tl=entry.bb:FindFirstChild("ToolLabel"); if tl then local ht=getHeldTool(entry.player); tl.Text=ht or ""
                if entry.nameLbl then entry.nameLbl.TextColor3=ht and DANGER_TOOLS[ht] and Color3.fromRGB(255,60,60) or Color3.fromRGB(255,255,255) end
            end end)
        end end
    else clearPlayerESP() end
end end)

-- Line To Base ESP (kinqs beam style)
do
    local plotBeam = nil
    local plotBeamAttachment0 = nil
    local plotBeamAttachment1 = nil

    local function findMyPlot()
        local plots = workspace:FindFirstChild("Plots")
        if not plots then return nil end
        for _, plot in ipairs(plots:GetChildren()) do
            local sign = plot:FindFirstChild("PlotSign")
            if sign then
                local surfaceGui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
                if surfaceGui then
                    local label = surfaceGui:FindFirstChildWhichIsA("TextLabel", true)
                    if label then
                        local text = label.Text:lower()
                        if text:find(LocalPlayer.DisplayName:lower(), 1, true) or text:find(LocalPlayer.Name:lower(), 1, true) then
                            return plot
                        end
                    end
                end
            end
        end
        return nil
    end

    local function createPlotBeam()
        if not Config.LineToBase then return end
        local myPlot = findMyPlot()
        if not myPlot or not myPlot.Parent then return end
        local character = LocalPlayer.Character
        if not character or not character.Parent then return end
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp or not hrp.Parent then return end
        if plotBeam then pcall(function() plotBeam:Destroy() end) end
        if plotBeamAttachment0 then pcall(function() plotBeamAttachment0:Destroy() end) end
        plotBeamAttachment0 = hrp:FindFirstChild("PlotBeamAttach_Player") or Instance.new("Attachment")
        plotBeamAttachment0.Name = "PlotBeamAttach_Player"
        plotBeamAttachment0.Position = Vector3.new(0, 0, 0)
        plotBeamAttachment0.Parent = hrp
        local plotPart = myPlot:FindFirstChild("MainRootPart") or myPlot:FindFirstChildWhichIsA("BasePart")
        if not plotPart or not plotPart.Parent then return end
        plotBeamAttachment1 = plotPart:FindFirstChild("PlotBeamAttach_Plot") or Instance.new("Attachment")
        plotBeamAttachment1.Name = "PlotBeamAttach_Plot"
        plotBeamAttachment1.Position = Vector3.new(0, 5, 0)
        plotBeamAttachment1.Parent = plotPart
        plotBeam = hrp:FindFirstChild("PlotBeam") or Instance.new("Beam")
        plotBeam.Name = "PlotBeam"
        plotBeam.Attachment0 = plotBeamAttachment0
        plotBeam.Attachment1 = plotBeamAttachment1
        plotBeam.FaceCamera = true
        plotBeam.LightEmission = 1
        plotBeam.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255))
        plotBeam.Transparency = NumberSequence.new(0)
        plotBeam.Width0 = 0.3
        plotBeam.Width1 = 0.3
        plotBeam.TextureMode = Enum.TextureMode.Wrap
        plotBeam.TextureSpeed = 0
        plotBeam.Parent = hrp
    end

    local function resetPlotBeam()
        if plotBeam then pcall(function() plotBeam:Destroy() end) end
        if plotBeamAttachment0 then pcall(function() plotBeamAttachment0:Destroy() end) end
        if plotBeamAttachment1 then pcall(function() plotBeamAttachment1:Destroy() end) end
        plotBeam = nil
        plotBeamAttachment0 = nil
        plotBeamAttachment1 = nil
    end

    task.spawn(function()
        local checkCounter = 0
        RunService.Heartbeat:Connect(function()
            if not Config.LineToBase then return end
            checkCounter = checkCounter + 1
            if checkCounter >= 30 then
                checkCounter = 0
                if not plotBeam or not plotBeam.Parent or not plotBeamAttachment0 or not plotBeamAttachment0.Parent then
                    pcall(createPlotBeam)
                end
            end
        end)
    end)

    LocalPlayer.CharacterAdded:Connect(function(character)
        task.wait(0.5)
        if Config.LineToBase and character then
            pcall(createPlotBeam)
        end
    end)

    if LocalPlayer.Character then
        task.spawn(function()
            task.wait(0.2)
            if Config.LineToBase then createPlotBeam() end
        end)
    end

    _G.createPlotBeam = createPlotBeam
    _G.resetPlotBeam = resetPlotBeam
end

-- Line To Best Brainrot (kinqs beam style)
do
    local BEAM_NAME    = "BestPetBeam"
    local ATT0_NAME    = "BestPetBeamAttach_Player"
    local ATT1_NAME    = "BestPetBeamAttach_Target"
    local BEAM_COLOR   = Color3.fromRGB(255, 0, 0)   -- red (line to best brainrot)

    local bestBeam     = nil
    local bestAtt0     = nil
    local bestAtt1     = nil
    local currentTargetPart = nil

    local function destroyBeam()
        if bestBeam then pcall(function() bestBeam:Destroy() end) end
        if bestAtt0 then pcall(function() bestAtt0:Destroy() end) end
        if bestAtt1 then pcall(function() bestAtt1:Destroy() end) end
        bestBeam, bestAtt0, bestAtt1 = nil, nil, nil
        currentTargetPart = nil
    end

    local function getBestTargetPart()
        local cache = SharedState.AllAnimalsCache
        if not cache or #cache == 0 then return nil end
        local bestData = nil
        local bestVal = 0
        -- Priority list first
        for _, pName in ipairs(priorityList) do
            local searchName = pName:lower()
            for _, a in ipairs(cache) do
                if a and a.name and a.name:lower() == searchName and a.owner ~= LocalPlayer.Name then
                    local score = (a.genValue or 0) + 1e15
                    if score > bestVal then
                        bestVal = score
                        bestData = a
                    end
                end
            end
            if bestData then break end
        end
        -- Then brainrots by value
        if not bestData then
            for _, a in ipairs(cache) do
                if a and a.owner ~= LocalPlayer.Name and a.genValue and a.genValue >= 10000000 then
                    if a.genValue > bestVal then
                        bestVal = a.genValue
                        bestData = a
                    end
                end
            end
        end
        -- Then highest gen fallback
        if not bestData then
            for _, a in ipairs(cache) do
                if a and a.owner ~= LocalPlayer.Name and (a.genValue or 0) >= 1 then
                    if (a.genValue or 0) > bestVal then
                        bestVal = a.genValue or 0
                        bestData = a
                    end
                end
            end
        end
        if not bestData then return nil end
        local adornee = findAdorneeGlobal(bestData)
        if adornee and adornee:IsA("BasePart") then return adornee end
        return nil
    end

    local function ensureBeam(hrp, targetPart)
        if not hrp or not hrp.Parent or not targetPart or not targetPart.Parent then return end
        if not bestAtt0 or not bestAtt0.Parent or bestAtt0.Parent ~= hrp then
            if bestAtt0 then pcall(function() bestAtt0:Destroy() end) end
            bestAtt0 = hrp:FindFirstChild(ATT0_NAME) or Instance.new("Attachment")
            bestAtt0.Name = ATT0_NAME
            bestAtt0.Position = Vector3.new(0, 0, 0)
            bestAtt0.Parent = hrp
        end
        if currentTargetPart ~= targetPart or not bestAtt1 or not bestAtt1.Parent then
            if bestAtt1 then pcall(function() bestAtt1:Destroy() end) end
            bestAtt1 = targetPart:FindFirstChild(ATT1_NAME) or Instance.new("Attachment")
            bestAtt1.Name = ATT1_NAME
            bestAtt1.Position = Vector3.new(0, 3, 0)
            bestAtt1.Parent = targetPart
            currentTargetPart = targetPart
            if bestBeam then
                bestBeam.Attachment1 = bestAtt1
            end
        end
        if not bestBeam or not bestBeam.Parent then
            if bestBeam then pcall(function() bestBeam:Destroy() end) end
            bestBeam = Instance.new("Beam")
            bestBeam.Name = BEAM_NAME
            bestBeam.Attachment0 = bestAtt0
            bestBeam.Attachment1 = bestAtt1
            bestBeam.FaceCamera = true
            bestBeam.LightEmission = 1
            bestBeam.Color = ColorSequence.new(BEAM_COLOR)
            bestBeam.Transparency = NumberSequence.new(0.35)
            bestBeam.Width0 = 0.45
            bestBeam.Width1 = 0.45
            bestBeam.TextureMode = Enum.TextureMode.Wrap
            bestBeam.TextureSpeed = 0
            bestBeam.Parent = hrp
        end
    end

    task.spawn(function()
        local check = 0
        RunService.Heartbeat:Connect(function()
            if not Config.LineToBrainrot then
                if bestBeam or bestAtt0 or bestAtt1 then destroyBeam() end
                return
            end
            check = check + 1
            if check < 10 then return end
            check = 0

            local char = LocalPlayer.Character
            local hrp  = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then
                if bestBeam or bestAtt0 or bestAtt1 then destroyBeam() end
                return
            end

            local targetPart = getBestTargetPart()
            if not targetPart or not targetPart.Parent then
                if bestBeam or bestAtt1 then
                    if bestBeam then pcall(function() bestBeam:Destroy() end); bestBeam = nil end
                    if bestAtt1 then pcall(function() bestAtt1:Destroy() end); bestAtt1 = nil end
                    currentTargetPart = nil
                end
                return
            end

            pcall(ensureBeam, hrp, targetPart)

            -- Color coding: green when stealing, yellow when close, red default
            if bestBeam then
                local col = BEAM_COLOR
                if LocalPlayer:GetAttribute("Stealing") then
                    col = Color3.fromRGB(80, 255, 120)
                else
                    local dist = hrp and targetPart and (hrp.Position - targetPart.Position).Magnitude or math.huge
                    if dist < 30 then
                        col = Color3.fromRGB(255, 196, 72)
                    end
                end
                bestBeam.Color = ColorSequence.new(col)
            end
        end)
    end)

    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.5)
        currentTargetPart = nil
        bestAtt0 = nil
        if bestBeam then pcall(function() bestBeam:Destroy() end); bestBeam = nil end
    end)

    _G.updateBrainrotBeam = function()
        -- beam auto-updates via heartbeat, nothing to do
    end
    _G.resetBrainrotBeam = destroyBeam
    _G.resetBestPetBeam = destroyBeam
end

-- Brainrot ESP
local brainrotESPEnabled=Config.BrainrotESP; local brainrotBillboards={}
local function createBrainrotBillboard(data)
    local bb=Instance.new("BillboardGui"); bb.Name="BrainrotESP_"..data.uid; bb.Size=UDim2.new(0,160,0,38)
    bb.StudsOffset=Vector3.new(0,1.8,0); bb.AlwaysOnTop=true; bb.LightInfluence=0; bb.MaxDistance=3000
    local container=Instance.new("Frame",bb); container.Size=UDim2.new(1,0,1,0); container.BackgroundColor3=Color3.fromRGB(0,0,0)
    container.BackgroundTransparency=0.5; container.BorderSizePixel=0; Instance.new("UICorner",container).CornerRadius=UDim.new(0,4)
    local stroke=Instance.new("UIStroke",container); stroke.Color=Color3.fromRGB(175,175,175); stroke.Thickness=1.5; stroke.Transparency=0.2
    local nameLabel=Instance.new("TextLabel",container); nameLabel.Size=UDim2.new(1,-6,0,18); nameLabel.Position=UDim2.new(0,3,0,2)
    nameLabel.BackgroundTransparency=1; nameLabel.Font=Enum.Font.GothamBlack; nameLabel.TextSize=13; nameLabel.TextColor3=Color3.fromRGB(175,175,175)
    nameLabel.TextStrokeTransparency=0; nameLabel.TextStrokeColor3=Color3.fromRGB(0,0,0); nameLabel.Text=data.name or "???"; nameLabel.TextXAlignment=Enum.TextXAlignment.Center
    local genLabel=Instance.new("TextLabel",container); genLabel.Size=UDim2.new(1,-6,0,14); genLabel.Position=UDim2.new(0,3,0,20)
    genLabel.BackgroundTransparency=1; genLabel.Font=Enum.Font.GothamBold; genLabel.TextSize=11; genLabel.TextColor3=Color3.fromRGB(255,255,255)
    genLabel.TextStrokeTransparency=0; genLabel.TextStrokeColor3=Color3.fromRGB(0,0,0); genLabel.Text=data.genText or ""; genLabel.TextXAlignment=Enum.TextXAlignment.Center
    return bb
end
local function refreshBrainrotESP()
    if not brainrotESPEnabled then return end; local cache=SharedState.AllAnimalsCache; if not cache or #cache==0 then return end
    local seen={}
    for _,data in ipairs(cache) do if data.genValue>=10000000 then seen[data.uid]=true
        if not brainrotBillboards[data.uid] then
            local adornee=findAdorneeGlobal(data)
            if adornee then local bb=createBrainrotBillboard(data); bb.Adornee=adornee; bb.Parent=adornee
                local hlParent=(adornee.Parent and adornee.Parent:IsA("Model") and adornee.Parent) or adornee
                local hl=Instance.new("Highlight"); hl.Name="BrainrotHL_"..data.uid; hl.FillColor=Color3.fromRGB(175,175,175); hl.FillTransparency=0.65
                hl.OutlineColor=Color3.fromRGB(175,175,175); hl.OutlineTransparency=0.1; hl.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop; hl.Adornee=hlParent; hl.Parent=hlParent
                brainrotBillboards[data.uid]={bb=bb,highlight=hl}
            end
        end
    end end
    for uid,entry in pairs(brainrotBillboards) do if not seen[uid] then
        if entry.bb then entry.bb:Destroy() end; if entry.highlight then entry.highlight:Destroy() end; brainrotBillboards[uid]=nil
    end end
end
local function clearBrainrotESP() for _,e in pairs(brainrotBillboards) do if e.bb then e.bb:Destroy() end; if e.highlight then e.highlight:Destroy() end end; brainrotBillboards={} end
task.spawn(function() while true do task.wait(0.3); if brainrotESPEnabled then pcall(refreshBrainrotESP) end end end)

-- Subspace Mine ESP
local subspaceMineESPEnabled=Config.SubspaceMineESP; local subspaceMineESPData={}
local function refreshSubspaceMineESP()
    if not subspaceMineESPEnabled then return end; local tools=Workspace:FindFirstChild("ToolsAdds"); if not tools then return end
    local currentMines={}
    for _,obj in ipairs(tools:GetChildren()) do if obj.Name:match("SubspaceTripmine") and obj:IsA("BasePart") then currentMines[obj]=true
        if not subspaceMineESPData[obj] then
            local ownerName=obj.Name:match("SubspaceTripmine(.+)") or "Unknown"
            local sel=Instance.new("SelectionBox",obj); sel.Color3=Color3.fromRGB(167,142,255); sel.LineThickness=0.05
            local bb=Instance.new("BillboardGui",obj); bb.Size=UDim2.new(0,250,0,50); bb.StudsOffset=Vector3.new(0,2.5,0); bb.AlwaysOnTop=false
            local lbl=Instance.new("TextLabel",bb); lbl.Size=UDim2.new(1,0,1,0); lbl.BackgroundTransparency=1; lbl.Text=ownerName.."'s Subspace Mine"
            lbl.TextColor3=Color3.fromRGB(167,142,255); lbl.TextStrokeTransparency=0; lbl.Font=Enum.Font.GothamBold; lbl.TextSize=16
            subspaceMineESPData[obj]={sel=sel,bb=bb}
        end
    end end
    for mineObj,data in pairs(subspaceMineESPData) do if not currentMines[mineObj] or not mineObj.Parent then
        data.sel:Destroy(); data.bb:Destroy(); subspaceMineESPData[mineObj]=nil
    end end
end
task.spawn(function() while true do if subspaceMineESPEnabled then pcall(refreshSubspaceMineESP) end; task.wait(0.5) end end)

-- Timer ESP
local timerESPEnabled=Config.TimerESP
local function clearTimerESP()
    local plots = Workspace:FindFirstChild("Plots")
    if plots then
        for _, plot in ipairs(plots:GetChildren()) do
            for _, desc in ipairs(plot:GetDescendants()) do
                if desc.Name == "TimerESP" and desc:IsA("BillboardGui") then
                    pcall(function() desc:Destroy() end)
                end
            end
        end
    end
end
_G.clearTimerESP = clearTimerESP

local function refreshTimerESP()
    if not timerESPEnabled then
        clearTimerESP()
        return
    end
    local plots=Workspace:FindFirstChild("Plots"); if not plots then return end
    for _,plot in ipairs(plots:GetChildren()) do
        -- The game shows a timer per floor. User wants ONLY the first floor, so
        -- gather this plot's timer billboards and keep just the lowest-Y ones.
        local found = {}
        local minY = math.huge
        for _,g in ipairs(plot:GetDescendants()) do
            if g:IsA("BillboardGui") and g:FindFirstChild("RemainingTime") then
                local base=g.Adornee or g.Parent
                if base and base:IsA("BasePart") then
                    found[#found+1] = {g=g, base=base, y=base.Position.Y}
                    if base.Position.Y < minY then minY = base.Position.Y end
                end
            end
        end
        for _,item in ipairs(found) do
            local base, g = item.base, item.g
            local existing = base:FindFirstChild("TimerESP")
            if item.y <= minY + 4 then   -- first floor only
                local rt = g:FindFirstChild("RemainingTime")
                if rt then
                    if not existing then
                        local bb=Instance.new("BillboardGui"); bb.Name="TimerESP"; bb.Adornee=base; bb.Size=UDim2.new(0,98,0,26); bb.AlwaysOnTop=true; bb.StudsOffsetWorldSpace=Vector3.new(0,1.6,0)
                        local lbl=Instance.new("TextLabel",bb); lbl.Size=UDim2.new(1,0,1,0); lbl.BackgroundTransparency=1; lbl.Text=rt.Text
                        lbl.Font=Enum.Font.GothamBold; lbl.TextSize=15; lbl.TextColor3=Color3.fromRGB(255,255,255)
                        lbl.TextStrokeTransparency=0.35; lbl.TextStrokeColor3=Color3.fromRGB(0,0,0); bb.Parent=base
                    else existing.TextLabel.Text=rt.Text end
                end
            elseif existing then
                pcall(function() existing:Destroy() end)   -- not first floor -> remove
            end
        end
    end
end
task.spawn(function() while true do if timerESPEnabled then pcall(refreshTimerESP) else clearTimerESP() end; task.wait(0.5) end end)

-- ============================================================
-- Base Owner ESP  (red outline + "BASE OWNER" tag on the owner of the
-- base you are CURRENTLY standing in -- one player at a time, not all bases)
-- Wrapped in a do-block + exposed via _G so it adds no chunk-level locals.
-- ============================================================
do
    local enabled = Config.BaseOwnerESP
    local entries = {}   -- userId -> { hl = Highlight, bb = BillboardGui }

    local function destroyEntry(e)
        if not e then return end
        if e.hl then pcall(function() e.hl:Destroy() end) end
        if e.bb then pcall(function() e.bb:Destroy() end) end
    end
    local function clearAll()
        for uid, e in pairs(entries) do destroyEntry(e); entries[uid] = nil end
    end

    local function makeTag()
        local bb = Instance.new("BillboardGui")
        bb.Name = "BaseOwnerTag"
        bb.Size = UDim2.new(0, 160, 0, 34)
        bb.StudsOffsetWorldSpace = Vector3.new(0, 3.6, 0)
        bb.AlwaysOnTop = true
        bb.LightInfluence = 0
        local lbl = Instance.new("TextLabel", bb)
        lbl.Size = UDim2.fromScale(1, 1)
        lbl.BackgroundTransparency = 1
        lbl.Font = Enum.Font.GothamBlack
        lbl.TextSize = 18
        lbl.TextColor3 = Color3.fromRGB(255, 60, 60)
        lbl.TextStrokeTransparency = 0
        lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        lbl.Text = "BASE OWNER"
        return bb
    end

    -- Highlight + tag ONLY the given player (owner of the base you're in).
    -- Passing nil clears everything.
    local function setTarget(plr)
        for uid, e in pairs(entries) do
            if not plr or uid ~= plr.UserId then destroyEntry(e); entries[uid] = nil end
        end
        if not plr or not plr.Character then return end
        local uid = plr.UserId
        local e = entries[uid]; if not e then e = {}; entries[uid] = e end
        if not e.hl or not e.hl.Parent then
            if e.hl then pcall(function() e.hl:Destroy() end) end
            local hl = Instance.new("Highlight")
            hl.Name = "BaseOwnerESP"
            hl.FillColor = Color3.fromRGB(255, 0, 0); hl.FillTransparency = 0.6
            hl.OutlineColor = Color3.fromRGB(255, 0, 0); hl.OutlineTransparency = 0
            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            hl.Parent = CoreGui
            e.hl = hl
        end
        if e.hl.Adornee ~= plr.Character then e.hl.Adornee = plr.Character end
        if not e.bb or not e.bb.Parent then
            if e.bb then pcall(function() e.bb:Destroy() end) end
            e.bb = makeTag(); e.bb.Parent = CoreGui
        end
        local head = plr.Character:FindFirstChild("Head") or plr.Character:FindFirstChild("HumanoidRootPart")
        if head and e.bb.Adornee ~= head then e.bb.Adornee = head end
    end

    local function refresh()
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then setTarget(nil); return end
        local plot = getPlotAtPosition(hrp.Position)
        local owner = plot and getPlotOwner(plot)
        if owner == LocalPlayer then owner = nil end   -- don't tag your own base
        setTarget(owner)
    end
    _G.setBaseOwnerESP = function(on)
        enabled = on
        Config.BaseOwnerESP = on
        saveConfig()
        setToggle("Base Owner ESP", on)
        if on then pcall(refresh) else clearAll() end
    end
    _G.clearBaseOwnerESP = clearAll
    task.spawn(function()
        while true do
            task.wait(0.5)
            if enabled then pcall(refresh) end
        end
    end)
end

-- ============================================================
-- FPS BOOST
-- ============================================================
local function IsProtected(obj)
    if not obj then return false end
    local name = obj.Name:lower()
    if name:find("laser") or name:find("door") or name:find("gate") or name:find("shield") or name:find("barrier") or name:find("fence") or name:find("forcefield") or name:find("wall") or name:find("protect") then
        return true
    end
    local parent = obj.Parent
    while parent and parent ~= Workspace do
        local pName = parent.Name:lower()
        if pName:find("laser") or pName:find("door") or pName:find("gate") or pName:find("shield") or pName:find("barrier") or pName:find("fence") or pName:find("forcefield") or pName:find("wall") or pName:find("protect") then
            return true
        end
        parent = parent.Parent
    end
    return false
end

local fpsBoostConnection = nil
local function setFPSBoost(enabled) Config.FPSBoost=enabled; saveConfig()
    setToggle("FPS Boost (normal)", enabled)
    if fpsBoostConnection then
        pcall(function() fpsBoostConnection:Disconnect() end)
        fpsBoostConnection = nil
    end
    if enabled then
        Lighting.GlobalShadows=false; Lighting.Brightness=2; Lighting.FogEnd=9e9; Lighting.FogStart=0
        Lighting.EnvironmentDiffuseScale=0; Lighting.EnvironmentSpecularScale=0
        for _,v in pairs(Lighting:GetChildren()) do
            if v:IsA("BloomEffect") or v:IsA("BlurEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("SunRaysEffect") or v:IsA("DepthOfFieldEffect") then pcall(function() v.Enabled=false end) elseif v:IsA("Atmosphere") then pcall(function() v:Destroy() end) end
        end
        for _,obj in ipairs(Workspace:GetDescendants()) do
            if not IsProtected(obj) then
                if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then pcall(function() obj.Enabled=false end) end
                if obj:IsA("BasePart") then pcall(function() obj.Material=Enum.Material.Plastic; obj.CastShadow=false end) end
                if obj:IsA("SurfaceAppearance") or obj:IsA("Texture") or obj:IsA("Decal") then pcall(function() obj:Destroy() end) end
            end
        end
        fpsBoostConnection = Workspace.DescendantAdded:Connect(function(obj)
            if not Config.FPSBoost then return end
            if not IsProtected(obj) then
                if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Smoke") then pcall(function() obj.Enabled=false end) end
                if obj:IsA("BasePart") then pcall(function() obj.Material=Enum.Material.Plastic; obj.CastShadow=false end) end
            end
        end)
    end
end

-- FPS BOOST ULTRA
local OriginalTransparency = setmetatable({}, {__mode = "k"})
local _ultraDescendantConn = nil
local _ultraLightingConn = nil
local _ultraMaterialConn = nil
local _ultraThreads = {}
local _ultraConnections = {}

local function AddUltraThread(f)
	table.insert(_ultraThreads, task.spawn(f))
end

local function AddUltraConnection(c)
	table.insert(_ultraConnections, c)
end

local function SafeDestroyUltra(obj)
	if obj.Name == "Overhead" then return end
	pcall(function() obj:Destroy() end)
end

local ClothingClasses = {
	"Shirt","Pants","ShirtGraphic",
	"Accessory","Hat","HairAccessory",
	"FaceAccessory","NeckAccessory","ShoulderAccessory",
	"FrontAccessory","BackAccessory","WaistAccessory",
}

local function IsClothing(obj)
	for _, c in ipairs(ClothingClasses) do
		if obj:IsA(c) then return true end
	end
end

local function IsCharacterPart(obj)
	local parent = obj.Parent
	while parent and parent ~= Workspace do
		if parent:IsA("Model") and Players:GetPlayerFromCharacter(parent) then
			return true
		end
		parent = parent.Parent
	end
	return false
end

local function IsOutOfRange(obj)
	if obj:IsA("BasePart") then
		local x = obj.Position.X
		return x < -560 or x > -240
	end
end

local BASE_NAMES = {
	["baseplate"] = true, ["spawnlocation"] = true, ["spawn location"] = true, ["spawn"] = true,
}

local function IsBase(obj)
	if not obj:IsA("BasePart") then return false end
	local nameLower = obj.Name:lower()
	if BASE_NAMES[nameLower] then return true end
	for n in pairs(BASE_NAMES) do
		if nameLower:find(n, 1, true) then return true end
	end
	return false
end

local function IsInBase(obj)
	local p = obj.Parent
	while p and p ~= workspace do
		if IsBase(p) then return true end
		p = p.Parent
	end
	return false
end

local function MakeTransparentUltra(obj)
	pcall(function()
		if IsBase(obj) and not IsCharacterPart(obj) then
			if OriginalTransparency[obj] == nil then
				OriginalTransparency[obj] = {trans = obj.Transparency, shadow = obj.CastShadow}
			end
			obj.Transparency = 1
			obj.CastShadow   = false
		end
	end)
end

local function StripObjectUltra(obj)
	pcall(function()
		if obj:IsA("Texture") or obj:IsA("Decal") or obj:IsA("SpecialMesh") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam")
			or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
			pcall(function() obj.Enabled = false end)
			SafeDestroyUltra(obj)
		elseif obj:IsA("SurfaceAppearance") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("BasePart") then
			obj.CastShadow      = false
			obj.Material        = Enum.Material.Plastic
			obj.MaterialVariant = ""
			obj.Reflectance     = 0
		end
	end)
end

local function CleanObjectUltra(obj)
	pcall(function()
		if obj:IsA("SurfaceAppearance") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("Decal") or obj:IsA("Texture") then
			if not (obj.Name == "face" and obj.Parent and obj.Parent.Name == "Head") then
				SafeDestroyUltra(obj)
			end
		elseif obj:IsA("SpecialMesh") then
			obj.TextureId = ""
		elseif obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("Fire") or obj:IsA("Smoke") or obj:IsA("Sparkles") or obj:IsA("Explosion") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("Animation") or obj:IsA("AnimationController") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("BasePart") then
			obj.CastShadow      = false
			obj.Material        = Enum.Material.Plastic
			obj.MaterialVariant = ""
			obj.Reflectance     = 0
		end
	end)
end

local function StopAnimationsUltra(animator)
	pcall(function()
		for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
			local isChar = false
			for _, plr in ipairs(Players:GetPlayers()) do
				if plr.Character and animator:IsDescendantOf(plr.Character) then
					isChar = true
					break
				end
			end
			if not isChar then
				track:Stop()
			end
		end
	end)
end

local function OptimizeCharacterUltra(char)
	if not char then return end
	task.spawn(function()
		task.wait(0.3)
		for _, obj in ipairs(char:GetDescendants()) do
			if IsClothing(obj) then
				SafeDestroyUltra(obj)
			else
				CleanObjectUltra(obj)
			end
		end
	end)
end

local function ApplyGreySkyUltra()
	pcall(function()
		for _, obj in ipairs(Lighting:GetChildren()) do
			if obj:IsA("Sky") then
				obj:Destroy()
			end
		end
		local sky        = Instance.new("Sky")
		sky.SkyboxBk     = ""
		sky.SkyboxDn     = ""
		sky.SkyboxFt     = ""
		sky.SkyboxLf     = ""
		sky.SkyboxRt     = ""
		sky.SkyboxUp     = ""
		sky.CelestialBodiesShown = false
		sky.Parent       = Lighting
	end)
end

local function OptimizeLightingUltra()
	Lighting.GlobalShadows            = false
	Lighting.FogEnd                   = 9e9
	Lighting.FogStart                 = 9e9
	Lighting.EnvironmentDiffuseScale  = 0
	Lighting.EnvironmentSpecularScale = 0
	Lighting.Brightness               = 1.5
	Lighting.Ambient                  = Color3.fromRGB(60, 60, 60)

	for _, v in ipairs(Lighting:GetChildren()) do
		if v:IsA("PostEffect") then
			pcall(function() v.Enabled = false end)
		elseif v:IsA("Atmosphere") or v:IsA("Clouds") then
			v:Destroy()
		end
	end
	ApplyGreySkyUltra()
end

local function ApplyTerrainUltra()
	pcall(function()
		local T = Workspace.Terrain
		T.Decoration        = false
		T.WaterWaveSize     = 0
		T.WaterWaveSpeed    = 0
		T.WaterReflectance  = 0
		T.WaterTransparency = 1
	end)
end

local function setFPSBoostUltra(enabled)
    Config.FPSBoostUltra = enabled
    saveConfig()
    setToggle("FPS Boost Ultra", enabled)
    
    if enabled then
        pcall(function()
            settings().Rendering.QualityLevel        = Enum.QualityLevel.Level01
            settings().Rendering.MeshPartDetailLevel = Enum.MeshPartDetailLevel.Level01
            settings().Physics.AllowSleep = true
            settings().Physics.PhysicsEnvironmentalThrottle = Enum.PhysicsEnvironmentalThrottle or Enum.EnviromentalPhysicsThrottle.Skip
        end)
        pcall(setfpscap, 0)   -- 0 = unlimited FPS (was 999)

        OptimizeLightingUltra()
        ApplyTerrainUltra()

        local allDesc = Workspace:GetDescendants()
        local BATCH_SIZE = 200
        for i = 1, #allDesc, BATCH_SIZE do
            local batchEnd = math.min(i + BATCH_SIZE - 1, #allDesc)
            for j = i, batchEnd do
                local obj = allDesc[j]
                if obj and obj.Parent then
                    if IsBase(obj) then
                        MakeTransparentUltra(obj)
                    elseif IsClothing(obj) then
                        SafeDestroyUltra(obj)
                    elseif IsInBase(obj) then
                        -- skip
                    elseif IsCharacterPart(obj) then
                        -- skip
                    elseif IsOutOfRange(obj) then
                        SafeDestroyUltra(obj)
                    else
                        CleanObjectUltra(obj)
                        StripObjectUltra(obj)
                        if obj:IsA("Animator") then
                            StopAnimationsUltra(obj)
                        end
                    end
                end
            end
            if i + BATCH_SIZE <= #allDesc then task.wait() end
        end

        AddUltraConnection(Workspace.DescendantAdded:Connect(function(obj)
            task.defer(function()
                if not Config.FPSBoostUltra then return end
                if IsBase(obj) then
                    MakeTransparentUltra(obj)
                    return
                end
                if IsClothing(obj) then
                    SafeDestroyUltra(obj)
                elseif IsInBase(obj) then
                    -- skip
                elseif IsCharacterPart(obj) then
                    -- skip
                elseif IsOutOfRange(obj) then
                    SafeDestroyUltra(obj)
                else
                    CleanObjectUltra(obj)
                    StripObjectUltra(obj)
                    if obj:IsA("Animator") then
                        StopAnimationsUltra(obj)
                    end
                end
            end)
        end))

        AddUltraConnection(Lighting.DescendantAdded:Connect(function(obj)
            if obj:IsA("PostEffect") then
                pcall(function() obj.Enabled = false end)
            elseif obj:IsA("Atmosphere") or obj:IsA("Clouds") then
                SafeDestroyUltra(obj)
            end
        end))

        local MaterialService = game:GetService("MaterialService")
        AddUltraConnection(MaterialService.DescendantAdded:Connect(function(obj)
            SafeDestroyUltra(obj)
        end))

        for _, plr in ipairs(Players:GetPlayers()) do
            OptimizeCharacterUltra(plr.Character)
            AddUltraConnection(plr.CharacterAdded:Connect(OptimizeCharacterUltra))
        end

        AddUltraConnection(Players.PlayerAdded:Connect(function(plr)
            AddUltraConnection(plr.CharacterAdded:Connect(OptimizeCharacterUltra))
        end))

        -- GC Loop
        AddUltraThread(function()
            while Config.FPSBoostUltra do
                task.wait(15)
                pcall(function() collectgarbage("collect") end)
            end
        end)
    else
        -- Clean up all connections and threads
        for _, conn in ipairs(_ultraConnections) do
            if typeof(conn) == "RBXScriptConnection" then
                conn:Disconnect()
            end
        end
        _ultraConnections = {}
        for _, thr in ipairs(_ultraThreads) do
            pcall(function() task.cancel(thr) end)
        end
        _ultraThreads = {}

        -- Restore Baseplates opacity
        pcall(function()
            for part, data in pairs(OriginalTransparency) do
                if part and part.Parent then
                    part.Transparency = data.trans
                    part.CastShadow = data.shadow
                end
            end
        end)
        OriginalTransparency = {}

        pcall(function()
            settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
            settings().Rendering.MeshPartDetailLevel = Enum.MeshPartDetailLevel.Automatic
            Lighting.GlobalShadows = true
            Lighting.Brightness = 2
            Lighting.FogEnd = 100000
            Workspace.Terrain.WaterWaveSize = 0.15
            Workspace.Terrain.WaterWaveSpeed = 1
            Workspace.Terrain.WaterReflectance = 0.5
            Workspace.Terrain.WaterTransparency = 0.3
            Workspace.Terrain.Decoration = true
        end)
    end
end

-- ============================================================
-- X-RAY
-- ============================================================
local setXRay
do -- X-RAY SCOPE
local xrayOriginalTransparencies = setmetatable({}, {__mode = "k"})
local xrayConnections = {}
local xrayLoopId = 0

local function setXRayTargetTransparency(instance, alphaPercent, loopId)
    if not instance then return end
    if loopId and loopId ~= xrayLoopId then return end   -- X-ray already turned off

    local function apply(obj)
        if obj:IsA("BasePart") then
            if xrayOriginalTransparencies[obj] == nil then
                if obj.Transparency == alphaPercent then xrayOriginalTransparencies[obj] = 0
                else xrayOriginalTransparencies[obj] = obj.Transparency end
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig < 1 then
                local target = orig + (1 - orig) * alphaPercent
                if math.abs(obj.Transparency - target) > 0.01 then obj.Transparency = target end
            end
        elseif obj:IsA("TextLabel") or obj:IsA("TextButton") then
            if xrayOriginalTransparencies[obj] == nil then
                local t, b = obj.TextTransparency, obj.BackgroundTransparency
                if t == alphaPercent then t = 0 end
                if b == alphaPercent then b = 0 end
                xrayOriginalTransparencies[obj] = {text = t, bg = b}
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig.text < 1 then
                local targetText = orig.text + (1 - orig.text) * alphaPercent
                if math.abs(obj.TextTransparency - targetText) > 0.01 then obj.TextTransparency = targetText end
            end
            if orig.bg < 1 then
                local targetBg = orig.bg + (1 - orig.bg) * alphaPercent
                if math.abs(obj.BackgroundTransparency - targetBg) > 0.01 then obj.BackgroundTransparency = targetBg end
            end
        elseif obj:IsA("Frame") or obj:IsA("ScrollingFrame") then
            if xrayOriginalTransparencies[obj] == nil then
                if obj.BackgroundTransparency == alphaPercent then xrayOriginalTransparencies[obj] = 0
                else xrayOriginalTransparencies[obj] = obj.BackgroundTransparency end
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig < 1 then
                local target = orig + (1 - orig) * alphaPercent
                if math.abs(obj.BackgroundTransparency - target) > 0.01 then obj.BackgroundTransparency = target end
            end
        elseif obj:IsA("ImageLabel") or obj:IsA("ImageButton") then
            if xrayOriginalTransparencies[obj] == nil then
                local i, b = obj.ImageTransparency, obj.BackgroundTransparency
                if i == alphaPercent then i = 0 end
                if b == alphaPercent then b = 0 end
                xrayOriginalTransparencies[obj] = {img = i, bg = b}
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig.img < 1 then
                local targetImg = orig.img + (1 - orig.img) * alphaPercent
                if math.abs(obj.ImageTransparency - targetImg) > 0.01 then obj.ImageTransparency = targetImg end
            end
            if orig.bg < 1 then
                local targetBg = orig.bg + (1 - orig.bg) * alphaPercent
                if math.abs(obj.BackgroundTransparency - targetBg) > 0.01 then obj.BackgroundTransparency = targetBg end
            end
        end
    end

    apply(instance)
    local descendants = instance:GetDescendants()
    for i, child in ipairs(descendants) do
        apply(child)
        if i % 300 == 0 then
            task.wait()
            if loopId and loopId ~= xrayLoopId then return end   -- aborted: X-ray turned off mid-apply
        end
    end
end

local XRAY_FOLDERS = {"Base","PlotSign","FriendPanel","Cash","Laser","Decorations","Skin","Unlock","Purchases"}

-- X-ray a subtree once, then keep newly-streamed parts x-rayed via a cheap
-- DescendantAdded hook. Event-driven -> ZERO cost while nothing changes
-- (replaces the old 1.5s full re-walk that caused the lag).
local function trackXRaySubtree(root, alphaPercent, loopId)
    if not root then return end
    if loopId ~= xrayLoopId then return end
    setXRayTargetTransparency(root, alphaPercent, loopId)
    if loopId ~= xrayLoopId then return end   -- don't re-arm a listener after X-ray was turned off
    xrayConnections[#xrayConnections+1] = root.DescendantAdded:Connect(function(obj)
        if loopId ~= xrayLoopId then return end
        setXRayTargetTransparency(obj, alphaPercent, loopId)
    end)
end

local function processPlotXRay(plot, alphaPercent, loopId)
    if not plot then return end
    if loopId ~= xrayLoopId then return end

    for _, fname in ipairs(XRAY_FOLDERS) do
        if loopId ~= xrayLoopId then return end
        trackXRaySubtree(plot:FindFirstChild(fname), alphaPercent, loopId)
    end
    if loopId ~= xrayLoopId then return end
    -- target folders that stream in after enable
    xrayConnections[#xrayConnections+1] = plot.ChildAdded:Connect(function(child)
        if loopId ~= xrayLoopId then return end
        for _, fname in ipairs(XRAY_FOLDERS) do
            if child.Name == fname then trackXRaySubtree(child, alphaPercent, loopId); break end
        end
    end)

    local animalPodiums = plot:FindFirstChild("AnimalPodiums")
    if animalPodiums then
        local function processPodium(podium)
            for _, child in ipairs(podium:GetChildren()) do
                if child.Name == "Claim" then
                    trackXRaySubtree(child, alphaPercent, loopId)
                elseif child.Name == "Base" then
                    trackXRaySubtree(child:FindFirstChild("Decorations"), alphaPercent, loopId)
                elseif child:IsA("Model") and child.Name ~= "Decorations" then
                    trackXRaySubtree(child, alphaPercent, loopId)
                end
            end
        end
        for _, podium in ipairs(animalPodiums:GetChildren()) do processPodium(podium) end
        xrayConnections[#xrayConnections+1] = animalPodiums.ChildAdded:Connect(function(podium)
            if loopId ~= xrayLoopId then return end
            task.wait(0.1)
            if loopId ~= xrayLoopId then return end
            processPodium(podium)
        end)
    end
end

local function applyTransparencyToAllPlotsXRay(alphaPercent, loopId)
    local plotsFolder = Workspace:FindFirstChild("Plots")
    if not plotsFolder then return end

    for _, plot in ipairs(plotsFolder:GetChildren()) do
        if loopId ~= xrayLoopId then return end
        processPlotXRay(plot, alphaPercent, loopId)
        task.wait()
    end
    -- new bases joining later
    xrayConnections[#xrayConnections+1] = plotsFolder.ChildAdded:Connect(function(plot)
        if loopId ~= xrayLoopId then return end
        task.wait(0.2)
        processPlotXRay(plot, alphaPercent, loopId)
    end)
end

function setXRay(enabled)
    Config.XRay = enabled
    saveConfig()
    setToggle("XRay", enabled)
    setToggle("X-Ray", enabled)
    setToggle("Xray", enabled)

    -- Clean up previous connections
    for _, conn in ipairs(xrayConnections) do
        if typeof(conn) == "RBXScriptConnection" then
            conn:Disconnect()
        end
    end
    xrayConnections = {}

    xrayLoopId = xrayLoopId + 1
    local currentLoopId = xrayLoopId

    if enabled then
        local alphaPercent = 0.5
        
        task.spawn(function()
            -- AÈ™teptÄƒm sÄƒ se Ã®ncarce folder-ul Plots
            while currentLoopId == xrayLoopId and not Workspace:FindFirstChild("Plots") do
                task.wait(0.5)
            end
            
            if currentLoopId ~= xrayLoopId then return end
            
            -- Initial call cu un mic delay sÄƒ dÄƒm timp primelor modele sÄƒ aparÄƒ
            if currentLoopId ~= xrayLoopId then return end
            pcall(applyTransparencyToAllPlotsXRay, alphaPercent, currentLoopId)
            
            -- Loop continuu È™i super-optimizat (fÄƒrÄƒ overlap de thread-uri)
            -- (perpetual 1.5s re-walk removed; upkeep is now event-driven)
        end)
    else
        -- Turn OFF instantly AND completely: swap the snapshot out atomically, then
        -- restore every captured original in ONE synchronous pass (no task.wait
        -- yields). This removes the old race where a restore could overlap a
        -- re-enable and leave some parts still see-through after toggling off.
        local snapshot = xrayOriginalTransparencies
        xrayOriginalTransparencies = setmetatable({}, {__mode = "k"})
        for obj, orig in pairs(snapshot) do
            pcall(function()
                if obj:IsA("BasePart") then
                    obj.Transparency = orig
                elseif obj:IsA("TextLabel") or obj:IsA("TextButton") then
                    obj.TextTransparency = orig.text
                    obj.BackgroundTransparency = orig.bg
                elseif obj:IsA("Frame") or obj:IsA("ScrollingFrame") then
                    obj.BackgroundTransparency = orig
                elseif obj:IsA("ImageLabel") or obj:IsA("ImageButton") then
                    obj.ImageTransparency = orig.img
                    obj.BackgroundTransparency = orig.bg
                end
            end)
        end
        -- User setting: X-ray OFF forces bases solid even when FPS Boost Ultra hid
        -- them. FPS Boost Ultra records each base's true transparency before hiding
        -- it, so re-apply those originals here -> hidden bases become visible again.
        for part, data in pairs(OriginalTransparency) do
            if part and part.Parent and typeof(data) == "table" then
                pcall(function() part.Transparency = data.trans end)
            end
        end
    end
end

_G.setXRay = setXRay
end -- END X-RAY SCOPE


-- ============================================================
-- ANTI BEE & DISCO
-- ============================================================
SharedState.ANTI_BEE_DISCO = {
    running = false,
    connections = {},
    originalMoveFunction = nil,
    controlsProtected = false,
    badLightingNames = { Blue = true, DiscoEffect = true, BeeBlur = true, ColorCorrection = true },
}
SharedState.ANTI_BEE_DISCO.nuke = function(obj)
    if not obj or not obj.Parent then return end
    if SharedState.ANTI_BEE_DISCO.badLightingNames[obj.Name] then
        pcall(function() obj:Destroy() end)
    end
end
SharedState.ANTI_BEE_DISCO.disconnectAll = function()
    for _, conn in ipairs(SharedState.ANTI_BEE_DISCO.connections) do
        if typeof(conn) == "RBXScriptConnection" then conn:Disconnect() end
    end
    SharedState.ANTI_BEE_DISCO.connections = {}
end
SharedState.ANTI_BEE_DISCO.protectControls = function()
    if SharedState.ANTI_BEE_DISCO.controlsProtected then return end
    pcall(function()
        local PlayerScripts = LocalPlayer.PlayerScripts
        local PlayerModule = PlayerScripts:FindFirstChild("PlayerModule")
        if not PlayerModule then return end
        local Controls = require(PlayerModule):GetControls()
        if not Controls then return end
        local ab = SharedState.ANTI_BEE_DISCO
        if not ab.originalMoveFunction then ab.originalMoveFunction = Controls.moveFunction end
        local function protectedMoveFunction(self, moveVector, relativeToCamera)
            if ab.originalMoveFunction then ab.originalMoveFunction(self, moveVector, relativeToCamera) end
        end
        table.insert(ab.connections, RunService.Heartbeat:Connect(function()
            if not ab.running or not Config.AntiBeeDisco then return end
            if _G._isTpMoving then return end
            if Controls.moveFunction ~= protectedMoveFunction then Controls.moveFunction = protectedMoveFunction end
        end))
        Controls.moveFunction = protectedMoveFunction
        ab.controlsProtected = true
    end)
end
SharedState.ANTI_BEE_DISCO.restoreControls = function()
    if not SharedState.ANTI_BEE_DISCO.controlsProtected then return end
    pcall(function()
        local PlayerModule = LocalPlayer.PlayerScripts:FindFirstChild("PlayerModule")
        if not PlayerModule then return end
        local Controls = require(PlayerModule):GetControls()
        local ab = SharedState.ANTI_BEE_DISCO
        if Controls and ab.originalMoveFunction then
            Controls.moveFunction = ab.originalMoveFunction
            ab.controlsProtected = false
        end
    end)
end
SharedState.ANTI_BEE_DISCO.blockBuzzingSound = function()
    pcall(function()
        local beeScript = LocalPlayer.PlayerScripts:FindFirstChild("Bee", true)
        if beeScript then
            local buzzing = beeScript:FindFirstChild("Buzzing")
            if buzzing and buzzing:IsA("Sound") then
                buzzing:Stop()
                buzzing.Volume = 0
            end
        end
    end)
end
SharedState.ANTI_BEE_DISCO.Enable = function()
    local ab = SharedState.ANTI_BEE_DISCO
    if ab.running then return end
    ab.running = true
    for _, inst in ipairs(Lighting:GetDescendants()) do ab.nuke(inst) end
    table.insert(ab.connections, Lighting.DescendantAdded:Connect(function(obj)
        if not ab.running or not Config.AntiBeeDisco then return end
        ab.nuke(obj)
    end))
    ab.protectControls()
    table.insert(ab.connections, RunService.Heartbeat:Connect(function()
        if not ab.running or not Config.AntiBeeDisco then return end
        ab.blockBuzzingSound()
    end))
    ShowNotification("ANTIBEE & DISCO", "Enabled")
end
SharedState.ANTI_BEE_DISCO.Disable = function()
    local ab = SharedState.ANTI_BEE_DISCO
    if not ab.running then return end
    ab.running = false
    ab.restoreControls()
    ab.disconnectAll()
    ShowNotification("ANTI-BEE & DISCO", "Disabled")
end
_G.ANTI_BEE_DISCO = SharedState.ANTI_BEE_DISCO
if Config.AntiBeeDisco then
    task.delay(1, function()
        if SharedState.ANTI_BEE_DISCO.Enable then SharedState.ANTI_BEE_DISCO.Enable() end
    end)
end

-- ============================================================
-- AUTO BUY 
-- ============================================================
local toggleAutoBuy
do -- AUTO BUY SCOPE (NOT lazy: toggleAutoBuy must exist immediately for the UI)
local autoBuyActive = false
local autoBuyRing = nil

local function createAutoBuyRing()
    local existing = Workspace:FindFirstChild("XiAutoBuyRing")
    if existing then existing:Destroy() end
    local r = Instance.new("Part")
    r.Name = "XiAutoBuyRing"
    r.Shape = Enum.PartType.Cylinder
    r.Anchored = true
    r.CanCollide = false
    r.CanTouch = false
    r.CanQuery = false
    r.CastShadow = false
    r.Material = Enum.Material.Neon
    r.Transparency = 0.5
    r.Color = Theme.Accent
    local range = Config.AutoBuyRange or 17
    r.Size = Vector3.new(0.5, range * 2, range * 2)
    r.Parent = Workspace
    autoBuyRing = r
end

local function destroyAutoBuyRing()
    if autoBuyRing then autoBuyRing:Destroy(); autoBuyRing = nil end
    local e = Workspace:FindFirstChild("XiAutoBuyRing")
    if e then e:Destroy() end
end

local _abRingFrame = 0
RunService.Heartbeat:Connect(function()
    if not autoBuyActive then return end
    _abRingFrame = _abRingFrame + 1
    if _abRingFrame < 3 then return end
    _abRingFrame = 0
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp or not autoBuyRing then return end
    local range = Config.AutoBuyRange or 17
    autoBuyRing.Size = Vector3.new(0.5, range * 2, range * 2)
    autoBuyRing.CFrame = (hrp.CFrame * CFrame.Angles(0, 0, math.rad(90))) + Vector3.new(0, -2.5, 0)
end)

-- Explicit assignment to the OUTER `local toggleAutoBuy` declared at line ~6899.
-- Was previously `function toggleAutoBuy(on)` -- syntactic sugar that some
-- contexts can mis-resolve; this form guarantees we update the outer upvalue
-- the UI closure at line 8131 captures.
toggleAutoBuy = function(on)
    if on ~= nil then
        autoBuyActive = on
    else
        autoBuyActive = not autoBuyActive
    end
    Config.AutoBuyEnabled = autoBuyActive
    pcall(saveConfig)
    pcall(setToggle, "Auto Buy", autoBuyActive)
    if autoBuyActive then createAutoBuyRing() else destroyAutoBuyRing() end
    pcall(ShowNotification, "AUTO BUY", autoBuyActive and "ENABLED" or "DISABLED")
    if _G.AutoBuyOnToggle then
        pcall(_G.AutoBuyOnToggle, autoBuyActive)
    end
end

local RARITY_WORDS = {
    common = true, uncommon = true, rare = true, epic = true,
    legendary = true, secret = true, divine = true, rainbow = true,
    cursed = true, gold = true, diamond = true,
}

local function getBrainrotName(model)
    if not model then return "Brainrot", "" end
    local nameFound, genFound = "", ""
    for _, bb in ipairs(model:GetDescendants()) do
        if bb:IsA("BillboardGui") then
            for _, lbl in ipairs(bb:GetDescendants()) do
                if lbl:IsA("TextLabel") and lbl.Text and lbl.Text ~= "" then
                    local t = lbl.Text:match("^%s*(.-)%s*$")
                    local tl = t:lower()
                    if RARITY_WORDS[tl] then continue end
                    if t:match("^%$[%d%.]+[KkMmBb]?/s$") then
                        if genFound == "" then genFound = t end
                        continue
                    end
                    if t:match("^%$[%d%.]+[KkMmBb]?$") then continue end
                    if t:match("^[%d%.]+[KkMmBb]?$") then continue end
                    if nameFound == "" and #t > 1 then nameFound = t end
                end
            end
        end
    end
    if nameFound == "" then
        pcall(function()
            local info = AnimalsData[model.Name]
            if info and info.DisplayName then
                nameFound = info.DisplayName
                local gv = AnimalsShared:GetGeneration(model.Name, nil, nil, nil)
                local gt = "$" .. NumberUtils:ToString(gv) .. "/s"
                genFound = gt
            end
        end)
    end
    if nameFound == "" then nameFound = model.Name ~= "" and model.Name or "Brainrot" end
    return nameFound, genFound
end

local function scanConveyor()
    local results = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if not (obj:IsA("ProximityPrompt") and obj.Enabled) then continue end
        local txt = obj.ActionText or ""
        if not (txt == "Purchase" or txt:lower():find("purchase") or txt:lower():find("comprar")) then continue end
        local part = obj.Parent
        if not part then continue end
        local realPart = (part:IsA("Attachment") and part.Parent) or part
        if not (realPart and realPart:IsA("BasePart")) then continue end
        local model, cur = nil, realPart
        for _ = 1, 8 do
            if cur and cur:IsA("Model") then model = cur; break end
            cur = cur and cur.Parent
        end
        local name, gen = getBrainrotName(model)
        table.insert(results, {
            name = name,
            gen = gen,
            prompt = obj,
            part = realPart,
            model = model,
            source = "ESTEIRA",
            uid = "esteira_" .. tostring(obj),
        })
    end
    return results
end

SharedState.ConveyorAnimals = {}
local function refreshConveyor()
    local ok, found = pcall(scanConveyor)
    if ok and found then
        SharedState.ConveyorAnimals = found
    end
end
refreshConveyor()
_G.refreshConveyor = refreshConveyor

local purchaseRemote = nil
local function resolvePurchaseRemote()
    if purchaseRemote and purchaseRemote.Parent then return purchaseRemote end
    pcall(function()
        local net = ReplicatedStorage:FindFirstChild("Packages") and ReplicatedStorage.Packages:FindFirstChild("Net")
        if not net then return end
        local kws = {"buy", "purchase", "animal", "shop", "acquire", "conveyor"}
        for _, v in ipairs(net:GetChildren()) do
            local nl = (v.Name or ""):lower()
            for _, kw in ipairs(kws) do
                if nl:find(kw) then
                    purchaseRemote = v
                    return
                end
            end
        end
        local paths = {"RF/ShopService/BuyAnimal", "RF/AnimalShop/Purchase", "RE/Shop/Buy", "RF/Shop/Buy"}
        for _, p in ipairs(paths) do
            local ok2, r = pcall(function() return Decrypted[p] end)
            if ok2 and r and r.Parent then
                purchaseRemote = r
                return
            end
        end
    end)
    return purchaseRemote
end

local function firePurchaseNatural(prompt)
    if not prompt or not prompt.Parent or not prompt.Enabled then return end
    pcall(function()
        if fireproximityprompt then fireproximityprompt(prompt) end
    end)
    task.spawn(function()
        local remote = resolvePurchaseRemote()
        if remote then
            pcall(function()
                if remote:IsA("RemoteFunction") then
                    remote:InvokeServer(prompt.Parent)
                elseif remote:IsA("RemoteEvent") then
                    remote:FireServer(prompt.Parent)
                end
            end)
        end
    end)
end

local carpetLockConn = nil
local function startCarpetLock()
    if carpetLockConn then carpetLockConn:Disconnect(); carpetLockConn = nil end
    local function ensureCarpet()
        pcall(function()
            local char = LocalPlayer.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if not hum then return end
            local toolName = (Config.TpSettings and Config.TpSettings.Tool) or "Flying Carpet"
            if not char:FindFirstChild(toolName) then
                local tool = LocalPlayer.Backpack:FindFirstChild(toolName)
                if tool then hum:EquipTool(tool) end
            end
        end)
    end
    task.spawn(function()
        for _ = 1, 15 do
            if not autoBuyActive then break end
            ensureCarpet()
            task.wait(0.3)
            local char = LocalPlayer.Character
            local toolName = (Config.TpSettings and Config.TpSettings.Tool) or "Flying Carpet"
            if char and char:FindFirstChild(toolName) then break end
        end
    end)
    carpetLockConn = RunService.Heartbeat:Connect(function()
        if not autoBuyActive then return end
        ensureCarpet()
    end)
end

local function stopCarpetLock()
    if carpetLockConn then carpetLockConn:Disconnect(); carpetLockConn = nil end
end

local HOVER_HEIGHT = 5
local BUY_INTERVAL = 0.08
local DETECT_RADIUS = 17
local lockedTarget = nil
local lockedPart = nil
local lockedModel = nil

local function partAlive()
    return lockedPart and lockedPart.Parent and lockedModel and lockedModel.Parent
end

local function promptAlive()
    return lockedTarget and lockedTarget.prompt and lockedTarget.prompt.Parent and lockedTarget.prompt.Enabled
end

local bodyPos = nil
local function ensureBodyPos(hrp)
    if bodyPos and bodyPos.Parent == hrp then
        local speed = math.clamp(Config.AutoGrabSpeed or 17, 5, 100)
        bodyPos.P = speed * 1000
        return bodyPos
    end
    if bodyPos then bodyPos:Destroy() end
    local speed = math.clamp(Config.AutoGrabSpeed or 17, 5, 100)
    local bp = Instance.new("BodyPosition", hrp)
    bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bp.P = speed * 1000
    bp.D = 1000
    bp.Position = hrp.Position
    bodyPos = bp
    return bp
end

local function destroyBodyPos()
    if bodyPos then bodyPos:Destroy(); bodyPos = nil end
end

RunService.Heartbeat:Connect(function()
    if not autoBuyActive or not partAlive() then
        destroyBodyPos()
        return
    end
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then destroyBodyPos(); return end
    local above = lockedPart.Position + Vector3.new(0, HOVER_HEIGHT, 0)
    local bp = ensureBodyPos(hrp)
    bp.Position = above
end)

task.spawn(function()
    while true do
        task.wait(BUY_INTERVAL)
        if not autoBuyActive then continue end
        if not partAlive() then continue end
        if promptAlive() then
            firePurchaseNatural(lockedTarget.prompt)
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(0.25)
        if not autoBuyActive then
            lockedTarget = nil
            lockedPart = nil
            lockedModel = nil
            stopCarpetLock()
            destroyBodyPos()
            continue
        end
        if lockedPart or lockedModel then
            if not partAlive() then
                ShowNotification("AUTO BUY", "Reached base, scanning...")
                pcall(refreshConveyor)
                lockedTarget = nil
                lockedPart = nil
                lockedModel = nil
            end
            continue
        end
        pcall(refreshConveyor)
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        local radius = Config.AutoBuyRange or DETECT_RADIUS
        local best, bestDist = nil, math.huge
        for _, entry in ipairs(SharedState.ConveyorAnimals) do
            if entry.prompt and entry.prompt.Parent and entry.prompt.Enabled and entry.part and entry.part.Parent then
                local d = (hrp.Position - entry.part.Position).Magnitude
                if d <= radius and d < bestDist then
                    bestDist = d
                    best = entry
                end
            end
        end
        if best then
            lockedTarget = best
            lockedPart = best.part
            lockedModel = best.model or best.part.Parent
            ShowNotification("AUTO BUY", "Locked: " .. best.name)
            startCarpetLock()
        end
    end
end)

_G.AutoBuyOnToggle = function(active)
    if active then
        if _G.refreshConveyor then pcall(_G.refreshConveyor) end
        startCarpetLock()
    else
        stopCarpetLock()
        destroyBodyPos()
    end
end
end -- END AUTO BUY SCOPE

-- ============================================================
-- ============================================================

local function hasExclamation(target) for _,d in ipairs(target:GetDescendants()) do if d:IsA("BillboardGui") then
    local label=d:FindFirstChildWhichIsA("TextLabel",true); if label and label.Text:find("!") then return true end
end end; return false end

task.spawn(function()
    while true do task.wait(0.1)
        if Config.AutoDestroyTurrets then
            if LocalPlayer:GetAttribute("Stealing")~=true then
                local char=LocalPlayer.Character; local hrp=char and char:FindFirstChild("HumanoidRootPart"); local hum=char and char:FindFirstChildOfClass("Humanoid")
                if hrp and hum then
                    local closest,shortestDist=nil,math.huge
                    for _,inst in ipairs(Workspace:GetDescendants()) do if inst.Name:match("^Sentry_") and hasExclamation(inst) then
                        local root=inst:IsA("BasePart") and inst or inst:FindFirstChildWhichIsA("BasePart",true)
                        if root then
                            local dist=(hrp.Position-root.Position).Magnitude
                            if dist <= 29 and dist<shortestDist then shortestDist=dist; closest=inst end
                        end
                    end end
                    if closest then
                        -- Make turret visible and move in front of player
                        for _,d in ipairs(closest:GetDescendants()) do if d:IsA("BasePart") then d.Transparency=0.5; d.CanCollide=false end end
                        local offset=hrp.CFrame.LookVector*4; local targetCF=CFrame.new(hrp.Position+offset,hrp.Position)
                        if closest:IsA("Model") then closest:PivotTo(targetCF) elseif closest:IsA("BasePart") then closest.CFrame=targetCF end
                        local bat=player.Backpack:FindFirstChild("Bat") or char:FindFirstChild("Bat")
                        if bat then if bat.Parent~=char then hum:EquipTool(bat) end; bat:Activate() end
                    end
                end
            end
        end
    end
end)

-- ============================================================
-- REMOTE SELLER
-- ============================================================
LazyInit("Remote Sell UI", function() -- REMOTE SELL SCOPE
local remoteSellGui=nil; local remoteSellBuilt=false
local function buildRemoteSell()
    if remoteSellBuilt then return end; remoteSellBuilt=true
    local Synchronizer_rs=nil; local AnimalsData_rs,AnimalsShared_rs,NumberUtils_rs=nil,nil,nil
    pcall(function()
        local Pkgs=ReplicatedStorage:WaitForChild("Packages",10); local Datas=ReplicatedStorage:WaitForChild("Datas",10)
        local Shared=ReplicatedStorage:WaitForChild("Shared",10); local Utils=ReplicatedStorage:WaitForChild("Utils",10)
        if Pkgs then Synchronizer_rs=require(Pkgs:WaitForChild("Synchronizer")) end
        if Datas then AnimalsData_rs=require(Datas:WaitForChild("Animals")) end
        if Shared then AnimalsShared_rs=require(Shared:WaitForChild("Animals")) end
        if Utils then NumberUtils_rs=require(Utils:WaitForChild("NumberUtils")) end
    end)

    local function findMyPlotRS()
        local plots=Workspace:FindFirstChild("Plots"); if not plots then return nil end
        for _,plot in ipairs(plots:GetChildren()) do
            if Synchronizer_rs then local ok,ch=pcall(function() return Synchronizer_rs:Get(plot.Name) end)
                if ok and ch then local owner=ch:Get("Owner")
                    if (typeof(owner)=="Instance" and owner==player) or (typeof(owner)=="table" and owner.UserId==player.UserId) then return plot end
                end
            end
            local sign=plot:FindFirstChild("PlotSign"); if sign then local sg=sign:FindFirstChildWhichIsA("SurfaceGui",true)
                if sg then local lbl=sg:FindFirstChildWhichIsA("TextLabel",true)
                    if lbl then local t=lbl.Text:lower(); if t:find(player.Name:lower(),1,true) or t:find(player.DisplayName:lower(),1,true) then return plot end end
                end
            end
        end; return nil
    end

    remoteSellGui=Instance.new("ScreenGui"); remoteSellGui.Name="ALPHA_RemoteSell"; remoteSellGui.ResetOnSpawn=false; remoteSellGui.Parent=playerGui
    local rsFrame=Instance.new("Frame", registerScreenGui(remoteSellGui)); rsFrame.Size=UDim2.new(0,190,0,240)
    rsFrame.Position=UDim2.new(0,350,1,-350); rsFrame.BackgroundColor3=Theme.MainBackground; rsFrame.BackgroundTransparency=0.06; rsFrame.BorderSizePixel=0
    Instance.new("UICorner",rsFrame).CornerRadius=UDim.new(0,12)
    local rsStroke=Instance.new("UIStroke",rsFrame); rsStroke.Color=Theme.AccentLight; rsStroke.Thickness=1.25; rsStroke.Transparency=0.08
    applySavedPosition("Remote Sell", rsFrame)

    local rsHeader=Instance.new("Frame",rsFrame); rsHeader.Size=UDim2.new(1,0,0,30); rsHeader.BackgroundTransparency=1; rsHeader.Active=true
    local rsTitle=Instance.new("TextLabel",rsHeader); rsTitle.Size=UDim2.new(1,-20,0,30); rsTitle.Position=UDim2.new(0,10,0,5)
    rsTitle.BackgroundTransparency=1; rsTitle.Text="Remote Sell"; rsTitle.Font=Enum.Font.GothamBlack; rsTitle.TextSize=13; rsTitle.TextColor3=Theme.Text; rsTitle.TextXAlignment=Enum.TextXAlignment.Left
    
    makeDraggable(rsFrame, rsHeader, "Remote Sell")

    local rsSellAll=Instance.new("TextButton",rsFrame); rsSellAll.Size=UDim2.new(1,-18,0,28); rsSellAll.Position=UDim2.new(0,9,0,38)
    rsSellAll.BackgroundColor3=Theme.Accent; rsSellAll.Text="Proximity sell"; rsSellAll.TextColor3=Color3.new(1,1,1); rsSellAll.Font=Enum.Font.GothamBlack; rsSellAll.TextSize=12; rsSellAll.BorderSizePixel=0
    Instance.new("UICorner",rsSellAll).CornerRadius=UDim.new(0,8)

    local rsStatus=Instance.new("TextLabel",rsFrame); rsStatus.Size=UDim2.new(1,-18,0,14); rsStatus.Position=UDim2.new(0,9,0,72)
    rsStatus.BackgroundTransparency=1; rsStatus.Text="Waiting for scan..."; rsStatus.TextColor3=Theme.Dim; rsStatus.Font=Enum.Font.Gotham; rsStatus.TextSize=10; rsStatus.TextXAlignment=Enum.TextXAlignment.Left

    local rsScroll=Instance.new("ScrollingFrame",rsFrame); rsScroll.Size=UDim2.new(1,-18,1,-102); rsScroll.Position=UDim2.new(0,9,0,92)
    rsScroll.BackgroundColor3=Theme.Panel; rsScroll.BorderSizePixel=0; rsScroll.ScrollBarThickness=3; rsScroll.ScrollBarImageColor3=Theme.Accent; rsScroll.Active=true
    Instance.new("UICorner",rsScroll).CornerRadius=UDim.new(0,8)
    local rsLayout=Instance.new("UIListLayout",rsScroll); rsLayout.Padding=UDim.new(0,4); rsLayout.SortOrder=Enum.SortOrder.LayoutOrder
    rsLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() rsScroll.CanvasSize=UDim2.new(0,0,0,rsLayout.AbsoluteContentSize.Y+8) end)

    local lastScanData = {}
    local allSellRows = {}
    local sellingAll = false

    local function doScan()
        if sellingAll then return end
        local plot = findMyPlotRS()
        if not plot then
            if rsStatus.Text ~= "No base found" then
                for _, ch in ipairs(rsScroll:GetChildren()) do if not ch:IsA("UIListLayout") then ch:Destroy() end end
                allSellRows = {}
                lastScanData = {}
                rsStatus.Text = "No base found"
            end
            return
        end
        local podiums = plot:FindFirstChild("AnimalPodiums")
        if not podiums then
            if rsStatus.Text ~= "No podiums" then
                for _, ch in ipairs(rsScroll:GetChildren()) do if not ch:IsA("UIListLayout") then ch:Destroy() end end
                allSellRows = {}
                lastScanData = {}
                rsStatus.Text = "No podiums"
            end
            return
        end

        local scanData = {}
        for _, podium in ipairs(podiums:GetChildren()) do
            local podiumNum = tonumber(podium.Name)
            if podiumNum then
                local base = podium:FindFirstChild("Base")
                local spawn_p = base and base:FindFirstChild("Spawn")
                local att = spawn_p and spawn_p:FindFirstChild("PromptAttachment")
                if att then
                    for _, pp in ipairs(att:GetChildren()) do
                        if pp:IsA("ProximityPrompt") and (pp.ActionText or ""):sub(1, 4) == "Sell" and pp.Enabled then
                            local name = "Slot " .. podiumNum
                            if Synchronizer_rs and AnimalsData_rs then
                                pcall(function()
                                    local ch = Synchronizer_rs:Get(plot.Name)
                                    if ch then
                                        local al = ch:Get("AnimalList")
                                        if al then
                                            local entry = al[podiumNum] or al[tostring(podiumNum)]
                                            if type(entry) == "table" and entry.Index then
                                                local info = AnimalsData_rs[entry.Index]
                                                name = (info and info.DisplayName) or entry.Index
                                            end
                                        end
                                    end
                                end)
                            end
                            table.insert(scanData, {
                                num = podiumNum,
                                prompt = pp,
                                name = name
                            })
                        end
                    end
                end
            end
        end
        table.sort(scanData, function(a, b) return a.num < b.num end)

        local changed = false
        if #scanData ~= #lastScanData then
            changed = true
        else
            for idx, item in ipairs(scanData) do
                local prev = lastScanData[idx]
                if prev.num ~= item.num or prev.prompt ~= item.prompt or prev.name ~= item.name then
                    changed = true
                    break
                end
            end
        end

        if changed then
            for _, ch in ipairs(rsScroll:GetChildren()) do if not ch:IsA("UIListLayout") then ch:Destroy() end end
            allSellRows = {}
            for _, item in ipairs(scanData) do
                local row = Instance.new("Frame", rsScroll)
                row.Size = UDim2.new(1, -8, 0, 30)
                row.BackgroundColor3 = Theme.Row
                row.BorderSizePixel = 0
                row.LayoutOrder = item.num
                Instance.new("UICorner", row).CornerRadius = UDim.new(0, 6)

                local nl = Instance.new("TextLabel", row)
                nl.Size = UDim2.new(1, -60, 1, 0)
                nl.Position = UDim2.new(0, 8, 0, 0)
                nl.BackgroundTransparency = 1
                nl.Text = item.name
                nl.Font = Enum.Font.GothamBold
                nl.TextSize = 10
                nl.TextColor3 = Theme.Text
                nl.TextXAlignment = Enum.TextXAlignment.Left
                nl.TextTruncate = Enum.TextTruncate.AtEnd
                nl.Parent = row

                local sb = Instance.new("TextButton", row)
                sb.Size = UDim2.new(0, 44, 0, 22)
                sb.Position = UDim2.new(1, -50, 0.5, -11)
                sb.BackgroundColor3 = Theme.Accent
                sb.Text = "SELL"
                sb.TextColor3 = Color3.new(1, 1, 1)
                sb.Font = Enum.Font.GothamBlack
                sb.TextSize = 10
                sb.BorderSizePixel = 0
                Instance.new("UICorner", sb).CornerRadius = UDim.new(0, 6)
                sb.Parent = row

                local prompt = item.prompt
                local function doSell()
                    pcall(function() fireproximityprompt(prompt) end)
                    sb.Text = "✔"
                    sb.BackgroundColor3 = Theme.Green
                    task.delay(0.5, function()
                        if row.Parent then
                            row:Destroy()
                        end
                    end)
                end
                sb.MouseButton1Click:Connect(doSell)
                table.insert(allSellRows, { fn = doSell, prompt = prompt })
            end
            lastScanData = scanData
        end

        local found = #scanData
        rsStatus.Text = (found == 0 and "Nothing to sell") or (found .. " brainrot(s) ready")
    end

    -- scan happens automatically every 0.3 seconds
    rsSellAll.MouseButton1Click:Connect(function()
        if sellingAll then return end
        sellingAll = true
        rsSellAll.Text="Selling..."
        task.spawn(function()
            local plot = findMyPlotRS()
            if plot then
                local podiums = plot:FindFirstChild("AnimalPodiums")
                if podiums then
                    local prompts = {}
                    for _, podium in ipairs(podiums:GetChildren()) do
                        local base = podium:FindFirstChild("Base")
                        local spawn_p = base and base:FindFirstChild("Spawn")
                        local att = spawn_p and spawn_p:FindFirstChild("PromptAttachment")
                        if att then
                            for _, pp in ipairs(att:GetChildren()) do
                                if pp:IsA("ProximityPrompt") and (pp.ActionText or ""):sub(1, 4) == "Sell" and pp.Enabled then
                                    table.insert(prompts, pp)
                                end
                            end
                        end
                    end
                    for _, prompt in ipairs(prompts) do
                        if prompt and prompt.Parent then
                            pcall(function() fireproximityprompt(prompt) end)
                            task.wait(0.08)
                        end
                    end
                end
            end
            task.wait(0.3)
            rsSellAll.Text="SELL ALL"
            sellingAll = false
            pcall(doScan)
        end)
    end)

    -- Auto scan loop (every 0.3s)
    task.spawn(function()
        while remoteSellGui and remoteSellGui.Parent do
            task.wait(0.3)
            if remoteSellGui.Enabled then
                pcall(doScan)
            end
        end
    end)
end

_G.toggleRemoteSell=function(enabled) Config.RemoteSellEnabled=enabled; saveConfig()
    if enabled then buildRemoteSell(); if remoteSellGui then remoteSellGui.Enabled=true end
    else if remoteSellGui then remoteSellGui.Enabled=false end end
end
end) -- END REMOTE SELL SCOPE

-- ============================================================
-- GUI HELPERS
-- ============================================================
main,mainBody,tabBar,bottomBar,fpsText = nil,nil,nil,nil,nil
panels,panelSetters,tabButtons={},{},{}
actionSettingsPanel,actionSettingsBody,tpSpeedSettingsPanel,tpSpeedSettingsBody = nil,nil,nil,nil
BoundToggles={}

stealProgressBarGui = nil

-- Pre-declare helper functions made global to free local registers in outer scope

_G.ShowStealProgressBar = function(targetName, duration)
    local ExploitGui = (gethui and gethui()) or game:GetService("CoreGui")
    
    if stealProgressBarGui then
        pcall(function() stealProgressBarGui:Destroy() end)
    end
    
    local sg = Instance.new("ScreenGui")
    sg.Name = "ALPHA_StealProgressBar"
    sg.ResetOnSpawn = false
    sg.Parent = ExploitGui
    stealProgressBarGui = sg
    
    local barWidth = 270
    local barHeight = 50
    
    local container = Instance.new("Frame")
    container.Size = UDim2.fromOffset(barWidth, barHeight)
    container.Position = UDim2.new(0.5, -barWidth/2, 1, -190)
    container.BackgroundColor3 = Theme.Background
    container.BackgroundTransparency = 0.02
    container.BorderSizePixel = 0
    container.Parent = registerScreenGui(sg)
    corner(container, 12)
    addOutline(container)
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(0.6, -14, 0, 16)
    title.Position = UDim2.new(0, 14, 0, 8)
    title.BackgroundTransparency = 1
    title.Text = "⚡ STEALING: " .. (targetName or "Brainrot"):upper()
    title.TextColor3 = Theme.Text
    title.Font = Enum.Font.GothamBold
    title.TextSize = 9.5
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = container
    
    local pctLabel = Instance.new("TextLabel")
    pctLabel.Size = UDim2.new(0.4, -14, 0, 16)
    pctLabel.Position = UDim2.new(0.6, 0, 0, 8)
    pctLabel.BackgroundTransparency = 1
    pctLabel.Text = "0%"
    pctLabel.TextColor3 = Theme.AccentLight
    pctLabel.Font = Enum.Font.GothamBold
    pctLabel.TextSize = 10
    pctLabel.TextXAlignment = Enum.TextXAlignment.Right
    pctLabel.Parent = container
    
    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, -28, 0, 6)
    track.Position = UDim2.new(0, 14, 0, 30)
    track.BackgroundColor3 = Theme.SliderBg
    track.BorderSizePixel = 0
    track.Parent = container
    corner(track, 3)
    stroke(track, Theme.Stroke, 1, 0.1)
    
    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = Theme.Accent
    fill.BorderSizePixel = 0
    fill.Parent = track
    corner(fill, 3)
    
    local fillGrad = Instance.new("UIGradient", fill)
    fillGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Theme.AccentLight),
        ColorSequenceKeypoint.new(1, Theme.Accent)
    })
    
    local safeDuration = tonumber(duration) or 5
    if safeDuration <= 0.01 then safeDuration = 5 end
    local startTick = tick()
    
    task.spawn(function()
        while sg and sg.Parent do
            local elapsed = tick() - startTick
            local t = math.clamp(elapsed / safeDuration, 0, 1)
            local ratio = 1 - (1 - t)^3
            
            pcall(function()
                pctLabel.Text = math.floor(ratio * 100) .. "%"
                fill.Size = UDim2.new(ratio, 0, 1, 0)
            end)
            
            if t >= 1 then break end
            task.wait()
        end
    end)
end

_G.HideStealProgressBar = function()
    local targetGui = stealProgressBarGui
    if targetGui then
        stealProgressBarGui = nil
        pcall(function()
            local container = targetGui:FindFirstChildWhichIsA("Frame")
            if container then
                tw(container, {BackgroundTransparency = 1}, 0.15)
                for _, child in ipairs(container:GetDescendants()) do
                    if child:IsA("TextLabel") then
                        tw(child, {TextTransparency = 1}, 0.15)
                    elseif child:IsA("Frame") then
                        tw(child, {BackgroundTransparency = 1}, 0.15)
                    elseif child:IsA("UIStroke") then
                        tw(child, {Transparency = 1}, 0.15)
                    end
                end
                task.wait(0.16)
            end
            targetGui:Destroy()
        end)
    end
end

-- Assign helper implementations to pre-declared outer local variables
corner = function(o,r) local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,r); c.Parent=o; return c end
stroke = function(o,col,th,tr) local s=Instance.new("UIStroke"); s.Color=col or Theme.Stroke; s.Thickness=th or 1; s.Transparency=tr or 0; s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; s.Parent=o; return s end
tw = function(o,p,t) TweenService:Create(o,TweenInfo.new(t or 0.14,Enum.EasingStyle.Quint,Enum.EasingDirection.Out),p):Play() end
addOutline = function(f) local o=Instance.new("UIStroke"); o.Color=Theme.AccentLight; o.Thickness=1.25; o.Transparency=0.08; o.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; o.Parent=f; return o end
function clearBody(body) for _,c in ipairs(body:GetChildren()) do if not c:IsA("UIListLayout") and not c:IsA("UIPadding") then c:Destroy() end end end

function openAnim(f) if not f then return end; local us=f:FindFirstChild("SXEScale") or Instance.new("UIScale"); us.Name="SXEScale"; us.Parent=f
    local tgt=f.Position; f.Visible=true; us.Scale=0.92; f.Position=UDim2.new(tgt.X.Scale,tgt.X.Offset,tgt.Y.Scale,tgt.Y.Offset+18); tw(us,{Scale=1},0.20); tw(f,{Position=tgt},0.20) end
function closeAnim(f) if not f then return end; f.Visible = false end

makeDraggable = function(frame,handle,saveName) local dragging,dragStart,startPos=false,nil,nil
    handle.InputBegan:Connect(function(i) if UI.Locked then return end; if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=true; dragStart=i.Position; startPos=frame.Position end end)
    UIS.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then if dragging and saveName then rememberPosition(saveName,frame) end; dragging=false end end)
    UIS.InputChanged:Connect(function(i) if dragging and not UI.Locked and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
        local d=i.Position-dragStart
        local scale = getGlobalScale()
        frame.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+(d.X/scale),startPos.Y.Scale,startPos.Y.Offset+(d.Y/scale))
    end end)
end

makeResizable = function(frame, minSize, panelName)
    local h = Instance.new("TextButton")
    h.Size = UDim2.new(0, 16, 0, 16)
    h.Position = UDim2.new(1, -16, 1, -16)
    h.BackgroundTransparency = 1
    h.Text = "◢"
    h.TextColor3 = Theme.AccentLight or Color3.new(1, 1, 1)
    h.TextSize = 12
    h.ZIndex = 100
    h.Parent = frame
    local dragging, dragStart, startSize = false, nil, nil
    h.InputBegan:Connect(function(i)
        if UI.Locked then return end
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = i.Position; startSize = frame.AbsoluteSize
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            if dragging then
                dragging = false
                if panelName then
                    if not Config.sizes then Config.sizes = {} end
                    Config.sizes[panelName] = {x = frame.Size.X.Offset, y = frame.Size.Y.Offset}
                    saveConfig()
                end
            end
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and not UI.Locked and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
            local d = i.Position - dragStart
            local scale = getGlobalScale()
            local nx = math.max(minSize.X.Offset, startSize.X + (d.X / scale))
            local ny = math.max(minSize.Y.Offset, startSize.Y + (d.Y / scale))
            frame.Size = UDim2.new(0, nx, 0, ny)
        end
    end)
end

function makeHeader(f,t,isMain) local h=Instance.new("Frame"); h.Size=UDim2.new(1,0,0,42); h.BackgroundTransparency=1; h.Parent=f
    local parts={}; for s in string.gmatch(t,"([^\n]+)") do table.insert(parts,s) end
    if isMain then local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-50,0,24); l.Position=UDim2.new(0,13,0,8); l.BackgroundTransparency=1; l.Text=parts[1] or "ALPHA HUB PRIVAT"; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamBlack; l.TextSize=16; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=h
    else local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-58,0,16); l.Position=UDim2.new(0,12,0,7); l.BackgroundTransparency=1; l.Text=parts[1] or "ALPHA HUB PRIVAT"; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamBlack; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Center; l.Parent=h
        local s=Instance.new("TextLabel"); s.Size=UDim2.new(1,-58,0,13); s.Position=UDim2.new(0,12,0,21); s.BackgroundTransparency=1; s.Text=parts[2] or ""; s.TextColor3=Theme.Dim; s.Font=Enum.Font.GothamMedium; s.TextSize=10; s.TextXAlignment=Enum.TextXAlignment.Center; s.Parent=h end
    local d=Instance.new("Frame"); d.Size=UDim2.new(1,-24,0,1); d.Position=UDim2.new(0,12,0,40); d.BackgroundColor3=Theme.AccentLight; d.BackgroundTransparency=isMain and 0.25 or 0.04; d.BorderSizePixel=0; d.Parent=f
    makeDraggable(f,h,t); return h end

function makeMainPanel(t,size,pos) local f=Instance.new("Frame"); f.Size=size; f.Position=pos; f.BackgroundColor3=Theme.MainBackground; f.BackgroundTransparency=0.06; f.BorderSizePixel=0; f.ClipsDescendants=true; f.Parent=gui; corner(f,12); addOutline(f); makeHeader(f,t,true)
    local body=Instance.new("ScrollingFrame"); body.Size=UDim2.new(1,-12,1,-82); body.Position=UDim2.new(0,6,0,76); body.BackgroundTransparency=1; body.BorderSizePixel=0; body.ScrollBarThickness=3; body.ScrollBarImageColor3=Theme.Accent; body.CanvasSize=UDim2.new(0,0,0,0); body.Active=true; body.Parent=f
    local lay=Instance.new("UIListLayout"); lay.Padding=UDim.new(0,6); lay.Parent=body
    lay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() body.CanvasSize=UDim2.new(0,0,0,lay.AbsoluteContentSize.Y+10) end); 
    if Config.sizes and Config.sizes[t] then f.Size = UDim2.new(0, Config.sizes[t].x, 0, Config.sizes[t].y) end
    makeResizable(f, UDim2.new(0, 200, 0, 150), t); return f,body end

function makeQuickPanel(t,size,pos) local f=Instance.new("Frame"); f.Size=size; f.Position=pos; f.BackgroundColor3=Theme.Background; f.BackgroundTransparency=0.04; f.BorderSizePixel=0; f.ClipsDescendants=true; f.Parent=gui; corner(f,12); addOutline(f); makeHeader(f,t,false)
    local body=Instance.new("ScrollingFrame"); body.Size=UDim2.new(1,-12,1,-50); body.Position=UDim2.new(0,6,0,46); body.BackgroundTransparency=1; body.BorderSizePixel=0; body.ScrollBarThickness=3; body.ScrollBarImageColor3=Theme.Accent; body.CanvasSize=UDim2.new(0,0,0,0); body.Active=true; body.Parent=f
    local lay=Instance.new("UIListLayout"); lay.Padding=UDim.new(0,6); lay.Parent=body
    lay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() body.CanvasSize=UDim2.new(0,0,0,lay.AbsoluteContentSize.Y+10) end); 
    if Config.sizes and Config.sizes[t] then f.Size = UDim2.new(0, Config.sizes[t].x, 0, Config.sizes[t].y) end
    if not string.find(t, "Admin Command Panel") then makeResizable(f, UDim2.new(0, 150, 0, 150), t) end; return f,body end

function makeSyncStateRow(parent,text,toggleName,callback)
    regToggle(toggleName,getToggle(toggleName))
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,34); row.BackgroundTransparency=1; row.Parent=parent
    local label=Instance.new("TextLabel"); label.Size=UDim2.new(1,-84,1,0); label.Position=UDim2.new(0,4,0,0); label.BackgroundTransparency=1; label.Text=text; label.TextColor3=Theme.Text; label.Font=Enum.Font.GothamSemibold; label.TextSize=12; label.TextXAlignment=Enum.TextXAlignment.Left; label.TextTruncate=Enum.TextTruncate.AtEnd; label.Parent=row
    local btn=Instance.new("TextButton"); btn.Name="WhiteTextBtn"; btn.Size=UDim2.new(0,72,0,30); btn.Position=UDim2.new(1,-74,0.5,-15); btn.TextColor3=Color3.new(1,1,1); btn.Font=Enum.Font.GothamBlack; btn.TextSize=12; btn.AutoButtonColor=false; btn.Parent=row; corner(btn,6)
    local function refresh(val) btn.BackgroundColor3=val and Theme.Green or Theme.ToggleOff2; btn.Text=val and "ON" or "OFF" end
    refresh(getToggle(toggleName)); onToggleChanged(toggleName,function(val) refresh(val) end)
    btn.MouseButton1Click:Connect(function() local nv=not getToggle(toggleName); setToggle(toggleName,nv); if callback then callback(nv) end end)
    return function(ns,fire) if typeof(ns)=="boolean" then setToggle(toggleName,ns); if fire~=false and callback then callback(ns) end end end, label
end

function makeSyncMainToggle(parent,text,toggleName,callback)
    regToggle(toggleName,getToggle(toggleName))
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.Panel; row.BackgroundTransparency=0.18; row.Parent=parent; corner(row,6)
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-54,1,0); l.Position=UDim2.new(0,8,0,0); l.BackgroundTransparency=1; l.Text=text; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row
    local toggle=Instance.new("TextButton"); toggle.Size=UDim2.new(0,38,0,21); toggle.Position=UDim2.new(1,-44,0.5,-10.5); toggle.Text=""; toggle.AutoButtonColor=false; toggle.Parent=row; corner(toggle,20)
    local dot=Instance.new("Frame"); dot.Size=UDim2.new(0,16,0,16); dot.BackgroundColor3=Theme.InputBg; dot.Parent=toggle; corner(dot,20)
    local function refresh(val) tw(toggle,{BackgroundColor3=val and Theme.Green or Theme.ToggleOff},0.12); tw(dot,{Position=val and UDim2.new(1,-19,0.5,-8) or UDim2.new(0,3,0.5,-8)},0.12) end
    refresh(getToggle(toggleName)); dot.Position=getToggle(toggleName) and UDim2.new(1,-19,0.5,-8) or UDim2.new(0,3,0.5,-8)
    toggle.BackgroundColor3=getToggle(toggleName) and Theme.Green or Theme.ToggleOff
    onToggleChanged(toggleName,function(val) refresh(val) end)
    toggle.MouseButton1Click:Connect(function() local nv=not getToggle(toggleName); setToggle(toggleName,nv); if callback then callback(nv) end end)
    BoundToggles[text]=function(ns,fire) if typeof(ns)=="boolean" then setToggle(toggleName,ns); if fire~=false and callback then callback(ns) end end end
    return BoundToggles[text]
end

function makeQuickButton(parent,text,callback,bg) local b=Instance.new("TextButton"); b.Size=UDim2.new(1,-4,0,36); b.BackgroundColor3=bg or Theme.SoftButton; b.BackgroundTransparency=0.02; b.Text=text; b.TextColor3=Theme.Text; b.Font=Enum.Font.GothamBold; b.TextSize=13; b.AutoButtonColor=false; b.Parent=parent; corner(b,6)
    b.MouseEnter:Connect(function() tw(b,{BackgroundColor3=bg or Theme.SoftButtonHover},0.12) end); b.MouseLeave:Connect(function() tw(b,{BackgroundColor3=bg or Theme.SoftButton},0.12) end)
    b.MouseButton1Click:Connect(function() if callback then callback() end end); return b end

function makeQuickSlider(parent,text,min,max,default,callback,suffix) local holder=Instance.new("Frame"); holder.Size=UDim2.new(1,-4,0,50); holder.BackgroundTransparency=1; holder.Parent=parent
    local label=Instance.new("TextLabel"); label.Size=UDim2.new(1,0,0,16); label.Position=UDim2.new(0,4,0,0); label.BackgroundTransparency=1; label.Text=text..": "..tostring(default)..(suffix or ""); label.TextColor3=Theme.Text; label.Font=Enum.Font.GothamMedium; label.TextSize=10; label.TextXAlignment=Enum.TextXAlignment.Left; label.Parent=holder
    local bar=Instance.new("Frame"); bar.Size=UDim2.new(1,-10,0,6); bar.Position=UDim2.new(0,4,0,26); bar.BackgroundColor3=Theme.SliderBg; bar.BorderSizePixel=0; bar.Parent=holder; corner(bar,10)
    local fill=Instance.new("Frame"); fill.Size=UDim2.new(math.clamp((default-min)/(max-min),0,1),0,1,0); fill.BackgroundColor3=Theme.Accent; fill.BorderSizePixel=0; fill.Parent=bar; corner(fill,10)
    local knob=Instance.new("Frame"); knob.Size=UDim2.new(0,14,0,14); knob.AnchorPoint=Vector2.new(0.5,0.5); knob.Position=UDim2.new(math.clamp((default-min)/(max-min),0,1),0,0.5,0); knob.Name = "WhiteSliderKnob"; knob.BackgroundColor3=Color3.fromRGB(255, 255, 255); knob.BorderSizePixel=0; knob.Parent=bar; corner(knob,20)
    local dragging=false
    local function update(x) local rel=math.clamp((x-bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1); local v=math.floor((min+(max-min)*rel)*10+0.5)/10; fill.Size=UDim2.new(rel,0,1,0); knob.Position=UDim2.new(rel,0,0.5,0); label.Text=text..": "..tostring(v)..(suffix or ""); if callback then callback(v) end end
    bar.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=true; update(i.Position.X) end end)
    UIS.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=false end end)
    UIS.InputChanged:Connect(function(i) if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then update(i.Position.X) end end)
    local function setVal(v, silent)
        v = math.clamp(v, min, max)
        local rel = (v - min) / (max - min)
        local displayVal = math.floor(v * 10 + 0.5) / 10
        fill.Size = UDim2.new(rel, 0, 1, 0)
        knob.Position = UDim2.new(rel, 0, 0.5, 0)
        label.Text = text..": "..tostring(displayVal)..(suffix or "")
        if callback and not silent then callback(displayVal) end
    end
    return {Set = setVal}
end

function makeMainSliderWithInput(parent,text,min,max,default,callback,suffix)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1,-4,0,48)
    row.BackgroundColor3 = Theme.Panel
    row.BackgroundTransparency = 0.18
    row.Parent = parent
    corner(row, 6)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.65,0,0,16)
    label.Position = UDim2.new(0,8,0,4)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Theme.Text
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 10
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = row

    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0.26,0,0,16)
    box.Position = UDim2.new(0.74,-8,0,4)
    box.BackgroundColor3 = Theme.InputBg
    box.BorderSizePixel = 0
    box.Text = tostring(default)..(suffix or "")
    box.TextColor3 = Theme.Text
    box.Font = Enum.Font.GothamBold
    box.TextSize = 9
    box.ClearTextOnFocus = false
    box.Parent = row
    corner(box,4)
    stroke(box, Theme.Stroke, 1, 0.4)

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1,-16,0,6)
    bar.Position = UDim2.new(0,8,0,28)
    bar.BackgroundColor3 = Theme.SliderBg
    bar.BorderSizePixel = 0
    bar.Parent = row
    corner(bar,10)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(math.clamp((default-min)/(max-min),0,1),0,1,0)
    fill.BackgroundColor3 = Theme.Accent
    fill.BorderSizePixel = 0
    fill.Parent = bar
    corner(fill,10)

    local knob = Instance.new("Frame")
    knob.Name = "WhiteSliderKnob"
    knob.Size = UDim2.new(0,14,0,14)
    knob.AnchorPoint = Vector2.new(0.5,0.5)
    knob.Position = UDim2.new(math.clamp((default-min)/(max-min),0,1),0,0.5,0)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.Parent = bar
    corner(knob,20)

    local dragging = false
    local function updateValue(val, skipBox)
        val = math.clamp(val, min, max)
        val = math.floor(val * 100 + 0.5) / 100
        local rel = (val - min) / (max - min)
        fill.Size = UDim2.new(rel,0,1,0)
        knob.Position = UDim2.new(rel,0,0.5,0)
        if not skipBox then
            box.Text = tostring(val)..(suffix or "")
        end
        if callback then callback(val) end
    end

    bar.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            local rel = math.clamp((i.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
            updateValue(min + (max - min) * rel)
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
            local rel = math.clamp((i.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
            updateValue(min + (max - min) * rel)
        end
    end)

    box.FocusLost:Connect(function()
        local raw = box.Text:gsub("[^%d%.]", "")
        local num = tonumber(raw)
        if num then
            updateValue(num)
        else
            box.Text = tostring(default)..(suffix or "")
        end
    end)
    return row
end

function makeMainButton(parent,text,callback,color) local b=Instance.new("TextButton"); b.Size=UDim2.new(1,-4,0,30); b.BackgroundColor3=color or Theme.Row; b.BackgroundTransparency=0.16; b.Text=text; b.TextColor3=Theme.Text; b.Font=Enum.Font.GothamBold; b.TextSize=11; b.AutoButtonColor=false; b.Parent=parent; corner(b,6); stroke(b,Theme.AccentLight,1,0.28)
    b.MouseEnter:Connect(function() tw(b,{BackgroundColor3=color or Theme.RowHover},0.12) end); b.MouseLeave:Connect(function() tw(b,{BackgroundColor3=color or Theme.Row},0.12) end)
    b.MouseButton1Click:Connect(function() if callback then callback() end end); return b end

function makeMainToggle(parent,text,enabled,callback)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.Panel; row.BackgroundTransparency=0.18; row.Parent=parent; corner(row,6)
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-54,1,0); l.Position=UDim2.new(0,8,0,0); l.BackgroundTransparency=1; l.Text=text; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row
    local toggle=Instance.new("TextButton"); toggle.Size=UDim2.new(0,38,0,21); toggle.Position=UDim2.new(1,-44,0.5,-10.5); toggle.BackgroundColor3=enabled and Theme.Green or Theme.ToggleOff; toggle.Text=""; toggle.AutoButtonColor=false; toggle.Parent=row; corner(toggle,20)
    local dot=Instance.new("Frame"); dot.Size=UDim2.new(0,16,0,16); dot.Position=enabled and UDim2.new(1,-19,0.5,-8) or UDim2.new(0,3,0.5,-8); dot.BackgroundColor3=Theme.InputBg; dot.Parent=toggle; corner(dot,20)
    local state=enabled
    local function setState(ns,fire) state=ns; tw(toggle,{BackgroundColor3=state and Theme.Green or Theme.ToggleOff},0.12); tw(dot,{Position=state and UDim2.new(1,-19,0.5,-8) or UDim2.new(0,3,0.5,-8)},0.12); if fire~=false and callback then callback(state) end end
    toggle.MouseButton1Click:Connect(function() setState(not state,true) end)
    BoundToggles[text]=function(ns,fire) if typeof(ns)=="boolean" then setState(ns,fire) else setState(not state,true) end end; return BoundToggles[text]
end

function makeMainTextBox(parent,text,default,placeholder,callback)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.Panel; row.BackgroundTransparency=0.18; row.Parent=parent; corner(row,6)
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-100,1,0); l.Position=UDim2.new(0,8,0,0); l.BackgroundTransparency=1; l.Text=text; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row
    local box=Instance.new("TextBox"); box.Size=UDim2.new(0,80,0,21); box.Position=UDim2.new(1,-88,0.5,-10.5); box.BackgroundColor3=Theme.InputBg; box.BorderSizePixel=0; box.Text=default or ""; box.PlaceholderText=placeholder or ""; box.TextColor3=Theme.Text; box.Font=Enum.Font.GothamMedium; box.TextSize=10; box.ClearTextOnFocus=false; box.Parent=row; corner(box,4)
    box.FocusLost:Connect(function()
        local raw = box.Text:gsub("%s", "")
        if callback then callback(raw) end
    end)
    return box
end

function makeKeybindRow(parent,nameText)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.Panel; row.BackgroundTransparency=0.18; row.Parent=parent; corner(row,6)
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-88,1,0); l.Position=UDim2.new(0,8,0,0); l.BackgroundTransparency=1; l.Text=nameText; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamSemibold; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=row
    local x=Instance.new("TextButton"); x.Name="WhiteTextBtn"; x.Size=UDim2.new(0,22,0,20); x.Position=UDim2.new(1,-74,0.5,-10); x.BackgroundColor3=Theme.Red; x.Text="X"; x.TextColor3=Color3.new(1,1,1); x.Font=Enum.Font.GothamBold; x.TextSize=10; x.Parent=row; corner(x,5)
    local key=Instance.new("TextButton"); key.Name="WhiteTextBtn"; key.Size=UDim2.new(0,50,0,20); key.Position=UDim2.new(1,-50,0.5,-10); key.BackgroundColor3=Theme.Accent; key.Text=Keybinds[nameText] or "NONE"; key.TextColor3=Color3.new(1,1,1); key.Font=Enum.Font.GothamBold; key.TextSize=9; key.Parent=row; corner(key,5)
    x.MouseButton1Click:Connect(function() Keybinds[nameText]="NONE"; Config.keybinds[nameText]="NONE"; saveConfig(); key.Text="NONE"; if updateMovementPanelLabels then updateMovementPanelLabels() end end)
    key.MouseButton1Click:Connect(function() key.Text="..."
        local con; con=UIS.InputBegan:Connect(function(input,gp) if gp then return end; if input.UserInputType==Enum.UserInputType.Keyboard then Keybinds[nameText]=input.KeyCode.Name; Config.keybinds[nameText]=input.KeyCode.Name; saveConfig(); key.Text=input.KeyCode.Name; if nameText=="Open Menu" then UI.OpenMenuKey=input.KeyCode end; con:Disconnect(); if updateMovementPanelLabels then updateMovementPanelLabels() end end end)
    end)
end

-- CREATE PANELS
main,mainBody=makeMainPanel("ALPHA HUB PRIVAT",UDim2.new(0,375,0,480),UDim2.new(0.5,-187,0.5,-255))
if Config.AutoCloseOnExec then main.Visible = false end
panels["Invisible Steal Panel"],panels["InvisStealBody"]=makeQuickPanel("ALPHA HUB PRIVAT\nInvisible Steal",UDim2.new(0,230,0,375),UDim2.new(0,80,0.5,-220))
panels["InvisStealBody"].ScrollBarThickness = 0
panels["InvisStealBody"].ScrollingEnabled = false
panels["Admin Command Panel"],panels["AdminBody"]=makeQuickPanel("ALPHA HUB PRIVAT\nAdmin Command Panel",UDim2.new(0,225,0,240),UDim2.new(0.5,85,1,-340))
panels["Command Cooldowns"],panels["CooldownBody"]=makeQuickPanel("ALPHA HUB PRIVAT\nCommand Cooldowns",UDim2.new(0,210,0,315),UDim2.new(0.5,245,1,-390))
panels["Actions"],panels["ActionsBody"]=makeQuickPanel("ALPHA HUB PRIVAT\nActions",UDim2.new(0,230,0,340),UDim2.new(0.5,505,1,-415))
panels["Steal Panel"],panels["StealBody"]=makeQuickPanel("ALPHA HUB PRIVAT\nSteal Panel",UDim2.new(0,235,0,300),UDim2.new(1,-300,1,-385))
panels["Steal Target"],panels["TargetBody"]=makeQuickPanel("ALPHA HUB PRIVAT\nSteal Target",UDim2.new(0,320,0,380),UDim2.new(1,-330,0,85))
actionSettingsPanel,actionSettingsBody=makeQuickPanel("ALPHA HUB PRIVAT\nAction Settings",UDim2.new(0,230,0,370),UDim2.new(0.5,745,1,-440))
actionSettingsPanel.Visible=false
tpSpeedSettingsPanel,tpSpeedSettingsBody=makeQuickPanel("ALPHA HUB PRIVAT\nTP & Clone Settings",UDim2.new(0,235,0,325),UDim2.new(0.5,745,1,-440))
tpSpeedSettingsPanel.Visible=false
for _,pair in ipairs({{"ALPHA HUB PRIVAT",main},{"ALPHA HUB PRIVAT\nInvisible Steal",panels["Invisible Steal Panel"]},
    {"ALPHA HUB PRIVAT\nAdmin Command Panel",panels["Admin Command Panel"]},{"ALPHA HUB PRIVAT\nCommand Cooldowns",panels["Command Cooldowns"]},
    {"ALPHA HUB PRIVAT\nActions",panels["Actions"]},{"ALPHA HUB PRIVAT\nSteal Panel",panels["Steal Panel"]},{"ALPHA HUB PRIVAT\nSteal Target",panels["Steal Target"]},
    {"ALPHA HUB PRIVAT\nAction Settings",actionSettingsPanel},{"ALPHA HUB PRIVAT\nTP & Clone Settings",tpSpeedSettingsPanel}}) do applySavedPosition(pair[1],pair[2]) end

-- LAZY UI LOADING
if _G.addLazyUI then
    _G.addLazyUI(main, not Config.AutoCloseOnExec)
    _G.addLazyUI(actionSettingsPanel, false)
    _G.addLazyUI(tpSpeedSettingsPanel, false)
end

local immediatePanels = {
    ["Steal Target"] = true,
    ["Steal Panel"] = true,
    ["Invisible Steal Panel"] = true
}

for name, panel in pairs(panels) do
    if not string.match(name, "Body$") then
        -- _G._SXEPanelVis survives re-executes; override Config if it has a value
        local fromG = _G._SXEPanelVis[name]
        local targetVis
        if fromG ~= nil then
            targetVis = fromG
            Config.Visibilities[name] = fromG  -- sync back so Config is consistent
        else
            if Config.Visibilities[name] ~= nil then targetVis = Config.Visibilities[name] else targetVis = true end
        end
        if immediatePanels[name] then
            panel.Visible = targetVis
        elseif _G.addLazyUI then
            _G.addLazyUI(panel, targetVis, false, name)
        end
    end
end

-- ACTIONS PANEL 
function rebuildActions()
    clearBody(panels["ActionsBody"])
    if actionConfig["Ragdoll Self (R)"] then makeQuickButton(panels["ActionsBody"],"Ragdoll Self (R)",function() pcall(runAdminCommand,player,"ragdoll") end) end
    if actionConfig["Rejoin PS"] then makeQuickButton(panels["ActionsBody"],"Rejoin PS",function() TeleportService:Teleport(game.PlaceId,player) end) end
    if actionConfig["Rejoin Job ID (J)"] then makeQuickButton(panels["ActionsBody"],"Rejoin Job ID (J)",function()
        pcall(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, player) end)
    end) end
    if actionConfig["Kick (Y)"] then makeQuickButton(panels["ActionsBody"],"Kick (Y)",function() kickPlayer() end) end
    if actionConfig["Kick To Private"] then makeQuickButton(panels["ActionsBody"],"Kick To Private",function() 
        if PrivateServerCode and PrivateServerCode ~= "" then
            task.delay(0.2, function()
                pcall(function() game:GetService("ExperienceService"):LaunchExperience({placeId=game.PlaceId,linkCode=PrivateServerCode}) end)
            end)
        end
    end) end
    if actionConfig["Reset (X)"] then makeQuickButton(panels["ActionsBody"],"Reset (X)",function() executeReset() end,Theme.SoftAccentHover) end
    if actionConfig["Anti Ragdoll"] then makeSyncStateRow(panels["ActionsBody"],"Anti Ragdoll:","Anti Ragdoll",function(on) if on then startAntiRagdoll() else stopAntiRagdoll() end end) end
    if actionConfig["Infinite Jump"] then makeSyncStateRow(panels["ActionsBody"],"Infinite Jump:","Infinite Jump",function(on) setInfiniteJump(on) end) end
    if actionConfig["Float"] then makeSyncStateRow(panels["ActionsBody"],"Float:","Float",function(on) setFloat(on) end) end
    if actionConfig["Carpet Speed"] then makeSyncStateRow(panels["ActionsBody"],"Carpet Speed:","Carpet Speed",function(on) setCarpetSpeed(on) end) end
    makeQuickButton(panels["ActionsBody"],"Settings",function()
        if actionSettingsPanel.Visible then closeAnim(actionSettingsPanel) else openAnim(actionSettingsPanel) end
    end,Theme.SoftAccent)
end

function rebuildActionSettings()
    clearBody(actionSettingsBody)
    for actionName,enabled in pairs(actionConfig) do
        local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,34); row.BackgroundTransparency=1; row.Parent=actionSettingsBody
        local label=Instance.new("TextLabel"); label.Size=UDim2.new(1,-84,1,0); label.Position=UDim2.new(0,4,0,0); label.BackgroundTransparency=1; label.Text=actionName; label.TextColor3=Theme.Text; label.Font=Enum.Font.GothamSemibold; label.TextSize=11; label.TextXAlignment=Enum.TextXAlignment.Left; label.TextTruncate=Enum.TextTruncate.AtEnd; label.Parent=row
        local btn=Instance.new("TextButton"); btn.Name="WhiteTextBtn"; btn.Size=UDim2.new(0,72,0,30); btn.Position=UDim2.new(1,-74,0.5,-15); btn.BackgroundColor3=enabled and Theme.Green or Theme.ToggleOff2; btn.Text=enabled and "ON" or "OFF"; btn.TextColor3=Color3.new(1,1,1); btn.Font=Enum.Font.GothamBlack; btn.TextSize=12; btn.AutoButtonColor=false; btn.Parent=row; corner(btn,6)
        btn.MouseButton1Click:Connect(function()
            actionConfig[actionName]=not actionConfig[actionName]; Config.actions[actionName]=actionConfig[actionName]; saveConfig()
            btn.BackgroundColor3=actionConfig[actionName] and Theme.Green or Theme.ToggleOff2; btn.Text=actionConfig[actionName] and "ON" or "OFF"
            rebuildActions()
        end)
    end
    makeQuickButton(actionSettingsBody,"Close",function() closeAnim(actionSettingsPanel) end,Theme.SoftAccentHover)
end

function rebuildTpSpeedSettings()
    clearBody(tpSpeedSettingsBody)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Fly TP Speed", 50, 300, Config.TpSettings.FlyTPSpeed or 160, function(v) Config.TpSettings.FlyTPSpeed=v; saveConfig() end)
    makeMainSliderWithInput(tpSpeedSettingsBody, "100 Studs Base Speed", 20, 250, Config.TpSettings.FlyTPCloseSpeed or 75, function(v) Config.TpSettings.FlyTPCloseSpeed=v; saveConfig() end)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Grabble TP Speed", 50, 600, Config.TpSettings.GrabbleTPSpeed or 230, function(v) Config.TpSettings.GrabbleTPSpeed=v; saveConfig(); if _G.SXESetCarpetSpeed then pcall(_G.SXESetCarpetSpeed, v) end end)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Walk To Brainrot Speed", 50, 300, Config.TpSettings.WalkTPSpeed or 190, function(v) Config.TpSettings.WalkTPSpeed=v; saveConfig() end)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Clone Delay", 0.05, 2.0, Config.TpSettings.CloneDelayVal or 0.1, function(v) Config.TpSettings.CloneDelayVal=v; saveConfig() end, "s")
    makeQuickButton(tpSpeedSettingsBody, "Close", function() closeAnim(tpSpeedSettingsPanel) end, Theme.SoftAccentHover)
end

do
    -- INVISIBLE STEAL PANEL POPULATION
    regToggle("Auto Recover Lagback", Config.AutoRecoverLagback)
    regToggle("Auto Steal Speed", Config.AutoStealSpeed)
    regToggle("Auto Invis During Steal", Config.AutoInvisDuringSteal)

    local enabledRow = makeSyncStateRow(panels["InvisStealBody"],"Enabled:","Invisible Steal",function(on) if _G.toggleInvisibleSteal then pcall(_G.toggleInvisibleSteal) end end)

    _G.updateMovementPanelLabels = function() end
    local rotSlider = makeQuickSlider(panels["InvisStealBody"],"Rotation",0,360,Config.InvisStealAngle or 225,function(v) _G.InvisStealAngle=v; Config.InvisStealAngle=v; saveConfig() end)
    local depthSlider = makeQuickSlider(panels["InvisStealBody"],"Depth",0,18,Config.SinkSliderValue or 7,function(v) _G.SinkSliderValue=v; Config.SinkSliderValue=v; saveConfig() end)
    local recoverToggle = makeSyncStateRow(panels["InvisStealBody"],"Auto Recover:","Auto Recover Lagback",function(on) _G.AutoRecoverLagback=on; Config.AutoRecoverLagback=on; saveConfig() end)
    local autoInvisToggle = makeSyncStateRow(panels["InvisStealBody"],"Auto Invis:","Auto Invis During Steal",function(on) _G.AutoInvisDuringSteal=on; Config.AutoInvisDuringSteal=on; saveConfig() end)

    -- WALKSPEED CONTROL (CFrame Bypass, min 15 max 29)
    regToggle("WalkSpeed", Config.WalkSpeedEnabled)
    makeSyncStateRow(panels["InvisStealBody"],"WalkSpeed:","WalkSpeed",function(on) setWalkSpeedEnabled(on) end)
    makeMainSliderWithInput(panels["InvisStealBody"], "Walk Speed", 15, 29, Config.WalkSpeedValue or 16, function(v)
        local clamped = setWalkSpeedValue(v)
    end)
    if Config.WalkSpeedEnabled then
        task.defer(function() setWalkSpeedEnabled(true) end)
    end


-- ADMIN QUICK PANEL
local function spamPlayerBaseOwner(targetPlayer)
    if not targetPlayer then return 0 end
    local cmdList = Config.SpamBaseOwnerOrder or AP_ALL_COMMANDS
    
    if Config.SpamBaseOwnerSingleCommand then
        local startIndex = _G.SpamBaseOwnerIndex or 1
        if startIndex > #cmdList then startIndex = 1 end
        
        local picked = nil
        local attempts = 0
        while attempts < #cmdList do
            local idx = ((startIndex - 1 + attempts) % #cmdList) + 1
            local cmd = cmdList[idx]
            if Config.SpamBaseOwnerCommands and Config.SpamBaseOwnerCommands[cmd] and not apIsOnCooldown(cmd) then
                picked = cmd
                _G.SpamBaseOwnerIndex = idx + 1
                break
            end
            attempts = attempts + 1
        end
        
        if picked then
            runAdminCommand(targetPlayer, picked)
            return 1
        end
        return 0
    else
        local activeCmds = {}
        for _, cmd in ipairs(cmdList) do
            if Config.SpamBaseOwnerCommands and Config.SpamBaseOwnerCommands[cmd] and not apIsOnCooldown(cmd) then
                table.insert(activeCmds, cmd)
            end
        end
        for i, cmd in ipairs(activeCmds) do
            task.spawn(function()
                task.wait((i - 1) * 0.01)
                runAdminCommand(targetPlayer, cmd)
            end)
        end
        return #activeCmds
    end
end

-- ADMIN QUICK PANEL
function spamBaseOwner()
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then
        ShowNotification("SPAM OWNER", "No character found")
        return
    end
    local nearestPlot = nil
    local nearestDist = math.huge
    local Plots = Workspace:FindFirstChild("Plots")
    if Plots then
        for _, plot in ipairs(Plots:GetChildren()) do
            local sign = plot:FindFirstChild("PlotSign")
            if sign then
                local yourBase = sign:FindFirstChild("YourBase")
                if not yourBase or not yourBase.Enabled then
                    local signPos = (sign:IsA("BasePart") and sign.Position)
                        or (sign.PrimaryPart and sign.PrimaryPart.Position)
                    if not signPos then
                        local part = sign:FindFirstChildWhichIsA("BasePart", true)
                        signPos = part and part.Position
                    end
                    if signPos then
                        local dist = (hrp.Position - signPos).Magnitude
                        if dist < nearestDist then
                            nearestDist = dist
                            nearestPlot = plot
                        end
                    end
                end
            end
        end
    end
    if not nearestPlot then
        ShowNotification("SPAM OWNER", "No nearby base found")
        return
    end
    local targetPlayer = nil
    local ok, Synchronizer = pcall(require, ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Synchronizer"))
    if ok and Synchronizer then
        local ch = Synchronizer:Get(nearestPlot.Name)
        if ch then
            local owner = ch:Get("Owner")
            if owner then
                if (typeof(owner) == "Instance") and owner:IsA("Player") then
                    targetPlayer = owner
                elseif (type(owner) == "table") and owner.Name then
                    targetPlayer = Players:FindFirstChild(owner.Name)
                end
            end
        end
    end
    if not targetPlayer then
        local sign = nearestPlot:FindFirstChild("PlotSign")
        local textLabel = sign
            and sign:FindFirstChild("SurfaceGui")
            and sign.SurfaceGui:FindFirstChild("Frame")
            and sign.SurfaceGui.Frame:FindFirstChild("TextLabel")
        if textLabel then
            local baseText = textLabel.Text
            local nickname = (baseText and baseText:match("^(.-)'")) or baseText
            if nickname then
                for _, p in ipairs(Players:GetPlayers()) do
                    if (p.DisplayName == nickname) or (p.Name == nickname) then
                        targetPlayer = p
                        break
                    end
                end
            end
        end
    end
    if not targetPlayer or (targetPlayer == LocalPlayer) then
        ShowNotification("SPAM OWNER", "Owner not found or is you")
        return
    end
    if isPlayerBlacklisted(targetPlayer) then
        ShowNotification("SPAM OWNER", targetPlayer.DisplayName .. " is blacklisted")
        return
    end
    ShowNotification("SPAM OWNER", "Spamming " .. targetPlayer.DisplayName)
    local sentCount = spamPlayerBaseOwner(targetPlayer)
    ShowNotification("SPAM OWNER", "Sent " .. tostring(sentCount) .. " commands")
end

makeSyncStateRow(panels["AdminBody"],"Click to AP:","Click to AP",function(on) Config.ClickToAP=on; saveConfig() end)
makeSyncStateRow(panels["AdminBody"],"Proximity:","Proximity",function(on) setProximityAP(on) end)
makeQuickSlider(panels["AdminBody"],"Distance",1,50,Config.ProximityRange or 15,function(v) Config.ProximityRange=v; saveConfig() end," studs")
makeQuickButton(panels["AdminBody"],"Spam Base Owner",spamBaseOwner)

-- COOLDOWN PANEL
LazyInit("Cooldown Panel", function() -- COOLDOWN PANEL SCOPE
local cooldownLabels={}
for _,item in ipairs({"jail","rocket","inverse","ragdoll","jumpscare","tiny","balloon","morph","nightvision"}) do
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,24); row.BackgroundTransparency=1; row.Parent=panels["CooldownBody"]
    local left=Instance.new("TextLabel"); left.Size=UDim2.new(0.58,0,1,0); left.Position=UDim2.new(0,6,0,0); left.BackgroundTransparency=1; left.Text=item:sub(1,1):upper()..item:sub(2); left.TextColor3=Theme.Text; left.Font=Enum.Font.GothamBold; left.TextSize=12; left.TextXAlignment=Enum.TextXAlignment.Left; left.Parent=row
    local right=Instance.new("TextLabel"); right.Size=UDim2.new(0.36,0,1,0); right.Position=UDim2.new(0.62,0,0,0); right.BackgroundTransparency=1; right.Text="READY"; right.TextColor3=Theme.Green; right.Font=Enum.Font.GothamBlack; right.TextSize=11; right.TextXAlignment=Enum.TextXAlignment.Right; right.Parent=row
    cooldownLabels[item]=right
end
task.spawn(function() while true do task.wait(0.5); for cmd,label in pairs(cooldownLabels) do local rem=apGetRemaining(cmd); if rem>0 then label.Text=string.format("%.0fs",rem); label.TextColor3=Theme.Red else label.Text="READY"; label.TextColor3=Theme.Green end end end end)
end) -- END COOLDOWN PANEL SCOPE (LazyInit)

-- STEAL PANEL
makeSyncStateRow(panels["StealBody"],"Auto Steal:","Auto Steal",function(on)
    autoStealEnabled=on; Config.AutoStealEnabled=on; saveConfig()
    -- SXE Clone-TP engine owns the auto-steal loop unconditionally.
    if _G.SXEAutoSteal then pcall(_G.SXEAutoSteal, on) end
end)
makeSyncStateRow(panels["StealBody"],"Steal Highest:","Steal Highest",function(on) if on then setStealMode("Highest") end end)
makeSyncStateRow(panels["StealBody"],"Steal Priority:","Steal Priority",function(on) if on then setStealMode("Priority") end end)
makeSyncStateRow(panels["StealBody"],"Steal Nearest:","Steal Nearest",function(on) if on then setStealMode("Nearest") end end)
makeSyncStateRow(panels["StealBody"],"Auto Buy:","Auto Buy",function(on)
    if toggleAutoBuy then toggleAutoBuy(on)
    else warn("[SXE] Auto Buy not ready yet -- try again in a sec") end
end)
makeSyncStateRow(panels["StealBody"],"Auto Kick:","Auto Kick",function(on) Config.AutoKickOnSteal=on; saveConfig() end)



-- STEAL TARGET (0.4s refresh) -- NOT lazy: panel must populate on load
do -- STEAL TARGET SCOPE
function refreshTargetPanel()
    clearBody(panels["TargetBody"])
    local cache = get_all_pets()
    if not cache or #cache == 0 then
        local l = Instance.new("TextLabel")
        l.Size = UDim2.new(1,-4,0,24)
        l.BackgroundTransparency = 1
        l.Text = "Scanning..."
        l.TextColor3 = Theme.Dim
        l.Font = Enum.Font.GothamSemibold
        l.TextSize = 11
        l.TextXAlignment = Enum.TextXAlignment.Center
        l.Parent = panels["TargetBody"]
        return
    end

    -- Build priority lookup
    local prioSet = {}
    for _, pName in ipairs(priorityList) do prioSet[pName:lower()] = true end

    -- Split into priority and non-priority
    local prioPets, otherPets = {}, {}
    for _, pet in ipairs(cache) do
        local isBrainrot = (pet.mpsValue and pet.mpsValue >= 10000000)
        if isBrainrot or (pet.petName and prioSet[pet.petName:lower()]) then
            table.insert(prioPets, pet)
        else
            table.insert(otherPets, pet)
        end
    end

    -- Sort priority pets by priority list order, keeping Brainrots at the absolute top
    table.sort(prioPets, function(a, b)
        local ai, bi = 999, 999
        for idx, pn in ipairs(priorityList) do
            local pnL = pn:lower()
            if a.petName and a.petName:lower() == pnL then ai = idx end
            if b.petName and b.petName:lower() == pnL then bi = idx end
        end
        
        local aIsBrainrot = (a.mpsValue and a.mpsValue >= 10000000)
        local bIsBrainrot = (b.mpsValue and b.mpsValue >= 10000000)
        
        if aIsBrainrot and not bIsBrainrot then return true end
        if bIsBrainrot and not aIsBrainrot then return false end
        
        if ai == bi then return (a.mpsValue or 0) > (b.mpsValue or 0) end
        return ai < bi
    end)

    local function makeTargetRow(pet, displayIndex, isPriority)
        local isSelected = (selectedTargetUID == pet.uid)
        local isManual = (manuallySelectedUID == pet.uid)
        local row = Instance.new("Frame")
        row.Size = UDim2.new(1,-4,0,44)
        row.BackgroundColor3 = isSelected and Theme.RowHover or Theme.Panel
        row.BackgroundTransparency = isPriority and 0.18 or 0.35
        row.Parent = panels["TargetBody"]
        corner(row, 6)
        if isSelected or isManual then
            local str = Instance.new("UIStroke", row)
            str.Color = isManual and Color3.fromRGB(56, 214, 110) or Theme.Accent
            str.Thickness = isManual and 1.75 or 1.25
        elseif isPriority then
            local str = Instance.new("UIStroke", row)
            str.Color = Color3.fromRGB(255, 90, 120)
            str.Thickness = 0.75
            str.Transparency = 0.3
        end
        
        local rankBox = Instance.new("Frame")
        rankBox.Size = UDim2.new(0, 26, 0, 26)
        rankBox.Position = UDim2.new(0, 8, 0.5, -13)
        rankBox.BackgroundColor3 = Color3.fromRGB(90, 25, 45)
        rankBox.BorderSizePixel = 0
        rankBox.Parent = row
        corner(rankBox, 4)
        
        local n = Instance.new("TextLabel")
        n.Size = UDim2.new(1,0,1,0)
        n.BackgroundTransparency = 1
        n.Text = "#"..displayIndex
        n.TextColor3 = Color3.new(1,1,1)
        n.Font = Enum.Font.GothamBold
        n.TextSize = 12
        n.Parent = rankBox
        
        local nm = Instance.new("TextLabel")
        nm.Size = UDim2.new(1,-50,0,18)
        nm.Position = UDim2.new(0,44,0,5)
        nm.BackgroundTransparency = 1
        nm.Text = pet.petName or "?"
        nm.TextColor3 = isPriority and Color3.new(1,1,1) or Theme.Text
        nm.Font = isPriority and Enum.Font.GothamBold or Enum.Font.GothamSemibold
        nm.TextSize = 13
        nm.TextXAlignment = Enum.TextXAlignment.Left
        nm.TextTruncate = Enum.TextTruncate.AtEnd
        nm.Parent = row
        
        local gn = Instance.new("TextLabel")
        gn.Size = UDim2.new(1,-50,0,14)
        gn.Position = UDim2.new(0,44,0,23)
        gn.BackgroundTransparency = 1
        gn.RichText = true
        local ownerText = pet.owner and (' <font color="#999999">| @' .. tostring(pet.owner) .. '</font>') or ""
        gn.Text = '<font color="#38D66E">Gem: ' .. (pet.mpsText or "") .. '</font>' .. ownerText
        gn.TextColor3 = Color3.new(1,1,1)
        gn.Font = Enum.Font.GothamMedium
        gn.TextSize = 11
        gn.TextXAlignment = Enum.TextXAlignment.Left
        gn.TextTruncate = Enum.TextTruncate.AtEnd
        gn.Parent = row

        local overlay = Instance.new("TextButton")
        overlay.Size = UDim2.new(1,0,1,0)
        overlay.BackgroundTransparency = 1
        overlay.Text = ""
        overlay.ZIndex = 8
        overlay.Parent = row
        overlay.MouseButton1Click:Connect(function()
            if manuallySelectedUID == pet.uid then
                manuallySelectedUID = nil
                selectedTargetUID = nil
                SharedState.SelectedPetData = nil
            else
                selectedTargetUID = pet.uid
                manuallySelectedUID = pet.uid
                SharedState.SelectedPetData = pet
                if pet then SharedState.LastTargetedPetMpsValue = pet.mpsValue or 0 end
            end
            refreshTargetPanel()

            if manuallySelectedUID then
                task.spawn(function()
                    local pr = PromptMemoryCache[pet.uid] or findProximityPromptForAnimal(pet.animalData)
                    if pr then
                        if _G.ALPHA_ExecuteManualSteal then
                            pcall(_G.ALPHA_ExecuteManualSteal, pr)
                        end
                    end
                end)
            end
        end)
    end

    local idx = 0
    for _, pet in ipairs(prioPets) do idx = idx + 1; makeTargetRow(pet, idx, true) end

    if #prioPets > 0 and #otherPets > 0 then
        local sep = Instance.new("Frame")
        sep.Size = UDim2.new(1,-20,0,1)
        sep.BackgroundColor3 = Theme.AccentLight
        sep.BackgroundTransparency = 0.5
        sep.BorderSizePixel = 0
        sep.Parent = panels["TargetBody"]
    end

    for _, pet in ipairs(otherPets) do idx = idx + 1; makeTargetRow(pet, idx, false) end
end
task.spawn(function() while true do task.wait(0.4); refreshTargetPanel() end end)
end -- END STEAL TARGET SCOPE

-- ============================================================
-- ADMIN PANEL UI
-- ============================================================
LazyInit("Admin Panel UI", function() -- ADMIN PANEL UI SCOPE
    pcall(function() local e=playerGui:FindFirstChild("XiAdminPanel"); if e then e:Destroy() end end)
    apGui=Instance.new("ScreenGui"); apGui.Name="XiAdminPanel"; apGui.ResetOnSpawn=false; apGui.IgnoreGuiInset=true; apGui.DisplayOrder=9999998; apGui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling; apGui.Parent=playerGui
    apGui.Enabled = (Config.AdminPanelUI == true)
    apOuter=Instance.new("Frame"); apOuter.Name="Frame"; apOuter.BackgroundTransparency=1; apOuter.BorderSizePixel=0; apOuter.Size=UDim2.fromOffset(480,0); apOuter.AutomaticSize=Enum.AutomaticSize.Y; apOuter.Position=UDim2.new(0.18,0,0.57,0); apOuter.ZIndex=10; apOuter.ClipsDescendants=true; apOuter.Parent=registerScreenGui(apGui)
    apBG=Instance.new("Frame"); apBG.BackgroundColor3=Theme.Background; apBG.BackgroundTransparency=0.50; apBG.BorderSizePixel=0; apBG.Position=UDim2.fromOffset(-3,-2); apBG.Size=UDim2.new(1,6,1,4); apBG.ZIndex=0; apBG.Parent=apOuter; corner(apBG,8)
    apTop=Instance.new("Frame"); apTop.BackgroundTransparency=1; apTop.BorderSizePixel=0; apTop.Size=UDim2.new(1,0,0,16); apTop.Parent=apOuter; corner(apTop,3)

    makeDraggable(apOuter,apTop,"AdminPanel"); applySavedPosition("AdminPanel",apOuter);

    apList=Instance.new("Frame"); apList.BackgroundTransparency=1; apList.BorderSizePixel=0; apList.Position=UDim2.new(0,0,0,20); apList.Size=UDim2.new(1,0,0,0); apList.AutomaticSize=Enum.AutomaticSize.Y; apList.Parent=apOuter; corner(apList,3)
    apPad=Instance.new("UIPadding"); apPad.PaddingTop=UDim.new(0,2); apPad.PaddingBottom=UDim.new(0,2); apPad.PaddingLeft=UDim.new(0,4); apPad.PaddingRight=UDim.new(0,4); apPad.Parent=apList
    Instance.new("UIListLayout",apList).SortOrder=Enum.SortOrder.LayoutOrder; apList:FindFirstChildOfClass("UIListLayout").Padding=UDim.new(0,2)

    apRows,stealLabels={},{}; rc=0
    -- Build RP_BUTTONS dynamically from Config.AdminPanelButtons
    local function buildRPButtons()
        local btns = {}
        local order = Config.AdminPanelOrder or AP_ALL_COMMANDS
        for _, cmd in ipairs(order) do
            if Config.AdminPanelButtons and Config.AdminPanelButtons[cmd] then
                table.insert(btns, {AP_COMMAND_EMOJIS[cmd] or "⚡", cmd})
            end
        end
        if #btns == 0 then btns = {{"🌀","ragdoll"},{"🔒","jail"},{"🚀","rocket"},{"🎈","balloon"}} end
        return btns
    end
    RP_BUTTONS = buildRPButtons()

    _G.refreshAdminPanelRows = function()
        for uid, row in pairs(apRows) do
            if row then row:Destroy() end
        end
        table.clear(apRows)
        table.clear(stealLabels)
        rc = 0
        RP_BUTTONS = buildRPButtons()
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= player then createAPRow(plr) end
        end
    end

    function createAPRow(plr)
        if not plr or plr==player then return end; if apRows[plr.UserId] and apRows[plr.UserId].Parent then return end
        rc=rc+1; local isAlt=(rc%2==0)
        local actW = (#RP_BUTTONS + 1) * 32 -- Buttons + Blacklist + padding
        local rowW = math.max(460, actW + 180) -- Base width 460, expand if buttons take too much space
        local row=Instance.new("Frame"); row.Name="Row_"..plr.UserId; row.BackgroundColor3=isAlt and Theme.Row or Theme.Panel; row.BackgroundTransparency=0.40; row.BorderSizePixel=0; row.Size=UDim2.new(0,rowW,0,54); row.ZIndex=5; row.ClipsDescendants=true; row.Parent=apList; corner(row,3)
        apRows[plr.UserId]=row
        row.MouseEnter:Connect(function() 
            if not isPlayerBlacklisted(plr) then
                row.BackgroundColor3 = Theme.RowHover
            else
                row.BackgroundColor3 = Theme.BlacklistHover
            end
        end)
        row.MouseLeave:Connect(function() 
            row.BackgroundColor3 = isAlt and Theme.Row or Theme.Panel
        end)

        local avatarH=Instance.new("Frame"); avatarH.BackgroundColor3=Theme.InputBg; avatarH.BorderSizePixel=0; avatarH.Size=UDim2.fromOffset(34,34); avatarH.Position=UDim2.fromOffset(8,10); avatarH.Parent=row; corner(avatarH,8)
        local avatar=Instance.new("ImageLabel"); avatar.BackgroundTransparency=1; avatar.Size=UDim2.fromScale(1,1); avatar.ZIndex=10; avatar.Parent=avatarH; corner(avatar,8)
        task.spawn(function() local ok,img=pcall(function() return Players:GetUserThumbnailAsync(plr.UserId,Enum.ThumbnailType.HeadShot,Enum.ThumbnailSize.Size48x48) end); if ok then avatar.Image=img end end)
        local headshotStroke=Instance.new("UIStroke",avatarH); headshotStroke.Color=Theme.Accent; headshotStroke.Thickness=2; headshotStroke.Transparency=0.3

        local txtW=-(actW+60)
        local nameLabel=Instance.new("TextLabel"); nameLabel.Size=UDim2.new(1,txtW,0,20); nameLabel.Position=UDim2.fromOffset(50,6); nameLabel.BackgroundTransparency=1; nameLabel.Text=plr.DisplayName; nameLabel.Font=Enum.Font.GothamBold; nameLabel.TextSize=14; nameLabel.TextColor3=Theme.Text; nameLabel.TextXAlignment=Enum.TextXAlignment.Left; nameLabel.ZIndex=10; nameLabel.Parent=row
        local userL=Instance.new("TextLabel"); userL.BackgroundTransparency=1; userL.Position=UDim2.fromOffset(50,23); userL.Size=UDim2.new(1,txtW,0,16); userL.TextXAlignment=Enum.TextXAlignment.Left; userL.Text="@"..plr.Name; userL.Font=Enum.Font.GothamMedium; userL.TextSize=10; userL.TextColor3=Theme.Dim; userL.ZIndex=10; userL.Parent=row
        local statusL=Instance.new("TextLabel"); statusL.BackgroundTransparency=1; statusL.Position=UDim2.fromOffset(50,37); statusL.Size=UDim2.new(1,txtW,0,14); statusL.TextXAlignment=Enum.TextXAlignment.Left; statusL.Text=""; statusL.Font=Enum.Font.GothamBold; statusL.TextSize=11; statusL.TextColor3=Theme.AccentLight; statusL.ZIndex=10; statusL.Parent=row
        stealLabels[plr.UserId]=statusL

        local actions=Instance.new("Frame"); actions.BackgroundTransparency=1; actions.AnchorPoint=Vector2.new(1,0.5); actions.Position=UDim2.new(1,-8,0.5,0); actions.Size=UDim2.fromOffset(actW+10,38); actions.ZIndex=12; actions.Parent=row
        local al=Instance.new("UIListLayout"); al.FillDirection=Enum.FillDirection.Horizontal; al.SortOrder=Enum.SortOrder.LayoutOrder; al.Padding=UDim.new(0,2); al.Parent=actions

        for i,b in ipairs(RP_BUTTONS) do
            local btn=Instance.new("TextButton"); btn.Size=UDim2.fromOffset(30,30); btn.BackgroundTransparency=0; btn.AutoButtonColor=false; btn.Text=b[1]; btn.TextSize=12; btn.LayoutOrder=i; btn.Parent=actions
            btn.BackgroundColor3=Theme.SoftButton; btn.Font=Enum.Font.GothamBold; btn.ZIndex=13
            Instance.new("UICorner",btn).CornerRadius=UDim.new(0,6)
            btn.MouseEnter:Connect(function() btn.BackgroundColor3=Theme.SoftButtonHover end)
            btn.MouseLeave:Connect(function() btn.BackgroundColor3=Theme.SoftButton end)
            btn.MouseButton1Click:Connect(function()
                if isPlayerBlacklisted(plr) then
                    ShowNotification("BLOCKED", plr.DisplayName .. " is blacklisted")
                    return
                end
                if apIsOnCooldown(b[2]) then return end; pcall(runAdminCommand,plr,b[2])
            end)
        end

        local blacklistBtn = Instance.new("TextButton")
        blacklistBtn.Name = "BlacklistBtn"
        blacklistBtn.Size = UDim2.fromOffset(30, 30)
        blacklistBtn.BackgroundTransparency = 0
        blacklistBtn.AutoButtonColor = false
        blacklistBtn.Text = "✖"
        blacklistBtn.TextSize = 12
        blacklistBtn.LayoutOrder = 5
        blacklistBtn.Parent = actions
        blacklistBtn.Font = Enum.Font.GothamBold
        blacklistBtn.ZIndex = 13
        Instance.new("UICorner", blacklistBtn).CornerRadius = UDim.new(0, 6)

        local function updateBlacklistVisuals()
            local isBlacklisted = isPlayerBlacklisted(plr)
            if isBlacklisted then
                blacklistBtn.BackgroundColor3 = Color3.fromRGB(255, 60, 60)
                blacklistBtn.TextColor3 = Color3.new(1, 1, 1)
                
                row.BackgroundTransparency = 0.85
                avatar.ImageTransparency = 0.75
                nameLabel.TextTransparency = 0.6
                userL.TextTransparency = 0.6
                statusL.TextTransparency = 0.6
                
                for _, child in ipairs(actions:GetChildren()) do
                    if child:IsA("TextButton") and child ~= blacklistBtn then
                        child.BackgroundTransparency = 0.8
                        child.TextTransparency = 0.8
                    end
                end
            else
                blacklistBtn.BackgroundColor3 = Theme.BlacklistLeave
                blacklistBtn.TextColor3 = Color3.fromRGB(255, 60, 60)
                
                row.BackgroundTransparency = 0.40
                avatar.ImageTransparency = 0
                nameLabel.TextTransparency = 0
                userL.TextTransparency = 0
                statusL.TextTransparency = 0
                
                for _, child in ipairs(actions:GetChildren()) do
                    if child:IsA("TextButton") then
                        child.BackgroundTransparency = 0
                        child.TextTransparency = 0
                    end
                end
            end
        end

        blacklistBtn.MouseEnter:Connect(function() 
            if not isPlayerBlacklisted(plr) then
                blacklistBtn.BackgroundColor3 = Theme.BlacklistHover
            end
        end)
        blacklistBtn.MouseLeave:Connect(function() 
            if not isPlayerBlacklisted(plr) then
                blacklistBtn.BackgroundColor3 = Theme.BlacklistLeave
            end
        end)

        blacklistBtn.MouseButton1Click:Connect(function()
            local isBlacklisted = isPlayerBlacklisted(plr)
            _G.apBlacklist[tostring(plr.UserId)] = not isBlacklisted
            Config.apBlacklist = _G.apBlacklist
            saveConfig()
            updateBlacklistVisuals()
            if not isBlacklisted then
                ShowNotification("BLACKLIST", plr.DisplayName .. " blacklisted")
            else
                ShowNotification("BLACKLIST", plr.DisplayName .. " whitelisted")
            end
        end)

        updateBlacklistVisuals()

        -- Click on player row = fire ALL available admin commands at once (like Click to AP)
        row.InputBegan:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
            -- Ignore clicks on the action buttons themselves
            local mousePos = UIS:GetMouseLocation()
            if actions and actions.AbsolutePosition and actions.AbsoluteSize then
                local ax, ay = actions.AbsolutePosition.X, actions.AbsolutePosition.Y
                local aw, ah = actions.AbsoluteSize.X, actions.AbsoluteSize.Y
                if mousePos.X >= ax and mousePos.X <= ax+aw and mousePos.Y >= ay and mousePos.Y <= ay+ah then
                    return
                end
            end
            if isPlayerBlacklisted(plr) then
                ShowNotification("BLOCKED", plr.DisplayName .. " is blacklisted")
                return
            end
            task.spawn(function()
                local activeCmds = {}
                for _,cmd in ipairs(AP_ALL_COMMANDS) do
                    if not apIsOnCooldown(cmd) then
                        table.insert(activeCmds, cmd)
                    end
                end
                if #activeCmds == 0 then
                    ShowNotification("ADMIN PANEL", "All commands on cooldown")
                    return
                end
                -- Fire ALL at once with minimal delay
                for i, cmd in ipairs(activeCmds) do
                    task.spawn(function()
                        task.wait((i - 1) * 0.01)
                        runAdminCommand(plr, cmd)
                    end)
                end
                ShowNotification("ADMIN PANEL", "Fired " .. #activeCmds .. " cmds on " .. plr.DisplayName)
            end)
        end)

        -- Steal / base-owner status. Reads the cached Synchronizer + memoized
        -- base-owner set, so every row shares one scan instead of each rescanning.
        task.spawn(function() while row.Parent do task.wait(0.5)
            if not plr or not plr.Parent then break end
            local st=stealLabels[plr.UserId]; if not st then break end

            local stealOwner, stealPet = getStealingInfo(plr)
            local curBaseOwnerId = _G.__getCurrentBaseOwnerId()

            if stealPet then
                local fromTxt = stealOwner and (" from " .. (stealOwner.DisplayName or stealOwner.Name)) or ""
                st.Text = "● Stealing " .. tostring(stealPet) .. fromTxt
                st.TextColor3 = Color3.fromRGB(255, 90, 90)
            elseif curBaseOwnerId and curBaseOwnerId == plr.UserId then
                st.Text = "● Base Owner"
                st.TextColor3 = Color3.fromRGB(90, 230, 120)
            else
                st.Text = ""
            end

            -- Danger-tool name highlight
            pcall(function() local ht=getHeldTool(plr)
                if ht and DANGER_TOOLS[ht] then nameLabel.TextColor3=Color3.fromRGB(255,60,60) else nameLabel.TextColor3=Theme.Text end
            end)
        end end)
    end

    for _,plr in ipairs(Players:GetPlayers()) do if plr~=player then createAPRow(plr) end end
    Players.PlayerAdded:Connect(function(plr) task.defer(function() createAPRow(plr) end) end)
    Players.PlayerRemoving:Connect(function(plr) local row=apRows[plr.UserId]; if row then row:Destroy(); apRows[plr.UserId]=nil end; stealLabels[plr.UserId]=nil end)
end) -- END ADMIN PANEL UI SCOPE (LazyInit)

-- ============================================================
-- TAB BAR + TABS
-- ============================================================
tabBar=Instance.new("Frame"); tabBar.Size=UDim2.new(1,-12,0,28); tabBar.Position=UDim2.new(0,6,0,43); tabBar.BackgroundTransparency=1; tabBar.Parent=main
local tabs={"Keybinds","Auto TP","ESP","UI","Misc","Priority","Performance"}
for i,name in ipairs(tabs) do local b=Instance.new("TextButton"); b.Size=UDim2.new(0,49,0,27); b.Position=UDim2.new(0,(i-1)*51,0,0); b.BackgroundColor3=Theme.SoftAccent; b.BackgroundTransparency=0.05; b.Text=name; b.TextColor3=Theme.Dim; b.Font=Enum.Font.GothamMedium; b.TextSize=8; b.AutoButtonColor=false; b.Parent=tabBar; corner(b,5); tabButtons[name]=b end

-- Cache all animal names for autocomplete. Source per request:
-- ReplicatedStorage.Animations.Animals (animation instances are named after
-- each brainrot). Fallback: ReplicatedStorage.Models.Animals if Animations
-- isn't present in the place.
local _allAnimalNames = nil
local function getAllAnimalNames()
    if _allAnimalNames and #_allAnimalNames > 0 then return _allAnimalNames end
    _allAnimalNames = {}
    pcall(function()
        local source = ReplicatedStorage:FindFirstChild("Animations")
        source = source and source:FindFirstChild("Animals")
        if not source then
            -- fallback if Animations.Animals isn't present
            local models = ReplicatedStorage:FindFirstChild("Models")
            source = models and models:FindFirstChild("Animals")
        end
        if source then
            local seen = {}
            for _, child in ipairs(source:GetChildren()) do
                if not seen[child.Name] then
                    seen[child.Name] = true
                    table.insert(_allAnimalNames, child.Name)
                end
            end
            table.sort(_allAnimalNames, function(a, b) return a:lower() < b:lower() end)
        end
    end)
    return _allAnimalNames
end
task.spawn(getAllAnimalNames)

-- Drag state for priority reorder
local _priDragState = {active = false, fromIndex = nil, ghostFrame = nil, overlay = nil}

local function cleanupPriDrag()
    if _priDragState.ghostFrame then pcall(function() _priDragState.ghostFrame:Destroy() end); _priDragState.ghostFrame = nil end
    if _priDragState.overlay then pcall(function() _priDragState.overlay:Destroy() end); _priDragState.overlay = nil end
    _priDragState.active = false; _priDragState.fromIndex = nil
end

function makePriorityRow(index)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.Panel; row.BackgroundTransparency=0.18; row.Parent=mainBody; corner(row,6); row.LayoutOrder=index

    -- Number label
    local num=Instance.new("TextLabel"); num.Size=UDim2.new(0,24,1,0); num.Position=UDim2.new(0,4,0,0); num.BackgroundTransparency=1; num.Text=tostring(index).."."; num.TextColor3=Theme.Dim; num.Font=Enum.Font.GothamBold; num.TextSize=10; num.TextXAlignment=Enum.TextXAlignment.Left; num.Parent=row

    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-120,1,0); l.Position=UDim2.new(0,28,0,0); l.BackgroundTransparency=1; l.Text=priorityList[index]; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row

    local up=Instance.new("TextButton"); up.Name="WhiteTextBtn"; up.Size=UDim2.new(0,26,0,22); up.Position=UDim2.new(1,-86,0.5,-11); up.BackgroundColor3=Theme.Accent; up.Text="▲"; up.TextColor3=Color3.new(1,1,1); up.Font=Enum.Font.GothamBold; up.TextSize=10; up.Parent=row; corner(up,5)
    local dn=Instance.new("TextButton"); dn.Name="WhiteTextBtn"; dn.Size=UDim2.new(0,26,0,22); dn.Position=UDim2.new(1,-56,0.5,-11); dn.BackgroundColor3=Theme.Accent; dn.Text="▼"; dn.TextColor3=Color3.new(1,1,1); dn.Font=Enum.Font.GothamBold; dn.TextSize=10; dn.Parent=row; corner(dn,5)
    local del=Instance.new("TextButton"); del.Name="WhiteTextBtn"; del.Size=UDim2.new(0,26,0,22); del.Position=UDim2.new(1,-26,0.5,-11); del.BackgroundColor3=Theme.Red; del.Text="X"; del.TextColor3=Color3.new(1,1,1); del.Font=Enum.Font.GothamBold; del.TextSize=10; del.Parent=row; corner(del,5)
    up.MouseButton1Click:Connect(function() if index>1 then priorityList[index],priorityList[index-1]=priorityList[index-1],priorityList[index]; Config.PriorityList=priorityList; saveConfig(); loadTab("Priority") end end)
    dn.MouseButton1Click:Connect(function() if index<#priorityList then priorityList[index],priorityList[index+1]=priorityList[index+1],priorityList[index]; Config.PriorityList=priorityList; saveConfig(); loadTab("Priority") end end)
    del.MouseButton1Click:Connect(function() local removedName=priorityList[index]; table.remove(priorityList,index); if not Config.RemovedFromPriority then Config.RemovedFromPriority={} end; local alreadyRemoved=false; for _,rn in ipairs(Config.RemovedFromPriority) do if rn==removedName then alreadyRemoved=true; break end end; if not alreadyRemoved then table.insert(Config.RemovedFromPriority,removedName) end; Config.PriorityList=priorityList; saveConfig(); loadTab("Priority") end)

    -- Drag & Drop
    local dragHandle = Instance.new("TextButton"); dragHandle.Size=UDim2.new(0,24,1,0); dragHandle.Position=UDim2.new(0,0,0,0); dragHandle.BackgroundTransparency=1; dragHandle.Text=""; dragHandle.ZIndex=9; dragHandle.Parent=row
    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
        cleanupPriDrag()
        _priDragState.active = true; _priDragState.fromIndex = index

        local ghost = Instance.new("Frame"); ghost.Size = UDim2.new(0, row.AbsoluteSize.X, 0, 31)
        ghost.BackgroundColor3 = Theme.Accent; ghost.BackgroundTransparency = 0.55; ghost.BorderSizePixel = 0; ghost.ZIndex = 100
        ghost.Parent = gui_sg; corner(ghost, 6)
        local gl = Instance.new("TextLabel"); gl.Size = UDim2.new(1,0,1,0); gl.BackgroundTransparency = 1; gl.Text = tostring(index)..". "..priorityList[index]; gl.TextColor3 = Color3.new(1,1,1); gl.Font = Enum.Font.GothamBold; gl.TextSize = 10; gl.Parent = ghost
        _priDragState.ghostFrame = ghost

        local moveConn, endConn
        moveConn = UIS.InputChanged:Connect(function(mi)
            if mi.UserInputType == Enum.UserInputType.MouseMovement or mi.UserInputType == Enum.UserInputType.Touch then
                ghost.Position = UDim2.new(0, mi.Position.X - ghost.AbsoluteSize.X/2, 0, mi.Position.Y - 15)
            end
        end)
        endConn = UIS.InputEnded:Connect(function(ei)
            if ei.UserInputType ~= Enum.UserInputType.MouseButton1 and ei.UserInputType ~= Enum.UserInputType.Touch then return end
            moveConn:Disconnect(); endConn:Disconnect()
            if not _priDragState.active then cleanupPriDrag(); return end

            -- Determine drop target index from Y position
            local dropY = ei.Position.Y
            local targetIndex = #priorityList
            for _, child in ipairs(mainBody:GetChildren()) do
                if child:IsA("Frame") and child.LayoutOrder and child.LayoutOrder >= 1 and child.LayoutOrder <= #priorityList then
                    local absY = child.AbsolutePosition.Y
                    local absH = child.AbsoluteSize.Y
                    if dropY < absY + absH / 2 then
                        targetIndex = child.LayoutOrder
                        break
                    end
                end
            end
            local fromIdx = _priDragState.fromIndex
            cleanupPriDrag()
            if fromIdx and targetIndex and fromIdx ~= targetIndex then
                local item = table.remove(priorityList, fromIdx)
                if targetIndex > fromIdx then targetIndex = targetIndex - 1 end
                targetIndex = math.clamp(targetIndex, 1, #priorityList + 1)
                table.insert(priorityList, targetIndex, item)
                Config.PriorityList = priorityList; saveConfig()
                loadTab("Priority")
            end
        end)
    end)
end

function makePriorityAddRow()
    local holder = Instance.new("Frame"); holder.Size=UDim2.new(1,-4,0,31); holder.BackgroundColor3=Theme.SoftAccent; holder.BackgroundTransparency=0.1; holder.ClipsDescendants=false; holder.Parent=mainBody; holder.ZIndex=20; corner(holder,6); holder.LayoutOrder = -2

    local box=Instance.new("TextBox"); box.Size=UDim2.new(1,-60,1,-6); box.Position=UDim2.new(0,6,0,3); box.BackgroundColor3=Theme.InputBg; box.BorderSizePixel=0; box.Text=""; box.PlaceholderText="Enter pet name..."; box.TextColor3=Theme.Text; box.PlaceholderColor3=Theme.Dim; box.Font=Enum.Font.GothamMedium; box.TextSize=10; box.ClearTextOnFocus=false; box.Parent=holder; box.ZIndex=21; corner(box,4)

    local addBtn=Instance.new("TextButton"); addBtn.Name="WhiteTextBtn"; addBtn.Size=UDim2.new(0,44,0,25); addBtn.Position=UDim2.new(1,-50,0.5,-12.5); addBtn.BackgroundColor3=Theme.Accent; addBtn.Text="ADD"; addBtn.TextColor3=Color3.new(1,1,1); addBtn.Font=Enum.Font.GothamBlack; addBtn.TextSize=10; addBtn.AutoButtonColor=false; addBtn.Parent=holder; addBtn.ZIndex=21; corner(addBtn,5)

    -- Dropdown for autocomplete
    local dropdown = Instance.new("Frame"); dropdown.Name="PriorityDropdown"; dropdown.Size=UDim2.new(1,-60,0,0); dropdown.Position=UDim2.new(0,6,1,2)
    dropdown.BackgroundColor3=Theme.Background; dropdown.BorderSizePixel=0; dropdown.ClipsDescendants=true; dropdown.Visible=false; dropdown.ZIndex=50; dropdown.Parent=holder; corner(dropdown,6)
    local ddStroke = Instance.new("UIStroke"); ddStroke.Color=Theme.AccentLight; ddStroke.Thickness=1; ddStroke.Parent=dropdown
    local ddScroll = Instance.new("ScrollingFrame"); ddScroll.Size=UDim2.new(1,0,1,0); ddScroll.BackgroundTransparency=1; ddScroll.BorderSizePixel=0; ddScroll.ScrollBarThickness=3; ddScroll.ScrollBarImageColor3=Theme.Accent; ddScroll.CanvasSize=UDim2.new(0,0,0,0); ddScroll.Active=true; ddScroll.ZIndex=51; ddScroll.Parent=dropdown
    local ddLayout = Instance.new("UIListLayout"); ddLayout.Padding=UDim.new(0,1); ddLayout.Parent=ddScroll
    ddLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() ddScroll.CanvasSize=UDim2.new(0,0,0,ddLayout.AbsoluteContentSize.Y) end)

    local function addName(name)
        local trimmed = name:match("^%s*(.-)%s*$")
        if not trimmed or trimmed == "" then return end
        local exists = false
        for _, pName in ipairs(priorityList) do if pName == trimmed then exists = true; break end end
        if not exists then
            table.insert(priorityList, trimmed)
            if Config.RemovedFromPriority then
                for i=#Config.RemovedFromPriority,1,-1 do
                    if Config.RemovedFromPriority[i] == trimmed then table.remove(Config.RemovedFromPriority, i) end
                end
            end
            Config.PriorityList=priorityList; saveConfig()
        end
        box.Text=""; dropdown.Visible=false; loadTab("Priority")
    end

    local function updateDropdown(query)
        for _, c in ipairs(ddScroll:GetChildren()) do if c:IsA("TextButton") then c:Destroy() end end
        if not query or query == "" then dropdown.Visible = false; return end
        local names = getAllAnimalNames()
        local q = query:lower()
        local matches = {}
        for _, name in ipairs(names) do
            if name:lower():find(q, 1, true) then
                local exists = false
                for _, pName in ipairs(priorityList) do if pName == name then exists = true; break end end
                if not exists then table.insert(matches, name); if #matches >= 8 then break end end
            end
        end
        if #matches == 0 then dropdown.Visible = false; return end
        local itemH = 24
        dropdown.Size = UDim2.new(1,-60,0,math.min(#matches,6)*itemH)
        dropdown.Visible = true
        for _, name in ipairs(matches) do
            local btn = Instance.new("TextButton"); btn.Size=UDim2.new(1,0,0,itemH); btn.BackgroundColor3=Theme.Row; btn.BackgroundTransparency=0.1; btn.Text=name; btn.TextColor3=Theme.Text; btn.Font=Enum.Font.GothamMedium; btn.TextSize=10; btn.AutoButtonColor=false; btn.ZIndex=52; btn.Parent=ddScroll
            btn.MouseEnter:Connect(function() btn.BackgroundColor3=Theme.RowHover end)
            btn.MouseLeave:Connect(function() btn.BackgroundColor3=Theme.Row end)
            btn.MouseButton1Click:Connect(function() addName(name) end)
        end
    end

    box:GetPropertyChangedSignal("Text"):Connect(function() updateDropdown(box.Text) end)
    box.FocusLost:Connect(function() task.delay(0.15, function() dropdown.Visible = false end) end)
    box.Focused:Connect(function() if box.Text ~= "" then updateDropdown(box.Text) end end)

    addBtn.MouseButton1Click:Connect(function() addName(box.Text) end)
end

function makePriorityRestoreRow()
    if not Config.RemovedFromPriority or #Config.RemovedFromPriority == 0 then return end
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.SoftAccent; row.BackgroundTransparency=0.1; row.Parent=mainBody; corner(row,6); row.LayoutOrder = -1
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-96,1,0); l.Position=UDim2.new(0,8,0,0); l.BackgroundTransparency=1; l.Text=tostring(#Config.RemovedFromPriority).." ignored pet(s) available"; l.TextColor3=Theme.Dim; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=row
    local restBtn=Instance.new("TextButton"); restBtn.Name="WhiteTextBtn"; restBtn.Size=UDim2.new(0,74,0,25); restBtn.Position=UDim2.new(1,-80,0.5,-12.5); restBtn.BackgroundColor3=Theme.Green; restBtn.Text="RESTORE"; restBtn.TextColor3=Color3.new(1,1,1); restBtn.Font=Enum.Font.GothamBlack; restBtn.TextSize=10; restBtn.AutoButtonColor=false; restBtn.Parent=row; corner(restBtn,5)
    restBtn.MouseButton1Click:Connect(function()
        for _, name in ipairs(Config.RemovedFromPriority) do
            local exists = false
            for _, pName in ipairs(priorityList) do
                if pName == name then exists = true; break end
            end
            if not exists then table.insert(priorityList, name) end
        end
        Config.RemovedFromPriority = {}
        Config.PriorityList = priorityList
        saveConfig()
        loadTab("Priority")
    end)
end

function loadTab(tabName)
    UI.CurrentTab=tabName; clearBody(mainBody)
    pcall(function() mainBody:FindFirstChildOfClass("UIListLayout").SortOrder = Enum.SortOrder.LayoutOrder end)
    for name,btn in pairs(tabButtons) do btn.BackgroundColor3=name==tabName and Theme.Accent or Theme.SoftAccent; btn.TextColor3=name==tabName and Color3.fromRGB(255,255,255) or Theme.Dim end

    if tabName=="Keybinds" then
        -- NO Steal Speed keybind, NO Unwalk
        for _,name in ipairs({"Kick","Rejoin Job ID","Clone","Manual TP","Invisible Steal","Job ID","Proximity","Carpet Boost","Open Menu","Ragdoll Self","Drop Brainrot","Float","Reset","Auto Buy","Click to AP"}) do makeKeybindRow(mainBody,name) end

    elseif tabName=="Auto TP" then
        makeMainToggle(mainBody,"Auto TP Priority Mode",Config.AutoTPPriority,function(on)
            Config.AutoTPPriority=on
            if on then
                if BoundToggles["Auto TP Highest Gen"] then BoundToggles["Auto TP Highest Gen"](false, false) end
                if BoundToggles["Auto TP Highest Value"] then BoundToggles["Auto TP Highest Value"](false, false) end
                Config.AutoTPHighestGen = false
                Config.AutoTPHighestValue = false
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"Auto TP Highest Gen",Config.AutoTPHighestGen,function(on)
            Config.AutoTPHighestGen=on
            if on then
                if BoundToggles["Auto TP Priority Mode"] then BoundToggles["Auto TP Priority Mode"](false, false) end
                if BoundToggles["Auto TP Highest Value"] then BoundToggles["Auto TP Highest Value"](false, false) end
                Config.AutoTPPriority = false
                Config.AutoTPHighestValue = false
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"Auto TP Highest Value",Config.AutoTPHighestValue,function(on)
            Config.AutoTPHighestValue=on
            if on then
                if BoundToggles["Auto TP Priority Mode"] then BoundToggles["Auto TP Priority Mode"](false, false) end
                if BoundToggles["Auto TP Highest Gen"] then BoundToggles["Auto TP Highest Gen"](false, false) end
                Config.AutoTPPriority = false
                Config.AutoTPHighestGen = false
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"TP on Load",Config.TpSettings.TpOnLoad,function(on) Config.TpSettings.TpOnLoad=on; saveConfig() end)
        -- TP Tool selection (radio button style)
        local toolOptions={"Flying Carpet","Cupid's Wings","Santa's Sleigh","Witch's Broom","Waverider"}
        local toolToggles={}
        for _,tn in ipairs(toolOptions) do
            toolToggles[tn]=makeMainToggle(mainBody,tn,Config.TpSettings.Tool==tn,function(on)
                if on then
                    Config.TpSettings.Tool=tn; saveConfig(); ShowNotification("TP TOOL",tn)
                    for otn,otf in pairs(toolToggles) do if otn~=tn then otf(false,false) end end
                end
            end)
        end
        -- TP Version
        makeMainToggle(mainBody,"Fly TP",Config.TpSettings.FlyTP,function(on)
            Config.TpSettings.FlyTP=on
            if on then
                Config.TpSettings.GrabbleTP = false
                if BoundToggles["Grabble TP"] then BoundToggles["Grabble TP"](false, false) end
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"Grabble TP",Config.TpSettings.GrabbleTP,function(on)
            Config.TpSettings.GrabbleTP=on
            if on then
                Config.TpSettings.FlyTP = false
                if BoundToggles["Fly TP"] then BoundToggles["Fly TP"](false, false) end
            end
            saveConfig()
            -- Grabble TP IS the SXE Clone-TP engine. Sync auto-steal loop state.
            if _G.SXEAutoSteal then
                pcall(_G.SXEAutoSteal, on and (Config.AutoStealEnabled or false))
            end
        end)
        makeMainToggle(mainBody,"Carpet to Brainrot",Config.TpSettings.BrainrotCarpet,function(on) Config.TpSettings.BrainrotCarpet=on; saveConfig() end)

        makeMainButton(mainBody, "Tp Speed", function()
            if tpSpeedSettingsPanel.Visible then closeAnim(tpSpeedSettingsPanel) else openAnim(tpSpeedSettingsPanel) end
        end, Theme.Panel)
        makeMainTextBox(mainBody,"Min Gen for Auto TP",Config.TpSettings.MinGenForTp,"e.g. 50k, 1m, 10b",function(v)
            Config.TpSettings.MinGenForTp = v
            saveConfig()
            ShowNotification("MIN GEN TP", v == "" and "No minimum" or "Min: " .. v)
        end)

    elseif tabName=="ESP" then

        makeMainToggle(mainBody,"Player ESP",playerESPEnabled,function(on) playerESPEnabled=on; Config.PlayerESP=on; saveConfig(); if on then task.spawn(function() for _,pl in ipairs(Players:GetPlayers()) do if pl~=LocalPlayer then pcall(createOrRefreshPlayerESP,pl) end end end) else clearPlayerESP() end end)
        makeMainToggle(mainBody,"Brainrot ESP",brainrotESPEnabled,function(on) brainrotESPEnabled=on; Config.BrainrotESP=on; saveConfig(); if on then task.spawn(function() pcall(refreshBrainrotESP) end) else clearBrainrotESP() end end)
        makeSyncMainToggle(mainBody,"Timer ESP","Timer ESP",function(on)
            timerESPEnabled=on
            Config.TimerESP=on
            saveConfig()
            if on then task.spawn(function() pcall(refreshTimerESP) end) else clearTimerESP() end
        end)
        makeMainToggle(mainBody,"Subspace Mine ESP",subspaceMineESPEnabled,function(on) subspaceMineESPEnabled=on; Config.SubspaceMineESP=on; saveConfig() end)
        makeMainToggle(mainBody,"Base Owner ESP",Config.BaseOwnerESP,function(on) if _G.setBaseOwnerESP then _G.setBaseOwnerESP(on) end end)
        makeMainToggle(mainBody,"Line To Base",Config.LineToBase,function(on)
            Config.LineToBase=on; saveConfig()
            if on then if _G.createPlotBeam then pcall(_G.createPlotBeam) end
            else if _G.resetPlotBeam then pcall(_G.resetPlotBeam) end end
        end)
        makeMainToggle(mainBody,"Line To Best Brainrot",Config.LineToBrainrot or false,function(on)
            Config.LineToBrainrot=on; saveConfig()
            if on then -- beam auto-creates via heartbeat
            else if _G.resetBrainrotBeam then pcall(_G.resetBrainrotBeam) end end
        end)

    elseif tabName=="UI" then
        for _,name in ipairs({"Invisible Steal Panel","Admin Command Panel","Command Cooldowns","Actions","Steal Panel","Steal Target"}) do
            makeMainToggle(mainBody,name,panels[name].Visible,function(on)
                Config.Visibilities[name]=on
                saveConfig()
                if on then openAnim(panels[name]) else closeAnim(panels[name]) end
            end)
        end
        makeMainToggle(mainBody,"Remote Sell Panel",Config.RemoteSellEnabled,function(on) if _G.toggleRemoteSell then _G.toggleRemoteSell(on) end end)
        makeMainToggle(mainBody,"Clear Error Popups",Config.CleanErrorGUIs,function(on) Config.CleanErrorGUIs=on; saveConfig() end)
        makeMainToggle(mainBody,"Auto Close Main UI on Execute",Config.AutoCloseOnExec,function(on) Config.AutoCloseOnExec=on; saveConfig() end)
        makeMainToggle(mainBody,"Admin Panel UI",Config.AdminPanelUI ~= nil and Config.AdminPanelUI or true,function(on)
            Config.AdminPanelUI = on
            saveConfig()
            if apGui then apGui.Enabled = on end
        end)
        local shareRow = Instance.new("Frame")
        shareRow.Size = UDim2.new(1, -4, 0, 31)
        shareRow.BackgroundColor3 = Theme.Panel
        shareRow.BackgroundTransparency = 0.18
        shareRow.Parent = mainBody
        corner(shareRow, 6)
        
        local shareBox = Instance.new("TextBox")
        shareBox.Size = UDim2.new(1, -12, 1, -6)
        shareBox.Position = UDim2.new(0, 6, 0, 3)
        shareBox.BackgroundColor3 = Theme.InputBg
        shareBox.BorderSizePixel = 0
        shareBox.Text = ""
        shareBox.PlaceholderText = "Paste config here to import, or click Export..."
        shareBox.TextColor3 = Theme.Text
        shareBox.Font = Enum.Font.GothamMedium
        shareBox.TextSize = 9
        shareBox.ClearTextOnFocus = false
        shareBox.Parent = shareRow
        corner(shareBox, 4)

        local btnRow = Instance.new("Frame")
        btnRow.Size = UDim2.new(1, -4, 0, 30)
        btnRow.BackgroundTransparency = 1
        btnRow.Parent = mainBody
        
        local expBtn = Instance.new("TextButton")
        expBtn.Size = UDim2.new(0.5, -3, 1, 0)
        expBtn.Position = UDim2.new(0, 0, 0, 0)
        expBtn.BackgroundColor3 = Theme.Row
        expBtn.BackgroundTransparency = 0.16
        expBtn.Text = "Export Config"
        expBtn.TextColor3 = Theme.Text
        expBtn.Font = Enum.Font.GothamBold
        expBtn.TextSize = 11
        expBtn.AutoButtonColor = false
        expBtn.Parent = btnRow
        corner(expBtn, 6)
        stroke(expBtn, Theme.AccentLight, 1, 0.28)
        expBtn.MouseEnter:Connect(function() tw(expBtn, {BackgroundColor3 = Theme.RowHover}, 0.12) end)
        expBtn.MouseLeave:Connect(function() tw(expBtn, {BackgroundColor3 = Theme.Row}, 0.12) end)
        expBtn.MouseButton1Click:Connect(function()
            local str = _G.exportConfig()
            if str then
                shareBox.Text = str
            end
        end)
        
        local impBtn = Instance.new("TextButton")
        impBtn.Size = UDim2.new(0.5, -3, 1, 0)
        impBtn.Position = UDim2.new(0.5, 3, 0, 0)
        impBtn.BackgroundColor3 = Theme.Row
        impBtn.BackgroundTransparency = 0.16
        impBtn.Text = "Import Config"
        impBtn.TextColor3 = Theme.Text
        impBtn.Font = Enum.Font.GothamBold
        impBtn.TextSize = 11
        impBtn.AutoButtonColor = false
        impBtn.Parent = btnRow
        corner(impBtn, 6)
        stroke(impBtn, Theme.AccentLight, 1, 0.28)
        impBtn.MouseEnter:Connect(function() tw(impBtn, {BackgroundColor3 = Theme.RowHover}, 0.12) end)
        impBtn.MouseLeave:Connect(function() tw(impBtn, {BackgroundColor3 = Theme.Row}, 0.12) end)
        impBtn.MouseButton1Click:Connect(function()
            _G.importConfig(shareBox.Text)
        end)

        -- White Mode / Dark Mode buttons removed â€” dark mode only
        -- (Theme is always dark)

        local rb=makeMainButton(mainBody,"Reset UI",function()
            main.Position=UDim2.new(0.5,-187,0.5,-255); panels["Invisible Steal Panel"].Position=UDim2.new(0,80,0.5,-220)
            panels["Admin Command Panel"].Position=UDim2.new(0.5,85,1,-340); panels["Command Cooldowns"].Position=UDim2.new(0.5,245,1,-390)
            panels["Actions"].Position=UDim2.new(0.5,505,1,-415); panels["Steal Panel"].Position=UDim2.new(1,-300,1,-385)
            panels["Steal Target"].Position=UDim2.new(1,-290,0,85); actionSettingsPanel.Position=UDim2.new(0.5,745,1,-400)
            bottomBar.Position=UDim2.new(0.5,-287,1,-125); Config.positions={}; saveConfig()
        end)
        local lockBtn; lockBtn=makeMainButton(mainBody,UI.Locked and "Locked" or "Unlocked",function() UI.Locked=not UI.Locked; Config.locked=UI.Locked; saveConfig(); lockBtn.Text=UI.Locked and "Locked" or "Unlocked" end,Theme.SoftButton)
        
        makeMainToggle(mainBody,"Priority Sound Alert",Config.PrioritySoundAlert,function(on) Config.PrioritySoundAlert=on; saveConfig() end)
        makeMainTextBox(mainBody,"Custom Sound ID",Config.PrioritySoundID or "e.g. 123456789","e.g. 123456789",function(v) Config.PrioritySoundID=v; saveConfig() end)

    elseif tabName=="Misc" then
        -- NO Unwalk, NO Steal Speed, NO FPS Boost (moved to Performance)
        makeMainToggle(mainBody,"Instant Clone",true)
        -- Desync removed
        makeMainToggle(mainBody,"Auto Invisible During Steal",Config.AutoInvisDuringSteal,function(on) _G.AutoInvisDuringSteal=on; Config.AutoInvisDuringSteal=on; saveConfig() end)
        makeMainToggle(mainBody,"Auto Unlock During Steal",Config.AutoUnlockOnSteal,function(on) Config.AutoUnlockOnSteal=on; saveConfig() end)
        makeSyncMainToggle(mainBody,"Anti Ragdoll","Anti Ragdoll",function(on) if on then startAntiRagdoll() else stopAntiRagdoll() end end)
        makeSyncMainToggle(mainBody,"Auto Reset Balloon","Auto Reset Balloon",function(on) Config.AutoResetBalloon=on; saveConfig() end)
        makeSyncMainToggle(mainBody,"Infinite Jump","Infinite Jump",function(on) setInfiniteJump(on) end)
        makeSyncMainToggle(mainBody,"Auto Kick On Steal","Auto Kick",function(on) Config.AutoKickOnSteal=on; saveConfig() end)
        makeMainToggle(mainBody,"Kick to Private Server",Config.KickToPrivateServer,function(on) Config.KickToPrivateServer=on; saveConfig() end)
        makeMainTextBox(mainBody,"Private Server Code",PrivateServerCode,"e.g. ABC123XYZ...",function(v) PrivateServerCode=v; savePSCode() end)
        makeSyncMainToggle(mainBody,"Unwalk","Unwalk",function(on) if _G.setUnwalk then _G.setUnwalk(on) end end)
        makeSyncMainToggle(mainBody,"Invisible Steal","Invisible Steal",function(on) if _G.toggleInvisibleSteal then pcall(_G.toggleInvisibleSteal) end end)
        makeSyncMainToggle(mainBody,"Float","Float",function(on) setFloat(on) end)
        makeSyncMainToggle(mainBody,"Carpet Speed","Carpet Speed",function(on) setCarpetSpeed(on) end)
        makeSyncMainToggle(mainBody,"Click to AP","ClickToAP",function(on) Config.ClickToAP=on; saveConfig() end)

        -- Helper: create a config panel with toggles + ordering for AP commands
        local function makeAPConfigPanel(panelKey, titleText, configTable, orderKey, onToggleCb)
            if panels[panelKey] then
                panels[panelKey].Visible = not panels[panelKey].Visible
                return
            end

            -- Initialize order from config or default
            if not Config[orderKey] then
                Config[orderKey] = {}
                for i, cmd in ipairs(AP_ALL_COMMANDS) do Config[orderKey][i] = cmd end
            end
            local cmdOrder = Config[orderKey]

            local cfgPanel = Instance.new("Frame")
            cfgPanel.Name = panelKey
            cfgPanel.Size = UDim2.fromOffset(240, 0)
            cfgPanel.AutomaticSize = Enum.AutomaticSize.Y
            cfgPanel.Position = panelKey == "AdminPanelCmds" and UDim2.new(0.5, 200, 0.5, -200) or UDim2.new(0.5, 200, 0.5, -50)
            cfgPanel.BackgroundColor3 = Theme.Background
            cfgPanel.BackgroundTransparency = 0.04
            cfgPanel.BorderSizePixel = 0
            cfgPanel.ZIndex = 100 -- High ZIndex to stay on top
            cfgPanel.Parent = gui_sg -- Parent to ScreenGui to act as a floating popup
            panels[panelKey] = cfgPanel
            corner(cfgPanel, 10)
            stroke(cfgPanel, Theme.Accent, 1.5, 0.3)
            makeDraggable(cfgPanel, cfgPanel)

            local cfgTitleLbl = Instance.new("TextLabel")
            cfgTitleLbl.Size = UDim2.new(1, 0, 0, 28)
            cfgTitleLbl.BackgroundColor3 = Theme.Panel
            cfgTitleLbl.BackgroundTransparency = 0.3
            cfgTitleLbl.Text = titleText
            cfgTitleLbl.TextColor3 = Theme.Text
            cfgTitleLbl.Font = Enum.Font.GothamBold
            cfgTitleLbl.TextSize = 12
            cfgTitleLbl.ZIndex = 101 -- Fix: Render on top of panel
            cfgTitleLbl.Parent = cfgPanel
            corner(cfgTitleLbl, 8)

            local cfgBody = Instance.new("Frame")
            cfgBody.Size = UDim2.new(1, -8, 0, 0)
            cfgBody.AutomaticSize = Enum.AutomaticSize.Y
            cfgBody.Position = UDim2.fromOffset(4, 32)
            cfgBody.BackgroundTransparency = 1
            cfgBody.ZIndex = 101 -- Fix: Render on top of panel
            cfgBody.Parent = cfgPanel
            Instance.new("UIListLayout", cfgBody).SortOrder = Enum.SortOrder.LayoutOrder
            cfgBody:FindFirstChildOfClass("UIListLayout").Padding = UDim.new(0, 2)

            local rowMap = {} -- cmd -> row frame
            local numMap = {} -- cmd -> number label

            local function refreshNumbers()
                for i, cmd in ipairs(cmdOrder) do
                    if rowMap[cmd] then rowMap[cmd].LayoutOrder = i end
                    if numMap[cmd] then numMap[cmd].Text = tostring(i) end
                end
            end

            local function swapOrder(idx1, idx2)
                if idx1 < 1 or idx2 < 1 or idx1 > #cmdOrder or idx2 > #cmdOrder then return end
                cmdOrder[idx1], cmdOrder[idx2] = cmdOrder[idx2], cmdOrder[idx1]
                Config[orderKey] = cmdOrder
                saveConfig()
                refreshNumbers()
            end

            for idx, cmd in ipairs(cmdOrder) do
                local emoji = AP_COMMAND_EMOJIS[cmd] or "⚡"
                local isOn = configTable[cmd] == true
                local row = Instance.new("Frame")
                row.Size = UDim2.new(1, 0, 0, 28)
                row.BackgroundColor3 = Theme.Panel
                row.BackgroundTransparency = 0.4
                row.LayoutOrder = idx
                row.ZIndex = 102 -- Fix: Render on top of body
                row.Parent = cfgBody
                corner(row, 5)
                rowMap[cmd] = row

                -- Number label
                local numLbl = Instance.new("TextLabel")
                numLbl.Size = UDim2.fromOffset(16, 28)
                numLbl.Position = UDim2.fromOffset(2, 0)
                numLbl.BackgroundTransparency = 1
                numLbl.Text = tostring(idx)
                numLbl.TextColor3 = Theme.Dim
                numLbl.Font = Enum.Font.GothamBold
                numLbl.TextSize = 10
                numLbl.ZIndex = 103
                numLbl.Parent = row
                numMap[cmd] = numLbl

                -- Up arrow
                local upBtn = Instance.new("TextButton")
                upBtn.Size = UDim2.fromOffset(16, 13)
                upBtn.Position = UDim2.fromOffset(18, 1)
                upBtn.BackgroundTransparency = 1
                upBtn.Text = "▲"
                upBtn.TextColor3 = Theme.AccentLight
                upBtn.Font = Enum.Font.GothamBold
                upBtn.TextSize = 8
                upBtn.AutoButtonColor = false
                upBtn.ZIndex = 103
                upBtn.Parent = row

                -- Down arrow
                local dnBtn = Instance.new("TextButton")
                dnBtn.Size = UDim2.fromOffset(16, 13)
                dnBtn.Position = UDim2.fromOffset(18, 14)
                dnBtn.BackgroundTransparency = 1
                dnBtn.Text = "▼"
                dnBtn.TextColor3 = Theme.AccentLight
                dnBtn.Font = Enum.Font.GothamBold
                dnBtn.TextSize = 8
                dnBtn.AutoButtonColor = false
                dnBtn.ZIndex = 103
                dnBtn.Parent = row

                upBtn.MouseButton1Click:Connect(function()
                    local curIdx
                    for i, c in ipairs(cmdOrder) do if c == cmd then curIdx = i; break end end
                    if curIdx and curIdx > 1 then swapOrder(curIdx, curIdx - 1) end
                end)
                dnBtn.MouseButton1Click:Connect(function()
                    local curIdx
                    for i, c in ipairs(cmdOrder) do if c == cmd then curIdx = i; break end end
                    if curIdx and curIdx < #cmdOrder then swapOrder(curIdx, curIdx + 1) end
                end)

                -- Command label
                local lbl = Instance.new("TextLabel")
                lbl.Size = UDim2.new(1, -90, 1, 0)
                lbl.Position = UDim2.fromOffset(36, 0)
                lbl.BackgroundTransparency = 1
                lbl.Text = emoji .. " " .. cmd
                lbl.TextColor3 = Theme.Text
                lbl.Font = Enum.Font.GothamMedium
                lbl.TextSize = 11
                lbl.TextXAlignment = Enum.TextXAlignment.Left
                lbl.ZIndex = 103
                lbl.Parent = row

                -- Toggle button
                local btn = Instance.new("TextButton")
                btn.Size = UDim2.fromOffset(36, 18)
                btn.Position = UDim2.new(1, -42, 0.5, -9)
                btn.BackgroundColor3 = isOn and Theme.Green or Theme.ToggleOff
                btn.Text = isOn and "ON" or "OFF"
                btn.TextColor3 = Color3.new(1, 1, 1)
                btn.Font = Enum.Font.GothamBold
                btn.TextSize = 9
                btn.AutoButtonColor = false
                btn.ZIndex = 103
                btn.Parent = row
                corner(btn, 5)

                btn.MouseButton1Click:Connect(function()
                    isOn = not isOn
                    configTable[cmd] = isOn
                    saveConfig()
                    btn.BackgroundColor3 = isOn and Theme.Green or Theme.ToggleOff
                    btn.Text = isOn and "ON" or "OFF"
                    if onToggleCb then onToggleCb(cmd, isOn) end
                end)
            end

            if panelKey == "ClickToAPCmds" then
                local function setZIndex(inst)
                    if inst:IsA("GuiObject") then
                        inst.ZIndex = 103
                    end
                    for _, child in ipairs(inst:GetChildren()) do
                        setZIndex(child)
                    end
                end

                local sep = Instance.new("Frame")
                sep.Size = UDim2.new(1, -8, 0, 1)
                sep.Position = UDim2.new(0, 4, 0, 0)
                sep.BackgroundColor3 = Theme.Stroke
                sep.BackgroundTransparency = 0.2
                sep.LayoutOrder = 1000
                sep.ZIndex = 103
                sep.Parent = cfgBody

                local subHeader = Instance.new("TextLabel")
                subHeader.Size = UDim2.new(1, 0, 0, 24)
                subHeader.BackgroundTransparency = 1
                subHeader.Text = "Settings"
                subHeader.TextColor3 = Theme.Accent
                subHeader.Font = Enum.Font.GothamBold
                subHeader.TextSize = 10
                subHeader.LayoutOrder = 1001
                subHeader.ZIndex = 103
                subHeader.Parent = cfgBody

                local sliderWrapper = Instance.new("Frame")
                sliderWrapper.Size = UDim2.new(1, 0, 0, 50)
                sliderWrapper.BackgroundTransparency = 1
                sliderWrapper.LayoutOrder = 1002
                sliderWrapper.Parent = cfgBody

                local sliderObj = makeQuickSlider(sliderWrapper, "Click Radius", 1, 50, Config.ClickToAPRadius or 8, function(v)
                    Config.ClickToAPRadius = v
                    saveConfig()
                end, " studs")
                setZIndex(sliderWrapper)
            end

            if panelKey == "SpamBaseOwnerCmds" then
                local function setZIndex(inst)
                    if inst:IsA("GuiObject") then
                        inst.ZIndex = 103
                    end
                    for _, child in ipairs(inst:GetChildren()) do
                        setZIndex(child)
                    end
                end

                local sep = Instance.new("Frame")
                sep.Size = UDim2.new(1, -8, 0, 1)
                sep.Position = UDim2.new(0, 4, 0, 0)
                sep.BackgroundColor3 = Theme.Stroke
                sep.BackgroundTransparency = 0.2
                sep.LayoutOrder = 1000
                sep.ZIndex = 103
                sep.Parent = cfgBody

                local subHeader = Instance.new("TextLabel")
                subHeader.Size = UDim2.new(1, 0, 0, 24)
                subHeader.BackgroundTransparency = 1
                subHeader.Text = "Settings"
                subHeader.TextColor3 = Theme.Accent
                subHeader.Font = Enum.Font.GothamBold
                subHeader.TextSize = 10
                subHeader.LayoutOrder = 1001
                subHeader.ZIndex = 103
                subHeader.Parent = cfgBody

                local row = Instance.new("Frame")
                row.Size = UDim2.new(1, 0, 0, 34)
                row.BackgroundTransparency = 1
                row.LayoutOrder = 1002
                row.Parent = cfgBody

                local toggleFunc = makeSyncStateRow(row, "Single Command:", "SpamBaseOwnerSingleCommand", function(on)
                    Config.SpamBaseOwnerSingleCommand = on
                    saveConfig()
                end)
                setZIndex(row)
            end

            local pad = Instance.new("Frame")
            pad.Size = UDim2.new(1, 0, 0, 6)
            pad.BackgroundTransparency = 1
            pad.LayoutOrder = 1004
            pad.Parent = cfgBody
        end

        -- Admin Panel Commands button (controls which buttons show per player in admin panel)
        makeMainButton(mainBody, "Admin Panel Commands", function()
            makeAPConfigPanel("AdminPanelCmds", "Admin Panel Commands", Config.AdminPanelButtons, "AdminPanelOrder", function()
                if _G.refreshAdminPanelRows then _G.refreshAdminPanelRows() end
            end)
        end)

        -- Click to AP Commands button (controls which commands fire when you click a player)
        makeMainButton(mainBody, "Click to AP Commands", function()
            makeAPConfigPanel("ClickToAPCmds", "Click to AP Commands", Config.ClickToAPCommands, "ClickToAPOrder")
        end)

        -- Spam Base Owner Commands button
        makeMainButton(mainBody, "Spam Base Owner Commands", function()
            makeAPConfigPanel("SpamBaseOwnerCmds", "Spam Base Owner Commands", Config.SpamBaseOwnerCommands, "SpamBaseOwnerOrder")
        end)
        makeSyncMainToggle(mainBody,"Anti-Bee & Anti-Disco","AntiBeeDisco",function(on)
            Config.AntiBeeDisco = on
            saveConfig()
            if on then
                if _G.ANTI_BEE_DISCO and _G.ANTI_BEE_DISCO.Enable then
                    _G.ANTI_BEE_DISCO.Enable()
                end
            else
                if _G.ANTI_BEE_DISCO and _G.ANTI_BEE_DISCO.Disable then
                    _G.ANTI_BEE_DISCO.Disable()
                end
            end
        end)
        makeMainTextBox(mainBody,"Min Gen for Nearest Grab",Config.TpSettings.MinGenForGrab,"e.g. 10k, 500k, 1m",function(v)
            Config.TpSettings.MinGenForGrab = v
            saveConfig()
            ShowNotification("MIN GEN GRAB", v == "" and "No minimum" or "Min: " .. v)
        end)

    elseif tabName=="Priority" then
        makePriorityAddRow(); makePriorityRestoreRow(); for i=1,#priorityList do makePriorityRow(i) end

    elseif tabName=="Performance" then
        makeSyncMainToggle(mainBody,"FPS Boost (normal)","FPS Boost (normal)",function(on) setFPSBoost(on) end)
        makeSyncMainToggle(mainBody,"FPS Boost Ultra","FPSBoostUltra",function(on) setFPSBoostUltra(on) end)
        makeSyncMainToggle(mainBody,"Xray","XRay",function(on) setXRay(on) end)
        makeQuickSlider(mainBody,"FOV",50,120,Config.FOV or 70,function(v) Config.FOV=v; saveConfig(); pcall(function() Workspace.CurrentCamera.FieldOfView=v end) end)
    end

    local idx = 0
    for _, child in ipairs(mainBody:GetChildren()) do
        if child:IsA("GuiObject") and not child:IsA("UIListLayout") and not child:IsA("UIPadding") then
            idx = idx + 1
            child.LayoutOrder = idx
        end
    end
end

for tabName,btn in pairs(tabButtons) do btn.MouseButton1Click:Connect(function() loadTab(tabName) end) end

-- BOTTOM BAR
bottomBar=Instance.new("Frame"); bottomBar.Size=UDim2.new(0,575,0,50); bottomBar.Position=UDim2.new(0.5,-287,1,-125); bottomBar.BackgroundColor3=Theme.Background; bottomBar.BackgroundTransparency=0.02; bottomBar.BorderSizePixel=0; bottomBar.Parent=gui; corner(bottomBar,12); addOutline(bottomBar)
local iw=Instance.new("Frame"); iw.Size=UDim2.new(0,34,0,34); iw.Position=UDim2.new(0,12,0.5,-17); iw.BackgroundColor3=Theme.SoftAccent; iw.BorderSizePixel=0; iw.Parent=bottomBar; corner(iw,8)
local ic=Instance.new("ImageLabel"); ic.Size=UDim2.new(1,0,1,0); ic.BackgroundTransparency=1; ic.ScaleType=Enum.ScaleType.Fit; ic.Parent=iw; corner(ic,8)

_G.updateLogoImage = function(isDark)
    if isDark then
        ic.Image = "rbxthumb://type=Asset&id=98944824494349&w=150&h=150"
    else
        ic.Image = "rbxthumb://type=Asset&id=110857950376835&w=150&h=150"
    end
    ic.ImageColor3 = Color3.fromRGB(255, 255, 255)
    iw.BackgroundColor3 = Theme.SoftAccent
end
_G.updateLogoImage(Config and Config.DarkMode or false)
local lg=Instance.new("TextLabel"); lg.Size=UDim2.new(0,150,0,26); lg.Position=UDim2.new(0,54,0,5); lg.BackgroundTransparency=1; lg.Text="ALPHA HUB PRIVAT"; lg.TextColor3=Theme.AccentLight; lg.Font=Enum.Font.GothamBlack; lg.TextSize=19; lg.TextXAlignment=Enum.TextXAlignment.Left; lg.Parent=bottomBar
local dd=Instance.new("TextLabel"); dd.Size=UDim2.new(0,20,0,26); dd.Position=UDim2.new(0,210,0,5); dd.BackgroundTransparency=1; dd.Text="|"; dd.TextColor3=Theme.AccentLight; dd.Font=Enum.Font.GothamBlack; dd.TextSize=18; dd.Parent=bottomBar
local dc=Instance.new("TextLabel"); dc.Size=UDim2.new(0,210,0,26); dc.Position=UDim2.new(0,230,0,5); dc.BackgroundTransparency=1; dc.Text="discord.gg/j2pxrygT4"; dc.TextColor3=Theme.AccentLight; dc.Font=Enum.Font.GothamBold; dc.TextSize=16; dc.TextXAlignment=Enum.TextXAlignment.Left; dc.Parent=bottomBar
local sb=Instance.new("TextLabel"); sb.Size=UDim2.new(0,290,0,14); sb.Position=UDim2.new(0,55,0,30); sb.BackgroundTransparency=1; sb.Text="By:@SE67 and @SXLVATORE"; sb.TextColor3=Theme.Dim; sb.Font=Enum.Font.GothamSemibold; sb.TextSize=8; sb.TextXAlignment=Enum.TextXAlignment.Left; sb.Parent=bottomBar
local rightDiv=Instance.new("Frame"); rightDiv.Size=UDim2.new(0,1,0,36); rightDiv.Position=UDim2.new(1,-138,0.5,-18); rightDiv.BackgroundColor3=Theme.Accent; rightDiv.BackgroundTransparency=0.35; rightDiv.BorderSizePixel=0; rightDiv.Parent=bottomBar
fpsText=Instance.new("TextLabel"); fpsText.Size=UDim2.new(0,126,1,0); fpsText.Position=UDim2.new(1,-128,0,0); fpsText.BackgroundTransparency=1; fpsText.Text="FPS: --\nPING: --ms"; fpsText.TextColor3=Theme.Green; fpsText.Font=Enum.Font.GothamBold; fpsText.TextSize=10; fpsText.TextXAlignment=Enum.TextXAlignment.Left; fpsText.Parent=bottomBar
if _G.addLazyUI then _G.addLazyUI(bottomBar, true) end

task.defer(function()
    loadTab("Auto TP")
    rebuildActions(); rebuildActionSettings(); rebuildTpSpeedSettings()
end)

-- FPS/PING COUNTER
local frames,lastT=0,tick()
_G.currentFPS = 60
RunService.RenderStepped:Connect(function() frames=frames+1; local now=tick(); if now-lastT>=1 then local fps=frames; _G.currentFPS=fps; frames=0; lastT=now; local ping=0; pcall(function() ping=math.floor(LocalPlayer:GetNetworkPing() * 1000) end); fpsText.Text="FPS: "..fps.."\nPING: "..ping.."ms" end end)

-- INPUT HANDLER (NO Steal Speed, NO Unwalk keybinds)
UIS.InputBegan:Connect(function(input,gp) if gp then return end; if input.UserInputType~=Enum.UserInputType.Keyboard then return end
    if input.KeyCode==UI.OpenMenuKey then
        if main.Visible then
            closeAnim(main)
            if tpSpeedSettingsPanel and tpSpeedSettingsPanel.Visible then closeAnim(tpSpeedSettingsPanel) end
            if actionSettingsPanel and actionSettingsPanel.Visible then closeAnim(actionSettingsPanel) end
            local apCmds = gui_sg and gui_sg:FindFirstChild("AdminPanelCmds")
            if apCmds and apCmds.Visible then apCmds.Visible = false end
            local clickCmds = gui_sg and gui_sg:FindFirstChild("ClickToAPCmds")
            if clickCmds and clickCmds.Visible then clickCmds.Visible = false end
            local sbOwnerCmds = gui_sg and gui_sg:FindFirstChild("SpamBaseOwnerCmds")
            if sbOwnerCmds and sbOwnerCmds.Visible then sbOwnerCmds.Visible = false end
        else openAnim(main) end
    return end
    local kn=input.KeyCode.Name
    local actions={
        ["Drop Brainrot"]=runDropBrainrot, ["Clone"]=instantClone,
        ["Float"]=function() setFloat(not FloatState.active) end,
        ["Carpet Boost"]=function() setCarpetSpeed(not CarpetState.enabled) end,
        ["Reset"]=executeReset, ["Kick"]=kickPlayer,
        ["Proximity"]=function() ProximityAPActive=not ProximityAPActive; setToggle("Proximity",ProximityAPActive) end,
        ["Ragdoll Self"]=function() pcall(runAdminCommand,player,"ragdoll") end,
        ["Invisible Steal"]=function() if _G.toggleInvisibleSteal then pcall(_G.toggleInvisibleSteal) end end,
        ["Rejoin Job ID"]=function() pcall(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, player) end) end,
        ["Manual TP"]=function() task.spawn(runAutoSnipe) end,
        ["Auto Buy"]=function() toggleAutoBuy() end,
        ["Click to AP"]=function()
            Config.ClickToAP = not Config.ClickToAP
            saveConfig()
            setToggle("Click to AP", Config.ClickToAP)
            setToggle("ClickToAP", Config.ClickToAP)
            ShowNotification("CLICK TO AP", Config.ClickToAP and "Enabled" or "Disabled")
        end,
    }
    for actionName,keyBind in pairs(Keybinds) do if keyBind==kn and actions[actionName] then actions[actionName](); return end end
end)


-- AUTO-EXECUTE FROM CONFIG (Robust Retry Loop)
if Config.TpSettings.TpOnLoad then task.spawn(function()
    local char = player.Character or player.CharacterAdded:Wait()
    char:WaitForChild("HumanoidRootPart", 20)
    char:WaitForChild("Humanoid", 20)
    
    local t = 0
    while (not SharedState.InitialScanComplete or #SharedState.AllAnimalsCache == 0) and t < 150 do
        task.wait(0.1)
        t = t + 1
    end
    task.wait(0.15) -- Extreme optimized network sync wait
    
    local minGen = parseMinGen(Config.TpSettings.MinGenForTp)
    local success = false
    local attempts = 0
    while not success and attempts < 30 do
        attempts = attempts + 1
        if player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChild("Humanoid") and player.Character.Humanoid.Health > 0 then
            local targetPetData = getTargetPetData()
            
            if targetPetData then
                local targetPart = findAdorneeGlobal(targetPetData)
                if targetPart then
                    task.spawn(runAutoSnipe)
                    success = true
                    break
                end
            end
        end
        task.wait(0.5)
    end
end) end

task.spawn(function() task.wait(0.5)
    -- Desync removed from startup
    if Config.ClickToAP then Config.ClickToAP = false end
    if Config.AntiRagdoll then startAntiRagdoll() end
    if Config.InfiniteJump then setInfiniteJump(true) end
    if Config.Float then setFloat(true) end
    if Config.Unwalk then setUnwalk(true) end

    if Config.AutoCloseOnExec then if main then main.Visible = false end end
    if Config.FPSBoost then setFPSBoost(true) end
    if Config.FPSBoostUltra then setFPSBoostUltra(true) end
    if Config.XRay then setXRay(true) end
    pcall(function() Workspace.CurrentCamera.FieldOfView = Config.FOV or 70 end)
    if Config.PlayerESP then playerESPEnabled=true end
    if Config.TimerESP then timerESPEnabled=true end
    if Config.SubspaceMineESP then subspaceMineESPEnabled=true end
    -- RemoteSell is initialized inside the lazy UI thread (at the top of the file)
    _G.initRemoteSellLazy = function()
        if Config.RemoteSellEnabled and _G.toggleRemoteSell then
            _G.toggleRemoteSell(true)
        end
    end
    if Config.ProximityAP then setProximityAP(true) end
    if Config.AutoBuyEnabled then toggleAutoBuy(true) end
    if Config.StealHighest then setStealMode("Highest")
    elseif Config.StealPriority then setStealMode("Priority")
    elseif Config.StealNearest then setStealMode("Nearest")
    else setStealMode("Highest") end
    if updateMovementPanelLabels then updateMovementPanelLabels() end
end)

_G.InvisStealAngle=Config.InvisStealAngle or 225; _G.SinkSliderValue=Config.SinkSliderValue or 7
_G.AutoRecoverLagback=true; _G.AutoInvisDuringSteal=Config.AutoInvisDuringSteal or false

print("ALPHA HUB PRIVAT loaded ")


task.spawn(function()
    while true do
        task.wait(15)
        pcall(collectgarbage, "collect")
    end
end)



RunService.RenderStepped:Connect(function()
    pcall(function()
        local cam = Workspace.CurrentCamera
        if cam and Config.FOV and cam.FieldOfView ~= Config.FOV then
            cam.FieldOfView = Config.FOV
        end
    end)
end)

local function doTP()
    local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    pcall(function()
        if sethiddenproperty then
            local cf = hrp.CFrame
            sethiddenproperty(hrp, "CFrame", CFrame.new(cf.Position.X + 500, cf.Position.Y, cf.Position.Z + 500) * (cf - cf.Position))
            task.wait(0.23)
            sethiddenproperty(hrp, "CFrame", cf)
        end
    end)
end

-- No Player Collision Logic
local lastNoclipUpdate = 0
RunService.Stepped:Connect(function()
    local now = tick()
    if now - lastNoclipUpdate < 0.1 then return end
    lastNoclipUpdate = now
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            for _, part in ipairs(p.Character:GetDescendants()) do
                if part:IsA("BasePart") then 
                    part.CanCollide = false 
                end
            end
        end
    end
end)
local ALPHA_AC = {} 
end 
   end

task.spawn(function()
    task.wait(0.15)
    pcall(applyTheme, "Dark")
end)


-- ============================================================
-- SXE AUTO-STEAL + CLONE-TP ENGINE (integrated) -- xentp.lua 1:1 port
-- Speed: Config.TpSettings.GrabbleTPSpeed | Clone Delay: Config.TpSettings.CloneDelayVal
-- ============================================================
do
    local Players    = game:GetService("Players")
    local RunService = game:GetService("RunService")
    local UIS        = game:GetService("UserInputService")
    local RS         = game:GetService("ReplicatedStorage")
    local LP = Players.LocalPlayer

    -- ===== Modules =====
    local Synchronizer, AnimalsData, AnimalsShared, NumberUtils
    local function loadModules()
        if Synchronizer then return true end
        local ok = pcall(function()
            local Packages = RS:WaitForChild("Packages", 5)
            local Datas = RS:WaitForChild("Datas", 5)
            local Shared = RS:WaitForChild("Shared", 5)
            local Utils = RS:WaitForChild("Utils", 5)
            Synchronizer = require(Packages:WaitForChild("Synchronizer"))
            AnimalsData = require(Datas:WaitForChild("Animals"))
            AnimalsShared = require(Shared:WaitForChild("Animals"))
            NumberUtils = require(Utils:WaitForChild("NumberUtils"))
        end)
        return ok and Synchronizer ~= nil
    end

    local NetModule
    local function loadNet()
        if NetModule then return true end
        local ok, mod = pcall(function()
            return require(RS:WaitForChild("Packages", 5):WaitForChild("Net", 5):FindFirstChildWhichIsA("ModuleScript", true))
        end)
        if not ok or type(mod) ~= "table" then return false end
        NetModule = mod
        return true
    end

    local function fireGrapple()
        if not NetModule then loadNet() end
        if not NetModule then return end
        local char = LP.Character
        if not char then return end
        if not char:FindFirstChild("Grapple Hook") then
            local bp = LP:FindFirstChild("Backpack")
            local tool = bp and bp:FindFirstChild("Grapple Hook")
            local hum = char:FindFirstChildOfClass("Humanoid")
            if tool and hum then pcall(function() hum:EquipTool(tool) end) end
        end
        if not char:FindFirstChild("Grapple Hook") then return end
        pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end)
    end
    _G.SXEFireGrapple = fireGrapple

    -- ===== Speed config (live-settable) =====
    local SXESpeed = { CARPET = 400, INBASE = 250 }
    _G.SXESetCarpetSpeed = function(v) v = tonumber(v); if v and v > 0 then SXESpeed.CARPET = v end end
    _G.SXESetInbaseSpeed = function(v) v = tonumber(v); if v and v > 0 then SXESpeed.INBASE = v end end
    _G.SXEGetCarpetSpeed = function() return SXESpeed.CARPET end
    if Config and Config.TpSettings then
        if tonumber(Config.TpSettings.GrabbleTPSpeed) then SXESpeed.CARPET = tonumber(Config.TpSettings.GrabbleTPSpeed) end
    end

    -- ===== Tools =====
    local CARPET_NAMES = { "Flying Carpet", "Carpet", "Cloud", "Witch's Broom", "Cupid's Wings", "Santa's Sleigh", "Magic Carpet" }
    local GRAPPLE_NAMES = { "Grapple Hook", "Grappling Hook", "Grapple", "Hook", "Web Slinger", "Grapple Gun", "GrappleHook" }
    local function findTool(name)
        local char = LP.Character
        local bp = LP:FindFirstChild("Backpack")
        return (char and char:FindFirstChild(name)) or (bp and bp:FindFirstChild(name))
    end
    local function findGrapple()
        for _, n in ipairs(GRAPPLE_NAMES) do
            local t = findTool(n); if t then return t end
        end
        return nil
    end
    local function equipCarpet()
        local char = LP.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return nil end
        local preferred = Config and Config.TpSettings and Config.TpSettings.Tool
        if preferred then
            local pt = findTool(preferred)
            if pt and pt:IsA("Tool") then
                if pt.Parent ~= char then pcall(function() hum:EquipTool(pt) end) end
                return preferred
            end
        end
        for _, n in ipairs(CARPET_NAMES) do
            local t = findTool(n)
            if t and t:IsA("Tool") then
                if t.Parent ~= char then pcall(function() hum:EquipTool(t) end) end
                return n
            end
        end
        return nil
    end
    local function carpetEngage()
        if not NetModule then pcall(loadNet) end
        local _t0 = os.clock()
        while not findTool("Grapple Hook") and os.clock() - _t0 < 5 do
            if not NetModule then pcall(loadNet) end
            RunService.Heartbeat:Wait()
        end
        local char = LP.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not char or not hum then return nil end
        if not char:FindFirstChild("Grapple Hook") then
            local g = findTool("Grapple Hook")
            if g then pcall(function() hum:EquipTool(g) end) end
        end
        task.wait(0.08)
        if NetModule and LP.Character and LP.Character:FindFirstChild("Grapple Hook") then
            pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end)
        end
        task.wait(0.15)
        local h = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
        if h then pcall(function() h:UnequipTools() end) end
        task.wait(0.15)
        local cn
        local _tc = os.clock()
        repeat
            cn = equipCarpet()
            local c = LP.Character
            if cn and c and c:FindFirstChild(cn) then break end
            RunService.Heartbeat:Wait()
        until os.clock() - _tc > 1
        return cn
    end

    -- ===== Priority system =====
    local PET_PRIORITY_TIERS = {
        [1]  = { pets = {"Headless Horseman"}, threshold = 0 },
        [2]  = { pets = {"Signore Carapace"}, threshold = 0 },
        [3]  = { pets = {"Strawberry Elephant"}, threshold = 0 },
        [4]  = { pets = {"Arcadragon"}, threshold = 0 },
        [5]  = { pets = {"Elefanto Frigo"}, threshold = 5e9 },
        [6]  = { pets = {"John Pork"}, threshold = 10e9 },
        [7]  = { pets = {"Meowl"}, threshold = 5e9 },
        [8]  = { pets = {"Skibidi Toilet"}, threshold = 5e9 },
        [9]  = { pets = {"Love Love Bear"}, threshold = 0 },
        [10] = { pets = {"Antonio"}, threshold = 0 },
        [11] = { pets = {"Pancake and Syrup"}, threshold = 0 },
        [12] = { pets = {"Griffin"}, threshold = 0 },
        [13] = { pets = {"La Supreme Combinasion","Fishino Clownino","Dragon Gingerini","Tirilikalika Tirilikalako"}, threshold = 5e9 },
        [14] = { pets = {"Ginger Gerat","Pet"}, threshold = 10e9 },
        [15] = { pets = {"Hydra Bunny","Digi Narwhal","Kalika Bros"}, threshold = 3e9 },
        [16] = { pets = {"Hydra Dragon Cannelloni","Dragon Cannelloni","Bunny and Eggy"}, threshold = 3e9 },
        [17] = { pets = {"Globa Steppa","Ketupat Bros","Rosey and Teddy","La Casa Boo","Fragola la la"}, threshold = 3e9 },
        [18] = { pets = {"Fragola La La La","Cerberus","Guest 666","Los Hackers"}, threshold = 1e9 },
        [19] = { pets = {"Garama and Madundung","Spooky and Pumpky","Reinito Sleighito","Burguro And Fryuro","Cooki and Milki","Fragrama and Chocrama","La Food Combinasion","Los Amigos","Foxini Lanternini","Capitano Moby","Fortunu and Cashuru","Los Sekolahs","Celestial Pegasus"}, threshold = 750e6 },
        [20] = { pets = {"La Secret Combinasion","Sammyni Fattini","Cloverat Clapat","Popcuru and Fizzuru"}, threshold = 1e9 },
    }
    local TIER_LOOKUP = {}
    for tier, data in pairs(PET_PRIORITY_TIERS) do
        for _, name in ipairs(data.pets) do TIER_LOOKUP[name] = tier end
    end
    local LOCKED_TIERS = { [1]=true,[2]=true,[3]=true,[4]=true }
    local DIRECT_THRESHOLDS = {
        [3] = { [4] = 10e9 },
        [4] = {},
        [5] = { [6] = math.huge },
        [6] = { [9] = math.huge, [10] = math.huge, [12] = 15e9 },
        [10] = { [12] = 20e9 },
        [11] = { [12] = 10e9 },
    }
    local MUTATION_PRIORITY = {
        ["Galaxy"]=1,["Candy"]=1,["Yin Yang"]=1,["YinYang"]=1,["Divine"]=1,
        ["Cursed"]=1,["Lava"]=1,["Radioactive"]=1,["Cyber"]=1,["Rainbow"]=1,["Bloodrot"]=2,
    }
    local MUTATED_BEATS_GRIFFIN = {
        ["Fishino Clownino"]=true,["Globa Steppa"]=true,
        ["La Supreme Combinasion"]=true,["Tirilikalika Tirilikalako"]=true,
    }
    local function getMutPrio(m)
        if not m or m == "" or m == "None" then return 0 end
        if MUTATION_PRIORITY[m] then return MUTATION_PRIORITY[m] end
        local n = tostring(m):lower():gsub("[%s%-_]","")
        if n == "bloodrot" then return 2 end
        if n == "yinyang" or n == "galaxy" or n == "candy" or n == "divine"
            or n == "cursed" or n == "lava" or n == "radioactive" or n == "cyber"
            or n == "rainbow" then return 1 end
        return 0
    end
    local function getCumThreshold(hi, lo)
        if DIRECT_THRESHOLDS[hi] and DIRECT_THRESHOLDS[hi][lo] then return DIRECT_THRESHOLDS[hi][lo] end
        if LOCKED_TIERS[hi] then return math.huge end
        local total = 0
        for t = hi + 1, lo do
            local td = PET_PRIORITY_TIERS[t]
            if td and td.threshold > 0 then total = total + td.threshold end
        end
        return total
    end
    local function petOutranks(aName, bName, aMut, bMut, aMPS, bMPS)
        if aName == "Strawberry Elephant" and bName == "John Pork" then return true end
        if aName == "John Pork" and bName == "Strawberry Elephant" then return false end
        if MUTATED_BEATS_GRIFFIN[aName] and bName == "Griffin" and getMutPrio(aMut) >= 1 then return true end
        if aName == "Griffin" and MUTATED_BEATS_GRIFFIN[bName] and getMutPrio(bMut) >= 1 then return false end
        if aName == "Antonio" and bName == "Elefanto Frigo" and getMutPrio(aMut) >= 1 then return true end
        if aName == "Elefanto Frigo" and bName == "Antonio" and getMutPrio(bMut) >= 1 then return false end
        local tA = TIER_LOOKUP[aName] or 99
        local tB = TIER_LOOKUP[bName] or 99
        if not (TIER_LOOKUP[aName] and TIER_LOOKUP[bName]) then
            if tA == tB then return (aMPS or 0) > (bMPS or 0) end
            return tA < tB
        end
        if tA == tB then
            local pA, pB = getMutPrio(aMut), getMutPrio(bMut)
            if pA ~= pB then return pA > pB end
            return (aMPS or 0) > (bMPS or 0)
        end
        if tA == 4 and tB == 3 then return true end
        if tA == 3 and tB == 4 then return false end
        local hi = math.min(tA, tB)
        local lo = math.max(tA, tB)
        local hiMPS = tA < tB and aMPS or bMPS
        local loMPS = tA < tB and bMPS or aMPS
        local cum = getCumThreshold(hi, lo)
        if cum > 0 and cum ~= math.huge then
            if (loMPS or 0) - (hiMPS or 0) > cum then return tA > tB end
        end
        return tA < tB
    end

    -- ===== Plot/Channel helpers =====
    local function getPlotChannel(plotName)
        if not Synchronizer then return nil end
        local channel
        pcall(function() channel = Synchronizer:Get(plotName) end)
        if not channel then pcall(function() channel = Synchronizer:Wait(plotName) end) end
        return channel
    end
    local function channelGet(channel, key)
        if not channel then return nil end
        local v
        pcall(function() if type(channel.Get) == "function" then v = channel:Get(key) end end)
        if v == nil then pcall(function() v = channel.CacheTable and channel.CacheTable[key] end) end
        return v
    end
    local function isMyPlot(channel)
        if not channel then return false end
        local owner = channelGet(channel, "Owner")
        if not owner then return false end
        local result = false
        pcall(function()
            if typeof(owner) == "Instance" and owner:IsA("Player") then
                result = owner.UserId == LP.UserId
            elseif type(owner) == "table" and owner.UserId then
                result = owner.UserId == LP.UserId
            elseif typeof(owner) == "Instance" then
                result = owner == LP
            end
        end)
        return result
    end
    local function ownerInGame(channel)
        if not channel then return false end
        local owner = channelGet(channel, "Owner")
        if not owner then return false end
        local inGame = false
        pcall(function()
            if typeof(owner) == "Instance" and owner:IsA("Player") then
                inGame = Players:FindFirstChild(owner.Name) ~= nil
            elseif type(owner) == "number" then
                inGame = Players:GetPlayerByUserId(owner) ~= nil
            elseif type(owner) == "table" and owner.Name then
                inGame = Players:FindFirstChild(tostring(owner.Name)) ~= nil
            elseif typeof(owner) == "Instance" and owner.Name then
                inGame = Players:FindFirstChild(owner.Name) ~= nil
            end
        end)
        return inGame
    end
    local function isPlotUnlocked(plotName)
        local ok, res = pcall(function()
            local channel = getPlotChannel(plotName)
            if not channel then return false end
            return channelGet(channel, "BlockEndTimeFirstFloor") == nil
        end)
        return ok and (res == true)
    end

    -- ===== Pet position =====
    local function getPetPosition(plot, slot)
        local podiums = plot:FindFirstChild("AnimalPodiums")
        if not podiums then return nil end
        local podium = podiums:FindFirstChild(tostring(slot))
        if not podium then return nil end
        for _, desc in ipairs(podium:GetDescendants()) do
            if desc:IsA("Model") and desc.Name ~= "Claim" and desc.Name ~= "Base" and desc.Name ~= "Decorations" then
                local hasMesh = false
                for _, c in ipairs(desc:GetDescendants()) do
                    if c:IsA("MeshPart") then hasMesh = true; break end
                end
                if hasMesh then
                    local ok, cf = pcall(function() return desc:GetBoundingBox() end)
                    if ok then return cf.Position end
                end
            end
        end
        local ok, cf = pcall(function() return podium:GetPivot() end)
        if ok then return cf.Position end
        return podium.Position
    end

    -- ===== Fusing check =====
    local _BLOCKING_MACHINE_TYPES = { Fuse=true, Duel=true, Trade=true, Crafting=true }
    local function _SXEIsFusing(animalData)
        if type(animalData) ~= "table" then return false end
        local m = animalData.Machine
        if type(m) ~= "table" then return false end
        return _BLOCKING_MACHINE_TYPES[m.Type] == true and m.Active == true
    end

    -- ===== Scanner =====
    local function scanAllPets()
        local pets = {}
        if not loadModules() then return pets end
        local Plots = workspace:FindFirstChild("Plots")
        if not Plots then return pets end
        for _, plot in ipairs(Plots:GetChildren()) do
            local channel = getPlotChannel(plot.Name)
            if not channel then continue end
            if isMyPlot(channel) then continue end
            if not ownerInGame(channel) then continue end
            local animalList = channelGet(channel, "AnimalList")
            if not animalList then continue end
            for slot, animalData in pairs(animalList) do
                if type(animalData) ~= "table" then continue end
                local animalName = animalData.Index
                if not animalName then continue end
                local animalInfo = AnimalsData and AnimalsData[animalName]
                if not animalInfo then continue end
                if _SXEIsFusing(animalData) then continue end
                local mutation = animalData.Mutation or "None"
                local genValue = 0
                pcall(function()
                    genValue = AnimalsShared:GetGeneration(animalName, animalData.Mutation, animalData.Traits, nil)
                end)
                local displayName = (animalInfo and animalInfo.DisplayName) or animalName
                local pos = getPetPosition(plot, slot)
                if pos then
                    table.insert(pets, {
                        name = displayName, index = animalName, mps = genValue,
                        mutation = mutation, position = pos, plot = plot.Name, slot = tostring(slot),
                    })
                end
            end
        end
        local conveyorFolder = workspace:FindFirstChild("RenderedMovingAnimals")
        if conveyorFolder then
            for _, model in ipairs(conveyorFolder:GetChildren()) do
                pcall(function()
                    if not model:IsA("Model") then return end
                    local animalInfo = AnimalsData and AnimalsData[model.Name]
                    if not animalInfo then return end
                    local mutation = model:GetAttribute("Mutation") or "None"
                    local genValue = 0
                    pcall(function()
                        genValue = AnimalsShared:GetGeneration(model.Name, model:GetAttribute("Mutation"), nil, nil) or 0
                    end)
                    if genValue <= 0 then return end
                    local part = model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart")
                    if not part then return end
                    table.insert(pets, {
                        name = animalInfo.DisplayName or model.Name, index = model.Name,
                        mps = genValue, mutation = mutation, position = part.Position,
                        plot = nil, slot = nil, conveyor = true, model = model,
                    })
                end)
            end
        end
        table.sort(pets, function(a, b)
            return petOutranks(a.name, b.name, a.mutation, b.mutation, a.mps, b.mps)
        end)
        return pets
    end

    -- ===== Sky coordinates =====
    local UPPER = {
        B = {{coord=Vector3.new(-487.921448,16.850713,-75.768013),facing="NORTH"},{coord=Vector3.new(-332.379730,16.850722,-75.762100),facing="NORTH"},{coord=Vector3.new(-487.134918,16.850713,-18.094154),facing="SOUTH"},{coord=Vector3.new(-316.300171,16.850713,-17.845898),facing="SOUTH"}},
        C = {{coord=Vector3.new(-330.765381,16.850713,31.424425),facing="NORTH"},{coord=Vector3.new(-502.989349,16.850713,31.172430),facing="NORTH"},{coord=Vector3.new(-489.077087,16.850713,89.010147),facing="SOUTH"},{coord=Vector3.new(-330.908936,16.850713,88.930145),facing="SOUTH"}},
        D = {{coord=Vector3.new(-331.264893,16.850713,138.209167),facing="NORTH"},{coord=Vector3.new(-487.935181,16.850713,138.026321),facing="NORTH"},{coord=Vector3.new(-487.774933,16.850713,195.882538),facing="SOUTH"},{coord=Vector3.new(-330.799133,16.850575,196.022354),facing="SOUTH"}},
    }
    local LOWER = {
        B = {{coord=Vector3.new(-335.725586,-3.048217,-74.984589),facing="NORTH"},{coord=Vector3.new(-503.214233,-3.048217,-75.043137),facing="NORTH"},{coord=Vector3.new(-483.619385,-3.718430,-18.844337),facing="SOUTH"},{coord=Vector3.new(-316.147095,-3.048218,-18.818844),facing="SOUTH"}},
        C = {{coord=Vector3.new(-335.985413,-3.048218,32.051426),facing="NORTH"},{coord=Vector3.new(-503.277008,-3.048217,31.956175),facing="NORTH"},{coord=Vector3.new(-483.749390,-3.048218,88.147003),facing="SOUTH"},{coord=Vector3.new(-315.793823,-3.048217,88.163979),facing="SOUTH"}},
        D = {{coord=Vector3.new(-335.476654,-3.048218,139.001083),facing="NORTH"},{coord=Vector3.new(-503.710083,-3.048218,138.989883),facing="NORTH"},{coord=Vector3.new(-315.654938,-3.048218,195.302444),facing="SOUTH"},{coord=Vector3.new(-483.859253,-3.048218,195.269043),facing="SOUTH"}},
    }
    local UPPER_Y_THRESHOLD = 7
    local TALL_PETS = { ["La Secret Combinasion"]=true, ["La Jolly Grande"]=true }
    local TALL_OFFSET = 3
    local BASES_LOW = {
        [1]=Vector3.new(-476.52,-2,220.94),[2]=Vector3.new(-476.52,-2,113.77),
        [3]=Vector3.new(-476.52,-2,6.18),[4]=Vector3.new(-476.52,-2,-101.07),
        [5]=Vector3.new(-342.66,-2,221.45),[6]=Vector3.new(-342.66,-2,113.41),
        [7]=Vector3.new(-342.66,-2,6.25),[8]=Vector3.new(-342.66,-2,-99.73),
    }
    local BASES_HIGH = {
        [1]=Vector3.new(-479.51,18,220.94),[2]=Vector3.new(-479.51,18,113.77),
        [3]=Vector3.new(-479.51,18,6.18),[4]=Vector3.new(-479.51,18,-101.07),
        [5]=Vector3.new(-339.48,18,221.45),[6]=Vector3.new(-339.48,18,113.41),
        [7]=Vector3.new(-339.48,18,6.25),[8]=Vector3.new(-339.48,18,-99.73),
    }
    local FRONT_Y_LOW = -3.048217
    local FRONT_Y_HIGH = 16.850713
    local COLUMN_SPLIT_X = -410
    local FRONT_Z_CLAMP = 18
    local SIDE_NEAR_Z = 45

    local function getClosestBaseIdx(pos)
        local closest, dist = 1, math.huge
        for i = 1, 8 do
            local b = BASES_LOW[i]
            local d = (pos.X - b.X)^2 + (pos.Z - b.Z)^2
            if d < dist then dist = d; closest = i end
        end
        return closest
    end
    local function buildFrontCandidate(idx, isUpper, playerZ)
        local base = isUpper and BASES_HIGH[idx] or BASES_LOW[idx]
        local frontY = isUpper and FRONT_Y_HIGH or FRONT_Y_LOW
        local frontZ = math.clamp(playerZ - base.Z, -FRONT_Z_CLAMP, FRONT_Z_CLAMP) + base.Z
        local coord = Vector3.new(base.X, frontY, frontZ)
        local faceDir = (idx <= 4) and Vector3.new(-1, 0, 0) or Vector3.new(1, 0, 0)
        return coord, faceDir
    end
    local function plotSides(coordTable, idx)
        local base = BASES_LOW[idx]
        local isWest = idx <= 4
        local out = {}
        for _, coords in pairs(coordTable) do
            for _, data in ipairs(coords) do
                if ((data.coord.X < COLUMN_SPLIT_X) == isWest)
                   and math.abs(data.coord.Z - base.Z) < SIDE_NEAR_Z then
                    out[#out + 1] = data
                end
            end
        end
        return out
    end
    local function findClosest(petPos, coordTable)
        local best, bestKey, bestDist = nil, nil, math.huge
        for skyKey, coords in pairs(coordTable) do
            for _, data in ipairs(coords) do
                local c = data.coord
                local d = math.sqrt((petPos.X - c.X)^2 + (petPos.Z - c.Z)^2)
                if d < bestDist then bestDist = d; best = data; bestKey = skyKey end
            end
        end
        return best, bestKey
    end

    -- ===== Viz =====
    local _vizParts = {}
    local function clearViz()
        for _, p in ipairs(_vizParts) do if p and p.Parent then p:Destroy() end end
        table.clear(_vizParts)
    end
    local function vizPath(fromPos, waypoints)
        -- Pathfinder lines/dots disabled (user request: no lines).
        return
    end

    -- ===== Movement =====
    local SPEED = 200
    local ARRIVE = 3
    local MAX_CLIMB = 60

    local function vZero(hrp)
        if hrp then hrp.AssemblyLinearVelocity = Vector3.zero; hrp.AssemblyAngularVelocity = Vector3.zero end
    end

    local function velMoveThrough(hrp, waypoints, speedOverride, allowJump, quickStart)
        if not hrp or not hrp.Parent or #waypoints == 0 then return end
        local _runSpeed = speedOverride or SXESpeed.CARPET
        vizPath(hrp.Position, waypoints)
        local wpIdx = 1
        local done = false
        local conn
        local function finish()
            if done then return end
            done = true
            if hrp and hrp.Parent then
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                local _, y = hrp.CFrame:ToEulerAnglesYXZ()
                hrp.CFrame = CFrame.new(waypoints[#waypoints]) * CFrame.Angles(0, y, 0)
            end
            if conn then conn:Disconnect() end
            clearViz()
        end
        local lastDist, stall = math.huge, 0
        if quickStart then
            local _hp = RaycastParams.new()
            _hp.FilterType = Enum.RaycastFilterType.Exclude
            _hp.IgnoreWater = true
            local _skip = {}
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl.Character then _skip[#_skip + 1] = pl.Character end
            end
            _hp.FilterDescendantsInstances = _skip
            for _ = 1, 3 do
                local target = waypoints[wpIdx]
                if not target then break end
                local flat = Vector3.new(target.X - hrp.Position.X, 0, target.Z - hrp.Position.Z)
                local mag = flat.Magnitude
                if mag < 1 then break end
                local nextPos = hrp.Position + flat.Unit * math.min(20, mag)
                local _hit = workspace:Raycast(hrp.Position, nextPos - hrp.Position, _hp)
                if _hit and _hit.Instance and _hit.Instance.CanCollide then break end
                hrp.CFrame = (hrp.CFrame - hrp.CFrame.Position) + nextPos
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                RunService.Heartbeat:Wait()
                if not hrp or not hrp.Parent then return end
            end
        end
        conn = RunService.Heartbeat:Connect(function()
            if not hrp or not hrp.Parent or done then
                if conn then conn:Disconnect() end; return
            end
            equipCarpet()
            local target = waypoints[wpIdx]
            local diff = target - hrp.Position
            local mag = diff.Magnitude
            if mag < ARRIVE then
                wpIdx = wpIdx + 1
                if wpIdx > #waypoints then finish(); return end
                lastDist, stall = math.huge, 0
                target = waypoints[wpIdx]
                diff = target - hrp.Position
                mag = diff.Magnitude
            end
            if mag > lastDist - 0.05 then stall = stall + 1 else stall = 0 end
            lastDist = mag
            if stall >= 18 then finish(); return end
            if mag >= 0.1 then
                local dir = diff.Unit
                if allowJump and diff.Y > 5 and wpIdx < #waypoints then
                    local hum = hrp.Parent and hrp.Parent:FindFirstChildOfClass("Humanoid")
                    if hum then
                        local st = hum:GetState()
                        if st ~= Enum.HumanoidStateType.Jumping and st ~= Enum.HumanoidStateType.Freefall then
                            pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end)
                            pcall(function() hum.Jump = true end)
                        end
                    end
                end
                local _vy = dir.Y * _runSpeed
                if _vy > MAX_CLIMB then _vy = MAX_CLIMB end
                hrp.Velocity = Vector3.new(dir.X * _runSpeed, _vy, dir.Z * _runSpeed)
            end
        end)
        local totalDist = 0
        local prev = hrp.Position
        for _, wp in ipairs(waypoints) do totalDist = totalDist + (prev - wp).Magnitude; prev = wp end
        local timeout = totalDist / math.min(SPEED, _runSpeed) + 2
        local elapsed = 0
        while not done and elapsed < timeout do task.wait(0.05); elapsed = elapsed + 0.05 end
        finish()
        vZero(hrp)
    end

    -- ===== Routing =====
    local _DIRS = { Vector3.new(1,0,0), Vector3.new(-1,0,0), Vector3.new(0,0,1), Vector3.new(0,0,-1) }
    local _STRUCT = { ["structure base home"]=true, ["Wall"]=true, ["Floor"]=true, ["Roof"]=true }
    local _SKIP_NAME = { ["DeliveryHitbox"]=true, ["StealHitbox"]=true, ["LaserHitbox"]=true,
        ["AnimalTarget"]=true, ["Multiplier"]=true, ["Laser"]=true, ["Hitbox"]=true,
        ["Spawn"]=true, ["MainRoot"]=true, ["SecondFloor"]=true, ["ThirdFloor"]=true, ["Slope"]=true }
    local function _blocks(inst)
        if not inst then return false end
        if _SKIP_NAME[inst.Name] then return false end
        if inst.CanCollide then return true end
        if _STRUCT[inst.Name] then return true end
        local s = inst.Size
        if s and math.max(s.X * s.Y, s.X * s.Z, s.Y * s.Z) > 150 then return true end
        return false
    end
    local function _blocksWide(inst)
        if not inst then return false end
        if _SKIP_NAME[inst.Name] then return false end
        if inst.CanCollide then return true end
        if _STRUCT[inst.Name] then return true end
        local s = inst.Size
        if s and math.max(s.X * s.Y, s.X * s.Z, s.Y * s.Z) > 30 then return true end
        return false
    end
    local function _block(origin, target, blockFn)
        blockFn = blockFn or _blocks
        local rp = RaycastParams.new(); rp.FilterType = Enum.RaycastFilterType.Exclude; rp.IgnoreWater = true
        local skip = {}
        for _, pl in ipairs(Players:GetPlayers()) do if pl.Character then skip[#skip + 1] = pl.Character end end
        local o = origin
        for _ = 1, 16 do
            rp.FilterDescendantsInstances = skip
            local d = target - o; if d.Magnitude < 0.05 then return nil end
            local res = workspace:Raycast(o, d, rp)
            if not res then return nil end
            if blockFn(res.Instance) then return res end
            skip[#skip + 1] = res.Instance; o = res.Position + d.Unit * 0.3
        end
        return nil
    end
    local function _clear(a, b) return _block(a, b) == nil end
    local function _clearDist(origin, dir, maxD)
        local res = _block(origin, origin + dir.Unit * maxD)
        if not res then return maxD end
        return (res.Position - origin).Magnitude
    end
    local function _pull(pts)
        if #pts <= 2 then return pts end
        local out = { pts[1] }; local i = 1
        while i < #pts do
            local j = #pts
            while j > i + 1 and not _clear(out[#out], pts[j]) do j = j - 1 end
            out[#out + 1] = pts[j]; i = j
        end
        return out
    end
    local function _clearWideRay(a, b) return _block(a, b, _blocksWide) == nil end
    local function _clearWide(a, b)
        if not _clear(a, b) then return false end
        local d = Vector3.new(b.X - a.X, 0, b.Z - a.Z)
        if d.Magnitude < 0.1 then return true end
        local _CLEARANCE = 10
        local perp = Vector3.new(-d.Z, 0, d.X).Unit * _CLEARANCE
        local up = Vector3.new(0, _CLEARANCE, 0)
        return _clearWideRay(a + perp, b + perp) and _clearWideRay(a - perp, b - perp)
            and _clearWideRay(a + up, b + up) and _clearWideRay(a - up, b - up)
    end
    local function _pullWide(pts)
        if #pts <= 2 then return pts end
        local out = { pts[1] }; local i = 1
        while i < #pts do
            local j = #pts
            while j > i + 1 and not _clearWide(out[#out], pts[j]) do j = j - 1 end
            out[#out + 1] = pts[j]; i = j
        end
        return out
    end
    local function _pushOffWalls(pts)
        if #pts <= 2 then return pts end
        local MARGIN = 10; local MAX_PUSH = 14
        local out = { pts[1] }
        for i = 2, #pts - 1 do
            local p = pts[i]; local shift = Vector3.zero
            for _, dr in ipairs(_DIRS) do
                local res = _block(p, p + dr * MARGIN, _blocks)
                if res then
                    local dist = (res.Position - p).Magnitude
                    if dist < MARGIN then shift = shift - dr * (MARGIN - dist) end
                end
            end
            if shift.Magnitude > 0.1 then
                if shift.Magnitude > MAX_PUSH then shift = shift.Unit * MAX_PUSH end
                local moved = p + shift
                if _clear(out[#out], moved) then out[#out + 1] = moved else out[#out + 1] = p end
            else
                out[#out + 1] = p
            end
        end
        out[#out + 1] = pts[#pts]
        return out
    end
    local PathfindingService = game:GetService("PathfindingService")
    local function computeRoute(fromPos, toPos, facingDir)
        if _clear(fromPos, toPos) then return { toPos } end
        local entry = facingDir and (toPos - facingDir * 14) or toPos
        local groundTo = Vector3.new(entry.X, fromPos.Y, entry.Z)
        local path = PathfindingService:CreatePath({
            AgentRadius = 12, AgentHeight = 5, AgentCanJump = true, AgentJumpHeight = 10, AgentMaxSlope = 89,
        })
        local FLOAT = 3
        local nav = { fromPos }
        local ok = pcall(function()
            path:ComputeAsync(Vector3.new(fromPos.X, fromPos.Y, fromPos.Z), groundTo)
        end)
        if ok and path.Status == Enum.PathStatus.Success then
            local last = fromPos
            for _, wp in ipairs(path:GetWaypoints()) do
                if (wp.Position - last).Magnitude >= 8 then
                    nav[#nav + 1] = wp.Position + Vector3.new(0, FLOAT, 0); last = wp.Position
                end
            end
        end
        nav[#nav + 1] = entry + Vector3.new(0, FLOAT, 0)
        nav = _pushOffWalls(nav)
        local route = _pullWide(nav)
        route[#route + 1] = toPos
        return route
    end

    -- ===== Clone / goToBrainrot =====
    local function doClone()
        if not NetModule then pcall(loadNet) end
        local char = LP.Character or LP.CharacterAdded:Wait()
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not char or not hum then return false end
        local cloner = (LP:FindFirstChild("Backpack") and LP.Backpack:FindFirstChild("Quantum Cloner"))
                    or char:FindFirstChild("Quantum Cloner")
        if not cloner then return false end
        if cloner.Parent ~= char then
            pcall(function() hum:EquipTool(cloner) end)
            task.wait()
        end
        if not NetModule then return false end
        local useOk = pcall(function() NetModule:RemoteEvent("UseItem"):FireServer() end)
        task.wait(0.05)
        local telOk = pcall(function() NetModule:RemoteEvent("QuantumCloner/OnTeleport"):FireServer() end)
        return useOk and telOk
    end

    -- CARPET GLIDE: fly straight to the pet on the equipped carpet at FULL 3D speed
    -- (same pace as velMoveThrough -- WalkTPSpeed in the actual direction, vertical
    -- included). The carpet is already equipped by goToBrainrot, so no extra equip
    -- wait. Used as the FINAL approach after the clone/grabble TP.
    local function carpetGlideTo(targetPos)
        if not targetPos then return end
        local char = LP.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        pcall(function() hrp.Anchored = false end)
        pcall(equipCarpet)
        local SPEED = (Config.TpSettings and (tonumber(Config.TpSettings.WalkTPSpeed) or tonumber(Config.TpSettings.GrabbleTPSpeed))) or 190
        local FLOAT_OFFSET = (targetPos.Y > 20) and -4 or 0
        local goal = Vector3.new(targetPos.X, targetPos.Y + FLOAT_OFFSET, targetPos.Z)
        local t0 = os.clock()
        local lastDist, stall = math.huge, 0
        while hrp.Parent and (os.clock() - t0) < 8 do
            if LP:GetAttribute("Stealing") then break end
            equipCarpet()
            local diff = goal - hrp.Position
            local mag = diff.Magnitude          -- full 3D distance (so it doesn't stop short on upper floors)
            if mag < 3 then break end
            if mag > lastDist - 0.05 then stall = stall + 1 else stall = 0 end
            lastDist = mag
            if stall >= 30 then break end       -- stuck on a wall -> give up
            hrp.AssemblyLinearVelocity = diff.Unit * SPEED   -- full speed straight at the pet
            RunService.Heartbeat:Wait()
        end
        if hrp and hrp.Parent then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end
    end

    local function goToBrainrot(petPos)
        if not petPos then return end
        local char, hrp, hum
        local _t0 = os.clock()
        repeat
            char = LP.Character
            hrp = char and char:FindFirstChild("HumanoidRootPart")
            hum = char and char:FindFirstChildOfClass("Humanoid")
            if hrp and hum then break end
            RunService.Heartbeat:Wait()
        until os.clock() - _t0 > 3
        if not hrp or not hum then return end
        pcall(function() hrp.Anchored = false end)
        local _equipped = false
        do
            local _e0 = os.clock()
            repeat
                char = LP.Character
                for _, _cn in ipairs(CARPET_NAMES) do
                    if char and char:FindFirstChild(_cn) then _equipped = true; break end
                end
                if _equipped then break end
                equipCarpet()
                RunService.Heartbeat:Wait()
            until _equipped or os.clock() - _e0 > 1.5
            if _equipped then task.wait(0.2) end
        end
        char = LP.Character
        hrp = char and char:FindFirstChild("HumanoidRootPart")
        hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hrp then return end
        pcall(function() hrp.Anchored = false end)
        local _plotRad = (petPos.Y <= 8.9) and 26 or 25
        do
            local _t0b = os.clock()
            repeat
                local p = hrp.Position
                local inRad = false
                local plotsFolder = workspace:FindFirstChild("Plots")
                if plotsFolder then
                    for _, plot in ipairs(plotsFolder:GetChildren()) do
                        pcall(function()
                            local pp = plot:GetPivot().Position
                            if math.abs(p.X - pp.X) < _plotRad and math.abs(p.Z - pp.Z) < _plotRad then inRad = true end
                        end)
                        if inRad then break end
                    end
                end
                if inRad then break end
                RunService.Heartbeat:Wait()
            until os.clock() - _t0b > 1.5
        end
        local h = petPos.Y
        local targetY = hrp.Position.Y
        if h > 23.15 then targetY = 21
        elseif h >= 11 and h <= 23.15 then targetY = 14.5
        elseif h >= -6.9 and h <= 8.9 then targetY = -4 end
        local _to = Vector3.new(petPos.X, targetY, petPos.Z)
        if Config.TpSettings and Config.TpSettings.BrainrotCarpet then
            -- Carpet to Brainrot: after the clone/grabble TP into the base, glide on
            -- the carpet straight to the pet (uses live Walk To Brainrot Speed).
            carpetGlideTo(petPos)
        else
            local _route = computeRoute(hrp.Position, _to, nil)
            if not _route or #_route == 0 then _route = { _to } end
            velMoveThrough(hrp, _route, (Config and Config.TpSettings and (tonumber(Config.TpSettings.WalkTPSpeed) or tonumber(Config.TpSettings.GrabbleTPSpeed))) or SXESpeed.INBASE, true, true)
        end
        if hrp and hrp.Parent then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end
        do
            local _platPos = (hrp and hrp.Parent and hrp.Position) or _to
            local _feetY = _platPos.Y - 3
            local _plat = Instance.new("Part")
            _plat.Name = "SXETempPlatform"; _plat.Size = Vector3.new(8, 1, 8)
            _plat.Position = Vector3.new(petPos.X, _feetY - 1.5, petPos.Z)
            _plat.Anchored = true; _plat.CanCollide = false; pcall(makeOneWay, _plat); _plat.Transparency = 1
            _plat.Material = Enum.Material.SmoothPlastic; _plat.Parent = workspace
            task.spawn(function()
                local _s = tick()
                while tick() - _s < 20 do
                    if LP:GetAttribute("Stealing") then break end
                    task.wait(0.1)
                end
                if _plat and _plat.Parent then _plat:Destroy() end
            end)
        end
    end

    -- ===== Steal engine =====
    local InternalStealCache = {}
    local STEAL_HOLD_DURATION = 1.3
    local _stealHoldStart = 0
    local _stealHoldActive = false

    local function buildStealCallbacks(prompt)
        if InternalStealCache[prompt] then return end
        if not prompt or not prompt.Parent then return end
        local data = { holdCallbacks = {}, triggerCallbacks = {}, holdEndCallbacks = {}, ready = true }
        local function grab(sig, into)
            local ok, conns = pcall(getconnections, sig)
            if ok and type(conns) == "table" then
                for _, c in ipairs(conns) do
                    if type(c.Function) == "function" then table.insert(into, c.Function) end
                end
            end
        end
        grab(prompt.PromptButtonHoldBegan, data.holdCallbacks)
        grab(prompt.Triggered, data.triggerCallbacks)
        grab(prompt.PromptButtonHoldEnded, data.holdEndCallbacks)
        if #data.holdCallbacks > 0 or #data.triggerCallbacks > 0 or #data.holdEndCallbacks > 0 then
            InternalStealCache[prompt] = data
        end
    end

    local function executeStealAsync(prompt)
        local data = InternalStealCache[prompt]
        if not data or not data.ready then return false end
        data.ready = false
        _stealHoldStart = tick()
        _stealHoldActive = true
        _G.ALPHA_StealStatus = _G.ALPHA_StealStatus or {}
        _G.ALPHA_StealStatus.active = true
        _G.ALPHA_StealStatus.start = _stealHoldStart
        _G.ALPHA_StealStatus.duration = STEAL_HOLD_DURATION

        task.spawn(function()
            for _, fn in ipairs(data.holdCallbacks) do task.spawn(fn) end
            pcall(function()
                local _st = prompt:GetAttribute("State")
                if _st ~= nil and _st ~= "Steal" then
                    if not _G._xenStealRemote then
                        local _net = _G.XenNet or require(game:GetService("ReplicatedStorage"):WaitForChild("Packages"):WaitForChild("Net"):FindFirstChildWhichIsA("ModuleScript", true))
                        _G._xenStealRemote = _net and _net:RemoteEvent("f40f7d9e-2f0d-4167-b250-899273f46874")
                    end
                    local r = _G._xenStealRemote
                    if r then
                        local _t = workspace:GetServerTimeNow() + 124
                        r:FireServer(_t, "68c86eb7-eb7e-4b4d-96ae-cf7cd847c5b0")
                        r:FireServer(_t, "07b9cc25-2a1f-4a26-a0ec-f2fab578d8bd")
                    end
                end
            end)
            local remain = STEAL_HOLD_DURATION - (tick() - _stealHoldStart)
            if remain > 0 then task.wait(remain) end
            if prompt and prompt.Parent then
                for _, fn in ipairs(data.triggerCallbacks) do task.spawn(fn) end
            end
            for _, fn in ipairs(data.holdEndCallbacks) do task.spawn(fn) end
            _stealHoldActive = false
            if _G.ALPHA_StealStatus then _G.ALPHA_StealStatus.active = false end
            task.wait(0.05)
            data.ready = true
        end)
        return true
    end

    local function findStealPrompt(pet)
        if pet.plot and pet.slot then
            local plots = workspace:FindFirstChild("Plots")
            local plot = plots and plots:FindFirstChild(pet.plot)
            local podiums = plot and plot:FindFirstChild("AnimalPodiums")
            local podium = podiums and podiums:FindFirstChild(tostring(pet.slot))
            if podium then
                local base = podium:FindFirstChild("Base")
                local spawn = base and base:FindFirstChild("Spawn")
                local attach = spawn and spawn:FindFirstChild("PromptAttachment")
                if attach then
                    for _, p in ipairs(attach:GetChildren()) do
                        if p:IsA("ProximityPrompt") then return p end
                    end
                end
                for _, d in ipairs(podium:GetDescendants()) do
                    if d:IsA("ProximityPrompt") then return d end
                end
            end
        end
        if pet.model and pet.model.Parent then
            for _, d in ipairs(pet.model:GetDescendants()) do
                if d:IsA("ProximityPrompt") then return d end
            end
        end
        return nil
    end

    local STEAL_PROXIMITY = 60
    local STEAL_ARM_TIMEOUT = 25
    local _stealTarget = nil
    local _stealArmedAt = 0

    local function timeUntilCanSteal()
        if LP:GetAttribute("Stealing") or LP:GetAttribute("IsTrading")
            or LP:GetAttribute("IsDuelSelecting") or LP:GetAttribute("Web") then
            return -1
        end
        return 0
    end

    local function _petKey(pet)
        if not pet then return nil end
        if pet.plot and pet.slot then return tostring(pet.plot) .. "|" .. tostring(pet.slot) end
        return "idx|" .. tostring(pet.index or pet.name)
    end
    local function _petStillExists(sel, pets)
        if not sel or not pets then return nil end
        local k = _petKey(sel)
        for _, p in ipairs(pets) do if _petKey(p) == k then return p end end
        return nil
    end
    local function _pickByMode(pets)
        if not pets or #pets == 0 then return nil end
        local C = Config or {}
        -- PRIORITY FIRST: honored when EITHER the Auto-TP "Priority Mode" OR the
        -- Steal "Priority" mode is on (the two tabs are now synced here). Follow
        -- the user's priorityList order (1 -> 2 -> 3 ...), match by DisplayName OR
        -- Index, and ALWAYS take the TOPMOST priority entry present on the server,
        -- regardless of distance/gen. So with Headless Horseman at #1, any Headless
        -- on the server is taken and everything else is ignored.
        local priorityMode = C.AutoTPPriority or (C.StealMode == "Priority")
        if priorityMode and priorityList and #priorityList > 0 then
            for _, pName in ipairs(priorityList) do
                local searchName = pName:lower()
                for _, p in ipairs(pets) do
                    if (p.name and p.name:lower() == searchName) or (p.index and p.index:lower() == searchName) then
                        return p
                    end
                end
            end
        end
        -- HIGHEST gen/value: Auto-TP Highest Gen/Value OR Steal "Highest".
        if C.AutoTPHighestGen or C.AutoTPHighestValue or (C.StealMode == "Highest") then
            local best, bv
            for _, p in ipairs(pets) do
                local v = p.mps or 0
                if not bv or v > bv then bv, best = v, p end
            end
            return best
        end
        return pets[1]
    end
    local function _samePet(a, b)
        if not a or not b then return false end
        return a.plot == b.plot and tostring(a.slot) == tostring(b.slot)
    end

    local _stealTarget2 = nil
    local function armSteal(pet)
        if not pet then return end
        _stealTarget = pet; _stealArmedAt = os.clock()
        _stealTarget2 = pet
        _G.ALPHA_StealStatus = _G.ALPHA_StealStatus or {}
        _G.ALPHA_StealStatus.target = pet
    end
    local function disarmSteal()
        _stealTarget = nil
        _G.ALPHA_StealStatus = _G.ALPHA_StealStatus or {}
        _G.ALPHA_StealStatus.target = nil
        _G.ALPHA_StealStatus.active = false
    end
    _G.SXEArmSteal = armSteal
    _G.SXEDisarmSteal = disarmSteal

    local AUTO_STEAL = (Config and Config.AutoStealEnabled) and true or false
    _G.SXEAutoSteal = function(on) AUTO_STEAL = on ~= false end

    local _stealLastScan = 0
    local _autoLastScan = 0
    local isTeleporting = false
    local _tpStartedAt = 0
    local _cloneTP = false
    local _cloneFired = false
    local _started = false

    RunService.Heartbeat:Connect(function()
        local now = os.clock()
        if AUTO_STEAL and _started and not LP:GetAttribute("Stealing")
            and not (isTeleporting and _cloneTP)
            and (now - _autoLastScan) >= 0.1 then
            _autoLastScan = now
            local char = LP.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                local ok, pets = pcall(scanAllPets)
                if ok and pets then
                    local best
                    -- Manual selection from SXEHub
                    if manuallySelectedUID then
                        for _, pet in ipairs(pets) do
                            if not pet.conveyor and pet.plot and pet.slot then
                                local uid = pet.plot .. "_" .. tostring(pet.slot)
                                if uid == manuallySelectedUID then best = pet; break end
                            end
                        end
                    end
                    if not best then
                        local STEAL_MODE = (Config and Config.StealMode) or "Priority"
                        if STEAL_MODE == "Nearest" then
                            local bd
                            for _, pet in ipairs(pets) do
                                if not pet.conveyor then
                                    local prompt = findStealPrompt(pet)
                                    if prompt and prompt.Parent then
                                        local pp = prompt.Parent
                                        local ppPos = (pp:IsA("BasePart") and pp.Position)
                                            or (pp.Parent and pp.Parent:IsA("BasePart") and pp.Parent.Position)
                                        if ppPos then
                                            local d = (hrp.Position - ppPos).Magnitude
                                            if (not bd or d < bd) then bd, best = d, pet end
                                        end
                                    end
                                end
                            end
                        else
                            local nc = {}
                            for _, p in ipairs(pets) do if not p.conveyor then nc[#nc + 1] = p end end
                            best = _pickByMode(nc)
                        end
                    end
                    if best then
                        if not _samePet(best, _stealTarget) then armSteal(best) end
                    elseif _stealTarget and not _stealHoldActive then
                        disarmSteal()
                    end
                end
            end
        end

        local pet = _stealTarget
        if not pet then return end
        local STEAL_MODE = (Config and Config.StealMode) or "Priority"
        if STEAL_MODE == "Nearest" and now - _stealArmedAt > STEAL_ARM_TIMEOUT then
            disarmSteal(); return
        end
        if now - _stealLastScan < 0.067 then return end
        _stealLastScan = now
        local t = timeUntilCanSteal()
        if t == -1 then
            if LP:GetAttribute("Stealing") then disarmSteal() end
            return
        end
        if t > 0 and t > STEAL_HOLD_DURATION then return end
        local char = LP.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        local prompt = findStealPrompt(pet)
        if not prompt or not prompt.Parent then return end
        local pp = prompt.Parent
        local ppPos = (pp and pp:IsA("BasePart") and pp.Position)
            or (pp and pp.Parent and pp.Parent:IsA("BasePart") and pp.Parent.Position)
        local _inCloneTP = isTeleporting and _cloneTP and _cloneFired
        if not _inCloneTP and ppPos and (hrp.Position - ppPos).Magnitude > STEAL_PROXIMITY then return end
        local oldMax
        pcall(function() oldMax = prompt.MaxActivationDistance end)
        pcall(function() prompt.MaxActivationDistance = math.huge end)
        buildStealCallbacks(prompt)
        if InternalStealCache[prompt] then executeStealAsync(prompt) end
        pcall(function() if oldMax ~= nil then prompt.MaxActivationDistance = oldMax end end)
    end)

    -- ===== Main TP function (1:1 xentp.lua doVelocityTP) =====
    local function doVelocityTP()
        if isTeleporting and (os.clock() - _tpStartedAt) < 30 then return end
        isTeleporting = true
        _tpStartedAt = os.clock()
        _cloneTP = false
        _cloneFired = false
        clearViz()
        if not NetModule then pcall(loadNet) end

        local char = LP.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then isTeleporting = false; return end

        local allPets = scanAllPets()
        if #allPets == 0 then
            local _t0 = os.clock()
            while #allPets == 0 and os.clock() - _t0 < 4 do
                task.wait(0.15)
                allPets = scanAllPets()
            end
        end
        if #allPets == 0 then isTeleporting = false; return end

        -- Respect SXEHub manual selection, then fall back to priority/mode
        local pet
        if manuallySelectedUID then
            for _, p in ipairs(allPets) do
                if p.plot and p.slot and (p.plot .. "_" .. tostring(p.slot)) == manuallySelectedUID then pet = p; break end
            end
        end
        if not pet then pet = _pickByMode(allPets) or allPets[1] end

        local _tpSpd = (Config and Config.TpSettings and Config.TpSettings.GrabbleTPSpeed) or 400
        local _cloneDelay = (Config and Config.TpSettings and Config.TpSettings.CloneDelayVal) or 0.35

        local petPos = pet.position
        local petName = pet.name

        local adjY = petPos.Y
        if TALL_PETS[petName] then adjY = petPos.Y - TALL_OFFSET end
        local coordTable = adjY > UPPER_Y_THRESHOLD and UPPER or LOWER

        -- CONVEYOR
        if pet.conveyor then
            local model = pet.model
            local maxHP = hum.MaxHealth
            hum.Health = maxHP
            local healConn = RunService.Heartbeat:Connect(function()
                if hum and hum.Parent then hum.Health = maxHP end
            end)
            carpetEngage()
            vZero(hrp)
            local function livePos()
                if not model or not model.Parent then return nil end
                local part = model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart")
                return part and part.Position or nil
            end
            local _t0 = os.clock()
            while os.clock() - _t0 < 8 do
                if not hrp or not hrp.Parent then break end
                local lp = livePos()
                if not lp then break end
                local diff = lp - hrp.Position
                if diff.Magnitude <= 6 then break end
                equipCarpet()
                hrp.AssemblyLinearVelocity = diff.Unit * _tpSpd
                RunService.Heartbeat:Wait()
            end
            if hrp and hrp.Parent then
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
            end
            healConn:Disconnect()
            isTeleporting = false
            return
        end

        -- FIRST FLOOR
        if petPos.Y <= 8.9 and isPlotUnlocked(pet.plot) then
            local maxHP = hum.MaxHealth
            hum.Health = maxHP
            local healConn = RunService.Heartbeat:Connect(function()
                if hum and hum.Parent then hum.Health = maxHP end
            end)
            carpetEngage()
            vZero(hrp)
            local _to = Vector3.new(petPos.X, -4, petPos.Z)
            local _faceDir
            do
                local idx = getClosestBaseIdx(petPos)
                local _, frontFace = buildFrontCandidate(idx, false, hrp.Position.Z)
                _faceDir = frontFace
            end
            local route = computeRoute(hrp.Position, _to, _faceDir)
            if not route or #route == 0 then route = { _to } end
            velMoveThrough(hrp, route, _tpSpd, true, true)
            if hrp and hrp.Parent then
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
            end
            healConn:Disconnect()
            isTeleporting = false
            return
        end

        -- SKY CLONE-TP
        _cloneTP = true
        disarmSteal()

        local closestData, skyKey = findClosest(petPos, coordTable)
        if not closestData or not skyKey then isTeleporting = false; return end

        local destPos = closestData.coord
        local maxHP = hum.MaxHealth
        hum.Health = maxHP
        local healConn = RunService.Heartbeat:Connect(function()
            if hum and hum.Parent then hum.Health = maxHP end
        end)

        carpetEngage()
        vZero(hrp)

        local facingDir = closestData.facing == "NORTH" and Vector3.new(0, 0, -1) or Vector3.new(0, 0, 1)
        do
            local isUpper = (coordTable == UPPER)
            local idx = getClosestBaseIdx(petPos)
            local frontCoord, frontFace = buildFrontCandidate(idx, isUpper, hrp.Position.Z)
            local bestCoord, bestFace = frontCoord, frontFace
            local bestDist = (hrp.Position - frontCoord).Magnitude
            for _, d in ipairs(plotSides(coordTable, idx)) do
                local dd = (hrp.Position - d.coord).Magnitude
                if dd < bestDist then
                    bestDist = dd
                    bestCoord = d.coord
                    bestFace = d.facing == "NORTH" and Vector3.new(0, 0, -1) or Vector3.new(0, 0, 1)
                end
            end
            destPos = bestCoord
            facingDir = bestFace
        end

        if hrp and hrp.Parent then
            hrp.CFrame = CFrame.new(hrp.Position, hrp.Position + facingDir)
            hrp.AssemblyAngularVelocity = Vector3.zero
        end

        local _route = computeRoute(hrp.Position, destPos, facingDir)
        local _stepped = {}
        do
            local startY = hrp.Position.Y
            local destY = destPos.Y
            local prev = hrp.Position
            local totalFlat = 0
            for _, wp in ipairs(_route) do
                totalFlat = totalFlat + (Vector3.new(wp.X, 0, wp.Z) - Vector3.new(prev.X, 0, prev.Z)).Magnitude
                prev = wp
            end
            if totalFlat < 0.01 then totalFlat = 0.01 end
            local SEG = 30
            prev = hrp.Position
            local travelled = 0
            for _, wp in ipairs(_route) do
                local flatVec = Vector3.new(wp.X, 0, wp.Z) - Vector3.new(prev.X, 0, prev.Z)
                local legFlat = flatVec.Magnitude
                if legFlat >= 0.01 then
                    local subs = math.max(1, math.ceil(legFlat / SEG))
                    for s = 1, subs do
                        local f = s / subs
                        local px = prev.X + (wp.X - prev.X) * f
                        local pz = prev.Z + (wp.Z - prev.Z) * f
                        local along = travelled + legFlat * f
                        local rampY = startY + (destY - startY) * (along / totalFlat)
                        _stepped[#_stepped + 1] = Vector3.new(px, rampY, pz)
                    end
                else
                    _stepped[#_stepped + 1] = wp
                end
                travelled = travelled + legFlat
                prev = wp
            end
            if #_stepped > 0 then _stepped[#_stepped] = _route[#_route] end
        end
        velMoveThrough(hrp, _stepped, _tpSpd, true, true)

        hrp.CFrame = CFrame.new(destPos, destPos + facingDir)
        vZero(hrp)

        local syncFrames = 5
        local syncConn
        syncConn = RunService.Heartbeat:Connect(function()
            if not hrp or not hrp.Parent then syncConn:Disconnect(); return end
            syncFrames = syncFrames - 1
            hrp.CFrame = CFrame.new(destPos, destPos + facingDir)
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            if syncFrames <= 0 then syncConn:Disconnect() end
        end)

        for _ = 1, 20 do
            task.wait(0.05)
            if hum.FloorMaterial ~= Enum.Material.Air then break end
        end

        healConn:Disconnect()
        armSteal(pet)
        _cloneFired = true

        local _ahrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
        local _clonePos = (_ahrp and _ahrp.Parent and _ahrp.Position) or destPos

        local _clonePlat = Instance.new("Part")
        _clonePlat.Name = "SXEClonePlatform"
        _clonePlat.Size = Vector3.new(12, 1, 12)
        _clonePlat.Position = Vector3.new(_clonePos.X, _clonePos.Y - 3, _clonePos.Z)
        _clonePlat.Anchored = true; _clonePlat.CanCollide = false; pcall(makeOneWay, _clonePlat); _clonePlat.Transparency = 1
        _clonePlat.Material = Enum.Material.SmoothPlastic; _clonePlat.Parent = workspace

        if _ahrp and _ahrp.Parent then
            _ahrp.AssemblyLinearVelocity = Vector3.zero
            _ahrp.AssemblyAngularVelocity = Vector3.zero
            pcall(function() _ahrp.Anchored = true end)
            task.delay(1, function()
                if _ahrp and _ahrp.Parent then pcall(function() _ahrp.Anchored = false end) end
            end)
        end

        local _charAdded = false
        local _caConn = LP.CharacterAdded:Connect(function() _charAdded = true end)

        task.wait(_cloneDelay)
        local _cloneOk = doClone()

        if _clonePlat then pcall(function() _clonePlat:Destroy() end); _clonePlat = nil end
        if _cloneOk then
            task.wait(0.3)
            local _cloneSucceeded = false
            do
                local _c = LP.Character
                local _h = _c and _c:FindFirstChild("HumanoidRootPart")
                local plotsFolder = workspace:FindFirstChild("Plots")
                if _h and plotsFolder then
                    local _rad = (petPos.Y <= 8.9) and 26 or 25
                    local p = _h.Position
                    for _, plot in ipairs(plotsFolder:GetChildren()) do
                        pcall(function()
                            local pp = plot:GetPivot().Position
                            if math.abs(p.X - pp.X) < _rad and math.abs(p.Z - pp.Z) < _rad then
                                _cloneSucceeded = true
                            end
                        end)
                        if _cloneSucceeded then break end
                    end
                end
            end
            if _caConn then _caConn:Disconnect() end
            if _cloneSucceeded then goToBrainrot(petPos) end
        else
            if _caConn then _caConn:Disconnect() end
        end
        isTeleporting = false
    end

    doGrabbleVelocityTP = doVelocityTP
    _G.SXEStartSideTP = doVelocityTP

    _G.ALPHA_ExecuteManualTP = function()
        task.spawn(function() pcall(doVelocityTP) end)
    end

    UIS.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        if input.KeyCode == Enum.KeyCode.T then
            task.spawn(function() pcall(doVelocityTP) end)
        end
    end)

    -- Boot
    task.spawn(function() pcall(loadModules) pcall(loadNet) end)
    task.spawn(function()
        local char = LP.Character or LP.CharacterAdded:Wait()
        char:WaitForChild("HumanoidRootPart", 10)
        char:WaitForChild("Humanoid", 10)
        pcall(loadModules); pcall(loadNet)
        local _t0 = os.clock()
        repeat
            local ok, pets = pcall(scanAllPets)
            if ok and pets and #pets > 0 then break end
            task.wait(0.3)
        until os.clock() - _t0 > 12
        _started = true
    end)
end
