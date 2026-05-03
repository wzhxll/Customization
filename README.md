local WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"))()

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

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local lp = Players.LocalPlayer
local camera = workspace.CurrentCamera
local pgui = lp:WaitForChild("PlayerGui")

local bv_orig, bg_orig, animCache_orig
local hrp_orig, hum_orig
local isFlying_orig = false
local flySpeed_orig = 40
local ControlModule = require(lp.PlayerScripts:WaitForChild("PlayerModule")):GetControls()

local function clearFlyRes_orig()
    if animCache_orig and lp.Character then animCache_orig.Parent = lp.Character end
    if bv_orig then bv_orig:Destroy() end
    if bg_orig then bg_orig:Destroy() end
    bv_orig, bg_orig = nil, nil
    if hum_orig and hum_orig.Parent then hum_orig:ChangeState(Enum.HumanoidStateType.Running) end
end

local function ensurePhysics_orig(hrp, useGyro)
    if hrp:FindFirstChild("LeipzigBV_orig") then hrp.LeipzigBV_orig:Destroy() end
    if hrp:FindFirstChild("LeipzigBG_orig") then hrp.LeipzigBG_orig:Destroy() end
    bv_orig = Instance.new("BodyVelocity", hrp)
    bv_orig.Name = "LeipzigBV_orig"
    bv_orig.MaxForce = Vector3.new(1e6, 1e6, 1e6)
    if useGyro then
        bg_orig = Instance.new("BodyGyro", hrp)
        bg_orig.Name = "LeipzigBG_orig"
        bg_orig.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
        bg_orig.P = 10000
        bg_orig.D = 50
    end
end

local function startFlyNormal_orig()
    local char = lp.Character
    if not char then return end
    hrp_orig = char:WaitForChild("HumanoidRootPart")
    hum_orig = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then animCache_orig = ani; ani.Parent = nil end
    ensurePhysics_orig(hrp_orig, true)
    task.spawn(function()
        while isFlying_orig and char.Parent do
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)

            if mv.Magnitude > 0 then
                bv_orig.Velocity = dir.Unit * flySpeed_orig
                local lookDir = dir.Unit
                bg_orig.CFrame = CFrame.lookAt(hrp_orig.Position, hrp_orig.Position + lookDir)
            else
                bv_orig.Velocity = Vector3.new(0,0.01,0)
            end
            hum_orig:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
        end
        clearFlyRes_orig()
    end)
end

local function startFly_orig()
    if isFlying_orig then return end
    isFlying_orig = true
    startFlyNormal_orig()
end

local function stopFly_orig()
    if not isFlying_orig then return end
    isFlying_orig = false
    clearFlyRes_orig()
end

local function bindCharacter_orig()
    local char = lp.Character or lp.CharacterAdded:Wait()
    hrp_orig = char:WaitForChild("HumanoidRootPart")
    hum_orig = char:WaitForChild("Humanoid")
    clearFlyRes_orig()
    char.AncestryChanged:Connect(function(_, parent)
        if not parent then
            clearFlyRes_orig()
            bindCharacter_orig()
        end
    end)
end
bindCharacter_orig()

local bv_new, bg_new, animCache_new
local hrp_new, hum_new
local isFlying_new = false
local flySpeed_new = 40
local isWallhack_new = false
local flyTurner_new = nil
local originalCollisions_new = {}

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

local function clearFlyRes_new()
    local char = lp.Character
    if char then
        local bodyParts = getBodyParts(char)
        for part, originalState in pairs(originalCollisions_new) do
            if part and part.Parent then
                for _, bp in ipairs(bodyParts) do
                    if bp == part then
                        part.CanCollide = originalState
                        break
                    end
                end
            end
        end
        originalCollisions_new = {}
    end
    if animCache_new and lp.Character then animCache_new.Parent = lp.Character end
    if bv_new then bv_new:Destroy() end
    if bg_new then bg_new:Destroy() end
    bv_new, bg_new = nil, nil
    if flyTurner_new then flyTurner_new:Destroy(); flyTurner_new = nil end
    if hum_new and hum_new.Parent then hum_new:ChangeState(Enum.HumanoidStateType.Running) end
end

