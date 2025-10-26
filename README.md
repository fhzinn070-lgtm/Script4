--[[ 
FHZINN HUB - Versão otimizada e compatível (mais leve / executor-agnóstico)
Mantive todas as funções originais e adicionei um UI fallback leve caso a
biblioteca Rayfield não possa ser carregada no executor do usuário.
Autor original do script: discordfernando32-byte
Optimizado por: copilot (assistente)
--]]

-- UTIL: safe pcall wrappers para funções específicas de executores
local function try(func, ...)
    local ok, res = pcall(func, ...)
    if ok then return res end
    return nil
end

-- DETECTORES DE AMBIENTE COMUM DE EXECUTORES
local HttpGet = nil
if type(syn) == "table" and syn.request then
    HttpGet = function(url)
        local ok, res = pcall(function()
            local r = syn.request({Url = url, Method = "GET"})
            return r and r.Body or nil
        end)
        return ok and res or nil
    end
elseif type(http_request) == "function" then
    HttpGet = function(url)
        local ok, res = pcall(function()
            local r = http_request({Url = url, Method = "GET"})
            return r and r.Body or nil
        end)
        return ok and res or nil
    end
elseif type(game) == "table" and type(game.HttpGet) == "function" then
    HttpGet = function(url)
        local ok, res = pcall(function() return game:HttpGet(url) end)
        return ok and res or nil
    end
else
    HttpGet = function() return nil end
end

-- fallback loadstring / load
local function safeLoad(code, chunkname)
    if not code then return nil end
    if type(loadstring) == "function" then
        local ok, f = pcall(loadstring, code)
        if ok then return f end
    end
    local ok, f = pcall(load, code, chunkname or "FHZINN")
    if ok then return f end
    return nil
end

-- SAFE METATABLE / NAMECALL HOOK (tenta ser universal, falha silenciosa)
local function tryProtectNamecall()
    -- prefer hookmetamethod if available
    local ok, hook = pcall(function() return hookmetamethod end)
    if ok and type(hookmetamethod) == "function" then
        -- Use hookmetamethod to hook __namecall
        pcall(function()
            local old = hookmetamethod(game, "__namecall", function(self, ...)
                local method = getnamecallmethod and getnamecallmethod() or ""
                if method and (method:lower() == "kick" or method:lower() == "remove") then
                    -- bloqueia
                    return nil
                end
                if self:IsA and self:IsA("RemoteEvent") and tostring(self.Name):lower():find("ban") then
                    return nil
                end
                return old(self, ...)
            end)
        end)
        return
    end

    -- fallback: try raw metatable override (synapse/krnl style)
    local ok2, mt = pcall(function() return getrawmetatable(game) end)
    if ok2 and mt and type(mt.__namecall) == "function" then
        pcall(function()
            local setreadonly = setreadonly or make_writeable or function() end
            setreadonly(mt, false)
            local oldNamecall = mt.__namecall
            mt.__namecall = newcclosure and newcclosure(function(self, ...)
                local method = (getnamecallmethod and getnamecallmethod() or "")
                if method and (method:lower() == "kick" or method:lower() == "remove") then
                    return nil
                end
                if self:IsA and self:IsA("RemoteEvent") and tostring(self.Name):lower():find("ban") then
                    return nil
                end
                return oldNamecall(self, ...)
            end) or function(self, ...) return oldNamecall(self, ...) end
            setreadonly(mt, true)
        end)
        return
    end
    -- else, couldn't hook; continue silently
end

-- APLICAR PROTEÇÃO ANTI-KICK/ANTI-BAN (tenta várias estratégias)
tryProtectNamecall()

-- Tenta sobrescrever métodos no LocalPlayer com pcall (silencioso)
local Players = game:GetService("Players")
local lp = Players.LocalPlayer
if lp then
    pcall(function() lp.Kick = function() return nil end end)
    pcall(function() lp.kick = function() return nil end end)
    pcall(function() lp.Remove = function() return nil end end)
    pcall(function() lp.remove = function() return nil end end)
end

-- Safe value setters (para evitar valores extremos e anti-ban)
local function safeSetWalkSpeed(hum, speed)
    if not hum then return end
    speed = tonumber(speed) or 16
    speed = math.clamp(speed, 16, 120)
    pcall(function() hum.WalkSpeed = speed end)
end

local function safeSetJump(hum, jump)
    if not hum then return end
    jump = tonumber(jump) or 35
    jump = math.clamp(jump, 35, 100)
    -- alguns jogos usam JumpPower, outros usam JumpHeight/JumpPower
    pcall(function() hum.JumpPower = jump end)
    if hum and hum:IsA and hum:IsA("Humanoid") and hum.JumpHeight then
        pcall(function() hum.JumpHeight = jump end)
    end
end

