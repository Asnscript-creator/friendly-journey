--==================================================
-- STEAL AN EGG - ALL IN ONE SYSTEM
-- Roblox Studio / ServerScriptService
--==================================================

local Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")

----------------------------------------------------
-- CONFIG
----------------------------------------------------

local CONFIG = {
	MAX_INVENTORY = 100,

	AUTO_REWARD_TIME = 60,
	AUTO_EVENT_TIME = 300,
	EVENT_DURATION = 120,

	AUTO_COLLECT_DISTANCE = 25,
	TREADMILL_REWARD = 2,

	VIP_GAMEPASS_ID = 0, -- isi GamePass ID milikmu

	EGG_RARITIES = {
		{Name = "Common", Chance = 60, Reward = 10},
		{Name = "Uncommon", Chance = 25, Reward = 25},
		{Name = "Rare", Chance = 10, Reward = 75},
		{Name = "Epic", Chance = 4, Reward = 200},
		{Name = "Legendary", Chance = 0.9, Reward = 1000},
		{Name = "Mythic", Chance = 0.1, Reward = 10000},
	}
}

-- GANTI DENGAN USER ID ADMIN
local ADMINS = {
	[123456789] = true,
}

----------------------------------------------------
-- DATASTORE
----------------------------------------------------

local DataStore = DataStoreService:GetDataStore("StealEgg_PlayerData_V1")

local PlayerData = {}

local DEFAULT_DATA = {
	Coins = 0,
	Eggs = 0,
	StolenEggs = 0,

	Inventory = {},

	Settings = {
		AutoCollect = false,
		AutoReward = true,
		AutoEvent = false,
		VisualEgg = true,
		VisualEvent = true,
	}
}

----------------------------------------------------
-- REMOTES
----------------------------------------------------

local RemoteFolder = Instance.new("Folder")
RemoteFolder.Name = "EggGameRemotes"
RemoteFolder.Parent = ReplicatedStorage

local function CreateRemote(name)
	local remote = Instance.new("RemoteEvent")
	remote.Name = name
	remote.Parent = RemoteFolder
	return remote
end

local CollectEgg = CreateRemote("CollectEgg")
local StealEgg = CreateRemote("StealEgg")
local ToggleSetting = CreateRemote("ToggleSetting")
local StartEvent = CreateRemote("StartEvent")
local AdminAction = CreateRemote("AdminAction")

----------------------------------------------------
-- UTILITY
----------------------------------------------------

local function DeepCopy(data)
	local copy = {}

	for key, value in pairs(data) do
		if type(value) == "table" then
			copy[key] = DeepCopy(value)
		else
			copy[key] = value
		end
	end

	return copy
end

local function GetRarity()
	local roll = math.random() * 100
	local current = 0

	for _, rarity in ipairs(CONFIG.EGG_RARITIES) do
		current += rarity.Chance

		if roll <= current then
			return rarity
		end
	end

	return CONFIG.EGG_RARITIES[1]
end

local function IsAdmin(player)
	return ADMINS[player.UserId] == true
end

----------------------------------------------------
-- LEADERSTATS
----------------------------------------------------

local function CreateStats(player, data)

	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local coins = Instance.new("IntValue")
	coins.Name = "Coins"
	coins.Value = data.Coins
	coins.Parent = leaderstats

	local eggs = Instance.new("IntValue")
	eggs.Name = "Eggs"
	eggs.Value = data.Eggs
	eggs.Parent = leaderstats

	local stolen = Instance.new("IntValue")
	stolen.Name = "StolenEggs"
	stolen.Value = data.StolenEggs
	stolen.Parent = leaderstats
end

----------------------------------------------------
-- LOAD
----------------------------------------------------

local function LoadPlayer(player)

	local data

	local success, result = pcall(function()
		return DataStore:GetAsync("Player_" .. player.UserId)
	end)

	if success and result then
		data = result
	else
		data = DeepCopy(DEFAULT_DATA)
	end

	PlayerData[player] = data

	CreateStats(player, data)

	------------------------------------------------
	-- VIP
	------------------------------------------------

	if CONFIG.VIP_GAMEPASS_ID > 0 then

		local successVIP, ownsVIP = pcall(function()
			return MarketplaceService:UserOwnsGamePassAsync(
				player.UserId,
				CONFIG.VIP_GAMEPASS_ID
			)
		end)

		if successVIP and ownsVIP then
			player:SetAttribute("VIP", true)
		else
			player:SetAttribute("VIP", false)
		end

	else
		player:SetAttribute("VIP", false)
	end

	------------------------------------------------
	-- PRIVATE SERVER
	------------------------------------------------

	if game.PrivateServerId ~= "" then
		player:SetAttribute("PrivateServer", true)
	else
		player:SetAttribute("PrivateServer", false)
	end
end

----------------------------------------------------
-- SAVE
----------------------------------------------------

