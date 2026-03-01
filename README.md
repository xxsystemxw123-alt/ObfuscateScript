-- === WHATHEDOGDOIN (COMPLETO - MOVE PARTS FUNCIONA) ===

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local StarterGui = game:GetService("StarterGui")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- Mensaje de carga
StarterGui:SetCore("SendNotification", {
    Title = "WHATHEDOGDOIN",
    Text = "[ WHATHEDOGDOIN ] Loaded!",
    Duration = 4
})

-- --- VARIABLES GLOBALES ---
_G.selectedPart = nil
_G.selectedHighlight = nil
_G.floatSpeed = 10 
_G.flySpeed = 1.5   
_G.isFlying = false
_G.hiddenfling = false 
_G.antiFling = false
_G.followCamera = false
_G.followDistance = 15
_G.moveDirection = Vector3.new(0, 0, 0)
_G.initialRotation = CFrame.new()
_G.flingThread = nil
_G.flingSpeed = 10000

-- Troll variables
_G.bangActive = false
_G.faceBangActive = false
_G.sitActive = false
_G.suckActive = false
_G.ragdollActive = false
_G.infiniteJumpActive = false
_G.noclipActive = false
_G.espEnabled = false
_G.jerkActive = false

-- Conexiones
_G.bangConnection = nil
_G.faceBangConnection = nil
_G.sitConnection = nil
_G.suckConnection = nil
_G.ragdollConnection = nil
_G.infiniteJumpConnection = nil
_G.noclipConnection = nil
_G.flingConnection = nil
_G.jerkTrack = nil
_G.closerhandsTrack = nil
_G.jerkTool = nil

-- Fly Car
_G.flyCarActive = false
_G.flyCarSpeed = 50
_G.flyCarBV = nil
_G.flyCarBG = nil

-- --- FUNCIONES BÁSICAS ---
local function getRoot()
    local char = LocalPlayer.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Humanoid")
end

-- --- FUNCIONES DE DIRECCIÓN (para MOVE PARTS) ---
local function getCameraRelativeDirection(direction)
    if direction.Y ~= 0 then return direction end
    local look = Camera.CFrame.LookVector
    local right = Camera.CFrame.RightVector
    local lookFlat = Vector3.new(look.X, 0, look.Z).Unit
    local rightFlat = Vector3.new(right.X, 0, right.Z).Unit
    if direction.Z > 0 then
        return lookFlat
    elseif direction.Z < 0 then
        return -lookFlat
    elseif direction.X > 0 then
        return rightFlat
    elseif direction.X < 0 then
        return -rightFlat
    end
    return direction
end

-- --- CONTROL DE OBJETOS (MOVE PARTS) ---
local function ProcessPart(part)
    if not part then return end
    
    for _, child in ipairs(part:GetChildren()) do
        if child:IsA("BodyVelocity") or child:IsA("BodyAngularVelocity") or child:IsA("BodyGyro") or child.Name == "ControlVelocity" or child.Name == "ControlGyro" then
            child:Destroy()
        end
    end
    
    local bv = Instance.new("BodyVelocity")
    bv.Name = "ControlVelocity"
    bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bv.Velocity = _G.moveDirection * _G.floatSpeed
    bv.P = 1250
    bv.Parent = part
    
    local bg = Instance.new("BodyGyro")
    bg.Name = "ControlGyro"
    bg.P = 9e4
    bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bg.CFrame = part.CFrame
    bg.Parent = part
end

local function UpdateVelocity(part)
    if not part or not part.Parent then return end
    
    local bv = part:FindFirstChild("ControlVelocity")
    if bv and bv:IsA("BodyVelocity") then
        bv.Velocity = _G.moveDirection * _G.floatSpeed
    end
end

local function ClearForces(part)
    if not part or not part.Parent then return end
    
    for _, child in ipairs(part:GetChildren()) do
        if child:IsA("BodyVelocity") or child:IsA("BodyPosition") or child:IsA("BodyGyro") or 
           child.Name == "ControlVelocity" or child.Name == "ControlGyro" then
            child:Destroy()
        end
    end
end

-- --- FUNCIONES DE TROLL ---

-- BANG
function toggleBang(btn)
    if _G.bangActive then
        _G.bangActive = false
        if _G.bangConnection then
            _G.bangConnection:Disconnect()
            _G.bangConnection = nil
        end
        
        -- Limpiar fuerzas del personaje
        local hrp = getRoot()
        if hrp then
            for _, child in ipairs(hrp:GetChildren()) do
                if child:IsA("BodyPosition") or child:IsA("BodyGyro") then
                    child:Destroy()
                end
            end
        end
        
        -- Restaurar humanoid
        local humanoid = getHumanoid()
        if humanoid then
            humanoid.Sit = false
            humanoid.PlatformStand = false
            humanoid.AutoRotate = true
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
        end
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200, 130, 110) end
        return
    end
    
    _G.bangActive = true
    if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    
    local targetPlayer = nil
    local myRoot = getRoot()
    if myRoot then
        local nearestDist = math.huge
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local root = player.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    local dist = (myRoot.Position - root.Position).Magnitude
                    if dist < nearestDist then
                        nearestDist = dist
                        targetPlayer = player
                    end
                end
            end
        end
    end
    
    if not targetPlayer then
        _G.bangActive = false
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200, 130, 110) end
        return
    end
    
    local lastTime = tick()
    local direction = 1
    
    _G.bangConnection = RunService.Heartbeat:Connect(function()
        if not _G.bangActive or not targetPlayer.Character then
            _G.bangActive = false
            if _G.bangConnection then
                _G.bangConnection:Disconnect()
                _G.bangConnection = nil
            end
            return
        end
        
        local targetRoot = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
        local myHrp = getRoot()
        if not targetRoot or not myHrp then return end
        
        if tick() - lastTime > 0.3 then
            direction = direction * -1
            lastTime = tick()
        end
        
        local behindPos = targetRoot.Position - targetRoot.CFrame.LookVector * 1.2
        behindPos = Vector3.new(behindPos.X, targetRoot.Position.Y, behindPos.Z)
        local targetPos = behindPos + targetRoot.CFrame.LookVector * direction * 1.0
        
        local bp = myHrp:FindFirstChild("BangBP") or Instance.new("BodyPosition")
        bp.Name = "BangBP"
        bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bp.P = 10000
        bp.Position = targetPos
        bp.Parent = myHrp
        
        local bg = myHrp:FindFirstChild("BangBG") or Instance.new("BodyGyro")
        bg.Name = "BangBG"
        bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        bg.CFrame = CFrame.new(myHrp.Position, targetRoot.Position)
        bg.Parent = myHrp
    end)
end

-- FACE BANG
function toggleFaceBang(btn)
    if _G.faceBangActive then
        _G.faceBangActive = false
        if _G.faceBangConnection then
            _G.faceBangConnection:Disconnect()
            _G.faceBangConnection = nil
        end
        
        -- Limpiar fuerzas del personaje
        local hrp = getRoot()
        if hrp then
            for _, child in ipairs(hrp:GetChildren()) do
                if child:IsA("BodyPosition") or child:IsA("BodyGyro") then
                    child:Destroy()
                end
            end
        end
        
        -- Restaurar humanoid
        local humanoid = getHumanoid()
        if humanoid then
            humanoid.Sit = false
            humanoid.PlatformStand = false
            humanoid.AutoRotate = true
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
        end
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 170, 200) end
        return
    end
    
    _G.faceBangActive = true
    if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    
    local targetPlayer = nil
    local myRoot = getRoot()
    if myRoot then
        local nearestDist = math.huge
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local root = player.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    local dist = (myRoot.Position - root.Position).Magnitude
                    if dist < nearestDist then
                        nearestDist = dist
                        targetPlayer = player
                    end
                end
            end
        end
    end
    
    if not targetPlayer then
        _G.faceBangActive = false
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 170, 200) end
        return
    end
    
    local humanoid = getHumanoid()
    if humanoid then humanoid.Sit = true end
    
    local lastTime = tick()
    local direction = 1
    
    _G.faceBangConnection = RunService.Heartbeat:Connect(function()
        if not _G.faceBangActive or not targetPlayer.Character then
            _G.faceBangActive = false
            if _G.faceBangConnection then
                _G.faceBangConnection:Disconnect()
                _G.faceBangConnection = nil
            end
            return
        end
        
        local targetHead = targetPlayer.Character:FindFirstChild("Head")
        local myHrp = getRoot()
        local myHead = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Head")
        if not targetHead or not myHrp or not myHead then return end
        
        if tick() - lastTime > 0.2 then
            direction = direction * -1
            lastTime = tick()
        end
        
        local basePos = targetHead.Position + targetHead.CFrame.LookVector * 1.0
        local targetPos = Vector3.new(basePos.X, targetHead.Position.Y - myHead.Size.Y/2 + 0.5, basePos.Z)
        targetPos = targetPos + targetHead.CFrame.LookVector * direction * 0.8
        
        local bp = myHrp:FindFirstChild("FaceBangBP") or Instance.new("BodyPosition")
        bp.Name = "FaceBangBP"
        bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bp.P = 10000
        bp.Position = targetPos
        bp.Parent = myHrp
        
        local bg = myHrp:FindFirstChild("FaceBangBG") or Instance.new("BodyGyro")
        bg.Name = "FaceBangBG"
        bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        bg.CFrame = CFrame.new(myHrp.Position, targetHead.Position)
        bg.Parent = myHrp
    end)
end

-- SIT
function toggleSit(btn)
    if _G.sitActive then
        _G.sitActive = false
        if _G.sitConnection then
            _G.sitConnection:Disconnect()
            _G.sitConnection = nil
        end
        
        -- Limpiar fuerzas del personaje
        local hrp = getRoot()
        if hrp then
            for _, child in ipairs(hrp:GetChildren()) do
                if child:IsA("BodyPosition") or child:IsA("BodyGyro") then
                    child:Destroy()
                end
            end
        end
        
        -- Restaurar humanoid
        local humanoid = getHumanoid()
        if humanoid then
            humanoid.Sit = false
            humanoid.PlatformStand = false
            humanoid.AutoRotate = true
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
        end
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 180, 150) end
        return
    end
    
    _G.sitActive = true
    if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    
    local targetPlayer = nil
    local myRoot = getRoot()
    if myRoot then
        local nearestDist = math.huge
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local root = player.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    local dist = (myRoot.Position - root.Position).Magnitude
                    if dist < nearestDist then
                        nearestDist = dist
                        targetPlayer = player
                    end
                end
            end
        end
    end
    
    if not targetPlayer then
        _G.sitActive = false
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 180, 150) end
        return
    end
    
    local humanoid = getHumanoid()
    if humanoid then humanoid.Sit = true end
    
    _G.sitConnection = RunService.Heartbeat:Connect(function()
        if not _G.sitActive or not targetPlayer.Character then
            _G.sitActive = false
            if _G.sitConnection then
                _G.sitConnection:Disconnect()
                _G.sitConnection = nil
            end
            return
        end
        
        local targetHead = targetPlayer.Character:FindFirstChild("Head")
        local myHrp = getRoot()
        if not targetHead or not myHrp then return end
        
        local bp = myHrp:FindFirstChild("SitBP") or Instance.new("BodyPosition")
        bp.Name = "SitBP"
        bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bp.P = 10000
        bp.Position = targetHead.Position + Vector3.new(0, 0.8, 0)
        bp.Parent = myHrp
        
        local bg = myHrp:FindFirstChild("SitBG") or Instance.new("BodyGyro")
        bg.Name = "SitBG"
        bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        bg.CFrame = targetHead.CFrame
        bg.Parent = myHrp
    end)
end

