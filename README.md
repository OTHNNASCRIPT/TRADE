-- Dual Wield GUI كامل مع تحريك + إظهار/إخفاء + رفع الأداة + إخفاء النصوص
-- Save بدون Clone (ما فيش تكرار أدوات في الشنطة)

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Debris = game:GetService("Debris")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local char = player.Character or player.CharacterAdded:Wait()
local humanoid = char:WaitForChild("Humanoid")

-- حفظ الأدوات الأصلية (مش Clone)
local savedTool1 = nil
local savedTool2 = nil

local dragging = false
local dragStart = nil
local startPos = nil

local guiVisible = true

print("[DualWield] تم التصليح: Save بدون Clone، ما فيش تكرار أدوات")

-- ================== دوال الأدوات ==================

local function getCurrentTool()
    for _, t in char:GetChildren() do
        if t:IsA("Tool") then return t end
    end
    return nil
end

local function makeToolInvisibleToOthers(tool)
    if not tool then return end
    
    for _, part in ipairs(tool:GetDescendants()) do
        if part:IsA("BasePart") or part:IsA("MeshPart") then
            part.LocalTransparencyModifier = 0
            part.Transparency = 1
            part.CanCollide = false
            part.CanQuery = false
            part.Massless = true
            
            pcall(function()
                part:SetNetworkOwner(player)
            end)
        end
    end
    
    local handle = tool:FindFirstChild("Handle") or tool:FindFirstChildWhichIsA("BasePart")
    if handle then
        handle.Transparency = 1
        handle.LocalTransparencyModifier = 0
    end
    
    hideOverheadTexts(tool)
    
    print("تم إخفاء Tool + نصوصها: " .. tool.Name)
end

local function hideOverheadTexts(target)
    target = target or char
    
    for _, obj in ipairs(target:GetDescendants()) do
        if obj:IsA("BillboardGui") or obj:IsA("SurfaceGui") then
            obj:Destroy()
            print("تم حذف " .. obj.ClassName .. ": " .. obj.Name)
        elseif obj:IsA("TextLabel") and obj.Parent:IsA("BillboardGui") then
            obj.TextTransparency = 1
            obj.BackgroundTransparency = 1
        end
    end
    
    local head = target:FindFirstChild("Head")
    if head then
        for _, g in head:GetChildren() do
            if g:IsA("BillboardGui") then
                g:Destroy()
            end
        end
    end
end

local function equipFirstTool(tool)
    if not tool then return end
    tool.Parent = char
    humanoid:EquipTool(tool)
    print("إمساك Tool 1: " .. tool.Name)
end

local function equipSecondHidden(tool2)
    if not tool2 then return end
    
    local leftHand = char:FindFirstChild("LeftHand") or char:FindFirstChild("Left Arm")
    local handle = tool2:FindFirstChild("Handle") or tool2:FindFirstChildWhichIsA("BasePart")
    
    if handle and leftHand then
        tool2.Parent = char
        
        local weld = Instance.new("WeldConstraint")
        weld.Part0 = leftHand
        weld.Part1 = handle
        weld.Parent = handle
        
        handle.CFrame = leftHand.CFrame * CFrame.new(0, -1, 0) * CFrame.Angles(0, math.pi, 0)
        
        makeToolInvisibleToOthers(tool2)
        
        task.delay(0.3, function()
            hideOverheadTexts(tool2)
            hideOverheadTexts(char)
        end)
        
        tool2.Activated:Connect(function()
            tool2:Activate()
        end)
        
        print("Tool 2 مخفية + نصوصها مخفية")
    end
end

