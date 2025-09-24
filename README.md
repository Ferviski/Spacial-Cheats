--// Script completo: GUI (bola), ESP, Aimbot, FOV ajustável, submenu e GodMode
-- Atenção: GodMode é client-side (pode não proteger contra validações server-side)

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
local humanoidHealthConn = nil -- para desconectar quando necessário

-- GUI PRINCIPAL
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ScriptBallGui"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

-- BOLA FLUTUANTE (ícone)
local ballButton = Instance.new("ImageButton")
ballButton.Name = "FloatingBall"
ballButton.Size = ballSize
ballButton.Position = GUIOffset
ballButton.BackgroundTransparency = 1
ballButton.Image = ballImageId
ballButton.Parent = ScreenGui
ballButton.ZIndex = 2

-- MENU
local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0,360,0,260)
Frame.Position = UDim2.new(0.5,-180,0.5,-130)
Frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
Frame.Visible = false
Frame.Parent = ScreenGui
local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0,10)
UICorner.Parent = Frame

-- FUNÇÕES GUI
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

local function CreateConfigIcon(parentBtn,callback)
    local icon = Instance.new("TextButton")
    icon.Size = UDim2.new(0,35,0,35)
    icon.Position = UDim2.new(0,parentBtn.Position.X.Offset + parentBtn.Size.X.Offset + 5,0,parentBtn.Position.Y.Offset)
    icon.BackgroundColor3 = Color3.fromRGB(50,50,50)
    icon.Text = "⚙️"
    icon.TextColor3 = Color3.fromRGB(255,255,255)
    icon.Parent = Frame
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0,8)
    corner.Parent = icon
    icon.MouseButton1Click:Connect(callback)
    return icon
end

local function CreateSlider(posY,min,max,default,callback,parent)
    local sliderFrame = Instance.new("Frame")
    sliderFrame.Size = UDim2.new(0,280,0,30)
    sliderFrame.Position = UDim2.new(0,20,0,posY)
    sliderFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
    sliderFrame.Parent = parent or Frame

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1,0,0,6)
    bar.Position = UDim2.new(0,0,0.5,-3)
    bar.BackgroundColor3 = Color3.fromRGB(100,100,100)
    bar.Parent = sliderFrame

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0,10,0,20)
    knob.BackgroundColor3 = Color3.fromRGB(200,200,200)
    knob.Position = UDim2.new((default-min)/(max-min),-5,0.5,-10)
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
            local mouse = UIS:GetMouseLocation().X
            local rel = math.clamp((mouse - bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1)
            knob.Position = UDim2.new(rel,-5,0.5,-10)
            local value = math.floor(min + (max-min)*rel)
            callback(value)
        end
    end)
end

-- BOTÕES PRINCIPAIS
CreateButton("Toggle ESP",20,function() ESPEnabled = not ESPEnabled end)
local AimbotBtn = CreateButton("Toggle Aimbot",70,function() AIMBOTEnabled = not AIMBOTEnabled end)

-- SUBMENU AIMBOT
local AimbotConfigFrame = Instance.new("Frame")
AimbotConfigFrame.Size = UDim2.new(0,300,0,100)
AimbotConfigFrame.Position = UDim2.new(0,20,0,120)
AimbotConfigFrame.BackgroundColor3 = Color3.fromRGB(40,40,40)
AimbotConfigFrame.Visible = false
AimbotConfigFrame.Parent = Frame
local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0,8)
corner.Parent = AimbotConfigFrame

local FOVToggle = Instance.new("TextButton")
FOVToggle.Size = UDim2.new(0,260,0,30)
FOVToggle.Position = UDim2.new(0,20,0,10)
FOVToggle.BackgroundColor3 = Color3.fromRGB(70,70,70)
FOVToggle.TextColor3 = Color3.fromRGB(255,255,255)
FOVToggle.Text = "FOV: ON"
FOVToggle.Parent = AimbotConfigFrame
local corner2 = Instance.new("UICorner")
corner2.CornerRadius = UDim.new(0,8)
corner2.Parent = FOVToggle
FOVToggle.MouseButton1Click:Connect(function()
    FOVEnabled = not FOVEnabled
    FOVToggle.Text = "FOV: " .. (FOVEnabled and "ON" or "OFF")
end)

