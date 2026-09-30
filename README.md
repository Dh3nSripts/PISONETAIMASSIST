-- DHEN AIM ASSIST + GOOD GRAPHICS
-- LocalScript
-- StarterPlayer > StarterPlayerScripts

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

--==================================================
-- AIM SETTINGS
--==================================================

local FOV_RADIUS = 100
local AIM_STRENGTH = 0.14
local MAX_DISTANCE = 300

local aiming = false

--==================================================
-- GOOD GRAPHICS / LONG-DISTANCE VISIBILITY
--==================================================

Lighting.GlobalShadows = true
Lighting.Brightness = 2
Lighting.ExposureCompensation = 0.05
Lighting.EnvironmentDiffuseScale = 0.85
Lighting.EnvironmentSpecularScale = 0.8
Lighting.ClockTime = 14

-- Remove strong fog/atmosphere that can hide distant objects
Lighting.FogStart = 0
Lighting.FogEnd = 100000

local atmosphere = Lighting:FindFirstChild("DhenAtmosphere")

if atmosphere then
	atmosphere.Density = 0.02
	atmosphere.Haze = 0.05
	atmosphere.Glare = 0
	atmosphere.Offset = 0
end

--==================================================
-- COLOR
--==================================================

local color = Lighting:FindFirstChild("DhenColor")

if not color then
	color = Instance.new("ColorCorrectionEffect")
	color.Name = "DhenColor"
	color.Parent = Lighting
end

color.Brightness = 0.02
color.Contrast = 0.10
color.Saturation = 0.06

--==================================================
-- LIGHT BLOOM
--==================================================

local bloom = Lighting:FindFirstChild("DhenBloom")

if not bloom then
	bloom = Instance.new("BloomEffect")
	bloom.Name = "DhenBloom"
	bloom.Parent = Lighting
end

bloom.Intensity = 0.08
bloom.Size = 16
bloom.Threshold = 1.2

--==================================================
-- SUN RAYS
--==================================================

local sun = Lighting:FindFirstChild("DhenSunRays")

if not sun then
	sun = Instance.new("SunRaysEffect")
	sun.Name = "DhenSunRays"
	sun.Parent = Lighting
end

sun.Intensity = 0.025
sun.Spread = 0.8

--==================================================
-- RIGHT CLICK
--==================================================

UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then
		return
	end

	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		aiming = true
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		aiming = false
	end
end)

--==================================================
-- WALL CHECK
--==================================================

local function wallCheck(character, targetPart)

	local origin = camera.CFrame.Position
	local direction = targetPart.Position - origin

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude

	params.FilterDescendantsInstances = {
		player.Character
	}

	local result = workspace:Raycast(
		origin,
		direction,
		params
	)

	if not result then
		return true
	end

	return result.Instance:IsDescendantOf(character)
end

--==================================================
-- FIND TARGET
--==================================================

local function getBestTarget()

	local bestTarget = nil
	local bestDistance = FOV_RADIUS

	local center = Vector2.new(
		camera.ViewportSize.X / 2,
		camera.ViewportSize.Y / 2
	)

	for _, otherPlayer in ipairs(Players:GetPlayers()) do

		if otherPlayer == player then
			continue
		end

		local character = otherPlayer.Character

		if not character then
			continue
		end

		local humanoid =
			character:FindFirstChildOfClass("Humanoid")

		local head =
			character:FindFirstChild("Head")

		-- DEAD CHECK
		if not humanoid or humanoid.Health <= 0 then
			continue
		end

		if not head then
			continue
		end

		-- DISTANCE CHECK
		local distance =
			(head.Position - camera.CFrame.Position).Magnitude

		if distance > MAX_DISTANCE then
			continue
		end

		-- FOV CHECK
		local screenPosition, visible =
			camera:WorldToViewportPoint(head.Position)

		if not visible then
			continue
		end

		local screenPoint = Vector2.new(
			screenPosition.X,
			screenPosition.Y
		)

		local screenDistance =
			(screenPoint - center).Magnitude

		if screenDistance > FOV_RADIUS then
			continue
		end

		-- WALL CHECK
		if not wallCheck(character, head) then
			continue
		end

		if screenDistance < bestDistance then
			bestDistance = screenDistance
			bestTarget = head
		end
	end

	return bestTarget
end

--==================================================
-- AIM
--==================================================

RunService.RenderStepped:Connect(function()

	if not aiming then
		return
	end

	local target = getBestTarget()

	if not target then
		return
	end

	local targetCFrame = CFrame.lookAt(
		camera.CFrame.Position,
		target.Position
	)

	camera.CFrame = camera.CFrame:Lerp(
		targetCFrame,
		AIM_STRENGTH
	)
end)
