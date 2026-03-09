-- === RIKESAHUB (OPTIMIZADO) ===
local Players, UIS, RS, TS, SGui, WS, Repl = 
    game:GetService("Players"), game:GetService("UserInputService"), game:GetService("RunService"), 
    game:GetService("TweenService"), game:GetService("StarterGui"), game:GetService("Workspace"), 
    game:GetService("ReplicatedStorage")

local LP, Camera = Players.LocalPlayer, WS.CurrentCamera

SGui:SetCore("SendNotification", {Title = "RIKESA HUB", Text = "[ RIKESA HUB ] Loaded!", Duration = 4})

-- GLOBALS
_G.selectedPart = nil; _G.selectedHighlight = nil
_G.floatSpeed = 10; _G.flySpeed = 1.5; _G.isFlying = false
_G.hiddenfling = false; _G.antiFling = false; _G.followCamera = false
_G.followDistance = 15; _G.moveDirection = Vector3.new(0,0,0)
_G.initialRotation = CFrame.new(); _G.flingSpeed = 10000
_G.multiSelect = {}  -- Tabla para partes seleccionadas
_G.selectRadius = 50 -- Radio del círculo (default 50)
_G.selectCircle = nil -- Referencia al círculo visual
_G.ringPartsEnabled = false
_G.tornadoConfig = {
    radius = 50,
    height = 100,
    rotationSpeed = 10,
    attractionStrength = 1000,
}
_G.tornadoParts = {}
_G.godModeActive = false
_G.godModeConnection = nil
_G.fallDamageConnection = nil
_G.originalWalkSpeed = 16
_G.originalJumpPower = 50
_G.flyBG = nil
_G.flyBV = nil
_G.flyConnection = nil

-- Troll
_G.bangActive = false; _G.faceBangActive = false; _G.sitActive = false
_G.suckActive = false; _G.ragdollActive = false; _G.infiniteJumpActive = false
_G.noclipActive = false; _G.espEnabled = false; _G.jerkActive = false
_G.zeroGActive = false  
_G.zeroGConnection = nil  

-- Conexiones
_G.bangConnection, _G.faceBangConnection, _G.sitConnection, _G.suckConnection = nil,nil,nil,nil
_G.ragdollConnection, _G.infiniteJumpConnection, _G.noclipConnection = nil,nil,nil
_G.flingConnection, _G.jerkTrack, _G.closerhandsTrack, _G.jerkTool = nil,nil,nil,nil

-- Fly Car
_G.flyCarActive = false; _G.flyCarSpeed = 50; _G.flyCarBV = nil; _G.flyCarBG = nil

-- Protección contra crashes
pcall(function()
    getgenv().__crashProtection = true
end)

-- Función segura para ejecutar código
local function safeCall(func, ...)
    local success, result = pcall(func, ...)
    if not success then
        warn("[RIKESA HUB] Error en ejecución:", result)
    end
    return result
end

-- Helpers
local function gR() local c = LP.Character return c and c:FindFirstChild("HumanoidRootPart") end
local function gH() local c = LP.Character return c and c:FindFirstChildOfClass("Humanoid") end
local function getCamDir(d)
    if d.Y ~= 0 then return d end
    local l, r = Camera.CFrame.LookVector, Camera.CFrame.RightVector
    local lf, rf = Vector3.new(l.X,0,l.Z).Unit, Vector3.new(r.X,0,r.Z).Unit
    if d.Z > 0 then return lf elseif d.Z < 0 then return -lf elseif d.X > 0 then return rf elseif d.X < 0 then return -rf end
    return d
end

-- Move Parts
local function clearForces(p) if not p then return end
    for _, c in p:GetChildren() do if c:IsA("BodyVelocity") or c:IsA("BodyPosition") or c:IsA("BodyGyro") then c:Destroy() end end
end

local function processPart(p)
    if not p then return end; clearForces(p)
    local bv = Instance.new("BodyVelocity", p); bv.Name = "CV"; bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bv.Velocity = _G.moveDirection * _G.floatSpeed; bv.P = 1250
    local bg = Instance.new("BodyGyro", p); bg.Name = "CG"; bg.P = 9e4; bg.MaxTorque = Vector3.new(9e9,9e9,9e9); bg.CFrame = p.CFrame
end

local function updateVel(p) 
    local bv = p and p:FindFirstChild("CV")
    if bv then bv.Velocity = _G.moveDirection * _G.floatSpeed end
end

-- Funciones para selección múltiple
local function createSelectionCircle(radius)
    if _G.selectCircle then _G.selectCircle:Destroy() end
    
    local circle = Instance.new("Part")
    circle.Name = "SelectionCircle"
    circle.Anchored = true
    circle.CanCollide = false
    circle.Transparency = 0.7
    circle.BrickColor = BrickColor.new("Bright yellow")
    circle.Material = Enum.Material.Neon
    circle.Size = Vector3.new(radius*2, 0.2, radius*2)
    circle.Shape = Enum.PartType.Cylinder
    circle.Parent = workspace
    
    local mesh = Instance.new("CylinderMesh", circle)
    mesh.Scale = Vector3.new(1, 0.1, 1)
    
    local hrp = gR()
    if hrp then
        circle.Position = hrp.Position - Vector3.new(0, 1, 0)
    end
    
    _G.selectCircle = circle
    return circle
end

local function updateCirclePosition()
    if _G.selectCircle and _G.selectCircle.Parent then
        local hrp = gR()
        if hrp then
            _G.selectCircle.Position = hrp.Position - Vector3.new(0, 1, 0)
        end
    end
end

local function selectAllUnanchored()
    -- Limpiar selección anterior
    for _, part in pairs(_G.multiSelect) do
        if part and part.Parent then
            for _, hl in pairs(part:GetChildren()) do
                if hl.Name == "MultiHighlight" then hl:Destroy() end
            end
        end
    end
    _G.multiSelect = {}
    
    -- Obtener posición del jugador
    local hrp = gR()
    if not hrp then return end
    
    local center = hrp.Position
    
    -- Buscar partes desancladas en el radio
    for _, part in pairs(workspace:GetDescendants()) do
        if part:IsA("BasePart") and not part.Anchored and part ~= hrp then
            local dist = (part.Position - center).Magnitude
            if dist <= _G.selectRadius then
                table.insert(_G.multiSelect, part)
                
                -- Crear highlight para esta parte
                local hl = Instance.new("SelectionBox", part)
                hl.Name = "MultiHighlight"
                hl.Adornee = part
                hl.Color3 = Color3.fromRGB(100, 255, 100)
                hl.LineThickness = 0.1
            end
        end
    end
    
    -- Si hay una parte seleccionada individual, mantenerla
    if _G.selectedPart and _G.selectedPart.Parent then
        -- Verificar si ya está en la tabla
        local found = false
        for _, part in pairs(_G.multiSelect) do
            if part == _G.selectedPart then found = true; break end
        end
        if not found then
            table.insert(_G.multiSelect, _G.selectedPart)
        end
    end
end

local function processMultiPart()
    for _, part in pairs(_G.multiSelect) do
        if part and part.Parent then
            clearForces(part)
            local bv = Instance.new("BodyVelocity", part)
            bv.Name = "CV"
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bv.Velocity = _G.moveDirection * _G.floatSpeed
            bv.P = 1250
            
            local bg = Instance.new("BodyGyro", part)
            bg.Name = "CG"
            bg.P = 9e4
            bg.MaxTorque = Vector3.new(9e9,9e9,9e9)
            bg.CFrame = part.CFrame
        end
    end
end

local function updateMultiVelocity()
    for _, part in pairs(_G.multiSelect) do
        if part and part.Parent then
            local bv = part:FindFirstChild("CV")
            if bv then
                bv.Velocity = _G.moveDirection * _G.floatSpeed
            end
        end
    end
end

local function clearMultiForces()
    for _, part in pairs(_G.multiSelect) do
        if part and part.Parent then
            clearForces(part)
        end
    end
end

-- Funciones para TORNADO
local function RetainTornadoPart(part)
    if part:IsA("BasePart") and not part.Anchored and part:IsDescendantOf(workspace) then
        if part.Parent == LP.Character or part:IsDescendantOf(LP.Character) then
            return false
        end
        return true
    end
    return false
end

local function addTornadoPart(part)
    if RetainTornadoPart(part) then
        -- DOBLE VERIFICACIÓN: asegurar que no sea del personaje
        if part.Parent ~= LP.Character and not part:IsDescendantOf(LP.Character) then
            if not table.find(_G.tornadoParts, part) then
                table.insert(_G.tornadoParts, part)
            end
        end
    end
end

local function removeTornadoPart(part)
    local index = table.find(_G.tornadoParts, part)
    if index then
        table.remove(_G.tornadoParts, index)
    end
end

--- Función para restaurar propiedades de partes al desactivar tornado
local function restoreTornadoParts()
    for _, part in pairs(_G.tornadoParts) do
        if part and part.Parent then
            -- Restaurar CanCollide
            part.CanCollide = true
            -- DETENER EL MOVIMIENTO
            part.Velocity = Vector3.new(0,0,0)
            part.RotVelocity = Vector3.new(0,0,0)
        end
    end
end

-- Inicializar partes para tornado
for _, part in pairs(workspace:GetDescendants()) do
    addTornadoPart(part)
end

workspace.DescendantAdded:Connect(addTornadoPart)
workspace.DescendantRemoving:Connect(removeTornadoPart)

-- LIMPIAR CUALQUIER PARTE DEL PERSONAJE QUE SE HAYA COLADO
if LP.Character then
    local partsToRemove = {}
    for i, part in ipairs(_G.tornadoParts) do
        if part.Parent == LP.Character or part:IsDescendantOf(LP.Character) then
            table.insert(partsToRemove, i)
        end
    end
    -- Eliminar de atrás hacia adelante para no alterar los índices
    for i = #partsToRemove, 1, -1 do
        table.remove(_G.tornadoParts, partsToRemove[i])
    end
end

