if _G.THHGrowersHardUnloaded then
	return
end

if type(_G.THHGrowersCleanup) == "function" then
	pcall(_G.THHGrowersCleanup)
end

--========================================================
-- SERVICES
--========================================================

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

--========================================================
-- VAMPAUTH
--========================================================

local PROJECT_ID = "PD6XH2EGXZXUHGED"
local AUTH_SECRET = "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"
local GET_KEY_URL = "https://vampauth.com/PD6XH2EGXZXUHGED/flow"
local VAMPAUTH_CLIENT_URL = "https://vampauth.com/client/vampauth.lua"

--========================================================
-- PLACE IDS
--========================================================

local SEARCH_FOR_NEEDLE_PLACE_ID = 77108422251420
local FARMHOUSE_PLACE_ID = 108628039999641
local BASEMENT_PLACE_ID = 83445806734780

--========================================================
-- ICONS
--========================================================

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

--========================================================
-- COLORS
--========================================================

local Colors = {
	Main = Color3.fromRGB(73, 76, 84),
	Sidebar = Color3.fromRGB(58, 61, 68),

	Card = Color3.fromRGB(101, 104, 113),
	CardDark = Color3.fromRGB(53, 56, 63),

	Stroke = Color3.fromRGB(190, 193, 201),

	Text = Color3.fromRGB(249, 250, 252),
	SubText = Color3.fromRGB(207, 210, 217),
	Muted = Color3.fromRGB(166, 170, 179),

	Accent = Color3.fromRGB(96, 226, 145),
	AccentDark = Color3.fromRGB(48, 100, 67),

	Beta = Color3.fromRGB(75, 160, 255),
	BetaDark = Color3.fromRGB(39, 83, 135),

	Danger = Color3.fromRGB(232, 88, 88)
}

--========================================================
-- CONFIG
--========================================================

local SPEED_AMOUNT = 50
local FLY_SPEED = 58

local PICKUP_DISTANCE = 2.7

local SELL_FIRST_TP_DISTANCE = 5
local SELL_SECOND_TP_DISTANCE = 2.25

local UFO_DISTANCE = 3

local FARM_GRABS_BEFORE_SELL = 5

local FARM_BETWEEN_HAY_DELAY = 0.08
local FARM_AFTER_SELL_DELAY = 0.15

local NORMAL_CLICK_BURST = 4
local SPECIAL_CLICK_BURST = 7
local KEY_CLICK_BURST = 8
local NEEDLE_CLICK_BURST = 8
local SELL_CLICK_BURST = 5

local CLICK_GAP = 0.025

local AUTO_SELL_INTERVAL = 2

-- Stops Auto Farm + Auto Sell from selling twice
-- almost at the exact same time.
local SHARED_SELL_COOLDOWN = 0.65

-- Reservation automatically expires if something
-- unexpected happens.
local RESERVATION_TIMEOUT = 2

--========================================================
-- STATE
--========================================================

local connections = {}

local screenGui
local mainFrame
local mainScale
local floatingButton
local notificationHolder

local VampauthClient

local cleaning = false

-- Only one teleport/pickup/sell interaction can move
-- the character at a time.
local interactionBusy = false
local interactionOwner = nil

local autoFarmEnabled = false
local autoHayEnabled = false
local autoSellEnabled = false

local autoDiamondEnabled = false
local autoColorHayEnabled = false

local autoFindKeyEnabled = false
local autoFindNeedleEnabled = false

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

local colorHayRunId = 0
local diamondRunId = 0
local keyRunId = 0
local needleRunId = 0

local flyConnection
local flyVelocity
local flyGyro
local flyHumanoid
local flyOldPlatformStand
local flyJumpUntil = 0

local lastSellTime = 0

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

--========================================================
-- SHARED OBJECT RESERVATIONS
--========================================================

-- Stops two mods from trying to grab the exact same
-- object at the same time.

local objectReservations =
	setmetatable({}, {
		__mode = "k"
	})

local function reserveObject(object, owner)
	if not object
		or not object.Parent then

		return false
	end

	local reservation =
		objectReservations[object]

	if reservation then
		if os.clock() - reservation.Time
			< RESERVATION_TIMEOUT then

			if reservation.Owner ~= owner then
				return false
			end
		end
	end

	objectReservations[object] = {
		Owner = owner,
		Time = os.clock()
	}

	return true
end

local function releaseObject(object, owner)
	local reservation =
		objectReservations[object]

	if not reservation then
		return
	end

	if reservation.Owner == owner then
		objectReservations[object] = nil
	end
end

local function isReservedByOther(
	object,
	owner
)
	local reservation =
		objectReservations[object]

	if not reservation then
		return false
	end

	if os.clock() - reservation.Time
		>= RESERVATION_TIMEOUT then

		objectReservations[object] = nil
		return false
	end

	return reservation.Owner ~= owner
end

--========================================================
-- CACHES
--========================================================

local hayCache = {}
local diamondCache = {}
local sellCache = {}
local ufoCache = {}
local keyCache = {}
local needleCache = {}

local colorHayState =
	setmetatable({}, {
		__mode = "k"
	})

local colorHayConnections =
	setmetatable({}, {
		__mode = "k"
	})

--========================================================
-- QUEUES
--========================================================

local diamondQueue = {}
local colorHayQueue = {}
local keyQueue = {}
local needleQueue = {}

local diamondQueued =
	setmetatable({}, {
		__mode = "k"
	})

local colorHayQueued =
	setmetatable({}, {
		__mode = "k"
	})

local keyQueued =
	setmetatable({}, {
		__mode = "k"
	})

local needleQueued =
	setmetatable({}, {
		__mode = "k"
	})

local notify = function()
end

--========================================================
-- BASIC HELPERS
--========================================================

local function track(connection)
	table.insert(
		connections,
		connection
	)

	return connection
end

local function create(
	className,
	properties
)
	local object =
		Instance.new(className)

	for property, value in pairs(
		properties or {}
	) do
		object[property] = value
	end

	return object
end

local function corner(
	object,
	radius
)
	return create("UICorner", {
		CornerRadius =
			UDim.new(
				0,
				radius or 10
			),

		Parent = object
	})
end

local function stroke(
	object,
	color,
	transparency,
	thickness
)
	return create("UIStroke", {
		Color =
			color or Colors.Stroke,

		Transparency =
			transparency or 0.5,

		Thickness =
			thickness or 1,

		ApplyStrokeMode =
			Enum.ApplyStrokeMode.Border,

		Parent = object
	})
end

local function padding(
	object,
	left,
	right,
	top,
	bottom
)
	return create("UIPadding", {
		PaddingLeft =
			UDim.new(
				0,
				left or 0
			),

		PaddingRight =
			UDim.new(
				0,
				right or 0
			),

		PaddingTop =
			UDim.new(
				0,
				top or 0
			),

		PaddingBottom =
			UDim.new(
				0,
				bottom or 0
			),

		Parent = object
	})
end

local function addGlassGradient(object)
	return create("UIGradient", {
		Rotation = 90,

		Color =
			ColorSequence.new({
				ColorSequenceKeypoint.new(
					0,
					Color3.fromRGB(
						112,
						115,
						124
					)
				),

				ColorSequenceKeypoint.new(
					1,
					Color3.fromRGB(
						68,
						71,
						79
					)
				)
			}),

		Parent = object
	})
end

local function copyText(text)
	text = tostring(text)

	if type(setclipboard)
		== "function" then

		return pcall(
			setclipboard,
			text
		)
	end

	if type(toclipboard)
		== "function" then

		return pcall(
			toclipboard,
			text
		)
	end

	return false
end

--========================================================
-- CHARACTER
--========================================================

local function getCharacter()
	return LocalPlayer.Character
end

local function getHumanoid()
	local character =
		getCharacter()

	if not character then
		return nil
	end

	return character:FindFirstChildOfClass(
		"Humanoid"
	)
end

local function getRoot()
	local character =
		getCharacter()

	if not character then
		return nil
	end

	return character:FindFirstChild(
		"HumanoidRootPart"
	)
		or character:FindFirstChild(
			"UpperTorso"
		)
		or character:FindFirstChild(
			"Torso"
		)
end

local function getHead()
	local character =
		getCharacter()

	if not character then
		return nil
	end

	return character:FindFirstChild("Head")
		or getRoot()
end

local function stopVelocity()
	local root =
		getRoot()

	if not root then
		return
	end

	pcall(function()
		root.AssemblyLinearVelocity =
			Vector3.zero

		root.AssemblyAngularVelocity =
			Vector3.zero
	end)
end

--========================================================
-- OBJECT HELPERS
--========================================================

