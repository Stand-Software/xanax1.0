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
    VisibleCheck = false,
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
    VisTeamCheck = false,
    ESPColor = Color3.fromRGB(255, 0, 85),
    Thickness = 1,
    BoxThickness = 2,
    MaxDistance = 500,
    -- Exploits / Fly
    FlyEnabled = false,
    IsFlying = false,
    FlyMethod = "Classic", 
    FlySpeed = 20,
    FlyBoost = 350,
    FlyKey = Enum.KeyCode.CapsLock,
    FlyInvisible = false,
    -- Exploits / Freecam
    FreecamEnabled = false,
    IsFreecamming = false,
    FreecamSpeed = 20,
    FreecamKey = Enum.KeyCode.P,
    -- Players
    SelectedPlayer = nil,
    PuxarLoop = false,
    SpectateEnabled = false,
    SpectateDist = 15,
    SpectateRotation = 0,
    FollowPlayer = false,
    EatPlayer = false,
    -- Misc
    SpinbotEnabled = false,
    SpinbotSpeed = 50
}

local Cache = {}
local NPCCacheList = {}
local isAiming = false
local bodyVelocity, bodyGyro, flyConnection, freecamConnection
local isUnloaded = false
local Hub = nil
local camRotX, camRotY = 0, 0

-- --- FUNÇÕES DE AUXÍLIO ---

local function IsVisible(part, character)
    if not part then return false end
    local char = player.Character
    if not char then return false end
    local origin = camera.CFrame.Position
    local direction = (part.Position - origin).Unit * (part.Position - origin).Magnitude
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = {char, camera}
    rayParams.FilterType = Enum.RaycastFilterType.Blacklist
    local result = workspace:Raycast(origin, direction, rayParams)
    return (result and result.Instance:IsDescendantOf(character)) or false
end

local function IsNPC(model)
    if not model:IsA("Model") then return false end
    if Players:GetPlayerFromCharacter(model) then return false end
    return model:FindFirstChildOfClass("Humanoid") and model:FindFirstChild("HumanoidRootPart")
end

task.spawn(function()
    while not isUnloaded do
        local tempNPCs = {}
        for _, obj in pairs(workspace:GetChildren()) do
            if IsNPC(obj) then table.insert(tempNPCs, obj)
            elseif obj:IsA("Folder") or obj:IsA("Model") then
                for _, sub in pairs(obj:GetChildren()) do if IsNPC(sub) then table.insert(tempNPCs, sub) end end
            end
        end
        NPCCacheList = tempNPCs
        task.wait(2)
    end
end)

local function GetValidTargets(includePlayers, includeNPCs)
    local targets = {}
    if includePlayers then
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= player and p.Character then
                if Settings.VisTeamCheck and p.Team == player.Team then continue end
                table.insert(targets, {Character = p.Character, Player = p})
            end
        end
    end
    if includeNPCs then
        for _, npc in pairs(NPCCacheList) do
            if npc and npc.Parent then table.insert(targets, {Character = npc, Player = nil}) end
        end
    end
    return targets
end

local function SaveConfig()
    local save = {}
    for k, v in pairs(Settings) do
        if typeof(v) == "Color3" then save[k] = {Type = "Color3", R = v.R, G = v.G, B = v.B}
        elseif typeof(v) == "EnumItem" then save[k] = {Type = "EnumItem", EnumType = tostring(v.EnumType), Name = v.Name}
        elseif type(v) ~= "userdata" and type(v) ~= "function" and type(v) ~= "table" then save[k] = v end
    end
    if writefile then writefile("XanaxHub_Config.json", HttpService:JSONEncode(save)) end
end

local function LoadConfig()
    if isfile and isfile("XanaxHub_Config.json") then
        local s, data = pcall(function() return HttpService:JSONDecode(readfile("XanaxHub_Config.json")) end)
        if s and type(data) == "table" then
            for k, v in pairs(data) do
                if type(v) == "table" and v.Type == "Color3" then Settings[k] = Color3.new(v.R, v.G, v.B)
                elseif type(v) == "table" and v.Type == "EnumItem" then pcall(function() Settings[k] = Enum[tostring(v.EnumType)][v.Name] end)
                else Settings[k] = v end
            end
            if Hub and Hub.UpdateFlags then Hub:UpdateFlags(Settings) end
        end
    end
