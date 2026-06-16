--[[
    Sistema Avançado de Combate - Parry e Esquiva Automática
    Detecta hitboxes de Hollows e reage a ataques M1 e Críticos
]]

local CombatSystem = {}
CombatSystem.__index = CombatSystem

-- Configurações
local CONFIG = {
    -- Distâncias de detecção
    DETECTION_RANGE = 15, -- Alcance para detectar Hollows
    PARRY_RANGE = 5, -- Alcance efetivo para parry
    DODGE_DISTANCE = 8, -- Distância do dodge
    
    -- Timing do parry (em segundos)
    PARRY_WINDOW_START = 0.1, -- Quanto tempo antes do impacto iniciar parry
    PARRY_WINDOW_END = 0.3, -- Margem de segurança após impacto
    
    -- Limiares de confiança
    ATTACK_CONFIDENCE_THRESHOLD = 0.7,
    CRITICAL_CONFIDENCE_THRESHOLD = 0.8,
    
    -- Cooldowns
    PARRY_COOLDOWN = 0.5,
    DODGE_COOLDOWN = 1.0,
    SCAN_INTERVAL = 0.05, -- Intervalo de scan em segundos
}

-- Estados de animação conhecidos para ataques
local ATTACK_ANIMATIONS = {
    NORMAL = {
        names = {"M1_Attack", "NormalSwing", "BasicAttack", "LightAttack"},
        speedThreshold = 0.5,
        durationMin = 0.3,
        durationMax = 1.0
    },
    CRITICAL = {
        names = {"CriticalAttack", "HeavySwing", "PowerAttack", "RedAttack", "Unblockable"},
        speedThreshold = 0.3,
        durationMin = 0.5,
        durationMax = 2.0,
        indicators = {"red_flash", "danger_zone", "critical_effect"}
    }
}

function CombatSystem.new()
    local self = setmetatable({}, CombatSystem)
    
    -- Estado interno
    self.activeHollows = {}
    self.lastParryTime = 0
    self.lastDodgeTime = 0
    self.isParrying = false
    self.isDodging = false
    self.attackPredictions = {}
    
    -- Sistema de tracking
    self.animationTracker = {}
    self.hitboxCache = {}
    
    -- Métricas de precisão
    self.successfulParries = 0
    self.failedParries = 0
    self.successfulDodges = 0
    
    -- Inicializar detecção
    self:initializeDetection()
    
    return self
end

function CombatSystem:initializeDetection()
    -- Configurar raycasting para detecção de hitboxes
    self.raycastParams = RaycastParams.new()
    self.raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    self.raycastParams.FilterDescendantsInstances = {game.Players.LocalPlayer.Character}
    
    print("[CombatSystem] Sistema de detecção inicializado")
end

function CombatSystem:detectNearbyHollows()
    local character = game.Players.LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then
        return {}
    end
    
    local playerPosition = character.HumanoidRootPart.Position
    local nearbyHollows = {}
    
    -- Procurar por Hollows na área
    for _, object in ipairs(workspace:GetDescendants()) do
        if object:IsA("Model") and (object.Name:find("Hollow") or object.Name:find("Hollow_")) then
            if object:FindFirstChild("Humanoid") and object:FindFirstChild("HumanoidRootPart") then
                local humanoid = object.Humanoid
                local rootPart = object.HumanoidRootPart
                
                if humanoid.Health > 0 then
                    local distance = (playerPosition - rootPart.Position).Magnitude
                    
                    if distance <= CONFIG.DETECTION_RANGE then
                        table.insert(nearbyHollows, {
                            model = object,
                            humanoid = humanoid,
                            rootPart = rootPart,
                            distance = distance,
                            tracked = self.activeHollows[object] ~= nil
                        })
                    end
                end
            end
        end
    end
    
    -- Atualizar lista de Hollows ativos
    local updatedHollows = {}
    for _, hollow in ipairs(nearbyHollows) do
        updatedHollows[hollow.model] = true
        
        if not self.activeHollows[hollow.model] then
            -- Novo Hollow detectado
            self:trackHollow(hollow.model)
        end
    end
    
    self.activeHollows = updatedHollows
    
    return nearbyHollows
