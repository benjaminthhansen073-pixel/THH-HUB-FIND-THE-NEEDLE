if _G.THHGrowersHardUnloaded then
	return
end

if type(_G.THHGrowersCleanup) == "function" then
	pcall(_G.THHGrowersCleanup)
end

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

local PROJECT_ID = "PD6XH2EGXZXUHGED"
local AUTH_SECRET = "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"

local GET_KEY_URL =
	"https://vampauth.com/PD6XH2EGXZXUHGED/flow"

local VAMPAUTH_CLIENT_URL =
	"https://vampauth.com/client/vampauth.lua"

local SEARCH_FOR_NEEDLE_PLACE_ID = 77108422251420
local FARMHOUSE_PLACE_ID = 108628039999641
local BASEMENT_PLACE_ID = 83445806734780

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
	Panel = Color3.fromRGB(70, 73, 80),
	Card = Color3.fromRGB(96, 99, 107),
	Dark = Color3.fromRGB(42, 45, 51),
	Stroke = Color3.fromRGB(190, 193, 200),
	Text = Color3.fromRGB(247, 248, 250),
	SubText = Color3.fromRGB(198, 201, 208)
}

local SPEED_AMOUNT = 50
local FLY_SPEED = 58

local PICKUP_DISTANCE = 3
local SELL_DISTANCE = 3
local UFO_DISTANCE = 3

local AUTO_SELL_INTERVAL = 2
local FARM_PICKUPS_PER_SELL = 4

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

local originalCameraMode =
	LocalPlayer.CameraMode

local originalMinZoom =
	LocalPlayer.CameraMinZoomDistance

local originalMaxZoom =
	LocalPlayer.CameraMaxZoomDistance

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

local hayCache = {}
local diamondCache = {}
local sellCache = {}
local ufoButtonCache = {}
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
local diamondQueued =
	setmetatable({}, {
		__mode = "k"
	})

local colorHayQueue = {}
local colorHayQueued =
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
		object[property] =
			value
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
				radius or 8
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
			color
			or Colors.Stroke,

		Transparency =
			transparency
			or 0,

		Thickness =
			thickness
			or 1,

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

local function tween(
	object,
	duration,
	properties
)
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

local function getPartFromObject(object)
	local current = object

	while current
		and current ~= Workspace do

		if current:IsA("BasePart") then
			return current
		end

		current =
			current.Parent
	end

	return nil
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

