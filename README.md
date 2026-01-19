-- AntiRagdollModule (ModuleScript)
-- Coloque este ModuleScript em ServerScriptService com o nome "AntiRagdollModule"

local Players = game:GetService("Players")
local Anti = {}
Anti.__index = Anti

-- Estado por jogador: { enabled = bool, connections = { ... }, templates = Folder }
local playerState = {}

local function cleanupPlayer(player)
    local state = playerState[player]
    if not state then return end

    -- desconecta conexões
    if state.connections then
        for _, conn in ipairs(state.connections) do
            if conn and type(conn.Disconnect) == "function" then
                pcall(function() conn:Disconnect() end)
            elseif conn and type(conn.disconnect) == "function" then
                pcall(function() conn:disconnect() end)
            end
        end
    end

    -- remove pasta de templates se existir
    if state.templates and state.templates.Parent then
        pcall(function() state.templates:Destroy() end)
    end

    playerState[player] = nil
end

local function restoreMotorFromTemplate(template, character)
    if not template then return end
    -- tenta clonar template e reaplicar
    local ok, err = pcall(function()
        local newJoint = template:Clone()
        -- verifica se as partes ainda existem no personagem (Part0 e Part1 podem referenciar instâncias antigas)
        if newJoint.Part0 and newJoint.Part0.Parent and newJoint.Part1 and newJoint.Part1.Parent then
            newJoint.Parent = newJoint.Part0.Parent
        else
            -- fallback: parent no personagem
            newJoint.Parent = character
        end
    end)
    if not ok then
        warn("AntiRagdoll: falha ao restaurar joint: "..tostring(err))
    end
end

local function setupCharacter(player, character)
    if not playerState[player] or not playerState[player].enabled then return end
    if not character then return end

    local humanoid = character:FindFirstChildOfClass("Humanoid") or character:WaitForChild("Humanoid", 5)
    if not humanoid then return end

    -- cria pasta de templates com clones dos Motor6D
    local templates = Instance.new("Folder")
    templates.Name = "_AntiRagdoll_Joints"
    templates.Parent = character
    for _, d in ipairs(character:GetDescendants()) do
        if d:IsA("Motor6D") then
            local c = d:Clone()
            c.Parent = templates
        end
    end

    -- guarda a pasta no estado
    playerState[player].templates = templates

    -- conectar DescendantRemoving para tentar restaurar Motor6D removidos
    local descRemovingConn
    descRemovingConn = character.DescendantRemoving:Connect(function(desc)
        -- se proteção foi desligada, desconecta
        if not playerState[player] or not playerState[player].enabled then
            if descRemovingConn then descRemovingConn:Disconnect() end
            return
        end
        if desc:IsA("Motor6D") then
            -- pequena espera para garantir remoção concluída
            task.wait(0.06)
            -- procurar template com mesmo Name
            local template = templates:FindFirstChild(desc.Name, true) -- deep find
            if template then
                restoreMotorFromTemplate(template, character)
            else
                -- se não encontrou por nome, tenta restaurar por matching nas partes
                for _, t in ipairs(templates:GetDescendants()) do
                    if t:IsA("Motor6D") then
                        -- heurística: mesmo Part0/Part1 names
                        if t.Part0 and desc.Part0 and t.Part0.Name == desc.Part0.Name then
                            restoreMotorFromTemplate(t, character)
                            break
                        end
                    end
                end
            end
        end
    end)

    -- conectar StateChanged do Humanoid para evitar estados tipo ragdoll
    local stateChangedConn
    stateChangedConn = humanoid.StateChanged:Connect(function(oldState, newState)
        if not playerState[player] or not playerState[player].enabled then
            if stateChangedConn then stateChangedConn:Disconnect() end
            return
        end
        if newState == Enum.HumanoidStateType.Physics or newState == Enum.HumanoidStateType.FallingDown then
            pcall(function()
                humanoid.PlatformStand = false
                humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
            end)
        end
    end)

    -- armazena conexões para limpeza posterior
    table.insert(playerState[player].connections, descRemovingConn)
    table.insert(playerState[player].connections, stateChangedConn)
end

function Anti.EnableForPlayer(player)
    if not player or not player:IsA("Player") then return end

    if not playerState[player] then
        playerState[player] = { enabled = true, connections = {}, templates = nil }
    else
        playerState[player].enabled = true
    end

    -- conectar CharacterAdded
    local charConn = player.CharacterAdded:Connect(function(character)
        setupCharacter(player, character)
    end)
    table.insert(playerState[player].connections, charConn)

    -- se já tiver personagem, configura imediatamente
    if player.Character then
        task.spawn(function()
            setupCharacter(player, player.Character)
        end)
    end