-- Función para activar/desactivar tornado
function toggleTornado(btn)
    _G.ringPartsEnabled = not _G.ringPartsEnabled
    if btn then
        btn.Text = _G.ringPartsEnabled and "🌪️ TORNADO: ON" or "🌪️ TORNADO: OFF"
        btn.BackgroundColor3 = _G.ringPartsEnabled and Color3.fromRGB(100,200,100) or Color3.fromRGB(210,160,130)
    end
    
    -- Si se desactiva el tornado, restaurar propiedades
    if not _G.ringPartsEnabled then
        restoreTornadoParts()
    end
end

-- TROLL FUNCTIONS
function toggleBang(btn)
    if _G.bangActive then
        _G.bangActive = false; if _G.bangConnection then _G.bangConnection:Disconnect(); _G.bangConnection = nil end
        local h = gR(); if h then for _, c in h:GetChildren() do if c:IsA("BodyPosition") or c:IsA("BodyGyro") then c:Destroy() end end end
        local hum = gH(); 
        if hum then 
            hum.Sit = false; hum.PlatformStand = false; hum.AutoRotate = true; 
            pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
            -- Restaurar velocidad original SOLO si existe
            if _G.originalWalkSpeed then
                pcall(function() hum.WalkSpeed = _G.originalWalkSpeed end)
            end
            -- Restaurar animación de caminar original SOLO si existe
            pcall(function()
                local animate = LP.Character and LP.Character:FindFirstChild("Animate")
                if animate and animate.walk and _G.originalWalkAnim then
                    animate.walk.WalkAnim.AnimationId = _G.originalWalkAnim
                end
            end)
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200,130,110) end; return
    end
    
    _G.bangActive = true; if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    
    -- USAR PCALL PARA TODO LO QUE PUEDE FALLAR
    pcall(function()
        -- Guardar velocidad original si no está guardada
        local hum = gH()
        if hum and not _G.originalWalkSpeed then
            _G.originalWalkSpeed = hum.WalkSpeed
        end
        
        -- Aumentar velocidad 20% (solo si hum existe)
        if hum then
            hum.WalkSpeed = (_G.originalWalkSpeed or 16) * 1.2
        end
        
        -- Guardar animación original si no está guardada
        local animate = LP.Character and LP.Character:FindFirstChild("Animate")
        if animate and animate.walk and not _G.originalWalkAnim then
            _G.originalWalkAnim = animate.walk.WalkAnim.AnimationId
        end
        
        -- Cambiar a animación ZOMBIE (solo si animate existe)
        if animate and animate.walk then
            animate.walk.WalkAnim.AnimationId = "http://www.roblox.com/asset/?id=616168032"
        end
    end)
    
    local target, myRoot, nearest = nil, gR(), math.huge
    if myRoot then 
        for _, p in Players:GetPlayers() do 
            if p ~= LP and p.Character then 
                local r = p.Character:FindFirstChild("HumanoidRootPart")
                if r then 
                    local d = (myRoot.Position - r.Position).Magnitude
                    if d < nearest then 
                        nearest = d; target = p 
                    end 
                end 
            end 
        end 
    end
    if not target then 
        _G.bangActive = false; 
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200,130,110) end 
        return 
    end
    
    local last, dir = tick(), 1
    _G.bangConnection = RS.Heartbeat:Connect(function()
        if not _G.bangActive or not target.Character then 
            _G.bangActive = false; 
            if _G.bangConnection then 
                _G.bangConnection:Disconnect()
                _G.bangConnection = nil 
            end 
            return 
        end
        local tr = target.Character:FindFirstChild("HumanoidRootPart")
        local hrp = gR()
        if not tr or not hrp then return end
        if tick() - last > 0.3 then 
            dir = dir * -1
            last = tick() 
        end
        local behind = tr.Position - tr.CFrame.LookVector * 1.2
        behind = Vector3.new(behind.X, tr.Position.Y, behind.Z)
        local bp = hrp:FindFirstChild("BangBP") or Instance.new("BodyPosition", hrp)
        bp.Name = "BangBP"
        bp.MaxForce = Vector3.new(math.huge,math.huge,math.huge)
        bp.P = 10000
        bp.Position = behind + tr.CFrame.LookVector * dir
        bp.Parent = hrp
        
        local bg = hrp:FindFirstChild("BangBG") or Instance.new("BodyGyro", hrp)
        bg.Name = "BangBG"
        bg.MaxTorque = Vector3.new(9e9,9e9,9e9)
        bg.CFrame = CFrame.new(hrp.Position, tr.Position)
        bg.Parent = hrp
    end)
end

function toggleFaceBang(btn)
    if _G.faceBangActive then
        _G.faceBangActive = false; if _G.faceBangConnection then _G.faceBangConnection:Disconnect(); _G.faceBangConnection = nil end
        local h = gR(); if h then for _, c in h:GetChildren() do if c:IsA("BodyPosition") or c:IsA("BodyGyro") then c:Destroy() end end end
        local hum = gH(); if hum then hum.Sit = false; hum.PlatformStand = false; hum.AutoRotate = true; hum:ChangeState(Enum.HumanoidStateType.Running) end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150,170,200) end; return
    end
    _G.faceBangActive = true; if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    local target, myRoot, nearest = nil, gR(), math.huge
    if myRoot then for _, p in Players:GetPlayers() do if p ~= LP and p.Character then local r = p.Character:FindFirstChild("HumanoidRootPart")
        if r then local d = (myRoot.Position - r.Position).Magnitude; if d < nearest then nearest = d; target = p end end end end end
    if not target then _G.faceBangActive = false; if btn then btn.BackgroundColor3 = Color3.fromRGB(150,170,200) end return end
    local hum = gH(); if hum then hum.Sit = true end
    local last, dir = tick(), 1
    _G.faceBangConnection = RS.Heartbeat:Connect(function()
        if not _G.faceBangActive or not target.Character then _G.faceBangActive = false; if _G.faceBangConnection then _G.faceBangConnection:Disconnect(); _G.faceBangConnection = nil end return end
        local th = target.Character:FindFirstChild("Head"); local hrp, head = gR(), LP.Character and LP.Character:FindFirstChild("Head")
        if not th or not hrp or not head then return end
        if tick() - last > 0.2 then dir = dir * -1; last = tick() end
        local base = th.Position + th.CFrame.LookVector; local targetPos = Vector3.new(base.X, th.Position.Y - head.Size.Y/2 + 0.5, base.Z) + th.CFrame.LookVector * dir * 0.8
        local bp = hrp:FindFirstChild("FaceBangBP") or Instance.new("BodyPosition", hrp); bp.Name = "FaceBangBP"; bp.MaxForce = Vector3.new(math.huge,math.huge,math.huge); bp.P = 10000; bp.Position = targetPos
        local bg = hrp:FindFirstChild("FaceBangBG") or Instance.new("BodyGyro", hrp); bg.Name = "FaceBangBG"; bg.MaxTorque = Vector3.new(9e9,9e9,9e9); bg.CFrame = CFrame.new(hrp.Position, th.Position)
    end)
end

function toggleSit(btn)
    if _G.sitActive then
        _G.sitActive = false; if _G.sitConnection then _G.sitConnection:Disconnect(); _G.sitConnection = nil end
        local h = gR(); if h then for _, c in h:GetChildren() do if c:IsA("BodyPosition") or c:IsA("BodyGyro") then c:Destroy() end end end
        local hum = gH(); if hum then hum.Sit = false; hum.PlatformStand = false; hum.AutoRotate = true; hum:ChangeState(Enum.HumanoidStateType.Running) end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150,180,150) end; return
    end
    _G.sitActive = true; if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    local target, myRoot, nearest = nil, gR(), math.huge
    if myRoot then for _, p in Players:GetPlayers() do if p ~= LP and p.Character then local r = p.Character:FindFirstChild("HumanoidRootPart")
        if r then local d = (myRoot.Position - r.Position).Magnitude; if d < nearest then nearest = d; target = p end end end end end
    if not target then _G.sitActive = false; if btn then btn.BackgroundColor3 = Color3.fromRGB(150,180,150) end return end
    local hum = gH(); if hum then hum.Sit = true end
    _G.sitConnection = RS.Heartbeat:Connect(function()
        if not _G.sitActive or not target.Character then _G.sitActive = false; if _G.sitConnection then _G.sitConnection:Disconnect(); _G.sitConnection = nil end return end
        local th = target.Character:FindFirstChild("Head"); local hrp = gR()
        if not th or not hrp then return end
        local bp = hrp:FindFirstChild("SitBP") or Instance.new("BodyPosition", hrp); bp.Name = "SitBP"; bp.MaxForce = Vector3.new(math.huge,math.huge,math.huge); bp.P = 10000; bp.Position = th.Position + Vector3.new(0,0.8,0)
        local bg = hrp:FindFirstChild("SitBG") or Instance.new("BodyGyro", hrp); bg.Name = "SitBG"; bg.MaxTorque = Vector3.new(9e9,9e9,9e9); bg.CFrame = th.CFrame
    end)
end

