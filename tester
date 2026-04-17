-- ╔══════════════════════════════════════════════════╗
-- ║   BG Testing Suite  v3.0                         ║
-- ║   Tabs: Follow | Combat                          ║
-- ║   For anti-cheat validation on your own game     ║
-- ╚══════════════════════════════════════════════════╝

local Players          = game:GetService("Players")
local TweenService     = game:GetService("TweenService")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local VIM              = game:GetService("VirtualInputManager")

local player    = Players.LocalPlayer
local guiParent = player:WaitForChild("PlayerGui")

-------------------------------------------------
-- FOLLOW CONFIG
-------------------------------------------------

local cfg = {
    followDistance = 5,
    stayTime       = 4,
    tweenSpeed     = 0.08,
}

-------------------------------------------------
-- SHARED STATE
-------------------------------------------------

local selectedPlayers  = {}  -- [Player] = true
local playerButtons    = {}  -- [Player] = { btn, dot }

-- Follow state
local currentTarget    = nil
local targetStartTime  = 0
local activeTween      = nil

-- Smart target cycling queue (HP-sorted)
local smartQueue       = {}  -- { {player=p, hp=n}, ... }
local smartQueueIdx    = 1
local smartQueueBuilt  = false

-- Combat timers
local lastPunch        = 0
local lastMove         = { 0, 0, 0, 0 }
local lastUlt          = 0
local blockHeld        = false
local smartUltFired    = false

-------------------------------------------------
-- INPUT EMULATION
-------------------------------------------------

-- Keys for moves 1-4, block, ultimate
local MOVE_KEYS = {
    Enum.KeyCode.One,
    Enum.KeyCode.Two,
    Enum.KeyCode.Three,
    Enum.KeyCode.Four,
}
local BLOCK_KEY = Enum.KeyCode.F
local ULT_KEY   = Enum.KeyCode.G

local function tapKey(keyCode)
    VIM:SendKeyEvent(true,  keyCode, false, game)
    task.delay(0.06, function()
        VIM:SendKeyEvent(false, keyCode, false, game)
    end)
end

local function holdKey(keyCode)
    VIM:SendKeyEvent(true, keyCode, false, game)
end

local function releaseKey(keyCode)
    VIM:SendKeyEvent(false, keyCode, false, game)
end

local function clickLMB()
    local c = workspace.CurrentCamera.ViewportSize / 2
    VIM:SendMouseButtonEvent(c.X, c.Y, 0, true,  game, 0)
    task.delay(0.06, function()
        VIM:SendMouseButtonEvent(c.X, c.Y, 0, false, game, 0)
    end)
end

-------------------------------------------------
-- COLOURS
-------------------------------------------------

local C = {
    bg      = Color3.fromRGB( 18,  18,  28),
    panel   = Color3.fromRGB( 28,  28,  42),
    accent  = Color3.fromRGB( 90,  90, 200),
    accentD = Color3.fromRGB( 50,  50, 120),
    textHi  = Color3.fromRGB(235, 235, 255),
    textDim = Color3.fromRGB(130, 130, 160),
    green   = Color3.fromRGB( 50, 210, 110),
    red     = Color3.fromRGB(220,  65,  65),
    orange  = Color3.fromRGB(220, 145,  40),
    stroke  = Color3.fromRGB( 70,  70, 110),
}

-------------------------------------------------
-- GUI ROOT
-------------------------------------------------

local gui = Instance.new("ScreenGui")
gui.Name            = "BGTestSuite"
gui.IgnoreGuiInset  = true
gui.DisplayOrder    = 999999
gui.ResetOnSpawn    = false
gui.ZIndexBehavior  = Enum.ZIndexBehavior.Sibling
gui.Parent          = guiParent

-------------------------------------------------
-- DRAG UTILITY
-------------------------------------------------