end

-- Carregando Hub (Substitua o link abaixo pelo seu link do Pastebin raw)
Hub = loadstring(game:HttpGet("https://pastebin.com/raw/cKgUQXMr"))()
local Window = Hub:CreateWindow({Title = "Xanax Hub V2"})
local AimTab = Window:CreateTab("Aimbot")
local VisTab = Window:CreateTab("Visuals")
local SelfTab = Window:CreateTab("Exploits")
local PlayersTab = Window:CreateTab("Players")
local MiscTab = Window:CreateTab("Misc")

local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness, FOVCircle.NumSides, FOVCircle.Filled, FOVCircle.Transparency, FOVCircle.Visible = 1, 100, false, 1, false

local function createDrawing(class, props)
    local obj = Drawing.new(class)
    for i, v in pairs(props) do obj[i] = v end
    return obj
end

local function hideAll(data)
    if not data then return end
    for _, l in pairs(data.Box) do l.Visible = false end
    for _, l in pairs(data.Skeleton) do l.Visible = false end
    data.Tracer.Visible, data.Name.Visible, data.Dist.Visible, data.HealthBarBG.Visible, data.HealthBarMain.Visible = false, false, false, false, false
end

local function removeESP(id)
    if Cache[id] then
        for _, obj in pairs(Cache[id]) do
            if type(obj) == "table" then for _, subObj in pairs(obj) do subObj:Remove() end else obj:Remove() end
        end
        Cache[id] = nil
    end
end

local function updateCharacterVisibility()
    local char = player.Character
    if not char then return end
    local trans = (Settings.IsFlying and Settings.FlyInvisible) and 1 or 0
    for _, part in ipairs(char:GetDescendants()) do
        if (part:IsA("BasePart") and part.Name ~= "HumanoidRootPart") or part:IsA("Decal") then part.Transparency = trans end
    end
end

local function disableFly()
    Settings.IsFlying = false
    if flyConnection then flyConnection:Disconnect(); flyConnection = nil end
    local char = player.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum.PlatformStand = false; hum:ChangeState(Enum.HumanoidStateType.Running) end
        if bodyVelocity then bodyVelocity:Destroy(); bodyVelocity = nil end
        if bodyGyro then bodyGyro:Destroy(); bodyGyro = nil end
        for _, part in ipairs(char:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = true end end
    end
    updateCharacterVisibility()
end

local function enableFly()
    local char = player.Character or player.CharacterAdded:Wait()
    local hum = char:WaitForChild("Humanoid")
    local root = char:WaitForChild("HumanoidRootPart")
    if Settings.FlyMethod == "Classic" then
        hum.PlatformStand = true
        bodyVelocity = Instance.new("BodyVelocity")
        bodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bodyVelocity.Parent = root
        bodyGyro = Instance.new("BodyGyro")
        bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
        bodyGyro.P, bodyGyro.D, bodyGyro.Parent = 1000, 100, root
    end
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
        if Settings.FlyMethod == "Classic" then
            bodyVelocity.Velocity = camera.CFrame:VectorToWorldSpace(moveDir) * speed
            bodyGyro.CFrame = camera.CFrame
        elseif Settings.FlyMethod == "Bypass" then
            hum:ChangeState(Enum.HumanoidStateType.Swimming)
            root.Velocity = camera.CFrame:VectorToWorldSpace(moveDir) * speed
        end
        for _, part in ipairs(char:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = false end end
    end)
end

local function toggleFreecam(enable)
    Settings.IsFreecamming = enable
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if enable then
        if hrp then hrp.Anchored = true end
        camera.CameraType = Enum.CameraType.Scriptable
        local look = camera.CFrame.LookVector
        camRotX, camRotY = math.asin(look.Y), math.atan2(-look.X, -look.Z)
        freecamConnection = RunService.RenderStepped:Connect(function()
            if not Settings.IsFreecamming then return end
            if UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
                UserInputService.MouseBehavior = Enum.MouseBehavior.LockCurrentPosition
                local delta = UserInputService:GetMouseDelta()
                camRotY = camRotY - (delta.X * 0.005)
                camRotX = math.clamp(camRotX - (delta.Y * 0.005), -math.rad(89), math.rad(89))
            else UserInputService.MouseBehavior = Enum.MouseBehavior.Default end
            camera.CFrame = CFrame.new(camera.CFrame.Position) * CFrame.Angles(0, camRotY, 0) * CFrame.Angles(camRotX, 0, 0)
            local moveDir = Vector3.new()
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir += Vector3.new(0,0,-1) end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir += Vector3.new(0,0,1) end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir += Vector3.new(-1,0,0) end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir += Vector3.new(1,0,0) end
            if UserInputService:IsKeyDown(Enum.KeyCode.E) then moveDir += Vector3.new(0,1,0) end
            if UserInputService:IsKeyDown(Enum.KeyCode.Q) then moveDir += Vector3.new(0,-1,0) end
            local speed = UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) and (Settings.FreecamSpeed * 2) or Settings.FreecamSpeed
            camera.CFrame = camera.CFrame + camera.CFrame:VectorToWorldSpace(moveDir) * (speed / 10)
        end)
    else
        if hrp then hrp.Anchored = false end
        if freecamConnection then freecamConnection:Disconnect(); freecamConnection = nil end
        camera.CameraType, UserInputService.MouseBehavior = Enum.CameraType.Custom, Enum.MouseBehavior.Default
    end