local function getMainPart(object)
	if not object then
		return nil
	end

	if object:IsA("BasePart") then
		return object
	end

	if object:IsA("Model")
		and object.PrimaryPart then

		return object.PrimaryPart
	end

	if object:IsA("Tool") then
		return object:FindFirstChild(
			"Handle"
		)
			or object:FindFirstChildWhichIsA(
				"BasePart",
				true
			)
	end

	return object:FindFirstChildWhichIsA(
		"BasePart",
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

	local part =
		getMainPart(object)

	if part then
		return part.CFrame
	end

	return nil
end

local function setObjectPivot(
	object,
	newCFrame
)
	if not object
		or not object.Parent
		or not newCFrame then

		return false
	end

	if object:IsA("Model") then
		return pcall(function()
			object:PivotTo(
				newCFrame
			)
		end)
	end

	if object:IsA("BasePart") then
		return pcall(function()
			object.CFrame =
				newCFrame
		end)
	end

	local part =
		getMainPart(object)

	if part then
		return pcall(function()
			part.CFrame =
				newCFrame
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

--========================================================
-- SHARED INTERACTION LOCK
--========================================================

local function withInteractionLock(
	owner,
	callback,
	shouldContinue
)
	while interactionBusy do

		if shouldContinue
			and not shouldContinue() then

			return false
		end

		task.wait(0.025)
	end

	if shouldContinue
		and not shouldContinue() then

		return false
	end

	interactionBusy = true
	interactionOwner = owner

	local success, result =
		pcall(callback)

	interactionOwner = nil
	interactionBusy = false

	if success then
		return result
	end

	warn(
		"[THH HUB] "
		.. tostring(owner)
		.. " interaction error:",
		result
	)

	return false
end

--========================================================
-- CAMERA LOCK
--========================================================

local function lockCamera()
	Camera =
		Workspace.CurrentCamera
		or Camera

	if not Camera then
		return function()
		end
	end

	local oldType =
		Camera.CameraType

	local oldSubject =
		Camera.CameraSubject

	local oldCFrame =
		Camera.CFrame

	local oldFocus =
		Camera.Focus

	Camera.CameraType =
		Enum.CameraType.Scriptable

	Camera.CFrame =
		oldCFrame

	Camera.Focus =
		oldFocus

	local connection

	connection =
		RunService.RenderStepped:Connect(function()

			if Camera then
				Camera.CFrame =
					oldCFrame

				Camera.Focus =
					oldFocus
			end
		end)

	local restored =
		false

	return function()
		if restored then
			return
		end

		restored =
			true

		if connection then
			pcall(function()
				connection:Disconnect()
			end)
		end

		Camera =
			Workspace.CurrentCamera
			or Camera

		if not Camera then
			return
		end

		pcall(function()
			Camera.CFrame =
				oldCFrame

			Camera.Focus =
				oldFocus

			Camera.CameraSubject =
				oldSubject

			Camera.CameraType =
				oldType
		end)
	end
end

--========================================================
-- RANGE / COOLDOWN
--========================================================

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
		HoldDuration =
			prompt.HoldDuration,

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

	rememberClickDetector(
		detector
	)

	local original =
		originalClickSettings[
			detector
		]

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

	rememberPrompt(
		prompt
	)

	local original =
		originalPromptSettings[
			prompt
		]

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
			(
				noPickupCooldownEnabled
				or infRangeEnabled
			)
				and true
				or original.Enabled
	end)
end

local function applyInteractionObject(object)
	if object:IsA("ClickDetector") then
		applyClickDetector(
			object
		)

	elseif object:IsA("ProximityPrompt") then
		applyPrompt(
			object
		)
	end
end

local function applySettingsToObject(object)
	if not object
		or not object.Parent then

		return
	end

	applyInteractionObject(
		object
	)

	for _, descendant in ipairs(
		object:GetDescendants()
	) do

		applyInteractionObject(
			descendant
		)
	end
end

local function refreshCachedSettings()
	for detector in pairs(
		originalClickSettings
	) do

		if detector
			and detector.Parent then

			applyClickDetector(
				detector
			)
		end
	end

	for prompt in pairs(
		originalPromptSettings
	) do

		if prompt
			and prompt.Parent then

			applyPrompt(
				prompt
			)
		end
	end

	for object in pairs(hayCache) do
		if object.Parent then
			applySettingsToObject(
				object
			)
		end
	end

	for object in pairs(diamondCache) do
		if object.Parent then
			applySettingsToObject(
				object
			)
		end
	end

	for object in pairs(keyCache) do
		if object.Parent then
			applySettingsToObject(
				object
			)
		end
	end

	for object in pairs(needleCache) do
		if object.Parent then
			applySettingsToObject(
				object
			)
		end
	end
end

local function restoreInteractionSettings()
	for detector, original in pairs(
		originalClickSettings
	) do

		if detector
			and detector.Parent then

			pcall(function()

				detector.MaxActivationDistance =
					original.MaxActivationDistance
			end)
		end
	end

	for prompt, original in pairs(
		originalPromptSettings
	) do

		if prompt
			and prompt.Parent then

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

--========================================================
-- KEY / NEEDLE NAMES
--========================================================

local function normalizeName(name)
	name =
		string.lower(
			tostring(
				name or ""
			)
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
	local value =
		normalizeName(
			name
		)

	return value ~= ""
		and string.find(
			value,
			"needle",
			1,
			true
		) ~= nil
end

-- User confirmed the Key is simply called Key.
local function isKeyName(name)
	return name == "Key"
end

local function resolveNamedRoot(
	object,
	checkFunction
)
	if not object then
		return nil
	end

	local current =
		object

	if current:IsA("ClickDetector")
		or current:IsA("ProximityPrompt") then

		current =
			current.Parent
	end

	local matched

	while current
		and current ~= Workspace do

		if checkFunction(
			current.Name
		) then

			matched =
				current

			break
		end

		current =
			current.Parent
	end

	if not matched then
		return nil
	end

	if matched:IsA("Tool") then
		return matched
	end

	local tool =
		matched:FindFirstAncestorOfClass(
			"Tool"
		)

	if tool
		and checkFunction(
			tool.Name
		) then

		return tool
	end

	current =
		matched

	while current
		and current ~= Workspace do

		if current:IsA("Model")
			and checkFunction(
				current.Name
			) then

			return current
		end

		current =
			current.Parent
	end

	return matched
end

--========================================================
-- QUEUES
--========================================================

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

	queued[object] =
		true

	table.insert(
		queue,
		object
	)
end

local function clearQueuedTable(queued)
	for object in pairs(
		queued
	) do
		queued[object] =
			nil
	end
end

--========================================================
-- COLOR HAY TRACKER
--========================================================

local function setupColorHayTracker(hay)
	if colorHayConnections[hay] then
		return
	end

	local part =
		getMainPart(
			hay
		)

	if not part then
		return
	end

	colorHayState[hay] = {
		LastColor =
			part.Color,

		LastChanged =
			0
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

				colorHayConnections[hay] =
					nil

				return
			end

			local state =
				colorHayState[hay]

			if not state then
				return
			end

			local oldColor =
				state.LastColor

			local newColor =
				part.Color

			local difference =
				math.abs(
					oldColor.R
						- newColor.R
				)
				+ math.abs(
					oldColor.G
						- newColor.G
				)
				+ math.abs(
					oldColor.B
						- newColor.B
				)

			state.LastColor =
				newColor

			if difference > 0.015 then

				state.LastChanged =
					os.clock()

				if autoColorHayEnabled then

					queueObject(
						colorHayQueue,
						colorHayQueued,
						hay
					)
				end
			end
		end)

	colorHayConnections[hay] =
		connection

	track(
		connection
	)
end

--========================================================
-- REGISTER OBJECTS
--========================================================

local function registerKeyCandidate(object)
	if not object then
		return
	end

	local key =
		resolveNamedRoot(
			object,
			isKeyName
		)

	if not key
		and isKeyName(
			object.Name
		) then

		key =
			object
	end

	if not key
		or not key.Parent then

		return
	end

	if not keyCache[key] then

		keyCache[key] =
			true

		applySettingsToObject(
			key
		)
	end

	if autoFindKeyEnabled then

		queueObject(
			keyQueue,
			keyQueued,
			key
		)
	end
end

local function registerNeedleCandidate(object)
	if not object then
		return
	end

	local needle =
		resolveNamedRoot(
			object,
			isNeedleName
		)

	if not needle
		and isNeedleName(
			object.Name
		) then

		needle =
			object
	end

	if not needle
		or not needle.Parent then

		return
	end

	if not needleCache[needle] then

		needleCache[needle] =
			true

		applySettingsToObject(
			needle
		)
	end

	if autoFindNeedleEnabled then

		queueObject(
			needleQueue,
			needleQueued,
			needle
		)
	end
end

local function registerObject(object)
	if object.Name == "HayPiece" then

		if not hayCache[object] then

			hayCache[object] =
				true

			applySettingsToObject(
				object
			)

			setupColorHayTracker(
				object
			)
		end
	end

	if object.Name == "Diamond" then

		if not diamondCache[object] then

			diamondCache[object] =
				true

			applySettingsToObject(
				object
			)

			if autoDiamondEnabled then

				queueObject(
					diamondQueue,
					diamondQueued,
					object
				)
			end
		end
	end

	if object.Name == "SellPart" then

		sellCache[object] =
			true
	end

	if object.Name == "UfoButtenPart" then

		ufoCache[object] =
			true
	end

	if object.Name == "Key" then

		registerKeyCandidate(
			object
		)
	end

	if isNeedleName(
		object.Name
	) then

		registerNeedleCandidate(
			object
		)
	end

	if object:IsA("ClickDetector")
		or object:IsA("ProximityPrompt") then

		local current =
			object.Parent

		while current
			and current ~= Workspace do

			if hayCache[current]
				or diamondCache[current]
				or keyCache[current]
				or needleCache[current]
				or current.Name == "Key"
				or isNeedleName(
					current.Name
				) then

				applyInteractionObject(
					object
				)

				break
			end

			current =
				current.Parent
		end
	end
end

local function unregisterObject(object)
	hayCache[object] =
		nil

	diamondCache[object] =
		nil

	sellCache[object] =
		nil

	ufoCache[object] =
		nil

	keyCache[object] =
		nil

	needleCache[object] =
		nil

	diamondQueued[object] =
		nil

	colorHayQueued[object] =
		nil

	keyQueued[object] =
		nil

	needleQueued[object] =
		nil

	colorHayState[object] =
		nil

	objectReservations[object] =
		nil
end

--========================================================
-- ONE STARTUP SCAN ONLY
--========================================================

for _, object in ipairs(
	Workspace:GetDescendants()
) do
	registerObject(
		object
	)
end

--========================================================
-- EVENT TRACKING AFTER STARTUP
--========================================================

track(
	Workspace.DescendantAdded:Connect(function(object)

		registerObject(
			object
		)

		task.defer(function()

			if not object
				or not object.Parent then

				return
			end

			registerKeyCandidate(
				object
			)

			registerNeedleCandidate(
				object
			)

			local parent =
				object.Parent

			if parent
				and parent ~= Workspace then

				registerObject(
					parent
				)
			end
		end)
	end)
)

track(
	Workspace.DescendantRemoving:Connect(function(object)

		unregisterObject(
			object
		)
	end)
)

--========================================================
-- MOVEMENT
--========================================================

local function teleportCharacterNear(
	position,
	distance
)
	local character =
		getCharacter()

	local root =
		getRoot()

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
			Vector3.new(
				0,
				0,
				1
			)
	end

	direction =
		direction.Unit

	local target =
		position
		+ direction * distance

	target =
		Vector3.new(
			target.X,
			position.Y + 2.6,
			target.Z
		)

	local lookAt =
		Vector3.new(
			position.X,
			target.Y,
			position.Z
		)

	pcall(function()

		character:PivotTo(
			CFrame.lookAt(
				target,
				lookAt
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
		distance or 4

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

	local root =
		getRoot()

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

--========================================================
-- CLICKING
--========================================================

local function directClick(detector)
	if not detector
		or not detector.Parent then

		return false
	end

	applyClickDetector(
		detector
	)

	if type(fireclickdetector)
		== "function" then

		return pcall(function()

			fireclickdetector(
				detector
			)
		end)
	end

	return false
end

local function firePrompt(prompt)
	if not prompt
		or not prompt.Parent then

		return false
	end

	applyPrompt(
		prompt
	)

	if type(fireproximityprompt)
		== "function" then

		return pcall(function()

			fireproximityprompt(
				prompt
			)
		end)
	end

	return pcall(function()

		prompt:InputHoldBegin()

		task.wait(
			noPickupCooldownEnabled
				and 0
				or math.min(
					prompt.HoldDuration,
					0.1
				)
		)

		prompt:InputHoldEnd()
	end)
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

	if type(mousemoveabs)
			== "function"
		and type(mouse1click)
			== "function" then

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

local function clickObjectBurst(
	object,
	count,
	gap,
	allowScreenFallback,
	shouldContinue
)
	if not object
		or not object.Parent then

		return false
	end

	count =
		count
		or NORMAL_CLICK_BURST

	gap =
		gap
		or CLICK_GAP

	if allowScreenFallback == nil then
		allowScreenFallback =
			true
	end

	local clicked =
		false

	for _ = 1, count do

		if shouldContinue
			and not shouldContinue() then

			break
		end

		if not object.Parent then
			return true
		end

		local currentSuccess =
			false

		local detector =
			getClickDetector(
				object
			)

		if detector then

			currentSuccess =
				directClick(
					detector
				)
		end

		if not currentSuccess then

			local prompt =
				getPrompt(
					object
				)

			if prompt then

				currentSuccess =
					firePrompt(
						prompt
					)
			end
		end

		if not currentSuccess
			and allowScreenFallback then

			local part =
				getMainPart(
					object
				)

			if part then

				currentSuccess =
					screenClickPart(
						part
					)
			end
		end

		if currentSuccess then
			clicked =
				true
		end

		if noPickupCooldownEnabled then

			RunService.Heartbeat:Wait()
		else

			task.wait(
				gap
			)
		end
	end

	return clicked
end

--========================================================
-- SHARED PICKUP SYSTEM
--
-- ALL pickup mods use this.
-- This is what makes the mods compatible.
--========================================================

local function pickupObject(
	object,
	options
)
	options =
		options or {}

	if not object
		or not object.Parent then

		return false
	end

	local owner =
		options.Owner
		or "Pickup"

	local shouldContinue =
		options.ShouldContinue

	if shouldContinue
		and not shouldContinue() then

		return false
	end

	if not reserveObject(
		object,
		owner
	) then

		return false
	end

	local previous =
		recentAttempts[object]

	if previous
		and os.clock() - previous
			< 0.07 then

		releaseObject(
			object,
			owner
		)

		return false
	end

	recentAttempts[object] =
		os.clock()

	local result =
		withInteractionLock(
			owner,

			function()

				if shouldContinue
					and not shouldContinue() then

					return false
				end

				if not object.Parent then
					return true
				end

				local character =
					getCharacter()

				local root =
					getRoot()

				local interactionPart =
					options.InteractionPart
					or getMainPart(
						object
					)

				if not character
					or not root
					or not interactionPart then

					return false
				end

				local oldCharacterPivot =
					character:GetPivot()

				local oldObjectPivot =
					getObjectPivot(
						object
					)

				if not oldObjectPivot then
					return false
				end

				local unlockCamera

				if options.LockCamera ~= false then

					unlockCamera =
						lockCamera()
				else

					unlockCamera =
						function()
						end
				end

				------------------------------------------
				-- TP TO REAL ITEM
				------------------------------------------

				if shouldContinue
					and not shouldContinue() then

					unlockCamera()
					return false
				end

				teleportCharacterNear(
					interactionPart.Position,

					options.Distance
						or PICKUP_DISTANCE
				)

				RunService.Heartbeat:Wait()

				if not object.Parent then

					if character.Parent then

						pcall(function()

							character:PivotTo(
								oldCharacterPivot
							)
						end)
					end

					unlockCamera()

					return true
				end

				if shouldContinue
					and not shouldContinue() then

					if character.Parent then

						pcall(function()

							character:PivotTo(
								oldCharacterPivot
							)
						end)
					end

					unlockCamera()

					return false
				end

				------------------------------------------
				-- MOVE OBJECT IN FRONT
				------------------------------------------

				moveObjectInFront(
					object,

					options.FrontDistance
						or 4
				)

				RunService.RenderStepped:Wait()

				------------------------------------------
				-- PICK UP
				------------------------------------------

				local clicked =
					clickObjectBurst(
						object,

						options.ClickCount
							or NORMAL_CLICK_BURST,

						options.ClickGap
							or CLICK_GAP,

						options.AllowScreenFallback,

						shouldContinue
					)

				------------------------------------------
				-- RESTORE OBJECT
				------------------------------------------

				if object
					and object.Parent then

					setObjectPivot(
						object,
						oldObjectPivot
					)
				end

				------------------------------------------
				-- RESTORE PLAYER
				------------------------------------------

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
			end,

			shouldContinue
		)

	releaseObject(
		object,
		owner
	)

	return result
end

--========================================================
-- LIGHTWEIGHT NEAREST SEARCH
--========================================================

local function getNearestFromCache(
	cache,
	excluded,
	owner
)
	local root =
		getRoot()

	if not root then
		return nil
	end

	local closest =
		nil

	local closestDistance =
		math.huge

	for object in pairs(
		cache
	) do

		if object
			and object.Parent
			and (
				not excluded
				or not excluded[object]
			)
			and not isReservedByOther(
				object,
				owner
			) then

			local part =
				getMainPart(
					object
				)

			if part then

				local distance =
					(
						root.Position
						- part.Position
					).Magnitude

				if distance
					< closestDistance then

					closestDistance =
						distance

					closest =
						object
				end
			end
		end
	end

	return closest
end

--========================================================
-- SELL HELPERS
--========================================================

local function addUnique(
	list,
	seen,
	object
)
	if not object
		or seen[object] then

		return
	end

	seen[object] =
		true

	table.insert(
		list,
		object
	)
end

local function getSellSearchObjects(
	sellPart
)
	local list = {}
	local seen = {}

	addUnique(
		list,
		seen,
		sellPart
	)

	for _, object in ipairs(
		sellPart:GetDescendants()
	) do

		addUnique(
			list,
			seen,
			object
		)
	end

	local parent =
		sellPart.Parent

	if parent
		and parent ~= Workspace then

		addUnique(
			list,
			seen,
			parent
		)

		for _, object in ipairs(
			parent:GetDescendants()
		) do

			addUnique(
				list,
				seen,
				object
			)
		end
	end

	return list
end

local function scanSellPart(
	sellPart
)
	if not sellPart
		or not sellPart.Parent then

		return nil
	end

	local result = {
		InteractionPart =
			getMainPart(
				sellPart
			),

		ClickDetector =
			nil,

		Prompt =
			nil
	}

	local main =
		getMainPart(
			sellPart
		)

	local bestScore =
		math.huge

	for _, object in ipairs(
		getSellSearchObjects(
			sellPart
		)
	) do

		if object:IsA("ClickDetector") then

			local part =
				getPartFromObject(
					object
				)

			local score =
				20

			if part == main then

				score =
					0

			elseif part
				and part.Name
					== "SellPart" then

				score =
					1

			elseif object.Parent
				== sellPart then

				score =
					2
			end

			if score < bestScore then

				bestScore =
					score

				result.ClickDetector =
					object

				if part then

					result.InteractionPart =
						part
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

--========================================================
-- SHARED AUTO SELL
--
-- Auto Farm and Auto Sell BOTH use this.
-- Duplicate sell calls are prevented.
--========================================================

local function sellOnePart(
	sellPart,
	owner,
	shouldContinue
)
	if not sellPart
		or not sellPart.Parent then

		return false
	end

	owner =
		owner or "Auto Sell"

	if shouldContinue
		and not shouldContinue() then

		return false
	end

	-- If another mod just sold, count this as handled
	-- instead of teleporting again.

	if os.clock() - lastSellTime
		< SHARED_SELL_COOLDOWN then

		return true
	end

	if not reserveObject(
		sellPart,
		owner
	) then

		return false
	end

	local result =
		withInteractionLock(
			owner,

			function()

				if shouldContinue
					and not shouldContinue() then

					return false
				end

				-- Check again because another sell could
				-- have happened while waiting for the lock.

				if os.clock() - lastSellTime
					< SHARED_SELL_COOLDOWN then

					return true
				end

				local scan =
					scanSellPart(
						sellPart
					)

				if not scan
					or not scan.InteractionPart then

					return false
				end

				local character =
					getCharacter()

				local root =
					getRoot()

				local head =
					getHead()

				local humanoid =
					getHumanoid()

				if not character
					or not root
					or not head then

					return false
				end

				local oldCharacterPivot =
					character:GetPivot()

				local oldSellPivot =
					getObjectPivot(
						sellPart
					)

				if not oldSellPivot then
					return false
				end

				local realSellPosition =
					scan.InteractionPart.Position

				local oldAnchored =
					root.Anchored

				local oldAutoRotate

				if humanoid then

					oldAutoRotate =
						humanoid.AutoRotate
				end

				-- Freeze movement but NO camera lock.

				pcall(function()
					root.Anchored =
						true
				end)

				if humanoid then

					pcall(function()

						humanoid.AutoRotate =
							false
					end)
				end

				------------------------------------------
				-- TP #1
				------------------------------------------

				teleportCharacterNear(
					realSellPosition,
					SELL_FIRST_TP_DISTANCE
				)

				task.wait(0.05)

				------------------------------------------
				-- TP #2
				------------------------------------------

				teleportCharacterNear(
					realSellPosition,
					SELL_SECOND_TP_DISTANCE
				)

				task.wait(0.06)

				------------------------------------------
				-- SELLPART TO HEAD ONCE
				------------------------------------------

				if sellPart.Parent
					and head.Parent then

					setObjectPivot(
						sellPart,

						head.CFrame
						* CFrame.new(
							0,
							0,
							-1.3
						)
					)
				end

				RunService.Heartbeat:Wait()

				local clicked =
					false

				if scan.ClickDetector
					and scan.ClickDetector.Parent then

					for _ = 1, SELL_CLICK_BURST do

						if directClick(
							scan.ClickDetector
						) then

							clicked =
								true
						end

						task.wait(0.025)
					end
				end

				if not clicked
					and scan.Prompt
					and scan.Prompt.Parent then

					if firePrompt(
						scan.Prompt
					) then

						clicked =
							true
					end
				end

				if not clicked then

					local part =
						getMainPart(
							sellPart
						)

					if part then

						for _ = 1, 2 do

							if screenClickPart(
								part
							) then

								clicked =
									true
							end

							task.wait(0.025)
						end
					end
				end

				------------------------------------------
				-- RESTORE SELLPART
				------------------------------------------

				if sellPart.Parent then

					setObjectPivot(
						sellPart,
						oldSellPivot
					)
				end

				------------------------------------------
				-- RESTORE PLAYER
				------------------------------------------

				if character.Parent then

					pcall(function()

						character:PivotTo(
							oldCharacterPivot
						)
					end)

					stopVelocity()
				end

				if root.Parent then

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

				lastSellTime =
					os.clock()

				return clicked
			end,

			shouldContinue
		)

	releaseObject(
		sellPart,
		owner
	)

	return result
end

local function sellNow(
	owner,
	shouldContinue
)
	local sellPart =
		getNearestFromCache(
			sellCache,
			nil,
			owner
		)

	if not sellPart then
		return false
	end

	return sellOnePart(
		sellPart,
		owner,
		shouldContinue
	)
end

--========================================================
-- AUTO FARM
--
-- Compatible with Auto Hay and Auto Sell.
-- One Hay at a time.
-- 5 successful grabs -> sell.
--========================================================

local function startAutoFarm()
	if autoFarmEnabled then
		return
	end

	autoFarmEnabled =
		true

	task.spawn(function()

		while autoFarmEnabled
			and screenGui
			and screenGui.Parent do

			local grabs =
				0

			local attempted =
				{}

			local attempts =
				0

			while autoFarmEnabled
				and grabs
					< FARM_GRABS_BEFORE_SELL
				and attempts < 18 do

				attempts +=
					1

				local hay =
					getNearestFromCache(
						hayCache,
						attempted,
						"Auto Farm"
					)

				if not hay then

					task.wait(0.1)
					break
				end

				attempted[hay] =
					true

				local success =
					pickupObject(
						hay,
						{
							Owner =
								"Auto Farm",

							ShouldContinue =
								function()
									return autoFarmEnabled
								end,

							LockCamera =
								true,

							Distance =
								PICKUP_DISTANCE,

							FrontDistance =
								4,

							ClickCount =
								NORMAL_CLICK_BURST,

							ClickGap =
								0.025
						}
					)

				if success
					or not hay.Parent then

					grabs +=
						1
				end

				task.wait(
					noPickupCooldownEnabled
						and 0.02
						or FARM_BETWEEN_HAY_DELAY
				)
			end

			if autoFarmEnabled
				and grabs
					>= FARM_GRABS_BEFORE_SELL then

				sellNow(
					"Auto Farm Sell",

					function()
						return autoFarmEnabled
					end
				)

				task.wait(
					FARM_AFTER_SELL_DELAY
				)
			else

				task.wait(0.1)
			end
		end
	end)
end

local function stopAutoFarm()
	autoFarmEnabled =
		false
end

--========================================================
-- AUTO PICK UP HAY
--========================================================

local function startAutoHay()
	if autoHayEnabled then
		return
	end

	autoHayEnabled =
		true

	task.spawn(function()

		while autoHayEnabled
			and screenGui
			and screenGui.Parent do

			local hay =
				getNearestFromCache(
					hayCache,
					nil,
					"Auto Hay"
				)

			if hay then

				pickupObject(
					hay,
					{
						Owner =
							"Auto Hay",

						ShouldContinue =
							function()
								return autoHayEnabled
							end,

						LockCamera =
							true,

						Distance =
							PICKUP_DISTANCE,

						FrontDistance =
							4,

						ClickCount =
							NORMAL_CLICK_BURST
					}
				)
			else

				task.wait(0.12)
			end

			task.wait(0.07)
		end
	end)
end

local function stopAutoHay()
	autoHayEnabled =
		false
end

--========================================================
-- AUTO SELL
--========================================================

local function startAutoSell()
	if autoSellEnabled then
		return
	end

	autoSellEnabled =
		true

	task.spawn(function()

		while autoSellEnabled
			and screenGui
			and screenGui.Parent do

			sellNow(
				"Auto Sell",

				function()
					return autoSellEnabled
				end
			)

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
	autoSellEnabled =
		false
end

--========================================================
-- AUTO DIAMOND
--========================================================

local function startAutoDiamond()
	if autoDiamondEnabled then
		return
	end

	autoDiamondEnabled =
		true

	diamondRunId +=
		1

	local runId =
		diamondRunId

	table.clear(
		diamondQueue
	)

	clearQueuedTable(
		diamondQueued
	)

	for diamond in pairs(
		diamondCache
	) do

		if diamond.Parent then

			queueObject(
				diamondQueue,
				diamondQueued,
				diamond
			)
		end
	end

	task.spawn(function()

		while autoDiamondEnabled
			and diamondRunId == runId
			and screenGui
			and screenGui.Parent do

			local diamond =
				table.remove(
					diamondQueue,
					1
				)

			if diamond then

				diamondQueued[diamond] =
					nil

				if diamond.Parent then

					pickupObject(
						diamond,
						{
							Owner =
								"Auto Diamond",

							ShouldContinue =
								function()
									return autoDiamondEnabled
										and diamondRunId == runId
								end,

							LockCamera =
								true,

							Distance =
								2.5,

							FrontDistance =
								4,

							ClickCount =
								SPECIAL_CLICK_BURST,

							ClickGap =
								0.02
						}
					)
				end
			else

				task.wait(0.1)
			end
		end
	end)
end

local function stopAutoDiamond()
	autoDiamondEnabled =
		false

	diamondRunId +=
		1

	table.clear(
		diamondQueue
	)

	clearQueuedTable(
		diamondQueued
	)
end

--========================================================
-- AUTO COLOR / RAINBOW HAY
--========================================================

local function startAutoColorHay()
	if autoColorHayEnabled then
		return
	end

	autoColorHayEnabled =
		true

	colorHayRunId +=
		1

	local runId =
		colorHayRunId

	table.clear(
		colorHayQueue
	)

	clearQueuedTable(
		colorHayQueued
	)

	for hay, state in pairs(
		colorHayState
	) do

		if hay.Parent
			and state.LastChanged > 0
			and os.clock()
				- state.LastChanged < 1.5 then

			queueObject(
				colorHayQueue,
				colorHayQueued,
				hay
			)
		end
	end

	task.spawn(function()

		while autoColorHayEnabled
			and colorHayRunId == runId
			and screenGui
			and screenGui.Parent do

			local hay =
				table.remove(
					colorHayQueue,
					1
				)

			if hay then

				colorHayQueued[hay] =
					nil

				if hay.Parent then

					pickupObject(
						hay,
						{
							Owner =
								"Color Hay",

							ShouldContinue =
								function()

									return autoColorHayEnabled
										and colorHayRunId == runId
								end,

							LockCamera =
								true,

							Distance =
								2.5,

							FrontDistance =
								4,

							ClickCount =
								SPECIAL_CLICK_BURST,

							ClickGap =
								0.02,

							-- Prevent mouse spam from
							-- fighting with the GUI.
							AllowScreenFallback =
								false
						}
					)
				end
			else

				task.wait(0.07)
			end
		end
	end)
end

local function stopAutoColorHay()
	autoColorHayEnabled =
		false

	colorHayRunId +=
		1

	table.clear(
		colorHayQueue
	)

	clearQueuedTable(
		colorHayQueued
	)
end

--========================================================
-- AUTO FIND KEY - BETA
--========================================================

local function startAutoFindKey()
	if autoFindKeyEnabled then
		return
	end

	autoFindKeyEnabled =
		true

	keyRunId +=
		1

	local runId =
		keyRunId

	table.clear(
		keyQueue
	)

	clearQueuedTable(
		keyQueued
	)

	-- Existing keys were already cached by the startup
	-- scan, so no new full Workspace scan is needed.

	for key in pairs(
		keyCache
	) do

		if key.Parent then

			queueObject(
				keyQueue,
				keyQueued,
				key
			)
		end
	end

	task.spawn(function()

		while autoFindKeyEnabled
			and keyRunId == runId
			and screenGui
			and screenGui.Parent do

			local key =
				table.remove(
					keyQueue,
					1
				)

			if key then

				keyQueued[key] =
					nil

				if key.Parent then

					local success =
						pickupObject(
							key,
							{
								Owner =
									"Auto Key",

								ShouldContinue =
									function()

										return autoFindKeyEnabled
											and keyRunId == runId
									end,

								LockCamera =
									true,

								Distance =
									2.1,

								FrontDistance =
									3.8,

								ClickCount =
									KEY_CLICK_BURST,

								ClickGap =
									0.02
							}
						)

					if success
						and not key.Parent then

						notify(
							"Auto Find Key",
							"Key picked up.",
							2
						)
					end

					if key.Parent
						and autoFindKeyEnabled
						and keyRunId == runId then

						task.delay(
							0.55,
							function()

								if autoFindKeyEnabled
									and keyRunId == runId
									and key.Parent then

									queueObject(
										keyQueue,
										keyQueued,
										key
									)
								end
							end
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
	autoFindKeyEnabled =
		false

	keyRunId +=
		1

	table.clear(
		keyQueue
	)

	clearQueuedTable(
		keyQueued
	)
end

--========================================================
-- AUTO FIND NEEDLE - BETA
--========================================================

local function startAutoFindNeedle()
	if autoFindNeedleEnabled then
		return
	end

	autoFindNeedleEnabled =
		true

	needleRunId +=
		1

	local runId =
		needleRunId

	table.clear(
		needleQueue
	)

	clearQueuedTable(
		needleQueued
	)

	for needle in pairs(
		needleCache
	) do

		if needle.Parent then

			queueObject(
				needleQueue,
				needleQueued,
				needle
			)
		end
	end

	task.spawn(function()

		while autoFindNeedleEnabled
			and needleRunId == runId
			and screenGui
			and screenGui.Parent do

			local needle =
				table.remove(
					needleQueue,
					1
				)

			if needle then

				needleQueued[needle] =
					nil

				if needle.Parent then

					local success =
						pickupObject(
							needle,
							{
								Owner =
									"Auto Needle",

								ShouldContinue =
									function()

										return autoFindNeedleEnabled
											and needleRunId == runId
									end,

								LockCamera =
									true,

								Distance =
									2.1,

								FrontDistance =
									3.8,

								ClickCount =
									NEEDLE_CLICK_BURST,

								ClickGap =
									0.02
							}
						)

					if success
						and not needle.Parent then

						notify(
							"Auto Find Needle",
							"Needle picked up.",
							2
						)
					end

					if needle.Parent
						and autoFindNeedleEnabled
						and needleRunId == runId then

						task.delay(
							0.55,
							function()

								if autoFindNeedleEnabled
									and needleRunId == runId
									and needle.Parent then

									queueObject(
										needleQueue,
										needleQueued,
										needle
									)
								end
							end
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
	autoFindNeedleEnabled =
		false

	needleRunId +=
		1

	table.clear(
		needleQueue
	)

	clearQueuedTable(
		needleQueued
	)
end

--========================================================
-- INF RANGE
--
-- Compatible with all auto pickup mods.
-- If an auto mod already owns the item, Inf Range waits
-- instead of fighting it.
--========================================================

local function getTrackedPickupFromTarget(
	target
)
	if not target then
		return nil
	end

	local current =
		target

	while current
		and current ~= Workspace do

		if hayCache[current]
			or diamondCache[current]
			or keyCache[current]
			or needleCache[current] then

			return current
		end

		if current.Name == "HayPiece" then

			hayCache[current] =
				true

			applySettingsToObject(
				current
			)

			return current
		end

		if current.Name == "Diamond" then

			diamondCache[current] =
				true

			applySettingsToObject(
				current
			)

			return current
		end

		if current.Name == "Key" then

			local key =
				resolveNamedRoot(
					current,
					isKeyName
				)
				or current

			keyCache[key] =
				true

			applySettingsToObject(
				key
			)

			return key
		end

		if isNeedleName(
			current.Name
		) then

			local needle =
				resolveNamedRoot(
					current,
					isNeedleName
				)
				or current

			needleCache[needle] =
				true

			applySettingsToObject(
				needle
			)

			return needle
		end

		current =
			current.Parent
	end

	return nil
end

local function raycastScreenPosition(
	x,
	y
)
	Camera =
		Workspace.CurrentCamera
		or Camera

	if not Camera then
		return nil
	end

	local ray =
		Camera:ViewportPointToRay(
			x,
			y
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

	return result
		and result.Instance
		or nil
end

local function getInputTarget(input)
	if input.UserInputType
		== Enum.UserInputType.Touch then

		return raycastScreenPosition(
			input.Position.X,
			input.Position.Y
		)
	end

	if Mouse
		and Mouse.Target then

		return Mouse.Target
	end

	local position =
		UserInputService:GetMouseLocation()

	return raycastScreenPosition(
		position.X,
		position.Y
	)
end

track(
	UserInputService.InputBegan:Connect(function(
		input,
		processed
	)

		if processed
			or not infRangeEnabled then

			return
		end

		if input.UserInputType
				~= Enum.UserInputType.MouseButton1
			and input.UserInputType
				~= Enum.UserInputType.Touch then

			return
		end

		local target =
			getInputTarget(
				input
			)

		local pickup =
			getTrackedPickupFromTarget(
				target
			)

		if not pickup then
			return
		end

		if isReservedByOther(
			pickup,
			"Inf Range"
		) then

			return
		end

		task.spawn(function()

			pickupObject(
				pickup,
				{
					Owner =
						"Inf Range",

					ShouldContinue =
						function()
							return infRangeEnabled
						end,

					LockCamera =
						true,

					Distance =
						2.2,

					FrontDistance =
						3.8,

					ClickCount =
						SPECIAL_CLICK_BURST,

					ClickGap =
						0.02
				}
			)
		end)
	end)
)

--========================================================
-- UFO
--========================================================

local function startUfoEvent()
	local button =
		getNearestFromCache(
			ufoCache,
			nil,
			"UFO"
		)

	if not button then

		notify(
			"UFO Event",
			"UFO button was not found.",
			3,
			"danger"
		)

		return
	end

	task.spawn(function()

		if not reserveObject(
			button,
			"UFO"
		) then

			return
		end

		withInteractionLock(
			"UFO",

			function()

				local character =
					getCharacter()

				local root =
					getRoot()

				local part =
					getMainPart(
						button
					)

				if not character
					or not root
					or not part then

					return false
				end

				local oldPivot =
					character:GetPivot()

				local oldAnchored =
					root.Anchored

				pcall(function()
					root.Anchored =
						true
				end)

				teleportCharacterNear(
					part.Position,
					UFO_DISTANCE
				)

				task.wait(0.08)

				clickObjectBurst(
					button,
					SPECIAL_CLICK_BURST,
					0.025,
					true
				)

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

				return true
			end
		)

		releaseObject(
			button,
			"UFO"
		)
	end)
end

--========================================================
-- PLAYER MODS
--========================================================

local function setSpeed(state)
	speedEnabled =
		state

	if not state then

		for humanoid, oldSpeed in pairs(
			originalWalkSpeeds
		) do

			if humanoid
				and humanoid.Parent then

				humanoid.WalkSpeed =
					oldSpeed
			end
		end
	end
end

local function setThirdPerson(state)
	thirdPersonEnabled =
		state

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

		flyConnection =
			nil
	end

	if flyVelocity then
		flyVelocity:Destroy()
		flyVelocity =
			nil
	end

	if flyGyro then
		flyGyro:Destroy()
		flyGyro =
			nil
	end

	if flyHumanoid
		and flyHumanoid.Parent then

		flyHumanoid.PlatformStand =
			flyOldPlatformStand == true
	end

	flyHumanoid =
		nil

	flyOldPlatformStand =
		nil
end

local function attachFly()
	clearFly()

	if not flyEnabled then
		return
	end

	local root =
		getRoot()

	local humanoid =
		getHumanoid()

	if not root
		or not humanoid then

		return
	end

	flyHumanoid =
		humanoid

	flyOldPlatformStand =
		humanoid.PlatformStand

	humanoid.PlatformStand =
		true

	flyVelocity =
		create("BodyVelocity", {
			MaxForce =
				Vector3.new(
					1e9,
					1e9,
					1e9
				),

			P =
				1250,

			Velocity =
				Vector3.zero,

			Parent =
				root
		})

	flyGyro =
		create("BodyGyro", {
			MaxTorque =
				Vector3.new(
					1e9,
					1e9,
					1e9
				),

			P =
				5000,

			D =
				100,

			CFrame =
				root.CFrame,

			Parent =
				root
		})

	flyConnection =
		RunService.RenderStepped:Connect(function()

			if not flyEnabled then
				return
			end

			-- Fly stays enabled, but pauses while another
			-- mod temporarily moves the character.

			if interactionBusy then

				if flyVelocity then

					flyVelocity.Velocity =
						Vector3.zero
				end

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

			if movement.Magnitude
				< 0.05 then

				movement =
					humanoid.MoveDirection
			end

			local vertical =
				0

			if UserInputService:IsKeyDown(
				Enum.KeyCode.Space
			) then

				vertical +=
					1
			end

			if UserInputService:IsKeyDown(
				Enum.KeyCode.LeftControl
			)
				or UserInputService:IsKeyDown(
					Enum.KeyCode.C
				) then

				vertical -=
					1
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

			local look =
				Vector3.new(
					Camera.CFrame.LookVector.X,
					0,
					Camera.CFrame.LookVector.Z
				)

			if look.Magnitude > 0.05 then

				flyGyro.CFrame =
					CFrame.lookAt(
						root.Position,

						root.Position
						+ look.Unit
					)
			end
		end)

	track(
		flyConnection
	)
end

local function setFly(state)
	flyEnabled =
		state

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

		if infiniteJumpEnabled
			and not interactionBusy then

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

--========================================================
-- PART NAME HOVER
--========================================================

local function stopPartNameHover()
	partNameHoverEnabled =
		false

	if partNameHoverConnection then

		partNameHoverConnection:Disconnect()

		partNameHoverConnection =
			nil
	end

	if partNameHoverLabel then

		partNameHoverLabel:Destroy()

		partNameHoverLabel =
			nil
	end
end

local function startPartNameHover()
	if partNameHoverEnabled then
		return
	end

	partNameHoverEnabled =
		true

	partNameHoverLabel =
		create("TextLabel", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.2,

			BorderSizePixel =
				0,

			Size =
				UDim2.fromOffset(
					230,
					34
				),

			Font =
				Enum.Font.GothamSemibold,

			Text =
				"",

			TextColor3 =
				Colors.Text,

			TextSize =
				12,

			Visible =
				false,

			ZIndex =
				9999,

			Parent =
				screenGui
		})

	corner(
		partNameHoverLabel,
		9
	)

	stroke(
		partNameHoverLabel,
		Colors.Stroke,
		0.6,
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

	track(
		partNameHoverConnection
	)
end

--========================================================
-- CLEANUP
--========================================================

local function cleanupAll(
	hardUnload
)
	if cleaning then
		return
	end

	cleaning =
		true

	autoFarmEnabled =
		false

	autoHayEnabled =
		false

	autoSellEnabled =
		false

	autoDiamondEnabled =
		false

	autoColorHayEnabled =
		false

	autoFindKeyEnabled =
		false

	autoFindNeedleEnabled =
		false

	diamondRunId +=
		1

	colorHayRunId +=
		1

	keyRunId +=
		1

	needleRunId +=
		1

	noPickupCooldownEnabled =
		false

	infRangeEnabled =
		false

	speedEnabled =
		false

	infiniteJumpEnabled =
		false

	setThirdPerson(
		false
	)

	setFly(
		false
	)

	stopPartNameHover()
	restoreInteractionSettings()

	if antiAfkConnection then

		pcall(function()
			antiAfkConnection:Disconnect()
		end)

		antiAfkConnection =
			nil
	end

	for humanoid, oldSpeed in pairs(
		originalWalkSpeeds
	) do

		if humanoid
			and humanoid.Parent then

			pcall(function()

				humanoid.WalkSpeed =
					oldSpeed
			end)
		end
	end

	for _, connection in ipairs(
		connections
	) do

		pcall(function()
			connection:Disconnect()
		end)
	end

	table.clear(
		connections
	)

	if screenGui then

		screenGui:Destroy()

		screenGui =
			nil
	end

	if hardUnload then

		_G.THHGrowersHardUnloaded =
			true
	end

	_G.THHGrowersCleanup =
		nil
end

_G.THHGrowersCleanup = function()
	cleanupAll(false)
end

--========================================================
-- GUI ROOT
--========================================================

local guiParent

if type(gethui)
	== "function" then

	local success, result =
		pcall(gethui)

	if success
		and result then

		guiParent =
			result
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
		Name =
			"THHGrowers",

		ResetOnSpawn =
			false,

		IgnoreGuiInset =
			true,

		DisplayOrder =
			999999,

		ZIndexBehavior =
			Enum.ZIndexBehavior.Sibling,

		Parent =
			guiParent
	})

--========================================================
-- NOTIFICATIONS
--========================================================

notificationHolder =
	create("Frame", {
		BackgroundTransparency =
			1,

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
				300,
				500
			),

		ZIndex =
			800,

		Parent =
			screenGui
	})

create("UIListLayout", {
	Padding =
		UDim.new(
			0,
			7
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
					0.14,

				Size =
					UDim2.fromOffset(
						290,
						72
					),

				ZIndex =
					801,

				Parent =
					notificationHolder
			})

		corner(
			card,
			11
		)

		stroke(
			card,

			kind == "danger"
				and Colors.Danger
				or Colors.Stroke,

			0.5,
			1
		)

		create("TextLabel", {
			BackgroundTransparency =
				1,

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

			Text =
				title,

			TextColor3 =
				Colors.Text,

			TextSize =
				13,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			ZIndex =
				802,

			Parent =
				card
		})

		create("TextLabel", {
			BackgroundTransparency =
				1,

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

			Text =
				message,

			TextColor3 =
				Colors.SubText,

			TextSize =
				10,

			TextWrapped =
				true,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			ZIndex =
				802,

			Parent =
				card
		})

		task.delay(
			duration or 2.5,
			function()

				if card
					and card.Parent then

					card:Destroy()
				end
			end
		)
	end

--========================================================
-- AUTH
--========================================================

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

				debug =
					false
			})
		end)

	if not clientOk then

		return nil,
			"Could not initialize Vampauth."
	end

	VampauthClient =
		client

	return client
end

local function validateKey(key)
	local client, errorText =
		getVampauthClient()

	if not client then
		return false, errorText
	end

	local success, valid, result =
		pcall(function()

			local okay, data =
				client:Check(
					key
				)

			return okay, data
		end)

	if not success then

		return false,
			"Validation failed."
	end

	if valid then
		return true
	end

	if type(result)
		== "table" then

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

--========================================================
-- DRAG
--========================================================

local function makeDraggable(
	frame,
	handle
)
	local dragging =
		false

	local dragInput
	local dragStart
	local startPosition

	track(
		handle.InputBegan:Connect(function(input)

			if input.UserInputType
					== Enum.UserInputType.MouseButton1
				or input.UserInputType
					== Enum.UserInputType.Touch then

				dragging =
					true

				dragStart =
					input.Position

				startPosition =
					frame.Position
			end
		end)
	)

	track(
		handle.InputChanged:Connect(function(input)

			if input.UserInputType
					== Enum.UserInputType.MouseMovement
				or input.UserInputType
					== Enum.UserInputType.Touch then

				dragInput =
					input
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

				dragging =
					false
			end
		end)
	)
end

--========================================================
-- DEVICE DRAWINGS
--========================================================

local function buildPcDrawing(parent)
	local monitor =
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
					-65,
					0,
					16
				),

			Size =
				UDim2.fromOffset(
					130,
					78
				),

			Parent =
				parent
		})

	corner(
		monitor,
		9
	)

	stroke(
		monitor,
		Color3.fromRGB(
			235,
			237,
			241
		),
		0.38,
		2
	)

	local display =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					99,
					103,
					113
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

			Parent =
				monitor
		})

	corner(
		display,
		5
	)

	addGlassGradient(
		display
	)

	create("Frame", {
		BackgroundColor3 =
			Color3.fromRGB(
				220,
				223,
				229
			),

		BorderSizePixel =
			0,

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
				22
			),

		Parent =
			monitor
	})

	local stand =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					220,
					223,
					229
				),

			BorderSizePixel =
				0,

			Position =
				UDim2.new(
					0.5,
					-32,
					1,
					18
				),

			Size =
				UDim2.fromOffset(
					64,
					7
				),

			Parent =
				monitor
		})

	corner(
		stand,
		4
	)