local function makeDraggable(frame, handle)
    handle = handle or frame
    local drag, dragStart, startPos = false, Vector3.new(), UDim2.new()

    handle.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            drag      = true
            dragStart = i.Position
            startPos  = frame.Position
        end
    end)

    UserInputService.InputChanged:Connect(function(i)
        if not drag then return end
        if i.UserInputType == Enum.UserInputType.MouseMovement
        or i.UserInputType == Enum.UserInputType.Touch then
            local d = i.Position - dragStart
            frame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + d.X,
                startPos.Y.Scale, startPos.Y.Offset + d.Y
            )
        end
    end)

    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1
        or i.UserInputType == Enum.UserInputType.Touch then
            drag = false
        end
    end)
end

-------------------------------------------------
-- UI FACTORY
-------------------------------------------------

local function corner(p, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 8)
    c.Parent = p
    return c
end

local function stroke(p, col, thick)
    local s = Instance.new("UIStroke")
    s.Color     = col   or C.stroke
    s.Thickness = thick or 1.2
    s.Parent    = p
    return s
end

local function lbl(text, sz, pos, parent, xAlign, fs)
    local l = Instance.new("TextLabel")
    l.Text              = text
    l.Size              = sz
    l.Position          = pos
    l.BackgroundTransparency = 1
    l.TextColor3        = C.textHi
    l.Font              = Enum.Font.GothamBold
    l.TextSize          = fs or 13
    l.TextXAlignment    = xAlign or Enum.TextXAlignment.Left
    l.Parent            = parent
    return l
end

local function btn(text, sz, pos, parent, bg)
    bg = bg or C.panel
    local b = Instance.new("TextButton")
    b.Text             = text
    b.Size             = sz
    b.Position         = pos
    b.BackgroundColor3 = bg
    b.TextColor3       = C.textHi
    b.Font             = Enum.Font.GothamBold
    b.TextSize         = 12
    b.AutoButtonColor  = false
    b.Parent           = parent
    corner(b, 6)
    stroke(b, C.stroke, 1)
    local hov = bg:Lerp(Color3.new(1, 1, 1), 0.1)
    b.MouseEnter:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.1), {BackgroundColor3 = hov}):Play()
    end)
    b.MouseLeave:Connect(function()
        TweenService:Create(b, TweenInfo.new(0.1), {BackgroundColor3 = bg}):Play()
    end)
    return b
end

local function ibox(default, sz, pos, parent)
    local b = Instance.new("TextBox")
    b.Text             = tostring(default)
    b.Size             = sz
    b.Position         = pos
    b.BackgroundColor3 = C.bg
    b.TextColor3       = C.textHi
    b.Font             = Enum.Font.Gotham
    b.TextSize         = 12
    b.ClearTextOnFocus = false
    b.Parent           = parent
    corner(b, 5)
    stroke(b, C.accentD, 1)
    return b
end

local function divLine(y, parent)
    local d = Instance.new("Frame")
    d.Size              = UDim2.new(1, -20, 0, 1)
    d.Position          = UDim2.new(0, 10, 0, y)
    d.BackgroundColor3  = C.stroke
    d.BackgroundTransparency = 0.4
    d.BorderSizePixel   = 0
    d.Parent            = parent
end

local function secLbl(text, y, parent)
    local l = lbl(text, UDim2.new(1, 0, 0, 14), UDim2.new(0, 14, 0, y),
        parent, Enum.TextXAlignment.Left, 9)
    l.TextColor3 = C.textDim
    return l
end

-- ON/OFF toggle that returns (button, getter, setter)
local function makeToggle(sz, pos, parent, onColor)
    onColor = onColor or C.green
    local state = false

    local b = Instance.new("TextButton")
    b.Text             = "OFF"
    b.Size             = sz
    b.Position         = pos
    b.BackgroundColor3 = C.panel
    b.TextColor3       = C.textDim
    b.Font             = Enum.Font.GothamBold
    b.TextSize         = 11
    b.AutoButtonColor  = false
    b.Parent           = parent
    corner(b, 5)
    local st = stroke(b, C.stroke, 1)

    local function set(v)
        state = v
        if v then
            b.Text             = "ON"
            b.BackgroundColor3 = onColor:Lerp(C.bg, 0.55)
            b.TextColor3       = onColor
            st.Color           = onColor
        else
            b.Text             = "OFF"
            b.BackgroundColor3 = C.panel
            b.TextColor3       = C.textDim
            st.Color           = C.stroke
        end
    end

    b.MouseButton1Click:Connect(function() set(not state) end)
    return b, function() return state end, set