local function getDistanceTo(object)
	local root =
		getRoot()

	local part =
		getMainPart(object)

	if not root
		or not part then

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

	local cameraLockConnection =
		RunService.RenderStepped:Connect(function()
			if Camera then
				Camera.CFrame =
					oldCFrame

				Camera.Focus =
					oldFocus
			end
		end)

	local unlocked = false

	return function()
		if unlocked then
			return
		end

		unlocked = true

		if cameraLockConnection then
			pcall(function()
				cameraLockConnection:Disconnect()
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

	rememberPrompt(prompt)

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
		applyClickDetector(object)

	elseif object:IsA("ProximityPrompt") then
		applyPrompt(object)
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

local function looksLikeNeedle(object)
	if not object then
		return false
	end

	if not object:IsA("BasePart")
		and not object:IsA("Model")
		and not object:IsA("Tool") then

		return false
	end

	local lower =
		string.lower(
			object.Name
		)

	return lower == "needle"
		or lower == "needlepart"
		or lower == "hiddenneedle"
		or lower == "hayneedle"
		or string.find(
			lower,
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

	local lower =
		string.lower(
			object.Name
		)

	if lower == "key"
		or lower == "keypart"
		or lower == "basementkey"
		or lower == "haykey"
		or lower == "chapter2key"
		or lower == "exitkey" then

		return true
	end

	if string.find(
		lower,
		"key",
		1,
		true
	) then

		if string.find(
			lower,
			"keypad",
			1,
			true
		)
			or string.find(
				lower,
				"keyboard",
				1,
				true
			)
			or string.find(
				lower,
				"keyhole",
				1,
				true
			)
			or string.find(
				lower,
				"monkey",
				1,
				true
			)
			or string.find(
				lower,
				"donkey",
				1,
				true
			) then

			return false
		end

		return true
	end

	return false
end

local function getTrackedPickupRoot(object)
	local current = object

	while current
		and current ~= Workspace do

		if hayCache[current]
			or diamondCache[current]
			or needleCache[current]
			or keyCache[current] then

			return current
		end

		current =
			current.Parent
	end

	return nil
end

local function queueDiamond(object)
	if not object
		or not object.Parent
		or diamondQueued[object] then

		return
	end

	diamondQueued[object] =
		true

	table.insert(
		diamondQueue,
		object
	)
end

local function queueColorHay(object)
	if not object
		or not object.Parent
		or colorHayQueued[object] then

		return
	end

	colorHayQueued[object] =
		true

	table.insert(
		colorHayQueue,
		object
	)
end

local function queueNeedle(object)
	if not object
		or not object.Parent
		or needleQueued[object] then

		return
	end

	needleQueued[object] =
		true

	table.insert(
		needleQueue,
		object
	)
end

local function queueKey(object)
	if not object
		or not object.Parent
		or keyQueued[object] then

		return
	end

	keyQueued[object] =
		true

	table.insert(
		keyQueue,
		object
	)
end

local function setupColorHayTracker(hay)
	if colorHayConnections[hay] then
		return
	end

	local part =
		getMainPart(hay)

	if not part then
		return
	end

	colorHayState[hay] = {
		LastColor =
			part.Color,

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

				colorHayConnections[hay] =
					nil

				return
			end

			local state =
				colorHayState[hay]

			if not state then
				state = {
					LastColor =
						part.Color,

					LastChanged = 0
				}

				colorHayState[hay] =
					state
			end

			local old =
				state.LastColor

			local new =
				part.Color

			local difference =
				math.abs(
					old.R - new.R
				)
				+ math.abs(
					old.G - new.G
				)
				+ math.abs(
					old.B - new.B
				)

			state.LastColor =
				new

			if difference > 0.015 then
				state.LastChanged =
					os.clock()

				if autoColorHayEnabled then
					queueColorHay(
						hay
					)
				end
			end
		end)

	colorHayConnections[hay] =
		connection

	track(connection)
end

local function registerHay(hay)
	if hayCache[hay] then
		return
	end

	hayCache[hay] =
		true

	applySettingsToObject(
		hay
	)

	setupColorHayTracker(
		hay
	)
end

local function registerDiamond(object)
	if diamondCache[object] then
		return
	end

	diamondCache[object] =
		true

	applySettingsToObject(
		object
	)

	if autoDiamondEnabled then
		queueDiamond(
			object
		)
	end
end

local function registerNeedle(object)
	if needleCache[object] then
		return
	end

	needleCache[object] =
		true

	applySettingsToObject(
		object
	)

	if autoFindNeedleEnabled then
		queueNeedle(
			object
		)
	end
end

local function registerKey(object)
	if keyCache[object] then
		return
	end

	keyCache[object] =
		true

	applySettingsToObject(
		object
	)

	if autoFindKeyEnabled then
		queueKey(
			object
		)
	end
end

local function registerWorkspaceObject(object)
	if object.Name == "HayPiece" then
		registerHay(object)
	end

	if object.Name == "Diamond" then
		registerDiamond(object)
	end

	if object.Name == "SellPart" then
		sellCache[object] =
			true
	end

	if object.Name == "UfoButtenPart" then
		ufoButtonCache[object] =
			true
	end

	if looksLikeNeedle(object) then
		registerNeedle(object)
	end

	if looksLikeKey(object) then
		registerKey(object)
	end

	local trackedRoot =
		getTrackedPickupRoot(
			object
		)

	if trackedRoot then
		if object:IsA("ClickDetector")
			or object:IsA("ProximityPrompt") then

			applyInteractionObject(
				object
			)
		end

		if hayCache[trackedRoot] then
			setupColorHayTracker(
				trackedRoot
			)
		end
	end
end

local function unregisterWorkspaceObject(object)
	hayCache[object] = nil
	diamondCache[object] = nil
	sellCache[object] = nil
	ufoButtonCache[object] = nil
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
	registerWorkspaceObject(
		object
	)
end

track(
	Workspace.DescendantAdded:Connect(function(object)
		registerWorkspaceObject(
			object
		)
	end)
)

track(
	Workspace.DescendantRemoving:Connect(function(object)
		unregisterWorkspaceObject(
			object
		)
	end)
)

local function refreshCachedInteractionSettings()
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

	for object in pairs(needleCache) do
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

local function moveCharacterNearPosition(
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

	if direction.Magnitude < 0.05 then
		direction =
			Vector3.new(
				0,
				0,
				1
			)
	end

	direction =
		direction.Unit

	local targetPosition =
		position
		+ direction
		* distance

	targetPosition =
		Vector3.new(
			targetPosition.X,
			position.Y + 2.6,
			targetPosition.Z
		)

	local lookPosition =
		Vector3.new(
			position.X,
			targetPosition.Y,
			position.Z
		)

	pcall(function()
		character:PivotTo(
			CFrame.lookAt(
				targetPosition,
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

	distance =
		distance
		or 5

	if Camera then
		local target =
			Camera.CFrame
			* CFrame.new(
				0,
				0,
				-distance
			)

		return setObjectPivot(
			object,
			target
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
				-2.5
			)
		)
	end

	return false
end

local function clickDetectorDirect(detector)
	if not detector
		or not detector.Parent then

		return false
	end

	applyClickDetector(
		detector
	)

	if type(fireclickdetector)
		== "function" then

		local success =
			pcall(function()
				fireclickdetector(
					detector
				)
			end)

		if success then
			return true
		end
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

				if noPickupCooldownEnabled then
					RunService.RenderStepped:Wait()
				else
					task.wait(0.018)
				end

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

			if not noPickupCooldownEnabled then
				task.wait(0.01)
			end

			mouse1click()
		end)
	end

	return false
end

local function realClickObject(object)
	if not object
		or not object.Parent then

		return false
	end

	local detector =
		getClickDetector(
			object
		)

	if detector
		and clickDetectorDirect(
			detector
		) then

		return true
	end

	local part =
		getMainPart(
			object
		)

	if part then
		return screenClickPart(
			part
		)
	end

	return false
end

local function pickupObject(object)
	if not object
		or not object.Parent then

		return false
	end

	return withInteractionLock(function()
		local character =
			getCharacter()

		local root =
			getRoot()

		local part =
			getMainPart(
				object
			)

		if not character
			or not root
			or not part then

			return false
		end

		local originalPlayerPivot =
			character:GetPivot()

		local originalObjectPivot =
			getObjectPivot(
				object
			)

		if not originalObjectPivot then
			return false
		end

		local originalPosition =
			originalObjectPivot.Position

		local unlockCamera =
			lockCamera()

		moveCharacterNearPosition(
			originalPosition,
			PICKUP_DISTANCE
		)

		if noPickupCooldownEnabled then
			RunService.RenderStepped:Wait()
		else
			task.wait(0.045)
		end

		if not object.Parent then
			pcall(function()
				character:PivotTo(
					originalPlayerPivot
				)
			end)

			unlockCamera()

			return true
		end

		moveObjectInFront(
			object,
			5
		)

		if noPickupCooldownEnabled then
			RunService.RenderStepped:Wait()
		else
			task.wait(0.025)
		end

		local clicked =
			realClickObject(
				object
			)

		if not noPickupCooldownEnabled then
			task.wait(0.025)
		end

		if object
			and object.Parent then

			setObjectPivot(
				object,
				originalObjectPivot
			)
		end

		if character
			and character.Parent then

			pcall(function()
				character:PivotTo(
					originalPlayerPivot
				)
			end)

			stopVelocity()
		end

		unlockCamera()

		return clicked
	end)
end

local function addUnique(
	list,
	object
)
	if not object then
		return
	end

	for _, existing in ipairs(list) do
		if existing == object then
			return
		end
	end

	table.insert(
		list,
		object
	)
end

local function getSellSearchObjects(sellPart)
	local objects = {}

	addUnique(
		objects,
		sellPart
	)

	for _, descendant in ipairs(
		sellPart:GetDescendants()
	) do
		addUnique(
			objects,
			descendant
		)
	end

	local parent =
		sellPart.Parent

	if parent
		and parent ~= Workspace then

		addUnique(
			objects,
			parent
		)

		for _, descendant in ipairs(
			parent:GetDescendants()
		) do
			addUnique(
				objects,
				descendant
			)
		end
	end

	local grandParent =
		parent
		and parent.Parent

	if grandParent
		and grandParent ~= Workspace
		and grandParent ~= game then

		for _, descendant in ipairs(
			grandParent:GetDescendants()
		) do
			if descendant.Name == "SellPart"
				or descendant:IsA("ClickDetector")
				or descendant:IsA("ProximityPrompt")
				or descendant:IsA("TouchTransmitter")
				or descendant:IsA("Attachment")
				or descendant:IsA("AlignPosition")
				or descendant:IsA("LinearVelocity")
				or descendant:IsA("VectorForce")
				or descendant:IsA("BodyPosition")
				or descendant:IsA("BodyVelocity") then

				addUnique(
					objects,
					descendant
				)
			end
		end
	end

	return objects
end

local function scanSellPart(sellPart)
	if not sellPart
		or not sellPart.Parent then

		return nil
	end

	local result = {
		SellPart = sellPart,

		InteractionPart =
			getMainPart(
				sellPart
			),

		ClickDetector = nil,
		Prompt = nil,
		TouchTransmitter = nil,
		Attachment = nil,
		Attractor = nil
	}

	local objects =
		getSellSearchObjects(
			sellPart
		)

	local sellMainPart =
		getMainPart(
			sellPart
		)

	local bestClickScore =
		math.huge

	for _, object in ipairs(objects) do
		if object:IsA("ClickDetector") then
			local clickPart =
				getPartFromObject(
					object
				)

			local score = 100

			if clickPart
				== sellMainPart then

				score = 0

			elseif clickPart
				and clickPart.Name
					== "SellPart" then

				score = 1

			elseif object.Parent
				== sellPart then

				score = 2
			else
				score = 10
			end

			if score
				< bestClickScore then

				bestClickScore =
					score

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

		elseif object:IsA("TouchTransmitter") then
			result.TouchTransmitter =
				result.TouchTransmitter
				or object

		elseif object:IsA("Attachment") then
			result.Attachment =
				result.Attachment
				or object

		elseif object:IsA("AlignPosition")
			or object:IsA("LinearVelocity")
			or object:IsA("VectorForce")
			or object:IsA("BodyPosition")
			or object:IsA("BodyVelocity") then

			result.Attractor =
				result.Attractor
				or object
		end
	end

	if result.ClickDetector then
		local clickPart =
			getPartFromObject(
				result.ClickDetector
			)

		if clickPart then
			result.InteractionPart =
				clickPart
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
			scanSellPart(
				sellPart
			)

		if not scan then
			return false
		end

		local part =
			scan.InteractionPart
			or getMainPart(
				sellPart
			)

		local character =
			getCharacter()

		local root =
			getRoot()

		local humanoid =
			getHumanoid()

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

		local unlockCamera =
			lockCamera()

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

		moveCharacterNearPosition(
			part.Position,
			SELL_DISTANCE
		)

		task.wait(0.08)

		local clicked = false

		if scan.ClickDetector then
			clicked =
				clickDetectorDirect(
					scan.ClickDetector
				)
		end

		if not clicked then
			clicked =
				screenClickPart(
					part
				)
		end

		task.wait(0.04)

		if character
			and character.Parent then

			pcall(function()
				character:PivotTo(
					oldPivot
				)
			end)

			stopVelocity()
		end

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

		unlockCamera()

		return clicked
	end)
end

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

local function getNearestCached(cache)
	local list =
		getCachedList(
			cache
		)

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

local function sellNow()
	local sellParts =
		getCachedList(
			sellCache
		)

	if #sellParts == 0 then
		return false
	end

	table.sort(
		sellParts,
		function(a, b)
			return getDistanceTo(a)
				< getDistanceTo(b)
		end
	)

	for _, sellPart in ipairs(
		sellParts
	) do
		if sellPart.Parent
			and sellOnePart(
				sellPart
			) then

			return true
		end
	end

	return false
end

local function startAutoFarm()
	if autoFarmEnabled then
		return
	end

	autoFarmEnabled =
		true

	task.spawn(function()
		local pickupCount = 0

		while autoFarmEnabled
			and screenGui
			and screenGui.Parent do

			if pickupCount
				>= FARM_PICKUPS_PER_SELL then

				sellNow()

				pickupCount = 0

				task.wait(0.08)

				continue
			end

			local hay =
				getNearestCached(
					hayCache
				)

			if hay
				and hay.Parent then

				pickupObject(
					hay
				)

				pickupCount += 1

				if noPickupCooldownEnabled then
					RunService.RenderStepped:Wait()
				else
					task.wait(0.04)
				end
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
				getNearestCached(
					hayCache
				)

			if hay
				and hay.Parent then

				pickupObject(
					hay
				)

				if noPickupCooldownEnabled then
					RunService.RenderStepped:Wait()
				else
					task.wait(0.05)
				end
			else
				task.wait(0.1)
			end
		end
	end)
end

local function stopAutoHay()
	autoHayEnabled =
		false
end

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

			sellNow()

			local started =
				os.clock()

			while autoSellEnabled
				and os.clock()
					- started
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

local function startAutoDiamond()
	if autoDiamondEnabled then
		return
	end

	autoDiamondEnabled =
		true

	for object in pairs(
		diamondCache
	) do
		if object.Parent then
			queueDiamond(
				object
			)
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
				diamondQueued[object] =
					nil

				if object.Parent then
					pickupObject(
						object
					)

					if object.Parent
						and autoDiamondEnabled then

						task.delay(
							0.4,
							function()

								if autoDiamondEnabled
									and object.Parent then

									queueDiamond(
										object
									)
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
	autoDiamondEnabled =
		false

	table.clear(
		diamondQueue
	)

	for object in pairs(
		diamondQueued
	) do
		diamondQueued[object] =
			nil
	end
end

local function startAutoColorHay()
	if autoColorHayEnabled then
		return
	end

	autoColorHayEnabled =
		true

	for hay, state in pairs(
		colorHayState
	) do
		if hay.Parent
			and state.LastChanged > 0
			and os.clock()
				- state.LastChanged
				< 1.5 then

			queueColorHay(
				hay
			)
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
				colorHayQueued[hay] =
					nil

				if hay.Parent then
					local state =
						colorHayState[
							hay
						]

					if state
						and os.clock()
							- state.LastChanged
							<= 1.5 then

						pickupObject(
							hay
						)
					end
				end
			else
				task.wait(0.06)
			end
		end
	end)
end

local function stopAutoColorHay()
	autoColorHayEnabled =
		false

	table.clear(
		colorHayQueue
	)

	for object in pairs(
		colorHayQueued
	) do
		colorHayQueued[object] =
			nil
	end
end

local function startAutoFindNeedle()
	if autoFindNeedleEnabled then
		return
	end

	autoFindNeedleEnabled =
		true

	for object in pairs(
		needleCache
	) do
		if object.Parent then
			queueNeedle(
				object
			)
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
				needleQueued[object] =
					nil

				if object.Parent then
					local success =
						pickupObject(
							object
						)

					if success then
						notifications(
							"Needle",
							"Needle detected and picked up.",
							3
						)
					end

					if object.Parent
						and autoFindNeedleEnabled then

						task.delay(
							0.75,
							function()

								if autoFindNeedleEnabled
									and object.Parent then

									queueNeedle(
										object
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

	table.clear(
		needleQueue
	)

	for object in pairs(
		needleQueued
	) do
		needleQueued[object] =
			nil
	end
end

local function startAutoFindKey()
	if autoFindKeyEnabled then
		return
	end

	autoFindKeyEnabled =
		true

	for object in pairs(
		keyCache
	) do
		if object.Parent then
			queueKey(
				object
			)
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
				keyQueued[object] =
					nil

				if object.Parent then
					local success =
						pickupObject(
							object
						)

					if success then
						notifications(
							"Key",
							"Key detected and picked up.",
							3
						)
					end

					if object.Parent
						and autoFindKeyEnabled then

						task.delay(
							0.75,
							function()

								if autoFindKeyEnabled
									and object.Parent then

									queueKey(
										object
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

	table.clear(
		keyQueue
	)

	for object in pairs(
		keyQueued
	) do
		keyQueued[object] =
			nil
	end
end

local function startUfoEvent()
	local button =
		getNearestCached(
			ufoButtonCache
		)

	if not button then
		notifications(
			"UFO Event",
			"UfoButtenPart was not found.",
			3,
			"danger"
		)

		return
	end

	task.spawn(function()
		local success =
			withInteractionLock(function()

				local character =
					getCharacter()

				local root =
					getRoot()

				local humanoid =
					getHumanoid()

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

				local oldAutoRotate

				if humanoid then
					oldAutoRotate =
						humanoid.AutoRotate
				end

				local unlockCamera =
					lockCamera()

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

				moveCharacterNearPosition(
					part.Position,
					UFO_DISTANCE
				)

				task.wait(0.08)

				local clicked =
					realClickObject(
						button
					)

				task.wait(0.04)

				if character.Parent then
					pcall(function()
						character:PivotTo(
							oldPivot
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

				unlockCamera()

				return clicked
			end)

		notifications(
			"UFO Event",
			success
				and "UFO button clicked."
				or "Could not click UfoButtenPart.",
			3,
			success
				and nil
				or "danger"
		)
	end)
end

local function setSpeed(state)
	speedEnabled =
		state

	if not state then
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
		pcall(function()
			flyVelocity:Destroy()
		end)

		flyVelocity =
			nil
	end

	if flyGyro then
		pcall(function()
			flyGyro:Destroy()
		end)

		flyGyro =
			nil
	end

	if flyHumanoid
		and flyHumanoid.Parent then

		pcall(function()
			flyHumanoid.PlatformStand =
				flyOldPlatformStand
					== true
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
			Name =
				"THHFlyVelocity",

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
			Name =
				"THHFlyGyro",

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
				if flyVelocity then
					flyVelocity.Velocity =
						Vector3.zero
				end

				return
			end

			if not root.Parent
				or not humanoid.Parent then

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

			if movement.Magnitude
				> 0.01 then

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

			if look.Magnitude
				> 0.01 then

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

local function getRayTarget()
	if Mouse
		and Mouse.Target then

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

	local position =
		Vector2.new(
			viewport.X / 2,
			viewport.Y / 2
		)

	local ray =
		Camera:ViewportPointToRay(
			position.X,
			position.Y
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
			ray.Direction * 5000,
			params
		)

	return result
		and result.Instance
		or nil
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
				pickup
			)
		end)
	end)
)

local function stopPartNameHover()
	partNameHoverEnabled =
		false

	if partNameHoverConnection then
		pcall(function()
			partNameHoverConnection:Disconnect()
		end)

		partNameHoverConnection =
			nil
	end

	if partNameHoverLabel then
		pcall(function()
			partNameHoverLabel:Destroy()
		end)

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
			Name =
				"PartNameHover",

			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.04,

			BorderSizePixel = 0,

			Size =
				UDim2.fromOffset(
					240,
					36
				),

			Font =
				Enum.Font.GothamSemibold,

			Text = "",

			TextColor3 =
				Colors.Text,

			TextSize = 12,

			TextTruncate =
				Enum.TextTruncate.AtEnd,

			Visible = false,

			ZIndex = 9999,

			Parent = screenGui
		})

	corner(
		partNameHoverLabel,
		8
	)

	stroke(
		partNameHoverLabel,
		Colors.Accent,
		0.35,
		1
	)

	partNameHoverConnection =
		RunService.RenderStepped:Connect(function()

			if not partNameHoverEnabled
				or not partNameHoverLabel
				or not partNameHoverLabel.Parent then

				return
			end

			local target =
				getRayTarget()

			if not target then
				partNameHoverLabel.Visible =
					false

				return
			end

			Camera =
				Workspace.CurrentCamera
				or Camera

			if not Camera then
				return
			end

			local position =
				UserInputService:GetMouseLocation()

			local viewport =
				Camera.ViewportSize

			local x =
				math.clamp(
					position.X + 16,
					8,
					math.max(
						8,
						viewport.X - 248
					)
				)

			local y =
				math.clamp(
					position.Y + 16,
					8,
					math.max(
						8,
						viewport.Y - 44
					)
				)

			partNameHoverLabel.Text =
				target.Name

			partNameHoverLabel.Position =
				UDim2.fromOffset(
					x,
					y
				)

			partNameHoverLabel.Visible =
				true
		end)

	track(
		partNameHoverConnection
	)
end

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
		pcall(function()
			screenGui:Destroy()
		end)

		screenGui = nil
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
		Name =
			"THHGrowers",

		ResetOnSpawn =
			false,

		IgnoreGuiInset =
			true,

		ZIndexBehavior =
			Enum.ZIndexBehavior.Sibling,

		DisplayOrder =
			999999,

		Parent =
			guiParent
	})

local function createNotificationHolder()
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
					-16,
					0,
					16
				),

			Size =
				UDim2.fromOffset(
					310,
					500
				),

			ZIndex =
				500,

			Parent =
				screenGui
		})

	create("UIListLayout", {
		Padding =
			UDim.new(
				0,
				8
			),

		HorizontalAlignment =
			Enum.HorizontalAlignment.Right,

		VerticalAlignment =
			Enum.VerticalAlignment.Top,

		Parent =
			notificationHolder
	})
