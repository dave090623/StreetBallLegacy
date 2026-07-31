--[[
    Script: Basquete PERFECT + TPWalk com Obsidian UI
    Descrição: Interface completa para arremesso PERFECT e TPWalk
    Biblioteca: Obsidian (https://github.com/deividcomsono/Obsidian)
]]

-- ===== CARREGAR BIBLIOTECA =====
local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()

-- ===== CONFIGURAÇÕES INICIAIS =====
local Options = Library.Options
local Toggles = Library.Toggles

-- ===== VARIÁVEIS GLOBAIS =====
local player = game.Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")

-- ============================================
-- ===== PARTE 1: SISTEMA DE ARREMESSO =====
-- ============================================

local netEvents = game:GetService("ReplicatedStorage"):WaitForChild("NetEvents")
local shootStart = netEvents:WaitForChild("RequestShootStart")
local shootEvent = netEvents:WaitForChild("RequestShoot")

-- Valores PERFECT (NÃO ALTERAR)
local PERFECT_VALUES = {
    releaseError = 0.01910017148645904,
    releaseProgress = 0.8661618232727051,
    tier = "PERFECT"
}

-- Estado do arremesso
local shootScriptEnabled = true
local shootInProgress = false

-- ===== FUNÇÃO DE POSIÇÃO DA MÃO =====
local function getHandPosition()
    if not character or not character.Parent then
        character = player.Character or player.CharacterAdded:Wait()
        humanoidRootPart = character:WaitForChild("HumanoidRootPart")
    end
    
    local pos = humanoidRootPart.Position
    local look = humanoidRootPart.CFrame.LookVector
    local right = humanoidRootPart.CFrame.RightVector
    
    -- Posição precisa da mão direita
    return pos + Vector3.new(
        look.X * 1.8 + right.X * 0.3,
        3.2,
        look.Z * 1.8 + right.Z * 0.3
    )
end

-- ===== FUNÇÃO PARA DISPARAR PERFECT =====
local function firePerfect()
    if not shootScriptEnabled then return false end
    if shootInProgress then return false end
    
    shootInProgress = true
    
    local args = {
        {
            releaseError = PERFECT_VALUES.releaseError,
            releaseProgress = PERFECT_VALUES.releaseProgress,
            handPosition = getHandPosition(),
            tier = PERFECT_VALUES.tier
        }
    }
    
    local success = pcall(function()
        shootEvent:FireServer(unpack(args))
    end)
    
    shootInProgress = false
    
    return success
end

-- ===== HOOK PRINCIPAL (VIA __NAMECALL) =====
local oldNamecall
oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
    local method = getnamecallmethod()
    
    if method == "FireServer" and self == shootStart and shootScriptEnabled then
        local result = oldNamecall(self, ...)
        
        task.spawn(function()
            local delay = Options.ShootTiming and Options.ShootTiming.Value or 0.05
            task.wait(delay)
            
            local maxAttempts = 3
            for attempt = 1, maxAttempts do
                if firePerfect() then
                    break
                end
                if attempt < maxAttempts then
                    task.wait(0.01)
                end
            end
        end)
        
        return result
    end
    
    return oldNamecall(self, ...)
end)

-- ============================================
-- ===== PARTE 2: SISTEMA DE TPWALK =====
-- ============================================

local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- Configurações do TPWalk
local MIN_SPEED = 0.1
local MAX_SPEED = 25
local DEFAULT_SPEED = 1

-- Estado do TPWalk
local tpwalking = false
local currentSpeed = DEFAULT_SPEED
local tpConnection = nil
local tpKeybind = Enum.KeyCode.Q

-- ===== FUNÇÃO PRINCIPAL DO TPWALK =====
local function tpWalk()
    local chr = player.Character
    local hum = chr and chr:FindFirstChildOfClass("Humanoid")
    
    while tpwalking and chr and hum and hum.Parent do
        local delta = RunService.Heartbeat:Wait()
        if hum.MoveDirection.Magnitude > 0 then
            chr:TranslateBy(hum.MoveDirection * currentSpeed * delta * 10)
        end
    end
end

-- ===== TOGGLE TPWALK =====
local function toggleTPWalk()
    tpwalking = not tpwalking
    
    if tpwalking then
        tpConnection = coroutine.create(tpWalk)
        coroutine.resume(tpConnection)
        Library:Notify({
            Title = "🏃 TPWalk Ativado",
            Description = string.format("Velocidade: %.2f", currentSpeed),
            Time = 2,
        })
    else
        Library:Notify({
            Title = "🏃 TPWalk Desativado",
            Time = 2,
        })
    end