-- Espera pelo carregamento do jogo (leve)
if not game:IsLoaded() then
    game.Loaded:Wait()
end

-- UI: tenta carregar Rayfield, se falhar, cria um fallback leve que expõe as mesmas chamadas usadas.
local UI = nil
do
    local rayfield_code = nil
    local success = false
    -- Tenta obter Rayfield (URL original do script)
    local url = "https://sirius.menu/rayfield"
    local body = HttpGet(url)
    if body then
        local f = safeLoad(body, "Rayfield")
        if f then
            local ok, res = pcall(f)
            if ok and type(res) == "table" then
                UI = res
                success = true
            elseif ok and type(_G) == "table" and type(res) == "function" then
                -- às vezes a lib retorna uma função que cria a UI
                local ok2, w = pcall(res)
                if ok2 and w then UI = w success = true end
            end
        end
    end

    -- fallback leve: cria um objeto minimal que fornece as chamadas usadas:
    if not success then
        -- Cria um ScreenGui com funções simples para tabs, sections e controles básicos.
        local GuiService = {}
        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Name = "FHZINN_HUB_FallbackUI"
        ScreenGui.ResetOnSpawn = false
        ScreenGui.Parent = Players.LocalPlayer:WaitForChild("PlayerGui")

        local baseX = 10
        local baseY = 10
        local currentY = 10
        local windows = {}

        local function createLabel(parent, txt, posY)
            local lbl = Instance.new("TextLabel")
            lbl.Size = UDim2.new(0, 300, 0, 26)
            lbl.Position = UDim2.new(0, baseX, 0, posY)
            lbl.Text = txt
            lbl.TextColor3 = Color3.fromRGB(255,255,255)
            lbl.BackgroundTransparency = 0.3
            lbl.BackgroundColor3 = Color3.fromRGB(30,30,30)
            lbl.Font = Enum.Font.SourceSans
            lbl.TextSize = 14
            lbl.Parent = parent
            return lbl
        end

        local function createButton(parent, txt, posY, cb)
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0, 300, 0, 30)
            btn.Position = UDim2.new(0, baseX, 0, posY)
            btn.Text = txt
            btn.TextColor3 = Color3.fromRGB(255,255,255)
            btn.BackgroundColor3 = Color3.fromRGB(60,60,60)
            btn.Font = Enum.Font.SourceSans
            btn.TextSize = 14
            btn.Parent = parent
            btn.MouseButton1Click:Connect(function() pcall(cb) end)
            return btn
        end

        local function createToggle(parent, txt, posY, cb, default)
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0, 300, 0, 30)
            btn.Position = UDim2.new(0, baseX, 0, posY)
            btn.Text = txt .. " [OFF]"
            btn.TextColor3 = Color3.fromRGB(255,255,255)
            btn.BackgroundColor3 = Color3.fromRGB(60,60,60)
            btn.Font = Enum.Font.SourceSans
            btn.TextSize = 14
            btn.Parent = parent
            local state = default or false
            btn.MouseButton1Click:Connect(function()
                state = not state
                btn.Text = txt .. (state and " [ON]" or " [OFF]")
                pcall(cb, state)
            end)
            return {
                _btn = btn,
                Set = function(_, v) state = v; btn.Text = txt .. (state and " [ON]" or " [OFF]") end,
                Get = function() return state end,
                CurrentValue = default or false
            }
        end

        local function createSlider(parent, txt, posY, minv, maxv, default, cb)
            local val = default or minv
            local lbl = Instance.new("TextLabel")
            lbl.Size = UDim2.new(0, 300, 0, 26)
            lbl.Position = UDim2.new(0, baseX, 0, posY)
            lbl.Text = txt .. " : " .. tostring(val)
            lbl.TextColor3 = Color3.fromRGB(255,255,255)
            lbl.BackgroundTransparency = 0.3
            lbl.BackgroundColor3 = Color3.fromRGB(30,30,30)
            lbl.Font = Enum.Font.SourceSans
            lbl.TextSize = 14
            lbl.Parent = parent

            -- clique para aumentar (simplicidade)
            lbl.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 then
                    val = math.clamp(val + 1, minv, maxv)
                    lbl.Text = txt .. " : " .. tostring(val)
                    pcall(cb, val)
                elseif input.UserInputType == Enum.UserInputType.MouseButton2 then
                    val = math.clamp(val - 1, minv, maxv)
                    lbl.Text = txt .. " : " .. tostring(val)
                    pcall(cb, val)
                end
            end)
            return {
                Set = function(_, v) val = v lbl.Text = txt .. " : " .. tostring(v) end,
                CurrentValue = val
            }
        end

        -- Estruturas compatíveis com chamadas do script original:
        local Window = {}
        function Window:CreateTab(name, icon)
            local frame = Instance.new("Frame")
            frame.Size = UDim2.new(0, 320, 0, 720)
            frame.Position = UDim2.new(0, baseX, 0, currentY)
            frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
            frame.Parent = ScreenGui
            currentY = currentY + 10
            local tab = {frame = frame, sections = {}}
            function tab:CreateSection(title)
                local secLabel = createLabel(frame, title, #frame:GetChildren()*30)
                local sec = {frame = frame}
                table.insert(self.sections, sec)
                return sec
            end
            function tab:CreateButton(opts)
                local fn = opts.Callback or function() end
                createButton(frame, opts.Name or "Button", 10 + #frame:GetChildren()*32, fn)
                return true
            end
            function tab:CreateToggle(opts)
                local cb = opts.Callback or function() end
                local t = createToggle(frame, opts.Name or "Toggle", 10 + #frame:GetChildren()*32, cb, opts.CurrentValue)
                t.CurrentValue = opts.CurrentValue
                return t
            end
            function tab:CreateSlider(opts)
                local cb = opts.Callback or function() end
                local s = createSlider(frame, opts.Name or "Slider", 10 + #frame:GetChildren()*32, opts.Range[1], opts.Range[2], opts.CurrentValue or opts.Range[1], cb)
                return s
            end
            function tab:CreateInput(opts)
                local input = Instance.new("TextBox")
                input.Size = UDim2.new(0, 300, 0, 30)
                input.Position = UDim2.new(0, baseX, 0, 10 + #frame:GetChildren()*32)
                input.PlaceholderText = opts.PlaceholderText or ""
                input.Text = ""
                input.Parent = frame
                input.FocusLost:Connect(function(enter)
                    if enter then pcall(opts.Callback, input.Text) end
                end)
                return input
            end
            function tab:CreateDropdown(opts)
                local dd = Instance.new("TextButton")
                dd.Size = UDim2.new(0, 300, 0, 30)
                dd.Position = UDim2.new(0, baseX, 0, 10 + #frame:GetChildren()*32)
                dd.Text = opts.CurrentOption or opts.Options[1] or "Select"
                dd.Parent = frame
                dd.MouseButton1Click:Connect(function()
                    local new = opts.Options[1] -- fallback
                    for i,v in ipairs(opts.Options) do
                        new = v; break
                    end
                    dd.Text = new
                    pcall(opts.Callback, new)
                end)
                return dd
            end
            function tab:CreateKeybind(opts)
                -- simples: botão que aguarda tecla
                local btn = createButton(frame, opts.Name .. " [Key:" .. (opts.CurrentKeybind or "F") .. "]", 10 + #frame:GetChildren()*32, function()
                    -- nada sofisticado no fallback
                    pcall(opts.Callback, opts.CurrentKeybind or "F")
                end)
                return btn
            end
            function tab:CreateParagraph(opts)
                createLabel(frame, opts.Title .. ": " .. tostring(opts.Content), 10 + #frame:GetChildren()*32)
                return true
            end
            return tab
        end

        function Window:CreateSection() end
        function Window:LoadConfiguration() end

        UI = Window
    end
end

-- Criando janela principal (usando UI carregada)
local Window = nil
if type(UI) == "table" and UI.CreateWindow then
    Window = UI:CreateWindow({
       Name = "FHZ HUB",
       Icon = nil,
       LoadingTitle = "Carregando o melhor script ",
       LoadingSubtitle = "by script",
       Theme = "Default",
       ToggleUIKeybind = "K",
       DisableRayfieldPrompts = false,
       DisableBuildWarnings = false,
       ConfigurationSaving = { Enabled = true, FolderName = nil, FileName = "FHZINN HUB" },
       Discord = { Enabled = true, Invite = "JF2F2RANud", RememberJoins = true },
       KeySystem = true,
       KeySettings = { Title = "Aura Keys", Subtitle = "Key System", Note = "Para conseguir a key, entre no discord da Aura", FileName = "Key", SaveKey = true, GrabKeyFromSite = false, Key = {"Aura", "Ore", "AuraHub"} }
    })
else
    -- se a UI for o fallback, ela já retornou um objeto compatível
    Window = UI
end

-- Criar tabs e seções (compatível com Rayfield e fallback)
local AuraHub = Window:CreateTab and Window:CreateTab("Aura Hub", 4483362458) or Window.CreateTab("Aura Hub", 4483362458)
local Farm = Window:CreateTab and Window:CreateTab("Farm", 4483362458) or Window.CreateTab("Farm", 4483362458)
local SectionMain = (AuraHub.CreateSection and AuraHub:CreateSection("Funções Principais")) or (AuraHub.CreateSection and AuraHub:CreateSection("Funções Principais")) or nil

----------------- UNIVERSAL PLAYER UTILITY -----------------
local function getPlayer()
    return Players.LocalPlayer
end

local function getCharacter()
    local player = getPlayer()
    return player and player.Character or player.CharacterAdded:Wait()
end

local function getHumanoid()
    local char = try(getCharacter)
    return char and char:FindFirstChildOfClass("Humanoid") or nil
end

local function getHumanoidRootPart()
    local char = try(getCharacter)
    return char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")) or nil
end

local function getPlayerGui()
    local p = getPlayer()
    if not p then return nil end
    return p:FindFirstChildOfClass("PlayerGui") or p:WaitForChild("PlayerGui")
end

------------------------------------------------------------
-- Funções AuraHub (Mantidas e levemente otimizadas)
------------------------------------------------------------

local function MatarJogador()
    local humanoid = getHumanoid()
    if humanoid then
        pcall(function() humanoid.Health = 0 end)
    end
end

if Farm and Farm.CreateButton then
    Farm:CreateButton({ Name = "Matar Jogador", Callback = MatarJogador })
else
    -- fallback minimal: try criar um botão simples se possível
    pcall(function() if AuraHub and AuraHub.CreateButton then AuraHub:CreateButton({ Name = "Matar Jogador", Callback = MatarJogador }) end end)
end

-- Flash (WalkSpeed)
local FlashSpeed = 99
local NormalSpeed = 57

local function SetFlashSpeed(newSpeed)
    FlashSpeed = newSpeed or FlashSpeed
    local humanoid = getHumanoid()
    if humanoid and ToggleFlash and ToggleFlash.CurrentValue then
        safeSetWalkSpeed(humanoid, FlashSpeed)
    end
end

local ToggleFlash = AuraHub:CreateToggle and AuraHub:CreateToggle({
    Name = "Flash",
    CurrentValue = false,
    Callback = function(Value)
        local humanoid = getHumanoid()
        if humanoid then
            safeSetWalkSpeed(humanoid, Value and FlashSpeed or NormalSpeed)
        end
    end
}) or { CurrentValue = false }

local SliderFlashSpeed = AuraHub.CreateSlider and AuraHub:CreateSlider({
    Name = "Velocidade do Flash",
    Range = {20, 120},
    Increment = 1,
    Suffix = " Speed",
    CurrentValue = FlashSpeed,
    Callback = SetFlashSpeed
}) or nil

-- Invisibilidade
local ToggleInvisible = AuraHub.CreateToggle and AuraHub:CreateToggle({
    Name = "Invisible",
    CurrentValue = false,
    Callback = function(Value)
        local char = try(getCharacter)
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                    pcall(function() part.Transparency = Value and 1 or 0 end)
                    for _, decal in pairs(part:GetChildren()) do
                        if decal:IsA("Decal") then
                            pcall(function() decal.Transparency = Value and 1 or 0 end)
                        end
                    end
                end
            end
        end
    end
}) or { CurrentValue = false }

-- Infinite Jump (uso leve de conexão)
local InfiniteJumpConnection

local ToggleInfiniteJump = AuraHub.CreateToggle and AuraHub:CreateToggle({
    Name = "Infinite Jump",
    CurrentValue = false,
    Callback = function(Value)
        if Value then
            InfiniteJumpConnection = game:GetService("UserInputService").JumpRequest:Connect(function()
                local humanoid = getHumanoid()
                if humanoid then
                    pcall(function() humanoid:ChangeState(Enum.HumanoidStateType.Jumping) end)
                end
            end)
        else
            if InfiniteJumpConnection then
                InfiniteJumpConnection:Disconnect()
                InfiniteJumpConnection = nil
            end
        end
    end
}) or { CurrentValue = false }

-- Imortalidade (God Mode)
local GodModeCoroutine
local ToggleGodMode = AuraHub.CreateToggle and AuraHub:CreateToggle({
    Name = "God Mode",
    CurrentValue = false,
    Callback = function(Value)
        local humanoid = getHumanoid()
        if humanoid then
            if Value then
                if not GodModeCoroutine then
                    GodModeCoroutine = coroutine.create(function()
                        while ToggleGodMode.CurrentValue and humanoid.Parent do
                            if humanoid.Health < humanoid.MaxHealth then
                                pcall(function() humanoid.Health = humanoid.MaxHealth end)
                            end
                            task.wait(0.1)
                        end
                    end)
                    coroutine.resume(GodModeCoroutine)
                end
            else
                GodModeCoroutine = nil
            end
        end
    end
}) or { CurrentValue = false }

-- Auto Clicker (leve)
local AutoClickerActive = false
local AutoClickerCoroutine
local ToggleAutoClicker = AuraHub.CreateToggle and AuraHub:CreateToggle({
    Name = "Auto Clicker",
    CurrentValue = false,
    Callback = function(Value)
        AutoClickerActive = Value
        local VirtualUser = game:GetService("VirtualUser")
        if Value then
            if not AutoClickerCoroutine then
                AutoClickerCoroutine = coroutine.create(function()
                    while AutoClickerActive do
                        pcall(function()
                            VirtualUser:Button1Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                            task.wait(0.08)
                            VirtualUser:Button1Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
                        end)
                        task.wait(0.1)
                    end
                end)
                coroutine.resume(AutoClickerCoroutine)
            end
        else
            AutoClickerCoroutine = nil
        end
    end
}) or { CurrentValue = false }

--------------------------------------------------------------------------------
-- Fly Script Universal MOBILE - Toggle ON/OFF (mantive funcionalidade)
--------------------------------------------------------------------------------

local flyGuiInstance = nil
local flyLoopConnection = nil

local function ActivateFly()
    if flyGuiInstance then
        pcall(function() flyGuiInstance:Destroy() end)
    end

    local camera = workspace.CurrentCamera
    local flying = false
    local speed = 5
    local BV, BG
    local directions = {forward = 0, backward = 0}

    local function getHRP()
        return getHumanoidRootPart()
    end

    flyGuiInstance = Instance.new("ScreenGui")
    flyGuiInstance.Name = "FlyMobileGUI"
    flyGuiInstance.Parent = getPlayerGui()

    local function createFlyButton(txt, pos, color, callback)
        local btn = Instance.new("TextButton")
        btn.Text = txt
        btn.Size = UDim2.new(0, 160, 0, 40)
        btn.Position = pos
        btn.BackgroundColor3 = color
        btn.TextColor3 = Color3.new(1,1,1)
        btn.Font = Enum.Font.SourceSans
        btn.TextSize = 20
        btn.BorderSizePixel = 1
        btn.AutoButtonColor = true
        btn.Parent = flyGuiInstance
        btn.MouseButton1Click:Connect(function() pcall(callback) end)
        return btn
    end

    local btnMais = createFlyButton("+ Velocidade", UDim2.new(0, 15, 0, 120), Color3.fromRGB(0, 170, 0), function()
        speed = math.clamp(speed + 1, 1, 50)
        btnMais.Text = "+ Velocidade (" .. speed .. ")"
    end)

    local btnMenos = createFlyButton("- Velocidade", UDim2.new(0, 15, 0, 165), Color3.fromRGB(180, 0, 0), function()
        speed = math.clamp(speed - 1, 1, 50)
        btnMais.Text = "+ Velocidade (" .. speed .. ")"
    end)

    local btnVoar = createFlyButton("VOAR", UDim2.new(0, 15, 0, 210), Color3.fromRGB(0, 110, 230), function()
        if flying then return end
        flying = true
        local HRP = getHRP()
        if HRP then
            BV = Instance.new("BodyVelocity")
            BV.MaxForce = Vector3.new(1e5, 1e5, 1e5)
            BV.Velocity = Vector3.new()
            BV.Parent = HRP

            BG = Instance.new("BodyGyro")
            BG.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
            BG.P = 1e4
            BG.CFrame = camera.CFrame
            BG.Parent = HRP
        end
    end)

    local btnUnfly = createFlyButton("PARAR DE VOAR", UDim2.new(0, 15, 0, 255), Color3.fromRGB(140, 100, 200), function()
        flying = false
        if BV then pcall(function() BV:Destroy() end) BV = nil end
        if BG then pcall(function() BG:Destroy() end) BG = nil end
    end)

    local btnFrente = Instance.new("TextButton")
    btnFrente.Text = "▶"
    btnFrente.Size = UDim2.new(0, 55, 0, 55)
    btnFrente.Position = UDim2.new(0, 185, 0, 120)
    btnFrente.BackgroundColor3 = Color3.fromRGB(255,215,0)
    btnFrente.TextColor3 = Color3.new(1,1,1)
    btnFrente.Font = Enum.Font.SourceSansBold
    btnFrente.TextSize = 36
    btnFrente.BorderSizePixel = 1
    btnFrente.AutoButtonColor = true
    btnFrente.Parent = flyGuiInstance

    btnFrente.MouseButton1Down:Connect(function() directions.forward = 1 end)
    btnFrente.MouseButton1Up:Connect(function() directions.forward = 0 end)
    -- TouchLongPress pode não existir no fallback; use TouchBegan/Ended
    btnFrente.TouchLongPress and btnFrente.TouchLongPress:Connect(function(_, state) directions.forward = (state == Enum.UserInputState.Begin) and 1 or 0 end)

    local btnTras = Instance.new("TextButton")
    btnTras.Text = "◀"
    btnTras.Size = UDim2.new(0, 55, 0, 55)
    btnTras.Position = UDim2.new(0, 185, 0, 185)
    btnTras.BackgroundColor3 = Color3.fromRGB(255,215,0)
    btnTras.TextColor3 = Color3.new(1,1,1)
    btnTras.Font = Enum.Font.SourceSansBold
    btnTras.TextSize = 36
    btnTras.BorderSizePixel = 1
    btnTras.AutoButtonColor = true
    btnTras.Parent = flyGuiInstance

    btnTras.MouseButton1Down:Connect(function() directions.backward = 1 end)
    btnTras.MouseButton1Up:Connect(function() directions.backward = 0 end)
    btnTras.TouchLongPress and btnTras.TouchLongPress:Connect(function(_, state) directions.backward = (state == Enum.UserInputState.Begin) and 1 or 0 end)

    flyLoopConnection = game:GetService("RunService").RenderStepped:Connect(function()
        if flying and BV and BG then
            local HRP = getHRP()
            if HRP then
                local mov = Vector3.new(directions.forward - directions.backward, 0, 0)
                if mov.Magnitude > 0 then
                    local camCF = camera.CFrame
                    local dir = camCF.LookVector * mov.x
                    BV.Velocity = dir * speed * 16
                else
                    BV.Velocity = Vector3.new()
                end
                BG.CFrame = camera.CFrame
            end
        end
    end)
end

local function DeactivateFly()
    if flyLoopConnection then
        pcall(function() flyLoopConnection:Disconnect() end)
        flyLoopConnection = nil
    end
    if flyGuiInstance then
        pcall(function() flyGuiInstance:Destroy() end)
        flyGuiInstance = nil
    end
end

local ToggleFlyMobile = AuraHub.CreateToggle and AuraHub:CreateToggle({
    Name = "Fly Mobile",
    CurrentValue = false,
    Callback = function(Value)
        if Value then ActivateFly() else DeactivateFly() end
    end
}) or { CurrentValue = false }

--------------------------------------------------------------------------------
-- Teleport Marker Script (GUI separada, Toggle ON/OFF)
--------------------------------------------------------------------------------

local tpGuiInstance = nil

local function ActivateTeleportMarker()
    if tpGuiInstance then pcall(function() tpGuiInstance:Destroy() end) end

    local markerPosition = nil

    tpGuiInstance = Instance.new("ScreenGui")
    tpGuiInstance.Name = "TeleportMarkerGUI"
    tpGuiInstance.Parent = getPlayerGui()

    local function createTPButton(text, pos, color, callback)
        local btn = Instance.new("TextButton")
        btn.Text = text
        btn.Size = UDim2.new(0, 180, 0, 40)
        btn.Position = pos
        btn.BackgroundColor3 = color
        btn.TextColor3 = Color3.new(1,1,1)
        btn.Font = Enum.Font.SourceSans
        btn.TextSize = 18
        btn.BorderSizePixel = 1
        btn.Parent = tpGuiInstance
        btn.MouseButton1Click:Connect(function() pcall(callback) end)
        return btn
    end

    local btnMark = createTPButton(
        "Marcar Posição",
        UDim2.new(0, 30, 0, 180),
        Color3.fromRGB(0, 170, 255),
        function()
            local hrp = getHumanoidRootPart()
            if hrp then
                markerPosition = hrp.Position
                btnMark.Text = "Posição Marcada!"
                task.wait(1)
                btnMark.Text = "Marcar Posição"
            end
        end
    )

    local btnGo = createTPButton(
        "Ir à Posição",
        UDim2.new(0, 30, 0, 230),
        Color3.fromRGB(0, 220, 120),
        function()
            local hrp = getHumanoidRootPart()
            if hrp and markerPosition then
                pcall(function() hrp.CFrame = CFrame.new(markerPosition) end)
            else
                btnGo.Text = "Sem posição marcada!"
                task.wait(1)
                btnGo.Text = "Ir à Posição"
            end
        end
    )

    local credits = Instance.new("TextLabel")
    credits.Text = "Teleport Marker by discordfernando32-byte"
    credits.Size = UDim2.new(0, 310, 0, 16)
    credits.Position = UDim2.new(0, 10, 0, 300)
    credits.BackgroundTransparency = 1
    credits.TextColor3 = Color3.fromRGB(180,180,180)
    credits.Font = Enum.Font.SourceSans
    credits.TextSize = 13
    credits.TextXAlignment = Enum.TextXAlignment.Left
    credits.Parent = tpGuiInstance
end

local function DeactivateTeleportMarker()
    if tpGuiInstance then pcall(function() tpGuiInstance:Destroy() end) tpGuiInstance = nil end
end

local ToggleTeleportMarker = AuraHub.CreateToggle and AuraHub:CreateToggle({
    Name = "Teleport Marker",
    CurrentValue = false,
    Callback = function(Value)
        if Value then ActivateTeleportMarker() else DeactivateTeleportMarker() end
    end
}) or { CurrentValue = false }

--------------------------------------------------------------------------------
-- Sliders e Extras (mantidos)
--------------------------------------------------------------------------------

local KillAuraRadius = 15

local function SetKillAuraRadius(Value)
    KillAuraRadius = Value
end

local SliderKillAuraRadius = AuraHub.CreateSlider and AuraHub:CreateSlider({
    Name = "Área da Kill Aura (Raio)",
    Range = {5, 50},
    Increment = 1,
    Suffix = " studs",
    CurrentValue = KillAuraRadius,
    Callback = SetKillAuraRadius
}) or nil

local SliderJump = Farm.CreateSlider and Farm:CreateSlider({
   Name = "Pulo (JumpPower)",
   Range = {35, 100},
   Increment = 1,
   Suffix = "",
   CurrentValue = 50,
   Callback = function(Value)
      local humanoid = getHumanoid()
      if humanoid then
         safeSetJump(humanoid, Value)
      end
   end
}) or nil

local SectionNPC = Farm.CreateSection and Farm:CreateSection("NPCs") or nil

local InputNPC = Farm.CreateInput and Farm:CreateInput({
   Name = "Nome do NPC",
   PlaceholderText = "Digite um NPC...",
   RemoveTextAfterFocusLost = false,
   Callback = function(Text)
      -- aqui você pode adicionar ação para o texto digitado
   end
}) or nil

local DropdownNPC = Farm.CreateDropdown and Farm:CreateDropdown({
   Name = "Selecionar NPC",
   Options = {"Npc 1", "Npc 2", "Npc 3"},
   CurrentOption = "Npc 1",
   Callback = function(Option)
      -- ação para opção selecionada
   end
}) or nil

local SectionExtra = Farm.CreateSection and Farm:CreateSection("Extras") or nil

local KeybindExample = Farm.CreateKeybind and Farm:CreateKeybind({
   Name = "Atalho de Teclado",
   CurrentKeybind = "F",
   HoldToInteract = false,
   Callback = function(Key)
      -- ação ao pressionar a tecla
   end
}) or nil

local ParagraphCreator = Farm.CreateParagraph and Farm:CreateParagraph({
   Title = "Criador",
   Content = "Aura Hub by Aura"
}) or nil

----------------- MENU IMORTALIDADE ABSOLUTA (DRAG + BOTÃO ATIVAR/DESATIVAR) -----------------
local MenuImortalidadeAtivo = false
local MenuImortalidadeGUI = nil

local function CriarMenuImortalidade()
    if MenuImortalidadeGUI then pcall(function() MenuImortalidadeGUI:Destroy() end) MenuImortalidadeGUI = nil end

    local gui = Instance.new("ScreenGui")
    gui.Name = "MenuImortalidade"
    gui.Parent = getPlayerGui()

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 250, 0, 180)
    frame.Position = UDim2.new(0.5, -125, 0.5, -90)
    frame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    frame.Active = true
    frame.Parent = gui

    -- Dragging minimal (mouse and touch)
    local dragging, dragStart, startPos
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
                                      startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)

    local title = Instance.new("TextLabel", frame)
    title.Size = UDim2.new(1, 0, 0, 36)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.Text = "Menu Imortalidade"
    title.BackgroundTransparency = 1
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.Font = Enum.Font.SourceSansBold
    title.TextSize = 20

    local statusLabel = Instance.new("TextLabel", frame)
    statusLabel.Size = UDim2.new(0.9, 0, 0, 20)
    statusLabel.Position = UDim2.new(0.05, 0, 0, 40)
    statusLabel.BackgroundTransparency = 1
    statusLabel.TextColor3 = Color3.fromRGB(150,255,150)
    statusLabel.Text = "Status: Desativado"
    statusLabel.Font = Enum.Font.SourceSans
    statusLabel.TextSize = 14

    local toggleBtn = Instance.new("TextButton", frame)
    toggleBtn.Size = UDim2.new(0.9, 0, 0, 36)
    toggleBtn.Position = UDim2.new(0.05, 0, 0, 70)
    toggleBtn.Text = "Ativar Imortalidade"
    toggleBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
    toggleBtn.TextColor3 = Color3.fromRGB(255,255,255)
    toggleBtn.Font = Enum.Font.SourceSans
    toggleBtn.TextSize = 16

    local closeBtn = Instance.new("TextButton", frame)
    closeBtn.Size = UDim2.new(0.9, 0, 0, 30)
    closeBtn.Position = UDim2.new(0.05, 0, 0, 120)
    closeBtn.Text = "Fechar Menu"
    closeBtn.BackgroundColor3 = Color3.fromRGB(100, 0, 0)
    closeBtn.TextColor3 = Color3.fromRGB(255,255,255)
    closeBtn.Font = Enum.Font.SourceSans
    closeBtn.TextSize = 14

    local immortalActive = false
    local heartbeatConnection
    local humConnections = {}

    local function protectHumanoid()
        local humanoid = getHumanoid()
        local char = try(getCharacter)
        if not humanoid or not char then return end
        for _,conn in pairs(humConnections) do pcall(function() conn:Disconnect() end) end
        humConnections = {}
        pcall(function() humanoid.Health = humanoid.MaxHealth end)
        table.insert(humConnections, humanoid:GetPropertyChangedSignal("Health"):Connect(function()
            if immortalActive and humanoid.Health < humanoid.MaxHealth then
                pcall(function() humanoid.Health = humanoid.MaxHealth end)
            end
        end))
        table.insert(humConnections, humanoid.StateChanged:Connect(function(_,state)
            if immortalActive and state == Enum.HumanoidStateType.Dead then
                pcall(function() humanoid.Health = humanoid.MaxHealth end)
                pcall(function() humanoid:ChangeState(Enum.HumanoidStateType.Physics) end)
            end
        end))
        table.insert(humConnections, humanoid.AncestryChanged:Connect(function(_,parent)
            if immortalActive and not parent then
                -- tenta esperar e proteger novamente
                pcall(function() getPlayer().CharacterAdded:Wait(); protectHumanoid() end)
            end
        end))
    end

    local function setImmortal(active)
        immortalActive = active
        if active then
            statusLabel.Text = "Status: Ativado"
            statusLabel.TextColor3 = Color3.fromRGB(150,255,150)
            toggleBtn.Text = "Desativar Imortalidade"
            game.Players.LocalPlayer.CharacterAdded:Connect(function() protectHumanoid() end)
            if getCharacter() then protectHumanoid() end
            if heartbeatConnection then pcall(function() heartbeatConnection:Disconnect() end) end
            heartbeatConnection = game:GetService("RunService").Heartbeat:Connect(function()
                if not immortalActive then return end
                local hrp = getHumanoidRootPart()
                if hrp and hrp.Position.Y < -50 then
                    pcall(function() hrp.CFrame = CFrame.new(0, 10, 0) end)
                end
            end)
        else
            statusLabel.Text = "Status: Desativado"
            statusLabel.TextColor3 = Color3.fromRGB(255,150,150)
            toggleBtn.Text = "Ativar Imortalidade"
            if heartbeatConnection then pcall(function() heartbeatConnection:Disconnect() end) heartbeatConnection = nil end
            for _,conn in pairs(humConnections) do pcall(function() conn:Disconnect() end) end
            humConnections = {}
        end
    end

    toggleBtn.MouseButton1Click:Connect(function()
        setImmortal(not immortalActive)
    end)

    closeBtn.MouseButton1Click:Connect(function()
        pcall(function() gui:Destroy() end)
        MenuImortalidadeGUI = nil
        MenuImortalidadeAtivo = false
    end)

    setImmortal(false)

    MenuImortalidadeGUI = gui
end

local ButtonImortalidadeMenu = AuraHub.CreateButton and AuraHub:CreateButton({
    Name = "Menu Imortalidade Absoluta",
    Callback = function()
        if not MenuImortalidadeAtivo then
            CriarMenuImortalidade()
            MenuImortalidadeAtivo = true
        else
            if MenuImortalidadeGUI then 
                pcall(function() MenuImortalidadeGUI:Destroy() end)
                MenuImortalidadeGUI = nil
            end
            MenuImortalidadeAtivo = false
        end
    end
}) or pcall(function() if AuraHub and AuraHub.CreateButton then AuraHub:CreateButton({ Name = "Menu Imortalidade Absoluta", Callback = function() if not MenuImortalidadeAtivo then CriarMenuImortalidade() MenuImortalidadeAtivo = true else if MenuImortalidadeGUI then pcall(function() MenuImortalidadeGUI:Destroy() end) MenuImortalidadeGUI = nil end MenuImortalidadeAtivo = false end end }) end end)

-- Tenta carregar configuração se disponível
if Window and Window.LoadConfiguration then
    pcall(function() Window:LoadConfiguration() end)
end

-- FIM DO SCRIPT
-- Observações:
-- 1) Mantive todas as funções originais e adicionei verificações pcall para tornar o script mais resiliente.
-- 2) Se a biblioteca Rayfield não puder ser carregada (por rede ou limitações do executor),
--    um UI fallback minimal será usado para garantir que as funcionalidades continuem acessíveis.
-- 3) Se quiser que eu adicione novas funções (você mencionou "adicione mais essas que estou pedindo")
--    por favor descreva quais funcionalidades extras deseja que eu inclua (ex.: Speed toggle global, noclip, ESP, auto-collect, etc).
-- 4) Posso também compactar ainda mais o script, remover comentários e otimizar micro-performances se desejar.
