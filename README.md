local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- --- CONFIGURAÇÕES GERAIS ---
local Settings = {
    -- Aimbot
    AimbotEnabled = false,
    AimbotNPCs = false,
    AimMethod = "Mouse",
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
    BoxMode = "2D",
    BoxStyle = "Cornered", 
    Skeleton = false,
    Tracers = false,
    Distance = false,
    Names = false,
    Health = false,
    LocalPlayer = false,
    VisualsNPCs = false,
    ESPColor = Color3.fromRGB(255, 0, 85),
    Thickness = 1,
    BoxThickness = 2,
    MaxDistance = 500,
    -- Exploits / Fly
    FlyEnabled = false,
    IsFlying = false,
    FlySpeed = 20,
    FlyBoost = 350,
    FlyKey = Enum.KeyCode.CapsLock,
    FlyInvisible = false,
    InfJump = false,
    -- Players
    SelectedPlayer = nil,
    PuxarLoop = false,
    SpectateEnabled = false,
    SpectateDist = 15,
    SpectateRotation = 0,
    FollowPlayer = false, -- Renomeado
    EatPlayer = false, -- Nova funcionalidade
    -- Misc
    SpinbotEnabled = false,
    SpinbotSpeed = 50
}

local Cache = {}
local NPCCacheList = {} -- Cache otimizado para NPCs
local isAiming = false
local bodyVelocity, bodyGyro, flyConnection
local isUnloaded = false
local Hub = nil

-- --- FUNÇÕES DE AUXÍLIO PARA NPCs ---
local function IsNPC(model)
    if not model:IsA("Model") then return false end
    if Players:GetPlayerFromCharacter(model) then return false end
    if model:FindFirstChildOfClass("Humanoid") and model:FindFirstChild("HumanoidRootPart") then
        return true
    end
    return false
end

-- Loop de atualização de cache de NPCs (Evita o crash de GetDescendants no loop principal)
task.spawn(function()
    while not isUnloaded do
        local tempNPCs = {}
        -- Procura em lugares comuns para NPCs para ser mais rápido que GetDescendants
        for _, obj in pairs(workspace:GetChildren()) do
            if IsNPC(obj) then
                table.insert(tempNPCs, obj)
            elseif obj:IsA("Folder") or obj:IsA("Model") then
                -- Checa um nível abaixo para NPCs dentro de pastas/grupos
                for _, sub in pairs(obj:GetChildren()) do
                    if IsNPC(sub) then table.insert(tempNPCs, sub) end
                end
            end
        end
        NPCCacheList = tempNPCs
        task.wait(2) -- Atualiza a lista a cada 2 segundos
    end
end)

local function GetValidTargets(includePlayers, includeNPCs)
    local targets = {}
    
    if includePlayers then
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= player and p.Character then
                table.insert(targets, {Character = p.Character, Player = p})
            end
        end
    end
    
    if includeNPCs then
        for _, npc in pairs(NPCCacheList) do
            if npc and npc.Parent then
                table.insert(targets, {Character = npc, Player = nil})
            end
        end
    end
    
    return targets
end

-- --- CONFIGURAÇÃO DE SALVAMENTO ---
local function SaveConfig()
    local save = {}
    for k, v in pairs(Settings) do
        if typeof(v) == "Color3" then
            save[k] = {Type = "Color3", R = v.R, G = v.G, B = v.B}
        elseif typeof(v) == "EnumItem" then
            save[k] = {Type = "EnumItem", EnumType = tostring(v.EnumType), Name = v.Name}
        elseif type(v) ~= "userdata" and type(v) ~= "function" and type(v) ~= "table" then
            save[k] = v
        end
    end
    if writefile then
        writefile("XanaxHub_Config.json", HttpService:JSONEncode(save))
    end
end

local function LoadConfig()
    if isfile and isfile("XanaxHub_Config.json") then
        local s, data = pcall(function() return HttpService:JSONDecode(readfile("XanaxHub_Config.json")) end)
        if s and type(data) == "table" then
            for k, v in pairs(data) do
                if type(v) == "table" and v.Type == "Color3" then
                    Settings[k] = Color3.new(v.R, v.G, v.B)
                elseif type(v) == "table" and v.Type == "EnumItem" then
                    pcall(function() Settings[k] = Enum[tostring(v.EnumType)][v.Name] end)
                else
                    Settings[k] = v
                end
            end
            if Hub and Hub.UpdateFlags then Hub:UpdateFlags(Settings) end
        end
    end
