if _G.THHGrowersHardUnloaded then
	return
end

if type(_G.THHGrowersCleanup) == "function" then
	pcall(_G.THHGrowersCleanup)
end

--// SERVICES

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local VirtualUser = game:GetService("VirtualUser")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local VirtualInputManager

pcall(function()
	VirtualInputManager = game:GetService("VirtualInputManager")
end)

local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = Workspace.CurrentCamera

--// VAMPAUTH - KEEPING YOUR EXISTING SYSTEM

local PROJECT_ID = "PD6XH2EGXZXUHGED"
local AUTH_SECRET = "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"

local GET_KEY_URL =
	"https://vampauth.com/PD6XH2EGXZXUHGED/flow"

local VAMPAUTH_CLIENT_URL =
	"https://vampauth.com/client/vampauth.lua"

--// SEARCH FOR THE NEEDLE IDS

local SEARCH_FOR_NEEDLE_PLACE_ID = 77108422251420
local FARMHOUSE_PLACE_ID = 108628039999641
local BASEMENT_PLACE_ID = 83445806734780

--// ICONS

local GAME_ID = game.GameId

if not GAME_ID or GAME_ID == 0 then
	GAME_ID = game.PlaceId
end

local GAME_ICON =
	"rbxthumb://type=GameIcon&id="
	.. tostring(GAME_ID)
	.. "&w=150&h=150"

local PLAYER_ICON =
	"rbxthumb://type=AvatarHeadShot&id="
	.. tostring(LocalPlayer.UserId)
	.. "&w=150&h=150"

--// COLORS

local Colors = {
	Background = Color3.fromRGB(10, 11, 14),
	Sidebar = Color3.fromRGB(14, 16, 20),
	Card = Color3.fromRGB(19, 22, 27),
	CardHover = Color3.fromRGB(24, 27, 33),
	Accent = Color3.fromRGB(86, 220, 132),
	AccentDark = Color3.fromRGB(27, 61, 41),
	Text = Color3.fromRGB(241, 243, 246),
	SubText = Color3.fromRGB(135, 140, 152),
	Stroke = Color3.fromRGB(38, 42, 50),
	Danger = Color3.fromRGB(224, 74, 74)
}

local Glass = {
	Panel = Color3.fromRGB(75, 78, 85),
	Card = Color3.fromRGB(100, 103, 111),
	Dark = Color3.fromRGB(45, 48, 54),
	Stroke = Color3.fromRGB(188, 191, 198),
	Text = Color3.fromRGB(246, 247, 249),
	SubText = Color3.fromRGB(205, 208, 214)
}

--// CONFIG

local SPEED_AMOUNT = 50
local FLY_SPEED = 58

local PICKUP_DISTANCE = 3
local SELL_DISTANCE = 3
local UFO_DISTANCE = 3

local NORMAL_CLICK_BURST = 6
local STRONG_CLICK_BURST = 8
local SELL_CLICK_BURST = 6

local CLICK_GAP = 0.025

local FARM_PICKUPS_PER_SELL = 4
local AUTO_SELL_INTERVAL = 2

--// STATE

local connections = {}

local screenGui
local mainFrame
local mainScale
local floatingButton
local notificationHolder

local VampauthClient

local cleaning = false
local interactionBusy = false

local autoFarmEnabled = false
local autoHayEnabled = false
local autoSellEnabled = false

local autoDiamondEnabled = false
local autoColorHayEnabled = false

local autoFindNeedleEnabled = false
local autoFindKeyEnabled = false

local noPickupCooldownEnabled = false
local infRangeEnabled = false

local speedEnabled = false
local infiniteJumpEnabled = false
local thirdPersonEnabled = false
local flyEnabled = false

local antiAfkEnabled = false
local antiAfkConnection

local partNameHoverEnabled = false
local partNameHoverConnection
local partNameHoverLabel

local flyConnection
local flyVelocity
local flyGyro
local flyHumanoid
local flyOldPlatformStand
local flyJumpUntil = 0

local originalCameraMode = LocalPlayer.CameraMode
local originalMinZoom = LocalPlayer.CameraMinZoomDistance
local originalMaxZoom = LocalPlayer.CameraMaxZoomDistance

local originalWalkSpeeds =
	setmetatable({}, {
		__mode = "k"
	})

local originalClickSettings =
	setmetatable({}, {
		__mode = "k"
	})

local originalPromptSettings =
	setmetatable({}, {
		__mode = "k"
	})

local recentAttempts =
	setmetatable({}, {
		__mode = "k"
	})

--// OBJECT CACHES

local hayCache = {}
local diamondCache = {}
local sellCache = {}
local ufoCache = {}
local needleCache = {}
local keyCache = {}

local colorHayState =
	setmetatable({}, {
		__mode = "k"
	})

local colorHayConnections =
	setmetatable({}, {
		__mode = "k"
	})

local colorHayQueue = {}
local colorHayQueued =
	setmetatable({}, {
		__mode = "k"
	})

local diamondQueue = {}
local diamondQueued =
	setmetatable({}, {
		__mode = "k"
	})

local needleQueue = {}
local needleQueued =
	setmetatable({}, {
		__mode = "k"
	})

local keyQueue = {}
local keyQueued =
	setmetatable({}, {
		__mode = "k"
	})

local notify = function()
end

--// BASIC HELPERS

local function track(connection)
	table.insert(connections, connection)
	return connection
end

local function create(className, properties)
	local object = Instance.new(className)

	for property, value in pairs(properties or {}) do
		object[property] = value
	end

	return object
end

local function corner(object, radius)
	return create("UICorner", {
		CornerRadius = UDim.new(0, radius or 8),
		Parent = object
	})
end

local function stroke(object, color, transparency, thickness)
	return create("UIStroke", {
		Color = color or Colors.Stroke,
		Transparency = transparency or 0,
		Thickness = thickness or 1,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		Parent = object
	})
end

local function padding(object, left, right, top, bottom)
	return create("UIPadding", {
		PaddingLeft = UDim.new(0, left or 0),
		PaddingRight = UDim.new(0, right or 0),
		PaddingTop = UDim.new(0, top or 0),
		PaddingBottom = UDim.new(0, bottom or 0),
		Parent = object
	})
end

local function tween(object, duration, properties)
	local animation =
		TweenService:Create(
			object,
			TweenInfo.new(
				duration or 0.16,
				Enum.EasingStyle.Quad,
				Enum.EasingDirection.Out
			),
			properties
		)

	animation:Play()

	return animation
end

local function copyText(text)
	if type(setclipboard) == "function" then
		return pcall(setclipboard, tostring(text))
	end

	if type(toclipboard) == "function" then
		return pcall(toclipboard, tostring(text))
	end

	return false
end

local function getCharacter()
	return LocalPlayer.Character
end

local function getHumanoid()
	local character = getCharacter()

	if not character then
		return nil
	end

	return character:FindFirstChildOfClass("Humanoid")
end

local function getRoot()
	local character = getCharacter()

	if not character then
		return nil
	end

	return character:FindFirstChild("HumanoidRootPart")
		or character:FindFirstChild("UpperTorso")
		or character:FindFirstChild("Torso")
end

local function getMainPart(object)
	if not object then
		return nil
	end

	if object:IsA("BasePart") then
		return object
	end

	if object:IsA("Model") and object.PrimaryPart then
		return object.PrimaryPart
	end

	if object:IsA("Tool") then
		return object:FindFirstChild("Handle")
			or object:FindFirstChildWhichIsA("BasePart", true)
	end

	return object:FindFirstChildWhichIsA("BasePart", true)
end

local function getClickDetector(object)
	if not object then
		return nil
	end

	if object:IsA("ClickDetector") then
		return object
	end

	return object:FindFirstChildWhichIsA(
		"ClickDetector",
		true
	)
end

local function getObjectPivot(object)
	if not object then
		return nil
	end

	if object:IsA("Model") then
		return object:GetPivot()
	end

	if object:IsA("BasePart") then
		return object.CFrame
	end

	local part = getMainPart(object)

	if part then
		return part.CFrame
	end

	return nil
end

local function setObjectPivot(object, cframe)
	if not object
		or not object.Parent
		or not cframe then

		return false
	end

	if object:IsA("Model") then
		return pcall(function()
			object:PivotTo(cframe)
		end)
	end

	if object:IsA("BasePart") then
		return pcall(function()
			object.CFrame = cframe
		end)
	end

	local part = getMainPart(object)

	if part then
		return pcall(function()
			part.CFrame = cframe
		end)
	end

	return false
end

local function getPartFromObject(object)
	local current = object

	while current and current ~= Workspace do
		if current:IsA("BasePart") then
			return current
		end

		current = current.Parent
	end

	return nil
end

local function stopVelocity()
	local root = getRoot()

	if not root then
		return
	end

	pcall(function()
		root.AssemblyLinearVelocity = Vector3.zero
		root.AssemblyAngularVelocity = Vector3.zero
	end)
end

local function getDistanceTo(object)
	local root = getRoot()
	local part = getMainPart(object)

	if not root or not part then
		return math.huge
	end

	return (root.Position - part.Position).Magnitude
end

local function withInteractionLock(callback)
	while interactionBusy do
		task.wait(0.025)
	end

	interactionBusy = true

	local success, result =
		pcall(callback)

	interactionBusy = false

	if success then
		return result
	end

	return false
end

--// CAMERA LOCK
--// USED FOR PICKUPS + INF RANGE
--// NOT USED FOR AUTO SELL

