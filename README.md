# Brainrot-hunter-vip-v3.1
local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- НАСТРОЙКИ
local WEBHOOK_URL = "https://discord.com/api/webhooks/1401072876995608616/Gd2lPRs2ErPcy09BMufxBmOhPe9-eoMpuDEKb1qysizmmyZ6g11oRJqLGdzGLQEwAt8k"
local MIN_PRICE = 1000000
local HOP_INTERVAL = 5
local MAX_HISTORY = 20

-- СОСТОЯНИЕ
local isRunning = false
local hopConnection = nil
local stats = {
    serversVisited = 0,
    brainrotsFound = 0
}
local serverHistory = {}
local blacklist = {}

-- СОЗДАНИЕ УСТОЙЧИВОГО ИНТЕРФЕЙСА
local screenGui = Instance.new("ScreenGui", CoreGui)
screenGui.Name = "BrainrotHunterPro"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local mainFrame = Instance.new("Frame", screenGui)
mainFrame.Size = UDim2.new(0, 320, 0, 220)
mainFrame.Position = UDim2.new(0.5, -160, 0.5, -110)
mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
mainFrame.BorderSizePixel = 0

local UICorner = Instance.new("UICorner", mainFrame)
UICorner.CornerRadius = UDim.new(0, 8)

local titleBar = Instance.new("Frame", mainFrame)
titleBar.Size = UDim2.new(1, 0, 0, 32)
titleBar.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
titleBar.BorderSizePixel = 0

local title = Instance.new("TextLabel", titleBar)
title.Size = UDim2.new(1, -40, 1, 0)
title.Text = "🧠 BRAINROT HUNTER PRO"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 14
title.TextXAlignment = Enum.TextXAlignment.Left
title.Position = UDim2.new(0, 10, 0, 0)
title.BackgroundTransparency = 1

local closeBtn = Instance.new("TextButton", titleBar)
closeBtn.Size = UDim2.new(0, 32, 0, 32)
closeBtn.Position = UDim2.new(1, -32, 0, 0)
closeBtn.Text = "×"
closeBtn.TextSize = 20
closeBtn.BackgroundTransparency = 1
closeBtn.TextColor3 = Color3.fromRGB(200, 200, 200)

local contentFrame = Instance.new("Frame", mainFrame)
contentFrame.Size = UDim2.new(1, -20, 1, -52)
contentFrame.Position = UDim2.new(0, 10, 0, 42)
contentFrame.BackgroundTransparency = 1

-- Элементы управления
local statusLabel = Instance.new("TextLabel", contentFrame)
statusLabel.Size = UDim2.new(1, 0, 0, 20)
statusLabel.Text = "Status: Ready"
statusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
statusLabel.Font = Enum.Font.Gotham
statusLabel.TextXAlignment = Enum.TextXAlignment.Left

local toggleBtn = Instance.new("TextButton", contentFrame)
toggleBtn.Size = UDim2.new(1, 0, 0, 40)
toggleBtn.Position = UDim2.new(0, 0, 0, 30)
toggleBtn.Text = "▶ START AUTO HOP"
toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 140, 60)
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(0, 6)

local statsLabel = Instance.new("TextLabel", contentFrame)
statsLabel.Size = UDim2.new(1, 0, 0, 40)
statsLabel.Position = UDim2.new(0, 0, 0, 80)
statsLabel.Text = "Servers: 0\nFinds: 0"
statsLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
statsLabel.TextXAlignment = Enum.TextXAlignment.Left
statsLabel.TextYAlignment = Enum.TextYAlignment.Top

-- ПЕРЕТАСКИВАНИЕ ОКНА
local dragging, dragInput, dragStart

titleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

titleBar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        local delta = input.Position - dragStart
        mainFrame.Position = UDim2.new(
            mainFrame.Position.X.Scale,
            mainFrame.Position.X.Offset + delta.X,
            mainFrame.Position.Y.Scale,
            mainFrame.Position.Y.Offset + delta.Y
        )
    end
end)