end

local function notifications(
	title,
	message,
	duration,
	notificationType
)
	if not notificationHolder then
		createNotificationHolder()
	end

	local accentColor =
		notificationType == "danger"
			and Colors.Danger
			or Colors.Accent

	local card =
		create("Frame", {
			BackgroundColor3 =
				Colors.Card,

			BackgroundTransparency =
				0.03,

			Size =
				UDim2.fromOffset(
					300,
					82
				),

			ZIndex =
				501,

			Parent =
				notificationHolder
		})

	corner(card, 9)

	stroke(
		card,
		Colors.Stroke,
		0.1,
		1
	)

	local accent =
		create("Frame", {
			BackgroundColor3 =
				accentColor,

			BorderSizePixel =
				0,

			Position =
				UDim2.fromOffset(
					8,
					8
				),

			Size =
				UDim2.new(
					0,
					4,
					1,
					-16
				),

			ZIndex =
				502,

			Parent =
				card
		})

	corner(accent, 4)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				22,
				11
			),

		Size =
			UDim2.new(
				1,
				-34,
				0,
				22
			),

		Font =
			Enum.Font.GothamSemibold,

		Text =
			title
			or "THH HUB",

		TextColor3 =
			Colors.Text,

		TextSize =
			14,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		ZIndex =
			502,

		Parent =
			card
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				22,
				36
			),

		Size =
			UDim2.new(
				1,
				-34,
				0,
				34
			),

		Font =
			Enum.Font.Gotham,

		Text =
			message
			or "",

		TextColor3 =
			Colors.SubText,

		TextSize =
			11,

		TextWrapped =
			true,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		ZIndex =
			502,

		Parent =
			card
	})

	card.Position =
		UDim2.fromOffset(
			20,
			0
		)

	card.BackgroundTransparency =
		1

	tween(card, 0.18, {
		Position =
			UDim2.fromOffset(
				0,
				0
			),

		BackgroundTransparency =
			0.03
	})

	task.delay(
		duration or 3,
		function()

			if not card
				or not card.Parent then

				return
			end

			local out =
				tween(
					card,
					0.18,
					{
						Position =
							UDim2.fromOffset(
								20,
								0
							),

						BackgroundTransparency =
							1
					}
				)

			out.Completed:Wait()

			if card then
				card:Destroy()
			end
		end
	)
