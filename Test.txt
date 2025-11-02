 --[[
    ╔══════════════════════════════════════════════════════════════╗
    ║          🌌 VOID ENGINE - GALAXY EDITION 🌌                  ║
    ║        Execute this entire script in your Lua executor       ║
    ╚══════════════════════════════════════════════════════════════╝
    
    🌌 FEATURES:
    - F5: Interactive Menu - All features with keybinds
    - F8: 3D ESP - WorldToScreen player tracking
    - F6: Performance Monitor (Draggable)
    - F7: Key Press Visualizer (Draggable)
    - F9: Custom HUD (Draggable & Resizable)
    - F10: Hit Log - Combat tracking
    - F1-F4: Combat features & utilities
    - F11-F12: Heal Cover & Edit Mode
    
    🎨 DESIGN:
    - Deep purple & cosmic pink gradients
    - Neon cyan & magenta accents
    - Glowing effects on all elements
    - Drag & drop any UI element
    - Resize with mouse wheel
    
    ⚡ All features render on Susano overlay (invisible to OBS)
]]

-- ============================================================
-- CONFIGURATION
-- ============================================================
local CONFIG = {
    -- Keybinds
    MENU_KEY = 0x74,          -- F5
    PERF_KEY = 0x75,          -- F6
    KEYS_KEY = 0x76,          -- F7
    HUD_KEY = 0x78,           -- F9
    HITLOG_KEY = 0x79,        -- F10
    HEAL_COVER_KEY = 0x7A,    -- F11
    EDIT_MODE_KEY = 0x7B,     -- F12
    COMBAT_STANCE_KEY = 0x70, -- F1
    COMBAT_HUD_KEY = 0x13,    -- PAUSE (no default key)
    WATERMARK_KEY = 0x13,     -- PAUSE (no default key)
    FPS_BOOST_KEY = 0x73,     -- F4
    ESP_KEY = 0x77,           -- F8
    RAINBOW_KEY = 0x71,       -- F2
    
    -- Menu keys
    UP_KEY = 0x26,
    DOWN_KEY = 0x28,
    SELECT_KEY = 0x0D,
    
    -- Mouse keys for drag/drop
    LEFT_MOUSE = 0x01,
    RIGHT_MOUSE = 0x02,
    SCROLL_UP = 0x0A,
    SCROLL_DOWN = 0x0B,
    
    -- Screen resolution (adjust if needed)
    SCREEN_W = 1920,
    SCREEN_H = 1080,
}

-- ============================================================
-- GLOBAL STATE
-- ============================================================

-- Font system (loaded on startup)
local CustomFonts = {
    loaded = false,
    main = nil,
    title = nil,
}
local Features = {
    menu = {enabled = false, selectedIndex = 1, animProgress = 0, x = 100, y = 150, w = 550, h = 520, scale = 1, dragging = false, resizing = false, activeTab = 1},
    performance = {enabled = false, lastFrameTime = 0, lastUpdateTime = 0, currentFPS = 0, displayFPS = 0, x = 20, y = 20, w = 180, h = 80, scale = 1, dragging = false, resizing = false, toggleAnim = 0},
    keyvis = {enabled = false, keyStates = {}, x = 960 - 160, y = 900, w = 320, h = 160, scale = 1, dragging = false, resizing = false, toggleAnim = 0},
    hud = {enabled = false, x = 20, y = 960, w = 300, h = 100, scale = 1, dragging = false, resizing = false, toggleAnim = 0, lastHealth = 200, lastArmor = 0, healthSegments = {}, armorSegments = {}, hitEffect = 0, critEffect = {active = false, x = 0, y = 0, time = 0, particles = {}}},
    hitlog = {enabled = true, hits = {}, maxHits = 5, x = 20, y = 100, scale = 1, dragging = false, resizing = false, toggleAnim = 0},
    combatHud = {enabled = false, x = CONFIG.SCREEN_W - 220, y = 20, w = 200, h = 100, scale = 1, dragging = false, resizing = false, toggleAnim = 0, shotsFired = 0, shotsHit = 0, totalDamage = 0, damageWindow = {}, lastShotTime = 0, reactionTimes = {}},
    healCover = {enabled = false, toggleAnim = 0},
    noCombatStance = {enabled = false, toggleAnim = 0},
    editMode = {enabled = false, toggleAnim = 0},
    fpsBoost = {enabled = false, toggleAnim = 0},
    watermark = {enabled = true, toggleAnim = 1, x = CONFIG.SCREEN_W - 180, y = CONFIG.SCREEN_H - 35, dragging = false},
    esp = {enabled = false, toggleAnim = 0, maxDistance = 150, showHealth = true, showName = true},
    rainbowMode = {enabled = false, toggleAnim = 0, hue = 0},
    loading = {active = true, progress = 0, startTime = 0, holdTime = 0},
    rebinding = {active = false, targetFeature = nil, waitingForKey = false},
}

local InputStates = {
    lastMenuKey = false,
    lastPerfKey = false,
    lastKeysKey = false,
    lastHudKey = false,
    lastHealCoverKey = false,
    lastEditModeKey = false,
    lastHitLogKey = false,
    lastCombatStanceKey = false,
    lastCombatHudKey = false,
    lastWatermarkKey = false,
    lastFpsBoostKey = false,
    lastEspKey = false,
    lastRainbowKey = false,
    lastUpKey = false,
    lastDownKey = false,
    lastSelectKey = false,
    lastLeftMouse = false,
    mouseX = 0,
    mouseY = 0,
    dragStartX = 0,
    dragStartY = 0,
    dragOffsetX = 0,
    dragOffsetY = 0,
    currentDragging = nil,
}

-- ============================================================
-- GALAXY COLOR PALETTE
-- ============================================================
local GALAXY = {
    DEEP_PURPLE = {0.25, 0.15, 0.45},
    COSMIC_PINK = {0.95, 0.3, 0.7},
    NEBULA_BLUE = {0.3, 0.5, 1},
    CYAN_GLOW = {0.2, 0.8, 1},
    MAGENTA = {0.8, 0.2, 0.9},
    DARK_SPACE = {0.08, 0.05, 0.15},
    STAR_WHITE = {1, 0.95, 1},
}

-- ============================================================
-- UTILITY FUNCTIONS
-- ============================================================

-- Convert virtual key code to readable name
local function VKeyToString(vk)
    local keyNames = {
        [0x70] = "F1", [0x71] = "F2", [0x72] = "F3", [0x73] = "F4",
        [0x74] = "F5", [0x75] = "F6", [0x76] = "F7", [0x77] = "F8",
        [0x78] = "F9", [0x79] = "F10", [0x7A] = "F11", [0x7B] = "F12",
        [0x30] = "0", [0x31] = "1", [0x32] = "2", [0x33] = "3", [0x34] = "4",
        [0x35] = "5", [0x36] = "6", [0x37] = "7", [0x38] = "8", [0x39] = "9",
        [0x41] = "A", [0x42] = "B", [0x43] = "C", [0x44] = "D", [0x45] = "E",
        [0x46] = "F", [0x47] = "G", [0x48] = "H", [0x49] = "I", [0x4A] = "J",
        [0x4B] = "K", [0x4C] = "L", [0x4D] = "M", [0x4E] = "N", [0x4F] = "O",
        [0x50] = "P", [0x51] = "Q", [0x52] = "R", [0x53] = "S", [0x54] = "T",
        [0x55] = "U", [0x56] = "V", [0x57] = "W", [0x58] = "X", [0x59] = "Y",
        [0x5A] = "Z",
        [0x60] = "NUM0", [0x61] = "NUM1", [0x62] = "NUM2", [0x63] = "NUM3",
        [0x64] = "NUM4", [0x65] = "NUM5", [0x66] = "NUM6", [0x67] = "NUM7",
        [0x68] = "NUM8", [0x69] = "NUM9",
        [0x20] = "SPACE", [0x0D] = "ENTER", [0x1B] = "ESC", [0x09] = "TAB",
        [0x10] = "SHIFT", [0x11] = "CTRL", [0x12] = "ALT",
        [0xA0] = "LSHIFT", [0xA1] = "RSHIFT", [0xA2] = "LCTRL", [0xA3] = "RCTRL",
        [0x08] = "BACK", [0x2E] = "DEL", [0x21] = "PGUP", [0x22] = "PGDN",
        [0x23] = "END", [0x24] = "HOME", [0x2D] = "INS",
        [0x25] = "LEFT", [0x26] = "UP", [0x27] = "RIGHT", [0x28] = "DOWN",
        [0x13] = "PAUSE",
    }
    return keyNames[vk] or string.format("0x%X", vk)
end

local function DrawRoundedRect(x, y, w, h, r, g, b, a)
    Susano.DrawRectFilled(x, y, w, h, r, g, b, a)
end

local function DrawOutlineRect(x, y, w, h, r, g, b, a, thickness)
    Susano.DrawRect(x, y, w, h, r, g, b, a, thickness or 2)
end

-- Draw gradient effect (simulated with multiple rects)
local function DrawGradientRect(x, y, w, h, r1, g1, b1, r2, g2, b2, a)
    local steps = 20
    local stepH = h / steps
    for i = 0, steps - 1 do
        local progress = i / steps
        local r = r1 + (r2 - r1) * progress
        local g = g1 + (g2 - g1) * progress
        local b = b1 + (b2 - b1) * progress
        DrawRoundedRect(x, y + i * stepH, w, stepH + 1, r, g, b, a)
    end
end

-- Draw glow effect
local function DrawGlow(x, y, w, h, r, g, b, intensity)
    local glowSize = 8
    for i = 1, 3 do
        local offset = i * glowSize
        local alpha = intensity * (0.3 - i * 0.08)
        DrawRoundedRect(x - offset, y - offset, w + offset * 2, h + offset * 2, r, g, b, alpha)
    end
end

-- Check if mouse is over rect
local function IsMouseOver(x, y, w, h)
    local mx, my = InputStates.mouseX, InputStates.mouseY
    return mx >= x and mx <= x + w and my >= y and my <= y + h
end

-- Get mouse position using Susano native API
local function UpdateMousePosition()
    -- Use Susano's native cursor position function (more reliable)
    local cursorX, cursorY = Susano.GetCursorPos()
    InputStates.mouseX = cursorX or 0
    InputStates.mouseY = cursorY or 0
end