local function lockCamera()
	Camera = Workspace.CurrentCamera or Camera

	if not Camera then
		return function()
		end
	end

	local oldCameraType = Camera.CameraType
	local oldCameraSubject = Camera.CameraSubject
	local oldCFrame = Camera.CFrame
	local oldFocus = Camera.Focus

	Camera.CameraType =
		Enum.CameraType.Scriptable

	Camera.CFrame = oldCFrame
	Camera.Focus = oldFocus

	local connection

	connection =
		RunService.RenderStepped:Connect(function()
			if not Camera then
				return
			end

			Camera.CFrame = oldCFrame
			Camera.Focus = oldFocus
		end)

	local finished = false

	return function()
		if finished then
			return
		end

		finished = true

		if connection then
			pcall(function()
				connection:Disconnect()
			end)
		end

		Camera = Workspace.CurrentCamera or Camera

		if not Camera then
			return
		end

		pcall(function()
			Camera.CFrame = oldCFrame
			Camera.Focus = oldFocus
			Camera.CameraSubject = oldCameraSubject
			Camera.CameraType = oldCameraType
		end)
	end
end

--// CLICK DETECTOR / PROMPT SETTINGS

local function rememberClickDetector(detector)
	if originalClickSettings[detector] then
		return
	end

	originalClickSettings[detector] = {
		MaxActivationDistance =
			detector.MaxActivationDistance
	}
end

local function rememberPrompt(prompt)
	if originalPromptSettings[prompt] then
		return
	end

	originalPromptSettings[prompt] = {
		HoldDuration = prompt.HoldDuration,

		MaxActivationDistance =
			prompt.MaxActivationDistance,

		RequiresLineOfSight =
			prompt.RequiresLineOfSight,

		Enabled =
			prompt.Enabled
	}
end

local function applyClickDetector(detector)
	if not detector
		or not detector.Parent then

		return
	end

	rememberClickDetector(detector)

	local original =
		originalClickSettings[detector]

	pcall(function()
		detector.MaxActivationDistance =
			infRangeEnabled
			and 1000000
			or original.MaxActivationDistance
	end)
end

local function applyPrompt(prompt)
	if not prompt
		or not prompt.Parent then

		return
	end

	rememberPrompt(prompt)

	local original =
		originalPromptSettings[prompt]

	pcall(function()
		prompt.HoldDuration =
			noPickupCooldownEnabled
			and 0
			or original.HoldDuration

		prompt.MaxActivationDistance =
			infRangeEnabled
			and 1000000
			or original.MaxActivationDistance

		prompt.RequiresLineOfSight =
			infRangeEnabled
			and false
			or original.RequiresLineOfSight

		prompt.Enabled =
			(noPickupCooldownEnabled or infRangeEnabled)
			and true
			or original.Enabled
	end)
end

local function applyInteractionObject(object)
	if object:IsA("ClickDetector") then
		applyClickDetector(object)

	elseif object:IsA("ProximityPrompt") then
		applyPrompt(object)
	end
end

local function applySettingsToObject(object)
	if not object or not object.Parent then
		return
	end

	applyInteractionObject(object)

	for _, descendant in ipairs(
		object:GetDescendants()
	) do
		applyInteractionObject(descendant)
	end
end

local function refreshCachedSettings()
	for detector in pairs(originalClickSettings) do
		if detector and detector.Parent then
			applyClickDetector(detector)
		end
	end

	for prompt in pairs(originalPromptSettings) do
		if prompt and prompt.Parent then
			applyPrompt(prompt)
		end
	end

	for object in pairs(hayCache) do
		if object.Parent then
			applySettingsToObject(object)
		end
	end

	for object in pairs(diamondCache) do
		if object.Parent then
			applySettingsToObject(object)
		end
	end

	for object in pairs(needleCache) do
		if object.Parent then
			applySettingsToObject(object)
		end
	end

	for object in pairs(keyCache) do
		if object.Parent then
			applySettingsToObject(object)
		end
	end
end

local function restoreInteractionSettings()
	for detector, original in pairs(
		originalClickSettings
	) do
		if detector and detector.Parent then
			pcall(function()
				detector.MaxActivationDistance =
					original.MaxActivationDistance
			end)
		end
	end

	for prompt, original in pairs(
		originalPromptSettings
	) do
		if prompt and prompt.Parent then
			pcall(function()
				prompt.HoldDuration =
					original.HoldDuration

				prompt.MaxActivationDistance =
					original.MaxActivationDistance

				prompt.RequiresLineOfSight =
					original.RequiresLineOfSight

				prompt.Enabled =
					original.Enabled
			end)
		end
	end
end

--// OBJECT IDENTIFICATION

local function looksLikeNeedle(object)
	if not object then
		return false
	end

	if not object:IsA("BasePart")
		and not object:IsA("Model")
		and not object:IsA("Tool") then

		return false
	end

	local name =
		string.lower(object.Name)

	return name == "needle"
		or name == "needlepart"
		or name == "hiddenneedle"
		or string.find(
			name,
			"needle",
			1,
			true
		) ~= nil
end

local function looksLikeKey(object)
	if not object then
		return false
	end

	if not object:IsA("BasePart")
		and not object:IsA("Model")
		and not object:IsA("Tool") then

		return false
	end

	local name =
		string.lower(object.Name)

	if name == "key"
		or name == "keypart"
		or name == "basementkey"
		or name == "haykey"
		or name == "chapter2key"
		or name == "exitkey" then

		return true
	end

	if string.find(
		name,
		"key",
		1,
		true
	) then

		if string.find(name, "keypad", 1, true)
			or string.find(name, "keyboard", 1, true)
			or string.find(name, "keyhole", 1, true)
			or string.find(name, "monkey", 1, true)
			or string.find(name, "donkey", 1, true) then

			return false
		end

		return true
	end

	return false
end

local function getTrackedPickupRoot(object)
	local current = object

	while current and current ~= Workspace do
		if hayCache[current]
			or diamondCache[current]
			or needleCache[current]
			or keyCache[current] then

			return current
		end

		current = current.Parent
	end

	return nil
end

--// QUEUES

local function queueObject(
	queue,
	queued,
	object
)
	if not object
		or not object.Parent
		or queued[object] then

		return
	end

	queued[object] = true

	table.insert(queue, object)
end

local function queueDiamond(object)
	queueObject(
		diamondQueue,
		diamondQueued,
		object
	)
end

local function queueColorHay(object)
	queueObject(
		colorHayQueue,
		colorHayQueued,
		object
	)
end

local function queueNeedle(object)
	queueObject(
		needleQueue,
		needleQueued,
		object
	)
end

local function queueKey(object)
	queueObject(
		keyQueue,
		keyQueued,
		object
	)
end

--// COLOR HAY EVENT TRACKER

local function setupColorHayTracker(hay)
	if colorHayConnections[hay] then
		return
	end

	local part = getMainPart(hay)

	if not part then
		return
	end

	colorHayState[hay] = {
		LastColor = part.Color,
		LastChanged = 0
	}

	local connection

	connection =
		part:GetPropertyChangedSignal(
			"Color"
		):Connect(function()

			if not hay.Parent
				or not part.Parent then

				if connection then
					connection:Disconnect()
				end

				colorHayConnections[hay] = nil

				return
			end

			local state =
				colorHayState[hay]

			if not state then
				state = {
					LastColor = part.Color,
					LastChanged = 0
				}

				colorHayState[hay] = state
			end

			local previous =
				state.LastColor

			local current =
				part.Color

			local difference =
				math.abs(previous.R - current.R)
				+ math.abs(previous.G - current.G)
				+ math.abs(previous.B - current.B)

			state.LastColor = current

			if difference > 0.015 then
				state.LastChanged =
					os.clock()

				if autoColorHayEnabled then
					queueColorHay(hay)
				end
			end
		end)

	colorHayConnections[hay] =
		connection

	track(connection)
end

--// CACHE REGISTRATION

local function registerObject(object)
	if object.Name == "HayPiece" then
		if not hayCache[object] then
			hayCache[object] = true

			applySettingsToObject(object)
			setupColorHayTracker(object)
		end
	end

	if object.Name == "Diamond" then
		if not diamondCache[object] then
			diamondCache[object] = true

			applySettingsToObject(object)

			if autoDiamondEnabled then
				queueDiamond(object)
			end
		end
	end

	if object.Name == "SellPart" then
		sellCache[object] = true
	end

	if object.Name == "UfoButtenPart" then
		ufoCache[object] = true
	end

	if looksLikeNeedle(object) then
		if not needleCache[object] then
			needleCache[object] = true

			applySettingsToObject(object)

			if autoFindNeedleEnabled then
				queueNeedle(object)
			end
		end
	end

	if looksLikeKey(object) then
		if not keyCache[object] then
			keyCache[object] = true

			applySettingsToObject(object)

			if autoFindKeyEnabled then
				queueKey(object)
			end
		end
	end

	if object:IsA("ClickDetector")
		or object:IsA("ProximityPrompt") then

		local root =
			getTrackedPickupRoot(object)

		if root then
			applyInteractionObject(object)
		end
	end
end

local function unregisterObject(object)
	hayCache[object] = nil
	diamondCache[object] = nil
	sellCache[object] = nil
	ufoCache[object] = nil
	needleCache[object] = nil
	keyCache[object] = nil

	colorHayState[object] = nil

	diamondQueued[object] = nil
	colorHayQueued[object] = nil
	needleQueued[object] = nil
	keyQueued[object] = nil
end

-- ONE INITIAL SCAN ONLY

for _, object in ipairs(
	Workspace:GetDescendants()
) do
	registerObject(object)
end

-- AFTER THAT EVERYTHING IS EVENT BASED

track(
	Workspace.DescendantAdded:Connect(function(object)
		registerObject(object)
	end)
)

track(
	Workspace.DescendantRemoving:Connect(function(object)
		unregisterObject(object)
	end)
)

--// MOVEMENT

local function moveCharacterNearPosition(
	position,
	distance
)
	local character = getCharacter()
	local root = getRoot()

	if not character
		or not root
		or not position then

		return false
	end

	distance =
		distance or PICKUP_DISTANCE

	local direction =
		Vector3.new(
			root.Position.X - position.X,
			0,
			root.Position.Z - position.Z
		)

	if direction.Magnitude < 0.1 then
		direction =
			Vector3.new(0, 0, 1)
	end

	direction = direction.Unit

	local positionToUse =
		position
		+ direction * distance

	positionToUse =
		Vector3.new(
			positionToUse.X,
			position.Y + 2.7,
			positionToUse.Z
		)

	local lookPosition =
		Vector3.new(
			position.X,
			positionToUse.Y,
			position.Z
		)

	pcall(function()
		character:PivotTo(
			CFrame.lookAt(
				positionToUse,
				lookPosition
			)
		)
	end)

	stopVelocity()

	return true
