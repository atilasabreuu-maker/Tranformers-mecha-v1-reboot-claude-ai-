--[[
	MECHA TRANSFORMER — LocalScript ÚNICO, 100% MOBILE-FIRST
	=========================================================
	ONDE COLOCAR: StarterPlayer > StarterPlayerScripts > (novo LocalScript) — cole tudo aqui.
	Não precisa de ModuleScript, RemoteEvent, servidor, modelo nem assets: o mecha é gerado por código.

	O QUE FAZ
	  • Humanoide -> Mecha (robô) -> Veículo (caminhão) -> Robô -> Humanoide
	  • Transformação em 4 fases: antecipação, avatar se dissolve em energia, peças voam e
	    se encaixam (com "Back easing"), finalização com flash + impacto
	  • Robô: caminhada procedural (pernas, joelhos, braços, tronco, respiração), cabeça segue a câmera,
	    passos pesados com tremor e poeira, pouso com amortecimento, giro "pesado"
	  • Veículo: direção arcade com o analógico (cima/baixo acelera, lados viram), rodas giram e esterçam,
	    carroceria inclina nas curvas
	  • TURBO (segura o botão): fogo nos propulsores, FOV aumenta, energia que recarrega
	  • UI feita para celular: botões grandes, zona segura (notch), reposiciona ao girar a tela,
	    barra de progresso, barra de energia; no PC mostra dicas de teclado
	  • Qualidade ADAPTATIVA: mede o FPS e reduz/aumenta partículas, luzes, blur e tremor sozinho

	CONTROLES
	  Celular : botões na tela (MECHA/VEÍCULO/ROBÔ, SAIR, TURBO)
	  PC      : T transforma | Y sai do mecha | Shift = turbo
	  Console : Y transforma | X sai | R1 = turbo

	LIMITAÇÃO IMPORTANTE (por ser só LocalScript): tudo é visível apenas para VOCÊ.
	Outros jogadores não veem o mecha nem a transformação. Para multiplayer use a versão
	com servidor (pasta MechaSystem).
]]

--------------------------------------------------------------------------------
-- SERVIÇOS
--------------------------------------------------------------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer

--------------------------------------------------------------------------------
-- CONFIGURAÇÃO (ajuste aqui)
--------------------------------------------------------------------------------
local CFG = {
	Keys = { Transform = Enum.KeyCode.T, Dismiss = Enum.KeyCode.Y, Boost = Enum.KeyCode.LeftShift },
	Pad = { Transform = Enum.KeyCode.ButtonY, Dismiss = Enum.KeyCode.ButtonX, Boost = Enum.KeyCode.ButtonR1 },

	Durations = { Summon = 3.6, Form = 2.6, Dismiss = 2.4, Death = 0.9 },

	-- HipHeight + RootSize.Y/2 = altura do centro do RootPart até o chão
	Robot = { WalkSpeed = 28, SprintSpeed = 52, JumpPower = 72, HipHeight = 8, RootSize = Vector3.new(4, 4, 2), TurnRate = 5.5 },
	Vehicle = { WalkSpeed = 75, SprintSpeed = 130, JumpPower = 0, HipHeight = 1.2, RootSize = Vector3.new(6, 4, 12), TurnRate = 2.2, Accel = 1.6, Brake = 4 },

	Boost = { Max = 100, Drain = 28, Regen = 14, RegenDelay = 1.2, MinToStart = 12 },

	Anim = { StrideRate = 0.21, LegSwing = 0.55, Knee = 0.7, ArmSwing = 0.35, Lean = 0.1, TorsoTwist = 0.06, BobHeight = 0.45 },

	Camera = {
		Fov = { Humanoid = 70, Robot = 72, Vehicle = 78 },
		BoostFovAdd = 14,
		ZoomMin = { Robot = 24, Vehicle = 28 },
		ZoomMax = { Robot = 80, Vehicle = 90 },
		Offset = { Humanoid = Vector3.new(0, 0, 0), Robot = Vector3.new(0, 5, 0), Vehicle = Vector3.new(0, 2.5, 0) },
		MaxShake = 1.1,
	},

	Explode = { Center = Vector3.new(0, 8, 0), Min = 10, Max = 24, OrbitTurns = 5.5, Lift = 3.2 },

	ReduceMotion = false, -- true = quase sem tremor/blur (acessibilidade ou aparelho fraco)
	AutoQuality = true,
	ShowButtonsOnDesktop = false,

	Colors = {
		Primary = Color3.fromRGB(70, 200, 255),
		Accent = Color3.fromRGB(255, 170, 40),
		Dark = Color3.fromRGB(12, 16, 24),
		Danger = Color3.fromRGB(255, 90, 90),
	},

	Text = {
		Summon = "SISTEMA DE TRANSFORMAÇÃO ATIVADO...",
		Form = "REARRANJANDO ESTRUTURA...",
		Dismiss = "DESATIVANDO SISTEMA...",
		ToRobot = "MECHA", ToVehicle = "VEÍCULO", ToRobotFromVehicle = "ROBÔ",
		Dismiss_Btn = "SAIR", Boost = "TURBO", Busy = "AGUARDE",
	},

	-- Sons: cole "rbxassetid://ID". Vazio = sem som (sem erro).
	Sounds = {
		Start = { Id = "", Volume = 0.8 },  -- servo motor
		Clunk = { Id = "", Volume = 1.0 },  -- clunk mecânico
		Surge = { Id = "", Volume = 0.9 },  -- energia
		Mid = { Id = "", Volume = 1.0 },    -- metal batendo
		End = { Id = "", Volume = 1.2 },    -- impacto pesado
		Step = { Id = "", Volume = 0.9 },   -- passo metálico
		Land = { Id = "", Volume = 1.2 },   -- pouso
	},
}

-- Linha do tempo da montagem (frações de 0 a 1 da duração)
local TL = {
	Ant = 0.14, DisEnd = 0.42, AsmStart = 0.40, AsmEnd = 0.86, PartDur = 0.26, Raise0 = 0.42, Raise1 = 0.60,
	Events = { { 0.00, "Start" }, { 0.14, "Clunk" }, { 0.40, "Surge" }, { 0.64, "Mid" }, { 0.88, "Finish" } },
	FormEvents = { { 0.02, "Start" }, { 0.50, "Mid" }, { 0.94, "Finish" } },
}

local PI = math.pi

--------------------------------------------------------------------------------
-- UTILIDADES
--------------------------------------------------------------------------------
local function clamp01(x) return x < 0 and 0 or (x > 1 and 1 or x) end
local function smooth(x) x = clamp01(x) return x * x * (3 - 2 * x) end
local function easeOutBack(x)
	x = clamp01(x)
	local c1 = 1.25
	local c3 = c1 + 1
	return 1 + c3 * (x - 1) ^ 3 + c1 * (x - 1) ^ 2
end
local function easeInOut(x)
	x = clamp01(x)
	if x < 0.5 then return 4 * x * x * x end
	return 1 - ((-2 * x + 2) ^ 3) / 2
end
local function damp(rate, dt) return 1 - math.exp(-rate * dt) end
local function hash01(name)
	local h = 7
	for i = 1, #name do h = (h * 31 + string.byte(name, i)) % 1000003 end
	return (h % 997) / 997
end

-- Ações (preenchidas mais abaixo; a UI chama estas funções)
local Actions = {}

--------------------------------------------------------------------------------
-- QUALIDADE ADAPTATIVA
--------------------------------------------------------------------------------
local TIERS = {
	{ name = "Low", part = 0.25, shake = 0.45, blur = 0, lights = false, trails = false, boxes = false },
	{ name = "Medium", part = 0.6, shake = 0.75, blur = 8, lights = true, trails = true, boxes = false },
	{ name = "High", part = 1.0, shake = 1.0, blur = 14, lights = true, trails = true, boxes = true },
}
local IS_MOBILE = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled
local tierIdx = IS_MOBILE and 2 or 3
local tierMax = IS_MOBILE and 2 or 3
do
	local ok, lvl = pcall(function() return settings().Rendering.QualityLevel.Value end)
	if ok and lvl > 0 then
		if lvl <= 4 then tierIdx = 1 elseif lvl <= 9 then tierIdx = math.min(tierIdx, 2) end
	end
	if CFG.ReduceMotion then tierIdx = 1 end
end
local function Q() return TIERS[tierIdx] end

do -- monitor de FPS
	local ema, timer, since, good = 1 / 60, 0, 0, 0
	RunService.Heartbeat:Connect(function(dt)
		if not CFG.AutoQuality then return end
		dt = math.clamp(dt, 1 / 240, 0.25)
		ema = ema + (dt - ema) * 0.05
		timer += dt
		since += dt
		if timer < 2 then return end
		timer = 0
		if since < 6 then return end
		local fps = 1 / ema
		if fps < 30 and tierIdx > 1 then
			tierIdx -= 1; since = 0; good = 0
		elseif fps > 56 and tierIdx < tierMax then
			good += 2
			if good >= 14 then tierIdx += 1; since = 0; good = 0 end
		else
			good = 0
		end
	end)
end

--------------------------------------------------------------------------------
-- ESTADO GLOBAL DA SESSÃO (um por personagem)
--------------------------------------------------------------------------------
local S = {
	ready = false,
	mode = "Humanoid",      -- modo estável atual
	tr = nil,               -- transição ativa {from,to,start,dur,gMecha,gOrig,gFrom,gTo,sizeSet}
	boosting = false,
	energy = CFG.Boost.Max,
	lastBoostEnd = 0,
	-- referências
	char = nil, hum = nil, root = nil, model = nil, entries = nil, fx = nil,
	conns = {},
	orig = nil,
}