function toggleSuck(btn)
    if _G.suckActive then
        _G.suckActive = false; if _G.suckConnection then _G.suckConnection:Disconnect(); _G.suckConnection = nil end
        local h = gR(); if h then for _, c in h:GetChildren() do if c:IsA("BodyPosition") or c:IsA("BodyGyro") then c:Destroy() end end end
        local hum = gH(); 
        if hum then 
            hum.Sit = false; hum.PlatformStand = false; hum.AutoRotate = true
            pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
            -- Restaurar animación de caminar original
            pcall(function()
                local animate = LP.Character and LP.Character:FindFirstChild("Animate")
                if animate and animate.walk and _G.originalWalkAnim then
                    animate.walk.WalkAnim.AnimationId = _G.originalWalkAnim
                end
            end)
        end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200,150,180) end
        return
    end
    
    _G.suckActive = true; if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    
    pcall(function()
        -- Guardar animación original si no está guardada
        local animate = LP.Character and LP.Character:FindFirstChild("Animate")
        if animate and animate.walk and not _G.originalWalkAnim then
            _G.originalWalkAnim = animate.walk.WalkAnim.AnimationId
        end
        
        -- Cambiar a animación GOJO
        if animate and animate.walk then
            animate.walk.WalkAnim.AnimationId = "http://www.roblox.com/asset/?id=95643163365384"
        end
        
        -- BAJAR UN POCO MÁS EL PERSONAJE (solo si existe)
        local hrp = gR()
        if hrp then
            hrp.CFrame = hrp.CFrame + Vector3.new(0, -0.5, 0)
        end
    end)
    
    local target, myRoot, nearest = nil, gR(), math.huge
    if myRoot then 
        for _, p in Players:GetPlayers() do 
            if p ~= LP and p.Character then 
                local r = p.Character:FindFirstChild("HumanoidRootPart")
                if r then 
                    local d = (myRoot.Position - r.Position).Magnitude
                    if d < nearest then 
                        nearest = d; target = p 
                    end 
                end 
            end 
        end 
    end
    if not target then 
        _G.suckActive = false
        if btn then btn.BackgroundColor3 = Color3.fromRGB(200,150,180) end
        return 
    end
    
    local last, dir = tick(), 1
    _G.suckConnection = RS.Heartbeat:Connect(function()
        if not _G.suckActive or not target.Character then 
            _G.suckActive = false
            if _G.suckConnection then 
                _G.suckConnection:Disconnect()
                _G.suckConnection = nil 
            end 
            return 
        end
        local tr = target.Character:FindFirstChild("HumanoidRootPart") or target.Character:FindFirstChild("Torso")
        local hrp, head = gR(), LP.Character and LP.Character:FindFirstChild("Head")
        if not tr or not hrp or not head then return end
        if tick() - last > 0.25 then 
            dir = dir * -1
            last = tick() 
        end
        local groin = tr.Position + Vector3.new(0,-2,0)
        local front = groin + tr.CFrame.LookVector * 1.2
        local targetPos = Vector3.new(front.X, groin.Y - head.Size.Y/2 - 0.2, front.Z) + tr.CFrame.LookVector * dir
        local bp = hrp:FindFirstChild("SuckBP") or Instance.new("BodyPosition", hrp)
        bp.Name = "SuckBP"
        bp.MaxForce = Vector3.new(math.huge,math.huge,math.huge)
        bp.P = 10000
        bp.Position = targetPos
        bp.Parent = hrp
        
        local bg = hrp:FindFirstChild("SuckBG") or Instance.new("BodyGyro", hrp)
        bg.Name = "SuckBG"
        bg.MaxTorque = Vector3.new(9e9,9e9,9e9)
        bg.CFrame = CFrame.new(hrp.Position, tr.Position)
        bg.Parent = hrp
    end)
end

function toggleRagdoll(btn)
    _G.ragdollActive = not _G.ragdollActive; local hum = gH()
    if not hum then return end
    if _G.ragdollActive then
        hum.PlatformStand = true; hum.AutoRotate = false; hum:ChangeState(Enum.HumanoidStateType.Physics)
        local hrp = gR(); if hrp then hrp.Velocity = Vector3.new(0,-10,0) end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    else
        hum.PlatformStand = false; hum.AutoRotate = true; hum:ChangeState(Enum.HumanoidStateType.Running)
        if btn then btn.BackgroundColor3 = Color3.fromRGB(210,160,130) end
    end
end

function toggleInfiniteJump(btn)
    _G.infiniteJumpActive = not _G.infiniteJumpActive
    if _G.infiniteJumpActive then
        _G.infiniteJumpConnection = UIS.JumpRequest:Connect(function()
            if not _G.infiniteJumpActive then if _G.infiniteJumpConnection then _G.infiniteJumpConnection:Disconnect(); _G.infiniteJumpConnection = nil end return end
            local hum = gH(); if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
        end)
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    else
        if _G.infiniteJumpConnection then _G.infiniteJumpConnection:Disconnect(); _G.infiniteJumpConnection = nil end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150,170,200) end
    end
end

function toggleNoclip(btn)
    _G.noclipActive = not _G.noclipActive
    if _G.noclipActive then
        _G.noclipConnection = RS.Stepped:Connect(function()
            if not _G.noclipActive or not LP.Character then if _G.noclipConnection then _G.noclipConnection:Disconnect(); _G.noclipConnection = nil end return end
            for _, p in LP.Character:GetDescendants() do if p:IsA("BasePart") then p.CanCollide = false end end
        end)
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    else
        if _G.noclipConnection then _G.noclipConnection:Disconnect(); _G.noclipConnection = nil end
        if LP.Character then for _, p in LP.Character:GetDescendants() do if p:IsA("BasePart") then p.CanCollide = true end end end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150,170,200) end
    end
end

function toggleESP(btn)
    _G.espEnabled = not _G.espEnabled
    if _G.espEnabled then
        for _, p in Players:GetPlayers() do if p ~= LP and p.Character then
            local h = Instance.new("Highlight", p.Character); h.Adornee = p.Character; h.FillColor = Color3.fromRGB(255,255,0); h.FillTransparency = 0.5
        end end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    else
        for _, p in Players:GetPlayers() do if p.Character then local o = p.Character:FindFirstChildOfClass("Highlight"); if o then o:Destroy() end end end
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150,170,200) end
    end
end

function resetStats(btn)
    if _G.bangActive then toggleBang() end; if _G.faceBangActive then toggleFaceBang() end; if _G.sitActive then toggleSit() end
    if _G.suckActive then toggleSuck() end; if _G.ragdollActive then toggleRagdoll() end; if _G.infiniteJumpActive then toggleInfiniteJump() end
    if _G.noclipActive then toggleNoclip() end; if _G.espEnabled then toggleESP() end; if _G.jerkActive then toggleJerk() end
    if _G.hiddenfling then _G.hiddenfling = false; if _G.flingConnection then _G.flingConnection:Disconnect(); _G.flingConnection = nil end end
    if btn then btn.BackgroundColor3 = Color3.fromRGB(180,140,100) end
end

-- Función para activar/desactivar God Mode (CORREGIDA)
function toggleGodMode(btn)
    _G.godModeActive = not _G.godModeActive
    local character = LP.Character
    if character then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            if _G.godModeActive then
                -- Activar god mode
                humanoid.MaxHealth = math.huge
                humanoid.Health = math.huge
                -- Prevenir daño por caída
                humanoid.UseJumpPower = true
                
                -- Conexión para mantener salud infinita
                if _G.godModeConnection then
                    _G.godModeConnection:Disconnect()
                end
                _G.godModeConnection = humanoid:GetPropertyChangedSignal("Health"):Connect(function()
                    if _G.godModeActive and humanoid.Health < humanoid.MaxHealth then
                        humanoid.Health = humanoid.MaxHealth
                    end
                end)
                
                -- Conexión específica para daño por caída
                if _G.fallDamageConnection then
                    _G.fallDamageConnection:Disconnect()
                end
                _G.fallDamageConnection = humanoid:GetPropertyChangedSignal("FloorMaterial"):Connect(function()
                    if _G.godModeActive and humanoid.Health < humanoid.MaxHealth then
                        humanoid.Health = humanoid.MaxHealth
                    end
                end)
            else
                -- Desactivar god mode
                if _G.godModeConnection then
                    _G.godModeConnection:Disconnect()
                    _G.godModeConnection = nil
                end
                if _G.fallDamageConnection then
                    _G.fallDamageConnection:Disconnect()
                    _G.fallDamageConnection = nil
                end
                humanoid.MaxHealth = 100
                humanoid.Health = 100
            end
        end
    end
    if btn then
        btn.Text = _G.godModeActive and "🛡️ GOD MODE: ON" or "🛡️ GOD MODE: OFF"
        btn.BackgroundColor3 = _G.godModeActive and Color3.fromRGB(0,170,0) or Color3.fromRGB(200,150,180)
    end
end

-- Manejar cuando el personaje respawnea (para mantener god mode)
LP.CharacterAdded:Connect(function(newChar)
    if _G.godModeActive then
        task.wait(0.5) -- esperar a que se cargue
        local humanoid = newChar:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.MaxHealth = math.huge
            humanoid.Health = math.huge
            humanoid.UseJumpPower = true
            
            if _G.godModeConnection then
                _G.godModeConnection:Disconnect()
            end
            _G.godModeConnection = humanoid:GetPropertyChangedSignal("Health"):Connect(function()
                if _G.godModeActive and humanoid.Health < humanoid.MaxHealth then
                    humanoid.Health = humanoid.MaxHealth
                end
            end)
            
            if _G.fallDamageConnection then
                _G.fallDamageConnection:Disconnect()
            end
            _G.fallDamageConnection = humanoid:GetPropertyChangedSignal("FloorMaterial"):Connect(function()
                if _G.godModeActive and humanoid.Health < humanoid.MaxHealth then
                    humanoid.Health = humanoid.MaxHealth
                end
            end)
        end
    end
end)

function toggleFling(btn, box)
    _G.hiddenfling = not _G.hiddenfling
    if _G.hiddenfling then
        if btn then btn.Text = "FLING: ON"; btn.BackgroundColor3 = Color3.fromRGB(200,100,100) end
        _G.flingConnection = RS.Heartbeat:Connect(function()
            if not _G.hiddenfling then if _G.flingConnection then _G.flingConnection:Disconnect(); _G.flingConnection = nil end return end
            local c = LP.Character; if not c then return end; local hrp = c:FindFirstChild("HumanoidRootPart"); if not hrp then return end
            local v = hrp.Velocity; hrp.Velocity = v * (_G.flingSpeed/5000) + Vector3.new(0,_G.flingSpeed/2,0)
            RS.RenderStepped:Wait(); hrp.Velocity = v; RS.Stepped:Wait(); hrp.Velocity = v + Vector3.new(0,0.1,0)
        end)
    else
        if _G.flingConnection then _G.flingConnection:Disconnect(); _G.flingConnection = nil end
        if btn then btn.Text = "FLING: OFF"; btn.BackgroundColor3 = Color3.fromRGB(255,255,255) end
    end
end

function toggleAntiFling(btn)
    _G.antiFling = not _G.antiFling
    if btn then btn.Text = _G.antiFling and "ANTIFLING: ON" or "ANTIFLING: OFF"; btn.BackgroundColor3 = _G.antiFling and Color3.fromRGB(150,180,150) or Color3.fromRGB(210,160,130) end
end

