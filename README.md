-- ========================================
-- CARRO DANÇANDO (trend Yaris)
-- Só pasta Body | sequência cima → lados
-- Key: dzin123
-- ========================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local player = Players.LocalPlayer
local PlayerGui = player:WaitForChild("PlayerGui")

pcall(function()
	if PlayerGui:FindFirstChild("CarDanceMenu") then PlayerGui.CarDanceMenu:Destroy() end
	if PlayerGui:FindFirstChild("CarDanceKey") then PlayerGui.CarDanceKey:Destroy() end
end)

_G.CarDanceToken = (_G.CarDanceToken or 0) + 1
local myToken = _G.CarDanceToken

if _G.CarDanceConn then
	pcall(function() _G.CarDanceConn:Disconnect() end)
	_G.CarDanceConn = nil
end

local CORRECT_KEY = "dzin123"

local settings = {
	Enabled = false,
	Speed = 2.0,
	MaxHeight = 1.6,
	MaxWidth = 1.5
}

local phase = 1
local progress = 0
local currentVehicle = nil
local bodyParts = {}

local function alive()
	return _G.CarDanceToken == myToken
end

local function getVehicle()
	local char = player.Character
	if not char then return nil end
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hum or not hum.SeatPart then return nil end

	local seat = hum.SeatPart
	local model = seat:FindFirstAncestorOfClass("Model")
	if not model then return seat.Parent end

	while model.Parent and model.Parent:IsA("Model") and model.Parent ~= workspace do
		local parent = model.Parent
		local p, c = 0, 0
		for _, o in ipairs(parent:GetDescendants()) do
			if o:IsA("BasePart") then p = p + 1 end
		end
		for _, o in ipairs(model:GetDescendants()) do
			if o:IsA("BasePart") then c = c + 1 end
		end
		if p > c then model = parent else break end
	end
	return model
end

local function getRoot(vehicle)
	return vehicle.PrimaryPart
		or vehicle:FindFirstChild("DriveSeat")
		or vehicle:FindFirstChildWhichIsA("VehicleSeat")
		or vehicle:FindFirstChildWhichIsA("BasePart")
end

local function getMesh(part)
	return part:FindFirstChildOfClass("SpecialMesh")
		or part:FindFirstChildOfClass("BlockMesh")
		or part:FindFirstChildOfClass("CylinderMesh")
end

local function ensureOriginal(part)
	if not part:GetAttribute("CD_OrigSize") then
		part:SetAttribute("CD_OrigSize", part.Size)
	end
	local mesh = getMesh(part)
	if mesh and not mesh:GetAttribute("CD_OrigScale") then
		mesh:SetAttribute("CD_OrigScale", mesh.Scale)
	end
end

local function getOrigSize(part)
	local v = part:GetAttribute("CD_OrigSize")
	if typeof(v) == "Vector3" then return v end
	return part.Size
end

local function getOrigMesh(mesh)
	local v = mesh:GetAttribute("CD_OrigScale")
	if typeof(v) == "Vector3" then return v end
	return mesh.Scale
end

local function collectBodyFolderParts(vehicle)
	local list, seen = {}, {}
	local function add(part)
		if part and part:IsA("BasePart") and not seen[part] then
			seen[part] = true
			ensureOriginal(part)
			table.insert(list, part)
		end
	end

	local function findBody(parent)
		for _, child in ipairs(parent:GetChildren()) do
			if string.lower(child.Name) == "body" then
				for _, o in ipairs(child:GetDescendants()) do
					if o:IsA("BasePart") then add(o) end
				end
				if child:IsA("BasePart") then add(child) end
			else
				findBody(child)
			end
		end
	end

	findBody(vehicle)

	local parent = vehicle.Parent
	if parent and parent ~= workspace then
		for _, sibling in ipairs(parent:GetChildren()) do
			if sibling ~= vehicle and string.lower(sibling.Name) == "body" then
				for _, o in ipairs(sibling:GetDescendants()) do
					if o:IsA("BasePart") then add(o) end
				end
			end
		end
	end
	return list
end