--------------------------------------------------------------------------------
-- EFEITOS: fábrica de emissores (tudo criado UMA vez, nunca no loop)
--------------------------------------------------------------------------------
local TEX_SPARK = "rbxasset://textures/particles/sparkles_main.dds"
local TEX_SMOKE = "rbxasset://textures/particles/smoke_main.dds"
local TEX_FIRE = "rbxasset://textures/particles/fire_main.dds"

local function nseq(a, b, c)
	if c then
		return NumberSequence.new({ NumberSequenceKeypoint.new(0, a), NumberSequenceKeypoint.new(0.5, b), NumberSequenceKeypoint.new(1, c) })
	end
	return NumberSequence.new(a, b)
end

local function emitter(tex)
	local p = Instance.new("ParticleEmitter")
	p.Texture = tex
	p.Enabled = false
	p.Rate = 0
	p.LockedToPart = false
	return p
end

local function newSpark()
	local p = emitter(TEX_SPARK)
	p.Color = ColorSequence.new(Color3.fromRGB(255, 250, 200), Color3.fromRGB(255, 150, 40))
	p.LightEmission = 1
	p.Size = nseq(0.7, 0.35, 0)
	p.Lifetime = NumberRange.new(0.25, 0.6)
	p.Speed = NumberRange.new(18, 40)
	p.SpreadAngle = Vector2.new(180, 180)
	p.Acceleration = Vector3.new(0, -60, 0)
	p.Drag = 2
	p.Transparency = nseq(0, 0.2, 1)
	p.Rotation = NumberRange.new(0, 360)
	return p
end

local function newDust()
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(190, 180, 168))
	p.Size = nseq(1.6, 4.5, 7)
	p.Lifetime = NumberRange.new(0.7, 1.3)
	p.Speed = NumberRange.new(10, 20)
	p.SpreadAngle = Vector2.new(85, 85)
	p.EmissionDirection = Enum.NormalId.Top
	p.Drag = 3
	p.Transparency = nseq(0.45, 0.7, 1)
	p.Rotation = NumberRange.new(0, 360)
	p.RotSpeed = NumberRange.new(-40, 40)
	return p
end

local function newDissolve()
	local p = emitter(TEX_SPARK)
	p.Color = ColorSequence.new(CFG.Colors.Primary, Color3.fromRGB(255, 255, 255))
	p.LightEmission = 1
	p.Size = nseq(0.5, 0.35, 0)
	p.Lifetime = NumberRange.new(0.6, 1.2)
	p.Speed = NumberRange.new(4, 12)
	p.SpreadAngle = Vector2.new(180, 180)
	p.Acceleration = Vector3.new(0, 14, 0)
	p.Drag = 1.5
	p.Transparency = nseq(0, 0.3, 1)
	return p
end

local function newFlame()
	local p = emitter(TEX_FIRE)
	p.Color = ColorSequence.new(Color3.fromRGB(190, 240, 255), Color3.fromRGB(40, 120, 255))
	p.LightEmission = 1
	p.Size = nseq(1.5, 1.0, 0.2)
	p.Lifetime = NumberRange.new(0.25, 0.45)
	p.Speed = NumberRange.new(26, 42)
	p.SpreadAngle = Vector2.new(6, 6)
	p.EmissionDirection = Enum.NormalId.Bottom
	p.Transparency = nseq(0, 0.3, 1)
	p:SetAttribute("BaseRate", 110)
	return p
end

local function newThrusterSmoke()
	local p = emitter(TEX_SMOKE)
	p.Color = ColorSequence.new(Color3.fromRGB(120, 130, 150))
	p.Size = nseq(1.2, 3.2, 5)
	p.Lifetime = NumberRange.new(0.6, 1.1)
	p.Speed = NumberRange.new(6, 14)
	p.SpreadAngle = Vector2.new(14, 14)
	p.EmissionDirection = Enum.NormalId.Bottom
	p.Transparency = nseq(0.6, 0.8, 1)
	p:SetAttribute("BaseRate", 26)
	return p
end

local function burst(p, count)
	if p then p:Emit(math.max(1, math.floor(count * Q().part + 0.5))) end
end

--------------------------------------------------------------------------------
-- CÂMERA: FOV, zoom, offset, tremor, blur, flash
--------------------------------------------------------------------------------
local Cam = {
	trauma = 0, flash = 0, fov = 70, fovT = 70,
	zMin = 0.5, zMax = 128, zMinT = 0.5, zMaxT = 128, dMin = 0.5, dMax = 128,
	off = Vector3.zero, offT = Vector3.zero, blurT = 0, appliedMin = nil, appliedMax = nil,
}
local blurFx = Instance.new("BlurEffect")
blurFx.Name = "MechaBlur"
blurFx.Size = 0
blurFx.Enabled = false
blurFx.Parent = Lighting
local ccFx = Instance.new("ColorCorrectionEffect")
ccFx.Name = "MechaFlash"
ccFx.Parent = Lighting

local function shake(a)
	local k = CFG.ReduceMotion and 0.15 or Q().shake
	Cam.trauma = math.min(1, Cam.trauma + a * k)
end
local function flashScreen(a) Cam.flash = math.max(Cam.flash, a) end

local function camSetTargets(mode, boosting)
	local C = CFG.Camera
	Cam.fovT = (C.Fov[mode] or 70) + (boosting and C.BoostFovAdd or 0)
	Cam.offT = C.Offset[mode] or Vector3.zero
	if mode == "Humanoid" then
		Cam.zMinT, Cam.zMaxT = Cam.dMin, Cam.dMax
	else
		Cam.zMinT, Cam.zMaxT = C.ZoomMin[mode], C.ZoomMax[mode]
	end
end

local function camRender(dt)
	dt = math.min(dt, 0.1)
	local cam = Workspace.CurrentCamera
	if not cam then return end

	Cam.fov += (Cam.fovT - Cam.fov) * damp(5, dt)
	if math.abs(cam.FieldOfView - Cam.fov) > 0.01 then cam.FieldOfView = Cam.fov end

	Cam.zMin += (Cam.zMinT - Cam.zMin) * damp(2.2, dt)
	Cam.zMax += (Cam.zMaxT - Cam.zMax) * damp(2.2, dt)
	if not Cam.appliedMin or math.abs(Cam.zMin - Cam.appliedMin) > 0.05 or math.abs(Cam.zMax - Cam.appliedMax) > 0.05 then
		local nMin, nMax = Cam.zMin, math.max(Cam.zMax, Cam.zMin + 4)
		pcall(function()
			if nMax > player.CameraMaxZoomDistance then
				player.CameraMaxZoomDistance = nMax
				player.CameraMinZoomDistance = nMin
			else
				player.CameraMinZoomDistance = nMin
				player.CameraMaxZoomDistance = nMax
			end
		end)
		Cam.appliedMin, Cam.appliedMax = nMin, nMax
	end

	local hum = S.hum
	if hum and hum.Parent then
		Cam.off = Cam.off:Lerp(Cam.offT, damp(2.5, dt))
		if (hum.CameraOffset - Cam.off).Magnitude > 0.01 then hum.CameraOffset = Cam.off end
	end

	if Cam.trauma > 0 then
		Cam.trauma = math.max(0, Cam.trauma - dt * 1.5)
		local sh = Cam.trauma * Cam.trauma * CFG.Camera.MaxShake
		local t = os.clock() * 28
		cam.CFrame = cam.CFrame
			* CFrame.new(math.noise(t, 0.31, 1.7) * sh * 0.35, math.noise(0.77, t, 7.3) * sh * 0.35, 0)
			* CFrame.Angles(0, 0, math.noise(t, t, 13.1) * sh * 0.02)
	end

	local bt = (Q().blur > 0 and not CFG.ReduceMotion) and Cam.blurT * Q().blur or 0
	blurFx.Size += (bt - blurFx.Size) * damp(10, dt)
	blurFx.Enabled = blurFx.Size > 0.2
	Cam.flash = math.max(0, Cam.flash - dt * 2.2)
	ccFx.Brightness = Cam.flash * 0.45
	ccFx.Contrast = Cam.flash * 0.25
end

--------------------------------------------------------------------------------
-- INTERFACE MOBILE
--------------------------------------------------------------------------------
local UI = { buttons = {}, built = false }

local function corner(parent, r)
	local c = Instance.new("UICorner")
	c.CornerRadius = r
	c.Parent = parent
	return c
end
local function stroke(parent, color, th, tr)
	local s = Instance.new("UIStroke")
	s.Color = color
	s.Thickness = th
	s.Transparency = tr or 0
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = parent
	return s
end

