-- ==========================================================
-- Private Hub (Sleek Outer Glow/Shadow Design & Fixed Palette)
-- V2.5. Visuals Tab -> MVSD Team ESP Integrated & Config Compatible
-- ==========================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local SoundService = game:GetService("SoundService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local CollectionService = game:GetService("CollectionService")
local Stats = game:GetService("Stats")

local LocalPlayer = Players.LocalPlayer
local playerGui = LocalPlayer:WaitForChild("PlayerGui")

if playerGui:FindFirstChild("PraveteHubGUI") then
	playerGui.PraveteHubGUI:Destroy()
end
-- ==========================================================
-- PrivateHub Teleport Auto Reload (重複起動防止版)
-- ==========================================================
do
    local RELOAD_URL =
        "https://raw.githubusercontent.com/robloxbad-lol/333/refs/heads/main/README.md"

    local qtp =
        (type(queue_on_teleport) == "function" and queue_on_teleport)
        or (syn and type(syn.queue_on_teleport) == "function" and syn.queue_on_teleport)
        or (fluxus and type(fluxus.queue_on_teleport) == "function" and fluxus.queue_on_teleport)

    -- 同じExecutor環境でスクリプトを再実行しても
    -- OnTeleport / queue_on_teleport が二重三重に登録されないようにする。
    if qtp and LocalPlayer and not _G.__PrivateHubTeleportReloadInstalled then
        _G.__PrivateHubTeleportReloadInstalled = true

        local function getReloadCode()
            return string.format([[
task.wait(2)

local URL = %q

-- 同一サーバー移動でキューが重複しても、1回だけ本体を起動する。
local __phJobId = game.JobId
if __phJobId ~= "" and _G.__PrivateHubQueuedReloadJobId == __phJobId then
    return
end
if __phJobId ~= "" then
    _G.__PrivateHubQueuedReloadJobId = __phJobId
end

local source

local req =
    (type(request) == "function" and request)
    or (syn and type(syn.request) == "function" and syn.request)
    or (http and type(http.request) == "function" and http.request)
    or (fluxus and type(fluxus.request) == "function" and fluxus.request)

if req then
    local ok, response = pcall(function()
        return req({
            Url = URL,
            Method = "GET"
        })
    end)

    if ok and response then
        if type(response) == "table" then
            source = response.Body
        elseif type(response) == "string" then
            source = response
        end
    end
end

if type(source) ~= "string" or #source < 100 then
    local ok, result = pcall(function()
        return game:HttpGet(URL)
    end)

    if ok and type(result) == "string" then
        source = result
    end
end

if type(source) == "string" and #source >= 100 then
    if type(loadstring) == "function" then
        local fn, err = loadstring(source)

        if type(fn) == "function" then
            task.spawn(function()
                local runOK, runErr = pcall(fn)

                if not runOK then
                    warn("PrivateHub Auto Reload runtime error:", runErr)
                end
            end)
        else
            warn("PrivateHub Auto Reload compile error:", err)
        end
    else
        warn("PrivateHub Auto Reload: loadstring unavailable")
    end
else
    warn("PrivateHub Auto Reload: source取得失敗")
end
]], RELOAD_URL)
        end

        -- TPするたびに新しいキューを登録
        LocalPlayer.OnTeleport:Connect(function(state)
            if state == Enum.TeleportState.Started
                or state == Enum.TeleportState.InProgress
                or state == Enum.TeleportState.WaitingForServer then

                pcall(function()
                    qtp(getReloadCode())
                end)
            end
        end)

        print("PrivateHub TP Auto Reload: ON")
    else
        warn("PrivateHub Auto Reload: queue_on_teleport unavailable")
    end
end
-- ==========================================
-- 統合変数・状態管理
-- ==========================================
local SilentAimEnabled = false
local AimbotEnabled = false
local WallCheckEnabled = false
local FOV_RADIUS = 300
local FOV_Color = Color3.fromRGB(255, 255, 255)
local FOV_Rainbow = false
local FOV_Filled = false -- FOV円の内側を半透明で塗りつぶす
_G.__PrivateHubAutoShotState = _G.__PrivateHubAutoShotState or {Enabled = false, Cooldown = 0.5, LastShot = 0}

-- ESP 関連変数
local MVSD_ESP_Enabled = false
local MVSD_ESP_Color = Color3.fromRGB(0, 245, 212)
local activeTargetCount = 0

-- Server Desync Visualizer / Hitbox states (Visuals > ESP)
local ServerDesync_Enabled = false
local ServerDesync_Color = MVSD_ESP_Color
local ServerDesync_UpdateInterval = 0.05
local ServerDesync_PositionHistory = {}
local ServerDesync_FakeCharacter = nil
local ServerDesync_FakeRootPart = nil
local ServerDesync_BoxAdornment = nil
local ServerDesync_Connection = nil
local ServerDesync_Setup
local ServerDesync_ClearFake
local Hitbox_Enabled = false
local Hitbox_Size = 10
local Hitbox_Color = MVSD_ESP_Color
local Hitbox_Original = {}

-- Appearance / Player Attribute state
local Appearance_IsVip = false
local Appearance_WinStreakEnabled = false
local Appearance_WinStreakValue = 999

-- カスタムクロスヘア & ウォーターマーク変数
local CustomCrosshairEnabled = false
local CustomCrosshairRainbow = false
local customCrosshairColor = Color3.fromRGB(170, 0, 255)
local watermarkTextValue = "asahara.gg"

-- World Time 変数
local WorldTimeEnabled = false
local WorldTimeValue = 12
local originalClockTime = Lighting.ClockTime

-- Information overlay state
_G.__PrivateHubInformationState = _G.__PrivateHubInformationState or {
    Enabled = false,
    ShowFPS = true,
    ShowPing = true,
    ShowTime = true,
    ShowConfig = true,
    ShowPlayers = true
}

-- World Changer (Atmosphere & Sky) 変数
local WorldChangerEnabled = false
_G.__PHWorldChangerColor = Color3.fromRGB(150, 0, 255)
_G.__PHWeatherState = _G.__PHWeatherState or {Enabled = false, Type = "Rain", Snowflakes = {}, RainDrops = {}, LastSnow = 0, LastRain = 0, RainSound = nil}
local originalLightingState = {
	Brightness = Lighting.Brightness,
	Ambient = Lighting.Ambient,
	OutdoorAmbient = Lighting.OutdoorAmbient,
	ClockTime = Lighting.ClockTime,
	FogEnd = Lighting.FogEnd,
	FogColor = Lighting.FogColor,
	FogStart = Lighting.FogStart
}

-- Minecraft Texture 変数
local MinecraftTextureEnabled = false
local minecraftConn = nil
local TEXTURES = {Water="http://www.roblox.com/asset/?id=80347853091743", Grass="http://www.roblox.com/asset/?id=87143577736788", Stone="http://www.roblox.com/asset/?id=84294299460452", Wood="http://www.roblox.com/asset/?id=121260432319162", Brick="http://www.roblox.com/asset/?id=105695637040817", Sand="http://www.roblox.com/asset/?id=108595274378469"}
local FACES = {Enum.NormalId.Top, Enum.NormalId.Bottom, Enum.NormalId.Left, Enum.NormalId.Right, Enum.NormalId.Front, Enum.NormalId.Back}

local fov_circle = nil
local target_text = nil

pcall(function()
	fov_circle = Drawing.new("Circle")
	fov_circle.Thickness = 2
	fov_circle.Visible = false
	fov_circle.Filled = FOV_Filled
	fov_circle.Transparency = 0.20

	target_text = Drawing.new("Text")
	target_text.Size = 14
	target_text.Center = true
	target_text.Outline = true
	target_text.Visible = false
	target_text.Font = Drawing.Fonts.Monospace
end)

-- ==========================================
-- ヘルパー関数群
-- ==========================================
local function addStroke(parent, color, transparency, thickness)
	local stroke = Instance.new("UIStroke")
	stroke.Color = color
	stroke.Transparency = transparency or 0
	stroke.Thickness = thickness or 1
	stroke.ApplyStrokeMode = parent:IsA("TextLabel") or parent:IsA("TextBox") and Enum.ApplyStrokeMode.Contextual or Enum.ApplyStrokeMode.Border
	stroke.Parent = parent
	return stroke
end

local function addPadding(parent, leftOffset)
	local pad = Instance.new("UIPadding")
	pad.PaddingLeft = UDim.new(0, leftOffset)
	pad.Parent = parent
	return pad
end

local FONT_MAIN = Enum.Font.GothamMedium
local FONT_BOLD = Enum.Font.GothamBold
local FONT_MONO = Enum.Font.RobotoMono

local function playSound(soundIdNum)
	if soundIdNum and soundIdNum > 0 then
		pcall(function()
			local sound = Instance.new("Sound")
			sound.SoundId = "rbxassetid://" .. tostring(soundIdNum)
			sound.Volume = 1
			sound.Parent = SoundService

			-- 3秒固定で削除せず、音声が最後まで再生されたら削除する
		local endedConnection
		endedConnection = sound.Ended:Connect(function()
			if endedConnection then
				endedConnection:Disconnect()
				endedConnection = nil
			end
			pcall(function()
				sound:Destroy()
			end)
		end)

		sound:Play()

		-- Ended が発火しない環境用の安全策。
		-- TimeLength取得後、音声の長さ+1秒まで待ってから削除する。
		task.spawn(function()
			local deadline = os.clock() + 10
			while sound.Parent and sound.TimeLength <= 0 and os.clock() < deadline do
				task.wait(0.1)
			end

			if not sound.Parent then return end

			local length = sound.TimeLength
			if length > 0 then
				task.wait(length + 1)
				if sound.Parent then
					pcall(function()
						sound:Destroy()
					end)
				end
			end
		end)
		end)
	end
end

-- 通知システム（右上スライドイン + 自動スタック + 消えるプログレスバー）
local notificationContainer = Instance.new("ScreenGui")
notificationContainer.Name = "PrivateHubNotifications"
notificationContainer.ResetOnSpawn = false
notificationContainer.IgnoreGuiInset = true
notificationContainer.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
notificationContainer.DisplayOrder = 10000
notificationContainer.Parent = playerGui

_G.__PHNotificationSoundBox = nil

local activeNotifications = {}
local NOTIF_WIDTH = 300
local NOTIF_HEIGHT = 58
local NOTIF_GAP = 8
local NOTIF_RIGHT = 18
local NOTIF_TOP = 18
local NOTIF_LIFETIME = 2.5

local function repositionNotifications()
	for i, item in ipairs(activeNotifications) do
		if item and item.Parent then
			local y = NOTIF_TOP + (i - 1) * (NOTIF_HEIGHT + NOTIF_GAP)
			TweenService:Create(item, TweenInfo.new(0.22, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Position = UDim2.new(1, -NOTIF_RIGHT, 0, y)}):Play()
		end
	end
end

local function removeNotification(notif)
	for i = #activeNotifications, 1, -1 do
		if activeNotifications[i] == notif then
			table.remove(activeNotifications, i)
			break
		end
	end
	if notif and notif.Parent then notif:Destroy() end
	repositionNotifications()
end

local function showNotification(titleOrText, message)
	local title = "PRIVATE HUB"
	local body = titleOrText
	if message ~= nil then
		title = tostring(titleOrText)
		body = tostring(message)
	else
		body = tostring(titleOrText)
	end

	local notif = Instance.new("Frame")
	notif.Name = "Notification"
	notif.Size = UDim2.new(0, NOTIF_WIDTH, 0, NOTIF_HEIGHT)
	notif.AnchorPoint = Vector2.new(1, 0)
	notif.Position = UDim2.new(1, NOTIF_WIDTH + 25, 0, NOTIF_TOP)
	notif.BackgroundColor3 = Color3.fromRGB(17, 17, 20)
	notif.BackgroundTransparency = 0.02
	notif.BorderSizePixel = 0
	notif.ZIndex = 100
	notif.Parent = notificationContainer

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 8)
	corner.Parent = notif
	addStroke(notif, Color3.fromRGB(75, 75, 82), 0.15, 1)

	local accent = Instance.new("Frame")
	accent.Name = "Accent"
	accent.Size = UDim2.new(0, 3, 1, -10)
	accent.Position = UDim2.new(0, 6, 0, 5)
	accent.BackgroundColor3 = Color3.fromRGB(210, 140, 180)
	accent.BorderSizePixel = 0
	accent.ZIndex = 101
	accent.Parent = notif
	Instance.new("UICorner", accent).CornerRadius = UDim.new(0, 2)

	local titleLabel = Instance.new("TextLabel")
	titleLabel.Name = "Title"
	titleLabel.Size = UDim2.new(1, -32, 0, 20)
	titleLabel.Position = UDim2.new(0, 20, 0, 7)
	titleLabel.BackgroundTransparency = 1
	titleLabel.Font = FONT_BOLD
	titleLabel.Text = title
	titleLabel.TextColor3 = Color3.fromRGB(245, 245, 248)
	titleLabel.TextSize = 12
	titleLabel.TextXAlignment = Enum.TextXAlignment.Left
	titleLabel.TextTruncate = Enum.TextTruncate.AtEnd
	titleLabel.ZIndex = 101
	titleLabel.Parent = notif

	local bodyLabel = Instance.new("TextLabel")
	bodyLabel.Name = "Message"
	bodyLabel.Size = UDim2.new(1, -32, 0, 22)
	bodyLabel.Position = UDim2.new(0, 20, 0, 27)
	bodyLabel.BackgroundTransparency = 1
	bodyLabel.Font = FONT_MAIN
	bodyLabel.Text = body
	bodyLabel.TextColor3 = Color3.fromRGB(185, 185, 192)
	bodyLabel.TextSize = 11
	bodyLabel.TextXAlignment = Enum.TextXAlignment.Left
	bodyLabel.TextTruncate = Enum.TextTruncate.AtEnd
	bodyLabel.ZIndex = 101
	bodyLabel.Parent = notif

	local progressBack = Instance.new("Frame")
	progressBack.Name = "ProgressBack"
	progressBack.Size = UDim2.new(1, -18, 0, 2)
	progressBack.Position = UDim2.new(0, 9, 1, -5)
	progressBack.BackgroundColor3 = Color3.fromRGB(45, 45, 50)
	progressBack.BorderSizePixel = 0
	progressBack.ZIndex = 101
	progressBack.Parent = notif

	local progress = Instance.new("Frame")
	progress.Name = "Progress"
	progress.Size = UDim2.new(1, 0, 1, 0)
	progress.BackgroundColor3 = Color3.fromRGB(210, 140, 180)
	progress.BorderSizePixel = 0
	progress.ZIndex = 102
	progress.Parent = progressBack

	table.insert(activeNotifications, notif)
	local index = #activeNotifications
	local targetY = NOTIF_TOP + (index - 1) * (NOTIF_HEIGHT + NOTIF_GAP)

	TweenService:Create(notif, TweenInfo.new(0.35, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {Position = UDim2.new(1, -NOTIF_RIGHT, 0, targetY)}):Play()
	TweenService:Create(progress, TweenInfo.new(NOTIF_LIFETIME, Enum.EasingStyle.Linear), {Size = UDim2.new(0, 0, 1, 0)}):Play()

	task.delay(NOTIF_LIFETIME, function()
		if not notif or not notif.Parent then return end
		local out = TweenService:Create(notif, TweenInfo.new(0.3, Enum.EasingStyle.Quart, Enum.EasingDirection.In), {Position = UDim2.new(1, NOTIF_WIDTH + 25, 0, notif.Position.Y.Offset)})
		out:Play()
		out.Completed:Wait()
		removeNotification(notif)
	end)

	if _G.__PHNotificationSoundBox then
		local soundIdNum = tonumber(_G.__PHNotificationSoundBox.Text)
		playSound(soundIdNum)
	end
end


-- ScreenGuiの作成
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PraveteHubGUI"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

-- カーソル用の独立したScreenGui
local CursorGui = Instance.new("ScreenGui", game:GetService("CoreGui"))
CursorGui.Name = "MVSD_Perfect_Crosshair_v3"
CursorGui.DisplayOrder = 99999
CursorGui.ResetOnSpawn = false
CursorGui.IgnoreGuiInset = true 
CursorGui.Enabled = false

local CursorCenter = Instance.new("Frame", CursorGui)
CursorCenter.Size = UDim2.new(0, 0, 0, 0)
CursorCenter.BackgroundTransparency = 1
CursorCenter.AnchorPoint = Vector2.new(0.5, 0.5)

local RotationContainer = Instance.new("Frame", CursorCenter)
RotationContainer.BackgroundTransparency = 1
RotationContainer.Size = UDim2.new(0, 0, 0, 0)

local Lines = {}
local function CreateILine(rotation)
	-- メインの「I」部分：短め＆Minecraft風のピクセル感
	local LineLabel = Instance.new("TextLabel", RotationContainer)
	LineLabel.Size = UDim2.new(0, 45, 0, 45)
	LineLabel.AnchorPoint = Vector2.new(0.5, 0.5)
	LineLabel.BackgroundTransparency = 1
	LineLabel.Text = "I"
	LineLabel.Font = Enum.Font.SourceSansBold
	LineLabel.TextSize = 37
	LineLabel.TextColor3 = customCrosshairColor
	LineLabel.Rotation = rotation
	LineLabel.ZIndex = 2

	-- 細い黒フチ
	local LineThick = Instance.new("UIStroke", LineLabel)
	LineThick.Thickness = 1.2
	LineThick.Color = Color3.fromRGB(0, 0, 0)
	LineThick.Transparency = 0.05

	-- 発光用の薄い後ろレイヤー
	local GlowLabel = Instance.new("TextLabel", RotationContainer)
	GlowLabel.Size = LineLabel.Size
	GlowLabel.AnchorPoint = LineLabel.AnchorPoint
	GlowLabel.BackgroundTransparency = 1
	GlowLabel.Text = "I"
	GlowLabel.Font = Enum.Font.SourceSansBold
	GlowLabel.TextSize = 37
	GlowLabel.TextColor3 = customCrosshairColor
	GlowLabel.TextTransparency = 0.45
	GlowLabel.Rotation = rotation
	GlowLabel.ZIndex = 1

	local GlowStroke = Instance.new("UIStroke", GlowLabel)
	GlowStroke.Thickness = 3
	GlowStroke.Color = customCrosshairColor
	GlowStroke.Transparency = 0.7

	table.insert(Lines, {
		Label = LineLabel,
		Glow = GlowLabel,
		Rotation = rotation,
		Stroke = LineThick,
		GlowStroke = GlowStroke
	})
end

CreateILine(0)
CreateILine(90)
CreateILine(180)
CreateILine(270)

local CursorText = Instance.new("TextLabel", CursorCenter)
CursorText.Size = UDim2.new(0, 200, 0, 25)
CursorText.AnchorPoint = Vector2.new(0.5, 0)
CursorText.Position = UDim2.new(0, 0, 0, 32)
CursorText.BackgroundTransparency = 1
CursorText.TextColor3 = customCrosshairColor
CursorText.Font = Enum.Font.Arcade
CursorText.TextSize = 17 
CursorText.TextXAlignment = Enum.TextXAlignment.Center
CursorText.Text = "asahara.gg"

local TextStroke = Instance.new("UIStroke", CursorText)
TextStroke.Thickness = 1.5
TextStroke.Color = Color3.fromRGB(0, 0, 0)
TextStroke.Transparency = 0.05

-- Decorative outer frames disabled: the main frame already has its own UIStroke.
-- Keeping separate outer frames visible can leave a colored block behind when MainFrame is dragged.
local outerGlow = Instance.new("Frame")
outerGlow.Name = "OuterGlow"
outerGlow.Size = UDim2.new(0, 0, 0, 0)
outerGlow.Visible = false
outerGlow.BackgroundTransparency = 1
outerGlow.BorderSizePixel = 0
outerGlow.Parent = screenGui

local outerBorder = Instance.new("Frame")
outerBorder.Name = "OuterBorder"
outerBorder.Size = UDim2.new(0, 0, 0, 0)
outerBorder.Visible = false
outerBorder.BackgroundTransparency = 1
outerBorder.BorderSizePixel = 0
outerBorder.Parent = screenGui

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 580, 0, 800)
mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
mainFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
mainFrame.BorderSizePixel = 0
mainFrame.Visible = false
mainFrame.Parent = screenGui

local mainFrameStroke = addStroke(mainFrame, Color3.fromRGB(210, 140, 180), 0.5, 1)
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 6)

local isGuiOpen = false
local function setGuiOpen(open)
	isGuiOpen = open
	if open then
		-- Never show the old detached decorative frames.
		outerGlow.Visible = false
		outerBorder.Visible = false
		mainFrame.Visible = true
		mainFrame.Size = UDim2.new(0, 520, 0, 720)
		mainFrame.BackgroundTransparency = 1

		for _, child in ipairs(mainFrame:GetChildren()) do
			if child:IsA("GuiObject") and child.Name ~= "PaletteFrame" then
				child.Visible = false
			end
		end

		local tween = TweenService:Create(mainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
			Size = UDim2.new(0, 580, 0, 800),
			BackgroundTransparency = 0
		})
		tween:Play()
		tween.Completed:Connect(function()
			for _, child in ipairs(mainFrame:GetChildren()) do
				if child:IsA("GuiObject") and child.Name ~= "PaletteFrame" then
					child.Visible = true
				end
			end
		end)
	else
		local tween = TweenService:Create(mainFrame, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
			Size = UDim2.new(0, 520, 0, 720),
			BackgroundTransparency = 1
		})
		tween:Play()
		tween.Completed:Connect(function()
			outerGlow.Visible = false
			outerBorder.Visible = false
			mainFrame.Visible = false
			mainFrame.Size = UDim2.new(0, 580, 0, 800)
			mainFrame.BackgroundTransparency = 0
		end)
	end
end

local toggleIconBtn = Instance.new("ImageButton")
toggleIconBtn.Name = "ToggleIconButton"
toggleIconBtn.Size = UDim2.new(0, 45, 0, 45)
toggleIconBtn.Position = UDim2.new(0, 20, 0.5, -22.5)
toggleIconBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
toggleIconBtn.BorderSizePixel = 0
toggleIconBtn.Parent = screenGui
Instance.new("UICorner", toggleIconBtn).CornerRadius = UDim.new(0, 8)
addStroke(toggleIconBtn, Color3.fromRGB(80, 80, 85), 0, 1)

local ANIM_IDS = {
	"rbxassetid://111771404021359",
	"rbxassetid://107809476024966",
	"rbxassetid://86815925625320",
	"rbxassetid://106479523523423",
	"rbxassetid://115909298943926"
}

task.spawn(function()
	local idx = 1
	while true do
		pcall(function()
			toggleIconBtn.Image = ANIM_IDS[idx]
		end)
		idx = idx + 1
		if idx > #ANIM_IDS then idx = 1 end
		task.wait(0.12)
	end
end)

local iconDragging, iconDragInput, iconDragStart, iconStartPos
toggleIconBtn.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		iconDragging = true
		iconDragStart = input.Position
		iconStartPos = toggleIconBtn.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				iconDragging = false
			end
		end)
	end
end)

toggleIconBtn.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		iconDragInput = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if input == iconDragInput and iconDragging then
		local delta = input.Position - iconDragStart
		toggleIconBtn.Position = UDim2.new(iconStartPos.X.Scale, iconStartPos.X.Offset + delta.X, iconStartPos.Y.Scale, iconStartPos.Y.Offset + delta.Y)
	end
end)

toggleIconBtn.MouseButton1Click:Connect(function()
	if not iconDragging then
		setGuiOpen(not isGuiOpen)
	end
end)

local topBar = Instance.new("Frame")
topBar.Name = "TopBar"
topBar.Size = UDim2.new(1, 0, 0, 40)
topBar.BackgroundColor3 = Color3.fromRGB(15, 15, 17)
topBar.BorderSizePixel = 0
topBar.Parent = mainFrame
Instance.new("UICorner", topBar).CornerRadius = UDim.new(0, 6)

local fixBar = Instance.new("Frame")
fixBar.Size = UDim2.new(1, 0, 0, 2)
fixBar.Position = UDim2.new(0, 0, 1, -2)
fixBar.BackgroundColor3 = Color3.fromRGB(210, 140, 180)
fixBar.BorderSizePixel = 0
fixBar.Parent = topBar

local titleLabel = Instance.new("TextLabel")
titleLabel.Name = "DynamicText"
titleLabel.Size = UDim2.new(0, 160, 1, 0)
titleLabel.Position = UDim2.new(0, 16, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Font = FONT_BOLD
titleLabel.Text = "PRIVATE HUB"
titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
titleLabel.TextSize = 14
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Parent = topBar

local glitchLabel = Instance.new("TextLabel")
glitchLabel.Size = UDim2.new(0, 100, 1, 0)
glitchLabel.Position = UDim2.new(0, 120, 0, 0)
glitchLabel.BackgroundTransparency = 1
glitchLabel.Font = FONT_MONO
glitchLabel.Text = "[0x00]"
glitchLabel.TextColor3 = Color3.fromRGB(210, 140, 180)
glitchLabel.TextSize = 13
glitchLabel.TextXAlignment = Enum.TextXAlignment.Left
glitchLabel.Parent = topBar

local containerArea = Instance.new("Frame")
containerArea.Name = "ContainerArea"
containerArea.Size = UDim2.new(1, -20, 1, -60)
containerArea.Position = UDim2.new(0, 10, 0, 50)
containerArea.BackgroundColor3 = Color3.fromRGB(14, 14, 16)
containerArea.BorderSizePixel = 0
containerArea.Parent = mainFrame
Instance.new("UICorner", containerArea).CornerRadius = UDim.new(0, 6)
addStroke(containerArea, Color3.fromRGB(40, 40, 45), 0, 1)

local tabMenu = Instance.new("Frame")
tabMenu.Size = UDim2.new(0, 130, 1, -16)
tabMenu.Position = UDim2.new(0, 8, 0, 8)
tabMenu.BackgroundColor3 = Color3.fromRGB(12, 12, 14)
tabMenu.BorderSizePixel = 0
tabMenu.Parent = containerArea
Instance.new("UICorner", tabMenu).CornerRadius = UDim.new(0, 6)
addStroke(tabMenu, Color3.fromRGB(30, 30, 35), 0, 1)

local contentArea = Instance.new("Frame")
contentArea.Size = UDim2.new(1, -146, 1, -16)
contentArea.Position = UDim2.new(0, 142, 0, 8)
contentArea.BackgroundColor3 = Color3.fromRGB(12, 12, 14)
contentArea.BorderSizePixel = 0
contentArea.ClipsDescendants = true
contentArea.Parent = containerArea
Instance.new("UICorner", contentArea).CornerRadius = UDim.new(0, 6)
addStroke(contentArea, Color3.fromRGB(30, 30, 35), 0, 1)

local tabNames = {"Main", "Visuals", "Misc", "Settings"}
local tabButtons = {}
local tabPages = {}
local currentActiveTab = "Main"

for i, name in ipairs(tabNames) do
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, -16, 0, 32)
	btn.Position = UDim2.new(0, 8, 0, 10 + (i - 1) * 38)
	btn.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
	btn.BorderSizePixel = 0
	btn.Font = FONT_MAIN
	btn.Text = name
	btn.TextColor3 = Color3.fromRGB(140, 140, 145)
	btn.TextSize = 13
	btn.Parent = tabMenu
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
	addStroke(btn, Color3.fromRGB(40, 40, 45), 0.5, 1)
	
	local page = Instance.new("Frame")
	page.Size = UDim2.new(1, 0, 1, 0)
	page.BackgroundTransparency = 1
	page.Visible = false
	page.Parent = contentArea
	
	tabButtons[name] = btn
	tabPages[name] = page
	
	btn.MouseButton1Click:Connect(function()
		if currentActiveTab == name then return end
		local oldTab = currentActiveTab
		currentActiveTab = name
		
		for tName, tBtn in pairs(tabButtons) do
			local isActive = (tName == name)
			TweenService:Create(tBtn, TweenInfo.new(0.2), {
				TextColor3 = isActive and Color3.fromRGB(240, 200, 210) or Color3.fromRGB(140, 140, 145),
				BackgroundColor3 = isActive and Color3.fromRGB(28, 28, 32) or Color3.fromRGB(20, 20, 23)
			}):Play()
		end
		
		local newPage = tabPages[name]
		local prevPage = tabPages[oldTab]
		
		newPage.Visible = true
		newPage.Position = UDim2.new(0, 30, 0, 0)
		newPage.BackgroundTransparency = 1
		
		TweenService:Create(newPage, TweenInfo.new(0.2, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
			Position = UDim2.new(0, 0, 0, 0)
		}):Play()
		
		if prevPage then
			local outTween = TweenService:Create(prevPage, TweenInfo.new(0.15, Enum.EasingStyle.Quart, Enum.EasingDirection.In), {
				Position = UDim2.new(0, -30, 0, 0)
			})
			outTween:Play()
			outTween.Completed:Connect(function()
				if currentActiveTab ~= oldTab then
					prevPage.Visible = false
					prevPage.Position = UDim2.new(0, 0, 0, 0)
				end
			end)
		end
	end)
end

tabPages["Main"].Visible = true
tabButtons["Main"].TextColor3 = Color3.fromRGB(240, 200, 210)
tabButtons["Main"].BackgroundColor3 = Color3.fromRGB(28, 28, 32)

-- ==========================================
-- プレイヤー移動設定・機能統合
-- ==========================================
local ModSpeedEnabled = false -- ★ スピード変更のオンオフ用変数
local ModSpeedValue = 100
local ModInfJumpEnabled = false
local ModNoclipEnabled = false

-- Spin Control state (use _G to avoid adding another top-level local register)
_G.__PrivateHubSpinState = _G.__PrivateHubSpinState or {Enabled = false, Speed = 10}

-- ==========================================
-- Match / Auto Queue state
-- ==========================================
_G.__PrivateHubMatchState = _G.__PrivateHubMatchState or {
    Enabled = false,
    Mode = "1v1"
}

_G.__PrivateHubAutoVoteMapState = _G.__PrivateHubAutoVoteMapState or {
    Enabled = false,
    LastVote = 0,
    LastButton = nil,
    Rank1 = "Barn",
    Rank2 = "None",
    Rank3 = "None"
}
_G.__PrivateHubAutoVoteMapState.Rank1 = _G.__PrivateHubAutoVoteMapState.Rank1 or "Barn"
_G.__PrivateHubAutoVoteMapState.Rank2 = _G.__PrivateHubAutoVoteMapState.Rank2 or "None"
_G.__PrivateHubAutoVoteMapState.Rank3 = _G.__PrivateHubAutoVoteMapState.Rank3 or "None"

-- ==========================================
-- Abilities / Knife / Cooldown / Dash
-- ==========================================
local KnifeSettings = {
    Enabled = false,
    Value = 150
}

local CustomCooldownEnabled = false
local CurrentCooldownValue = 2.5

local DashEnabled = false
local DASH_SPEED = 60
local DASH_TIME = 0.2
local DASH_COOLDOWN = 1.0
local isDashable = true
local currentDashCharacter = nil
local currentDashHumanoid = nil
local currentDashRoot = nil
local currentDashUpperTorso = nil

