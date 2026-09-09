if _G.THHGrowersHardUnloaded then
	return
end

if type(_G.THHGrowersCleanup) == "function" then
	pcall(_G.THHGrowersCleanup)
end

--==================================================
-- SERVICES
--==================================================

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

--==================================================
-- AUTH
--==================================================

local PROJECT_ID = "PD6XH2EGXZXUHGED"
local AUTH_SECRET = "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"

local GET_KEY_URL =
	"https://vampauth.com/PD6XH2EGXZXUHGED/flow"

local VAMPAUTH_CLIENT_URL =
	"https://vampauth.com/client/vampauth.lua"

--==================================================
-- PLACE IDS
--==================================================

local SEARCH_FOR_NEEDLE_PLACE_ID = 77108422251420
local FARMHOUSE_PLACE_ID = 108628039999641
local BASEMENT_PLACE_ID = 83445806734780

--==================================================
-- ICONS
--==================================================

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

--==================================================
-- LIGHT GRAY GLASS DESIGN
--==================================================

local Colors = {
	Main = Color3.fromRGB(66, 69, 76),
	Sidebar = Color3.fromRGB(54, 57, 64),

	Card = Color3.fromRGB(91, 94, 102),
	CardHover = Color3.fromRGB(105, 108, 116),
	CardDark = Color3.fromRGB(48, 51, 58),

	Input = Color3.fromRGB(50, 53, 60),

	Stroke = Color3.fromRGB(187, 190, 198),

	Text = Color3.fromRGB(249, 250, 252),
	SubText = Color3.fromRGB(207, 210, 217),
	Muted = Color3.fromRGB(168, 172, 181),

	Accent = Color3.fromRGB(97, 230, 147),
	AccentDark = Color3.fromRGB(49, 98, 67),

	Danger = Color3.fromRGB(234, 91, 91),

	White = Color3.fromRGB(255, 255, 255)
}

--==================================================
-- CONFIG
--==================================================

local SPEED_AMOUNT = 50
local FLY_SPEED = 58

local PICKUP_DISTANCE = 3
local SELL_DISTANCE = 3
local UFO_DISTANCE = 3

local NORMAL_CLICK_BURST = 6
local STRONG_CLICK_BURST = 10
local NEEDLE_CLICK_BURST = 12
local KEY_CLICK_BURST = 12
local SELL_CLICK_BURST = 6

local CLICK_GAP = 0.025

local FARM_PICKUPS_PER_SELL = 4
local AUTO_SELL_INTERVAL = 2

local NEEDLE_RETRY_DELAY = 0.4
local KEY_RETRY_DELAY = 0.4

--==================================================
-- STATE
--==================================================

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

--==================================================
-- CACHES
--==================================================

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

local diamondQueue = {}
local colorHayQueue = {}
local needleQueue = {}
local keyQueue = {}

local diamondQueued =
	setmetatable({}, {
		__mode = "k"
	})

local colorHayQueued =
	setmetatable({}, {
		__mode = "k"
	})

local needleQueued =
	setmetatable({}, {
		__mode = "k"
	})

local keyQueued =
	setmetatable({}, {
		__mode = "k"
	})

local needleLastAttempt =
	setmetatable({}, {
		__mode = "k"
	})

local keyLastAttempt =
	setmetatable({}, {
		__mode = "k"
	})

local notify = function() end

--==================================================
-- UI HELPERS
--==================================================

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
		CornerRadius = UDim.new(0, radius or 10),
		Parent = object
	})
end