local function makeButton(gui, textKey, color)
	local b = Instance.new("TextButton")
	b.Name = "Btn_" .. textKey
	b.AutoButtonColor = false
	b.Selectable = false
	b.Active = true
	b.AnchorPoint = Vector2.new(1, 0)
	b.BackgroundColor3 = CFG.Colors.Dark
	b.BackgroundTransparency = 0.25
	b.Text = ""
	b.Parent = gui
	corner(b, UDim.new(1, 0))
	local st = stroke(b, color, 3, 0.05)

	local ring = Instance.new("Frame") -- anel interno decorativo
	ring.AnchorPoint = Vector2.new(0.5, 0.5)
	ring.Position = UDim2.fromScale(0.5, 0.5)
	ring.Size = UDim2.fromScale(0.82, 0.82)
	ring.BackgroundTransparency = 1
	ring.Parent = b
	corner(ring, UDim.new(1, 0))
	stroke(ring, color, 1, 0.65)

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.AnchorPoint = Vector2.new(0.5, 0.5)
	label.Position = UDim2.fromScale(0.5, 0.5)
	label.Size = UDim2.fromScale(0.7, 0.34)
	label.Font = Enum.Font.GothamBlack
	label.TextScaled = true
	label.TextColor3 = Color3.new(1, 1, 1)
	label.Text = ""
	label.Parent = b

	local hint = Instance.new("TextLabel") -- tecla (só PC)
	hint.BackgroundTransparency = 1
	hint.AnchorPoint = Vector2.new(0.5, 0)
	hint.Position = UDim2.fromScale(0.5, 0.74)
	hint.Size = UDim2.fromScale(0.5, 0.16)
	hint.Font = Enum.Font.GothamMedium
	hint.TextScaled = true
	hint.TextColor3 = color
	hint.Text = ""
	hint.Parent = b

	local sc = Instance.new("UIScale")
	sc.Parent = b
	local function press(down)
		TweenService:Create(sc, TweenInfo.new(0.08, Enum.EasingStyle.Quad), { Scale = down and 0.9 or 1 }):Play()
	end
	b.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then press(true) end
	end)
	b.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then press(false) end
	end)

	local rec = { frame = b, label = label, hint = hint, stroke = st, color = color, enabled = true }
	UI.buttons[textKey] = rec
	return rec
end

local function buildUI()
	if UI.built then return end
	UI.built = true
	local gui = Instance.new("ScreenGui")
	gui.Name = "MechaUI"
	gui.ResetOnSpawn = false
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 50
	gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	pcall(function() gui.ScreenInsets = Enum.ScreenInsets.DeviceSafeInsets end)
	gui.Parent = player:WaitForChild("PlayerGui")
	UI.gui = gui

	-- botões
	local bT = makeButton(gui, "Transform", CFG.Colors.Primary)
	local bD = makeButton(gui, "Dismiss", CFG.Colors.Danger)
	local bB = makeButton(gui, "Boost", CFG.Colors.Accent)
	bT.frame.Activated:Connect(function() Actions.transform() end)
	bD.frame.Activated:Connect(function() Actions.dismiss() end)
	bD.label.Text = CFG.Text.Dismiss_Btn
	bB.label.Text = CFG.Text.Boost
	bT.hint.Text = IS_MOBILE and "" or "T"
	bD.hint.Text = IS_MOBILE and "" or "Y"
	bB.hint.Text = IS_MOBILE and "" or "SHIFT"

	-- TURBO: segurar. Rastreia o toque exato para soltar mesmo que o dedo saia do botão
	local boostInput
	bB.frame.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then
			boostInput = i
			Actions.boost(true)
		end
	end)
	UserInputService.InputEnded:Connect(function(i)
		if boostInput and i == boostInput then
			boostInput = nil
			Actions.boost(false)
		end
	end)

	-- painel de progresso (topo, centro)
	local panel = Instance.new("Frame")
	panel.Name = "Progress"
	panel.AnchorPoint = Vector2.new(0.5, 0)
	panel.BackgroundColor3 = CFG.Colors.Dark
	panel.BackgroundTransparency = 0.2
	panel.Visible = false
	panel.Parent = gui
	corner(panel, UDim.new(0, 10))
	stroke(panel, CFG.Colors.Primary, 2, 0.2)
	local title = Instance.new("TextLabel")
	title.BackgroundTransparency = 1
	title.Position = UDim2.fromScale(0.04, 0.08)
	title.Size = UDim2.fromScale(0.74, 0.42)
	title.Font = Enum.Font.GothamBlack
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.TextScaled = true
	title.TextColor3 = CFG.Colors.Primary
	title.Parent = panel
	local pct = Instance.new("TextLabel")
	pct.BackgroundTransparency = 1
	pct.Position = UDim2.fromScale(0.78, 0.08)
	pct.Size = UDim2.fromScale(0.18, 0.42)
	pct.Font = Enum.Font.GothamBold
	pct.TextXAlignment = Enum.TextXAlignment.Right
	pct.TextScaled = true
	pct.TextColor3 = Color3.new(1, 1, 1)
	pct.Parent = panel
	local track = Instance.new("Frame")
	track.Position = UDim2.fromScale(0.04, 0.6)
	track.Size = UDim2.fromScale(0.92, 0.22)
	track.BackgroundColor3 = Color3.fromRGB(30, 38, 52)
	track.BorderSizePixel = 0
	track.ClipsDescendants = true
	track.Parent = panel
	corner(track, UDim.new(1, 0))
	local fill = Instance.new("Frame")
	fill.Size = UDim2.fromScale(0, 1)
	fill.BackgroundColor3 = CFG.Colors.Primary
	fill.BorderSizePixel = 0
	fill.Parent = track
	corner(fill, UDim.new(1, 0))
	local grad = Instance.new("UIGradient")
	grad.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(40, 120, 255)),
		ColorSequenceKeypoint.new(0.5, Color3.fromRGB(120, 235, 255)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(40, 120, 255)),
	})
	grad.Parent = fill
	UI.panel, UI.title, UI.pct, UI.fill, UI.grad = panel, title, pct, fill, grad

	-- barra de energia (topo)
	local eb = Instance.new("Frame")
	eb.Name = "Energy"
	eb.AnchorPoint = Vector2.new(0.5, 0)
	eb.BackgroundColor3 = CFG.Colors.Dark
	eb.BackgroundTransparency = 0.3
	eb.Visible = false
	eb.Parent = gui
	corner(eb, UDim.new(1, 0))
	stroke(eb, CFG.Colors.Accent, 2, 0.3)
	local ef = Instance.new("Frame")
	ef.Size = UDim2.fromScale(1, 1)
	ef.BackgroundColor3 = CFG.Colors.Accent
	ef.BorderSizePixel = 0
	ef.Parent = eb
	corner(ef, UDim.new(1, 0))
	UI.energy, UI.energyFill = eb, ef

	-- dica de teclado (PC)
	local tip = Instance.new("TextLabel")
	tip.AnchorPoint = Vector2.new(0.5, 1)
	tip.BackgroundTransparency = 1
	tip.Font = Enum.Font.GothamMedium
	tip.TextColor3 = Color3.fromRGB(220, 235, 255)
	tip.TextStrokeTransparency = 0.5
	tip.Text = "T: transformar   •   Y: sair   •   SHIFT: turbo"
	tip.Visible = false
	tip.Parent = gui
	UI.tip = tip
end

local function layoutUI()
	if not UI.built then return end
	local cam = Workspace.CurrentCamera
	local vp = cam and cam.ViewportSize or Vector2.new(800, 400)
	local size = math.clamp(math.min(vp.X, vp.Y) * 0.17, 58, 96)
	local margin = size * 0.3
	local top = vp.Y * 0.24
	local gap = size * 0.22

	local B = UI.buttons
	for _, k in ipairs({ "Transform", "Dismiss", "Boost" }) do B[k].frame.Size = UDim2.fromOffset(size, size) end
	-- coluna da direita: transformar (cima) + sair (baixo); coluna ao lado: turbo
	B.Transform.frame.Position = UDim2.fromOffset(-margin, top)
	B.Dismiss.frame.Position = UDim2.fromOffset(-margin, top + size + gap)
	B.Boost.frame.Position = UDim2.fromOffset(-(margin + size + gap), top + size * 0.55)
	for _, k in ipairs({ "Transform", "Dismiss", "Boost" }) do B[k].frame.Position = UDim2.new(1, B[k].frame.Position.X.Offset, 0, B[k].frame.Position.Y.Offset) end

	local pw = math.clamp(vp.X * 0.6, 220, 440)
	UI.panel.Size = UDim2.fromOffset(pw, math.clamp(vp.Y * 0.13, 44, 62))
	UI.panel.Position = UDim2.new(0.5, 0, 0, 34)
	UI.energy.Size = UDim2.fromOffset(pw * 0.7, 9)
	UI.energy.Position = UDim2.new(0.5, 0, 0, 12)
	UI.tip.Size = UDim2.fromOffset(vp.X * 0.8, 22)
	UI.tip.TextSize = math.clamp(vp.Y * 0.035, 12, 18)
	UI.tip.Position = UDim2.new(0.5, 0, 1, -8)
end

local lastInput = UserInputService:GetLastInputType()
local function wantButtons()
	if CFG.ShowButtonsOnDesktop then return true end
	if lastInput == Enum.UserInputType.Touch then return true end
	return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled and not UserInputService.GamepadEnabled
end
UserInputService.LastInputTypeChanged:Connect(function(t) lastInput = t end)

local function setButton(rec, show, enabled, text)
	rec.frame.Visible = show
	if text then rec.label.Text = text end
	if rec.enabled ~= enabled then
		rec.enabled = enabled
		rec.frame.BackgroundTransparency = enabled and 0.25 or 0.65
		rec.label.TextTransparency = enabled and 0 or 0.55
		rec.stroke.Transparency = enabled and 0.05 or 0.6
	end
end

