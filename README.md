-- language: Luau, target: Roblox MOBILE, UI: Orion
-- 7zK FREE — funções premium travadas

local SCRIPT_URL = ""

--=====================================================================
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace        = game:GetService("Workspace")
local Lighting         = game:GetService("Lighting")
local TeleportService  = game:GetService("TeleportService")
local HttpService      = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
local Camera      = Workspace.CurrentCamera

local OrionLib = loadstring(game:HttpGet(
    "https://raw.githubusercontent.com/Qanuir/orion-ui/refs/heads/main/source.lua"
))()

--=====================================================================
local State = {
    Aimbot = false, AimPart = "Head", FOV = 140, ShowFov = false,
    IgnoreDead = true, ThroughWalls = false, SilentAim = false,
    StickClosest = false, StickDistance = 30,

    ESPEnabled = false, ESPColor = Color3.fromRGB(0, 255, 120),
    ESPLine = false, ESPLineOrigin = "Bottom",
    ESPHealth = false, ESPDistance = false, ESPHitCheck = false,
    HitColorOK = Color3.fromRGB(40, 220, 90),
    HitColorNo = Color3.fromRGB(240, 65, 95),

    Spinbot = false, SpinSpeed = 30, ThirdPerson = false,

    NoFog = false, NoCamShake = false, NoEffects = false,
    TimeOfDay = 12,

    AntiAdmin = false, AntiAFK = false, AutoServerHop = false,
    AutoLoad = false,
}

local Drawings = {}
local spinAngle = 0
local spinJointRef = nil
local spinJointOrigC0 = nil
local originalFogEnd = nil
local originalFogStart = nil
local originalFogColor = nil
local disabledEffects = {}

--=====================================================================
-- HELPER PREMIUM
--=====================================================================
local AVISO = "Use a versão com key para ativar essa função"

local function avisoPremium()
    pcall(function()
        OrionLib:MakeNotification({
            Name = "7zK",
            Content = AVISO,
            Time = 4,
        })
    end)
end

local function avisoETrava(controle, valorDefault)
    avisoPremium()
    task.defer(function()
        task.wait(0.05)
        pcall(function()
            if controle and controle.Set then
                controle:Set(valorDefault)
            end
        end)
    end)
end

--=====================================================================
-- UTIL
--=====================================================================
local function isAlive(plr)
    local char = plr.Character
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

local function screenCenter()
    local v = Camera.ViewportSize
    return Vector2.new(v.X / 2, v.Y / 2)
end

local function worldToScreen(pos)
    local p, onScreen = Camera:WorldToViewportPoint(pos)
    return Vector2.new(p.X, p.Y), onScreen, p.Z
end

local function getAimPart(plr)
    local char = plr.Character
    if not char then return nil end
    if State.AimPart == "Head" then
        return char:FindFirstChild("Head")
    end
    return char:FindFirstChild("UpperTorso")
        or char:FindFirstChild("Torso")
        or char:FindFirstChild("Head")
end

local function parseColor(c)
    if typeof(c) == "Color3" then return c end
    if type(c) == "table" then
        if c.R and c.G and c.B then return Color3.fromRGB(c.R, c.G, c.B) end
        if c[1] and c[2] and c[3] then return Color3.fromRGB(c[1], c[2], c[3]) end
        if c.Color then return parseColor(c.Color) end
    end
    return State.ESPColor
end

local function healthColor(ratio)
    if ratio <= 0.30 then return Color3.fromRGB(230, 40, 40) end
    if ratio <= 0.60 then return Color3.fromRGB(240, 200, 40) end
    return Color3.fromRGB(40, 220, 90)
end

local function hasLineOfSight(plr, part)
    local origin = Camera.CFrame.Position
    local dir    = part.Position - origin
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local myChar = LocalPlayer.Character
    params.FilterDescendantsInstances = myChar and { myChar } or {}
    local hit = Workspace:Raycast(origin, dir, params)
    if not hit then return true end
    local hitChar = hit.Instance:FindFirstAncestorOfClass("Model")
    return hitChar == plr.Character
