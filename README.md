local Workspace = cloneref(game:GetService("Workspace"))
local RunService = cloneref(game:GetService("RunService"))
local Players = cloneref(game:GetService("Players"))
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local ESP = {
    Enabled = true,
    TeamCheck = true,
    MaxDistance = 2000,
    Scale = 0.95,
    ReferenceResolution = { Width = 1536, Height = 658 },
    Options = { Teamcheck = true, Friendcheck = true },
    Drawing = {
        Names = { Enabled = true, RGB = Color3.fromRGB(180, 220, 255) },
        Distances = { Enabled = true, RGB = Color3.fromRGB(150, 200, 255), Position = "Bottom" },
        Healthbar = {
            Enabled = true,
            Width = 0.8,
            RGB = Color3.fromRGB(100, 180, 255),   
            LowRGB = Color3.fromRGB(50, 120, 200), 
            BackgroundRGB = Color3.fromRGB(30, 30, 50),
            BackgroundTransparency = 0.35,
            HealthText = true,
            HealthTextRGB = Color3.fromRGB(100, 180, 255),
        },
        Boxes = {
            Filled = { Enabled = false, Transparency = 0.65, RGB = Color3.fromRGB(80, 150, 255) },
            Gradient = { From = Color3.fromRGB(30, 55, 110), To = Color3.fromRGB(80, 150, 255) },
            Corner = { Enabled = false, RGB = Color3.fromRGB(120, 180, 255), Thickness = 1 },
        },
        Chams = {
            Enabled = true,
            FillRGB = Color3.fromRGB(0, 150, 255),         
            FillTransparency = 0.6,                      
            OutlineRGB = Color3.fromRGB(0, 255, 255),     
            OutlineTransparency = 0.0,                   
            AlwaysOnTop = true,                          
        },
    }
}

local ScreenGui
local PlayerList = {}
local PlayerCount = 0
local ESPObjects = {}

local PixelFont = Font.new("rbxassetid://12187371840", Enum.FontWeight.Regular, Enum.FontStyle.Normal)

local NamesEnabled = ESP.Drawing.Names.Enabled
local DistancesEnabled = ESP.Drawing.Distances.Enabled
local HealthbarEnabled = ESP.Drawing.Healthbar.Enabled
local HealthTextEnabled = ESP.Drawing.Healthbar.HealthText
local BoxFilledEnabled = ESP.Drawing.Boxes.Filled.Enabled
local CornerEnabled = ESP.Drawing.Boxes.Corner.Enabled
local CornerThickness = ESP.Drawing.Boxes.Corner.Thickness
local CornerRGB = ESP.Drawing.Boxes.Corner.RGB
local ChamsEnabled = ESP.Drawing.Chams.Enabled
local TeamCheckEnabled = ESP.TeamCheck

local Scale = ESP.Scale
local RefHeight = ESP.ReferenceResolution.Height
local BoxWidthFactor = 2.4 * Scale
local BoxHeightFactor = 3.6 * Scale

local HealthLowRGB = ESP.Drawing.Healthbar.LowRGB
local HealthHighRGB = ESP.Drawing.Healthbar.RGB
local LocalTeam = LocalPlayer.Team
local StudsPerMeter = 3.5714285714
local MaxStuds = ESP.MaxDistance * StudsPerMeter
local MaxSqDist = MaxStuds * MaxStuds

local HealthColorCache = {}
local HealthSizeCache = {}
for Step = 0, 200 do
    local Ratio = Step / 200
    HealthColorCache[Step] = HealthLowRGB:Lerp(HealthHighRGB, Ratio)
    HealthSizeCache[Step] = UDim2.new(1, 0, Ratio, 0)
end

local DistTextCache = {}
local LayoutCache = {}

local function GetLayout(ViewportY)
    local Layout = LayoutCache[ViewportY]
    if Layout then return Layout end
    local ResScale = ViewportY / RefHeight
    Layout = {
        FontMin = math.floor(12 * Scale * ResScale + 0.5),
        FontMax = math.floor(15 * Scale * ResScale + 0.5),
        NameWidth = math.floor(220 * Scale * ResScale + 0.5),
        HPWidth = math.floor(50 * Scale * ResScale + 0.5),
        LabelPad = math.floor(4 * ResScale + 0.5),
        HBOffset = -math.floor(6 * Scale * ResScale + 0.5),
        HTOffset = -math.floor(10 * Scale * ResScale + 0.5),
        HBWidth = math.max(1, math.floor(ESP.Drawing.Healthbar.Width * ResScale + 0.5)),
        CornerT = math.max(1, math.floor(CornerThickness * ResScale + 0.5)),
        NameGap = -math.floor(ResScale + 0.5),
        DistGap = math.floor(ResScale + 0.5),
        WeaponGap = math.floor(ResScale + 0.5),
        NameSizes = {},
        HPSizes = {},
    }
    Layout.HPFontMin = Layout.FontMin - 2
    Layout.HPFontMax = Layout.FontMax - 2
    LayoutCache[ViewportY] = Layout
    return Layout