-- Additional Ability states
local UnlockMovementEnabled = false
local unlockMovementConnection = nil

local InvisibilityEnabled = false
local invisChair = nil

local INVISIBILITY_POSITION = Vector3.new(-25.95, 500000, 3537.55)

local function setUnlockMovementEnabled(enabled)
    UnlockMovementEnabled = enabled

    if unlockMovementConnection then
        unlockMovementConnection:Disconnect()
        unlockMovementConnection = nil
    end

    if enabled then
        unlockMovementConnection = RunService.Heartbeat:Connect(function()
            local character = LocalPlayer.Character
            local root = character and character:FindFirstChild("HumanoidRootPart")
            if root and root:IsA("BasePart") then
                root.Anchored = false
                local hum = character:FindFirstChildOfClass("Humanoid")
                if hum and hum.WalkSpeed <= 0 then
                    hum.WalkSpeed = 16
                end
            end
        end)
    end
end

local function setCharacterTransparency(character, transparency)
    for _, descendant in ipairs(character:GetDescendants()) do
        if descendant:IsA("BasePart") or descendant:IsA("Decal") then
            descendant.Transparency = transparency
        end
    end
end

local function clearInvisibility()
    if invisChair then
        pcall(function()
            invisChair:Destroy()
        end)
        invisChair = nil
    else
        local oldChair = Workspace:FindFirstChild("invischair")
        if oldChair then
            pcall(function()
                oldChair:Destroy()
            end)
        end
    end

    if LocalPlayer.Character then
        setCharacterTransparency(LocalPlayer.Character, 0)
    end
end

local function setInvisibilityEnabled(enabled)
    -- 同じInvisibility処理をConfig読込/起動時に二重実行しない。
    -- 既にinvischairが存在して有効なら、そのまま維持する。
    if enabled and InvisibilityEnabled and invisChair and invisChair.Parent then
        return
    end

    InvisibilityEnabled = enabled

    if not enabled then
        clearInvisibility()
        return
    end

    local character = LocalPlayer.Character
    if not character then
        return
    end

    local root = character:FindFirstChild("HumanoidRootPart")
    local torso = character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso")
    if not root or not torso then
        return
    end

    local savedCFrame = root.CFrame

    clearInvisibility()

    pcall(function()
        character:MoveTo(INVISIBILITY_POSITION)
        task.wait(0.15)

        local seat = Instance.new("Seat")
        seat.Name = "invischair"
        seat.Anchored = false
        seat.CanCollide = false
        seat.Transparency = 1
        seat.Position = INVISIBILITY_POSITION
        seat.Parent = Workspace

        local weld = Instance.new("Weld")
        weld.Part0 = seat
        weld.Part1 = torso
        weld.Parent = seat

        invisChair = seat

        task.wait()
        seat.CFrame = savedCFrame

        setCharacterTransparency(character, 0.5)
    end)
end

LocalPlayer.CharacterAdded:Connect(function(character)
    clearInvisibility()
    task.wait(0.2)
    if InvisibilityEnabled then
        setInvisibilityEnabled(true)
    end
end)

-- Appearance: IsVip / WinStreak
local function applyAppearanceAttributes()
    pcall(function()
        LocalPlayer:SetAttribute("IsVip", Appearance_IsVip)
        if Appearance_WinStreakEnabled then
            LocalPlayer:SetAttribute("WinStreak", Appearance_WinStreakValue)
        end
    end)
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.5)
    applyAppearanceAttributes()
end)

-- World Changer: selected palette color directly controls the atmosphere.
local function applyWorldChanger()
	local c = _G.__PHWorldChangerColor or Color3.fromRGB(150, 0, 255)
	local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
	if not atmosphere then
		atmosphere = Instance.new("Atmosphere")
		atmosphere.Parent = Lighting
	end

	local r = math.clamp(c.R, 0, 1)
	local g = math.clamp(c.G, 0, 1)
	local b = math.clamp(c.B, 0, 1)
	local luminance = (r * 0.299) + (g * 0.587) + (b * 0.114)
	local ambient = Color3.new(r * 0.55, g * 0.55, b * 0.55)
	local outdoor = Color3.new(r * 0.32, g * 0.32, b * 0.32)

	Lighting.Brightness = 1.5 + (luminance * 1.5)
	Lighting.Ambient = ambient
	Lighting.OutdoorAmbient = outdoor
	Lighting.FogColor = c
	Lighting.FogStart = 0
	Lighting.FogEnd = 1000 + (luminance * 4000)

	atmosphere.Density = 0.25 + ((1 - luminance) * 0.25)
	atmosphere.Haze = 1 + ((1 - luminance) * 4)
	atmosphere.Color = c
	atmosphere.Decay = Color3.new(r * 0.45, g * 0.45, b * 0.45)
end

RunService.Stepped:Connect(function()
	local char = LocalPlayer.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	
	if char and hum then
		-- 修正: オンのときだけWalkSpeedを変更し、オフのときはデフォルトに戻す処理を行わない
		if ModSpeedEnabled then
			hum.WalkSpeed = ModSpeedValue
		end
		
		if ModNoclipEnabled then
			for _, part in pairs(char:GetChildren()) do
				if part:IsA("BasePart") then part.CanCollide = false end
			end
		end
		-- Spin Control
		if _G.__PrivateHubSpinState.Enabled then
			local root = char:FindFirstChild("HumanoidRootPart")
			if root and root:IsA("BasePart") then
				root.CFrame = root.CFrame * CFrame.Angles(0, math.rad(_G.__PrivateHubSpinState.Speed), 0)
			end
		end
	end

	if WorldTimeEnabled then Lighting.ClockTime = WorldTimeValue end
	if WorldChangerEnabled then applyWorldChanger() end
end)
-- Weather runtime (client-side anchored particles: stable rain instead of physics parts)
task.spawn(function()
    while true do
        local ws = _G.__PHWeatherState
        if ws and ws.Enabled and ws.Type == "Rain" then
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                local now = tick()
                if now - (ws.LastRain or 0) >= 0.04 then
                    ws.LastRain = now
                    for _ = 1, 32 do
                        local p = Instance.new("Part")
                        p.Size = Vector3.new(0.04, 3.8, 0.04)
                        p.Color = Color3.fromRGB(180, 185, 195)
                        p.Material = Enum.Material.SmoothPlastic
                        p.Transparency = 0.25
                        p.CanCollide = false
                        p.CanTouch = false
                        p.CanQuery = false
                        p.Anchored = true
                        local pos = hrp.Position + Vector3.new(math.random(-70,70), math.random(32,58), math.random(-70,70))
                        p.CFrame = CFrame.new(pos)
                        p.Parent = workspace
                        table.insert(ws.RainDrops, {Part = p, Pos = pos, Speed = math.random(170,220)})
                    end
                end
            end
        end
        task.wait(0.03)
    end
end)

RunService.Heartbeat:Connect(function(dt)
    local ws = _G.__PHWeatherState
    if not ws then return end

    if ws.Enabled and ws.Type == "Rain" then
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if hrp then
            for i = #ws.RainDrops, 1, -1 do
                local d = ws.RainDrops[i]
                local p = d and d.Part
                if p and p.Parent then
                    d.Pos = d.Pos + Vector3.new(0, -(d.Speed or 190) * dt, 0)
                    p.CFrame = CFrame.new(d.Pos)
                    if d.Pos.Y <= hrp.Position.Y - 6 then
                        p:Destroy()
                        table.remove(ws.RainDrops, i)
                    end
                else
                    table.remove(ws.RainDrops, i)
                end
            end
        end
    else
        for i = #ws.RainDrops, 1, -1 do
            local d = ws.RainDrops[i]
            if d and d.Part then pcall(function() d.Part:Destroy() end) end
            table.remove(ws.RainDrops, i)
        end
    end

    if not ws.Enabled or ws.Type ~= "Snow" then
        for i = #ws.Snowflakes, 1, -1 do
            local d = ws.Snowflakes[i]
            if d and d.Part then pcall(function() d.Part:Destroy() end) end
            table.remove(ws.Snowflakes, i)
        end
        return
    end

    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local now = tick()
    if now - (ws.LastSnow or 0) >= 0.08 then
        ws.LastSnow = now
        for _ = 1, 3 do
            local p = Instance.new("Part")
            p.Shape = Enum.PartType.Ball
            p.Size = Vector3.new(0.25, 0.25, 0.25)
            p.Color = Color3.fromRGB(255, 255, 255)
            p.Material = Enum.Material.Neon
            p.Transparency = 0.3
            p.CanCollide = false
            p.CanTouch = false
            p.CanQuery = false
            p.Anchored = true
            local pos = hrp.Position + Vector3.new(math.random(-60,60), math.random(35,50), math.random(-60,60))
            p.Position = pos
            p.Parent = workspace
            table.insert(ws.Snowflakes, {Part=p, CurrentPos=pos, FallSpeed=math.random(4,7)/10, WindSpeedX=math.random(1,4)/10, WindSpeedZ=math.random(1,4)/10, DriftRange=math.random(30,50)/10, RandomOffset=math.random()*100, LifeTime=0})
        end
    end
    local groundY = hrp.Position.Y - 5
    for i = #ws.Snowflakes, 1, -1 do
        local d = ws.Snowflakes[i]
        local p = d and d.Part
        if p and p.Parent then
            d.LifeTime = d.LifeTime + dt
            d.CurrentPos = d.CurrentPos + Vector3.new(math.sin(d.LifeTime*d.WindSpeedX+d.RandomOffset)*d.DriftRange*dt, -d.FallSpeed*dt*10, math.cos(d.LifeTime*d.WindSpeedZ+d.RandomOffset)*d.DriftRange*dt)
            p.Position = d.CurrentPos
            if d.CurrentPos.Y <= groundY or d.LifeTime > 7 then p:Destroy(); table.remove(ws.Snowflakes,i) end
        else
            table.remove(ws.Snowflakes,i)
        end
    end
end)

-- UI要素作成ヘルパー関数群
--------------------------------------------------
local CheckboxSetters = {}
local SliderSetters = {}
local ColorBtnSetters = {}

local function createCheckboxToggle(parent, text, yPos, callback)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -20, 0, 30)
	row.Position = UDim2.new(0, 10, 0, yPos)
	row.BackgroundTransparency = 1
	row.Parent = parent

	local lbl = Instance.new("TextLabel")
	lbl.Name = "DynamicText"
	lbl.Size = UDim2.new(0.7, 0, 1, 0)
	lbl.BackgroundTransparency = 1
	lbl.Font = FONT_MAIN
	lbl.Text = text
	lbl.TextColor3 = Color3.fromRGB(200, 200, 205)
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Parent = row

	local box = Instance.new("TextButton")
	box.Size = UDim2.new(0, 22, 0, 22)
	box.Position = UDim2.new(1, -22, 0.5, -11)
	box.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
	box.BorderSizePixel = 0
	box.Font = FONT_BOLD
	box.Text = ""
	box.TextColor3 = Color3.fromRGB(255, 255, 255)
	box.TextSize = 14
	box.Parent = row
	Instance.new("UICorner", box).CornerRadius = UDim.new(0, 4)
	local stroke = addStroke(box, Color3.fromRGB(50, 50, 55), 0, 1)

	local isChecked = false
	
	local function updateState(state, suppressNotify)
		isChecked = state
		if isChecked then
			box.Text = "✓"
			box.BackgroundColor3 = Color3.fromRGB(210, 140, 180)
			stroke.Color = Color3.fromRGB(210, 140, 180)
		else
			box.Text = ""
			box.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
			stroke.Color = Color3.fromRGB(50, 50, 55)
		end
		
		callback(isChecked)
		if not suppressNotify then
			showNotification(text .. ": " .. (isChecked and "ON" or "OFF"))
		end
	end

	box.MouseButton1Click:Connect(function()
		updateState(not isChecked)
	end)

	return updateState
end

local function createSliderRow(parent, text, minVal, maxVal, defaultVal, yPos, callback)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -20, 0, 45)
	row.Position = UDim2.new(0, 10, 0, yPos)
	row.BackgroundTransparency = 1
	row.Parent = parent

	local lbl = Instance.new("TextLabel")
	lbl.Name = "DynamicText"
	lbl.Size = UDim2.new(0.6, 0, 0, 20)
	lbl.BackgroundTransparency = 1
	lbl.Font = FONT_MAIN
	lbl.Text = text
	lbl.TextColor3 = Color3.fromRGB(200, 200, 205)
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Parent = row

	local valLbl = Instance.new("TextLabel")
	valLbl.Name = "DynamicText"
	valLbl.Size = UDim2.new(0.4, 0, 0, 20)
	valLbl.Position = UDim2.new(0.6, 0, 0, 0)
	valLbl.BackgroundTransparency = 1
	valLbl.Font = FONT_MAIN
	valLbl.Text = tostring(defaultVal)
	valLbl.TextColor3 = Color3.fromRGB(180, 180, 185)
	valLbl.TextSize = 12
	valLbl.TextXAlignment = Enum.TextXAlignment.Right
	valLbl.Parent = row

	local track = Instance.new("Frame")
	track.Size = UDim2.new(1, 0, 0, 6)
	track.Position = UDim2.new(0, 0, 0, 28)
	track.BackgroundColor3 = Color3.fromRGB(25, 25, 28)
	track.BorderSizePixel = 0
	track.Parent = row
	Instance.new("UICorner", track).CornerRadius = UDim.new(0, 3)

	local fill = Instance.new("Frame")
	fill.Size = UDim2.new((defaultVal - minVal) / (maxVal - minVal), 0, 1, 0)
	fill.BackgroundColor3 = Color3.fromRGB(210, 140, 180)
	fill.BorderSizePixel = 0
	fill.Parent = track
	Instance.new("UICorner", fill).CornerRadius = UDim.new(0, 3)

	local currentVal = defaultVal

	local function setValue(val)
		val = math.clamp(val, minVal, maxVal)
		if text == "WinStreak Value" then
			currentVal = math.floor(val + 0.5)
			valLbl.Text = tostring(currentVal)
		else
			currentVal = math.floor(val * 10 + 0.5) / 10
			valLbl.Text = string.format("%.1f", currentVal)
		end
		fill.Size = UDim2.new((currentVal - minVal) / (maxVal - minVal), 0, 1, 0)
		callback(currentVal)
	end

	local draggingSlider = false
	track.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			draggingSlider = true
			local pos = math.clamp((input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
			setValue(minVal + (maxVal - minVal) * pos)
		end
	end)

	track.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			draggingSlider = false
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local pos = math.clamp((input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
			setValue(minVal + (maxVal - minVal) * pos)
		end
	end)

	return setValue
end

local function createColorPreviewRowInParent(parent, name, defaultColor, yPos)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -20, 0, 30)
	row.Position = UDim2.new(0, 10, 0, yPos)
	row.BackgroundTransparency = 1
	row.Parent = parent

	local lbl = Instance.new("TextLabel")
	lbl.Name = "DynamicText"
	lbl.Size = UDim2.new(0.5, 0, 1, 0)
	lbl.BackgroundTransparency = 1
	lbl.Font = FONT_MAIN
	lbl.Text = name
	lbl.TextColor3 = Color3.fromRGB(200, 200, 205)
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Parent = row

	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0.5, 0, 1, 0)
	btn.Position = UDim2.new(0.5, 0, 0, 0)
	btn.BackgroundColor3 = defaultColor
	btn.BorderSizePixel = 0
	btn.Font = FONT_MAIN
	btn.Text = ""
	btn.Parent = row
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
	addStroke(btn, Color3.fromRGB(80, 80, 85), 0.5, 1)

	return btn
end

--------------------------------------------------
-- Main タブ内容構築
--------------------------------------------------
local mainPage = tabPages["Main"]
local mainScroll = Instance.new("ScrollingFrame")
mainScroll.Size = UDim2.new(1, 0, 1, 0)
mainScroll.BackgroundTransparency = 1
mainScroll.BorderSizePixel = 0
mainScroll.CanvasSize = UDim2.new(0, 0, 0, 3295)
mainScroll.ScrollBarThickness = 4
mainScroll.ScrollingDirection = Enum.ScrollingDirection.Y
mainScroll.ScrollingEnabled = true
mainScroll.Active = true
mainScroll.CanvasPosition = Vector2.new(0, 0)
mainScroll.Parent = mainPage

local playerSection = Instance.new("Frame")
playerSection.Size = UDim2.new(0.92, 0, 0, 280) -- Spin Control 分を追加
playerSection.Position = UDim2.new(0.04, 0, 0, 15)
playerSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
playerSection.BorderSizePixel = 0
playerSection.Parent = mainScroll
Instance.new("UICorner", playerSection).CornerRadius = UDim.new(0, 6)
addStroke(playerSection, Color3.fromRGB(40, 40, 45), 0, 1)

local playerTitle = Instance.new("TextLabel")
playerTitle.Name = "DynamicText"
playerTitle.Size = UDim2.new(1, 0, 0, 35)
playerTitle.BackgroundTransparency = 1
playerTitle.Font = FONT_BOLD
playerTitle.Text = "Player"
playerTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
playerTitle.TextSize = 13
playerTitle.Parent = playerSection

-- ★ WalkSpeed のオンオフ用トグルを追加
CheckboxSetters["WalkSpeedToggle"] = createCheckboxToggle(playerSection, "Enable WalkSpeed", 35, function(enabled)
	ModSpeedEnabled = enabled
end)

SliderSetters["WalkSpeed"] = createSliderRow(playerSection, "WalkSpeed Value", 16, 350, 100, 75, function(val)
	ModSpeedValue = val
end)

CheckboxSetters["InfJump"] = createCheckboxToggle(playerSection, "Infinite Air Jump", 130, function(enabled)
	ModInfJumpEnabled = enabled
end)

CheckboxSetters["Noclip"] = createCheckboxToggle(playerSection, "Wall Noclip", 170, function(enabled)
	ModNoclipEnabled = enabled
	if not enabled and LocalPlayer.Character then
		for _, part in pairs(LocalPlayer.Character:GetChildren()) do
			if part:IsA("BasePart") then
				part.CanCollide = true
			end
		end
	end
end)

-- Spin Control
CheckboxSetters["Spin"] = createCheckboxToggle(playerSection, "Spin Control", 210, function(enabled)
	_G.__PrivateHubSpinState.Enabled = enabled
end)

SliderSetters["SpinSpeed"] = createSliderRow(playerSection, "Spin Speed", 1, 100, 10, 245, function(val)
	_G.__PrivateHubSpinState.Speed = val
end)

-- Infinite Air Jump
-- JumpRequest works with keyboard/controller/mobile jump input.
local infiniteJumpConnection = UserInputService.JumpRequest:Connect(function()
    if not ModInfJumpEnabled then return end

    local character = LocalPlayer.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if humanoid and humanoid.Health > 0 then
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
end)

-- ==========================================
-- Match
-- ==========================================
local matchSection = Instance.new("Frame")
matchSection.Name = "MatchSection"
    matchSection.ClipsDescendants = false
matchSection.Size = UDim2.new(0.92, 0, 0, 620)
matchSection.Position = UDim2.new(0.04, 0, 0, 305)
matchSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
matchSection.BorderSizePixel = 0
matchSection.Parent = mainScroll
Instance.new("UICorner", matchSection).CornerRadius = UDim.new(0, 6)
addStroke(matchSection, Color3.fromRGB(40, 40, 45), 0, 1)

local matchTitle = Instance.new("TextLabel")
matchTitle.Name = "DynamicText"
matchTitle.Size = UDim2.new(1, 0, 0, 35)
matchTitle.BackgroundTransparency = 1
matchTitle.Font = FONT_BOLD
matchTitle.Text = "Match"
matchTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
matchTitle.TextSize = 13
matchTitle.Parent = matchSection

local matchModeBtn = Instance.new("TextButton")
matchModeBtn.Name = "MatchModeDropdown"
matchModeBtn.Size = UDim2.new(1, -20, 0, 32)
matchModeBtn.Position = UDim2.new(0, 10, 0, 40)
matchModeBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
matchModeBtn.BorderSizePixel = 0
matchModeBtn.Font = FONT_MAIN
matchModeBtn.Text = "Mode: " .. tostring(_G.__PrivateHubMatchState.Mode)
matchModeBtn.TextColor3 = Color3.fromRGB(220, 220, 225)
matchModeBtn.TextSize = 12
matchModeBtn.TextXAlignment = Enum.TextXAlignment.Left
matchModeBtn.Parent = matchSection
Instance.new("UICorner", matchModeBtn).CornerRadius = UDim.new(0, 4)
addPadding(matchModeBtn, 10)
addStroke(matchModeBtn, Color3.fromRGB(50, 50, 55), 0, 1)

local matchDropdown = Instance.new("ScrollingFrame")
matchDropdown.Name = "ModeList"
matchDropdown.Size = UDim2.new(1, -20, 0, 92)
matchDropdown.Position = UDim2.new(0, 10, 0, 74)
matchDropdown.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
matchDropdown.BorderSizePixel = 0
matchDropdown.ScrollBarThickness = 2
matchDropdown.CanvasSize = UDim2.new(0, 0, 0, 4 * 28)
matchDropdown.Visible = false
matchDropdown.ZIndex = 20
matchDropdown.Parent = matchSection
Instance.new("UICorner", matchDropdown).CornerRadius = UDim.new(0, 4)
addStroke(matchDropdown, Color3.fromRGB(50, 50, 55), 0, 1)

local matchModes = {"1v1", "2v2", "3v3", "4v4"}

_G.__PrivateHubMatchSetMode = function(mode)
    if mode ~= "1v1" and mode ~= "2v2" and mode ~= "3v3" and mode ~= "4v4" then
        mode = "1v1"
    end
    _G.__PrivateHubMatchState.Mode = mode
    matchModeBtn.Text = "Mode: " .. mode
    matchDropdown.Visible = false
end

for i, modeName in ipairs(matchModes) do
    local modeBtn = Instance.new("TextButton")
    modeBtn.Size = UDim2.new(1, 0, 0, 28)
    modeBtn.Position = UDim2.new(0, 0, 0, (i - 1) * 28)
    modeBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
    modeBtn.BorderSizePixel = 0
    modeBtn.Font = FONT_MAIN
    modeBtn.Text = modeName
    modeBtn.TextColor3 = Color3.fromRGB(205, 205, 210)
    modeBtn.TextSize = 12
    modeBtn.TextXAlignment = Enum.TextXAlignment.Left
    modeBtn.ZIndex = 21
    modeBtn.Parent = matchDropdown
    addPadding(modeBtn, 10)

    modeBtn.MouseButton1Click:Connect(function()
        _G.__PrivateHubMatchSetMode(modeName)
    end)
end

matchModeBtn.MouseButton1Click:Connect(function()
    matchDropdown.Visible = not matchDropdown.Visible
end)

CheckboxSetters["AutoMatch"] = createCheckboxToggle(matchSection, "Auto Re-Queue", 145, function(enabled)
    _G.__PrivateHubMatchState.Enabled = enabled
end)

-- ==========================================================
-- Auto Match / Target Snipe (DuelRing)
-- 元の Auto Match & Snipe Panel を Match 項目へ統合
-- ==========================================================
do
    _G.__PrivateHubMatchTargetState = _G.__PrivateHubMatchTargetState or {
        AutoMatchEnabled = false,
        TargetSnipeEnabled = false,
        TargetName = ""
    }

    local targetState = _G.__PrivateHubMatchTargetState

    CheckboxSetters["MatchRingAuto"] = createCheckboxToggle(matchSection, "Auto Match (Ring)", 181, function(enabled)
        targetState.AutoMatchEnabled = enabled == true
        if enabled then
            showNotification("Auto Match", "Ring detection enabled")
        end
    end)

    local targetBtn = Instance.new("TextButton")
    targetBtn.Name = "MatchTargetSelector"
    targetBtn.Size = UDim2.new(1, -20, 0, 30)
    targetBtn.Position = UDim2.new(0, 10, 0, 217)
    targetBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
    targetBtn.BorderSizePixel = 0
    targetBtn.Font = FONT_MAIN
    targetBtn.Text = "Target: " .. (targetState.TargetName ~= "" and targetState.TargetName or "Select player...")
    targetBtn.TextColor3 = Color3.fromRGB(205, 205, 210)
    targetBtn.TextSize = 12
    targetBtn.TextXAlignment = Enum.TextXAlignment.Left
    targetBtn.Parent = matchSection
    targetBtn.ZIndex = 30
    Instance.new("UICorner", targetBtn).CornerRadius = UDim.new(0, 4)
    addPadding(targetBtn, 10)
    addStroke(targetBtn, Color3.fromRGB(50, 50, 55), 0, 1)

    local targetList = Instance.new("ScrollingFrame")
    targetList.Name = "MatchTargetList"
    targetList.Size = UDim2.new(1, -20, 0, 120)
    targetList.Position = UDim2.new(0, 10, 0, 249)
    targetList.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
    targetList.BorderSizePixel = 0
    targetList.ScrollBarThickness = 3
    targetList.CanvasSize = UDim2.new(0, 0, 0, 0)
    targetList.Visible = false
    targetList.ZIndex = 80
    targetList.Parent = matchSection
    Instance.new("UICorner", targetList).CornerRadius = UDim.new(0, 4)
    addStroke(targetList, Color3.fromRGB(50, 50, 55), 0, 1)

    local targetLayout = Instance.new("UIListLayout")
    targetLayout.SortOrder = Enum.SortOrder.Name
    targetLayout.Padding = UDim.new(0, 2)
    targetLayout.Parent = targetList

    local function refreshMatchTargets()
        for _, child in ipairs(targetList:GetChildren()) do
            if child:IsA("TextButton") then child:Destroy() end
        end

        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local button = Instance.new("TextButton")
                button.Size = UDim2.new(1, -4, 0, 28)
                button.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
                button.BorderSizePixel = 0
                button.Font = FONT_MAIN
                button.Text = player.DisplayName .. " (@" .. player.Name .. ")"
                button.TextColor3 = Color3.fromRGB(235, 235, 240)
                button.TextSize = 11
                button.TextXAlignment = Enum.TextXAlignment.Left
                button.ZIndex = 81
                button.Parent = targetList
                addPadding(button, 8)

                button.MouseButton1Click:Connect(function()
                    targetState.TargetName = player.Name
                    targetBtn.Text = "Target: " .. player.DisplayName
                    targetList.Visible = false
                end)
            end
        end

        targetList.CanvasSize = UDim2.new(0, 0, 0, targetLayout.AbsoluteContentSize.Y + 4)
    end

    targetBtn.MouseButton1Click:Connect(function()
        refreshMatchTargets()
        targetList.Visible = not targetList.Visible
    end)

    Players.PlayerAdded:Connect(function()
        if targetList.Visible then refreshMatchTargets() end
    end)

    Players.PlayerRemoving:Connect(function(player)
        if targetState.TargetName == player.Name then
            targetState.TargetName = ""
            targetBtn.Text = "Target: Select player..."
            targetState.TargetSnipeEnabled = false
        end
        if targetList.Visible then refreshMatchTargets() end
    end)

    CheckboxSetters["TargetSnipe"] = createCheckboxToggle(matchSection, "Target Snipe", 253, function(enabled)
        targetState.TargetSnipeEnabled = enabled == true
        if enabled then
            refreshMatchTargets()
            showNotification("Target Snipe", targetState.TargetName ~= "" and ("Target: " .. targetState.TargetName) or "Select a target")
        end
    end)

    -- DuelRing検出 / ターゲットの敵側パッドへ追従
    task.spawn(function()
        local rings = {}
        local locking = false
        local touchDistance = 6

        local function refreshRings()
            table.clear(rings)
            for _, descendant in ipairs(Workspace:GetDescendants()) do
                if (descendant:IsA("Folder") or descendant:IsA("Model")) and string.sub(descendant.Name, 1, 9) == "DuelRing_" then
                    local pads = {}
                    for _, inner in ipairs(descendant:GetDescendants()) do
                        if inner.Name == "DuelPad" then
                            local pad = nil
                            if inner:IsA("BasePart") then
                                pad = inner
                            elseif inner:IsA("Model") then
                                pad = inner:FindFirstChild("Pad", true)
                                if not (pad and pad:IsA("BasePart")) then
                                    pad = inner.PrimaryPart
                                end
                            end
                            if pad and pad:IsA("BasePart") then
                                table.insert(pads, pad)
                            end
                        end
                    end
                    if #pads > 0 then
                        table.insert(rings, {Instance = descendant, Pads = pads, Mode = descendant.Name})
                    end
                end
            end
        end

        local function playerPad(player)
            local char = player and player.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if not root or not hum or hum.Health <= 0 then return nil, nil end

            for _, ring in ipairs(rings) do
                if ring.Instance.Parent then
                    for index, pad in ipairs(ring.Pads) do
                        if pad and pad.Parent and (root.Position - pad.Position).Magnitude <= touchDistance then
                            return ring, index
                        end
                    end
                end
            end
            return nil, nil
        end

        local function padCounts(ring)
            local counts = {}
            for i = 1, #ring.Pads do counts[i] = 0 end
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    local r, index = playerPad(player)
                    if r == ring and index then counts[index] = (counts[index] or 0) + 1 end
                end
            end
            return counts
        end

        local function moveToTargetEnemy()
            local target = Players:FindFirstChild(targetState.TargetName)
            local char = LocalPlayer.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            if not target or not root then return false end

            local ring, targetPadIndex = playerPad(target)
            if not ring or not targetPadIndex then return false end

            local counts = padCounts(ring)
            local bestIndex, bestCount = nil, math.huge
            for index, count in ipairs(counts) do
                if index ~= targetPadIndex and count < bestCount then
                    bestCount = count
                    bestIndex = index
                end
            end

            local pad = bestIndex and ring.Pads[bestIndex]
            if pad and pad.Parent then
                root.AssemblyLinearVelocity = Vector3.zero
                root.AssemblyAngularVelocity = Vector3.zero
                char:PivotTo(pad.CFrame * CFrame.new(0, -15, 0))
                return true
            end
            return false
        end

        while true do
            local active = targetState.TargetSnipeEnabled or targetState.AutoMatchEnabled
            if active and not locking then
                if #rings == 0 then refreshRings() end

                if targetState.TargetSnipeEnabled and targetState.TargetName ~= "" then
                    if moveToTargetEnemy() then
                        locking = true
                        task.wait(0.1)
                        locking = false
                    end
                elseif targetState.AutoMatchEnabled then
                    local found = false
                    for _, ring in ipairs(rings) do
                        local counts = padCounts(ring)
                        local minIndex, minCount, maxCount, total = nil, math.huge, -1, 0
                        for index, count in ipairs(counts) do
                            total += count
                            if count < minCount then minCount, minIndex = count, index end
                            if count > maxCount then maxCount = count end
                        end
                        if total > 0 and maxCount > minCount and ring.Pads[minIndex] then
                            local char = LocalPlayer.Character
                            local root = char and char:FindFirstChild("HumanoidRootPart")
                            if root then
                                root.AssemblyLinearVelocity = Vector3.zero
                                root.AssemblyAngularVelocity = Vector3.zero
                                char:PivotTo(ring.Pads[minIndex].CFrame * CFrame.new(0, -15, 0))
                                found = true
                                break
                            end
                        end
                    end
                    if found then task.wait(0.1) end
                end
            else
                if #rings > 0 then table.clear(rings) end
            end

            task.wait(0.15)
        end
    end)