end

local function getClosestTarget()
    local target, shortestDistance = nil, math.huge
    local detectionOrigin = (Settings.AimMethod == "Camera") and (camera.ViewportSize / 2) or UserInputService:GetMouseLocation()
    for _, data in pairs(GetValidTargets(true, Settings.AimbotNPCs)) do
        local char, p = data.Character, data.Player
        local part, hum, hrp = char:FindFirstChild(Settings.TargetPart), char:FindFirstChildOfClass("Humanoid"), char:FindFirstChild("HumanoidRootPart")
        if part and hum and hum.Health > 0 and hrp then
            if Settings.VisibleCheck and not IsVisible(part, char) then continue end
            local mag = (hrp.Position - camera.CFrame.Position).Magnitude
            if mag <= Settings.AimDistance then
                local screenPos, onScreen = camera:WorldToViewportPoint(part.Position)
                if onScreen then
                    local distFromOrigin = (Vector2.new(screenPos.X, screenPos.Y) - detectionOrigin).Magnitude
                    if distFromOrigin <= Settings.FOVRadius and mag < shortestDistance then target, shortestDistance = part, mag end
                end
            end
        end
    end
    return target
end

local inputBeganConn = UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then isAiming = true end
    if Settings.FlyEnabled and input.KeyCode == Settings.FlyKey then
        Settings.IsFlying = not Settings.IsFlying
        if Settings.IsFlying then if Settings.IsFreecamming then toggleFreecam(false) end; enableFly() else disableFly() end
    end
    if Settings.FreecamEnabled and input.KeyCode == Settings.FreecamKey then
        Settings.IsFreecamming = not Settings.IsFreecamming
        if Settings.IsFreecamming then if Settings.IsFlying then disableFly() end; toggleFreecam(true) else toggleFreecam(false) end
    end
end)

local inputEndedConn = UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then isAiming = false end
end)