function toggleJerk(btn)
    local hum = gH(); if not hum then return end
    local c = LP.Character; if not c or not c:FindFirstChild("Torso") then
        SGui:SetCore("SendNotification", {Title = "Error", Text = "Jerk solo R6", Duration = 2}); return
    end
    if _G.jerkActive then
        _G.jerkActive = false; if _G.jerkTrack then _G.jerkTrack:Stop(); _G.jerkTrack = nil end
        if _G.closerhandsTrack then _G.closerhandsTrack:Stop(); _G.closerhandsTrack = nil end
        hum.WalkSpeed = 16; hum.JumpPower = 50
        if _G.jerkTool then _G.jerkTool:Destroy(); _G.jerkTool = nil end
        if btn then btn.Text = "🍆 JERK"; btn.BackgroundColor3 = Color3.fromRGB(180,140,200) end
    else
        _G.jerkActive = true; _G.jerkTool = Instance.new("Tool"); _G.jerkTool.Name = "jerk"; _G.jerkTool.RequiresHandle = false
        local new = _G.jerkTool:Clone(); new.Parent = LP.Backpack
        local jerking = false; local ws, jp = hum.WalkSpeed, hum.JumpPower
        new.Equipped:Connect(function() jerking = true end)
        new.Unequipped:Connect(function() jerking = false; hum.WalkSpeed = ws; hum.JumpPower = jp
            if _G.jerkTrack then _G.jerkTrack:Stop(); _G.jerkTrack = nil end
            if _G.closerhandsTrack then _G.closerhandsTrack:Stop(); _G.closerhandsTrack = nil end
        end)
        local conn = RS.RenderStepped:Connect(function()
            if not hum or not hum.Parent then return end
            if jerking and _G.jerkActive then
                hum.WalkSpeed = 0; hum.JumpPower = 0
                if not _G.jerkTrack then local a = Instance.new("Animation"); a.AnimationId = "rbxassetid://99198989"; _G.jerkTrack = hum:LoadAnimation(a); _G.jerkTrack.Looped = true; _G.jerkTrack:Play() end
                if not _G.closerhandsTrack then local a = Instance.new("Animation"); a.AnimationId = "rbxassetid://168086975"; _G.closerhandsTrack = hum:LoadAnimation(a); _G.closerhandsTrack:Play() end
            elseif not jerking then
                hum.WalkSpeed = ws; hum.JumpPower = jp
                if _G.jerkTrack then _G.jerkTrack:Stop(); _G.jerkTrack = nil end
                if _G.closerhandsTrack then _G.closerhandsTrack:Stop(); _G.closerhandsTrack = nil end
            end
        end)
        new.AncestryChanged:Connect(function()
            if not new.Parent then conn:Disconnect(); jerking = false; _G.jerkActive = false; hum.WalkSpeed = ws; hum.JumpPower = jp
                if btn then btn.Text = "🍆 JERK"; btn.BackgroundColor3 = Color3.fromRGB(180,140,200) end
            end
        end)
        if btn then btn.Text = "✅ JERK"; btn.BackgroundColor3 = Color3.fromRGB(100,200,100) end
    end
end

-- Función para activar/desactivar gravedad cero (solo eso)
function toggleZeroG(btn)
    _G.zeroGActive = not _G.zeroGActive
    local character = LP.Character
    if not character then return end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local rootPart = character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso")
    
    if _G.zeroGActive then
        -- Guardar valores originales
        _G.savedWalkSpeed = humanoid and humanoid.WalkSpeed or 16
        _G.savedJumpPower = humanoid and humanoid.JumpPower or 50
        
        -- Activar RAGDOLL
        if humanoid then
            humanoid.PlatformStand = true
            humanoid:ChangeState(Enum.HumanoidStateType.Physics)
        end
        
        -- Aplicar GRAVEDAD CERO
        if rootPart then
            -- Eliminar cualquier BodyForce anterior
            local oldForce = rootPart:FindFirstChild("ZeroGForce")
            if oldForce then oldForce:Destroy() end
            
            -- Crear BodyForce que cancela la gravedad
            local bodyForce = Instance.new("BodyForce")
            bodyForce.Name = "ZeroGForce"
            bodyForce.Force = Vector3.new(0, workspace.Gravity * rootPart.AssemblyMass, 0)
            bodyForce.Parent = rootPart
        end
        
        if btn then
            btn.Text = "🪐 GRAVEDAD 0: ON"
            btn.BackgroundColor3 = Color3.fromRGB(100, 100, 200)
        end
        
    else
        -- Desactivar modo
        if rootPart then
            local bodyForce = rootPart:FindFirstChild("ZeroGForce")
            if bodyForce then bodyForce:Destroy() end
        end
        
        -- Restaurar humanoid
        if humanoid then
            humanoid.PlatformStand = false
            humanoid:ChangeState(Enum.HumanoidStateType.Running)
            humanoid.WalkSpeed = _G.savedWalkSpeed
            humanoid.JumpPower = _G.savedJumpPower
        end
        
        if btn then
            btn.Text = "🪐 GRAVEDAD 0: OFF"
            btn.BackgroundColor3 = Color3.fromRGB(200, 150, 180)
        end
    end
end

function toggleFlyCar(btn)
    if _G.flyCarActive then
        local h = gR(); if h then if _G.flyCarBV then _G.flyCarBV:Destroy() end; if _G.flyCarBG then _G.flyCarBG:Destroy() end end
        _G.flyCarBV, _G.flyCarBG, _G.flyCarActive = nil, nil, false
        if btn then btn.Text = "OFF"; btn.BackgroundColor3 = Color3.fromRGB(200,150,120) end
    else
        local h = gR(); if not h then return end
        _G.flyCarBV = Instance.new("BodyVelocity", h); _G.flyCarBG = Instance.new("BodyGyro", h)
        _G.flyCarBV.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        RS.RenderStepped:Connect(function() if _G.flyCarBG and _G.flyCarBG.Parent then _G.flyCarBG.MaxTorque = Vector3.new(math.huge, math.huge, math.huge); _G.flyCarBG.CFrame = Camera.CFrame end end)
        _G.flyCarActive = true; if btn then btn.Text = "ON"; btn.BackgroundColor3 = Color3.fromRGB(150,200,150) end
    end
end
function flyCarForward() if _G.flyCarActive and _G.flyCarBV then _G.flyCarBV.Velocity = Camera.CFrame.LookVector * _G.flyCarSpeed end end
function flyCarBackward() if _G.flyCarActive and _G.flyCarBV then _G.flyCarBV.Velocity = Camera.CFrame.LookVector * -_G.flyCarSpeed end end
function flyCarStop() if _G.flyCarBV then _G.flyCarBV.Velocity = Vector3.new(0,0,0) end end

function toggleFly(btn)
    _G.isFlying = not _G.isFlying
    local c = LP.Character
    if not c then return end
    
    local hum, root = c:FindFirstChildOfClass("Humanoid"), c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("Torso")
    if not hum or not root then return end
    
    if _G.isFlying then
        -- Guardar valores originales
        _G.originalWalkSpeed = hum.WalkSpeed
        _G.originalJumpPower = hum.JumpPower
        
        -- Deshabilitar estados normales del humanoid
        for _, s in Enum.HumanoidStateType:GetEnumItems() do 
            hum:SetStateEnabled(s, false) 
        end
        hum:ChangeState(Enum.HumanoidStateType.Swimming)
        
        -- Crear y guardar las instancias de vuelo
        if _G.flyBG then _G.flyBG:Destroy() end
        if _G.flyBV then _G.flyBV:Destroy() end
        
        _G.flyBG = Instance.new("BodyGyro", root)
        _G.flyBG.MaxTorque = Vector3.new(9e9,9e9,9e9)
        
        _G.flyBV = Instance.new("BodyVelocity", root)
        _G.flyBV.MaxForce = Vector3.new(9e9,9e9,9e9)
        
        -- Conexión de vuelo
        if _G.flyConnection then
            _G.flyConnection:Disconnect()
        end
        
        _G.flyConnection = RS.RenderStepped:Connect(function()
            -- Verificar si todo sigue existiendo y el vuelo está activo
            if not _G.isFlying or not c or not c.Parent or not hum or not hum.Parent or not root or not root.Parent then
                if _G.flyConnection then
                    _G.flyConnection:Disconnect()
                    _G.flyConnection = nil
                end
                return
            end
            
            -- Aplicar fuerzas de vuelo
            if _G.flyBG and _G.flyBG.Parent then
                _G.flyBG.CFrame = Camera.CoordinateFrame
            end
            if _G.flyBV and _G.flyBV.Parent then
                if hum.MoveDirection.Magnitude > 0 then
                    _G.flyBV.Velocity = hum.MoveDirection * (45 * _G.flySpeed)
                else
                    _G.flyBV.Velocity = Vector3.new(0,0,0)
                end
            end
        end)
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(150,180,150) end
        
    else -- Desactivar vuelo
        -- Desconectar la conexión de vuelo
        if _G.flyConnection then
            _G.flyConnection:Disconnect()
            _G.flyConnection = nil
        end
        
        -- Destruir las instancias de vuelo
        if _G.flyBG then 
            pcall(function() _G.flyBG:Destroy() end)
            _G.flyBG = nil 
        end
        if _G.flyBV then 
            pcall(function() _G.flyBV:Destroy() end)
            _G.flyBV = nil 
        end
        
        -- Restaurar estados del humanoid
        for _, s in Enum.HumanoidStateType:GetEnumItems() do 
            pcall(function() hum:SetStateEnabled(s, true) end)
        end
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.RunningNoPhysics) end)
        
        -- Restaurar velocidad original
        pcall(function() 
            hum.WalkSpeed = _G.originalWalkSpeed or 16
            hum.JumpPower = _G.originalJumpPower or 50
        end)
        
        -- Asegurar que el personaje no tenga velocidad residual
        if root then
            pcall(function()
                root.Velocity = Vector3.new(0,0,0)
                root.RotVelocity = Vector3.new(0,0,0)
            end)
        end
        
        if btn then btn.BackgroundColor3 = Color3.fromRGB(210,160,130) end
    end
end