end

local function GetNameSize(Layout, Size)
    local Cached = Layout.NameSizes[Size]
    if not Cached then
        Cached = UDim2.fromOffset(Layout.NameWidth, Size + Layout.LabelPad)
        Layout.NameSizes[Size] = Cached
    end
    return Cached
end

local function GetHPSize(Layout, Size)
    local Cached = Layout.HPSizes[Size]
    if not Cached then
        Cached = UDim2.fromOffset(Layout.HPWidth, Size + Layout.LabelPad)
        Layout.HPSizes[Size] = Cached
    end
    return Cached
end

local CurrentCam = Workspace.CurrentCamera
local LastCamCF = nil
local CameraChanged = true

Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(function()
    CurrentCam = Workspace.CurrentCamera
    CameraChanged = true
end)

local CurrentLayout = GetLayout((CurrentCam and CurrentCam.ViewportSize.Y) or RefHeight)

local function Create(class, properties)
    local object = Instance.new(class)
    for property, value in pairs(properties) do
        object[property] = value
    end
    return object
end

ScreenGui = Create("ScreenGui", {
    Name = "ESPHolder",
    Parent = CoreGui,
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
})

LocalPlayer:GetPropertyChangedSignal("Team"):Connect(function()
    LocalTeam = LocalPlayer.Team
end)

