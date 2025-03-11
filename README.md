-- Tải Fluent UI Library
local Library = loadstring(game:HttpGetAsync("https://github.com/ActualMasterOogway/Fluent-Renewed/releases/latest/download/Fluent.luau"))()
-- Âm thanh khởi động
local startupSound = Instance.new("Sound")
startupSound.SoundId = "rbxassetid://81720963182969"
startupSound.Volume = 5 -- Điều chỉnh âm lượng nếu cần
startupSound.Looped = false -- Không lặp lại âm thanh
startupSound.Parent = game.CoreGui-- Đặt parent vào CoreGui để đảm bảo âm thanh phát
startupSound:Play() -- Phát âm thanh khi script chạy
-- Tạo cửa sổ chính
local Window = Library:CreateWindow{
    Title = "Spock Hub version find volcano",
    SubTitle = "https://discord.gg/EewSkNM6",
    TabWidth = 160,
    Size = UDim2.fromOffset(1280, 860),
    Resize = true,
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.RightControl -- Phím thu nhỏ
}

-- Tạo ScreenGui chứa nút điều khiển
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ControlGUI"
screenGui.Parent = game.CoreGui

-- Tạo nút (ImageButton)
local toggleButton = Instance.new("ImageButton")
toggleButton.Size = UDim2.new(0, 50, 0, 50) -- Kích thước nhỏ, hình vuông
toggleButton.Position = UDim2.new(1, -60, 0, 10) -- Vị trí gần góc phải trên
toggleButton.Image = "rbxassetid://83756310740222" -- Hình ảnh của nút
toggleButton.BackgroundTransparency = 1 -- Không có nền
toggleButton.Parent = screenGui

-- Biến lưu trạng thái hiển thị Fluent UI
local isFluentVisible = true

-- Di chuyển nút
local dragging, dragInput, dragStart, startPos

local function update(input)
    local delta = input.Position - dragStart
    toggleButton.Position = UDim2.new(
        startPos.X.Scale,
        startPos.X.Offset + delta.X,
        startPos.Y.Scale,
        startPos.Y.Offset + delta.Y
    )
end

toggleButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = toggleButton.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

toggleButton.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

game:GetService("UserInputService").InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        update(input)
    end
end)

-- Ẩn/Hiện Fluent UI khi nhấn nút
toggleButton.MouseButton1Click:Connect(function()
    isFluentVisible = not isFluentVisible

    if isFluentVisible then
        -- Hiện Fluent UI
        Window:Minimize(false) -- Mở lại cửa sổ
    else
        -- Ẩn Fluent UI
        Window:Minimize(true) -- Thu nhỏ cửa sổ
    end
end)

-- Tạo các tab
local MainTab = Window:AddTab({ Title = "Main", Icon = "" })
local PlayerTab = Window:AddTab({ Title = "Player", Icon = "" })
local IslandTab = Window:AddTab({ Title = "Island", Icon = "" })
local OtherTab = Window:AddTab({ Title = "Other", Icon = "" })
local FruitTab = Window:AddTab({ Title = "Fruit", Icon = "" }) -- Tab Fruit mới

-- Biến toàn cục
local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer
local Fluent = Library -- Gán biến Fluent

local currentTween = nil
local leviathanGateConnection = nil
local rockConnection = nil -- No longer used for DescendantAdded
local scriptEnabled = false
local lockHeightMode = true
local destroyRockMode = false
local customY = 100
local tweenSpeed = 350
local minimized = false
local lastBoat, lastSeat = nil, nil
local playerTweening = false -- Khôi phục biến playerTweening
local autoReturnEnabled = false
local killAuraEnabled = false
local killAuraRange = 100
local selectedIslandCoord = nil
local islandTweening = false
local boatProtectEnabled = false -- Bảo vệ thuyền
local boatProtectImmediateEnabled = false -- Bảo vệ thuyền ngay lập tức
local boatProtectTween = nil
local boatProtectConnection = nil
local originalBoatProtectY = nil
local currentBoat = nil
_G.Random_Auto = false -- Auto Random Fruit
local selectedPlayer = nil -- Biến lưu player được chọn cho Spectate và Tween Player
local playerTweenInfo = nil
local rockTouchConnection = nil -- Biến connection cho sự kiện chạm đá

-- Danh sách đảo
local islands = {
    Tiki = Vector3.new(-16208.6, 10.4, 500.9),
    Hydra = Vector3.new(5164.5, 48.1, 2094.1),
    ["Pháo đài"] = Vector3.new(-5122.9, 315.7, -2961.9),
    ["Dinh Thự"] = Vector3.new(-12551.5, 338.5, -7552.3),
    ["Lâu Đài Bóng Tối"] = Vector3.new(-9508.9, 158.4, 4894.1),
    ["Cảng"] = Vector3.new(-221.3, 22.3, 5517.8),
    ["Cây Đại Thụ"] = Vector3.new(2130.8, 23.1, -6635.9),
    ["Đảo Bánh"] = Vector3.new(-1928.6, 15.1, -11582.4)
}

