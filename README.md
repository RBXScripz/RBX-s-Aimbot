-- Load OrionLib
local OrionLib = loadstring(game:HttpGet('https://raw.githubusercontent.com/shlexware/Orion/main/source'))()
local Window = OrionLib:MakeWindow({Name = "RBX's Aimlocking Script", HidePremium = false, SaveConfig = true, ConfigFolder = "OrionTest"})

-- Variables for flying and aimlock
local player = game.Players.LocalPlayer
local camera = workspace.CurrentCamera
local userInputService = game:GetService("UserInputService")
local runService = game:GetService("RunService")
local flying = false
local speed = 50
local bodyGyro, bodyVelocity
local aimlockEnabled = false
local lockedTarget = nil

-- Function to create ESP for enemy players
local function createESP(targetPlayer)
    local espBox = Instance.new("BoxHandleAdornment")
    espBox.Adornee = targetPlayer.Character.HumanoidRootPart
    espBox.Size = Vector3.new(2, 5, 1) -- Size of the ESP box
    espBox.Color3 = targetPlayer.TeamColor.Color
    espBox.Transparency = 0.5
    espBox.ZIndex = 10
    espBox.AlwaysOnTop = true
    espBox.Parent = targetPlayer.Character.HumanoidRootPart
end

-- Function to find the closest enemy target
local function findClosestTarget()
    local closestTarget = nil
    local closestDistance = math.huge
    local playerTeam = player.Team

    for _, targetPlayer in pairs(game.Players:GetPlayers()) do
        if targetPlayer ~= player and targetPlayer.Team ~= playerTeam and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local targetPosition = targetPlayer.Character.HumanoidRootPart.Position
            local distance = (player.Character.HumanoidRootPart.Position - targetPosition).Magnitude
            
            if distance < closestDistance then
                closestDistance = distance
                closestTarget = targetPlayer.Character.HumanoidRootPart
            end
        end
    end

    return closestTarget
end

-- Function to smoothly aim at the target
local function aimAtTarget(target)
    if target then
        local targetPosition = target.Position
        local desiredCFrame = CFrame.new(camera.CFrame.Position, targetPosition)
        camera.CFrame = camera.CFrame:Lerp(desiredCFrame, 0.1) -- Smooth aiming
    end
end

-- Toggle Fly Function
local function toggleFly()
    if flying then
        -- Disable flying
        flying = false
        if bodyGyro then bodyGyro:Destroy() end
        if bodyVelocity then bodyVelocity:Destroy() end
        player.Character:WaitForChild("Humanoid").PlatformStand = false -- Allow normal physics
    else
        -- Enable flying
        flying = true
        player.Character:WaitForChild("Humanoid").PlatformStand = true -- Prevent ragdolling

        -- Create BodyGyro and BodyVelocity for flying
        bodyGyro = Instance.new("BodyGyro")
        bodyGyro.CFrame = player.Character.HumanoidRootPart.CFrame
        bodyGyro.P = 9e4
        bodyGyro.MaxTorque = Vector3.new(9e4, 9e4, 9e4)
        bodyGyro.Parent = player.Character.HumanoidRootPart

        bodyVelocity = Instance.new("BodyVelocity")
        bodyVelocity.Velocity = Vector3.new(0, 0, 0)
        bodyVelocity.MaxForce = Vector3.new(9e4, 9e4, 9e4)
        bodyVelocity.Parent = player.Character.HumanoidRootPart

        -- Flying loop
        runService.Heartbeat:Connect(function()
            if flying then
                local moveDirection = Vector3.new(0, 0, 0)

                -- Mobile Controls: Touch input for moving
                if userInputService.TouchEnabled then
                    local touches = userInputService:GetTouches()
                    for _, touch in ipairs(touches) do
                        if touch.UserInputState == Enum.UserInputState.Begin then
                            local touchPosition = touch.Position
                            local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)

                            -- Calculate movement direction based on touch position relative to center
                            local deltaX = touchPosition.X - screenCenter.X
                            local deltaY = touchPosition.Y - screenCenter.Y

                            moveDirection = Vector3.new(deltaX, 0, deltaY).unit
                            bodyVelocity.Velocity = Vector3.new(moveDirection.X * speed, moveDirection.Y * speed, moveDirection.Z * speed)
                        end
                    end
                else
                    -- PC Controls: W, A, S, D
                    if userInputService:IsKeyDown(Enum.KeyCode.W) then
                        moveDirection = moveDirection + camera.CFrame.LookVector
                    end
                    if userInputService:IsKeyDown(Enum.KeyCode.S) then
                        moveDirection = moveDirection - camera.CFrame.LookVector
                    end
                    if userInputService:IsKeyDown(Enum.KeyCode.A) then
                        moveDirection = moveDirection - camera.CFrame.RightVector
                    end
                    if userInputService:IsKeyDown(Enum.KeyCode.D) then
                        moveDirection = moveDirection + camera.CFrame.RightVector
                    end

                    bodyVelocity.Velocity = moveDirection * speed
                end

                bodyGyro.CFrame = camera.CFrame -- Lock orientation to camera

                -- Aimlock functionality
                if aimlockEnabled then
                    lockedTarget = findClosestTarget()
                    aimAtTarget(lockedTarget)
                end
            end
        end)
    end
end

-- Create UI Buttons
local Tab = Window:MakeTab({
    Name = "Main",
    Icon = "rbxassetid://4483345998",
    PremiumOnly = false
})

-- Fly Button
Tab:AddButton({
    Name = "Toggle Fly",
    Callback = function()
        toggleFly()
    end    
})

-- Aimlock Toggle
Tab:AddToggle({
    Name = "Toggle Aimlock",
    Default = false,
    Callback = function(Value)
        aimlockEnabled = Value
    end    
})

-- ESP Toggle
Tab:AddToggle({
    Name = "Toggle ESP",
    Default = true,
    Callback = function(Value)
        for _, targetPlayer in pairs(game.Players:GetPlayers()) do
            if targetPlayer ~= player and targetPlayer.Team ~= player.Team and targetPlayer.Character then
                if Value then
                    createESP(targetPlayer)
                end
            end
        end
    end    
})

-- ESP Refresh every second
runService.Heartbeat:Connect(function()
    if aimlockEnabled then
        lockedTarget = findClosestTarget() -- Update the locked target every frame
        aimAtTarget(lockedTarget)
    end
end)

OrionLib:Init()