end

-- Carregando a Lib do Hub
Hub = loadstring(game:HttpGet("https://raw.githubusercontent.com/Stand-Software/hub/refs/heads/main/README.md"))()

local Window = Hub:CreateWindow({
    Title = "Xanax Hub"
})

local AimTab = Window:CreateTab("Aimbot")
local VisTab = Window:CreateTab("Visuals")
local SelfTab = Window:CreateTab("Exploits")
local PlayersTab = Window:CreateTab("Players")
local MiscTab = Window:CreateTab("Misc")

-- --- OBJETOS DE DESENHO (FOV) ---
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 1
FOVCircle.NumSides = 100
FOVCircle.Filled = false
FOVCircle.Transparency = 1
FOVCircle.Visible = false

-- --- FUNÇÕES VISUAIS ---
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

local function removeESP(id)
    if Cache[id] then
        for _, obj in pairs(Cache[id]) do
            if type(obj) == "table" then
                for _, subObj in pairs(obj) do subObj:Remove() end
            else
                obj:Remove()
            end
        end
        Cache[id] = nil
    end
end

-- --- LÓGICA DE VISIBILIDADE (Para Fly Invisible) ---
local function updateCharacterVisibility()
    local char = player.Character
    if not char then return end
    local transparencyValue = (Settings.IsFlying and Settings.FlyInvisible) and 1 or 0
    for _, part in ipairs(char:GetDescendants()) do
        if (part:IsA("BasePart") and part.Name ~= "HumanoidRootPart") or part:IsA("Decal") then
            part.Transparency = transparencyValue
        end
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
    updateCharacterVisibility()
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

    updateCharacterVisibility()

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
local function getClosestTarget()
    local target = nil
    local shortestDistance = math.huge
    local detectionOrigin = (Settings.AimMethod == "Camera") and (camera.ViewportSize / 2) or UserInputService:GetMouseLocation()
    local potentialTargets = GetValidTargets(true, Settings.AimbotNPCs)

    for _, data in pairs(potentialTargets) do
        local char = data.Character
        local p = data.Player
        if p and Settings.TeamCheck and p.Team == player.Team then continue end

        local part = char:FindFirstChild(Settings.TargetPart)
        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")

        if part and hum and hum.Health > 0 and hrp then
            local mag = (hrp.Position - camera.CFrame.Position).Magnitude
            if mag <= Settings.AimDistance then
                local screenPos, onScreen = camera:WorldToViewportPoint(part.Position)
                if onScreen then
                    local distFromOrigin = (Vector2.new(screenPos.X, screenPos.Y) - detectionOrigin).Magnitude
                    if distFromOrigin <= Settings.FOVRadius then
                        if mag < shortestDistance then
                            target = part
                            shortestDistance = mag
                        end
                    end
                end
            end
        end
    end
    return target
end

-- Detecção de Input
local inputBeganConn = UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then isAiming = true end
    
    if Settings.FlyEnabled and input.KeyCode == Settings.FlyKey then
        Settings.IsFlying = not Settings.IsFlying
        if Settings.IsFlying then enableFly() else disableFly() end
    end
end)