end

-- Convenience: label + optional interval box + toggle in one row
-- Returns: toggleGetter, setter, intervalBox (may be nil)
local screenY = workspace.CurrentCamera.ViewportSize.Y
local WIN_H = math.min(484, screenY - 80)

local function combatRowToggle(parent, labelText, y, onColor)
    lbl(labelText, UDim2.new(0, 200, 0, 28), UDim2.new(0, 14, 0, y), parent, nil, 12)
    local _, get, set = makeToggle(UDim2.new(0, 82, 0, 28), UDim2.new(0, WIN_W - 96, 0, y), parent, onColor)
    return get, set
end

local function combatRowWithInt(parent, labelText, y, defaultInt, onColor)
    lbl(labelText, UDim2.new(0, 138, 0, 28), UDim2.new(0, 14, 0, y), parent, nil, 12)
    -- interval label + box
    local intLbl = lbl("int:", UDim2.new(0, 22, 0, 28), UDim2.new(0, 155, 0, y), parent, nil, 10)
    intLbl.TextColor3 = C.textDim
    local box = ibox(defaultInt, UDim2.new(0, 46, 0, 28), UDim2.new(0, 175, 0, y), parent)
    local sLbl = lbl("s", UDim2.new(0, 10, 0, 28), UDim2.new(0, 223, 0, y), parent, nil, 11)
    sLbl.TextColor3 = C.textDim
    local _, get, set = makeToggle(UDim2.new(0, 82, 0, 28), UDim2.new(0, WIN_W - 96, 0, y), parent, onColor)
    return get, set, box
end

-------------------------------------------------
-- FLOATING TOGGLE BUTTON
-------------------------------------------------

local toggleBtn = btn("⚔  Tester", UDim2.new(0, 112, 0, 32), UDim2.new(0, 20, 0, 20), gui, C.panel)
toggleBtn.BackgroundTransparency = 0.15
stroke(toggleBtn, C.accent, 1.5)
makeDraggable(toggleBtn)

-------------------------------------------------
-- MAIN WINDOW
-------------------------------------------------

local WIN_H = 484

local main = Instance.new("Frame")
main.Name                   = "MainWindow"
main.Size                   = UDim2.new(0, WIN_W, 0, WIN_H)
main.Position               = UDim2.new(0, 20, 0, 66)
main.BackgroundColor3       = C.bg
main.BackgroundTransparency = 0.08
main.Visible                = false
main.Parent                 = gui
corner(main, 10)
stroke(main, C.accent, 1.5)

-------------------------------------------------
-- TITLE BAR
-------------------------------------------------

local titleBar = Instance.new("Frame")
titleBar.Size                   = UDim2.new(1, 0, 0, 40)
titleBar.BackgroundColor3       = C.panel
titleBar.BackgroundTransparency = 0.05
titleBar.BorderSizePixel        = 0
titleBar.Parent                 = main
corner(titleBar, 10)

-- flatten the bottom corners of the title bar
local titleFix = Instance.new("Frame")
titleFix.Size                   = UDim2.new(1, 0, 0, 10)
titleFix.Position               = UDim2.new(0, 0, 1, -10)
titleFix.BackgroundColor3       = C.panel
titleFix.BackgroundTransparency = 0.05
titleFix.BorderSizePixel        = 0
titleFix.Parent                 = titleBar

lbl("⚔  BG Test Suite  v3.0", UDim2.new(1, -40, 1, 0),
    UDim2.new(0, 12, 0, 0), titleBar, Enum.TextXAlignment.Left, 13)

local closeBtn = btn("✕", UDim2.new(0, 26, 0, 26), UDim2.new(1, -32, 0, 7),
    titleBar, Color3.fromRGB(155, 38, 38))