end

createNotificationHolder()

local function getVampauthClient()
	if VampauthClient then
		return VampauthClient
	end

	if type(loadstring)
		~= "function" then

		return nil,
			"loadstring is unavailable."
	end

	local sourceSuccess, source =
		pcall(function()
			return game:HttpGet(
				VAMPAUTH_CLIENT_URL
			)
		end)

	if not sourceSuccess
		or type(source) ~= "string"
		or source == "" then

		return nil,
			"Could not load Vampauth."
	end

	local compileSuccess, chunk =
		pcall(
			loadstring,
			source
		)

	if not compileSuccess
		or type(chunk)
			~= "function" then

		return nil,
			"Could not start Vampauth."
	end

	local moduleSuccess, Vampauth =
		pcall(chunk)

	if not moduleSuccess
		or type(Vampauth) ~= "table"
		or type(Vampauth.new)
			~= "function" then

		return nil,
			"Vampauth failed to load."
	end

	local clientSuccess, client =
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

	if not clientSuccess
		or not client then

		return nil,
			"Could not initialize Vampauth."
	end

	VampauthClient =
		client

	return client
end

local function validateKey(key)
	key =
		tostring(
			key or ""
		):match(
			"^%s*(.-)%s*$"
		)

	if key == "" then
		return false,
			"Enter your access key."
	end

	local client, errorMessage =
		getVampauthClient()

	if not client then
		return false,
			errorMessage
	end

	local success, valid, result =
		pcall(function()

			local ok, data =
				client:Check(
					key
				)

			return ok, data
		end)

	if not success then
		return false,
			"Key validation failed."
	end

	if valid then
		return true,
			result
	end

	if type(result)
		== "table" then

		return false,
			tostring(
				result.error
				or result.message
				or result.status
				or "Invalid key."
			)
	end

	return false,
		tostring(
			result
			or "Invalid key."
		)
