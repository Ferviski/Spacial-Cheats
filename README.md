--// Script completo: GUI (bola), ESP, Aimbot, FOV ajustável, submenus e GodMode melhorado
-- Nota: GodMode é client-side; pode não impedir mortes em servidores com validação server-side.

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera

-- CONFIG
local ballImageId = "rbxassetid://81168570238976"
local ballSize = UDim2.new(0,50,0,50)
local GUIOffset = UDim2.new(0,20,0,50)
local ESP_COLOR = Color3.fromRGB(255,0,0)
local AIMBOT_KEY = Enum.UserInputType.MouseButton2
local Smoothness = 0.2

-- VARIÁVEIS
local ESPEnabled = false
local AIMBOTEnabled = false
local FOVEnabled = true
local ESPObjects = {}
local aiming = false
local FOV = 200

-- GODMODE VARS
local GODMODEEnabled = false
local humanoidHealthConn = nil

-- UTIL: limpa esp de 1 jogador
local function ClearESP(player)
    if ESPObjects[player] then
        pcall(function()
            if ESPObjects[player].Box then ESPObjects[player].Box:Remove() end
            if ESPObjects[player].Name then ESPObjects[player].Name:Remove() end
        end)
        ESPObjects[player] = nil
    end
end

-- Limpa ESP quando player sai
Players.PlayerRemoving:Connect(function(plr)
    ClearESP(plr)
end)

-- GUI BASE
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ScriptBallGui"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local ballButton = Instance.new("ImageButton")
ballButton.Name = "FloatingBall"
ballButton.Size = ballSize
ballButton.Position = GUIOffset
ballButton.BackgroundTransparency = 1
ballButton.Image = ballImageId
ballButton.Parent = ScreenGui
ballButton.ZIndex = 2

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0,360,0,300)
Frame.Position = UDim2.new(0.5,-180,0.5,-150)
Frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
Frame.Visible = false
Frame.Parent = ScreenGui
local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0,10)
UICorner.Parent = Frame

-- FUNÇÕES DE UI
local function CreateButton(text,posY,callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0,250,0,35)
    btn.Position = UDim2.new(0,20,0,posY)
    btn.BackgroundColor3 = Color3.fromRGB(70,70,70)
    btn.TextColor3 = Color3.fromRGB(255,255,255)
    btn.Text = text
    btn.Parent = Frame

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,8)
    corner.Parent = btn

    btn.MouseButton1Click:Connect(callback)
    return btn
end

local function CreateConfigIcon(parentBtn, callback)
    local icon = Instance.new("TextButton")
    icon.Size = UDim2.new(0,30,0,30)
    icon.Position = UDim2.new(0, parentBtn.Position.X.Offset + parentBtn.Size.X.Offset + 8, 0, parentBtn.Position.Y.Offset + 2)
    icon.BackgroundColor3 = Color3.fromRGB(50,50,50)
    icon.Text = "⚙️"
    icon.TextColor3 = Color3.fromRGB(255,255,255)
    icon.Parent = Frame

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,6)
    corner.Parent = icon

    icon.MouseButton1Click:Connect(callback)
    return icon
end

local function CreateSlider(posY,min,max,default,callback,parent)
    parent = parent or Frame
    local sliderFrame = Instance.new("Frame")
    sliderFrame.Size = UDim2.new(0,280,0,30)
    sliderFrame.Position = UDim2.new(0,20,0,posY)
    sliderFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
    sliderFrame.Parent = parent

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1,0,0,6)
    bar.Position = UDim2.new(0,0,0.5,-3)
    bar.BackgroundColor3 = Color3.fromRGB(100,100,100)
    bar.Parent = sliderFrame

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0,12,0,20)
    knob.BackgroundColor3 = Color3.fromRGB(200,200,200)
    knob.Position = UDim2.new((default-min)/(max-min),-6,0.5,-10)
    knob.Parent = sliderFrame

    local dragging = false
    knob.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true end
    end)
    knob.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)

    RunService.RenderStepped:Connect(function()
        if dragging then
            local mouseX = UIS:GetMouseLocation().X
            local rel = math.clamp((mouseX - bar.AbsolutePosition.X)/bar.AbsoluteSize.X, 0, 1)
            knob.Position = UDim2.new(rel, -6, 0.5, -10)
            local value = math.floor(min + (max - min) * rel)
            callback(value)
        end
    end)