local function ensurePhysics_new(hrp, useGyro)
    if hrp:FindFirstChild("LeipzigBV_new") then hrp.LeipzigBV_new:Destroy() end
    if hrp:FindFirstChild("LeipzigBG_new") then hrp.LeipzigBG_new:Destroy() end
    bv_new = Instance.new("BodyVelocity", hrp)
    bv_new.Name = "LeipzigBV_new"
    bv_new.MaxForce = Vector3.new(1e6, 1e6, 1e6)
    if useGyro then
        if flyTurner_new then flyTurner_new:Destroy() end
        flyTurner_new = SmoothTurner.new(hrp, workspace.CurrentCamera)
        flyTurner_new:Start()
    end
end

local function applyWallhackState_new()
    local char = lp.Character
    if not char then return end
    if isWallhack_new then
        local bodyParts = getBodyParts(char)
        originalCollisions_new = {}
        for _, part in ipairs(bodyParts) do
            originalCollisions_new[part] = part.CanCollide
            part.CanCollide = false
        end
    else
        for part, originalState in pairs(originalCollisions_new) do
            if part and part.Parent then
                part.CanCollide = originalState
            end
        end
        originalCollisions_new = {}
    end
end

local function startFlyNormal_new()
    local char = lp.Character
    if not char then return end
    hrp_new = char:WaitForChild("HumanoidRootPart")
    hum_new = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then animCache_new = ani; ani.Parent = nil end
    ensurePhysics_new(hrp_new, true)
    task.spawn(function()
        while isFlying_new and char.Parent do
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)
            if mv.Magnitude > 0 then
                bv_new.Velocity = dir.Unit * flySpeed_new
            else
                bv_new.Velocity = Vector3.new(0,0.01,0)
            end
            hum_new:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
        end
        clearFlyRes_new()
    end)
end

local function startFlyWallhack_new()
    local char = lp.Character
    if not char then return end
    hrp_new = char:WaitForChild("HumanoidRootPart")
    hum_new = char:WaitForChild("Humanoid")
    local ani = char:FindFirstChild("Animate")
    if ani then animCache_new = ani; ani.Parent = nil end
    applyWallhackState_new()
    ensurePhysics_new(hrp_new, true)
    task.spawn(function()
        local lastPos = hrp_new.Position
        local lastTime = tick()
        while isFlying_new and char.Parent do
            local dt = tick() - lastTime
            lastTime = tick()
            local mv = ControlModule:GetMoveVector()
            local cf = camera.CFrame
            local dir = (cf.LookVector * -mv.Z) + (cf.RightVector * mv.X)
            local targetVelocity
            if mv.Magnitude > 0 then
                targetVelocity = dir.Unit * flySpeed_new
                bv_new.Velocity = targetVelocity
            else
                bv_new.Velocity = Vector3.new(0,0.01,0)
                targetVelocity = Vector3.new(0,0.01,0)
            end
            hum_new:ChangeState(Enum.HumanoidStateType.Climbing)
            RunService.RenderStepped:Wait()
            local expectedPos = lastPos + targetVelocity * dt
            local actualPos = hrp_new.Position
            local deviation = actualPos - expectedPos
            if deviation.Magnitude > 0.00001 then
                hrp_new.CFrame = CFrame.new(expectedPos) * hrp_new.CFrame.Rotation
                bv_new.Velocity = targetVelocity
                lastPos = expectedPos
            else
                lastPos = actualPos
            end
        end
        clearFlyRes_new()
    end)
end

local function startFly_new()
    if isFlying_new then return end
    isFlying_new = true
    if isWallhack_new then
        startFlyWallhack_new()
    else
        startFlyNormal_new()
    end
end

local function stopFly_new()
    if not isFlying_new then return end
    isFlying_new = false
    clearFlyRes_new()
end

local function bindCharacter_new()
    local char = lp.Character or lp.CharacterAdded:Wait()
    hrp_new = char:WaitForChild("HumanoidRootPart")
    hum_new = char:WaitForChild("Humanoid")
    clearFlyRes_new()
    char.AncestryChanged:Connect(function(_, parent)
        if not parent then
            clearFlyRes_new()
            bindCharacter_new()
        end
    end)
end
bindCharacter_new()

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