-- Animations
local Anims = {
    Walk = {Furry="102269417125238", Catwalk="109168724482748", Gojo="95643163365384", Geto="85811471336028", Adidas="122150855457006", Zombie="616168032", Ninja="656121766", Robot="10921250460", Stylish="10921276116", Sneaky="1132510133"},
    Run = {Furry="102269417125238", Adidas="82598234841035", Heavy="3236836670", Cartoony="10921076136", Zombie="616163682", Ninja="656118852", Robot="10921250460", Sneaky="1132494274"},
    Idle = {Furry={"102269417125238","102269417125238"}, Astronaut={"891621366","891633237"}, Bold={"16738333868","16738334710"}, Zombie={"616158929","616160636"}, Ninja={"656117400","656118341"}, Stylish={"616136790","616138447"}},
    Jump = {Furry="102269417125238", Adidas="75290611992385", Ninja="656117878", Robot="616090535", Stylish="616139451"},
    Fall = {Furry="102269417125238", Adidas="98600215928904", Ninja="656115606", Robot="616087089"}
}
local function setAnim(t, n)
    local d = Anims[t] and Anims[t][n]; if not d then return end
    local hum = gH(); if not hum then return end
    local a = LP.Character and LP.Character:FindFirstChild("Animate"); if not a then return end
    for _, tr in hum:GetPlayingAnimationTracks() do tr:Stop() end
    pcall(function()
        if t == "Idle" and type(d)=="table" then
    if a.idle then 
        a.idle.Animation1.AnimationId = "http://www.roblox.com/asset/?id="..d[1]
        a.idle.Animation2.AnimationId = "http://www.roblox.com/asset/?id="..d[2]
        -- Guardar la animación idle actual
        _G.savedIdleAnim1 = a.idle.Animation1.AnimationId
        _G.savedIdleAnim2 = a.idle.Animation2.AnimationId
    end
        elseif t == "Walk" and a.walk then a.walk.WalkAnim.AnimationId = "http://www.roblox.com/asset/?id="..d
        elseif t == "Run" and a.run then a.run.RunAnim.AnimationId = "http://www.roblox.com/asset/?id="..d
        elseif t == "Jump" and a.jump then a.jump.JumpAnim.AnimationId = "http://www.roblox.com/asset/?id="..d
        elseif t == "Fall" and a.fall then a.fall.FallAnim.AnimationId = "http://www.roblox.com/asset/?id="..d end
    end)
end

-- Función para cambiar la animación idle
local function setIdleAnimation(animName)
    local humanoid = gH()
    if not humanoid then return end
    
    local animate = LP.Character and LP.Character:FindFirstChild("Animate")
    if not animate then return end
    
    -- Detener animaciones actuales
    for _, track in pairs(humanoid:GetPlayingAnimationTracks()) do
        track:Stop()
    end
    
    -- Aplicar la nueva animación idle
    if animName == "Old School" then
        if animate.idle then
            animate.idle.Animation1.AnimationId = "http://www.roblox.com/asset/?id=616136790"
            animate.idle.Animation2.AnimationId = "http://www.roblox.com/asset/?id=616138447"
        end
    elseif animName == "Default" then
        -- Restaurar animación default
        if animate.idle then
            animate.idle.Animation1.AnimationId = "http://www.roblox.com/asset/?id=616139622"
            animate.idle.Animation2.AnimationId = "http://www.roblox.com/asset/?id=616140386"
        end
    elseif animName == "Furry" then
        if animate.idle then
            animate.idle.Animation1.AnimationId = "http://www.roblox.com/asset/?id=102269417125238"
            animate.idle.Animation2.AnimationId = "http://www.roblox.com/asset/?id=102269417125238"
        end
    end
end

-- Main loop
RS.Heartbeat:Connect(function()
    safeCall(function()
        pcall(function() sethiddenproperty(LP, "SimulationRadius", math.huge); sethiddenproperty(LP, "MaxSimulationRadius", math.huge) end)
        
        -- TORNADO EFFECT
        if _G.ringPartsEnabled then
            local hrp = gR()
            if hrp then
                local tornadoCenter = hrp.Position
                for _, part in pairs(_G.tornadoParts) do
                    if part and part.Parent and not part.Anchored then
                        -- EXCLUIR partes del personaje
                        if part.Parent ~= LP.Character and not part:IsDescendantOf(LP.Character) then
                            -- Aplicar CanCollide SOLO cuando el tornado está activo
                            part.CanCollide = false
                            
                            local pos = part.Position
                            local distance = (Vector3.new(pos.X, tornadoCenter.Y, pos.Z) - tornadoCenter).Magnitude
                            local angle = math.atan2(pos.Z - tornadoCenter.Z, pos.X - tornadoCenter.X)
                            local newAngle = angle + math.rad(_G.tornadoConfig.rotationSpeed)
                            local targetPos = Vector3.new(
                                tornadoCenter.X + math.cos(newAngle) * math.min(_G.tornadoConfig.radius, distance),
                                tornadoCenter.Y + (_G.tornadoConfig.height * (math.abs(math.sin((pos.Y - tornadoCenter.Y) / _G.tornadoConfig.height)))),
                                tornadoCenter.Z + math.sin(newAngle) * math.min(_G.tornadoConfig.radius, distance)
                            )
                            local directionToTarget = (targetPos - part.Position).unit
                            part.Velocity = directionToTarget * _G.tornadoConfig.attractionStrength
                        end
                    end
                end
            end
        end
        
        -- Actualizar posición del círculo (solo si existe)
        updateCirclePosition()
        
        -- Mover partes seleccionadas (solo si no está activo el tornado)
        if not _G.ringPartsEnabled then
            if next(_G.multiSelect) and not _G.followCamera then
                updateMultiVelocity()
            elseif _G.selectedPart and _G.selectedPart.Parent and not _G.followCamera then 
                updateVel(_G.selectedPart) 
            end
        end
        
        if _G.followCamera and not _G.ringPartsEnabled then
            if next(_G.multiSelect) then
                -- Seguir cámara con múltiples partes
                for _, part in pairs(_G.multiSelect) do
                    if part and part.Parent and LP.Character then
                        clearForces(part)
                        local r = LP.Character:FindFirstChild("HumanoidRootPart") or LP.Character:FindFirstChild("Torso")
                        if r then
                            local bp = Instance.new("BodyPosition", part)
                            bp.Name = "FollowBP"
                            bp.MaxForce = Vector3.new(math.huge,math.huge,math.huge)
                            bp.P = 20000; bp.D = 800
                            bp.Position = r.Position + (Camera.CFrame.LookVector * _G.followDistance)
                            
                            local bg = Instance.new("BodyGyro", part)
                            bg.Name = "FollowBG"
                            bg.MaxTorque = Vector3.new(math.huge,math.huge,math.huge)
                            bg.CFrame = part.CFrame
                        end
                    end
                end
            elseif _G.selectedPart and _G.selectedPart.Parent and LP.Character then
                clearForces(_G.selectedPart)
                local r = LP.Character:FindFirstChild("HumanoidRootPart") or LP.Character:FindFirstChild("Torso")
                if r then
                    local bp = Instance.new("BodyPosition", _G.selectedPart)
                    bp.Name = "FollowBP"
                    bp.MaxForce = Vector3.new(math.huge,math.huge,math.huge)
                    bp.P = 20000; bp.D = 800
                    bp.Position = r.Position + (Camera.CFrame.LookVector * _G.followDistance)
                    
                    local bg = Instance.new("BodyGyro", _G.selectedPart)
                    bg.Name = "FollowBG"
                    bg.MaxTorque = Vector3.new(math.huge,math.huge,math.huge)
                    bg.CFrame = _G.initialRotation
                end
            end
        end
    end)
end)

-- UI
local function CreateHub()
    local sg = Instance.new("ScreenGui", LP:WaitForChild("PlayerGui")); sg.Name = "Whathedogdoin"; sg.ResetOnSpawn = false

    local moon = Instance.new("TextButton", sg); moon.Name = "MoonButton"; moon.Size = UDim2.new(0,55,0,55); moon.Position = UDim2.new(0.02,0,0.5,-27.5); moon.BackgroundColor3 = Color3.fromRGB(120,90,0); moon.Text = "🧀\nRH"; moon.TextColor3 = Color3.fromRGB(255,255,0); moon.Font = Enum.Font.GothamBold; moon.TextSize = 12; moon.Visible = false; moon.Draggable = true
    Instance.new("UICorner", moon).CornerRadius = UDim.new(0,27.5)

    local main = Instance.new("Frame", sg)
main.Size = UDim2.new(0,340,0,615)
main.Position = UDim2.new(0.5,-170,0.5,-375)
main.BackgroundColor3 = Color3.fromRGB(240,215,170)
main.BorderSizePixel = 0
main.Active = true
main.Draggable = true
main.Visible = false  -- Oculto inicialmente
main.Transparency = 1
main.Rotation = -5
Instance.new("UICorner", main).CornerRadius = UDim.new(0,12)
local scale = Instance.new("UIScale", main); scale.Scale = 0.8

-- Pequeña pausa para asegurar que todo está cargado
task.wait(0.1)

-- Mostrar y animar
main.Visible = true
TS:Create(main, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
    Transparency = 0,
    Rotation = 0
}):Play()