end

--=====================================================================
-- SPIN BOT (só roda se State.Spinbot — que nunca vai ligar no free)
--=====================================================================
local function resetSpinRefs()
    if spinJointRef and spinJointOrigC0 then
        pcall(function() spinJointRef.C0 = spinJointOrigC0 end)
    end
    spinJointRef = nil
    spinJointOrigC0 = nil
end

RunService:BindToRenderStep("7zK_Spinbot", Enum.RenderPriority.Character.Value + 10, function(dt)
    if not State.Spinbot then resetSpinRefs(); return end
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local rj = hrp:FindFirstChild("RootJoint")
    if not rj or not rj:IsA("Motor6D") then return end
    if spinJointRef ~= rj then
        spinJointRef = rj
        spinJointOrigC0 = rj.C0
    end
    spinAngle = (spinAngle + State.SpinSpeed * dt * 8) % 360
    rj.C0 = spinJointOrigC0 * CFrame.Angles(0, math.rad(spinAngle), 0)
end)

--=====================================================================
-- 3 PESSOA (funções ficam definidas, mas nunca chamadas no free)
--=====================================================================
local function applyThirdPerson()
    pcall(function() LocalPlayer.CameraMode = Enum.CameraMode.Classic end)
    pcall(function()
        LocalPlayer.CameraMaxZoomDistance = 128
        LocalPlayer.CameraMinZoomDistance = 12.5
    end)
    pcall(function()
        if Camera.CameraType == Enum.CameraType.Scriptable then
            Camera.CameraType = Enum.CameraType.Custom
        end
    end)
    pcall(function()
        Camera.CameraSubject = LocalPlayer.Character
            and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") or nil
    end)
end

local function stopThirdPerson()
    pcall(function() LocalPlayer.CameraMode = Enum.CameraMode.Classic end)
end

RunService.RenderStepped:Connect(function()
    if not State.ThirdPerson then return end
    pcall(function()
        if LocalPlayer.CameraMode ~= Enum.CameraMode.Classic then
            LocalPlayer.CameraMode = Enum.CameraMode.Classic
        end
        if LocalPlayer.CameraMinZoomDistance < 12.5 then
            LocalPlayer.CameraMinZoomDistance = 12.5
        end
        if LocalPlayer.CameraMaxZoomDistance < 128 then
            LocalPlayer.CameraMaxZoomDistance = 128
        end
        if Camera.CameraType == Enum.CameraType.Scriptable then
            Camera.CameraType = Enum.CameraType.Custom
        end
    end)
end)

--=====================================================================
-- VISUAL
--=====================================================================
if originalFogEnd == nil then
    originalFogEnd = Lighting.FogEnd
    originalFogStart = Lighting.FogStart
    originalFogColor = Lighting.FogColor
end

local function applyNoFog()
    if State.NoFog then
        pcall(function()
            Lighting.FogEnd = 100000
            Lighting.FogStart = 100000
            Lighting.FogColor = Color3.fromRGB(255, 255, 255)
        end)
    else
        pcall(function()
            Lighting.FogEnd = originalFogEnd or 100000
            Lighting.FogStart = originalFogStart or 0
            Lighting.FogColor = originalFogColor or Color3.fromRGB(192, 192, 192)
        end)
    end
end

local function applyTimeOfDay()
    pcall(function() Lighting.ClockTime = State.TimeOfDay end)
end

task.spawn(function()
    while true do
        task.wait(0.5)
        if State.NoFog then applyNoFog() end
        if State.TimeOfDay ~= 12 then applyTimeOfDay() end
    end
end)