local function stroke(object, color, transparency, thickness)
	return create("UIStroke", {
		Color = color or Colors.Stroke,
		Transparency = transparency or 0.45,
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

local function addGlassGradient(object)
	create("UIGradient", {
		Rotation = 90,

		Color = ColorSequence.new({
			ColorSequenceKeypoint.new(
				0,
				Color3.fromRGB(112, 115, 123)
			),

			ColorSequenceKeypoint.new(
				1,
				Color3.fromRGB(67, 70, 77)
			)
		}),

		Parent = object
	})
end

local function tween(object, duration, properties)
	local animation =
		TweenService:Create(
			object,
			TweenInfo.new(
				duration or 0.15,
				Enum.EasingStyle.Quad,
				Enum.EasingDirection.Out
			),
			properties
		)

	animation:Play()

	return animation
end

local function copyText(text)
	text = tostring(text)

	if type(setclipboard) == "function" then
		return pcall(setclipboard, text)
	end

	if type(toclipboard) == "function" then
		return pcall(toclipboard, text)
	end

	return false
end

--==================================================
-- CHARACTER HELPERS
--==================================================

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

local function getPrompt(object)
	if not object then
		return nil
	end

	if object:IsA("ProximityPrompt") then
		return object
	end

	return object:FindFirstChildWhichIsA(
		"ProximityPrompt",
		true
	)
end

local function getPartFromObject(object)
	local current = object

	while current
		and current ~= Workspace do

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

	return (
		root.Position
		- part.Position
	).Magnitude
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

--==================================================
-- CAMERA LOCK
--==================================================

local function lockCamera()
	Camera =
		Workspace.CurrentCamera
		or Camera

	if not Camera then
		return function() end
	end

	local oldType = Camera.CameraType
	local oldSubject = Camera.CameraSubject
	local oldCFrame = Camera.CFrame
	local oldFocus = Camera.Focus

	Camera.CameraType =
		Enum.CameraType.Scriptable

	Camera.CFrame = oldCFrame
	Camera.Focus = oldFocus

	local cameraConnection

	cameraConnection =
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

		if cameraConnection then
			pcall(function()
				cameraConnection:Disconnect()
			end)
		end

		Camera =
			Workspace.CurrentCamera
			or Camera

		if not Camera then
			return
		end

		pcall(function()
			Camera.CFrame = oldCFrame
			Camera.Focus = oldFocus
			Camera.CameraSubject = oldSubject
			Camera.CameraType = oldType
		end)
	end
end

--==================================================
-- RANGE / COOLDOWN
--==================================================

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

		Enabled = prompt.Enabled
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
	for detector in pairs(
		originalClickSettings
	) do
		if detector and detector.Parent then
			applyClickDetector(detector)
		end
	end

	for prompt in pairs(
		originalPromptSettings
	) do
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

--==================================================
-- NAME DETECTION
--==================================================

local function normalizeName(name)
	name =
		string.lower(
			tostring(name or "")
		)

	name =
		string.gsub(
			name,
			"[%s_%-%.]",
			""
		)

	return name
end

local function isNeedleName(name)
	local lower =
		normalizeName(name)

	if lower == "" then
		return false
	end

	local exact = {
		needle = true,
		needlepart = true,
		needlemodel = true,
		needlepickup = true,
		needlehandle = true,

		hayneedle = true,
		hayneedlepart = true,
		hayneedlemodel = true,

		haystackneedle = true,
		haystackneedlepart = true,

		hiddenneedle = true,
		hiddenneedlepart = true,

		goldneedle = true,
		goldenneedle = true,

		secretneedle = true,
		smallneedle = true,

		farmneedle = true,
		basementneedle = true
	}

	if exact[lower] then
		return true
	end

	return string.find(
		lower,
		"needle",
		1,
		true
	) ~= nil
end

local function isBadKeyName(name)
	local lower =
		normalizeName(name)

	local bad = {
		"keyboard",
		"keypad",
		"keyhole",
		"keybind",
		"keycode",
		"keyframe",
		"monkey",
		"donkey",
		"turkey",
		"hockey"
	}

	for _, word in ipairs(bad) do
		if string.find(
			lower,
			word,
			1,
			true
		) then
			return true
		end
	end

	return false
end

local function isKeyName(name)
	if isBadKeyName(name) then
		return false
	end

	local lower =
		normalizeName(name)

	local exact = {
		key = true,
		keypart = true,
		keyhandle = true,
		keymodel = true,
		keypickup = true,

		basementkey = true,
		basementkeypart = true,

		haykey = true,
		haystackkey = true,

		hiddenkey = true,
		exitkey = true,

		chapter2key = true,
		rustykey = true,
		oldkey = true
	}

	if exact[lower] then
		return true
	end

	return string.find(
		lower,
		"key",
		1,
		true
	) ~= nil
end

local function hasNeedleAttribute(object)
	if not object then
		return false
	end

	local success, attributes =
		pcall(function()
			return object:GetAttributes()
		end)

	if not success then
		return false
	end

	for name, value in pairs(attributes) do
		if isNeedleName(name) then
			return true
		end

		if type(value) == "string"
			and isNeedleName(value) then

			return true
		end
	end

	return false
end

local function hasKeyAttribute(object)
	if not object then
		return false
	end

	local success, attributes =
		pcall(function()
			return object:GetAttributes()
		end)

	if not success then
		return false
	end

	for name, value in pairs(attributes) do
		if isKeyName(name) then
			return true
		end

		if type(value) == "string"
			and isKeyName(value) then

			return true
		end
	end

	return false
end

local function findNeedleAncestor(object)
	local current = object

	if current
		and (
			current:IsA("ClickDetector")
			or current:IsA("ProximityPrompt")
		) then

		current = current.Parent
	end

	while current
		and current ~= Workspace do

		if isNeedleName(current.Name)
			or hasNeedleAttribute(current) then

			return current
		end

		current = current.Parent
	end

	return nil
end

local function findKeyAncestor(object)
	local current = object

	if current
		and (
			current:IsA("ClickDetector")
			or current:IsA("ProximityPrompt")
		) then

		current = current.Parent
	end

	while current
		and current ~= Workspace do

		if isKeyName(current.Name)
			or hasKeyAttribute(current) then

			return current
		end

		current = current.Parent
	end

	return nil
end

local function resolveNeedleRoot(object)
	if not object then
		return nil
	end

	local matched =
		findNeedleAncestor(object)
		or object

	local tool =
		matched:FindFirstAncestorOfClass(
			"Tool"
		)

	if tool
		and (
			isNeedleName(tool.Name)
			or isNeedleName(matched.Name)
		) then

		return tool
	end

	local current = matched

	while current
		and current ~= Workspace do

		if current:IsA("Model")
			and (
				isNeedleName(current.Name)
				or hasNeedleAttribute(current)
			) then

			return current
		end

		current = current.Parent
	end

	return matched
end

local function resolveKeyRoot(object)
	if not object then
		return nil
	end

	local matched =
		findKeyAncestor(object)
		or object

	local tool =
		matched:FindFirstAncestorOfClass(
			"Tool"
		)

	if tool
		and (
			isKeyName(tool.Name)
			or isKeyName(matched.Name)
		) then

		return tool
	end

	local current = matched

	while current
		and current ~= Workspace do

		if current:IsA("Model")
			and (
				isKeyName(current.Name)
				or hasKeyAttribute(current)
			) then

			return current
		end

		current = current.Parent
	end

	return matched
end

--==================================================
-- QUEUES
--==================================================

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

	table.insert(
		queue,
		object
	)
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

--==================================================
-- COLOR HAY
--==================================================

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
				return
			end

			local old = state.LastColor
			local new = part.Color

			local difference =
				math.abs(old.R - new.R)
				+ math.abs(old.G - new.G)
				+ math.abs(old.B - new.B)

			state.LastColor = new

			if difference > 0.015 then
				state.LastChanged = os.clock()

				if autoColorHayEnabled then
					queueColorHay(hay)
				end
			end
		end)

	colorHayConnections[hay] =
		connection

	track(connection)
end

--==================================================
-- REGISTER OBJECTS
--==================================================

local function registerNeedleCandidate(object)
	local matched =
		findNeedleAncestor(object)

	if not matched
		and (
			isNeedleName(object.Name)
			or hasNeedleAttribute(object)
		) then

		matched = object
	end

	if not matched then
		return
	end

	local needle =
		resolveNeedleRoot(matched)

	if not needle
		or not needle.Parent
		or needleCache[needle] then

		return
	end

	needleCache[needle] = true

	applySettingsToObject(needle)

	if autoFindNeedleEnabled then
		queueNeedle(needle)
	end
end

local function registerKeyCandidate(object)
	local matched =
		findKeyAncestor(object)

	if not matched
		and (
			isKeyName(object.Name)
			or hasKeyAttribute(object)
		) then

		matched = object
	end

	if not matched then
		return
	end

	local key =
		resolveKeyRoot(matched)

	if not key
		or not key.Parent
		or keyCache[key] then

		return
	end

	keyCache[key] = true

	applySettingsToObject(key)

	if autoFindKeyEnabled then
		queueKey(key)
	end
end

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

	registerNeedleCandidate(object)
	registerKeyCandidate(object)
end

local function unregisterObject(object)
	hayCache[object] = nil
	diamondCache[object] = nil
	sellCache[object] = nil
	ufoCache[object] = nil
	needleCache[object] = nil
	keyCache[object] = nil

	diamondQueued[object] = nil
	colorHayQueued[object] = nil
	needleQueued[object] = nil
	keyQueued[object] = nil

	colorHayState[object] = nil
end

for _, object in ipairs(
	Workspace:GetDescendants()
) do
	registerObject(object)
end

track(
	Workspace.DescendantAdded:Connect(function(object)

		registerObject(object)

		task.defer(function()
			if not object
				or not object.Parent then

				return
			end

			registerNeedleCandidate(object)
			registerKeyCandidate(object)

			if object.Parent then
				registerNeedleCandidate(
					object.Parent
				)

				registerKeyCandidate(
					object.Parent
				)
			end
		end)
	end)
)

track(
	Workspace.DescendantRemoving:Connect(function(object)
		unregisterObject(object)
	end)
)

--==================================================
-- MOVEMENT
--==================================================

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
		distance
		or PICKUP_DISTANCE

	local direction =
		Vector3.new(
			root.Position.X
				- position.X,

			0,

			root.Position.Z
				- position.Z
		)

	if direction.Magnitude < 0.1 then
		direction =
			Vector3.new(0, 0, 1)
	end

	direction = direction.Unit

	local target =
		position
		+ direction * distance

	target =
		Vector3.new(
			target.X,
			position.Y + 2.7,
			target.Z
		)

	pcall(function()
		character:PivotTo(
			CFrame.lookAt(
				target,

				Vector3.new(
					position.X,
					target.Y,
					position.Z
				)
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

	distance =
		distance or 4.5

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

	return false
end

--==================================================
-- CLICKING
--==================================================

local function directClick(detector)
	if not detector
		or not detector.Parent then

		return false
	end

	applyClickDetector(detector)

	if type(fireclickdetector)
		== "function" then

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

	count =
		count or NORMAL_CLICK_BURST

	gap =
		gap or CLICK_GAP

	local success = false

	for _ = 1, count do
		if not object.Parent then
			return true
		end

		local detector =
			getClickDetector(object)

		local clicked = false

		if detector then
			clicked =
				directClick(detector)
		end

		if not clicked then
			local part =
				getMainPart(object)

			if part then
				clicked =
					screenClickPart(part)
			end
		end

		if clicked then
			success = true
		end

		if noPickupCooldownEnabled then
			RunService.Heartbeat:Wait()
		else
			task.wait(gap)
		end
	end

	return success
end

--==================================================
-- PICKUP
--==================================================

local function pickupObject(
	object,
	options
)
	options = options or {}

	if not object
		or not object.Parent then

		return false
	end

	local previous =
		recentAttempts[object]

	if previous
		and os.clock() - previous < 0.08 then

		return false
	end

	recentAttempts[object] =
		os.clock()

	return withInteractionLock(function()

		local character = getCharacter()
		local root = getRoot()

		local part =
			options.InteractionPart
			or getMainPart(object)

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

		local unlockCamera

		if options.LockCamera ~= false then
			unlockCamera =
				lockCamera()
		else
			unlockCamera =
				function() end
		end

		moveCharacterNearPosition(
			part.Position,

			options.Distance
			or PICKUP_DISTANCE
		)

		RunService.Heartbeat:Wait()
		RunService.RenderStepped:Wait()

		if not object.Parent then
			pcall(function()
				character:PivotTo(
					oldCharacterPivot
				)
			end)

			unlockCamera()
			return true
		end

		moveObjectInFront(
			object,

			options.FrontDistance
			or 4.5
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

		if object
			and object.Parent then

			setObjectPivot(
				object,
				oldObjectPivot
			)
		end

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

--==================================================
-- CACHE HELPERS
--==================================================

local function getCachedList(cache)
	local list = {}

	for object in pairs(cache) do
		if object
			and object.Parent then

			table.insert(
				list,
				object
			)
		end
	end

	return list
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

--==================================================
-- SELL
--==================================================

local function getSellSearchObjects(sellPart)
	local result = {}
	local seen = {}

	local function add(object)
		if not object
			or seen[object] then

			return
		end

		seen[object] = true

		table.insert(
			result,
			object
		)
	end

	add(sellPart)

	for _, object in ipairs(
		sellPart:GetDescendants()
	) do
		add(object)
	end

	local parent = sellPart.Parent

	if parent
		and parent ~= Workspace then

		add(parent)

		for _, object in ipairs(
			parent:GetDescendants()
		) do
			add(object)
		end
	end

	return result
end

local function scanSellPart(sellPart)
	local result = {
		InteractionPart =
			getMainPart(sellPart),

		ClickDetector = nil
	}

	local main =
		getMainPart(sellPart)

	local bestScore = math.huge

	for _, object in ipairs(
		getSellSearchObjects(sellPart)
	) do
		if object:IsA("ClickDetector") then
			local part =
				getPartFromObject(object)

			local score = 20

			if part == main then
				score = 0

			elseif part
				and part.Name == "SellPart" then

				score = 1

			elseif object.Parent == sellPart then
				score = 2
			end

			if score < bestScore then
				bestScore = score

				result.ClickDetector =
					object

				if part then
					result.InteractionPart =
						part
				end
			end
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

		local character = getCharacter()
		local root = getRoot()
		local humanoid = getHumanoid()

		if not scan
			or not character
			or not root
			or not scan.InteractionPart then

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
			humanoid.AutoRotate = false
		end

		-- Auto Sell does NOT lock camera.

		moveCharacterNearPosition(
			scan.InteractionPart.Position,
			SELL_DISTANCE
		)

		task.wait(0.1)

		local clicked = false

		if scan.ClickDetector then
			for _ = 1, SELL_CLICK_BURST do
				if directClick(
					scan.ClickDetector
				) then

					clicked = true
				end

				task.wait(0.03)
			end
		end

		if not clicked then
			for _ = 1, SELL_CLICK_BURST do
				if screenClickPart(
					scan.InteractionPart
				) then

					clicked = true
				end

				task.wait(0.03)
			end
		end

		if character.Parent then
			character:PivotTo(
				oldPivot
			)

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

		return clicked
	end)
end

local function sellNow()
	local list =
		getCachedList(
			sellCache
		)

	for _, sellPart in ipairs(list) do
		if sellOnePart(sellPart) then
			return true
		end
	end

	return false
end

--==================================================
-- AUTO FARM
--==================================================

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
			local used = {}

			while autoFarmEnabled
				and collected < FARM_PICKUPS_PER_SELL do

				local list =
					getCachedList(
						hayCache
					)

				table.sort(
					list,
					function(a, b)
						return getDistanceTo(a)
							< getDistanceTo(b)
					end
				)

				local target

				for _, hay in ipairs(list) do
					if hay.Parent
						and not used[hay] then

						target = hay
						break
					end
				end

				if not target then
					break
				end

				used[target] = true

				pickupObject(
					target,
					{
						LockCamera = true,
						ClickCount = NORMAL_CLICK_BURST
					}
				)

				collected += 1

				task.wait(0.04)
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

			if hay then
				pickupObject(
					hay,
					{
						LockCamera = true
					}
				)
			end

			task.wait(0.05)
		end
	end)
end

local function stopAutoHay()
	autoHayEnabled = false
end

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

			task.wait(
				AUTO_SELL_INTERVAL
			)
		end
	end)
end

local function stopAutoSell()
	autoSellEnabled = false
end

--==================================================
-- DIAMOND
--==================================================

local function startAutoDiamond()
	if autoDiamondEnabled then
		return
	end

	autoDiamondEnabled = true

	for object in pairs(
		diamondCache
	) do
		queueDiamond(object)
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
							FrontDistance = 4
						}
					)
				end
			else
				task.wait(0.08)
			end
		end
	end)
