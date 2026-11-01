--[[ Steal a Egg | Mobile + Auto Hop (somente servidores >= 8B ovos/s)
     Executor mobile: Delta / Arceus X / Codex / Fluxus / Hydrogen
     Precisa de loadstring + http ativados. ]]

local Players            = game:GetService("Players")
local TeleportService    = game:GetService("TeleportService")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local HttpService        = game:GetService("HttpService")
local UserInputService   = game:GetService("UserInputService")
local VirtualInputManager= game:GetService("VirtualInputManager")
local TweenService       = game:GetService("TweenService")

local LP = Players.LocalPlayer

local CONFIG = {
    SCRIPT_URL    = "https://flowauth.net/v1/loaders/cacd453cf780cca1885f79c734f492fa.lua",
    MIN_EPS       = 8e9,   -- 8.000.000.000 (piso absoluto)
    AUTO_HOP      = true,
    CHECK_DELAY   = 12,    -- espera os dados do servidor carregarem antes de medir
    HOP_COOLDOWN  = 8,
    MAX_HOPS      = 40,    -- limite por sessao (evita loop infinito)
    DEBUG         = false,
}

local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
local measured, hopping, finished = 0, false, false
local hopCount = 0

--------------------------------------------------------------------
-- 1) Carrega o script original
--------------------------------------------------------------------
pcall(function()
    local src = game:HttpGet(CONFIG.SCRIPT_URL)
    loadstring(src)()
end)

--------------------------------------------------------------------
-- 2) Parser de numeros com sufixo (8.5B, 1.2T, 300M...)
--------------------------------------------------------------------
local MULT = { k = 1e3, m = 1e6, b = 1e9, t = 1e12, qa = 1e15 }

local function parseNum(str)
    if type(str) ~= "string" then str = tostring(str) end
    str = str:lower():gsub("[,%s]", "")
    local num, suf = str:match("([%d%.]+)([kmbtq]?a?)")
    if not num then return 0 end
    return (tonumber(num) or 0) * (MULT[suf] or 1)
end