closeBtn.TextSize = 14
stroke(closeBtn, Color3.fromRGB(200, 60, 60), 1)

makeDraggable(main, titleBar)

-------------------------------------------------
-- TAB BAR
-------------------------------------------------

local tabBar = Instance.new("Frame")
tabBar.Size              = UDim2.new(1, 0, 0, 34)
tabBar.Position          = UDim2.new(0, 0, 0, 40)
tabBar.BackgroundColor3  = C.bg
tabBar.BackgroundTransparency = 0.25
tabBar.BorderSizePixel   = 0
tabBar.Parent            = main

local function makeTab(text, xScale)
    local t = Instance.new("TextButton")
    t.Size              = UDim2.new(0.5, 0, 1, 0)
    t.Position          = UDim2.new(xScale, 0, 0, 0)
    t.BackgroundTransparency = 1
    t.TextColor3        = C.textDim
    t.Font              = Enum.Font.GothamBold
    t.TextSize          = 12
    t.Text              = text
    t.AutoButtonColor   = false
    t.BorderSizePixel   = 0
    t.Parent            = tabBar
    return t
end

local tabFollow = makeTab("📍  Follow", 0)
local tabCombat = makeTab("⚔  Combat", 0.5)

-- animated underline indicator
local tabLine = Instance.new("Frame")
tabLine.Size             = UDim2.new(0.5, 0, 0, 2)
tabLine.Position         = UDim2.new(0, 0, 1, -2)
tabLine.BackgroundColor3 = C.accent
tabLine.BorderSizePixel  = 0
tabLine.Parent           = tabBar

-------------------------------------------------
-- TAB CONTENT FRAMES
-------------------------------------------------

local CONTENT_Y = 74   -- titleBar(40) + tabBar(34)
local CONTENT_H = WIN_H - CONTENT_Y

local function makeContentFrame()

    local f = Instance.new("ScrollingFrame")
    f.Size                   = UDim2.new(1, 0, 0, CONTENT_H)
    f.Position               = UDim2.new(0, 0, 0, CONTENT_Y)
    f.BackgroundTransparency = 1
    f.BorderSizePixel        = 0

    f.ScrollBarThickness     = 4
    f.ScrollBarImageColor3   = C.accent
    f.CanvasSize             = UDim2.new(0,0,0,0)
    f.AutomaticCanvasSize    = Enum.AutomaticSize.Y

    f.Parent = main

    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0,6)
    layout.Parent = f

    return f
end
local followFrame = makeContentFrame()
local combatFrame = makeContentFrame()
combatFrame.Visible = false

followFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
combatFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y

local function switchTab(toFollow)
    followFrame.Visible = toFollow
    combatFrame.Visible = not toFollow
    tabFollow.TextColor3 = toFollow and C.textHi or C.textDim
    tabCombat.TextColor3 = toFollow and C.textDim or C.textHi
    TweenService:Create(tabLine, TweenInfo.new(0.16, Enum.EasingStyle.Quad),
        { Position = UDim2.new(toFollow and 0 or 0.5, 0, 1, -2) }):Play()
end

tabFollow.MouseButton1Click:Connect(function() switchTab(true)  end)
tabCombat.MouseButton1Click:Connect(function() switchTab(false) end)
switchTab(true)

-------------------------------------------------
-- TOGGLE WINDOW
-------------------------------------------------

toggleBtn.MouseButton1Click:Connect(function()
    main.Visible = not main.Visible
end)

closeBtn.MouseButton1Click:Connect(function()
    main.Visible = false
end)

-------------------------------------------------
-- ╔══════════════════════════╗
-- ║   FOLLOW TAB CONTENT     ║
-- ╚══════════════════════════╝
-------------------------------------------------

-- Status bar
local statusBar = Instance.new("Frame")
statusBar.Size              = UDim2.new(1, -20, 0, 26)
statusBar.Position          = UDim2.new(0, 10, 0, 8)
statusBar.BackgroundColor3  = C.panel
statusBar.BackgroundTransparency = 0.2
statusBar.Parent            = followFrame
corner(statusBar, 5)
stroke(statusBar, C.stroke, 1)

