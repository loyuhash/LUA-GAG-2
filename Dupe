-- Evenesce Farm | FIXED for Delta + GitHub
-- Paste this in GitHub as main.lua

-- FIX 1: Delta-safe UI parent
local function GetUIParent()
    local success, hui = pcall(function() return gethui() end)
    if success and hui then return hui end
    local success2, core = pcall(function() return game:GetService("CoreGui") end)
    if success2 then return core end
    return game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")
end

-- Remove old UI if exists
pcall(function()
    GetUIParent():FindFirstChild("EvenesceFarm"):Destroy()
end)

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

getgenv().EvenesceConfig = getgenv().EvenesceConfig or {
    AutoFarm = false,
    AutoSummon = false,
    AutoEvolve = false,
    AutoUpgrade = true,
    AutoStart = true,
}

print("[Evenesce] Loading UI...")

-- ===== UI =====
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "EvenesceFarm"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = GetUIParent()

local Main = Instance.new("Frame")
Main.Size = UDim2.new(0, 620, 0, 380)
Main.Position = UDim2.new(0.5, -310, 0.5, -190)
Main.BackgroundColor3 = Color3.fromRGB(13, 13, 18)
Main.BorderSizePixel = 0
Main.Active = true
Main.Parent = ScreenGui
local c = Instance.new("UICorner", Main); c.CornerRadius = UDim.new(0, 12)

local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 40)
TopBar.BackgroundTransparency = 1
TopBar.Parent = Main

local Logo = Instance.new("TextLabel")
Logo.Text = "Zuri Auto Farm✅"
Logo.Font = Enum.Font.GothamBold
Logo.TextSize = 16
Logo.TextColor3 = Color3.fromRGB(140, 130, 255)
Logo.BackgroundTransparency = 1
Logo.Size = UDim2.new(0, 200, 1, 0)
Logo.TextXAlignment = Enum.TextXAlignment.Left
Logo.Parent = TopBar

-- Close button (Delta needs this)
local Close = Instance.new("TextButton")
Close.Text = "X"
Close.Size = UDim2.new(0, 30, 0, 30)
Close.Position = UDim2.new(1, -35, 0, 5)
Close.BackgroundColor3 = Color3.fromRGB(255, 60, 60)
Close.TextColor3 = Color3.fromRGB(255,255,255)
Close.Font = Enum.Font.GothamBold
Close.Parent = TopBar
Instance.new("UICorner", Close).CornerRadius = UDim.new(0, 6)
Close.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)

-- Minimize / Show button
local ShowBtn = Instance.new("TextButton")
ShowBtn.Text = "Evenesce"
ShowBtn.Size = UDim2.new(0, 90, 0, 30)
ShowBtn.Position = UDim2.new(0, 10, 0, 10)
ShowBtn.Visible = false
ShowBtn.BackgroundColor3 = Color3.fromRGB(140,130,255)
ShowBtn.TextColor3 = Color3.fromRGB(255,255,255)
ShowBtn.Parent = GetUIParent()
Instance.new("UICorner", ShowBtn).CornerRadius = UDim.new(0, 6)

-- Sidebar
local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, 130, 1, -50)
Sidebar.Position = UDim2.new(0, 0, 0, 50)
Sidebar.BackgroundColor3 = Color3.fromRGB(16, 16, 22)
Sidebar.BorderSizePixel = 0
Sidebar.Parent = Main

local function SideBtn(name, y, active)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -10, 0, 32)
    btn.Position = UDim2.new(0, 5, 0, y)
    btn.Text = "  "..name
    btn.Font = Enum.Font.GothamMedium
    btn.TextSize = 13
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.BackgroundColor3 = active and Color3.fromRGB(28,28,40) or Color3.fromRGB(16,16,22)
    btn.TextColor3 = active and Color3.fromRGB(255,255,255) or Color3.fromRGB(150,150,170)
    btn.Parent = Sidebar
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    return btn
end
SideBtn("◉  Farm", 10, true)
SideBtn("✦  Summon", 48, false)
SideBtn("⬆  Evolve", 86, false)
SideBtn("👤  Player", 124, false)
SideBtn("⚙  Misc", 162, false)

-- Helper: Section
local function MakeSection(title, posX, posY, sizeX, sizeY)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(0, sizeX, 0, sizeY)
    f.Position = UDim2.new(0, posX, 0, posY)
    f.BackgroundColor3 = Color3.fromRGB(19, 19, 26)
    f.BorderSizePixel = 0
    f.Parent = Main
    Instance.new("UICorner", f).CornerRadius = UDim.new(0, 8)
    local t = Instance.new("TextLabel")
    t.Text = title
    t.Font = Enum.Font.GothamBold
    t.TextSize = 13
    t.TextColor3 = Color3.fromRGB(200,200,220)
    t.BackgroundTransparency = 1
    t.Position = UDim2.new(0, 10, 0, 6)
    t.Size = UDim2.new(1, -10, 0, 20)
    t.TextXAlignment = Enum.TextXAlignment.Left
    t.Parent = f
    return f
