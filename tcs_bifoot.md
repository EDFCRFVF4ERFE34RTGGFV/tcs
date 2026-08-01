--[[

Aipim melhor do mercado!
]]

--// Loaded Check

if getgenv().gamesense and getgenv().gamesense.loaded then
	warn("GS ALREADY LOADED GG")
	return
end
getgenv().gamesense = {loaded = true}

--///// Services
local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")
local TextChatService = game:GetService("TextChatService")
local HTTPService = game:GetService("HttpService")

--///// UI Library
local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()
local AnimSocket =  loadstring(game:HttpGet("https://raw.github.com/0zBug/AnimSocket/main/main.lua"))()

local Options = Library.Options
local Toggles = Library.Toggles

Library.ForceCheckbox = false
Library.ShowToggleFrameInKeybinds = true

local Window = Library:CreateWindow({
	Title = "gamesense.mps",
	Footer = "VEF Version",
	Icon = 70680258649133,
	NotifySide = "Right",
	ShowCustomCursor = true,
})

local Tabs = {}

--///// Variables
--// Tables
local Dump = {
	BallPrediction = {}
}

--// Miscs
--// Instances
local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character
local Humanoid = Character and Character:FindFirstChild("Humanoid")
local HRP = Character and Character:FindFirstChild("HumanoidRootPart")

local ChangeOwner = ReplicatedStorage:FindFirstChild("ChangeOwner")
local ChangeValue = ReplicatedStorage:FindFirstChild("ChangeValue")
local CatchBall = ReplicatedStorage:FindFirstChild("CatchBall")
local ReactEvent = ReplicatedStorage:FindFirstChild("React")
local DropBall = ReplicatedStorage:FindFirstChild("DropBall")
local ClientRemote = ReplicatedStorage:FindFirstChild("Client")

local MainModule = nil
local BALL_NAME = "TPS"
local _BALLS = {}

local function RefreshBalls()
	local newBalls = {}
	for _, v in workspace:GetChildren() do
		if v:IsA("Part") and v.Name == BALL_NAME then
			table.insert(newBalls, v)
		end
	end
	local stadium = workspace:FindFirstChild("WorkspaceStadiumFolder")
	if stadium then
		for _, v in stadium:GetChildren() do
			if v:IsA("Part") and v.Name == BALL_NAME then
				table.insert(newBalls, v)
			end
		end
	end
	_BALLS = newBalls
end
RefreshBalls()

local StarterPack = game:GetService("StarterPack")
local ToolManagementModule = StarterPack:FindFirstChild("ToolManagement")
if ToolManagementModule then
	MainModule = require(ToolManagementModule)
end


local ReachBox = Instance.new("Part")
ReachBox.Name = tostring(math.random(100000, 999999))
ReachBox.Size = Vector3.new(0, 0, 0)
ReachBox.CFrame = CFrame.new(math.huge, math.huge, math.huge)
ReachBox.Anchored = true
ReachBox.CanCollide = false
ReachBox.CanTouch = true
ReachBox.CanQuery = false
ReachBox.Massless = true
ReachBox.Transparency = 0.9
ReachBox.Material = Enum.Material.SmoothPlastic
ReachBox.CastShadow = false
ReachBox.Parent = workspace

local GS_GUI = Instance.new("ScreenGui")
GS_GUI.Name = tostring(math.random(100000, 999999))
GS_GUI.IgnoreGuiInset = true
GS_GUI.Parent = game:GetService("CoreGui")
local Dumbass = Instance.new("ImageLabel")
Dumbass.Name = "Dumbass"
Dumbass.AnchorPoint = Vector2.new(1, 1)
Dumbass.BackgroundColor3 = Color3.new(0, 0, 0)
Dumbass.BackgroundTransparency = 1
Dumbass.Position = UDim2.new(1, 25, 1, 15)
Dumbass.Size = UDim2.new(0, 150, 0, 300)
Dumbass.Image = "rbxassetid://95334651149795"
Dumbass.Visible = false
Dumbass.Parent = GS_GUI

--// Utility Functions
local function quadraticSolver(a, b, c)
	local x1 = (-b + math.sqrt((b*b) -4 * a * c)) / (2 * a)
	local x2 = (-b - math.sqrt((b*b) -4 * a * c)) / (2 * a)
	if x2 > x1 then
		return x2
	else
		return x1
	end
end
local function findTimeAtHeight(a, vel, h, startingH)
	local x = h - startingH
	local x1 = (math.sqrt((vel * vel) + 2 * a * x ) - vel) / a
	local x2 = -(math.sqrt((vel * vel) + 2 * a * x) + vel) / a
	if x2 > x1 then
		return x2
	else
		return x1
	end
end
local function findHeightAtTime(vel, t, acc)
	return vel.Y * t + 0.5 * -acc * (t*t)
end
local function findPositionAtTime(vel, t, startingPos, acc)
	local height = findHeightAtTime(vel, t, acc)
	return startingPos + Vector3.new(vel.X * t, height, vel.Z * t)
end

--///// Ball Functions
local function findLandingPosition(Vo, startingPosition, acc, max)
	local seconds = quadraticSolver((0.5 * -acc), Vo.Y, startingPosition.Y)
	local lastPosition = startingPosition
	local predParams = RaycastParams.new()
	predParams.FilterType = Enum.RaycastFilterType.Exclude
	predParams.FilterDescendantsInstances = {_BALLS}
	for i = 1, max do
		local t = seconds * (1/max * i)
		local nextPosition = findPositionAtTime(Vo, t, startingPosition, acc)
		local result = workspace:Raycast(lastPosition, (nextPosition - lastPosition))
		if result then
			local baseHeight = result.Position.Y
			local timeAtHeight = findTimeAtHeight(-acc, Vo.Y, baseHeight, startingPosition.Y)
			local offset = findPositionAtTime(Vo, timeAtHeight, startingPosition, acc)
			return offset
		end
		lastPosition = nextPosition
	end
	local horizontalVel = Vector3.new(Vo.X, 0, Vo.Z)
	local endingOffset = horizontalVel * seconds
	return startingPosition + endingOffset + Vector3.new(0, findHeightAtTime(Vo, seconds, acc), 0)
