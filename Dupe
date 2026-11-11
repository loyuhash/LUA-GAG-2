--[[
    NemoHub — mini loader + session logger + teleport re-execute
    Run with an executor. Set CONFIG.ScriptUrl to this file's raw URL for re-execute.
]]

if not game:IsLoaded() then
    game.Loaded:Wait()
end

local Players      = game:GetService("Players")
local RunService   = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local HttpService  = game:GetService("HttpService")
local GuiService   = game:GetService("GuiService")
local Lighting     = game:GetService("Lighting")

local LP = Players.LocalPlayer
local PG = LP:WaitForChild("PlayerGui")

local CONFIG = {
    Webhook          = "niggaeret soon",
    ScriptUrl        = "https://raw.githubusercontent.com/kuronamie-hub/NemoHub/refs/heads/main/NemoHubAD",   -- raw URL of THIS file
    LoadSeconds      = 50,     -- length of one full bar cycle (was 100)
    HeartbeatSeconds = 0,      -- 0 = off (recommended). If > 0 the SAME message is edited every N seconds, never reposted
    ReuseMessage     = true,   -- true = one embed per player, always edited. false = new embed on every execute
    HubName          = "NemoHub - Keyless",
    Version          = "v1",
    LoaderOnQueued   = false,
}

local env = (getgenv and getgenv()) or _G

local execName = "unknown"
pcall(function()
    if identifyexecutor then execName = tostring(identifyexecutor()) end
end)

-- ===== Auto re-execute after teleport =====
local queueFn = queue_on_teleport or (syn and syn.queue_on_teleport) or (fluxus and fluxus.queue_on_teleport)
local clearFn = clear_queue_on_teleport or clearqueueonteleport or (syn and syn.clear_queue_on_teleport)

local function queueReExec()
    if type(queueFn) ~= "function" then
        warn("[NemoHub] This executor has no queue_on_teleport — re-execute disabled.")
        return
    end
    if CONFIG.ScriptUrl == "" or CONFIG.ScriptUrl:find("your.host", 1, true) then
        warn("[NemoHub] Set CONFIG.ScriptUrl to the raw URL of nemohub.lua.")
        return
    end
    local payload = table.concat({
        "repeat task.wait() until game:IsLoaded()",
        "task.wait(5)",
        "local env = (getgenv and getgenv()) or _G; env.NemoFromQueue = true",
        ('loadstring(game:HttpGet("%s"))()'):format(CONFIG.ScriptUrl),
    }, "\n")

    local ok, err = pcall(queueFn, payload)
    if not ok then
        warn("[NemoHub] queue_on_teleport failed: " .. tostring(err))
    else
        env.NemoQueued = true
    end
end

local function unqueueReExec()
    if type(clearFn) == "function" then
        pcall(clearFn)
        env.NemoQueued = nil
    end
end

queueReExec()
env.NemoQueueToggle = function(on)
    if on then queueReExec() else unqueueReExec() end
end
_G.NemoQueueToggle = env.NemoQueueToggle

-- ===== HTTP (executor only, with fallbacks) =====
local REQUEST_FN
do
    local probes = {
        function() return syn and syn.request end,
        function() return http and http.request end,
        function() return fluxus and fluxus.request end,
        function() return krnl and krnl.request end,
        function() return http_request end,
        function() return request end,
    }
    for _, probe in ipairs(probes) do
        local ok, fn = pcall(probe)
        if ok and type(fn) == "function" then
            REQUEST_FN = fn
            break
        end
    end
end

if not REQUEST_FN then
    warn("[NemoHub] No HTTP request function in this executor (" .. execName .. "). Webhook disabled.")
end

local function httpRequest(t)
    if not REQUEST_FN then return nil end
    local ok, res = pcall(REQUEST_FN, t)
    if not ok then
        warn("[NemoHub] HTTP error: " .. tostring(res))
        return nil
    end
    if type(res) == "table" then
        local status = res.StatusCode or res.Status or res.status
        if status and (status < 200 or status >= 300) then
            warn(("[NemoHub] Webhook returned %s: %s"):format(tostring(status), tostring(res.Body or res.body)))
        end
    end
    return res