end

local function moveObjectInFront(
	object,
	distance
)
	if not object
		or not object.Parent then

		return false
	end

	Camera =
		Workspace.CurrentCamera
		or Camera

	distance = distance or 5

	if Camera then
		return setObjectPivot(
			object,
			Camera.CFrame
				* CFrame.new(
					0,
					0,
					-distance
				)
		)
	end

	local root = getRoot()

	if root then
		return setObjectPivot(
			object,
			root.CFrame
				* CFrame.new(
					0,
					0,
					-3
				)
		)
	end

	return false
end

--// CLICKING

local function directClick(detector)
	if not detector
		or not detector.Parent then

		return false
	end

	applyClickDetector(detector)

	if type(fireclickdetector) == "function" then
		return pcall(function()
			fireclickdetector(detector)
		end)
	end

	return false
end

local function screenClickPart(part)
	if not part
		or not part.Parent then

		return false
	end

	Camera =
		Workspace.CurrentCamera
		or Camera

	if not Camera then
		return false
	end

	local position, visible =
		Camera:WorldToViewportPoint(
			part.Position
		)

	if not visible
		or position.Z <= 0 then

		return false
	end

	if VirtualInputManager then
		local success =
			pcall(function()

				VirtualInputManager:SendMouseButtonEvent(
					position.X,
					position.Y,
					0,
					true,
					game,
					0
				)

				RunService.Heartbeat:Wait()

				VirtualInputManager:SendMouseButtonEvent(
					position.X,
					position.Y,
					0,
					false,
					game,
					0
				)
			end)

		if success then
			return true
		end
	end

	if type(mousemoveabs) == "function"
		and type(mouse1click) == "function" then

		return pcall(function()
			mousemoveabs(
				position.X,
				position.Y
			)

			mouse1click()
		end)
	end

	return false
end

local function clickObjectOnce(object)
	if not object
		or not object.Parent then

		return false
	end

	local detector =
		getClickDetector(object)

	if detector
		and directClick(detector) then

		return true
	end

	local part =
		getMainPart(object)

	if part then
		return screenClickPart(part)
	end

	return false
end

local function clickObjectBurst(
	object,
	count,
	gap
)
	if not object
		or not object.Parent then

		return false
	end

	count = count or NORMAL_CLICK_BURST
	gap = gap or CLICK_GAP

	local detector =
		getClickDetector(object)

	local succeeded = false

	for _ = 1, count do
		if not object
			or not object.Parent then

			succeeded = true
			break
		end

		local currentSuccess = false

		if detector
			and detector.Parent then

			currentSuccess =
				directClick(detector)
		end

		if not currentSuccess then
			local part =
				getMainPart(object)

			if part then
				currentSuccess =
					screenClickPart(part)
			end
		end

		if currentSuccess then
			succeeded = true
		end

		if not object.Parent then
			break
		end

		if noPickupCooldownEnabled then
			RunService.Heartbeat:Wait()
		else
			task.wait(gap)
		end
	end

	return succeeded
end

--// PICKUP SEQUENCE

local function pickupObject(
	object,
	options
)
	options = options or {}

	if not object
		or not object.Parent then

		return false
	end

	-- prevents hammering the exact same surviving instance
	local previousAttempt =
		recentAttempts[object]

	if previousAttempt
		and os.clock() - previousAttempt < 0.08 then

		return false
	end

	recentAttempts[object] =
		os.clock()

	return withInteractionLock(function()

		local character = getCharacter()
		local root = getRoot()

		local part =
			getMainPart(object)

		if not character
			or not root
			or not part then

			return false
		end

		local oldCharacterPivot =
			character:GetPivot()

		local oldObjectPivot =
			getObjectPivot(object)

		if not oldObjectPivot then
			return false
		end

		local originalWorldPosition =
			oldObjectPivot.Position

		local unlockCamera

		if options.LockCamera ~= false then
			unlockCamera =
				lockCamera()
		else
			unlockCamera =
				function()
				end
		end

		-- First TP player near where object ACTUALLY was.

		moveCharacterNearPosition(
			originalWorldPosition,
			options.Distance
				or PICKUP_DISTANCE
		)

		RunService.Heartbeat:Wait()

		if not object.Parent then
			pcall(function()
				character:PivotTo(
					oldCharacterPivot
				)
			end)

			unlockCamera()

			return true
		end

		-- Now put object directly in front of the locked camera.

		moveObjectInFront(
			object,
			options.FrontDistance or 5
		)

		RunService.RenderStepped:Wait()

		local clicked =
			clickObjectBurst(
				object,
				options.ClickCount
					or NORMAL_CLICK_BURST,
				options.ClickGap
					or CLICK_GAP
			)

		-- Restore object if the game did not remove it.

		if object
			and object.Parent
			and oldObjectPivot then

			setObjectPivot(
				object,
				oldObjectPivot
			)
		end

		-- Restore player.

		if character
			and character.Parent then

			pcall(function()
				character:PivotTo(
					oldCharacterPivot
				)
			end)

			stopVelocity()
		end

		unlockCamera()

		return clicked
	end)
end

--// CACHE HELPERS

local function getCachedList(cache)
	local result = {}

	for object in pairs(cache) do
		if object and object.Parent then
			table.insert(
				result,
				object
			)
		end
	end

	return result
end

local function getNearest(cache)
	local list =
		getCachedList(cache)

	if #list == 0 then
		return nil
	end

	table.sort(
		list,
		function(a, b)
			return getDistanceTo(a)
				< getDistanceTo(b)
		end
	)

	return list[1]
end

--// AUTO SELL
--// IMPORTANT:
--// CAMERA IS NOT LOCKED HERE

local function getSellSearchObjects(sellPart)
	local result = {}
	local seen = {}

	local function add(object)
		if object and not seen[object] then
			seen[object] = true

			table.insert(
				result,
				object
			)
		end
	end

	add(sellPart)

	for _, object in ipairs(
		sellPart:GetDescendants()
	) do
		add(object)
	end

	local parent =
		sellPart.Parent

	if parent
		and parent ~= Workspace then

		add(parent)

		for _, object in ipairs(
			parent:GetDescendants()
		) do
			add(object)
		end
	end

	local grandParent =
		parent
		and parent.Parent

	if grandParent
		and grandParent ~= Workspace
		and grandParent ~= game then

		for _, object in ipairs(
			grandParent:GetDescendants()
		) do
			if object:IsA("ClickDetector")
				or object:IsA("ProximityPrompt")
				or object.Name == "SellPart" then

				add(object)
			end
		end
	end

	return result
end

local function scanSellPart(sellPart)
	if not sellPart
		or not sellPart.Parent then

		return nil
	end

	local result = {
		InteractionPart =
			getMainPart(sellPart),

		ClickDetector = nil,

		Prompt = nil
	}

	local mainPart =
		getMainPart(sellPart)

	local bestClickScore =
		math.huge

	for _, object in ipairs(
		getSellSearchObjects(sellPart)
	) do
		if object:IsA("ClickDetector") then
			local clickPart =
				getPartFromObject(object)

			local score = 20

			if clickPart == mainPart then
				score = 0

			elseif clickPart
				and clickPart.Name == "SellPart" then

				score = 1

			elseif object.Parent == sellPart then
				score = 2
			end

			if score < bestClickScore then
				bestClickScore = score

				result.ClickDetector =
					object

				if clickPart then
					result.InteractionPart =
						clickPart
				end
			end

		elseif object:IsA("ProximityPrompt") then
			result.Prompt =
				result.Prompt
				or object
		end
	end

	return result
end

local function sellOnePart(sellPart)
	if not sellPart
		or not sellPart.Parent then

		return false
	end

	return withInteractionLock(function()

		local scan =
			scanSellPart(sellPart)

		if not scan then
			return false
		end

		local character = getCharacter()
		local root = getRoot()
		local humanoid = getHumanoid()

		local part =
			scan.InteractionPart
			or getMainPart(sellPart)

		if not character
			or not root
			or not part then

			return false
		end

		local oldPivot =
			character:GetPivot()

		local oldAnchored =
			root.Anchored

		local oldAutoRotate

		if humanoid then
			oldAutoRotate =
				humanoid.AutoRotate
		end

		-- freeze player

		pcall(function()
			root.Anchored = true
		end)

		if humanoid then
			pcall(function()
				humanoid.AutoRotate = false
			end)
		end

		-- REAL TP TO SELLPART

		moveCharacterNearPosition(
			part.Position,
			SELL_DISTANCE
		)

		stopVelocity()

		task.wait(0.1)

		local clicked = false

		if scan.ClickDetector
			and scan.ClickDetector.Parent then

			for _ = 1, SELL_CLICK_BURST do
				local success =
					directClick(
						scan.ClickDetector
					)

				if success then
					clicked = true
				end

				task.wait(0.03)
			end
		end

		if not clicked then
			for _ = 1, SELL_CLICK_BURST do
				if screenClickPart(part) then
					clicked = true
				end

				task.wait(0.03)
			end
		end

		task.wait(0.05)

		-- return

		if character
			and character.Parent then

			pcall(function()
				character:PivotTo(
					oldPivot
				)
			end)

			stopVelocity()
		end

		-- unfreeze

		if root
			and root.Parent then

			pcall(function()
				root.Anchored =
					oldAnchored
			end)
		end

		if humanoid
			and humanoid.Parent
			and oldAutoRotate ~= nil then

			pcall(function()
				humanoid.AutoRotate =
					oldAutoRotate
			end)
		end

		return clicked
	end)
end