end

local function makeDraggable(
	frame,
	handle
)
	handle =
		handle
		or frame

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

local function fitScale(
	scale,
	width,
	height,
	margin
)
	local function update()
		Camera =
			Workspace.CurrentCamera
			or Camera

		if not Camera then
			return
		end

		local viewport =
			Camera.ViewportSize

		scale.Scale =
			math.clamp(
				math.min(
					(
						viewport.X
						- margin
					)
					/ width,

					(
						viewport.Y
						- margin
					)
					/ height,

					1
				),

				0.45,
				1
			)
	end

	update()

	if Camera then
		track(
			Camera:GetPropertyChangedSignal(
				"ViewportSize"
			):Connect(
				update
			)
		)
	end
end

local function buildMainMenu(mode)
	local phoneMode =
		mode == "PHONE"

	local width =
		phoneMode
			and 430
			or 700

	local height =
		phoneMode
			and 620
			or 455

	local sidebarWidth =
		phoneMode
			and 112
			or 176

	local controlHeight =
		phoneMode
			and 60
			or 48

	local pages = {}
	local navButtons = {}
	local currentPage =
		"Home"

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

			BorderSizePixel =
				0,

			ClipsDescendants =
				true,

			Parent =
				screenGui
		})

	corner(
		mainFrame,
		12
	)

	stroke(
		mainFrame,
		Colors.Stroke,
		0,
		1
	)

	if phoneMode then
		mainScale =
			create("UIScale", {
				Scale =
					1,

				Parent =
					mainFrame
			})

		fitScale(
			mainScale,
			width,
			height,
			20
		)
	end

	local sidebar =
		create("Frame", {
			BackgroundColor3 =
				Colors.Sidebar,

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

	local brand =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.Card,

			Position =
				UDim2.fromOffset(
					phoneMode
						and 12
						or 16,
					14
				),

			Size =
				UDim2.fromOffset(
					42,
					42
				),

			Image =
				GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			Parent =
				sidebar
		})

	corner(
		brand,
		9
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				phoneMode
					and 60
					or 66,
				13
			),

		Size =
			UDim2.new(
				1,
				-70,
				0,
				23
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
			phoneMode
				and 14
				or 15,

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
				phoneMode
					and 60
					or 66,
				35
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
				and "HUB"
				or "PC mode",

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

			Position =
				UDim2.fromOffset(
					phoneMode
						and 7
						or 10,
					73
				),

			Size =
				UDim2.new(
					1,
					phoneMode
						and -14
						or -20,
					0,
					72
				),

			Parent =
				sidebar
		})

	corner(
		profile,
		8
	)

	local avatar =
		create("ImageLabel", {
			BackgroundColor3 =
				Colors.Background,

			Position =
				UDim2.fromOffset(
					8,
					14
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
				56,
				12
			),

		Size =
			UDim2.new(
				1,
				-60,
				0,
				20
			),

		Font =
			Enum.Font.GothamSemibold,

		Text =
			LocalPlayer.DisplayName,

		TextTruncate =
			Enum.TextTruncate.AtEnd,

		TextColor3 =
			Colors.Text,

		TextSize =
			11,

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
				56,
				35
			),

		Size =
			UDim2.new(
				1,
				-60,
				0,
				18
			),

		Font =
			Enum.Font.Gotham,

		Text =
			"@" .. LocalPlayer.Name,

		TextTruncate =
			Enum.TextTruncate.AtEnd,

		TextColor3 =
			Colors.SubText,

		TextSize =
			9,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			profile
	})

	local nav =
		create("ScrollingFrame", {
			BackgroundTransparency =
				1,

			BorderSizePixel =
				0,

			Position =
				UDim2.fromOffset(
					phoneMode
						and 7
						or 10,
					154
				),

			Size =
				UDim2.new(
					1,
					phoneMode
						and -14
						or -20,
					1,
					-164
				),

			CanvasSize =
				UDim2.new(),

			AutomaticCanvasSize =
				Enum.AutomaticSize.Y,

			ScrollBarThickness =
				phoneMode
					and 5
					or 2,

			ScrollBarImageColor3 =
				Colors.Stroke,

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
			nav
	})

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
			BackgroundTransparency =
				1,

			Size =
				UDim2.new(
					1,
					0,
					0,
					58
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
				-120,
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
					31
				),

			Size =
				UDim2.new(
					1,
					-120,
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

	local minimize =
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
					-52,
					0,
					10
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
				Colors.SubText,

			TextSize =
				16,

			Parent =
				header
		})

	corner(
		minimize,
		7
	)

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
					-10,
					0,
					10
				),

			Size =
				UDim2.fromOffset(
					36,
					36
				),

			Font =
				Enum.Font.GothamBold,

			Text =
				"×",

			TextColor3 =
				Colors.SubText,

			TextSize =
				20,

			Parent =
				header
		})

	corner(
		close,
		7
	)

	local holder =
		create("Frame", {
			BackgroundTransparency =
				1,

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

			ClipsDescendants =
				true,

			Parent =
				content
		})

	local function createPage(name)
		local page =
			create("ScrollingFrame", {
				Name =
					name
					.. "Page",

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

				ScrollBarImageColor3 =
					Colors.Stroke,

				Visible =
					false,

				Parent =
					holder
			})

		padding(
			page,
			phoneMode
				and 12
				or 16,
			phoneMode
				and 12
				or 16,
			phoneMode
				and 12
				or 16,
			phoneMode
				and 12
				or 16
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
			local active =
				navName == name

			button.BackgroundColor3 =
				active
					and Colors.AccentDark
					or Colors.Sidebar

			button.TextColor3 =
				active
					and Colors.Accent
					or Colors.SubText
		end

		currentPage =
			name

		pageTitle.Text =
			name
	end

	local function createNav(name)
		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Sidebar,

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
					12,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				Parent =
					nav
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
			7
		)

		track(
			button.MouseButton1Click:Connect(function()
				setPage(
					name
				)
			end)
		)

		navButtons[name] =
			button

		return button
	end

	local function section(
		parent,
		title,
		subtitle
	)
		local frame =
			create("Frame", {
				BackgroundColor3 =
					Colors.Card,

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
			9
		)

		stroke(
			frame,
			Colors.Stroke,
			0.15,
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

		if subtitle then
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
					subtitle,

				TextColor3 =
					Colors.SubText,

				TextSize =
					11,

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

	local function createToggle(
		parent,
		title,
		subtitle,
		default,
		callback
	)
		local state =
			default == true

		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Background,

				BorderSizePixel =
					0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						controlHeight
					),

				Text =
					"",

				Parent =
					parent
			})

		corner(
			button,
			7
		)

		stroke(
			button,
			Colors.Stroke,
			0.25,
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
					-88,
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
				phoneMode
					and 13
					or 12,

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
					phoneMode
						and 30
						or 27
				),

			Size =
				UDim2.new(
					1,
					-92,
					0,
					18
				),

			Font =
				Enum.Font.Gotham,

			Text =
				subtitle
				or "",

			TextColor3 =
				Colors.SubText,

			TextSize =
				phoneMode
					and 10
					or 9,

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
						phoneMode
							and 52
							or 44,

						phoneMode
							and 29
							or 24
					),

				Parent =
					button
			})

		corner(
			switch,
			20
		)

		local knob =
			create("Frame", {
				BackgroundColor3 =
					Colors.SubText,

				AnchorPoint =
					Vector2.new(
						0.5,
						0.5
					),

				Size =
					UDim2.fromOffset(
						phoneMode
							and 23
							or 18,

						phoneMode
							and 23
							or 18
					),

				Parent =
					switch
			})

		corner(
			knob,
			20
		)

		local function refresh()
			switch.BackgroundColor3 =
				state
					and Colors.AccentDark
					or Colors.Stroke

			knob.BackgroundColor3 =
				state
					and Colors.Accent
					or Colors.SubText

			knob.Position =
				state
					and UDim2.new(
						1,
						-(
							phoneMode
								and 14.5
								or 12
						),
						0.5,
						0
					)
					or UDim2.new(
						0,
						phoneMode
							and 14.5
							or 12,
						0.5,
						0
					)
		end

		refresh()

		track(
			button.MouseButton1Click:Connect(function()

				state =
					not state

				refresh()

				if callback then
					callback(
						state
					)
				end
			end)
		)

		return button
	end

	local function createAction(
		parent,
		title,
		subtitle,
		callback,
		danger
	)
		local button =
			create("TextButton", {
				BackgroundColor3 =
					Colors.Background,

				BorderSizePixel =
					0,

				Size =
					UDim2.new(
						1,
						0,
						0,
						controlHeight
					),

				Text =
					"",

				Parent =
					parent
			})

		corner(
			button,
			7
		)

		stroke(
			button,
			Colors.Stroke,
			0.25,
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
					27
				),

			Size =
				UDim2.new(
					1,
					-24,
					0,
					18
				),

			Font =
				Enum.Font.Gotham,

			Text =
				subtitle
				or "",

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
			button.MouseButton1Click:Connect(function()

				if callback then
					callback()
				end
			end)
		)

		return button
	end

	local function createSlider(
		parent,
		title,
		minimum,
		maximum,
		default,
		callback
	)
		local holder =
			create("Frame", {
				BackgroundColor3 =
					Colors.Background,

				Size =
					UDim2.new(
						1,
						0,
						0,
						68
					),

				Parent =
					parent
			})

		corner(
			holder,
			7
		)

		return holder
	end

	local function createNumberInput(
		parent,
		title,
		default,
		minimum,
		maximum,
		callback
	)
		local holder =
			create("Frame", {
				BackgroundColor3 =
					Colors.Background,

				Size =
					UDim2.new(
						1,
						0,
						0,
						controlHeight
					),

				Parent =
					parent
			})

		corner(
			holder,
			7
		)

		return holder
	end

	local function createUpdateCard(
		parent,
		title,
		date,
		text
	)
		local frame =
			section(
				parent,
				title,
				date
			)

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
				text,

			TextColor3 =
				Colors.SubText,

			TextSize =
				11,

			TextWrapped =
				true,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				frame
		})

		return frame
	end

	createNav("Home")
	createNav("Mods")
	createNav("Player")
	createNav("Event")
	createNav("Utility")
	createNav("Updates")

	local homePage =
		createPage("Home")

	local modsPage =
		createPage("Mods")

	local playerPage =
		createPage("Player")

	local eventPage =
		createPage("Event")

	local utilityPage =
		createPage("Utility")

	local updatesPage =
		createPage("Updates")

	section(
		homePage,
		"welcome to THH Growers",
		"camera-locked pickup system loaded"
	)

	local farming =
		section(
			modsPage,
			"Farming",
			"hay collection and selling"
		)

	createToggle(
		farming,
		"Auto Farm",
		"pick 4 HayPieces → sell → repeat",
		false,
		function(state)

			if state then
				startAutoFarm()
			else
				stopAutoFarm()
			end

			notifications(
				"Auto Farm",
				state
					and "4 pickups → sell enabled."
					or "Auto Farm disabled.",
				2
			)
		end
	)

	createToggle(
		farming,
		"Auto Pick Up Hay",
		"continuously picks HayPiece without auto selling",
		false,
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
		"freeze → TP to real SellPart interaction → click → return",
		false,
		function(state)

			if state then
				startAutoSell()
			else
				stopAutoSell()
			end

			notifications(
				"Auto Sell",
				state
					and "Auto Sell enabled."
					or "Auto Sell disabled.",
				2
			)
		end
	)

	createToggle(
		farming,
		"No Pickup Cooldown",
		"removes THH's local pickup waits and prompt hold time",
		false,
		function(state)

			noPickupCooldownEnabled =
				state

			refreshCachedInteractionSettings()

			notifications(
				"No Pickup Cooldown",
				state
					and "Local pickup delay reduced."
					or "Local pickup delay restored.",
				2
			)
		end
	)

	createToggle(
		farming,
		"Inf Range",
		"far-click Hay, Diamond, color Hay, Needle and Key",
		false,
		function(state)

			infRangeEnabled =
				state

			refreshCachedInteractionSettings()

			notifications(
				"Inf Range",
				state
					and "Inf Range enabled."
					or "Inf Range disabled.",
				2
			)
		end
	)

	local special =
		section(
			modsPage,
			"Special Pickups",
			"same camera-locked pickup method"
		)

	createToggle(
		special,
		"Auto Pick Up Diamond",
		"TP near real Diamond → move Diamond in front → click → return",
		false,
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
		"color change → TP near → move in front → click → return",
		false,
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
			playerPage,
			"Movement",
			"player movement mods"
		)

	createToggle(
		movement,
		"Speed",
		"WalkSpeed 50",
		false,
		function(state)

			setSpeed(
				state
			)
		end
	)

	createToggle(
		movement,
		"Infinite Jump",
		"jump again while airborne",
		false,
		function(state)

			infiniteJumpEnabled =
				state
		end
	)

	createToggle(
		movement,
		"Unlock 3rd Person",
		"unlock Classic camera zoom",
		false,
		function(state)

			setThirdPerson(
				state
			)
		end
	)

	createToggle(
		movement,
		"Fly",
		"WASD + Space/Ctrl",
		false,
		function(state)

			setFly(
				state
			)
		end
	)

	local events =
		section(
			eventPage,
			"Events",
			"Search For The Needle event tools"
		)

	createAction(
		events,
		"Start UFO Event",
		"freeze → TP to UfoButtenPart → click → return",
		function()

			startUfoEvent()
		end
	)

	createToggle(
		events,
		"Auto Find Needle",
		"wait for Needle to replicate → pickup automatically",
		false,
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
		"find Key/KeyPart in Basement → pickup automatically",
		false,
		function(state)

			if state then
				startAutoFindKey()
			else
				stopAutoFindKey()
			end

			notifications(
				"Auto Find Key",
				state
					and "Watching for the Basement Key."
					or "Auto Find Key disabled.",
				2
			)
		end
	)

	local gameIds =
		section(
			eventPage,
			"Place IDs",
			"Search For The Needle"
		)

	createAction(
		gameIds,
		"Copy Main Place ID",
		tostring(
			SEARCH_FOR_NEEDLE_PLACE_ID
		),
		function()

			copyText(
				tostring(
					SEARCH_FOR_NEEDLE_PLACE_ID
				)
			)
		end
	)

	createAction(
		gameIds,
		"Copy Farmhouse ID",
		tostring(
			FARMHOUSE_PLACE_ID
		),
		function()

			copyText(
				tostring(
					FARMHOUSE_PLACE_ID
				)
			)
		end
	)

	createAction(
		gameIds,
		"Copy Basement ID",
		tostring(
			BASEMENT_PLACE_ID
		),
		function()

			copyText(
				tostring(
					BASEMENT_PLACE_ID
				)
			)
		end
	)

	local tools =
		section(
			utilityPage,
			"Tools",
			"world and menu tools"
		)

	createToggle(
		tools,
		"Part Name Hover",
		"shows the exact part name",
		false,
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
		"prevent normal Roblox idle kick",
		false,
		function(state)

			antiAfkEnabled =
				state

			if antiAfkConnection then
				pcall(function()
					antiAfkConnection:Disconnect()
				end)

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
			utilityPage,
			"Server",
			"server and menu actions"
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
		"Reset Character",
		"reset your character",
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
		"copy JobId",
		function()

			if copyText(
				game.JobId
			) then

				notifications(
					"Server ID",
					"Copied.",
					2
				)
			end
		end
	)

	createAction(
		server,
		"Unload Menu",
		"disable all mods and unload THH HUB",
		function()

			local overlay =
				create("Frame", {
					BackgroundColor3 =
						Color3.new(
							0,
							0,
							0
						),

					BackgroundTransparency =
						0.3,

					Size =
						UDim2.fromScale(
							1,
							1
						),

					ZIndex =
						5000,

					Parent =
						screenGui
				})

			local modal =
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
							350,
							210
						),

					BackgroundColor3 =
						Colors.Card,

					ZIndex =
						5001,

					Parent =
						overlay
				})

			corner(
				modal,
				12
			)

			create("TextLabel", {
				BackgroundTransparency =
					1,

				Position =
					UDim2.fromOffset(
						18,
						18
					),

				Size =
					UDim2.new(
						1,
						-36,
						0,
						30
					),

				Font =
					Enum.Font.GothamBold,

				Text =
					"Unload THH Growers?",

				TextColor3 =
					Colors.Text,

				TextSize =
					16,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				ZIndex =
					5002,

				Parent =
					modal
			})

			create("TextLabel", {
				BackgroundTransparency =
					1,

				Position =
					UDim2.fromOffset(
						18,
						60
					),

				Size =
					UDim2.new(
						1,
						-36,
						0,
						65
					),

				Font =
					Enum.Font.GothamSemibold,

				Text =
					"ALL MODS WILL BE TURNED OFF.\n"
					.. "YOU WILL NO LONGER BE ABLE TO OPEN THIS MENU.",

				TextColor3 =
					Colors.Danger,

				TextSize =
					11,

				TextWrapped =
					true,

				TextXAlignment =
					Enum.TextXAlignment.Left,

				ZIndex =
					5002,

				Parent =
					modal
			})

			local cancel =
				create("TextButton", {
					BackgroundColor3 =
						Colors.Background,

					Position =
						UDim2.new(
							0,
							18,
							1,
							-60
						),

					Size =
						UDim2.new(
							0.5,
							-23,
							0,
							42
						),

					Font =
						Enum.Font.GothamSemibold,

					Text =
						"Cancel",

					TextColor3 =
						Colors.Text,

					TextSize =
						12,

					ZIndex =
						5002,

					Parent =
						modal
				})

			corner(
				cancel,
				7
			)

			local unload =
				create("TextButton", {
					BackgroundColor3 =
						Colors.Danger,

					Position =
						UDim2.new(
							0.5,
							5,
							1,
							-60
						),

					Size =
						UDim2.new(
							0.5,
							-23,
							0,
							42
						),

					Font =
						Enum.Font.GothamSemibold,

					Text =
						"Unload Menu",

					TextColor3 =
						Colors.Text,

					TextSize =
						12,

					ZIndex =
						5002,

					Parent =
						modal
				})

			corner(
				unload,
				7
			)

			track(
				cancel.MouseButton1Click:Connect(function()
					overlay:Destroy()
				end)
			)

			track(
				unload.MouseButton1Click:Connect(function()
					cleanupAll(
						true
					)
				end)
			)
		end,
		true
	)

	createUpdateCard(
		updatesPage,
		"pickup update",
		"09/09/26",
		"• camera now stays locked during TPs\n"
		.. "• restored wider Auto Sell scanner\n"
		.. "• Auto Farm now picks 4 then sells\n"
		.. "• added Auto Pick Up Hay\n"
		.. "• Diamond moves in front before click\n"
		.. "• Color Hay moves in front before click\n"
		.. "• added Auto Find Key"
	)

	local function setVisible(visible)
		mainFrame.Visible =
			visible

		if phoneMode
			and floatingButton then

			floatingButton.Visible =
				not visible
		end
	end

	track(
		minimize.MouseButton1Click:Connect(function()
			setVisible(
				false
			)
		end)
	)

	track(
		close.MouseButton1Click:Connect(function()
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
			14
		)

		stroke(
			floatingButton,
			Colors.Accent,
			0.15,
			1.5
		)

		track(
			floatingButton.MouseButton1Click:Connect(function()
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

	notifications(
		"THH HUB",
		phoneMode
			and "Phone mode loaded."
			or "PC mode loaded. RightShift toggles the menu.",
		3
	)

	return {
		createPage =
			createPage,

		createNav =
			createNav,

		section =
			section,

		createToggle =
			createToggle,

		createAction =
			createAction,

		createSlider =
			createSlider,

		createNumberInput =
			createNumberInput,

		createUpdateCard =
			createUpdateCard,

		notifications =
			notifications
	}
end

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
				0.25,

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
					460,
					290
				),

			BackgroundColor3 =
				Glass.Panel,

			BackgroundTransparency =
				0.28,

			BorderSizePixel =
				0,

			Parent =
				overlay
		})

	corner(
		panel,
		15
	)

	stroke(
		panel,
		Glass.Stroke,
		0.65,
		1
	)

	local scale =
		create("UIScale", {
			Scale =
				1,

			Parent =
				panel
		})

	fitScale(
		scale,
		460,
		290,
		28
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.fromOffset(
				20,
				18
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

		Text =
			"what are you on?",

		TextColor3 =
			Glass.Text,

		TextSize =
			19,

		Parent =
			panel
	})

	create("TextLabel", {
		BackgroundTransparency =
			1,

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

		Text =
			"pick pc or phone",

		TextColor3 =
			Glass.SubText,

		TextSize =
			12,

		Parent =
			panel
	})

	local pc =
		create("TextButton", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.4,

			Position =
				UDim2.fromOffset(
					25,
					86
				),

			Size =
				UDim2.fromOffset(
					195,
					177
				),

			Text =
				"",

			Parent =
				panel
		})

	corner(
		pc,
		12
	)

	local monitor =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					31,
					34,
					40
				),

			Position =
				UDim2.new(
					0.5,
					-57,
					0,
					24
				),

			Size =
				UDim2.fromOffset(
					114,
					72
				),

			Parent =
				pc
		})

	corner(
		monitor,
		8
	)

	stroke(
		monitor,
		Glass.Text,
		0.22,
		2
	)

	create("Frame", {
		BackgroundColor3 =
			Glass.Text,

		BorderSizePixel =
			0,

		Position =
			UDim2.new(
				0.5,
				-3,
				1,
				0
			),

		Size =
			UDim2.fromOffset(
				6,
				18
			),

		Parent =
			monitor
	})

	local stand =
		create("Frame", {
			BackgroundColor3 =
				Glass.Text,

			BorderSizePixel =
				0,

			Position =
				UDim2.new(
					0.5,
					-28,
					1,
					16
				),

			Size =
				UDim2.fromOffset(
					56,
					6
				),

			Parent =
				monitor
		})

	corner(
		stand,
		3
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-38
			),

		Size =
			UDim2.new(
				1,
				0,
				0,
				26
			),

		Font =
			Enum.Font.GothamBold,

		Text =
			"PC",

		TextColor3 =
			Glass.Text,

		TextSize =
			16,

		Parent =
			pc
	})

	local phone =
		create("TextButton", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.4,

			Position =
				UDim2.fromOffset(
					240,
					86
				),

			Size =
				UDim2.fromOffset(
					195,
					177
				),

			Text =
				"",

			Parent =
				panel
		})

	corner(
		phone,
		12
	)

	local phoneBody =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					31,
					34,
					40
				),

			Position =
				UDim2.new(
					0.5,
					-35,
					0,
					15
				),

			Size =
				UDim2.fromOffset(
					70,
					111
				),

			Parent =
				phone
		})

	corner(
		phoneBody,
		13
	)

	stroke(
		phoneBody,
		Glass.Text,
		0.2,
		2
	)

	local phoneScreen =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					105,
					109,
					119
				),

			Position =
				UDim2.fromOffset(
					6,
					13
				),

			Size =
				UDim2.new(
					1,
					-12,
					1,
					-27
				),

			Parent =
				phoneBody
		})

	corner(
		phoneScreen,
		7
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

		Position =
			UDim2.new(
				0,
				0,
				1,
				-38
			),

		Size =
			UDim2.new(
				1,
				0,
				0,
				26
			),

		Font =
			Enum.Font.GothamBold,

		Text =
			"PHONE",

		TextColor3 =
			Glass.Text,

		TextSize =
			16,

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
		pc.MouseButton1Click:Connect(function()
			choose(
				"PC"
			)
		end)
	)

	track(
		phone.MouseButton1Click:Connect(function()
			choose(
				"PHONE"
			)
		end)
	)