end

local function ClearDump(DumpType)
	for ID, DumpInstance in pairs(Dump[DumpType]) do
		if DumpInstance:IsA("Instance") then
			DumpInstance:Destroy()
		end
	end
	Dump[DumpType] = {}
end

local function GetPlayerFromString(String)
	for ID, TargetPlayer in pairs(Players:GetPlayers()) do
		if TargetPlayer.Name:sub(1, #String):lower() == String:lower() then
			return TargetPlayer
		end
	end
end

local function PredictBall(Ball)
	local LandingPosition = findLandingPosition(Ball.AssemblyLinearVelocity, Ball.Position, workspace.Gravity, Options.BallPredAccuracy.Value)
	local BallPredAtt0 = Instance.new("Attachment")
	BallPredAtt0.Name = tostring(math.random(100000,999999))
	BallPredAtt0.WorldPosition = Ball.Position
	BallPredAtt0.Parent = workspace.Terrain
	table.insert(Dump.BallPrediction, BallPredAtt0)
	local BallPredAtt1 = Instance.new("Attachment")
	BallPredAtt1.Name = tostring(math.random(100000,999999))
	BallPredAtt1.WorldPosition = LandingPosition
	BallPredAtt1.Parent = workspace.Terrain
	table.insert(Dump.BallPrediction, BallPredAtt1)
	local BallPredBeam = Instance.new("Beam")
	BallPredBeam.Name = tostring(math.random(100000,999999))
	BallPredBeam.Color = ColorSequence.new(Options.BallPredTrailColor.Value)
	BallPredBeam.LightEmission = 0
	BallPredBeam.LightInfluence = 0
	BallPredBeam.Brightness = 1
	BallPredBeam.Transparency = NumberSequence.new(Options.BallPredTrailColor.Transparency)
	BallPredBeam.Attachment0 = BallPredAtt0
	BallPredBeam.Attachment1 = BallPredAtt1
	BallPredBeam.FaceCamera = true
	BallPredBeam.Width0 = Options.BallPredTrailSize.Value
	BallPredBeam.Width1 = Options.BallPredTrailSize.Value
	BallPredBeam.Parent = Ball
	table.insert(Dump.BallPrediction, BallPredBeam)
end

Tabs.Reach = Window:AddTab("Reach", "footprints")
Tabs.Ball = Window:AddTab("Ball", "volleyball")
Tabs.Character = Window:AddTab("Character", "volleyball")
Tabs.Visuals = Window:AddTab("Visuals", "eye")
Tabs.Miscs = Window:AddTab("Miscs", "ellipsis")
Tabs.Config = Window:AddTab("UI Settings", "settings")
--// Reach Tab
local ReachMainGroupbox = Tabs.Reach:AddLeftGroupbox("Main")
local ReachTabbox = Tabs.Reach:AddRightTabbox()
local ReachMainTab
local ReachShootTab
local ReachPassTab
local ReachLongTab
local ReachTackleTab
local ReachDribbleTab
local ReachSaveTab
if UserInputService.TouchEnabled then
	ReachMainTab = ReachTabbox:AddTab('Main')
else
	ReachShootTab = ReachTabbox:AddTab('Shoot')
	ReachPassTab = ReachTabbox:AddTab('Pass')
	ReachLongTab = ReachTabbox:AddTab('Long')
	ReachTackleTab = ReachTabbox:AddTab('Tackle')
	ReachDribbleTab = ReachTabbox:AddTab('Dribble')
	ReachSaveTab = ReachTabbox:AddTab('Save')
end

ReachMainGroupbox:AddToggle("ReachMasterToggle", {
	Text = "Enabled",
	Tooltip = "Master toggle for reach.",
})
ReachMainGroupbox:AddToggle("ReachVisualizerToggle", {
	Text = "Visualizer",
	Tooltip = "Visualizer of the reach box.",
}):AddColorPicker("ReachVisualizerColor", {
	Default = Color3.new(1, 1, 1),
	Title = "Visualizer Color.",
	Transparency = 0.9,
})

local function CreateReachTab(TabsElement, ReachType)
	TabsElement:AddToggle("Reach"..ReachType.."Toggle", {
		Text = "Enabled",
		Tooltip = "Toggle for "..string.lower(ReachType).." reach.",
	})
	TabsElement:AddToggle("InfiniteReach"..ReachType.."Toggle", {
		Text = "Infinite Reach",
		Risky = true,
		Tooltip = "Makes the "..string.lower(ReachType).." reach infinite.",
	})
	TabsElement:AddToggle("Reach"..ReachType.."CompToggle", {
		Text = "Comp Reach",
		Tooltip = "Reaches only if you dont have network ownership.",
	})
	TabsElement:AddSlider("Reach"..ReachType.."SizeX", {
		Text = "Size X",
		Default = 0,
		Min = 0,
		Max = 25,
		Rounding = 1,
		Tooltip = "Sets the "..string.lower(ReachType).." reach X size.",
	})
	TabsElement:AddSlider("Reach"..ReachType.."SizeY", {
		Text = "Size Y",
		Default = 0,
		Min = 0,
		Max = 25,
		Rounding = 1,
		Tooltip = "Sets the "..string.lower(ReachType).." reach Y size.",
	})
	TabsElement:AddSlider("Reach"..ReachType.."SizeZ", {
		Text = "Size Z",
		Default = 0,
		Min = 0,
		Max = 25,
		Rounding = 1,
		Tooltip = "Sets the "..string.lower(ReachType).." reach Z size.",
	})
	TabsElement:AddSlider("Reach"..ReachType.."OffsetX", {
		Text = "Offset X",
		Default = 0,
		Min = -10,
		Max = 10,
		Rounding = 1,
		Tooltip = "Sets the "..string.lower(ReachType).." reach X offset.",
	})
	TabsElement:AddSlider("Reach"..ReachType.."OffsetY", {
		Text = "Offset Y",
		Default = 0,
		Min = -10,
		Max = 10,
		Rounding = 1,
		Tooltip = "Sets the "..string.lower(ReachType).." reach Y offset.",
	})
	TabsElement:AddSlider("Reach"..ReachType.."OffsetZ", {
		Text = "Offset Z",
		Default = 0,
		Min = -10,
		Max = 10,
		Rounding = 1,
		Tooltip = "Sets the "..string.lower(ReachType).." reach Z offset.",
	})
	TabsElement:AddDropdown("Reach"..ReachType.."BallSelector", {
		Values = {"Closest to character", "Furthest to character"},
		Default = "Closest to character",
		Text = "Ball selection",
		Tooltip = "Choose which ball should be the prority for reach.",
	})
end

if UserInputService.TouchEnabled then
	CreateReachTab(ReachMainTab, "Main")
else
	CreateReachTab(ReachShootTab, "Shoot")
	CreateReachTab(ReachPassTab, "Pass")
	CreateReachTab(ReachLongTab, "Long")
	CreateReachTab(ReachTackleTab, "Tackle")
	CreateReachTab(ReachDribbleTab, "Dribble")
	CreateReachTab(ReachSaveTab, "Save")
end

--// Ball Tab
local BallPredictionGroupbox = Tabs.Ball:AddLeftGroupbox("Main")
local BallMacrosGroupbox = Tabs.Ball:AddRightGroupbox("Macros")

BallPredictionGroupbox:AddToggle("BallPredToggle", {
	Text = "Ball Prediction",
	Tooltip = "Predicts where the ball will land.",
	Default = false,
	Visible = true,
}):AddColorPicker("BallPredTrailColor", {
	Default = Color3.new(1, 1, 1),
	Title = " Color",
	Transparency = 0,
})
BallPredictionGroupbox:AddSlider("BallPredTrailSize", {
	Text = "Trail Size",
	Default = 0.1,
	Min = 0.01,
	Max = 0.5,
	Rounding = 2,
	Tooltip = "Changes the trail thickness.",
})
BallPredictionGroupbox:AddSlider("BallPredHZ", {
	Text = "Refresh Rate",
	Default = 0.1,
	Min = 0,
	Max = 1,
	Rounding = 2,
	Tooltip = "Changes how often the ball prediction will refresh.",
})
BallPredictionGroupbox:AddSlider("BallPredThreshold", {
	Text = "Prediction Threshold",
	Default = 25,
	Min = 0,
	Max = 100,
	Rounding = 0,
	Tooltip = "Sets the minimal ball velocity to show the prediction trail.",
})
BallPredictionGroupbox:AddSlider("BallPredAccuracy", {
	Text = "Prediction Accuracy",
	Default = 5,
	Min = 1,
	Max = 10,
	Rounding = 0,
	Tooltip = "Sets how many raycast will be fired per prediction refresh to ensure accracy.",
})

BallMacrosGroupbox:AddToggle("HomboloMacroToggle", {
	Text = "Hombolo Macro",
	Tooltip = "Makes the ball always be over your head",
	Default = false,
	Visible = true,
}):AddKeyPicker("HomboloMacroKeybind", {
	Default = "P",
	SyncToggleState = true,
	Mode = "Toggle",
	Text = "Hombolo Macro",
	NoUI = false,
})

BallMacrosGroupbox:AddToggle("ReactKillerToggle", {
	Text = "React Killer",
	Tooltip = "Prevents other players from reacting to the ball. Only you can react.",
	Default = false,
	Visible = true,
}):AddKeyPicker("ReactKillerKeybind", {
	Default = "K",
	SyncToggleState = true,
	Mode = "Toggle",
	Text = "React Killer",
	NoUI = false,
})

BallMacrosGroupbox:AddSlider("ReactKillerPolling", {
	Text = "Polling Time",
	Default = 0.005,
	Min = 0.001,
	Max = 1,
	Rounding = 3,
	Tooltip = "Interval in seconds between ownership claims. Lower = faster but more server spam.",
})

--// Character Tab
local CharHumGroupbox = Tabs.Character:AddLeftGroupbox("Humanoid")
CharHumGroupbox:AddToggle("CharSpeedToggle", {
	Text = "Speed",
	Tooltip = "Changes how fast the character moves. (CFrame based)",
	Default = false,
	Visible = true,
})
CharHumGroupbox:AddSlider("CharSpeed", {
	Text = "Speed Value",
	Default = 0,
	Min = 0,
	Max = 10,
	Rounding = 1,
	Tooltip = "Sets the speed value.",
})

if not UserInputService.TouchEnabled then
	local CharCustomMovesGroupbox = Tabs.Character:AddRightGroupbox("Custom Moves")
	CharCustomMovesGroupbox:AddToggle("InsanePowerShoot", {
		Text = "Insane Power Shoot",
		Tooltip = "Keybind: G",
		Default = false,
		Visible = true,
	})
	CharCustomMovesGroupbox:AddSlider("InsanePowerValue", {
		Text = "Insane Power Value",
		Default = 250,
		Min = 100,
		Max = 1000,
		Rounding = 0,
		Tooltip = "Sets how powerful will the insane power be.",
	})
	CharCustomMovesGroupbox:AddSlider("InsanePowerHeight", {
		Text = "Insane Power Height",
		Default = 10,
		Min = 0,
		Max = 100,
		Rounding = 0,
		Tooltip = "Sets how high will the insane power go.",
	})
end

--// Visuals Tab
local VisualsLightingTab = Tabs.Visuals:AddLeftGroupbox("Lighting")
local VisualsOtherTab = Tabs.Visuals:AddRightGroupbox("Other")
VisualsLightingTab:AddToggle("SkyboxColorToggle", {
	Text = "Skybox Color",
	Tooltip = "Changes the skybox color",
	Default = false,
	Visible = true,
}):AddColorPicker("SkyboxColor", {
	Default = Color3.new(1, 1, 1),
	Title = "Skybox Color",
}):AddColorPicker("SkyboxDecay", {
	Default = Color3.new(1, 1, 1),
	Title = "Skybox Decay",
})
VisualsLightingTab:AddSlider("SkyboxGlare", {
	Text = "Skybox Glare",
	Default = 1,
	Min = 0,
	Max = 10,
	Rounding = 2,
})
VisualsLightingTab:AddSlider("SkyboxHaze", {
	Text = "Skybox Haze",
	Default = 1,
	Min = 0,
	Max = 10,
	Rounding = 2,
})
VisualsLightingTab:AddDivider()
VisualsLightingTab:AddToggle("CorrectionToggle", {
	Text = "Correction",
	Tooltip = "Lets you correct the lighting in the game",
	Default = false,
	Visible = true,
}):AddColorPicker("CorrectionColor", {
	Default = Color3.new(1, 1, 1),
	Title = "Tint Color ",
})
VisualsLightingTab:AddSlider("CorrectionBrightness", {
	Text = "Brightness",
	Default = 0,
	Min = -1,
	Max = 1,
	Rounding = 2,
})
VisualsLightingTab:AddSlider("CorrectionContrast", {
	Text = "Contrast",
	Default = 0,
	Min = -1,
	Max = 1,
	Rounding = 2,
})
VisualsLightingTab:AddSlider("CorrectionSaturation", {
	Text = "Saturation",
	Default = 0,
	Min = -1,
	Max = 1,
	Rounding = 2,
})

VisualsOtherTab:AddToggle("DumbassToggle", {
	Text = "Dumbass on your screen",
	Tooltip = "What do you even want to know?",
	Default = false,
	Visible = true,
})

--// Miscs Tab

--// Settings Tab
local ConfigGroupbox = Tabs.Config:AddLeftGroupbox("Menu")

ConfigGroupbox:AddToggle("KeybindMenuOpen", {
	Default = Library.KeybindFrame.Visible,
	Text = "Open Keybind Menu",
	Callback = function(value)
		Library.KeybindFrame.Visible = value
	end,
})
ConfigGroupbox:AddToggle("ShowCustomCursor", {
	Text = "Custom Cursor",
	Default = true,
	Callback = function(Value)
		Library.ShowCustomCursor = Value
	end,
})
ConfigGroupbox:AddDropdown("NotificationSide", {
	Values = { "Left", "Right" },
	Default = "Right",
	Text = "Notification Side",
	Callback = function(Value)
		Library:SetNotifySide(Value)
	end,
})
ConfigGroupbox:AddDropdown("DPIDropdown", {
	Values = { "50%", "75%", "100%", "125%", "150%", "175%", "200%" },
	Default = "100%",
	Text = "DPI Scale",
	Callback = function(Value)
		Value = Value:gsub("%%", "")
		local DPI = tonumber(Value)
		Library:SetDPIScale(DPI)
	end,
})
ConfigGroupbox:AddDivider()
ConfigGroupbox:AddLabel("Menu bind")
	:AddKeyPicker("MenuKeybind", { Default = "RightShift", NoUI = true, Text = "Menu keybind" })

Library.ToggleKeybind = Options.MenuKeybind

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)

