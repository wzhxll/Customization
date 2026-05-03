-- 加载 WindUI 一次
local WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"))()

-- 本地化配置
WindUI:Localization({
    Enabled = true,
    Prefix = "loc:",
    DefaultLanguage = "zh-cn",
    Translations = {
        ["ru"] = {
            ["WINDUI_EXAMPLE"] = "WindUI Пример",
            ["WELCOME"] = "Добро пожаловать в WindUI!",
            ["LIB_DESC"] = "Библиотека для создания красивых интерфейсов",
            ["SETTINGS"] = "Настройки",
            ["APPEARANCE"] = "Внешний вид",
            ["FEATURES"] = "Функционал",
            ["UTILITIES"] = "Инструменты",
            ["UI_ELEMENTS"] = "UI Элементы",
            ["CONFIGURATION"] = "Конфигурация",
            ["SAVE_CONFIG"] = "Сохранить конфигурацию",
            ["LOAD_CONFIG"] = "Загрузить конфигурацию",
            ["THEME_SELECT"] = "Выберите тему",
            ["TRANSPARENCY"] = "Прозрачность окна"
        },
        ["en"] = {
            ["WINDUI_EXAMPLE"] = "WindUI Example",
            ["WELCOME"] = "Welcome to WindUI!",
            ["LIB_DESC"] = "Beautiful UI library for Roblox",
            ["SETTINGS"] = "Settings",
            ["APPEARANCE"] = "Appearance",
            ["FEATURES"] = "Features",
            ["UTILITIES"] = "Utilities",
            ["UI_ELEMENTS"] = "UI Elements",
            ["CONFIGURATION"] = "Configuration",
            ["SAVE_CONFIG"] = "Save Configuration",
            ["LOAD_CONFIG"] = "Load Configuration",
            ["THEME_SELECT"] = "Select Theme",
            ["TRANSPARENCY"] = "Window Transparency"
        },
        ["zh-cn"] = {
            ["WINDUI_EXAMPLE"] = "WindUI 示例",
            ["WELCOME"] = "欢迎使用 WindUI！",
            ["LIB_DESC"] = "为 Roblox 设计的精美 UI 库",
            ["SETTINGS"] = "设置",
            ["APPEARANCE"] = "外观",
            ["FEATURES"] = "功能",
            ["UTILITIES"] = "工具",
            ["UI_ELEMENTS"] = "UI 元素",
            ["CONFIGURATION"] = "配置",
            ["SAVE_CONFIG"] = "保存配置",
            ["LOAD_CONFIG"] = "加载配置",
            ["THEME_SELECT"] = "选择主题",
            ["TRANSPARENCY"] = "窗口透明度"
        }
    }
})

WindUI.TransparencyValue = 0.2
WindUI:SetTheme("Indigo")

-- ==================== 初始化服务 ====================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local lp = Players.LocalPlayer
local camera = workspace.CurrentCamera
local pgui = lp:WaitForChild("PlayerGui")
local ControlModule = require(lp.PlayerScripts:WaitForChild("PlayerModule")):GetControls()

-- ==================== 飞行功能（整合为一套） ====================
local bv, bg, animCache
local hrp, hum
local isFlying = false
local flySpeed = 40
local isWallhack = false
local flyTurner = nil
local originalCollisions = {}

local function getBodyParts(character)
    local parts = {}
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        local success, rigParts = pcall(function() return humanoid:GetRigParts() end)
        if success and rigParts then
            for _, part in ipairs(rigParts) do
                if part:IsA("BasePart") then
                    table.insert(parts, part)
                end
            end
        end
    end
    if #parts == 0 then
        local bodyNames = {"Head", "Torso", "UpperTorso", "LowerTorso", "HumanoidRootPart",
                           "Left Arm", "Right Arm", "Left Leg", "Right Leg",
                           "LeftUpperArm", "LeftLowerArm", "RightUpperArm", "RightLowerArm",
                           "LeftUpperLeg", "LeftLowerLeg", "RightUpperLeg", "RightLowerLeg"}
        for _, name in ipairs(bodyNames) do
            local part = character:FindFirstChild(name)
            if part and part:IsA("BasePart") then
                table.insert(parts, part)
            end
        end
    end
    return parts
end

local SmoothTurner = {}
SmoothTurner.__index = SmoothTurner