--------------------------------------------------------------------
-- 3) Leitura do total de ovos/s do servidor
--------------------------------------------------------------------
local function scanContainer(root, found)
    if not root then return end
    for _, d in ipairs(root:GetDescendants()) do
        local n = d.Name:lower()

        if d:IsA("ValueBase") and (n:find("eggpersecond") or n:find("eggspersecond")
            or n:find("eggspersec") or n:find("eggsrate") or n == "eps" or n == "eggrate") then
            found[#found + 1] = parseNum(d.Value)

        elseif d:IsA("TextLabel") or d:IsA("TextButton") then
            local t = d.Text:lower()
            if t:find("egg") and (t:find("per sec") or t:find("/s") or t:find("por seg")) then
                found[#found + 1] = parseNum(t)
            end
        end
    end
end

local function getServerEPS()
    local found = {}

    scanContainer(ReplicatedStorage, found)
    scanContainer(LP:FindFirstChild("PlayerGui"), found)
    scanContainer(workspace, found)

    -- fallback: soma de leaderstats de todos os jogadores
    if #found == 0 then
        for _, plr in ipairs(Players:GetPlayers()) do
            local ls = plr:FindFirstChild("leaderstats")
            if ls then
                for _, v in ipairs(ls:GetChildren()) do
                    if v.Name:lower():find("egg") then
                        found[#found + 1] = parseNum(v.Value)
                    end
                end
            end
        end
    end

    if CONFIG.DEBUG then
        warn("[EPS] amostras: " .. HttpService:JSONEncode(found))
    end

    -- usa o MAIOR valor encontrado (o HUD do servidor costuma ser o total)
    local best = 0
    for _, v in ipairs(found) do
        if v > best then best = v end
    end
    return best
end

--------------------------------------------------------------------
-- 4) Escolha de servidor (lista publica, prioriza cheios)
--------------------------------------------------------------------
local function pickServer()
    local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Desc&limit=100"):format(game.PlaceId)
    local ok, body = pcall(game.HttpGet, game, url, true)
    if not ok or not body then return nil end

    local ok2, data = pcall(HttpService.JSONDecode, HttpService, body)
    if not ok2 or not data or not data.data then return nil end

    local pool = {}
    for _, s in ipairs(data.data) do
        if s.id ~= game.JobId and (s.playing or 0) < (s.maxPlayers or 50) - 2 then
            pool[#pool + 1] = s
        end
    end
    if #pool == 0 then return nil end

    table.sort(pool, function(a, b) return (a.playing or 0) > (b.playing or 0) end)

    local top = math.min(#pool, 10)
    return pool[math.random(1, top)].id
end

local function hop(motivo)
    if hopping or finished then return end
    if hopCount >= CONFIG.MAX_HOPS then
        setStatus("Limite de hops atingido")
        return
    end
    hopping = true
    hopCount = hopCount + 1
    setStatus(("Saindo (%d/%d): %s"):format(hopCount, CONFIG.MAX_HOPS, motivo))

    local jobId = pickServer()
    local ok
    if jobId then
        ok = pcall(TeleportService.TeleportToPlaceInstance, TeleportService, game.PlaceId, jobId, LP)
    end
    if not ok then
        ok = pcall(TeleportService.Teleport, TeleportService, game.PlaceId, LP)
    end
    if not ok then
        setStatus("Falha no teleport, tentando rejoin")
        pcall(function()
            TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LP)
        end)
    end

    task.wait(CONFIG.HOP_COOLDOWN)
    hopping = false
end

--------------------------------------------------------------------
-- 5) UI mobile
--------------------------------------------------------------------
local function fmt(n)
    if n >= 1e12 then return ("%.2fT"):format(n / 1e12) end
    if n >= 1e9  then return ("%.2fB"):format(n / 1e9)  end
    if n >= 1e6  then return ("%.2fM"):format(n / 1e6)  end
    if n >= 1e3  then return ("%.1fK"):format(n / 1e3)  end
    return tostring(math.floor(n))
end

local gui = Instance.new("ScreenGui")
gui.Name = "SAE_Mobile"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.Parent = LP:WaitForChild("PlayerGui")

local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(230, 168)
main.Position = UDim2.new(0, 10, 0.3, 0)
main.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
main.BackgroundTransparency = 0.08
main.BorderSizePixel = 0
main.Active = true
main.Parent = gui
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 12)
Instance.new("UIStroke", main).Color = Color3.fromRGB(80, 80, 100)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 26)
title.BackgroundTransparency = 1
title.Font = Enum.Font.GothamBold
title.TextSize = 13
title.TextColor3 = Color3.fromRGB(235, 235, 245)
title.Text = "Steal a Egg - Mobile | >= 8B/s"
title.Parent = main

local status = Instance.new("TextLabel")
status.Size = UDim2.new(1, -16, 0, 18)
status.Position = UDim2.new(0, 8, 0, 28)
status.BackgroundTransparency = 1
status.Font = Enum.Font.Gotham
status.TextSize = 12
status.TextXAlignment = Enum.TextXAlignment.Left
status.TextColor3 = Color3.fromRGB(150, 220, 150)
status.Text = "Aguardando carregar..."
status.Parent = main

local function setStatus(txt, color)
    status.Text = txt
    status.TextColor3 = color or Color3.fromRGB(150, 220, 150)
end

local function mkButton(txt, y, color, cb)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0.5, -12, 0, 34)
    b.Position = UDim2.new(0, 8, 0, y)
    b.BackgroundColor3 = color
    b.BorderSizePixel = 0
    b.Font = Enum.Font.GothamBold
    b.TextSize = 13
    b.TextColor3 = Color3.fromRGB(255, 255, 255)
    b.Text = txt
    b.AutoButtonColor = true
    b.Parent = main
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 8)
    b.MouseButton1Click:Connect(cb)
    return b
end

local hopBtn = mkButton("Hop agora", 52, Color3.fromRGB(45, 110, 200), function()
    hop("manual")
end)

