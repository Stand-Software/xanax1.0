local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- --- CONFIGURAÇÕES GERAIS ---
local Settings = {
    -- Aimbot
    AimbotEnabled = false,
    TeamCheck = false,
    ShowFOV = false,
    FOVRadius = 100,
    FOVColor = Color3.fromRGB(255, 255, 255),
    TargetPart = "Head", 
    AimKey = Enum.UserInputType.MouseButton2,
    AimDistance = 500,
    Smoothing = 2,
    -- ESP / Visuals
    Box = false,
    Skeleton = false,
    Tracers = false,
    Distance = false,
    Names = false,
    Health = false,
    LocalPlayer = false,
    ESPColor = Color3.fromRGB(255, 0, 85),
    Thickness = 1,
    BoxThickness = 2,
    MaxDistance = 500,
    -- Pessoal / Fly
    FlyEnabled = false,
    IsFlying = false,
    FlySpeed = 20,
    FlyBoost = 350,
    FlyKey = Enum.KeyCode.CapsLock,
    InfJump = false,
    -- Jogadores
    SelectedPlayer = nil,
    PuxarLoop = false,
    SpectateEnabled = false,
    SpectateDist = 15,
    SpectateRotation = 0,
    -- Misc (NOVO)
    SpinbotEnabled = false,
    SpinbotSpeed = 50
}

local Cache = {}
local isAiming = false
local bodyVelocity, bodyGyro, flyConnection

-- Carregando a Lib do Hub
local Hub = loadstring(game:HttpGet("https://raw.githubusercontent.com/Stand-Software/hub/refs/heads/main/README.md"))()

local Window = Hub:CreateWindow({
    Title = "Xanax Hub"
})

local AimTab = Window:CreateTab("Aimbot")
local VisTab = Window:CreateTab("Visuals")
local SelfTab = Window:CreateTab("Pessoal")
local PlayersTab = Window:CreateTab("Jogadores")
local MiscTab = Window:CreateTab("Misc") -- Criando a nova aba

-- --- OBJETOS DE DESENHO (FOV) ---
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 1
FOVCircle.NumSides = 100
FOVCircle.Filled = false
FOVCircle.Transparency = 1
FOVCircle.Visible = false

-- --- FUNÇÕES AUXILIARES ---
local function create(class, props)
    local obj = Instance.new(class)
    for i, v in pairs(props) do obj[i] = v end
    return obj
end

local function createDrawing(class, props)
    local obj = Drawing.new(class)
    for i, v in pairs(props) do obj[i] = v end
    return obj
end

local function hideAll(data)
    if not data then return end
    for _, l in pairs(data.Box) do l.Visible = false end
    for _, l in pairs(data.Skeleton) do l.Visible = false end
    data.Tracer.Visible = false
    data.Name.Visible = false
    data.Dist.Visible = false
    data.HealthBarBG.Visible = false
    data.HealthBarMain.Visible = false
end

local function removeESP(p)
    if Cache[p] then
        for _, obj in pairs(Cache[p]) do
            if type(obj) == "table" then
                for _, subObj in pairs(obj) do subObj:Remove() end
            else
                obj:Remove()
            end
        end
        Cache[p] = nil
    end
end

-- --- LÓGICA VOO (FLY) ---
local function disableFly()
    Settings.IsFlying = false
    if flyConnection then flyConnection:Disconnect(); flyConnection = nil end
    local char = player.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum.PlatformStand = false end
        if bodyVelocity then bodyVelocity:Destroy(); bodyVelocity = nil end
        if bodyGyro then bodyGyro:Destroy(); bodyGyro = nil end
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = true end
        end
    end
end