-- رفع الأداة + إخفاء الاسم من البار
local function throwCurrentTool()
    local tool = getCurrentTool()
    if not tool then
        statusLabel.Text = "مفيش أداة ماسكها حاليًا"
        return
    end
    
    tool.ToolTip = ""
    tool.RequiresHandle = false
    
    local handle = tool:FindFirstChild("Handle") or tool:FindFirstChildWhichIsA("BasePart")
    if handle then
        for _, w in handle:GetChildren() do
            if w:IsA("WeldConstraint") or w:IsA("Weld") then w:Destroy() end
        end
        
        hideOverheadTexts(tool)
        
        humanoid:UnequipTools()
        
        local originalCFrame = handle.CFrame
        local targetCFrame = originalCFrame * CFrame.new(0, 20, 0)
        
        local tweenInfo = TweenInfo.new(1.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        local tween = TweenService:Create(handle, tweenInfo, {CFrame = targetCFrame})
        
        tween:Play()
        
        tween.Completed:Connect(function()
            tool:Destroy()
            statusLabel.Text = "تم رفع وإخفاء الأداة + اسمها من البار"
        end)
    end
end

-- ================== الـ GUI ==================

local sg = Instance.new("ScreenGui")
sg.Name = "DualToolGUI"
sg.ResetOnSpawn = false
sg.DisplayOrder = 10000
sg.Parent = playerGui

local frame = Instance.new("Frame")
frame.Name = "MainFrame"
frame.Size = UDim2.new(0, 300, 0, 300)
frame.Position = UDim2.new(0.5, -150, 0.5, -150)
frame.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
frame.BorderSizePixel = 0
frame.ClipsDescendants = true
frame.Parent = sg

local uicorner = Instance.new("UICorner")
uicorner.CornerRadius = UDim.new(0, 12)
uicorner.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 45)
title.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
title.Text = "🎮 Dual Wield - اسحب من هنا"
title.TextColor3 = Color3.new(1,1,1)
title.TextScaled = true
title.Font = Enum.Font.GothamBold
title.Parent = frame

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0, 40, 0, 40)
toggleBtn.Position = UDim2.new(1, -45, 0, 5)
toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
toggleBtn.Text = "−"
toggleBtn.TextColor3 = Color3.new(1,1,1)
toggleBtn.TextScaled = true
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.Parent = frame

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 8)
toggleCorner.Parent = toggleBtn

local save1 = Instance.new("TextButton")
save1.Size = UDim2.new(0.45, 0, 0, 45)
save1.Position = UDim2.new(0.03, 0, 0.22, 0)
save1.BackgroundColor3 = Color3.fromRGB(0, 180, 0)
save1.Text = "Save Tool 1"
save1.TextColor3 = Color3.new(1,1,1)
save1.TextScaled = true
save1.Font = Enum.Font.Gotham
save1.Parent = frame
Instance.new("UICorner", save1).CornerRadius = UDim.new(0, 8)

local save2 = Instance.new("TextButton")
save2.Size = UDim2.new(0.45, 0, 0, 45)
save2.Position = UDim2.new(0.52, 0, 0.22, 0)
save2.BackgroundColor3 = Color3.fromRGB(0, 180, 0)
save2.Text = "Save Tool 2"
save2.TextColor3 = Color3.new(1,1,1)
save2.TextScaled = true
save2.Font = Enum.Font.Gotham
save2.Parent = frame
Instance.new("UICorner", save2).CornerRadius = UDim.new(0, 8)

local activate = Instance.new("TextButton")
activate.Size = UDim2.new(0.94, 0, 0, 45)
activate.Position = UDim2.new(0.03, 0, 0.44, 0)
activate.BackgroundColor3 = Color3.fromRGB(220, 100, 0)
activate.Text = "Activate → Tool1 ثم Tool2 مخفية"
activate.TextColor3 = Color3.new(1,1,1)
activate.TextScaled = true
activate.Font = Enum.Font.GothamBold
activate.Parent = frame
Instance.new("UICorner", activate).CornerRadius = UDim.new(0, 8)

local throwBtn = Instance.new("TextButton")
throwBtn.Size = UDim2.new(0.94, 0, 0, 45)
throwBtn.Position = UDim2.new(0.03, 0, 0.66, 0)
throwBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
throwBtn.Text = "رفع الأداة (تطير وتختفي)"
throwBtn.TextColor3 = Color3.new(1,1,1)
throwBtn.TextScaled = true
throwBtn.Font = Enum.Font.GothamBold
throwBtn.Parent = frame
Instance.new("UICorner", throwBtn).CornerRadius = UDim.new(0, 8)

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(0.94, 0, 0, 30)
statusLabel.Position = UDim2.new(0.03, 0, 0.88, 0)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = "جاهز..."
statusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
statusLabel.TextScaled = true
statusLabel.Font = Enum.Font.Gotham
statusLabel.Parent = frame