local inputEndedConn = UserInputService.InputEnded:Connect(function(input)
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

-- --- LOOP PRINCIPAL ---
local renderConn = RunService.RenderStepped:Connect(function()
    FOVCircle.Visible = Settings.ShowFOV
    FOVCircle.Radius = Settings.FOVRadius
    FOVCircle.Color = Settings.FOVColor
    
    if Settings.AimMethod == "Camera" then
        FOVCircle.Position = camera.ViewportSize / 2
    else
        FOVCircle.Position = UserInputService:GetMouseLocation()
    end

    if Settings.AimbotEnabled and isAiming then
        local target = getClosestTarget()
        if target then
            local screenPos, onScreen = camera:WorldToViewportPoint(target.Position)
            if onScreen then
                if Settings.AimMethod == "Mouse" then
                    local mousePos = UserInputService:GetMouseLocation()
                    local moveX = (screenPos.X - mousePos.X) / Settings.Smoothing
                    local moveY = (screenPos.Y - mousePos.Y) / Settings.Smoothing
                    if mousemoverel then mousemoverel(moveX, moveY) end
                elseif Settings.AimMethod == "Camera" then
                    camera.CFrame = camera.CFrame:Lerp(CFrame.new(camera.CFrame.Position, target.Position), 1 / Settings.Smoothing)
                end
            end
        end
    end

    for _, data in pairs(Cache) do hideAll(data) end

    local currentTargets = GetValidTargets(true, Settings.VisualsNPCs)
    if Settings.LocalPlayer and player.Character then
        table.insert(currentTargets, {Character = player.Character, Player = player})
    end

    for _, targetData in pairs(currentTargets) do
        local char = targetData.Character
        local p = targetData.Player
        local id = p and p.UserId or char:GetAttribute("NPC_ID") or char.Name..char:GetDebugId()

        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")

        if hrp and hum and hum.Health > 0 then
            local mag = (hrp.Position - camera.CFrame.Position).Magnitude
            if mag <= Settings.MaxDistance then
                local pos, onScreen = camera:WorldToViewportPoint(hrp.Position)

                if onScreen then
                    if not Cache[id] then
                        Cache[id] = {
                            Box = {
                                createDrawing("Line",{}), createDrawing("Line",{}),
                                createDrawing("Line",{}), createDrawing("Line",{}),
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

                    local data = Cache[id]
                    local head = char:FindFirstChild("Head")
                    local headPos = head and camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
                    local legPos = camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))

                    if headPos and legPos then
                        local height = math.abs(headPos.Y - legPos.Y)
                        local width = height / 1.5
                        local x, y = pos.X - width / 2, pos.Y - height / 2
                        local l = width / 4

                        if Settings.Box then
                            local box = data.Box
                            if Settings.BoxMode == "3D" then
                                local cframe, size = char:GetBoundingBox()
                                if size.Magnitude == 0 then size = Vector3.new(4, 5.5, 4) end
                                local c = {
                                    cframe * CFrame.new(size.X/2, size.Y/2, size.Z/2),
                                    cframe * CFrame.new(-size.X/2, size.Y/2, size.Z/2),
                                    cframe * CFrame.new(-size.X/2, -size.Y/2, size.Z/2),
                                    cframe * CFrame.new(size.X/2, -size.Y/2, size.Z/2),
                                    cframe * CFrame.new(size.X/2, size.Y/2, -size.Z/2),
                                    cframe * CFrame.new(-size.X/2, size.Y/2, -size.Z/2),
                                    cframe * CFrame.new(-size.X/2, -size.Y/2, -size.Z/2),
                                    cframe * CFrame.new(size.X/2, -size.Y/2, -size.Z/2)
                                }
                                local pts = {}
                                local allOnScreen = true
                                for i=1, 8 do
                                    local pt, on = camera:WorldToViewportPoint(c[i].Position)
                                    pts[i] = Vector2.new(pt.X, pt.Y)
                                    if not on or pt.Z < 0 then allOnScreen = false end
                                end
                                if allOnScreen then
                                    local edges = {
                                        {1,2}, {2,3}, {3,4}, {4,1},
                                        {5,6}, {6,7}, {7,8}, {8,5},
                                        {1,5}, {2,6}, {3,7}, {4,8}
                                    }
                                    for i=1, 12 do
                                        box[i].From = pts[edges[i][1]]
                                        box[i].To = pts[edges[i][2]]
                                        box[i].Visible = true
                                        box[i].Color = Settings.ESPColor
                                        box[i].Thickness = Settings.BoxThickness
                                    end
                                else
                                    for i=1, 12 do box[i].Visible = false end
                                end
                            else
                                if Settings.BoxStyle == "Full" then
                                    local tl, tr, bl, br = Vector2.new(x, y), Vector2.new(x + width, y), Vector2.new(x, y + height), Vector2.new(x + width, y + height)
                                    box[1].From = tl; box[1].To = tr; box[1].Visible = true
                                    box[2].From = bl; box[2].To = br; box[2].Visible = true
                                    box[3].From = tl; box[3].To = bl; box[3].Visible = true
                                    box[4].From = tr; box[4].To = br; box[4].Visible = true
                                    for i=1, 4 do box[i].Color = Settings.ESPColor; box[i].Thickness = Settings.BoxThickness end
                                    for i=5, 12 do box[i].Visible = false end
                                else
                                    local tl, tr, bl, br = Vector2.new(x, y), Vector2.new(x + width, y), Vector2.new(x, y + height), Vector2.new(x + width, y + height)
                                    box[1].From = tl; box[1].To = tl + Vector2.new(l, 0)
                                    box[2].From = tl; box[2].To = tl + Vector2.new(0, l)
                                    box[3].From = tr; box[3].To = tr - Vector2.new(l, 0)
                                    box[4].From = tr; box[4].To = tr + Vector2.new(0, l)
                                    box[5].From = bl; box[5].To = bl + Vector2.new(l, 0)
                                    box[6].From = bl; box[6].To = bl - Vector2.new(0, l)
                                    box[7].From = br; box[7].To = br - Vector2.new(l, 0)
                                    box[8].From = br; box[8].To = br - Vector2.new(0, l)
                                    for i=1, 8 do 
                                        box[i].Visible = true
                                        box[i].Color = Settings.ESPColor
                                        box[i].Thickness = Settings.BoxThickness 
                                    end
                                    for i=9, 12 do box[i].Visible = false end
                                end
                            end
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

                        if Settings.Names then 
                            data.Name.Visible = true
                            data.Name.Text = (p and (p.DisplayName or p.Name)) or ("[NPC] " .. char.Name)
                            data.Name.Position = Vector2.new(pos.X, y - 18)
                            data.Name.Color = Settings.ESPColor 
                        end
                        if Settings.Distance then 
                            data.Dist.Visible = true
                            data.Dist.Text = math.floor(mag) .. "m"
                            data.Dist.Position = Vector2.new(pos.X, y + height + 2)
                            data.Dist.Color = Settings.ESPColor 
                        end
                        if Settings.Tracers and char ~= player.Character then 
                            data.Tracer.Visible = true
                            data.Tracer.From = Vector2.new(camera.ViewportSize.X / 2, 0)
                            data.Tracer.To = Vector2.new(pos.X, y)
                            data.Tracer.Color = Settings.ESPColor 
                        end
                    end
                end
            end
        end
    end

    if Settings.SpectateEnabled and Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
        local targetHum = Settings.SelectedPlayer.Character:FindFirstChildOfClass("Humanoid")
        if targetHum and camera.CameraSubject ~= targetHum then
            camera.CameraSubject = targetHum
        end
    elseif not Settings.SpectateEnabled then
        local myHum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if myHum and camera.CameraSubject ~= myHum then
            camera.CameraSubject = myHum
        end
    end
end)

-- --- LÓGICA EXTRAS ---
task.spawn(function()
    while not isUnloaded do
        if Settings.SpinbotEnabled and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = player.Character.HumanoidRootPart
            hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(Settings.SpinbotSpeed), 0)
        end
        task.wait()
    end
end)