local function applyNoEffects()
    if State.NoEffects then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if obj:IsA("ParticleEmitter") or obj:IsA("Trail")
            or obj:IsA("Beam") or obj:IsA("Fire")
            or obj:IsA("Smoke") or obj:IsA("Sparkles") then
                if not disabledEffects[obj] then
                    disabledEffects[obj] = obj.Enabled
                    obj.Enabled = false
                end
            end
        end
    else
        for obj, state in pairs(disabledEffects) do
            pcall(function()
                if obj and obj.Parent then obj.Enabled = state end
            end)
        end
        disabledEffects = {}
    end
end

task.spawn(function()
    while true do
        task.wait(1)
        if State.NoEffects then applyNoEffects() end
    end
end)

--=====================================================================
-- SEGURANÇA
--=====================================================================
local ADMIN_KEYWORDS = {
    "admin", "administrador", "mod", "moderador", "moderator", "owner",
    "dono", "staff", "dev", "developer", "criador", "creator",
    "manager", "gerente", "supervisor", "super", "hoster", "host",
    "support", "suporte", "[adm]", "[admin]", "[mod]", "[owner]",
}

local function isLikelyAdmin(plr)
    if plr == LocalPlayer then return false end
    local name = string.lower(plr.Name)
    local display = string.lower(plr.DisplayName or "")
    for _, kw in ipairs(ADMIN_KEYWORDS) do
        if name:find(kw, 1, true) or display:find(kw, 1, true) then return true end
    end
    local ok, rank = pcall(function() return plr:GetRankInGroup(game.CreatorId) end)
    if ok and typeof(rank) == "number" and rank >= 200 then return true end
    return false
end

task.spawn(function()
    while true do
        task.wait(1)
        if State.AntiAdmin then
            for _, plr in ipairs(Players:GetPlayers()) do
                if isLikelyAdmin(plr) then
                    warn("[7zK] admin: " .. plr.Name)
                    pcall(function() LocalPlayer:Kick("admin") end)
                    pcall(function() TeleportService:Teleport(game.PlaceId, LocalPlayer) end)
                    break
                end
            end
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(30)
        if State.AntiAFK then
            local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            if hum then pcall(function() hum.Jump = true end) end
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(600)
        if State.AutoServerHop then
            local okFetch = pcall(function()
                local url = string.format(
                    "https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100",
                    game.PlaceId
                )
                local raw = game:HttpGet(url)
                local data = HttpService:JSONDecode(raw)
                if not data or not data.data then return end
                for _, srv in ipairs(data.data) do
                    if srv.id and srv.playing < srv.maxPlayers and srv.id ~= game.JobId then
                        TeleportService:TeleportToPlaceInstance(game.PlaceId, srv.id, LocalPlayer)
                        return true
                    end
                end
            end)
            if not okFetch then
                pcall(function() TeleportService:Teleport(game.PlaceId, LocalPlayer) end)
            end
        end
    end
end)

--=====================================================================
-- AIMBOT
--=====================================================================
local function getClosestTarget()
    local center = screenCenter()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    local best, bestVal = nil, math.huge

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        if State.IgnoreDead and not isAlive(plr) then continue end
        local char = plr.Character
        local part = getAimPart(plr)
        local hrp  = char and char:FindFirstChild("HumanoidRootPart")
        if not (part and hrp) then continue end
        local dist3d = myRoot and (myRoot.Position - hrp.Position).Magnitude or math.huge
        local sp, onScreen = Camera:WorldToViewportPoint(part.Position)
        if not onScreen or sp.Z <= 0 then continue end
        local screenDist = (Vector2.new(sp.X, sp.Y) - center).Magnitude
        local metric
        if State.StickClosest then
            if dist3d > State.StickDistance then continue end
            metric = dist3d
        else
            if screenDist > State.FOV then continue end
            metric = screenDist
        end
        if metric >= bestVal then continue end
        if not State.ThroughWalls then
            if not hasLineOfSight(plr, part) then continue end
        end
        bestVal = metric
        best = plr
    end
    return best
end