-- Hàm hỗ trợ
local function setBoatNoClip(boat, enable)
    for _, desc in ipairs(boat:GetDescendants()) do
        if desc:IsA("BasePart") then
            desc.CanCollide = not enable
        end
    end
end

local function setPlayerNoClip(char, enable)
    for _, desc in ipairs(char:GetDescendants()) do
        if desc:IsA("BasePart") then
            desc.CanCollide = not enable
        end
    end
end

local function getBoatVehicleSeat()
    local boatsFolder = Workspace:FindFirstChild("Boats")
    if not boatsFolder then return nil, nil end

    for _, boat in ipairs(boatsFolder:GetChildren()) do
        local seat = boat:FindFirstChild("VehicleSeat")
        if seat and seat.Occupant and seat.Occupant.Parent == LocalPlayer.Character then
            return boat, seat
        end
    end
    return nil, nil
end

local function stopTween(reason)
    if currentTween then
        currentTween:Cancel()
        currentTween = nil
        print(reason or "Tween dừng.")
    end
    if leviathanGateConnection then
        leviathanGateConnection:Disconnect()
        leviathanGateConnection = nil
    end
    if playerTweenInfo then
        playerTweenInfo:Cancel()
        playerTweenInfo = nil
    end
    scriptEnabled = false
    playerTweening = false
    islandTweening = false
    local boat, _ = getBoatVehicleSeat()
    if boat then
        setBoatNoClip(boat, false)
    end
    if LocalPlayer.Character then
        setPlayerNoClip(LocalPlayer.Character, false)
    end
    stopBoatProtect() -- Dừng bảo vệ thuyền khi tween dừng
    setRockDestruction(false) -- Ensure rock destruction is off when tween is stopped
end

local function stopIfMapObjectAppears(child)
    if child.Name == "LeviathanGate" then
        stopTween("LeviathanGate xuất hiện.")
    elseif child.Name == "PrehistoricIsland" then
        stopTween("PrehistoricIsland xuất hiện.")
    end
end

local function checkLeaveBoat()
    local _, seat = getBoatVehicleSeat()
    if not seat then
        stopTween("Bạn đã rời thuyền.")
    end
end