end

-- BOTÕES PRINCIPAIS
local espBtn = CreateButton("ESP: OFF", 20, function()
    ESPEnabled = not ESPEnabled
    espBtn.Text = "ESP: " .. (ESPEnabled and "ON" or "OFF")
end)

local aimbotBtn = CreateButton("Aimbot: OFF", 70, function()
    AIMBOTEnabled = not AIMBOTEnabled
    aimbotBtn.Text = "Aimbot: " .. (AIMBOTEnabled and "ON" or "OFF")
end)

-- SUBMENU AIMBOT
local aimbotConfig = Instance.new("Frame")
aimbotConfig.Size = UDim2.new(0,300,0,120)
aimbotConfig.Position = UDim2.new(0,20,0,115)
aimbotConfig.BackgroundColor3 = Color3.fromRGB(40,40,40)
aimbotConfig.Visible = false
aimbotConfig.Parent = Frame
local aimbotCorner = Instance.new("UICorner"); aimbotCorner.CornerRadius = UDim.new(0,8); aimbotCorner.Parent = aimbotConfig

local fovToggle = Instance.new("TextButton")
fovToggle.Size = UDim2.new(0,260,0,30)
fovToggle.Position = UDim2.new(0,20,0,10)
fovToggle.BackgroundColor3 = Color3.fromRGB(70,70,70)
fovToggle.TextColor3 = Color3.fromRGB(255,255,255)
fovToggle.Text = "FOV: ON"
fovToggle.Parent = aimbotConfig
local fovToggleCorner = Instance.new("UICorner"); fovToggleCorner.CornerRadius = UDim.new(0,8); fovToggleCorner.Parent = fovToggle
fovToggle.MouseButton1Click:Connect(function()
    FOVEnabled = not FOVEnabled
    fovToggle.Text = "FOV: " .. (FOVEnabled and "ON" or "OFF")
end)

CreateSlider(50,50,600,FOV, function(val) FOV = val end, aimbotConfig)
CreateConfigIcon(aimbotBtn, function()
    aimbotConfig.Visible = not aimbotConfig.Visible
end)

-- BOTÃO GODMODE + SUBMENU DO GODMODE
local godBtn = CreateButton("GodMode: OFF", 210, function()
    GODMODEEnabled = not GODMODEEnabled
    godBtn.Text = "GodMode: " .. (GODMODEEnabled and "ON" or "OFF")
    if GODMODEEnabled then
        local char = LocalPlayer.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                pcall(function() hum.Health = hum.MaxHealth end)
            end
        end
    else
        -- quando desativar, desconecta listener se existir
        if humanoidHealthConn then
            pcall(function() humanoidHealthConn:Disconnect() end)
            humanoidHealthConn = nil
        end
    end
end)

local godConfig = Instance.new("Frame")
godConfig.Size = UDim2.new(0,300,0,60)
godConfig.Position = UDim2.new(0,20,0,250)
godConfig.BackgroundColor3 = Color3.fromRGB(40,40,40)
godConfig.Visible = false
godConfig.Parent = Frame
local godCorner = Instance.new("UICorner"); godCorner.CornerRadius = UDim.new(0,8); godCorner.Parent = godConfig

local godLabel = Instance.new("TextLabel")
godLabel.Size = UDim2.new(1,-10,1,-10)
godLabel.Position = UDim2.new(0,5,0,5)
godLabel.BackgroundTransparency = 1
godLabel.TextColor3 = Color3.fromRGB(255,255,255)
godLabel.TextWrapped = true
godLabel.Text = "GodMode força Health ao máximo no cliente. Pode não funcionar em servidores com validação server-side."
godLabel.Parent = godConfig

