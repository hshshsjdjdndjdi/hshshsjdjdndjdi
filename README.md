local triggerPart = script.Parent:FindFirstChildWhichIsA("BasePart", true)
local soundTemplate = script.Parent:FindFirstChild("Lego", true)
local RAGDOLL_DURATION = 4

local jointsToRagdoll = {
	"Left Shoulder", "Right Shoulder",
	"Left Hip", "Right Hip",
	"Neck", "RootJoint"
}

local function detachMotorsR6(char)
	local storedMotors = {}

	for _, jointName in ipairs(jointsToRagdoll) do
		local motor = char:FindFirstChild(jointName, true)
		if motor and motor:IsA("Motor6D") then
			motor.Parent = nil
			table.insert(storedMotors, motor)

			local a0 = Instance.new("Attachment")
			local a1 = Instance.new("Attachment")
			a0.Name = "RagdollA0"
			a1.Name = "RagdollA1"
			a0.CFrame = motor.C0
			a1.CFrame = motor.C1
			a0.Parent = motor.Part0
			a1.Parent = motor.Part1

			local socket = Instance.new("BallSocketConstraint")
			socket.Name = "RagdollSocket"
			socket.Attachment0 = a0
			socket.Attachment1 = a1
			socket.Parent = motor.Part0
		end
	end

	return storedMotors
end

local function restoreMotorsR6(char, storedMotors)
	for _, motor in ipairs(storedMotors) do
		motor.Parent = motor.Part0
	end

	for _, obj in ipairs(char:GetDescendants()) do
		if obj:IsA("BallSocketConstraint") and obj.Name == "RagdollSocket" then
			obj:Destroy()
		elseif obj:IsA("Attachment") and (obj.Name == "RagdollA0" or obj.Name == "RagdollA1") then
			obj:Destroy()
		end
	end
end

local function setRagdoll(char, enabled, storedMotors)
	local humanoid = char:FindFirstChildOfClass("Humanoid")
	if not humanoid then return end

	if enabled then
		humanoid:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
		humanoid.PlatformStand = true
		humanoid:ChangeState(Enum.HumanoidStateType.Physics)
	else
		humanoid.PlatformStand = false
		humanoid:SetStateEnabled(Enum.HumanoidStateType.Dead, true)
		humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
	end

	for _, part in ipairs(char:GetChildren()) do
		if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
			part.CanCollide = enabled
			part.Massless = false
		end
	end

	local animate = char:FindFirstChild("Animate")
	if animate then
		animate.Disabled = enabled
	end

	local aiScript = char:FindFirstChild("AIScript")
	if aiScript then
		aiScript.Disabled = enabled
	end

	if not enabled and storedMotors then
		restoreMotorsR6(char, storedMotors)
	end
end

-- ฟังค์ชันหลักเวลาโดนชน
triggerPart.Touched:Connect(function(hit)
	local char = hit:FindFirstAncestorOfClass("Model")
	if not char or char:FindFirstChild("AlreadyRagdolled") then return end

	if not char:IsDescendantOf(workspace:FindFirstChild("ActiveEnemies")) then return end

	local humanoid = char:FindFirstChildOfClass("Humanoid")
	if not humanoid or humanoid.Health <= 0 then return end

	-- ป้องกันซ้ำซ้อน
	local tag = Instance.new("BoolValue")
	tag.Name = "AlreadyRagdolled"
	tag.Parent = char
	game:GetService("Debris"):AddItem(tag, RAGDOLL_DURATION)

	local storedMotors = detachMotorsR6(char)
	setRagdoll(char, true, storedMotors)

	-- แรงกระแทก
	local root = char:FindFirstChild("HumanoidRootPart")
	if root then
		local force = Vector3.new(math.random(-60, 60), 150, math.random(-60, 60))
		root:ApplyImpulse(force * root:GetMass())
	end

	-- เสียง
	if soundTemplate then
		local sound = soundTemplate:Clone()
		sound.Parent = workspace
		sound:Play()
		game:GetService("Debris"):AddItem(sound, 4)
	end

	-- รอให้ฟื้นตัว
	task.delay(RAGDOLL_DURATION, function()
		if humanoid and humanoid.Health > 0 then
			setRagdoll(char, false, storedMotors)

			-- เริ่มเดินใหม่
			humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
			humanoid.PlatformStand = false
		end
	end)
end)