-- Handle drag and drop in edit mode
local function ProcessDragAndDrop()
    if not Features.editMode.enabled then
        InputStates.currentDragging = nil
        return
    end
    
    local leftDown, leftPressed = Susano.GetAsyncKeyState(CONFIG.LEFT_MOUSE)
    local mx, my = InputStates.mouseX, InputStates.mouseY
    
    -- Start dragging
    if leftPressed and not InputStates.lastLeftMouse then
        -- Check which element is clicked
        if not InputStates.currentDragging then
            local p = Features.performance
            local w = p.w * p.scale
            local h = p.h * p.scale
            if IsMouseOver(p.x, p.y, w, h) then
                InputStates.currentDragging = "performance"
                InputStates.dragOffsetX = mx - p.x
                InputStates.dragOffsetY = my - p.y
                p.dragging = true
            end
        end
        
        if not InputStates.currentDragging then
            local kv = Features.keyvis
            local kvW = kv.w * kv.scale
            local kvH = kv.h * kv.scale
            if IsMouseOver(kv.x - 10, kv.y - 35, kvW, kvH) then
                InputStates.currentDragging = "keyvis"
                InputStates.dragOffsetX = mx - kv.x
                InputStates.dragOffsetY = my - kv.y
                kv.dragging = true
            end
        end
        
        if not InputStates.currentDragging then
            local hud = Features.hud
            local w = hud.w * hud.scale
            local h = hud.h * hud.scale
            if IsMouseOver(hud.x, hud.y, w, h) then
                InputStates.currentDragging = "hud"
                InputStates.dragOffsetX = mx - hud.x
                InputStates.dragOffsetY = my - hud.y
                hud.dragging = true
            end
        end
        
        if not InputStates.currentDragging then
            local hl = Features.hitlog
            if IsMouseOver(hl.x, hl.y, 300, 300) then
                InputStates.currentDragging = "hitlog"
                InputStates.dragOffsetX = mx - hl.x
                InputStates.dragOffsetY = my - hl.y
                hl.dragging = true
            end
        end
        
        if not InputStates.currentDragging then
            local ch = Features.combatHud
            if IsMouseOver(ch.x, ch.y, 220, 110) then
                InputStates.currentDragging = "combathud"
                InputStates.dragOffsetX = mx - ch.x
                InputStates.dragOffsetY = my - ch.y
                ch.dragging = true
            end
        end
        
        if not InputStates.currentDragging then
            local wm = Features.watermark
            if IsMouseOver(wm.x, wm.y, 170, 28) then
                InputStates.currentDragging = "watermark"
                InputStates.dragOffsetX = mx - wm.x
                InputStates.dragOffsetY = my - wm.y
                wm.dragging = true
            end
        end
        
        -- Check menu header/title bar for dragging last (top 45px)
        if not InputStates.currentDragging then
            local m = Features.menu
            local w = 550
            local titleBarHeight = 45
            if IsMouseOver(m.x, m.y, w, titleBarHeight) then
                InputStates.currentDragging = "menu"
                InputStates.dragOffsetX = mx - m.x
                InputStates.dragOffsetY = my - m.y
                m.dragging = true
            end
        end
    end
    
    -- Update dragging position
    if leftDown and InputStates.currentDragging then
        local newX = mx - InputStates.dragOffsetX
        local newY = my - InputStates.dragOffsetY
        
        if InputStates.currentDragging == "menu" then
            Features.menu.x = newX
            Features.menu.y = newY
        elseif InputStates.currentDragging == "performance" then
            Features.performance.x = newX
            Features.performance.y = newY
        elseif InputStates.currentDragging == "keyvis" then
            Features.keyvis.x = newX
            Features.keyvis.y = newY
        elseif InputStates.currentDragging == "hud" then
            Features.hud.x = newX
            Features.hud.y = newY
        elseif InputStates.currentDragging == "hitlog" then
            Features.hitlog.x = newX
            Features.hitlog.y = newY
        elseif InputStates.currentDragging == "combathud" then
            Features.combatHud.x = newX
            Features.combatHud.y = newY
        elseif InputStates.currentDragging == "watermark" then
            Features.watermark.x = newX
            Features.watermark.y = newY
        end
    end
    
    -- Stop dragging
    if not leftDown and InputStates.currentDragging then
        Features.menu.dragging = false
        Features.performance.dragging = false
        Features.keyvis.dragging = false
        Features.hud.dragging = false
        Features.hitlog.dragging = false
        Features.combatHud.dragging = false
        Features.watermark.dragging = false
        InputStates.currentDragging = nil
    end
    
    InputStates.lastLeftMouse = leftPressed
end

-- ============================================================
-- FEATURE 1: INTERACTIVE MENU SYSTEM
-- ============================================================

local MenuItems = {
    {name = "--- VOID ENGINE ---", action = nil},
    {name = "Performance Monitor [F6]", action = function() Features.performance.enabled = not Features.performance.enabled end},
    {name = "Key Visualizer [F7]", action = function() Features.keyvis.enabled = not Features.keyvis.enabled end},
    {name = "Custom HUD [F9]", action = function() Features.hud.enabled = not Features.hud.enabled end},
    {name = "Hit Log", action = function() Features.hitlog.enabled = not Features.hitlog.enabled end},
    {name = "--- SETTINGS ---", action = nil},
    {name = "Edit Mode [F10]", action = function() Features.editMode.enabled = not Features.editMode.enabled end},
    {name = "Enable All", action = function() 
        Features.performance.enabled = true
        Features.keyvis.enabled = true
        Features.hud.enabled = true
        Features.hitlog.enabled = true
    end},
    {name = "Disable All", action = function() 
        Features.performance.enabled = false
        Features.keyvis.enabled = false
        Features.hud.enabled = false
        Features.hitlog.enabled = false
    end},
    {name = "Close Menu", action = function() Features.menu.enabled = false end},
}