SaveManager:IgnoreThemeSettings()

SaveManager:SetIgnoreIndexes({"MenuKeybind"})

ThemeManager:SetFolder("gamesense-mps")
SaveManager:SetFolder("gamesense-mps/mps")

SaveManager:BuildConfigSection(Tabs.Config)

ThemeManager:ApplyToTab(Tabs.Config)

SaveManager:LoadAutoloadConfig()

Library:Notify({
	Title = "Ball Detection",
	Description = 'Balls detectadas: ' .. #_BALLS .. ' TPS',
	Time = 5,
})

--// UI Change Events
Toggles.DumbassToggle:OnChanged(function()
	Dumbass.Visible = Toggles.DumbassToggle.Value
end)

local OldAtmosphere = nil
local OldCorrection = nil
Toggles.SkyboxColorToggle:OnChanged(function()
	if Toggles.SkyboxColorToggle.Value then
		if Lighting:FindFirstChildOfClass("Atmosphere") then
			OldAtmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
			OldAtmosphere.Parent = nil
		end
		if Lighting:FindFirstChild("GS_ATMO") then
			Lighting:WaitForChild("GS_ATMO"):Destroy()
		end
		local GS_ATMO = Instance.new("Atmosphere")
		GS_ATMO.Name = "GS_ATMO"
		GS_ATMO.Density = 0
		GS_ATMO.Offset = 0
		GS_ATMO.Color = Options.SkyboxColor.Value
		GS_ATMO.Decay = Options.SkyboxDecay.Value
		GS_ATMO.Glare = Options.SkyboxGlare.Value
		GS_ATMO.Haze = Options.SkyboxHaze.Value
		GS_ATMO.Parent = Lighting
	else
		if Lighting:FindFirstChild("GS_ATMO") then
			Lighting:WaitForChild("GS_ATMO"):Destroy()
		end
		if OldAtmosphere then
			OldAtmosphere.Parent = Lighting
			OldAtmosphere = nil
		end
	end
end)
Toggles.CorrectionToggle:OnChanged(function()
	if Toggles.CorrectionToggle.Value then
		if Lighting:FindFirstChildOfClass("ColorCorrectionEffect") then
			OldCorrection = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
			OldCorrection.Parent = nil
		end
		if Lighting:FindFirstChild("GS_CCOR") then
			Lighting:WaitForChild("GS_CCOR"):Destroy()
		end
		local GS_CCOR = Instance.new("ColorCorrectionEffect")
		GS_CCOR.Name = "GS_CCOR"
		GS_CCOR.Brightness = Options.CorrectionBrightness.Value
		GS_CCOR.Contrast = Options.CorrectionContrast.Value
		GS_CCOR.Saturation = Options.CorrectionSaturation.Value
		GS_CCOR.TintColor = Options.CorrectionColor.Value
		GS_CCOR.Parent = Lighting
	else
		if Lighting:FindFirstChild("GS_CCOR") then
			Lighting:WaitForChild("GS_CCOR"):Destroy()
		end
		if OldCorrection then
			OldCorrection.Parent = Lighting
			OldCorrection = nil
		end
	end
end)

