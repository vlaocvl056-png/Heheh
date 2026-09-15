-- ROCKET MM2 Trade Freeze v2 | Delta Executor
-- Перехват RemoteEvent через hookmetamethod

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

--// Проверка поддержки хука
if not hookmetamethod or not getnamecallmethod then
    warn("[ROCKET] Твой экзекутор не поддерживает hookmetamethod")
    return
end

--// UI
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "RocketMM2v2"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 240, 0, 170)
Frame.Position = UDim2.new(0, 20, 0, 100)
Frame.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true
Frame.Parent = ScreenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = Frame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 32)
Title.BackgroundColor3 = Color3.fromRGB(30, 30, 42)
Title.BorderSizePixel = 0
Title.Text = "ROCKET | MM2 v2"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 14
Title.Parent = Frame

local corner2 = Instance.new("UICorner")
corner2.CornerRadius = UDim.new(0, 8)
corner2.Parent = Title

local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Size = UDim2.new(0, 200, 0, 42)
ToggleBtn.Position = UDim2.new(0, 20, 0, 50)
ToggleBtn.BackgroundColor3 = Color3.fromRGB(55, 55, 70)
ToggleBtn.BorderSizePixel = 0
ToggleBtn.Text = "Freeze: OFF"
ToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleBtn.Font = Enum.Font.GothamBold
ToggleBtn.TextSize = 14
ToggleBtn.Parent = Frame

local corner3 = Instance.new("UICorner")
corner3.CornerRadius = UDim.new(0, 6)
corner3.Parent = ToggleBtn

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, -20, 0, 50)
Status.Position = UDim2.new(0, 10, 0, 105)
Status.BackgroundTransparency = 1
Status.Text = "Ожидание..."
Status.TextColor3 = Color3.fromRGB(160, 160, 170)
Status.Font = Enum.Font.Gotham
Status.TextSize = 11
Status.TextWrapped = true
Status.Parent = Frame

--// Конфигурация
local freezeEnabled = false
local originalNamecall = nil
local tradeRemote = nil
local lastTradeData = {}

--// Поиск RemoteEvent, связанного с трейдом
local function findTradeRemote()
    -- Проверяем ReplicatedStorage
    for _, obj in pairs(ReplicatedStorage:GetDescendants()) do
        if obj:IsA("RemoteEvent") then
            local nameLower = obj.Name:lower()
            if nameLower:find("trade") or nameLower:find("offer") or nameLower:find("item") then
                return obj
            end
        end
    end
    
    -- Проверяем Workspace
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("RemoteEvent") then
            local nameLower = obj.Name:lower()
            if nameLower:find("trade") or nameLower:find("offer") then
                return obj
            end
        end
    end
    
    return nil
end

--// Перехват __namecall
local function setupHook()
    originalNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
        local method = getnamecallmethod()
        local args = {...}
        
        -- Проверяем, что это FireServer и freeze активен
        if freezeEnabled and method == "FireServer" then
            -- Если self - RemoteEvent с "trade" в имени
            if typeof(self) == "Instance" and self:IsA("RemoteEvent") then
                local nameLower = self.Name:lower()
                if nameLower:find("trade") or nameLower:find("offer") or nameLower:find("item") then
                    tradeRemote = self
                    
                    -- Модифицируем аргументы: заменяем предмет на nil или пустой
                    -- Это предотвращает отправку предмета на сервер
                    for i, arg in pairs(args) do
                        if typeof(arg) == "Instance" then
                            -- Если это предмет из инвентаря - блокируем
                            if arg:IsDescendantOf(LocalPlayer) then
                                args[i] = nil
                                Status.Text = "Перехвачен предмет: " .. arg.Name
                            end
                        end
                    end
                    
                    -- Отправляем модифицированные аргументы
                    return originalNamecall(self, table.unpack(args))
                end
            end
        end
        
        return originalNamecall(self, ...)
    end))
    
    Status.Text = "Хук установлен ✓"
end

--// Кнопка
ToggleBtn.MouseButton1Click:Connect(function()
    freezeEnabled = not freezeEnabled
    
    if freezeEnabled then
        ToggleBtn.Text = "Freeze: ON"
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 140, 80)
        Status.Text = "Активно. Перехват включён."
        Status.TextColor3 = Color3.fromRGB(100, 255, 150)
    else
        ToggleBtn.Text = "Freeze: OFF"
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(55, 55, 70)
        Status.Text = "Отключено."
        Status.TextColor3 = Color3.fromRGB(160, 160, 170)
    end
end)

--// Инициализация
tradeRemote = findTradeRemote()
if tradeRemote then
    Status.Text = "Найден Remote: " .. tradeRemote.Name
else
    Status.Text = "Remote не найден, поиск динамический..."
end

setupHook()

--// Автопоиск трейд-ремоутов при появлении
ReplicatedStorage.DescendantAdded:Connect(function(obj)
    if not freezeEnabled then return end
    if obj:IsA("RemoteEvent") then
        local nameLower = obj.Name:lower()
        if nameLower:find("trade") or nameLower:find("offer") then
            tradeRemote = obj
            Status.Text = "Новый Remote: " .. obj.Name
        end
    end
end)

print("[ROCKET] MM2 Trade Freeze v2 загружен. Хук через __namecall активен.")
