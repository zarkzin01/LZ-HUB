--[[
    LZ HUB v1
    - Ré com assistência de drift natural
    - Sem impulso lateral (não joga pro lado)
    - Impulso angular proporcional (controlável)
    - Sem tremor
    - Sem sistema de gravidade
    SHIFT/S = ré + drift | ESPAÇO = drift manual | F1 = painel
--]]

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local LP = Players.LocalPlayer

local cfg = {
    speed = 1,
    force = 120,
    driftPower = 40,
    autoDrift = true,
}

local isRev, isDrift, panelOpen = false, false, true
local curCar, curSeat
local running = false
local loopThread = nil

-- ===== DETECÇÃO DE CARRO =====
local function findCar()
    local char = LP.Character
    if not char or not char.Parent then return nil end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return nil end
    local seat = hum.SeatPart
    if not seat or not seat:IsA("VehicleSeat") then return nil end

    local p = seat.Parent
    while p and p ~= workspace do
        if p:IsA("Model") and (p.PrimaryPart or p:FindFirstChildWhichIsA("BasePart")) then
            return p, seat
        end
        p = p.Parent
    end
    return seat.Parent, seat
end

-- ===== ACHAR EIXO TRASEIRO =====
local function getRearAxle(car)
    if not car then return nil end

    local base = car.PrimaryPart
    if not base then
        for _, p in ipairs(car:GetDescendants()) do
            if p:IsA("BasePart") then base = p break end
        end
    end
    if not base then return nil end

    local parts = {}
    for _, p in ipairs(car:GetDescendants()) do
        if p:IsA("BasePart") then
            table.insert(parts, p)
        end
    end

    -- Por nome
    local rearNames = {"rear", "back", "traseira", "traseiro", "rwheel", "bwheel"}
    local rearParts = {}
    for _, p in ipairs(parts) do
        local n = p.Name:lower()
        for _, rn in ipairs(rearNames) do
            if n:find(rn) then
                table.insert(rearParts, p)
                break
            end
        end
    end
    if #rearParts > 0 then return rearParts end

    -- Por posição (mais atrás)
    local look = base.CFrame.LookVector
    local basePos = base.Position
    local bestScore = -math.huge
    local rearPart = nil

    for _, p in ipairs(parts) do
        local size = p.Size.Magnitude
        if size > 2 and size < 30 then
            local offset = p.Position - basePos
            local score = -(offset:Dot(look))
            if score > bestScore then
                bestScore = score
                rearPart = p
            end
        end
    end

    if rearPart then return {rearPart} end
    return {base}
end

-- ===== LOOP PRINCIPAL =====
local function stopLoop()
    running = false
    if loopThread then
        task.cancel(loopThread)
        loopThread = nil
    end
end