end

local function stopAutoDiamond()
	autoDiamondEnabled = false

	table.clear(
		diamondQueue
	)
end

--==================================================
-- COLOR HAY
--==================================================

local function startAutoColorHay()
	if autoColorHayEnabled then
		return
	end

	autoColorHayEnabled = true

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
					pickupObject(
						hay,
						{
							LockCamera = true,
							ClickCount = STRONG_CLICK_BURST,
							FrontDistance = 4,
							ClickGap = 0.02
						}
					)
				end
			else
				task.wait(0.05)
			end
		end
	end)
end

local function stopAutoColorHay()
	autoColorHayEnabled = false

	table.clear(
		colorHayQueue
	)
end

--==================================================
-- NEEDLE
--==================================================

local function startAutoFindNeedle()
	if autoFindNeedleEnabled then
		return
	end

	autoFindNeedleEnabled = true

	for _, object in ipairs(
		Workspace:GetDescendants()
	) do
		registerNeedleCandidate(object)
	end

	for needle in pairs(
		needleCache
	) do
		queueNeedle(needle)
	end

	task.spawn(function()

		while autoFindNeedleEnabled
			and screenGui
			and screenGui.Parent do

			local needle =
				table.remove(
					needleQueue,
					1
				)

			if needle then
				needleQueued[needle] = nil

				if needle.Parent then
					local previous =
						needleLastAttempt[
							needle
						]

					if not previous
						or os.clock() - previous
							>= NEEDLE_RETRY_DELAY then

						needleLastAttempt[needle] =
							os.clock()

						pickupObject(
							resolveNeedleRoot(
								needle
							) or needle,

							{
								LockCamera = true,
								ClickCount = NEEDLE_CLICK_BURST,
								FrontDistance = 4,
								Distance = 2.5
							}
						)
					end
				end
			else
				task.wait(0.08)
			end
		end
	end)
end

local function stopAutoFindNeedle()
	autoFindNeedleEnabled = false

	table.clear(
		needleQueue
	)
end

--==================================================
-- KEY
--==================================================