function SmoothTurner.new(rootPart, camera, options)
    options = options or {}
    local self = setmetatable({}, SmoothTurner)
    self.RootPart = rootPart
    self.Camera = camera or workspace.CurrentCamera
    self.Enabled = false
    self.BodyGyro = nil
    self.P = options.P or 10000
    self.D = options.D or 50
    self.MaxTorque = options.MaxTorque or Vector3.new(math.huge, math.huge, math.huge)
    return self
end

function SmoothTurner:Start()
    if self.Enabled then return end
    if not self.RootPart or not self.RootPart.Parent then return end
    local gyro = Instance.new("BodyGyro")
    gyro.MaxTorque = self.MaxTorque
    gyro.P = self.P
    gyro.D = self.D
    gyro.CFrame = self.RootPart.CFrame
    gyro.Parent = self.RootPart
    self.BodyGyro = gyro
    self.Enabled = true
    self:_startHeartbeat()
end

function SmoothTurner:Stop()
    if self.BodyGyro then
        self.BodyGyro:Destroy()
        self.BodyGyro = nil
    end
    self.Enabled = false
    if self.HeartbeatConn then
        self.HeartbeatConn:Disconnect()
        self.HeartbeatConn = nil
    end
end

function SmoothTurner:SetDirection(direction)
    if not self.Enabled or not self.BodyGyro or not self.RootPart then return end
    local newCFrame = CFrame.lookAt(self.RootPart.Position, self.RootPart.Position + direction.Unit)
    self.BodyGyro.CFrame = newCFrame
end

function SmoothTurner:_startHeartbeat()
    if self.HeartbeatConn then self.HeartbeatConn:Disconnect() end
    self.HeartbeatConn = RunService.Heartbeat:Connect(function()
        if not self.Enabled or not self.BodyGyro or not self.RootPart or not self.Camera then return end
        local look = self.Camera.CFrame.LookVector
        self:SetDirection(look)
    end)
end

function SmoothTurner:Destroy()
    self:Stop()
    self.RootPart = nil
    self.Camera = nil
end

local function clearFlyRes()
    if animCache and lp.Character then animCache.Parent = lp.Character end
    if bv then bv:Destroy() end
    if bg then bg:Destroy() end
    bv, bg = nil, nil
    if flyTurner then flyTurner:Destroy(); flyTurner = nil end
    if hum and hum.Parent then hum:ChangeState(Enum.HumanoidStateType.Running) end
    -- 恢复碰撞
    if lp.Character then
        local bodyParts = getBodyParts(lp.Character)
        for part, originalState in pairs(originalCollisions) do
            if part and part.Parent then
                for _, bp in ipairs(bodyParts) do
                    if bp == part then
                        part.CanCollide = originalState
                        break
                    end
                end
            end
        end
        originalCollisions = {}
    end
end

local function ensurePhysics(hrp, useGyro)
    if hrp:FindFirstChild("LeipzigBV") then hrp.LeipzigBV:Destroy() end
    if hrp:FindFirstChild("LeipzigBG") then hrp.LeipzigBG:Destroy() end
    bv = Instance.new("BodyVelocity", hrp)
    bv.Name = "LeipzigBV"
    bv.MaxForce = Vector3.new(1e6, 1e6, 1e6)
    if useGyro then
        if flyTurner then flyTurner:Destroy() end
        flyTurner = SmoothTurner.new(hrp, workspace.CurrentCamera)
        flyTurner:Start()
    end
end

local function applyWallhackState()
    local char = lp.Character
    if not char then return end
    if isWallhack then
        local bodyParts = getBodyParts(char)
        originalCollisions = {}
        for _, part in ipairs(bodyParts) do
            originalCollisions[part] = part.CanCollide
            part.CanCollide = false
        end
    else
        for part, originalState in pairs(originalCollisions) do
            if part and part.Parent then
                part.CanCollide = originalState
            end
        end
        originalCollisions = {}
    end
end

local function startFlyNormal()
    local char = lp.Character
    if not char then return end
    hrp = char:WaitForChild("HumanoidRootPart")
    hum = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then animCache = ani; ani.Parent = nil end
    ensurePhysics(hrp, true)
    task.spawn(function()
        while isFlying and char.Parent do
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)
            if mv.Magnitude > 0 then
                bv.Velocity = dir.Unit * flySpeed
            else
                bv.Velocity = Vector3.new(0,0.01,0)
            end
            hum:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
        end
        clearFlyRes()
    end)