end

function CombatSystem:trackHollow(hollowModel)
    -- Inicializar tracking de animação para o Hollow
    self.animationTracker[hollowModel] = {
        currentAnimation = nil,
        animationStartTime = 0,
        attackType = nil,
        confidence = 0,
        hitboxData = nil
    }
    
    -- Detectar hitbox do Hollow
    self:detectHitbox(hollowModel)
end

function CombatSystem:detectHitbox(hollowModel)
    local hitboxData = {
        size = nil,
        offset = nil,
        attackRange = 0
    }
    
    -- Procurar por partes que podem ser hitboxes
    for _, part in ipairs(hollowModel:GetDescendants()) do
        if part:IsA("BasePart") then
            -- Identificar possíveis partes de ataque
            if part.Name:find("Hitbox") or part.Name:find("Attack") or 
               part.Name:find("Sword") or part.Name:find("Weapon") then
                
                if part:FindFirstChildOfClass("TouchTransmitter") or 
                   part:FindFirstChild("Damage") then
                    
                    hitboxData.size = part.Size
                    hitboxData.offset = part.Position - hollowModel:FindFirstChild("HumanoidRootPart").Position
                    hitboxData.attackRange = part.Size.Magnitude * 1.5
                    hitboxData.part = part
                    
                    break
                end
            end
        end
    end
    
    -- Se não encontrou hitbox específica, estimar baseado no tamanho do Hollow
    if not hitboxData.size then
        local rootPart = hollowModel:FindFirstChild("HumanoidRootPart")
        if rootPart then
            hitboxData.attackRange = rootPart.Size.Magnitude * 1.2
            hitboxData.estimated = true
        end
    end
    
    self.hitboxCache[hollowModel] = hitboxData
    
    return hitboxData
end

function CombatSystem:analyzeHollowAnimation(hollow)
    local model = hollow.model
    local humanoid = hollow.humanoid
    local tracker = self.animationTracker[model]
    
    if not tracker then return end
    
    -- Tentar detectar animação atual do Hollow
    local animator = humanoid:FindFirstChild("Animator")
    local currentAnim = nil
    
    if animator then
        -- Verificar tracks de animação ativas
        for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
            if track.Animation then
                local animName = track.Animation.Name
                
                -- Verificar se é uma animação de ataque
                for _, attackName in ipairs(ATTACK_ANIMATIONS.NORMAL.names) do
                    if animName:find(attackName) then
                        currentAnim = {
                            name = animName,
                            startTime = tick(),
                            type = "NORMAL"
                        }
                        break
                    end
                end
                
                if not currentAnim then
                    for _, critName in ipairs(ATTACK_ANIMATIONS.CRITICAL.names) do
                        if animName:find(critName) then
                            currentAnim = {
                                name = animName,
                                startTime = tick(),
                                type = "CRITICAL"
                            }
                            break
                        end
                    end
                end
            end
        end
    end
    
    -- Método alternativo: detectar por movimento
    if not currentAnim then
        local velocity = humanoid.MoveDirection.Magnitude
        local rootVelocity = hollow.rootPart.Velocity.Magnitude
        
        -- Detectar animação por mudança brusca de velocidade
        if rootVelocity > ATTACK_ANIMATIONS.NORMAL.speedThreshold and not humanoid.Sit then
            local attackType = "NORMAL"
            
            -- Verificar indicadores visuais de ataque crítico
            if self:detectCriticalIndicators(model) then
                attackType = "CRITICAL"
            end
            
            currentAnim = {
                name = "Movement_Detected",
                startTime = tick(),
                type = attackType
            }
        end
    end
    
    -- Atualizar tracker
    if currentAnim then
        tracker.currentAnimation = currentAnim
        tracker.animationStartTime = currentAnim.startTime
        tracker.attackType = currentAnim.type
        tracker.confidence = currentAnim.type == "CRITICAL" and 0.9 or 0.75
    end