local function updateUI(t)
	if not UI.built then return end
	local mode = S.mode
	local tr = S.tr
	local active = mode ~= "Humanoid" or tr ~= nil
	local touch = wantButtons()
	local busy = tr ~= nil
	local B = UI.buttons

	local target = tr and tr.to or mode
	local text
	if busy then
		text = CFG.Text.Busy
	elseif mode == "Humanoid" then
		text = CFG.Text.ToRobot
	elseif mode == "Robot" then
		text = CFG.Text.ToVehicle
	else
		text = CFG.Text.ToRobotFromVehicle
	end
	setButton(B.Transform, touch, S.ready and not busy, text)
	setButton(B.Dismiss, touch and (mode ~= "Humanoid") and not (tr and tr.to == "Humanoid"), not busy, nil)
	setButton(B.Boost, touch and (target == "Robot" or target == "Vehicle"), not busy and S.energy >= CFG.Boost.MinToStart * 0.5, nil)

	UI.energy.Visible = (target ~= "Humanoid")
	if UI.energy.Visible then
		UI.energyFill.Size = UDim2.fromScale(clamp01(S.energy / CFG.Boost.Max), 1)
		UI.energyFill.BackgroundColor3 = S.energy < CFG.Boost.MinToStart and CFG.Colors.Danger or CFG.Colors.Accent
	end

	UI.panel.Visible = busy
	if busy then
		local u = clamp01((t - tr.start) / tr.dur)
		UI.fill.Size = UDim2.fromScale(u, 1)
		UI.pct.Text = string.format("%d%%", math.floor(u * 100))
		UI.title.Text = (tr.from == "Humanoid" and CFG.Text.Summon) or (tr.to == "Humanoid" and CFG.Text.Dismiss) or CFG.Text.Form
		UI.grad.Offset = Vector2.new(((t * 0.9) % 2) - 1, 0)
	end
	UI.tip.Visible = (not touch) and not busy
end

--------------------------------------------------------------------------------
-- SONS (opcionais)
--------------------------------------------------------------------------------
local function playSound(key, pitchVar)
	local fx = S.fx
	local snd = fx and fx.sounds[key]
	if not snd then return end
	local def = CFG.Sounds[key]
	snd.PlaybackSpeed = (def.Pitch or 1) * (1 + (math.random() - 0.5) * (pitchVar or 0))
	snd.TimePosition = 0
	snd:Play()
end

--------------------------------------------------------------------------------
-- MECHA PROCEDURAL (geometria + poses de Robô e Veículo)
-- Espaço das poses: origem no CHÃO sob o centro, Y para cima, frente = -Z
--------------------------------------------------------------------------------
local RED = Color3.fromRGB(176, 32, 38)
local BLUE = Color3.fromRGB(34, 70, 170)
local GBLUE = Color3.fromRGB(70, 90, 130)
local GRAY = Color3.fromRGB(120, 126, 136)
local DARK = Color3.fromRGB(38, 40, 46)
local GLOW = Color3.fromRGB(80, 230, 255)
local YELLOW = Color3.fromRGB(240, 190, 40)
local TIRE = Color3.fromRGB(26, 26, 30)
local HP = PI / 2

local STD_PARENT = {
	LowerTorso = false, UpperTorso = "LowerTorso", Head = "UpperTorso",
	LeftUpperArm = "UpperTorso", LeftLowerArm = "LeftUpperArm", LeftHand = "LeftLowerArm",
	RightUpperArm = "UpperTorso", RightLowerArm = "RightUpperArm", RightHand = "RightLowerArm",
	LeftUpperLeg = "LowerTorso", LeftLowerLeg = "LeftUpperLeg", LeftFoot = "LeftLowerLeg",
	RightUpperLeg = "LowerTorso", RightLowerLeg = "RightUpperLeg", RightFoot = "RightLowerLeg",
}
local ASM = {
	LeftFoot = 0, RightFoot = 0.03, LeftLowerLeg = 0.10, RightLowerLeg = 0.13, LeftUpperLeg = 0.20, RightUpperLeg = 0.23,
	LowerTorso = 0.36, UpperTorso = 0.52, LeftUpperArm = 0.60, RightUpperArm = 0.63, LeftLowerArm = 0.70, RightLowerArm = 0.73,
	LeftHand = 0.82, RightHand = 0.85, Head = 0.97,
}
local FORMORD = {
	LeftHand = 0, RightHand = 0.02, LeftFoot = 0.05, RightFoot = 0.07, LeftLowerArm = 0.12, RightLowerArm = 0.14,
	LeftLowerLeg = 0.18, RightLowerLeg = 0.20, LeftUpperArm = 0.28, RightUpperArm = 0.30, LeftUpperLeg = 0.36,
	RightUpperLeg = 0.38, Head = 0.60, LowerTorso = 0.78, UpperTorso = 0.90,
}