local function sellNow()
	local list =
		getCachedList(
			sellCache
		)

	if #list == 0 then
		return false
	end

	table.sort(
		list,
		function(a, b)
			return getDistanceTo(a)
				< getDistanceTo(b)
		end
	)

	for _, sellPart in ipairs(list) do
		if sellPart.Parent
			and sellOnePart(sellPart) then

			return true
		end
	end

	return false
end

--// AUTO FARM
--// 4 HAY -> SELL -> REPEAT

local function startAutoFarm()
	if autoFarmEnabled then
		return
	end

	autoFarmEnabled = true

	task.spawn(function()

		while autoFarmEnabled
			and screenGui
			and screenGui.Parent do

			local collected = 0
			local usedThisRound = {}

			while autoFarmEnabled
				and collected
					< FARM_PICKUPS_PER_SELL do

				local hayList =
					getCachedList(
						hayCache
					)

				table.sort(
					hayList,
					function(a, b)
						return getDistanceTo(a)
							< getDistanceTo(b)
					end
				)

				local target

				for _, hay in ipairs(hayList) do
					if hay.Parent
						and not usedThisRound[hay] then

						target = hay
						break
					end
				end

				if not target then
					task.wait(0.1)
					break
				end

				usedThisRound[target] =
					true

				pickupObject(
					target,
					{
						LockCamera = true,
						ClickCount = NORMAL_CLICK_BURST,
						Distance = PICKUP_DISTANCE,
						FrontDistance = 5
					}
				)

				collected += 1

				task.wait(
					noPickupCooldownEnabled
					and 0.01
					or 0.04
				)
			end

			if autoFarmEnabled
				and collected > 0 then

				sellNow()
			end

			task.wait(0.08)
		end
	end)
end

local function stopAutoFarm()
	autoFarmEnabled = false
end

--// AUTO HAY

local function startAutoHay()
	if autoHayEnabled then
		return
	end

	autoHayEnabled = true

	task.spawn(function()
		while autoHayEnabled
			and screenGui
			and screenGui.Parent do

			local hay =
				getNearest(hayCache)

			if hay and hay.Parent then
				pickupObject(
					hay,
					{
						LockCamera = true,
						ClickCount = NORMAL_CLICK_BURST
					}
				)
			else
				task.wait(0.1)
			end

			task.wait(
				noPickupCooldownEnabled
				and 0.01
				or 0.05
			)
		end
	end)
end

local function stopAutoHay()
	autoHayEnabled = false
end

--// AUTO SELL LOOP

local function startAutoSell()
	if autoSellEnabled then
		return
	end

	autoSellEnabled = true

	task.spawn(function()

		while autoSellEnabled
			and screenGui
			and screenGui.Parent do

			sellNow()

			local started =
				os.clock()

			while autoSellEnabled
				and os.clock() - started
					< AUTO_SELL_INTERVAL do

				task.wait(0.1)
			end
		end
	end)
end

local function stopAutoSell()
	autoSellEnabled = false
end

--// DIAMOND

local function startAutoDiamond()
	if autoDiamondEnabled then
		return
	end

	autoDiamondEnabled = true

	for object in pairs(diamondCache) do
		if object.Parent then
			queueDiamond(object)
		end
	end

	task.spawn(function()

		while autoDiamondEnabled
			and screenGui
			and screenGui.Parent do

			local object =
				table.remove(
					diamondQueue,
					1
				)

			if object then
				diamondQueued[object] = nil

				if object.Parent then
					pickupObject(
						object,
						{
							LockCamera = true,
							ClickCount = STRONG_CLICK_BURST,
							FrontDistance = 5
						}
					)

					if object.Parent
						and autoDiamondEnabled then

						task.delay(
							0.4,
							function()
								if autoDiamondEnabled
									and object.Parent then

									queueDiamond(object)
								end
							end
						)
					end
				end
			else
				task.wait(0.08)
			end
		end
	end)
end

local function stopAutoDiamond()
	autoDiamondEnabled = false

	table.clear(diamondQueue)

	for object in pairs(diamondQueued) do
		diamondQueued[object] = nil
	end
end

--// COLOR HAY
--// CAMERA LOCKED
--// STRONG 8-CLICK BURST

local function startAutoColorHay()
	if autoColorHayEnabled then
		return
	end

	autoColorHayEnabled = true

	for hay, state in pairs(
		colorHayState
	) do
		if hay.Parent
			and state.LastChanged > 0
			and os.clock()
				- state.LastChanged
				< 1.5 then

			queueColorHay(hay)
		end
	end

	task.spawn(function()

		while autoColorHayEnabled
			and screenGui
			and screenGui.Parent do

			local hay =
				table.remove(
					colorHayQueue,
					1
				)

			if hay then
				colorHayQueued[hay] = nil

				if hay.Parent then
					local state =
						colorHayState[hay]

					if state
						and os.clock()
							- state.LastChanged
							<= 1.5 then

						pickupObject(
							hay,
							{
								LockCamera = true,
								ClickCount = STRONG_CLICK_BURST,
								ClickGap = 0.02,
								FrontDistance = 4.5
							}
						)
					end
				end
			else
				task.wait(0.05)
			end
		end
	end)
end

local function stopAutoColorHay()
	autoColorHayEnabled = false

	table.clear(colorHayQueue)

	for object in pairs(colorHayQueued) do
		colorHayQueued[object] = nil
	end
end

--// AUTO FIND NEEDLE

local function startAutoFindNeedle()
	if autoFindNeedleEnabled then
		return
	end

	autoFindNeedleEnabled = true

	for object in pairs(needleCache) do
		if object.Parent then
			queueNeedle(object)
		end
	end

	task.spawn(function()

		while autoFindNeedleEnabled
			and screenGui
			and screenGui.Parent do

			local object =
				table.remove(
					needleQueue,
					1
				)

			if object then
				needleQueued[object] = nil

				if object.Parent then
					local success =
						pickupObject(
							object,
							{
								LockCamera = true,
								ClickCount = STRONG_CLICK_BURST
							}
						)

					if success then
						notify(
							"Needle",
							"Needle interaction sent.",
							2
						)
					end
				end
			else
				task.wait(0.1)
			end
		end
	end)
end

local function stopAutoFindNeedle()
	autoFindNeedleEnabled = false

	table.clear(needleQueue)

	for object in pairs(needleQueued) do
		needleQueued[object] = nil
	end
end

--// AUTO FIND KEY

local function startAutoFindKey()
	if autoFindKeyEnabled then
		return
	end

	autoFindKeyEnabled = true

	for object in pairs(keyCache) do
		if object.Parent then
			queueKey(object)
		end
	end

	task.spawn(function()

		while autoFindKeyEnabled
			and screenGui
			and screenGui.Parent do

			local object =
				table.remove(
					keyQueue,
					1
				)

			if object then
				keyQueued[object] = nil

				if object.Parent then
					local success =
						pickupObject(
							object,
							{
								LockCamera = true,
								ClickCount = STRONG_CLICK_BURST
							}
						)

					if success then
						notify(
							"Key",
							"Key interaction sent.",
							2
						)
					end
				end
			else
				task.wait(0.1)
			end
		end
	end)
end

local function stopAutoFindKey()
	autoFindKeyEnabled = false

	table.clear(keyQueue)

	for object in pairs(keyQueued) do
		keyQueued[object] = nil
	end
end

--// INF RANGE
--// CAMERA LOCKED

local function getRayTarget()
	if Mouse and Mouse.Target then
		return Mouse.Target
	end

	Camera =
		Workspace.CurrentCamera
		or Camera

	if not Camera then
		return nil
	end

	local viewport =
		Camera.ViewportSize

	local ray =
		Camera:ViewportPointToRay(
			viewport.X / 2,
			viewport.Y / 2
		)

	local params =
		RaycastParams.new()

	params.FilterType =
		Enum.RaycastFilterType.Exclude

	if LocalPlayer.Character then
		params.FilterDescendantsInstances = {
			LocalPlayer.Character
		}
	end

	local result =
		Workspace:Raycast(
			ray.Origin,
			ray.Direction * 10000,
			params
		)

	return result and result.Instance or nil
end

track(
	UserInputService.InputBegan:Connect(function(
		input,
		processed
	)
		if processed
			or not infRangeEnabled
			or interactionBusy then

			return
		end

		if input.UserInputType
				~= Enum.UserInputType.MouseButton1
			and input.UserInputType
				~= Enum.UserInputType.Touch then

			return
		end

		local target =
			getRayTarget()

		if not target then
			return
		end

		local pickup =
			getTrackedPickupRoot(
				target
			)

		if not pickup then
			return
		end

		task.spawn(function()
			pickupObject(
				pickup,
				{
					LockCamera = true,
					ClickCount = STRONG_CLICK_BURST,
					ClickGap = 0.02,
					FrontDistance = 4.5
				}
			)
		end)
	end)
)

--// UFO EVENT

local function startUfoEvent()
	local button =
		getNearest(ufoCache)

	if not button then
		notify(
			"UFO Event",
			"UfoButtenPart was not found.",
			3,
			"danger"
		)

		return
	end

	task.spawn(function()

		withInteractionLock(function()

			local character = getCharacter()
			local root = getRoot()
			local humanoid = getHumanoid()
			local part = getMainPart(button)

			if not character
				or not root
				or not part then

				return false
			end

			local oldPivot =
				character:GetPivot()

			local oldAnchored =
				root.Anchored

			local oldAutoRotate

			if humanoid then
				oldAutoRotate =
					humanoid.AutoRotate
			end

			pcall(function()
				root.Anchored = true
			end)

			if humanoid then
				pcall(function()
					humanoid.AutoRotate = false
				end)
			end

			moveCharacterNearPosition(
				part.Position,
				UFO_DISTANCE
			)

			task.wait(0.08)

			local success =
				clickObjectBurst(
					button,
					STRONG_CLICK_BURST,
					0.025
				)

			if character.Parent then
				pcall(function()
					character:PivotTo(
						oldPivot
					)
				end)

				stopVelocity()
			end

			if root.Parent then
				root.Anchored =
					oldAnchored
			end

			if humanoid
				and humanoid.Parent
				and oldAutoRotate ~= nil then

				humanoid.AutoRotate =
					oldAutoRotate
			end

			notify(
				"UFO Event",
				success
					and "UFO button clicked."
					or "Could not click UfoButtenPart.",
				2,
				success and nil or "danger"
			)

			return success
		end)
	end)