Options.SkyboxColor:OnChanged(function()
	if Lighting:FindFirstChild("GS_ATMO") then
		Lighting:FindFirstChild("GS_ATMO").Color = Options.SkyboxColor.Value
	end
end)
Options.SkyboxDecay:OnChanged(function()
	if Lighting:FindFirstChild("GS_ATMO") then
		Lighting:FindFirstChild("GS_ATMO").Decay = Options.SkyboxDecay.Value
	end
end)
Options.SkyboxGlare:OnChanged(function()
	if Lighting:FindFirstChild("GS_ATMO") then
		Lighting:FindFirstChild("GS_ATMO").Glare = Options.SkyboxGlare.Value
	end
end)
Options.SkyboxHaze:OnChanged(function()
	if Lighting:FindFirstChild("GS_ATMO") then
		Lighting:FindFirstChild("GS_ATMO").Haze = Options.SkyboxHaze.Value
	end
end)

Options.CorrectionColor:OnChanged(function()
	if Lighting:FindFirstChild("GS_CCOR") then
		Lighting:FindFirstChild("GS_CCOR").TintColor = Options.CorrectionColor.Value
	end
end)
Options.CorrectionBrightness:OnChanged(function()
	if Lighting:FindFirstChild("GS_CCOR") then
		Lighting:FindFirstChild("GS_CCOR").Brightness = Options.CorrectionBrightness.Value
	end
end)
Options.CorrectionContrast:OnChanged(function()
	if Lighting:FindFirstChild("GS_CCOR") then
		Lighting:FindFirstChild("GS_CCOR").Contrast = Options.CorrectionContrast.Value
	end
end)
Options.CorrectionSaturation:OnChanged(function()
	if Lighting:FindFirstChild("GS_CCOR") then
		Lighting:FindFirstChild("GS_CCOR").Saturation = Options.CorrectionSaturation.Value
	end
end)