end

local function buildPhoneDrawing(parent)
	local body =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					38,
					41,
					47
				),

			Position =
				UDim2.new(
					0.5,
					-41,
					0,
					9
				),

			Size =
				UDim2.fromOffset(
					82,
					130
				),

			Parent =
				parent
		})

	corner(
		body,
		16
	)

	stroke(
		body,
		Color3.fromRGB(
			235,
			237,
			241
		),
		0.38,
		2
	)

	local display =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					99,
					103,
					113
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
					-29
				),

			Parent =
				body
		})

	corner(
		display,
		8
	)

	addGlassGradient(
		display
	)

	local speaker =
		create("Frame", {
			BackgroundColor3 =
				Colors.Text,

			BorderSizePixel =
				0,

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

			Parent =
				body
		})

	corner(
		speaker,
		3
	)

	local home =
		create("Frame", {
			BackgroundColor3 =
				Colors.Text,

			BorderSizePixel =
				0,

			Position =
				UDim2.new(
					0.5,
					-14,
					1,
					-7
				),

			Size =
				UDim2.fromOffset(
					28,
					3
				),

			Parent =
				body
		})

	corner(
		home,
		3
	)
end

--========================================================
-- MAIN MENU
--========================================================

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
				0.14,

			BorderSizePixel =
				0,

			ClipsDescendants =
				true,

			Parent =
				screenGui
		})

	corner(
		mainFrame,
		16
	)

	stroke(
		mainFrame,
		Colors.Stroke,
		0.42,
		1.2
	)

	addGlassGradient(
		mainFrame
	)

	if phoneMode then

		mainScale =
			create("UIScale", {
				Scale =
					1,

				Parent =
					mainFrame
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

	--====================================================
	-- SIDEBAR
	--====================================================

	local sidebar =
		create("Frame", {
			BackgroundColor3 =
				Colors.Sidebar,

			BackgroundTransparency =
				0.15,

			BorderSizePixel =
				0,

			Size =
				UDim2.new(
					0,
					sidebarWidth,
					1,
					0
				),

			Parent =
				mainFrame
		})

	local gameIcon =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.CardDark,

			Position =
				UDim2.fromOffset(
					12,
					12
				),

			Size =
				UDim2.fromOffset(
					44,
					44
				),

			Image =
				GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			Parent =
				sidebar
		})

	corner(
		gameIcon,
		10
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				64,
				12
			),

		Size =
			UDim2.new(
				1,
				-70,
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

		TextSize =
			14,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			sidebar
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				64,
				34
			),

		Size =
			UDim2.new(
				1,
				-70,
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

		TextSize =
			9,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			sidebar
	})

	local profile =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.35,

			Position =
				UDim2.fromOffset(
					10,
					68
				),

			Size =
				UDim2.new(
					1,
					-20,
					0,
					62
				),

			Parent =
				sidebar
		})

	corner(
		profile,
		11
	)

	local avatar =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.CardDark,

			Position =
				UDim2.fromOffset(
					9,
					10
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

			Parent =
				profile
		})

	corner(
		avatar,
		21
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				58,
				10
			),

		Size =
			UDim2.new(
				1,
				-64,
				0,
				20
			),

		Font =
			Enum.Font.GothamSemibold,

		Text =
			LocalPlayer.DisplayName,

		TextColor3 =
			Colors.Text,

		TextSize =
			10,

		TextTruncate =
			Enum.TextTruncate.AtEnd,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			profile
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				58,
				31
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
			"@" .. LocalPlayer.Name,

		TextColor3 =
			Colors.SubText,

		TextSize =
			8,

		TextTruncate =
			Enum.TextTruncate.AtEnd,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			profile
	})

	local navHolder =
		create("ScrollingFrame", {
			BackgroundTransparency =
				1,

			BorderSizePixel =
				0,

			Position =
				UDim2.fromOffset(
					10,
					142
				),

			Size =
				UDim2.new(
					1,
					-20,
					1,
					-152
				),

			CanvasSize =
				UDim2.new(),

			AutomaticCanvasSize =
				Enum.AutomaticSize.Y,

			ScrollBarThickness =
				2,

			Parent =
				sidebar
		})

	create("UIListLayout", {
		Padding =
			UDim.new(
				0,
				6
			),

		Parent =
			navHolder
	})

	--====================================================
	-- CONTENT
	--====================================================

	local content =
		create("Frame", {
			BackgroundTransparency =
				1,

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

			Parent =
				mainFrame
		})

	local header =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.7,

			Size =
				UDim2.new(
					1,
					0,
					0,
					62
				),

			Parent =
				content
		})

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				16,
				8
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

		Text =
			"THH MENU",

		TextColor3 =
			Colors.Text,

		TextSize =
			18,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			header
	})

	local pageTitle =
		create("TextLabel", {
			BackgroundTransparency =
				1,

			Position =
				UDim2.fromOffset(
					16,
					34
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

			Text =
				"Home",

			TextColor3 =
				Colors.SubText,

			TextSize =
				10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				header
		})

	local hide =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.25,

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
					13
				),

			Size =
				UDim2.fromOffset(
					36,
					36
				),

			Font =
				Enum.Font.GothamBold,

			Text =
				"—",

			TextColor3 =
				Colors.Text,

			TextSize =
				16,

			Parent =
				header
		})

	corner(
		hide,
		10
	)

	local pageHolder =
		create("Frame", {
			BackgroundTransparency =
				1,

			Position =
				UDim2.fromOffset(
					0,
					62
				),

			Size =
				UDim2.new(
					1,
					0,
					1,
					-62
				),

			Parent =
				content
		})

	local pages = {}
	local navButtons = {}

	local function createPage(name)

		local page =
			create("ScrollingFrame", {
				BackgroundTransparency =
					1,

				BorderSizePixel =
					0,

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

				Visible =
					false,

				Parent =
					pageHolder
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

			Parent =
				page
		})

		pages[name] =
			page

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

			button.BackgroundTransparency =
				selected
					and 0.25
					or 1

			button.BackgroundColor3 =
				selected
					and Colors.Card
					or Colors.Sidebar

			button.TextColor3 =
				selected
					and Colors.Text
					or Colors.SubText
		end

		pageTitle.Text =
			name
	end

	local function createNav(name)

		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Sidebar,

				BackgroundTransparency =
					1,

				BorderSizePixel =
					0,

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

				Text =
					name,

				TextColor3 =
					Colors.SubText,

				TextSize =
					11,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				Parent =
					navHolder
			})

		padding(
			button,
			12,
			6,
			0,
			0
		)

		corner(
			button,
			9
		)

		track(
			button.Activated:Connect(function()

				setPage(
					name
				)
			end)
		)

		navButtons[name] =
			button
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
					0.28,

				BorderSizePixel =
					0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						0
					),

				AutomaticSize =
					Enum.AutomaticSize.Y,

				Parent =
					parent
			})

		corner(
			frame,
			12
		)

		stroke(
			frame,
			Colors.Stroke,
			0.68,
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

			Parent =
				frame
		})

		create("TextLabel", {
			BackgroundTransparency =
				1,

			Size =
				UDim2.new(
					1,
					0,
					0,
					22
				),

			Font =
				Enum.Font.GothamSemibold,

			Text =
				title,

			TextColor3 =
				Colors.Text,

			TextSize =
				14,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				frame
		})

		if description then

			create("TextLabel", {
				BackgroundTransparency =
					1,

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

				Text =
					description,

				TextColor3 =
					Colors.SubText,

				TextSize =
					10,

				TextWrapped =
					true,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				Parent =
					frame
			})
		end

		return frame
	end

	--====================================================
	-- TOGGLE
	--====================================================

	local function createToggle(
		parent,
		title,
		description,
		callback,
		showBeta
	)
		local enabled =
			false

		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.CardDark,

				BackgroundTransparency =
					0.38,

				BorderSizePixel =
					0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						phoneMode
							and 62
							or 52
					),

				Text =
					"",

				AutoButtonColor =
					false,

				Parent =
					parent
			})

		corner(
			button,
			10
		)

		stroke(
			button,
			Colors.Stroke,
			0.82,
			1
		)

		create("TextLabel", {
			BackgroundTransparency =
				1,

			Position =
				UDim2.fromOffset(
					12,
					7
				),

			Size =
				UDim2.new(
					1,
					showBeta
						and -140
						or -90,
					0,
					20
				),

			Font =
				Enum.Font.GothamSemibold,

			Text =
				title,

			TextColor3 =
				Colors.Text,

			TextSize =
				12,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				button
		})

		if showBeta then

			local beta =
				create("TextLabel", {
					BackgroundColor3 =
						Colors.BetaDark,

					BackgroundTransparency =
						0.08,

					Position =
						UDim2.new(
							1,
							-132,
							0,
							8
						),

					Size =
						UDim2.fromOffset(
							43,
							18
						),

					Font =
						Enum.Font.GothamBold,

					Text =
						"BETA",

					TextColor3 =
						Colors.Beta,

					TextSize =
						9,

					Parent =
						button
				})

			corner(
				beta,
				6
			)

			stroke(
				beta,
				Colors.Beta,
				0.45,
				1
			)
		end

		create("TextLabel", {
			BackgroundTransparency =
				1,

			Position =
				UDim2.fromOffset(
					12,
					29
				),

			Size =
				UDim2.new(
					1,
					-96,
					0,
					18
				),

			Font =
				Enum.Font.Gotham,

			Text =
				description,

			TextColor3 =
				Colors.SubText,

			TextSize =
				9,

			TextTruncate =
				Enum.TextTruncate.AtEnd,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				button
		})

		local switch =
			create("Frame", {
				BackgroundColor3 =
					Color3.fromRGB(
						117,
						120,
						128
					),

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

				Parent =
					button
			})

		corner(
			switch,
			14
		)

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

				Parent =
					switch
			})

		corner(
			knob,
			10
		)

		local function updateVisual()

			switch.BackgroundColor3 =
				enabled
					and Colors.AccentDark
					or Color3.fromRGB(
						117,
						120,
						128
					)

			knob.BackgroundColor3 =
				enabled
					and Colors.Accent
					or Colors.Text

			knob.Position =
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
					)
		end

		track(
			button.Activated:Connect(function()

				enabled =
					not enabled

				updateVisual()

				task.spawn(
					callback,
					enabled
				)
			end)
		)
	end

	--====================================================
	-- ACTION
	--====================================================

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

				BorderSizePixel =
					0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						50
					),

				Text =
					"",

				Parent =
					parent
			})

		corner(
			button,
			10
		)

		create("TextLabel", {
			BackgroundTransparency =
				1,

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

			Text =
				title,

			TextColor3 =
				danger
					and Colors.Danger
					or Colors.Text,

			TextSize =
				12,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				button
		})

		create("TextLabel", {
			BackgroundTransparency =
				1,

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

			Text =
				description,

			TextColor3 =
				Colors.SubText,

			TextSize =
				9,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				button
		})

		track(
			button.Activated:Connect(
				callback
			)
		)
	end

	--====================================================
	-- PAGES
	--====================================================

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
		"You can now use multiple mods at the same time."
	)

	--====================================================
	-- MODS
	--====================================================

	local farming =
		section(
			mods,
			"Farming",
			"Mods that automatically collect and sell Hay."
		)

	createToggle(
		farming,
		"Auto Farm",
		"Automatically picks up 5 HayPieces, sells them, then repeats.",
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
		"Automatically picks up HayPieces.",
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
		"Automatically sells your Hay for you.",
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
		"Makes pickups happen faster by reducing local wait time.",
		function(state)

			noPickupCooldownEnabled =
				state

			refreshCachedSettings()
		end,
		true
	)

	createToggle(
		farming,
		"Inf Range",
		"Lets you click pickups from far away.",
		function(state)

			infRangeEnabled =
				state

			refreshCachedSettings()
		end
	)

	local special =
		section(
			mods,
			"Special Pickups",
			"Mods that automatically pick up special items."
		)

	createToggle(
		special,
		"Auto Pick Up Diamond",
		"Automatically picks up Diamonds.",
		function(state)

			if state then
				startAutoDiamond()
			else
				stopAutoDiamond()
			end
		end
	)

	createToggle(
		special,
		"Auto Pick Up Color Hay",
		"Automatically picks up color-changing HayPieces.",
		function(state)

			if state then
				startAutoColorHay()
			else
				stopAutoColorHay()
			end
		end
	)

	--====================================================
	-- PLAYER
	--====================================================

	local movement =
		section(
			player,
			"Movement",
			"Mods that change how your character moves."
		)

	createToggle(
		movement,
		"Speed",
		"Makes your character walk faster.",
		function(state)

			setSpeed(
				state
			)
		end
	)

	createToggle(
		movement,
		"Infinite Jump",
		"Lets you jump again while you're already in the air.",
		function(state)

			infiniteJumpEnabled =
				state
		end
	)

	createToggle(
		movement,
		"Unlock 3rd Person",
		"Lets you zoom out and use a normal third-person camera.",
		function(state)

			setThirdPerson(
				state
			)
		end
	)

	createToggle(
		movement,
		"Fly",
		"Lets your character fly around the map.",
		function(state)

			setFly(
				state
			)
		end
	)

	--====================================================
	-- EVENT
	--====================================================

	local eventTools =
		section(
			event,
			"Events",
			"Mods for the UFO, Key and Needle."
		)

	createAction(
		eventTools,
		"Start UFO Event",
		"Automatically goes to the UFO button and clicks it.",
		startUfoEvent
	)

	createToggle(
		eventTools,
		"Auto Find Key",
		"Automatically finds and picks up the Key.",
		function(state)

			if state then

				startAutoFindKey()

				notify(
					"Auto Find Key",
					"Looking for Key.",
					2
				)
			else

				stopAutoFindKey()
			end
		end,
		true
	)

	createToggle(
		eventTools,
		"Auto Find Needle",
		"Automatically finds and picks up the Needle.",
		function(state)

			if state then

				startAutoFindNeedle()

				notify(
					"Auto Find Needle",
					"Looking for Needle.",
					2
				)
			else

				stopAutoFindNeedle()
			end
		end,
		true
	)

	local ids =
		section(
			event,
			"Place IDs",
			"Copy the Place ID for each part of the game."
		)

	createAction(
		ids,
		"Copy Main Place ID",
		"Copies the main game's Place ID.",
		function()

			copyText(
				SEARCH_FOR_NEEDLE_PLACE_ID
			)
		end
	)

	createAction(
		ids,
		"Copy Farmhouse ID",
		"Copies the Farmhouse Place ID.",
		function()

			copyText(
				FARMHOUSE_PLACE_ID
			)
		end
	)

	createAction(
		ids,
		"Copy Basement ID",
		"Copies the Basement Place ID.",
		function()

			copyText(
				BASEMENT_PLACE_ID
			)
		end
	)

	--====================================================
	-- UTILITY
	--====================================================

	local tools =
		section(
			utility,
			"Tools",
			"Extra tools that can help while playing."
		)

	createToggle(
		tools,
		"Part Name Hover",
		"Shows the name of the part under your mouse.",
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
		"Stops Roblox from kicking you for being idle.",
		function(state)

			antiAfkEnabled =
				state

			if antiAfkConnection then

				antiAfkConnection:Disconnect()

				antiAfkConnection =
					nil
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
			"Buttons for your character, server and menu."
		)

	createAction(
		server,
		"Rejoin Server",
		"Rejoins the server you're currently in.",
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
		"Resets your character.",
		function()

			local humanoid =
				getHumanoid()

			if humanoid then

				humanoid.Health =
					0
			end
		end
	)

	createAction(
		server,
		"Copy Server ID",
		"Copies the ID of the server you're in.",
		function()

			copyText(
				game.JobId
			)
		end
	)

	createAction(
		server,
		"Leave Server",
		"Leaves the server.",
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
		"Turns off all mods and removes THH HUB.",
		function()

			cleanupAll(
				true
			)
		end,
		true
	)

	--====================================================
	-- UPDATES
	--====================================================

	section(
		updates,
		"Compatibility Update",
		"• Mods can now stay enabled together\n"
		.. "• Added one shared interaction lock\n"
		.. "• Added item reservations so two mods do not grab the same item\n"
		.. "• Auto Farm and Auto Hay can run together\n"
		.. "• Auto Farm and Auto Sell share the same sell system\n"
		.. "• Duplicate sells are prevented\n"
		.. "• Diamond, Color Hay, Key and Needle wait for other interactions\n"
		.. "• Inf Range no longer fights Auto Pickup mods\n"
		.. "• Fly pauses only while another mod is teleporting\n"
		.. "• Turning off Color Hay, Key, Needle or Diamond cancels their queued work\n"
		.. "• Removed extra Workspace scans when enabling Key or Needle\n"
		.. "• Kept one-object-at-a-time farming to reduce lag"
	)

	--====================================================
	-- SHOW / HIDE
	--====================================================

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
		hide.Activated:Connect(function()

			setVisible(
				false
			)
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

				Image =
					GAME_ICON,

				ScaleType =
					Enum.ScaleType.Crop,

				Visible =
					false,

				ZIndex =
					10000,

				Parent =
					screenGui
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
			floatingButton.Activated:Connect(function()

				setVisible(
					true
				)
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

	setPage(
		"Home"
	)

	notify(
		"THH HUB",

		phoneMode
			and "Phone mode loaded."
			or "PC mode loaded.",

		2
	)
end

--========================================================
-- DEVICE CHOOSER
--========================================================

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

			Parent =
				screenGui
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

			BorderSizePixel =
				0,

			Parent =
				overlay
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

	addGlassGradient(
		panel
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

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

		TextSize =
			20,

		Parent =
			panel
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

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

		TextSize =
			11,

		Parent =
			panel
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

			Text =
				"",

			Parent =
				panel
		})

	corner(
		pc,
		15
	)

	stroke(
		pc,
		Colors.Stroke,
		0.58,
		1
	)

	buildPcDrawing(
		pc
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-56
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

		Text =
			"PC",

		TextColor3 =
			Colors.Text,

		TextSize =
			16,

		Parent =
			pc
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-31
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

		TextSize =
			9,

		Parent =
			pc
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

			Text =
				"",

			Parent =
				panel
		})

	corner(
		phone,
		15
	)

	stroke(
		phone,
		Colors.Stroke,
		0.58,
		1
	)

	buildPhoneDrawing(
		phone
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-56
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

		Text =
			"PHONE",

		TextColor3 =
			Colors.Text,

		TextSize =
			16,

		Parent =
			phone
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-31
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

		TextSize =
			9,

		Parent =
			phone
	})

	local selected =
		false

	local function choose(mode)

		if selected then
			return
		end

		selected =
			true

		overlay:Destroy()

		buildMainMenu(
			mode
		)
	end

	track(
		pc.Activated:Connect(function()

			choose(
				"PC"
			)
		end)
	)

	track(
		phone.Activated:Connect(function()

			choose(
				"PHONE"
			)
		end)
	)
