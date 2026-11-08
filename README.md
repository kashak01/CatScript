# CatScript
--[[
    CATScript Visual — Modern Dashboard
    Visual and Settings pages with Hover Highlight, editable color picker, keybind, and optional auto-save.

    Menu controls:
      Configurable keybind (default RightShift) - show/hide menu
      Drag top bar - move menu
      — - collapse/expand content
      × - hide menu
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")
local PathfindingService = game:GetService("PathfindingService")
local StatsService
pcall(function() StatsService = game:GetService("Stats") end)

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then
    warn("[CATScript] LocalPlayer not found. Run from a client executor.")
    return
end

local GUI_NAME = "CATScriptVisual_ModernDashboard"

-- Keep a single active copy of CATScript. Older injected copies can otherwise
-- leave white ring parts and Highlights in Workspace, making recoloring look broken.
local RuntimeEnvironment = (type(getgenv) == "function" and getgenv()) or _G
local RuntimeKey = "__CATScriptVisualRuntime"
local PreviousRuntime = RuntimeEnvironment[RuntimeKey]
if type(PreviousRuntime) == "table" and type(PreviousRuntime.stop) == "function" then
    pcall(PreviousRuntime.stop)
end
local RuntimeState = {active = true}
RuntimeEnvironment[RuntimeKey] = RuntimeState
-- Remove legacy angel-wing geometry left by older CATScript versions.
if LocalPlayer.Character then
    local previousWings = LocalPlayer.Character:FindFirstChild("CATScript_AngelWings")
    if previousWings then pcall(function() previousWings:Destroy() end) end
end

-- Optional executor file persistence. Auto-save is enabled by default and can
-- be disabled in Settings. When disabled, later changes are not written.
local CONFIG_FILE = "CATScriptVisual_Settings.json"
local SavedConfig = {}
if type(readfile) == "function" then
    local canRead = true
    if type(isfile) == "function" then
        local existsOk, exists = pcall(isfile, CONFIG_FILE)
        canRead = existsOk and exists == true
    end
    if canRead then
        local readOk, raw = pcall(readfile, CONFIG_FILE)
        if readOk and type(raw) == "string" and raw ~= "" then
            local decodeOk, decoded = pcall(function()
                return HttpService:JSONDecode(raw)
            end)
            if decodeOk and type(decoded) == "table" then
                SavedConfig = decoded
            end
        end
    end
end

-- User-selected full-window background image asset ID. Read the former banner
-- key as a one-time compatibility fallback so existing users keep their image.
local backgroundAssetId = ""
local storedBackgroundId = SavedConfig.backgroundAssetId or SavedConfig.bannerAssetId
if type(storedBackgroundId) == "string" then
    storedBackgroundId = storedBackgroundId:match("^%s*(.-)%s*$")
    if storedBackgroundId and storedBackgroundId:match("^%d+$") then
        backgroundAssetId = storedBackgroundId
    end
end

local autoSaveEnabled = true
if type(SavedConfig.autoSave) == "boolean" then
    autoSaveEnabled = SavedConfig.autoSave
end

-- Client utility preferences. Freecam is intentionally session-only so a saved
-- config can never unexpectedly leave the player's camera detached on startup.
RuntimeState.Utility = {
    antiAfkEnabled = SavedConfig.antiAfk == true,
    noTextureEnabled = SavedConfig.noTexture == true,
    noGuiEnabled = SavedConfig.noGui == true,
    autoSessionRewardEnabled = SavedConfig.autoSessionReward == true,
    autoSkipCratesEnabled = SavedConfig.autoSkipCrates == true,
    noGui = {
        enabled = false,
        originals = {},
        coreGuiOriginals = {},
        playerGuiAddedConnection = nil,
        workspaceAddedConnection = nil,
    },
    freecam = {
        enabled = false,
        keys = {},
        speed = math.clamp(tonumber(SavedConfig.freecamSpeed) or 48, 10, 1000),
        sensitivity = 0.0025,
        rotating = false,
        originalCameras = {},
    },
    fly = {
        enabled = false,
        keys = {},
        speed = math.clamp(tonumber(SavedConfig.flySpeed) or 80, 10, 1000),
        connections = {},
        humanoid = nil,
        root = nil,
        bodyVelocity = nil,
        bodyGyro = nil,
        humanoidOriginal = nil,
        playerControls = nil,
    },
    noTexture = {
        enabled = false,
        generation = 0,
        originals = {},
        addedConnection = nil,
    },
    antiAfkConnection = nil,
}

local SavedPlayer = type(SavedConfig.player) == "table" and SavedConfig.player or {}
RuntimeState.PlayerTools = {
    walkSpeed = math.clamp(tonumber(SavedPlayer.walkSpeed) or 16, 10, 1000),
    jumpPower = math.clamp(tonumber(SavedPlayer.jumpPower) or 50, 10, 1000),
    vehicleFlySpeed = math.clamp(tonumber(SavedPlayer.vehicleFlySpeed) or 80, 10, 1000),
    floatEnabled = false, noclipEnabled = false,
    fov = math.clamp(tonumber(SavedPlayer.fov) or 70, 30, 120),
    aimEnabled = false, aimFov = math.clamp(tonumber(SavedPlayer.aimFov) or 120, 10, 1000),
    aimSmoothness = math.clamp(tonumber(SavedPlayer.aimSmoothness) or 100, 10, 1000),
    aimRange = math.clamp(tonumber(SavedPlayer.aimRange) or 1000, 10, 10000),
    originals = {}, collisionOriginals = {}, noclipConnection = nil,
    floatPart = nil, floatConnection = nil, vehicleFly = {enabled=false, keys={}, connections={}, mover=nil, gyro=nil, seat=nil},
    aimConnections = {}, aimHeld = false,
}

local SavedWater = type(SavedConfig.water) == "table" and SavedConfig.water or {}
RuntimeState.Water = {
    enabled = SavedWater.enabled == true,
    transparency = math.clamp(tonumber(SavedWater.transparency) or 0.88, 0, 1),
    originals = {}, terrainTransparency = nil, terrainReflectance = nil,
    addedConnection = nil, generation = 0,
}
RuntimeState.NoFog = {
    enabled = false,
    savedEnabled = SavedConfig.noFog == true,
    fogStart = nil,
    fogEnd = nil,
    atmosphereOriginals = {},
    addedConnection = nil,
    generation = 0,
}
RuntimeState.Keystrokes = {enabled = SavedConfig.keystrokesEnabled == true, keys = {}, cells = {}, connections = {}}

-- Global UI font preference. Invalid or outdated saved names fall back safely.
RuntimeState.defaultFontName = "Gotham"
RuntimeState.selectedFontName = type(SavedConfig.fontName) == "string" and SavedConfig.fontName or RuntimeState.defaultFontName
RuntimeState.selectedFont = Enum.Font.Gotham
pcall(function()
    local storedFont = Enum.Font[RuntimeState.selectedFontName]
    if storedFont then
        RuntimeState.selectedFont = storedFont
    else
        RuntimeState.selectedFontName = RuntimeState.defaultFontName
    end
end)

-- Performance HUD preferences. GPU is represented by render frame time (ms),
-- not a fabricated hardware-utilization percentage.
local metricSettings = {
    fps = true,
    ping = true,
    gpu = true,
    memory = true,
}
if type(SavedConfig.metrics) == "table" then
    for key in pairs(metricSettings) do
        if type(SavedConfig.metrics[key]) == "boolean" then
            metricSettings[key] = SavedConfig.metrics[key]
        end
    end
end

-- Custom cursor preferences. The cursor itself is drawn with native GUI
-- primitives, so no external image assets or unknown asset IDs are required.
local cursorPalette = {
    Purple = Color3.fromRGB(166, 120, 255),
    White = Color3.fromRGB(245, 247, 255),
    Cyan = Color3.fromRGB(85, 220, 255),
    Pink = Color3.fromRGB(255, 115, 190),
    Lime = Color3.fromRGB(175, 255, 120),
}
-- Fifteen built-in cursor designs. All are drawn with Roblox GUI primitives,
-- so they need no uploaded image IDs and inherit the user's custom color.
local cursorStyles = {
    "Arrow", "Dot", "Ring", "Crosshair", "Cat",
    "Diamond", "Star", "Heart", "Plus", "Target",
    "Square", "Spark", "Flower", "Chevron", "Crown",
}
local cursorStyleLabels = {
    Arrow = "Стрелка", Dot = "Точка", Ring = "Кольцо", Crosshair = "Прицел", Cat = "Кот",
    Diamond = "Ромб", Star = "Звезда", Heart = "Сердце", Plus = "Плюс", Target = "Метка",
    Square = "Квадрат", Spark = "Искра", Flower = "Цветок", Chevron = "Шеврон", Crown = "Корона",
}
local storedCursor = type(SavedConfig.customCursor) == "table" and SavedConfig.customCursor or {}
local initialCursorStyle = type(storedCursor.style) == "string" and storedCursor.style or "Ring"
if not table.find(cursorStyles, initialCursorStyle) then initialCursorStyle = "Ring" end
local initialCursorColor = type(storedCursor.colorName) == "string" and storedCursor.colorName or "Purple"
if not cursorPalette[initialCursorColor] then initialCursorColor = "Purple" end
local initialCursorColorValue = cursorPalette[initialCursorColor]
if type(storedCursor.color) == "table" then
    local storedR, storedG, storedB = tonumber(storedCursor.color.r), tonumber(storedCursor.color.g), tonumber(storedCursor.color.b)
    if storedR and storedG and storedB then
        initialCursorColorValue = Color3.fromRGB(
            math.clamp(math.floor(storedR + 0.5), 0, 255),
            math.clamp(math.floor(storedG + 0.5), 0, 255),
            math.clamp(math.floor(storedB + 0.5), 0, 255)
        )
        local matchedPreset = false
        for presetName, presetColor in pairs(cursorPalette) do
            if math.abs(presetColor.R - initialCursorColorValue.R) < 0.002
                and math.abs(presetColor.G - initialCursorColorValue.G) < 0.002
                and math.abs(presetColor.B - initialCursorColorValue.B) < 0.002 then
                initialCursorColor = presetName
                matchedPreset = true
                break
            end
        end
        if not matchedPreset then initialCursorColor = "Custom" end
    end
end
local cursorSettings = {
    enabled = type(storedCursor.enabled) == "boolean" and storedCursor.enabled or false,
    style = initialCursorStyle,
    colorName = initialCursorColor,
    color = initialCursorColorValue,
    size = type(storedCursor.size) == "number" and math.clamp(storedCursor.size, 14, 40) or 24,
}

-- Clean effect instances left behind by older versions that did not have
-- singleton cleanup. Do not touch unrelated Workspace objects.
for _, object in ipairs(workspace:GetDescendants()) do
    if object.Name == "CATScript_PlayerESP"
        or object.Name == "CATScript_ESP"
        or object.Name == "CATScript_HoverBox"
        or object.Name == "CATScript_HoverRingGlow"
        or object.Name == "CATScript_HoverRing"
        or object.Name == "CATScript_HoverHighlight"
        or object.Name == "CATScript_LocalCharacterTrail"
        or object.Name == "CATScript_LocalTrailLeft"
        or object.Name == "CATScript_LocalTrailRight"
        or object.Name == "CATScript_JumpCircleSegment"
        or object.Name == "CATScript_KasaHat"
        or object.Name == "CATScript_SamuraiHat"
        or object.Name == "CATScript_CustomAuraEffects" then
        pcall(function() object:Destroy() end)
    end
end

-- Remove an older copy of this menu.
pcall(function()
    local old = CoreGui:FindFirstChild(GUI_NAME)
    if old then old:Destroy() end
end)
pcall(function()
    if type(gethui) == "function" then
        local hui = gethui()
        local old = hui and hui:FindFirstChild(GUI_NAME)
        if old then old:Destroy() end
    end
end)

local COLORS = {
    background = Color3.fromRGB(10, 14, 17),
    sidebar = Color3.fromRGB(12, 17, 20),
    topbar = Color3.fromRGB(13, 18, 21),
    panel = Color3.fromRGB(17, 23, 27),
    panel2 = Color3.fromRGB(20, 27, 31),
    outline = Color3.fromRGB(36, 46, 51),
    text = Color3.fromRGB(231, 237, 240),
    muted = Color3.fromRGB(133, 146, 153),
    dim = Color3.fromRGB(77, 90, 97),
    accent = Color3.fromRGB(134, 111, 244),
    white = Color3.fromRGB(255, 255, 255),
}

-- Fully editable UI palette. Saved RGB values are validated before being used.
RuntimeState.THEME_KEYS = {"accent", "background", "sidebar", "topbar", "panel", "panel2", "outline", "text", "muted", "dim", "white"}
RuntimeState.DefaultThemeColors = {
    accent = Color3.fromRGB(134, 111, 244),
    background = Color3.fromRGB(10, 14, 17),
    sidebar = Color3.fromRGB(12, 17, 20),
    topbar = Color3.fromRGB(13, 18, 21),
    panel = Color3.fromRGB(17, 23, 27),
    panel2 = Color3.fromRGB(20, 27, 31),
    outline = Color3.fromRGB(36, 46, 51),
    text = Color3.fromRGB(231, 237, 240),
    muted = Color3.fromRGB(133, 146, 153),
    dim = Color3.fromRGB(77, 90, 97),
    white = Color3.fromRGB(255, 255, 255),
}
-- Saved theme colors are applied after the UI palette roles are registered.
-- This preserves the identity of each role even when two saved colors are equal.
RuntimeState.menuTransparency = type(SavedConfig.menuTransparency) == "number"
    and math.clamp(SavedConfig.menuTransparency, 0, 0.70) or 0

local function make(className, properties, parent)
    local object = Instance.new(className)
    for key, value in pairs(properties or {}) do
        object[key] = value
    end
    if parent then object.Parent = parent end
    return object
end

local function corner(parent, radius)
    return make("UICorner", {CornerRadius = UDim.new(0, radius or 8)}, parent)
end

local function stroke(parent, color, transparency, thickness)
    return make("UIStroke", {
        Color = color or COLORS.outline,
        Transparency = transparency or 0,
        Thickness = thickness or 1,
    }, parent)
end

local function addLabel(parent, text, size, color, font, xAlign)
    return make("TextLabel", {
        BackgroundTransparency = 1,
        Text = text,
        TextColor3 = color or COLORS.text,
        TextSize = size or 13,
        Font = font or Enum.Font.Gotham,
        TextXAlignment = xAlign or Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center,
        Size = UDim2.new(1, 0, 0, 20),
    }, parent)
end

local function tween(object, properties, duration)
    pcall(function()
        TweenService:Create(
            object,
            TweenInfo.new(duration or 0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            properties
        ):Play()
    end)
end

-- Choose a GUI parent supported by common client executors.
local guiParent
local okHui, hui = pcall(function()
    if type(gethui) == "function" then return gethui() end
    return nil
end)
if okHui and hui then
    guiParent = hui
else
    guiParent = CoreGui
end

local ScreenGui = make("ScreenGui", {
    Name = GUI_NAME,
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    DisplayOrder = 900,
}, guiParent)

-- Normalize mouse/touch positions into the same coordinate space as GuiObject.AbsolutePosition.
-- Using one conversion for initial press and all drag updates prevents the picker marker
-- from offsetting when moving between InputObject.Position and GetMouseLocation().
local function getColorEditorPointer(input)
    local point
    if input and input.UserInputType == Enum.UserInputType.Touch then
        point = Vector2.new(input.Position.X, input.Position.Y)
    else
        local mousePoint = UserInputService:GetMouseLocation()
        point = Vector2.new(mousePoint.X, mousePoint.Y)
    end
    if not ScreenGui.IgnoreGuiInset then
        local inset = game:GetService("GuiService"):GetGuiInset()
        point -= inset
    end
    return point
end

-- Weather is rendered with local 3D objects in Workspace, not as a screen overlay.

-- Detached color editors live at ScreenGui level, not inside the main Window.
-- They remain independently usable while the dashboard itself is hidden.
RuntimeState.floatingEditorsLayer = make("Frame", {
    Name = "FloatingColorEditorsLayer",
    Position = UDim2.fromScale(0, 0),
    Size = UDim2.fromScale(1, 1),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Active = false,
    ZIndex = 100,
}, ScreenGui)

-- Custom cursor overlay. Active=false on every primitive keeps clicks routed
-- to the actual game/UI controls underneath the visual cursor.
RuntimeState.CustomCursorOverlay = make("Frame", {
    Name = "CATScriptCustomCursorOverlay",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(40, 40),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Active = false,
    Visible = false,
    ZIndex = 1500,
}, ScreenGui)

local function clearCustomCursorVisual()
    for _, child in ipairs(RuntimeState.CustomCursorOverlay:GetChildren()) do
        pcall(function() child:Destroy() end)
    end
end

local function makeCursorShape(name, x, y, width, height, color, isCircle, outlined)
    local shape = make("Frame", {
        Name = name,
        Position = UDim2.fromOffset(x, y),
        Size = UDim2.fromOffset(width, height),
        BackgroundColor3 = color,
        BackgroundTransparency = outlined and 1 or 0,
        BorderSizePixel = 0,
        Active = false,
        Selectable = false,
        ZIndex = 1501,
    }, RuntimeState.CustomCursorOverlay)
    if isCircle then corner(shape, math.ceil(math.min(width, height) / 2)) end
    if outlined then stroke(shape, color, 0, math.max(1, math.floor(cursorSettings.size / 12))) end
    return shape
end

local function makeCursorGlyph(glyph, color, size)
    return make("TextLabel", {
        Name = "CursorGlyph_" .. glyph,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Position = UDim2.fromScale(0, 0),
        Size = UDim2.fromScale(1, 1),
        Text = glyph,
        TextColor3 = color,
        TextSize = math.max(10, math.floor(size + 0.5)),
        Font = Enum.Font.GothamBold,
        TextScaled = false,
        TextXAlignment = Enum.TextXAlignment.Center,
        TextYAlignment = Enum.TextYAlignment.Center,
        TextStrokeColor3 = Color3.new(0, 0, 0),
        TextStrokeTransparency = 0.45,
        Active = false,
        Selectable = false,
        ZIndex = 1502,
    }, RuntimeState.CustomCursorOverlay)
end

RuntimeState.rebuildCustomCursor = function()
    clearCustomCursorVisual()
    local size = math.clamp(cursorSettings.size, 14, 40)
    local color = cursorSettings.color
    RuntimeState.CustomCursorOverlay.Size = UDim2.fromOffset(size, size)

    if cursorSettings.style == "Dot" then
        makeCursorShape("CursorDotOuter", size * 0.25, size * 0.25, size * 0.50, size * 0.50, color, true, true)
        makeCursorShape("CursorDotCore", size * 0.42, size * 0.42, size * 0.16, size * 0.16, color, true, false)
    elseif cursorSettings.style == "Ring" then
        local diameter = size * 0.72
        makeCursorShape("CursorRingOuter", (size - diameter) * 0.5, (size - diameter) * 0.5, diameter, diameter, color, true, true)
        makeCursorShape("CursorRingCore", size * 0.44, size * 0.44, size * 0.12, size * 0.12, color, true, false)
        local tick = math.max(2, size * 0.10)
        makeCursorShape("CursorRingTop", size * 0.47, size * 0.02, size * 0.06, tick, color, false, false)
        makeCursorShape("CursorRingBottom", size * 0.47, size * 0.92, size * 0.06, tick, color, false, false)
        makeCursorShape("CursorRingLeft", size * 0.02, size * 0.47, tick, size * 0.06, color, false, false)
        makeCursorShape("CursorRingRight", size * 0.92, size * 0.47, tick, size * 0.06, color, false, false)
    elseif cursorSettings.style == "Crosshair" then
        local line = math.max(2, size * 0.075)
        local gap = size * 0.16
        local mid = size * 0.5
        makeCursorShape("CursorCrossTop", mid - line * 0.5, size * 0.05, line, math.max(2, mid - gap - size * 0.05), color, false, false)
        makeCursorShape("CursorCrossBottom", mid - line * 0.5, mid + gap, line, math.max(2, size * 0.45 - gap), color, false, false)
        makeCursorShape("CursorCrossLeft", size * 0.05, mid - line * 0.5, math.max(2, mid - gap - size * 0.05), line, color, false, false)
        makeCursorShape("CursorCrossRight", mid + gap, mid - line * 0.5, math.max(2, size * 0.45 - gap), line, color, false, false)
        makeCursorShape("CursorCrossCenter", mid - line, mid - line, line * 2, line * 2, color, true, false)
    elseif cursorSettings.style == "Cat" then
        -- Minimal cat-face cursor built from simple strokes and dots.
        local faceW, faceH = size * 0.58, size * 0.48
        local faceX, faceY = size * 0.21, size * 0.39
        makeCursorShape("CursorCatFace", faceX, faceY, faceW, faceH, color, true, true)
        local earW = size * 0.20
        local leftEar = makeCursorShape("CursorCatEarLeft", size * 0.19, size * 0.18, earW, math.max(2, size * 0.09), color, false, false)
        leftEar.Rotation = -48
        local rightEar = makeCursorShape("CursorCatEarRight", size * 0.61, size * 0.18, earW, math.max(2, size * 0.09), color, false, false)
        rightEar.Rotation = 48
        makeCursorShape("CursorCatEyeLeft", size * 0.38, size * 0.55, math.max(2, size * 0.07), math.max(2, size * 0.07), color, true, false)
        makeCursorShape("CursorCatEyeRight", size * 0.57, size * 0.55, math.max(2, size * 0.07), math.max(2, size * 0.07), color, true, false)
        makeCursorShape("CursorCatNose", size * 0.49, size * 0.67, math.max(2, size * 0.06), math.max(2, size * 0.05), color, true, false)
        makeCursorShape("CursorCatWhiskerLeft", size * 0.02, size * 0.64, size * 0.19, math.max(1, size * 0.035), color, false, false)
        makeCursorShape("CursorCatWhiskerRight", size * 0.79, size * 0.64, size * 0.19, math.max(1, size * 0.035), color, false, false)
    elseif cursorSettings.style == "Arrow" then
        makeCursorGlyph("➤", color, size * 0.88)
    elseif cursorSettings.style == "Diamond" then
        local diamond = makeCursorShape("CursorDiamond", size * 0.25, size * 0.25, size * 0.50, size * 0.50, color, false, true)
        diamond.Rotation = 45
        makeCursorShape("CursorDiamondCore", size * 0.44, size * 0.44, size * 0.12, size * 0.12, color, true, false)
    elseif cursorSettings.style == "Star" then
        makeCursorGlyph("★", color, size * 0.92)
    elseif cursorSettings.style == "Heart" then
        makeCursorGlyph("♥", color, size * 0.90)
    elseif cursorSettings.style == "Plus" then
        local arm = size * 0.16
        makeCursorShape("CursorPlusVertical", size * 0.5 - arm * 0.5, size * 0.12, arm, size * 0.76, color, true, false)
        makeCursorShape("CursorPlusHorizontal", size * 0.12, size * 0.5 - arm * 0.5, size * 0.76, arm, color, true, false)
    elseif cursorSettings.style == "Target" then
        makeCursorShape("CursorTargetOuter", size * 0.12, size * 0.12, size * 0.76, size * 0.76, color, true, true)
        makeCursorShape("CursorTargetInner", size * 0.30, size * 0.30, size * 0.40, size * 0.40, color, true, true)
        makeCursorShape("CursorTargetCenter", size * 0.44, size * 0.44, size * 0.12, size * 0.12, color, true, false)
        makeCursorShape("CursorTargetTop", size * 0.47, 0, size * 0.06, size * 0.16, color, false, false)
        makeCursorShape("CursorTargetBottom", size * 0.47, size * 0.84, size * 0.06, size * 0.16, color, false, false)
    elseif cursorSettings.style == "Square" then
        makeCursorShape("CursorSquareFrame", size * 0.18, size * 0.18, size * 0.64, size * 0.64, color, false, true)
        makeCursorShape("CursorSquareCore", size * 0.43, size * 0.43, size * 0.14, size * 0.14, color, true, false)
    elseif cursorSettings.style == "Spark" then
        local beam = math.max(2, size * 0.09)
        makeCursorShape("CursorSparkVertical", size * 0.5 - beam * 0.5, size * 0.08, beam, size * 0.84, color, true, false)
        makeCursorShape("CursorSparkHorizontal", size * 0.08, size * 0.5 - beam * 0.5, size * 0.84, beam, color, true, false)
        local diagonalA = makeCursorShape("CursorSparkDiagonalA", size * 0.5 - beam * 0.5, size * 0.13, beam, size * 0.74, color, true, false)
        diagonalA.Rotation = 45
        local diagonalB = makeCursorShape("CursorSparkDiagonalB", size * 0.5 - beam * 0.5, size * 0.13, beam, size * 0.74, color, true, false)
        diagonalB.Rotation = -45
    elseif cursorSettings.style == "Flower" then
        local petal = size * 0.30
        makeCursorShape("CursorFlowerTop", size * 0.35, size * 0.06, petal, petal, color, true, false)
        makeCursorShape("CursorFlowerBottom", size * 0.35, size * 0.64, petal, petal, color, true, false)
        makeCursorShape("CursorFlowerLeft", size * 0.06, size * 0.35, petal, petal, color, true, false)
        makeCursorShape("CursorFlowerRight", size * 0.64, size * 0.35, petal, petal, color, true, false)
        makeCursorShape("CursorFlowerCenter", size * 0.39, size * 0.39, size * 0.22, size * 0.22, color, true, false)
    elseif cursorSettings.style == "Chevron" then
        local armLength = size * 0.48
        local armWidth = math.max(2, size * 0.10)
        local leftArm = makeCursorShape("CursorChevronLeft", size * 0.20, size * 0.29, armWidth, armLength, color, true, false)
        leftArm.Rotation = -42
        local rightArm = makeCursorShape("CursorChevronRight", size * 0.60, size * 0.29, armWidth, armLength, color, true, false)
        rightArm.Rotation = 42
    elseif cursorSettings.style == "Crown" then
        makeCursorShape("CursorCrownBase", size * 0.18, size * 0.67, size * 0.64, size * 0.13, color, true, false)
        makeCursorShape("CursorCrownLeft", size * 0.20, size * 0.28, size * 0.14, size * 0.47, color, true, false)
        makeCursorShape("CursorCrownMiddle", size * 0.43, size * 0.12, size * 0.14, size * 0.63, color, true, false)
        makeCursorShape("CursorCrownRight", size * 0.66, size * 0.28, size * 0.14, size * 0.47, color, true, false)
        local crownLeftTip = makeCursorShape("CursorCrownLeftTip", size * 0.10, size * 0.20, size * 0.22, size * 0.10, color, true, false)
        crownLeftTip.Rotation = -30
        local crownRightTip = makeCursorShape("CursorCrownRightTip", size * 0.68, size * 0.20, size * 0.22, size * 0.10, color, true, false)
        crownRightTip.Rotation = 30
    end
end

RuntimeState.setCustomCursorEnabled = function(enabled)
    cursorSettings.enabled = enabled == true
    if cursorSettings.enabled then
        if RuntimeState.previousMouseIconEnabled == nil then
            local readOk, previous = pcall(function() return UserInputService.MouseIconEnabled end)
            RuntimeState.previousMouseIconEnabled = readOk and previous or true
        end
        pcall(function() UserInputService.MouseIconEnabled = false end)
        RuntimeState.CustomCursorOverlay.Visible = true
        RuntimeState.rebuildCustomCursor()
    else
        RuntimeState.CustomCursorOverlay.Visible = false
        if RuntimeState.previousMouseIconEnabled ~= nil then
            local previous = RuntimeState.previousMouseIconEnabled
            pcall(function() UserInputService.MouseIconEnabled = previous end)
            RuntimeState.previousMouseIconEnabled = nil
        end
    end
end

RuntimeState.customCursorRenderConnection = RunService.RenderStepped:Connect(function()
    if not RuntimeState.active or not cursorSettings.enabled then return end
    local mouseEnabled = true
    pcall(function() mouseEnabled = UserInputService.MouseEnabled end)
    if not mouseEnabled then
        RuntimeState.CustomCursorOverlay.Visible = false
        return
    end
    RuntimeState.CustomCursorOverlay.Visible = true
    local mousePosition = UserInputService:GetMouseLocation()
    RuntimeState.CustomCursorOverlay.Position = UDim2.fromOffset(mousePosition.X, mousePosition.Y)
end)
if cursorSettings.enabled then
    RuntimeState.setCustomCursorEnabled(true)
end

local viewport = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1280, 720)
local scale = math.min((viewport.X - 24) / 1120, (viewport.Y - 24) / 570, 1)
scale = math.max(scale, 0.55)

local Window = make("Frame", {
    Name = "Window",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(1120, 570),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
    ClipsDescendants = true,
    Visible = false,
}, ScreenGui)

RuntimeState.LoadingOverlay = make("Frame", {
    Name = "CATScriptTerminalBootOverlay",
    Size = UDim2.fromScale(1, 1),
    Position = UDim2.fromScale(0, 0),
    BackgroundColor3 = Color3.fromRGB(5, 4, 11),
    BackgroundTransparency = 0.02,
    BorderSizePixel = 0,
    ZIndex = 1000,
}, ScreenGui)
RuntimeState.LoadingCard = make("Frame", {
    Name = "CATScriptTerminalBoot",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(590, 326),
    BackgroundColor3 = Color3.fromRGB(10, 8, 19),
    BackgroundTransparency = 0.02,
    BorderSizePixel = 0,
    ZIndex = 1001,
}, RuntimeState.LoadingOverlay)
corner(RuntimeState.LoadingCard, 12)
stroke(RuntimeState.LoadingCard, COLORS.accent, 0.12, 1.4)
RuntimeState.LoadingTopbar = make("Frame", {
    Name = "TerminalTopbar",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, 0, 0, 42),
    BackgroundColor3 = Color3.fromRGB(20, 15, 32),
    BorderSizePixel = 0,
    ZIndex = 1002,
}, RuntimeState.LoadingCard)
corner(RuntimeState.LoadingTopbar, 12)
make("Frame", {
    Position = UDim2.new(0, 0, 1, -12),
    Size = UDim2.new(1, 0, 0, 12),
    BackgroundColor3 = Color3.fromRGB(20, 15, 32),
    BorderSizePixel = 0,
    ZIndex = 1002,
}, RuntimeState.LoadingTopbar)
RuntimeState.LoadingWindowDots = addLabel(RuntimeState.LoadingTopbar, "●  ●  ●", 10, COLORS.accent, Enum.Font.GothamBold)
RuntimeState.LoadingWindowDots.Position = UDim2.fromOffset(15, 0)
RuntimeState.LoadingWindowDots.Size = UDim2.fromOffset(56, 42)
RuntimeState.LoadingWindowDots.ZIndex = 1003
RuntimeState.LoadingTitle = addLabel(RuntimeState.LoadingTopbar, "CATScript  //  TERMINAL BOOT", 11, COLORS.text, Enum.Font.Code)
RuntimeState.LoadingTitle.Position = UDim2.fromOffset(82, 0)
RuntimeState.LoadingTitle.Size = UDim2.new(1, -180, 1, 0)
RuntimeState.LoadingTitle.ZIndex = 1003
RuntimeState.LoadingMode = addLabel(RuntimeState.LoadingTopbar, "CLIENT", 8, COLORS.accent, Enum.Font.Code, Enum.TextXAlignment.Right)
RuntimeState.LoadingMode.Position = UDim2.new(1, -76, 0, 0)
RuntimeState.LoadingMode.Size = UDim2.fromOffset(58, 42)
RuntimeState.LoadingMode.ZIndex = 1003
make("Frame", {
    Position = UDim2.fromOffset(18, 54),
    Size = UDim2.new(1, -36, 0, 1),
    BackgroundColor3 = COLORS.outline,
    BackgroundTransparency = 0.15,
    BorderSizePixel = 0,
    ZIndex = 1002,
}, RuntimeState.LoadingCard)
RuntimeState.LoadingSubtitle = addLabel(RuntimeState.LoadingCard, "> STARTING CATScript VISUAL RUNTIME", 10, COLORS.accent, Enum.Font.Code)
RuntimeState.LoadingSubtitle.Position = UDim2.fromOffset(22, 64)
RuntimeState.LoadingSubtitle.Size = UDim2.new(1, -44, 0, 18)
RuntimeState.LoadingSubtitle.ZIndex = 1003
RuntimeState.LoadingLogLines = {}
for loadingLineIndex = 1, 8 do
    local loadingLine = addLabel(RuntimeState.LoadingCard, "", 10, COLORS.muted, Enum.Font.Code)
    loadingLine.Name = "BootLogLine" .. loadingLineIndex
    loadingLine.Position = UDim2.fromOffset(26, 91 + (loadingLineIndex - 1) * 19)
    loadingLine.Size = UDim2.new(1, -52, 0, 18)
    loadingLine.ZIndex = 1003
    RuntimeState.LoadingLogLines[loadingLineIndex] = loadingLine
end
RuntimeState.LoadingTrack = make("Frame", {
    Position = UDim2.fromOffset(22, 264),
    Size = UDim2.new(1, -44, 0, 6),
    BackgroundColor3 = Color3.fromRGB(35, 28, 49),
    BorderSizePixel = 0,
    ZIndex = 1002,
}, RuntimeState.LoadingCard)
corner(RuntimeState.LoadingTrack, 5)
RuntimeState.LoadingFill = make("Frame", {
    Size = UDim2.fromScale(0.02, 1),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    ZIndex = 1003,
}, RuntimeState.LoadingTrack)
corner(RuntimeState.LoadingFill, 5)
RuntimeState.LoadingStatus = addLabel(RuntimeState.LoadingCard, "INITIALIZING...", 8, COLORS.muted, Enum.Font.Code)
RuntimeState.LoadingStatus.Position = UDim2.fromOffset(22, 278)
RuntimeState.LoadingStatus.Size = UDim2.new(1, -105, 0, 20)
RuntimeState.LoadingStatus.ZIndex = 1003
RuntimeState.LoadingPercent = addLabel(RuntimeState.LoadingCard, "00%", 10, COLORS.accent, Enum.Font.Code, Enum.TextXAlignment.Right)
RuntimeState.LoadingPercent.Position = UDim2.new(1, -72, 0, 278)
RuntimeState.LoadingPercent.Size = UDim2.fromOffset(50, 20)
RuntimeState.LoadingPercent.ZIndex = 1003
RuntimeState.LoadingFooter = addLabel(RuntimeState.LoadingCard, "LOCAL SESSION  •  PURPLE BUILD", 7, COLORS.dim, Enum.Font.Code)
RuntimeState.LoadingFooter.Position = UDim2.fromOffset(22, 302)
RuntimeState.LoadingFooter.Size = UDim2.new(1, -44, 0, 14)
RuntimeState.LoadingFooter.ZIndex = 1003

-- User image is the actual background of the entire menu window (not a banner).
-- The tint keeps text readable while allowing the selected image to show through
-- translucent dashboard surfaces.
local BackgroundImage = make("ImageLabel", {
    Name = "CATScriptFullWindowBackground",
    Position = UDim2.fromScale(0, 0),
    Size = UDim2.fromScale(1, 1),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Image = backgroundAssetId ~= "" and ("rbxassetid://" .. backgroundAssetId) or "",
    ScaleType = Enum.ScaleType.Crop,
    ImageTransparency = 0.02,
    Visible = backgroundAssetId ~= "",
    ZIndex = 0,
}, Window)
corner(BackgroundImage, 12)
local BackgroundTint = make("Frame", {
    Name = "CATScriptBackgroundTint",
    Position = UDim2.fromScale(0, 0),
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.fromRGB(5, 7, 10),
    BackgroundTransparency = backgroundAssetId ~= "" and 0.58 or 1,
    BorderSizePixel = 0,
    Visible = backgroundAssetId ~= "",
    ZIndex = 0,
}, Window)
corner(BackgroundTint, 12)

make("UIScale", {Scale = scale}, Window)
corner(Window, 12)
stroke(Window, COLORS.outline, 0, 1)

-- Sidebar: Visual, Player and Settings pages.
local Sidebar = make("Frame", {
    Name = "Sidebar",
    Size = UDim2.new(0, 174, 1, 0),
    BackgroundColor3 = COLORS.sidebar,
    BackgroundTransparency = 0.24,
    BorderSizePixel = 0,
}, Window)
make("Frame", {
    Position = UDim2.new(1, -1, 0, 0),
    Size = UDim2.new(0, 1, 1, 0),
    BackgroundColor3 = COLORS.outline,
    BorderSizePixel = 0,
}, Sidebar)

local Brand = make("Frame", {
    BackgroundTransparency = 1,
    Position = UDim2.fromOffset(14, 12),
    Size = UDim2.new(1, -28, 0, 44),
}, Sidebar)
local BrandMark = make("ImageLabel", {
    Name = "BrandLogo",
    Size = UDim2.fromOffset(32, 32),
    Position = UDim2.fromOffset(-2, 5),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Image = "rbxassetid://11176073563",
    ScaleType = Enum.ScaleType.Fit,
    ImageColor3 = Color3.new(1, 1, 1),
}, Brand)
corner(BrandMark, 8)
local BrandTitle = addLabel(Brand, "CATScript", 15, COLORS.text, Enum.Font.GothamBold)
BrandTitle.Position = UDim2.fromOffset(38, 2)
BrandTitle.Size = UDim2.new(1, -38, 0, 20)
local BrandSub = addLabel(Brand, "VISUAL DASHBOARD", 8, COLORS.muted, Enum.Font.GothamMedium)
BrandSub.Position = UDim2.fromOffset(38, 21)
BrandSub.Size = UDim2.new(1, -38, 0, 14)

local NavLabel = addLabel(Sidebar, "WORKSPACE", 9, COLORS.dim, Enum.Font.GothamBold)
NavLabel.Position = UDim2.fromOffset(16, 72)
NavLabel.Size = UDim2.new(1, -28, 0, 18)

local NavHolder = make("Frame", {
    BackgroundTransparency = 1,
    Position = UDim2.fromOffset(9, 96),
    Size = UDim2.new(1, -18, 0, 240),
}, Sidebar)

local VisualNav = make("TextButton", {
    Name = "VisualNavigation",
    Text = "",
    BackgroundColor3 = COLORS.panel2,
    Size = UDim2.new(1, 0, 0, 34),
    AutoButtonColor = false,
}, NavHolder)
corner(VisualNav, 7)
local indicator = make("Frame", {
    Position = UDim2.fromOffset(0, 7),
    Size = UDim2.fromOffset(3, 20),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
}, VisualNav)
corner(indicator, 3)
local navIcon = addLabel(VisualNav, "◈", 15, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
navIcon.Position = UDim2.fromOffset(8, 0)
navIcon.Size = UDim2.fromOffset(22, 34)
local navText = addLabel(VisualNav, "Visual", 10, COLORS.text, Enum.Font.GothamSemibold)
navText.Position = UDim2.fromOffset(37, 0)
navText.Size = UDim2.new(1, -44, 1, 0)

local PlayerNav = make("TextButton", {
    Name = "PlayerNavigation",
    Text = "",
    BackgroundColor3 = COLORS.sidebar,
    Position = UDim2.fromOffset(0, 40),
    Size = UDim2.new(1, 0, 0, 34),
    AutoButtonColor = false,
}, NavHolder)
corner(PlayerNav, 7)
local playerIndicator = make("Frame", {
    Position = UDim2.fromOffset(0, 7),
    Size = UDim2.fromOffset(3, 20),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    BackgroundTransparency = 1,
}, PlayerNav)
corner(playerIndicator, 3)
local playerIcon = addLabel(PlayerNav, "♙", 15, COLORS.muted, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
playerIcon.Position = UDim2.fromOffset(8, 0)
playerIcon.Size = UDim2.fromOffset(22, 34)
local playerText = addLabel(PlayerNav, "Player", 10, COLORS.muted, Enum.Font.GothamSemibold)
playerText.Position = UDim2.fromOffset(37, 0)
playerText.Size = UDim2.new(1, -44, 1, 0)

local ESPNav = make("TextButton", {
    Name = "ESPNavigation",
    Text = "",
    BackgroundColor3 = COLORS.sidebar,
    Position = UDim2.fromOffset(0, 80),
    Size = UDim2.new(1, 0, 0, 34),
    AutoButtonColor = false,
}, NavHolder)
corner(ESPNav, 7)
local espIndicator = make("Frame", {
    Position = UDim2.fromOffset(0, 7), Size = UDim2.fromOffset(3, 20),
    BackgroundColor3 = COLORS.accent, BorderSizePixel = 0, BackgroundTransparency = 1,
}, ESPNav)
corner(espIndicator, 3)
local espIcon = addLabel(ESPNav, "◎", 15, COLORS.muted, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
espIcon.Position = UDim2.fromOffset(8, 0)
espIcon.Size = UDim2.fromOffset(22, 34)
local espText = addLabel(ESPNav, "ESP", 10, COLORS.muted, Enum.Font.GothamSemibold)
espText.Position = UDim2.fromOffset(37, 0)
espText.Size = UDim2.new(1, -44, 1, 0)

local TeleportNav = make("TextButton", {
    Name = "TeleportNavigation",
    Text = "",
    BackgroundColor3 = COLORS.sidebar,
    Position = UDim2.fromOffset(0, 120),
    Size = UDim2.new(1, 0, 0, 34),
    AutoButtonColor = false,
}, NavHolder)
corner(TeleportNav, 7)
local teleportIndicator = make("Frame", {
    Position = UDim2.fromOffset(0, 7),
    Size = UDim2.fromOffset(3, 20),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    BackgroundTransparency = 1,
}, TeleportNav)
corner(teleportIndicator, 3)
local teleportIcon = addLabel(TeleportNav, "⌖", 15, COLORS.muted, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
teleportIcon.Position = UDim2.fromOffset(8, 0)
teleportIcon.Size = UDim2.fromOffset(22, 34)
local teleportText = addLabel(TeleportNav, "Teleport", 10, COLORS.muted, Enum.Font.GothamSemibold)
teleportText.Position = UDim2.fromOffset(37, 0)
teleportText.Size = UDim2.new(1, -44, 1, 0)

local SettingsNav = make("TextButton", {
    Name = "SettingsNavigation",
    Text = "",
    BackgroundColor3 = COLORS.sidebar,
    Position = UDim2.fromOffset(0, 160),
    Size = UDim2.new(1, 0, 0, 34),
    AutoButtonColor = false,
}, NavHolder)
corner(SettingsNav, 7)
local settingsIndicator = make("Frame", {
    Position = UDim2.fromOffset(0, 7),
    Size = UDim2.fromOffset(3, 20),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    BackgroundTransparency = 1,
}, SettingsNav)
corner(settingsIndicator, 3)
local settingsIcon = addLabel(SettingsNav, "⚙", 14, COLORS.muted, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
settingsIcon.Position = UDim2.fromOffset(8, 0)
settingsIcon.Size = UDim2.fromOffset(22, 34)
local settingsText = addLabel(SettingsNav, "Settings", 10, COLORS.muted, Enum.Font.GothamSemibold)
settingsText.Position = UDim2.fromOffset(37, 0)
settingsText.Size = UDim2.new(1, -44, 1, 0)

local SideBottom = make("Frame", {
    BackgroundTransparency = 1,
    Position = UDim2.new(0, 12, 1, -74),
    Size = UDim2.new(1, -24, 0, 62),
}, Sidebar)
make("Frame", {
    Size = UDim2.new(1, 0, 0, 1),
    BackgroundColor3 = COLORS.outline,
    BorderSizePixel = 0,
}, SideBottom)
local UserLabel = make("TextButton", {
    Name = "InfoButton",
    Text = "info",
    TextColor3 = COLORS.text,
    TextSize = 11,
    Font = Enum.Font.GothamSemibold,
    TextXAlignment = Enum.TextXAlignment.Left,
    TextYAlignment = Enum.TextYAlignment.Center,
    Position = UDim2.fromOffset(0, 9),
    Size = UDim2.new(1, 0, 0, 30),
    BackgroundTransparency = 1,
    AutoButtonColor = false,
    BorderSizePixel = 0,
}, SideBottom)
local AboutUsButton = make("TextButton", {
    Name = "AboutUsButton",
    Text = "About Us",
    TextColor3 = COLORS.muted,
    TextSize = 10,
    Font = Enum.Font.GothamSemibold,
    TextXAlignment = Enum.TextXAlignment.Left,
    TextYAlignment = Enum.TextYAlignment.Center,
    Position = UDim2.fromOffset(0, 41),
    Size = UDim2.new(1, 0, 0, 19),
    BackgroundTransparency = 1,
    AutoButtonColor = false,
    BorderSizePixel = 0,
}, SideBottom)

-- Profile-style information card opened from the "info" item at the bottom
-- of the sidebar. Displayed identity/account values come from LocalPlayer.
local SessionStartedAt = os.clock()
local function formatSessionDuration()
    local totalSeconds = math.max(0, math.floor(os.clock() - SessionStartedAt))
    local hours = math.floor(totalSeconds / 3600)
    local minutes = math.floor((totalSeconds % 3600) / 60)
    local seconds = totalSeconds % 60
    if hours > 0 then
        return string.format("%dh %02dm %02ds", hours, minutes, seconds)
    end
    return string.format("%dm %02ds", minutes, seconds)
end

local InfoOverlay = make("Frame", {
    Name = "InfoOverlay",
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.fromRGB(0, 0, 0),
    BackgroundTransparency = 0.38,
    BorderSizePixel = 0,
    Visible = false,
    ZIndex = 100,
}, Window)
local AboutUsOverlay = make("Frame", {
    Name = "AboutUsOverlay",
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.fromRGB(0, 0, 0),
    BackgroundTransparency = 0.38,
    BorderSizePixel = 0,
    Visible = false,
    ZIndex = 110,
}, Window)
local AboutUsCard = make("Frame", {
    Name = "AboutUsCard",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(500, 252),
    BackgroundColor3 = Color3.fromRGB(13, 14, 16),
    BorderSizePixel = 0,
    ZIndex = 111,
}, AboutUsOverlay)
corner(AboutUsCard, 8)
stroke(AboutUsCard, COLORS.accent, 0.18, 1)
local AboutUsHeader = make("Frame", {
    Name = "AboutUsHeader",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, 0, 0, 42),
    BackgroundColor3 = Color3.fromRGB(22, 18, 31),
    BorderSizePixel = 0,
    ZIndex = 112,
}, AboutUsCard)
corner(AboutUsHeader, 8)
make("Frame", {
    Position = UDim2.new(0, 0, 1, -8),
    Size = UDim2.new(1, 0, 0, 8),
    BackgroundColor3 = Color3.fromRGB(22, 18, 31),
    BorderSizePixel = 0,
    ZIndex = 112,
}, AboutUsHeader)
local AboutUsTitle = addLabel(AboutUsHeader, "ABOUT US", 11, COLORS.text, Enum.Font.GothamBold)
AboutUsTitle.Position = UDim2.fromOffset(16, 0)
AboutUsTitle.Size = UDim2.new(1, -64, 1, 0)
AboutUsTitle.ZIndex = 113
local AboutUsCloseButton = make("TextButton", {
    Name = "AboutUsCloseButton",
    Text = "×",
    TextColor3 = COLORS.muted,
    TextSize = 19,
    Font = Enum.Font.Gotham,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -8, 0.5, 0),
    Size = UDim2.fromOffset(26, 26),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    AutoButtonColor = false,
    ZIndex = 114,
}, AboutUsHeader)
local AboutUsDragHandle = make("TextButton", {
    Name = "AboutUsDragHandle",
    Text = "",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, -42, 1, 0),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    AutoButtonColor = false,
    TextTransparency = 1,
    ZIndex = 113,
}, AboutUsHeader)
local AboutUsBody = addLabel(AboutUsCard,
    "CATScript is a customizable Roblox interface that brings visual tools and useful quality-of-life settings together in one place.\n\nOur goal is to keep the experience simple, flexible, and easy to personalize. You can adjust the interface, explore visual options, and make the layout your own.\n\nThank you for using CATScript!  ♡",
    11, Color3.fromRGB(196, 198, 204), Enum.Font.Gotham)
AboutUsBody.Position = UDim2.fromOffset(20, 62)
AboutUsBody.Size = UDim2.new(1, -40, 0, 148)
AboutUsBody.TextWrapped = true
AboutUsBody.TextYAlignment = Enum.TextYAlignment.Top
AboutUsBody.ZIndex = 113
local AboutUsFooter = addLabel(AboutUsCard, "CATScript  •  CUSTOMIZE YOUR EXPERIENCE", 8, COLORS.accent, Enum.Font.GothamSemibold)
AboutUsFooter.Position = UDim2.new(0, 20, 1, -25)
AboutUsFooter.Size = UDim2.new(1, -40, 0, 14)
AboutUsFooter.ZIndex = 113
local InfoCard = make("Frame", {
    Name = "InfoCard",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(520, 190),
    BackgroundColor3 = Color3.fromRGB(13, 14, 16),
    BorderSizePixel = 0,
    ZIndex = 101,
}, InfoOverlay)
corner(InfoCard, 3)
stroke(InfoCard, Color3.fromRGB(67, 70, 74), 0.08, 1)

local InfoAvatarFrame = make("Frame", {
    Name = "InfoAvatarFrame",
    Position = UDim2.fromOffset(12, 12),
    Size = UDim2.fromOffset(108, 166),
    BackgroundColor3 = Color3.fromRGB(7, 8, 9),
    BorderSizePixel = 0,
    ZIndex = 102,
}, InfoCard)
stroke(InfoAvatarFrame, Color3.fromRGB(63, 66, 70), 0, 1)
local InfoAvatar = make("ImageLabel", {
    Name = "InfoAvatar",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.new(1, -8, 1, -8),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Image = "",
    ScaleType = Enum.ScaleType.Fit,
    ZIndex = 103,
}, InfoAvatarFrame)

local InfoWelcome = addLabel(InfoCard, "WELCOME BACK", 9, Color3.fromRGB(142, 145, 149), Enum.Font.GothamMedium)
InfoWelcome.Position = UDim2.fromOffset(136, 14)
InfoWelcome.Size = UDim2.new(1, -174, 0, 14)
InfoWelcome.ZIndex = 102
local InfoDisplayName = addLabel(InfoCard, LocalPlayer.DisplayName, 17, Color3.fromRGB(238, 239, 241), Enum.Font.GothamMedium)
InfoDisplayName.Position = UDim2.fromOffset(136, 29)
InfoDisplayName.Size = UDim2.new(1, -174, 0, 24)
InfoDisplayName.ZIndex = 102
local InfoUsername = addLabel(InfoCard, "@" .. LocalPlayer.Name, 9, Color3.fromRGB(144, 147, 151), Enum.Font.Code)
InfoUsername.Position = UDim2.fromOffset(136, 52)
InfoUsername.Size = UDim2.new(1, -174, 0, 16)
InfoUsername.ZIndex = 102
local InfoProfileDivider = make("Frame", {
    Position = UDim2.fromOffset(136, 74),
    Size = UDim2.new(1, -152, 0, 1),
    BackgroundColor3 = Color3.fromRGB(57, 60, 64),
    BorderSizePixel = 0,
    ZIndex = 102,
}, InfoCard)

local function makeInfoRow(labelText, yPosition, initialValue)
    local left = addLabel(InfoCard, labelText, 8, Color3.fromRGB(130, 133, 137), Enum.Font.GothamMedium)
    left.Position = UDim2.fromOffset(136, yPosition)
    left.Size = UDim2.new(0, 150, 0, 16)
    left.ZIndex = 102
    local right = addLabel(InfoCard, initialValue, 10, Color3.fromRGB(235, 236, 238), Enum.Font.GothamMedium, Enum.TextXAlignment.Right)
    right.Position = UDim2.fromOffset(286, yPosition)
    right.Size = UDim2.new(1, -302, 0, 16)
    right.ZIndex = 102
    return right
end

-- Identify the active client executor only when the environment exposes a
-- supported identification function. Never guess an executor name.
local function getExecutorName()
    local identifyFunctions = {}
    if type(identifyexecutor) == "function" then
        table.insert(identifyFunctions, identifyexecutor)
    end
    if type(getexecutorname) == "function" then
        table.insert(identifyFunctions, getexecutorname)
    end

    for _, identify in ipairs(identifyFunctions) do
        local ok, result = pcall(identify)
        if ok then
            if type(result) == "table" then
                result = result.Name or result.name or result[1]
            end
            if type(result) == "string" and result:gsub("%s+", "") ~= "" then
                return result
            end
        end
    end
    return "Unknown"
end

local InfoExecutorValue = makeInfoRow("EXECUTOR", 84, getExecutorName())
local InfoModeValue = makeInfoRow("MODE", 103, "Local")
local InfoAccountValue = makeInfoRow("ACCOUNT", 122, tostring(LocalPlayer.AccountAge) .. " days")
local InfoSessionValue = makeInfoRow("SESSION", 141, formatSessionDuration())

local InfoCloseButton = make("TextButton", {
    Name = "InfoCloseButton",
    Text = "×",
    TextColor3 = Color3.fromRGB(135, 138, 142),
    TextSize = 18,
    Font = Enum.Font.Gotham,
    AnchorPoint = Vector2.new(1, 0),
    Position = UDim2.new(1, -5, 0, 4),
    Size = UDim2.fromOffset(20, 20),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    AutoButtonColor = false,
    ZIndex = 104,
}, InfoCard)

-- Fetch the actual Roblox avatar thumbnail without blocking menu creation.
task.spawn(function()
    local thumbnailOk, thumbnail = pcall(function()
        local content, _ready = Players:GetUserThumbnailAsync(
            LocalPlayer.UserId,
            Enum.ThumbnailType.AvatarBust,
            Enum.ThumbnailSize.Size150x150
        )
        return content
    end)
    if thumbnailOk and type(thumbnail) == "string" then
        InfoAvatar.Image = thumbnail
    end
end)

-- Main area and top bar.
RuntimeState.Main = make("Frame", {
    Name = "Main",
    Position = UDim2.fromOffset(174, 0),
    Size = UDim2.new(1, -174, 1, 0),
    BackgroundTransparency = 1,
}, Window)

RuntimeState.Topbar = make("Frame", {
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, 0, 0, 54),
    BackgroundColor3 = COLORS.topbar,
    BackgroundTransparency = 0.24,
    BorderSizePixel = 0,
}, RuntimeState.Main)
make("Frame", {
    Position = UDim2.new(0, 0, 1, -1),
    Size = UDim2.new(1, 0, 0, 1),
    BackgroundColor3 = COLORS.outline,
    BorderSizePixel = 0,
}, RuntimeState.Topbar)

RuntimeState.Breadcrumb = addLabel(RuntimeState.Topbar, "CATScript   /   Visual", 12, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.Breadcrumb.Position = UDim2.fromOffset(20, 0)
RuntimeState.Breadcrumb.Size = UDim2.new(0.45, 0, 1, 0)

RuntimeState.Search = make("TextBox", {
    Name = "Search",
    PlaceholderText = "Search visual...",
    PlaceholderColor3 = COLORS.dim,
    Text = "",
    TextColor3 = COLORS.text,
    TextSize = 11,
    Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
    TextXAlignment = Enum.TextXAlignment.Left,
    Position = UDim2.new(1, -240, 0, 12),
    Size = UDim2.fromOffset(190, 30),
    BackgroundColor3 = COLORS.panel,
    BorderSizePixel = 0,
}, RuntimeState.Topbar)
corner(RuntimeState.Search, 7)
stroke(RuntimeState.Search, COLORS.outline, 0, 1)
make("UIPadding", {PaddingLeft = UDim.new(0, 11), PaddingRight = UDim.new(0, 8)}, RuntimeState.Search)

-- Full-window background is created as the bottom layer of Window above.

RuntimeState.MinimizeButton = make("TextButton", {
    Text = "—",
    TextColor3 = COLORS.muted,
    TextSize = 16,
    Font = Enum.Font.GothamBold,
    Position = UDim2.new(1, -42, 0, 10),
    Size = UDim2.fromOffset(28, 30),
    BackgroundColor3 = COLORS.panel,
    BorderSizePixel = 0,
    AutoButtonColor = false,
}, RuntimeState.Topbar)
corner(RuntimeState.MinimizeButton, 7)

RuntimeState.CloseButton = make("TextButton", {
    Text = "×",
    TextColor3 = COLORS.muted,
    TextSize = 20,
    Font = Enum.Font.Gotham,
    Position = UDim2.new(1, -10, 0, 10),
    AnchorPoint = Vector2.new(1, 0),
    Size = UDim2.fromOffset(28, 30),
    BackgroundColor3 = COLORS.panel,
    BorderSizePixel = 0,
    AutoButtonColor = false,
}, RuntimeState.Topbar)
corner(RuntimeState.CloseButton, 7)

-- Content: Visual card with Hover Highlight, Character Trail, and trail size/color editors.
RuntimeState.Content = make("ScrollingFrame", {
    Name = "Content",
    Position = UDim2.fromOffset(0, 64),
    Size = UDim2.new(1, -322, 1, -102),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    ScrollBarThickness = 3,
    ScrollBarImageColor3 = COLORS.dim,
    CanvasSize = UDim2.fromOffset(0, 0),
    AutomaticCanvasSize = Enum.AutomaticSize.Y,
}, RuntimeState.Main)
make("UIPadding", {
    PaddingTop = UDim.new(0, 16),
    PaddingBottom = UDim.new(0, 14),
    PaddingLeft = UDim.new(0, 18),
    PaddingRight = UDim.new(0, 18),
}, RuntimeState.Content)


-- Right-side live 3D avatar preview. This panel stays visible on both pages.
RuntimeState.PreviewPanel = make("Frame", {
    Name = "LiveAvatarPreviewPanel",
    Position = UDim2.new(1, -310, 0, 64),
    Size = UDim2.new(0, 298, 1, -102),
    BackgroundColor3 = COLORS.panel,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
    ZIndex = 1,
}, RuntimeState.Main)
corner(RuntimeState.PreviewPanel, 10)
stroke(RuntimeState.PreviewPanel, COLORS.outline, 0.12, 1)
RuntimeState.PreviewTitle = addLabel(RuntimeState.PreviewPanel, "LIVE AVATAR PREVIEW", 9, COLORS.text, Enum.Font.GothamBold)
RuntimeState.PreviewTitle.ZIndex = 3
RuntimeState.PreviewTitle.Position = UDim2.fromOffset(12, 10)
RuntimeState.PreviewTitle.Size = UDim2.new(1, -24, 0, 16)
RuntimeState.PreviewHint = addLabel(RuntimeState.PreviewPanel, "DRAG TO ROTATE  /  360°", 7, COLORS.muted, Enum.Font.GothamMedium)
RuntimeState.PreviewHint.ZIndex = 3
RuntimeState.PreviewHint.Position = UDim2.fromOffset(12, 27)
RuntimeState.PreviewHint.Size = UDim2.new(1, -24, 0, 13)
RuntimeState.PreviewViewport = make("ViewportFrame", {
    Name = "AvatarViewport",
    Active = true,
    Position = UDim2.fromOffset(10, 48),
    Size = UDim2.new(1, -20, 1, -91),
    BackgroundColor3 = Color3.fromRGB(8, 10, 15),
    BorderSizePixel = 0,
    Ambient = Color3.fromRGB(175, 175, 190),
    LightColor = Color3.fromRGB(255, 255, 255),
    LightDirection = Vector3.new(-0.35, -1, -0.45),
    ZIndex = 2,
}, RuntimeState.PreviewPanel)
corner(RuntimeState.PreviewViewport, 8)
stroke(RuntimeState.PreviewViewport, COLORS.outline, 0.25, 1)
RuntimeState.PreviewStatus = addLabel(RuntimeState.PreviewPanel, "LOADING YOUR AVATAR...", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.PreviewStatus.ZIndex = 3
RuntimeState.PreviewStatus.Position = UDim2.new(0, 12, 1, -31)
RuntimeState.PreviewStatus.Size = UDim2.new(1, -24, 0, 18)
RuntimeState.PreviewStatus.TextXAlignment = Enum.TextXAlignment.Center
RuntimeState.Preview = {angle = 0, dragging = false, dragX = 0, model = nil, effects = nil, highlight = nil, refreshQueued = false}

RuntimeState.VisualCard = make("Frame", {
    Name = "VisualCard",
    BackgroundColor3 = COLORS.panel,
    BackgroundTransparency = 0.24,
    BorderSizePixel = 0,
    Size = UDim2.new(1, 0, 0, 804),
}, RuntimeState.Content)
corner(RuntimeState.VisualCard, 10)
stroke(RuntimeState.VisualCard, COLORS.outline, 0, 1)
make("UIPadding", {
    PaddingTop = UDim.new(0, 14),
    PaddingBottom = UDim.new(0, 12),
    PaddingLeft = UDim.new(0, 16),
    PaddingRight = UDim.new(0, 16),
}, RuntimeState.VisualCard)

RuntimeState.VisualTitle = addLabel(RuntimeState.VisualCard, "Visual", 12, COLORS.text, Enum.Font.GothamBold)
RuntimeState.VisualTitle.Size = UDim2.new(1, 0, 0, 18)
RuntimeState.VisualSubtitle = addLabel(RuntimeState.VisualCard, "Local visual effects", 9, COLORS.muted, Enum.Font.Gotham)
RuntimeState.VisualSubtitle.Position = UDim2.fromOffset(0, 19)
RuntimeState.VisualSubtitle.Size = UDim2.new(1, 0, 0, 15)

RuntimeState.HoverRow = make("Frame", {
    Name = "HoverHighlightRow",
    Position = UDim2.fromOffset(0, 44),
    Size = UDim2.new(1, 0, 0, 78),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
    Visible = false, -- ESP/Hover Highlight removed from the Visual page.
}, RuntimeState.VisualCard)
corner(RuntimeState.HoverRow, 8)
stroke(RuntimeState.HoverRow, COLORS.outline, 0.35, 1)
RuntimeState.HoverTitle = addLabel(RuntimeState.HoverRow, "Hover Highlight", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.HoverTitle.Position = UDim2.fromOffset(12, 10)
RuntimeState.HoverTitle.Size = UDim2.new(1, -130, 0, 17)
RuntimeState.HoverDescription = addLabel(RuntimeState.HoverRow, "Four bright L-shaped corner brackets and a neon ring appear around the player under your cursor.", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.HoverDescription.Position = UDim2.fromOffset(12, 29)
RuntimeState.HoverDescription.Size = UDim2.new(1, -130, 0, 15)
RuntimeState.HoverStatus = addLabel(RuntimeState.HoverRow, "GLOWING CORNERS  /  NEON RING", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.HoverStatus.Position = UDim2.fromOffset(12, 49)
RuntimeState.HoverStatus.Size = UDim2.new(1, -130, 0, 13)

RuntimeState.HoverToggleButton = make("TextButton", {
    Name = "HoverHighlightToggle",
    Text = "",
    AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(42, 23),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
}, RuntimeState.HoverRow)
corner(RuntimeState.HoverToggleButton, 12)
RuntimeState.HoverToggleKnob = make("Frame", {
    Name = "Knob",
    Position = UDim2.fromOffset(21, 3),
    Size = UDim2.fromOffset(17, 17),
    BackgroundColor3 = COLORS.white,
    BorderSizePixel = 0,
}, RuntimeState.HoverToggleButton)
corner(RuntimeState.HoverToggleKnob, 9)

-- Compact HSV color editor, styled after the supplied color-picker reference.
local hoverColor = Color3.fromRGB(238, 247, 255)
if type(SavedConfig.hoverColor) == "table" then
    local saved = SavedConfig.hoverColor
    local r, g, b = tonumber(saved.r), tonumber(saved.g), tonumber(saved.b)
    if r and g and b then
        hoverColor = Color3.fromRGB(
            math.clamp(math.floor(r + 0.5), 0, 255),
            math.clamp(math.floor(g + 0.5), 0, 255),
            math.clamp(math.floor(b + 0.5), 0, 255)
        )
    end
end
local hoverHue, hoverSaturation, hoverValue = hoverColor:ToHSV()

-- Local character trail settings.
local trailEnabled = SavedConfig.trailEnabled == true
local trailColor = Color3.fromRGB(134, 111, 244)
if type(SavedConfig.trailColor) == "table" then
    local saved = SavedConfig.trailColor
    local r, g, b = tonumber(saved.r), tonumber(saved.g), tonumber(saved.b)
    if r and g and b then
        trailColor = Color3.fromRGB(
            math.clamp(math.floor(r + 0.5), 0, 255),
            math.clamp(math.floor(g + 0.5), 0, 255),
            math.clamp(math.floor(b + 0.5), 0, 255)
        )
    end
end
local trailHue, trailSaturation, trailValue = trailColor:ToHSV()

-- Jump Circle effect settings: it is local-only and triggers when the local
-- Humanoid enters the Jumping state. Color and style are persisted with Auto Save.
local jumpCircleEnabled = SavedConfig.jumpCircleEnabled == true
RuntimeState.JUMP_CIRCLE_STYLES = {
    "Ring", "Double Ring", "Triple Ring", "Hexagon", "Octagon",
    "Pentagon", "Triangle", "Diamond", "Starburst", "Double Star",
    "Pulse", "Ripple", "Compass", "Crosshair", "Flower",
    "Petal", "Gear", "Spiral", "Wave", "Rune",
}
local jumpCircleStyle = "Ring"
if type(SavedConfig.jumpCircleStyle) == "string" then
    for _, styleName in ipairs(RuntimeState.JUMP_CIRCLE_STYLES) do
        if SavedConfig.jumpCircleStyle == styleName then
            jumpCircleStyle = styleName
            break
        end
    end
end
local jumpCircleColor = Color3.fromRGB(115, 210, 255)
if type(SavedConfig.jumpCircleColor) == "table" then
    local saved = SavedConfig.jumpCircleColor
    local r, g, b = tonumber(saved.r), tonumber(saved.g), tonumber(saved.b)
    if r and g and b then
        jumpCircleColor = Color3.fromRGB(
            math.clamp(math.floor(r + 0.5), 0, 255),
            math.clamp(math.floor(g + 0.5), 0, 255),
            math.clamp(math.floor(b + 0.5), 0, 255)
        )
    end
end
local jumpCircleHue, jumpCircleSaturation, jumpCircleValue = jumpCircleColor:ToHSV()

-- Custom Aura: ten local 3D styles, one arbitrary user-selected color.
RuntimeState.CustomAura = (function()
    local savedAura = type(SavedConfig.customAura) == "table" and SavedConfig.customAura or {}
    local styles = {
        "Orbit Orbs", "Double Helix", "Energy Ring", "Flame Spiral", "Triple Halo",
        "Electric Arc", "Pulse Sphere", "Rune Orbit", "Comet Ring", "Cosmic Dust",
    }
    local style = type(savedAura.style) == "string" and savedAura.style or styles[1]
    if not table.find(styles, style) then style = styles[1] end
    local color = Color3.fromRGB(166, 120, 255)
    if type(savedAura.color) == "table" then
        local red, green, blue = tonumber(savedAura.color.r), tonumber(savedAura.color.g), tonumber(savedAura.color.b)
        if red and green and blue then
            color = Color3.fromRGB(
                math.clamp(math.floor(red + 0.5), 0, 255),
                math.clamp(math.floor(green + 0.5), 0, 255),
                math.clamp(math.floor(blue + 0.5), 0, 255)
            )
        end
    end
    local hue, saturation, value = color:ToHSV()
    return {
        styles = styles,
        style = style,
        size = math.clamp(tonumber(savedAura.size) or 1, 0.5, 3.5),
        offsetX = math.clamp(tonumber(savedAura.offsetX) or 0, -10, 10),
        offsetY = math.clamp(tonumber(savedAura.offsetY) or 0, -10, 10),
        offsetZ = math.clamp(tonumber(savedAura.offsetZ) or 0, -10, 10),
        enabled = savedAura.enabled == true,
        color = color,
        hue = hue,
        saturation = saturation,
        value = value,
        folder = nil,
        parts = {},
        phase = 0,
        connection = nil,
        pickerDragMode = nil,
        pickerInputChanged = nil,
        pickerInputEnded = nil,
        sizeDragging = false,
        offsetDraggingAxis = nil,
        offsetControls = {},
        sizeInputChanged = nil,
        sizeInputEnded = nil,
    }
end)()

-- Local 3D weather volume. Parts are placed in Workspace around the character
-- using world axes, never camera position or camera orientation.
RuntimeState.Weather = {settings = {}, colorNames = {}, particles = {}, random = Random.new(), folder = nil, editors = {}}
RuntimeState.Weather.defaults = {
    Snow = {amount = 30, speed = 1, size = 1, radius = 42, wind = 0},
    Rain = {amount = 42, speed = 1, size = 1, radius = 42, wind = -7},
    Stars = {amount = 12, speed = 1, size = 1, radius = 42, wind = 11},
}
RuntimeState.Weather.limits = {
    Snow = {amount = 70, size = 3.5},
    Rain = {amount = 90, size = 3.5},
    Stars = {amount = 24, size = 3.5},
}

for _, weatherName in ipairs({"Snow", "Rain", "Stars"}) do
    local savedWeather = type(SavedConfig.weather) == "table" and SavedConfig.weather[weatherName] or nil
    local savedColor = type(savedWeather) == "table" and savedWeather.color or nil
    local savedSettings = type(savedWeather) == "table" and type(savedWeather.settings) == "table" and savedWeather.settings or {}
    local defaults = RuntimeState.Weather.defaults[weatherName]
    local weatherColor = Color3.fromRGB(255, 255, 255)
    if type(savedColor) == "table" and tonumber(savedColor.r) and tonumber(savedColor.g) and tonumber(savedColor.b) then
        weatherColor = Color3.fromRGB(math.clamp(tonumber(savedColor.r),0,255), math.clamp(tonumber(savedColor.g),0,255), math.clamp(tonumber(savedColor.b),0,255))
    elseif weatherName == "Rain" then
        weatherColor = Color3.fromRGB(120, 190, 255)
    elseif weatherName == "Stars" then
        weatherColor = Color3.fromRGB(255, 225, 150)
    end
    RuntimeState.Weather.settings[weatherName] = {
        enabled = type(savedWeather) == "table" and savedWeather.enabled == true or false,
        color = weatherColor,
        amount = math.floor(math.clamp(tonumber(savedSettings.amount) or defaults.amount, 0, RuntimeState.Weather.limits[weatherName].amount)),
        speed = math.clamp(tonumber(savedSettings.speed) or defaults.speed, 0.1, 5),
        size = math.clamp(tonumber(savedSettings.size) or defaults.size, 0.25, RuntimeState.Weather.limits[weatherName].size),
        radius = math.clamp(tonumber(savedSettings.radius) or defaults.radius, 12, 1000),
        wind = math.clamp(tonumber(savedSettings.wind) or defaults.wind, -25, 25),
    }
end
RuntimeState.Weather.colorNames = {WeatherSnow = "Snow", WeatherRain = "Rain", WeatherStars = "Stars"}
RuntimeState.Weather.folder = make("Folder", {Name = "CATScript_Local3DWeather"}, workspace)
RuntimeState.Weather.overlay = RuntimeState.Weather.folder

-- Local Skybox Studio: 41 catalog/model skyboxes plus 9 six-face texture presets.
-- Imported models are inspected only for their Sky instance; no model scripts run.
RuntimeState.Skybox = {
    lighting = game:GetService("Lighting"),
    selectedName = type(SavedConfig.skyboxPreset) == "string" and SavedConfig.skyboxPreset or "",
    presets = {},
    buttons = {},
    activeSky = nil,
    originalHolder = nil,
}
RuntimeState.Skybox.presets = {
    {name="Broken Sky", assetId=47339},
    {name="Walls Of Autumn", assetId=47347},
    {name="Alien Red", assetId=47410},
    {name="Winterness", assetId=311580},
    {name="Starry Night (Classic)", assetId=47344},
    {name="The Great West", assetId=311557},
    {name="The Utter East", assetId=47346},
    {name="Oblivion", assetId=47343},
    {name="Shiverfrost", assetId=311594},
    {name="John Tron", assetId=47431},
    {name="Alien Harbor", assetId=108743092193471},
    {name="Ammonia", assetId=73737952108384},
    {name="Armageddon", assetId=115749850572333},
    {name="Backlit", assetId=110647969895287},
    {name="Clear Grey", assetId=88473811492122},
    {name="Cold Mountains", assetId=72443200622092},
    {name="Light Blue", assetId=75260239981592},
    {name="Mysterious", assetId=74651702873710},
    {name="Onimous", assetId=107257505430848},
    {name="Random", assetId=127581315250671},
    {name="Summer Day", assetId=133154146467731},
    {name="Yeller", assetId=74829495982134},
    {name="Autumn Day", assetId=107707022762708},
    {name="Summer Day 1", assetId=94831321187073},
    {name="Violet 02", assetId=127575046313619},
    {name="Antarctica", assetId=89001314806779},
    {name="Parfait", assetId=136868402397717},
    {name="Sunny Sky", assetId=132503658233433},
    {name="Summer Day 4", assetId=87195549252016},
    {name="Space (Racer)", assetId=80612076591201},
    {name="Sunwalk", assetId=138130199105905},
    {name="Underwater", assetId=131375863385877},
    {name="HD Skybox Sunset", assetId=148409227},
    {name="Goodsky", assetId=117575265},
    {name="DerpySky", assetId=117574169},
    {name="Skybox 2024", assetId=17054037007},
    {name="Skybox Pastel Blue", assetId=251053867},
    {name="Roblox SkyBox", assetId=9863391},
    {name="Pastel Sunset", assetId=159068040},
    {name="Solid Black", assetId=113702652},
    {name="Pastel Pink Clouds", assetId=152276191},
    {name="Tropic", faces={
        SkyboxBk="rbxassetid://169210090", SkyboxDn="rbxassetid://169210108",
        SkyboxFt="rbxassetid://169210121", SkyboxLf="rbxassetid://169210133",
        SkyboxRt="rbxassetid://169210143", SkyboxUp="rbxassetid://169210149",
    }},
    {name="Sunset", faces={
        SkyboxBk="rbxassetid://323494035", SkyboxDn="rbxassetid://323494368",
        SkyboxFt="rbxassetid://323494130", SkyboxLf="rbxassetid://323494252",
        SkyboxRt="rbxassetid://323494067", SkyboxUp="rbxassetid://323493360",
    }},
    {name="Red Night", faces={
        SkyboxBk="rbxassetid://401664839", SkyboxDn="rbxassetid://401664862",
        SkyboxFt="rbxassetid://401664960", SkyboxLf="rbxassetid://401664881",
        SkyboxRt="rbxassetid://401664901", SkyboxUp="rbxassetid://401664936",
    }},
    {name="Blue Horizon", faces={
        SkyboxBk="rbxassetid://135483466", SkyboxDn="rbxassetid://135483484",
        SkyboxFt="rbxassetid://135483461", SkyboxLf="rbxassetid://135483495",
        SkyboxRt="rbxassetid://135483499", SkyboxUp="rbxassetid://135483475",
    }},
    {name="Galaxy", faces={
        SkyboxBk="rbxassetid://159454299", SkyboxDn="rbxassetid://159454296",
        SkyboxFt="rbxassetid://159454293", SkyboxLf="rbxassetid://159454286",
        SkyboxRt="rbxassetid://159454300", SkyboxUp="rbxassetid://159454288",
    }},
    {name="Vaporwave", faces={
        SkyboxBk="rbxassetid://8631780182", SkyboxDn="rbxassetid://8631784904",
        SkyboxFt="rbxassetid://8631769834", SkyboxLf="rbxassetid://8631777199",
        SkyboxRt="rbxassetid://8631735555", SkyboxUp="rbxassetid://8631782345",
    }},
    {name="Pinkful Morning", faces={
        SkyboxBk="rbxassetid://12635309703", SkyboxDn="rbxassetid://12635311686",
        SkyboxFt="rbxassetid://12635312870", SkyboxLf="rbxassetid://12635313718",
        SkyboxRt="rbxassetid://12635315817", SkyboxUp="rbxassetid://12635316856",
    }},
    {name="Starry Deep Sky", faces={
        SkyboxBk="rbxassetid://2570432999", SkyboxDn="rbxassetid://2570433005",
        SkyboxFt="rbxassetid://2570432998", SkyboxLf="rbxassetid://2570433000",
        SkyboxRt="rbxassetid://2570432997", SkyboxUp="rbxassetid://2570432996",
    }},
    {name="Modern Roblox Baseplate", faces={
        SkyboxBk="rbxassetid://162001887", SkyboxDn="rbxassetid://161998893",
        SkyboxFt="rbxassetid://162001897", SkyboxLf="rbxassetid://162001904",
        SkyboxRt="rbxassetid://162001919", SkyboxUp="rbxassetid://162001926",
    }},
}

RuntimeState.Skybox.restore = function()
    local skyState = RuntimeState.Skybox
    if skyState.activeSky then
        pcall(function() skyState.activeSky:Destroy() end)
        skyState.activeSky = nil
    end
    if skyState.originalHolder and skyState.originalHolder.Parent then
        for _, savedSky in ipairs(skyState.originalHolder:GetChildren()) do
            if savedSky:IsA("Sky") then
                pcall(function() savedSky.Parent = skyState.lighting end)
            end
        end
        pcall(function() skyState.originalHolder:Destroy() end)
        skyState.originalHolder = nil
    end
    skyState.selectedName = ""
    if RuntimeState.SkyboxCurrentLabel then
        RuntimeState.SkyboxCurrentLabel.Text = "CURRENT: GAME SKY"
    end
end

RuntimeState.Skybox.prepareTarget = function()
    local skyState = RuntimeState.Skybox
    if not skyState.originalHolder then
        skyState.originalHolder = Instance.new("Folder")
        skyState.originalHolder.Name = "CATScript_OriginalSkyHolder"
        skyState.originalHolder.Parent = skyState.lighting
        for _, oldSky in ipairs(skyState.lighting:GetChildren()) do
            if oldSky:IsA("Sky") and oldSky ~= skyState.activeSky then
                oldSky.Parent = skyState.originalHolder
            end
        end
    end
    if not skyState.activeSky or not skyState.activeSky.Parent then
        skyState.activeSky = Instance.new("Sky")
        skyState.activeSky.Name = "CATScript_VisualSkybox"
        skyState.activeSky.Parent = skyState.lighting
    end
    return skyState.activeSky
end

RuntimeState.Skybox.copyAssetSky = function(assetId, target)
    local loadedRoots = nil
    local function findSky(roots)
        for _, rootObject in ipairs(roots or {}) do
            if rootObject:IsA("Sky") then return rootObject end
            local nestedSky = rootObject:FindFirstChildWhichIsA("Sky", true)
            if nestedSky then return nestedSky end
        end
        return nil
    end
    local assetService = game:GetService("AssetService")
    local asyncOk, asyncAsset = pcall(function()
        return assetService:LoadAssetAsync(tonumber(assetId))
    end)
    if asyncOk and asyncAsset then loadedRoots = {asyncAsset} end
    local sourceSky = findSky(loadedRoots)
    if not sourceSky then
        for _, rootObject in ipairs(loadedRoots or {}) do pcall(function() rootObject:Destroy() end) end
        loadedRoots = nil
        local loadedOk, loaded = pcall(function()
            return game:GetObjects("rbxassetid://" .. tostring(assetId))
        end)
        if loadedOk and type(loaded) == "table" then loadedRoots = loaded end
        sourceSky = findSky(loadedRoots)
    end
    if not sourceSky then
        for _, rootObject in ipairs(loadedRoots or {}) do pcall(function() rootObject:Destroy() end) end
        loadedRoots = nil
        local insertService = game:GetService("InsertService")
        local insertOk, inserted = pcall(function()
            return insertService:LoadAsset(tonumber(assetId))
        end)
        if insertOk and inserted then
            loadedRoots = {inserted}
            sourceSky = findSky(loadedRoots)
        end
    end
    if not sourceSky then
        for _, rootObject in ipairs(loadedRoots or {}) do pcall(function() rootObject:Destroy() end) end
        return false, "Asset недоступен или в нём не найден объект Sky"
    end

    for _, propertyName in ipairs({"SkyboxBk", "SkyboxDn", "SkyboxFt", "SkyboxLf", "SkyboxRt", "SkyboxUp", "SkyboxOrientation", "CelestialBodiesShown", "StarCount", "SunTextureId", "SunAngularSize", "MoonTextureId", "MoonAngularSize"}) do
        pcall(function() target[propertyName] = sourceSky[propertyName] end)
    end
    for _, rootObject in ipairs(loadedRoots or {}) do pcall(function() rootObject:Destroy() end) end
    return true
end

RuntimeState.Skybox.applyPreset = function(preset, shouldSave)
    if not RuntimeState.active or not preset then return false end
    local target = RuntimeState.Skybox.prepareTarget()
    local applied = false
    local reason = nil
    if type(preset.faces) == "table" then
        local assignedFaces = 0
        for _, faceName in ipairs({"SkyboxBk", "SkyboxDn", "SkyboxFt", "SkyboxLf", "SkyboxRt", "SkyboxUp"}) do
            if type(preset.faces[faceName]) == "string" then
                local ok, err = pcall(function() target[faceName] = preset.faces[faceName] end)
                if ok then assignedFaces = assignedFaces + 1 else reason = tostring(err) end
            end
        end
        applied = assignedFaces == 6
        if not applied and not reason then reason = "Не удалось назначить все шесть сторон Skybox" end
    elseif preset.assetId then
        applied, reason = RuntimeState.Skybox.copyAssetSky(preset.assetId, target)
    end
    if not RuntimeState.active then return false end
    if not applied then
        RuntimeState.Skybox.restore()
        return false, reason or "Не удалось применить skybox"
    end
    RuntimeState.Skybox.selectedName = preset.name
    if RuntimeState.SkyboxCurrentLabel then RuntimeState.SkyboxCurrentLabel.Text = "CURRENT: " .. preset.name end
    if RuntimeState.SkyboxStatus then RuntimeState.SkyboxStatus.Text = "APPLIED  /  " .. preset.name end
    if shouldSave ~= false and scheduleConfigSave then scheduleConfigSave() end
    return true
end

local jumpCircleParts = {}
local jumpCircleToken = 0
local jumpCircleConnection = nil
local applyJumpCircleColorValue

-- All editable Jump Circle dimensions are grouped to keep the main chunk small.
local savedJumpSizes = type(SavedConfig.jumpCircleSizes) == "table" and SavedConfig.jumpCircleSizes or {}
RuntimeState.jumpCircleSizes = {
    radius = math.clamp(tonumber(savedJumpSizes.radius) or 2.1, 0.5, 8),
    expansion = math.clamp(tonumber(savedJumpSizes.expansion) or 2.0, 0.1, 8),
    thickness = math.clamp(tonumber(savedJumpSizes.thickness) or 0.11, 0.025, 0.5),
    duration = math.clamp(tonumber(savedJumpSizes.duration) or 0.68, 0.15, 2.5),
    ringGap = math.clamp(tonumber(savedJumpSizes.ringGap) or 0.38, 0.1, 1.5),
    height = math.clamp(tonumber(savedJumpSizes.height) or 0.08, 0, 0.5),
}

-- Adjustable local trail dimensions. Values are clamped so malformed or old
-- config entries cannot produce invalid Roblox Trail properties.
local savedTrailSizes = type(SavedConfig.trailSizes) == "table" and SavedConfig.trailSizes or {}
local function loadTrailNumber(key, defaultValue, minimum, maximum)
    local value = tonumber(savedTrailSizes[key])
    if not value then return defaultValue end
    return math.clamp(value, minimum, maximum)
end
local trailStartWidth = loadTrailNumber("startWidth", 0.80, 0.10, 3.00)
local trailEndWidth = loadTrailNumber("endWidth", 0.04, 0.00, 2.00)
local trailLifetime = loadTrailNumber("lifetime", 0.55, 0.10, 4.00)
local trailMinLength = loadTrailNumber("minLength", 0.05, 0.01, 0.50)
local trailSpread = loadTrailNumber("spread", 1.10, 0.20, 2.50)
local applyTrailStyle
local trailSizeInputChangedConnection
local trailSizeInputEndedConnection

RuntimeState.HoverColorButton = make("TextButton", {
    Name = "HoverColorButton",
    Text = "",
    AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -62, 0.5, 0),
    Size = UDim2.fromOffset(27, 23),
    BackgroundColor3 = hoverColor,
    BorderSizePixel = 0,
}, RuntimeState.HoverRow)
corner(RuntimeState.HoverColorButton, 5)
stroke(RuntimeState.HoverColorButton, COLORS.outline, 0, 1)

RuntimeState.HoverColorPicker = make("Frame", {
    Name = "HoverHighlightColorEditor",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
    Visible = false,
    ZIndex = 210,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.HoverColorPicker, 8)
stroke(RuntimeState.HoverColorPicker, COLORS.outline, 0.25, 1)

RuntimeState.PickerTitle = addLabel(RuntimeState.HoverColorPicker, "HOVER HIGHLIGHT COLOR", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.PickerTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.PickerTitle.Size = UDim2.new(1, -24, 0, 17)

RuntimeState.SVArea = make("Frame", {
    Name = "SaturationValueArea",
    Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158),
    BackgroundColor3 = Color3.fromHSV(hoverHue, 1, 1),
    BorderSizePixel = 0,
    ClipsDescendants = true,
}, RuntimeState.HoverColorPicker)
corner(RuntimeState.SVArea, 3)

RuntimeState.SVWhite = make("Frame", {
    Name = "WhiteGradient",
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0,
}, RuntimeState.SVArea)
make("UIGradient", {
    Rotation = 0,
    Color = ColorSequence.new(Color3.new(1, 1, 1)),
    Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0),
        NumberSequenceKeypoint.new(1, 1),
    }),
}, RuntimeState.SVWhite)

RuntimeState.SVBlack = make("Frame", {
    Name = "BlackGradient",
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.new(0, 0, 0),
    BorderSizePixel = 0,
}, RuntimeState.SVArea)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new(Color3.new(0, 0, 0)),
    Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 1),
        NumberSequenceKeypoint.new(1, 0),
    }),
}, RuntimeState.SVBlack)

RuntimeState.SVInput = make("TextButton", {
    Name = "SVInput",
    Text = "",
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Size = UDim2.fromScale(1, 1),
    AutoButtonColor = false,
    ZIndex = 10,
}, RuntimeState.SVArea)
RuntimeState.SVMarker = make("Frame", {
    Name = "SelectionMarker",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(hoverSaturation, 1 - hoverValue),
    Size = UDim2.fromOffset(11, 11),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    ZIndex = 12,
}, RuntimeState.SVArea)
corner(RuntimeState.SVMarker, 8)
stroke(RuntimeState.SVMarker, Color3.new(1, 1, 1), 0, 2)

RuntimeState.HueInput = make("TextButton", {
    Name = "HueInput",
    Position = UDim2.fromOffset(224, 29),
    Size = UDim2.fromOffset(18, 158),
    Text = "",
    BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0,
    AutoButtonColor = false,
    ClipsDescendants = true,
}, RuntimeState.HoverColorPicker)
corner(RuntimeState.HueInput, 3)
RuntimeState.HueGradient = make("Frame", {
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0,
}, RuntimeState.HueInput)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
        ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
        ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
        ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
        ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
    }),
}, RuntimeState.HueGradient)
RuntimeState.HueMarker = make("Frame", {
    Name = "HueMarker",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.new(0.5, 0, hoverHue, 0),
    Size = UDim2.new(1, 4, 0, 3),
    BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0,
    ZIndex = 12,
}, RuntimeState.HueInput)
stroke(RuntimeState.HueMarker, Color3.fromRGB(30, 30, 30), 0, 1)

RuntimeState.HexBox = make("TextBox", {
    Name = "HexInput",
    Position = UDim2.fromOffset(12, 198),
    Size = UDim2.fromOffset(118, 26),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
    Text = "#EEF7FF",
    TextColor3 = COLORS.text,
    PlaceholderText = "#RRGGBB",
    PlaceholderColor3 = COLORS.dim,
    TextSize = 10,
    Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.HoverColorPicker)
corner(RuntimeState.HexBox, 3)
stroke(RuntimeState.HexBox, COLORS.outline, 0, 1)

RuntimeState.RGBBox = make("TextBox", {
    Name = "RGBInput",
    Position = UDim2.fromOffset(138, 198),
    Size = UDim2.fromOffset(126, 26),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
    Text = "238, 247, 255",
    TextColor3 = COLORS.text,
    PlaceholderText = "R, G, B",
    PlaceholderColor3 = COLORS.dim,
    TextSize = 10,
    Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.HoverColorPicker)
corner(RuntimeState.RGBBox, 3)
stroke(RuntimeState.RGBBox, COLORS.outline, 0, 1)

-- Trail behind the local character (not tracers to other players).
RuntimeState.TrailRow = make("Frame", {
    Name = "LocalTrailRow",
    Position = UDim2.fromOffset(0, 44),
    Size = UDim2.new(1, 0, 0, 78),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
}, RuntimeState.VisualCard)
corner(RuntimeState.TrailRow, 8)
stroke(RuntimeState.TrailRow, COLORS.outline, 0.35, 1)
RuntimeState.TrailTitle = addLabel(RuntimeState.TrailRow, "Character Trail", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.TrailTitle.Position = UDim2.fromOffset(12, 10)
RuntimeState.TrailTitle.Size = UDim2.new(1, -170, 0, 17)
RuntimeState.TrailDescription = addLabel(RuntimeState.TrailRow, "Leaves a colored fading trail behind your own character while moving.", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.TrailDescription.Position = UDim2.fromOffset(12, 29)
RuntimeState.TrailDescription.Size = UDim2.new(1, -170, 0, 15)
RuntimeState.TrailStatus = addLabel(RuntimeState.TrailRow, "LOCAL CHARACTER TRAIL", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.TrailStatus.Position = UDim2.fromOffset(12, 49)
RuntimeState.TrailStatus.Size = UDim2.new(1, -170, 0, 13)

RuntimeState.TrailToggleButton = make("TextButton", {
    Name = "CharacterTrailToggle", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(42, 23),
    BackgroundColor3 = trailEnabled and COLORS.accent or COLORS.dim,
    BorderSizePixel = 0,
}, RuntimeState.TrailRow)
corner(RuntimeState.TrailToggleButton, 12)
RuntimeState.TrailToggleKnob = make("Frame", {
    Name = "Knob", Position = trailEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    Size = UDim2.fromOffset(17, 17), BackgroundColor3 = COLORS.white, BorderSizePixel = 0,
}, RuntimeState.TrailToggleButton)
corner(RuntimeState.TrailToggleKnob, 9)

RuntimeState.trailSizeButton = make("TextButton", {
    Name = "TrailSizeButton", Text = "SIZE  ⚙",
    TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -98, 0.5, 0), Size = UDim2.fromOffset(56, 25),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
}, RuntimeState.TrailRow)
corner(RuntimeState.trailSizeButton, 5)
stroke(RuntimeState.trailSizeButton, COLORS.outline, 0, 1)

RuntimeState.TrailColorButton = make("TextButton", {
    Name = "TrailColorButton", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -62, 0.5, 0),
    Size = UDim2.fromOffset(27, 23), BackgroundColor3 = trailColor, BorderSizePixel = 0,
}, RuntimeState.TrailRow)
corner(RuntimeState.TrailColorButton, 5)
stroke(RuntimeState.TrailColorButton, COLORS.outline, 0, 1)

RuntimeState.TrailColorPicker = make("Frame", {
    Name = "CharacterTrailColorEditor", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 240,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.TrailColorPicker, 8)
stroke(RuntimeState.TrailColorPicker, COLORS.outline, 0.25, 1)
RuntimeState.TrailPickerTitle = addLabel(RuntimeState.TrailColorPicker, "CHARACTER TRAIL COLOR", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.TrailPickerTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.TrailPickerTitle.Size = UDim2.new(1, -24, 0, 17)

RuntimeState.TrailSVArea = make("Frame", {
    Name = "TrailSaturationValueArea", Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158), BackgroundColor3 = Color3.fromHSV(trailHue, 1, 1),
    BorderSizePixel = 0, ClipsDescendants = true,
}, RuntimeState.TrailColorPicker)
corner(RuntimeState.TrailSVArea, 3)
RuntimeState.TrailSVWhite = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0,
}, RuntimeState.TrailSVArea)
make("UIGradient", {
    Rotation = 0, Color = ColorSequence.new(Color3.new(1, 1, 1)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1)}),
}, RuntimeState.TrailSVWhite)
RuntimeState.TrailSVBlack = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0,
}, RuntimeState.TrailSVArea)
make("UIGradient", {
    Rotation = 90, Color = ColorSequence.new(Color3.new(0, 0, 0)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0)}),
}, RuntimeState.TrailSVBlack)
RuntimeState.TrailSVInput = make("TextButton", {
    Name = "TrailSVInput", Text = "", BackgroundTransparency = 1, BorderSizePixel = 0,
    Size = UDim2.fromScale(1, 1), AutoButtonColor = false, ZIndex = 10,
}, RuntimeState.TrailSVArea)
RuntimeState.TrailSVMarker = make("Frame", {
    Name = "TrailSelectionMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(trailSaturation, 1 - trailValue), Size = UDim2.fromOffset(11, 11),
    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.TrailSVArea)
corner(RuntimeState.TrailSVMarker, 8)
stroke(RuntimeState.TrailSVMarker, Color3.new(1, 1, 1), 0, 2)

RuntimeState.TrailHueInput = make("TextButton", {
    Name = "TrailHueInput", Position = UDim2.fromOffset(224, 29), Size = UDim2.fromOffset(18, 158),
    Text = "", BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, AutoButtonColor = false,
    ClipsDescendants = true,
}, RuntimeState.TrailColorPicker)
corner(RuntimeState.TrailHueInput, 3)
RuntimeState.TrailHueGradient = make("Frame", {Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0}, RuntimeState.TrailHueInput)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
        ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
        ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
        ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
        ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
    }),
}, RuntimeState.TrailHueGradient)
RuntimeState.TrailHueMarker = make("Frame", {
    Name = "TrailHueMarker", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, trailHue, 0),
    Size = UDim2.new(1, 4, 0, 3), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.TrailHueInput)
stroke(RuntimeState.TrailHueMarker, Color3.fromRGB(30, 30, 30), 0, 1)

RuntimeState.TrailHexBox = make("TextBox", {
    Name = "TrailHexInput", Position = UDim2.fromOffset(12, 198), Size = UDim2.fromOffset(118, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, Text = "#866FF4", TextColor3 = COLORS.text,
    PlaceholderText = "#RRGGBB", PlaceholderColor3 = COLORS.dim, TextSize = 10, Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.TrailColorPicker)
corner(RuntimeState.TrailHexBox, 3)
stroke(RuntimeState.TrailHexBox, COLORS.outline, 0, 1)
RuntimeState.TrailRGBBox = make("TextBox", {
    Name = "TrailRGBInput", Position = UDim2.fromOffset(138, 198), Size = UDim2.fromOffset(126, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, Text = "134, 111, 244", TextColor3 = COLORS.text,
    PlaceholderText = "R, G, B", PlaceholderColor3 = COLORS.dim, TextSize = 10, Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.TrailColorPicker)
corner(RuntimeState.TrailRGBBox, 3)
stroke(RuntimeState.TrailRGBBox, COLORS.outline, 0, 1)


-- Character Trail sizing controls: ribbon width at both ends, fade length,
-- segment spacing and the distance between its two attachment rails.
RuntimeState.TrailSizeCard = make("Frame", {
    Name = "CharacterTrailSizeEditor",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(320, 222),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
    Visible = false,
    ZIndex = 255,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.TrailSizeCard, 8)
stroke(RuntimeState.TrailSizeCard, COLORS.outline, 0.35, 1)
RuntimeState.TrailSizeTitle = addLabel(RuntimeState.TrailSizeCard, "CHARACTER TRAIL SIZE", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.TrailSizeTitle.Position = UDim2.fromOffset(12, 7)
RuntimeState.TrailSizeTitle.Size = UDim2.new(1, -24, 0, 15)
RuntimeState.TrailSizeHint = addLabel(RuntimeState.TrailSizeCard, "Adjust width, length and trail detail", 7, COLORS.dim, Enum.Font.Gotham)
RuntimeState.TrailSizeHint.Position = UDim2.fromOffset(12, 22)
RuntimeState.TrailSizeHint.Size = UDim2.new(1, -24, 0, 12)

RuntimeState.TrailSizeControls = {}
local function createTrailSizeSlider(key, title, y, minimum, maximum, step, value, suffix, decimals)
    local row = make("Frame", {
        Name = "TrailSize_" .. key,
        Position = UDim2.fromOffset(12, y),
        Size = UDim2.new(1, -24, 0, 34),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
    }, RuntimeState.TrailSizeCard)

    local titleLabel = make("TextLabel", {
        Name = "Title", BackgroundTransparency = 1, BorderSizePixel = 0,
        Position = UDim2.fromOffset(0, 0), Size = UDim2.new(1, -90, 0, 14),
        Font = Enum.Font.Gotham, Text = title, TextSize = 8,
        TextColor3 = COLORS.text, TextXAlignment = Enum.TextXAlignment.Left,
    }, row)
    local valueLabel = make("TextLabel", {
        Name = "Value", BackgroundTransparency = 1, BorderSizePixel = 0,
        AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0),
        Size = UDim2.fromOffset(88, 14), Font = Enum.Font.GothamSemibold,
        Text = "", TextSize = 8, TextColor3 = COLORS.accent,
        TextXAlignment = Enum.TextXAlignment.Right,
    }, row)
    local track = make("Frame", {
        Name = "Track", Position = UDim2.fromOffset(0, 21),
        Size = UDim2.new(1, 0, 0, 5), BackgroundColor3 = COLORS.background,
        BorderSizePixel = 0, ZIndex = 2,
    }, row)
    corner(track, 3)
    local alpha = math.clamp((value - minimum) / (maximum - minimum), 0, 1)
    local fill = make("Frame", {
        Name = "Fill", Size = UDim2.new(alpha, 0, 1, 0),
        BackgroundColor3 = COLORS.accent, BorderSizePixel = 0, ZIndex = 3,
    }, track)
    corner(fill, 3)
    local knob = make("Frame", {
        Name = "Knob", AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.new(alpha, 0, 0.5, 0), Size = UDim2.fromOffset(11, 11),
        BackgroundColor3 = COLORS.white, BorderSizePixel = 0, ZIndex = 5,
    }, track)
    corner(knob, 6)
    stroke(knob, COLORS.accent, 0, 1)
    local input = make("TextButton", {
        Name = "Input", Position = UDim2.fromOffset(-3, 14),
        Size = UDim2.new(1, 6, 0, 19), Text = "", AutoButtonColor = false,
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 8,
    }, row)

    local control = {
        key = key, minimum = minimum, maximum = maximum, step = step,
        value = value, suffix = suffix, decimals = decimals or 2,
        row = row, valueLabel = valueLabel, track = track,
        fill = fill, knob = knob, input = input,
    }
    RuntimeState.TrailSizeControls[#RuntimeState.TrailSizeControls + 1] = control
    return control
end

createTrailSizeSlider("startWidth", "Start width", 39, 0.10, 3.00, 0.05, trailStartWidth, "x", 2)
createTrailSizeSlider("endWidth", "End width", 75, 0.00, 2.00, 0.05, trailEndWidth, "x", 2)
createTrailSizeSlider("lifetime", "Trail length", 111, 0.10, 4.00, 0.05, trailLifetime, " s", 2)
createTrailSizeSlider("minLength", "Segment spacing", 147, 0.01, 0.50, 0.01, trailMinLength, " studs", 2)
createTrailSizeSlider("spread", "Trail spread", 183, 0.20, 2.50, 0.05, trailSpread, " studs", 2)

-- Jump Circle: selectable ground-ring styles shown when the local player jumps.
RuntimeState.JumpCircleRow = make("Frame", {
    Name = "JumpCircleRow",
    Position = UDim2.fromOffset(0, 134),
    Size = UDim2.new(1, 0, 0, 78),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
}, RuntimeState.VisualCard)
corner(RuntimeState.JumpCircleRow, 8)
stroke(RuntimeState.JumpCircleRow, COLORS.outline, 0.35, 1)
RuntimeState.JumpCircleTitle = addLabel(RuntimeState.JumpCircleRow, "Jump Circle", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.JumpCircleTitle.Position = UDim2.fromOffset(12, 9)
RuntimeState.JumpCircleTitle.Size = UDim2.new(1, -270, 0, 17)
RuntimeState.JumpCircleDescription = addLabel(RuntimeState.JumpCircleRow, "A colored ring effect appears around your character when you jump.", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.JumpCircleDescription.Position = UDim2.fromOffset(12, 28)
RuntimeState.JumpCircleDescription.Size = UDim2.new(1, -270, 0, 15)
RuntimeState.JumpCircleStatus = addLabel(RuntimeState.JumpCircleRow, "DISABLED  /  LOCAL JUMP EFFECT", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.JumpCircleStatus.Position = UDim2.fromOffset(12, 49)
RuntimeState.JumpCircleStatus.Size = UDim2.new(1, -270, 0, 13)

RuntimeState.JumpCircleToggleButton = make("TextButton", {
    Name = "JumpCircleToggle", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(42, 23),
    BackgroundColor3 = jumpCircleEnabled and COLORS.accent or COLORS.dim,
    BorderSizePixel = 0,
}, RuntimeState.JumpCircleRow)
corner(RuntimeState.JumpCircleToggleButton, 12)
RuntimeState.JumpCircleToggleKnob = make("Frame", {
    Name = "Knob", Position = jumpCircleEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    Size = UDim2.fromOffset(17, 17), BackgroundColor3 = COLORS.white, BorderSizePixel = 0,
}, RuntimeState.JumpCircleToggleButton)
corner(RuntimeState.JumpCircleToggleKnob, 9)

RuntimeState.JumpCircleColorButton = make("TextButton", {
    Name = "JumpCircleColorButton", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -62, 0.5, 0),
    Size = UDim2.fromOffset(27, 23), BackgroundColor3 = jumpCircleColor, BorderSizePixel = 0,
}, RuntimeState.JumpCircleRow)
corner(RuntimeState.JumpCircleColorButton, 5)
stroke(RuntimeState.JumpCircleColorButton, COLORS.outline, 0, 1)

RuntimeState.JumpCircleStyleButton = make("TextButton", {
    Name = "JumpCircleStyleButton", Text = "STYLE  ▾",
    TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -100, 0.5, 0), Size = UDim2.fromOffset(88, 25),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
}, RuntimeState.JumpCircleRow)
corner(RuntimeState.JumpCircleStyleButton, 5)
stroke(RuntimeState.JumpCircleStyleButton, COLORS.outline, 0, 1)

RuntimeState.JumpCircleSizeButton = make("TextButton", {
    Name = "JumpCircleSizeButton", Text = "SIZE  ⚙",
    TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -194, 0.5, 0), Size = UDim2.fromOffset(82, 25),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
}, RuntimeState.JumpCircleRow)
corner(RuntimeState.JumpCircleSizeButton, 5)
stroke(RuntimeState.JumpCircleSizeButton, COLORS.outline, 0, 1)

RuntimeState.JumpCircleColorPicker = make("Frame", {
    Name = "JumpCircleColorEditor", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 270,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.JumpCircleColorPicker, 8)
stroke(RuntimeState.JumpCircleColorPicker, COLORS.outline, 0.25, 1)
RuntimeState.JumpCirclePickerTitle = addLabel(RuntimeState.JumpCircleColorPicker, "JUMP CIRCLE COLOR", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.JumpCirclePickerTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.JumpCirclePickerTitle.Size = UDim2.new(1, -24, 0, 17)

RuntimeState.JumpCircleSVArea = make("Frame", {
    Name = "JumpCircleSaturationValueArea", Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158), BackgroundColor3 = Color3.fromHSV(jumpCircleHue, 1, 1),
    BorderSizePixel = 0, ClipsDescendants = true,
}, RuntimeState.JumpCircleColorPicker)
corner(RuntimeState.JumpCircleSVArea, 3)
RuntimeState.JumpCircleSVWhite = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0,
}, RuntimeState.JumpCircleSVArea)
make("UIGradient", {
    Rotation = 0, Color = ColorSequence.new(Color3.new(1, 1, 1)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1)}),
}, RuntimeState.JumpCircleSVWhite)
RuntimeState.JumpCircleSVBlack = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0,
}, RuntimeState.JumpCircleSVArea)
make("UIGradient", {
    Rotation = 90, Color = ColorSequence.new(Color3.new(0, 0, 0)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0)}),
}, RuntimeState.JumpCircleSVBlack)
RuntimeState.JumpCircleSVInput = make("TextButton", {
    Name = "JumpCircleSVInput", Text = "", BackgroundTransparency = 1,
    BorderSizePixel = 0, Size = UDim2.fromScale(1, 1), AutoButtonColor = false, ZIndex = 10,
}, RuntimeState.JumpCircleSVArea)
RuntimeState.JumpCircleSVMarker = make("Frame", {
    Name = "JumpCircleSelectionMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(jumpCircleSaturation, 1 - jumpCircleValue),
    Size = UDim2.fromOffset(11, 11), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.JumpCircleSVArea)
corner(RuntimeState.JumpCircleSVMarker, 8)
stroke(RuntimeState.JumpCircleSVMarker, Color3.new(1, 1, 1), 0, 2)

RuntimeState.JumpCircleHueInput = make("TextButton", {
    Name = "JumpCircleHueInput", Position = UDim2.fromOffset(224, 29),
    Size = UDim2.fromOffset(18, 158), Text = "", BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, AutoButtonColor = false, ClipsDescendants = true,
}, RuntimeState.JumpCircleColorPicker)
corner(RuntimeState.JumpCircleHueInput, 3)
RuntimeState.JumpCircleHueGradient = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0,
}, RuntimeState.JumpCircleHueInput)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
        ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
        ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
        ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
        ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
    }),
}, RuntimeState.JumpCircleHueGradient)
RuntimeState.JumpCircleHueMarker = make("Frame", {
    Name = "JumpCircleHueMarker", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, jumpCircleHue, 0),
    Size = UDim2.new(1, 4, 0, 3), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.JumpCircleHueInput)
stroke(RuntimeState.JumpCircleHueMarker, Color3.fromRGB(30, 30, 30), 0, 1)

RuntimeState.JumpCircleHexBox = make("TextBox", {
    Name = "JumpCircleHexInput", Position = UDim2.fromOffset(12, 198), Size = UDim2.fromOffset(118, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, Text = "#73D2FF", TextColor3 = COLORS.text,
    PlaceholderText = "#RRGGBB", PlaceholderColor3 = COLORS.dim, TextSize = 10, Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.JumpCircleColorPicker)
corner(RuntimeState.JumpCircleHexBox, 3)
stroke(RuntimeState.JumpCircleHexBox, COLORS.outline, 0, 1)
RuntimeState.JumpCircleRGBBox = make("TextBox", {
    Name = "JumpCircleRGBInput", Position = UDim2.fromOffset(138, 198), Size = UDim2.fromOffset(126, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, Text = "115, 210, 255", TextColor3 = COLORS.text,
    PlaceholderText = "R, G, B", PlaceholderColor3 = COLORS.dim, TextSize = 10, Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.JumpCircleColorPicker)
corner(RuntimeState.JumpCircleRGBBox, 3)
stroke(RuntimeState.JumpCircleRGBBox, COLORS.outline, 0, 1)

-- Custom Aura color editor: full HSV field plus HEX and RGB entry.
RuntimeState.CustomAuraColorPicker = make("Frame", {
    Name = "CustomAuraColorEditor", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 280,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.CustomAuraColorPicker, 8)
stroke(RuntimeState.CustomAuraColorPicker, COLORS.outline, 0.25, 1)
RuntimeState.CustomAuraPickerTitle = addLabel(RuntimeState.CustomAuraColorPicker, "CUSTOM AURA COLOR", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CustomAuraPickerTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.CustomAuraPickerTitle.Size = UDim2.new(1, -24, 0, 17)
RuntimeState.CustomAuraSVArea = make("Frame", {
    Name = "CustomAuraSaturationValueArea", Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158), BackgroundColor3 = Color3.fromHSV(RuntimeState.CustomAura.hue, 1, 1),
    BorderSizePixel = 0, ClipsDescendants = true,
}, RuntimeState.CustomAuraColorPicker)
corner(RuntimeState.CustomAuraSVArea, 3)
RuntimeState.CustomAuraSVWhite = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0,
}, RuntimeState.CustomAuraSVArea)
make("UIGradient", {
    Rotation = 0, Color = ColorSequence.new(Color3.new(1, 1, 1)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1)}),
}, RuntimeState.CustomAuraSVWhite)
RuntimeState.CustomAuraSVBlack = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0,
}, RuntimeState.CustomAuraSVArea)
make("UIGradient", {
    Rotation = 90, Color = ColorSequence.new(Color3.new(0, 0, 0)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0)}),
}, RuntimeState.CustomAuraSVBlack)
RuntimeState.CustomAuraSVInput = make("TextButton", {
    Name = "CustomAuraSVInput", Text = "", BackgroundTransparency = 1,
    BorderSizePixel = 0, Size = UDim2.fromScale(1, 1), AutoButtonColor = false, ZIndex = 10,
}, RuntimeState.CustomAuraSVArea)
RuntimeState.CustomAuraSVMarker = make("Frame", {
    Name = "CustomAuraSVMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(RuntimeState.CustomAura.saturation, 1 - RuntimeState.CustomAura.value),
    Size = UDim2.fromOffset(11, 11), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.CustomAuraSVArea)
corner(RuntimeState.CustomAuraSVMarker, 8)
stroke(RuntimeState.CustomAuraSVMarker, Color3.new(1, 1, 1), 0, 2)
RuntimeState.CustomAuraHueInput = make("TextButton", {
    Name = "CustomAuraHueInput", Position = UDim2.fromOffset(224, 29), Size = UDim2.fromOffset(18, 158),
    Text = "", BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0,
    AutoButtonColor = false, ClipsDescendants = true,
}, RuntimeState.CustomAuraColorPicker)
corner(RuntimeState.CustomAuraHueInput, 3)
RuntimeState.CustomAuraHueGradient = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0,
}, RuntimeState.CustomAuraHueInput)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
        ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
        ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
        ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
        ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
    }),
}, RuntimeState.CustomAuraHueGradient)
RuntimeState.CustomAuraHueMarker = make("Frame", {
    Name = "CustomAuraHueMarker", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, RuntimeState.CustomAura.hue, 0),
    Size = UDim2.new(1, 4, 0, 3), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.CustomAuraHueInput)
stroke(RuntimeState.CustomAuraHueMarker, Color3.fromRGB(30, 30, 30), 0, 1)
RuntimeState.CustomAuraHexBox = make("TextBox", {
    Name = "CustomAuraHexInput", Position = UDim2.fromOffset(12, 198), Size = UDim2.fromOffset(118, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
    Text = string.format("#%02X%02X%02X", math.floor(RuntimeState.CustomAura.color.R * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.G * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.B * 255 + 0.5)),
    TextColor3 = COLORS.text, PlaceholderText = "#RRGGBB", PlaceholderColor3 = COLORS.dim,
    TextSize = 10, Font = Enum.Font.Gotham, ClearTextOnFocus = false,
}, RuntimeState.CustomAuraColorPicker)
corner(RuntimeState.CustomAuraHexBox, 3)
stroke(RuntimeState.CustomAuraHexBox, COLORS.outline, 0, 1)
RuntimeState.CustomAuraRGBBox = make("TextBox", {
    Name = "CustomAuraRGBInput", Position = UDim2.fromOffset(138, 198), Size = UDim2.fromOffset(126, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
    Text = string.format("%d, %d, %d", math.floor(RuntimeState.CustomAura.color.R * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.G * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.B * 255 + 0.5)),
    TextColor3 = COLORS.text, PlaceholderText = "R, G, B", PlaceholderColor3 = COLORS.dim,
    TextSize = 10, Font = Enum.Font.Gotham, ClearTextOnFocus = false,
}, RuntimeState.CustomAuraColorPicker)
corner(RuntimeState.CustomAuraRGBBox, 3)
stroke(RuntimeState.CustomAuraRGBBox, COLORS.outline, 0, 1)

-- Ten selectable aura presets in a detached two-column style chooser.
RuntimeState.CustomAuraStyleEditor = make("Frame", {
    Name = "CustomAuraStyleEditor", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(300, 238), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 290,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.CustomAuraStyleEditor, 8)
stroke(RuntimeState.CustomAuraStyleEditor, COLORS.outline, 0.25, 1)
RuntimeState.CustomAuraStyleTitle = addLabel(RuntimeState.CustomAuraStyleEditor, "CUSTOM AURA  /  10 STYLES", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CustomAuraStyleTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.CustomAuraStyleTitle.Size = UDim2.new(1, -24, 0, 17)
RuntimeState.CustomAuraStyleButtons = {}
for styleIndex, styleName in ipairs(RuntimeState.CustomAura.styles) do
    local column = (styleIndex - 1) % 2
    local row = math.floor((styleIndex - 1) / 2)
    local styleButton = make("TextButton", {
        Name = "CustomAuraStyle_" .. tostring(styleIndex),
        Position = UDim2.fromOffset(12 + column * 140, 31 + row * 37),
        Size = UDim2.fromOffset(132, 30),
        BackgroundColor3 = styleName == RuntimeState.CustomAura.style and COLORS.accent or COLORS.background,
        BorderSizePixel = 0, AutoButtonColor = false,
        Text = styleName, TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    }, RuntimeState.CustomAuraStyleEditor)
    corner(styleButton, 6)
    stroke(styleButton, COLORS.outline, 0.25, 1)
    RuntimeState.CustomAuraStyleButtons[styleName] = styleButton
    styleButton.MouseButton1Click:Connect((function(selectedStyle)
        return function()
            RuntimeState.setCustomAuraStyle(selectedStyle)
            RuntimeState.CustomAuraStyleEditor.Visible = false
        end
    end)(styleName))
end

-- Twenty selectable Jump Circle presets in a detached chooser, matching the
-- two-column selection layout used by Custom Aura.
RuntimeState.JumpCircleStyleEditor = make("Frame", {
    Name = "JumpCircleStyleEditor", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(300, 408), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 292,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.JumpCircleStyleEditor, 8)
stroke(RuntimeState.JumpCircleStyleEditor, COLORS.outline, 0.25, 1)
RuntimeState.JumpCircleStyleTitle = addLabel(RuntimeState.JumpCircleStyleEditor, "JUMP CIRCLE  /  20 STYLES", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.JumpCircleStyleTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.JumpCircleStyleTitle.Size = UDim2.new(1, -24, 0, 17)
RuntimeState.JumpCircleStyleButtons = {}
for styleIndex, styleName in ipairs(RuntimeState.JUMP_CIRCLE_STYLES) do
    local column = (styleIndex - 1) % 2
    local row = math.floor((styleIndex - 1) / 2)
    local styleButton = make("TextButton", {
        Name = "JumpCircleStyle_" .. tostring(styleIndex),
        Position = UDim2.fromOffset(12 + column * 140, 31 + row * 36),
        Size = UDim2.fromOffset(132, 30),
        BackgroundColor3 = styleName == jumpCircleStyle and COLORS.accent or COLORS.background,
        BorderSizePixel = 0, AutoButtonColor = false,
        Text = styleName, TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    }, RuntimeState.JumpCircleStyleEditor)
    corner(styleButton, 6)
    stroke(styleButton, COLORS.outline, 0.25, 1)
    RuntimeState.JumpCircleStyleButtons[styleName] = styleButton
end

-- Compact detached size editor: scales the complete 3D aura uniformly.
RuntimeState.CustomAuraSizeEditor = make("Frame", {
    Name = "CustomAuraSizeEditor", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(320, 238), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 290,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.CustomAuraSizeEditor, 8)
stroke(RuntimeState.CustomAuraSizeEditor, COLORS.outline, 0.25, 1)
RuntimeState.CustomAuraSizeTitle = addLabel(RuntimeState.CustomAuraSizeEditor, "CUSTOM AURA SIZE + POSITION", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CustomAuraSizeTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.CustomAuraSizeTitle.Size = UDim2.new(1, -48, 0, 17)
RuntimeState.CustomAuraSizeHint = addLabel(RuntimeState.CustomAuraSizeEditor, "X / Y / Z offsets are relative to your character", 7, COLORS.dim, Enum.Font.Gotham)
RuntimeState.CustomAuraSizeHint.Position = UDim2.fromOffset(12, 26)
RuntimeState.CustomAuraSizeHint.Size = UDim2.new(1, -24, 0, 14)
RuntimeState.CustomAuraSizeValue = addLabel(RuntimeState.CustomAuraSizeEditor, "1.00x", 8, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
RuntimeState.CustomAuraSizeValue.Position = UDim2.new(1, -72, 0, 45)
RuntimeState.CustomAuraSizeValue.Size = UDim2.fromOffset(58, 14)
RuntimeState.CustomAuraSizeTrack = make("Frame", {
    Name = "CustomAuraSizeTrack", Position = UDim2.fromOffset(14, 72),
    Size = UDim2.new(1, -28, 0, 5), BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0, ZIndex = 2,
}, RuntimeState.CustomAuraSizeEditor)
corner(RuntimeState.CustomAuraSizeTrack, 3)
local auraSizeAlpha = (RuntimeState.CustomAura.size - 0.5) / 3
RuntimeState.CustomAuraSizeFill = make("Frame", {
    Name = "Fill", Size = UDim2.new(auraSizeAlpha, 0, 1, 0),
    BackgroundColor3 = COLORS.accent, BorderSizePixel = 0, ZIndex = 3,
}, RuntimeState.CustomAuraSizeTrack)
corner(RuntimeState.CustomAuraSizeFill, 3)
RuntimeState.CustomAuraSizeKnob = make("Frame", {
    Name = "Knob", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.new(auraSizeAlpha, 0, 0.5, 0), Size = UDim2.fromOffset(11, 11),
    BackgroundColor3 = COLORS.white, BorderSizePixel = 0, ZIndex = 5,
}, RuntimeState.CustomAuraSizeTrack)
corner(RuntimeState.CustomAuraSizeKnob, 6)
stroke(RuntimeState.CustomAuraSizeKnob, COLORS.accent, 0, 1)
RuntimeState.CustomAuraSizeInput = make("TextButton", {
    Name = "CustomAuraSizeInput", Position = UDim2.fromOffset(-4, -8),
    Size = UDim2.new(1, 8, 0, 21), Text = "", AutoButtonColor = false,
    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 8,
}, RuntimeState.CustomAuraSizeTrack)

-- Three independent local-space offsets let the user move the whole aura
-- around the character without changing its size or its animation path.
RuntimeState.createCustomAuraOffsetControl = function(axis, y, value)
    local editor = RuntimeState.CustomAuraSizeEditor
    local row = make("Frame", {
        Name = "AuraOffset" .. axis .. "Row", Position = UDim2.fromOffset(10, y),
        Size = UDim2.new(1, -20, 0, 34), BackgroundTransparency = 1, BorderSizePixel = 0,
    }, editor)
    local label = addLabel(row, "OFFSET " .. string.upper(axis), 7, COLORS.muted, Enum.Font.GothamBold)
    label.Position = UDim2.fromOffset(2, 1)
    label.Size = UDim2.fromOffset(66, 14)
    local valueLabel = addLabel(row, string.format("%+.1f", value), 8, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
    valueLabel.Position = UDim2.fromOffset(254, 0)
    valueLabel.Size = UDim2.fromOffset(50, 14)
    local track = make("Frame", {
        Name = "Offset" .. axis .. "Track", Position = UDim2.fromOffset(74, 19),
        Size = UDim2.fromOffset(174, 5), BackgroundColor3 = COLORS.background,
        BorderSizePixel = 0, ZIndex = 2,
    }, row)
    corner(track, 3)
    local alpha = math.clamp((value + 10) / 20, 0, 1)
    local fill = make("Frame", {
        Name = "Fill", Size = UDim2.new(alpha, 0, 1, 0),
        BackgroundColor3 = COLORS.accent, BorderSizePixel = 0, ZIndex = 3,
    }, track)
    corner(fill, 3)
    local knob = make("Frame", {
        Name = "Knob", AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.new(alpha, 0, 0.5, 0), Size = UDim2.fromOffset(10, 10),
        BackgroundColor3 = COLORS.white, BorderSizePixel = 0, ZIndex = 5,
    }, track)
    corner(knob, 6)
    stroke(knob, COLORS.accent, 0, 1)
    local input = make("TextButton", {
        Name = "Offset" .. axis .. "Input", Position = UDim2.fromOffset(-4, -8),
        Size = UDim2.new(1, 8, 0, 21), Text = "", AutoButtonColor = false,
        BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 8,
    }, track)
    RuntimeState.CustomAura.offsetControls[axis] = {
        row = row, value = valueLabel, track = track, fill = fill, knob = knob, input = input,
    }
    return input
end
RuntimeState.CustomAuraOffsetInputs = {
    X = RuntimeState.createCustomAuraOffsetControl("X", 104, RuntimeState.CustomAura.offsetX),
    Y = RuntimeState.createCustomAuraOffsetControl("Y", 145, RuntimeState.CustomAura.offsetY),
    Z = RuntimeState.createCustomAuraOffsetControl("Z", 186, RuntimeState.CustomAura.offsetZ),
}

-- The palettes are independent floating windows. Lift all their
-- descendants above the dashboard and leave their internal controls intact.
for _, editor in ipairs({RuntimeState.HoverColorPicker, RuntimeState.TrailColorPicker, RuntimeState.JumpCircleColorPicker, RuntimeState.JumpCircleStyleEditor, RuntimeState.TrailSizeCard, RuntimeState.CustomAuraColorPicker, RuntimeState.CustomAuraStyleEditor, RuntimeState.CustomAuraSizeEditor}) do
    local baseZ = editor.ZIndex
    for _, descendant in ipairs(editor:GetDescendants()) do
        if descendant:IsA("GuiObject") then
            descendant.ZIndex = baseZ + descendant.ZIndex
        end
    end
end

local scheduleConfigSave

-- Every detached window gets its own size, visual style and custom accent color.
-- Settings are keyed by the actual window name, so changing one editor never
-- recolors or resizes another one. They are persisted through the existing Auto Save.
RuntimeState.FloatingWindowAppearance = {
    settings = {},
    baseSizes = setmetatable({}, {__mode = "k"}),
    activeEditor = nil,
}
RuntimeState.FloatingWindowAppearance.saved = type(SavedConfig.floatingWindowAppearance) == "table"
    and SavedConfig.floatingWindowAppearance or {}

RuntimeState.getFloatingWindowAppearance = function(editor)
    local state = RuntimeState.FloatingWindowAppearance
    local key = editor.Name
    local cfg = state.settings[key]
    if cfg then return cfg end
    local saved = type(state.saved[key]) == "table" and state.saved[key] or {}
    local savedColor = type(saved.color) == "table" and saved.color or {}
    local red = math.clamp(tonumber(savedColor.r) or 134, 0, 255)
    local green = math.clamp(tonumber(savedColor.g) or 111, 0, 255)
    local blue = math.clamp(tonumber(savedColor.b) or 244, 0, 255)
    cfg = {
        scale = math.clamp(tonumber(saved.scale) or 1, 0.70, 1.35),
        style = (saved.style == "Minimal" or saved.style == "Neon") and saved.style or "Glass",
        color = Color3.fromRGB(red, green, blue),
    }
    state.settings[key] = cfg
    return cfg
end

RuntimeState.applyFloatingWindowAppearance = function(editor)
    if not editor or not editor.Parent then return end
    local state = RuntimeState.FloatingWindowAppearance
    local cfg = RuntimeState.getFloatingWindowAppearance(editor)
    local scaleObject = editor:FindFirstChild("CATScriptWindowScale")
    if not scaleObject then
        scaleObject = make("UIScale", {Name = "CATScriptWindowScale", Scale = 1}, editor)
    end
    scaleObject.Scale = cfg.scale

    local accent = cfg.color
    local style = cfg.style
    local tintStrength = style == "Neon" and 0.22 or (style == "Glass" and 0.15 or 0.08)
    local darkBase = Color3.fromRGB(9, 10, 16)
    editor.BackgroundColor3 = darkBase:Lerp(accent, tintStrength)
    editor.BackgroundTransparency = style == "Glass" and 0.16 or (style == "Neon" and 0.04 or 0.02)

    local rootStroke = editor:FindFirstChildOfClass("UIStroke")
    if rootStroke then
        rootStroke.Color = accent
        rootStroke.Transparency = style == "Neon" and 0.02 or (style == "Glass" and 0.28 or 0.48)
        rootStroke.Thickness = style == "Neon" and 1.8 or 1
    end
    local rootCorner = editor:FindFirstChildOfClass("UICorner")
    if rootCorner then
        rootCorner.CornerRadius = UDim.new(0, style == "Minimal" and 4 or (style == "Neon" and 11 or 8))
    end

    for _, descendant in ipairs(editor:GetDescendants()) do
        if descendant:IsA("GuiObject") then
            if descendant.Name == "Fill" then
                descendant.BackgroundColor3 = accent
            elseif descendant:IsA("TextButton") and descendant.BackgroundColor3 == COLORS.accent then
                descendant.BackgroundColor3 = accent
            end
            if descendant:IsA("TextLabel") or descendant:IsA("TextButton") or descendant:IsA("TextBox") then
                if descendant.TextColor3 == COLORS.accent then descendant.TextColor3 = accent end
            end
        end
    end
    local title = editor:FindFirstChild("FloatingEditorTitle")
    if title and title:IsA("TextLabel") then title.TextColor3 = accent end
    local appearanceButton = editor:FindFirstChild("FloatingAppearanceButton")
    if appearanceButton and appearanceButton:IsA("TextButton") then appearanceButton.TextColor3 = accent end
end

RuntimeState.createFloatingAppearancePanel = function()
    local panel = make("Frame", {
        Name = "FloatingWindowAppearancePanel", Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(270, 190), BackgroundColor3 = COLORS.panel2,
        BackgroundTransparency = 0.02, BorderSizePixel = 0, Visible = false, ZIndex = 1000,
        Active = true,
    }, RuntimeState.floatingEditorsLayer)
    corner(panel, 9)
    stroke(panel, COLORS.accent, 0.08, 1.4)
    local heading = addLabel(panel, "WINDOW CUSTOMIZER", 9, COLORS.text, Enum.Font.GothamBold)
    heading.Name = "FloatingAppearanceTitle"
    heading.Position = UDim2.fromOffset(12, 6)
    heading.Size = UDim2.new(1, -48, 0, 17)
    heading.Active = true
    local close = make("TextButton", {
        Name = "CloseAppearancePanel", Text = "×", TextColor3 = COLORS.muted,
        TextSize = 16, Font = Enum.Font.GothamBold, AutoButtonColor = false,
        Position = UDim2.new(1, -29, 0, 3), Size = UDim2.fromOffset(22, 20),
        BackgroundColor3 = COLORS.background, BorderSizePixel = 0, ZIndex = 1002,
    }, panel)
    corner(close, 5)
    close.MouseButton1Click:Connect(function() panel.Visible = false end)
    panel.Active = true
    local function beginAppearancePanelDrag(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
            and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local localX = input.Position.X - panel.AbsolutePosition.X
        local localY = input.Position.Y - panel.AbsolutePosition.Y
        if localY < 0 or localY > 27 or localX >= panel.AbsoluteSize.X - 34 then return end
        RuntimeState.floatingEditorDrag = {editor = panel, startPointer = input.Position, startPosition = panel.Position}
    end
    panel.InputBegan:Connect(beginAppearancePanelDrag)
    heading.InputBegan:Connect(beginAppearancePanelDrag)

    local targetLabel = addLabel(panel, "TARGET: -", 7, COLORS.muted, Enum.Font.Gotham)
    targetLabel.Position = UDim2.fromOffset(12, 25)
    targetLabel.Size = UDim2.new(1, -24, 0, 13)

    local sizeLabel = addLabel(panel, "SIZE", 7, COLORS.muted, Enum.Font.GothamBold)
    sizeLabel.Position = UDim2.fromOffset(12, 43)
    sizeLabel.Size = UDim2.fromOffset(50, 14)
    local sizeButtons = {}
    for index, item in ipairs({{"75%", 0.75}, {"90%", 0.90}, {"100%", 1.00}, {"110%", 1.10}, {"125%", 1.25}}) do
        local sizePreset = item
        local button = make("TextButton", {
            Name = "WindowSize" .. tostring(index), Text = sizePreset[1],
            Position = UDim2.fromOffset(12 + (index - 1) * 50, 59),
            Size = UDim2.fromOffset(45, 22), BackgroundColor3 = COLORS.background,
            TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
            AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 1001,
        }, panel)
        corner(button, 5)
        sizeButtons[index] = {button = button, value = sizePreset[2]}
        button.MouseButton1Click:Connect(function()
            local editor = RuntimeState.FloatingWindowAppearance.activeEditor
            if not editor or not editor.Parent then return end
            local cfg = RuntimeState.getFloatingWindowAppearance(editor)
            cfg.scale = sizePreset[2]
            RuntimeState.applyFloatingWindowAppearance(editor)
            if RuntimeState.scheduleFloatingAppearanceSave then RuntimeState.scheduleFloatingAppearanceSave() end
            for _, entry in ipairs(sizeButtons) do
                entry.button.BackgroundColor3 = math.abs(entry.value - cfg.scale) < 0.001 and cfg.color or COLORS.background
            end
        end)
    end

    local styleLabel = addLabel(panel, "STYLE", 7, COLORS.muted, Enum.Font.GothamBold)
    styleLabel.Position = UDim2.fromOffset(12, 88)
    styleLabel.Size = UDim2.fromOffset(50, 14)
    local styleButtons = {}
    for index, styleName in ipairs({"Glass", "Minimal", "Neon"}) do
        local selectedStyle = styleName
        local button = make("TextButton", {
            Name = "WindowStyle" .. selectedStyle, Text = string.upper(selectedStyle),
            Position = UDim2.fromOffset(12 + (index - 1) * 82, 104),
            Size = UDim2.fromOffset(76, 22), BackgroundColor3 = COLORS.background,
            TextColor3 = COLORS.text, TextSize = 7, Font = Enum.Font.GothamBold,
            AutoButtonColor = false, BorderSizePixel = 0, ZIndex = 1001,
        }, panel)
        corner(button, 5)
        styleButtons[selectedStyle] = button
        button.MouseButton1Click:Connect(function()
            local editor = RuntimeState.FloatingWindowAppearance.activeEditor
            if not editor or not editor.Parent then return end
            local cfg = RuntimeState.getFloatingWindowAppearance(editor)
            cfg.style = selectedStyle
            RuntimeState.applyFloatingWindowAppearance(editor)
            for name, styleButton in pairs(styleButtons) do
                styleButton.BackgroundColor3 = name == selectedStyle and cfg.color or COLORS.background
            end
            if RuntimeState.scheduleFloatingAppearanceSave then RuntimeState.scheduleFloatingAppearanceSave() end
        end)
    end

    local colorLabel = addLabel(panel, "CUSTOM COLOR  /  HEX", 7, COLORS.muted, Enum.Font.GothamBold)
    colorLabel.Position = UDim2.fromOffset(12, 132)
    colorLabel.Size = UDim2.fromOffset(130, 14)
    local hexBox = make("TextBox", {
        Name = "FloatingAppearanceHex", Position = UDim2.fromOffset(12, 149),
        Size = UDim2.fromOffset(112, 25), BackgroundColor3 = COLORS.background,
        BorderSizePixel = 0, Text = "#866FF4", TextColor3 = COLORS.text,
        PlaceholderText = "#RRGGBB", PlaceholderColor3 = COLORS.dim,
        TextSize = 9, Font = Enum.Font.Gotham, ClearTextOnFocus = false, ZIndex = 1001,
    }, panel)
    corner(hexBox, 5)
    stroke(hexBox, COLORS.outline, 0.1, 1)
    local preview = make("Frame", {
        Name = "FloatingAppearanceColorPreview", Position = UDim2.fromOffset(132, 149),
        Size = UDim2.fromOffset(25, 25), BackgroundColor3 = COLORS.accent,
        BorderSizePixel = 0, ZIndex = 1001,
    }, panel)
    corner(preview, 5)
    local applyColor = make("TextButton", {
        Name = "ApplyFloatingAppearanceColor", Text = "APPLY",
        Position = UDim2.fromOffset(164, 149), Size = UDim2.fromOffset(94, 25),
        BackgroundColor3 = COLORS.accent, TextColor3 = COLORS.white,
        TextSize = 8, Font = Enum.Font.GothamBold, AutoButtonColor = false,
        BorderSizePixel = 0, ZIndex = 1001,
    }, panel)
    corner(applyColor, 5)

    RuntimeState.FloatingWindowAppearance.panel = panel
    RuntimeState.FloatingWindowAppearance.targetLabel = targetLabel
    RuntimeState.FloatingWindowAppearance.sizeButtons = sizeButtons
    RuntimeState.FloatingWindowAppearance.styleButtons = styleButtons
    RuntimeState.FloatingWindowAppearance.hexBox = hexBox
    RuntimeState.FloatingWindowAppearance.preview = preview
    RuntimeState.FloatingWindowAppearance.applyColorButton = applyColor

    RuntimeState.applyFloatingAppearanceHex = function()
        local editor = RuntimeState.FloatingWindowAppearance.activeEditor
        if not editor or not editor.Parent then return end
        local hex = tostring(hexBox.Text or ""):gsub("#", ""):gsub("%s", "")
        if #hex ~= 6 or not hex:match("^%x%x%x%x%x%x$") then
            hexBox.TextColor3 = Color3.fromRGB(255, 115, 115)
            return
        end
        local red = tonumber(hex:sub(1, 2), 16)
        local green = tonumber(hex:sub(3, 4), 16)
        local blue = tonumber(hex:sub(5, 6), 16)
        local cfg = RuntimeState.getFloatingWindowAppearance(editor)
        cfg.color = Color3.fromRGB(red, green, blue)
        hexBox.Text = string.format("#%02X%02X%02X", red, green, blue)
        hexBox.TextColor3 = COLORS.text
        preview.BackgroundColor3 = cfg.color
        applyColor.BackgroundColor3 = cfg.color
        RuntimeState.applyFloatingWindowAppearance(editor)
        for _, entry in ipairs(sizeButtons) do
            entry.button.BackgroundColor3 = math.abs(entry.value - cfg.scale) < 0.001 and cfg.color or COLORS.background
        end
        for name, button in pairs(styleButtons) do
            button.BackgroundColor3 = name == cfg.style and cfg.color or COLORS.background
        end
        if RuntimeState.scheduleFloatingAppearanceSave then RuntimeState.scheduleFloatingAppearanceSave() end
    end
    applyColor.MouseButton1Click:Connect(RuntimeState.applyFloatingAppearanceHex)
    hexBox.FocusLost:Connect(function(enterPressed)
        if enterPressed then RuntimeState.applyFloatingAppearanceHex() end
    end)

    RuntimeState.openFloatingAppearance = function(editor)
        if not editor or not editor.Parent then return end
        local state = RuntimeState.FloatingWindowAppearance
        state.activeEditor = editor
        local cfg = RuntimeState.getFloatingWindowAppearance(editor)
        targetLabel.Text = "TARGET: " .. string.upper(editor.Name:gsub("(%u)", " %1"))
        hexBox.Text = string.format("#%02X%02X%02X", math.floor(cfg.color.R * 255 + 0.5), math.floor(cfg.color.G * 255 + 0.5), math.floor(cfg.color.B * 255 + 0.5))
        hexBox.TextColor3 = COLORS.text
        preview.BackgroundColor3 = cfg.color
        applyColor.BackgroundColor3 = cfg.color
        for _, entry in ipairs(sizeButtons) do
            entry.button.BackgroundColor3 = math.abs(entry.value - cfg.scale) < 0.001 and cfg.color or COLORS.background
        end
        for name, button in pairs(styleButtons) do
            button.BackgroundColor3 = name == cfg.style and cfg.color or COLORS.background
        end
        local camera = workspace.CurrentCamera
        local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
        local width, height = panel.AbsoluteSize.X, panel.AbsoluteSize.Y
        local editorPosition = editor.AbsolutePosition
        local x = editorPosition.X + editor.AbsoluteSize.X + 8
        if x + width > view.X then x = editorPosition.X - width - 8 end
        x = math.clamp(x, 0, math.max(0, view.X - width))
        local y = math.clamp(editorPosition.Y, 0, math.max(0, view.Y - height))
        panel.Position = UDim2.fromOffset(x, y)
        panel.Visible = true
        panel.ZIndex = 1000
        for _, descendant in ipairs(panel:GetDescendants()) do
            if descendant:IsA("GuiObject") then descendant.ZIndex = math.max(descendant.ZIndex, 1001) end
        end
    end
    return panel
end

RuntimeState.createFloatingAppearancePanel()
RuntimeState.scheduleFloatingAppearanceSave = function()
    if scheduleConfigSave then scheduleConfigSave() end
end

-- Add a full-width draggable header, independent customizer and close controls to every detached editor.
-- The helper is stored on RuntimeState to keep Luau local-register usage low.
RuntimeState.configureFloatingColorEditor = function(editor, titleLabel, slot)
    titleLabel.Active = true
    titleLabel.Name = "FloatingEditorTitle"
    titleLabel.Size = UDim2.new(1, -76, 0, 17)
    local appearanceButton = make("TextButton", {
        Name = "FloatingAppearanceButton", Text = "⚙", TextColor3 = COLORS.accent,
        TextSize = 11, Font = Enum.Font.GothamBold, AutoButtonColor = false,
        Position = UDim2.new(1, -55, 0, 3), Size = UDim2.fromOffset(22, 20),
        BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
        ZIndex = editor.ZIndex + 30,
    }, editor)
    corner(appearanceButton, 5)
    appearanceButton.MouseButton1Click:Connect(function()
        if RuntimeState.openFloatingAppearance then RuntimeState.openFloatingAppearance(editor) end
    end)
    local closeButton = make("TextButton", {
        Name = "CloseFloatingEditor", Text = "×", TextColor3 = COLORS.muted,
        TextSize = 16, Font = Enum.Font.GothamBold, AutoButtonColor = false,
        Position = UDim2.new(1, -29, 0, 3), Size = UDim2.fromOffset(22, 20),
        BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
        ZIndex = editor.ZIndex + 30,
    }, editor)
    corner(closeButton, 5)
    closeButton.MouseButton1Click:Connect(function()
        editor.Visible = false
    end)
    editor.Active = true
    local function beginFloatingWindowDrag(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
            and input.UserInputType ~= Enum.UserInputType.Touch then return end

        -- The complete top strip is a drag handle, not only the title text.
        -- Keep the appearance/close buttons on the right clickable.
        local localX = input.Position.X - editor.AbsolutePosition.X
        local localY = input.Position.Y - editor.AbsolutePosition.Y
        if localY < 0 or localY > 28 then return end
        if localX >= editor.AbsoluteSize.X - 62 then return end

        RuntimeState.floatingEditorDrag = {
            editor = editor,
            startPointer = input.Position,
            startPosition = editor.Position,
        }
        editor.ZIndex = math.max(editor.ZIndex, 300 + slot)
        for _, child in ipairs(editor:GetDescendants()) do
            if child:IsA("GuiObject") then
                child.ZIndex = math.max(child.ZIndex, editor.ZIndex + 1)
            end
        end
    end
    -- Root catches empty portions of the header; titleLabel keeps text dragging
    -- reliable on clients where GuiObject input does not bubble to its parent.
    editor.InputBegan:Connect(beginFloatingWindowDrag)
    titleLabel.InputBegan:Connect(beginFloatingWindowDrag)
    RuntimeState.applyFloatingWindowAppearance(editor)
end

RuntimeState.configureFloatingColorEditor(RuntimeState.HoverColorPicker, RuntimeState.PickerTitle, 1)
RuntimeState.configureFloatingColorEditor(RuntimeState.TrailColorPicker, RuntimeState.TrailPickerTitle, 2)
RuntimeState.configureFloatingColorEditor(RuntimeState.JumpCircleColorPicker, RuntimeState.JumpCirclePickerTitle, 3)
RuntimeState.configureFloatingColorEditor(RuntimeState.TrailSizeCard, RuntimeState.TrailSizeTitle, 4)
RuntimeState.configureFloatingColorEditor(RuntimeState.CustomAuraColorPicker, RuntimeState.CustomAuraPickerTitle, 6)
RuntimeState.configureFloatingColorEditor(RuntimeState.CustomAuraStyleEditor, RuntimeState.CustomAuraStyleTitle, 7)
RuntimeState.configureFloatingColorEditor(RuntimeState.CustomAuraSizeEditor, RuntimeState.CustomAuraSizeTitle, 13)
RuntimeState.configureFloatingColorEditor(RuntimeState.JumpCircleStyleEditor, RuntimeState.JumpCircleStyleTitle, 14)

RuntimeState.floatingEditorOpened = RuntimeState.floatingEditorOpened or {}
RuntimeState.openFloatingColorEditor = function(editor, slot)
    if editor.Visible then
        editor.Visible = false
        return
    end
    local camera = workspace.CurrentCamera
    local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
    local width, height = editor.AbsoluteSize.X, editor.AbsoluteSize.Y
    if width <= 0 then width = 276 end
    if height <= 0 then height = 232 end
    local maxX = math.max(0, view.X - width)
    local maxY = math.max(0, view.Y - 30)
    if not RuntimeState.floatingEditorOpened[slot] then
        local centerX = (view.X - width) * 0.5
        local centerY = (view.Y - height) * 0.5
        local x = math.clamp(centerX + (slot - 2) * 160, 0, maxX)
        local y = math.clamp(centerY + (slot - 2) * 26, 0, maxY)
        editor.Position = UDim2.fromOffset(x, y)
        RuntimeState.floatingEditorOpened[slot] = true
    end
    editor.Visible = true
end

RuntimeState.floatingEditorDragChanged = UserInputService.InputChanged:Connect(function(input)
    local drag = RuntimeState.floatingEditorDrag
    if not drag then return end
    if input.UserInputType ~= Enum.UserInputType.MouseMovement
        and input.UserInputType ~= Enum.UserInputType.Touch then return end
    local pointer = getColorEditorPointer(input)
    local delta = pointer - drag.startPointer
    local editor = drag.editor
    if not editor or not editor.Parent then
        RuntimeState.floatingEditorDrag = nil
        return
    end
    local camera = workspace.CurrentCamera
    local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
    local width, height = editor.AbsoluteSize.X, editor.AbsoluteSize.Y
    local x = drag.startPosition.X.Offset + delta.X
    local y = drag.startPosition.Y.Offset + delta.Y
    x = math.clamp(x, 0, math.max(0, view.X - math.min(width, view.X)))
    y = math.clamp(y, 0, math.max(0, view.Y - math.min(height, 30)))
    editor.Position = UDim2.fromOffset(x, y)
end)

RuntimeState.floatingEditorDragEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.floatingEditorDrag = nil
    end
end)

-- Jump Circle size editor is built in its own function scope to avoid Luau's
-- 200-local-register limit in this large UI script.
RuntimeState.createJumpCircleSizeEditor = function()
    local panel = make("Frame", {
        Name = "JumpCircleSizeEditor", Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(300, 250), BackgroundColor3 = COLORS.panel2,
        BorderSizePixel = 0, Visible = false, ZIndex = 250,
    }, RuntimeState.floatingEditorsLayer)
    corner(panel, 8)
    stroke(panel, COLORS.outline, 0.25, 1)
    local heading = addLabel(panel, "JUMP CIRCLE DIMENSIONS", 8, COLORS.muted, Enum.Font.GothamBold)
    heading.Position = UDim2.fromOffset(12, 5)
    heading.Size = UDim2.new(1, -48, 0, 18)
    RuntimeState.configureFloatingColorEditor(panel, heading, 5)

    local definitions = {
        {key = "radius", title = "Start radius", min = 0.5, max = 8, step = 0.1, unit = " studs"},
        {key = "expansion", title = "Expansion", min = 0.1, max = 8, step = 0.1, unit = " studs"},
        {key = "thickness", title = "Line thickness", min = 0.025, max = 0.5, step = 0.005, unit = " studs"},
        {key = "duration", title = "Fade duration", min = 0.15, max = 2.5, step = 0.05, unit = " s"},
        {key = "ringGap", title = "Ring spacing", min = 0.1, max = 1.5, step = 0.05, unit = " studs"},
        {key = "height", title = "Ground height", min = 0, max = 0.5, step = 0.01, unit = " studs"},
    }
    local controls = {}
    for index, definition in ipairs(definitions) do
        local y = 29 + (index - 1) * 34
        local title = addLabel(panel, definition.title, 8, COLORS.text, Enum.Font.Gotham)
        title.Position = UDim2.fromOffset(12, y)
        title.Size = UDim2.fromOffset(105, 18)
        local value = addLabel(panel, "", 8, COLORS.accent, Enum.Font.GothamSemibold, Enum.TextXAlignment.Right)
        value.Position = UDim2.new(1, -68, 0, y)
        value.Size = UDim2.fromOffset(56, 18)
        local track = make("TextButton", {
            Name = "JumpSize_" .. definition.key, Text = "", AutoButtonColor = false,
            Position = UDim2.fromOffset(120, y + 7), Size = UDim2.new(1, -205, 0, 5),
            BackgroundColor3 = COLORS.outline, BorderSizePixel = 0,
        }, panel)
        corner(track, 4)
        local fill = make("Frame", {Size = UDim2.new(0, 0, 1, 0), BackgroundColor3 = COLORS.accent, BorderSizePixel = 0}, track)
        corner(fill, 4)
        local knob = make("Frame", {AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0, 0, 0.5, 0), Size = UDim2.fromOffset(11, 11), BackgroundColor3 = COLORS.white, BorderSizePixel = 0}, track)
        corner(knob, 6)
        controls[definition.key] = {definition = definition, value = value, track = track, fill = fill, knob = knob}
    end

    local function refresh(key, newValue, shouldSave)
        local control = controls[key]
        if not control then return end
        local definition = control.definition
        local snapped = math.floor((newValue / definition.step) + 0.5) * definition.step
        snapped = math.clamp(snapped, definition.min, definition.max)
        RuntimeState.jumpCircleSizes[key] = snapped
        local fraction = (snapped - definition.min) / (definition.max - definition.min)
        control.fill.Size = UDim2.new(fraction, 0, 1, 0)
        control.knob.Position = UDim2.new(fraction, 0, 0.5, 0)
        control.value.Text = string.format("%.2f%s", snapped, definition.unit)
        if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
        if shouldSave and scheduleConfigSave then scheduleConfigSave() end
    end
    for key, control in pairs(controls) do
        refresh(key, RuntimeState.jumpCircleSizes[key], false)
        control.track.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                RuntimeState.jumpCircleSizeDragging = key
                local x = math.clamp((input.Position.X - control.track.AbsolutePosition.X) / math.max(control.track.AbsoluteSize.X, 1), 0, 1)
                refresh(key, control.definition.min + x * (control.definition.max - control.definition.min), true)
            end
        end)
    end
    RuntimeState.jumpCircleSizeInputChanged = UserInputService.InputChanged:Connect(function(input)
        local key = RuntimeState.jumpCircleSizeDragging
        if not key then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            local control = controls[key]
            if control then
                local position = getColorEditorPointer(input)
                local x = math.clamp((position.X - control.track.AbsolutePosition.X) / math.max(control.track.AbsoluteSize.X, 1), 0, 1)
                refresh(key, control.definition.min + x * (control.definition.max - control.definition.min), true)
            end
        end
    end)
    RuntimeState.jumpCircleSizeInputEnded = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            RuntimeState.jumpCircleSizeDragging = nil
        end
    end)
    return panel
end
RuntimeState.jumpCircleSizeEditor = RuntimeState.createJumpCircleSizeEditor()

-- Hat feature removed: clear legacy hat instances at startup and in preview clones.
RuntimeState.VisualCard.Size = UDim2.new(1, 0, 0, 812)

RuntimeState.CustomAuraRow = make("Frame", {
    Name = "CustomAuraRow", Position = UDim2.fromOffset(0, 220), Size = UDim2.new(1, 0, 0, 78),
    BackgroundColor3 = COLORS.panel2, BorderSizePixel = 0,
}, RuntimeState.VisualCard)
corner(RuntimeState.CustomAuraRow, 8)
stroke(RuntimeState.CustomAuraRow, COLORS.outline, 0.35, 1)
RuntimeState.CustomAuraTitle = addLabel(RuntimeState.CustomAuraRow, "Custom Aura", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.CustomAuraTitle.Position = UDim2.fromOffset(12, 9)
RuntimeState.CustomAuraTitle.Size = UDim2.new(1, -270, 0, 17)
RuntimeState.CustomAuraDescription = addLabel(RuntimeState.CustomAuraRow, "Animated 3D aura around your character with 10 styles.", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.CustomAuraDescription.Position = UDim2.fromOffset(12, 28)
RuntimeState.CustomAuraDescription.Size = UDim2.new(1, -270, 0, 15)
RuntimeState.CustomAuraStatus = addLabel(RuntimeState.CustomAuraRow, "DISABLED  /  " .. string.upper(RuntimeState.CustomAura.style), 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.CustomAuraStatus.Position = UDim2.fromOffset(12, 49)
RuntimeState.CustomAuraStatus.Size = UDim2.new(1, -270, 0, 13)
RuntimeState.CustomAuraToggleButton = make("TextButton", {
    Name = "CustomAuraToggle", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(42, 23),
    BackgroundColor3 = RuntimeState.CustomAura.enabled and COLORS.accent or COLORS.dim,
    BorderSizePixel = 0,
}, RuntimeState.CustomAuraRow)
corner(RuntimeState.CustomAuraToggleButton, 12)
RuntimeState.CustomAuraToggleKnob = make("Frame", {
    Name = "Knob", Position = RuntimeState.CustomAura.enabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    Size = UDim2.fromOffset(17, 17), BackgroundColor3 = COLORS.white, BorderSizePixel = 0,
}, RuntimeState.CustomAuraToggleButton)
corner(RuntimeState.CustomAuraToggleKnob, 9)
RuntimeState.CustomAuraColorButton = make("TextButton", {
    Name = "CustomAuraColorButton", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -62, 0.5, 0),
    Size = UDim2.fromOffset(27, 23), BackgroundColor3 = RuntimeState.CustomAura.color, BorderSizePixel = 0,
}, RuntimeState.CustomAuraRow)
corner(RuntimeState.CustomAuraColorButton, 5)
stroke(RuntimeState.CustomAuraColorButton, COLORS.outline, 0, 1)
RuntimeState.CustomAuraSizeButton = make("TextButton", {
    Name = "CustomAuraSizeButton", Text = "SIZE + XYZ  ⚙",
    TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -204, 0.5, 0), Size = UDim2.fromOffset(92, 25),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
}, RuntimeState.CustomAuraRow)
corner(RuntimeState.CustomAuraSizeButton, 5)
stroke(RuntimeState.CustomAuraSizeButton, COLORS.outline, 0.25, 1)
RuntimeState.CustomAuraStyleButton = make("TextButton", {
    Name = "CustomAuraStyleButton", Text = "STYLE  ▾",
    TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -100, 0.5, 0), Size = UDim2.fromOffset(84, 25),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
}, RuntimeState.CustomAuraRow)
corner(RuntimeState.CustomAuraStyleButton, 5)
stroke(RuntimeState.CustomAuraStyleButton, COLORS.outline, 0, 1)

RuntimeState.WeatherHeader = addLabel(RuntimeState.VisualCard, "WEATHER EFFECTS", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.WeatherHeader.Position = UDim2.fromOffset(0, 312)
RuntimeState.WeatherHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.WeatherRows = {}

-- Every weather type gets its own detached settings panel. Its sliders alter
-- the live 3D particle pool without touching the camera or the main dashboard.
RuntimeState.createWeatherSettingsEditor = function(weatherName, slot)
    local panel = make("Frame", {
        Name = "Weather" .. weatherName .. "SettingsEditor",
        Position = UDim2.fromOffset(0, 0), Size = UDim2.fromOffset(340, 220),
        BackgroundColor3 = COLORS.panel2, BorderSizePixel = 0, Visible = false, ZIndex = 260,
    }, RuntimeState.floatingEditorsLayer)
    corner(panel, 8)
    stroke(panel, COLORS.outline, 0.25, 1)
    local headingText = weatherName == "Stars" and "FALLING STARS SETTINGS" or (string.upper(weatherName) .. " SETTINGS")
    local heading = addLabel(panel, headingText, 8, COLORS.muted, Enum.Font.GothamBold)
    heading.Position = UDim2.fromOffset(12, 5)
    heading.Size = UDim2.new(1, -48, 0, 18)
    RuntimeState.configureFloatingColorEditor(panel, heading, slot)

    local amountMax = RuntimeState.Weather.limits[weatherName].amount
    local sizeTitle = weatherName == "Snow" and "Flake size" or (weatherName == "Rain" and "Drop length" or "Star size / trail")
    local windTitle = weatherName == "Snow" and "Wind drift" or (weatherName == "Rain" and "Wind / slant" or "Horizontal drift")
    local definitions = {
        {key = "amount", title = "Particle amount", min = 0, max = amountMax, step = 1, unit = " pcs"},
        {key = "speed", title = "Fall speed", min = 0.1, max = 5, step = 0.1, unit = "x"},
        {key = "size", title = sizeTitle, min = 0.25, max = 3.5, step = 0.05, unit = "x"},
        {key = "radius", title = "World radius", min = 12, max = 1000, step = 1, unit = " studs"},
        {key = "wind", title = windTitle, min = -25, max = 25, step = 1, unit = " studs/s"},
    }
    local controls = {}
    for index, definition in ipairs(definitions) do
        local y = 29 + (index - 1) * 35
        local title = addLabel(panel, definition.title, 8, COLORS.text, Enum.Font.Gotham)
        title.Position = UDim2.fromOffset(12, y)
        title.Size = UDim2.fromOffset(126, 18)
        local value = addLabel(panel, "", 8, COLORS.accent, Enum.Font.GothamSemibold, Enum.TextXAlignment.Right)
        value.Position = UDim2.new(1, -67, 0, y)
        value.Size = UDim2.fromOffset(57, 18)
        local track = make("TextButton", {
            Name = "Weather" .. weatherName .. "_" .. definition.key,
            Text = "", AutoButtonColor = false, Position = UDim2.fromOffset(141, y + 7),
            Size = UDim2.new(1, -222, 0, 5), BackgroundColor3 = COLORS.outline, BorderSizePixel = 0,
        }, panel)
        corner(track, 4)
        local fill = make("Frame", {Size = UDim2.new(0, 0, 1, 0), BackgroundColor3 = COLORS.accent, BorderSizePixel = 0}, track)
        corner(fill, 4)
        local knob = make("Frame", {AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0, 0, 0.5, 0), Size = UDim2.fromOffset(11, 11), BackgroundColor3 = COLORS.white, BorderSizePixel = 0}, track)
        corner(knob, 6)
        controls[definition.key] = {definition = definition, value = value, track = track, fill = fill, knob = knob}
    end

    local function refresh(key, newValue, shouldSave)
        local control = controls[key]
        if not control then return end
        local definition = control.definition
        local snapped = math.floor((newValue / definition.step) + 0.5) * definition.step
        snapped = math.clamp(tonumber(string.format("%.3f", snapped)) or definition.min, definition.min, definition.max)
        if key == "amount" or key == "radius" or key == "wind" then snapped = math.floor(snapped + 0.5) end
        RuntimeState.Weather.settings[weatherName][key] = snapped
        local fraction = (snapped - definition.min) / (definition.max - definition.min)
        control.fill.Size = UDim2.new(fraction, 0, 1, 0)
        control.knob.Position = UDim2.new(fraction, 0, 0.5, 0)
        local numberText = definition.step < 1 and string.format("%.2f", snapped) or string.format("%.0f", snapped)
        control.value.Text = numberText .. definition.unit
        if shouldSave and scheduleConfigSave then scheduleConfigSave() end
    end

    for key, control in pairs(controls) do
        local currentKey = key
        local currentControl = control
        refresh(currentKey, RuntimeState.Weather.settings[weatherName][currentKey], false)
        currentControl.track.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                RuntimeState.weatherSettingsDrag = {weatherName = weatherName, key = currentKey}
                local x = math.clamp((input.Position.X - currentControl.track.AbsolutePosition.X) / math.max(currentControl.track.AbsoluteSize.X, 1), 0, 1)
                refresh(currentKey, currentControl.definition.min + x * (currentControl.definition.max - currentControl.definition.min), true)
            end
        end)
    end
    RuntimeState.Weather.settingsInputChanged = RuntimeState.Weather.settingsInputChanged or {}
    RuntimeState.Weather.settingsInputChanged[weatherName] = UserInputService.InputChanged:Connect(function(input)
        local drag = RuntimeState.weatherSettingsDrag
        if not drag or drag.weatherName ~= weatherName then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local control = controls[drag.key]
        if control then
            local pointer = getColorEditorPointer(input)
            local x = math.clamp((pointer.X - control.track.AbsolutePosition.X) / math.max(control.track.AbsoluteSize.X, 1), 0, 1)
            refresh(drag.key, control.definition.min + x * (control.definition.max - control.definition.min), true)
        end
    end)
    RuntimeState.Weather.settingsInputEnded = RuntimeState.Weather.settingsInputEnded or {}
    RuntimeState.Weather.settingsInputEnded[weatherName] = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            local drag = RuntimeState.weatherSettingsDrag
            if drag and drag.weatherName == weatherName then RuntimeState.weatherSettingsDrag = nil end
        end
    end)
    for _, descendant in ipairs(panel:GetDescendants()) do
        if descendant:IsA("GuiObject") then
            descendant.ZIndex = math.max(descendant.ZIndex, panel.ZIndex + 1)
        end
    end
    return panel
end

for index, weatherName in ipairs({"Snow", "Rain", "Stars"}) do
    local weatherKey = weatherName
    local editorSlot = 8 + index
    local rowY = 334 + (index - 1) * 62
    local row = make("Frame", {Name="Weather"..weatherKey.."Row", Position=UDim2.fromOffset(0,rowY), Size=UDim2.new(1,0,0,54), BackgroundColor3=COLORS.panel2, BorderSizePixel=0}, RuntimeState.VisualCard)
    corner(row, 8); stroke(row, COLORS.outline, 0.35, 1)
    local titleText = weatherKey == "Stars" and "Falling Stars" or weatherKey
    local title = addLabel(row, titleText, 10, COLORS.text, Enum.Font.GothamSemibold)
    title.Position = UDim2.fromOffset(11,7); title.Size = UDim2.new(1,-190,0,16)
    local hint = addLabel(row, weatherKey == "Snow" and "3D snowflakes around your character" or weatherKey == "Rain" and "3D rain in the world" or "Glowing stars in the world", 8, COLORS.muted, Enum.Font.Gotham)
    hint.Position = UDim2.fromOffset(11,26); hint.Size = UDim2.new(1,-190,0,14)
    local settingsButton = make("TextButton", {Name="Weather"..weatherKey.."Settings", Text="⚙", TextSize=13, Font=Enum.Font.GothamBold, TextColor3=COLORS.text, AutoButtonColor=false, Position=UDim2.new(1,-119,0.5,-11), Size=UDim2.fromOffset(22,22), BackgroundColor3=COLORS.background, BorderSizePixel=0}, row)
    corner(settingsButton, 6); stroke(settingsButton, COLORS.outline, 0.2, 1)
    local colorButton = make("TextButton", {Name="Weather"..weatherKey.."Color", Text="", AutoButtonColor=false, Position=UDim2.new(1,-91,0.5,-12), Size=UDim2.fromOffset(24,24), BackgroundColor3=RuntimeState.Weather.settings[weatherKey].color, BorderSizePixel=0}, row)
    corner(colorButton, 6); stroke(colorButton, COLORS.outline, 0, 1)
    local toggle = make("TextButton", {Name="Weather"..weatherKey.."Toggle", Text=RuntimeState.Weather.settings[weatherKey].enabled and "ON" or "OFF", TextSize=8, Font=Enum.Font.GothamBold, TextColor3=COLORS.white, AutoButtonColor=false, Position=UDim2.new(1,-59,0.5,-11), Size=UDim2.fromOffset(48,22), BackgroundColor3=RuntimeState.Weather.settings[weatherKey].enabled and COLORS.accent or COLORS.dim, BorderSizePixel=0}, row)
    corner(toggle, 7)
    local editor = RuntimeState.createWeatherSettingsEditor(weatherKey, editorSlot)
    RuntimeState.Weather.editors[weatherKey] = editor
    RuntimeState.WeatherRows[weatherKey] = {row=row, color=colorButton, toggle=toggle, settings=settingsButton, editor=editor}
    toggle.MouseButton1Click:Connect(function()
        RuntimeState.Weather.settings[weatherKey].enabled = not RuntimeState.Weather.settings[weatherKey].enabled
        toggle.Text = RuntimeState.Weather.settings[weatherKey].enabled and "ON" or "OFF"
        toggle.BackgroundColor3 = RuntimeState.Weather.settings[weatherKey].enabled and COLORS.accent or COLORS.dim
        if scheduleConfigSave then scheduleConfigSave() end
    end)
    colorButton.MouseButton1Click:Connect(function()
        RuntimeState.openThemePicker("Weather"..weatherKey)
    end)
    settingsButton.MouseButton1Click:Connect(function()
        RuntimeState.openFloatingColorEditor(editor, editorSlot)
    end)
end

-- SKYBOX LIBRARY -------------------------------------------------------------
RuntimeState.SkyboxHeader = addLabel(RuntimeState.VisualCard, "SKYBOX LIBRARY  /  50 PRESETS", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.SkyboxHeader.Position = UDim2.fromOffset(0, 522)
RuntimeState.SkyboxHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.SkyboxRow = make("Frame", {
    Name = "SkyboxLibraryRow", Position = UDim2.fromOffset(0, 542),
    Size = UDim2.new(1, 0, 0, 68), BackgroundColor3 = COLORS.panel2, BorderSizePixel = 0,
}, RuntimeState.VisualCard)
corner(RuntimeState.SkyboxRow, 8)
stroke(RuntimeState.SkyboxRow, COLORS.outline, 0.35, 1)
RuntimeState.SkyboxCurrentLabel = addLabel(RuntimeState.SkyboxRow, "CURRENT: GAME SKY", 8, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.SkyboxCurrentLabel.Position = UDim2.fromOffset(11, 9)
RuntimeState.SkyboxCurrentLabel.Size = UDim2.new(1, -150, 0, 17)
RuntimeState.SkyboxStatus = addLabel(RuntimeState.SkyboxRow, "Choose a skybox from 50 presets", 7, COLORS.muted, Enum.Font.Gotham)
RuntimeState.SkyboxStatus.Position = UDim2.fromOffset(11, 31)
RuntimeState.SkyboxStatus.Size = UDim2.new(1, -150, 0, 15)
RuntimeState.SkyboxOpenButton = make("TextButton", {
    Name = "OpenSkyboxLibrary", Text = "CHOOSE  ▾", TextColor3 = COLORS.white,
    TextSize = 8, Font = Enum.Font.GothamBold, AutoButtonColor = false,
    Position = UDim2.new(1, -112, 0, 9), Size = UDim2.fromOffset(100, 24),
    BackgroundColor3 = COLORS.accent, BorderSizePixel = 0,
}, RuntimeState.SkyboxRow)
corner(RuntimeState.SkyboxOpenButton, 6)
RuntimeState.SkyboxResetButton = make("TextButton", {
    Name = "ResetSkybox", Text = "RESET", TextColor3 = COLORS.muted,
    TextSize = 7, Font = Enum.Font.GothamBold, AutoButtonColor = false,
    Position = UDim2.new(1, -112, 0, 39), Size = UDim2.fromOffset(100, 20),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
}, RuntimeState.SkyboxRow)
corner(RuntimeState.SkyboxResetButton, 6)

RuntimeState.SkyboxEditor = make("Frame", {
    Name = "SkyboxPresetLibrary", Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(382, 454), BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0, Visible = false, ZIndex = 350,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.SkyboxEditor, 9)
stroke(RuntimeState.SkyboxEditor, COLORS.outline, 0.2, 1)
RuntimeState.SkyboxEditorTitle = addLabel(RuntimeState.SkyboxEditor, "CATScript  /  SKYBOX PRESETS", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.SkyboxEditorTitle.Position = UDim2.fromOffset(12, 5)
RuntimeState.SkyboxEditorTitle.Size = UDim2.new(1, -50, 0, 17)
RuntimeState.SkyboxEditorTitle.ZIndex = 351
RuntimeState.configureFloatingColorEditor(RuntimeState.SkyboxEditor, RuntimeState.SkyboxEditorTitle, 12)
RuntimeState.SkyboxSearch = make("TextBox", {
    Name = "SkyboxSearch", Position = UDim2.fromOffset(12, 29), Size = UDim2.new(1, -24, 0, 29),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, ClearTextOnFocus = false,
    PlaceholderText = "Search 50 skies...", PlaceholderColor3 = COLORS.dim,
    Text = "", TextColor3 = COLORS.text, TextSize = 9, Font = Enum.Font.Gotham,
    TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 352,
}, RuntimeState.SkyboxEditor)
corner(RuntimeState.SkyboxSearch, 6)
make("UIPadding", {PaddingLeft=UDim.new(0,9), PaddingRight=UDim.new(0,9)}, RuntimeState.SkyboxSearch)
RuntimeState.SkyboxList = make("ScrollingFrame", {
    Name = "SkyboxPresetList", Position = UDim2.fromOffset(12, 68),
    Size = UDim2.new(1, -24, 1, -80), BackgroundTransparency = 1,
    BorderSizePixel = 0, ScrollBarThickness = 3, ScrollBarImageColor3 = COLORS.accent,
    CanvasSize = UDim2.fromOffset(0, 0), AutomaticCanvasSize = Enum.AutomaticSize.Y,
    ZIndex = 352,
}, RuntimeState.SkyboxEditor)
make("UIListLayout", {Padding=UDim.new(0,5), SortOrder=Enum.SortOrder.LayoutOrder}, RuntimeState.SkyboxList)
RuntimeState.Skybox.refreshFilter = function()
    local query = string.lower(RuntimeState.SkyboxSearch.Text or "")
    for _, entry in ipairs(RuntimeState.Skybox.buttons) do
        entry.button.Visible = query == "" or string.find(string.lower(entry.name), query, 1, true) ~= nil
    end
end
for skyIndex, preset in ipairs(RuntimeState.Skybox.presets) do
    local presetEntry = preset
    local presetButton = make("TextButton", {
        Name = "SkyPreset_" .. tostring(skyIndex), Size = UDim2.new(1, -4, 0, 29),
        BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
        Text = string.format("%02d   %s", skyIndex, preset.name),
        TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamSemibold,
        TextXAlignment = Enum.TextXAlignment.Left, AutoButtonColor = false,
        LayoutOrder = skyIndex, ZIndex = 353,
    }, RuntimeState.SkyboxList)
    corner(presetButton, 5)
    make("UIPadding", {PaddingLeft=UDim.new(0,10)}, presetButton)
    RuntimeState.Skybox.buttons[#RuntimeState.Skybox.buttons+1] = {name=preset.name, button=presetButton}
    presetButton.MouseButton1Click:Connect(function()
        RuntimeState.SkyboxStatus.Text = "LOADING  /  " .. presetEntry.name
        task.spawn(function()
            local ok, err = RuntimeState.Skybox.applyPreset(presetEntry, true)
            if not ok then
                RuntimeState.SkyboxStatus.Text = "FAILED  /  " .. tostring(err or "asset unavailable")
            end
        end)
    end)
end
RuntimeState.SkyboxSearch:GetPropertyChangedSignal("Text"):Connect(RuntimeState.Skybox.refreshFilter)
RuntimeState.SkyboxOpenButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.SkyboxEditor, 12)
end)
RuntimeState.SkyboxResetButton.MouseButton1Click:Connect(function()
    RuntimeState.Skybox.restore()
    RuntimeState.SkyboxCurrentLabel.Text = "CURRENT: GAME SKY"
    RuntimeState.SkyboxStatus.Text = "Restored the original game sky"
    if scheduleConfigSave then scheduleConfigSave() end
end)

-- Clear-water control. Terrain water uses its native transparency property;
-- custom water parts/decals are remembered and restored when the effect turns off.
RuntimeState.applyWater = function(enabled)
    local water = RuntimeState.Water
    water.enabled = enabled == true
    water.generation = (water.generation or 0) + 1
    local generation = water.generation
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if water.enabled then
        if terrain then
            if water.terrainTransparency == nil then water.terrainTransparency = terrain.WaterTransparency end
            if water.terrainReflectance == nil then water.terrainReflectance = terrain.WaterReflectance end
            pcall(function() terrain.WaterTransparency = water.transparency end)
            pcall(function() terrain.WaterReflectance = math.min(terrain.WaterReflectance, 0.08) end)
        end
        local function looksLikeWater(object)
            if object:IsA("BasePart") then
                local name = string.lower(object.Name)
                return object.Material == Enum.Material.Water or string.find(name, "water", 1, true)
                    or string.find(name, "ocean", 1, true) or string.find(name, "sea", 1, true)
            end
            return false
        end
        local function applyObject(object)
            if not water.enabled or water.generation ~= generation or not object or not object.Parent then return end
            local current = object
            local insideWater = false
            while current and current ~= workspace do
                if looksLikeWater(current) then insideWater = true break end
                current = current.Parent
            end
            if looksLikeWater(object) then
                if water.originals[object] == nil then water.originals[object] = {kind="part", transparency=object.Transparency} end
                pcall(function() object.Transparency = water.transparency end)
            elseif insideWater and (object:IsA("Texture") or object:IsA("Decal")) then
                if water.originals[object] == nil then water.originals[object] = {kind="image", transparency=object.Transparency} end
                pcall(function() object.Transparency = math.max(object.Transparency, water.transparency) end)
            end
        end
        if water.addedConnection then pcall(function() water.addedConnection:Disconnect() end) end
        water.addedConnection = workspace.DescendantAdded:Connect(function(object)
            task.defer(function() applyObject(object) end)
        end)
        task.spawn(function()
            local descendants = workspace:GetDescendants()
            for index, object in ipairs(descendants) do
                if not water.enabled or water.generation ~= generation then return end
                applyObject(object)
                if index % 300 == 0 then task.wait() end
            end
        end)
        return "CLEAR WATER ACTIVE"
    end
    if water.addedConnection then pcall(function() water.addedConnection:Disconnect() end); water.addedConnection = nil end
    if terrain then
        if water.terrainTransparency ~= nil then pcall(function() terrain.WaterTransparency = water.terrainTransparency end) end
        if water.terrainReflectance ~= nil then pcall(function() terrain.WaterReflectance = water.terrainReflectance end) end
    end
    for object, original in pairs(water.originals) do
        if object and object.Parent then pcall(function() object.Transparency = original.transparency end) end
    end
    table.clear(water.originals)
    return "ORIGINAL WATER RESTORED"
end
RuntimeState.setWaterTransparency = function(value)
    local water = RuntimeState.Water
    water.transparency = math.clamp(tonumber(value) or 0.88, 0, 1)
    if water.enabled then
        local terrain = workspace:FindFirstChildOfClass("Terrain")
        if terrain then pcall(function() terrain.WaterTransparency = water.transparency end) end
        for object,original in pairs(water.originals) do
            if object and object.Parent then
                pcall(function() object.Transparency = water.transparency end)
            end
        end
    end
    if scheduleConfigSave then scheduleConfigSave() end
end
RuntimeState.WaterHeader = addLabel(RuntimeState.VisualCard, "WATER  /  CLEAR MODE", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.WaterHeader.Position = UDim2.fromOffset(0, 620)
RuntimeState.WaterHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.WaterRow = make("Frame", {
    Name="ClearWaterRow", Position=UDim2.fromOffset(0,640), Size=UDim2.new(1,0,0,72),
    BackgroundColor3=COLORS.panel2, BorderSizePixel=0,
}, RuntimeState.VisualCard)
corner(RuntimeState.WaterRow, 8); stroke(RuntimeState.WaterRow, COLORS.outline, 0.35, 1)
RuntimeState.WaterTitle = addLabel(RuntimeState.WaterRow, "Transparent Water", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.WaterTitle.Position = UDim2.fromOffset(11, 7); RuntimeState.WaterTitle.Size = UDim2.new(1,-150,0,16)
RuntimeState.WaterValue = addLabel(RuntimeState.WaterRow, tostring(math.floor(RuntimeState.Water.transparency*100+0.5)).."%", 8, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
RuntimeState.WaterValue.Position = UDim2.new(1,-120,0,7); RuntimeState.WaterValue.Size = UDim2.fromOffset(50,16)
RuntimeState.WaterToggle = make("TextButton", {Name="ClearWaterToggle",Text=RuntimeState.Water.enabled and "ON" or "OFF",TextSize=8,Font=Enum.Font.GothamBold,TextColor3=COLORS.white,AutoButtonColor=false,Position=UDim2.new(1,-61,0,5),Size=UDim2.fromOffset(48,22),BackgroundColor3=RuntimeState.Water.enabled and COLORS.accent or COLORS.dim,BorderSizePixel=0}, RuntimeState.WaterRow)
corner(RuntimeState.WaterToggle, 7)
RuntimeState.WaterTrack = make("TextButton", {Name="WaterTransparencyTrack",Text="",AutoButtonColor=false,Position=UDim2.fromOffset(12,42),Size=UDim2.new(1,-24,0,6),BackgroundColor3=COLORS.background,BorderSizePixel=0}, RuntimeState.WaterRow)
corner(RuntimeState.WaterTrack, 3)
RuntimeState.WaterFill = make("Frame", {Size=UDim2.new(RuntimeState.Water.transparency,0,1,0),BackgroundColor3=COLORS.accent,BorderSizePixel=0}, RuntimeState.WaterTrack)
corner(RuntimeState.WaterFill, 3)
local waterDragging = false
local function updateWaterFromX(pointerX)
    local fraction = math.clamp((pointerX-RuntimeState.WaterTrack.AbsolutePosition.X)/math.max(RuntimeState.WaterTrack.AbsoluteSize.X,1),0,1)
    RuntimeState.setWaterTransparency(math.floor(fraction*100+0.5)/100)
    RuntimeState.WaterFill.Size = UDim2.new(RuntimeState.Water.transparency,0,1,0)
    RuntimeState.WaterValue.Text = tostring(math.floor(RuntimeState.Water.transparency*100+0.5)).."%"
end
RuntimeState.WaterTrack.InputBegan:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
        waterDragging=true; updateWaterFromX(input.Position.X)
    end
end)
RuntimeState.waterInputChanged = UserInputService.InputChanged:Connect(function(input)
    if not waterDragging then return end
    if input.UserInputType==Enum.UserInputType.MouseMovement then updateWaterFromX(UserInputService:GetMouseLocation().X)
    elseif input.UserInputType==Enum.UserInputType.Touch then updateWaterFromX(input.Position.X) end
end)
RuntimeState.waterInputEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then waterDragging=false end
end)
RuntimeState.WaterToggle.MouseButton1Click:Connect(function()
    local enabled = not RuntimeState.Water.enabled
    local message = RuntimeState.applyWater(enabled)
    RuntimeState.WaterToggle.Text = enabled and "ON" or "OFF"
    RuntimeState.WaterToggle.BackgroundColor3 = enabled and COLORS.accent or COLORS.dim
    if RuntimeState.WaterStatus then RuntimeState.WaterStatus.Text = message end
    if scheduleConfigSave then scheduleConfigSave() end
end)
if RuntimeState.Water.enabled then RuntimeState.applyWater(true) end

-- No Fog: clear legacy Lighting fog and Atmosphere haze locally. Keep the
-- exact original values so disabling the option or stopping CATScript restores
-- the game's lighting rather than leaving modified global properties behind.
RuntimeState.applyNoFog = function(enabled)
    local noFog = RuntimeState.NoFog
    local lighting = RuntimeState.Skybox and RuntimeState.Skybox.lighting or game:GetService("Lighting")
    enabled = enabled == true
    if enabled == noFog.enabled then
        return enabled and "NO FOG ACTIVE" or "ORIGINAL FOG RESTORED"
    end

    noFog.enabled = enabled
    noFog.generation = (noFog.generation or 0) + 1
    local generation = noFog.generation

    local function clearAtmosphere(object)
        if not noFog.enabled or noFog.generation ~= generation or not object or not object:IsA("Atmosphere") then return end
        if noFog.atmosphereOriginals[object] == nil then
            local saved = {}
            for _, propertyName in ipairs({"Density", "Haze", "Glare"}) do
                local ok, value = pcall(function() return object[propertyName] end)
                if ok then saved[propertyName] = value end
            end
            noFog.atmosphereOriginals[object] = saved
        end
        pcall(function() object.Density = 0 end)
        pcall(function() object.Haze = 0 end)
        pcall(function() object.Glare = 0 end)
    end

    if enabled then
        if noFog.fogStart == nil then noFog.fogStart = lighting.FogStart end
        if noFog.fogEnd == nil then noFog.fogEnd = lighting.FogEnd end
        pcall(function() lighting.FogStart = 0 end)
        pcall(function() lighting.FogEnd = 1000000 end)

        for _, object in ipairs(lighting:GetDescendants()) do
            if object:IsA("Atmosphere") then clearAtmosphere(object) end
        end
        if noFog.addedConnection then pcall(function() noFog.addedConnection:Disconnect() end) end
        noFog.addedConnection = lighting.DescendantAdded:Connect(function(object)
            if object:IsA("Atmosphere") then
                task.defer(function()
                    if noFog.enabled and noFog.generation == generation and object.Parent then
                        clearAtmosphere(object)
                    end
                end)
            end
        end)
        return "NO FOG ACTIVE"
    end

    if noFog.addedConnection then
        pcall(function() noFog.addedConnection:Disconnect() end)
        noFog.addedConnection = nil
    end
    if noFog.fogStart ~= nil then pcall(function() lighting.FogStart = noFog.fogStart end) end
    if noFog.fogEnd ~= nil then pcall(function() lighting.FogEnd = noFog.fogEnd end) end
    noFog.fogStart, noFog.fogEnd = nil, nil
    for object, saved in pairs(noFog.atmosphereOriginals) do
        if object and object.Parent then
            for propertyName, value in pairs(saved) do
                pcall(function() object[propertyName] = value end)
            end
        end
    end
    table.clear(noFog.atmosphereOriginals)
    return "ORIGINAL FOG RESTORED"
end

RuntimeState.NoFogHeader = addLabel(RuntimeState.VisualCard, "ATMOSPHERE  /  VISIBILITY", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.NoFogHeader.Position = UDim2.fromOffset(0, 722)
RuntimeState.NoFogHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.NoFogRow = make("Frame", {
    Name = "NoFogRow", Position = UDim2.fromOffset(0, 742), Size = UDim2.new(1, 0, 0, 56),
    BackgroundColor3 = COLORS.panel2, BorderSizePixel = 0,
}, RuntimeState.VisualCard)
corner(RuntimeState.NoFogRow, 8)
stroke(RuntimeState.NoFogRow, COLORS.outline, 0.35, 1)
RuntimeState.NoFogTitle = addLabel(RuntimeState.NoFogRow, "No Fog", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.NoFogTitle.Position = UDim2.fromOffset(11, 8)
RuntimeState.NoFogTitle.Size = UDim2.new(1, -150, 0, 16)
RuntimeState.NoFogDescription = addLabel(RuntimeState.NoFogRow, "Removes Lighting fog and Atmosphere haze locally.", 7, COLORS.muted, Enum.Font.Gotham)
RuntimeState.NoFogDescription.Position = UDim2.fromOffset(11, 29)
RuntimeState.NoFogDescription.Size = UDim2.new(1, -150, 0, 13)
RuntimeState.NoFogToggle = make("TextButton", {
    Name = "NoFogToggle", Text = RuntimeState.NoFog.savedEnabled and "ON" or "OFF", TextSize = 8,
    Font = Enum.Font.GothamBold, TextColor3 = COLORS.white, AutoButtonColor = false,
    Position = UDim2.new(1, -61, 0.5, -11), Size = UDim2.fromOffset(48, 22),
    BackgroundColor3 = RuntimeState.NoFog.savedEnabled and COLORS.accent or COLORS.dim, BorderSizePixel = 0,
}, RuntimeState.NoFogRow)
corner(RuntimeState.NoFogToggle, 7)
RuntimeState.NoFogStatus = addLabel(RuntimeState.NoFogRow, RuntimeState.NoFog.savedEnabled and "NO FOG ACTIVE" or "DISABLED", 7, RuntimeState.NoFog.savedEnabled and COLORS.accent or COLORS.dim, Enum.Font.GothamBold)
RuntimeState.NoFogStatus.Position = UDim2.new(1, -124, 0, 35)
RuntimeState.NoFogStatus.Size = UDim2.fromOffset(112, 12)
RuntimeState.NoFogStatus.TextXAlignment = Enum.TextXAlignment.Right
RuntimeState.NoFogToggle.MouseButton1Click:Connect(function()
    local enabled = not RuntimeState.NoFog.enabled
    local statusText = RuntimeState.applyNoFog(enabled)
    RuntimeState.NoFogToggle.Text = enabled and "ON" or "OFF"
    RuntimeState.NoFogToggle.BackgroundColor3 = enabled and COLORS.accent or COLORS.dim
    RuntimeState.NoFogStatus.Text = statusText
    RuntimeState.NoFogStatus.TextColor3 = enabled and COLORS.accent or COLORS.dim
    if scheduleConfigSave then scheduleConfigSave() end
end)
if RuntimeState.NoFog.savedEnabled then
    RuntimeState.applyNoFog(true)
end

-- Optional shot visualization feature omitted.

if RuntimeState.Skybox.selectedName ~= "" then
    task.defer(function()
        for _, preset in ipairs(RuntimeState.Skybox.presets) do
            if preset.name == RuntimeState.Skybox.selectedName then
                RuntimeState.Skybox.applyPreset(preset, false)
                break
            end
        end
    end)
end

-- Build a reusable 3D pool for each type; count sliders control how many parts
-- are active while unused parts remain invisible to avoid per-change allocation.
for _, weatherName in ipairs({"Snow", "Rain", "Stars"}) do
    local count = RuntimeState.Weather.limits[weatherName].amount
    local settings = RuntimeState.Weather.settings[weatherName]
    for i = 1, count do
        local part
        if weatherName == "Snow" then
            part = make("Part", {Name="CATScript_Snow3D", Shape=Enum.PartType.Ball, Size=Vector3.new(0.2,0.2,0.2), Material=Enum.Material.SmoothPlastic, Color=settings.color, Transparency=1, Anchored=true, CanCollide=false, CanTouch=false, CanQuery=false, CastShadow=false}, RuntimeState.Weather.folder)
        elseif weatherName == "Rain" then
            part = make("Part", {Name="CATScript_Rain3D", Shape=Enum.PartType.Block, Size=Vector3.new(0.055,2.7,0.055), Material=Enum.Material.Neon, Color=settings.color, Transparency=1, Anchored=true, CanCollide=false, CanTouch=false, CanQuery=false, CastShadow=false}, RuntimeState.Weather.folder)
        else
            part = make("Part", {Name="CATScript_FallingStar3D", Shape=Enum.PartType.Block, Size=Vector3.new(0.09,0.09,1.25), Material=Enum.Material.Neon, Color=settings.color, Transparency=1, Anchored=true, CanCollide=false, CanTouch=false, CanQuery=false, CastShadow=false}, RuntimeState.Weather.folder)
        end
        RuntimeState.Weather.particles[#RuntimeState.Weather.particles+1] = {
            object = part,
            kind = weatherName,
            index = i,
            baseSize = part.Size,
            appliedSize = 1,
            x = RuntimeState.Weather.random:NextNumber(-settings.radius, settings.radius),
            y = RuntimeState.Weather.random:NextNumber(-8,48),
            z = RuntimeState.Weather.random:NextNumber(-settings.radius, settings.radius),
            baseSpeed = weatherName == "Rain" and RuntimeState.Weather.random:NextNumber(34,52) or (weatherName == "Snow" and RuntimeState.Weather.random:NextNumber(3,8) or RuntimeState.Weather.random:NextNumber(18,30)),
            drift = RuntimeState.Weather.random:NextNumber(-3,3),
            phase = RuntimeState.Weather.random:NextNumber(0,6.28),
        }
    end
end

RuntimeState.weatherRenderConnection = RunService.Heartbeat:Connect(function(dt)
    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root or not RuntimeState.Weather.folder or not RuntimeState.Weather.folder.Parent then
        for _, particle in ipairs(RuntimeState.Weather.particles) do
            if particle.object then particle.object.Transparency = 1 end
        end
        return
    end
    local worldCenter = root.Position
    for _, particle in ipairs(RuntimeState.Weather.particles) do
        local settings = RuntimeState.Weather.settings[particle.kind]
        local object = particle.object
        if settings.enabled and particle.index <= settings.amount then
            particle.y = particle.y - particle.baseSpeed * settings.speed * dt
            if particle.kind == "Snow" then
                particle.x = particle.x + (particle.drift + settings.wind + math.sin(os.clock()*0.8 + particle.phase)*1.2) * dt * settings.speed
            elseif particle.kind == "Rain" then
                particle.x = particle.x + settings.wind * dt
            else
                particle.x = particle.x + settings.wind * dt * settings.speed
                particle.z = particle.z - 4 * settings.speed * dt
            end
            local radius = settings.radius
            if particle.y < -12 or math.abs(particle.x) > radius or math.abs(particle.z) > radius then
                particle.x = RuntimeState.Weather.random:NextNumber(-radius,radius)
                particle.y = RuntimeState.Weather.random:NextNumber(30,52)
                particle.z = RuntimeState.Weather.random:NextNumber(-radius,radius)
            end
            if particle.appliedSize ~= settings.size then
                object.Size = particle.baseSize * settings.size
                particle.appliedSize = settings.size
            end
            local worldPosition = worldCenter + Vector3.new(particle.x, particle.y, particle.z)
            object.Color = settings.color
            object.Transparency = particle.kind == "Rain" and 0.2 or 0.06
            if particle.kind == "Snow" then
                local sway = math.sin(os.clock()*1.5 + particle.phase) * 0.25
                object.CFrame = CFrame.new(worldPosition + Vector3.new(sway,0,0))
            elseif particle.kind == "Rain" then
                object.CFrame = CFrame.lookAt(worldPosition, worldPosition + Vector3.new(-0.8 + settings.wind * 0.035,-12,0)) * CFrame.Angles(math.rad(90),0,0)
            else
                local direction = Vector3.new(0.45,-0.25,-0.15)
                object.CFrame = CFrame.lookAt(worldPosition, worldPosition + direction)
                object.Transparency = 0.08 + 0.18 * (0.5 + 0.5 * math.sin(os.clock()*4 + particle.phase))
            end
        else
            object.Transparency = 1
        end
    end
end)

-- LOCAL CLIENT UTILITIES ------------------------------------------------------
-- Anti-AFK uses the LocalPlayer.Idled event and VirtualUser when available.
-- The calls are protected because some Roblox environments restrict VirtualUser.
RuntimeState.setAntiAfk = function(enabled)
    local utility = RuntimeState.Utility
    utility.antiAfkEnabled = enabled == true
    if utility.antiAfkConnection then
        pcall(function() utility.antiAfkConnection:Disconnect() end)
        utility.antiAfkConnection = nil
    end
    if not utility.antiAfkEnabled then
        return "IDLE PREVENTION DISABLED"
    end

    local virtualUser
    local serviceOk = pcall(function() virtualUser = game:GetService("VirtualUser") end)
    utility.antiAfkConnection = LocalPlayer.Idled:Connect(function()
        if not RuntimeState.active or not utility.antiAfkEnabled or not serviceOk or not virtualUser then
            return
        end
        pcall(function()
            virtualUser:CaptureController()
            virtualUser:ClickButton2(Vector2.new(0, 0))
        end)
    end)
    return serviceOk and "IDLE PREVENTION ACTIVE" or "ENABLED / VIRTUALUSER UNAVAILABLE"
end

-- Freecam runs locally. WASD moves, E/Q moves vertically, Shift boosts speed,
-- and holding the right mouse button turns the view. The original camera and
-- mouse state are restored when Freecam is switched off or the script stops.
RuntimeState.setFreecam = function(enabled)
    local state = RuntimeState.Utility.freecam
    enabled = enabled == true
    if state.enabled == enabled then
        return true
    end

    local function freezeHumanoid(humanoid)
        if not humanoid or not humanoid.Parent or state.frozenHumanoids[humanoid] then
            return
        end
        local snapshot = {}
        pcall(function() snapshot.walkSpeed = humanoid.WalkSpeed end)
        pcall(function() snapshot.jumpPower = humanoid.JumpPower end)
        pcall(function() snapshot.jumpHeight = humanoid.JumpHeight end)
        pcall(function() snapshot.useJumpPower = humanoid.UseJumpPower end)
        pcall(function() snapshot.autoRotate = humanoid.AutoRotate end)
        state.frozenHumanoids[humanoid] = snapshot
        pcall(function()
            humanoid.WalkSpeed = 0
            humanoid.JumpPower = 0
            humanoid.JumpHeight = 0
            humanoid.AutoRotate = false
        end)
    end

    local function restoreFrozenHumanoids()
        for humanoid, snapshot in pairs(state.frozenHumanoids or {}) do
            if humanoid and humanoid.Parent then
                pcall(function()
                    if snapshot.useJumpPower ~= nil then humanoid.UseJumpPower = snapshot.useJumpPower end
                    if snapshot.walkSpeed ~= nil then humanoid.WalkSpeed = snapshot.walkSpeed end
                    if snapshot.jumpPower ~= nil then humanoid.JumpPower = snapshot.jumpPower end
                    if snapshot.jumpHeight ~= nil then humanoid.JumpHeight = snapshot.jumpHeight end
                    if snapshot.autoRotate ~= nil then humanoid.AutoRotate = snapshot.autoRotate end
                end)
            end
        end
        table.clear(state.frozenHumanoids)
    end

    state.enabled = enabled

    if not enabled then
        for connectionName, connection in pairs(state.connections or {}) do
            if connection then pcall(function() connection:Disconnect() end) end
            state.connections[connectionName] = nil
        end
        state.keys = {}
        state.rotating = false
        for camera, original in pairs(state.originalCameras or {}) do
            if camera then
                pcall(function() camera.CameraType = original.cameraType end)
                pcall(function() camera.CameraSubject = original.cameraSubject end)
                pcall(function() camera.CFrame = original.cframe end)
                pcall(function() camera.FieldOfView = original.fieldOfView end)
            end
        end
        table.clear(state.originalCameras)
        if state.playerControls then
            pcall(function() state.playerControls:Enable() end)
            state.playerControls = nil
        end
        restoreFrozenHumanoids()
        if state.originalMouseBehavior ~= nil then
            pcall(function() UserInputService.MouseBehavior = state.originalMouseBehavior end)
            state.originalMouseBehavior = nil
        else
            pcall(function() UserInputService.MouseBehavior = Enum.MouseBehavior.Default end)
        end
        if state.originalMouseIconEnabled ~= nil then
            pcall(function() UserInputService.MouseIconEnabled = state.originalMouseIconEnabled end)
            state.originalMouseIconEnabled = nil
        end
        state.camera = nil
        return true
    end

    local camera = workspace.CurrentCamera
    if not camera then
        state.enabled = false
        return false
    end

    state.originalCameras = state.originalCameras or {}
    state.frozenHumanoids = state.frozenHumanoids or {}
    state.connections = {}
    state.originalCameras[camera] = {
        cameraType = camera.CameraType,
        cameraSubject = camera.CameraSubject,
        cframe = camera.CFrame,
        fieldOfView = camera.FieldOfView,
    }
    state.originalMouseBehavior = UserInputService.MouseBehavior
    state.originalMouseIconEnabled = UserInputService.MouseIconEnabled

    -- Prefer Roblox's own control module. If that module is inaccessible in a
    -- client environment, freeze the current humanoid as a local fallback.
    local controlsDisabled = false
    pcall(function()
        local playerScripts = LocalPlayer:FindFirstChild("PlayerScripts")
        local playerModuleScript = playerScripts and playerScripts:FindFirstChild("PlayerModule")
        if playerModuleScript then
            local playerModule = require(playerModuleScript)
            if playerModule and type(playerModule.GetControls) == "function" then
                state.playerControls = playerModule:GetControls()
                if state.playerControls and type(state.playerControls.Disable) == "function" then
                    state.playerControls:Disable()
                    controlsDisabled = true
                end
            end
        end
    end)
    if not controlsDisabled then
        local character = LocalPlayer.Character
        local humanoid = character and character:FindFirstChildOfClass("Humanoid")
        freezeHumanoid(humanoid)
    end

    state.camera = camera
    state.position = camera.CFrame.Position
    local lookVector = camera.CFrame.LookVector
    state.yaw = math.atan2(-lookVector.X, -lookVector.Z)
    state.pitch = math.asin(math.clamp(lookVector.Y, -1, 1))
    state.keys = {}
    state.rotating = false
    pcall(function() UserInputService.MouseBehavior = Enum.MouseBehavior.Default end)

    -- If the character respawns while Freecam is on and the PlayerModule path
    -- was unavailable, freeze the replacement Humanoid too.
    state.connections.characterAdded = LocalPlayer.CharacterAdded:Connect(function(character)
        if not state.enabled or state.playerControls then return end
        task.spawn(function()
            local humanoid = character:WaitForChild("Humanoid", 8)
            if state.enabled and humanoid then freezeHumanoid(humanoid) end
        end)
    end)

    state.connections.inputBegan = UserInputService.InputBegan:Connect(function(input)
        if not state.enabled or UserInputService:GetFocusedTextBox() then return end
        if input.UserInputType == Enum.UserInputType.MouseButton2 then
            state.rotating = true
            pcall(function() UserInputService.MouseBehavior = Enum.MouseBehavior.LockCurrentPosition end)
            pcall(function() UserInputService.MouseIconEnabled = false end)
            return
        end
        local key = input.KeyCode
        if key == Enum.KeyCode.W or key == Enum.KeyCode.A or key == Enum.KeyCode.S or key == Enum.KeyCode.D
            or key == Enum.KeyCode.E or key == Enum.KeyCode.Q or key == Enum.KeyCode.LeftShift
            or key == Enum.KeyCode.RightShift then
            state.keys[key] = true
        end
    end)
    state.connections.inputEnded = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton2 then
            state.rotating = false
            local restoreBehavior = state.originalMouseBehavior or Enum.MouseBehavior.Default
            pcall(function() UserInputService.MouseBehavior = restoreBehavior end)
            if state.originalMouseIconEnabled ~= nil then
                pcall(function() UserInputService.MouseIconEnabled = state.originalMouseIconEnabled end)
            end
            return
        end
        state.keys[input.KeyCode] = nil
    end)
    state.connections.inputChanged = UserInputService.InputChanged:Connect(function(input)
        if state.enabled and state.rotating and input.UserInputType == Enum.UserInputType.MouseMovement then
            state.yaw = state.yaw - input.Delta.X * state.sensitivity
            state.pitch = math.clamp(state.pitch - input.Delta.Y * state.sensitivity, math.rad(-88), math.rad(88))
        end
    end)
    state.connections.render = RunService.RenderStepped:Connect(function(deltaTime)
        if not RuntimeState.active or not state.enabled then return end
        local activeCamera = workspace.CurrentCamera
        if not activeCamera then return end
        if activeCamera ~= state.camera then
            if not state.originalCameras[activeCamera] then
                state.originalCameras[activeCamera] = {
                    cameraType = activeCamera.CameraType,
                    cameraSubject = activeCamera.CameraSubject,
                    cframe = activeCamera.CFrame,
                    fieldOfView = activeCamera.FieldOfView,
                }
            end
            state.camera = activeCamera
        end

        local right = (state.keys[Enum.KeyCode.D] and 1 or 0) - (state.keys[Enum.KeyCode.A] and 1 or 0)
        local up = (state.keys[Enum.KeyCode.E] and 1 or 0) - (state.keys[Enum.KeyCode.Q] and 1 or 0)
        local forward = (state.keys[Enum.KeyCode.W] and 1 or 0) - (state.keys[Enum.KeyCode.S] and 1 or 0)
        local moveVector = Vector3.new(right, up, -forward)
        local orientation = CFrame.Angles(0, state.yaw, 0) * CFrame.Angles(state.pitch, 0, 0)
        if moveVector.Magnitude > 0 then
            local boost = (state.keys[Enum.KeyCode.LeftShift] or state.keys[Enum.KeyCode.RightShift]) and 2.5 or 1
            local speed = math.min(math.clamp(tonumber(state.speed) or 48, 10, 1000) * boost, 1000)
            state.position = state.position + orientation:VectorToWorldSpace(moveVector.Unit) * speed * math.clamp(deltaTime, 0, 0.1)
        end
        activeCamera.CameraType = Enum.CameraType.Scriptable
        activeCamera.CFrame = CFrame.new(state.position) * orientation
        activeCamera.Focus = CFrame.new(state.position + activeCamera.CFrame.LookVector * 512)
    end)
    return true
end

-- Character Fly: Infinite Yield-style BodyVelocity + BodyGyro flight.
-- Camera-relative WASD movement with E/Q vertical control. Space/Ctrl are aliases.
-- PlatformStand intentionally stays false and ragdoll states remain disabled.
RuntimeState.setFly = function(enabled)
    local state = RuntimeState.Utility.fly
    enabled = enabled == true
    if state.enabled == enabled then return true end

    state.connections = state.connections or {}
    state.keys = state.keys or {}

    local function disconnectAll()
        for key, connection in pairs(state.connections) do
            if connection then pcall(function() connection:Disconnect() end) end
            state.connections[key] = nil
        end
    end

    local function removeMovers()
        if state.bodyVelocity then
            pcall(function() state.bodyVelocity:Destroy() end)
            state.bodyVelocity = nil
        end
        if state.bodyGyro then
            pcall(function() state.bodyGyro:Destroy() end)
            state.bodyGyro = nil
        end
        if state.root and state.root.Parent then
            pcall(function() state.root.AssemblyLinearVelocity = Vector3.zero end)
            pcall(function() state.root.AssemblyAngularVelocity = Vector3.zero end)
        end
    end

    local function restoreHumanoid()
        local humanoid = state.humanoid
        local original = state.humanoidOriginal
        if humanoid and humanoid.Parent and original then
            pcall(function()
                humanoid.PlatformStand = original.platformStand == true
                humanoid.AutoRotate = original.autoRotate
                humanoid.WalkSpeed = original.walkSpeed
                humanoid.JumpPower = original.jumpPower
                humanoid.JumpHeight = original.jumpHeight
                if original.useJumpPower ~= nil then humanoid.UseJumpPower = original.useJumpPower end
                if original.stateEnabled then
                    for humanoidState, wasEnabled in pairs(original.stateEnabled) do
                        pcall(function() humanoid:SetStateEnabled(humanoidState, wasEnabled) end)
                    end
                end
                if original.state then humanoid:ChangeState(original.state) end
            end)
        end
        state.humanoid = nil
        state.root = nil
        state.humanoidOriginal = nil
    end

    local function detachCharacter()
        removeMovers()
        restoreHumanoid()
    end

    local function attachCharacter(character)
        if not state.enabled or not character then return false end
        detachCharacter()
        local humanoid = character:FindFirstChildOfClass("Humanoid") or character:WaitForChild("Humanoid", 8)
        local root = character:FindFirstChild("HumanoidRootPart") or character:WaitForChild("HumanoidRootPart", 8)
        if not state.enabled or not humanoid or not root or not root:IsA("BasePart") then return false end

        local snapshot = {}
        pcall(function() snapshot.platformStand = humanoid.PlatformStand end)
        pcall(function() snapshot.autoRotate = humanoid.AutoRotate end)
        pcall(function() snapshot.walkSpeed = humanoid.WalkSpeed end)
        pcall(function() snapshot.jumpPower = humanoid.JumpPower end)
        pcall(function() snapshot.jumpHeight = humanoid.JumpHeight end)
        pcall(function() snapshot.useJumpPower = humanoid.UseJumpPower end)
        pcall(function() snapshot.state = humanoid:GetState() end)
        snapshot.stateEnabled = {}
        for _, humanoidState in ipairs({Enum.HumanoidStateType.Ragdoll, Enum.HumanoidStateType.FallingDown}) do
            pcall(function()
                snapshot.stateEnabled[humanoidState] = humanoid:GetStateEnabled(humanoidState)
            end)
        end
        state.humanoid = humanoid
        state.root = root
        state.humanoidOriginal = snapshot

        pcall(function()
            humanoid.PlatformStand = false
            humanoid.AutoRotate = false
            humanoid.WalkSpeed = 0
            humanoid.JumpPower = 0
            humanoid.JumpHeight = 0
            humanoid:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
            humanoid:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
            humanoid:ChangeState(Enum.HumanoidStateType.Freefall)
        end)

        -- The classic Infinite Yield flight feel: BodyGyro follows the camera,
        -- BodyVelocity applies camera-relative motion and holds position at rest.
        local gyro = Instance.new("BodyGyro")
        gyro.Name = "CATScript_IYFlyGyro"
        gyro.P = 9e4
        gyro.D = 1000
        gyro.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        gyro.CFrame = (workspace.CurrentCamera and workspace.CurrentCamera.CFrame) or root.CFrame
        gyro.Parent = root
        state.bodyGyro = gyro

        local bodyVelocity = Instance.new("BodyVelocity")
        bodyVelocity.Name = "CATScript_IYFlyVelocity"
        bodyVelocity.MaxForce = Vector3.new(9e9, 9e9, 9e9)
        bodyVelocity.Velocity = Vector3.zero
        bodyVelocity.Parent = root
        state.bodyVelocity = bodyVelocity
        return true
    end

    if not enabled then
        state.enabled = false
        disconnectAll()
        table.clear(state.keys)
        detachCharacter()
        if state.playerControls then
            pcall(function() state.playerControls:Enable() end)
            state.playerControls = nil
        end
        return true
    end

    local character = LocalPlayer.Character
    if not character then return false end
    state.enabled = true

    -- Suspend default character controls while BodyVelocity owns movement.
    pcall(function()
        local playerScripts = LocalPlayer:FindFirstChild("PlayerScripts")
        local moduleScript = playerScripts and playerScripts:FindFirstChild("PlayerModule")
        if moduleScript then
            local module = require(moduleScript)
            if module and type(module.GetControls) == "function" then
                state.playerControls = module:GetControls()
                if state.playerControls and type(state.playerControls.Disable) == "function" then
                    state.playerControls:Disable()
                end
            end
        end
    end)

    if not attachCharacter(character) then
        state.enabled = false
        if state.playerControls then
            pcall(function() state.playerControls:Enable() end)
            state.playerControls = nil
        end
        return false
    end

    state.connections.characterAdded = LocalPlayer.CharacterAdded:Connect(function(newCharacter)
        task.spawn(function()
            if not state.enabled then return end
            local attached = attachCharacter(newCharacter)
            if not attached and state.enabled then
                task.wait(0.35)
                if state.enabled then attachCharacter(newCharacter) end
            end
        end)
    end)

    local keyMap = {
        [Enum.KeyCode.W] = "forward",
        [Enum.KeyCode.S] = "back",
        [Enum.KeyCode.A] = "left",
        [Enum.KeyCode.D] = "right",
        [Enum.KeyCode.E] = "up",
        [Enum.KeyCode.Q] = "down",
        [Enum.KeyCode.Space] = "up",
        [Enum.KeyCode.LeftControl] = "down",
        [Enum.KeyCode.RightControl] = "down",
    }

    state.connections.inputBegan = UserInputService.InputBegan:Connect(function(input, processed)
        if not state.enabled or processed or UserInputService:GetFocusedTextBox() then return end
        local action = keyMap[input.KeyCode]
        if action then state.keys[action] = true end
    end)
    state.connections.inputEnded = UserInputService.InputEnded:Connect(function(input)
        local action = keyMap[input.KeyCode]
        if action then state.keys[action] = nil end
    end)

    state.connections.render = RunService.RenderStepped:Connect(function()
        if not RuntimeState.active or not state.enabled then return end
        local root = state.root
        local bodyVelocity = state.bodyVelocity
        local bodyGyro = state.bodyGyro
        if not root or not root.Parent or not bodyVelocity or not bodyGyro then return end
        local camera = workspace.CurrentCamera
        if not camera then return end

        local forward = (state.keys.forward and 1 or 0) - (state.keys.back and 1 or 0)
        local side = (state.keys.right and 1 or 0) - (state.keys.left and 1 or 0)
        local vertical = (state.keys.up and 1 or 0) - (state.keys.down and 1 or 0)
        local direction = camera.CFrame.LookVector * forward
            + camera.CFrame.RightVector * side
            + Vector3.new(0, vertical, 0)
        local speed = math.clamp(tonumber(state.speed) or 80, 10, 1000)
        bodyVelocity.Velocity = direction * speed
        bodyGyro.CFrame = camera.CFrame

        if state.humanoid and state.humanoid.Parent then
            pcall(function()
                -- Keep Infinite Yield-style movement without its ragdoll-like
                -- PlatformStand=true behavior.
                if state.humanoid.PlatformStand then state.humanoid.PlatformStand = false end
                if state.humanoid.AutoRotate then state.humanoid.AutoRotate = false end
            end)
        end
    end)
    return true
end

-- No GUI hides the game's local HUD and world GUI layers while keeping CATScript
-- visible. Original Enabled flags and CoreGui states are restored on toggle-off
-- and during runtime cleanup. Newly added game GUI layers are hidden as well.
RuntimeState.setNoGui = function(enabled)
    local state = RuntimeState.Utility.noGui
    enabled = enabled == true
    RuntimeState.Utility.noGuiEnabled = enabled
    if state.enabled == enabled then
        return enabled and "GAME GUI HIDDEN / CATScript VISIBLE" or "GAME GUI RESTORED"
    end
    state.enabled = enabled

    local function isCatScriptOwned(object)
        local current = object
        while current and current ~= game do
            if current == ScreenGui or current.Name == GUI_NAME then return true end
            current = current.Parent
        end
        return false
    end

    local function isGameGuiLayer(object)
        return object and (object:IsA("ScreenGui") or object:IsA("BillboardGui") or object:IsA("SurfaceGui"))
    end

    local function hideGameGui(object)
        if not state.enabled or not isGameGuiLayer(object) or isCatScriptOwned(object) then return end
        if state.originals[object] == nil then
            local ok, previous = pcall(function() return object.Enabled end)
            if not ok then return end
            state.originals[object] = previous
        end
        pcall(function() object.Enabled = false end)
    end

    if enabled then
        local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
        if playerGui then
            for _, object in ipairs(playerGui:GetDescendants()) do
                hideGameGui(object)
            end
            state.playerGuiAddedConnection = playerGui.DescendantAdded:Connect(hideGameGui)
        end

        for _, object in ipairs(workspace:GetDescendants()) do
            if object:IsA("BillboardGui") or object:IsA("SurfaceGui") then
                hideGameGui(object)
            end
        end
        state.workspaceAddedConnection = workspace.DescendantAdded:Connect(function(object)
            if object:IsA("BillboardGui") or object:IsA("SurfaceGui") then
                hideGameGui(object)
            end
        end)

        pcall(function()
            local starterGui = game:GetService("StarterGui")
            for _, coreType in ipairs(Enum.CoreGuiType:GetEnumItems()) do
                if coreType.Name ~= "All" then
                    local readOk, previous = pcall(function()
                        return starterGui:GetCoreGuiEnabled(coreType)
                    end)
                    if readOk then
                        state.coreGuiOriginals[coreType] = previous
                        pcall(function() starterGui:SetCoreGuiEnabled(coreType, false) end)
                    end
                end
            end
        end)
        return "GAME GUI HIDDEN / CATScript VISIBLE"
    end

    if state.playerGuiAddedConnection then
        pcall(function() state.playerGuiAddedConnection:Disconnect() end)
        state.playerGuiAddedConnection = nil
    end
    if state.workspaceAddedConnection then
        pcall(function() state.workspaceAddedConnection:Disconnect() end)
        state.workspaceAddedConnection = nil
    end
    for object, previous in pairs(state.originals) do
        if object then pcall(function() object.Enabled = previous end) end
    end
    table.clear(state.originals)
    for coreType, previous in pairs(state.coreGuiOriginals) do
        pcall(function()
            game:GetService("StarterGui"):SetCoreGuiEnabled(coreType, previous)
        end)
    end
    table.clear(state.coreGuiOriginals)
    return "GAME GUI RESTORED"
end

-- No Texture changes only local client visuals: materials become SmoothPlastic,
-- map textures/decals are hidden, and SurfaceAppearance maps are detached. All
-- original properties are retained and restored when the toggle is disabled.
RuntimeState.Utility.noTextureApplyObject = function(object)
    local state = RuntimeState.Utility.noTexture
    if not state.enabled or not object or not object.Parent then return end
    local ancestor = object
    while ancestor and ancestor ~= workspace do
        if string.sub(ancestor.Name, 1, 10) == "CATScript_" then return end
        ancestor = ancestor.Parent
    end

    local original = state.originals[object]
    if original then return end
    if object:IsA("BasePart") then
        original = {kind = "BasePart", material = object.Material}
        pcall(function() original.materialVariant = object.MaterialVariant end)
        if object:IsA("MeshPart") then
            pcall(function() original.textureID = object.TextureID end)
        end
        state.originals[object] = original
        pcall(function() object.Material = Enum.Material.SmoothPlastic end)
        pcall(function() object.MaterialVariant = "" end)
        if object:IsA("MeshPart") then pcall(function() object.TextureID = "" end) end
    elseif object:IsA("SpecialMesh") then
        original = {kind = "SpecialMesh", textureId = object.TextureId}
        state.originals[object] = original
        pcall(function() object.TextureId = "" end)
    elseif object:IsA("SurfaceAppearance") then
        original = {kind = "SurfaceAppearance", parent = object.Parent}
        state.originals[object] = original
        pcall(function() object.Parent = nil end)
    elseif object:IsA("Texture") or object:IsA("Decal") then
        original = {kind = "ImageTexture", transparency = object.Transparency}
        state.originals[object] = original
        pcall(function() object.Transparency = 1 end)
    end
end

RuntimeState.setNoTexture = function(enabled)
    local state = RuntimeState.Utility.noTexture
    enabled = enabled == true
    state.enabled = enabled
    RuntimeState.Utility.noTextureEnabled = enabled
    state.generation += 1
    local generation = state.generation

    if state.addedConnection then
        pcall(function() state.addedConnection:Disconnect() end)
        state.addedConnection = nil
    end

    if enabled then
        state.addedConnection = workspace.DescendantAdded:Connect(function(object)
            RuntimeState.Utility.noTextureApplyObject(object)
        end)
        task.spawn(function()
            local descendants = workspace:GetDescendants()
            for index, object in ipairs(descendants) do
                if not RuntimeState.active or not state.enabled or state.generation ~= generation then return end
                RuntimeState.Utility.noTextureApplyObject(object)
                if index % 250 == 0 then task.wait() end
            end
            if state.enabled and state.generation == generation and RuntimeState.NoTextureRow then
                RuntimeState.NoTextureRow.setStatus("TEXTURES FLATTENED")
            end
        end)
        return "APPLYING FLAT MATERIALS..."
    end

    for object, original in pairs(state.originals) do
        if object then
            if original.kind == "BasePart" and object.Parent then
                pcall(function() object.Material = original.material end)
                if original.materialVariant ~= nil then pcall(function() object.MaterialVariant = original.materialVariant end) end
                if original.textureID ~= nil then pcall(function() object.TextureID = original.textureID end) end
            elseif original.kind == "SpecialMesh" and object.Parent then
                pcall(function() object.TextureId = original.textureId end)
            elseif original.kind == "SurfaceAppearance" and original.parent then
                pcall(function() object.Parent = original.parent end)
            elseif original.kind == "ImageTexture" and object.Parent then
                pcall(function() object.Transparency = original.transparency end)
            end
        end
    end
    table.clear(state.originals)
    if RuntimeState.NoTextureRow then RuntimeState.NoTextureRow.setStatus("ORIGINAL TEXTURES RESTORED") end
    return "ORIGINAL TEXTURES RESTORED"
end

-- Settings page: menu keybind, Auto Save, performance switches, and background controls.
RuntimeState.SettingsCard = make("Frame", {
    Name = "SettingsCard",
    BackgroundColor3 = COLORS.panel,
    BackgroundTransparency = 0.24,
    BorderSizePixel = 0,
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, 0, 0, 2080),
    Visible = false,
}, RuntimeState.Content)
corner(RuntimeState.SettingsCard, 10)
stroke(RuntimeState.SettingsCard, COLORS.outline, 0, 1)

-- Player controls. These are client-side controls; experiences may enforce
-- their own server-authoritative movement rules.
RuntimeState.applyPlayerMovement = function()
    local character = LocalPlayer.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return false end
    local state = RuntimeState.PlayerTools
    if not state.originals[humanoid] then
        local snapshot = {}
        pcall(function() snapshot.walkSpeed = humanoid.WalkSpeed end)
        pcall(function() snapshot.jumpPower = humanoid.JumpPower end)
        pcall(function() snapshot.jumpHeight = humanoid.JumpHeight end)
        pcall(function() snapshot.useJumpPower = humanoid.UseJumpPower end)
        state.originals[humanoid] = snapshot
    end
    pcall(function() humanoid.WalkSpeed = state.walkSpeed end)
    pcall(function() humanoid.UseJumpPower = true; humanoid.JumpPower = state.jumpPower end)
    return true
end
RuntimeState.setPlayerFov = function(value)
    RuntimeState.PlayerTools.fov = math.clamp(tonumber(value) or 70, 30, 120)
    local camera = workspace.CurrentCamera
    if camera then
        if RuntimeState.PlayerTools.originalFov == nil then pcall(function() RuntimeState.PlayerTools.originalFov = camera.FieldOfView end) end
        pcall(function() camera.FieldOfView = RuntimeState.PlayerTools.fov end)
    end
end
RuntimeState.setNoclip = function(enabled)
    local state = RuntimeState.PlayerTools
    state.noclipEnabled = enabled == true
    if state.noclipConnection then state.noclipConnection:Disconnect(); state.noclipConnection = nil end
    local function apply()
        local character = LocalPlayer.Character
        if not character then return end
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") then
                if state.noclipEnabled then
                    if state.collisionOriginals[part] == nil then state.collisionOriginals[part] = part.CanCollide end
                    part.CanCollide = false
                elseif state.collisionOriginals[part] ~= nil then
                    part.CanCollide = state.collisionOriginals[part]; state.collisionOriginals[part] = nil
                end
            end
        end
    end
    if state.noclipEnabled then
        state.noclipConnection = RunService.Stepped:Connect(apply)
        apply()
    else
        apply()
        for part, value in pairs(state.collisionOriginals) do
            if part and part.Parent then pcall(function() part.CanCollide = value end) end
            state.collisionOriginals[part] = nil
        end
    end
    return state.noclipEnabled
end
RuntimeState.setFloat = function(enabled)
    local state = RuntimeState.PlayerTools
    state.floatEnabled = enabled == true

    if state.floatConnection then
        state.floatConnection:Disconnect()
        state.floatConnection = nil
    end
    if state.floatPart then
        state.floatPart:Destroy()
        state.floatPart = nil
    end
    state.floatPlatformY = nil
    state.floatWasAirborne = false
    state.floatLastHumanoid = nil

    if not state.floatEnabled then return false end

    -- Платформа появляется только после прыжка/начала падения.
    -- Её высота фиксируется в момент появления, а X/Z следуют за игроком,
    -- чтобы игрок мог приземлиться на неё и стоять.
    local function createOrMovePlatform(root, humanoid, resetHeight)
        if not state.floatPart or not state.floatPart.Parent then
            local platform = Instance.new("Part")
            platform.Name = "CATScript_FloatPlatform"
            platform.Size = Vector3.new(8, 0.4, 8)
            platform.Transparency = 1
            platform.Anchored = true
            platform.CanCollide = true
            platform.CanTouch = false
            platform.CanQuery = false
            platform.CastShadow = false
            platform.Material = Enum.Material.SmoothPlastic
            platform.Parent = workspace
            state.floatPart = platform
            resetHeight = true
        end

        if resetHeight or not state.floatPlatformY then
            state.floatPlatformY = root.Position.Y
                - (humanoid.HipHeight + root.Size.Y * 0.5 + 0.25)
        end

        state.floatPart.CFrame = CFrame.new(
            root.Position.X,
            state.floatPlatformY,
            root.Position.Z
        )
    end

    state.floatConnection = RunService.Heartbeat:Connect(function()
        if not state.floatEnabled then return end

        local character = LocalPlayer.Character
        local humanoid = character and character:FindFirstChildOfClass("Humanoid")
        local root = character and character:FindFirstChild("HumanoidRootPart")

        if not humanoid or not root then
            if state.floatPart then
                state.floatPart:Destroy()
                state.floatPart = nil
            end
            state.floatPlatformY = nil
            state.floatWasAirborne = false
            state.floatLastHumanoid = nil
            return
        end

        if state.floatLastHumanoid ~= humanoid then
            state.floatLastHumanoid = humanoid
            state.floatWasAirborne = false
            if state.floatPart then
                state.floatPart:Destroy()
                state.floatPart = nil
            end
            state.floatPlatformY = nil
        end

        local humanoidState = humanoid:GetState()
        local isJumping = humanoidState == Enum.HumanoidStateType.Jumping
        local isAirborne = humanoidState == Enum.HumanoidStateType.Freefall
            or isJumping
            or humanoidState == Enum.HumanoidStateType.FallingDown

        if isJumping then
            -- На каждом новом прыжке ставим платформу прямо под ногами.
            createOrMovePlatform(root, humanoid, true)
            state.floatWasAirborne = true
        elseif isAirborne then
            -- Если персонаж упал с края, создаём платформу под ним при первом
            -- обнаружении воздуха, затем сохраняем её высоту для приземления.
            if not state.floatPart or not state.floatPart.Parent then
                createOrMovePlatform(root, humanoid, true)
            else
                createOrMovePlatform(root, humanoid, false)
            end
            state.floatWasAirborne = true
        elseif state.floatPart and state.floatPart.Parent then
            -- После приземления оставляем платформу на месте по высоте,
            -- чтобы на ней можно было спокойно стоять.
            createOrMovePlatform(root, humanoid, false)
            state.floatWasAirborne = false
        end
    end)

    return true
end
RuntimeState.setVehicleFly = function(enabled)
    local state = RuntimeState.PlayerTools.vehicleFly
    enabled = enabled == true
    if state.enabled == enabled then return true end
    state.enabled = enabled
    for _, connection in pairs(state.connections) do pcall(function() connection:Disconnect() end) end
    table.clear(state.connections)
    state.keys = {}
    if state.mover then state.mover:Destroy(); state.mover = nil end
    if state.gyro then state.gyro:Destroy(); state.gyro = nil end
    if not enabled then state.seat = nil; return true end

    state.connections.began = UserInputService.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Keyboard and not UserInputService:GetFocusedTextBox() then
            state.keys[input.KeyCode] = true
        end
    end)
    state.connections.ended = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Keyboard then
            state.keys[input.KeyCode] = nil
        end
    end)

    state.connections.step = RunService.Heartbeat:Connect(function()
        if not state.enabled then return end
        local character = LocalPlayer.Character
        local humanoid = character and character:FindFirstChildOfClass("Humanoid")
        local seat = humanoid and humanoid.SeatPart

        -- Поддерживаем как VehicleSeat, так и обычный Seat в пользовательском транспорте.
        local isVehicleSeat = seat and (seat:IsA("VehicleSeat") or seat:IsA("Seat"))
        if not isVehicleSeat then
            if state.mover then state.mover:Destroy(); state.mover = nil end
            if state.gyro then state.gyro:Destroy(); state.gyro = nil end
            state.seat = nil
            return
        end

        local root = seat.AssemblyRootPart or seat
        if not root or not root:IsA("BasePart") or root.Anchored then
            if state.mover then state.mover:Destroy(); state.mover = nil end
            if state.gyro then state.gyro:Destroy(); state.gyro = nil end
            state.seat = nil
            return
        end

        if state.seat ~= root then
            if state.mover then state.mover:Destroy() end
            if state.gyro then state.gyro:Destroy() end
            state.seat = root
            state.mover = Instance.new("BodyVelocity")
            state.mover.Name = "CATScriptVehicleFlyVelocity"
            state.mover.MaxForce = Vector3.new(1e9, 1e9, 1e9)
            state.mover.P = 50000
            state.mover.Velocity = Vector3.zero
            state.mover.Parent = root

            state.gyro = Instance.new("BodyGyro")
            state.gyro.Name = "CATScriptVehicleFlyGyro"
            state.gyro.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
            state.gyro.P = 50000
            state.gyro.D = 1200
            state.gyro.CFrame = root.CFrame
            state.gyro.Parent = root
        end

        local camera = workspace.CurrentCamera
        if not camera or not state.mover or not state.gyro then return end

        local forward = (state.keys[Enum.KeyCode.W] and 1 or 0) - (state.keys[Enum.KeyCode.S] and 1 or 0)
        local right = (state.keys[Enum.KeyCode.D] and 1 or 0) - (state.keys[Enum.KeyCode.A] and 1 or 0)
        local up = (state.keys[Enum.KeyCode.Space] and 1 or 0)
            - ((state.keys[Enum.KeyCode.LeftControl] or state.keys[Enum.KeyCode.RightControl] or state.keys[Enum.KeyCode.Q]) and 1 or 0)

        local look = camera.CFrame.LookVector
        local moveDirection = look * forward + camera.CFrame.RightVector * right + Vector3.new(0, up, 0)
        if moveDirection.Magnitude > 1 then moveDirection = moveDirection.Unit end

        state.mover.Velocity = moveDirection * (RuntimeState.PlayerTools.vehicleFlySpeed or 80)
        state.gyro.CFrame = CFrame.lookAt(root.Position, root.Position + look, Vector3.yAxis)
    end)
    return true
end
RuntimeState.setAimAssist = function(enabled)
    local state = RuntimeState.PlayerTools
    state.aimEnabled = enabled == true
    for _, connection in pairs(state.aimConnections) do pcall(function() connection:Disconnect() end) end
    table.clear(state.aimConnections)
    if not state.aimEnabled then state.aimHeld = false; return false end
    state.aimConnections.began = UserInputService.InputBegan:Connect(function(input, processed)
        if input.UserInputType == Enum.UserInputType.MouseButton2 and not processed then state.aimHeld = true end
    end)
    state.aimConnections.ended = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton2 then state.aimHeld = false end
    end)
    state.aimConnections.render = RunService.RenderStepped:Connect(function(dt)
        if not state.aimEnabled or not state.aimHeld then return end
        local camera = workspace.CurrentCamera; if not camera then return end
        local center = camera.ViewportSize * 0.5
        local bestPart, bestScore = nil, math.huge
        local ownChar = LocalPlayer.Character
        local ownRoot = ownChar and ownChar:FindFirstChild("HumanoidRootPart")
        if not ownRoot then return end
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local targetHum = player.Character:FindFirstChildOfClass("Humanoid")
                local targetRoot = player.Character:FindFirstChild("HumanoidRootPart")
                local targetPart = player.Character:FindFirstChild("Head") or targetRoot
                if targetHum and targetHum.Health > 0 and targetRoot and targetPart and (targetRoot.Position-ownRoot.Position).Magnitude <= state.aimRange then
                    local point, visible = camera:WorldToViewportPoint(targetPart.Position)
                    if visible then
                        local screenDistance = (Vector2.new(point.X,point.Y)-center).Magnitude
                        if screenDistance <= state.aimFov and screenDistance < bestScore then bestScore=screenDistance; bestPart=targetPart end
                    end
                end
            end
        end
        if bestPart then
            local desired = CFrame.lookAt(camera.CFrame.Position, bestPart.Position)
            local smooth = math.clamp(10 / state.aimSmoothness, 0.01, 1)
            camera.CFrame = camera.CFrame:Lerp(desired, 1 - math.pow(1 - smooth, math.clamp(dt*60,0,3)))
        end
    end)
    return true
end

RuntimeState.PlayerCard = make("Frame", {
    Name = "PlayerCard",
    BackgroundColor3 = COLORS.panel,
    BackgroundTransparency = 0.24,
    BorderSizePixel = 0,
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, 0, 0, 250),
    Visible = false,
}, RuntimeState.Content)
corner(RuntimeState.PlayerCard, 10)
stroke(RuntimeState.PlayerCard, COLORS.outline, 0, 1)
make("UIPadding", {
    PaddingTop = UDim.new(0, 14),
    PaddingBottom = UDim.new(0, 12),
    PaddingLeft = UDim.new(0, 16),
    PaddingRight = UDim.new(0, 16),
}, RuntimeState.PlayerCard)
RuntimeState.PlayerTitle = addLabel(RuntimeState.PlayerCard, "Player", 12, COLORS.text, Enum.Font.GothamBold)
RuntimeState.PlayerTitle.Size = UDim2.new(1, 0, 0, 18)
RuntimeState.PlayerSubtitle = addLabel(RuntimeState.PlayerCard, "Character movement tools", 9, COLORS.muted, Enum.Font.Gotham)
RuntimeState.PlayerSubtitle.Position = UDim2.fromOffset(0, 19)
RuntimeState.PlayerSubtitle.Size = UDim2.new(1, 0, 0, 15)

RuntimeState.FlyRow = make("Frame", {
    Name = "FlyRow", Position = UDim2.fromOffset(0, 46), Size = UDim2.new(1, 0, 0, 74),
    BackgroundColor3 = COLORS.panel2, BackgroundTransparency = 0.18, BorderSizePixel = 0,
}, RuntimeState.PlayerCard)
corner(RuntimeState.FlyRow, 8)
stroke(RuntimeState.FlyRow, COLORS.outline, 0.35, 1)
RuntimeState.FlyTitle = addLabel(RuntimeState.FlyRow, "FLY", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.FlyTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.FlyTitle.Size = UDim2.new(1, -145, 0, 17)
RuntimeState.FlyDescription = addLabel(RuntimeState.FlyRow, "INFINITE YIELD STYLE  /  WASD move  /  E up  /  Q down", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.FlyDescription.Position = UDim2.fromOffset(12, 28)
RuntimeState.FlyDescription.Size = UDim2.new(1, -145, 0, 14)
RuntimeState.FlyStatus = addLabel(RuntimeState.FlyRow, "DISABLED", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.FlyStatus.Position = UDim2.fromOffset(12, 51)
RuntimeState.FlyStatus.Size = UDim2.new(1, -145, 0, 12)
RuntimeState.FlyToggle = make("TextButton", {
    Name = "FlyToggle", Text = "", AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(44, 24), BackgroundColor3 = COLORS.dim, BorderSizePixel = 0,
}, RuntimeState.FlyRow)
corner(RuntimeState.FlyToggle, 12)
RuntimeState.FlyToggleKnob = make("Frame", {
    Name = "Knob", Position = UDim2.fromOffset(4, 3), Size = UDim2.fromOffset(18, 18),
    BackgroundColor3 = COLORS.white, BorderSizePixel = 0,
}, RuntimeState.FlyToggle)
corner(RuntimeState.FlyToggleKnob, 9)
RuntimeState.FlyToggle.MouseButton1Click:Connect(function()
    local enabled = not RuntimeState.Utility.fly.enabled
    local ok = RuntimeState.setFly(enabled)
    if not ok then enabled = false end
    RuntimeState.FlyToggle.BackgroundColor3 = enabled and COLORS.accent or COLORS.dim
    RuntimeState.FlyToggleKnob.Position = enabled and UDim2.fromOffset(22, 3) or UDim2.fromOffset(4, 3)
    RuntimeState.FlyStatus.Text = enabled and "FLY ACTIVE" or (ok and "DISABLED" or "CHARACTER NOT READY")
    RuntimeState.FlyStatus.TextColor3 = enabled and COLORS.accent or COLORS.dim
    if scheduleConfigSave then scheduleConfigSave() end
end)

RuntimeState.FlySpeedRow = make("Frame", {
    Name = "FlySpeedRow", Position = UDim2.fromOffset(0, 132), Size = UDim2.new(1, 0, 0, 72),
    BackgroundColor3 = COLORS.panel2, BackgroundTransparency = 0.18, BorderSizePixel = 0,
}, RuntimeState.PlayerCard)
corner(RuntimeState.FlySpeedRow, 8)
stroke(RuntimeState.FlySpeedRow, COLORS.outline, 0.35, 1)
RuntimeState.FlySpeedTitle = addLabel(RuntimeState.FlySpeedRow, "FLY SPEED", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.FlySpeedTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.FlySpeedTitle.Size = UDim2.new(1, -120, 0, 16)
RuntimeState.FlySpeedValue = addLabel(RuntimeState.FlySpeedRow, tostring(RuntimeState.Utility.fly.speed) .. " studs/s", 8, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
RuntimeState.FlySpeedValue.Position = UDim2.new(1, -118, 0, 8)
RuntimeState.FlySpeedValue.Size = UDim2.fromOffset(106, 16)
RuntimeState.FlySpeedTrack = make("Frame", {
    Name = "FlySpeedTrack", Position = UDim2.fromOffset(13, 39), Size = UDim2.new(1, -26, 0, 5),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, ZIndex = 2,
}, RuntimeState.FlySpeedRow)
corner(RuntimeState.FlySpeedTrack, 3)
RuntimeState.FlySpeedFill = make("Frame", {
    Name = "FlySpeedFill", Size = UDim2.new((RuntimeState.Utility.fly.speed - 10) / 990, 0, 1, 0),
    BackgroundColor3 = COLORS.accent, BorderSizePixel = 0, ZIndex = 3,
}, RuntimeState.FlySpeedTrack)
corner(RuntimeState.FlySpeedFill, 3)
RuntimeState.FlySpeedKnob = make("Frame", {
    Name = "FlySpeedKnob", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.new((RuntimeState.Utility.fly.speed - 10) / 990, 0, 0.5, 0),
    Size = UDim2.fromOffset(11, 11), BackgroundColor3 = COLORS.white, BorderSizePixel = 0, ZIndex = 5,
}, RuntimeState.FlySpeedTrack)
corner(RuntimeState.FlySpeedKnob, 6)
stroke(RuntimeState.FlySpeedKnob, COLORS.accent, 0, 1)
RuntimeState.FlySpeedInput = make("TextButton", {
    Name = "FlySpeedInput", Position = UDim2.fromOffset(8, 30), Size = UDim2.new(1, -16, 0, 20),
    Text = "", AutoButtonColor = false, BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 8,
}, RuntimeState.FlySpeedRow)
RuntimeState.setFlySpeedFromX = function(pointerX)
    local track = RuntimeState.FlySpeedTrack
    local alpha = math.clamp((pointerX - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X, 1), 0, 1)
    local speed = math.clamp(math.floor(10 + alpha * 990 + 0.5), 10, 1000)
    local normalized = (speed - 10) / 990
    RuntimeState.Utility.fly.speed = speed
    RuntimeState.FlySpeedFill.Size = UDim2.new(normalized, 0, 1, 0)
    RuntimeState.FlySpeedKnob.Position = UDim2.new(normalized, 0, 0.5, 0)
    RuntimeState.FlySpeedValue.Text = tostring(speed) .. " studs/s"
    if scheduleConfigSave then scheduleConfigSave() end
end
RuntimeState.FlySpeedInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.flySpeedDragging = true
        RuntimeState.setFlySpeedFromX(input.Position.X)
    end
end)
RuntimeState.flySpeedInputChangedConnection = UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.flySpeedDragging then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        RuntimeState.setFlySpeedFromX(UserInputService:GetMouseLocation().X)
    elseif input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.setFlySpeedFromX(input.Position.X)
    end
end)
RuntimeState.flySpeedInputEndedConnection = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.flySpeedDragging = false
    end
end)

-- Compact reusable slider/toggle builders for the remaining Player controls.
RuntimeState.PlayerCard.Size = UDim2.new(1, 0, 0, 900)
RuntimeState.playerSliderControls = {}
RuntimeState.createPlayerSlider = function(key, title, y, minimum, maximum, suffix, getter, setter)
    local row = make("Frame", {Name=key.."Row", Position=UDim2.fromOffset(0,y), Size=UDim2.new(1,0,0,58), BackgroundColor3=COLORS.panel2, BackgroundTransparency=0.18, BorderSizePixel=0}, RuntimeState.PlayerCard)
    corner(row,8); stroke(row,COLORS.outline,0.35,1)
    local heading = addLabel(row,title,9,COLORS.text,Enum.Font.GothamSemibold); heading.Position=UDim2.fromOffset(12,7); heading.Size=UDim2.new(1,-125,0,15)
    local valueLabel = addLabel(row,"",8,COLORS.accent,Enum.Font.GothamBold,Enum.TextXAlignment.Right); valueLabel.Position=UDim2.new(1,-112,0,7); valueLabel.Size=UDim2.fromOffset(100,15)
    local track = make("Frame",{Position=UDim2.fromOffset(12,36),Size=UDim2.new(1,-24,0,5),BackgroundColor3=COLORS.background,BorderSizePixel=0},row); corner(track,3)
    local fill = make("Frame",{Size=UDim2.new(0,0,1,0),BackgroundColor3=COLORS.accent,BorderSizePixel=0},track); corner(fill,3)
    local knob = make("Frame",{AnchorPoint=Vector2.new(0.5,0.5),Position=UDim2.new(0,0,0.5,0),Size=UDim2.fromOffset(11,11),BackgroundColor3=COLORS.white,BorderSizePixel=0},track); corner(knob,6); stroke(knob,COLORS.accent,0,1)
    local hit = make("TextButton",{Position=UDim2.fromOffset(6,25),Size=UDim2.new(1,-12,0,25),Text="",BackgroundTransparency=1,AutoButtonColor=false},row)
    local control={key=key,min=minimum,max=maximum,suffix=suffix or "",track=track,fill=fill,knob=knob,label=valueLabel,setter=setter,dragging=false}
    local function refresh(value,save)
        value=math.clamp(math.floor((tonumber(value) or minimum)+0.5),minimum,maximum); control.value=value
        local a=(value-minimum)/(maximum-minimum); fill.Size=UDim2.new(a,0,1,0); knob.Position=UDim2.new(a,0,0.5,0); valueLabel.Text=tostring(value)..control.suffix
        setter(value); if save and scheduleConfigSave then scheduleConfigSave() end
    end
    control.refresh=refresh; hit.InputBegan:Connect(function(input) if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then control.dragging=true; refresh(minimum+(math.clamp((input.Position.X-track.AbsolutePosition.X)/math.max(track.AbsoluteSize.X,1),0,1))*(maximum-minimum),true) end end)
    control.changed=UserInputService.InputChanged:Connect(function(input) if control.dragging then local x=input.UserInputType==Enum.UserInputType.MouseMovement and UserInputService:GetMouseLocation().X or (input.UserInputType==Enum.UserInputType.Touch and input.Position.X); if x then refresh(minimum+math.clamp((x-track.AbsolutePosition.X)/math.max(track.AbsoluteSize.X,1),0,1)*(maximum-minimum),true) end end end)
    control.ended=UserInputService.InputEnded:Connect(function(input) if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then control.dragging=false end end)
    refresh(getter(),false); table.insert(RuntimeState.playerSliderControls,control); return row
end
RuntimeState.createPlayerToggle = function(key,title,y,getter,setter,description)
    local row=make("Frame",{Name=key.."Row",Position=UDim2.fromOffset(0,y),Size=UDim2.new(1,0,0,description and 58 or 42),BackgroundColor3=COLORS.panel2,BackgroundTransparency=0.18,BorderSizePixel=0},RuntimeState.PlayerCard); corner(row,8); stroke(row,COLORS.outline,0.35,1)
    local label=addLabel(row,title,9,COLORS.text,Enum.Font.GothamSemibold); label.Position=UDim2.fromOffset(12,description and 7 or 0); label.Size=UDim2.new(1,-90,0,18)
    if description then local desc=addLabel(row,description,7,COLORS.muted,Enum.Font.Gotham); desc.Position=UDim2.fromOffset(12,27); desc.Size=UDim2.new(1,-100,0,14) end
    local button=make("TextButton",{AnchorPoint=Vector2.new(1,0.5),Position=UDim2.new(1,-12,0.5,0),Size=UDim2.fromOffset(44,23),Text="",AutoButtonColor=false,BackgroundColor3=COLORS.dim,BorderSizePixel=0},row); corner(button,12)
    local knob=make("Frame",{Position=UDim2.fromOffset(3,3),Size=UDim2.fromOffset(17,17),BackgroundColor3=COLORS.white,BorderSizePixel=0},button); corner(knob,9)
    local function refresh() local on=getter()==true; button.BackgroundColor3=on and COLORS.accent or COLORS.dim; knob.Position=on and UDim2.fromOffset(24,3) or UDim2.fromOffset(3,3) end
    button.MouseButton1Click:Connect(function() setter(not (getter()==true)); refresh(); if scheduleConfigSave then scheduleConfigSave() end end); refresh(); return row
end

RuntimeState.createPlayerSlider("WalkSpeed","WALKSPEED",216,10,1000,"",function() return RuntimeState.PlayerTools.walkSpeed end,function(v) RuntimeState.PlayerTools.walkSpeed=v; RuntimeState.applyPlayerMovement() end)
RuntimeState.createPlayerSlider("JumpPower","JUMP POWER",280,10,1000,"",function() return RuntimeState.PlayerTools.jumpPower end,function(v) RuntimeState.PlayerTools.jumpPower=v; RuntimeState.applyPlayerMovement() end)
RuntimeState.createPlayerToggle("Float","FLOAT",344,function() return RuntimeState.PlayerTools.floatEnabled end,function(v) RuntimeState.setFloat(v); end)
RuntimeState.createPlayerToggle("Noclip","NOCLIP",392,function() return RuntimeState.PlayerTools.noclipEnabled end,function(v) RuntimeState.setNoclip(v); end)
RuntimeState.createPlayerSlider("FOV","CAMERA FOV",440,30,120,"°",function() return RuntimeState.PlayerTools.fov end,function(v) RuntimeState.setPlayerFov(v) end)
RuntimeState.createPlayerSlider("VehicleFlySpeed","VEHICLE FLY SPEED",504,10,1000," studs/s",function() return RuntimeState.PlayerTools.vehicleFlySpeed end,function(v) RuntimeState.PlayerTools.vehicleFlySpeed=v end)
RuntimeState.createPlayerToggle("VehicleFly","VEHICLE FLY",568,function() return RuntimeState.PlayerTools.vehicleFly.enabled end,function(v) RuntimeState.setVehicleFly(v) end,"Sit in a VehicleSeat; WASD, Space and Ctrl/Q control flight")
RuntimeState.createPlayerToggle("AimAssist","AIM ASSIST (HOLD RMB)",632,function() return RuntimeState.PlayerTools.aimEnabled end,function(v) RuntimeState.setAimAssist(v) end,"Only tracks while right mouse button is held")
RuntimeState.createPlayerSlider("AimFOV","AIM ASSIST FOV",696,10,1000," px",function() return RuntimeState.PlayerTools.aimFov end,function(v) RuntimeState.PlayerTools.aimFov=v end)
RuntimeState.createPlayerSlider("AimSmoothness","AIM SMOOTHNESS",760,10,1000,"",function() return RuntimeState.PlayerTools.aimSmoothness end,function(v) RuntimeState.PlayerTools.aimSmoothness=v end)
RuntimeState.createPlayerSlider("AimRange","AIM RANGE",824,10,10000," studs",function() return RuntimeState.PlayerTools.aimRange end,function(v) RuntimeState.PlayerTools.aimRange=v end)
RuntimeState.playerCharacterAddedConnection=LocalPlayer.CharacterAdded:Connect(function() task.wait(0.35); RuntimeState.applyPlayerMovement(); if RuntimeState.PlayerTools.noclipEnabled then RuntimeState.setNoclip(true) end end)
RuntimeState.applyPlayerMovement()
RuntimeState.setPlayerFov(RuntimeState.PlayerTools.fov)

make("UIPadding", {
    PaddingTop = UDim.new(0, 14),
    PaddingBottom = UDim.new(0, 12),
    PaddingLeft = UDim.new(0, 16),
    PaddingRight = UDim.new(0, 16),
}, RuntimeState.SettingsCard)

RuntimeState.SettingsTitle = addLabel(RuntimeState.SettingsCard, "Settings", 12, COLORS.text, Enum.Font.GothamBold)
RuntimeState.SettingsTitle.Size = UDim2.new(1, 0, 0, 18)
RuntimeState.SettingsSubtitle = addLabel(RuntimeState.SettingsCard, "Menu configuration", 9, COLORS.muted, Enum.Font.Gotham)
RuntimeState.SettingsSubtitle.Position = UDim2.fromOffset(0, 19)
RuntimeState.SettingsSubtitle.Size = UDim2.new(1, 0, 0, 15)

RuntimeState.KeybindRow = make("Frame", {
    Name = "MenuKeybindRow",
    Position = UDim2.fromOffset(0, 44),
    Size = UDim2.new(1, 0, 0, 86),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.KeybindRow, 8)
stroke(RuntimeState.KeybindRow, COLORS.outline, 0.35, 1)
RuntimeState.KeybindTitle = addLabel(RuntimeState.KeybindRow, "Menu Toggle Key", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.KeybindTitle.Position = UDim2.fromOffset(12, 10)
RuntimeState.KeybindTitle.Size = UDim2.new(1, -160, 0, 17)
RuntimeState.KeybindDescription = addLabel(RuntimeState.KeybindRow, "Choose the keyboard key used to show or hide CATScript.", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.KeybindDescription.Position = UDim2.fromOffset(12, 31)
RuntimeState.KeybindDescription.Size = UDim2.new(1, -160, 0, 15)
RuntimeState.KeybindStatus = addLabel(RuntimeState.KeybindRow, "CLICK THE BUTTON, THEN PRESS A KEY", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.KeybindStatus.Position = UDim2.fromOffset(12, 55)
RuntimeState.KeybindStatus.Size = UDim2.new(1, -160, 0, 13)

RuntimeState.KeybindButton = make("TextButton", {
    Name = "MenuKeybindButton",
    Text = "RightShift",
    TextColor3 = COLORS.text,
    TextSize = 10,
    Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(118, 32),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
}, RuntimeState.KeybindRow)
corner(RuntimeState.KeybindButton, 6)
stroke(RuntimeState.KeybindButton, COLORS.outline, 0, 1)

RuntimeState.AutoSaveRow = make("Frame", {
    Name = "AutoSaveRow",
    Position = UDim2.fromOffset(0, 140),
    Size = UDim2.new(1, 0, 0, 76),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.AutoSaveRow, 8)
stroke(RuntimeState.AutoSaveRow, COLORS.outline, 0.35, 1)
RuntimeState.AutoSaveTitle = addLabel(RuntimeState.AutoSaveRow, "Auto Save", 10, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.AutoSaveTitle.Position = UDim2.fromOffset(12, 10)
RuntimeState.AutoSaveTitle.Size = UDim2.new(1, -160, 0, 17)
RuntimeState.AutoSaveDescription = addLabel(RuntimeState.AutoSaveRow, "Automatically save your menu preferences between sessions.", 8, COLORS.muted, Enum.Font.Gotham)
RuntimeState.AutoSaveDescription.Position = UDim2.fromOffset(12, 31)
RuntimeState.AutoSaveDescription.Size = UDim2.new(1, -160, 0, 15)
RuntimeState.AutoSaveStatus = addLabel(RuntimeState.AutoSaveRow, "", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.AutoSaveStatus.Position = UDim2.fromOffset(12, 53)
RuntimeState.AutoSaveStatus.Size = UDim2.new(1, -160, 0, 13)
RuntimeState.AutoSaveToggleButton = make("TextButton", {
    Name = "AutoSaveToggle",
    Text = "",
    AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(42, 23),
    BackgroundColor3 = autoSaveEnabled and COLORS.accent or COLORS.dim,
    BorderSizePixel = 0,
}, RuntimeState.AutoSaveRow)
corner(RuntimeState.AutoSaveToggleButton, 12)
RuntimeState.AutoSaveToggleKnob = make("Frame", {
    Name = "Knob",
    Position = autoSaveEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    Size = UDim2.fromOffset(17, 17),
    BackgroundColor3 = COLORS.white,
    BorderSizePixel = 0,
}, RuntimeState.AutoSaveToggleButton)
corner(RuntimeState.AutoSaveToggleKnob, 9)

RuntimeState.MetricsHeader = addLabel(RuntimeState.SettingsCard, "PERFORMANCE INDICATORS", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.MetricsHeader.Position = UDim2.fromOffset(0, 226)
RuntimeState.MetricsHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.MetricsHint = addLabel(RuntimeState.SettingsCard, "Choose which live values appear in the on-screen monitor.", 8, COLORS.dim, Enum.Font.Gotham)
RuntimeState.MetricsHint.Position = UDim2.fromOffset(0, 243)
RuntimeState.MetricsHint.Size = UDim2.new(1, 0, 0, 14)

RuntimeState.MetricToggleControls = {}
RuntimeState.MetricRows = {}
local function createMetricToggle(key, title, description, y)
    local row = make("Frame", {
        Name = "Metric_" .. key,
        Position = UDim2.fromOffset(0, y),
        Size = UDim2.new(1, 0, 0, 46),
        BackgroundColor3 = COLORS.panel2,
        BorderSizePixel = 0,
    }, RuntimeState.SettingsCard)
    RuntimeState.MetricRows[key] = row
    corner(row, 7)
    stroke(row, COLORS.outline, 0.4, 1)

    local titleLabel = addLabel(row, title, 9, COLORS.text, Enum.Font.GothamSemibold)
    titleLabel.Position = UDim2.fromOffset(11, 5)
    titleLabel.Size = UDim2.new(1, -100, 0, 16)
    local descLabel = addLabel(row, description, 7, COLORS.muted, Enum.Font.Gotham)
    descLabel.Position = UDim2.fromOffset(11, 23)
    descLabel.Size = UDim2.new(1, -100, 0, 13)

    local button = make("TextButton", {
        Name = "Toggle_" .. key,
        Text = "",
        AutoButtonColor = false,
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -11, 0.5, 0),
        Size = UDim2.fromOffset(40, 22),
        BackgroundColor3 = metricSettings[key] and COLORS.accent or COLORS.dim,
        BorderSizePixel = 0,
    }, row)
    corner(button, 12)
    local knob = make("Frame", {
        Name = "Knob",
        Position = metricSettings[key] and UDim2.fromOffset(20, 3) or UDim2.fromOffset(3, 3),
        Size = UDim2.fromOffset(16, 16),
        BackgroundColor3 = COLORS.white,
        BorderSizePixel = 0,
    }, button)
    corner(knob, 9)

    RuntimeState.MetricToggleControls[key] = {button = button, knob = knob}
end

createMetricToggle("fps", "FPS", "Current client frames per second.", 264)
createMetricToggle("ping", "Ping", "Network latency to the game server.", 314)
createMetricToggle("gpu", "GPU frame time", "GPU render time per frame, in milliseconds.", 364)
createMetricToggle("memory", "Memory", "Current Roblox client memory usage.", 414)

RuntimeState.BackgroundSettingsHeader = addLabel(RuntimeState.SettingsCard, "MENU BACKGROUND", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.BackgroundSettingsHeader.Position = UDim2.fromOffset(0, 475)
RuntimeState.BackgroundSettingsHeader.Size = UDim2.new(1, 0, 0, 16)

RuntimeState.BackgroundSettingsRow = make("Frame", {
    Name = "BackgroundSettingsRow",
    Position = UDim2.fromOffset(0, 496),
    Size = UDim2.new(1, 0, 0, 108),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.BackgroundSettingsRow, 8)
stroke(RuntimeState.BackgroundSettingsRow, COLORS.outline, 0.35, 1)

RuntimeState.BackgroundSettingsTitle = addLabel(RuntimeState.BackgroundSettingsRow, "Full Menu Background Image", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.BackgroundSettingsTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.BackgroundSettingsTitle.Size = UDim2.new(1, -24, 0, 16)

RuntimeState.BackgroundSettingsDescription = addLabel(RuntimeState.BackgroundSettingsRow, "Paste a Roblox image asset ID to change the entire menu background.", 7, COLORS.muted, Enum.Font.Gotham)
RuntimeState.BackgroundSettingsDescription.Position = UDim2.fromOffset(12, 25)
RuntimeState.BackgroundSettingsDescription.Size = UDim2.new(1, -24, 0, 13)

RuntimeState.BackgroundIdBox = make("TextBox", {
    Name = "BackgroundImageIdInput",
    Position = UDim2.fromOffset(12, 47),
    Size = UDim2.new(1, -198, 0, 28),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
    Text = backgroundAssetId,
    TextColor3 = COLORS.text,
    PlaceholderText = "Image asset ID (numbers only)",
    PlaceholderColor3 = COLORS.dim,
    TextSize = 9,
    Font = Enum.Font.Gotham,
    ClearTextOnFocus = false,
}, RuntimeState.BackgroundSettingsRow)
corner(RuntimeState.BackgroundIdBox, 5)
stroke(RuntimeState.BackgroundIdBox, COLORS.outline, 0, 1)
make("UIPadding", {PaddingLeft = UDim.new(0, 9), PaddingRight = UDim.new(0, 7)}, RuntimeState.BackgroundIdBox)

RuntimeState.ApplyBackgroundButton = make("TextButton", {
    Name = "ApplyBackgroundButton",
    Position = UDim2.new(1, -176, 0, 47),
    Size = UDim2.fromOffset(78, 28),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    Text = "Apply",
    TextColor3 = COLORS.white,
    TextSize = 9,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
}, RuntimeState.BackgroundSettingsRow)
corner(RuntimeState.ApplyBackgroundButton, 5)

RuntimeState.ClearBackgroundButton = make("TextButton", {
    Name = "ClearBackgroundButton",
    Position = UDim2.new(1, -90, 0, 47),
    Size = UDim2.fromOffset(78, 28),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
    Text = "Remove",
    TextColor3 = COLORS.muted,
    TextSize = 9,
    Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false,
}, RuntimeState.BackgroundSettingsRow)
corner(RuntimeState.ClearBackgroundButton, 5)
stroke(RuntimeState.ClearBackgroundButton, COLORS.outline, 0, 1)


RuntimeState.BackgroundSettingsStatus = addLabel(RuntimeState.BackgroundSettingsRow, backgroundAssetId ~= "" and "BACKGROUND IMAGE ID LOADED FROM CONFIG" or "ENTER AN IMAGE ASSET ID", 7, COLORS.dim, Enum.Font.GothamBold)
RuntimeState.BackgroundSettingsStatus.Position = UDim2.fromOffset(12, 81)
RuntimeState.BackgroundSettingsStatus.Size = UDim2.new(1, -24, 0, 13)

local function normalizeBackgroundId(value)
    local cleaned = tostring(value or ""):match("^%s*(.-)%s*$") or ""
    cleaned = cleaned:gsub("^rbxassetid://", "")
    if cleaned == "" then return "" end
    if not cleaned:match("^%d+$") then return nil end
    return cleaned
end

local function applyBackgroundId(value)
    local normalized = normalizeBackgroundId(value)
    if normalized == nil then
        RuntimeState.BackgroundSettingsStatus.Text = "INVALID ID  /  USE NUMBERS ONLY"
        RuntimeState.BackgroundSettingsStatus.TextColor3 = Color3.fromRGB(230, 120, 120)
        return false
    end

    backgroundAssetId = normalized
    RuntimeState.BackgroundIdBox.Text = backgroundAssetId
    if backgroundAssetId == "" then
        BackgroundImage.Image = ""
        BackgroundImage.Visible = false
        BackgroundTint.Visible = false
        BackgroundTint.BackgroundTransparency = 1
        RuntimeState.BackgroundSettingsStatus.Text = "MENU BACKGROUND RESET"
    else
        BackgroundImage.Image = "rbxassetid://" .. backgroundAssetId
        BackgroundImage.Visible = true
        BackgroundTint.Visible = true
        BackgroundTint.BackgroundTransparency = 0.58
        RuntimeState.BackgroundSettingsStatus.Text = "FULL MENU BACKGROUND APPLIED"
    end
    RuntimeState.BackgroundSettingsStatus.TextColor3 = COLORS.accent
    if scheduleConfigSave then scheduleConfigSave() end
    return true
end

RuntimeState.ApplyBackgroundButton.MouseButton1Click:Connect(function()
    applyBackgroundId(RuntimeState.BackgroundIdBox.Text)
end)
RuntimeState.BackgroundIdBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then applyBackgroundId(RuntimeState.BackgroundIdBox.Text) end
end)
RuntimeState.ClearBackgroundButton.MouseButton1Click:Connect(function()
    applyBackgroundId("")
end)

-- THEME STUDIO ---------------------------------------------------------------
RuntimeState.ThemeHeader = addLabel(RuntimeState.SettingsCard, "РЕДАКТОР ТЕМЫ", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.ThemeHeader.Position = UDim2.fromOffset(0, 620)
RuntimeState.ThemeHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.ThemeHint = addLabel(RuntimeState.SettingsCard, "Выбери любой элемент и настрой его цвет через HSV, HEX или RGB.", 8, COLORS.dim, Enum.Font.Gotham)
RuntimeState.ThemeHint.Position = UDim2.fromOffset(0, 638)
RuntimeState.ThemeHint.Size = UDim2.new(1, 0, 0, 14)

RuntimeState.ThemeButtons = {}
RuntimeState.ThemeRoleLabels = {
    accent = "Акцентный цвет",
    background = "Главный фон",
    sidebar = "Боковая панель",
    topbar = "Верхняя панель",
    panel = "Карточки",
    panel2 = "Внутренние карточки",
    outline = "Обводка и разделители",
    text = "Основной текст",
    muted = "Вторичный текст",
    dim = "Приглушённый текст",
    white = "Белый / яркие элементы",
}
for themeIndex, themeKey in ipairs(RuntimeState.THEME_KEYS) do
    local themeColumn = (themeIndex - 1) % 2
    local themeRow = math.floor((themeIndex - 1) / 2)
    local themeButton = make("TextButton", {
        Name = "ThemeColor_" .. themeKey,
        Position = UDim2.new(themeColumn * 0.5, themeColumn == 0 and 0 or 5, 0, 660 + themeRow * 38),
        Size = UDim2.new(0.5, -5, 0, 32),
        BackgroundColor3 = COLORS.panel2,
        BorderSizePixel = 0,
        AutoButtonColor = false,
        Text = RuntimeState.ThemeRoleLabels[themeKey],
        TextColor3 = COLORS.text,
        TextSize = 8,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.GothamSemibold,
    }, RuntimeState.SettingsCard)
    corner(themeButton, 6)
    stroke(themeButton, COLORS.outline, 0.25, 1)
    make("UIPadding", {PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 36)}, themeButton)
    local themeSwatch = make("Frame", {
        Name = "Swatch",
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -8, 0.5, 0),
        Size = UDim2.fromOffset(22, 18),
        BackgroundColor3 = COLORS[themeKey],
        BorderSizePixel = 0,
    }, themeButton)
    corner(themeSwatch, 5)
    stroke(themeSwatch, COLORS.white, 0.68, 1)
    RuntimeState.ThemeButtons[themeKey] = {button = themeButton, swatch = themeSwatch}
end

RuntimeState.ThemeTransparencyRow = make("Frame", {
    Name = "MenuTransparencySettings",
    Position = UDim2.fromOffset(0, 896),
    Size = UDim2.new(1, 0, 0, 72),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.ThemeTransparencyRow, 8)
stroke(RuntimeState.ThemeTransparencyRow, COLORS.outline, 0.3, 1)
RuntimeState.ThemeTransparencyTitle = addLabel(RuntimeState.ThemeTransparencyRow, "Прозрачность меню", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.ThemeTransparencyTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.ThemeTransparencyTitle.Size = UDim2.new(1, -95, 0, 16)
RuntimeState.ThemeTransparencyValue = addLabel(RuntimeState.ThemeTransparencyRow, "0%", 9, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
RuntimeState.ThemeTransparencyValue.Position = UDim2.new(1, -78, 0, 8)
RuntimeState.ThemeTransparencyValue.Size = UDim2.fromOffset(64, 16)
RuntimeState.ThemeTransparencyHint = addLabel(RuntimeState.ThemeTransparencyRow, "От более плотного меню до лёгкого полупрозрачного стекла.", 7, COLORS.muted, Enum.Font.Gotham)
RuntimeState.ThemeTransparencyHint.Position = UDim2.fromOffset(12, 25)
RuntimeState.ThemeTransparencyHint.Size = UDim2.new(1, -24, 0, 13)
RuntimeState.ThemeTransparencyTrack = make("TextButton", {
    Name = "MenuTransparencyTrack",
    Position = UDim2.fromOffset(12, 52),
    Size = UDim2.new(1, -24, 0, 6),
    BackgroundColor3 = COLORS.outline,
    BorderSizePixel = 0,
    Text = "",
    AutoButtonColor = false,
}, RuntimeState.ThemeTransparencyRow)
corner(RuntimeState.ThemeTransparencyTrack, 5)
RuntimeState.ThemeTransparencyFill = make("Frame", {
    Size = UDim2.new(RuntimeState.menuTransparency / 0.70, 0, 1, 0),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
}, RuntimeState.ThemeTransparencyTrack)
corner(RuntimeState.ThemeTransparencyFill, 5)
RuntimeState.ThemeTransparencyKnob = make("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.new(RuntimeState.menuTransparency / 0.70, 0, 0.5, 0),
    Size = UDim2.fromOffset(12, 12),
    BackgroundColor3 = COLORS.white,
    BorderSizePixel = 0,
}, RuntimeState.ThemeTransparencyTrack)
corner(RuntimeState.ThemeTransparencyKnob, 7)

RuntimeState.ResetThemeButton = make("TextButton", {
    Name = "ResetThemeButton",
    Position = UDim2.fromOffset(0, 978),
    Size = UDim2.new(1, 0, 0, 32),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
    Text = "↺   Сбросить тему и прозрачность",
    TextColor3 = COLORS.text,
    TextSize = 8,
    Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false,
}, RuntimeState.SettingsCard)
corner(RuntimeState.ResetThemeButton, 7)
stroke(RuntimeState.ResetThemeButton, COLORS.outline, 0.25, 1)

-- FONT STUDIO ---------------------------------------------------------------
RuntimeState.FontHeader = addLabel(RuntimeState.SettingsCard, "РЕДАКТОР ШРИФТА", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.FontHeader.Position = UDim2.fromOffset(0, 1022)
RuntimeState.FontHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.FontHint = addLabel(RuntimeState.SettingsCard, "Выбери шрифт — он сразу применится ко всему интерфейсу CATScript.", 8, COLORS.dim, Enum.Font.Gotham)
RuntimeState.FontHint.Position = UDim2.fromOffset(0, 1040)
RuntimeState.FontHint.Size = UDim2.new(1, 0, 0, 14)
RuntimeState.FontSettingsRow = make("Frame", {
    Name = "FontSettingsRow",
    Position = UDim2.fromOffset(0, 1060),
    Size = UDim2.new(1, 0, 0, 82),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.FontSettingsRow, 8)
stroke(RuntimeState.FontSettingsRow, COLORS.outline, 0.3, 1)
RuntimeState.FontSettingsTitle = addLabel(RuntimeState.FontSettingsRow, "Текущий шрифт", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.FontSettingsTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.FontSettingsTitle.Size = UDim2.new(1, -200, 0, 15)
RuntimeState.FontPreviewSample = addLabel(RuntimeState.FontSettingsRow, "CATScript  •  Aa Bb 123", 12, COLORS.accent, RuntimeState.selectedFont)
RuntimeState.FontPreviewSample.Position = UDim2.fromOffset(12, 29)
RuntimeState.FontPreviewSample.Size = UDim2.new(1, -260, 0, 28)
RuntimeState.FontCurrentButton = make("TextButton", {
    Name = "FontCurrentButton",
    AnchorPoint = Vector2.new(1, 0),
    Position = UDim2.new(1, -88, 0, 25),
    Size = UDim2.fromOffset(142, 32),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    Text = RuntimeState.selectedFontName .. "  ▾",
    TextColor3 = COLORS.white,
    TextSize = 9,
    Font = RuntimeState.selectedFont,
    AutoButtonColor = false,
}, RuntimeState.FontSettingsRow)
corner(RuntimeState.FontCurrentButton, 6)
RuntimeState.FontResetButton = make("TextButton", {
    Name = "FontResetButton",
    AnchorPoint = Vector2.new(1, 0),
    Position = UDim2.new(1, -10, 0, 25),
    Size = UDim2.fromOffset(68, 32),
    BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0,
    Text = "Сброс",
    TextColor3 = COLORS.muted,
    TextSize = 8,
    Font = Enum.Font.GothamSemibold,
    AutoButtonColor = false,
}, RuntimeState.FontSettingsRow)
corner(RuntimeState.FontResetButton, 6)

-- CUSTOM CURSOR SETTINGS -----------------------------------------------------
RuntimeState.CursorHeader = addLabel(RuntimeState.SettingsCard, "КАСТОМНЫЙ КУРСОР", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CursorHeader.Position = UDim2.fromOffset(0, 1152)
RuntimeState.CursorHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.CursorHint = addLabel(RuntimeState.SettingsCard, "Выбери форму, цвет и размер курсора. Он не перекрывает клики по интерфейсу.", 8, COLORS.dim, Enum.Font.Gotham)
RuntimeState.CursorHint.Position = UDim2.fromOffset(0, 1170)
RuntimeState.CursorHint.Size = UDim2.new(1, 0, 0, 14)
RuntimeState.CursorSettingsRow = make("Frame", {
    Name = "CustomCursorSettingsRow",
    Position = UDim2.fromOffset(0, 1190),
    Size = UDim2.new(1, 0, 0, 184),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.CursorSettingsRow, 8)
stroke(RuntimeState.CursorSettingsRow, COLORS.outline, 0.3, 1)
RuntimeState.CursorSettingsTitle = addLabel(RuntimeState.CursorSettingsRow, "Кастомный курсор", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.CursorSettingsTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.CursorSettingsTitle.Size = UDim2.new(1, -160, 0, 16)
RuntimeState.CursorSettingsDescription = addLabel(RuntimeState.CursorSettingsRow, "Включи, чтобы заменить стандартный курсор Roblox.", 7, COLORS.muted, Enum.Font.Gotham)
RuntimeState.CursorSettingsDescription.Position = UDim2.fromOffset(12, 25)
RuntimeState.CursorSettingsDescription.Size = UDim2.new(1, -160, 0, 13)
RuntimeState.CursorToggleButton = make("TextButton", {
    Name = "CustomCursorToggle",
    Text = "",
    AutoButtonColor = false,
    AnchorPoint = Vector2.new(1, 0),
    Position = UDim2.new(1, -12, 0, 11),
    Size = UDim2.fromOffset(42, 23),
    BackgroundColor3 = cursorSettings.enabled and COLORS.accent or COLORS.dim,
    BorderSizePixel = 0,
}, RuntimeState.CursorSettingsRow)
corner(RuntimeState.CursorToggleButton, 12)
RuntimeState.CursorToggleKnob = make("Frame", {
    Name = "Knob",
    Position = cursorSettings.enabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    Size = UDim2.fromOffset(17, 17),
    BackgroundColor3 = COLORS.white,
    BorderSizePixel = 0,
}, RuntimeState.CursorToggleButton)
corner(RuntimeState.CursorToggleKnob, 9)
RuntimeState.CursorStatus = addLabel(RuntimeState.CursorSettingsRow, cursorSettings.enabled and "ВКЛЮЧЁН" or "ВЫКЛЮЧЕН", 7, cursorSettings.enabled and COLORS.accent or COLORS.dim, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
RuntimeState.CursorStatus.AnchorPoint = Vector2.new(1, 0)
RuntimeState.CursorStatus.Position = UDim2.new(1, -12, 0, 37)
RuntimeState.CursorStatus.Size = UDim2.fromOffset(100, 12)

RuntimeState.CursorStyleLabel = addLabel(RuntimeState.CursorSettingsRow, "ФОРМА · 15 ШТ.", 7, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CursorStyleLabel.Position = UDim2.fromOffset(12, 47)
RuntimeState.CursorStyleLabel.Size = UDim2.fromOffset(54, 48)
RuntimeState.CursorStyleLabel.TextWrapped = true
RuntimeState.CursorStyleButtons = {}
for styleIndex, styleName in ipairs(cursorStyles) do
    local styleColumn = (styleIndex - 1) % 5
    local styleRow = math.floor((styleIndex - 1) / 5)
    local button = make("TextButton", {
        Name = "CursorStyle_" .. styleName,
        Position = UDim2.fromOffset(70 + styleColumn * 78, 47 + styleRow * 28),
        Size = UDim2.fromOffset(72, 24),
        BackgroundColor3 = cursorSettings.style == styleName and COLORS.accent or COLORS.panel,
        BorderSizePixel = 0,
        Text = cursorStyleLabels[styleName] or styleName,
        TextColor3 = cursorSettings.style == styleName and COLORS.white or COLORS.text,
        TextSize = 7,
        Font = Enum.Font.GothamSemibold,
        AutoButtonColor = false,
    }, RuntimeState.CursorSettingsRow)
    corner(button, 5)
    stroke(button, COLORS.outline, 0.4, 1)
    RuntimeState.CursorStyleButtons[styleName] = button
end

RuntimeState.CursorColorLabel = addLabel(RuntimeState.CursorSettingsRow, "ЦВЕТ", 7, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CursorColorLabel.Position = UDim2.fromOffset(12, 145)
RuntimeState.CursorColorLabel.Size = UDim2.fromOffset(54, 24)
RuntimeState.CursorColorButtons = {}
local cursorColorIndex = 0
local cursorColorNames = {"Purple", "White", "Cyan", "Pink", "Lime"}
for _, colorName in ipairs(cursorColorNames) do
    local colorValue = cursorPalette[colorName]
    cursorColorIndex += 1
    local button = make("TextButton", {
        Name = "CursorColor_" .. colorName,
        Position = UDim2.fromOffset(70 + (cursorColorIndex - 1) * 36, 145),
        Size = UDim2.fromOffset(25, 25),
        BackgroundColor3 = colorValue,
        BorderSizePixel = 0,
        Text = "",
        AutoButtonColor = false,
    }, RuntimeState.CursorSettingsRow)
    corner(button, 13)
    local outline = stroke(button, cursorSettings.colorName == colorName and COLORS.white or COLORS.outline, cursorSettings.colorName == colorName and 0 or 0.35, cursorSettings.colorName == colorName and 2 or 1)
    RuntimeState.CursorColorButtons[colorName] = {button = button, outline = outline}
end
RuntimeState.CursorColorPickerButton = make("TextButton", {
    Name = "CursorOpenFullColorPicker",
    Position = UDim2.fromOffset(244, 145),
    Size = UDim2.fromOffset(40, 25),
    BackgroundColor3 = cursorSettings.color,
    BorderSizePixel = 0,
    Text = "RGB",
    TextColor3 = Color3.new(1, 1, 1),
    TextSize = 8,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
}, RuntimeState.CursorSettingsRow)
corner(RuntimeState.CursorColorPickerButton, 5)
RuntimeState.CursorColorPickerButtonOutline = stroke(RuntimeState.CursorColorPickerButton, COLORS.white, 0, 2)
RuntimeState.CursorSizeLabel = addLabel(RuntimeState.CursorSettingsRow, "РАЗМЕР", 7, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CursorSizeLabel.Position = UDim2.fromOffset(290, 145)
RuntimeState.CursorSizeLabel.Size = UDim2.fromOffset(52, 24)
RuntimeState.CursorSizeMinus = make("TextButton", {
    Name = "CursorSizeMinus", Position = UDim2.fromOffset(344, 145), Size = UDim2.fromOffset(25, 25),
    BackgroundColor3 = COLORS.panel, BorderSizePixel = 0, Text = "−", TextColor3 = COLORS.text,
    TextSize = 12, Font = Enum.Font.GothamBold, AutoButtonColor = false,
}, RuntimeState.CursorSettingsRow)
corner(RuntimeState.CursorSizeMinus, 5)
RuntimeState.CursorSizeValue = addLabel(RuntimeState.CursorSettingsRow, tostring(cursorSettings.size) .. " px", 8, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
RuntimeState.CursorSizeValue.Position = UDim2.fromOffset(372, 145)
RuntimeState.CursorSizeValue.Size = UDim2.fromOffset(48, 25)
RuntimeState.CursorSizePlus = make("TextButton", {
    Name = "CursorSizePlus", Position = UDim2.fromOffset(423, 145), Size = UDim2.fromOffset(25, 25),
    BackgroundColor3 = COLORS.panel, BorderSizePixel = 0, Text = "+", TextColor3 = COLORS.text,
    TextSize = 12, Font = Enum.Font.GothamBold, AutoButtonColor = false,
}, RuntimeState.CursorSettingsRow)
corner(RuntimeState.CursorSizePlus, 5)

-- Anti-AFK, Freecam, and No Texture utility switches.
RuntimeState.UtilityHeader = addLabel(RuntimeState.SettingsCard, "GAME UTILITIES", 9, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.UtilityHeader.Position = UDim2.fromOffset(0, 1390)
RuntimeState.UtilityHeader.Size = UDim2.new(1, 0, 0, 16)
RuntimeState.UtilityHint = addLabel(RuntimeState.SettingsCard, "Local client tools. Freecam controls: WASD + E/Q, hold RMB to look, Shift to move faster.", 8, COLORS.dim, Enum.Font.Gotham)
RuntimeState.UtilityHint.Position = UDim2.fromOffset(0, 1408)
RuntimeState.UtilityHint.Size = UDim2.new(1, 0, 0, 15)

RuntimeState.createUtilityToggleRow = function(slotName, titleText, descriptionText, yPosition, initialValue, callback)
    local row = make("Frame", {
        Name = slotName .. "Row",
        Position = UDim2.fromOffset(0, yPosition),
        Size = UDim2.new(1, 0, 0, 72),
        BackgroundColor3 = COLORS.panel2,
        BackgroundTransparency = 0.18,
        BorderSizePixel = 0,
    }, RuntimeState.SettingsCard)
    corner(row, 8)
    stroke(row, COLORS.outline, 0.35, 1)
    local title = addLabel(row, titleText, 10, COLORS.text, Enum.Font.GothamSemibold)
    title.Position = UDim2.fromOffset(12, 9)
    title.Size = UDim2.new(1, -145, 0, 17)
    local description = addLabel(row, descriptionText, 8, COLORS.muted, Enum.Font.Gotham)
    description.Position = UDim2.fromOffset(12, 29)
    description.Size = UDim2.new(1, -145, 0, 14)
    local status = addLabel(row, initialValue and "ENABLED" or "DISABLED", 7, initialValue and COLORS.accent or COLORS.dim, Enum.Font.GothamBold)
    status.Position = UDim2.fromOffset(12, 49)
    status.Size = UDim2.new(1, -145, 0, 12)
    local button = make("TextButton", {
        Name = slotName .. "Toggle",
        Text = "",
        AutoButtonColor = false,
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -12, 0.5, 0),
        Size = UDim2.fromOffset(42, 23),
        BackgroundColor3 = initialValue and COLORS.accent or COLORS.dim,
        BorderSizePixel = 0,
    }, row)
    corner(button, 12)
    local knob = make("Frame", {
        Name = "Knob",
        Position = initialValue and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
        Size = UDim2.fromOffset(17, 17),
        BackgroundColor3 = COLORS.white,
        BorderSizePixel = 0,
    }, button)
    corner(knob, 9)
    local value = initialValue == true
    local update = function(nextValue)
        value = nextValue == true
        tween(button, {BackgroundColor3 = value and COLORS.accent or COLORS.dim}, 0.12)
        tween(knob, {Position = value and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3)}, 0.12)
        status.Text = value and "ENABLED" or "DISABLED"
        status.TextColor3 = value and COLORS.accent or COLORS.dim
    end
    local setStatus = function(textValue)
        status.Text = textValue
        status.TextColor3 = string.find(string.upper(textValue), "UNAVAILABLE", 1, true) and Color3.fromRGB(230, 165, 90) or (value and COLORS.accent or COLORS.dim)
    end
    button.MouseButton1Click:Connect(function()
        update(not value)
        local message = callback(value)
        if type(message) == "string" then setStatus(message) end
    end)
    return {row = row, button = button, status = status, update = update, setStatus = setStatus}
end

RuntimeState.AntiAfkRow = RuntimeState.createUtilityToggleRow(
    "AntiAfk", "Anti-AFK", "Prevents idle timeout using the local Idled event.", 1430,
    RuntimeState.Utility.antiAfkEnabled,
    function(enabled)
        local statusText = RuntimeState.setAntiAfk(enabled)
        if scheduleConfigSave then scheduleConfigSave() end
        return statusText
    end
)
RuntimeState.FreecamRow = RuntimeState.createUtilityToggleRow(
    "Freecam", "Freecam", "Fly the camera independently without moving your character.", 1510,
    false,
    function(enabled)
        local ok = RuntimeState.setFreecam(enabled)
        if not ok and RuntimeState.FreecamRow then
            RuntimeState.FreecamRow.update(false)
        end
        return ok and (enabled and "WASD MOVE / RMB LOOK / SHIFT BOOST" or "CAMERA RESTORED") or "CAMERA UNAVAILABLE"
    end
)

-- Freecam movement speed slider (10-1000 studs/second). Changes are applied
-- immediately because the RenderStepped movement loop reads freecam.speed.
RuntimeState.FreecamSpeedRow = make("Frame", {
    Name = "FreecamSpeedRow",
    Position = UDim2.fromOffset(0, 1590),
    Size = UDim2.new(1, 0, 0, 72),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.SettingsCard)
corner(RuntimeState.FreecamSpeedRow, 8)
stroke(RuntimeState.FreecamSpeedRow, COLORS.outline, 0.35, 1)
RuntimeState.FreecamSpeedTitle = addLabel(RuntimeState.FreecamSpeedRow, "FREECAM SPEED", 9, COLORS.text, Enum.Font.GothamSemibold)
RuntimeState.FreecamSpeedTitle.Position = UDim2.fromOffset(12, 8)
RuntimeState.FreecamSpeedTitle.Size = UDim2.new(1, -120, 0, 16)
RuntimeState.FreecamSpeedValue = addLabel(RuntimeState.FreecamSpeedRow, tostring(RuntimeState.Utility.freecam.speed) .. " studs/s", 8, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
RuntimeState.FreecamSpeedValue.Position = UDim2.new(1, -118, 0, 8)
RuntimeState.FreecamSpeedValue.Size = UDim2.fromOffset(106, 16)
RuntimeState.FreecamSpeedTrack = make("Frame", {
    Name = "FreecamSpeedTrack", Position = UDim2.fromOffset(13, 39),
    Size = UDim2.new(1, -26, 0, 5), BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0, ZIndex = 2,
}, RuntimeState.FreecamSpeedRow)
corner(RuntimeState.FreecamSpeedTrack, 3)
RuntimeState.FreecamSpeedFill = make("Frame", {
    Name = "FreecamSpeedFill",
    Size = UDim2.new((RuntimeState.Utility.freecam.speed - 10) / 990, 0, 1, 0),
    BackgroundColor3 = COLORS.accent, BorderSizePixel = 0, ZIndex = 3,
}, RuntimeState.FreecamSpeedTrack)
corner(RuntimeState.FreecamSpeedFill, 3)
RuntimeState.FreecamSpeedKnob = make("Frame", {
    Name = "FreecamSpeedKnob", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.new((RuntimeState.Utility.freecam.speed - 10) / 990, 0, 0.5, 0),
    Size = UDim2.fromOffset(11, 11), BackgroundColor3 = COLORS.white,
    BorderSizePixel = 0, ZIndex = 5,
}, RuntimeState.FreecamSpeedTrack)
corner(RuntimeState.FreecamSpeedKnob, 6)
stroke(RuntimeState.FreecamSpeedKnob, COLORS.accent, 0, 1)
RuntimeState.FreecamSpeedMin = addLabel(RuntimeState.FreecamSpeedRow, "10", 7, COLORS.dim, Enum.Font.Gotham)
RuntimeState.FreecamSpeedMin.Position = UDim2.fromOffset(13, 49)
RuntimeState.FreecamSpeedMin.Size = UDim2.fromOffset(36, 12)
RuntimeState.FreecamSpeedMax = addLabel(RuntimeState.FreecamSpeedRow, "1000", 7, COLORS.dim, Enum.Font.Gotham, Enum.TextXAlignment.Right)
RuntimeState.FreecamSpeedMax.Position = UDim2.new(1, -49, 0, 49)
RuntimeState.FreecamSpeedMax.Size = UDim2.fromOffset(36, 12)
RuntimeState.FreecamSpeedInput = make("TextButton", {
    Name = "FreecamSpeedInput", Position = UDim2.fromOffset(8, 30),
    Size = UDim2.new(1, -16, 0, 20), Text = "", AutoButtonColor = false,
    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 8,
}, RuntimeState.FreecamSpeedRow)
RuntimeState.setFreecamSpeedFromX = function(pointerX)
    local track = RuntimeState.FreecamSpeedTrack
    local width = math.max(track.AbsoluteSize.X, 1)
    local alpha = math.clamp((pointerX - track.AbsolutePosition.X) / width, 0, 1)
    local speed = math.clamp(math.floor(10 + alpha * 990 + 0.5), 10, 1000)
    local normalized = (speed - 10) / 990
    RuntimeState.Utility.freecam.speed = speed
    RuntimeState.FreecamSpeedFill.Size = UDim2.new(normalized, 0, 1, 0)
    RuntimeState.FreecamSpeedKnob.Position = UDim2.new(normalized, 0, 0.5, 0)
    RuntimeState.FreecamSpeedValue.Text = tostring(speed) .. " studs/s"
    if scheduleConfigSave then scheduleConfigSave() end
end
RuntimeState.FreecamSpeedInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.freecamSpeedDragging = true
        RuntimeState.setFreecamSpeedFromX(input.Position.X)
    end
end)
RuntimeState.freecamSpeedInputChangedConnection = UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.freecamSpeedDragging then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        RuntimeState.setFreecamSpeedFromX(UserInputService:GetMouseLocation().X)
    elseif input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.setFreecamSpeedFromX(input.Position.X)
    end
end)
RuntimeState.freecamSpeedInputEndedConnection = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.freecamSpeedDragging = false
    end
end)

RuntimeState.NoTextureRow = RuntimeState.createUtilityToggleRow(
    "NoTexture", "No Texture / Flat Materials", "Hides map textures and replaces part materials with SmoothPlastic.", 1670,
    RuntimeState.Utility.noTextureEnabled,
    function(enabled)
        local statusText = RuntimeState.setNoTexture(enabled)
        if scheduleConfigSave then scheduleConfigSave() end
        return statusText
    end
)
RuntimeState.NoGuiRow = RuntimeState.createUtilityToggleRow(
    "NoGui", "No GUI / Hide Game Interface", "Hides game HUD, PlayerGui and world labels. CATScript stays visible.", 1750,
    RuntimeState.Utility.noGuiEnabled,
    function(enabled)
        local statusText = RuntimeState.setNoGui(enabled)
        if scheduleConfigSave then scheduleConfigSave() end
        return statusText
    end
)
RuntimeState.KeystrokesRow = RuntimeState.createUtilityToggleRow(
    "Keystrokes", "Keystrokes  /  WASD + Mouse", "Shows pressed WASD keys and left/right mouse buttons in a small HUD.", 1830,
    RuntimeState.Keystrokes.enabled,
    function(enabled)
        RuntimeState.setKeystrokes(enabled)
        if scheduleConfigSave then scheduleConfigSave() end
        return enabled and "KEYSTROKES HUD ACTIVE" or "KEYSTROKES HUD HIDDEN"
    end
)

-- War Tycoon Session Rewards helper. It only looks for visible Claim/Collect
-- buttons inside a Session/Playtime reward panel and activates the UI signal.
-- It deliberately does not invoke game RemoteEvents or guess remote names.
local sessionRewardLoopVersion = 0
local sessionRewardLastClickAt = 0
local sessionRewardLastButtonClick = setmetatable({}, {__mode = "k"})

local function sessionRewardIsVisible(button, playerGui)
    local node = button
    while node and node ~= playerGui do
        if node:IsA("GuiObject") and not node.Visible then return false end
        if node:IsA("ScreenGui") and not node.Enabled then return false end
        node = node.Parent
    end
    return true
end

local function collectSessionRewardScopes(playerGui)
    local scopes, seen = {}, {}
    local function addScope(scope)
        if scope and scope:IsA("GuiObject") and not seen[scope] then
            seen[scope] = true
            table.insert(scopes, scope)
        end
    end
    for _, object in ipairs(playerGui:GetDescendants()) do
        if object:IsA("GuiObject") then
            local objectName = string.lower(object.Name or "")
            if objectName:find("session", 1, true) or objectName:find("playtime", 1, true)
                or objectName:find("time_reward", 1, true) or objectName:find("timereward", 1, true) then
                addScope(object)
            end
            if object:IsA("TextLabel") or object:IsA("TextButton") then
                local labelText = string.lower(object.Text or "")
                if labelText:find("session reward", 1, true) or labelText:find("playtime reward", 1, true)
                    or labelText:find("session rewards", 1, true) or labelText:find("playtime rewards", 1, true) then
                    local scope = object.Parent
                    for _ = 1, 2 do
                        addScope(scope)
                        scope = scope and scope.Parent
                    end
                end
            end
        end
    end
    return scopes
end

local function isInsideSessionRewardScope(button, playerGui, scopes)
    local node = button
    while node and node ~= playerGui do
        local nodeName = string.lower(node.Name or "")
        if nodeName:find("session", 1, true) or nodeName:find("playtime", 1, true)
            or nodeName:find("time_reward", 1, true) or nodeName:find("timereward", 1, true) then
            return true
        end
        node = node.Parent
    end
    for _, scope in ipairs(scopes) do
        local ok, inside = pcall(function() return button:IsDescendantOf(scope) end)
        if ok and inside then return true end
    end
    return false
end

local function getSessionRewardButtonText(button)
    local pieces = {}
    if button:IsA("TextButton") then table.insert(pieces, button.Text or "") end
    for _, child in ipairs(button:GetDescendants()) do
        if child:IsA("TextLabel") or child:IsA("TextButton") then
            if child.Text and child.Text ~= "" then table.insert(pieces, child.Text) end
        end
    end
    return string.lower(table.concat(pieces, " "))
end

local function isReadySessionRewardButton(button)
    if not (button:IsA("TextButton") or button:IsA("ImageButton")) then return false end
    -- Do not require Visible=true here. War Tycoon's rewards panel may be
    -- closed/hidden while its Claim buttons and Activated connections remain
    -- in PlayerGui. The background scanner must still be able to find them.
    if not button.Parent then return false end
    local activeOk, active = pcall(function() return button.Active end)
    if activeOk and active == false then return false end
    local interactableOk, interactable = pcall(function() return button.Interactable end)
    if interactableOk and interactable == false then return false end

    local text = getSessionRewardButtonText(button)
    if text:find("claimed", 1, true) or text:find("claiming", 1, true)
        or text:find("collected", 1, true) or text:find("collecting", 1, true) then
        return false
    end
    local hasReadyText = text:find("claim", 1, true) or text:find("collect", 1, true) or text:find("redeem", 1, true)
    if hasReadyText then return true end

    -- Some reward buttons are image-only and identify themselves by instance name.
    local buttonName = string.lower(button.Name or "")
    local nameLooksClaimable = buttonName:find("claim", 1, true) or buttonName:find("collect", 1, true)
    local looksLikeTimer = text:match("%d+:%d+") or text:find("min", 1, true) or text:find("sec", 1, true)
    return nameLooksClaimable and not looksLikeTimer and text == ""
end

local function fireSessionRewardButton(button)
    -- Prefer the executor's signal bridge. Signal activation can work even
    -- while the reward panel or its ScreenGui is hidden/disabled.
    if not button or not button.Parent then return false end
    if type(firesignal) == "function" then
        local function hasConnections(signal)
            if type(getconnections) ~= "function" then return nil end
            local ok, connections = pcall(getconnections, signal)
            if not ok or type(connections) ~= "table" then return nil end
            return #connections > 0
        end
        local activatedHasConnections = hasConnections(button.Activated)
        local clickHasConnections = hasConnections(button.MouseButton1Click)
        if activatedHasConnections == true then
            local ok = pcall(firesignal, button.Activated)
            if ok then return true end
        elseif clickHasConnections == true then
            local ok = pcall(firesignal, button.MouseButton1Click)
            if ok then return true end
        elseif activatedHasConnections == nil and clickHasConnections == nil then
            local ok = pcall(firesignal, button.Activated)
            if ok then return true end
        end
    end

    -- Physical mouse fallback is only safe when the target is actually
    -- visible. Never click stale coordinates for a hidden rewards panel.
    local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
    if sessionRewardIsVisible(button, playerGui)
        and type(mousemoveabs) == "function" and type(mouse1click) == "function" then
        local previousMouse
        pcall(function() previousMouse = UserInputService:GetMouseLocation() end)
        local center = button.AbsolutePosition + (button.AbsoluteSize / 2)
        local moved = pcall(mousemoveabs, math.floor(center.X), math.floor(center.Y))
        if moved then
            task.wait(0.06)
            local clicked = pcall(mouse1click)
            task.wait(0.06)
            if previousMouse then pcall(mousemoveabs, math.floor(previousMouse.X), math.floor(previousMouse.Y)) end
            if clicked then return true end
        end
    end
    return false
end

RuntimeState.setAutoSessionReward = function(enabled)
    RuntimeState.Utility.autoSessionRewardEnabled = enabled == true
    sessionRewardLoopVersion += 1
    if not RuntimeState.Utility.autoSessionRewardEnabled then
        return "AUTO SESSION REWARD DISABLED"
    end

    local thisLoopVersion = sessionRewardLoopVersion
    task.spawn(function()
        while RuntimeState.active and sessionRewardLoopVersion == thisLoopVersion
            and RuntimeState.Utility.autoSessionRewardEnabled do
            local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
            local foundRewardUi, clicked = false, false
            if playerGui then
                local scopes = collectSessionRewardScopes(playerGui)
                for _, button in ipairs(playerGui:GetDescendants()) do
                    if (button:IsA("TextButton") or button:IsA("ImageButton"))
                        and isInsideSessionRewardScope(button, playerGui, scopes) then
                        foundRewardUi = true
                        if isReadySessionRewardButton(button) then
                            local now = os.clock()
                            local lastButtonClick = sessionRewardLastButtonClick[button] or 0
                            if now - lastButtonClick >= 12 and now - sessionRewardLastClickAt >= 2.5 then
                                if fireSessionRewardButton(button) then
                                    sessionRewardLastButtonClick[button] = now
                                    sessionRewardLastClickAt = now
                                    clicked = true
                                    break
                                end
                            end
                        end
                    end
                end
            end

            if RuntimeState.AutoSessionRewardRow then
                if clicked then
                    RuntimeState.AutoSessionRewardRow.setStatus("CLAIM SENT  /  WAITING FOR GAME UI")
                elseif foundRewardUi then
                    RuntimeState.AutoSessionRewardRow.setStatus("SCANNING REWARDS  /  PANEL CAN STAY CLOSED")
                elseif type(firesignal) ~= "function" and not (type(mousemoveabs) == "function" and type(mouse1click) == "function") then
                    RuntimeState.AutoSessionRewardRow.setStatus("EXECUTOR UI INPUT UNAVAILABLE")
                else
                    RuntimeState.AutoSessionRewardRow.setStatus("SEARCHING SESSION REWARD UI  /  CLOSED OR OPEN")
                end
            end
            task.wait(1.25)
        end
    end)
    return "SCANNING SESSION / PLAYTIME REWARDS"
end

RuntimeState.AutoSessionRewardRow = RuntimeState.createUtilityToggleRow(
    "AutoSessionReward", "Auto Session Reward", "Scans and claims ready War Tycoon session/playtime rewards even while the rewards panel is hidden, when its UI controls remain loaded.", 1910,
    RuntimeState.Utility.autoSessionRewardEnabled,
    function(enabled)
        local statusText = RuntimeState.setAutoSessionReward(enabled)
        if scheduleConfigSave then scheduleConfigSave() end
        return statusText
    end
)
if RuntimeState.Utility.autoSessionRewardEnabled then
    RuntimeState.AutoSessionRewardRow.setStatus(RuntimeState.setAutoSessionReward(true))
end

-- Auto Skip for skin crates. This watches visible PlayerGui controls only,
-- requires crate/skin context, presses Skip first, then Confirm in the same UI
-- scope. It never invokes game remotes or unrelated confirmation buttons.
local autoSkipLoopVersion = 0
local autoSkipLastButtonAt = setmetatable({}, {__mode = "k"})
local autoSkipPending = nil
local autoSkipContextWords = {
    "crate", "case", "skin", "camo", "unbox", "opening", "loot",
    "new item", "new skin", "you got", "obtained", "unlocked",
}

local function autoSkipGetButtonText(button)
    local parts = {}
    if button:IsA("TextButton") then
        table.insert(parts, button.Text or "")
    end
    for _, child in ipairs(button:GetDescendants()) do
        if child:IsA("TextLabel") or child:IsA("TextButton") or child:IsA("TextBox") then
            local textValue = child.Text
            if type(textValue) == "string" and textValue ~= "" then
                table.insert(parts, textValue)
            end
        end
    end
    return string.lower(table.concat(parts, " "))
end

local function autoSkipButtonVisible(button, playerGui)
    if not button or not button.Parent then return false end
    local node = button
    while node and node ~= playerGui do
        if node:IsA("GuiObject") and not node.Visible then return false end
        if node:IsA("ScreenGui") and not node.Enabled then return false end
        node = node.Parent
    end
    local activeOk, active = pcall(function() return button.Active end)
    if activeOk and active == false then return false end
    local interactableOk, interactable = pcall(function() return button.Interactable end)
    if interactableOk and interactable == false then return false end
    return true
end

local function autoSkipHasContext(scope)
    if not scope then return false end
    local nodeName = string.lower(scope.Name or "")
    for _, word in ipairs(autoSkipContextWords) do
        if string.find(nodeName, word, 1, true) then return true end
    end
    local scanned = 0
    for _, child in ipairs(scope:GetDescendants()) do
        if child:IsA("TextLabel") or child:IsA("TextButton") or child:IsA("TextBox") then
            local textValue = string.lower(child.Text or "")
            for _, word in ipairs(autoSkipContextWords) do
                if string.find(textValue, word, 1, true) then return true end
            end
            scanned += 1
            if scanned >= 180 then break end
        end
    end
    return false
end

local function autoSkipFindCrateScope(button, playerGui)
    local node = button.Parent
    local depth = 0
    while node and node ~= playerGui and depth < 9 do
        if node:IsA("GuiObject") and autoSkipHasContext(node) then
            return node
        end
        node = node.Parent
        depth += 1
    end
    return nil
end

local function autoSkipFindScreenGui(object, playerGui)
    local node = object
    while node and node ~= playerGui do
        if node:IsA("ScreenGui") then return node end
        node = node.Parent
    end
    return nil
end

local function autoSkipSameScope(button, root, screenGui, playerGui)
    if not button or not button.Parent then return false end
    if root then
        local ok, inside = pcall(function() return button:IsDescendantOf(root) end)
        if ok and inside then return true end
    end
    local candidateRoot = autoSkipFindCrateScope(button, playerGui)
    if root and candidateRoot then
        local rootContains = false
        pcall(function() rootContains = candidateRoot == root or candidateRoot:IsDescendantOf(root) or root:IsDescendantOf(candidateRoot) end)
        if rootContains then return true end
    end
    local candidateScreen = autoSkipFindScreenGui(button, playerGui)
    return screenGui ~= nil and candidateScreen == screenGui and candidateRoot ~= nil
end

local function autoSkipButtonKind(button)
    if not (button:IsA("TextButton") or button:IsA("ImageButton")) then return nil end
    local textValue = autoSkipGetButtonText(button)
    local nameValue = string.lower(button.Name or "")
    local skipText = textValue:match("%f[%a]skip%f[%A]") ~= nil
        or string.find(textValue, "skip animation", 1, true) ~= nil
        or string.find(textValue, "skip opening", 1, true) ~= nil
        or string.find(textValue, "skip reveal", 1, true) ~= nil
    local skipName = nameValue:find("skip", 1, true) ~= nil
    if skipText or (skipName and textValue == "") then return "skip" end

    local confirmText = textValue:match("%f[%a]confirm%f[%A]") ~= nil
    local confirmName = nameValue:find("confirm", 1, true) ~= nil
    if confirmText or (confirmName and textValue == "") then return "confirm" end
    return nil
end

local function autoSkipFireButton(button, playerGui)
    if not autoSkipButtonVisible(button, playerGui) then return false end
    if type(firesignal) == "function" then
        if type(getconnections) == "function" then
            local activatedOk, activatedConnections = pcall(getconnections, button.Activated)
            if activatedOk and type(activatedConnections) == "table" and #activatedConnections > 0 then
                local ok = pcall(firesignal, button.Activated)
                if ok then return true end
            end
            local clickOk, clickConnections = pcall(getconnections, button.MouseButton1Click)
            if clickOk and type(clickConnections) == "table" and #clickConnections > 0 then
                local ok = pcall(firesignal, button.MouseButton1Click)
                if ok then return true end
            end
        else
            local ok = pcall(firesignal, button.Activated)
            if ok then return true end
        end
    end

    -- Physical input fallback is restricted to buttons that are currently visible.
    if type(mousemoveabs) == "function" and type(mouse1click) == "function" then
        local previousMouse
        pcall(function() previousMouse = UserInputService:GetMouseLocation() end)
        local center = button.AbsolutePosition + (button.AbsoluteSize / 2)
        local moved = pcall(mousemoveabs, math.floor(center.X), math.floor(center.Y))
        if moved then
            task.wait(0.05)
            local clicked = pcall(mouse1click)
            task.wait(0.05)
            if previousMouse then pcall(mousemoveabs, math.floor(previousMouse.X), math.floor(previousMouse.Y)) end
            if clicked then return true end
        end
    end
    return false
end

RuntimeState.setAutoSkipCrates = function(enabled)
    RuntimeState.Utility.autoSkipCratesEnabled = enabled == true
    autoSkipLoopVersion += 1
    autoSkipPending = nil
    if not RuntimeState.Utility.autoSkipCratesEnabled then
        return "AUTO SKIP DISABLED"
    end

    local thisLoopVersion = autoSkipLoopVersion
    task.spawn(function()
        while RuntimeState.active and autoSkipLoopVersion == thisLoopVersion
            and RuntimeState.Utility.autoSkipCratesEnabled do
            local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
            local didAction = false
            local now = os.clock()

            if playerGui then
                if autoSkipPending then
                    if now - autoSkipPending.startedAt > 12 then
                        autoSkipPending = nil
                    else
                        for _, button in ipairs(playerGui:GetDescendants()) do
                            if (button:IsA("TextButton") or button:IsA("ImageButton"))
                                and autoSkipButtonKind(button) == "confirm"
                                and autoSkipButtonVisible(button, playerGui)
                                and autoSkipSameScope(button, autoSkipPending.root, autoSkipPending.screenGui, playerGui) then
                                local lastAt = autoSkipLastButtonAt[button] or 0
                                if now - lastAt >= 4 and autoSkipFireButton(button, playerGui) then
                                    autoSkipLastButtonAt[button] = now
                                    didAction = true
                                    autoSkipPending = nil
                                    break
                                end
                            end
                        end
                    end
                else
                    for _, button in ipairs(playerGui:GetDescendants()) do
                        if (button:IsA("TextButton") or button:IsA("ImageButton"))
                            and autoSkipButtonKind(button) == "skip"
                            and autoSkipButtonVisible(button, playerGui) then
                            local scope = autoSkipFindCrateScope(button, playerGui)
                            if scope then
                                local lastAt = autoSkipLastButtonAt[button] or 0
                                if now - lastAt >= 5 and autoSkipFireButton(button, playerGui) then
                                    autoSkipLastButtonAt[button] = now
                                    autoSkipPending = {
                                        root = scope,
                                        screenGui = autoSkipFindScreenGui(button, playerGui),
                                        startedAt = now,
                                    }
                                    didAction = true
                                    break
                                end
                            end
                        end
                    end
                end
            end

            if RuntimeState.AutoSkipCratesRow then
                if didAction and autoSkipPending then
                    RuntimeState.AutoSkipCratesRow.setStatus("SKIP PRESSED  /  WAITING FOR CONFIRM")
                elseif didAction then
                    RuntimeState.AutoSkipCratesRow.setStatus("CONFIRM PRESSED  /  CRATE COMPLETE")
                elseif autoSkipPending then
                    RuntimeState.AutoSkipCratesRow.setStatus("WAITING FOR CONFIRM BUTTON")
                else
                    RuntimeState.AutoSkipCratesRow.setStatus("WAITING FOR SKIN CRATE OPENING")
                end
            end
            task.wait(0.35)
        end
    end)
    return "AUTO SKIP READY  /  SKIP THEN CONFIRM"
end

RuntimeState.AutoSkipCratesRow = RuntimeState.createUtilityToggleRow(
    "AutoSkipCrates", "Auto Skip Skin Crates", "Skips the crate reveal animation, then presses Confirm in the same skin/crate UI.", 1990,
    RuntimeState.Utility.autoSkipCratesEnabled,
    function(enabled)
        local statusText = RuntimeState.setAutoSkipCrates(enabled)
        if scheduleConfigSave then scheduleConfigSave() end
        return statusText
    end
)
if RuntimeState.Utility.autoSkipCratesEnabled then
    RuntimeState.AutoSkipCratesRow.setStatus(RuntimeState.setAutoSkipCrates(true))
end

-- Restore persistent local preferences after the controls have been created.
if RuntimeState.Utility.antiAfkEnabled then
    RuntimeState.AntiAfkRow.setStatus(RuntimeState.setAntiAfk(true))
end
if RuntimeState.Utility.noTextureEnabled then
    RuntimeState.NoTextureRow.setStatus(RuntimeState.setNoTexture(true))
end
if RuntimeState.Utility.noGuiEnabled then
    RuntimeState.NoGuiRow.setStatus(RuntimeState.setNoGui(true))
end

-- Theme color editor: same compact layout and interaction model as Jump Circle.
RuntimeState.ThemeColorPicker = make("Frame", {
    Name = "CATScriptThemeColorEditor",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
    Visible = false,
    ZIndex = 360,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.ThemeColorPicker, 8)
stroke(RuntimeState.ThemeColorPicker, COLORS.outline, 0.25, 1)
RuntimeState.ThemePickerTitle = addLabel(RuntimeState.ThemeColorPicker, "THEME COLOR", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.ThemePickerTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.ThemePickerTitle.Size = UDim2.new(1, -48, 0, 17)

RuntimeState.ThemePickerSVArea = make("Frame", {
    Name = "ThemeSVArea", Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158),
    BackgroundColor3 = Color3.fromHSV(0.72, 1, 1), BorderSizePixel = 0, ClipsDescendants = true, ZIndex = 361,
}, RuntimeState.ThemeColorPicker)
corner(RuntimeState.ThemePickerSVArea, 3)
RuntimeState.ThemePickerSVWhite = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, ZIndex = 362,
}, RuntimeState.ThemePickerSVArea)
make("UIGradient", {
    Rotation = 0, Color = ColorSequence.new(Color3.new(1, 1, 1)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1)}),
}, RuntimeState.ThemePickerSVWhite)
RuntimeState.ThemePickerSVBlack = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0, ZIndex = 363,
}, RuntimeState.ThemePickerSVArea)
make("UIGradient", {
    Rotation = 90, Color = ColorSequence.new(Color3.new(0, 0, 0)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0)}),
}, RuntimeState.ThemePickerSVBlack)
RuntimeState.ThemePickerSVInput = make("TextButton", {
    Name = "ThemeSVInput", Text = "", BackgroundTransparency = 1, BorderSizePixel = 0,
    Size = UDim2.fromScale(1, 1), AutoButtonColor = false, ZIndex = 365,
}, RuntimeState.ThemePickerSVArea)
RuntimeState.ThemePickerSVMarker = make("Frame", {
    Name = "ThemeSelectionMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(1, 0), Size = UDim2.fromOffset(11, 11),
    BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 366,
}, RuntimeState.ThemePickerSVArea)
corner(RuntimeState.ThemePickerSVMarker, 8)
stroke(RuntimeState.ThemePickerSVMarker, Color3.new(1, 1, 1), 0, 2)

RuntimeState.ThemePickerHueInput = make("TextButton", {
    Name = "ThemeHueInput", Position = UDim2.fromOffset(224, 29),
    Size = UDim2.fromOffset(18, 158), Text = "", BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, AutoButtonColor = false, ClipsDescendants = true, ZIndex = 361,
}, RuntimeState.ThemeColorPicker)
corner(RuntimeState.ThemePickerHueInput, 3)
RuntimeState.ThemePickerHueGradient = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0, ZIndex = 362,
}, RuntimeState.ThemePickerHueInput)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
        ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
        ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
        ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
        ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
    }),
}, RuntimeState.ThemePickerHueGradient)
RuntimeState.ThemePickerHueMarker = make("Frame", {
    Name = "ThemeHueMarker", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 0, 0),
    Size = UDim2.new(1, 4, 0, 3), BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, ZIndex = 366,
}, RuntimeState.ThemePickerHueInput)
stroke(RuntimeState.ThemePickerHueMarker, Color3.fromRGB(30, 30, 30), 0, 1)

RuntimeState.ThemePickerHexBox = make("TextBox", {
    Name = "ThemeHexInput", Position = UDim2.fromOffset(12, 198),
    Size = UDim2.fromOffset(118, 26), BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
    Text = "#866FF4", TextColor3 = COLORS.text, PlaceholderText = "#RRGGBB",
    PlaceholderColor3 = COLORS.dim, TextSize = 10, Font = Enum.Font.Gotham, ClearTextOnFocus = false,
}, RuntimeState.ThemeColorPicker)
corner(RuntimeState.ThemePickerHexBox, 3)
stroke(RuntimeState.ThemePickerHexBox, COLORS.outline, 0, 1)
RuntimeState.ThemePickerRGBBox = make("TextBox", {
    Name = "ThemeRGBInput", Position = UDim2.fromOffset(138, 198),
    Size = UDim2.fromOffset(126, 26), BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
    Text = "134, 111, 244", TextColor3 = COLORS.text, PlaceholderText = "R, G, B",
    PlaceholderColor3 = COLORS.dim, TextSize = 10, Font = Enum.Font.Gotham, ClearTextOnFocus = false,
}, RuntimeState.ThemeColorPicker)
corner(RuntimeState.ThemePickerRGBBox, 3)
stroke(RuntimeState.ThemePickerRGBBox, COLORS.outline, 0, 1)

RuntimeState.configureFloatingColorEditor(RuntimeState.ThemeColorPicker, RuntimeState.ThemePickerTitle, 6)

-- Full-range custom cursor color editor: HSV square + hue rail + HEX/RGB input.
RuntimeState.CursorColorHSV = {}
RuntimeState.CursorColorHSV.h, RuntimeState.CursorColorHSV.s, RuntimeState.CursorColorHSV.v = cursorSettings.color:ToHSV()
RuntimeState.CursorColorPicker = make("Frame", {
    Name = "CATScriptCursorColorEditor",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232),
    BackgroundColor3 = COLORS.panel2,
    BorderSizePixel = 0,
    Visible = false,
    ZIndex = 380,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.CursorColorPicker, 8)
stroke(RuntimeState.CursorColorPicker, COLORS.outline, 0.25, 1)
RuntimeState.CursorColorPickerTitle = addLabel(RuntimeState.CursorColorPicker, "ЦВЕТ КУРСОРА", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.CursorColorPickerTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.CursorColorPickerTitle.Size = UDim2.new(1, -48, 0, 17)
RuntimeState.CursorColorSVArea = make("Frame", {
    Name = "CursorSaturationValueArea",
    Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158),
    BackgroundColor3 = Color3.fromHSV(RuntimeState.CursorColorHSV.h, 1, 1),
    BorderSizePixel = 0,
    ClipsDescendants = true,
    ZIndex = 381,
}, RuntimeState.CursorColorPicker)
corner(RuntimeState.CursorColorSVArea, 3)
RuntimeState.CursorColorSVWhite = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, ZIndex = 382,
}, RuntimeState.CursorColorSVArea)
make("UIGradient", {
    Rotation = 0, Color = ColorSequence.new(Color3.new(1, 1, 1)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1)}),
}, RuntimeState.CursorColorSVWhite)
RuntimeState.CursorColorSVBlack = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0),
    BorderSizePixel = 0, ZIndex = 383,
}, RuntimeState.CursorColorSVArea)
make("UIGradient", {
    Rotation = 90, Color = ColorSequence.new(Color3.new(0, 0, 0)),
    Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0)}),
}, RuntimeState.CursorColorSVBlack)
RuntimeState.CursorColorSVInput = make("TextButton", {
    Name = "CursorColorSVInput", Text = "", BackgroundTransparency = 1,
    BorderSizePixel = 0, Size = UDim2.fromScale(1, 1), AutoButtonColor = false, ZIndex = 385,
}, RuntimeState.CursorColorSVArea)
RuntimeState.CursorColorSVMarker = make("Frame", {
    Name = "CursorColorSVMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(RuntimeState.CursorColorHSV.s, 1 - RuntimeState.CursorColorHSV.v),
    Size = UDim2.fromOffset(11, 11), BackgroundTransparency = 1,
    BorderSizePixel = 0, ZIndex = 386,
}, RuntimeState.CursorColorSVArea)
corner(RuntimeState.CursorColorSVMarker, 8)
stroke(RuntimeState.CursorColorSVMarker, Color3.new(1, 1, 1), 0, 2)
RuntimeState.CursorColorHueInput = make("TextButton", {
    Name = "CursorColorHueInput", Position = UDim2.fromOffset(224, 29),
    Size = UDim2.fromOffset(18, 158), Text = "", BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, AutoButtonColor = false, ClipsDescendants = true, ZIndex = 381,
}, RuntimeState.CursorColorPicker)
corner(RuntimeState.CursorColorHueInput, 3)
RuntimeState.CursorColorHueGradient = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, ZIndex = 382,
}, RuntimeState.CursorColorHueInput)
make("UIGradient", {
    Rotation = 90,
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
        ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
        ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
        ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
        ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
    }),
}, RuntimeState.CursorColorHueGradient)
RuntimeState.CursorColorHueMarker = make("Frame", {
    Name = "CursorColorHueMarker", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 0, 0),
    Size = UDim2.new(1, 4, 0, 3), BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, ZIndex = 386,
}, RuntimeState.CursorColorHueInput)
stroke(RuntimeState.CursorColorHueMarker, Color3.fromRGB(30, 30, 30), 0, 1)
RuntimeState.CursorColorHexBox = make("TextBox", {
    Name = "CursorColorHexInput", Position = UDim2.fromOffset(12, 198),
    Size = UDim2.fromOffset(118, 26), BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0, Text = "#A678FF", TextColor3 = COLORS.text,
    PlaceholderText = "#RRGGBB", PlaceholderColor3 = COLORS.dim,
    TextSize = 10, Font = Enum.Font.Gotham, ClearTextOnFocus = false, ZIndex = 381,
}, RuntimeState.CursorColorPicker)
corner(RuntimeState.CursorColorHexBox, 3)
stroke(RuntimeState.CursorColorHexBox, COLORS.outline, 0, 1)
RuntimeState.CursorColorRGBBox = make("TextBox", {
    Name = "CursorColorRGBInput", Position = UDim2.fromOffset(138, 198),
    Size = UDim2.fromOffset(126, 26), BackgroundColor3 = COLORS.background,
    BorderSizePixel = 0, Text = "166, 120, 255", TextColor3 = COLORS.text,
    PlaceholderText = "R, G, B", PlaceholderColor3 = COLORS.dim,
    TextSize = 10, Font = Enum.Font.Gotham, ClearTextOnFocus = false, ZIndex = 381,
}, RuntimeState.CursorColorPicker)
corner(RuntimeState.CursorColorRGBBox, 3)
stroke(RuntimeState.CursorColorRGBBox, COLORS.outline, 0, 1)
RuntimeState.configureFloatingColorEditor(RuntimeState.CursorColorPicker, RuntimeState.CursorColorPickerTitle, 8)
for _, cursorPickerDescendant in ipairs(RuntimeState.CursorColorPicker:GetDescendants()) do
    if cursorPickerDescendant:IsA("GuiObject") then
        cursorPickerDescendant.ZIndex = math.max(cursorPickerDescendant.ZIndex, 381)
    end
end

for _, themeDescendant in ipairs(RuntimeState.ThemeColorPicker:GetDescendants()) do
    if themeDescendant:IsA("GuiObject") then
        themeDescendant.ZIndex = math.max(themeDescendant.ZIndex, 361)
    end
end

RuntimeState.ActiveThemeKey = "accent"
RuntimeState.ThemeHue = 0.72
RuntimeState.ThemeSaturation = 0.55
RuntimeState.ThemeValue = 0.95
RuntimeState.refreshThemePicker = function()
    local weatherKey = RuntimeState.Weather.colorNames[RuntimeState.ActiveThemeKey]
    local themeColor = (weatherKey and RuntimeState.Weather.settings[weatherKey].color) or COLORS[RuntimeState.ActiveThemeKey] or COLORS.accent
    RuntimeState.ThemeHue, RuntimeState.ThemeSaturation, RuntimeState.ThemeValue = themeColor:ToHSV()
    RuntimeState.ThemePickerSVArea.BackgroundColor3 = Color3.fromHSV(RuntimeState.ThemeHue, 1, 1)
    RuntimeState.ThemePickerSVMarker.Position = UDim2.fromScale(RuntimeState.ThemeSaturation, 1 - RuntimeState.ThemeValue)
    RuntimeState.ThemePickerHueMarker.Position = UDim2.new(0.5, 0, RuntimeState.ThemeHue, 0)
    RuntimeState.ThemePickerHexBox.Text = string.format("#%02X%02X%02X", math.floor(themeColor.R * 255 + 0.5), math.floor(themeColor.G * 255 + 0.5), math.floor(themeColor.B * 255 + 0.5))
    RuntimeState.ThemePickerRGBBox.Text = string.format("%d, %d, %d", math.floor(themeColor.R * 255 + 0.5), math.floor(themeColor.G * 255 + 0.5), math.floor(themeColor.B * 255 + 0.5))
    RuntimeState.ThemePickerTitle.Text = "РЕДАКТОР: " .. string.upper(RuntimeState.ThemeRoleLabels[RuntimeState.ActiveThemeKey] or "COLOR")
end

-- Register each UI color property to one semantic palette role. A global
-- "same RGB value" replacement can accidentally recolor other roles whenever
-- two palette entries share a color, so changes are now role-specific.
RuntimeState.ThemeColorRegistry = nil
RuntimeState.ThemeRegistrySeen = setmetatable({}, {__mode = "k"})

RuntimeState.registerThemeColorProperty = function(uiObject, propertyName)
    if not RuntimeState.ThemeColorRegistry then return end
    local seen = RuntimeState.ThemeRegistrySeen[uiObject]
    if not seen then
        seen = {}
        RuntimeState.ThemeRegistrySeen[uiObject] = seen
    end
    if seen[propertyName] then return end

    local ok, currentColor = pcall(function() return uiObject[propertyName] end)
    if not ok or typeof(currentColor) ~= "Color3" then return end

    local matchedKey = nil
    local matchCount = 0
    for _, candidateKey in ipairs(RuntimeState.THEME_KEYS) do
        if currentColor == COLORS[candidateKey] then
            matchedKey = candidateKey
            matchCount += 1
        end
    end

    -- Only claim a property if the starting palette identifies one clear role.
    if matchCount ~= 1 then return end
    local bucket = RuntimeState.ThemeColorRegistry[matchedKey]
    if not bucket then
        bucket = {}
        RuntimeState.ThemeColorRegistry[matchedKey] = bucket
    end
    local entry = {object = uiObject, property = propertyName, role = matchedKey}
    bucket[#bucket + 1] = entry
    seen[propertyName] = entry

    -- Toggle/status controls can change between palette roles at runtime
    -- (for example dim -> accent). Keep their semantic ownership in sync.
    if propertyName == "BackgroundColor3" or propertyName == "TextColor3" then
        local signalOk, propertySignal = pcall(function()
            return uiObject:GetPropertyChangedSignal(propertyName)
        end)
        if signalOk and propertySignal then
            entry.changedConnection = propertySignal:Connect(function()
                if not uiObject.Parent then return end
                local valueOk, changedColor = pcall(function() return uiObject[propertyName] end)
                if not valueOk or typeof(changedColor) ~= "Color3" then return end
                if entry.role and changedColor == COLORS[entry.role] then return end

                local newRole = nil
                local newRoleCount = 0
                for _, candidateKey in ipairs(RuntimeState.THEME_KEYS) do
                    if changedColor == COLORS[candidateKey] then
                        newRole = candidateKey
                        newRoleCount += 1
                    end
                end
                if newRoleCount == 1 then
                    entry.role = newRole
                    local newBucket = RuntimeState.ThemeColorRegistry[newRole]
                    if not newBucket then
                        newBucket = {}
                        RuntimeState.ThemeColorRegistry[newRole] = newBucket
                    end
                    newBucket[#newBucket + 1] = entry
                else
                    entry.role = nil
                end
            end)
        end
    end
end

RuntimeState.registerThemeColorObject = function(uiObject)
    if not uiObject or not uiObject.Parent then return end
    if RuntimeState.CustomCursorOverlay and (uiObject == RuntimeState.CustomCursorOverlay or uiObject:IsDescendantOf(RuntimeState.CustomCursorOverlay)) then return end
    for _, propertyName in ipairs({
        "BackgroundColor3", "TextColor3", "TextStrokeColor3", "PlaceholderColor3",
        "BorderColor3", "ImageColor3", "ScrollBarImageColor3",
    }) do
        RuntimeState.registerThemeColorProperty(uiObject, propertyName)
    end
    if uiObject:IsA("UIStroke") then
        RuntimeState.registerThemeColorProperty(uiObject, "Color")
    end
end

RuntimeState.ensureThemeColorRegistry = function()
    if RuntimeState.ThemeColorRegistry then return end
    RuntimeState.ThemeColorRegistry = {}
    for _, uiObject in ipairs(ScreenGui:GetDescendants()) do
        RuntimeState.registerThemeColorObject(uiObject)
    end
    RuntimeState.ThemeRegistryConnection = ScreenGui.DescendantAdded:Connect(function(uiObject)
        RuntimeState.registerThemeColorObject(uiObject)
    end)
end

RuntimeState.applyThemeColor = function(themeKey, newColor, shouldSave)
    if typeof(newColor) ~= "Color3" then return end
    local weatherKey = RuntimeState.Weather.colorNames[themeKey]
    if weatherKey then
        RuntimeState.Weather.settings[weatherKey].color = newColor
        local weatherRow = RuntimeState.WeatherRows and RuntimeState.WeatherRows[weatherKey]
        if weatherRow then weatherRow.color.BackgroundColor3 = newColor end
        if RuntimeState.ActiveThemeKey == themeKey then RuntimeState.refreshThemePicker() end
        if shouldSave ~= false and scheduleConfigSave then scheduleConfigSave() end
        return
    end
    if not COLORS[themeKey] then return end
    RuntimeState.ensureThemeColorRegistry()
    COLORS[themeKey] = newColor

    local roleProperties = RuntimeState.ThemeColorRegistry[themeKey]
    if roleProperties then
        for _, reference in ipairs(roleProperties) do
            local object = reference.object
            if reference.role == themeKey and object and object.Parent then
                pcall(function() object[reference.property] = newColor end)
            end
        end
    end

    for _, paletteKey in ipairs(RuntimeState.THEME_KEYS) do
        local entry = RuntimeState.ThemeButtons[paletteKey]
        if entry then entry.swatch.BackgroundColor3 = COLORS[paletteKey] end
    end
    if RuntimeState.ThemeColorPicker and RuntimeState.ActiveThemeKey == themeKey then
        RuntimeState.refreshThemePicker()
    end
    if shouldSave ~= false and scheduleConfigSave then scheduleConfigSave() end
end

RuntimeState.openThemePicker = function(themeKey)
    RuntimeState.ActiveThemeKey = themeKey
    RuntimeState.refreshThemePicker()
    -- Keep the same floating picker open when the user switches palette entries.
    if not RuntimeState.ThemeColorPicker.Visible then
        RuntimeState.openFloatingColorEditor(RuntimeState.ThemeColorPicker, 6)
    end
end
for _, themeKey in ipairs(RuntimeState.THEME_KEYS) do
    local entry = RuntimeState.ThemeButtons[themeKey]
    entry.button.MouseButton1Click:Connect(function()
        RuntimeState.openThemePicker(themeKey)
    end)
end

RuntimeState.updateThemeColorFromSV = function(pointer)
    local origin = RuntimeState.ThemePickerSVArea.AbsolutePosition
    local bounds = RuntimeState.ThemePickerSVArea.AbsoluteSize
    if bounds.X <= 0 or bounds.Y <= 0 then return end
    RuntimeState.ThemeSaturation = math.clamp((pointer.X - origin.X) / bounds.X, 0, 1)
    RuntimeState.ThemeValue = 1 - math.clamp((pointer.Y - origin.Y) / bounds.Y, 0, 1)
    RuntimeState.applyThemeColor(RuntimeState.ActiveThemeKey, Color3.fromHSV(RuntimeState.ThemeHue, RuntimeState.ThemeSaturation, RuntimeState.ThemeValue), true)
end
RuntimeState.updateThemeHueFromPointer = function(pointer)
    local origin = RuntimeState.ThemePickerHueInput.AbsolutePosition
    local bounds = RuntimeState.ThemePickerHueInput.AbsoluteSize
    if bounds.Y <= 0 then return end
    RuntimeState.ThemeHue = math.clamp((pointer.Y - origin.Y) / bounds.Y, 0, 1)
    RuntimeState.applyThemeColor(RuntimeState.ActiveThemeKey, Color3.fromHSV(RuntimeState.ThemeHue, RuntimeState.ThemeSaturation, RuntimeState.ThemeValue), true)
end
RuntimeState.ThemeColorDragMode = nil
RuntimeState.ThemePickerSVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.ThemeColorDragMode = "sv"
        RuntimeState.updateThemeColorFromSV(getColorEditorPointer(input))
    end
end)
RuntimeState.ThemePickerHueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.ThemeColorDragMode = "hue"
        RuntimeState.updateThemeHueFromPointer(getColorEditorPointer(input))
    end
end)
RuntimeState.themePickerInputChanged = UserInputService.InputChanged:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
    local pointer = getColorEditorPointer(input)
    if RuntimeState.ThemeColorDragMode == "sv" then
        RuntimeState.updateThemeColorFromSV(pointer)
    elseif RuntimeState.ThemeColorDragMode == "hue" then
        RuntimeState.updateThemeHueFromPointer(pointer)
    end
    if RuntimeState.menuTransparencyDragging then
        local bounds = RuntimeState.ThemeTransparencyTrack.AbsoluteSize
        local origin = RuntimeState.ThemeTransparencyTrack.AbsolutePosition
        local fraction = bounds.X > 0 and math.clamp((pointer.X - origin.X) / bounds.X, 0, 1) or 0
        RuntimeState.menuTransparency = fraction * 0.70
        RuntimeState.applyMenuTransparency()
    end
end)
RuntimeState.themePickerInputEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.ThemeColorDragMode = nil
        if RuntimeState.menuTransparencyDragging then
            RuntimeState.menuTransparencyDragging = false
            if scheduleConfigSave then scheduleConfigSave() end
        end
    end
end)

RuntimeState.ThemePickerHexBox.FocusLost:Connect(function()
    local cleanedHex = RuntimeState.ThemePickerHexBox.Text:gsub("#", ""):match("^%x%x%x%x%x%x$")
    if cleanedHex then
        local numeric = tonumber(cleanedHex, 16)
        RuntimeState.applyThemeColor(RuntimeState.ActiveThemeKey, Color3.fromRGB(math.floor(numeric / 65536) % 256, math.floor(numeric / 256) % 256, numeric % 256), true)
    else
        RuntimeState.refreshThemePicker()
    end
end)
RuntimeState.ThemePickerRGBBox.FocusLost:Connect(function()
    local red, green, blue = RuntimeState.ThemePickerRGBBox.Text:match("^%s*(%d+)%s*,%s*(%d+)%s*,%s*(%d+)%s*$")
    if red and green and blue then
        RuntimeState.applyThemeColor(RuntimeState.ActiveThemeKey, Color3.fromRGB(math.clamp(tonumber(red), 0, 255), math.clamp(tonumber(green), 0, 255), math.clamp(tonumber(blue), 0, 255)), true)
    else
        RuntimeState.refreshThemePicker()
    end
end)

RuntimeState.applyMenuTransparency = function()
    local amount = math.clamp(RuntimeState.menuTransparency or 0, 0, 0.70)
    local function panelAlpha(base)
        return math.clamp(base + (1 - base) * amount, 0, 0.96)
    end
    Window.BackgroundTransparency = amount
    Sidebar.BackgroundTransparency = panelAlpha(0.24)
    RuntimeState.Topbar.BackgroundTransparency = panelAlpha(0.24)
    RuntimeState.VisualCard.BackgroundTransparency = panelAlpha(0.24)
    RuntimeState.SettingsCard.BackgroundTransparency = panelAlpha(0.24)
    if RuntimeState.PlayerCard then RuntimeState.PlayerCard.BackgroundTransparency = panelAlpha(0.24) end
    RuntimeState.PreviewPanel.BackgroundTransparency = panelAlpha(0.18)
    RuntimeState.PreviewViewport.BackgroundTransparency = panelAlpha(0.02)
    RuntimeState.Console.BackgroundTransparency = panelAlpha(0.08)
    RuntimeState.HoverRow.BackgroundTransparency = panelAlpha(0)
    RuntimeState.TrailRow.BackgroundTransparency = panelAlpha(0)
    RuntimeState.JumpCircleRow.BackgroundTransparency = panelAlpha(0)
    if RuntimeState.CustomAuraRow then RuntimeState.CustomAuraRow.BackgroundTransparency = panelAlpha(0) end
    RuntimeState.KeybindRow.BackgroundTransparency = panelAlpha(0.18)
    RuntimeState.AutoSaveRow.BackgroundTransparency = panelAlpha(0.18)
    RuntimeState.BackgroundSettingsRow.BackgroundTransparency = panelAlpha(0.18)
    RuntimeState.ThemeTransparencyRow.BackgroundTransparency = panelAlpha(0)
    RuntimeState.FontSettingsRow.BackgroundTransparency = panelAlpha(0.18)
    RuntimeState.CursorSettingsRow.BackgroundTransparency = panelAlpha(0.18)
    RuntimeState.KeybindButton.BackgroundTransparency = panelAlpha(0)
    RuntimeState.ResetThemeButton.BackgroundTransparency = panelAlpha(0)
    InfoCard.BackgroundTransparency = panelAlpha(0.38)
    for _, metricRow in pairs(RuntimeState.MetricRows) do
        metricRow.BackgroundTransparency = panelAlpha(0)
    end
    for _, themeEntry in pairs(RuntimeState.ThemeButtons) do
        themeEntry.button.BackgroundTransparency = panelAlpha(0)
    end
    RuntimeState.ThemeTransparencyFill.Size = UDim2.new(amount / 0.70, 0, 1, 0)
    RuntimeState.ThemeTransparencyKnob.Position = UDim2.new(amount / 0.70, 0, 0.5, 0)
    RuntimeState.ThemeTransparencyValue.Text = tostring(math.floor(amount * 100 + 0.5)) .. "%"
    if scheduleConfigSave then scheduleConfigSave() end
end
RuntimeState.ThemeTransparencyTrack.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.menuTransparencyDragging = true
        local bounds = RuntimeState.ThemeTransparencyTrack.AbsoluteSize
        local origin = RuntimeState.ThemeTransparencyTrack.AbsolutePosition
        local fraction = bounds.X > 0 and math.clamp((input.Position.X - origin.X) / bounds.X, 0, 1) or 0
        RuntimeState.menuTransparency = fraction * 0.70
        RuntimeState.applyMenuTransparency()
    end
end)

RuntimeState.ResetThemeButton.MouseButton1Click:Connect(function()
    for _, themeKey in ipairs(RuntimeState.THEME_KEYS) do
        RuntimeState.applyThemeColor(themeKey, RuntimeState.DefaultThemeColors[themeKey], false)
    end
    RuntimeState.menuTransparency = 0
    RuntimeState.applyMenuTransparency()
    RuntimeState.refreshThemePicker()
    if scheduleConfigSave then scheduleConfigSave() end
end)
RuntimeState.refreshThemePicker()

RuntimeState.buildFontEditor = function()
    -- Floating font editor: each option previews itself; choosing one updates every
    -- other text object in the menu immediately and is persisted with Auto Save.
    RuntimeState.FontPicker = make("Frame", {
        Name = "CATScriptFontEditor",
        Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(278, 326),
        BackgroundColor3 = COLORS.panel2,
        BorderSizePixel = 0,
        Visible = false,
        ZIndex = 420,
    }, RuntimeState.floatingEditorsLayer)
    corner(RuntimeState.FontPicker, 8)
    stroke(RuntimeState.FontPicker, COLORS.outline, 0.25, 1)
    RuntimeState.FontPickerTitle = addLabel(RuntimeState.FontPicker, "ВЫБОР ШРИФТА", 8, COLORS.muted, Enum.Font.GothamBold)
    RuntimeState.FontPickerTitle.Position = UDim2.fromOffset(12, 6)
    RuntimeState.FontPickerTitle.Size = UDim2.new(1, -48, 0, 17)
    RuntimeState.FontPickerTitle.ZIndex = 421
    RuntimeState.FontPickerHint = addLabel(RuntimeState.FontPicker, "Нажми на вариант, чтобы применить его сразу", 7, COLORS.dim, Enum.Font.Gotham)
    RuntimeState.FontPickerHint.Position = UDim2.fromOffset(12, 24)
    RuntimeState.FontPickerHint.Size = UDim2.new(1, -24, 0, 14)
    RuntimeState.FontPickerHint.ZIndex = 421
    RuntimeState.FontOptionList = make("ScrollingFrame", {
        Name = "FontOptionList",
        Position = UDim2.fromOffset(10, 44),
        Size = UDim2.new(1, -20, 1, -54),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        ScrollBarThickness = 3,
        ScrollBarImageColor3 = COLORS.dim,
        CanvasSize = UDim2.fromOffset(0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ZIndex = 421,
    }, RuntimeState.FontPicker)
    make("UIPadding", {
        PaddingTop = UDim.new(0, 2), PaddingBottom = UDim.new(0, 4),
        PaddingLeft = UDim.new(0, 2), PaddingRight = UDim.new(0, 5),
    }, RuntimeState.FontOptionList)
    make("UIListLayout", {
        Padding = UDim.new(0, 4),
        SortOrder = Enum.SortOrder.LayoutOrder,
    }, RuntimeState.FontOptionList)
    RuntimeState.configureFloatingColorEditor(RuntimeState.FontPicker, RuntimeState.FontPickerTitle, 7)

    RuntimeState.FontOptions = {}
    RuntimeState.FontOptionButtons = {}
    RuntimeState.FontLookup = {}
    RuntimeState.FontAdded = {}
    pcall(function()
        for _, fontItem in ipairs(Enum.Font:GetEnumItems()) do
            RuntimeState.FontLookup[fontItem.Name] = fontItem
        end
    end)
    RuntimeState.FontPreferredOrder = {
        "Gotham", "GothamMedium", "GothamBold", "GothamBlack",
        "BuilderSans", "BuilderSansMedium", "BuilderSansBold", "BuilderSansExtraBold",
        "SourceSans", "SourceSansBold", "SourceSansSemibold", "Arial", "ArialBold",
        "Cartoon", "Code", "Fantasy", "SciFi", "Antique", "Arcade",
        "Highway", "Bodoni", "Garamond", "Legacy",
    }
    for _, fontName in ipairs(RuntimeState.FontPreferredOrder) do
        local fontItem = RuntimeState.FontLookup[fontName]
        if fontItem and not RuntimeState.FontAdded[fontName] then
            RuntimeState.FontAdded[fontName] = true
            RuntimeState.FontOptions[#RuntimeState.FontOptions + 1] = fontItem
        end
    end
    for _, fontItem in ipairs(Enum.Font:GetEnumItems()) do
        if not RuntimeState.FontAdded[fontItem.Name] then
            RuntimeState.FontAdded[fontItem.Name] = true
            RuntimeState.FontOptions[#RuntimeState.FontOptions + 1] = fontItem
        end
    end

    RuntimeState.refreshFontEditor = function()
        RuntimeState.FontCurrentButton.Text = RuntimeState.selectedFontName .. "  ▾"
        RuntimeState.FontPreviewSample.Font = RuntimeState.selectedFont
        RuntimeState.FontPreviewSample.Text = "CATScript  •  Aa Bb 123"
        RuntimeState.FontSettingsTitle.Text = "Текущий шрифт: " .. RuntimeState.selectedFontName
        for _, option in ipairs(RuntimeState.FontOptionButtons) do
            local selected = option.Name == RuntimeState.selectedFontName
            option.Button.BackgroundColor3 = selected and COLORS.accent or COLORS.panel
            option.Button.TextColor3 = selected and COLORS.white or COLORS.text
        end
    end

    RuntimeState.applyFontChoice = function(fontName, shouldSave)
        local chosenFont = RuntimeState.FontLookup[fontName]
        if not chosenFont then return false end
        RuntimeState.selectedFontName = chosenFont.Name
        RuntimeState.selectedFont = chosenFont
        for _, uiObject in ipairs(ScreenGui:GetDescendants()) do
            if (uiObject:IsA("TextLabel") or uiObject:IsA("TextButton") or uiObject:IsA("TextBox"))
                and not uiObject:IsDescendantOf(RuntimeState.FontOptionList) then
                pcall(function() uiObject.Font = chosenFont end)
            end
        end
        RuntimeState.refreshFontEditor()
        if shouldSave ~= false and scheduleConfigSave then scheduleConfigSave() end
        return true
    end

    RuntimeState.createFontOptionButton = function(fontItem, fontIndex)
        local optionButton = make("TextButton", {
            Name = "FontOption_" .. fontItem.Name,
            LayoutOrder = fontIndex,
            Size = UDim2.new(1, -4, 0, 29),
            BackgroundColor3 = fontItem.Name == RuntimeState.selectedFontName and COLORS.accent or COLORS.panel,
            BorderSizePixel = 0,
            Text = "  " .. fontItem.Name .. "     Aa Bb 123",
            TextColor3 = fontItem.Name == RuntimeState.selectedFontName and COLORS.white or COLORS.text,
            TextSize = 10,
            TextXAlignment = Enum.TextXAlignment.Left,
            Font = fontItem,
            AutoButtonColor = false,
            ZIndex = 422,
        }, RuntimeState.FontOptionList)
        corner(optionButton, 5)
        RuntimeState.FontOptionButtons[#RuntimeState.FontOptionButtons + 1] = {Name = fontItem.Name, Button = optionButton}
        optionButton.MouseButton1Click:Connect(function()
            RuntimeState.applyFontChoice(fontItem.Name, true)
        end)
    end
    for fontIndex, fontItem in ipairs(RuntimeState.FontOptions) do
        RuntimeState.createFontOptionButton(fontItem, fontIndex)
    end

    RuntimeState.FontCurrentButton.MouseButton1Click:Connect(function()
        RuntimeState.openFloatingColorEditor(RuntimeState.FontPicker, 7)
    end)
    RuntimeState.FontResetButton.MouseButton1Click:Connect(function()
        RuntimeState.applyFontChoice(RuntimeState.defaultFontName, true)
    end)

    -- Apply the saved font once and keep future text objects consistent as UI elements
    -- are created by widgets or other CATScript panels.
    RuntimeState.applyFontChoice(RuntimeState.selectedFontName, false)
    RuntimeState.FontRegistryConnection = ScreenGui.DescendantAdded:Connect(function(uiObject)
        if uiObject:IsA("TextLabel") or uiObject:IsA("TextButton") or uiObject:IsA("TextBox") then
            if not uiObject:IsDescendantOf(RuntimeState.FontOptionList) then
                pcall(function() uiObject.Font = RuntimeState.selectedFont end)
            end
        end
    end)
end
RuntimeState.buildFontEditor()

-- CATScript compact performance HUD (variant 6). The whole card is draggable
-- and remains visible independently of the main menu.
local savedMetricsPosition = type(SavedConfig.metricsPosition) == "table" and SavedConfig.metricsPosition or {}
local metricsPosition = UDim2.new(1, -195, 1, -66)
if type(savedMetricsPosition.xScale) == "number"
    and type(savedMetricsPosition.xOffset) == "number"
    and type(savedMetricsPosition.yScale) == "number"
    and type(savedMetricsPosition.yOffset) == "number" then
    -- Migrate the untouched default position from the old strip to the new
    -- bottom-right CATScript card; preserve positions the player moved manually.
    local oldDefaultPosition = (savedMetricsPosition.xScale == 0.5
        and savedMetricsPosition.xOffset == 0
        and savedMetricsPosition.yScale == 0
        and savedMetricsPosition.yOffset == 12)
        or (savedMetricsPosition.xScale == 1
        and savedMetricsPosition.xOffset == -310
        and savedMetricsPosition.yScale == 1
        and savedMetricsPosition.yOffset == -92)
    if not oldDefaultPosition then
        metricsPosition = UDim2.new(
            math.clamp(savedMetricsPosition.xScale, -1, 2),
            math.clamp(savedMetricsPosition.xOffset, -4000, 4000),
            math.clamp(savedMetricsPosition.yScale, -1, 2),
            math.clamp(savedMetricsPosition.yOffset, -4000, 4000)
        )
    end
end

RuntimeState.PerformanceWidget = make("Frame", {
    Name = "CATScriptPerformanceWidget",
    AnchorPoint = Vector2.new(0.5, 0),
    Position = metricsPosition,
    Size = UDim2.fromOffset(366, 58),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Active = true,
    Visible = true,
    ZIndex = 800,
}, ScreenGui)

RuntimeState.PerformancePanel = make("Frame", {
    Name = "PerformancePanel",
    Position = UDim2.fromOffset(0, 8),
    Size = UDim2.fromOffset(366, 48),
    BackgroundColor3 = COLORS.panel,
    BackgroundTransparency = 0.08,
    BorderSizePixel = 0,
    ZIndex = 800,
}, RuntimeState.PerformanceWidget)
corner(RuntimeState.PerformancePanel, 13)
stroke(RuntimeState.PerformancePanel, COLORS.outline, 0.12, 1)

RuntimeState.PerformanceBadge = make("Frame", {
    Name = "PerformanceBrandBadge",
    Position = UDim2.fromOffset(8, 0),
    Size = UDim2.fromOffset(86, 16),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    ZIndex = 803,
}, RuntimeState.PerformanceWidget)
corner(RuntimeState.PerformanceBadge, 9)
RuntimeState.PerformanceBrandText = addLabel(RuntimeState.PerformanceBadge, "✦  CATScript", 7, COLORS.white, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
RuntimeState.PerformanceBrandText.Size = UDim2.fromScale(1, 1)
RuntimeState.PerformanceBrandText.ZIndex = 804
RuntimeState.PerformanceBrandText.Active = true

-- Small angled accent at the far-right edge completes the compact CATScript look.
make("Frame", {
    Name = "PerformanceEndAccent",
    AnchorPoint = Vector2.new(1, 1),
    Position = UDim2.new(1, -9, 1, -4),
    Size = UDim2.fromOffset(22, 3),
    Rotation = -8,
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    ZIndex = 802,
}, RuntimeState.PerformanceWidget)

RuntimeState.PERFORMANCE_ORDER = {"fps", "ping", "gpu", "memory"}
RuntimeState.PERFORMANCE_TITLES = {
    fps = "FPS",
    ping = "PING",
    gpu = "GPU",
    memory = "MEMORY",
}
RuntimeState.PERFORMANCE_ICONS = {
    fps = "▣",
    ping = "⌁",
    gpu = "▦",
    memory = "▤",
}
RuntimeState.PERFORMANCE_CELL_WIDTH = 82
RuntimeState.PERFORMANCE_CELL_GAP = 4
RuntimeState.PERFORMANCE_PADDING = 7
RuntimeState.PerformanceCells = {}
RuntimeState.PerformanceDividers = {}

for index, key in ipairs(RuntimeState.PERFORMANCE_ORDER) do
    local cell = make("Frame", {
        Name = "MetricCell_" .. key,
        Position = UDim2.fromOffset(RuntimeState.PERFORMANCE_PADDING + (index - 1) * (RuntimeState.PERFORMANCE_CELL_WIDTH + RuntimeState.PERFORMANCE_CELL_GAP), 17),
        Size = UDim2.fromOffset(RuntimeState.PERFORMANCE_CELL_WIDTH, 32),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Active = true,
        ZIndex = 801,
    }, RuntimeState.PerformanceWidget)

    local icon = addLabel(cell, RuntimeState.PERFORMANCE_ICONS[key], 12, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Center)
    icon.Position = UDim2.fromOffset(0, 7)
    icon.Size = UDim2.fromOffset(16, 18)
    icon.ZIndex = 802
    icon.Active = true

    local title = addLabel(cell, RuntimeState.PERFORMANCE_TITLES[key], 6, COLORS.accent, Enum.Font.GothamBold, Enum.TextXAlignment.Left)
    title.Position = UDim2.fromOffset(17, 1)
    title.Size = UDim2.new(1, -18, 0, 9)
    title.ZIndex = 802
    title.Active = true

    local value = addLabel(cell, "--", 9, COLORS.text, Enum.Font.GothamSemibold, Enum.TextXAlignment.Left)
    value.Position = UDim2.fromOffset(17, 11)
    value.Size = UDim2.new(1, -18, 0, 16)
    value.ZIndex = 802
    value.Active = true

    RuntimeState.PerformanceCells[key] = {frame = cell, icon = icon, title = title, value = value}
end

for index = 1, #RuntimeState.PERFORMANCE_ORDER - 1 do
    RuntimeState.PerformanceDividers[index] = make("Frame", {
        Name = "PerformanceDivider_" .. index,
        Position = UDim2.fromOffset(RuntimeState.PERFORMANCE_PADDING + index * RuntimeState.PERFORMANCE_CELL_WIDTH
            + (index - 1) * RuntimeState.PERFORMANCE_CELL_GAP + 3, 24),
        Size = UDim2.fromOffset(1, 15),
        BackgroundColor3 = COLORS.outline,
        BackgroundTransparency = 0.1,
        BorderSizePixel = 0,
        Visible = false,
        ZIndex = 801,
    }, RuntimeState.PerformanceWidget)
end

-- Prefer the file-backed key when auto-save is enabled. The executor environment
-- remains a fallback for older versions that only stored the key in memory.
local menuKeyName = type(SavedConfig.menuKey) == "string" and SavedConfig.menuKey or nil
if not menuKeyName and autoSaveEnabled then
    menuKeyName = RuntimeEnvironment.__CATScriptMenuKeyName
end
local menuKeyCode = Enum.KeyCode.RightShift
if type(menuKeyName) == "string" then
    local storedKey = Enum.KeyCode[menuKeyName]
    if storedKey then menuKeyCode = storedKey end
end
RuntimeState.KeybindButton.Text = menuKeyCode.Name
local waitingForKey = false

UserLabel.MouseEnter:Connect(function()
    tween(UserLabel, {TextColor3 = COLORS.accent}, 0.12)
end)
UserLabel.MouseLeave:Connect(function()
    tween(UserLabel, {TextColor3 = COLORS.text}, 0.12)
end)
UserLabel.MouseButton1Click:Connect(function()
    InfoDisplayName.Text = LocalPlayer.DisplayName
    InfoUsername.Text = "@" .. LocalPlayer.Name
    InfoExecutorValue.Text = getExecutorName()
    InfoAccountValue.Text = tostring(LocalPlayer.AccountAge) .. " days"
    InfoSessionValue.Text = formatSessionDuration()
    AboutUsOverlay.Visible = false
    InfoOverlay.Visible = true
end)
AboutUsButton.MouseEnter:Connect(function()
    tween(AboutUsButton, {TextColor3 = COLORS.accent}, 0.12)
end)
AboutUsButton.MouseLeave:Connect(function()
    tween(AboutUsButton, {TextColor3 = COLORS.muted}, 0.12)
end)
AboutUsButton.MouseButton1Click:Connect(function()
    InfoOverlay.Visible = false
    AboutUsOverlay.Visible = true
end)

-- Keep the session field live while the profile card is open.
task.spawn(function()
    while RuntimeState.active and ScreenGui.Parent do
        if InfoOverlay.Visible then
            InfoSessionValue.Text = formatSessionDuration()
        end
        task.wait(1)
    end
end)
InfoCloseButton.MouseButton1Click:Connect(function()
    InfoOverlay.Visible = false
end)
AboutUsCloseButton.MouseButton1Click:Connect(function()
    AboutUsOverlay.Visible = false
end)
AboutUsOverlay.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local mousePosition = UserInputService:GetMouseLocation()
        local cardPosition = AboutUsCard.AbsolutePosition
        local cardSize = AboutUsCard.AbsoluteSize
        local insideCard = mousePosition.X >= cardPosition.X
            and mousePosition.X <= cardPosition.X + cardSize.X
            and mousePosition.Y >= cardPosition.Y
            and mousePosition.Y <= cardPosition.Y + cardSize.Y
        if not insideCard then
            AboutUsOverlay.Visible = false
        end
    end
end)
-- The About Us panel is draggable by its header; the close button remains independent.
local aboutUsDragging = false
local aboutUsDragStart = nil
local aboutUsStartPosition = nil
AboutUsDragHandle.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        aboutUsDragging = true
        aboutUsDragStart = input.Position
        aboutUsStartPosition = AboutUsCard.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if not aboutUsDragging then return end
    if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
    local pointer = input.UserInputType == Enum.UserInputType.MouseMovement and UserInputService:GetMouseLocation() or input.Position
    local delta = pointer - aboutUsDragStart
    local x = aboutUsStartPosition.X.Offset + delta.X
    local y = aboutUsStartPosition.Y.Offset + delta.Y
    local parentSize = AboutUsOverlay.AbsoluteSize
    local cardSize = AboutUsCard.AbsoluteSize
    local maxX = math.max(0, (parentSize.X - cardSize.X) / 2)
    local maxY = math.max(0, (parentSize.Y - cardSize.Y) / 2)
    x = math.clamp(x, -maxX, maxX)
    y = math.clamp(y, -maxY, maxY)
    AboutUsCard.Position = UDim2.new(0.5, x, 0.5, y)
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        aboutUsDragging = false
        aboutUsDragStart = nil
        aboutUsStartPosition = nil
    end
end)
InfoOverlay.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local mousePosition = UserInputService:GetMouseLocation()
        local cardPosition = InfoCard.AbsolutePosition
        local cardSize = InfoCard.AbsoluteSize
        local insideCard = mousePosition.X >= cardPosition.X
            and mousePosition.X <= cardPosition.X + cardSize.X
            and mousePosition.Y >= cardPosition.Y
            and mousePosition.Y <= cardPosition.Y + cardSize.Y
        if not insideCard then
            InfoOverlay.Visible = false
        end
    end
end)

local function setMenuKey(keyCode)
    menuKeyCode = keyCode
    menuKeyName = keyCode.Name
    if autoSaveEnabled then
        RuntimeEnvironment.__CATScriptMenuKeyName = menuKeyName
    end
    RuntimeState.KeybindButton.Text = menuKeyName
    RuntimeState.KeybindStatus.Text = "MENU KEY SET"
    RuntimeState.KeybindStatus.TextColor3 = COLORS.accent
    if scheduleConfigSave then scheduleConfigSave() end
end

RuntimeState.KeybindButton.MouseButton1Click:Connect(function()
    waitingForKey = true
    RuntimeState.KeybindButton.Text = "Press a key..."
    RuntimeState.KeybindStatus.Text = "PRESS ANY KEY  /  ESC TO CANCEL"
    RuntimeState.KeybindStatus.TextColor3 = COLORS.accent
end)

-- KEYS HUD / WASD + MOUSE ----------------------------------------------------
do
    local savedPosition = type(SavedConfig.keystrokesPosition) == "table" and SavedConfig.keystrokesPosition or nil
    local widgetPosition = UDim2.new(1, -22, 1, -22)
    if savedPosition
        and type(savedPosition.xScale) == "number"
        and type(savedPosition.xOffset) == "number"
        and type(savedPosition.yScale) == "number"
        and type(savedPosition.yOffset) == "number" then
        widgetPosition = UDim2.new(
            math.clamp(savedPosition.xScale, -1, 2),
            math.clamp(savedPosition.xOffset, -4000, 4000),
            math.clamp(savedPosition.yScale, -1, 2),
            math.clamp(savedPosition.yOffset, -4000, 4000)
        )
    end

    RuntimeState.KeystrokesWidget = make("Frame", {
        Name = "CATScript_KeystrokesWidget", AnchorPoint = Vector2.new(1, 1),
        Position = widgetPosition, Size = UDim2.fromOffset(142, 110),
        BackgroundColor3 = COLORS.panel2, BackgroundTransparency = 0.12,
        BorderSizePixel = 0, Visible = RuntimeState.Keystrokes.enabled,
        Active = true, ZIndex = 80,
    }, ScreenGui)
    corner(RuntimeState.KeystrokesWidget, 8)
    stroke(RuntimeState.KeystrokesWidget, COLORS.outline, 0.2, 1)

    local function addKeyCell(key, text, x, y, width)
        local cell = make("TextLabel", {
            Name = "Key_" .. key, Position = UDim2.fromOffset(x, y),
            Size = UDim2.fromOffset(width or 28, 25),
            BackgroundColor3 = COLORS.background, BorderSizePixel = 0,
            Text = text, TextColor3 = COLORS.muted, TextSize = 8,
            Font = Enum.Font.GothamBold, Active = true, ZIndex = 81,
        }, RuntimeState.KeystrokesWidget)
        corner(cell, 5)
        RuntimeState.Keystrokes.cells[key] = cell
    end

    addKeyCell("W", "W", 37, 8, 28)
    addKeyCell("A", "A", 8, 37, 28)
    addKeyCell("S", "S", 37, 37, 28)
    addKeyCell("D", "D", 66, 37, 28)
    addKeyCell("Mouse1", "LMB", 8, 72, 56)
    addKeyCell("Mouse2", "RMB", 72, 72, 56)

    -- Move the complete keys HUD by dragging any empty area or key cell.
    RuntimeState.keystrokesDragging = false
    RuntimeState.keystrokesDragStart = nil
    RuntimeState.keystrokesStartPosition = nil
    RuntimeState.beginKeystrokesDrag = function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            RuntimeState.keystrokesDragging = true
            RuntimeState.keystrokesDragStart = input.Position
            RuntimeState.keystrokesStartPosition = RuntimeState.KeystrokesWidget.Position
        end
    end

    RuntimeState.KeystrokesWidget.InputBegan:Connect(RuntimeState.beginKeystrokesDrag)
    for _, cell in pairs(RuntimeState.Keystrokes.cells) do
        cell.InputBegan:Connect(RuntimeState.beginKeystrokesDrag)
    end

    RuntimeState.keystrokesDragChangedConnection = UserInputService.InputChanged:Connect(function(input)
        if not RuntimeState.keystrokesDragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
            and input.UserInputType ~= Enum.UserInputType.Touch then return end

        local pointer = input.UserInputType == Enum.UserInputType.Touch
            and input.Position or UserInputService:GetMouseLocation()
        local delta = pointer - RuntimeState.keystrokesDragStart
        local camera = workspace.CurrentCamera
        local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
        local widget = RuntimeState.KeystrokesWidget
        local width = math.min(widget.AbsoluteSize.X, view.X)
        local height = math.min(widget.AbsoluteSize.Y, view.Y)
        local start = RuntimeState.keystrokesStartPosition
        local anchorX = start.X.Scale * view.X + start.X.Offset + delta.X
        local anchorY = start.Y.Scale * view.Y + start.Y.Offset + delta.Y
        anchorX = math.clamp(anchorX, width, view.X)
        anchorY = math.clamp(anchorY, height, view.Y)
        widget.Position = UDim2.fromOffset(anchorX, anchorY)
    end)

    RuntimeState.keystrokesDragEndedConnection = UserInputService.InputEnded:Connect(function(input)
        if RuntimeState.keystrokesDragging and (input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch) then
            RuntimeState.keystrokesDragging = false
            RuntimeState.keystrokesDragStart = nil
            RuntimeState.keystrokesStartPosition = nil
            if scheduleConfigSave then scheduleConfigSave() end
        end
    end)
end
RuntimeState.refreshKeystrokes=function()
    for key,cell in pairs(RuntimeState.Keystrokes.cells) do
        local pressed=RuntimeState.Keystrokes.keys[key]==true
        cell.BackgroundColor3=pressed and COLORS.accent or COLORS.background
        cell.TextColor3=pressed and COLORS.white or COLORS.muted
    end
end
RuntimeState.setKeystrokes=function(enabled)
    RuntimeState.Keystrokes.enabled=enabled==true
    RuntimeState.KeystrokesWidget.Visible=RuntimeState.Keystrokes.enabled
    if not RuntimeState.Keystrokes.enabled then table.clear(RuntimeState.Keystrokes.keys); RuntimeState.refreshKeystrokes() end
end
RuntimeState.Keystrokes.connections.began=UserInputService.InputBegan:Connect(function(input)
    local key=input.KeyCode.Name
    if RuntimeState.Keystrokes.cells[key] then RuntimeState.Keystrokes.keys[key]=true end
    if input.UserInputType==Enum.UserInputType.MouseButton1 then RuntimeState.Keystrokes.keys.Mouse1=true end
    if input.UserInputType==Enum.UserInputType.MouseButton2 then RuntimeState.Keystrokes.keys.Mouse2=true end
    RuntimeState.refreshKeystrokes()
end)
RuntimeState.Keystrokes.connections.ended=UserInputService.InputEnded:Connect(function(input)
    RuntimeState.Keystrokes.keys[input.KeyCode.Name]=nil
    if input.UserInputType==Enum.UserInputType.MouseButton1 then RuntimeState.Keystrokes.keys.Mouse1=nil end
    if input.UserInputType==Enum.UserInputType.MouseButton2 then RuntimeState.Keystrokes.keys.Mouse2=nil end
    RuntimeState.refreshKeystrokes()
end)

-- Page switching swaps Visual, Player and Settings inside the shared dashboard.
local currentPage = "Visual"

-- Fully configurable local-player ESP page. ESP colors load/save independently.
RuntimeState.loadESPColor = function(key, fallback)
    local savedColors = type(SavedConfig.espColors) == "table" and SavedConfig.espColors or {}
    local saved = type(savedColors[key]) == "table" and savedColors[key] or nil
    if not saved then return fallback end
    local r, g, b = tonumber(saved.r), tonumber(saved.g), tonumber(saved.b)
    if not r or not g or not b then return fallback end
    return Color3.fromRGB(math.clamp(math.floor(r + 0.5), 0, 255), math.clamp(math.floor(g + 0.5), 0, 255), math.clamp(math.floor(b + 0.5), 0, 255))
end
local SavedESPBoxStyle = type(SavedConfig.espCornerBox) == "table" and SavedConfig.espCornerBox or {}
RuntimeState.ESPConfig = {
    enabled = false, names = true, distance = true, health = true,
    highlight = true, maxDistance = 2500,
    cornerBox = SavedESPBoxStyle.enabled ~= false,
    cornerBoxWidthScale = math.clamp(tonumber(SavedESPBoxStyle.widthScale) or 1, 0.5, 1.6),
    cornerBoxHeightScale = math.clamp(tonumber(SavedESPBoxStyle.heightScale) or 1, 0.5, 1.6),
    cornerBoxLength = math.clamp(tonumber(SavedESPBoxStyle.length) or 0.22, 0.08, 0.45),
    cornerBoxThickness = math.clamp(tonumber(SavedESPBoxStyle.thickness) or 2.5, 1, 8),
    cornerBoxOpacity = math.clamp(tonumber(SavedESPBoxStyle.opacity) or 0.04, 0, 0.85),
    cornerBoxGlow = math.clamp(tonumber(SavedESPBoxStyle.glow) or 0.82, 0, 1),
    cornerBoxPadding = math.clamp(tonumber(SavedESPBoxStyle.padding) or 2, 0, 16),
    outlineColor = RuntimeState.loadESPColor("outlineColor", Color3.fromRGB(190, 120, 255)),
    fillColor = RuntimeState.loadESPColor("fillColor", Color3.fromRGB(145, 70, 220)),
    textColor = RuntimeState.loadESPColor("textColor", Color3.fromRGB(255, 255, 255)),
    healthColor = RuntimeState.loadESPColor("healthColor", Color3.fromRGB(80, 235, 130)),
    cornerBoxColor = RuntimeState.loadESPColor("cornerBoxColor", Color3.fromRGB(205, 229, 255)),
    cornerBoxGlowColor = RuntimeState.loadESPColor("cornerBoxGlowColor", Color3.fromRGB(105, 174, 255)),
}
RuntimeState.ESPObjects = {}
RuntimeState.ESPCard = make("Frame", {
    Name = "ESPCard", BackgroundColor3 = COLORS.panel, BackgroundTransparency = 0.24,
    BorderSizePixel = 0, Size = UDim2.new(1, 0, 0, 770), Visible = false,
}, RuntimeState.Content)
corner(RuntimeState.ESPCard, 10)
stroke(RuntimeState.ESPCard, COLORS.outline, 0, 1)
make("UIPadding", {
    PaddingTop = UDim.new(0, 14), PaddingBottom = UDim.new(0, 12),
    PaddingLeft = UDim.new(0, 16), PaddingRight = UDim.new(0, 16),
}, RuntimeState.ESPCard)
local espTitle = addLabel(RuntimeState.ESPCard, "ESP", 12, COLORS.text, Enum.Font.GothamBold)
espTitle.Size = UDim2.new(1, 0, 0, 18)
local espSubtitle = addLabel(RuntimeState.ESPCard, "Player overlay • customizable colors", 9, COLORS.muted, Enum.Font.Gotham)
espSubtitle.Position = UDim2.fromOffset(0, 19)
espSubtitle.Size = UDim2.new(1, 0, 0, 15)

local function espToggleRow(key, title, y, configKey)
    local row = make("Frame", {
        Name = key .. "Row", Position = UDim2.fromOffset(0, y), Size = UDim2.new(1, 0, 0, 38),
        BackgroundColor3 = COLORS.panel2, BackgroundTransparency = 0.18, BorderSizePixel = 0,
    }, RuntimeState.ESPCard)
    corner(row, 8); stroke(row, COLORS.outline, 0.35, 1)
    local label = addLabel(row, title, 9, COLORS.text, Enum.Font.GothamSemibold)
    label.Position = UDim2.fromOffset(12, 0); label.Size = UDim2.new(1, -80, 1, 0)
    local button = make("TextButton", {
        Position = UDim2.new(1, -58, 0.5, -11), Size = UDim2.fromOffset(44, 22),
        Text = "", AutoButtonColor = false, BackgroundColor3 = COLORS.dim, BorderSizePixel = 0,
    }, row)
    corner(button, 11)
    local knob = make("Frame", {
        Position = UDim2.fromOffset(3, 3), Size = UDim2.fromOffset(16, 16),
        BackgroundColor3 = COLORS.white, BorderSizePixel = 0,
    }, button)
    corner(knob, 8)
    local function refresh()
        local on = RuntimeState.ESPConfig[configKey] == true
        button.BackgroundColor3 = on and COLORS.accent or COLORS.dim
        knob.Position = on and UDim2.new(1, -19, 0, 3) or UDim2.fromOffset(3, 3)
    end
    button.MouseButton1Click:Connect(function()
        RuntimeState.ESPConfig[configKey] = not RuntimeState.ESPConfig[configKey]
        refresh()
        if scheduleConfigSave then scheduleConfigSave() end
    end)
    refresh()
end

RuntimeState.ESPColorButtons = {}
local function espColorRow(title, y, configKey)
    local row = make("Frame", {
        Name = configKey .. "ColorRow", Position = UDim2.fromOffset(0, y),
        Size = UDim2.new(1, 0, 0, 48), BackgroundColor3 = COLORS.panel2,
        BackgroundTransparency = 0.18, BorderSizePixel = 0,
    }, RuntimeState.ESPCard)
    corner(row, 8); stroke(row, COLORS.outline, 0.35, 1)
    local label = addLabel(row, title, 8, COLORS.text, Enum.Font.GothamSemibold)
    label.Position = UDim2.fromOffset(12, 0); label.Size = UDim2.new(1, -100, 1, 0)
    local pickerButton = make("TextButton", {
        Name = configKey .. "ChooseColor",
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -10, 0.5, 0),
        Size = UDim2.fromOffset(72, 27),
        BackgroundColor3 = RuntimeState.ESPConfig[configKey],
        BorderSizePixel = 0, Text = "CHOOSE",
        TextColor3 = COLORS.white, TextSize = 8, Font = Enum.Font.GothamBold,
        AutoButtonColor = false,
    }, row)
    corner(pickerButton, 6); stroke(pickerButton, COLORS.outline, 0.15, 1)
    RuntimeState.ESPColorButtons[configKey] = pickerButton
    pickerButton.MouseButton1Click:Connect(function()
        if RuntimeState.openESPColorEditor then RuntimeState.openESPColorEditor(configKey) end
    end)
end

espToggleRow("ESPEnabled", "Enable ESP", 46, "enabled")
espToggleRow("ESPNames", "Player names", 88, "names")
espToggleRow("ESPDistance", "Distance", 130, "distance")
espToggleRow("ESPHealth", "Health bar", 172, "health")
espToggleRow("ESPHighlight", "Character highlight", 214, "highlight")
espToggleRow("ESPCornerBox", "Reference corner box", 256, "cornerBox")
espColorRow("Outline / highlight color", 300, "outlineColor")
espColorRow("Fill color", 352, "fillColor")
espColorRow("Text color", 404, "textColor")
espColorRow("Health color", 456, "healthColor")
espColorRow("Corner box core color", 508, "cornerBoxColor")
espColorRow("Corner box glow color", 560, "cornerBoxGlowColor")

-- One independent full-range HSV picker serves all ESP color channels, including the corner core and glow.
-- Colors are selected visually; HEX/RGB are shown only as read-only precision values.
RuntimeState.ESPColorEditor = make("Frame", {
    Name = "CATScriptESPColorEditor",
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.fromOffset(276, 232),
    BackgroundColor3 = COLORS.panel2, BorderSizePixel = 0,
    Visible = false, ZIndex = 410,
}, RuntimeState.floatingEditorsLayer)
corner(RuntimeState.ESPColorEditor, 8)
stroke(RuntimeState.ESPColorEditor, COLORS.outline, 0.25, 1)
RuntimeState.ESPColorEditorTitle = addLabel(RuntimeState.ESPColorEditor, "ESP COLOR", 8, COLORS.muted, Enum.Font.GothamBold)
RuntimeState.ESPColorEditorTitle.Position = UDim2.fromOffset(12, 6)
RuntimeState.ESPColorEditorTitle.Size = UDim2.new(1, -24, 0, 17)
RuntimeState.ESPColorHSV = {h = 0.72, s = 0.52, v = 1}
RuntimeState.ESPColorSVArea = make("Frame", {
    Name = "ESPColorSVArea", Position = UDim2.fromOffset(12, 29),
    Size = UDim2.fromOffset(205, 158), BackgroundColor3 = Color3.fromHSV(0.72, 1, 1),
    BorderSizePixel = 0, ClipsDescendants = true,
}, RuntimeState.ESPColorEditor)
corner(RuntimeState.ESPColorSVArea, 3)
RuntimeState.ESPColorSVWhite = make("Frame", {Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0}, RuntimeState.ESPColorSVArea)
make("UIGradient", {Rotation = 0, Color = ColorSequence.new(Color3.new(1, 1, 1)), Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(1, 1)})}, RuntimeState.ESPColorSVWhite)
RuntimeState.ESPColorSVBlack = make("Frame", {Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0}, RuntimeState.ESPColorSVArea)
make("UIGradient", {Rotation = 90, Color = ColorSequence.new(Color3.new(0, 0, 0)), Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0)})}, RuntimeState.ESPColorSVBlack)
RuntimeState.ESPColorSVInput = make("TextButton", {
    Name = "ESPColorSVInput", Text = "", Position = UDim2.fromScale(0, 0),
    Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
    AutoButtonColor = false, ZIndex = 10,
}, RuntimeState.ESPColorSVArea)
RuntimeState.ESPColorSVMarker = make("Frame", {
    Name = "ESPColorSVMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(RuntimeState.ESPColorHSV.s, 1 - RuntimeState.ESPColorHSV.v),
    Size = UDim2.fromOffset(11, 11), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.ESPColorSVArea)
corner(RuntimeState.ESPColorSVMarker, 8); stroke(RuntimeState.ESPColorSVMarker, Color3.new(1, 1, 1), 0, 2)
RuntimeState.ESPColorHueInput = make("TextButton", {
    Name = "ESPColorHueInput", Position = UDim2.fromOffset(224, 29),
    Size = UDim2.fromOffset(18, 158), Text = "", BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, AutoButtonColor = false, ClipsDescendants = true,
}, RuntimeState.ESPColorEditor)
corner(RuntimeState.ESPColorHueInput, 3)
RuntimeState.ESPColorHueGradient = make("Frame", {Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(1, 1, 1), BorderSizePixel = 0}, RuntimeState.ESPColorHueInput)
make("UIGradient", {Rotation = 90, Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 0)),
    ColorSequenceKeypoint.new(0.17, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(0.33, Color3.fromRGB(0, 255, 0)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 255, 255)),
    ColorSequenceKeypoint.new(0.67, Color3.fromRGB(0, 0, 255)),
    ColorSequenceKeypoint.new(0.83, Color3.fromRGB(255, 0, 255)),
    ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 0)),
})}, RuntimeState.ESPColorHueGradient)
RuntimeState.ESPColorHueMarker = make("Frame", {
    Name = "ESPColorHueMarker", AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.new(0.5, 0, RuntimeState.ESPColorHSV.h, 0),
    Size = UDim2.new(1, 4, 0, 3), BackgroundColor3 = Color3.new(1, 1, 1),
    BorderSizePixel = 0, ZIndex = 12,
}, RuntimeState.ESPColorHueInput)
stroke(RuntimeState.ESPColorHueMarker, Color3.fromRGB(30, 30, 30), 0, 1)
RuntimeState.ESPColorPreview = make("Frame", {
    Name = "ESPColorPreview", Position = UDim2.fromOffset(12, 198),
    Size = UDim2.fromOffset(28, 24), BackgroundColor3 = Color3.fromRGB(190, 120, 255),
    BorderSizePixel = 0,
}, RuntimeState.ESPColorEditor)
corner(RuntimeState.ESPColorPreview, 5)
RuntimeState.ESPColorReadout = addLabel(RuntimeState.ESPColorEditor, "#BE78FF  •  190, 120, 255", 8, COLORS.text, Enum.Font.Code)
RuntimeState.ESPColorReadout.Position = UDim2.fromOffset(50, 198)
RuntimeState.ESPColorReadout.Size = UDim2.new(1, -62, 0, 24)

RuntimeState.ESPColorEditorTitle.Text = "ESP COLOR"
RuntimeState.ESPColorDragMode = nil
RuntimeState.refreshESPColorEditor = function()
    local key = RuntimeState.activeESPColorKey
    if not key then return end
    local color = RuntimeState.ESPConfig[key]
    local h, sat, val = color:ToHSV()
    RuntimeState.ESPColorHSV.h, RuntimeState.ESPColorHSV.s, RuntimeState.ESPColorHSV.v = h, sat, val
    RuntimeState.ESPColorSVArea.BackgroundColor3 = Color3.fromHSV(h, 1, 1)
    RuntimeState.ESPColorSVMarker.Position = UDim2.fromScale(sat, 1 - val)
    RuntimeState.ESPColorHueMarker.Position = UDim2.new(0.5, 0, h, 0)
    RuntimeState.ESPColorPreview.BackgroundColor3 = color
    RuntimeState.ESPColorReadout.Text = string.format("#%02X%02X%02X  •  %d, %d, %d",
        math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5),
        math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5))
    local button = RuntimeState.ESPColorButtons[key]
    if button then button.BackgroundColor3 = color end
end
RuntimeState.applyESPColorFromHSV = function()
    local key = RuntimeState.activeESPColorKey
    if not key then return end
    local color = Color3.fromHSV(RuntimeState.ESPColorHSV.h, RuntimeState.ESPColorHSV.s, RuntimeState.ESPColorHSV.v)
    RuntimeState.ESPConfig[key] = color
    RuntimeState.ESPColorSVArea.BackgroundColor3 = Color3.fromHSV(RuntimeState.ESPColorHSV.h, 1, 1)
    RuntimeState.ESPColorSVMarker.Position = UDim2.fromScale(RuntimeState.ESPColorHSV.s, 1 - RuntimeState.ESPColorHSV.v)
    RuntimeState.ESPColorHueMarker.Position = UDim2.new(0.5, 0, RuntimeState.ESPColorHSV.h, 0)
    RuntimeState.ESPColorPreview.BackgroundColor3 = color
    RuntimeState.ESPColorReadout.Text = string.format("#%02X%02X%02X  •  %d, %d, %d",
        math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5),
        math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5))
    local button = RuntimeState.ESPColorButtons[key]
    if button then button.BackgroundColor3 = color end
    if scheduleConfigSave then scheduleConfigSave() end
end
RuntimeState.updateESPColorFromSV = function(pointer)
    local origin, size = RuntimeState.ESPColorSVArea.AbsolutePosition, RuntimeState.ESPColorSVArea.AbsoluteSize
    if size.X <= 0 or size.Y <= 0 then return end
    RuntimeState.ESPColorHSV.s = math.clamp((pointer.X - origin.X) / size.X, 0, 1)
    RuntimeState.ESPColorHSV.v = 1 - math.clamp((pointer.Y - origin.Y) / size.Y, 0, 1)
    RuntimeState.applyESPColorFromHSV()
end
RuntimeState.updateESPColorHue = function(pointer)
    local origin, size = RuntimeState.ESPColorHueInput.AbsolutePosition, RuntimeState.ESPColorHueInput.AbsoluteSize
    if size.Y <= 0 then return end
    RuntimeState.ESPColorHSV.h = math.clamp((pointer.Y - origin.Y) / size.Y, 0, 1)
    RuntimeState.applyESPColorFromHSV()
end
RuntimeState.openESPColorEditor = function(configKey)
    if typeof(RuntimeState.ESPConfig[configKey]) ~= "Color3" then return end
    local sameSelection = RuntimeState.activeESPColorKey == configKey
    RuntimeState.activeESPColorKey = configKey
    local titles = {outlineColor = "OUTLINE COLOR", fillColor = "FILL COLOR", textColor = "TEXT COLOR", healthColor = "HEALTH COLOR", cornerBoxColor = "CORNER BOX CORE COLOR", cornerBoxGlowColor = "CORNER BOX GLOW COLOR"}
    RuntimeState.ESPColorEditorTitle.Text = titles[configKey] or "ESP COLOR"
    RuntimeState.refreshESPColorEditor()
    if RuntimeState.ESPColorEditor.Visible then
        if sameSelection then RuntimeState.ESPColorEditor.Visible = false end
    else
        RuntimeState.openFloatingColorEditor(RuntimeState.ESPColorEditor, 15)
    end
end
RuntimeState.ESPColorSVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.ESPColorDragMode = "sv"
        RuntimeState.updateESPColorFromSV(getColorEditorPointer(input))
    end
end)
RuntimeState.ESPColorHueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.ESPColorDragMode = "hue"
        RuntimeState.updateESPColorHue(getColorEditorPointer(input))
    end
end)
RuntimeState.ESPColorInputChanged = UserInputService.InputChanged:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
    if not RuntimeState.ESPColorDragMode then return end
    local pointer = getColorEditorPointer(input)
    if RuntimeState.ESPColorDragMode == "sv" then
        RuntimeState.updateESPColorFromSV(pointer)
    elseif RuntimeState.ESPColorDragMode == "hue" then
        RuntimeState.updateESPColorHue(pointer)
    end
end)
RuntimeState.ESPColorInputEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.ESPColorDragMode = nil
    end
end)
RuntimeState.configureFloatingColorEditor(RuntimeState.ESPColorEditor, RuntimeState.ESPColorEditorTitle, 15)

-- Dedicated editor for the glowing L-shaped corner box shown in the user's reference.
-- Every value is independent and persisted in the CATScript config.
do
    local editor = make("Frame", {
        Name = "CATScriptESPCornerBoxEditor",
        Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(366, 412),
        BackgroundColor3 = COLORS.panel2,
        BorderSizePixel = 0,
        Visible = false,
        ZIndex = 430,
    }, RuntimeState.floatingEditorsLayer)
    corner(editor, 9)
    stroke(editor, COLORS.outline, 0.2, 1)
    local title = addLabel(editor, "CORNER BOX CUSTOMIZER", 9, COLORS.text, Enum.Font.GothamBold)
    title.Position = UDim2.fromOffset(12, 6)
    title.Size = UDim2.new(1, -24, 0, 18)
    title.ZIndex = 431
    RuntimeState.ESPCornerBoxEditor = editor
    RuntimeState.ESPCornerBoxSliders = {}

    local sliderDefinitions = {
        {key="cornerBoxWidthScale", label="WIDTH", min=0.5, max=1.6, step=0.01, suffix="%", factor=100},
        {key="cornerBoxHeightScale", label="HEIGHT", min=0.5, max=1.6, step=0.01, suffix="%", factor=100},
        {key="cornerBoxLength", label="CORNER LENGTH", min=0.08, max=0.45, step=0.01, suffix="%", factor=100},
        {key="cornerBoxThickness", label="LINE THICKNESS", min=1, max=8, step=0.5, suffix=" px", factor=1},
        {key="cornerBoxOpacity", label="CORE TRANSPARENCY", min=0, max=0.85, step=0.01, suffix="%", factor=100},
        {key="cornerBoxGlow", label="GLOW INTENSITY", min=0, max=1, step=0.01, suffix="%", factor=100},
        {key="cornerBoxPadding", label="BOX PADDING", min=0, max=16, step=0.5, suffix=" px", factor=1},
    }

    local function refreshCornerSlider(slider)
        local config = RuntimeState.ESPConfig
        local value = config[slider.key]
        local alpha = math.clamp((value - slider.min) / (slider.max - slider.min), 0, 1)
        slider.fill.Size = UDim2.new(alpha, 0, 1, 0)
        slider.knob.Position = UDim2.new(alpha, -7, 0.5, -7)
        slider.value.Text = string.format("%g%s", value * slider.factor, slider.suffix)
    end

    for index, definition in ipairs(sliderDefinitions) do
        local y = 31 + (index - 1) * 46
        local label = addLabel(editor, definition.label, 7, COLORS.muted, Enum.Font.GothamBold)
        label.Position = UDim2.fromOffset(12, y)
        label.Size = UDim2.new(0.68, -12, 0, 14)
        label.ZIndex = 431
        local valueLabel = addLabel(editor, "", 7, COLORS.text, Enum.Font.GothamBold, Enum.TextXAlignment.Right)
        valueLabel.Position = UDim2.new(0.68, 0, 0, y)
        valueLabel.Size = UDim2.new(0.28, 0, 0, 14)
        valueLabel.ZIndex = 431
        local track = make("TextButton", {
            Name = "Slider_" .. definition.key,
            Text = "", AutoButtonColor = false,
            Position = UDim2.fromOffset(12, y + 22),
            Size = UDim2.new(1, -24, 0, 8),
            BackgroundColor3 = COLORS.background,
            BorderSizePixel = 0,
            ZIndex = 432,
        }, editor)
        corner(track, 5)
        local fill = make("Frame", {
            Size = UDim2.new(0, 0, 1, 0),
            BackgroundColor3 = COLORS.accent,
            BorderSizePixel = 0,
            ZIndex = 433,
        }, track)
        corner(fill, 5)
        local knob = make("Frame", {
            AnchorPoint = Vector2.new(0, 0.5),
            Position = UDim2.fromOffset(0, 4),
            Size = UDim2.fromOffset(14, 14),
            BackgroundColor3 = COLORS.white,
            BorderSizePixel = 0,
            ZIndex = 434,
        }, track)
        corner(knob, 7)
        stroke(knob, COLORS.accent, 0.1, 1)
        local slider = {
            key=definition.key, min=definition.min, max=definition.max,
            step=definition.step, factor=definition.factor, suffix=definition.suffix,
            track=track, fill=fill, knob=knob, value=valueLabel,
        }
        RuntimeState.ESPCornerBoxSliders[definition.key] = slider
        refreshCornerSlider(slider)
        track.InputBegan:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
            RuntimeState.ESPCornerBoxDragging = definition.key
            local point = input.UserInputType == Enum.UserInputType.Touch and input.Position or UserInputService:GetMouseLocation()
            local origin, size = track.AbsolutePosition, track.AbsoluteSize
            if size.X > 0 then
                local alpha = math.clamp((point.X - origin.X) / size.X, 0, 1)
                local raw = definition.min + alpha * (definition.max - definition.min)
                local snapped = definition.min + math.floor((raw - definition.min) / definition.step + 0.5) * definition.step
                RuntimeState.ESPConfig[definition.key] = math.clamp(snapped, definition.min, definition.max)
                refreshCornerSlider(slider)
                if scheduleConfigSave then scheduleConfigSave() end
            end
        end)
    end

    RuntimeState.ESPCornerBoxInputChanged = UserInputService.InputChanged:Connect(function(input)
        if not RuntimeState.ESPCornerBoxDragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local slider = RuntimeState.ESPCornerBoxSliders[RuntimeState.ESPCornerBoxDragging]
        if not slider then return end
        local point = input.UserInputType == Enum.UserInputType.Touch and input.Position or UserInputService:GetMouseLocation()
        local origin, size = slider.track.AbsolutePosition, slider.track.AbsoluteSize
        if size.X <= 0 then return end
        local alpha = math.clamp((point.X - origin.X) / size.X, 0, 1)
        local raw = slider.min + alpha * (slider.max - slider.min)
        local snapped = slider.min + math.floor((raw - slider.min) / slider.step + 0.5) * slider.step
        RuntimeState.ESPConfig[slider.key] = math.clamp(snapped, slider.min, slider.max)
        refreshCornerSlider(slider)
        if scheduleConfigSave then scheduleConfigSave() end
    end)
    RuntimeState.ESPCornerBoxInputEnded = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            RuntimeState.ESPCornerBoxDragging = nil
        end
    end)

    local reset = make("TextButton", {
        Name = "ResetCornerBoxStyle", Text = "RESET STYLE", TextColor3 = COLORS.text,
        TextSize = 8, Font = Enum.Font.GothamBold, AutoButtonColor = false,
        Position = UDim2.new(0, 12, 1, -31), Size = UDim2.new(1, -24, 0, 22),
        BackgroundColor3 = COLORS.background, BorderSizePixel = 0, ZIndex = 432,
    }, editor)
    corner(reset, 5)
    reset.MouseButton1Click:Connect(function()
        local defaults = {
            cornerBoxWidthScale=1, cornerBoxHeightScale=1, cornerBoxLength=0.22,
            cornerBoxThickness=2.5, cornerBoxOpacity=0.04, cornerBoxGlow=0.82, cornerBoxPadding=2,
        }
        for key, value in pairs(defaults) do
            RuntimeState.ESPConfig[key] = value
            local slider = RuntimeState.ESPCornerBoxSliders[key]
            if slider then refreshCornerSlider(slider) end
        end
        if scheduleConfigSave then scheduleConfigSave() end
    end)

    RuntimeState.ESPCornerBoxOpenButton = make("TextButton", {
        Name = "OpenCornerBoxCustomizer", Text = "EDIT CORNER BOX STYLE  ⚙",
        TextColor3 = COLORS.text, TextSize = 8, Font = Enum.Font.GothamBold,
        AutoButtonColor = false, Position = UDim2.fromOffset(0, 710),
        Size = UDim2.new(1, 0, 0, 34), BackgroundColor3 = COLORS.panel2,
        BorderSizePixel = 0,
    }, RuntimeState.ESPCard)
    corner(RuntimeState.ESPCornerBoxOpenButton, 7)
    stroke(RuntimeState.ESPCornerBoxOpenButton, COLORS.outline, 0.3, 1)
    RuntimeState.ESPCornerBoxOpenButton.MouseButton1Click:Connect(function()
        RuntimeState.openFloatingColorEditor(editor, 16)
    end)
    RuntimeState.configureFloatingColorEditor(editor, title, 16)
end

local distanceRow = make("Frame", {
    Name = "ESPMaxDistanceRow", Position = UDim2.fromOffset(0, 612),
    Size = UDim2.new(1, 0, 0, 48), BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18, BorderSizePixel = 0,
}, RuntimeState.ESPCard)
corner(distanceRow, 8); stroke(distanceRow, COLORS.outline, 0.35, 1)
local distanceLabel = addLabel(distanceRow, "Max distance (studs)", 8, COLORS.text, Enum.Font.GothamSemibold)
distanceLabel.Position = UDim2.fromOffset(12, 0); distanceLabel.Size = UDim2.new(0.55, -12, 1, 0)
local distanceInput = make("TextBox", {
    Position = UDim2.new(0.68, 0, 0.5, -13), Size = UDim2.new(0.28, 0, 0, 26),
    BackgroundColor3 = COLORS.background, BorderSizePixel = 0, Text = "2500",
    TextColor3 = COLORS.text, TextSize = 9, Font = Enum.Font.GothamBold,
    ClearTextOnFocus = false,
}, distanceRow)
corner(distanceInput, 6); stroke(distanceInput, COLORS.outline, 0.25, 1)
local function applyESPDistance()
    local value = tonumber(distanceInput.Text)
    if value then
        RuntimeState.ESPConfig.maxDistance = math.clamp(math.floor(value), 50, 10000)
        distanceInput.Text = tostring(RuntimeState.ESPConfig.maxDistance)
    else
        distanceInput.Text = tostring(RuntimeState.ESPConfig.maxDistance)
    end
end
distanceInput.FocusLost:Connect(function() applyESPDistance(); if scheduleConfigSave then scheduleConfigSave() end end)

local function clearESPForPlayer(player)
    local entry = RuntimeState.ESPObjects[player]
    if entry then
        if entry.highlight then entry.highlight:Destroy() end
        if entry.billboard then entry.billboard:Destroy() end
        if entry.cornerBox and entry.cornerBox.root then entry.cornerBox.root:Destroy() end
        RuntimeState.ESPObjects[player] = nil
    end
end

local function createCornerBoxOverlay(player)
    local overlay = make("Frame", {
        Name = "CATScript_ESPCornerBox_" .. tostring(player.UserId),
        BackgroundTransparency = 1, BorderSizePixel = 0,
        Position = UDim2.fromScale(0, 0), Size = UDim2.fromScale(1, 1),
        Visible = false, Active = false, ZIndex = 620,
    }, ScreenGui)
    local segments = {}
    for index = 1, 8 do
        local outer = make("Frame", {
            Name = "GlowOuter" .. index, BackgroundColor3 = Color3.fromRGB(205, 229, 255),
            BackgroundTransparency = 0.72, BorderSizePixel = 0, ZIndex = 620,
        }, overlay)
        corner(outer, 3)
        local inner = make("Frame", {
            Name = "GlowInner" .. index, BackgroundColor3 = Color3.fromRGB(205, 229, 255),
            BackgroundTransparency = 0.48, BorderSizePixel = 0, ZIndex = 621,
        }, overlay)
        corner(inner, 2)
        local core = make("Frame", {
            Name = "Core" .. index, BackgroundColor3 = Color3.fromRGB(205, 229, 255),
            BackgroundTransparency = 0.04, BorderSizePixel = 0, ZIndex = 622,
        }, overlay)
        corner(core, 2)
        segments[index] = {outer=outer, inner=inner, core=core}
    end
    return {root=overlay, segments=segments}
end

local function setCornerLayer(gui, x, y, width, height, expand)
    gui.Position = UDim2.fromOffset(x - expand, y - expand)
    gui.Size = UDim2.fromOffset(math.max(1, width + expand * 2), math.max(1, height + expand * 2))
end

local function updateCornerBoxForPlayer(entry, player, character, camera)
    local config = RuntimeState.ESPConfig
    if not config.cornerBox then
        if entry.cornerBox and entry.cornerBox.root then entry.cornerBox.root.Visible = false end
        return
    end
    if not entry.cornerBox or not entry.cornerBox.root or not entry.cornerBox.root.Parent then
        entry.cornerBox = createCornerBoxOverlay(player)
    end
    local ok, boundsCFrame, boundsSize = pcall(function() return character:GetBoundingBox() end)
    if not ok or not boundsCFrame or not boundsSize then
        entry.cornerBox.root.Visible = false
        return
    end

    local minX, minY = math.huge, math.huge
    local maxX, maxY = -math.huge, -math.huge
    local frontCount = 0
    for _, xSign in ipairs({-1, 1}) do
        for _, ySign in ipairs({-1, 1}) do
            for _, zSign in ipairs({-1, 1}) do
                local world = boundsCFrame:PointToWorldSpace(Vector3.new(
                    boundsSize.X * 0.5 * xSign,
                    boundsSize.Y * 0.5 * ySign,
                    boundsSize.Z * 0.5 * zSign
                ))
                local point, onScreen = camera:WorldToViewportPoint(world)
                if point.Z > 0 then
                    frontCount += 1
                    minX = math.min(minX, point.X)
                    minY = math.min(minY, point.Y)
                    maxX = math.max(maxX, point.X)
                    maxY = math.max(maxY, point.Y)
                end
            end
        end
    end

    local viewport = camera.ViewportSize
    if frontCount == 0 or maxX < 0 or maxY < 0 or minX > viewport.X or minY > viewport.Y then
        entry.cornerBox.root.Visible = false
        return
    end
    minX = math.clamp(minX, 0, viewport.X)
    minY = math.clamp(minY, 0, viewport.Y)
    maxX = math.clamp(maxX, 0, viewport.X)
    maxY = math.clamp(maxY, 0, viewport.Y)

    local centerX, centerY = (minX + maxX) * 0.5, (minY + maxY) * 0.5
    local width = (maxX - minX) * config.cornerBoxWidthScale
    local height = (maxY - minY) * config.cornerBoxHeightScale
    local pad = config.cornerBoxPadding
    width += pad * 2
    height += pad * 2
    minX, maxX = centerX - width * 0.5, centerX + width * 0.5
    minY, maxY = centerY - height * 0.5, centerY + height * 0.5

    local shortSide = math.max(8, math.min(width, height))
    local cornerLength = math.clamp(shortSide * config.cornerBoxLength, 5, shortSide * 0.48)
    local thickness = config.cornerBoxThickness
    local color = config.cornerBoxColor
    local glow = config.cornerBoxGlow
    local segments = entry.cornerBox.segments
    local rects = {
        {minX, minY, cornerLength, thickness},
        {minX, minY, thickness, cornerLength},
        {maxX - cornerLength, minY, cornerLength, thickness},
        {maxX - thickness, minY, thickness, cornerLength},
        {minX, maxY - thickness, cornerLength, thickness},
        {minX, maxY - cornerLength, thickness, cornerLength},
        {maxX - cornerLength, maxY - thickness, cornerLength, thickness},
        {maxX - thickness, maxY - cornerLength, thickness, cornerLength},
    }
    for index, rect in ipairs(rects) do
        local layers = segments[index]
        local x, y, w, h = rect[1], rect[2], rect[3], rect[4]
        setCornerLayer(layers.outer, x, y, w, h, thickness * 1.9)
        setCornerLayer(layers.inner, x, y, w, h, thickness * 0.85)
        setCornerLayer(layers.core, x, y, w, h, 0)
        layers.outer.BackgroundColor3 = config.cornerBoxGlowColor
        layers.inner.BackgroundColor3 = config.cornerBoxGlowColor
        layers.core.BackgroundColor3 = color
        layers.outer.BackgroundTransparency = 1 - glow * 0.34
        layers.inner.BackgroundTransparency = 1 - glow * 0.58
        layers.core.BackgroundTransparency = config.cornerBoxOpacity
    end
    entry.cornerBox.root.Visible = true
end

local function updateESPForPlayer(player, cameraPosition, camera)
    if player == LocalPlayer then return end
    local config = RuntimeState.ESPConfig
    local character = player.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    local head = character and (character:FindFirstChild("Head") or root)
    if not config.enabled or not character or not root or not humanoid or humanoid.Health <= 0 or not head then
        clearESPForPlayer(player)
        return
    end
    local distance = (root.Position - cameraPosition).Magnitude
    if distance > config.maxDistance then
        clearESPForPlayer(player)
        return
    end

    local entry = RuntimeState.ESPObjects[player]
    if not entry then entry = {}; RuntimeState.ESPObjects[player] = entry end
    updateCornerBoxForPlayer(entry, player, character, camera)

    if config.highlight then
        if not entry.highlight or not entry.highlight.Parent then
            entry.highlight = Instance.new("Highlight")
            entry.highlight.Name = "CATScript_ESPHighlight"
            entry.highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            entry.highlight.Parent = character
        end
        entry.highlight.Adornee = character
        entry.highlight.OutlineColor = config.outlineColor
        entry.highlight.FillColor = config.fillColor
        entry.highlight.FillTransparency = 0.72
        entry.highlight.OutlineTransparency = 0
    elseif entry.highlight then
        entry.highlight:Destroy(); entry.highlight = nil
    end

    if not entry.billboard or not entry.billboard.Parent then
        local billboard = Instance.new("BillboardGui")
        billboard.Name = "CATScript_ESPInfo"
        billboard.Adornee = head
        billboard.AlwaysOnTop = true
        billboard.Size = UDim2.fromOffset(180, 48)
        billboard.StudsOffset = Vector3.new(0, 2.7, 0)
        billboard.MaxDistance = config.maxDistance
        billboard.Parent = head

        local nameLabel = Instance.new("TextLabel")
        nameLabel.Name = "NameDistance"
        nameLabel.BackgroundTransparency = 1
        nameLabel.Size = UDim2.new(1, 0, 0, 22)
        nameLabel.Font = Enum.Font.GothamBold
        nameLabel.TextSize = 12
        nameLabel.TextStrokeTransparency = 0.25
        nameLabel.Parent = billboard

        local barBack = Instance.new("Frame")
        barBack.Name = "HealthBack"
        barBack.Position = UDim2.fromOffset(30, 26)
        barBack.Size = UDim2.fromOffset(120, 5)
        barBack.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
        barBack.BorderSizePixel = 0
        barBack.Parent = billboard
        local barFill = Instance.new("Frame")
        barFill.Name = "HealthFill"
        barFill.Size = UDim2.fromScale(1, 1)
        barFill.BorderSizePixel = 0
        barFill.Parent = barBack
        entry.billboard = billboard
        entry.nameLabel = nameLabel
        entry.healthBack = barBack
        entry.healthFill = barFill
    end

    entry.billboard.Adornee = head
    entry.billboard.MaxDistance = config.maxDistance
    entry.nameLabel.Visible = config.names or config.distance
    local display = config.names and player.DisplayName or ""
    if config.names and player.DisplayName ~= player.Name then display = display .. " (@" .. player.Name .. ")" end
    if config.distance then
        local distanceText = string.format("%dm", math.floor(distance + 0.5))
        display = display == "" and distanceText or (display .. "  •  " .. distanceText)
    end
    entry.nameLabel.Text = display
    entry.nameLabel.TextColor3 = config.textColor
    entry.healthBack.Visible = config.health
    entry.healthFill.Visible = config.health
    entry.healthFill.BackgroundColor3 = config.healthColor
    entry.healthFill.Size = UDim2.new(math.clamp(humanoid.Health / math.max(humanoid.MaxHealth, 1), 0, 1), 0, 1, 0)
end

RuntimeState.espConnection = RunService.Heartbeat:Connect(function()
    local config = RuntimeState.ESPConfig
    if not config or not config.enabled then
        for player in pairs(RuntimeState.ESPObjects) do clearESPForPlayer(player) end
        return
    end
    local camera = workspace.CurrentCamera
    if not camera then return end
    local cameraPosition = camera.CFrame.Position
    for _, player in ipairs(Players:GetPlayers()) do
        updateESPForPlayer(player, cameraPosition, camera)
    end
end)
Players.PlayerRemoving:Connect(clearESPForPlayer)

-- Teleport page: remember the local character's current position and return to it later.
-- Coordinates are stored in the existing CATScript config file when Auto Save is enabled.
local savedTeleportPosition = nil
do
    local stored = type(SavedConfig.teleportSave) == "table" and SavedConfig.teleportSave or nil
    if stored then
        local x, y, z = tonumber(stored.x), tonumber(stored.y), tonumber(stored.z)
        local valid = x and y and z and x == x and y == y and z == z
            and math.abs(x) < 10000000 and math.abs(y) < 10000000 and math.abs(z) < 10000000
        if valid then savedTeleportPosition = Vector3.new(x, y, z) end
    end
end
RuntimeState.TeleportSave = {position = savedTeleportPosition}

RuntimeState.TeleportCard = make("Frame", {
    Name = "TeleportCard",
    BackgroundColor3 = COLORS.panel,
    BackgroundTransparency = 0.24,
    BorderSizePixel = 0,
    Position = UDim2.fromOffset(0, 0),
    Size = UDim2.new(1, 0, 0, 278),
    Visible = false,
}, RuntimeState.Content)
corner(RuntimeState.TeleportCard, 10)
stroke(RuntimeState.TeleportCard, COLORS.outline, 0, 1)
make("UIPadding", {
    PaddingTop = UDim.new(0, 14),
    PaddingBottom = UDim.new(0, 12),
    PaddingLeft = UDim.new(0, 16),
    PaddingRight = UDim.new(0, 16),
}, RuntimeState.TeleportCard)

local teleportTitle = addLabel(RuntimeState.TeleportCard, "Teleport", 12, COLORS.text, Enum.Font.GothamBold)
teleportTitle.Size = UDim2.new(1, 0, 0, 18)
local teleportSubtitle = addLabel(RuntimeState.TeleportCard, "Save your current location and return to it anytime.", 9, COLORS.muted, Enum.Font.Gotham)
teleportSubtitle.Position = UDim2.fromOffset(0, 19)
teleportSubtitle.Size = UDim2.new(1, 0, 0, 15)

local teleportSaveRow = make("Frame", {
    Name = "TeleportSaveRow",
    Position = UDim2.fromOffset(0, 48),
    Size = UDim2.new(1, 0, 0, 76),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.TeleportCard)
corner(teleportSaveRow, 8)
stroke(teleportSaveRow, COLORS.outline, 0.35, 1)
local teleportSaveTitle = addLabel(teleportSaveRow, "TELEPORT SAVE", 9, COLORS.text, Enum.Font.GothamSemibold)
teleportSaveTitle.Position = UDim2.fromOffset(12, 10)
teleportSaveTitle.Size = UDim2.new(1, -154, 0, 16)
local teleportSaveDescription = addLabel(teleportSaveRow, "Remember the coordinates where your character is standing now.", 8, COLORS.muted, Enum.Font.Gotham)
teleportSaveDescription.Position = UDim2.fromOffset(12, 31)
teleportSaveDescription.Size = UDim2.new(1, -154, 0, 28)
teleportSaveDescription.TextWrapped = true
local teleportSaveButton = make("TextButton", {
    Name = "TeleportSaveButton",
    Text = "SAVE POSITION",
    TextColor3 = COLORS.white,
    TextSize = 8,
    Font = Enum.Font.GothamBold,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(124, 30),
    BackgroundColor3 = COLORS.accent,
    BorderSizePixel = 0,
    AutoButtonColor = false,
}, teleportSaveRow)
corner(teleportSaveButton, 7)

local teleportReturnRow = make("Frame", {
    Name = "TeleportReturnRow",
    Position = UDim2.fromOffset(0, 136),
    Size = UDim2.new(1, 0, 0, 76),
    BackgroundColor3 = COLORS.panel2,
    BackgroundTransparency = 0.18,
    BorderSizePixel = 0,
}, RuntimeState.TeleportCard)
corner(teleportReturnRow, 8)
stroke(teleportReturnRow, COLORS.outline, 0.35, 1)
local teleportReturnTitle = addLabel(teleportReturnRow, "TELEPORT", 9, COLORS.text, Enum.Font.GothamSemibold)
teleportReturnTitle.Position = UDim2.fromOffset(12, 10)
teleportReturnTitle.Size = UDim2.new(1, -154, 0, 16)
local teleportReturnDescription = addLabel(teleportReturnRow, "Move your character back to the saved location.", 8, COLORS.muted, Enum.Font.Gotham)
teleportReturnDescription.Position = UDim2.fromOffset(12, 31)
teleportReturnDescription.Size = UDim2.new(1, -154, 0, 28)
teleportReturnDescription.TextWrapped = true
local teleportReturnButton = make("TextButton", {
    Name = "TeleportReturnButton",
    Text = "TELEPORT",
    TextColor3 = COLORS.white,
    TextSize = 8,
    Font = Enum.Font.GothamBold,
    AnchorPoint = Vector2.new(1, 0.5),
    Position = UDim2.new(1, -12, 0.5, 0),
    Size = UDim2.fromOffset(124, 30),
    BackgroundColor3 = COLORS.dim,
    BorderSizePixel = 0,
    AutoButtonColor = false,
}, teleportReturnRow)
corner(teleportReturnButton, 7)

local teleportSavedReadout = addLabel(RuntimeState.TeleportCard, "NO SAVED POSITION  /  PRESS SAVE POSITION", 8, COLORS.dim, Enum.Font.Code)
teleportSavedReadout.Position = UDim2.fromOffset(2, 222)
teleportSavedReadout.Size = UDim2.new(1, -4, 0, 18)
teleportSavedReadout.TextWrapped = true
local teleportStatus = addLabel(RuntimeState.TeleportCard, "LOCAL COORDINATES  •  AUTO SAVE USES YOUR CATScript CONFIG", 7, COLORS.dim, Enum.Font.GothamMedium)
teleportStatus.Position = UDim2.fromOffset(2, 548)
teleportStatus.Size = UDim2.new(1, -4, 0, 16)
teleportStatus.TextWrapped = true

-- Three built-in teleport destinations. The supplied data included the first
-- Y coordinate as "L11942.947678"; this is interpreted as 119.42947678.
-- Six rotation entries were supplied for each CFrame (the first two matrix
-- rows). The final row is reconstructed with a cross product to complete the
-- orthonormal rotation matrix.
local function makePresetCFrame(x, y, z, r00, r01, r02, r10, r11, r12)
    local r20 = r01 * r12 - r02 * r11
    local r21 = r02 * r10 - r00 * r12
    local r22 = r00 * r11 - r01 * r10
    return CFrame.new(x, y, z, r00, r01, r02, r10, r11, r12, r20, r21, r22)
end

local teleportPresets = {
    {
        name = "1barel",
        cframe = makePresetCFrame(
            -1936.05457, 119.42947678, -3695.01465,
            0.505478442, 6.06631687e-08, 0.862839222,
            4.99525354e-09, 1, -7.32328331e-08
        ),
    },
    {
        name = "2barel",
        cframe = makePresetCFrame(
            1706.30664, 120.947685, 3776.1958,
            0.0264643189, 5.39139116e-08, -0.999649763,
            2.17682423e-08, 1, 5.45090835e-08
        ),
    },
    {
        name = "3barel",
        cframe = makePresetCFrame(
            -1208.11548, 72.6962891, -881.476624,
            0.708935976, 4.2734321e-08, -0.705272853,
            3.07026493e-09, 1, 6.36788187e-08
        ),
    },
    {
        name = "Capture Point",
        cframe = CFrame.new(-504.91, 177.04, -1024.32),
    },
}
RuntimeState.TeleportPresets = teleportPresets

local presetHeader = addLabel(RuntimeState.TeleportCard, "SAVED MAP LOCATIONS", 9, COLORS.text, Enum.Font.GothamBold)
presetHeader.Position = UDim2.fromOffset(2, 264)
presetHeader.Size = UDim2.new(1, -4, 0, 16)

local function teleportCharacterTo(destination, successMessage)
    local character = LocalPlayer.Character
    if not character or not character.Parent then
        teleportStatus.Text = "CHARACTER NOT READY  /  TRY AGAIN"
        teleportStatus.TextColor3 = Color3.fromRGB(235, 145, 120)
        return false
    end
    local ok, err = pcall(function()
        character:PivotTo(destination)
    end)
    if ok then
        teleportStatus.Text = successMessage
        teleportStatus.TextColor3 = COLORS.accent
        return true
    end
    teleportStatus.Text = "TELEPORT FAILED  /  CHARACTER NOT READY"
    teleportStatus.TextColor3 = Color3.fromRGB(235, 145, 120)
    warn("[CATScript] Teleport failed: " .. tostring(err))
    return false
end

for index, preset in ipairs(teleportPresets) do
    local rowY = 286 + (index - 1) * 64
    local row = make("Frame", {
        Name = "TeleportPreset_" .. preset.name,
        Position = UDim2.fromOffset(0, rowY),
        Size = UDim2.new(1, 0, 0, 56),
        BackgroundColor3 = COLORS.panel2,
        BackgroundTransparency = 0.18,
        BorderSizePixel = 0,
    }, RuntimeState.TeleportCard)
    corner(row, 8)
    stroke(row, COLORS.outline, 0.35, 1)

    local nameLabel = addLabel(row, preset.name, 9, COLORS.text, Enum.Font.GothamSemibold)
    nameLabel.Position = UDim2.fromOffset(12, 7)
    nameLabel.Size = UDim2.new(1, -142, 0, 15)

    local coordinatesLabel = addLabel(row, string.format(
        "X %.3f   Y %.3f   Z %.3f", preset.cframe.Position.X,
        preset.cframe.Position.Y, preset.cframe.Position.Z
    ), 7, COLORS.muted, Enum.Font.Code)
    coordinatesLabel.Position = UDim2.fromOffset(12, 27)
    coordinatesLabel.Size = UDim2.new(1, -142, 0, 18)
    coordinatesLabel.TextWrapped = true

    local button = make("TextButton", {
        Name = "TeleportTo_" .. preset.name,
        Text = "TELEPORT",
        TextColor3 = COLORS.white,
        TextSize = 8,
        Font = Enum.Font.GothamBold,
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -10, 0.5, 0),
        Size = UDim2.fromOffset(112, 29),
        BackgroundColor3 = COLORS.accent,
        BorderSizePixel = 0,
        AutoButtonColor = false,
    }, row)
    corner(button, 7)
    button.MouseButton1Click:Connect(function()
        teleportCharacterTo(preset.cframe, "TELEPORTED  /  " .. preset.name)
    end)
end
RuntimeState.TeleportCard.Size = UDim2.new(1, 0, 0, 574)

local function refreshTeleportReadout()
    local position = RuntimeState.TeleportSave.position
    if typeof(position) == "Vector3" then
        teleportSavedReadout.Text = string.format("SAVED  /  X %.2f   Y %.2f   Z %.2f", position.X, position.Y, position.Z)
        teleportSavedReadout.TextColor3 = COLORS.accent
        teleportReturnButton.BackgroundColor3 = COLORS.accent
        teleportReturnButton.TextColor3 = COLORS.white
    else
        teleportSavedReadout.Text = "NO SAVED POSITION  /  PRESS SAVE POSITION"
        teleportSavedReadout.TextColor3 = COLORS.dim
        teleportReturnButton.BackgroundColor3 = COLORS.dim
        teleportReturnButton.TextColor3 = COLORS.white
    end
end

teleportSaveButton.MouseButton1Click:Connect(function()
    local character = LocalPlayer.Character
    local root = character and (character:FindFirstChild("HumanoidRootPart") or character.PrimaryPart)
    if not root then
        teleportStatus.Text = "CHARACTER NOT READY  /  TRY AGAIN"
        teleportStatus.TextColor3 = Color3.fromRGB(235, 145, 120)
        return
    end
    RuntimeState.TeleportSave.position = root.Position
    refreshTeleportReadout()
    teleportStatus.Text = autoSaveEnabled and "POSITION SAVED  /  AUTO SAVE ENABLED" or "POSITION SAVED FOR THIS SESSION  /  AUTO SAVE OFF"
    teleportStatus.TextColor3 = COLORS.accent
    if scheduleConfigSave then scheduleConfigSave() end
end)

teleportReturnButton.MouseButton1Click:Connect(function()
    local position = RuntimeState.TeleportSave.position
    if typeof(position) ~= "Vector3" then
        teleportStatus.Text = "NO SAVED POSITION  /  PRESS SAVE POSITION FIRST"
        teleportStatus.TextColor3 = Color3.fromRGB(235, 145, 120)
        return
    end
    local character = LocalPlayer.Character
    if not character or not character.Parent then
        teleportStatus.Text = "CHARACTER NOT READY  /  TRY AGAIN"
        teleportStatus.TextColor3 = Color3.fromRGB(235, 145, 120)
        return
    end
    local ok, err = pcall(function()
        local currentPivot = character:GetPivot()
        character:PivotTo(CFrame.new(position) * currentPivot.Rotation)
    end)
    if ok then
        teleportStatus.Text = "TELEPORTED TO SAVED POSITION"
        teleportStatus.TextColor3 = COLORS.accent
    else
        teleportStatus.Text = "TELEPORT FAILED  /  CHARACTER NOT READY"
        teleportStatus.TextColor3 = Color3.fromRGB(235, 145, 120)
        warn("[CATScript] Teleport failed: " .. tostring(err))
    end
end)
refreshTeleportReadout()

-- Smooth cross-fade + short horizontal slide between dashboard pages.
-- Transparency defaults are cached per instance so repeated transitions never
-- progressively alter the original styling of controls and cards.
local pageCards = {
    Visual = RuntimeState.VisualCard,
    Player = RuntimeState.PlayerCard,
    ESP = RuntimeState.ESPCard,
    Teleport = RuntimeState.TeleportCard,
    Settings = RuntimeState.SettingsCard,
}
local pageVisualDefaults = setmetatable({}, {__mode = "k"})
local pageTransitionTweens = setmetatable({}, {__mode = "k"})
local pageTransitionGeneration = 0
local PAGE_TRANSITION_TIME = 0.22
local pageFadeInfo = TweenInfo.new(PAGE_TRANSITION_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

local function capturePageVisualDefaults(card)
    local snapshot = pageVisualDefaults[card]
    if not snapshot then
        snapshot = {
            position = card.Position,
            entries = {},
            captured = setmetatable({}, {__mode = "k"}),
        }
        pageVisualDefaults[card] = snapshot
    end

    local function captureObject(object)
        if snapshot.captured[object] then return end
        snapshot.captured[object] = true
        local properties = {}
        local function remember(property)
            local ok, value = pcall(function() return object[property] end)
            if ok and type(value) == "number" then properties[property] = value end
        end

        if object:IsA("GuiObject") then
            remember("BackgroundTransparency")
        end
        if object:IsA("TextLabel") or object:IsA("TextButton") or object:IsA("TextBox") then
            remember("TextTransparency")
            remember("TextStrokeTransparency")
        end
        if object:IsA("ImageLabel") or object:IsA("ImageButton") then
            remember("ImageTransparency")
        end
        if object:IsA("ScrollingFrame") then
            remember("ScrollBarImageTransparency")
        end
        if object:IsA("CanvasGroup") then
            remember("GroupTransparency")
        end
        if object:IsA("UIStroke") then
            remember("Transparency")
        end

        if next(properties) then
            table.insert(snapshot.entries, {object = object, properties = properties})
        end
    end

    captureObject(card)
    for _, object in ipairs(card:GetDescendants()) do captureObject(object) end
    return snapshot
end

local function cancelPageTweens(card)
    local running = pageTransitionTweens[card]
    if running then
        for _, tween in ipairs(running) do pcall(function() tween:Cancel() end) end
    end
    pageTransitionTweens[card] = {}
end

local function setPageVisualAlpha(card, alpha)
    local snapshot = capturePageVisualDefaults(card)
    for _, entry in ipairs(snapshot.entries) do
        if entry.object and entry.object.Parent then
            for property, original in pairs(entry.properties) do
                pcall(function() entry.object[property] = alpha >= 1 and 1 or original end)
            end
        end
    end
end

local function tweenPageVisuals(card, alpha, targetPosition)
    local snapshot = capturePageVisualDefaults(card)
    local running = pageTransitionTweens[card] or {}
    pageTransitionTweens[card] = running
    for _, entry in ipairs(snapshot.entries) do
        if entry.object and entry.object.Parent then
            local goals = {}
            for property, original in pairs(entry.properties) do
                goals[property] = alpha >= 1 and 1 or original
            end
            local ok, tween = pcall(function()
                return TweenService:Create(entry.object, pageFadeInfo, goals)
            end)
            if ok and tween then table.insert(running, tween); tween:Play() end
        end
    end
    if targetPosition then
        local ok, tween = pcall(function()
            return TweenService:Create(card, pageFadeInfo, {Position = targetPosition})
        end)
        if ok and tween then table.insert(running, tween); tween:Play() end
    end
end

local function offsetPagePosition(position, xOffset)
    return UDim2.new(position.X.Scale, position.X.Offset + xOffset, position.Y.Scale, position.Y.Offset)
end

local function showPage(pageName)
    local visualActive = pageName == "Visual"
    local playerActive = pageName == "Player"
    local settingsActive = pageName == "Settings"
    local espActive = pageName == "ESP"
    local teleportActive = pageName == "Teleport"
    local oldCard = pageCards[currentPage]
    local newCard = pageCards[pageName]
    local transitionStarted = pageName ~= currentPage and oldCard ~= nil and newCard ~= nil

    if transitionStarted then
        pageTransitionGeneration = pageTransitionGeneration + 1
        local thisGeneration = pageTransitionGeneration

        -- Cancel any unfinished animation and restore each page's exact base
        -- appearance before starting the next transition (rapid clicks are safe).
        for _, card in pairs(pageCards) do
            cancelPageTweens(card)
            local snapshot = capturePageVisualDefaults(card)
            setPageVisualAlpha(card, 0)
            card.Position = snapshot.position
        end

        oldCard.Visible = true
        newCard.Visible = true
        local newSnapshot = capturePageVisualDefaults(newCard)
        setPageVisualAlpha(newCard, 1)
        newCard.Position = offsetPagePosition(newSnapshot.position, 16)

        local oldSnapshot = capturePageVisualDefaults(oldCard)
        tweenPageVisuals(oldCard, 1, offsetPagePosition(oldSnapshot.position, -12))
        tweenPageVisuals(newCard, 0, newSnapshot.position)

        for name, card in pairs(pageCards) do
            if card ~= oldCard and card ~= newCard then card.Visible = false end
        end
        task.delay(PAGE_TRANSITION_TIME + 0.04, function()
            if pageTransitionGeneration ~= thisGeneration then return end
            oldCard.Visible = false
            oldCard.Position = oldSnapshot.position
            setPageVisualAlpha(oldCard, 0)
            newCard.Position = newSnapshot.position
        end)
    else
        for name, card in pairs(pageCards) do
            card.Visible = name == pageName
            cancelPageTweens(card)
            local snapshot = capturePageVisualDefaults(card)
            setPageVisualAlpha(card, 0)
            card.Position = snapshot.position
        end
    end

    currentPage = pageName

    VisualNav.BackgroundColor3 = visualActive and COLORS.panel2 or COLORS.sidebar
    indicator.BackgroundTransparency = visualActive and 0 or 1
    navIcon.TextColor3 = visualActive and COLORS.accent or COLORS.muted
    navText.TextColor3 = visualActive and COLORS.text or COLORS.muted

    PlayerNav.BackgroundColor3 = playerActive and COLORS.panel2 or COLORS.sidebar
    playerIndicator.BackgroundTransparency = playerActive and 0 or 1
    playerIcon.TextColor3 = playerActive and COLORS.accent or COLORS.muted
    playerText.TextColor3 = playerActive and COLORS.text or COLORS.muted

    ESPNav.BackgroundColor3 = espActive and COLORS.panel2 or COLORS.sidebar
    espIndicator.BackgroundTransparency = espActive and 0 or 1
    espIcon.TextColor3 = espActive and COLORS.accent or COLORS.muted
    espText.TextColor3 = espActive and COLORS.text or COLORS.muted

    TeleportNav.BackgroundColor3 = teleportActive and COLORS.panel2 or COLORS.sidebar
    teleportIndicator.BackgroundTransparency = teleportActive and 0 or 1
    teleportIcon.TextColor3 = teleportActive and COLORS.accent or COLORS.muted
    teleportText.TextColor3 = teleportActive and COLORS.text or COLORS.muted

    SettingsNav.BackgroundColor3 = settingsActive and COLORS.panel2 or COLORS.sidebar
    settingsIndicator.BackgroundTransparency = settingsActive and 0 or 1
    settingsIcon.TextColor3 = settingsActive and COLORS.accent or COLORS.muted
    settingsText.TextColor3 = settingsActive and COLORS.text or COLORS.muted

    RuntimeState.Breadcrumb.Text = "CATScript   /   " .. pageName
    RuntimeState.Search.PlaceholderText = pageName == "Visual" and "Search visual..." or (pageName == "Player" and "Search player..." or (pageName == "ESP" and "Search ESP..." or (pageName == "Teleport" and "Search teleport..." or "Search settings...")))
    RuntimeState.Content.CanvasPosition = Vector2.new(0, 0)
end
VisualNav.MouseButton1Click:Connect(function() showPage("Visual") end)
PlayerNav.MouseButton1Click:Connect(function() showPage("Player") end)
ESPNav.MouseButton1Click:Connect(function() showPage("ESP") end)
TeleportNav.MouseButton1Click:Connect(function() showPage("Teleport") end)
SettingsNav.MouseButton1Click:Connect(function() showPage("Settings") end)

-- Bottom console strip, kept as part of the dashboard appearance.
RuntimeState.Console = make("Frame", {
    Name = "Console",
    Position = UDim2.new(0, 0, 1, -38),
    Size = UDim2.new(1, 0, 0, 38),
    BackgroundColor3 = COLORS.topbar,
    BorderSizePixel = 0,
}, RuntimeState.Main)
make("Frame", {
    Position = UDim2.new(0, 0, 0, 0),
    Size = UDim2.new(1, 0, 0, 1),
    BackgroundColor3 = COLORS.outline,
    BorderSizePixel = 0,
}, RuntimeState.Console)
RuntimeState.ConsoleLabel = addLabel(RuntimeState.Console, ">_  CONSOLE", 9, COLORS.muted, Enum.Font.Code)
RuntimeState.ConsoleLabel.Position = UDim2.fromOffset(16, 1)
RuntimeState.ConsoleLabel.Size = UDim2.fromOffset(100, 16)
RuntimeState.ConsoleStatus = addLabel(RuntimeState.Console, "CATScript Visual UI loaded.", 9, COLORS.dim, Enum.Font.Code)
RuntimeState.ConsoleStatus.Position = UDim2.fromOffset(16, 17)
RuntimeState.ConsoleStatus.Size = UDim2.new(1, -30, 0, 16)

-- Hover effect: draw a square screen-space box around the player under the cursor, plus a neon ring at their feet.
local hoverEnabled = false -- Player ESP / Hover Highlight is deliberately removed.
local activeCharacter = nil
local activeHighlight = nil
local ringParts = {}
local mouse = LocalPlayer:GetMouse()
RuntimeState.RING_SEGMENTS = 28
RuntimeState.RING_RADIUS = 2.35
RuntimeState.RING_COLOR = hoverColor

-- Screen-space corner-bracket box. It intentionally draws ONLY four L-shaped
-- corners (like the supplied reference), not a continuous rectangle.
RuntimeState.BoxOverlay = make("Frame", {
    Name = "CATScript_HoverBoxOverlay",
    Position = UDim2.fromScale(0, 0),
    Size = UDim2.fromScale(1, 1),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Visible = false,
    Active = false,
    ZIndex = 500,
}, ScreenGui)

RuntimeState.BoxFrame = make("Frame", {
    Name = "CornerBracketBounds",
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    Visible = true,
    ZIndex = 500,
}, RuntimeState.BoxOverlay)

-- Eight bars form four L-shaped corners. Each bar has two soft glow layers
-- behind a crisp white-blue core to reproduce the bright reference brackets.
RuntimeState.BoxSegments = {}
for i = 1, 8 do
    local outerGlow = make("Frame", {
        Name = "CornerOuterGlow" .. i,
        BackgroundColor3 = RuntimeState.RING_COLOR,
        BackgroundTransparency = 0.84,
        BorderSizePixel = 0,
        ZIndex = 501,
    }, RuntimeState.BoxOverlay)
    corner(outerGlow, 3)

    local innerGlow = make("Frame", {
        Name = "CornerInnerGlow" .. i,
        BackgroundColor3 = RuntimeState.RING_COLOR,
        BackgroundTransparency = 0.58,
        BorderSizePixel = 0,
        ZIndex = 502,
    }, RuntimeState.BoxOverlay)
    corner(innerGlow, 2)

    local core = make("Frame", {
        Name = "CornerCore" .. i,
        BackgroundColor3 = hoverColor,
        BackgroundTransparency = 0,
        BorderSizePixel = 0,
        ZIndex = 503,
    }, RuntimeState.BoxOverlay)
    corner(core, 2)
    RuntimeState.BoxSegments[i] = {outer = outerGlow, inner = innerGlow, core = core}
end

local function colorToHex(color)
    local r = math.floor(color.R * 255 + 0.5)
    local g = math.floor(color.G * 255 + 0.5)
    local b = math.floor(color.B * 255 + 0.5)
    return string.format("#%02X%02X%02X", r, g, b)
end

local function colorToRGBText(color)
    local r = math.floor(color.R * 255 + 0.5)
    local g = math.floor(color.G * 255 + 0.5)
    local b = math.floor(color.B * 255 + 0.5)
    return string.format("%d, %d, %d", r, g, b)
end

local function refreshColorPicker()
    RuntimeState.SVArea.BackgroundColor3 = Color3.fromHSV(hoverHue, 1, 1)
    RuntimeState.SVMarker.Position = UDim2.fromScale(hoverSaturation, 1 - hoverValue)
    RuntimeState.HueMarker.Position = UDim2.new(0.5, 0, hoverHue, 0)
    RuntimeState.HoverColorButton.BackgroundColor3 = hoverColor
    if not RuntimeState.HexBox:IsFocused() then RuntimeState.HexBox.Text = colorToHex(hoverColor) end
    if not RuntimeState.RGBBox:IsFocused() then RuntimeState.RGBBox.Text = colorToRGBText(hoverColor) end
end

local function trailColorToHex(color)
    return string.format("#%02X%02X%02X", math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5))
end
local function trailColorToRGBText(color)
    return string.format("%d, %d, %d", math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5))
end
local applyTrailColor
local function applyTrailColorValue(color)
    if typeof(color) ~= "Color3" then return end
    trailColor = color
    trailHue, trailSaturation, trailValue = color:ToHSV()
    RuntimeState.TrailSVArea.BackgroundColor3 = Color3.fromHSV(trailHue, 1, 1)
    RuntimeState.TrailSVMarker.Position = UDim2.fromScale(trailSaturation, 1 - trailValue)
    RuntimeState.TrailHueMarker.Position = UDim2.new(0.5, 0, trailHue, 0)
    RuntimeState.TrailColorButton.BackgroundColor3 = trailColor
    if not RuntimeState.TrailHexBox:IsFocused() then RuntimeState.TrailHexBox.Text = trailColorToHex(trailColor) end
    if not RuntimeState.TrailRGBBox:IsFocused() then RuntimeState.TrailRGBBox.Text = trailColorToRGBText(trailColor) end
    applyTrailColor()
    if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
    if scheduleConfigSave then scheduleConfigSave() end
end
applyJumpCircleColorValue = function(color)
    if typeof(color) ~= "Color3" then return end
    jumpCircleColor = color
    jumpCircleHue, jumpCircleSaturation, jumpCircleValue = color:ToHSV()
    RuntimeState.JumpCircleSVArea.BackgroundColor3 = Color3.fromHSV(jumpCircleHue, 1, 1)
    RuntimeState.JumpCircleSVMarker.Position = UDim2.fromScale(jumpCircleSaturation, 1 - jumpCircleValue)
    RuntimeState.JumpCircleHueMarker.Position = UDim2.new(0.5, 0, jumpCircleHue, 0)
    RuntimeState.JumpCircleColorButton.BackgroundColor3 = jumpCircleColor
    if not RuntimeState.JumpCircleHexBox:IsFocused() then RuntimeState.JumpCircleHexBox.Text = colorToHex(jumpCircleColor) end
    if not RuntimeState.JumpCircleRGBBox:IsFocused() then RuntimeState.JumpCircleRGBBox.Text = colorToRGBText(jumpCircleColor) end
    for _, entry in ipairs(jumpCircleParts) do
        local part = entry.part
        if part and part.Parent then part.Color = jumpCircleColor end
    end
    if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
    if scheduleConfigSave then scheduleConfigSave() end
end

local function updateJumpCircleSVFromPosition(position)
    local origin, size = RuntimeState.JumpCircleSVArea.AbsolutePosition, RuntimeState.JumpCircleSVArea.AbsoluteSize
    if size.X <= 0 or size.Y <= 0 then return end
    jumpCircleSaturation = math.clamp((position.X - origin.X) / size.X, 0, 1)
    jumpCircleValue = 1 - math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    applyJumpCircleColorValue(Color3.fromHSV(jumpCircleHue, jumpCircleSaturation, jumpCircleValue))
end
local function updateJumpCircleHueFromPosition(position)
    local origin, size = RuntimeState.JumpCircleHueInput.AbsolutePosition, RuntimeState.JumpCircleHueInput.AbsoluteSize
    if size.Y <= 0 then return end
    jumpCircleHue = math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    applyJumpCircleColorValue(Color3.fromHSV(jumpCircleHue, jumpCircleSaturation, jumpCircleValue))
end

local function updateTrailSVFromPosition(position)
    local origin, size = RuntimeState.TrailSVArea.AbsolutePosition, RuntimeState.TrailSVArea.AbsoluteSize
    if size.X <= 0 or size.Y <= 0 then return end
    trailSaturation = math.clamp((position.X - origin.X) / size.X, 0, 1)
    trailValue = 1 - math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    applyTrailColorValue(Color3.fromHSV(trailHue, trailSaturation, trailValue))
end
local function updateTrailHueFromPosition(position)
    local origin, size = RuntimeState.TrailHueInput.AbsolutePosition, RuntimeState.TrailHueInput.AbsoluteSize
    if size.Y <= 0 then return end
    trailHue = math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    applyTrailColorValue(Color3.fromHSV(trailHue, trailSaturation, trailValue))
end

local function syncHoverVisualColors()
    -- Apply one shared color to every visible part of the effect. This is also
    -- called while hovering so stale instances cannot keep an old white color.
    RuntimeState.HoverColorButton.BackgroundColor3 = hoverColor
    for _, segment in ipairs(RuntimeState.BoxSegments) do
        segment.outer.BackgroundColor3 = hoverColor
        segment.inner.BackgroundColor3 = hoverColor
        segment.core.BackgroundColor3 = hoverColor
    end
    for _, part in ipairs(ringParts) do
        if part and part.Parent then
            part.Color = hoverColor
        end
    end
    if activeHighlight and activeHighlight.Parent then
        activeHighlight.OutlineColor = hoverColor
        activeHighlight.FillColor = hoverColor
    end
end

local function applyHoverColor(color)
    if typeof(color) ~= "Color3" then return end
    hoverColor = color
    hoverHue, hoverSaturation, hoverValue = color:ToHSV()
    RuntimeState.RING_COLOR = color
    if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
    syncHoverVisualColors()
    refreshColorPicker()
    if scheduleConfigSave then scheduleConfigSave() end
end

local function updateSVFromPosition(position)
    local origin = RuntimeState.SVArea.AbsolutePosition
    local size = RuntimeState.SVArea.AbsoluteSize
    if size.X <= 0 or size.Y <= 0 then return end
    hoverSaturation = math.clamp((position.X - origin.X) / size.X, 0, 1)
    hoverValue = 1 - math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    applyHoverColor(Color3.fromHSV(hoverHue, hoverSaturation, hoverValue))
end

local function updateHueFromPosition(position)
    local origin = RuntimeState.HueInput.AbsolutePosition
    local size = RuntimeState.HueInput.AbsoluteSize
    if size.Y <= 0 then return end
    hoverHue = math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    applyHoverColor(Color3.fromHSV(hoverHue, hoverSaturation, hoverValue))
end

local function parseHex(text)
    local value = tostring(text or ""):gsub("#", ""):gsub("%s", "")
    if #value ~= 6 or not value:match("^%x%x%x%x%x%x$") then return nil end
    local r = tonumber(value:sub(1, 2), 16)
    local g = tonumber(value:sub(3, 4), 16)
    local b = tonumber(value:sub(5, 6), 16)
    return Color3.fromRGB(r, g, b)
end

local function parseRGB(text)
    local values = {}
    for number in tostring(text or ""):gmatch("%d+") do
        table.insert(values, tonumber(number))
    end
    if #values ~= 3 then return nil end
    return Color3.fromRGB(
        math.clamp(values[1], 0, 255),
        math.clamp(values[2], 0, 255),
        math.clamp(values[3], 0, 255)
    )
end

RuntimeState.applyCustomAuraColor = function(color)
    if typeof(color) ~= "Color3" then return end
    local aura = RuntimeState.CustomAura
    aura.color = color
    aura.hue, aura.saturation, aura.value = color:ToHSV()
    RuntimeState.CustomAuraSVArea.BackgroundColor3 = Color3.fromHSV(aura.hue, 1, 1)
    RuntimeState.CustomAuraSVMarker.Position = UDim2.fromScale(aura.saturation, 1 - aura.value)
    RuntimeState.CustomAuraHueMarker.Position = UDim2.new(0.5, 0, aura.hue, 0)
    RuntimeState.CustomAuraColorButton.BackgroundColor3 = color
    if not RuntimeState.CustomAuraHexBox:IsFocused() then
        RuntimeState.CustomAuraHexBox.Text = string.format("#%02X%02X%02X", math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5))
    end
    if not RuntimeState.CustomAuraRGBBox:IsFocused() then
        RuntimeState.CustomAuraRGBBox.Text = string.format("%d, %d, %d", math.floor(color.R * 255 + 0.5), math.floor(color.G * 255 + 0.5), math.floor(color.B * 255 + 0.5))
    end
    for _, part in ipairs(aura.parts) do
        if part and part.Parent then part.Color = color end
    end
    if scheduleConfigSave then scheduleConfigSave() end
end

RuntimeState.updateCustomAuraSVFromPosition = function(position)
    local area = RuntimeState.CustomAuraSVArea
    local origin, size = area.AbsolutePosition, area.AbsoluteSize
    if size.X <= 0 or size.Y <= 0 then return end
    local aura = RuntimeState.CustomAura
    aura.saturation = math.clamp((position.X - origin.X) / size.X, 0, 1)
    aura.value = 1 - math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    RuntimeState.applyCustomAuraColor(Color3.fromHSV(aura.hue, aura.saturation, aura.value))
end

RuntimeState.updateCustomAuraHueFromPosition = function(position)
    local input = RuntimeState.CustomAuraHueInput
    local origin, size = input.AbsolutePosition, input.AbsoluteSize
    if size.Y <= 0 then return end
    local aura = RuntimeState.CustomAura
    aura.hue = math.clamp((position.Y - origin.Y) / size.Y, 0, 1)
    RuntimeState.applyCustomAuraColor(Color3.fromHSV(aura.hue, aura.saturation, aura.value))
end

local function setBoxSegment(index, x, y, width, height)
    local segment = RuntimeState.BoxSegments[index]
    if not segment then return end
    segment.outer.Position = UDim2.fromOffset(x - 3, y - 3)
    segment.outer.Size = UDim2.fromOffset(width + 6, height + 6)
    segment.inner.Position = UDim2.fromOffset(x - 1, y - 1)
    segment.inner.Size = UDim2.fromOffset(width + 2, height + 2)
    segment.core.Position = UDim2.fromOffset(x, y)
    segment.core.Size = UDim2.fromOffset(width, height)
end

local function hidePlayerBox()
    RuntimeState.BoxOverlay.Visible = false
end

local function updatePlayerBox(character, root)
    local camera = workspace.CurrentCamera
    if not camera or not character or not root then
        hidePlayerBox()
        return
    end

    local ok, boundsCFrame, boundsSize = pcall(function()
        return character:GetBoundingBox()
    end)
    if not ok or not boundsCFrame or not boundsSize then
        hidePlayerBox()
        return
    end

    local half = boundsSize * 0.5
    local corners = {
        Vector3.new(-half.X, -half.Y, -half.Z),
        Vector3.new(-half.X, -half.Y,  half.Z),
        Vector3.new(-half.X,  half.Y, -half.Z),
        Vector3.new(-half.X,  half.Y,  half.Z),
        Vector3.new( half.X, -half.Y, -half.Z),
        Vector3.new( half.X, -half.Y,  half.Z),
        Vector3.new( half.X,  half.Y, -half.Z),
        Vector3.new( half.X,  half.Y,  half.Z),
    }

    local minX, minY = math.huge, math.huge
    local maxX, maxY = -math.huge, -math.huge
    local frontCount = 0
    for _, cornerOffset in ipairs(corners) do
        local worldPoint = boundsCFrame:PointToWorldSpace(cornerOffset)
        local screenPoint = camera:WorldToViewportPoint(worldPoint)
        if screenPoint.Z > 0 then
            frontCount += 1
            minX = math.min(minX, screenPoint.X)
            minY = math.min(minY, screenPoint.Y)
            maxX = math.max(maxX, screenPoint.X)
            maxY = math.max(maxY, screenPoint.Y)
        end
    end

    local viewportSize = camera.ViewportSize
    if frontCount == 0 or maxX < 0 or maxY < 0 or minX > viewportSize.X or minY > viewportSize.Y then
        hidePlayerBox()
        return
    end

    minX = math.clamp(minX, 0, viewportSize.X)
    minY = math.clamp(minY, 0, viewportSize.Y)
    maxX = math.clamp(maxX, 0, viewportSize.X)
    maxY = math.clamp(maxY, 0, viewportSize.Y)

    local width = maxX - minX
    local height = maxY - minY
    if width < 5 or height < 10 then
        hidePlayerBox()
        return
    end

    -- A little breathing room around the avatar, as in the reference image.
    local padX = math.clamp(width * 0.025, 1, 4)
    local padY = math.clamp(height * 0.018, 1, 4)
    minX -= padX
    maxX += padX
    minY -= padY
    maxY += padY

    RuntimeState.BoxFrame.Position = UDim2.fromOffset(minX, minY)
    RuntimeState.BoxFrame.Size = UDim2.fromOffset(maxX - minX, maxY - minY)
    local width2 = maxX - minX
    local height2 = maxY - minY
    local shortSide = math.min(width2, height2)
    local cornerLength = math.clamp(shortSide * 0.20, 7, 18)
    local thickness = math.clamp(shortSide * 0.027, 1.5, 2.5)

    -- Four separate L-shaped corner brackets: top-left, top-right,
    -- bottom-left and bottom-right. There is deliberately no full border.
    setBoxSegment(1, minX, minY, cornerLength, thickness)
    setBoxSegment(2, minX, minY, thickness, cornerLength)
    setBoxSegment(3, maxX - cornerLength, minY, cornerLength, thickness)
    setBoxSegment(4, maxX - thickness, minY, thickness, cornerLength)
    setBoxSegment(5, minX, maxY - thickness, cornerLength, thickness)
    setBoxSegment(6, minX, maxY - cornerLength, thickness, cornerLength)
    setBoxSegment(7, maxX - cornerLength, maxY - thickness, cornerLength, thickness)
    setBoxSegment(8, maxX - thickness, maxY - cornerLength, thickness, cornerLength)

    RuntimeState.BoxOverlay.Visible = true
end

local function clearHoverEffect()
    if activeHighlight then
        pcall(function() activeHighlight:Destroy() end)
        activeHighlight = nil
    end
    for _, part in ipairs(ringParts) do
        pcall(function() part:Destroy() end)
    end
    table.clear(ringParts)
    hidePlayerBox()
    activeCharacter = nil
end

local function getHoveredPlayerCharacter()
    local target = mouse.Target
    if not target then
        -- A raycast fallback helps when the executor's Mouse.Target is stale.
        local unitRay = mouse.UnitRay
        local params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        params.FilterDescendantsInstances = LocalPlayer.Character and {LocalPlayer.Character} or {}
        params.IgnoreWater = true
        local hit = workspace:Raycast(unitRay.Origin, unitRay.Direction * 1500, params)
        target = hit and hit.Instance or nil
    end
    if not target then return nil end

    -- Walk upward until the actual character model is found. This avoids
    -- stopping at a nested weapon/model that happens to be attached to a rig.
    local character = nil
    local ancestor = target
    while ancestor and ancestor ~= workspace do
        if ancestor:IsA("Model") and ancestor:FindFirstChildOfClass("Humanoid") then
            character = ancestor
            break
        end
        ancestor = ancestor.Parent
    end

    if not character or character == LocalPlayer.Character then return nil end
    if not Players:GetPlayerFromCharacter(character) then return nil end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local root = character:FindFirstChild("HumanoidRootPart")
    if not humanoid or humanoid.Health <= 0 or not root then return nil end

    return character, humanoid, root
end

local function createRing()
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.IgnoreWater = true

    local circumferenceStep = (2 * math.pi * RuntimeState.RING_RADIUS) / RuntimeState.RING_SEGMENTS
    for i = 1, RuntimeState.RING_SEGMENTS do
        local angle = (i - 1) * (2 * math.pi / RuntimeState.RING_SEGMENTS)
        local tangent = Vector3.new(-math.sin(angle), 0, math.cos(angle))
        local position = Vector3.new(math.cos(angle) * RuntimeState.RING_RADIUS, 0, math.sin(angle) * RuntimeState.RING_RADIUS)

        local glow = Instance.new("Part")
        glow.Name = "CATScript_HoverRingGlow"
        glow.Anchored = true
        glow.CanCollide = false
        glow.CanTouch = false
        glow.CanQuery = false
        glow.CastShadow = false
        glow.Material = Enum.Material.Neon
        glow.Color = hoverColor
        glow.Transparency = 0.72
        glow.Size = Vector3.new(0.28, 0.055, circumferenceStep + 0.06)
        glow.CFrame = CFrame.lookAt(position, position + tangent, Vector3.yAxis)
        glow.Parent = workspace
        table.insert(ringParts, glow)

        local segment = Instance.new("Part")
        segment.Name = "CATScript_HoverRing"
        segment.Anchored = true
        segment.CanCollide = false
        segment.CanTouch = false
        segment.CanQuery = false
        segment.CastShadow = false
        segment.Material = Enum.Material.Neon
        segment.Color = hoverColor
        segment.Transparency = 0.12
        segment.Size = Vector3.new(0.105, 0.075, circumferenceStep + 0.055)
        segment.CFrame = CFrame.lookAt(position, position + tangent, Vector3.yAxis)
        segment.Parent = workspace
        table.insert(ringParts, segment)
    end
end

local function setHoveredCharacter(character)
    if activeCharacter == character then return end
    clearHoverEffect()
    activeCharacter = character

    local highlight = Instance.new("Highlight")
    highlight.Name = "CATScript_HoverHighlight"
    highlight.Adornee = character
    highlight.DepthMode = Enum.HighlightDepthMode.Occluded
    highlight.FillColor = hoverColor
    highlight.FillTransparency = 0.92
    highlight.OutlineColor = hoverColor
    highlight.OutlineTransparency = 0
    highlight.Parent = workspace
    activeHighlight = highlight

    createRing()
end

local function updateRing(character, humanoid, root)
    syncHoverVisualColors()
    local exclusions = {character}
    if LocalPlayer.Character then
        table.insert(exclusions, LocalPlayer.Character)
    end

    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = exclusions
    params.IgnoreWater = true

    local ground = workspace:Raycast(root.Position, Vector3.new(0, -18, 0), params)
    local groundY
    if ground then
        groundY = ground.Position.Y + 0.075
    else
        groundY = root.Position.Y - humanoid.HipHeight - (root.Size.Y * 0.5) + 0.12
    end

    local circumferenceStep = (2 * math.pi * RuntimeState.RING_RADIUS) / RuntimeState.RING_SEGMENTS
    local partIndex = 1
    for i = 1, RuntimeState.RING_SEGMENTS do
        local angle = (i - 1) * (2 * math.pi / RuntimeState.RING_SEGMENTS)
        local tangent = Vector3.new(-math.sin(angle), 0, math.cos(angle))
        local position = Vector3.new(
            root.Position.X + math.cos(angle) * RuntimeState.RING_RADIUS,
            groundY,
            root.Position.Z + math.sin(angle) * RuntimeState.RING_RADIUS
        )
        local frame = CFrame.lookAt(position, position + tangent, Vector3.yAxis)

        local glow = ringParts[partIndex]
        local segment = ringParts[partIndex + 1]
        if glow and segment then
            glow.CFrame = frame
            segment.CFrame = frame
        end
        partIndex += 2
    end
end

local characterTrail = nil
local trailAttachments = {}
local trailCharacterAddedConnection = nil
local function removeCharacterTrail()
    if characterTrail then pcall(function() characterTrail:Destroy() end) end
    characterTrail = nil
    for _, attachment in ipairs(trailAttachments) do
        if attachment and attachment.Parent then pcall(function() attachment:Destroy() end) end
    end
    table.clear(trailAttachments)
end

applyTrailColor = function()
    RuntimeState.TrailColorButton.BackgroundColor3 = trailColor
    if characterTrail then
        characterTrail.Color = ColorSequence.new(trailColor)
    end
end

applyTrailStyle = function()
    if characterTrail then
        characterTrail.WidthScale = NumberSequence.new({
            NumberSequenceKeypoint.new(0, trailStartWidth),
            NumberSequenceKeypoint.new(1, trailEndWidth),
        })
        characterTrail.Lifetime = trailLifetime
        characterTrail.MinLength = trailMinLength
    end
    if trailAttachments[1] and trailAttachments[1].Parent then
        trailAttachments[1].Position = Vector3.new(-trailSpread * 0.5, 0.05, 0.85)
    end
    if trailAttachments[2] and trailAttachments[2].Parent then
        trailAttachments[2].Position = Vector3.new(trailSpread * 0.5, 0.05, 0.85)
    end
end

local function attachCharacterTrail(character)
    removeCharacterTrail()
    if not trailEnabled or not character then return end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then
        root = character:WaitForChild("HumanoidRootPart", 5)
    end
    if not root or not root:IsA("BasePart") or not trailEnabled then return end

    local left = Instance.new("Attachment")
    left.Name = "CATScript_LocalTrailLeft"
    left.Position = Vector3.new(-trailSpread * 0.5, 0.05, 0.85)
    left.Parent = root
    local right = Instance.new("Attachment")
    right.Name = "CATScript_LocalTrailRight"
    right.Position = Vector3.new(trailSpread * 0.5, 0.05, 0.85)
    right.Parent = root
    table.insert(trailAttachments, left)
    table.insert(trailAttachments, right)

    local trail = Instance.new("Trail")
    trail.Name = "CATScript_LocalCharacterTrail"
    trail.Attachment0 = left
    trail.Attachment1 = right
    trail.Color = ColorSequence.new(trailColor)
    trail.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.12),
        NumberSequenceKeypoint.new(1, 1),
    })
    trail.WidthScale = NumberSequence.new({
        NumberSequenceKeypoint.new(0, trailStartWidth),
        NumberSequenceKeypoint.new(1, trailEndWidth),
    })
    trail.Lifetime = trailLifetime
    trail.MinLength = trailMinLength
    trail.LightEmission = 0.85
    trail.FaceCamera = true
    trail.Enabled = true
    trail.Parent = root
    characterTrail = trail
    applyTrailStyle()
end

-- Local-only Custom Aura renderer. A compact pool of 16 anchored Neon parts is
-- reused for all ten styles; no loops/instances are spawned every frame.
RuntimeState.ensureCustomAuraParts = function()
    local aura = RuntimeState.CustomAura
    if aura.folder and aura.folder.Parent then return end
    aura.folder = Instance.new("Folder")
    aura.folder.Name = "CATScript_CustomAuraEffects"
    aura.folder.Parent = workspace
    aura.parts = {}
    for index = 1, 18 do
        local part = Instance.new("Part")
        part.Name = "AuraElement_" .. tostring(index)
        part.Shape = Enum.PartType.Ball
        part.Material = Enum.Material.Neon
        part.Color = aura.color
        part.Size = Vector3.new(0.2, 0.2, 0.2)
        part.Anchored = true
        part.CanCollide = false
        part.CanTouch = false
        part.CanQuery = false
        part.CastShadow = false
        part.Transparency = 0.08
        part.Parent = aura.folder
        aura.parts[index] = part
    end
end

RuntimeState.updateCustomAura = function(deltaTime)
    local aura = RuntimeState.CustomAura
    if not aura.enabled or not aura.folder or not aura.folder.Parent then return end
    local character = LocalPlayer.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root or not root:IsA("BasePart") then
        for _, part in ipairs(aura.parts) do part.Transparency = 1 end
        return
    end
    aura.phase = (aura.phase + math.clamp(deltaTime or 0, 0, 0.1)) % 100000
    local phase = aura.phase
    local count = #aura.parts
    for index, part in ipairs(aura.parts) do
        local fraction = (index - 1) / count
        local angle = phase * 1.15 + fraction * math.pi * 2
        local radius, height, size, transparency, shape, rotationY, rotationZ
        radius, height, transparency, shape, rotationY, rotationZ = 1.6, 0, 0.06, Enum.PartType.Ball, angle, 0
        if aura.style == "Orbit Orbs" then
            radius = 1.8
            height = math.sin(angle * 2) * 1.05
            size = Vector3.new(0.23, 0.23, 0.23)
        elseif aura.style == "Double Helix" then
            local lane = (index - 1) % 2
            local step = math.floor((index - 1) / 2) / 8
            angle = phase * 1.7 + step * math.pi * 2 + lane * math.pi
            radius = 1.12
            height = -1.8 + step * 3.6
            size = Vector3.new(0.17, 0.17, 0.17)
        elseif aura.style == "Energy Ring" then
            angle = phase * 0.9 + fraction * math.pi * 2
            radius = 1.68 + math.sin(phase * 3) * 0.12
            height = math.sin(angle * 3) * 0.07
            size = Vector3.new(0.09, 0.09, 0.46)
            shape = Enum.PartType.Block
            rotationY = angle + math.pi / 2
        elseif aura.style == "Flame Spiral" then
            local progress = (fraction + (phase * 0.42) % 1) % 1
            angle = phase * 1.3 + progress * math.pi * 2
            radius = 0.48 + progress * 0.78
            height = -1.65 + progress * 3.3
            local diameter = 0.12 + (1 - progress) * 0.1
            size = Vector3.new(diameter, diameter, diameter)
            transparency = 0.08 + progress * 0.58
        elseif aura.style == "Triple Halo" then
            local ring = math.floor((index - 1) / 6)
            local segment = ((index - 1) % 6) / 6
            angle = phase * (ring == 1 and -0.72 or 0.72) + segment * math.pi * 2
            radius = 1.48
            height = -1.12 + ring * 1.12
            size = Vector3.new(0.34, 0.075, 0.075)
            shape = Enum.PartType.Block
            rotationY = angle + math.pi / 2
        elseif aura.style == "Electric Arc" then
            angle = phase * 2.8 + fraction * math.pi * 2
            radius = 1.46 + math.sin(phase * 11 + index * 3.7) * 0.2
            height = math.sin(phase * 6.4 + index * 1.7) * 1.35
            size = Vector3.new(0.07, 0.07, 0.5)
            shape = Enum.PartType.Block
            rotationY = angle
            rotationZ = math.sin(phase * 7 + index) * 0.4
        elseif aura.style == "Pulse Sphere" then
            local vertical = 1 - 2 * fraction
            local horizontal = math.sqrt(math.max(0, 1 - vertical * vertical))
            angle = fraction * math.pi * 8 + phase * 0.35
            local shell = 1.05 + math.sin(phase * 2.1) * 0.24
            radius = shell * horizontal
            height = vertical * shell * 1.65
            size = Vector3.new(0.14, 0.14, 0.14)
        elseif aura.style == "Rune Orbit" then
            angle = phase * 0.48 + fraction * math.pi * 2
            radius = 1.7
            height = math.sin(angle * 2) * 1.12
            size = Vector3.new(0.2, 0.38, 0.075)
            shape = Enum.PartType.Block
            rotationY = -angle
            rotationZ = math.rad(45)
        elseif aura.style == "Comet Ring" then
            angle = phase * 1.35 + fraction * math.pi * 2
            radius = 1.72
            height = 0.18 + math.sin(angle * 2) * 0.28
            local diameter = index == 1 and 0.38 or (0.1 + (1 - fraction) * 0.08)
            size = Vector3.new(diameter, diameter, diameter)
            transparency = index == 1 and 0.02 or (0.18 + fraction * 0.64)
        else -- Cosmic Dust
            angle = phase * (0.3 + fraction * 0.14) + fraction * math.pi * 2
            radius = 1.2 + 0.42 * math.cos(phase * 0.8 + index * 1.3)
            height = math.sin(phase * 0.7 + index * 2.39) * 1.48
            local diameter = 0.075 + (index % 4) * 0.025
            size = Vector3.new(diameter, diameter, diameter)
        end
        -- One global multiplier scales both the aura's footprint and every element.
        local sizeScale = math.clamp(tonumber(aura.size) or 1, 0.5, 3.5)
        radius = radius * sizeScale
        height = height * sizeScale
        size = (size or Vector3.new(0.18, 0.18, 0.18)) * sizeScale
        local position = Vector3.new(math.cos(angle) * radius, height, math.sin(angle) * radius)
        position += Vector3.new(aura.offsetX or 0, aura.offsetY or 0, aura.offsetZ or 0)
        part.Shape = shape
        part.Size = size
        part.Color = aura.color
        part.Transparency = math.clamp(transparency or 0.06, 0, 0.92)
        part.CFrame = root.CFrame * CFrame.new(position) * CFrame.Angles(0, rotationY or angle, rotationZ or 0)
    end
end

RuntimeState.setCustomAura = function(enabled)
    local aura = RuntimeState.CustomAura
    aura.enabled = enabled == true
    if not aura.enabled then
        if aura.connection then pcall(function() aura.connection:Disconnect() end); aura.connection = nil end
        if aura.folder then pcall(function() aura.folder:Destroy() end); aura.folder = nil end
        table.clear(aura.parts)
        return false
    end
    RuntimeState.ensureCustomAuraParts()
    if aura.connection then pcall(function() aura.connection:Disconnect() end) end
    aura.connection = RunService.RenderStepped:Connect(function(deltaTime)
        if not RuntimeState.active then return end
        RuntimeState.updateCustomAura(deltaTime)
    end)
    RuntimeState.updateCustomAura(0)
    return true
end

RuntimeState.refreshCustomAuraUI = function()
    local aura = RuntimeState.CustomAura
    if RuntimeState.CustomAuraToggleButton then
        RuntimeState.CustomAuraToggleButton.BackgroundColor3 = aura.enabled and COLORS.accent or COLORS.dim
        RuntimeState.CustomAuraToggleKnob.Position = aura.enabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3)
        RuntimeState.CustomAuraStatus.Text = (aura.enabled and "ACTIVE  /  " or "DISABLED  /  ") .. string.upper(aura.style)
        RuntimeState.CustomAuraStatus.TextColor3 = aura.enabled and COLORS.accent or COLORS.dim
    end
    if RuntimeState.CustomAuraColorButton then RuntimeState.CustomAuraColorButton.BackgroundColor3 = aura.color end
    for styleName, button in pairs(RuntimeState.CustomAuraStyleButtons or {}) do
        button.BackgroundColor3 = styleName == aura.style and COLORS.accent or COLORS.background
    end
    if RuntimeState.CustomAuraStyleButton then RuntimeState.CustomAuraStyleButton.Text = "STYLE  ▾" end
end

RuntimeState.setCustomAuraStyle = function(styleName)
    local aura = RuntimeState.CustomAura
    if not table.find(aura.styles, styleName) then return false end
    aura.style = styleName
    RuntimeState.refreshCustomAuraUI()
    if scheduleConfigSave then scheduleConfigSave() end
    return true
end

RuntimeState.setCustomAuraSize = function(newSize, shouldSave)
    local aura = RuntimeState.CustomAura
    local snapped = math.floor((tonumber(newSize) or aura.size) / 0.05 + 0.5) * 0.05
    aura.size = math.clamp(tonumber(string.format("%.2f", snapped)) or 1, 0.5, 3.5)
    local alpha = (aura.size - 0.5) / 3
    if RuntimeState.CustomAuraSizeFill then RuntimeState.CustomAuraSizeFill.Size = UDim2.new(alpha, 0, 1, 0) end
    if RuntimeState.CustomAuraSizeKnob then RuntimeState.CustomAuraSizeKnob.Position = UDim2.new(alpha, 0, 0.5, 0) end
    if RuntimeState.CustomAuraSizeValue then RuntimeState.CustomAuraSizeValue.Text = string.format("%.2fx", aura.size) end
    if shouldSave ~= false and scheduleConfigSave then scheduleConfigSave() end
    return aura.size
end

RuntimeState.updateCustomAuraSizeFromPointer = function(pointer, shouldSave)
    local track = RuntimeState.CustomAuraSizeTrack
    if not track then return end
    local width = track.AbsoluteSize.X
    if width <= 0 then return end
    local alpha = math.clamp((pointer.X - track.AbsolutePosition.X) / width, 0, 1)
    RuntimeState.setCustomAuraSize(0.5 + alpha * 3, shouldSave)
end

RuntimeState.setCustomAuraOffset = function(axis, newValue, shouldSave)
    local aura = RuntimeState.CustomAura
    axis = string.upper(tostring(axis or ""))
    local key = axis == "X" and "offsetX" or (axis == "Y" and "offsetY" or (axis == "Z" and "offsetZ" or nil))
    if not key then return 0 end
    local raw = (tonumber(newValue) or aura[key] or 0) * 10
    local snapped = (raw >= 0 and math.floor(raw + 0.5) or math.ceil(raw - 0.5)) / 10
    aura[key] = math.clamp(snapped, -10, 10)
    local control = aura.offsetControls and aura.offsetControls[axis]
    if control then
        local alpha = (aura[key] + 10) / 20
        control.fill.Size = UDim2.new(alpha, 0, 1, 0)
        control.knob.Position = UDim2.new(alpha, 0, 0.5, 0)
        control.value.Text = string.format("%+.1f", aura[key])
    end
    if shouldSave ~= false and scheduleConfigSave then scheduleConfigSave() end
    return aura[key]
end

RuntimeState.updateCustomAuraOffsetFromPointer = function(axis, pointer, shouldSave)
    local control = RuntimeState.CustomAura.offsetControls and RuntimeState.CustomAura.offsetControls[axis]
    local track = control and control.track
    if not track then return end
    local width = track.AbsoluteSize.X
    if width <= 0 then return end
    local alpha = math.clamp((pointer.X - track.AbsolutePosition.X) / width, 0, 1)
    RuntimeState.setCustomAuraOffset(axis, -10 + alpha * 20, shouldSave)
end

local function clearJumpCircleEffects()
    jumpCircleToken += 1
    for _, entry in ipairs(jumpCircleParts) do
        local part = entry.part
        if part then pcall(function() part:Destroy() end) end
    end
    table.clear(jumpCircleParts)
end

local function buildJumpCircleSpecs(styleName)
    local specs = {}
    local function regularPoints(count, phase)
        local points = {}
        phase = phase or 0
        for i = 1, count do
            local angle = ((i - 1) / count) * math.pi * 2 + phase
            points[i] = Vector2.new(math.cos(angle), math.sin(angle))
        end
        return points
    end
    local function radialPoints(count, radiusFunction, phase)
        local points = {}
        phase = phase or 0
        for i = 1, count do
            local angle = ((i - 1) / count) * math.pi * 2 + phase
            local radius = radiusFunction(angle, i)
            points[i] = Vector2.new(math.cos(angle) * radius, math.sin(angle) * radius)
        end
        return points
    end
    local function starPoints(pointCount, innerRadius, phase)
        return radialPoints(pointCount * 2, function(_, index)
            return index % 2 == 0 and innerRadius or 1
        end, phase or 0)
    end

    if styleName == "Double Ring" then
        specs = {
            {points = regularPoints(32), radius = 0.76, delay = 0},
            {points = regularPoints(32), radius = 1.0 + (RuntimeState.jumpCircleSizes.ringGap / 1.5), delay = 0.06},
        }
    elseif styleName == "Triple Ring" then
        specs = {
            {points = regularPoints(32), radius = 0.58, delay = 0},
            {points = regularPoints(32), radius = 0.82, delay = 0.07},
            {points = regularPoints(32), radius = 1.06, delay = 0.14},
        }
    elseif styleName == "Hexagon" then
        specs = {{points = regularPoints(6, math.pi / 6), radius = 1, delay = 0, subdivisions = 5}}
    elseif styleName == "Octagon" then
        specs = {{points = regularPoints(8, math.pi / 8), radius = 1, delay = 0, subdivisions = 4}}
    elseif styleName == "Pentagon" then
        specs = {{points = regularPoints(5, -math.pi / 2), radius = 1, delay = 0, subdivisions = 6}}
    elseif styleName == "Triangle" then
        specs = {{points = regularPoints(3, -math.pi / 2), radius = 1, delay = 0, subdivisions = 10}}
    elseif styleName == "Diamond" then
        specs = {{points = regularPoints(4, math.pi / 4), radius = 1, delay = 0, subdivisions = 7}}
    elseif styleName == "Starburst" then
        specs = {{points = starPoints(16, 0.58), radius = 1, delay = 0, subdivisions = 2}}
    elseif styleName == "Double Star" then
        specs = {
            {points = starPoints(12, 0.56), radius = 1, delay = 0, subdivisions = 2},
            {points = starPoints(12, 0.56, math.pi / 12), radius = 0.66, delay = 0.08, subdivisions = 2},
        }
    elseif styleName == "Pulse" then
        specs = {
            {points = regularPoints(36), radius = 1, delay = 0},
            {points = regularPoints(36), radius = 1, delay = 0.17},
        }
    elseif styleName == "Ripple" then
        specs = {
            {points = regularPoints(40), radius = 0.48, delay = 0},
            {points = regularPoints(40), radius = 0.76, delay = 0.11},
            {points = regularPoints(40), radius = 1.04, delay = 0.22},
        }
    elseif styleName == "Compass" then
        specs = {{points = starPoints(8, 0.30), radius = 1, delay = 0, subdivisions = 3}}
    elseif styleName == "Crosshair" then
        local cross = {
            Vector2.new(-0.22, 1), Vector2.new(0.22, 1), Vector2.new(0.22, 0.22),
            Vector2.new(1, 0.22), Vector2.new(1, -0.22), Vector2.new(0.22, -0.22),
            Vector2.new(0.22, -1), Vector2.new(-0.22, -1), Vector2.new(-0.22, -0.22),
            Vector2.new(-1, -0.22), Vector2.new(-1, 0.22), Vector2.new(-0.22, 0.22),
        }
        specs = {{points = cross, radius = 1, delay = 0, subdivisions = 3}}
    elseif styleName == "Flower" then
        specs = {{points = radialPoints(56, function(angle) return 0.78 + 0.22 * math.cos(angle * 8) end), radius = 1, delay = 0}}
    elseif styleName == "Petal" then
        specs = {{points = radialPoints(56, function(angle) return 0.78 + 0.20 * math.cos(angle * 5) end), radius = 1, delay = 0}}
    elseif styleName == "Gear" then
        specs = {{points = radialPoints(48, function(_, index) return index % 2 == 0 and 0.88 or 1 end), radius = 1, delay = 0}}
    elseif styleName == "Spiral" then
        local spiral = {}
        for index = 1, 72 do
            local t = (index - 1) / 71
            local angle = t * math.pi * 4
            local radius = 0.18 + 0.82 * t
            spiral[index] = Vector2.new(math.cos(angle) * radius, math.sin(angle) * radius)
        end
        specs = {{points = spiral, radius = 1, delay = 0}}
    elseif styleName == "Wave" then
        specs = {{points = radialPoints(64, function(angle) return 0.84 + 0.14 * math.sin(angle * 10) end), radius = 1, delay = 0}}
    elseif styleName == "Rune" then
        specs = {
            {points = starPoints(12, 0.76), radius = 1, delay = 0, subdivisions = 2},
            {points = regularPoints(12, math.pi / 12), radius = 0.72, delay = 0.04, subdivisions = 2},
        }
    else
        specs = {{points = regularPoints(36), radius = 1, delay = 0}}
    end
    return specs
end

-- The preview is an isolated clone inside a ViewportFrame. It never parents
-- preview effects to Workspace or edits the real LocalPlayer character.
do
    local P = RuntimeState.Preview
    P.World = Instance.new("WorldModel")
    P.World.Name = "CATScriptPreviewWorld"
    P.World.Parent = RuntimeState.PreviewViewport
    P.Camera = Instance.new("Camera")
    P.Camera.Name = "CATScriptPreviewCamera"
    P.Camera.FieldOfView = 35
    P.Camera.Parent = RuntimeState.PreviewViewport
    RuntimeState.PreviewViewport.CurrentCamera = P.Camera

    P.setModelRotation = function()
        if P.model and P.model.Parent then
            pcall(function()
                P.model:PivotTo(CFrame.new(0, 0, 0) * CFrame.Angles(0, math.pi + P.angle, 0))
            end)
            if P.scheduleRefresh then P.scheduleRefresh() end
        end
    end

    P.addSegment = function(parent, name, pointA, pointB, width, height, color, transparency)
        local delta = pointB - pointA
        if delta.Magnitude < 0.015 then return nil end
        local part = Instance.new("Part")
        part.Name = name
        part.Anchored = true
        part.CanCollide = false
        part.CanTouch = false
        part.CanQuery = false
        part.CastShadow = false
        part.Material = Enum.Material.Neon
        part.Color = color
        part.Transparency = transparency or 0.1
        part.Size = Vector3.new(math.max(width, 0.015), math.max(height, 0.012), delta.Magnitude * 1.04)
        part.CFrame = CFrame.lookAt((pointA + pointB) * 0.5, (pointA + pointB) * 0.5 + delta.Unit, Vector3.yAxis)
        part.Parent = parent
        return part
    end

    P.refresh = function()
        if not RuntimeState.active or not P.model or not P.model.Parent then return end
        if P.effects then P.effects:Destroy() end
        P.effects = Instance.new("Folder")
        P.effects.Name = "PreviewOnlyEffects"
        P.effects.Parent = P.World

        local pivot = P.model:GetPivot()
        local groundY = -2.82
        local boundsOk, boundsCF, boundsSize = pcall(function() return P.model:GetBoundingBox() end)
        if boundsOk then
            local center = boundsCF.Position
            local avatarHeight = boundsSize.Y
            local distance = math.max(7.2, avatarHeight * 1.85)
            P.Camera.CFrame = CFrame.lookAt(center + Vector3.new(0, 0.25, distance), center + Vector3.new(0, 0.1, 0))
        end

        if hoverEnabled then
            P.highlight = P.World:FindFirstChild("PreviewHoverHighlight")
            if P.highlight then P.highlight:Destroy() end
            P.highlight = Instance.new("Highlight")
            P.highlight.Name = "PreviewHoverHighlight"
            P.highlight.Adornee = P.model
            P.highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            P.highlight.FillColor = hoverColor
            P.highlight.FillTransparency = 0.94
            P.highlight.OutlineColor = hoverColor
            P.highlight.OutlineTransparency = 0
            P.highlight.Parent = P.World

            local radius = RuntimeState.RING_RADIUS or 2.35
            local count = 32
            for index = 1, count do
                local a1 = ((index - 1) / count) * math.pi * 2
                local a2 = (index / count) * math.pi * 2
                local p1 = pivot * Vector3.new(math.cos(a1) * radius, groundY, math.sin(a1) * radius)
                local p2 = pivot * Vector3.new(math.cos(a2) * radius, groundY, math.sin(a2) * radius)
                P.addSegment(P.effects, "PreviewHoverRing", p1, p2, 0.105, 0.065, hoverColor, 0.12)
            end
        else
            P.highlight = P.World:FindFirstChild("PreviewHoverHighlight")
            if P.highlight then P.highlight:Destroy() end
        end

        if trailEnabled then
            for sideIndex = 1, 2 do
                local side = sideIndex == 1 and -1 or 1
                for index = 1, 9 do
                    local alpha1 = (index - 1) / 9
                    local alpha2 = index / 9
                    local x1 = side * trailSpread * 0.5 * (1 - alpha1 * 0.22)
                    local x2 = side * trailSpread * 0.5 * (1 - alpha2 * 0.22)
                    local p1 = pivot * Vector3.new(x1, 0.08 + alpha1 * 0.05, 0.82 + alpha1 * 2.55)
                    local p2 = pivot * Vector3.new(x2, 0.08 + alpha2 * 0.05, 0.82 + alpha2 * 2.55)
                    local width = trailStartWidth + (trailEndWidth - trailStartWidth) * alpha1
                    P.addSegment(P.effects, "PreviewCharacterTrail", p1, p2, width, math.max(width * 0.16, 0.025), trailColor, 0.16 + alpha1 * 0.77)
                end
            end
        end

        if jumpCircleEnabled then
            local specs = buildJumpCircleSpecs(jumpCircleStyle)
            local sizes = RuntimeState.jumpCircleSizes
            local baseRadius = sizes.radius + sizes.expansion * 0.22
            for _, spec in ipairs(specs) do
                local points = spec.points
                local subdivisions = spec.subdivisions or 1
                local radius = baseRadius * (spec.radius or 1)
                for index = 1, #points do
                    local first = points[index]
                    local second = points[(index % #points) + 1]
                    for stepIndex = 0, subdivisions - 1 do
                        local t1 = stepIndex / subdivisions
                        local t2 = (stepIndex + 1) / subdivisions
                        local p1 = first + (second - first) * t1
                        local p2 = first + (second - first) * t2
                        local worldA = pivot * Vector3.new(p1.X * radius, groundY + sizes.height, p1.Y * radius)
                        local worldB = pivot * Vector3.new(p2.X * radius, groundY + sizes.height, p2.Y * radius)
                        P.addSegment(P.effects, "PreviewJumpCircle", worldA, worldB, sizes.thickness, math.max(sizes.thickness * 0.5, 0.02), jumpCircleColor, 0.08)
                    end
                end
            end
        end

        local active = {}
        if hoverEnabled then table.insert(active, "HOVER") end
        if trailEnabled then table.insert(active, "TRAIL") end
        if jumpCircleEnabled then table.insert(active, "JUMP CIRCLE") end
        RuntimeState.PreviewStatus.Text = #active > 0 and ("PREVIEW  /  " .. table.concat(active, "  +  ")) or "PREVIEW  /  NO EFFECTS ENABLED"
        RuntimeState.PreviewStatus.TextColor3 = #active > 0 and COLORS.accent or COLORS.dim
    end

    P.scheduleRefresh = function()
        if P.refreshQueued then return end
        P.refreshQueued = true
        task.defer(function()
            P.refreshQueued = false
            if RuntimeState.active then P.refresh() end
        end)
    end

    P.load = function()
        if not RuntimeState.active then return end
        local character = LocalPlayer.Character
        if not character then
            RuntimeState.PreviewStatus.Text = "WAITING FOR CHARACTER..."
            return
        end
        if P.model then P.model:Destroy(); P.model = nil end
        local previousArchivable = character.Archivable
        character.Archivable = true
        local cloneOk, clone = pcall(function() return character:Clone() end)
        character.Archivable = previousArchivable
        if not cloneOk or not clone then
            RuntimeState.PreviewStatus.Text = "AVATAR PREVIEW UNAVAILABLE"
            return
        end
        clone.Name = "CATScriptPreviewAvatar"
        for _, descendant in ipairs(clone:GetDescendants()) do
            if descendant.Name == "CATScript_KasaHat" or descendant.Name == "CATScript_SamuraiHat" then
                pcall(function() descendant:Destroy() end)
            elseif descendant.Name == "CATScript_AngelWings" then
                pcall(function() descendant:Destroy() end)
            elseif descendant:IsA("Script") or descendant:IsA("LocalScript") or descendant:IsA("ModuleScript") or descendant:IsA("Tool") then
                pcall(function() descendant:Destroy() end)
            elseif descendant:IsA("BasePart") then
                descendant.Anchored = true
                descendant.CanCollide = false
                descendant.CanTouch = false
                descendant.CanQuery = false
                pcall(function() descendant.LocalTransparencyModifier = 0 end)
            elseif descendant:IsA("Humanoid") then
                descendant.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None
                descendant.AutoRotate = false
            end
        end
        clone.Parent = P.World
        P.model = clone
        P.setModelRotation()

        local boundsOk, boundsCF, boundsSize = pcall(function() return clone:GetBoundingBox() end)
        local center = boundsOk and boundsCF.Position or Vector3.new(0, 1.2, 0)
        local height = boundsOk and boundsSize.Y or 5
        local distance = math.max(7.2, height * 1.85)
        P.Camera.CFrame = CFrame.lookAt(center + Vector3.new(0, 0.25, distance), center + Vector3.new(0, 0.1, 0))
        RuntimeState.PreviewStatus.Text = "AVATAR LOADED  /  DRAG TO ROTATE"
        P.scheduleRefresh()
    end

    P.dragInputType = nil
    P.viewportInputConnection = RuntimeState.PreviewViewport.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            P.dragging = true
            P.dragInputType = input.UserInputType
            -- Use the same coordinate source for the full drag to avoid a
            -- half-turn caused by mixing GUI and screen coordinates.
            P.dragX = input.UserInputType == Enum.UserInputType.Touch
                and input.Position.X
                or UserInputService:GetMouseLocation().X
        end
    end)
    P.inputChangedConnection = UserInputService.InputChanged:Connect(function(input)
        if not P.dragging then return end
        local isMouseMove = P.dragInputType == Enum.UserInputType.MouseButton1
            and input.UserInputType == Enum.UserInputType.MouseMovement
        local isTouchMove = P.dragInputType == Enum.UserInputType.Touch
            and input.UserInputType == Enum.UserInputType.Touch
        if isMouseMove or isTouchMove then
            local currentX = isTouchMove and input.Position.X or UserInputService:GetMouseLocation().X
            local delta = currentX - P.dragX
            P.dragX = currentX
            -- One horizontal viewport-width drag now equals a full 360° turn.
            -- The angle is intentionally unbounded so repeated drags keep spinning.
            local viewportWidth = math.max(RuntimeState.PreviewViewport.AbsoluteSize.X, 1)
            P.angle = P.angle + delta * ((math.pi * 2) / viewportWidth)
            P.setModelRotation()
        end
    end)
    P.inputEndedConnection = UserInputService.InputEnded:Connect(function(input)
        if (P.dragInputType == Enum.UserInputType.MouseButton1 and input.UserInputType == Enum.UserInputType.MouseButton1)
            or (P.dragInputType == Enum.UserInputType.Touch and input.UserInputType == Enum.UserInputType.Touch) then
            P.dragging = false
            P.dragInputType = nil
        end
    end)
    RuntimeState.previewCharacterConnection = LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.8)
        P.load()
    end)
    P.load()
end


local function spawnJumpCircle(character)
    if not jumpCircleEnabled or not character or not RuntimeState.active then return end
    local root = character:FindFirstChild("HumanoidRootPart")
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not root or not humanoid then return end

    clearJumpCircleEffects()
    local token = jumpCircleToken
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    rayParams.FilterDescendantsInstances = {character}
    rayParams.IgnoreWater = true
    local hit = workspace:Raycast(root.Position, Vector3.new(0, -24, 0), rayParams)
    local groundY
    if hit then
        groundY = hit.Position.Y + 0.07
    else
        groundY = root.Position.Y - humanoid.HipHeight - (root.Size.Y * 0.5) + 0.07
    end
    local center = Vector3.new(root.Position.X, groundY, root.Position.Z)
    local specs = buildJumpCircleSpecs(jumpCircleStyle)
    for _, spec in ipairs(specs) do
        local points = spec.points
        local subdivisions = spec.subdivisions or 1
        for index = 1, #points do
            local first = points[index]
            local second = points[(index % #points) + 1]
            for stepIndex = 0, subdivisions - 1 do
                local t1 = stepIndex / subdivisions
                local t2 = (stepIndex + 1) / subdivisions
                local p1 = first + (second - first) * t1
                local p2 = first + (second - first) * t2
                local part = Instance.new("Part")
                part.Name = "CATScript_JumpCircleSegment"
                part.Anchored = true
                part.CanCollide = false
                part.CanTouch = false
                part.CanQuery = false
                part.CastShadow = false
                part.Material = Enum.Material.Neon
                part.Color = jumpCircleColor
                part.Transparency = 1
                part.Size = Vector3.new(0.11, 0.055, 0.2)
                part.Parent = workspace
                table.insert(jumpCircleParts, {
                    part = part,
                    p1 = p1,
                    p2 = p2,
                    radius = spec.radius or 1,
                    delay = spec.delay or 0,
                })
            end
        end
    end

    local started = os.clock()
    local sizes = RuntimeState.jumpCircleSizes
    local duration = sizes.duration
    local latestDelay = 0
    for _, spec in ipairs(specs) do
        latestDelay = math.max(latestDelay, spec.delay or 0)
    end
    task.spawn(function()
        while RuntimeState.active and jumpCircleToken == token do
            local elapsed = os.clock() - started
            if elapsed > duration + latestDelay then break end
            for _, entry in ipairs(jumpCircleParts) do
                local part = entry.part
                if part and part.Parent then
                    local localAge = elapsed - entry.delay
                    if localAge < 0 then
                        part.Transparency = 1
                    else
                        local alpha = math.clamp(localAge / duration, 0, 1)
                        local radius = (sizes.radius + alpha * sizes.expansion) * entry.radius
                        local lift = sizes.height + alpha * sizes.height
                        local p1 = center + Vector3.new(entry.p1.X * radius, lift, entry.p1.Y * radius)
                        local p2 = center + Vector3.new(entry.p2.X * radius, lift, entry.p2.Y * radius)
                        local delta = p2 - p1
                        local midpoint = (p1 + p2) * 0.5
                        part.CFrame = CFrame.lookAt(midpoint, midpoint + delta.Unit, Vector3.yAxis)
                        part.Size = Vector3.new(sizes.thickness * (1 - alpha * 0.35), math.max(sizes.thickness * 0.5, 0.012), math.max(delta.Magnitude * 1.08, 0.04))
                        part.Color = jumpCircleColor
                        part.Transparency = math.clamp(0.08 + alpha * 0.92, 0, 1)
                    end
                end
            end
            RunService.Heartbeat:Wait()
        end
        if jumpCircleToken == token then
            for _, entry in ipairs(jumpCircleParts) do
                if entry.part then pcall(function() entry.part:Destroy() end) end
            end
            table.clear(jumpCircleParts)
        end
    end)
end

local function bindJumpCircleCharacter(character)
    if jumpCircleConnection then
        pcall(function() jumpCircleConnection:Disconnect() end)
        jumpCircleConnection = nil
    end
    if not character or not RuntimeState.active then return end
    local humanoid = character:FindFirstChildOfClass("Humanoid") or character:WaitForChild("Humanoid", 5)
    if not humanoid or not RuntimeState.active then return end
    jumpCircleConnection = humanoid.Jumping:Connect(function(isJumping)
        if isJumping and jumpCircleEnabled then
            spawnJumpCircle(character)
        end
    end)
end

local function updateJumpCircleToggle()
    tween(RuntimeState.JumpCircleToggleButton, {BackgroundColor3 = jumpCircleEnabled and COLORS.accent or COLORS.dim}, 0.12)
    tween(RuntimeState.JumpCircleToggleKnob, {Position = jumpCircleEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3)}, 0.12)
    RuntimeState.JumpCircleStatus.Text = jumpCircleEnabled and ("ENABLED  /  " .. string.upper(jumpCircleStyle)) or "DISABLED  /  LOCAL JUMP EFFECT"
    RuntimeState.JumpCircleStatus.TextColor3 = jumpCircleEnabled and COLORS.accent or COLORS.dim
end

local function updateJumpCircleStyleButton()
    RuntimeState.JumpCircleStyleButton.Text = jumpCircleStyle .. "  ▾"
    for styleName, styleButton in pairs(RuntimeState.JumpCircleStyleButtons or {}) do
        styleButton.BackgroundColor3 = styleName == jumpCircleStyle and COLORS.accent or COLORS.background
    end
end

local function updateTrailToggle()
    tween(RuntimeState.TrailToggleButton, {BackgroundColor3 = trailEnabled and COLORS.accent or COLORS.dim}, 0.12)
    tween(RuntimeState.TrailToggleKnob, {Position = trailEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3)}, 0.12)
    RuntimeState.TrailStatus.Text = trailEnabled and "TRAIL ACTIVE  /  LOCAL CHARACTER" or "DISABLED"
    RuntimeState.TrailStatus.TextColor3 = trailEnabled and COLORS.accent or COLORS.dim
end

local function updateHoverToggle()
    tween(RuntimeState.HoverToggleButton, {
        BackgroundColor3 = hoverEnabled and COLORS.accent or COLORS.dim,
    }, 0.12)
    tween(RuntimeState.HoverToggleKnob, {
        Position = hoverEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    }, 0.12)
    RuntimeState.HoverStatus.Text = hoverEnabled and "GLOWING CORNERS  /  NEON RING" or "DISABLED"
    RuntimeState.HoverStatus.TextColor3 = hoverEnabled and COLORS.accent or COLORS.dim
end

local saveRevision = 0
local function updateAutoSaveToggle()
    tween(RuntimeState.AutoSaveToggleButton, {
        BackgroundColor3 = autoSaveEnabled and COLORS.accent or COLORS.dim,
    }, 0.12)
    tween(RuntimeState.AutoSaveToggleKnob, {
        Position = autoSaveEnabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3),
    }, 0.12)
    RuntimeState.AutoSaveStatus.Text = autoSaveEnabled and "ENABLED  /  SAVING CONFIG" or "DISABLED  /  NOT SAVING"
    RuntimeState.AutoSaveStatus.TextColor3 = autoSaveEnabled and COLORS.accent or COLORS.dim
end

local function writeConfigNow()
    if type(writefile) ~= "function" then
        RuntimeState.AutoSaveStatus.Text = "FILESYSTEM API NOT AVAILABLE"
        RuntimeState.AutoSaveStatus.TextColor3 = Color3.fromRGB(230, 165, 90)
        return false
    end

    local r = math.floor(hoverColor.R * 255 + 0.5)
    local g = math.floor(hoverColor.G * 255 + 0.5)
    local b = math.floor(hoverColor.B * 255 + 0.5)
    local data = {
        version = 2,
        autoSave = autoSaveEnabled,
        antiAfk = RuntimeState.Utility.antiAfkEnabled,
        noTexture = RuntimeState.Utility.noTextureEnabled,
        noGui = RuntimeState.Utility.noGuiEnabled,
        autoSessionReward = RuntimeState.Utility.autoSessionRewardEnabled == true,
        autoSkipCrates = RuntimeState.Utility.autoSkipCratesEnabled == true,
        noFog = RuntimeState.NoFog.enabled,
        teleportSave = (RuntimeState.TeleportSave and typeof(RuntimeState.TeleportSave.position) == "Vector3") and {
            x = RuntimeState.TeleportSave.position.X,
            y = RuntimeState.TeleportSave.position.Y,
            z = RuntimeState.TeleportSave.position.Z,
        } or nil,
        freecamSpeed = RuntimeState.Utility.freecam.speed,
        flySpeed = RuntimeState.Utility.fly.speed,
        player = {walkSpeed=RuntimeState.PlayerTools.walkSpeed, jumpPower=RuntimeState.PlayerTools.jumpPower, vehicleFlySpeed=RuntimeState.PlayerTools.vehicleFlySpeed, fov=RuntimeState.PlayerTools.fov, aimFov=RuntimeState.PlayerTools.aimFov, aimSmoothness=RuntimeState.PlayerTools.aimSmoothness, aimRange=RuntimeState.PlayerTools.aimRange},
        keystrokesEnabled = RuntimeState.Keystrokes.enabled,
        keystrokesPosition = {
            xScale = RuntimeState.KeystrokesWidget.Position.X.Scale,
            xOffset = RuntimeState.KeystrokesWidget.Position.X.Offset,
            yScale = RuntimeState.KeystrokesWidget.Position.Y.Scale,
            yOffset = RuntimeState.KeystrokesWidget.Position.Y.Offset,
        },
        water = {enabled=RuntimeState.Water.enabled, transparency=RuntimeState.Water.transparency},
        menuKey = menuKeyCode.Name,
        backgroundAssetId = backgroundAssetId,
        menuTransparency = RuntimeState.menuTransparency,
        fontName = RuntimeState.selectedFontName,
        customCursor = {
            enabled = cursorSettings.enabled,
            style = cursorSettings.style,
            colorName = cursorSettings.colorName,
            color = {
                r = math.floor(cursorSettings.color.R * 255 + 0.5),
                g = math.floor(cursorSettings.color.G * 255 + 0.5),
                b = math.floor(cursorSettings.color.B * 255 + 0.5),
            },
            size = cursorSettings.size,
        },
        themeColors = {},
        hoverEnabled = false, -- legacy key retained only for config compatibility.
        hoverColor = {r = r, g = g, b = b},
        trailEnabled = trailEnabled,
        trailColor = {
            r = math.floor(trailColor.R * 255 + 0.5),
            g = math.floor(trailColor.G * 255 + 0.5),
            b = math.floor(trailColor.B * 255 + 0.5),
        },
        trailSizes = {
            startWidth = trailStartWidth,
            endWidth = trailEndWidth,
            lifetime = trailLifetime,
            minLength = trailMinLength,
            spread = trailSpread,
        },
        jumpCircleEnabled = jumpCircleEnabled,
        jumpCircleStyle = jumpCircleStyle,
        jumpCircleSizes = RuntimeState.jumpCircleSizes,
        jumpCircleColor = {
            r = math.floor(jumpCircleColor.R * 255 + 0.5),
            g = math.floor(jumpCircleColor.G * 255 + 0.5),
            b = math.floor(jumpCircleColor.B * 255 + 0.5),
        },
        customAura = {
            enabled = RuntimeState.CustomAura.enabled,
            style = RuntimeState.CustomAura.style,
            size = RuntimeState.CustomAura.size,
            offsetX = RuntimeState.CustomAura.offsetX,
            offsetY = RuntimeState.CustomAura.offsetY,
            offsetZ = RuntimeState.CustomAura.offsetZ,
            color = {
                r = math.floor(RuntimeState.CustomAura.color.R * 255 + 0.5),
                g = math.floor(RuntimeState.CustomAura.color.G * 255 + 0.5),
                b = math.floor(RuntimeState.CustomAura.color.B * 255 + 0.5),
            },
        },
        skyboxPreset = RuntimeState.Skybox and RuntimeState.Skybox.selectedName or "",
        weather = {
            Snow = {
                enabled = RuntimeState.Weather.settings.Snow.enabled,
                color = {r=math.floor(RuntimeState.Weather.settings.Snow.color.R*255+0.5), g=math.floor(RuntimeState.Weather.settings.Snow.color.G*255+0.5), b=math.floor(RuntimeState.Weather.settings.Snow.color.B*255+0.5)},
                settings = {amount=RuntimeState.Weather.settings.Snow.amount, speed=RuntimeState.Weather.settings.Snow.speed, size=RuntimeState.Weather.settings.Snow.size, radius=RuntimeState.Weather.settings.Snow.radius, wind=RuntimeState.Weather.settings.Snow.wind},
            },
            Rain = {
                enabled = RuntimeState.Weather.settings.Rain.enabled,
                color = {r=math.floor(RuntimeState.Weather.settings.Rain.color.R*255+0.5), g=math.floor(RuntimeState.Weather.settings.Rain.color.G*255+0.5), b=math.floor(RuntimeState.Weather.settings.Rain.color.B*255+0.5)},
                settings = {amount=RuntimeState.Weather.settings.Rain.amount, speed=RuntimeState.Weather.settings.Rain.speed, size=RuntimeState.Weather.settings.Rain.size, radius=RuntimeState.Weather.settings.Rain.radius, wind=RuntimeState.Weather.settings.Rain.wind},
            },
            Stars = {
                enabled = RuntimeState.Weather.settings.Stars.enabled,
                color = {r=math.floor(RuntimeState.Weather.settings.Stars.color.R*255+0.5), g=math.floor(RuntimeState.Weather.settings.Stars.color.G*255+0.5), b=math.floor(RuntimeState.Weather.settings.Stars.color.B*255+0.5)},
                settings = {amount=RuntimeState.Weather.settings.Stars.amount, speed=RuntimeState.Weather.settings.Stars.speed, size=RuntimeState.Weather.settings.Stars.size, radius=RuntimeState.Weather.settings.Stars.radius, wind=RuntimeState.Weather.settings.Stars.wind},
            },
        },
        metrics = {
            fps = metricSettings.fps,
            ping = metricSettings.ping,
            gpu = metricSettings.gpu,
            memory = metricSettings.memory,
        },
        metricsPosition = {
            xScale = RuntimeState.PerformanceWidget.Position.X.Scale,
            xOffset = RuntimeState.PerformanceWidget.Position.X.Offset,
            yScale = RuntimeState.PerformanceWidget.Position.Y.Scale,
            yOffset = RuntimeState.PerformanceWidget.Position.Y.Offset,
        },
        espColors = {
            outlineColor = {r = math.floor(RuntimeState.ESPConfig.outlineColor.R * 255 + 0.5), g = math.floor(RuntimeState.ESPConfig.outlineColor.G * 255 + 0.5), b = math.floor(RuntimeState.ESPConfig.outlineColor.B * 255 + 0.5)},
            fillColor = {r = math.floor(RuntimeState.ESPConfig.fillColor.R * 255 + 0.5), g = math.floor(RuntimeState.ESPConfig.fillColor.G * 255 + 0.5), b = math.floor(RuntimeState.ESPConfig.fillColor.B * 255 + 0.5)},
            textColor = {r = math.floor(RuntimeState.ESPConfig.textColor.R * 255 + 0.5), g = math.floor(RuntimeState.ESPConfig.textColor.G * 255 + 0.5), b = math.floor(RuntimeState.ESPConfig.textColor.B * 255 + 0.5)},
            healthColor = {r = math.floor(RuntimeState.ESPConfig.healthColor.R * 255 + 0.5), g = math.floor(RuntimeState.ESPConfig.healthColor.G * 255 + 0.5), b = math.floor(RuntimeState.ESPConfig.healthColor.B * 255 + 0.5)},
            cornerBoxColor = {r = math.floor(RuntimeState.ESPConfig.cornerBoxColor.R * 255 + 0.5), g = math.floor(RuntimeState.ESPConfig.cornerBoxColor.G * 255 + 0.5), b = math.floor(RuntimeState.ESPConfig.cornerBoxColor.B * 255 + 0.5)},
            cornerBoxGlowColor = {r = math.floor(RuntimeState.ESPConfig.cornerBoxGlowColor.R * 255 + 0.5), g = math.floor(RuntimeState.ESPConfig.cornerBoxGlowColor.G * 255 + 0.5), b = math.floor(RuntimeState.ESPConfig.cornerBoxGlowColor.B * 255 + 0.5)},
        },
        espCornerBox = {
            enabled = RuntimeState.ESPConfig.cornerBox,
            widthScale = RuntimeState.ESPConfig.cornerBoxWidthScale,
            heightScale = RuntimeState.ESPConfig.cornerBoxHeightScale,
            length = RuntimeState.ESPConfig.cornerBoxLength,
            thickness = RuntimeState.ESPConfig.cornerBoxThickness,
            opacity = RuntimeState.ESPConfig.cornerBoxOpacity,
            glow = RuntimeState.ESPConfig.cornerBoxGlow,
            padding = RuntimeState.ESPConfig.cornerBoxPadding,
        },
        floatingWindowAppearance = (function()
            local result = {}
            for windowName, config in pairs(RuntimeState.FloatingWindowAppearance.settings) do
                result[windowName] = {
                    scale = config.scale,
                    style = config.style,
                    color = {
                        r = math.floor(config.color.R * 255 + 0.5),
                        g = math.floor(config.color.G * 255 + 0.5),
                        b = math.floor(config.color.B * 255 + 0.5),
                    },
                }
            end
            return result
        end)(),
    }

    for _, themeKey in ipairs(RuntimeState.THEME_KEYS) do
        local themeColor = COLORS[themeKey]
        data.themeColors[themeKey] = {
            r = math.floor(themeColor.R * 255 + 0.5),
            g = math.floor(themeColor.G * 255 + 0.5),
            b = math.floor(themeColor.B * 255 + 0.5),
        }
    end

    local ok, err = pcall(function()
        writefile(CONFIG_FILE, HttpService:JSONEncode(data))
    end)
    if ok then
        RuntimeState.AutoSaveStatus.Text = autoSaveEnabled and "SAVED  /  LOCAL CONFIG" or "AUTOSAVE DISABLED"
        RuntimeState.AutoSaveStatus.TextColor3 = autoSaveEnabled and COLORS.accent or COLORS.dim
        return true
    end

    RuntimeState.AutoSaveStatus.Text = "SAVE FAILED  /  CHECK EXECUTOR"
    RuntimeState.AutoSaveStatus.TextColor3 = Color3.fromRGB(230, 120, 120)
    warn("[CATScript] Could not save config: " .. tostring(err))
    return false
end

scheduleConfigSave = function(force)
    if not autoSaveEnabled and not force then return end
    saveRevision += 1
    local revision = saveRevision
    task.delay(force and 0 or 0.45, function()
        if revision ~= saveRevision or not RuntimeState.active then return end
        if autoSaveEnabled or force then
            writeConfigNow()
        end
    end)
end

RuntimeState.AutoSaveToggleButton.MouseButton1Click:Connect(function()
    autoSaveEnabled = not autoSaveEnabled
    updateAutoSaveToggle()
    -- Persist the new toggle state once so OFF remains OFF after reinjection.
    if autoSaveEnabled then
        RuntimeEnvironment.__CATScriptMenuKeyName = menuKeyCode.Name
    end
    if scheduleConfigSave then scheduleConfigSave(true) end
end)
updateAutoSaveToggle()

RuntimeState.refreshCursorColorPicker = function()
    RuntimeState.CursorColorHSV.h, RuntimeState.CursorColorHSV.s, RuntimeState.CursorColorHSV.v = cursorSettings.color:ToHSV()
    RuntimeState.CursorColorSVArea.BackgroundColor3 = Color3.fromHSV(RuntimeState.CursorColorHSV.h, 1, 1)
    RuntimeState.CursorColorSVMarker.Position = UDim2.fromScale(RuntimeState.CursorColorHSV.s, 1 - RuntimeState.CursorColorHSV.v)
    RuntimeState.CursorColorHueMarker.Position = UDim2.new(0.5, 0, RuntimeState.CursorColorHSV.h, 0)
    RuntimeState.CursorColorHexBox.Text = string.format("#%02X%02X%02X", math.floor(cursorSettings.color.R * 255 + 0.5), math.floor(cursorSettings.color.G * 255 + 0.5), math.floor(cursorSettings.color.B * 255 + 0.5))
    RuntimeState.CursorColorRGBBox.Text = string.format("%d, %d, %d", math.floor(cursorSettings.color.R * 255 + 0.5), math.floor(cursorSettings.color.G * 255 + 0.5), math.floor(cursorSettings.color.B * 255 + 0.5))
end

RuntimeState.applyCustomCursorColor = function(colorValue)
    cursorSettings.color = colorValue
    cursorSettings.colorName = "Custom"
    for presetName, presetColor in pairs(cursorPalette) do
        if math.abs(presetColor.R - colorValue.R) < 0.002
            and math.abs(presetColor.G - colorValue.G) < 0.002
            and math.abs(presetColor.B - colorValue.B) < 0.002 then
            cursorSettings.colorName = presetName
            break
        end
    end
    RuntimeState.refreshCursorSettings()
    RuntimeState.refreshCursorColorPicker()
    if scheduleConfigSave then scheduleConfigSave() end
end

RuntimeState.updateCursorColorFromSV = function(pointer)
    local origin = RuntimeState.CursorColorSVArea.AbsolutePosition
    local bounds = RuntimeState.CursorColorSVArea.AbsoluteSize
    if bounds.X <= 0 or bounds.Y <= 0 then return end
    RuntimeState.CursorColorHSV.s = math.clamp((pointer.X - origin.X) / bounds.X, 0, 1)
    RuntimeState.CursorColorHSV.v = 1 - math.clamp((pointer.Y - origin.Y) / bounds.Y, 0, 1)
    RuntimeState.applyCustomCursorColor(Color3.fromHSV(RuntimeState.CursorColorHSV.h, RuntimeState.CursorColorHSV.s, RuntimeState.CursorColorHSV.v))
end

RuntimeState.updateCursorHueFromPointer = function(pointer)
    local origin = RuntimeState.CursorColorHueInput.AbsolutePosition
    local bounds = RuntimeState.CursorColorHueInput.AbsoluteSize
    if bounds.Y <= 0 then return end
    RuntimeState.CursorColorHSV.h = math.clamp((pointer.Y - origin.Y) / bounds.Y, 0, 1)
    RuntimeState.applyCustomCursorColor(Color3.fromHSV(RuntimeState.CursorColorHSV.h, RuntimeState.CursorColorHSV.s, RuntimeState.CursorColorHSV.v))
end

RuntimeState.CursorColorDragMode = nil
RuntimeState.CursorColorSVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CursorColorDragMode = "sv"
        RuntimeState.updateCursorColorFromSV(getColorEditorPointer(input))
    end
end)
RuntimeState.CursorColorHueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CursorColorDragMode = "hue"
        RuntimeState.updateCursorHueFromPointer(getColorEditorPointer(input))
    end
end)
RuntimeState.cursorColorInputChanged = UserInputService.InputChanged:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
    if not RuntimeState.CursorColorDragMode then return end
    local pointer = getColorEditorPointer(input)
    if RuntimeState.CursorColorDragMode == "sv" then
        RuntimeState.updateCursorColorFromSV(pointer)
    elseif RuntimeState.CursorColorDragMode == "hue" then
        RuntimeState.updateCursorHueFromPointer(pointer)
    end
end)
RuntimeState.cursorColorInputEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CursorColorDragMode = nil
    end
end)
RuntimeState.CursorColorHexBox.FocusLost:Connect(function()
    local cleanedHex = RuntimeState.CursorColorHexBox.Text:gsub("#", ""):match("^%x%x%x%x%x%x$")
    if cleanedHex then
        local numeric = tonumber(cleanedHex, 16)
        RuntimeState.applyCustomCursorColor(Color3.fromRGB(math.floor(numeric / 65536) % 256, math.floor(numeric / 256) % 256, numeric % 256))
    else
        RuntimeState.refreshCursorColorPicker()
    end
end)
RuntimeState.CursorColorRGBBox.FocusLost:Connect(function()
    local red, green, blue = RuntimeState.CursorColorRGBBox.Text:match("^%s*(%d+)%s*,%s*(%d+)%s*,%s*(%d+)%s*$")
    if red and green and blue then
        RuntimeState.applyCustomCursorColor(Color3.fromRGB(math.clamp(tonumber(red), 0, 255), math.clamp(tonumber(green), 0, 255), math.clamp(tonumber(blue), 0, 255)))
    else
        RuntimeState.refreshCursorColorPicker()
    end
end)
RuntimeState.CursorColorPickerButton.MouseButton1Click:Connect(function()
    RuntimeState.refreshCursorColorPicker()
    RuntimeState.openFloatingColorEditor(RuntimeState.CursorColorPicker, 8)
end)

RuntimeState.refreshCursorSettings = function()
    local enabled = cursorSettings.enabled
    tween(RuntimeState.CursorToggleButton, {BackgroundColor3 = enabled and COLORS.accent or COLORS.dim}, 0.12)
    tween(RuntimeState.CursorToggleKnob, {Position = enabled and UDim2.fromOffset(21, 3) or UDim2.fromOffset(4, 3)}, 0.12)
    RuntimeState.CursorStatus.Text = enabled and "ВКЛЮЧЁН" or "ВЫКЛЮЧЕН"
    RuntimeState.CursorStatus.TextColor3 = enabled and COLORS.accent or COLORS.dim
    for styleName, button in pairs(RuntimeState.CursorStyleButtons) do
        local selected = cursorSettings.style == styleName
        button.BackgroundColor3 = selected and COLORS.accent or COLORS.panel
        button.TextColor3 = selected and COLORS.white or COLORS.text
    end
    for colorName, entry in pairs(RuntimeState.CursorColorButtons) do
        local selected = cursorSettings.colorName == colorName
        entry.outline.Color = selected and COLORS.white or COLORS.outline
        entry.outline.Transparency = selected and 0 or 0.35
        entry.outline.Thickness = selected and 2 or 1
    end
    RuntimeState.CursorColorPickerButton.BackgroundColor3 = cursorSettings.color
    RuntimeState.CursorColorPickerButtonOutline.Color = cursorSettings.colorName == "Custom" and COLORS.white or COLORS.outline
    RuntimeState.CursorColorPickerButtonOutline.Thickness = cursorSettings.colorName == "Custom" and 2 or 1
    RuntimeState.CursorSizeValue.Text = tostring(cursorSettings.size) .. " px"
    RuntimeState.rebuildCustomCursor()
    RuntimeState.CustomCursorOverlay.Visible = enabled
end

RuntimeState.CursorToggleButton.MouseButton1Click:Connect(function()
    RuntimeState.setCustomCursorEnabled(not cursorSettings.enabled)
    RuntimeState.refreshCursorSettings()
    if scheduleConfigSave then scheduleConfigSave() end
end)
for styleName, styleButton in pairs(RuntimeState.CursorStyleButtons) do
    styleButton.MouseButton1Click:Connect(function()
        cursorSettings.style = styleName
        RuntimeState.refreshCursorSettings()
        if scheduleConfigSave then scheduleConfigSave() end
    end)
end
for colorName, colorEntry in pairs(RuntimeState.CursorColorButtons) do
    colorEntry.button.MouseButton1Click:Connect(function()
        cursorSettings.colorName = colorName
        cursorSettings.color = cursorPalette[colorName]
        RuntimeState.refreshCursorSettings()
        RuntimeState.refreshCursorColorPicker()
        if scheduleConfigSave then scheduleConfigSave() end
    end)
end
RuntimeState.CursorSizeMinus.MouseButton1Click:Connect(function()
    cursorSettings.size = math.clamp(cursorSettings.size - 2, 14, 40)
    RuntimeState.refreshCursorSettings()
    if scheduleConfigSave then scheduleConfigSave() end
end)
RuntimeState.CursorSizePlus.MouseButton1Click:Connect(function()
    cursorSettings.size = math.clamp(cursorSettings.size + 2, 14, 40)
    RuntimeState.refreshCursorSettings()
    if scheduleConfigSave then scheduleConfigSave() end
end)
RuntimeState.refreshCursorSettings()
RuntimeState.refreshCursorColorPicker()
if cursorSettings.enabled then RuntimeState.setCustomCursorEnabled(true) end

RuntimeState.updateMetricToggle = function(key)
    local control = RuntimeState.MetricToggleControls[key]
    if not control then return end
    local enabled = metricSettings[key]
    tween(control.button, {
        BackgroundColor3 = enabled and COLORS.accent or COLORS.dim,
    }, 0.12)
    tween(control.knob, {
        Position = enabled and UDim2.fromOffset(20, 3) or UDim2.fromOffset(3, 3),
    }, 0.12)
end

RuntimeState.updatePerformanceWidget = function(fpsValue)
    local values = {}
    values.fps = tostring(fpsValue or "--")

    local pingText = "N/A"
    if StatsService then
        local ok, value = pcall(function()
            local network = StatsService:FindFirstChild("Network")
            local serverStats = network and network:FindFirstChild("ServerStatsItem")
            local pingItem = serverStats and serverStats:FindFirstChild("Data Ping")
            if not pingItem then return nil end
            local numeric = pingItem:GetValue()
            if type(numeric) == "number" then return math.floor(numeric + 0.5) end
            local valueString = pingItem:GetValueString()
            local parsed = type(valueString) == "string" and tonumber(valueString:match("[%d%.]+"))
            return parsed and math.floor(parsed + 0.5) or nil
        end)
        if ok and value then pingText = tostring(value) .. " ms" end
    end
    values.ping = pingText

    -- Stats exposes GPU render frame time, not GPU utilization percentage.
    -- RenderGPUFrameTime is a frame-duration measurement; display it in ms.
    local gpuText = "N/A"
    if StatsService then
        local ok, value = pcall(function()
            return StatsService.RenderGPUFrameTime
        end)
        if ok and type(value) == "number" and value >= 0 then
            gpuText = string.format("%.1f ms", value * 1000)
        end
    end
    values.gpu = gpuText

    local memoryText = "N/A"
    if StatsService then
        local ok, value = pcall(function()
            return StatsService:GetTotalMemoryUsageMb()
        end)
        if ok and type(value) == "number" then
            if value >= 1024 then
                memoryText = string.format("%.1f GB", value / 1024)
            else
                memoryText = string.format("%.0f MB", value)
            end
        end
    end
    values.memory = memoryText

    local visibleCount = 0
    for _, key in ipairs(RuntimeState.PERFORMANCE_ORDER) do
        local enabled = metricSettings[key] == true
        local cell = RuntimeState.PerformanceCells[key]
        cell.frame.Visible = enabled
        if enabled then
            local left = RuntimeState.PERFORMANCE_PADDING + visibleCount * (RuntimeState.PERFORMANCE_CELL_WIDTH + RuntimeState.PERFORMANCE_CELL_GAP)
            cell.frame.Position = UDim2.fromOffset(left, 17)
            cell.value.Text = values[key] or "N/A"
            visibleCount += 1
        end
    end

    for index, divider in ipairs(RuntimeState.PerformanceDividers) do
        local visible = index < visibleCount
        divider.Visible = visible
        if visible then
            local x = RuntimeState.PERFORMANCE_PADDING + index * RuntimeState.PERFORMANCE_CELL_WIDTH
                + (index - 1) * RuntimeState.PERFORMANCE_CELL_GAP + 3
            divider.Position = UDim2.fromOffset(x, 24)
        end
    end

    RuntimeState.PerformanceWidget.Visible = visibleCount > 0
    if visibleCount == 0 then return end

    local width = RuntimeState.PERFORMANCE_PADDING * 2
        + visibleCount * RuntimeState.PERFORMANCE_CELL_WIDTH
        + math.max(0, visibleCount - 1) * RuntimeState.PERFORMANCE_CELL_GAP
    local panelWidth = math.max(176, width)
    RuntimeState.PerformanceWidget.Size = UDim2.fromOffset(panelWidth + 12, 56)
    RuntimeState.PerformancePanel.Size = UDim2.fromOffset(panelWidth + 12, 48)
end

for key, control in pairs(RuntimeState.MetricToggleControls) do
    RuntimeState.updateMetricToggle(key)
    control.button.MouseButton1Click:Connect(function()
        metricSettings[key] = not metricSettings[key]
        RuntimeState.updateMetricToggle(key)
        RuntimeState.updatePerformanceWidget()
        if scheduleConfigSave then scheduleConfigSave() end
    end)
end

RuntimeState.perfElapsed = 0
RuntimeState.perfFrames = 0
RuntimeState.latestFPS = 0
RuntimeState.performanceRenderConnection = nil
RuntimeState.performanceRenderConnection = RunService.RenderStepped:Connect(function(deltaTime)
    RuntimeState.perfElapsed += deltaTime
    RuntimeState.perfFrames += 1
    if RuntimeState.perfElapsed >= 0.5 then
        RuntimeState.latestFPS = math.floor((RuntimeState.perfFrames / math.max(RuntimeState.perfElapsed, 0.001)) + 0.5)
        RuntimeState.perfElapsed = 0
        RuntimeState.perfFrames = 0
        RuntimeState.updatePerformanceWidget(RuntimeState.latestFPS)
    end
end)
RuntimeState.updatePerformanceWidget("--")

-- Capture the default palette ownership first, then apply saved colors role by role.
-- This prevents old configs (including configs with duplicate RGB values) from
-- merging unrelated theme entries into one shared color group.
RuntimeState.applySavedThemeColors = function()
    RuntimeState.ensureThemeColorRegistry()
    for _, themeKey in ipairs(RuntimeState.THEME_KEYS) do
        local savedColor = type(SavedConfig.themeColors) == "table" and SavedConfig.themeColors[themeKey] or nil
        if type(savedColor) == "table"
            and type(savedColor.r) == "number"
            and type(savedColor.g) == "number"
            and type(savedColor.b) == "number" then
            RuntimeState.applyThemeColor(themeKey, Color3.fromRGB(
                math.clamp(math.floor(savedColor.r + 0.5), 0, 255),
                math.clamp(math.floor(savedColor.g + 0.5), 0, 255),
                math.clamp(math.floor(savedColor.b + 0.5), 0, 255)
            ), false)
        end
    end
    RuntimeState.refreshThemePicker()
end
RuntimeState.applySavedThemeColors()

-- Smoothly fade and slightly slide the complete menu when it opens/closes.
-- Transparency targets are captured from the live UI, so custom theme opacity
-- and individual widget styling are preserved instead of being reset.
RuntimeState.menuFadeDuration = 0.38
RuntimeState.menuFadeTargets = nil
RuntimeState.menuFadeTweens = {}
RuntimeState.menuTransitionToken = 0
RuntimeState.menuTransitionState = "hidden"
RuntimeState.menuRestorePosition = Window.Position

local function captureMenuFadeTargets()
    local targets = {}
    local function captureObject(object)
        local properties = {}
        if object:IsA("GuiObject") then
            properties.BackgroundTransparency = object.BackgroundTransparency
        end
        if object:IsA("TextLabel") or object:IsA("TextButton") or object:IsA("TextBox") then
            properties.TextTransparency = object.TextTransparency
            properties.TextStrokeTransparency = object.TextStrokeTransparency
        end
        if object:IsA("ImageLabel") or object:IsA("ImageButton") then
            properties.ImageTransparency = object.ImageTransparency
        end
        if object:IsA("ScrollingFrame") then
            properties.ScrollBarImageTransparency = object.ScrollBarImageTransparency
        end
        if object:IsA("UIStroke") then
            properties.Transparency = object.Transparency
        end
        if next(properties) then
            targets[object] = properties
        end
    end

    captureObject(Window)
    for _, object in ipairs(Window:GetDescendants()) do
        captureObject(object)
    end
    return targets
end

local function cancelMenuFadeTweens()
    for _, activeTween in ipairs(RuntimeState.menuFadeTweens) do
        pcall(function() activeTween:Cancel() end)
    end
    table.clear(RuntimeState.menuFadeTweens)
end

local function tweenMenuTransparency(toHidden, token)
    local targets = RuntimeState.menuFadeTargets or {}
    local info = TweenInfo.new(RuntimeState.menuFadeDuration, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    for object, originalProperties in pairs(targets) do
        if object and object.Parent then
            local goal = {}
            for propertyName, originalValue in pairs(originalProperties) do
                goal[propertyName] = toHidden and 1 or originalValue
            end
            local ok, activeTween = pcall(function()
                return TweenService:Create(object, info, goal)
            end)
            if ok and activeTween then
                table.insert(RuntimeState.menuFadeTweens, activeTween)
                activeTween:Play()
            end
        end
    end
end

RuntimeState.setMenuVisible = function(shouldShow)
    shouldShow = shouldShow == true
    if shouldShow and RuntimeState.menuTransitionState == "shown" and Window.Visible then
        return
    end
    if not shouldShow and not Window.Visible then
        RuntimeState.menuTransitionState = "hidden"
        return
    end

    RuntimeState.menuTransitionToken = RuntimeState.menuTransitionToken + 1
    local token = RuntimeState.menuTransitionToken
    cancelMenuFadeTweens()

    if shouldShow then
        if not RuntimeState.menuFadeTargets then
            RuntimeState.menuFadeTargets = captureMenuFadeTargets()
        end
        if RuntimeState.menuTransitionState == "hidden" or not RuntimeState.menuRestorePosition then
            RuntimeState.menuRestorePosition = Window.Position
        end
        local targetPosition = RuntimeState.menuRestorePosition or Window.Position
        Window.Position = UDim2.new(
            targetPosition.X.Scale, targetPosition.X.Offset,
            targetPosition.Y.Scale, targetPosition.Y.Offset + 8
        )
        for object, properties in pairs(RuntimeState.menuFadeTargets) do
            if object and object.Parent then
                for propertyName in pairs(properties) do
                    pcall(function() object[propertyName] = 1 end)
                end
            end
        end
        Window.Visible = true
        RuntimeState.menuTransitionState = "opening"
        tweenMenuTransparency(false, token)
        local positionTween = TweenService:Create(
            Window,
            TweenInfo.new(RuntimeState.menuFadeDuration, Enum.EasingStyle.Quart, Enum.EasingDirection.Out),
            {Position = targetPosition}
        )
        table.insert(RuntimeState.menuFadeTweens, positionTween)
        positionTween:Play()
        task.delay(RuntimeState.menuFadeDuration + 0.04, function()
            if RuntimeState.active and RuntimeState.menuTransitionToken == token then
                RuntimeState.menuTransitionState = "shown"
                Window.Position = targetPosition
                table.clear(RuntimeState.menuFadeTweens)
            end
        end)
    else
        if RuntimeState.menuTransitionState == "shown" then
            RuntimeState.menuFadeTargets = captureMenuFadeTargets()
            RuntimeState.menuRestorePosition = Window.Position
        end
        local targetPosition = RuntimeState.menuRestorePosition or Window.Position
        RuntimeState.menuTransitionState = "closing"
        tweenMenuTransparency(true, token)
        local positionTween = TweenService:Create(
            Window,
            TweenInfo.new(RuntimeState.menuFadeDuration, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
            {Position = UDim2.new(
                targetPosition.X.Scale, targetPosition.X.Offset,
                targetPosition.Y.Scale, targetPosition.Y.Offset + 8
            )}
        )
        table.insert(RuntimeState.menuFadeTweens, positionTween)
        positionTween:Play()
        task.delay(RuntimeState.menuFadeDuration + 0.05, function()
            if RuntimeState.active and RuntimeState.menuTransitionToken == token then
                Window.Visible = false
                Window.Position = targetPosition
                RuntimeState.menuTransitionState = "hidden"
                table.clear(RuntimeState.menuFadeTweens)
            end
        end)
    end
end

-- Polished startup reveal with an animated progress bar and soft fade.
RuntimeState.applyMenuTransparency()

RuntimeState.playLoadingScreen = function()
    task.spawn(function()
        local loadingSteps = {
            {progress = 0.08, message = "INITIALIZING CORE", line = "[BOOT] CATScript Visual runtime starting..."},
            {progress = 0.20, message = "LINKING CLIENT SERVICES", line = "[ OK ] Roblox client services linked"},
            {progress = 0.34, message = "BUILDING USER INTERFACE", line = "[ OK ] Dashboard and navigation ready"},
            {progress = 0.49, message = "LOADING PLAYER MODULES", line = "[ OK ] Player controls initialized"},
            {progress = 0.64, message = "LOADING VISUAL MODULES", line = "[ OK ] Visual effects initialized"},
            {progress = 0.78, message = "LOADING ESP MODULE", line = "[ OK ] ESP components initialized"},
            {progress = 0.91, message = "APPLYING LOCAL SETTINGS", line = "[ CFG ] Local preferences applied"},
            {progress = 1.00, message = "STARTUP COMPLETE", line = "[ DONE ] CATScript is ready. Welcome!"},
        }
        for stepIndex, step in ipairs(loadingSteps) do
            if not RuntimeState.active or not RuntimeState.LoadingOverlay or not RuntimeState.LoadingOverlay.Parent then return end
            local logLine = RuntimeState.LoadingLogLines and RuntimeState.LoadingLogLines[stepIndex]
            if logLine then
                logLine.Text = step.line
                logLine.TextColor3 = stepIndex == #loadingSteps and COLORS.accent or COLORS.text
                logLine.TextTransparency = 1
                tween(logLine, {TextTransparency = 0}, 0.18)
            end
            RuntimeState.LoadingStatus.Text = step.message
            RuntimeState.LoadingPercent.Text = string.format("%02d%%", math.floor(step.progress * 100 + 0.5))
            tween(RuntimeState.LoadingFill, {Size = UDim2.new(step.progress, 0, 1, 0)}, 0.34)
            task.wait(stepIndex == #loadingSteps and 0.48 or 0.30)
        end

        tween(RuntimeState.LoadingCard, {Position = UDim2.fromScale(0.5, 0.48)}, 0.26)
        task.wait(0.10)
        for _, loadingObject in ipairs(RuntimeState.LoadingOverlay:GetDescendants()) do
            if loadingObject:IsA("TextLabel") or loadingObject:IsA("TextButton") or loadingObject:IsA("TextBox") then
                tween(loadingObject, {TextTransparency = 1, BackgroundTransparency = 1}, 0.25)
            elseif loadingObject:IsA("GuiObject") then
                tween(loadingObject, {BackgroundTransparency = 1}, 0.25)
            elseif loadingObject:IsA("UIStroke") then
                tween(loadingObject, {Transparency = 1}, 0.25)
            end
        end
        tween(RuntimeState.LoadingOverlay, {BackgroundTransparency = 1}, 0.30)
        task.wait(0.34)
        if RuntimeState.LoadingOverlay and RuntimeState.LoadingOverlay.Parent then
            RuntimeState.LoadingOverlay:Destroy()
        end
        if RuntimeState.active and Window and Window.Parent then
            Window.Position = UDim2.fromScale(0.5, 0.5)
            RuntimeState.menuRestorePosition = Window.Position
            RuntimeState.menuFadeTargets = captureMenuFadeTargets()
            RuntimeState.setMenuVisible(true)
        end
    end)
end
RuntimeState.playLoadingScreen()

-- Drag the performance strip from anywhere on its surface. Its position is
-- stored in the same optional local config when Auto Save is enabled.
RuntimeState.performanceDragging = false
RuntimeState.performanceDragStart = nil
RuntimeState.performanceStartPosition = nil
RuntimeState.performanceDragChangedConnection = nil
RuntimeState.performanceDragEndedConnection = nil
RuntimeState.beginPerformanceDrag = function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.performanceDragging = true
        RuntimeState.performanceDragStart = input.Position
        RuntimeState.performanceStartPosition = RuntimeState.PerformanceWidget.Position
    end
end

-- Connect the parent and each metric cell so the HUD remains draggable when
-- the pointer starts directly over a text label rather than empty panel space.
RuntimeState.PerformanceWidget.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
RuntimeState.PerformanceBadge.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
RuntimeState.PerformanceBrandText.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
for _, cell in pairs(RuntimeState.PerformanceCells) do
    cell.frame.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
    cell.icon.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
    cell.title.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
    cell.value.InputBegan:Connect(RuntimeState.beginPerformanceDrag)
end

RuntimeState.performanceDragChangedConnection = UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.performanceDragging then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        local delta = input.Position - RuntimeState.performanceDragStart
        local camera = workspace.CurrentCamera
        local viewportSize = camera and camera.ViewportSize or Vector2.new(1280, 720)
        local width = RuntimeState.PerformanceWidget.AbsoluteSize.X
        local height = RuntimeState.PerformanceWidget.AbsoluteSize.Y
        local anchorX = RuntimeState.performanceStartPosition.X.Scale * viewportSize.X + RuntimeState.performanceStartPosition.X.Offset
        local anchorY = RuntimeState.performanceStartPosition.Y.Scale * viewportSize.Y + RuntimeState.performanceStartPosition.Y.Offset
        local nextX = anchorX + delta.X
        local nextY = anchorY + delta.Y
        -- Keep the whole strip on-screen while still allowing free placement.
        local halfWidth = math.min(width * 0.5, viewportSize.X * 0.5)
        nextX = math.clamp(nextX, halfWidth, math.max(halfWidth, viewportSize.X - halfWidth))
        nextY = math.clamp(nextY, 0, math.max(0, viewportSize.Y - height))
        RuntimeState.PerformanceWidget.Position = UDim2.fromOffset(nextX, nextY)
    end
end)

RuntimeState.performanceDragEndedConnection = UserInputService.InputEnded:Connect(function(input)
    if RuntimeState.performanceDragging and (input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch) then
        RuntimeState.performanceDragging = false
        RuntimeState.performanceDragStart = nil
        RuntimeState.performanceStartPosition = nil
        if scheduleConfigSave then scheduleConfigSave() end
    end
end)

-- Each swatch opens its own detached, draggable color editor.
RuntimeState.HoverColorButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.HoverColorPicker, 1)
end)

RuntimeState.pickerDragMode = nil
RuntimeState.SVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.pickerDragMode = "sv"
        updateSVFromPosition(getColorEditorPointer(input))
    end
end)
RuntimeState.HueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.pickerDragMode = "hue"
        updateHueFromPosition(getColorEditorPointer(input))
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.pickerDragMode then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        local position = getColorEditorPointer(input)
        if RuntimeState.pickerDragMode == "sv" then
            updateSVFromPosition(position)
        elseif RuntimeState.pickerDragMode == "hue" then
            updateHueFromPosition(position)
        end
    elseif input.UserInputType == Enum.UserInputType.Touch then
        if RuntimeState.pickerDragMode == "sv" then
            updateSVFromPosition(getColorEditorPointer(input))
        elseif RuntimeState.pickerDragMode == "hue" then
            updateHueFromPosition(getColorEditorPointer(input))
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.pickerDragMode = nil
    end
end)

RuntimeState.HexBox.FocusLost:Connect(function()
    local color = parseHex(RuntimeState.HexBox.Text)
    if color then
        applyHoverColor(color)
    else
        RuntimeState.HexBox.Text = colorToHex(hoverColor)
    end
end)

RuntimeState.RGBBox.FocusLost:Connect(function()
    local color = parseRGB(RuntimeState.RGBBox.Text)
    if color then
        applyHoverColor(color)
    else
        RuntimeState.RGBBox.Text = colorToRGBText(hoverColor)
    end
end)

RuntimeState.trailPickerDragMode = nil
RuntimeState.TrailSVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.trailPickerDragMode = "sv"
        updateTrailSVFromPosition(getColorEditorPointer(input))
    end
end)
RuntimeState.TrailHueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.trailPickerDragMode = "hue"
        updateTrailHueFromPosition(getColorEditorPointer(input))
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.trailPickerDragMode then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        local position = getColorEditorPointer(input)
        if RuntimeState.trailPickerDragMode == "sv" then updateTrailSVFromPosition(position) else updateTrailHueFromPosition(position) end
    elseif input.UserInputType == Enum.UserInputType.Touch then
        if RuntimeState.trailPickerDragMode == "sv" then updateTrailSVFromPosition(getColorEditorPointer(input)) else updateTrailHueFromPosition(getColorEditorPointer(input)) end
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.trailPickerDragMode = nil
    end
end)
RuntimeState.formatTrailSliderValue = function(control, value)
    return string.format("%." .. tostring(control.decimals) .. "f%s", value, control.suffix)
end

RuntimeState.refreshTrailSizeSlider = function(control)
    local alpha = math.clamp((control.value - control.minimum) / (control.maximum - control.minimum), 0, 1)
    control.fill.Size = UDim2.new(alpha, 0, 1, 0)
    control.knob.Position = UDim2.new(alpha, 0, 0.5, 0)
    control.valueLabel.Text = RuntimeState.formatTrailSliderValue(control, control.value)
end

RuntimeState.applyTrailSizeValue = function(control, screenX)
    local trackPosition = control.track.AbsolutePosition.X
    local trackWidth = control.track.AbsoluteSize.X
    if trackWidth <= 0 then return end

    local alpha = math.clamp((screenX - trackPosition) / trackWidth, 0, 1)
    local rawValue = control.minimum + (control.maximum - control.minimum) * alpha
    local stepped = control.minimum + math.floor(((rawValue - control.minimum) / control.step) + 0.5) * control.step
    control.value = math.clamp(stepped, control.minimum, control.maximum)

    if control.key == "startWidth" then
        trailStartWidth = control.value
    elseif control.key == "endWidth" then
        trailEndWidth = control.value
    elseif control.key == "lifetime" then
        trailLifetime = control.value
    elseif control.key == "minLength" then
        trailMinLength = control.value
    elseif control.key == "spread" then
        trailSpread = control.value
    end

    RuntimeState.refreshTrailSizeSlider(control)
    applyTrailStyle()
    if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
    if scheduleConfigSave then scheduleConfigSave() end
end

RuntimeState.activeTrailSizeSlider = nil
for _, control in ipairs(RuntimeState.TrailSizeControls) do
    RuntimeState.refreshTrailSizeSlider(control)
    control.input.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            RuntimeState.activeTrailSizeSlider = control
            RuntimeState.applyTrailSizeValue(control, input.Position.X)
        end
    end)
end

trailSizeInputChangedConnection = UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.activeTrailSizeSlider then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        RuntimeState.applyTrailSizeValue(RuntimeState.activeTrailSizeSlider, UserInputService:GetMouseLocation().X)
    elseif input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.applyTrailSizeValue(RuntimeState.activeTrailSizeSlider, input.Position.X)
    end
end)

trailSizeInputEndedConnection = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.activeTrailSizeSlider = nil
    end
end)

RuntimeState.TrailHexBox.FocusLost:Connect(function()
    local parsed = parseHex(RuntimeState.TrailHexBox.Text)
    if parsed then applyTrailColorValue(parsed) else RuntimeState.TrailHexBox.Text = trailColorToHex(trailColor) end
end)
RuntimeState.TrailRGBBox.FocusLost:Connect(function()
    local parsed = parseRGB(RuntimeState.TrailRGBBox.Text)
    if parsed then applyTrailColorValue(parsed) else RuntimeState.TrailRGBBox.Text = trailColorToRGBText(trailColor) end
end)

RuntimeState.jumpCirclePickerDragMode = nil
RuntimeState.JumpCircleSVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.jumpCirclePickerDragMode = "sv"
        updateJumpCircleSVFromPosition(getColorEditorPointer(input))
    end
end)
RuntimeState.JumpCircleHueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.jumpCirclePickerDragMode = "hue"
        updateJumpCircleHueFromPosition(getColorEditorPointer(input))
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if not RuntimeState.jumpCirclePickerDragMode then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        local position = getColorEditorPointer(input)
        if RuntimeState.jumpCirclePickerDragMode == "sv" then
            updateJumpCircleSVFromPosition(position)
        else
            updateJumpCircleHueFromPosition(position)
        end
    elseif input.UserInputType == Enum.UserInputType.Touch then
        if RuntimeState.jumpCirclePickerDragMode == "sv" then
            updateJumpCircleSVFromPosition(getColorEditorPointer(input))
        else
            updateJumpCircleHueFromPosition(getColorEditorPointer(input))
        end
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.jumpCirclePickerDragMode = nil
    end
end)
RuntimeState.JumpCircleHexBox.FocusLost:Connect(function()
    local parsed = parseHex(RuntimeState.JumpCircleHexBox.Text)
    if parsed then applyJumpCircleColorValue(parsed) else RuntimeState.JumpCircleHexBox.Text = colorToHex(jumpCircleColor) end
end)
RuntimeState.JumpCircleRGBBox.FocusLost:Connect(function()
    local parsed = parseRGB(RuntimeState.JumpCircleRGBBox.Text)
    if parsed then applyJumpCircleColorValue(parsed) else RuntimeState.JumpCircleRGBBox.Text = colorToRGBText(jumpCircleColor) end
end)
RuntimeState.CustomAuraSVInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CustomAura.pickerDragMode = "sv"
        RuntimeState.updateCustomAuraSVFromPosition(getColorEditorPointer(input))
    end
end)
RuntimeState.CustomAuraHueInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CustomAura.pickerDragMode = "hue"
        RuntimeState.updateCustomAuraHueFromPosition(getColorEditorPointer(input))
    end
end)
RuntimeState.CustomAura.pickerInputChanged = UserInputService.InputChanged:Connect(function(input)
    local mode = RuntimeState.CustomAura.pickerDragMode
    if not mode then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        local position = getColorEditorPointer(input)
        if mode == "sv" then RuntimeState.updateCustomAuraSVFromPosition(position) else RuntimeState.updateCustomAuraHueFromPosition(position) end
    elseif input.UserInputType == Enum.UserInputType.Touch then
        if mode == "sv" then RuntimeState.updateCustomAuraSVFromPosition(getColorEditorPointer(input)) else RuntimeState.updateCustomAuraHueFromPosition(getColorEditorPointer(input)) end
    end
end)
RuntimeState.CustomAura.pickerInputEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CustomAura.pickerDragMode = nil
    end
end)
RuntimeState.CustomAuraHexBox.FocusLost:Connect(function()
    local parsed = parseHex(RuntimeState.CustomAuraHexBox.Text)
    if parsed then RuntimeState.applyCustomAuraColor(parsed)
    else RuntimeState.CustomAuraHexBox.Text = string.format("#%02X%02X%02X", math.floor(RuntimeState.CustomAura.color.R * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.G * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.B * 255 + 0.5)) end
end)
RuntimeState.CustomAuraRGBBox.FocusLost:Connect(function()
    local parsed = parseRGB(RuntimeState.CustomAuraRGBBox.Text)
    if parsed then RuntimeState.applyCustomAuraColor(parsed)
    else RuntimeState.CustomAuraRGBBox.Text = string.format("%d, %d, %d", math.floor(RuntimeState.CustomAura.color.R * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.G * 255 + 0.5), math.floor(RuntimeState.CustomAura.color.B * 255 + 0.5)) end
end)
RuntimeState.CustomAuraColorButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.CustomAuraColorPicker, 6)
end)
RuntimeState.CustomAuraStyleButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.CustomAuraStyleEditor, 7)
end)
RuntimeState.CustomAuraSizeButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.CustomAuraSizeEditor, 13)
end)
RuntimeState.CustomAuraSizeInput.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CustomAura.sizeDragging = true
        RuntimeState.CustomAura.offsetDraggingAxis = nil
        RuntimeState.updateCustomAuraSizeFromPointer(input.Position, true)
    end
end)
for axis, offsetInput in pairs(RuntimeState.CustomAuraOffsetInputs) do
    local selectedAxis = axis
    offsetInput.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            RuntimeState.CustomAura.sizeDragging = false
            RuntimeState.CustomAura.offsetDraggingAxis = selectedAxis
            RuntimeState.updateCustomAuraOffsetFromPointer(selectedAxis, input.Position, true)
        end
    end)
end
RuntimeState.CustomAura.sizeInputChanged = UserInputService.InputChanged:Connect(function(input)
    local aura = RuntimeState.CustomAura
    if not aura.sizeDragging and not aura.offsetDraggingAxis then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        local pointer = getColorEditorPointer(input)
        if aura.sizeDragging then
            RuntimeState.updateCustomAuraSizeFromPointer(pointer, true)
        elseif aura.offsetDraggingAxis then
            RuntimeState.updateCustomAuraOffsetFromPointer(aura.offsetDraggingAxis, pointer, true)
        end
    end
end)
RuntimeState.CustomAura.sizeInputEnded = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        RuntimeState.CustomAura.sizeDragging = false
        RuntimeState.CustomAura.offsetDraggingAxis = nil
    end
end)
RuntimeState.setCustomAuraSize(RuntimeState.CustomAura.size, false)
RuntimeState.setCustomAuraOffset("X", RuntimeState.CustomAura.offsetX, false)
RuntimeState.setCustomAuraOffset("Y", RuntimeState.CustomAura.offsetY, false)
RuntimeState.setCustomAuraOffset("Z", RuntimeState.CustomAura.offsetZ, false)
RuntimeState.CustomAuraToggleButton.MouseButton1Click:Connect(function()
    RuntimeState.setCustomAura(not RuntimeState.CustomAura.enabled)
    RuntimeState.refreshCustomAuraUI()
    if scheduleConfigSave then scheduleConfigSave() end
end)

RuntimeState.JumpCircleColorButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.JumpCircleColorPicker, 3)
end)
RuntimeState.JumpCircleSizeButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.jumpCircleSizeEditor, 5)
end)
RuntimeState.JumpCircleStyleButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.JumpCircleStyleEditor, 14)
end)
for styleName, styleButton in pairs(RuntimeState.JumpCircleStyleButtons or {}) do
    styleButton.MouseButton1Click:Connect((function(selectedStyle)
        return function()
            jumpCircleStyle = selectedStyle
            updateJumpCircleStyleButton()
            updateJumpCircleToggle()
            RuntimeState.JumpCircleStyleEditor.Visible = false
            if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
            if scheduleConfigSave then scheduleConfigSave() end
        end
    end)(styleName))
end
RuntimeState.JumpCircleToggleButton.MouseButton1Click:Connect(function()
    jumpCircleEnabled = not jumpCircleEnabled
    updateJumpCircleToggle()
    if not jumpCircleEnabled then clearJumpCircleEffects() end
    if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
    if scheduleConfigSave then scheduleConfigSave() end
end)
RuntimeState.TrailColorButton.MouseButton1Click:Connect(function()
    RuntimeState.openFloatingColorEditor(RuntimeState.TrailColorPicker, 2)
end)
-- Open Character Trail SIZE as a side panel next to the main menu.
-- Prefer the right side; use the left side on wide layouts when necessary.
-- On narrow screens, keep it docked to the right edge so it remains separate
-- from the Visual card and can still be moved by dragging its title.
RuntimeState.openTrailSizeEditor = function()
    if RuntimeState.TrailSizeCard.Visible then
        RuntimeState.TrailSizeCard.Visible = false
        return
    end

    local camera = workspace.CurrentCamera
    local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
    local editorWidth = RuntimeState.TrailSizeCard.AbsoluteSize.X
    local editorHeight = RuntimeState.TrailSizeCard.AbsoluteSize.Y
    if editorWidth <= 0 then editorWidth = 320 end
    if editorHeight <= 0 then editorHeight = 222 end

    local windowLeft = Window.AbsolutePosition.X
    local windowRight = windowLeft + Window.AbsoluteSize.X
    local gap = 12
    local rightX = windowRight + gap
    local leftX = windowLeft - editorWidth - gap
    local x

    if rightX + editorWidth <= view.X - gap then
        x = rightX
    elseif leftX >= gap then
        x = leftX
    else
        -- There is not enough room for a completely detached side-by-side
        -- layout; pin the editor to the screen's right side instead.
        x = math.clamp(view.X - editorWidth - gap, gap, math.max(gap, view.X - editorWidth - gap))
    end

    local rowCenterY = RuntimeState.TrailRow.AbsolutePosition.Y + RuntimeState.TrailRow.AbsoluteSize.Y * 0.5
    local y = math.clamp(
        rowCenterY - editorHeight * 0.5,
        gap,
        math.max(gap, view.Y - editorHeight - gap)
    )

    RuntimeState.TrailSizeCard.Position = UDim2.fromOffset(math.floor(x), math.floor(y))
    RuntimeState.TrailSizeCard.Visible = true
end

RuntimeState.trailSizeButton.MouseButton1Click:Connect(function()
    RuntimeState.openTrailSizeEditor()
end)
RuntimeState.TrailToggleButton.MouseButton1Click:Connect(function()
    trailEnabled = not trailEnabled
    updateTrailToggle()
    if trailEnabled then
        attachCharacterTrail(LocalPlayer.Character)
    else
        removeCharacterTrail()
    end
    if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
    if scheduleConfigSave then scheduleConfigSave() end
end)
trailCharacterAddedConnection = LocalPlayer.CharacterAdded:Connect(function(character)
    if trailEnabled then
        task.spawn(function()
            attachCharacterTrail(character)
        end)
    end
    task.spawn(bindJumpCircleCharacter, character)
end)

refreshColorPicker()
applyTrailColorValue(trailColor)
applyJumpCircleColorValue(jumpCircleColor)
updateJumpCircleStyleButton()
updateJumpCircleToggle()
updateTrailToggle()
RuntimeState.applyCustomAuraColor(RuntimeState.CustomAura.color)
RuntimeState.refreshCustomAuraUI()
if RuntimeState.CustomAura.enabled then RuntimeState.setCustomAura(true) end
if RuntimeState.Preview and RuntimeState.Preview.scheduleRefresh then RuntimeState.Preview.scheduleRefresh() end
if LocalPlayer.Character then
    task.spawn(bindJumpCircleCharacter, LocalPlayer.Character)
end
if trailEnabled and LocalPlayer.Character then
    task.spawn(function() attachCharacterTrail(LocalPlayer.Character) end)
end

-- Player ESP / Hover Highlight toggle removed. The row is hidden and the effect stays disabled.
updateHoverToggle()
if autoSaveEnabled and scheduleConfigSave then
    scheduleConfigSave()
end

RuntimeState.hoverRenderConnection = nil
RuntimeState.menuInputConnection = nil
RuntimeState.stop = function()
    RuntimeState.active = false
    pcall(function() if RuntimeState.ESPColorInputChanged then RuntimeState.ESPColorInputChanged:Disconnect(); RuntimeState.ESPColorInputChanged = nil end end)
    pcall(function() if RuntimeState.ESPColorInputEnded then RuntimeState.ESPColorInputEnded:Disconnect(); RuntimeState.ESPColorInputEnded = nil end end)
    pcall(function() if RuntimeState.ESPCornerBoxInputChanged then RuntimeState.ESPCornerBoxInputChanged:Disconnect(); RuntimeState.ESPCornerBoxInputChanged = nil end end)
    pcall(function() if RuntimeState.ESPCornerBoxInputEnded then RuntimeState.ESPCornerBoxInputEnded:Disconnect(); RuntimeState.ESPCornerBoxInputEnded = nil end end)
    pcall(function() RuntimeState.setCustomAura(false) end)
    for _, runningTweens in pairs(pageTransitionTweens or {}) do
        for _, tween in ipairs(runningTweens) do pcall(function() tween:Cancel() end) end
    end
    pcall(function() RuntimeState.setFreecam(false) end)
    pcall(function() RuntimeState.setFly(false) end)
    pcall(function() RuntimeState.setVehicleFly(false) end)
    pcall(function() RuntimeState.setFloat(false) end)
    pcall(function() RuntimeState.setNoclip(false) end)
    pcall(function() if RuntimeState.ESPConfig then
    RuntimeState.ESPConfig.enabled = false
    if RuntimeState.espConnection then RuntimeState.espConnection:Disconnect(); RuntimeState.espConnection = nil end
    for player in pairs(RuntimeState.ESPObjects or {}) do clearESPForPlayer(player) end
end
RuntimeState.setAimAssist(false) end)
    for humanoid, snapshot in pairs(RuntimeState.PlayerTools.originals or {}) do
        if humanoid and humanoid.Parent then
            pcall(function()
                if snapshot.useJumpPower ~= nil then humanoid.UseJumpPower = snapshot.useJumpPower end
                if snapshot.walkSpeed ~= nil then humanoid.WalkSpeed = snapshot.walkSpeed end
                if snapshot.jumpPower ~= nil then humanoid.JumpPower = snapshot.jumpPower end
                if snapshot.jumpHeight ~= nil then humanoid.JumpHeight = snapshot.jumpHeight end
            end)
        end
    end
    if RuntimeState.PlayerTools.originalFov ~= nil and workspace.CurrentCamera then
        pcall(function() workspace.CurrentCamera.FieldOfView = RuntimeState.PlayerTools.originalFov end)
    end
    if RuntimeState.playerCharacterAddedConnection then pcall(function() RuntimeState.playerCharacterAddedConnection:Disconnect() end); RuntimeState.playerCharacterAddedConnection=nil end
    for _, control in ipairs(RuntimeState.playerSliderControls or {}) do
        if control.changed then pcall(function() control.changed:Disconnect() end) end
        if control.ended then pcall(function() control.ended:Disconnect() end) end
    end
    pcall(function() RuntimeState.setAntiAfk(false) end)
    pcall(function() RuntimeState.setNoTexture(false) end)
    pcall(function() RuntimeState.setNoGui(false) end)
    pcall(function() RuntimeState.setAutoSkipCrates(false) end)
    pcall(function() RuntimeState.setAutoSessionReward(false) end)
    pcall(function() RuntimeState.applyWater(false) end)
    pcall(function() RuntimeState.applyNoFog(false) end)
    pcall(function() RuntimeState.setKeystrokes(false) end)
    for _,connection in pairs({RuntimeState.waterInputChanged,RuntimeState.waterInputEnded}) do if connection then pcall(function() connection:Disconnect() end) end end
    for _,connection in pairs(RuntimeState.Keystrokes and RuntimeState.Keystrokes.connections or {}) do if connection then pcall(function() connection:Disconnect() end) end end
    if RuntimeState.customCursorRenderConnection then
        pcall(function() RuntimeState.customCursorRenderConnection:Disconnect() end)
        RuntimeState.customCursorRenderConnection = nil
    end
    if RuntimeState.previousMouseIconEnabled ~= nil then
        local previousMouseIconEnabled = RuntimeState.previousMouseIconEnabled
        pcall(function() UserInputService.MouseIconEnabled = previousMouseIconEnabled end)
        RuntimeState.previousMouseIconEnabled = nil
    end
    if RuntimeState.menuInputConnection then
        pcall(function() RuntimeState.menuInputConnection:Disconnect() end)
        RuntimeState.menuInputConnection = nil
    end
    if RuntimeState.hoverRenderConnection then
        pcall(function() RuntimeState.hoverRenderConnection:Disconnect() end)
        RuntimeState.hoverRenderConnection = nil
    end
    if RuntimeState.freecamSpeedInputChangedConnection then
        pcall(function() RuntimeState.freecamSpeedInputChangedConnection:Disconnect() end)
        RuntimeState.freecamSpeedInputChangedConnection = nil
    end
    if RuntimeState.freecamSpeedInputEndedConnection then
        pcall(function() RuntimeState.freecamSpeedInputEndedConnection:Disconnect() end)
        RuntimeState.freecamSpeedInputEndedConnection = nil
    end
    RuntimeState.freecamSpeedDragging = false
    if RuntimeState.flySpeedInputChangedConnection then
        pcall(function() RuntimeState.flySpeedInputChangedConnection:Disconnect() end)
        RuntimeState.flySpeedInputChangedConnection = nil
    end
    if RuntimeState.flySpeedInputEndedConnection then
        pcall(function() RuntimeState.flySpeedInputEndedConnection:Disconnect() end)
        RuntimeState.flySpeedInputEndedConnection = nil
    end
    RuntimeState.flySpeedDragging = false
    if RuntimeState.performanceRenderConnection then
        pcall(function() RuntimeState.performanceRenderConnection:Disconnect() end)
        RuntimeState.performanceRenderConnection = nil
    end
    if RuntimeState.performanceDragChangedConnection then
        pcall(function() RuntimeState.performanceDragChangedConnection:Disconnect() end)
        RuntimeState.performanceDragChangedConnection = nil
    end
    if RuntimeState.performanceDragEndedConnection then
        pcall(function() RuntimeState.performanceDragEndedConnection:Disconnect() end)
        RuntimeState.performanceDragEndedConnection = nil
    end
    for _, connectionName in ipairs({
        "keystrokesDragChangedConnection", "keystrokesDragEndedConnection",
        "mainWindowDragChangedConnection", "mainWindowDragEndedConnection",
        "infoCardDragChangedConnection", "infoCardDragEndedConnection",
        "universalWindowDragChangedConnection", "universalWindowDragEndedConnection",
        "universalFloatingChildAdded",
    }) do
        local connection = RuntimeState[connectionName]
        if connection then
            pcall(function() connection:Disconnect() end)
            RuntimeState[connectionName] = nil
        end
    end
    if RuntimeState.floatingEditorDragChanged then
        pcall(function() RuntimeState.floatingEditorDragChanged:Disconnect() end)
        RuntimeState.floatingEditorDragChanged = nil
    end
    if RuntimeState.floatingEditorDragEnded then
        pcall(function() RuntimeState.floatingEditorDragEnded:Disconnect() end)
        RuntimeState.floatingEditorDragEnded = nil
    end
    if RuntimeState.ThemeRegistryConnection then
        pcall(function() RuntimeState.ThemeRegistryConnection:Disconnect() end)
        RuntimeState.ThemeRegistryConnection = nil
    end
    if RuntimeState.FontRegistryConnection then
        pcall(function() RuntimeState.FontRegistryConnection:Disconnect() end)
        RuntimeState.FontRegistryConnection = nil
    end
    for _, propertyMap in pairs(RuntimeState.ThemeRegistrySeen or {}) do
        for _, entry in pairs(propertyMap) do
            if entry.changedConnection then
                pcall(function() entry.changedConnection:Disconnect() end)
                entry.changedConnection = nil
            end
        end
    end
    if RuntimeState.weatherRenderConnection then
        pcall(function() RuntimeState.weatherRenderConnection:Disconnect() end)
        RuntimeState.weatherRenderConnection = nil
    end
    if RuntimeState.Weather then
        for _, connection in pairs(RuntimeState.Weather.settingsInputChanged or {}) do
            pcall(function() connection:Disconnect() end)
        end
        for _, connection in pairs(RuntimeState.Weather.settingsInputEnded or {}) do
            pcall(function() connection:Disconnect() end)
        end
    end
    if RuntimeState.Weather.folder then pcall(function() RuntimeState.Weather.folder:Destroy() end) end
    if RuntimeState.Skybox and RuntimeState.Skybox.restore then pcall(RuntimeState.Skybox.restore) end
    if RuntimeState.cursorColorInputChanged then
        pcall(function() RuntimeState.cursorColorInputChanged:Disconnect() end)
        RuntimeState.cursorColorInputChanged = nil
    end
    if RuntimeState.cursorColorInputEnded then
        pcall(function() RuntimeState.cursorColorInputEnded:Disconnect() end)
        RuntimeState.cursorColorInputEnded = nil
    end
    if RuntimeState.themePickerInputChanged then
        pcall(function() RuntimeState.themePickerInputChanged:Disconnect() end)
        RuntimeState.themePickerInputChanged = nil
    end
    if RuntimeState.themePickerInputEnded then
        pcall(function() RuntimeState.themePickerInputEnded:Disconnect() end)
        RuntimeState.themePickerInputEnded = nil
    end
    if RuntimeState.CustomAura and RuntimeState.CustomAura.pickerInputChanged then
        pcall(function() RuntimeState.CustomAura.pickerInputChanged:Disconnect() end)
        RuntimeState.CustomAura.pickerInputChanged = nil
    end
    if RuntimeState.CustomAura and RuntimeState.CustomAura.pickerInputEnded then
        pcall(function() RuntimeState.CustomAura.pickerInputEnded:Disconnect() end)
        RuntimeState.CustomAura.pickerInputEnded = nil
    end
    if RuntimeState.CustomAura then
        RuntimeState.CustomAura.sizeDragging = false
        RuntimeState.CustomAura.offsetDraggingAxis = nil
    end
    if RuntimeState.CustomAura and RuntimeState.CustomAura.sizeInputChanged then
        pcall(function() RuntimeState.CustomAura.sizeInputChanged:Disconnect() end)
        RuntimeState.CustomAura.sizeInputChanged = nil
    end
    if RuntimeState.CustomAura and RuntimeState.CustomAura.sizeInputEnded then
        pcall(function() RuntimeState.CustomAura.sizeInputEnded:Disconnect() end)
        RuntimeState.CustomAura.sizeInputEnded = nil
    end
    if RuntimeState.jumpCircleSizeInputChanged then
        pcall(function() RuntimeState.jumpCircleSizeInputChanged:Disconnect() end)
        RuntimeState.jumpCircleSizeInputChanged = nil
    end
    if RuntimeState.jumpCircleSizeInputEnded then
        pcall(function() RuntimeState.jumpCircleSizeInputEnded:Disconnect() end)
        RuntimeState.jumpCircleSizeInputEnded = nil
    end
    if trailSizeInputChangedConnection then
        pcall(function() trailSizeInputChangedConnection:Disconnect() end)
        trailSizeInputChangedConnection = nil
    end
    if trailSizeInputEndedConnection then
        pcall(function() trailSizeInputEndedConnection:Disconnect() end)
        trailSizeInputEndedConnection = nil
    end
    pcall(clearHoverEffect)
    pcall(removeCharacterTrail)
    pcall(clearJumpCircleEffects)
    if jumpCircleConnection then
        pcall(function() jumpCircleConnection:Disconnect() end)
        jumpCircleConnection = nil
    end
    if trailCharacterAddedConnection then
        pcall(function() trailCharacterAddedConnection:Disconnect() end)
        trailCharacterAddedConnection = nil
    end
    if RuntimeState.previewCharacterConnection then
        pcall(function() RuntimeState.previewCharacterConnection:Disconnect() end)
        RuntimeState.previewCharacterConnection = nil
    end
    if RuntimeState.Preview then
        for _, connection in pairs({RuntimeState.Preview.viewportInputConnection, RuntimeState.Preview.inputChangedConnection, RuntimeState.Preview.inputEndedConnection}) do
            if connection then pcall(function() connection:Disconnect() end) end
        end
        if RuntimeState.Preview.model then pcall(function() RuntimeState.Preview.model:Destroy() end) end
        if RuntimeState.Preview.World then pcall(function() RuntimeState.Preview.World:Destroy() end) end
    end
    pcall(function()
        if ScreenGui and ScreenGui.Parent then ScreenGui:Destroy() end
    end)
    for _, object in ipairs(workspace:GetDescendants()) do
        if object.Name == "CATScript_HoverRingGlow"
            or object.Name == "CATScript_HoverRing"
            or object.Name == "CATScript_HoverHighlight" then
            pcall(function() object:Destroy() end)
        end
    end
end

RuntimeState.hoverRenderConnection = nil -- Player ESP render loop removed.

-- Drag the main dashboard from its top bar or breadcrumb. Mouse and touch
-- are supported, interactive search/window buttons are excluded, and the
-- dashboard is kept on-screen after every move.
do
    RuntimeState.mainWindowDragging = false
    RuntimeState.mainWindowDragStart = nil
    RuntimeState.mainWindowStartPosition = nil

    local function isInside(guiObject, point)
        if not guiObject or not guiObject.Visible then return false end
        local position = guiObject.AbsolutePosition
        local size = guiObject.AbsoluteSize
        return point.X >= position.X and point.X <= position.X + size.X
            and point.Y >= position.Y and point.Y <= position.Y + size.Y
    end

    local function beginMainWindowDrag(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
            and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local point = input.Position
        if isInside(RuntimeState.Search, point)
            or isInside(RuntimeState.MinimizeButton, point)
            or isInside(RuntimeState.CloseButton, point) then return end

        RuntimeState.mainWindowDragging = true
        RuntimeState.mainWindowDragStart = point
        RuntimeState.mainWindowStartPosition = Window.Position
    end

    RuntimeState.Topbar.Active = true
    RuntimeState.Breadcrumb.Active = true
    RuntimeState.Topbar.InputBegan:Connect(beginMainWindowDrag)
    RuntimeState.Breadcrumb.InputBegan:Connect(beginMainWindowDrag)

    -- A transparent TextButton is placed above the topbar labels. Roblox GUI
    -- input does not bubble from child labels to the parent Frame, so the old
    -- Frame.InputBegan-only approach missed most of the title area.
    local topbarDragHandle = make("TextButton", {
        Name = "CATScriptMainTopbarDragHandle",
        Position = UDim2.fromOffset(0, 0),
        Size = UDim2.new(1, -330, 1, 0),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Text = "",
        TextTransparency = 1,
        AutoButtonColor = false,
        Active = true,
        Selectable = false,
        ZIndex = 5000,
    }, RuntimeState.Topbar)
    topbarDragHandle.InputBegan:Connect(beginMainWindowDrag)

    -- The logo/brand block in the left sidebar is also a safe drag surface.
    local brandDragHandle = make("TextButton", {
        Name = "CATScriptBrandDragHandle",
        Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromScale(1, 1),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Text = "",
        TextTransparency = 1,
        AutoButtonColor = false,
        Active = true,
        Selectable = false,
        ZIndex = 5000,
    }, Brand)
    brandDragHandle.InputBegan:Connect(beginMainWindowDrag)

    RuntimeState.mainWindowDragChangedConnection = UserInputService.InputChanged:Connect(function(input)
        if not RuntimeState.mainWindowDragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
            and input.UserInputType ~= Enum.UserInputType.Touch then return end

        local pointer = input.Position
        local delta = pointer - RuntimeState.mainWindowDragStart
        local camera = workspace.CurrentCamera
        local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
        local width = math.min(Window.AbsoluteSize.X, view.X)
        local height = math.min(Window.AbsoluteSize.Y, view.Y)
        local start = RuntimeState.mainWindowStartPosition
        local centerX = start.X.Scale * view.X + start.X.Offset + delta.X
        local centerY = start.Y.Scale * view.Y + start.Y.Offset + delta.Y
        local halfWidth = width * 0.5
        local halfHeight = height * 0.5
        centerX = math.clamp(centerX, halfWidth, math.max(halfWidth, view.X - halfWidth))
        centerY = math.clamp(centerY, halfHeight, math.max(halfHeight, view.Y - halfHeight))
        Window.Position = UDim2.fromOffset(centerX, centerY)
        RuntimeState.menuRestorePosition = Window.Position
    end)

    RuntimeState.mainWindowDragEndedConnection = UserInputService.InputEnded:Connect(function(input)
        if RuntimeState.mainWindowDragging and (input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch) then
            RuntimeState.mainWindowDragging = false
            RuntimeState.mainWindowDragStart = nil
            RuntimeState.mainWindowStartPosition = nil
            RuntimeState.menuRestorePosition = Window.Position
            if scheduleConfigSave then scheduleConfigSave() end
        end
    end)
end

-- The profile popup is a separate window too: drag it by its heading/name.
do
    RuntimeState.infoCardDragging = false
    RuntimeState.infoCardDragStart = nil
    RuntimeState.infoCardStartPosition = nil

    local dragHandles = {InfoWelcome, InfoDisplayName, InfoUsername}
    local function beginInfoCardDrag(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            RuntimeState.infoCardDragging = true
            RuntimeState.infoCardDragStart = input.Position
            RuntimeState.infoCardStartPosition = InfoCard.Position
        end
    end

    InfoCard.Active = true
    local function beginInfoCardHeaderDrag(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
            and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local localX = input.Position.X - InfoCard.AbsolutePosition.X
        local localY = input.Position.Y - InfoCard.AbsolutePosition.Y
        -- Keep the close button and avatar image out of the header drag zone.
        if localY < 0 or localY > 74 or localX < 128 or localX >= InfoCard.AbsoluteSize.X - 28 then return end
        beginInfoCardDrag(input)
    end
    InfoCard.InputBegan:Connect(beginInfoCardHeaderDrag)
    for _, handle in ipairs(dragHandles) do
        handle.Active = true
        handle.InputBegan:Connect(beginInfoCardDrag)
    end

    RuntimeState.infoCardDragChangedConnection = UserInputService.InputChanged:Connect(function(input)
        if not RuntimeState.infoCardDragging or not InfoOverlay.Visible then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
            and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local pointer = input.UserInputType == Enum.UserInputType.Touch
            and input.Position or UserInputService:GetMouseLocation()
        local delta = pointer - RuntimeState.infoCardDragStart
        local parentSize = InfoOverlay.AbsoluteSize
        local width = math.min(InfoCard.AbsoluteSize.X, parentSize.X)
        local height = math.min(InfoCard.AbsoluteSize.Y, parentSize.Y)
        local start = RuntimeState.infoCardStartPosition
        local centerX = start.X.Scale * parentSize.X + start.X.Offset + delta.X
        local centerY = start.Y.Scale * parentSize.Y + start.Y.Offset + delta.Y
        local halfWidth = width * 0.5
        local halfHeight = height * 0.5
        centerX = math.clamp(centerX, halfWidth, math.max(halfWidth, parentSize.X - halfWidth))
        centerY = math.clamp(centerY, halfHeight, math.max(halfHeight, parentSize.Y - halfHeight))
        InfoCard.Position = UDim2.fromOffset(centerX, centerY)
    end)

    RuntimeState.infoCardDragEndedConnection = UserInputService.InputEnded:Connect(function(input)
        if RuntimeState.infoCardDragging and (input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch) then
            RuntimeState.infoCardDragging = false
            RuntimeState.infoCardDragStart = nil
            RuntimeState.infoCardStartPosition = nil
        end
    end)
end

-- UNIVERSAL WINDOW DRAG LAYER -------------------------------------------------
-- Every detached editor receives a real, transparent TextButton drag surface.
-- This deliberately avoids relying on InputBegan bubbling from child labels,
-- which Roblox does not provide. The header remains draggable even when the
-- title text is under the pointer; controls below the header remain clickable.
do
    RuntimeState.universalWindowDrag = nil
    RuntimeState.universalDragWindows = setmetatable({}, {__mode = "k"})

    local function getWindowBounds(window)
        local parent = window.Parent
        if parent and parent:IsA("GuiObject") then
            return parent.AbsolutePosition, parent.AbsoluteSize
        end
        local camera = workspace.CurrentCamera
        local view = camera and camera.ViewportSize or Vector2.new(1280, 720)
        return Vector2.new(0, 0), view
    end

    local function beginUniversalWindowDrag(window, input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
            and input.UserInputType ~= Enum.UserInputType.Touch then return end
        if not window or not window.Parent or not window.Visible then return end

        local boundsOrigin, boundsSize = getWindowBounds(window)
        RuntimeState.universalWindowDrag = {
            window = window,
            startPointer = input.Position,
            startAbsolute = window.AbsolutePosition,
            startPosition = window.Position,
            boundsOrigin = boundsOrigin,
            boundsSize = boundsSize,
        }
    end

    local function attachUniversalDrag(window, options)
        if not window or not window:IsA("GuiObject") or RuntimeState.universalDragWindows[window] then
            return
        end
        options = options or {}
        RuntimeState.universalDragWindows[window] = true

        local maxZ = window.ZIndex
        for _, descendant in ipairs(window:GetDescendants()) do
            if descendant:IsA("GuiObject") then
                maxZ = math.max(maxZ, descendant.ZIndex)
            end
        end

        local left = options.left or 0
        local top = options.top or 0
        local height = options.height or 26
        local rightGap = options.right or 0
        local handle = make("TextButton", {
            Name = "CATScriptUniversalDragHandle",
            Position = UDim2.new(0, left, 0, top),
            Size = UDim2.new(1, -(left + rightGap), 0, height),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Text = "",
            TextTransparency = 1,
            AutoButtonColor = false,
            Active = true,
            Selectable = false,
            ZIndex = maxZ + 25,
        }, window)
        handle.InputBegan:Connect(function(input)
            beginUniversalWindowDrag(window, input)
        end)
    end

    -- All top-level detached windows: color pickers, size/style editors,
    -- custom aura, jump circle, weather, skybox, theme, cursor, font, and the
    -- per-window appearance customizer.
    for _, child in ipairs(RuntimeState.floatingEditorsLayer:GetChildren()) do
        if child:IsA("GuiObject") then
            local rightGap = 0
            if child:FindFirstChild("FloatingAppearanceButton") or child:FindFirstChild("CloseFloatingEditor") then
                rightGap = 64 -- leave gear and close buttons outside the drag hitbox
            elseif child:FindFirstChild("CloseAppearancePanel") then
                rightGap = 34
            end
            attachUniversalDrag(child, {left = 0, right = rightGap, top = 0, height = 25})
        end
    end

    RuntimeState.universalFloatingChildAdded = RuntimeState.floatingEditorsLayer.ChildAdded:Connect(function(child)
        if child:IsA("GuiObject") then
            task.defer(function()
                if not RuntimeState.active or not child.Parent then return end
                local rightGap = 0
                if child:FindFirstChild("FloatingAppearanceButton") or child:FindFirstChild("CloseFloatingEditor") then
                    rightGap = 64
                elseif child:FindFirstChild("CloseAppearancePanel") then
                    rightGap = 34
                end
                attachUniversalDrag(child, {left = 0, right = rightGap, top = 0, height = 25})
            end)
        end
    end)

    -- Profile card: drag from the top text strip, excluding the close button.
    attachUniversalDrag(InfoCard, {left = 128, right = 30, top = 0, height = 27})

    -- HUDs have no interactive controls, so the entire surface is a handle.
    -- This works even when the pointer is over key cells or metric labels.
    if RuntimeState.PerformanceWidget then
        attachUniversalDrag(RuntimeState.PerformanceWidget, {left = 0, right = 0, top = 0, height = RuntimeState.PerformanceWidget.Size.Y.Offset})
    end
    if RuntimeState.KeystrokesWidget then
        attachUniversalDrag(RuntimeState.KeystrokesWidget, {left = 0, right = 0, top = 0, height = RuntimeState.KeystrokesWidget.Size.Y.Offset})
    end

    RuntimeState.universalWindowDragChangedConnection = UserInputService.InputChanged:Connect(function(input)
        local drag = RuntimeState.universalWindowDrag
        if not drag then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
            and input.UserInputType ~= Enum.UserInputType.Touch then return end

        local window = drag.window
        if not window or not window.Parent then
            RuntimeState.universalWindowDrag = nil
            return
        end

        local delta = input.Position - drag.startPointer
        local boundsOrigin, boundsSize = getWindowBounds(window)
        local startAbsolute = drag.startAbsolute
        local width, height = window.AbsoluteSize.X, window.AbsoluteSize.Y
        local minX, minY = boundsOrigin.X, boundsOrigin.Y
        local maxX = math.max(minX, boundsOrigin.X + boundsSize.X - width)
        local maxY = math.max(minY, boundsOrigin.Y + boundsSize.Y - height)
        local targetX = math.clamp(startAbsolute.X + delta.X, minX, maxX)
        local targetY = math.clamp(startAbsolute.Y + delta.Y, minY, maxY)
        local shiftX, shiftY = targetX - startAbsolute.X, targetY - startAbsolute.Y
        local startPosition = drag.startPosition
        window.Position = UDim2.new(
            startPosition.X.Scale, startPosition.X.Offset + shiftX,
            startPosition.Y.Scale, startPosition.Y.Offset + shiftY
        )
    end)

    RuntimeState.universalWindowDragEndedConnection = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            local drag = RuntimeState.universalWindowDrag
            RuntimeState.universalWindowDrag = nil
            if drag and drag.window and drag.window.Parent and scheduleConfigSave then
                scheduleConfigSave()
            end
        end
    end)
end

RuntimeState.MinimizeButton.MouseButton1Click:Connect(function()
    RuntimeState.Content.Visible = not RuntimeState.Content.Visible
    RuntimeState.Console.Visible = RuntimeState.Content.Visible
    RuntimeState.MinimizeButton.Text = RuntimeState.Content.Visible and "—" or "+"
end)

RuntimeState.CloseButton.MouseButton1Click:Connect(function()
    RuntimeState.setMenuVisible(false)
end)

RuntimeState.menuInputConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType ~= Enum.UserInputType.Keyboard or input.KeyCode == Enum.KeyCode.Unknown then
        return
    end

    -- Capture the next keyboard key even if Roblox marks it as game-processed.
    if waitingForKey then
        if input.KeyCode == Enum.KeyCode.Escape then
            waitingForKey = false
            RuntimeState.KeybindButton.Text = menuKeyCode.Name
            RuntimeState.KeybindStatus.Text = "KEYBIND CHANGE CANCELLED"
            RuntimeState.KeybindStatus.TextColor3 = COLORS.dim
        else
            setMenuKey(input.KeyCode)
            waitingForKey = false
        end
        return
    end

    -- Do not hide the menu while the user is typing into a setting field.
    if UserInputService:GetFocusedTextBox() then return end
    if input.KeyCode == menuKeyCode then
        RuntimeState.setMenuVisible(not Window.Visible)
    end
end)

print("[CATScript Visual] Loaded. Hover Highlight is enabled; " .. menuKeyCode.Name .. " toggles the menu.")