-- Loop de Interação com Player (Seguir / Comer)
task.spawn(function()
    while not isUnloaded do
        if Settings.SelectedPlayer and Settings.SelectedPlayer.Character and player.Character then
            local myHrp = player.Character:FindFirstChild("HumanoidRootPart")
            local targetHrp = Settings.SelectedPlayer.Character:FindFirstChild("HumanoidRootPart")
            
            if myHrp and targetHrp then
                if Settings.FollowPlayer then
                    -- Seguir Player: Distância fixa
                    myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, 1.5)
                elseif Settings.EatPlayer then
                    -- Comer Player: Grudado e oscilando para frente/trás
                    local oscillation = math.sin(tick() * 15) * 1.3 -- Velocidade e amplitude do movimento
                    myHrp.CFrame = targetHrp.CFrame * CFrame.new(0, 0, 0.5 + oscillation)
                end
            end
        end
        task.wait()
    end
end)

local playerRemovedConn = Players.PlayerRemoving:Connect(function(p) removeESP(p.UserId) end)

-- --- INTERFACE ---

-- AIMBOT
AimTab:CreateToggle({Name = "Aimbot", Flag = "AimbotEnabled", Default = false, Callback = function(v) Settings.AimbotEnabled = v end})
AimTab:CreateToggle({Name = "Show FOV", Flag = "ShowFOV", Default = false, Callback = function(v) Settings.ShowFOV = v end})
AimTab:CreateToggle({Name = "Enable NPC", Flag = "AimbotNPCs", Default = false, Callback = function(v) Settings.AimbotNPCs = v end})
AimTab:CreateToggle({Name = "Team Check", Flag = "TeamCheck", Default = false, Callback = function(v) Settings.TeamCheck = v end})
AimTab:CreateKeybind({Name = "Keybind", Flag = "AimKey", Default = Enum.UserInputType.MouseButton2, Callback = function(v) Settings.AimKey = v end})
AimTab:CreateDropdown({Name = "Aim Method", Flag = "AimMethod", Options = {"Mouse", "Camera"}, Default = "Mouse", Callback = function(v) Settings.AimMethod = v end})
AimTab:CreateSlider({Name = "Smoothing", Flag = "Smoothing", Min = 1, Max = 10, Default = 2, Callback = function(v) Settings.Smoothing = v end})
AimTab:CreateSlider({Name = "Fov Radius", Flag = "FOVRadius", Min = 50, Max = 500, Default = 100, Callback = function(v) Settings.FOVRadius = v end})
AimTab:CreateSlider({Name = "Alcance do Aim (M)", Flag = "AimDistance", Min = 50, Max = 2000, Default = 500, Callback = function(v) Settings.AimDistance = v end})
AimTab:CreateColorPicker({Name = "Fov Color", Flag = "FOVColor", Default = Settings.FOVColor, Callback = function(v) Settings.FOVColor = v end})