local MacroBall = nil
local BallAttachment = nil
local CharAttachment = nil
local BallForce = nil
Toggles.HomboloMacroToggle:OnChanged(function()
	if Toggles.HomboloMacroToggle.Value then
		if HRP and Character and Character:FindFirstChild("Head") then
			local Balls = _BALLS
			table.sort(Balls, function(a, b) return (a.Position - HRP.Position).Magnitude < (b.Position - HRP.Position).Magnitude end)
			if #Balls > 0 then
				MacroBall = Balls[1]
				BallAttachment = Instance.new("Attachment")
				BallAttachment.Name = tostring(math.random(100000,999999))
				BallAttachment.Parent = MacroBall
				CharAttachment = Instance.new("Attachment")
				CharAttachment.Name = tostring(math.random(100000,999999))
				CharAttachment.Parent = Character.Head
				BallForce = Instance.new("AlignPosition")
				BallForce.Name = tostring(math.random(100000,999999))
				BallForce.ApplyAtCenterOfMass = true
				BallForce.Attachment0 = BallAttachment
				BallForce.Attachment1 = CharAttachment
				BallForce.ForceLimitMode = Enum.ForceLimitMode.PerAxis
				BallForce.MaxAxesForce = Vector3.new(math.huge, 0, math.huge)
				BallForce.Responsiveness = 200
				BallForce.Parent = MacroBall
			end
		end
	else
		if BallAttachment then BallAttachment:Destroy() end
		if CharAttachment then CharAttachment:Destroy() end
		if BallForce then BallForce:Destroy() end
		MacroBall = nil
	end
end)

local ReactKillerState = {
	Active = false,
	HbConn = nil,
	TouchConn = nil,
}