end

-- ===== ATUALIZAR VELOCIDADE =====
local function updateSpeed(value)
    local newSpeed = tonumber(value)
    
    if newSpeed and newSpeed >= MIN_SPEED and newSpeed <= MAX_SPEED then
        currentSpeed = newSpeed
        return true
    end
    return false
end

-- ============================================
-- ===== PARTE 3: CRIAÇÃO DA INTERFACE =====
-- ============================================

-- ===== CONFIGURAÇÃO DA JANELA =====
local Window = Library:CreateWindow({
    Title = "🏀 Basquete + TPWalk",
    Footer = "Sistema Completo v2.0",
    Icon = 95816097006870,
    NotifySide = "Right",
    ShowCustomCursor = true,
    Resizable = true,
})

-- ============================================
-- ===== ABA: ARREMESSO (ÍCONE: BASKETBALL) =====
-- ============================================

local ShootTab = Window:AddTab("Arremesso", "basketball")

-- Grupo de Controles do Arremesso
local ShootControlGroup = ShootTab:AddLeftGroupbox("🎯 Controles de Arremesso", "target")

-- Toggle para ativar/desativar o arremesso
ShootControlGroup:AddToggle("ShootEnabled", {
    Text = "✅ Arremesso PERFECT Automático",
    Tooltip = "Ativa/Desativa o hook do arremesso PERFECT",
    Default = true,
    Callback = function(Value)
        shootScriptEnabled = Value
        if Value then
            Library:Notify({
                Title = "Arremesso Ativado",
                Description = "Todos os arremessos serão PERFECT!",
                Time = 2,
            })
        else
            Library:Notify({
                Title = "Arremesso Desativado",
                Description = "Arremessos voltam ao normal.",
                Time = 2,
            })
        end
    end
})

-- Slider para ajustar o timing
ShootControlGroup:AddSlider("ShootTiming", {
    Text = "⏱️ Timing do PERFECT",
    Default = 0.05,
    Min = 0.01,
    Max = 0.15,
    Rounding = 3,
    Suffix = "s",
    Tooltip = "Ajuste fino para acertar o PERFECT (0.01-0.15 segundos)",
    Callback = function(Value)
        print("[Timing] Ajustado para:", Value, "segundos")
    end
})

-- Slider para posição da mão
ShootControlGroup:AddSlider("HandHeight", {
    Text = "📏 Altura da Mão",
    Default = 3.2,
    Min = 2.5,
    Max = 4.5,
    Rounding = 1,
    Tooltip = "Ajuste a altura da posição da mão",
    Callback = function(Value)
        -- Atualiza a função de posição
        getHandPosition = function()
            local pos = humanoidRootPart.Position
            local look = humanoidRootPart.CFrame.LookVector
            local right = humanoidRootPart.CFrame.RightVector
            
            return pos + Vector3.new(
                look.X * 1.8 + right.X * 0.3,
                Value,
                look.Z * 1.8 + right.Z * 0.3
            )
        end
    end
})

-- Botão para testar arremesso
ShootControlGroup:AddButton({
    Text = "🧪 Testar Arremesso PERFECT",
    Func = function()
        Library:Notify({
            Title = "Testando Arremesso",
            Description = "Disparando PERFECT...",
            Time = 2,
        })
        
        pcall(function()
            shootStart:FireServer()
        end)
        
        task.wait(Options.ShootTiming.Value or 0.05)
        firePerfect()
    end,
    Tooltip = "Clique para testar o arremesso PERFECT manualmente"
})

-- Grupo de Informações do Arremesso
local ShootInfoGroup = ShootTab:AddRightGroupbox("📊 Informações", "info")

-- Labels com informações
ShootInfoGroup:AddLabel("📊 Status do Arremesso")
ShootInfoGroup:AddLabel("", false, "ShootStatusLabel")

-- Label de configurações
ShootInfoGroup:AddDivider()
ShootInfoGroup:AddLabel("⚙️ Configurações Atuais")
ShootInfoGroup:AddLabel("", false, "ShootConfigLabel")

-- ============================================
-- ===== ABA: TPWALK (ÍCONE: RUNNING) =====
-- ============================================

local TPWalkTab = Window:AddTab("TPWalk", "running")