local function startLoop()
    if running then return end
    running = true
    loopThread = task.spawn(function()
        while running do
            local char = LP.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            local seat = hum and hum.SeatPart

            if not seat or not seat:IsA("VehicleSeat") then
                running = false
                break
            end

            local car, _ = findCar()
            if not car then
                running = false
                break
            end

            curCar = car

            if isRev and cfg.speed > 0 then
                local base = car.PrimaryPart
                if not base then
                    for _, p in ipairs(car:GetDescendants()) do
                        if p:IsA("BasePart") then base = p break end
                    end
                end

                if base and base.AssemblyMass > 0 then
                    local mass = base.AssemblyMass
                    local interval = 0.1  -- Aumentado para evitar tremor

                    -- Força pra trás (ré)
                    local back = -base.CFrame.LookVector
                    local impulse = back * cfg.speed * cfg.force * mass * interval

                    -- Aplica no eixo traseiro
                    local rearParts = getRearAxle(car)
                    if rearParts then
                        for _, p in ipairs(rearParts) do
                            if p and p.Parent and p.AssemblyMass > 0 then
                                p:ApplyImpulse(impulse / #rearParts)
                            end
                        end
                    end

                    -- Drift: reduz o grip da roda traseira (sem impulso lateral)
                    if isDrift and cfg.driftPower > 0 then
                        local rearParts2 = getRearAxle(car)
                        if rearParts2 then
                            -- Calcula a velocidade atual do carro para proporção
                            local velocity = base.AssemblyLinearVelocity.Magnitude
                            -- Grip diminui conforme velocidade aumenta (derrapagem natural)
                            local gripFactor = math.clamp(1 - (velocity / 100) * (cfg.driftPower / 100), 0.1, 0.9)

                            for _, p in ipairs(rearParts2) do
                                if p and p.Parent then
                                    p.CustomPhysicalProperties = PhysicalProperties.new(
                                        0.7,        -- density
                                        gripFactor, -- friction
                                        0.3,        -- elasticity
                                        1,          -- frictionWeight
                                        1           -- elasticityWeight
                                    )
                                end
                            end
                        end
                    else
                        -- Restaura grip normal quando não estiver driftando
                        local rearParts3 = getRearAxle(car)
                        if rearParts3 then
                            for _, p in ipairs(rearParts3) do
                                if p and p.Parent then
                                    p.CustomPhysicalProperties = nil
                                end
                            end
                        end
                    end
                end
            end

            task.wait(0.1)  -- Intervalo maior = menos tremor
        end
    end)
end

-- ===== INPUTS =====
UIS.InputBegan:Connect(function(i, gp)
    if gp then return end
    local k = i.KeyCode
    if k == Enum.KeyCode.S or k == Enum.KeyCode.Down or k == Enum.KeyCode.LeftShift or k == Enum.KeyCode.RightShift then
        isRev = true
        if cfg.autoDrift then isDrift = true end
    elseif k == Enum.KeyCode.Space then
        isDrift = true
    elseif k == Enum.KeyCode.F1 then
        panelOpen = not panelOpen
        if _G.lzGui then _G.lzGui.Enabled = panelOpen end
    end
end)

UIS.InputEnded:Connect(function(i, gp)
    if gp then return end
    local k = i.KeyCode
    if k == Enum.KeyCode.S or k == Enum.KeyCode.Down or k == Enum.KeyCode.LeftShift or k == Enum.KeyCode.RightShift then
        isRev = false
        if not UIS:IsKeyDown(Enum.KeyCode.Space) then isDrift = false end
    elseif k == Enum.KeyCode.Space then
        if not (cfg.autoDrift and isRev) then isDrift = false end
    end
end)

-- ===== DETECÇÃO IMEDIATA =====
local function hookCharacter(char)
    local hum = char:WaitForChild("Humanoid", 5)
    if not hum then return end

    hum.Seated:Connect(function(active, seat)
        if active and seat and seat:IsA("VehicleSeat") then
            task.wait(0.1)
            startLoop()
        else
            stopLoop()
            curCar = nil
            curSeat = nil
        end
    end)
end

if LP.Character then hookCharacter(LP.Character) end
LP.CharacterAdded:Connect(hookCharacter)

-- ===== UI =====
local gui = Instance.new("ScreenGui")
gui.Name = "LZ_HUB_v1"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.Parent = LP:WaitForChild("PlayerGui")
_G.lzGui = gui

local f = Instance.new("Frame")
f.Size = UDim2.new(0, 240, 0, 230)
f.Position = UDim2.new(0.5, -120, 0.2, 0)
f.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
f.BorderSizePixel = 0
f.Active = true
f.Parent = gui
Instance.new("UICorner", f).CornerRadius = UDim.new(0, 10)

-- Drag
local dragging, dragStart, startPos = false, nil, nil
f.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = f.Position
    end
end)
UIS.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        f.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