CreateSlider(50,50,600,FOV,function(val) FOV = val end,AimbotConfigFrame)
CreateConfigIcon(AimbotBtn,function() AimbotConfigFrame.Visible = not AimbotConfigFrame.Visible end)

-- BOTÃO GODMODE
CreateButton("Toggle GodMode", 170, function()
    GODMODEEnabled = not GODMODEEnabled
    -- Se ativou agora, aplica imediatamente ao personagem atual
    if GODMODEEnabled then
        -- Apply to current char (function abaixo fará bind)
        local char = LocalPlayer.Character
        if char and char:FindFirstChildOfClass("Humanoid") then
            -- force immediate full health
            local hum = char:FindFirstChildOfClass("Humanoid")
            hum.Health = hum.MaxHealth
        end
    end
end)

-- GUI abrir/fechar pela bola
ballButton.MouseButton1Click:Connect(function() Frame.Visible = not Frame.Visible end)

-- ARRASTAR BOLA
local dragging=false; local dragStart,startPos
ballButton.InputBegan:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 then
        dragging=true
        dragStart=input.Position
        startPos=ballButton.Position
    end
end)
ballButton.InputEnded:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 then dragging=false end
end)
ballButton.InputChanged:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseMovement and dragging then
        local delta=input.Position-dragStart
        ballButton.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+delta.X,startPos.Y.Scale,startPos.Y.Offset+delta.Y)
    end
end)

-- FUNÇÕES AUX
local function IsEnemy(player)
    if player==LocalPlayer then return false end
    if player.Team and LocalPlayer.Team then return player.Team~=LocalPlayer.Team end
    return true
end

local function GetClosestEnemy()
    local closest,dist=nil,FOV
    local mouse=UIS:GetMouseLocation()
    for _,player in ipairs(Players:GetPlayers()) do
        if player.Character and player.Character:FindFirstChild("Head") and IsEnemy(player) then
            local pos,vis=Camera:WorldToViewportPoint(player.Character.Head.Position)
            if vis then
                local mag=(Vector2.new(pos.X,pos.Y)-mouse).Magnitude
                if mag<dist then dist=mag closest=player end
            end
        end
    end
    return closest
end

-- LIMPAR ESP
local function ClearESP(player)
    if ESPObjects[player] then
        if ESPObjects[player].Box then ESPObjects[player].Box:Remove() end
        if ESPObjects[player].Name then ESPObjects[player].Name:Remove() end
        ESPObjects[player]=nil
    end
end
Players.PlayerRemoving:Connect(ClearESP)

-- DESENHOS
local CircleFOV = Drawing.new("Circle")
CircleFOV.Color=Color3.fromRGB(0,255,0)
CircleFOV.Thickness=1.5
CircleFOV.NumSides=100
CircleFOV.Filled=false
CircleFOV.Visible=true

local LineToEnemy=Drawing.new("Line")
LineToEnemy.Color=Color3.fromRGB(0,255,255)
LineToEnemy.Thickness=1.5
LineToEnemy.Visible=false

local HeadDot=Drawing.new("Circle")
HeadDot.Color=Color3.fromRGB(255,255,0)
HeadDot.Radius=4
HeadDot.Filled=true
HeadDot.Visible=false

-- Função para conectar proteção do humanoid (godmode)
local function BindHumanoidGodMode(hum)
    -- desconecta conexão antiga se houver
    if humanoidHealthConn then
        humanoidHealthConn:Disconnect()
        humanoidHealthConn = nil
    end
    if not hum then return end
    -- garantir max health
    pcall(function()
        hum.Health = hum.MaxHealth
    end)
    -- escuta mudanças de Health e restaura se GODMODEEnabled
    humanoidHealthConn = hum:GetPropertyChangedSignal("Health"):Connect(function()
        if GODMODEEnabled then
            -- pequenas pausas para evitar conflito do set
            pcall(function()
                if hum and hum.Parent then
                    if hum.Health < hum.MaxHealth then
                        hum.Health = hum.MaxHealth
                    end
                end
            end)
        end
    end)
end