-- ОСНОВНЫЕ ФУНКЦИИ
local function updateStats()
    statsLabel.Text = string.format("Servers: %d\nFinds: %d", stats.serversVisited, stats.brainrotsFound)
end

local function sendToDiscord(brain)
    local embed = {
        title = "💰 ДОРОГОЙ БРЕЙНРОТ НАЙДЕН!",
        description = string.format(
            "**Сервер:** [%s](https://www.roblox.com/games/%s?gameInstanceId=%s)\n"..
            "**Юнит:** %s\n"..
            "**Цена:** %s/s",
            game.JobId,
            game.PlaceId,
            game.JobId,
            brain.Name,
            brain.PricePerSecond.Value
        ),
        color = 65280,
        footer = {text = os.date("%d.%m.%Y %H:%M:%S")}
    }
    
    pcall(function()
        HttpService:PostAsync(WEBHOOK_URL, HttpService:JSONEncode({
            content = "@here",
            embeds = {embed}
        }))
    end)
end

local function getBestServer()
    local url = string.format("https://games.roblox.com/v1/games/%s/servers/Public?sortOrder=Asc&limit=100", game.PlaceId)
    local success, response = pcall(function()
        return game:HttpGet(url, true)
    end)
    
    if success then
        local data = HttpService:JSONDecode(response)
        if data and data.data then
            for _, server in ipairs(data.data) do
                if server.id ~= game.JobId 
                   and not blacklist[server.id] 
                   and not serverHistory[server.id] 
                   and server.playing < server.maxPlayers then
                    return server.id
                end
            end
        end
    end
    return nil
end

local function hopToBestServer()
    local serverId = getBestServer()
    if not serverId then return false end
    
    serverHistory[serverId] = true
    if table.getn(serverHistory) > MAX_HISTORY then
        table.remove(serverHistory, 1)
    end
    
    stats.serversVisited += 1
    updateStats()
    
    TeleportService:TeleportToPlaceInstance(game.PlaceId, serverId)
    return true
end

local function scanForBrainrots()
    local brains = workspace:FindFirstChild("Brains") or workspace:FindFirstChild("Мозги")
    if not brains then return false end
    
    for _, brain in pairs(brains:GetChildren()) do
        if brain:FindFirstChild("PricePerSecond") and brain.PricePerSecond.Value >= MIN_PRICE then
            sendToDiscord(brain)
            stats.brainrotsFound += 1
            updateStats()
            return true
        end
    end
    return false
end

-- АВТОХОП СИСТЕМА
local function startAutoHop()
    if isRunning then return end
    isRunning = true
    
    toggleBtn.Text = "⏹ STOP HOPPING"
    toggleBtn.BackgroundColor3 = Color3.fromRGB(140, 60, 60)
    statusLabel.Text = "Searching servers..."
    
    hopConnection = RunService.Heartbeat:Connect(function()
        if not scanForBrainrots() then
            local success = hopToBestServer()
            statusLabel.Text = success and "Hopping to new server..." or "No servers found"
            task.wait(HOP_INTERVAL)
        else
            statusLabel.Text = "Found brainrot! Waiting..."
            task.wait(HOP_INTERVAL * 2)
        end
    end)
end

local function stopAutoHop()
    if not isRunning then return end
    isRunning = false
    
    toggleBtn.Text = "▶ START AUTO HOP"
    toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 140, 60)
    statusLabel.Text = "Stopped"
    
    if hopConnection then
        hopConnection:Disconnect()
        hopConnection = nil
    end
end

-- ОБРАБОТЧИКИ
toggleBtn.MouseButton1Click:Connect(function()
    if isRunning then
        stopAutoHop()
    else
        startAutoHop()
    end
end)

closeBtn.MouseButton1Click:Connect(function()
    stopAutoHop()
    screenGui:Destroy()
end)

-- ЗАЩИТА ОТ ОШИБОК
coroutine.wrap(function()
    while true do
        if not mainFrame.Parent then
            stopAutoHop()
            break
        end
        task.wait(5)
    end
end)()