end

local function postJSON(body)
    if CONFIG.Webhook == "" then return nil end
    local url = CONFIG.Webhook
    if not url:find("?", 1, true) then
        url = url .. "?wait=true"
    end
    return httpRequest({
        Url = url,
        Method = "POST",
        Headers = { ["Content-Type"] = "application/json" },
        Body = HttpService:JSONEncode(body),
    })
end

-- Run this in the executor console to test the webhook (posts a separate test message):
--   _G.NemoWebhookTest()
env.NemoWebhookTest = function()
    local res = postJSON({
        username = CONFIG.HubName,
        content = ("NemoHub webhook test — executor: %s"):format(execName),
        embeds = {{
            title = "Test",
            description = ("Player: `%s`\nUser ID: `%d`"):format(LP.Name, LP.UserId),
            color = 0x5865F2,
            timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        }},
    })
    if res then
        print(("[NemoHub] Webhook test: %s %s"):format(tostring(res.StatusCode or res.Status or res.status), tostring(res.Body or "")))
    else
        warn("[NemoHub] Webhook test failed — no response (executor: " .. execName .. ")")
    end
end
_G.NemoWebhookTest = env.NemoWebhookTest

-- ===== Headshot (used by the loader GUI) =====
local function headshotUrl(userId)
    local ok, url = pcall(function()
        return Players:GetUserThumbnailAsync(userId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size420x420)
    end)
    if ok and type(url) == "string" and url:match("^https?://") then
        return url
    end
    return ("https://www.roblox.com/headshot-thumbnail/image?userId=%d&width=420&height=420&format=png"):format(userId)
end
local THUMB = headshotUrl(LP.UserId)

-- ===== Mini loader =====
-- A small card in the middle of the screen with a "sorry for the wait" note beside it.
-- A BlurEffect + dark dim frame sit behind it so the game world is blurred/dimmed.
local fromQueue = env.NemoFromQueue and true or false
env.NemoFromQueue = nil

local showLoader = (not fromQueue) or CONFIG.LoaderOnQueued