end

local function startFlyWallhack()
    local char = lp.Character
    if not char then return end
    hrp = char:WaitForChild("HumanoidRootPart")
    hum = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then animCache = ani; ani.Parent = nil end
    applyWallhackState()
    ensurePhysics(hrp, true)
    task.spawn(function()
        local lastPos = hrp.Position
        local lastTime = tick()
        while isFlying and char.Parent do
            local dt = tick() - lastTime
            lastTime = tick()
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)
            local targetVelocity
            if mv.Magnitude > 0 then
                targetVelocity = dir.Unit * flySpeed
                bv.Velocity = targetVelocity
            else
                bv.Velocity = Vector3.new(0,0.01,0)
                targetVelocity = Vector3.new(0,0.01,0)
            end
            hum:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
            local expectedPos = lastPos + targetVelocity * dt
            local actualPos = hrp.Position
            local deviation = actualPos - expectedPos
            if deviation.Magnitude > 0.00001 then
                hrp.CFrame = CFrame.new(expectedPos) * hrp.CFrame.Rotation
                bv.Velocity = targetVelocity
                lastPos = expectedPos
            else
                lastPos = actualPos
            end
        end
        clearFlyRes()
    end)
end

local function startFly()
    if isFlying then return end
    isFlying = true
    if isWallhack then
        startFlyWallhack()
    else
        startFlyNormal()
    end
end

local function stopFly()
    if not isFlying then return end
    isFlying = false
    clearFlyRes()
end

local function bindCharacter()
    local char = lp.Character or lp.CharacterAdded:Wait()
    hrp = char:WaitForChild("HumanoidRootPart")
    hum = char:WaitForChild("Humanoid")
    clearFlyRes()
    char.AncestryChanged:Connect(function(_, parent)
        if not parent then
            clearFlyRes()
            bindCharacter()
        end
    end)
end
bindCharacter()