local function SavePlayer(player)

	local data = PlayerData[player]

	if not data then
		return
	end

	local stats = player:FindFirstChild("leaderstats")

	if stats then

		local coins = stats:FindFirstChild("Coins")
		local eggs = stats:FindFirstChild("Eggs")
		local stolen = stats:FindFirstChild("StolenEggs")

		if coins then
			data.Coins = coins.Value
		end

		if eggs then
			data.Eggs = eggs.Value
		end

		if stolen then
			data.StolenEggs = stolen.Value
		end
	end

	pcall(function()
		DataStore:SetAsync(
			"Player_" .. player.UserId,
			data
		)
	end)
end

----------------------------------------------------
-- PLAYER
----------------------------------------------------

Players.PlayerAdded:Connect(function(player)

	LoadPlayer(player)

	player.CharacterAdded:Connect(function(character)

		local humanoid = character:WaitForChild("Humanoid")

		humanoid.Died:Connect(function()
			-- tempat sistem tambahan
		end)

	end)
end)

Players.PlayerRemoving:Connect(function(player)

	SavePlayer(player)

	PlayerData[player] = nil
end)

----------------------------------------------------
-- CREATE EGG
----------------------------------------------------

local function CreateEgg(position)

	local rarity = GetRarity()

	local egg = Instance.new("Part")

	egg.Name = rarity.Name .. "_Egg"
	egg.Size = Vector3.new(2, 2, 2)
	egg.Shape = Enum.PartType.Ball
	egg.Anchored = true
	egg.CanCollide = false

	egg.Position = position

	egg:SetAttribute("IsEgg", true)
	egg:SetAttribute("Rarity", rarity.Name)
	egg:SetAttribute("Reward", rarity.Reward)

	egg.Parent = workspace

	local billboard = Instance.new("BillboardGui")

	billboard.Size = UDim2.new(0, 150, 0, 50)
	billboard.StudsOffset = Vector3.new(0, 2.5, 0)
	billboard.AlwaysOnTop = true
	billboard.Parent = egg

	local text = Instance.new("TextLabel")

	text.Size = UDim2.fromScale(1, 1)
	text.BackgroundTransparency = 1
	text.Text = rarity.Name .. "\n+" .. rarity.Reward .. " Coins"
	text.TextScaled = true
	text.Parent = billboard

	return egg
end

----------------------------------------------------
-- EGG SPAWNER
----------------------------------------------------