end

--// PLAYER MODS

local function setSpeed(state)
	speedEnabled = state

	if not state then
		for humanoid, oldSpeed in pairs(
			originalWalkSpeeds
		) do
			if humanoid and humanoid.Parent then
				humanoid.WalkSpeed =
					oldSpeed
			end
		end
	end
end

local function setThirdPerson(state)
	thirdPersonEnabled = state

	if not state then
		pcall(function()
			LocalPlayer.CameraMode =
				originalCameraMode

			LocalPlayer.CameraMinZoomDistance =
				originalMinZoom

			LocalPlayer.CameraMaxZoomDistance =
				originalMaxZoom
		end)
	end
end

local function clearFly()
	if flyConnection then
		pcall(function()
			flyConnection:Disconnect()
		end)

		flyConnection = nil
	end

	if flyVelocity then
		pcall(function()
			flyVelocity:Destroy()
		end)

		flyVelocity = nil
	end

	if flyGyro then
		pcall(function()
			flyGyro:Destroy()
		end)

		flyGyro = nil
	end

	if flyHumanoid
		and flyHumanoid.Parent then

		pcall(function()
			flyHumanoid.PlatformStand =
				flyOldPlatformStand == true
		end)
	end

	flyHumanoid = nil
	flyOldPlatformStand = nil
end

local function attachFly()
	clearFly()

	if not flyEnabled then
		return
	end

	local root = getRoot()
	local humanoid = getHumanoid()

	if not root or not humanoid then
		return
	end

	flyHumanoid = humanoid

	flyOldPlatformStand =
		humanoid.PlatformStand

	humanoid.PlatformStand = true

	flyVelocity =
		create("BodyVelocity", {
			Name = "THHFlyVelocity",

			MaxForce =
				Vector3.new(
					1e9,
					1e9,
					1e9
				),

			P = 1250,

			Velocity =
				Vector3.zero,

			Parent = root
		})

	flyGyro =
		create("BodyGyro", {
			Name = "THHFlyGyro",

			MaxTorque =
				Vector3.new(
					1e9,
					1e9,
					1e9
				),

			P = 5000,
			D = 100,

			CFrame =
				root.CFrame,

			Parent = root
		})

	flyConnection =
		RunService.RenderStepped:Connect(function()

			if not flyEnabled then
				return
			end

			if interactionBusy then
				flyVelocity.Velocity =
					Vector3.zero

				return
			end

			Camera =
				Workspace.CurrentCamera
				or Camera

			if not Camera then
				return
			end

			local movement =
				Vector3.zero

			local keyboard =
				false

			if UserInputService:IsKeyDown(
				Enum.KeyCode.W
			) then
				movement +=
					Camera.CFrame.LookVector

				keyboard = true
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.S
			) then
				movement -=
					Camera.CFrame.LookVector

				keyboard = true
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.A
			) then
				movement -=
					Camera.CFrame.RightVector

				keyboard = true
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.D
			) then
				movement +=
					Camera.CFrame.RightVector

				keyboard = true
			end

			if not keyboard
				and humanoid.MoveDirection.Magnitude
					> 0.05 then

				movement +=
					humanoid.MoveDirection
			end

			local vertical = 0

			if UserInputService:IsKeyDown(
				Enum.KeyCode.Space
			) then
				vertical += 1
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.LeftControl
			)
				or UserInputService:IsKeyDown(
					Enum.KeyCode.C
				) then

				vertical -= 1
			end

			if os.clock()
				< flyJumpUntil then

				vertical =
					math.max(
						vertical,
						1
					)
			end

			local velocity =
				Vector3.zero

			if movement.Magnitude > 0.01 then
				velocity +=
					movement.Unit
					* FLY_SPEED
			end

			velocity +=
				Vector3.new(
					0,
					vertical
						* FLY_SPEED,
					0
				)

			flyVelocity.Velocity =
				velocity

			local look =
				Vector3.new(
					Camera.CFrame.LookVector.X,
					0,
					Camera.CFrame.LookVector.Z
				)

			if look.Magnitude > 0.01 then
				flyGyro.CFrame =
					CFrame.lookAt(
						root.Position,
						root.Position
							+ look.Unit
					)
			end
		end)

	track(flyConnection)
end

local function setFly(state)
	flyEnabled = state

	if state then
		attachFly()
	else
		clearFly()
	end
end

track(
	UserInputService.JumpRequest:Connect(function()

		flyJumpUntil =
			os.clock() + 0.25

		if infiniteJumpEnabled then
			local humanoid =
				getHumanoid()

			if humanoid then
				pcall(function()
					humanoid:ChangeState(
						Enum.HumanoidStateType.Jumping
					)
				end)
			end
		end
	end)
)

track(
	RunService.Heartbeat:Connect(function()

		if speedEnabled
			and not interactionBusy then

			local humanoid =
				getHumanoid()

			if humanoid then
				if originalWalkSpeeds[humanoid]
					== nil then

					originalWalkSpeeds[humanoid] =
						humanoid.WalkSpeed
				end

				humanoid.WalkSpeed =
					SPEED_AMOUNT
			end
		end

		if thirdPersonEnabled then
			pcall(function()
				LocalPlayer.CameraMode =
					Enum.CameraMode.Classic

				LocalPlayer.CameraMinZoomDistance =
					0.5

				LocalPlayer.CameraMaxZoomDistance =
					128
			end)
		end
	end)
)

track(
	LocalPlayer.CharacterAdded:Connect(function()

		task.wait(0.5)

		if flyEnabled then
			attachFly()
		end
	end)
)

--// PART NAME HOVER

local function stopPartNameHover()
	partNameHoverEnabled = false

	if partNameHoverConnection then
		pcall(function()
			partNameHoverConnection:Disconnect()
		end)

		partNameHoverConnection = nil
	end

	if partNameHoverLabel then
		partNameHoverLabel:Destroy()
		partNameHoverLabel = nil
	end
end

local function startPartNameHover()
	if partNameHoverEnabled then
		return
	end

	partNameHoverEnabled = true

	partNameHoverLabel =
		create("TextLabel", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.04,

			BorderSizePixel = 0,

			Size =
				UDim2.fromOffset(
					230,
					34
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = "",

			TextColor3 =
				Colors.Text,

			TextSize = 12,

			Visible = false,

			ZIndex = 9999,

			Parent = screenGui
		})

	corner(
		partNameHoverLabel,
		8
	)

	partNameHoverConnection =
		RunService.RenderStepped:Connect(function()

			if not partNameHoverEnabled
				or not partNameHoverLabel
				or not partNameHoverLabel.Parent then

				return
			end

			local target =
				Mouse.Target

			if not target then
				partNameHoverLabel.Visible =
					false

				return
			end

			local position =
				UserInputService:GetMouseLocation()

			partNameHoverLabel.Text =
				target.Name

			partNameHoverLabel.Position =
				UDim2.fromOffset(
					position.X + 14,
					position.Y + 14
				)

			partNameHoverLabel.Visible =
				true
		end)

	track(partNameHoverConnection)
end

--// CLEANUP

local function cleanupAll(hardUnload)
	if cleaning then
		return
	end

	cleaning = true

	autoFarmEnabled = false
	autoHayEnabled = false
	autoSellEnabled = false

	autoDiamondEnabled = false
	autoColorHayEnabled = false

	autoFindNeedleEnabled = false
	autoFindKeyEnabled = false

	noPickupCooldownEnabled = false
	infRangeEnabled = false

	speedEnabled = false
	infiniteJumpEnabled = false

	setThirdPerson(false)
	setFly(false)

	stopPartNameHover()
	restoreInteractionSettings()

	antiAfkEnabled = false

	if antiAfkConnection then
		pcall(function()
			antiAfkConnection:Disconnect()
		end)

		antiAfkConnection = nil
	end

	for humanoid, speed in pairs(
		originalWalkSpeeds
	) do
		if humanoid and humanoid.Parent then
			pcall(function()
				humanoid.WalkSpeed = speed
			end)
		end
	end

	for _, connection in ipairs(connections) do
		pcall(function()
			connection:Disconnect()
		end)
	end

	table.clear(connections)

	if screenGui then
		pcall(function()
			screenGui:Destroy()
		end)

		screenGui = nil
	end

	if hardUnload then
		_G.THHGrowersHardUnloaded = true
	end

	_G.THHGrowersCleanup = nil
end

_G.THHGrowersCleanup = function()
	cleanupAll(false)
end

--// GUI PARENT

local guiParent

if type(gethui) == "function" then
	local success, result =
		pcall(gethui)

	if success and result then
		guiParent = result
	end
end

if not guiParent then
	guiParent =
		LocalPlayer:WaitForChild(
			"PlayerGui"
		)
end

screenGui =
	create("ScreenGui", {
		Name = "THHGrowers",

		ResetOnSpawn = false,

		IgnoreGuiInset = true,

		ZIndexBehavior =
			Enum.ZIndexBehavior.Sibling,

		DisplayOrder = 999999,

		Parent = guiParent
	})

--// NOTIFICATIONS

notificationHolder =
	create("Frame", {
	BackgroundTransparency = 1,

	AnchorPoint =
		Vector2.new(1, 0),

	Position =
		UDim2.new(
			1,
			-14,
			0,
			14
		),

	Size =
		UDim2.fromOffset(
			300,
			500
		),

	ZIndex = 500,

	Parent = screenGui
})

create("UIListLayout", {
	Padding =
		UDim.new(0, 7),

	HorizontalAlignment =
		Enum.HorizontalAlignment.Right,

	Parent =
		notificationHolder
})