-- VISUALS
VisTab:CreateLabel("Elementos Visuais")
VisTab:CreateToggle({Name = "Usernames", Flag = "Names", Default = false, Callback = function(v) Settings.Names = v end})
VisTab:CreateToggle({Name = "Enable NPC", Flag = "VisualsNPCs", Default = false, Callback = function(v) Settings.VisualsNPCs = v end})
VisTab:CreateToggle({Name = "Health Bar", Flag = "Health", Default = false, Callback = function(v) Settings.Health = v end})
VisTab:CreateToggle({Name = "Skeleton", Flag = "Skeleton", Default = false, Callback = function(v) Settings.Skeleton = v end})
VisTab:CreateToggle({Name = "Tracers", Flag = "Tracers", Default = false, Callback = function(v) Settings.Tracers = v end})
VisTab:CreateToggle({Name = "Distance", Flag = "Distance", Default = false, Callback = function(v) Settings.Distance = v end})

VisTab:CreateLabel("Configurações de Box")
VisTab:CreateToggle({Name = "Box ESP", Flag = "Box", Default = false, Callback = function(v) Settings.Box = v end})
VisTab:CreateDropdown({Name = "Box Mode", Flag = "BoxMode", Options = {"2D", "3D"}, Default = "2D", Callback = function(v) Settings.BoxMode = v end})
VisTab:CreateDropdown({Name = "Box Style (2D)", Flag = "BoxStyle", Options = {"Cornered", "Full"}, Default = "Full", Callback = function(v) Settings.BoxStyle = v end})

VisTab:CreateLabel("Filtros e Alcance")
VisTab:CreateToggle({Name = "Show Local Player", Flag = "LocalPlayer", Default = false, Callback = function(v) Settings.LocalPlayer = v end})
VisTab:CreateSlider({Name = "Alcance Máximo (M)", Flag = "MaxDistance", Min = 50, Max = 3500, Default = 500, Callback = function(v) Settings.MaxDistance = v end})

VisTab:CreateLabel("Personalização")
VisTab:CreateColorPicker({Name = "Cor do ESP", Flag = "ESPColor", Default = Settings.ESPColor, Callback = function(v) Settings.ESPColor = v end})