local function startAutoFindKey()
	if autoFindKeyEnabled then
		return
	end

	autoFindKeyEnabled = true

	for _, object in ipairs(
		Workspace:GetDescendants()
	) do
		registerKeyCandidate(object)
	end

	for key in pairs(
		keyCache
	) do
		queueKey(key)
	end

	task.spawn(function()

		while autoFindKeyEnabled
			and screenGui
			and screenGui.Parent do

			local key =
				table.remove(
					keyQueue,
					1
				)

			if key then
				keyQueued[key] = nil

				if key.Parent then
					local previous =
						keyLastAttempt[
							key
						]

					if not previous
						or os.clock() - previous
							>= KEY_RETRY_DELAY then

						keyLastAttempt[key] =
							os.clock()

						pickupObject(
							resolveKeyRoot(key)
								or key,

							{
								LockCamera = true,
								ClickCount = KEY_CLICK_BURST,
								FrontDistance = 4,
								Distance = 2.5
							}
						)
					end
				end
			else
				task.wait(0.08)
			end
		end
	end)
end

local function stopAutoFindKey()
	autoFindKeyEnabled = false

	table.clear(
		keyQueue
	)
end

--==================================================
-- UFO
--==================================================

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

		local character =
			getCharacter()

		local root =
			getRoot()

		if not character
			or not root then

			return
		end

		local oldPivot =
			character:GetPivot()

		moveCharacterNearPosition(
			getMainPart(button).Position,
			UFO_DISTANCE
		)

		task.wait(0.08)

		clickObjectBurst(
			button,
			STRONG_CLICK_BURST,
			0.025
		)

		if character.Parent then
			character:PivotTo(
				oldPivot
			)

			stopVelocity()
		end
	end)
end

--==================================================
-- PLAYER MODS
--==================================================

local function setSpeed(state)
	speedEnabled = state

	if not state then
		for humanoid, oldSpeed in pairs(
			originalWalkSpeeds
		) do
			if humanoid.Parent then
				humanoid.WalkSpeed =
					oldSpeed
			end
		end
	end
end

local function setThirdPerson(state)
	thirdPersonEnabled = state

	if not state then
		LocalPlayer.CameraMode =
			originalCameraMode

		LocalPlayer.CameraMinZoomDistance =
			originalMinZoom

		LocalPlayer.CameraMaxZoomDistance =
			originalMaxZoom
	end
end

local function clearFly()
	if flyConnection then
		flyConnection:Disconnect()
		flyConnection = nil
	end

	if flyVelocity then
		flyVelocity:Destroy()
		flyVelocity = nil
	end

	if flyGyro then
		flyGyro:Destroy()
		flyGyro = nil
	end

	if flyHumanoid
		and flyHumanoid.Parent then

		flyHumanoid.PlatformStand =
			flyOldPlatformStand == true
	end
end

local function attachFly()
	clearFly()

	if not flyEnabled then
		return
	end

	local root = getRoot()
	local humanoid = getHumanoid()

	if not root
		or not humanoid then

		return
	end

	flyHumanoid = humanoid
	flyOldPlatformStand =
		humanoid.PlatformStand

	humanoid.PlatformStand = true

	flyVelocity =
		create("BodyVelocity", {
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

			if UserInputService:IsKeyDown(
				Enum.KeyCode.W
			) then
				movement +=
					Camera.CFrame.LookVector
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.S
			) then
				movement -=
					Camera.CFrame.LookVector
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.A
			) then
				movement -=
					Camera.CFrame.RightVector
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.D
			) then
				movement +=
					Camera.CFrame.RightVector
			end

			if movement.Magnitude < 0.05 then
				movement =
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
				Vector3.new(
					0,
					vertical * FLY_SPEED,
					0
				)

			if movement.Magnitude > 0.05 then
				velocity +=
					movement.Unit
					* FLY_SPEED
			end

			flyVelocity.Velocity =
				velocity

			flyGyro.CFrame =
				CFrame.lookAt(
					root.Position,

					root.Position
					+ Vector3.new(
						Camera.CFrame.LookVector.X,
						0,
						Camera.CFrame.LookVector.Z
					)
				)
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
				humanoid:ChangeState(
					Enum.HumanoidStateType.Jumping
				)
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
				if originalWalkSpeeds[
					humanoid
				] == nil then

					originalWalkSpeeds[
						humanoid
					] =
						humanoid.WalkSpeed
				end

				humanoid.WalkSpeed =
					SPEED_AMOUNT
			end
		end

		if thirdPersonEnabled then
			LocalPlayer.CameraMode =
				Enum.CameraMode.Classic

			LocalPlayer.CameraMinZoomDistance =
				0.5

			LocalPlayer.CameraMaxZoomDistance =
				128
		end
	end)
)

--==================================================
-- PART NAME HOVER
--==================================================

local function stopPartNameHover()
	partNameHoverEnabled = false

	if partNameHoverConnection then
		partNameHoverConnection:Disconnect()
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
				0.18,

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
		9
	)

	stroke(
		partNameHoverLabel,
		Colors.Stroke,
		0.55,
		1
	)

	partNameHoverConnection =
		RunService.RenderStepped:Connect(function()

			local target =
				Mouse.Target

			if not target then
				partNameHoverLabel.Visible =
					false

				return
			end

			local mousePosition =
				UserInputService:GetMouseLocation()

			partNameHoverLabel.Text =
				target.Name

			partNameHoverLabel.Position =
				UDim2.fromOffset(
					mousePosition.X + 14,
					mousePosition.Y + 14
				)

			partNameHoverLabel.Visible =
				true
		end)

	track(
		partNameHoverConnection
	)
end

--==================================================
-- CLEANUP
--==================================================

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

	if antiAfkConnection then
		antiAfkConnection:Disconnect()
		antiAfkConnection = nil
	end

	for humanoid, speed in pairs(
		originalWalkSpeeds
	) do
		if humanoid.Parent then
			humanoid.WalkSpeed =
				speed
		end
	end

	for _, connection in ipairs(
		connections
	) do
		pcall(function()
			connection:Disconnect()
		end)
	end

	table.clear(connections)

	if screenGui then
		screenGui:Destroy()
		screenGui = nil
	end

	if hardUnload then
		_G.THHGrowersHardUnloaded =
			true
	end

	_G.THHGrowersCleanup = nil
end

_G.THHGrowersCleanup = function()
	cleanupAll(false)
end

--==================================================
-- GUI ROOT
--==================================================

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

		DisplayOrder = 999999,

		ZIndexBehavior =
			Enum.ZIndexBehavior.Sibling,

		Parent = guiParent
	})

--==================================================
-- NOTIFICATIONS
--==================================================

notificationHolder =
	create("Frame", {
	BackgroundTransparency = 1,

	AnchorPoint =
		Vector2.new(1, 0),

	Position =
		UDim2.new(
			1,
			-16,
			0,
			16
		),

	Size =
		UDim2.fromOffset(
			310,
			500
		),

	ZIndex = 800,

	Parent = screenGui
})

create("UIListLayout", {
	Padding =
		UDim.new(
			0,
			8
		),

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

				BackgroundTransparency =
					0.12,

				Size =
					UDim2.fromOffset(
						300,
						76
					),

				ZIndex = 801,

				Parent =
					notificationHolder
			})

		corner(card, 12)

		stroke(
			card,

			kind == "danger"
				and Colors.Danger
				or Colors.Stroke,

			0.45,
			1
		)

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					14,
					10
				),

			Size =
				UDim2.new(
					1,
					-28,
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

			ZIndex = 802,

			Parent = card
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					14,
					34
				),

			Size =
				UDim2.new(
					1,
					-28,
					0,
					28
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

			ZIndex = 802,

			Parent = card
		})

		task.delay(
			duration or 2.5,
			function()

				if card.Parent then
					card:Destroy()
				end
			end
		)
	end