notify =
	function(
		title,
		message,
		duration,
		kind
	)
		local card =
			create("Frame", {
				BackgroundColor3 =
					Colors.Card,

				Size =
					UDim2.fromOffset(
						290,
						72
					),

				ZIndex = 501,

				Parent =
					notificationHolder
			})

		corner(card, 9)

		stroke(
			card,
			kind == "danger"
				and Colors.Danger
				or Colors.Accent,
			0.3,
			1
		)

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					8
				),

			Size =
				UDim2.new(
					1,
					-24,
					0,
					20
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = title,

			TextColor3 =
				Colors.Text,

			TextSize = 13,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			ZIndex = 502,

			Parent = card
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					31
				),

			Size =
				UDim2.new(
					1,
					-24,
					0,
					30
				),

			Font =
				Enum.Font.Gotham,

			Text = message,

			TextColor3 =
				Colors.SubText,

			TextSize = 10,

			TextWrapped = true,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			ZIndex = 502,

			Parent = card
		})

		task.delay(
			duration or 2.5,
			function()

				if card and card.Parent then
					card:Destroy()
				end
			end
		)
	end

--// AUTH

local function getVampauthClient()
	if VampauthClient then
		return VampauthClient
	end

	if type(loadstring)
		~= "function" then

		return nil,
			"loadstring unavailable."
	end

	local ok, source =
		pcall(function()
			return game:HttpGet(
				VAMPAUTH_CLIENT_URL
			)
		end)

	if not ok
		or type(source) ~= "string" then

		return nil,
			"Could not load Vampauth."
	end

	local compileOk, chunk =
		pcall(
			loadstring,
			source
		)

	if not compileOk
		or type(chunk) ~= "function" then

		return nil,
			"Could not start Vampauth."
	end

	local moduleOk, Vampauth =
		pcall(chunk)

	if not moduleOk
		or type(Vampauth) ~= "table"
		or type(Vampauth.new)
			~= "function" then

		return nil,
			"Vampauth failed."
	end

	local clientOk, client =
		pcall(function()

			return Vampauth.new({
				projectId = PROJECT_ID,
				authSecret = AUTH_SECRET,
				debug = false
			})
		end)

	if not clientOk then
		return nil,
			"Could not initialize Vampauth."
	end

	VampauthClient = client

	return client
end

local function validateKey(key)
	local client, errorText =
		getVampauthClient()

	if not client then
		return false,
			errorText
	end

	local success, valid, result =
		pcall(function()

			local ok, data =
				client:Check(key)

			return ok, data
		end)

	if not success then
		return false,
			"Validation failed."
	end

	if valid then
		return true
	end

	if type(result) == "table" then
		return false,
			tostring(
				result.error
				or result.message
				or "Invalid key."
			)
	end

	return false,
		tostring(
			result
			or "Invalid key."
		)
end

--// DRAG

local function makeDraggable(frame, handle)
	local dragging = false
	local dragStart
	local startPosition

	handle.InputBegan:Connect(function(input)

		if input.UserInputType
				== Enum.UserInputType.MouseButton1
			or input.UserInputType
				== Enum.UserInputType.Touch then

			dragging = true
			dragStart = input.Position
			startPosition = frame.Position
		end
	end)

	UserInputService.InputChanged:Connect(function(input)

		if not dragging then
			return
		end

		if input.UserInputType
				~= Enum.UserInputType.MouseMovement
			and input.UserInputType
				~= Enum.UserInputType.Touch then

			return
		end

		local delta =
			input.Position
			- dragStart

		frame.Position =
			UDim2.new(
				startPosition.X.Scale,
				startPosition.X.Offset + delta.X,
				startPosition.Y.Scale,
				startPosition.Y.Offset + delta.Y
			)
	end)

	UserInputService.InputEnded:Connect(function(input)

		if input.UserInputType
				== Enum.UserInputType.MouseButton1
			or input.UserInputType
				== Enum.UserInputType.Touch then

			dragging = false
		end
	end)
end

--// MAIN GUI