end

-- ==========================================================
-- Auto Vote Map (Priority 1 / 2 / 3)
-- Provided Map.StartMapVoting / Map.VoteMap implementation
-- ==========================================================
do
    local mapOptions = {"None", "Barn", "Prototype", "Ridge", "Lodge", "Aerial"}
    local voteUI = _G.__PHAutoVoteMapUI or {}
    _G.__PHAutoVoteMapUI = voteUI
    voteUI.ActiveList = nil

    local function closeList(list)
        if list then
            list.Visible = false
            list.Size = UDim2.new(1, -20, 0, 0)
        end
        if voteUI.ActiveList == list then voteUI.ActiveList = nil end
    end

    local function makePriority(rank, y, defaultValue)
        local btn = Instance.new("TextButton")
        btn.Name = "AutoVoteMapPriority" .. tostring(rank)
        btn.Size = UDim2.new(1, -20, 0, 30)
        btn.Position = UDim2.new(0, 10, 0, y)
        btn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
        btn.BorderSizePixel = 0
        btn.Font = FONT_MAIN
        btn.Text = ""
        btn.TextColor3 = Color3.fromRGB(205, 205, 210)
        btn.TextSize = 12
        btn.TextXAlignment = Enum.TextXAlignment.Left
        btn.Parent = matchSection
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
        addStroke(btn, Color3.fromRGB(50, 50, 55), 0, 1)
        addPadding(btn, 10)

        local list = Instance.new("ScrollingFrame")
        list.Name = "AutoVoteMapPriority" .. tostring(rank) .. "List"
        list.Size = UDim2.new(1, -20, 0, 0)
        list.Position = UDim2.new(0, 10, 0, y + 32)
        list.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
        list.BorderSizePixel = 0
        list.ScrollBarThickness = 2
        list.CanvasSize = UDim2.new(0, 0, 0, #mapOptions * 27)
        list.Visible = false
        list.ZIndex = 1000
        list.Parent = matchSection
        Instance.new("UICorner", list).CornerRadius = UDim.new(0, 4)
        addStroke(list, Color3.fromRGB(50, 50, 55), 0, 1)

        local function setValue(value)
            _G.__PrivateHubAutoVoteMapState["Rank" .. tostring(rank)] = value
            btn.Text = "Priority " .. tostring(rank) .. ": " .. value
            closeList(list)
        end
        setValue(defaultValue)

        for i, option in ipairs(mapOptions) do
            local opt = Instance.new("TextButton")
            opt.Size = UDim2.new(1, 0, 0, 27)
            opt.Position = UDim2.new(0, 0, 0, (i - 1) * 27)
            opt.BackgroundColor3 = Color3.fromRGB(25, 25, 28)
            opt.BorderSizePixel = 0
            opt.Font = FONT_MAIN
            opt.Text = option
            opt.TextColor3 = Color3.fromRGB(215, 215, 220)
            opt.TextSize = 12
            opt.TextXAlignment = Enum.TextXAlignment.Left
            opt.ZIndex = 1001
            opt.Parent = list
            addPadding(opt, 10)
            opt.MouseButton1Click:Connect(function() setValue(option) end)
        end

        btn.MouseButton1Click:Connect(function()
            if list.Visible then
                closeList(list)
            else
                if voteUI.ActiveList and voteUI.ActiveList ~= list then closeList(voteUI.ActiveList) end
                list.Visible = true
                list.Size = UDim2.new(1, -20, 0, math.min(#mapOptions * 27, 162))
                voteUI.ActiveList = list
            end
        end)
        voteUI["Rank" .. tostring(rank) .. "Button"] = btn
        voteUI["Rank" .. tostring(rank) .. "Set"] = setValue
        return btn
    end

    CheckboxSetters["AutoVoteMap"] = createCheckboxToggle(matchSection, "Auto Vote Map", 289, function(enabled)
        _G.__PrivateHubAutoVoteMapState.Enabled = enabled == true
        if enabled then showNotification("Auto Vote Map", "Enabled") end
    end)

    makePriority(1, 323, _G.__PrivateHubAutoVoteMapState.Rank1 or "Barn")
    makePriority(2, 359, _G.__PrivateHubAutoVoteMapState.Rank2 or "None")
    makePriority(3, 395, _G.__PrivateHubAutoVoteMapState.Rank3 or "None")

    -- 実際のマップ投票Remoteを使用（ポーリングなし）
    task.spawn(function()
        local mapFolder = ReplicatedStorage:FindFirstChild("Map")
        if not mapFolder then
            mapFolder = ReplicatedStorage:WaitForChild("Map", 8)
        end
        if not mapFolder then return end
        local startVoting = mapFolder:FindFirstChild("StartMapVoting")
        local voteMap = mapFolder:FindFirstChild("VoteMap")
        if not startVoting or not voteMap or not startVoting:IsA("RemoteEvent") then return end

        if _G.__PHAutoVoteMapConnection then
            pcall(function() _G.__PHAutoVoteMapConnection:Disconnect() end)
            _G.__PHAutoVoteMapConnection = nil
        end

        _G.__PHAutoVoteMapConnection = startVoting.OnClientEvent:Connect(function(mapList)
            if not _G.__PrivateHubAutoVoteMapState.Enabled or type(mapList) ~= "table" then return end
            local priorities = {
                _G.__PrivateHubAutoVoteMapState.Rank1 or "Barn",
                _G.__PrivateHubAutoVoteMapState.Rank2 or "None",
                _G.__PrivateHubAutoVoteMapState.Rank3 or "None"
            }
            for _, targetMap in ipairs(priorities) do
                if targetMap ~= "None" then
                    for _, availableMap in pairs(mapList) do
                        if tostring(availableMap) == targetMap then
                            local ok = pcall(function() voteMap:FireServer(targetMap) end)
                            if ok then
                                _G.__PrivateHubAutoVoteMapState.LastVote = os.clock()
                                _G.__PrivateHubAutoVoteMapState.LastButton = targetMap
                                showNotification("Auto Vote Map", "Voted: " .. targetMap)
                            end
                            return
                        end
                    end
                end
            end
        end)
    end)
end


-- ==========================================================
-- Teleport Section
-- 追従TP / 固定TP / ランダムTPをMainタブに統合
-- ==========================================================
task.spawn(function()
    _G.__PrivateHubTeleportState = _G.__PrivateHubTeleportState or {
        FollowEnabled = false,
        RandomEnabled = false,
        TargetName = "",
        OffsetX = 0,
        OffsetY = 3,
        OffsetZ = 0,
        RandomX = 0,
        RandomY = 50,
        RandomZ = 0,
        RandomRange = 500,
        RandomYRange = 100
    }

    local state = _G.__PrivateHubTeleportState
    local teleportSection = Instance.new("Frame")
    teleportSection.Name = "TeleportSection"
    teleportSection.Size = UDim2.new(0.92, 0, 0, 620)
    teleportSection.Position = UDim2.new(0.04, 0, 0, 1630)
    teleportSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
    teleportSection.BorderSizePixel = 0
    teleportSection.Parent = mainScroll
    Instance.new("UICorner", teleportSection).CornerRadius = UDim.new(0, 6)
    addStroke(teleportSection, Color3.fromRGB(40, 40, 45), 0, 1)

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 30)
    title.BackgroundTransparency = 1
    title.Font = FONT_BOLD
    title.Text = "Teleport"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.TextSize = 13
    title.Parent = teleportSection

    -- Target selection
    local targetBtn = Instance.new("TextButton")
    targetBtn.Size = UDim2.new(1, -20, 0, 32)
    targetBtn.Position = UDim2.new(0, 10, 0, 36)
    targetBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
    targetBtn.BorderSizePixel = 0
    targetBtn.Font = FONT_MAIN
    targetBtn.Text = "Target: " .. (state.TargetName ~= "" and state.TargetName or "Select player...")
    targetBtn.TextColor3 = Color3.fromRGB(205, 205, 210)
    targetBtn.TextSize = 12
    targetBtn.TextXAlignment = Enum.TextXAlignment.Left
    targetBtn.Parent = teleportSection
    Instance.new("UICorner", targetBtn).CornerRadius = UDim.new(0, 4)
    addPadding(targetBtn, 10)
    addStroke(targetBtn, Color3.fromRGB(50, 50, 55), 0, 1)

    local targetList = Instance.new("ScrollingFrame")
    targetList.Size = UDim2.new(1, -20, 0, 100)
    targetList.Position = UDim2.new(0, 10, 0, 70)
    targetList.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
    targetList.BorderSizePixel = 0
    targetList.ScrollBarThickness = 3
    targetList.Visible = false
    targetList.ZIndex = 60
    targetList.Parent = teleportSection
    Instance.new("UICorner", targetList).CornerRadius = UDim.new(0, 4)
    addStroke(targetList, Color3.fromRGB(50, 50, 55), 0, 1)

    local function refreshTargetList()
        for _, child in ipairs(targetList:GetChildren()) do
            if child:IsA("TextButton") then child:Destroy() end
        end
        local count = 0
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                count += 1
                local b = Instance.new("TextButton")
                b.Size = UDim2.new(1, -4, 0, 27)
                b.Position = UDim2.new(0, 2, 0, (count - 1) * 27)
                b.BackgroundTransparency = 1
                b.BorderSizePixel = 0
                b.Font = FONT_MAIN
                b.Text = player.Name
                b.TextColor3 = Color3.fromRGB(205, 205, 210)
                b.TextSize = 12
                b.TextXAlignment = Enum.TextXAlignment.Left
                b.ZIndex = 61
                b.Parent = targetList
                addPadding(b, 8)
                b.MouseButton1Click:Connect(function()
                    state.TargetName = player.Name
                    targetBtn.Text = "Target: " .. player.Name
                    targetList.Visible = false
                end)
            end
        end
        targetList.CanvasSize = UDim2.new(0, 0, 0, count * 27)
    end

    targetBtn.MouseButton1Click:Connect(function()
        refreshTargetList()
        targetList.Visible = not targetList.Visible
    end)
    Players.PlayerAdded:Connect(refreshTargetList)
    Players.PlayerRemoving:Connect(refreshTargetList)

    CheckboxSetters["TeleportFollow"] = createCheckboxToggle(teleportSection, "Target Follow TP", 178, function(enabled)
        state.FollowEnabled = enabled == true
    end)

    SliderSetters["TeleportOffsetX"] = createSliderRow(teleportSection, "Offset X", -100, 100, state.OffsetX, 214, function(v)
        state.OffsetX = v
    end)
    SliderSetters["TeleportOffsetY"] = createSliderRow(teleportSection, "Offset Y", -100, 100, state.OffsetY, 259, function(v)
        state.OffsetY = v
    end)
    SliderSetters["TeleportOffsetZ"] = createSliderRow(teleportSection, "Offset Z", -100, 100, state.OffsetZ, 304, function(v)
        state.OffsetZ = v
    end)

    CheckboxSetters["TeleportRandom"] = createCheckboxToggle(teleportSection, "Random TP", 352, function(enabled)
        state.RandomEnabled = enabled == true
    end)

    SliderSetters["TeleportRandomX"] = createSliderRow(teleportSection, "Random X Center", -5000, 5000, state.RandomX, 388, function(v)
        state.RandomX = v
    end)
    SliderSetters["TeleportRandomY"] = createSliderRow(teleportSection, "Random Y Center", -500, 5000, state.RandomY, 433, function(v)
        state.RandomY = v
    end)
    SliderSetters["TeleportRandomZ"] = createSliderRow(teleportSection, "Random Z Center", -5000, 5000, state.RandomZ, 478, function(v)
        state.RandomZ = v
    end)
    SliderSetters["TeleportRandomRange"] = createSliderRow(teleportSection, "Random Range", 0, 5000, state.RandomRange, 523, function(v)
        state.RandomRange = v
    end)
    SliderSetters["TeleportRandomYRange"] = createSliderRow(teleportSection, "Random Y Range", 0, 1000, state.RandomYRange, 568, function(v)
        state.RandomYRange = v
    end)

    _G.__PrivateHubTeleportUI = {
        Section = teleportSection,
        TargetRefresh = refreshTargetList
    }

    _G.__PrivateHubTeleportApplyConfig = function(data)
        if type(data) ~= "table" then return end
        local bools = {
            TeleportFollow = "followTeleport",
            TeleportRandom = "randomTeleport"
        }
        for key, field in pairs(bools) do
            if data[field] ~= nil and CheckboxSetters[key] then
                CheckboxSetters[key](data[field], true)
            end
        end
        local sliders = {
            {"TeleportOffsetX","teleportOffsetX"},
            {"TeleportOffsetY","teleportOffsetY"},
            {"TeleportOffsetZ","teleportOffsetZ"},
            {"TeleportRandomX","teleportRandomX"},
            {"TeleportRandomY","teleportRandomY"},
            {"TeleportRandomZ","teleportRandomZ"},
            {"TeleportRandomRange","teleportRandomRange"},
            {"TeleportRandomYRange","teleportRandomYRange"}
        }
        for _, item in ipairs(sliders) do
            if data[item[2]] ~= nil and SliderSetters[item[1]] then
                SliderSetters[item[1]](tonumber(data[item[2]]) or 0)
            end
        end
        if data.teleportTargetName then
            state.TargetName = tostring(data.teleportTargetName)
            targetBtn.Text = "Target: " .. (state.TargetName ~= "" and state.TargetName or "Select player...")
        end
    end

    -- 追従TP
    RunService.RenderStepped:Connect(function()
        if not state.FollowEnabled or state.TargetName == "" then return end
        local target = Players:FindFirstChild(state.TargetName)
        local char = LocalPlayer.Character
        local tchar = target and target.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local troot = tchar and tchar:FindFirstChild("HumanoidRootPart")
        if root and troot then
            root.CFrame = troot.CFrame * CFrame.new(state.OffsetX, state.OffsetY, state.OffsetZ)
        end
    end)

    -- ランダムTP
    RunService.Heartbeat:Connect(function()
        if not state.RandomEnabled then return end
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        local range = state.RandomRange
        local yRange = state.RandomYRange
        local x = state.RandomX + (math.random() * 2 - 1) * range
        local y = state.RandomY + (math.random() * 2 - 1) * yRange
        local z = state.RandomZ + (math.random() * 2 - 1) * range
        root.CFrame = CFrame.new(x, y, z)
    end)
end)

-- Auto Matchmaking: 試合終了時だけ再キュー（常時ポーリングなし）
do
    local modeMap = { ["1v1"] = "Solo", ["2v2"] = "Duo", ["3v3"] = "Trio", ["4v4"] = "Squad" }

    if _G.__PHMatchFinishedConnection then
        pcall(function() _G.__PHMatchFinishedConnection:Disconnect() end)
        _G.__PHMatchFinishedConnection = nil
    end

    local function getMatchRemotesFast()
        local gm = ReplicatedStorage:FindFirstChild("GlobalMatchmaking")
        local gmRemotes = gm and gm:FindFirstChild("Remotes")
        local queueRemote = gmRemotes and gmRemotes:FindFirstChild("JoinQueue")
        local remotes = ReplicatedStorage:FindFirstChild("Remotes")
        local finishedRemote = remotes and remotes:FindFirstChild("OnMatchFinished")
        return queueRemote, finishedRemote
    end

    local function invokeJoinQueue()
        if not _G.__PrivateHubMatchState.Enabled then return end
        local queueRemote = getMatchRemotesFast()
        if not queueRemote then return end
        local queueArg = modeMap[tostring(_G.__PrivateHubMatchState.Mode or "1v1")] or "Solo"
        pcall(function()
            if queueRemote:IsA("RemoteFunction") then
                queueRemote:InvokeServer(queueArg)
            elseif queueRemote:IsA("RemoteEvent") then
                queueRemote:FireServer(queueArg)
            end
        end)
    end

    local function installMatchConnection()
        local _, finishedRemote = getMatchRemotesFast()
        if not finishedRemote or not finishedRemote:IsA("RemoteEvent") then return false end
        _G.__PHMatchFinishedConnection = finishedRemote.OnClientEvent:Connect(function()
            if not _G.__PrivateHubMatchState.Enabled then return end
            task.delay(1.5, function()
                if _G.__PrivateHubMatchState.Enabled then invokeJoinQueue() end
            end)
        end)
        return true
    end

    -- Remoteのロード待ちは最大8秒、1秒ごとに1回だけ確認。
    task.spawn(function()
        for _ = 1, 8 do
            if installMatchConnection() then break end
            task.wait(1)
        end
    end)
end

local silentAimSection = Instance.new("Frame")
silentAimSection.Size = UDim2.new(0.92, 0, 0, 400)
silentAimSection.Position = UDim2.new(0.04, 0, 0, 2845)
silentAimSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
silentAimSection.BorderSizePixel = 0
silentAimSection.Parent = mainScroll
Instance.new("UICorner", silentAimSection).CornerRadius = UDim.new(0, 6)
addStroke(silentAimSection, Color3.fromRGB(40, 40, 45), 0, 1)

local silentTitle = Instance.new("TextLabel")
silentTitle.Name = "DynamicText"
silentTitle.Size = UDim2.new(1, 0, 0, 30)
silentTitle.BackgroundTransparency = 1
silentTitle.Font = FONT_BOLD
silentTitle.Text = "Silent Aim & Aimbot"
silentTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
silentTitle.TextSize = 13
silentTitle.Parent = silentAimSection

CheckboxSetters["SilentAim"] = createCheckboxToggle(silentAimSection, "Silent Aim", 32, function(enabled)
	SilentAimEnabled = enabled
end)

CheckboxSetters["Aimbot"] = createCheckboxToggle(silentAimSection, "Aimbot (Remote Hook)", 68, function(enabled)
	AimbotEnabled = enabled
end)

CheckboxSetters["WallCheck"] = createCheckboxToggle(silentAimSection, "Wall Check", 104, function(enabled)
	WallCheckEnabled = enabled
end)

SliderSetters["FOVRadius"] = createSliderRow(silentAimSection, "FOV Radius", 10, 800, 300, 140, function(val)
	FOV_RADIUS = val
end)

local fovColorBtn = createColorPreviewRowInParent(silentAimSection, "FOV Color", FOV_Color, 192)
ColorBtnSetters["FOVColor"] = fovColorBtn

CheckboxSetters["FOVRainbow"] = createCheckboxToggle(silentAimSection, "Rainbow FOV", 232, function(enabled)
	FOV_Rainbow = enabled
end)

CheckboxSetters["FOVFilled"] = createCheckboxToggle(silentAimSection, "Fill FOV", 268, function(enabled)
	FOV_Filled = enabled
	if fov_circle then
		fov_circle.Filled = enabled
		fov_circle.Transparency = enabled and 0.20 or 1
	end
end)

CheckboxSetters["AutoShot"] = createCheckboxToggle(silentAimSection, "Auto Shot", 304, function(enabled)
    _G.__PrivateHubAutoShotState.Enabled = enabled
    if not enabled then
        _G.__PrivateHubAutoShotState.LastShot = 0
    end
end)

SliderSetters["AutoShotCooldown"] = createSliderRow(silentAimSection, "Auto Shot Cooldown", 0.05, 5, _G.__PrivateHubAutoShotState.Cooldown, 340, function(val)
    _G.__PrivateHubAutoShotState.Cooldown = val
end)

--------------------------------------------------
-- Visuals タブ内容構築 (完全統合型 ESP System)
--------------------------------------------------
local visualsPage = tabPages["Visuals"]
local visualsScroll = Instance.new("ScrollingFrame")
visualsScroll.Size = UDim2.new(1, 0, 1, 0)
visualsScroll.BackgroundTransparency = 1
visualsScroll.BorderSizePixel = 0
visualsScroll.CanvasSize = UDim2.new(0, 0, 0, 565)
visualsScroll.ScrollBarThickness = 2
visualsScroll.Parent = visualsPage

local espSection = Instance.new("Frame")
espSection.Size = UDim2.new(0.92, 0, 0, 315)
espSection.Position = UDim2.new(0.04, 0, 0, 15)
espSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
espSection.BorderSizePixel = 0
espSection.Parent = visualsScroll
Instance.new("UICorner", espSection).CornerRadius = UDim.new(0, 6)
addStroke(espSection, Color3.fromRGB(40, 40, 45), 0, 1)

local espTitle = Instance.new("TextLabel")
espTitle.Name = "DynamicText"
espTitle.Size = UDim2.new(1, 0, 0, 35)
espTitle.BackgroundTransparency = 1
espTitle.Font = FONT_BOLD
espTitle.Text = "Team ESP System"
espTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
espTitle.TextSize = 13
espTitle.Parent = espSection

-- ESP Color 選択ボタン
local espColorBtn = createColorPreviewRowInParent(espSection, "ESP Color", MVSD_ESP_Color, 40)
ColorBtnSetters["ESPColor"] = espColorBtn

-- ESP オンオフ トグル
CheckboxSetters["MVSD_ESP"] = createCheckboxToggle(espSection, "Enable Team ESP", 80, function(enabled)
	MVSD_ESP_Enabled = enabled
end)

-- Server Desync Visualizer
local serverDesyncColorBtn = createColorPreviewRowInParent(espSection, "Server Desync Color", ServerDesync_Color, 115)
ColorBtnSetters["ServerDesyncColor"] = serverDesyncColorBtn

-- Keep the live Server Desync visualizer synchronized with the shared palette.
if type(serverDesyncColorBtn) == "function" then
	local _oldServerDesyncColorBtn = serverDesyncColorBtn
	ColorBtnSetters["ServerDesyncColor"] = function(c)
		ServerDesync_Color = c
		if ServerDesync_BoxAdornment then ServerDesync_BoxAdornment.Color3 = c end
		return _oldServerDesyncColorBtn(c)
	end
end
CheckboxSetters["ServerDesync"] = createCheckboxToggle(espSection, "Server Desync Visualizer", 155, function(enabled)
	ServerDesync_Enabled = enabled
	if enabled then ServerDesync_Setup() else ServerDesync_ClearFake() end
end)

-- Hitbox
CheckboxSetters["Hitbox"] = createCheckboxToggle(espSection, "Hitbox", 190, function(enabled)
	Hitbox_Enabled = enabled
end)
SliderSetters["HitboxSize"] = createSliderRow(espSection, "Hitbox Size", 1, 50, Hitbox_Size, 225, function(val)
	Hitbox_Size = val
end)

-- Hitbox Color（Private Hub 共通パレット）
local hitboxColorBtn = createColorPreviewRowInParent(espSection, "Hitbox Color", Hitbox_Color, 270)
ColorBtnSetters["HitboxColor"] = hitboxColorBtn

-- Server Desync Visualizer runtime
function ServerDesync_ClearFake()
	if ServerDesync_Connection then ServerDesync_Connection:Disconnect(); ServerDesync_Connection = nil end
	if ServerDesync_FakeCharacter then ServerDesync_FakeCharacter:Destroy(); ServerDesync_FakeCharacter = nil end
	ServerDesync_FakeRootPart = nil
	ServerDesync_BoxAdornment = nil
	ServerDesync_PositionHistory = {}
end

function ServerDesync_Setup()
	ServerDesync_ClearFake()
	if not ServerDesync_Enabled then return end
	local char = LocalPlayer.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	if not root then
		task.spawn(function()
			if not ServerDesync_Enabled then return end
			local c = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
			local r = c:WaitForChild("HumanoidRootPart", 5)
			if ServerDesync_Enabled and r then ServerDesync_Setup() end
		end)
		return
	end
	ServerDesync_FakeCharacter = Instance.new("Model")
	ServerDesync_FakeCharacter.Name = "FakeCharacter_ServerView"
	ServerDesync_FakeCharacter.Parent = Workspace
	ServerDesync_FakeRootPart = Instance.new("Part")
	ServerDesync_FakeRootPart.Name = "HumanoidRootPart"
	ServerDesync_FakeRootPart.Size = root.Size
	ServerDesync_FakeRootPart.CFrame = root.CFrame
	ServerDesync_FakeRootPart.Anchored = true
	ServerDesync_FakeRootPart.CanCollide = false
	ServerDesync_FakeRootPart.Transparency = 1
	ServerDesync_FakeRootPart.Material = Enum.Material.Plastic
	ServerDesync_FakeRootPart.Parent = ServerDesync_FakeCharacter
	ServerDesync_BoxAdornment = Instance.new("BoxHandleAdornment")
	ServerDesync_BoxAdornment.Name = "ServerViewBox"
	ServerDesync_BoxAdornment.Adornee = ServerDesync_FakeRootPart
	ServerDesync_BoxAdornment.Size = Vector3.new(3.5, 6, 2.5)
	ServerDesync_BoxAdornment.Color3 = ServerDesync_Color
	ServerDesync_BoxAdornment.Transparency = 0.7
	ServerDesync_BoxAdornment.AlwaysOnTop = true
	ServerDesync_BoxAdornment.Parent = ServerDesync_FakeRootPart
	ServerDesync_Connection = RunService.Heartbeat:Connect(function()
		if not ServerDesync_Enabled then return end
		local c = LocalPlayer.Character
		local r = c and c:FindFirstChild("HumanoidRootPart")
		if not r or not ServerDesync_FakeRootPart then return end
		table.insert(ServerDesync_PositionHistory, {p=r.Position,t=tick()})
		local maxHistory = math.ceil(2000/(ServerDesync_UpdateInterval*1000))+10
		while #ServerDesync_PositionHistory > maxHistory do table.remove(ServerDesync_PositionHistory,1) end
		local ping=0
		pcall(function() ping=Stats.Network.ServerStatsItem["Data Ping"]:GetValue() end)
		ping=math.max(10,math.min(1000,tonumber(ping) or 0))
		local target=tick()-ping/1000
		local best=ServerDesync_PositionHistory[#ServerDesync_PositionHistory]
		for i=1,#ServerDesync_PositionHistory do
			local e=ServerDesync_PositionHistory[i]
			if math.abs(e.t-target)<math.abs(best.t-target) then best=e end
		end
		local cf=r.CFrame
		ServerDesync_FakeRootPart.CFrame=CFrame.new(best.p)*(cf-cf.Position)
	end)
end

LocalPlayer.CharacterAdded:Connect(function()
	if ServerDesync_Enabled then task.wait(0.5); ServerDesync_Setup() end
end)

-- Hitbox helper functions (貼ってくれたコードと同じ)
local function GetPlayerTeam(plr)
	local success, val = pcall(function()
		return plr.Team
	end)
	if success and val then
		if typeof(val) == "Instance" then return val.Name
		elseif typeof(val) == "string" then return val end
	end

	local teamChild = plr:FindFirstChild("Team")
	if teamChild then
		if teamChild:IsA("ValueBase") then return tostring(teamChild.Value)
		elseif teamChild:IsA("StringValue") then return teamChild.Value
		else return teamChild.Name end
	end

	return ""
end

local function IsInSameMatchWorkspace(plr)
	local char = plr.Character
	if char and char.Parent == Workspace then
		return true
	end
	if Workspace:FindFirstChild(plr.Name) then
		return true
	end
	return false
end

-- Hitbox runtime (貼ってくれたコードと同じ敵判定)
RunService.RenderStepped:Connect(function()
    if not Hitbox_Enabled then return end

    local myTeamName = GetPlayerTeam(LocalPlayer):lower()
    if myTeamName ~= "team1" and myTeamName ~= "team2" then return end

    for _, v in ipairs(Players:GetPlayers()) do
        if v ~= LocalPlayer and v.Character then
            local char = v.Character
            local root = char:FindFirstChild("HumanoidRootPart")
            local hum = char:FindFirstChild("Humanoid")
            local targetTeamName = GetPlayerTeam(v):lower()

            local isEnemy = false
            if (targetTeamName == "team1" or targetTeamName == "team2") then
                if myTeamName ~= targetTeamName then
                    if IsInSameMatchWorkspace(v) then
                        isEnemy = true
                    end
                end
            end

            if isEnemy and root and hum and hum.Health > 0 then
                root.Size = Vector3.new(Hitbox_Size, Hitbox_Size, Hitbox_Size)
                root.Color = Hitbox_Color
                root.Transparency = 0.6
                root.CanCollide = false
            end
        end
    end
end)

-- Server Desync: toggle後にも確実に生成されるよう、キャラクター未準備時は再試行
local _OldServerDesyncSetup = ServerDesync_Setup
ServerDesync_Setup = function()
	_OldServerDesyncSetup()
	if ServerDesync_Enabled and not ServerDesync_FakeCharacter then
		task.delay(0.25, function()
			if ServerDesync_Enabled and not ServerDesync_FakeCharacter then
				_OldServerDesyncSetup()
			end
		end)
	end
end

-- Appearance セクション
local appearanceSection = Instance.new("Frame")
appearanceSection.Name = "AppearanceSection"
appearanceSection.Size = UDim2.new(0.92, 0, 0, 195)
appearanceSection.Position = UDim2.new(0.04, 0, 0, 345)
appearanceSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
appearanceSection.BorderSizePixel = 0
appearanceSection.Parent = visualsScroll
Instance.new("UICorner", appearanceSection).CornerRadius = UDim.new(0, 6)
addStroke(appearanceSection, Color3.fromRGB(40, 40, 45), 0, 1)

local appearanceTitle = Instance.new("TextLabel")
appearanceTitle.Name = "DynamicText"
appearanceTitle.Size = UDim2.new(1, 0, 0, 35)
appearanceTitle.BackgroundTransparency = 1
appearanceTitle.Font = FONT_BOLD
appearanceTitle.Text = "Appearance"
appearanceTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
appearanceTitle.TextSize = 13
appearanceTitle.Parent = appearanceSection

CheckboxSetters["Appearance_IsVip"] = createCheckboxToggle(appearanceSection, "IsVip", 35, function(enabled)
    Appearance_IsVip = enabled
    pcall(function()
        LocalPlayer:SetAttribute("IsVip", enabled)
    end)
end)

CheckboxSetters["Appearance_WinStreak"] = createCheckboxToggle(appearanceSection, "WinStreak", 75, function(enabled)
    Appearance_WinStreakEnabled = enabled
    if enabled then
        pcall(function()
            LocalPlayer:SetAttribute("WinStreak", Appearance_WinStreakValue)
        end)
    end
end)

SliderSetters["Appearance_WinStreakValue"] = createSliderRow(
    appearanceSection,
    "WinStreak Value",
    0,
    9999,
    Appearance_WinStreakValue,
    115,
    function(val)
        val = math.floor(tonumber(val) or 0)
        Appearance_WinStreakValue = val
        if Appearance_WinStreakEnabled then
            pcall(function()
                LocalPlayer:SetAttribute("WinStreak", val)
            end)
        end
    end
)

-- 統合ESP描画＆ロジック (Workspace Match & Team1/Team2 Filter)
local playerBoxes = {}
local playerTracers = {}
local playerLabels = {}

local function InitializeESPForPlayer(v)
	if v == LocalPlayer then return end

	local box = Drawing.new("Square")
	box.Thickness = 1.5
	box.Filled = false
	box.Transparency = 1
	box.Visible = false
	playerBoxes[v] = box

	local tracer = Drawing.new("Line")
	tracer.Thickness = 1.2
	tracer.Transparency = 0.8
	tracer.Visible = false
	playerTracers[v] = tracer

	local label = Drawing.new("Text")
	label.Size = 14
	label.Center = true
	label.Outline = true
	label.Transparency = 1
	label.Visible = false
	playerLabels[v] = label
end

for _, v in pairs(Players:GetPlayers()) do InitializeESPForPlayer(v) end
Players.PlayerAdded:Connect(InitializeESPForPlayer)

Players.PlayerRemoving:Connect(function(v)
	if playerBoxes[v] then playerBoxes[v]:Remove(); playerBoxes[v] = nil end
	if playerTracers[v] then playerTracers[v]:Remove(); playerTracers[v] = nil end
	if playerLabels[v] then playerLabels[v]:Remove(); playerLabels[v] = nil end
end)





RunService.RenderStepped:Connect(function()
	local Camera = workspace.CurrentCamera
	if not Camera then return end
	
	local myTeamName = GetPlayerTeam(LocalPlayer):lower()
	activeTargetCount = 0

	if myTeamName ~= "team1" and myTeamName ~= "team2" then
		for v, box in pairs(playerBoxes) do
			box.Visible = false
			if playerTracers[v] then playerTracers[v].Visible = false end
			if playerLabels[v] then playerLabels[v].Visible = false end
		end
		return
	end

	for v, box in pairs(playerBoxes) do
		local tracer = playerTracers[v]
		local label = playerLabels[v]
		
		local char = v.Character
		local root = char and char:FindFirstChild("HumanoidRootPart")
		local hum = char and char:FindFirstChild("Humanoid")
		local targetTeamName = GetPlayerTeam(v):lower()

		local isEnemy = false
		if (targetTeamName == "team1" or targetTeamName == "team2") then
			if myTeamName ~= targetTeamName then
				if IsInSameMatchWorkspace(v) then
					isEnemy = true
				end
			end
		end

		if MVSD_ESP_Enabled and isEnemy and char and root and hum and hum.Health > 0 then
			activeTargetCount = activeTargetCount + 1
			local rootPos, onScreen = Camera:WorldToViewportPoint(root.Position)

			if onScreen then
				local sizeX = 2000 / rootPos.Z
				local sizeY = 3000 / rootPos.Z

				box.Size = Vector2.new(sizeX, sizeY)
				box.Position = Vector2.new(rootPos.X - sizeX / 2, rootPos.Y - sizeY / 2)
				box.Color = MVSD_ESP_Color
				box.Visible = true

				if label then
					label.Text = v.Name
					label.Position = Vector2.new(rootPos.X, rootPos.Y - (sizeY / 2) - 18)
					label.Color = Color3.fromRGB(255, 255, 255)
					label.Visible = true
				end

				local myChar = LocalPlayer.Character
				local myHead = myChar and myChar:FindFirstChild("Head")
				if myHead and tracer then
					local startScreenPos = Camera:WorldToViewportPoint(myHead.Position)
					tracer.From = Vector2.new(startScreenPos.X, startScreenPos.Y)
					tracer.To = Vector2.new(rootPos.X, rootPos.Y)
					tracer.Color = MVSD_ESP_Color
					tracer.Visible = true
				elseif tracer then
					tracer.Visible = false
				end
			else
				box.Visible = false
				if tracer then tracer.Visible = false end
				if label then label.Visible = false end
			end
		else
			box.Visible = false
			if tracer then tracer.Visible = false end
			if label then label.Visible = false end
		end
	end
end)


--------------------------------------------------
-- Misc タブ内容構築 (Information / World Time / World Changer / Minecraft Texture / Crosshair)
--------------------------------------------------
local miscPage = tabPages["Misc"]
local miscScroll = Instance.new("ScrollingFrame")
miscScroll.Size = UDim2.new(1, 0, 1, 0)
miscScroll.BackgroundTransparency = 1
miscScroll.BorderSizePixel = 0
miscScroll.CanvasSize = UDim2.new(0, 0, 0, 1850)
miscScroll.ScrollBarThickness = 2
miscScroll.Parent = miscPage

-- ==================================================
-- Information Overlay
-- FPS / Ping / Time / Config / Server Players を選択表示
-- ==================================================
local informationSection = Instance.new("Frame")
informationSection.Name = "InformationSection"
informationSection.Size = UDim2.new(0.92, 0, 0, 270)
informationSection.Position = UDim2.new(0.04, 0, 0, 15)
informationSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
informationSection.BorderSizePixel = 0
informationSection.Parent = miscScroll
Instance.new("UICorner", informationSection).CornerRadius = UDim.new(0, 7)
addStroke(informationSection, Color3.fromRGB(40, 40, 45), 0, 1)

local informationTitle = Instance.new("TextLabel")
informationTitle.Name = "DynamicText"
informationTitle.Size = UDim2.new(1, -20, 0, 30)
informationTitle.Position = UDim2.new(0, 10, 0, 5)
informationTitle.BackgroundTransparency = 1
informationTitle.Font = FONT_BOLD
informationTitle.Text = "Information"
informationTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
informationTitle.TextSize = 13
informationTitle.TextXAlignment = Enum.TextXAlignment.Left
informationTitle.Parent = informationSection

local informationHint = Instance.new("TextLabel")
informationHint.Size = UDim2.new(1, -20, 0, 18)
informationHint.Position = UDim2.new(0, 10, 0, 31)
informationHint.BackgroundTransparency = 1
informationHint.Font = FONT_MAIN
informationHint.Text = "Choose what appears on the status bar"
informationHint.TextColor3 = Color3.fromRGB(115, 115, 122)
informationHint.TextSize = 10
informationHint.TextXAlignment = Enum.TextXAlignment.Left
informationHint.Parent = informationSection

CheckboxSetters["Information"] = createCheckboxToggle(informationSection, "Enable Information Bar", 51, function(enabled)
    _G.__PrivateHubInformationState.Enabled = enabled
end)

CheckboxSetters["InformationFPS"] = createCheckboxToggle(informationSection, "Show FPS", 87, function(enabled)
    _G.__PrivateHubInformationState.ShowFPS = enabled
end)

CheckboxSetters["InformationPing"] = createCheckboxToggle(informationSection, "Show Ping", 123, function(enabled)
    _G.__PrivateHubInformationState.ShowPing = enabled
end)

CheckboxSetters["InformationTime"] = createCheckboxToggle(informationSection, "Show Time", 159, function(enabled)
    _G.__PrivateHubInformationState.ShowTime = enabled
end)

CheckboxSetters["InformationConfig"] = createCheckboxToggle(informationSection, "Show Current Config", 195, function(enabled)
    _G.__PrivateHubInformationState.ShowConfig = enabled
end)

CheckboxSetters["InformationPlayers"] = createCheckboxToggle(informationSection, "Show Server Players", 231, function(enabled)
    _G.__PrivateHubInformationState.ShowPlayers = enabled
end)

-- Reflect saved/default Information state in the checkbox UI without notifications.
CheckboxSetters["Information"](_G.__PrivateHubInformationState.Enabled, true)
CheckboxSetters["InformationFPS"](_G.__PrivateHubInformationState.ShowFPS, true)
CheckboxSetters["InformationPing"](_G.__PrivateHubInformationState.ShowPing, true)
CheckboxSetters["InformationTime"](_G.__PrivateHubInformationState.ShowTime, true)
CheckboxSetters["InformationConfig"](_G.__PrivateHubInformationState.ShowConfig, true)
CheckboxSetters["InformationPlayers"](_G.__PrivateHubInformationState.ShowPlayers, true)

-- Status bar is independent from the hub window and stays visible when the menu is closed.
local informationGui = playerGui:FindFirstChild("PrivateHubInformation")
if informationGui then
    informationGui:Destroy()
end

informationGui = Instance.new("ScreenGui")
informationGui.Name = "PrivateHubInformation"
informationGui.ResetOnSpawn = false
informationGui.IgnoreGuiInset = true
informationGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
informationGui.Parent = playerGui

local informationBar = Instance.new("Frame")
informationBar.Name = "StatusBar"
informationBar.Size = UDim2.new(0, 390, 0, 32)
informationBar.Position = UDim2.new(0.5, -195, 1, -82)
informationBar.BackgroundColor3 = Color3.fromRGB(13, 13, 16)
informationBar.BackgroundTransparency = 0.08
informationBar.BorderSizePixel = 0
informationBar.Visible = false
informationBar.Parent = informationGui
Instance.new("UICorner", informationBar).CornerRadius = UDim.new(0, 8)
addStroke(informationBar, Color3.fromRGB(70, 70, 78), 0.15, 1)

local informationAccent = Instance.new("Frame")
informationAccent.Size = UDim2.new(0, 3, 1, -10)
informationAccent.Position = UDim2.new(0, 5, 0, 5)
informationAccent.BackgroundColor3 = Color3.fromRGB(210, 140, 180)
informationAccent.BorderSizePixel = 0
informationAccent.Parent = informationBar
Instance.new("UICorner", informationAccent).CornerRadius = UDim.new(0, 2)

local informationLabel = Instance.new("TextLabel")
informationLabel.Size = UDim2.new(1, -20, 1, 0)
informationLabel.Position = UDim2.new(0, 14, 0, 0)
informationLabel.BackgroundTransparency = 1
informationLabel.Font = FONT_MONO
informationLabel.TextColor3 = Color3.fromRGB(235, 235, 240)
informationLabel.TextSize = 12
informationLabel.TextXAlignment = Enum.TextXAlignment.Center
informationLabel.TextYAlignment = Enum.TextYAlignment.Center
informationLabel.Text = ""
informationLabel.Parent = informationBar

local informationLastUpdate = 0
local informationCachedFPS = 0
local informationCachedPing = 0

RunService.RenderStepped:Connect(function(dt)
    informationLastUpdate = informationLastUpdate + dt
    if informationLastUpdate < 0.25 then return end
    informationLastUpdate = 0

    local st = _G.__PrivateHubInformationState
    if not st or not st.Enabled then
        informationBar.Visible = false
        return
    end

    informationBar.Visible = true
    informationCachedFPS = math.max(0, math.floor((1 / math.max(dt, 0.001)) + 0.5))
    pcall(function()
        informationCachedPing = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue() + 0.5)
    end)

    local parts = {}
    if st.ShowFPS then table.insert(parts, string.format("FPS %3d", informationCachedFPS)) end
    if st.ShowPing then table.insert(parts, string.format("PING %3dms", informationCachedPing)) end
    if st.ShowTime then table.insert(parts, os.date("%H:%M:%S")) end

    if st.ShowConfig then
        local configName = tostring(Config_selectedConfigName or "")
        if configName == "" then configName = "None" end
        table.insert(parts, "CFG " .. configName)
    end

    if st.ShowPlayers then
        table.insert(parts, string.format("PLAYERS %d/%d", #Players:GetPlayers(), Players.MaxPlayers))
    end

    informationLabel.Text = (#parts > 0) and table.concat(parts, "  •  ") or "No information selected"
end)

-- World Time セクション
local worldTimeSection = Instance.new("Frame")
worldTimeSection.Size = UDim2.new(0.92, 0, 0, 145)
worldTimeSection.Position = UDim2.new(0.04, 0, 0, 285)
worldTimeSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
worldTimeSection.BorderSizePixel = 0
worldTimeSection.Parent = miscScroll
Instance.new("UICorner", worldTimeSection).CornerRadius = UDim.new(0, 6)
addStroke(worldTimeSection, Color3.fromRGB(40, 40, 45), 0, 1)

local worldTimeTitle = Instance.new("TextLabel")
worldTimeTitle.Name = "DynamicText"
worldTimeTitle.Size = UDim2.new(1, 0, 0, 35)
worldTimeTitle.BackgroundTransparency = 1
worldTimeTitle.Font = FONT_BOLD
worldTimeTitle.Text = "World Time"
worldTimeTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
worldTimeTitle.TextSize = 13
worldTimeTitle.Parent = worldTimeSection

CheckboxSetters["WorldTimeEnabled"] = createCheckboxToggle(worldTimeSection, "Enable Time Changer", 35, function(enabled)
	WorldTimeEnabled = enabled
	if not enabled then
		Lighting.ClockTime = originalClockTime
	end
end)

SliderSetters["WorldTimeValue"] = createSliderRow(worldTimeSection, "Clock Time (0 - 24)", 0, 24, 12, 70, function(val)
	WorldTimeValue = val
	if WorldTimeEnabled then
		Lighting.ClockTime = val
	end
end)

local resetTimeBtn = Instance.new("TextButton")
resetTimeBtn.Size = UDim2.new(1, -20, 0, 25)
resetTimeBtn.Position = UDim2.new(0, 10, 0, 115)
resetTimeBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
resetTimeBtn.BorderSizePixel = 0
resetTimeBtn.Font = FONT_MAIN
resetTimeBtn.Text = "Reset to Default Time"
resetTimeBtn.TextColor3 = Color3.fromRGB(180, 180, 185)
resetTimeBtn.TextSize = 11
resetTimeBtn.Parent = worldTimeSection
Instance.new("UICorner", resetTimeBtn).CornerRadius = UDim.new(0, 4)
addStroke(resetTimeBtn, Color3.fromRGB(50, 50, 55), 0, 1)

resetTimeBtn.MouseButton1Click:Connect(function()
	WorldTimeValue = originalClockTime
	Lighting.ClockTime = originalClockTime
	if SliderSetters["WorldTimeValue"] then
		SliderSetters["WorldTimeValue"](originalClockTime)
	end
	showNotification("Reset world time to default")
end)

-- World Changer セクション
local worldChangerSection = Instance.new("Frame")
worldChangerSection.Size = UDim2.new(0.92, 0, 0, 140)
worldChangerSection.Position = UDim2.new(0.04, 0, 0, 445)
worldChangerSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
worldChangerSection.BorderSizePixel = 0
worldChangerSection.Parent = miscScroll
Instance.new("UICorner", worldChangerSection).CornerRadius = UDim.new(0, 6)
addStroke(worldChangerSection, Color3.fromRGB(40, 40, 45), 0, 1)

local worldChangerTitle = Instance.new("TextLabel")
worldChangerTitle.Name = "DynamicText"
worldChangerTitle.Size = UDim2.new(1, 0, 0, 35)
worldChangerTitle.BackgroundTransparency = 1
worldChangerTitle.Font = FONT_BOLD
worldChangerTitle.Text = "World Changer"
worldChangerTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
worldChangerTitle.TextSize = 13
worldChangerTitle.Parent = worldChangerSection

CheckboxSetters["WorldChangerEnabled"] = createCheckboxToggle(worldChangerSection, "Enable World Changer", 35, function(enabled)
	WorldChangerEnabled = enabled
	if not enabled then
		Lighting.Brightness = originalLightingState.Brightness
		Lighting.Ambient = originalLightingState.Ambient
		Lighting.OutdoorAmbient = originalLightingState.OutdoorAmbient
		Lighting.ClockTime = originalLightingState.ClockTime
		Lighting.FogEnd = originalLightingState.FogEnd
		Lighting.FogColor = originalLightingState.FogColor
		Lighting.FogStart = originalLightingState.FogStart
		local atmos = Lighting:FindFirstChildOfClass("Atmosphere")
		if atmos then atmos:Destroy() end
	end
end)

local worldChangerColorBtn = createColorPreviewRowInParent(worldChangerSection, "Palette Theme Color", Color3.fromRGB(150, 0, 255), 75)

-- Weather Control セクション (他の項目と同じチェックボックスUI)
local weatherSection = Instance.new("Frame")
weatherSection.Size = UDim2.new(0.92, 0, 0, 175)
weatherSection.Position = UDim2.new(0.04, 0, 0, 595)
weatherSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
weatherSection.BorderSizePixel = 0
weatherSection.Parent = miscScroll
Instance.new("UICorner", weatherSection).CornerRadius = UDim.new(0, 6)
addStroke(weatherSection, Color3.fromRGB(40, 40, 45), 0, 1)

local weatherTitle = Instance.new("TextLabel")
weatherTitle.Name = "DynamicText"
weatherTitle.Size = UDim2.new(1, 0, 0, 35)
weatherTitle.BackgroundTransparency = 1
weatherTitle.Font = FONT_BOLD
weatherTitle.Text = "Weather Control"
weatherTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
weatherTitle.TextSize = 13
weatherTitle.Parent = weatherSection

local weatherRainSetter
local weatherSnowSetter
local weatherEnabledSetter

weatherRainSetter = createCheckboxToggle(weatherSection, "Rain", 38, function(state)
    if state then
        if weatherSnowSetter then weatherSnowSetter(false, true) end
        _G.__PHWeatherState.Type = "Rain"
        if _G.__PHWeatherState.Enabled and _G.__PHWeatherState.RainSound then _G.__PHWeatherState.RainSound:Play() end
    elseif _G.__PHWeatherState.Type == "Rain" then
        _G.__PHWeatherState.Type = "Snow"
        if weatherSnowSetter then weatherSnowSetter(true, true) end
    end
end)

weatherSnowSetter = createCheckboxToggle(weatherSection, "Snow", 68, function(state)
    if state then
        if weatherRainSetter then weatherRainSetter(false, true) end
        _G.__PHWeatherState.Type = "Snow"
        if _G.__PHWeatherState.RainSound then _G.__PHWeatherState.RainSound:Stop() end
    elseif _G.__PHWeatherState.Type == "Snow" then
        _G.__PHWeatherState.Type = "Rain"
        if weatherRainSetter then weatherRainSetter(true, true) end
    end
end)

weatherEnabledSetter = createCheckboxToggle(weatherSection, "Weather", 98, function(state)
    _G.__PHWeatherState.Enabled = state == true
    if _G.__PHWeatherState.RainSound then
        if _G.__PHWeatherState.Enabled and _G.__PHWeatherState.Type == "Rain" then
            _G.__PHWeatherState.RainSound:Play()
        else
            _G.__PHWeatherState.RainSound:Stop()
        end
    end
    if not _G.__PHWeatherState.Enabled then
        _G.__PHWeatherState.LastRain = 0
        _G.__PHWeatherState.LastSnow = 0
    end
end)

local function setWeatherType(t)
    _G.__PHWeatherState.Type = (t == "Snow") and "Snow" or "Rain"
    if _G.__PHWeatherState.Type == "Rain" then
        if weatherSnowSetter then weatherSnowSetter(false, true) end
        if weatherRainSetter then weatherRainSetter(true, true) end
    else
        if weatherRainSetter then weatherRainSetter(false, true) end
        if weatherSnowSetter then weatherSnowSetter(true, true) end
    end
    if _G.__PHWeatherState.RainSound then
        if _G.__PHWeatherState.Enabled and _G.__PHWeatherState.Type == "Rain" then _G.__PHWeatherState.RainSound:Play() else _G.__PHWeatherState.RainSound:Stop() end
    end
end

local function setWeatherEnabled(enabled)
    _G.__PHWeatherState.Enabled = enabled == true
    if weatherEnabledSetter then weatherEnabledSetter(_G.__PHWeatherState.Enabled, true) end
    if _G.__PHWeatherState.RainSound then
        if _G.__PHWeatherState.Enabled and _G.__PHWeatherState.Type == "Rain" then _G.__PHWeatherState.RainSound:Play() else _G.__PHWeatherState.RainSound:Stop() end
    end
end

if _G.__PHWeatherState.Type == "Snow" then
    setWeatherType("Snow")
else
    setWeatherType("Rain")
end

-- Weather sound (single shared instance; no duplicate GUI)
if not _G.__PHWeatherState.RainSound or not _G.__PHWeatherState.RainSound.Parent then
    local rs = Instance.new("Sound")
    rs.Name = "PrivateHubWeatherRainSound"
    rs.SoundId = "rbxassetid://237985445"
    rs.Volume = 0.5
    rs.Looped = true
    rs.Parent = workspace
    _G.__PHWeatherState.RainSound = rs
end

-- Minecraft Texture セクション
local mcTextureSection = Instance.new("Frame")
mcTextureSection.Size = UDim2.new(0.92, 0, 0, 95)
mcTextureSection.Position = UDim2.new(0.04, 0, 0, 780)
mcTextureSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
mcTextureSection.BorderSizePixel = 0
mcTextureSection.Parent = miscScroll
Instance.new("UICorner", mcTextureSection).CornerRadius = UDim.new(0, 6)
addStroke(mcTextureSection, Color3.fromRGB(40, 40, 45), 0, 1)

local mcTextureTitle = Instance.new("TextLabel")
mcTextureTitle.Name = "DynamicText"
mcTextureTitle.Size = UDim2.new(1, 0, 0, 35)
mcTextureTitle.BackgroundTransparency = 1
mcTextureTitle.Font = FONT_BOLD
mcTextureTitle.Text = "Minecraft Texture"
mcTextureTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
mcTextureTitle.TextSize = 13
mcTextureTitle.Parent = mcTextureSection

local function isPlayerCharacter(part) 
	for _,p in ipairs(Players:GetPlayers()) do 
		if p.Character and part:IsDescendantOf(p.Character) then return true end 
	end 
	return false 
end

local function isTreeOrPlant(part) 
	local cur = part 
	while cur and cur ~= Workspace do 
		local n = cur.Name:lower() 
		if n:find("palm") or n:find("tree") or n:find("leaf") or n:find("leaves") or n:find("plant") or n:find("bush") then return true end 
		cur = cur.Parent 
	end 
	return false 
end

local function detectType(part)
	local name, colorName = part.Name:lower(), part.BrickColor.Name:lower()
	if isTreeOrPlant(part) then return (name:find("trunk") or name:find("stem") or colorName:find("brown")) and "Wood" or nil end
	if colorName:find("green") or colorName:find("lime") or colorName:find("grime") or colorName:find("olive") or colorName:find("moss") or colorName:find("bamboo") or colorName:find("sage") or colorName:find("chartreuse") or colorName:find("mint") or colorName:find("khaki") or colorName:find("spring") or colorName:find("leaf") then return "Grass" end
	if colorName:find("red") or colorName:find("rust") or colorName:find("terracotta") or colorName:find("copper") or colorName:find("crimson") or colorName:find("maroon") or colorName:find("coral") or colorName:find("brick") or colorName:find("rose") or colorName:find("clement") or colorName:find("dusty") or colorName:find("magenta") then return "Brick" end
	if colorName:find("blue") or colorName:find("cyan") or colorName:find("teal") or colorName:find("aqua") then return "Water" end
	if colorName:find("nougat") or colorName:find("flesh") or colorName:find("peach") or colorName:find("pastel orange") or colorName:find("cork") or colorName:find("sand") or colorName:find("tan") or colorName:find("beige") then return "Sand" end
	if colorName:find("brown") or colorName:find("sienna") or colorName:find("umber") or colorName:find("dirt") then return "Wood" end
	if colorName:find("grey") or colorName:find("gray") or colorName:find("slate") or colorName:find("flint") or colorName:find("black") then return "Stone" end
	if name:find("grass") or name:find("lawn") or name:find("field") or name:find("ground") or name:find("floor") then return "Grass" end
	if name:find("brick") or name:find("wall") then return "Brick" end
	if name:find("water") or name:find("sea") or name:find("river") then return "Water" end
	if name:find("stone") or name:find("rock") or name:find("cobble") or name:find("path") then return "Stone" end
	if name:find("sand") or name:find("beach") then return "Sand" end
	if name:find("wood") or name:find("plank") or name:find("board") or name:find("bridge") or name:find("deck") or name:find("log") then return "Wood" end
	return nil
end

local function processPart(obj)
	if not obj:IsA("BasePart") or isPlayerCharacter(obj) or obj.Transparency > 0.5 then return end
	local t = detectType(obj)
	if t and TEXTURES[t] then
		for _, c in ipairs(obj:GetChildren()) do if c:IsA("Texture") or c:IsA("Decal") then c:Destroy() end end
		for _, f in ipairs(FACES) do 
			local tex = Instance.new("Texture") 
			tex.Name = "AutoTexture_"..t 
			tex.Texture = TEXTURES[t] 
			tex.Face = f 
			tex.StudsPerTileU = 4 
			tex.StudsPerTileV = 4 
			tex.Parent = obj 
		end
	end
end

CheckboxSetters["MinecraftTexture"] = createCheckboxToggle(mcTextureSection, "Enable MC Textures", 35, function(enabled)
	MinecraftTextureEnabled = enabled
	if not enabled then
		if minecraftConn then minecraftConn:Disconnect() minecraftConn = nil end
		for _, o in ipairs(Workspace:GetDescendants()) do 
			if o:IsA("Texture") and o.Name:find("AutoTexture_") then o:Destroy() end 
		end
	else
		for _, o in ipairs(Workspace:GetDescendants()) do processPart(o) end
		minecraftConn = Workspace.DescendantAdded:Connect(processPart)
	end
end)

-- Custom Crosshair & Watermark セクション
do
local crosshairSection = Instance.new("Frame")
crosshairSection.Size = UDim2.new(0.92, 0, 0, 250)
crosshairSection.Position = UDim2.new(0.04, 0, 0, 890)
crosshairSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
crosshairSection.BorderSizePixel = 0
crosshairSection.Parent = miscScroll
Instance.new("UICorner", crosshairSection).CornerRadius = UDim.new(0, 6)
addStroke(crosshairSection, Color3.fromRGB(40, 40, 45), 0, 1)

local crosshairTitle = Instance.new("TextLabel")
crosshairTitle.Name = "DynamicText"
crosshairTitle.Size = UDim2.new(1, 0, 0, 35)
crosshairTitle.BackgroundTransparency = 1
crosshairTitle.Font = FONT_BOLD
crosshairTitle.Text = "Custom Crosshair & Watermark"
crosshairTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
crosshairTitle.TextSize = 13
crosshairTitle.Parent = crosshairSection

CheckboxSetters["CustomCrosshair"] = createCheckboxToggle(crosshairSection, "Enable Custom Crosshair", 35, function(enabled)
	CustomCrosshairEnabled = enabled
end)

CheckboxSetters["CustomCrosshairRainbow"] = createCheckboxToggle(crosshairSection, "Rainbow Crosshair", 75, function(enabled)
	CustomCrosshairRainbow = enabled
end)

_G.__PHCrosshairColorBtn = createColorPreviewRowInParent(crosshairSection, "Crosshair Color", customCrosshairColor, 115)
ColorBtnSetters["CustomCrosshairColor"] = _G.__PHCrosshairColorBtn

_G.__PHCrosshairColorBtn:GetPropertyChangedSignal("BackgroundColor3"):Connect(function()
	customCrosshairColor = _G.__PHCrosshairColorBtn.BackgroundColor3
end)

local textRow = Instance.new("Frame")
textRow.Size = UDim2.new(1, -20, 0, 50)
textRow.Position = UDim2.new(0, 10, 0, 155)
textRow.BackgroundTransparency = 1
textRow.Parent = crosshairSection

local textLbl = Instance.new("TextLabel")
textLbl.Name = "DynamicText"
textLbl.Size = UDim2.new(1, 0, 0, 20)
textLbl.BackgroundTransparency = 1
textLbl.Font = FONT_MAIN
textLbl.Text = "Watermark Text"
textLbl.TextColor3 = Color3.fromRGB(200, 200, 205)
textLbl.TextSize = 12
textLbl.TextXAlignment = Enum.TextXAlignment.Left
textLbl.Parent = textRow

_G.__PHWatermarkBox = Instance.new("TextBox")
_G.__PHWatermarkBox.Size = UDim2.new(1, 0, 0, 26)
_G.__PHWatermarkBox.Position = UDim2.new(0, 0, 0, 22)
_G.__PHWatermarkBox.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
_G.__PHWatermarkBox.BorderSizePixel = 0
_G.__PHWatermarkBox.Font = FONT_MAIN
_G.__PHWatermarkBox.Text = "asahara.gg"
_G.__PHWatermarkBox.TextColor3 = Color3.fromRGB(220, 220, 225)
_G.__PHWatermarkBox.TextSize = 12
_G.__PHWatermarkBox.Parent = textRow
Instance.new("UICorner", _G.__PHWatermarkBox).CornerRadius = UDim.new(0, 4)
addStroke(_G.__PHWatermarkBox, Color3.fromRGB(50, 50, 55), 0, 1)
addPadding(_G.__PHWatermarkBox, 8)

_G.__PHWatermarkBox:GetPropertyChangedSignal("Text"):Connect(function()
	if _G.__PHWatermarkBox.Text ~= "" then
		CursorText.Text = _G.__PHWatermarkBox.Text
		watermarkTextValue = _G.__PHWatermarkBox.Text
	else
		CursorText.Text = "asahara.gg"
		watermarkTextValue = "asahara.gg"
	end
end)

RunService.RenderStepped:Connect(function()
	local guiOpenState = (typeof(isGuiOpen) ~= "nil" and isGuiOpen) or false

	if guiOpenState then
		UserInputService.MouseIconEnabled = true
		CursorGui.Enabled = false
		return
	end

	if CustomCrosshairEnabled then
		UserInputService.MouseIconEnabled = false
		CursorGui.Enabled = true
	else
		UserInputService.MouseIconEnabled = true
		CursorGui.Enabled = false
		return
	end
	
	local mousePos = UserInputService:GetMouseLocation()
	CursorCenter.Position = UDim2.new(0, mousePos.X, 0, mousePos.Y)
	
	local time = os.clock()
	RotationContainer.Rotation = (time * 250) % 360
	
	local baseOffset = 21
	local dynamicOffset = baseOffset + (math.sin(time * 5) * 4)
	
	local activeColor = customCrosshairColor
	if CustomCrosshairRainbow then
		activeColor = Color3.fromHSV((time * 0.5) % 1, 1, 1)
	end
	
	CursorText.TextColor3 = activeColor
	for _, item in pairs(Lines) do
		local rad = math.rad(item.Rotation)
		local x = math.sin(rad) * dynamicOffset
		local y = -math.cos(rad) * dynamicOffset

		item.Label.Position = UDim2.new(0, x, 0, y)
		item.Label.TextColor3 = activeColor
		item.Stroke.Color = Color3.fromRGB(0, 0, 0)

		-- 発光レイヤーも完全に同じ位置・色で追従
		if item.Glow then
			item.Glow.Position = UDim2.new(0, x, 0, y)
			item.Glow.TextColor3 = activeColor
		end
		if item.GlowStroke then
			item.GlowStroke.Color = activeColor
		end
	end
end)


end

-- ==========================================================
-- Effect Control
-- Kill Effect / Image / Sound replacement
-- ==========================================================
_G.__PHEffectState = _G.__PHEffectState or {
    Enabled = false,
    Random = false,
    ImageEnabled = false,
    SoundEnabled = false,
    Selected = "Vaporize",
    ImageId = "rbxassetid://97649520599364",
    SoundId = "rbxassetid://8549031795",
    Processed = {},
    OriginalImages = {},
    OriginalSounds = {}
}

_G.__PHEffectList = {
    "Vaporize", "Void", "Burn", "Bat", "GradeA", "Splatter", "Vip", "Reef", "PurpleBubble", "BlueBubble", "PinkBubble", "RedBubble", "BlackBubble", "WhiteBubble", "FrostBubble", "Reaper", "FrostReaper", "ReaperEnchanted", "WhiteIcePegasus", "BlueIcePegasus", "PurpleIcePegasus", "RedIcePegasus", "BlackIcePegasus", "RedRavager", "GreenRavager", "PinkRavager", "WhiteRavager", "BlackRavager", "BlueCaged", "RedCaged", "PurpleCaged", "GreenCaged", "WhiteCaged", "Android", "Frosthorn", "Solstice", "Winx", "Conjure", "Crystal", "Shroom", "Sakura", "Hive", "Rainbow", "Sunflower", "Watergun", "IceCream", "PurpleStrife", "RedStrife", "GreenStrife", "GoldStrife", "GreenWillow", "RedWillow", "BlueWillow", "PurpleWillow", "RedPeppermint", "BluePeppermint", "GreenPeppermint", "PurplePeppermint", "Frostbite", "BlueElderFlame", "BlackElderFlame", "RedElderFlame", "PurpleElderFlame", "FrostElderFlame", "Rosethorn", "RosethornFrost", "PurpleRhinestone", "RedRhinestone", "BlueRhinestone", "BlackRhinestone", "FrostRhinestone", "Techno", "Celestial", "FrostCelestial", "Leprichaun", "PurpleFlutter", "PinkFlutter", "RedFlutter", "BlueFlutter", "BlackFlutter", "FrostFlutter", "Harmonic", "RedFang", "GreenFang", "BlueFang", "GrayFang", "PurpleFang", "GreenPurpleFang", "FrostFang", "DefaultSmoke", "HalloweenWinx", "HalloweenSolstice", "FestiveNebula", "HeartExplosion"
}

-- Apply the media replacements to newly-created GUI/sounds and existing descendants.
_G.__PHCheckEffectObject = function(obj)
    local st = _G.__PHEffectState
    if not obj or not st then return end
    if obj:IsA("ImageLabel") and obj.Name == "Skull" then
        if st.OriginalImages[obj] == nil then st.OriginalImages[obj] = obj.Image end
        if st.Enabled and st.ImageEnabled and st.ImageId and st.ImageId ~= "" then
            obj.Image = st.ImageId
        elseif st.OriginalImages[obj] ~= nil then
            obj.Image = st.OriginalImages[obj]
        end
    elseif obj:IsA("Sound") and (obj.Name == "Sound" or obj.Name == "KillSound") then
        if st.OriginalSounds[obj] == nil then st.OriginalSounds[obj] = obj.SoundId end
        if st.Enabled and st.SoundEnabled and st.SoundId and st.SoundId ~= "" then
            obj.SoundId = st.SoundId
        elseif st.OriginalSounds[obj] ~= nil then
            obj.SoundId = st.OriginalSounds[obj]
        end
    end
end

_G.__PHSetupEffectUI = function()
    if _G.__PHEffectUI then return end
    _G.__PHEffectUI = true
    _G.__PHEffectUIRefs = {}
    local oldSection = miscScroll:FindFirstChild("EffectSection")
    if oldSection then oldSection:Destroy() end

    local sec = Instance.new("Frame")
    sec.Name = "EffectSection"
    sec.Size = UDim2.new(0.92, 0, 0, 465)
    sec.Position = UDim2.new(0.04, 0, 0, 1160)
    sec.BackgroundColor3 = Color3.fromRGB(16,16,18)
    sec.BorderSizePixel = 0
    sec.Parent = miscScroll
    Instance.new("UICorner", sec).CornerRadius = UDim.new(0,6)
    addStroke(sec, Color3.fromRGB(40,40,45), 0, 1)

    local title = Instance.new("TextLabel")
    title.Name = "DynamicText"
    title.Size = UDim2.new(1,0,0,35)
    title.BackgroundTransparency = 1
    title.Font = FONT_BOLD
    title.Text = "Effect"
    title.TextColor3 = Color3.fromRGB(255,255,255)
    title.TextSize = 13
    title.Parent = sec

    CheckboxSetters["EffectEnabled"] = createCheckboxToggle(sec, "Enable Kill Effect", 35, function(v)
        _G.__PHEffectState.Enabled = v == true
        for _, o in ipairs(playerGui:GetDescendants()) do _G.__PHCheckEffectObject(o) end
    end)
    CheckboxSetters["EffectRandom"] = createCheckboxToggle(sec, "Random Effect", 70, function(v)
        _G.__PHEffectState.Random = v == true
    end)
    CheckboxSetters["EffectImageEnabled"] = createCheckboxToggle(sec, "Custom Kill Image", 105, function(v)
        _G.__PHEffectState.ImageEnabled = v == true
        for _, o in ipairs(playerGui:GetDescendants()) do _G.__PHCheckEffectObject(o) end
    end)
    CheckboxSetters["EffectSoundEnabled"] = createCheckboxToggle(sec, "Custom Kill Sound", 140, function(v)
        _G.__PHEffectState.SoundEnabled = v == true
        for _, o in ipairs(playerGui:GetDescendants()) do _G.__PHCheckEffectObject(o) end
    end)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(.32,0,0,20)
    label.Position = UDim2.new(0,10,0,180)
    label.BackgroundTransparency = 1
    label.Font = FONT_MAIN
    label.Text = "Kill Effect"
    label.TextColor3 = Color3.fromRGB(200,200,205)
    label.TextSize = 12
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = sec

    local drop = Instance.new("TextButton")
    drop.Size = UDim2.new(.62,0,0,28)
    drop.Position = UDim2.new(.32,0,0,176)
    drop.BackgroundColor3 = Color3.fromRGB(22,22,25)
    drop.BorderSizePixel = 0
    drop.Font = FONT_MAIN
    drop.Text = _G.__PHEffectState.Selected
    drop.TextColor3 = Color3.fromRGB(220,220,225)
    drop.TextSize = 12
    drop.Parent = sec
    _G.__PHEffectUIRefs.drop = drop
    Instance.new("UICorner",drop).CornerRadius=UDim.new(0,4)
    addStroke(drop,Color3.fromRGB(50,50,55),0,1)

    local list = Instance.new("ScrollingFrame")
    list.Size = UDim2.new(.62,0,0,150)
    list.Position = UDim2.new(.32,0,0,208)
    list.BackgroundColor3 = Color3.fromRGB(20,20,23)
    list.BorderSizePixel=0
    list.ScrollBarThickness=2
    list.CanvasSize=UDim2.new(0,0,0,#_G.__PHEffectList*24)
    list.Visible=false
    list.ZIndex=30
    list.Parent=sec
    Instance.new("UICorner",list).CornerRadius=UDim.new(0,4)
    addStroke(list,Color3.fromRGB(50,50,55),0,1)

    for n, name in ipairs(_G.__PHEffectList) do
        local b=Instance.new("TextButton")
        b.Size=UDim2.new(1,-4,0,24)
        b.Position=UDim2.new(0,2,0,(n-1)*24)
        b.BackgroundTransparency=1
        b.Text=name
        b.Font=FONT_MAIN
        b.TextSize=11
        b.TextColor3=Color3.fromRGB(190,190,195)
        b.TextXAlignment=Enum.TextXAlignment.Left
        b.ZIndex=31
        b.Parent=list
        addPadding(b,7)
        b.MouseButton1Click:Connect(function()
            _G.__PHEffectState.Selected=name
            drop.Text=name
            list.Visible=false
        end)
    end
    drop.MouseButton1Click:Connect(function() list.Visible=not list.Visible end)

    local function makeIdBox(text, y, key, toggleKey)
        local l=Instance.new("TextLabel")
        l.Size=UDim2.new(.32,0,0,20)
        l.Position=UDim2.new(0,10,0,y+3)
        l.BackgroundTransparency=1
        l.Font=FONT_MAIN
        l.Text=text
        l.TextColor3=Color3.fromRGB(200,200,205)
        l.TextSize=12
        l.TextXAlignment=Enum.TextXAlignment.Left
        l.Parent=sec
        local box=Instance.new("TextBox")
        box.Size=UDim2.new(.62,0,0,26)
        box.Position=UDim2.new(.32,0,0,y)
        box.BackgroundColor3=Color3.fromRGB(22,22,25)
        box.BorderSizePixel=0
        box.Font=FONT_MAIN
        box.Text=_G.__PHEffectState[key] or ""
        box.TextColor3=Color3.fromRGB(220,220,225)
        box.TextSize=11
        box.ClearTextOnFocus=false
        box.Parent=sec
        _G.__PHEffectUIRefs[key] = box
        Instance.new("UICorner",box).CornerRadius=UDim.new(0,4)
        addStroke(box,Color3.fromRGB(50,50,55),0,1)
        addPadding(box,7)
        box.FocusLost:Connect(function()
            local id=box.Text:gsub("^%s+",""):gsub("%s+$","")
            id=id:gsub("^rbxassetid://","")
            if id ~= "" then
                _G.__PHEffectState[key]="rbxassetid://"..id
                box.Text=_G.__PHEffectState[key]
                for _,o in ipairs(playerGui:GetDescendants()) do _G.__PHCheckEffectObject(o) end
            end
        end)
        return box
    end

    makeIdBox("Kill Image ID",370,"ImageId","EffectImageEnabled")
    makeIdBox("Kill Sound ID",407,"SoundId","EffectSoundEnabled")
end
_G.__PHSetupEffectUI()

-- Start/reuse the replacement monitor.
if not _G.__PHEffectDescendantConnection then
    _G.__PHEffectDescendantConnection = playerGui.DescendantAdded:Connect(_G.__PHCheckEffectObject)
end
for _, obj in ipairs(playerGui:GetDescendants()) do
    _G.__PHCheckEffectObject(obj)
end

if not _G.__PHEffectKillConnection then
    local remotes = ReplicatedStorage:FindFirstChild("Remotes")
    local killRemote = remotes and remotes:FindFirstChild("OnPlayerKilled")
    if killRemote and killRemote:IsA("RemoteEvent") then
        _G.__PHEffectKillConnection = killRemote.OnClientEvent:Connect(function(victim)
            local st = _G.__PHEffectState
            if not st.Enabled or not victim or not victim.Character or st.Processed[victim] then return end
            local hum = victim.Character:FindFirstChildOfClass("Humanoid")
            if not hum then return end
            st.Processed[victim] = true
            local selected = st.Selected
            if st.Random then selected = _G.__PHEffectList[math.random(1, #_G.__PHEffectList)] end
            pcall(function() hum:SetAttribute("KillEffect", selected) end)
            task.delay(5, function() st.Processed[victim] = nil end)
        end)
    end
end


-- ==========================================================
-- GUNcolor
-- ShootGun の発射エフェクトをカスタム色・カスタム音に変更
-- ==========================================================
_G.__PHGunColorState = _G.__PHGunColorState or {
    Enabled = false,
    Color = Color3.fromRGB(170, 0, 255),
    SoundId = "rbxassetid://3442683707",
    LastShot = 0
}

_G.__PHGunColorPlaySound = function()
    local st = _G.__PHGunColorState
    if not st or not st.Enabled then return end
    local sound = Instance.new("Sound")
    sound.SoundId = (st.SoundId or ""):gsub("^rbxassetid://", "")
    sound.SoundId = "rbxassetid://" .. sound.SoundId
    sound.Volume = 1
    sound.Parent = game:GetService("SoundService")
    sound:Play()
    task.delay(2, function()
        pcall(function() sound:Destroy() end)
    end)
end

_G.__PHGunColorTracer = function(startPos, endPos)
    local st = _G.__PHGunColorState
    if not st or not st.Enabled then return end
    local dist = (startPos - endPos).Magnitude
    if dist < 0.5 then return end

    local part = Instance.new("Part")
    part.Material = Enum.Material.Neon
    part.Color = st.Color
    part.CanCollide = false
    part.Anchored = true
    part.Size = Vector3.new(0.2, 0.2, dist)
    part.CFrame = CFrame.new(startPos:Lerp(endPos, 0.5), endPos)
    part.Parent = workspace
    task.delay(1, function()
        pcall(function() part:Destroy() end)
    end)
end

-- 既存の Aimbot 用 __namecall hook と競合しないよう、GUNcolor は一度だけ登録。
if not _G.__PHGunColorHookInstalled then
    _G.__PHGunColorHookInstalled = true
    _G.__PHGunColorOldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
        local method = getnamecallmethod()
        local st = _G.__PHGunColorState
        local remote = _G.__PHGunColorShootRemote

        if st and st.Enabled and remote and self == remote and (method == "FireServer" or method == "fireServer") then
            local args = {...}
            local hitPos = nil
            for _, value in ipairs(args) do
                if typeof(value) == "Vector3" and value.Magnitude > 0 then hitPos = value break end
            end
            local now = tick()
            if typeof(hitPos) == "Vector3" and hitPos.Magnitude > 0 and (now - (st.LastShot or 0)) > 0.05 then
                st.LastShot = now
                task.spawn(function()
                    local character = LocalPlayer and LocalPlayer.Character
                    local head = character and character:FindFirstChild("Head")
                    if head then
                        _G.__PHGunColorTracer(head.Position, hitPos)
                        _G.__PHGunColorPlaySound()
                    end
                end)
            end
        end

        return _G.__PHGunColorOldNamecall(self, ...)
    end)
end

_G.__PHGunColorShootRemote = _G.__PHGunColorShootRemote or (ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("ShootGun"))
if not _G.__PHGunColorShootRemote then
    pcall(function() _G.__PHGunColorShootRemote = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("ShootGun", 5) end)
end

-- Fire オブジェクトを有効時だけ除去。
if not _G.__PHGunColorFireConnection then
    _G.__PHGunColorFireConnection = RunService.RenderStepped:Connect(function()
        local st = _G.__PHGunColorState
        if not st or not st.Enabled then return end
        local player = LocalPlayer
        local locations = {player and player:FindFirstChild("Backpack"), player and player.Character}
        for _, location in ipairs(locations) do
            if location then
                for _, tool in ipairs(location:GetChildren()) do
                    if tool:IsA("Tool") then
                        local fire = tool:FindFirstChild("Fire")
                        if fire then
                            fire:Destroy()
                        end
                    end
                end
            end
        end
    end)
end

-- GUNcolor UI は _G に保持して、巨大なトップレベル local を増やさない。
_G.__PHGunColorUI = _G.__PHGunColorUI or {}
do
    local ui = _G.__PHGunColorUI
    ui.section = Instance.new("Frame")
    ui.section.Name = "GUNcolorSection"
    ui.section.Size = UDim2.new(0.92, 0, 0, 175)
    ui.section.Position = UDim2.new(0.04, 0, 0, 1620)
    ui.section.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
    ui.section.BorderSizePixel = 0
    ui.section.Parent = miscScroll
    Instance.new("UICorner", ui.section).CornerRadius = UDim.new(0, 6)
    addStroke(ui.section, Color3.fromRGB(40, 40, 45), 0, 1)

    ui.title = Instance.new("TextLabel")
    ui.title.Name = "DynamicText"
    ui.title.Size = UDim2.new(1, -20, 0, 30)
    ui.title.Position = UDim2.new(0, 10, 0, 5)
    ui.title.BackgroundTransparency = 1
    ui.title.Font = FONT_BOLD
    ui.title.Text = "GUNcolor"
    ui.title.TextColor3 = Color3.fromRGB(255, 255, 255)
    ui.title.TextSize = 13
    ui.title.TextXAlignment = Enum.TextXAlignment.Left
    ui.title.Parent = ui.section

    CheckboxSetters["GUNcolor"] = createCheckboxToggle(ui.section, "Enable GUNcolor", 35, function(enabled)
        _G.__PHGunColorState.Enabled = enabled == true
    end)

    ui.colorBtn = createColorPreviewRowInParent(ui.section, "Tracer Color", _G.__PHGunColorState.Color, 73)
    _G.__PHGunColorBtn = ui.colorBtn

    ui.soundLabel = Instance.new("TextLabel")
    ui.soundLabel.Size = UDim2.new(0.32, 0, 0, 20)
    ui.soundLabel.Position = UDim2.new(0, 10, 0, 113)
    ui.soundLabel.BackgroundTransparency = 1
    ui.soundLabel.Font = FONT_MAIN
    ui.soundLabel.Text = "Sound ID"
    ui.soundLabel.TextColor3 = Color3.fromRGB(200, 200, 205)
    ui.soundLabel.TextSize = 12
    ui.soundLabel.TextXAlignment = Enum.TextXAlignment.Left
    ui.soundLabel.Parent = ui.section

    ui.soundBox = Instance.new("TextBox")
    ui.soundBox.Size = UDim2.new(0.62, 0, 0, 26)
    ui.soundBox.Position = UDim2.new(0.32, 0, 0, 110)
    ui.soundBox.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
    ui.soundBox.BorderSizePixel = 0
    ui.soundBox.Font = FONT_MAIN
    ui.soundBox.Text = _G.__PHGunColorState.SoundId
    ui.soundBox.TextColor3 = Color3.fromRGB(220, 220, 225)
    ui.soundBox.TextSize = 11
    ui.soundBox.ClearTextOnFocus = false
    ui.soundBox.Parent = ui.section
    Instance.new("UICorner", ui.soundBox).CornerRadius = UDim.new(0, 4)
    addStroke(ui.soundBox, Color3.fromRGB(50, 50, 55), 0, 1)
    addPadding(ui.soundBox, 7)

    ui.soundBox.FocusLost:Connect(function()
        local id = ui.soundBox.Text
        if id ~= "" then
            if not id:find("rbxassetid://", 1, true) then id = "rbxassetid://" .. id end
            _G.__PHGunColorState.SoundId = id
            ui.soundBox.Text = id
        end
    end)
end


--------------------------------------------------
-- ターゲット・エイム・ウォールチェック処理
--------------------------------------------------
local function isVisible(targetPart)
	if not WallCheckEnabled then return true end
	local origin = workspace.CurrentCamera.CFrame.Position
	local direction = targetPart.Position - origin
	local raycastParams = RaycastParams.new()
	raycastParams.FilterDescendantsInstances = {LocalPlayer.Character, workspace.CurrentCamera}
	raycastParams.FilterType = Enum.RaycastFilterType.Exclude
	
	local raycastResult = workspace:Raycast(origin, direction, raycastParams)
	return raycastResult == nil or raycastResult.Instance:IsDescendantOf(targetPart.Parent)
end

local function getClosestToMouse()
	local target, closestDist = nil, FOV_RADIUS
	local mousePos = UserInputService:GetMouseLocation()
	
	for _, v in pairs(Players:GetPlayers()) do
		if v ~= LocalPlayer and v.Character and v.Character:FindFirstChild("HumanoidRootPart") then
			local hum = v.Character:FindFirstChild("Humanoid")
			if hum and hum.Health > 0 then
				local rootPart = v.Character.HumanoidRootPart
				local screenPos, onScreen = workspace.CurrentCamera:WorldToViewportPoint(rootPart.Position)
				
				if onScreen and isVisible(rootPart) then
					local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
					if dist < closestDist then
						target = v
						closestDist = dist
					end
				end
			end
		end
	end
	return target
end

-- Auto Shot専用: 壁越しのターゲットは常に無視する
local function isAutoShotVisible(targetPart)
    local camera = workspace.CurrentCamera
    if not camera or not targetPart or not targetPart.Parent then return false end

    local origin = camera.CFrame.Position
    local direction = targetPart.Position - origin
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character, camera}
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude

    local result = workspace:Raycast(origin, direction, raycastParams)
    return result == nil or result.Instance:IsDescendantOf(targetPart.Parent)
end

local function getAutoShotTarget()
    local target, closestDist = nil, FOV_RADIUS
    local mousePos = UserInputService:GetMouseLocation()
    local myTeamName = GetPlayerTeam(LocalPlayer):lower()

    -- ESPと完全に同じ Team1 / Team2 + Workspace 判定
    if myTeamName ~= "team1" and myTeamName ~= "team2" then
        return nil
    end

    for _, v in ipairs(Players:GetPlayers()) do
        if v ~= LocalPlayer and v.Character then
            local targetTeamName = GetPlayerTeam(v):lower()
            local isEnemy = false

            if (targetTeamName == "team1" or targetTeamName == "team2")
                and myTeamName ~= targetTeamName then
                if IsInSameMatchWorkspace(v) then
                    isEnemy = true
                end
            end

            if isEnemy then
                local hum = v.Character:FindFirstChildOfClass("Humanoid")
                local root = v.Character:FindFirstChild("HumanoidRootPart")
                if hum and hum.Health > 0 and root then
                    local screenPos, onScreen = workspace.CurrentCamera:WorldToViewportPoint(root.Position)
                    if onScreen and isAutoShotVisible(root) then
                        local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if dist <= closestDist then
                            target = v
                            closestDist = dist
                        end
                    end
                end
            end
        end
    end
    return target
end

-- FOV内の見えているターゲットをクールダウン付きで自動発射
task.spawn(function()
    while true do
        local state = _G.__PrivateHubAutoShotState
        if state and state.Enabled and SilentAimEnabled then
            local now = os.clock()
            local cooldown = math.max(0.05, tonumber(state.Cooldown) or 0.5)
            if now - (state.LastShot or 0) >= cooldown then
                local target = getAutoShotTarget()
                local root = target and target.Character and target.Character:FindFirstChild("HumanoidRootPart")
                local remotes = ReplicatedStorage:FindFirstChild("Remotes")
                local remote = remotes and remotes:FindFirstChild("ShootGun")
                if root and remote and remote:IsA("RemoteEvent") then
                    local pos = root.Position
                    local fired = pcall(function()
                        remote:FireServer(pos, pos, root, pos)
                    end)
                    if fired then
                        state.LastShot = now
                    end
                end
            end
        end
        RunService.Heartbeat:Wait()
    end
end)

_G.__PrivateHubTextColor = _G.__PrivateHubTextColor or Color3.fromRGB(255, 255, 255)

RunService.RenderStepped:Connect(function()
	if fov_circle and target_text then
		if SilentAimEnabled then
			local mouseLoc = UserInputService:GetMouseLocation()
			fov_circle.Visible = true
			fov_circle.Position = mouseLoc
			fov_circle.Radius = FOV_RADIUS
			fov_circle.Filled = FOV_Filled
			fov_circle.Transparency = FOV_Filled and 0.20 or 1
			
			if FOV_Rainbow then
				fov_circle.Color = Color3.fromHSV((os.clock() * 0.5) % 1, 1, 1)
			else
				fov_circle.Color = fovColorBtn.BackgroundColor3
			end

			local target = getClosestToMouse()
			if target and target.Name then
				target_text.Visible = true
				target_text.Text = "Target: " .. target.Name
				target_text.Position = mouseLoc + Vector2.new(0, FOV_RADIUS + 10)
				target_text.Color = _G.__PrivateHubTextColor
			else
				target_text.Visible = false
			end
		else
			fov_circle.Visible = false
			target_text.Visible = false
		end
	end
end)

_G.__PHHooked = nil
_G.__PHHooked = hookmetamethod(game, "__namecall", function(self, ...)
	local args = {...}
	local method = getnamecallmethod()

	if method == "FireServer" and self.Name == "ShootGun" and AimbotEnabled then
		local target = getClosestToMouse()
		if target and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
			local root = target.Character.HumanoidRootPart
			local pos = root.Position
			return self.FireServer(self, pos, pos, root, pos)
		end
	end
	return _G.__PHHooked(self, ...)
end)

UserInputService.InputBegan:Connect(function(input, processed)
	if not processed and SilentAimEnabled and not AimbotEnabled and input.UserInputType == Enum.UserInputType.MouseButton1 then
		local p = getClosestToMouse()
		if p and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
			local Remote = ReplicatedStorage:WaitForChild("Remotes", 2) and ReplicatedStorage.Remotes:WaitForChild("ShootGun", 2)
			if Remote then
				local pos = p.Character.HumanoidRootPart.Position
				pcall(function()
					Remote:FireServer(pos, pos, p.Character.HumanoidRootPart, pos)
				end)
			end
		end
	end
end)

--------------------------------------------------
-- Abilities 機能
-- Knife Speed / Cooldown / Dash / Unlock Movement / Invisibility
--------------------------------------------------

-- Knife Speed: Knife_Equip の Tool を監視して ThrowSpeed を適用
_G.__PHApplyKnifeSpeed = function()
    if not KnifeSettings.Enabled then return end

    local character = LocalPlayer.Character
    local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
    local tools = {}

    if character then
        for _, v in ipairs(character:GetChildren()) do
            if v:IsA("Tool") then
                table.insert(tools, v)
            end
        end
    end

    if backpack then
        for _, v in ipairs(backpack:GetChildren()) do
            if v:IsA("Tool") then
                table.insert(tools, v)
            end
        end
    end

    for _, tool in ipairs(tools) do
        local success, anim = pcall(function()
            return tool.EquipAnimation
        end)

        if (success and anim == "Knife_Equip") or tool:GetAttribute("EquipAnimation") == "Knife_Equip" then
            local tsObj = tool:FindFirstChild("ThrowSpeed")
            if tsObj and (tsObj:IsA("NumberValue") or tsObj:IsA("IntValue")) then
                if tsObj.Value ~= KnifeSettings.Value then
                    tsObj.Value = KnifeSettings.Value
                end
            end

            if tool:GetAttribute("ThrowSpeed") ~= nil then
                if tool:GetAttribute("ThrowSpeed") ~= KnifeSettings.Value then
                    tool:SetAttribute("ThrowSpeed", KnifeSettings.Value)
                end
            end
        end
    end
end

task.spawn(function()
    while true do
        task.wait(0.2)
        _G.__PHApplyKnifeSpeed()
    end
end)

-- Cooldown: 現在の Tool / Backpack 内の Tool を監視
-- Cooldown helper is stored in _G to avoid consuming another top-level local register.
_G.__PHApplyCooldown = function(tool)
    if not tool or not CustomCooldownEnabled then return end
    pcall(function()
        local cdObj = tool:FindFirstChild("cooldown")
        if not cdObj then cdObj = tool:FindFirstChild("Cooldown") end
        if cdObj and (cdObj:IsA("NumberValue") or cdObj:IsA("IntValue")) then
            cdObj.Value = CurrentCooldownValue
        end
        local attrName
        if tool:GetAttribute("cooldown") ~= nil then
            attrName = "cooldown"
        elseif tool:GetAttribute("Cooldown") ~= nil then
            attrName = "Cooldown"
        end
        if attrName then tool:SetAttribute(attrName, CurrentCooldownValue) end
    end)
end

task.spawn(function()
    while true do
        task.wait(0.5)

        if CustomCooldownEnabled then
            local character = LocalPlayer.Character
            if character then
                local currentTool = character:FindFirstChildOfClass("Tool")
                if currentTool then
                    _G.__PHApplyCooldown(currentTool)
                end
            end

            local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
            if backpack then
                for _, tool in ipairs(backpack:GetChildren()) do
                    if tool:IsA("Tool") then
                        _G.__PHApplyCooldown(tool)
                    end
                end
            end
        end
    end
end)

-- Dash: Q キーで発動
_G.__PHTriggerCurrentDash = function()
    if not DashEnabled or not isDashable then return end
    local character = currentDashCharacter or LocalPlayer.Character
    local humanoid = currentDashHumanoid
    local root = currentDashRoot
    if not humanoid and character then humanoid = character:FindFirstChildOfClass("Humanoid") end
    if not root and character then root = character:FindFirstChild("HumanoidRootPart") end
    if not character or not humanoid or not root or humanoid.Health <= 0 then return end
    isDashable = false
    local attachment = root:FindFirstChild("RootAttachment")
    if not attachment then
        attachment = Instance.new("Attachment")
        attachment.Name = "RootAttachment"
        attachment.Parent = root
    end
    local velocity = Instance.new("LinearVelocity")
    velocity.VectorVelocity = root.CFrame.LookVector * DASH_SPEED
    velocity.MaxForce = 50000
    velocity.Attachment0 = attachment
    velocity.Parent = root
    game:GetService("Debris"):AddItem(velocity, DASH_TIME)
    local torso = currentDashUpperTorso or (character:FindFirstChild("UpperTorso") or root)
    if torso then
        pcall(function() CollectionService:AddTag(torso, "SpeedTrail") end)
        task.delay(DASH_TIME, function()
            if torso and torso.Parent then pcall(function() CollectionService:RemoveTag(torso, "SpeedTrail") end) end
        end)
    end
    task.delay(DASH_COOLDOWN, function() isDashable = true end)
end

_G.__PHSetupDashCharacter = function(character)
    if not character then return end
    currentDashCharacter = character
    currentDashHumanoid = character:WaitForChild("Humanoid", 5)
    currentDashRoot = character:WaitForChild("HumanoidRootPart", 5)
    currentDashUpperTorso = character:FindFirstChild("UpperTorso") or currentDashRoot
    isDashable = true
end

LocalPlayer.CharacterAdded:Connect(_G.__PHSetupDashCharacter)
if LocalPlayer.Character then
    task.spawn(_G.__PHSetupDashCharacter, LocalPlayer.Character)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed or not DashEnabled then return end
    if input.KeyCode == Enum.KeyCode.Q then
        _G.__PHTriggerCurrentDash()
    end
end)
-- ==========================================
-- Main タブ内 Lag / Anti-Lag (修正統合版)
-- ==========================================
_G.__PHLagPlayers = game:GetService("Players")
_G.__PHLagLocalPlayer = _G.__PHLagPlayers.LocalPlayer

_G.__PHLagSection = Instance.new("Frame")
_G.__PHLagSection.Size = UDim2.new(0.92, 0, 0, 185)
_G.__PHLagSection.Position = UDim2.new(0.04, 0, 0, 940)
_G.__PHLagSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
_G.__PHLagSection.BorderSizePixel = 0
_G.__PHLagSection.Parent = mainScroll
Instance.new("UICorner", _G.__PHLagSection).CornerRadius = UDim.new(0, 6)
addStroke(_G.__PHLagSection, Color3.fromRGB(40, 40, 45), 0, 1)

_G.__PHLagTitle = Instance.new("TextLabel")
_G.__PHLagTitle.Name = "DynamicText"
_G.__PHLagTitle.Size = UDim2.new(1, 0, 0, 35)
_G.__PHLagTitle.BackgroundTransparency = 1
_G.__PHLagTitle.Font = FONT_BOLD
_G.__PHLagTitle.Text = "Lag"
_G.__PHLagTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
_G.__PHLagTitle.TextSize = 13
_G.__PHLagTitle.Parent = _G.__PHLagSection

_G.__PHAntiLagEnabled = false
_G.__PHAntiLagLastNotify = 0

CheckboxSetters["AntiLag"] = createCheckboxToggle(_G.__PHLagSection, "Anti Lag", 38, function(enabled)
    _G.__PHAntiLagEnabled = enabled
end)

-- Knife Lag の設定変数を統合
_G.__PHKnifeLagEnabled = false
_G.__PHKnifeLagIntensity = 25 -- スライダーの初期値に合わせる

-- リモート取得関数
_G.__PHGetKnifeRemotes = function()
    local remotes = ReplicatedStorage:FindFirstChild("Remotes")
    return {
        Start = remotes and remotes:FindFirstChild("ThrowStart"),
        Hit = remotes and remotes:FindFirstChild("ThrowHit")
    }
end

-- ナイフ自動装備用の関数
_G.__PHGetKnifeByProperty = function()
    local bp = _G.__PHLagLocalPlayer:FindFirstChild("Backpack")
    local char = _G.__PHLagLocalPlayer.Character

    if char then
        for _, tool in ipairs(char:GetChildren()) do
            if tool:IsA("Tool") then
                local success, value = pcall(function() return tool.EquipAnimation end)
                if (success and value == "Knife_Equip") or tool:GetAttribute("EquipAnimation") == "Knife_Equip" then
                    return tool
                end
            end
        end
    end

    if not bp then return nil end
    for _, tool in ipairs(bp:GetChildren()) do
        if tool:IsA("Tool") then
            local success, value = pcall(function() return tool.EquipAnimation end)
            if (success and value == "Knife_Equip") or tool:GetAttribute("EquipAnimation") == "Knife_Equip" then
                return tool
            end
        end
    end
    return nil
end

_G.__PHEquipKnife = function()
    local char = _G.__PHLagLocalPlayer.Character
    local humanoid = char and char:FindFirstChildOfClass("Humanoid")
    local knife = _G.__PHGetKnifeByProperty()

    if knife and humanoid and knife.Parent ~= char then
        pcall(function()
            humanoid:EquipTool(knife)
        end)
    end
end

CheckboxSetters["KnifeLag"] = createCheckboxToggle(_G.__PHLagSection, "Knife Lag (Server)", 72, function(enabled)
    _G.__PHKnifeLagEnabled = enabled
end)

SliderSetters["_G.__PHKnifeLagIntensity"] = createSliderRow(_G.__PHLagSection, "Server Load", 1, 100, _G.__PHKnifeLagIntensity, 108, function(val)
    _G.__PHKnifeLagIntensity = val
end)

-- ナイフ限定型・超高速パケット負荷生成ループ（統合版）
task.spawn(function()
    while true do
        if _G.__PHKnifeLagEnabled then
            _G.__PHEquipKnife()

            local myChar = _G.__PHLagLocalPlayer.Character
            local myPos = (myChar and myChar.PrimaryPart) and myChar.PrimaryPart.Position or Vector3.new(0, 0, 0)
            local rs = _G.__PHGetKnifeRemotes()

            if rs and rs.Start and rs.Hit then
                local targetPart = workspace:FindFirstChild("Ground", true) or workspace:FindFirstChild("Baseplate") or workspace:FindFirstChildOfClass("Part")

                if targetPart then
                    for i = 1, _G.__PHKnifeLagIntensity do
                        pcall(function()
                            rs.Start:FireServer(myPos, Vector3.new(0, -1, 0))
                            rs.Hit:FireServer(targetPart, targetPart.Position)
                        end)

                        if i % 50 == 0 then RunService.Heartbeat:Wait() end
                    end
                end
            end
        end
        RunService.Heartbeat:Wait()
    end
end)

_G.__PHLagStatus = Instance.new("TextLabel")
_G.__PHLagStatus.Name = "DynamicText"
_G.__PHLagStatus.Size = UDim2.new(1, -20, 0, 28)
_G.__PHLagStatus.Position = UDim2.new(0, 10, 0, 105)
_G.__PHLagStatus.BackgroundTransparency = 1
_G.__PHLagStatus.Font = FONT_MONO
_G.__PHLagStatus.TextColor3 = Color3.fromRGB(135, 135, 140)
_G.__PHLagStatus.TextSize = 10
_G.__PHLagStatus.TextXAlignment = Enum.TextXAlignment.Left
_G.__PHLagStatus.Parent = _G.__PHLagSection

RunService.Heartbeat:Connect(function()
    if not _G.__PHAntiLagEnabled then return end

    local now = os.clock()
    local knifeFolder = workspace:FindFirstChild("KnifeProjectile") or workspace:FindFirstChild("KnifeProjectiles")
    if knifeFolder and #knifeFolder:GetChildren() >= 10 and now - _G.__PHAntiLagLastNotify >= 3 then
        _G.__PHAntiLagLastNotify = now
        pcall(function()
            showNotification("Anti Lag", "Lag detected.")
        end)
    end

    for _, child in ipairs(workspace:GetChildren()) do
        if child.Name == "ShadowProjectile" then
            for _, desc in ipairs(child:GetDescendants()) do
                if desc:IsA("Sound") then
                    pcall(function() desc:Destroy() end)
                end
            end
        end
    end

    for _, desc in ipairs(workspace:GetDescendants()) do
        if desc:IsA("Sound") and desc.Name == "ThrowSound" then
            pcall(function() desc:Destroy() end)
        end
    end
end)
-- ==========================================
-- Main タブ内 Abilities 内容構築
-- Knife / Cooldown / Dash / Unlock Movement / Invisibility
-- ==========================================
_G.__PHAbilitiesSection = Instance.new("Frame")
_G.__PHAbilitiesSection.Size = UDim2.new(0.92, 0, 0, 470)
_G.__PHAbilitiesSection.Position = UDim2.new(0.04, 0, 0, 1140)
_G.__PHAbilitiesSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
_G.__PHAbilitiesSection.BorderSizePixel = 0
_G.__PHAbilitiesSection.Parent = mainScroll
Instance.new("UICorner", _G.__PHAbilitiesSection).CornerRadius = UDim.new(0, 6)
addStroke(_G.__PHAbilitiesSection, Color3.fromRGB(40, 40, 45), 0, 1)

_G.__PHAbilitiesTitle = Instance.new("TextLabel")
_G.__PHAbilitiesTitle.Name = "DynamicText"
_G.__PHAbilitiesTitle.Size = UDim2.new(1, 0, 0, 35)
_G.__PHAbilitiesTitle.BackgroundTransparency = 1
_G.__PHAbilitiesTitle.Font = FONT_BOLD
_G.__PHAbilitiesTitle.Text = "Abilities"
_G.__PHAbilitiesTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
_G.__PHAbilitiesTitle.TextSize = 13
_G.__PHAbilitiesTitle.Parent = _G.__PHAbilitiesSection

CheckboxSetters["KnifeSpeed"] = createCheckboxToggle(_G.__PHAbilitiesSection, "Knife ThrowSpeed", 35, function(enabled)
    KnifeSettings.Enabled = enabled
end)

SliderSetters["KnifeSpeedValue"] = createSliderRow(_G.__PHAbilitiesSection, "ThrowSpeed Value", 1, 500, KnifeSettings.Value, 70, function(val)
    KnifeSettings.Value = val
end)

CheckboxSetters["CustomCooldown"] = createCheckboxToggle(_G.__PHAbilitiesSection, "Custom Cooldown", 120, function(enabled)
    CustomCooldownEnabled = enabled
end)

SliderSetters["CooldownValue"] = createSliderRow(_G.__PHAbilitiesSection, "Cooldown Value", 0.1, 10, CurrentCooldownValue, 155, function(val)
    CurrentCooldownValue = val
end)

CheckboxSetters["Dash"] = createCheckboxToggle(_G.__PHAbilitiesSection, "Enable Dash (Q)", 205, function(enabled)
    DashEnabled = enabled
end)

SliderSetters["DashSpeed"] = createSliderRow(_G.__PHAbilitiesSection, "Dash Speed", 10, 200, DASH_SPEED, 240, function(val)
    DASH_SPEED = val
end)

SliderSetters["DashTime"] = createSliderRow(_G.__PHAbilitiesSection, "Dash Time", 0.05, 1, DASH_TIME, 290, function(val)
    DASH_TIME = val
end)

SliderSetters["DashCooldown"] = createSliderRow(_G.__PHAbilitiesSection, "Dash Cooldown", 0.1, 5, DASH_COOLDOWN, 340, function(val)
    DASH_COOLDOWN = val
end)

CheckboxSetters["UnlockMovement"] = createCheckboxToggle(_G.__PHAbilitiesSection, "Unlock Movement", 390, function(enabled)
    setUnlockMovementEnabled(enabled)
end)

CheckboxSetters["Invisibility"] = createCheckboxToggle(_G.__PHAbilitiesSection, "Invisibility", 425, function(enabled)
    setInvisibilityEnabled(enabled)
end)

--------------------------------------------------
-- Settings タブ内容構築 (Config & Palette)
--------------------------------------------------
settingsPage = tabPages["Settings"]
settingsScroll = Instance.new("ScrollingFrame")
settingsScroll.Size = UDim2.new(1, 0, 1, 0)
settingsScroll.BackgroundTransparency = 1
settingsScroll.BorderSizePixel = 0
settingsScroll.CanvasSize = UDim2.new(0, 0, 0, 720)
settingsScroll.ScrollBarThickness = 2
settingsScroll.Parent = settingsPage

configFrame = Instance.new("Frame")
configFrame.Size = UDim2.new(0.92, 0, 0, 310)
configFrame.Position = UDim2.new(0.04, 0, 0, 15)
configFrame.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
configFrame.BorderSizePixel = 0
configFrame.Parent = settingsScroll
Instance.new("UICorner", configFrame).CornerRadius = UDim.new(0, 6)
addStroke(configFrame, Color3.fromRGB(40, 40, 45), 0, 1)

configTitle = Instance.new("TextLabel")
configTitle.Name = "DynamicText"
configTitle.Size = UDim2.new(1, 0, 0, 35)
configTitle.BackgroundTransparency = 1
configTitle.Font = FONT_BOLD
configTitle.Text = "Configuration"
configTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
configTitle.TextSize = 13
configTitle.Parent = configFrame

nameLabel = Instance.new("TextLabel")
nameLabel.Name = "DynamicText"
nameLabel.Size = UDim2.new(1, -20, 0, 20)
nameLabel.Position = UDim2.new(0, 10, 0, 40)
nameLabel.BackgroundTransparency = 1
nameLabel.Font = FONT_MAIN
nameLabel.Text = "Config name"
nameLabel.TextColor3 = Color3.fromRGB(200, 200, 205)
nameLabel.TextSize = 12
nameLabel.TextXAlignment = Enum.TextXAlignment.Left
nameLabel.Parent = configFrame

nameBox = Instance.new("TextBox")
nameBox.Size = UDim2.new(1, -20, 0, 30)
nameBox.Position = UDim2.new(0, 10, 0, 62)
nameBox.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
nameBox.BorderSizePixel = 0
nameBox.Font = FONT_MAIN
nameBox.Text = ""
nameBox.TextColor3 = Color3.fromRGB(220, 220, 225)
nameBox.TextSize = 13
nameBox.TextXAlignment = Enum.TextXAlignment.Left
nameBox.Parent = configFrame
Instance.new("UICorner", nameBox).CornerRadius = UDim.new(0, 4)
addStroke(nameBox, Color3.fromRGB(50, 50, 55), 0, 1)
addPadding(nameBox, 8)

listLabel = Instance.new("TextLabel")
listLabel.Name = "DynamicText"
listLabel.Size = UDim2.new(1, -20, 0, 20)
listLabel.Position = UDim2.new(0, 10, 0, 100)
listLabel.BackgroundTransparency = 1
listLabel.Font = FONT_MAIN
listLabel.Text = "Config list"
listLabel.TextColor3 = Color3.fromRGB(200, 200, 205)
listLabel.TextSize = 12
listLabel.TextXAlignment = Enum.TextXAlignment.Left
listLabel.Parent = configFrame

dropdownBtn = Instance.new("TextButton")
dropdownBtn.Size = UDim2.new(1, -20, 0, 30)
dropdownBtn.Position = UDim2.new(0, 10, 0, 122)
dropdownBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
dropdownBtn.BorderSizePixel = 0
dropdownBtn.Font = FONT_MAIN
dropdownBtn.Text = "Select config..."
dropdownBtn.TextColor3 = Color3.fromRGB(180, 180, 185)
dropdownBtn.TextSize = 13
dropdownBtn.TextXAlignment = Enum.TextXAlignment.Left
dropdownBtn.Parent = configFrame
Instance.new("UICorner", dropdownBtn).CornerRadius = UDim.new(0, 4)
addStroke(dropdownBtn, Color3.fromRGB(50, 50, 55), 0, 1)
addPadding(dropdownBtn, 8)

dropdownListFrame = Instance.new("ScrollingFrame")
dropdownListFrame.Size = UDim2.new(1, -20, 0, 80)
dropdownListFrame.Position = UDim2.new(0, 10, 0, 155)
dropdownListFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
dropdownListFrame.BorderSizePixel = 0
dropdownListFrame.Visible = false
dropdownListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
dropdownListFrame.ScrollBarThickness = 2
dropdownListFrame.ZIndex = 5
dropdownListFrame.Parent = configFrame
Instance.new("UICorner", dropdownListFrame).CornerRadius = UDim.new(0, 4)
addStroke(dropdownListFrame, Color3.fromRGB(60, 60, 65), 0, 1)

listLayout = Instance.new("UIListLayout")
listLayout.SortOrder = Enum.SortOrder.LayoutOrder
listLayout.Parent = dropdownListFrame

btnCreate = Instance.new("TextButton")
btnCreate.Size = UDim2.new(0.5, -13, 0, 30)
btnCreate.Position = UDim2.new(0, 10, 0, 162)
btnCreate.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
btnCreate.BorderSizePixel = 0
btnCreate.Font = FONT_MAIN
btnCreate.Text = "Create"
btnCreate.TextColor3 = Color3.fromRGB(180, 180, 185)
btnCreate.TextSize = 12
btnCreate.Parent = configFrame
Instance.new("UICorner", btnCreate).CornerRadius = UDim.new(0, 4)
addStroke(btnCreate, Color3.fromRGB(50, 50, 55), 0, 1)

btnLoad = Instance.new("TextButton")
btnLoad.Size = UDim2.new(0.5, -13, 0, 30)
btnLoad.Position = UDim2.new(0.5, 3, 0, 162)
btnLoad.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
btnLoad.BorderSizePixel = 0
btnLoad.Font = FONT_MAIN
btnLoad.Text = "Load"
btnLoad.TextColor3 = Color3.fromRGB(180, 180, 185)
btnLoad.TextSize = 12
btnLoad.Parent = configFrame
Instance.new("UICorner", btnLoad).CornerRadius = UDim.new(0, 4)
addStroke(btnLoad, Color3.fromRGB(50, 50, 55), 0, 1)

btnOverwrite = Instance.new("TextButton")
btnOverwrite.Size = UDim2.new(0.5, -13, 0, 30)
btnOverwrite.Position = UDim2.new(0, 10, 0, 198)
btnOverwrite.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
btnOverwrite.BorderSizePixel = 0
btnOverwrite.Font = FONT_MAIN
btnOverwrite.Text = "Overwrite"
btnOverwrite.TextColor3 = Color3.fromRGB(180, 180, 185)
btnOverwrite.TextSize = 12
btnOverwrite.Parent = configFrame
Instance.new("UICorner", btnOverwrite).CornerRadius = UDim.new(0, 4)
addStroke(btnOverwrite, Color3.fromRGB(50, 50, 55), 0, 1)

btnDelete = Instance.new("TextButton")
btnDelete.Size = UDim2.new(0.5, -13, 0, 30)
btnDelete.Position = UDim2.new(0.5, 3, 0, 198)
btnDelete.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
btnDelete.BorderSizePixel = 0
btnDelete.Font = FONT_MAIN
btnDelete.Text = "Delete"
btnDelete.TextColor3 = Color3.fromRGB(180, 180, 185)
btnDelete.TextSize = 12
btnDelete.Parent = configFrame
Instance.new("UICorner", btnDelete).CornerRadius = UDim.new(0, 4)
addStroke(btnDelete, Color3.fromRGB(50, 50, 55), 0, 1)

btnRefresh = Instance.new("TextButton")
btnRefresh.Size = UDim2.new(1, -20, 0, 26)
btnRefresh.Position = UDim2.new(0, 10, 0, 236)
btnRefresh.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
btnRefresh.BorderSizePixel = 0
btnRefresh.Font = FONT_MAIN
btnRefresh.Text = "Refresh list"
btnRefresh.TextColor3 = Color3.fromRGB(150, 150, 155)
btnRefresh.TextSize = 11
btnRefresh.Parent = configFrame
Instance.new("UICorner", btnRefresh).CornerRadius = UDim.new(0, 4)
addStroke(btnRefresh, Color3.fromRGB(40, 40, 45), 0, 1)

settingGuiFrame = Instance.new("Frame")
settingGuiFrame.Size = UDim2.new(0.92, 0, 0, 350)
settingGuiFrame.Position = UDim2.new(0.04, 0, 0, 340)
settingGuiFrame.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
settingGuiFrame.BorderSizePixel = 0
settingGuiFrame.Parent = settingsScroll
Instance.new("UICorner", settingGuiFrame).CornerRadius = UDim.new(0, 6)
addStroke(settingGuiFrame, Color3.fromRGB(40, 40, 45), 0, 1)

settingGuiTitle = Instance.new("TextLabel")
settingGuiTitle.Name = "DynamicText"
settingGuiTitle.Size = UDim2.new(1, 0, 0, 35)
settingGuiTitle.BackgroundTransparency = 1
settingGuiTitle.Font = FONT_BOLD
settingGuiTitle.Text = "Setting GUI"
settingGuiTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
settingGuiTitle.TextSize = 13
settingGuiTitle.Parent = settingGuiFrame

createButtonRow = function(name, defaultVal, yPos)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -20, 0, 30)
	row.Position = UDim2.new(0, 10, 0, yPos)
	row.BackgroundTransparency = 1
	row.Parent = settingGuiFrame

	local lbl = Instance.new("TextLabel")
	lbl.Name = "DynamicText"
	lbl.Size = UDim2.new(0.5, 0, 1, 0)
	lbl.BackgroundTransparency = 1
	lbl.Font = FONT_MAIN
	lbl.Text = name
	lbl.TextColor3 = Color3.fromRGB(200, 200, 205)
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Parent = row

	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0.5, 0, 1, 0)
	btn.Position = UDim2.new(0.5, 0, 0, 0)
	btn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
	btn.BorderSizePixel = 0
	btn.Font = FONT_MAIN
	btn.Text = defaultVal
	btn.TextColor3 = Color3.fromRGB(220, 220, 225)
	btn.TextSize = 12
	btn.Parent = row
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
	addStroke(btn, Color3.fromRGB(50, 50, 55), 0, 1)

	return btn
end

createInputRow = function(name, defaultVal, yPos)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -20, 0, 30)
	row.Position = UDim2.new(0, 10, 0, yPos)
	row.BackgroundTransparency = 1
	row.Parent = settingGuiFrame

	local lbl = Instance.new("TextLabel")
	lbl.Name = "DynamicText"
	lbl.Size = UDim2.new(0.5, 0, 1, 0)
	lbl.BackgroundTransparency = 1
	lbl.Font = FONT_MAIN
	lbl.Text = name
	lbl.TextColor3 = Color3.fromRGB(200, 200, 205)
	lbl.TextSize = 12
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Parent = row

	local box = Instance.new("TextBox")
	box.Size = UDim2.new(0.5, 0, 1, 0)
	box.Position = UDim2.new(0.5, 0, 0, 0)
	box.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
	box.BorderSizePixel = 0
	box.Font = FONT_MAIN
	box.Text = tostring(defaultVal)
	box.TextColor3 = Color3.fromRGB(220, 220, 225)
	box.TextSize = 12
	box.Parent = row
	Instance.new("UICorner", box).CornerRadius = UDim.new(0, 4)
	addStroke(box, Color3.fromRGB(50, 50, 55), 0, 1)

	return box
end

_G.__PHMainColorBtn = createColorPreviewRowInParent(settingGuiFrame, "Main Color", Color3.fromRGB(18, 18, 20), 40)
_G.__PHAccentColorBtn = createColorPreviewRowInParent(settingGuiFrame, "Accent Color", Color3.fromRGB(210, 140, 180), 78)
_G.__PHTextColorBtn = createColorPreviewRowInParent(settingGuiFrame, "Text Color", Color3.fromRGB(255, 255, 255), 116)

_G.__PrivateHubToggleKeyBtn = createButtonRow("Toggle Key", "RightShift", 154)
_G.__PHStartupSoundBox = createInputRow("Startup Sound ID", "0", 192)
_G.__PHNotificationSoundBox = createInputRow("Notification Sound ID", "0", 230)

applyThemeColors = function()
	local mainCol = _G.__PHMainColorBtn.BackgroundColor3
	local accentCol = _G.__PHAccentColorBtn.BackgroundColor3
	local textCol = _G.__PHTextColorBtn.BackgroundColor3
	
	mainFrame.BackgroundColor3 = mainCol
	mainFrameStroke.Color = accentCol
	outerGlow.BackgroundColor3 = accentCol
	fixBar.BackgroundColor3 = accentCol
	glitchLabel.TextColor3 = accentCol
	_G.__PrivateHubTextColor = textCol

	outerBorder.BackgroundColor3 = Color3.new(
		math.clamp(accentCol.R * 0.4, 0, 1),
		math.clamp(accentCol.G * 0.4, 0, 1),
		math.clamp(accentCol.B * 0.4, 0, 1)
	)
	
	for _, desc in ipairs(screenGui:GetDescendants()) do
		if desc:IsA("TextLabel") and desc.Name == "DynamicText" then
			desc.TextColor3 = textCol
		end
	end
end

do -- Palette scope isolation: no new outer locals
__Palette = __Palette or {}
__Palette.frame = Instance.new("Frame")
__Palette.frame.Name = "PaletteFrame"
__Palette.frame.Size = UDim2.new(0, 280, 0, 200)
__Palette.frame.Position = UDim2.new(0.5, -140, 0.5, -100)
__Palette.frame.BackgroundColor3 = Color3.fromRGB(20, 20, 24)
__Palette.frame.BorderSizePixel = 0
__Palette.frame.Visible = false
__Palette.frame.ZIndex = 20
__Palette.frame.Parent = mainFrame
Instance.new("UICorner", __Palette.frame).CornerRadius = UDim.new(0, 6)
addStroke(__Palette.frame, Color3.fromRGB(60, 60, 65), 0, 1)

__Palette.active = nil
function openPalette(targetBtn)
	__Palette.active = targetBtn
	__Palette.frame.Visible = true
end

__Palette.colors = {
	Color3.fromRGB(255,255,255), Color3.fromRGB(200,200,200), Color3.fromRGB(150,150,150), Color3.fromRGB(100,100,100), Color3.fromRGB(50,50,50), Color3.fromRGB(0,0,0),
	Color3.fromRGB(255,100,100), Color3.fromRGB(255,150,50), Color3.fromRGB(255,255,0), Color3.fromRGB(100,255,100), Color3.fromRGB(50,200,255), Color3.fromRGB(100,100,255), Color3.fromRGB(200,100,255),
	Color3.fromRGB(180,40,40), Color3.fromRGB(200,100,0), Color3.fromRGB(180,180,0), Color3.fromRGB(0,150,0), Color3.fromRGB(0,120,150), Color3.fromRGB(0,0,180), Color3.fromRGB(120,0,150),
	Color3.fromRGB(18,18,20), Color3.fromRGB(14,14,16), Color3.fromRGB(210,140,180), Color3.fromRGB(100,150,220), Color3.fromRGB(100,200,130)
}

__Palette.i = 1
while __Palette.i <= #__Palette.colors do
	__Palette.col = __Palette.colors[__Palette.i]
	__Palette.x = (__Palette.i - 1) % 7
	__Palette.y = math.floor((__Palette.i - 1) / 7)
	__Palette.pBtn = Instance.new("TextButton")
	__Palette.pBtn.Size = UDim2.new(0,32,0,26)
	__Palette.pBtn.Position = UDim2.new(0,15 + (__Palette.x * 36),0,20 + (__Palette.y * 32))
	__Palette.pBtn.BackgroundColor3 = __Palette.col
	__Palette.pBtn.BorderSizePixel = 0
	__Palette.pBtn.Text = ""
	__Palette.pBtn.ZIndex = 21
	__Palette.pBtn.Parent = __Palette.frame
	Instance.new("UICorner", __Palette.pBtn).CornerRadius = UDim.new(0,4)
	addStroke(__Palette.pBtn, Color3.fromRGB(255,255,255), 0.8, 1)
	-- Bind this exact palette button so the callback cannot accidentally
	-- read the loop's final __Palette.pBtn / __Palette.col value.
	__Palette.bindButton = function(button)
		button.MouseButton1Click:Connect(function()
			if __Palette.active then
				local selectedColor = button.BackgroundColor3
				local target = __Palette.active
				target.BackgroundColor3 = selectedColor

				if target == fovColorBtn then
					FOV_Color = selectedColor
				elseif target == espColorBtn then
					MVSD_ESP_Color = selectedColor
				elseif target == serverDesyncColorBtn then
					ServerDesync_Color = selectedColor
					if ServerDesync_BoxAdornment then ServerDesync_BoxAdornment.Color3 = selectedColor end
				elseif target == hitboxColorBtn then
					Hitbox_Color = selectedColor
				elseif target == worldChangerColorBtn then
					_G.__PHWorldChangerColor = selectedColor
					if WorldChangerEnabled then applyWorldChanger() end
				elseif target == _G.__PHCrosshairColorBtn then
					customCrosshairColor = selectedColor
				elseif target == _G.__PHGunColorBtn then
					_G.__PHGunColorState.Color = selectedColor
                elseif target == _G.__PHRageTargetMarkerColorBtn then
                    _G.__PrivateHubRageTargetState.TargetMarkerColor = selectedColor
                elseif target == _G.__PHRageTextColorBtn then
                    _G.__PrivateHubRageTargetState.TextColor = selectedColor
                    if centerKillLabel then centerKillLabel.TextColor3 = selectedColor end
				end

				applyThemeColors()
			end
			__Palette.frame.Visible = false
		end)
	end
	__Palette.bindButton(__Palette.pBtn)
	__Palette.i = __Palette.i + 1
end

_G.__PHMainColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHMainColorBtn) end)
_G.__PHAccentColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHAccentColorBtn) end)
_G.__PHTextColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHTextColorBtn) end)
if _G.__PHGunColorBtn then _G.__PHGunColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHGunColorBtn) end) end
fovColorBtn.MouseButton1Click:Connect(function() openPalette(fovColorBtn) end)
worldChangerColorBtn.MouseButton1Click:Connect(function() openPalette(worldChangerColorBtn) end)
_G.__PHCrosshairColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHCrosshairColorBtn) end)
espColorBtn.MouseButton1Click:Connect(function() openPalette(espColorBtn) end)
serverDesyncColorBtn.MouseButton1Click:Connect(function() openPalette(serverDesyncColorBtn) end)
hitboxColorBtn.MouseButton1Click:Connect(function() openPalette(hitboxColorBtn) end)