-- ================== السحب ==================
title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = frame.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- زر الإظهار / الإخفاء
toggleBtn.MouseButton1Click:Connect(function()
    guiVisible = not guiVisible
    if guiVisible then
        frame.Visible = true
        toggleBtn.Text = "−"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
    else
        frame.Visible = false
        toggleBtn.Text = "+"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(255, 80, 80)
    end
end)

-- ================== زر رفع الأداة ==================
throwBtn.MouseButton1Click:Connect(function()
    local tool = getCurrentTool()
    if not tool then
        statusLabel.Text = "مفيش أداة ماسكها حاليًا"
        return
    end
    
    tool.ToolTip = ""
    tool.RequiresHandle = false
    
    local handle = tool:FindFirstChild("Handle") or tool:FindFirstChildWhichIsA("BasePart")
    if handle then
        for _, w in handle:GetChildren() do
            if w:IsA("WeldConstraint") or w:IsA("Weld") then w:Destroy() end
        end
        
        hideOverheadTexts(tool)
        
        humanoid:UnequipTools()
        
        local originalCFrame = handle.CFrame
        local targetCFrame = originalCFrame * CFrame.new(0, 20, 0)
        
        local tweenInfo = TweenInfo.new(1.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        local tween = TweenService:Create(handle, tweenInfo, {CFrame = targetCFrame})
        
        tween:Play()
        
        tween.Completed:Connect(function()
            tool:Destroy()
            statusLabel.Text = "تم رفع وإخفاء الأداة + اسمها من البار"
        end)
    end
end)

-- ================== Save بدون Clone (حفظ الأداة الأصلية) ==================
save1.MouseButton1Click:Connect(function()
    local t = getCurrentTool()
    if t then
        -- نقل الأداة الأصلية لمكان آمن (مش clone)
        t.Parent = playerGui  -- مخفية وآمنة
        savedTool1 = t
        statusLabel.Text = "Tool 1 محفوظ (الأصلية): " .. t.Name
    else
        statusLabel.Text = "مفيش أداة في يدك"
    end
end)

save2.MouseButton1Click:Connect(function()
    local t = getCurrentTool()
    if t then
        t.Parent = playerGui
        savedTool2 = t
        statusLabel.Text = "Tool 2 محفوظ (الأصلية): " .. t.Name
    else
        statusLabel.Text = "مفيش أداة في يدك"
    end
end)

-- ================== التفعيل ==================
activate.MouseButton1Click:Connect(function()
    if not savedTool1 then
        statusLabel.Text = "احفظ Tool 1 أولاً"
        return
    end
    
    activate.Text = "جاري..."
    activate.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    
    -- رجع Tool 1 للشخصية
    savedTool1.Parent = char
    equipFirstTool(savedTool1)
    statusLabel.Text = "Tool 1 مُمسكة... انتظر"
    
    task.wait(1.3)
    
    if savedTool2 then
        savedTool2.Parent = char
        equipSecondHidden(savedTool2)
        statusLabel.Text = "تم! Tool 2 مخفية عن الآخرين"
    else
        statusLabel.Text = "Tool 2 مش محفوظة"
    end
    
    task.wait(0.8)
    activate.Text = "Activate → Tool1 ثم Tool2 مخفية"
    activate.BackgroundColor3 = Color3.fromRGB(220, 100, 0)
end)

player.CharacterAdded:Connect(function(newChar)
    char = newChar
    humanoid = newChar:WaitForChild("Humanoid")
    savedTool1 = nil
    savedTool2 = nil
    statusLabel.Text = "شخصية جديدة - ابدأ من جديد"
end)

print("[DualWield] جاهز – Save بدون تكرار، الأدوات الأصلية فقط")
