--!strict
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local lp = Players.LocalPlayer
local playerGui = lp:WaitForChild("PlayerGui")

-- Variables de configuración / Sliders
local SCALE_MULTIPLIER = 1
local MECH_HEIGHT_OFFSET = 2
local BACKWARD_OFFSET_Z = 0
local BACKWARD_OFFSET_X = 0
local PROP_ROTATION_X = 0
local PROP_ROTATION_Y = 0
local PROP_ROTATION_Z = 0

-- Contenedor principal (Prime)
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PrimeMenuGui"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local targetParent = (pcall(function() return CoreGui.Name end) and CoreGui) or playerGui
screenGui.Parent = targetParent

local structureData = {
    {type = "Light", cframe = CFrame.new(203.401901, 5.0834837, -32.8884125, -0.99950397, -0.0314921588, 3.7830323e-06, -0.000240206718, -1, 0.0314921588, -0.99950397, 0.000240206718, 0)},
    {type = "Light", cframe = CFrame.new(205.087448, 2.28348351, -34.2391853, -0.99950397, -0.0314921588, 3.7830323e-06, -0.000240206718, -1, 0.0314921588, -0.99950397, 0.000240206718, 0)},
    {type = "Light", cframe = CFrame.new(198.332138, 2.28348446, -35.3929024, -0.99950397, -0.0314921588, 3.7830323e-06, -0.000240206718, -1, 0.0314921588, -0.99950397, 0.000240206718, 0)},
    {type = "Light", cframe = CFrame.new(199.285278, 6.4334836, -33.8421593, -0.99950397, -0.0314921588, 3.7830323e-06, -0.000240206718, -1, 0.0314921588, -0.99950397, 0.000240206718, 0)},
    {type = "Switch", cframe = CFrame.new(201.746902, 0.17499724, -35.2829399, -0.000780344009, -0.999996305, -0.00259310007, -2.44379044e-06, 0.00259310007, -0.999996662, 0.999999702, -0.000780373812, -4.529953e-06)}
}

local activeProps = {}
local activeRemotes = {} -- Caché de remotes para evitar búsquedas lentas
local activeColorRemotes = {}
local isFollowing = false
local rawBaseOffsets = {}
local propIndicesMap = {}

local centerOffset = CFrame.new()
do
    local sumPos = Vector3.new()
    local count = 0
    for _, data in ipairs(structureData) do
        sumPos += data.cframe.Position
        count += 1
    end
    if count > 0 then
        centerOffset = CFrame.new(sumPos / count)
    end
end

local nextNetworkUpdate = 0
local NETWORK_INTERVAL = 0.06 -- Ligeramente más espaciado para optimizar red

local function updatePropTransform(prop: Instance, targetCFrame: CFrame, forceRemote: boolean, idx: number)
    if prop:IsA("Model") then
        prop:PivotTo(targetCFrame)
    elseif prop:IsA("BasePart") then
        prop.CFrame = targetCFrame
    end
    
    if forceRemote then
        local remote = activeRemotes[prop]
        if remote then
            task.spawn(function()
                pcall(function() remote:InvokeServer(targetCFrame) end)
            end)
        end
    end
end

local function changePropColor(prop: Instance, targetColor: Color3)
    local remote = activeColorRemotes[prop]
    if remote then
        task.spawn(function()
            pcall(function() remote:InvokeServer(targetColor) end)
        end)
    end
end

local function executeStructureMove()
    local com = workspace:FindFirstChild("WorkspaceCom")
    if not com then return end

    local availableProps = {}
    local function collectProps(parent)
        for _, child in ipairs(parent:GetChildren()) do
            if child.Name == "Prop" .. lp.Name then
                table.insert(availableProps, child)
            end
            if child:IsA("Folder") or child:IsA("Model") then
                collectProps(child)
            end
        end
    end
    collectProps(com)

    if #availableProps == 0 then return end
    activeProps = {}
    rawBaseOffsets = {}
    propIndicesMap = {}
    activeRemotes = {}
    activeColorRemotes = {}

    for index, data in ipairs(structureData) do
        local prop = availableProps[index]
        if prop then
            table.insert(activeProps, prop)
            rawBaseOffsets[prop] = centerOffset:ToObjectSpace(data.cframe)
            propIndicesMap[prop] = index
            
            -- Guardar remotes en caché para optimizar rendimiento
            local remote = prop:FindFirstChild("SetCurrentCFrame")
            if remote and remote:IsA("RemoteFunction") then
                activeRemotes[prop] = remote
            end
            local colorRemote = prop:FindFirstChild("ChangePropColor")
            if colorRemote and colorRemote:IsA("RemoteFunction") then
                activeColorRemotes[prop] = colorRemote
            end

            updatePropTransform(prop, data.cframe, true, index)
        end
    end