RunService:BindToRenderStep("7zK_Aimbot", Enum.RenderPriority.Camera.Value + 1, function()
    if not State.Aimbot then return end
    local target = getClosestTarget()
    if not target then return end
    local part = getAimPart(target)
    if not part then return end
    local goal = CFrame.new(Camera.CFrame.Position, part.Position)
    local alpha
    if State.SilentAim then alpha = 1.0
    elseif State.ThroughWalls then alpha = 0.85
    elseif State.StickClosest then alpha = 0.65
    else alpha = 0.45 end
    Camera.CFrame = Camera.CFrame:Lerp(goal, alpha)
end)

--=====================================================================
-- MOSTRAR FOV
--=====================================================================
local FovCircle = Drawing.new("Circle")
FovCircle.Thickness = 1.5
FovCircle.NumSides = 96
FovCircle.Filled = false
FovCircle.Transparency = 1
FovCircle.Color = Color3.fromRGB(255, 255, 255)
FovCircle.Visible = false

RunService.RenderStepped:Connect(function()
    if not State.ShowFov or State.StickClosest then
        FovCircle.Visible = false
        return
    end
    FovCircle.Visible  = true
    FovCircle.Radius   = State.FOV
    FovCircle.Position = screenCenter()
end)

--=====================================================================
-- ESP
--=====================================================================
local function newDrawing(class)
    local ok, d = pcall(Drawing.new, class)
    if not ok then return nil end
    d.Visible = false
    return d
end

local function ensureESP(plr)
    if Drawings[plr] then return Drawings[plr] end
    local d = {
        box_fill = newDrawing("Square"), box_line = newDrawing("Square"),
        c_tl_h = newDrawing("Line"), c_tl_v = newDrawing("Line"),
        c_tr_h = newDrawing("Line"), c_tr_v = newDrawing("Line"),
        c_bl_h = newDrawing("Line"), c_bl_v = newDrawing("Line"),
        c_br_h = newDrawing("Line"), c_br_v = newDrawing("Line"),
        hp_bg = newDrawing("Line"), hp_fill = newDrawing("Line"),
        hp_text = newDrawing("Text"), name = newDrawing("Text"),
        dist = newDrawing("Text"), line = newDrawing("Line"),
    }
    Drawings[plr] = d
    return d
end

local function hideAll(d)
    for _, v in pairs(d) do if v then v.Visible = false end end
end