-- Grupo de Controles do TPWalk
local TPControlGroup = TPWalkTab:AddLeftGroupbox("🏃 Controles de TPWalk", "settings")

-- Toggle principal do TPWalk
TPControlGroup:AddToggle("TPWalkEnabled", {
    Text = "🏃 TPWalk Ativado",
    Tooltip = "Ativa/Desativa o TPWalk",
    Default = false,
    Callback = function(Value)
        if Value and not tpwalking then
            toggleTPWalk()
        elseif not Value and tpwalking then
            toggleTPWalk()
        end
    end
})

-- Slider de velocidade
TPControlGroup:AddSlider("TPSpeed", {
    Text = "⚡ Velocidade do TPWalk",
    Default = DEFAULT_SPEED,
    Min = MIN_SPEED,
    Max = MAX_SPEED,
    Rounding = 2,
    Suffix = "x",
    Tooltip = "Velocidade do TPWalk (0.1 - 25)",
    Callback = function(Value)
        currentSpeed = Value
    end
})

-- Keybind do TPWalk
TPControlGroup:AddLabel("⌨️ Tecla para TPWalk")
    :AddKeyPicker("TPKeybind", {
        Default = "Q",
        Text = "Tecla TPWalk",
        Mode = "Press",
        Callback = function()
            toggleTPWalk()
            -- Sincroniza o toggle da UI
            if Toggles.TPWalkEnabled then
                Toggles.TPWalkEnabled:SetValue(tpwalking)
            end
        end
    })

-- Botão para criar botão externo
TPControlGroup:AddButton({
    Text = "🎮 Criar Botão Externo",
    Func = function()
        createExternalTPButton()
    end,
    Tooltip = "Cria um botão flutuante na tela para controlar o TPWalk"
})

-- Grupo de Status do TPWalk
local TPInfoGroup = TPWalkTab:AddRightGroupbox("📊 Status TPWalk", "info")

-- Label de status
TPInfoGroup:AddLabel("📊 Status do TPWalk")
TPInfoGroup:AddLabel("", false, "TPStatusLabel")

-- Label de velocidade atual
TPInfoGroup:AddDivider()
TPInfoGroup:AddLabel("⚡ Velocidade Atual")
TPInfoGroup:AddLabel("", false, "TPSpeedLabel")

-- ============================================
-- ===== ABA: UI SETTINGS (ÍCONE: SETTINGS) =====
-- ============================================

local UISettingsTab = Window:AddTab("UI", "settings")

-- Configurações da UI - Grupo Esquerdo
local MenuGroup = UISettingsTab:AddLeftGroupbox("🎨 Aparência", "paintbrush")

MenuGroup:AddToggle("ShowCustomCursor", {
    Text = "Cursor Personalizado",
    Default = true,
    Callback = function(Value)
        Library.ShowCustomCursor = Value
    end,
})

MenuGroup:AddSlider("UICornerSlider", {
    Text = "Raio das Bordas",
    Default = Library.CornerRadius,
    Min = 0,
    Max = 20,
    Rounding = 0,
    Callback = function(value)
        Window:SetCornerRadius(value)
    end
})

-- Grupo de DPI
local DPIGroup = UISettingsTab:AddRightGroupbox("📐 DPI", "monitor")
DPIGroup:AddDropdown("DPIDropdown", {
    Values = { "50%", "75%", "100%", "125%", "150%", "175%", "200%" },
    Default = "100%",
    Text = "Escala DPI",
    Callback = function(Value)
        Value = Value:gsub("%%", "")
        local DPI = tonumber(Value)
        Library:SetDPIScale(DPI)
    end,
})

-- ===== GRUPO DE KEYBINDS (ABAIXO DO TOGGLE DA LIBRARY) =====
local KeybindGroup = UISettingsTab:AddLeftGroupbox("⌨️ Keybinds", "keyboard")

KeybindGroup:AddLabel("🔑 Tecla para abrir o menu")
    :AddKeyPicker("MenuKeybind", { 
        Default = "RightShift", 
        NoUI = true, 
        Text = "Tecla do Menu" 
    })

KeybindGroup:AddLabel("🔒 Botão de Lock")
    :AddKeyPicker("LockKeybind", {
        Default = "RightControl",
        Text = "Tecla de Lock",
        Mode = "Toggle",
        Callback = function(Value)
            if Value then
                Library:ToggleLock()
                Library:Notify({
                    Title = "🔒 Menu Bloqueado",
                    Description = "Clique novamente para desbloquear",
                    Time = 2,
                })
            end
        end
    })

