-------------------------------------------------------
-- AutoBankRob
-------------------------------------------------------
local AutoBankRob = {
    running = false,
    terminated = false,
    moneyBoxCount = 0,
    firstButtonClicked = false,
}

local function log(msg) print("[AutoBankRob] " .. tostring(msg)) end
local function waitForHRP(char) return char:WaitForChild("HumanoidRootPart", 10) end
local character = player.Character or player.CharacterAdded:Wait()
local humanoidRootPart = waitForHRP(character)
player.CharacterAdded:Connect(function(newChar)
    humanoidRootPart = waitForHRP(newChar)
end)

local function teleportPlayer(pos)
    if humanoidRootPart and humanoidRootPart.Parent then
        humanoidRootPart.CFrame = CFrame.new(pos)
    end
end

local function forceHoldDurationZero()
    local bank = workspace:FindFirstChild("BANK")
    if not bank then return end
    local folder = bank:FindFirstChild("MoneyBoxes")
    if not folder then return end
    for _, box in ipairs(folder:GetChildren()) do
        local part = box:IsA("BasePart") and box or box:FindFirstChildWhichIsA("BasePart")
        if not part then
            for _, d in ipairs(box:GetDescendants()) do
                if d:IsA("BasePart") then part = d break end
            end
        end
        if part then
            local prompt = box:FindFirstChildWhichIsA("ProximityPrompt") or part:FindFirstChildWhichIsA("ProximityPrompt")
            if prompt then pcall(function() prompt.HoldDuration = 0 end) end
        end
    end
end

local function activatePrompt(prompt)
    if not prompt or not prompt:IsA("ProximityPrompt") or not prompt.Enabled then return false end
    task.wait(0.15)
    pcall(function()
        prompt:InputHoldBegin()
        task.wait(0.05)
        prompt:InputHoldEnd()
    end)
    return true
end

local function interactWithBankButton()
    if AutoBankRob.firstButtonClicked then return true end
    local bank = workspace:FindFirstChild("BANK")
    if not bank then return false end
    local buttonPanel = bank:FindFirstChild("ButtonPanel")
    if not buttonPanel then return false end
    local promptPart = buttonPanel:FindFirstChild("PromptPart")
    if not promptPart then return false end
    local prompt = promptPart:FindFirstChild("ProximityPrompt")

    local targetPos = promptPart.Position
    teleportPlayer(targetPos + Vector3.new(0, 3, 0))
    task.wait(1)

    if not prompt or not prompt.Enabled then
        teleportPlayer(targetPos + Vector3.new(0, 0, 15))
        AutoBankRob.firstButtonClicked = true
        return true
    end

    activatePrompt(prompt)
    local timeout = 0
    while prompt.Enabled and timeout < 5 do
        task.wait(0.1)
        timeout += 0.1
    end
    AutoBankRob.firstButtonClicked = true
    task.wait(1)
    teleportPlayer(targetPos + Vector3.new(0, 0, 15))
    return true
end

local function getClaimableBoxes()
    local bank = workspace:FindFirstChild("BANK")
    if not bank then return {} end
    local folder = bank:FindFirstChild("MoneyBoxes")
    if not folder then return {} end

    local claimable = {}
    for _, box in ipairs(folder:GetChildren()) do
        local part = box:IsA("BasePart") and box or box:FindFirstChildWhichIsA("BasePart")
        if not part then
            for _, d in ipairs(box:GetDescendants()) do
                if d:IsA("BasePart") then part = d break end
            end
        end
        local prompt = box:FindFirstChildWhichIsA("ProximityPrompt") or (part and part:FindFirstChildWhichIsA("ProximityPrompt"))
        if prompt and prompt.Enabled then table.insert(claimable, box) end
    end
    return claimable
end

local function autoRobAllMoneyBoxes()
    if AutoBankRob.terminated or not AutoBankRob.running then return end
    forceHoldDurationZero()
    if not AutoBankRob.firstButtonClicked then interactWithBankButton() end

    for _, box in ipairs(getClaimableBoxes()) do
        if not AutoBankRob.running or AutoBankRob.terminated then break end
        local part = box:IsA("BasePart") and box or box:FindFirstChildWhichIsA("BasePart")
        if not part then
            for _, d in ipairs(box:GetDescendants()) do
                if d:IsA("BasePart") then part = d break end
            end
        end
        local prompt = box:FindFirstChildWhichIsA("ProximityPrompt") or (part and part:FindFirstChildWhichIsA("ProximityPrompt"))
        if prompt and prompt.Enabled then
            teleportPlayer(part.Position + Vector3.new(0, 3, 0))
            activatePrompt(prompt)
            AutoBankRob.moneyBoxCount += 1
            log("Robbed a MoneyBox. Total: " .. AutoBankRob.moneyBoxCount)
        end
    end
end

task.spawn(function()
    while not AutoBankRob.terminated do
        if AutoBankRob.running then autoRobAllMoneyBoxes() end
        task.wait(0.25)
    end
end)
