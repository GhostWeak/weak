-- CamLock LocalScript
-- Coloque dentro de StarterPlayerScripts ou StarterCharacterScripts

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- =============================================
-- CONFIGURAÇÕES
-- =============================================
local CONFIG = {
    MaxDistance    = 200,  -- Distância máxima para detectar alvos (studs)
    WallCheckParts = true, -- true = só trava se tiver visão livre (sem paredes)
}

-- =============================================
-- ESTADO
-- =============================================
local camLockEnabled  = false
local currentTarget   = nil
local connection      = nil
local deathConnection = nil

-- =============================================
-- GUI — Botão Arrastável
-- =============================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name           = "CamLockGUI"
ScreenGui.ResetOnSpawn   = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent         = LocalPlayer.PlayerGui

local Frame = Instance.new("Frame")
Frame.Name             = "DragFrame"
Frame.Size             = UDim2.new(0, 140, 0, 44)
Frame.Position         = UDim2.new(0.5, -70, 0, 60)
Frame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
Frame.BorderSizePixel  = 0
Frame.Active           = true
Frame.Parent           = ScreenGui

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 8)
Corner.Parent       = Frame

local Stroke = Instance.new("UIStroke")
Stroke.Color     = Color3.fromRGB(80, 120, 255)
Stroke.Thickness = 1.5
Stroke.Parent    = Frame

local StatusBar = Instance.new("Frame")
StatusBar.Name             = "StatusBar"
StatusBar.Size             = UDim2.new(0, 4, 1, -8)
StatusBar.Position         = UDim2.new(0, 5, 0, 4)
StatusBar.BackgroundColor3 = Color3.fromRGB(80, 120, 255)
StatusBar.BorderSizePixel  = 0
StatusBar.Parent           = Frame

local StatusCorner = Instance.new("UICorner")
StatusCorner.CornerRadius = UDim.new(0, 4)
StatusCorner.Parent       = StatusBar

local Button = Instance.new("TextButton")
Button.Name                   = "CamLockButton"
Button.Size                   = UDim2.new(1, -18, 1, 0)
Button.Position               = UDim2.new(0, 18, 0, 0)
Button.BackgroundTransparency = 1
Button.Text                   = "⊕  CAM LOCK"
Button.TextColor3             = Color3.fromRGB(200, 210, 255)
Button.TextSize               = 13
Button.Font                   = Enum.Font.GothamBold
Button.TextXAlignment         = Enum.TextXAlignment.Left
Button.Parent                 = Frame

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Name                   = "StatusLabel"
StatusLabel.Size                   = UDim2.new(1, 0, 0, 10)
StatusLabel.Position               = UDim2.new(0, 18, 1, 2)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text                   = "OFF"
StatusLabel.TextColor3             = Color3.fromRGB(100, 110, 160)
StatusLabel.TextSize               = 9
StatusLabel.Font                   = Enum.Font.GothamBold
StatusLabel.TextXAlignment         = Enum.TextXAlignment.Left
StatusLabel.Parent                 = Frame

-- =============================================
-- TOGGLE (declarado antes do drag para o tap mobile poder chamar)
-- =============================================
local function toggle() end  -- definido abaixo, placeholder para referência antecipada

-- =============================================
-- ARRASTAR (Drag) — PC e Mobile com tap
-- =============================================
local dragging      = false
local dragMoved     = false
local dragStart, startPos
local DRAG_THRESHOLD = 10  -- pixels; abaixo disso é tap, acima é arrasto

local function updateDrag(input)
    local delta = input.Position - dragStart
    if not dragMoved then
        if math.abs(delta.X) > DRAG_THRESHOLD or math.abs(delta.Y) > DRAG_THRESHOLD then
            dragMoved = true
        else
            return
        end
    end
    Frame.Position = UDim2.new(
        startPos.X.Scale, startPos.X.Offset + delta.X,
        startPos.Y.Scale, startPos.Y.Offset + delta.Y
    )
end

Frame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging  = true
        dragMoved = false
        dragStart = input.Position
        startPos  = Frame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                -- Toque rápido sem mover = tap → ativa/desativa
                if not dragMoved
                and input.UserInputType == Enum.UserInputType.Touch then
                    toggle()
                end
                dragging  = false
                dragMoved = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement
    or input.UserInputType == Enum.UserInputType.Touch then
        updateDrag(input)
    end
end)

-- =============================================
-- FUNÇÕES AUXILIARES
-- =============================================

