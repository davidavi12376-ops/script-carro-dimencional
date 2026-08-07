-- ========================================
-- SCRIPT DE TAMANHO DO CARRO (CORRIGIDO)
-- Key: dzin123
-- Agora dimensiona de verdade + meshes + luzes
-- ========================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local player = Players.LocalPlayer
local PlayerGui = player:WaitForChild("PlayerGui")

if PlayerGui:FindFirstChild("CarSizeMenu") then PlayerGui.CarSizeMenu:Destroy() end
if PlayerGui:FindFirstChild("CarSizeKey") then PlayerGui.CarSizeKey:Destroy() end

local CORRECT_KEY = "dzin123"

local settings = {
	Enabled = false,
	Speed = 1.5,
	MaxHeight = 1.8,
	MaxWidth = 1.6
}

local phase = 1
local progress = 0
local currentVehicle = nil
local originalData = {} -- [vehicle] = { [part] = {Size, MeshScale} }

local LIGHT_KEYWORDS = {
	"light", "farol", "lamp", "headlight", "taillight", "brake", "led",
	"luz", "lanterna", "sinal", "seta", "fog", "beam", "neon", "glow"
}

local function isLightName(name)
	name = string.lower(tostring(name or ""))
	for _, key in ipairs(LIGHT_KEYWORDS) do
		if string.find(name, key, 1, true) then
			return true
		end
	end
	return false
end

-- Acha o Model principal do carro
local function getVehicle()
	local char = player.Character
	if not char then return nil end

	local humanoid = char:FindFirstChildOfClass("Humanoid")
	if not humanoid then return nil end

	local seat = humanoid.SeatPart
	if not seat then return nil end

	-- Sobe até o Model do carro
	local model = seat:FindFirstAncestorOfClass("Model")
	if not model then
		return seat.Parent
	end

	-- Se o parent também for Model e tiver MAIS peças, sobe (carro completo)
	while model.Parent and model.Parent:IsA("Model") and model.Parent ~= workspace do
		local parent = model.Parent
		local parentCount = 0
		local currentCount = 0

		for _, o in ipairs(parent:GetDescendants()) do
			if o:IsA("BasePart") then parentCount = parentCount + 1 end
		end
		for _, o in ipairs(model:GetDescendants()) do
			if o:IsA("BasePart") then currentCount = currentCount + 1 end
		end

		if parentCount > currentCount then
			model = parent
		else
			break
		end
	end

	return model
end

local function getMesh(part)
	return part:FindFirstChildOfClass("SpecialMesh")
		or part:FindFirstChildOfClass("BlockMesh")
		or part:FindFirstChildOfClass("CylinderMesh")
end

local function getAllParts(vehicle)
	local parts = {}
	local added = {}

	local function add(part)
		if part and part:IsA("BasePart") and not added[part] then
			added[part] = true
			table.insert(parts, part)
		end
	end

	-- Tudo dentro do carro
	for _, obj in ipairs(vehicle:GetDescendants()) do
		if obj:IsA("BasePart") then
			add(obj)
		end
	end

	-- Pastas irmãs (luzes separadas)
	local parent = vehicle.Parent
	if parent and parent ~= workspace then
		for _, sibling in ipairs(parent:GetChildren()) do
			if sibling ~= vehicle then
				if isLightName(sibling.Name) then
					for _, obj in ipairs(sibling:GetDescendants()) do
						if obj:IsA("BasePart") then add(obj) end
					end
				else
					for _, obj in ipairs(sibling:GetDescendants()) do
						if obj:IsA("BasePart") and isLightName(obj.Name) then
							add(obj)
						end
					end
				end
			end
		end
	end

	return parts
end

local function saveOriginal(vehicle)
	originalData[vehicle] = {}

	for _, part in ipairs(getAllParts(vehicle)) do
		local data = {
			Size = part.Size,
			MeshScale = nil
		}
		local mesh = getMesh(part)
		if mesh then
			data.MeshScale = mesh.Scale
		end
		originalData[vehicle][part] = data
	end
end

local function applyScale(vehicle, heightMult, widthMult)
	local data = originalData[vehicle]
	if not data then return end

	for part, info in pairs(data) do
		if part and part.Parent then
			pcall(function()
				-- Size da peça
				part.Size = Vector3.new(
					info.Size.X * widthMult,
					info.Size.Y * heightMult,
					info.Size.Z * widthMult
				)

				-- Mesh (muitos carros usam isso pro visual)
				local mesh = getMesh(part)
				if mesh and info.MeshScale then
					mesh.Scale = Vector3.new(
						info.MeshScale.X * widthMult,
						info.MeshScale.Y * heightMult,
						info.MeshScale.Z * widthMult
					)
				end
			end)
		end
	end