espColorBtn:GetPropertyChangedSignal("BackgroundColor3"):Connect(function()
	MVSD_ESP_Color = espColorBtn.BackgroundColor3
	Hitbox_Color = MVSD_ESP_Color
end)
end -- Palette scope

-- Avoid creating another top-level local here: the script is already near Luau's
-- 200 local-register limit. Store this temporary key-listening state outside
-- the current local scope instead.
_G.__PrivateHubListeningForKey = false

_G.__PrivateHubToggleKeyBtn.MouseButton1Click:Connect(function()
	_G.__PrivateHubListeningForKey = true
	_G.__PrivateHubToggleKeyBtn.Text = "Press key..."
end)

UserInputService.InputBegan:Connect(function(input, gp)
	if _G.__PrivateHubListeningForKey then
		if input.UserInputType == Enum.UserInputType.Keyboard then
			if input.KeyCode ~= Enum.KeyCode.Unknown then
				_G.__PrivateHubListeningForKey = false
				_G.__PrivateHubToggleKeyBtn.Text = input.KeyCode.Name
				showNotification("Toggle key changed to " .. input.KeyCode.Name)
			end
		end
		return
	end
	
	if input.UserInputType == Enum.UserInputType.Keyboard then
		if input.KeyCode.Name == _G.__PrivateHubToggleKeyBtn.Text then
			setGuiOpen(not isGuiOpen)
		end
	end
end)

