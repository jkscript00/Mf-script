--==================================================
--                 JK RAYFIELD HUB
--              SISTEMA PARA SEU JOGO
--==================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local Rayfield = loadstring(game:HttpGet(
    "https://sirius.menu/rayfield"
))()

local Window = Rayfield:CreateWindow({
    Name = "JK HUB",
    LoadingTitle = "JK HUB",
    LoadingSubtitle = "Sistema de testes",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "JKHub",
        FileName = "Config"
    }
})

--==================================================
-- CONFIG
--==================================================

local AimEnabled = false
local ESPEnabled = false
local FOV = 150

local ESPObjects = {}

-- 200 METROS ≈ 714 STUDS
local MAX_TP_DISTANCE = 714

--==================================================
-- AIM
--==================================================

local AimTab = Window:CreateTab("Aim", 4483362458)

AimTab:CreateToggle({
    Name = "Aimbot",
    CurrentValue = false,
    Flag = "Aimbot",

    Callback = function(Value)
        AimEnabled = Value
    end
})

AimTab:CreateSlider({
    Name = "Tamanho do FOV",
    Range = {50, 500},
    Increment = 10,
    Suffix = " px",
    CurrentValue = 150,
    Flag = "FOV",

    Callback = function(Value)
        FOV = Value
    end
})

--==================================================
-- ESP
--==================================================

local ESPTab = Window:CreateTab("ESP", 4483362458)

ESPTab:CreateToggle({
    Name = "ESP",
    CurrentValue = false,
    Flag = "ESP",

    Callback = function(Value)
        ESPEnabled = Value

        if not Value then
            for _, objects in pairs(ESPObjects) do
                objects.Box.Visible = false
                objects.Health.Visible = false
            end
        end
    end
})

--==================================================
-- FOV CIRCLE
--==================================================

local FOVCircle = Drawing.new("Circle")

FOVCircle.Visible = false
FOVCircle.Filled = false
FOVCircle.Thickness = 2
FOVCircle.NumSides = 64

--==================================================
-- ESP
--==================================================

local function CreateESP(player)

    if ESPObjects[player] then
        return
    end

    local box = Drawing.new("Square")
    box.Filled = false
    box.Thickness = 2
    box.Visible = false

    local health = Drawing.new("Square")
    health.Filled = true
    health.Visible = false

    ESPObjects[player] = {
        Box = box,
        Health = health
    }
end

local function RemoveESP(player)

    local objects = ESPObjects[player]

    if not objects then
        return
    end

    for _, object in pairs(objects) do
        object:Remove()
    end

    ESPObjects[player] = nil
end

--==================================================
-- WALL CHECK
--==================================================

local function IsVisible(part)

    if not part then
        return false
    end

    local origin = Camera.CFrame.Position
    local direction = part.Position - origin

    local params = RaycastParams.new()

    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {
        LocalPlayer.Character
    }

    local result = workspace:Raycast(
        origin,
        direction,
        params
    )

    if not result then
        return true
    end

    return result.Instance:IsDescendantOf(
        part.Parent
    )
end

--==================================================
-- TARGET AIM
--==================================================

local function GetTarget()

    local closest = nil
    local shortest = FOV

    local center = Vector2.new(
        Camera.ViewportSize.X / 2,
        Camera.ViewportSize.Y / 2
    )

    for _, player in ipairs(Players:GetPlayers()) do

        if player ~= LocalPlayer then

            local character = player.Character
            local humanoid = character
                and character:FindFirstChildOfClass("Humanoid")
            local head = character
                and character:FindFirstChild("Head")

            if humanoid
                and humanoid.Health > 0
                and head
                and IsVisible(head) then

                local screenPos, visible =
                    Camera:WorldToViewportPoint(
                        head.Position
                    )

                if visible then

                    local distance = (
                        Vector2.new(
                            screenPos.X,
                            screenPos.Y
                        ) - center
                    ).Magnitude

                    if distance < shortest then
                        shortest = distance
                        closest = head
                    end
                end
            end
        end
    end

    return closest
end

--==================================================
-- ESP UPDATE
--==================================================

