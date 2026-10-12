-- โหลด Rayfield UI Library
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "Ryujin | Saitama Roadworks Auto Farm",
    LoadingTitle = "Loading Script...",
    LoadingSubtitle = "by Educational Guide",
    ConfigurationSaving = { Enabled = false },
    KeySystem = false,
})

local Tab = Window:CreateTab("Auto Farm", 4483362458)

_G.AutoRoadworks = false
_G.TravelOffset = -8.5 -- ความลึกขณะกำลังเดินทางระหว่างจุด
_G.CollectOffset = -7.9 -- ความลึกเมื่อถึงจุด Part เพื่อเด้งเก็บเควส

Tab:CreateToggle({
    Name = "Auto Farm Long Roadworks (Dynamic Depth -8.5 / -7.9)",
    CurrentValue = false,
    Flag = "AutoRoadworksToggle",
    Callback = function(Value)
        _G.AutoRoadworks = Value
        if Value then
            task.spawn(StartHoverAndNoclipLoop)
            task.spawn(StartRoadworksLoop)
        else
            StopHoverAndNoclip()
        end
    end,
})

-- ส่วนแสดงผล Stats บน UI
local SectionStats = Tab:CreateSection("Farm Statistics")
local LabelStatus = Tab:CreateLabel("Status: Idle")
local LabelCompleted = Tab:CreateLabel("Quests Completed: 0")

local statsCompleted = 0
local statsStatus = "Idle"

local function updateStats(statusText)
    statsStatus = statusText
    LabelStatus:Set("Status: " .. statsStatus)
    LabelCompleted:Set("Quests Completed: " .. tostring(statsCompleted))
end

-- ระบบ Noclip และค้ำจุนตำแหน่งลอยตัว
local isUnderground = false

function StartHoverAndNoclipLoop()
    local RunService = game:GetService("RunService")
    
    while _G.AutoRoadworks do
        local player = game.Players.LocalPlayer
        local character = player and player.Character
        
        if character then
            local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            
            -- ปิด CanCollide เฉพาะตอนดำดิน
            if isUnderground then
                for _, part in ipairs(character:GetDescendants()) do
                    if part:IsA("BasePart") and part.CanCollide then
                        part.CanCollide = false
                    end
                end
                
                if humanoidRootPart then
                    humanoidRootPart.Velocity = Vector3.new(0, 0, 0)
                    humanoidRootPart.RotVelocity = Vector3.new(0, 0, 0)
                    
                    if not humanoidRootPart:FindFirstChild("HoverVelocity") then
                        local bv = Instance.new("BodyVelocity")
                        bv.Name = "HoverVelocity"
                        bv.Velocity = Vector3.new(0, 0, 0)
                        bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                        bv.Parent = humanoidRootPart
                    end
                end
                
                if humanoid then
                    humanoid.AutoRotate = false
                    humanoid:ChangeState(Enum.HumanoidStateType.Physics)
                end
            else
                -- เมื่ออยู่บนดินรับเควส คืนค่า Physics ปกติ
                if humanoidRootPart and humanoidRootPart:FindFirstChild("HoverVelocity") then
                    humanoidRootPart.HoverVelocity:Destroy()
                end
                if humanoid then
                    humanoid.AutoRotate = true
                    humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
                end
            end
        end
        
        RunService.Stepped:Wait()
    end
    
    StopHoverAndNoclip()
end

function StopHoverAndNoclip()
    isUnderground = false
    local player = game.Players.LocalPlayer
    local character = player and player.Character
    if character then
        local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
        if humanoidRootPart and humanoidRootPart:FindFirstChild("HoverVelocity") then
            humanoidRootPart.HoverVelocity:Destroy()
        end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.AutoRotate = true
            humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
        end
    end
end

-- ฟังก์ชันคำนวณ CFrame ดำดินตามความลึกที่ระบุ
local function getUndergroundCFrame(targetCFrame, depthOffset)
    return targetCFrame * CFrame.new(0, depthOffset, 0) * CFrame.Angles(math.rad(90), math.rad(180), 0)
end