local Bones = {
    R15 = {{"UpperTorso", "Head"}, {"UpperTorso", "LowerTorso"}, {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"}, {"LeftLowerLeg", "LeftFoot"}, {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}, {"RightLowerLeg", "RightFoot"}, {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"}, {"LeftLowerArm", "LeftHand"}, {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"}, {"RightLowerArm", "Hand"}},
    R6 = {{"Torso", "Head"}, {"Torso", "Left Arm"}, {"Torso", "Right Arm"}, {"Torso", "Left Leg"}, {"Torso", "Right Leg"}}
}

local renderConn = RunService.RenderStepped:Connect(function()
    FOVCircle.Visible, FOVCircle.Radius, FOVCircle.Color = Settings.ShowFOV, Settings.FOVRadius, Settings.FOVColor
    FOVCircle.Position = (Settings.AimMethod == "Camera") and (camera.ViewportSize / 2) or UserInputService:GetMouseLocation()

    if Settings.AimbotEnabled and isAiming then
        local target = getClosestTarget()
        if target then
            local screenPos, onScreen = camera:WorldToViewportPoint(target.Position)
            if onScreen then
                if Settings.AimMethod == "Mouse" then
                    local mousePos = UserInputService:GetMouseLocation()
                    if mousemoverel then mousemoverel((screenPos.X - mousePos.X) / Settings.Smoothing, (screenPos.Y - mousePos.Y) / Settings.Smoothing) end
                elseif Settings.AimMethod == "Camera" then
                    camera.CFrame = camera.CFrame:Lerp(CFrame.new(camera.CFrame.Position, target.Position), 1 / Settings.Smoothing)
                end
            end
        end
    end

    for _, data in pairs(Cache) do hideAll(data) end
    local currentTargets = GetValidTargets(true, Settings.VisualsNPCs)
    if Settings.LocalPlayer and player.Character then table.insert(currentTargets, {Character = player.Character, Player = player}) end

    for _, targetData in pairs(currentTargets) do
        local char, p = targetData.Character, targetData.Player
        local id = p and p.UserId or char:GetAttribute("NPC_ID") or char.Name..char:GetDebugId()
        local hrp, hum = char:FindFirstChild("HumanoidRootPart"), char:FindFirstChildOfClass("Humanoid")
        if hrp and hum and hum.Health > 0 then
            local mag = (hrp.Position - camera.CFrame.Position).Magnitude
            if mag <= Settings.MaxDistance then
                local pos, onScreen = camera:WorldToViewportPoint(hrp.Position)
                if onScreen then
                    if not Cache[id] then
                        Cache[id] = {
                            Box = {createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{}), createDrawing("Line",{})},
                            Skeleton = {}, Tracer = createDrawing("Line", {Thickness = Settings.Thickness}), Name = createDrawing("Text", {Size = 13, Center = true, Outline = true}), Dist = createDrawing("Text", {Size = 12, Center = true, Outline = true}), HealthBarBG = createDrawing("Line", {Thickness = 5, Color = Color3.new(0,0,0)}), HealthBarMain = createDrawing("Line", {Thickness = 3})
                        }
                    end
                    local data, head = Cache[id], char:FindFirstChild("Head")
                    local headPos = head and camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
                    local legPos = camera:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))
                    if headPos and legPos then
                        local height = math.abs(headPos.Y - legPos.Y)
                        local width, x, y = height / 1.5, pos.X - (height / 1.5) / 2, pos.Y - height / 2
                        if Settings.Box then
                            local box = data.Box
                            if Settings.BoxMode == "3D" then
                                local cf, size = char:GetBoundingBox()
                                if size.Magnitude == 0 then size = Vector3.new(4, 5.5, 4) end
                                local pts, allOn = {}, true
                                local corners = {cf*CFrame.new(size.X/2,size.Y/2,size.Z/2), cf*CFrame.new(-size.X/2,size.Y/2,size.Z/2), cf*CFrame.new(-size.X/2,-size.Y/2,size.Z/2), cf*CFrame.new(size.X/2,-size.Y/2,size.Z/2), cf*CFrame.new(size.X/2,size.Y/2,-size.Z/2), cf*CFrame.new(-size.X/2,size.Y/2,-size.Z/2), cf*CFrame.new(-size.X/2,-size.Y/2,-size.Z/2), cf*CFrame.new(size.X/2,-size.Y/2,-size.Z/2)}
                                for i=1,8 do local pt, on = camera:WorldToViewportPoint(corners[i].Position); pts[i] = Vector2.new(pt.X,pt.Y); if not on or pt.Z < 0 then allOn = false end end
                                if allOn then
                                    local ed = {{1,2},{2,3},{3,4},{4,1},{5,6},{6,7},{7,8},{8,5},{1,5},{2,6},{3,7},{4,8}}
                                    for i=1,12 do box[i].From, box[i].To, box[i].Visible, box[i].Color, box[i].Thickness = pts[ed[i][1]], pts[ed[i][2]], true, Settings.ESPColor, Settings.BoxThickness end
                                else for i=1,12 do box[i].Visible = false end end
                            else
                                local tl, tr, bl, br = Vector2.new(x, y), Vector2.new(x+width, y), Vector2.new(x, y+height), Vector2.new(x+width, y+height)
                                if Settings.BoxStyle == "Full" then
                                    box[1].From, box[1].To, box[2].From, box[2].To, box[3].From, box[3].To, box[4].From, box[4].To = tl, tr, bl, br, tl, bl, tr, br
                                    for i=1,4 do box[i].Visible, box[i].Color, box[i].Thickness = true, Settings.ESPColor, Settings.BoxThickness end
                                    for i=5,12 do box[i].Visible = false end
                                else
                                    local l = width/4
                                    box[1].From, box[1].To, box[2].From, box[2].To, box[3].From, box[3].To, box[4].From, box[4].To = tl, tl+Vector2.new(l,0), tl, tl+Vector2.new(0,l), tr, tr-Vector2.new(l,0), tr, tr+Vector2.new(0,l)
                                    box[5].From, box[5].To, box[6].From, box[6].To, box[7].From, box[7].To, box[8].From, box[8].To = bl, bl+Vector2.new(l,0), bl, bl-Vector2.new(0,l), br, br-Vector2.new(l,0), br, br-Vector2.new(0,l)
                                    for i=1,8 do box[i].Visible, box[i].Color, box[i].Thickness = true, Settings.ESPColor, Settings.BoxThickness end
                                    for i=9,12 do box[i].Visible = false end
                                end
                            end
                        end
                        if Settings.Health then
                            local hp = hum.Health / hum.MaxHealth
                            data.HealthBarBG.Visible, data.HealthBarBG.From, data.HealthBarBG.To = true, Vector2.new(x-6, y+height), Vector2.new(x-6, y)
                            data.HealthBarMain.Visible, data.HealthBarMain.From, data.HealthBarMain.To, data.HealthBarMain.Color = true, Vector2.new(x-6, y+height), Vector2.new(x-6, y+height-(height*hp)), Color3.fromHSV(hp*0.3, 1, 1)
                        end
                        if Settings.Skeleton then
                            local rig = (hum.RigType == Enum.HumanoidRigType.R15) and "R15" or "R6"
                            local bConf = Bones[rig]
                            if #data.Skeleton ~= #bConf then for _, b in pairs(data.Skeleton) do b:Remove() end; data.Skeleton = {}; for i=1,#bConf do table.insert(data.Skeleton, createDrawing("Line", {Thickness = Settings.Thickness})) end end
                            for i, b in pairs(bConf) do
                                local p1, p2 = char:FindFirstChild(b[1]), char:FindFirstChild(b[2])
                                if p1 and p2 then
                                    local v1, o1 = camera:WorldToViewportPoint(p1.Position); local v2, o2 = camera:WorldToViewportPoint(p2.Position)
                                    if o1 and o2 then data.Skeleton[i].From, data.Skeleton[i].To, data.Skeleton[i].Color, data.Skeleton[i].Visible = Vector2.new(v1.X, v1.Y), Vector2.new(v2.X, v2.Y), Settings.ESPColor, true end
                                end
                            end
                        end
                        if Settings.Names then data.Name.Visible, data.Name.Text, data.Name.Position, data.Name.Color = true, (p and (p.DisplayName or p.Name)) or ("[NPC] "..char.Name), Vector2.new(pos.X, y-18), Settings.ESPColor end
                        if Settings.Distance then data.Dist.Visible, data.Dist.Text, data.Dist.Position, data.Dist.Color = true, math.floor(mag).."m", Vector2.new(pos.X, y+height+2), Settings.ESPColor end
                        if Settings.Tracers and char ~= player.Character then data.Tracer.Visible, data.Tracer.From, data.Tracer.To, data.Tracer.Color = true, Vector2.new(camera.ViewportSize.X/2, 0), Vector2.new(pos.X, y), Settings.ESPColor end
                    end
                end
            end
        end
    end
    if Settings.SpectateEnabled and Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
        local tHum = Settings.SelectedPlayer.Character:FindFirstChildOfClass("Humanoid")
        if tHum and camera.CameraSubject ~= tHum then camera.CameraSubject = tHum end
    elseif not Settings.SpectateEnabled and not Settings.IsFreecamming then
        local mHum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if mHum and camera.CameraSubject ~= mHum then camera.CameraSubject = mHum end
    end