-- SUCK
function toggleSuck(btn)
    if _G.suckActive then
        _G.suckActive = false
        if _G.suckConnection then
            _G.suckConnection:Disconnect()
            _G.suckConnection = nil
        end
        
        -- Limpiar fuerzas del personaje
        local hrp = getRoot()
        if hrp then
            for _, child in ipairs(hrp:GetChildren()) do
                if child:IsA("BodyPosition") or child:IsA("BodyGyro") then
                    child:Destroy()
                end
            end
        end
        
        -- Restaurar humanoid
        local humanoid = getHumanoid()
        if humanoid then
            humanoid.Sit = false
            humanoid.PlatformStand = false
            humanoid.AutoRotate = true
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
        end
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200, 150, 180) end
        return
    end
    
    _G.suckActive = true
    if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    
    local targetPlayer = nil
    local myRoot = getRoot()
    if myRoot then
        local nearestDist = math.huge
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local root = player.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    local dist = (myRoot.Position - root.Position).Magnitude
                    if dist < nearestDist then
                        nearestDist = dist
                        targetPlayer = player
                    end
                end
            end
        end
    end
    
    if not targetPlayer then
        _G.suckActive = false
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200, 150, 180) end
        return
    end
    
    local lastTime = tick()
    local direction = 1
    
    _G.suckConnection = RunService.Heartbeat:Connect(function()
        if not _G.suckActive or not targetPlayer.Character then
            _G.suckActive = false
            if _G.suckConnection then
                _G.suckConnection:Disconnect()
                _G.suckConnection = nil
            end
            return
        end
        
        local targetRoot = targetPlayer.Character:FindFirstChild("HumanoidRootPart") or targetPlayer.Character:FindFirstChild("Torso")
        local myHrp = getRoot()
        local myHead = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Head")
        if not targetRoot or not myHrp or not myHead then return end
        
        if tick() - lastTime > 0.25 then
            direction = direction * -1
            lastTime = tick()
        end
        
        local groinPos = targetRoot.Position + Vector3.new(0, -2, 0)
        local frontPos = groinPos + targetRoot.CFrame.LookVector * 1.2
        local targetPos = Vector3.new(frontPos.X, groinPos.Y - myHead.Size.Y/2 + 0.3, frontPos.Z)
        targetPos = targetPos + targetRoot.CFrame.LookVector * direction * 1.0
        
        local bp = myHrp:FindFirstChild("SuckBP") or Instance.new("BodyPosition")
        bp.Name = "SuckBP"
        bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bp.P = 10000
        bp.Position = targetPos
        bp.Parent = myHrp
        
        local bg = myHrp:FindFirstChild("SuckBG") or Instance.new("BodyGyro")
        bg.Name = "SuckBG"
        bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        bg.CFrame = CFrame.new(myHrp.Position, targetRoot.Position)
        bg.Parent = myHrp
    end)
end

-- RAGDOLL (SOLO CAER, SIN CONVULSIONES)
function toggleRagdoll(btn)
    _G.ragdollActive = not _G.ragdollActive
    
    local humanoid = getHumanoid()
    if not humanoid then return end
    
    if _G.ragdollActive then
        humanoid.PlatformStand = true
        humanoid.AutoRotate = false
        humanoid:ChangeState(Enum.HumanoidStateType.Physics)
        
        local hrp = getRoot()
        if hrp then
            hrp.Velocity = Vector3.new(0, -10, 0) -- Solo caída suave
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    else
        humanoid.PlatformStand = false
        humanoid.AutoRotate = true
        humanoid:ChangeState(Enum.HumanoidStateType.Running)
        if btn then btn.BackgroundColor3 = Color3.fromRGB(210, 160, 130) end
    end
end

-- INFINITE JUMP
function toggleInfiniteJump(btn)
    _G.infiniteJumpActive = not _G.infiniteJumpActive
    
    if _G.infiniteJumpActive then
        _G.infiniteJumpConnection = UserInputService.JumpRequest:Connect(function()
            if not _G.infiniteJumpActive then
                if _G.infiniteJumpConnection then
                    _G.infiniteJumpConnection:Disconnect()
                    _G.infiniteJumpConnection = nil
                end
                return
            end
            local humanoid = getHumanoid()
            if humanoid then humanoid:ChangeState(Enum.HumanoidStateType.Jumping) end
        end)
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    else
        if _G.infiniteJumpConnection then
            _G.infiniteJumpConnection:Disconnect()
            _G.infiniteJumpConnection = nil
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 170, 200) end
    end
end

-- NOCLIP
function toggleNoclip(btn)
    _G.noclipActive = not _G.noclipActive
    
    if _G.noclipActive then
        _G.noclipConnection = RunService.Stepped:Connect(function()
            if not _G.noclipActive or not LocalPlayer.Character then
                if _G.noclipConnection then
                    _G.noclipConnection:Disconnect()
                    _G.noclipConnection = nil
                end
                return
            end
            for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
        end)
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    else
        if _G.noclipConnection then
            _G.noclipConnection:Disconnect()
            _G.noclipConnection = nil
        end
        if LocalPlayer.Character then
            for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = true end
            end
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 170, 200) end
    end
end

-- ESP
function toggleESP(btn)
    _G.espEnabled = not _G.espEnabled
    
    if _G.espEnabled then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                local highlight = Instance.new("Highlight")
                highlight.Adornee = player.Character
                highlight.FillColor = Color3.fromRGB(255, 255, 0)
                highlight.FillTransparency = 0.5
                highlight.Parent = player.Character
            end
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100) end
    else
        for _, player in ipairs(Players:GetPlayers()) do
            if player.Character then
                local old = player.Character:FindFirstChildOfClass("Highlight")
                if old then old:Destroy() end
            end
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 170, 200) end
    end
end

-- RESET
function resetStats(btn)
    if _G.bangActive then toggleBang() end
    if _G.faceBangActive then toggleFaceBang() end
    if _G.sitActive then toggleSit() end
    if _G.suckActive then toggleSuck() end
    if _G.ragdollActive then toggleRagdoll() end
    if _G.infiniteJumpActive then toggleInfiniteJump() end
    if _G.noclipActive then toggleNoclip() end
    if _G.espEnabled then toggleESP() end
    if _G.jerkActive then toggleJerk() end
    if _G.hiddenfling then 
        _G.hiddenfling = false
        if _G.flingConnection then
            _G.flingConnection:Disconnect()
            _G.flingConnection = nil
        end
    end
    
    if btn then 
        btn.BackgroundColor3 = Color3.fromRGB(180, 140, 100)
        task.wait(0.2)
        btn.BackgroundColor3 = Color3.fromRGB(180, 140, 100)
    end
end

-- FLING (con control de velocidad)
function toggleFling(btn, speedBox)
    _G.hiddenfling = not _G.hiddenfling
    
    if _G.hiddenfling then
        if btn then 
            btn.Text = "FLING: ON"
            btn.BackgroundColor3 = Color3.fromRGB(200, 100, 100)
        end
        
        _G.flingConnection = RunService.Heartbeat:Connect(function()
            if not _G.hiddenfling then
                if _G.flingConnection then
                    _G.flingConnection:Disconnect()
                    _G.flingConnection = nil
                end
                return
            end
            
            local char = LocalPlayer.Character
            if not char then return end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            
            local vel = hrp.Velocity
            hrp.Velocity = vel * (_G.flingSpeed/5000) + Vector3.new(0, _G.flingSpeed/2, 0)
            RunService.RenderStepped:Wait()
            hrp.Velocity = vel
            RunService.Stepped:Wait()
            hrp.Velocity = vel + Vector3.new(0, 0.1, 0)
        end)
    else
        if _G.flingConnection then
            _G.flingConnection:Disconnect()
            _G.flingConnection = nil
        end
        if btn then 
            btn.Text = "FLING: OFF"
            btn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        end
    end
end

-- ANTIFLING
function toggleAntiFling(btn)
    _G.antiFling = not _G.antiFling
    if btn then 
        btn.Text = _G.antiFling and "ANTIFLING: ON" or "ANTIFLING: OFF"
        btn.BackgroundColor3 = _G.antiFling and Color3.fromRGB(150, 180, 150) or Color3.fromRGB(210, 160, 130)
    end
end

-- JERK
function toggleJerk(btn)
    local humanoid = getHumanoid()
    if not humanoid then return end
    
    local Character = LocalPlayer.Character
    if not Character or not Character:FindFirstChild("Torso") then
        StarterGui:SetCore("SendNotification", {
            Title = "Error",
            Text = "Jerk solo funciona en R6",
            Duration = 2
        })
        return
    end
    
    if _G.jerkActive then
        _G.jerkActive = false
        if _G.jerkTrack then
            _G.jerkTrack:Stop()
            _G.jerkTrack = nil
        end
        if _G.closerhandsTrack then
            _G.closerhandsTrack:Stop()
            _G.closerhandsTrack = nil
        end
        humanoid.WalkSpeed = 16
        humanoid.JumpPower = 50
        if _G.jerkTool then
            _G.jerkTool:Destroy()
            _G.jerkTool = nil
        end
        if btn then 
            btn.Text = "🍆 JERK (R6)"
            btn.BackgroundColor3 = Color3.fromRGB(180, 140, 200)
        end
    else
        _G.jerkActive = true
        _G.jerkTool = Instance.new("Tool")
        _G.jerkTool.Name = "jerk (r6)"
        _G.jerkTool.RequiresHandle = false
        
        local newTool = _G.jerkTool:Clone()
        newTool.Parent = LocalPlayer.Backpack
        
        local jerkingOff = false
        local originalWalkSpeed = humanoid.WalkSpeed
        local originalJumpPower = humanoid.JumpPower
        
        newTool.Equipped:Connect(function()
            jerkingOff = true
        end)
        
        newTool.Unequipped:Connect(function()
            jerkingOff = false
            humanoid.WalkSpeed = originalWalkSpeed
            humanoid.JumpPower = originalJumpPower
            if _G.jerkTrack then
                _G.jerkTrack:Stop()
                _G.jerkTrack = nil
            end
            if _G.closerhandsTrack then
                _G.closerhandsTrack:Stop()
                _G.closerhandsTrack = nil
            end
        end)
        
        local connection = RunService.RenderStepped:Connect(function()
            if not humanoid or not humanoid.Parent then return end
            
            if jerkingOff and _G.jerkActive then
                humanoid.WalkSpeed = 0
                humanoid.JumpPower = 0
                
                if not _G.jerkTrack then
                    local anim = Instance.new("Animation")
                    anim.AnimationId = "rbxassetid://99198989"
                    _G.jerkTrack = humanoid:LoadAnimation(anim)
                    _G.jerkTrack.Looped = true
                    _G.jerkTrack:Play()
                end
                
                if not _G.closerhandsTrack then
                    local anim = Instance.new("Animation")
                    anim.AnimationId = "rbxassetid://168086975"
                    _G.closerhandsTrack = humanoid:LoadAnimation(anim)
                    _G.closerhandsTrack:Play()
                end
            elseif not jerkingOff then
                humanoid.WalkSpeed = originalWalkSpeed
                humanoid.JumpPower = originalJumpPower
                if _G.jerkTrack then
                    _G.jerkTrack:Stop()
                    _G.jerkTrack = nil
                end
                if _G.closerhandsTrack then
                    _G.closerhandsTrack:Stop()
                    _G.closerhandsTrack = nil
                end
            end
        end)
        
        newTool.AncestryChanged:Connect(function()
            if not newTool.Parent then
                connection:Disconnect()
                jerkingOff = false
                _G.jerkActive = false
                humanoid.WalkSpeed = originalWalkSpeed
                humanoid.JumpPower = originalJumpPower
                if btn then 
                    btn.Text = "🍆 JERK (R6)"
                    btn.BackgroundColor3 = Color3.fromRGB(180, 140, 200)
                end
            end
        end)
        
        if btn then 
            btn.Text = "✅ JERK (R6)"
            btn.BackgroundColor3 = Color3.fromRGB(100, 200, 100)
        end
    end
end

-- FLY CAR
function toggleFlyCar(btn)
    if _G.flyCarActive then
        local hrp = getRoot()
        if hrp then
            if _G.flyCarBV then _G.flyCarBV:Destroy() end
            if _G.flyCarBG then _G.flyCarBG:Destroy() end
        end
        _G.flyCarBV = nil
        _G.flyCarBG = nil
        _G.flyCarActive = false
        if btn then 
            btn.Text = "OFF"
            btn.BackgroundColor3 = Color3.fromRGB(200, 150, 120)
        end
    else
        local hrp = getRoot()
        if not hrp then return end
        
        _G.flyCarBV = Instance.new("BodyVelocity", hrp)
        _G.flyCarBG = Instance.new("BodyGyro", hrp)
        _G.flyCarBV.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        
        RunService.RenderStepped:Connect(function()
            if _G.flyCarBG and _G.flyCarBG.Parent then
                _G.flyCarBG.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
                _G.flyCarBG.CFrame = Camera.CFrame
            end
        end)
        
        _G.flyCarActive = true
        if btn then 
            btn.Text = "ON"
            btn.BackgroundColor3 = Color3.fromRGB(150, 200, 150)
        end
    end
end

function flyCarForward()
    if not _G.flyCarActive or not _G.flyCarBV then return end
    _G.flyCarBV.Velocity = Camera.CFrame.LookVector * _G.flyCarSpeed
end