local function RenderMenu()
    local m = Features.menu
    if not m.enabled and m.animProgress <= 0 then return end
    
    -- Animation
    if m.enabled then
        m.animProgress = math.min(1, m.animProgress + 0.15)
    else
        m.animProgress = math.max(0, m.animProgress - 0.15)
    end
    
    local alpha = m.animProgress
    local offsetX = (1 - alpha) * -50
    
    local x = m.x + offsetX
    local y = m.y
    local w = 550 -- Wider menu for two-column layout
    local h = 520 -- Updated height to match menu size
    
    -- Drag indicator when dragging
    if m.dragging then
        DrawGlow(x, y, w, h, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.2)
    end
    
    -- Main background with glow (reduced)
    DrawGlow(x, y, w, h, GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], alpha * 0.1)
    DrawRoundedRect(x, y, w, h, 0.05, 0.08, 0.12, 0.95 * alpha) -- Darker background like Neverlose
    
    -- Top header bar with gradient
    DrawGradientRect(x, y, w, 45, 0.08, 0.12, 0.18, 0.06, 0.10, 0.15, alpha * 0.9)
    DrawRoundedRect(x, y + 45, w, 2, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.6)
    
    -- Logo/Title with icon-like styling
    Susano.DrawCircle(x + 22, y + 22, 12, true, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.3, 1, 16)
    Susano.DrawCircle(x + 22, y + 22, 8, false, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * 0.8, 2, 16)
    
    -- Use custom font if loaded
    if CustomFonts.loaded and CustomFonts.title then
        Susano.PushFont(CustomFonts.title)
    end
    Susano.DrawText(x + 45, y + 13, "Void Engine Beta", 18, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], alpha)
    if CustomFonts.loaded and CustomFonts.title then
        Susano.PopFont()
    end
    
    -- Update toggle animations with smoother speed (0.15 for buttery smoothness)
    Features.performance.toggleAnim = Features.performance.enabled and math.min(1, Features.performance.toggleAnim + 0.15) or math.max(0, Features.performance.toggleAnim - 0.15)
    Features.keyvis.toggleAnim = Features.keyvis.enabled and math.min(1, Features.keyvis.toggleAnim + 0.15) or math.max(0, Features.keyvis.toggleAnim - 0.15)
    Features.hud.toggleAnim = Features.hud.enabled and math.min(1, Features.hud.toggleAnim + 0.15) or math.max(0, Features.hud.toggleAnim - 0.15)
    Features.hitlog.toggleAnim = Features.hitlog.enabled and math.min(1, Features.hitlog.toggleAnim + 0.15) or math.max(0, Features.hitlog.toggleAnim - 0.15)
    Features.healCover.toggleAnim = Features.healCover.enabled and math.min(1, Features.healCover.toggleAnim + 0.15) or math.max(0, Features.healCover.toggleAnim - 0.15)
    Features.noCombatStance.toggleAnim = Features.noCombatStance.enabled and math.min(1, Features.noCombatStance.toggleAnim + 0.15) or math.max(0, Features.noCombatStance.toggleAnim - 0.15)
    Features.combatHud.toggleAnim = Features.combatHud.enabled and math.min(1, Features.combatHud.toggleAnim + 0.15) or math.max(0, Features.combatHud.toggleAnim - 0.15)
    Features.watermark.toggleAnim = Features.watermark.enabled and math.min(1, Features.watermark.toggleAnim + 0.15) or math.max(0, Features.watermark.toggleAnim - 0.15)
    Features.esp.toggleAnim = Features.esp.enabled and math.min(1, Features.esp.toggleAnim + 0.15) or math.max(0, Features.esp.toggleAnim - 0.15)
    Features.rainbowMode.toggleAnim = Features.rainbowMode.enabled and math.min(1, Features.rainbowMode.toggleAnim + 0.15) or math.max(0, Features.rainbowMode.toggleAnim - 0.15)
    Features.editMode.toggleAnim = Features.editMode.enabled and math.min(1, Features.editMode.toggleAnim + 0.15) or math.max(0, Features.editMode.toggleAnim - 0.15)
    Features.fpsBoost.toggleAnim = Features.fpsBoost.enabled and math.min(1, Features.fpsBoost.toggleAnim + 0.15) or math.max(0, Features.fpsBoost.toggleAnim - 0.15)
    
    -- Tab bar
    local tabY = y + 52
    local tabW = w / 3
    local tabH = 35
    local tabs = {{"COMBAT", 1}, {"VISUAL", 2}, {"UTILITY", 3}}
    
    for i, tab in ipairs(tabs) do
        local tabX = x + (i - 1) * tabW
        local isActive = (m.activeTab == tab[2])
        
        -- Tab background
        if isActive then
            DrawRoundedRect(tabX, tabY, tabW, tabH, GALAXY.COSMIC_PINK[1] * 0.3, GALAXY.COSMIC_PINK[2] * 0.3, GALAXY.COSMIC_PINK[3] * 0.3, alpha * 0.5)
            DrawRoundedRect(tabX, tabY + tabH - 2, tabW, 2, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha)
        else
            DrawRoundedRect(tabX, tabY, tabW, tabH, 0.1, 0.13, 0.16, alpha * 0.3)
        end
        
        -- Tab text
        local textColor = isActive and GALAXY.COSMIC_PINK or {0.5, 0.55, 0.6}
        local textSize = 14
        local textWidth = Susano.GetTextWidth(tab[1], textSize)
        local textX = tabX + (tabW / 2) - (textWidth / 2)
        Susano.DrawText(textX, tabY + 10, tab[1], textSize, textColor[1], textColor[2], textColor[3], alpha)
    end
    
    -- Define toggleable items with their state (get keys from CONFIG)
    local toggleItems = {
        -- COMBAT category (items 1-2)
        {name = "Heal Behind Cover", key = "[" .. VKeyToString(CONFIG.HEAL_COVER_KEY) .. "]", enabled = Features.healCover.enabled, feature = "healcover", animProgress = Features.healCover.toggleAnim, category = "COMBAT", tabId = 1},
        {name = "No Combat Stance", key = "[" .. VKeyToString(CONFIG.COMBAT_STANCE_KEY) .. "]", enabled = Features.noCombatStance.enabled, feature = "nocombat", animProgress = Features.noCombatStance.toggleAnim, category = "COMBAT", tabId = 1},
        -- VISUAL category (items 3-10)
        {name = "Performance Monitor", key = CONFIG.PERF_KEY == 0x13 and "NO KEYBIND" or "[" .. VKeyToString(CONFIG.PERF_KEY) .. "]", enabled = Features.performance.enabled, feature = "performance", animProgress = Features.performance.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "Key Visualizer", key = CONFIG.KEYS_KEY == 0x13 and "NO KEYBIND" or "[" .. VKeyToString(CONFIG.KEYS_KEY) .. "]", enabled = Features.keyvis.enabled, feature = "keyvis", animProgress = Features.keyvis.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "Custom HUD", key = CONFIG.HUD_KEY == 0x13 and "NO KEYBIND" or "[" .. VKeyToString(CONFIG.HUD_KEY) .. "]", enabled = Features.hud.enabled, feature = "hud", animProgress = Features.hud.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "Hit Log", key = "[" .. VKeyToString(CONFIG.HITLOG_KEY) .. "]", enabled = Features.hitlog.enabled, feature = "hitlog", animProgress = Features.hitlog.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "Combat HUD", key = "[" .. VKeyToString(CONFIG.COMBAT_HUD_KEY) .. "]", enabled = Features.combatHud.enabled, feature = "combathud", animProgress = Features.combatHud.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "3D ESP", key = "[" .. VKeyToString(CONFIG.ESP_KEY) .. "]", enabled = Features.esp.enabled, feature = "esp", animProgress = Features.esp.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "Rainbow Mode", key = "[" .. VKeyToString(CONFIG.RAINBOW_KEY) .. "]", enabled = Features.rainbowMode.enabled, feature = "rainbow", animProgress = Features.rainbowMode.toggleAnim, category = "VISUAL", tabId = 2},
        {name = "Watermark", key = CONFIG.WATERMARK_KEY == 0x13 and "[NONE]" or "[" .. VKeyToString(CONFIG.WATERMARK_KEY) .. "]", enabled = Features.watermark.enabled, feature = "watermark", animProgress = Features.watermark.toggleAnim, category = "VISUAL", tabId = 2},
        -- UTILITY category (items 11-12)
        {name = "Edit Mode", key = CONFIG.EDIT_MODE_KEY == 0x13 and "NO KEYBIND" or "[" .. VKeyToString(CONFIG.EDIT_MODE_KEY) .. "]", enabled = Features.editMode.enabled, feature = "editmode", animProgress = Features.editMode.toggleAnim, category = "UTILITY", tabId = 3},
        {name = "FPS Booster", key = "[" .. VKeyToString(CONFIG.FPS_BOOST_KEY) .. "]", enabled = Features.fpsBoost.enabled, feature = "fpsboost", animProgress = Features.fpsBoost.toggleAnim, category = "UTILITY", tabId = 3},
    }
    
    -- Filter items by active tab
    local visibleItems = {}
    for _, item in ipairs(toggleItems) do
        if item.tabId == m.activeTab then
            table.insert(visibleItems, item)
        end
    end
    
    -- Draw items in grid layout
    local leftX = x + 20
    local rightX = x + w/2 + 10
    local startY = y + 100
    local itemHeight = 32
    local categoryY = startY
    
    -- Render visible items for current tab (single column layout)
    for i, item in ipairs(visibleItems) do
        local itemY = startY + (i - 1) * itemHeight
        local isSelected = (i == m.selectedIndex)
        
        -- Hover/selected background
        if isSelected then
            DrawRoundedRect(leftX - 5, itemY - 2, w - 40, itemHeight - 4, 0.1, 0.15, 0.2, 0.6 * alpha)
            DrawRoundedRect(leftX - 5, itemY - 2, 2, itemHeight - 4, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha)
        end
        
        -- Item name
        local textColor = isSelected and GALAXY.STAR_WHITE or {0.7, 0.75, 0.8}
        Susano.DrawText(leftX, itemY, item.name, 14, textColor[1], textColor[2], textColor[3], alpha)
        
        -- Switch toggle indicator with smooth animation
        local toggleX = leftX + 200
        local toggleY = itemY + 7
        
        -- Smooth ease-out-back for bounce effect
        local function easeOutBack(t)
            local c1 = 1.70158
            local c3 = c1 + 1
            return 1 + c3 * math.pow(t - 1, 3) + c1 * math.pow(t - 1, 2)
        end
        
        local animEased = easeOutBack(item.animProgress)
        
        -- Toggle background with smooth color transition
        local bgR = 0.15 + (GALAXY.CYAN_GLOW[1] - 0.15) * item.animProgress
        local bgG = 0.2 + (GALAXY.CYAN_GLOW[2] - 0.2) * item.animProgress
        local bgB = 0.25 + (GALAXY.CYAN_GLOW[3] - 0.25) * item.animProgress
        local bgAlpha = alpha * (0.5 + 0.4 * item.animProgress)
        
        -- Background shadow/outline
        DrawRoundedRect(toggleX - 1, toggleY - 1, 34, 16, bgR * 0.5, bgG * 0.5, bgB * 0.5, bgAlpha * 0.3)
        DrawRoundedRect(toggleX, toggleY, 32, 14, bgR, bgG, bgB, bgAlpha)
        
        -- Toggle circle with smooth position animation
        local circleStartX = toggleX + 7
        local circleEndX = toggleX + 21
        local circleX = circleStartX + (circleEndX - circleStartX) * animEased
        
        -- Multi-layer glow effect when enabled
        if item.animProgress > 0 then
            Susano.DrawCircle(circleX, toggleY + 7, 10, true, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * 0.15 * item.animProgress, 1, 16)
            Susano.DrawCircle(circleX, toggleY + 7, 8, true, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * 0.25 * item.animProgress, 1, 14)
            Susano.DrawCircle(circleX, toggleY + 7, 7, true, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], alpha * 0.4 * item.animProgress, 1, 12)
        end
        
        -- Main toggle circle with subtle scale effect
        local circleSize = 6 + (item.animProgress * 0.5)
        Susano.DrawCircle(circleX, toggleY + 7, circleSize, true, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], alpha, 1, 12)
        
        -- Keybind (clickable for rebinding) - show for all items with keys
        if item.key ~= "" then
            local keybindX = toggleX + 40
            local keybindW = item.key == "NO KEYBIND" and 80 or 50
            local keybindH = 18
            
            -- Check if we're rebinding this feature
            local isRebinding = Features.rebinding.active and Features.rebinding.targetFeature == item.feature
            local isHoveringKeybind = IsMouseOver(keybindX - 2, itemY - 2, keybindW, keybindH)
            
            -- Draw keybind background if hovering
            if isHoveringKeybind and not isRebinding then
                DrawRoundedRect(keybindX - 2, itemY - 2, keybindW, keybindH, 0.15, 0.2, 0.25, alpha * 0.8)
            end
            
            -- Draw keybind text or "waiting" message
            if isRebinding then
                DrawRoundedRect(keybindX - 2, itemY - 2, 80, keybindH, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.3)
                Susano.DrawText(keybindX, itemY, "Press key...", 11, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha)
            else
                Susano.DrawText(keybindX, itemY, item.key, 11, isHoveringKeybind and 0.8 or 0.5, isHoveringKeybind and 0.85 or 0.55, isHoveringKeybind and 0.9 or 0.6, alpha * 0.7)
            end
        end
    end
    
    -- Bottom section - Quick actions
    local bottomY = y + h - 95
    DrawRoundedRect(x, bottomY, w, 1, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.3)
    
    Susano.DrawText(leftX, bottomY + 15, "QUICK ACTIONS", 15, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * 0.9)
    
    -- Action buttons
    local btnY = bottomY + 45
    local btn1Selected = (m.selectedIndex == 13)
    local btn2Selected = (m.selectedIndex == 14)
    local btn3Selected = (m.selectedIndex == 15)
    
    -- Enable All button
    if btn1Selected then
        DrawRoundedRect(leftX - 3, btnY - 3, 90, 26, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.3)
    end
    DrawRoundedRect(leftX, btnY, 85, 22, 0.1, 0.15, 0.2, alpha * 0.7)
    DrawOutlineRect(leftX, btnY, 85, 22, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * (btn1Selected and 0.9 or 0.4), 1)
    Susano.DrawText(leftX + 10, btnY + 5, "Enable All", 13, btn1Selected and GALAXY.STAR_WHITE[1] or 0.7, btn1Selected and GALAXY.STAR_WHITE[2] or 0.75, btn1Selected and GALAXY.STAR_WHITE[3] or 0.8, alpha)
    
    -- Disable All button
    if btn2Selected then
        DrawRoundedRect(leftX + 92, btnY - 3, 95, 26, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.3)
    end
    DrawRoundedRect(leftX + 95, btnY, 90, 22, 0.1, 0.15, 0.2, alpha * 0.7)
    DrawOutlineRect(leftX + 95, btnY, 90, 22, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * (btn2Selected and 0.9 or 0.4), 1)
    Susano.DrawText(leftX + 103, btnY + 5, "Disable All", 13, btn2Selected and GALAXY.STAR_WHITE[1] or 0.7, btn2Selected and GALAXY.STAR_WHITE[2] or 0.75, btn2Selected and GALAXY.STAR_WHITE[3] or 0.8, alpha)
    
    -- Close Menu button
    if btn3Selected then
        DrawRoundedRect(leftX + 192, btnY - 3, 110, 26, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha * 0.3)
    end
    DrawRoundedRect(leftX + 195, btnY, 105, 22, 0.1, 0.15, 0.2, alpha * 0.7)
    DrawOutlineRect(leftX + 195, btnY, 105, 22, GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], alpha * (btn3Selected and 0.9 or 0.4), 1)
    Susano.DrawText(leftX + 203, btnY + 5, "Close Menu", 13, btn3Selected and GALAXY.STAR_WHITE[1] or 0.7, btn3Selected and GALAXY.STAR_WHITE[2] or 0.75, btn3Selected and GALAXY.STAR_WHITE[3] or 0.8, alpha)
    
    -- Footer info
    Susano.DrawText(x + 15, y + h - 20, "F5: Close Menu | Click: Toggle", 11, 0.4, 0.45, 0.5, alpha * 0.8)
    
    -- Mouse cursor indicator when menu is open
    local mx, my = InputStates.mouseX, InputStates.mouseY
    Susano.DrawCircle(mx, my, 8, true, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.9 * alpha, 2, 16)
    Susano.DrawCircle(mx, my, 12, false, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.7 * alpha, 2, 16)
end