TS:Create(scale, TweenInfo.new(0.6, Enum.EasingStyle.Elastic, Enum.EasingDirection.Out), {
    Scale = 1
}):Play()

    local topBtn = Instance.new("TextButton", main); topBtn.Size = UDim2.new(0,160,0,30); topBtn.Position = UDim2.new(0.5,-80,0,8); topBtn.Text = "▲ EXTRAS ▼"; topBtn.TextColor3 = Color3.fromRGB(255,255,255); topBtn.BackgroundColor3 = Color3.fromRGB(180,130,90); topBtn.BorderSizePixel = 0; topBtn.Font = Enum.Font.GothamBold; topBtn.TextSize = 12
    Instance.new("UICorner", topBtn).CornerRadius = UDim.new(0,8)

    local top = Instance.new("Frame", main); top.Size = UDim2.new(0,280,0,0); top.Position = UDim2.new(0.5,-140,0,-140); top.BackgroundColor3 = Color3.fromRGB(220,190,150); top.BorderSizePixel = 0; top.Visible = false; top.ClipsDescendants = true
    Instance.new("UICorner", top).CornerRadius = UDim.new(0,10)

    local ty, th = 10, 0
    local rag = createBtn(top, {0.45,0,0,28}, {0.05,0,0,ty}, "😆 RAGDOLL", Color3.fromRGB(210,160,130))
    local inf = createBtn(top, {0.45,0,0,28}, {0.5,0,0,ty}, "🦘 INF JUMP", Color3.fromRGB(150,170,200))
    local noc = createBtn(top, {0.45,0,0,28}, {0.05,0,0,ty+35}, "🚪 NOCLIP", Color3.fromRGB(150,170,200))
    local esp = createBtn(top, {0.45,0,0,28}, {0.5,0,0,ty+35}, "👁️ ESP", Color3.fromRGB(150,170,200))
    local rst = createBtn(top, {0.45,0,0,28}, {0.275,0,0,ty+70}, "🔄 RESET", Color3.fromRGB(180,140,100))
    th = ty + 105

    local title = Instance.new("TextLabel", main); title.Size = UDim2.new(1,-80,0,45); title.Position = UDim2.new(0,10,0,45); title.Text = "RIKESA HUB"; title.TextColor3 = Color3.fromRGB(255,255,255); title.BackgroundColor3 = Color3.fromRGB(200,160,120); title.BorderSizePixel = 0; title.Font = Enum.Font.GothamBold; title.TextSize = 20; title.TextXAlignment = Enum.TextXAlignment.Left
    Instance.new("UICorner", title).CornerRadius = UDim.new(0,10)

    local sd = createBtn(main, {0,28,0,28}, {1,-65,0,50}, "•", Color3.fromRGB(210,160,130))
    local su = createBtn(main, {0,28,0,28}, {1,-33,0,50}, "×", Color3.fromRGB(210,160,130))
    local close = createBtn(main, {0,28,0,28}, {1,-98,0,50}, "X", Color3.fromRGB(220,120,100))

    local y, ws, jp = 90, 16, 50

    local spdL = createLbl(main, {0.4,0,0,28}, {0.1,0,0,y}, "VEL: "..ws, Color3.fromRGB(200,120,80))
    local spdM = createBtn(main, {0,28,0,28}, {0.55,0,0,y}, "-", Color3.fromRGB(200,130,110))
    local spdP = createBtn(main, {0,28,0,28}, {0.7,0,0,y}, "+", Color3.fromRGB(150,180,150))
    spdM.MouseButton1Click:Connect(function() ws = math.max(10,ws-1); spdL.Text = "VEL: "..ws; local h = gH(); if h then h.WalkSpeed = ws end end)
    spdP.MouseButton1Click:Connect(function() ws = math.min(100,ws+1); spdL.Text = "VEL: "..ws; local h = gH(); if h then h.WalkSpeed = ws end end)

    y = y + 33
    local jmpL = createLbl(main, {0.4,0,0,28}, {0.1,0,0,y}, "SALTO: "..jp, Color3.fromRGB(200,120,80))
    local jmpM = createBtn(main, {0,28,0,28}, {0.55,0,0,y}, "-", Color3.fromRGB(200,130,110))
    local jmpP = createBtn(main, {0,28,0,28}, {0.7,0,0,y}, "+", Color3.fromRGB(150,180,150))
    jmpM.MouseButton1Click:Connect(function() jp = math.max(20,jp-5); jmpL.Text = "SALTO: "..jp; local h = gH(); if h then h.JumpPower = jp end end)
    jmpP.MouseButton1Click:Connect(function() jp = math.min(250,jp+5); jmpL.Text = "SALTO: "..jp; local h = gH(); if h then h.JumpPower = jp end end)

    y = y + 38
    local flySec = createFrame(main, {0.9,0,0,90}, {0.05,0,0,y}, Color3.fromRGB(220,190,150))
    createLbl(flySec, {1,0,0,22}, {0,0,0,3}, "✈️ VUELO", Color3.fromRGB(200,120,80), true)
    local flyB = createBtn(flySec, {0.3,0,0,32}, {0.1,0,0,28}, "FLY", Color3.fromRGB(210,160,130))
    local flySpd = createLbl(flySec, {0.15,0,0,32}, {0.45,0,0,28}, string.format("%.1f",_G.flySpeed), Color3.fromRGB(200,120,80))
    local flyM = createBtn(flySec, {0,24,0,24}, {0.6,0,0,32}, "-", Color3.fromRGB(200,130,110))
    local flyP = createBtn(flySec, {0,24,0,24}, {0.7,0,0,32}, "+", Color3.fromRGB(150,180,150))
    flyB.MouseButton1Click:Connect(function() toggleFly(flyB) end)
    flyM.MouseButton1Click:Connect(function() _G.flySpeed = math.max(0.1,_G.flySpeed-0.5); flySpd.Text = string.format("%.1f",_G.flySpeed) end)
    flyP.MouseButton1Click:Connect(function() _G.flySpeed = math.min(10,_G.flySpeed+0.5); flySpd.Text = string.format("%.1f",_G.flySpeed) end)

    y = y + 100
    local flingSec = createFrame(main, {0.9,0,0,80}, {0.05,0,0,y}, Color3.fromRGB(220,190,150))
    createLbl(flingSec, {1,0,0,20}, {0,0,0,3}, "🌀 FLING", Color3.fromRGB(200,120,80), true)
    local flingBox = createBox(flingSec, {0,50,0,25}, {0.05,0,0,28}, tostring(_G.flingSpeed))
    flingBox.FocusLost:Connect(function() local n = tonumber(flingBox.Text); if n and n>0 then _G.flingSpeed = n else flingBox.Text = tostring(_G.flingSpeed) end end)
    local flingB = createBtn(flingSec, {0.3,0,0,25}, {0.25,0,0,28}, "FLING: OFF", Color3.fromRGB(255,255,255))
    local antiB = createBtn(flingSec, {0.3,0,0,25}, {0.6,0,0,28}, "ANTIFLING: OFF", Color3.fromRGB(210,160,130))
    local flingM = createBtn(flingSec, {0,20,0,20}, {0.05,0,0,55}, "-", Color3.fromRGB(200,130,110))
    local flingP = createBtn(flingSec, {0,20,0,20}, {0.15,0,0,55}, "+", Color3.fromRGB(150,180,150))
    flingB.MouseButton1Click:Connect(function() toggleFling(flingB, flingBox) end)
    flingM.MouseButton1Click:Connect(function() _G.flingSpeed = math.max(1000,_G.flingSpeed-1000); flingBox.Text = tostring(_G.flingSpeed) end)
    flingP.MouseButton1Click:Connect(function() _G.flingSpeed = math.min(50000,_G.flingSpeed+1000); flingBox.Text = tostring(_G.flingSpeed) end)
    antiB.MouseButton1Click:Connect(function() toggleAntiFling(antiB) end)

    y = y + 90
    local userBox = createBox(main, {0.6,0,0,28}, {0.2,0,0,y}, "", "username")
    y = y + 33
    local tpB = createBtn(main, {0.6,0,0,32}, {0.2,0,0,y}, "🚀 TELEPORT", Color3.fromRGB(150,170,200))
    tpB.MouseButton1Click:Connect(function()
        for _, p in Players:GetPlayers() do if p.Name:lower():find(userBox.Text:lower()) and p~=LP and p.Character then
            local r = p.Character:FindFirstChild("HumanoidRootPart"); local mr = gR(); if r and mr then mr.CFrame = r.CFrame + Vector3.new(0,3,0) end; break
        end end
    end)

    y = y + 37