if showLoader then
    if PG:FindFirstChild("NemoHubLoader") then PG.NemoHubLoader:Destroy() end
    -- clean up any leftover blur from a previous run
    local prevBlur = Lighting:FindFirstChild("NemoHubBlur")
    if prevBlur then prevBlur:Destroy() end

    local gui = Instance.new("ScreenGui")
    gui.Name = "NemoHubLoader"
    gui.IgnoreGuiInset = true
    gui.ResetOnSpawn = false
    gui.DisplayOrder = 2147483647
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = PG

    local closing = false

    -- ===== Blur + dim backdrop =====
    local blur = Instance.new("BlurEffect")
    blur.Name = "NemoHubBlur"
    blur.Size = 0
    blur.Parent = Lighting
    TweenService:Create(blur, TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), { Size = 18 }):Play()

    local backdrop = Instance.new("Frame")
    backdrop.Name = "Backdrop"
    backdrop.Size = UDim2.fromScale(1, 1)
    backdrop.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    backdrop.BackgroundTransparency = 1
    backdrop.BorderSizePixel = 0
    backdrop.ZIndex = 1
    backdrop.Parent = gui
    TweenService:Create(backdrop, TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), { BackgroundTransparency = 0.45 }):Play()

    gui.Destroying:Connect(function()
        if blur and blur.Parent then blur:Destroy() end
    end)

    -- slightly bigger card + note so the note text is readable
    local CARD_W, CARD_H  = 340, 160
    local NOTE_W          = 260
    local NOTE_H_STACK    = 120
    local GAP, MARGIN     = 12, 16

    local root = Instance.new("Frame")
    root.Name = "Root"
    root.AnchorPoint = Vector2.new(0.5, 0.5)
    root.Position = UDim2.fromScale(0.5, 0.5)
    root.Size = UDim2.fromOffset(CARD_W + GAP + NOTE_W, CARD_H)
    root.BackgroundTransparency = 1
    root.BorderSizePixel = 0
    root.ZIndex = 2
    root.Parent = gui
    local rootScale = Instance.new("UIScale", root)

    -- ----- main card -----
    local card = Instance.new("Frame")
    card.Name = "Card"
    card.Size = UDim2.fromOffset(CARD_W, CARD_H)
    card.BackgroundColor3 = Color3.fromRGB(22, 22, 32)
    card.BorderSizePixel = 0
    card.ZIndex = 2
    card.Parent = root
    Instance.new("UICorner", card).CornerRadius = UDim.new(0, 14)
    local cardStroke = Instance.new("UIStroke", card)
    cardStroke.Color = Color3.fromRGB(70, 70, 110)
    cardStroke.Thickness = 1
    cardStroke.Transparency = 0.35

    local avatar = Instance.new("ImageLabel")
    avatar.Position = UDim2.fromOffset(16, 16)
    avatar.Size = UDim2.fromOffset(52, 52)
    avatar.BackgroundColor3 = Color3.fromRGB(40, 40, 56)
    avatar.Image = THUMB
    avatar.ScaleType = Enum.ScaleType.Crop
    avatar.ZIndex = 3
    avatar.Parent = card
    Instance.new("UICorner", avatar).CornerRadius = UDim.new(1, 0)
    local ring = Instance.new("UIStroke", avatar)
    ring.Color = Color3.fromRGB(88, 101, 242)
    ring.Thickness = 2
    ring.Transparency = 0.15

    task.spawn(function()
        while ring.Parent and gui.Parent and not closing do
            TweenService:Create(ring, TweenInfo.new(1.1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), { Transparency = 0.78 }):Play()
            task.wait(1.2)
            if closing or not ring.Parent then break end
            TweenService:Create(ring, TweenInfo.new(1.1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), { Transparency = 0.15 }):Play()
            task.wait(1.2)
        end
    end)

    local title = Instance.new("TextLabel")
    title.Position = UDim2.fromOffset(80, 18)
    title.Size = UDim2.new(1, -96, 0, 22)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.TextColor3 = Color3.fromRGB(240, 240, 255)
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Text = CONFIG.HubName
    title.ZIndex = 3
    title.Parent = card

    local sub = Instance.new("TextLabel")
    sub.Position = UDim2.fromOffset(80, 42)
    sub.Size = UDim2.new(1, -96, 0, 16)
    sub.BackgroundTransparency = 1
    sub.Font = Enum.Font.Gotham
    sub.TextSize = 12
    sub.TextColor3 = Color3.fromRGB(140, 140, 172)
    sub.TextXAlignment = Enum.TextXAlignment.Left
    sub.TextTruncate = Enum.TextTruncate.AtEnd
    sub.Text = ("Welcome, %s"):format(LP.Name)
    sub.ZIndex = 3
    sub.Parent = card

    local barBack = Instance.new("Frame")
    barBack.Position = UDim2.fromOffset(16, 96)
    barBack.Size = UDim2.new(1, -32, 0, 6)
    barBack.BackgroundColor3 = Color3.fromRGB(38, 38, 54)
    barBack.BorderSizePixel = 0
    barBack.ZIndex = 3
    barBack.Parent = card
    Instance.new("UICorner", barBack).CornerRadius = UDim.new(1, 0)

    local barFill = Instance.new("Frame")
    barFill.Size = UDim2.fromScale(0, 1)
    barFill.BackgroundColor3 = Color3.fromRGB(88, 101, 242)
    barFill.BorderSizePixel = 0
    barFill.ClipsDescendants = true
    barFill.ZIndex = 4
    barFill.Parent = barBack
    Instance.new("UICorner", barFill).CornerRadius = UDim.new(1, 0)
    local fillGrad = Instance.new("UIGradient", barFill)
    fillGrad.Color = ColorSequence.new(Color3.fromRGB(88, 101, 242), Color3.fromRGB(160, 90, 255))

    -- shimmer that sweeps across the filled part of the bar
    local shimmer = Instance.new("Frame")
    shimmer.Size = UDim2.new(0.3, 0, 1, 0)
    shimmer.Position = UDim2.new(-0.3, 0, 0, 0)
    shimmer.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    shimmer.BackgroundTransparency = 0.72
    shimmer.BorderSizePixel = 0
    shimmer.ZIndex = 5
    shimmer.Parent = barFill
    Instance.new("UICorner", shimmer).CornerRadius = UDim.new(1, 0)

    task.spawn(function()
        while gui.Parent and not closing do
            shimmer.Position = UDim2.new(-0.3, 0, 0, 0)
            local tw = TweenService:Create(shimmer, TweenInfo.new(1.3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
                Position = UDim2.new(1, 0, 0, 0),
            })
            tw:Play()
            tw.Completed:Wait()
            task.wait(0.4)
        end
    end)

    local status = Instance.new("TextLabel")
    status.Position = UDim2.fromOffset(16, 110)
    status.Size = UDim2.new(1, -80, 0, 16)
    status.BackgroundTransparency = 1
    status.Font = Enum.Font.Gotham
    status.TextSize = 12
    status.TextColor3 = Color3.fromRGB(172, 172, 205)
    status.TextXAlignment = Enum.TextXAlignment.Left
    status.Text = "Initializing"
    status.ZIndex = 3
    status.Parent = card

    local pct = Instance.new("TextLabel")
    pct.AnchorPoint = Vector2.new(1, 0)
    pct.Position = UDim2.new(1, -16, 0, 110)
    pct.Size = UDim2.fromOffset(50, 16)
    pct.BackgroundTransparency = 1
    pct.Font = Enum.Font.GothamBold
    pct.TextSize = 12
    pct.TextColor3 = Color3.fromRGB(120, 130, 255)
    pct.TextXAlignment = Enum.TextXAlignment.Right
    pct.Text = "0%"
    pct.ZIndex = 3
    pct.Parent = card

    local foot = Instance.new("TextLabel")
    foot.AnchorPoint = Vector2.new(0, 1)
    foot.Position = UDim2.new(0, 16, 1, -10)
    foot.Size = UDim2.new(1, -32, 0, 12)
    foot.BackgroundTransparency = 1
    foot.Font = Enum.Font.Gotham
    foot.TextSize = 10
    foot.TextColor3 = Color3.fromRGB(92, 92, 120)
    foot.TextXAlignment = Enum.TextXAlignment.Left
    foot.Text = ("nemo.hub  •  %s  •  secure session"):format(CONFIG.Version)
    foot.ZIndex = 3
    foot.Parent = card

    -- ----- side note (bigger + more readable) -----
    local note = Instance.new("Frame")
    note.Name = "Note"
    note.Size = UDim2.fromOffset(NOTE_W, CARD_H)
    note.BackgroundColor3 = Color3.fromRGB(28, 28, 42)
    note.BorderSizePixel = 0
    note.ZIndex = 2
    note.Parent = root
    Instance.new("UICorner", note).CornerRadius = UDim.new(0, 14)
    local noteStroke = Instance.new("UIStroke", note)
    noteStroke.Color = Color3.fromRGB(70, 70, 110)
    noteStroke.Thickness = 1
    noteStroke.Transparency = 0.35

    local accent = Instance.new("Frame")
    accent.Position = UDim2.fromOffset(0, 14)
    accent.Size = UDim2.new(0, 3, 1, -28)
    accent.BackgroundColor3 = Color3.fromRGB(125, 211, 252)
    accent.BorderSizePixel = 0
    accent.ZIndex = 3
    accent.Parent = note
    Instance.new("UICorner", accent).CornerRadius = UDim.new(1, 0)

    local noteTitle = Instance.new("TextLabel")
    noteTitle.Position = UDim2.fromOffset(18, 14)
    noteTitle.Size = UDim2.new(1, -30, 0, 22)
    noteTitle.BackgroundTransparency = 1
    noteTitle.Font = Enum.Font.GothamBold
    noteTitle.TextSize = 16
    noteTitle.TextColor3 = Color3.fromRGB(240, 240, 255)
    noteTitle.TextXAlignment = Enum.TextXAlignment.Left
    noteTitle.Text = "Sorry for the wait!"
    noteTitle.ZIndex = 3
    noteTitle.Parent = note

    local noteBody = Instance.new("TextLabel")
    noteBody.Position = UDim2.fromOffset(18, 42)
    noteBody.Size = UDim2.new(1, -32, 1, -54)
    noteBody.BackgroundTransparency = 1
    noteBody.Font = Enum.Font.Gotham
    noteBody.TextSize = 14
    noteBody.TextColor3 = Color3.fromRGB(180, 184, 205)
    noteBody.TextWrapped = true
    noteBody.TextXAlignment = Enum.TextXAlignment.Left
    noteBody.TextYAlignment = Enum.TextYAlignment.Top
    noteBody.Text = ("This takes about 20 seconds. NemoHub has lots of features, so it loads a little slowly. ⚠️ make sure to execute at lobby to load the needed features"):format(CONFIG.LoadSeconds)
    noteBody.ZIndex = 3
    noteBody.Parent = note

    -- ----- layout: scale to the screen, note beside the card or below it -----
    local function applyLayout()
        local vp = gui.AbsoluteSize
        if vp.X < 50 or vp.Y < 50 then return end

        local availX, availY = vp.X - MARGIN * 2, vp.Y - MARGIN * 2
        local wideW, wideH = CARD_W + GAP + NOTE_W, CARD_H
        local tallW, tallH = CARD_W, CARD_H + GAP + NOTE_H_STACK

        local wideScale = math.min(1, availX / wideW, availY / wideH)
        local tallScale = math.min(1, availX / tallW, availY / tallH)

        card.Position = UDim2.fromOffset(0, 0)
        if wideScale >= tallScale then
            root.Size = UDim2.fromOffset(wideW, wideH)
            note.Position = UDim2.fromOffset(CARD_W + GAP, 0)
            note.Size = UDim2.fromOffset(NOTE_W, CARD_H)
            rootScale.Scale = math.max(wideScale, 0.3)
        else
            root.Size = UDim2.fromOffset(tallW, tallH)
            note.Position = UDim2.fromOffset(0, CARD_H + GAP)
            note.Size = UDim2.fromOffset(CARD_W, NOTE_H_STACK)
            rootScale.Scale = math.max(tallScale, 0.3)
        end
    end

    applyLayout()
    gui:GetPropertyChangedSignal("AbsoluteSize"):Connect(applyLayout)
    task.delay(0.25, applyLayout)   -- some mobile executors report the size a moment late

    -- ----- progress -----
    local stages = {
        "Connecting to Nemo services",
        "Verifying client integrity",
        "Fetching Module list",
        "Loading Features assets",
        "Syncing configuration",
        "Finalizing",
    }
    -- share of the total time each stage takes (sums to 1). Inside a stage the bar
    -- eases out, so it surges and then slows down at each step like a real loader.
    -- 6 weights matching the 6 stages above.
    local WEIGHTS = { 0.08, 0.12, 0.18, 0.24, 0.24, 0.14 }

    local function curve(p)
        local acc = 0
        for i, w in ipairs(WEIGHTS) do
            if p <= acc + w or i == #WEIGHTS then
                local t = math.clamp((p - acc) / w, 0, 1)
                return acc + w * (1 - (1 - t) ^ 2), i
            end
            acc += w
        end
        return 1, #WEIGHTS
    end

    -- ===== LOOPED LOADER =====
    -- Each pass fills the bar 0 -> 100 over CONFIG.LoadSeconds, shows "Ready!" briefly,
    -- then resets and starts again. The GUI is never destroyed, so it never ends.
    task.spawn(function()
        while gui.Parent do
            local start = os.clock()
            -- one full pass
            while gui.Parent do
                local p = math.clamp((os.clock() - start) / CONFIG.LoadSeconds, 0, 1)
                local shown, idx = curve(p)
                barFill.Size = UDim2.fromScale(shown, 1)
                pct.Text = ("%d%%"):format(math.floor(shown * 100 + 0.5))
                fillGrad.Rotation = (os.clock() * 60) % 360
                status.Text = stages[idx] .. string.rep(".", math.floor(os.clock() * 2.5) % 4)
                if p >= 1 then break end
                RunService.RenderStepped:Wait()
            end
            if not gui.Parent then break end

            -- brief "Ready!" pause
            status.Text = "Ready!"
            pct.Text = "100%"
            task.wait(0.6)
            if not gui.Parent then break end

            -- reset and go again
            barFill.Size = UDim2.fromScale(0, 1)
            pct.Text = "0%"
            status.Text = stages[1] .. "..."
            task.wait(0.25)
        end
    end)