local function ProcessMenuInput()
    if not Features.menu.enabled then return end
    
    local m = Features.menu
    local mx, my = InputStates.mouseX, InputStates.mouseY
    local leftDown, leftPressed = Susano.GetAsyncKeyState(CONFIG.LEFT_MOUSE)
    
    -- Calculate menu position and dimensions
    local alpha = m.animProgress
    local offsetX = (1 - alpha) * -50
    local x = m.x + offsetX
    local y = m.y
    local w = 550
    local h = 520
    
    local leftX = x + 20
    local rightX = x + w/2 + 10
    local startY = y + 70
    local itemHeight = 32
    local categoryY = startY + 30
    
    -- Keybind data for click detection (must match menu display)
    local toggleItems = {
        {name = "Heal Behind Cover", key = "[" .. VKeyToString(CONFIG.HEAL_COVER_KEY) .. "]", enabled = Features.healCover.enabled, feature = "healcover", configKey = "HEAL_COVER_KEY", tabId = 1},
        {name = "No Combat Stance", key = "[" .. VKeyToString(CONFIG.COMBAT_STANCE_KEY) .. "]", enabled = Features.noCombatStance.enabled, feature = "nocombat", configKey = "COMBAT_STANCE_KEY", tabId = 1},
        {name = "Performance Monitor", key = "[" .. VKeyToString(CONFIG.PERF_KEY) .. "]", enabled = Features.performance.enabled, feature = "performance", configKey = "PERF_KEY", tabId = 2},
        {name = "Key Visualizer", key = "[" .. VKeyToString(CONFIG.KEYS_KEY) .. "]", enabled = Features.keyvis.enabled, feature = "keyvis", configKey = "KEYS_KEY", tabId = 2},
        {name = "Custom HUD", key = "[" .. VKeyToString(CONFIG.HUD_KEY) .. "]", enabled = Features.hud.enabled, feature = "hud", configKey = "HUD_KEY", tabId = 2},
        {name = "Hit Log", key = "[" .. VKeyToString(CONFIG.HITLOG_KEY) .. "]", enabled = Features.hitlog.enabled, feature = "hitlog", configKey = "HITLOG_KEY", tabId = 2},
        {name = "Combat HUD", key = CONFIG.COMBAT_HUD_KEY == 0x13 and "[NONE]" or "[" .. VKeyToString(CONFIG.COMBAT_HUD_KEY) .. "]", enabled = Features.combatHud.enabled, feature = "combathud", configKey = "COMBAT_HUD_KEY", tabId = 2},
        {name = "3D ESP", key = "[" .. VKeyToString(CONFIG.ESP_KEY) .. "]", enabled = Features.esp.enabled, feature = "esp", configKey = "ESP_KEY", tabId = 2},
        {name = "Rainbow Mode", key = "[" .. VKeyToString(CONFIG.RAINBOW_KEY) .. "]", enabled = Features.rainbowMode.enabled, feature = "rainbow", configKey = "RAINBOW_KEY", tabId = 2},
        {name = "Watermark", key = CONFIG.WATERMARK_KEY == 0x13 and "[NONE]" or "[" .. VKeyToString(CONFIG.WATERMARK_KEY) .. "]", enabled = Features.watermark.enabled, feature = "watermark", configKey = "WATERMARK_KEY", tabId = 2},
        {name = "Edit Mode", key = "[" .. VKeyToString(CONFIG.EDIT_MODE_KEY) .. "]", enabled = Features.editMode.enabled, feature = "editmode", configKey = "EDIT_MODE_KEY", tabId = 3},
        {name = "FPS Booster", key = "[" .. VKeyToString(CONFIG.FPS_BOOST_KEY) .. "]", enabled = Features.fpsBoost.enabled, feature = "fpsboost", configKey = "FPS_BOOST_KEY", tabId = 3},
    }
    
    -- If waiting for key rebind, capture any key
    if Features.rebinding.active then
        -- Scan for any key press (excluding mouse buttons)
        for vk = 0x08, 0xFE do
            if vk ~= 0x01 and vk ~= 0x02 then -- Exclude left/right mouse
                local down, pressed = Susano.GetAsyncKeyState(vk)
                if pressed then
                    -- Update the config with new key
                    local targetItem = nil
                    for _, item in ipairs(toggleItems) do
                        if item.feature == Features.rebinding.targetFeature then
                            targetItem = item
                            break
                        end
                    end
                    
                    if targetItem and targetItem.configKey then
                        CONFIG[targetItem.configKey] = vk
                    end
                    
                    -- Exit rebinding mode
                    Features.rebinding.active = false
                    Features.rebinding.targetFeature = nil
                    return
                end
            end
        end
        return -- Don't process other clicks while rebinding
    end
    
    -- Check tab clicks
    local tabY = y + 52
    local tabW = w / 3
    local tabH = 35
    for i = 1, 3 do
        local tabX = x + (i - 1) * tabW
        if leftPressed and not InputStates.lastLeftMouse and IsMouseOver(tabX, tabY, tabW, tabH) then
            m.activeTab = i
            InputStates.lastLeftMouse = leftPressed
            return
        end
    end
    
    -- Reset selected index (hover-based)
    m.selectedIndex = 0
    
    -- Get visible items for current tab
    local visibleItems = {}
    for _, item in ipairs(toggleItems) do
        if item.tabId == m.activeTab then
            table.insert(visibleItems, item)
        end
    end
    
    -- Check hover for visible items
    local startY = y + 100
    local itemHeight = 32
    for i, item in ipairs(visibleItems) do
        local itemY = startY + (i - 1) * itemHeight
        if IsMouseOver(leftX - 5, itemY - 2, w - 40, itemHeight - 4) then
            m.selectedIndex = i
        end
    end
    
    -- Check hover for action buttons (adjust based on visible items)
    local bottomY = y + h - 95
    local btnY = bottomY + 45
    
    if IsMouseOver(leftX, btnY, 85, 22) then
        m.selectedIndex = 13 -- Enable All
    elseif IsMouseOver(leftX + 95, btnY, 90, 22) then
        m.selectedIndex = 14 -- Disable All
    elseif IsMouseOver(leftX + 195, btnY, 105, 22) then
        m.selectedIndex = 15 -- Close Menu
    end
    
    -- Handle mouse clicks (only when not in edit mode)
    if leftPressed and not InputStates.lastLeftMouse then
        local idx = m.selectedIndex
        
        -- Check toggle switch clicks for visible items
        local toggleX = leftX + 200
        for i, item in ipairs(visibleItems) do
            local itemY = startY + (i - 1) * itemHeight
            local toggleY = itemY + 7
            
            -- Check if clicking on toggle switch area (32px wide, 14px tall)
            if IsMouseOver(toggleX, toggleY, 32, 14) then
                -- Toggle the feature
                if item.feature == "healcover" then
                    Features.healCover.enabled = not Features.healCover.enabled
                elseif item.feature == "nocombat" then
                    Features.noCombatStance.enabled = not Features.noCombatStance.enabled
                elseif item.feature == "performance" then
                    Features.performance.enabled = not Features.performance.enabled
                elseif item.feature == "keyvis" then
                    Features.keyvis.enabled = not Features.keyvis.enabled
                elseif item.feature == "hud" then
                    Features.hud.enabled = not Features.hud.enabled
                elseif item.feature == "hitlog" then
                    Features.hitlog.enabled = not Features.hitlog.enabled
                elseif item.feature == "combathud" then
                    Features.combatHud.enabled = not Features.combatHud.enabled
                elseif item.feature == "esp" then
                    Features.esp.enabled = not Features.esp.enabled
                elseif item.feature == "rainbow" then
                    Features.rainbowMode.enabled = not Features.rainbowMode.enabled
                elseif item.feature == "watermark" then
                    Features.watermark.enabled = not Features.watermark.enabled
                elseif item.feature == "editmode" then
                    Features.editMode.enabled = not Features.editMode.enabled
                elseif item.feature == "fpsboost" then
                    Features.fpsBoost.enabled = not Features.fpsBoost.enabled
                end
                InputStates.lastLeftMouse = leftPressed
                return
            end
        end
        
        -- Check keybind clicks for visible items
        for i, item in ipairs(visibleItems) do
            local itemY = startY + (i - 1) * itemHeight
            local keybindX = toggleX + 40
            local keybindW = item.key == "NO KEYBIND" and 80 or 50
            
            if item.key ~= "" and IsMouseOver(keybindX - 2, itemY - 2, keybindW, 18) then
                -- Start rebinding
                Features.rebinding.active = true
                Features.rebinding.targetFeature = item.feature
                InputStates.lastLeftMouse = leftPressed
                return
            end
        end
        
        -- Toggle items (dynamically based on visible items)
        if idx > 0 and idx <= #visibleItems then
            local item = visibleItems[idx]
            if item.feature == "healcover" then
                Features.healCover.enabled = not Features.healCover.enabled
            elseif item.feature == "nocombat" then
                Features.noCombatStance.enabled = not Features.noCombatStance.enabled
            elseif item.feature == "performance" then
                Features.performance.enabled = not Features.performance.enabled
            elseif item.feature == "keyvis" then
                Features.keyvis.enabled = not Features.keyvis.enabled
            elseif item.feature == "hud" then
                Features.hud.enabled = not Features.hud.enabled
            elseif item.feature == "hitlog" then
                Features.hitlog.enabled = not Features.hitlog.enabled
            elseif item.feature == "combathud" then
                Features.combatHud.enabled = not Features.combatHud.enabled
            elseif item.feature == "esp" then
                Features.esp.enabled = not Features.esp.enabled
            elseif item.feature == "rainbow" then
                Features.rainbowMode.enabled = not Features.rainbowMode.enabled
            elseif item.feature == "watermark" then
                Features.watermark.enabled = not Features.watermark.enabled
            elseif item.feature == "editmode" then
                Features.editMode.enabled = not Features.editMode.enabled
            elseif item.feature == "fpsboost" then
                Features.fpsBoost.enabled = not Features.fpsBoost.enabled
            end
        -- Action buttons (13-15)
        elseif idx == 13 then
            -- Enable All
            Features.healCover.enabled = true
            Features.noCombatStance.enabled = true
            Features.performance.enabled = true
            Features.keyvis.enabled = true
            Features.hud.enabled = true
            Features.hitlog.enabled = true
            Features.combatHud.enabled = true
            Features.esp.enabled = true
            Features.rainbowMode.enabled = true
            Features.watermark.enabled = true
            Features.editMode.enabled = true
            Features.fpsBoost.enabled = true
        elseif idx == 14 then
            -- Disable All
            Features.healCover.enabled = false
            Features.noCombatStance.enabled = false
            Features.performance.enabled = false
            Features.keyvis.enabled = false
            Features.hud.enabled = false
            Features.hitlog.enabled = false
            Features.combatHud.enabled = false
            Features.esp.enabled = false
            Features.rainbowMode.enabled = false
            Features.watermark.enabled = false
            Features.editMode.enabled = false
            Features.fpsBoost.enabled = false
        elseif idx == 15 then
            -- Close Menu
            Features.menu.enabled = false
        end
    end
    
    InputStates.lastLeftMouse = leftPressed
end

-- ============================================================
-- FEATURE 2: PERFORMANCE MONITOR (CLEAN)
-- ============================================================

local function UpdatePerformance()
    local p = Features.performance
    local currentTime = GetGameTimer()
    
    if p.lastFrameTime == 0 then
        p.lastFrameTime = currentTime
        p.lastUpdateTime = currentTime
        return
    end
    
    local deltaTime = currentTime - p.lastFrameTime
    if deltaTime > 0 then
        p.currentFPS = math.floor(1000 / deltaTime)
    end
    
    -- Only update display FPS every 1 second to prevent flickering
    if (currentTime - p.lastUpdateTime) >= 1000 then
        p.displayFPS = p.currentFPS
        p.lastUpdateTime = currentTime
    end
    
    -- Initialize displayFPS on first run
    if p.displayFPS == 0 then
        p.displayFPS = p.currentFPS
    end
    
    p.lastFrameTime = currentTime
end

