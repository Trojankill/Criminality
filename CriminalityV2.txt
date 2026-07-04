local environment = getfenv and getfenv() or _ENV
local unpackValues = table.unpack or unpack
local interfaceLibrary
if readfile and isfile and isfile("Library Source.lua") then
    local ok, lib = pcall(function()
        return loadstring(readfile("Library Source.lua"))()
    end)
    if ok then
        interfaceLibrary = lib
    end
end
if not interfaceLibrary then
    local libraryUrls = {
        "https://raw.githubusercontent.com/jianlobiano/Serotonin-Library-Modified/refs/heads/main/Library.lua",
        "https://raw.githubusercontent.com/jianlobiano/LOADER/refs/heads/main/JX-UI/temp",
    }
    for _, libraryUrl in ipairs(libraryUrls) do
        local ok, libraryResult = pcall(function()
            return loadstring(game:HttpGet(libraryUrl))()
        end)
        if ok and libraryResult then
            interfaceLibrary = libraryResult
            break
        end
    end
end
assert(interfaceLibrary, "Unable to load interface library")
interfaceLibrary.Folders = {
    Directory = "JX-Criminality",
    Configs = "JX-Criminality/Configs",
    Assets = "JX-Criminality/Assets",
}
interfaceLibrary.Theme.Accent = Color3.fromRGB(255, 80, 80)
interfaceLibrary.Theme.AccentGradient = Color3.fromRGB(120, 20, 20)
interfaceLibrary:ChangeTheme("Accent", Color3.fromRGB(255, 80, 80))
interfaceLibrary:ChangeTheme("AccentGradient", Color3.fromRGB(120, 20, 20))
local mainWindow = interfaceLibrary:Window({
    Name = "Criminality",
    SubName = "JX | Dsc.gg/getjxs",
    Logo = "85279746515974",
})
local playersService = game:GetService("Players")
local runService = game:GetService("RunService")
local replicatedStorage = game:GetService("ReplicatedStorage")
local userInputService = game:GetService("UserInputService")
local coreGui = game:GetService("CoreGui")
local lighting = game:GetService("Lighting")
local httpService = game:GetService("HttpService")
local marketplaceService = game:GetService("MarketplaceService")
local virtualUser = game:GetService("VirtualUser")
local textChatService = game:GetService("TextChatService")
local tweenService = game:GetService("TweenService")
local localPlayer = playersService.LocalPlayer
local localMouse = localPlayer:GetMouse()
local currentCamera = workspace.CurrentCamera
local mousePosition = userInputService:GetMouseLocation()
local remoteEvents = replicatedStorage:FindFirstChild("Events")
local runtimeState = {
    connections = {
    },
    tasks = {
    },
    data = {
    },
    originalValues = {
    },
    wallBang = false,
}
local function disconnectConnection(connectionName)
    local connection = runtimeState.connections[connectionName]
    if connection then
        pcall(function()
            connection:Disconnect()
        end)
        runtimeState.connections[connectionName] = nil
    end
end
local function stopTask(taskName)
    local runningTask = runtimeState.tasks[taskName]
    if runningTask then
        pcall(task.cancel, runningTask)
        runtimeState.tasks[taskName] = nil
    end
end
local function connectSignal(connectionName, signal, callback)
    disconnectConnection(connectionName)
    runtimeState.connections[connectionName] = signal:Connect(callback)
    return runtimeState.connections[connectionName]
end
local function startTask(taskName, callback)
    stopTask(taskName)
    runtimeState.tasks[taskName] = task.spawn(callback)
    return runtimeState.tasks[taskName]
end
local function getCharacter(player)
    return player and player.Character
end
local function getHumanoid(player)
    local character = getCharacter(player)
    return character and character:FindFirstChildOfClass("Humanoid")
end
local function getRootPart(player)
    local character = getCharacter(player)
    return character and character:FindFirstChild("HumanoidRootPart")
end
local function isPlayerDowned(player)
    local succeeded, downed = pcall(function()
        return replicatedStorage.CharStats[player.Name].Downed.Value
    end)
    return succeeded and downed == true
end
local function shouldIgnorePlayer(player, settings)
    if not player or player == localPlayer then
        return true
    end
    local character = getCharacter(player)
    local humanoid = getHumanoid(player)
    if not character or not humanoid or humanoid.Health <= 0 then
        return true
    end
    if settings.FriendCheck or settings.CheckFriend then
        local isFriend = false
        pcall(function()
            isFriend = localPlayer:IsFriendsWith(player.UserId)
        end)
        if isFriend then
            return true
        end
    end
    if settings.EnemyCheck or settings.CheckEnemy then
        if player.Team and localPlayer.Team and player.Team == localPlayer.Team then
            return true
        end
    end
    if settings.CheckTeam and player.Team and localPlayer.Team and player.Team == localPlayer.Team then
        return true
    end
    if settings.CheckWhitelist and runtimeState.list.whitelist[player.Name] then
        return true
    end
    if settings.CheckTarget and next(runtimeState.list.targets) and not runtimeState.list.targets[player.Name] then
        return true
    end
    if settings.CheckDown and isPlayerDowned(player) then
        return true
    end
    if settings.CheckForceShield and character:FindFirstChildOfClass("ForceField") then
        return true
    end
    return false
end
local function hasLineOfSight(targetPart)
    if runtimeState.wallBang then
        return true
    end
    local character = getCharacter(localPlayer)
    if not character or not targetPart then
        return false
    end
    local raycastParameters = RaycastParams.new()
    raycastParameters.FilterType = Enum.RaycastFilterType.Exclude
    raycastParameters.FilterDescendantsInstances = {
        character,
        currentCamera,
        targetPart.Parent,
    }
    raycastParameters.IgnoreWater = true
    return workspace:Raycast(currentCamera.CFrame.Position, targetPart.Position - currentCamera.CFrame.Position, raycastParameters) == nil
end
local function getTargetPart(character, partName, targetBodyParts)
    if not character then
        return nil
    end
    if partName ~= "Random" then
        local targetPart = character:FindFirstChild(partName)
        if targetPart and targetPart:IsA("BasePart") then
            return targetPart
        end
    end
    local values1 = {
    }
    for _, currentName in ipairs(targetBodyParts) do
        local targetPart = character:FindFirstChild(currentName)
        if targetPart and targetPart:IsA("BasePart") then
            values1[#values1 + 1] = targetPart
        end
    end
    if #values1 > 0 then
        return values1[math.random(1, #values1)]
    end
end
local function projectToScreen(part)
    local viewportPoint, isVisible = currentCamera:WorldToViewportPoint(part.Position)
    return Vector2.new(viewportPoint.X, viewportPoint.Y), isVisible and viewportPoint.Z > 0
end
local function getScreenCenter(settings)
    if settings.FOVCenterOnly or settings.CircleCenterOnly or settings.FOVCentered then
        return Vector2.new(currentCamera.ViewportSize.X * 0.5, currentCamera.ViewportSize.Y * 0.5)
    end
    return userInputService:GetMouseLocation()
end
local function selectTarget(settings, targetBodyParts, mode)
    local bestPlayer, bestTargetPart
    local bestScore = math.huge
    local screenCenter = getScreenCenter(settings)
    local localRootPart = getRootPart(localPlayer)
    local lockedPlayer = settings.LockedTarget
    if (settings.StickyAim or settings.FullLock) and lockedPlayer and not shouldIgnorePlayer(lockedPlayer, settings) then
        local lockedCharacter = getCharacter(lockedPlayer)
        local lockedPart = getTargetPart(lockedCharacter, settings.HitPart or "Head", targetBodyParts)
        if lockedPart and (not settings.WallCheck or hasLineOfSight(lockedPart)) then
            if settings.FullLock then
                return lockedPlayer, lockedPart, 0
            end
            local lockedPosition, lockedVisible = projectToScreen(lockedPart)
            if lockedVisible then
                local lockedDistance = (lockedPosition - screenCenter).Magnitude
                local lockedRadius = settings.CircleRadius or settings.FOVRadius or settings.FOV or 120
                if lockedDistance <= lockedRadius then
                    return lockedPlayer, lockedPart, lockedDistance
                end
            end
        end
    end
    for _, currentPlayer3 in ipairs(playersService:GetPlayers()) do
        if not shouldIgnorePlayer(currentPlayer3, settings) then
            local character1 = getCharacter(currentPlayer3)
            local part = getTargetPart(character1, settings.HitPart or "Head", targetBodyParts)
            if part and (not settings.WallCheck or hasLineOfSight(part)) then
                local projectedPosition, isOnScreen = projectToScreen(part)
                if isOnScreen then
                    local screenDistance = (projectedPosition - screenCenter).Magnitude
                    local targetDistance = localRootPart and (part.Position - localRootPart.Position).Magnitude or math.huge
                    local score
                    if mode == "Distance" then
                        if targetDistance <= (settings.MaxDistance or settings.Distance or math.huge) then
                            score = targetDistance
                        end
                    elseif mode == "Circle" then
                        if screenDistance <= (settings.CircleRadius or settings.FOVRadius or settings.FOV or 120) then
                            score = screenDistance
                        end
                    else
                        if screenDistance <= (settings.FOVRadius or settings.FOV or 120) then
                            score = screenDistance
                        end
                    end
                    if score and score < bestScore then
                        bestScore = score
                        bestPlayer = currentPlayer3
                        bestTargetPart = part
                    end
                end
            end
        end
    end
    if settings.StickyAim or settings.FullLock then
        settings.LockedTarget = bestPlayer
    end
    return bestPlayer, bestTargetPart, bestScore
end
local function createDrawingCircle(fill, zIndex, color)
    local drawingObject = Drawing.new("Circle")
    drawingObject.Visible = false
    drawingObject.Filled = fill
    drawingObject.Color = color
    drawingObject.Thickness = 1
    drawingObject.Transparency = fill and 0.85 or 1
    drawingObject.NumSides = 64
    drawingObject.Radius = 120
    drawingObject.ZIndex = zIndex
    return drawingObject
end
local function addToggle(section, name, flag, defaultValue, callback)
    return section:Toggle({
        Name = name,
        Flag = flag,
        Default = defaultValue,
        Callback = callback,
    })
end
local function addSlider(section, name, flag, minimum, maximum, defaultValue, suffix, callback)
    return section:Slider({
        Name = name,
        Flag = flag,
        Min = minimum,
        Max = maximum,
        Default = defaultValue,
        Suffix = suffix or "",
        Callback = callback,
    })
end
local function addDropdown(section, name, flag, defaultValue, items, multiSelect, callback)
    return section:Dropdown({
        Name = name,
        Flag = flag,
        Default = defaultValue,
        Items = items,
        Multi = multiSelect or false,
        Callback = callback,
    })
end
local function createShotIdentifier()
    local currentValues = table.create(31)
    for currentIndex = 1, 30 do
        currentValues[currentIndex] = string.char(math.random(97, 122))
    end
    currentValues[31] = "0"
    return table.concat(currentValues)
end
runtimeState.list = {
    whitelist = {
    },
    targets = {
    },
    whitelistHighlights = {
    },
    targetHighlights = {
    },
    whitelistHighlightsEnabled = false,
    targetHighlightsEnabled = false,
    whitelistHighlightColor = Color3.fromRGB(0, 255, 0),
    targetHighlightColor = Color3.fromRGB(255, 0, 0),
}
local playerHighlightFolder = Instance.new("Folder")
playerHighlightFolder.Name = "JX_PL_Chams"
playerHighlightFolder.Parent = coreGui
local function getOtherPlayerNames()
    local values2 = {
    }
    for _, currentPlayer4 in ipairs(playersService:GetPlayers()) do
        if currentPlayer4 ~= localPlayer then
            values2[#values2 + 1] = currentPlayer4.Name
        end
    end
    return values2
end
local function createPlayerSelectionHighlight(player, color, highlightMap)
    if highlightMap[player] then
        return 
    end
    local instanceObject = Instance.new("Highlight")
    instanceObject.Name = player.Name .. "_PL"
    instanceObject.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    instanceObject.FillColor = color
    instanceObject.FillTransparency = 0.5
    instanceObject.OutlineColor = color
    instanceObject.OutlineTransparency = 0
    instanceObject.Enabled = true
    instanceObject.Adornee = player.Character
    instanceObject.Parent = playerHighlightFolder
    local characterConnection
    characterConnection = player.CharacterAdded:Connect(function(character)
        if instanceObject.Parent then
            instanceObject.Adornee = character
        else
            characterConnection:Disconnect()
        end
    end)
    highlightMap[player] = {
        highlight = instanceObject,
        connection = characterConnection,
    }
end
local function removePlayerSelectionHighlight(player, highlightMap)
    local highlightData = highlightMap[player]
    if not highlightData then
        return
    end
    pcall(function()
        highlightData.connection:Disconnect()
    end)
    pcall(function()
        highlightData.highlight:Destroy()
    end)
    highlightMap[player] = nil
end
local function synchronizePlayerSelectionHighlights(selectedPlayers, highlightMap, color)
    for player in pairs(highlightMap) do
        if not selectedPlayers[player.Name] then
            removePlayerSelectionHighlight(player, highlightMap)
        end
    end
    for _, player in ipairs(playersService:GetPlayers()) do
        if player ~= localPlayer and selectedPlayers[player.Name] then
            createPlayerSelectionHighlight(player, color, highlightMap)
        end
    end
end
local function replaceSelectedPlayerNames(selectedValues, destination)
    table.clear(destination)
    if type(selectedValues) ~= "table" then
        return
    end
    for key, value in pairs(selectedValues) do
        if type(key) == "number" and type(value) == "string" then
            destination[value] = true
        elseif type(key) == "string" and value == true then
            destination[key] = true
        end
    end
end
interfaceLibrary:KeybindList("Keybinds")
interfaceLibrary:Watermark({
    "JX",
    "By Jianlobiano",
    85279746515974,
})
interfaceLibrary:Notification({
    Title = "JX",
    Description = "Dont Forget To Join Discord Server | Dsc.gg/getjxs!",
    Duration = 5,
    Icon = "85279746515974",
})
startTask("wm", function()
    while task.wait(1) do
        local ping = 0
        pcall(function()
            ping = math.floor(game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue())
        end)
        interfaceLibrary:Watermark({
            85279746515974,
            "JX | Dsc.gg/getjxs",
            "By Jianlobiano",
            "FPS: " .. math.floor(1 / runService.RenderStepped:Wait()),
            "Ping: " .. ping .. "ms",
        })
    end
end)
userInputService.InputChanged:Connect(function(currentObject4)
    if currentObject4.UserInputType == Enum.UserInputType.MouseMovement then
        mousePosition = userInputService:GetMouseLocation()
    end
end)
local targetBodyParts = {
    "Head",
    "UpperTorso",
    "Torso",
    "LowerTorso",
    "HumanoidRootPart",
    "RightUpperArm",
    "RightLowerArm",
    "RightHand",
    "LeftUpperArm",
    "LeftLowerArm",
    "LeftHand",
    "RightUpperLeg",
    "RightLowerLeg",
    "RightFoot",
    "LeftUpperLeg",
    "LeftLowerLeg",
    "LeftFoot",
    "Right Arm",
    "Left Arm",
    "Right Leg",
    "Left Leg",
}
local desktopAimbot = {
    Enabled = false,
    Held = false,
    Smoothness = 0.05,
    SmoothnessOn = false,
    FOV = 120,
    HitPart = "Head",
    TeamCheck = false,
    WallCheck = true,
    CheckDown = true,
    CheckForceShield = true,
    CheckWhitelist = false,
    CheckTarget = false,
    DrawFOV = false,
    FOVCenterOnly = false,
    FOVFilled = false,
    FOVColor = Color3.fromRGB(255, 255, 255),
    FOVFillColor = Color3.fromRGB(175, 86, 86),
    PredictionX = 20,
    PredictionY = 20,
    PredictionOn = false,
    FriendCheck = false,
    EnemyCheck = false,
    FullLock = false,
    StickyAim = false,
    TweenSpeedOn = false,
    TweenSpeed = 20,
    LockedColor = Color3.fromRGB(255, 0, 0),
    LockedTarget = nil,
}
local desktopAimbotFovOutline = createDrawingCircle(false, 999, desktopAimbot.FOVColor)
local desktopAimbotFovFill = createDrawingCircle(true, 998, desktopAimbot.FOVFillColor)
runService.RenderStepped:Connect(function()
    local currentCharacter3 = getScreenCenter(desktopAimbot)
    desktopAimbotFovOutline.Position = currentCharacter3
    desktopAimbotFovOutline.Radius = desktopAimbot.FOV
    desktopAimbotFovOutline.Color = desktopAimbot.LockedTarget and desktopAimbot.LockedColor or desktopAimbot.FOVColor
    desktopAimbotFovOutline.Visible = desktopAimbot.DrawFOV
    desktopAimbotFovFill.Position = currentCharacter3
    desktopAimbotFovFill.Radius = desktopAimbot.FOV
    desktopAimbotFovFill.Color = desktopAimbot.FOVFillColor
    desktopAimbotFovFill.Visible = desktopAimbot.FOVFilled
    if not desktopAimbot.Enabled or not desktopAimbot.Held then
        desktopAimbot.LockedTarget = nil
        return 
    end
    local _, currentPlayer9 = selectTarget(desktopAimbot, targetBodyParts, "Camera")
    if not currentPlayer9 then
        return 
    end
    local pos = currentPlayer9.Position
    if desktopAimbot.PredictionOn then
        local humanoid1 = currentPlayer9.Parent and currentPlayer9.Parent:FindFirstChild("HumanoidRootPart")
        if humanoid1 then
            local humanoid2 = humanoid1.AssemblyLinearVelocity or humanoid1.Velocity
            pos = pos + Vector3.new(humanoid2.X * desktopAimbot.PredictionX * 0.01, humanoid2.Y * desktopAimbot.PredictionY * 0.01, humanoid2.Z * desktopAimbot.PredictionX * 0.01)
        end
    end
    local targetCFrame = CFrame.new(currentCamera.CFrame.Position, pos)
    local aimAlpha = 1
    if desktopAimbot.SmoothnessOn then
        aimAlpha = desktopAimbot.Smoothness
    end
    if desktopAimbot.TweenSpeedOn then
        aimAlpha = math.clamp(desktopAimbot.TweenSpeed / 100, 0.01, 1)
    end
    currentCamera.CFrame = aimAlpha < 1 and currentCamera.CFrame:Lerp(targetCFrame, aimAlpha) or targetCFrame
end)
local mobileAimbot = {
    Enabled = false,
    Held = false,
    ShowCircleBtn = true,
    Smoothness = 0.05,
    SmoothnessOn = false,
    FOV = 120,
    HitPart = "Head",
    TeamCheck = false,
    WallCheck = true,
    CheckDown = true,
    CheckForceShield = true,
    CheckWhitelist = false,
    CheckTarget = false,
    DrawFOV = false,
    FOVFilled = false,
    FOVColor = Color3.fromRGB(255, 255, 255),
    FOVFillColor = Color3.fromRGB(175, 86, 86),
    PredictionX = 20,
    PredictionY = 20,
    PredictionOn = false,
    FriendCheck = false,
    EnemyCheck = false,
    FullLock = false,
    StickyAim = false,
    TweenSpeedOn = false,
    TweenSpeed = 20,
    LockedColor = Color3.fromRGB(255, 0, 0),
    LockedTarget = nil,
    FOVCenterOnly = true,
    BtnPosX = 0,
    BtnPosY = 0,
    BtnSize = 60,
}
local mobileAimbotGui = Instance.new("ScreenGui")
mobileAimbotGui.Name = "JX_AimBtn"
mobileAimbotGui.ResetOnSpawn = false
mobileAimbotGui.IgnoreGuiInset = true
mobileAimbotGui.Enabled = false
mobileAimbotGui.Parent = coreGui
local mobileAimbotButton = Instance.new("TextButton")
mobileAimbotButton.AnchorPoint = Vector2.new(0.5, 0.5)
mobileAimbotButton.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
mobileAimbotButton.BackgroundTransparency = 0.2
mobileAimbotButton.Text = "AIM"
mobileAimbotButton.TextColor3 = Color3.new(1, 1, 1)
mobileAimbotButton.TextScaled = true
mobileAimbotButton.Font = Enum.Font.GothamBold
mobileAimbotButton.AutoButtonColor = false
mobileAimbotButton.Parent = mobileAimbotGui
local mobileAimbotCorner = Instance.new("UICorner")
mobileAimbotCorner.CornerRadius = UDim.new(1, 0)
mobileAimbotCorner.Parent = mobileAimbotButton
local mobileAimbotStroke = Instance.new("UIStroke")
mobileAimbotStroke.Color = Color3.fromRGB(255, 80, 80)
mobileAimbotStroke.Thickness = 2
mobileAimbotStroke.Parent = mobileAimbotButton
local mobileAimbotFovOutline = createDrawingCircle(false, 997, mobileAimbot.FOVColor)
local mobileAimbotFovFill = createDrawingCircle(true, 996, mobileAimbot.FOVFillColor)
local function updateMobileAimbotButton()
    mobileAimbotGui.Enabled = mobileAimbot.Enabled and mobileAimbot.ShowCircleBtn
    mobileAimbotButton.Size = UDim2.fromOffset(mobileAimbot.BtnSize, mobileAimbot.BtnSize)
    mobileAimbotButton.Position = UDim2.new(0.5, mobileAimbot.BtnPosX, 0.5, mobileAimbot.BtnPosY)
    mobileAimbotButton.BackgroundColor3 = mobileAimbot.Held and Color3.fromRGB(120, 20, 20) or Color3.fromRGB(20, 20, 20)
end
mobileAimbotButton.MouseButton1Click:Connect(function()
    mobileAimbot.Held = not mobileAimbot.Held
    updateMobileAimbotButton()
end)
runService.RenderStepped:Connect(function()
    local currentCharacter4 = getScreenCenter(mobileAimbot)
    mobileAimbotFovOutline.Position = currentCharacter4
    mobileAimbotFovOutline.Radius = mobileAimbot.FOV
    mobileAimbotFovOutline.Color = mobileAimbot.LockedTarget and mobileAimbot.LockedColor or mobileAimbot.FOVColor
    mobileAimbotFovOutline.Visible = mobileAimbot.DrawFOV
    mobileAimbotFovFill.Position = currentCharacter4
    mobileAimbotFovFill.Radius = mobileAimbot.FOV
    mobileAimbotFovFill.Color = mobileAimbot.FOVFillColor
    mobileAimbotFovFill.Visible = mobileAimbot.FOVFilled
    if not mobileAimbot.Enabled or not mobileAimbot.Held then
        mobileAimbot.LockedTarget = nil
        return 
    end
    local _, currentPlayer10 = selectTarget(mobileAimbot, targetBodyParts, "Camera")
    if not currentPlayer10 then
        return 
    end
    local pos = currentPlayer10.Position
    if mobileAimbot.PredictionOn then
        local humanoid3 = currentPlayer10.Parent and currentPlayer10.Parent:FindFirstChild("HumanoidRootPart")
        if humanoid3 then
            local humanoid4 = humanoid3.AssemblyLinearVelocity or humanoid3.Velocity
            pos = pos + Vector3.new(humanoid4.X * mobileAimbot.PredictionX * 0.01, humanoid4.Y * mobileAimbot.PredictionY * 0.01, humanoid4.Z * mobileAimbot.PredictionX * 0.01)
        end
    end
    local targetCFrame2 = CFrame.new(currentCamera.CFrame.Position, pos)
    local aimAlpha2 = 1
    if mobileAimbot.SmoothnessOn then
        aimAlpha2 = mobileAimbot.Smoothness
    end
    if mobileAimbot.TweenSpeedOn then
        aimAlpha2 = math.clamp(mobileAimbot.TweenSpeed / 100, 0.01, 1)
    end
    currentCamera.CFrame = aimAlpha2 < 1 and currentCamera.CFrame:Lerp(targetCFrame2, aimAlpha2) or targetCFrame2
end)
local bulletBeamStyles = {
    Classic = {
        id = "rbxassetid://446111271",
        len = 1,
        spd = 1,
    },
    Rainbow = {
        id = "rbxassetid://2490624870",
        len = 3,
        spd = 2,
    },
}
local bulletBeamSettings = {
    On = false,
    Col = Color3.fromRGB(255, 0, 0),
    Thick = 0.1,
    Life = 2,
    Trans = 0.65,
    Design = "Classic",
}
local function createBulletBeam(currentValues2, secondaryValue)
    local ter = workspace:FindFirstChildOfClass("Terrain")
    if not ter then
        return 
    end
    local instanceObject2 = Instance.new("Attachment")
    local instanceObject3 = Instance.new("Attachment")
    instanceObject2.Position = currentValues2
    instanceObject3.Position = secondaryValue
    instanceObject2.Parent = ter
    instanceObject3.Parent = ter
    local beam = Instance.new("Beam")
    local localValue4 = bulletBeamStyles[bulletBeamSettings.Design] or bulletBeamStyles.Classic
    beam.Attachment0 = instanceObject2
    beam.Attachment1 = instanceObject3
    beam.Color = ColorSequence.new(bulletBeamSettings.Col)
    beam.Width0 = bulletBeamSettings.Thick
    beam.Width1 = bulletBeamSettings.Thick * 0.4
    beam.Texture = localValue4.id
    beam.TextureLength = localValue4.len
    beam.TextureSpeed = localValue4.spd
    beam.TextureMode = Enum.TextureMode.Wrap
    beam.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, bulletBeamSettings.Trans * 0.4),
        NumberSequenceKeypoint.new(0.5, bulletBeamSettings.Trans),
        NumberSequenceKeypoint.new(1, 1),
    })
    beam.FaceCamera = true
    beam.LightEmission = 0.6
    beam.LightInfluence = 0.1
    beam.Parent = ter
    task.delay(bulletBeamSettings.Life, function()
        pcall(function()
            beam:Destroy()
        end)
        pcall(function()
            instanceObject2:Destroy()
        end)
        pcall(function()
            instanceObject3:Destroy()
        end)
    end)
end
runtimeState.data.clearBulletTracerConnections = function()
    local connections = runtimeState.data.bulletTracerConnections or {}
    for _, connection in ipairs(connections) do
        pcall(connection.Disconnect, connection)
    end
    runtimeState.data.bulletTracerConnections = {}
end
runtimeState.data.traceBulletDirections = function(tool, directions, fallbackOrigin)
    if not bulletBeamSettings.On or type(directions) ~= "table" then
        return
    end
    local character = getCharacter(localPlayer)
    if not character or character:FindFirstChildOfClass("Tool") ~= tool then
        return
    end
    local muzzle = tool and (tool:FindFirstChild("Muzzle", true) or tool:FindFirstChild("FirePoint", true))
    if not muzzle then
        local weaponHandle = tool and tool:FindFirstChild("WeaponHandle", true)
        muzzle = weaponHandle and (weaponHandle:FindFirstChild("Muzzle", true) or weaponHandle:FindFirstChild("FirePoint", true))
    end
    local origin
    if muzzle then
        if muzzle:IsA("Attachment") then
            origin = muzzle.WorldPosition
        elseif muzzle:IsA("BasePart") then
            origin = muzzle.Position
        end
    end
    if not origin and typeof(fallbackOrigin) == "Vector3" then
        origin = fallbackOrigin
    end
    origin = origin or currentCamera.CFrame.Position
    for _, direction in pairs(directions) do
        if typeof(direction) == "Vector3" and direction.Magnitude > 0 then
            local raycastParameters = RaycastParams.new()
            raycastParameters.FilterType = Enum.RaycastFilterType.Exclude
            raycastParameters.FilterDescendantsInstances = {currentCamera, character, tool}
            raycastParameters.IgnoreWater = true
            local result = workspace:Raycast(origin, direction.Unit * 1000, raycastParameters)
            createBulletBeam(origin, result and result.Position or origin + direction.Unit * 500)
        end
    end
end
runtimeState.data.setupBulletTracerConnections = function()
    runtimeState.data.clearBulletTracerConnections()
    local connections = runtimeState.data.bulletTracerConnections
    local events2 = replicatedStorage:FindFirstChild("Events2")
    local visualize = events2 and events2:FindFirstChild("Visualize")
    if visualize and visualize.Event then
        connections[#connections + 1] = visualize.Event:Connect(function(arg1, arg2, arg3, tool, arg5, origin, directions)
            runtimeState.data.traceBulletDirections(tool, directions, origin)
        end)
    end
    local events = replicatedStorage:FindFirstChild("Events")
    if events then
        local function attachRemote(remote)
            if not remote:IsA("RemoteEvent") or remote.Name == "ZFKLF__H" then
                return
            end
            connections[#connections + 1] = remote.OnClientEvent:Connect(function(...)
                local args = {...}
                local tool = args[3]
                local directions = args[6]
                if typeof(tool) == "Instance" and tool:IsA("Tool") and type(directions) == "table" then
                    runtimeState.data.traceBulletDirections(tool, directions, args[5])
                end
            end)
        end
        for _, remote in ipairs(events:GetChildren()) do
            attachRemote(remote)
        end
        connections[#connections + 1] = events.ChildAdded:Connect(attachRemote)
    end
end
runtimeState.data.setupBulletTracerConnections()
local rageBot = {
    Enabled = false,
    ShootSpeed = 0.00505,
    CheckDown = true,
    CheckTeam = false,
    WallCheck = false,
    FindType = "Distance",
    MaxDistance = 500,
    CircleRadius = 150,
    ShowCircle = false,
    CircleCenterOnly = false,
    FillCircle = false,
    CircleColor = Color3.fromRGB(255, 50, 50),
    FillColor = Color3.fromRGB(255, 50, 50),
    NotifyHit = true,
    CheckWhitelist = false,
    CheckTarget = false,
    CheckForceShield = true,
    LockedColor = Color3.fromRGB(0, 255, 0),
    LockedTarget = nil,
    MobileUI = false,
    BeastMode = false,
}
local rageBotFovOutline = createDrawingCircle(false, 995, rageBot.CircleColor)
local rageBotFovFill = createDrawingCircle(true, 994, rageBot.FillColor)
local rageFireRemote
local rageHitRemote
local function findRageRemotes()
    rageFireRemote = rageFireRemote or replicatedStorage:FindFirstChild("GNX_S", true)
    rageHitRemote = rageHitRemote or replicatedStorage:FindFirstChild("ZFKLF__H", true)
end
local function fireRageShot(currentPlayer11, part)
    findRageRemotes()
    if not rageFireRemote or not rageHitRemote then
        return 
    end
    local character2 = getCharacter(localPlayer)
    local rootPart2 = getRootPart(localPlayer)
    local tool = character2 and character2:FindFirstChildOfClass("Tool")
    local vals = tool and tool:FindFirstChild("Values")
    local ammo = vals and vals:FindFirstChild("SERVER_Ammo")
    if not character2 or not rootPart2 or not tool or not tool:FindFirstChild("Hitmarker") or (not rageBot.BeastMode and (not ammo or ammo.Value <= 0)) then
        return 
    end
    local localValue5 = createShotIdentifier()
    local org = rootPart2.Position + Vector3.new(math.random(-18, 18), math.random(-18, 18), math.random(-18, 18))
    pcall(function()
        rageFireRemote:FireServer(tick(), localValue5, tool, "FDS9I83", org, {
            nil,
        }, false)
    end)
    if rageBot.ShootSpeed > 0 and not rageBot.BeastMode then
        task.wait(rageBot.ShootSpeed)
    end
    pcall(function()
        rageHitRemote:FireServer(utf8.char(129480), tool, localValue5, 1, part, part.Position, nil)
    end)
    if ammo and not rageBot.BeastMode then
        ammo.Value = math.max(ammo.Value - 1, 0)
    end
    if bulletBeamSettings.On then
        createBulletBeam(org, part.Position)
    end
    if rageBot.NotifyHit then
        local humanoid5 = getHumanoid(currentPlayer11)
        interfaceLibrary:Notification({
            Title = "Rage Bot",
            Description = currentPlayer11.Name .. " : Hit | HP " .. tostring(humanoid5 and math.floor(humanoid5.Health) or 0),
            Duration = 2,
            Icon = "85279746515974",
        })
    end
end
local function startRageBot()
    if runtimeState.tasks.rageBot then
        return 
    end
    startTask("rage", function()
        while rageBot.Enabled do
            local currentPlayer12, part = selectTarget(rageBot, targetBodyParts, rageBot.FindType)
            rageBot.LockedTarget = currentPlayer12
            if currentPlayer12 and part then
                pcall(fireRageShot, currentPlayer12, part)
            end
            task.wait()
        end
    end)
end
local function stopRageBot()
    rageBot.Enabled = false
    stopTask("rage")
end
runService.RenderStepped:Connect(function()
    local currentCharacter5 = getScreenCenter(rageBot)
    rageBotFovOutline.Position = currentCharacter5
    rageBotFovOutline.Radius = rageBot.CircleRadius
    rageBotFovOutline.Color = rageBot.LockedTarget and rageBot.LockedColor or rageBot.CircleColor
    rageBotFovOutline.Visible = rageBot.ShowCircle and rageBot.FindType == "Circle"
    rageBotFovFill.Position = currentCharacter5
    rageBotFovFill.Radius = rageBot.CircleRadius
    rageBotFovFill.Color = rageBot.FillColor
    rageBotFovFill.Visible = rageBot.ShowCircle and rageBot.FillCircle and rageBot.FindType == "Circle"
end)
local silentAimV2 = {
    Enabled = false,
    HitPart = "Head",
    HitChance = 100,
    WallCheck = true,
    CheckDown = true,
    CheckTeam = false,
    CheckForceShield = true,
    RandomInterval = 3,
    FOVRadius = 120,
    DrawFOV = false,
    FOVCenterOnly = false,
    FOVFilled = false,
    FOVColor = Color3.fromRGB(255, 255, 255),
    FOVFillColor = Color3.fromRGB(175, 86, 86),
    UseDist = false,
    MaxDist = 200,
    CheckWhitelist = false,
    CheckTarget = false,
    target = nil,
}
local silentAimV1 = {
    Enabled = false,
    HitPart = "Head",
    HitChance = 100,
    WallCheck = true,
    MaxDistance = 750,
    CheckDown = true,
    CheckTeam = false,
    CheckForceShield = true,
    CheckWhitelist = false,
    CheckTarget = false,
    DrawFOV = false,
    FOVFilled = false,
    FOVCenterOnly = false,
    FOVRadius = 100,
    FOVColor = Color3.fromRGB(255, 255, 255),
    FOVFillColor = Color3.fromRGB(175, 86, 86),
    target = nil,
}
local silentAimV2BodyParts = {
    "Head",
    "HumanoidRootPart",
    "Left Hand",
    "Right Hand",
    "Left Leg",
    "Right Leg",
}
local silentAimV1BodyParts = {
    "Head",
    "HumanoidRootPart",
    "Left Hand",
    "Right Hand",
    "Left Leg",
    "Right Leg",
}
local s2Ring = createDrawingCircle(false, 993, silentAimV2.FOVColor)
local s2Fill = createDrawingCircle(true, 992, silentAimV2.FOVFillColor)
local s1Ring = createDrawingCircle(false, 991, silentAimV1.FOVColor)
local s1Fill = createDrawingCircle(true, 990, silentAimV1.FOVFillColor)
runService.Heartbeat:Connect(function()
    if silentAimV2.Enabled then
        local currentPlayer13, part = selectTarget(silentAimV2, silentAimV2BodyParts, "Circle")
        if currentPlayer13 and part and (not silentAimV2.UseDist or ((getRootPart(localPlayer) and (part.Position - getRootPart(localPlayer).Position).Magnitude or math.huge) <= silentAimV2.MaxDist)) then
            silentAimV2.target = part
        else
            silentAimV2.target = nil
        end
    else
        silentAimV2.target = nil
    end
    if silentAimV1.Enabled then
        local currentPlayer14, part = selectTarget(silentAimV1, silentAimV1BodyParts, "Circle")
        if currentPlayer14 and part and (getRootPart(localPlayer) and (part.Position - getRootPart(localPlayer).Position).Magnitude or math.huge) <= silentAimV1.MaxDistance then
            silentAimV1.target = part
        else
            silentAimV1.target = nil
        end
    else
        silentAimV1.target = nil
    end
end)
runService.RenderStepped:Connect(function()
    local localValue6 = getScreenCenter(silentAimV2)
    s2Ring.Position = localValue6
    s2Ring.Radius = silentAimV2.FOVRadius
    s2Ring.Color = silentAimV2.FOVColor
    s2Ring.Visible = silentAimV2.DrawFOV
    s2Fill.Position = localValue6
    s2Fill.Radius = silentAimV2.FOVRadius
    s2Fill.Color = silentAimV2.FOVFillColor
    s2Fill.Visible = silentAimV2.FOVFilled
    local localValue7 = getScreenCenter(silentAimV1)
    s1Ring.Position = localValue7
    s1Ring.Radius = silentAimV1.FOVRadius
    s1Ring.Color = silentAimV1.FOVColor
    s1Ring.Visible = silentAimV1.DrawFOV
    s1Fill.Position = localValue7
    s1Fill.Radius = silentAimV1.FOVRadius
    s1Fill.Color = silentAimV1.FOVFillColor
    s1Fill.Visible = silentAimV1.FOVFilled
end)
local oldNamecall
if hookmetamethod and newcclosure and getnamecallmethod then
    oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
        local args = {
            ...,
        }
        local method = getnamecallmethod()
        local target
        local chance
        if silentAimV2.Enabled and silentAimV2.target then
            target = silentAimV2.target
            chance = silentAimV2.HitChance
        elseif silentAimV1.Enabled and silentAimV1.target then
            target = silentAimV1.target
            chance = silentAimV1.HitChance
        end
        if method == "Raycast" and target and math.random(1, 100) <= chance then
            local external = true
            pcall(function()
                external = not checkcaller()
            end)
            if external and self == workspace and args[1] and typeof(args[1]) == "Vector3" then
                args[2] = (target.Position - args[1]).Unit * 9999
                return oldNamecall(self, unpackValues(args))
            end
        end
        return oldNamecall(self, ...)
    end))