do
-- ConfigState はローカルレジスタを一切消費しないよう _G に保持。
-- このスコープでは ConfigState という local も作らない。
_G.__PrivateHubConfigState = _G.__PrivateHubConfigState or {}
--------------------------------------------------
-- ファイルシステムとConfigの処理 (ESPデータ追加対応)
--------------------------------------------------
Config_folderPath = "private setting"
Config_subFolderPath = "private setting/setting"
Config_lastConfigPath = "private setting/last_config.txt"

if makefolder and not isfolder(Config_folderPath) then
	pcall(function() makefolder(Config_folderPath) end)
end
if makefolder and not isfolder(Config_subFolderPath) then
	pcall(function() makefolder(Config_subFolderPath) end)
end

Config_selectedConfigName = ""

_G.__PrivateHubConfigState.refreshConfigList = function()
	for _, child in ipairs(dropdownListFrame:GetChildren()) do
		if child:IsA("TextButton") then
			child:Destroy()
		end
	end
	
	if not listfiles then return end
	local success, files = pcall(function() return listfiles(Config_subFolderPath) end)
	if not success or not files then return end
	
	Config_count = 0
	for _, Config_filePath in ipairs(files) do
		if string.sub(Config_filePath, -4) == ".txt" then
			Config_fileName = Config_filePath:match("([^/]+)$") or Config_filePath
			Config_fileName = Config_fileName:match("([^\\]+)$") or Config_fileName
			Config_configName = string.sub(Config_fileName, 1, -5)
			
			Config_count = Config_count + 1
			Config_itemBtn = Instance.new("TextButton")
			Config_itemBtn.Size = UDim2.new(1, 0, 0, 24)
			Config_itemBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
			Config_itemBtn.BackgroundTransparency = 1
			Config_itemBtn.BorderSizePixel = 0
			Config_itemBtn.Font = FONT_MAIN
			Config_itemBtn.Text = Config_configName
			Config_itemBtn.TextColor3 = Color3.fromRGB(180, 180, 185)
			Config_itemBtn.TextSize = 12
			Config_itemBtn.TextXAlignment = Enum.TextXAlignment.Left
			Config_itemBtn.ZIndex = 6
			Config_itemBtn.Parent = dropdownListFrame
			addPadding(Config_itemBtn, 10)
			
			-- 各ボタンごとにConfig名を保持する
			local buttonConfigName = Config_configName
			Config_itemBtn.MouseButton1Click:Connect(function()
				Config_selectedConfigName = buttonConfigName
				dropdownBtn.Text = buttonConfigName
				dropdownListFrame.Visible = false
			end)
		end
	end
	dropdownListFrame.CanvasSize = UDim2.new(0, 0, 0, Config_count * 24)