local function RenderPerformance()
    if not Features.performance.enabled then return end
    
    local p = Features.performance
    local x, y = p.x, p.y
    local w, h = 180, 80 -- Increased size for better aesthetics
    
    -- Drag indicator with reduced glow
    if p.dragging then
        DrawGlow(x, y, w, h, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.2)
    end
    
    -- Background with reduced glow
    DrawGlow(x, y, w, h, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.08)
    DrawRoundedRect(x, y, w, h, 0.05, 0.08, 0.12, 0.95)
    
    -- Top header bar with gradient
    DrawGradientRect(x, y, w, 30, 0.08, 0.12, 0.18, 0.06, 0.10, 0.15, 0.9)
    DrawRoundedRect(x, y + 30, w, 2, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.6)
    
    -- Icon circle
    Susano.DrawCircle(x + 15, y + 15, 10, true, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.3, 1, 16)
    Susano.DrawCircle(x + 15, y + 15, 7, false, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.8, 2, 16)
    
    -- Title
    Susano.DrawText(x + 30, y + 8, "Performance", 16, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
    
    -- FPS display with dynamic color - clean and sharp
    local fpsColor = p.displayFPS >= 60 and GALAXY.CYAN_GLOW or (p.displayFPS >= 30 and {1, 0.8, 0.3} or GALAXY.COSMIC_PINK)
    local fpsText = tostring(p.displayFPS)
    local fpsSize = 24 -- Smaller size for crisp rendering
    local fpsY = y + 38
    
    -- Use GetTextWidth for perfect centering
    local fpsWidth = Susano.GetTextWidth(fpsText, fpsSize)
    local fpsX = x + (w / 2) - (fpsWidth / 2)
    
    -- Single clean render - no multiple layers that cause blur
    Susano.DrawText(fpsX, fpsY, fpsText, fpsSize, fpsColor[1], fpsColor[2], fpsColor[3], 1)
    
    -- "FPS" label centered below
    local labelText = "FPS"
    local labelSize = 13
    local labelWidth = Susano.GetTextWidth(labelText, labelSize)
    Susano.DrawText(x + (w / 2) - (labelWidth / 2), fpsY + 28, labelText, labelSize, 0.6, 0.65, 0.7, 0.9)
end

-- ============================================================
-- FEATURE 3: KEY PRESS VISUALIZER
-- ============================================================

local KeyDefs = {
    {vk = 0x57, label = "W", x = 1, y = 0},
    {vk = 0x41, label = "A", x = 0, y = 1},
    {vk = 0x53, label = "S", x = 1, y = 1},
    {vk = 0x44, label = "D", x = 2, y = 1},
    {vk = 0x20, label = "SPC", x = 0.5, y = 2, w = 2},
    {vk = 0x01, label = "LMB", x = 4, y = 0},
    {vk = 0x02, label = "RMB", x = 5, y = 0},
}

for _, key in ipairs(KeyDefs) do
    Features.keyvis.keyStates[key.vk] = {pressed = false, anim = 0}
end

local function RenderKeyVisualizer()
    if not Features.keyvis.enabled then return end
    
    local kv = Features.keyvis
    local keySize = 45
    local spacing = 6
    local baseX = kv.x
    local baseY = kv.y
    
    -- Drag indicator
    if kv.dragging then
        DrawGlow(baseX - 10, baseY - 35, 320, 195, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.15)
    end
    
    -- Background panel with reduced glow
    DrawGlow(baseX - 10, baseY - 10, 320, 160, GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], 0.08)
    DrawRoundedRect(baseX - 10, baseY - 10, 320, 160, GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.85)
    
    -- Title with gradient
    DrawGradientRect(baseX - 10, baseY - 35, 320, 22, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3],
                     GALAXY.NEBULA_BLUE[1], GALAXY.NEBULA_BLUE[2], GALAXY.NEBULA_BLUE[3], 0.7)
    Susano.DrawText(baseX, baseY - 30, "KEYPRESS", 15, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
    
    for _, key in ipairs(KeyDefs) do
        local down = Susano.GetAsyncKeyState(key.vk)
        local state = kv.keyStates[key.vk]
        
        if down and not state.pressed then
            state.pressed = true
            state.anim = 1
        elseif not down and state.pressed then
            state.pressed = false
        end
        
        if state.anim > 0 then
            state.anim = state.anim - 0.1
        end
        
        local keyW = (key.w or 1) * keySize + ((key.w or 1) - 1) * spacing
        local x = baseX + key.x * (keySize + spacing)
        local y = baseY + key.y * (keySize + spacing)
        
        local scale = 1 - (state.anim * 0.1)
        local scaledW = keyW * scale
        local scaledH = keySize * scale
        local offsetX = (keyW - scaledW) / 2
        local offsetY = (keySize - scaledH) / 2
        
        if state.pressed then
            -- Pressed key with galaxy glow
            DrawGlow(x + offsetX, y + offsetY, scaledW, scaledH, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.5)
            DrawGradientRect(x + offsetX, y + offsetY, scaledW, scaledH, 
                           GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3],
                           GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], 0.95)
            DrawOutlineRect(x + offsetX, y + offsetY, scaledW, scaledH, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 1, 2)
        else
            -- Unpressed key
            DrawRoundedRect(x + offsetX, y + offsetY, scaledW, scaledH, GALAXY.DARK_SPACE[1] * 2, GALAXY.DARK_SPACE[2] * 2, GALAXY.DARK_SPACE[3] * 2, 0.8)
            DrawOutlineRect(x + offsetX, y + offsetY, scaledW, scaledH, GALAXY.DEEP_PURPLE[1], GALAXY.DEEP_PURPLE[2], GALAXY.DEEP_PURPLE[3], 0.5, 2)
        end
        
        local textX = x + (keyW - #key.label * 8) / 2
        local textY = y + (keySize - 15) / 2
        local textColor = state.pressed and GALAXY.STAR_WHITE or {0.7, 0.7, 0.8}
        Susano.DrawText(textX, textY, key.label, 15, textColor[1], textColor[2], textColor[3], 1)
    end
end

-- ============================================================
-- FEATURE 4: LOADING ANIMATION
-- ============================================================

local function RenderLoadingAnimation()
    local load = Features.loading
    if not load.active then return end
    
    -- Safety check: if startTime is 0, initialize it now
    if load.startTime == 0 then
        load.startTime = GetGameTimer()
    end
    
    -- Calculate animation progress (0 to 1)
    local currentTime = GetGameTimer()
    local elapsed = (currentTime - load.startTime) / 1000
    local loadDuration = 2.5 -- Animation to 100%
    local holdDuration = 2.0 -- Hold at 100%
    local totalDuration = loadDuration + holdDuration
    
    -- Safety: disable after 10 seconds regardless
    if elapsed > 10 then
        load.active = false
        return
    end
    
    -- Quick skip: Press ENTER to skip loading animation
    local enterDown, enterPressed = Susano.GetAsyncKeyState(0x0D) -- Enter key
    if enterPressed then
        load.active = false
        return
    end
    
    -- Smooth easing function (ease out cubic)
    local function easeOutCubic(t)
        return 1 - math.pow(1 - t, 3)
    end
    
    if elapsed < loadDuration then
        -- Loading phase with smooth easing
        local rawProgress = elapsed / loadDuration
        load.progress = easeOutCubic(rawProgress)
        -- Keep at 100% without fading out to dark
        if load.progress >= 100 then
            if load.holdTime >= 2000 then -- Hold for 2 seconds at 100%
                load.active = false
                return
            end
            alpha = 1 -- Stay at full brightness
        end
    elseif elapsed < totalDuration then
        -- Hold at 100%
        load.progress = 1
        load.holdTime = elapsed - loadDuration
    else
        -- Fade out and end
        load.active = false
        return
    end
    
    local screenW = CONFIG.SCREEN_W
    local screenH = CONFIG.SCREEN_H
    
    -- Keep at full brightness - no fade out to dark
    local fadeOutAlpha = 1
    
    -- Full screen dark overlay - consistent brightness throughout
    local overlayAlpha = 0.75 -- Keep constant at moderate level
    DrawRoundedRect(0, 0, screenW, screenH, 0, 0, 0, overlayAlpha)
    
    -- Calculate position (moves from bottom to center, perfectly centered)
    local startY = screenH + 200
    local endY = (screenH / 2) - 100
    local y = startY + (endY - startY) * easeOutCubic(math.min(load.progress * 1.2, 1))
    
    local w = 500
    local h = 200
    local x = (screenW / 2) - (w / 2) -- Perfectly centered
    
    -- Fade in alpha
    local alpha = math.min(load.progress * 2.5, 1) * fadeOutAlpha
    
    -- Logo box with reduced glow
    DrawGlow(x, y, w, h, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.2 * alpha)
    DrawGradientRect(x, y, w, h, 
                     GALAXY.DEEP_PURPLE[1], GALAXY.DEEP_PURPLE[2], GALAXY.DEEP_PURPLE[3],
                     GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.95 * alpha)
    
    -- Border glow
    DrawOutlineRect(x, y, w, h, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.9 * alpha, 3)
    
    -- Title in upper area - 28px for optimal sharpness
    local titleText = "VOID ENGINE BETA"
    local titleSize = 28
    local titleWidth = Susano.GetTextWidth(titleText, titleSize)
    local titleX = x + (w / 2) - (titleWidth / 2) -- Perfectly centered using GetTextWidth
    local titleY = y + 40 -- Position in upper area of box
    
    if CustomFonts.loaded and CustomFonts.title then
        Susano.PushFont(CustomFonts.title)
    end
    Susano.DrawText(titleX, titleY, titleText, titleSize, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], alpha)
    if CustomFonts.loaded and CustomFonts.title then
        Susano.PopFont()
    end
    
    -- Subtitle message centered using GetTextWidth for precision
    local subtitleText = "Please wait a moment..."
    local subtitleSize = 16
    local subtitleWidth = Susano.GetTextWidth(subtitleText, subtitleSize)
    local subtitleX = x + (w / 2) - (subtitleWidth / 2) -- Perfectly centered
    local subtitleY = y + 95 -- Between title and progress bar
    Susano.DrawText(subtitleX, subtitleY, subtitleText, subtitleSize, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * 0.85)
    
    -- Loading bar
    local barW = w - 60
    local barH = 10
    local barX = x + 30
    local barY = y + h - 50
    
    DrawRoundedRect(barX, barY, barW, barH, GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.8 * alpha)
    
    -- Progress bar with gradient
    local progressW = barW * load.progress
    if progressW > 0 then
        DrawGradientRect(barX, barY, progressW, barH,
                        GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3],
                        GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha)
        -- Reduced glow on progress bar
        DrawGlow(barX, barY, progressW, barH, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.15 * alpha)
    end
    
    -- Loading text (centered) - higher quality with dynamic centering
    local loadText = string.format("Loading... %d%%", math.floor(load.progress * 100))
    local loadTextSize = 18
    local loadTextWidth = Susano.GetTextWidth(loadText, loadTextSize)
    local textX = x + (w / 2) - (loadTextWidth / 2) -- Perfectly centered
    Susano.DrawText(textX, y + h - 28, loadText, loadTextSize, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], alpha * 0.8)
    
    -- Particles effect (smoother orbit)
    for i = 1, 15 do
        local time = elapsed * 1.5
        local angle = (time * 60 + i * 24) * (math.pi / 180)
        local radius = 170 + math.sin(time * 2 + i) * 25
        local px = x + w/2 + math.cos(angle) * radius
        local py = y + h/2 + math.sin(angle) * radius
        local particleAlpha = alpha * (0.4 + math.sin(time * 8 + i) * 0.3)
        
        Susano.DrawCircle(px, py, 5, true, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], particleAlpha, 1, 12)
    end
end

-- ============================================================
-- FEATURE 5: HIT LOG SYSTEM
-- ============================================================