-- ===== BOTÃO DE UNLOAD =====
local UnloadGroup = UISettingsTab:AddRightGroupbox("🔄 Script", "power")

UnloadGroup:AddButton("🔄 Descarregar Script", function()
    Library:Unload()
end)

UnloadGroup:AddButton("💾 Salvar Configuração", function()
    SaveManager:Save()
    Library:Notify({
        Title = "Configuração Salva!",
        Time = 2,
    })
end)

-- ===== FUNÇÃO PARA CRIAR BOTÃO EXTERNO =====
local externalButtonCreated = false
local ExternalToggleGui = nil
local ExternalButton = nil

local function createRGBBorderExternal(parent, thickness)
    local border = Instance.new("Frame")
    border.Name = "RGBBorder"
    border.Size = UDim2.new(1, thickness * 2, 1, thickness * 2)
    border.Position = UDim2.new(0, -thickness, 0, -thickness)
    border.BackgroundColor3 = Color3.fromRGB(255,255,255)
    border.ZIndex = parent.ZIndex - 1
    border.Parent = parent
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = border
    
    local gradient = Instance.new("UIGradient")
    gradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255,0,0)),
        ColorSequenceKeypoint.new(0.15, Color3.fromRGB(255,128,0)),
        ColorSequenceKeypoint.new(0.33, Color3.fromRGB(255,255,0)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(0,255,0)),
        ColorSequenceKeypoint.new(0.66, Color3.fromRGB(0,128,255)),
        ColorSequenceKeypoint.new(0.82, Color3.fromRGB(0,0,255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(255,0,255)),
    })
    gradient.Parent = border
    
    spawn(function()
        while border.Parent do
            gradient.Rotation = (gradient.Rotation + 1) % 360
            task.wait(0.03)
        end
    end)
    
    return border
end

local function createExternalTPButton()
    -- Se já existe, remove
    if externalButtonCreated and ExternalToggleGui and ExternalToggleGui.Parent then
        ExternalToggleGui:Destroy()
        ExternalToggleGui = nil
        ExternalButton = nil
        externalButtonCreated = false
        Library:Notify({
            Title = "Botão Removido",
            Time = 2,
        })
        return
    end
    
    local CoreGui = game:GetService("CoreGui")
    
    ExternalToggleGui = Instance.new("ScreenGui")
    ExternalToggleGui.Name = "ExternalTPWalkButton"
    ExternalToggleGui.ResetOnSpawn = false
    ExternalToggleGui.Parent = CoreGui
    
    ExternalButton = Instance.new("TextButton")
    ExternalButton.Name = "ExternalButton"
    ExternalButton.Size = UDim2.new(0, 130, 0, 50)
    ExternalButton.Position = UDim2.new(0, 15, 0, 75)
    ExternalButton.Text = "❌ TPWalk OFF"
    ExternalButton.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    ExternalButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    ExternalButton.Font = Enum.Font.GothamBold
    ExternalButton.TextSize = 16
    ExternalButton.BorderSizePixel = 0
    ExternalButton.AutoButtonColor = true
    ExternalButton.ZIndex = 2
    ExternalButton.Parent = ExternalToggleGui
    
    local extCorner = Instance.new("UICorner")
    extCorner.CornerRadius = UDim.new(0, 10)
    extCorner.Parent = ExternalButton
    
    createRGBBorderExternal(ExternalButton, 2)
    
    -- Tornar arrastável
    local extDragging, extDragInput, extDragStart, extStartPos
    
    ExternalButton.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            extDragging = true
            extDragStart = input.Position
            extStartPos = ExternalButton.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    extDragging = false
                end
            end)
        end
    end)
    
    ExternalButton.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            extDragInput = input
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if input == extDragInput and extDragging then
            local delta = input.Position - extDragStart
            ExternalButton.Position = UDim2.new(extStartPos.X.Scale, extStartPos.X.Offset + delta.X, extStartPos.Y.Scale, extStartPos.Y.Offset + delta.Y)
        end
    end)
    
    -- Evento de clique
    ExternalButton.MouseButton1Click:Connect(function()
        toggleTPWalk()
        if Toggles.TPWalkEnabled then
            Toggles.TPWalkEnabled:SetValue(tpwalking)
        end
        if ExternalButton then
            ExternalButton.Text = tpwalking and "✅ TPWalk ON" or "❌ TPWalk OFF"
        end
    end)
    
    externalButtonCreated = true
    
    Library:Notify({
        Title = "Botão Externo Criado!",
        Description = "Clique nele para ativar/desativar o TPWalk",
        Time = 3,
    })