Toggles.ReactKillerToggle:OnChanged(function()
	ReactKillerState.Active = Toggles.ReactKillerToggle.Value

	if ReactKillerState.Active then

		local oldBillboardNI
		oldBillboardNI = hookmetamethod(Instance.new("BillboardGui"), "__newindex", newcclosure(function(Self, Key, Value)
			if not checkcaller() and Self.Name == "AnchorUI" and Key == "Enabled" then
				return
			end
			return oldBillboardNI(Self, Key, Value)
		end))

		local function NukeBubble(ball)
			local anchor = ball:FindFirstChild("AnchorUI")
			if anchor and anchor:IsA("BillboardGui") then
				anchor.Enabled = false
				anchor.StudsOffset = Vector3.new(0, 99999, 0)
				anchor.Size = UDim2.new(0, 0, 0, 0)
				local img = anchor:FindFirstChildOfClass("ImageLabel")
				if img then
					img.Visible = false
					img.ImageTransparency = 1
				end
			end
			for _, v in ball:GetChildren() do
				if v:IsA("BillboardGui") and v ~= anchor then
					v.Enabled = false
					v.StudsOffset = Vector3.new(0, 99999, 0)
					v.Size = UDim2.new(0, 0, 0, 0)
				end
			end
		end

		ReactKillerState.HbConn = RunService.Heartbeat:Connect(function()
			for _, ball in ipairs(_BALLS) do
				NukeBubble(ball)
				local ownerTag = ball:FindFirstChild("Owner")
				if ownerTag and ownerTag:IsA("ObjectValue") then
					if ownerTag.Value ~= LocalPlayer and ownerTag.Value ~= LocalPlayer.Name and ownerTag.Value ~= LocalPlayer.UserId then
						pcall(function()
							if ChangeOwner then
								ChangeOwner:FireServer(ball, "Tackle")
							end
						end)
					end
				end
			end
		end)

		
		ReactKillerState.TouchConn = RunService.RenderStepped:Connect(function()
			if not Character then return end
			for _, ball in ipairs(_BALLS) do
				for _, limb in Character:GetChildren() do
					if limb:IsA("BasePart") then
						firetouchinterest(limb, ball, 0)
						firetouchinterest(limb, ball, 1)
					end
				end
			end
		end)

		task.spawn(function()
			while ReactKillerState.Active do
				pcall(RefreshBalls)
				for _, ball in ipairs(_BALLS) do
					if ChangeOwner then
						pcall(function()
							ChangeOwner:FireServer(ball, "Tackle")
						end)
					end
				end
				task.wait(Options.ReactKillerPolling.Value)
			end
		end)

	else
		-- Cleanup
		if ReactKillerState.HbConn then
			ReactKillerState.HbConn:Disconnect()
			ReactKillerState.HbConn = nil
		end
		if ReactKillerState.TouchConn then
			ReactKillerState.TouchConn:Disconnect()
			ReactKillerState.TouchConn = nil
		end
	end
end)

--// AnimSocket Channel
local Channel = AnimSocket.Connect("GS-CHANNEL")
Channel.OnMessage:Connect(function(Player, Message)
	Message = string.split(Message, "+")
	if Message[1] == "kick" and Message[2] == LocalPlayer.Name then
		if Message[3] then
			LocalPlayer:Kick(Message[3])
		else
			LocalPlayer:Kick()
		end
	end
end)

--// Character Respawn + Bypass (handler unico)
LocalPlayer.CharacterAdded:Connect(function(NewCharacter)
	Character = NewCharacter
	Humanoid = Character:WaitForChild("Humanoid")
	HRP = NewCharacter:WaitForChild("HumanoidRootPart")

	task.defer(function()
		RefreshBalls()
		local sp = StarterPack
		local tm = sp:FindFirstChild("ToolManagement")
		if tm then
			MainModule = require(tm)
			MainModule.GetUsing = function() return false end
			MainModule.check = function() return "R" end
		end
	end)

	local BypassSuccess = pcall(function()
		for ID, Function in pairs(getgc(true)) do
			if typeof(Function) == "function" then
				local FunctionName = debug.info(Function, "n")
				if FunctionName == "reachcheck" then
					hookfunction(Function, newcclosure(function() return false end))
				elseif FunctionName == "touchingcheck" then
					hookfunction(Function, newcclosure(function() return false end))
				elseif FunctionName == "IsBallBoundingHitbox" then
					hookfunction(Function, newcclosure(function() return true end))
				elseif FunctionName == "CheckBodyParts" then
					hookfunction(Function, newcclosure(function() return false end))
				end
			end
		end
	end)
	if not BypassSuccess then
		Library:Notify({
			Title = "Critical Error",
			Description = "Bypass failed!",
			Time = 10,
		})
	end
end)

pcall(function()
	for ID, Function in pairs(getgc(true)) do
		if typeof(Function) == "function" then
			local FunctionName = debug.info(Function, "n")
			if FunctionName == "reachcheck" then
				hookfunction(Function, newcclosure(function() return false end))
			elseif FunctionName == "touchingcheck" then
				hookfunction(Function, newcclosure(function() return false end))
			elseif FunctionName == "IsBallBoundingHitbox" then
				hookfunction(Function, newcclosure(function() return true end))
			elseif FunctionName == "CheckBodyParts" then
				hookfunction(Function, newcclosure(function() return false end))
			end
		end
	end
end)


if MainModule then
	MainModule.GetUsing = function() return false end
	MainModule.check = function() return "R" end
end