-- Add hit to log
local function AddHit(playerName, bodyPart, damage)
    local hit = {
        player = playerName,
        part = bodyPart,
        damage = damage,
        time = GetGameTimer(),
        fadeTime = 2000, -- 2 seconds
        slideProgress = 0
    }
    
    table.insert(Features.hitlog.hits, 1, hit)
    
    -- Keep only max hits
    while #Features.hitlog.hits > Features.hitlog.maxHits do
        table.remove(Features.hitlog.hits)
    end
end

-- Get body part name
local function GetBodyPartName(boneId)
    local parts = {
        [31086] = "HEAD",
        [24818] = "NECK", 
        [11816] = "PELVIS",
        [24816] = "LEFT ARM",
        [40269] = "RIGHT ARM",
        [58271] = "LEFT LEG",
        [63931] = "RIGHT LEG",
        [24817] = "SPINE",
        [64729] = "LEFT HAND",
        [28422] = "RIGHT HAND",
    }
    return parts[boneId] or "BODY"
end

local function RenderHitLog()
    if not Features.hitlog.enabled then return end
    
    local hl = Features.hitlog
    local baseX = hl.x
    local baseY = hl.y
    local currentTime = GetGameTimer()
    
    if #hl.hits == 0 then return end
    
    -- Render each hit as a slide-in message
    local yOffset = 0
    for i = #hl.hits, 1, -1 do -- Render from bottom to top
        local hit = hl.hits[i]
        local elapsed = currentTime - hit.time
        
        -- Update slide animation
        if hit.slideProgress < 1 then
            hit.slideProgress = math.min(1, hit.slideProgress + 0.15)
        end
        
        -- Calculate fade (fade in first 200ms, fade out last 500ms)
        local alpha = 1
        if elapsed < 200 then
            alpha = elapsed / 200
        elseif elapsed > hit.fadeTime - 500 then
            alpha = (hit.fadeTime - elapsed) / 500
        end
        
        if alpha > 0 then
            -- Slide animation (ease out)
            local slideOffset = (1 - hit.slideProgress) * -150
            local slideEase = 1 - math.pow(1 - hit.slideProgress, 3)
            local xPos = baseX + slideOffset
            local yPos = baseY + yOffset
            
            local w = 250
            local h = 35
            
            -- Background with glow
            DrawGlow(xPos, yPos, w, h, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.4 * alpha * slideEase)
            DrawRoundedRect(xPos, yPos, w, h, GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.9 * alpha * slideEase)
            
            -- Left border accent
            DrawRoundedRect(xPos, yPos, 4, h, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 1 * alpha * slideEase)
            
            -- Hit text
            local hitText = string.format("HIT %s [%s] -%d HP", hit.player, hit.part, hit.damage)
            Susano.DrawText(xPos + 12, yPos + 10, hitText, 14, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], alpha * slideEase)
            
            yOffset = yOffset + h + 5
        else
            -- Remove faded hits
            table.remove(hl.hits, i)
        end
    end
end

-- Track entity health to detect actual hits with real damage
local trackedEntities = {}

Citizen.CreateThread(function()
    while true do
        Citizen.Wait(100) -- Check every 100ms
        
        if Features.hitlog.enabled then
            local playerPed = PlayerPedId()
            local allPeds = GetGamePool('CPed')
            
            for _, entity in ipairs(allPeds) do
                if entity ~= playerPed and DoesEntityExist(entity) then
                    local currentHealth = GetEntityHealth(entity)
                    
                    -- Initialize tracking for new entities
                    if not trackedEntities[entity] then
                        trackedEntities[entity] = currentHealth
                    else
                        local previousHealth = trackedEntities[entity]
                        
                        -- Check if health decreased (damage taken)
                        if currentHealth < previousHealth and currentHealth > 0 then
                            local damage = previousHealth - currentHealth
                            
                            -- Check if WE caused the damage
                            if HasEntityBeenDamagedByEntity(entity, playerPed, 1) then
                                -- Get target name
                                local targetName = "NPC"
                                if IsPedAPlayer(entity) then
                                    local playerId = NetworkGetPlayerIndexFromPed(entity)
                                    if playerId ~= -1 then
                                        targetName = GetPlayerName(playerId)
                                    end
                                end
                                
                                -- Get body part hit
                                local bodyPart = "BODY"
                                local success, boneHit = GetPedLastDamageBone(entity)
                                if success then
                                    bodyPart = GetBodyPartName(boneHit)
                                end
                                
                                -- Add to log with REAL damage
                                AddHit(targetName, bodyPart, math.floor(damage))
                                
                                -- Clear damage flag
                                ClearEntityLastDamageEntity(entity)
                            end
                        end
                        
                        -- Update tracked health
                        trackedEntities[entity] = currentHealth
                    end
                end
            end
            
            -- Clean up dead/removed entities
            for entity, _ in pairs(trackedEntities) do
                if not DoesEntityExist(entity) then
                    trackedEntities[entity] = nil
                end
            end
        end
    end
end)

-- ============================================================
-- FEATURE 6: CUSTOM HUD
-- ============================================================