end

local colorToggleTimer = 0
local currentColorState = false

local function handleColorCycling(dt)
    colorToggleTimer += dt
    if colorToggleTimer >= 1.0 then
        colorToggleTimer = 0
        currentColorState = not currentColorState
        
        for prop, idx in pairs(propIndicesMap) do
            if idx >= 1 and idx <= 4 then
                local isWhiteNow = (idx == 1 or idx == 3) and currentColorState or not currentColorState
                changePropColor(prop, isWhiteNow and Color3.new(1, 1, 1) or Color3.new(0, 0, 0))
            end
        end
    end
end

local currentPosition = Vector3.new()
local currentRotation = CFrame.new()

RunService.RenderStepped:Connect(function(dt)
    if not isFollowing then return end
    local character = lp.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    
    local rootPart = character.HumanoidRootPart
    handleColorCycling(dt)

    local t = os.clock()
    local shakeX = math.sin(t * 8) * 0.12
    local shakeY = math.cos(t * 6) * 0.08
    local shakeZ = math.sin(t * 7) * 0.12
    local lifeTremorCF = CFrame.new(shakeX, shakeY, shakeZ)

    local targetPos = rootPart.Position + Vector3.new(BACKWARD_OFFSET_X, MECH_HEIGHT_OFFSET, BACKWARD_OFFSET_Z)
    local _, yaw, _ = rootPart.CFrame:ToEulerAnglesYXZ()
    local targetRot = CFrame.Angles(0, yaw, 0) * CFrame.Angles(math.rad(PROP_ROTATION_X), math.rad(PROP_ROTATION_Y), math.rad(PROP_ROTATION_Z))

    if currentPosition == Vector3.new() then
        currentPosition = targetPos
        currentRotation = targetRot
    else
        local alphaPos = math.clamp(dt * 22, 0, 1)
        local alphaRot = math.clamp(dt * 20, 0, 1)
        currentPosition = currentPosition:Lerp(targetPos, alphaPos)
        currentRotation = currentRotation:Lerp(targetRot, alphaRot)
    end

    local playerBaseCF = CFrame.new(currentPosition) * currentRotation
    local currentTime = os.clock()
    local shouldSendRemote = false
    if currentTime >= nextNetworkUpdate then
        nextNetworkUpdate = currentTime + NETWORK_INTERVAL
        shouldSendRemote = true
    end

    for _, prop in ipairs(activeProps) do
        if prop and prop.Parent and rawBaseOffsets[prop] then
            local rawRel = rawBaseOffsets[prop]
            local scaledPos = rawRel.Position * SCALE_MULTIPLIER
            local finalCFrame = playerBaseCF * CFrame.new(scaledPos) * rawRel.Rotation

            local idx = propIndicesMap[prop]
            if idx and idx >= 1 and idx <= 4 then
                finalCFrame = finalCFrame * lifeTremorCF
            end

            updatePropTransform(prop, finalCFrame, shouldSendRemote, idx)
        end
    end
end)

-- === INTERFAZ GRÁFICA ===
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 280, 0, 360)
mainFrame.Position = UDim2.new(0.5, -140, 0.5, -180)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui

Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 8)

local topBar = Instance.new("Frame", mainFrame)
topBar.Size = UDim2.new(1, 0, 0, 30)
topBar.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
topBar.BorderSizePixel = 0
Instance.new("UICorner", topBar).CornerRadius = UDim.new(0, 8)

local mainTitle = Instance.new("TextLabel", topBar)
mainTitle.Size = UDim2.new(1, -90, 1, 0)
mainTitle.Position = UDim2.new(0, 10, 0, 0)
mainTitle.BackgroundTransparency = 1
mainTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
mainTitle.TextSize = 13
mainTitle.Font = Enum.Font.GothamBold
mainTitle.TextXAlignment = Enum.TextXAlignment.Left
mainTitle.Text = "Prime - Fase 1 (Optimizado)"

local closeButton = Instance.new("TextButton", topBar)
closeButton.Size = UDim2.new(0, 30, 0, 30)
closeButton.Position = UDim2.new(1, -30, 0, 0)
closeButton.BackgroundTransparency = 1
closeButton.TextColor3 = Color3.fromRGB(200, 50, 50)
closeButton.TextSize = 16
closeButton.Font = Enum.Font.GothamBold
closeButton.Text = "X"