function flyCarBackward()
    if not _G.flyCarActive or not _G.flyCarBV then return end
    _G.flyCarBV.Velocity = Camera.CFrame.LookVector * -_G.flyCarSpeed
end

function flyCarStop()
    if _G.flyCarBV then
        _G.flyCarBV.Velocity = Vector3.new(0,0,0)
    end
end

-- VUELO (NORMAL - SIN NOCLIP, SIN GOD MODE)
function toggleFly(btn)
    _G.isFlying = not _G.isFlying
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso")
    if not hum or not root then return end
    
    if _G.isFlying then
        for _, state in ipairs(Enum.HumanoidStateType:GetEnumItems()) do
            hum:SetStateEnabled(state, false)
        end
        hum:ChangeState(Enum.HumanoidStateType.Swimming)
        
        local bg = Instance.new("BodyGyro", root)
        bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
        local bv = Instance.new("BodyVelocity", root)
        bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
        
        spawn(function()
            while _G.isFlying and RunService.RenderStepped:Wait() do
                if not char.Parent or not hum or not hum.Parent then break end
                bg.CFrame = Camera.CoordinateFrame
                if hum.MoveDirection.Magnitude > 0 then
                    bv.Velocity = hum.MoveDirection * (45 * _G.flySpeed)
                else
                    bv.Velocity = Vector3.new(0, 0, 0)
                end
            end
            bg:Destroy()
            bv:Destroy()
        end)
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150, 180, 150) end
    else
        if hum and hum.Parent then
            for _, state in ipairs(Enum.HumanoidStateType:GetEnumItems()) do
                hum:SetStateEnabled(state, true)
            end
            hum:ChangeState(Enum.HumanoidStateType.RunningNoPhysics)
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(210, 160, 130) end
    end
end

-- --- FUNCIONES PARA SEGUIR CÁMARA ---
local function updateSliderUI(dist, sliderBg, sliderCircle, distLabel)
    local clampedDist = math.clamp(dist, 5, 150)
    local relPos = (clampedDist - 5) / 145
    sliderCircle.Position = UDim2.new(relPos, -6, 0.5, -6)
    distLabel.Text = "🎯 " .. math.floor(clampedDist) .. "m"
    _G.followDistance = clampedDist
end

-- --- ANIMATIONS DATA ---
local Animations = {
    ["Walk"] = {
        ["Furry Walk"] = "102269417125238",
        ["Catwalk Glam"] = "109168724482748",
        ["Gojo"] = "95643163365384",
        ["Geto"] = "85811471336028",
        ["Adidas"] = "122150855457006",
        ["Zombie"] = "616168032",
        ["Ninja"] = "656121766",
        ["Robot"] = "10921250460",
        ["Stylish"] = "10921276116",
        ["Sneaky"] = "1132510133",
    },
    ["Run"] = {
        ["Furry Run"] = "102269417125238",
        ["Adidas Run"] = "82598234841035",
        ["Heavy Run"] = "3236836670",
        ["Cartoony"] = "10921076136",
        ["Zombie"] = "616163682",
        ["Ninja"] = "656118852",
        ["Robot"] = "10921250460",
        ["Sneaky"] = "1132494274",
    },
    ["Idle"] = {
        ["Furry Idle"] = {"102269417125238", "102269417125238"},
        ["Astronaut"] = {"891621366", "891633237"},
        ["Bold"] = {"16738333868", "16738334710"},
        ["Zombie"] = {"616158929", "616160636"},
        ["Ninja"] = {"656117400", "656118341"},
        ["Stylish"] = {"616136790", "616138447"},
    },
    ["Jump"] = {
        ["Furry Jump"] = "102269417125238",
        ["Adidas"] = "75290611992385",
        ["Ninja"] = "656117878",
        ["Robot"] = "616090535",
        ["Stylish"] = "616139451",
    },
    ["Fall"] = {
        ["Furry Fall"] = "102269417125238",
        ["Adidas"] = "98600215928904",
        ["Ninja"] = "656115606",
        ["Robot"] = "616087089",
    }
}

-- ANIMATION FUNCTIONS
local function setAnimation(animType, animName)
    local animData = Animations[animType] and Animations[animType][animName]
    if not animData then return end
    
    local humanoid = getHumanoid()
    if not humanoid then return end
    
    local animate = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Animate")
    if not animate then return end
    
    for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
        track:Stop()
    end
    
    pcall(function()
        if animType == "Idle" and type(animData) == "table" then
            if animate.idle then
                animate.idle.Animation1.AnimationId = "http://www.roblox.com/asset/?id=" .. animData[1]
                animate.idle.Animation2.AnimationId = "http://www.roblox.com/asset/?id=" .. animData[2]
            end
        elseif animType == "Walk" and animate.walk then
            animate.walk.WalkAnim.AnimationId = "http://www.roblox.com/asset/?id=" .. animData
        elseif animType == "Run" and animate.run then
            animate.run.RunAnim.AnimationId = "http://www.roblox.com/asset/?id=" .. animData
        elseif animType == "Jump" and animate.jump then
            animate.jump.JumpAnim.AnimationId = "http://www.roblox.com/asset/?id=" .. animData
        elseif animType == "Fall" and animate.fall then
            animate.fall.FallAnim.AnimationId = "http://www.roblox.com/asset/?id=" .. animData
        end
    end)
end

-- --- BUCLE PRINCIPAL (para MOVE PARTS y SEGUIR CÁMARA) ---
RunService.Heartbeat:Connect(function()
    pcall(function()
        sethiddenproperty(LocalPlayer, "SimulationRadius", math.huge)
        sethiddenproperty(LocalPlayer, "MaxSimulationRadius", math.huge)
    end)
    
    if _G.selectedPart and _G.selectedPart.Parent and not _G.followCamera then
        UpdateVelocity(_G.selectedPart)
    end
    
    if _G.followCamera and _G.selectedPart and _G.selectedPart.Parent and LocalPlayer.Character then
        ClearForces(_G.selectedPart)
        
        local character = LocalPlayer.Character
        local rootPart = character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso")
        
        if rootPart then
            local targetPos = rootPart.Position + (Camera.CFrame.LookVector * _G.followDistance)
            
            local bp = Instance.new("BodyPosition")
            bp.Name = "FollowBP"
            bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bp.P = 20000
            bp.D = 800
            bp.Position = targetPos
            bp.Parent = _G.selectedPart
            
            local bg = Instance.new("BodyGyro")
            bg.Name = "FollowBG"
            bg.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
            bg.CFrame = _G.initialRotation
            bg.Parent = _G.selectedPart
        end
    end
end)

-- --- ANTIFLING LOOP ---
spawn(function()
    while true do
        RunService.Heartbeat:Wait()
        pcall(function()
            sethiddenproperty(LocalPlayer, "SimulationRadius", math.huge)
            sethiddenproperty(LocalPlayer, "MaxSimulationRadius", math.huge)
        end)
        if _G.antiFling and LocalPlayer.Character then
            local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            if root then root.RotVelocity = Vector3.new(0,0,0) end
            for _, p in pairs(Players:GetPlayers()) do
                if p ~= LocalPlayer and p.Character then
                    for _, v in pairs(p.Character:GetChildren()) do
                        if v:IsA("BasePart") then v.CanCollide = false end
                    end
                end
            end
        end
    end
end)

