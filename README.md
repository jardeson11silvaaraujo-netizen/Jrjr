--// STEAL AN EGG - MOVEMENT FORENSIC SCANNER
--// SOMENTE OBSERVAÇÃO
--// COMEÇAR = inicia gravação
--// PARAR = encerra gravação
--// COPIAR LOG = copia SOMENTE quando você apertar o botão

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer

local logs = {}
local recording = false
local startedAt = 0
local connections = {}
local sampleConnection
local lastSample = 0
local sampleInterval = 0.05 -- 20 amostras/s

local function now()
    return os.clock() - startedAt
end

local function push(tag, msg)
    if not recording then return end

    local line = string.format(
        "[%8.3f] %-24s | %s",
        now(),
        tag,
        msg or ""
    )

    logs[#logs + 1] = line
end

local function v3(v)
    if typeof(v) ~= "Vector3" then
        return "nil"
    end

    return string.format(
        "(%.2f, %.2f, %.2f)",
        v.X, v.Y, v.Z
    )
end

local function getCharacter()
    local c = LocalPlayer.Character
    if not c then return nil end

    local root = c:FindFirstChild("HumanoidRootPart")
    local hum = c:FindFirstChildOfClass("Humanoid")

    return c, root, hum
end

local function getArea()
    local c = LocalPlayer.Character
    if not c then return "nil" end

    local area =
        c:GetAttribute("AreaId")
        or c:GetAttribute("Area")
        or c:GetAttribute("CurrentArea")

    if area then
        return tostring(area)
    end

    local ok, result = pcall(function()
        local plots = workspace:FindFirstChild("Plots")
        if not plots then return nil end

        for _, obj in ipairs(plots:GetChildren()) do
            local areaId = obj:GetAttribute("AreaId")
            if areaId then
                return areaId
            end
        end
    end)

    if ok and result then
        return tostring(result)
    end

    return "?"
end

local function findSnapshotRF()
    local packages = ReplicatedStorage:FindFirstChild("Packages")
    if not packages then return nil end

    local networking = packages:FindFirstChild("Networking")
    if not networking then return nil end

    local rfFolder = networking:FindFirstChild("RF")
    if not rfFolder then return nil end

    local eggWorld = rfFolder:FindFirstChild("EggWorld")
    if not eggWorld then return nil end

    return eggWorld:FindFirstChild("AskFieldEggSnapshot")
end

local function snapshotSummary()
    local rf = findSnapshotRF()
    if not rf then
        return "snapshotRF=nil"
    end

    local ok, result = pcall(function()
        return table.pack(rf:InvokeServer())
    end)

    if not ok then
        return "snapshotERR=" .. tostring(result)
    end

    local count = result.n or 0
    local carried = 0
    local slots = 0
    local targetUid = nil

    for i = 1, count do
        local value = result[i]

        if type(value) == "table" then
            local records = value.Records

            if type(records) == "table" then
                for _, rec in pairs(records) do
                    if type(rec) == "table" then
                        if rec.State == "Carried" then
                            carried += 1
                            targetUid = targetUid or rec.Uid
                        elseif rec.State == "Slot" then
                            slots += 1
                        end
                    end
                end
            end
        end
    end

    return string.format(
        "snapshot=OK values=%d carried=%d slots=%d carriedUid=%s",
        count,
        carried,
        slots,
        tostring(targetUid)
    )
end

local function disconnectAll()
    for _, c in ipairs(connections) do
        pcall(function()
            c:Disconnect()
        end)
    end

    table.clear(connections)

    if sampleConnection then
        pcall(function()
            sampleConnection:Disconnect()
        end)
        sampleConnection = nil
    end
end

local function startRecording()
    if recording then return end

    disconnectAll()

    table.clear(logs)

    recording = true
    startedAt = os.clock()
    lastSample = 0

    push("RECORD_START", "scanner iniciado")

    local c, root, hum = getCharacter()

    if root then
        push(
            "INITIAL_ROOT",
            "pos=" .. v3(root.Position)
            .. " | vel=" .. v3(root.AssemblyLinearVelocity)
            .. " | anchored=" .. tostring(root.Anchored)
        )
    end

    if hum then
        push(
            "INITIAL_HUM",
            "state=" .. tostring(hum:GetState())
            .. " | walkspeed=" .. tostring(hum.WalkSpeed)
            .. " | hp=" .. string.format("%.1f", hum.Health)
        )
    end

    --==============================================================
    -- RIGSYNC / REFRESH PASSIVO
    --==============================================================

    local refresh

    pcall(function()
        refresh =
            ReplicatedStorage
            :WaitForChild("Packages")
            :WaitForChild("Networking")
            :WaitForChild("RE")
            :WaitForChild("RigSync")
            :WaitForChild("Refresh")
    end)

    if refresh and refresh:IsA("RemoteEvent") then

        table.insert(connections, refresh.OnClientEvent:Connect(function(...)
            if not recording then return end

            local args = table.pack(...)

            local pieces = {}

            for i = 1, math.min(args.n or 0, 4) do
                local value = args[i]

                if type(value) == "table" then
                    local action =
                        value.Action
                        or value.action

                    local seq =
                        value.Sequence
                        or value.sequence

                    table.insert(
                        pieces,
                        string.format(
                            "arg%d=table Action=%s Seq=%s",
                            i,
                            tostring(action),
                            tostring(seq)
                        )
                    )
                else
                    table.insert(
                        pieces,
                        string.format(
                            "arg%d=%s",
                            i,
                            tostring(value)
                        )
                    )
                end
            end

            push(
                "RIGSYNC_REFRESH",
                table.concat(pieces, " ; ")
            )
        end))
    end

    --==============================================================
    -- CHARACTER OBSERVER
    --==============================================================

    local function attachCharacter(character)

        local root =
            character:WaitForChild(
                "HumanoidRootPart",
                5
            )

        local hum =
            character:FindFirstChildOfClass("Humanoid")

        if not root or not hum then
            push(
                "CHARACTER_ERROR",
                "Root/Humanoid nao encontrado"
            )
            return
        end

        push(
            "CHARACTER_ATTACH",
            "root=" .. root:GetFullName()
        )

        table.insert(
            connections,
            hum.StateChanged:Connect(function(oldState, newState)
                push(
                    "HUM_STATE",
                    tostring(oldState)
                    .. " -> "
                    .. tostring(newState)
                )
            end)
        )

        table.insert(
            connections,
            hum:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
                push(
                    "WALKSPEED_CHANGE",
                    tostring(hum.WalkSpeed)
                )
            end)
        )

        table.insert(
            connections,
            root:GetPropertyChangedSignal("Anchored"):Connect(function()
                push(
                    "ROOT_ANCHORED",
                    tostring(root.Anchored)
                )
            end)
        )

        table.insert(
            connections,
            character:GetAttributeChangedSignal("AreaId"):Connect(function()
                push(
                    "AREA_ATTR",
                    "AreaId=" .. tostring(character:GetAttribute("AreaId"))
                )
            end)
        )

        table.insert(
            connections,
            character:GetAttributeChangedSignal("Area"):Connect(function()
                push(
                    "AREA_ATTR",
                    "Area=" .. tostring(character:GetAttribute("Area"))
                )
            end)
        )
    end

    if LocalPlayer.Character then
        attachCharacter(LocalPlayer.Character)
    end

    table.insert(
        connections,
        LocalPlayer.CharacterAdded:Connect(function(character)
            if recording then
                attachCharacter(character)
            end
        end)
    )

    --==============================================================
    -- HEARTBEAT FORENSIC
    --==============================================================

    sampleConnection = RunService.Heartbeat:Connect(function()
        if not recording then return end

        local t = os.clock()

        if t - lastSample < sampleInterval then
            return
        end

        lastSample = t

        local character, root, hum = getCharacter()

        if not root or not hum then
            push(
                "FRAME",
                "character/root/humanoid ausente"
            )
            return
        end

        local velocity = root.AssemblyLinearVelocity
        local speed = velocity.Magnitude

        push(
            "FRAME",
            "pos=" .. v3(root.Position)
            .. " | vel=" .. v3(velocity)
            .. " | speed=" .. string.format("%.1f", speed)
            .. " | state=" .. tostring(hum:GetState())
            .. " | ws=" .. tostring(hum.WalkSpeed)
            .. " | anchored=" .. tostring(root.Anchored)
            .. " | area=" .. getArea()
        )
    end)