local function enableFly()
    local char = player.Character or player.CharacterAdded:Wait()
    local hum = char:WaitForChild("Humanoid")
    local root = char:WaitForChild("HumanoidRootPart")
    hum.PlatformStand = true

    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bodyVelocity.Parent = root

    bodyGyro = Instance.new("BodyGyro")
    bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
    bodyGyro.P = 1000
    bodyGyro.D = 100
    bodyGyro.Parent = root

    flyConnection = RunService.Heartbeat:Connect(function()
        if not Settings.IsFlying then return end
        local moveDir = Vector3.new()
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir += Vector3.new(0,0,-1) end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir += Vector3.new(0,0,1) end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir += Vector3.new(-1,0,0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir += Vector3.new(1,0,0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.E) then moveDir += Vector3.new(0,1,0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.Q) then moveDir += Vector3.new(0,-1,0) end

        local speed = UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) and (Settings.FlySpeed + Settings.FlyBoost) or Settings.FlySpeed
        bodyVelocity.Velocity = camera.CFrame:VectorToWorldSpace(moveDir) * speed
        bodyGyro.CFrame = camera.CFrame

        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end)
end

-- --- LÓGICA AIMBOT ---
local function getClosestPlayer()
    local target = nil
    local shortestDistance = math.huge
    local mousePos = UserInputService:GetMouseLocation()

    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character then
            if Settings.TeamCheck and p.Team == player.Team then continue end

            local char = p.Character
            local part = char:FindFirstChild(Settings.TargetPart)
            local hum = char:FindFirstChild("Humanoid")
            local hrp = char:FindFirstChild("HumanoidRootPart")

            if part and hum and hum.Health > 0 and hrp then
                local mag = (hrp.Position - camera.CFrame.Position).Magnitude
                if mag <= Settings.AimDistance then
                    local screenPos, onScreen = camera:WorldToViewportPoint(part.Position)
                    if onScreen then
                        local distFromMouse = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if distFromMouse <= Settings.FOVRadius then
                            if mag < shortestDistance then
                                target = part
                                shortestDistance = mag
                            end
                        end
                    end
                end
            end
        end
    end
    return target
end

-- Detecção de Input
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then isAiming = true end
    
    if Settings.FlyEnabled and input.KeyCode == Settings.FlyKey then
        Settings.IsFlying = not Settings.IsFlying
        if Settings.IsFlying then enableFly() else disableFly() end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then isAiming = false end
end)