end

dropdownBtn.MouseButton1Click:Connect(function()
	dropdownListFrame.Visible = not dropdownListFrame.Visible
	if dropdownListFrame.Visible then _G.__PrivateHubConfigState.refreshConfigList() end
end)

btnRefresh.MouseButton1Click:Connect(function()
	_G.__PrivateHubConfigState.refreshConfigList()
	showNotification("Config list refreshed")
end)

Config_colorToHex = function(col)
	return string.format("#%02X%02X%02X", math.floor(col.R*255), math.floor(col.G*255), math.floor(col.B*255))
end

Config_hexToColor = function(hex)
	hex = hex:gsub("#","")
	return Color3.fromRGB(
		tonumber("0x"..hex:sub(1,2)) or 24,
		tonumber("0x"..hex:sub(3,4)) or 24,
		tonumber("0x"..hex:sub(5,6)) or 26
	)
end

-- Shared config-backed combat/lag states (declared before Config_gatherSettingsData)
RageKillEnabled = false
RageKillMode = "GUN"
SlowKillEnabled = false
SlowKillDelay = 0.1
AutoEquipEnabled = false

-- Rage Kill target-lock state (kept in _G to avoid local-register pressure).
_G.__PrivateHubRageTargetState = _G.__PrivateHubRageTargetState or {
    Enabled = false,
    TargetName = "",
    TargetMarkerColor = Color3.fromRGB(255, 70, 70),
    TextColor = Color3.fromRGB(255, 50, 50),
    Highlight = nil
}