local minimizeButton = Instance.new("TextButton", topBar)
minimizeButton.Size = UDim2.new(0, 30, 0, 30)
minimizeButton.Position = UDim2.new(1, -60, 0, 0)
minimizeButton.BackgroundTransparency = 1
minimizeButton.TextColor3 = Color3.fromRGB(200, 200, 200)
minimizeButton.TextSize = 18
minimizeButton.Font = Enum.Font.GothamBold
minimizeButton.Text = "-"

local menuScroll = Instance.new("ScrollingFrame", mainFrame)
menuScroll.Size = UDim2.fromScale(0.95, 0.84)
menuScroll.Position = UDim2.fromScale(0.025, 0.12)
menuScroll.BackgroundTransparency = 1
menuScroll.ScrollBarThickness = 4
menuScroll.CanvasSize = UDim2.fromScale(0, 1.2)
menuScroll.Visible = true

local menuLayout = Instance.new("UIListLayout", menuScroll)
menuLayout.SortOrder = Enum.SortOrder.LayoutOrder
menuLayout.Padding = UDim.new(0.03, 0)
menuLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

local creditLabel = Instance.new("TextLabel", mainFrame)
creditLabel.Size = UDim2.new(1, -10, 0, 15)
creditLabel.Position = UDim2.new(0, 5, 1, -18)
creditLabel.BackgroundTransparency = 1
creditLabel.TextColor3 = Color3.fromRGB(120, 120, 120)
creditLabel.TextSize = 10
creditLabel.Font = Enum.Font.GothamMedium
creditLabel.TextXAlignment = Enum.TextXAlignment.Right
creditLabel.Text = "Prime Phase 1"

local function createButton(parent, text, color, callback)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.fromScale(0.9, 0.13)
    btn.BackgroundColor3 = color or Color3.fromRGB(45, 120, 220)
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 11
    btn.Font = Enum.Font.GothamBold
    btn.Text = text
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    btn.MouseButton1Click:Connect(callback)
    return btn
end

local executeBtn = createButton(menuScroll, "Spawn y Seguir (Activar)", Color3.fromRGB(45, 120, 220), function()
    isFollowing = not isFollowing
    if isFollowing then
        executeStructureMove()
        executeBtn.Text = "Prime: ACTIVO"
        executeBtn.BackgroundColor3 = Color3.fromRGB(40, 160, 80)
    else
        executeBtn.Text = "Spawn y Seguir (Activar)"
        executeBtn.BackgroundColor3 = Color3.fromRGB(45, 120, 220)
    end
end)

createButton(menuScroll, "⚙ Configuración / Sliders", Color3.fromRGB(60, 60, 60), function()
    menuScroll.Visible = false
    if _G.showPrimeSettings then _G.showPrimeSettings() end
end)

local settingsScroll = Instance.new("ScrollingFrame", mainFrame)
settingsScroll.Size = UDim2.fromScale(0.95, 0.84)
settingsScroll.Position = UDim2.fromScale(0.025, 0.12)
settingsScroll.BackgroundTransparency = 1
settingsScroll.ScrollBarThickness = 4
settingsScroll.CanvasSize = UDim2.fromScale(0, 2.5)
settingsScroll.Visible = false

_G.showPrimeSettings = function()
    settingsScroll.Visible = true
    mainTitle.Text = "Ajustes - Prime"
end

local settingsLayout = Instance.new("UIListLayout", settingsScroll)
settingsLayout.SortOrder = Enum.SortOrder.LayoutOrder
settingsLayout.Padding = UDim.new(0.02, 0)
settingsLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

local backBtn = Instance.new("TextButton", settingsScroll)
backBtn.Size = UDim2.fromScale(0.9, 0.06)
backBtn.BackgroundColor3 = Color3.fromRGB(90, 40, 40)
backBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
backBtn.TextSize = 11
backBtn.Font = Enum.Font.GothamBold
backBtn.Text = "← Volver al Menú"
Instance.new("UICorner", backBtn).CornerRadius = UDim.new(0, 6)
backBtn.MouseButton1Click:Connect(function()
    settingsScroll.Visible = false
    menuScroll.Visible = true
    mainTitle.Text = "Prime - Fase 1"
end)