end

function Anti.DisableForPlayer(player)
    if not player or not player:IsA("Player") then return end
    if not playerState[player] then return end

    playerState[player].enabled = false
    cleanupPlayer(player)
end

Players.PlayerRemoving:Connect(function(player)
    cleanupPlayer(player)
end)

return Anti
-- Script do servidor que recebe toggle do client
-- Coloque este Script em ServerScriptService com o nome "AntiRagdollServer"

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

-- garante que exista um RemoteEvent em ReplicatedStorage
local remote = ReplicatedStorage:FindFirstChild("AntiRagdollToggle")
if not remote then
    remote = Instance.new("RemoteEvent")
    remote.Name = "AntiRagdollToggle"
    remote.Parent = ReplicatedStorage
    print("AntiRagdollServer: RemoteEvent 'AntiRagdollToggle' criado em ReplicatedStorage")
end

local Anti = require(script.Parent:WaitForChild("AntiRagdollModule"))

remote.OnServerEvent:Connect(function(player, enabled)
    if type(enabled) ~= "boolean" then return end

    if enabled then
        Anti.EnableForPlayer(player)
        warn(("AntiRagdoll ativado para %s"):format(player.Name))
    else
        Anti.DisableForPlayer(player)
        warn(("AntiRagdoll desativado para %s"):format(player.Name))
    end
end)

Players.PlayerAdded:Connect(function(player)
    -- estado padrão: desligado; aguarda ação do cliente
end)
-- LocalScript que cria um pequeno hub de toggle na tela e comunica com o servidor
-- Coloque este LocalScript em StarterPlayerScripts (ou StarterGui) para que rode no cliente

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local remote = ReplicatedStorage:WaitForChild("AntiRagdollToggle")

-- cria GUI simples
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AntiRagdollHub"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Name = "MainFrame"
frame.Size = UDim2.new(0, 220, 0, 80)
frame.Position = UDim2.new(0, 12, 0, 12)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
frame.BorderSizePixel = 0
frame.Parent = screenGui
frame.Active = true
frame.Draggable = true

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, -12, 0, 24)
title.Position = UDim2.new(0, 6, 0, 6)
title.BackgroundTransparency = 1
title.Text = "AntiRagdoll Hub"
title.TextColor3 = Color3.fromRGB(230,230,230)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 18
title.Parent = frame

local statusLabel = Instance.new("TextLabel")
statusLabel.Name = "StatusLabel"
statusLabel.Size = UDim2.new(1, -12, 0, 20)
statusLabel.Position = UDim2.new(0, 6, 0, 30)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = "AntiRagdoll: OFF"
statusLabel.TextColor3 = Color3.fromRGB(200,0,0)
statusLabel.Font = Enum.Font.SourceSans
statusLabel.TextSize = 16
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Parent = frame

local toggleButton = Instance.new("TextButton")
toggleButton.Name = "ToggleButton"
toggleButton.Size = UDim2.new(0, 120, 0, 28)
toggleButton.Position = UDim2.new(1, -126, 1, -36)
toggleButton.AnchorPoint = Vector2.new(0, 0)
toggleButton.BackgroundColor3 = Color3.fromRGB(50,120,200)
toggleButton.TextColor3 = Color3.fromRGB(255,255,255)
toggleButton.Font = Enum.Font.SourceSansBold
toggleButton.TextSize = 16
toggleButton.Text = "Ligar"
toggleButton.Parent = frame

local enabled = false

local function updateGui()
    toggleButton.Text = enabled and "Desligar" or "Ligar"
    statusLabel.Text = enabled and "AntiRagdoll: ON" or "AntiRagdoll: OFF"
    statusLabel.TextColor3 = enabled and Color3.fromRGB(0,200,0) or Color3.fromRGB(200,0,0)
    if enabled then
        toggleButton.BackgroundColor3 = Color3.fromRGB(200,50,50)
    else
        toggleButton.BackgroundColor3 = Color3.fromRGB(50,120,200)
    end
end

toggleButton.MouseButton1Click:Connect(function()
    enabled = not enabled
    updateGui()
    pcall(function()
        remote:FireServer(enabled)
    end)
end)

updateGui()