local Bones = {
    R15 = {
        {"UpperTorso", "Head"}, {"UpperTorso", "LowerTorso"},
        {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"}, {"LeftLowerLeg", "LeftFoot"},
        {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}, {"RightLowerLeg", "RightFoot"},
        {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"}, {"LeftLowerArm", "LeftHand"},
        {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"}, {"RightLowerArm", "Hand"}
    },
    R6 = {
        {"Torso", "Head"}, {"Torso", "Left Arm"}, {"Torso", "Right Arm"},
        {"Torso", "Left Leg"}, {"Torso", "Right Leg"}
    }
}

-- --- LOOP PRINCIPAL (RENDER STEPPED) ---
RunService.RenderStepped:Connect(function()
    -- FOV Circle
    FOVCircle.Visible = Settings.ShowFOV
    FOVCircle.Radius = Settings.FOVRadius
    FOVCircle.Color = Settings.FOVColor
    FOVCircle.Position = UserInputService:GetMouseLocation()

    -- Aimbot Logic
    if Settings.AimbotEnabled and isAiming then
        local target = getClosestPlayer()
        if target then
            local screenPos, onScreen = camera:WorldToViewportPoint(target.Position)
            if onScreen then
                local mousePos = UserInputService:GetMouseLocation()
                local moveX = (screenPos.X - mousePos.X) / Settings.Smoothing
                local moveY = (screenPos.Y - mousePos.Y) / Settings.Smoothing
                if mousemoverel then mousemoverel(moveX, moveY) end
            end
        end
    end

    -- ESP Logic
    for _, data in pairs(Cache) do hideAll(data) end

    for _, p in pairs(Players:GetPlayers()) do
        if (p ~= player or Settings.LocalPlayer) and p.Character then 
            local char = p.Character
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local hum = char:FindFirstChild("Humanoid")

            if hrp and hum and hum.Health > 0 then
                local mag = (hrp.Position - camera.CFrame.Position).Magnitude
                if mag <= Settings.MaxDistance then
                    local pos, onScreen = camera:WorldToViewportPoint(hrp.Position)

                    if onScreen then
                        if not Cache[p] then
                            Cache[p] = {
                                Box = {
                                    createDrawing("Line",{}), createDrawing("Line",{}),
                                    createDrawing("Line",{}), createDrawing("Line",{}),
                                    createDrawing("Line",{}), createDrawing("Line",{}),
                                    createDrawing("Line",{}), createDrawing("Line",{})
                                },
                                Skeleton = {},
                                Tracer = createDrawing("Line", {Thickness = Settings.Thickness}),
                                Name = createDrawing("Text", {Size = 13, Center = true, Outline = true}),
                                Dist = createDrawing("Text", {Size = 12, Center = true, Outline = true}),
                                HealthBarBG = createDrawing("Line", {Thickness = 5, Color = Color3.new(0,0,0)}),
                                HealthBarMain = createDrawing("Line", {Thickness = 3})
                            }
                        end

                        local data = Cache[p]
                        local head = char:FindFirstChild("Head")
                        local headPos = head and camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
                        local legPos = camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))

                        if headPos and legPos then
                            local height = math.abs(headPos.Y - legPos.Y)
                            local width = height / 1.5
                            local x, y = pos.X - width / 2, pos.Y - height / 2
                            local l = width / 4

                            if Settings.Box then
                                local tl, tr, bl, br = Vector2.new(x, y), Vector2.new(x + width, y), Vector2.new(x, y + height), Vector2.new(x + width, y + height)
                                local box = data.Box
                                box[1].From = tl; box[1].To = tl + Vector2.new(l, 0)
                                box[2].From = tl; box[2].To = tl + Vector2.new(0, l)
                                box[3].From = tr; box[3].To = tr - Vector2.new(l, 0)
                                box[4].From = tr; box[4].To = tr + Vector2.new(0, l)
                                box[5].From = bl; box[5].To = bl + Vector2.new(l, 0)
                                box[6].From = bl; box[6].To = bl - Vector2.new(0, l)
                                box[7].From = br; box[7].To = br - Vector2.new(l, 0)
                                box[8].From = br; box[8].To = br - Vector2.new(0, l)
                                for _, line in pairs(box) do line.Visible = true; line.Color = Settings.ESPColor; line.Thickness = Settings.BoxThickness end
                            end

                            if Settings.Health then
                                local hp = hum.Health / hum.MaxHealth
                                data.HealthBarBG.Visible = true; data.HealthBarBG.From = Vector2.new(x - 6, y + height); data.HealthBarBG.To = Vector2.new(x - 6, y)
                                data.HealthBarMain.Visible = true; data.HealthBarMain.From = Vector2.new(x - 6, y + height); data.HealthBarMain.To = Vector2.new(x - 6, y + height - (height * hp)); data.HealthBarMain.Color = Color3.fromHSV(hp * 0.3, 1, 1)
                            end

                            if Settings.Skeleton then
                                local rigType = (hum.RigType == Enum.HumanoidRigType.R15) and "R15" or "R6"
                                local boneConfig = Bones[rigType]
                                if #data.Skeleton ~= #boneConfig then
                                    for _, b in pairs(data.Skeleton) do b:Remove() end
                                    data.Skeleton = {}
                                    for i = 1, #boneConfig do table.insert(data.Skeleton, createDrawing("Line", {Thickness = Settings.Thickness})) end
                                end
                                for i, bone in pairs(boneConfig) do
                                    local p1, p2 = char:FindFirstChild(bone[1]), char:FindFirstChild(bone[2])
                                    if p1 and p2 then
                                        local v1, o1 = camera:WorldToViewportPoint(p1.Position)
                                        local v2, o2 = camera:WorldToViewportPoint(p2.Position)
                                        if o1 and o2 then
                                            local line = data.Skeleton[i]
                                            line.From = Vector2.new(v1.X, v1.Y); line.To = Vector2.new(v2.X, v2.Y)
                                            line.Color = Settings.ESPColor; line.Visible = true
                                        end
                                    end
                                end
                            end

                            if Settings.Names then data.Name.Visible = true; data.Name.Text = p.DisplayName or p.Name; data.Name.Position = Vector2.new(pos.X, y - 18); data.Name.Color = Settings.ESPColor end
                            if Settings.Distance then data.Dist.Visible = true; data.Dist.Text = math.floor(mag) .. "m"; data.Dist.Position = Vector2.new(pos.X, y + height + 2); data.Dist.Color = Settings.ESPColor end
                            if Settings.Tracers and p ~= player then data.Tracer.Visible = true; data.Tracer.From = Vector2.new(camera.ViewportSize.X / 2, 0); data.Tracer.To = Vector2.new(pos.X, y); data.Tracer.Color = Settings.ESPColor end
                        end
                    end
                end
            end
        end
    end

    -- Espectar
    if Settings.SpectateEnabled and Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
        local targetHead = Settings.SelectedPlayer.Character:FindFirstChild("Head")
        if targetHead then
            camera.CameraType = Enum.CameraType.Scriptable
            local rot = CFrame.Angles(0, math.rad(Settings.SpectateRotation), 0)
            camera.CFrame = CFrame.new((targetHead.CFrame * rot * CFrame.new(0, 0, Settings.SpectateDist)).Position, targetHead.Position)
        end
    elseif not Settings.SpectateEnabled and camera.CameraType == Enum.CameraType.Scriptable then
        camera.CameraType = Enum.CameraType.Custom
    end
end)

-- --- LÓGICA DO SPINBOT (Misc) ---
task.spawn(function()
    while true do
        if Settings.SpinbotEnabled and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = player.Character.HumanoidRootPart
            hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(Settings.SpinbotSpeed), 0)
        end
        task.wait()
    end
end)