local function tweenBoatToVector3(targetPos)
    -- Nếu đang tween => dừng cũ
    if currentTween then
        stopTween("Dừng tween cũ.")
    end
    local boat, seat = getBoatVehicleSeat()
    if not boat then
        warn("Không tìm thấy thuyền bạn đang ngồi!")
        return
    end
    lastBoat, lastSeat = boat, seat
    currentBoat = boat -- Cập nhật thuyền hiện tại cho bảo vệ thuyền

    local primary = boat.PrimaryPart
    if not primary then
        warn("Thuyền không có PrimaryPart!")
        return
    end

    -- Bật no clip
    setBoatNoClip(boat, true)
    if LocalPlayer.Character then
        setPlayerNoClip(LocalPlayer.Character, true)
    end

    local currentPos = primary.Position
    local finalY = lockHeightMode and customY or currentPos.Y
    if lockHeightMode then
        primary.CFrame = CFrame.new(currentPos.X, finalY, currentPos.Z)
    end

    local distance = (Vector3.new(targetPos.X, finalY, targetPos.Z) - primary.Position).Magnitude
    local tweenTime = distance / tweenSpeed

    local tweenInfo = TweenInfo.new(tweenTime, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
    local goal = { CFrame = CFrame.new(targetPos.X, finalY, targetPos.Z) }

    currentTween = TweenService:Create(primary, tweenInfo, goal)
    currentTween:Play()

    task.spawn(function()
        while currentTween and currentTween.PlaybackState == Enum.PlaybackState.Playing do
            checkLeaveBoat()
            task.wait(0.1)
        end
        -- Khi tween xong => tắt no clip
        setBoatNoClip(boat, false)
        if LocalPlayer.Character then
            setPlayerNoClip(LocalPlayer.Character, false)
        end
        islandTweening = false
        currentBoat = nil -- Reset thuyền hiện tại khi tween xong
    end)

    -- Theo dõi LeviathanGate/PrehistoricIsland
    if leviathanGateConnection then
        leviathanGateConnection:Disconnect()
    end
    leviathanGateConnection = Workspace:WaitForChild("Map").ChildAdded:Connect(stopIfMapObjectAppears)
end

-- Hàm Tween thuyền đến player
local function tweenBoatToPlayer(targetPlayer)
    if playerTweening then
        warn("Đang Tween đến player khác, hãy STOP trước.")
        return
    end
    playerTweening = true

    -- Nếu đang tween => dừng cũ
    if currentTween then
        stopTween("Dừng tween cũ.")
    end

    local boat, seat = getBoatVehicleSeat()
    if not boat then
        warn("Không tìm thấy thuyền đang ngồi!")
        playerTweening = false
        return
    end
    lastBoat, lastSeat = boat, seat

    local primary = boat.PrimaryPart
    if not primary then
        warn("Thuyền không có PrimaryPart!")
        playerTweening = false
        return
    end

    -- Bật no clip
    setBoatNoClip(boat, true)
    if LocalPlayer.Character then
        setPlayerNoClip(LocalPlayer.Character, true)
    end

    local char = targetPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then
        warn("Player chưa load xong hoặc không có HRP!")
        playerTweening = false
        -- Tắt no clip
        setBoatNoClip(boat, false)
        if LocalPlayer.Character then
            setPlayerNoClip(LocalPlayer.Character, false)
        end
        return
    end

    local playerPos = char.HumanoidRootPart.Position
    local currentPos = primary.Position
    local finalY = lockHeightMode and customY or currentPos.Y
    if lockHeightMode then
        primary.CFrame = CFrame.new(currentPos.X, finalY, currentPos.Z)
    end

    local distance = (Vector3.new(playerPos.X, finalY, playerPos.Z) - primary.Position).Magnitude
    local tweenTime = distance / tweenSpeed

    local tweenInfo = TweenInfo.new(tweenTime, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
    local goal = { CFrame = CFrame.new(playerPos.X, finalY, playerPos.Z) }

    playerTweenInfo = TweenService:Create(primary, tweenInfo, goal)
    playerTweenInfo:Play()

    task.spawn(function()
        while playerTweenInfo and playerTweenInfo.PlaybackState == Enum.PlaybackState.Playing do
            checkLeaveBoat()
            task.wait(0.1)
        end
        -- Tắt no clip
        setBoatNoClip(boat, false)
        if LocalPlayer.Character then
            setPlayerNoClip(LocalPlayer.Character, false)
        end
        playerTweening = false
    end)

    if leviathanGateConnection then
        leviathanGateConnection:Disconnect()
    end
    leviathanGateConnection = Workspace:WaitForChild("Map").ChildAdded:Connect(stopIfMapObjectAppears)
end

-- Hàm Auto Return
local function autoReturnCheck()
    while autoReturnEnabled do
        if lastBoat and lastSeat and lastBoat.Parent then
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                local seatPos = lastSeat.Position
                local distance = (hrp.Position - seatPos).Magnitude
                if distance > 10 then
                    local speed = 300
                    local time = distance / speed

                    local info = TweenInfo.new(time, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
                    local goal = { CFrame = CFrame.new(seatPos + Vector3.new(0,3,0)) }
                    local tween = TweenService:Create(hrp, info, goal)
                    tween:Play()
                end
            end
        end
        task.wait(1)
    end
end

-- Hàm Kill Aura
local function killAuraLoop()
    local monstersFolder = Workspace:FindFirstChild("Enemies")
    while killAuraEnabled do
        task.wait(1)
        local char = LocalPlayer.Character
        if not char then continue end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp or not monstersFolder then continue end

        for _, monster in pairs(monstersFolder:GetChildren()) do
            local hum = monster:FindFirstChild("Humanoid")
            local mrp = monster:FindFirstChild("HumanoidRootPart")
            if hum and mrp and hum.Health > 0 then
                local distance = (mrp.Position - hrp.Position).Magnitude
                if distance <= killAuraRange then
                    hum.Health = 0
                end
            end
        end
    end
end

-- Hàm Anti Die
local antiDieEnabled = false
local originalPosition = nil
local teleporting = false
local function teleportPlayerUp()
    while antiDieEnabled and teleporting do
        local character = LocalPlayer.Character
        if character and character:FindFirstChild("HumanoidRootPart") then
            local hrp = character.HumanoidRootPart
            hrp.CFrame = hrp.CFrame + Vector3.new(0, 200, 0) -- Teleport lên 200 studs
        end
        task.wait(0.1) -- Lặp lại sau mỗi 0.1 giây
    end
end

local function checkHealth()
    while antiDieEnabled do
        local character = LocalPlayer.Character
        if character and character:FindFirstChild("Humanoid") then
            local humanoid = character.Humanoid
            if humanoid.Health == 0 then
                teleporting = false
            elseif humanoid.Health <= (humanoid.MaxHealth * 0.3) then
                if not teleporting then
                    teleporting = true
                    originalPosition = character.HumanoidRootPart.Position -- Lưu vị trí cũ
                    teleportPlayerUp() -- Bắt đầu teleport lên trời
                end
            else
                if teleporting then
                    if originalPosition then
                        local hrp = character:FindFirstChild("HumanoidRootPart")
                        if hrp then
                            hrp.CFrame = CFrame.new(originalPosition)
                        end
                        originalPosition = nil
                    end
                    teleporting = false -- Dừng teleport nếu HP > 30%
                end
            end
        end
        task.wait(0.1) -- Kiểm tra HP mỗi 0.1 giây
    end
end

-- Hàm DashNoCD
_G.DodgewithoutCool = false
local dashNoCDEnabled = false
local function NoCooldown()
    if _G.DodgewithoutCool then
        local player = game.Players.LocalPlayer
        local character = player.Character or player.CharacterAdded:Wait() -- Chờ nhân vật được load.

        -- Đảm bảo chờ 1 giây sau khi respawn
        wait(3)

        for i, v in next, getgc() do
            if typeof(v) == "function" then
                if getfenv(v).script == character:WaitForChild("Dodge") then
                    for i2, v2 in next, getupvalues(v) do
                        if tostring(v2) == "0.4" then -- Giá trị cooldown gốc.
                            repeat
                                wait(0.1) -- Kiểm tra cooldown liên tục.
                                setupvalue(v, i2, 0) -- Đặt cooldown thành 0.
                            until not _G.DodgewithoutCool
                        end
                    end
                end
            end
        end
    end
end

-- Hàm WalkSpeed
local hb = game:GetService("RunService").Heartbeat
local speaker = game:GetService("Players").LocalPlayer
local tpwalking = false
local currentSpeed = 90  -- Mặc định ban đầu là 90
local walkSpeedEnabled = false
local function startTeleportWalk(character)
    local hum = character:WaitForChild("Humanoid")

    -- Lắng nghe sự kiện thay đổi máu
    hum.HealthChanged:Connect(function(health)
        local maxHealth = hum.MaxHealth
        local hpPercent = (health / maxHealth) * 100
        if hpPercent <= 30 then
            currentSpeed = 190
        elseif hpPercent >= 50 then
            currentSpeed = 90
        end
        -- Nếu muốn có thêm logic cho mức 30%-50% thì thêm vào đây
    end)

    while tpwalking and hum and hum.Parent do
        local delta = hb:Wait()
        if hum.MoveDirection.Magnitude > 0 then
            character:TranslateBy(hum.MoveDirection * currentSpeed * delta)
        end
    end
end

-- Lắng nghe sự kiện nhân vật tái sinh
speaker.CharacterAdded:Connect(function(character)
    if tpwalking then
        startTeleportWalk(character)
    end
end)

-- Nếu nhân vật đã tồn tại, chạy ngay lập tức
if speaker.Character and tpwalking then
    startTeleportWalk(speaker.Character)
end

-- Hàm Gas
local Players = game:GetService("Players")
local StarterGui = game:GetService("StarterGui")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local toggleNoCooldown = false
local gasEnabled = false

-- Hàm hiển thị thông báo
local function notify(title, text)
    StarterGui:SetCore("SendNotification", {
        Title = spock hub ben loaded;
        Text = spock hub on top;
        Duration = 3; -- Thời gian hiển thị thông báo
    })
end

-- Hàm tạo Tool và cài đặt toggle
local function createNoCooldownTool()
    local backpack = player:FindFirstChild("Backpack")
    if not backpack then
        return
    end

    -- Kiểm tra nếu Tool đã tồn tại, nếu có thì xóa trước
    local oldTool = backpack:FindFirstChild("NoCooldownToggle")
    if oldTool then
        oldTool:Destroy()
    end

    -- Tạo Tool
    local tool = Instance.new("Tool")
    tool.Name = "NoCooldownToggle"
    tool.RequiresHandle = false
    tool.CanBeDropped = false
    tool.Parent = backpack

    -- Khi Tool được nhấn (Activated), bật/tắt tính năng NoCooldown
    tool.Activated:Connect(function()
        toggleNoCooldown = not toggleNoCooldown
        if toggleNoCooldown then
            notify("No Cooldown M1", "Enabled")
        else
            notify("No Cooldown M1", "Disabled")
        end
    end)
end

-- Gọi hàm spawn để thực thi liên tục kiểm tra NoCooldown
task.spawn(function()
    while task.wait(0.1) do
        if toggleNoCooldown then
            local backpack = player:FindFirstChild("Backpack")
            local character = player.Character
            if not backpack or not character then
                continue
            end

            local tool = backpack:FindFirstChild("Gas-Gas") or character:FindFirstChild("Gas-Gas")
            if tool then
                -- Nhấn chuột trái tấn công
                local leftClickRemote = tool:FindFirstChild("LeftClickRemote")
                if leftClickRemote then
                    local hrp = character:FindFirstChild("HumanoidRootPart")
                    local direction = hrp and hrp.CFrame.LookVector or Vector3.new(0, 0, -1)
                    local args = {
                        [1] = direction, -- Hướng phía trước
                        [2] = 4         -- Sức mạnh hoặc tầm đánh
                    }
                    leftClickRemote:FireServer(unpack(args))
                end

                -- Gửi sự kiện tắt cooldown
                local remoteEvent = tool:FindFirstChild("RemoteEvent")
                if remoteEvent then
                    local args = {
                        [1] = false -- Vô hiệu hóa cooldown
                    }
                    remoteEvent:FireServer(unpack(args))
                end
            end
        end
    end
end)

-- Find Fruit
local fruitsESP = {}
local findFruitEnabled = false
local function createESP(fruit)
    if not fruit:FindFirstChild("Handle") then return end

    if fruitsESP[fruit] then
        fruitsESP[fruit]:Destroy()
        fruitsESP[fruit] = nil
    end

    local billboard = Instance.new("BillboardGui")
    billboard.Adornee = fruit.Handle
    billboard.Size = UDim2.new(0, 200, 0, 50)
    billboard.StudsOffset = Vector3.new(0, 2, 0)
    billboard.AlwaysOnTop = true

    local label = Instance.new("TextLabel", billboard)
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = fruit.Name
    label.TextColor3 = Color3.fromRGB(255, 255, 0)
    label.TextScaled = true

    billboard.Parent = fruit
    fruitsESP[fruit] = billboard
end

local function findFruits()
    local fruits = {}
    for _, object in pairs(workspace:GetChildren()) do
        if object:IsA("Tool") and object.Name:lower():find("fruit") then
            table.insert(fruits, object)
            if findFruitEnabled then
                createESP(object)
            end
        end
    end
    return fruits
end

local function teleportToFruit(fruit)
    if fruit and fruit:FindFirstChild("Handle") then
        if findFruitEnabled then
            LocalPlayer.Character.HumanoidRootPart.CFrame = fruit.Handle.CFrame
        end
    end
end

local runService = game:GetService("RunService")
runService.Heartbeat:Connect(function()
    local fruits = findFruits()
    if #fruits > 0 then
        for _, fruit in pairs(fruits) do
            if fruit.Parent == workspace then
                teleportToFruit(fruit)
                wait(1)
            end
        end
    end
end)

workspace.ChildAdded:Connect(function(child)
    if child:IsA("Tool") and child.Name:lower():find("fruit") then
        wait(0.5)
        if findFruitEnabled then
            createESP(child)
        end
        teleportToFruit(child)
    end
end)

workspace.ChildRemoved:Connect(function(child)
    if fruitsESP[child] then
        fruitsESP[child]:Destroy()
        fruitsESP[child] = nil
    end
end)

-- Boat Protect
local function tweenBoatToProtectHeight(boat)
    local primary = boat.PrimaryPart
    if not primary then
        warn("Thuyền không có PrimaryPart để tween bảo vệ!")
        return
    end

    local currentPos = primary.Position
    local targetY = 200 -- Mặc định là 200
    if lockHeightMode then -- Nếu bật khóa độ cao
        local numY = tonumber(customY)
        if numY then
            targetY = numY -- Sử dụng giá trị Y đã nhập nếu hợp lệ
        end
    end

    if not originalBoatProtectY then
        originalBoatProtectY = currentPos.Y
    end

    local distance = math.abs(targetY - currentPos.Y)
    local tweenTime = distance / tweenSpeed

    local tweenInfo = TweenInfo.new(tweenTime, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
    local goal = { CFrame = CFrame.new(currentPos.X, targetY, currentPos.Z) }

    if boatProtectTween then
        boatProtectTween:Cancel()
        boatProtectTween = nil
    end

    setBoatNoClip(boat, true)
    if LocalPlayer.Character then
        setPlayerNoClip(LocalPlayer.Character, true)
    end

    boatProtectTween = TweenService:Create(primary, tweenInfo, goal)
    boatProtectTween:Play()

    local startY = currentPos.Y
    task.spawn(function()
        while boatProtectTween and boatProtectTween.PlaybackState == Enum.PlaybackState.Playing do
            task.wait(0.1)
            if boat and primary then
                local newCurrentPos = primary.Position
                local newTargetY = targetY
                if lockHeightMode then -- Update targetY during tween if lockHeightMode is on
                    local numY = tonumber(customY)
                    if numY then
                        newTargetY = numY
                    end
                end
                primary.CFrame = CFrame.new(newCurrentPos.X, newTargetY, newCurrentPos.Z)
            end
        end
        if not boatProtectImmediateEnabled then
            setBoatNoClip(boat, false)
            if LocalPlayer.Character then
                setPlayerNoClip(LocalPlayer.Character, false)
            end
        end
    end)
end

local function checkBoatHealth()
    while boatProtectEnabled do
        task.wait(1)
        local boat = currentBoat
        if not boat then
            boat, _ = getBoatVehicleSeat()
            currentBoat = boat
        end
        if boat then
            local primary = boat.PrimaryPart
            if primary then
                local currentPos = primary.Position
                local boatHealth = 100 -- Máu thuyền (có thể cần điều chỉnh logic lấy máu thuyền nếu có)
                local char = LocalPlayer.Character
                if char then
                    local humanoid = char:FindFirstChild("Humanoid")
                    if humanoid then
                        boatHealth = humanoid.Health -- Tạm thời dùng máu người chơi làm máu thuyền
                    end
                end

                local currentY = primary.Position.Y
                local targetY = 200
                if lockHeightMode then
                    local numY = tonumber(customY)
                    if numY then
                        targetY = numY
                    else
                        targetY = 200 -- Default to 200 if customY is not valid and lockHeightMode is on
                    end
                end

                if boatHealth <= 50 or (originalBoatProtectY and math.abs(currentY - originalBoatProtectY) > 10 ) then
                    if not boatProtectImmediateEnabled then
                        tweenBoatToProtectHeight(boat)
                    end
                end
            end
        else
            currentBoat = nil
        end
    end
end

local function boatProtectImmediateLoop()
    while boatProtectImmediateEnabled do
        task.wait(1)
        local boat = currentBoat
        if not boat then
            boat, _ = getBoatVehicleSeat()
            currentBoat = boat
        end
        if boat then
            tweenBoatToProtectHeight(boat)
        else
            currentBoat = nil
        end
    end
end

-- Function to stop boat protect
local function stopBoatProtect()
    if boatProtectTween then
        boatProtectTween:Cancel()
        boatProtectTween = nil
    end
    local boat = currentBoat
    if boat then
        setBoatNoClip(boat, false)
        if LocalPlayer.Character then
            setPlayerNoClip(LocalPlayer.Character, false)
        end
    end

    if boatProtectConnection then
        boatProtectConnection:Disconnect()
        boatProtectConnection = nil
    end

    currentBoat = nil
    originalBoatProtectY = nil
end

-- Rock Destruction Logic
local rockDestructionLoop = nil -- Biến để lưu task loop phá đá

local function setRockDestruction(enabled)
    destroyRockMode = enabled -- Cập nhật trạng thái destroyRockMode

    if rockDestructionLoop then -- Nếu loop đang chạy, dừng nó
        task.cancel(rockDestructionLoop)
        rockDestructionLoop = nil
    end

    if enabled then
        rockDestructionLoop = task.spawn(function() -- Bắt đầu loop mới
            while destroyRockMode do -- Loop chỉ chạy khi destroyRockMode là true
                local rocksFolder = Workspace:FindFirstChild("Rocks")
                if rocksFolder then
                    for _, rock in ipairs(rocksFolder:GetChildren()) do
                        if rock:IsA("BasePart") then
                            pcall(function() -- Sử dụng pcall để tránh lỗi script khi destroy object
                                rock:Destroy()
                            end)
                        end
                    end
                end
                task.wait(0.1) -- Chờ một chút trước khi kiểm tra lại, giảm tải cho game
            end
            print("Rock Destruction Loop Stopped") -- Feedback when loop stops
        end)
        print("Rock Destruction Enabled - Continuous") -- Feedback when enabled
    else
        print("Rock Destruction Disabled") -- Feedback when disabled
    end
end


-- Tab Main
MainTab:AddToggle("LevithanToggle", {
    Title = "Find Levithan and vulcano",
    Description = "Enable/Disable Levithan search and vulcano",
    Callback = function(Value)
        if Value then -- Toggle is ON
            if not scriptEnabled then -- Only start if not already enabled
                scriptEnabled = true
                tweenBoatToVector3(Vector3.new(-99999999, 0, 0))
            end
        else -- Toggle is OFF
            if scriptEnabled then -- Only stop if currently enabled
                stopTween("Người dùng tắt Tìm Levithan.")
            end
        end
    end
})

MainTab:AddToggle("Khóa", {
    Title = "flying Boat",
    Description = "Enable/Disable Flight Lock",
    Callback = function(Value)
        lockHeightMode = Value
    end
})

MainTab:AddToggle("Rock", {
    Title = "Breaking rocks",
    Description = "Continuous ice breaking on/off",
    Callback = function(Value)
        setRockDestruction(Value) --  <- This line calls the setRockDestruction function NOW.
    end
})

MainTab:AddInput("Y", {
    Title = "Flying Altitude",
    Description = "Enter altitude",
    Callback = function(Value)
        local val = tonumber(Value)
        if val then
            customY = val
        else
            customY = 100
        end
    end
})

MainTab:AddInput("Speed", {
    Title = "Speed",
    Description = "Enter tween speed",
    Callback = function(Value)
        local val = tonumber(Value)
        if val and val > 0 then
            tweenSpeed = val
        else
            tweenSpeed = 350
        end
    end
})

MainTab:AddToggle("Auto Return", {
    Title = "Automatically return",
    Description = "Auto return on/off",
    Callback = function(Value)
        autoReturnEnabled = Value
        if autoReturnEnabled then
            task.spawn(autoReturnCheck)
        end
    end
})

MainTab:AddToggle("Kill Aura", {
    Title = "Golem and Monster Killed",
    Description = "Kill Aura",
    Callback = function(Value)
        killAuraEnabled = Value
        if killAuraEnabled then
            task.spawn(killAuraLoop)
        end
    end
})

-- Tab Player
local playerDropdown = PlayerTab:AddDropdown("Chọn người chơi", {
    Title = "Select players",
    Description = "List of players",
    Values = {}, -- Danh sách người chơi sẽ được cập nhật sau
    Callback = function(Value)
        local selected = Players:FindFirstChild(Value)
        if selected then
            selectedPlayer = selected
            print("Selected player:", Value)
        else
            selectedPlayer = nil
            print("Player not found:", Value)
        end
    end
})

-- Hàm làm mới danh sách người chơi
local function refreshPlayerList()
    local playerNames = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            table.insert(playerNames, player.Name)
        end
    end
    playerDropdown:SetValues(playerNames) -- Cập nhật danh sách người chơi
end

-- Thêm TextBox để tìm tên người chơi
local playerSearchBox = PlayerTab:AddInput("Find players", {
    Title = "Find players",
    Description = "Enter player name to search",
    Callback = function(Value)
        -- Lọc danh sách người chơi dựa trên từ khóa
        local filteredPlayers = {}
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and string.find(string.lower(player.Name), string.lower(Value)) then
                table.insert(filteredPlayers, player.Name)
            end
        end
        playerDropdown:SetValues(filteredPlayers) -- Cập nhật danh sách người chơi
    end
})

-- Nút Refresh
PlayerTab:AddButton({
    Title = "Refresh",
    Description = "Refresh player list",
    Callback = function()
        refreshPlayerList()
    end
})

-- Làm mới danh sách người chơi khi mở tab
refreshPlayerList()

-- Tab Đảo
local islandDropdown = IslandTab:AddDropdown("Select island", {
    Title = "Select island",
    Description = "List of islands",
    Values = {"Tiki", "Hydra", "Fortress", "Mansion", "Dark Castle", "Port", "Big Tree", "Cake Island"},
    Callback = function(Value)
        selectedIslandCoord = islands[Value] -- Lưu tọa độ đảo được chọn
        print("Island selected:", Value)
    end
})

-- Thêm TextBox để tìm tên đảo
local islandSearchBox = IslandTab:AddInput("Find the island", {
    Title = "Find the island",
    Description = "Enter island name to search",
    Callback = function(Value)
        -- Lọc danh sách đảo dựa trên từ khóa
        local filteredIslands = {}
        for islandName, _ in pairs(islands) do
            if string.find(string.lower(islandName), string.lower(Value)) then
                table.insert(filteredIslands, islandName)
            end
        end
        islandDropdown:SetValues(filteredIslands) -- Cập nhật danh sách đảo
    end
})

-- Nút Tween đến đảo
IslandTab:AddButton({
    Title = "Tween: OFF",
    Description = "Tween to island on/off",
    Callback = function()
        if islandTweening then
            stopTween("User turns off Tween Island.")
            islandTweening = false
            Fluent:Notify({
                Title = "Tween Island",
                Content = "Tween Island is off.",
                Duration = 3
            })
        else
            if not selectedIslandCoord then
                Fluent:Notify({
                    Title = "Error",
                    Content = "Haven't selected an island yet!",
                    Duration = 3
                })
                return
            end
            islandTweening = true
            Fluent:Notify({
                Title = "Tween Island",
                Content = "Đang tween đến đảo...",
                Duration = 3
            })
            tweenBoatToVector3(selectedIslandCoord)
        end
    end
})

-- Nút Tween đến Hydra
IslandTab:AddToggle("HydraIslandToggle", {
    Title = "Pull the heart to Hydra",
    Description = "Enable/Disable Drag Heart to Hydra",
    Callback = function(Value)
        if Value then -- Bật
            islandTweening = true
            Fluent:Notify({
                Title = "Tween đến Hydra",
                Content = "Đang tween đến Hydra...",
                Duration = 3
            })
            tweenBoatToVector3(islands["Hydra"])
        else -- Tắt
            stopTween("User disables Tween to Hydra.")
            islandTweening = false
            Fluent:Notify({
                Title = "Tween to Hydra",
                Content = "Tween off to Hydra.",
                Duration = 3
            })
        end
    end
})

-- Nút Tween đến Tiki
IslandTab:AddToggle("TikiIslandToggle", {
    Title = "Drag heart to tiki",
    Description = "Enable/Disable Tiki Heart Drag",
    Callback = function(Value)
        if Value then -- Bật
            islandTweening = true
            Fluent:Notify({
                Title = "Tween đến Tiki",
                Content = "Tween coming to Tiki...",
                Duration = 3
            })
            tweenBoatToVector3(islands["Tiki"])
        else -- Tắt
            stopTween("User turns off Tween to Tiki.")
            islandTweening = false
            Fluent:Notify({
                Title = "Tween đến Tiki",
                Content = "Đã tắt Tween đến Tiki.",
                Duration = 3
            })
        end
    end
})

-- Tab Khác
OtherTab:AddToggle("Anti Die", {
    Title = "Anti Die",
    Description = "Anti die is only effective when HP=30%",
    Callback = function(Value)
        antiDieEnabled = Value
        if not antiDieEnabled and originalPosition then
            -- Đưa player về vị trí cũ
            local character = LocalPlayer.Character
            if character and character:FindFirstChild("HumanoidRootPart") then
                character.HumanoidRootPart.CFrame = CFrame.new(originalPosition)
            end
            originalPosition = nil -- Reset vị trí cũ
        end
        if antiDieEnabled then
            task.spawn(checkHealth)
        end
    end
})

OtherTab:AddToggle("DashNoCD", {
    Title = "DashNoCD",
    Description = "DashNoCD",
    Callback = function(Value)
        dashNoCDEnabled = Value
        _G.DodgewithoutCool = dashNoCDEnabled
        if dashNoCDEnabled then
            task.spawn(NoCooldown)
        end
    end
})

OtherTab:AddToggle("WalkSpeed", {
    Title = "WalkSpeed",
    Description = "WalkSpeed",
    Callback = function(Value)
        walkSpeedEnabled = Value
        tpwalking = walkSpeedEnabled
        if walkSpeedEnabled and speaker.Character then
            startTeleportWalk(speaker.Character)
        end
    end
})

OtherTab:AddToggle("Gas", {
    Title = "Gas",
    Description = "Exclusively for Developers",
    Callback = function(Value)
        local playerName = player.Name
        if playerName == "VITORFFPRO123" then -- Kiểm tra tên người chơi
            gasEnabled = Value
            if gasEnabled then
                createNoCooldownTool()
            else
                local backpack = LocalPlayer:FindFirstChild("Backpack")
                if backpack then
                    local tool = backpack:FindFirstChild("NoCooldownToggle")
                    if tool then
                        tool:Destroy()
                    end
                end
            end
        else
            gasEnabled = false
            Fluent:Notify({
                Title = "Lỗi",
                Content = "Chỉ Developer mới dùng được tính năng này",
                Duration = 3
            })
        end
    end
})

-- Boat Protect Toggle
OtherTab:AddToggle("BoatProtect", {
    Title = "Boat Protection 🔰",
    Description = "Enable/Disable Boat Protection When HP=50%",
    Callback = function(Value)
        boatProtectEnabled = Value
        if boatProtectEnabled then
            task.spawn(checkBoatHealth)
        else
            stopBoatProtect()
        end
    end
})

-- Boat Protect Immediate Toggle
OtherTab:AddToggle("BoatProtectImmediate", {
    Title = "Protect Your Boat Immediately",
    Description = "Instant Boat Protection On/Off",
    Callback = function(Value)
        boatProtectImmediateEnabled = Value
        if boatProtectImmediateEnabled then
            task.spawn(boatProtectImmediateLoop)
            local boat = currentBoat
            if not boat then
                boat, _ = getBoatVehicleSeat()
                currentBoat = boat
            end
            if boat then
                tweenBoatToProtectHeight(boat)
            end
        else
            stopBoatProtect()
        end
    end
})

-- Tab Fruit
FruitTab:AddToggle("Find Fruit", {
    Title = "Find Fruit",
    Description = "Auto Pick Up Fruit + Esp Fruit",
    Callback = function(Value)
        findFruitEnabled = Value
        if not findFruitEnabled then
            local fruits = findFruits()
            for _, fruit in ipairs(fruits) do
                if fruitsESP[fruit] then
                    fruitsESP[fruit]:Destroy()
                    fruitsESP[fruit] = nil
                end
            end
        end
    end
})

-- Auto Random Fruit Toggle
FruitTab:AddToggle("Auto Random Fruit", {
    Title = "Auto Random Fruit",
    Description = "Enable/Disable Auto Buy Random Devil Fruit",
    Callback = function(state)
        _G.Random_Auto = state
        if state then
            task.spawn(function()
                pcall(function()
                    while _G.Random_Auto do
                        wait(0.1)
                        game:GetService("ReplicatedStorage").Remotes.CommF_:InvokeServer("Cousin", "Buy") -- Mua random fruit
                    end
                end)
            end)
        else
            _G.Random_Auto = false -- Ensure to set _G.Random_Auto to false when disabling
        end
    end
})

-- Tab Player - Nút Spectate
PlayerTab:AddToggle("Spectate", {
    Title = "See Player",
    Description = "Player Spectate",
    Callback = function(Value)
        if Value then
            if selectedPlayer and selectedPlayer.Character then
                localplr = LocalPlayer
                localplr.CameraMaxZoomDistance = 100
                localplr.CameraMinZoomDistance = 0.1
                workspace.CurrentCamera.CameraSubject = selectedPlayer.Character
            end
        else
            workspace.CurrentCamera.CameraSubject = LocalPlayer.Character
        end
    end
})

-- Nút Tween Player trong Tab "Player"
PlayerTab:AddButton({
    Title = "Tween Player",
    Description = "Tween boat to selected Player",
    Callback = function()
        if playerTweening then
            stopTween("User turns off Tween Player.")
            playerTweening = false
            Fluent:Notify({
                Title = "Tween Player",
                Content = "Đã tắt Tween Player.",
                Duration = 3
            })
            return -- Thêm return để ngăn gọi tweenBoatToPlayer khi đang tween
        end
        if not selectedPlayer then
            Fluent:Notify({
                Title = "Error",
                Content = "No player selected yet!",
                Duration = 3
            })
            return
        end
        Fluent:Notify({
            Title = "Tween Player",
            Content = "Tweening to player...",
            Duration = 3
        })
        tweenBoatToPlayer(selectedPlayer) -- Gọi hàm tweenBoatToPlayer đã sửa đổi
    end
})

-- Hàm để đóng hoặc thu nhỏ cửa sổ (toggle)
local minimized = false
local function toggleMinimize()
    minimized = not minimized
    if minimized then
        Window:SetSize(UDim2.fromOffset(160, 60)) -- Adjust for a smaller minimized size
        Window:SetProperty("Acrylic", false)

        -- Ẩn tất cả các tab
        for _, tab in ipairs(Window:GetTabs()) do
            tab:SetProperty("Visible", false)
        end

        Window.TitleBar:SetProperty("Visible", false)
        Window.SubTitleLabel:SetProperty("Visible", false)

    else
        Window:SetSize(UDim2.fromOffset(580, 460)) -- Restore original size
        Window:SetProperty("Acrylic", true)
        for _, tab in ipairs(Window:GetTabs()) do
            tab:SetProperty("Visible", true)
        end

        Window.TitleBar:SetProperty("Visible", true)
        Window.SubTitleLabel:SetProperty("Visible", true)
    end
end
Window.MinimizeKeybind = toggleMinimize
while true do task.wait(0.00001) 
print("Ey yo skider")
end