--==================================================
-- AUTH FUNCTIONS
--==================================================

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
		or type(source)
			~= "string" then

		return nil,
			"Could not load Vampauth."
	end

	local compileOk, chunk =
		pcall(
			loadstring,
			source
		)

	if not compileOk
		or type(chunk)
			~= "function" then

		return nil,
			"Could not start Vampauth."
	end

	local moduleOk, Vampauth =
		pcall(chunk)

	if not moduleOk
		or type(Vampauth)
			~= "table"
		or type(Vampauth.new)
			~= "function" then

		return nil,
			"Vampauth failed."
	end

	local clientOk, client =
		pcall(function()

			return Vampauth.new({
				projectId =
					PROJECT_ID,

				authSecret =
					AUTH_SECRET,

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

	return false,
		type(result) == "table"
			and tostring(
				result.error
				or result.message
				or "Invalid key."
			)
			or tostring(
				result
				or "Invalid key."
			)
end

--==================================================
-- DRAG
--==================================================

local function makeDraggable(frame, handle)
	local dragging = false
	local dragInput
	local dragStart
	local startPosition

	track(
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
	)

	track(
		handle.InputChanged:Connect(function(input)

			if input.UserInputType
					== Enum.UserInputType.MouseMovement
				or input.UserInputType
					== Enum.UserInputType.Touch then

				dragInput = input
			end
		end)
	)

	track(
		UserInputService.InputChanged:Connect(function(input)

			if dragging
				and input == dragInput then

				local delta =
					input.Position
					- dragStart

				frame.Position =
					UDim2.new(
						startPosition.X.Scale,

						startPosition.X.Offset
							+ delta.X,

						startPosition.Y.Scale,

						startPosition.Y.Offset
							+ delta.Y
					)
			end
		end)
	)

	track(
		UserInputService.InputEnded:Connect(function(input)

			if input.UserInputType
					== Enum.UserInputType.MouseButton1
				or input.UserInputType
					== Enum.UserInputType.Touch then

				dragging = false
			end
		end)
	)
end

--==================================================
-- DEVICE DRAWINGS
--==================================================

local function buildPcDrawing(parent)
	local monitor =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					40,
					43,
					49
				),

			Position =
				UDim2.new(
					0.5,
					-65,
					0,
					18
				),

			Size =
				UDim2.fromOffset(
					130,
					78
				),

			Parent = parent
		})

	corner(monitor, 9)

	stroke(
		monitor,
		Colors.White,
		0.35,
		2
	)

	local screen =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					92,
					97,
					108
				),

			Position =
				UDim2.fromOffset(
					7,
					7
				),

			Size =
				UDim2.new(
					1,
					-14,
					1,
					-20
				),

			Parent = monitor
		})

	corner(screen, 5)

	addGlassGradient(screen)

	create("Frame", {
		BackgroundColor3 =
			Color3.fromRGB(
				220,
				223,
				228
			),

		BorderSizePixel = 0,

		Position =
			UDim2.new(
				0.5,
				-4,
				1,
				0
			),

		Size =
			UDim2.fromOffset(
				8,
				24
			),

		Parent = monitor
	})

	local stand =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					220,
					223,
					228
				),

			BorderSizePixel = 0,

			Position =
				UDim2.new(
					0.5,
					-32,
					1,
					20
				),

			Size =
				UDim2.fromOffset(
					64,
					7
				),

			Parent = monitor
		})

	corner(stand, 4)

	create("Frame", {
		BackgroundColor3 =
			Colors.Accent,

		BorderSizePixel = 0,

		Position =
			UDim2.new(
				1,
				-14,
				1,
				-9
			),

		Size =
			UDim2.fromOffset(
				4,
				4
			),

		Parent = monitor
	})
end

local function buildPhoneDrawing(parent)
	local phoneBody =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					39,
					42,
					48
				),

			Position =
				UDim2.new(
					0.5,
					-42,
					0,
					10
				),

			Size =
				UDim2.fromOffset(
					84,
					130
				),

			Parent = parent
		})

	corner(phoneBody, 16)

	stroke(
		phoneBody,
		Colors.White,
		0.32,
		2
	)

	local screen =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					96,
					100,
					110
				),

			Position =
				UDim2.fromOffset(
					7,
					15
				),

			Size =
				UDim2.new(
					1,
					-14,
					1,
					-30
				),

			Parent = phoneBody
		})

	corner(screen, 9)

	addGlassGradient(screen)

	local speaker =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					216,
					219,
					225
				),

			BorderSizePixel = 0,

			Position =
				UDim2.new(
					0.5,
					-10,
					0,
					7
				),

			Size =
				UDim2.fromOffset(
					20,
					3
				),

			Parent = phoneBody
		})

	corner(speaker, 3)

	local cameraDot =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					145,
					149,
					160
				),

			BorderSizePixel = 0,

			Position =
				UDim2.new(
					0.5,
					-17,
					0,
					6
				),

			Size =
				UDim2.fromOffset(
					4,
					4
				),

			Parent = phoneBody
		})

	corner(cameraDot, 4)

	local home =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					220,
					223,
					228
				),

			BorderSizePixel = 0,

			Position =
				UDim2.new(
					0.5,
					-14,
					1,
					-8
				),

			Size =
				UDim2.fromOffset(
					28,
					3
				),

			Parent = phoneBody
		})

	corner(home, 3)
end

--==================================================
-- MAIN MENU
--==================================================