local statusDot = Instance.new("Frame")
statusDot.Size             = UDim2.new(0, 8, 0, 8)
statusDot.Position         = UDim2.new(0, 10, 0.5, -4)
statusDot.BackgroundColor3 = C.textDim
statusDot.Parent           = statusBar
corner(statusDot, 4)

local statusText = lbl("Idle — no targets selected",
    UDim2.new(1, -28, 1, 0), UDim2.new(0, 26, 0, 0),
    statusBar, Enum.TextXAlignment.Left, 11)
statusText.Font       = Enum.Font.Gotham
statusText.TextColor3 = C.textDim

local function setStatus(text, color)
    color              = color or C.textDim
    statusText.Text    = text
    statusText.TextColor3 = color
    statusDot.BackgroundColor3 = color
end

-- Settings
divLine(42, followFrame)
secLbl("SETTINGS", 48, followFrame)

local function cfgRow(txt, default, y)
    lbl(txt, UDim2.new(0, 178, 0, 24), UDim2.new(0, 14, 0, y), followFrame, nil, 12)
    return ibox(default, UDim2.new(0, 68, 0, 24), UDim2.new(0, WIN_W - 82, 0, y), followFrame)
end

local distBox  = cfgRow("Behind Distance  (studs)",  cfg.followDistance, 62)
local timeBox  = cfgRow("Stay Time  (sec)",           cfg.stayTime,       90)
local tweenBox = cfgRow("Tween Speed",                cfg.tweenSpeed,     118)

-- Player list
divLine(150, followFrame)
secLbl("TARGET PLAYERS", 156, followFrame)

local list = Instance.new("ScrollingFrame")
list.Size                   = UDim2.new(1, -20, 0, 155)
list.Position               = UDim2.new(0, 10, 0, 172)
list.BackgroundColor3       = C.panel
list.BackgroundTransparency = 0.15
list.ScrollBarThickness     = 3
list.ScrollBarImageColor3   = C.accent
list.CanvasSize             = UDim2.new(0, 0, 0, 0)
list.BorderSizePixel        = 0
list.Parent                 = followFrame
corner(list, 7)
stroke(list, C.stroke, 1)

local listLayout = Instance.new("UIListLayout")
listLayout.Padding    = UDim.new(0, 3)
listLayout.SortOrder  = Enum.SortOrder.Name
listLayout.Parent     = list

local listPad = Instance.new("UIPadding")
listPad.PaddingTop   = UDim.new(0, 5)
listPad.PaddingLeft  = UDim.new(0, 5)
listPad.PaddingRight = UDim.new(0, 8)
listPad.Parent       = list

listLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    list.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 10)
end)

local refreshBtn = btn("↻  Refresh Players",
    UDim2.new(1, -20, 0, 30), UDim2.new(0, 10, 0, 336), followFrame, C.accentD)
refreshBtn.TextSize = 13
stroke(refreshBtn, C.accent, 1.2)