end)

task.spawn(function()
    while not isUnloaded do
        if Settings.SpinbotEnabled and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = player.Character.HumanoidRootPart
            hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(Settings.SpinbotSpeed), 0)
        end
        task.wait()
    end
end)

task.spawn(function()
    while not isUnloaded do
        if Settings.SelectedPlayer and Settings.SelectedPlayer.Character and player.Character then
            local mHrp, tHrp = player.Character:FindFirstChild("HumanoidRootPart"), Settings.SelectedPlayer.Character:FindFirstChild("HumanoidRootPart")
            if mHrp and tHrp then
                if Settings.FollowPlayer then mHrp.CFrame = tHrp.CFrame * CFrame.new(0, 0, 1.5)
                elseif Settings.EatPlayer then mHrp.CFrame = tHrp.CFrame * CFrame.new(0, 0, 0.5 + math.sin(tick() * 15) * 1.3) end
            end
        end
        task.wait()
    end
end)

local playerRemovedConn = Players.PlayerRemoving:Connect(function(p) removeESP(p.UserId) end)

-- --- INTERFACE (TODOS EM UMA LINHA) ---

-- AIMBOT
AimTab:CreateToggle({Name = "Aimbot", Flag = "AimbotEnabled", Default = false, Callback = function(v) Settings.AimbotEnabled = v end})
AimTab:CreateToggle({Name = "Show FOV", Flag = "ShowFOV", Default = false, Callback = function(v) Settings.ShowFOV = v end})
AimTab:CreateToggle({Name = "Enable NPC", Flag = "AimbotNPCs", Default = false, Callback = function(v) Settings.AimbotNPCs = v end})
AimTab:CreateToggle({Name = "Team Check", Flag = "TeamCheck", Default = false, Callback = function(v) Settings.TeamCheck = v end})
AimTab:CreateToggle({Name = "Visible Check", Flag = "VisibleCheck", Default = false, Callback = function(v) Settings.VisibleCheck = v end})
AimTab:CreateKeybind({Name = "Keybind", Flag = "AimKey", Default = Enum.UserInputType.MouseButton2, Callback = function(v) Settings.AimKey = v end})
AimTab:CreateDropdown({Name = "Aim Method", Flag = "AimMethod", Options = {"Mouse", "Camera"}, Default = "Mouse", Callback = function(v) Settings.AimMethod = v end})
AimTab:CreateSlider({Name = "Smoothing", Flag = "Smoothing", Min = 1, Max = 10, Default = 2, Callback = function(v) Settings.Smoothing = v end})
AimTab:CreateSlider({Name = "Fov Radius", Flag = "FOVRadius", Min = 50, Max = 500, Default = 100, Callback = function(v) Settings.FOVRadius = v end})
AimTab:CreateSlider({Name = "Aimbot Distance", Flag = "AimDistance", Min = 50, Max = 2000, Default = 500, Callback = function(v) Settings.AimDistance = v end})
AimTab:CreateColorPicker({Name = "Fov Color", Flag = "FOVColor", Default = Settings.FOVColor, Callback = function(v) Settings.FOVColor = v end})