Config_gatherSettingsData = function()
	Config_data = {
		mainColor = Config_colorToHex(_G.__PHMainColorBtn.BackgroundColor3),
		accentColor = Config_colorToHex(_G.__PHAccentColorBtn.BackgroundColor3),
		textColor = Config_colorToHex(_G.__PHTextColorBtn.BackgroundColor3),
		toggleKey = _G.__PrivateHubToggleKeyBtn.Text,
		startupSound = _G.__PHStartupSoundBox.Text,
		notificationSound = _G.__PHNotificationSoundBox.Text,
		walkSpeedEnabled = ModSpeedEnabled, -- ★ Config保存用
		walkSpeed = ModSpeedValue,
		infJump = ModInfJumpEnabled,
		noclip = ModNoclipEnabled,
		spinEnabled = _G.__PrivateHubSpinState.Enabled,
		spinSpeed = _G.__PrivateHubSpinState.Speed,
		matchAutoQueue = _G.__PrivateHubMatchState.Enabled,
		matchMode = _G.__PrivateHubMatchState.Mode,
		followTeleport = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.FollowEnabled == true,
		randomTeleport = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.RandomEnabled == true,
		teleportTargetName = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.TargetName or "",
		teleportOffsetX = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.OffsetX or 0,
		teleportOffsetY = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.OffsetY or 3,
		teleportOffsetZ = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.OffsetZ or 0,
		teleportRandomX = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.RandomX or 0,
		teleportRandomY = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.RandomY or 50,
		teleportRandomZ = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.RandomZ or 0,
		teleportRandomRange = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.RandomRange or 500,
		teleportRandomYRange = _G.__PrivateHubTeleportState and _G.__PrivateHubTeleportState.RandomYRange or 100,
		autoVoteMap = _G.__PrivateHubAutoVoteMapState.Enabled == true,
		autoVoteMapRank1 = _G.__PrivateHubAutoVoteMapState.Rank1 or "Barn",
		autoVoteMapRank2 = _G.__PrivateHubAutoVoteMapState.Rank2 or "None",
		autoVoteMapRank3 = _G.__PrivateHubAutoVoteMapState.Rank3 or "None",
		silentAim = SilentAimEnabled,
		aimbot = AimbotEnabled,
		wallCheck = WallCheckEnabled,
		fovRadius = FOV_RADIUS,
		fovColor = Config_colorToHex(fovColorBtn.BackgroundColor3),
		fovRainbow = FOV_Rainbow,
		fovFilled = FOV_Filled,
		autoShot = _G.__PrivateHubAutoShotState.Enabled == true,
		autoShotCooldown = _G.__PrivateHubAutoShotState.Cooldown or 0.5,
		mvsdEspEnabled = MVSD_ESP_Enabled,
		mvsdEspColor = Config_colorToHex(espColorBtn.BackgroundColor3),
		serverDesyncEnabled = ServerDesync_Enabled,
		serverDesyncColor = Config_colorToHex(serverDesyncColorBtn.BackgroundColor3),
		hitboxEnabled = Hitbox_Enabled,
		hitboxSize = Hitbox_Size,
		hitboxColor = Config_colorToHex(Hitbox_Color),

		-- Appearance
		appearanceIsVip = Appearance_IsVip,
		appearanceWinStreakEnabled = Appearance_WinStreakEnabled,
		appearanceWinStreakValue = Appearance_WinStreakValue,

		informationEnabled = _G.__PrivateHubInformationState.Enabled,
		informationShowFPS = _G.__PrivateHubInformationState.ShowFPS,
		informationShowPing = _G.__PrivateHubInformationState.ShowPing,
		informationShowTime = _G.__PrivateHubInformationState.ShowTime,
		informationShowConfig = _G.__PrivateHubInformationState.ShowConfig,
		informationShowPlayers = _G.__PrivateHubInformationState.ShowPlayers,

		worldTimeEnabled = WorldTimeEnabled,
		worldTimeValue = WorldTimeValue,
		worldChangerEnabled = WorldChangerEnabled,
		worldChangerColor = Config_colorToHex(worldChangerColorBtn.BackgroundColor3),
		weatherEnabled = _G.__PHWeatherState.Enabled == true,
		weatherType = _G.__PHWeatherState.Type or "Rain",
		minecraftTextureEnabled = MinecraftTextureEnabled,
		customCrosshairEnabled = CustomCrosshairEnabled,
		customCrosshairRainbow = CustomCrosshairRainbow,
		customCrosshairColor = Config_colorToHex(customCrosshairColor),
		watermarkText = _G.__PHWatermarkBox.Text,
		effectEnabled = _G.__PHEffectState and _G.__PHEffectState.Enabled == true,
		effectRandom = _G.__PHEffectState and _G.__PHEffectState.Random == true,
		effectImageEnabled = _G.__PHEffectState and _G.__PHEffectState.ImageEnabled == true,
		effectSoundEnabled = _G.__PHEffectState and _G.__PHEffectState.SoundEnabled == true,
		effectSelected = (_G.__PHEffectState and _G.__PHEffectState.Selected) or "Vaporize",
		effectImageId = (_G.__PHEffectState and _G.__PHEffectState.ImageId) or "rbxassetid://97649520599364",
		effectSoundId = (_G.__PHEffectState and _G.__PHEffectState.SoundId) or "rbxassetid://8549031795",
		gunColorEnabled = _G.__PHGunColorState and _G.__PHGunColorState.Enabled == true,
		gunColor = _G.__PHGunColorState and Config_colorToHex(_G.__PHGunColorState.Color) or "#AA00FF",
		gunColorSoundId = (_G.__PHGunColorState and _G.__PHGunColorState.SoundId) or "rbxassetid://3442683707",

		-- Abilities
		knifeSpeedEnabled = KnifeSettings.Enabled,
		knifeSpeedValue = KnifeSettings.Value,
		customCooldownEnabled = CustomCooldownEnabled,
		cooldownValue = CurrentCooldownValue,
		dashEnabled = DashEnabled,
		dashSpeed = DASH_SPEED,
		dashTime = DASH_TIME,
		dashCooldown = DASH_COOLDOWN,
		unlockMovementEnabled = UnlockMovementEnabled,
		invisibilityEnabled = InvisibilityEnabled,
        rageKill = RageKillEnabled,
        rageKillMode = RageKillMode,
        rageKillTargetEnabled = _G.__PrivateHubRageTargetState.Enabled == true,
        rageKillTargetName = _G.__PrivateHubRageTargetState.TargetName or "",
        rageKillTargetMarkerColor = Config_colorToHex(_G.__PrivateHubRageTargetState.TargetMarkerColor or Color3.fromRGB(255, 70, 70)),
        rageKillTextColor = Config_colorToHex(_G.__PrivateHubRageTargetState.TextColor or Color3.fromRGB(255, 50, 50)),
		slowKill = SlowKillEnabled,
		autoEquip = AutoEquipEnabled,
		antiLag = _G.__PHAntiLagEnabled == true
	}
	return HttpService:JSONEncode(Config_data)
end

btnCreate.MouseButton1Click:Connect(function()
	Config_cName = nameBox.Text
	if Config_cName == "" then return end
	Config_filePath = Config_subFolderPath .. "/" .. Config_cName .. ".txt"
	
	if writefile then
		pcall(function() writefile(Config_filePath, Config_gatherSettingsData()) end)
	end
	_G.__PrivateHubConfigState.refreshConfigList()
	Config_selectedConfigName = Config_cName
	dropdownBtn.Text = Config_cName
	showNotification("Config created: " .. Config_cName)
end)

btnOverwrite.MouseButton1Click:Connect(function()
	if Config_selectedConfigName == "" then return end
	local Config_filePath = Config_subFolderPath .. "/" .. Config_selectedConfigName .. ".txt"
	
	if writefile then
		pcall(function() writefile(Config_filePath, Config_gatherSettingsData()) end)
	end
	showNotification("Config overwritten")
end)

btnDelete.MouseButton1Click:Connect(function()
	if Config_selectedConfigName == "" then return end
	local Config_filePath = Config_subFolderPath .. "/" .. Config_selectedConfigName .. ".txt"
	
	if delfile then
		pcall(function() delfile(Config_filePath) end)
	end
	Config_selectedConfigName = ""
	dropdownBtn.Text = "Select config..."
	_G.__PrivateHubConfigState.refreshConfigList()
	showNotification("Config deleted")
end)

Config_loadConfigByName = function(Config_cName)
	local Config_filePath = Config_subFolderPath .. "/" .. Config_cName .. ".txt"
	if readfile then
		local success, content = pcall(function() return readfile(Config_filePath) end)
		if success and content then
			local ok, Config_data = pcall(function() return HttpService:JSONDecode(content) end)
			if ok and Config_data then
				if Config_data.mainColor then _G.__PHMainColorBtn.BackgroundColor3 = Config_hexToColor(Config_data.mainColor) end
				if Config_data.accentColor then _G.__PHAccentColorBtn.BackgroundColor3 = Config_hexToColor(Config_data.accentColor) end
				if Config_data.textColor then _G.__PHTextColorBtn.BackgroundColor3 = Config_hexToColor(Config_data.textColor) end
				if Config_data.informationEnabled ~= nil and CheckboxSetters["Information"] then CheckboxSetters["Information"](Config_data.informationEnabled, true) end
				if Config_data.informationShowFPS ~= nil and CheckboxSetters["InformationFPS"] then CheckboxSetters["InformationFPS"](Config_data.informationShowFPS, true) end
				if Config_data.informationShowPing ~= nil and CheckboxSetters["InformationPing"] then CheckboxSetters["InformationPing"](Config_data.informationShowPing, true) end
				if Config_data.informationShowTime ~= nil and CheckboxSetters["InformationTime"] then CheckboxSetters["InformationTime"](Config_data.informationShowTime, true) end
				if Config_data.informationShowConfig ~= nil and CheckboxSetters["InformationConfig"] then CheckboxSetters["InformationConfig"](Config_data.informationShowConfig, true) end
				if Config_data.informationShowPlayers ~= nil and CheckboxSetters["InformationPlayers"] then CheckboxSetters["InformationPlayers"](Config_data.informationShowPlayers, true) end
				if Config_data.toggleKey then _G.__PrivateHubToggleKeyBtn.Text = Config_data.toggleKey end
				if Config_data.startupSound then _G.__PHStartupSoundBox.Text = Config_data.startupSound end
				if Config_data.notificationSound then _G.__PHNotificationSoundBox.Text = Config_data.notificationSound end
				
				if Config_data.walkSpeedEnabled ~= nil and CheckboxSetters["WalkSpeedToggle"] then CheckboxSetters["WalkSpeedToggle"](Config_data.walkSpeedEnabled, true) end -- ★ Config読み込み用
				if Config_data.walkSpeed and SliderSetters["WalkSpeed"] then SliderSetters["WalkSpeed"](Config_data.walkSpeed) end
				if Config_data.infJump ~= nil and CheckboxSetters["InfJump"] then CheckboxSetters["InfJump"](Config_data.infJump, true) end
				if Config_data.noclip ~= nil and CheckboxSetters["Noclip"] then CheckboxSetters["Noclip"](Config_data.noclip, true) end
				if Config_data.spinEnabled ~= nil and CheckboxSetters["Spin"] then CheckboxSetters["Spin"](Config_data.spinEnabled, true) end
				if Config_data.spinSpeed and SliderSetters["SpinSpeed"] then SliderSetters["SpinSpeed"](Config_data.spinSpeed) end

				if Config_data.matchMode and _G.__PrivateHubMatchSetMode then
					_G.__PrivateHubMatchSetMode(Config_data.matchMode)
				end
				if Config_data.matchAutoQueue ~= nil and CheckboxSetters["AutoMatch"] then
					CheckboxSetters["AutoMatch"](Config_data.matchAutoQueue, true)
				end
				if Config_data.autoVoteMapRank1 and _G.__PHAutoVoteMapUI and _G.__PHAutoVoteMapUI.Rank1Set then _G.__PHAutoVoteMapUI.Rank1Set(Config_data.autoVoteMapRank1) end
				if Config_data.autoVoteMapRank2 and _G.__PHAutoVoteMapUI and _G.__PHAutoVoteMapUI.Rank2Set then _G.__PHAutoVoteMapUI.Rank2Set(Config_data.autoVoteMapRank2) end
				if Config_data.autoVoteMapRank3 and _G.__PHAutoVoteMapUI and _G.__PHAutoVoteMapUI.Rank3Set then _G.__PHAutoVoteMapUI.Rank3Set(Config_data.autoVoteMapRank3) end
				if Config_data.autoVoteMap ~= nil and CheckboxSetters["AutoVoteMap"] then
					CheckboxSetters["AutoVoteMap"](Config_data.autoVoteMap, true)
				end
				if _G.__PrivateHubTeleportApplyConfig then
					_G.__PrivateHubTeleportApplyConfig(Config_data)
				end

				if Config_data.silentAim ~= nil and CheckboxSetters["SilentAim"] then CheckboxSetters["SilentAim"](Config_data.silentAim, true) end
				if Config_data.aimbot ~= nil and CheckboxSetters["Aimbot"] then CheckboxSetters["Aimbot"](Config_data.aimbot, true) end
				if Config_data.wallCheck ~= nil and CheckboxSetters["WallCheck"] then CheckboxSetters["WallCheck"](Config_data.wallCheck, true) end
				if Config_data.fovRadius and SliderSetters["FOVRadius"] then SliderSetters["FOVRadius"](Config_data.fovRadius) end
				if Config_data.fovColor then fovColorBtn.BackgroundColor3 = Config_hexToColor(Config_data.fovColor) end
				if Config_data.fovRainbow ~= nil and CheckboxSetters["FOVRainbow"] then CheckboxSetters["FOVRainbow"](Config_data.fovRainbow, true) end
				if Config_data.fovFilled ~= nil and CheckboxSetters["FOVFilled"] then CheckboxSetters["FOVFilled"](Config_data.fovFilled, true) end
				if Config_data.autoShotCooldown and SliderSetters["AutoShotCooldown"] then SliderSetters["AutoShotCooldown"](Config_data.autoShotCooldown) end
				if Config_data.autoShot ~= nil and CheckboxSetters["AutoShot"] then CheckboxSetters["AutoShot"](Config_data.autoShot, true) end

				-- Team ESP 読み込み
				if Config_data.mvsdEspEnabled ~= nil and CheckboxSetters["MVSD_ESP"] then CheckboxSetters["MVSD_ESP"](Config_data.mvsdEspEnabled, true) end
				if Config_data.mvsdEspColor then 
					MVSD_ESP_Color = Config_hexToColor(Config_data.mvsdEspColor)
					espColorBtn.BackgroundColor3 = MVSD_ESP_Color
				end

				if Config_data.serverDesyncColor then
					ServerDesync_Color = Config_hexToColor(Config_data.serverDesyncColor)
					serverDesyncColorBtn.BackgroundColor3 = ServerDesync_Color
				end
				if Config_data.serverDesyncEnabled ~= nil and CheckboxSetters["ServerDesync"] then CheckboxSetters["ServerDesync"](Config_data.serverDesyncEnabled, true) end
				if Config_data.hitboxEnabled ~= nil and CheckboxSetters["Hitbox"] then CheckboxSetters["Hitbox"](Config_data.hitboxEnabled, true) end
				if Config_data.hitboxSize and SliderSetters["HitboxSize"] then SliderSetters["HitboxSize"](Config_data.hitboxSize) end
				if Config_data.hitboxColor then
					Hitbox_Color = Config_hexToColor(Config_data.hitboxColor)
					hitboxColorBtn.BackgroundColor3 = Hitbox_Color
				end

				-- Appearance
				if Config_data.appearanceIsVip ~= nil and CheckboxSetters["Appearance_IsVip"] then
					CheckboxSetters["Appearance_IsVip"](Config_data.appearanceIsVip, true)
				end
				if Config_data.appearanceWinStreakEnabled ~= nil and CheckboxSetters["Appearance_WinStreak"] then
					CheckboxSetters["Appearance_WinStreak"](Config_data.appearanceWinStreakEnabled, true)
				end
				if Config_data.appearanceWinStreakValue ~= nil and SliderSetters["Appearance_WinStreakValue"] then
					SliderSetters["Appearance_WinStreakValue"](Config_data.appearanceWinStreakValue)
				end
				applyAppearanceAttributes()

				if Config_data.worldTimeEnabled ~= nil and CheckboxSetters["WorldTimeEnabled"] then CheckboxSetters["WorldTimeEnabled"](Config_data.worldTimeEnabled, true) end
				if Config_data.worldTimeValue and SliderSetters["WorldTimeValue"] then SliderSetters["WorldTimeValue"](Config_data.worldTimeValue) end

				if Config_data.worldChangerEnabled ~= nil and CheckboxSetters["WorldChangerEnabled"] then CheckboxSetters["WorldChangerEnabled"](Config_data.worldChangerEnabled, true) end
				if Config_data.worldChangerColor then
					_G.__PHWorldChangerColor = Config_hexToColor(Config_data.worldChangerColor)
					worldChangerColorBtn.BackgroundColor3 = _G.__PHWorldChangerColor
					if WorldChangerEnabled then applyWorldChanger() end
				end

				if Config_data.weatherType then setWeatherType(Config_data.weatherType == "Snow" and "Snow" or "Rain") end
				if Config_data.weatherEnabled ~= nil then setWeatherEnabled(Config_data.weatherEnabled == true) end

				if Config_data.minecraftTextureEnabled ~= nil and CheckboxSetters["MinecraftTexture"] then CheckboxSetters["MinecraftTexture"](Config_data.minecraftTextureEnabled, true) end

				if Config_data.customCrosshairEnabled ~= nil and CheckboxSetters["CustomCrosshair"] then CheckboxSetters["CustomCrosshair"](Config_data.customCrosshairEnabled, true) end
				if Config_data.customCrosshairRainbow ~= nil and CheckboxSetters["CustomCrosshairRainbow"] then CheckboxSetters["CustomCrosshairRainbow"](Config_data.customCrosshairRainbow, true) end
				if Config_data.customCrosshairColor then 
					customCrosshairColor = Config_hexToColor(Config_data.customCrosshairColor)
					_G.__PHCrosshairColorBtn.BackgroundColor3 = customCrosshairColor
				end
				if Config_data.watermarkText then
					_G.__PHWatermarkBox.Text = Config_data.watermarkText
					CursorText.Text = Config_data.watermarkText
				end
				if Config_data.effectSelected and _G.__PHEffectState then
					_G.__PHEffectState.Selected = Config_data.effectSelected
					if _G.__PHEffectUIRefs and _G.__PHEffectUIRefs.drop then _G.__PHEffectUIRefs.drop.Text = Config_data.effectSelected end
				end
				if Config_data.effectImageId and _G.__PHEffectState then
					_G.__PHEffectState.ImageId = Config_data.effectImageId
					if _G.__PHEffectUIRefs and _G.__PHEffectUIRefs.ImageId then _G.__PHEffectUIRefs.ImageId.Text = Config_data.effectImageId end
				end
				if Config_data.effectSoundId and _G.__PHEffectState then
					_G.__PHEffectState.SoundId = Config_data.effectSoundId
					if _G.__PHEffectUIRefs and _G.__PHEffectUIRefs.SoundId then _G.__PHEffectUIRefs.SoundId.Text = Config_data.effectSoundId end
				end
				if Config_data.effectEnabled ~= nil and CheckboxSetters["EffectEnabled"] then CheckboxSetters["EffectEnabled"](Config_data.effectEnabled, true) end
				if Config_data.effectRandom ~= nil and CheckboxSetters["EffectRandom"] then CheckboxSetters["EffectRandom"](Config_data.effectRandom, true) end
				if Config_data.effectImageEnabled ~= nil and CheckboxSetters["EffectImageEnabled"] then CheckboxSetters["EffectImageEnabled"](Config_data.effectImageEnabled, true) end
				if Config_data.effectSoundEnabled ~= nil and CheckboxSetters["EffectSoundEnabled"] then CheckboxSetters["EffectSoundEnabled"](Config_data.effectSoundEnabled, true) end

				if Config_data.gunColor then
					_G.__PHGunColorState.Color = Config_hexToColor(Config_data.gunColor)
					if _G.__PHGunColorBtn then _G.__PHGunColorBtn.BackgroundColor3 = _G.__PHGunColorState.Color end
				end
				if Config_data.gunColorSoundId and _G.__PHGunColorState then
					_G.__PHGunColorState.SoundId = Config_data.gunColorSoundId
					if _G.__PHGunColorUI and _G.__PHGunColorUI.soundBox then _G.__PHGunColorUI.soundBox.Text = Config_data.gunColorSoundId end
				end
				if Config_data.gunColorEnabled ~= nil and CheckboxSetters["GUNcolor"] then CheckboxSetters["GUNcolor"](Config_data.gunColorEnabled, true) end

				-- Abilities
				if Config_data.knifeSpeedEnabled ~= nil and CheckboxSetters["KnifeSpeed"] then
					CheckboxSetters["KnifeSpeed"](Config_data.knifeSpeedEnabled, true)
				end
				if Config_data.knifeSpeedValue and SliderSetters["KnifeSpeedValue"] then
					SliderSetters["KnifeSpeedValue"](Config_data.knifeSpeedValue)
				end

				if Config_data.customCooldownEnabled ~= nil and CheckboxSetters["CustomCooldown"] then
					CheckboxSetters["CustomCooldown"](Config_data.customCooldownEnabled, true)
				end
				if Config_data.cooldownValue and SliderSetters["CooldownValue"] then
					SliderSetters["CooldownValue"](Config_data.cooldownValue)
				end

				if Config_data.dashEnabled ~= nil and CheckboxSetters["Dash"] then
					CheckboxSetters["Dash"](Config_data.dashEnabled, true)
				end
				if Config_data.dashSpeed and SliderSetters["DashSpeed"] then
					SliderSetters["DashSpeed"](Config_data.dashSpeed)
				end
				if Config_data.dashTime and SliderSetters["DashTime"] then
					SliderSetters["DashTime"](Config_data.dashTime)
				end
				if Config_data.dashCooldown and SliderSetters["DashCooldown"] then
					SliderSetters["DashCooldown"](Config_data.dashCooldown)
				end

				if Config_data.unlockMovementEnabled ~= nil and CheckboxSetters["UnlockMovement"] then
					CheckboxSetters["UnlockMovement"](Config_data.unlockMovementEnabled, true)
				end

				if Config_data.invisibilityEnabled ~= nil and CheckboxSetters["Invisibility"] then
					CheckboxSetters["Invisibility"](Config_data.invisibilityEnabled, true)
				end

				if Config_data.rageKill ~= nil and CheckboxSetters["RageKill"] then CheckboxSetters["RageKill"](Config_data.rageKill, true) end
				if Config_data.rageKillMode ~= nil then
					RageKillMode = (Config_data.rageKillMode == "KNIFE") and "KNIFE" or "GUN"
					if _G.__PrivateHubRageModeBtn then
						_G.__PrivateHubRageModeBtn.Text = "Weapon: " .. RageKillMode
					end
					if RageKillEnabled then
						isRoundActive = (RageKillMode == "GUN")
					end
				end
                if Config_data.rageKillTargetEnabled ~= nil then
                    _G.__PrivateHubRageTargetState.Enabled = Config_data.rageKillTargetEnabled == true
                    if CheckboxSetters["RageKillTarget"] then
                        CheckboxSetters["RageKillTarget"](_G.__PrivateHubRageTargetState.Enabled, true)
                    end
                end
                if Config_data.rageKillTargetName ~= nil then
                    _G.__PrivateHubRageTargetState.TargetName = tostring(Config_data.rageKillTargetName or "")
                end
                if Config_data.rageKillTargetMarkerColor then
                    _G.__PrivateHubRageTargetState.TargetMarkerColor = Config_hexToColor(Config_data.rageKillTargetMarkerColor)
                end
                if Config_data.rageKillTextColor then
                    _G.__PrivateHubRageTargetState.TextColor = Config_hexToColor(Config_data.rageKillTextColor)
                end

                if Config_data.slowKill ~= nil and CheckboxSetters["SlowKill"] then CheckboxSetters["SlowKill"](Config_data.slowKill, true) end
                if Config_data.slowKillDelay ~= nil then
                    SlowKillDelay = math.clamp(tonumber(Config_data.slowKillDelay) or 0.1, 0.1, 1.0)
                    _G.__PrivateHubSlowKillState = _G.__PrivateHubSlowKillState or {}
                    _G.__PrivateHubSlowKillState.Delay = SlowKillDelay
                    if SliderSetters["SlowKillDelay"] then SliderSetters["SlowKillDelay"](SlowKillDelay) end
                end
				if Config_data.autoEquip ~= nil and CheckboxSetters["AutoEquip"] then CheckboxSetters["AutoEquip"](Config_data.autoEquip, true) end
				if Config_data.antiLag ~= nil and CheckboxSetters["AntiLag"] then CheckboxSetters["AntiLag"](Config_data.antiLag, true) end

				applyThemeColors()
				
				if writefile then
					pcall(function() writefile(Config_lastConfigPath, Config_cName) end)
				end
				return true
			end
		end
	end
	return false
end

btnLoad.MouseButton1Click:Connect(function()
	if Config_selectedConfigName == "" then return end
	if Config_loadConfigByName(Config_selectedConfigName) then
		showNotification("Config loaded: " .. Config_selectedConfigName)
	end
end)

_G.__PrivateHubConfigState.refreshConfigList()

if readfile and isfile and isfile(Config_lastConfigPath) then
	-- ローカルレジスタ節約のため、この一時値は _G に置く
	_G.__PrivateHubLastConfigName = nil
	if pcall(function()
		_G.__PrivateHubLastConfigName = readfile(Config_lastConfigPath)
	end) then
		if _G.__PrivateHubLastConfigName and _G.__PrivateHubLastConfigName ~= "" then
			if isfile(Config_subFolderPath .. "/" .. _G.__PrivateHubLastConfigName .. ".txt") then
				Config_selectedConfigName = _G.__PrivateHubLastConfigName
				dropdownBtn.Text = _G.__PrivateHubLastConfigName
				Config_loadConfigByName(_G.__PrivateHubLastConfigName)
			end
		end
	end
	_G.__PrivateHubLastConfigName = nil
end

end
playSound(tonumber(_G.__PHStartupSoundBox.Text) or 0)

_G.__PrivateHubGlitchTime = os.clock()
RunService.RenderStepped:Connect(function()
	if os.clock() - _G.__PrivateHubGlitchTime >= 0.06 then
		_G.__PrivateHubGlitchTime = os.clock()
		glitchLabel.Text = "[" .. string.char(
			math.random(65, 90), math.random(65, 90), math.random(48, 57),
			math.random(65, 90), math.random(48, 57), math.random(65, 90)
		) .. "]"
	end
end)

-- MainFrame.Draggable は使わない。
-- Roblox の標準 Draggable は子要素（スライダーを含む）の入力でも
-- MainFrame を移動させるため、TopBar だけをドラッグ領域にする。
-- ローカルレジスタ節約:
-- setupMainFrameDrag 自体を local function にせず、状態も _G に退避する。
_G.__PrivateHubDragState = _G.__PrivateHubDragState or {
	dragActive = false,
	dragStart = nil,
	dragStartPosition = nil,
	initialized = false
}

if not _G.__PrivateHubDragState.initialized then
	_G.__PrivateHubDragState.initialized = true

	mainFrame.Active = true
	topBar.Active = true

	topBar.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			_G.__PrivateHubDragState.dragActive = true
			_G.__PrivateHubDragState.dragStart = input.Position
			_G.__PrivateHubDragState.dragStartPosition = mainFrame.Position

			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					_G.__PrivateHubDragState.dragActive = false
				end
			end)
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		local state = _G.__PrivateHubDragState
		if not state.dragActive then return end
		if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end

		local delta = input.Position - state.dragStart
		mainFrame.Position = UDim2.new(
			state.dragStartPosition.X.Scale,
			state.dragStartPosition.X.Offset + delta.X,
			state.dragStartPosition.Y.Scale,
			state.dragStartPosition.Y.Offset + delta.Y
		)
	end)
end