-- --- CREAR INTERFAZ ---
local function CreateHub()
    local sg = Instance.new("ScreenGui")
    sg.Name = "Whathedogdoin"
    sg.Parent = LocalPlayer:WaitForChild("PlayerGui")
    sg.ResetOnSpawn = false

    -- Botón lunar
    local moonBtn = Instance.new("TextButton")
    moonBtn.Name = "MoonButton"
    moonBtn.Size = UDim2.new(0, 55, 0, 55)
    moonBtn.Position = UDim2.new(0.02, 0, 0.5, -27.5)
    moonBtn.BackgroundColor3 = Color3.fromRGB(230, 180, 150)
    moonBtn.Text = "🐕\nWH"
    moonBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    moonBtn.Font = Enum.Font.GothamBold
    moonBtn.TextSize = 12
    moonBtn.Visible = false
    moonBtn.Draggable = true
    moonBtn.Parent = sg
    local moonCorner = Instance.new("UICorner")
    moonCorner.CornerRadius = UDim.new(0, 27.5)
    moonCorner.Parent = moonBtn

    -- Marco principal
    local main = Instance.new("Frame")
    main.Size = UDim2.new(0, 340, 0, 615)
    main.Position = UDim2.new(0.5, -170, 0.5, -375)
    main.BackgroundColor3 = Color3.fromRGB(245, 225, 200)
    main.BorderSizePixel = 4
    main.BorderColor3 = Color3.fromRGB(200, 150, 120)
    main.Active = true
    main.Draggable = true
    main.Parent = sg
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 12)
    mainCorner.Parent = main

    -- UIScale
    local uiScale = Instance.new("UIScale")
    uiScale.Parent = main
    uiScale.Scale = 1

    -- Panel SUPERIOR (EXTRAS)
    local topPanelBtn = Instance.new("TextButton")
    topPanelBtn.Size = UDim2.new(0, 160, 0, 30)
    topPanelBtn.Position = UDim2.new(0.5, -80, 0, 8)
    topPanelBtn.Text = "▲ EXTRAS ▼"
    topPanelBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    topPanelBtn.BackgroundColor3 = Color3.fromRGB(150, 170, 150)
    topPanelBtn.BorderSizePixel = 2
    topPanelBtn.BorderColor3 = Color3.fromRGB(100, 120, 100)
    topPanelBtn.Font = Enum.Font.GothamBold
    topPanelBtn.TextSize = 12
    topPanelBtn.Parent = main
    local topBtnCorner = Instance.new("UICorner")
    topBtnCorner.CornerRadius = UDim.new(0, 8)
    topBtnCorner.Parent = topPanelBtn

    local topPanel = Instance.new("Frame")
    topPanel.Size = UDim2.new(0, 280, 0, 0)
    topPanel.Position = UDim2.new(0.5, -140, 0, -140)
    topPanel.BackgroundColor3 = Color3.fromRGB(225, 200, 170)
    topPanel.BorderSizePixel = 3
    topPanel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    topPanel.Visible = false
    topPanel.ClipsDescendants = true
    topPanel.Parent = main
    local topPanelCorner = Instance.new("UICorner")
    topPanelCorner.CornerRadius = UDim.new(0, 10)
    topPanelCorner.Parent = topPanel

    -- Botones EXTRAS
    local extrasY = 10
    
    local ragdollBtn = Instance.new("TextButton")
    ragdollBtn.Size = UDim2.new(0.45, 0, 0, 28)
    ragdollBtn.Position = UDim2.new(0.05, 0, 0, extrasY)
    ragdollBtn.Text = "😆 RAGDOLL"
    ragdollBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    ragdollBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    ragdollBtn.BorderSizePixel = 1
    ragdollBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    ragdollBtn.Font = Enum.Font.GothamBold
    ragdollBtn.TextSize = 10
    ragdollBtn.Parent = topPanel
    local ragdollCorner = Instance.new("UICorner")
    ragdollCorner.CornerRadius = UDim.new(0, 6)
    ragdollCorner.Parent = ragdollBtn
    
    local infJumpBtn = Instance.new("TextButton")
    infJumpBtn.Size = UDim2.new(0.45, 0, 0, 28)
    infJumpBtn.Position = UDim2.new(0.5, 0, 0, extrasY)
    infJumpBtn.Text = "🦘 INF JUMP"
    infJumpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    infJumpBtn.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    infJumpBtn.BorderSizePixel = 1
    infJumpBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    infJumpBtn.Font = Enum.Font.GothamBold
    infJumpBtn.TextSize = 10
    infJumpBtn.Parent = topPanel
    local infCorner = Instance.new("UICorner")
    infCorner.CornerRadius = UDim.new(0, 6)
    infCorner.Parent = infJumpBtn
    
    local noclipBtn = Instance.new("TextButton")
    noclipBtn.Size = UDim2.new(0.45, 0, 0, 28)
    noclipBtn.Position = UDim2.new(0.05, 0, 0, extrasY + 35)
    noclipBtn.Text = "🚪 NOCLIP"
    noclipBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    noclipBtn.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    noclipBtn.BorderSizePixel = 1
    noclipBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    noclipBtn.Font = Enum.Font.GothamBold
    noclipBtn.TextSize = 10
    noclipBtn.Parent = topPanel
    local noclipCorner = Instance.new("UICorner")
    noclipCorner.CornerRadius = UDim.new(0, 6)
    noclipCorner.Parent = noclipBtn
    
    local espBtn = Instance.new("TextButton")
    espBtn.Size = UDim2.new(0.45, 0, 0, 28)
    espBtn.Position = UDim2.new(0.5, 0, 0, extrasY + 35)
    espBtn.Text = "👁️ ESP"
    espBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    espBtn.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    espBtn.BorderSizePixel = 1
    espBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    espBtn.Font = Enum.Font.GothamBold
    espBtn.TextSize = 10
    espBtn.Parent = topPanel
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 6)
    espCorner.Parent = espBtn
    
    local resetBtn = Instance.new("TextButton")
    resetBtn.Size = UDim2.new(0.45, 0, 0, 28)
    resetBtn.Position = UDim2.new(0.275, 0, 0, extrasY + 70)
    resetBtn.Text = "🔄 RESET"
    resetBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    resetBtn.BackgroundColor3 = Color3.fromRGB(180, 140, 100)
    resetBtn.BorderSizePixel = 1
    resetBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    resetBtn.Font = Enum.Font.GothamBold
    resetBtn.TextSize = 10
    resetBtn.Parent = topPanel
    local resetCorner = Instance.new("UICorner")
    resetCorner.CornerRadius = UDim.new(0, 6)
    resetCorner.Parent = resetBtn

    local topPanelHeight = 10 + 105

    -- Título
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -80, 0, 45)
    title.Position = UDim2.new(0, 0, 0, 45)
    title.Text = "WHATHEDOGDOIN"
    title.TextColor3 = Color3.fromRGB(200, 120, 80)
    title.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    title.BorderSizePixel = 3
    title.BorderColor3 = Color3.fromRGB(180, 130, 100)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 20
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = main
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 10)
    titleCorner.Parent = title

    -- Botones de escala
    local scaleDownBtn = Instance.new("TextButton")
    scaleDownBtn.Size = UDim2.new(0, 28, 0, 28)
    scaleDownBtn.Position = UDim2.new(1, -65, 0, 50)
    scaleDownBtn.Text = "•"
    scaleDownBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    scaleDownBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    scaleDownBtn.BorderSizePixel = 2
    scaleDownBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    scaleDownBtn.Font = Enum.Font.GothamBold
    scaleDownBtn.TextSize = 22
    scaleDownBtn.Parent = main
    local scaleDownCorner = Instance.new("UICorner")
    scaleDownCorner.CornerRadius = UDim.new(0, 8)
    scaleDownCorner.Parent = scaleDownBtn
    
    local scaleUpBtn = Instance.new("TextButton")
    scaleUpBtn.Size = UDim2.new(0, 28, 0, 28)
    scaleUpBtn.Position = UDim2.new(1, -33, 0, 50)
    scaleUpBtn.Text = "×"
    scaleUpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    scaleUpBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    scaleUpBtn.BorderSizePixel = 2
    scaleUpBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    scaleUpBtn.Font = Enum.Font.GothamBold
    scaleUpBtn.TextSize = 22
    scaleUpBtn.Parent = main
    local scaleUpCorner = Instance.new("UICorner")
    scaleUpCorner.CornerRadius = UDim.new(0, 8)
    scaleUpCorner.Parent = scaleUpBtn

    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 28, 0, 28)
    closeBtn.Position = UDim2.new(1, -98, 0, 50)
    closeBtn.Text = "X"
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.BackgroundColor3 = Color3.fromRGB(220, 120, 100)
    closeBtn.BorderSizePixel = 2
    closeBtn.BorderColor3 = Color3.fromRGB(150, 80, 60)
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 16
    closeBtn.Parent = main
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 8)
    closeCorner.Parent = closeBtn

    -- Contenido principal
    local yPos = 90
    local walkSpeed = 16
    local jumpPower = 50
    
    -- VELOCIDAD
    local speedLabel = Instance.new("TextLabel")
    speedLabel.Size = UDim2.new(0.4, 0, 0, 28)
    speedLabel.Position = UDim2.new(0.1, 0, 0, yPos)
    speedLabel.Text = "VEL: 16"
    speedLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    speedLabel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    speedLabel.BorderSizePixel = 1
    speedLabel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    speedLabel.Font = Enum.Font.GothamBold
    speedLabel.TextSize = 13
    speedLabel.Parent = main
    local speedCorner = Instance.new("UICorner")
    speedCorner.CornerRadius = UDim.new(0, 6)
    speedCorner.Parent = speedLabel

    local speedMinus = Instance.new("TextButton")
    speedMinus.Size = UDim2.new(0, 28, 0, 28)
    speedMinus.Position = UDim2.new(0.55, 0, 0, yPos)
    speedMinus.Text = "-"
    speedMinus.TextColor3 = Color3.fromRGB(255, 255, 255)
    speedMinus.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    speedMinus.BorderSizePixel = 1
    speedMinus.BorderColor3 = Color3.fromRGB(150, 80, 60)
    speedMinus.Font = Enum.Font.GothamBold
    speedMinus.TextSize = 18
    speedMinus.Parent = main
    local speedMinusCorner = Instance.new("UICorner")
    speedMinusCorner.CornerRadius = UDim.new(0, 6)
    speedMinusCorner.Parent = speedMinus

    local speedPlus = Instance.new("TextButton")
    speedPlus.Size = UDim2.new(0, 28, 0, 28)
    speedPlus.Position = UDim2.new(0.7, 0, 0, yPos)
    speedPlus.Text = "+"
    speedPlus.TextColor3 = Color3.fromRGB(255, 255, 255)
    speedPlus.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    speedPlus.BorderSizePixel = 1
    speedPlus.BorderColor3 = Color3.fromRGB(100, 130, 100)
    speedPlus.Font = Enum.Font.GothamBold
    speedPlus.TextSize = 18
    speedPlus.Parent = main
    local speedPlusCorner = Instance.new("UICorner")
    speedPlusCorner.CornerRadius = UDim.new(0, 6)
    speedPlusCorner.Parent = speedPlus

    yPos = yPos + 33

    -- SALTO
    local jumpLabel = Instance.new("TextLabel")
    jumpLabel.Size = UDim2.new(0.4, 0, 0, 28)
    jumpLabel.Position = UDim2.new(0.1, 0, 0, yPos)
    jumpLabel.Text = "SALTO: 50"
    jumpLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    jumpLabel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    jumpLabel.BorderSizePixel = 1
    jumpLabel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    jumpLabel.Font = Enum.Font.GothamBold
    jumpLabel.TextSize = 13
    jumpLabel.Parent = main
    local jumpCorner = Instance.new("UICorner")
    jumpCorner.CornerRadius = UDim.new(0, 6)
    jumpCorner.Parent = jumpLabel

    local jumpMinus = Instance.new("TextButton")
    jumpMinus.Size = UDim2.new(0, 28, 0, 28)
    jumpMinus.Position = UDim2.new(0.55, 0, 0, yPos)
    jumpMinus.Text = "-"
    jumpMinus.TextColor3 = Color3.fromRGB(255, 255, 255)
    jumpMinus.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    jumpMinus.BorderSizePixel = 1
    jumpMinus.BorderColor3 = Color3.fromRGB(150, 80, 60)
    jumpMinus.Font = Enum.Font.GothamBold
    jumpMinus.TextSize = 18
    jumpMinus.Parent = main
    local jumpMinusCorner = Instance.new("UICorner")
    jumpMinusCorner.CornerRadius = UDim.new(0, 6)
    jumpMinusCorner.Parent = jumpMinus

    local jumpPlus = Instance.new("TextButton")
    jumpPlus.Size = UDim2.new(0, 28, 0, 28)
    jumpPlus.Position = UDim2.new(0.7, 0, 0, yPos)
    jumpPlus.Text = "+"
    jumpPlus.TextColor3 = Color3.fromRGB(255, 255, 255)
    jumpPlus.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    jumpPlus.BorderSizePixel = 1
    jumpPlus.BorderColor3 = Color3.fromRGB(100, 130, 100)
    jumpPlus.Font = Enum.Font.GothamBold
    jumpPlus.TextSize = 18
    jumpPlus.Parent = main
    local jumpPlusCorner = Instance.new("UICorner")
    jumpPlusCorner.CornerRadius = UDim.new(0, 6)
    jumpPlusCorner.Parent = jumpPlus

    yPos = yPos + 38

    -- VUELO
    local flySection = Instance.new("Frame")
    flySection.Size = UDim2.new(0.9, 0, 0, 90)
    flySection.Position = UDim2.new(0.05, 0, 0, yPos)
    flySection.BackgroundColor3 = Color3.fromRGB(225, 190, 160)
    flySection.BorderSizePixel = 3
    flySection.BorderColor3 = Color3.fromRGB(180, 130, 100)
    flySection.Parent = main
    local flySectionCorner = Instance.new("UICorner")
    flySectionCorner.CornerRadius = UDim.new(0, 10)
    flySectionCorner.Parent = flySection

    local flyTitle = Instance.new("TextLabel")
    flyTitle.Size = UDim2.new(1, 0, 0, 22)
    flyTitle.Position = UDim2.new(0, 0, 0, 3)
    flyTitle.Text = "✈️ VUELO"
    flyTitle.TextColor3 = Color3.fromRGB(200, 120, 80)
    flyTitle.BackgroundTransparency = 1
    flyTitle.Font = Enum.Font.GothamBold
    flyTitle.TextSize = 14
    flyTitle.Parent = flySection

    local flyBtn = Instance.new("TextButton")
    flyBtn.Size = UDim2.new(0.3, 0, 0, 32)
    flyBtn.Position = UDim2.new(0.1, 0, 0, 28)
    flyBtn.Text = "FLY"
    flyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    flyBtn.BorderSizePixel = 2
    flyBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    flyBtn.Font = Enum.Font.Gotham
    flyBtn.TextSize = 13
    flyBtn.Parent = flySection
    local flyBtnCorner = Instance.new("UICorner")
    flyBtnCorner.CornerRadius = UDim.new(0, 6)
    flyBtnCorner.Parent = flyBtn

    local flySpeedLabel = Instance.new("TextLabel")
    flySpeedLabel.Size = UDim2.new(0.15, 0, 0, 32)
    flySpeedLabel.Position = UDim2.new(0.45, 0, 0, 28)
    flySpeedLabel.Text = string.format("%.1f", _G.flySpeed)
    flySpeedLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    flySpeedLabel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    flySpeedLabel.BorderSizePixel = 1
    flySpeedLabel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    flySpeedLabel.Font = Enum.Font.GothamBold
    flySpeedLabel.TextSize = 15
    flySpeedLabel.Parent = flySection
    local flySpeedCorner = Instance.new("UICorner")
    flySpeedCorner.CornerRadius = UDim.new(0, 6)
    flySpeedCorner.Parent = flySpeedLabel

    local flyMinus = Instance.new("TextButton")
    flyMinus.Size = UDim2.new(0, 24, 0, 24)
    flyMinus.Position = UDim2.new(0.6, 0, 0, 32)
    flyMinus.Text = "-"
    flyMinus.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyMinus.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    flyMinus.BorderSizePixel = 1
    flyMinus.BorderColor3 = Color3.fromRGB(150, 80, 60)
    flyMinus.Font = Enum.Font.GothamBold
    flyMinus.TextSize = 16
    flyMinus.Parent = flySection
    local flyMinusCorner = Instance.new("UICorner")
    flyMinusCorner.CornerRadius = UDim.new(0, 6)
    flyMinusCorner.Parent = flyMinus

    local flyPlus = Instance.new("TextButton")
    flyPlus.Size = UDim2.new(0, 24, 0, 24)
    flyPlus.Position = UDim2.new(0.7, 0, 0, 32)
    flyPlus.Text = "+"
    flyPlus.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyPlus.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    flyPlus.BorderSizePixel = 1
    flyPlus.BorderColor3 = Color3.fromRGB(100, 130, 100)
    flyPlus.Font = Enum.Font.GothamBold
    flyPlus.TextSize = 16
    flyPlus.Parent = flySection
    local flyPlusCorner = Instance.new("UICorner")
    flyPlusCorner.CornerRadius = UDim.new(0, 6)
    flyPlusCorner.Parent = flyPlus

    yPos = yPos + 100

    -- FLING CON CONTROL DE VELOCIDAD
    local flingSection = Instance.new("Frame")
    flingSection.Size = UDim2.new(0.9, 0, 0, 80)
    flingSection.Position = UDim2.new(0.05, 0, 0, yPos)
    flingSection.BackgroundColor3 = Color3.fromRGB(225, 190, 160)
    flingSection.BorderSizePixel = 2
    flingSection.BorderColor3 = Color3.fromRGB(180, 130, 100)
    flingSection.Parent = main
    local flingSectionCorner = Instance.new("UICorner")
    flingSectionCorner.CornerRadius = UDim.new(0, 10)
    flingSectionCorner.Parent = flingSection

    local flingTitle = Instance.new("TextLabel")
    flingTitle.Size = UDim2.new(1, 0, 0, 20)
    flingTitle.Position = UDim2.new(0, 0, 0, 3)
    flingTitle.Text = "🌀 FLING"
    flingTitle.TextColor3 = Color3.fromRGB(200, 120, 80)
    flingTitle.BackgroundTransparency = 1
    flingTitle.Font = Enum.Font.GothamBold
    flingTitle.TextSize = 12
    flingTitle.Parent = flingSection

    -- Cuadro de velocidad
    local flingSpeedBox = Instance.new("TextBox")
    flingSpeedBox.Size = UDim2.new(0, 50, 0, 25)
    flingSpeedBox.Position = UDim2.new(0.05, 0, 0, 28)
    flingSpeedBox.Text = tostring(_G.flingSpeed)
    flingSpeedBox.TextColor3 = Color3.fromRGB(0, 0, 0)
    flingSpeedBox.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    flingSpeedBox.BorderSizePixel = 1
    flingSpeedBox.BorderColor3 = Color3.fromRGB(150, 100, 70)
    flingSpeedBox.Font = Enum.Font.GothamBold
    flingSpeedBox.TextSize = 12
    flingSpeedBox.PlaceholderText = "Vel"
    flingSpeedBox.Parent = flingSection
    local flingSpeedCorner = Instance.new("UICorner")
    flingSpeedCorner.CornerRadius = UDim.new(0, 4)
    flingSpeedCorner.Parent = flingSpeedBox
    
    flingSpeedBox.FocusLost:Connect(function()
        local newSpeed = tonumber(flingSpeedBox.Text)
        if newSpeed and newSpeed > 0 then
            _G.flingSpeed = newSpeed
        else
            flingSpeedBox.Text = tostring(_G.flingSpeed)
        end
    end)

    -- FLING
    local flingBtn = Instance.new("TextButton")
    flingBtn.Size = UDim2.new(0.3, 0, 0, 25)
    flingBtn.Position = UDim2.new(0.25, 0, 0, 28)
    flingBtn.Text = "FLING: OFF"
    flingBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
    flingBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    flingBtn.BorderSizePixel = 2
    flingBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    flingBtn.Font = Enum.Font.SourceSansItalic
    flingBtn.TextSize = 12
    flingBtn.Parent = flingSection
    local flingBtnCorner = Instance.new("UICorner")
    flingBtnCorner.CornerRadius = UDim.new(0, 6)
    flingBtnCorner.Parent = flingBtn

    -- ANTIFLING
    local antiBtn = Instance.new("TextButton")
    antiBtn.Size = UDim2.new(0.3, 0, 0, 25)
    antiBtn.Position = UDim2.new(0.6, 0, 0, 28)
    antiBtn.Text = "ANTIFLING: OFF"
    antiBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    antiBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    antiBtn.BorderSizePixel = 2
    antiBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    antiBtn.Font = Enum.Font.Gotham
    antiBtn.TextSize = 10
    antiBtn.Parent = flingSection
    local antiBtnCorner = Instance.new("UICorner")
    antiBtnCorner.CornerRadius = UDim.new(0, 6)
    antiBtnCorner.Parent = antiBtn

    -- Botones de ajuste de velocidad
    local flingMinus = Instance.new("TextButton")
    flingMinus.Size = UDim2.new(0, 20, 0, 20)
    flingMinus.Position = UDim2.new(0.05, 0, 0, 55)
    flingMinus.Text = "-"
    flingMinus.TextColor3 = Color3.fromRGB(255, 255, 255)
    flingMinus.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    flingMinus.BorderSizePixel = 1
    flingMinus.BorderColor3 = Color3.fromRGB(150, 80, 60)
    flingMinus.Font = Enum.Font.GothamBold
    flingMinus.TextSize = 14
    flingMinus.Parent = flingSection
    local flingMinusCorner = Instance.new("UICorner")
    flingMinusCorner.CornerRadius = UDim.new(0, 4)
    flingMinusCorner.Parent = flingMinus
    
    local flingPlus = Instance.new("TextButton")
    flingPlus.Size = UDim2.new(0, 20, 0, 20)
    flingPlus.Position = UDim2.new(0.15, 0, 0, 55)
    flingPlus.Text = "+"
    flingPlus.TextColor3 = Color3.fromRGB(255, 255, 255)
    flingPlus.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    flingPlus.BorderSizePixel = 1
    flingPlus.BorderColor3 = Color3.fromRGB(100, 130, 100)
    flingPlus.Font = Enum.Font.GothamBold
    flingPlus.TextSize = 14
    flingPlus.Parent = flingSection
    local flingPlusCorner = Instance.new("UICorner")
    flingPlusCorner.CornerRadius = UDim.new(0, 4)
    flingPlusCorner.Parent = flingPlus

    yPos = yPos + 90

    -- TELEPORT
    local usernameBox = Instance.new("TextBox")
    usernameBox.Size = UDim2.new(0.6, 0, 0, 28)
    usernameBox.Position = UDim2.new(0.2, 0, 0, yPos)
    usernameBox.PlaceholderText = "username"
    usernameBox.Text = ""
    usernameBox.TextColor3 = Color3.fromRGB(0, 0, 0)
    usernameBox.PlaceholderColor3 = Color3.fromRGB(150, 100, 70)
    usernameBox.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    usernameBox.BorderSizePixel = 2
    usernameBox.BorderColor3 = Color3.fromRGB(180, 130, 100)
    usernameBox.Font = Enum.Font.Gotham
    usernameBox.TextSize = 13
    usernameBox.Parent = main
    local userCorner = Instance.new("UICorner")
    userCorner.CornerRadius = UDim.new(0, 6)
    userCorner.Parent = usernameBox

    yPos = yPos + 33

    local tpBtn = Instance.new("TextButton")
    tpBtn.Size = UDim2.new(0.6, 0, 0, 32)
    tpBtn.Position = UDim2.new(0.2, 0, 0, yPos)
    tpBtn.Text = "🚀 TELEPORT"
    tpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    tpBtn.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    tpBtn.BorderSizePixel = 2
    tpBtn.BorderColor3 = Color3.fromRGB(100, 120, 150)
    tpBtn.Font = Enum.Font.GothamBold
    tpBtn.TextSize = 13
    tpBtn.Parent = main
    local tpCorner = Instance.new("UICorner")
    tpCorner.CornerRadius = UDim.new(0, 6)
    tpCorner.Parent = tpBtn

    yPos = yPos + 37

    local clickTPBtn = Instance.new("TextButton")
    clickTPBtn.Size = UDim2.new(0.6, 0, 0, 32)
    clickTPBtn.Position = UDim2.new(0.2, 0, 0, yPos)
    clickTPBtn.Text = "📍 CLICK TP"
    clickTPBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    clickTPBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 180)
    clickTPBtn.BorderSizePixel = 2
    clickTPBtn.BorderColor3 = Color3.fromRGB(150, 100, 130)
    clickTPBtn.Font = Enum.Font.GothamBold
    clickTPBtn.TextSize = 13
    clickTPBtn.Parent = main
    local clickCorner = Instance.new("UICorner")
    clickCorner.CornerRadius = UDim.new(0, 6)
    clickCorner.Parent = clickTPBtn

    yPos = yPos + 42

    -- FLY CAR
    local flyCarFrame = Instance.new("Frame")
    flyCarFrame.Size = UDim2.new(0.9, 0, 0, 95)
    flyCarFrame.Position = UDim2.new(0.05, 0, 0, yPos)
    flyCarFrame.BackgroundColor3 = Color3.fromRGB(225, 190, 160)
    flyCarFrame.BorderSizePixel = 3
    flyCarFrame.BorderColor3 = Color3.fromRGB(180, 130, 100)
    flyCarFrame.Parent = main
    local flyCarFrameCorner = Instance.new("UICorner")
    flyCarFrameCorner.CornerRadius = UDim.new(0, 10)
    flyCarFrameCorner.Parent = flyCarFrame

    local flyCarLabel = Instance.new("TextLabel")
    flyCarLabel.Size = UDim2.new(1, 0, 0, 22)
    flyCarLabel.Position = UDim2.new(0, 0, 0, 3)
    flyCarLabel.Text = "🚗 FLY CAR"
    flyCarLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    flyCarLabel.BackgroundTransparency = 1
    flyCarLabel.Font = Enum.Font.GothamBold
    flyCarLabel.TextSize = 14
    flyCarLabel.Parent = flyCarFrame

    local flyCarBtn = Instance.new("TextButton")
    flyCarBtn.Size = UDim2.new(0.3, 0, 0, 28)
    flyCarBtn.Position = UDim2.new(0.1, 0, 0, 28)
    flyCarBtn.Text = "OFF"
    flyCarBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyCarBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 120)
    flyCarBtn.BorderSizePixel = 2
    flyCarBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    flyCarBtn.Font = Enum.Font.GothamBold
    flyCarBtn.TextSize = 13
    flyCarBtn.Parent = flyCarFrame
    local flyCarBtnCorner = Instance.new("UICorner")
    flyCarBtnCorner.CornerRadius = UDim.new(0, 6)
    flyCarBtnCorner.Parent = flyCarBtn

    local flyCarSpeedLabel = Instance.new("TextLabel")
    flyCarSpeedLabel.Size = UDim2.new(0.15, 0, 0, 28)
    flyCarSpeedLabel.Position = UDim2.new(0.45, 0, 0, 28)
    flyCarSpeedLabel.Text = tostring(_G.flyCarSpeed)
    flyCarSpeedLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    flyCarSpeedLabel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    flyCarSpeedLabel.BorderSizePixel = 1
    flyCarSpeedLabel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    flyCarSpeedLabel.Font = Enum.Font.GothamBold
    flyCarSpeedLabel.TextSize = 15
    flyCarSpeedLabel.Parent = flyCarFrame
    local flyCarSpeedCorner = Instance.new("UICorner")
    flyCarSpeedCorner.CornerRadius = UDim.new(0, 6)
    flyCarSpeedCorner.Parent = flyCarSpeedLabel

    local flyCarMinus = Instance.new("TextButton")
    flyCarMinus.Size = UDim2.new(0, 24, 0, 24)
    flyCarMinus.Position = UDim2.new(0.6, 0, 0, 30)
    flyCarMinus.Text = "-"
    flyCarMinus.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyCarMinus.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    flyCarMinus.BorderSizePixel = 1
    flyCarMinus.BorderColor3 = Color3.fromRGB(150, 80, 60)
    flyCarMinus.Font = Enum.Font.GothamBold
    flyCarMinus.TextSize = 16
    flyCarMinus.Parent = flyCarFrame
    local flyCarMinusCorner = Instance.new("UICorner")
    flyCarMinusCorner.CornerRadius = UDim.new(0, 6)
    flyCarMinusCorner.Parent = flyCarMinus

    local flyCarPlus = Instance.new("TextButton")
    flyCarPlus.Size = UDim2.new(0, 24, 0, 24)
    flyCarPlus.Position = UDim2.new(0.7, 0, 0, 30)
    flyCarPlus.Text = "+"
    flyCarPlus.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyCarPlus.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    flyCarPlus.BorderSizePixel = 1
    flyCarPlus.BorderColor3 = Color3.fromRGB(100, 130, 100)
    flyCarPlus.Font = Enum.Font.GothamBold
    flyCarPlus.TextSize = 16
    flyCarPlus.Parent = flyCarFrame
    local flyCarPlusCorner = Instance.new("UICorner")
    flyCarPlusCorner.CornerRadius = UDim.new(0, 6)
    flyCarPlusCorner.Parent = flyCarPlus

    local flyCarUp = Instance.new("TextButton")
    flyCarUp.Size = UDim2.new(0.35, 0, 0, 28)
    flyCarUp.Position = UDim2.new(0.1, 0, 0, 60)
    flyCarUp.Text = "▲"
    flyCarUp.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyCarUp.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    flyCarUp.BorderSizePixel = 2
    flyCarUp.BorderColor3 = Color3.fromRGB(100, 120, 150)
    flyCarUp.Font = Enum.Font.GothamBold
    flyCarUp.TextSize = 18
    flyCarUp.Parent = flyCarFrame
    local flyCarUpCorner = Instance.new("UICorner")
    flyCarUpCorner.CornerRadius = UDim.new(0, 6)
    flyCarUpCorner.Parent = flyCarUp

    local flyCarDown = Instance.new("TextButton")
    flyCarDown.Size = UDim2.new(0.35, 0, 0, 28)
    flyCarDown.Position = UDim2.new(0.55, 0, 0, 60)
    flyCarDown.Text = "▼"
    flyCarDown.TextColor3 = Color3.fromRGB(255, 255, 255)
    flyCarDown.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    flyCarDown.BorderSizePixel = 2
    flyCarDown.BorderColor3 = Color3.fromRGB(100, 120, 150)
    flyCarDown.Font = Enum.Font.GothamBold
    flyCarDown.TextSize = 18
    flyCarDown.Parent = flyCarFrame
    local flyCarDownCorner = Instance.new("UICorner")
    flyCarDownCorner.CornerRadius = UDim.new(0, 6)
    flyCarDownCorner.Parent = flyCarDown

    yPos = yPos + 100

    -- Panel TROLL (SIN ESPACIO EXTRA ABAJO)
    local trollBtn = Instance.new("TextButton")
    trollBtn.Size = UDim2.new(0.9, 0, 0, 35)
    trollBtn.Position = UDim2.new(0.05, 0, 0, yPos)
    trollBtn.Text = "👹 TROLL ▼"
    trollBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    trollBtn.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    trollBtn.BorderSizePixel = 2
    trollBtn.BorderColor3 = Color3.fromRGB(150, 80, 60)
    trollBtn.Font = Enum.Font.GothamBold
    trollBtn.TextSize = 13
    trollBtn.Parent = main
    local trollBtnCorner = Instance.new("UICorner")
    trollBtnCorner.CornerRadius = UDim.new(0, 6)
    trollBtnCorner.Parent = trollBtn

    local trollPanel = Instance.new("Frame")
    trollPanel.Size = UDim2.new(0.9, 0, 0, 0)
    trollPanel.Position = UDim2.new(0.05, 0, 0, yPos + 40)
    trollPanel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    trollPanel.BorderSizePixel = 2
    trollPanel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    trollPanel.Visible = false
    trollPanel.ClipsDescendants = true
    trollPanel.Parent = main
    local trollPanelCorner = Instance.new("UICorner")
    trollPanelCorner.CornerRadius = UDim.new(0, 8)
    trollPanelCorner.Parent = trollPanel

    local trollY = 5
    local bangBtn = Instance.new("TextButton")
    bangBtn.Size = UDim2.new(0.45, 0, 0, 28)
    bangBtn.Position = UDim2.new(0.05, 0, 0, trollY)
    bangBtn.Text = "💥 BANG"
    bangBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    bangBtn.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    bangBtn.BorderSizePixel = 1
    bangBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    bangBtn.Font = Enum.Font.GothamBold
    bangBtn.TextSize = 10
    bangBtn.Parent = trollPanel
    local bangCorner = Instance.new("UICorner")
    bangCorner.CornerRadius = UDim.new(0, 6)
    bangCorner.Parent = bangBtn

    local faceBtn = Instance.new("TextButton")
    faceBtn.Size = UDim2.new(0.45, 0, 0, 28)
    faceBtn.Position = UDim2.new(0.5, 0, 0, trollY)
    faceBtn.Text = "😳 FACE"
    faceBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    faceBtn.BackgroundColor3 = Color3.fromRGB(150, 170, 200)
    faceBtn.BorderSizePixel = 1
    faceBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    faceBtn.Font = Enum.Font.GothamBold
    faceBtn.TextSize = 10
    faceBtn.Parent = trollPanel
    local faceCorner = Instance.new("UICorner")
    faceCorner.CornerRadius = UDim.new(0, 6)
    faceCorner.Parent = faceBtn

    local sitBtn = Instance.new("TextButton")
    sitBtn.Size = UDim2.new(0.45, 0, 0, 28)
    sitBtn.Position = UDim2.new(0.05, 0, 0, trollY + 30)
    sitBtn.Text = "🪑 SIT"
    sitBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    sitBtn.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    sitBtn.BorderSizePixel = 1
    sitBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    sitBtn.Font = Enum.Font.GothamBold
    sitBtn.TextSize = 10
    sitBtn.Parent = trollPanel
    local sitCorner = Instance.new("UICorner")
    sitCorner.CornerRadius = UDim.new(0, 6)
    sitCorner.Parent = sitBtn

    local suckBtn = Instance.new("TextButton")
    suckBtn.Size = UDim2.new(0.45, 0, 0, 28)
    suckBtn.Position = UDim2.new(0.5, 0, 0, trollY + 30)
    suckBtn.Text = "👅 SUCK"
    suckBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    suckBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 180)
    suckBtn.BorderSizePixel = 1
    suckBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    suckBtn.Font = Enum.Font.GothamBold
    suckBtn.TextSize = 10
    suckBtn.Parent = trollPanel
    local suckCorner = Instance.new("UICorner")
    suckCorner.CornerRadius = UDim.new(0, 6)
    suckCorner.Parent = suckBtn

    local jerkBtn = Instance.new("TextButton")
    jerkBtn.Size = UDim2.new(0.45, 0, 0, 28)
    jerkBtn.Position = UDim2.new(0.275, 0, 0, trollY + 60)
    jerkBtn.Text = "🍆 JERK (R6)"
    jerkBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    jerkBtn.BackgroundColor3 = Color3.fromRGB(180, 140, 200)
    jerkBtn.BorderSizePixel = 1
    jerkBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    jerkBtn.Font = Enum.Font.GothamBold
    jerkBtn.TextSize = 10
    jerkBtn.Parent = trollPanel
    local jerkCorner = Instance.new("UICorner")
    jerkCorner.CornerRadius = UDim.new(0, 6)
    jerkCorner.Parent = jerkBtn


    -- Panel izquierdo (ANIMACIONES)
    local leftPanelBtn = Instance.new("TextButton")
    leftPanelBtn.Size = UDim2.new(0, 35, 0, 35)
    leftPanelBtn.Position = UDim2.new(0, -10, 0.5, -17.5)
    leftPanelBtn.Text = "▶"
    leftPanelBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    leftPanelBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    leftPanelBtn.BorderSizePixel = 2
    leftPanelBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    leftPanelBtn.Font = Enum.Font.GothamBold
    leftPanelBtn.TextSize = 22
    leftPanelBtn.ZIndex = 10
    leftPanelBtn.Parent = main
    local leftBtnCorner = Instance.new("UICorner")
    leftBtnCorner.CornerRadius = UDim.new(0, 10)
    leftBtnCorner.Parent = leftPanelBtn

    local leftPanel = Instance.new("Frame")
    leftPanel.Size = UDim2.new(0, 250, 0, 440)
    leftPanel.Position = UDim2.new(0, -255, 0, 90)
    leftPanel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    leftPanel.BorderSizePixel = 3
    leftPanel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    leftPanel.Visible = false
    leftPanel.ClipsDescendants = true
    leftPanel.Parent = main
    local leftPanelCorner = Instance.new("UICorner")
    leftPanelCorner.CornerRadius = UDim.new(0, 10)
    leftPanelCorner.Parent = leftPanel

    local leftTitle = Instance.new("TextLabel")
    leftTitle.Size = UDim2.new(1, 0, 0, 35)
    leftTitle.Text = "ANIMACIONES"
    leftTitle.TextColor3 = Color3.fromRGB(200, 120, 80)
    leftTitle.BackgroundColor3 = Color3.fromRGB(215, 175, 140)
    leftTitle.BorderSizePixel = 2
    leftTitle.BorderColor3 = Color3.fromRGB(160, 110, 80)
    leftTitle.Font = Enum.Font.GothamBold
    leftTitle.TextSize = 14
    leftTitle.Parent = leftPanel
    local leftTitleCorner = Instance.new("UICorner")
    leftTitleCorner.CornerRadius = UDim.new(0, 10)
    leftTitleCorner.Parent = leftTitle

    local animScroll = Instance.new("ScrollingFrame")
    animScroll.Size = UDim2.new(0.9, 0, 0.85, 0)
    animScroll.Position = UDim2.new(0.05, 0, 0.1, 0)
    animScroll.BackgroundColor3 = Color3.fromRGB(245, 225, 200)
    animScroll.BorderSizePixel = 1
    animScroll.BorderColor3 = Color3.fromRGB(180, 130, 100)
    animScroll.ScrollBarThickness = 6
    animScroll.Parent = leftPanel

    local animY = 5
    for animType, anims in pairs(Animations) do
        local typeLabel = Instance.new("TextLabel")
        typeLabel.Size = UDim2.new(1, 0, 0, 22)
        typeLabel.Position = UDim2.new(0, 0, 0, animY)
        typeLabel.Text = "=== " .. animType:upper() .. " ==="
        typeLabel.TextColor3 = Color3.fromRGB(180, 100, 60)
        typeLabel.BackgroundTransparency = 1
        typeLabel.Font = Enum.Font.GothamBold
        typeLabel.TextSize = 11
        typeLabel.Parent = animScroll
        animY = animY + 24

        for name, _ in pairs(anims) do
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0.9, 0, 0, 24)
            btn.Position = UDim2.new(0.05, 0, 0, animY)
            btn.Text = name
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
            btn.BorderSizePixel = 1
            btn.BorderColor3 = Color3.fromRGB(150, 100, 70)
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 10
            btn.Parent = animScroll
            local btnCorner = Instance.new("UICorner")
            btnCorner.CornerRadius = UDim.new(0, 6)
            btnCorner.Parent = btn

            btn.MouseButton1Click:Connect(function()
                setAnimation(animType, name)
            end)

            animY = animY + 26
        end
        animY = animY + 5
    end
    animScroll.CanvasSize = UDim2.new(0, 0, 0, animY)

    -- Panel derecho (MOVE PARTS)
    local rightPanelBtn = Instance.new("TextButton")
    rightPanelBtn.Size = UDim2.new(0, 35, 0, 35)
    rightPanelBtn.Position = UDim2.new(1, -20, 0.5, -17.5)
    rightPanelBtn.Text = "◀"
    rightPanelBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    rightPanelBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    rightPanelBtn.BorderSizePixel = 2
    rightPanelBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    rightPanelBtn.Font = Enum.Font.GothamBold
    rightPanelBtn.TextSize = 22
    rightPanelBtn.ZIndex = 10
    rightPanelBtn.Parent = main
    local rightBtnCorner = Instance.new("UICorner")
    rightBtnCorner.CornerRadius = UDim.new(0, 10)
    rightBtnCorner.Parent = rightPanelBtn

    local rightPanel = Instance.new("Frame")
    rightPanel.Size = UDim2.new(0, 300, 0, 520)
    rightPanel.Position = UDim2.new(1, 5, 0, 90)
    rightPanel.BackgroundColor3 = Color3.fromRGB(235, 200, 170)
    rightPanel.BorderSizePixel = 3
    rightPanel.BorderColor3 = Color3.fromRGB(180, 130, 100)
    rightPanel.Visible = false
    rightPanel.ClipsDescendants = true
    rightPanel.Parent = main
    local rightPanelCorner = Instance.new("UICorner")
    rightPanelCorner.CornerRadius = UDim.new(0, 10)
    rightPanelCorner.Parent = rightPanel

    local rightTitle = Instance.new("TextLabel")
    rightTitle.Size = UDim2.new(1, 0, 0, 35)
    rightTitle.Text = "MOVE PARTS"
    rightTitle.TextColor3 = Color3.fromRGB(200, 120, 80)
    rightTitle.BackgroundColor3 = Color3.fromRGB(215, 175, 140)
    rightTitle.BorderSizePixel = 2
    rightTitle.BorderColor3 = Color3.fromRGB(160, 110, 80)
    rightTitle.Font = Enum.Font.GothamBold
    rightTitle.TextSize = 14
    rightTitle.Parent = rightPanel
    local rightTitleCorner = Instance.new("UICorner")
    rightTitleCorner.CornerRadius = UDim.new(0, 10)
    rightTitleCorner.Parent = rightTitle

    -- DPAD
    local arrowY = 40
    local btnSize = 42

    -- ▲ (Arriba)
    local upBtn = Instance.new("TextButton")
    upBtn.Size = UDim2.new(0, btnSize, 0, btnSize)
    upBtn.Position = UDim2.new(0.5, -btnSize/2, 0, arrowY)
    upBtn.Text = "▲"
    upBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    upBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    upBtn.BorderSizePixel = 2
    upBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    upBtn.Font = Enum.Font.GothamBold
    upBtn.TextSize = 22
    upBtn.Parent = rightPanel
    local upCorner = Instance.new("UICorner")
    upCorner.CornerRadius = UDim.new(0, 8)
    upCorner.Parent = upBtn

    -- ▼ (Abajo)
    local downBtn = Instance.new("TextButton")
    downBtn.Size = UDim2.new(0, btnSize, 0, btnSize)
    downBtn.Position = UDim2.new(0.5, -btnSize/2, 0, arrowY + btnSize + 10)
    downBtn.Text = "▼"
    downBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    downBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    downBtn.BorderSizePixel = 2
    downBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    downBtn.Font = Enum.Font.GothamBold
    downBtn.TextSize = 22
    downBtn.Parent = rightPanel
    local downCorner = Instance.new("UICorner")
    downCorner.CornerRadius = UDim.new(0, 8)
    downCorner.Parent = downBtn

    -- ◀ (Izquierda)
    local leftBtn = Instance.new("TextButton")
    leftBtn.Size = UDim2.new(0, btnSize, 0, btnSize)
    leftBtn.Position = UDim2.new(0.5, -btnSize*1.5 - 10, 0, arrowY + btnSize/2)
    leftBtn.Text = "◀"
    leftBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    leftBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    leftBtn.BorderSizePixel = 2
    leftBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    leftBtn.Font = Enum.Font.GothamBold
    leftBtn.TextSize = 22
    leftBtn.Parent = rightPanel
    local leftCorner = Instance.new("UICorner")
    leftCorner.CornerRadius = UDim.new(0, 8)
    leftCorner.Parent = leftBtn

    -- ▶ (Derecha)
    local rightBtn = Instance.new("TextButton")
    rightBtn.Size = UDim2.new(0, btnSize, 0, btnSize)
    rightBtn.Position = UDim2.new(0.5, btnSize/2 + 10, 0, arrowY + btnSize/2)
    rightBtn.Text = "▶"
    rightBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    rightBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    rightBtn.BorderSizePixel = 2
    rightBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    rightBtn.Font = Enum.Font.GothamBold
    rightBtn.TextSize = 22
    rightBtn.Parent = rightPanel
    local rightCorner = Instance.new("UICorner")
    rightCorner.CornerRadius = UDim.new(0, 8)
    rightCorner.Parent = rightBtn

    -- + (Adelante)
    local fwdBtn = Instance.new("TextButton")
    fwdBtn.Size = UDim2.new(0, btnSize, 0, btnSize)
    fwdBtn.Position = UDim2.new(0.5, -btnSize*1.5 - 10, 0, arrowY + btnSize*1.5 + 15)
    fwdBtn.Text = "+"
    fwdBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    fwdBtn.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    fwdBtn.BorderSizePixel = 2
    fwdBtn.BorderColor3 = Color3.fromRGB(100, 130, 100)
    fwdBtn.Font = Enum.Font.GothamBold
    fwdBtn.TextSize = 22
    fwdBtn.Parent = rightPanel
    local fwdCorner = Instance.new("UICorner")
    fwdCorner.CornerRadius = UDim.new(0, 8)
    fwdCorner.Parent = fwdBtn

    -- - (Atrás)
    local bwdBtn = Instance.new("TextButton")
    bwdBtn.Size = UDim2.new(0, btnSize, 0, btnSize)
    bwdBtn.Position = UDim2.new(0.5, btnSize/2 + 10, 0, arrowY + btnSize*1.5 + 15)
    bwdBtn.Text = "-"
    bwdBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    bwdBtn.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    bwdBtn.BorderSizePixel = 2
    bwdBtn.BorderColor3 = Color3.fromRGB(150, 80, 60)
    bwdBtn.Font = Enum.Font.GothamBold
    bwdBtn.TextSize = 22
    bwdBtn.Parent = rightPanel
    local bwdCorner = Instance.new("UICorner")
    bwdCorner.CornerRadius = UDim.new(0, 8)
    bwdCorner.Parent = bwdBtn

    -- Velocidad Objeto
    local objSpeedLabel = Instance.new("TextLabel")
    objSpeedLabel.Size = UDim2.new(1, 0, 0, 22)
    objSpeedLabel.Position = UDim2.new(0, 0, 0, 240)
    objSpeedLabel.Text = "⚡ VEL. OBJETO: ".._G.floatSpeed
    objSpeedLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    objSpeedLabel.BackgroundTransparency = 1
    objSpeedLabel.Font = Enum.Font.GothamBold
    objSpeedLabel.TextSize = 13
    objSpeedLabel.Parent = rightPanel

    local objMinus = Instance.new("TextButton")
    objMinus.Size = UDim2.new(0, 32, 0, 28)
    objMinus.Position = UDim2.new(0.3, 0, 0, 265)
    objMinus.Text = "-"
    objMinus.TextColor3 = Color3.fromRGB(255, 255, 255)
    objMinus.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    objMinus.BorderSizePixel = 2
    objMinus.BorderColor3 = Color3.fromRGB(150, 80, 60)
    objMinus.Font = Enum.Font.GothamBold
    objMinus.TextSize = 18
    objMinus.Parent = rightPanel
    local objMinusCorner = Instance.new("UICorner")
    objMinusCorner.CornerRadius = UDim.new(0, 6)
    objMinusCorner.Parent = objMinus

    local objPlus = Instance.new("TextButton")
    objPlus.Size = UDim2.new(0, 32, 0, 28)
    objPlus.Position = UDim2.new(0.6, 0, 0, 265)
    objPlus.Text = "+"
    objPlus.TextColor3 = Color3.fromRGB(255, 255, 255)
    objPlus.BackgroundColor3 = Color3.fromRGB(150, 180, 150)
    objPlus.BorderSizePixel = 2
    objPlus.BorderColor3 = Color3.fromRGB(100, 130, 100)
    objPlus.Font = Enum.Font.GothamBold
    objPlus.TextSize = 18
    objPlus.Parent = rightPanel
    local objPlusCorner = Instance.new("UICorner")
    objPlusCorner.CornerRadius = UDim.new(0, 6)
    objPlusCorner.Parent = objPlus

    -- Botones SELECCIONAR y DETENER
    local selectBtn = Instance.new("TextButton")
    selectBtn.Size = UDim2.new(0.4, 0, 0, 32)
    selectBtn.Position = UDim2.new(0.05, 0, 0, 300)
    selectBtn.Text = "SELECCIONAR"
    selectBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    selectBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
    selectBtn.BorderSizePixel = 2
    selectBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    selectBtn.Font = Enum.Font.GothamBold
    selectBtn.TextSize = 11
    selectBtn.Parent = rightPanel
    local selectCorner = Instance.new("UICorner")
    selectCorner.CornerRadius = UDim.new(0, 6)
    selectCorner.Parent = selectBtn

    local stopBtn = Instance.new("TextButton")
    stopBtn.Size = UDim2.new(0.4, 0, 0, 32)
    stopBtn.Position = UDim2.new(0.55, 0, 0, 300)
    stopBtn.Text = "DETENER"
    stopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    stopBtn.BackgroundColor3 = Color3.fromRGB(200, 130, 110)
    stopBtn.BorderSizePixel = 2
    stopBtn.BorderColor3 = Color3.fromRGB(150, 80, 60)
    stopBtn.Font = Enum.Font.GothamBold
    stopBtn.TextSize = 11
    stopBtn.Parent = rightPanel
    local stopCorner = Instance.new("UICorner")
    stopCorner.CornerRadius = UDim.new(0, 6)
    stopCorner.Parent = stopBtn

    -- SEGUIR CÁMARA
    local followBtn = Instance.new("TextButton")
    followBtn.Size = UDim2.new(0.6, 0, 0, 32)
    followBtn.Position = UDim2.new(0.2, 0, 0, 340)
    followBtn.Text = _G.followCamera and "✓ SEGUIR: ON" or "◯ SEGUIR: OFF"
    followBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    followBtn.BackgroundColor3 = _G.followCamera and Color3.fromRGB(150, 180, 150) or Color3.fromRGB(210, 160, 130)
    followBtn.BorderSizePixel = 2
    followBtn.BorderColor3 = Color3.fromRGB(150, 100, 70)
    followBtn.Font = Enum.Font.GothamBold
    followBtn.TextSize = 11
    followBtn.Parent = rightPanel
    local followCorner = Instance.new("UICorner")
    followCorner.CornerRadius = UDim.new(0, 6)
    followCorner.Parent = followBtn

    -- SLIDER DE DISTANCIA
    local distLabel = Instance.new("TextLabel")
    distLabel.Size = UDim2.new(1, 0, 0, 22)
    distLabel.Position = UDim2.new(0, 0, 0, 375)
    distLabel.Text = "🎯 " .. _G.followDistance .. "m"
    distLabel.TextColor3 = Color3.fromRGB(200, 120, 80)
    distLabel.BackgroundTransparency = 1
    distLabel.Font = Enum.Font.GothamBold
    distLabel.TextSize = 13
    distLabel.Parent = rightPanel

    local sliderBg = Instance.new("Frame")
    sliderBg.Size = UDim2.new(0.8, 0, 0, 12)
    sliderBg.Position = UDim2.new(0.1, 0, 0, 400)
    sliderBg.BackgroundColor3 = Color3.fromRGB(200, 150, 120)
    sliderBg.BorderSizePixel = 2
    sliderBg.BorderColor3 = Color3.fromRGB(150, 100, 70)
    sliderBg.Parent = rightPanel
    local sliderBgCorner = Instance.new("UICorner")
    sliderBgCorner.CornerRadius = UDim.new(0, 6)
    sliderBgCorner.Parent = sliderBg

    local sliderCircle = Instance.new("TextButton")
    sliderCircle.Size = UDim2.new(0, 16, 0, 16)
    sliderCircle.Position = UDim2.new((_G.followDistance - 5) / 145, -8, 0.5, -8)
    sliderCircle.Text = ""
    sliderCircle.BackgroundColor3 = Color3.fromRGB(255, 215, 190)
    sliderCircle.BorderSizePixel = 2
    sliderCircle.BorderColor3 = Color3.fromRGB(150, 100, 70)
    sliderCircle.Parent = sliderBg
    local sliderCircleCorner = Instance.new("UICorner")
    sliderCircleCorner.CornerRadius = UDim.new(0, 8)
    sliderCircleCorner.Parent = sliderCircle

    -- --- CONEXIONES DE BOTONES ---
    
    -- Velocidad
    speedMinus.MouseButton1Click:Connect(function()
        walkSpeed = math.max(10, walkSpeed - 1)
        speedLabel.Text = "VEL: "..walkSpeed
        local hum = getHumanoid()
        if hum then hum.WalkSpeed = walkSpeed end
    end)
    
    speedPlus.MouseButton1Click:Connect(function()
        walkSpeed = math.min(100, walkSpeed + 1)
        speedLabel.Text = "VEL: "..walkSpeed
        local hum = getHumanoid()
        if hum then hum.WalkSpeed = walkSpeed end
    end)
    
    -- Salto
    jumpMinus.MouseButton1Click:Connect(function()
        jumpPower = math.max(20, jumpPower - 5)
        jumpLabel.Text = "SALTO: "..jumpPower
        local hum = getHumanoid()
        if hum then hum.JumpPower = jumpPower end
    end)
    
    jumpPlus.MouseButton1Click:Connect(function()
        jumpPower = math.min(250, jumpPower + 5)
        jumpLabel.Text = "SALTO: "..jumpPower
        local hum = getHumanoid()
        if hum then hum.JumpPower = jumpPower end
    end)
    
    -- Vuelo
    flyBtn.MouseButton1Click:Connect(function()
        toggleFly(flyBtn)
    end)
    
    flyMinus.MouseButton1Click:Connect(function()
        _G.flySpeed = math.max(0.1, _G.flySpeed - 0.5)
        flySpeedLabel.Text = string.format("%.1f", _G.flySpeed)
    end)
    
    flyPlus.MouseButton1Click:Connect(function()
        _G.flySpeed = math.min(10, _G.flySpeed + 0.5)
        flySpeedLabel.Text = string.format("%.1f", _G.flySpeed)
    end)
    
    -- Fling
    flingBtn.MouseButton1Click:Connect(function()
        toggleFling(flingBtn, flingSpeedBox)
    end)
    
    flingMinus.MouseButton1Click:Connect(function()
        _G.flingSpeed = math.max(1000, _G.flingSpeed - 1000)
        flingSpeedBox.Text = tostring(_G.flingSpeed)
    end)
    
    flingPlus.MouseButton1Click:Connect(function()
        _G.flingSpeed = math.min(50000, _G.flingSpeed + 1000)
        flingSpeedBox.Text = tostring(_G.flingSpeed)
    end)
    
    -- AntiFling
    antiBtn.MouseButton1Click:Connect(function()
        toggleAntiFling(antiBtn)
    end)
    
    -- Fly Car
    flyCarBtn.MouseButton1Click:Connect(function()
        toggleFlyCar(flyCarBtn)
    end)
    
    flyCarMinus.MouseButton1Click:Connect(function()
        _G.flyCarSpeed = math.max(10, _G.flyCarSpeed - 5)
        flyCarSpeedLabel.Text = tostring(_G.flyCarSpeed)
    end)
    
    flyCarPlus.MouseButton1Click:Connect(function()
        _G.flyCarSpeed = math.min(200, _G.flyCarSpeed + 5)
        flyCarSpeedLabel.Text = tostring(_G.flyCarSpeed)
    end)
    
    flyCarUp.MouseButton1Down:Connect(flyCarForward)
    flyCarUp.MouseButton1Up:Connect(flyCarStop)
    flyCarDown.MouseButton1Down:Connect(flyCarBackward)
    flyCarDown.MouseButton1Up:Connect(flyCarStop)
    
    -- TROLL
    bangBtn.MouseButton1Click:Connect(function()
        toggleBang(bangBtn)
    end)
    
    faceBtn.MouseButton1Click:Connect(function()
        toggleFaceBang(faceBtn)
    end)
    
    sitBtn.MouseButton1Click:Connect(function()
        toggleSit(sitBtn)
    end)
    
    suckBtn.MouseButton1Click:Connect(function()
        toggleSuck(suckBtn)
    end)
    
    jerkBtn.MouseButton1Click:Connect(function()
        toggleJerk(jerkBtn)
    end)
    
    -- EXTRAS
    ragdollBtn.MouseButton1Click:Connect(function()
        toggleRagdoll(ragdollBtn)
    end)
    
    infJumpBtn.MouseButton1Click:Connect(function()
        toggleInfiniteJump(infJumpBtn)
    end)
    
    noclipBtn.MouseButton1Click:Connect(function()
        toggleNoclip(noclipBtn)
    end)
    
    espBtn.MouseButton1Click:Connect(function()
        toggleESP(espBtn)
    end)
    
    resetBtn.MouseButton1Click:Connect(function()
        resetStats(resetBtn)
    end)
    
    -- Teleport
    tpBtn.MouseButton1Click:Connect(function()
        local target = nil
        for _, player in ipairs(Players:GetPlayers()) do
            if player.Name:lower():find(usernameBox.Text:lower()) and player ~= LocalPlayer then
                target = player
                break
            end
        end
        
        if target and target.Character then
            local root = target.Character:FindFirstChild("HumanoidRootPart")
            local myRoot = getRoot()
            if root and myRoot then
                myRoot.CFrame = root.CFrame + Vector3.new(0, 3, 0)
            end
        end
    end)
    
    -- Click TP
    local clickTool = nil
    clickTPBtn.MouseButton1Click:Connect(function()
        if clickTool then
            clickTool:Destroy()
            clickTool = nil
            clickTPBtn.Text = "📍 CLICK TP"
        else
            clickTool = Instance.new("Tool")
            clickTool.Name = "Click TP"
            clickTool.RequiresHandle = false
            clickTool.Parent = LocalPlayer.Backpack
            
            clickTool.Activated:Connect(function()
                local mouse = LocalPlayer:GetMouse()
                local myRoot = getRoot()
                if myRoot and mouse.Hit then
                    myRoot.CFrame = CFrame.new(mouse.Hit.p + Vector3.new(0, 3, 0))
                end
            end)
            clickTPBtn.Text = "✅ CLICK TP (ACTIVO)"
        end
    end)
    
    -- SEGUIR CÁMARA
    followBtn.MouseButton1Click:Connect(function()
        _G.followCamera = not _G.followCamera
        followBtn.Text = _G.followCamera and "✓ SEGUIR: ON" or "◯ SEGUIR: OFF"
        followBtn.BackgroundColor3 = _G.followCamera and Color3.fromRGB(150, 180, 150) or Color3.fromRGB(210, 160, 130)
        
        if _G.selectedPart and _G.selectedPart.Parent then
            if _G.followCamera then
                _G.initialRotation = _G.selectedPart.CFrame
                ClearForces(_G.selectedPart)
            else
                ProcessPart(_G.selectedPart)
            end
        end
    end)
    
    -- SLIDER
    local draggingSlider = false
    sliderCircle.MouseButton1Down:Connect(function()
        draggingSlider = true
    end)

    UserInputService.InputChanged:Connect(function(input)
        if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local absX = input.Position.X
            local bgAbsPos, bgSize = sliderBg.AbsolutePosition, sliderBg.AbsoluteSize
            local relX = math.clamp((absX - bgAbsPos.X) / bgSize.X, 0, 1)
            local newDist = math.floor(5 + relX * 145)
            updateSliderUI(newDist, sliderBg, sliderCircle, distLabel)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingSlider = false
        end
    end)
    
    -- MOVE PARTS Dirección
    upBtn.MouseButton1Click:Connect(function()
        if _G.selectedPart and _G.selectedPart.Parent then
            _G.moveDirection = getCameraRelativeDirection(Vector3.new(0, 1, 0))
            ProcessPart(_G.selectedPart)
        end
    end)

    downBtn.MouseButton1Click:Connect(function()
        if _G.selectedPart and _G.selectedPart.Parent then
            _G.moveDirection = getCameraRelativeDirection(Vector3.new(0, -1, 0))
            ProcessPart(_G.selectedPart)
        end
    end)

    leftBtn.MouseButton1Click:Connect(function()
        if _G.selectedPart and _G.selectedPart.Parent then
            _G.moveDirection = getCameraRelativeDirection(Vector3.new(-1, 0, 0))
            ProcessPart(_G.selectedPart)
        end
    end)

    rightBtn.MouseButton1Click:Connect(function()
        if _G.selectedPart and _G.selectedPart.Parent then
            _G.moveDirection = getCameraRelativeDirection(Vector3.new(1, 0, 0))
            ProcessPart(_G.selectedPart)
        end
    end)

    fwdBtn.MouseButton1Click:Connect(function()
        if _G.selectedPart and _G.selectedPart.Parent then
            _G.moveDirection = getCameraRelativeDirection(Vector3.new(0, 0, 1))
            ProcessPart(_G.selectedPart)
        end
    end)

    bwdBtn.MouseButton1Click:Connect(function()
        if _G.selectedPart and _G.selectedPart.Parent then
            _G.moveDirection = getCameraRelativeDirection(Vector3.new(0, 0, -1))
            ProcessPart(_G.selectedPart)
        end
    end)

    objMinus.MouseButton1Click:Connect(function()
        _G.floatSpeed = math.max(5, _G.floatSpeed - 5)
        objSpeedLabel.Text = "⚡ VEL. OBJETO: ".._G.floatSpeed
        if _G.selectedPart and _G.selectedPart.Parent then
            ProcessPart(_G.selectedPart)
        end
    end)

    objPlus.MouseButton1Click:Connect(function()
        _G.floatSpeed = _G.floatSpeed + 5
        objSpeedLabel.Text = "⚡ VEL. OBJETO: ".._G.floatSpeed
        if _G.selectedPart and _G.selectedPart.Parent then
            ProcessPart(_G.selectedPart)
        end
    end)

    -- Selección
    local isSelecting = false
    selectBtn.MouseButton1Click:Connect(function()
        isSelecting = not isSelecting
        selectBtn.BackgroundColor3 = isSelecting and Color3.fromRGB(150, 180, 150) or Color3.fromRGB(210, 160, 130)
        selectBtn.Text = isSelecting and "ACTIVO" or "SELECCIONAR"
    end)

    stopBtn.MouseButton1Click:Connect(function()
        _G.moveDirection = Vector3.new(0, 0, 0)
        if _G.selectedPart and _G.selectedPart.Parent then
            ProcessPart(_G.selectedPart)
        end
    end)

    UserInputService.InputBegan:Connect(function(input)
        if isSelecting and (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) then
            local ray = Camera:ViewportPointToRay(input.Position.X, input.Position.Y)
            local part = Workspace:FindPartOnRay(Ray.new(ray.Origin, ray.Direction * 1000), LocalPlayer.Character)
            if part and not part.Anchored then
                if _G.selectedHighlight then _G.selectedHighlight:Destroy() end
                
                _G.selectedPart = part
                _G.initialRotation = part.CFrame
                
                local h = Instance.new("SelectionBox", part)
                h.Adornee = part
                h.Color3 = Color3.fromRGB(200, 120, 80)
                _G.selectedHighlight = h
                
                ProcessPart(part)
                
                isSelecting = false
                selectBtn.BackgroundColor3 = Color3.fromRGB(210, 160, 130)
                selectBtn.Text = "SELECCIONAR"
            end
        end
    end)
    
    -- Expandir/contraer paneles
    local topExpanded = false
    topPanelBtn.MouseButton1Click:Connect(function()
        topExpanded = not topExpanded
        topPanelBtn.Text = topExpanded and "▼ EXTRAS ▲" or "▲ EXTRAS ▼"
        
        if topExpanded then
            topPanel.Visible = true
            topPanel.Size = UDim2.new(0, 280, 0, 0)
            TweenService:Create(topPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0, 280, 0, topPanelHeight)
            }):Play()
        else
            TweenService:Create(topPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0, 280, 0, 0)
            }):Play()
            task.wait(0.3)
            topPanel.Visible = false
        end
    end)
    
    local leftExpanded = false
    leftPanelBtn.MouseButton1Click:Connect(function()
        leftExpanded = not leftExpanded
        leftPanelBtn.Text = leftExpanded and "◀" or "▶"
        
        if leftExpanded then
            leftPanel.Visible = true
            leftPanel.Size = UDim2.new(0, 0, 0, 440)
            TweenService:Create(leftPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0, 250, 0, 440)
            }):Play()
        else
            TweenService:Create(leftPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0, 0, 0, 440)
            }):Play()
            task.wait(0.3)
            leftPanel.Visible = false
        end
    end)
    
    local rightExpanded = false
    rightPanelBtn.MouseButton1Click:Connect(function()
        rightExpanded = not rightExpanded
        rightPanelBtn.Text = rightExpanded and "▶" or "◀"
        
        if rightExpanded then
            rightPanel.Visible = true
            rightPanel.Size = UDim2.new(0, 0, 0, 520)
            TweenService:Create(rightPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0, 300, 0, 520)
            }):Play()
        else
            TweenService:Create(rightPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0, 0, 0, 520)
            }):Play()
            task.wait(0.3)
            rightPanel.Visible = false
        end
    end)
    
    local trollExpanded = false
    trollBtn.MouseButton1Click:Connect(function()
        trollExpanded = not trollExpanded
        trollBtn.Text = trollExpanded and "👹 TROLL ▲" or "👹 TROLL ▼"
        
        if trollExpanded then
            trollPanel.Visible = true
            trollPanel.Size = UDim2.new(0.9, 0, 0, 0)
            TweenService:Create(trollPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0.9, 0, 0, 95)
            }):Play()
        else
            TweenService:Create(trollPanel, TweenInfo.new(0.3), {
                Size = UDim2.new(0.9, 0, 0, 0)
            }):Play()
            task.wait(0.3)
            trollPanel.Visible = false
        end
    end)
    
    -- Escala
    scaleDownBtn.MouseButton1Click:Connect(function()
        uiScale.Scale = math.max(0.5, uiScale.Scale - 0.1)
    end)
    
    scaleUpBtn.MouseButton1Click:Connect(function()
        uiScale.Scale = math.min(1.5, uiScale.Scale + 0.1)
    end)
    
    -- Cerrar
    closeBtn.MouseButton1Click:Connect(function()
        if _G.hiddenfling then _G.hiddenfling = false end
        if _G.isFlying then _G.isFlying = false end
        main.Visible = false
        moonBtn.Visible = true
        moonBtn.Size = UDim2.new(0, 0, 0, 0)
        TweenService:Create(moonBtn, TweenInfo.new(0.3), {
            Size = UDim2.new(0, 55, 0, 55)
        }):Play()
    end)
    
    moonBtn.MouseButton1Click:Connect(function()
        main.Visible = true
        moonBtn.Visible = false
    end)
end

-- Ejecutar
CreateHub()