local function updateESP(plr)
    local d = Drawings[plr]
    if not d then return end
    local char = plr.Character
    if not char then hideAll(d); return end
    local hum  = char:FindFirstChildOfClass("Humanoid")
    local hrp  = char:FindFirstChild("HumanoidRootPart")
    local head = char:FindFirstChild("Head")
    if not (hum and hrp and head) then hideAll(d); return end
    if State.IgnoreDead and hum.Health <= 0 then hideAll(d); return end

    local color = State.ESPColor
    if State.ESPHitCheck then
        local canHit = hasLineOfSight(plr, head)
        color = canHit and State.HitColorOK or State.HitColorNo
    end

    local top, onTop = worldToScreen(head.Position + Vector3.new(0, 0.6, 0))
    local bot, onBot = worldToScreen(hrp.Position - Vector3.new(0, 3.2, 0))
    if not (State.ESPEnabled and onTop and onBot) then hideAll(d); return end

    local h = math.abs(bot.Y - top.Y)
    local w = h * 0.55
    local x = top.X - w / 2
    local y = top.Y

    if d.box_fill then
        d.box_fill.Visible = true
        d.box_fill.Color = color
        d.box_fill.Transparency = 0.82
        d.box_fill.Filled = true
        d.box_fill.From = Vector2.new(x, y)
        d.box_fill.To = Vector2.new(x + w, y + h)
    end
    if d.box_line then
        d.box_line.Visible = true
        d.box_line.Color = color
        d.box_line.Thickness = 1
        d.box_line.Transparency = 1
        d.box_line.Filled = false
        d.box_line.From = Vector2.new(x, y)
        d.box_line.To = Vector2.new(x + w, y + h)
    end

    local len = math.clamp(math.min(w, h) * 0.22, 4, 14)
    local function setLine(line, from, to)
        if not line then return end
        line.Visible = true
        line.Color = color
        line.Thickness = 2
        line.From = from
        line.To = to
    end
    setLine(d.c_tl_h, Vector2.new(x, y), Vector2.new(x + len, y))
    setLine(d.c_tl_v, Vector2.new(x, y), Vector2.new(x, y + len))
    setLine(d.c_tr_h, Vector2.new(x + w, y), Vector2.new(x + w - len, y))
    setLine(d.c_tr_v, Vector2.new(x + w, y), Vector2.new(x + w, y + len))
    setLine(d.c_bl_h, Vector2.new(x, y + h), Vector2.new(x + len, y + h))
    setLine(d.c_bl_v, Vector2.new(x, y + h), Vector2.new(x, y + h - len))
    setLine(d.c_br_h, Vector2.new(x + w, y + h), Vector2.new(x + w - len, y + h))
    setLine(d.c_br_v, Vector2.new(x + w, y + h), Vector2.new(x + w, y + h - len))

    if d.hp_bg and d.hp_fill and d.hp_text then
        if State.ESPHealth then
            local bar_x = x - 6
            local bar_top = y
            local bar_bot = y + h
            local ratio = math.clamp(hum.Health / math.max(hum.MaxHealth, 1), 0, 1)
            local fill_bot = bar_bot - (h * ratio)
            d.hp_bg.Visible = true
            d.hp_bg.Color = Color3.fromRGB(10, 10, 10)
            d.hp_bg.Thickness = 3
            d.hp_bg.From = Vector2.new(bar_x, bar_top)
            d.hp_bg.To = Vector2.new(bar_x, bar_bot)
            d.hp_fill.Visible = true
            d.hp_fill.Color = healthColor(ratio)
            d.hp_fill.Thickness = 3
            d.hp_fill.From = Vector2.new(bar_x, fill_bot)
            d.hp_fill.To = Vector2.new(bar_x, bar_bot)
            d.hp_text.Visible = true
            d.hp_text.Color = Color3.fromRGB(255, 255, 255)
            d.hp_text.Size = 12
            d.hp_text.Center = false
            d.hp_text.Outline = true
            d.hp_text.Text = string.format("%d", math.floor(hum.Health))
            d.hp_text.Position = Vector2.new(x + w + 4, y + h - 14)
        else
            d.hp_bg.Visible = false
            d.hp_fill.Visible = false
            d.hp_text.Visible = false
        end
    end

    if d.name then
        local np = worldToScreen(head.Position + Vector3.new(0, 2.4, 0))
        d.name.Visible = true
        d.name.Color = color
        d.name.Size = 14
        d.name.Center = true
        d.name.Outline = true
        d.name.Text = plr.Name
        d.name.Position = np
    end

    if d.dist then
        if State.ESPDistance then
            local myChar = LocalPlayer.Character
            local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
            local dist = myRoot and (myRoot.Position - hrp.Position).Magnitude or 0
            d.dist.Visible = true
            d.dist.Color = color
            d.dist.Size = 12
            d.dist.Center = true
            d.dist.Outline = true
            d.dist.Text = string.format("[%dm]", math.floor(dist))
            d.dist.Position = Vector2.new((x + x + w) / 2, y + h + 8)
        else
            d.dist.Visible = false
        end
    end

    if d.line then
        if State.ESPLine then
            d.line.Visible = true
            d.line.Color = color
            d.line.Thickness = 1
            d.line.From = (State.ESPLineOrigin == "Bottom")
                and Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                or  Vector2.new(Camera.ViewportSize.X / 2, 0)
            d.line.To = Vector2.new((x + x + w) / 2, y + h / 2)
        else
            d.line.Visible = false
        end
    end
end

RunService.RenderStepped:Connect(function()
    if not State.ESPEnabled then
        for _, d in pairs(Drawings) do hideAll(d) end
        return
    end
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        ensureESP(plr)
        updateESP(plr)
    end
end)