do
-- ==========================================================
-- NOTE: Isolate Kill Engine in a nested function to avoid Luau local-register exhaustion.
-- Shared values from the main hub become upvalues instead of additional top-level locals.
task.spawn(function()
-- PRIVATE HUB EXTENSION: KILL ENGINE & AUTO EQUIP
-- (Main Tab UI / Config Compatible / Center Minecraft UI)
-- ==========================================================

-- 1. 画面中央用の Minecraft 風 Killer UI の構築
local killEngineGui = Instance.new("ScreenGui")
killEngineGui.Name = "PrivateHubKillEngineGui"
killEngineGui.ResetOnSpawn = false
killEngineGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
killEngineGui.Parent = playerGui

local centerKillFrame = Instance.new("Frame")
centerKillFrame.Name = "CenterKillFrame"
centerKillFrame.Size = UDim2.new(0, 450, 0, 50)
centerKillFrame.AnchorPoint = Vector2.new(0.5, 0.5)
centerKillFrame.Position = UDim2.new(0.5, 0, 0.4, 0)
centerKillFrame.BackgroundTransparency = 1
centerKillFrame.Visible = false
centerKillFrame.Parent = killEngineGui

local centerKillLabel = Instance.new("TextLabel")
centerKillLabel.Size = UDim2.new(1, 0, 1, 0)
centerKillLabel.BackgroundTransparency = 1
centerKillLabel.Font = Enum.Font.Arcade
centerKillLabel.Text = "Rage Kill: Idle"
centerKillLabel.TextColor3 = _G.__PrivateHubRageTargetState.TextColor or Color3.fromRGB(255, 50, 50)
centerKillLabel.TextSize = 22
centerKillLabel.Parent = centerKillFrame
_G.__PHRageKillLabel = centerKillLabel

local centerStroke = Instance.new("UIStroke")
centerStroke.Thickness = 2
centerStroke.Color = Color3.fromRGB(0, 0, 0)
centerStroke.Parent = centerKillLabel

-- 2. 機能状態フラグ変数
local SLOW_KILL_RANGE = 1000
-- Slow Kill interval is config-backed and controlled by the UI slider (0.1s - 1.0s).
_G.__PrivateHubSlowKillState = _G.__PrivateHubSlowKillState or {Delay = SlowKillDelay}
SlowKillDelay = math.clamp(tonumber(_G.__PrivateHubSlowKillState.Delay) or 0.1, 0.1, 1.0)
_G.__PrivateHubSlowKillState.Delay = SlowKillDelay
local EQUIP_CHECK_RATE = 0.1

local isRoundActive = false
local currentTargetIndex = 1

-- 3. 共通ヘルパー関数（銃自動検出）
local function findCorrectGunGlobal()
	local bp = LocalPlayer:FindFirstChild("Backpack")
	if not bp then return nil end
	for _, tool in ipairs(bp:GetChildren()) do
		if tool:IsA("Tool") then
			if tool:GetAttribute("EquipAnimation") == "Gun_Equip" then return tool end
			local prop = tool:FindFirstChild("EquipAnimation")
			if prop and prop.Value == "Gun_Equip" then return tool end
			local success, value = pcall(function() return tool.EquipAnimation end)
			if success and value == "Gun_Equip" then return tool end
		end
	end
	return nil
end

-- 4. リモートイベント・タスク定義
local RemotesFolder = ReplicatedStorage:WaitForChild("Remotes", 5)
local OnRoundIntermissionStarted = RemotesFolder and RemotesFolder:FindFirstChild("OnRoundIntermissionStarted")
local OnRoundEnded = RemotesFolder and RemotesFolder:FindFirstChild("OnRoundEnded")
local ShootGunRemote = RemotesFolder and RemotesFolder:FindFirstChild("ShootGun")

if OnRoundIntermissionStarted then
	OnRoundIntermissionStarted.OnClientEvent:Connect(function()
		if not RageKillEnabled or RageKillMode ~= "GUN" then return end
		centerKillLabel.Text = "Rage Kill: Starting in 3s..."
		centerKillFrame.Visible = true
		task.delay(3, function()
			if RageKillEnabled and RageKillMode == "GUN" then
				isRoundActive = true
				currentTargetIndex = 1
			end
		end)
	end)
end

if OnRoundEnded then
	OnRoundEnded.OnClientEvent:Connect(function()
		isRoundActive = false
		currentTargetIndex = 1
		if RageKillEnabled and RageKillMode == "GUN" then
			centerKillLabel.Text = "Rage Kill: Round Ended"
		end
	end)
end

-- [[ ループ1: Rage Kill GUN メイン処理 ]]
-- 旧ターゲット1人ずつ処理を廃止し、提示された Faster Max 方式に置換。
-- Team1/Team2 + 同一Match Workspace の敵だけを対象に、
-- Heartbeatごとに最大1500 studsまで ShootGun を送る。
task.spawn(function()
    while true do
        RunService.Heartbeat:Wait()

        if not RageKillEnabled or RageKillMode ~= "GUN" then
            if not RageKillEnabled or RageKillMode == "KNIFE" then
                centerKillFrame.Visible = false
            end
            continue
        end

        centerKillFrame.Visible = true
        centerKillLabel.TextColor3 = _G.__PrivateHubRageTargetState.TextColor or Color3.fromRGB(255, 50, 50)

        local char = LocalPlayer.Character
        local myHRP = char and char:FindFirstChild("HumanoidRootPart")
        if not myHRP then
            centerKillLabel.Text = "Rage Kill: Waiting Character..."
            continue
        end

        local myTeamName = GetPlayerTeam(LocalPlayer):lower()
        if myTeamName ~= "team1" and myTeamName ~= "team2" then
            centerKillLabel.Text = "Rage Kill: Waiting Team..."
            continue
        end

        if not ShootGunRemote then
            local remotes = ReplicatedStorage:FindFirstChild("Remotes")
            ShootGunRemote = remotes and remotes:FindFirstChild("ShootGun")
        end

        if not ShootGunRemote then
            centerKillLabel.Text = "Rage Kill: ShootGun Missing"
            continue
        end

        local myPos = myHRP.Position
        local targetCount = 0

        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local targetTeamName = GetPlayerTeam(player):lower()
                local isEnemy =
                    (targetTeamName == "team1" or targetTeamName == "team2")
                    and myTeamName ~= targetTeamName
                    and IsInSameMatchWorkspace(player)

                if isEnemy and _G.__PrivateHubRageTargetState.Enabled then
                    isEnemy = (_G.__PrivateHubRageTargetState.TargetName ~= "" and player.Name == _G.__PrivateHubRageTargetState.TargetName)
                end

                if isEnemy then
                    local tChar = player.Character
                    local tHRP = tChar and tChar:FindFirstChild("HumanoidRootPart")
                    local tHum = tChar and tChar:FindFirstChildOfClass("Humanoid")

                    if tHRP and tHum and tHum.Health > 0
                        and (myPos - tHRP.Position).Magnitude <= 1500 then
                        targetCount = targetCount + 1
                        pcall(function()
                            ShootGunRemote:FireServer(myPos, tHRP.Position, tHRP, tHRP.Position)
                        end)
                    end
                end
            end
        end

        if _G.__PrivateHubRageTargetState.Enabled and _G.__PrivateHubRageTargetState.TargetName == "" then
            centerKillLabel.Text = "Rage Kill: Select Target"
        elseif targetCount > 0 then
            centerKillLabel.Text = _G.__PrivateHubRageTargetState.Enabled
                and ("Rage Kill: " .. _G.__PrivateHubRageTargetState.TargetName)
                or ("Rage Kill: " .. tostring(targetCount) .. " Target(s)")
        elseif _G.__PrivateHubRageTargetState.Enabled then
            centerKillLabel.Text = "Rage Kill: Target Not Available"
        else
            centerKillLabel.Text = "Rage Kill: No Targets"
        end
    end
end)

-- [[ ループ2: Rage Kill KNIFE / AREA KILL ]]
-- GUNモードの既存Rage Killはそのまま維持し、KNIFE選択時だけこちらを動かす。
_G.__PHRageGetKnifeByProperty = function()
	local bp = LocalPlayer:FindFirstChild("Backpack")
	if not bp then return nil end
	for _, tool in ipairs(bp:GetChildren()) do
		if tool:IsA("Tool") then
			local success, value = pcall(function() return tool.EquipAnimation end)
			if (success and value == "Knife_Equip") or tool:GetAttribute("EquipAnimation") == "Knife_Equip" then
				return tool
			end
		end
	end
	return nil
end

_G.__PHRageGetKnifeRemotes = function()
	local remotes = ReplicatedStorage:FindFirstChild("Remotes")
	return remotes and remotes:FindFirstChild("ThrowHit")
end

task.spawn(function()
    while true do
        if RageKillEnabled and RageKillMode == "KNIFE" then
            local myChar = LocalPlayer.Character
            local hum = myChar and myChar:FindFirstChildOfClass("Humanoid")
            if myChar and hum and not myChar:FindFirstChildOfClass("Tool") then
                local knife = _G.__PHRageGetKnifeByProperty()
                if knife then pcall(function() hum:EquipTool(knife) end) end
            end

            local hitRemote = _G.__PHRageGetKnifeRemotes()
            local myPos = (myChar and myChar.PrimaryPart) and myChar.PrimaryPart.Position or Vector3.new(0, 0, 0)
            if hitRemote then
                local myTeamName = GetPlayerTeam(LocalPlayer):lower()
                if myTeamName == "team1" or myTeamName == "team2" then
                    for _, p in ipairs(Players:GetPlayers()) do
                        if p ~= LocalPlayer and p.Character then
                            local targetTeamName = GetPlayerTeam(p):lower()
                            local isEnemy = false

                            if (targetTeamName == "team1" or targetTeamName == "team2") and myTeamName ~= targetTeamName then
                                if IsInSameMatchWorkspace(p) then isEnemy = true end
                            end

                            if isEnemy and _G.__PrivateHubRageTargetState.Enabled then
                                isEnemy = (_G.__PrivateHubRageTargetState.TargetName ~= "" and p.Name == _G.__PrivateHubRageTargetState.TargetName)
                            end

                            if isEnemy then
                                local head = p.Character:FindFirstChild("Head") or p.Character.PrimaryPart
                                local enemyHum = p.Character:FindFirstChildOfClass("Humanoid")
                                if head and enemyHum and enemyHum.Health > 0 and (head.Position - myPos).Magnitude <= 1000 then
                                    pcall(function() hitRemote:FireServer(head, head.Position) end)
                                end
                            end
                        end
                    end
                end
            end

            centerKillFrame.Visible = true
            centerKillLabel.TextColor3 = _G.__PrivateHubRageTargetState.TextColor or Color3.fromRGB(255, 50, 50)
            if _G.__PrivateHubRageTargetState.Enabled and _G.__PrivateHubRageTargetState.TargetName ~= "" then
                centerKillLabel.Text = "Rage Kill: " .. _G.__PrivateHubRageTargetState.TargetName
            elseif _G.__PrivateHubRageTargetState.Enabled then
                centerKillLabel.Text = "Rage Kill: Select Target"
            else
                centerKillLabel.Text = "Rage Kill: KNIFE"
            end
        end
        RunService.Heartbeat:Wait()
    end
end)

-- [[ ループ3: Slow All Kill メイン処理 ]]
task.spawn(function()
	while true do
		task.wait(math.clamp(tonumber(_G.__PrivateHubSlowKillState and _G.__PrivateHubSlowKillState.Delay) or SlowKillDelay or 0.1, 0.1, 1.0))
		if not SlowKillEnabled then continue end

		local char = LocalPlayer.Character
		local myHRP = char and char:FindFirstChild("HumanoidRootPart")
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if not myHRP or not hum then continue end

		local myTeamName = GetPlayerTeam(LocalPlayer):lower()
		if myTeamName ~= "team1" and myTeamName ~= "team2" then continue end

		if not char:FindFirstChildOfClass("Tool") then
			local targetGun = findCorrectGunGlobal()
			if targetGun then pcall(function() hum:EquipTool(targetGun) end) end
		end

		if ShootGunRemote then
			for _, player in ipairs(Players:GetPlayers()) do
				if player == LocalPlayer then continue end
				
				local targetTeamName = GetPlayerTeam(player):lower()
				local isEnemy = false
				
				if targetTeamName == "team1" or targetTeamName == "team2" then
					if myTeamName ~= targetTeamName then
						if IsInSameMatchWorkspace(player) then
							isEnemy = true
						end
					end
				end

				if isEnemy then
					local tChar = player.Character
					local tHRP = tChar and tChar:FindFirstChild("HumanoidRootPart")
					local tHum = tChar and tChar:FindFirstChildOfClass("Humanoid")

					if tHRP and tHum and tHum.Health > 0 then
						if (myHRP.Position - tHRP.Position).Magnitude <= SLOW_KILL_RANGE then
							pcall(function()
								ShootGunRemote:FireServer(myHRP.Position, tHRP.Position, tHRP, tHRP.Position)
							end)
						end
					end
				end
			end
		end
	end
end)

-- [[ ループ3: Auto Equip Gun メイン処理 ]]
task.spawn(function()
	while true do
		task.wait(EQUIP_CHECK_RATE)
		if not AutoEquipEnabled then continue end

		local character = LocalPlayer.Character
		if not character then continue end
		local humanoid = character:FindFirstChildOfClass("Humanoid")
		if not humanoid or humanoid.Health <= 0 then continue end

		local currentTool = character:FindFirstChildOfClass("Tool")
		local hasCorrectTool = false
		
		if currentTool then
			local success, value = pcall(function() return currentTool.EquipAnimation end)
			if (success and value == "Gun_Equip") or currentTool:GetAttribute("EquipAnimation") == "Gun_Equip" then
				local valObj = currentTool:FindFirstChild("EquipAnimation")
				if (valObj and valObj:IsA("StringValue") and valObj.Value == "Gun_Equip") or not valObj then
					hasCorrectTool = true
				end
			end
		end

		if not hasCorrectTool then
			local targetGun = findCorrectGunGlobal()
			if targetGun then
				pcall(function()
					humanoid:EquipTool(targetGun)
				end)
			end
		end
	end
end)

-- 5. Main タブに Combat Section（トグル枠）を新しく追加
local combatSection = Instance.new("Frame")
combatSection.Name = "CombatSection"
combatSection.Size = UDim2.new(0.92, 0, 0, 555)
combatSection.Position = UDim2.new(0.04, 0, 0, 2270)
combatSection.BackgroundColor3 = Color3.fromRGB(16, 16, 18)
combatSection.BorderSizePixel = 0
combatSection.Parent = mainScroll
Instance.new("UICorner", combatSection).CornerRadius = UDim.new(0, 6)
addStroke(combatSection, Color3.fromRGB(40, 40, 45), 0, 1)

mainScroll.CanvasSize = UDim2.new(0, 0, 0, 3295)

local combatTitle = Instance.new("TextLabel")
combatTitle.Name = "DynamicText"
combatTitle.Size = UDim2.new(1, 0, 0, 30)
combatTitle.BackgroundTransparency = 1
combatTitle.Font = FONT_BOLD
combatTitle.Text = "Combat Engine"
combatTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
combatTitle.TextSize = 13
combatTitle.Parent = combatSection

CheckboxSetters["RageKill"] = createCheckboxToggle(combatSection, "Rage Kill", 32, function(enabled)
	RageKillEnabled = enabled
	if enabled then
		isRoundActive = (RageKillMode == "GUN")
	else
		isRoundActive = false
	end
end)

-- Rage Kill weapon selector: GUN / KNIFE
_G.__PrivateHubRageSetMode = function(mode)
	if mode ~= "GUN" and mode ~= "KNIFE" then mode = "GUN" end
	RageKillMode = mode
	if _G.__PrivateHubRageModeBtn then
		_G.__PrivateHubRageModeBtn.Text = "Weapon: " .. mode
	end
	if mode == "GUN" and RageKillEnabled then
		isRoundActive = true
	elseif mode == "KNIFE" then
		isRoundActive = false
	end
	if _G.__PrivateHubRageDropdown then _G.__PrivateHubRageDropdown.Visible = false end
end

_G.__PrivateHubRageModeBtn = Instance.new("TextButton")
_G.__PrivateHubRageModeBtn.Name = "RageKillWeaponDropdown"
_G.__PrivateHubRageModeBtn.Size = UDim2.new(1, -20, 0, 30)
_G.__PrivateHubRageModeBtn.Position = UDim2.new(0, 10, 0, 69)
_G.__PrivateHubRageModeBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
_G.__PrivateHubRageModeBtn.BorderSizePixel = 0
_G.__PrivateHubRageModeBtn.Font = FONT_MAIN
_G.__PrivateHubRageModeBtn.Text = "Weapon: " .. RageKillMode
_G.__PrivateHubRageModeBtn.TextColor3 = Color3.fromRGB(205, 205, 210)
_G.__PrivateHubRageModeBtn.TextSize = 12
_G.__PrivateHubRageModeBtn.TextXAlignment = Enum.TextXAlignment.Left
_G.__PrivateHubRageModeBtn.Parent = combatSection
Instance.new("UICorner", _G.__PrivateHubRageModeBtn).CornerRadius = UDim.new(0, 4)
addPadding(_G.__PrivateHubRageModeBtn, 10)
addStroke(_G.__PrivateHubRageModeBtn, Color3.fromRGB(50, 50, 55), 0, 1)

_G.__PrivateHubRageDropdown = Instance.new("Frame")
_G.__PrivateHubRageDropdown.Name = "RageKillWeaponList"
_G.__PrivateHubRageDropdown.Size = UDim2.new(1, -20, 0, 58)
_G.__PrivateHubRageDropdown.Position = UDim2.new(0, 10, 0, 101)
_G.__PrivateHubRageDropdown.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
_G.__PrivateHubRageDropdown.BorderSizePixel = 0
_G.__PrivateHubRageDropdown.Visible = false
_G.__PrivateHubRageDropdown.ZIndex = 20
_G.__PrivateHubRageDropdown.Parent = combatSection
Instance.new("UICorner", _G.__PrivateHubRageDropdown).CornerRadius = UDim.new(0, 4)
addStroke(_G.__PrivateHubRageDropdown, Color3.fromRGB(50, 50, 55), 0, 1)

for i, modeName in ipairs({"GUN", "KNIFE"}) do
	local modeBtn = Instance.new("TextButton")
	modeBtn.Size = UDim2.new(1, 0, 0, 29)
	modeBtn.Position = UDim2.new(0, 0, 0, (i - 1) * 29)
	modeBtn.BackgroundTransparency = 1
	modeBtn.BorderSizePixel = 0
	modeBtn.Font = FONT_MAIN
	modeBtn.Text = modeName
	modeBtn.TextColor3 = Color3.fromRGB(205, 205, 210)
	modeBtn.TextSize = 12
	modeBtn.TextXAlignment = Enum.TextXAlignment.Left
	modeBtn.ZIndex = 21
	modeBtn.Parent = _G.__PrivateHubRageDropdown
	addPadding(modeBtn, 10)
	modeBtn.MouseButton1Click:Connect(function() _G.__PrivateHubRageSetMode(modeName) end)
end

_G.__PrivateHubRageModeBtn.MouseButton1Click:Connect(function()
	_G.__PrivateHubRageDropdown.Visible = not _G.__PrivateHubRageDropdown.Visible
end)

-- ==========================================================
-- Rage Kill Target Lock / Marker
-- ==========================================================
local function clearRageTargetMarker()
    local h = _G.__PrivateHubRageTargetState.Highlight
    _G.__PrivateHubRageTargetState.Highlight = nil
    if h then pcall(function() h:Destroy() end) end
end

local function findRageTargetPlayer()
    local name = _G.__PrivateHubRageTargetState.TargetName
    if not name or name == "" then return nil end
    local p = Players:FindFirstChild(name)
    if not p or p == LocalPlayer then return nil end

    local myTeam = GetPlayerTeam(LocalPlayer):lower()
    local theirTeam = GetPlayerTeam(p):lower()
    if (myTeam ~= "team1" and myTeam ~= "team2")
        or (theirTeam ~= "team1" and theirTeam ~= "team2")
        or myTeam == theirTeam
        or not IsInSameMatchWorkspace(p) then
        return nil
    end

    local hum = p.Character and p.Character:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return nil end
    return p
end

local function updateRageTargetMarker()
    local p = findRageTargetPlayer()
    if not _G.__PrivateHubRageTargetState.Enabled or not p or not p.Character then
        clearRageTargetMarker()
        return
    end

    local h = _G.__PrivateHubRageTargetState.Highlight
    if not h or h.Parent ~= p.Character then
        clearRageTargetMarker()
        h = Instance.new("Highlight")
        h.Name = "PrivateHub_RageTargetMarker"
        h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        h.FillTransparency = 1
        h.OutlineTransparency = 0
        h.Parent = p.Character
        _G.__PrivateHubRageTargetState.Highlight = h
    end
    h.OutlineColor = _G.__PrivateHubRageTargetState.TargetMarkerColor or Color3.fromRGB(255, 70, 70)
    h.FillColor = _G.__PrivateHubRageTargetState.TargetMarkerColor or Color3.fromRGB(255, 70, 70)
end

local function setRageTargetEnabled(enabled)
    _G.__PrivateHubRageTargetState.Enabled = enabled == true
    if not _G.__PrivateHubRageTargetState.Enabled then clearRageTargetMarker() end
end

CheckboxSetters["RageKillTarget"] = createCheckboxToggle(combatSection, "Target Lock", 105, function(enabled)
    setRageTargetEnabled(enabled)
end)
if _G.__PrivateHubRageTargetState.Enabled then
    CheckboxSetters["RageKillTarget"](_G.__PrivateHubRageTargetState.Enabled, true)
end

_G.__PrivateHubRageTargetBtn = Instance.new("TextButton")
_G.__PrivateHubRageTargetBtn.Name = "RageKillTargetDropdown"
_G.__PrivateHubRageTargetBtn.Size = UDim2.new(1, -20, 0, 30)
_G.__PrivateHubRageTargetBtn.Position = UDim2.new(0, 10, 0, 142)
_G.__PrivateHubRageTargetBtn.BackgroundColor3 = Color3.fromRGB(22, 22, 25)
_G.__PrivateHubRageTargetBtn.BorderSizePixel = 0
_G.__PrivateHubRageTargetBtn.Font = FONT_MAIN
_G.__PrivateHubRageTargetBtn.Text = "Target: " .. (_G.__PrivateHubRageTargetState.TargetName ~= "" and _G.__PrivateHubRageTargetState.TargetName or "Select player...")
_G.__PrivateHubRageTargetBtn.TextColor3 = Color3.fromRGB(205, 205, 210)
_G.__PrivateHubRageTargetBtn.TextSize = 12
_G.__PrivateHubRageTargetBtn.TextXAlignment = Enum.TextXAlignment.Left
_G.__PrivateHubRageTargetBtn.Parent = combatSection
Instance.new("UICorner", _G.__PrivateHubRageTargetBtn).CornerRadius = UDim.new(0, 4)
addPadding(_G.__PrivateHubRageTargetBtn, 10)
addStroke(_G.__PrivateHubRageTargetBtn, Color3.fromRGB(50, 50, 55), 0, 1)

_G.__PrivateHubRageTargetList = Instance.new("ScrollingFrame")
_G.__PrivateHubRageTargetList.Name = "RageKillTargetList"
_G.__PrivateHubRageTargetList.Size = UDim2.new(1, -20, 0, 110)
_G.__PrivateHubRageTargetList.Position = UDim2.new(0, 10, 0, 174)
_G.__PrivateHubRageTargetList.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
_G.__PrivateHubRageTargetList.BorderSizePixel = 0
_G.__PrivateHubRageTargetList.Visible = false
_G.__PrivateHubRageTargetList.ZIndex = 30
_G.__PrivateHubRageTargetList.ScrollBarThickness = 3
_G.__PrivateHubRageTargetList.CanvasSize = UDim2.new(0, 0, 0, 0)
_G.__PrivateHubRageTargetList.Parent = combatSection
Instance.new("UICorner", _G.__PrivateHubRageTargetList).CornerRadius = UDim.new(0, 4)
addStroke(_G.__PrivateHubRageTargetList, Color3.fromRGB(50, 50, 55), 0, 1)

local function refreshRageTargetList()
    for _, child in ipairs(_G.__PrivateHubRageTargetList:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end

    local myTeam = GetPlayerTeam(LocalPlayer):lower()
    local names = {}
    if myTeam == "team1" or myTeam == "team2" then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer then
                local t = GetPlayerTeam(p):lower()
                if (t == "team1" or t == "team2") and t ~= myTeam and IsInSameMatchWorkspace(p) then
                    local hum = p.Character and p.Character:FindFirstChildOfClass("Humanoid")
                    if hum and hum.Health > 0 then table.insert(names, p.Name) end
                end
            end
        end
    end

    table.sort(names, function(a, b) return a:lower() < b:lower() end)

    for i, name in ipairs(names) do
        local b = Instance.new("TextButton")
        b.Size = UDim2.new(1, -4, 0, 27)
        b.Position = UDim2.new(0, 2, 0, (i - 1) * 27)
        b.BackgroundTransparency = 1
        b.BorderSizePixel = 0
        b.Font = FONT_MAIN
        b.Text = name
        b.TextColor3 = (name == _G.__PrivateHubRageTargetState.TargetName) and Color3.fromRGB(240, 200, 210) or Color3.fromRGB(205, 205, 210)
        b.TextSize = 12
        b.TextXAlignment = Enum.TextXAlignment.Left
        b.ZIndex = 31
        b.Parent = _G.__PrivateHubRageTargetList
        addPadding(b, 8)
        b.MouseButton1Click:Connect(function()
            _G.__PrivateHubRageTargetState.TargetName = name
            _G.__PrivateHubRageTargetBtn.Text = "Target: " .. name
            _G.__PrivateHubRageTargetList.Visible = false
            updateRageTargetMarker()
        end)
    end

    if #names == 0 then
        local empty = Instance.new("TextLabel")
        empty.Size = UDim2.new(1, -8, 0, 27)
        empty.Position = UDim2.new(0, 4, 0, 0)
        empty.BackgroundTransparency = 1
        empty.Font = FONT_MAIN
        empty.Text = "No enemy targets"
        empty.TextColor3 = Color3.fromRGB(140, 140, 145)
        empty.TextSize = 12
        empty.TextXAlignment = Enum.TextXAlignment.Left
        empty.ZIndex = 31
        empty.Parent = _G.__PrivateHubRageTargetList
        addPadding(empty, 4)
    end
    _G.__PrivateHubRageTargetList.CanvasSize = UDim2.new(0, 0, 0, math.max(27, #names * 27))
end

_G.__PrivateHubRageTargetBtn.MouseButton1Click:Connect(function()
    refreshRageTargetList()
    _G.__PrivateHubRageTargetList.Visible = not _G.__PrivateHubRageTargetList.Visible
end)

_G.__PHRageTargetMarkerColorBtn = createColorPreviewRowInParent(combatSection, "Target Marker Color", _G.__PrivateHubRageTargetState.TargetMarkerColor, 292)
_G.__PHRageTextColorBtn = createColorPreviewRowInParent(combatSection, "RageKill Text Color", _G.__PrivateHubRageTargetState.TextColor, 330)

_G.__PHRageTargetMarkerColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHRageTargetMarkerColorBtn) end)
_G.__PHRageTextColorBtn.MouseButton1Click:Connect(function() openPalette(_G.__PHRageTextColorBtn) end)

RunService.Heartbeat:Connect(updateRageTargetMarker)

CheckboxSetters["SlowKill"] = createCheckboxToggle(combatSection, "Slow Kill (Team)", 368, function(enabled)
	SlowKillEnabled = enabled
end)

-- Slow Kill interval: 0.1s - 1.0s, saved in CONFIG.
SliderSetters["SlowKillDelay"] = createSliderRow(
    combatSection,
    "Slow Kill Speed",
    0.1,
    1.0,
    SlowKillDelay,
    444,
    function(value)
        SlowKillDelay = math.clamp(tonumber(value) or 0.1, 0.1, 1.0)
        _G.__PrivateHubSlowKillState = _G.__PrivateHubSlowKillState or {}
        _G.__PrivateHubSlowKillState.Delay = SlowKillDelay
    end
)

CheckboxSetters["AutoEquip"] = createCheckboxToggle(combatSection, "Auto Equip Gun", 408, function(enabled)
	AutoEquipEnabled = enabled
end)

-- ==========================================================
-- PRIVATE HUB EXTENSION: CONFIG SAVE / LOAD PATCH
-- ==========================================================

if gatherSettingsData then
	local original_gatherSettingsData = gatherSettingsData
	gatherSettingsData = function()
		local jsonString = original_gatherSettingsData()
		local ok, data = pcall(function() return HttpService:JSONDecode(jsonString) end)
		if ok and type(data) == "table" then
			data.rageKill = RageKillEnabled
			data.rageKillMode = RageKillMode
            data.rageKillTargetEnabled = _G.__PrivateHubRageTargetState.Enabled == true
            data.rageKillTargetName = _G.__PrivateHubRageTargetState.TargetName or ""
            data.rageKillTargetMarkerColor = Config_colorToHex(_G.__PrivateHubRageTargetState.TargetMarkerColor or Color3.fromRGB(255, 70, 70))
            data.rageKillTextColor = Config_colorToHex(_G.__PrivateHubRageTargetState.TextColor or Color3.fromRGB(255, 50, 50))
			data.slowKill = SlowKillEnabled
            data.slowKillDelay = SlowKillDelay
			data.autoEquip = AutoEquipEnabled
			data.walkSpeedEnabled = ModSpeedEnabled -- 設定保存にも追加
			return HttpService:JSONEncode(data)
		end
		return jsonString
	end
end

if loadConfigByName then
	local original_loadConfigByName = loadConfigByName
	loadConfigByName = function(cName)
		local filePath = subFolderPath .. "/" .. cName .. ".txt"
		if readfile and isfile and isfile(filePath) then
			local success, content = pcall(function() return readfile(filePath) end)
			if success and content then
				local ok, data = pcall(function() return HttpService:JSONDecode(content) end)
				if ok and type(data) == "table" then
					original_loadConfigByName(cName)

					if data.rageKill ~= nil and CheckboxSetters["RageKill"] then 
						CheckboxSetters["RageKill"](data.rageKill, true) 
					end
					if data.rageKillMode ~= nil then
						if _G.__PrivateHubRageSetMode then
							_G.__PrivateHubRageSetMode(data.rageKillMode)
						else
							RageKillMode = (data.rageKillMode == "KNIFE") and "KNIFE" or "GUN"
						end
					end
                    if data.rageKillTargetEnabled ~= nil then
                        _G.__PrivateHubRageTargetState.Enabled = data.rageKillTargetEnabled == true
                        if CheckboxSetters["RageKillTarget"] then
                            CheckboxSetters["RageKillTarget"](_G.__PrivateHubRageTargetState.Enabled, true)
                        end
                    end
                    if data.rageKillTargetName ~= nil then
                        _G.__PrivateHubRageTargetState.TargetName = tostring(data.rageKillTargetName or "")
                        if _G.__PrivateHubRageTargetBtn then
                            _G.__PrivateHubRageTargetBtn.Text = "Target: " .. (_G.__PrivateHubRageTargetState.TargetName ~= "" and _G.__PrivateHubRageTargetState.TargetName or "Select player...")
                        end
                        if _G.__PrivateHubRageTargetState.Enabled then
                            updateRageTargetMarker()
                        end
                    end
                    if data.rageKillTargetMarkerColor then
                        _G.__PrivateHubRageTargetState.TargetMarkerColor = Config_hexToColor(data.rageKillTargetMarkerColor)
                        if _G.__PHRageTargetMarkerColorBtn then _G.__PHRageTargetMarkerColorBtn.BackgroundColor3 = _G.__PrivateHubRageTargetState.TargetMarkerColor end
                    end
                    if data.rageKillTextColor then
                        _G.__PrivateHubRageTargetState.TextColor = Config_hexToColor(data.rageKillTextColor)
                        if _G.__PHRageTextColorBtn then _G.__PHRageTextColorBtn.BackgroundColor3 = _G.__PrivateHubRageTargetState.TextColor end
                    end
                    if data.slowKill ~= nil and CheckboxSetters["SlowKill"] then 
						CheckboxSetters["SlowKill"](data.slowKill, true) 
					end
                    if data.slowKillDelay ~= nil then
                        SlowKillDelay = math.clamp(tonumber(data.slowKillDelay) or 0.1, 0.1, 1.0)
                        _G.__PrivateHubSlowKillState = _G.__PrivateHubSlowKillState or {}
                        _G.__PrivateHubSlowKillState.Delay = SlowKillDelay
                        if SliderSetters["SlowKillDelay"] then SliderSetters["SlowKillDelay"](SlowKillDelay) end
                    end
					if data.autoEquip ~= nil and CheckboxSetters["AutoEquip"] then 
						CheckboxSetters["AutoEquip"](data.autoEquip, true) 
					end

					return true
				end
			end
		end
		return original_loadConfigByName(cName)
	end
end
end)

end