local function buildMecha(char, root)
	local model = Instance.new("Model")
	model.Name = "LocalMecha"
	local entries = {}
	local byName = {}
	local fx = { thrusters = {}, eyes = {}, hands = {}, feet = {}, sounds = {} }

	local function add(name, size, color, rPos, rRot, vPos, vRot, o)
		o = o or {}
		local p = Instance.new("Part")
		p.Name = name
		p.Size = size
		p.Color = color
		p.Material = o.mat or Enum.Material.Metal
		p.TopSurface = Enum.SurfaceType.Smooth
		p.BottomSurface = Enum.SurfaceType.Smooth
		p.CanCollide = false
		p.CanTouch = false
		p.CanQuery = false
		p.Massless = true
		p.Anchored = false
		p.Transparency = 1
		if o.cyl then p.Shape = Enum.PartType.Cylinder end
		rRot = rRot or Vector3.zero
		vRot = vRot or Vector3.zero
		local rcf = CFrame.new(rPos) * CFrame.Angles(rRot.X, rRot.Y, rRot.Z)
		local vcf = CFrame.new(vPos) * CFrame.Angles(vRot.X, vRot.Y, vRot.Z)
		p.CFrame = root.CFrame
		p.Parent = model

		local pivot = Vector3.zero
		if name:find("Arm") or name:find("Leg") or name:find("Hand") or name:find("Foot") then
			pivot = Vector3.new(0, size.Y * 0.5, 0)
		elseif name == "UpperTorso" or name == "Head" then
			pivot = Vector3.new(0, -size.Y * 0.5, 0)
		end

		local rp = rcf.Position
		local horiz = Vector3.new(rp.X, 0, rp.Z)
		if horiz.Magnitude < 0.5 then horiz = Vector3.new(0, 0, 1) end

		local e = {
			part = p, name = name, role = (STD_PARENT[name] ~= nil) and name or nil,
			robot = rcf, veh = vcf, pivot = (rcf * CFrame.new(pivot)).Position,
			parentName = o.parent, pi = 0, baseT = 0, vt = 1,
			wheel = o.wheel == true, steer = o.steer == true, radius = o.radius or 1.8,
			lift = (horiz.Unit * 0.9 + Vector3.new(0, 0.8, 0)) * CFG.Explode.Lift,
			motor = nil, lastC0 = nil,
		}
		table.insert(entries, e)
		byName[name] = e

		local m = Instance.new("Motor6D") -- liga a peça ao HumanoidRootPart
		m.Name = "MechaJoint"
		m.Part0 = root
		m.Part1 = p
		m.Parent = p
		e.motor = m
		return p, e
	end
	local function att(part, name, pos)
		local a = Instance.new("Attachment")
		a.Name = name
		a.Position = pos
		a.Parent = part
		return a
	end

	-- tronco / cabeça
	add("LowerTorso", Vector3.new(5.4, 2.2, 3.2), BLUE, Vector3.new(0, 10.6, 0), nil, Vector3.new(0, 3.7, -8.3), nil)
	local upper = add("UpperTorso", Vector3.new(6.4, 4.6, 3.8), RED, Vector3.new(0, 14, 0), nil, Vector3.new(0, 4.9, -4.8), nil)
	fx.chest = upper
	add("Head", Vector3.new(2.4, 2.4, 2.4), GRAY, Vector3.new(0, 17.9, 0), nil, Vector3.new(0, 7.6, -4.8), nil)
	add("Reactor", Vector3.new(1.6, 1.6, 0.3), GLOW, Vector3.new(0, 14.4, -1.95), nil, Vector3.new(0, 5.2, -6.85), nil, { mat = Enum.Material.Neon, parent = "UpperTorso" })
	for _, sx in ipairs({ -1, 1 }) do
		local eye, ee = add(sx < 0 and "EyeL" or "EyeR", Vector3.new(0.8, 0.35, 0.2), GLOW, Vector3.new(sx * 0.55, 18.1, -1.25), nil, Vector3.new(sx * 0.55, 7.8, -6.05), nil, { mat = Enum.Material.Neon, parent = "Head" })
		table.insert(fx.eyes, eye)
	end

	-- membros (esquerda = -X, direita = +X)
	for _, sx in ipairs({ -1, 1 }) do
		local s = sx < 0 and "Left" or "Right"
		add(s .. "UpperLeg", Vector3.new(2.4, 4.4, 2.6), BLUE, Vector3.new(sx * 1.7, 7.5, 0), nil, Vector3.new(sx * 2.5, 3.9, -0.6), Vector3.new(HP, 0, 0))
		add(s .. "LowerLeg", Vector3.new(2.0, 4.2, 2.2), GBLUE, Vector3.new(sx * 1.7, 3.3, 0), nil, Vector3.new(sx * 2.5, 3.7, 3.6), Vector3.new(HP, 0, 0))
		local foot = add(s .. "Foot", Vector3.new(2.6, 1.2, 4.0), DARK, Vector3.new(sx * 1.7, 0.6, -0.7), nil, Vector3.new(sx * 1.5, 3.2, 6.9), nil)
		fx.feet[sx < 0 and 1 or 2] = foot
		add(s .. "UpperArm", Vector3.new(2.0, 3.6, 2.0), RED, Vector3.new(sx * 4.7, 14.2, 0), nil, Vector3.new(sx * 3.9, 4.7, -2.0), nil)
		add(s .. "LowerArm", Vector3.new(2.2, 3.6, 2.4), GRAY, Vector3.new(sx * 4.7, 10.7, 0), nil, Vector3.new(sx * 3.6, 2.5, 1.0), Vector3.new(HP, 0, 0))
		local hand = add(s .. "Hand", Vector3.new(1.8, 1.6, 1.8), DARK, Vector3.new(sx * 4.7, 8.1, 0), nil, Vector3.new(sx * 2.1, 3.6, -9.9), nil)
		table.insert(fx.hands, hand)
		add("ShoulderPad_" .. s, Vector3.new(3.0, 0.8, 3.0), YELLOW, Vector3.new(sx * 4.9, 16.5, 0), nil, Vector3.new(sx * 2.9, 7.5, -4.8), nil, { parent = "UpperTorso" })
		local th = add("Thruster_" .. s, Vector3.new(1.2, 2.4, 1.2), DARK, Vector3.new(sx * 1.5, 8.4, 3.6), nil, Vector3.new(sx * 1.8, 3.4, 9.5), Vector3.new(-HP, 0, 0), { parent = "LowerTorso" })
		table.insert(fx.thrusters, att(th, "Thruster", Vector3.new(0, -1.2, 0)))
	end

	-- chassi (painéis nas costas) e rodas (rack nas costas)
	add("Chassis_A", Vector3.new(6.2, 1.2, 8.5), DARK, Vector3.new(-2, 13, 3), Vector3.new(HP, 0, 0), Vector3.new(0, 2, -4.25), nil, { parent = "UpperTorso" })
	add("Chassis_B", Vector3.new(6.2, 1.2, 8.5), DARK, Vector3.new(2, 13, 3.5), Vector3.new(HP, 0, 0), Vector3.new(0, 2, 4.25), nil, { parent = "UpperTorso" })
	local wh = { { "Wheel_FL", -3, 15.5, -3.7, -6, true }, { "Wheel_FR", 3, 15.5, 3.7, -6, true }, { "Wheel_RL", -3, 11, -3.7, 5.5, false }, { "Wheel_RR", 3, 11, 3.7, 5.5, false } }
	for _, w in ipairs(wh) do
		add(w[1], Vector3.new(1.6, 3.6, 3.6), TIRE, Vector3.new(w[2], w[3], 5.6), nil, Vector3.new(w[4], 1.8, w[5]), nil, { cyl = true, mat = Enum.Material.SmoothPlastic, parent = "UpperTorso", wheel = true, steer = w[6], radius = 1.8 })
	end

	-- hierarquia: padrão R15; extras seguem o "parent" informado
	for _, e in ipairs(entries) do
		local pe
		if e.parentName then pe = byName[e.parentName]
		elseif e.role and STD_PARENT[e.role] then pe = byName[STD_PARENT[e.role]] end
		e.parentEntry = pe
	end
	local ordered, seen = {}, {}
	local function visit(e)
		if seen[e] then return end
		seen[e] = true
		if e.parentEntry then visit(e.parentEntry) end
		table.insert(ordered, e)
	end
	for _, e in ipairs(entries) do visit(e) end
	for i, e in ipairs(ordered) do e.idx = i end
	for _, e in ipairs(ordered) do e.pi = e.parentEntry and e.parentEntry.idx or 0 end

	-- tempos por peça e pose "explodida" (de onde elas voam)
	local rng = Random.new(os.time() % 10007)
	local E = CFG.Explode
	for _, e in ipairs(ordered) do
		local ao = ASM[e.name] or (0.25 + 0.65 * hash01(e.name))
		e.as = TL.AsmStart + ao * (TL.AsmEnd - TL.AsmStart - TL.PartDur)
		e.ae = e.as + TL.PartDur
		local fo = FORMORD[e.name] or (0.1 + 0.6 * hash01(e.name .. "f"))
		e.fs = fo * 0.35
		e.fe = math.min(0.96, e.fs + 0.55)
		local dir = Vector3.new(rng:NextNumber(-1, 1), rng:NextNumber(-0.3, 1), rng:NextNumber(-1, 1))
		if dir.Magnitude < 0.2 then dir = Vector3.new(0, 1, 0) end
		local pos = E.Center + dir.Unit * rng:NextNumber(E.Min, E.Max)
		pos = Vector3.new(pos.X, math.max(pos.Y, 3), pos.Z)
		e.exploded = CFrame.new(pos) * CFrame.Angles(rng:NextNumber(-PI, PI), rng:NextNumber(-PI, PI), rng:NextNumber(-PI, PI))
		e.baseT = 0
		if e.name:sub(1, 3) == "Eye" then e.glow = true end
	end

	model.Parent = char
	return model, ordered, fx
end

--------------------------------------------------------------------------------
-- EFEITOS DO MECHA (criados uma vez por sessão)
--------------------------------------------------------------------------------
local function buildFX(fx, root)
	local anchor = Instance.new("Attachment")
	anchor.Name = "MechaFxAnchor"
	anchor.Parent = root
	fx.anchor = anchor
	fx.spark = newSpark(); fx.spark.Parent = anchor
	fx.dust = newDust(); fx.dust.Parent = anchor
	fx.dissolve = newDissolve(); fx.dissolve.Parent = anchor

	fx.flames = {}
	for _, a in ipairs(fx.thrusters) do
		local f, sm = newFlame(), newThrusterSmoke()
		f.Parent = a
		sm.Parent = a
		table.insert(fx.flames, { f, sm })
	end

	-- luz de flash e luz do reator
	fx.flash = Instance.new("PointLight")
	fx.flash.Color = CFG.Colors.Primary
	fx.flash.Range = 38
	fx.flash.Brightness = 0
	fx.flash.Parent = root
	fx.reactor = Instance.new("PointLight")
	fx.reactor.Color = CFG.Colors.Primary
	fx.reactor.Range = 16
	fx.reactor.Brightness = 0
	fx.reactor.Enabled = false
	fx.reactor.Parent = fx.chest

	-- anel de energia no chão
	local ring = Instance.new("Part")
	ring.Name = "MechaRing"
	ring.Shape = Enum.PartType.Cylinder
	ring.Size = Vector3.new(0.3, 2, 2)
	ring.Anchored = true
	ring.CanCollide = false
	ring.CanQuery = false
	ring.CanTouch = false
	ring.CastShadow = false
	ring.Material = Enum.Material.Neon
	ring.Color = CFG.Colors.Primary
	ring.Transparency = 1
	ring.Parent = Workspace.CurrentCamera
	fx.ring = ring
	fx.ringT = nil

	-- rastros nos punhos
	fx.trails = {}
	for _, hand in ipairs(fx.hands) do
		local a0 = Instance.new("Attachment"); a0.Position = Vector3.new(0, hand.Size.Y * 0.5, 0); a0.Parent = hand
		local a1 = Instance.new("Attachment"); a1.Position = Vector3.new(0, -hand.Size.Y * 0.5, 0); a1.Parent = hand
		local t = Instance.new("Trail")
		t.Attachment0, t.Attachment1 = a0, a1
		t.Lifetime = 0.25
		t.MinLength = 0.1
		t.LightEmission = 0.8
		t.Color = ColorSequence.new(CFG.Colors.Primary, Color3.new(1, 1, 1))
		t.Transparency = nseq(0.3, 0.6, 1)
		t.Enabled = false
		t.Parent = hand
		table.insert(fx.trails, t)
	end

	-- caixas de seleção nos olhos (só qualidade Alta)
	fx.boxes = {}
	for _, eye in ipairs(fx.eyes) do
		local b = Instance.new("SelectionBox")
		b.Adornee = eye
		b.Color3 = CFG.Colors.Primary
		b.SurfaceColor3 = CFG.Colors.Primary
		b.SurfaceTransparency = 0.6
		b.LineThickness = 0.03
		b.Visible = false
		b.Parent = eye
		table.insert(fx.boxes, b)
	end

	-- sons
	for key, def in pairs(CFG.Sounds) do
		if def.Id and def.Id ~= "" then
			local s = Instance.new("Sound")
			s.Name = "Mecha_" .. key
			s.SoundId = def.Id
			s.Volume = def.Volume or 1
			s.RollOffMode = Enum.RollOffMode.InverseTapered
			s.RollOffMaxDistance = 200
			s.Parent = root
			fx.sounds[key] = s
		end
	end
end

--------------------------------------------------------------------------------
-- STATUS / TRANSIÇÕES
--------------------------------------------------------------------------------
local function groundDist()
	return S.hum.HipHeight + S.root.Size.Y * 0.5
end

-- troca o tamanho do RootPart mantendo a altura do corpo
local function setRootSize(size)
	local g = groundDist()
	S.root.Size = size
	S.hum.HipHeight = math.max(0, g - size.Y * 0.5)
end