local function buildMainMenu(mode)
	local phoneMode =
		mode == "PHONE"

	local width =
		phoneMode and 440 or 710

	local height =
		phoneMode and 620 or 470

	local sidebarWidth =
		phoneMode and 115 or 175

	mainFrame =
		create("Frame", {
			AnchorPoint =
				Vector2.new(
					0.5,
					0.5
				),

			Position =
				UDim2.fromScale(
					0.5,
					0.5
				),

			Size =
				UDim2.fromOffset(
					width,
					height
				),

			BackgroundColor3 =
				Colors.Background,

			BorderSizePixel = 0,

			ClipsDescendants = true,

			Parent = screenGui
		})

	corner(mainFrame, 12)

	stroke(
		mainFrame,
		Colors.Stroke,
		0,
		1
	)

	if phoneMode then
		mainScale =
			create("UIScale", {
				Scale = 1,
				Parent = mainFrame
			})

		local function updateScale()
			Camera =
				Workspace.CurrentCamera
				or Camera

			if not Camera then
				return
			end

			mainScale.Scale =
				math.clamp(
					math.min(
						(Camera.ViewportSize.X - 15)
							/ width,

						(Camera.ViewportSize.Y - 20)
							/ height,

						1
					),
					0.45,
					1
				)
		end

		updateScale()

		if Camera then
			track(
				Camera:GetPropertyChangedSignal(
					"ViewportSize"
				):Connect(updateScale)
			)
		end
	end

	local sidebar =
		create("Frame", {
			BackgroundColor3 =
				Colors.Sidebar,

			Size =
				UDim2.new(
					0,
					sidebarWidth,
					1,
					0
				),

			BorderSizePixel = 0,

			Parent = mainFrame
		})

	local icon =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.Card,

			Position =
				UDim2.fromOffset(
					12,
					12
				),

			Size =
				UDim2.fromOffset(
					42,
					42
				),

			Image = GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			Parent = sidebar
		})

	corner(icon, 9)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				62,
				12
			),

		Size =
			UDim2.new(
				1,
				-66,
				0,
				24
			),

		Font =
			Enum.Font.GothamBold,

		Text =
			phoneMode
			and "THH"
			or "THH HUB",

		TextColor3 =
			Colors.Text,

		TextSize = 14,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = sidebar
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				62,
				35
			),

		Size =
			UDim2.new(
				1,
				-66,
				0,
				18
			),

		Font =
			Enum.Font.Gotham,

		Text =
			phoneMode
			and "PHONE"
			or "PC",

		TextColor3 =
			Colors.SubText,

		TextSize = 9,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = sidebar
	})

	local navHolder =
		create("ScrollingFrame", {
			BackgroundTransparency = 1,

			BorderSizePixel = 0,

			Position =
				UDim2.fromOffset(
					8,
					70
				),

			Size =
				UDim2.new(
					1,
					-16,
					1,
					-80
				),

			CanvasSize =
				UDim2.new(),

			AutomaticCanvasSize =
				Enum.AutomaticSize.Y,

			ScrollBarThickness = 2,

			Parent = sidebar
		})

	create("UIListLayout", {
		Padding =
			UDim.new(
				0,
				6
			),

		Parent = navHolder
	})

	local content =
		create("Frame", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					sidebarWidth,
					0
				),

			Size =
				UDim2.new(
					1,
					-sidebarWidth,
					1,
					0
				),

			Parent = mainFrame
		})

	local header =
		create("Frame", {
			BackgroundTransparency = 1,

			Size =
				UDim2.new(
					1,
					0,
					0,
					58
				),

			Parent = content
		})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				16,
				9
			),

		Size =
			UDim2.new(
				1,
				-100,
				0,
				24
			),

		Font =
			Enum.Font.GothamBold,

		Text = "THH MENU",

		TextColor3 =
			Colors.Text,

		TextSize = 18,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = header
	})

	local pageTitle =
		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					16,
					33
				),

			Size =
				UDim2.new(
					1,
					-100,
					0,
					18
				),

			Font =
				Enum.Font.Gotham,

			Text = "Home",

			TextColor3 =
				Colors.SubText,

			TextSize = 10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = header
		})

	local close =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			AnchorPoint =
				Vector2.new(
					1,
					0
				),

			Position =
				UDim2.new(
					1,
					-12,
					0,
					11
				),

			Size =
				UDim2.fromOffset(
					34,
					34
				),

			Font =
				Enum.Font.GothamBold,

			Text = "—",

			TextColor3 =
				Colors.SubText,

			TextSize = 15,

			Parent = header
		})

	corner(close, 7)

	local pageHolder =
		create("Frame", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					0,
					58
				),

			Size =
				UDim2.new(
					1,
					0,
					1,
					-58
				),

			Parent = content
		})

	local pages = {}
	local navButtons = {}

	local function createPage(name)
		local page =
			create("ScrollingFrame", {
				BackgroundTransparency = 1,

				BorderSizePixel = 0,

				Size =
					UDim2.fromScale(
						1,
						1
					),

				CanvasSize =
					UDim2.new(),

				AutomaticCanvasSize =
					Enum.AutomaticSize.Y,

				ScrollBarThickness =
					phoneMode
					and 7
					or 4,

				Visible = false,

				Parent = pageHolder
			})

		padding(
			page,
			12,
			12,
			12,
			12
		)

		create("UIListLayout", {
			Padding =
				UDim.new(
					0,
					10
				),

			Parent = page
		})

		pages[name] = page

		return page
	end

	local function setPage(name)
		for pageName, page in pairs(pages) do
			page.Visible =
				pageName == name
		end

		for navName, button in pairs(navButtons) do
			local selected =
				navName == name

			button.BackgroundColor3 =
				selected
				and Colors.AccentDark
				or Colors.Sidebar

			button.TextColor3 =
				selected
				and Colors.Accent
				or Colors.SubText
		end

		pageTitle.Text = name
	end

	local function createNav(name)
		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Sidebar,

				BorderSizePixel = 0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						phoneMode
							and 48
							or 40
					),

				Font =
					Enum.Font.GothamSemibold,

				Text = name,

				TextColor3 =
					Colors.SubText,

				TextSize = 12,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				Parent = navHolder
			})

		padding(
			button,
			12,
			5,
			0,
			0
		)

		corner(button, 7)

		track(
			button.MouseButton1Click:Connect(function()
				setPage(name)
			end)
		)

		navButtons[name] = button
	end

	local function section(
		parent,
		title,
		description
	)
		local frame =
			create("Frame", {
				BackgroundColor3 =
					Colors.Card,

				Size =
					UDim2.new(
						1,
						0,
						0,
						0
					),

				AutomaticSize =
					Enum.AutomaticSize.Y,

				Parent = parent
			})

		corner(frame, 9)

		stroke(
			frame,
			Colors.Stroke,
			0.2,
			1
		)

		padding(
			frame,
			12,
			12,
			12,
			12
		)

		create("UIListLayout", {
			Padding =
				UDim.new(
					0,
					7
				),

			Parent = frame
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Size =
				UDim2.new(
					1,
					0,
					0,
					22
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = title,

			TextColor3 =
				Colors.Text,

			TextSize = 14,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = frame
		})

		if description then
			create("TextLabel", {
				BackgroundTransparency = 1,

				Size =
					UDim2.new(
						1,
						0,
						0,
						0
					),

				AutomaticSize =
					Enum.AutomaticSize.Y,

				Font =
					Enum.Font.Gotham,

				Text = description,

				TextColor3 =
					Colors.SubText,

				TextSize = 10,

				TextWrapped = true,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				Parent = frame
			})
		end

		return frame
	end

	local function createToggle(
		parent,
		title,
		description,
		callback
	)
		local enabled = false

		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Background,

				Size =
					UDim2.new(
						1,
						0,
						0,
						phoneMode
							and 60
							or 48
					),

				Text = "",

				Parent = parent
			})

		corner(button, 7)

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					7
				),

			Size =
				UDim2.new(
					1,
					-85,
					0,
					20
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = title,

			TextColor3 =
				Colors.Text,

			TextSize = 12,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = button
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					27
				),

			Size =
				UDim2.new(
					1,
					-90,
					0,
					18
				),

			Font =
				Enum.Font.Gotham,

			Text = description,

			TextColor3 =
				Colors.SubText,

			TextSize = 9,

			TextTruncate =
				Enum.TextTruncate.AtEnd,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = button
		})

		local switch =
			create("Frame", {
				BackgroundColor3 =
					Colors.Stroke,

				AnchorPoint =
					Vector2.new(
						1,
						0.5
					),

				Position =
					UDim2.new(
						1,
						-12,
						0.5,
						0
					),

				Size =
					UDim2.fromOffset(
						44,
						24
					),

				Parent = button
			})

		corner(switch, 12)

		local knob =
			create("Frame", {
				BackgroundColor3 =
					Colors.SubText,

				AnchorPoint =
					Vector2.new(
						0.5,
						0.5
					),

				Position =
					UDim2.new(
						0,
						12,
						0.5,
						0
					),

				Size =
					UDim2.fromOffset(
						18,
						18
					),

				Parent = switch
			})

		corner(knob, 9)

		track(
			button.MouseButton1Click:Connect(function()

				enabled =
					not enabled

				switch.BackgroundColor3 =
					enabled
						and Colors.AccentDark
						or Colors.Stroke

				knob.BackgroundColor3 =
					enabled
						and Colors.Accent
						or Colors.SubText

				knob.Position =
					enabled
						and UDim2.new(
							1,
							-12,
							0.5,
							0
						)
						or UDim2.new(
							0,
							12,
							0.5,
							0
						)

				callback(enabled)
			end)
		)
	end

	local function createAction(
		parent,
		title,
		description,
		callback,
		danger
	)
		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Background,

				Size =
					UDim2.new(
						1,
						0,
						0,
						48
					),

				Text = "",

				Parent = parent
			})

		corner(button, 7)

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					7
				),

			Size =
				UDim2.new(
					1,
					-24,
					0,
					20
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = title,

			TextColor3 =
				danger
				and Colors.Danger
				or Colors.Text,

			TextSize = 12,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = button
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					27
				),

			Size =
				UDim2.new(
					1,
					-24,
					0,
					16
				),

			Font =
				Enum.Font.Gotham,

			Text = description,

			TextColor3 =
				Colors.SubText,

			TextSize = 9,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = button
		})

		track(
			button.MouseButton1Click:Connect(
				callback
			)
		)
	end

	createNav("Home")
	createNav("Mods")
	createNav("Player")
	createNav("Event")
	createNav("Utility")
	createNav("Updates")

	local home =
		createPage("Home")

	local mods =
		createPage("Mods")

	local player =
		createPage("Player")

	local event =
		createPage("Event")

	local utility =
		createPage("Utility")

	local updates =
		createPage("Updates")

	section(
		home,
		"welcome to THH Growers",
		"optimized pickup queues + camera hidden teleports"
	)

	local farming =
		section(
			mods,
			"Farming",
			"optimized farming and selling"
		)

	createToggle(
		farming,
		"Auto Farm",
		"4 HayPieces → sell → repeat",
		function(state)

			if state then
				startAutoFarm()
			else
				stopAutoFarm()
			end
		end
	)

	createToggle(
		farming,
		"Auto Pick Up Hay",
		"automatic HayPiece pickup",
		function(state)

			if state then
				startAutoHay()
			else
				stopAutoHay()
			end
		end
	)

	createToggle(
		farming,
		"Auto Sell",
		"TP to SellPart + click + return",
		function(state)

			if state then
				startAutoSell()
			else
				stopAutoSell()
			end
		end
	)

	createToggle(
		farming,
		"No Pickup Cooldown",
		"reduces local THH pickup waits",
		function(state)

			noPickupCooldownEnabled =
				state

			refreshCachedSettings()
		end
	)

	createToggle(
		farming,
		"Inf Range",
		"far pickups with camera lock",
		function(state)

			infRangeEnabled =
				state

			refreshCachedSettings()
		end
	)

	local specials =
		section(
			mods,
			"Special Pickups",
			"strong click burst + hidden camera teleport"
		)

	createToggle(
		specials,
		"Auto Pick Up Diamond",
		"TP close → front → 8 clicks",
		function(state)

			if state then
				startAutoDiamond()
			else
				stopAutoDiamond()
			end
		end
	)

	createToggle(
		specials,
		"Auto Pick Up Color Hay",
		"color changes → TP close → front → 8 clicks",
		function(state)

			if state then
				startAutoColorHay()
			else
				stopAutoColorHay()
			end
		end
	)

	local movement =
		section(
			player,
			"Movement",
			"player movement mods"
		)

	createToggle(
		movement,
		"Speed",
		"WalkSpeed 50",
		function(state)
			setSpeed(state)
		end
	)

	createToggle(
		movement,
		"Infinite Jump",
		"jump while airborne",
		function(state)
			infiniteJumpEnabled =
				state
		end
	)

	createToggle(
		movement,
		"Unlock 3rd Person",
		"Classic camera zoom",
		function(state)
			setThirdPerson(state)
		end
	)

	createToggle(
		movement,
		"Fly",
		"WASD + Space/Ctrl",
		function(state)
			setFly(state)
		end
	)

	local events =
		section(
			event,
			"Events",
			"event automation"
		)

	createAction(
		events,
		"Start UFO Event",
		"TP to UfoButtenPart and click",
		startUfoEvent
	)

	createToggle(
		events,
		"Auto Find Needle",
		"auto interact when Needle appears",
		function(state)

			if state then
				startAutoFindNeedle()
			else
				stopAutoFindNeedle()
			end
		end
	)

	createToggle(
		events,
		"Auto Find Key",
		"auto interact when Key appears",
		function(state)

			if state then
				startAutoFindKey()
			else
				stopAutoFindKey()
			end
		end
	)

	local ids =
		section(
			event,
			"Place IDs",
			"Search For The Needle"
		)

	createAction(
		ids,
		"Copy Main Place ID",
		tostring(SEARCH_FOR_NEEDLE_PLACE_ID),
		function()
			copyText(
				SEARCH_FOR_NEEDLE_PLACE_ID
			)
		end
	)

	createAction(
		ids,
		"Copy Farmhouse ID",
		tostring(FARMHOUSE_PLACE_ID),
		function()
			copyText(
				FARMHOUSE_PLACE_ID
			)
		end
	)

	createAction(
		ids,
		"Copy Basement ID",
		tostring(BASEMENT_PLACE_ID),
		function()
			copyText(
				BASEMENT_PLACE_ID
			)
		end
	)

	local tools =
		section(
			utility,
			"Tools",
			"utility mods"
		)

	createToggle(
		tools,
		"Part Name Hover",
		"shows exact hovered part name",
		function(state)

			if state then
				startPartNameHover()
			else
				stopPartNameHover()
			end
		end
	)

	createToggle(
		tools,
		"Anti AFK",
		"prevents normal idle kicks",
		function(state)

			antiAfkEnabled =
				state

			if antiAfkConnection then
				antiAfkConnection:Disconnect()
				antiAfkConnection = nil
			end

			if state then
				antiAfkConnection =
					LocalPlayer.Idled:Connect(function()

						if not antiAfkEnabled then
							return
						end

						Camera =
							Workspace.CurrentCamera
							or Camera

						if not Camera then
							return
						end

						pcall(function()
							VirtualUser:CaptureController()

							VirtualUser:Button2Down(
								Vector2.zero,
								Camera.CFrame
							)

							task.wait(0.05)

							VirtualUser:Button2Up(
								Vector2.zero,
								Camera.CFrame
							)
						end)
					end)

				track(antiAfkConnection)
			end
		end
	)

	local server =
		section(
			utility,
			"Server",
			"server actions"
		)

	createAction(
		server,
		"Rejoin Server",
		"rejoin current server",
		function()

			if game.JobId ~= "" then
				TeleportService:TeleportToPlaceInstance(
					game.PlaceId,
					game.JobId,
					LocalPlayer
				)
			else
				TeleportService:Teleport(
					game.PlaceId,
					LocalPlayer
				)
			end
		end
	)

	createAction(
		server,
		"Reset Character",
		"reset character",
		function()

			local humanoid =
				getHumanoid()

			if humanoid then
				humanoid.Health = 0
			end
		end
	)

	createAction(
		server,
		"Copy Server ID",
		"copy current JobId",
		function()
			copyText(game.JobId)
		end
	)

	createAction(
		server,
		"Leave Server",
		"leave current server",
		function()
			LocalPlayer:Kick(
				"Left with THH HUB"
			)
		end,
		true
	)

	createAction(
		server,
		"Unload Menu",
		"turn off every mod",
		function()
			cleanupAll(true)
		end,
		true
	)

	section(
		updates,
		"09/09/26",
		"• Inf Range camera lock\n"
		.. "• Color Hay camera lock\n"
		.. "• stronger controlled click bursts\n"
		.. "• Auto Sell no camera lock\n"
		.. "• Auto Farm = 4 Hay then sell\n"
		.. "• object caches instead of repeated Workspace scans\n"
		.. "• interaction queue prevents TP fighting and spikes"
	)

	local function setVisible(value)
		mainFrame.Visible = value

		if phoneMode
			and floatingButton then

			floatingButton.Visible =
				not value
		end
	end

	track(
		close.MouseButton1Click:Connect(function()
			setVisible(false)
		end)
	)

	if phoneMode then
		floatingButton =
			create("ImageButton", {
				BackgroundColor3 =
					Colors.Card,

				AnchorPoint =
					Vector2.new(
						0.5,
						0
					),

				Position =
					UDim2.new(
						0.5,
						0,
						0,
						10
					),

				Size =
					UDim2.fromOffset(
						50,
						50
					),

				Image = GAME_ICON,

				ScaleType =
					Enum.ScaleType.Crop,

				Visible = false,

				ZIndex = 10000,

				Parent = screenGui
			})

		corner(
			floatingButton,
			14
		)

		track(
			floatingButton.MouseButton1Click:Connect(function()
				setVisible(true)
			end)
		)
	else
		makeDraggable(
			mainFrame,
			header
		)

		track(
			UserInputService.InputBegan:Connect(function(
				input,
				processed
			)
				if processed then
					return
				end

				if input.KeyCode
					== Enum.KeyCode.RightShift then

					setVisible(
						not mainFrame.Visible
					)
				end
			end)
		)
	end

	setPage("Home")

	notify(
		"THH HUB",
		phoneMode
			and "Phone mode loaded."
			or "PC mode loaded.",
		2
	)
