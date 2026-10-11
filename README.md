--// Hanse On Top V7 — Dual Parry + Full Feature
--// (patch: stamina ID-agnostic, listener atribut ringan, meter jaringan. Auto parry TIDAK diubah)

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local VIM = game:GetService("VirtualInputManager")
local RS = game:GetService("ReplicatedStorage")
local Stats = game:GetService("Stats")
local HttpService = game:GetService("HttpService")
local Camera = workspace.CurrentCamera
local LP = Players.LocalPlayer

local env = (getgenv and getgenv()) or _G
if env.HanseUnload then pcall(env.HanseUnload) env.HanseUnload = nil end

local alive = true
local Conns = {}
local function keep(c) Conns[#Conns + 1] = c return c end

-- ==========================================
-- CONFIG
-- ==========================================
local Cfg = {
    -- defense
    parry = false, parryM1 = true, parryM2 = true, parryDist = 14,
    pingComp = true, respectCd = true, lead = 0, parryOffset = 0, jitter = 8, perfectRate = 100, autoTune = true,
    debug = false,
    -- dual parry
    parryMode = "Dual",
    remoteOffsetMs = -8,
    -- visual
    esp = false, espSelf = true, espHp = true, espDisplay = false, espRadius = 360,
    styleTag = true, bothSlots = true, m2Tag = true,
    pingHud = true, staminaHud = true,
    -- cam
    camLock = false,
}

local KEY = { Block = Enum.KeyCode.F, M2 = Enum.KeyCode.R, Equip = Enum.KeyCode.T }
local CAL_FILE = "hanse_timing_gakuran_v7.json"
local MAX_RADIUS = 1000
local DOWN_HP = 5

local function logLine(str)
    if Cfg.debug then print("[Hanse] " .. str) end
end

-- ==========================================
-- UTIL + CALIBRATION
-- ==========================================
local function median(t)
    local n = #t
    if n == 0 then return nil end
    local c = table.clone(t)
    table.sort(c)
    if n % 2 == 1 then return c[(n + 1) / 2] end
    return (c[n / 2] + c[n / 2 + 1]) / 2
end

local function trim(s) return (s:gsub("^%s+", ""):gsub("%s+$", "")) end

local Cal = { impact = {}, pf = 0.75 }
do
    if isfile and readfile then
        local ok, data = pcall(function()
            if isfile(CAL_FILE) then return HttpService:JSONDecode(readfile(CAL_FILE)) end
        end)
        if ok and type(data) == "table" then
            Cal.impact = type(data.impact) == "table" and data.impact or {}
            if type(data.pf) == "number" then Cal.pf = math.clamp(data.pf, 0.3, 1.3) end
        end
    end
end

local saveQueued = false
local function saveCal()
    if saveQueued or not writefile then return end
    saveQueued = true
    task.delay(2, function()
        saveQueued = false
        pcall(function() writefile(CAL_FILE, HttpService:JSONEncode(Cal)) end)
    end)
end

local function addImpact(id, animTime)
    if animTime < 0.04 or animTime > 1.5 then return 0 end
    local s = Cal.impact[id]
    if not s then s = {} Cal.impact[id] = s end
    s[#s + 1] = animTime
    if #s > 15 then table.remove(s, 1) end
    saveCal()
    return #s
end

local function getImpact(id)
    local s = Cal.impact[id]
    if s and #s >= 2 then return median(s) end
    return nil
end

local St = {
    ping = 0.05, fps = 60,
    lastMyAtk = 0, lastMyIdx = 0, lastMyM2 = 0, lastR = 0,
    m1Pause = 0, pf = Cal.pf,
}

-- ==========================================
-- HUD (Ping/FPS + Stamina)
-- ==========================================
for _, n in ipairs({ "ESPHolder_Rayfield", "HanseHUD_Rayfield", "HanseESP", "HanseHUD" }) do
    local o = CoreGui:FindFirstChild(n)
    if o then o:Destroy() end
end

local EspFolder = Instance.new("ScreenGui")
EspFolder.Name = "HanseESP"
EspFolder.ResetOnSpawn = false
EspFolder.Parent = CoreGui

local Hud = Instance.new("ScreenGui")
Hud.Name = "HanseHUD"
Hud.ResetOnSpawn = false
Hud.IgnoreGuiInset = true
Hud.Parent = CoreGui

local UI = {}
local updateStamina
do
    local pl = Instance.new("TextLabel")
    pl.BackgroundTransparency = 1
    pl.Font = Enum.Font.GothamBold
    pl.TextStrokeTransparency = 0.3
    pl.TextColor3 = Color3.new(1, 1, 1)
    pl.AnchorPoint = Vector2.new(1, 0)
    pl.Position = UDim2.new(1, -10, 0, 6)
    pl.Size = UDim2.fromOffset(300, 16)
    pl.TextSize = 12
    pl.TextXAlignment = Enum.TextXAlignment.Right
    pl.Visible = Cfg.pingHud
    pl.Parent = Hud
    UI.ping = pl

    local sf = Instance.new("Frame")
    sf.AnchorPoint = Vector2.new(0.5, 1)
    sf.Position = UDim2.new(0.5, 0, 0.95, 0)
    sf.Size = UDim2.fromOffset(200, 10)
    sf.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
    sf.BorderSizePixel = 1
    sf.BorderColor3 = Color3.new(0, 0, 0)
    sf.Visible = false
    sf.Parent = Hud
    UI.stamina = sf

    local fill = Instance.new("Frame")
    fill.Size = UDim2.fromScale(1, 1)
    fill.BackgroundColor3 = Color3.fromRGB(80, 170, 255)
    fill.BorderSizePixel = 0
    fill.Parent = sf

    local txt = Instance.new("TextLabel")
    txt.BackgroundTransparency = 1
    txt.Position = UDim2.new(0, 0, 0, -16)
    txt.Size = UDim2.new(1, 0, 0, 14)
    txt.Font = Enum.Font.GothamBold
    txt.TextSize = 11
    txt.TextColor3 = Color3.new(1, 1, 1)
    txt.TextStrokeTransparency = 0
    txt.Parent = sf

    local val, mx = nil, 100
    updateStamina = function(v)
        val = v
        if v > mx then mx = v end
        fill.Size = UDim2.fromScale(math.clamp(v / mx, 0, 1), 1)
        txt.Text = "Stamina: " .. tostring(v)
        sf.Visible = Cfg.staminaHud
    end
    UI.stamHas = function() return val ~= nil end
end

-- ==========================================
-- MODUL: CombatConfig
-- ==========================================
local G = { CC = nil, awake = nil, remote = nil, remoteOK = false }

task.spawn(function()
    local ok, cc = pcall(function()
        local sh = RS:WaitForChild("Shared", 10)
        local cfg = sh:WaitForChild("Config", 10)
        return require(cfg:WaitForChild("CombatConfig", 10))
    end)
    if ok and type(cc) == "table" then G.CC = cc else logLine("CombatConfig gagal") end
end)

task.spawn(function()
    local ok, ac = pcall(function()
        local sh = RS:WaitForChild("Shared", 10)
        local cfg = sh:WaitForChild("Config", 10)
        return require(cfg:WaitForChild("AwakeningConfig", 10))
    end)
    if ok and type(ac) == "table" then G.awake = ac end
end)

task.spawn(function()
    local ok, r = pcall(function() return RS:WaitForChild("Remotes", 15) end)
    if not (ok and r) then return end
    local sr = r:FindFirstChild("Server") or r:WaitForChild("Server", 8)
    if sr then
        G.remote = sr
        G.remoteOK = true
        print("[Hanse] Remote Block OK - Dual mode ready")
    end
end)

function G.windowBonus()
    local aw = G.awake
    local c = LP.Character
    if not (aw and c) then return 0 end
    local s = c:GetAttribute(aw.CHARACTER_AWAKENED_ATTR or "AwakenedStyle")
    if type(s) ~= "string" or s == "" then return 0 end
    local st = aw.Styles and aw.Styles[s:lower()]
    local b = st and st.Buff and st.Buff.PerfectBlockWindowBonus
    return (type(b) == "number" and b > 0) and b or 0
end

function G.perfectWindow()
    local cc = G.CC
    local w = cc and cc.Block and cc.Block.PerfectBlockWindow
    if not (type(w) == "number" and w > 0) then w = 0.125 end
    return w + G.windowBonus()
end

function G.blockCd()
    local cc = G.CC
    local c = cc and cc.Block and cc.Block.CooldownSeconds
    if type(c) == "number" and c >= 0 then return c end
    return 0.5
end

local STYLE_ALIAS = { hakariother = "hakari" }
local function styleKey(s) s = tostring(s or ""):lower() return STYLE_ALIAS[s] or s end
local KeyCache = {}
local function resolveKey(style)
    local k = styleKey(style)
    local c = KeyCache[k]
    if c then return c[1], c[2] end
    local cc = G.CC
    local st = cc and cc.Styles
    if type(st) ~= "table" then return k, false end
    local key, known = k, false
    if st[k] ~= nil then known = true
    else
        local bestLen = 0
        for ck in pairs(st) do
            if type(ck) == "string" and ck ~= "default" then
                local n = math.min(#ck, #k)
                if n >= 4 and n > bestLen and ck:sub(1, n) == k:sub(1, n) then
                    key, known, bestLen = ck, true, n
                end
            end
        end
    end
    KeyCache[k] = { key, known }
    return key, known
end

function G.impact(style, info, speed)
    local cc = G.CC
    if not cc then return nil end
    local key, known = resolveKey(style)
    local ok, v = pcall(function()
        if info.kind == "M1" then
            return cc.GetScaledStyleM1HitboxDelay(key, info.index or 1, speed)
        elseif info.kind == "M2" then
            local variant = info.name:lower():find("right", 1, true) and "Right" or "Left"
            return cc.GetScaledHitboxDelay(cc.GetStyleM2HitboxDelay(key, false, variant), speed)
        end
    end)
    if ok and type(v) == "number" and v > 0 then return v, known end
    return nil
end

function G.m2Cd(style)
    local cc = G.CC
    if cc then
        local ok, v = pcall(cc.GetStyleM2Cooldown, (resolveKey(style)))
        if ok and type(v) == "number" then return v end
    end
    return 7
end

function G.multiHit(style)
    local cc = G.CC
    if not cc then return nil end
    local ok, v = pcall(cc.GetStyleM2MultiHitDelays, (resolveKey(style)))
    if ok and type(v) == "table" and #v >= 2 then return v end
    return nil
end

-- ==========================================
-- STYLE
-- ==========================================
local PlayerStyle, PlayerSlots, StyleFromAttr = {}, {}, {}
local refreshEsp = function(_) end
local M2Used = {}

local function getStyleInfo(plr)
    local c = plr.Character
    local pd = c and c:FindFirstChild("PlayerData")
    local active
    if pd then
        local s = pd:GetAttribute("CombatStyleOverride")
        if typeof(s) ~= "string" or s == "" then s = pd:GetAttribute("CombatStyle") end
        if typeof(s) == "string" then
            s = trim(s)
            active = (s ~= "") and s or "Basic"
        end
    end
    local list = { plr }
    if c then list[#list + 1] = c end
    if pd then list[#list + 1] = pd end
    local s1, s2, slot
    for _, inst in ipairs(list) do
        local a1 = inst:GetAttribute("CombatStyleSlot1")
        local a2 = inst:GetAttribute("CombatStyleSlot2")
        local eq = inst:GetAttribute("CombatStyleEquippedSlot")
        if a1 ~= nil or a2 ~= nil or eq ~= nil then
            s1 = type(a1) == "string" and a1 or ""
            s2 = type(a2) == "string" and a2 or ""
            slot = (tonumber(eq) == 2) and 2 or 1
            break
        end
    end
    if not active and slot then
        active = (slot == 2) and s2 or s1
        if active == "" then active = "Basic" end
    end
    return active, s1, s2, slot
end

local function updateStyleFromAttrs(plr)
    local active, s1, s2, slot = getStyleInfo(plr)
    if not active then return end
    StyleFromAttr[plr] = true
    PlayerStyle[plr] = active
    PlayerSlots[plr] = slot and { s1 = s1, s2 = s2, slot = slot } or nil
    refreshEsp(plr)
end

-- ==========================================
-- ANIM DATABASE
-- ==========================================
local Anim, animCount = {}, 0

local function classify(name)
    local n = name:lower()
    if n:find("parry", 1, true) or n:find("perfect", 1, true) then return "Parry" end
    if n:find("guardbreak", 1, true) then return "GuardBreakHit" end
    if n:find("blockhit", 1, true) then return "BlockHit" end
    if n:match("ehit$") then return "EHit" end
    if n:match("^m2") or n:match("m2$") then return "M2" end
    if n:match("m1$") or (n:match("^m1") and not n:find("hit", 1, true)) then return "M1" end
    return nil
end

local function addAnim(inst, styleName, forceKind)
    local id = tostring(inst.AnimationId):match("%d+")
    if not id then return end
    local kind = forceKind or classify(inst.Name)
    if not kind then return end
    local key = styleName:lower()
    local e = Anim[id]
    if e then
        if e.key ~= key then
            e.shared = true
            e.styles[key] = true
        end
        return
    end
    Anim[id] = {
        style = styleName, key = key, kind = kind, name = inst.Name,
        index = tonumber(inst.Name:match("^(%d)")), styles = { [key] = true },
    }
    animCount += 1
end

local function scanContainer(container, styleName, forceKind)
    for _, d in ipairs(container:GetDescendants()) do
        if d:IsA("Animation") then addAnim(d, styleName, forceKind) end
    end
end

task.spawn(function()
    for _, rootName in ipairs({ "Animations", "AnimationsLEGACY" }) do
        local root = RS:WaitForChild(rootName, 8)
        if root then
            local combat = root:FindFirstChild("Combat")
            if combat then
                for _, child in ipairs(combat:GetChildren()) do
                    if child.Name ~= "Dodges" and child.Name ~= "Grappling" then
                        if child:IsA("Animation") then addAnim(child, "Base")
                        else scanContainer(child, (child.Name:gsub("Anims$", ""))) end
                    end
                end
                keep(combat.DescendantAdded:Connect(function(d)
                    if d:IsA("Animation") then
                        local top = d
                        while top.Parent and top.Parent ~= combat do top = top.Parent end
                        if top.Name ~= "Dodges" and top.Name ~= "Grappling" then
                            addAnim(d, (top.Name:gsub("Anims$", "")))
                        end
                    end
                end))
            end
            local base = root:FindFirstChild("BaseCombat")
            if base then scanContainer(base, "Base") end
            local pb = root:FindFirstChild("PerfectBlockAnims")
            if pb then scanContainer(pb, "Base", "Parry") end
        end
    end
    logLine(("Database animasi: %d"):format(animCount))
end)

-- ==========================================
-- INPUT VIM (V5.2 ASLI)
-- ==========================================
local Key = { down = false, lastDown = 0, lastUp = 0, releaseAt = 0, coverUntil = 0 }

local function keySend(key, state)
    pcall(VIM.SendKeyEvent, VIM, state, key, false, game)
end

function Key.press(hold, coverUntil)
    if Key.down then return false end
    local now = os.clock()
    keySend(KEY.Block, true)
    Key.down = true
    Key.lastDown = now
    Key.releaseAt = now + hold
    Key.coverUntil = coverUntil
    return true
end

function Key.release()
    if not Key.down then return end
    keySend(KEY.Block, false)
    Key.down = false
    Key.lastUp = os.clock()
end

local function tapKey(key)
    task.spawn(function()
        keySend(key, true)
        task.wait(0.06)
        keySend(key, false)
    end)
end

-- ==========================================
-- REMOTE BLOCK
-- ==========================================
local RemoteBlock = { down = false, lastDown = 0, lastUp = 0, releaseAt = 0, coverUntil = 0, fires = 0, fails = 0 }

local function getServerTime()
    if workspace.GetServerTimeNow then
        local ok, t = pcall(function() return workspace:GetServerTimeNow() end)
        if ok and type(t) == "number" then return t end
    end
    return os.clock()
end

local function remoteAct()
    if not G.remote then RemoteBlock.fails += 1 return false end
    local ok = pcall(G.remote.FireServer, G.remote,
        { Type = "Combat", Action = "Block", Func = "Activated" },
        getServerTime())
    if ok then RemoteBlock.fires += 1 else RemoteBlock.fails += 1 end
    return ok
end

local function remoteDeact()
    if not G.remote then return false end
    local ok = pcall(G.remote.FireServer, G.remote,
        { Type = "Combat", Action = "Block", Func = "Deactivated" })
    if ok then RemoteBlock.fires += 1 end
    return ok
end

function RemoteBlock.press(hold, coverUntil)
    if RemoteBlock.down then return false end
    if not remoteAct() then return false end
    local now = os.clock()
    RemoteBlock.down = true
    RemoteBlock.lastDown = now
    RemoteBlock.releaseAt = now + hold
    RemoteBlock.coverUntil = coverUntil
    return true
end

function RemoteBlock.release()
    if not RemoteBlock.down then return end
    remoteDeact()
    RemoteBlock.down = false
    RemoteBlock.lastUp = os.clock()
end

function RemoteBlock.forceRelease()
    if RemoteBlock.down then remoteDeact() RemoteBlock.down = false end
end

local function useVIM()
    return Cfg.parryMode == "Dual" or Cfg.parryMode == "VIM"
end

local function useRemote()
    local mode = Cfg.parryMode
    if mode == "Remote" then return true end
    if mode == "Dual" then return G.remoteOK end
    return false
end

-- ==========================================
-- STATE
-- ==========================================
local LastAtk, Pending, Threats, GuardBroken, CounterUntil = {}, {}, {}, {}, {}
local ParryCtx = nil

local function meBusy()
    local c = LP.Character
    if not c then return true end
    return c:GetAttribute("Ragdoll") == true or c:GetAttribute("Downed") == true or c:GetAttribute("GuardBroken") == true
end

local function threatActive(now)
    for ch, t in pairs(Threats) do
        if ch.Parent and now < t.impactAt + 0.22 and t.track.IsPlaying then
            return true
        end
        Threats[ch] = nil
    end
    return false
end

local function currentLead(kind, win)
    local base = (Cfg.lead > 0) and Cfg.lead or (win * 0.5)
    return math.clamp(base, 0.01, win * 0.95)
end

local function tunePf(delta)
    if not Cfg.autoTune then return end
    St.pf = math.clamp(St.pf + delta, 0.3, 1.3)
    Cal.pf = St.pf
    saveCal()
end

local function onMyOutcome(kind, now, atkName)
    local ctx = ParryCtx
    if not ctx or now - ctx.press > 1.0 then return end
    local off = now - ctx.press
    local exact = (atkName ~= nil and atkName == ctx.atk)

    if kind == "Parry" then
        if ctx.perfect then return end
        ctx.perfect, ctx.done, ctx.tuned = true, true, true
        table.clear(Threats)
        logLine(("PERFECT: F %.3fs [%s] mode=%s"):format(off, ctx.src, Cfg.parryMode))
    elseif kind == "BlockHit" then
        if not ctx.blockAt then
            ctx.blockAt = now
            task.delay(0.3, function()
                if not (ctx.perfect or ctx.done) then
                    ctx.done = true
                    logLine(("BLOCK biasa: F %.3fs [%s]"):format(off, ctx.src))
                end
            end)
        end
        if exact and not ctx.tuned then
            ctx.tuned = true
            task.delay(0.3, function()
                if not ctx.perfect then tunePf(-0.08) end
            end)
        end
    else
        if not (ctx.done or ctx.blockAt) then
            ctx.done = true
            logLine(("KENA: F %.3fs [%s]"):format(off, ctx.src))
        end
        if exact and not ctx.tuned and not ctx.blockAt and off < St.ping + 0.02 then
            ctx.tuned = true
            tunePf(0.1)
        end
    end
end

-- ==========================================
-- IMPACT ESTIMATOR
-- ==========================================
local function estimateImpact(id, info, track, speed, styleName)
    local cv, known = G.impact(styleName, info, speed)
    if cv and known then return cv + 0.01, "CFG" end
    local c = getImpact(id)
    if c then return c / speed, "CAL" end
    if cv then return cv + 0.01, "CFG?" end
    local len = track.Length
    if len and len > 0 then
        return (len * (info.kind == "M2" and 0.45 or 0.35)) / speed, "EST"
    end
    return (info.kind == "M2") and 0.35 or 0.2, "DEF"
end

local function sampleImpact(victim, info)
    local now = os.clock()
    local vHRP = victim:FindFirstChild("HumanoidRootPart")
    if not vHRP then return end
    local best
    for pass = 1, 2 do
        local bestDist
        for ch, a in pairs(LastAtk) do
            if ch ~= victim and not a.sampled and ch.Parent then
                local dt = now - a.t0
                if dt >= 0.05 and dt <= 1.2 then
                    local ok = true
                    if info then
                        if info.index and a.info.index and info.index ~= a.info.index then ok = false end
                        if ok and pass == 1 and info.key ~= "base" and a.info.key ~= info.key then ok = false end
                    end
                    if ok then
                        local h = ch:FindFirstChild("HumanoidRootPart")
                        if h then
                            local d = (h.Position - vHRP.Position).Magnitude
                            if d <= 14 and (not bestDist or d < bestDist) then best, bestDist = a, d end
                        end
                    end
                end
            end
        end
        if best then break end
    end
    if best then
        best.sampled = true
        addImpact(best.id, (now - best.t0) * best.speed)
    end
end

-- ==========================================
-- PARRY SCHEDULER (DUAL)
-- ==========================================
local function scheduleParry(char, kind, id, impactReal, src, valid, pendKey)
    local token = {}
    local pk = pendKey or char
    Pending[pk] = token
    local t0 = os.clock()
    local ping = Cfg.pingComp and (St.ping * St.pf) or 0
    local win = G.perfectWindow()
    local lead = currentLead(kind, win)
    if Cfg.perfectRate < 100 and math.random() * 100 > Cfg.perfectRate then
        lead = win + 0.07 + math.random() * 0.06
    end
    local jit = (Cfg.jitter > 0) and ((math.random() * 2 - 1) * Cfg.jitter / 1000) or 0
    local delay = math.max(0, impactReal - lead - ping + Cfg.parryOffset + jit)
    local impactAt = t0 + impactReal
    local pressAt = t0 + delay
    local cd = Cfg.respectCd and G.blockCd() or 0

    local function extendHold()
        Key.releaseAt = math.max(Key.releaseAt, impactAt + 0.15)
        Key.coverUntil = math.max(Key.coverUntil, impactAt + 0.05)
        RemoteBlock.releaseAt = math.max(RemoteBlock.releaseAt, impactAt + 0.15)
        RemoteBlock.coverUntil = math.max(RemoteBlock.coverUntil, impactAt + 0.05)
    end

    task.spawn(function()
        if Key.down or RemoteBlock.down then
            if cd > 0 then extendHold() return end
            local w = (pressAt - 0.075) - os.clock()
            if w > 0 then task.wait(w) end
            if not alive or Pending[pk] ~= token then return end
            if Key.down then
                if os.clock() < Key.coverUntil then return end
                Key.release()
            end
            if RemoteBlock.down then
                if os.clock() < RemoteBlock.coverUntil then return end
                RemoteBlock.release()
            end
        end

        if cd > 0 then
            local earliestV = Key.lastUp + cd
            local earliestR = RemoteBlock.lastUp + cd
            local earliest = math.max(earliestV, earliestR)
            if earliest > pressAt then
                if earliest > impactAt + 0.02 then return end
                pressAt = earliest
            end
        end

        local w = pressAt - os.clock() - 0.5 / math.max(St.fps, 20)
        if w > 0 then task.wait(w) end
        if not alive or Pending[pk] ~= token or not Cfg.parry then return end
        if valid and not valid() then return end
        if meBusy() then return end

        local myChar = LP.Character
        local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
        local tHRP = char:FindFirstChild("HumanoidRootPart")
        if not (myHRP and tHRP) then return end
        if (myHRP.Position - tHRP.Position).Magnitude > Cfg.parryDist + 3 then return end

        local gap = 0.065 - (os.clock() - math.max(Key.lastUp, RemoteBlock.lastUp))
        if gap > 0 then task.wait(gap) end
        if not alive or Pending[pk] ~= token then return end

        local hold = math.clamp(math.max(lead + 0.17, impactAt - os.clock() + 0.17), 0.18, 0.7)

        local firedVIM, firedRemote = false, false

        if useRemote() then
            local remoteOffset = Cfg.remoteOffsetMs / 1000
            if remoteOffset > 0 then task.wait(remoteOffset) end
            firedRemote = RemoteBlock.press(hold, impactAt + 0.05)
        end

        if useVIM() then
            firedVIM = Key.press(hold, impactAt + 0.05)
        end

        if firedVIM or firedRemote then
            ParryCtx = {
                id = id, kind = kind, src = src,
                press = Key.lastDown > 0 and Key.lastDown or RemoteBlock.lastDown,
                atk = char.Name, impactAt = impactAt,
                firedVIM = firedVIM, firedRemote = firedRemote,
            }
        end
    end)
end

-- ==========================================
-- HANDLER ANIMASI
-- ==========================================
local AnimConns = {}

local function onImpactAnim(char, info, isSelf, now)
    if info.kind == "GuardBreakHit" then GuardBroken[char] = now + 1.2 end
    if info.kind ~= "Parry" then pcall(sampleImpact, char, info) end
    if isSelf then
        local k = info.kind
        if k == "GuardBreakHit" then k = "GuardBroken" end
        onMyOutcome(k, now, nil)
    end
end

local function onAttackAnim(plr, char, track, id, info, now)
    local speed = track.Speed
    if not speed or speed <= 0.05 then speed = 1 end
    LastAtk[char] = { id = id, info = info, t0 = now, speed = speed, sampled = false }

    if info.kind == "M2" then
        M2Used[char] = now
        if plr then refreshEsp(plr) end
    end

    if plr and not info.shared and not StyleFromAttr[plr] and PlayerStyle[plr] ~= info.style then
        PlayerStyle[plr] = info.style
        if Cfg.styleTag then refreshEsp(plr) end
    end

    if not Cfg.parry then return end
    if info.kind == "M1" and not Cfg.parryM1 then return end
    if info.kind == "M2" and not Cfg.parryM2 then return end

    local myHRP = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    local tHRP = char:FindFirstChild("HumanoidRootPart")
    if not (myHRP and tHRP) then return end
    if (myHRP.Position - tHRP.Position).Magnitude > Cfg.parryDist then return end

    local styleName = (info.shared and plr and PlayerStyle[plr]) or info.style
    local impactReal, src = estimateImpact(id, info, track, speed, styleName)
    local valid = function() return track.IsPlaying end

    local extra, lastImpact = nil, impactReal
    if info.kind == "M2" then
        local delays = G.multiHit(styleName)
        if delays then
            extra = {}
            local d1 = tonumber(delays[1])
            for k = 2, #delays do
                local dk = tonumber(delays[k])
                if d1 and dk and dk > d1 then
                    extra[#extra + 1] = impactReal + (dk - d1) / speed
                end
            end
            lastImpact = extra[#extra] or impactReal
        end
    end

    Threats[char] = { impactAt = now + lastImpact, track = track }
    scheduleParry(char, info.kind, id, impactReal, src, valid)
    if extra then
        for k, ir in ipairs(extra) do
            scheduleParry(char, info.kind, id, ir, src .. "#" .. (k + 1), valid, char.Name .. "#" .. (k + 1))
        end
    end
end

local function hookChar(plr, char)
    if AnimConns[char] then return end
    local hum = char:WaitForChild("Humanoid", 5)
    local animator = hum and (hum:FindFirstChildOfClass("Animator") or hum:WaitForChild("Animator", 5))
    if not animator or AnimConns[char] or not alive then return end
    local isSelf = (plr == LP) or (char == LP.Character)

    AnimConns[char] = animator.AnimationPlayed:Connect(function(track)
        if not alive then return end
        local anim = track.Animation
        local id = anim and tostring(anim.AnimationId):match("%d+")
        local info = id and Anim[id]
        if not info then return end
        local kind = info.kind
        local now = os.clock()
        if kind == "M1" or kind == "M2" then
            if isSelf then
                St.lastMyAtk = now
                St.lastMyIdx = (kind == "M1") and (info.index or 1) or 0
                if kind == "M2" then St.lastMyM2 = now end
            else
                onAttackAnim(plr, char, track, id, info, now)
            end
        elseif kind == "BlockHit" or kind == "EHit" or kind == "GuardBreakHit" or kind == "Parry" then
            onImpactAnim(char, info, isSelf, now)
        end
    end)
end

local function unhookChar(char)
    if AnimConns[char] then AnimConns[char]:Disconnect() AnimConns[char] = nil end
    LastAtk[char], Pending[char], Threats[char], GuardBroken[char], M2Used[char] = nil, nil, nil, nil, nil
    for k = 2, 4 do Pending[char.Name .. "#" .. k] = nil end
end

-- ==========================================
-- CAM LOCK
-- ==========================================
local camTarget, highlight, aimAtt, aimAO

local function getClosestTarget()
    local myHRP = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not myHRP then return nil end
    local closest, closestDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr.Character then
            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                local d = (hrp.Position - myHRP.Position).Magnitude
                if d < closestDist then closest, closestDist = plr, d end
            end
        end
    end
    return closest, closestDist
end

local function createHighlight(plr)
    if highlight then highlight:Destroy() highlight = nil end
    if not plr or not plr.Character then return end
    highlight = Instance.new("Highlight")
    highlight.Adornee = plr.Character
    highlight.FillTransparency = 1
    highlight.OutlineTransparency = 0
    highlight.OutlineColor = Color3.fromRGB(255, 0, 0)
    highlight.Parent = CoreGui
end

local function cleanAim()
    if aimAO then aimAO:Destroy() aimAO = nil end
    if aimAtt then aimAtt:Destroy() aimAtt = nil end
end

local function ensureAim(hrp)
    if aimAO and aimAO.Parent and aimAtt and aimAtt.Parent == hrp then return end
    cleanAim()
    aimAtt = Instance.new("Attachment")
    aimAtt.Name = "HanseAimAtt"
    aimAtt.Parent = hrp
    aimAO = Instance.new("AlignOrientation")
    aimAO.Name = "HanseAim"
    aimAO.Mode = Enum.OrientationAlignmentMode.OneAttachment
    aimAO.Attachment0 = aimAtt
    aimAO.RigidityEnabled = false
    aimAO.MaxTorque = 1e7
    aimAO.MaxAngularVelocity = math.huge
    aimAO.Responsiveness = 100
    aimAO.Parent = hrp
end

local function stopCamLock()
    if highlight then highlight:Destroy() highlight = nil end
    camTarget = nil
    cleanAim()
    local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
    if hum then hum.AutoRotate = true end
    Camera.CameraType = Enum.CameraType.Custom
    if hum then Camera.CameraSubject = hum end
end

local function setCamLock(v)
    if v == Cfg.camLock then return end
    if v then
        camTarget = getClosestTarget()
        if not camTarget then return false end
        Cfg.camLock = true
        createHighlight(camTarget)
        Camera.CameraType = Enum.CameraType.Scriptable
    else
        Cfg.camLock = false
        stopCamLock()
    end
    return true
end

local function cycleTarget()
    local myHRP = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not myHRP then return end
    local list = {}
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr.Character then
            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                list[#list + 1] = { plr = plr, dist = (hrp.Position - myHRP.Position).Magnitude }
            end
        end
    end
    if #list == 0 then return end
    table.sort(list, function(a, b) return a.dist < b.dist end)
    local idx = 0
    for i, e in ipairs(list) do
        if e.plr == camTarget then idx = i break end
    end
    camTarget = list[(idx % #list) + 1].plr
    createHighlight(camTarget)
end

-- ==========================================
-- ESP
-- ==========================================
local EspData, Building, PlayerConns = {}, {}, {}
local E = { NAME = 14, HP = 11, BW = 90, BH = 7 }
E.NAME_H = E.NAME + 2
E.BAR_Y = E.NAME_H + 3
E.HP_Y = E.BAR_Y + E.BH + 2
E.GUI_H = E.HP_Y + E.HP + 2
local COL_WHITE = Color3.new(1, 1, 1)
local COL_SELF = Color3.fromRGB(85, 255, 85)
local COL_DOWN = Color3.fromRGB(255, 0, 0)

local function removeEsp(player)
    Building[player] = nil
    local data = EspData[player]
    if data then
        for _, c in ipairs(data.conns) do c:Disconnect() end
        data.gui:Destroy()
        EspData[player] = nil
    end
end

local function buildTags(player)
    local txt = ""
    if Cfg.styleTag then
        local slots = PlayerSlots[player]
        if StyleFromAttr[player] and slots and Cfg.bothSlots then
            local a = (slots.s1 ~= "" and slots.s1 or "-")
            local b = (slots.s2 ~= "" and slots.s2 or "-")
            if slots.slot == 2 then
                txt = txt .. " [" .. a .. " | >" .. b .. "]"
            else
                txt = txt .. " [>" .. a .. " | " .. b .. "]"
            end
        elseif PlayerStyle[player] then
            txt = txt .. " [" .. PlayerStyle[player] .. "]"
        end
    end
    if Cfg.m2Tag then
        local ch = player.Character
        local t = ch and M2Used[ch]
        if t then
            local rem = t + G.m2Cd(PlayerStyle[player] or "Basic") - os.clock()
            if rem > 0.05 then txt = txt .. (" [M2 %ds]"):format(math.ceil(rem)) end
        end
    end
    return txt
end

local function buildNameText(player, isDown)
    local txt = player.Name
    if Cfg.espDisplay and player.DisplayName ~= player.Name then
        txt = player.DisplayName .. " (@" .. player.Name .. ")"
    end
    txt = txt .. buildTags(player)
    if isDown then return "[DOWN] " .. txt end
    if player == LP then return "[YOU] " .. txt end
    return txt
end

local function applyHealth(data, force)
    local hum = data.hum
    local hp, maxHp = hum.Health, hum.MaxHealth
    if maxHp < 1 then maxHp = 100 end
    local pct = math.clamp(hp / maxHp, 0, 1)
    local down = hp < DOWN_HP
    local step = math.floor(pct * 100 + 0.5)

    local changed = force or down ~= data.down
    if changed then
        data.down = down
        data.name.Text = buildNameText(data.player, down)
        data.name.TextColor3 = down and COL_DOWN or (data.player == LP and COL_SELF or COL_WHITE)
    end
    if changed or step ~= data.step then
        data.step = step
        data.fill.Size = UDim2.fromScale(math.max(pct, 0.02), 1)
        data.fill.BackgroundColor3 = down and COL_DOWN or Color3.fromHSV(pct * 0.33, 0.85, 1)
    end
    if Cfg.espHp then
        local txt = math.floor(hp + 0.5) .. "/" .. math.floor(maxHp + 0.5)
        if txt ~= data.hpTxt then
            data.hpTxt = txt
            data.hpLabel.Text = txt
        end
    end
end

refreshEsp = function(player)
    local d = EspData[player]
    if d then applyHealth(d, true) end
end

local function createEsp(player, char, hum, adornee)
    local gui = Instance.new("BillboardGui")
    gui.Name = player.Name
    gui.Adornee = adornee
    gui.AlwaysOnTop = true
    gui.LightInfluence = 0
    gui.Size = UDim2.fromOffset(340, E.GUI_H)
    gui.StudsOffsetWorldSpace = Vector3.new(0, adornee.Name == "Head" and 2.6 or 4.6, 0)
    gui.MaxDistance = Cfg.espRadius

    local name = Instance.new("TextLabel")
    name.BackgroundTransparency = 1
    name.Size = UDim2.new(1, 0, 0, E.NAME_H)
    name.Font = Enum.Font.GothamBold
    name.TextSize = E.NAME
    name.TextColor3 = COL_WHITE
    name.TextStrokeTransparency = 0
    name.TextTruncate = Enum.TextTruncate.AtEnd
    name.Parent = gui

    local barBg = Instance.new("Frame")
    barBg.AnchorPoint = Vector2.new(0.5, 0)
    barBg.Position = UDim2.new(0.5, 0, 0, E.BAR_Y)
    barBg.Size = UDim2.fromOffset(E.BW, E.BH)
    barBg.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
    barBg.BorderSizePixel = 1
    barBg.BorderColor3 = Color3.new(0, 0, 0)
    barBg.Parent = gui

    local fill = Instance.new("Frame")
    fill.Size = UDim2.fromScale(1, 1)
    fill.BackgroundColor3 = Color3.fromRGB(50, 255, 50)
    fill.BorderSizePixel = 0
    fill.Parent = barBg

    local hpLabel = Instance.new("TextLabel")
    hpLabel.BackgroundTransparency = 1
    hpLabel.Position = UDim2.new(0, 0, 0, E.HP_Y)
    hpLabel.Size = UDim2.new(1, 0, 0, E.HP + 2)
    hpLabel.Font = Enum.Font.GothamBold
    hpLabel.TextSize = E.HP
    hpLabel.TextColor3 = COL_WHITE
    hpLabel.TextStrokeTransparency = 0
    hpLabel.Visible = Cfg.espHp
    hpLabel.Parent = gui

    local data = { player = player, char = char, hum = hum, gui = gui, name = name, fill = fill, hpLabel = hpLabel, conns = {} }
    data.conns[1] = hum.HealthChanged:Connect(function() applyHealth(data) end)
    data.conns[2] = hum:GetPropertyChangedSignal("MaxHealth"):Connect(function() applyHealth(data) end)
    applyHealth(data, true)
    gui.Parent = EspFolder
    return data
end

local function attachEsp(player, char)
    if not Cfg.esp or not char then return end
    if player == LP and not Cfg.espSelf then return end
    local cur = EspData[player]
    if (cur and cur.char == char) or Building[player] == char then return end

    Building[player] = char
    task.spawn(function()
        local hum = char:WaitForChild("Humanoid", 5)
        local adornee = char:WaitForChild("Head", 3) or char:FindFirstChild("HumanoidRootPart")
        local stale = Building[player] ~= char
        if not stale then Building[player] = nil end
        if stale or not Cfg.esp or player.Character ~= char or not hum or not adornee then return end
        removeEsp(player)
        EspData[player] = createEsp(player, char, hum, adornee)
    end)
end

local function setEsp(state)
    if Cfg.esp == state then return end
    Cfg.esp = state
    if state then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr.Character then attachEsp(plr, plr.Character) end
        end
    else
        table.clear(Building)
        for plr in pairs(EspData) do removeEsp(plr) end
    end
end

-- ==========================================
-- SETUP PLAYER
-- (listener atribut: cuma atribut style yang didengerin, bukan semua atribut)
-- ==========================================
local STYLE_ATTRS = { "CombatStyle", "CombatStyleOverride", "CombatStyleSlot1", "CombatStyleSlot2", "CombatStyleEquippedSlot" }

local function watchStyle(pc, group, inst, plr)
    for _, name in ipairs(STYLE_ATTRS) do
        local k = group .. name
        if pc[k] then pc[k]:Disconnect() end
        pc[k] = inst:GetAttributeChangedSignal(name):Connect(function() updateStyleFromAttrs(plr) end)
    end
end

local function setupPlayer(plr)
    if PlayerConns[plr] then return end
    local pc = {}
    PlayerConns[plr] = pc

    local function onChar(char)
        attachEsp(plr, char)
        watchStyle(pc, "chr", char, plr)
        task.defer(updateStyleFromAttrs, plr)
        task.spawn(function()
            local pd = char:WaitForChild("PlayerData", 10)
            if pd and alive and plr.Character == char then
                watchStyle(pc, "pd", pd, plr)
                updateStyleFromAttrs(plr)
            end
        end)
        task.spawn(hookChar, plr, char)
    end

    pc.added = plr.CharacterAdded:Connect(onChar)
    pc.removing = plr.CharacterRemoving:Connect(function(char)
        removeEsp(plr)
        unhookChar(char)
    end)
    watchStyle(pc, "plr", plr, plr)

    updateStyleFromAttrs(plr)
    if plr.Character then onChar(plr.Character) end
end

for _, plr in ipairs(Players:GetPlayers()) do setupPlayer(plr) end
keep(Players.PlayerAdded:Connect(setupPlayer))
keep(Players.PlayerRemoving:Connect(function(plr)
    removeEsp(plr)
    PlayerStyle[plr], PlayerSlots[plr], StyleFromAttr[plr] = nil, nil, nil
    local pc = PlayerConns[plr]
    if pc then
        for _, c in pairs(pc) do pcall(function() c:Disconnect() end) end
        PlayerConns[plr] = nil
    end
end))

-- ==========================================
-- NETWORK
-- ==========================================
local seenPair, seenCount = {}, 0

local function onPair(key, a, v, uid)
    if not alive or type(key) ~= "string" then return end
    local now = os.clock()
    if key == "WingChunCounterStartup" and type(a) == "string" then
        CounterUntil[a] = now + 0.7
    end
    if type(uid) ~= "string" then return end
    if seenPair[uid] then return end
    seenPair[uid] = true
    seenCount += 1
    if seenCount > 200 then table.clear(seenPair) seenCount = 0 end

    local me = LP.Name
    if v == me then
        if key:find("PerfectBlocked", 1, true) then onMyOutcome("Parry", now, a)
        elseif key:find("GuardBroken", 1, true) then onMyOutcome("GuardBroken", now, a)
        elseif key:find("Blocked", 1, true) then onMyOutcome("BlockHit", now, a)
        elseif key:find("Hit", 1, true) then onMyOutcome("EHit", now, a) end
    elseif a == me and key:find("PerfectBlocked", 1, true) then
        St.m1Pause = now + (key:sub(1, 2) == "M2" and 1.0 or 0.35)
    end
end

task.spawn(function()
    local sh = RS:WaitForChild("Shared", 10)
    local net = sh and sh:WaitForChild("Network", 10)
    if not net or not alive then return end
    local ure = net:WaitForChild("CombatBroadcastURE", 10)
    if ure and alive then
        keep(ure.OnClientEvent:Connect(function(key, a, v, uid) onPair(key, a, v, uid) end))
    end
    local cre = net:WaitForChild("CombatClientRemoteEvent", 10)
    if cre and alive then
        keep(cre.OnClientEvent:Connect(function(action, key, a, v, uid)
            if action == "CombatPairCosmetic" then onPair(key, a, v, uid) end
        end))
    end
end)

-- ==========================================
-- STAMINA (ID-agnostic: nempel ke API Blink game, bukan baca nomor paket)
-- ==========================================
local Stam = { state = "belum", restore = nil }

local function getUpvals(fn)
    local ok, res = pcall(function()
        if debug and debug.getupvalues then return debug.getupvalues(fn) end
        if getupvalues then return getupvalues(fn) end
        local get = (debug and debug.getupvalue) or getupvalue
        if get then
            local t, i = {}, 1
            while i <= 20 do
                local okg, a, b = pcall(get, fn, i)
                if not okg or (a == nil and b == nil) then break end
                t[#t + 1] = (b ~= nil) and b or a
                i += 1
            end
            return t
        end
    end)
    if ok then return res end
    return nil
end

-- return nil, "Tabel[slot]" kalau berhasil; return "alasan gagal" kalau gagal
local function hookStaminaViaClient()
    local sh = RS:WaitForChild("Shared", 10)
    local net = sh and sh:WaitForChild("Network", 10)
    local mod = net and net:WaitForChild("Client", 10)
    if not mod then return "module Client tidak ketemu" end
    local Client = require(mod)
    local api = type(Client) == "table" and Client.MovementStaminaUpdate
    if type(api) ~= "table" or type(api.On) ~= "function" then return "API MovementStaminaUpdate hilang" end

    local handlers
    local ups = getUpvals(api.On)
    if ups then
        for _, u in pairs(ups) do
            if type(u) == "table" and type(rawget(u, "Unreliable")) == "table" then handlers = u break end
        end
    end
    if not handlers then return "executor gak bisa baca upvalue" end

    local names = { "Unreliable", "Reliable" }
    local before = {}
    for _, tn in ipairs(names) do
        before[tn] = {}
        for i, f in pairs(handlers[tn] or {}) do before[tn][i] = f end
    end

    local probe = function() end
    api.On(probe) -- nimpa handler game sebentar, langsung dibalikin (dibungkus) di bawah

    local tname, slot
    for _, tn in ipairs(names) do
        for i, f in pairs(handlers[tn] or {}) do
            if f == probe then tname, slot = tn, i break end
        end
        if slot then break end
    end
    if not slot then
        for _, tn in ipairs(names) do
            for i, f in pairs(handlers[tn] or {}) do
                if before[tn][i] ~= f then tname, slot = tn, i break end
            end
            if slot then break end
        end
    end
    if not slot then return "slot handler gak ketemu" end

    local slots = handlers[tname]
    local orig = before[tname][slot]
    local wrapper
    wrapper = function(v, ...)
        if type(v) == "number" then pcall(updateStamina, v) end
        if orig then return orig(v, ...) end -- handler asli game tetap jalan
    end
    slots[slot] = wrapper
    Stam.restore = function()
        if slots[slot] == wrapper then slots[slot] = orig end
    end

    -- kalau game daftar ulang handler-nya (nimpa wrapper), pasang lagi
    task.spawn(function()
        while alive do
            task.wait(3)
            local cur = slots[slot]
            if alive and cur ~= wrapper then
                orig = cur
                slots[slot] = wrapper
            end
        end
    end)
    return nil, ("%s[%d]"):format(tname, slot)
end

-- cadangan: kalau stamina ternyata disimpan sebagai atribut angka
local function hookStaminaViaAttr()
    local c = LP.Character
    local cands = { LP }
    if c then
        cands[#cands + 1] = c
        local pd = c:FindFirstChild("PlayerData")
        if pd then cands[#cands + 1] = pd end
    end
    for _, inst in ipairs(cands) do
        for k, v in pairs(inst:GetAttributes()) do
            if type(v) == "number" and tostring(k):lower():find("stamina", 1, true) then
                local function push()
                    local x = inst:GetAttribute(k)
                    if type(x) == "number" then updateStamina(math.floor(x + 0.5)) end
                end
                keep(inst:GetAttributeChangedSignal(k):Connect(push))
                push()
                return k
            end
        end
    end
    return nil
end

task.spawn(function()
    task.wait(2) -- biar module game selesai load
    local ok, err, slot = pcall(hookStaminaViaClient)
    if ok and err == nil and slot then
        Stam.state = "OK (" .. slot .. ")"
        return
    end
    local reason = ok and tostring(err) or ("error: " .. tostring(err))
    local ok2, attr = pcall(hookStaminaViaAttr)
    if ok2 and attr then
        Stam.state = "OK (atribut " .. tostring(attr) .. ")"
        return
    end
    Stam.state = "GAGAL: " .. reason
    Cfg.staminaHud = false
    UI.stamina.Visible = false
    warn("[Hanse] HUD Stamina dimatikan - " .. reason)
end)

-- ==========================================
-- LOOP UTAMA
-- ==========================================
keep(RunService.Heartbeat:Connect(function(dt)
    St.fps = St.fps + ((1 / math.max(dt, 1e-4)) - St.fps) * 0.05
    if Key.down and os.clock() >= Key.releaseAt then Key.release() end
    if RemoteBlock.down and os.clock() >= RemoteBlock.releaseAt then RemoteBlock.release() end
end))

keep(RunService.RenderStepped:Connect(function()
    if not Cfg.camLock then return end
    local char = LP.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not (hrp and hum) then return end

    if not camTarget or not camTarget.Character or not camTarget.Character:FindFirstChild("HumanoidRootPart") then
        camTarget = getClosestTarget()
        if camTarget then createHighlight(camTarget) end
    end

    local tc = camTarget and camTarget.Character
    local focus = tc and (tc:FindFirstChild("UpperTorso") or tc:FindFirstChild("Torso") or tc:FindFirstChild("HumanoidRootPart"))
    if not focus then hum.AutoRotate = true return end

    hum.AutoRotate = false
    local p, f = hrp.Position, focus.Position
    if (f.X - p.X) ^ 2 + (f.Z - p.Z) ^ 2 > 0.01 then
        ensureAim(hrp)
        aimAO.CFrame = CFrame.lookAt(p, Vector3.new(f.X, p.Y, f.Z))
    end
    local mid = (p + f) / 2
    local desired = p + (hrp.CFrame.LookVector * -12) + Vector3.new(0, 6, 0)
    Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(desired, mid), 0.25)
end))

task.spawn(function()
    local first = true
    while alive do
        local ok, v = pcall(function() return Stats.Network.ServerStatsItem["Data Ping"]:GetValue() end)
        if ok and type(v) == "number" then
            St.ping = first and (v / 1000) or (St.ping * 0.5 + (v / 1000) * 0.5)
            first = false
        end
        if Cfg.pingHud then
            local rem = St.lastMyM2 + G.m2Cd(PlayerStyle[LP] or "Basic") - os.clock()
            local m2txt = (St.lastMyM2 > 0 and rem > 0) and ("M2 %ds"):format(math.ceil(rem)) or "M2 READY"
            local up = 0
            pcall(function() up = Stats.DataSendKbps end)
            UI.ping.Text = ("Ping %dms | FPS %d | %s | %s | UP %.1fkb/s"):format(
                math.floor(St.ping * 1000 + 0.5), math.floor(St.fps + 0.5), m2txt, Cfg.parryMode, up)
        end
        if Cfg.m2Tag and Cfg.esp then
            local now = os.clock()
            for _, d in pairs(EspData) do
                local t = M2Used[d.char]
                if t and now - t < 25 then pcall(applyHealth, d, true) end
            end
        end
        task.wait(0.5)
    end
end)

-- ==========================================
-- UNLOAD
-- ==========================================
env.HanseUnload = function()
    alive = false
    pcall(function() if Stam.restore then Stam.restore() end end)
    Key.release()
    RemoteBlock.forceRelease()
    for _, c in ipairs(Conns) do pcall(function() c:Disconnect() end) end
    for _, pc in pairs(PlayerConns) do
        for _, c in pairs(pc) do pcall(function() c:Disconnect() end) end
    end
    for _, c in pairs(AnimConns) do pcall(function() c:Disconnect() end) end
    table.clear(PlayerConns) table.clear(AnimConns)
    table.clear(LastAtk) table.clear(Pending) table.clear(Threats) table.clear(M2Used)
    for plr in pairs(EspData) do pcall(removeEsp, plr) end
    if Cfg.camLock then Cfg.camLock = false pcall(stopCamLock) end
    pcall(cleanAim)
    pcall(function() if highlight then highlight:Destroy() end end)
    pcall(function() EspFolder:Destroy() end)
    pcall(function() Hud:Destroy() end)
    pcall(function() if env.HanseFluent then env.HanseFluent:Destroy() end end)
end

-- ==========================================
-- FLUENT UI
-- ==========================================
local function buildUI()
    local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
    env.HanseFluent = Fluent

    local Window = Fluent:CreateWindow({
        Title = "Hanse On Top",
        SubTitle = "V7 Dual + ESP",
        TabWidth = 150,
        Size = UDim2.fromOffset(560, 440),
        Acrylic = false,
        Theme = "Dark",
        MinimizeKey = Enum.KeyCode.RightControl,
    })

    local Tabs = {
        Combat = Window:AddTab({ Title = "Combat", Icon = "swords" }),
        Visuals = Window:AddTab({ Title = "Visuals", Icon = "eye" }),
        Tuning = Window:AddTab({ Title = "Tuning", Icon = "sliders-horizontal" }),
        Settings = Window:AddTab({ Title = "Settings", Icon = "settings" }),
    }

    local function toggle(tab, id, title, key, cb, desc)
        local t = tab:AddToggle(id, { Title = title, Description = desc, Default = Cfg[key] })
        t:OnChanged(function(v)
            Cfg[key] = v
            if cb then cb(v) end
        end)
        return t
    end

    local function slider(tab, id, title, key, min, max, rounding, desc)
        local s = tab:AddSlider(id, { Title = title, Description = desc, Default = Cfg[key], Min = min, Max = max, Rounding = rounding })
        s:OnChanged(function(v)
            local n = tonumber(v)
            if n then Cfg[key] = n end
        end)
        return s
    end

    -- ===== COMBAT =====
    local cl = Tabs.Combat:AddToggle("CamLock", { Title = "Cam Lock", Default = false })
    cl:OnChanged(function(v)
        if v ~= Cfg.camLock then
            local ok = setCamLock(v)
            if ok == false then cl:SetValue(false) end
        end
    end)
    Tabs.Combat:AddKeybind("RetargetKey", { Title = "Retarget Cam Lock", Mode = "Toggle", Default = "G" })
        :OnClick(function() if Cfg.camLock then cycleTarget() end end)

    Tabs.Combat:AddDropdown("ParryMode", {
        Title = "Parry Mode",
        Description = "Dual = VIM + Remote barengan | VIM = cuma input | Remote = cuma event",
        Values = { "Dual", "VIM", "Remote" },
        Default = 1,
    }):OnChanged(function(v) if type(v) == "string" then Cfg.parryMode = v end end)

    toggle(Tabs.Combat, "AutoParry", "Auto Parry", "parry")
    toggle(Tabs.Combat, "ParryM1", "Parry M1", "parryM1")
    toggle(Tabs.Combat, "ParryM2", "Parry M2", "parryM2")
    slider(Tabs.Combat, "ParryDist", "Parry Distance", "parryDist", 5, 40, 0)

    -- ===== VISUALS =====
    local es = Tabs.Visuals:AddToggle("EspToggle", { Title = "ESP (nama + HP + style)", Default = false })
    es:OnChanged(function(v) setEsp(v) end)

    toggle(Tabs.Visuals, "StyleTag", "Tag Style", "styleTag", function()
        for _, d in pairs(EspData) do applyHealth(d, true) end
    end)
    toggle(Tabs.Visuals, "BothSlots", "Tampilkan 2 Slot Style", "bothSlots", function()
        for _, d in pairs(EspData) do applyHealth(d, true) end
    end)
    toggle(Tabs.Visuals, "M2Tag", "Tag Cooldown M2 Lawan", "m2Tag", function()
        for _, d in pairs(EspData) do applyHealth(d, true) end
    end)
    toggle(Tabs.Visuals, "HpText", "Angka HP", "espHp", function(v)
        for _, d in pairs(EspData) do
            d.hpLabel.Visible = v
            d.hpTxt = nil
            if v then applyHealth(d) end
        end
    end)
    toggle(Tabs.Visuals, "EspSelf", "ESP Diri Sendiri", "espSelf", function(v)
        if not Cfg.esp then return end
        if v then attachEsp(LP, LP.Character) else removeEsp(LP) end
    end)
    toggle(Tabs.Visuals, "DispName", "Display Name", "espDisplay", function()
        for _, d in pairs(EspData) do applyHealth(d, true) end
    end)
    Tabs.Visuals:AddSlider("EspRadius", {
        Title = "Radius ESP", Default = Cfg.espRadius, Min = 10, Max = MAX_RADIUS, Rounding = 0,
    }):OnChanged(function(v)
        local n = tonumber(v)
        if not n then return end
        Cfg.espRadius = n
        for _, d in pairs(EspData) do d.gui.MaxDistance = n end
    end)
    toggle(Tabs.Visuals, "PingHud", "HUD Ping / FPS / M2", "pingHud", function(v) UI.ping.Visible = v end)
    toggle(Tabs.Visuals, "StamHud", "HUD Stamina", "staminaHud", function(v)
        UI.stamina.Visible = v and UI.stamHas()
    end)

    -- ===== TUNING =====
    toggle(Tabs.Tuning, "PingComp", "Kompensasi Ping", "pingComp")
    toggle(Tabs.Tuning, "RespectCd", "Hormati Block Cooldown", "respectCd")
    toggle(Tabs.Tuning, "AutoTune", "Auto-tune Ping Factor", "autoTune")
    slider(Tabs.Tuning, "LeadManual", "Lead Manual (0 = auto)", "lead", 0, 0.12, 3)
    slider(Tabs.Tuning, "Offset", "Offset Delay", "parryOffset", -0.15, 0.15, 3)
    slider(Tabs.Tuning, "Jitter", "Jitter (ms)", "jitter", 0, 40, 0)
    slider(Tabs.Tuning, "PerfectRate", "Perfect Rate (%)", "perfectRate", 50, 100, 0)
    Tabs.Tuning:AddSlider("RemoteOffset", {
        Title = "Remote Offset (ms)",
        Description = "Negatif = remote lebih awal, Positif = VIM duluan",
        Default = Cfg.remoteOffsetMs, Min = -100, Max = 100, Rounding = 0,
    }):OnChanged(function(v) Cfg.remoteOffsetMs = tonumber(v) or -8 end)
    toggle(Tabs.Tuning, "Debug", "Debug (F9)", "debug")

    Tabs.Tuning:AddButton({
        Title = "Dump Status (F9)",
        Callback = function()
            print("======== [Hanse V7] STATUS ========")
            print(("  Parry Mode    : %s"):format(Cfg.parryMode))
            print(("  Parry         : %s"):format(Cfg.parry and "ON" or "OFF"))
            print(("  Remote OK     : %s"):format(G.remoteOK and "YES" or "NO"))
            print(("  Remote Fires  : %d | Fails: %d"):format(RemoteBlock.fires, RemoteBlock.fails))
            print(("  Ping (RTT)    : %.0f ms"):format(St.ping * 1000))
            print(("  Ping Factor   : %.2f"):format(St.pf))
            print(("  CombatConfig  : %s"):format(G.CC and "OK" or "NO"))
            print(("  Anim DB       : %d"):format(animCount))
            print(("  Stamina HUD   : %s"):format(Stam.state))
            print(("  PerfectWindow : %.3f s"):format(G.perfectWindow()))
            print("===================================")
        end,
    })

    Tabs.Tuning:AddButton({
        Title = "Reset Auto-Tune",
        Callback = function()
            St.pf = 0.75
            Cal.pf = 0.75
            saveCal()
        end,
    })

    -- ===== SETTINGS =====
    Tabs.Settings:AddButton({
        Title = "Unload Hanse",
        Callback = function() if env.HanseUnload then env.HanseUnload() end end,
    })
end

local ok, err = pcall(buildUI)
if not ok then warn("[Hanse] UI failed: " .. tostring(err)) end