-- Build / rebuild player list
local function refreshPlayers()
    for _, v in pairs(list:GetChildren()) do
        if v:IsA("TextButton") then v:Destroy() end
    end
    playerButtons = {}

    for _, p in pairs(Players:GetPlayers()) do
        if p == player then continue end

        local sel = selectedPlayers[p] == true

        local row = Instance.new("TextButton")
        row.Size             = UDim2.new(1, 0, 0, 30)
        row.BackgroundColor3 = sel and Color3.fromRGB(30, 80, 55) or C.panel
        row.Text             = ""
        row.AutoButtonColor  = false
        row.Parent           = list
        corner(row, 5)
        stroke(row, sel and C.green or C.stroke, 1)

        local nl = lbl(p.Name, UDim2.new(1, -50, 1, 0), UDim2.new(0, 10, 0, 0), row, nil, 12)
        nl.Font = Enum.Font.GothamBold

        if p.DisplayName ~= p.Name then
            nl.Size     = UDim2.new(1, -50, 0, 16)
            nl.Position = UDim2.new(0, 10, 0, 3)
            local sub   = lbl(p.DisplayName, UDim2.new(1, -50, 0, 12),
                UDim2.new(0, 10, 0, 17), row, nil, 10)
            sub.Font       = Enum.Font.Gotham
            sub.TextColor3 = C.textDim
        end

        local dot = Instance.new("Frame")
        dot.Size             = UDim2.new(0, 9, 0, 9)
        dot.Position         = UDim2.new(1, -18, 0.5, -4)
        dot.BackgroundColor3 = sel and C.green or C.textDim
        dot.Parent           = row
        corner(dot, 5)

        playerButtons[p] = { btn = row, dot = dot }

        row.MouseButton1Click:Connect(function()
            if selectedPlayers[p] then
                selectedPlayers[p]   = nil
                smartQueueBuilt      = false          -- force queue rebuild
                TweenService:Create(row, TweenInfo.new(0.15),
                    {BackgroundColor3 = C.panel}):Play()
                stroke(row, C.stroke, 1)
                dot.BackgroundColor3 = C.textDim
            else
                selectedPlayers[p]   = true
                smartQueueBuilt      = false
                TweenService:Create(row, TweenInfo.new(0.15),
                    {BackgroundColor3 = Color3.fromRGB(30, 80, 55)}):Play()
                stroke(row, C.green, 1)
                dot.BackgroundColor3 = C.green
            end
        end)
    end
end

-- Auto-clean departed players
Players.PlayerRemoving:Connect(function(p)
    selectedPlayers[p] = nil
    smartQueueBuilt    = false
    if playerButtons[p] then
        local b = playerButtons[p].btn
        if b and b.Parent then b:Destroy() end
        playerButtons[p] = nil
    end
    -- if they were the current follow target, reset
    if currentTarget == p then
        currentTarget = nil
    end
end)

refreshBtn.MouseButton1Click:Connect(refreshPlayers)
refreshPlayers()

-------------------------------------------------
-- ╔══════════════════════════╗
-- ║   COMBAT TAB CONTENT     ║
-- ╚══════════════════════════╝
-------------------------------------------------

-- ── AUTO PUNCH ───────────────────────────────
secLbl("AUTO PUNCH", 8, combatFrame)

local getPunch, _, punchIntBox
do
    local g, s, b = combatRowWithInt(combatFrame, "Punch  (LMB click)", 24, "0.20", C.green)
    getPunch   = g
    punchIntBox = b
end

divLine(60, combatFrame)

-- ── MOVE ABILITIES ───────────────────────────
secLbl("MOVE ABILITIES  (Keys 1 – 4)", 66, combatFrame)

local getMoves    = {}   -- [1..4] = getter
local moveIntBoxes = {}  -- [1..4] = TextBox

for i = 1, 4 do
    local y = 82 + (i - 1) * 36
    local keyName = tostring(i)
    local g, _, b = combatRowWithInt(combatFrame,
        "Move " .. i .. "  (Key " .. keyName .. ")", y, "0.50", C.accent)
    getMoves[i]     = g
    moveIntBoxes[i] = b
end

-- Move 4 bottom = 82 + 3*36 + 28 = 218 → divider at 226
divLine(226, combatFrame)

-- ── SPECIAL ──────────────────────────────────
secLbl("SPECIAL", 232, combatFrame)

-- Auto Block (hold F)
local getBlock
do
    local g, _ = combatRowToggle(combatFrame, "Auto Block  (hold F key)", 248, C.orange)
    getBlock = g
end

-- Auto Ultimate (tap G on interval)
local getUlt, _, ultIntBox
do
    local g, s, b = combatRowWithInt(combatFrame, "Auto Ultimate  (key G)", 284, "3.00", C.red)
    getUlt   = g
    ultIntBox = b
end

divLine(320, combatFrame)