-- VISUALS
VisTab:CreateLabel("Elementos Visuais")
VisTab:CreateToggle({Name = "Usernames", Flag = "Names", Default = false, Callback = function(v) Settings.Names = v end})
VisTab:CreateToggle({Name = "Health Bar", Flag = "Health", Default = false, Callback = function(v) Settings.Health = v end})
VisTab:CreateToggle({Name = "Skeleton", Flag = "Skeleton", Default = false, Callback = function(v) Settings.Skeleton = v end})
VisTab:CreateToggle({Name = "Tracers", Flag = "Tracers", Default = false, Callback = function(v) Settings.Tracers = v end})
VisTab:CreateToggle({Name = "Distance", Flag = "Distance", Default = false, Callback = function(v) Settings.Distance = v end})
VisTab:CreateLabel("Configurações de Box")
VisTab:CreateToggle({Name = "Box ESP", Flag = "Box", Default = false, Callback = function(v) Settings.Box = v end})
VisTab:CreateDropdown({Name = "Box Mode", Flag = "BoxMode", Options = {"2D", "3D"}, Default = "2D", Callback = function(v) Settings.BoxMode = v end})
VisTab:CreateDropdown({Name = "Box Style (2D)", Flag = "BoxStyle", Options = {"Cornered", "Full"}, Default = "Full", Callback = function(v) Settings.BoxStyle = v end})
VisTab:CreateLabel("Filtros e Alcance")
VisTab:CreateToggle({Name = "Enable NPC", Flag = "VisualsNPCs", Default = false, Callback = function(v) Settings.VisualsNPCs = v end})
VisTab:CreateToggle({Name = "Team Check", Flag = "VisTeamCheck", Default = false, Callback = function(v) Settings.VisTeamCheck = v end})
VisTab:CreateToggle({Name = "Show Local Player", Flag = "LocalPlayer", Default = false, Callback = function(v) Settings.LocalPlayer = v end})
VisTab:CreateSlider({Name = "ESP Distance (M)", Flag = "MaxDistance", Min = 50, Max = 3500, Default = 500, Callback = function(v) Settings.MaxDistance = v end})
VisTab:CreateLabel("Personalização")
VisTab:CreateColorPicker({Name = "ESP Color", Flag = "ESPColor", Default = Settings.ESPColor, Callback = function(v) Settings.ESPColor = v end})