end

local function showKeySystem()
	local overlay =
		create("Frame", {
			Name =
				"KeySystem",

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
					410,
					385
				),

			BackgroundColor3 =
				Glass.Panel,

			BackgroundTransparency =
				0.24,

			BorderSizePixel =
				0,

			Parent =
				overlay
		})

	corner(
		panel,
		17
	)

	stroke(
		panel,
		Glass.Stroke,
		0.68,
		1
	)

	local scale =
		create("UIScale", {
			Scale =
				1,

			Parent =
				panel
		})

	fitScale(
		scale,
		410,
		385,
		28
	)

	local header =
		create("Frame", {
			BackgroundColor3 =
				Color3.fromRGB(
					110,
					113,
					121
				),

			BackgroundTransparency =
				0.76,

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

			Parent =
				panel
		})

	corner(
		header,
		13
	)

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

			Image =
				GAME_ICON,

			ScaleType =
				Enum.ScaleType.Crop,

			Parent =
				header
		})

	corner(
		icon,
		13
	)

	create("TextLabel", {
		BackgroundTransparency =
			1,

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

		Text =
			"THH HUB",

		TextColor3 =
			Glass.Text,

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

		Text =
			"Authentication",

		TextColor3 =
			Glass.SubText,

		TextSize =
			12,

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
				25,
				132
			),

		Size =
			UDim2.new(
				1,
				-50,
				0,
				20
			),

		Font =
			Enum.Font.GothamSemibold,

		Text =
			"Access Key",

		TextColor3 =
			Glass.Text,

		TextSize =
			12,

		TextXAlignment =
			Enum.TextXAlignment.Left,

		Parent =
			panel
	})

	local keyBox =
		create("TextBox", {
			BackgroundColor3 =
				Glass.Dark,

			BackgroundTransparency =
				0.18,

			Position =
				UDim2.fromOffset(
					25,
					160
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

			PlaceholderColor3 =
				Color3.fromRGB(
					157,
					160,
					168
				),

			Text =
				"",

			TextColor3 =
				Glass.Text,

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
		10
	)

	stroke(
		keyBox,
		Glass.Stroke,
		0.75,
		1
	)

	local continueButton =
		create("TextButton", {
			BackgroundColor3 =
				Color3.fromRGB(
					201,
					204,
					210
				),

			BackgroundTransparency =
				0.08,

			Position =
				UDim2.fromOffset(
					25,
					229
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

			Text =
				"Continue",

			TextColor3 =
				Color3.fromRGB(
					37,
					40,
					46
				),

			TextSize =
				12,

			Parent =
				panel
		})

	corner(
		continueButton,
		10
	)

	local getKeyButton =
		create("TextButton", {
			BackgroundColor3 =
				Glass.Card,

			BackgroundTransparency =
				0.48,

			Position =
				UDim2.fromOffset(
					25,
					291
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

			Text =
				"Get Access Key",

			TextColor3 =
				Glass.Text,

			TextSize =
				12,

			Parent =
				panel
		})

	corner(
		getKeyButton,
		10
	)

	local status =
		create("TextLabel", {
			BackgroundTransparency =
				1,

			Position =
				UDim2.fromOffset(
					25,
					348
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

			TextSize =
				10,

			TextXAlignment =
				Enum.TextXAlignment.Left,

			Parent =
				panel
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

	local checking =
		false

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

		checking =
			true

		continueButton.Text =
			"Checking..."

		status.Text =
			"validating key..."

		status.TextColor3 =
			Glass.SubText

		task.spawn(function()

			local valid, result =
				validateKey(
					key
				)

			if not panel
				or not panel.Parent then

				return
			end

			if valid then
				status.Text =
					"key accepted"

				status.TextColor3 =
					Colors.Accent

				continueButton.Text =
					"Accepted"

				task.wait(0.25)

				overlay:Destroy()

				showDeviceChooser()

				return
			end

			checking =
				false

			continueButton.Text =
				"Continue"

			status.Text =
				tostring(
					result
					or "Invalid key."
				)

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