local function applyStats(mode)
	local hum, root = S.hum, S.root
	if mode == "Humanoid" then
		local o = S.orig
		hum.WalkSpeed = o.walk
		hum.UseJumpPower = o.useJP
		hum.JumpPower = o.jp
		hum.JumpHeight = o.jh
		hum.AutoRotate = true
		if root.Size ~= o.size then setRootSize(o.size) end
		hum.HipHeight = o.hip
	else
		local st = CFG[mode]
		hum.UseJumpPower = true
		hum.WalkSpeed = S.boosting and st.SprintSpeed or st.WalkSpeed
		hum.JumpPower = st.JumpPower
		hum.AutoRotate = false
		if root.Size ~= st.RootSize then setRootSize(st.RootSize) end
		hum.HipHeight = st.HipHeight
	end
end

local function setBoost(on)
	if S.boosting == on then return end
	if on and (S.mode == "Humanoid" or S.tr or S.energy < CFG.Boost.MinToStart) then return end
	S.boosting = on
	if not on then S.lastBoostEnd = os.clock() end
	if S.ready and not S.tr and S.mode ~= "Humanoid" then
		local st = CFG[S.mode]
		S.hum.WalkSpeed = on and st.SprintSpeed or st.WalkSpeed
	end
end

local function beginTransition(to, durOverride)
	if not S.ready or S.tr or to == S.mode then return end
	local hum, root = S.hum, S.root
	if hum.Health <= 0 then return end
	local from = S.mode
	local kind, dur
	if from == "Humanoid" then kind, dur = "Summon", CFG.Durations.Summon
	elseif to == "Humanoid" then kind, dur = "Dismiss", CFG.Durations.Dismiss
	else kind, dur = "Form", CFG.Durations.Form end
	dur = durOverride or dur
	setBoost(false)

	if kind == "Summon" then
		S.orig = { walk = hum.WalkSpeed, useJP = hum.UseJumpPower, jp = hum.JumpPower, jh = hum.JumpHeight, hip = hum.HipHeight, size = root.Size }
	end

	local tr = { from = from, to = to, start = os.clock(), dur = dur, kind = kind, sizeSet = false }
	local gNow = groundDist()
	if kind == "Summon" then
		local st = CFG[to]
		setRootSize(st.RootSize) -- invisível: só prepara a colisão
		tr.gOrig = gNow
		tr.gMecha = st.HipHeight + st.RootSize.Y * 0.5
		tr.sizeSet = true
	elseif kind == "Dismiss" then
		local o = S.orig
		tr.gMecha = gNow
		tr.gOrig = o.hip + o.size.Y * 0.5
	else
		local st = CFG[to]
		tr.gFrom = gNow
		tr.gTo = st.HipHeight + st.RootSize.Y * 0.5
	end

	hum.UseJumpPower = true
	hum.WalkSpeed = 0
	hum.JumpPower = 0
	S.tr = tr
	S.evKey, S.evVal = nil, nil
	S.hiddenApplied = false
end

Actions.transform = function()
	if not S.ready or S.tr then return end
	if S.mode == "Humanoid" then beginTransition("Robot")
	elseif S.mode == "Robot" then beginTransition("Vehicle")
	else beginTransition("Robot") end
end
Actions.dismiss = function()
	if S.ready and not S.tr and S.mode ~= "Humanoid" then beginTransition("Humanoid") end
end
Actions.boost = function(on) setBoost(on) end

--------------------------------------------------------------------------------
-- EVENTOS DA LINHA DO TEMPO (sons, faíscas, anel, flash, tremor)
--------------------------------------------------------------------------------
local M -- estado de movimento/animação (definido mais abaixo)

local function groundPos()
	return S.root.Position - Vector3.new(0, groundDist() - 0.4, 0)
end

local function fireEvent(name)
	local fx = S.fx
	if name == "Start" then
		playSound("Start", 0.05)
		shake(0.2)
	elseif name == "Clunk" then
		playSound("Clunk", 0.08)
		shake(0.3)
		fx.anchor.WorldPosition = S.root.Position
		burst(fx.spark, 14)
	elseif name == "Surge" then
		playSound("Surge", 0.05)
		shake(0.35)
		fx.anchor.WorldPosition = S.root.Position
		burst(fx.dissolve, 60)
		fx.ringT = 0
		flashScreen(0.25)
	elseif name == "Mid" then
		playSound("Mid", 0.08)
		shake(0.4)
	elseif name == "Finish" then
		playSound("End", 0.05)
		shake(0.9)
		flashScreen(0.8)
		fx.flashLevel = 1
		fx.anchor.WorldPosition = groundPos()
		burst(fx.dust, 26)
		burst(fx.spark, 20)
	end
end

local function onLand(impact)
	local fx = S.fx
	M.springV = impact * 7
	fx.anchor.WorldPosition = groundPos()
	burst(fx.dust, math.floor(14 * impact))
	shake(0.5 * impact)
	playSound("Land", 0.08)
end

local function onStep(side)
	local fx = S.fx
	local foot = fx.feet[side]
	if not foot then return end
	fx.anchor.WorldPosition = foot.Position - Vector3.new(0, 0.6, 0)
	burst(fx.dust, 4)
	shake(0.22)
	playSound("Step", 0.12)
end

--------------------------------------------------------------------------------
-- ATUALIZAÇÃO PRINCIPAL: pose, animação, efeitos (RenderStepped)
--------------------------------------------------------------------------------
local rx, ry, rz = {}, {}, {}
M = {
	phase = 0, lastStep = 0, spd = 0, air = 0, airTime = 0, lastVy = 0, grounded = true, wasGrounded = true,
	spring = 0, springV = 0, aimYaw = 0, aimPitch = 0, spin = 0, steer = 0, lastYaw = nil, yawRate = 0,
	lastFwd = 0, accel = 0, avatarAlpha = 1, tick = 0, vehSpeed = 0, dustTimer = 0, boostSm = 0,
}
local avatarParts = {}

local function setAvatarAlpha(a, force)
	if math.abs(a - M.avatarAlpha) > 0.003 or (a < 0.999 and force) then
		M.avatarAlpha = a
		local v = 1 - a
		for i = #avatarParts, 1, -1 do
			local p = avatarParts[i]
			if p.Parent then p.LocalTransparencyModifier = v else table.remove(avatarParts, i) end
		end
	end
end

local function updateMotion(dt, t)
	local root, hum = S.root, S.hum
	local vel = root.AssemblyLinearVelocity
	local look = root.CFrame.LookVector
	local hs = math.sqrt(vel.X * vel.X + vel.Z * vel.Z)
	local fwd = vel.X * look.X + vel.Z * look.Z
	M.hs, M.fwd = hs, fwd

	local grounded = hum.FloorMaterial ~= Enum.Material.Air
	if grounded then
		if not M.wasGrounded and M.airTime > 0.25 then
			onLand(math.clamp(-M.lastVy / 70, 0.15, 1.5))
		end
		M.airTime = 0
		M.lastVy = 0
	else
		M.airTime += dt
		if vel.Y < M.lastVy then M.lastVy = vel.Y end
	end
	M.wasGrounded = grounded
	M.grounded = grounded
	M.air += ((grounded and 0 or 1) - M.air) * damp(9, dt)

	-- mola do amortecimento de pouso
	M.springV += (-M.spring * 160 - M.springV * 14) * dt
	M.spring += M.springV * dt

	local ref = CFG.Robot.WalkSpeed
	M.spd += (clamp01(hs / ref) * (hs > 0.5 and 1 or 0) - M.spd) * damp(8, dt)
	if grounded and hs > 0.5 then M.phase += hs * dt * CFG.Anim.StrideRate end

	-- rotação (para inclinar o veículo nas curvas)
	local yaw = math.atan2(-look.X, -look.Z)
	if M.lastYaw then
		local d = (yaw - M.lastYaw + PI) % (2 * PI) - PI
		M.yawRate += (d / math.max(dt, 1 / 240) - M.yawRate) * damp(10, dt)
	end
	M.lastYaw = yaw
	M.accel += (((fwd - M.lastFwd) / math.max(dt, 1 / 240)) - M.accel) * damp(6, dt)
	M.lastFwd = fwd

	-- mira da cabeça = direção da câmera relativa ao corpo
	local cam = Workspace.CurrentCamera
	if cam then
		local rel = root.CFrame:VectorToObjectSpace(cam.CFrame.LookVector)
		local ty = math.clamp(math.atan2(-rel.X, -rel.Z), -1.1, 1.1)
		local tp = math.clamp(math.asin(math.clamp(rel.Y, -1, 1)), -0.5, 0.5)
		M.aimYaw += (ty - M.aimYaw) * damp(10, dt)
		M.aimPitch += (tp - M.aimPitch) * damp(10, dt)
	end

	-- rodas
	M.spin -= fwd / 1.8 * dt
	M.steer += (math.clamp(M.yawRate * 0.35, -0.5, 0.5) - M.steer) * damp(8, dt)
end