CreateConfigIcon(godBtn, function()
    godConfig.Visible = not godConfig.Visible
end)

-- ABRIR/FECHAR MENU E ARRASTAR BOLA
ballButton.MouseButton1Click:Connect(function()
    Frame.Visible = not Frame.Visible
end)

do
    local dragging = false
    local dragStart, startPos
    ballButton.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = ballButton.Position
        end
    end)
    ballButton.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    ballButton.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement and dragging then
            local delta = input.Position - dragStart
            ballButton.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- FUNÇÕES AUXILIARES
local function IsEnemy(player)
    if player == LocalPlayer then return false end
    if player.Team and LocalPlayer.Team then
        return player.Team ~= LocalPlayer.Team
    end
    return true -- FFA fallback
end

local function GetClosestEnemy()
    local closest, dist = nil, FOV
    local mouse = UIS:GetMouseLocation()
    for _, player in ipairs(Players:GetPlayers()) do
        if player.Character and player.Character:FindFirstChild("Head") and IsEnemy(player) then
            local pos, vis = Camera:WorldToViewportPoint(player.Character.Head.Position)
            if vis then
                local mag = (Vector2.new(pos.X, pos.Y) - mouse).Magnitude
                if mag < dist then
                    dist = mag
                    closest = player
                end
            end
        end
    end
    return closest
end

-- DRAWINGS
local CircleFOV = Drawing.new("Circle")
CircleFOV.Color = Color3.fromRGB(0,255,0)
CircleFOV.Thickness = 1.5
CircleFOV.NumSides = 100
CircleFOV.Filled = false
CircleFOV.Visible = false

local LineToEnemy = Drawing.new("Line")
LineToEnemy.Color = Color3.fromRGB(0,255,255)
LineToEnemy.Thickness = 1.5
LineToEnemy.Visible = false

local HeadDot = Drawing.new("Circle")
HeadDot.Color = Color3.fromRGB(255,255,0)
HeadDot.Radius = 4
HeadDot.Filled = true
HeadDot.Visible = false

-- MONITORAR humanoid changes em respawns para limpar ESP do self e bindar godmode listener
local function OnCharacterAdded(char)
    if humanoidHealthConn then
        pcall(function() humanoidHealthConn:Disconnect() end)
        humanoidHealthConn = nil
    end

    local hum = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 5)
    if hum then
        hum.Died:Connect(function()
            if humanoidHealthConn then
                pcall(function() humanoidHealthConn:Disconnect() end)
                humanoidHealthConn = nil
            end
            ClearESP(LocalPlayer)
        end)

        if GODMODEEnabled then
            pcall(function() hum.Health = hum.MaxHealth end)
            humanoidHealthConn = hum:GetPropertyChangedSignal("Health"):Connect(function()
                if GODMODEEnabled and hum and hum.Parent then
                    if hum.Health < hum.MaxHealth then
                        pcall(function() hum.Health = hum.MaxHealth end)
                    end
                end
            end)
        end
    end
end

LocalPlayer.CharacterAdded:Connect(OnCharacterAdded)
if LocalPlayer.Character then
    OnCharacterAdded(LocalPlayer.Character)
end

-- Ao remover players, limpar ESP
Players.PlayerRemoving:Connect(function(plr) ClearESP(plr) end)