-- ==================== UI 初始化 ====================
local function gradient(text, startColor, endColor)
    local result = ""
    for i = 1, #text do
        local t = (i - 1) / (#text - 1)
        local r = math.floor((startColor.R + (endColor.R - startColor.R) * t) * 255)
        local g = math.floor((startColor.G + (endColor.G - startColor.G) * t) * 255)
        local b = math.floor((startColor.B + (endColor.B - startColor.B) * t) * 255)
        result = result .. string.format('<font color="rgb(%d,%d,%d)">%s</font>', r, g, b, text:sub(i, i))
    end
    return result
end

WindUI:Popup({
    Title = gradient("WindUI 演示", Color3.fromHex("#6A11CB"), Color3.fromHex("#2575FC")),
    Icon = "sparkles",
    Content = "loc:LIB_DESC",
    Buttons = {
        {
            Title = "开始使用",
            Icon = "arrow-right",
            Variant = "Primary",
            Callback = function() end
        }
    }
})

local Window = WindUI:CreateWindow({
    Title = "定制版",
    Icon = "palette",
    Author = "定制",
    Folder = "WindUI_Example",
    Size = UDim2.fromOffset(650, 450),
    Theme = "Indigo",
    User = {
        Enabled = true,
        Anonymous = true,
        Callback = function()
            WindUI:Notify({
                Title = "用户资料",
                Content = "用户头像被点击！",
                Duration = 3
            })
        end
    },
    SideBarWidth = 220,
    ScrollBarEnabled = true
})

Window:Tag({
    Title = "v1.6.4",
    Color = Color3.fromHex("#30ff6a")
})

Window:CreateTopbarButton("theme-switcher", "moon", function()
    WindUI:SetTheme(WindUI:GetCurrentTheme() == "Indigo" and "Dark" or "Indigo")
    WindUI:Notify({
        Title = "主题已更改",
        Content = "当前主题："..WindUI:GetCurrentTheme(),
        Duration = 2
    })
end, 990)

-- ==================== 杀戮光环功能 ====================
local KillSection = Window:Section({ Title = "杀戮光环", Opened = true })
local KillTab = KillSection:Tab({ Title = "高频杀戮", Icon = "sword" })

local zombieTypeNames = {
    Barrel   = "自爆",
    Axe      = "斧头僵尸",
    Eye      = "红眼",
    Sword    = "胸甲骑兵",
    FTorso   = "提灯人",
    Normal   = "山伯乐"
}
local defaultSelected = {}
for typeKey, name in pairs(zombieTypeNames) do
    if typeKey ~= "Barrel" then
        table.insert(defaultSelected, name)
    end
end

local selectedZombieTypes = {}
for _, label in ipairs(defaultSelected) do
    for k, v in pairs(zombieTypeNames) do
        if v == label then
            selectedZombieTypes[k] = true
            break
        end
    end
end

KillTab:Dropdown({
    Title = "攻击僵尸类型",
    Desc = "选择攻击的僵尸类型",
    Values = {"自爆", "斧头僵尸", "红眼", "胸甲骑兵", "提灯人", "山伯乐"},
    Multi = true,
    Default = defaultSelected,
    Callback = function(selected)
        selectedZombieTypes = {}
        for _, label in ipairs(selected) do
            for typeKey, displayName in pairs(zombieTypeNames) do
                if displayName == label then
                    selectedZombieTypes[typeKey] = true
                    break
                end
            end
        end
    end
})

KillTab:Divider()

local function isBarrelZombie(zombie)
    if zombie:FindFirstChild("Barrel") then return true end
    if zombie:GetAttribute("Type") == "Barrel" then return true end
    return false
end

local function getZombieTypeKey(zombie)
    if zombie:FindFirstChild("Axe") then return "Axe"
    elseif zombie:FindFirstChild("Eye") then return "Eye"
    elseif zombie:FindFirstChild("Sword") then return "Sword"
    elseif zombie:FindFirstChild("FTorso") then return "FTorso"
    else return "Normal" end
end

local function isZombieAttackAllowed(zombie)
    if isBarrelZombie(zombie) then
        return selectedZombieTypes["Barrel"] == true
    else
        local typeKey = getZombieTypeKey(zombie)
        return selectedZombieTypes[typeKey] == true
    end
end

local function getHeldMelee()
    local char = lp.Character
    if not char then return nil end
    for _, item in pairs(char:GetChildren()) do
        if item:IsA("Tool") and item:GetAttribute("Melee") then
            return item
        end
    end
    return nil
end

local auraEnabled = false
local attackThread = nil
local attackCount = 2
local displayRange = 45

local function getActualRange()
    return displayRange * (25 / 45)
end

local function attackLoop()
    while auraEnabled do
        local weapon = getHeldMelee()
        if weapon then
            local char = lp.Character
            if char then
                local myRoot = char:FindFirstChild("HumanoidRootPart")
                if myRoot then
                    local range = getActualRange()
                    local zombies = {}
                    local folder = workspace:FindFirstChild("Zombies")
                    if folder then
                        for _, z in pairs(folder:GetChildren()) do
                            if z:IsA("Model") and z:FindFirstChild("HumanoidRootPart") then
                                if not isZombieAttackAllowed(z) then continue end
                                if z:FindFirstChild("State") and z.State.Value == "Spawn" then continue end
                                local dist = (z.HumanoidRootPart.Position - myRoot.Position).Magnitude
                                if dist <= range then
                                    table.insert(zombies, {zombie = z, dist = dist})
                                end
                            end
                        end
                    end
                    table.sort(zombies, function(a,b) return a.dist < b.dist end)
                    local toAttack = math.min(attackCount, #zombies)
                    for i = 1, toAttack do
                        local remote = weapon:FindFirstChild("RemoteEvent")
                        if remote then
                            local head = zombies[i].zombie:FindFirstChild("Head")
                            if head then
                                remote:FireServer("Swing", "Side")
                                remote:FireServer("HitZombie", zombies[i].zombie, head.Position, true)
                            end
                        end
                    end
                end
            end
        end
        task.wait(0.05)
    end
end

local function startAura()
    if attackThread then return end
    auraEnabled = true
    attackThread = task.spawn(attackLoop)
end

local function stopAura()
    auraEnabled = false
    if attackThread then
        task.cancel(attackThread)
        attackThread = nil
    end
end

lp.CharacterAdded:Connect(function()
    if auraEnabled then
        task.wait(0.5)
        stopAura()
        task.wait(0.1)
        startAura()
    end
end)

KillTab:Toggle({
    Title = "启用杀戮光环",
    Desc = "自动攻击范围内近战僵尸",
    Value = false,
    Callback = function(state)
        if state then startAura() else stopAura() end
    end
})

KillTab:Slider({
    Title = "攻击距离",
    Desc = "杀戮光环攻击距离",
    Value = { Min = 10, Max = 45, Default = 45 },
    Callback = function(v) displayRange = v end
})

KillTab:Slider({
    Title = "攻击数量",
    Desc = "攻击僵尸数量",
    Value = { Min = 1, Max = 5, Default = 2 },
    Callback = function(v) attackCount = v end
})

KillTab:Divider()

-- ==================== 僵尸透视功能 ====================
local ZombieESPSection = Window:Section({ Title = "僵尸透视", Opened = true })
local ZombieESPTab = ZombieESPSection:Tab({ Title = "僵尸高亮", Icon = "eye" })

local zombieEspEnabled = { Axe = false, Eye = false, Sword = false, Barrel = false, FTorso = false, Normal = false }
local zombieEffects = {}
local ZOMBIE_ESP_RANGE = 200

local function lightenColor(color, factor)
    factor = factor or 0.5
    return Color3.new(
        color.R + (1 - color.R) * factor,
        color.G + (1 - color.G) * factor,
        color.B + (1 - color.B) * factor
    )
end

local ZOMBIE_TYPES = {
    Axe    = { name = "斧头僵尸", color = Color3.fromRGB(180, 0, 250), highlightColor = lightenColor(Color3.fromRGB(180, 0, 250)), part = "Axe" },
    Eye    = { name = "红眼",      color = Color3.fromRGB(255, 50, 50),  highlightColor = lightenColor(Color3.fromRGB(255, 50, 50)),  part = "Eye" },
    Sword  = { name = "胸甲骑兵",  color = Color3.fromRGB(255, 0, 255),  highlightColor = lightenColor(Color3.fromRGB(255, 0, 255)),  part = "Sword" },
    Barrel = { name = "自爆",      color = Color3.fromRGB(250, 250, 0),  highlightColor = lightenColor(Color3.fromRGB(250, 250, 0)),  part = "Barrel" },
    FTorso = { name = "提灯人",    color = Color3.fromRGB(255, 120, 0),  highlightColor = lightenColor(Color3.fromRGB(255, 120, 0)),  part = "FTorso" },
    Normal = { name = "山伯乐",    color = Color3.fromRGB(144, 238, 144), highlightColor = Color3.fromRGB(144, 238, 144), part = nil }
}

local function getZombieTypeKey(zombie)
    for typeKey, config in pairs(ZOMBIE_TYPES) do
        if config.part and zombie:FindFirstChild(config.part) then
            return typeKey
        end
    end
    return "Normal"
end

local function createTag(zombie, typeKey)
    local config = ZOMBIE_TYPES[typeKey]
    if not config then return nil end
    local attachPart = zombie.PrimaryPart or zombie:FindFirstChild("Head") or zombie:FindFirstChild("HumanoidRootPart")
    if not attachPart then return nil end
    local tag = Instance.new("BillboardGui")
    tag.Size = UDim2.new(0, 120, 0, 30)
    tag.StudsOffset = Vector3.new(0, 2.5, 0)
    tag.AlwaysOnTop = true
    tag.Adornee = attachPart
    tag.Parent = zombie
    local label = Instance.new("TextLabel")
    label.Text = config.name
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.TextColor3 = config.highlightColor
    label.TextTransparency = 0.3
    label.Font = Enum.Font.GothamBold
    label.TextSize = 14
    label.TextStrokeTransparency = 0.5
    label.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    label.Parent = tag
    return tag
end

local function createHighlight(zombie, color)
    local hl = Instance.new("Highlight")
    hl.FillColor = color
    hl.FillTransparency = 0.7
    hl.OutlineColor = color
    hl.OutlineTransparency = 0.7
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Adornee = zombie
    hl.Parent = zombie
    return hl
end

local function removeZombieEffects(zombie)
    local effects = zombieEffects[zombie]
    if effects then
        if effects.tag then effects.tag:Destroy() end
        if effects.highlight then effects.highlight:Destroy() end
        zombieEffects[zombie] = nil
    end
end

local function clearAllZombieEffects()
    for zombie, _ in pairs(zombieEffects) do
        removeZombieEffects(zombie)
    end
end

local function updateZombieESP()
    local anyEnabled = false
    for _, v in pairs(zombieEspEnabled) do
        if v then anyEnabled = true; break end
    end
    if not anyEnabled then
        clearAllZombieEffects()
        return
    end
    local char = lp.Character
    local rootPart = char and char:FindFirstChild("HumanoidRootPart")
    local playerPos = rootPart and rootPart.Position
    if not playerPos then
        clearAllZombieEffects()
        return
    end
    for zombie, _ in pairs(zombieEffects) do
        if not zombie.Parent then
            removeZombieEffects(zombie)
        end
    end
    local cameraFolder = workspace:FindFirstChild("Camera")
    if not cameraFolder then return end
    local zombies = cameraFolder:GetDescendants()
    for i = 1, #zombies do
        local zombie = zombies[i]
        if zombie:IsA("Model") and zombie.Name:find("Zombie") then
            local root = zombie:FindFirstChild("HumanoidRootPart") or zombie:FindFirstChild("Head") or zombie:FindFirstChild("Torso")
            if root then
                local dist = (root.Position - playerPos).Magnitude
                local typeKey = getZombieTypeKey(zombie)
                local enabled = zombieEspEnabled[typeKey]
                if enabled and dist <= ZOMBIE_ESP_RANGE then
                    if not zombieEffects[zombie] then
                        local config = ZOMBIE_TYPES[typeKey]
                        local tag = createTag(zombie, typeKey)
                        local highlight = createHighlight(zombie, config.highlightColor)
                        zombieEffects[zombie] = { tag = tag, highlight = highlight, typeKey = typeKey }
                    end
                else
                    if zombieEffects[zombie] then
                        removeZombieEffects(zombie)
                    end
                end
            end
        end
    end
end

local lastZombieESPUpdate = 0
local zombieESPHeartbeatConn = RunService.Heartbeat:Connect(function()
    local now = tick()
    if now - lastZombieESPUpdate >= 0.2 then
        lastZombieESPUpdate = now
        updateZombieESP()
    end
end)

lp.CharacterAdded:Connect(function()
    task.wait(0.5)
    updateZombieESP()
end)

-- 僵尸透视 UI
ZombieESPTab:Toggle({ Title = "透视斧头僵尸", Value = false, Callback = function(s) zombieEspEnabled.Axe = s; updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视红眼", Value = false, Callback = function(s) zombieEspEnabled.Eye = s; updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视胸甲骑兵", Value = false, Callback = function(s) zombieEspEnabled.Sword = s; updateZombieESP() end })

-- 冲锋提醒
local CuirassierChargeNotify = false
local LastChargeState = false
local chargeCheckThread = nil
local function CheckCuirassierCharge()
    local zFolder = workspace:FindFirstChild("Zombies")
    if not zFolder then return end
    local slim = zFolder:FindFirstChild("Slim")
    if not slim then return end
    local stateVal = slim:FindFirstChild("State")
    if not stateVal or not stateVal:IsA("StringValue") then return end
    local charging = (stateVal.Value == "BeginCharge" or stateVal.Value == "Charge")
    if charging and not LastChargeState and CuirassierChargeNotify then
        WindUI:Notify({ Title = "胸甲骑兵冲锋", Content = "胸甲骑兵冲锋中", Duration = 3, Icon = "bell" })
    end
    LastChargeState = charging
end
local function startChargeCheck()
    if chargeCheckThread then return end
    chargeCheckThread = task.spawn(function()
        while CuirassierChargeNotify do
            CheckCuirassierCharge()
            task.wait(0.5)
        end
        chargeCheckThread = nil
    end)
end
ZombieESPTab:Toggle({
    Title = "冲锋提醒",
    Value = false,
    Callback = function(state)
        CuirassierChargeNotify = state
        if state then
            LastChargeState = false
            startChargeCheck()
        else
            if chargeCheckThread then
                task.cancel(chargeCheckThread)
                chargeCheckThread = nil
            end
        end
    end
})
ZombieESPTab:Toggle({ Title = "透视自爆", Value = false, Callback = function(s) zombieEspEnabled.Barrel = s; updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视提灯人", Value = false, Callback = function(s) zombieEspEnabled.FTorso = s; updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视山伯乐", Value = false, Callback = function(s) zombieEspEnabled.Normal = s; updateZombieESP() end })

-- ==================== 玩家透视功能 ====================
local PlayerESPSection = Window:Section({ Title = "玩家透视", Opened = true })
local PlayerESPTab = PlayerESPSection:Tab({ Title = "玩家高亮", Icon = "users" })

local playerHighlights = {}
local playerNameTags = {}
local playerDots = {}
local espPlayerEnabled = false
local espShowNames = false
local espTeamCheckPlayer = false

local REFRESH_INTERVAL = 0.2
local MAX_DIST = 300

local function getPlayerTeam(player)
    if player.Team then return player.Team end
    local teamAttr = player:GetAttribute("Team")
    if teamAttr then return teamAttr end
    local char = player.Character
    if char then
        local teamTag = char:FindFirstChild("TeamTag") or char:FindFirstChild("Team")
        if teamTag then return teamTag.Value end
    end
    return nil
end

local function isSameTeam(player)
    if not espTeamCheckPlayer then return false end
    local myTeam = getPlayerTeam(lp)
    local theirTeam = getPlayerTeam(player)
    if myTeam and theirTeam then
        return myTeam == theirTeam
    end
    return false
end

local function getColorsForPlayer(player)
    if not espTeamCheckPlayer then
        return { highlight = Color3.fromRGB(255,255,255), dot = Color3.fromRGB(160,160,160), name = Color3.fromRGB(255,255,255) }
    end
    if isSameTeam(player) then
        return { highlight = Color3.fromRGB(100,150,255), dot = Color3.fromRGB(0,30,180), name = Color3.fromRGB(100,150,255) }
    else
        return { highlight = Color3.fromRGB(255,100,100), dot = Color3.fromRGB(180,0,0), name = Color3.fromRGB(255,100,100) }
    end
end

local function destroyPlayerComponents(player)
    if playerHighlights[player] then playerHighlights[player]:Destroy(); playerHighlights[player] = nil end
    if playerNameTags[player] then playerNameTags[player]:Destroy(); playerNameTags[player] = nil end
    if playerDots[player] then playerDots[player]:Destroy(); playerDots[player] = nil end
end

local function updatePlayerESP(player)
    if not espPlayerEnabled then
        destroyPlayerComponents(player)
        return
    end
    local char = player.Character
    if not char or char == lp.Character then
        destroyPlayerComponents(player)
        return
    end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local colors = getColorsForPlayer(player)
    local myChar = lp.Character
    local myPos = myChar and myChar:FindFirstChild("HumanoidRootPart") and myChar.HumanoidRootPart.Position or Vector3.new()
    local dist = (hrp.Position - myPos).Magnitude
    local near = dist <= MAX_DIST

    if near then
        if not playerHighlights[player] then
            local hl = Instance.new("Highlight")
            hl.Adornee = char
            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            hl.FillTransparency = 0.3
            hl.OutlineTransparency = 0.3
            hl.Parent = char
            playerHighlights[player] = hl
        end
        playerHighlights[player].FillColor = colors.highlight
        playerHighlights[player].OutlineColor = colors.highlight
    else
        if playerHighlights[player] then
            playerHighlights[player]:Destroy()
            playerHighlights[player] = nil
        end
    end

    if not playerDots[player] then
        local dotGui = Instance.new("BillboardGui")
        dotGui.Name = "PlayerDot"
        dotGui.Size = UDim2.new(0,5,0,5)
        dotGui.StudsOffset = Vector3.new(0,0,0)
        dotGui.AlwaysOnTop = true
        dotGui.Adornee = hrp
        dotGui.Parent = char
        local dotFrame = Instance.new("Frame")
        dotFrame.Size = UDim2.new(1,0,1,0)
        dotFrame.BackgroundColor3 = colors.dot
        dotFrame.BackgroundTransparency = 0
        dotFrame.BorderSizePixel = 0
        dotFrame.Parent = dotGui
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1,0)
        corner.Parent = dotFrame
        playerDots[player] = dotGui
    else
        local dotGui = playerDots[player]
        if dotGui.Adornee ~= hrp then dotGui.Adornee = hrp end
        local dotFrame = dotGui:FindFirstChildWhichIsA("Frame")
        if dotFrame then dotFrame.BackgroundColor3 = colors.dot end
        if not dotGui.Parent or not dotGui.Parent:IsDescendantOf(char) then dotGui.Parent = char end
    end

    if espShowNames then
        if not playerNameTags[player] then
            local nameGui = Instance.new("BillboardGui")
            nameGui.Name = "PlayerNameTag"
            nameGui.Size = UDim2.new(0,150,0,30)
            nameGui.StudsOffset = Vector3.new(0,-2.5,0)
            nameGui.AlwaysOnTop = true
            nameGui.Adornee = hrp
            nameGui.Parent = char
            local label = Instance.new("TextLabel")
            label.Size = UDim2.new(1,0,1,0)
            label.BackgroundTransparency = 1
            label.Text = player.Name
            label.TextColor3 = colors.name
            label.TextSize = 11
            label.Font = Enum.Font.GothamBold
            label.TextStrokeTransparency = 0
            label.TextStrokeColor3 = Color3.fromRGB(0,0,0)
            label.Parent = nameGui
            playerNameTags[player] = nameGui
        else
            local nameGui = playerNameTags[player]
            if nameGui.Adornee ~= hrp then nameGui.Adornee = hrp end
            if not nameGui.Parent or not nameGui.Parent:IsDescendantOf(char) then nameGui.Parent = char end
            local label = nameGui:FindFirstChildOfClass("TextLabel")
            if label then
                label.Text = player.Name
                label.TextColor3 = colors.name
            end
        end
    else
        if playerNameTags[player] then
            playerNameTags[player]:Destroy()
            playerNameTags[player] = nil
        end
    end
end

local function refreshAllPlayers()
    for _, player in ipairs(Players:GetPlayers()) do
        updatePlayerESP(player)
    end
end

local refreshThread = nil
local function startRefreshLoop()
    if refreshThread then return end
    refreshThread = task.spawn(function()
        while espPlayerEnabled do
            task.wait(REFRESH_INTERVAL)
            if espPlayerEnabled then
                for _, player in ipairs(Players:GetPlayers()) do
                    updatePlayerESP(player)
                end
            end
        end
        refreshThread = nil
    end)
end

local function stopRefreshLoop()
    if refreshThread then task.cancel(refreshThread); refreshThread = nil end
end

Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function()
        task.wait(0.2)
        if espPlayerEnabled then updatePlayerESP(player) end
    end)
    player.CharacterRemoving:Connect(function() destroyPlayerComponents(player) end)
    if espPlayerEnabled then updatePlayerESP(player) end