local function evaluate(dt, t)
	local tr = S.tr
	local mode = S.mode
	local u, kind, s, f0, f1 = 1, "Idle", 1, 0, 0
	if tr then
		u = clamp01((t - tr.start) / tr.dur)
		if tr.from == "Humanoid" then
			kind, s = "Summon", u
			f0 = (tr.to == "Vehicle") and 1 or 0
			f1 = f0
		elseif tr.to == "Humanoid" then
			kind, s = "Dismiss", 1 - u
			f0 = (tr.from == "Vehicle") and 1 or 0
			f1 = f0
		else
			kind, s = "Form", 1
			f0 = (tr.from == "Vehicle") and 1 or 0
			f1 = (tr.to == "Vehicle") and 1 or 0
		end
	else
		s = (mode == "Humanoid") and 0 or 1
		f0 = (mode == "Vehicle") and 1 or 0
		f1 = f0
	end

	local E = S.entries
	local n = #E
	local fx = S.fx

	-- Humanoide estável: tudo escondido, custo ~zero
	if not tr and mode == "Humanoid" then
		if not S.hiddenApplied then
			for i = 1, n do E[i].part.Transparency = 1; E[i].vt = 1 end
			setAvatarAlpha(1, false)
			S.hiddenApplied = true
		end
		return kind, u, s
	end

	-- eventos (sons/efeitos) quando a linha do tempo cruza um marco
	local evKey = (kind == "Form") and "F" or "S"
	local evVal = (kind == "Form") and u or s
	if S.evKey == evKey and S.evVal then
		local list = (kind == "Form") and TL.FormEvents or TL.Events
		for _, ev in ipairs(list) do
			if (S.evVal < ev[1]) ~= (evVal < ev[1]) then fireEvent(ev[2]) end
		end
	end
	S.evKey, S.evVal = evKey, evVal

	-- altura do corpo (HipHeight) acompanha a transformação
	if tr then
		local g
		if kind == "Form" then
			g = tr.gFrom + (tr.gTo - tr.gFrom) * smooth(u)
			if u > 0.5 and not tr.sizeSet then
				tr.sizeSet = true
				setRootSize(CFG[tr.to].RootSize)
			end
		else
			local a = smooth((s - TL.Raise0) / (TL.Raise1 - TL.Raise0))
			g = tr.gOrig + (tr.gMecha - tr.gOrig) * a
		end
		S.hum.HipHeight = math.max(0, g - S.root.Size.Y * 0.5)
	end

	-- pesos da animação procedural
	local aw = (kind == "Idle" or kind == "Form") and 1 or smooth((s - (TL.AsmEnd - 0.04)) / (1 - TL.AsmEnd + 0.04))
	local fG = (kind == "Form") and (f0 + (f1 - f0) * smooth(u)) or f0
	local robotW = aw * (1 - fG)
	local vehW = aw * fG

	updateMotion(dt, t)
	M.tick += 1
	local A = CFG.Anim

	-- passos
	local stepIdx = math.floor(M.phase / PI)
	if stepIdx ~= M.lastStep then
		M.lastStep = stepIdx
		if robotW > 0.5 and M.spd > 0.35 and M.grounded then onStep((stepIdx % 2 == 0) and 2 or 1) end
	end

	-- ângulos das juntas (modo robô)
	local bodyY, pitch, roll = 0, 0, 0
	if robotW > 0.001 then
		local ph = M.phase
		local sL, sR = math.sin(ph), -math.sin(ph)
		local cL, cR = math.cos(ph), -math.cos(ph)
		local spd = M.spd * robotW
		local air = M.air * robotW
		local breathe = math.sin(t * 1.4) * 0.02 * robotW
		local legA, kneeA, armA = A.LegSwing * spd, A.Knee * spd, A.ArmSwing * spd
		rx.LeftUpperLeg = legA * sL + air * 0.45
		rx.RightUpperLeg = legA * sR + air * 0.15
		rx.LeftLowerLeg = -kneeA * math.max(0, cL) - air * 0.8
		rx.RightLowerLeg = -kneeA * math.max(0, cR) - air * 0.5
		rx.LeftFoot = -(rx.LeftUpperLeg + rx.LeftLowerLeg) * 0.8
		rx.RightFoot = -(rx.RightUpperLeg + rx.RightLowerLeg) * 0.8
		rx.LeftUpperArm = -armA * sL + air * 0.7 + breathe * 3
		rx.RightUpperArm = -armA * sR + air * 0.7 - breathe * 3
		rx.LeftLowerArm = (0.25 + 0.35 * M.spd) * robotW
		rx.RightLowerArm = (0.25 + 0.35 * M.spd) * robotW
		rx.UpperTorso = breathe - A.Lean * spd
		ry.UpperTorso = math.sin(ph) * A.TorsoTwist * spd + M.aimYaw * 0.25 * robotW
		rz.UpperTorso = math.clamp(-M.yawRate * 0.012, -0.12, 0.12) * robotW
		ry.Head = M.aimYaw * 0.75 * robotW
		rx.Head = M.aimPitch * robotW
		bodyY = -math.abs(math.sin(ph)) * A.BobHeight * spd - math.clamp(M.spring, 0, 1.2) * 1.6 * robotW
	end
	if vehW > 0.001 then
		pitch = math.clamp(-M.accel * 0.0006, -0.05, 0.05) * vehW
		roll = math.clamp(-M.yawRate * math.abs(M.fwd) * 0.0012, -0.07, 0.07) * vehW
		bodyY += math.sin(t * 17) * 0.04 * clamp01(M.hs / 60) * vehW
	end
	local D0 = CFrame.new(0, bodyY, 0) * CFrame.Angles(pitch, 0, roll)

	local D = S.D
	local groundCF = CFrame.new(0, -groundDist(), 0)
	local visGlobal = (kind == "Idle" or kind == "Form") and 1 or smooth((s - 0.36) / 0.08)
	local orbitTh = s * CFG.Explode.OrbitTurns
	local snaps = 0

	for i = 1, n do
		local e = E[i]
		local pD = (e.pi > 0) and D[e.pi] or D0
		local role = e.role
		if role and robotW > 0.001 then
			local ax, ay, az = rx[role] or 0, ry[role] or 0, rz[role] or 0
			if ax ~= 0 or ay ~= 0 or az ~= 0 then
				local p = e.pivot
				D[i] = pD * CFrame.new(p) * CFrame.fromEulerAnglesYXZ(ax, ay, az) * CFrame.new(-p)
			else
				D[i] = pD
			end
		else
			D[i] = pD
		end

		-- pose base: robô <-> veículo (com "levantar" de painel no meio)
		local base
		if kind == "Form" then
			local x = clamp01((u - e.fs) / (e.fe - e.fs))
			base = e.robot:Lerp(e.veh, f0 + (f1 - f0) * easeInOut(x))
			local lift = math.sin(x * PI)
			if lift > 0 then base = base + e.lift * lift end
		else
			base = (f0 >= 0.5) and e.veh or e.robot
		end
		if e.wheel and vehW > 0.001 then
			if e.steer then base = CFrame.new(base.Position) * CFrame.Angles(0, M.steer * vehW, 0) * base.Rotation end
			base = base * CFrame.Angles(M.spin * vehW, 0, 0)
		end

		-- montagem: da pose "explodida" (orbitando) até o encaixe, com overshoot
		local P = base
		local vis = 1
		if s < 1 then
			local x = (s - e.as) / (e.ae - e.as)
			local xc = clamp01(x)
			local pa = easeOutBack(xc)
			local orb = 1 - smooth(xc)
			local ex = e.exploded
			if orb > 0 then
				ex = CFrame.Angles(0, orbitTh * orb, 0) * ex + Vector3.new(0, math.sin(s * 24 + i * 1.7) * 0.5 * orb, 0)
			end
			P = ex:Lerp(base, pa)
			vis = visGlobal
			if pa >= 0.92 and (e.prevPA or 0) < 0.92 and snaps < 3 and vis > 0.5 then
				snaps += 1
				fx.anchor.WorldPosition = (S.root.CFrame * (groundCF * (D[i] * P))).Position
				burst(fx.spark, 6)
			end
			e.prevPA = pa
		else
			e.prevPA = 1
		end

		local c0 = groundCF * (D[i] * P)
		if c0 ~= e.lastC0 then
			e.motor.C0 = c0
			e.lastC0 = c0
		end
		local tv = 1 - vis * (1 - e.baseT)
		if math.abs(tv - e.vt) > 0.004 then
			e.part.Transparency = tv
			e.vt = tv
		end
	end

	-- avatar se dissolve (só quando está montando/desmontando)
	local aA = 1 - smooth((s - TL.Ant) / (TL.DisEnd - TL.Ant))
	setAvatarAlpha(aA, (M.tick % 8) == 0)

	return kind, u, s, robotW, vehW
end

--------------------------------------------------------------------------------
-- CONTROLE: robô pesado / direção do veículo
--------------------------------------------------------------------------------
local controls
task.spawn(function()
	local ok, res = pcall(function()
		return require(player:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 15)):GetControls()
	end)
	if ok then controls = res end
end)

local function drive(dt)
	local hum, root = S.hum, S.root
	if S.tr then
		hum:Move(Vector3.zero, false)
		return
	end
	if S.mode == "Robot" then
		hum.AutoRotate = false
		local md = hum.MoveDirection
		if md.Magnitude > 0.1 then
			local look = root.CFrame.LookVector
			local cur = math.atan2(-look.X, -look.Z)
			local target = math.atan2(-md.X, -md.Z)
			local diff = (target - cur + PI) % (2 * PI) - PI
			local maxStep = CFG.Robot.TurnRate * dt
			local step = math.clamp(diff, -maxStep, maxStep)
			root.CFrame = CFrame.new(root.Position) * CFrame.Angles(0, cur + step, 0)
		end
	elseif S.mode == "Vehicle" then
		hum.AutoRotate = false
		local mv = controls and controls:GetMoveVector() or Vector3.zero
		local throttle, steer = -mv.Z, mv.X
		local V = CFG.Vehicle
		local target = throttle >= 0 and throttle or throttle * 0.5
		local rate = (target == 0 or target * M.vehSpeed < 0) and V.Brake or V.Accel
		M.vehSpeed += math.clamp(target - M.vehSpeed, -rate * dt, rate * dt)
		hum:Move(root.CFrame.LookVector * M.vehSpeed, false)
		local dir = M.vehSpeed < -0.02 and -1 or 1
		local turn = -steer * V.TurnRate * (0.25 + 0.75 * math.min(1, math.abs(M.vehSpeed))) * dir
		root.CFrame = root.CFrame * CFrame.Angles(0, turn * dt, 0)
		root.AssemblyAngularVelocity = Vector3.new(root.AssemblyAngularVelocity.X, 0, root.AssemblyAngularVelocity.Z)
	end