Players.PlayerRemoving:Connect(removeESP)

-- --- INTERFACE ---

-- AIMBOT
AimTab:CreateToggle({Name = "Aimbot", Default = false, Callback = function(v) Settings.AimbotEnabled = v end})
AimTab:CreateToggle({Name = "Show FOV", Default = false, Callback = function(v) Settings.ShowFOV = v end})
AimTab:CreateToggle({Name = "Team Check", Default = false, Callback = function(v) Settings.TeamCheck = v end})
AimTab:CreateKeybind({Name = "Keybind", Default = Enum.UserInputType.MouseButton2, Callback = function(v) Settings.AimKey = v end})
AimTab:CreateSlider({Name = "Smoothing", Min = 1, Max = 10, Default = 2, Callback = function(v) Settings.Smoothing = v end})
AimTab:CreateSlider({Name = "Fov Radius", Min = 50, Max = 500, Default = 100, Callback = function(v) Settings.FOVRadius = v end})
AimTab:CreateSlider({Name = "Alcance do Aim (M)", Min = 50, Max = 2000, Default = 500, Callback = function(v) Settings.AimDistance = v end})
AimTab:CreateColorPicker({Name = "Fov Color", Default = Settings.FOVColor, Callback = function(v) Settings.FOVColor = v end})

-- VISUALS
VisTab:CreateLabel("Componentes Visuais")
VisTab:CreateToggle({Name = "Usernames", Default = false, Callback = function(v) Settings.Names = v end})
VisTab:CreateToggle({Name = "Box Corner", Default = false, Callback = function(v) Settings.Box = v end})
VisTab:CreateToggle({Name = "Health Bar", Default = false, Callback = function(v) Settings.Health = v end})
VisTab:CreateToggle({Name = "Skeleton", Default = false, Callback = function(v) Settings.Skeleton = v end})
VisTab:CreateToggle({Name = "Tracers", Default = false, Callback = function(v) Settings.Tracers = v end})
VisTab:CreateToggle({Name = "Distance", Default = false, Callback = function(v) Settings.Distance = v end})
VisTab:CreateLabel("Filtros")
VisTab:CreateToggle({Name = "Show Local Player", Default = false, Callback = function(v) Settings.LocalPlayer = v end})
VisTab:CreateSlider({Name = "Alcance Máximo (M)", Min = 50, Max = 3500, Default = 500, Callback = function(v) Settings.MaxDistance = v end})
VisTab:CreateColorPicker({Name = "Cor do ESP", Default = Settings.ESPColor, Callback = function(v) Settings.ESPColor = v end})

-- PESSOAL / SELF
SelfTab:CreateToggle({Name = "Fly", Default = false, Callback = function(v) Settings.FlyEnabled = v; if not v then disableFly() end end})
SelfTab:CreateKeybind({Name = "Keybind", Default = Enum.KeyCode.CapsLock, Callback = function(key) Settings.FlyKey = key end})
SelfTab:CreateSlider({Name = "Speed Fly", Min = 10, Max = 300, Default = 20, Callback = function(v) Settings.FlySpeed = v end})