local function CreateESP(Player)
    if Player == LocalPlayer or ESPObjects[Player] then return end

    local Container = {
        Player = Player,
        Shown = false,
        Character = Player.Character,
        Humanoid = nil,
        Root = nil,
        RootSizeY = 1,
        Health = 0,
        MaxHealth = 1,
        Team = Player.Team,
        IsFriend = false,
        GX = nil, GY = nil, GW = nil, GH = nil,
        LastPos = nil, LastSqDist = nil, LastFontSize = nil,
        LastDistInt = nil, LastHPInt = nil, LastStep = nil,
        Weapon = nil, LastWeapon = "",
    }

    ESPObjects[Player] = Container
    PlayerCount = PlayerCount + 1
    Container.Index = PlayerCount
    PlayerList[PlayerCount] = Container

    local Layout = CurrentLayout
    local T = Layout.CornerT

    local Anchor = Create("Frame", {
        Parent = ScreenGui,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Visible = false,
    })

    local Box = Create("Frame", {
        Parent = Anchor,
        Position = UDim2.new(0, 0, 0, 0),
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Visible = BoxFilledEnabled,
    })

    local BoxStroke = Instance.new("UIStroke")
    BoxStroke.Parent = Box
    BoxStroke.Color = ESP.Drawing.Boxes.Filled.RGB
    BoxStroke.Thickness = 1.5
    BoxStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

    local Corners = {}
    for i = 1, 8 do
        Corners[i] = Create("Frame", { Parent = Anchor, BorderSizePixel = 0, BackgroundColor3 = CornerRGB, Visible = CornerEnabled })
    end
    Corners[1].Position = UDim2.new(0,0,0,0); Corners[1].Size = UDim2.new(0.2,0,0,T)
    Corners[2].Position = UDim2.new(0,0,0,0); Corners[2].Size = UDim2.new(0,T,0.2,0)
    Corners[3].Position = UDim2.new(0.8,0,0,0); Corners[3].Size = UDim2.new(0.2,0,0,T)
    Corners[4].Position = UDim2.new(1,-T,0,0); Corners[4].Size = UDim2.new(0,T,0.2,0)
    Corners[5].Position = UDim2.new(0,0,0.8,0); Corners[5].Size = UDim2.new(0,T,0.2,0)
    Corners[6].Position = UDim2.new(0,0,1,-T); Corners[6].Size = UDim2.new(0.2,0,0,T)
    Corners[7].Position = UDim2.new(1,-T,0.8,0); Corners[7].Size = UDim2.new(0,T,0.2,0)
    Corners[8].Position = UDim2.new(0.8,0,1,-T); Corners[8].Size = UDim2.new(0.2,0,0,T)

    local HealthBackground = Create("Frame", {
        Parent = Anchor,
        Position = UDim2.new(0, Layout.HBOffset, 0, 0),
        Size = UDim2.new(0, Layout.HBWidth, 1, 0),
        BackgroundColor3 = ESP.Drawing.Healthbar.BackgroundRGB,
        BackgroundTransparency = ESP.Drawing.Healthbar.BackgroundTransparency,
        BorderSizePixel = 0,
        Visible = HealthbarEnabled,
    })

    local Healthbar = Create("Frame", {
        Parent = HealthBackground,
        AnchorPoint = Vector2.new(0, 1),
        Position = UDim2.new(0, 0, 1, 0),
        Size = UDim2.new(1, 0, 0, 0),
        BackgroundColor3 = HealthHighRGB,
        BackgroundTransparency = 0,
        BorderSizePixel = 0,
        Visible = false,
    })

    local Stroke = Instance.new("UIStroke")
    Stroke.Parent = Healthbar
    Stroke.Color = Color3.fromRGB(0, 0, 0)
    Stroke.Thickness = 0.5
    Stroke.Transparency = 0.3
    Stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

    local HealthText = Create("TextLabel", {
        Parent = Anchor,
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(0, Layout.HTOffset, 0.38, 0),
        Size = GetHPSize(Layout, Layout.HPFontMax),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = ESP.Drawing.Healthbar.HealthTextRGB,
        FontFace = PixelFont,
        TextSize = Layout.HPFontMax,
        TextStrokeTransparency = 0,
        TextStrokeColor3 = Color3.fromRGB(0, 0, 0),
        TextXAlignment = Enum.TextXAlignment.Right,
        TextYAlignment = Enum.TextYAlignment.Center,
        Visible = HealthbarEnabled and HealthTextEnabled,
    })

    local Name = Create("TextLabel", {
        Parent = Anchor,
        AnchorPoint = Vector2.new(0.5, 1),
        Position = UDim2.new(0.5, 0, 0, Layout.NameGap),
        Size = GetNameSize(Layout, Layout.FontMax),
        BackgroundTransparency = 1,
        Text = Player.Name,
        TextColor3 = ESP.Drawing.Names.RGB,   
        FontFace = PixelFont,
        TextSize = Layout.FontMax,
        TextStrokeTransparency = 0,
        TextStrokeColor3 = Color3.fromRGB(0, 0, 0),
        TextXAlignment = Enum.TextXAlignment.Center,
        TextYAlignment = Enum.TextYAlignment.Bottom,
        Visible = NamesEnabled,
    })

    local Distance = Create("TextLabel", {
        Parent = Anchor,
        AnchorPoint = Vector2.new(0.5, 0),
        Position = UDim2.new(0.5, 0, 1, Layout.DistGap),
        Size = GetNameSize(Layout, Layout.FontMax),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = ESP.Drawing.Distances.RGB,   
        FontFace = PixelFont,
        TextSize = Layout.FontMax,
        TextStrokeTransparency = 0,
        TextStrokeColor3 = Color3.fromRGB(0, 0, 0),
        TextXAlignment = Enum.TextXAlignment.Center,
        TextYAlignment = Enum.TextYAlignment.Top,
        Visible = DistancesEnabled,
    })

    local Weapon = Create("TextLabel", {
        Parent = Anchor,
        AnchorPoint = Vector2.new(0.5, 0),
        Position = UDim2.new(0.5, 0, 1, Layout.WeaponGap),
        Size = GetNameSize(Layout, Layout.FontMin),
        BackgroundTransparency = 1,
        Text = "",
        TextColor3 = Color3.fromRGB(255, 200, 100),
        FontFace = PixelFont,
        TextSize = Layout.FontMin,
        TextStrokeTransparency = 0,
        TextStrokeColor3 = Color3.fromRGB(0, 0, 0),
        TextXAlignment = Enum.TextXAlignment.Center,
        TextYAlignment = Enum.TextYAlignment.Top,
        Visible = true,
    })
    Container.Weapon = Weapon

    local Highlight = Instance.new("Highlight")
    Highlight.Name = "OutlineChamsHighlight"
    Highlight.Adornee = nil
    Highlight.FillColor = ESP.Drawing.Chams.FillRGB
    Highlight.FillTransparency = ESP.Drawing.Chams.FillTransparency
    Highlight.OutlineColor = ESP.Drawing.Chams.OutlineRGB
    Highlight.OutlineTransparency = ESP.Drawing.Chams.OutlineTransparency
    Highlight.DepthMode = ESP.Drawing.Chams.AlwaysOnTop and Enum.HighlightDepthMode.AlwaysOnTop or Enum.HighlightDepthMode.Occluded
    Highlight.Enabled = false
    Highlight.Parent = CoreGui

    Container.Highlight = Highlight
    Container.Anchor = Anchor
    Container.Corners = Corners
    Container.HealthBackground = HealthBackground
    Container.Healthbar = Healthbar
    Container.HealthText = HealthText
    Container.Name = Name
    Container.Distance = Distance

    Container.CharRemovingConn = Player.CharacterRemoving:Connect(function()
        Container.Character = nil
        Container.Humanoid = nil
        Container.Root = nil
        Container.LastPos = nil
        Container.LastSqDist = nil
        if Container.Highlight then
            Container.Highlight.Adornee = nil
            Container.Highlight.Enabled = false
        end
    end)

    Container.TeamConn = Player:GetPropertyChangedSignal("Team"):Connect(function()
        Container.Team = Player.Team
        Container.LastPos = nil
    end)

    Container.CharAddedConn = Player.CharacterAdded:Connect(function(Character)
        Container.Character = Character
        Container.Humanoid = nil
        Container.Root = nil
        Container.RootSizeY = 1
        Container.Health = 0
        Container.MaxHealth = 1
        Container.LastPos = nil
        Container.LastSqDist = nil
        Container._displayHealth = nil
        Container._displayRatio = nil
        
        task.wait(0.15)
        if Container.Highlight and ChamsEnabled then
            Container.Highlight.Adornee = Character
        end
    end)

    if Player.Character then
        Highlight.Adornee = Player.Character
    end

    task.spawn(function()
        pcall(function()
            if LocalPlayer:IsFriendsWith(Player.UserId) then
                Container.IsFriend = true
                if ESP.Options.Friendcheck then Name.Text = "F  " .. Player.Name end
            end
        end)
    end)