end)
Players.PlayerRemoving:Connect(function(player) destroyPlayerComponents(player) end)
lp.CharacterAdded:Connect(function()
    task.wait(0.5)
    if espPlayerEnabled then refreshAllPlayers() end
end)

PlayerESPTab:Toggle({
    Title = "启用玩家透视",
    Desc = "高亮显示其他玩家",
    Value = false,
    Callback = function(state)
        espPlayerEnabled = state
        if state then
            refreshAllPlayers()
            startRefreshLoop()
        else
            stopRefreshLoop()
            for _, player in ipairs(Players:GetPlayers()) do
                destroyPlayerComponents(player)
            end
        end
    end
})
PlayerESPTab:Toggle({
    Title = "显示玩家名称",
    Desc = "在玩家头顶显示名字",
    Value = false,
    Callback = function(state)
        espShowNames = state
        refreshAllPlayers()
    end
})
PlayerESPTab:Toggle({
    Title = "队伍检测",
    Desc = "高亮区分 红色敌方 蓝色友方",
    Value = false,
    Callback = function(state)
        espTeamCheckPlayer = state
        refreshAllPlayers()
    end
})

-- ==================== 清理函数 ====================
Window:OnDestroy(function()
    print("窗口已销毁")
    if isFlying then stopFly() end
    if auraEnabled then stopAura() end
    if zombieESPHeartbeatConn then zombieESPHeartbeatConn:Disconnect() end
    clearAllZombieEffects()
    stopRefreshLoop()
    for _, player in ipairs(Players:GetPlayers()) do
        destroyPlayerComponents(player)
    end
end)