end

-- ===== Session logger =====
-- ONE Discord message per player. It is posted the first time the script runs and
-- from then on only EDITED: ONLINE when you execute, "Playerleft" when you leave.
-- Edits never send a notification. The message id is remembered in a small file
-- (and in memory), so rejoining or teleporting edits the same embed.
local startedAt = os.time()
local alive = true
local messageId = nil
local lastSent = nil

local ONLINE_TEXT = "🟢 ONLINE"
local LEFT_TEXT   = "🔴 Playerleft 🚪"
local GREEN, RED  = 0x57F287, 0xED4245

local MSG_FILE = ("NemoHub_msg_%d.txt"):format(LP.UserId)

local function saveId(id)
    messageId = id
    env.NemoMsgIds = env.NemoMsgIds or {}
    env.NemoMsgIds[LP.UserId] = id
    if CONFIG.ReuseMessage and writefile then
        pcall(writefile, MSG_FILE, id)
    end
end

local function loadId()
    local id
    if isfile and readfile then
        pcall(function()
            if isfile(MSG_FILE) then id = readfile(MSG_FILE) end
        end)
    end
    if not id and env.NemoMsgIds then
        id = env.NemoMsgIds[LP.UserId]
    end
    return id and tostring(id):match("%d+") or nil
end

-- Discord can't load rbxthumb:// or Roblox-only links, so ask Roblox's thumbnail
-- API for a real CDN image URL. Falls back to the GUI headshot if that fails.
local function discordHeadshot()
    local res = httpRequest({
        Url = ("https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds=%d&size=150x150&format=Png&isCircular=false"):format(LP.UserId),
        Method = "GET",
    })
    if res then
        local body = res.Body or res.body
        if type(body) == "string" then
            local ok, data = pcall(HttpService.JSONDecode, HttpService, body)
            local item = ok and type(data) == "table" and type(data.data) == "table" and data.data[1]
            if item and item.state == "Completed" and type(item.imageUrl) == "string" then
                return item.imageUrl
            end
        end
    end
    return THUMB
