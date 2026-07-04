local CriminalityV1 = {}

do
	local MainTabModule = {}

	local Players = game:GetService("Players")
	local RunService = game:GetService("RunService")
	local UserInputService = game:GetService("UserInputService")
	local ReplicatedStorage = game:GetService("ReplicatedStorage")
	local Workspace = game:GetService("Workspace")

	local LocalPlayer = Players.LocalPlayer
	local Camera = Workspace.CurrentCamera

	function MainTabModule.CreateSilentAimV2()
		local SilentAim = {
			enabled = false,
			settings = {
				checkDowned = true,
				teamCheck = false,
				checkWhitelist = false,
				checkTarget = false,
				fovCircleCentered = false,
				drawColor = Color3.fromRGB(255, 255, 255),
				drawSize = 100,
				useHitChance = true,
				hitChance = 100,
				checkWall = true,
				targetPart = "Head",
				actualPart = "Head",
				drawCircle = false,
				targetChangeTime = 0.5,
				showHitPartNotification = false,
				notificationSize = 20,
				maxDistance = 120,
			},
			currentTarget = nil,
			visualizeConnection = nil,
			randomizerTimer = 0,
		}

		local ValidParts = { "Head", "HumanoidRootPart", "Left Arm", "Right Arm", "Left Leg", "Right Leg" }

		local circle = Drawing.new("Circle")
		circle.Visible = false
		circle.Transparency = 1
		circle.Thickness = 1.5
		circle.Filled = false

		local notificationText = Drawing.new("Text")
		notificationText.Visible = false
		notificationText.Color = Color3.fromRGB(255, 255, 255)
		notificationText.Center = true
		notificationText.Outline = true
		notificationText.Font = 2

		local function UpdateCircleProps()
			circle.Color = SilentAim.settings.drawColor
			circle.Radius = SilentAim.settings.drawSize
		end

		local DrawCircleConnection
		local function UpdateCircle()
			if DrawCircleConnection then
				DrawCircleConnection:Disconnect()
				DrawCircleConnection = nil
			end
			if SilentAim.settings.drawCircle then
				circle.Visible = true
				UpdateCircleProps()
				local screenCenter = SilentAim.settings.fovCircleCentered
						and Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
					or nil
				DrawCircleConnection = RunService.RenderStepped:Connect(function()
					circle.Position = screenCenter or UserInputService:GetMouseLocation()
				end)
			else
				circle.Visible = false
			end
		end

		local function ShowHitPartNotification(newPart)
			if not SilentAim.settings.showHitPartNotification then
				return
			end
			notificationText.Size = SilentAim.settings.notificationSize
			notificationText.Text = "New Hit Part: " .. newPart
			notificationText.Position = Vector2.new(Camera.ViewportSize.X - 200, Camera.ViewportSize.Y / 2)
			notificationText.Visible = true
			task.delay(1, function()
				notificationText.Visible = false
			end)
		end

		local randomTargetConnection
		local function StopRandomizer()
			if randomTargetConnection then
				randomTargetConnection:Disconnect()
				randomTargetConnection = nil
			end
		end

		local function StartRandomizer()
			StopRandomizer()
			if not SilentAim.enabled or SilentAim.settings.targetPart ~= "Random" then
				return
			end
			SilentAim.randomizerTimer = 0
			randomTargetConnection = RunService.Heartbeat:Connect(function(dt)
				SilentAim.randomizerTimer = (SilentAim.randomizerTimer or 0) + dt
				if SilentAim.randomizerTimer >= SilentAim.settings.targetChangeTime then
					local currentPart = SilentAim.settings.actualPart
					local newPart = ValidParts[math.random(1, #ValidParts)]
					if newPart ~= currentPart then
						SilentAim.settings.actualPart = newPart
						ShowHitPartNotification(newPart)
					end
					SilentAim.randomizerTimer = 0
				end
			end)
		end

		local function IsPlayerDowned(p)
			if not p or not p.Character then
				return false
			end
			local hum = p.Character:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health <= 15 then
				return true
			end

			local cs = p.Character:FindFirstChild("CharStats")
			if cs then
				local downed = cs:FindFirstChild("Downed")
				if downed and typeof(downed.Value) == "boolean" then
					return downed.Value
				end
			end
			return false
		end

		local function GetSilentAimTarget(IsPlayerWhitelisted, IsPlayerTargeted)
			if not SilentAim.enabled then
				return nil
			end
			local closest, minDist = nil, SilentAim.settings.drawSize
			local screenCenter = SilentAim.settings.fovCircleCentered
					and Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
				or UserInputService:GetMouseLocation()
			local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
			if not localRoot then
				return nil
			end

			for _, player in pairs(Players:GetPlayers()) do
				if player == LocalPlayer then
					continue
				end
				if SilentAim.settings.teamCheck and player.Team == LocalPlayer.Team then
					continue
				end
				if SilentAim.settings.checkWhitelist and IsPlayerWhitelisted and IsPlayerWhitelisted(player) then
					continue
				end
				if SilentAim.settings.checkTarget and IsPlayerTargeted and not IsPlayerTargeted(player) then
					continue
				end

				local character = player.Character
				if not character then
					continue
				end
				local humanoid = character:FindFirstChildOfClass("Humanoid")
				local root = character:FindFirstChild("HumanoidRootPart")
				if
					not humanoid
					or not root
					or humanoid.Health <= 0
					or character:FindFirstChildOfClass("ForceField")
				then
					continue
				end
				if SilentAim.settings.checkDowned and IsPlayerDowned(player) then
					continue
				end

				local partName = SilentAim.settings.targetPart == "Random" and SilentAim.settings.actualPart
					or SilentAim.settings.targetPart
				local part = character:FindFirstChild(partName)
				if not part then
					continue
				end

				if (localRoot.Position - part.Position).Magnitude > SilentAim.settings.maxDistance then
					continue
				end
				if SilentAim.settings.checkWall then
					local parts = Camera:GetPartsObscuringTarget(
						{ part.Position },
						{ Camera, LocalPlayer.Character, character }
					)
					if #parts > 0 then
						continue
					end
				end

				local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
				if onScreen then
					local distance = (screenCenter - Vector2.new(screenPos.X, screenPos.Y)).Magnitude
					if distance < minDist then
						closest = player
						minDist = distance
					end
				end
			end

			if closest and SilentAim.settings.useHitChance and math.random(1, 100) > SilentAim.settings.hitChance then
				return nil
			end
			return closest
		end

		function SilentAim:Toggle(val, IsPlayerWhitelisted, IsPlayerTargeted)
			self.enabled = val

			if self.visualizeConnection then
				self.visualizeConnection:Disconnect()
				self.visualizeConnection = nil
			end
			if self.silentAimTask then
				task.cancel(self.silentAimTask)
				self.silentAimTask = nil
			end

			if val then
				self.silentAimTask = task.spawn(function()
					while self.enabled do
						self.currentTarget = GetSilentAimTarget(IsPlayerWhitelisted, IsPlayerTargeted)
						task.wait(0.1)
					end
				end)
				StartRandomizer()

				local success1, visualizeEvent = pcall(function()
					return ReplicatedStorage:WaitForChild("Events2", 5):WaitForChild("Visualize", 5)
				end)

				local success2, damageEvent = pcall(function()
					return ReplicatedStorage:WaitForChild("Events", 5):WaitForChild("ZFKLF__H", 5)
				end)

				if success1 and success2 and visualizeEvent and damageEvent then
					self.visualizeConnection = visualizeEvent.Event:Connect(
						function(_, shotCode, _, gun, _, startPos, bulletsPerShot)
							local target = self.currentTarget
							if not self.enabled or not gun or not target or not target.Character then
								return
							end
							local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
							if not tool or gun ~= tool or target.Character:FindFirstChildOfClass("ForceField") then
								return
							end

							local partName = self.settings.targetPart == "Random" and self.settings.actualPart
								or self.settings.targetPart
							local validPartName = partName
							if partName == "HumanoidRootPart" then
								validPartName = target.Character:FindFirstChild("UpperTorso") and "UpperTorso"
									or target.Character:FindFirstChild("Torso") and "Torso"
									or target.Character:FindFirstChild("LowerTorso") and "LowerTorso"
									or "Head"
							end

							local hitPart = target.Character:FindFirstChild(validPartName)
							if not hitPart then
								return
							end

							local hitPos = hitPart.Position
							local bullets = {}
							local bulletCount = type(bulletsPerShot) == "table" and #bulletsPerShot or 1
							for i = 1, math.clamp(bulletCount, 1, 100) do
								bullets[i] = CFrame.new(startPos, hitPos).LookVector
							end

							task.wait(0.005)
							for idx, direction in pairs(bullets) do
								pcall(function()
									damageEvent:FireServer("🧈", gun, shotCode, idx, hitPart, hitPos, direction)
								end)
							end

							if gun:FindFirstChild("Hitmarker") then
								pcall(function()
									gun.Hitmarker:Fire(hitPart)
								end)
							end
						end
					)
				end
			else
				StopRandomizer()
			end
		end

		function SilentAim:UpdateCircle()
			UpdateCircle()
		end

		function SilentAim:UpdateCircleProps()
			UpdateCircleProps()
		end

		function SilentAim:StartRandomizer()
			StartRandomizer()
		end

		function SilentAim:StopRandomizer()
			StopRandomizer()
		end

		return SilentAim
	end

	function MainTabModule.CreateSilentAimV1()
		local SilentAimV1 = {
			enabled = false,
			settings = {
				targetPart = "Head",
				hitChance = 100,
				wallCheck = true,
				maxDistance = 750,
				showFOVCircle = false,
				fovSize = 100,
				fovCircleCentered = false,
				checkDowned = true,
				teamCheck = false,
				checkWhitelist = false,
				checkTarget = false,
				useHitChance = true,
			},
			validTargets = {},
			lastUpdateTime = 0,
			updateInterval = 0.05,
			cachedPlayers = {},
			lastPlayerCount = 0,
		}

		local FOVCircle = Drawing.new("Circle")
		FOVCircle.Color = Color3.new(1, 1, 1)
		FOVCircle.Thickness = 2
		FOVCircle.Filled = false
		FOVCircle.Transparency = 0.5
		FOVCircle.Visible = false
		FOVCircle.Radius = 100

		local raycastParams = RaycastParams.new()
		raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
		raycastParams.IgnoreWater = true

		local FOVCircleConnection
		local function UpdateCircle()
			if FOVCircleConnection then
				FOVCircleConnection:Disconnect()
				FOVCircleConnection = nil
			end

			if SilentAimV1.settings.showFOVCircle then
				FOVCircle.Visible = true
				FOVCircle.Radius = SilentAimV1.settings.fovSize

				FOVCircleConnection = RunService.Heartbeat:Connect(function()
					if SilentAimV1.settings.fovCircleCentered then
						local screenSize = Camera.ViewportSize
						FOVCircle.Position = Vector2.new(screenSize.X * 0.5, screenSize.Y * 0.5)
					else
						FOVCircle.Position = UserInputService:GetMouseLocation()
					end
				end)
			else
				FOVCircle.Visible = false
			end
		end

		local downedCache = {}
		local downedCacheTime = {}
		local DOWNED_CACHE_DURATION = 0.5

		local function IsPlayerDowned(p)
			if not p or not p.Character then
				return false
			end

			local currentTime = tick()
			local playerId = p.UserId

			if
				downedCache[playerId]
				and downedCacheTime[playerId]
				and (currentTime - downedCacheTime[playerId]) < DOWNED_CACHE_DURATION
			then
				return downedCache[playerId]
			end

			local hum = p.Character:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health <= 15 then
				downedCache[playerId] = true
				downedCacheTime[playerId] = currentTime
				return true
			end

			local cs = p.Character:FindFirstChild("CharStats")
			if cs then
				local downed = cs:FindFirstChild("Downed")
				if downed and typeof(downed.Value) == "boolean" then
					downedCache[playerId] = downed.Value
					downedCacheTime[playerId] = currentTime
					return downed.Value
				end
			end

			downedCache[playerId] = false
			downedCacheTime[playerId] = currentTime
			return false
		end

		local function UpdateCachedPlayers()
			local currentPlayerCount = #Players:GetPlayers()
			if currentPlayerCount == SilentAimV1.lastPlayerCount then
				return
			end

			SilentAimV1.cachedPlayers = {}
			for _, player in ipairs(Players:GetPlayers()) do
				if player ~= LocalPlayer then
					if not (SilentAimV1.settings.teamCheck and player.Team == LocalPlayer.Team) then
						table.insert(SilentAimV1.cachedPlayers, player)
					end
				end
			end
			SilentAimV1.lastPlayerCount = currentPlayerCount
		end

		local renderConnection
		function SilentAimV1:Toggle(val, IsPlayerWhitelisted, IsPlayerTargeted)
			self.enabled = val

			if renderConnection then
				renderConnection:Disconnect()
				renderConnection = nil
			end

			if val then
				renderConnection = RunService.Heartbeat:Connect(function()
					local currentTime = tick()

					if currentTime - self.lastUpdateTime < self.updateInterval then
						return
					end
					self.lastUpdateTime = currentTime

					self.validTargets = {}
					if not self.enabled then
						return
					end

					local character = LocalPlayer.Character
					if not character then
						return
					end
					local root = character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso")
					if not root then
						return
					end

					local origin = Camera.CFrame.Position
					local screenCenter = self.settings.fovCircleCentered
							and Vector2.new(Camera.ViewportSize.X * 0.5, Camera.ViewportSize.Y * 0.5)
						or UserInputService:GetMouseLocation()

					UpdateCachedPlayers()

					local maxDistanceSquared = self.settings.maxDistance * self.settings.maxDistance
					local fovSizeSquared = self.settings.fovSize * self.settings.fovSize

					for _, player in ipairs(self.cachedPlayers) do
						if self.settings.checkWhitelist and IsPlayerWhitelisted and IsPlayerWhitelisted(player) then
							continue
						end
						if self.settings.checkTarget and IsPlayerTargeted and not IsPlayerTargeted(player) then
							continue
						end

						local targetChar = player.Character
						if not targetChar then
							continue
						end

						local humanoid = targetChar:FindFirstChildOfClass("Humanoid")
						if not humanoid or humanoid.Health <= 0 then
							continue
						end
						if targetChar:FindFirstChildOfClass("ForceField") then
							continue
						end
						if self.settings.checkDowned and IsPlayerDowned(player) then
							continue
						end

						local targetPart = targetChar:FindFirstChild(self.settings.targetPart)
						if not targetPart then
							continue
						end

						local distanceSquared = (origin - targetPart.Position).Magnitude
						if distanceSquared > maxDistanceSquared then
							continue
						end

						local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
						if not onScreen then
							continue
						end

						local screenDistance = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter)
						local magSquared = screenDistance.X * screenDistance.X + screenDistance.Y * screenDistance.Y
						if magSquared > fovSizeSquared then
							continue
						end

						if self.settings.wallCheck then
							raycastParams.FilterDescendantsInstances = { character, Camera }

							local result = Workspace:Raycast(
								origin,
								(targetPart.Position - origin).Unit * self.settings.maxDistance,
								raycastParams
							)
							if not result or not result.Instance or not result.Instance:IsDescendantOf(targetChar) then
								continue
							end
						end

						table.insert(self.validTargets, {
							Player = player,
							Part = targetPart,
							ScreenDistance = math.sqrt(magSquared),
						})
					end

					if #self.validTargets > 1 then
						table.sort(self.validTargets, function(a, b)
							return a.ScreenDistance < b.ScreenDistance
						end)
					end
				end)

				local oldNamecall
				oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
					local args = { ... }
					local method = getnamecallmethod()

					if not checkcaller() and SilentAimV1.enabled and method == "Raycast" then
						if #SilentAimV1.validTargets > 0 then
							if
								not SilentAimV1.settings.useHitChance
								or math.random(1, 100) <= SilentAimV1.settings.hitChance
							then
								local target = SilentAimV1.validTargets[1]
								if target and target.Part then
									local origin = Camera.CFrame.Position
									local direction = (target.Part.Position - origin).Unit * 9999
									args[2] = direction
									return oldNamecall(self, unpack(args))
								end
							end
						end
					end

					return oldNamecall(self, ...)
				end)
			end
		end

		function SilentAimV1:UpdateCircle()
			UpdateCircle()
		end

		function SilentAimV1:Cleanup()
			downedCache = {}
			downedCacheTime = {}
			self.cachedPlayers = {}
			self.validTargets = {}
		end

		return SilentAimV1
	end

	function MainTabModule.CreateRageBot()
		local RageBot = {
			enabled = false,
			settings = {
				checkDowned = true,
				wallCheck = true,
				hitlogEnabled = true,
				checkWhitelist = false,
				checkTarget = false,
				useFOV = false,
				teamCheck = false,
				fovRadius = 75,
				shootSpeed = 15,
				fireInterval = 0.17,
				maxDistance = 100,
				bulletTracerEnabled = false,
				tracerColor = Color3.fromRGB(255, 0, 0),
				showFOV = false,
			},
			loopTask = nil,
			lastShotTime = 0,
			lastHitNotify = {},
		}

		local NOTIFY_COOLDOWN = 0.35
		local WallbangSamples = 72
		local WallbangRadius = 10
		local WallbangHeight = 6

		local RageCircle = Drawing.new("Circle")
		RageCircle.Color = Color3.fromRGB(255, 255, 255)
		RageCircle.Thickness = 1.5
		RageCircle.Filled = false
		RageCircle.Transparency = 0.5
		RageCircle.Radius = 75
		RageCircle.Visible = false

		local RageCircleConnection
		local function UpdateRageCircle()
			if RageCircleConnection then
				RageCircleConnection:Disconnect()
				RageCircleConnection = nil
			end

			if RageBot.settings.showFOV then
				RageCircle.Visible = true
				RageCircle.Radius = RageBot.settings.fovRadius
				RageCircleConnection = RunService.RenderStepped:Connect(function()
					RageCircle.Position = UserInputService:GetMouseLocation()
				end)
			else
				RageCircle.Visible = false
			end
		end

		local function createTracer(startPos, endPos)
			if not RageBot.settings.bulletTracerEnabled then
				return
			end
			local tracer = Instance.new("Part")
			tracer.Anchored = true
			tracer.CanCollide = false
			tracer.Material = Enum.Material.Neon
			tracer.Color = RageBot.settings.tracerColor
			tracer.Shape = Enum.PartType.Cylinder
			local distance = (startPos - endPos).Magnitude
			tracer.Size = Vector3.new(distance, 0.12, 0.12)
			tracer.CFrame = CFrame.new((startPos + endPos) / 2, endPos) * CFrame.Angles(0, math.pi / 2, 0)
			tracer.Parent = Workspace
			coroutine.wrap(function()
				for t = 0, 1, 0.02 do
					if tracer then
						tracer.Transparency = t
						task.wait(1 / 50)
					end
				end
				if tracer then
					tracer:Destroy()
				end
			end)()
		end

		local function RandomString(len)
			local s = ""
			for i = 1, len do
				s = s .. string.char(math.random(97, 122))
			end
			return s
		end

		local function IsPlayerDowned(p)
			if not p or not p.Character then
				return false
			end
			local hum = p.Character:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health <= 15 then
				return true
			end

			local cs = p.Character:FindFirstChild("CharStats")
			if cs then
				local downed = cs:FindFirstChild("Downed")
				if downed and typeof(downed.Value) == "boolean" then
					return downed.Value
				end
			end
			return false
		end

		local function IsValidTarget(p, IsPlayerWhitelisted, IsPlayerTargeted)
			if not p or not p.Character then
				return false
			end
			if p == LocalPlayer then
				return false
			end
			if RageBot.settings.teamCheck and p.Team == LocalPlayer.Team then
				return false
			end
			if RageBot.settings.checkWhitelist and IsPlayerWhitelisted(p) then
				return false
			end
			if RageBot.settings.checkTarget and not IsPlayerTargeted(p) then
				return false
			end
			local hum = p.Character:FindFirstChildOfClass("Humanoid")
			local hrp = p.Character:FindFirstChild("HumanoidRootPart")
			if not hum or not hrp then
				return false
			end
			if hum.Health <= 0 then
				return false
			end
			if p.Character:FindFirstChildOfClass("ForceField") then
				return false
			end
			if RageBot.settings.checkDowned and IsPlayerDowned(p) then
				return false
			end
			return true
		end

		local function GetHeadPart(char)
			if not char then
				return nil
			end
			return char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
		end

		local function MakeRaycastParams()
			local rp = RaycastParams.new()
			rp.FilterType = Enum.RaycastFilterType.Blacklist
			rp.FilterDescendantsInstances = {}
			if LocalPlayer.Character then
				table.insert(rp.FilterDescendantsInstances, LocalPlayer.Character)
			end
			rp.IgnoreWater = true
			return rp
		end

		local function FindWallbangPoint(origin, targetPart)
			if not origin or not targetPart then
				return nil
			end
			local base = targetPart.Position
			local rp = MakeRaycastParams()
			for i = 1, WallbangSamples do
				local angle = (i / WallbangSamples) * math.pi * 2
				local r = WallbangRadius * (0.6 + math.random() * 0.8)
				local yOff = (math.random() * 2 - 1) * WallbangHeight
				local offset = Vector3.new(math.cos(angle) * r, yOff, math.sin(angle) * r)
				local testPoint = base + offset
				local dir = (testPoint - origin)
				if dir.Magnitude > 0 then
					local result = Workspace:Raycast(origin, dir, rp)
					if result then
						if result.Instance and result.Instance:IsDescendantOf(targetPart.Parent) then
							return testPoint
						else
							local distHitToTarget = (result.Position - base).Magnitude
							if distHitToTarget <= 2.0 then
								return testPoint
							end
						end
					else
						return testPoint
					end
				end
			end
			return nil
		end

		local function GetClosestEnemy(IsPlayerWhitelisted, IsPlayerTargeted)
			local me = LocalPlayer.Character
			if not me or not me:FindFirstChild("HumanoidRootPart") then
				return nil
			end
			local closest, shortest = nil, math.huge
			local originPos = me.HumanoidRootPart.Position

			for _, p in ipairs(Players:GetPlayers()) do
				if IsValidTarget(p, IsPlayerWhitelisted, IsPlayerTargeted) then
					local head = GetHeadPart(p.Character)
					if head then
						local dist3D = (originPos - head.Position).Magnitude
						if dist3D <= RageBot.settings.maxDistance then
							if RageBot.settings.useFOV then
								local mousePos = UserInputService:GetMouseLocation()
								local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
								if onScreen then
									local d2 = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
									if d2 <= RageBot.settings.fovRadius and d2 < shortest then
										shortest = d2
										closest = p
									end
								end
							else
								if dist3D < shortest then
									shortest = dist3D
									closest = p
								end
							end
						end
					end
				end
			end
			return closest
		end

		local function SendHitNotification(targetPlayer, health, Library)
			if not RageBot.settings.hitlogEnabled then
				return
			end
			if not targetPlayer then
				return
			end
			local now = tick()
			local id = targetPlayer.UserId or targetPlayer.Name
			if RageBot.lastHitNotify[id] and now - RageBot.lastHitNotify[id] < NOTIFY_COOLDOWN then
				return
			end
			RageBot.lastHitNotify[id] = now

			if typeof(Library) == "table" and type(Library.Notify) == "function" then
				pcall(function()
					Library:Notify({
						Title = "HitLog",
						Description = "Hit " .. (targetPlayer.Name or "Unknown") .. " | Health: " .. tostring(
							math.floor(tonumber(health) or 0)
						),
						Time = 2,
					})
				end)
			end
		end

		local function Shoot(target, Library)
			if not target or not target.Character then
				return
			end
			local head = GetHeadPart(target.Character)
			if not head then
				return
			end
			if not LocalPlayer.Character then
				return
			end
			local tool = LocalPlayer.Character:FindFirstChildOfClass("Tool")
			if not tool then
				return
			end
			local values = tool:FindFirstChild("Values")
			local hitMarker = tool:FindFirstChild("Hitmarker")
			if not values or not hitMarker then
				return
			end
			local ammo = values:FindFirstChild("SERVER_Ammo")
			if not ammo or ammo.Value <= 0 then
				return
			end

			local handle = tool:FindFirstChild("WeaponHandle")
			local origin = (handle and handle.Position) or Camera.CFrame.Position
			local aimPos = head.Position
			local dir = (aimPos - origin)
			local rp = MakeRaycastParams()

			local ray = Workspace:Raycast(origin, dir, rp)
			local lineOfSight = false
			if ray and ray.Instance and ray.Instance:IsDescendantOf(target.Character) then
				lineOfSight = true
			elseif not ray then
				lineOfSight = true
			end

			local chosenAimPoint = aimPos
			local wallbangFound = false
			if not lineOfSight then
				local found = FindWallbangPoint(origin, head)
				if found then
					chosenAimPoint = found
					wallbangFound = true
				end
			end

			if not lineOfSight and RageBot.settings.wallCheck and not wallbangFound then
				return
			end

			local finalDir = (chosenAimPoint - origin).Unit
			local key = RandomString(30) .. "0"

			pcall(function()
				if
					ReplicatedStorage
					and ReplicatedStorage:FindFirstChild("Events")
					and ReplicatedStorage.Events:FindFirstChild("GNX_S")
				then
					ReplicatedStorage.Events.GNX_S:FireServer(tick(), key, tool, "FDS9I83", origin, { finalDir }, false)
				end
			end)
			pcall(function()
				if
					ReplicatedStorage
					and ReplicatedStorage:FindFirstChild("Events")
					and ReplicatedStorage.Events:FindFirstChild("ZFKLF__H")
				then
					ReplicatedStorage.Events.ZFKLF__H:FireServer("🧈", tool, key, 1, head, chosenAimPoint, finalDir)
				end
			end)

			ammo.Value = math.max(0, ammo.Value - 1)
			pcall(function()
				hitMarker:Fire(head)
			end)
			createTracer(origin, chosenAimPoint)

			local hum = target.Character and target.Character:FindFirstChildOfClass("Humanoid")
			local remaining = hum and hum.Health or 0
			SendHitNotification(target, remaining, Library)
		end

		function RageBot:Toggle(val, IsPlayerWhitelisted, IsPlayerTargeted, Library)
			self.enabled = val
			if val then
				if self.loopTask then
					return
				end
				self.loopTask = task.spawn(function()
					while self.enabled and LocalPlayer.Character do
						local ok, tool = pcall(function()
							return LocalPlayer.Character:FindFirstChildOfClass("Tool")
						end)
						if not ok or not tool then
							task.wait(0.2)
							continue
						end

						local target = GetClosestEnemy(IsPlayerWhitelisted, IsPlayerTargeted)
						if target then
							local now = tick()
							if now - self.lastShotTime >= self.settings.fireInterval then
								Shoot(target, Library)
								self.lastShotTime = now
							end
						end
						task.wait(0.05)
					end
					self.loopTask = nil
				end)
			end
		end

		function RageBot:UpdateShootSpeed(val)
			self.settings.shootSpeed = val
			self.settings.fireInterval = math.max(0.03, 0.35 - (val * 0.012))
		end

		function RageBot:UpdateFOVCircle()
			UpdateRageCircle()
		end

		return RageBot
	end

	function MainTabModule.CreateAimbot()
		local Aimbot = {
			enabled = false,
			settings = {
				usePrediction = false,
				wallCheckEnabled = true,
				lockKeyActive = false,
				lockedTarget = nil,
				predictionMultiplier = 0.10,
				showFOV = false,
				fovRadius = 150,
				teamCheckEnabled = false,
				checkWhitelist = false,
				checkTarget = false,
				checkDowned = false,
				lockedPart = "Head",
				lockMethod = "Toggle",
				showCircleButton = false,
				circleCenterOnly = false,
				randomChangeTime = 0.5,
				currentRandomPart = "Head",
				randomTimer = 0,
				smoothingX = 25,
				smoothingY = 25,
				circleSize = 40,
				circleXPercent = 50,
				circleYPercent = 50,
				autoChangeTarget = true,
				lockMouseButton = Enum.UserInputType.MouseButton2,
			},
			gui = nil,
			fovCircle = nil,
			button = nil,
			IsPlayerWhitelisted = nil,
			IsPlayerTargeted = nil,
		}

		local aimbotGUI = Instance.new("ScreenGui")
		aimbotGUI.Name = "AimbotGUI"
		aimbotGUI.Parent = game.CoreGui
		Aimbot.gui = aimbotGUI

		local aimbotFovCircle = Drawing.new("Circle")
		aimbotFovCircle.Color = Color3.fromRGB(255, 255, 255)
		aimbotFovCircle.Thickness = 1
		aimbotFovCircle.Filled = false
		aimbotFovCircle.Transparency = 0.5
		aimbotFovCircle.Radius = 150
		aimbotFovCircle.Visible = false
		Aimbot.fovCircle = aimbotFovCircle

		local FOVCircleConnection
		local function UpdateFOVCircle()
			if FOVCircleConnection then
				FOVCircleConnection:Disconnect()
				FOVCircleConnection = nil
			end

			if Aimbot.settings.showFOV then
				aimbotFovCircle.Visible = true
				aimbotFovCircle.Radius = Aimbot.settings.fovRadius
				FOVCircleConnection = RunService.RenderStepped:Connect(function()
					if Aimbot.settings.circleCenterOnly then
						aimbotFovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
					else
						aimbotFovCircle.Position = UserInputService:GetMouseLocation()
					end
				end)
			else
				aimbotFovCircle.Visible = false
			end
		end

		local function IsPlayerDowned(p)
			if not p or not p.Character then
				return false
			end
			local hum = p.Character:FindFirstChildOfClass("Humanoid")
			if hum and hum.Health <= 15 then
				return true
			end

			local cs = p.Character:FindFirstChild("CharStats")
			if cs then
				local downed = cs:FindFirstChild("Downed")
				if downed and typeof(downed.Value) == "boolean" then
					return downed.Value
				end
			end
			return false
		end

		local function GetClosestPlayerInFOV()
			local shortest = math.huge
			local fovCenter = Aimbot.settings.circleCenterOnly
					and Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
				or UserInputService:GetMouseLocation()
			local best = nil
			local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
			if not localRoot then
				return nil
			end

			for _, player in pairs(Players:GetPlayers()) do
				if player ~= LocalPlayer and player.Character then
					local humanoid = player.Character:FindFirstChild("Humanoid")
					if not humanoid or humanoid.Health <= 0 then
						continue
					end
					if Aimbot.settings.teamCheckEnabled and player.Team == LocalPlayer.Team then
						continue
					end
					if
						Aimbot.settings.checkWhitelist
						and Aimbot.IsPlayerWhitelisted
						and Aimbot.IsPlayerWhitelisted(player)
					then
						continue
					end
					if
						Aimbot.settings.checkTarget
						and Aimbot.IsPlayerTargeted
						and not Aimbot.IsPlayerTargeted(player)
					then
						continue
					end
					if Aimbot.settings.checkDowned and IsPlayerDowned(player) then
						continue
					end

					local partName = (Aimbot.settings.lockedPart == "Random" and Aimbot.settings.currentRandomPart)
						or Aimbot.settings.lockedPart
					local part = player.Character:FindFirstChild(partName)
					if not part then
						continue
					end

					local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
					if onScreen then
						local distance = (Vector2.new(screenPos.X, screenPos.Y) - fovCenter).Magnitude
						if distance < Aimbot.settings.fovRadius and distance < shortest then
							if Aimbot.settings.wallCheckEnabled then
								local origin = Camera.CFrame.Position
								local direction = (part.Position - origin)
								local raycastParams = RaycastParams.new()
								raycastParams.FilterDescendantsInstances = { LocalPlayer.Character }
								raycastParams.FilterType = Enum.RaycastFilterType.Blacklist

								local raycastResult = Workspace:Raycast(origin, direction, raycastParams)
								if raycastResult and not raycastResult.Instance:IsDescendantOf(player.Character) then
									continue
								end
							end
							best = player
							shortest = distance
						end
					end
				end
			end
			return best
		end

		local function SmoothAim(targetPosition)
			if not targetPosition then
				return
			end

			local currentCFrame = Camera.CFrame
			local targetCFrame = CFrame.new(currentCFrame.Position, targetPosition)

			local currentLook = currentCFrame.LookVector
			local targetLook = targetCFrame.LookVector

			local smoothFactorX = math.min(1, Aimbot.settings.smoothingX / 50)
			local smoothFactorY = math.min(1, Aimbot.settings.smoothingY / 50)

			local smoothedLookX = currentLook:lerp(targetLook, smoothFactorX)
			local smoothedLookY = currentLook:lerp(targetLook, smoothFactorY)

			local finalLook =
				Vector3.new(smoothedLookX.X, smoothedLookY.Y, (smoothedLookX.Z + smoothedLookY.Z) / 2).Unit

			Camera.CFrame = CFrame.new(currentCFrame.Position, currentCFrame.Position + finalLook)
		end

		function Aimbot:CreateCircleButton()
			if self.button then
				return
			end

			local button = Instance.new("TextButton")
			button.Parent = self.gui
			button.Name = "LockButton"
			button.BackgroundColor3 = Color3.new(0, 0, 0)
			button.Size = UDim2.new(0, self.settings.circleSize, 0, self.settings.circleSize)
			button.TextSize = 12
			button.TextColor3 = Color3.new(1, 1, 1)
			button.Text = "Lock"

			Instance.new("UICorner", button).CornerRadius = UDim.new(1, 8)
			local stroke = Instance.new("UIStroke", button)
			stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
			stroke.Color = Color3.new(1, 1, 1)
			stroke.LineJoinMode = Enum.LineJoinMode.Round
			stroke.Thickness = 1

			button.MouseButton1Click:Connect(function()
				if self.enabled then
					if self.settings.lockMethod == "Toggle" then
						self.settings.lockKeyActive = not self.settings.lockKeyActive
						if not self.settings.lockKeyActive then
							self.settings.lockedTarget = nil
						end
					elseif self.settings.lockMethod == "Hold" then
						self.settings.lockKeyActive = true
					end
				end
			end)

			self.button = button
			self:UpdateButtonPosition()
		end

		function Aimbot:UpdateButtonPosition()
			if not self.button then
				return
			end
			self.button.Size = UDim2.new(0, self.settings.circleSize, 0, self.settings.circleSize)
			self.button.Position = UDim2.new(
				self.settings.circleXPercent / 100,
				-self.settings.circleSize / 2,
				self.settings.circleYPercent / 100,
				-self.settings.circleSize / 2
			)
		end

		function Aimbot:Toggle(val)
			self.enabled = val
			if not val then
				self.settings.lockKeyActive = false
				self.settings.lockedTarget = nil
			end
		end

		function Aimbot:UpdateFOVCircle()
			UpdateFOVCircle()
		end

		function Aimbot:SetupInputBindings()
			UserInputService.InputBegan:Connect(function(input, gameProcessed)
				if gameProcessed or not self.enabled or self.settings.showCircleButton then
					return
				end

				if input.UserInputType == self.settings.lockMouseButton then
					if self.settings.lockMethod == "Hold" then
						self.settings.lockKeyActive = true
					elseif self.settings.lockMethod == "Toggle" then
						self.settings.lockKeyActive = not self.settings.lockKeyActive
						if not self.settings.lockKeyActive then
							self.settings.lockedTarget = nil
						end
					end
				end
			end)

			UserInputService.InputEnded:Connect(function(input)
				if self.settings.lockMethod ~= "Hold" or self.settings.showCircleButton or not self.enabled then
					return
				end

				if input.UserInputType == self.settings.lockMouseButton then
					self.settings.lockKeyActive = false
					self.settings.lockedTarget = nil
				end
			end)
		end

		function Aimbot:StartMainLoop()
			RunService.RenderStepped:Connect(function(deltaTime)
				if self.settings.lockedPart == "Random" then
					self.settings.randomTimer = self.settings.randomTimer + deltaTime
					if self.settings.randomTimer >= self.settings.randomChangeTime then
						local parts = { "Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg" }
						self.settings.currentRandomPart = parts[math.random(1, #parts)]
						self.settings.randomTimer = 0
					end
				end

				if self.settings.autoChangeTarget and self.settings.lockKeyActive then
					local needsNewTarget = false

					if self.settings.lockedTarget and self.settings.lockedTarget.Character then
						local humanoid = self.settings.lockedTarget.Character:FindFirstChild("Humanoid")

						if not humanoid or humanoid.Health <= 0 then
							needsNewTarget = true
						end

						if self.settings.checkDowned and IsPlayerDowned(self.settings.lockedTarget) then
							needsNewTarget = true
						end

						if self.settings.wallCheckEnabled and not needsNewTarget then
							local partName = (self.settings.lockedPart == "Random" and self.settings.currentRandomPart)
								or self.settings.lockedPart
							local targetPart = self.settings.lockedTarget.Character:FindFirstChild(partName)

							if targetPart then
								local origin = Camera.CFrame.Position
								local direction = (targetPart.Position - origin)
								local raycastParams = RaycastParams.new()
								raycastParams.FilterDescendantsInstances = { LocalPlayer.Character }
								raycastParams.FilterType = Enum.RaycastFilterType.Blacklist

								local raycastResult = Workspace:Raycast(origin, direction, raycastParams)
								if
									raycastResult
									and not raycastResult.Instance:IsDescendantOf(self.settings.lockedTarget.Character)
								then
									needsNewTarget = true
								end
							end
						end

						if not needsNewTarget then
							local partName = (self.settings.lockedPart == "Random" and self.settings.currentRandomPart)
								or self.settings.lockedPart
							local targetPart = self.settings.lockedTarget.Character:FindFirstChild(partName)

							if targetPart then
								local fovCenter = self.settings.circleCenterOnly
										and Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
									or UserInputService:GetMouseLocation()

								local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
								if onScreen then
									local distance = (Vector2.new(screenPos.X, screenPos.Y) - fovCenter).Magnitude
									if distance >= self.settings.fovRadius then
										needsNewTarget = true
									end
								else
									needsNewTarget = true
								end
							end
						end
					else
						needsNewTarget = true
					end

					if needsNewTarget then
						self.settings.lockedTarget = GetClosestPlayerInFOV()
					end
				end

				if
					self.enabled
					and self.settings.lockKeyActive
					and self.settings.lockedTarget
					and self.settings.lockedTarget.Character
				then
					local partName = (self.settings.lockedPart == "Random" and self.settings.currentRandomPart)
						or self.settings.lockedPart
					local targetPart = self.settings.lockedTarget.Character:FindFirstChild(partName)

					if targetPart then
						local aimPosition = targetPart.Position

						if
							self.settings.usePrediction
							and self.settings.lockedTarget.Character:FindFirstChild("HumanoidRootPart")
						then
							local velocity = self.settings.lockedTarget.Character.HumanoidRootPart.Velocity
							aimPosition = aimPosition + (velocity * self.settings.predictionMultiplier)
						end

						SmoothAim(aimPosition)
					end
				end
			end)
		end

		return Aimbot
	end

	function MainTabModule.CreateMeleeAura()
		local MeleeAura = {
			enabled = false,
			settings = {
				showAnim = false,
				targetPart = "Head",
				checkDowned = true,
				teamCheck = false,
				checkWhitelist = false,
				checkTarget = false,
				distance = 10,
				targetChangeTime = 0.5,
			},
			randomPart = "Head",
		}

		local ValidParts = { "Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg" }

		local function IsDowned(player)
			local character = player.Character
			local humanoid = character and character:FindFirstChildOfClass("Humanoid")
			if not character or not humanoid then
				return true
			end
			if humanoid.Health <= 15 then
				return true
			end

			local stats = character:FindFirstChild("CharStats")
			if not stats then
				for _ = 1, 5 do
					task.wait(0.1)
					stats = character:FindFirstChild("CharStats")
					if stats then
						break
					end
				end
			end
			local downed = stats and stats:FindFirstChild("Downed")
			if downed and downed:IsA("BoolValue") then
				return downed.Value == true
			end

			return false
		end

		function MeleeAura:StartRandomizer()
			task.spawn(function()
				while true do
					if self.enabled and self.settings.targetPart == "Random" then
						self.randomPart = ValidParts[math.random(1, #ValidParts)]
						task.wait(self.settings.targetChangeTime)
					else
						task.wait(0.5)
					end
				end
			end)
		end

		function MeleeAura:StartMainLoop(IsPlayerWhitelisted, IsPlayerTargeted)
			task.spawn(function()
				local remote1, remote2

				pcall(function()
					remote1 = ReplicatedStorage:WaitForChild("Events", 5):WaitForChild("XMHH.2", 5)
					remote2 = ReplicatedStorage:WaitForChild("Events", 5):WaitForChild("XMHH2.2", 5)
				end)

				if not remote1 or not remote2 then
					warn("Melee Aura: Required remotes not found")
					return
				end

				local attackTick = tick()
				local attackCooldown = 0.1

				local AttackCooldowns = {
					["Fists"] = 0.05,
					["Knuckledusters"] = 0.05,
					["Nunchucks"] = 0.05,
					["Shiv"] = 0.05,
					["Bat"] = 1,
					["Metal-Bat"] = 1,
					["Chainsaw"] = 2.5,
					["Balisong"] = 0.05,
					["Rambo"] = 0.3,
					["Shovel"] = 3,
					["Sledgehammer"] = 2,
					["Katana"] = 0.1,
					["Wrench"] = 0.1,
					["Fire Axe"] = 2,
				}

				local function GetTool()
					local character = LocalPlayer.Character
					return character and character:FindFirstChildOfClass("Tool")
				end

				local function GetMyHRP()
					local character = LocalPlayer.Character
					return character and character:FindFirstChild("HumanoidRootPart")
				end

				local function Attack(target)
					if not target then
						return
					end

					local character = LocalPlayer.Character
					local tool = GetTool()
					if not tool then
						return
					end

					local animationFolder = tool:FindFirstChild("AnimsFolder")
					local slashAnimation = animationFolder and animationFolder:FindFirstChild("Slash1")

					if tick() - attackTick >= attackCooldown then
						local success, result = pcall(function()
							return remote1:InvokeServer("🍞", tick(), tool, "43TRFWX", "Normal", tick(), true)
						end)

						if not success then
							return
						end

						attackCooldown = AttackCooldowns[tool.Name] or 0.5

						if self.settings.showAnim then
							local humanoid = character and character:FindFirstChildOfClass("Humanoid")
							if humanoid and slashAnimation then
								local animator = humanoid:FindFirstChild("Animator")
								if animator then
									local animationTrack = animator:LoadAnimation(slashAnimation)
									animationTrack:Play()
									animationTrack:AdjustSpeed(1.3)
								end
							end
						end

						task.wait(0.3)

						local handle = tool:FindFirstChild("WeaponHandle") or tool:FindFirstChild("Handle")
						if not handle and character then
							handle = character:FindFirstChild("Right Arm")
						end

						local hitPartName = self.settings.targetPart == "Random" and self.randomPart
							or self.settings.targetPart
						local targetPart = target:FindFirstChild(hitPartName)
						local myHRP = GetMyHRP()

						if not handle or not targetPart or not myHRP then
							return
						end

						local arguments = {
							"🍞",
							tick(),
							tool,
							"2389ZFX34",
							result,
							true,
							handle,
							targetPart,
							target,
							myHRP.Position,
							targetPart.Position,
						}

						pcall(function()
							if tool.Name == "Chainsaw" then
								for i = 1, 15 do
									remote2:FireServer(unpack(arguments))
								end
							else
								remote2:FireServer(unpack(arguments))
							end
						end)

						attackTick = tick()
					end
				end

				LocalPlayer.CharacterAdded:Connect(function(character)
					repeat
						task.wait()
					until character:FindFirstChild("HumanoidRootPart")
				end)

				while true do
					if self.enabled then
						local myHRP = GetMyHRP()
						if myHRP then
							for _, player in pairs(Players:GetPlayers()) do
								if player ~= LocalPlayer then
									local character = player.Character
									local humanoidRootPart = character and character:FindFirstChild("HumanoidRootPart")
									local humanoid = character and character:FindFirstChildOfClass("Humanoid")

									if humanoidRootPart and humanoid then
										local distance = (myHRP.Position - humanoidRootPart.Position).Magnitude
										if distance <= self.settings.distance then
											if self.settings.teamCheck and player.Team == LocalPlayer.Team then
												continue
											end
											if self.settings.checkDowned and IsDowned(player) then
												continue
											end
											if self.settings.checkWhitelist and IsPlayerWhitelisted(player) then
												continue
											end
											if self.settings.checkTarget and not IsPlayerTargeted(player) then
												continue
											end

											Attack(character)
										end
									end
								end
							end
						end
					end
					task.wait(0.1)
				end
			end)
		end

		return MeleeAura
	end

	function MainTabModule.CreateMeleeReach()
		local MeleeReach = {
			enabled = false,
			loopRunning = false,
			loopTask = nil,
			settings = {
				checkDowned = true,
				checkWhitelist = false,
				checkTarget = false,
				distance = 10,
				teamCheck = false,
			},
		}

		local function isDowned(player)
			local character = player and player.Character
			local humanoid = character and character:FindFirstChildOfClass("Humanoid")

			if not character or not humanoid then
				return true
			end

			if humanoid.Health <= 15 then
				return true
			end

			local stats = character:FindFirstChild("CharStats")
			local downed = stats and stats:FindFirstChild("Downed")

			return downed and downed:IsA("BoolValue") and downed.Value == true or false
		end

		local function isEligible(player, isPlayerWhitelisted, isPlayerTargeted)
			if not player or player == LocalPlayer then
				return false
			end

			local character = player.Character
			local humanoid = character and character:FindFirstChildOfClass("Humanoid")
			local root = character and character:FindFirstChild("HumanoidRootPart")

			if not character or not humanoid or not root or humanoid.Health <= 0 then
				return false
			end

			if character:FindFirstChildOfClass("ForceField") then
				return false
			end

			if MeleeReach.settings.teamCheck and player.Team == LocalPlayer.Team then
				return false
			end

			if MeleeReach.settings.checkDowned and isDowned(player) then
				return false
			end

			if MeleeReach.settings.checkWhitelist and isPlayerWhitelisted and isPlayerWhitelisted(player) then
				return false
			end

			if MeleeReach.settings.checkTarget and isPlayerTargeted and not isPlayerTargeted(player) then
				return false
			end

			return true
		end

		local function getEquippedTool(character)
			if not character then
				return nil
			end

			for _, child in pairs(character:GetChildren()) do
				if child:IsA("Tool") then
					return child
				end
			end

			return nil
		end

		local function randomAxisOffset(size)
			local extent = math.max(0, math.floor(math.abs(size) * 10))

			if extent == 0 then
				return 0
			end

			return math.random(-extent, extent) / 10
		end

		local function randomPointInPart(part)
			local size = part.Size

			return part.Position
				+ Vector3.new(randomAxisOffset(size.X), randomAxisOffset(size.Y), randomAxisOffset(size.Z))
		end

		local function collectReachObjects(tool, character)
			local objects = {}

			if not tool then
				return objects
			end

			local weaponHandle = tool:FindFirstChild("WeaponHandle")

			if weaponHandle then
				for _, child in pairs(weaponHandle:GetChildren()) do
					objects[#objects + 1] = child
				end
			end

			if tool.Name == "Fists" and character then
				local leftArm = character:FindFirstChild("Left Arm")
				local rightArm = character:FindFirstChild("Right Arm")

				if leftArm then
					objects[#objects + 1] = leftArm
				end

				if rightArm then
					objects[#objects + 1] = rightArm
				end
			end

			if tool.Name == "Sledgehammer" and weaponHandle then
				objects[#objects + 1] = weaponHandle
			end

			return objects
		end

		local function placeReachObject(object, targetRoot)
			if not object or not targetRoot then
				return
			end

			local targetFrame = CFrame.new(randomPointInPart(targetRoot))

			if object:IsA("BasePart") then
				pcall(function()
					object.CFrame = targetFrame
				end)
			else
				pcall(function()
					object.WorldCFrame = targetFrame * CFrame.new(0, 0.5, 0)
				end)

				pcall(function()
					object.CFrame = CFrame.new(0, 0.4, 0)
				end)
			end
		end

		local function processTarget(tool, localCharacter, player)
			local targetCharacter = player.Character
			local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")
			local localRoot = localCharacter and localCharacter:FindFirstChild("HumanoidRootPart")

			if not targetRoot or not localRoot then
				return
			end

			if (targetRoot.Position - localRoot.Position).Magnitude > MeleeReach.settings.distance then
				return
			end

			for _, object in pairs(collectReachObjects(tool, localCharacter)) do
				placeReachObject(object, targetRoot)
			end
		end

		function MeleeReach:StartMainLoop(isPlayerWhitelisted, isPlayerTargeted)
			if self.loopRunning then
				return self.loopTask
			end

			self.loopRunning = true
			self.loopTask = task.spawn(function()
				while self.loopRunning do
					if self.enabled then
						local character = LocalPlayer.Character
						local tool = getEquippedTool(character)

						if character and tool then
							for _, player in pairs(Players:GetPlayers()) do
								if isEligible(player, isPlayerWhitelisted, isPlayerTargeted) then
									processTarget(tool, character, player)
								end
							end
						end
					end

					task.wait(0.05)
				end

				self.loopTask = nil
			end)

			return self.loopTask
		end

		return MeleeReach
	end

	CriminalityV1.Main = MainTabModule
end

local VisualModule

do
	local Players = game:GetService("Players")
	local RunService = game:GetService("RunService")
	local UserInputService = game:GetService("UserInputService")
	local Workspace = game:GetService("Workspace")

	local LocalPlayer = Players.LocalPlayer
	local Camera = Workspace.CurrentCamera
	local ESP = {
		Enabled = {
			Box = false,
			HealthBar = false,
			Health = false,
			Name = false,
			Distance = false,
			Tool = false,
			Tracer = false,
			Highlight = false,
		},
		Settings = {
			BoxColor = Color3.fromRGB(255, 255, 255),
			HighlightColor = Color3.fromRGB(255, 255, 255),
			TeamCheck = false,
			TracerPosition = "Bottom",
		},
		Data = {
			Boxes = {},
			HealthBars = {},
			HealthLabels = {},
			NameLabels = {},
			DistanceLabels = {},
			ToolLabels = {},
			Tracers = {},
			Highlights = {},
		},
	}

	local ArmsChams

	local PlayerDetect = {
		Enabled = false,
		Distance = 100,
		Messages = {},
	}

	local function createText(color)
		local text = Drawing.new("Text")
		text.Size = 14
		text.Center = true
		text.Outline = true
		text.Visible = false
		text.Color = color
		text.Font = 2
		return text
	end

	local function getBoxColor(player)
		if ESP.Settings.TeamCheck then
			return player.TeamColor.Color
		end

		return ESP.Settings.BoxColor
	end

	local function getHighlightColor(player)
		if ESP.Settings.TeamCheck then
			return player.TeamColor.Color
		end

		return ESP.Settings.HighlightColor
	end

	local function hideDrawings(player)
		local box = ESP.Data.Boxes[player]
		if box then
			box.Visible = false
		end

		local nameLabel = ESP.Data.NameLabels[player]
		if nameLabel then
			nameLabel.Visible = false
		end

		local healthLabel = ESP.Data.HealthLabels[player]
		if healthLabel then
			healthLabel.Visible = false
		end

		local healthBar = ESP.Data.HealthBars[player]
		if healthBar then
			healthBar.Visible = false
		end

		local toolLabel = ESP.Data.ToolLabels[player]
		if toolLabel then
			toolLabel.Visible = false
		end

		local distanceLabel = ESP.Data.DistanceLabels[player]
		if distanceLabel then
			distanceLabel.Visible = false
		end

		local tracer = ESP.Data.Tracers[player]
		if tracer then
			tracer.Visible = false
		end
	end

	local function removePlayerVisuals(player)
		local box = ESP.Data.Boxes[player]
		if box then
			box:Remove()
			ESP.Data.Boxes[player] = nil
		end

		local highlight = ESP.Data.Highlights[player]
		if highlight then
			highlight:Destroy()
			ESP.Data.Highlights[player] = nil
		end

		local nameLabel = ESP.Data.NameLabels[player]
		if nameLabel then
			nameLabel:Remove()
			ESP.Data.NameLabels[player] = nil
		end

		local healthLabel = ESP.Data.HealthLabels[player]
		if healthLabel then
			healthLabel:Remove()
			ESP.Data.HealthLabels[player] = nil
		end

		local healthBar = ESP.Data.HealthBars[player]
		if healthBar then
			healthBar:Remove()
			ESP.Data.HealthBars[player] = nil
		end

		local toolLabel = ESP.Data.ToolLabels[player]
		if toolLabel then
			toolLabel:Remove()
			ESP.Data.ToolLabels[player] = nil
		end

		local distanceLabel = ESP.Data.DistanceLabels[player]
		if distanceLabel then
			distanceLabel:Remove()
			ESP.Data.DistanceLabels[player] = nil
		end

		local tracer = ESP.Data.Tracers[player]
		if tracer then
			tracer:Remove()
			ESP.Data.Tracers[player] = nil
		end
	end

	local function updateBox(player, head, root)
		local box = ESP.Data.Boxes[player]

		if not ESP.Enabled.Box then
			if box then
				box.Visible = false
			end
			return
		end

		local color = getBoxColor(player)

		if not box then
			box = Drawing.new("Square")
			box.Thickness = 1
			box.Filled = false
			box.Visible = false
			box.Color = color
			ESP.Data.Boxes[player] = box
		end

		local top = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
		local bottom = Camera:WorldToViewportPoint(root.Position - Vector3.new(0, 3, 0))
		local height = bottom.Y - top.Y
		local width = height / 2

		box.Size = Vector2.new(width, height)
		box.Position = Vector2.new(top.X - width / 2, top.Y)
		box.Color = color
		box.Visible = true
	end

	local function ensureHighlight(player, character)
		local highlight = ESP.Data.Highlights[player]
		local color = getHighlightColor(player)

		if not highlight then
			highlight = Instance.new("Highlight")
			highlight.Name = "JX_Highlight"
			highlight.Adornee = character
			highlight.FillColor = color
			highlight.OutlineColor = color
			highlight.FillTransparency = 0.5
			highlight.OutlineTransparency = 0
			highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
			highlight.Parent = game.CoreGui
			ESP.Data.Highlights[player] = highlight
		else
			highlight.Adornee = character
			highlight.Enabled = true
		end

		return highlight
	end

	local function updateHighlight(player, character)
		local highlight = ESP.Data.Highlights[player]

		if not ESP.Enabled.Highlight then
			if highlight then
				highlight.Enabled = false
			end
			return
		end

		highlight = ensureHighlight(player, character)

		local color = getHighlightColor(player)
		highlight.FillColor = color
		highlight.OutlineColor = color
	end

	local function updateName(player, screenPosition)
		local label = ESP.Data.NameLabels[player]

		if not ESP.Enabled.Name then
			if label then
				label.Visible = false
			end
			return
		end

		if not label then
			label = Drawing.new("Text")
			label.Size = 14
			label.Center = true
			label.Outline = true
			label.Visible = false
			label.Color = Color3.new(1, 1, 1)
			label.Font = 2
			ESP.Data.NameLabels[player] = label
		end

		label.Text = player.Name
		label.Position = Vector2.new(screenPosition.X, screenPosition.Y - 35)
		label.Color = getBoxColor(player)
		label.Visible = true
	end

	local function updateHealth(player, screenPosition, humanoid)
		local label = ESP.Data.HealthLabels[player]

		if not ESP.Enabled.Health then
			if label then
				label.Visible = false
			end
			return
		end

		local color = Color3.new(0, 1, 0)

		if not label then
			label = createText(color)
			ESP.Data.HealthLabels[player] = label
		end

		label.Text = "HP: " .. tostring(math.floor(humanoid.Health))
		label.Position = Vector2.new(screenPosition.X, screenPosition.Y + 35)
		label.Visible = true
	end

	local function updateHealthBar(player, screenPosition, humanoid)
		local bar = ESP.Data.HealthBars[player]

		if not ESP.Enabled.HealthBar then
			if bar then
				bar.Visible = false
			end
			return
		end

		if not bar then
			bar = Drawing.new("Line")
			bar.Thickness = 2
			bar.Visible = false
			ESP.Data.HealthBars[player] = bar
		end

		local ratio = humanoid.Health / humanoid.MaxHealth
		local bottomY = screenPosition.Y + 30.0

		bar.From = Vector2.new(screenPosition.X - 30, bottomY - 60 * ratio)
		bar.To = Vector2.new(screenPosition.X - 30, bottomY)
		bar.Color = Color3.fromRGB(255 * (1 - ratio), 255 * ratio, 0)
		bar.Visible = true
	end

	local function updateTool(player, character, screenPosition)
		if not ESP.Enabled.Tool then
			return
		end

		local label = ESP.Data.ToolLabels[player]
		local tool = character:FindFirstChildOfClass("Tool")

		if not tool then
			if label then
				label.Visible = false
			end
			return
		end

		local color = Color3.new(1, 1, 0)

		if not label then
			label = createText(color)
			ESP.Data.ToolLabels[player] = label
		end

		label.Text = "[" .. tool.Name .. "]"
		label.Position = Vector2.new(screenPosition.X, screenPosition.Y + 55)
		label.Visible = true
	end

	local function updateDistance(player, screenPosition, root)
		local label = ESP.Data.DistanceLabels[player]

		if not ESP.Enabled.Distance then
			if label then
				label.Visible = false
			end
			return
		end

		local color = Color3.fromRGB(0, 255, 255)

		if not label then
			label = createText(color)
			ESP.Data.DistanceLabels[player] = label
		end

		local distance = math.floor((root.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude)

		label.Text = "[" .. tostring(distance) .. "m]"
		label.Position = Vector2.new(screenPosition.X, screenPosition.Y + 70)
		label.Visible = true
	end

	local function getTracerOrigin()
		if ESP.Settings.TracerPosition == "Top" then
			return Vector2.new(Camera.ViewportSize.X / 2, 0)
		end

		if ESP.Settings.TracerPosition == "Mouse" then
			return UserInputService:GetMouseLocation()
		end

		if ESP.Settings.TracerPosition == "Middle" then
			return Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
		end

		return Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
	end

	local function updateTracer(player, screenPosition)
		local tracer = ESP.Data.Tracers[player]

		if not ESP.Enabled.Tracer then
			if tracer then
				tracer.Visible = false
			end
			return
		end

		local color = getBoxColor(player)

		if not tracer then
			tracer = Drawing.new("Line")
			tracer.Thickness = 1.5
			tracer.Color = color
			tracer.Visible = false
			ESP.Data.Tracers[player] = tracer
		end

		tracer.From = getTracerOrigin()
		tracer.To = Vector2.new(screenPosition.X, screenPosition.Y)
		tracer.Color = color
		tracer.Visible = true
	end

	local function updatePlayer(player)
		if player == LocalPlayer then
			return
		end

		local character = player.Character

		if not character then
			removePlayerVisuals(player)
			return
		end

		local head = character:FindFirstChild("Head")

		if not head then
			removePlayerVisuals(player)
			return
		end

		local root = character:FindFirstChild("HumanoidRootPart")
		local humanoid = character:FindFirstChild("Humanoid")

		if not root or not humanoid then
			return
		end

		local screenPosition, visible = Camera:WorldToViewportPoint(root.Position)

		if not visible then
			hideDrawings(player)
			return
		end

		updateBox(player, head, root)
		updateHighlight(player, character)
		updateName(player, screenPosition)
		updateHealth(player, screenPosition, humanoid)
		updateHealthBar(player, screenPosition, humanoid)
		updateTool(player, character, screenPosition)
		updateDistance(player, screenPosition, root)
		updateTracer(player, screenPosition)
	end

	local function ApplyArmsChams()
		local viewModel = Camera:FindFirstChild("ViewModel")

		if not viewModel then
			return
		end

		local leftArm = viewModel:FindFirstChild("Left Arm")
		local rightArm = viewModel:FindFirstChild("Right Arm")
		local material = ArmsChams.Enabled and Enum.Material.ForceField or Enum.Material.Plastic

		if leftArm then
			leftArm.Material = material
			leftArm.Color = ArmsChams.Color
		end

		if rightArm then
			rightArm.Material = material
			rightArm.Color = ArmsChams.Color
		end
	end

	local function enableDetectHighlight(character)
		local highlight = character:FindFirstChild("DetectHighlight")

		if not highlight then
			highlight = Instance.new("Highlight")
			highlight.Name = "DetectHighlight"
			highlight.FillColor = Color3.fromRGB(255, 0, 0)
			highlight.OutlineColor = Color3.fromRGB(255, 255, 0)
			highlight.FillTransparency = 0.3
			highlight.OutlineTransparency = 0
			highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
			highlight.Adornee = character
			highlight.Parent = character
		end

		character:FindFirstChild("DetectHighlight").Enabled = true
	end

	local function createDetectMessage(playerName)
		local label = Instance.new("TextLabel")
		label.Name = playerName
		label.Size = UDim2.new(0, 250, 0, 30)
		label.AnchorPoint = Vector2.new(1, 0)
		label.Position = UDim2.new(1, -20, 0, 100)
		label.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
		label.BackgroundTransparency = 0.3
		label.BorderSizePixel = 0
		label.TextColor3 = Color3.fromRGB(255, 0, 0)
		label.TextSize = 16
		label.Font = Enum.Font.GothamBold
		label.Text = ""
		label.Visible = false
		label.TextXAlignment = Enum.TextXAlignment.Right
		label.Parent = PlayerDetect.ScreenGui

		local stroke = Instance.new("UIStroke")
		stroke.Thickness = 1.2
		stroke.Color = Color3.fromRGB(0, 0, 0)
		stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual
		stroke.Parent = label

		return label
	end

	local function repositionDetectMessages()
		local y = 100

		for _, label in pairs(PlayerDetect.Messages) do
			label.Position = UDim2.new(1, -20, 0, y)
			y = y + 35
		end
	end

	local function updatePlayerDetect()
		local character = LocalPlayer.Character
		local root = character and character:FindFirstChild("HumanoidRootPart")

		if not root then
			return
		end

		local active = {}

		for _, player in ipairs(Players:GetPlayers()) do
			if player ~= LocalPlayer then
				local targetCharacter = player.Character
				local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")

				if targetCharacter and targetRoot then
					local distance = math.floor((targetRoot.Position - root.Position).Magnitude)

					if distance <= PlayerDetect.Distance then
						local playerName = player.Name
						active[playerName] = true
						enableDetectHighlight(targetCharacter)

						local label = PlayerDetect.Messages[playerName]

						if not label then
							label = createDetectMessage(playerName)
							PlayerDetect.Messages[playerName] = label
							repositionDetectMessages()
						end

						label.Text = "👤 " .. playerName .. " | " .. tostring(distance) .. " studs"
						label.Visible = true
					end
				end
			end
		end

		for playerName, label in pairs(PlayerDetect.Messages) do
			if not active[playerName] then
				local player = Players:FindFirstChild(playerName)
				local characterToClear = player and player.Character
				local highlight = characterToClear and characterToClear:FindFirstChild("DetectHighlight")

				if highlight then
					highlight:Destroy()
				end

				label:Destroy()
				PlayerDetect.Messages[playerName] = nil
			end
		end
	end

	local function EnablePlayerDetect()
		if PlayerDetect.Enabled then
			return
		end

		PlayerDetect.Enabled = true
		PlayerDetect.ScreenGui = Instance.new("ScreenGui", LocalPlayer:WaitForChild("PlayerGui"))
		PlayerDetect.ScreenGui.Name = "NearbyPlayerUI"
		PlayerDetect.ScreenGui.ResetOnSpawn = false
		PlayerDetect.Connection = RunService.Heartbeat:Connect(updatePlayerDetect)
	end

	local function DisablePlayerDetect()
		if not PlayerDetect.Enabled then
			return
		end

		PlayerDetect.Enabled = false

		if PlayerDetect.Connection then
			PlayerDetect.Connection:Disconnect()
			PlayerDetect.Connection = nil
		end

		for _, label in pairs(PlayerDetect.Messages) do
			label:Destroy()
		end

		PlayerDetect.Messages = {}

		for _, player in ipairs(Players:GetPlayers()) do
			local character = player.Character
			local highlight = character and character:FindFirstChild("DetectHighlight")

			if highlight then
				highlight:Destroy()
			end
		end

		if PlayerDetect.ScreenGui then
			PlayerDetect.ScreenGui:Destroy()
			PlayerDetect.ScreenGui = nil
		end
	end

	local function connectCharacter(player)
		player.CharacterAdded:Connect(function(character)
			task.wait(0.5)

			if ESP.Enabled.Highlight then
				ensureHighlight(player, character)
			end
		end)
	end

	local ESPLoop = RunService.RenderStepped:Connect(function()
		for _, player in ipairs(Players:GetPlayers()) do
			updatePlayer(player)
		end
	end)

	Players.PlayerRemoving:Connect(removePlayerVisuals)

	Players.PlayerAdded:Connect(function(player)
		connectCharacter(player)
	end)

	for _, player in ipairs(Players:GetPlayers()) do
		if player ~= LocalPlayer then
			connectCharacter(player)
		end
	end

	local ArmsChams = {
		Enabled = false,
		Color = Color3.fromRGB(255, 255, 255),
	}

	LocalPlayer.CharacterAdded:Connect(function()
		if Camera:FindFirstChild("ViewModel") then
			task.wait()
			ApplyArmsChams()
		end
	end)

	VisualModule = {
		ESP = ESP,
		ArmsChams = ArmsChams,
		PlayerDetect = PlayerDetect,
		ESPLoop = ESPLoop,
		ApplyArmsChams = ApplyArmsChams,
		EnablePlayerDetect = EnablePlayerDetect,
		DisablePlayerDetect = DisablePlayerDetect,
	}
end

local VerifiedCallbacks = {}

function VerifiedCallbacks.MeleeAuraChainsawBurst(tool, firstAction, secondAction, ...)
	if not tool or tool.Name ~= "Chainsaw" then
		return
	end

	for _ = 1, 15 do
		pcall(firstAction, ...)
		pcall(secondAction, ...)
	end
end

function VerifiedCallbacks.RageBotHitNotification(notifier, player, health)
	local playerName = player and player.Name or "Unknown"
	local remainingHealth = math.floor(tonumber(health) or 0)
	local text = "Hit " .. playerName .. " | Health: " .. remainingHealth

	return notifier:Notify(text)
end

function VerifiedCallbacks.RageBotTracerFade(tracer, step, delay)
	local alpha = 0

	while tracer and alpha < 1 do
		alpha = math.min(alpha + step, 1)
		tracer.Transparency = alpha
		task.wait(delay)
	end
end

function VerifiedCallbacks.RageBotResolveAndFireRemote(replicatedStorage, ...)
	local events = replicatedStorage:FindFirstChild("Events")
	local remote = events and events:FindFirstChild("ZFKLF__H")

	if remote then
		return remote:FireServer(...)
	end
end

function VerifiedCallbacks.RageBotForwardEvent(event, value)
	return event:Fire(value)
end

function VerifiedCallbacks.SilentAimV2ForwardEvent(event, value)
	return event:Fire(value)
end

function VerifiedCallbacks.SilentAimV2HideNotification(notification)
	notification.Visible = false
end

function VerifiedCallbacks.SilentAimV1TargetComparator(left, right, key)
	return left[key] < right[key]
end

local RootModule = {}

function RootModule.new(library)
	local Players = game:GetService("Players")
	local RunService = game:GetService("RunService")
	local UserInputService = game:GetService("UserInputService")
	local ReplicatedStorage = game:GetService("ReplicatedStorage")
	local Workspace = game:GetService("Workspace")
	local Lighting = game:GetService("Lighting")
	local VirtualUser = game:GetService("VirtualUser")
	local TextChatService = game:GetService("TextChatService")
	local LocalPlayer = Players.LocalPlayer
	local Camera = Workspace.CurrentCamera
	local Main = CriminalityV1.Main
	local Visual = VisualModule
	local callbacks = {}
	local state = {
		Whitelist = {},
		Targets = {},
		HighlightWhitelist = false,
		HighlightTarget = false,
		WhitelistColor = Color3.fromRGB(0, 255, 0),
		TargetColor = Color3.fromRGB(255, 0, 0),
		FastPickup = false,
		InfiniteStamina = false,
		NoFailLockpick = false,
		Noclip = false,
		CtrlClickTP = false,
		QTeleport = false,
		StompSpeed = 1,
		AntiSmoke = false,
		NoFall = false,
		AntiFractured = false,
		NoBarriers = false,
		NoNeck = false,
		AutoBreakSafeRegister = false,
		AutoLockpick = false,
		AutoPickUpTool = false,
		AutoPickUpCash = false,
		AutoRefill = false,
		AutoDeposit = false,
		AutoUnlockDoor = false,
		AutoCloseDoors = false,
		AutoOpenDoors = false,
		AutoRespawn = false,
		AutoFarmAllowance = false,
		SpeedEnabled = false,
		Speed = 16,
		JumpEnabled = false,
		JumpPower = 50,
		FlyEnabled = false,
		FlySpeed = 50,
		FlyMethod = "Velocity",
		WallBangEnabled = false,
		InstantEquip = false,
		InstantReload = false,
		CustomRecoilEnabled = false,
		RecoilScale = 0,
		InfiniteSpray = false,
		PepperAuraEnabled = false,
		PepperAuraRange = 20,
		C4Enabled = false,
		C4Speed = 50,
		ExplosionAmmoEnabled = false,
		ExplosionAmmoSpeed = 50,
		ScrapESP = false,
		ScrapDistance = 1000,
		ScrapTypes = {},
		CashDropESP = false,
		ToolsESP = false,
		SafeESP = false,
		ATMESP = false,
		DealerESP = false,
		HideLevelUI = false,
		VisualLevel = "",
		CustomRegion = "",
		CustomName = "",
		Fullbright = false,
		AdminCheck = false,
		UseCustomFOV = false,
		FOV = 70,
		CameraDistance = 15,
		Hug = false,
		Jerk = false,
		Carpet = false,
		FakeDowned = false,
		HideHead = false,
		HideBody = false,
		KeyESP = false,
		HighlightExit = false,
		DealerStockChecker = false,
		SelectedItems = {},
		NotifyNewStock = false,
		ESPStockDealer = false,
		SelectedStockDealer = nil,
		SelectedSkins = {},
		AutoSkins = {},
		Connections = {},
		Objects = {},
		Loops = {},
		Highlights = {},
		FullbrightBackup = {},
	}

	local instances = {
		SilentAimV2 = Main.CreateSilentAimV2(),
		SilentAimV1 = Main.CreateSilentAimV1(),
		RageBot = Main.CreateRageBot(),
		Aimbot = Main.CreateAimbot(),
		MeleeAura = Main.CreateMeleeAura(),
		MeleeReach = Main.CreateMeleeReach(),
	}

	local function notify(title, description, duration)
		if library and type(library.Notify) == "function" then
			pcall(function()
				library:Notify({
					Title = title,
					Description = description,
					Time = duration or 3,
				})
			end)
		end
	end

	local function disconnect(name)
		local connection = state.Connections[name]
		if connection then
			pcall(function()
				connection:Disconnect()
			end)
			state.Connections[name] = nil
		end
	end

	local function destroy(name)
		local object = state.Objects[name]
		if object then
			pcall(function()
				object:Destroy()
			end)
			state.Objects[name] = nil
		end
	end

	local function startLoop(name, interval, predicate, action)
		state.Loops[name] = true
		task.spawn(function()
			while state.Loops[name] do
				if not predicate or predicate() then
					pcall(action)
				end
				task.wait(interval)
			end
		end)
	end

	local function stopLoop(name)
		state.Loops[name] = nil
	end

	local function getCharacter()
		return LocalPlayer.Character
	end

	local function getRoot(character)
		character = character or getCharacter()
		return character and (character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso"))
	end

	local function getHumanoid(character)
		character = character or getCharacter()
		return character and character:FindFirstChildOfClass("Humanoid")
	end

	local function getTool(character)
		character = character or getCharacter()
		return character and character:FindFirstChildOfClass("Tool")
	end

	local function getEvents()
		return ReplicatedStorage:FindFirstChild("Events")
	end

	local function getEvent(name)
		local events = getEvents()
		return events and events:FindFirstChild(name)
	end

	local function normalizeSelection(value)
		local result = {}
		if type(value) == "table" then
			for key, selected in pairs(value) do
				if type(key) == "string" and selected then
					result[key] = true
				elseif type(selected) == "string" then
					result[selected] = true
				elseif typeof(selected) == "Instance" and selected:IsA("Player") then
					result[selected.Name] = true
				end
			end
		elseif type(value) == "string" then
			result[value] = true
		elseif typeof(value) == "Instance" and value:IsA("Player") then
			result[value.Name] = true
		end
		return result
	end

	local function isWhitelisted(player)
		return player and state.Whitelist[player.Name] == true
	end

	local function isTargeted(player)
		return player and state.Targets[player.Name] == true
	end

	local function updatePlayerHighlight(player)
		if not player or player == LocalPlayer then
			return
		end
		local character = player.Character
		if not character then
			return
		end
		local name = "JXSelectionHighlight"
		local highlight = character:FindFirstChild(name)
		local color
		if state.HighlightTarget and isTargeted(player) then
			color = state.TargetColor
		elseif state.HighlightWhitelist and isWhitelisted(player) then
			color = state.WhitelistColor
		end
		if color then
			if not highlight then
				highlight = Instance.new("Highlight")
				highlight.Name = name
				highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
				highlight.FillTransparency = 0.5
				highlight.OutlineTransparency = 0
				highlight.Parent = character
			end
			highlight.FillColor = color
			highlight.OutlineColor = color
		elseif highlight then
			highlight:Destroy()
		end
	end

	local function updateAllPlayerHighlights()
		for _, player in ipairs(Players:GetPlayers()) do
			updatePlayerHighlight(player)
		end
	end

	local function playerNames()
		local names = {}
		for _, player in ipairs(Players:GetPlayers()) do
			if player ~= LocalPlayer then
				names[#names + 1] = player.Name
			end
		end
		table.sort(names)
		return names
	end

	local function ensureForceField(character)
		if not state.NoFall or not character then
			return
		end
		local forceField = character:FindFirstChild("JXNoFall")
		if not forceField then
			forceField = Instance.new("ForceField")
			forceField.Name = "JXNoFall"
			forceField.Visible = false
			forceField.Parent = character
		end
	end

	local function removeForceField(character)
		character = character or getCharacter()
		local forceField = character and character:FindFirstChild("JXNoFall")
		if forceField then
			forceField:Destroy()
		end
	end

	local function setBarrierCollision(enabled)
		local filter = Workspace:FindFirstChild("Filter")
		local parts = filter and filter:FindFirstChild("Parts")
		local barrierFolder = parts and parts:FindFirstChild("F_Parts")
		if not barrierFolder then
			return
		end
		for _, object in ipairs(barrierFolder:GetDescendants()) do
			if object:IsA("BasePart") then
				object.CanCollide = enabled
			end
		end
	end

	local function removeSmoke(object)
		if not object then
			return
		end
		if object:IsA("Smoke") or object:IsA("ParticleEmitter") or object.Name:lower():find("smoke", 1, true) then
			object:Destroy()
		end
	end

	local function repairFractures(character)
		if not character then
			return
		end
		for _, object in ipairs(character:GetDescendants()) do
			local lower = object.Name:lower()
			if lower:find("fracture", 1, true) or lower:find("broken", 1, true) then
				if object:IsA("BoolValue") then
					object.Value = false
				else
					pcall(function()
						object:Destroy()
					end)
				end
			end
		end
	end

	local function nearestPrompt(root, predicate)
		local best
		local bestDistance = math.huge
		for _, object in ipairs(Workspace:GetDescendants()) do
			if object:IsA("ProximityPrompt") and (not predicate or predicate(object)) then
				local parent = object.Parent
				local part = parent and (parent:IsA("BasePart") and parent or parent:FindFirstChildWhichIsA("BasePart"))
				if part and root then
					local distance = (root.Position - part.Position).Magnitude
					if distance < bestDistance then
						best = object
						bestDistance = distance
					end
				end
			end
		end
		return best, bestDistance
	end

	local function activatePrompt(prompt)
		if not prompt then
			return
		end
		if fireproximityprompt then
			pcall(fireproximityprompt, prompt)
		else
			pcall(function()
				prompt:InputHoldBegin()
				task.wait(prompt.HoldDuration or 0)
				prompt:InputHoldEnd()
			end)
		end
	end

	local function touchObject(root, object)
		if not root or not object then
			return
		end
		local part = object:IsA("BasePart") and object or object:FindFirstChildWhichIsA("BasePart")
		if not part then
			return
		end
		if firetouchinterest then
			pcall(firetouchinterest, root, part, 0)
			pcall(firetouchinterest, root, part, 1)
		else
			root.CFrame = part.CFrame
		end
	end

	local function findMapFolder(name)
		local map = Workspace:FindFirstChild("Map")
		return map and map:FindFirstChild(name)
	end

	local function processDoors(mode)
		local doors = findMapFolder("Doors")
		local root = getRoot()
		if not doors or not root then
			return
		end
		for _, door in ipairs(doors:GetChildren()) do
			local base = door:FindFirstChild("DoorBase") or door:FindFirstChildWhichIsA("BasePart")
			if base and (base.Position - root.Position).Magnitude <= 20 then
				local prompt = door:FindFirstChildWhichIsA("ProximityPrompt", true)
				local values = door:FindFirstChild("Values")
				local openValue = values and (values:FindFirstChild("Open") or values:FindFirstChild("Opened"))
				local lockedValue = values and (values:FindFirstChild("Locked") or values:FindFirstChild("Lock"))
				if mode == "unlock" then
					if not lockedValue or lockedValue.Value then
						activatePrompt(prompt)
					end
				elseif mode == "open" then
					if not openValue or not openValue.Value then
						activatePrompt(prompt)
					end
				elseif mode == "close" then
					if not openValue or openValue.Value then
						activatePrompt(prompt)
					end
				end
			end
		end
	end

	local function applyMovement()
		local humanoid = getHumanoid()
		if not humanoid then
			return
		end
		if state.SpeedEnabled then
			humanoid.WalkSpeed = state.Speed
		end
		if state.JumpEnabled then
			humanoid.UseJumpPower = true
			humanoid.JumpPower = state.JumpPower
		end
	end

	local function stopFly()
		disconnect("Fly")
		destroy("FlyVelocity")
		destroy("FlyGyro")
	end

	local function startFly()
		stopFly()
		local root = getRoot()
		if not root then
			return
		end
		local velocity = Instance.new("BodyVelocity")
		velocity.Name = "JXFlyVelocity"
		velocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
		velocity.Velocity = Vector3.zero
		velocity.Parent = root
		local gyro = Instance.new("BodyGyro")
		gyro.Name = "JXFlyGyro"
		gyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
		gyro.P = 90000
		gyro.CFrame = Camera.CFrame
		gyro.Parent = root
		state.Objects.FlyVelocity = velocity
		state.Objects.FlyGyro = gyro
		state.Connections.Fly = RunService.RenderStepped:Connect(function()
			if not state.FlyEnabled then
				return
			end
			local direction = Vector3.zero
			if UserInputService:IsKeyDown(Enum.KeyCode.W) then
				direction += Camera.CFrame.LookVector
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.S) then
				direction -= Camera.CFrame.LookVector
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.A) then
				direction -= Camera.CFrame.RightVector
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.D) then
				direction += Camera.CFrame.RightVector
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
				direction += Vector3.yAxis
			end
			if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
				direction -= Vector3.yAxis
			end
			velocity.Velocity = direction.Magnitude > 0 and direction.Unit * state.FlySpeed or Vector3.zero
			gyro.CFrame = Camera.CFrame
		end)
	end

	local function resolveSkinAsset(tool, selected)
		if type(selected) == "number" then
			return "rbxassetid://" .. tostring(selected)
		end
		if type(selected) == "string" then
			if selected:match("^rbxasset") then
				return selected
			end
			local number = selected:match("%d+")
			if number and #number >= 6 then
				return "rbxassetid://" .. number
			end
			local skins = tool and (tool:FindFirstChild("Skins") or tool:FindFirstChild("SkinAssets"))
			local skin = skins and skins:FindFirstChild(selected)
			if skin then
				if skin:IsA("StringValue") then
					return skin.Value
				end
				if skin:IsA("IntValue") or skin:IsA("NumberValue") then
					return "rbxassetid://" .. tostring(skin.Value)
				end
				local texture = skin:FindFirstChildWhichIsA("Texture") or skin:FindFirstChildWhichIsA("Decal")
				if texture then
					return texture.Texture
				end
			end
		end
		return nil
	end

	local function applySkinToTool(tool, weaponName)
		if not tool or tool.Name ~= weaponName then
			return
		end
		local selected = state.SelectedSkins[weaponName]
		if not selected then
			return
		end
		local asset = resolveSkinAsset(tool, selected)
		for _, object in ipairs(tool:GetDescendants()) do
			if object:IsA("MeshPart") and asset then
				object.TextureID = asset
			elseif object:IsA("SpecialMesh") and asset then
				object.TextureId = asset
			elseif (object:IsA("Texture") or object:IsA("Decal")) and asset then
				object.Texture = asset
			end
		end
	end

	local function applySkin(weaponName)
		local character = getCharacter()
		local backpack = LocalPlayer:FindFirstChild("Backpack")
		if character then
			for _, object in ipairs(character:GetChildren()) do
				if object:IsA("Tool") then
					applySkinToTool(object, weaponName)
				end
			end
		end
		if backpack then
			for _, object in ipairs(backpack:GetChildren()) do
				if object:IsA("Tool") then
					applySkinToTool(object, weaponName)
				end
			end
		end
	end

	local function setAutoSkin(weaponName, enabled)
		state.AutoSkins[weaponName] = enabled
		applySkin(weaponName)
		if not state.Connections.AutoSkinCharacter then
			state.Connections.AutoSkinCharacter = LocalPlayer.CharacterAdded:Connect(function(character)
				task.wait(0.6)
				for name, active in pairs(state.AutoSkins) do
					if active then
						applySkin(name)
					end
				end
				disconnect("AutoSkinCharacterChild")
				state.Connections.AutoSkinCharacterChild = character.ChildAdded:Connect(function(child)
					if child:IsA("Tool") and state.AutoSkins[child.Name] then
						task.wait()
						applySkinToTool(child, child.Name)
					end
				end)
			end)
		end
		local backpack = LocalPlayer:FindFirstChild("Backpack")
		if backpack and not state.Connections.AutoSkinBackpack then
			state.Connections.AutoSkinBackpack = backpack.ChildAdded:Connect(function(child)
				if child:IsA("Tool") and state.AutoSkins[child.Name] then
					task.wait()
					applySkinToTool(child, child.Name)
				end
			end)
		end
		local character = getCharacter()
		if character and not state.Connections.AutoSkinCharacterChild then
			state.Connections.AutoSkinCharacterChild = character.ChildAdded:Connect(function(child)
				if child:IsA("Tool") and state.AutoSkins[child.Name] then
					task.wait()
					applySkinToTool(child, child.Name)
				end
			end)
		end
	end

	local function removeNamedHighlights(prefix)
		for _, object in ipairs(Workspace:GetDescendants()) do
			if object:IsA("Highlight") and object.Name:sub(1, #prefix) == prefix then
				object:Destroy()
			end
		end
	end

	local function createHighlight(target, name, color)
		if not target then
			return nil
		end
		local highlight = target:FindFirstChild(name)
		if not highlight then
			highlight = Instance.new("Highlight")
			highlight.Name = name
			highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
			highlight.FillTransparency = 0.5
			highlight.OutlineTransparency = 0
			highlight.Parent = target
		end
		highlight.FillColor = color
		highlight.OutlineColor = color
		return highlight
	end

	local function scanESP(name, enabled, predicate, color)
		removeNamedHighlights(name)
		if not enabled then
			return
		end
		local root = getRoot()
		for _, object in ipairs(Workspace:GetDescendants()) do
			if predicate(object) then
				local part = object:IsA("BasePart") and object or object:FindFirstChildWhichIsA("BasePart")
				if part and (not root or (part.Position - root.Position).Magnitude <= state.ScrapDistance) then
					createHighlight(object, name, color)
				end
			end
		end
	end

	local function findCoreGui()
		local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
		return playerGui and playerGui:FindFirstChild("CoreGUI")
	end

	local function applyVisualLevel()
		local core = findCoreGui()
		if not core then
			return
		end
		local levelFrame = core:FindFirstChild("LevelFrame", true)
		if levelFrame then
			for _, object in ipairs(levelFrame:GetDescendants()) do
				if object:IsA("TextLabel") or object:IsA("TextButton") then
					object.Text = tostring(state.VisualLevel)
				end
			end
		end
	end

	local function applyRegion()
		local core = findCoreGui()
		local clientFrame = core and core:FindFirstChild("ClientFrame", true)
		local region = clientFrame and clientFrame:FindFirstChild("Region", true)
		if region and (region:IsA("TextLabel") or region:IsA("TextButton")) then
			region.Text = tostring(state.CustomRegion)
		end
	end

	local function applyName()
		local core = findCoreGui()
		local label = core and core:FindFirstChild("DisplayNameLabel", true)
		if label and (label:IsA("TextLabel") or label:IsA("TextButton")) then
			label.Text = tostring(state.CustomName)
		end
	end

	local function setFullbright(enabled)
		state.Fullbright = enabled
		disconnect("FullbrightClock")
		disconnect("FullbrightBrightness")
		disconnect("FullbrightExposure")
		if enabled then
			state.FullbrightBackup.ClockTime = Lighting.ClockTime
			state.FullbrightBackup.Brightness = Lighting.Brightness
			state.FullbrightBackup.ExposureCompensation = Lighting.ExposureCompensation
			state.FullbrightBackup.GlobalShadows = Lighting.GlobalShadows
			Lighting.ClockTime = 14
			Lighting.Brightness = 4
			Lighting.ExposureCompensation = 0.7
			Lighting.GlobalShadows = false
			local function enforce()
				if state.Fullbright then
					Lighting.ClockTime = 14
					Lighting.Brightness = 4
					Lighting.ExposureCompensation = 0.7
					Lighting.GlobalShadows = false
				end
			end
			state.Connections.FullbrightClock = Lighting:GetPropertyChangedSignal("ClockTime"):Connect(enforce)
			state.Connections.FullbrightBrightness = Lighting:GetPropertyChangedSignal("Brightness"):Connect(enforce)
			state.Connections.FullbrightExposure = Lighting:GetPropertyChangedSignal("ExposureCompensation")
				:Connect(enforce)
		else
			for property, value in pairs(state.FullbrightBackup) do
				pcall(function()
					Lighting[property] = value
				end)
			end
		end
	end

	local function isAdmin(player)
		if not player then
			return false
		end
		local groups = { 4165692 }
		for _, groupId in ipairs(groups) do
			local ok, rank = pcall(function()
				return player:GetRankInGroup(groupId)
			end)
			if ok and rank and rank > 0 then
				return true
			end
		end
		return false
	end

	local function checkAdmin(player)
		if state.AdminCheck and isAdmin(player) then
			notify("Admin Check", player.Name .. " joined with a group rank", 6)
		end
	end

	local function createTool(name, activated)
		local tool = Instance.new("Tool")
		tool.Name = name
		tool.RequiresHandle = false
		tool.Activated:Connect(function()
			pcall(activated, tool)
		end)
		tool.Parent = LocalPlayer:FindFirstChild("Backpack") or getCharacter()
		return tool
	end

	local function destroyTool(name)
		local backpack = LocalPlayer:FindFirstChild("Backpack")
		local character = getCharacter()
		local tool = backpack and backpack:FindFirstChild(name)
		if not tool and character then
			tool = character:FindFirstChild(name)
		end
		if tool then
			tool:Destroy()
		end
	end

	local function dealerStock()
		local result = {}
		local shops = findMapFolder("Shopz")
		if not shops then
			return result
		end
		for _, shop in ipairs(shops:GetChildren()) do
			local stocks = shop:FindFirstChild("CurrentStocks")
			if stocks then
				for _, value in ipairs(stocks:GetChildren()) do
					local amount = value:IsA("ValueBase") and value.Value or tonumber(value:GetAttribute("Stock"))
					result[value.Name] = amount or 0
				end
			end
		end
		return result
	end

	instances.Aimbot:SetupInputBindings()
	instances.Aimbot:StartMainLoop()
	instances.MeleeAura:StartRandomizer()
	instances.MeleeAura:StartMainLoop(isWhitelisted, isTargeted)
	instances.MeleeReach:StartMainLoop(isWhitelisted, isTargeted)

	callbacks.SilentAimV2Toggle = function(value)
		instances.SilentAimV2:Toggle(value, isWhitelisted, isTargeted)
	end
	callbacks.SilentAimV2CheckWhitelist = function(value)
		instances.SilentAimV2.settings.checkWhitelist = value
	end
	callbacks.SilentAimV2CheckTarget = function(value)
		instances.SilentAimV2.settings.checkTarget = value
	end
	callbacks.SilentAimV2DrawCircle = function(value)
		instances.SilentAimV2.settings.drawCircle = value
		instances.SilentAimV2:UpdateCircle()
	end
	callbacks.SilentAimV2FOVCenter = function(value)
		instances.SilentAimV2.settings.fovCircleCentered = value
		instances.SilentAimV2:UpdateCircle()
	end
	callbacks.SilentAimV2FOV = function(value)
		instances.SilentAimV2.settings.drawSize = value
		instances.SilentAimV2:UpdateCircleProps()
	end
	callbacks.SilentAimV2Distance = function(value)
		instances.SilentAimV2.settings.maxDistance = value
	end
	callbacks.SilentAimV2CheckWall = function(value)
		instances.SilentAimV2.settings.checkWall = value
	end
	callbacks.SilentAimV2UseHitChance = function(value)
		instances.SilentAimV2.settings.useHitChance = value
	end
	callbacks.SilentAimV2HitChance = function(value)
		instances.SilentAimV2.settings.hitChance = value
	end
	callbacks.SilentAimV2TargetPart = function(value)
		instances.SilentAimV2.settings.targetPart = value
		if value == "Random" then
			instances.SilentAimV2:StartRandomizer()
		else
			instances.SilentAimV2:StopRandomizer()
			instances.SilentAimV2.settings.actualPart = value
		end
	end
	callbacks.SilentAimV2TargetChangeTime = function(value)
		instances.SilentAimV2.settings.targetChangeTime = value
	end
	callbacks.SilentAimV2CheckDowned = function(value)
		instances.SilentAimV2.settings.checkDowned = value
	end
	callbacks.SilentAimV2ShowHitPartNotification = function(value)
		instances.SilentAimV2.settings.showHitPartNotification = value
	end
	callbacks.SilentAimV2NotificationSize = function(value)
		instances.SilentAimV2.settings.notificationSize = value
	end
	callbacks.SilentAimV2TeamCheck = function(value)
		instances.SilentAimV2.settings.teamCheck = value
	end
	callbacks.SilentAimV1Toggle = function(value)
		instances.SilentAimV1:Toggle(value, isWhitelisted, isTargeted)
	end
	callbacks.SilentAimV1DrawCircle = function(value)
		instances.SilentAimV1.settings.showFOVCircle = value
		instances.SilentAimV1:UpdateCircle()
	end
	callbacks.SilentAimV1FOVCenter = function(value)
		instances.SilentAimV1.settings.fovCircleCentered = value
		instances.SilentAimV1:UpdateCircle()
	end
	callbacks.SilentAimV1FOV = function(value)
		instances.SilentAimV1.settings.fovSize = value
		instances.SilentAimV1:UpdateCircle()
	end
	callbacks.SilentAimV1Distance = function(value)
		instances.SilentAimV1.settings.maxDistance = value
	end
	callbacks.SilentAimV1CheckWall = function(value)
		instances.SilentAimV1.settings.wallCheck = value
	end
	callbacks.SilentAimV1UseHitChance = function(value)
		instances.SilentAimV1.settings.useHitChance = value
	end
	callbacks.SilentAimV1HitChance = function(value)
		instances.SilentAimV1.settings.hitChance = value
	end
	callbacks.SilentAimV1TargetPart = function(value)
		instances.SilentAimV1.settings.targetPart = value
	end
	callbacks.SilentAimV1CheckDowned = function(value)
		instances.SilentAimV1.settings.checkDowned = value
	end
	callbacks.SilentAimV1TeamCheck = function(value)
		instances.SilentAimV1.settings.teamCheck = value
	end
	callbacks.RagebotToggle = function(value)
		instances.RageBot:Toggle(value, isWhitelisted, isTargeted, library)
	end
	callbacks.RagebotCheckWhitelist = function(value)
		instances.RageBot.settings.checkWhitelist = value
	end
	callbacks.RagebotCheckTarget = function(value)
		instances.RageBot.settings.checkTarget = value
	end
	callbacks.RagebotCheckDowned = function(value)
		instances.RageBot.settings.checkDowned = value
	end
	callbacks.RagebotWallCheck = function(value)
		instances.RageBot.settings.wallCheck = value
	end
	callbacks.RagebotHitlog = function(value)
		instances.RageBot.settings.hitlogEnabled = value
	end
	callbacks.RagebotShowFOV = function(value)
		instances.RageBot.settings.showFOV = value
		instances.RageBot:UpdateFOVCircle()
	end
	callbacks.RagebotUseFOV = function(value)
		instances.RageBot.settings.useFOV = value
	end
	callbacks.RagebotTeamCheck = function(value)
		instances.RageBot.settings.teamCheck = value
	end
	callbacks.RagebotFOVSlider = function(value)
		instances.RageBot.settings.fovRadius = value
		instances.RageBot:UpdateFOVCircle()
	end
	callbacks.RagebotShootSpeed = function(value)
		instances.RageBot:UpdateShootSpeed(value)
	end
	callbacks.RagebotDistanceSlider = function(value)
		instances.RageBot.settings.maxDistance = value
	end
	callbacks.BulletTracerToggle = function(value)
		instances.RageBot.settings.bulletTracerEnabled = value
	end
	callbacks.AimbotToggle = function(value)
		instances.Aimbot:Toggle(value)
	end
	callbacks.AimbotCheckWhitelist = function(value)
		instances.Aimbot.settings.checkWhitelist = value
	end
	callbacks.AimbotCheckTarget = function(value)
		instances.Aimbot.settings.checkTarget = value
	end
	callbacks.AimbotCheckDowned = function(value)
		instances.Aimbot.settings.checkDowned = value
	end
	callbacks.AimbotLockMouseButton = function(value)
		local key = tostring(value):upper()
		if key == "MB1" then
			instances.Aimbot.settings.lockMouseButton = Enum.UserInputType.MouseButton1
		elseif key == "MB2" then
			instances.Aimbot.settings.lockMouseButton = Enum.UserInputType.MouseButton2
		else
			instances.Aimbot.settings.lockMouseButton = Enum.KeyCode[key] or value
		end
	end
	callbacks.LockMethodDropdown = function(value)
		instances.Aimbot.settings.lockMethod = value
	end
	callbacks.PredictionToggle = function(value)
		instances.Aimbot.settings.usePrediction = value
	end
	callbacks.PredictionSlider = function(value)
		instances.Aimbot.settings.predictionMultiplier = value
	end
	callbacks.WallCheckToggle = function(value)
		instances.Aimbot.settings.wallCheckEnabled = value
	end
	callbacks.TeamCheckToggle = function(value)
		instances.Aimbot.settings.teamCheckEnabled = value
	end
	callbacks.ShowFOVToggle = function(value)
		instances.Aimbot.settings.showFOV = value
		instances.Aimbot:UpdateFOVCircle()
	end
	callbacks.CircleCenterOnlyToggle = function(value)
		instances.Aimbot.settings.circleCenterOnly = value
		instances.Aimbot:UpdateFOVCircle()
	end
	callbacks.FOVSlider = function(value)
		instances.Aimbot.settings.fovRadius = value
		instances.Aimbot:UpdateFOVCircle()
	end
	callbacks.SmoothingXSlider = function(value)
		instances.Aimbot.settings.smoothingX = value
	end
	callbacks.SmoothingYSlider = function(value)
		instances.Aimbot.settings.smoothingY = value
	end
	callbacks.LockPartDropdown = function(value)
		instances.Aimbot.settings.lockedPart = value
	end
	callbacks.RandomChangeTimeSlider = function(value)
		instances.Aimbot.settings.randomChangeTime = value
	end
	callbacks.AutoChangeTargetToggle = function(value)
		instances.Aimbot.settings.autoChangeTarget = value
	end
	callbacks.CircleButtonToggle = function(value)
		instances.Aimbot.settings.showCircleButton = value
		instances.Aimbot:CreateCircleButton()
		if instances.Aimbot.button then
			instances.Aimbot.button.Visible = value
		end
	end
	callbacks.CircleSizeSlider = function(value)
		instances.Aimbot.settings.circleSize = value
		instances.Aimbot:UpdateButtonPosition()
	end
	callbacks.CircleXSlider = function(value)
		instances.Aimbot.settings.circleXPercent = value
		instances.Aimbot:UpdateButtonPosition()
	end
	callbacks.CircleYSlider = function(value)
		instances.Aimbot.settings.circleYPercent = value
		instances.Aimbot:UpdateButtonPosition()
	end
	callbacks.MeleeAuraToggle = function(value)
		instances.MeleeAura.enabled = value
	end
	callbacks.MeleeAuraCheckWhitelist = function(value)
		instances.MeleeAura.settings.checkWhitelist = value
	end
	callbacks.MeleeAuraCheckTarget = function(value)
		instances.MeleeAura.settings.checkTarget = value
	end
	callbacks.MeleeAuraShowAnim = function(value)
		instances.MeleeAura.settings.showAnim = value
	end
	callbacks.MeleeAuraCheckDowned = function(value)
		instances.MeleeAura.settings.checkDowned = value
	end
	callbacks.MeleeAuraTeamCheck = function(value)
		instances.MeleeAura.settings.teamCheck = value
	end
	callbacks.MeleeAuraTargetPart = function(value)
		instances.MeleeAura.settings.targetPart = value
	end
	callbacks.MeleeAuraTargetChangeTime = function(value)
		instances.MeleeAura.settings.targetChangeTime = value
	end
	callbacks.MeleeAuraDistance = function(value)
		instances.MeleeAura.settings.distance = value
	end
	callbacks.MeleeReachToggle = function(value)
		instances.MeleeReach.enabled = value
	end
	callbacks.MeleeReachCheckWhitelist = function(value)
		instances.MeleeReach.settings.checkWhitelist = value
	end
	callbacks.MeleeReachCheckTarget = function(value)
		instances.MeleeReach.settings.checkTarget = value
	end
	callbacks.MeleeReachCheckDowned = function(value)
		instances.MeleeReach.settings.checkDowned = value
	end
	callbacks.MeleeReachTeamCheck = function(value)
		instances.MeleeReach.settings.teamCheck = value
	end
	callbacks.MeleeReachDistance = function(value)
		instances.MeleeReach.settings.distance = value
	end

	callbacks.WhitelistDropdown = function(value)
		state.Whitelist = normalizeSelection(value)
		updateAllPlayerHighlights()
	end
	callbacks.HighlightWhitelist = function(value)
		state.HighlightWhitelist = value
		updateAllPlayerHighlights()
	end
	callbacks.WhitelistColorPicker = function(value)
		state.WhitelistColor = value
		updateAllPlayerHighlights()
	end
	callbacks.RefreshWhitelist = function()
		return playerNames()
	end
	callbacks.ClearWhitelist = function()
		table.clear(state.Whitelist)
		updateAllPlayerHighlights()
	end
	callbacks.TargetDropdown = function(value)
		state.Targets = normalizeSelection(value)
		updateAllPlayerHighlights()
	end
	callbacks.HighlightTarget = function(value)
		state.HighlightTarget = value
		updateAllPlayerHighlights()
	end
	callbacks.TargetColorPicker = function(value)
		state.TargetColor = value
		updateAllPlayerHighlights()
	end
	callbacks.RefreshTarget = function()
		return playerNames()
	end
	callbacks.ClearTarget = function()
		table.clear(state.Targets)
		updateAllPlayerHighlights()
	end
	callbacks.FastPickupToggle = function(value)
		state.FastPickup = value
	end
	callbacks.InfiniteStaminaToggle = function(value)
		state.InfiniteStamina = value
		if value then
			startLoop("InfiniteStamina", 0, nil, function()
				local character = getCharacter()
				local stats = character and character:FindFirstChild("CharStats")
				local stamina = stats and (stats:FindFirstChild("Stamina") or stats:FindFirstChild("Energy"))
				if stamina and stamina:IsA("ValueBase") then
					stamina.Value = stamina:GetAttribute("Max") or 100
				end
			end)
		else
			stopLoop("InfiniteStamina")
		end
	end
	callbacks.NoFailLockpickToggle = function(value)
		state.NoFailLockpick = value
	end
	callbacks.NoclipToggle = function(value)
		state.Noclip = value
		disconnect("Noclip")
		if value then
			state.Connections.Noclip = RunService.Stepped:Connect(function()
				local character = getCharacter()
				if character then
					for _, object in ipairs(character:GetDescendants()) do
						if object:IsA("BasePart") then
							object.CanCollide = false
						end
					end
				end
			end)
		end
	end
	callbacks.CtrlClickTP = function(value)
		state.CtrlClickTP = value
	end
	callbacks.QTeleportToggle = function(value)
		state.QTeleport = value
	end
	callbacks.ChatToggle = function(value)
		pcall(function()
			TextChatService.ChatWindowConfiguration.Enabled = value
		end)
	end
	callbacks.StompSpeedSlider = function(value)
		state.StompSpeed = value
	end
	callbacks.ToggleAntiSmoke = function(value)
		state.AntiSmoke = value
		disconnect("AntiSmoke")
		if value then
			local debris = Workspace:FindFirstChild("Debris") or Workspace
			state.Connections.AntiSmoke = debris.ChildAdded:Connect(removeSmoke)
			for _, object in ipairs(debris:GetDescendants()) do
				removeSmoke(object)
			end
		end
	end
	callbacks.NoFallToggle = function(value)
		state.NoFall = value
		disconnect("NoFallCharacter")
		if value then
			ensureForceField(getCharacter())
			state.Connections.NoFallCharacter = LocalPlayer.CharacterAdded:Connect(function(character)
				character:WaitForChild("HumanoidRootPart")
				character:WaitForChild("Humanoid")
				ensureForceField(character)
			end)
		else
			removeForceField()
		end
	end
	callbacks.ToggleAntiFractured = function(value)
		state.AntiFractured = value
		disconnect("AntiFractured")
		if value then
			repairFractures(getCharacter())
			state.Connections.AntiFractured = RunService.Heartbeat:Connect(function()
				repairFractures(getCharacter())
			end)
		end
	end
	callbacks.NoBarriersToggle = function(value)
		state.NoBarriers = value
		setBarrierCollision(not value)
	end
	callbacks.NoNeckToggle = function(value)
		state.NoNeck = value
		local character = getCharacter()
		if character then
			character:SetAttribute("NoNeckMovement", value)
		end
	end
	callbacks.AntiAFKToggle = function(value)
		disconnect("AntiAFK")
		if value then
			state.Connections.AntiAFK = LocalPlayer.Idled:Connect(function()
				VirtualUser:CaptureController()
				VirtualUser:ClickButton2(Vector2.new())
			end)
		end
	end

	callbacks.AutoBreakSafeRegister = function(value)
		state.AutoBreakSafeRegister = value
		if value then
			startLoop("AutoBreakSafeRegister", 0.35, function()
				return state.AutoBreakSafeRegister
			end, function()
				local root = getRoot()
				local tool = getTool()
				if not root or not tool then
					return
				end
				for _, object in ipairs(Workspace:GetDescendants()) do
					local lower = object.Name:lower()
					if lower:find("safe", 1, true) or lower:find("register", 1, true) then
						local part = object:IsA("BasePart") and object or object:FindFirstChildWhichIsA("BasePart")
						if part and (part.Position - root.Position).Magnitude <= 12 then
							pcall(function()
								tool:Activate()
							end)
							local prompt = object:FindFirstChildWhichIsA("ProximityPrompt", true)
							activatePrompt(prompt)
						end
					end
				end
			end)
		else
			stopLoop("AutoBreakSafeRegister")
		end
	end
	callbacks.AutoLockpick = function(value)
		state.AutoLockpick = value
		disconnect("AutoLockpick")
		local playerGui = LocalPlayer:WaitForChild("PlayerGui")
		local function solve(gui)
			if not state.AutoLockpick or not gui then
				return
			end
			for _, object in ipairs(gui:GetDescendants()) do
				if object:IsA("GuiButton") then
					pcall(function()
						object:Activate()
					end)
				elseif object:IsA("NumberValue") then
					object.Value = 0
				elseif object:IsA("BoolValue") then
					object.Value = true
				end
			end
		end
		if value then
			local existing = playerGui:FindFirstChild("LockpickGUI")
			if existing then
				solve(existing)
			end
			state.Connections.AutoLockpick = playerGui.ChildAdded:Connect(function(child)
				if child.Name == "LockpickGUI" then
					task.defer(solve, child)
				end
			end)
		end
	end
	callbacks.AutoPickUpTool = function(value)
		state.AutoPickUpTool = value
		if value then
			startLoop("AutoPickUpTool", 0.15, function()
				return state.AutoPickUpTool
			end, function()
				local filter = Workspace:FindFirstChild("Filter")
				local folder = filter and filter:FindFirstChild("SpawnedTools")
				local root = getRoot()
				if not folder or not root then
					return
				end
				for _, object in ipairs(folder:GetChildren()) do
					local part = object:FindFirstChildOfClass("MeshPart") or object:FindFirstChildWhichIsA("BasePart")
					if part and (root.Position - part.Position).Magnitude <= 35 then
						touchObject(root, object)
						local prompt = object:FindFirstChildWhichIsA("ProximityPrompt", true)
						activatePrompt(prompt)
					end
				end
			end)
		else
			stopLoop("AutoPickUpTool")
		end
	end
	callbacks.AutoPickUpCash = function(value)
		state.AutoPickUpCash = value
		if value then
			startLoop("AutoPickUpCash", 0.12, function()
				return state.AutoPickUpCash
			end, function()
				local filter = Workspace:FindFirstChild("Filter")
				local folder = filter and filter:FindFirstChild("SpawnedBread")
				local root = getRoot()
				if not folder or not root then
					return
				end
				for _, object in ipairs(folder:GetChildren()) do
					local part = object:IsA("BasePart") and object or object:FindFirstChildWhichIsA("BasePart")
					if part and (root.Position - part.Position).Magnitude <= 45 then
						touchObject(root, object)
						local prompt = object:FindFirstChildWhichIsA("ProximityPrompt", true)
						activatePrompt(prompt)
					end
				end
			end)
		else
			stopLoop("AutoPickUpCash")
		end
	end
	callbacks.AutoRepairToggle = function(value)
		state.AutoRefill = value
		if value then
			startLoop("AutoRefill", 0.4, function()
				return state.AutoRefill
			end, function()
				local root = getRoot()
				if not root then
					return
				end
				local shops = findMapFolder("Shopz")
				if not shops then
					return
				end
				for _, shop in ipairs(shops:GetChildren()) do
					local part = shop:FindFirstChild("MainPart") or shop:FindFirstChildWhichIsA("BasePart")
					if part and (root.Position - part.Position).Magnitude <= 15 then
						local prompt = shop:FindFirstChildWhichIsA("ProximityPrompt", true)
						activatePrompt(prompt)
					end
				end
			end)
		else
			stopLoop("AutoRefill")
		end
	end
	callbacks.AutoDeposit = function(value)
		state.AutoDeposit = value
		disconnect("AutoDeposit")
		if value then
			state.Connections.AutoDeposit = RunService.Heartbeat:Connect(function()
				local root = getRoot()
				local atms = findMapFolder("ATMz")
				if not root or not atms then
					return
				end
				for _, atm in ipairs(atms:GetChildren()) do
					local part = atm:FindFirstChild("MainPart") or atm:FindFirstChildWhichIsA("BasePart")
					if part and (part.Position - root.Position).Magnitude <= 15 then
						activatePrompt(atm:FindFirstChildWhichIsA("ProximityPrompt", true))
					end
				end
			end)
		end
	end
	callbacks.AutoUnlockDoor = function(value)
		state.AutoUnlockDoor = value
		if value then
			startLoop("AutoUnlockDoor", 0.25, function()
				return state.AutoUnlockDoor
			end, function()
				processDoors("unlock")
			end)
		else
			stopLoop("AutoUnlockDoor")
		end
	end
	callbacks.AutoCloseDoors = function(value)
		state.AutoCloseDoors = value
		if value then
			startLoop("AutoCloseDoors", 0.25, function()
				return state.AutoCloseDoors
			end, function()
				processDoors("close")
			end)
		else
			stopLoop("AutoCloseDoors")
		end
	end
	callbacks.AutoOpenDoors = function(value)
		state.AutoOpenDoors = value
		if value then
			startLoop("AutoOpenDoors", 0.25, function()
				return state.AutoOpenDoors
			end, function()
				processDoors("open")
			end)
		else
			stopLoop("AutoOpenDoors")
		end
	end
	callbacks.AutoRespawnToggle = function(value)
		state.AutoRespawn = value
		disconnect("AutoRespawn")
		local humanoid = getHumanoid()
		if value and humanoid then
			state.Connections.AutoRespawn = humanoid.Died:Connect(function()
				task.wait(2)
				local remote = getEvent("Respawn")
				if remote then
					remote:FireServer()
				else
					pcall(function()
						LocalPlayer:LoadCharacter()
					end)
				end
			end)
		end
	end
	callbacks.autofarmandnocliptoggle = function(value)
		callbacks.NoclipToggle(value)
	end
	callbacks.autofarmallowance = function()
		state.AutoFarmAllowance = true
		local prompt = nearestPrompt(getRoot(), function(object)
			local name = object.Parent and object.Parent.Name:lower() or ""
			return name:find("allowance", 1, true) or name:find("dealer", 1, true)
		end)
		activatePrompt(prompt)
	end

	callbacks.IncreaseSpeed = function(value)
		state.SpeedEnabled = value
		disconnect("Speed")
		if value then
			state.Connections.Speed = RunService.Heartbeat:Connect(applyMovement)
		else
			local humanoid = getHumanoid()
			if humanoid then
				humanoid.WalkSpeed = 16
			end
		end
	end
	callbacks.SpeedValue = function(value)
		state.Speed = value
		applyMovement()
	end
	callbacks.JumpPowerToggle = function(value)
		state.JumpEnabled = value
		disconnect("Jump")
		if value then
			state.Connections.Jump = RunService.Heartbeat:Connect(applyMovement)
		else
			local humanoid = getHumanoid()
			if humanoid then
				humanoid.JumpPower = 50
			end
		end
	end
	callbacks.JumpPowerSlider = function(value)
		state.JumpPower = value
		applyMovement()
	end
	callbacks.FlyToggle = function(value)
		state.FlyEnabled = value
		if value then
			startFly()
		else
			stopFly()
		end
	end
	callbacks.MobileFlyToggle = callbacks.FlyToggle
	callbacks.FlySpeedSlider = function(value)
		state.FlySpeed = value
	end
	callbacks.FlyMethodDropdown = function(value)
		state.FlyMethod = value
	end

	callbacks.WallBangToggle = function(value)
		state.WallBangEnabled = value
		local map = Workspace:FindFirstChild("Map")
		local characters = Workspace:FindFirstChild("Characters")
		local parts = map and map:FindFirstChild("Parts")
		local wallParts = parts and parts:FindFirstChild("M_Parts")
		if not wallParts then
			return
		end
		if value then
			state.WallPartsOriginalParent = wallParts.Parent
			wallParts.Parent = characters or Workspace
		else
			wallParts.Parent = state.WallPartsOriginalParent or parts
		end
	end
	callbacks.InstantEquipToggle = function(value)
		state.InstantEquip = value
		disconnect("InstantEquip")
		local backpack = LocalPlayer:FindFirstChild("Backpack")
		if value and backpack then
			state.Connections.InstantEquip = backpack.ChildAdded:Connect(function(tool)
				if tool:IsA("Tool") then
					task.defer(function()
						local character = getCharacter()
						if character then
							tool.Parent = character
						end
					end)
				end
			end)
		end
	end
	callbacks.InstantReload = function(value)
		state.InstantReload = value
		disconnect("InstantReload")
		local function bind(tool)
			if not state.InstantReload or not tool then
				return
			end
			local values = tool:FindFirstChild("Values")
			local stored = values and values:FindFirstChild("SERVER_StoredAmmo")
			if stored then
				state.Connections.InstantReload = stored:GetPropertyChangedSignal("Value"):Connect(function()
					local remote = getEvent("GNX_R")
					if remote then
						remote:FireServer(tool)
					end
				end)
			end
		end
		if value then
			bind(getTool())
			disconnect("InstantReloadCharacter")
			state.Connections.InstantReloadCharacter = LocalPlayer.CharacterAdded:Connect(function(character)
				state.Connections.InstantReloadTool = character.ChildAdded:Connect(function(child)
					if child:IsA("Tool") then
						bind(child)
					end
				end)
			end)
		end
	end
	callbacks.CustomRecoilToggle = function(value)
		state.CustomRecoilEnabled = value
	end
	callbacks.CustomRecoilSlider = function(value)
		state.RecoilScale = value
	end
	callbacks.InfinitePepper = function(value)
		state.InfiniteSpray = value
		if value then
			startLoop("InfinitePepper", 0, function()
				return state.InfiniteSpray
			end, function()
				local tool = getCharacter() and getCharacter():FindFirstChild("Pepper-spray")
				local ammo = tool and tool:FindFirstChild("Ammo")
				if ammo and ammo:IsA("ValueBase") then
					ammo.Value = math.huge
				end
			end)
		else
			stopLoop("InfinitePepper")
		end
	end
	callbacks.PepperAura = function(value)
		state.PepperAuraEnabled = value
		disconnect("PepperAura")
		if value then
			state.Connections.PepperAura = RunService.RenderStepped:Connect(function()
				local character = getCharacter()
				local root = getRoot(character)
				local tool = character and character:FindFirstChild("Pepper-spray")
				local ammo = tool and tool:FindFirstChild("Ammo")
				local remote = tool and tool:FindFirstChild("RemoteEvent")
				if ammo and ammo:IsA("ValueBase") then
					ammo.Value = math.huge
				end
				if root and remote then
					for _, player in ipairs(Players:GetPlayers()) do
						if player ~= LocalPlayer and player.Character then
							local targetRoot = getRoot(player.Character)
							if
								targetRoot
								and (targetRoot.Position - root.Position).Magnitude <= state.PepperAuraRange
							then
								remote:FireServer(targetRoot.Position, targetRoot)
							end
						end
					end
				end
			end)
		end
	end
	callbacks.PepperAuraRange = function(value)
		state.PepperAuraRange = value
	end
	callbacks.C4Toggle = function(value)
		state.C4Enabled = value
	end
	callbacks.C4Key = function()
		if not state.C4Enabled then
			return
		end
		local character = getCharacter()
		local c4 = character and character:FindFirstChild("C4")
		local root = getRoot(character)
		if c4 and root then
			local part = c4:FindFirstChildWhichIsA("BasePart")
			if part then
				part.AssemblyLinearVelocity = Camera.CFrame.LookVector * state.C4Speed
			end
		end
	end
	callbacks.C4Speed = function(value)
		state.C4Speed = value
	end
	callbacks.ExplosionAmmoToggle = function(value)
		state.ExplosionAmmoEnabled = value
	end
	callbacks.ExplosionAmmoKey = function()
		if not state.ExplosionAmmoEnabled then
			return
		end
		local remote = getEvent("ZFKLF__H")
		local tool = getTool()
		local root = getRoot()
		if remote and tool and root then
			remote:FireServer(
				"🧈",
				tool,
				tostring(tick()),
				1,
				root,
				root.Position + Camera.CFrame.LookVector * state.ExplosionAmmoSpeed,
				Camera.CFrame.LookVector
			)
		end
	end
	callbacks.ExplosionAmmoSpeed = function(value)
		state.ExplosionAmmoSpeed = value
	end

	callbacks.ESPBox = function(value)
		Visual.ESP.Enabled.Box = value
	end
	callbacks.ESPBoxColor = function(value)
		Visual.ESP.Settings.BoxColor = value
	end
	callbacks.ESPHighlight = function(value)
		Visual.ESP.Enabled.Highlight = value
	end
	callbacks.ESPHighlightColor = function(value)
		Visual.ESP.Settings.HighlightColor = value
	end
	callbacks.ESPName = function(value)
		Visual.ESP.Enabled.Name = value
	end
	callbacks.ESPHealth = function(value)
		Visual.ESP.Enabled.Health = value
	end
	callbacks.ESPHealthBar = function(value)
		Visual.ESP.Enabled.HealthBar = value
	end
	callbacks.ESPTool = function(value)
		Visual.ESP.Enabled.Tool = value
	end
	callbacks.ESPDistance = function(value)
		Visual.ESP.Enabled.Distance = value
	end
	callbacks.ESPTeamCheck = function(value)
		Visual.ESP.Settings.TeamCheck = value
	end
	callbacks.ESPTracer = function(value)
		Visual.ESP.Enabled.Tracer = value
	end
	callbacks.TracerPosition = function(value)
		Visual.ESP.Settings.TracerPosition = value
	end
	callbacks.ArmsChams = function(value)
		Visual.ArmsChams.Enabled = value
		Visual.ApplyArmsChams()
	end
	callbacks.ArmsChamColor = function(value)
		Visual.ArmsChams.Color = value
		Visual.ApplyArmsChams()
	end
	callbacks.DetectNearby = function(value)
		if value then
			Visual.EnablePlayerDetect()
		else
			Visual.DisablePlayerDetect()
		end
	end
	callbacks.DetectDistance = function(value)
		Visual.PlayerDetect.Distance = value
	end
	callbacks.ScrapESPToggle = function(value)
		state.ScrapESP = value
		scanESP("JXScrapESP", value, function(object)
			local lower = object.Name:lower()
			return lower:find("scrap", 1, true) ~= nil
		end, Color3.fromRGB(255, 200, 0))
	end
	callbacks.ScrapDistance = function(value)
		state.ScrapDistance = value
		if state.ScrapESP then
			callbacks.ScrapESPToggle(true)
		end
	end
	callbacks.ScrapRarity = function(value)
		state.ScrapTypes = normalizeSelection(value)
		if state.ScrapESP then
			callbacks.ScrapESPToggle(true)
		end
	end
	callbacks.CashDropESPToggle = function(value)
		state.CashDropESP = value
		scanESP("JXCashESP", value, function(object)
			local lower = object.Name:lower()
			return lower:find("bread", 1, true) or lower:find("cash", 1, true)
		end, Color3.fromRGB(0, 255, 0))
	end
	callbacks.ToolsESPToggle = function(value)
		state.ToolsESP = value
		scanESP("JXToolESP", value, function(object)
			return object:IsA("Tool")
		end, Color3.fromRGB(0, 170, 255))
	end
	callbacks.SafeESP = function(value)
		state.SafeESP = value
		scanESP("JXSafeESP", value, function(object)
			local lower = object.Name:lower()
			return lower:find("safe", 1, true) or lower:find("register", 1, true)
		end, Color3.fromRGB(255, 0, 0))
	end
	callbacks.ESPATM = function(value)
		state.ATMESP = value
		scanESP("JXATMESP", value, function(object)
			return object.Name:lower():find("atm", 1, true) ~= nil
		end, Color3.fromRGB(0, 255, 255))
	end
	callbacks.ESPDealer = function(value)
		state.DealerESP = value
		scanESP("JXDealerESP", value, function(object)
			local lower = object.Name:lower()
			return lower:find("dealer", 1, true) or lower:find("shop", 1, true)
		end, Color3.fromRGB(255, 0, 255))
	end
	callbacks.HideLevelUI = function(value)
		state.HideLevelUI = value
		local core = findCoreGui()
		local level = core and core:FindFirstChild("LevelFrame", true)
		if level and level:IsA("GuiObject") then
			level.Visible = not value
		end
	end
	callbacks.VisualLevel = function(value)
		state.VisualLevel = value
	end
	callbacks.ApplyVisualLevel = applyVisualLevel
	callbacks.CustomRegion = function(value)
		state.CustomRegion = value
	end
	callbacks.ApplyRegion = applyRegion
	callbacks.CustomName = function(value)
		state.CustomName = value
	end
	callbacks.ApplyName = applyName
	callbacks.DeleteAllDoors = function()
		local doors = findMapFolder("Doors")
		if doors then
			for _, object in ipairs(doors:GetChildren()) do
				if object:IsA("Model") or object:IsA("BasePart") then
					object:Destroy()
				end
			end
		end
	end
	callbacks.FullBrightToggle = setFullbright
	callbacks.AdminCheckToggle = function(value)
		state.AdminCheck = value
		disconnect("AdminCheck")
		if value then
			for _, player in ipairs(Players:GetPlayers()) do
				checkAdmin(player)
			end
			state.Connections.AdminCheck = Players.PlayerAdded:Connect(checkAdmin)
		end
	end
	callbacks.UseFOVToggle = function(value)
		state.UseCustomFOV = value
		disconnect("CustomFOV")
		if value then
			Camera.FieldOfView = state.FOV
			state.Connections.CustomFOV = RunService.RenderStepped:Connect(function()
				Camera.FieldOfView = state.FOV
			end)
		end
	end
	callbacks.FOVSliderMisc = function(value)
		state.FOV = value
		if state.UseCustomFOV then
			Camera.FieldOfView = value
		end
	end
	callbacks.CameraZoomSlider = function(value)
		state.CameraDistance = value
		LocalPlayer.CameraMaxZoomDistance = value
		LocalPlayer.CameraMinZoomDistance = math.min(LocalPlayer.CameraMinZoomDistance, value)
	end
	callbacks.HugToolToggle = function(value)
		state.Hug = value
		destroyTool("Hug")
		if value then
			state.Objects.Hug = createTool("Hug", function()
				local humanoid = getHumanoid()
				if not humanoid then
					return
				end
				local first = Instance.new("Animation")
				first.AnimationId = "rbxassetid://283545583"
				local second = Instance.new("Animation")
				second.AnimationId = "rbxassetid://225975820"
				local firstTrack = humanoid:LoadAnimation(first)
				local secondTrack = humanoid:LoadAnimation(second)
				firstTrack:Play()
				secondTrack:Play()
			end)
		end
	end
	callbacks.JerkToolToggle = function(value)
		state.Jerk = value
		destroyTool("Jerk")
		if value then
			state.Objects.Jerk = createTool("Jerk", function()
				local humanoid = getHumanoid()
				if not humanoid then
					return
				end
				for _, track in ipairs(humanoid:GetPlayingAnimationTracks()) do
					if track.Priority == Enum.AnimationPriority.Action4 then
						track:Stop()
					end
				end
			end)
		end
	end
	callbacks.CarpetToolToggle = function(value)
		state.Carpet = value
		destroyTool("Carpet")
		if value then
			state.Objects.Carpet = createTool("Carpet", function()
				local root = getRoot()
				if root then
					root.CFrame *= CFrame.Angles(0, math.rad(180), 0)
				end
			end)
		end
	end
	callbacks.FakeDownedToolToggle = function(value)
		state.FakeDowned = value
		destroyTool("Fake-Downed")
		if value then
			state.Objects.FakeDowned = createTool("Fake-Downed", function()
				local humanoid = getHumanoid()
				if humanoid then
					humanoid:ChangeState(Enum.HumanoidStateType.Physics)
				end
			end)
		end
	end
	callbacks.HideHead = function(value)
		state.HideHead = value
		local character = getCharacter()
		local humanoid = getHumanoid(character)
		if not humanoid then
			return
		end
		if value then
			local animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)
			local animation = Instance.new("Animation")
			animation.AnimationId = "rbxassetid://68339848"
			local track = animator:LoadAnimation(animation)
			track.Looped = true
			track:Play()
			state.Objects.HideHeadTrack = track
		else
			local track = state.Objects.HideHeadTrack
			if track then
				track:Stop()
				state.Objects.HideHeadTrack = nil
			end
		end
	end
	callbacks.hidebodytoggle = function(value)
		state.HideBody = value
		local character = getCharacter()
		local root = getRoot(character)
		local humanoid = getHumanoid(character)
		local torso = character and (character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso"))
		if value and root then
			Camera.CameraSubject = root
		elseif humanoid then
			Camera.CameraSubject = humanoid
		end
		if torso then
			torso.LocalTransparencyModifier = value and 1 or 0
		end
	end
	callbacks.KeyESPToggle = function(value)
		state.KeyESP = value
		local keys = Workspace:FindFirstChild("Map")
		local squid = keys and keys:FindFirstChild("SquidDirectory")
		local hide = squid and squid:FindFirstChild("HideAndSeek")
		local scriptables = hide and hide:FindFirstChild("Scriptables")
		local folder = scriptables and scriptables:FindFirstChild("Keys")
		if folder then
			for _, key in ipairs(folder:GetChildren()) do
				if value then
					createHighlight(key, "JXKeyESP", Color3.fromRGB(255, 255, 0))
				else
					local h = key:FindFirstChild("JXKeyESP")
					if h then
						h:Destroy()
					end
				end
			end
		end
	end
	callbacks.HighlightExitToggle = function(value)
		state.HighlightExit = value
		local map = Workspace:FindFirstChild("Map")
		local squid = map and map:FindFirstChild("SquidDirectory")
		local hide = squid and squid:FindFirstChild("HideAndSeek")
		local scriptables = hide and hide:FindFirstChild("Scriptables")
		local exits = scriptables and scriptables:FindFirstChild("Exits")
		if exits then
			for _, exit in ipairs(exits:GetChildren()) do
				if value then
					createHighlight(exit, "ExitHighlight", Color3.fromRGB(0, 255, 0))
				else
					local h = exit:FindFirstChild("ExitHighlight")
					if h then
						h:Destroy()
					end
				end
			end
		end
	end
	callbacks.RemoveFakeGlass = function()
		local map = Workspace:FindFirstChild("Map")
		local squid = map and map:FindFirstChild("SquidDirectory")
		local hopscotch = squid and squid:FindFirstChild("Hopscotch")
		local scriptables = hopscotch and hopscotch:FindFirstChild("Scriptables")
		local glass = scriptables and scriptables:FindFirstChild("Glass")
		if glass then
			for _, object in ipairs(glass:GetChildren()) do
				if object:GetAttribute("Fake") or object.Name:lower():find("fake", 1, true) then
					object:Destroy()
				end
			end
		end
	end
	callbacks.DealerStockChecker = function(value)
		state.DealerStockChecker = value
		if value then
			startLoop("DealerStock", 2, function()
				return state.DealerStockChecker
			end, function()
				local stock = dealerStock()
				for item, amount in pairs(stock) do
					if state.SelectedItems[item] and amount > 0 and state.NotifyNewStock then
						notify("Dealer Stock", item .. ": " .. tostring(amount), 4)
					end
				end
			end)
		else
			stopLoop("DealerStock")
		end
	end
	callbacks.SelectItemsToCheck = function(value)
		state.SelectedItems = normalizeSelection(value)
	end
	callbacks.NotifyNewStock = function(value)
		state.NotifyNewStock = value
	end
	callbacks.ESPStockDealerToggle = function(value)
		state.ESPStockDealer = value
		callbacks.ESPDealer(value)
	end
	callbacks.ESPStockDealer = function(value)
		state.SelectedStockDealer = value
	end
	callbacks.RefreshStock = dealerStock
	callbacks.KeybindMenuOpen = function(value)
		if library then
			library.KeybindFrame.Visible = value
		end
	end
	callbacks.ShowCustomCursor = function(value)
		if library then
			library.ShowCustomCursor = value
		end
	end
	callbacks.NotificationSide = function(value)
		if library then
			library.NotificationSide = value
		end
	end
	callbacks.DPIScale = function(value)
		if library and type(library.SetDPIScale) == "function" then
			library:SetDPIScale(value)
		end
	end
	callbacks.UnloadScript = function()
		for name in pairs(state.Loops) do
			stopLoop(name)
		end
		for name in pairs(state.Connections) do
			disconnect(name)
		end
		for name in pairs(state.Objects) do
			destroy(name)
		end
		pcall(function()
			instances.SilentAimV1:Cleanup()
		end)
		pcall(function()
			Visual.ESPLoop:Disconnect()
		end)
		if library and type(library.Unload) == "function" then
			library:Unload()
		end
	end

	state.Connections.TeleportInput = UserInputService.InputBegan:Connect(function(input, processed)
		if processed then
			return
		end
		local root = getRoot()
		if not root then
			return
		end
		if
			state.CtrlClickTP
			and input.UserInputType == Enum.UserInputType.MouseButton1
			and (
				UserInputService:IsKeyDown(Enum.KeyCode.LeftControl)
				or UserInputService:IsKeyDown(Enum.KeyCode.RightControl)
			)
		then
			local mouse = LocalPlayer:GetMouse()
			if mouse and mouse.Hit then
				root.CFrame = mouse.Hit + Vector3.new(0, 3, 0)
			end
		elseif state.QTeleport and input.KeyCode == Enum.KeyCode.Q then
			local mouse = LocalPlayer:GetMouse()
			if mouse and mouse.Hit then
				root.CFrame = mouse.Hit + Vector3.new(0, 3, 0)
			end
		end
	end)

	state.Connections.PlayerAddedHighlights = Players.PlayerAdded:Connect(function(player)
		player.CharacterAdded:Connect(function()
			task.wait(0.5)
			updatePlayerHighlight(player)
		end)
	end)

	local skinWeapons = {
		"Beretta",
		"G-17",
		"TEC-9",
		"M1911",
		"FNP-45",
		"Deagle",
		"Uzi",
		"MAC-10",
		"Tommy",
		"Sawn-Off",
		"Ithaca-37",
		"Super-Shorty",
		"AKS-74U",
		"M4A1",
		"SKS",
		"FN-FAL-S",
		"M320-1",
		"RPG-7",
		"Shiv",
		"Bayonet",
		"Taiga",
		"Rambo",
		"Baton",
		"Machete",
		"Fireaxe",
		"Crowbar",
		"Chainsaw",
		"Scythe",
		"Balisong",
		"Bat",
		"Golf Club",
		"Katana",
		"Metal Bat",
		"Shovel",
		"Slayer",
		"Wrench",
		"Sledgehammer",
	}

	local skinFlags = {
		Beretta = "gun_Beretta",
		["G-17"] = "gun_G17",
		["TEC-9"] = "gun_TEC9",
		M1911 = "gun_M1911",
		["FNP-45"] = "gun_FNP45",
		Deagle = "gun_Deagle",
		Uzi = "gun_Uzi",
		["MAC-10"] = "gun_MAC10",
		Tommy = "gun_Tommy",
		["Sawn-Off"] = "gun_SawnOff",
		["Ithaca-37"] = "gun_Ithaca37",
		["Super-Shorty"] = "gun_SuperShorty",
		["AKS-74U"] = "gun_AKS74U",
		M4A1 = "gun_M4A1",
		SKS = "gun_SKS",
		["FN-FAL-S"] = "gun_FNFALS",
		["M320-1"] = "gun_M3201",
		["RPG-7"] = "gun_RPG7",
		Shiv = "melee_Shiv",
		Bayonet = "melee_Bayonet",
		Taiga = "melee_Taiga",
		Rambo = "melee_Rambo",
		Baton = "melee_Baton",
		Machete = "melee_Machete",
		Fireaxe = "melee_Fireaxe",
		Crowbar = "melee_Crowbar",
		Chainsaw = "melee_Chainsaw",
		Scythe = "melee_Scythe",
		Balisong = "melee_Balisong",
		Bat = "melee_Bat",
		["Golf Club"] = "melee_Golf_Club",
		Katana = "melee_Katana",
		["Metal Bat"] = "melee_Metal_Bat",
		Shovel = "melee_Shovel",
		Slayer = "melee_Slayer",
		Wrench = "melee_Wrench",
		Sledgehammer = "melee_Sledgehammer",
	}

	for _, weaponName in ipairs(skinWeapons) do
		local prefix = skinFlags[weaponName]
		callbacks[prefix .. "_SkinDropdown"] = function(value)
			state.SelectedSkins[weaponName] = value
		end
		callbacks[prefix .. "_ApplySkin"] = function()
			applySkin(weaponName)
		end
		callbacks[prefix .. "_AutoToggle"] = function(value)
			setAutoSkin(weaponName, value)
		end
	end

	callbacks["Remove Fake Glass"] = callbacks.RemoveFakeGlass
	callbacks["Refresh Stock"] = callbacks.RefreshStock
	callbacks["Unload Script"] = callbacks.UnloadScript
	callbacks["FOVSlider"] = callbacks.FOVSlider
	callbacks["FOVSliderMisc"] = callbacks.FOVSliderMisc
	callbacks.FlyKeybind = function()
		return nil
	end

	return {
		Callbacks = callbacks,
		State = state,
		Instances = instances,
		IsPlayerWhitelisted = isWhitelisted,
		IsPlayerTargeted = isTargeted,
		ApplySkin = applySkin,
		RefreshStock = dealerStock,
		Unload = callbacks.UnloadScript,
	}
end

CriminalityV1.Visual = VisualModule
CriminalityV1.VerifiedCallbacks = VerifiedCallbacks
CriminalityV1.Root = RootModule

return CriminalityV1