end

-- ============================================
-- ===== SISTEMA DE ATUALIZAÇÃO DE STATUS =====
-- ============================================

-- Função para atualizar o status do arremesso
local function updateShootStatus()
    local label = Options.ShootStatusLabel
    if label then
        local status = shootScriptEnabled and "🟢 ATIVO" or "🔴 DESATIVADO"
        label:SetText("Status: " .. status)
    end
    
    local configLabel = Options.ShootConfigLabel
    if configLabel then
        local timing = Options.ShootTiming and Options.ShootTiming.Value or 0.05
        local height = Options.HandHeight and Options.HandHeight.Value or 3.2
        configLabel:SetText(string.format(
            "Timing: %.3fs | Altura: %.1f",
            timing, height
        ))
    end
end

-- Função para atualizar o status do TPWalk
local function updateTPStatus()
    local label = Options.TPStatusLabel
    if label then
        local status = tpwalking and "🏃 ATIVO" or "⏸️ DESATIVADO"
        label:SetText("Status: " .. status)
    end
    
    local speedLabel = Options.TPSpeedLabel
    if speedLabel then
        speedLabel:SetText(string.format("Velocidade: %.2fx", currentSpeed))
    end
end

-- Conecta os eventos de mudança
Toggles.ShootEnabled:OnChanged(function()
    updateShootStatus()
end)

Options.ShootTiming:OnChanged(function()
    updateShootStatus()
end)

Options.HandHeight:OnChanged(function()
    updateShootStatus()
end)

Toggles.TPWalkEnabled:OnChanged(function()
    updateTPStatus()
end)

Options.TPSpeed:OnChanged(function()
    updateTPStatus()
end)

-- ============================================
-- ===== CONFIGURAÇÕES DOS MANAGERS =====
-- ============================================

Library.ToggleKeybind = Options.MenuKeybind
Library.LockKeybind = Options.LockKeybind -- Adiciona o keybind de lock

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)

SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ "MenuKeybind", "LockKeybind" })

ThemeManager:SetFolder("BasquetePerfect")
SaveManager:SetFolder("BasquetePerfect")
SaveManager:SetSubFolder("Arremesso")

-- Build da UI - Agora com os grupos organizados
SaveManager:BuildConfigSection(UISettingsTab)
ThemeManager:ApplyToTab(UISettingsTab)

-- ============================================
-- ===== EVENTOS DO TPWALK =====
-- ============================================

-- Atualiza o toggle da UI quando TPWalk é ativado/desativado
local function updateTPToggle()
    if Toggles.TPWalkEnabled then
        Toggles.TPWalkEnabled:SetValue(tpwalking)
    end
    updateTPStatus()
end

-- Sobrescreve toggleTPWalk para atualizar a UI
local originalToggle = toggleTPWalk
toggleTPWalk = function()
    originalToggle()
    updateTPToggle()
end

-- Atualiza a UI quando o botão externo for clicado
local originalCreateExternal = createExternalTPButton
createExternalTPButton = function()
    originalCreateExternal()
    updateTPToggle()
end

-- ============================================
-- ===== EVENTOS DE RESPAWN =====
-- ============================================

player.CharacterAdded:Connect(function(newChar)
    character = newChar
    humanoidRootPart = character:WaitForChild("HumanoidRootPart")
    
    if tpwalking then
        tpwalking = false
        updateTPToggle()
    end
end)

-- ============================================
-- ===== INICIALIZAÇÃO =====
-- ============================================

-- Atualiza os labels
task.wait(0.1)
updateShootStatus()
updateTPStatus()

-- Carrega configuração automática
SaveManager:LoadAutoloadConfig()

-- ===== NOTIFICAÇÃO DE INÍCIO =====
Library:Notify({
    Title = "🏀 Sistema Completo v2.0",
    Description = "Arremesso PERFECT + TPWalk carregados!",
    Time = 3,
})

print("✅ Sistema completo carregado com sucesso!")
print("📊 Abas disponíveis: Arremesso | TPWalk | UI")
print("🔑 Tecla padrão para abrir menu: RightShift")
print("🔑 Tecla padrão para TPWalk: Q")
print("🔒 Tecla padrão para Lock: RightControl")