-- EXPLOITS / SELF
SelfTab:CreateLabel("Configurações de Fly")
SelfTab:CreateToggle({Name = "Fly", Flag = "FlyEnabled", Default = false, Callback = function(v) Settings.FlyEnabled = v; if not v then disableFly() end end})
SelfTab:CreateToggle({Name = "Invisible Fly", Flag = "FlyInvisible", Default = false, Callback = function(v) Settings.FlyInvisible = v; updateCharacterVisibility() end})
SelfTab:CreateKeybind({Name = "Fly Keybind", Flag = "FlyKey", Default = Enum.KeyCode.CapsLock, Callback = function(k) Settings.FlyKey = k end})
SelfTab:CreateDropdown({Name = "Fly Method", Flag = "FlyMethod", Options = {"Classic", "Bypass"}, Default = "Classic", Callback = function(v) Settings.FlyMethod = v end})
SelfTab:CreateSlider({Name = "FlySpeed", Flag = "FlySpeed", Min = 10, Max = 300, Default = 20, Callback = function(v) Settings.FlySpeed = v end})
SelfTab:CreateLabel("Configurações de Freecam")
SelfTab:CreateToggle({Name = "Freecam", Flag = "FreecamEnabled", Default = false, Callback = function(v) Settings.FreecamEnabled = v; if not v then toggleFreecam(false) end end})
SelfTab:CreateKeybind({Name = "Freecam Keybind", Flag = "FreecamKey", Default = Enum.KeyCode.P, Callback = function(k) Settings.FreecamKey = k end})
SelfTab:CreateSlider({Name = "Freecam Speed", Flag = "FreecamSpeed", Min = 5, Max = 200, Default = 20, Callback = function(v) Settings.FreecamSpeed = v end})
SelfTab:CreateLabel("SpinBot")
SelfTab:CreateToggle({Name = "Spinbot", Flag = "SpinbotEnabled", Default = false, Callback = function(v) Settings.SpinbotEnabled = v end})
SelfTab:CreateSlider({Name = "Spin Speed", Flag = "SpinbotSpeed", Min = 10, Max = 500, Default = 50, Callback = function(v) Settings.SpinbotSpeed = v end})