-- ── SMART FEATURES ───────────────────────────
secLbl("SMART FEATURES", 326, combatFrame)

-- Smart Ultimate: fire G once when self HP < threshold
lbl("Smart Ult  (G when own HP <",
    UDim2.new(0, 175, 0, 28), UDim2.new(0, 14, 0, 342), combatFrame, nil, 12)
local smartUltHPBox = ibox("20", UDim2.new(0, 36, 0, 28), UDim2.new(0, 192, 0, 342), combatFrame)
local hpPctLbl = lbl("%)", UDim2.new(0, 16, 0, 28), UDim2.new(0, 230, 0, 342), combatFrame, nil, 12)
hpPctLbl.TextColor3 = C.textDim
local getSmartUlt
do
    local _, get, _ = makeToggle(UDim2.new(0, 60, 0, 28), UDim2.new(0, WIN_W - 74, 0, 342), combatFrame, C.red)
    getSmartUlt = get
end

-- Smart Target: cycle selected players by HP (lowest → highest), refreshing each cycle
lbl("Smart Target  (lowest HP → highest)",
    UDim2.new(0, 220, 0, 28), UDim2.new(0, 14, 0, 378), combatFrame, nil, 12)
local getSmartTarg
do
    local _, get, _ = makeToggle(UDim2.new(0, 60, 0, 28), UDim2.new(0, WIN_W - 74, 0, 378), combatFrame, C.accent)
    getSmartTarg = get
end

-- Note at bottom
local noteLabel = lbl("Smart Target overrides Stay Time cycling.",
    UDim2.new(1, -20, 0, 14), UDim2.new(0, 14, 0, 412), combatFrame, nil, 9)
noteLabel.TextColor3 = C.textDim
noteLabel.Font = Enum.Font.Gotham

-------------------------------------------------
-- FOLLOW LOGIC
-------------------------------------------------

-- Returns current HP of a player (math.huge if unavailable)
local function getHP(p)
    if not p or not p.Character then return math.huge end
    local h = p.Character:FindFirstChildOfClass("Humanoid")
    return h and h.Health or math.huge
end

-- Sort selected players by HP ascending and store as queue
local function buildSmartQueue()
    local t = {}
    for p, _ in pairs(selectedPlayers) do
        if Players:FindFirstChild(p.Name) then
            table.insert(t, p)
        end
    end
    table.sort(t, function(a, b) return getHP(a) < getHP(b) end)
    smartQueue      = t
    smartQueueIdx   = 1
    smartQueueBuilt = true
end

-- Tween character behind a target
local function moveBehind(target)
    local myChar   = player.Character
    local targChar = target.Character
    if not myChar or not targChar then return end

    local myRoot   = myChar:FindFirstChild("HumanoidRootPart")
    local targRoot = targChar:FindFirstChild("HumanoidRootPart")
    if not myRoot or not targRoot then return end

    local dist  = tonumber(distBox.Text)  or cfg.followDistance
    local speed = tonumber(tweenBox.Text) or cfg.tweenSpeed

    -- Multiply in local space: +Z = behind target, orientation = matches target's
    local goalCF = targRoot.CFrame * CFrame.new(0, 0, dist)

    if activeTween then activeTween:Cancel() end
    activeTween = TweenService:Create(myRoot,
        TweenInfo.new(speed, Enum.EasingStyle.Linear, Enum.EasingDirection.Out),
        { CFrame = goalCF })
    activeTween:Play()
end

-------------------------------------------------
-- MAIN FOLLOW LOOP  (Heartbeat for accuracy)
-------------------------------------------------