if MainModule then
	local ApplyGKReact = ReplicatedStorage:FindFirstChild("ApplyGKReact")
	local Client = ClientRemote

	MainModule.ApplyGKForce = function(p42)
		if not Character or not HRP then return end

		local pingMs = math.round(LocalPlayer:GetNetworkPing() * 2000)
		if pingMs >= 1000 then return end

		if typeof(p42) ~= "Instance" or not p42:IsA("BasePart") then return end
		if p42.Name ~= "TPS" then return end
		if p42.Size ~= Vector3.new(2.5, 2.5, 2.5) then return end

		if Character:FindFirstChild("ValueSmartMPS") then return end
		if not Character:FindFirstChild("ValueGKMPS") then return end
		local team = LocalPlayer.Team
		if not team then return end
		local teamName = team.Name
		if teamName ~= "-Home GK" and teamName ~= "-Away GK" then return end

		local rightArm = Character:FindFirstChild("Right Arm")
		if rightArm and rightArm:FindFirstChild("handsOn") then return end

		if Client then Client:FireServer("BallSound", p42) end

		for _, child in p42:GetChildren() do
			if child:IsA("BodyVelocity") then child:Destroy() end
		end

		MainModule.DestroyBodyMovers()
		MainModule.StopPhysics()

		local Owner = p42:FindFirstChild("Owner")
		if Owner and Owner:IsA("ObjectValue") then
			if Owner.Value ~= LocalPlayer then
				if ChangeOwner then ChangeOwner:FireServer(p42, "Tackle") end
				if ApplyGKReact then ApplyGKReact:FireServer(p42, p42.CFrame) end
				return
			end
			if ApplyGKReact then ApplyGKReact:FireServer(p42, p42.CFrame) end
		end
	end

	MainModule.attachBall = function(p40)
		if not Character or not HRP then return end

		local pingMs = math.round(LocalPlayer:GetNetworkPing() * 2000)
		if pingMs >= 1000 then return end

		if typeof(p40) ~= "Instance" or not p40:IsA("BasePart") then return end
		if p40.Name ~= "TPS" then return end
		if p40.Size ~= Vector3.new(2.5, 2.5, 2.5) then return end

		if Character:FindFirstChild("ValueSmartMPS") then return end
		if not Character:FindFirstChild("ValueGKMPS") then return end

		if ChangeOwner then ChangeOwner:FireServer(p40, "Tackle") end
		if Client then Client:FireServer("BallSound", p40) end
		if CatchBall then CatchBall:FireServer(p40) end
	end
end

local oldPingIndex
oldPingIndex = hookmetamethod(LocalPlayer, "__index", newcclosure(function(Self, Key)
	if not checkcaller() and Self == LocalPlayer and Key == "GetNetworkPing" then
		return function() return 0.001 end
	end
	return oldPingIndex(Self, Key)
end))

pcall(function()
	local OldTouchingParts
	OldTouchingParts = hookmetamethod(Instance.new("Part"), "__namecall", newcclosure(function(Self, ...)
		local Method = getnamecallmethod()
		if not checkcaller() then
			if Method == "GetTouchingParts" then
				if table.find(_BALLS, Self) then
					local TouchingParts = OldTouchingParts(Self, ...)
					if Toggles.ReactKillerToggle.Value then
						for i = #TouchingParts, 1, -1 do
							local part = TouchingParts[i]
							if part:IsA("BasePart") then
								local character = part:FindFirstAncestorOfClass("Model")
								if character then
									local player = Players:GetPlayerFromCharacter(character)
									if player and player ~= LocalPlayer then
										table.remove(TouchingParts, i)
									end
								end
							end
						end
					end
					if Character then
						for ID, Limb in pairs(Character:GetChildren()) do
							if Limb:IsA("Part") then
								table.insert(TouchingParts, Limb)
							end
						end
					end
					return TouchingParts
				end
			elseif Method == "GetClosestPointOnSurface" then
				if Self.Name == "RL" or Self.Name == "LL" or Self.Name == "Right Leg" or Self.Name == "Left Leg" or Self.Name == "Right Arm" or Self.Name == "Left Arm" or Self.Name == "Head" or Self.Name == "Torso" or Self.Name == "HumanoidRootPart" then
					return ...
				end
			end
		end
		return OldTouchingParts(Self, ...)
	end))
end)

workspace.ChildAdded:Connect(function(v)
	if v:IsA("Part") and v.Name == BALL_NAME then
		table.insert(_BALLS, v)
	end
end)
workspace.ChildRemoved:Connect(function(v)
	if v:IsA("Part") and v.Name == BALL_NAME then
		for i = #_BALLS, 1, -1 do
			if _BALLS[i] == v then
				table.remove(_BALLS, i)
				break
			end
		end
	end
end)
local stadium = workspace:FindFirstChild("WorkspaceStadiumFolder")
if stadium then
	stadium.ChildAdded:Connect(function(v)
		if v:IsA("Part") and v.Name == BALL_NAME then
			table.insert(_BALLS, v)
		end
	end)
	stadium.ChildRemoved:Connect(function(v)
		if v:IsA("Part") and v.Name == BALL_NAME then
			for i = #_BALLS, 1, -1 do
				if _BALLS[i] == v then
					table.remove(_BALLS, i)
					break
				end
			end
		end
	end)
end

