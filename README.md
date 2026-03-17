-- Script Profissional Otimizado para Ricardo 💖
-- Foco: Correção Definitiva da UI, Aimbot via Mouse e Estabilidade Total

local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- Carregando a Lib (Certifique-se que o executor suporta game:HttpGet)
local Hub = loadstring(game:HttpGet("https://raw.githubusercontent.com/Stand-Software/hub/refs/heads/main/README.md"))()

local Window = Hub:CreateWindow({
    Title = "XANAX ULTIMATE ESP 💖"
})

local AimTab = Window:CreateTab("Aimbot")
local Tab = Window:CreateTab("Visuals")

-- --- CONFIGURAÇÕES ---
local Settings = {
    -- ESP
    Box = false,
    Skeleton = false,
    Tracers = false,
    Distance = false,
    Names = false,
    Health = false,
    LocalPlayer = false,
    Color = Color3.fromRGB(255, 0, 85),
    Thickness = 1,
    BoxThickness = 2,
    MaxDistance = 500,
    -- Aimbot
    AimbotEnabled = false,
    ShowFOV = false,
    FOVRadius = 100,
    FOVColor = Color3.fromRGB(255, 255, 255),
    TargetPart = "Head", -- Alvo fixo na cabeça conforme solicitado
    AimKey = Enum.UserInputType.MouseButton2,
    AimDistance = 500,
    Smoothing = 2 
}

local Cache = {}
local isAiming = false

-- --- OBJETOS DE DESENHO FIXOS ---
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness = 1
FOVCircle.NumSides = 100
FOVCircle.Filled = false
FOVCircle.Transparency = 1
FOVCircle.Visible = false

-- --- FUNÇÕES AUXILIARES ---
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

-- Função para pegar o alvo mais próximo do cursor dentro do FOV
local function getClosestPlayer()
    local target = nil
    local shortestDistance = math.huge
    local mousePos = UserInputService:GetMouseLocation()

    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character then
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
                        if distFromMouse <= Settings.FOVRadius and distFromMouse < shortestDistance then
                            target = part
                            shortestDistance = distFromMouse
                        end
                    end
                end
            end
        end
    end
    return target
end