-- ฟังก์ชันจำลองการกดปุ่มคีย์บอร์ด
local function pressKey(keyCode)
    local vim = game:GetService("VirtualInputManager")
    if vim then
        vim:SendKeyEvent(true, keyCode, false, game)
        task.wait(0.05)
        vim:SendKeyEvent(false, keyCode, false, game)
    end
end

-- ฟังก์ชันคลิก UI บทสนทนา
local function clickGuiOption(targetText, optionIndex)
    local player = game.Players.LocalPlayer
    local playerGui = player and player:FindFirstChild("PlayerGui")
    if not playerGui then return false end
    
    local vim = game:GetService("VirtualInputManager")
    local clicked = false
    
    for _, gui in ipairs(playerGui:GetDescendants()) do
        if (gui:IsA("TextLabel") or gui:IsA("TextButton")) and gui.Visible then
            local txt = gui.Text
            if txt:find(targetText) or txt:find("^" .. tostring(optionIndex) .. "%.") then
                local btn = gui:IsA("TextButton") and gui or gui.Parent
                if btn and btn:IsA("TextButton") then
                    local pos = btn.AbsolutePosition
                    local size = btn.AbsoluteSize
                    local centerX = pos.X + (size.X / 2)
                    local centerY = pos.Y + (size.Y / 2) + 36
                    
                    vim:SendMouseButtonEvent(centerX, centerY, 0, true, game, 1)
                    task.wait(0.02)
                    vim:SendMouseButtonEvent(centerX, centerY, 0, false, game, 1)
                    
                    if firesignal then
                        pcall(function() firesignal(btn.MouseButton1Click) end)
                    end
                    clicked = true
                end
            end
        end
    end
    return clicked
end

-- ระบบเช็คเควส: ตรวจสอบ RoadworkPoint ใน Terrain
local function hasRoadworkPoint()
    local terrain = workspace:FindFirstChildOfClass("Terrain") or workspace.Terrain
    if terrain then
        return terrain:FindFirstChild("RoadworkPoint") ~= nil
    end
    return false
end

-- ฟังก์ชันรับเควส Long จาก Saitama
local function interactWithSaitamaLong()
    pressKey(Enum.KeyCode.E)
    task.wait(0.4)
    
    clickGuiOption("training routes", 1)
    pressKey(Enum.KeyCode.One)
    task.wait(0.4)
    
    clickGuiOption("Long run", 1)
    pressKey(Enum.KeyCode.One)
    task.wait(0.4)
    
    clickGuiOption("End Dialogue", 1)
    pressKey(Enum.KeyCode.One)
    task.wait(0.2)
    pressKey(Enum.KeyCode.Space)
end