end

local function ResolveRoot(Container)
    if Container.Root and Container.Humanoid then return Container.Root end

    if Container.HealthConn then Container.HealthConn:Disconnect(); Container.HealthConn = nil end
    if Container.MaxHealthConn then Container.MaxHealthConn:Disconnect(); Container.MaxHealthConn = nil end

    local Character = Container.Character
    if not Character then return nil end

    local Humanoid = Character:FindFirstChildOfClass("Humanoid")
    local Root = Character:FindFirstChild("HumanoidRootPart")
    if not Humanoid or not Root then return nil end

    Container.Humanoid = Humanoid
    Container.Root = Root
    Container.RootSizeY = Root.Size.Y
    Container.Health = Humanoid.Health
    Container.MaxHealth = math.max(Humanoid.MaxHealth, 1)

    Container._displayHealth = Container.Health
    Container._displayRatio = Container.Health / Container.MaxHealth

    if Container.Highlight and ChamsEnabled and Container.Highlight.Adornee ~= Character then
        Container.Highlight.Adornee = Character
    end

    Container.HealthConn = Humanoid.HealthChanged:Connect(function(Value)
        if (Container.Health <= 0) ~= (Value <= 0) then Container.LastPos = nil end
        Container.Health = Value
    end)

    Container.MaxHealthConn = Humanoid:GetPropertyChangedSignal("MaxHealth"):Connect(function()
        Container.MaxHealth = math.max(Humanoid.MaxHealth, 1)
    end)

    return Root
end

local function Hide(Container)
    if not Container.Shown then return end
    Container.Shown = false
    Container.Anchor.Visible = false
    if Container.Highlight then Container.Highlight.Enabled = false end
end

local function Show(Container)
    if Container.Shown then return end
    Container.Shown = true
    Container.Anchor.Visible = true
    if Container.Highlight and ChamsEnabled then Container.Highlight.Enabled = true end
end