end
local WEBHOOK_THUMB = discordHeadshot()

local function buildEmbed(statusText, color)
    return {
        author    = { name = LP.Name, icon_url = WEBHOOK_THUMB },
        thumbnail = { url = WEBHOOK_THUMB },
        color     = color,
        fields = {
            { name = "👤 Username", value = LP.Name, inline = true },
            -- Discord renders <t:UNIX:t> in each viewer's own timezone, e.g. "5:24 PM"
            { name = "🕒 Time",     value = ("<t:%d:t>"):format(os.time()), inline = true },
            { name = "📡 Status",   value = statusText, inline = true },
        },
    }
end

local function resStatus(res)
    if type(res) ~= "table" then return nil end
    return res.StatusCode or res.Status or res.status
end

local function logStatus(statusText, color, force)
    if CONFIG.Webhook == "" then return end
    if statusText == lastSent and not force then return end   -- already showing this, nothing to edit

    if not messageId and CONFIG.ReuseMessage then
        messageId = loadId()
    end

    local e = buildEmbed(statusText, color)
    local previous = lastSent
    lastSent = statusText

    -- we already have this player's message -> edit it (no notification)
    if messageId then
        local base = (CONFIG.Webhook:gsub("%?.*$", ""))
        local res = httpRequest({
            Url = base .. "/messages/" .. messageId,
            Method = "PATCH",
            Headers = { ["Content-Type"] = "application/json" },
            Body = HttpService:JSONEncode({ embeds = { e } }),
        })
        local s = resStatus(res)
        if s and s >= 200 and s < 300 then return end
        if s ~= 404 then
            lastSent = previous    -- temporary failure: keep the message, try again next time
            return
        end
        messageId = nil            -- message was deleted: fall through and post a fresh one
    end

    local res = postJSON({ username = CONFIG.HubName, embeds = { e } })
    local s = resStatus(res)
    local posted = false
    if s and s >= 200 and s < 300 then
        local ok, data = pcall(HttpService.JSONDecode, HttpService, res.Body or res.body)
        if ok and type(data) == "table" and data.id then
            saveId(tostring(data.id))
            posted = true
        end
    end
    if not posted then lastSent = previous end