local function buildMainMenu(mode)
	local phoneMode =
		mode == "PHONE"

	local width =
		phoneMode
			and 445
			or 720

	local height =
		phoneMode
			and 625
			or 470

	local sidebarWidth =
		phoneMode
			and 116
			or 182

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
				Colors.Main,

			BackgroundTransparency =
				0.12,

			BorderSizePixel = 0,

			ClipsDescendants = true,

			Parent = screenGui
		})

	corner(
		mainFrame,
		16
	)

	stroke(
		mainFrame,
		Colors.Stroke,
		0.42,
		1.25
	)

	addGlassGradient(
		mainFrame
	)

	create("ImageLabel", {
		BackgroundTransparency = 1,

		Image =
			"rbxassetid://1316045217",

		ImageColor3 =
			Color3.fromRGB(
				255,
				255,
				255
			),

		ImageTransparency =
			0.93,

		ScaleType =
			Enum.ScaleType.Tile,

		TileSize =
			UDim2.fromOffset(
				120,
				120
			),

		Size =
			UDim2.fromScale(
				1,
				1
			),

		ZIndex = 0,

		Parent = mainFrame
	})

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
						(
							Camera.ViewportSize.X
							- 18
						) / width,

						(
							Camera.ViewportSize.Y
							- 18
						) / height,

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
				):Connect(
					updateScale
				)
			)
		end
	end

	-- Sidebar

	local sidebar =
		create("Frame", {
			BackgroundColor3 =
				Colors.Sidebar,

			BackgroundTransparency =
				0.14,

			BorderSizePixel = 0,

			Size =
				UDim2.new(
					0,
					sidebarWidth,
					1,
					0
				),

			ZIndex = 2,

			Parent = mainFrame
		})

	create("Frame", {
		BackgroundColor3 =
			Colors.White,

		BackgroundTransparency =
			0.84,

		BorderSizePixel = 0,

		Position =
			UDim2.new(
				1,
				-1,
				0,
				0
			),

		Size =
			UDim2.new(
				0,
				1,
				1,
				0
			),

		ZIndex = 3,

		Parent = sidebar
	})

	local brand =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.28,

			Position =
				UDim2.fromOffset(
					10,
					12
				),

			Size =
				UDim2.new(
					1,
					-20,
					0,
					58
				),

			ZIndex = 3,

			Parent = sidebar
		})

	corner(brand, 12)

	stroke(
		brand,
		Colors.Stroke,
		0.65,
		1
	)

	local gameIcon =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.CardDark,

			Position =
				UDim2.fromOffset(
					8,
					8
				),

			Size =
				UDim2.fromOffset(
					42,
					42
				),

			Image = GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			ZIndex = 4,

			Parent = brand
		})

	corner(gameIcon, 10)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				58,
				8
			),

		Size =
			UDim2.new(
				1,
				-64,
				0,
				22
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

		ZIndex = 4,

		Parent = brand
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				58,
				29
			),

		Size =
			UDim2.new(
				1,
				-64,
				0,
				18
			),

		Font =
			Enum.Font.Gotham,

		Text =
			phoneMode
				and "Mobile"
				or "Desktop",

		TextColor3 =
			Colors.SubText,

		TextSize = 9,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		ZIndex = 4,

		Parent = brand
	})

	local profile =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.32,

			Position =
				UDim2.fromOffset(
					10,
					80
				),

			Size =
				UDim2.new(
					1,
					-20,
					0,
					66
				),

			ZIndex = 3,

			Parent = sidebar
		})

	corner(profile, 12)

	local avatar =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.CardDark,

			Position =
				UDim2.fromOffset(
					9,
					12
				),

			Size =
				UDim2.fromOffset(
					42,
					42
				),

			Image =
				PLAYER_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			ZIndex = 4,

			Parent = profile
		})

	corner(avatar, 21)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				58,
				11
			),

		Size =
			UDim2.new(
				1,
				-62,
				0,
				20
			),

		Font =
			Enum.Font.GothamSemibold,

		Text =
			LocalPlayer.DisplayName,

		TextColor3 =
			Colors.Text,

		TextSize = 10,

		TextTruncate =
			Enum.TextTruncate.AtEnd,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		ZIndex = 4,

		Parent = profile
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				58,
				33
			),

		Size =
			UDim2.new(
				1,
				-62,
				0,
				18
			),

		Font =
			Enum.Font.Gotham,

		Text =
			"@" .. LocalPlayer.Name,

		TextColor3 =
			Colors.SubText,

		TextSize = 8,

		TextTruncate =
			Enum.TextTruncate.AtEnd,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		ZIndex = 4,

		Parent = profile
	})

	local navHolder =
		create("ScrollingFrame", {
			BackgroundTransparency = 1,

			BorderSizePixel = 0,

			Position =
				UDim2.fromOffset(
					10,
					158
				),

			Size =
				UDim2.new(
					1,
					-20,
					1,
					-168
				),

			CanvasSize =
				UDim2.new(),

			AutomaticCanvasSize =
				Enum.AutomaticSize.Y,

			ScrollBarThickness = 2,

			ScrollBarImageColor3 =
				Colors.Stroke,

			ZIndex = 3,

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

	-- Content

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

			ZIndex = 2,

			Parent = mainFrame
		})

	local header =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.72,

			Position =
				UDim2.fromOffset(
					0,
					0
				),

			Size =
				UDim2.new(
					1,
					0,
					0,
					64
				),

			ZIndex = 3,

			Parent = content
		})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				18,
				9
			),

		Size =
			UDim2.new(
				1,
				-110,
				0,
				25
			),

		Font =
			Enum.Font.GothamBold,

		Text =
			"THH MENU",

		TextColor3 =
			Colors.Text,

		TextSize = 18,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		ZIndex = 4,

		Parent = header
	})

	local pageTitle =
		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					18,
					35
				),

			Size =
				UDim2.new(
					1,
					-110,
					0,
					17
				),

			Font =
				Enum.Font.Gotham,

			Text = "Home",

			TextColor3 =
				Colors.SubText,

			TextSize = 10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			ZIndex = 4,

			Parent = header
		})

	local hideButton =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.26,

			AnchorPoint =
				Vector2.new(
					1,
					0
				),

			Position =
				UDim2.new(
					1,
					-14,
					0,
					14
				),

			Size =
				UDim2.fromOffset(
					36,
					36
				),

			Font =
				Enum.Font.GothamBold,

			Text = "—",

			TextColor3 =
				Colors.Text,

			TextSize = 17,

			ZIndex = 5,

			Parent = header
		})

	corner(hideButton, 10)

	stroke(
		hideButton,
		Colors.Stroke,
		0.7,
		1
	)

	local pageHolder =
		create("Frame", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					0,
					64
				),

			Size =
				UDim2.new(
					1,
					0,
					1,
					-64
				),

			ZIndex = 3,

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

				ScrollBarImageColor3 =
					Colors.Stroke,

				Visible = false,

				ZIndex = 3,

				Parent = pageHolder
			})

		padding(
			page,
			14,
			14,
			14,
			14
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
		for pageName, page in pairs(
			pages
		) do
			page.Visible =
				pageName == name
		end

		for navName, button in pairs(
			navButtons
		) do
			local selected =
				navName == name

			tween(
				button,
				0.12,
				{
					BackgroundColor3 =
						selected
						and Colors.Card
						or Colors.Sidebar,

					BackgroundTransparency =
						selected
						and 0.18
						or 1,

					TextColor3 =
						selected
						and Colors.Text
						or Colors.SubText
				}
			)
		end

		pageTitle.Text = name
	end

	local function createNav(name)
		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Sidebar,

				BackgroundTransparency = 1,

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

				TextSize = 11,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				ZIndex = 4,

				Parent = navHolder
			})

		padding(
			button,
			12,
			8,
			0,
			0
		)

		corner(button, 9)

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

				BackgroundTransparency =
					0.26,

				BorderSizePixel = 0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						0
					),

				AutomaticSize =
					Enum.AutomaticSize.Y,

				ZIndex = 4,

				Parent = parent
			})

		corner(frame, 13)

		stroke(
			frame,
			Colors.Stroke,
			0.67,
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
					8
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

			ZIndex = 5,

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

				ZIndex = 5,

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
					Colors.CardDark,

				BackgroundTransparency =
					0.38,

				BorderSizePixel = 0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						phoneMode
							and 61
							or 50
					),

				Text = "",

				ZIndex = 5,

				Parent = parent
			})

		corner(button, 10)

		stroke(
			button,
			Colors.Stroke,
			0.8,
			1
		)

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
					-88,
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

			ZIndex = 6,

			Parent = button
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					28
				),

			Size =
				UDim2.new(
					1,
					-94,
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

			ZIndex = 6,

			Parent = button
		})

		local switch =
			create("Frame", {
				BackgroundColor3 =
					Color3.fromRGB(
						116,
						119,
						127
					),

				BackgroundTransparency =
					0.2,

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
						46,
						25
					),

				ZIndex = 6,

				Parent = button
			})

		corner(switch, 14)

		local knob =
			create("Frame", {
				BackgroundColor3 =
					Colors.Text,

				AnchorPoint =
					Vector2.new(
						0.5,
						0.5
					),

				Position =
					UDim2.new(
						0,
						12.5,
						0.5,
						0
					),

				Size =
					UDim2.fromOffset(
						18,
						18
					),

				ZIndex = 7,

				Parent = switch
			})

		corner(knob, 10)

		track(
			button.MouseButton1Click:Connect(function()

				enabled =
					not enabled

				tween(
					switch,
					0.13,
					{
						BackgroundColor3 =
							enabled
							and Colors.AccentDark
							or Color3.fromRGB(
								116,
								119,
								127
							)
					}
				)

				tween(
					knob,
					0.13,
					{
						Position =
							enabled
							and UDim2.new(
								1,
								-12.5,
								0.5,
								0
							)
							or UDim2.new(
								0,
								12.5,
								0.5,
								0
							),

						BackgroundColor3 =
							enabled
							and Colors.Accent
							or Colors.Text
					}
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
					Colors.CardDark,

				BackgroundTransparency =
					0.38,

				BorderSizePixel = 0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						50
					),

				Text = "",

				ZIndex = 5,

				Parent = parent
			})

		corner(button, 10)

		stroke(
			button,
			danger
				and Colors.Danger
				or Colors.Stroke,
			danger
				and 0.6
				or 0.8,
			1
		)

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

			ZIndex = 6,

			Parent = button
		})

		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					12,
					28
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

			ZIndex = 6,

			Parent = button
		})

		track(
			button.MouseButton1Click:Connect(
				callback
			)
		)
	end

	-- Navigation

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
		"Light gray glass UI • optimized interaction system"
	)

	-- Mods

	local farming =
		section(
			mods,
			"Farming",
			"Hay collection and selling"
		)

	createToggle(
		farming,
		"Auto Farm",
		"Pick 4 HayPieces → sell → repeat",
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
		"Automatically picks up HayPiece",
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
		"TP to SellPart → click → return",
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
		"Reduces local THH pickup delay",
		function(state)

			noPickupCooldownEnabled =
				state

			refreshCachedSettings()
		end
	)

	createToggle(
		farming,
		"Inf Range",
		"Far pickup with hidden camera teleport",
		function(state)

			infRangeEnabled = state

			refreshCachedSettings()
		end
	)

	local pickups =
		section(
			mods,
			"Special Pickups",
			"Automatic special item collection"
		)

	createToggle(
		pickups,
		"Auto Pick Up Diamond",
		"TP close → move Diamond in front → click",
		function(state)

			if state then
				startAutoDiamond()
			else
				stopAutoDiamond()
			end
		end
	)

	createToggle(
		pickups,
		"Auto Pick Up Color Hay",
		"Detect changing HayPiece → pickup",
		function(state)

			if state then
				startAutoColorHay()
			else
				stopAutoColorHay()
			end
		end
	)

	-- Player

	local movement =
		section(
			player,
			"Movement",
			"Player movement modifications"
		)

	createToggle(
		movement,
		"Speed",
		"WalkSpeed = 50",
		function(state)
			setSpeed(state)
		end
	)

	createToggle(
		movement,
		"Infinite Jump",
		"Jump again while airborne",
		function(state)
			infiniteJumpEnabled =
				state
		end
	)

	createToggle(
		movement,
		"Unlock 3rd Person",
		"Classic camera with extended zoom",
		function(state)
			setThirdPerson(state)
		end
	)

	createToggle(
		movement,
		"Fly",
		"WASD + Space/Ctrl on PC",
		function(state)
			setFly(state)
		end
	)

	-- Event

	local eventTools =
		section(
			event,
			"Events",
			"UFO, Needle and Key automation"
		)

	createAction(
		eventTools,
		"Start UFO Event",
		"TP to UfoButtenPart and click it",
		startUfoEvent
	)

	createToggle(
		eventTools,
		"Auto Find Needle",
		"Needle / Hay Needle / Haystack Needle / Hidden Needle",
		function(state)

			if state then
				startAutoFindNeedle()
			else
				stopAutoFindNeedle()
			end
		end
	)

	createToggle(
		eventTools,
		"Auto Find Key",
		"Key / Basement Key / Hay Key / Hidden Key",
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
			"Search For The Needle",
			"Place IDs"
		)

	createAction(
		ids,
		"Copy Main Place ID",
		tostring(
			SEARCH_FOR_NEEDLE_PLACE_ID
		),
		function()
			copyText(
				SEARCH_FOR_NEEDLE_PLACE_ID
			)
		end
	)

	createAction(
		ids,
		"Copy Farmhouse ID",
		tostring(
			FARMHOUSE_PLACE_ID
		),
		function()
			copyText(
				FARMHOUSE_PLACE_ID
			)
		end
	)

	createAction(
		ids,
		"Copy Basement ID",
		tostring(
			BASEMENT_PLACE_ID
		),
		function()
			copyText(
				BASEMENT_PLACE_ID
			)
		end
	)

	-- Utility

	local tools =
		section(
			utility,
			"Tools",
			"Useful THH tools"
		)

	createToggle(
		tools,
		"Part Name Hover",
		"Shows the exact part under your cursor",
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
		"Prevents normal idle kicks",
		function(state)

			antiAfkEnabled = state

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

				track(
					antiAfkConnection
				)
			end
		end
	)

	local server =
		section(
			utility,
			"Server",
			"Server and menu actions"
		)

	createAction(
		server,
		"Rejoin Server",
		"Rejoin your current server",
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
		"Reset your character",
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
		"Copy current JobId",
		function()

			copyText(
				game.JobId
			)
		end
	)

	createAction(
		server,
		"Leave Server",
		"Leave the current server",
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
		"Disable every mod and remove THH HUB",
		function()

			cleanupAll(true)
		end,
		true
	)

	-- Updates

	section(
		updates,
		"UI Update • 09/09/26",
		"• lighter gray transparent menu\n"
		.. "• cleaner glass-style cards\n"
		.. "• softer borders and controls\n"
		.. "• real monitor-style PC selector\n"
		.. "• real smartphone-style Phone selector\n"
		.. "• improved sidebar/profile area\n"
		.. "• cleaner toggles and sections"
	)

	local function setVisible(value)
		mainFrame.Visible =
			value

		if phoneMode
			and floatingButton then

			floatingButton.Visible =
				not value
		end
	end

	track(
		hideButton.MouseButton1Click:Connect(function()
			setVisible(false)
		end)
	)

	if phoneMode then
		floatingButton =
			create("ImageButton", {
				BackgroundColor3 =
					Colors.Card,

				BackgroundTransparency =
					0.18,

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
						12
					),

				Size =
					UDim2.fromOffset(
						52,
						52
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
			15
		)

		stroke(
			floatingButton,
			Colors.Stroke,
			0.45,
			1.2
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

--==================================================
-- DEVICE CHOOSER
--==================================================

local function showDeviceChooser()
	local overlay =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					20,
					22,
					26
				),

			BackgroundTransparency =
				0.38,

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
					500,
					335
				),

			BackgroundColor3 =
				Colors.Main,

			BackgroundTransparency =
				0.16,

			BorderSizePixel = 0,

			Parent = overlay
		})

	corner(
		panel,
		18
	)

	stroke(
		panel,
		Colors.Stroke,
		0.42,
		1.2
	)

	addGlassGradient(panel)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				24,
				18
			),

		Size =
			UDim2.new(
				1,
				-48,
				0,
				28
			),

		Font =
			Enum.Font.GothamBold,

		Text =
			"what are you on?",

		TextColor3 =
			Colors.Text,

		TextSize = 20,

		TextXAlignment =
			Enum.TextXAlignment.Center,

		Parent = panel
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				24,
				50
			),

		Size =
			UDim2.new(
				1,
				-48,
				0,
				20
			),

		Font =
			Enum.Font.Gotham,

		Text =
			"pick pc or phone",

		TextColor3 =
			Colors.SubText,

		TextSize = 11,

		Parent = panel
	})

	local pc =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.26,

			Position =
				UDim2.fromOffset(
					24,
					88
				),

			Size =
				UDim2.fromOffset(
					215,
					215
				),

			Text = "",

			Parent = panel
		})

	corner(pc, 15)

	stroke(
		pc,
		Colors.Stroke,
		0.58,
		1
	)

	buildPcDrawing(pc)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-57
			),

		Size =
			UDim2.new(
				1,
				0,
				0,
				24
			),

		Font =
			Enum.Font.GothamBold,

		Text = "PC",

		TextColor3 =
			Colors.Text,

		TextSize = 16,

		Parent = pc
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-33
			),

		Size =
			UDim2.new(
				1,
				0,
				0,
				18
			),

		Font =
			Enum.Font.Gotham,

		Text =
			"Desktop controls",

		TextColor3 =
			Colors.SubText,

		TextSize = 9,

		Parent = pc
	})

	local phone =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.26,

			Position =
				UDim2.fromOffset(
					261,
					88
				),

			Size =
				UDim2.fromOffset(
					215,
					215
				),

			Text = "",

			Parent = panel
		})

	corner(phone, 15)

	stroke(
		phone,
		Colors.Stroke,
		0.58,
		1
	)

	buildPhoneDrawing(phone)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-57
			),

		Size =
			UDim2.new(
				1,
				0,
				0,
				24
			),

		Font =
			Enum.Font.GothamBold,

		Text = "PHONE",

		TextColor3 =
			Colors.Text,

		TextSize = 16,

		Parent = phone
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-33
			),

		Size =
			UDim2.new(
				1,
				0,
				0,
				18
			),

		Font =
			Enum.Font.Gotham,

		Text =
			"Touch controls",

		TextColor3 =
			Colors.SubText,

		TextSize = 9,

		Parent = phone
	})

	local selected = false

	local function choose(mode)
		if selected then
			return
		end

		selected = true

		overlay:Destroy()

		buildMainMenu(mode)
	end

	track(
		pc.MouseEnter:Connect(function()
			tween(
				pc,
				0.13,
				{
					BackgroundTransparency = 0.15
				}
			)
		end)
	)

	track(
		pc.MouseLeave:Connect(function()
			tween(
				pc,
				0.13,
				{
					BackgroundTransparency = 0.26
				}
			)
		end)
	)

	track(
		phone.MouseEnter:Connect(function()
			tween(
				phone,
				0.13,
				{
					BackgroundTransparency = 0.15
				}
			)
		end)
	)

	track(
		phone.MouseLeave:Connect(function()
			tween(
				phone,
				0.13,
				{
					BackgroundTransparency = 0.26
				}
			)
		end)
	)

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