end

local function resetScale(vehicle)
	local data = originalData[vehicle]
	if not data then return end

	for part, info in pairs(data) do
		if part and part.Parent then
			pcall(function()
				part.Size = info.Size
				local mesh = getMesh(part)
				if mesh and info.MeshScale then
					mesh.Scale = info.MeshScale
				end
			end)
		end
	end
end

local function startScript()
	RunService.RenderStepped:Connect(function(dt)
		local vehicle = getVehicle()

		if vehicle ~= currentVehicle then
			if currentVehicle then
				resetScale(currentVehicle)
			end
			currentVehicle = vehicle
			phase = 1
			progress = 0
			originalData[vehicle] = nil
			if vehicle then
				saveOriginal(vehicle)
			end
		end

		if not settings.Enabled or not vehicle then return end

		if not originalData[vehicle] then
			saveOriginal(vehicle)
		end

		progress = progress + settings.Speed * dt

		local heightMult = 1
		local widthMult = 1

		if phase == 1 then
			heightMult = 1 + (settings.MaxHeight - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 2 end
		elseif phase == 2 then
			heightMult = settings.MaxHeight - (settings.MaxHeight - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 3 end
		elseif phase == 3 then
			widthMult = 1 + (settings.MaxWidth - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 4 end
		elseif phase == 4 then
			widthMult = settings.MaxWidth - (settings.MaxWidth - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 1 end
		end

		applyScale(vehicle, heightMult, widthMult)
	end)
end

local function MakeDraggable(frame, handle)
	local dragging = false
	local dragStart, startPos

	handle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = frame.Position
			local conn
			conn = input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
					conn:Disconnect()
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

local function CreateMainMenu()
	local ScreenGui = Instance.new("ScreenGui")
	ScreenGui.Name = "CarSizeMenu"
	ScreenGui.ResetOnSpawn = false
	ScreenGui.Parent = PlayerGui

	local Main = Instance.new("Frame")
	Main.Size = UDim2.new(0, 260, 0, 300)
	Main.Position = UDim2.new(1, -280, 0.5, -150)
	Main.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
	Main.BorderSizePixel = 0
	Main.Active = true
	Main.Parent = ScreenGui

	local Corner = Instance.new("UICorner")
	Corner.CornerRadius = UDim.new(0, 6)
	Corner.Parent = Main

	local Stroke = Instance.new("UIStroke")
	Stroke.Color = Color3.fromRGB(140, 140, 155)
	Stroke.Thickness = 1.4
	Stroke.Parent = Main

	local TitleBar = Instance.new("Frame")
	TitleBar.Size = UDim2.new(1, 0, 0, 34)
	TitleBar.BackgroundColor3 = Color3.fromRGB(25, 25, 32)
	TitleBar.BorderSizePixel = 0
	TitleBar.Parent = Main

	local TitleCorner = Instance.new("UICorner")
	TitleCorner.CornerRadius = UDim.new(0, 6)
	TitleCorner.Parent = TitleBar

	local TitleFix = Instance.new("Frame")
	TitleFix.Size = UDim2.new(1, 0, 0, 10)
	TitleFix.Position = UDim2.new(0, 0, 1, -10)
	TitleFix.BackgroundColor3 = Color3.fromRGB(25, 25, 32)
	TitleFix.BorderSizePixel = 0
	TitleFix.Parent = TitleBar

	local Title = Instance.new("TextLabel")
	Title.Size = UDim2.new(1, -10, 1, 0)
	Title.Position = UDim2.new(0, 10, 0, 0)
	Title.BackgroundTransparency = 1
	Title.Text = "Tamanho do Carro"
	Title.TextColor3 = Color3.fromRGB(230, 230, 230)
	Title.TextSize = 14
	Title.Font = Enum.Font.GothamBold
	Title.TextXAlignment = Enum.TextXAlignment.Left
	Title.Parent = TitleBar

	MakeDraggable(Main, TitleBar)

	local StatusLabel = Instance.new("TextLabel")
	StatusLabel.Size = UDim2.new(1, -20, 0, 16)
	StatusLabel.Position = UDim2.new(0, 10, 0, 40)
	StatusLabel.BackgroundTransparency = 1
	StatusLabel.Text = "Entre em um carro..."
	StatusLabel.TextColor3 = Color3.fromRGB(160, 160, 170)
	StatusLabel.TextSize = 11
	StatusLabel.Font = Enum.Font.Gotham
	StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
	StatusLabel.Parent = Main

	-- Atualiza status
	task.spawn(function()
		while ScreenGui.Parent do
			local v = getVehicle()
			if v then
				local count = 0
				if originalData[v] then
					for _ in pairs(originalData[v]) do count = count + 1 end
				else
					count = #getAllParts(v)
				end
				StatusLabel.Text = "Carro: " .. v.Name .. " | Peças: " .. count
				StatusLabel.TextColor3 = Color3.fromRGB(100, 200, 120)
			else
				StatusLabel.Text = "Entre em um carro..."
				StatusLabel.TextColor3 = Color3.fromRGB(160, 160, 170)
			end
			task.wait(0.5)
		end
	end)

	local ToggleBtn = Instance.new("TextButton")
	ToggleBtn.Size = UDim2.new(1, -20, 0, 30)
	ToggleBtn.Position = UDim2.new(0, 10, 0, 60)
	ToggleBtn.BackgroundColor3 = Color3.fromRGB(140, 40, 40)
	ToggleBtn.Text = "DESATIVADO"
	ToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	ToggleBtn.TextSize = 13
	ToggleBtn.Font = Enum.Font.GothamMedium
	ToggleBtn.Parent = Main

	local ToggleCorner = Instance.new("UICorner")
	ToggleCorner.CornerRadius = UDim.new(0, 4)
	ToggleCorner.Parent = ToggleBtn

	ToggleBtn.MouseButton1Click:Connect(function()
		settings.Enabled = not settings.Enabled
		if settings.Enabled then
			ToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 130, 90)
			ToggleBtn.Text = "ATIVADO"
			if currentVehicle then
				originalData[currentVehicle] = nil
				saveOriginal(currentVehicle)
			end
		else
			ToggleBtn.BackgroundColor3 = Color3.fromRGB(140, 40, 40)
			ToggleBtn.Text = "DESATIVADO"
			if currentVehicle then
				resetScale(currentVehicle)
			end
			phase = 1
			progress = 0
		end
	end)

	local function createControl(name, yPos, getValue, setValue, min, max, step)
		local label = Instance.new("TextLabel")
		label.Size = UDim2.new(1, -20, 0, 18)
		label.Position = UDim2.new(0, 10, 0, yPos)
		label.BackgroundTransparency = 1
		label.Text = name .. ": " .. string.format("%.1f", getValue())
		label.TextColor3 = Color3.fromRGB(200, 200, 210)
		label.TextSize = 13
		label.Font = Enum.Font.Gotham
		label.TextXAlignment = Enum.TextXAlignment.Left
		label.Parent = Main

		local minus = Instance.new("TextButton")
		minus.Size = UDim2.new(0, 36, 0, 26)
		minus.Position = UDim2.new(0, 10, 0, yPos + 22)
		minus.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
		minus.Text = "-"
		minus.TextColor3 = Color3.fromRGB(255, 255, 255)
		minus.TextSize = 18
		minus.Font = Enum.Font.GothamBold
		minus.Parent = Main

		local plus = Instance.new("TextButton")
		plus.Size = UDim2.new(0, 36, 0, 26)
		plus.Position = UDim2.new(0, 52, 0, yPos + 22)
		plus.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
		plus.Text = "+"
		plus.TextColor3 = Color3.fromRGB(255, 255, 255)
		plus.TextSize = 18
		plus.Font = Enum.Font.GothamBold
		plus.Parent = Main

		local c1 = Instance.new("UICorner")
		c1.CornerRadius = UDim.new(0, 4)
		c1.Parent = minus
		local c2 = Instance.new("UICorner")
		c2.CornerRadius = UDim.new(0, 4)
		c2.Parent = plus

		minus.MouseButton1Click:Connect(function()
			local v = math.max(min, getValue() - step)
			setValue(v)
			label.Text = name .. ": " .. string.format("%.1f", v)
		end)

		plus.MouseButton1Click:Connect(function()
			local v = math.min(max, getValue() + step)
			setValue(v)
			label.Text = name .. ": " .. string.format("%.1f", v)
		end)
	end

	createControl("Velocidade", 100, function() return settings.Speed end, function(v) settings.Speed = v end, 0.3, 13, 0.2)
	createControl("Cresce pra cima", 160, function() return settings.MaxHeight end, function(v) settings.MaxHeight = v end, 1.1, 13, 0.1)
	createControl("Cresce pros lados", 220, function() return settings.MaxWidth end, function(v) settings.MaxWidth = v end, 1.1, 13, 0.1)
end

-- ==================== KEY ====================
local KeyGui = Instance.new("ScreenGui")
KeyGui.Name = "CarSizeKey"
KeyGui.ResetOnSpawn = false
KeyGui.Parent = PlayerGui

local KeyFrame = Instance.new("Frame")
KeyFrame.Size = UDim2.new(0, 300, 0, 160)
KeyFrame.Position = UDim2.new(0.5, -150, 0.5, -80)
KeyFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
KeyFrame.BorderSizePixel = 0
KeyFrame.Parent = KeyGui

local KeyCorner = Instance.new("UICorner")
KeyCorner.CornerRadius = UDim.new(0, 6)
KeyCorner.Parent = KeyFrame

local KeyStroke = Instance.new("UIStroke")
KeyStroke.Color = Color3.fromRGB(140, 140, 155)
KeyStroke.Thickness = 1.4
KeyStroke.Parent = KeyFrame

local KeyTitle = Instance.new("TextLabel")
KeyTitle.Size = UDim2.new(1, -20, 0, 30)
KeyTitle.Position = UDim2.new(0, 10, 0, 12)
KeyTitle.BackgroundTransparency = 1
KeyTitle.Text = "Digite a Key"
KeyTitle.TextColor3 = Color3.fromRGB(230, 230, 230)
KeyTitle.TextSize = 16
KeyTitle.Font = Enum.Font.GothamBold
KeyTitle.Parent = KeyFrame

local KeyBox = Instance.new("TextBox")
KeyBox.Size = UDim2.new(1, -20, 0, 34)
KeyBox.Position = UDim2.new(0, 10, 0, 50)
KeyBox.BackgroundColor3 = Color3.fromRGB(32, 32, 40)
KeyBox.BorderSizePixel = 0
KeyBox.Text = ""
KeyBox.PlaceholderText = "Key..."
KeyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
KeyBox.PlaceholderColor3 = Color3.fromRGB(120, 120, 130)
KeyBox.TextSize = 14
KeyBox.Font = Enum.Font.Gotham
KeyBox.ClearTextOnFocus = false
KeyBox.Parent = KeyFrame

local KeyBoxCorner = Instance.new("UICorner")
KeyBoxCorner.CornerRadius = UDim.new(0, 4)
KeyBoxCorner.Parent = KeyBox

local KeyBoxPadding = Instance.new("UIPadding")
KeyBoxPadding.PaddingLeft = UDim.new(0, 10)
KeyBoxPadding.Parent = KeyBox

local ErrorLabel = Instance.new("TextLabel")
ErrorLabel.Size = UDim2.new(1, -20, 0, 18)
ErrorLabel.Position = UDim2.new(0, 10, 0, 90)
ErrorLabel.BackgroundTransparency = 1
ErrorLabel.Text = ""
ErrorLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
ErrorLabel.TextSize = 12
ErrorLabel.Font = Enum.Font.Gotham
ErrorLabel.Parent = KeyFrame

local EnterBtn = Instance.new("TextButton")
EnterBtn.Size = UDim2.new(1, -20, 0, 32)
EnterBtn.Position = UDim2.new(0, 10, 0, 115)
EnterBtn.BackgroundColor3 = Color3.fromRGB(0, 130, 90)
EnterBtn.Text = "Entrar"
EnterBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
EnterBtn.TextSize = 14
EnterBtn.Font = Enum.Font.GothamMedium
EnterBtn.Parent = KeyFrame

local EnterCorner = Instance.new("UICorner")
EnterCorner.CornerRadius = UDim.new(0, 4)
EnterCorner.Parent = EnterBtn

local function tryKey()
	if KeyBox.Text == CORRECT_KEY then
		KeyGui:Destroy()
		CreateMainMenu()
		startScript()
	else
		ErrorLabel.Text = "Key incorreta!"
		KeyBox.Text = ""
	end
end

EnterBtn.MouseButton1Click:Connect(tryKey)
KeyBox.FocusLost:Connect(function(enter)
	if enter then tryKey() end
end)

task.wait(0.1)
KeyBox:CaptureFocus()