Players.PlayerRemoving:Connect(function(plr)
    local d = Drawings[plr]
    if not d then return end
    for _, v in pairs(d) do pcall(function() v:Remove() end) end
    Drawings[plr] = nil
end)

--=====================================================================
-- UTILS SISTEMA
--=====================================================================
local function resetCharacter()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum and hum.Health > 0 then
        hum.Health = 0
        task.delay(0.15, function()
            if hum.Parent and hum.Health > 0 then
                hum:TakeDamage(hum.MaxHealth)
            end
        end)
    end
    task.delay(0.4, function()
        if char.Parent and hum and hum.Health > 0 then
            pcall(function() LocalPlayer.Character = nil end)
        end
    end)
end

local function queueAutoLoad()
    if not SCRIPT_URL or #SCRIPT_URL == 0 then return false end
    local fn = queue_on_teleport or queueonteleport
    if type(fn) ~= "function" then return false end
    local payload = string.format('loadstring(game:HttpGet("%s"))()', SCRIPT_URL)
    return pcall(fn, payload)
end

local function switchServer()
    queueAutoLoad()
    local okFetch = pcall(function()
        local url = string.format(
            "https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100",
            game.PlaceId
        )
        local raw = game:HttpGet(url)
        local data = HttpService:JSONDecode(raw)
        if not data or not data.data then return end
        for _, srv in ipairs(data.data) do
            if srv.id and srv.playing < srv.maxPlayers and srv.id ~= game.JobId then
                TeleportService:TeleportToPlaceInstance(game.PlaceId, srv.id, LocalPlayer)
                return true
            end
        end
    end)
    if okFetch then return end
    pcall(function() TeleportService:Teleport(game.PlaceId, LocalPlayer) end)
end

--=====================================================================
-- ORION WINDOW
--=====================================================================
local Window = OrionLib:MakeWindow({
    Name = "7zK | @iphz7",
    HidePremium = false,
    SaveConfig = true,
    ConfigFolder = "7zK_Free",
    IntroEnabled = true,
    IntroText = "7zK FREE | @iphz7",
})

--===================== ABA AIMBOT =====================
local AimTab = Window:MakeTab({
    Name = "Aimbot", Icon = "rbxassetid://4483345998", PremiumOnly = false,
})

AimTab:AddSection({ Name = "Aimbot Funções" })

AimTab:AddToggle({
    Name = "Aimbot", Default = false, Flag = "aim_enabled", Save = true,
    Callback = function(v) State.Aimbot = v end,
})

AimTab:AddDropdown({
    Name = "Local da mira", Default = "Cabeça",
    Options = { "Peito", "Cabeça" }, Flag = "aim_part", Save = true,
    Callback = function(v)
        State.AimPart = (v == "Cabeça") and "Head" or "UpperTorso"
    end,
})

AimTab:AddSlider({
    Name = "Regulador de FOV", Min = 1, Max = 440, Default = 140,
    Increment = 1, ValueName = "px", Flag = "aim_fov", Save = true,
    Callback = function(v) State.FOV = tonumber(v) or 140 end,
})

local ctrlShowFov = AimTab:AddToggle({
    Name = "Mostrar fov", Default = false, Flag = "aim_show_fov", Save = true,
    Callback = function(v)
        State.ShowFov = false
        if v then avisoETrava(ctrlShowFov, false) end
    end,
})

local ctrlThrough = AimTab:AddToggle({
    Name = "Grudar através de paredes", Default = false,
    Flag = "aim_through_walls", Save = true,
    Callback = function(v)
        State.ThroughWalls = false
        if v then avisoETrava(ctrlThrough, false) end
    end,
})

AimTab:AddSection({ Name = "Outros" })

AimTab:AddToggle({
    Name = "Ignorar mortos", Default = true,
    Flag = "aim_ignore_dead", Save = true,
    Callback = function(v) State.IgnoreDead = v end,
})