end

local function stopRecording()
    if not recording then return end

    push("RECORD_STOP", "scanner parado")

    recording = false

    if sampleConnection then
        pcall(function()
            sampleConnection:Disconnect()
        end)

        sampleConnection = nil
    end

    -- mantém conexões de eventos desconectadas
    for _, c in ipairs(connections) do
        pcall(function()
            c:Disconnect()
        end)
    end

    table.clear(connections)

    print("========== SCANNER PARADO ==========")
    print("Amostras/logs:", #logs)
end

local function copyLog()
    if #logs == 0 then
        warn("SCANNER: nenhum log gravado.")
        return
    end

    local text = table.concat(logs, "\n")

    if setclipboard then
        setclipboard(text)
        print("SCANNER: LOG COPIADO.")
    else
        warn("SCANNER: setclipboard nao disponivel.")
        print(text)
    end
end

--==============================================================
-- UI
--==============================================================

local gui = Instance.new("ScreenGui")
gui.Name = "EggForensicScanner"
gui.ResetOnSpawn = false
gui.Parent = game:GetService("CoreGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(270, 175)
frame.Position = UDim2.new(0, 15, 0.5, -87)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
frame.BorderSizePixel = 0
frame.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -20, 0, 35)
title.Position = UDim2.fromOffset(10, 5)
title.BackgroundTransparency = 1
title.Text = "FORENSIC SCANNER"
title.TextColor3 = Color3.new(1, 1, 1)
title.TextSize = 17
title.Font = Enum.Font.GothamBold
title.Parent = frame

local status = Instance.new("TextLabel")
status.Size = UDim2.new(1, -20, 0, 25)
status.Position = UDim2.fromOffset(10, 38)
status.BackgroundTransparency = 1
status.Text = "● PARADO"
status.TextColor3 = Color3.fromRGB(255, 190, 80)
status.TextSize = 13
status.Font = Enum.Font.Gotham
status.Parent = frame

local function button(text, y)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -20, 0, 32)
    b.Position = UDim2.fromOffset(10, y)
    b.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    b.BorderSizePixel = 0
    b.Text = text
    b.TextColor3 = Color3.new(1, 1, 1)
    b.TextSize = 13
    b.Font = Enum.Font.GothamBold
    b.Parent = frame

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 7)
    c.Parent = b

    return b
end

local startButton = button("▶ COMEÇAR A GRAVAR", 67)
local stopButton = button("■ PARAR", 105)
local copyButton = button("▣ COPIAR LOG", 143)

startButton.Activated:Connect(function()
    startRecording()
    status.Text = "● GRAVANDO"
    status.TextColor3 = Color3.fromRGB(100, 255, 120)
end)

stopButton.Activated:Connect(function()
    stopRecording()
    status.Text = "● PARADO"
    status.TextColor3 = Color3.fromRGB(255, 190, 80)
end)

copyButton.Activated:Connect(function()
    copyLog()
end)

print("FORENSIC SCANNER carregado.")
print("Aperte COMEÇAR antes de testar.")