-- JOGADORES / PLAYERS
local SelectedLabel = PlayersTab:CreateLabel("Selecionado: Nenhum")

local PlayerGui = player:WaitForChild("PlayerGui")
local ListGui = create("ScreenGui", {Name = "Xanax_PlayerList", ResetOnSpawn = false, Parent = game:GetService("CoreGui") or PlayerGui})
local MainFrame = create("Frame", {Size = UDim2.new(0, 300, 0, 450), Position = UDim2.new(0.5, 380, 0.5, -225), BackgroundColor3 = Color3.fromRGB(18, 18, 22), Visible = false, Parent = ListGui})
create("UICorner", {CornerRadius = UDim.new(0, 10), Parent = MainFrame})
create("UIStroke", {Color = Color3.fromRGB(35, 35, 40), Thickness = 1.5, Parent = MainFrame})

-- Arrastar Frame
local dragging, dragInput, dragStart, startPos
MainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true; dragStart = input.Position; startPos = MainFrame.Position
        input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end)
    end
end)
MainFrame.InputChanged:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end end)
UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

local Scroll = create("ScrollingFrame", {Size = UDim2.new(1, -20, 1, -70), Position = UDim2.new(0, 10, 0, 55), BackgroundTransparency = 1, Parent = MainFrame})
create("UIListLayout", {Padding = UDim.new(0, 6), HorizontalAlignment = Enum.HorizontalAlignment.Center, Parent = Scroll})

local function RefreshList()
    for _, child in pairs(Scroll:GetChildren()) do if child:IsA("TextButton") then child:Destroy() end end
    for _, p in pairs(Players:GetPlayers()) do
        if p == player then continue end
        local btn = create("TextButton", {Size = UDim2.new(1, -5, 0, 38), BackgroundColor3 = Color3.fromRGB(25, 25, 30), Text = p.DisplayName or p.Name, TextColor3 = Color3.new(1,1,1), Parent = Scroll})
        create("UICorner", {CornerRadius = UDim.new(0, 6), Parent = btn})
        btn.MouseButton1Click:Connect(function()
            Settings.SelectedPlayer = p
            SelectedLabel:SetText("Selecionado: " .. (p.DisplayName or p.Name))
        end)
    end
    Scroll.CanvasSize = UDim2.new(0, 0, 0, Scroll.UIListLayout.AbsoluteContentSize.Y + 10)
end

PlayersTab:CreateButton({Name = "Show/Close Players List", Callback = function() MainFrame.Visible = not MainFrame.Visible; if MainFrame.Visible then RefreshList() end end})
PlayersTab:CreateButton({
    Name = "Teleport",
    Callback = function()
        if Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
            player.Character:MoveTo(Settings.SelectedPlayer.Character.HumanoidRootPart.Position)
        end
    end
})
PlayersTab:CreateToggle({Name = "Spectate", Default = false, Callback = function(v) Settings.SpectateEnabled = v end})
PlayersTab:CreateToggle({Name = "Teleport To Me", Default = false, Callback = function(v) Settings.PuxarLoop = v end})

-- MISC (NOVO)
MiscTab:CreateLabel("Configurações Extras")
MiscTab:CreateToggle({
    Name = "Spinbot", 
    Default = false, 
    Callback = function(v) 
        Settings.SpinbotEnabled = v 
    end
})
MiscTab:CreateSlider({
    Name = "Spin Velocity", 
    Min = 10, 
    Max = 300, 
    Default = 50, 
    Callback = function(v) 
        Settings.SpinbotSpeed = v 
    end
})

-- Loop de Puxar
task.spawn(function()
    while true do
        if Settings.PuxarLoop and Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
            local targetHRP = Settings.SelectedPlayer.Character:FindFirstChild("HumanoidRootPart")
            local myHRP = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            if targetHRP and myHRP then
                targetHRP.CFrame = myHRP.CFrame * CFrame.new(0, 0, -3)
            end
        end
        task.wait(0.1)
    end
end)