end

function CombatSystem:detectCriticalIndicators(hollowModel)
    -- Procurar por indicadores visuais de ataque crítico
    for _, part in ipairs(hollowModel:GetDescendants()) do
        if part:IsA("BasePart") then
            -- Verificar cores/efeitos que indicam ataque crítico
            if part.BrickColor == BrickColor.new("Really red") or
               part.BrickColor == BrickColor.new("Bright red") then
                return true
            end
            
            -- Verificar se há partículas ou efeitos
            for _, child in ipairs(part:GetChildren()) do
                if child:IsA("ParticleEmitter") and child.Enabled then
                    if child.Color and child.Color == Color3.new(1, 0, 0) then
                        return true
                    end
                end
            end
        end
    end
    
    return false
end

function CombatSystem:predictAttackImpact(hollow)
    local hitboxData = self.hitboxCache[hollow.model]
    local tracker = self.animationTracker[hollow.model]
    
    if not hitboxData or not tracker or not tracker.currentAnimation then
        return nil
    end
    
    local character = game.Players.LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then
        return nil
    end
    
    local playerPos = character.HumanoidRootPart.Position
    local hollowPos = hollow.rootPart.Position
    
    -- Calcular distância considerando hitbox
    local effectiveRange = hitboxData.attackRange
    
    -- Estimar tempo de impacto baseado na distância e velocidade
    local distance = (playerPos - hollowPos).Magnitude
    local hollowSpeed = hollow.rootPart.Velocity.Magnitude
    
    if hollowSpeed > 0.1 then
        local timeToImpact = distance / hollowSpeed
        
        -- Ajustar baseado no tipo de ataque
        local animationDuration = tracker.attackType == "CRITICAL" and 
                                  ATTACK_ANIMATIONS.CRITICAL.durationMax or 
                                  ATTACK_ANIMATIONS.NORMAL.durationMax
        
        if timeToImpact <= animationDuration then
            return {
                timeToImpact = timeToImpact,
                attackType = tracker.attackType,
                confidence = tracker.confidence,
                distance = distance,
                effectiveRange = effectiveRange
            }
        end
    end
    
    -- Se o Hollow está muito próximo, considerar como ataque iminente
    if distance <= effectiveRange then
        return {
            timeToImpact = 0.1, -- Muito próximo
            attackType = tracker.attackType or "NORMAL",
            confidence = 0.6,
            distance = distance,
            effectiveRange = effectiveRange
        }
    end
    
    return nil
end

function CombatSystem:executeParry()
    local currentTime = tick()
    
    -- Verificar cooldown
    if currentTime - self.lastParryTime < CONFIG.PARRY_COOLDOWN then
        return false
    end
    
    -- Verificar se não está em meio a outra ação
    if self.isDodging then
        return false
    end
    
    self.isParrying = true
    self.lastParryTime = currentTime
    
    -- Executar parry (pressionar F ou botão de bloqueio)
    -- Simular pressionamento de tecla
    pcall(function()
        -- Método 1: KeyPress
        local VirtualInputManager = game:GetService("VirtualInputManager")
        VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.F, false, nil)
        
        task.wait(0.05) -- Pequena duração do input
        
        VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.F, false, nil)
    end)
    
    task.wait(CONFIG.PARRY_WINDOW_END)
    self.isParrying = false
    
    return true
end

function CombatSystem:executeDodge()
    local currentTime = tick()
    
    -- Verificar cooldown
    if currentTime - self.lastDodgeTime < CONFIG.DODGE_COOLDOWN then
        return false
    end
    
    -- Verificar se não está em meio a outra ação
    if self.isParrying then
        return false
    end
    
    self.isDodging = true
    self.lastDodgeTime = currentTime
    
    -- Executar dodge para trás (tecla Q)
    pcall(function()
        local VirtualInputManager = game:GetService("VirtualInputManager")
        
        -- Pressionar Q para dodge
        VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Q, false, nil)
        
        task.wait(0.1) -- Duração do dodge
        
        VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Q, false, nil)
    end)
    
    task.wait(0.3) -- Tempo de recuperação
    self.isDodging = false
    
    return true