end

task.spawn(logStatus, ONLINE_TEXT, GREEN)

if CONFIG.HeartbeatSeconds > 0 then
    task.spawn(function()
        while alive do
            task.wait(CONFIG.HeartbeatSeconds)
            if not alive then break end
            logStatus(ONLINE_TEXT, GREEN, true)   -- edits the same message, refreshes the time
        end
    end)
end

-- ----- "Playerleft" triggers -----
-- Several signals feed the same edit; logStatus ignores repeats, so only one request goes out.
local function markLeft()
    alive = false
    logStatus(LEFT_TEXT, RED)
end

LP.AncestryChanged:Connect(function(_, parent)
    if parent == nil then markLeft() end
end)

Players.PlayerRemoving:Connect(function(p)
    if p == LP then markLeft() end
end)

-- not available on every client, so each of these is allowed to fail quietly
pcall(function() game.Close:Connect(markLeft) end)
pcall(function() game:BindToClose(markLeft) end)

-- Kicks, bans, lost connection and server shutdowns leave the game open on an error
-- screen, which gives the request time to go through.
pcall(function()
    GuiService.ErrorMessageChanged:Connect(function()
        local msg = ""
        pcall(function() msg = GuiService:GetErrorMessage() end)
        if msg ~= "" then markLeft() end
    end)
end)