end

--========================================================
-- KEY GUI
--========================================================

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

			Parent =
				screenGui
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

			BorderSizePixel =
				0,

			Parent =
				overlay
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

	addGlassGradient(
		panel
	)

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

			Parent =
				panel
		})

	corner(
		header,
		14
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

			Parent =
				header
		})

	corner(
		icon,
		14
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

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

		Text =
			"THH HUB",

		TextColor3 =
			Colors.Text,

		TextSize =
			23,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			header
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

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

		TextSize =
			11,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			header
	})

	local keyBox =
		create("TextBox", {
			BackgroundColor3 =
				Colors.CardDark,

			BackgroundTransparency =
				0.15,

			Position =
				UDim2.fromOffset(
					26,
					145
				),

			Size =
				UDim2.new(
					1,
					-52,
					0,
					52
				),

			ClearTextOnFocus =
				false,

			Font =
				Enum.Font.Gotham,

			PlaceholderText =
				"paste your key here...",

			PlaceholderColor3 =
				Colors.Muted,

			Text =
				"",

			TextColor3 =
				Colors.Text,

			TextSize =
				12,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				panel
		})

	padding(
		keyBox,
		14,
		14,
		0,
		0
	)

	corner(
		keyBox,
		11
	)

	local continue =
		create("TextButton", {
			BackgroundColor3 =
				Color3.fromRGB(
					220,
					223,
					229
				),

			Position =
				UDim2.fromOffset(
					26,
					215
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

			Text =
				"Continue",

			TextColor3 =
				Color3.fromRGB(
					42,
					45,
					51
				),

			TextSize =
				12,

			Parent =
				panel
		})

	corner(
		continue,
		11
	)

	local getKey =
		create("TextButton", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.24,

			Position =
				UDim2.fromOffset(
					26,
					280
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

			TextSize =
				12,

			Parent =
				panel
		})

	corner(
		getKey,
		11
	)

	local status =
		create("TextLabel", {
			BackgroundTransparency =
				1,

			Position =
				UDim2.fromOffset(
					26,
					345
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

			TextSize =
				10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				panel
		})

	track(
		getKey.Activated:Connect(function()

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

	local checking =
		false

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
				"Enter your access key."

			status.TextColor3 =
				Colors.Danger

			return
		end

		checking =
			true

		continue.Text =
			"Checking..."

		task.spawn(function()

			local valid, result =
				validateKey(
					key
				)

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

				checking =
					false

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
		continue.Activated:Connect(
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