-- ==================== 杀戮光环功能卡 ====================
local KillSection = Window:Section({ Title = "杀戮光环", Opened = true })
local KillTab = KillSection:Tab({ Title = "高频杀戮", Icon = "sword" })

-- 全局数据表
local L = {}
L.auraEnabled = false
L.attackThread = nil
L.attackCount = 2
L.displayRange = 45
L.selectedZombieTypes = {}

-- 僵尸类型映射
local zombieTypeNames = {
    Barrel   = "自爆",
    Axe      = "斧头僵尸",
    Eye      = "红眼",
    Sword    = "胸甲骑兵",
    FTorso   = "提灯人",
    Normal   = "山伯乐"
}
-- 默认选中除自爆外的所有类型
local defaultSelected = {}
for typeKey, name in pairs(zombieTypeNames) do
    if typeKey ~= "Barrel" then
        table.insert(defaultSelected, name)
    end
end
for _, label in ipairs(defaultSelected) do
    for k, v in pairs(zombieTypeNames) do
        if v == label then
            L.selectedZombieTypes[k] = true
            break
        end
    end
end

-- 下拉框
KillTab:Dropdown({
    Title = "攻击僵尸类型",
    Desc = "选择要攻击的僵尸种类",
    Values = {"自爆", "斧头僵尸", "红眼", "胸甲骑兵", "提灯人", "山伯乐"},
    Multi = true,
    Default = defaultSelected,
    Callback = function(selected)
        L.selectedZombieTypes = {}
        for _, label in ipairs(selected) do
            for typeKey, displayName in pairs(zombieTypeNames) do
                if displayName == label then
                    L.selectedZombieTypes[typeKey] = true
                    break
                end
            end
        end
    end
})

KillTab:Divider()

-- 辅助函数
function L.isBarrelZombie(zombie)
    return zombie:FindFirstChild("Barrel") ~= nil or zombie:GetAttribute("Type") == "Barrel"
end
function L.getZombieTypeKey(zombie)
    if zombie:FindFirstChild("Axe") then return "Axe"
    elseif zombie:FindFirstChild("Eye") then return "Eye"
    elseif zombie:FindFirstChild("Sword") then return "Sword"
    elseif zombie:FindFirstChild("FTorso") then return "FTorso"
    else return "Normal" end
end
function L.isZombieAttackAllowed(zombie)
    if L.isBarrelZombie(zombie) then
        return L.selectedZombieTypes["Barrel"] == true
    else
        local typeKey = L.getZombieTypeKey(zombie)
        return L.selectedZombieTypes[typeKey] == true
    end
end
function L.getHeldMelee()
    local char = lp.Character
    if not char then return nil end
    for _, item in pairs(char:GetChildren()) do
        if item:IsA("Tool") and item:GetAttribute("Melee") then
            return item
        end
    end
    return nil
end
local function getActualRange()
    return L.displayRange * (25 / 45)
end