end

function CombatSystem:processAttack(hollow, impactPrediction)
    if not impactPrediction then return end
    
    local character = game.Players.LocalPlayer.Character
    if not character then return end
    
    -- Verificar se o ataque realmente vai atingir
    local playerHumanoid = character:FindFirstChild("Humanoid")
    if playerHumanoid and playerHumanoid.Health <= 0 then
        return -- Jogador já está morto
    end
    
    -- Verificar se há risco real de acerto
    if impactPrediction.distance > impactPrediction.effectiveRange * 1.2 then
        return -- Muito longe para ser atingido
    end
    
    -- Processar baseado no tipo de ataque
    if impactPrediction.attackType == "CRITICAL" then
        if impactPrediction.confidence >= CONFIG.CRITICAL_CONFIDENCE_THRESHOLD then
            print("[CombatSystem] Ataque crítico detectado! Executando dodge...")
            self:executeDodge()
            self.successfulDodges = self.successfulDodges + 1
        end
    else -- Ataque normal (M1)
        if impactPrediction.confidence >= CONFIG.ATTACK_CONFIDENCE_THRESHOLD then
            -- Verificar timing do parry
            if impactPrediction.timeToImpact <= CONFIG.PARRY_WINDOW_END and 
               impactPrediction.timeToImpact >= CONFIG.PARRY_WINDOW_START then
                
                print("[CombatSystem] Ataque normal detectado! Executando parry...")
                self:executeParry()
                self.successfulParries = self.successfulParries + 1
            end
        end
    end
end

function CombatSystem:update()
    -- Detectar Hollows próximos
    local nearbyHollows = self:detectNearbyHollows()
    
    if #nearbyHollows == 0 then return end
    
    -- Analisar cada Hollow
    for _, hollow in ipairs(nearbyHollows) do
        -- Atualizar análise de animação
        self:analyzeHollowAnimation(hollow)
        
        -- Prever impacto do ataque
        local impactPrediction = self:predictAttackImpact(hollow)
        
        -- Processar ataque se houver previsão
        if impactPrediction then
            self:processAttack(hollow, impactPrediction)
        end
    end
end

function CombatSystem:start()
    print("[CombatSystem] Sistema de combate iniciado!")
    print("[CombatSystem] Monitorando Hollows e aguardando ataques...")
    
    -- Loop principal
    self.running = true
    
    task.spawn(function()
        while self.running do
            local success, err = pcall(function()
                self:update()
            end)
            
            if not success then
                warn("[CombatSystem] Erro no update:", err)
            end
            
            task.wait(CONFIG.SCAN_INTERVAL)
        end
    end)
end

function CombatSystem:stop()
    self.running = false
    print("[CombatSystem] Sistema de combate parado.")
    print(string.format("[CombatSystem] Estatísticas - Parries: %d, Dodges: %d",
          self.successfulParries, self.successfulDodges))
end

function CombatSystem:getStats()
    return {
        successfulParries = self.successfulParries,
        failedParries = self.failedParries,
        successfulDodges = self.successfulDodges,
        activeHollows = #self.activeHollows
    }
end

-- Funções de debug
function CombatSystem:toggleDebug(enabled)
    self.debugMode = enabled
end

function CombatSystem:visualizeHitbox(hollow)
    if not self.debugMode then return end
    
    local hitboxData = self.hitboxCache[hollow.model]
    if hitboxData and hitboxData.part then
        -- Criar highlight visual
        local highlight = Instance.new("Highlight")
        highlight.FillColor = Color3.new(1, 1, 0)
        highlight.OutlineColor = Color3.new(1, 0.5, 0)
        highlight.Parent = hitboxData.part
        task.wait(0.1)
        highlight:Destroy()
    end
end

-- Inicializar e iniciar o sistema
local combatSystem = CombatSystem.new()
combatSystem:start()

-- Expor sistema globalmente para debug
_G.CombatSystem = combatSystem

return combatSystem