-- Ao respawnar, vincula o humanoid
LocalPlayer.CharacterAdded:Connect(function(char)
    -- desconectar previous
    if humanoidHealthConn then
        humanoidHealthConn:Disconnect()
        humanoidHealthConn = nil
    end
    local hum = char:WaitForChild("Humanoid", 5)
    if hum then
        -- se GodMode está ativo, bind
        if GODMODEEnabled then
            BindHumanoidGodMode(hum)
        end
        -- também limpa ESP do próprio jogador caso precise
        ClearESP(LocalPlayer)
        -- detecta morte para limpar ESP do personagem (e garantir rebind após respawn)
        hum.Died:Connect(function()
            -- em algumas situações o humanoid muda, garantimos limpar conexões
            if humanoidHealthConn then
                humanoidHealthConn:Disconnect()
                humanoidHealthConn = nil
            end
        end)
    end
end)

-- Se já tiver character ao executar o script, tenta bindar
do
    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum and GODMODEEnabled then
            BindHumanoidGodMode(hum)
        end
    end
end

-- LOOP ESP + FOV
RunService.RenderStepped:Connect(function()
    local mouse=UIS:GetMouseLocation()
    CircleFOV.Position=mouse
    CircleFOV.Radius=FOV
    CircleFOV.Visible=FOVEnabled

    for _,player in ipairs(Players:GetPlayers()) do
        if ESPEnabled and player.Character and player.Character:FindFirstChild("Head") and IsEnemy(player) then
            local root=player.Character:FindFirstChild("HumanoidRootPart") or player.Character.Head
            local pos,vis=Camera:WorldToViewportPoint(root.Position)
            local hum=player.Character:FindFirstChild("Humanoid")

            if vis and hum and hum.Health>0 then
                local scale=2000/(Camera.CFrame.Position-root.Position).Magnitude
                local size=Vector2.new(4*scale,5*scale)
                local boxpos=Vector2.new(pos.X-size.X/2,pos.Y-size.Y/2)

                if not ESPObjects[player] then
                    local box=Drawing.new("Square")
                    box.Thickness=1.5 box.Color=ESP_COLOR box.Filled=false
                    local name=Drawing.new("Text")
                    name.Text=player.Name name.Size=13 name.Color=ESP_COLOR name.Center=true
                    ESPObjects[player]={Box=box,Name=name}
                end

                local esp=ESPObjects[player]
                esp.Box.Size=size
                esp.Box.Position=boxpos
                esp.Box.Visible=true
                esp.Name.Position=Vector2.new(pos.X,pos.Y-size.Y/2-10)
                esp.Name.Visible=true
            else
                if ESPObjects[player] then
                    ESPObjects[player].Box.Visible=false
                    ESPObjects[player].Name.Visible=false
                end
            end
        else
            if ESPObjects[player] then
                ESPObjects[player].Box.Visible=false
                ESPObjects[player].Name.Visible=false
            end
        end
    end

    -- Linha e bolinha no inimigo mais próximo
    local target= (FOVEnabled or true) and (function() return (function() return (function() return (function() end)() end)() end)() end)()
    -- O trecho acima é só placeholder sem efeito; o GetClosestEnemy abaixo é chamado de verdade:
    local closest = (function() return (function() return (function() return (function() end)() end)() end)() end)()
    -- usamos a função GetClosestEnemy real:
    local tgt = nil
    local ok, res = pcall(GetClosestEnemy)
    if ok then tgt = res end

    if tgt and tgt.Character and tgt.Character:FindFirstChild("Head") and FOVEnabled then
        local headPos,vis=Camera:WorldToViewportPoint(tgt.Character.Head.Position)
        if vis then
            local head2D=Vector2.new(headPos.X,headPos.Y)
            local dist=(mouse-head2D).Magnitude
            if dist<=FOV then
                LineToEnemy.From=mouse
                LineToEnemy.To=head2D
                LineToEnemy.Visible=true
                HeadDot.Position=head2D
                HeadDot.Visible=true
            else
                LineToEnemy.Visible=false
                HeadDot.Visible=false
            end
        end
    else
        LineToEnemy.Visible=false
        HeadDot.Visible=false
    end
end)

-- AIMBOT INPUT
UIS.InputBegan:Connect(function(input) if input.UserInputType==AIMBOT_KEY then aiming=true end end)
UIS.InputEnded:Connect(function(input) if input.UserInputType==AIMBOT_KEY then aiming=false end end)

-- Aimbot movement loop
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