local godBtn = createBtn(main, {0.6,0,0,32}, {0.2,0,0,y}, "🛡️ GOD MODE: OFF", Color3.fromRGB(200,150,180))
godBtn.MouseButton1Click:Connect(function()
    toggleGodMode(godBtn)
end)

    y = y + 42
    local fcSec = createFrame(main, {0.9,0,0,95}, {0.05,0,0,y}, Color3.fromRGB(220,190,150))
    createLbl(fcSec, {1,0,0,22}, {0,0,0,3}, "🚗 FLY CAR", Color3.fromRGB(200,120,80), true)
    local fcB = createBtn(fcSec, {0.3,0,0,28}, {0.1,0,0,28}, "OFF", Color3.fromRGB(200,150,120))
    local fcSpd = createLbl(fcSec, {0.15,0,0,28}, {0.45,0,0,28}, tostring(_G.flyCarSpeed), Color3.fromRGB(200,120,80))
    local fcM = createBtn(fcSec, {0,24,0,24}, {0.6,0,0,30}, "-", Color3.fromRGB(200,130,110))
    local fcP = createBtn(fcSec, {0,24,0,24}, {0.7,0,0,30}, "+", Color3.fromRGB(150,180,150))
    local fcU = createBtn(fcSec, {0.35,0,0,28}, {0.1,0,0,60}, "▲", Color3.fromRGB(150,170,200))
    local fcD = createBtn(fcSec, {0.35,0,0,28}, {0.55,0,0,60}, "▼", Color3.fromRGB(150,170,200))
    fcB.MouseButton1Click:Connect(function() toggleFlyCar(fcB) end)
    fcM.MouseButton1Click:Connect(function() _G.flyCarSpeed = math.max(10,_G.flyCarSpeed-5); fcSpd.Text = tostring(_G.flyCarSpeed) end)
    fcP.MouseButton1Click:Connect(function() _G.flyCarSpeed = math.min(200,_G.flyCarSpeed+5); fcSpd.Text = tostring(_G.flyCarSpeed) end)
    fcU.MouseButton1Down:Connect(flyCarForward); fcU.MouseButton1Up:Connect(flyCarStop)
    fcD.MouseButton1Down:Connect(flyCarBackward); fcD.MouseButton1Up:Connect(flyCarStop)

    y = y + 100
    local trollBtn = createBtn(main, {0.9,0,0,35}, {0.05,0,0,y}, "👹 TROLL ▼", Color3.fromRGB(200,130,110))
    local troll = Instance.new("Frame", main); troll.Size = UDim2.new(0.9,0,0,0); troll.Position = UDim2.new(0.05,0,0,y+40); troll.BackgroundColor3 = Color3.fromRGB(220,190,150); troll.BorderSizePixel = 0; troll.Visible = false; troll.ClipsDescendants = true
    Instance.new("UICorner", troll).CornerRadius = UDim.new(0,8)

    local ty2 = 5
    local bang = createBtn(troll, {0.45,0,0,28}, {0.05,0,0,ty2}, "💥 BANG", Color3.fromRGB(200,130,110))
    local face = createBtn(troll, {0.45,0,0,28}, {0.5,0,0,ty2}, "😳 FACE", Color3.fromRGB(150,170,200))
    local sit = createBtn(troll, {0.45,0,0,28}, {0.05,0,0,ty2+30}, "🪑 SIT", Color3.fromRGB(150,180,150))
    local suck = createBtn(troll, {0.45,0,0,28}, {0.5,0,0,ty2+30}, "👅 SUCK", Color3.fromRGB(200,150,180))
    local jerk = createBtn(troll, {0.45,0,0,28}, {0.275,0,0,ty2+60}, "🍆 JERK", Color3.fromRGB(180,140,200))
    local zeroG = createBtn(troll, {0.45,0,0,28}, {0.275,0,0,ty2+90}, "🪐 GRAVEDAD 0: OFF", Color3.fromRGB(200,150,180))
    bang.MouseButton1Click:Connect(function() toggleBang(bang) end)
    face.MouseButton1Click:Connect(function() toggleFaceBang(face) end)
    sit.MouseButton1Click:Connect(function() toggleSit(sit) end)
    suck.MouseButton1Click:Connect(function() toggleSuck(suck) end)
    jerk.MouseButton1Click:Connect(function() toggleJerk(jerk) end)
    zeroG.MouseButton1Click:Connect(function() toggleZeroG(zeroG) end)

    local leftBtn = createBtn(main, {0,35,0,35}, {0,-10,0.5,-17.5}, "▶", Color3.fromRGB(180,130,90))
    leftBtn.TextSize = 22; leftBtn.ZIndex = 10
    local left = Instance.new("Frame", main); left.Size = UDim2.new(0,0,0,440); left.Position = UDim2.new(0,-255,0,90); left.BackgroundColor3 = Color3.fromRGB(220,190,150); left.BorderSizePixel = 0; left.Visible = false; left.ClipsDescendants = true
    Instance.new("UICorner", left).CornerRadius = UDim.new(0,10)
    createLbl(left, {1,0,0,35}, {0,0,0,0}, "ANIMACIONES", Color3.fromRGB(200,120,80))
    local scroll = Instance.new("ScrollingFrame", left); scroll.Size = UDim2.new(0.9,0,0.85,0); scroll.Position = UDim2.new(0.05,0,0.1,0); scroll.BackgroundColor3 = Color3.fromRGB(240,215,190); scroll.BorderSizePixel = 0; scroll.ScrollBarThickness = 6
    local ay = 5
    for t, an in pairs(Anims) do
        createLbl(scroll, {1,0,0,22}, {0,0,0,ay}, "=== "..t:upper().." ===", Color3.fromRGB(180,100,60), true); ay = ay + 24
        for n, _ in pairs(an) do
            local b = createBtn(scroll, {0.9,0,0,24}, {0.05,0,0,ay}, n, Color3.fromRGB(210,160,130))
            b.MouseButton1Click:Connect(function() setAnim(t, n) end); ay = ay + 26
        end; ay = ay + 5
    end; scroll.CanvasSize = UDim2.new(0,0,0,ay)

    local rightBtn = createBtn(main, {0,35,0,35}, {1,-20,0.5,-17.5}, "◀", Color3.fromRGB(180,130,90))
    rightBtn.TextSize = 22; rightBtn.ZIndex = 10
    local right = Instance.new("Frame", main); right.Size = UDim2.new(0,0,0,520); right.Position = UDim2.new(1,5,0,90); right.BackgroundColor3 = Color3.fromRGB(220,190,150); right.BorderSizePixel = 0; right.Visible = false; right.ClipsDescendants = true
    Instance.new("UICorner", right).CornerRadius = UDim.new(0,10)
    createLbl(right, {1,0,0,35}, {0,0,0,0}, "MOVE PARTS", Color3.fromRGB(200,120,80))

    local ay2, bs = 40, 42
    local up = createBtn(right, {0,bs,0,bs}, {0.5,-bs/2,0,ay2}, "▲", Color3.fromRGB(210,160,130))
    local down = createBtn(right, {0,bs,0,bs}, {0.5,-bs/2,0,ay2+bs+10}, "▼", Color3.fromRGB(210,160,130))
    local leftBtn2 = createBtn(right, {0,bs,0,bs}, {0.5,-bs*1.5-10,0,ay2+bs/2}, "◀", Color3.fromRGB(210,160,130))
    local rightBtn2 = createBtn(right, {0,bs,0,bs}, {0.5,bs/2+10,0,ay2+bs/2}, "▶", Color3.fromRGB(210,160,130))
    local fwd = createBtn(right, {0,bs,0,bs}, {0.5,-bs*1.5-10,0,ay2+bs*1.5+15}, "+", Color3.fromRGB(150,180,150))
    local bwd = createBtn(right, {0,bs,0,bs}, {0.5,bs/2+10,0,ay2+bs*1.5+15}, "-", Color3.fromRGB(200,130,110))

    local objSpdL = createLbl(right, {1,0,0,22}, {0,0,0,240}, "⚡ OBJ: ".._G.floatSpeed, Color3.fromRGB(200,120,80), true)
    local objM = createBtn(right, {0,32,0,28}, {0.3,0,0,265}, "-", Color3.fromRGB(200,130,110))
    local objP = createBtn(right, {0,32,0,28}, {0.6,0,0,265}, "+", Color3.fromRGB(150,180,150))
    local sel = createBtn(right, {0.4,0,0,32}, {0.05,0,0,300}, "SELECCIONAR", Color3.fromRGB(210,160,130))
    local stop = createBtn(right, {0.4,0,0,32}, {0.55,0,0,300}, "DETENER", Color3.fromRGB(200,130,110))
    local follow = createBtn(right, {0.6,0,0,32}, {0.2,0,0,340}, "◯ SEGUIR: OFF", Color3.fromRGB(210,160,130))

-- Botón TORNADO
local tornadoBtn = createBtn(right, {0.6,0,0,32}, {0.2,0,0,420}, "🌪️ TORNADO: OFF", Color3.fromRGB(210,160,130))

-- Sliders del tornado
local tornadoRadiusLabel = createLbl(right, {1,0,0,22}, {0,0,0,450}, "⚡ RADIO: ".._G.tornadoConfig.radius, Color3.fromRGB(200,120,80), true)
local tornadoRadiusSlider = Instance.new("Frame", right)
tornadoRadiusSlider.Size = UDim2.new(0.8,0,0,12)
tornadoRadiusSlider.Position = UDim2.new(0.1,0,0,470)
tornadoRadiusSlider.BackgroundColor3 = Color3.fromRGB(200,150,120)
tornadoRadiusSlider.BorderSizePixel = 0
Instance.new("UICorner", tornadoRadiusSlider).CornerRadius = UDim.new(0,6)
local tornadoRadiusCircle = Instance.new("TextButton", tornadoRadiusSlider)
tornadoRadiusCircle.Size = UDim2.new(0,16,0,16)
tornadoRadiusCircle.Position = UDim2.new((_G.tornadoConfig.radius-10)/990, -8, 0.5, -8)
tornadoRadiusCircle.Text = ""
tornadoRadiusCircle.BackgroundColor3 = Color3.fromRGB(255,215,190)
tornadoRadiusCircle.BorderSizePixel = 0
Instance.new("UICorner", tornadoRadiusCircle).CornerRadius = UDim.new(0,8)

local heightLabel = createLbl(right, {1,0,0,22}, {0,0,0,495}, "📏 ALTURA: ".._G.tornadoConfig.height, Color3.fromRGB(200,120,80), true)
local heightSlider = Instance.new("Frame", right)
heightSlider.Size = UDim2.new(0.8,0,0,12)
heightSlider.Position = UDim2.new(0.1,0,0,520)
heightSlider.BackgroundColor3 = Color3.fromRGB(200,150,120)
heightSlider.BorderSizePixel = 0
Instance.new("UICorner", heightSlider).CornerRadius = UDim.new(0,6)
local heightCircle = Instance.new("TextButton", heightSlider)
heightCircle.Size = UDim2.new(0,16,0,16)
heightCircle.Position = UDim2.new((_G.tornadoConfig.height-10)/990, -8, 0.5, -8)
heightCircle.Text = ""
heightCircle.BackgroundColor3 = Color3.fromRGB(255,215,190)
heightCircle.BorderSizePixel = 0
Instance.new("UICorner", heightCircle).CornerRadius = UDim.new(0,8)

-- Aumentar tamaño del panel
right.Size = UDim2.new(0,300,0,500)

-- Slider de distancia
local distL = createLbl(right, {1,0,0,22}, {0,0,0,375}, "🎯 ".._G.followDistance.."m", Color3.fromRGB(200,120,80), true)
local slider = Instance.new("Frame", right)
slider.Size = UDim2.new(0.8,0,0,12)
slider.Position = UDim2.new(0.1,0,0,400)
slider.BackgroundColor3 = Color3.fromRGB(200,150,120)
slider.BorderSizePixel = 0
Instance.new("UICorner", slider).CornerRadius = UDim.new(0,6)
local circle = Instance.new("TextButton", slider)
circle.Size = UDim2.new(0,16,0,16)
circle.Position = UDim2.new((_G.followDistance-5)/145, -8, 0.5, -8)
circle.Text = ""
circle.BackgroundColor3 = Color3.fromRGB(255,215,190)
circle.BorderSizePixel = 0
Instance.new("UICorner", circle).CornerRadius = UDim.new(0,8)

-- Variables de estado
local selActive, drag = false, false
local tornadoRadiusDrag, heightDrag = false, false

-- Conexiones de botones existentes
sel.MouseButton1Click:Connect(function() 
    selActive = not selActive; 
    sel.BackgroundColor3 = selActive and Color3.fromRGB(150,180,150) or Color3.fromRGB(210,160,130); 
    sel.Text = selActive and "ACTIVO" or "SELECCIONAR" 
end)

stop.MouseButton1Click:Connect(function() 
    _G.moveDirection = Vector3.new(0,0,0); 
    if _G.selectedPart and _G.selectedPart.Parent then 
        processPart(_G.selectedPart) 
    end 
end)