local ctrlSilent = AimTab:AddToggle({
    Name = "Aim Silent", Default = false,
    Flag = "aim_silent", Save = true,
    Callback = function(v)
        State.SilentAim = false
        if v then avisoETrava(ctrlSilent, false) end
    end,
})

local ctrlStick = AimTab:AddToggle({
    Name = "Grudar no alvo mais próximo", Default = false,
    Flag = "aim_stick_closest", Save = true,
    Callback = function(v)
        State.StickClosest = false
        if v then avisoETrava(ctrlStick, false) end
    end,
})

local ctrlStickDist = AimTab:AddSlider({
    Name = "Distância do grudar", Min = 5, Max = 500, Default = 30,
    Increment = 1, ValueName = "m", Flag = "aim_stick_distance", Save = true,
    Callback = function(v)
        if v and v ~= 30 then avisoETrava(ctrlStickDist, 30) end
    end,
})

--===================== ABA ESP =====================
local EspTab = Window:MakeTab({
    Name = "ESP", Icon = "rbxassetid://4483345998", PremiumOnly = false,
})

EspTab:AddSection({ Name = "Cor do ESP" })

EspTab:AddColorpicker({
    Name = "ESP Color", Default = State.ESPColor,
    Flag = "esp_color", Save = true,
    Callback = function(v) State.ESPColor = parseColor(v) end,
})

EspTab:AddSection({ Name = "ESP" })

EspTab:AddToggle({
    Name = "ESP", Default = false, Flag = "esp_enabled", Save = true,
    Callback = function(v) State.ESPEnabled = v end,
})

EspTab:AddToggle({
    Name = "Acerta?", Default = false, Flag = "esp_hit_check", Save = true,
    Callback = function(v) State.ESPHitCheck = v end,
})

local ctrlEspLine = EspTab:AddToggle({
    Name = "ESP Linha", Default = false, Flag = "esp_line", Save = true,
    Callback = function(v)
        State.ESPLine = false
        if v then avisoETrava(ctrlEspLine, false) end
    end,
})

local ctrlEspLineOrigin = EspTab:AddDropdown({
    Name = "Origem da linha", Default = "Baixo",
    Options = { "Baixo", "Cima" }, Flag = "esp_line_origin", Save = true,
    Callback = function(v)
        State.ESPLineOrigin = "Bottom"
        if v and v ~= "Baixo" then avisoETrava(ctrlEspLineOrigin, "Baixo") end
    end,
})

local ctrlEspHealth = EspTab:AddToggle({
    Name = "ESP Vida", Default = false, Flag = "esp_health", Save = true,
    Callback = function(v)
        State.ESPHealth = false
        if v then avisoETrava(ctrlEspHealth, false) end
    end,
})

local ctrlEspDist = EspTab:AddToggle({
    Name = "ESP Distância", Default = false, Flag = "esp_dist", Save = true,
    Callback = function(v)
        State.ESPDistance = false
        if v then avisoETrava(ctrlEspDist, false) end
    end,
})

--===================== ABA ADICIONAIS =====================
local ExtraTab = Window:MakeTab({
    Name = "Adicionais", Icon = "rbxassetid://4483345998", PremiumOnly = false,
})

ExtraTab:AddSection({ Name = "Segurança" })

local ctrlAntiAdmin = ExtraTab:AddToggle({
    Name = "Auto-Kick em Admin", Default = false,
    Flag = "anti_admin", Save = true,
    Callback = function(v)
        State.AntiAdmin = false
        if v then avisoETrava(ctrlAntiAdmin, false) end
    end,
})

local ctrlAntiAFK = ExtraTab:AddToggle({
    Name = "Anti-AFK", Default = false,
    Flag = "anti_afk", Save = true,
    Callback = function(v)
        State.AntiAFK = false
        if v then avisoETrava(ctrlAntiAFK, false) end
    end,
})