RunService.RenderStepped:Connect(function(DeltaTime)
	if HRP and Humanoid then
		if Toggles.ReachMasterToggle.Value then
			local ReachType = nil
			local InfReach = false
			if UserInputService.TouchEnabled and Toggles.ReachMainToggle and Toggles.ReachMainToggle.Value then
				ReachType = "Main"
				if Toggles.InfiniteReachMainToggle.Value then InfReach = true end
			elseif (Character:FindFirstChild("Shoot") or Character:FindFirstChild("Kick")) and Toggles.ReachShootToggle.Value then
				ReachType = "Shoot"
				if Toggles.InfiniteReachShootToggle.Value then InfReach = true end
			elseif Character:FindFirstChild("Pass") and Toggles.ReachPassToggle.Value then
				ReachType = "Pass"
				if Toggles.InfiniteReachPassToggle.Value then InfReach = true end
			elseif Character:FindFirstChild("Long") and Toggles.ReachLongToggle.Value then
				ReachType = "Long"
				if Toggles.InfiniteReachLongToggle.Value then InfReach = true end
			elseif Character:FindFirstChild("Tackle") and Toggles.ReachTackleToggle.Value then
				ReachType = "Tackle"
				if Toggles.InfiniteReachTackleToggle.Value then InfReach = true end
			elseif Character:FindFirstChild("Dribble") and Toggles.ReachDribbleToggle.Value then
				ReachType = "Dribble"
				if Toggles.InfiniteReachDribbleToggle.Value then InfReach = true end
			elseif (Character:FindFirstChild("Save") or Character:FindFirstChild("Clear") or Character:FindFirstChild("GK")) and Toggles.ReachSaveToggle.Value then
				ReachType = "Save"
				if Toggles.InfiniteReachSaveToggle.Value then InfReach = true end
			end
			if ReachType == nil or InfReach then
				ReachBox.Size = Vector3.new(0, 0, 0)
				ReachBox.CFrame = CFrame.new(math.huge, math.huge, math.huge)
			else
				ReachBox.Size = Vector3.new(Options["Reach"..ReachType.."SizeX"].Value, Options["Reach"..ReachType.."SizeY"].Value, Options["Reach"..ReachType.."SizeZ"].Value)
				ReachBox.CFrame = HRP.CFrame * CFrame.new(Options["Reach"..ReachType.."OffsetX"].Value, Options["Reach"..ReachType.."OffsetY"].Value, Options["Reach"..ReachType.."OffsetZ"].Value)
			end
			ReachBox.Color = Options.ReachVisualizerColor.Value
			if Toggles.ReachVisualizerToggle.Value then
				ReachBox.Transparency = Options.ReachVisualizerColor.Transparency
			else
				ReachBox.Transparency = 1
			end

			task.spawn(function()
				RunService.Heartbeat:Wait()
				local ReachOverlapParams = OverlapParams.new()
				ReachOverlapParams.FilterType = Enum.RaycastFilterType.Include
				ReachOverlapParams.FilterDescendantsInstances = {_BALLS}

				local TouchingBalls
				if InfReach then
					TouchingBalls = _BALLS
				else
					TouchingBalls = workspace:GetPartsInPart(ReachBox, ReachOverlapParams)
				end
				table.sort(TouchingBalls, function(a, b) return (a.Position - HRP.Position).Magnitude < (b.Position - HRP.Position).Magnitude end)
				if #TouchingBalls > 0 then
					local Ball = TouchingBalls[1]
					if Toggles["Reach"..ReachType.."CompToggle"].Value then
						if Ball:FindFirstChild("Owner") then
							local OwnerTag = Ball:FindFirstChild("Owner")
							if OwnerTag.Value == LocalPlayer or OwnerTag.Value == LocalPlayer.Name or OwnerTag.Value == LocalPlayer.UserId then
								return
							end
						end
					end
					for ID, Limb in pairs(Character:GetChildren()) do
						if Limb:IsA("Part") then
							firetouchinterest(Limb, Ball, 0)
							firetouchinterest(Limb, Ball, 1)
						end
					end
				end
			end)
		else
			ReachBox.Size = Vector3.new(0, 0, 0)
			ReachBox.CFrame = CFrame.new(math.huge, math.huge, math.huge)
		end
	end
end)

RunService.Heartbeat:Connect(function(DeltaTime)
	if HRP and Humanoid then
		if Toggles.CharSpeedToggle.Value then
			HRP:PivotTo(HRP.CFrame + Humanoid.MoveDirection * DeltaTime * Options.CharSpeed.Value)
		end
	end
end)

UserInputService.InputBegan:Connect(function(Key, GameProcessed)
	if not GameProcessed then
		if Key.KeyCode == Enum.KeyCode.G then
			if not UserInputService.TouchEnabled then
				if Character:FindFirstChild("Shoot") and Toggles.InsanePowerShoot.Value then
					local function FindOwnedBall()
						local best, bestDist = nil, math.huge
						for _, b in ipairs(_BALLS) do
							local ownerTag = b:FindFirstChild("Owner")
							if ownerTag and (ownerTag.Value == LocalPlayer or ownerTag.Value == LocalPlayer.Name) then
								local dist = (b.Position - HRP.Position).Magnitude
								if dist < bestDist then
									best, bestDist = b, dist
								end
							end
						end
						return best
					end
					local ball = FindOwnedBall()
					if ball and MainModule then
						
						local velocity = HRP.CFrame.LookVector * Options.InsanePowerValue.Value + Vector3.new(0, Options.InsanePowerHeight.Value, 0)
						local maxForce = Vector3.new(math.huge, math.huge, math.huge)
						pcall(function()
							MainModule.ApplyForce(ball, maxForce, velocity, "Right Leg", true)
						end)
					end
				end
			end
		end
	end
end)

task.spawn(function()
	while task.wait(Options.BallPredHZ.Value) do
		ClearDump("BallPrediction")
		if Toggles.BallPredToggle.Value then
			for _, Ball in ipairs(_BALLS) do
				if Ball.AssemblyLinearVelocity.Magnitude > Options.BallPredThreshold.Value then
					PredictBall(Ball)
				end
			end
		end
	end
end)