local function createRealSlider(parent, titleText, minVal, maxVal, currentVal, callback)
    local container = Instance.new("Frame", parent)
    container.Size = UDim2.fromScale(0.9, 0.11)
    container.BackgroundTransparency = 1

    local titleLabel = Instance.new("TextLabel", container)
    titleLabel.Size = UDim2.fromScale(1, 0.35)
    titleLabel.BackgroundTransparency = 1
    titleLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    titleLabel.TextSize = 11
    titleLabel.Font = Enum.Font.GothamMedium
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Text = titleText

    local sliderBar = Instance.new("Frame", container)
    sliderBar.Size = UDim2.fromScale(1, 0.3)
    sliderBar.Position = UDim2.fromScale(0, 0.5)
    sliderBar.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    sliderBar.BorderSizePixel = 0
    Instance.new("UICorner", sliderBar).CornerRadius = UDim.new(1, 0)

    local fillBar = Instance.new("Frame", sliderBar)
    fillBar.Size = UDim2.fromScale(math.clamp((currentVal - minVal) / (maxVal - minVal), 0, 1), 1)
    fillBar.BackgroundColor3 = Color3.fromRGB(60, 180, 120)
    fillBar.BorderSizePixel = 0
    Instance.new("UICorner", fillBar).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("TextButton", sliderBar)
    knob.Size = UDim2.new(0, 14, 0, 14)
    knob.AnchorPoint = Vector2.new(0.5, 0.5)
    knob.Position = UDim2.fromScale(fillBar.Size.X.Scale, 0.5)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.Text = ""
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    local dragging = false

    local function updateInput(input)
        local pos = sliderBar.AbsolutePosition.X
        local size = sliderBar.AbsoluteSize.X
        local clickX = math.clamp(input.Position.X, pos, pos + size)
        local delta = (clickX - pos) / size
        
        local val = minVal + (maxVal - minVal) * delta
        fillBar.Size = UDim2.fromScale(delta, 1)
        knob.Position = UDim2.fromScale(delta, 0.5)
        callback(val)
    end

    knob.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
        end
    end)

    sliderBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            updateInput(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            updateInput(input)
        end
    end)
end

createRealSlider(settingsScroll, "Scale Multiplier", 0.5, 10, SCALE_MULTIPLIER, function(val) SCALE_MULTIPLIER = val end)
createRealSlider(settingsScroll, "Height Offset", 0, 30, MECH_HEIGHT_OFFSET, function(val) MECH_HEIGHT_OFFSET = val end)
createRealSlider(settingsScroll, "Position Z (Forward/Back)", -30, 30, BACKWARD_OFFSET_Z, function(val) BACKWARD_OFFSET_Z = val end)
createRealSlider(settingsScroll, "Position X (Left/Right)", -30, 30, BACKWARD_OFFSET_X, function(val) BACKWARD_OFFSET_X = val end)
createRealSlider(settingsScroll, "Rotation X", 0, 360, PROP_ROTATION_X, function(val) PROP_ROTATION_X = val end)
createRealSlider(settingsScroll, "Rotation Y", 0, 360, PROP_ROTATION_Y, function(val) PROP_ROTATION_Y = val end)
createRealSlider(settingsScroll, "Rotation Z", 0, 360, PROP_ROTATION_Z, function(val) PROP_ROTATION_Z = val end)

local primeIcon = Instance.new("TextButton")
primeIcon.Name = "PrimeFloatingIcon"
primeIcon.Size = UDim2.new(0, 45, 0, 45)
primeIcon.Position = UDim2.new(0, 20, 0.5, -22)
primeIcon.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
primeIcon.TextColor3 = Color3.fromRGB(255, 255, 255)
primeIcon.TextSize = 11
primeIcon.Font = Enum.Font.GothamBold
primeIcon.Text = "Prime"
primeIcon.Visible = false
primeIcon.Parent = screenGui

Instance.new("UICorner", primeIcon).CornerRadius = UDim.new(1, 0)
local iconStroke = Instance.new("UIStroke", primeIcon)
iconStroke.Color = Color3.fromRGB(60, 140, 255)
iconStroke.Thickness = 2

local function makeDraggable(frame, handle)
    local dragging, dragStart, startPos
    handle.InputBegan:Connect(function(input)
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
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

makeDraggable(mainFrame, topBar)
makeDraggable(primeIcon, primeIcon)

closeButton.MouseButton1Click:Connect(function()
    isFollowing = false
    screenGui:Destroy()
end)

minimizeButton.MouseButton1Click:Connect(function()
    mainFrame.Visible = false
    primeIcon.Visible = true
end)

primeIcon.MouseButton1Click:Connect(function()
    primeIcon.Visible = false
    mainFrame.Visible = true
end)