local title = Instance.new("TextLabel", f)
title.Size = UDim2.new(1, 0, 0, 28)
title.BackgroundTransparency = 1
title.Text = "⚡ LZ HUB v1"
title.TextColor3 = Color3.fromRGB(150, 100, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 14

local status = Instance.new("TextLabel", f)
status.Size = UDim2.new(1, -10, 0, 16)
status.Position = UDim2.new(0, 5, 0, 28)
status.BackgroundTransparency = 1
status.Text = "🔴 Sem carro"
status.TextColor3 = Color3.fromRGB(255, 100, 100)
status.Font = Enum.Font.Gotham
status.TextSize = 11
status.TextXAlignment = Enum.TextXAlignment.Left

task.spawn(function()
    while true do
        task.wait(0.3)
        if curCar and curCar.Parent then
            status.Text = "🟢 " .. curCar.Name
            status.TextColor3 = Color3.fromRGB(100, 255, 100)
        else
            status.Text = "🔴 Sem carro"
            status.TextColor3 = Color3.fromRGB(255, 100, 100)
        end
    end
end)

local input = Instance.new("TextBox", f)
input.Size = UDim2.new(0.9, 0, 0, 32)
input.Position = UDim2.new(0.05, 0, 0, 50)
input.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
input.TextColor3 = Color3.fromRGB(255, 255, 255)
input.PlaceholderText = "Velocidade (ex: 1.2)"
input.Font = Enum.Font.Gotham
input.TextSize = 14
input.ClearTextOnFocus = false
Instance.new("UICorner", input).CornerRadius = UDim.new(0, 6)

local apply = Instance.new("TextButton", f)
apply.Size = UDim2.new(0.9, 0, 0, 30)
apply.Position = UDim2.new(0.05, 0, 0, 88)
apply.BackgroundColor3 = Color3.fromRGB(150, 100, 255)
apply.Text = "Aplicar"
apply.TextColor3 = Color3.fromRGB(255, 255, 255)
apply.Font = Enum.Font.GothamBold
apply.TextSize = 13
Instance.new("UICorner", apply).CornerRadius = UDim.new(0, 6)

apply.MouseButton1Click:Connect(function()
    local n = tonumber(input.Text)
    if n and n > 0 then
        cfg.speed = n
        apply.Text = "✓ " .. n
        task.delay(1, function() apply.Text = "Aplicar" end)
    else
        apply.Text = "Inválido"
        task.delay(1, function() apply.Text = "Aplicar" end)
    end
end)

-- Toggle Drift Auto
local b = Instance.new("TextButton", f)
b.Size = UDim2.new(0.9, 0, 0, 26)
b.Position = UDim2.new(0.05, 0, 0, 126)
b.BackgroundColor3 = Color3.fromRGB(0, 150, 80)
b.Text = "Drift Auto: ON"
b.TextColor3 = Color3.fromRGB(255, 255, 255)
b.Font = Enum.Font.GothamBold
b.TextSize = 11
Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
b.MouseButton1Click:Connect(function()
    cfg.autoDrift = not cfg.autoDrift
    b.Text = "Drift Auto: " .. (cfg.autoDrift and "ON" or "OFF")
    b.BackgroundColor3 = cfg.autoDrift and Color3.fromRGB(0, 150, 80) or Color3.fromRGB(150, 50, 50)
end)

-- Força impulso
local flabel = Instance.new("TextLabel", f)
flabel.Size = UDim2.new(0.9, 0, 0, 14)
flabel.Position = UDim2.new(0.05, 0, 0, 158)
flabel.BackgroundTransparency = 1
flabel.Text = "Força impulso: " .. cfg.force
flabel.TextColor3 = Color3.fromRGB(180, 180, 180)
flabel.Font = Enum.Font.Gotham
flabel.TextSize = 10
flabel.TextXAlignment = Enum.TextXAlignment.Left

local fm = Instance.new("TextButton", f)
fm.Size = UDim2.new(0.15, 0, 0, 26)
fm.Position = UDim2.new(0.05, 0, 0, 174)
fm.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
fm.Text = "−"
fm.TextColor3 = Color3.fromRGB(255, 255, 255)
fm.Font = Enum.Font.GothamBold
fm.TextSize = 16
Instance.new("UICorner", fm).CornerRadius = UDim.new(0, 6)

local fp = Instance.new("TextButton", f)
fp.Size = UDim2.new(0.15, 0, 0, 26)
fp.Position = UDim2.new(0.80, 0, 0, 174)
fp.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
fp.Text = "+"
fp.TextColor3 = Color3.fromRGB(255, 255, 255)
fp.Font = Enum.Font.GothamBold
fp.TextSize = 16
Instance.new("UICorner", fp).CornerRadius = UDim.new(0, 6)

local fv = Instance.new("TextLabel", f)
fv.Size = UDim2.new(0.55, 0, 0, 26)
fv.Position = UDim2.new(0.225, 0, 0, 174)
fv.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
fv.Text = tostring(cfg.force)
fv.TextColor3 = Color3.fromRGB(255, 255, 255)
fv.Font = Enum.Font.GothamBold
fv.TextSize = 13
Instance.new("UICorner", fv).CornerRadius = UDim.new(0, 6)

fm.MouseButton1Click:Connect(function()
    cfg.force = math.max(10, cfg.force - 10)
    fv.Text = tostring(cfg.force)
    flabel.Text = "Força impulso: " .. cfg.force
end)
fp.MouseButton1Click:Connect(function()
    cfg.force = math.min(1000, cfg.force + 10)
    fv.Text = tostring(cfg.force)
    flabel.Text = "Força impulso: " .. cfg.force
end)

-- Força drift (agora só controla o grip)
local dlabel = Instance.new("TextLabel", f)
dlabel.Size = UDim2.new(0.9, 0, 0, 14)
dlabel.Position = UDim2.new(0.05, 0, 0, 204)
dlabel.BackgroundTransparency = 1
dlabel.Text = "Drift (grip): " .. cfg.driftPower
dlabel.TextColor3 = Color3.fromRGB(180, 180, 180)
dlabel.Font = Enum.Font.Gotham
dlabel.TextSize = 10
dlabel.TextXAlignment = Enum.TextXAlignment.Left

local dm = Instance.new("TextButton", f)
dm.Size = UDim2.new(0.15, 0, 0, 26)
dm.Position = UDim2.new(0.05, 0, 0, 220)
dm.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
dm.Text = "−"
dm.TextColor3 = Color3.fromRGB(255, 255, 255)
dm.Font = Enum.Font.GothamBold
dm.TextSize = 16
Instance.new("UICorner", dm).CornerRadius = UDim.new(0, 6)

local dp = Instance.new("TextButton", f)
dp.Size = UDim2.new(0.15, 0, 0, 26)
dp.Position = UDim2.new(0.80, 0, 0, 220)
dp.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
dp.Text = "+"
dp.TextColor3 = Color3.fromRGB(255, 255, 255)
dp.Font = Enum.Font.GothamBold
dp.TextSize = 16
Instance.new("UICorner", dp).CornerRadius = UDim.new(0, 6)

local dv = Instance.new("TextLabel", f)
dv.Size = UDim2.new(0.55, 0, 0, 26)
dv.Position = UDim2.new(0.225, 0, 0, 220)
dv.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
dv.Text = tostring(cfg.driftPower)
dv.TextColor3 = Color3.fromRGB(255, 255, 255)
dv.Font = Enum.Font.GothamBold
dv.TextSize = 13
Instance.new("UICorner", dv).CornerRadius = UDim.new(0, 6)

dm.MouseButton1Click:Connect(function()
    cfg.driftPower = math.max(10, cfg.driftPower - 5)
    dv.Text = tostring(cfg.driftPower)
    dlabel.Text = "Drift (grip): " .. cfg.driftPower
end)
dp.MouseButton1Click:Connect(function()
    cfg.driftPower = math.min(200, cfg.driftPower + 5)
    dv.Text = tostring(cfg.driftPower)
    dlabel.Text = "Drift (grip): " .. cfg.driftPower
end)

f.Size = UDim2.new(0, 240, 0, 260)

-- Botão flutuante
local openBtn = Instance.new("TextButton", gui)
openBtn.Size = UDim2.new(0, 45, 0, 45)
openBtn.Position = UDim2.new(0, 20, 0.5, -22)
openBtn.BackgroundColor3 = Color3.fromRGB(150, 100, 255)
openBtn.Text = "LZ"
openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
openBtn.Font = Enum.Font.GothamBold
openBtn.TextSize = 16
openBtn.Visible = false
openBtn.Active = true
Instance.new("UICorner", openBtn).CornerRadius = UDim.new(1, 0)

local btnDrag, btnStart, btnPos = false, nil, nil
openBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        btnDrag = true
        btnStart = input.Position
        btnPos = openBtn.Position
    end
end)
UIS.InputChanged:Connect(function(input)
    if btnDrag and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local d = input.Position - btnStart
        openBtn.Position = UDim2.new(btnPos.X.Scale, btnPos.X.Offset + d.X, btnPos.Y.Scale, btnPos.Y.Offset + d.Y)
    end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        btnDrag = false
    end
end)

local closeBtn = Instance.new("TextButton", f)
closeBtn.Size = UDim2.new(0, 24, 0, 24)
closeBtn.Position = UDim2.new(1, -28, 0, 4)
closeBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
closeBtn.Text = "✕"
closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 14
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)

closeBtn.MouseButton1Click:Connect(function()
    f.Visible = false
    openBtn.Visible = true
end)

openBtn.MouseButton1Click:Connect(function()
    f.Visible = true
    openBtn.Visible = false
end)

print("[LZ HUB v1] SHIFT/S = ré+drift | ESPAÇO = drift | F1 = painel")