-- Exploits / SELF
SelfTab:CreateToggle({Name = "Fly", Flag = "FlyEnabled", Default = false, Callback = function(v) Settings.FlyEnabled = v; if not v then disableFly() end end})
SelfTab:CreateToggle({Name = "Fly Invisible", Flag = "FlyInvisible", Default = false, Callback = function(v) Settings.FlyInvisible = v updateCharacterVisibility() end})
SelfTab:CreateKeybind({Name = "Keybind", Flag = "FlyKey", Default = Enum.KeyCode.CapsLock, Callback = function(key) Settings.FlyKey = key end})
SelfTab:CreateSlider({Name = "Speed Fly", Flag = "FlySpeed", Min = 10, Max = 300, Default = 20, Callback = function(v) Settings.FlySpeed = v end})

-- Players / PLAYERS
local playerOptions = {"Nenhum"}
for _, p in pairs(Players:GetPlayers()) do
    if p ~= player then table.insert(playerOptions, p.DisplayName or p.Name) end
end

local PlayerDropdown = PlayersTab:CreateDropdown({
    Name = "Select Player",
    Options = playerOptions,
    Default = "Nenhum",
    Callback = function(v)
        if v == "Nenhum" then
            Settings.SelectedPlayer = nil
        else
            for _, p in pairs(Players:GetPlayers()) do
                if (p.DisplayName == v or p.Name == v) and p ~= player then
                    Settings.SelectedPlayer = p
                    break
                end
            end
        end
    end
})

PlayersTab:CreateButton({
    Name = "Refresh Player List",
    Callback = function()
        local newOpts = {"Nenhum"}
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= player then table.insert(newOpts, p.DisplayName or p.Name) end
        end
        PlayerDropdown:Refresh(newOpts)
    end
})

PlayersTab:CreateButton({
    Name = "Teleport",
    Callback = function()
        if Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
            player.Character:MoveTo(Settings.SelectedPlayer.Character.HumanoidRootPart.Position)
        end
    end
})
PlayersTab:CreateToggle({Name = "Spectate", Flag = "SpectateEnabled", Default = false, Callback = function(v) Settings.SpectateEnabled = v end})
PlayersTab:CreateToggle({Name = "Teleport To Me", Flag = "PuxarLoop", Default = false, Callback = function(v) Settings.PuxarLoop = v end})
PlayersTab:CreateToggle({Name = "Seguir Player", Flag = "FollowPlayer", Default = false, Callback = function(v) Settings.FollowPlayer = v end})
PlayersTab:CreateToggle({Name = "Comer Player", Flag = "EatPlayer", Default = false, Callback = function(v) Settings.EatPlayer = v end})

-- MISC
MiscTab:CreateLabel("Configurações Extras")
MiscTab:CreateToggle({Name = "Spinbot", Flag = "SpinbotEnabled", Default = false, Callback = function(v) Settings.SpinbotEnabled = v end})
MiscTab:CreateSlider({Name = "Spin Velocity", Flag = "SpinbotSpeed", Min = 10, Max = 300, Default = 50, Callback = function(v) Settings.SpinbotSpeed = v end})

MiscTab:CreateLabel("Configurações")
MiscTab:CreateButton({Name = "Save Config", Callback = SaveConfig})
MiscTab:CreateButton({Name = "Load Config", Callback = LoadConfig})

MiscTab:CreateButton({
    Name = "Unload Hub",
    Callback = function()
        isUnloaded = true
        if renderConn then renderConn:Disconnect() end
        if inputBeganConn then inputBeganConn:Disconnect() end
        if inputEndedConn then inputEndedConn:Disconnect() end
        if playerRemovedConn then playerRemovedConn:Disconnect() end
        
        disableFly()
        for _, data in pairs(Cache) do hideAll(data) end
        Cache = {}
        
        if FOVCircle then FOVCircle:Remove() end
        
        local coreGui = game:GetService("CoreGui")
        local pg = player:FindFirstChild("PlayerGui")
        
        local hubGui = coreGui:FindFirstChild("XanaxHub_V2") or (pg and pg:FindFirstChild("XanaxHub_V2"))
        if hubGui then hubGui:Destroy() end
        
        camera.CameraType = Enum.CameraType.Custom
    end
})

-- Loop de Puxar
task.spawn(function()
    while not isUnloaded do
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