--==================================================
-- KEY SYSTEM
--==================================================

local function showKeySystem()
	local overlay =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					18,
					20,
					24
				),

			BackgroundTransparency =
				0.38,

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
					420,
					410
				),

			BackgroundColor3 =
				Colors.Main,

			BackgroundTransparency =
				0.15,

			BorderSizePixel = 0,

			Parent = overlay
		})

	corner(panel, 18)

	stroke(
		panel,
		Colors.Stroke,
		0.42,
		1.2
	)

	addGlassGradient(panel)

	local header =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.28,

			Position =
				UDim2.fromOffset(
					16,
					16
				),

			Size =
				UDim2.new(
					1,
					-32,
					0,
					96
				),

			Parent = panel
		})

	corner(header, 14)

	stroke(
		header,
		Colors.Stroke,
		0.65,
		1
	)

	makeDraggable(
		panel,
		header
	)

	local icon =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.CardDark,

			Position =
				UDim2.fromOffset(
					16,
					15
				),

			Size =
				UDim2.fromOffset(
					66,
					66
				),

			Image =
				GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			Parent = header
		})

	corner(icon, 14)

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				98,
				20
			),

		Size =
			UDim2.new(
				1,
				-115,
				0,
				30
			),

		Font =
			Enum.Font.GothamBold,

		Text = "THH HUB",

		TextColor3 =
			Colors.Text,

		TextSize = 23,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = header
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				98,
				52
			),

		Size =
			UDim2.new(
				1,
				-115,
				0,
				22
			),

		Font =
			Enum.Font.Gotham,

		Text =
			"Authentication",

		TextColor3 =
			Colors.SubText,

		TextSize = 11,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = header
	})

	create("TextLabel", {
		BackgroundTransparency = 1,

		Position =
			UDim2.fromOffset(
				26,
				134
			),

		Size =
			UDim2.new(
				1,
				-52,
				0,
				20
			),

		Font =
			Enum.Font.GothamSemibold,

		Text =
			"Access Key",

		TextColor3 =
			Colors.Text,

		TextSize = 11,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent = panel
	})

	local keyBox =
		create("TextBox", {
			BackgroundColor3 =
				Colors.Input,

			BackgroundTransparency =
				0.16,

			Position =
				UDim2.fromOffset(
					26,
					160
				),

			Size =
				UDim2.new(
					1,
					-52,
					0,
					52
				),

			ClearTextOnFocus = false,

			Font =
				Enum.Font.Gotham,

			PlaceholderText =
				"paste your key here...",

			PlaceholderColor3 =
				Colors.Muted,

			Text = "",

			TextColor3 =
				Colors.Text,

			TextSize = 12,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = panel
		})

	padding(
		keyBox,
		14,
		14,
		0,
		0
	)

	corner(keyBox, 11)

	stroke(
		keyBox,
		Colors.Stroke,
		0.72,
		1
	)

	local continueButton =
		create("TextButton", {
			BackgroundColor3 =
				Color3.fromRGB(
					219,
					222,
					227
				),

			BackgroundTransparency =
				0.02,

			Position =
				UDim2.fromOffset(
					26,
					230
				),

			Size =
				UDim2.new(
					1,
					-52,
					0,
					48
				),

			Font =
				Enum.Font.GothamBold,

			Text = "Continue",

			TextColor3 =
				Color3.fromRGB(
					45,
					48,
					54
				),

			TextSize = 12,

			Parent = panel
		})

	corner(
		continueButton,
		11
	)

	local getKeyButton =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.24,

			Position =
				UDim2.fromOffset(
					26,
					294
				),

			Size =
				UDim2.new(
					1,
					-52,
					0,
					46
				),

			Font =
				Enum.Font.GothamSemibold,

			Text =
				"Get Access Key",

			TextColor3 =
				Colors.Text,

			TextSize = 12,

			Parent = panel
		})

	corner(
		getKeyButton,
		11
	)

	stroke(
		getKeyButton,
		Colors.Stroke,
		0.68,
		1
	)

	local status =
		create("TextLabel", {
			BackgroundTransparency = 1,

			Position =
				UDim2.fromOffset(
					26,
					357
				),

			Size =
				UDim2.new(
					1,
					-52,
					0,
					30
				),

			Font =
				Enum.Font.Gotham,

			Text =
				"enter your key to continue",

			TextColor3 =
				Colors.SubText,

			TextSize = 10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent = panel
		})

	track(
		getKeyButton.MouseButton1Click:Connect(function()

			if copyText(
				GET_KEY_URL
			) then

				status.Text =
					"Get Key link copied."

				status.TextColor3 =
					Colors.Accent
			else
				status.Text =
					"Clipboard unavailable."

				status.TextColor3 =
					Colors.Danger
			end
		end)
	)

	local checking = false

	local function check()
		if checking then
			return
		end

		local key =
			keyBox.Text:match(
				"^%s*(.-)%s*$"
			)

		if key == "" then
			status.Text =
				"Enter your access key."

			status.TextColor3 =
				Colors.Danger

			return
		end

		checking = true

		continueButton.Text =
			"Checking..."

		status.Text =
			"validating key..."

		status.TextColor3 =
			Colors.SubText

		task.spawn(function()

			local valid, result =
				validateKey(key)

			if not panel.Parent then
				return
			end

			if valid then
				status.Text =
					"key accepted"

				status.TextColor3 =
					Colors.Accent

				continueButton.Text =
					"Accepted"

				task.wait(0.2)

				overlay:Destroy()

				showDeviceChooser()

				return
			end

			checking = false

			continueButton.Text =
				"Continue"

			status.Text =
				result

			status.TextColor3 =
				Colors.Danger
		end)
	end

	track(
		continueButton.MouseButton1Click:Connect(
			check
		)
	)

	track(
		keyBox.FocusLost:Connect(function(
			enterPressed
		)
			if enterPressed then
				check()
			end
		end)
	)
end

showKeySystem()