local function applyScale(h, w)
	for _, part in ipairs(bodyParts) do
		if part and part.Parent then
			pcall(function()
				local orig = getOrigSize(part)
				part.Size = Vector3.new(orig.X * w, orig.Y * h, orig.Z * w)
				local mesh = getMesh(part)
				if mesh then
					local mo = getOrigMesh(mesh)
					mesh.Scale = Vector3.new(mo.X * w, mo.Y * h, mo.Z * w)
				end
			end)
		end
	end
end

local function resetAll()
	for _, part in ipairs(bodyParts) do
		if part and part.Parent then
			pcall(function()
				part.Size = getOrigSize(part)
				local mesh = getMesh(part)
				if mesh then
					mesh.Scale = getOrigMesh(mesh)
				end
			end)
		end
	end
end

local function startLoop()
	if _G.CarDanceConn then
		pcall(function() _G.CarDanceConn:Disconnect() end)
		_G.CarDanceConn = nil
	end

	_G.CarDanceConn = RunService.Heartbeat:Connect(function(dt)
		if not alive() then return end
		if not settings.Enabled then return end

		local vehicle = getVehicle()

		if vehicle ~= currentVehicle then
			if currentVehicle then resetAll() end
			currentVehicle = vehicle
			phase = 1
			progress = 0
			bodyParts = {}
			if vehicle then
				bodyParts = collectBodyFolderParts(vehicle)
			end
		end

		if not vehicle then return end
		if #bodyParts == 0 then
			bodyParts = collectBodyFolderParts(vehicle)
		end

		local root = getRoot(vehicle)
		local linVel, angVel
		if root then
			linVel = root.AssemblyLinearVelocity
			angVel = root.AssemblyAngularVelocity
		end

		progress = progress + settings.Speed * dt

		local h, w = 1, 1
		-- Dança: sobe → volta → lados → volta
		if phase == 1 then
			h = 1 + (settings.MaxHeight - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 2 end
		elseif phase == 2 then
			h = settings.MaxHeight - (settings.MaxHeight - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 3 end
		elseif phase == 3 then
			w = 1 + (settings.MaxWidth - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 4 end
		elseif phase == 4 then
			w = settings.MaxWidth - (settings.MaxWidth - 1) * math.min(progress, 1)
			if progress >= 1 then progress = 0 phase = 1 end
		end

		applyScale(h, w)

		if root and linVel and angVel then
			pcall(function()
				root.AssemblyLinearVelocity = linVel
				root.AssemblyAngularVelocity = angVel
			end)
		end
	end)
end

local function MakeDraggable(frame, handle)
	local dragging, dragStart, startPos = false, nil, nil
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
	ScreenGui.Name = "CarDanceMenu"
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
	Title.Text = "Carro Dançando"
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

	task.spawn(function()
		while ScreenGui.Parent and alive() do
			local v = getVehicle()
			if v then
				local n = #bodyParts
				if n == 0 then n = #collectBodyFolderParts(v) end
				StatusLabel.Text = "Carro: " .. v.Name .. " | Body: " .. n
				StatusLabel.TextColor3 = n > 0 and Color3.fromRGB(100, 200, 120) or Color3.fromRGB(255, 150, 80)
			else
				StatusLabel.Text = "Entre em um carro..."
				StatusLabel.TextColor3 = Color3.fromRGB(160, 160, 170)
			end
			task.wait(0.4)
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
		if not alive() then return end

		if settings.Enabled then
			settings.Enabled = false
			ToggleBtn.BackgroundColor3 = Color3.fromRGB(140, 40, 40)
			ToggleBtn.Text = "DESATIVADO"
			resetAll()
			phase = 1
			progress = 0
		else
			settings.Enabled = true
			ToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 130, 90)
			ToggleBtn.Text = "DANÇANDO"
			local v = getVehicle()
			if v then
				currentVehicle = v
				bodyParts = collectBodyFolderParts(v)
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

	createControl("Velocidade da dança", 100, function() return settings.Speed end, function(v) settings.Speed = v end, 0.3, 13, 0.2)
	createControl("Estica pra cima", 160, function() return settings.MaxHeight end, function(v) settings.MaxHeight = v end, 1.1, 13, 0.1)
	createControl("Expande pro lado", 220, function() return settings.MaxWidth end, function(v) settings.MaxWidth = v end, 1.1, 13, 0.1)
end

-- ==================== KEY ====================
local KeyGui = Instance.new("ScreenGui")
KeyGui.Name = "CarDanceKey"
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
		startLoop()
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