end
do
    local events2 = replicatedStorage:FindFirstChild("Events2")
    local visualize = events2 and events2:FindFirstChild("Visualize")
    local events = replicatedStorage:FindFirstChild("Events")
    local hitRemote = events and events:FindFirstChild("ZFKLF__H")
    if visualize and visualize.Event and hitRemote then
        connectSignal("silentAimV2Visualize", visualize.Event, function(arg1, shotIdentifier, arg3, tool, arg5, origin, pelletDirections)
            if not silentAimV2.Enabled or not tool then
                return
            end
            local character = getCharacter(localPlayer)
            if not character or character:FindFirstChildOfClass("ForceField") then
                return
            end
            local equippedTool = character:FindFirstChildOfClass("Tool")
            if equippedTool ~= tool or math.random(1, 100) > silentAimV2.HitChance then
                return
            end
            local targetPart = silentAimV2.target
            if not targetPart or not targetPart.Parent then
                return
            end
            if typeof(origin) ~= "Vector3" then
                local weaponHandle = tool:FindFirstChild("WeaponHandle", true) or tool:FindFirstChild("Handle", true)
                local firePoint = weaponHandle and (weaponHandle:FindFirstChild("FirePoint", true) or weaponHandle:FindFirstChild("Muzzle", true))
                origin = firePoint and (firePoint:IsA("Attachment") and firePoint.WorldPosition or firePoint.Position) or currentCamera.CFrame.Position
            end
            local pelletCount = type(pelletDirections) == "table" and math.max(1, #pelletDirections) or 1
            local redirectedDirections = table.create(pelletCount, CFrame.new(origin, targetPart.Position).LookVector)
            task.wait()
            for pelletIndex = 1, pelletCount do
                pcall(function()
                    hitRemote:FireServer(utf8.char(129480), tool, shotIdentifier, pelletIndex, targetPart, targetPart.Position, redirectedDirections[pelletIndex])
                end)
            end
            local hitmarker = tool:FindFirstChild("Hitmarker")
            if hitmarker and hitmarker.Fire then
                pcall(hitmarker.Fire, hitmarker, targetPart)
            end
            if bulletBeamSettings.On then
                createBulletBeam(origin, targetPart.Position)
            end
        end)
    end
end
local meleeAura = {
    Enabled = false,
    ShowAnim = false,
    HitPart = "Head",
    Distance = 15,
    CheckDown = true,
    CheckTeam = false,
    CheckWhitelist = false,
    CheckTarget = false,
    CheckForceShield = true,
    RandomInterval = 3,
    last = 0,
    rate = 0.1,
}
local meleeBodyParts = {
    "Head",
    "HumanoidRootPart",
    "Right Arm",
    "Left Arm",
    "Right Leg",
    "Left Leg",
}
local meleeDelayOptions = {
    Fists = 0.05,
    Knuckledusters = 0.05,
    Nunchucks = 0.05,
    Shiv = 0.05,
    Bat = 1,
    ["Metal-Bat"] = 1,
    Chainsaw = 2.5,
    Balisong = 0.05,
    Rambo = 0.3,
    Shovel = 3,
    Sledgehammer = 2,
    Katana = 0.1,
    Wrench = 0.1,
    ["Fire Axe"] = 2,
}
local function performMeleeHit(currentPlayer15)
    if shouldIgnorePlayer(currentPlayer15, meleeAura) then
        return 
    end
    local character3 = getCharacter(currentPlayer15)
    local rootPart3 = getRootPart(currentPlayer15)
    local myRoot = getRootPart(localPlayer)
    if not character3 or not rootPart3 or not myRoot or (rootPart3.Position - myRoot.Position).Magnitude > meleeAura.Distance then
        return 
    end
    local myChar = getCharacter(localPlayer)
    local tool = myChar and myChar:FindFirstChildOfClass("Tool")
    if not tool or tick() - meleeAura.last < meleeAura.rate then
        return 
    end
    local part = getTargetPart(character3, meleeAura.HitPart, meleeBodyParts) or character3:FindFirstChild("Head")
    if not part then
        return 
    end
    meleeAura.last = tick()
    meleeAura.rate = meleeDelayOptions[tool.Name] or 0.5
    local remoteObject = remoteEvents and remoteEvents:FindFirstChild("XMHH.2")
    local remoteObject2 = remoteEvents and remoteEvents:FindFirstChild("XMHH2.2")
    local swing = replicatedStorage:FindFirstChildWhichIsA("RemoteFunction", true)
    if swing then
        pcall(function()
            swing:InvokeServer(utf8.char(127838), tick(), part, "43TRFWX", "Normal", tick(), true)
        end)
    end
    if meleeAura.ShowAnim then
        local humanoid6 = getHumanoid(localPlayer)
        local anim = tool:FindFirstChild("AnimsFolder")
        anim = anim and anim:FindFirstChild("Slash1")
        if humanoid6 and anim then
            local bulletBeamSection = humanoid6:FindFirstChildOfClass("Animator") and humanoid6:FindFirstChildOfClass("Animator"):LoadAnimation(anim)
            if bulletBeamSection then
                bulletBeamSection:Play()
                bulletBeamSection:AdjustSpeed(1.3)
            end
        end
    end
    task.wait(0.3)
    if not meleeAura.Enabled then
        return 
    end
    if remoteObject then
        pcall(function()
            remoteObject:FireServer(tick(), createShotIdentifier(), tool, "43TRFWX", myRoot.Position, {
                part,
            }, false)
        end)
    end
    if remoteObject2 then
        pcall(function()
            remoteObject2:FireServer(utf8.char(129480), tool, createShotIdentifier(), 1, part, part.Position, nil)
        end)
    end
end
local function startMeleeAura()
    startTask("melee", function()
        while meleeAura.Enabled do
            for _, currentPlayer16 in ipairs(playersService:GetPlayers()) do
                if not meleeAura.Enabled then
                    break
                end
                pcall(performMeleeHit, currentPlayer16)
            end
            runService.Heartbeat:Wait()
        end
    end)
end
local function stopMeleeAura()
    meleeAura.Enabled = false
    stopTask("melee")
end
local playerEsp = {
    Enabled = false,
    TeamCheck = false,
    TeamCheckMethod = "Instance",
    TeamBoxColor = Color3.fromRGB(0, 255, 0),
    WallCheck = false,
    ForceShieldCheck = false,
    DistanceMode = false,
    DistanceStuds = 100,
    BoxEnabled = false,
    BoxType = "2D",
    BoxDesign = {
        "Box",
    },
    BoxColor = Color3.fromRGB(255, 255, 255),
    FillColor = Color3.fromRGB(255, 255, 255),
    BoxThickness = 2,
    BoxDrawTrans = 1,
    FillDrawTrans = 0.3,
    HealthBar = false,
    HealthBarColor = Color3.fromRGB(0, 255, 0),
    HealthText = false,
    HealthTextColor = Color3.fromRGB(255, 255, 255),
    SkeletonEnabled = false,
    SkeletonColor = Color3.fromRGB(255, 255, 255),
    HeadDotEnabled = false,
    HeadDotColor = Color3.fromRGB(255, 255, 255),
    TracerEnabled = false,
    TracerColor = Color3.fromRGB(255, 255, 255),
    TracerPosition = "Bottom",
    ChamsEnabled = false,
    ChamsType = {
        "Fill",
        "Outline",
    },
    ChamsFillColor = Color3.fromRGB(175, 25, 255),
    ChamsFillTrans = 0.5,
    ChamsOutlineEnabled = true,
    ChamsOutlineColor = Color3.fromRGB(255, 255, 255),
    ChamsOutlineTrans = 0,
    ToolEnabled = false,
    ToolColor = Color3.fromRGB(255, 255, 0),
    NameEnabled = false,
    NameColor = Color3.fromRGB(255, 255, 255),
    NameSize = 13,
    DistanceEnabled = false,
    DistanceColor = Color3.fromRGB(255, 255, 255),
    DistanceSize = 13,
}
local espFolder = Instance.new("Folder")
espFolder.Name = "JX_Highlight_Storage"
espFolder.Parent = coreGui
local playerEspObjects = {
}
local function hasSelectedOption(currentValues3, currentValue2)
    for _, currentObject5 in pairs(currentValues3 or {
    }) do
        if currentObject5 == currentValue2 then
            return true
        end
    end
    return false
end
local function createDrawingObject(kind, fill, currentScreenZ2)
    local drawingObject2 = Drawing.new(kind)
    drawingObject2.Visible = false
    drawingObject2.Transparency = 1
    drawingObject2.ZIndex = currentScreenZ2 or 5
    if kind == "Square" then
        drawingObject2.Filled = fill or false
        drawingObject2.Thickness = 1
        drawingObject2.Color = Color3.new(1, 1, 1)
    elseif kind == "Line" then
        drawingObject2.Thickness = 1
        drawingObject2.Color = Color3.new(1, 1, 1)
    elseif kind == "Text" then
        drawingObject2.Center = true
        drawingObject2.Outline = true
        drawingObject2.Font = 2
        drawingObject2.Size = 13
        drawingObject2.Color = Color3.new(1, 1, 1)
    elseif kind == "Circle" then
        drawingObject2.Filled = fill or false
        drawingObject2.Thickness = 1
        drawingObject2.NumSides = 24
        drawingObject2.Radius = 4
        drawingObject2.Color = Color3.new(1, 1, 1)
    end
    return drawingObject2
end
local function hidePlayerEsp(currentObject6)
    if not currentObject6 then
        return 
    end
    for _, currentData in pairs(currentObject6.createDrawingObject) do
        currentData.Visible = false
    end
    if currentObject6.cham then
        currentObject6.cham.Enabled = false
    end
end
local function removePlayerEsp(currentPlayer17)
    local currentObject7 = playerEspObjects[currentPlayer17]
    if not currentObject7 then
        return 
    end
    for _, currentData2 in pairs(currentObject7.createDrawingObject) do
        pcall(function()
            currentData2:Remove()
        end)
    end
    if currentObject7.cham then
        pcall(function()
            currentObject7.cham:Destroy()
        end)
    end
    if currentObject7.cc then
        pcall(function()
            currentObject7.cc:Disconnect()
        end)
    end
    playerEspObjects[currentPlayer17] = nil
end
local function createPlayerEsp(currentPlayer18)
    if currentPlayer18 == localPlayer or playerEspObjects[currentPlayer18] then
        return 
    end
    local values3 = {
        createDrawingObject = {
            box = createDrawingObject("Square", false, 6),
            fill = createDrawingObject("Square", true, 5),
            hp = createDrawingObject("Line", false, 7),
            hpBg = createDrawingObject("Line", false, 6),
            hpText = createDrawingObject("Text", false, 7),
            name = createDrawingObject("Text", false, 7),
            dist = createDrawingObject("Text", false, 7),
            tool = createDrawingObject("Text", false, 7),
            tracer = createDrawingObject("Line", false, 6),
            head = createDrawingObject("Circle", false, 7),
        },
        skel = {
        },
    }
    for currentIndex2 = 1, 14 do
        values3.skel[currentIndex2] = createDrawingObject("Line", false, 6)
        values3.createDrawingObject["sk" .. currentIndex2] = values3.skel[currentIndex2]
    end
    local instanceObject4 = Instance.new("Highlight")
    instanceObject4.Name = currentPlayer18.Name
    instanceObject4.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    instanceObject4.Enabled = false
    instanceObject4.Parent = espFolder
    instanceObject4.Adornee = currentPlayer18.Character
    values3.cham = instanceObject4
    values3.cc = currentPlayer18.CharacterAdded:Connect(function(currentCharacter6)
        instanceObject4.Adornee = currentCharacter6
    end)
    playerEspObjects[currentPlayer18] = values3
end
local function getPlayerTeamColor(currentPlayer19)
    if playerEsp.TeamCheck and currentPlayer19.Team and localPlayer.Team and currentPlayer19.Team == localPlayer.Team then
        return playerEsp.TeamBoxColor
    end
    return playerEsp.BoxColor
end
local function updateDrawingLine(currentData3, currentValues4, secondaryValue2, col, localValue8)
    currentData3.From = currentValues4
    currentData3.To = secondaryValue2
    currentData3.Color = col
    currentData3.Visible = localValue8
end
local skeletonConnections = {
    {
        "Head",
        "UpperTorso",
    },
    {
        "UpperTorso",
        "LowerTorso",
    },
    {
        "UpperTorso",
        "LeftUpperArm",
    },
    {
        "LeftUpperArm",
        "LeftLowerArm",
    },
    {
        "LeftLowerArm",
        "LeftHand",
    },
    {
        "UpperTorso",
        "RightUpperArm",
    },
    {
        "RightUpperArm",
        "RightLowerArm",
    },
    {
        "RightLowerArm",
        "RightHand",
    },
    {
        "LowerTorso",
        "LeftUpperLeg",
    },
    {
        "LeftUpperLeg",
        "LeftLowerLeg",
    },
    {
        "LeftLowerLeg",
        "LeftFoot",
    },
    {
        "LowerTorso",
        "RightUpperLeg",
    },
    {
        "RightUpperLeg",
        "RightLowerLeg",
    },
    {
        "RightLowerLeg",
        "RightFoot",
    },
}
local function updatePlayerEsp(currentPlayer20, currentObject8)
    if not playerEsp.Enabled or shouldIgnorePlayer(currentPlayer20, {
        CheckTeam = false,
        CheckWhitelist = false,
        CheckTarget = false,
        CheckDown = false,
        CheckForceShield = playerEsp.ForceShieldCheck,
    }) then
        hidePlayerEsp(currentObject8)
        return 
    end
    local character4 = getCharacter(currentPlayer20)
    local humanoid7 = getHumanoid(currentPlayer20)
    local rootPart4 = getRootPart(currentPlayer20)
    local myRoot = getRootPart(localPlayer)
    local head = character4 and character4:FindFirstChild("Head")
    if not character4 or not humanoid7 or not rootPart4 or not head then
        hidePlayerEsp(currentObject8)
        return 
    end
    local dist = myRoot and (rootPart4.Position - myRoot.Position).Magnitude or math.huge
    if playerEsp.TeamCheck and playerEsp.TeamCheckMethod == "Hide Team"
        and currentPlayer20.Team and localPlayer.Team and currentPlayer20.Team == localPlayer.Team then
        hidePlayerEsp(currentObject8)
        return
    end
    if playerEsp.DistanceMode and dist > playerEsp.DistanceStuds then
        hidePlayerEsp(currentObject8)
        return 
    end
    if playerEsp.WallCheck and not hasLineOfSight(rootPart4) then
        hidePlayerEsp(currentObject8)
        return 
    end
    local positionValue4, localValue9 = currentCamera:WorldToViewportPoint(rootPart4.Position)
    if not localValue9 or positionValue4.Z <= 0 then
        hidePlayerEsp(currentObject8)
        return 
    end
    local positionValue5, _ = currentCamera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
    local positionValue6, _ = currentCamera:WorldToViewportPoint(rootPart4.Position - Vector3.new(0, 3, 0))
    local height = math.abs(positionValue5.Y - positionValue6.Y)
    local width = height * 0.6
    local pos = Vector2.new(positionValue4.X - width * 0.5, positionValue4.Y - height * 0.5)
    local size = Vector2.new(width, height)
    local col = getPlayerTeamColor(currentPlayer20)
    local currentData4 = currentObject8.createDrawingObject
    currentData4.box.Position = pos
    currentData4.box.Size = size
    currentData4.box.Color = col
    currentData4.box.Thickness = playerEsp.BoxThickness
    currentData4.box.Transparency = playerEsp.BoxDrawTrans
    currentData4.box.Visible = playerEsp.BoxEnabled
    currentData4.fill.Position = pos
    currentData4.fill.Size = size
    currentData4.fill.Color = playerEsp.FillColor
    currentData4.fill.Transparency = playerEsp.FillDrawTrans
    currentData4.fill.Visible = playerEsp.BoxEnabled and hasSelectedOption(playerEsp.BoxDesign, "Fill")
    local pct = math.clamp(humanoid7.Health / math.max(humanoid7.MaxHealth, 1), 0, 1)
    updateDrawingLine(currentData4.hpBg, Vector2.new(pos.X - 5, pos.Y + size.Y), Vector2.new(pos.X - 5, pos.Y), Color3.new(0, 0, 0), playerEsp.HealthBar)
    currentData4.hpBg.Thickness = 4
    updateDrawingLine(currentData4.hp, Vector2.new(pos.X - 5, pos.Y + size.Y), Vector2.new(pos.X - 5, pos.Y + size.Y * (1 - pct)), playerEsp.HealthBarColor, playerEsp.HealthBar)
    currentData4.hp.Thickness = 2
    currentData4.hpText.Text = tostring(math.floor(humanoid7.Health))
    currentData4.hpText.Position = Vector2.new(pos.X - 18, pos.Y + size.Y * (1 - pct))
    currentData4.hpText.Color = playerEsp.HealthTextColor
    currentData4.hpText.Visible = playerEsp.HealthText
    currentData4.name.Text = currentPlayer20.Name
    currentData4.name.Position = Vector2.new(pos.X + size.X * 0.5, pos.Y - 15)
    currentData4.name.Size = playerEsp.NameSize
    currentData4.name.Color = playerEsp.NameColor
    currentData4.name.Visible = playerEsp.NameEnabled
    currentData4.dist.Text = tostring(math.floor(dist)) .. "m"
    currentData4.dist.Position = Vector2.new(pos.X + size.X * 0.5, pos.Y + size.Y + 3)
    currentData4.dist.Size = playerEsp.DistanceSize
    currentData4.dist.Color = playerEsp.DistanceColor
    currentData4.dist.Visible = playerEsp.DistanceEnabled
    local tool = character4:FindFirstChildOfClass("Tool")
    currentData4.tool.Text = tool and tool.Name or "None"
    currentData4.tool.Position = Vector2.new(pos.X + size.X * 0.5, pos.Y + size.Y + 17)
    currentData4.tool.Color = playerEsp.ToolColor
    currentData4.tool.Visible = playerEsp.ToolEnabled
    local from = Vector2.new(currentCamera.ViewportSize.X * 0.5, currentCamera.ViewportSize.Y)
    if playerEsp.TracerPosition == "Center" then
        from = Vector2.new(currentCamera.ViewportSize.X * 0.5, currentCamera.ViewportSize.Y * 0.5)
    elseif playerEsp.TracerPosition == "Mouse" then
        from = mousePosition
    end
    updateDrawingLine(currentData4.tracer, from, Vector2.new(positionValue4.X, positionValue4.Y), playerEsp.TracerColor, playerEsp.TracerEnabled)
    currentData4.head.Position = Vector2.new(positionValue5.X, positionValue5.Y)
    currentData4.head.Radius = math.max(2, width * 0.12)
    currentData4.head.Color = playerEsp.HeadDotColor
    currentData4.head.Visible = playerEsp.HeadDotEnabled
    for currentIndex3, pair in ipairs(skeletonConnections) do
        local currentValues5 = character4:FindFirstChild(pair[1])
        local secondaryValue3 = character4:FindFirstChild(pair[2])
        local show = false
        local localValue10, localValue11
        if currentValues5 and secondaryValue3 then
            local positionValue7, localValue12 = currentCamera:WorldToViewportPoint(currentValues5.Position)
            local positionValue8, localValue13 = currentCamera:WorldToViewportPoint(secondaryValue3.Position)
            show = localValue12 and localValue13 and positionValue7.Z > 0 and positionValue8.Z > 0 and playerEsp.SkeletonEnabled
            localValue10 = Vector2.new(positionValue7.X, positionValue7.Y)
            localValue11 = Vector2.new(positionValue8.X, positionValue8.Y)
        end
        if show then
            updateDrawingLine(currentObject8.skel[currentIndex3], localValue10, localValue11, playerEsp.SkeletonColor, true)
        else
            currentObject8.skel[currentIndex3].Visible = false
        end
    end
    currentObject8.cham.FillColor = playerEsp.ChamsFillColor
    currentObject8.cham.FillTransparency = hasSelectedOption(playerEsp.ChamsType, "Fill") and playerEsp.ChamsFillTrans or 1
    currentObject8.cham.OutlineColor = playerEsp.ChamsOutlineColor
    currentObject8.cham.OutlineTransparency = playerEsp.ChamsOutlineEnabled
        and hasSelectedOption(playerEsp.ChamsType, "Outline") and playerEsp.ChamsOutlineTrans or 1
    currentObject8.cham.Enabled = playerEsp.ChamsEnabled
end
for _, currentPlayer21 in ipairs(playersService:GetPlayers()) do
    createPlayerEsp(currentPlayer21)
end
playersService.PlayerAdded:Connect(createPlayerEsp)
playersService.PlayerRemoving:Connect(function(currentPlayer22)
    removePlayerSelectionHighlight(currentPlayer22, runtimeState.list.whitelistHighlights)
    removePlayerSelectionHighlight(currentPlayer22, runtimeState.list.targetHighlights)
    removePlayerEsp(currentPlayer22)
end)
runService.RenderStepped:Connect(function()
    for currentPlayer23, currentObject9 in pairs(playerEspObjects) do
        pcall(updatePlayerEsp, currentPlayer23, currentObject9)
    end
end)
local worldEspSettings = {
    CashDrop = false,
    Tools = false,
    Safe = false,
    Dealer = false,
    ATM = false,
    Stock = false,
    Key = false,
    Scrap = false,
    Distance = 2000,
}
local tagFolder = Instance.new("Folder")
tagFolder.Name = "JX_WorldESP"
tagFolder.Parent = coreGui
local worldEspTags = {
}
local function createWorldEspTag(obj, localValue14, text, col)
    local old = worldEspTags[localValue14]
    if old and old.obj == obj and old.gui.Parent then
        old.lbl.Text = text
        old.lbl.TextColor3 = col
        return 
    end
    if old then
        pcall(function()
            old.gui:Destroy()
        end)
    end
    local part = obj:IsA("BasePart") and obj or obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart", true)
    if not part then
        return 
    end
    local gui = Instance.new("BillboardGui")
    gui.Name = "JX_" .. localValue14
    gui.AlwaysOnTop = true
    gui.Size = UDim2.fromOffset(180, 30)
    gui.StudsOffset = Vector3.new(0, 2, 0)
    gui.Adornee = part
    gui.Parent = tagFolder
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.fromScale(1, 1)
    lbl.BackgroundTransparency = 1
    lbl.TextStrokeTransparency = 0
    lbl.Font = Enum.Font.GothamBold
    lbl.TextScaled = true
    lbl.Text = text
    lbl.TextColor3 = col
    lbl.Parent = gui
    worldEspTags[localValue14] = {
        obj = obj,
        gui = gui,
        lbl = lbl,
    }
end
local function clearWorldEspTags()
    for localValue15, currentObject10 in pairs(worldEspTags) do
        if not currentObject10.obj.Parent then
            pcall(function()
                currentObject10.gui:Destroy()
            end)
            worldEspTags[localValue15] = nil
        end
    end
end
local function scanWorldEspObjects()
    clearWorldEspTags()
    local myRoot = getRootPart(localPlayer)
    if not myRoot then
        return 
    end
    for _, currentObject11 in ipairs(workspace:GetDescendants()) do
        local currentName2 = currentObject11.Name:lower()
        local parentName = currentObject11.Parent and currentObject11.Parent.Name:lower() or ""
        local grandParentName = currentObject11.Parent and currentObject11.Parent.Parent and currentObject11.Parent.Parent.Name:lower() or ""
        local inSpawnedBread = parentName == "spawnedbread" or grandParentName == "spawnedbread"
        local inSpawnedPiles = parentName == "spawnedpiles" or grandParentName == "spawnedpiles"
        local inDealerFolder = parentName == "bredmakurz" or grandParentName == "bredmakurz"
        local part = currentObject11:IsA("BasePart") and currentObject11 or nil
        if part and (part.Position - myRoot.Position).Magnitude <= worldEspSettings.Distance then
            if worldEspSettings.CashDrop and (currentName2 == "cashdrop1" or inSpawnedBread or currentName2:find("cash") or currentName2:find("money")) then
                createWorldEspTag(currentObject11, "cash_" .. currentObject11:GetDebugId(), "CashDrop", Color3.fromRGB(0, 255, 0))
            elseif worldEspSettings.Key and currentName2:find("key") then
                createWorldEspTag(currentObject11, "key_" .. currentObject11:GetDebugId(), "Key", Color3.fromRGB(255, 255, 0))
            elseif worldEspSettings.Scrap and (inSpawnedPiles or currentName2:find("scrap")) then
                createWorldEspTag(currentObject11, "scrap_" .. currentObject11:GetDebugId(), "Scrap", Color3.fromRGB(180, 180, 180))
            elseif worldEspSettings.ATM and currentName2:find("atm") then
                createWorldEspTag(currentObject11, "atm_" .. currentObject11:GetDebugId(), "ATM", Color3.fromRGB(100, 200, 255))
            end
        end
        if currentObject11:IsA("Tool") and worldEspSettings.Tools then
            createWorldEspTag(currentObject11, "tool_" .. currentObject11:GetDebugId(), currentObject11.Name, Color3.fromRGB(255, 255, 0))
        elseif currentObject11:IsA("Model") then
            if worldEspSettings.CashDrop and (currentName2 == "cashdrop1" or inSpawnedBread) then
                createWorldEspTag(currentObject11, "cash_" .. currentObject11:GetDebugId(), "CashDrop", Color3.fromRGB(0, 255, 0))
            elseif worldEspSettings.Safe and (currentName2 == "small safe" or currentName2:find("safe") or currentName2:find("register")) then
                createWorldEspTag(currentObject11, "safe_" .. currentObject11:GetDebugId(), currentObject11.Name, Color3.fromRGB(255, 100, 100))
            elseif worldEspSettings.Dealer and (inDealerFolder or currentName2 == "armorydealer" or currentName2:find("dealer") or currentName2:find("armory")) then
                local dealerText = currentName2 == "armorydealer" and "Armory Dealer" or currentObject11.Name
                createWorldEspTag(currentObject11, "dealer_" .. currentObject11:GetDebugId(), dealerText, Color3.fromRGB(255, 150, 0))
            elseif worldEspSettings.Stock and (inDealerFolder or currentName2:find("stock")) then
                createWorldEspTag(currentObject11, "stock_" .. currentObject11:GetDebugId(), currentObject11.Name, Color3.fromRGB(170, 100, 255))
            elseif worldEspSettings.Scrap and inSpawnedPiles then
                createWorldEspTag(currentObject11, "scrap_" .. currentObject11:GetDebugId(), currentObject11.Name, Color3.fromRGB(180, 180, 180))
            end
        end
    end
end
startTask("worldEsp", function()
    while task.wait(1) do
        if worldEspSettings.CashDrop or worldEspSettings.Tools or worldEspSettings.Safe or worldEspSettings.Dealer or worldEspSettings.ATM or worldEspSettings.Stock or worldEspSettings.Key or worldEspSettings.Scrap then
            pcall(scanWorldEspObjects)
        end
    end
end)
local armChamsSettings = {
    Enabled = false,
    Color = Color3.fromRGB(255, 0, 0),
}
local originalArmAppearance = {
}
local function updateArmChams()
    local viewModel = currentCamera:FindFirstChild("ViewModel")
    if viewModel then
        for _, armName in ipairs({"Left Arm", "Right Arm"}) do
            local arm = viewModel:FindFirstChild(armName)
            if arm and arm:IsA("BasePart") then
                if armChamsSettings.Enabled then
                    arm.Material = Enum.Material.ForceField
                    arm.Color = armChamsSettings.Color
                else
                    arm.Material = Enum.Material.Plastic
                    arm.Color = Color3.fromRGB(215, 132, 111)
                end
            end
        end
    end
    local character5 = getCharacter(localPlayer)
    if not character5 then
        return 
    end
    for _, currentObject12 in ipairs(character5:GetDescendants()) do
        if currentObject12:IsA("BasePart") and (currentObject12.Name:find("Arm") or currentObject12.Name:find("Hand") or currentObject12.Parent:IsA("Tool")) then
            if armChamsSettings.Enabled then
                if not originalArmAppearance[currentObject12] then
                    originalArmAppearance[currentObject12] = {
                        currentObject12.Color,
                        currentObject12.Material,
                    }
                end
                currentObject12.Color = armChamsSettings.Color
                currentObject12.Material = Enum.Material.Neon
            elseif originalArmAppearance[currentObject12] then
                currentObject12.Color = originalArmAppearance[currentObject12][1]
                currentObject12.Material = originalArmAppearance[currentObject12][2]
                originalArmAppearance[currentObject12] = nil
            end
        end
    end
end
runService.RenderStepped:Connect(updateArmChams)
local skyPresets = {
    Nebula = {
        MoonTextureId = "rbxassetid://1075087760",
        SkyboxBk = "rbxassetid://2118763079",
        SkyboxDn = "rbxassetid://2118766919",
        SkyboxFt = "rbxassetid://2118765204",
        SkyboxLf = "rbxassetid://2118764070",
        SkyboxRt = "rbxassetid://2118761853",
        SkyboxUp = "rbxassetid://2118766003",
        StarCount = 0,
    },
    ["Red Nebula"] = {
        MoonTextureId = "rbxassetid://1075087760",
        SkyboxBk = "rbxassetid://75202130006087",
        SkyboxDn = "rbxassetid://84899615600068",
        SkyboxFt = "rbxassetid://123583852168685",
        SkyboxLf = "rbxassetid://91852061002963",
        SkyboxRt = "rbxassetid://138329424663418",
        SkyboxUp = "rbxassetid://98269626597694",
        StarCount = 0,
    },
    ["Nebula Pink"] = {
        MoonTextureId = "rbxasset://sky/moon.jpg",
        SkyboxBk = "rbxassetid://13581437029",
        SkyboxDn = "rbxassetid://13581439832",
        SkyboxFt = "rbxassetid://13581447312",
        SkyboxLf = "rbxassetid://13581443463",
        SkyboxRt = "rbxassetid://13581452875",
        SkyboxUp = "rbxassetid://13581450222",
        StarCount = 3000,
    },
    ["White Galaxy"] = {
        MoonTextureId = "rbxasset://sky/moon.jpg",
        SkyboxBk = "rbxassetid://5540798456",
        SkyboxDn = "rbxassetid://5540799894",
        SkyboxFt = "rbxassetid://5540801779",
        SkyboxLf = "rbxassetid://5540801192",
        SkyboxRt = "rbxassetid://5540799108",
        SkyboxUp = "rbxassetid://5540800635",
        StarCount = 5000,
        SunAngularSize = 1,
        SunTextureId = "rbxasset://sky/sun.jpg",
    },
    ["Purple Nebula"] = {
        MoonTextureId = "rbxasset://sky/moon.jpg",
        SkyboxBk = "rbxassetid://94797807540176",
        SkyboxDn = "rbxassetid://135040133024386",
        SkyboxFt = "rbxassetid://134956217810021",
        SkyboxLf = "rbxassetid://77274943792368",
        SkyboxRt = "rbxassetid://86193107896056",
        SkyboxUp = "rbxassetid://72286287669628",
        StarCount = 3000,
        SunAngularSize = 11,
        SunTextureId = "rbxasset://sky/sun.jpg",
    },
}
local visualSettings = {
    SkyOn = false,
    Sky = "Nebula",
    FogOn = false,
    FogColor = Color3.fromRGB(11, 14, 199),
    FogDensity = 0.439,
    BlurOn = false,
    Blur = 0.5,
    FovOn = false,
    Fov = 70,
    RecoilOn = false,
    Recoil = 0,
    FullBright = false,
    MotionBlur = false,
    Stretch = false,
    StretchPosition = 0.5,
    CameraDistance = math.floor(localPlayer.CameraMaxZoomDistance),
    HideHead = false,
    HideBody = false,
}
local customSky
local blurEffect
local atmosphereEffect
local function setCustomSky(localValue16)
    if customSky then
        customSky:Destroy()
        customSky = nil
    end
    if not localValue16 then
        return 
    end
    local cfg = skyPresets[visualSettings.Sky]
    if not cfg then
        return 
    end
    customSky = Instance.new("Sky")
    for currentKey2, currentValue3 in pairs(cfg) do
        customSky[currentKey2] = currentValue3
    end
    customSky.Parent = lighting
end
local function setCustomFog(localValue17)
    if atmosphereEffect then
        atmosphereEffect:Destroy()
        atmosphereEffect = nil
    end
    if not localValue17 then
        return 
    end
    atmosphereEffect = Instance.new("Atmosphere")
    atmosphereEffect.Density = visualSettings.FogDensity
    atmosphereEffect.Offset = 1
    atmosphereEffect.Color = visualSettings.FogColor
    atmosphereEffect.Decay = Color3.fromRGB(7, 4, 92)
    atmosphereEffect.Glare = 0
    atmosphereEffect.Haze = 10
    atmosphereEffect.Parent = lighting
end
runtimeState.data.setFullBright = function(enabled)
    visualSettings.FullBright = enabled
    disconnectConnection("fullBrightClock")
    disconnectConnection("fullBrightBrightness")
    if enabled then
        if not runtimeState.data.fullBrightOriginal then
            runtimeState.data.fullBrightOriginal = {
                ClockTime = lighting.ClockTime,
                Brightness = lighting.Brightness,
                ExposureCompensation = lighting.ExposureCompensation,
                FogEnd = lighting.FogEnd,
                GlobalShadows = lighting.GlobalShadows,
                OutdoorAmbient = lighting.OutdoorAmbient,
            }
        end
        local indexFolder = runtimeState.data.fullBrightIndex
        if not indexFolder then
            indexFolder = Instance.new("Folder")
            indexFolder.Name = "JX_FBIndex"
            indexFolder.Parent = coreGui
            runtimeState.data.fullBrightIndex = indexFolder
            for _, child in ipairs(lighting:GetChildren()) do
                if child ~= customSky and child ~= atmosphereEffect and child ~= blurEffect then
                    child.Parent = indexFolder
                end
            end
        end
        lighting.ClockTime = 14
        lighting.Brightness = 4
        lighting.ExposureCompensation = 0.7
        lighting.FogEnd = 100000
        lighting.GlobalShadows = false
        lighting.OutdoorAmbient = Color3.new(1, 1, 1)
        connectSignal("fullBrightClock", lighting:GetPropertyChangedSignal("ClockTime"), function()
            if visualSettings.FullBright and lighting.ClockTime ~= 14 then
                lighting.ClockTime = 14
            end
        end)
        connectSignal("fullBrightBrightness", lighting:GetPropertyChangedSignal("Brightness"), function()
            if visualSettings.FullBright and lighting.Brightness ~= 4 then
                lighting.Brightness = 4
            end
        end)
    else
        local indexFolder = runtimeState.data.fullBrightIndex
        if indexFolder then
            for _, child in ipairs(indexFolder:GetChildren()) do
                child.Parent = lighting
            end
            pcall(indexFolder.Destroy, indexFolder)
            runtimeState.data.fullBrightIndex = nil
        end
        local original = runtimeState.data.fullBrightOriginal
        if original then
            lighting.ClockTime = original.ClockTime
            lighting.Brightness = original.Brightness
            lighting.ExposureCompensation = original.ExposureCompensation
            lighting.FogEnd = original.FogEnd
            lighting.GlobalShadows = original.GlobalShadows
            lighting.OutdoorAmbient = original.OutdoorAmbient
            runtimeState.data.fullBrightOriginal = nil
        end
    end
end
local function setCustomBlur(localValue18)
    if blurEffect then
        blurEffect:Destroy()
        blurEffect = nil
    end
    if not localValue18 then
        return 
    end
    blurEffect = Instance.new("BlurEffect")
    blurEffect.Size = visualSettings.Blur * 56
    blurEffect.Parent = lighting
end
runService.RenderStepped:Connect(function()
    if visualSettings.FovOn then
        currentCamera.FieldOfView = visualSettings.Fov
    end
    if visualSettings.FullBright then
        lighting.Brightness = 4
        lighting.ClockTime = 14
        lighting.FogEnd = 100000
        lighting.GlobalShadows = false
        lighting.OutdoorAmbient = Color3.new(1, 1, 1)
    end
    if blurEffect then
        blurEffect.Size = visualSettings.Blur * 56
    end
    if atmosphereEffect then
        atmosphereEffect.Density = visualSettings.FogDensity
        atmosphereEffect.Color = visualSettings.FogColor
    end
end)
local movementSettings = {
    Fly = false,
    FlySpeed = 5,
    FlyMethod = "Bypass",
    Speed = false,
    SpeedValue = 70,
    Jump = false,
    JumpValue = 200,
    Noclip = false,
    QTp = false,
    ControlClickTeleport = false,
    ControlHeld = false,
}
local flyVelocity
local flyGyroscope
local function stopFly()
    movementSettings.Fly = false
    disconnectConnection("fly")
    if flyVelocity then
        flyVelocity:Destroy()
        flyVelocity = nil
    end
    if flyGyroscope then
        flyGyroscope:Destroy()
        flyGyroscope = nil
    end
end
local function startFly()
    stopFly()
    movementSettings.Fly = true
    local rootPart5 = getRootPart(localPlayer)
    if not rootPart5 then
        return 
    end
    flyVelocity = Instance.new("BodyVelocity")
    flyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    flyVelocity.Velocity = Vector3.zero
    flyVelocity.Parent = rootPart5
    flyGyroscope = Instance.new("BodyGyro")
    flyGyroscope.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
    flyGyroscope.P = 9000
    flyGyroscope.CFrame = currentCamera.CFrame
    flyGyroscope.Parent = rootPart5
    connectSignal("fly", runService.RenderStepped, function()
        local dir = Vector3.zero
        if userInputService:IsKeyDown(Enum.KeyCode.W) then
            dir = dir + currentCamera.CFrame.LookVector
        end
        if userInputService:IsKeyDown(Enum.KeyCode.S) then
            dir = dir - currentCamera.CFrame.LookVector
        end
        if userInputService:IsKeyDown(Enum.KeyCode.A) then
            dir = dir - currentCamera.CFrame.RightVector
        end
        if userInputService:IsKeyDown(Enum.KeyCode.D) then
            dir = dir + currentCamera.CFrame.RightVector
        end
        if userInputService:IsKeyDown(Enum.KeyCode.Space) then
            dir = dir + currentCamera.CFrame.UpVector
        end
        if userInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
            dir = dir - currentCamera.CFrame.UpVector
        end
        flyVelocity.Velocity = dir.Magnitude > 0 and dir.Unit * movementSettings.FlySpeed or Vector3.zero
        flyGyroscope.CFrame = currentCamera.CFrame
        if movementSettings.FlyMethod == "Bypass" and remoteEvents then
            local remoteObject3 = remoteEvents:FindFirstChild("__---r")
            if remoteObject3 then
                pcall(function()
                    remoteObject3:FireServer("__---r", Vector3.zero, rootPart5.CFrame, false)
                end)
            end
        end
    end)
end
runService.Heartbeat:Connect(function()
    local character6 = getCharacter(localPlayer)
    local humanoid8 = getHumanoid(localPlayer)
    local rootPart6 = getRootPart(localPlayer)
    if movementSettings.Noclip and character6 then
        for _, currentObject13 in ipairs(character6:GetDescendants()) do
            if currentObject13:IsA("BasePart") then
                currentObject13.CanCollide = false
            end
        end
    end
    if movementSettings.Speed and rootPart6 then
        local dir = humanoid8 and humanoid8.MoveDirection or Vector3.zero
        rootPart6.AssemblyLinearVelocity = Vector3.new(dir.X * movementSettings.SpeedValue, rootPart6.AssemblyLinearVelocity.Y, dir.Z * movementSettings.SpeedValue)
    end
    if movementSettings.Jump and humanoid8 then
        humanoid8.JumpPower = movementSettings.JumpValue
        humanoid8.UseJumpPower = true
    end
end)
userInputService.InputBegan:Connect(function(currentObject14, currentGroup)
    if currentGroup then
        return 
    end
    if movementSettings.QTp and currentObject14.KeyCode == Enum.KeyCode.Q then
        local rootPart7 = getRootPart(localPlayer)
        if rootPart7 then
            local pos = userInputService:GetMouseLocation()
            local ray = currentCamera:ViewportPointToRay(pos.X, pos.Y)
            local hit = workspace:Raycast(ray.Origin, ray.Direction * 5000)
            if hit then
                rootPart7.CFrame = CFrame.new(hit.Position + Vector3.new(0, 3, 0))
            end
        end
    end
end)
userInputService.InputBegan:Connect(function(inputObject, gameProcessed)
    if gameProcessed then
        return 
    end
    if inputObject.KeyCode == Enum.KeyCode.LeftControl or inputObject.KeyCode == Enum.KeyCode.RightControl then
        movementSettings.ControlHeld = true
    end
end)
userInputService.InputEnded:Connect(function(inputObject)
    if inputObject.KeyCode == Enum.KeyCode.LeftControl or inputObject.KeyCode == Enum.KeyCode.RightControl then
        movementSettings.ControlHeld = false
    end
end)
localMouse.Button1Down:Connect(function()
    if not movementSettings.ControlClickTeleport or not movementSettings.ControlHeld then
        return 
    end
    local rootPart = getRootPart(localPlayer)
    if rootPart and localMouse.Hit then
        rootPart.CFrame = CFrame.new(localMouse.Hit.Position + Vector3.new(0, 5, 0))
    end
end)
local utilitySettings = {
    InfStamina = false,
    AutoRespawn = false,
    NoFail = false,
    InfPepper = false,
    AntiAfk = false,
    AutoUnlock = false,
    AutoCash = false,
    AutoSafe = false,
    AutoOpen = false,
    AutoClose = false,
    NoFall = false,
    NoNeck = false,
    NoBarriers = false,
    AntiFracture = false,
    AntiSmoke = false,
    InstantEquip = false,
    FastPickup = false,
    AutoRepair = false,
    HighlightExit = false,
    AutoPickUpTool = false,
    AutoRefill = false,
    AutoDeposit = false,
    AutoClaimAllowance = false,
    AutoFarmAllowance = false,
    InstantReload = false,
    RemoveFakeGlass = false,
    ChatEnabled = false,
    WallBang = false,
}
local spraySettings = {
    Enabled = false,
    CheckWhitelist = false,
    CheckTarget = false,
    Range = 1000,
    FinishSpeedMultiplier = 0.2,
}
local scrapEspSettings = {
    Enabled = false,
    MaxDistance = 1000,
    MinimumRarity = "Common",
    Types = {
        Text = true,
        Highlight = false,
        Tracer = false,
    },
    Objects = {
    },
}
local keyEspSettings = {
    Enabled = false,
    Objects = {
    },
}
local dealerItemNames = {
    "AKS-74U",
    "M4A1",
    "MAC-10",
    "UMP-45",
    "Uzi",
    "Tommy Gun",
    "Scout",
    "BFG-50",
    "Super Shorty",
    "RPG",
    "Grenade Launcher",
    "C4",
    "Pepper Spray",
    "Lockpick",
    "Shovel",
    "Bat",
    "Fireaxe",
    "Chainsaw",
    "Golf Club",
    "Sledgehammer",
    "Balisong",
    "Fleshgrinder",
    "Messer",
}
local stockCheckerSettings = {
    Enabled = false,
    SelectedItems = {
    },
    NotifyNewStock = true,
    DealerEsp = false,
    EspTypes = {
        Text = true,
        Highlight = false,
        Tracer = false,
    },
    LastSnapshot = {
    },
    EspObjects = {
    },
}
local allowanceFarmPositions = {
    Vector3.new(-5000.67, 1.95, -373.1),
    Vector3.new(-4790.3, 1.95, -383.11),
    Vector3.new(-4789.91, 1.96, -372.03),
    Vector3.new(-4627.25, 1.67, -980.48),
}
local rarityOrder = {
    Common = 1,
    Uncommon = 2,
    Rare = 3,
    Epic = 4,
    Legendary = 5,
    Mythic = 6,
}
local staminaStateTables = {
}
if getgc then
    pcall(function()
        for _, currentObject15 in ipairs(getgc(true)) do
            if type(currentObject15) == "table" and rawget(currentObject15, "S") then
                staminaStateTables[#staminaStateTables + 1] = currentObject15
            end
        end
    end)
end
runService.Heartbeat:Connect(function()
    if utilitySettings.InfStamina then
        for _, currentObject16 in ipairs(staminaStateTables) do
            currentObject16.S = 100
        end
    end
    local character7 = getCharacter(localPlayer)
    if character7 then
        character7:SetAttribute("NoNeckMovement", utilitySettings.NoNeck)
        if utilitySettings.AntiFracture then
            local stats = replicatedStorage:FindFirstChild("CharStats")
            stats = stats and stats:FindFirstChild(localPlayer.Name)
            local localValue19 = stats and stats:FindFirstChild("HealthValues")
            if localValue19 then
                for _, currentName3 in ipairs({
                    "Left Arm",
                    "Right Arm",
                    "Left Leg",
                    "Right Leg",
                }) do
                    local currentValue4 = localValue19:FindFirstChild(currentName3)
                    local secondaryValue4 = currentValue4 and currentValue4:FindFirstChild("Broken")
                    if secondaryValue4 then
                        secondaryValue4.Value = false
                    end
                end
            end
        end
        if utilitySettings.InfPepper then
            local tool = character7:FindFirstChildOfClass("Tool")
            local vals = tool and tool:FindFirstChild("Values")
            for _, currentName4 in ipairs({
                "SERVER_Ammo",
                "Ammo",
                "Sprays",
            }) do
                local currentValue5 = vals and vals:FindFirstChild(currentName4)
                if currentValue5 and (currentValue5:IsA("NumberValue") or currentValue5:IsA("IntValue")) then
                    currentValue5.Value = math.max(currentValue5.Value, 100)
                end
            end
        end
    end
end)
localPlayer.Idled:Connect(function()
    if utilitySettings.AntiAfk then
        virtualUser:CaptureController()
        virtualUser:ClickButton2(Vector2.zero)
    end
end)
runtimeState.data.setNoNeck = function(enabled, character)
    utilitySettings.NoNeck = enabled
    local oldTrack = runtimeState.data.noNeckTrack
    if oldTrack then
        pcall(oldTrack.Stop, oldTrack)
        runtimeState.data.noNeckTrack = nil
    end
    character = character or getCharacter(localPlayer)
    if character then
        character:SetAttribute("NoNeckMovement", enabled)
    end
    if not enabled or not character then
        return
    end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then
        return
    end
    local animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)
    local animation = Instance.new("Animation")
    animation.AnimationId = "rbxassetid://68339848"
    local ok, track = pcall(animator.LoadAnimation, animator, animation)
    pcall(animation.Destroy, animation)
    if ok and track then
        track.Looped = true
        track:Play()
        runtimeState.data.noNeckTrack = track
    end
end
localPlayer.CharacterAdded:Connect(function(currentCharacter7)
    if utilitySettings.NoNeck then
        task.wait(0.5)
        runtimeState.data.setNoNeck(true, currentCharacter7)
    end
    if utilitySettings.AutoRespawn then
        task.wait(0.2)
        local remoteObject4 = remoteEvents and remoteEvents:FindFirstChild("DeathRespawn")
        if remoteObject4 then
            pcall(function()
                remoteObject4:FireServer()
            end)
        end
    end
end)
local function activateNearbyPrompts(mode)
    local rootPart8 = getRootPart(localPlayer)
    if not rootPart8 then
        return 
    end
    for _, currentObject17 in ipairs(workspace:GetDescendants()) do
        if currentObject17:IsA("ProximityPrompt") and currentObject17.Enabled then
            local currentPlayer24 = currentObject17.Parent
            local bodyPosition = currentPlayer24 and (currentPlayer24:IsA("BasePart") and currentPlayer24 or currentPlayer24:FindFirstChildWhichIsA("BasePart"))
            local currentName5 = (currentPlayer24 and currentPlayer24.Name or ""):lower()
            if bodyPosition and (bodyPosition.Position - rootPart8.Position).Magnitude <= currentObject17.MaxActivationDistance + 4 then
                local ok = mode == "all"
                ok = ok or mode == "cash" and (currentName5:find("cash") or currentName5:find("money"))
                ok = ok or mode == "door" and currentName5:find("door")
                ok = ok or mode == "safe" and (currentName5:find("safe") or currentName5:find("register"))
                ok = ok or mode == "repair" and currentName5:find("repair")
                if ok and utilitySettings.FastPickup then
                    pcall(function()
                        currentObject17.HoldDuration = 0
                        currentObject17:SetAttribute("JXFastHooked", true)
                    end)
                end
                if ok and fireproximityprompt then
                    pcall(fireproximityprompt, currentObject17)
                end
            end
        end
    end
end
startTask("util", function()
    while task.wait(0.2) do
        if utilitySettings.AutoCash or utilitySettings.FastPickup then
            activateNearbyPrompts("cash")
        end
        if utilitySettings.AutoSafe then
            activateNearbyPrompts("safe")
        end
        if utilitySettings.AutoOpen or utilitySettings.AutoClose or utilitySettings.AutoUnlock then
            activateNearbyPrompts("door")
        end
        if utilitySettings.AutoRepair then
            activateNearbyPrompts("repair")
        end
        if utilitySettings.NoBarriers then
            local currentCallback = workspace:FindFirstChild("Filter")
            currentCallback = currentCallback and currentCallback:FindFirstChild("Parts")
            if currentCallback then
                for _, currentObject18 in ipairs(currentCallback:GetDescendants()) do
                    if currentObject18:IsA("BasePart") then
                        currentObject18.CanTouch = false
                        currentObject18.CanCollide = false
                    end
                end
            end
        end
    end
end)
local function addButton(section, name, callback)
    return section:Button({
        Name = name,
        Callback = callback,
    })
end
local function addColorPicker(control, flag, defaultValue, callback)
    if control and control.SubColorpicker then
        return control:SubColorpicker({
            Flag = flag,
            Default = defaultValue,
            Callback = callback,
        })
    end
end
local function addKeybind(control, flag, defaultValue, callback)
    if control and control.SubKeybind then
        return control:SubKeybind({
            Flag = flag,
            Default = defaultValue,
            Callback = callback,
        })
    end
end
local spectateSettings = {
    On = false,
    Name = nil,
}
local function stopSpectating()
    spectateSettings.On = false
    local humanoid9 = getHumanoid(localPlayer)
    if humanoid9 then
        currentCamera.CameraSubject = humanoid9
    end
end
local function startSpectating()
    startTask("spec", function()
        while spectateSettings.On do
            local player1 = spectateSettings.Name and playersService:FindFirstChild(spectateSettings.Name)
            local humanoid10 = getHumanoid(player1)
            if humanoid10 then
                currentCamera.CameraSubject = humanoid10
            end
            task.wait(0.1)
        end
        local humanoid11 = getHumanoid(localPlayer)
        if humanoid11 then
            currentCamera.CameraSubject = humanoid11
        end
    end)
end
playersService.PlayerAdded:Connect(function(currentPlayer25)
    currentPlayer25.CharacterAdded:Connect(function()
        if runtimeState.list.whitelistHighlightsEnabled and runtimeState.list.whitelist[currentPlayer25.Name] then
            createPlayerSelectionHighlight(currentPlayer25, runtimeState.list.whitelistHighlightColor, runtimeState.list.whitelistHighlights)
        end
        if runtimeState.list.targetHighlightsEnabled and runtimeState.list.targets[currentPlayer25.Name] then
            createPlayerSelectionHighlight(currentPlayer25, runtimeState.list.targetHighlightColor, runtimeState.list.targetHighlights)
        end
    end)
end)
playersService.PlayerRemoving:Connect(function(currentPlayer26)
    removePlayerSelectionHighlight(currentPlayer26, runtimeState.list.whitelistHighlights)
    removePlayerSelectionHighlight(currentPlayer26, runtimeState.list.targetHighlights)
    removePlayerEsp(currentPlayer26)
end)
local funSettings = {
    C4 = false,
    C4Speed = 0.1,
    Boom = false,
    BoomSpeed = 0.1,
    Pepper = false,
    PepperRange = 15,
    Stomp = false,
    StompSpeed = 0.1,
    Increase = false,
}
local function findTool(...)
    local character8 = getCharacter(localPlayer)
    local tool1 = localPlayer:FindFirstChildOfClass("Backpack")
    for _, currentName6 in ipairs({
        ...,
    }) do
        local temporaryValue = character8 and character8:FindFirstChild(currentName6)
        temporaryValue = temporaryValue or tool1 and tool1:FindFirstChild(currentName6)
        if temporaryValue then
            return temporaryValue
        end
    end
end
local function fireFirstRemote(getOtherPlayerNames, ...)
    for _, currentName7 in ipairs(getOtherPlayerNames) do
        local currentObject19 = replicatedStorage:FindFirstChild(currentName7, true)
        if currentObject19 and currentObject19:IsA("RemoteEvent") then
            local ok = pcall(currentObject19.FireServer, currentObject19, ...)
            if ok then
                return true
            end
        elseif currentObject19 and currentObject19:IsA("RemoteFunction") then
            local ok = pcall(currentObject19.InvokeServer, currentObject19, ...)
            if ok then
                return true
            end
        end
    end
    return false
end
local function activateExplosiveTool(kind)
    local rootPart9 = getRootPart(localPlayer)
    if not rootPart9 then
        return 
    end
    local temporaryValue2
    if kind == "c4" then
        temporaryValue2 = findTool("C4", "C-4", "Remote C4")
    else
        temporaryValue2 = findTool("RPG", "Rocket Launcher", "Grenade Launcher", "M320", "M79")
    end
    if temporaryValue2 and temporaryValue2.Parent ~= getCharacter(localPlayer) then
        local humanoid12 = getHumanoid(localPlayer)
        if humanoid12 then
            humanoid12:EquipTool(temporaryValue2)
        end
    end
    if temporaryValue2 then
        pcall(temporaryValue2.Activate, temporaryValue2)
    end
    fireFirstRemote({
        "C4",
        "C4Remote",
        "Explode",
        "Explosion",
        "Rocket",
        "FireRocket",
    }, temporaryValue2, rootPart9.CFrame, rootPart9.Position, currentCamera.CFrame.LookVector)
end
local function runPepperAura()
    if not funSettings.Pepper then
        return 
    end
    local rootPart10 = getRootPart(localPlayer)
    local tool2 = findTool("Pepper Spray", "PepperSpray", "Pepper")
    if not rootPart10 or not tool2 then
        return 
    end
    for _, currentPlayer27 in ipairs(playersService:GetPlayers()) do
        local rootPart11 = getRootPart(currentPlayer27)
        if currentPlayer27 ~= localPlayer and rootPart11 and not shouldIgnorePlayer(currentPlayer27, {
            CheckDown = false,
            CheckTeam = false,
            CheckWhitelist = spraySettings.CheckWhitelist,
            CheckTarget = spraySettings.CheckTarget,
            CheckForceShield = false,
        }) and (rootPart11.Position - rootPart10.Position).Magnitude <= funSettings.PepperRange then
            fireFirstRemote({
                "Peppershot",
                "PepperShot",
                "Pepper",
            }, tool2, rootPart11.Position, rootPart11)
        end
    end
end
startTask("fun", function()
    while task.wait(0.05) do
        if funSettings.C4 then
            activateExplosiveTool("c4")
            task.wait(funSettings.C4Speed)
        end
        if funSettings.Boom then
            activateExplosiveTool("boom")
            task.wait(funSettings.BoomSpeed)
        end
        if funSettings.Pepper then
            runPepperAura()
        end
        if funSettings.Stomp then
            local rootPart12 = getRootPart(localPlayer)
            if rootPart12 then
                for _, currentPlayer28 in ipairs(playersService:GetPlayers()) do
                    local rootPart13 = getRootPart(currentPlayer28)
                    if currentPlayer28 ~= localPlayer and rootPart13 and isPlayerDowned(currentPlayer28) and (rootPart13.Position - rootPart12.Position).Magnitude <= 12 then
                        fireFirstRemote({
                            "Stomp",
                            "StompEvent",
                            "Finish",
                        }, currentPlayer28.Character, rootPart13)
                    end
                end
            end
            task.wait(funSettings.StompSpeed)
        end
    end
end)
local skinChanger = {
    On = false,
    Name = "None",
    Map = {
        Acacia = "rbxassetid://16688144837",
        Alchemist = "rbxassetid://88337986924078",
        Arctx = "rbxassetid://15695443241",
        Dragon = "rbxassetid://17519365000",
        Gold = "rbxassetid://15012855048",
        ["Hallows Blade"] = "rbxassetid://15177260870",
        Modest = "rbxassetid://15445243396",
        ["Neo-blade"] = "rbxassetid://15653919187",
        Saphira = "rbxassetid://14983754881",
        ["Yule Tide"] = "rbxassetid://78387945331940",
    },
    Old = {
    },
}
local function applySkinToObject(currentObject20, localValue20)
    if currentObject20:IsA("MeshPart") then
        skinChanger.Old[currentObject20] = skinChanger.Old[currentObject20] or currentObject20.TextureID
        currentObject20.TextureID = localValue20
    elseif currentObject20:IsA("SpecialMesh") then
        skinChanger.Old[currentObject20] = skinChanger.Old[currentObject20] or currentObject20.TextureId
        currentObject20.TextureId = localValue20
    elseif currentObject20:IsA("Texture") or currentObject20:IsA("Decal") then
        skinChanger.Old[currentObject20] = skinChanger.Old[currentObject20] or currentObject20.Texture
        currentObject20.Texture = localValue20
    end
end
local function applySelectedSkin(currentCharacter8)
    if not skinChanger.On then
        return 
    end
    local localValue21 = skinChanger.Map[skinChanger.Name]
    if not localValue21 or not currentCharacter8 then
        return 
    end
    for _, currentObject21 in ipairs(currentCharacter8:GetDescendants()) do
        pcall(applySkinToObject, currentObject21, localValue21)
    end
end
local function restoreOriginalSkins()
    for currentObject22, currentValue6 in pairs(skinChanger.Old) do
        pcall(function()
            if currentObject22:IsA("MeshPart") then
                currentObject22.TextureID = currentValue6
            elseif currentObject22:IsA("SpecialMesh") then
                currentObject22.TextureId = currentValue6
            elseif currentObject22:IsA("Texture") or currentObject22:IsA("Decal") then
                currentObject22.Texture = currentValue6
            end
        end)
    end
    table.clear(skinChanger.Old)
end
local function watchCharacterSkins(currentCharacter9)
    applySelectedSkin(currentCharacter9)
    if currentCharacter9 then
        currentCharacter9.ChildAdded:Connect(function(currentObject23)
            task.wait(0.2)
            applySelectedSkin(currentObject23)
        end)
    end
end
localPlayer.CharacterAdded:Connect(function(currentCharacter10)
    task.wait(0.6)
    watchCharacterSkins(currentCharacter10)
end)
local localBackpack = localPlayer:FindFirstChildOfClass("Backpack")
if localBackpack then
    localBackpack.ChildAdded:Connect(function(currentObject24)
        task.wait(0.2)
        applySelectedSkin(currentObject24)
    end)
end
local createdToyTools = {
}
runtimeState.data.toyAnimationStates = runtimeState.data.toyAnimationStates or {}
local function removeToyTool(currentName8)
    local state = runtimeState.data.toyAnimationStates[currentName8]
    if state then
        state.active = false
        for _, track in ipairs(state.tracks or {}) do
            pcall(track.Stop, track)
        end
        runtimeState.data.toyAnimationStates[currentName8] = nil
    end
    local temporaryValue3 = createdToyTools[currentName8]
    if temporaryValue3 then
        pcall(temporaryValue3.Destroy, temporaryValue3)
        createdToyTools[currentName8] = nil
    end
end
local function createToyTool(currentName9)
    removeToyTool(currentName9)
    local instanceObject5 = Instance.new("Tool")
    instanceObject5.Name = currentName9
    instanceObject5.RequiresHandle = false
    instanceObject5.CanBeDropped = false
    local state = {active = false, tracks = {}}
    runtimeState.data.toyAnimationStates[currentName9] = state
    local function stopAnimations()
        state.active = false
        for _, track in ipairs(state.tracks) do
            pcall(track.Stop, track)
        end
        table.clear(state.tracks)
    end
    local function playAnimation(animationId)
        local humanoid = getHumanoid(localPlayer)
        if not humanoid then
            return nil
        end
        local animation = Instance.new("Animation")
        animation.AnimationId = animationId
        local ok, track = pcall(humanoid.LoadAnimation, humanoid, animation)
        pcall(animation.Destroy, animation)
        if ok and track then
            track.Looped = true
            track:Play()
            state.tracks[#state.tracks + 1] = track
            return track
        end
    end
    instanceObject5.Activated:Connect(function()
        if currentName9 == "Fake-Downed" then
            state.active = not state.active
            pcall(function()
                local charStats = replicatedStorage:FindFirstChild("CharStats")
                local playerStats = charStats and charStats:FindFirstChild(localPlayer.Name)
                local downed = playerStats and playerStats:FindFirstChild("Downed")
                if downed then
                    downed.Value = state.active
                end
            end)
            return
        end
        if state.active then
            stopAnimations()
            return
        end
        state.active = true
        if currentName9 == "Carpet" then
            playAnimation("rbxassetid://282574440")
        elseif currentName9 == "Jerk" then
            local humanoid = getHumanoid(localPlayer)
            local animationId = humanoid and humanoid.RigType == Enum.HumanoidRigType.R15
                and "rbxassetid://698251653" or "rbxassetid://72042024"
            playAnimation(animationId)
        elseif currentName9 == "Hug" then
            playAnimation("rbxassetid://283545583")
            playAnimation("rbxassetid://225975820")
        end
    end)
    local function bindDeath(character)
        local humanoid = character and character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.Died:Connect(stopAnimations)
        end
    end
    bindDeath(getCharacter(localPlayer))
    instanceObject5.AncestryChanged:Connect(function(_, parent)
        if not parent then
            stopAnimations()
        end
    end)
    instanceObject5.Parent = localPlayer:FindFirstChildOfClass("Backpack") or localPlayer
    createdToyTools[currentName9] = instanceObject5
    return instanceObject5
end
local toyToolSettings = {
    Hug = false,
    Carpet = false,
    Jerk = false,
    Down = false,
}
local dealerStockSettings = {
    On = false,
    Last = "",
}
local function readDealerStock()
    local out = {
    }
    for _, currentObject25 in ipairs(workspace:GetDescendants()) do
        local currentName10 = currentObject25.Name:lower()
        if currentName10:find("dealer") or currentName10:find("stock") or currentName10:find("armory") then
            for _, currentValue7 in ipairs(currentObject25:GetDescendants()) do
                if currentValue7:IsA("IntValue") or currentValue7:IsA("NumberValue") or currentValue7:IsA("StringValue") or currentValue7:IsA("BoolValue") then
                    out[#out + 1] = currentValue7.Name .. ": " .. tostring(currentValue7.Value)
                end
            end
        end
    end
    if #out == 0 then
        out[1] = "No stock values found"
    end
    return table.concat(out, " | ")
end
startTask("stock", function()
    while task.wait(3) do
        if dealerStockSettings.On then
            local currentSettings = readDealerStock()
            if currentSettings ~= dealerStockSettings.Last then
                dealerStockSettings.Last = currentSettings
                interfaceLibrary:Notification({
                    Title = "Dealer Stock",
                    Description = currentSettings,
                    Duration = 5,
                    Icon = "85279746515974",
                })
            end
        end
    end
end)
local administratorUserIds = {
    [3294804378] = true,
    [93676120] = true,
    [54087314] = true,
    [81275825] = true,
    [140837601] = true,
    [1229486091] = true,
    [46567801] = true,
    [418086275] = true,
    [29706395] = true,
    [3717066084] = true,
    [1424338327] = true,
    [5046662686] = true,
    [5046661126] = true,
    [5046659439] = true,
    [418199326] = true,
    [1024216621] = true,
    [1810535041] = true,
    [63238912] = true,
    [111250044] = true,
    [63315426] = true,
    [730176906] = true,
    [141193516] = true,
    [194512073] = true,
    [193945439] = true,
    [412741116] = true,
    [195538733] = true,
    [102045519] = true,
    [955294] = true,
    [957835150] = true,
    [25689921] = true,
    [366613818] = true,
    [281593651] = true,
    [455275714] = true,
    [208929505] = true,
    [96783330] = true,
    [156152502] = true,
    [93281166] = true,
    [959606619] = true,
    [142821118] = true,
    [632886139] = true,
    [175931803] = true,
    [122209625] = true,
    [278097946] = true,
    [142989311] = true,
    [1517131734] = true,
    [446849296] = true,
    [87189764] = true,
    [67180844] = true,
    [9212846] = true,
    [47352513] = true,
    [48058122] = true,
    [155413858] = true,
    [10497435] = true,
    [513615792] = true,
    [55893752] = true,
    [55476024] = true,
    [151691292] = true,
    [136584758] = true,
    [16983447] = true,
    [3111449] = true,
    [94693025] = true,
    [271400893] = true,
    [5005262660] = true,
    [295331237] = true,
    [64489098] = true,
    [244844600] = true,
    [114332275] = true,
    [25048901] = true,
    [69262878] = true,
    [50801509] = true,
    [92504899] = true,
    [42066711] = true,
    [50585425] = true,
    [31365111] = true,
    [166406495] = true,
    [2457253857] = true,
    [29761878] = true,
    [21831137] = true,
    [948293345] = true,
    [439942262] = true,
    [38578487] = true,
    [1163048] = true,
    [7713309208] = true,
    [3659305297] = true,
    [15598614] = true,
    [34616594] = true,
    [626833004] = true,
    [198610386] = true,
    [153835477] = true,
    [3923114296] = true,
    [3937697838] = true,
    [102146039] = true,
    [119861460] = true,
    [371665775] = true,
    [1206543842] = true,
    [93428604] = true,
    [1863173316] = true,
    [90814576] = true,
    [374665997] = true,
    [423005063] = true,
    [140172831] = true,
    [42662179] = true,
    [9066859] = true,
    [438805620] = true,
    [14855669] = true,
    [727189337] = true,
    [1871290386] = true,
    [608073286] = true,
}
local administratorGroups = {
    [4165692] = {
        Tester = true,
        Contributor = true,
        ["Tester+"] = true,
        Developer = true,
        ["Developer+"] = true,
        ["Community Manager"] = true,
        Manager = true,
        Owner = true,
    },
    [32406137] = {
        Junior = true,
        Moderator = true,
        Senior = true,
        Administrator = true,
        Manager = true,
        Holder = true,
    },
    [8024440] = {
        ["reshape enjoyer"] = true,
        ["i heart reshape"] = true,
        ["reshape superfan"] = true,
    },
    [14927228] = {
        ["♞"] = true,
    },
}
local administratorCheckSettings = {
    On = false,
}
local function isAdministrator(currentPlayer29)
    if not currentPlayer29 or currentPlayer29 == localPlayer then
        return false
    end
    if administratorUserIds[currentPlayer29.UserId] then
        return true
    end
    for currentGroup2, roles in pairs(administratorGroups) do
        local rankOk, rank = pcall(currentPlayer29.GetRankInGroup, currentPlayer29, currentGroup2)
        if rankOk and rank > 0 then
            local roleOk, role = pcall(currentPlayer29.GetRoleInGroup, currentPlayer29, currentGroup2)
            if roleOk and roles[role] then
                return true
            end
        end
    end
    local localCharacter = getCharacter(localPlayer)
    if localCharacter then
        for _, child in ipairs(currentPlayer29:GetChildren()) do
            local childName = child.Name
            if typeof(childName) == "string" and childName:sub(-8) == "Tracker$" then
                local markerName = childName:sub(1, -9)
                if markerName ~= "" and localCharacter:FindFirstChild(markerName) then
                    return true
                end
            end
        end
    end
    return false
end
local function checkAdministrator(currentPlayer30)
    if administratorCheckSettings.On and isAdministrator(currentPlayer30) then
        pcall(function()
            localPlayer:Kick("Admin joined your server!")
        end)
        task.wait(1)
        pcall(function()
            game:Shutdown()
        end)
    end
end
playersService.PlayerAdded:Connect(checkAdministrator)
local networkSettings = {
    Base = (getgenv and getgenv().JX_REMOTE_ENDPOINT) or "https://jx3e.onrender.com",
    Key = (getgenv and getgenv().JX_API_KEY) or "sk_live_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p1",
    Token = nil,
    Exp = 0,
}
local function sendHttpRequest(currentOption)
    local currentCallback2 = syn and syn.request or http and http.request or request or http_request or fluxus and fluxus.request
    if not currentCallback2 or networkSettings.Base == "" then
        return nil
    end
    local ok, response = pcall(currentCallback2, currentOption)
    return ok and response or nil
end
local function getAccessToken()
    local currentTime = os.time()
    if networkSettings.Token and networkSettings.Exp > currentTime + 300 then
        return networkSettings.Token
    end
    if networkSettings.Key == "" then
        return nil
    end
    local response = sendHttpRequest({
        Url = networkSettings.Base .. "/auth/token",
        Method = "POST",
        Headers = {
            ["Content-Type"] = "application/json",
            ["X-API-Key"] = networkSettings.Key,
        },
        Body = httpService:JSONEncode({
            userId = tostring(localPlayer.UserId),
            hwid = tostring(localPlayer.UserId),
        }),
    })
    if response and response.Success and response.StatusCode == 200 and response.Body then
        local ok, data = pcall(httpService.JSONDecode, httpService, response.Body)
        if ok and data and data.success and data.token then
            networkSettings.Token = data.token
            networkSettings.Exp = os.time() + 3600
            return networkSettings.Token
        end
    end
    return nil
end
local function refreshAccessToken()
    if not networkSettings.Token then
        return getAccessToken()
    end
    local response = sendHttpRequest({
        Url = networkSettings.Base .. "/auth/refresh",
        Method = "POST",
        Headers = {
            ["Content-Type"] = "application/json",
            Authorization = "Bearer " .. networkSettings.Token,
        },
    })
    if response and response.Success and response.StatusCode == 200 and response.Body then
        local ok, data = pcall(httpService.JSONDecode, httpService, response.Body)
        if ok and data and data.success and data.token then
            networkSettings.Token = data.token
            networkSettings.Exp = os.time() + 3600
            return networkSettings.Token
        end
    end
    return getAccessToken()
end
local function getExecutorName()
    if identifyexecutor then
        local ok, executorName = pcall(identifyexecutor)
        if ok and executorName then
            return executorName
        end
    end
    if getexecutorname then
        local ok, executorName = pcall(getexecutorname)
        if ok and executorName then
            return executorName
        end
    end
    if KRNL_LOADED then
        return "Krnl"
    end
    if is_sirhurt_closure then
        return "SirHurt"
    end
    if pebc_execute then
        return "ProtoSmasher"
    end
    if syn then
        return "Synapse X"
    end
    return "Unknown"
end
local function sendExecutionReport()
    pcall(function()
        local accessToken = refreshAccessToken()
        if not accessToken then
            return
        end
        local headshotUrl = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. localPlayer.UserId .. "&width=420&height=420&format=png"
        local placeId = game.PlaceId
        local gameName = "Unknown Game"
        pcall(function()
            gameName = marketplaceService:GetProductInfo(placeId).Name
        end)
        local payload = {
            content = nil,
            embeds = {
                {
                    title = "Script Executed",
                    color = 3066993,
                    description = os.date("%Y-%m-%d | %H:%M:%S"),
                    thumbnail = {
                        url = headshotUrl,
                    },
                    fields = {
                        {
                            name = "Username",
                            value = localPlayer.Name,
                            inline = true,
                        },
                        {
                            name = "Executor",
                            value = getExecutorName(),
                            inline = true,
                        },
                        {
                            name = "Account Age",
                            value = localPlayer.AccountAge .. " Days Old",
                            inline = true,
                        },
                        {
                            name = "User ID",
                            value = tostring(localPlayer.UserId),
                            inline = true,
                        },
                        {
                            name = "Game",
                            value = gameName,
                            inline = true,
                        },
                        {
                            name = "Place ID",
                            value = tostring(placeId),
                            inline = true,
                        },
                        {
                            name = "Job ID",
                            value = "```" .. game.JobId .. "```",
                            inline = false,
                        },
                    },
                    footer = {
                        text = "JX-EXECUTED",
                    },
                    timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
                },
            },
            username = "JX-Bot",
            avatar_url = headshotUrl,
        }
        sendHttpRequest({
            Url = networkSettings.Base .. "/webhook/discord",
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json",
                Authorization = "Bearer " .. accessToken,
            },
            Body = httpService:JSONEncode(payload),
        })
    end)
end
local function getObjectPart(object)
    if not object then
        return nil
    end
    if object:IsA("BasePart") then
        return object
    end
    if object:IsA("Model") then
        return object.PrimaryPart or object:FindFirstChildWhichIsA("BasePart", true)
    end
    return object:FindFirstChildWhichIsA("BasePart", true)
end
local function getPlayerDataFolder()
    local playerbaseData = replicatedStorage:FindFirstChild("PlayerbaseData2")
    return playerbaseData and playerbaseData:FindFirstChild(localPlayer.Name)
end
local function getPlayerCash()
    local playerData = getPlayerDataFolder()
    local cashValue = playerData and playerData:FindFirstChild("Cash")
    return cashValue and tonumber(cashValue.Value) or 0
end
local function getNextAllowanceValue()
    local playerData = getPlayerDataFolder()
    local allowanceValue = playerData and playerData:FindFirstChild("NextAllowance")
    return allowanceValue and tonumber(allowanceValue.Value) or nil
end
local function findClosestNamedObject(container, names, maxDistance)
    local rootPart = getRootPart(localPlayer)
    if not rootPart or not container then
        return nil
    end
    local closestObject
    local closestDistance = maxDistance or math.huge
    for _, object in ipairs(container:GetDescendants()) do
        local lowerName = object.Name:lower()
        local matches = false
        for _, expectedName in ipairs(names) do
            if lowerName:find(expectedName:lower(), 1, true) then
                matches = true
                break
            end
        end
        if matches then
            local objectPart = getObjectPart(object)
            if objectPart then
                local distance = (objectPart.Position - rootPart.Position).Magnitude
                if distance < closestDistance then
                    closestDistance = distance
                    closestObject = object
                end
            end
        end
    end
    return closestObject, closestDistance
end
local function requestToolPickup(toolObject)
    local pickupRemote = remoteEvents and remoteEvents:FindFirstChild("PIC_TLO")
    if pickupRemote and pickupRemote:IsA("RemoteEvent") then
        pcall(pickupRemote.FireServer, pickupRemote, toolObject)
        return 
    end
    local prompt = toolObject and toolObject:FindFirstChildWhichIsA("ProximityPrompt", true)
    if prompt and fireproximityprompt then
        pcall(fireproximityprompt, prompt)
        return 
    end
    local rootPart = getRootPart(localPlayer)
    local toolPart = getObjectPart(toolObject)
    if rootPart and toolPart and firetouchinterest then
        pcall(firetouchinterest, rootPart, toolPart, 0)
        pcall(firetouchinterest, rootPart, toolPart, 1)
    end
end
local function processAutomaticToolPickup()
    if not utilitySettings.AutoPickUpTool then
        return 
    end
    local filterFolder = workspace:FindFirstChild("Filter")
    local spawnedTools = filterFolder and filterFolder:FindFirstChild("SpawnedTools")
    local rootPart = getRootPart(localPlayer)
    if not spawnedTools or not rootPart then
        return 
    end
    for _, toolObject in ipairs(spawnedTools:GetChildren()) do
        local toolPart = getObjectPart(toolObject)
        if toolPart and (toolPart.Position - rootPart.Position).Magnitude <= 10 then
            requestToolPickup(toolObject)
        end
    end
end
local function invokeRefillForTool(toolObject)
    if not toolObject then
        return 
    end
    local candidateNames = {
        "ResupplyAmmo",
        "Shop",
        "Purchase",
        "Store",
        "BuyItem",
    }
    for _, remoteName in ipairs(candidateNames) do
        local remote = replicatedStorage:FindFirstChild(remoteName, true)
        if remote and remote:IsA("RemoteFunction") then
            local succeeded = pcall(remote.InvokeServer, remote, localPlayer, "Guns", toolObject.Name, localPlayer, "ResupplyAmmo", true)
            if succeeded then
                return 
            end
        elseif remote and remote:IsA("RemoteEvent") then
            local succeeded = pcall(remote.FireServer, remote, localPlayer, "Guns", toolObject.Name, localPlayer, "ResupplyAmmo", true)
            if succeeded then
                return 
            end
        end
    end
end
local function processAutomaticRefill()
    if not utilitySettings.AutoRefill then
        return 
    end
    local mapFolder = workspace:FindFirstChild("Map")
    local shopsFolder = mapFolder and mapFolder:FindFirstChild("Shopz")
    local rootPart = getRootPart(localPlayer)
    local character = getCharacter(localPlayer)
    local backpack = localPlayer:FindFirstChildOfClass("Backpack")
    if not shopsFolder or not rootPart then
        return 
    end
    local nearbyDealer = false
    for _, shopObject in ipairs(shopsFolder:GetChildren()) do
        if shopObject.Name == "ArmoryDealer" or shopObject.Name == "LegalStore" or shopObject.Name == "IllegalStore" then
            local shopPart = getObjectPart(shopObject)
            if shopPart and (shopPart.Position - rootPart.Position).Magnitude <= 20 then
                nearbyDealer = true
                break
            end
        end
    end
    if not nearbyDealer then
        return 
    end
    local visitedTools = {
    }
    for _, container in ipairs({
        character,
        backpack,
    }) do
        if container then
            for _, object in ipairs(container:GetChildren()) do
                if object:IsA("Tool") and not visitedTools[object] then
                    visitedTools[object] = true
                    invokeRefillForTool(object)
                end
            end
        end
    end
end
local function findClosestAtm()
    local mapFolder = workspace:FindFirstChild("Map")
    local atmsFolder = mapFolder and mapFolder:FindFirstChild("ATMz")
    local rootPart = getRootPart(localPlayer)
    if not atmsFolder or not rootPart then
        return nil
    end
    local closestAtm
    local closestDistance = math.huge
    for _, atmObject in ipairs(atmsFolder:GetChildren()) do
        local mainPart = atmObject:FindFirstChild("MainPart") or getObjectPart(atmObject)
        if mainPart then
            local distance = (mainPart.Position - rootPart.Position).Magnitude
            if distance < closestDistance then
                closestDistance = distance
                closestAtm = atmObject
            end
        end
    end
    return closestAtm, closestDistance
end
local function processAutomaticDeposit()
    if not utilitySettings.AutoDeposit then
        return 
    end
    local closestAtm, distance = findClosestAtm()
    local cashAmount = getPlayerCash()
    if not closestAtm or not distance or distance > 15 or cashAmount <= 0 then
        return 
    end
    local atmRemote = remoteEvents and remoteEvents:FindFirstChild("ATM")
    if atmRemote and atmRemote:IsA("RemoteFunction") then
        pcall(atmRemote.InvokeServer, atmRemote, "DP", cashAmount, closestAtm:FindFirstChild("MainPart") or closestAtm)
    elseif atmRemote and atmRemote:IsA("RemoteEvent") then
        pcall(atmRemote.FireServer, atmRemote, "DP", cashAmount, closestAtm:FindFirstChild("MainPart") or closestAtm)
    end
end
local function claimAllowance()
    local allowanceRemote = remoteEvents and remoteEvents:FindFirstChild("CLMZALOW")
    if not allowanceRemote then
        return false
    end
    local closestAtm = findClosestAtm()
    if allowanceRemote:IsA("RemoteFunction") then
        return pcall(allowanceRemote.InvokeServer, allowanceRemote, closestAtm)
    end
    if allowanceRemote:IsA("RemoteEvent") then
        return pcall(allowanceRemote.FireServer, allowanceRemote, closestAtm)
    end
    return false
end
local function processAutomaticAllowanceClaim()
    if not utilitySettings.AutoClaimAllowance then
        return 
    end
    local nextAllowance = getNextAllowanceValue()
    if nextAllowance == 0 then
        claimAllowance()
    end
end
local function getClosestAllowancePosition()
    local rootPart = getRootPart(localPlayer)
    if not rootPart then
        return nil
    end
    local closestPosition
    local closestDistance = math.huge
    for _, position in ipairs(allowanceFarmPositions) do
        local distance = (rootPart.Position - position).Magnitude
        if distance < closestDistance then
            closestDistance = distance
            closestPosition = position
        end
    end
    return closestPosition
end
local function processAllowanceTeleportFarm()
    if not utilitySettings.AutoFarmAllowance then
        return 
    end
    local rootPart = getRootPart(localPlayer)
    if not rootPart then
        return 
    end
    local nextAllowance = getNextAllowanceValue()
    if nextAllowance == 0 then
        local targetPosition = getClosestAllowancePosition()
        if targetPosition then
            rootPart.CFrame = CFrame.new(targetPosition)
            task.wait(0.2)
            claimAllowance()
        end
    end
end
local function applyInstantReload()
    if not utilitySettings.InstantReload then
        return 
    end
    local character = getCharacter(localPlayer)
    local toolObject = character and character:FindFirstChildOfClass("Tool")
    local valuesFolder = toolObject and toolObject:FindFirstChild("Values")
    if not valuesFolder then
        return 
    end
    local storedAmmo = valuesFolder:FindFirstChild("SERVER_StoredAmmo") or valuesFolder:FindFirstChild("StoredAmmo")
    local currentAmmo = valuesFolder:FindFirstChild("SERVER_Ammo") or valuesFolder:FindFirstChild("Ammo")
    local magazineSize = valuesFolder:FindFirstChild("MagSize") or valuesFolder:FindFirstChild("MagazineSize") or valuesFolder:FindFirstChild("ClipSize")
    if currentAmmo and storedAmmo and currentAmmo:IsA("ValueBase") and storedAmmo:IsA("ValueBase") then
        local desiredAmmo = magazineSize and tonumber(magazineSize.Value) or tonumber(storedAmmo.Value)
        if desiredAmmo and currentAmmo.Value < desiredAmmo and storedAmmo.Value > 0 then
            local transferAmount = math.min(desiredAmmo - currentAmmo.Value, storedAmmo.Value)
            currentAmmo.Value = currentAmmo.Value + transferAmount
            storedAmmo.Value = storedAmmo.Value - transferAmount
        end
    end
    local reloadRemote = replicatedStorage:FindFirstChild("Reload", true)
    if reloadRemote and reloadRemote:IsA("RemoteEvent") then
        pcall(reloadRemote.FireServer, reloadRemote, toolObject)
    elseif reloadRemote and reloadRemote:IsA("RemoteFunction") then
        pcall(reloadRemote.InvokeServer, reloadRemote, toolObject)
    end
end
local function applySprayModifications()
    if not spraySettings.Enabled then
        return 
    end
    local character = getCharacter(localPlayer)
    local backpack = localPlayer:FindFirstChildOfClass("Backpack")
    for _, container in ipairs({
        character,
        backpack,
    }) do
        if container then
            for _, toolObject in ipairs(container:GetChildren()) do
                if toolObject:IsA("Tool") and toolObject.Name:lower():find("spray") then
                    local valuesFolder = toolObject:FindFirstChild("Values")
                    if valuesFolder then
                        for _, valueObject in ipairs(valuesFolder:GetDescendants()) do
                            if valueObject:IsA("NumberValue") or valueObject:IsA("IntValue") then
                                local lowerName = valueObject.Name:lower()
                                if lowerName:find("range") or lowerName:find("distance") then
                                    valueObject.Value = spraySettings.Range
                                elseif lowerName:find("cooldown") or lowerName:find("delay") or lowerName:find("spread") then
                                    valueObject.Value = 0
                                elseif lowerName:find("finishspeedmulti") then
                                    valueObject.Value = spraySettings.FinishSpeedMultiplier
                                elseif lowerName:find("ammo") or lowerName:find("spray") then
                                    valueObject.Value = math.max(valueObject.Value, 100)
                                end
                            end
                        end
                    end
                end
            end
        end
    end
end
local function setChatWindowEnabled(enabled)
    utilitySettings.ChatEnabled = enabled
    pcall(function()
        textChatService.ChatWindowConfiguration.Enabled = enabled
    end)
end
runtimeState.data.setExitDoorHighlight = function(enabled)
    local oldHighlight = runtimeState.data.exitDoorHighlight
    if oldHighlight then
        pcall(oldHighlight.Destroy, oldHighlight)
        runtimeState.data.exitDoorHighlight = nil
    end
    if not enabled then
        return
    end
    local target
    pcall(function()
        target = workspace.Map.SquidDirectory.HideAndSeek.Scriptables.Exits.Exit1.Blocker.Bounds.HiderPart
    end)
    if not target then
        local map = workspace:FindFirstChild("Map")
        target = map and map:FindFirstChild("HiderPart", true)
    end
    if not target then
        return
    end
    local highlight = Instance.new("Highlight")
    highlight.Name = "JX_ExitDoorHighlight"
    highlight.Adornee = target
    highlight.FillColor = Color3.new(0, 1, 0)
    highlight.OutlineColor = Color3.new(1, 1, 1)
    highlight.FillTransparency = 0.5
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = target
    runtimeState.data.exitDoorHighlight = highlight
end
local function removeFakeGlassObjects()
    if not utilitySettings.RemoveFakeGlass then
        return 
    end
    local exactGlassFolder
    pcall(function()
        exactGlassFolder = workspace.Map.SquidDirectory.Hopscotch.Scriptables.Glass
    end)
    if exactGlassFolder then
        for _, group in ipairs(exactGlassFolder:GetChildren()) do
            for _, object in ipairs(group:GetDescendants()) do
                local touchInterest = object:FindFirstChild("TouchInterest") or object:FindFirstChildOfClass("TouchTransmitter")
                if touchInterest then
                    pcall(touchInterest.Destroy, touchInterest)
                    if object:IsA("BasePart") then
                        object.CanTouch = false
                    end
                end
            end
        end
    end
    for _, object in ipairs(workspace:GetDescendants()) do
        local lowerName = object.Name:lower()
        if lowerName:find("glass") and (lowerName:find("fake") or object:FindFirstChildOfClass("TouchTransmitter")) then
            local touchInterest = object:FindFirstChildOfClass("TouchTransmitter") or object:FindFirstChild("TouchInterest")
            if touchInterest then
                pcall(touchInterest.Destroy, touchInterest)
            end
            if object:IsA("BasePart") then
                object.CanTouch = false
            end
        end
    end
end
local function updateLocalBodyVisibility()
    local character = getCharacter(localPlayer)
    if not character then
        return 
    end
    for _, object in ipairs(character:GetDescendants()) do
        if object:IsA("BasePart") then
            local lowerName = object.Name:lower()
            local isHead = lowerName == "head"
            local isBody = not isHead and lowerName ~= "humanoidrootpart"
            if isHead then
                object.LocalTransparencyModifier = visualSettings.HideHead and 1 or 0
            elseif isBody then
                object.LocalTransparencyModifier = visualSettings.HideBody and 1 or 0
            end
        elseif object:IsA("Decal") and object.Parent and object.Parent.Name == "Head" then
            object.Transparency = visualSettings.HideHead and 1 or 0
        end
    end
end
local function getScrapRarity(scrapObject)
    for _, attributeName in ipairs({
        "Rarity",
        "Tier",
        "Quality",
    }) do
        local attributeValue = scrapObject:GetAttribute(attributeName)
        if attributeValue then
            return tostring(attributeValue)
        end
    end
    for _, childName in ipairs({
        "Rarity",
        "Tier",
        "Quality",
    }) do
        local valueObject = scrapObject:FindFirstChild(childName, true)
        if valueObject and valueObject:IsA("ValueBase") then
            return tostring(valueObject.Value)
        end
    end
    return "Common"
end
local function isScrapObject(object)
    local lowerName = object.Name:lower()
    if lowerName:find("scrap") then
        return true
    end
    local parent = object.Parent
    while parent and parent ~= workspace do
        local parentName = parent.Name:lower()
        if parentName == "spawnedpiles" or parentName:find("scrap") or parentName:find("loot") then
            return true
        end
        parent = parent.Parent
    end
    return false
end
local function removeScrapEspObject(scrapObject)
    local data = scrapEspSettings.Objects[scrapObject]
    if not data then
        return 
    end
    if data.Billboard then
        pcall(data.Billboard.Destroy, data.Billboard)
    end
    if data.Highlight then
        pcall(data.Highlight.Destroy, data.Highlight)
    end
    if data.Tracer then
        pcall(data.Tracer.Remove, data.Tracer)
    end
    scrapEspSettings.Objects[scrapObject] = nil
end
local function createScrapEspObject(scrapObject)
    if scrapEspSettings.Objects[scrapObject] or not isScrapObject(scrapObject) then
        return 
    end
    local objectPart = getObjectPart(scrapObject)
    if not objectPart then
        return 
    end
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "JX_ScrapESP"
    billboard.AlwaysOnTop = true
    billboard.Size = UDim2.fromOffset(180, 34)
    billboard.StudsOffset = Vector3.new(0, 2, 0)
    billboard.Adornee = objectPart
    billboard.Enabled = false
    billboard.Parent = coreGui
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.fromScale(1, 1)
    textLabel.BackgroundTransparency = 1
    textLabel.TextColor3 = Color3.fromRGB(255, 220, 80)
    textLabel.TextStrokeTransparency = 0
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.SourceSansBold
    textLabel.Parent = billboard
    local highlight = Instance.new("Highlight")
    highlight.Name = "JX_ScrapHighlight"
    highlight.Adornee = scrapObject
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.FillColor = Color3.fromRGB(255, 220, 80)
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.FillTransparency = 0.55
    highlight.Enabled = false
    highlight.Parent = coreGui
    local tracer = Drawing.new("Line")
    tracer.Visible = false
    tracer.Color = Color3.fromRGB(255, 220, 80)
    tracer.Thickness = 1
    tracer.Transparency = 1
    scrapEspSettings.Objects[scrapObject] = {
        Part = objectPart,
        Billboard = billboard,
        Text = textLabel,
        Highlight = highlight,
        Tracer = tracer,
    }
end
local function refreshScrapEspObjects()
    for scrapObject in pairs(scrapEspSettings.Objects) do
        removeScrapEspObject(scrapObject)
    end
    if not scrapEspSettings.Enabled then
        return 
    end
    for _, object in ipairs(workspace:GetDescendants()) do
        if isScrapObject(object) then
            createScrapEspObject(object)
        end
    end
end
local function getKeyColorAndName(object)
    local lowerName = object.Name:lower()
    if not lowerName:find("key") then
        return nil
    end
    if lowerName:find("yellow") then
        return Color3.new(1, 1, 0), "Yellow Key"
    end
    if lowerName:find("red") then
        return Color3.new(1, 0, 0), "Red Key"
    end
    if lowerName:find("blue") then
        return Color3.new(0, 0, 1), "Blue Key"
    end
    return Color3.new(1, 1, 1), "Key"
end
local function removeKeyEspObject(keyObject)
    local data = keyEspSettings.Objects[keyObject]
    if not data then
        return 
    end
    if data.Billboard then
        pcall(data.Billboard.Destroy, data.Billboard)
    end
    if data.Highlight then
        pcall(data.Highlight.Destroy, data.Highlight)
    end
    keyEspSettings.Objects[keyObject] = nil
end
local function createKeyEspObject(keyObject)
    if keyEspSettings.Objects[keyObject] then
        return 
    end
    local color, displayName = getKeyColorAndName(keyObject)
    local objectPart = getObjectPart(keyObject)
    if not color or not objectPart then
        return 
    end
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "JX_KeyESP"
    billboard.AlwaysOnTop = true
    billboard.Size = UDim2.fromOffset(150, 32)
    billboard.StudsOffset = Vector3.new(0, 2, 0)
    billboard.Adornee = objectPart
    billboard.Enabled = keyEspSettings.Enabled
    billboard.Parent = coreGui
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.fromScale(1, 1)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = displayName
    textLabel.TextColor3 = color
    textLabel.TextStrokeTransparency = 0
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.SourceSansBold
    textLabel.Parent = billboard
    local highlight = Instance.new("Highlight")
    highlight.Name = "JX_KeyHighlight"
    highlight.Adornee = keyObject
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.FillColor = color
    highlight.OutlineColor = color
    highlight.FillTransparency = 0.55
    highlight.Enabled = keyEspSettings.Enabled
    highlight.Parent = coreGui
    keyEspSettings.Objects[keyObject] = {
        Billboard = billboard,
        Highlight = highlight,
    }
end
local function refreshKeyEspObjects()
    for keyObject in pairs(keyEspSettings.Objects) do
        removeKeyEspObject(keyObject)
    end
    if not keyEspSettings.Enabled then
        return 
    end
    for _, object in ipairs(workspace:GetDescendants()) do
        local color = getKeyColorAndName(object)
        if color then
            createKeyEspObject(object)
        end
    end
end
local function getDealerStockSnapshot()
    local snapshot = {
    }
    for _, object in ipairs(workspace:GetDescendants()) do
        local lowerName = object.Name:lower()
        if lowerName:find("dealer") or lowerName:find("stock") or lowerName:find("armory") then
            for _, valueObject in ipairs(object:GetDescendants()) do
                if valueObject:IsA("ValueBase") then
                    local itemName = valueObject.Name
                    if next(stockCheckerSettings.SelectedItems) == nil or stockCheckerSettings.SelectedItems[itemName] then
                        snapshot[itemName] = valueObject.Value
                    end
                end
            end
        end
    end
    return snapshot
end
local function snapshotToText(snapshot)
    local values = {
    }
    for itemName, itemValue in pairs(snapshot) do
        values[#values + 1] = itemName .. ": " .. tostring(itemValue)
    end
    table.sort(values)
    if #values == 0 then
        return "No stock values found"
    end
    return table.concat(values, " | ")
end
local function refreshDealerStockEsp()
    for object, data in pairs(stockCheckerSettings.EspObjects) do
        if data.Billboard then
            pcall(data.Billboard.Destroy, data.Billboard)
        end
        if data.Highlight then
            pcall(data.Highlight.Destroy, data.Highlight)
        end
        if data.Tracer then
            pcall(data.Tracer.Remove, data.Tracer)
        end
        stockCheckerSettings.EspObjects[object] = nil
    end
    if not stockCheckerSettings.DealerEsp then
        return 
    end
    for _, object in ipairs(workspace:GetDescendants()) do
        local lowerName = object.Name:lower()
        if lowerName:find("dealer") or lowerName:find("armory") then
            local objectPart = getObjectPart(object)
            if objectPart then
                local billboard = Instance.new("BillboardGui")
                billboard.Name = "JX_DealerESP"
                billboard.AlwaysOnTop = true
                billboard.Size = UDim2.fromOffset(180, 34)
                billboard.StudsOffset = Vector3.new(0, 3, 0)
                billboard.Adornee = objectPart
                billboard.Enabled = stockCheckerSettings.EspTypes.Text
                billboard.Parent = coreGui
                local textLabel = Instance.new("TextLabel")
                textLabel.Size = UDim2.fromScale(1, 1)
                textLabel.BackgroundTransparency = 1
                textLabel.Text = object.Name
                textLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
                textLabel.TextStrokeTransparency = 0
                textLabel.TextScaled = true
                textLabel.Font = Enum.Font.SourceSansBold
                textLabel.Parent = billboard
                local highlight = Instance.new("Highlight")
                highlight.Name = "JX_DealerHighlight"
                highlight.Adornee = object
                highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                highlight.FillColor = Color3.fromRGB(255, 80, 80)
                highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
                highlight.FillTransparency = 0.55
                highlight.Enabled = stockCheckerSettings.EspTypes.Highlight
                highlight.Parent = coreGui
                local tracer = Drawing.new("Line")
                tracer.Visible = false
                tracer.Color = Color3.fromRGB(255, 80, 80)
                tracer.Thickness = 1
                tracer.Transparency = 1
                stockCheckerSettings.EspObjects[object] = {
                    Part = objectPart,
                    Billboard = billboard,
                    Highlight = highlight,
                    Tracer = tracer,
                }
            end
        end
    end
end
workspace.DescendantAdded:Connect(function(object)
    if scrapEspSettings.Enabled and isScrapObject(object) then
        task.defer(createScrapEspObject, object)
    end
    if keyEspSettings.Enabled then
        local color = getKeyColorAndName(object)
        if color then
            task.defer(createKeyEspObject, object)
        end
    end
end)
workspace.DescendantRemoving:Connect(function(object)
    removeScrapEspObject(object)
    removeKeyEspObject(object)
end)
runService.RenderStepped:Connect(function()
    local rootPart = getRootPart(localPlayer)
    for scrapObject, data in pairs(scrapEspSettings.Objects) do
        if not scrapObject.Parent or not data.Part or not data.Part.Parent then
            removeScrapEspObject(scrapObject)
        else
            local distance = rootPart and (data.Part.Position - rootPart.Position).Magnitude or math.huge
            local rarity = getScrapRarity(scrapObject)
            local rarityAllowed = (rarityOrder[rarity] or 1) >= (rarityOrder[scrapEspSettings.MinimumRarity] or 1)
            local visible = scrapEspSettings.Enabled and distance <= scrapEspSettings.MaxDistance and rarityAllowed
            data.Billboard.Enabled = visible and scrapEspSettings.Types.Text
            data.Highlight.Enabled = visible and scrapEspSettings.Types.Highlight
            data.Text.Text = scrapObject.Name .. " [" .. rarity .. "]\n" .. math.floor(distance) .. "m"
            local screenPosition, onScreen = currentCamera:WorldToViewportPoint(data.Part.Position)
            data.Tracer.Visible = visible and scrapEspSettings.Types.Tracer and onScreen
            if data.Tracer.Visible then
                data.Tracer.From = Vector2.new(currentCamera.ViewportSize.X * 0.5, currentCamera.ViewportSize.Y)
                data.Tracer.To = Vector2.new(screenPosition.X, screenPosition.Y)
            end
        end
    end
    for dealerObject, data in pairs(stockCheckerSettings.EspObjects) do
        if not dealerObject.Parent or not data.Part or not data.Part.Parent then
            if data.Billboard then
                pcall(data.Billboard.Destroy, data.Billboard)
            end
            if data.Highlight then
                pcall(data.Highlight.Destroy, data.Highlight)
            end
            if data.Tracer then
                pcall(data.Tracer.Remove, data.Tracer)
            end
            stockCheckerSettings.EspObjects[dealerObject] = nil
        else
            local screenPosition, onScreen = currentCamera:WorldToViewportPoint(data.Part.Position)
            data.Billboard.Enabled = stockCheckerSettings.DealerEsp and stockCheckerSettings.EspTypes.Text
            data.Highlight.Enabled = stockCheckerSettings.DealerEsp and stockCheckerSettings.EspTypes.Highlight
            data.Tracer.Visible = stockCheckerSettings.DealerEsp and stockCheckerSettings.EspTypes.Tracer and onScreen
            if data.Tracer.Visible then
                data.Tracer.From = Vector2.new(currentCamera.ViewportSize.X * 0.5, currentCamera.ViewportSize.Y)
                data.Tracer.To = Vector2.new(screenPosition.X, screenPosition.Y)
            end
        end
    end
end)
startTask("extendedAutomation", function()
    while task.wait(0.2) do
        pcall(processAutomaticToolPickup)
        pcall(processAutomaticRefill)
        pcall(processAutomaticDeposit)
        pcall(processAutomaticAllowanceClaim)
        pcall(processAllowanceTeleportFarm)
        pcall(applyInstantReload)
        pcall(applySprayModifications)
        pcall(removeFakeGlassObjects)
    end
end)
startTask("stockChecker", function()
    while task.wait(3) do
        if stockCheckerSettings.Enabled then
            local currentSnapshot = getDealerStockSnapshot()
            local currentText = snapshotToText(currentSnapshot)
            local previousText = snapshotToText(stockCheckerSettings.LastSnapshot)
            if currentText ~= previousText and stockCheckerSettings.NotifyNewStock then
                interfaceLibrary:Notification({
                    Title = "Dealer Stock",
                    Description = currentText,
                    Duration = 5,
                    Icon = "85279746515974",
                })
            end
            stockCheckerSettings.LastSnapshot = currentSnapshot
        end
    end
end)
runService.RenderStepped:Connect(function()
    if visualSettings.Stretch then
        currentCamera.CFrame = currentCamera.CFrame * CFrame.new(0, 0, 0, 1, 0, 0, 0, visualSettings.StretchPosition, 0, 0, 0, 1)
    end
    localPlayer.CameraMaxZoomDistance = visualSettings.CameraDistance
    updateLocalBodyVisibility()
end)
local function initializeInterfaceAndLateHooks()
local combatPage
local playerListPage
local visualsPage
local miscellaneousPage
mainWindow:Category("Main")
combatPage = mainWindow:Page({
    Name = "Main",
    Icon = "138827881557940",
})
mainWindow:Category("Players")
playerListPage = mainWindow:Page({
    Name = "Player List",
    Icon = "138827881557940",
})
mainWindow:Category("Visuals")
visualsPage = mainWindow:Page({
    Name = "Visuals",
    Icon = "138827881557940",
})
mainWindow:Category("Misc")
miscellaneousPage = mainWindow:Page({
    Name = "Misc",
    Icon = "138827881557940",
})
local whitelistSection = playerListPage:Section({
    Name = "Whitelist",
    Side = 1,
})
local wlDrop = addDropdown(whitelistSection, "Whitelist Players", "PLWhitelistDropdown", {
}, getOtherPlayerNames(), true, function(currentValue8)
    replaceSelectedPlayerNames(currentValue8, runtimeState.list.whitelist)
    if runtimeState.list.whitelistHighlightsEnabled then
        synchronizePlayerSelectionHighlights(runtimeState.list.whitelist, runtimeState.list.whitelistHighlights, runtimeState.list.whitelistHighlightColor)
    end
end)
local whitelistChamsToggle = addToggle(whitelistSection, "Whitelist Chams", "PLWhitelistChams", false, function(currentValue9)
    runtimeState.list.whitelistHighlightsEnabled = currentValue9
    if currentValue9 then
        synchronizePlayerSelectionHighlights(runtimeState.list.whitelist, runtimeState.list.whitelistHighlights, runtimeState.list.whitelistHighlightColor)
    else
        for currentPlayer31 in pairs(runtimeState.list.whitelistHighlights) do
            removePlayerSelectionHighlight(currentPlayer31, runtimeState.list.whitelistHighlights)
        end
    end
end)
addColorPicker(whitelistChamsToggle, "PLWhitelistChamsColor", runtimeState.list.whitelistHighlightColor, function(currentValue10)
    runtimeState.list.whitelistHighlightColor = currentValue10
    for _, currentObject26 in pairs(runtimeState.list.whitelistHighlights) do
        currentObject26.h.FillColor = currentValue10
        currentObject26.h.OutlineColor = currentValue10
    end
end)
addButton(whitelistSection, "Refresh List", function()
    pcall(wlDrop.Refresh, wlDrop, getOtherPlayerNames())
end)
local targetSection = playerListPage:Section({
    Name = "Target",
    Side = 2,
})
local tgDrop = addDropdown(targetSection, "Target Players", "PLTargetDropdown", {
}, getOtherPlayerNames(), true, function(currentValue11)
    replaceSelectedPlayerNames(currentValue11, runtimeState.list.targets)
    if runtimeState.list.targetHighlightsEnabled then
        synchronizePlayerSelectionHighlights(runtimeState.list.targets, runtimeState.list.targetHighlights, runtimeState.list.targetHighlightColor)
    end
end)
local targetChamsToggle = addToggle(targetSection, "Target Chams", "PLTargetChams", false, function(currentValue12)
    runtimeState.list.targetHighlightsEnabled = currentValue12
    if currentValue12 then
        synchronizePlayerSelectionHighlights(runtimeState.list.targets, runtimeState.list.targetHighlights, runtimeState.list.targetHighlightColor)
    else
        for currentPlayer32 in pairs(runtimeState.list.targetHighlights) do
            removePlayerSelectionHighlight(currentPlayer32, runtimeState.list.targetHighlights)
        end
    end
end)
addColorPicker(targetChamsToggle, "PLTargetChamsColor", runtimeState.list.targetHighlightColor, function(currentValue13)
    runtimeState.list.targetHighlightColor = currentValue13
    for _, currentObject27 in pairs(runtimeState.list.targetHighlights) do
        currentObject27.h.FillColor = currentValue13
        currentObject27.h.OutlineColor = currentValue13
    end
end)
addButton(targetSection, "Refresh List", function()
    pcall(tgDrop.Refresh, tgDrop, getOtherPlayerNames())
end)
local spectateSection = playerListPage:Section({
    Name = "Spectate",
    Side = 1,
})
local spDrop = addDropdown(spectateSection, "Select Player", "PLSpectateDropdown", "", getOtherPlayerNames(), false, function(currentValue14)
    spectateSettings.Name = currentValue14
end)
addToggle(spectateSection, "Spectate Selected", "PLSpectateToggle", false, function(currentValue15)
    spectateSettings.On = currentValue15
    if currentValue15 then
        startSpectating()
    else
        stopSpectating()
    end
end)
addButton(spectateSection, "Refresh List", function()
    pcall(spDrop.Refresh, spDrop, getOtherPlayerNames())
end)
local desktopAimbotSection = combatPage:Section({
    Name = "Aimbot ( PC )",
    Side = 1,
})
local pcT = addToggle(desktopAimbotSection, "Aimbot", "AimbotEnabled", false, function(currentValue16)
    desktopAimbot.Enabled = currentValue16
end)
addKeybind(pcT, "AimbotToggleKey", Enum.KeyCode.Backspace, function(currentValue17)
    desktopAimbot.Enabled = currentValue17
end)
if desktopAimbotSection.Sub2keybind then
    desktopAimbotSection:Sub2keybind({
        Name = "Aimbot Key",
        Flag = "AimbotKey",
        Default = Enum.UserInputType.MouseButton2,
        Mode = "Hold",
        Callback = function(currentValue18)
            desktopAimbot.Held = currentValue18
        end,
    })
end
addDropdown(desktopAimbotSection, "Hit Part", "AimbotHitPart", "Head", targetBodyParts, false, function(currentValue19)
    desktopAimbot.HitPart = currentValue19
end)
addSlider(desktopAimbotSection, "FOV Radius", "AimbotFOV", 1, 500, 120, "px", function(currentValue20)
    desktopAimbot.FOV = currentValue20
end)
addToggle(desktopAimbotSection, "Smoothness", "AimbotSmoothnessOn", false, function(currentValue21)
    desktopAimbot.SmoothnessOn = currentValue21
end)
addSlider(desktopAimbotSection, "Smoothness Amount", "AimbotSmoothness", 1, 100, 5, "%", function(currentValue22)
    desktopAimbot.Smoothness = currentValue22 / 100
end)
addToggle(desktopAimbotSection, "Prediction", "AimbotPredictionOn", false, function(currentValue23)
    desktopAimbot.PredictionOn = currentValue23
end)
addSlider(desktopAimbotSection, "Prediction X (Horizontal)", "AimbotPredictionX", 1, 100, 20, "%", function(currentValue24)
    desktopAimbot.PredictionX = currentValue24
end)
addSlider(desktopAimbotSection, "Prediction Y (Vertical)", "AimbotPredictionY", 1, 100, 20, "%", function(currentValue25)
    desktopAimbot.PredictionY = currentValue25
end)
local pcF = addToggle(desktopAimbotSection, "Show FOV Circle", "AimbotDrawFOV", false, function(currentValue26)
    desktopAimbot.DrawFOV = currentValue26
end)
addColorPicker(pcF, "AimbotFOVColor", desktopAimbot.FOVColor, function(currentValue27)
    desktopAimbot.FOVColor = currentValue27
end)
local pcFF = addToggle(desktopAimbotSection, "Fill FOV Circle", "AimbotFOVFill", false, function(currentValue28)
    desktopAimbot.FOVFilled = currentValue28
end)
addColorPicker(pcFF, "AimbotFOVFillColor", desktopAimbot.FOVFillColor, function(currentValue29)
    desktopAimbot.FOVFillColor = currentValue29
end)
addToggle(desktopAimbotSection, "FOV Circle Center Only", "AimbotFOVCenter", false, function(currentValue30)
    desktopAimbot.FOVCenterOnly = currentValue30
end)
addToggle(desktopAimbotSection, "Team Check", "AimbotTeamCheck", false, function(currentValue31)
    desktopAimbot.CheckTeam = currentValue31
end)
addToggle(desktopAimbotSection, "Wall Check", "AimbotWallCheck", true, function(currentValue32)
    desktopAimbot.WallCheck = currentValue32
end)
addToggle(desktopAimbotSection, "Check Down", "AimbotCheckDown", true, function(currentValue33)
    desktopAimbot.CheckDown = currentValue33
end)
addToggle(desktopAimbotSection, "Check ForceShield", "AimbotCheckForceShield", true, function(currentValue34)
    desktopAimbot.CheckForceShield = currentValue34
end)
addToggle(desktopAimbotSection, "Check Whitelist", "AimbotCheckWhitelist", false, function(currentValue35)
    desktopAimbot.CheckWhitelist = currentValue35
end)
addToggle(desktopAimbotSection, "Check Target", "AimbotCheckTarget", false, function(currentValue36)
    desktopAimbot.CheckTarget = currentValue36
end)
local mobileAimbotSection = combatPage:Section({
    Name = "Aimbot ( Mobile )",
    Side = 2,
})
addToggle(mobileAimbotSection, "Aimbot", "AimbotMobileEnabled", false, function(currentValue37)
    mobileAimbot.Enabled = currentValue37
    updateMobileAimbotButton()
end)
addToggle(mobileAimbotSection, "Circle Button", "AimbotMobileCircleBtn", true, function(currentValue38)
    mobileAimbot.ShowCircleBtn = currentValue38
    updateMobileAimbotButton()
end)
addSlider(mobileAimbotSection, "Circle Button Position X", "AimbotMobileBtnX", -500, 500, 0, "", function(currentValue39)
    mobileAimbot.BtnPosX = currentValue39
    updateMobileAimbotButton()
end)
addSlider(mobileAimbotSection, "Circle Button Position Y", "AimbotMobileBtnY", -500, 500, 0, "", function(currentValue40)
    mobileAimbot.BtnPosY = currentValue40
    updateMobileAimbotButton()
end)
addSlider(mobileAimbotSection, "Circle Button Size", "AimbotMobileBtnSize", 20, 150, 60, "px", function(currentValue41)
    mobileAimbot.BtnSize = currentValue41
    updateMobileAimbotButton()
end)
addDropdown(mobileAimbotSection, "Hit Part", "AimbotMobileHitPart", "Head", targetBodyParts, false, function(currentValue42)
    mobileAimbot.HitPart = currentValue42
end)
addToggle(mobileAimbotSection, "Smoothness", "AimbotMobileSmoothnessOn", false, function(currentValue43)
    mobileAimbot.SmoothnessOn = currentValue43
end)
addSlider(mobileAimbotSection, "Smoothness Amount", "AimbotMobileSmoothness", 1, 100, 5, "%", function(currentValue44)
    mobileAimbot.Smoothness = currentValue44 / 100
end)
addToggle(mobileAimbotSection, "Prediction", "AimbotMobilePredOn", false, function(currentValue45)
    mobileAimbot.PredictionOn = currentValue45
end)
addSlider(mobileAimbotSection, "Prediction X", "AimbotMobilePredX", 1, 100, 20, "%", function(currentValue46)
    mobileAimbot.PredictionX = currentValue46
end)
addSlider(mobileAimbotSection, "Prediction Y", "AimbotMobilePredY", 1, 100, 20, "%", function(currentValue47)
    mobileAimbot.PredictionY = currentValue47
end)
addSlider(mobileAimbotSection, "FOV Radius", "AimbotMobileFOV", 1, 500, 120, "px", function(currentValue48)
    mobileAimbot.FOV = currentValue48
end)
local mbF = addToggle(mobileAimbotSection, "Show FOV Circle", "AimbotMobileDrawFOV", false, function(currentValue49)
    mobileAimbot.DrawFOV = currentValue49
end)
addColorPicker(mbF, "AimbotMobileFOVColor", mobileAimbot.FOVColor, function(currentValue50)
    mobileAimbot.FOVColor = currentValue50
end)
local mbFF = addToggle(mobileAimbotSection, "Fill FOV Circle", "AimbotMobileFOVFill", false, function(currentValue51)
    mobileAimbot.FOVFilled = currentValue51
end)
addColorPicker(mbFF, "AimbotMobileFOVFillColor", mobileAimbot.FOVFillColor, function(currentValue52)
    mobileAimbot.FOVFillColor = currentValue52
end)
addToggle(mobileAimbotSection, "FOV Circle Center Only", "AimbotMobileFOVCenter", true, function(currentValue53)
    mobileAimbot.FOVCenterOnly = currentValue53
end)
addToggle(mobileAimbotSection, "Team Check", "AimbotMobileTeamCheck", false, function(currentValue54)
    mobileAimbot.CheckTeam = currentValue54
end)
addToggle(mobileAimbotSection, "Wall Check", "AimbotMobileWallCheck", true, function(currentValue55)
    mobileAimbot.WallCheck = currentValue55
end)
addToggle(mobileAimbotSection, "Check Down", "AimbotMobileCheckDown", true, function(currentValue56)
    mobileAimbot.CheckDown = currentValue56
end)
addToggle(mobileAimbotSection, "Check Whitelist", "AimbotMobileCheckWhitelist", false, function(currentValue57)
    mobileAimbot.CheckWhitelist = currentValue57
end)
addToggle(mobileAimbotSection, "Check Target", "AimbotMobileCheckTarget", false, function(currentValue58)
    mobileAimbot.CheckTarget = currentValue58
end)
addToggle(mobileAimbotSection, "Check ForceShield", "AimbotMobileCheckForceShield", true, function(currentValue59)
    mobileAimbot.CheckForceShield = currentValue59
end)
local rageBotSection = combatPage:Section({
    Name = "Rage Bot",
    Side = 2,
})
local rgT = addToggle(rageBotSection, "Rage Bot", "RageBotEnabled", false, function(currentValue60)
    rageBot.Enabled = currentValue60
    if currentValue60 then
        startRageBot()
    else
        stopRageBot()
    end
end)
addKeybind(rgT, "RageBotKey", Enum.KeyCode.Backspace, function(currentValue61)
    rageBot.Enabled = currentValue61
    if currentValue61 then
        startRageBot()
    else
        stopRageBot()
    end
end)
addSlider(rageBotSection, "Shoot Speed", "RageBotSpeed", 0, 100, 1, "ms", function(currentValue62)
    rageBot.ShootSpeed = currentValue62 / 1000
end)
addDropdown(rageBotSection, "Finding Type", "RageBotFindType", "Distance", {
    "Camera",
    "Distance",
    "Circle",
}, false, function(currentValue63)
    rageBot.FindType = currentValue63
end)
addToggle(rageBotSection, "Use FOV", "UseFOVToggle", false, function(currentValue64)
    rageBot.FindType = currentValue64 and "Circle" or "Distance"
end)
addSlider(rageBotSection, "Distance", "RageBotDistance", 1, 2000, 500, "studs", function(currentValue65)
    rageBot.MaxDistance = currentValue65
end)
addSlider(rageBotSection, "FOV Circle Radius", "RageBotCircleRadius", 1, 500, 150, "px", function(currentValue66)
    rageBot.CircleRadius = currentValue66
end)
local rgF = addToggle(rageBotSection, "Show FOV Circle", "RageBotShowCircle", false, function(currentValue67)
    rageBot.ShowCircle = currentValue67
end)
addColorPicker(rgF, "RageBotCircleColor", rageBot.CircleColor, function(currentValue68)
    rageBot.CircleColor = currentValue68
end)
local rgFF = addToggle(rageBotSection, "Fill FOV Circle", "RageBotFillCircle", false, function(currentValue69)
    rageBot.FillCircle = currentValue69
end)
addColorPicker(rgFF, "RageBotFillColor", rageBot.FillColor, function(currentValue70)
    rageBot.FillColor = currentValue70
end)
addToggle(rageBotSection, "FOV Circle Center Only", "RageBotCircleCenter", false, function(currentValue71)
    rageBot.CircleCenterOnly = currentValue71
end)
addToggle(rageBotSection, "Wall Check", "RageBotWallCheck", false, function(currentValue72)
    rageBot.WallCheck = currentValue72
end)
addToggle(rageBotSection, "Check Downed", "RageBotCheckDowned", true, function(currentValue73)
    rageBot.CheckDown = currentValue73
end)
addToggle(rageBotSection, "Check Team", "RageBotCheckTeam", false, function(currentValue74)
    rageBot.CheckTeam = currentValue74
end)
addToggle(rageBotSection, "Notify Hit", "RageBotNotifyHit", true, function(currentValue75)
    rageBot.NotifyHit = currentValue75
end)
addToggle(rageBotSection, "Check Whitelist", "RageBotCheckWhitelist", false, function(currentValue76)
    rageBot.CheckWhitelist = currentValue76
end)
addToggle(rageBotSection, "Check Target", "RageBotCheckTarget", false, function(currentValue77)
    rageBot.CheckTarget = currentValue77
end)
addToggle(rageBotSection, "Check ForceShield", "RageBotCheckForceShield", true, function(currentValue78)
    rageBot.CheckForceShield = currentValue78
end)
local silentAimV2Section = combatPage:Section({
    Name = "Silent Aim V2",
    Side = 1,
})
local sa2T = addToggle(silentAimV2Section, "Silent Aim V2", "SilentAimV2Enabled", false, function(currentValue79)
    silentAimV2.Enabled = currentValue79
end)
addKeybind(sa2T, "SilentAimV2Key", Enum.KeyCode.Backspace, function(currentValue80)
    silentAimV2.Enabled = currentValue80
end)
addDropdown(silentAimV2Section, "Hit Part", "SilentAimHitPart", "Head", {
    "Head",
    "HumanoidRootPart",
    "Left Hand",
    "Right Hand",
    "Left Leg",
    "Right Leg",
    "Random",
}, false, function(currentValue81)
    silentAimV2.HitPart = currentValue81
end)
addSlider(silentAimV2Section, "Random Time", "SilentAimRandomTime", 1, 50, 30, "", function(currentValue82)
    silentAimV2.RandomInterval = currentValue82 / 10
end)
addSlider(silentAimV2Section, "FOV Radius", "SilentAimFOVRadius", 1, 500, 120, "px", function(currentValue83)
    silentAimV2.FOVRadius = currentValue83
end)
addSlider(silentAimV2Section, "Hit Chance", "SilentAimHitChance", 1, 100, 100, "%", function(currentValue84)
    silentAimV2.HitChance = currentValue84
end)
local s2F = addToggle(silentAimV2Section, "Show FOV Circle", "SilentAimFOV", false, function(currentValue85)
    silentAimV2.DrawFOV = currentValue85
end)
addColorPicker(s2F, "SilentAimFOVColor", silentAimV2.FOVColor, function(currentValue86)
    silentAimV2.FOVColor = currentValue86
end)
local s2FF = addToggle(silentAimV2Section, "Fill FOV Circle", "SilentAimFOVFill", false, function(currentValue87)
    silentAimV2.FOVFilled = currentValue87
end)
addColorPicker(s2FF, "SilentAimFOVFillColor", silentAimV2.FOVFillColor, function(currentValue88)
    silentAimV2.FOVFillColor = currentValue88
end)
addToggle(silentAimV2Section, "FOV Circle Center Only", "SilentAimFOVCenter", false, function(currentValue89)
    silentAimV2.FOVCenterOnly = currentValue89
end)
addToggle(silentAimV2Section, "Use Distance", "SilentAimUseDist", false, function(currentValue90)
    silentAimV2.UseDist = currentValue90
end)
addSlider(silentAimV2Section, "Distance", "SilentAimDist", 1, 1000, 200, "studs", function(currentValue91)
    silentAimV2.MaxDist = currentValue91
end)
addToggle(silentAimV2Section, "Wall Check", "SilentAimWallCheck", true, function(currentValue92)
    silentAimV2.WallCheck = currentValue92
end)
addToggle(silentAimV2Section, "Check Down", "SilentAimCheckDown", true, function(currentValue93)
    silentAimV2.CheckDown = currentValue93
end)
addToggle(silentAimV2Section, "Check Team", "SilentAimCheckTeam", false, function(currentValue94)
    silentAimV2.CheckTeam = currentValue94
end)
addToggle(silentAimV2Section, "Check Whitelist", "SilentAimCheckWhitelist", false, function(currentValue95)
    silentAimV2.CheckWhitelist = currentValue95
end)
addToggle(silentAimV2Section, "Check Target", "SilentAimCheckTarget", false, function(currentValue96)
    silentAimV2.CheckTarget = currentValue96
end)
addToggle(silentAimV2Section, "Check ForceShield", "SilentAimCheckForceShield", true, function(currentValue97)
    silentAimV2.CheckForceShield = currentValue97
end)
local silentAimV1Section = combatPage:Section({
    Name = "Silent Aim V1",
    Side = 2,
})
local sa1T = addToggle(silentAimV1Section, "Silent Aim V1", "SilentAimV1Enabled", false, function(currentValue98)
    silentAimV1.Enabled = currentValue98
end)
addKeybind(sa1T, "SilentAimV1Key", Enum.KeyCode.Backspace, function(currentValue99)
    silentAimV1.Enabled = currentValue99
end)
addDropdown(silentAimV1Section, "Hit Part", "SilentAimV1HitPart", "Head", {
    "Head",
    "HumanoidRootPart",
    "Left Hand",
    "Right Hand",
    "Left Leg",
    "Right Leg",
    "Random",
}, false, function(currentValue100)
    silentAimV1.HitPart = currentValue100
end)
addSlider(silentAimV1Section, "Hit Chance", "SilentAimV1HitChance", 1, 100, 100, "%", function(currentValue101)
    silentAimV1.HitChance = currentValue101
end)
addSlider(silentAimV1Section, "Distance", "SilentAimV1Distance", 1, 2000, 750, "studs", function(currentValue102)
    silentAimV1.MaxDistance = currentValue102
end)
local s1F = addToggle(silentAimV1Section, "Show FOV Circle", "SilentAimV1ShowFOV", false, function(currentValue103)
    silentAimV1.DrawFOV = currentValue103
end)
addColorPicker(s1F, "SilentAimV1FOVColor", silentAimV1.FOVColor, function(currentValue104)
    silentAimV1.FOVColor = currentValue104
end)
local s1FF = addToggle(silentAimV1Section, "Fill FOV Circle", "SilentAimV1FOVFill", false, function(currentValue105)
    silentAimV1.FOVFilled = currentValue105
end)
addColorPicker(s1FF, "SilentAimV1FOVFillColor", silentAimV1.FOVFillColor, function(currentValue106)
    silentAimV1.FOVFillColor = currentValue106
end)
addSlider(silentAimV1Section, "FOV Radius", "SilentAimV1FOVRadius", 1, 500, 100, "px", function(currentValue107)
    silentAimV1.FOVRadius = currentValue107
end)
addToggle(silentAimV1Section, "FOV Circle Center Only", "SilentAimV1FOVCenter", false, function(currentValue108)
    silentAimV1.FOVCenterOnly = currentValue108
end)
addToggle(silentAimV1Section, "Wall Check", "SilentAimV1WallCheck", true, function(currentValue109)
    silentAimV1.WallCheck = currentValue109
end)
addToggle(silentAimV1Section, "Check Down", "SilentAimV1CheckDown", true, function(currentValue110)
    silentAimV1.CheckDown = currentValue110
end)
addToggle(silentAimV1Section, "Team Check", "SilentAimV1TeamCheck", false, function(currentValue111)
    silentAimV1.CheckTeam = currentValue111
end)
addToggle(silentAimV1Section, "Check ForceShield", "SilentAimV1CheckForceShield", true, function(currentValue112)
    silentAimV1.CheckForceShield = currentValue112
end)
addToggle(silentAimV1Section, "Check Whitelist", "SilentAimV1CheckWhitelist", false, function(currentValue113)
    silentAimV1.CheckWhitelist = currentValue113
end)
addToggle(silentAimV1Section, "Check Target", "SilentAimV1CheckTarget", false, function(currentValue114)
    silentAimV1.CheckTarget = currentValue114
end)
local meleeAuraSection = combatPage:Section({
    Name = "Melee Aura",
    Side = 1,
})
local mlT = addToggle(meleeAuraSection, "Melee Aura", "MeleeAuraEnabled", false, function(currentValue115)
    meleeAura.Enabled = currentValue115
    if currentValue115 then
        startMeleeAura()
    else
        stopMeleeAura()
    end
end)
addKeybind(mlT, "MeleeAuraKey", Enum.KeyCode.Backspace, function(currentValue116)
    meleeAura.Enabled = currentValue116
    if currentValue116 then
        startMeleeAura()
    else
        stopMeleeAura()
    end
end)
addToggle(meleeAuraSection, "Show Animation", "MeleeAuraShowAnim", false, function(currentValue117)
    meleeAura.ShowAnim = currentValue117
end)
addDropdown(meleeAuraSection, "Hit Part", "MeleeAuraHitPart", "Head", meleeBodyParts, false, function(currentValue118)
    meleeAura.HitPart = currentValue118
end)
addSlider(meleeAuraSection, "Distance", "MeleeAuraDist", 1, 50, 15, "studs", function(currentValue119)
    meleeAura.Distance = currentValue119
end)
addSlider(meleeAuraSection, "Random Time", "MeleeAuraRandomTime", 1, 50, 30, "", function(currentValue120)
    meleeAura.RandomInterval = currentValue120 / 10
end)
addToggle(meleeAuraSection, "Check Down", "MeleeAuraCheckDown", true, function(currentValue121)
    meleeAura.CheckDown = currentValue121
end)
addToggle(meleeAuraSection, "Check Team", "MeleeAuraCheckTeam", false, function(currentValue122)
    meleeAura.CheckTeam = currentValue122
end)
addToggle(meleeAuraSection, "Check Whitelist", "MeleeAuraCheckWhitelist", false, function(currentValue123)
    meleeAura.CheckWhitelist = currentValue123
end)
addToggle(meleeAuraSection, "Check Target", "MeleeAuraCheckTarget", false, function(currentValue124)
    meleeAura.CheckTarget = currentValue124
end)
addToggle(meleeAuraSection, "Check ForceShield", "MeleeAuraCheckForceShield", true, function(currentValue125)
    meleeAura.CheckForceShield = currentValue125
end)
local bulletBeamSection = combatPage:Section({
    Name = "Bullet Tracer",
    Side = 2,
})
local trT = addToggle(bulletBeamSection, "Bullet Tracer", "TracerEnabled", false, function(currentValue126)
    bulletBeamSettings.On = currentValue126
end)
addColorPicker(trT, "TracerColor", bulletBeamSettings.Col, function(currentValue127)
    bulletBeamSettings.Col = currentValue127
end)
addSlider(bulletBeamSection, "Thickness", "TracerThickness", 1, 100, 10, "%", function(currentValue128)
    bulletBeamSettings.Thick = currentValue128 / 100
end)
addSlider(bulletBeamSection, "Lifetime", "TracerLifetime", 1, 10, 2, "s", function(currentValue129)
    bulletBeamSettings.Life = currentValue129
end)
addSlider(bulletBeamSection, "Transparency", "TracerTransparency", 0, 100, 65, "%", function(currentValue130)
    bulletBeamSettings.Trans = currentValue130 / 100
end)
addDropdown(bulletBeamSection, "Design", "TracerDesign", "Classic", {
    "Classic",
    "Rainbow",
}, false, function(currentValue131)
    bulletBeamSettings.Design = currentValue131
end)
local playerEspSection = visualsPage:Section({
    Name = "Player ESP",
    Side = 1,
})
addToggle(playerEspSection, "ESP", "ESPEnabled", false, function(currentValue132)
    playerEsp.Enabled = currentValue132
end)
addToggle(playerEspSection, "Visuals ESP", "VisualsESP", false, function(currentValue133)
    playerEsp.Enabled = currentValue133
end)
addToggle(playerEspSection, "Team Check", "ESPTeamCheck", false, function(currentValue134)
    playerEsp.TeamCheck = currentValue134
end)
addDropdown(playerEspSection, "Team Check Method", "ESPTeamMethod", "Instance", {
    "Instance",
    "Color",
    "Hide Team",
}, false, function(currentValue135)
    playerEsp.TeamCheckMethod = currentValue135
end)
local localValue22 = addToggle(playerEspSection, "Box ESP", "BoxESP", false, function(currentValue136)
    playerEsp.BoxEnabled = currentValue136
end)
addColorPicker(localValue22, "BoxESPColor", playerEsp.BoxColor, function(currentValue137)
    playerEsp.BoxColor = currentValue137
end)
addDropdown(playerEspSection, "ESP Type", "ESPTypes", {
    "Box",
}, {
    "Box",
    "Corner",
    "Fill",
}, true, function(currentValue138)
    playerEsp.BoxDesign = currentValue138
end)
addSlider(playerEspSection, "Box Thickness", "BoxThickness", 1, 5, 2, "", function(currentValue139)
    playerEsp.BoxThickness = currentValue139
end)
addSlider(playerEspSection, "Box Transparency", "BoxDrawTrans", 0, 100, 100, "%", function(currentValue140)
    playerEsp.BoxDrawTrans = currentValue140 / 100
end)
local localValue23 = addToggle(playerEspSection, "Fill ESP", "FillESP", false, function(currentValue141)
    if currentValue141 then
        playerEsp.BoxDesign = {
            "Box",
            "Fill",
        }
    end
end)
addColorPicker(localValue23, "FillESPColor", playerEsp.FillColor, function(currentValue142)
    playerEsp.FillColor = currentValue142
end)
addSlider(playerEspSection, "Fill Transparency", "FillDrawTrans", 0, 100, 30, "%", function(currentValue143)
    playerEsp.FillDrawTrans = currentValue143 / 100
end)
local localValue24 = addToggle(playerEspSection, "Health Bar", "HealthBarESP", false, function(currentValue144)
    playerEsp.HealthBar = currentValue144
end)
addColorPicker(localValue24, "HealthBarColor", playerEsp.HealthBarColor, function(currentValue145)
    playerEsp.HealthBarColor = currentValue145
end)
local localValue25 = addToggle(playerEspSection, "Health Text", "HealthTextESP", false, function(currentValue146)
    playerEsp.HealthText = currentValue146
end)
addColorPicker(localValue25, "HealthTextColor", playerEsp.HealthTextColor, function(currentValue147)
    playerEsp.HealthTextColor = currentValue147
end)
local localValue26 = addToggle(playerEspSection, "Skeleton", "VisSkelESP", false, function(currentValue148)
    playerEsp.SkeletonEnabled = currentValue148
end)
addColorPicker(localValue26, "SkeletonColor", playerEsp.SkeletonColor, function(currentValue149)
    playerEsp.SkeletonColor = currentValue149
end)
local localValue27 = addToggle(playerEspSection, "Head Dot", "HeadDotESP", false, function(currentValue150)
    playerEsp.HeadDotEnabled = currentValue150
end)
addColorPicker(localValue27, "HeadDotColor", playerEsp.HeadDotColor, function(currentValue151)
    playerEsp.HeadDotColor = currentValue151
end)
local localValue28 = addToggle(playerEspSection, "Tracer", "TracerESP", false, function(currentValue152)
    playerEsp.TracerEnabled = currentValue152
end)
addColorPicker(localValue28, "TracerESPColor", playerEsp.TracerColor, function(currentValue153)
    playerEsp.TracerColor = currentValue153
end)
addDropdown(playerEspSection, "Tracer Position", "TracerESPPos", "Bottom", {
    "Top",
    "Center",
    "Bottom",
    "Mouse",
}, false, function(currentValue154)
    playerEsp.TracerPosition = currentValue154
end)
local localValue29 = addToggle(playerEspSection, "Name", "NameESP", false, function(currentValue155)
    playerEsp.NameEnabled = currentValue155
end)
addColorPicker(localValue29, "NameESPColor", playerEsp.NameColor, function(currentValue156)
    playerEsp.NameColor = currentValue156
end)
local worldDistance = addToggle(playerEspSection, "Distance", "DistanceESP", false, function(currentValue157)
    playerEsp.DistanceEnabled = currentValue157
end)
addColorPicker(worldDistance, "DistanceESPColor", playerEsp.DistanceColor, function(currentValue158)
    playerEsp.DistanceColor = currentValue158
end)
local localValue30 = addToggle(playerEspSection, "Tool", "ToolESP", false, function(currentValue159)
    playerEsp.ToolEnabled = currentValue159
end)
addColorPicker(localValue30, "ToolESPColor", playerEsp.ToolColor, function(currentValue160)
    playerEsp.ToolColor = currentValue160
end)
local chamsSection = visualsPage:Section({
    Name = "Chams",
    Side = 2,
})
local chT = addToggle(chamsSection, "Chams", "ChamsEnabled", false, function(currentValue161)
    playerEsp.ChamsEnabled = currentValue161
end)
addColorPicker(chT, "ChamsFillColor", playerEsp.ChamsFillColor, function(currentValue162)
    playerEsp.ChamsFillColor = currentValue162
end)
addDropdown(chamsSection, "Chams Type", "ChamsType", {
    "Fill",
    "Outline",
}, {
    "Fill",
    "Outline",
}, true, function(currentValue163)
    playerEsp.ChamsType = currentValue163
end)
addSlider(chamsSection, "Chams Fill Transparency", "ChamsFillTransSlider", 0, 100, 50, "%", function(currentValue164)
    playerEsp.ChamsFillTrans = currentValue164 / 100
end)
local chamsOutlineToggle = addToggle(chamsSection, "Chams Outline", "ChamsOutline", true, function(value)
    playerEsp.ChamsOutlineEnabled = value
end)
addColorPicker(chamsOutlineToggle, "ChamsOutlineColor", playerEsp.ChamsOutlineColor, function(currentValue165)
    playerEsp.ChamsOutlineColor = currentValue165
end)
addSlider(chamsSection, "Chams Outline Transparency", "ChamsOutlineTransSlider", 0, 100, 0, "%", function(currentValue166)
    playerEsp.ChamsOutlineTrans = currentValue166 / 100
end)
addToggle(chamsSection, "Wall Check", "ESPWallCheck", false, function(currentValue167)
    playerEsp.WallCheck = currentValue167
end)
addToggle(chamsSection, "ForceShield Check", "ESPForceShieldCheck", false, function(currentValue168)
    playerEsp.ForceShieldCheck = currentValue168
end)
addToggle(chamsSection, "Use Distance", "ESPDistanceMode", false, function(currentValue169)
    playerEsp.DistanceMode = currentValue169
end)
addSlider(chamsSection, "Distance", "ESPDistanceStuds", 1, 3000, 100, "studs", function(currentValue170)
    playerEsp.DistanceStuds = currentValue170
end)
local worldEspSection = visualsPage:Section({
    Name = "World ESP",
    Side = 2,
})
addToggle(worldEspSection, "ESP CashDrop", "CashDropESPToggle", false, function(currentValue171)
    worldEspSettings.CashDrop = currentValue171
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "ESP Tools", "ToolsESPToggle", false, function(currentValue172)
    worldEspSettings.Tools = currentValue172
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "ESP Safe/Register", "SafeESP", false, function(currentValue173)
    worldEspSettings.Safe = currentValue173
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "ESP Dealer", "ESPDealer", false, function(currentValue174)
    worldEspSettings.Dealer = currentValue174
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "ESP ATM", "ESPATM", false, function(currentValue175)
    worldEspSettings.ATM = currentValue175
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "ESP Stock Dealer", "ESPStockDealerToggle", false, function(currentValue176)
    worldEspSettings.Stock = currentValue176
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "Key ESP", "KeyESPToggle", false, function(currentValue177)
    worldEspSettings.Key = currentValue177
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "Scrap ESP", "ScrapESPToggle", false, function(currentValue178)
    worldEspSettings.Scrap = currentValue178
    scanWorldEspObjects()
end)
addToggle(worldEspSection, "Highlight Exit Door", "HighlightExitDoor", false, function(currentValue179)
    utilitySettings.HighlightExit = currentValue179
    runtimeState.data.setExitDoorHighlight(currentValue179)
    scanWorldEspObjects()
end)
addButton(worldEspSection, "Refresh ESP", scanWorldEspObjects)
local visualSettingsSection = visualsPage:Section({
    Name = "Visuals",
    Side = 1,
})
local localValue31 = addToggle(visualSettingsSection, "Arms Chams", "ArmsChamsESP", false, function(currentValue180)
    armChamsSettings.Enabled = currentValue180
    updateArmChams()
end)
addColorPicker(localValue31, "ArmsChamsColor", armChamsSettings.Color, function(currentValue181)
    armChamsSettings.Color = currentValue181
    updateArmChams()
end)
addToggle(visualSettingsSection, "Custom Sky", "CustomSkyToggle", false, function(currentValue182)
    visualSettings.SkyOn = currentValue182
    setCustomSky(currentValue182)
end)
addDropdown(visualSettingsSection, "Sky Preset", "CustomSkyDropdown", "Nebula", {
    "Nebula",
    "White Galaxy",
    "Purple Nebula",
}, false, function(currentValue183)
    visualSettings.Sky = currentValue183
    if visualSettings.SkyOn then
        setCustomSky(true)
    end
end)
local localValue32 = addToggle(visualSettingsSection, "Custom Fog", "CustomFogToggle", false, function(currentValue184)
    visualSettings.FogOn = currentValue184
    setCustomFog(currentValue184)
end)
addColorPicker(localValue32, "CustomFogColor", visualSettings.FogColor, function(currentValue185)
    visualSettings.FogColor = currentValue185
    if visualSettings.FogOn then
        setCustomFog(true)
    end
end)
addSlider(visualSettingsSection, "Fog Density", "FogStrengthSlider", 0, 1000, 439, "", function(currentValue186)
    visualSettings.FogDensity = currentValue186 / 1000
end)
addToggle(visualSettingsSection, "Motion Blur", "MotionBlurToggle", false, function(currentValue187)
    visualSettings.BlurOn = currentValue187
    setCustomBlur(currentValue187)
end)
addSlider(visualSettingsSection, "Blur Strength", "BlurStrengthSlider", 0, 100, 50, "%", function(currentValue188)
    visualSettings.Blur = currentValue188 / 100
end)
addToggle(visualSettingsSection, "Custom FOV", "CustomFOVToggle", false, function(currentValue189)
    visualSettings.FovOn = currentValue189
end)
addSlider(visualSettingsSection, "FOV", "FOVSlider", 1, 120, 70, "", function(currentValue190)
    visualSettings.Fov = currentValue190
end)
addToggle(visualSettingsSection, "Custom Recoil", "CustomRecoilToggle", false, function(currentValue191)
    visualSettings.RecoilOn = currentValue191
end)
addSlider(visualSettingsSection, "Recoil Amount", "CustomRecoilSlider", 0, 100, 0, "%", function(currentValue192)
    visualSettings.Recoil = currentValue192
end)
addToggle(visualSettingsSection, "Full Bright", "FullBrightToggle", false, function(currentValue193)
    runtimeState.data.setFullBright(currentValue193)
end)
addToggle(visualSettingsSection, "Stretch Screen", "StretchScreenToggle", false, function(currentValue194)
    visualSettings.Stretch = currentValue194
end)
local movementSection = miscellaneousPage:Section({
    Name = "Fly & Speeds",
    Side = 1,
})
local flyT = addToggle(movementSection, "Fly", "FlyToggle", false, function(currentValue195)
    if currentValue195 then
        startFly()
    else
        stopFly()
    end
end)
addKeybind(flyT, "FlyKey", Enum.KeyCode.Backspace, function(currentValue196)
    if currentValue196 then
        startFly()
    else
        stopFly()
    end
end)
addSlider(movementSection, "Fly Speed", "FlySpeedSlider", 1, 300, 5, "", function(currentValue197)
    movementSettings.FlySpeed = currentValue197
end)
addDropdown(movementSection, "Fly Method", "FlyMethodDropdown", "Bypass", {
    "Bypass",
    "Normal",
}, false, function(currentValue198)
    movementSettings.FlyMethod = currentValue198
end)
local speedT = addToggle(movementSection, "Speedhack", "Speedhack", false, function(currentValue199)
    movementSettings.Speed = currentValue199
end)
addKeybind(speedT, "SpeedKey", Enum.KeyCode.LeftShift, function(currentValue200)
    movementSettings.Speed = currentValue200
end)
addSlider(movementSection, "Speed Value", "SpeedValue", 1, 300, 70, "", function(currentValue201)
    movementSettings.SpeedValue = currentValue201
end)
local jumpT = addToggle(movementSection, "Jump Power", "JumpPowerToggle", false, function(currentValue202)
    movementSettings.Jump = currentValue202
end)
addKeybind(jumpT, "JumpPowerKey", Enum.KeyCode.Space, function(currentValue203)
    movementSettings.Jump = currentValue203
end)
addSlider(movementSection, "Jump Height", "JumpPowerSlider", 1, 500, 200, "", function(currentValue204)
    movementSettings.JumpValue = currentValue204
end)
addToggle(movementSection, "Noclip", "NoclipToggle", false, function(currentValue205)
    movementSettings.Noclip = currentValue205
end)
addToggle(movementSection, "Q Teleport", "QTeleportToggle", false, function(currentValue206)
    movementSettings.QTp = currentValue206
end)
local utilitiesSection = miscellaneousPage:Section({
    Name = "Utilities",
    Side = 2,
})
addToggle(utilitiesSection, "Infinite Stamina", "InfiniteStaminaToggle", false, function(currentValue207)
    utilitySettings.InfStamina = currentValue207
end)
addToggle(utilitiesSection, "Auto Respawn", "AutoRespawnToggle", false, function(currentValue208)
    utilitySettings.AutoRespawn = currentValue208
end)
addToggle(utilitiesSection, "No Fail Lockpick", "NoFailLockpickToggle", false, function(currentValue209)
    utilitySettings.NoFail = currentValue209
end)
addToggle(utilitiesSection, "Infinite Pepper Spray", "InfinitePepper", false, function(currentValue210)
    utilitySettings.InfPepper = currentValue210
end)
addToggle(utilitiesSection, "Anti AFK", "AntiAFKToggle", false, function(currentValue211)
    utilitySettings.AntiAfk = currentValue211
end)
addToggle(utilitiesSection, "Auto Unlock Door", "AutoUnlockDoor", false, function(currentValue212)
    utilitySettings.AutoUnlock = currentValue212
end)
addToggle(utilitiesSection, "Auto Pick Up Cash", "AutoPickUpCash", false, function(currentValue213)
    utilitySettings.AutoCash = currentValue213
end)
addToggle(utilitiesSection, "Auto Break Safe/Register", "AutoBreakSafeRegister", false, function(currentValue214)
    utilitySettings.AutoSafe = currentValue214
end)
addToggle(utilitiesSection, "Auto Open Doors", "AutoOpenDoors", false, function(currentValue215)
    utilitySettings.AutoOpen = currentValue215
end)
addToggle(utilitiesSection, "Auto Close Doors", "AutoCloseDoors", false, function(currentValue216)
    utilitySettings.AutoClose = currentValue216
end)
addToggle(utilitiesSection, "No Fall", "NoFallToggle", false, function(currentValue217)
    utilitySettings.NoFall = currentValue217
end)
addToggle(utilitiesSection, "No Neck", "NoNeckToggle", false, function(currentValue218)
    runtimeState.data.setNoNeck(currentValue218)
end)
addToggle(utilitiesSection, "No Barriers", "NoBarriersToggle", false, function(currentValue219)
    utilitySettings.NoBarriers = currentValue219
end)
addToggle(utilitiesSection, "Anti Fracture", "ToggleAntiFractured", false, function(currentValue220)
    utilitySettings.AntiFracture = currentValue220
end)
addToggle(utilitiesSection, "Anti Smoke", "ToggleAntiSmoke", false, function(currentValue221)
    utilitySettings.AntiSmoke = currentValue221
end)
addToggle(utilitiesSection, "Instant Equip", "InstantEquipToggle", false, function(currentValue222)
    utilitySettings.InstantEquip = currentValue222
end)
addToggle(utilitiesSection, "Fast Pickup", "FastPickupToggle", false, function(currentValue223)
    utilitySettings.FastPickup = currentValue223
end)
addToggle(utilitiesSection, "Auto Repair", "AutoRepairToggle", false, function(currentValue224)
    utilitySettings.AutoRepair = currentValue224
end)
addToggle(utilitiesSection, "Auto Stomp", "AutoStompToggle", false, function(currentValue225)
    funSettings.Stomp = currentValue225
end)
addSlider(utilitiesSection, "Stomp Speed", "StompSpeedSlider", 1, 100, 10, "ms", function(currentValue226)
    funSettings.StompSpeed = currentValue226 / 100
end)
addToggle(utilitiesSection, "Wall Bang", "WallBangToggle", false, function(currentValue227)
    utilitySettings.WallBang = currentValue227
    runtimeState.wallBang = currentValue227
end)
local funSection = miscellaneousPage:Section({
    Name = "Fun",
    Side = 1,
})
local c4T = addToggle(funSection, "C4 Control", "C4Toggle", false, function(currentValue228)
    funSettings.C4 = currentValue228
end)
addKeybind(c4T, "C4Key", Enum.KeyCode.C, function(currentValue229)
    funSettings.C4 = currentValue229
end)
addSlider(funSection, "C4 Speed", "C4Speed", 1, 100, 10, "ms", function(currentValue230)
    funSettings.C4Speed = currentValue230 / 100
end)
addToggle(funSection, "Increase Speed", "IncreaseSpeed", false, function(currentValue231)
    funSettings.Increase = currentValue231
end)
local exT = addToggle(funSection, "Explosion Ammo", "ExplosionAmmoToggle", false, function(currentValue232)
    funSettings.Boom = currentValue232
end)
addKeybind(exT, "ExplosionAmmoKey", Enum.KeyCode.X, function(currentValue233)
    funSettings.Boom = currentValue233
end)
addSlider(funSection, "Explosion Ammo Speed", "ExplosionAmmoSpeed", 1, 100, 10, "ms", function(currentValue234)
    funSettings.BoomSpeed = currentValue234 / 100
end)
addToggle(funSection, "Pepper Spray Aura", "PepperAura", false, function(currentValue235)
    funSettings.Pepper = currentValue235
end)
addSlider(funSection, "Pepper Aura Range", "PepperAuraRange", 1, 50, 15, "studs", function(currentValue236)
    funSettings.PepperRange = currentValue236
end)
addToggle(funSection, "Hug Tool", "HugToolToggle", false, function(currentValue237)
    toyToolSettings.Hug = currentValue237
    if currentValue237 then
        createToyTool("Hug")
    else
        removeToyTool("Hug")
    end
end)
addToggle(funSection, "Carpet Tool", "CarpetToolToggle", false, function(currentValue238)
    toyToolSettings.Carpet = currentValue238
    if currentValue238 then
        createToyTool("Carpet")
    else
        removeToyTool("Carpet")
    end
end)
addToggle(funSection, "Jerk Tool", "JerkToolToggle", false, function(currentValue239)
    toyToolSettings.Jerk = currentValue239
    if currentValue239 then
        createToyTool("Jerk")
    else
        removeToyTool("Jerk")
    end
end)
addToggle(funSection, "Fake Downed Tool", "FakeDownedToolToggle", false, function(currentValue240)
    toyToolSettings.Down = currentValue240
    if currentValue240 then
        createToyTool("Fake-Downed")
    else
        removeToyTool("Fake-Downed")
    end
end)
local skinChangerSection = miscellaneousPage:Section({
    Name = "Skin Changer",
    Side = 2,
})
local skinNames = {
    "None",
}
for currentName13 in pairs(skinChanger.Map) do
    skinNames[#skinNames + 1] = currentName13
end
table.sort(skinNames)
addToggle(skinChangerSection, "Skin Changer", "SkinChangerToggle", false, function(currentValue241)
    skinChanger.On = currentValue241
    if currentValue241 then
        applySelectedSkin(getCharacter(localPlayer))
    else
        restoreOriginalSkins()
    end
end)
addDropdown(skinChangerSection, "Skins", "SkinChangerSkin", "None", skinNames, false, function(currentValue242)
    skinChanger.Name = currentValue242
    if skinChanger.On then
        restoreOriginalSkins()
        applySelectedSkin(getCharacter(localPlayer))
    end
end)
addButton(skinChangerSection, "Apply Skin", function()
    restoreOriginalSkins()
    applySelectedSkin(getCharacter(localPlayer))
    interfaceLibrary:Notification({
        Title = "Skin Applied",
        Description = skinChanger.Name,
        Duration = 2,
        Icon = "85279746515974",
    })
end)
local automationSection = miscellaneousPage:Section({
    Name = "Automation",
    Side = 1,
})
addToggle(automationSection, "Auto Pick Up Tool", "AutoPickUpTool", false, function(value)
    utilitySettings.AutoPickUpTool = value
end)
addToggle(automationSection, "Auto Refill", "AutoRefillToggle", false, function(value)
    utilitySettings.AutoRefill = value
end)
addToggle(automationSection, "Auto Deposit All", "AutoDeposit", false, function(value)
    utilitySettings.AutoDeposit = value
end)
addToggle(automationSection, "Auto Claim Allowance", "autofarmallowance", false, function(value)
    utilitySettings.AutoClaimAllowance = value
end)
addToggle(automationSection, "AutoFarm Allowance (TP)", "autofarmandnocliptoggle", false, function(value)
    utilitySettings.AutoFarmAllowance = value
end)
addToggle(automationSection, "Instant Reload", "InstantReload", false, function(value)
    utilitySettings.InstantReload = value
end)
addToggle(automationSection, "Remove Fake Glass", "RemoveFakeGlass", false, function(value)
    utilitySettings.RemoveFakeGlass = value
end)
addToggle(automationSection, "Chat Enabler", "ChatToggle", false, setChatWindowEnabled)
local teleportSection = miscellaneousPage:Section({
    Name = "Teleport",
    Side = 1,
})
addToggle(teleportSection, "CTRL + CLICK = TP", "CtrlClickTP", false, function(value)
    movementSettings.ControlClickTeleport = value
end)
addToggle(teleportSection, "Q = TP to Mouse", "QTeleportToggle", false, function(value)
    movementSettings.QTp = value
end)
local cameraSection = visualsPage:Section({
    Name = "Camera",
    Side = 2,
})
addSlider(cameraSection, "Camera Distance", "CameraZoomSlider", 10, 1000, math.floor(localPlayer.CameraMaxZoomDistance), "", function(value)
    visualSettings.CameraDistance = value
    localPlayer.CameraMaxZoomDistance = value
end)
addToggle(cameraSection, "Stretch Screen", "StretchScreenToggle", false, function(value)
    visualSettings.Stretch = value
end)
addSlider(cameraSection, "Camera Stretch Position", "StretchScreenPosition", 10, 200, 50, "", function(value)
    visualSettings.StretchPosition = (value or 50) / 100
end)
addToggle(cameraSection, "Hide Head", "HideHead", false, function(value)
    visualSettings.HideHead = value
end)
addToggle(cameraSection, "Hide Body", "HideBody", false, function(value)
    visualSettings.HideBody = value
end)
local spraySection = miscellaneousPage:Section({
    Name = "Spray Mods",
    Side = 2,
})
addToggle(spraySection, "Spray Mods", "SprayMods", false, function(value)
    spraySettings.Enabled = value
end)
addToggle(spraySection, "Check Whitelist", "SprayCheckWhitelist", false, function(value)
    spraySettings.CheckWhitelist = value
end)
addToggle(spraySection, "Check Target", "SprayCheckTarget", false, function(value)
    spraySettings.CheckTarget = value
end)
addSlider(spraySection, "Spray Range", "SprayRange", 10, 2000, 1000, "", function(value)
    spraySettings.Range = value
end)
local scrapSection = visualsPage:Section({
    Name = "Scrap ESP",
    Side = 2,
})
addToggle(scrapSection, "Scrap ESP", "ScrapESPToggle", false, function(value)
    scrapEspSettings.Enabled = value
    refreshScrapEspObjects()
end)
addSlider(scrapSection, "Scrap Distance", "ScrapDistance", 10, 5000, 1000, "m", function(value)
    scrapEspSettings.MaxDistance = value
end)
addDropdown(scrapSection, "Scrap Rarity", "ScrapRarity", "Common", {
    "Common",
    "Uncommon",
    "Rare",
    "Epic",
    "Legendary",
    "Mythic",
}, false, function(value)
    scrapEspSettings.MinimumRarity = value
end)
addDropdown(scrapSection, "Show Scrap Types", "ScrapESPTypes", {
    "Text",
}, {
    "Text",
    "Highlight",
    "Tracer",
}, true, function(value)
    scrapEspSettings.Types = {
        Text = value.Text or false,
        Highlight = value.Highlight or false,
        Tracer = value.Tracer or false,
    }
end)
addButton(scrapSection, "Refresh Scrap", refreshScrapEspObjects)
local eventSection = visualsPage:Section({
    Name = "Game Event",
    Side = 2,
})
addToggle(eventSection, "Key ESP", "KeyESPToggle", false, function(value)
    keyEspSettings.Enabled = value
    refreshKeyEspObjects()
end)
local stockSection = miscellaneousPage:Section({
    Name = "Stock Checker",
    Side = 2,
})
addToggle(stockSection, "Stock Checker", "DealerStockChecker", false, function(value)
    stockCheckerSettings.Enabled = value
    if value then
        stockCheckerSettings.LastSnapshot = getDealerStockSnapshot()
    end
end)
addDropdown(stockSection, "Select Items To Check", "SelectItemsToCheck", {
}, dealerItemNames, true, function(value)
    stockCheckerSettings.SelectedItems = {
    }
    for itemName, selected in pairs(value) do
        if selected == true then
            stockCheckerSettings.SelectedItems[itemName] = true
        elseif type(itemName) == "number" and type(selected) == "string" then
            stockCheckerSettings.SelectedItems[selected] = true
        end
    end
end)
addToggle(stockSection, "Notify New Stock", "NotifyNewStock", true, function(value)
    stockCheckerSettings.NotifyNewStock = value
end)
addToggle(stockSection, "ESP Stock Dealer", "ESPStockDealerToggle", false, function(value)
    stockCheckerSettings.DealerEsp = value
    refreshDealerStockEsp()
end)
addDropdown(stockSection, "Dealer ESP Types", "DealerESPTypes", {
    "Text",
}, {
    "Text",
    "Highlight",
    "Tracer",
}, true, function(value)
    stockCheckerSettings.EspTypes = {
        Text = value.Text or false,
        Highlight = value.Highlight or false,
        Tracer = value.Tracer or false,
    }
    refreshDealerStockEsp()
end)
addButton(stockSection, "Manual Refresh", function()
    stockCheckerSettings.LastSnapshot = getDealerStockSnapshot()
    interfaceLibrary:Notification({
        Title = "Dealer Stock",
        Description = snapshotToText(stockCheckerSettings.LastSnapshot),
        Duration = 5,
        Icon = "85279746515974",
    })
    refreshDealerStockEsp()
end)
local serverSection = miscellaneousPage:Section({
    Name = "Server",
    Side = 2,
})
addToggle(serverSection, "Admin Check", "AdminCheckToggle", false, function(currentValue243)
    administratorCheckSettings.On = currentValue243
    if currentValue243 then
        for _, currentPlayer33 in ipairs(playersService:GetPlayers()) do
            checkAdministrator(currentPlayer33)
        end
    end
end)
addToggle(serverSection, "Dealer Stock", "DealerStockChecker", false, function(currentValue244)
    dealerStockSettings.On = currentValue244
    if currentValue244 then
        dealerStockSettings.Last = ""
    end
end)
addButton(serverSection, "Check Dealer Stock", function()
    interfaceLibrary:Notification({
        Title = "Dealer Stock",
        Description = readDealerStock(),
        Duration = 5,
        Icon = "85279746515974",
    })
end)
addButton(serverSection, "Delete All Doors", function()
    for _, currentObject28 in ipairs(workspace:GetDescendants()) do
        if currentObject28.Name:lower():find("door") then
            pcall(currentObject28.Destroy, currentObject28)
        end
    end
end)
runService.Heartbeat:Connect(function()
    if utilitySettings.NoFall then
        local humanoid15 = getHumanoid(localPlayer)
        if humanoid15 then
            humanoid15:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
            humanoid15:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
        end
    end
    if utilitySettings.AntiSmoke then
        for _, currentObject29 in ipairs(workspace:GetDescendants()) do
            if currentObject29.Name == "SmokeExplosion" then
                pcall(currentObject29.Destroy, currentObject29)
            elseif currentObject29:IsA("Smoke") or currentObject29:IsA("ParticleEmitter") and currentObject29.Name:lower():find("smoke") then
                currentObject29.Enabled = false
            end
        end
    end
    if utilitySettings.InstantEquip then
        local character9 = getCharacter(localPlayer)
        local tool3 = localPlayer:FindFirstChildOfClass("Backpack")
        local humanoid16 = getHumanoid(localPlayer)
        if character9 and tool3 and humanoid16 and not character9:FindFirstChildOfClass("Tool") then
            local tool4 = tool3:FindFirstChildOfClass("Tool")
            if tool4 then
                humanoid16:EquipTool(tool4)
            end
        end
    end
end)
for _, currentPlayer34 in ipairs(playersService:GetPlayers()) do
    if currentPlayer34 ~= localPlayer then
        createPlayerEsp(currentPlayer34)
    end
end
playersService.PlayerAdded:Connect(createPlayerEsp)
watchCharacterSkins(getCharacter(localPlayer))
scanWorldEspObjects()
task.spawn(function()
    task.wait(1)
    sendExecutionReport()
end)
local recoilTables = {
}
local recoilOriginals = {
}
local genericRecoilValues = {
}
local lockpickTables = {
}
local lockpickScaleValues = {
}
runtimeState.data.processLockpickGui = function(lockpickGui, enabled)
    if not lockpickGui then
        return
    end
    local mf = lockpickGui:FindFirstChild("MF") or lockpickGui:FindFirstChild("MF", true)
    local lpFrame = mf and (mf:FindFirstChild("LP_Frame") or mf:FindFirstChild("LP_Frame", true))
    local frames = lpFrame and (lpFrame:FindFirstChild("Frames") or lpFrame:FindFirstChild("Frames", true))
    if not frames then
        frames = lockpickGui:FindFirstChild("Frames", true)
    end
    if not frames then
        return
    end
    for _, frame in ipairs(frames:GetChildren()) do
        local bar = frame:FindFirstChild("Bar") or frame:FindFirstChild("Bar", true)
        local uiScale = bar and (bar:FindFirstChild("UIScale") or bar:FindFirstChildOfClass("UIScale"))
        if uiScale then
            if enabled then
                if lockpickScaleValues[uiScale] == nil then
                    lockpickScaleValues[uiScale] = uiScale.Scale
                end
                uiScale.Scale = 10
            elseif lockpickScaleValues[uiScale] ~= nil then
                uiScale.Scale = lockpickScaleValues[uiScale]
                lockpickScaleValues[uiScale] = nil
            end
        end
    end
end
do
    local playerGui = localPlayer:FindFirstChildOfClass("PlayerGui") or localPlayer:WaitForChild("PlayerGui")
    connectSignal("lockpickGuiWatcher", playerGui.ChildAdded, function(child)
        if child.Name == "LockpickGUI" then
            task.defer(runtimeState.data.processLockpickGui, child, utilitySettings.NoFail)
        end
    end)
end
if getgc then
    pcall(function()
        for _, currentObject30 in ipairs(getgc(true)) do
            if type(currentObject30) == "table" then
                if rawget(currentObject30, "EquipTime") ~= nil and rawget(currentObject30, "Recoil") ~= nil then
                    recoilTables[#recoilTables + 1] = currentObject30
                    recoilOriginals[currentObject30] = {
                        Recoil = currentObject30.Recoil or 0,
                        AngleX_Min = currentObject30.AngleX_Min or 0,
                        AngleX_Max = currentObject30.AngleX_Max or 0,
                        AngleY_Min = currentObject30.AngleY_Min or 0,
                        AngleY_Max = currentObject30.AngleY_Max or 0,
                        AngleZ_Min = currentObject30.AngleZ_Min or 0,
                        AngleZ_Max = currentObject30.AngleZ_Max or 0,
                        RecoilSpeed = currentObject30.RecoilSpeed or 0,
                        RecoilDamper = currentObject30.RecoilDamper or 1,
                        Accuracy = currentObject30.Accuracy or 1,
                        RecoilReduction = currentObject30.RecoilReduction or 1,
                        CameraRecoilingEnabled = currentObject30.CameraRecoilingEnabled ~= false,
                    }
                else
                    for _, currentKey3 in ipairs({
                        "Recoil",
                        "CameraRecoil",
                        "Kickback",
                        "RecoilAmount",
                    }) do
                        if type(rawget(currentObject30, currentKey3)) == "number" then
                            genericRecoilValues[#genericRecoilValues + 1] = {
                                currentObject30,
                                currentKey3,
                                currentObject30[currentKey3],
                            }
                        end
                    end
                end
                if rawget(currentObject30, "Lockpick") or rawget(currentObject30, "LockPicking") or rawget(currentObject30, "PickSpeed") then
                    lockpickTables[#lockpickTables + 1] = currentObject30
                end
            end
        end
    end)
end
runService.Heartbeat:Connect(function()
    local recoilFactor = visualSettings.Recoil / 100
    for _, recoilTable in ipairs(recoilTables) do
        local original = recoilOriginals[recoilTable]
        if original then
            if visualSettings.RecoilOn then
                recoilTable.CameraRecoilingEnabled = recoilFactor > 0
                recoilTable.Recoil = original.Recoil * recoilFactor
                recoilTable.AngleX_Min = original.AngleX_Min * recoilFactor
                recoilTable.AngleX_Max = original.AngleX_Max * recoilFactor
                recoilTable.AngleY_Min = original.AngleY_Min * recoilFactor
                recoilTable.AngleY_Max = original.AngleY_Max * recoilFactor
                recoilTable.AngleZ_Min = original.AngleZ_Min * recoilFactor
                recoilTable.AngleZ_Max = original.AngleZ_Max * recoilFactor
                recoilTable.RecoilSpeed = original.RecoilSpeed * recoilFactor
                recoilTable.RecoilDamper = original.RecoilDamper
                recoilTable.Accuracy = original.Accuracy
                recoilTable.RecoilReduction = original.RecoilReduction
                if recoilTable.SprayLerp then
                    recoilTable.SprayLerp.Enabled = recoilFactor > 0
                end
            else
                recoilTable.CameraRecoilingEnabled = original.CameraRecoilingEnabled
                recoilTable.Recoil = original.Recoil
                recoilTable.AngleX_Min = original.AngleX_Min
                recoilTable.AngleX_Max = original.AngleX_Max
                recoilTable.AngleY_Min = original.AngleY_Min
                recoilTable.AngleY_Max = original.AngleY_Max
                recoilTable.AngleZ_Min = original.AngleZ_Min
                recoilTable.AngleZ_Max = original.AngleZ_Max
                recoilTable.RecoilSpeed = original.RecoilSpeed
                recoilTable.RecoilDamper = original.RecoilDamper
                recoilTable.Accuracy = original.Accuracy
                recoilTable.RecoilReduction = original.RecoilReduction
                if recoilTable.SprayLerp then
                    recoilTable.SprayLerp.Enabled = true
                end
            end
        end
    end
    for _, currentObject31 in ipairs(genericRecoilValues) do
        if visualSettings.RecoilOn then
            currentObject31[1][currentObject31[2]] = currentObject31[3] * recoilFactor
        elseif currentObject31[1][currentObject31[2]] ~= currentObject31[3] then
            currentObject31[1][currentObject31[2]] = currentObject31[3]
        end
    end
    if utilitySettings.NoFail then
        for _, currentObject32 in ipairs(lockpickTables) do
            if rawget(currentObject32, "Fail") ~= nil then
                currentObject32.Fail = false
            end
            if rawget(currentObject32, "Failed") ~= nil then
                currentObject32.Failed = false
            end
            if rawget(currentObject32, "PickSpeed") ~= nil and type(currentObject32.PickSpeed) == "number" then
                currentObject32.PickSpeed = math.max(currentObject32.PickSpeed, 100)
            end
            if rawget(currentObject32, "Success") ~= nil and type(currentObject32.Success) == "boolean" then
                currentObject32.Success = true
            end
        end
    end
    local lockpickGui = localPlayer:FindFirstChildOfClass("PlayerGui")
    lockpickGui = lockpickGui and lockpickGui:FindFirstChild("LockpickGUI")
    runtimeState.data.processLockpickGui(lockpickGui, utilitySettings.NoFail)
    if visualSettings.Stretch and not visualSettings.FovOn then
        currentCamera.FieldOfView = 100
    end
end)
local rocketObjectNames = {
    RPG_Rocket = true,
    GrenadeLauncherGrenade = true,
    AT4_Rocket = true,
    SBL_Rocket = true,
    Panzer_Rocket = true,
    FireworkLauncher_Rocket = true,
    Hallows_Rocket = true,
    Hallows_Rocket2 = true,
}
local controlledRockets = {
}
local function releaseRocket(currentObject33)
    local currentData7 = controlledRockets[currentObject33]
    if not currentData7 then
        return 
    end
    if currentData7.c then
        pcall(currentData7.c.Disconnect, currentData7.c)
    end
    if currentData7.bp then
        pcall(currentData7.bp.Destroy, currentData7.bp)
    end
    if currentData7.bv then
        pcall(currentData7.bv.Destroy, currentData7.bv)
    end
    if currentData7.bg then
        pcall(currentData7.bg.Destroy, currentData7.bg)
    end
    controlledRockets[currentObject33] = nil
    local rootPart14 = getRootPart(localPlayer)
    if rootPart14 then
        rootPart14.Anchored = false
    end
    local humanoid17 = getHumanoid(localPlayer)
    if humanoid17 then
        currentCamera.CameraSubject = humanoid17
    end
end
local function controlRocket(currentObject34)
    if not currentObject34:IsA("BasePart") then
        return
    end
    local isC4Projectile = currentObject34.Name == "TransIgnore"
    local isRocketProjectile = rocketObjectNames[currentObject34.Name] == true
    if (isC4Projectile and not funSettings.C4) or (isRocketProjectile and not (funSettings.C4 or funSettings.Boom)) or (not isC4Projectile and not isRocketProjectile) then
        return
    end
    local character = getCharacter(localPlayer)
    local rootPart15 = getRootPart(localPlayer)
    if not character or not rootPart15 then
        return
    end
    if isC4Projectile and not character:FindFirstChild("C4") then
        return
    end
    if isRocketProjectile then
        local creator = currentObject34:FindFirstChild("Creator")
        if creator and creator:IsA("ObjectValue") and creator.Value ~= localPlayer then
            return
        end
    end
    if not isC4Projectile and (currentObject34.Position - rootPart15.Position).Magnitude > 20 then
        return
    end
    if controlledRockets[currentObject34] then
        return
    end
    currentObject34.Massless = true
    if isC4Projectile or isRocketProjectile then
        for _, childName in ipairs({"BodyForce", "BodyAngularVelocity", "Sound"}) do
            local child = currentObject34:FindFirstChild(childName)
            if child then
                pcall(child.Destroy, child)
            end
        end
        local rotPart = currentObject34:FindFirstChild("RotPart")
        local rotAngularVelocity = rotPart and rotPart:FindFirstChild("BodyAngularVelocity")
        if rotAngularVelocity then
            pcall(rotAngularVelocity.Destroy, rotAngularVelocity)
        end
    end
    local bodyPosition
    local bodyVelocity
    if isC4Projectile then
        bodyVelocity = Instance.new("BodyVelocity")
        bodyVelocity.MaxForce = Vector3.new(1000000000, 1000000000, 1000000000)
        bodyVelocity.Velocity = Vector3.zero
        bodyVelocity.Parent = currentObject34
    else
        bodyPosition = Instance.new("BodyPosition")
        bodyPosition.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bodyPosition.P = 10000
        bodyPosition.D = 100
        bodyPosition.Position = currentObject34.Position
        bodyPosition.Parent = currentObject34
    end
    local bodyGyro = Instance.new("BodyGyro")
    bodyGyro.P = isC4Projectile and 90000 or 10000
    bodyGyro.D = isC4Projectile and 0 or 100
    bodyGyro.MaxTorque = Vector3.new(1000000000, 1000000000, 1000000000)
    bodyGyro.CFrame = currentObject34.CFrame
    bodyGyro.Parent = currentObject34
    rootPart15.Anchored = true
    currentCamera.CameraSubject = currentObject34
    local heartbeatConnection
    heartbeatConnection = runService.Heartbeat:Connect(function(deltaTime)
        if not currentObject34.Parent or (isC4Projectile and not funSettings.C4) or (isRocketProjectile and not (funSettings.C4 or funSettings.Boom)) then
            releaseRocket(currentObject34)
            return
        end
        local humanoid18 = getHumanoid(localPlayer)
        local moveDirection = humanoid18 and humanoid18.MoveDirection or Vector3.zero
        local speedMultiplier = isC4Projectile and math.max(funSettings.C4Speed, 0.05) or math.max(funSettings.BoomSpeed, 0.05)
        local speed = (funSettings.Increase and 350 or 150) * speedMultiplier
        local cameraFrame = currentCamera.CFrame
        bodyGyro.CFrame = cameraFrame
        if bodyVelocity then
            bodyVelocity.Velocity = (cameraFrame.LookVector + moveDirection) * speed
            currentCamera.CFrame = currentCamera.CFrame:Lerp(currentObject34.CFrame * CFrame.new(0, 1, 1) + Vector3.new(0, 5, 0), 0.1)
        elseif bodyPosition then
            bodyPosition.Position = currentObject34.Position + (cameraFrame.LookVector + moveDirection) * speed * deltaTime
        end
    end)
    controlledRockets[currentObject34] = {
        bp = bodyPosition,
        bv = bodyVelocity,
        bg = bodyGyro,
        c = heartbeatConnection,
    }
    currentObject34.AncestryChanged:Connect(function(_, parent)
        if not parent then
            releaseRocket(currentObject34)
        end
    end)
end
workspace.DescendantAdded:Connect(function(currentObject35)
    task.wait()
    if currentObject35.Name == "C4Explosion" then
        for rocket in pairs(controlledRockets) do
            if rocket.Name == "TransIgnore" then
                releaseRocket(rocket)
            end
        end
        return
    end
    pcall(controlRocket, currentObject35)
end)
end
initializeInterfaceAndLateHooks()
(function()
local recoveredState = {
    Trigger = {
        Enabled = false,
        Held = false,
        TeamCheck = false,
        FriendCheck = false,
        EnemyCheck = false,
        CheckDown = true,
        CheckForceShield = true,
        WallCheck = true,
        Part = {
            "Head",
            "HumanoidRootPart",
            "Left Hand",
            "Right Hand",
            "Left Leg",
            "Right Leg",
        },
        Method = "Hold",
        ClickMs = 40,
        LastClick = 0,
    },
    Crosshair = {
        Enabled = false,
        Design = "Cross",
        Gap = 5,
        LineLength = 8,
        CrossThickness = 2,
        CircleSize = 8,
        Spin = false,
        SpinSpeed = 90,
        CrossColor = Color3.fromRGB(0, 255, 0),
        CircleColor = Color3.fromRGB(0, 255, 0),
        Angle = 0,
    },
    PlayerStatus = {},
    SelectedPlayer = "",
    SelectedStatus = "Neutral",
    AutoLockpickSafe = false,
    FlyMobile = false,
}
local virtualInputManager
pcall(function()
    virtualInputManager = game:GetService("VirtualInputManager")
end)
local function recoveredClick()
    if recoveredState.Trigger.Method == "Hold" and mouse1press and mouse1release then
        pcall(mouse1press)
        task.wait(recoveredState.Trigger.ClickMs / 1000)
        pcall(mouse1release)
        return
    end
    if mouse1click then
        pcall(mouse1click)
        return
    end
    if mouse1press and mouse1release then
        pcall(mouse1press)
        pcall(mouse1release)
        return
    end
    if virtualInputManager then
        local p = userInputService:GetMouseLocation()
        pcall(function()
            virtualInputManager:SendMouseButtonEvent(p.X, p.Y, 0, true, game, 0)
            task.wait(recoveredState.Trigger.Method == "Hold" and recoveredState.Trigger.ClickMs / 1000 or 0)
            virtualInputManager:SendMouseButtonEvent(p.X, p.Y, 0, false, game, 0)
        end)
    end
end
local function recoveredTriggerTarget()
    local center = currentCamera.ViewportSize * 0.5
    local ray = currentCamera:ViewportPointToRay(center.X, center.Y)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {
        getCharacter(localPlayer),
        currentCamera,
    }
    params.IgnoreWater = true
    local result = workspace:Raycast(ray.Origin, ray.Direction * 5000, params)
    if not result or not result.Instance then
        return
    end
    local model = result.Instance:FindFirstAncestorOfClass("Model")
    local player = model and playersService:GetPlayerFromCharacter(model)
    if not player then
        return
    end
    if shouldIgnorePlayer(player, {
        CheckTeam = recoveredState.Trigger.TeamCheck,
        FriendCheck = recoveredState.Trigger.FriendCheck,
        EnemyCheck = recoveredState.Trigger.EnemyCheck,
        CheckDown = recoveredState.Trigger.CheckDown,
        CheckForceShield = recoveredState.Trigger.CheckForceShield,
    }) then
        return
    end
    if recoveredState.Trigger.WallCheck and not hasLineOfSight(result.Instance) then
        return
    end
    local selectedParts = recoveredState.Trigger.Part
    if type(selectedParts) == "table" then
        local selected = selectedParts[result.Instance.Name] == true
        if not selected then
            for _, partName in pairs(selectedParts) do
                if partName == result.Instance.Name then
                    selected = true
                    break
                end
            end
        end
        if not selected then
            return
        end
    elseif selectedParts ~= result.Instance.Name then
        return
    end
    if tick() - recoveredState.Trigger.LastClick < recoveredState.Trigger.ClickMs / 1000 then
        return
    end
    recoveredState.Trigger.LastClick = tick()
    recoveredClick()
end
runService.RenderStepped:Connect(function()
    if recoveredState.Trigger.Enabled and recoveredState.Trigger.Held then
        recoveredTriggerTarget()
    end
end)
local recoveredCrosshairLines = {
    createDrawingObject("Line", false, 1000),
    createDrawingObject("Line", false, 1000),
    createDrawingObject("Line", false, 1000),
    createDrawingObject("Line", false, 1000),
}
local recoveredCrosshairCircle = createDrawingObject("Circle", false, 1000)
runService.RenderStepped:Connect(function(deltaTime)
    local cfg = recoveredState.Crosshair
    local center = userInputService:GetMouseLocation()
    if cfg.Spin then
        cfg.Angle = cfg.Angle + cfg.SpinSpeed * deltaTime
    end
    local angle = math.rad(cfg.Angle)
    local directions = {
        Vector2.new(math.cos(angle), math.sin(angle)),
        Vector2.new(math.cos(angle + math.pi), math.sin(angle + math.pi)),
        Vector2.new(math.cos(angle + math.pi * 0.5), math.sin(angle + math.pi * 0.5)),
        Vector2.new(math.cos(angle + math.pi * 1.5), math.sin(angle + math.pi * 1.5)),
    }
    for i, line in ipairs(recoveredCrosshairLines) do
        local direction = directions[i]
        line.From = center + direction * cfg.Gap
        line.To = center + direction * (cfg.Gap + cfg.LineLength)
        line.Color = cfg.CrossColor
        line.Thickness = cfg.CrossThickness
        line.Visible = cfg.Enabled and (cfg.Design == "Cross" or cfg.Design == "Combined")
    end
    recoveredCrosshairCircle.Position = center
    recoveredCrosshairCircle.Radius = cfg.CircleSize
    recoveredCrosshairCircle.Color = cfg.CircleColor
    recoveredCrosshairCircle.Thickness = cfg.CrossThickness
    recoveredCrosshairCircle.Visible = cfg.Enabled and (cfg.Design == "Circle" or cfg.Design == "Combined")
end)
local rageMobileGui = Instance.new("ScreenGui")
rageMobileGui.Name = "JX_RageMobile"
rageMobileGui.IgnoreGuiInset = true
rageMobileGui.ResetOnSpawn = false
rageMobileGui.Enabled = false
rageMobileGui.Parent = coreGui
local rageMobileButton = Instance.new("TextButton")
rageMobileButton.AnchorPoint = Vector2.new(0.5, 0.5)
rageMobileButton.Position = UDim2.new(0.78, 0, 0.72, 0)
rageMobileButton.Size = UDim2.fromOffset(72, 72)
rageMobileButton.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
rageMobileButton.BackgroundTransparency = 0.15
rageMobileButton.Text = "RAGE"
rageMobileButton.TextColor3 = Color3.fromRGB(255, 255, 255)
rageMobileButton.TextScaled = true
rageMobileButton.Font = Enum.Font.GothamBold
rageMobileButton.Parent = rageMobileGui
local rageMobileCorner = Instance.new("UICorner")
rageMobileCorner.CornerRadius = UDim.new(1, 0)
rageMobileCorner.Parent = rageMobileButton
rageMobileButton.MouseButton1Click:Connect(function()
    rageBot.Enabled = not rageBot.Enabled
    rageMobileButton.BackgroundColor3 = rageBot.Enabled and Color3.fromRGB(120, 20, 20) or Color3.fromRGB(20, 20, 20)
    if rageBot.Enabled then
        startRageBot()
    else
        stopRageBot()
    end
end)
local flyMobileGui = Instance.new("ScreenGui")
flyMobileGui.Name = "JX_FlyMobile"
flyMobileGui.IgnoreGuiInset = true
flyMobileGui.ResetOnSpawn = false
flyMobileGui.Enabled = false
flyMobileGui.Parent = coreGui
local flyMobileButton = Instance.new("TextButton")
flyMobileButton.AnchorPoint = Vector2.new(0.5, 0.5)
flyMobileButton.Position = UDim2.new(0.2, 0, 0.72, 0)
flyMobileButton.Size = UDim2.fromOffset(64, 64)
flyMobileButton.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
flyMobileButton.BackgroundTransparency = 0.15
flyMobileButton.Text = "FLY"
flyMobileButton.TextColor3 = Color3.fromRGB(255, 255, 255)
flyMobileButton.TextScaled = true
flyMobileButton.Font = Enum.Font.GothamBold
flyMobileButton.Parent = flyMobileGui
local flyMobileCorner = Instance.new("UICorner")
flyMobileCorner.CornerRadius = UDim.new(1, 0)
flyMobileCorner.Parent = flyMobileButton
flyMobileButton.MouseButton1Click:Connect(function()
    if movementSettings.Flying then
        stopFly()
        flyMobileButton.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    else
        startFly()
        flyMobileButton.BackgroundColor3 = Color3.fromRGB(120, 20, 20)
    end
end)
local function recoveredInventoryText(player)
    if not player then
        return "Player not found"
    end
    local names = {}
    local seen = {}
    local function scan(container)
        if not container then
            return
        end
        for _, object in ipairs(container:GetChildren()) do
            if object:IsA("Tool") and not seen[object.Name] then
                seen[object.Name] = true
                names[#names + 1] = object.Name
            end
        end
    end
    scan(player:FindFirstChildOfClass("Backpack"))
    scan(player.Character)
    table.sort(names)
    return #names > 0 and table.concat(names, "\n") or "Empty"
end
runtimeState.data.playerStatus = recoveredState.PlayerStatus
local function recoveredSafePrompt(prompt)
    if not recoveredState.AutoLockpickSafe or not prompt:IsA("ProximityPrompt") then
        return
    end
    local parent = prompt.Parent
    local model = parent and parent:FindFirstAncestorOfClass("Model")
    local objectName = ((model and model.Name) or (parent and parent.Name) or ""):lower()
    if not objectName:find("safe") and not objectName:find("register") then
        return
    end
    local root = getRootPart(localPlayer)
    local part = prompt.Parent
    if part and part:IsA("Attachment") then
        part = part.Parent
    end
    if root and part and part:IsA("BasePart") and (part.Position - root.Position).Magnitude <= prompt.MaxActivationDistance + 5 then
        pcall(function()
            prompt.HoldDuration = 0
            if fireproximityprompt then
                fireproximityprompt(prompt)
            end
        end)
    end
end
startTask("recoveredAutoLockpick", function()
    while task.wait(0.2) do
        if recoveredState.AutoLockpickSafe then
            for _, object in ipairs(workspace:GetDescendants()) do
                if object:IsA("ProximityPrompt") then
                    recoveredSafePrompt(object)
                end
            end
        end
    end
end)
local function latestSetFromMulti(value)
    local result = {}
    if type(value) ~= "table" then
        return result
    end
    for key, item in pairs(value) do
        if type(key) == "number" and type(item) == "string" then
            result[item] = true
        elseif type(key) == "string" and item == true then
            result[key] = true
        end
    end
    return result
end
local function latestHas(values, item)
    if type(values) ~= "table" then
        return false
    end
    if values[item] == true then
        return true
    end
    for _, value in pairs(values) do
        if value == item then
            return true
        end
    end
    return false
end
local latestEsp = {
    Enabled = false,
    TeamCheck = false,
    UseStatusColor = true,
    FriendColor = Color3.fromRGB(0, 255, 0),
    EnemyColor = Color3.fromRGB(255, 0, 0),
    StatusVisualApply = latestSetFromMulti({"Box", "Chams", "Text", "Tracer Line", "Skeleton", "Arrow"}),
    UseDistance = false,
    MaxDistance = 160,
    PlayerVisualLimit = 20,
    VisualHertz = 60,
    VisibleCheck = false,
    VisibleFunction = {},
    FontSize = 11,
    BoxEnabled = false,
    BoxRGB = Color3.fromRGB(255, 255, 255),
    BoxType = "Full",
    BoxFilledEnabled = false,
    BoxFilledRGB = Color3.fromRGB(0, 0, 0),
    BoxFilledTransparency = 0.6,
    BoxDesign = "Default",
    ChamsEnabled = false,
    ChamsOutlineRGB = Color3.fromRGB(234, 37, 37),
    ChamsOutlineTransparency = 0,
    ChamsFillEnabled = false,
    ChamsFillRGB = Color3.fromRGB(214, 35, 35),
    ChamsFillTransparency = 0.5,
    ChamsThermal = false,
    HealthbarEnabled = false,
    HealthbarRGB = Color3.fromRGB(0, 255, 0),
    HealthbarWidth = 2,
    HealthbarType = "Default",
    AmmoBarEnabled = false,
    AmmoBarRGB = Color3.fromRGB(0, 140, 255),
    AmmoBarWidth = 2,
    NamesEnabled = false,
    NamesRGB = Color3.fromRGB(255, 255, 255),
    DistancesEnabled = false,
    DistancesRGB = Color3.fromRGB(255, 255, 255),
    DistancesPosition = "Bottom",
    ToolEnabled = false,
    ToolRGB = Color3.fromRGB(255, 255, 0),
    ToolType = "Always",
    ToolExtra = "Name",
    HealthTextEnabled = false,
    HealthTextRGB = Color3.fromRGB(255, 255, 255),
    HealthTextType = "One",
    StatusEnabled = false,
    ActivityEnabled = false,
    SkeletonEnabled = false,
    SkeletonRGB = Color3.fromRGB(0, 0, 0),
    SkeletonFilled = false,
    SkeletonFilledRGB = Color3.fromRGB(255, 0, 0),
    TracerEnabled = false,
    TracerRGB = Color3.fromRGB(255, 255, 255),
    TracerPosition = "Bottom",
    TracerTarget = "Bottom",
    WeaponImageEnabled = false,
    ArrowEnabled = false,
    ArrowRGB = Color3.fromRGB(255, 255, 255),
    MeterTextEnabled = true,
    NearColorEnabled = true,
    NearColorRGB = Color3.fromRGB(255, 0, 0),
    NearDistance = 20,
    ArrowRadius = 100,
    EnemyChamsEnabled = false,
    EnemyChamsColor = Color3.fromRGB(255, 0, 0),
    FriendChamsEnabled = false,
    FriendChamsColor = Color3.fromRGB(0, 255, 0),
    LastUpdate = 0,
}
local latestEspFolder = Instance.new("Folder")
latestEspFolder.Name = "JX_LatestESP"
latestEspFolder.Parent = coreGui
local latestEspObjects = {}
local latestSkeletonPairs = {
    {"Head", "UpperTorso"},
    {"UpperTorso", "LowerTorso"},
    {"UpperTorso", "LeftUpperArm"},
    {"LeftUpperArm", "LeftLowerArm"},
    {"LeftLowerArm", "LeftHand"},
    {"UpperTorso", "RightUpperArm"},
    {"RightUpperArm", "RightLowerArm"},
    {"RightLowerArm", "RightHand"},
    {"LowerTorso", "LeftUpperLeg"},
    {"LeftUpperLeg", "LeftLowerLeg"},
    {"LeftLowerLeg", "LeftFoot"},
    {"LowerTorso", "RightUpperLeg"},
    {"RightUpperLeg", "RightLowerLeg"},
    {"RightLowerLeg", "RightFoot"},
}
local function latestDrawing(kind, filled, zindex)
    local object = Drawing.new(kind)
    object.Visible = false
    object.Transparency = 1
    object.ZIndex = zindex or 20
    if kind == "Square" then
        object.Filled = filled or false
        object.Thickness = 1
        object.Color = Color3.new(1, 1, 1)
    elseif kind == "Line" then
        object.Thickness = 1
        object.Color = Color3.new(1, 1, 1)
    elseif kind == "Text" then
        object.Center = true
        object.Outline = true
        object.Font = 2
        object.Size = 11
        object.Color = Color3.new(1, 1, 1)
    elseif kind == "Triangle" then
        object.Filled = true
        object.Thickness = 1
        object.Color = Color3.new(1, 1, 1)
    end
    return object
end
local function latestCreateEsp(player)
    if player == localPlayer or latestEspObjects[player] then
        return
    end
    local drawings = {
        box = latestDrawing("Square", false, 24),
        fill = latestDrawing("Square", true, 20),
        healthBg = latestDrawing("Line", false, 23),
        health = latestDrawing("Line", false, 24),
        ammoBg = latestDrawing("Line", false, 23),
        ammo = latestDrawing("Line", false, 24),
        name = latestDrawing("Text", false, 25),
        distance = latestDrawing("Text", false, 25),
        tool = latestDrawing("Text", false, 25),
        healthText = latestDrawing("Text", false, 25),
        status = latestDrawing("Text", false, 25),
        activity = latestDrawing("Text", false, 25),
        meter = latestDrawing("Text", false, 25),
        weapon = latestDrawing("Text", false, 25),
        tracer = latestDrawing("Line", false, 22),
        arrow = latestDrawing("Triangle", true, 25),
    }
    local corners = {}
    for index = 1, 8 do
        corners[index] = latestDrawing("Line", false, 24)
        drawings["corner" .. index] = corners[index]
    end
    local skeleton = {}
    for index = 1, #latestSkeletonPairs do
        skeleton[index] = latestDrawing("Line", false, 23)
        drawings["skeleton" .. index] = skeleton[index]
    end
    local highlight = Instance.new("Highlight")
    highlight.Name = "JX_Latest_" .. player.Name
    highlight.Enabled = false
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = latestEspFolder
    highlight.Adornee = player.Character
    local characterConnection = player.CharacterAdded:Connect(function(character)
        highlight.Adornee = character
    end)
    latestEspObjects[player] = {
        drawings = drawings,
        corners = corners,
        skeleton = skeleton,
        highlight = highlight,
        connection = characterConnection,
    }
end
local function latestRemoveEsp(player)
    local data = latestEspObjects[player]
    if not data then
        return
    end
    for _, object in pairs(data.drawings) do
        pcall(function()
            object:Remove()
        end)
    end
    pcall(function()
        data.highlight:Destroy()
    end)
    pcall(function()
        data.connection:Disconnect()
    end)
    latestEspObjects[player] = nil
end
local function latestHideEsp(data)
    if not data then
        return
    end
    for _, object in pairs(data.drawings) do
        object.Visible = false
    end
    data.highlight.Enabled = false
end
local function latestPlayerStatus(player)
    local selected = recoveredState.PlayerStatus[player.Name]
    if selected then
        return selected
    end
    if latestEsp.TeamCheck and player.Team and localPlayer.Team and player.Team == localPlayer.Team then
        return "Friend"
    end
    local friend = false
    pcall(function()
        friend = player:IsFriendsWith(localPlayer.UserId)
    end)
    return friend and "Friend" or "Enemy"
end
local function latestStatusColor(player, distance)
    if latestEsp.NearColorEnabled and distance <= latestEsp.NearDistance then
        return latestEsp.NearColorRGB
    end
    local status = latestPlayerStatus(player)
    if status == "Friend" then
        return latestEsp.FriendColor
    end
    return latestEsp.EnemyColor
end
local function latestVisualColor(baseColor, statusColor, visualName)
    if latestEsp.UseStatusColor and latestHas(latestEsp.StatusVisualApply, visualName) then
        return statusColor
    end
    return baseColor
end
local function latestGetAmmo(tool)
    if not tool then
        return 0, 0
    end
    local values = tool:FindFirstChild("Values")
    local current = values and (values:FindFirstChild("SERVER_Ammo") or values:FindFirstChild("Ammo") or values:FindFirstChild("StoredAmmo"))
    local maximum = values and (values:FindFirstChild("MagSize") or values:FindFirstChild("MaxAmmo") or values:FindFirstChild("StoredAmmo"))
    local currentValue = current and tonumber(current.Value) or 0
    local maximumValue = maximum and tonumber(maximum.Value) or math.max(currentValue, 1)
    if maximumValue <= 0 then
        maximumValue = math.max(currentValue, 1)
    end
    return currentValue, maximumValue
end
local function latestActivity(humanoid, character)
    if not humanoid then
        return "Idle"
    end
    if character and character:GetAttribute("Downed") then
        return "Downed"
    end
    if humanoid.Health <= 0 then
        return "Dead"
    end
    if humanoid.MoveDirection.Magnitude > 0.05 then
        return "Moving"
    end
    if humanoid.FloorMaterial == Enum.Material.Air then
        return "Air"
    end
    return "Idle"
end
local function latestSetLine(line, from, to, color, visible, thickness)
    line.From = from
    line.To = to
    line.Color = color
    line.Thickness = thickness or 1
    line.Visible = visible
end
local function latestSetCorners(lines, position, size, color, visible)
    local x = position.X
    local y = position.Y
    local w = size.X
    local h = size.Y
    local lx = w * 0.28
    local ly = h * 0.22
    latestSetLine(lines[1], Vector2.new(x, y), Vector2.new(x + lx, y), color, visible, 1)
    latestSetLine(lines[2], Vector2.new(x, y), Vector2.new(x, y + ly), color, visible, 1)
    latestSetLine(lines[3], Vector2.new(x + w, y), Vector2.new(x + w - lx, y), color, visible, 1)
    latestSetLine(lines[4], Vector2.new(x + w, y), Vector2.new(x + w, y + ly), color, visible, 1)
    latestSetLine(lines[5], Vector2.new(x, y + h), Vector2.new(x + lx, y + h), color, visible, 1)
    latestSetLine(lines[6], Vector2.new(x, y + h), Vector2.new(x, y + h - ly), color, visible, 1)
    latestSetLine(lines[7], Vector2.new(x + w, y + h), Vector2.new(x + w - lx, y + h), color, visible, 1)
    latestSetLine(lines[8], Vector2.new(x + w, y + h), Vector2.new(x + w, y + h - ly), color, visible, 1)
end
local function latestTracerOrigin()
    local viewport = currentCamera.ViewportSize
    if latestEsp.TracerPosition == "Top" then
        return Vector2.new(viewport.X * 0.5, 0)
    elseif latestEsp.TracerPosition == "Center" then
        return viewport * 0.5
    elseif latestEsp.TracerPosition == "Mouse" then
        return userInputService:GetMouseLocation()
    end
    return Vector2.new(viewport.X * 0.5, viewport.Y)
end
local function latestArrow(data, projected, statusColor, visible)
    local arrow = data.drawings.arrow
    if not visible then
        arrow.Visible = false
        return
    end
    local viewport = currentCamera.ViewportSize
    local center = viewport * 0.5
    local delta = Vector2.new(projected.X, projected.Y) - center
    if projected.Z < 0 then
        delta = -delta
    end
    if delta.Magnitude < 1 then
        delta = Vector2.new(0, -1)
    else
        delta = delta.Unit
    end
    local radius = math.min(latestEsp.ArrowRadius, math.min(viewport.X, viewport.Y) * 0.45)
    local point = center + delta * radius
    local side = Vector2.new(-delta.Y, delta.X)
    arrow.PointA = point + delta * 10
    arrow.PointB = point - delta * 8 + side * 7
    arrow.PointC = point - delta * 8 - side * 7
    arrow.Color = latestVisualColor(latestEsp.ArrowRGB, statusColor, "Arrow")
    arrow.Visible = true
end
local function latestUpdateEsp(player, data, allowed)
    if not latestEsp.Enabled or not allowed then
        latestHideEsp(data)
        return
    end
    local character = getCharacter(player)
    local humanoid = getHumanoid(player)
    local root = getRootPart(player)
    local head = character and character:FindFirstChild("Head")
    local myRoot = getRootPart(localPlayer)
    if not character or not humanoid or humanoid.Health <= 0 or not root or not head then
        latestHideEsp(data)
        return
    end
    if latestEsp.TeamCheck and player.Team and localPlayer.Team and player.Team == localPlayer.Team and recoveredState.PlayerStatus[player.Name] ~= "Enemy" then
        latestHideEsp(data)
        return
    end
    local distance = myRoot and (root.Position - myRoot.Position).Magnitude or math.huge
    if latestEsp.UseDistance and distance > latestEsp.MaxDistance then
        latestHideEsp(data)
        return
    end
    local rootProjection, onScreen = currentCamera:WorldToViewportPoint(root.Position)
    local statusColor = latestStatusColor(player, distance)
    latestArrow(data, rootProjection, statusColor, latestEsp.ArrowEnabled and not onScreen)
    if not onScreen or rootProjection.Z <= 0 then
        for key, object in pairs(data.drawings) do
            if key ~= "arrow" then
                object.Visible = false
            end
        end
        data.highlight.Enabled = false
        return
    end
    data.drawings.arrow.Visible = false
    local visible = hasLineOfSight(root)
    local boxesVisible = not latestEsp.VisibleCheck or visible or not latestHas(latestEsp.VisibleFunction, "Boxes")
    local chamsVisible = not latestEsp.VisibleCheck or visible or not latestHas(latestEsp.VisibleFunction, "Chams")
    local textVisible = not latestEsp.VisibleCheck or visible or not latestHas(latestEsp.VisibleFunction, "Text")
    local topProjection = currentCamera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.75, 0))
    local bottomProjection = currentCamera:WorldToViewportPoint(root.Position - Vector3.new(0, 3.1, 0))
    local height = math.max(12, math.abs(topProjection.Y - bottomProjection.Y))
    local width = height * 0.58
    local position = Vector2.new(rootProjection.X - width * 0.5, rootProjection.Y - height * 0.5)
    local size = Vector2.new(width, height)
    local boxColor = latestVisualColor(latestEsp.BoxRGB, statusColor, "Box")
    data.drawings.box.Position = position
    data.drawings.box.Size = size
    data.drawings.box.Color = boxColor
    data.drawings.box.Visible = latestEsp.BoxEnabled and latestEsp.BoxType == "Full" and boxesVisible
    data.drawings.fill.Position = position
    data.drawings.fill.Size = size
    data.drawings.fill.Color = latestEsp.BoxFilledRGB
    data.drawings.fill.Transparency = 1 - latestEsp.BoxFilledTransparency
    data.drawings.fill.Visible = latestEsp.BoxFilledEnabled and boxesVisible
    latestSetCorners(data.corners, position, size, boxColor, latestEsp.BoxEnabled and latestEsp.BoxType == "Corner" and boxesVisible)
    local healthPercent = math.clamp(humanoid.Health / math.max(humanoid.MaxHealth, 1), 0, 1)
    local healthColor = latestEsp.HealthbarRGB
    if latestEsp.HealthbarType == "Gradient" then
        healthColor = Color3.fromRGB(255, 0, 0):Lerp(Color3.fromRGB(0, 255, 0), healthPercent)
    end
    latestSetLine(data.drawings.healthBg, Vector2.new(position.X - 5, position.Y + size.Y), Vector2.new(position.X - 5, position.Y), Color3.new(0, 0, 0), latestEsp.HealthbarEnabled, latestEsp.HealthbarWidth + 2)
    latestSetLine(data.drawings.health, Vector2.new(position.X - 5, position.Y + size.Y), Vector2.new(position.X - 5, position.Y + size.Y * (1 - healthPercent)), healthColor, latestEsp.HealthbarEnabled, latestEsp.HealthbarWidth)
    local tool = character:FindFirstChildOfClass("Tool")
    local ammo, maxAmmo = latestGetAmmo(tool)
    local ammoPercent = math.clamp(ammo / math.max(maxAmmo, 1), 0, 1)
    latestSetLine(data.drawings.ammoBg, Vector2.new(position.X, position.Y + size.Y + 4), Vector2.new(position.X + size.X, position.Y + size.Y + 4), Color3.new(0, 0, 0), latestEsp.AmmoBarEnabled, latestEsp.AmmoBarWidth + 2)
    latestSetLine(data.drawings.ammo, Vector2.new(position.X, position.Y + size.Y + 4), Vector2.new(position.X + size.X * ammoPercent, position.Y + size.Y + 4), latestEsp.AmmoBarRGB, latestEsp.AmmoBarEnabled, latestEsp.AmmoBarWidth)
    local topOffset = -15
    local bottomOffset = latestEsp.AmmoBarEnabled and 12 or 5
    local textColor = latestVisualColor(latestEsp.NamesRGB, statusColor, "Text")
    data.drawings.name.Text = player.Name
    if latestEsp.DistancesEnabled and latestEsp.DistancesPosition == "Text" then
        data.drawings.name.Text = data.drawings.name.Text .. " [" .. tostring(math.floor(distance)) .. "m]"
    end
    data.drawings.name.Position = Vector2.new(position.X + size.X * 0.5, position.Y + topOffset)
    data.drawings.name.Size = latestEsp.FontSize
    data.drawings.name.Color = textColor
    data.drawings.name.Visible = latestEsp.NamesEnabled and textVisible
    data.drawings.distance.Text = tostring(math.floor(distance)) .. "m"
    data.drawings.distance.Position = Vector2.new(position.X + size.X * 0.5, position.Y + size.Y + bottomOffset)
    data.drawings.distance.Size = latestEsp.FontSize
    data.drawings.distance.Color = latestVisualColor(latestEsp.DistancesRGB, statusColor, "Text")
    data.drawings.distance.Visible = latestEsp.DistancesEnabled and latestEsp.DistancesPosition == "Bottom" and textVisible
    if data.drawings.distance.Visible then
        bottomOffset = bottomOffset + latestEsp.FontSize + 2
    end
    local toolText = tool and tool.Name or "None"
    if latestEsp.ToolExtra == "Combine" and tool then
        toolText = toolText .. " [" .. tostring(ammo) .. "/" .. tostring(maxAmmo) .. "]"
    end
    data.drawings.tool.Text = toolText
    data.drawings.tool.Position = Vector2.new(position.X + size.X * 0.5, position.Y + size.Y + bottomOffset)
    data.drawings.tool.Size = latestEsp.FontSize
    data.drawings.tool.Color = latestVisualColor(latestEsp.ToolRGB, statusColor, "Text")
    data.drawings.tool.Visible = latestEsp.ToolEnabled and textVisible and (latestEsp.ToolType == "Always" or tool ~= nil)
    if data.drawings.tool.Visible then
        bottomOffset = bottomOffset + latestEsp.FontSize + 2
    end
    data.drawings.healthText.Text = latestEsp.HealthTextType == "Double" and (tostring(math.floor(humanoid.Health)) .. "/" .. tostring(math.floor(humanoid.MaxHealth))) or tostring(math.floor(humanoid.Health))
    data.drawings.healthText.Position = Vector2.new(position.X - 20, position.Y + size.Y * (1 - healthPercent))
    data.drawings.healthText.Size = latestEsp.FontSize
    data.drawings.healthText.Color = latestVisualColor(latestEsp.HealthTextRGB, statusColor, "Text")
    data.drawings.healthText.Visible = latestEsp.HealthTextEnabled and textVisible
    data.drawings.status.Text = latestPlayerStatus(player)
    data.drawings.status.Position = Vector2.new(position.X + size.X + 6, position.Y + latestEsp.FontSize)
    data.drawings.status.Center = false
    data.drawings.status.Size = latestEsp.FontSize
    data.drawings.status.Color = latestVisualColor(statusColor, statusColor, "Text")
    data.drawings.status.Visible = latestEsp.StatusEnabled and textVisible
    data.drawings.activity.Text = "Activity: " .. latestActivity(humanoid, character)
    data.drawings.activity.Position = Vector2.new(position.X + size.X + 6, position.Y + latestEsp.FontSize * 2.3)
    data.drawings.activity.Center = false
    data.drawings.activity.Size = latestEsp.FontSize
    data.drawings.activity.Color = latestVisualColor(Color3.fromRGB(255, 255, 255), statusColor, "Text")
    data.drawings.activity.Visible = latestEsp.ActivityEnabled and textVisible
    data.drawings.meter.Text = tostring(math.floor(distance)) .. " studs"
    data.drawings.meter.Position = Vector2.new(position.X + size.X * 0.5, position.Y + size.Y + bottomOffset)
    data.drawings.meter.Size = latestEsp.FontSize
    data.drawings.meter.Color = latestVisualColor(Color3.fromRGB(255, 255, 255), statusColor, "Text")
    data.drawings.meter.Visible = latestEsp.MeterTextEnabled and textVisible
    if data.drawings.meter.Visible then
        bottomOffset = bottomOffset + latestEsp.FontSize + 2
    end
    data.drawings.weapon.Text = tool and ("[" .. tool.Name .. "]") or ""
    data.drawings.weapon.Position = Vector2.new(position.X + size.X * 0.5, position.Y + size.Y + bottomOffset)
    data.drawings.weapon.Size = latestEsp.FontSize
    data.drawings.weapon.Color = latestVisualColor(latestEsp.ToolRGB, statusColor, "Text")
    data.drawings.weapon.Visible = latestEsp.WeaponImageEnabled and tool ~= nil and textVisible
    local tracerTarget
    if latestEsp.TracerTarget == "Top" then
        tracerTarget = Vector2.new(position.X + size.X * 0.5, position.Y)
    elseif latestEsp.TracerTarget == "Middle" then
        tracerTarget = Vector2.new(position.X + size.X * 0.5, position.Y + size.Y * 0.5)
    else
        tracerTarget = Vector2.new(position.X + size.X * 0.5, position.Y + size.Y)
    end
    latestSetLine(data.drawings.tracer, latestTracerOrigin(), tracerTarget, latestVisualColor(latestEsp.TracerRGB, statusColor, "Tracer Line"), latestEsp.TracerEnabled, 1)
    for index, pair in ipairs(latestSkeletonPairs) do
        local first = character:FindFirstChild(pair[1])
        local second = character:FindFirstChild(pair[2])
        local show = false
        local firstScreen
        local secondScreen
        if first and second then
            local a, aVisible = currentCamera:WorldToViewportPoint(first.Position)
            local b, bVisible = currentCamera:WorldToViewportPoint(second.Position)
            show = aVisible and bVisible and a.Z > 0 and b.Z > 0 and latestEsp.SkeletonEnabled
            firstScreen = Vector2.new(a.X, a.Y)
            secondScreen = Vector2.new(b.X, b.Y)
        end
        local color = latestVisualColor(latestEsp.SkeletonFilled and latestEsp.SkeletonFilledRGB or latestEsp.SkeletonRGB, statusColor, "Skeleton")
        if show then
            latestSetLine(data.skeleton[index], firstScreen, secondScreen, color, true, latestEsp.SkeletonFilled and 3 or 1)
        else
            data.skeleton[index].Visible = false
        end
    end
    local status = latestPlayerStatus(player)
    local statusChams = status == "Friend" and latestEsp.FriendChamsEnabled or status ~= "Friend" and latestEsp.EnemyChamsEnabled
    local statusChamsColor = status == "Friend" and latestEsp.FriendChamsColor or latestEsp.EnemyChamsColor
    local useChams = (latestEsp.ChamsEnabled or statusChams) and chamsVisible
    local fillColor = statusChams and statusChamsColor or latestVisualColor(latestEsp.ChamsFillRGB, statusColor, "Chams")
    local outlineColor = statusChams and statusChamsColor or latestVisualColor(latestEsp.ChamsOutlineRGB, statusColor, "Chams")
    if latestEsp.ChamsThermal then
        fillColor = Color3.fromRGB(255, math.floor(healthPercent * 255), 0)
        outlineColor = Color3.fromRGB(255, 255, 255)
    end
    data.highlight.Adornee = character
    data.highlight.FillColor = fillColor
    data.highlight.OutlineColor = outlineColor
    data.highlight.FillTransparency = latestEsp.ChamsFillEnabled or statusChams and 0.45 or 1
    data.highlight.OutlineTransparency = latestEsp.ChamsOutlineTransparency
    if latestEsp.ChamsFillEnabled then
        data.highlight.FillTransparency = latestEsp.ChamsFillTransparency
    end
    data.highlight.Enabled = useChams
end
for _, player in ipairs(playersService:GetPlayers()) do
    latestCreateEsp(player)
end
playersService.PlayerAdded:Connect(latestCreateEsp)
playersService.PlayerRemoving:Connect(latestRemoveEsp)
runService.RenderStepped:Connect(function()
    local now = os.clock()
    local interval = 1 / math.max(latestEsp.VisualHertz, 1)
    if now - latestEsp.LastUpdate < interval then
        return
    end
    latestEsp.LastUpdate = now
    local myRoot = getRootPart(localPlayer)
    local ordered = {}
    for player, data in pairs(latestEspObjects) do
        local root = getRootPart(player)
        ordered[#ordered + 1] = {
            player = player,
            data = data,
            distance = myRoot and root and (root.Position - myRoot.Position).Magnitude or math.huge,
        }
    end
    table.sort(ordered, function(a, b)
        return a.distance < b.distance
    end)
    for index, item in ipairs(ordered) do
        pcall(latestUpdateEsp, item.player, item.data, index <= latestEsp.PlayerVisualLimit)
    end
end)
local latestArms = {
    ArmsChamsEnabled = false,
    ArmsChamsColor = Color3.fromRGB(255, 0, 0),
    ForceFieldEnabled = false,
    ForceFieldColor = Color3.fromRGB(255, 0, 0),
    ForceFieldTransparency = 0.15,
    WeaponChamsEnabled = false,
    WeaponColor = Color3.fromRGB(255, 0, 0),
}
local latestArmOriginal = setmetatable({}, {__mode = "k"})
local function latestRememberPart(part)
    if not latestArmOriginal[part] then
        latestArmOriginal[part] = {
            Color = part.Color,
            Material = part.Material,
            Transparency = part.Transparency,
        }
    end
end
local function latestRestorePart(part)
    local old = latestArmOriginal[part]
    if old and part.Parent then
        part.Color = old.Color
        part.Material = old.Material
        part.Transparency = old.Transparency
        latestArmOriginal[part] = nil
    end
end
local function latestUpdateArms()
    local character = getCharacter(localPlayer)
    local tool = character and character:FindFirstChildOfClass("Tool")
    local active = {}
    local containers = {character, currentCamera}
    for _, container in ipairs(containers) do
        if container then
            for _, object in ipairs(container:GetDescendants()) do
                if object:IsA("BasePart") then
                    local lower = object.Name:lower()
                    local arm = lower:find("arm") or lower:find("hand")
                    local weapon = tool and object:IsDescendantOf(tool)
                    if arm and (latestArms.ArmsChamsEnabled or latestArms.ForceFieldEnabled) then
                        latestRememberPart(object)
                        active[object] = true
                        if latestArms.ForceFieldEnabled then
                            object.Material = Enum.Material.ForceField
                            object.Color = latestArms.ForceFieldColor
                            object.Transparency = latestArms.ForceFieldTransparency
                        else
                            object.Material = Enum.Material.Neon
                            object.Color = latestArms.ArmsChamsColor
                            object.Transparency = 0
                        end
                    elseif weapon and latestArms.WeaponChamsEnabled then
                        latestRememberPart(object)
                        active[object] = true
                        object.Material = Enum.Material.Neon
                        object.Color = latestArms.WeaponColor
                        object.Transparency = 0
                    end
                end
            end
        end
    end
    for part in pairs(latestArmOriginal) do
        if not active[part] then
            latestRestorePart(part)
        end
    end
end
runService.RenderStepped:Connect(latestUpdateArms)
local latestSounds = {
    HitEnabled = false,
    HitType = "XP",
    KillEnabled = false,
    KillType = "EZ",
}
local latestHitSoundIds = {
    Neverlose = "rbxassetid://8726881116",
    Bubble = "rbxassetid://198598793",
    Minecraft = "rbxassetid://4018616850",
    Gamesense = "rbxassetid://4817809188",
    Rust = "rbxassetid://5043539486",
    Teddy = "rbxassetid://377904055",
    XP = "rbxassetid://1053296915",
    Pop = "rbxassetid://9118828564",
}
local latestKillSoundIds = {
    EZ = "rbxassetid://6349641063",
    Money = "rbxassetid://1240516814",
}
local function latestPlaySound(id)
    if not id then
        return
    end
    local sound = Instance.new("Sound")
    sound.SoundId = id
    sound.Volume = 1
    sound.Parent = game:GetService("SoundService")
    sound:Play()
    game:GetService("Debris"):AddItem(sound, 5)
end
local latestHitmarkerConnections = setmetatable({}, {__mode = "k"})
local function latestBindHitmarker(tool)
    if not tool or not tool:IsA("Tool") or latestHitmarkerConnections[tool] then
        return
    end
    local marker = tool:FindFirstChild("Hitmarker", true)
    if not marker then
        return
    end
    local signal
    pcall(function()
        signal = marker.Event
    end)
    if not signal or not signal.Connect then
        return
    end
    latestHitmarkerConnections[tool] = signal:Connect(function(target)
        if latestSounds.HitEnabled then
            latestPlaySound(latestHitSoundIds[latestSounds.HitType])
        end
        if latestSounds.KillEnabled and typeof(target) == "Instance" then
            local model = target:FindFirstAncestorOfClass("Model")
            local humanoid = model and model:FindFirstChildOfClass("Humanoid")
            task.delay(0.05, function()
                if humanoid and humanoid.Health <= 0 then
                    latestPlaySound(latestKillSoundIds[latestSounds.KillType])
                end
            end)
        end
    end)
end
local function latestScanHitmarkers()
    local character = getCharacter(localPlayer)
    local backpack = localPlayer:FindFirstChildOfClass("Backpack")
    for _, container in ipairs({character, backpack}) do
        if container then
            for _, object in ipairs(container:GetChildren()) do
                if object:IsA("Tool") then
                    latestBindHitmarker(object)
                end
            end
        end
    end
end
startTask("latestHitmarkerScan", function()
    while task.wait(1) do
        latestScanHitmarkers()
    end
end)
local latestExitDoorHighlight
local function latestSetExitDoor(enabled)
    if latestExitDoorHighlight then
        latestExitDoorHighlight:Destroy()
        latestExitDoorHighlight = nil
    end
    if not enabled then
        return
    end
    local map = workspace:FindFirstChild("Map")
    local squid = map and map:FindFirstChild("SquidDirectory", true)
    if not squid then
        return
    end
    local candidate
    for _, object in ipairs(squid:GetDescendants()) do
        local lower = object.Name:lower()
        if lower:find("exit") or lower:find("door") then
            candidate = object:IsA("Model") and object or object:FindFirstAncestorOfClass("Model") or object
            break
        end
    end
    if candidate then
        latestExitDoorHighlight = Instance.new("Highlight")
        latestExitDoorHighlight.Name = "JX_ExitDoor"
        latestExitDoorHighlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        latestExitDoorHighlight.FillColor = Color3.fromRGB(0, 255, 0)
        latestExitDoorHighlight.OutlineColor = Color3.fromRGB(255, 255, 255)
        latestExitDoorHighlight.FillTransparency = 0.35
        latestExitDoorHighlight.Adornee = candidate
        latestExitDoorHighlight.Parent = coreGui
    end
end
local latestDealerColors = {
    Tracer = Color3.fromRGB(255, 255, 0),
    Highlight = Color3.fromRGB(255, 255, 0),
    Text = Color3.fromRGB(255, 255, 0),
}
runService.RenderStepped:Connect(function()
    for _, data in pairs(stockCheckerSettings.EspObjects) do
        if data.Tracer then
            data.Tracer.Color = latestDealerColors.Tracer
        end
        if data.Highlight then
            data.Highlight.FillColor = latestDealerColors.Highlight
            data.Highlight.OutlineColor = latestDealerColors.Highlight
        end
        if data.Billboard then
            local label = data.Billboard:FindFirstChildOfClass("TextLabel")
            if label then
                label.TextColor3 = latestDealerColors.Text
            end
        end
    end
end)
local function latestFpsBoost()
    for _, object in ipairs(game:GetDescendants()) do
        pcall(function()
            if object:IsA("ParticleEmitter") or object:IsA("Trail") or object:IsA("Beam") or object:IsA("Smoke") or object:IsA("Fire") or object:IsA("Sparkles") then
                object.Enabled = false
            elseif object:IsA("Decal") or object:IsA("Texture") then
                object.Transparency = 1
            elseif object:IsA("BasePart") then
                object.Material = Enum.Material.SmoothPlastic
                object.Reflectance = 0
                object.CastShadow = false
            elseif object:IsA("PostEffect") then
                object.Enabled = false
            end
        end)
    end
    lighting.GlobalShadows = false
    lighting.FogEnd = 100000
end
local latestDealerItems = {
    "Corruptis",
    "X24",
    "Relic",
    "SoulContract",
    "_FallenBlade",
    "__NecromancerKit",
    "__SlayerKit",
    "MonsterMashPotion",
    "_CopeCoin26",
    "Scythe",
    "ERADICATOR",
    "SlayerArmour",
    "SlayerSword",
}
mainWindow:Category("Latest")
local latestVisualPage = mainWindow:Page({Name = "Visuals+", Icon = "138827881557940"})
local latestCombatPage = mainWindow:Page({Name = "Combat+", Icon = "138827881557940"})
local latestPlayerPage = mainWindow:Page({Name = "Players+", Icon = "138827881557940"})
local latestMiscPage = mainWindow:Page({Name = "Misc+", Icon = "138827881557940"})
local latestCharacterPage = mainWindow:Page({Name = "Character+", Icon = "138827881557940"})
local latestEspMain = latestVisualPage:Section({Name = "Main", Side = 1})
addToggle(latestEspMain, "ESP Enabled", "ESPEnabled", false, function(value)
    latestEsp.Enabled = value
    playerEsp.Enabled = false
end)
addToggle(latestEspMain, "Team Check", "ESPTeamCheck", false, function(value)
    latestEsp.TeamCheck = value
end)
local latestStatusColorToggle = addToggle(latestEspMain, "Use Status Color", "ESPUseStatusColor", true, function(value)
    latestEsp.UseStatusColor = value
end)
addColorPicker(latestStatusColorToggle, "ESPFriendColor", latestEsp.FriendColor, function(value)
    latestEsp.FriendColor = value
end)
addColorPicker(latestStatusColorToggle, "ESPEnemyColor", latestEsp.EnemyColor, function(value)
    latestEsp.EnemyColor = value
end)
addDropdown(latestEspMain, "Status Visual Apply", "ESPStatusVisualApply", {"Box", "Chams", "Text", "Tracer Line", "Skeleton", "Arrow"}, {"Box", "Chams", "Text", "Tracer Line", "Skeleton", "Arrow"}, true, function(value)
    latestEsp.StatusVisualApply = latestSetFromMulti(value)
end)
addToggle(latestEspMain, "Use Distance", "ESPUseDistance", false, function(value)
    latestEsp.UseDistance = value
end)
addSlider(latestEspMain, "Max Distance", "ESPMaxDistance", 50, 1000, 160, "m", function(value)
    latestEsp.MaxDistance = value
end)
addSlider(latestEspMain, "Player Visual Limit", "ESPPlayerLimit", 1, 200, 20, " players", function(value)
    latestEsp.PlayerVisualLimit = value
end)
addSlider(latestEspMain, "Visual Hertz", "ESPVisualHertz", 1, 60, 60, " hz", function(value)
    latestEsp.VisualHertz = value
end)
addToggle(latestEspMain, "Visible Check", "ESPVisibleCheck", false, function(value)
    latestEsp.VisibleCheck = value
end)
addDropdown(latestEspMain, "Visible Function", "ESPVisibleFn", {}, {"Boxes", "Chams", "Text"}, true, function(value)
    latestEsp.VisibleFunction = latestSetFromMulti(value)
end)
addSlider(latestEspMain, "Font Size", "ESPFontSize", 8, 24, 11, "px", function(value)
    latestEsp.FontSize = value
end)
addButton(latestEspMain, "FPS Boost", latestFpsBoost)
local latestBoxes = latestVisualPage:Section({Name = "Boxes", Side = 1})
local latestBoxToggle = addToggle(latestBoxes, "Box", "ESPBoxEnabled", false, function(value)
    latestEsp.BoxEnabled = value
end)
addColorPicker(latestBoxToggle, "ESPBoxColor", latestEsp.BoxRGB, function(value)
    latestEsp.BoxRGB = value
end)
addDropdown(latestBoxes, "Box Type", "ESPBoxType", "Full", {"Full", "Corner"}, false, function(value)
    latestEsp.BoxType = value
end)
local latestFilledToggle = addToggle(latestBoxes, "Filled Box", "ESPBoxFilled", false, function(value)
    latestEsp.BoxFilledEnabled = value
end)
addColorPicker(latestFilledToggle, "ESPBoxFilledColor", latestEsp.BoxFilledRGB, function(value)
    latestEsp.BoxFilledRGB = value
end)
addSlider(latestBoxes, "Fill Transparency", "ESPBoxFillTrans", 0, 100, 60, "%", function(value)
    latestEsp.BoxFilledTransparency = value / 100
end)
addDropdown(latestBoxes, "Box Design", "ESPBoxDesign", "Default", {"Default", "Gradient"}, false, function(value)
    latestEsp.BoxDesign = value
end)
local latestChams = latestVisualPage:Section({Name = "Chams", Side = 1})
local latestChamsToggle = addToggle(latestChams, "Chams", "ESPChamsEnabled", false, function(value)
    latestEsp.ChamsEnabled = value
end)
addColorPicker(latestChamsToggle, "ESPChamsOutColor", latestEsp.ChamsOutlineRGB, function(value)
    latestEsp.ChamsOutlineRGB = value
end)
addSlider(latestChams, "Outline Transparency", "ESPChamsOutTrans", 0, 100, 0, "%", function(value)
    latestEsp.ChamsOutlineTransparency = value / 100
end)
local latestFillChamsToggle = addToggle(latestChams, "Filled Chams", "ESPChamsFill", false, function(value)
    latestEsp.ChamsFillEnabled = value
end)
addColorPicker(latestFillChamsToggle, "ESPChamsFillColor", latestEsp.ChamsFillRGB, function(value)
    latestEsp.ChamsFillRGB = value
end)
addSlider(latestChams, "Fill Transparency", "ESPChamsFillTrans", 0, 100, 50, "%", function(value)
    latestEsp.ChamsFillTransparency = value / 100
end)
addToggle(latestChams, "Thermal Effect", "ESPChamsThermal", false, function(value)
    latestEsp.ChamsThermal = value
end)
local latestArmsToggle = addToggle(latestChams, "Arms Chams", "ESPArmsChams", false, function(value)
    latestArms.ArmsChamsEnabled = value
    armChamsSettings.Enabled = value
end)
addColorPicker(latestArmsToggle, "ESPArmsChamsColor", latestArms.ArmsChamsColor, function(value)
    latestArms.ArmsChamsColor = value
    armChamsSettings.Color = value
end)
local latestForceToggle = addToggle(latestChams, "ForceField", "ESPArmsForceField", false, function(value)
    latestArms.ForceFieldEnabled = value
end)
addColorPicker(latestForceToggle, "ESPArmsForceFieldColor", latestArms.ForceFieldColor, function(value)
    latestArms.ForceFieldColor = value
end)
addSlider(latestChams, "ForceField Transparency", "ESPArmsForceFieldTrans", 0, 100, 15, "%", function(value)
    latestArms.ForceFieldTransparency = value / 100
end)
local latestWeaponChamsToggle = addToggle(latestChams, "Weapon Chams", "ESPArmsWeaponChams", false, function(value)
    latestArms.WeaponChamsEnabled = value
end)
addColorPicker(latestWeaponChamsToggle, "ESPArmsWeaponChamsColor", latestArms.WeaponColor, function(value)
    latestArms.WeaponColor = value
end)
local latestBars = latestVisualPage:Section({Name = "Bar", Side = 2})
local latestHealthBarToggle = addToggle(latestBars, "Health Bar", "ESPHealthbar", false, function(value)
    latestEsp.HealthbarEnabled = value
end)
addColorPicker(latestHealthBarToggle, "ESPHealthbarColor", latestEsp.HealthbarRGB, function(value)
    latestEsp.HealthbarRGB = value
end)
addSlider(latestBars, "Bar Width", "ESPHealthWidth", 1, 10, 2, "", function(value)
    latestEsp.HealthbarWidth = value
end)
addDropdown(latestBars, "Health Bar Type", "ESPHealthType", "Default", {"Default", "Gradient"}, false, function(value)
    latestEsp.HealthbarType = value
end)
local latestAmmoToggle = addToggle(latestBars, "Ammo Bar", "ESPAmmoBar", false, function(value)
    latestEsp.AmmoBarEnabled = value
end)
addColorPicker(latestAmmoToggle, "ESPAmmoBarColor", latestEsp.AmmoBarRGB, function(value)
    latestEsp.AmmoBarRGB = value
end)
addSlider(latestBars, "Ammo Bar Width", "ESPAmmoBarWidth", 1, 10, 2, "", function(value)
    latestEsp.AmmoBarWidth = value
end)
local latestText = latestVisualPage:Section({Name = "Text", Side = 2})
local latestNamesToggle = addToggle(latestText, "Names", "ESPNames", false, function(value)
    latestEsp.NamesEnabled = value
end)
addColorPicker(latestNamesToggle, "ESPNamesColor", latestEsp.NamesRGB, function(value)
    latestEsp.NamesRGB = value
end)
local latestDistanceToggle = addToggle(latestText, "Distances", "ESPDistances", false, function(value)
    latestEsp.DistancesEnabled = value
end)
addColorPicker(latestDistanceToggle, "ESPDistancesColor", latestEsp.DistancesRGB, function(value)
    latestEsp.DistancesRGB = value
end)
addDropdown(latestText, "Distance Position", "ESPDistPos", "Bottom", {"Text", "Bottom"}, false, function(value)
    latestEsp.DistancesPosition = value
end)
local latestToolToggle = addToggle(latestText, "Tool", "ESPTool", false, function(value)
    latestEsp.ToolEnabled = value
end)
addColorPicker(latestToolToggle, "ESPToolColor", latestEsp.ToolRGB, function(value)
    latestEsp.ToolRGB = value
end)
addDropdown(latestText, "Tool Type", "ESPToolType", "Always", {"Always", "Equip"}, false, function(value)
    latestEsp.ToolType = value
end)
addDropdown(latestText, "Tool Extra", "ESPToolExtra", "Name", {"Name", "Combine"}, false, function(value)
    latestEsp.ToolExtra = value
end)
local latestHealthTextToggle = addToggle(latestText, "Health", "ESPHealthTextTgl", false, function(value)
    latestEsp.HealthTextEnabled = value
end)
addColorPicker(latestHealthTextToggle, "ESPHealthTextColor", latestEsp.HealthTextRGB, function(value)
    latestEsp.HealthTextRGB = value
end)
addDropdown(latestText, "Health Text Type", "ESPHealthTextType", "One", {"One", "Double"}, false, function(value)
    latestEsp.HealthTextType = value
end)
addToggle(latestText, "Status", "ESPStatus", false, function(value)
    latestEsp.StatusEnabled = value
end)
addToggle(latestText, "Activity", "ESPActivity", false, function(value)
    latestEsp.ActivityEnabled = value
end)
local latestOther = latestVisualPage:Section({Name = "Other", Side = 2})
local latestSkeletonToggle = addToggle(latestOther, "Skeleton", "ESPSkeleton", false, function(value)
    latestEsp.SkeletonEnabled = value
end)
addColorPicker(latestSkeletonToggle, "ESPSkeletonColor", latestEsp.SkeletonRGB, function(value)
    latestEsp.SkeletonRGB = value
end)
local latestFilledSkeletonToggle = addToggle(latestOther, "Filled Skeleton", "ESPSkeletonFilled", false, function(value)
    latestEsp.SkeletonFilled = value
end)
addColorPicker(latestFilledSkeletonToggle, "ESPSkeletonFilledColor", latestEsp.SkeletonFilledRGB, function(value)
    latestEsp.SkeletonFilledRGB = value
end)
local latestTracerToggle = addToggle(latestOther, "Tracer Line", "ESPTracer", false, function(value)
    latestEsp.TracerEnabled = value
end)
addColorPicker(latestTracerToggle, "ESPTracerColor", latestEsp.TracerRGB, function(value)
    latestEsp.TracerRGB = value
end)
addDropdown(latestOther, "Line Position", "ESPTracerPos", "Bottom", {"Bottom", "Top", "Center", "Mouse"}, false, function(value)
    latestEsp.TracerPosition = value
end)
addDropdown(latestOther, "To Player Position", "ESPTracerTarget", "Bottom", {"Bottom", "Top", "Middle"}, false, function(value)
    latestEsp.TracerTarget = value
end)
addToggle(latestOther, "Weapon Image", "ESPWeaponImage", false, function(value)
    latestEsp.WeaponImageEnabled = value
end)
local latestArrowToggle = addToggle(latestOther, "Arrow", "ESPArrow", false, function(value)
    latestEsp.ArrowEnabled = value
end)
addColorPicker(latestArrowToggle, "ESPArrowColor", latestEsp.ArrowRGB, function(value)
    latestEsp.ArrowRGB = value
end)
addToggle(latestOther, "Meter Text", "ESPMeterText", true, function(value)
    latestEsp.MeterTextEnabled = value
end)
local latestNearToggle = addToggle(latestOther, "Change Color If Near", "ESPNearColorToggle", true, function(value)
    latestEsp.NearColorEnabled = value
end)
addColorPicker(latestNearToggle, "ESPNearColor", latestEsp.NearColorRGB, function(value)
    latestEsp.NearColorRGB = value
end)
addSlider(latestOther, "Near Distance", "ESPNearDistance", 1, 100, 20, "m", function(value)
    latestEsp.NearDistance = value
end)
addSlider(latestOther, "Arrow Radius", "ESPArrowRadius", 25, 250, 100, "", function(value)
    latestEsp.ArrowRadius = value
end)
local latestCombatExtra = latestCombatPage:Section({Name = "Aimbot Extra", Side = 1})
addToggle(latestCombatExtra, "Check ForceShield", "AimbotForceShield", true, function(value)
    desktopAimbot.CheckForceShield = value
end)
addToggle(latestCombatExtra, "Mobile Prediction", "AimbotMobilePredictionOn", false, function(value)
    mobileAimbot.PredictionOn = value
end)
addSlider(latestCombatExtra, "Mobile Prediction X", "AimbotMobilePredictionX", 0, 100, 15, "", function(value)
    mobileAimbot.PredictionX = value
end)
addSlider(latestCombatExtra, "Mobile Prediction Y", "AimbotMobilePredictionY", 0, 100, 15, "", function(value)
    mobileAimbot.PredictionY = value
end)
addToggle(latestCombatExtra, "Mobile Check ForceShield", "AimbotMobileForceShield", true, function(value)
    mobileAimbot.CheckForceShield = value
end)
addToggle(latestCombatExtra, "Beast Mode", "RBBeastMode", false, function(value)
    rageBot.BeastMode = value
end)
addToggle(latestCombatExtra, "Silent Aim Check ForceShield", "SilentAimForceShieldCheck", true, function(value)
    silentAimV1.CheckForceShield = value
    silentAimV2.CheckForceShield = value
end)
addToggle(latestCombatExtra, "Sticky Aim", "AimbotStickyAim", false, function(value)
    desktopAimbot.StickyAim = value
end)
addToggle(latestCombatExtra, "Full Lock", "AimbotFullLock", false, function(value)
    desktopAimbot.FullLock = value
end)
addToggle(latestCombatExtra, "Tween Speed", "AimbotTweenSpeedOn", false, function(value)
    desktopAimbot.TweenSpeedOn = value
end)
addSlider(latestCombatExtra, "Tween Speed Amount", "AimbotTweenSpeed", 1, 100, 50, "%", function(value)
    desktopAimbot.TweenSpeed = value
end)
local latestDesktopLockColor = addToggle(latestCombatExtra, "Locked FOV Color", "AimbotLockedColorToggle", true, function() end)
addColorPicker(latestDesktopLockColor, "AimbotFOVLockedColor", Color3.fromRGB(0, 255, 0), function(value)
    desktopAimbot.LockedColor = value
end)
addToggle(latestCombatExtra, "Friend Check", "AimbotFriendCheck", false, function(value)
    desktopAimbot.FriendCheck = value
end)
addToggle(latestCombatExtra, "Enemy Check", "AimbotEnemyCheck", false, function(value)
    desktopAimbot.EnemyCheck = value
end)
addToggle(latestCombatExtra, "Mobile Sticky Aim", "AimbotMobileStickyAim", false, function(value)
    mobileAimbot.StickyAim = value
end)
addToggle(latestCombatExtra, "Mobile Full Lock", "AimbotMobileFullLock", false, function(value)
    mobileAimbot.FullLock = value
end)
addToggle(latestCombatExtra, "Mobile Tween Speed", "AimbotMobileTweenSpeedOn", false, function(value)
    mobileAimbot.TweenSpeedOn = value
end)
addSlider(latestCombatExtra, "Mobile Tween Speed Amount", "AimbotMobileTweenSpeed", 1, 100, 50, "%", function(value)
    mobileAimbot.TweenSpeed = value
end)
local latestMobileLockColor = addToggle(latestCombatExtra, "Mobile Locked FOV Color", "AimbotMobileLockedColorToggle", true, function() end)
addColorPicker(latestMobileLockColor, "AimbotMobileFOVLockedColor", Color3.fromRGB(0, 255, 0), function(value)
    mobileAimbot.LockedColor = value
end)
addToggle(latestCombatExtra, "Mobile Friend Check", "AimbotMobileFriendCheck", false, function(value)
    mobileAimbot.FriendCheck = value
end)
addToggle(latestCombatExtra, "Mobile Enemy Check", "AimbotMobileEnemyCheck", false, function(value)
    mobileAimbot.EnemyCheck = value
end)
local latestRageLockColor = addToggle(latestCombatExtra, "Rage Locked Color", "RageBotLockedColorToggle", true, function() end)
addColorPicker(latestRageLockColor, "RageBotLockedColor", Color3.fromRGB(0, 255, 0), function(value)
    rageBot.LockedColor = value
end)
addColorPicker(latestRageLockColor, "SilentAimV1FOVFillColor", Color3.fromRGB(175, 86, 86), function(value)
    silentAimV1.FOVFillColor = value
end)
addSlider(latestCombatExtra, "Random Change Time", "SilentAimRandomTime", 1, 50, 30, "", function(value)
    silentAimV2.RandomInterval = value / 10
end)
addColorPicker(latestRageLockColor, "SilentAimFOVFillColor", Color3.fromRGB(175, 86, 86), function(value)
    silentAimV2.FOVFillColor = value
end)
addSlider(latestCombatExtra, "Melee Random Change Time", "MeleeAuraRandomTime", 1, 50, 30, "", function(value)
    meleeAura.RandomInterval = value / 10
end)
local latestStatusChams = latestCombatPage:Section({Name = "Status Chams", Side = 2})
local latestEnemyChamsToggle = addToggle(latestStatusChams, "Enemy Chams", "EnemyChamsToggle", false, function(value)
    latestEsp.EnemyChamsEnabled = value
end)
addColorPicker(latestEnemyChamsToggle, "EnemyChamsColor", latestEsp.EnemyChamsColor, function(value)
    latestEsp.EnemyChamsColor = value
end)
local latestFriendChamsToggle = addToggle(latestStatusChams, "Friend Chams", "FriendChamsToggle", false, function(value)
    latestEsp.FriendChamsEnabled = value
end)
addColorPicker(latestFriendChamsToggle, "FriendChamsColor", latestEsp.FriendChamsColor, function(value)
    latestEsp.FriendChamsColor = value
end)
local latestTrigger = latestCombatPage:Section({Name = "Trigger Bot", Side = 2})
addToggle(latestTrigger, "Trigger Bot", "TriggerBotEnabled", false, function(value)
    recoveredState.Trigger.Enabled = value
end)
local latestTriggerKey = addToggle(latestTrigger, "Trigger Active", "TriggerBotToggle", false, function(value)
    recoveredState.Trigger.Held = value
end)
addKeybind(latestTriggerKey, "TriggerBotToggleKey", Enum.KeyCode.Backspace, function(value)
    recoveredState.Trigger.Held = value
end)
addKeybind(latestTriggerKey, "TriggerBotKey", Enum.UserInputType.MouseButton2, function(value)
    recoveredState.Trigger.Held = value
end)
addDropdown(latestTrigger, "Trigger Part", "TriggerBotPart", {"Head", "HumanoidRootPart", "Left Hand", "Right Hand", "Left Leg", "Right Leg"}, {"Head", "HumanoidRootPart", "Left Hand", "Right Hand", "Left Leg", "Right Leg"}, true, function(value)
    recoveredState.Trigger.Part = value
end)
addDropdown(latestTrigger, "Trigger Method", "TriggerBotMethod", "Hold", {"Hold", "Click"}, false, function(value)
    recoveredState.Trigger.Method = value
end)
addSlider(latestTrigger, "Click Ms", "TriggerBotClickMs", 1, 250, 40, " ms", function(value)
    recoveredState.Trigger.ClickMs = value
end)
addToggle(latestTrigger, "Team Check", "TriggerBotTeamCheck", false, function(value)
    recoveredState.Trigger.TeamCheck = value
end)
addToggle(latestTrigger, "Friend Check", "TriggerBotFriendCheck", false, function(value)
    recoveredState.Trigger.FriendCheck = value
end)
addToggle(latestTrigger, "Enemy Check", "TriggerBotEnemyCheck", false, function(value)
    recoveredState.Trigger.EnemyCheck = value
end)
addToggle(latestTrigger, "Check Down", "TriggerBotCheckDown", true, function(value)
    recoveredState.Trigger.CheckDown = value
end)
addToggle(latestTrigger, "Check ForceShield", "TriggerBotForceShield", true, function(value)
    recoveredState.Trigger.CheckForceShield = value
end)
addToggle(latestTrigger, "Wall Check", "TriggerBotWallCheck", true, function(value)
    recoveredState.Trigger.WallCheck = value
end)
local latestPlayers = latestPlayerPage:Section({Name = "Player Information", Side = 1})
local latestPlayerList
local latestInventoryList
if latestPlayers.Listbox then
    latestPlayerList = latestPlayers:Listbox({Name = "Select Player", Flag = "PlayerManagerSelect", Default = "", Items = getOtherPlayerNames(), MaxSize = 500, Callback = function(value)
        recoveredState.SelectedPlayer = value
        local selected = playersService:FindFirstChild(value)
        if latestInventoryList and latestInventoryList.SetItems then
            local items = {}
            local backpack = selected and selected:FindFirstChildOfClass("Backpack")
            if backpack then
                for _, object in ipairs(backpack:GetChildren()) do
                    if object:IsA("Tool") then
                        items[#items + 1] = object.Name
                    end
                end
            end
            if #items == 0 then
                items[1] = "Empty"
            end
            latestInventoryList:SetItems(items)
        end
    end})
    latestInventoryList = latestPlayers:Listbox({Name = "Inventory", Flag = "PlayerInventoryViewer", Items = {"Empty"}, MaxSize = 150})
else
    latestPlayerList = addDropdown(latestPlayers, "Select Player", "PlayerManagerSelect", "", getOtherPlayerNames(), false, function(value)
        recoveredState.SelectedPlayer = value
    end)
end
addButton(latestPlayers, "Refresh List", function()
    local values = getOtherPlayerNames()
    if latestPlayerList and latestPlayerList.SetItems then
        latestPlayerList:SetItems(values)
    end
end)
addButton(latestPlayers, "Copy User ID", function()
    local player = playersService:FindFirstChild(recoveredState.SelectedPlayer)
    if player and setclipboard then
        setclipboard(tostring(player.UserId))
    end
end)
addButton(latestPlayers, "Spectate Player", function()
    spectateSettings.Name = recoveredState.SelectedPlayer
    spectateSettings.On = true
    startSpectating()
end)
local latestPlayerStatusSection = latestPlayerPage:Section({Name = "Status", Side = 2})
addDropdown(latestPlayerStatusSection, "Status List", "PlayerStatusSelect", "Neutral", {"Neutral", "Enemy", "Friend"}, false, function(value)
    recoveredState.SelectedStatus = value
end)
addButton(latestPlayerStatusSection, "Set Status", function()
    if recoveredState.SelectedPlayer ~= "" then
        recoveredState.PlayerStatus[recoveredState.SelectedPlayer] = recoveredState.SelectedStatus
    end
end)
addButton(latestPlayerStatusSection, "Reset All", function()
    table.clear(recoveredState.PlayerStatus)
end)
local latestBullet = latestMiscPage:Section({Name = "Bullet Tracer", Side = 1})
local latestBeamToggle = addToggle(latestBullet, "Bullet Beam", "BulletBeamEnabled", false, function(value)
    bulletBeamSettings.On = value
    if value then
        runtimeState.data.setupBulletTracerConnections()
    else
        runtimeState.data.clearBulletTracerConnections()
    end
end)
addColorPicker(latestBeamToggle, "BulletBeamColor", bulletBeamSettings.Col, function(value)
    bulletBeamSettings.Col = value
end)
addDropdown(latestBullet, "Bullet Beam Design", "BulletBeamDesign", "Wave", {"Classic", "Rainbow", "Wave"}, false, function(value)
    bulletBeamSettings.Design = value
end)
addSlider(latestBullet, "BulletBeam Thickness", "BulletBeamThickness", 0, 20, 20, "", function(value)
    bulletBeamSettings.Thick = value / 10
end)
addSlider(latestBullet, "BulletBeam Lifetime", "BulletBeamLifetime", 0, 10, 2, "s", function(value)
    bulletBeamSettings.Life = value
end)
addSlider(latestBullet, "BulletBeam Transparency", "BulletBeamTransparency", 0, 100, 65, "%", function(value)
    bulletBeamSettings.Trans = value / 100
end)
local latestSound = latestMiscPage:Section({Name = "Custom Sound", Side = 1})
addToggle(latestSound, "Hit Sound", "HitSoundEnabled", false, function(value)
    latestSounds.HitEnabled = value
end)
addDropdown(latestSound, "Sound Type", "HitSoundType", "XP", {"Neverlose", "Bubble", "Minecraft", "Gamesense", "Rust", "Teddy", "XP", "Pop"}, false, function(value)
    latestSounds.HitType = value
end)
addToggle(latestSound, "Kill Sound", "KillSoundEnabled", false, function(value)
    latestSounds.KillEnabled = value
end)
addDropdown(latestSound, "Kill Sound Type", "KillSoundType", "EZ", {"EZ", "Money"}, false, function(value)
    latestSounds.KillType = value
end)
local latestIge = latestMiscPage:Section({Name = "In-Game ESP", Side = 1})
addToggle(latestIge, "ESP Scrap", "IGEScrap", false, function(value)
    worldEspSettings.Scrap = value
    scanWorldEspObjects()
end)
addSlider(latestIge, "Scrap Distance", "IGEScrapDistance", 20, 1000, 120, "m", function(value)
    worldEspSettings.Distance = value
end)
addDropdown(latestIge, "Show Scrap Types", "IGEScrapType", {"Bad", "Good", "Rare", "Legendary"}, {"Bad", "Good", "Rare", "Legendary"}, true, function(value)
    runtimeState.data.latestScrapTypes = latestSetFromMulti(value)
end)
addToggle(latestIge, "ESP CashDrop", "IGECash", false, function(value)
    worldEspSettings.CashDrop = value
    scanWorldEspObjects()
end)
addToggle(latestIge, "ESP Tools", "IGETools", false, function(value)
    worldEspSettings.Tools = value
    scanWorldEspObjects()
end)
addToggle(latestIge, "ESP Safe/Register", "IGESafe", false, function(value)
    worldEspSettings.Safe = value
    scanWorldEspObjects()
end)
addToggle(latestIge, "ESP ATM", "IGEATM", false, function(value)
    worldEspSettings.ATM = value
    scanWorldEspObjects()
end)
addToggle(latestIge, "ESP Dealer", "IGEDealer", false, function(value)
    worldEspSettings.Dealer = value
    scanWorldEspObjects()
end)
local latestWorld = latestMiscPage:Section({Name = "World", Side = 2})
addToggle(latestWorld, "Full Bright", "WorldFullBright", false, function(value)
    visualSettings.FullBright = value
    runtimeState.data.setFullBright(value)
end)
addSlider(latestWorld, "Clock Time", "WorldClockTime", 0, 24, 14, "", function(value)
    lighting.ClockTime = value
end)
addToggle(latestWorld, "Admin Check", "WorldAdminCheck", false, function(value)
    administratorCheckSettings.On = value
    if value then
        for _, player in ipairs(playersService:GetPlayers()) do
            if player ~= localPlayer and isAdministrator(player) then
                interfaceLibrary:Notification({Title = "Admin Check", Description = player.Name, Duration = 5, Icon = "85279746515974"})
            end
        end
    end
end)
addToggle(latestWorld, "Custom FOV", "WorldCustomFOV", false, function(value)
    visualSettings.FovOn = value
    currentCamera.FieldOfView = value and visualSettings.Fov or 70
end)
addSlider(latestWorld, "FOV", "WorldFOV", 70, 120, 70, "", function(value)
    visualSettings.Fov = value
    if visualSettings.FovOn then
        currentCamera.FieldOfView = value
    end
end)
addSlider(latestWorld, "Camera Distance", "WorldCamDistance", 10, 1000, 10, "", function(value)
    visualSettings.CameraDistance = value
    localPlayer.CameraMaxZoomDistance = value
end)
local latestFun = latestMiscPage:Section({Name = "Fun", Side = 2})
addToggle(latestFun, "Hug Tool", "FunHugTool", false, function(value)
    toyToolSettings.Hug = value
    if value then createToyTool("Hug") else removeToyTool("Hug") end
end)
addToggle(latestFun, "Jerk Tool", "FunJerkTool", false, function(value)
    toyToolSettings.Jerk = value
    if value then createToyTool("Jerk") else removeToyTool("Jerk") end
end)
addToggle(latestFun, "Carpet Tool", "FunCarpetTool", false, function(value)
    toyToolSettings.Carpet = value
    if value then createToyTool("Carpet") else removeToyTool("Carpet") end
end)
addToggle(latestFun, "Fake-Downed Tool", "FunFakeDownedTool", false, function(value)
    toyToolSettings.Down = value
    if value then createToyTool("Fake-Downed") else removeToyTool("Fake-Downed") end
end)
addToggle(latestFun, "Hide Head", "FunHideHead", false, function(value)
    visualSettings.HideHead = value
    updateLocalBodyVisibility()
end)
local latestHideBodyToggle = addToggle(latestFun, "Hide Body", "FunHideBody", false, function(value)
    visualSettings.HideBody = value
    updateLocalBodyVisibility()
end)
addKeybind(latestHideBodyToggle, "FunHideBodyKey", Enum.KeyCode.Backspace, function(value)
    visualSettings.HideBody = value
    updateLocalBodyVisibility()
end)
addButton(latestFun, "Stop Dance", function()
    for name, state in pairs(runtimeState.data.toyAnimationStates or {}) do
        state.active = false
        for _, track in ipairs(state.tracks or {}) do
            pcall(track.Stop, track)
        end
        removeToyTool(name)
    end
end)
local latestGameEvent = latestMiscPage:Section({Name = "Game Event", Side = 2})
addToggle(latestGameEvent, "Key ESP", "GameEventKeyESP", false, function(value)
    keyEspSettings.Enabled = value
    worldEspSettings.Key = value
    refreshKeyEspObjects()
    scanWorldEspObjects()
end)
addToggle(latestGameEvent, "Highlight Exit Door", "GameEventExitDoor", false, latestSetExitDoor)
local latestDealer = latestMiscPage:Section({Name = "Dealer Stock", Side = 2})
addDropdown(latestDealer, "Items to Monitor", "DealerStockMonitor", {}, latestDealerItems, true, function(value)
    stockCheckerSettings.SelectedItems = latestSetFromMulti(value)
end)
addToggle(latestDealer, "Notify New Stock", "DealerStockNotify", true, function(value)
    stockCheckerSettings.NotifyNewStock = value
    dealerStockSettings.On = value
end)
addToggle(latestDealer, "ESP Stock Dealer", "DealerStockESP", true, function(value)
    stockCheckerSettings.DealerEsp = value
    refreshDealerStockEsp()
end)
local latestDealerTracer = addToggle(latestDealer, "Tracer", "DealerStockESPTracer", true, function(value)
    stockCheckerSettings.EspTypes.Tracer = value
    refreshDealerStockEsp()
end)
addColorPicker(latestDealerTracer, "DealerStockESPTracerColor", latestDealerColors.Tracer, function(value)
    latestDealerColors.Tracer = value
end)
local latestDealerHighlight = addToggle(latestDealer, "Highlight", "DealerStockESPHighlight", true, function(value)
    stockCheckerSettings.EspTypes.Highlight = value
    refreshDealerStockEsp()
end)
addColorPicker(latestDealerHighlight, "DealerStockESPHighlightColor", latestDealerColors.Highlight, function(value)
    latestDealerColors.Highlight = value
end)
local latestDealerText = addToggle(latestDealer, "Text", "DealerStockESPText", true, function(value)
    stockCheckerSettings.EspTypes.Text = value
    refreshDealerStockEsp()
end)
addColorPicker(latestDealerText, "DealerStockESPTextColor", latestDealerColors.Text, function(value)
    latestDealerColors.Text = value
end)
local latestCrosshair = latestMiscPage:Section({Name = "Crosshair", Side = 1})
local latestCrosshairToggle = addToggle(latestCrosshair, "Crosshair", "MiscCrosshairEnabled", false, function(value)
    recoveredState.Crosshair.Enabled = value
end)
addDropdown(latestCrosshair, "Crosshair Design", "MiscCrosshairDesign", "Cross", {"Cross", "Circle"}, false, function(value)
    recoveredState.Crosshair.Design = value
end)
addToggle(latestCrosshair, "Spin Crosshair", "MiscCrosshairSpin", false, function(value)
    recoveredState.Crosshair.Spin = value
end)
addColorPicker(latestCrosshairToggle, "MiscCrosshairCrossColor", recoveredState.Crosshair.CrossColor, function(value)
    recoveredState.Crosshair.CrossColor = value
end)
addColorPicker(latestCrosshairToggle, "MiscCrosshairCircleColor", recoveredState.Crosshair.CircleColor, function(value)
    recoveredState.Crosshair.CircleColor = value
end)
local latestCharacter = latestCharacterPage:Section({Name = "Character", Side = 1})
addToggle(latestCharacter, "Auto Lockpick Safe", "CharacterAutoLockpickSafe", false, function(value)
    recoveredState.AutoLockpickSafe = value
    if value then utilitySettings.NoFail = true end
end)
addToggle(latestCharacter, "Fly Mobile", "CharacterFlyMobile", false, function(value)
    recoveredState.FlyMobile = value
    flyMobileGui.Enabled = value
end)
addToggle(latestCharacter, "Ragebot UI Mobile", "RageBotMobileUI", false, function(value)
    rageBot.MobileUI = value
    rageMobileGui.Enabled = value
end)
end)()