-- 攻击循环
function L.attackLoop()
    while L.auraEnabled do
        local weapon = L.getHeldMelee()
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
                                if not L.isZombieAttackAllowed(z) then continue end
                                if z:FindFirstChild("State") and z.State.Value == "Spawn" then continue end
                                local dist = (z.HumanoidRootPart.Position - myRoot.Position).Magnitude
                                if dist <= range then
                                    table.insert(zombies, {zombie = z, dist = dist})
                                end
                            end
                        end
                    end
                    table.sort(zombies, function(a,b) return a.dist < b.dist end)
                    local toAttack = math.min(L.attackCount, #zombies)
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
function L.startAura()
    if L.attackThread then return end
    L.auraEnabled = true
    L.attackThread = task.spawn(L.attackLoop)
end
function L.stopAura()
    L.auraEnabled = false
    if L.attackThread then
        task.cancel(L.attackThread)
        L.attackThread = nil
    end
end
lp.CharacterAdded:Connect(function()
    if L.auraEnabled then
        task.wait(0.5)
        L.stopAura()
        task.wait(0.1)
        L.startAura()
    end
end)

-- UI 控件
KillTab:Toggle({
    Title = "高频杀戮光环（防封）",
    Desc = "极致爽感，可自定义攻击僵尸类型",
    Value = false,
    Callback = function(state)
        if state then L.startAura() else L.stopAura() end
    end
})
KillTab:Slider({
    Title = "攻击距离",
    Desc = "杀戮光环有效距离",
    Value = { Min = 10, Max = 45, Default = 45 },
    Callback = function(v) L.displayRange = v end
})
KillTab:Slider({
    Title = "攻击数量",
    Desc = "每次攻击最多几个僵尸",
    Value = { Min = 1, Max = 5, Default = 2 },
    Callback = function(v) L.attackCount = v end
})

-- ==================== 僵尸透视功能卡 ====================
local ZombieESPSection = Window:Section({ Title = "僵尸透视", Opened = true })
local ZombieESPTab = ZombieESPSection:Tab({ Title = "僵尸高亮", Icon = "eye" })

L.zombieEspEnabled = { Axe = false, Eye = false, Sword = false, Barrel = false, FTorso = false, Normal = false }
L.zombieEffects = {}
L.ZOMBIE_ESP_RANGE = 200

function L.lightenColor(color, factor)
    factor = factor or 0.5
    return Color3.new(
        color.R + (1 - color.R) * factor,
        color.G + (1 - color.G) * factor,
        color.B + (1 - color.B) * factor
    )
end
L.ZOMBIE_TYPES = {
    Axe    = { name = "斧头僵尸", color = Color3.fromRGB(180, 0, 250), highlightColor = L.lightenColor(Color3.fromRGB(180, 0, 250)), part = "Axe" },
    Eye    = { name = "红眼",      color = Color3.fromRGB(255, 50, 50),  highlightColor = L.lightenColor(Color3.fromRGB(255, 50, 50)),  part = "Eye" },
    Sword  = { name = "胸甲骑兵",  color = Color3.fromRGB(255, 0, 255),  highlightColor = L.lightenColor(Color3.fromRGB(255, 0, 255)),  part = "Sword" },
    Barrel = { name = "自爆",      color = Color3.fromRGB(250, 250, 0),  highlightColor = L.lightenColor(Color3.fromRGB(250, 250, 0)),  part = "Barrel" },
    FTorso = { name = "提灯人",    color = Color3.fromRGB(255, 120, 0),  highlightColor = L.lightenColor(Color3.fromRGB(255, 120, 0)),  part = "FTorso" },
    Normal = { name = "山伯乐",    color = Color3.fromRGB(144, 238, 144), highlightColor = Color3.fromRGB(144, 238, 144), part = nil }
}
function L.getZombieTypeKey(zombie)
    for typeKey, config in pairs(L.ZOMBIE_TYPES) do
        if config.part and zombie:FindFirstChild(config.part) then
            return typeKey
        end
    end
    return "Normal"
end
function L.createTag(zombie, typeKey)
    local config = L.ZOMBIE_TYPES[typeKey]
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
function L.createHighlight(zombie, color)
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
function L.removeZombieEffects(zombie)
    local effects = L.zombieEffects[zombie]
    if effects then
        if effects.tag then effects.tag:Destroy() end
        if effects.highlight then effects.highlight:Destroy() end
        L.zombieEffects[zombie] = nil
    end
end
function L.clearAllZombieEffects()
    for zombie, _ in pairs(L.zombieEffects) do
        L.removeZombieEffects(zombie)
    end
end
function L.updateZombieESP()
    local anyEnabled = false
    for _, v in pairs(L.zombieEspEnabled) do
        if v then anyEnabled = true; break end
    end
    if not anyEnabled then
        L.clearAllZombieEffects()
        return
    end
    local char = lp.Character
    local rootPart = char and char:FindFirstChild("HumanoidRootPart")
    local playerPos = rootPart and rootPart.Position
    if not playerPos then
        L.clearAllZombieEffects()
        return
    end
    for zombie, _ in pairs(L.zombieEffects) do
        if not zombie.Parent then
            L.removeZombieEffects(zombie)
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
                local typeKey = L.getZombieTypeKey(zombie)
                local enabled = L.zombieEspEnabled[typeKey]
                if enabled and dist <= L.ZOMBIE_ESP_RANGE then
                    if not L.zombieEffects[zombie] then
                        local config = L.ZOMBIE_TYPES[typeKey]
                        local tag = L.createTag(zombie, typeKey)
                        local highlight = L.createHighlight(zombie, config.highlightColor)
                        L.zombieEffects[zombie] = { tag = tag, highlight = highlight, typeKey = typeKey }
                    end
                else
                    if L.zombieEffects[zombie] then
                        L.removeZombieEffects(zombie)
                    end
                end
            end
        end
    end
end
L.lastZombieESPUpdate = 0
L.zombieESPHeartbeatConn = RunService.Heartbeat:Connect(function()
    local now = tick()
    if now - L.lastZombieESPUpdate >= 0.2 then
        L.lastZombieESPUpdate = now
        L.updateZombieESP()
    end
end)
lp.CharacterAdded:Connect(function()
    task.wait(0.5)
    L.updateZombieESP()
end)

-- 僵尸透视 UI 控件
ZombieESPTab:Toggle({ Title = "透视斧头僵尸", Value = false, Callback = function(s) L.zombieEspEnabled.Axe = s; L.updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视红眼", Value = false, Callback = function(s) L.zombieEspEnabled.Eye = s; L.updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视胸甲骑兵", Value = false, Callback = function(s) L.zombieEspEnabled.Sword = s; L.updateZombieESP() end })
-- 冲锋提醒（独立）
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
ZombieESPTab:Toggle({ Title = "透视自爆", Value = false, Callback = function(s) L.zombieEspEnabled.Barrel = s; L.updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视提灯人", Value = false, Callback = function(s) L.zombieEspEnabled.FTorso = s; L.updateZombieESP() end })
ZombieESPTab:Toggle({ Title = "透视山伯乐", Value = false, Callback = function(s) L.zombieEspEnabled.Normal = s; L.updateZombieESP() end })

-- ==================== 玩家透视功能卡 ====================
local PlayerESPSection = Window:Section({ Title = "玩家透视", Opened = true })
local PlayerESPTab = PlayerESPSection:Tab({ Title = "玩家高亮", Icon = "users" })

L.playerHighlights = {}
L.playerNameTags = {}
L.playerDots = {}
L.espPlayerEnabled = false
L.espShowNames = false
L.espTeamCheckPlayer = false

local REFRESH_INTERVAL = 0.2
local MAX_DIST = 300

function L.getPlayerTeam(player)
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
function L.isSameTeam(player)
    if not L.espTeamCheckPlayer then return false end
    local myTeam = L.getPlayerTeam(lp)
    local theirTeam = L.getPlayerTeam(player)
    if myTeam and theirTeam then
        return myTeam == theirTeam
    end
    return false
end
function L.getColorsForPlayer(player)
    if not L.espTeamCheckPlayer then
        return { highlight = Color3.fromRGB(255,255,255), dot = Color3.fromRGB(160,160,160), name = Color3.fromRGB(255,255,255) }
    end
    if L.isSameTeam(player) then
        return { highlight = Color3.fromRGB(100,150,255), dot = Color3.fromRGB(0,30,180), name = Color3.fromRGB(100,150,255) }
    else
        return { highlight = Color3.fromRGB(255,100,100), dot = Color3.fromRGB(180,0,0), name = Color3.fromRGB(255,100,100) }
    end
end
function L.destroyPlayerComponents(player)
    if L.playerHighlights[player] then L.playerHighlights[player]:Destroy(); L.playerHighlights[player] = nil end
    if L.playerNameTags[player] then L.playerNameTags[player]:Destroy(); L.playerNameTags[player] = nil end
    if L.playerDots[player] then L.playerDots[player]:Destroy(); L.playerDots[player] = nil end
end
function L.updatePlayerESP(player)
    if not L.espPlayerEnabled then
        L.destroyPlayerComponents(player)
        return
    end
    local char = player.Character
    if not char or char == lp.Character then
        L.destroyPlayerComponents(player)
        return
    end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local colors = L.getColorsForPlayer(player)
    local myChar = lp.Character
    local myPos = myChar and myChar:FindFirstChild("HumanoidRootPart") and myChar.HumanoidRootPart.Position or Vector3.new()
    local dist = (hrp.Position - myPos).Magnitude
    local near = dist <= MAX_DIST

    if near then
        if not L.playerHighlights[player] then
            local hl = Instance.new("Highlight")
            hl.Adornee = char
            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            hl.FillTransparency = 0.3
            hl.OutlineTransparency = 0.3
            hl.Parent = char
            L.playerHighlights[player] = hl
        end
        L.playerHighlights[player].FillColor = colors.highlight
        L.playerHighlights[player].OutlineColor = colors.highlight
    else
        if L.playerHighlights[player] then
            L.playerHighlights[player]:Destroy()
            L.playerHighlights[player] = nil
        end
    end

    if not L.playerDots[player] then
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
        L.playerDots[player] = dotGui
    else
        local dotGui = L.playerDots[player]
        if dotGui.Adornee ~= hrp then dotGui.Adornee = hrp end
        local dotFrame = dotGui:FindFirstChildWhichIsA("Frame")
        if dotFrame then dotFrame.BackgroundColor3 = colors.dot end
        if not dotGui.Parent or not dotGui.Parent:IsDescendantOf(char) then dotGui.Parent = char end
    end

    if L.espShowNames then
        if not L.playerNameTags[player] then
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
            L.playerNameTags[player] = nameGui
        else
            local nameGui = L.playerNameTags[player]
            if nameGui.Adornee ~= hrp then nameGui.Adornee = hrp end
            if not nameGui.Parent or not nameGui.Parent:IsDescendantOf(char) then nameGui.Parent = char end
            local label = nameGui:FindFirstChildOfClass("TextLabel")
            if label then
                label.Text = player.Name
                label.TextColor3 = colors.name
            end
        end
    else
        if L.playerNameTags[player] then
            L.playerNameTags[player]:Destroy()
            L.playerNameTags[player] = nil
        end
    end
end
function L.refreshAllPlayers()
    for _, player in ipairs(Players:GetPlayers()) do
        L.updatePlayerESP(player)
    end
end
local refreshThread = nil
local function startRefreshLoop()
    if refreshThread then return end
    refreshThread = task.spawn(function()
        while L.espPlayerEnabled do
            task.wait(REFRESH_INTERVAL)
            if L.espPlayerEnabled then
                for _, player in ipairs(Players:GetPlayers()) do
                    L.updatePlayerESP(player)
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
        if L.espPlayerEnabled then L.updatePlayerESP(player) end
    end)
    player.CharacterRemoving:Connect(function() L.destroyPlayerComponents(player) end)
    if L.espPlayerEnabled then L.updatePlayerESP(player) end
end)
Players.PlayerRemoving:Connect(function(player) L.destroyPlayerComponents(player) end)
lp.CharacterAdded:Connect(function()
    task.wait(0.5)
    if L.espPlayerEnabled then L.refreshAllPlayers() end
end)

PlayerESPTab:Toggle({
    Title = "启用玩家透视",
    Desc = "高亮显示其他玩家",
    Value = false,
    Callback = function(state)
        L.espPlayerEnabled = state
        if state then
            L.refreshAllPlayers()
            startRefreshLoop()
        else
            stopRefreshLoop()
            for _, player in ipairs(Players:GetPlayers()) do
                L.destroyPlayerComponents(player)
            end
        end
    end
})
PlayerESPTab:Toggle({
    Title = "显示玩家名称",
    Desc = "在玩家头顶显示名字",
    Value = false,
    Callback = function(state)
        L.espShowNames = state
        L.refreshAllPlayers()
    end
})
PlayerESPTab:Toggle({
    Title = "队伍检测",
    Desc = "高亮区分 红色敌方 蓝色友方",
    Value = false,
    Callback = function(state)
        L.espTeamCheckPlayer = state
        L.refreshAllPlayers()
    end
})

-- 清理防冲突（原有 silent aim 未使用，保留占位）
local function disableSilentAim() end

Window:OnDestroy(function()
    print("窗口已销毁")
    isFlying_orig = false
    clearFlyRes_orig()
    isFlying_new = false
    clearFlyRes_new()
    disableSilentAim()
    -- 清理杀戮光环
    if L.auraEnabled then L.stopAura() end
    -- 清理僵尸透视
    if L.zombieESPHeartbeatConn then L.zombieESPHeartbeatConn:Disconnect() end
    L.clearAllZombieEffects()
    -- 清理玩家透视
    stopRefreshLoop()
    for _, player in ipairs(Players:GetPlayers()) do
        L.destroyPlayerComponents(player)
    end
end)