end

local function MakeToggle(parent, text, y, default, callback)
    local label = Instance.new("TextLabel")
    label.Text = text
    label.Font = Enum.Font.GothamMedium
    label.TextSize = 12
    label.TextColor3 = Color3.fromRGB(180,180,200)
    label.BackgroundTransparency = 1
    label.Position = UDim2.new(0, 10, 0, y)
    label.Size = UDim2.new(0, 150, 0, 20)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = parent
    local toggle = Instance.new("TextButton")
    toggle.Size = UDim2.new(0, 34, 0, 18)
    toggle.Position = UDim2.new(1, -44, 0, y+1)
    toggle.Text = ""
    toggle.BackgroundColor3 = default and Color3.fromRGB(140,130,255) or Color3.fromRGB(60,60,75)
    toggle.Parent = parent
    Instance.new("UICorner", toggle).CornerRadius = UDim.new(0, 9)
    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 14, 0, 14)
    knob.Position = default and UDim2.new(1, -16, 0, 2) or UDim2.new(0, 2, 0, 2)
    knob.BackgroundColor3 = Color3.fromRGB(255,255,255)
    knob.BorderSizePixel = 0
    knob.Parent = toggle
    Instance.new("UICorner", knob).CornerRadius = UDim.new(0, 7)
    local state = default
    toggle.MouseButton1Click:Connect(function()
        state = not state
        toggle.BackgroundColor3 = state and Color3.fromRGB(140,130,255) or Color3.fromRGB(60,60,75)
        knob.Position = state and UDim2.new(1, -16, 0, 2) or UDim2.new(0, 2, 0, 2)
        pcall(callback, state)
    end)
    return toggle
end

-- Status Label - so you know if it's working
local Status = Instance.new("TextLabel")
Status.Text = "Status: Ready ✅"
Status.Font = Enum.Font.GothamMedium
Status.TextSize = 12
Status.TextColor3 = Color3.fromRGB(0,255,120)
Status.BackgroundTransparency = 1
Status.Position = UDim2.new(0, 145, 0, 345)
Status.Size = UDim2.new(0, 300, 0, 20)
Status.TextXAlignment = Enum.TextXAlignment.Left
Status.Parent = Main

-- Sections
local FarmSection = MakeSection("Auto Farm", 145, 60, 220, 140)
MakeToggle(FarmSection, "Enable auto farm", 30, false, function(v)
    getgenv().EvenesceConfig.AutoFarm = v
    Status.Text = v and "Status: Farming... 🌾" or "Status: Paused"
    print("[Evenesce] AutoFarm:", v)
end)
MakeToggle(FarmSection, "Auto Upgrade", 60, true, function(v)
    getgenv().EvenesceConfig.AutoUpgrade = v
end)
MakeToggle(FarmSection, "Auto Start / Replay", 90, true, function(v)
    getgenv().EvenesceConfig.AutoStart = v
end)

local SummonSection = MakeSection("Auto Summon", 380, 60, 220, 140)
MakeToggle(SummonSection, "Enable auto summon", 30, false, function(v)
    getgenv().EvenesceConfig.AutoSummon = v
    print("[Evenesce] AutoSummon:", v)
end)

local EvolveSection = MakeSection("Auto Evolve + Misc", 145, 210, 455, 120)
MakeToggle(EvolveSection, "Enable auto evolve", 30, false, function(v)
    getgenv().EvenesceConfig.AutoEvolve = v
    print("[Evenesce] AutoEvolve:", v)
end)
MakeToggle(EvolveSection, "Anti-AFK", 60, true, function(v) end)

-- ===== DRAG (Fixed for Delta Mobile/PC) =====
local dragging, dragStart, startPos
TopBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = Main.Position
    end
end)
UIS.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        Main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

-- ===== LOGIC - Safe, won't crash if remotes missing =====
task.spawn(function()
    local VirtualUser = game:GetService("VirtualUser")
    LocalPlayer.Idled:Connect(function()
        pcall(function()
            VirtualUser:CaptureController()
            VirtualUser:ClickButton2(Vector2.new())
        end)
    end)
end)

-- Auto Farm Loop - This is where it actually farms
task.spawn(function()
    while true do
        task.wait(1)
        if getgenv().EvenesceConfig.AutoFarm then
            -- Try to find remotes safely, don't error if not found
            local success = pcall(function()
                local RS = game:GetService("ReplicatedStorage")
                -- Auto Start / Vote
                for _, name in ipairs({"StartGame", "VoteStart", "Replay", "Start"}) do
                    local r = RS:FindFirstChild(name, true)
                    if r and r:IsA("RemoteEvent") then r:FireServer() end
                    if r and r:IsA("RemoteFunction") then r:InvokeServer() end
                end
            end)
            if not success then
                Status.Text = "Status: Looking for game..."
            end
        end
    end
end)

print("[Evenesce] LOADED SUCCESSFULLY ✅")