end

--// DEVICE CHOOSER

local function showDeviceChooser()
	local overlay =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					5,
					6,
					8
				),

			BackgroundTransparency =
				0.3,

			Size =
				UDim2.fromScale(
					1,
					1
				),

			Parent = screenGui
		})

	local panel =
		create("Frame", {
			AnchorPoint =
				Vector2.new(
					0.5,
					0.5
				),

			Position =
				UDim2.fromScale(
					0.5,
					0.5
				),

			Size =
				UDim2.fromOffset(
					440,
					260
				),

			BackgroundColor3 =
				Glass.Panel,

			BackgroundTransparency =
				0.25,

			Parent = overlay
		})

	corner(panel, 14)

	stroke(
		panel,
		Glass.Stroke,
		0.65,
		1
	)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				20,
				17
			),

		Size =
			UDim2.new(
				1,
				-40,
				0,
				28
			),

		Font =
			Enum.Font.GothamBold,

		Text = "what are you on?",

		TextColor3 =
			Glass.Text,

		TextSize = 19,

		Parent = panel
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				20,
				47
			),

		Size =
			UDim2.new(
				1,
				-40,
				0,
				20
			),

		Font =
			Enum.Font.Gotham,

		Text = "pick pc or phone",

		TextColor3 =
			Glass.SubText,

		TextSize = 12,

		Parent = panel
	})

	local pc =
		create("TextButton", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.35,

			Position =
				UDim2.fromOffset(
					25,
					85
				),

			Size =
				UDim2.fromOffset(
					185,
					145
				),

			Font =
				Enum.Font.GothamBold,

			Text = "🖥\nPC",

			TextColor3 =
				Glass.Text,

			TextSize = 20,

			Parent = panel
		})

	corner(pc, 12)

	local phone =
		create("TextButton", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.35,

			Position =
				UDim2.fromOffset(
					230,
					85
				),

			Size =
				UDim2.fromOffset(
					185,
					145
				),

			Font =
				Enum.Font.GothamBold,

			Text = "▯\nPHONE",

			TextColor3 =
				Glass.Text,

			TextSize = 20,

			Parent = panel
		})

	corner(phone, 12)

	local chosen = false

	local function choose(mode)
		if chosen then
			return
		end

		chosen = true

		overlay:Destroy()

		buildMainMenu(mode)
	end

	track(
		pc.MouseButton1Click:Connect(function()
			choose("PC")
		end)
	)

	track(
		phone.MouseButton1Click:Connect(function()
			choose("PHONE")
		end)
	)
end

--// KEY GUI

local function showKeySystem()
	local overlay =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					7,
					8,
					10
				),

			BackgroundTransparency =
				0.3,

			Size =
				UDim2.fromScale(
					1,
					1
				),

			Parent = screenGui
		})

	local panel =
		create("Frame", {
			AnchorPoint =
				Vector2.new(
					0.5,
					0.5
				),

			Position =
				UDim2.fromScale(
					0.5,
					0.5
				),

			Size =
				UDim2.fromOffset(
					410,
					385
				),

			BackgroundColor3 =
				Glass.Panel,

			BackgroundTransparency =
				0.24,

			Parent = overlay
		})

	corner(panel, 17)

	stroke(
		panel,
		Glass.Stroke,
		0.68,
		1
	)

	local header =
		create("Frame", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.6,

			Position =
				UDim2.fromOffset(
					15,
					15
				),

			Size =
				UDim2.new(
					1,
					-30,
					0,
					94
				),

			Parent = panel
		})

	corner(header, 13)

	makeDraggable(
		panel,
		header
	)

	local icon =
		create("ImageLabel", {
			BackgroundColor3 =
				Glass.Dark,

			Position =
				UDim2.fromOffset(
					17,
					14
				),

			Size =
				UDim2.fromOffset(
					66,
					66
				),

			Image = GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			Parent = header
		})

	corner(icon, 13)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				100,
				18
			),

		Size =
			UDim2.new(
				1,
				-120,
				0,
				30
			),

		Font =
			Enum.Font.GothamBold,

		Text = "THH HUB",

		TextColor3 =
			Glass.Text,

		TextSize = 23,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = header
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				100,
				50
			),

		Size =
			UDim2.new(
				1,
				-120,
				0,
				22
			),

		Font =
			Enum.Font.Gotham,

		Text = "Authentication",

		TextColor3 =
			Glass.SubText,

		TextSize = 12,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = header
	})

	local keyBox =
		create("TextBox", {
			BackgroundColor3 =
				Glass.Dark,

			BackgroundTransparency =
				0.15,

			Position =
				UDim2.fromOffset(
					25,
					145
				),

			Size =
				UDim2.new(
					1,
					-50,
					0,
					52
				),

			ClearTextOnFocus =
				false,

			Font =
				Enum.Font.Gotham,

			PlaceholderText =
				"paste your key here...",

			Text = "",

			TextColor3 =
				Glass.Text,

			TextSize = 12,

			Parent = panel
		})

	corner(keyBox, 10)

	local continue =
		create("TextButton", {
			BackgroundColor3 =
				Color3.fromRGB(
					205,
					208,
					214
				),

			Position =
				UDim2.fromOffset(
					25,
					215
				),

			Size =
				UDim2.new(
					1,
					-50,
					0,
					48
				),

			Font =
				Enum.Font.GothamBold,

			Text = "Continue",

			TextColor3 =
				Color3.fromRGB(
					35,
					38,
					43
				),

			TextSize = 12,

			Parent = panel
		})

	corner(continue, 10)

	local getKey =
		create("TextButton", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.4,

			Position =
				UDim2.fromOffset(
					25,
					280
				),

			Size =
				UDim2.new(
					1,
					-50,
					0,
					46
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = "Get Access Key",

			TextColor3 =
				Glass.Text,

			TextSize = 12,

			Parent = panel
		})

	corner(getKey, 10)

	local status =
		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					25,
					340
				),

			Size =
				UDim2.new(
					1,
					-50,
					0,
					24
				),

			Font =
				Enum.Font.Gotham,

			Text =
				"enter your key to continue",

			TextColor3 =
				Glass.SubText,

			TextSize = 10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = panel
		})

	track(
		getKey.MouseButton1Click:Connect(function()

			if copyText(GET_KEY_URL) then
				status.Text =
					"Get Key link copied."

				status.TextColor3 =
					Colors.Accent
			end
		end)
	)

	local checking = false

	local function checkKey()
		if checking then
			return
		end

		local key =
			keyBox.Text:match(
				"^%s*(.-)%s*$"
			)

		if key == "" then
			status.Text =
				"Enter your key."

			status.TextColor3 =
				Colors.Danger

			return
		end

		checking = true

		continue.Text =
			"Checking..."

		task.spawn(function()

			local valid, result =
				validateKey(key)

			if valid then
				continue.Text =
					"Accepted"

				status.Text =
					"key accepted"

				status.TextColor3 =
					Colors.Accent

				task.wait(0.2)

				overlay:Destroy()

				showDeviceChooser()
			else
				checking = false

				continue.Text =
					"Continue"

				status.Text =
					result

				status.TextColor3 =
					Colors.Danger
			end
		end)
	end

	track(
		continue.MouseButton1Click:Connect(
			checkKey
		)
	)

	track(
		keyBox.FocusLost:Connect(function(
			enterPressed
		)
			if enterPressed then
				checkKey()
			end
		end)
	)
end

showKeySystem()