end

--------------------------------------------------------------------------------
-- LOOP POR FRAME
--------------------------------------------------------------------------------
local function frame(dt)
	if not S.ready then return end
	dt = math.min(dt, 0.1)
	local hum, root = S.hum, S.root
	if not hum.Parent or not root.Parent then return end
	local t = os.clock()

	-- fim de transição
	local tr = S.tr
	if tr and t - tr.start >= tr.dur then
		S.tr = nil
		S.mode = tr.to
		M.vehSpeed = 0
		applyStats(S.mode)
		if S.mode == "Humanoid" then setAvatarAlpha(1, false) end
	end

	-- energia do turbo (simulada localmente)
	if S.boosting then
		S.energy = math.max(0, S.energy - CFG.Boost.Drain * dt)
		if S.energy <= 0 then setBoost(false) end
	elseif S.energy < CFG.Boost.Max and os.clock() - S.lastBoostEnd > CFG.Boost.RegenDelay then
		S.energy = math.min(CFG.Boost.Max, S.energy + CFG.Boost.Regen * dt)
	end

	local kind, u, s, robotW, vehW = evaluate(dt, t)
	local fx = S.fx
	local tier = Q()
	local target = S.tr and S.tr.to or S.mode

	-- efeitos contínuos
	fx.boostSm = (fx.boostSm or 0) + ((S.boosting and 1 or 0) - (fx.boostSm or 0)) * damp(10, dt)
	local flamesOn = S.boosting and target ~= "Humanoid"
	for _, pair in ipairs(fx.flames) do
		local f, sm = pair[1], pair[2]
		if f.Enabled ~= flamesOn then
			f.Enabled = flamesOn
			sm.Enabled = flamesOn
		end
		if flamesOn then
			f.Rate = f:GetAttribute("BaseRate") * tier.part
			sm.Rate = sm:GetAttribute("BaseRate") * tier.part
		end
	end

	local moving = tier.trails and robotW and robotW > 0.5 and (M.spd > 0.6 or S.boosting)
	for _, tl in ipairs(fx.trails) do
		if tl.Enabled ~= (moving == true) then tl.Enabled = moving == true end
	end
	local boxesOn = tier.boxes and S.tr ~= nil
	for _, b in ipairs(fx.boxes) do
		if b.Visible ~= boxesOn then b.Visible = boxesOn end
	end

	-- poeira dos pneus
	if vehW and vehW > 0.5 and M.grounded and M.hs > 25 then
		M.dustTimer -= dt
		if M.dustTimer <= 0 then
			M.dustTimer = 0.12
			fx.anchor.WorldPosition = groundPos() + root.CFrame.LookVector * 5
			burst(fx.dust, 1)
		end
	end

	-- luzes
	fx.flashLevel = math.max(0, (fx.flashLevel or 0) - dt * 3)
	fx.flash.Brightness = tier.lights and fx.flashLevel * 10 or 0
	local reactorOn = tier.lights and target ~= "Humanoid"
	if fx.reactor.Enabled ~= reactorOn then fx.reactor.Enabled = reactorOn end
	if reactorOn then fx.reactor.Brightness = 1.2 + 0.6 * math.sin(t * 3) end

	-- anel de energia no chão
	if fx.ringT then
		fx.ringT += dt / 0.9
		if fx.ringT >= 1 then
			fx.ringT = nil
			fx.ring.Transparency = 1
		else
			local d = 2 + 34 * smooth(fx.ringT)
			fx.ring.Size = Vector3.new(0.3, d, d)
			fx.ring.Transparency = 0.15 + 0.85 * fx.ringT
			fx.ring.CFrame = CFrame.new(groundPos()) * CFrame.Angles(0, 0, PI / 2)
		end
	end

	-- câmera: alvo, blur durante a transformação
	camSetTargets(target, S.boosting)
	Cam.blurT = S.tr and math.sin(u * PI) ^ 1.5 or 0

	drive(dt)
	updateUI(t)
end

--------------------------------------------------------------------------------
-- SESSÃO DO PERSONAGEM (cria / destrói tudo por respawn)
--------------------------------------------------------------------------------
local function destroySession()
	S.ready = false
	for _, c in ipairs(S.conns) do c:Disconnect() end
	S.conns = {}
	if S.model then S.model:Destroy() end
	if S.fx then
		if S.fx.ring then S.fx.ring:Destroy() end
		if S.fx.anchor then S.fx.anchor:Destroy() end
		if S.fx.flash then S.fx.flash:Destroy() end
	end
	S.model, S.entries, S.fx, S.tr = nil, nil, nil, nil
	S.mode = "Humanoid"
	S.boosting = false
	S.energy = CFG.Boost.Max
	avatarParts = {}
	M.avatarAlpha = 1
	M.vehSpeed = 0
	camSetTargets("Humanoid", false)
end

local function setupCharacter(char)
	destroySession()
	local hum = char:WaitForChild("Humanoid", 15)
	local root = char:WaitForChild("HumanoidRootPart", 15)
	if not hum or not root then return end
	if hum.RigType ~= Enum.HumanoidRigType.R15 then
		warn("[Mecha] Este sistema exige avatar R15.")
		return
	end

	S.char, S.hum, S.root = char, hum, root
	S.orig = { walk = hum.WalkSpeed, useJP = hum.UseJumpPower, jp = hum.JumpPower, jh = hum.JumpHeight, hip = hum.HipHeight, size = root.Size }

	local model, entries, fx = buildMecha(char, root)
	buildFX(fx, root)
	S.model, S.entries, S.fx = model, entries, fx
	S.D = table.create(#entries, CFrame.identity)
	S.hiddenApplied = false

	-- peças do avatar (para dissolver): tudo que não é do mecha
	avatarParts = {}
	for _, d in ipairs(char:GetDescendants()) do
		if (d:IsA("BasePart") or d:IsA("Decal")) and not d:IsDescendantOf(model) then table.insert(avatarParts, d) end
	end
	table.insert(S.conns, char.DescendantAdded:Connect(function(d)
		if (d:IsA("BasePart") or d:IsA("Decal")) and not d:IsDescendantOf(model) then table.insert(avatarParts, d) end
	end))

	-- morreu como mecha: desmonta rápido
	table.insert(S.conns, hum.Died:Connect(function()
		setBoost(false)
		if S.mode ~= "Humanoid" and not S.tr then
			S.tr = { from = S.mode, to = "Humanoid", start = os.clock(), dur = CFG.Durations.Death, kind = "Dismiss", gMecha = groundDist(), gOrig = S.orig.hip + S.orig.size.Y * 0.5 }
			S.evKey = nil
			S.hiddenApplied = false
		end
	end))

	camSetTargets("Humanoid", false)
	S.ready = true
end

--------------------------------------------------------------------------------
-- ENTRADA: teclado / controle (botões do celular ficam na UI)
--------------------------------------------------------------------------------
local function actionHandler(name, state)
	if name == "MechaTransform" and state == Enum.UserInputState.Begin then
		Actions.transform()
	elseif name == "MechaDismiss" and state == Enum.UserInputState.Begin then
		Actions.dismiss()
	elseif name == "MechaBoost" then
		if state == Enum.UserInputState.Begin then Actions.boost(true)
		elseif state == Enum.UserInputState.End or state == Enum.UserInputState.Cancel then Actions.boost(false) end
	end
	return Enum.ContextActionResult.Sink
end
local prio = Enum.ContextActionPriority.High.Value
ContextActionService:BindActionAtPriority("MechaTransform", actionHandler, false, prio, CFG.Keys.Transform, CFG.Pad.Transform)
ContextActionService:BindActionAtPriority("MechaDismiss", actionHandler, false, prio, CFG.Keys.Dismiss, CFG.Pad.Dismiss)
ContextActionService:BindActionAtPriority("MechaBoost", actionHandler, false, prio, CFG.Keys.Boost, CFG.Pad.Boost)

--------------------------------------------------------------------------------
-- INICIALIZAÇÃO
--------------------------------------------------------------------------------
Cam.dMin, Cam.dMax = player.CameraMinZoomDistance, player.CameraMaxZoomDistance
Cam.zMin, Cam.zMax, Cam.zMinT, Cam.zMaxT = Cam.dMin, Cam.dMax, Cam.dMin, Cam.dMax

buildUI()
layoutUI()
local function watchViewport()
	local cam = Workspace.CurrentCamera
	if cam then cam:GetPropertyChangedSignal("ViewportSize"):Connect(layoutUI) end
end
watchViewport()
Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(function() watchViewport(); layoutUI() end)

RunService:BindToRenderStep("MechaCamFX", Enum.RenderPriority.Camera.Value + 1, camRender)
RunService:BindToRenderStep("MechaFrame", Enum.RenderPriority.Camera.Value + 2, frame)

player.CharacterAdded:Connect(setupCharacter)
if player.Character then task.spawn(setupCharacter, player.Character) end