task.spawn(function()

	while true do

		task.wait(10)

		local folder = workspace:FindFirstChild("EggSpawn")

		if folder then

			local points = folder:GetChildren()

			if #points > 0 then

				local point = points[
					math.random(1, #points)
				]

				CreateEgg(point.Position + Vector3.new(0, 2, 0))
			end
		end
	end
end)

----------------------------------------------------
-- COLLECT EGG
----------------------------------------------------

local function Collect(player, egg, stolen)

	if not PlayerData[player] then
		return
	end

	if not egg then
		return
	end

	if not egg:IsDescendantOf(workspace) then
		return
	end

	if egg:GetAttribute("IsEgg") ~= true then
		return
	end

	local character = player.Character

	if not character then
		return
	end

	local root = character:FindFirstChild("HumanoidRootPart")

	if not root then
		return
	end

	local distance =
		(root.Position - egg.Position).Magnitude

	if distance > CONFIG.AUTO_COLLECT_DISTANCE then
		return
	end

	local data = PlayerData[player]

	if #data.Inventory >= CONFIG.MAX_INVENTORY then
		return
	end

	local rarity = egg:GetAttribute("Rarity")
	local reward = egg:GetAttribute("Reward") or 1

	table.insert(data.Inventory, {
		Rarity = rarity,
		Reward = reward
	})

	data.Eggs += 1

	if stolen then
		data.StolenEggs += 1
	end

	local stats = player:FindFirstChild("leaderstats")

	if stats then

		stats.Eggs.Value = data.Eggs

		stats.StolenEggs.Value =
			data.StolenEggs

		local coins = stats.Coins

		local finalReward = reward

		if player:GetAttribute("VIP") then
			finalReward *= 2
		end

		coins.Value += finalReward
		data.Coins = coins.Value
	end

	egg:Destroy()
end

----------------------------------------------------
-- REMOTE COLLECT
----------------------------------------------------

CollectEgg.OnServerEvent:Connect(function(player, egg)

	Collect(
		player,
		egg,
		false
	)

end)

----------------------------------------------------
-- STEAL EGG
----------------------------------------------------

StealEgg.OnServerEvent:Connect(function(player, egg)

	-- server validation
	Collect(
		player,
		egg,
		true
	)

end)

----------------------------------------------------
-- AUTO COLLECT
----------------------------------------------------

task.spawn(function()

	while true do

		task.wait(1)

		for player, data in pairs(PlayerData) do

			if data.Settings.AutoCollect then

				local character =
					player.Character

				if character then

					local root =
						character:FindFirstChild(
							"HumanoidRootPart"
						)

					if root then

						for _, object in ipairs(
							workspace:GetChildren()
						) do

							if object:IsA("BasePart")
								and object:GetAttribute(
									"IsEgg"
								) == true then

								local distance =
									(root.Position -
										object.Position).Magnitude

								if distance <=
									CONFIG.AUTO_COLLECT_DISTANCE then

									Collect(
										player,
										object,
										false
									)

									break
								end
							end
						end
					end
				end
			end
		end
	end
end)

----------------------------------------------------
-- SETTINGS
----------------------------------------------------

ToggleSetting.OnServerEvent:Connect(
	function(player, setting, value)

		local data = PlayerData[player]

		if not data then
			return
		end

		if data.Settings[setting] ~= nil then

			if type(value) == "boolean" then

				data.Settings[setting] = value
			end
		end
	end
)

----------------------------------------------------
-- AUTO REWARD
----------------------------------------------------

task.spawn(function()

	while true do

		task.wait(
			CONFIG.AUTO_REWARD_TIME
		)

		for player, data in pairs(PlayerData) do

			if data.Settings.AutoReward then

				local stats =
					player:FindFirstChild(
						"leaderstats"
					)

				if stats then

					local reward = 50

					if player:GetAttribute("VIP") then
						reward *= 2
					end

					stats.Coins.Value += reward

					data.Coins =
						stats.Coins.Value
				end
			end
		end
	end
end)

----------------------------------------------------
-- TREADMILL
----------------------------------------------------

local function IsOnTreadmill(player)

	local treadmill =
		workspace:FindFirstChild(
			"Treadmill"
		)

	if not treadmill then
		return false
	end

	local character =
		player.Character

	if not character then
		return false
	end

	local root =
		character:FindFirstChild(
			"HumanoidRootPart"
		)

	if not root then
		return false
	end

	return (
		root.Position -
			treadmill.Position
	).Magnitude <= 10
end

task.spawn(function()

	while true do

		task.wait(1)

		for player, data in pairs(PlayerData) do

			if IsOnTreadmill(player) then

				local stats =
					player:FindFirstChild(
						"leaderstats"
					)

				if stats then

					local reward =
						CONFIG.TREADMILL_REWARD

					if player:GetAttribute(
						"VIP"
					) then

						reward *= 2
					end

					stats.Coins.Value += reward

					data.Coins =
						stats.Coins.Value
				end
			end
		end
	end
end)

----------------------------------------------------
-- EVENTS
----------------------------------------------------

local EventActive = false

local function StartGlobalEvent()

	if EventActive then
		return
	end

	EventActive = true

	print("GLOBAL EGG EVENT STARTED!")

	for player, data in pairs(PlayerData) do

		if data.Settings.AutoEvent then

			local stats =
				player:FindFirstChild(
					"leaderstats"
				)

			if stats then
				stats.Coins.Value += 500
				data.Coins =
					stats.Coins.Value
			end
		end
	end

	task.delay(
		CONFIG.EVENT_DURATION,
		function()

			EventActive = false

			print("GLOBAL EGG EVENT ENDED!")
		end
	)
end

----------------------------------------------------
-- AUTO EVENT
----------------------------------------------------

task.spawn(function()

	while true do

		task.wait(
			CONFIG.AUTO_EVENT_TIME
		)

		StartGlobalEvent()
	end
end)

----------------------------------------------------
-- MANUAL EVENT
----------------------------------------------------

StartEvent.OnServerEvent:Connect(function(player)

	if EventActive then
		return
	end

	StartGlobalEvent()

end)

----------------------------------------------------
-- ADMIN SYSTEM
----------------------------------------------------

AdminAction.OnServerEvent:Connect(
	function(player, action, amount)

		if not IsAdmin(player) then
			return
		end

		local stats =
			player:FindFirstChild(
				"leaderstats"
			)

		if not stats then
			return
		end

		if action == "GiveCoins" then

			amount = tonumber(amount)

			if amount and amount > 0 then
				stats.Coins.Value += amount
			end

		elseif action == "GiveEgg" then

			amount = tonumber(amount)

			if amount and amount > 0 then

				for i = 1, amount do

					table.insert(
						PlayerData[player].Inventory,
						{
							Rarity = "Admin",
							Reward = 1000
						}
					)

				end

				stats.Eggs.Value += amount
			end

		elseif action == "StartEvent" then

			StartGlobalEvent()
		end
	end
)

----------------------------------------------------
-- AUTOSAVE
----------------------------------------------------

task.spawn(function()

	while true do

		task.wait(60)

		for player in pairs(PlayerData) do
			SavePlayer(player)
		end
	end
end)

----------------------------------------------------
-- SERVER SHUTDOWN
----------------------------------------------------

game:BindToClose(function()

	for player in pairs(PlayerData) do
		SavePlayer(player)
	end

	task.wait(2)
end)

print("================================")
print("STEAL AN EGG SYSTEM LOADED")
print("Eggs       : ON")
print("AutoCollect: ON")
print("Events     : ON")
print("Treadmill  : ON")
print("VIP        : ON")
print("Inventory  : ON")
print("DataStore  : ON")
print("Admin      : ON")
print("================================")