local function UpdateESP(Container, Cam, CamPos, ViewportY, Team, GeometryChanged, Layout)
    local Root = ResolveRoot(Container)
    if not Root or Container.Health <= 0 then
        Hide(Container)
        return
    end

    if TeamCheckEnabled then
        local PlayerTeam = Container.Team
        if Team ~= nil and PlayerTeam ~= nil and Team == PlayerTeam then
            Hide(Container)
            return
        end
    end

    local Position = Root.Position
    local ScreenPosition, OnScreen = Cam:WorldToViewportPoint(Position)
    if not OnScreen or ScreenPosition.Z <= 0 then
        Hide(Container)
        return
    end

    Show(Container)

    local DeltaX = CamPos.X - Position.X
    local DeltaY = CamPos.Y - Position.Y
    local DeltaZ = CamPos.Z - Position.Z
    local SqDist = DeltaX * DeltaX + DeltaY * DeltaY + DeltaZ * DeltaZ
    if SqDist > MaxSqDist then
        Hide(Container)
        return
    end

    if GeometryChanged or Position ~= Container.LastPos then
        Container.LastPos = Position

        local Z = ScreenPosition.Z
        local ScaleFactor = Container.RootSizeY * ViewportY
        local Width = math.floor(BoxWidthFactor * ScaleFactor / (Z * 2) + 0.5)
        local Height = math.floor(BoxHeightFactor * ScaleFactor / (Z * 2) + 0.5)
        local X = math.floor(ScreenPosition.X + 0.5)
        local Y = math.floor(ScreenPosition.Y + 0.5)

        if Container.GX ~= X or Container.GY ~= Y or Container.GW ~= Width or Container.GH ~= Height then
            Container.GX = X; Container.GY = Y; Container.GW = Width; Container.GH = Height
            Container.Anchor.Position = UDim2.fromOffset(X - Width // 2, Y - Height // 2)
            Container.Anchor.Size = UDim2.fromOffset(Width, Height)

            local ScaledFontSize = math.clamp((Height + 2) // 4, Layout.FontMin, Layout.FontMax)
            if Container.LastFontSize ~= ScaledFontSize then
                Container.LastFontSize = ScaledFontSize
                local HPFontSize = math.clamp(ScaledFontSize - 2, Layout.HPFontMin, Layout.HPFontMax)
                local NameSize = GetNameSize(Layout, ScaledFontSize)
                Container.Name.Size = NameSize
                Container.Name.TextSize = ScaledFontSize
                Container.Distance.Size = NameSize
                Container.Distance.TextSize = ScaledFontSize
                Container.HealthText.Size = GetHPSize(Layout, HPFontSize)
                Container.HealthText.TextSize = HPFontSize
                
                local WeaponFontSize = math.clamp(ScaledFontSize - 2, Layout.FontMin, Layout.FontMin)
                Container.Weapon.Size = GetNameSize(Layout, WeaponFontSize)
                Container.Weapon.TextSize = WeaponFontSize
                
                local currentDistGap = DistancesEnabled and (ScaledFontSize + 2) or 0
                Container.Weapon.Position = UDim2.new(0.5, 0, 1, currentDistGap)
            end
        end

        if SqDist ~= Container.LastSqDist then
            Container.LastSqDist = SqDist
            local DistInt = math.floor(math.sqrt(SqDist) / StudsPerMeter + 0.5)
            if Container.LastDistInt ~= DistInt then
                Container.LastDistInt = DistInt
                local Text = DistTextCache[DistInt] or (DistInt .. "m")
                DistTextCache[DistInt] = Text
                Container.Distance.Text = Text
            end
        end

        local char = Container.Character
        local tool = char and char:FindFirstChildOfClass("Tool")
        local weaponName = tool and tool.Name or ""
        if Container.LastWeapon ~= weaponName then
            Container.LastWeapon = weaponName
            Container.Weapon.Text = weaponName
        end
    end

    if HealthbarEnabled then
        local targetHealth = Container.Health
        local maxHealth = Container.MaxHealth

        if not Container._displayHealth then
            Container._displayHealth = targetHealth
            Container._displayRatio = targetHealth / maxHealth
        end

        if HealthTextEnabled then
            local diff = targetHealth - Container._displayHealth
            if math.abs(diff) > 0.01 then
                Container._displayHealth = Container._displayHealth + diff * 0.10
            else
                Container._displayHealth = targetHealth
            end
            local displayInt = math.floor(Container._displayHealth + 0.5)
            if Container.LastHPInt ~= displayInt then
                Container.LastHPInt = displayInt
                Container.HealthText.Text = tostring(displayInt)
            end
        end

        local targetRatio = math.clamp(targetHealth / maxHealth, 0, 1)
        local diff = targetRatio - (Container._displayRatio or targetRatio)
        if math.abs(diff) > 0.001 then
            Container._displayRatio = (Container._displayRatio or targetRatio) + diff * 0.10
        else
            Container._displayRatio = targetRatio
        end

        local Step = math.floor(Container._displayRatio * 200 + 0.5)
        if Container.LastStep ~= Step then
            Container.LastStep = Step
            Container.Healthbar.Visible = Step > 0
            Container.Healthbar.Size = HealthSizeCache[Step]
            Container.Healthbar.BackgroundColor3 = HealthColorCache[Step]
        end
    end
    Container.LastPos = Position
end

local function ApplyLayout(Container, Layout)
    local T = Layout.CornerT
    local Corners = Container.Corners
    Corners[1].Position = UDim2.new(0,0,0,0); Corners[1].Size = UDim2.new(0.2,0,0,T)
    Corners[2].Position = UDim2.new(0,0,0,0); Corners[2].Size = UDim2.new(0,T,0.2,0)
    Corners[3].Position = UDim2.new(0.8,0,0,0); Corners[3].Size = UDim2.new(0.2,0,0,T)
    Corners[4].Position = UDim2.new(1,-T,0,0); Corners[4].Size = UDim2.new(0,T,0.2,0)
    Corners[5].Position = UDim2.new(0,0,0.8,0); Corners[5].Size = UDim2.new(0,T,0.2,0)
    Corners[6].Position = UDim2.new(0,0,1,-T); Corners[6].Size = UDim2.new(0.2,0,0,T)
    Corners[7].Position = UDim2.new(1,-T,0.8,0); Corners[7].Size = UDim2.new(0,T,0.2,0)
    Corners[8].Position = UDim2.new(0.8,0,1,-T); Corners[8].Size = UDim2.new(0.2,0,0,T)

    Container.HealthBackground.Position = UDim2.new(0, Layout.HBOffset, 0, 0)
    Container.HealthBackground.Size = UDim2.new(0, Layout.HBWidth, 1, 0)
    Container.HealthText.Position = UDim2.new(0, Layout.HTOffset, 0.38, 0)
    Container.Name.Position = UDim2.new(0.5, 0, 0, Layout.NameGap)
    Container.Distance.Position = UDim2.new(0.5, 0, 1, Layout.DistGap)

    Container.LastFontSize = nil
    Container.GX, Container.GY, Container.GW, Container.GH = nil, nil, nil, nil
end

local function RemoveESP(Player)
    local Data = ESPObjects[Player]
    if not Data then return end

    if Data.CharAddedConn then Data.CharAddedConn:Disconnect() end
    if Data.CharRemovingConn then Data.CharRemovingConn:Disconnect() end
    if Data.TeamConn then Data.TeamConn:Disconnect() end
    if Data.HealthConn then Data.HealthConn:Disconnect() end
    if Data.MaxHealthConn then Data.MaxHealthConn:Disconnect() end

    if Data.Anchor then Data.Anchor:Destroy() end
    if Data.Highlight then Data.Highlight:Destroy() end

    local Index = Data.Index
    local Last = PlayerList[PlayerCount]
    if Last ~= Data then
        PlayerList[Index] = Last
        Last.Index = Index
    end

    PlayerList[PlayerCount] = nil
    PlayerCount = PlayerCount - 1
    ESPObjects[Player] = nil
end

for _, Player in ipairs(Players:GetPlayers()) do
    CreateESP(Player)
end

Players.PlayerAdded:Connect(CreateESP)
Players.PlayerRemoving:Connect(RemoveESP)

local PrevEnabled = ESP.Enabled

RunService:BindToRenderStep("ESPUpdate", Enum.RenderPriority.Camera.Value + 1, function()
    local Cam = CurrentCam
    if not Cam then return end
    Camera = Cam

    local ViewportY = Cam.ViewportSize.Y
    local Layout = GetLayout(ViewportY)
    local LayoutChanged = false

    if Layout ~= CurrentLayout then
        LayoutChanged = true
        CurrentLayout = Layout
        for Index = 1, PlayerCount do
            ApplyLayout(PlayerList[Index], Layout)
        end
    end

    if not ESP.Enabled then
        if PrevEnabled then
            PrevEnabled = false
            for Index = 1, PlayerCount do
                Hide(PlayerList[Index])
                PlayerList[Index].LastPos = nil
            end
        end
        return
    end

    PrevEnabled = true
    local CamCF = Cam.CFrame
    local GeometryChanged = false

    if CameraChanged or LayoutChanged or CamCF ~= LastCamCF then
        CameraChanged = false
        LastCamCF = CamCF
        GeometryChanged = true
    end

    local CamPos = CamCF.Position
    local Team = LocalTeam

    for Index = 1, PlayerCount do
        UpdateESP(PlayerList[Index], Cam, CamPos, ViewportY, Team, GeometryChanged, Layout)
    end
end)