-- Botones de dirección
up.MouseButton1Click:Connect(function() 
    _G.moveDirection = getCamDir(Vector3.new(0,1,0))
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

down.MouseButton1Click:Connect(function() 
    _G.moveDirection = getCamDir(Vector3.new(0,-1,0))
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

leftBtn2.MouseButton1Click:Connect(function() 
    _G.moveDirection = getCamDir(Vector3.new(-1,0,0))
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

rightBtn2.MouseButton1Click:Connect(function() 
    _G.moveDirection = getCamDir(Vector3.new(1,0,0))
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

fwd.MouseButton1Click:Connect(function() 
    _G.moveDirection = getCamDir(Vector3.new(0,0,1))
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

bwd.MouseButton1Click:Connect(function() 
    _G.moveDirection = getCamDir(Vector3.new(0,0,-1))
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

objM.MouseButton1Click:Connect(function() 
    _G.floatSpeed = math.max(5,_G.floatSpeed-5)
    objSpdL.Text = "⚡ OBJ: ".._G.floatSpeed
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

objP.MouseButton1Click:Connect(function() 
    _G.floatSpeed = _G.floatSpeed+5
    objSpdL.Text = "⚡ OBJ: ".._G.floatSpeed
    if next(_G.multiSelect) then 
        processMultiPart()
    elseif _G.selectedPart then 
        processPart(_G.selectedPart) 
    end
end)

follow.MouseButton1Click:Connect(function()
    _G.followCamera = not _G.followCamera
    follow.Text = _G.followCamera and "✓ SEGUIR: ON" or "◯ SEGUIR: OFF"
    follow.BackgroundColor3 = _G.followCamera and Color3.fromRGB(150,180,150) or Color3.fromRGB(210,160,130)
    if _G.selectedPart then
        if _G.followCamera then 
            _G.initialRotation = _G.selectedPart.CFrame
            clearForces(_G.selectedPart)
        else 
            processPart(_G.selectedPart) 
        end
    end
end)

-- Drag del slider de distancia
circle.MouseButton1Down:Connect(function() drag = true end)

-- Drag de los sliders del tornado
tornadoRadiusCircle.MouseButton1Down:Connect(function() tornadoRadiusDrag = true end)
heightCircle.MouseButton1Down:Connect(function() heightDrag = true end)

-- ÚNICO InputChanged para todos los sliders
UIS.InputChanged:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
        -- Slider distancia
        if drag then
            local x = i.Position.X
            local ap, sz = slider.AbsolutePosition, slider.AbsoluteSize
            local r = math.clamp((x - ap.X)/sz.X, 0, 1)
            local nd = math.floor(5 + r * 145)
            _G.followDistance = nd
            circle.Position = UDim2.new(r, -8, 0.5, -8)
            distL.Text = "🎯 "..nd.."m"
        end
        
        -- Slider radio tornado
        if tornadoRadiusDrag then
            local x = i.Position.X
            local ap, sz = tornadoRadiusSlider.AbsolutePosition, tornadoRadiusSlider.AbsoluteSize
            local r = math.clamp((x - ap.X)/sz.X, 0, 1)
            local newRad = math.floor(10 + r * 990)
            _G.tornadoConfig.radius = newRad
            tornadoRadiusCircle.Position = UDim2.new(r, -8, 0.5, -8)
            tornadoRadiusLabel.Text = "⚡ RADIO: "..newRad
        end
        
                -- Slider altura
        if heightDrag then
            local x = i.Position.X
            local ap, sz = heightSlider.AbsolutePosition, heightSlider.AbsoluteSize
            local r = math.clamp((x - ap.X)/sz.X, 0, 1)
            local newHeight = math.floor(10 + r * 990)
            _G.tornadoConfig.height = newHeight
            heightCircle.Position = UDim2.new(r, -8, 0.5, -8)
            heightLabel.Text = "📏 ALTURA: "..newHeight
        end
    end
end)

UIS.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        drag = false
        tornadoRadiusDrag = false
        heightDrag = false
    end
end)

-- Botón tornado
tornadoBtn.MouseButton1Click:Connect(function()
    toggleTornado(tornadoBtn)
end)

    UIS.InputBegan:Connect(function(i)
        if selActive and (i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch) then
            local r = Camera:ViewportPointToRay(i.Position.X, i.Position.Y)
            local p = WS:FindPartOnRay(Ray.new(r.Origin, r.Direction * 1000), LP.Character)
            if p and not p.Anchored then
                if _G.selectedHighlight then _G.selectedHighlight:Destroy() end
                _G.selectedPart = p; _G.initialRotation = p.CFrame
                local h = Instance.new("SelectionBox", p); h.Adornee = p; h.Color3 = Color3.fromRGB(200,120,80); _G.selectedHighlight = h
                processPart(p); selActive = false; sel.BackgroundColor3 = Color3.fromRGB(210,160,130); sel.Text = "SELECCIONAR"
            end
        end
    end)

    local te, le, re, tre = false, false, false, false
    topBtn.MouseButton1Click:Connect(function() te = not te; topBtn.Text = te and "▼ EXTRAS ▲" or "▲ EXTRAS ▼"
        if te then top.Visible = true; TS:Create(top, TweenInfo.new(0.3), {Size = UDim2.new(0,280,0,th)}):Play()
        else TS:Create(top, TweenInfo.new(0.3), {Size = UDim2.new(0,280,0,0)}):Play(); task.wait(0.3); top.Visible = false end
    end)
    leftBtn.MouseButton1Click:Connect(function() le = not le; leftBtn.Text = le and "◀" or "▶"
        if le then left.Visible = true; TS:Create(left, TweenInfo.new(0.3), {Size = UDim2.new(0,250,0,440)}):Play()
        else TS:Create(left, TweenInfo.new(0.3), {Size = UDim2.new(0,0,0,440)}):Play(); task.wait(0.3); left.Visible = false end
    end)
    rightBtn.MouseButton1Click:Connect(function() re = not re; rightBtn.Text = re and "▶" or "◀"
        if re then right.Visible = true; TS:Create(right, TweenInfo.new(0.3), {Size = UDim2.new(0,300,0,555)}):Play()
        else TS:Create(right, TweenInfo.new(0.3), {Size = UDim2.new(0,0,0,555)}):Play(); task.wait(0.3); right.Visible = false end
    end)
    trollBtn.MouseButton1Click:Connect(function() tre = not tre; trollBtn.Text = tre and "👹 TROLL ▲" or "👹 TROLL ▼"
        if tre then troll.Visible = true; TS:Create(troll, TweenInfo.new(0.3), {Size = UDim2.new(0.9,0,0,125)}):Play()
        else TS:Create(troll, TweenInfo.new(0.3), {Size = UDim2.new(0.9,0,0,0)}):Play(); task.wait(0.3); troll.Visible = false end
    end)

    sd.MouseButton1Click:Connect(function() scale.Scale = math.max(0.5, scale.Scale - 0.1) end)
    su.MouseButton1Click:Connect(function() scale.Scale = math.min(1.5, scale.Scale + 0.1) end)
    close.MouseButton1Click:Connect(function()
        _G.hiddenfling = false; _G.isFlying = false; main.Visible = false; moon.Visible = true
        moon.Size = UDim2.new(0,0,0,0); TS:Create(moon, TweenInfo.new(0.3), {Size = UDim2.new(0,55,0,55)}):Play()
    end)
    moon.MouseButton1Click:Connect(function() main.Visible = true; moon.Visible = false end)

    -- Extra connections
    rag.MouseButton1Click:Connect(function() toggleRagdoll(rag) end)
    inf.MouseButton1Click:Connect(function() toggleInfiniteJump(inf) end)
    noc.MouseButton1Click:Connect(function() toggleNoclip(noc) end)
    esp.MouseButton1Click:Connect(function() toggleESP(esp) end)
    rst.MouseButton1Click:Connect(function() resetStats(rst) end)
end

-- UI Helpers
function createBtn(p, s, po, t, c)
    local b = Instance.new("TextButton", p); b.Size = UDim2.new(s[1],s[2],s[3],s[4]); b.Position = UDim2.new(po[1],po[2],po[3],po[4]); b.Text = t; b.TextColor3 = Color3.fromRGB(255,255,255); b.BackgroundColor3 = c; b.BorderSizePixel = 0; b.Font = Enum.Font.GothamBold; b.TextSize = 10
    Instance.new("UICorner", b).CornerRadius = UDim.new(0,6); return b
end
function createLbl(p, s, po, t, tc, trans)
    local l = Instance.new("TextLabel", p); l.Size = UDim2.new(s[1],s[2],s[3],s[4]); l.Position = UDim2.new(po[1],po[2],po[3],po[4]); l.Text = t; l.TextColor3 = tc; l.BackgroundColor3 = Color3.fromRGB(235,200,170); l.BorderSizePixel = 0; l.Font = Enum.Font.GothamBold; l.TextSize = 12
    if trans then l.BackgroundTransparency = 1 end; Instance.new("UICorner", l).CornerRadius = UDim.new(0,6); return l
end
function createFrame(p, s, po, c)
    local f = Instance.new("Frame", p); f.Size = UDim2.new(s[1],s[2],s[3],s[4]); f.Position = UDim2.new(po[1],po[2],po[3],po[4]); f.BackgroundColor3 = c; f.BorderSizePixel = 0
    Instance.new("UICorner", f).CornerRadius = UDim.new(0,10); return f
end
function createBox(p, s, po, t, ph)
    local b = Instance.new("TextBox", p); b.Size = UDim2.new(s[1],s[2],s[3],s[4]); b.Position = UDim2.new(po[1],po[2],po[3],po[4]); b.Text = t; b.PlaceholderText = ph or ""; b.TextColor3 = Color3.fromRGB(0,0,0); b.PlaceholderColor3 = Color3.fromRGB(150,100,70); b.BackgroundColor3 = Color3.fromRGB(255,255,255); b.BorderSizePixel = 0; b.Font = Enum.Font.Gotham; b.TextSize = 12
    Instance.new("UICorner", b).CornerRadius = UDim.new(0,6); return b
end

CreateHub()