-- Verifica linha de visão usando dois raios (olhos do personagem + câmera)
-- Mira no TORSO (HumanoidRootPart) do alvo
local function hasLineOfSight(targetPlayer)
    local myChar = LocalPlayer.Character
    if not myChar then return false end
    local myRoot = myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return false end

    local targetChar = targetPlayer.Character
    if not targetChar then return false end

    local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
    if not targetRoot then return false end

    local aimPos = targetRoot.Position

    local ignoreList = { myChar, targetChar }
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = ignoreList
    rayParams.FilterType = Enum.RaycastFilterType.Exclude

    local origins = {
        myRoot.Position + Vector3.new(0, 1.5, 0), -- altura dos olhos
        Camera.CFrame.Position,                    -- câmera
    }

    for _, origin in ipairs(origins) do
        local dir    = aimPos - origin
        local result = workspace:Raycast(origin, dir, rayParams)
        if result == nil then
            return true  -- visão livre por pelo menos um raio
        end
    end

    return false  -- todos os raios bloqueados = atrás de parede
end

-- Verifica se o player está vivo
local function isAlive(player)
    if not player then return false end
    local char = player.Character
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return false end
    return hum.Health > 0
end

-- Busca o player mais próximo em 360° com visão livre
local function getClosestTarget()
    local char = LocalPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end

    local closest     = nil
    local closestDist = math.huge

    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end

        local targetChar = player.Character
        if not targetChar then continue end
        local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
        local hum        = targetChar:FindFirstChildOfClass("Humanoid")
        if not targetRoot or not hum then continue end
        if hum.Health <= 0 then continue end

        local dist = (root.Position - targetRoot.Position).Magnitude
        if dist > CONFIG.MaxDistance then continue end

        if CONFIG.WallCheckParts and not hasLineOfSight(player) then
            continue
        end

        if dist < closestDist then
            closestDist = dist
            closest     = player
        end
    end

    return closest
end

-- =============================================
-- ATUALIZAÇÃO VISUAL
-- =============================================
local function updateUI()
    if camLockEnabled then
        Button.Text                = "⊗  CAM LOCK"
        Button.TextColor3          = Color3.fromRGB(120, 220, 120)
        Stroke.Color               = Color3.fromRGB(80, 220, 100)
        StatusBar.BackgroundColor3 = Color3.fromRGB(80, 220, 100)
        StatusLabel.TextColor3     = Color3.fromRGB(80, 180, 80)
        StatusLabel.Text           = currentTarget
            and ("ON → " .. currentTarget.Name)
            or  "ON — Procurando..."
    else
        Button.Text                = "⊕  CAM LOCK"
        Button.TextColor3          = Color3.fromRGB(200, 210, 255)
        Stroke.Color               = Color3.fromRGB(80, 120, 255)
        StatusBar.BackgroundColor3 = Color3.fromRGB(80, 120, 255)
        StatusLabel.Text           = "OFF"
        StatusLabel.TextColor3     = Color3.fromRGB(100, 110, 160)
    end
end

-- =============================================
-- MONITORAMENTO DE MORTE
-- =============================================
local function clearDeathWatch()
    if deathConnection then
        deathConnection:Disconnect()
        deathConnection = nil
    end
end

local function watchTargetDeath(player)
    clearDeathWatch()
    if not player then return end
    local char = player.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    deathConnection = hum.Died:Connect(function()
        currentTarget = nil
        clearDeathWatch()
        updateUI()
    end)
end

-- =============================================
-- LOOP PRINCIPAL
-- =============================================
local function startCamLock()
    connection = RunService.RenderStepped:Connect(function()
        if not camLockEnabled then return end

        -- Reavalia se o alvo morreu ou foi parede
        if not isAlive(currentTarget)
        or (CONFIG.WallCheckParts and currentTarget and not hasLineOfSight(currentTarget)) then
            currentTarget = getClosestTarget()
            watchTargetDeath(currentTarget)
            updateUI()
        end

        if not currentTarget then return end

        local targetChar = currentTarget.Character
        if not targetChar then return end

        -- Mira no TORSO do alvo (HumanoidRootPart)
        local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
        if not targetRoot then return end

        -- INSTANTÂNEO: câmera aponta direto para o torso
        Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetRoot.Position)
    end)
end

local function stopCamLock()
    if connection then
        connection:Disconnect()
        connection = nil
    end
    clearDeathWatch()
    currentTarget = nil
    updateUI()
end

-- =============================================
-- TOGGLE (implementação real)
-- =============================================
toggle = function()
    camLockEnabled = not camLockEnabled
    if camLockEnabled then
        LocalPlayer.CameraMode = Enum.CameraMode.Classic
        Camera.CameraType      = Enum.CameraType.Custom
        startCamLock()
    else
        stopCamLock()
    end
    updateUI()
end

-- PC: clique normal no botão
Button.MouseButton1Click:Connect(toggle)

-- PC/Mobile via teclado
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.V then toggle() end
end)

-- =============================================
-- INICIALIZAÇÃO
-- =============================================
updateUI()
print("[CamLock] Carregado! Toque no botão ou pressione V para ativar.")