local ctrlAutoHop = ExtraTab:AddToggle({
    Name = "Trocar de Server a cada 10min", Default = false,
    Flag = "auto_server_hop", Save = true,
    Callback = function(v)
        State.AutoServerHop = false
        if v then avisoETrava(ctrlAutoHop, false) end
    end,
})

ExtraTab:AddSection({ Name = "Visual" })

local ctrlNoFog = ExtraTab:AddToggle({
    Name = "No Fog", Default = false, Flag = "no_fog", Save = true,
    Callback = function(v)
        State.NoFog = false
        applyNoFog()
        if v then avisoETrava(ctrlNoFog, false) end
    end,
})

local ctrlNoShake = ExtraTab:AddToggle({
    Name = "Camera Shake Off", Default = false, Flag = "no_cam_shake", Save = true,
    Callback = function(v)
        State.NoCamShake = false
        if v then avisoETrava(ctrlNoShake, false) end
    end,
})

local ctrlNoEffects = ExtraTab:AddToggle({
    Name = "Remove Effects", Default = false, Flag = "no_effects", Save = true,
    Callback = function(v)
        State.NoEffects = false
        applyNoEffects()
        if v then avisoETrava(ctrlNoEffects, false) end
    end,
})

local ctrlTimeOfDay = ExtraTab:AddSlider({
    Name = "Time of Day", Min = 0, Max = 24, Default = 12,
    Increment = 1, ValueName = "h", Flag = "time_of_day", Save = true,
    Callback = function(v)
        State.TimeOfDay = 12
        if v and v ~= 12 then avisoETrava(ctrlTimeOfDay, 12) end
    end,
})

ExtraTab:AddSection({ Name = "Câmera" })

local ctrlThirdPerson = ExtraTab:AddToggle({
    Name = "3 Pessoa", Default = false, Flag = "third_person", Save = true,
    Callback = function(v)
        State.ThirdPerson = false
        stopThirdPerson()
        if v then avisoETrava(ctrlThirdPerson, false) end
    end,
})

ExtraTab:AddSection({ Name = "Spin Bot" })

local ctrlSpinbot = ExtraTab:AddToggle({
    Name = "Spin Bot", Default = false, Flag = "spinbot_enabled", Save = true,
    Callback = function(v)
        State.Spinbot = false
        if v then avisoETrava(ctrlSpinbot, false) end
    end,
})

local ctrlSpinSpeed = ExtraTab:AddSlider({
    Name = "Regular velocidade do spin bot", Min = 1, Max = 75, Default = 30,
    Increment = 1, ValueName = "", Flag = "spinbot_speed", Save = true,
    Callback = function(v)
        if v and v ~= 30 then avisoETrava(ctrlSpinSpeed, 30) end
    end,
})

ExtraTab:AddSection({ Name = "Discord" })

ExtraTab:AddButton({
    Name = "Copiar link do Discord",
    Callback = function()
        if setclipboard then
            pcall(setclipboard, "https://discord.gg/hnWmz64FR")
        end
    end,
})

ExtraTab:AddButton({
    Name = "7zK Creator",
    Callback = function()
        if setclipboard then
            pcall(setclipboard, "PgKkj7")
        end
    end,
})

ExtraTab:AddSection({ Name = "Sistema" })

ExtraTab:AddButton({
    Name = "Reset Personagem",
    Callback = function() resetCharacter() end,
})

ExtraTab:AddButton({
    Name = "Trocar de Servidor",
    Callback = function() switchServer() end,
})

ExtraTab:AddToggle({
    Name = "Auto-Load Script", Default = false, Flag = "autoload", Save = true,
    Callback = function(v)
        State.AutoLoad = v
        if v then queueAutoLoad() end
    end,
})

--===================== ABA OUTROS - EXTRAS =====================
local ExtrasTab = Window:MakeTab({
    Name = "Outros - Extras",
    Icon = "rbxassetid://4483345998",
    PremiumOnly = false,
})

-- (vazia — pronta pra futuras funções)

OrionLib:Init()