local autoBtn = mkButton("Auto: ON", 52, Color3.fromRGB(40, 150, 90), function()
    CONFIG.AUTO_HOP = not CONFIG.AUTO_HOP
    autoBtn.Text = CONFIG.AUTO_HOP and "Auto: ON" or "Auto: OFF"
    autoBtn.BackgroundColor3 = CONFIG.AUTO_HOP and Color3.fromRGB(40, 150, 90) or Color3.fromRGB(90, 90, 90)
    if CONFIG.AUTO_HOP then task.spawn(checkServer) end
end)
autoBtn.Position = UDim2.new(0.5, 4, 0, 52)

-- teclas uteis para mobile (simula keybind do script original)
local keysRow = Instance.new("Frame")
keysRow.Size = UDim2.new(1, -16, 0, 34)
keysRow.Position = UDim2.new(0, 8, 0, 92)
keysRow.BackgroundTransparency = 1
keysRow.Parent = main

local function pressKey(key)
    pcall(function()
        VirtualInputManager:SendKeyEvent(true, key, false, game)
        task.wait(0.06)
        VirtualInputManager:SendKeyEvent(false, key, false, game)
    end)
end

for i, key in ipairs({ "E", "F", "Q", "R", "Space" }) do
    local kb = Instance.new("TextButton")
    kb.Size = UDim2.new(0.19, 0, 1, 0)
    kb.Position = UDim2.new((i - 1) * 0.2025, 0, 0, 0)
    kb.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
    kb.BorderSizePixel = 0
    kb.Font = Enum.Font.GothamBold
    kb.TextSize = 12
    kb.TextColor3 = Color3.fromRGB(230, 230, 240)
    kb.Text = (key == "Space") and "Pulo" or key
    kb.Parent = keysRow
    Instance.new("UICorner", kb).CornerRadius = UDim.new(0, 8)
    kb.MouseButton1Click:Connect(function() pressKey(key) end)
end

-- arrastar painel
local dragging, dragStart, startPos
title.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging, dragStart, startPos = true, i.Position, main.Position
        i.Changed:Connect(function()
            if i.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dragging and (i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseMovement) then
        local d = i.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
    end
end)

-- botao minimizar
local min = Instance.new("TextButton")
min.Size = UDim2.fromOffset(26, 26)
min.Position = UDim2.new(1, -30, 0, 2)
min.BackgroundColor3 = Color3.fromRGB(60, 60, 72)
min.BorderSizePixel = 0
min.Font = Enum.Font.GothamBold
min.TextSize = 14
min.TextColor3 = Color3.fromRGB(255, 255, 255)
min.Text = "-"
min.Parent = main
Instance.new("UICorner", min).CornerRadius = UDim.new(0, 8)

local minimized = false
min.MouseButton1Click:Connect(function()
    minimized = not minimized
    TweenService:Create(main, TweenInfo.new(0.2), {
        Size = minimized and UDim2.fromOffset(230, 30) or UDim2.fromOffset(230, 168)
    }):Play()
    for _, c in ipairs(main:GetChildren()) do
        if c ~= title and c ~= min then
            c.Visible = not minimized
        end
    end
end)

--------------------------------------------------------------------
-- 6) Logica de verificacao
--------------------------------------------------------------------
function checkServer()
    if finished then return end
    if not game:IsLoaded() then game.Loaded:Wait() end
    task.wait(CONFIG.CHECK_DELAY)
    if finished then return end

    measured = getServerEPS()
    setStatus(("Ovos/s: %s"):format(fmt(measured)))

    if measured >= CONFIG.MIN_EPS then
        finished = true
        setStatus(("OK - servidor %s/s"):format(fmt(measured)))
        return
    end

    setStatus(("Abaixo de %s/s"):format(fmt(CONFIG.MIN_EPS)), Color3.fromRGB(230, 180, 90))

    if CONFIG.AUTO_HOP then
        hop(("EPS %s < 8B"):format(fmt(measured)))
    end
end

task.spawn(checkServer)# Tu