local function UpdateESP(player)

    local objects = ESPObjects[player]

    if not objects then
        CreateESP(player)
        objects = ESPObjects[player]
    end

    local character = player.Character
    local humanoid = character
        and character:FindFirstChildOfClass("Humanoid")
    local root = character
        and character:FindFirstChild("HumanoidRootPart")

    if not ESPEnabled
        or not character
        or not humanoid
        or not root
        or humanoid.Health <= 0 then

        objects.Box.Visible = false
        objects.Health.Visible = false
        return
    end

    local rootPos, onScreen =
        Camera:WorldToViewportPoint(root.Position)

    if not onScreen then
        objects.Box.Visible = false
        objects.Health.Visible = false
        return
    end

    local top =
        Camera:WorldToViewportPoint(
            root.Position + Vector3.new(0, 3, 0)
        )

    local bottom =
        Camera:WorldToViewportPoint(
            root.Position - Vector3.new(0, 3, 0)
        )

    local height = math.abs(top.Y - bottom.Y)
    local width = height * 0.55

    local position = Vector2.new(
        rootPos.X - width / 2,
        rootPos.Y - height / 2
    )

    objects.Box.Position = position
    objects.Box.Size = Vector2.new(width, height)
    objects.Box.Visible = true

    local healthPercent = math.clamp(
        humanoid.Health / humanoid.MaxHealth,
        0,
        1
    )

    objects.Health.Position = Vector2.new(
        position.X - 7,
        position.Y + height * (1 - healthPercent)
    )

    objects.Health.Size = Vector2.new(
        4,
        height * healthPercent
    )

    objects.Health.Visible = true
end

--==================================================
-- TELEPORT
--==================================================

local TeleportTab = Window:CreateTab(
    "Teleport",
    4483362458
)

local function GetRandomTargetWithinRange()

    local targets = {}

    local myCharacter = LocalPlayer.Character
    local myRoot = myCharacter
        and myCharacter:FindFirstChild("HumanoidRootPart")

    if not myRoot then
        return nil
    end

    for _, player in ipairs(Players:GetPlayers()) do

        if player ~= LocalPlayer then

            local character = player.Character
            local humanoid = character
                and character:FindFirstChildOfClass("Humanoid")
            local root = character
                and character:FindFirstChild("HumanoidRootPart")

            if humanoid
                and humanoid.Health > 0
                and root then

                local distance =
                    (root.Position - myRoot.Position).Magnitude

                -- Máximo de aproximadamente 200 metros
                if distance <= MAX_TP_DISTANCE then
                    table.insert(targets, player)
                end
            end
        end
    end

    if #targets == 0 then
        return nil
    end

    return targets[
        math.random(1, #targets)
    ]
end

TeleportTab:CreateButton({
    Name = "TP Atrás de Inimigo Aleatório",

    Callback = function()

        local target = GetRandomTargetWithinRange()

        if not target then

            Rayfield:Notify({
                Title = "JK HUB",
                Content = "Nenhum inimigo vivo encontrado em até 200 metros.",
                Duration = 4
            })

            return
        end

        local myCharacter = LocalPlayer.Character
        local targetCharacter = target.Character

        local myRoot = myCharacter
            and myCharacter:FindFirstChild("HumanoidRootPart")

        local targetRoot = targetCharacter
            and targetCharacter:FindFirstChild("HumanoidRootPart")

        if not myRoot or not targetRoot then
            return
        end

        -- 4 studs atrás do alvo
        local behindPosition =
            targetRoot.Position -
            targetRoot.CFrame.LookVector * 4

        myRoot.CFrame = CFrame.new(
            behindPosition,
            targetRoot.Position
        )

        Rayfield:Notify({
            Title = "JK HUB",
            Content =
                "TP realizado atrás de " ..
                target.Name,
            Duration = 3
        })
    end
})

--==================================================
-- CONFIG
--==================================================

local ConfigTab = Window:CreateTab(
    "Config",
    4483362458
)

ConfigTab:CreateButton({
    Name = "Desativar Tudo",

    Callback = function()

        AimEnabled = false
        ESPEnabled = false

        FOVCircle.Visible = false

        for _, objects in pairs(ESPObjects) do
            objects.Box.Visible = false
            objects.Health.Visible = false
        end

        Rayfield:Notify({
            Title = "JK HUB",
            Content = "Tudo desativado.",
            Duration = 3
        })
    end
})

--==================================================
-- LOOP
--==================================================

RunService.RenderStepped:Connect(function()

    local viewport = Camera.ViewportSize

    FOVCircle.Position = Vector2.new(
        viewport.X / 2,
        viewport.Y / 2
    )

    FOVCircle.Radius = FOV
    FOVCircle.Visible = AimEnabled

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            UpdateESP(player)
        end
    end

    if AimEnabled then

        local target = GetTarget()

        if target and IsVisible(target) then

            Camera.CFrame = CFrame.lookAt(
                Camera.CFrame.Position,
                target.Position
            )
        end
    end
end)

--==================================================
-- PLAYERS
--==================================================

Players.PlayerAdded:Connect(function(player)
    CreateESP(player)
end)

Players.PlayerRemoving:Connect(function(player)
    RemoveESP(player)
end)

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end

--==================================================
-- START
--==================================================

Rayfield:Notify({
    Title = "JK HUB",
    Content = "Hub carregado.",
    Duration = 4
})