-- PLAYERS
local pOpts = {"None"}
for _, p in pairs(Players:GetPlayers()) do if p ~= player then table.insert(pOpts, p.DisplayName or p.Name) end end
local PlayerDropdown = PlayersTab:CreateDropdown({Name = "Select Player", Options = pOpts, Default = "None", Callback = function(v) if v == "None" then Settings.SelectedPlayer = nil else for _, p in pairs(Players:GetPlayers()) do if (p.DisplayName == v or p.Name == v) and p ~= player then Settings.SelectedPlayer = p; break end end end end})
PlayersTab:CreateButton({Name = "Refresh Player List", Callback = function() local n = {"None"}; for _, p in pairs(Players:GetPlayers()) do if p ~= player then table.insert(n, p.DisplayName or p.Name) end end; PlayerDropdown:Refresh(n) end})
PlayersTab:CreateButton({Name = "Teleport", Callback = function() if Settings.SelectedPlayer and Settings.SelectedPlayer.Character then player.Character:MoveTo(Settings.SelectedPlayer.Character.HumanoidRootPart.Position) end end})
PlayersTab:CreateToggle({Name = "Spectate", Flag = "SpectateEnabled", Default = false, Callback = function(v) Settings.SpectateEnabled = v end})
PlayersTab:CreateToggle({Name = "Teleport To Me", Flag = "PuxarLoop", Default = false, Callback = function(v) Settings.PuxarLoop = v end})
PlayersTab:CreateToggle({Name = "Follow Player", Flag = "FollowPlayer", Default = false, Callback = function(v) Settings.FollowPlayer = v end})
PlayersTab:CreateToggle({Name = "Fuck Player", Flag = "EatPlayer", Default = false, Callback = function(v) Settings.EatPlayer = v end})

-- MISC
MiscTab:CreateLabel("Configurações")
MiscTab:CreateButton({Name = "Save Config", Callback = SaveConfig})
MiscTab:CreateButton({Name = "Load Config", Callback = LoadConfig})
MiscTab:CreateButton({Name = "Unload Hub", Callback = function() isUnloaded = true; if renderConn then renderConn:Disconnect() end; if inputBeganConn then inputBeganConn:Disconnect() end; if inputEndedConn then inputEndedConn:Disconnect() end; if playerRemovedConn then playerRemovedConn:Disconnect() end; disableFly(); toggleFreecam(false); for _, d in pairs(Cache) do hideAll(d) end; Cache = {}; if FOVCircle then FOVCircle:Remove() end; local hG = game:GetService("CoreGui"):FindFirstChild("XanaxHub_V2") or player:FindFirstChild("PlayerGui"):FindFirstChild("XanaxHub_V2"); if hG then hG:Destroy() end; camera.CameraType = Enum.CameraType.Custom end})

-- Loop de Puxar
task.spawn(function()
    while not isUnloaded do
        if Settings.PuxarLoop and Settings.SelectedPlayer and Settings.SelectedPlayer.Character then
            local tH, mH = Settings.SelectedPlayer.Character:FindFirstChild("HumanoidRootPart"), player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            if tH and mH then tH.CFrame = mH.CFrame * CFrame.new(0, 0, -3) end
        end
        task.wait(0.1)
    end
end)