-- Detecção de Tecla/Mouse para Aimbot
UserInputService.InputBegan:Connect(function(input)
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then
        isAiming = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Settings.AimKey or input.KeyCode == Settings.AimKey then
        isAiming = false
    end
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
RunService.RenderStepped:Connect(function()
    -- Atualiza FOV Circle
    FOVCircle.Visible = Settings.ShowFOV
    FOVCircle.Radius = Settings.FOVRadius
    FOVCircle.Color = Settings.FOVColor
    FOVCircle.Position = UserInputService:GetMouseLocation()

    -- Lógica do Aimbot (Movimentação do Mouse)
    if Settings.AimbotEnabled and isAiming then
        local target = getClosestPlayer()
        if target then
            local screenPos, onScreen = camera:WorldToViewportPoint(target.Position)
            if onScreen then
                local mousePos = UserInputService:GetMouseLocation()
                local moveX = (screenPos.X - mousePos.X) / Settings.Smoothing
                local moveY = (screenPos.Y - mousePos.Y) / Settings.Smoothing
                
                -- Requer mousemoverel no executor
                if mousemoverel then
                    mousemoverel(moveX, moveY)
                end
            end
        end
    end

    -- Lógica do ESP
    for _, data in pairs(Cache) do hideAll(data) end

    for _, p in pairs(Players:GetPlayers()) do
        if (p ~= player or Settings.LocalPlayer) and p.Parent then 
            local char = p.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChild("Humanoid")

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
                                Name = createDrawing("Text", {Size = 13, Center = true, Outline = true, OutlineColor = Color3.new(0,0,0)}),
                                Dist = createDrawing("Text", {Size = 12, Center = true, Outline = true, OutlineColor = Color3.new(0,0,0)}),
                                HealthBarBG = createDrawing("Line", {Thickness = 5, Color = Color3.new(0,0,0)}),
                                HealthBarMain = createDrawing("Line", {Thickness = 3, Color = Color3.new(0,1,0)})
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
                            local lineLen = width / 4

                            if Settings.Box then
                                local tl = Vector2.new(x, y)
                                local tr = Vector2.new(x + width, y)
                                local bl = Vector2.new(x, y + height)
                                local br = Vector2.new(x + width, y + height)
                                local box = data.Box
                                box[1].From = tl; box[1].To = tl + Vector2.new(lineLen, 0)
                                box[2].From = tl; box[2].To = tl + Vector2.new(0, lineLen)
                                box[3].From = tr; box[3].To = tr - Vector2.new(lineLen, 0)
                                box[4].From = tr; box[4].To = tr + Vector2.new(0, lineLen)
                                box[5].From = bl; box[5].To = bl + Vector2.new(lineLen, 0)
                                box[6].From = bl; box[6].To = bl - Vector2.new(0, lineLen)
                                box[7].From = br; box[7].To = br - Vector2.new(lineLen, 0)
                                box[8].From = br; box[8].To = br - Vector2.new(0, lineLen)
                                for _, l in pairs(box) do 
                                    l.Visible = true; l.Color = Settings.Color; l.Thickness = Settings.BoxThickness
                                end
                            end

                            if Settings.Health then
                                local healthPercent = hum.Health / hum.MaxHealth
                                local barPos = x - 7
                                data.HealthBarBG.From = Vector2.new(barPos, y + height); data.HealthBarBG.To = Vector2.new(barPos, y); data.HealthBarBG.Visible = true
                                data.HealthBarMain.From = Vector2.new(barPos, y + height); data.HealthBarMain.To = Vector2.new(barPos, (y + height) - (height * healthPercent)); data.HealthBarMain.Color = Color3.fromHSV(healthPercent * 0.3, 1, 1); data.HealthBarMain.Visible = true
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
                                    local p1_part, p2_part = char:FindFirstChild(bone[1]), char:FindFirstChild(bone[2])
                                    if p1_part and p2_part then
                                        local v1, o1 = camera:WorldToViewportPoint(p1_part.Position)
                                        local v2, o2 = camera:WorldToViewportPoint(p2_part.Position)
                                        if o1 and o2 then
                                            local line = data.Skeleton[i]
                                            line.From = Vector2.new(v1.X, v1.Y); line.To = Vector2.new(v2.X, v2.Y)
                                            line.Color = Settings.Color; line.Thickness = Settings.Thickness; line.Visible = true
                                        end
                                    end
                                end
                            end

                            if Settings.Tracers and p ~= player then
                                data.Tracer.Visible = true; data.Tracer.From = Vector2.new(camera.ViewportSize.X / 2, 0); data.Tracer.To = Vector2.new(pos.X, y); data.Tracer.Color = Settings.Color; data.Tracer.Thickness = Settings.Thickness
                            end

                            if Settings.Names then
                                data.Name.Visible = true; data.Name.Text = p.DisplayName or p.Name; data.Name.Position = Vector2.new(pos.X, y - 18); data.Name.Color = Settings.Color
                            end

                            if Settings.Distance then
                                data.Dist.Visible = true; data.Dist.Text = math.floor(mag) .. "m"; data.Dist.Position = Vector2.new(pos.X, y + height + 2); data.Dist.Color = Settings.Color
                            end
                        end
                    end
                end
            end
        end
    end
end)

Players.PlayerRemoving:Connect(removeESP)

-- --- INTERFACE
AimTab:CreateToggle({
    Name = "Ativar Aimbot",
    Default = false,
    Callback = function(v) Settings.AimbotEnabled = v end
})

AimTab:CreateToggle({
    Name = "Ativar FOV (Círculo)",
    Default = false,
    Callback = function(v) Settings.ShowFOV = v end
})

AimTab:CreateKeybind({
    Name = "Tecla para Segurar",
    Default = Enum.UserInputType.MouseButton2,
    Callback = function(v) Settings.AimKey = v end
})

AimTab:CreateColorPicker({
    Name = "Cor do Círculo FOV",
    Default = Settings.FOVColor,
    Callback = function(v) Settings.FOVColor = v end
})

AimTab:CreateSlider({
    Name = "Tamanho do FOV (Raio)",
    Min = 10, 
    Max = 600, 
    Default = 100,
    Callback = function(v) Settings.FOVRadius = v end
})

AimTab:CreateSlider({
    Name = "Alcance do Aim (m)",
    Min = 50, 
    Max = 5000, 
    Default = 500,
    Callback = function(v) Settings.AimDistance = v end
})

AimTab:CreateSlider({
    Name = "Suavização (Smoothing)",
    Min = 1, 
    Max = 20, 
    Default = 2,
    Callback = function(v) Settings.Smoothing = v end
})

Tab:CreateLabel("Componentes Visuais")
Tab:CreateToggle({Name = "Box Corner ESP", Default = false, Callback = function(v) Settings.Box = v end})
Tab:CreateToggle({Name = "Health Bar (Vida)", Default = false, Callback = function(v) Settings.Health = v end})
Tab:CreateToggle({Name = "Skeleton ESP (R6/R15)", Default = false, Callback = function(v) Settings.Skeleton = v end})
Tab:CreateToggle({Name = "Tracers (Do Topo)", Default = false, Callback = function(v) Settings.Tracers = v end})
Tab:CreateToggle({Name = "Show Distance", Default = false, Callback = function(v) Settings.Distance = v end})
Tab:CreateToggle({Name = "Show Usernames", Default = false, Callback = function(v) Settings.Names = v end})
Tab:CreateLabel("Filtros")
Tab:CreateToggle({Name = "Show Local Player", Default = false, Callback = function(v) Settings.LocalPlayer = v end})
Tab:CreateLabel("Ajustes de Renderização")
Tab:CreateColorPicker({Name = "Cor do ESP", Default = Settings.Color, Callback = function(v) Settings.Color = v end})
Tab:CreateSlider({Name = "Alcance Máximo (m)", Min = 50, Max = 5000, Default = 500, Callback = function(v) Settings.MaxDistance = v end})