-- LOOP principal: ESP, FOV, linha e head dot
RunService.RenderStepped:Connect(function()
    local mouse = UIS:GetMouseLocation()
    CircleFOV.Position = mouse
    CircleFOV.Radius = FOV
    CircleFOV.Visible = FOVEnabled

    for _, player in ipairs(Players:GetPlayers()) do
        if ESPEnabled and player.Character and player.Character.Parent and player.Character:FindFirstChild("Head") and IsEnemy(player) then
            local root = player.Character:FindFirstChild("HumanoidRootPart") or player.Character:FindFirstChild("Torso") or player.Character.Head
            local pos, vis = Camera:WorldToViewportPoint(root.Position)
            local hum = player.Character:FindFirstChildOfClass("Humanoid")
            if vis and hum and hum.Health > 0 then
                local dist3 = (Camera.CFrame.Position - root.Position).Magnitude
                local scale = 2000 / math.max(dist3, 1)
                local size = Vector2.new(4 * scale, 5 * scale)
                local boxpos = Vector2.new(pos.X - size.X / 2, pos.Y - size.Y / 2)

                if not ESPObjects[player] then
                    local box = Drawing.new("Square")
                    box.Thickness = 1.5
                    box.Color = ESP_COLOR
                    box.Filled = false

                    local name = Drawing.new("Text")
                    name.Text = player.Name
                    name.Size = 13
                    name.Color = ESP_COLOR
                    name.Center = true

                    ESPObjects[player] = { Box = box, Name = name }
                end

                local esp = ESPObjects[player]
                esp.Box.Size = size
                esp.Box.Position = boxpos
                esp.Box.Visible = true
                esp.Name.Position = Vector2.new(pos.X, pos.Y - size.Y / 2 - 10)
                esp.Name.Visible = true
            else
                if ESPObjects[player] then
                    ESPObjects[player].Box.Visible = false
                    ESPObjects[player].Name.Visible = false
                end
            end
        else
            if ESPObjects[player] then
                ESPObjects[player].Box.Visible = false
                ESPObjects[player].Name.Visible = false
            end
        end
    end

    -- linha e bolinha no inimigo mais próximo
    local tgt = nil
    local ok, res = pcall(GetClosestEnemy)
    if ok then tgt = res end

    if tgt and tgt.Character and tgt.Character:FindFirstChild("Head") and FOVEnabled then
        local headPos, vis = Camera:WorldToViewportPoint(tgt.Character.Head.Position)
        if vis then
            local head2D = Vector2.new(headPos.X, headPos.Y)
            local dist = (mouse - head2D).Magnitude
            if dist <= FOV then
                LineToEnemy.From = mouse
                LineToEnemy.To = head2D
                LineToEnemy.Visible = true
                HeadDot.Position = head2D
                HeadDot.Visible = true
            else
                LineToEnemy.Visible = false
                HeadDot.Visible = false
            end
        else
            LineToEnemy.Visible = false
            HeadDot.Visible = false
        end
    else
        LineToEnemy.Visible = false
        HeadDot.Visible = false
    end
end)

-- INPUT para Aimbot
UIS.InputBegan:Connect(function(input)
    if input.UserInputType == AIMBOT_KEY then aiming = true end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == AIMBOT_KEY then aiming = false end
end)

-- Aimbot loop
RunService.RenderStepped:Connect(function()
    if AIMBOTEnabled and aiming then
        local target = nil
        local ok, res = pcall(GetClosestEnemy)
        if ok then target = res end
        if target and target.Character and target.Character:FindFirstChild("Head") then
            local head = target.Character.Head.Position
            local lookAt = CFrame.new(Camera.CFrame.Position, head)
            Camera.CFrame = Camera.CFrame:Lerp(lookAt, Smoothness)
        end
    end
end)

-- GodMode: forçar Health no loop (mais agressivo) e reconectar property changed
RunService.RenderStepped:Connect(function()
    if GODMODEEnabled and LocalPlayer.Character and LocalPlayer.Character.Parent then
        local hum = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then
            pcall(function()
                if hum.Health < hum.MaxHealth then
                    hum.Health = hum.MaxHealth
                end
            end)
            if not humanoidHealthConn then
                humanoidHealthConn = hum:GetPropertyChangedSignal("Health"):Connect(function()
                    if GODMODEEnabled and hum and hum.Parent then
                        if hum.Health < hum.MaxHealth then
                            pcall(function() hum.Health = hum.MaxHealth end)
                        end
                    end
                end)
            end
        end
    end
end)