RunService.Heartbeat:Connect(function()
    -- Build valid target list (skip players who left)
    local targets = {}
    for p, _ in pairs(selectedPlayers) do
        if Players:FindFirstChild(p.Name) then
            table.insert(targets, p)
        end
    end

    if #targets == 0 then
        if currentTarget then
            currentTarget = nil
            setStatus("Idle — no targets selected")
        end
        return
    end

    local stay = tonumber(timeBox.Text) or cfg.stayTime

    -- ── Smart Target mode ──────────────────────
    if getSmartTarg() then
        -- Rebuild queue if stale or at start of cycle
        if not smartQueueBuilt then
            buildSmartQueue()
        end

        -- Remove any players who have left
        while smartQueueIdx <= #smartQueue
          and not Players:FindFirstChild(smartQueue[smartQueueIdx].Name) do
            table.remove(smartQueue, smartQueueIdx)
        end

        if #smartQueue == 0 then
            setStatus("Smart Target — waiting for targets")
            return
        end

        -- Set current target from queue
        if not currentTarget or currentTarget ~= smartQueue[smartQueueIdx] then
            currentTarget   = smartQueue[smartQueueIdx]
            targetStartTime = tick()
        end

        moveBehind(currentTarget)
        setStatus(string.format("Smart→ %s  (%.0f HP)", currentTarget.Name, getHP(currentTarget)), C.accent)

        -- Advance after stayTime
        if tick() - targetStartTime >= stay then
            smartQueueIdx = smartQueueIdx + 1
            if smartQueueIdx > #smartQueue then
                -- Completed a full cycle — rebuild with updated HP values
                buildSmartQueue()
            end
            currentTarget   = smartQueue[smartQueueIdx] or targets[1]
            targetStartTime = tick()
        end
        return
    end

    -- ── Normal timed-cycle mode ────────────────
    smartQueueBuilt = false   -- reset so smart mode re-sorts when re-enabled

    if not currentTarget or not selectedPlayers[currentTarget]
       or not Players:FindFirstChild(currentTarget.Name) then
        currentTarget   = targets[1]
        targetStartTime = tick()
    end

    if tick() - targetStartTime >= stay then
        local idx = table.find(targets, currentTarget) or 0
        idx           = (idx % #targets) + 1
        currentTarget   = targets[idx]
        targetStartTime = tick()
    end

    moveBehind(currentTarget)
    setStatus("Following: " .. currentTarget.Name, C.green)
end)

-------------------------------------------------
-- COMBAT LOOP  (Heartbeat)
-------------------------------------------------

RunService.Heartbeat:Connect(function()
    local now = tick()

    -- ── Auto Punch ─────────────────────────────
    if getPunch() then
        local interval = tonumber(punchIntBox.Text) or 0.2
        if now - lastPunch >= interval then
            clickLMB()
            lastPunch = now
        end
    end

    -- ── Auto Moves 1-4 ─────────────────────────
    for i = 1, 4 do
        if getMoves[i]() then
            local interval = tonumber(moveIntBoxes[i].Text) or 0.5
            if now - lastMove[i] >= interval then
                tapKey(MOVE_KEYS[i])
                lastMove[i] = now
            end
        end
    end

    -- ── Auto Block (hold F) ────────────────────
    if getBlock() then
        if not blockHeld then
            holdKey(BLOCK_KEY)
            blockHeld = true
        end
    else
        if blockHeld then
            releaseKey(BLOCK_KEY)
            blockHeld = false
        end
    end

    -- ── Auto Ultimate ──────────────────────────
    if getUlt() then
        local interval = tonumber(ultIntBox.Text) or 3.0
        if now - lastUlt >= interval then
            tapKey(ULT_KEY)
            lastUlt = now
        end
    end

    -- ── Smart Ultimate ─────────────────────────
    -- Fires G exactly once when self HP drops below threshold,
    -- resets the flag when HP recovers so it can fire again
    if getSmartUlt() then
        local myChar = player.Character
        if myChar then
            local hum = myChar:FindFirstChildOfClass("Humanoid")
            if hum and hum.MaxHealth > 0 then
                local pct       = (hum.Health / hum.MaxHealth) * 100
                local threshold = tonumber(smartUltHPBox.Text) or 20
                if pct < threshold and not smartUltFired then
                    tapKey(ULT_KEY)
                    smartUltFired = true
                elseif pct >= threshold then
                    smartUltFired = false   -- allow re-trigger on next HP drop
                end
            end
        end
    end
end)