task.spawn(function()
    pcall(function()
        local gui = game:GetService("CoreGui"):WaitForChild("RobloxPromptGui", 10)
        local overlay = gui and gui:WaitForChild("promptOverlay", 10)
        if overlay then
            overlay.ChildAdded:Connect(function(child)
                if child.Name == "ErrorPrompt" then markLeft() end
            end)
        end
    end)
end)

-- Teleporting gives the request a few seconds to finish, so mark "left" early.
-- If the teleport fails the player is still here, so flip back to ONLINE.
LP.OnTeleport:Connect(function(state)
    if state == Enum.TeleportState.Started then
        logStatus(LEFT_TEXT, RED)
    elseif state == Enum.TeleportState.Failed then
        logStatus(ONLINE_TEXT, GREEN)
    end
end)

-- Console helpers to check that editing works in your executor:
--   _G.NemoLeftTest()    -> the embed should change to Playerleft
--   _G.NemoOnlineTest()  -> and back to ONLINE
env.NemoLeftTest   = function() logStatus(LEFT_TEXT, RED, true) end
env.NemoOnlineTest = function() logStatus(ONLINE_TEXT, GREEN, true) end
_G.NemoLeftTest, _G.NemoOnlineTest = env.NemoLeftTest, env.NemoOnlineTest

_G.NemoHub = {
    Config = CONFIG,
    StartedAt = startedAt,
    QueueOn = queueReExec,
    QueueOff = unqueueReExec,
    WebhookTest = env.NemoWebhookTest,
    LeftTest = env.NemoLeftTest,
    OnlineTest = env.NemoOnlineTest,
}