local function RenderCustomHUD()
    if not Features.hud.enabled then return end
    
    local ped = PlayerPedId()
    if not ped or not DoesEntityExist(ped) then return end
    
    local hud = Features.hud
    local x, y = hud.x, hud.y
    local width = hud.w * hud.scale
    local height = hud.h * hud.scale
    
    -- Drag indicator
    if hud.dragging then
        DrawGlow(x, y, width, height, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.15)
    end
    
    -- Background with reduced glow
    DrawGlow(x, y, width, height, GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], 0.08)
    DrawRoundedRect(x, y, width, height, GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.88)
    
    -- Hit flash effect
    if hud.hitEffect > 0 then
        DrawRoundedRect(x, y, width, height, 1, 0.2, 0.2, hud.hitEffect * 0.3)
    end
    
    -- Health
    local health = GetEntityHealth(ped) - 100
    local maxHealth = GetEntityMaxHealth(ped) - 100
    local healthPercent = math.max(0, math.min(100, (health / maxHealth) * 100))
    
    Susano.DrawText(x + 10, y + 10, "HEALTH", 14, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 1)
    
    -- Simple health bar (no segments)
    local barWidth = width - 20
    local barHeight = 15
    local barY = y + 28
    
    -- Background bar
    DrawRoundedRect(x + 10, barY, barWidth, barHeight, GALAXY.DARK_SPACE[1] * 1.5, GALAXY.DARK_SPACE[2] * 1.5, GALAXY.DARK_SPACE[3] * 1.5, 0.9)
    
    -- Filled health bar with gradient
    local healthBarWidth = barWidth * (healthPercent / 100)
    if healthBarWidth > 0 then
        DrawGradientRect(x + 10, barY, healthBarWidth, barHeight,
                        GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3],
                        GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], 0.9)
        -- Glow on top
        DrawRoundedRect(x + 10, barY, healthBarWidth, 3, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 1)
    end
    
    -- Warning glow when low health
    if healthPercent < 30 then
        local pulse = 0.5 + math.sin(GetGameTimer() / 200) * 0.5
        DrawGlow(x + 10, barY, barWidth, barHeight, 1, 0.1, 0.1, pulse * 0.3)
    end
    
    -- Health percentage text
    local healthText = string.format("%.0f%%", healthPercent)
    local healthTextWidth = Susano.GetTextWidth(healthText, 13)
    Susano.DrawText(x + (width / 2) - (healthTextWidth / 2), barY + 1, healthText, 13, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
    
    -- Armor
    local armor = GetPedArmour(ped)
    Susano.DrawText(x + 10, y + 50, "ARMOR", 14, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 1)
    
    -- Simple armor bar (no segments)
    local armorBarY = y + 68
    
    -- Background bar
    DrawRoundedRect(x + 10, armorBarY, barWidth, barHeight, GALAXY.DARK_SPACE[1] * 1.5, GALAXY.DARK_SPACE[2] * 1.5, GALAXY.DARK_SPACE[3] * 1.5, 0.9)
    
    -- Filled armor bar with gradient
    local armorBarWidth = barWidth * (armor / 100)
    if armorBarWidth > 0 then
        DrawGradientRect(x + 10, armorBarY, armorBarWidth, barHeight,
                        GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3],
                        GALAXY.NEBULA_BLUE[1], GALAXY.NEBULA_BLUE[2], GALAXY.NEBULA_BLUE[3], 0.9)
        -- Glow on top
        DrawRoundedRect(x + 10, armorBarY, armorBarWidth, 3, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 1)
    end
    
    -- Warning glow when low armor
    if armor > 0 and armor < 30 then
        local pulse = 0.5 + math.sin(GetGameTimer() / 250) * 0.5
        DrawGlow(x + 10, armorBarY, barWidth, barHeight, 0.2, 0.6, 1, pulse * 0.3)
    end
    
    -- Armor percentage text
    local armorText = string.format("%d%%", armor)
    local armorTextWidth = Susano.GetTextWidth(armorText, 13)
    Susano.DrawText(x + (width / 2) - (armorTextWidth / 2), armorBarY + 1, armorText, 13, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
end

-- ============================================================
-- FEATURE 7: COMBAT HUD (STATS TRACKER)
-- ============================================================

-- Track combat statistics
Citizen.CreateThread(function()
    while true do
        Citizen.Wait(0)
        
        if Features.combatHud.enabled then
            local ped = PlayerPedId()
            local currentTime = GetGameTimer()
            
            -- Track shots fired
            if IsPedShooting(ped) then
                if currentTime - Features.combatHud.lastShotTime > 100 then
                    Features.combatHud.shotsFired = Features.combatHud.shotsFired + 1
                    Features.combatHud.lastShotTime = currentTime
                    
                    -- Check if we hit something
                    local hasHit, hitEntity = GetEntityPlayerIsFreeAimingAt(PlayerId())
                    if hasHit and IsEntityAPed(hitEntity) then
                        Features.combatHud.shotsHit = Features.combatHud.shotsHit + 1
                    end
                end
            end
            
            -- Clean up old damage entries (keep last 1 second for DPS)
            local damageWindow = Features.combatHud.damageWindow
            for i = #damageWindow, 1, -1 do
                if currentTime - damageWindow[i].time > 1000 then
                    table.remove(damageWindow, i)
                end
            end
        end
    end
end)

-- Hook into hit log to track damage
local originalAddHit = AddHit
function AddHit(playerName, bodyPart, damage)
    -- Track damage for combat HUD
    if Features.combatHud.enabled then
        Features.combatHud.totalDamage = Features.combatHud.totalDamage + damage
        table.insert(Features.combatHud.damageWindow, {damage = damage, time = GetGameTimer()})
    end
    
    -- Call original function
    originalAddHit(playerName, bodyPart, damage)
end

local function RenderCombatHUD()
    if not Features.combatHud.enabled then return end
    
    local ch = Features.combatHud
    local x, y = ch.x, ch.y
    local width = 220
    local height = 110
    
    -- Drag indicator
    if ch.dragging then
        DrawGlow(x, y, width, height, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.15)
    end
    
    -- Background with reduced glow
    DrawGlow(x, y, width, height, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.08)
    DrawRoundedRect(x, y, width, height, GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.92)
    
    -- Top header
    DrawGradientRect(x, y, width, 28, 0.08, 0.12, 0.18, 0.06, 0.10, 0.15, 0.9)
    DrawRoundedRect(x, y + 28, width, 2, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.6)
    
    -- Title
    Susano.DrawCircle(x + 15, y + 14, 8, true, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.3, 1, 16)
    Susano.DrawCircle(x + 15, y + 14, 5, false, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.8, 2, 16)
    Susano.DrawText(x + 28, y + 7, "Combat Stats", 15, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
    
    -- Calculate stats
    local accuracy = ch.shotsFired > 0 and (ch.shotsHit / ch.shotsFired * 100) or 0
    
    -- Calculate DPS (damage in last 1 second)
    local dps = 0
    for _, dmg in ipairs(ch.damageWindow) do
        dps = dps + dmg.damage
    end
    
    local yPos = y + 42
    
    -- Accuracy
    Susano.DrawText(x + 10, yPos, "Accuracy:", 14, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.9)
    Susano.DrawText(x + width - 50, yPos, string.format("%.0f%%", accuracy), 14, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
    
    yPos = yPos + 28
    
    -- DPS
    Susano.DrawText(x + 10, yPos, "Damage/sec:", 14, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.9)
    Susano.DrawText(x + width - 50, yPos, tostring(dps), 14, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
end

-- ============================================================
-- COMBAT FEATURES
-- ============================================================

-- Heal Behind Cover - Heal when in cover
Citizen.CreateThread(function()
    while true do
        Citizen.Wait(1000) -- Check every second
        
        if Features.healCover.enabled then
            local ped = PlayerPedId()
            
            -- Check if player is in cover
            if IsPedInCover(ped, false) then
                local health = GetEntityHealth(ped)
                local maxHealth = GetEntityMaxHealth(ped)
                
                if health < maxHealth then
                    SetEntityHealth(ped, math.min(health + 5, maxHealth))
                end
            end
        end
    end
end)

-- No Combat Stance - Prevents weapon aiming stance
Citizen.CreateThread(function()
    while true do
        Citizen.Wait(0)
        
        if Features.noCombatStance.enabled then
            local ped = PlayerPedId()
            
            -- Disable combat crouch/stance
            SetPedUsingActionMode(ped, false, -1, "DEFAULT_ACTION")
            SetPedStealthMovement(ped, false, "DEFAULT_ACTION")
        end
    end
end)

-- ============================================================
-- ENHANCED HUD TRACKING
-- ============================================================

-- Track health/armor changes for segment animations
Citizen.CreateThread(function()
    while true do
        Citizen.Wait(50)
        
        if Features.hud.enabled then
            local ped = PlayerPedId()
            if ped and DoesEntityExist(ped) then
                local currentHealth = GetEntityHealth(ped)
                local currentArmor = GetPedArmour(ped)
                local hud = Features.hud
                
                -- Detect health loss
                if currentHealth < hud.lastHealth then
                    hud.hitEffect = 1.0 -- Trigger hit flash
                    -- Add cracked segment animation
                    local damageTaken = hud.lastHealth - currentHealth
                    table.insert(hud.healthSegments, {time = GetGameTimer(), damage = damageTaken, fade = 1.0})
                end
                
                -- Detect armor loss
                if currentArmor < hud.lastArmor then
                    hud.hitEffect = 1.0
                    local damageTaken = hud.lastArmor - currentArmor
                    table.insert(hud.armorSegments, {time = GetGameTimer(), damage = damageTaken, fade = 1.0})
                end
                
                -- Fade out hit effect
                if hud.hitEffect > 0 then
                    hud.hitEffect = math.max(0, hud.hitEffect - 0.1)
                end
                
                -- Update tracked values
                hud.lastHealth = currentHealth
                hud.lastArmor = currentArmor
                
                -- Clean up old segments
                for i = #hud.healthSegments, 1, -1 do
                    hud.healthSegments[i].fade = hud.healthSegments[i].fade - 0.02
                    if hud.healthSegments[i].fade <= 0 then
                        table.remove(hud.healthSegments, i)
                    end
                end
                
                for i = #hud.armorSegments, 1, -1 do
                    hud.armorSegments[i].fade = hud.armorSegments[i].fade - 0.02
                    if hud.armorSegments[i].fade <= 0 then
                        table.remove(hud.armorSegments, i)
                    end
                end
            end
        end
    end
end)

-- ============================================================
-- CRITICAL HIT PARTICLE BURST EFFECT
-- ============================================================

local function RenderCriticalHitEffect()
    local hud = Features.hud
    local crit = hud.critEffect
    
    if not crit.active then return end
    
    local currentTime = GetGameTimer()
    local elapsed = (currentTime - crit.time) / 1000
    
    -- Effect lasts 1 second
    if elapsed > 1.0 then
        crit.active = false
        return
    end
    
    -- ✨ ENHANCED: Controller Vibration
    if Features.critAlerts.enabled and Features.critAlerts.vibration and elapsed < 0.3 then
        -- Pulse vibration effect
        local vibIntensity = math.floor((1 - elapsed / 0.3) * 255)
        SetControlNormal(0, 27, vibIntensity / 255) -- Trigger vibration
    end
    
    -- Update and render particles
    for i = #crit.particles, 1, -1 do
        local p = crit.particles[i]
        
        -- Update particle position
        p.x = p.x + p.vx
        p.y = p.y + p.vy
        p.vy = p.vy + 0.5 -- Gravity
        p.life = p.life - 0.05
        
        if p.life > 0 then
            local particleX = crit.x + p.x
            local particleY = crit.y + p.y
            
            -- Draw particle with fade
            local alpha = p.life
            Susano.DrawCircle(particleX, particleY, p.size, true, 1, 0.8, 0.2, alpha, 1, 12)
            -- Inner glow
            Susano.DrawCircle(particleX, particleY, p.size - 1, true, 1, 1, 0.5, alpha * 0.8, 1, 10)
        else
            table.remove(crit.particles, i)
        end
    end
    
    -- ✨ ENHANCED: Screen Flash Effect
    if Features.critAlerts.enabled and elapsed < 0.2 then
        local flashAlpha = (1 - elapsed / 0.2) * 0.4 -- Increased intensity
        DrawRoundedRect(0, 0, CONFIG.SCREEN_W, CONFIG.SCREEN_H, 1, 0.9, 0.3, flashAlpha)
        
        -- ✨ Screen Shake Effect
        if Features.critAlerts.screenShake then
            local shakeIntensity = (1 - elapsed / 0.2) * 5
            GameplayCamShake("SMALL_EXPLOSION_SHAKE", shakeIntensity)
        end
    end
    
    -- "CRITICAL!" text with enhanced effects
    if elapsed < 0.8 then
        local textAlpha = elapsed < 0.1 and (elapsed / 0.1) or (elapsed < 0.6 and 1.0 or (1 - (elapsed - 0.6) / 0.2))
        local textScale = 1.0 + (1 - elapsed) * 0.5
        local textY = crit.y - 50 - (elapsed * 30)
        
        -- Multi-layer shadow for depth
        Susano.DrawText(crit.x - 60 + 3, textY + 3, "CRITICAL!", math.floor(24 * textScale), 0, 0, 0, textAlpha * 0.9)
        Susano.DrawText(crit.x - 60 + 2, textY + 2, "CRITICAL!", math.floor(24 * textScale), 0, 0, 0, textAlpha * 0.7)
        -- Main text
        Susano.DrawText(crit.x - 60, textY, "CRITICAL!", math.floor(24 * textScale), 1, 0.8, 0.2, textAlpha)
        -- Enhanced glow
        Susano.DrawText(crit.x - 61, textY - 1, "CRITICAL!", math.floor(24 * textScale), 1, 1, 0.5, textAlpha * 0.6)
        Susano.DrawText(crit.x - 62, textY - 2, "CRITICAL!", math.floor(24 * textScale), 1, 1, 0.8, textAlpha * 0.3)
    end
end

-- ============================================================
-- NEW FEATURE: 3D ESP (Using WorldToScreen API)
-- ============================================================

local function Render3DESP()
    -- Temporarily disabled due to crashes
    -- TODO: Fix WorldToScreen API compatibility
    return
end

-- ============================================================
-- NEW FEATURE: WATERMARK
-- ============================================================

local function RenderWatermark()
    if not Features.watermark.enabled then return end
    
    local wm = Features.watermark
    local x = wm.x
    local y = wm.y
    local time = GetGameTimer() / 1000
    
    -- Drag indicator
    if wm.dragging then
        DrawGlow(x, y, 170, 28, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.15)
    end
    
    -- Animated background
    local pulse = 0.7 + math.sin(time * 2) * 0.3
    DrawRoundedRect(x, y, 170, 28, GALAXY.DARK_SPACE[1], GALAXY.DARK_SPACE[2], GALAXY.DARK_SPACE[3], 0.85)
    DrawGlow(x, y, 170, 28, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], pulse * 0.1)
    
    -- Border with gradient effect
    local borderGlow = 0.5 + math.sin(time * 3) * 0.3
    DrawOutlineRect(x, y, 170, 28, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], borderGlow, 2)
    
    -- Icon
    Susano.DrawCircle(x + 15, y + 14, 8, true, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.4, 1, 16)
    Susano.DrawCircle(x + 15, y + 14, 5, false, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.9, 2, 16)
    
    -- Text with font
    if CustomFonts.loaded and CustomFonts.title then
        Susano.PushFont(CustomFonts.title)
    end
    
    Susano.DrawText(x + 30, y + 6, "VOID ENGINE", 14, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 1)
    Susano.DrawText(x + 30, y + 18, "v1.2 Beta", 10, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.8)
    
    if CustomFonts.loaded and CustomFonts.title then
        Susano.PopFont()
    end
end

-- ============================================================
-- NEW FEATURE: FPS BOOSTER
-- ============================================================

-- FPS Booster runs as a background optimization
Citizen.CreateThread(function()
    while true do
        if Features.fpsBoost.enabled then
            -- Reduce draw distance for better FPS
            SetTimecycleModifier("cinema")
            
            -- Disable some expensive effects
            SetArtificialLightsState(true)
            
            Citizen.Wait(100)
        else
            -- Reset to normal
            ClearTimecycleModifier()
            SetArtificialLightsState(false)
            Citizen.Wait(1000)
        end
    end
end)

-- ============================================================
-- NEW FEATURE: RAINBOW MODE (Color Cycle Effect)
-- ============================================================

local function HSVtoRGB(h, s, v)
    local c = v * s
    local x = c * (1 - math.abs((h / 60) % 2 - 1))
    local m = v - c
    
    local r, g, b = 0, 0, 0
    if h < 60 then
        r, g, b = c, x, 0
    elseif h < 120 then
        r, g, b = x, c, 0
    elseif h < 180 then
        r, g, b = 0, c, x
    elseif h < 240 then
        r, g, b = 0, x, c
    elseif h < 300 then
        r, g, b = x, 0, c
    else
        r, g, b = c, 0, x
    end
    
    return r + m, g + m, b + m
end

-- Apply rainbow colors to galaxy palette
Citizen.CreateThread(function()
    while true do
        if Features.rainbowMode.enabled then
            Features.rainbowMode.hue = (Features.rainbowMode.hue + 2) % 360
            local r, g, b = HSVtoRGB(Features.rainbowMode.hue, 0.8, 1.0)
            GALAXY.COSMIC_PINK = {r, g, b}
            GALAXY.CYAN_GLOW = {HSVtoRGB((Features.rainbowMode.hue + 180) % 360, 0.8, 1.0)}
            Citizen.Wait(16) -- ~60 FPS color update
        else
            -- Reset to original colors
            GALAXY.COSMIC_PINK = {0.95, 0.3, 0.7}
            GALAXY.CYAN_GLOW = {0.2, 0.8, 1}
            Citizen.Wait(100)
        end
    end
end)

-- ============================================================
-- MAIN INPUT HANDLER
-- ============================================================

local function ProcessGlobalInput()
    -- Menu toggle
    local menuDown, menuPressed = Susano.GetAsyncKeyState(CONFIG.MENU_KEY)
    if menuPressed and not InputStates.lastMenuKey then
        Features.menu.enabled = not Features.menu.enabled
    end
    InputStates.lastMenuKey = menuPressed
    
    -- Performance toggle (skip if NO KEYBIND)
    if CONFIG.PERF_KEY ~= 0x13 then
        local perfDown, perfPressed = Susano.GetAsyncKeyState(CONFIG.PERF_KEY)
        if perfPressed and not InputStates.lastPerfKey then
            Features.performance.enabled = not Features.performance.enabled
        end
        InputStates.lastPerfKey = perfPressed
    end
    
    -- Keys toggle (skip if NO KEYBIND)
    if CONFIG.KEYS_KEY ~= 0x13 then
        local keysDown, keysPressed = Susano.GetAsyncKeyState(CONFIG.KEYS_KEY)
        if keysPressed and not InputStates.lastKeysKey then
            Features.keyvis.enabled = not Features.keyvis.enabled
        end
        InputStates.lastKeysKey = keysPressed
    end
    
    -- HUD toggle (skip if NO KEYBIND)
    if CONFIG.HUD_KEY ~= 0x13 then
        local hudDown, hudPressed = Susano.GetAsyncKeyState(CONFIG.HUD_KEY)
        if hudPressed and not InputStates.lastHudKey then
            Features.hud.enabled = not Features.hud.enabled
        end
        InputStates.lastHudKey = hudPressed
    end
    
    -- Heal Behind Cover toggle
    local healCoverDown, healCoverPressed = Susano.GetAsyncKeyState(CONFIG.HEAL_COVER_KEY)
    if healCoverPressed and not InputStates.lastHealCoverKey then
        Features.healCover.enabled = not Features.healCover.enabled
    end
    InputStates.lastHealCoverKey = healCoverPressed
    
    -- Edit mode toggle
    local editDown, editPressed = Susano.GetAsyncKeyState(CONFIG.EDIT_MODE_KEY)
    if editPressed and not InputStates.lastEditModeKey then
        Features.editMode.enabled = not Features.editMode.enabled
    end
    InputStates.lastEditModeKey = editPressed
    
    -- Hit Log toggle
    local hitlogDown, hitlogPressed = Susano.GetAsyncKeyState(CONFIG.HITLOG_KEY)
    if hitlogPressed and not InputStates.lastHitLogKey then
        Features.hitlog.enabled = not Features.hitlog.enabled
    end
    InputStates.lastHitLogKey = hitlogPressed
    
    -- No Combat Stance toggle
    local combatStanceDown, combatStancePressed = Susano.GetAsyncKeyState(CONFIG.COMBAT_STANCE_KEY)
    if combatStancePressed and not InputStates.lastCombatStanceKey then
        Features.noCombatStance.enabled = not Features.noCombatStance.enabled
    end
    InputStates.lastCombatStanceKey = combatStancePressed
    
    -- Combat HUD toggle
    local combatHudDown, combatHudPressed = Susano.GetAsyncKeyState(CONFIG.COMBAT_HUD_KEY)
    if combatHudPressed and not InputStates.lastCombatHudKey then
        Features.combatHud.enabled = not Features.combatHud.enabled
    end
    InputStates.lastCombatHudKey = combatHudPressed
    
    -- Watermark toggle
    local watermarkDown, watermarkPressed = Susano.GetAsyncKeyState(CONFIG.WATERMARK_KEY)
    if watermarkPressed and not InputStates.lastWatermarkKey then
        Features.watermark.enabled = not Features.watermark.enabled
    end
    InputStates.lastWatermarkKey = watermarkPressed
    
    -- FPS Booster toggle
    local fpsBoostDown, fpsBoostPressed = Susano.GetAsyncKeyState(CONFIG.FPS_BOOST_KEY)
    if fpsBoostPressed and not InputStates.lastFpsBoostKey then
        Features.fpsBoost.enabled = not Features.fpsBoost.enabled
    end
    InputStates.lastFpsBoostKey = fpsBoostPressed
    
    -- 3D ESP toggle
    local espDown, espPressed = Susano.GetAsyncKeyState(CONFIG.ESP_KEY)
    if espPressed and not InputStates.lastEspKey then
        Features.esp.enabled = not Features.esp.enabled
    end
    InputStates.lastEspKey = espPressed
    
    -- Rainbow Mode toggle
    local rainbowDown, rainbowPressed = Susano.GetAsyncKeyState(CONFIG.RAINBOW_KEY)
    if rainbowPressed and not InputStates.lastRainbowKey then
        Features.rainbowMode.enabled = not Features.rainbowMode.enabled
    end
    InputStates.lastRainbowKey = rainbowPressed
end

-- Render edit mode overlay
local function RenderEditModeOverlay()
    if not Features.editMode.enabled then return end
    
    local screenW = CONFIG.SCREEN_W
    local screenH = CONFIG.SCREEN_H
        
    -- Darker grey overlay (0.6 opacity)
    DrawRoundedRect(0, 0, screenW, screenH, 0.2, 0.2, 0.2, 0.6)
    
    -- Center coordinates
    local centerX = screenW / 2
    local centerY = screenH / 2
    
    -- Title - 32px for SHARP, crisp rendering with precise centering
    local titleText = "EDIT MODE ACTIVE"
    local titleSize = 32
    local titleWidth = Susano.GetTextWidth(titleText, titleSize)
    Susano.DrawText(centerX - (titleWidth / 2), centerY - 80, titleText, titleSize, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 1)
    
    -- All lines perfectly centered using GetTextWidth for pixel-perfect alignment
    local line1Text = "Click and drag any UI element to move it"
    local line1Size = 18
    local line1Width = Susano.GetTextWidth(line1Text, line1Size)
    Susano.DrawText(centerX - (line1Width / 2), centerY - 15, line1Text, line1Size, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 1)
    
    local line2Text = "Press F12 to exit Edit Mode"
    local line2Size = 18
    local line2Width = Susano.GetTextWidth(line2Text, line2Size)
    Susano.DrawText(centerX - (line2Width / 2), centerY + 20, line2Text, line2Size, GALAXY.STAR_WHITE[1], GALAXY.STAR_WHITE[2], GALAXY.STAR_WHITE[3], 0.9)
    
    local line3Text = "Game controls disabled"
    local line3Size = 16
    local line3Width = Susano.GetTextWidth(line3Text, line3Size)
    Susano.DrawText(centerX - (line3Width / 2), centerY + 55, line3Text, line3Size, GALAXY.MAGENTA[1], GALAXY.MAGENTA[2], GALAXY.MAGENTA[3], 0.8)

    -- Mouse cursor indicator
    local mx, my = InputStates.mouseX, InputStates.mouseY
    Susano.DrawCircle(mx, my, 8, true, GALAXY.COSMIC_PINK[1], GALAXY.COSMIC_PINK[2], GALAXY.COSMIC_PINK[3], 0.9, 2, 16)
    Susano.DrawCircle(mx, my, 12, false, GALAXY.CYAN_GLOW[1], GALAXY.CYAN_GLOW[2], GALAXY.CYAN_GLOW[3], 0.7, 2, 16)
end

-- Block game inputs during edit mode or when menu is open
Citizen.CreateThread(function()
    while true do
        Citizen.Wait(0)
        
        if Features.editMode.enabled or Features.menu.enabled then
            -- Disable ALL player actions
            DisableAllControlActions(0) -- Disable everything
            
            -- Re-enable only what we need for the menu
            EnableControlAction(0, 249, true) -- Push to talk (if needed)
            EnableControlAction(0, 1, true) -- Camera LR
            EnableControlAction(0, 2, true) -- Camera UD
            EnableControlAction(0, 239, true) -- Cursor X
            EnableControlAction(0, 240, true) -- Cursor Y
        end
    end
end)

-- ============================================================
-- MAIN RENDER LOOP
-- ============================================================

-- Check if Susano API exists
if not Susano then
    print("^1ERROR: Susano API not found! Make sure you're using a compatible executor.^0")
    return
end

print("^2[VOID ENGINE] Starting initialization...^0")

-- Initialize font system (optional - loads Windows fonts)
local function InitializeFonts()
    -- Try to load custom fonts from Windows directory
    local success1, font1 = pcall(function()
        return Susano.LoadFont("C:\\\\Windows\\\\Fonts\\\\segoeui.ttf")
    end)
    
    local success2, font2 = pcall(function()
        return Susano.LoadFont("C:\\\\Windows\\\\Fonts\\\\seguibl.ttf")
    end)
    
    if success1 and font1 then
        CustomFonts.main = font1
        CustomFonts.loaded = true
        print("^2[VOID ENGINE] Custom font loaded (Segoe UI)^0")
    end
    
    if success2 and font2 then
        CustomFonts.title = font2
        print("^2[VOID ENGINE] Title font loaded (Segoe UI Bold)^0")
    end
end

Citizen.CreateThread(function()
    -- Wait a frame before initializing
    Citizen.Wait(100)
    
    -- Initialize fonts
    InitializeFonts()
    
    -- Initialize loading animation
    Features.loading.startTime = GetGameTimer()
    Features.loading.active = true
    
    print("^5========================================^0")
    print("^5|  ^6*** VOID ENGINE LOADED! ***^5       |^0")
    print("^5========================================^0")
    print("^6|   F5: Menu    ^3|^6 F2: Rainbow Mode   ^5|^0")
    print("^6|   F8: 3D ESP  ^3|^6 F4: FPS Boost      ^5|^0")
    print("^6|   F12: Edit Mode (Drag UI)           ^5|^0")
    print("^5========================================^0")
    print("^3*** Theme: Cosmic Galaxy Edition^0")
    print("^2*** 12 Features - Stable & Optimized!^0")
    print("^2*** Hit Logger Fixed - Enable in Menu!^0")
    print("^2[VOID ENGINE] Main loop starting...^0")
    
    while true do
        local success, err = pcall(function()
            -- Update
            UpdatePerformance()
            UpdateMousePosition()
            ProcessGlobalInput()
            ProcessDragAndDrop() -- Process drag BEFORE menu input
            ProcessMenuInput()
            
            -- Render all features
            Susano.BeginFrame()
            
            -- Render 3D ESP (uses WorldToScreen API)
            Render3DESP()
            
            -- Render UI elements (except loading)
            RenderPerformance()
            RenderKeyVisualizer()
            RenderCustomHUD()
            RenderCombatHUD()
            RenderHitLog()
            RenderWatermark() -- Branding watermark
            RenderMenu() -- Menu renders last (on top)
            
            -- Edit mode overlay (renders above UI)
            RenderEditModeOverlay()
            
            -- Loading animation renders on top of everything
            RenderLoadingAnimation()
            
            Susano.SubmitFrame()
        end)
        
        if not success then
            print("^1[VOID ENGINE ERROR] " .. tostring(err) .. "^0")
            Citizen.Wait(1000) -- Wait a second before retrying
        end
        
        Citizen.Wait(0)
    end
end)

print("^2[VOID ENGINE] Initialization complete!^0")