-- ฟังก์ชันลูปหลักการทำงาน Auto Farm
function StartRoadworksLoop()
    while _G.AutoRoadworks do
        task.wait(0.2)
        
        local player = game.Players.LocalPlayer
        if not player then continue end
        
        local character = player.Character or player.CharacterAdded:Wait()
        local humanoidRootPart = character:WaitForChild("HumanoidRootPart", 5)
        if not humanoidRootPart then continue end
        
        local ts = game:GetService("TweenService")
        
        -------------------------------------------------------------
        -- ขั้นตอนที่ 1: ยืนข้างหลัง Saitama แบบปกติเพื่อคุยรับเควส
        -------------------------------------------------------------
        while not hasRoadworkPoint() and _G.AutoRoadworks do
            isUnderground = false -- ปิดโหมดดำดิน
            updateStats("Taking Long Quest (Behind Saitama)...")
            
            local npcFolder = workspace:FindFirstChild("Ignore") 
                and workspace.Ignore:FindFirstChild("NPCs") 
                and workspace.Ignore.NPCs:FindFirstChild("Important NPCs")
            local saitama = npcFolder and npcFolder:FindFirstChild("Saitama") 
                or workspace:FindFirstChild("Saitama", true)
            
            if saitama and saitama:FindFirstChild("HumanoidRootPart") then
                -- บินไปยืนอยู่ทางด้านหลังของ Saitama
                local behindSaitamaCFrame = saitama.HumanoidRootPart.CFrame * CFrame.new(0, 0, 3)
                local distance = (humanoidRootPart.Position - behindSaitamaCFrame.Position).Magnitude
                local flightTime = math.max(distance / 60, 0.1)
                
                local tween = ts:Create(humanoidRootPart, TweenInfo.new(flightTime, Enum.EasingStyle.Linear), {CFrame = behindSaitamaCFrame})
                tween:Play()
                tween.Completed:Wait()
                
                task.wait(0.3)
                
                -- คุยรับเควส
                interactWithSaitamaLong()
                
                local waitTimer = tick() + 2
                while not hasRoadworkPoint() and tick() < waitTimer and _G.AutoRoadworks do
                    task.wait(0.1)
                end
            else
                updateStats("Searching for Saitama NPC...")
                task.wait(0.5)
            end
        end
        
        -------------------------------------------------------------
        -- ขั้นตอนที่ 2: รับเควสเสร็จแล้ว มุดดิน -8.5 บินไปหาจุด แล้วเด้งขึ้น -7.9 เพื่อเก็บ
        -------------------------------------------------------------
        if hasRoadworkPoint() then
            isUnderground = true -- เปิดโหมดดำดิน
            updateStats("Dipping Underground (-8.5 Units)...")
            
            -- มุดลงดิน ณ จุดข้างหลัง Saitama ทันทีที่ระดับ -8.5
            local dipTargetCFrame = getUndergroundCFrame(humanoidRootPart.CFrame, _G.TravelOffset)
            local dipTween = ts:Create(humanoidRootPart, TweenInfo.new(0.3, Enum.EasingStyle.Linear), {CFrame = dipTargetCFrame})
            dipTween:Play()
            dipTween.Completed:Wait()
            humanoidRootPart.CFrame = dipTargetCFrame
            
            task.wait(0.1)
            
            local roadworksFolder = workspace:FindFirstChild("Ignore")
                and workspace.Ignore:FindFirstChild("Interactables")
                and workspace.Ignore.Interactables:FindFirstChild("Roadworks")
                
            if roadworksFolder then
                local routeFolder = roadworksFolder:FindFirstChild("Long")
                if routeFolder then
                    local maxParts = 16
                    
                    for i = 1, maxParts do
                        if not _G.AutoRoadworks then break end
                        
                        local part = routeFolder:FindFirstChild(tostring(i))
                        if part and part:IsA("BasePart") then
                            -- 1. บินใต้พื้นระดับ -8.5 ไปยังพาร์ทเป้าหมาย
                            updateStats("Moving Underground (-8.5) to Part " .. tostring(i) .. "/" .. tostring(maxParts))
                            local travelCFrame = getUndergroundCFrame(part.CFrame, _G.TravelOffset)
                            local distance = (humanoidRootPart.Position - travelCFrame.Position).Magnitude
                            local flightTime = math.max(distance / 60, 0.05)
                            
                            local flightTween = ts:Create(humanoidRootPart, TweenInfo.new(flightTime, Enum.EasingStyle.Linear), {CFrame = travelCFrame})
                            flightTween:Play()
                            flightTween.Completed:Wait()
                            
                            -- 2. เมื่อถึงจุดพาร์ท ให้เด้งขึ้นมาที่ -7.9 ทันทีเพื่อเก็บเควส
                            updateStats("Collecting Part " .. tostring(i) .. " (-7.9 Units)...")
                            local collectCFrame = getUndergroundCFrame(part.CFrame, _G.CollectOffset)
                            local collectTween = ts:Create(humanoidRootPart, TweenInfo.new(0.08, Enum.EasingStyle.Linear), {CFrame = collectCFrame})
                            collectTween:Play()
                            collectTween.Completed:Wait()
                            
                            humanoidRootPart.CFrame = collectCFrame
                            task.wait(0.05)
                        end
                    end
                    
                    if _G.AutoRoadworks then
                        statsCompleted = statsCompleted + 1
                        updateStats("Completed 1 Round! Waiting for next loop...")
                    end
                end
            end
        end
        
        task.wait(0.3)
    end
    
    isUnderground = false
    updateStats("Idle (Farm Stopped)")
end
