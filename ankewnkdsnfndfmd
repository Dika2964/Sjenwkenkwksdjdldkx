if game.PlaceId == 85247278545351 or game.PlaceId == 134459712190924 then return end
-- CTD: fungsi dikumpulkan sebelum UI. Cari penanda TAMBAH untuk menambah fitur.

-- ================================================================
-- 1. SERVICE
-- ================================================================

local RS = game:GetService("ReplicatedStorage")
local HS = game:GetService("HttpService")
local LP = game:GetService("Players").LocalPlayer

-- ================================================================
-- 2. VARIABEL DAN KONFIGURASI
-- ================================================================

local functions = {}
local env = getgenv()
local lifecycle = {Alive = true, Connections = {}}
local logLines, featureStates = {}, {}
local macroConnections = {}
local PlayerGui, fireServer, oldNamecall
local R = {}
local Library, SaveManager, LibraryNotify, Options, Toggles
local Window, Tabs, Macro, Voting, Helper
local Mutator, Ability, Cards, Summon, Webhook
local macroStatus, selectedName
local uiSync = false
local wave, waveStart, waveKnown

local Toggle = {
    PlayMacro = false,
    RecordMacro = false,
    AutoVoting = false,
    AutoJoin = false,
    AutoReply = false,
    AutoSkip = false,
    AutoSetSpeed = false,
    AutoSummon = false,
}

local M = {
    Directory = "GoaHub/Ctd/Macro",
    FileName = "Default.json",

    Mode = "Money",
    SellMode = "Time",
    TargetMode = "Time",

    TimeSource = "Time",
    WaveAttribute = "Wave",

    ResponseTimeout = 15,
    UpgradeResponseTimeout = 90, -- total waktu menunggu konfirmasi; bukan retry
    UpgradeSendDelay = 0.25, -- memberi jeda setelah aksi sebelumnya
    WaitTimeout = 600,
    PositionTolerance = 0.5,

    TimeAlsoWaitMoney = true,
    Debug = true,

    UnitIds = {},

    Actions = {},
    Pending = {},
    Towers = {},
    PlaybackTowers = {},

    Count = 0,
    Run = 0,

    Recording = false,
    Saving = false,
    Playing = false,
}

local Variables = {
    Difficulty = 'Easy',
    Speed = 1,
    AmountSummon = 1,
}

functions.Macro = M

-- ================================================================
-- 3. FUNGSI BANTUAN — dipanggil fitur, bukan lokasi toggle
-- ================================================================

functions.logMessage = function(level, message)
    local line = string.format("[%s] [GoaHub] %s", level, tostring(message))
    if level == "ERROR" then warn(line) else print(line) end
    logLines[#logLines + 1] = line
    if #logLines > 200 then table.remove(logLines, 1) end
    if type(writefile) == "function" and type(isfolder) == "function"
        and type(makefolder) == "function" then
        local ok, err = pcall(function()
            if not isfolder("GoaHub") then makefolder("GoaHub") end
            if not isfolder("GoaHub/Ctd") then makefolder("GoaHub/Ctd") end
            writefile("GoaHub/Ctd/Debug.log", table.concat(logLines, "\n"))
        end)
        if not ok then warn("[GoaHub] Gagal menulis Debug.log: " .. tostring(err)) end
    end
end

functions.traceback = function(message)
    if debug and type(debug.traceback) == "function" then
        return debug.traceback(tostring(message), 2)
    end
    return tostring(message)
end

functions.protected = function(label, callback, ...)
    local args = table.pack(...)
    local result = table.pack(xpcall(function()
        return callback(table.unpack(args, 1, args.n))
    end, functions.traceback))
    if not result[1] then functions.logMessage("ERROR", label .. ": " .. tostring(result[2])) end
    return table.unpack(result, 1, result.n)
end

functions.spawnTask = function(callback)
    task.spawn(function()
        if lifecycle.Alive then functions.protected("Background task", callback) end
    end)
end

functions.connect = function(signal, callback)
    local connection = signal:Connect(function(...)
        if lifecycle.Alive then functions.protected("Event callback", callback, ...) end
    end)
    table.insert(lifecycle.Connections, connection)
    return connection
end

functions.waitChild = function(parent, name)
    local value = parent:FindFirstChild(name) or parent:WaitForChild(name, 15)
    assert(value, "Objek tidak ditemukan (15 detik): " .. parent:GetFullName() .. "." .. name)
    return value
end

functions.prepareDirectory = function()
    assert(
        type(isfolder) == "function"
            and type(makefolder) == "function",
        "isfolder/makefolder tidak tersedia"
    )

    local current = ""

    for part in M.Directory:gmatch("[^/]+") do
        current = current == "" and part or current .. "/" .. part

        if not isfolder(current) then
            makefolder(current)
        end
    end
end

functions.filePath = function()
    assert(
        type(M.FileName) == "string"
            and M.FileName ~= ""
            and not M.FileName:find("[/\\]"),
        "FileName harus berupa nama file tanpa folder"
    )

    return M.Directory .. "/" .. M.FileName
end

functions.getMacroFiles = function()
    functions.prepareDirectory()
    assert(type(listfiles) == "function", "listfiles tidak tersedia")
    local names = {}
    for _, path in ipairs(listfiles(M.Directory)) do
        local basename = path:match("[^/\\]+$")
        local name = basename and basename:match("^(.*)%.json$")
        if name and name ~= "" then names[#names + 1] = name end
    end
    table.sort(names)
    return names
end

functions.loadLibrary = function(url, label)
    functions.logMessage("INFO", "Memuat " .. label .. "...")
    local done, ok, source = false, false, nil
    task.spawn(function()
        ok, source = pcall(function() return game:HttpGet(url) end)
        done = true
    end)
    local deadline = os.clock() + 30
    while not done do
        assert(os.clock() < deadline, "Timeout download 30 detik: " .. label)
        task.wait(0.05)
    end
    assert(ok, "Gagal download " .. label .. ": " .. tostring(source))
    local chunk, err = loadstring(source)
    assert(chunk, "Kode " .. label .. " tidak valid: " .. tostring(err))
    local success, result = pcall(chunk)
    assert(success and type(result) == "table", "Gagal menjalankan " .. label .. ": " .. tostring(result))
    return result
end
functions.status = function(text)
    if macroStatus then macroStatus:SetText("Status : " .. text) end
end

functions.notify = function(text)
    functions.logMessage("ERROR", tostring(text))
    pcall(function()
        LibraryNotify:Notify({Title = "GoaHub", Description = tostring(text), Time = 5})
    end)
end

functions.setToggle = function(name, value)
    local toggle = Toggles and Toggles[name]
    if toggle then
        uiSync = true
        local ok, err = pcall(function() toggle:SetValue(value) end)
        uiSync = false
        if not ok then functions.logMessage("ERROR", tostring(err)) end
    end
end

functions.busy = function()
    return M.Recording or M.Playing or M.Saving
end

functions.uiAction = function(callback)
    local ok, err = functions.protected("UI action", callback)
    if not ok then functions.notify(err) end
end

functions.debugLog = function(message)
    if M.Debug then
        print("[Macro] " .. message)
    end
end

functions.num = function(value)
    if type(value) == "string" then
        value = value:gsub(",", ""):gsub("%s+", "")
        value = tonumber(value)
    end

    if type(value) ~= "number"
        or value ~= value
        or math.abs(value) == math.huge
    then
        return nil
    end

    return value
end

functions.totalTime = function()
    local value = workspace:GetAttribute("TotalTime")

    if type(value) == "number" then
        return functions.num(value)
    end

    if type(value) ~= "string" then
        return nil
    end

    local minutes, seconds = value:match("^%s*(%d+):(%d%d)%s*$")

    if minutes then
        seconds = tonumber(seconds)

        if seconds >= 60 then
            return nil
        end

        return tonumber(minutes) * 60 + seconds
    end

    return functions.num(value)
end

functions.updateWave = function()
    local current = functions.num(workspace:GetAttribute(M.WaveAttribute))

    if current ~= wave then
        waveKnown = current ~= nil and (wave ~= nil or current == 0)
        wave, waveStart = current, os.clock()
    end
end

functions.timestamp = function()
    functions.updateWave()

    return {
        Time = functions.totalTime(),
        Wave = wave,
        WaveTime = waveKnown and os.clock() - waveStart or nil,
    }
end

functions.prunePending = function()
    local now = os.clock()

    for i = #M.Pending, 1, -1 do
        local pending = M.Pending[i]

        if pending.Done then
            table.remove(M.Pending, i)
        elseif now - pending.Created >= M.ResponseTimeout then
            table.remove(M.Pending, i)

            functions.debugLog(
                "Tidak direkam: "
                .. pending.Kind
                .. " tidak mendapat konfirmasi dalam batas waktu"
            )
        end
    end
end

functions.matches = function(pending, data)
    if type(data) ~= "table" then
        return false
    end

    if pending.Kind == "TowerAdd" then
        local position = data.HomePos or data.Pos

        return functions.num(data.OwnerUserId) == LP.UserId
            and type(data.Id) == "string"
            and data.Name == pending.Data.Name
            and data.UnitId == pending.Data.UnitId
            and typeof(position) == "Vector3"
            and (position - pending.Data.Pos).Magnitude
                <= M.PositionTolerance
    end

    if data.Id ~= pending.Data.Id then
        return false
    end

    if pending.Kind == "TowerUpgrade" then
        local level = functions.num(data.Level)
        return level ~= nil and level % 1 == 0
            and level > pending.BeforeLevel
            and (not pending.ExpectedLevel or level >= pending.ExpectedLevel)
    end

    return pending.Kind == "TowerTarget"
        and type(data.TargetMode) == "string"
end

functions.appendAction = function(action)
    action.Sequence = #M.Actions + 1
    table.insert(M.Actions, action)

    functions.debugLog(string.format(
        "REC #%d | %s | TowerIndex=%s | Level=%s | Cost=%s | Target=%s | Time=%s",
        action.Sequence,
        action.ActionType,
        tostring(action.TowerIndex),
        tostring(action.Level or "-"),
        tostring(action.Cost or "-"),
        tostring(action.TargetMode or "-"),
        tostring(action.Timestamp.Time or "-")
    ))
end

functions.recordResponse = function(kind, data)
    functions.prunePending()

    if kind == "TowerAdd" then
        if M.Towers[data.Id] then
            return
        end
    else
        local tower = M.Towers[data.Id]

        if not tower or tower.Sold then
            return
        end

        if kind == "TowerUpgrade" then
            local level = functions.num(data.Level)

            if not level or level <= tower.Level then
                return
            end
        elseif data.TargetMode == tower.TargetMode then
            return
        end
    end

    local pending

    -- Ambil permintaan cocok paling awal, bukan membuat error ambigu.
    for _, candidate in ipairs(M.Pending) do
        if not candidate.Done
            and candidate.Kind == kind
            and functions.matches(candidate, data)
        then
            pending = candidate
            break
        end
    end

    if not pending then
        return
    end

    local action = {
        ActionType = kind,
        Timestamp = pending.Timestamp,
        ServerIndex = data.Index,
        Confirmed = true,
    }

    if kind == "TowerAdd" or kind == "TowerUpgrade" then
        action.Level = functions.num(data.Level)
        action.Cost = functions.num(data.Stats and data.Stats.Price)

        assert(
            action.Level
                and action.Level >= 1
                and action.Level % 1 == 0
                and action.Cost
                and action.Cost >= 0,
            "Balasan " .. kind .. ": Level/Stats.Price tidak valid"
        )
    end

    if kind == "TowerAdd" then
        action.TowerIndex = pending.TowerIndex
        action.Name = pending.Data.Name
        action.UnitId = data.UnitId
        action.RecordedId = data.Id
        action.CFrame = pending.CFrame
        action.Pos = pending.Pos

        M.Towers[data.Id] = {
            TowerIndex = action.TowerIndex,
            Level = action.Level,
            TargetMode = data.TargetMode,
        }
    else
        local tower = M.Towers[data.Id]

        action.TowerIndex = tower.TowerIndex
        action.RecordedId = data.Id

        if kind == "TowerUpgrade" then
            action.FromLevel = tower.Level
            tower.Level = action.Level
        else
            action.TargetMode = data.TargetMode
            tower.TargetMode = data.TargetMode
        end
    end

    pending.Done = true
    functions.appendAction(action)
end

functions.capture = function(kind, data)
    assert(type(data) == "table", "Payload tidak valid")

    functions.prunePending()

    local pending = {
        Kind = kind,
        Created = os.clock(),
        Timestamp = functions.timestamp(),
        Data = {},
    }

    if kind == "TowerAdd" then
        assert(
            typeof(data.CFrame) == "CFrame"
                and typeof(data.Pos) == "Vector3",
            "CFrame/Pos tidak valid"
        )

        assert(
            type(data.Name) == "string"
                and type(data.UnitId) == "string",
            "Name/UnitId tidak valid"
        )

        M.Count = M.Count + 1

        pending.TowerIndex = M.Count
        pending.CFrame = { data.CFrame:GetComponents() }
        pending.Pos = { data.Pos.X, data.Pos.Y, data.Pos.Z }

        pending.Data = {
            Name = data.Name,
            UnitId = data.UnitId,
            Pos = data.Pos,
        }
    else
        local tower = M.Towers[data.Id]

        if not tower or tower.Sold then
            functions.debugLog("Permintaan diabaikan: tower belum tercatat/sudah dijual")
            return
        end

        pending.Data.Id = data.Id
        pending.BeforeLevel = tower.Level

        if kind == "TowerSell" then
            functions.appendAction({
                ActionType = kind,
                TowerIndex = tower.TowerIndex,
                RecordedId = data.Id,
                Timestamp = pending.Timestamp,
                Cost = 0,
                RequestOnly = true,
            })

            tower.Sold = true
            return
        end
    end

    table.insert(M.Pending, pending)
end

functions.waitFor = function(predicate, run, timeout, description)
    local deadline = os.clock() + timeout

    while true do
        assert(
            Toggle.PlayMacro and M.Run == run,
            "Playback dihentikan"
        )

        if predicate() then
            return
        end

        assert(os.clock() < deadline, "Timeout: " .. description)

        task.wait(0.03)
    end
end

-- Respons server memperbarui level walaupun tidak ada request yang menunggu.
-- Level lama/duplikat tidak boleh menurunkan level yang sudah terkonfirmasi.
functions.syncPlaybackTower = function(kind, data)
    if not M.Playing or type(data.Id) ~= "string" then return end
    for _, tower in pairs(M.PlaybackTowers) do
        if tower.Id == data.Id and not tower.Sold then
            if kind == "TowerUpgrade" then
                local level = functions.num(data.Level)
                if level and level >= 1 and level % 1 == 0
                    and level > tower.Level then
                    tower.Level = level
                    tower.ServerIndex = data.Index or tower.ServerIndex
                    functions.debugLog("SYNC Upgrade | Id=" .. data.Id .. " | Level=" .. level)
                end
            elseif kind == "TowerTarget" and type(data.TargetMode) == "string" then
                tower.TargetMode = data.TargetMode
                tower.ServerIndex = data.Index or tower.ServerIndex
            end
            return
        end
    end
end

functions.request = function(kind, data, tower, run, action)
    assert(not M.Waiting, "Masih ada request macro yang belum selesai")
    local pending = {
        Kind = kind,
        Data = data,
        BeforeLevel = tower and tower.Level or 0,
        BeforeTarget = tower and tower.TargetMode,
        ExpectedLevel = action and action.Level,
    }
    local started = os.clock()
    local timeout = kind == "TowerUpgrade"
        and math.max(M.ResponseTimeout, M.UpgradeResponseTimeout)
        or M.ResponseTimeout
    local description = string.format(
        "%s | Aksi=%s | TowerIndex=%s | Id=%s | Level=%s->%s | Cost=%s | Money=%s",
        kind, tostring(action and action.Sequence or "-"),
        tostring(action and action.TowerIndex or "-"), tostring(data.Id or "-"),
        tostring(pending.BeforeLevel), tostring(pending.ExpectedLevel or "-"),
        tostring(action and action.Cost or "-"), tostring(functions.GetCash())
    )
    M.Waiting = pending
    functions.debugLog("SEND " .. description)
    local ok, result = xpcall(function()
        assert(Toggle.PlayMacro and M.Run == run, "Playback dihentikan")
        local remote = assert(R[kind], "Remote tidak tersedia: " .. kind)
        fireServer(remote, data) -- hanya sekali; timeout tidak membuktikan gagal
        local warned = false
        functions.waitFor(function()
            if pending.Result then return true end
            if not warned and kind == "TowerUpgrade"
                and os.clock() - started >= M.ResponseTimeout then
                warned = true
                functions.logMessage("WARN", "Respons terlambat; masih menunggu tanpa mengirim ulang. "
                    .. description)
                functions.status("Menunggu konfirmasi upgrade")
            end
            return false
        end, run, timeout,
            description .. " | Status server belum pasti; tidak dikirim ulang otomatis")
        return pending.Result
    end, functions.traceback)
    -- Bersihkan juga jika FireServer error, timeout, atau pengguna menekan Stop.
    if M.Waiting == pending then M.Waiting = nil end
    if not ok then error(result, 0) end
    functions.status("Playing")
    functions.debugLog("ACK " .. description .. " | LevelBalasan="
        .. tostring(result.Level or "-") .. " | Detik="
        .. string.format("%.2f", os.clock() - started))
    return result
end

functions.playUpgrade = function(action, tower, run, waitMoney)
    assert(not tower.Sold, "Tower sudah dijual")
    -- Jeda dapat dibatalkan dan memberi kesempatan respons sebelumnya diproses.
    local readyAt = os.clock() + M.UpgradeSendDelay
    functions.waitFor(function()
        return os.clock() >= readyAt
    end, run, M.WaitTimeout, "Jeda upgrade")

    if waitMoney then
        functions.waitFor(function()
            -- Jika server sudah mengonfirmasi level tujuan, tidak perlu uang lagi.
            return tower.Level >= action.Level or functions.GetCash() >= action.Cost
        end, run, M.WaitTimeout, "Money upgrade | TowerIndex=" .. action.TowerIndex
            .. " | Cost=" .. tostring(action.Cost))
    end

    if tower.Level >= action.Level then
        functions.debugLog("SKIP Upgrade sudah terkonfirmasi | TowerIndex="
            .. action.TowerIndex .. " | Level=" .. tower.Level
            .. " | Target=" .. action.Level)
        return
    end

    assert(tower.Level == action.FromLevel,
        "Level sebelum upgrade berbeda | TowerIndex=" .. action.TowerIndex
        .. " | Id=" .. tower.Id .. " | ServerTerakhir=" .. tower.Level
        .. " | Rekaman=" .. action.FromLevel)

    local data = functions.request("TowerUpgrade", {Id = tower.Id}, tower, run, action)
    local level = functions.num(data.Level)
    assert(level and level >= action.Level, "Konfirmasi level upgrade tidak valid")
    tower.Level = math.max(tower.Level, level)
    tower.ServerIndex = data.Index or tower.ServerIndex
    if tower.Level > action.Level then
        functions.logMessage("WARN", "Level server melebihi rekaman | TowerIndex="
            .. action.TowerIndex .. " | Level=" .. tower.Level .. " | Target=" .. action.Level)
    end
end

functions.actionMode = function(kind, defaultMode)
    if kind == "TowerSell" then
        return M.SellMode
    elseif kind == "TowerTarget" then
        return M.TargetMode
    end

    return defaultMode
end

functions.waitTime = function(stamp, source, run)
    assert(type(stamp) == "table", "Timestamp tidak tersedia")

    functions.waitFor(function()
        if source == "Time" then
            local current = functions.totalTime()

            assert(
                current and functions.num(stamp.Time),
                "TotalTime tidak tersedia/tidak valid"
            )

            return current >= stamp.Time
        end

        functions.updateWave()

        assert(
            wave and functions.num(stamp.Wave) and functions.num(stamp.WaveTime),
            "Timestamp wave tidak tersedia"
        )

        assert(wave <= stamp.Wave, "Wave aksi sudah terlewat")

        if wave ~= stamp.Wave then
            return false
        end

        assert(waveKnown, "Awal wave tidak teramati")

        return os.clock() - waveStart >= stamp.WaveTime
    end, run, M.WaitTimeout, "waktu")
end

functions.validate = function(document, mode, source)
    assert(
        type(document) == "table" and document.Version == 2,
        "Format file berbeda; rekam ulang menggunakan script ini"
    )

    assert(document.PlaceId == game.PlaceId, "PlaceId berbeda")

    assert(
        type(document.Actions) == "table"
            and #document.Actions > 0,
        "Rekaman kosong"
    )

    local levels = {}

    for sequence, action in ipairs(document.Actions) do
        assert(
            type(action) == "table"
                and action.Sequence == sequence
                and R[action.ActionType],
            "Aksi tidak valid"
        )

        local index = action.TowerIndex
        local kind = action.ActionType

        assert(
            type(index) == "number"
                and index >= 1
                and index % 1 == 0,
            "TowerIndex tidak valid"
        )

        if kind == "TowerAdd" then
            assert(levels[index] == nil, "TowerIndex duplikat")

            assert(
                type(action.Name) == "string"
                    and type(action.UnitId) == "string",
                "Name/UnitId tidak valid"
            )

            for _, spec in ipairs({
                { action.CFrame, 12 },
                { action.Pos, 3 },
            }) do
                assert(
                    type(spec[1]) == "table"
                        and #spec[1] == spec[2],
                    "Posisi tidak valid"
                )

                for _, value in ipairs(spec[1]) do
                    assert(
                        type(value) == "number" and functions.num(value),
                        "Komponen posisi tidak valid"
                    )
                end
            end
        else
            assert(levels[index], "Referensi tower tidak valid")
        end

        if kind == "TowerAdd" or kind == "TowerUpgrade" then
            assert(
                type(action.Level) == "number"
                    and action.Level >= 1
                    and action.Level % 1 == 0,
                "Level tidak valid"
            )

            assert(
                type(action.Cost) == "number"
                    and functions.num(action.Cost)
                    and action.Cost >= 0,
                "Cost tidak valid"
            )

            if kind == "TowerUpgrade" then
                assert(
                    action.FromLevel == levels[index]
                        and action.Level == levels[index] + 1,
                    "Ada level upgrade yang terlewat; rekam ulang"
                )
            end

            levels[index] = action.Level
        elseif kind == "TowerTarget" then
            assert(
                type(action.TargetMode) == "string",
                "TargetMode tidak valid"
            )
        end

        if kind == "TowerSell" then
            assert(action.RequestOnly == true, "Sell tidak valid")
            levels[index] = false
        else
            assert(action.Confirmed == true, "Aksi belum dikonfirmasi")
        end

        if functions.actionMode(kind, mode) == "Time" then
            local stamp = action.Timestamp

            assert(type(stamp) == "table", "Timestamp tidak tersedia")

            if source == "Time" then
                assert(functions.num(stamp.Time), "TotalTime tidak tercatat")
            else
                assert(
                    functions.num(stamp.Wave) and functions.num(stamp.WaveTime),
                    "Waktu wave tidak tercatat"
                )
            end
        end
    end
end

-- Cari ulang remote setiap fitur diaktifkan; remote tidak wajib ada saat startup.
functions.findRemote = function(name, className)
    local node = RS
    for _, part in ipairs({"ModuleLoader", "Shared", "Network", className}) do
        node = node:FindFirstChild(part)
        if not node then return nil end
    end
    local remote = node:FindFirstChild(name)
    if remote and remote:IsA(className) then return remote end
    return nil
end

functions.prepareMacro = function(recording)
    local missing = {}
    for _, name in ipairs({"TowerAdd", "TowerUpgrade", "TowerSell", "TowerTarget"}) do
        R[name] = functions.findRemote(name, "RemoteEvent")
        if not R[name] then missing[#missing + 1] = name end
    end
    if #missing > 0 then
        return false, "Macro belum tersedia di tempat ini. Remote tidak ada: "
            .. table.concat(missing, ", ")
    end
    fireServer = R.TowerAdd.FireServer
    functions.BindMacroResponses()
    if recording then functions.InstallMacroHook() end
    return true
end

functions.stopFeature = function(name)
    Toggle[name] = false
    local previous = featureStates[name]
    if previous then
        for _, connection in ipairs(previous.Connections) do connection:Disconnect() end
        featureStates[name] = nil
    end
end

functions.visibleFeature = function(name, enabled, frameName, remoteName, send)
    functions.stopFeature(name)
    if not enabled then return true end
    local remote = functions.findRemote(remoteName, "RemoteEvent")
    if not remote or not remote:IsA("RemoteEvent") then
        return false, "Remote " .. remoteName .. " tidak ditemukan/tidak valid"
    end
    Toggle[name] = true
    local state = {Connections = {}, Bound = setmetatable({}, {__mode = "k"})}
    featureStates[name] = state
    local function active()
        return lifecycle.Alive and Toggle[name] and featureStates[name] == state
    end
    local function listen(signal, callback)
        local c = functions.connect(signal, callback)
        state.Connections[#state.Connections + 1] = c
    end
    local function bind(object)
        if not active() or state.Bound[object] or object.Name ~= frameName
            or not object:IsA("GuiObject") then return end
        local parent = object.Parent
        if not parent or parent.Name ~= "GameUI" or parent.Parent ~= PlayerGui then return end
        state.Bound[object] = true
        local lastVisible = false
        local function check()
            if not active() then return end
            local current = object.Visible
            local opened = current and not lastVisible
            lastVisible = current
            if not opened then return end
            functions.spawnTask(function()
                if not active() or not object.Visible then return end
                local ok, err = functions.protected(name, send, remote, active)
                if not ok and active() then
                    functions.stopFeature(name)
                    functions.setToggle(name, false)
                    functions.notify(err)
                end
            end)
        end
        listen(object:GetPropertyChangedSignal("Visible"), check)
        check()
    end
    listen(PlayerGui.DescendantAdded, bind)
    for _, object in ipairs(PlayerGui:GetDescendants()) do bind(object) end
    functions.logMessage("INFO", name .. " aktif; menunggu " .. frameName .. ".Visible")
    return true
end


-- ================================================================
-- 4. FUNGSI FITUR — tambahkan functions.Nama di bagian ini
-- ================================================================

functions.GetCash = function()
    local leaderstats = LP:FindFirstChild("leaderstats")
    local money = leaderstats and leaderstats:FindFirstChild("Money")
    local value = money and functions.num(money.Value)

    assert(value and value >= 0, "Money tidak ditemukan/tidak valid")

    return value
end

functions.StopAndSave = function()
    if M.Saving
        or (not M.Recording and #M.Actions == 0)
    then
        return false
    end

    M.Recording = false
    Toggle.RecordMacro = false
    M.Saving = true
    functions.status("Menyimpan...")
    functions.setToggle("RecordMacro", false)

    local success, err = functions.protected("StopAndSave", function()
        -- Tunggu permintaan yang masih berjalan sampai dikonfirmasi/timeout.
        while true do
            functions.prunePending()

            if #M.Pending == 0 then
                break
            end

            task.wait(0.03)
        end

        assert(not M.Error, M.Error or "Recording bermasalah")
        assert(#M.Actions > 0, "Tidak ada aksi yang berhasil direkam")
        assert(type(writefile) == "function", "writefile tidak tersedia")

        functions.prepareDirectory()

        writefile(functions.filePath(), HS:JSONEncode({
            Version = 2,
            PlaceId = game.PlaceId,
            Actions = M.Actions,
        }))
    end)

    M.Saving = false
    M.SaveFailed = not success

    if not success then
        functions.status("Gagal menyimpan; lihat console")
        functions.logMessage("ERROR", "Gagal menyimpan: " .. tostring(err))
        return false, err
    end

    functions.status("Tersimpan: " .. M.FileName)
    functions.debugLog("SAVE | " .. #M.Actions .. " aksi | " .. functions.filePath())

    return true
end

functions.StopPlayback = function()
    Toggle.PlayMacro = false
    M.Run = M.Run + 1
end

functions.StartRecording = function(enabled)
    if not enabled then
        return functions.StopAndSave()
    end

    if M.Playing or M.Saving then
        return false
    end

    if M.Recording then
        return true
    end

    local ready, reason = functions.prepareMacro(true)
    if not ready then return false, reason end

    local success, err = pcall(functions.prepareDirectory)

    if not success then
        functions.logMessage("ERROR", "Gagal membuat folder: " .. tostring(err))
        return false, err
    end

    M.Actions = {}
    M.Pending = {}
    M.Towers = {}
    M.Count = 0
    M.Error = nil
    M.SaveFailed = false

    M.Recording = true
    Toggle.RecordMacro = true

    functions.debugLog("Recording dimulai")
    functions.status("Recording")
    return true
end

functions.Play = function(bool)
    if bool == false then functions.StopPlayback(); return true end
    local mode = type(bool) == "string" and bool or M.Mode
    if M.Playing or M.Recording or M.Saving then
        return false
    end

    local source = M.TimeSource

    for _, selected in ipairs({ mode, M.SellMode, M.TargetMode }) do
        if selected ~= "Money" and selected ~= "Time" then
            return false, "Mode harus Money atau Time"
        end
    end

    if source ~= "Time" and source ~= "Wave" then
        return false, "TimeSource harus Time atau Wave"
    end
    
    local ready, reason = functions.prepareMacro(false)
    if not ready then return false, reason end

    functions.status("Playing")
    M.Playing = true
    Toggle.PlayMacro = true
    M.Run = M.Run + 1

    local run = M.Run
    local towers = {}

    M.PlaybackTowers = towers

    local success, err = functions.protected("Playback", function()
        functions.waitFor(function()
            local gameUI = PlayerGui:FindFirstChild("GameUI")
            local voting = gameUI and gameUI:FindFirstChild("GamemodeVoting")
            return not voting or not voting.Visible
        end, run, M.WaitTimeout, "GamemodeVoting masih terlihat")
        assert(
            type(isfile) == "function"
                and type(readfile) == "function",
            "API pembacaan file tidak tersedia"
        )

        assert(isfile(functions.filePath()), "File macro tidak ditemukan")

        local document = HS:JSONDecode(readfile(functions.filePath()))

        functions.validate(document, mode, source)

        for _, action in ipairs(document.Actions) do
            assert(
                Toggle.PlayMacro and M.Run == run,
                "Playback dihentikan"
            )

            local kind = action.ActionType
            local index = action.TowerIndex
            local selected = functions.actionMode(kind, mode)

            if selected == "Time" then
                functions.waitTime(action.Timestamp, source, run)
            end

            if kind ~= "TowerUpgrade" and action.Cost
                and (selected == "Money" or M.TimeAlsoWaitMoney)
            then
                functions.waitFor(function()
                    return functions.GetCash() >= action.Cost
                end, run, M.WaitTimeout, "Money")
            end

            if kind == "TowerAdd" then
                local data = functions.request(kind, {
                    Name = action.Name,

                    UnitId = M.UnitIds[action.UnitId]
                        or M.UnitIds[action.Name]
                        or action.UnitId,

                    CFrame = CFrame.new(table.unpack(action.CFrame)),
                    Pos = Vector3.new(table.unpack(action.Pos)),
                }, nil, run)

                assert(
                    functions.num(data.Level) == action.Level,
                    "Level awal berbeda"
                )

                for _, existing in pairs(towers) do
                    assert(existing.Id ~= data.Id, "ID tower duplikat")
                end

                towers[index] = {
                    Id = data.Id,
                    ServerIndex = data.Index,
                    UnitId = data.UnitId,
                    Level = functions.num(data.Level),
                    TargetMode = data.TargetMode,
                }
            else
                local tower = assert(
                    towers[index],
                    "Tower index " .. index .. " tidak tersedia"
                )

                assert(not tower.Sold, "Tower sudah dijual")

                if kind == "TowerSell" then
                    fireServer(R[kind], {
                        Id = tower.Id,
                    })

                    tower.Sold = true

                    functions.debugLog(
                        "Sell dikirim; belum ada konfirmasi server"
                    )

                elseif kind == "TowerUpgrade" then
                    functions.playUpgrade(action, tower, run,
                        selected == "Money" or M.TimeAlsoWaitMoney)

                elseif tower.TargetMode ~= action.TargetMode then
                    local data = functions.request(kind, {
                        Id = tower.Id,
                    }, tower, run)

                    assert(
                        data.TargetMode == action.TargetMode,
                        "TargetMode hasil tidak sesuai"
                    )

                    tower.TargetMode = data.TargetMode
                    tower.ServerIndex = data.Index
                end
            end

            functions.debugLog(
                "PLAY #"
                .. action.Sequence
                .. " | "
                .. kind
                .. " | TowerIndex="
                .. index
            )

            task.wait(0.05)
        end
    end)

    M.Playing = false
    Toggle.PlayMacro = false
    M.Waiting = nil
    functions.setToggle("MacroPlay", false)
    functions.status(success and "Playback selesai" or "Playback berhenti; lihat console")

    if not success then
        functions.logMessage("ERROR", "Playback berhenti: " .. tostring(err))
        return false, err
    end

    functions.debugLog("Playback selesai")

    return true
end

functions.AutoVoting = function(enabled)
    return functions.visibleFeature("AutoVoting", enabled, "GamemodeVoting", "ModeVote", function(remote, active)
        if active() then
            remote:FireServer(Variables.Difficulty)
            functions.logMessage("INFO", "ModeVote dikirim: " .. tostring(Variables.Difficulty))
        end
    end)
end

functions.AutoReply = function(enabled)
    return functions.visibleFeature("AutoReply", enabled, "FinalUI", "PlayAgain", function(remote, active)
        -- Jangan memulai ulang pertandingan sebelum rekaman selesai disimpan.
        if M.Recording then
            local saved, err = functions.StopAndSave()
            assert(saved, err or "Rekaman belum tersimpan; Auto Reply dibatalkan")
        end
        local deadline = os.clock() + M.ResponseTimeout + 5
        while M.Saving do
            if not active() then return end
            assert(os.clock() < deadline, "Timeout menunggu penyimpanan sebelum Auto Reply")
            task.wait(0.05)
        end
        assert(not M.SaveFailed, "Penyimpanan gagal; Auto Reply dibatalkan")
        if active() then
            remote:FireServer()
            functions.logMessage("INFO", "PlayAgain dikirim; keberhasilan server belum diverifikasi")
        end
    end)
end

functions.AutoSkip = function(enabled)
    functions.stopFeature("AutoSkip")
    if not enabled then return true end
    local remote = functions.findRemote("WaveSkip", "RemoteEvent")
    if not remote or not remote:IsA("RemoteEvent") then return false, "WaveSkip tidak tersedia" end
    Toggle.AutoSkip = true
    local state = {Connections = {}, Bound = {}}
    featureStates.AutoSkip = state
    local sent = false
    local function check()
        if not lifecycle.Alive or not Toggle.AutoSkip or featureStates.AutoSkip ~= state then return end
        if workspace:GetAttribute("CanSkip") ~= true then sent = false; return end
        local gui = PlayerGui:FindFirstChild("GameUI")
        local voting = gui and gui:FindFirstChild("GamemodeVoting")
        if voting and voting.Visible then return end
        if sent then return end
        sent = true
        local ok = functions.protected("WaveSkip", function() remote:FireServer() end)
        if not ok then
            functions.stopFeature("AutoSkip")
            functions.setToggle("AutoSkip", false)
        end
    end
    local function listen(signal, callback)
        state.Connections[#state.Connections + 1] = functions.connect(signal, callback)
    end
    local function bind(object)
        if object.Name == "GamemodeVoting" and object:IsA("GuiObject")
            and object.Parent and object.Parent.Name == "GameUI"
            and object.Parent.Parent == PlayerGui and not state.Bound[object] then
            state.Bound[object] = true
            listen(object:GetPropertyChangedSignal("Visible"), check)
            check()
        end
    end
    listen(workspace:GetAttributeChangedSignal("CanSkip"), check)
    listen(PlayerGui.DescendantAdded, bind)
    for _, object in ipairs(PlayerGui:GetDescendants()) do bind(object) end
    check()
    return true
end

functions.AutoSetSpeed = function(enabled)
    functions.stopFeature("AutoSetSpeed")
    if not enabled then return true end
    local remote = functions.findRemote("ChangeSpeed", "RemoteFunction")
    if not remote or not remote:IsA("RemoteFunction") then return false, "ChangeSpeed tidak tersedia" end
    Toggle.AutoSetSpeed = true
    local state = {Connections = {}, Busy = false}
    featureStates.AutoSetSpeed = state
    local function check()
        if not lifecycle.Alive or not Toggle.AutoSetSpeed
            or featureStates.AutoSetSpeed ~= state or state.Busy then return end
        local current = tonumber(workspace:GetAttribute("GameSpeed"))
        if not current or current == Variables.Speed then return end
        state.Busy = true
        functions.spawnTask(function()
            if not Toggle.AutoSetSpeed or featureStates.AutoSetSpeed ~= state then return end
            local ok = functions.protected("ChangeSpeed", function()
                remote:InvokeServer(Variables.Speed)
            end)
            state.Busy = false
            if not ok and featureStates.AutoSetSpeed == state then
                functions.stopFeature("AutoSetSpeed")
                functions.setToggle("AutoSetSpeed", false)
            end
        end)
    end
    state.Connections[1] = functions.connect(workspace:GetAttributeChangedSignal("GameSpeed"), check)
    check()
    return true
end

functions.AutoSummon = function(enabled)
    functions.stopFeature("AutoSummon")
    if not enabled then return true end
    local remote = functions.findRemote("SummonTowers", "RemoteFunction")
    if not remote or not remote:IsA("RemoteFunction") then return false, "SummonTowers tidak tersedia" end
    Toggle.AutoSummon = true
    local state = {Connections = {}, Busy = false}
    featureStates.AutoSummon = state
    functions.spawnTask(function()
        local function active()
            return lifecycle.Alive and Toggle.AutoSummon
                and featureStates.AutoSummon == state
        end
        while active() do
            local ok, err = functions.protected("AutoSummon", function()
                remote:InvokeServer("Basic Crate", Variables.AmountSummon)
            end)
            if not ok then
                if active() then
                    functions.stopFeature("AutoSummon")
                    functions.setToggle("AutoSummon", false)
                    functions.notify(err)
                end
                return
            end
            if active() then task.wait(0.3) end
        end
    end)
    return true
end

-- TAMBAH FUNGSI FITUR BARU DI ATAS BARIS INI.
-- Contoh (hapus tanda -- untuk mengaktifkan):
-- functions.FiturBaru = function(bool)
--     Toggle.FiturBaru = bool
--     if not bool then return end
--     print("Fitur aktif")
-- end

-- ================================================================
-- 5. UI DAN TOGGLE — tambahkan toggle di dalam BuildUI
-- ================================================================

-- SaveManager asli memuat tiap item memakai task.spawn.
-- Pengganti Load ini memulihkan pilihan dahulu, baru menjalankan toggle.
functions.ConfigureSaveManager = function()
    SaveManager.Load = function(self, name)
        if self.CTDLoading then return false, "Konfigurasi sedang dimuat" end
        if functions.busy() then return false, "Hentikan macro sebelum memuat config" end
        if type(name) ~= "string" or name == "" then return false, "Pilih config terlebih dahulu" end
        self.CTDLoading = true
        local previousAutoSave = self.AutoSave
        self.AutoSave = false
        local desired = {}
        local ok, err = functions.protected("Load config " .. name, function()
            self:CheckFolderTree()
            local directory = self.Folder .. "/settings/"
            if self:CheckSubFolder(true) then directory = directory .. self.SubFolder .. "/" end
            local path = directory .. name .. ".json"
            assert(isfile(path), "Config tidak ditemukan: " .. path)
            local data = HS:JSONDecode(readfile(path))
            assert(type(data) == "table" and type(data.objects) == "table", "Format config tidak valid")
            -- Hentikan fitur lama sebelum mengganti pilihan.
            for _, id in ipairs({"AutoSummon", "AutoVoting", "AutoReply", "AutoSkip", "AutoSetSpeed"}) do
                functions.stopFeature(id)
            end
            uiSync = true
            for _, option in ipairs(data.objects) do
                if type(option) == "table" and not self.Ignore[option.idx] then
                    -- Migrasi Speed yang dahulu salah disimpan sebagai Difficulty.
                    local id = option.idx
                    if id == "Difficulty" and tonumber(option.value) then
                        id = "Speed"
                        functions.logMessage("INFO", "Migrasi Difficulty numerik menjadi Speed")
                    end
                    local parser = self.Parser[option.type]
                    if parser and option.type ~= "Toggle" then
                        parser.Load(id, option)
                    elseif parser and type(option.value) == "boolean" then
                        desired[id] = option.value
                    end
                end
            end
            -- Beberapa callback diabaikan selama uiSync, maka salin nilai finalnya.
            local selected = Options.ListMacro.Value
            selectedName = type(selected) == "string" and selected ~= "" and selected or nil
            M.FileName = selectedName and selectedName .. ".json" or "Default.json"
            local mode = Options.ModeMacro.Value
            if mode == "Money" or mode == "Time" then M.Mode = mode end
            local difficulty = Options.Difficulty.Value
            if difficulty == "Easy" or difficulty == "Normal" or difficulty == "Hard" or difficulty == "Insane" then
                Variables.Difficulty = difficulty
            end
            local speed = tonumber(Options.Speed.Value)
            if speed == 1 or speed == 1.5 or speed == 2 then Variables.Speed = speed end
            if data.custom then
                for key, handler in pairs(self.CustomData or {}) do
                    if data.custom[key] ~= nil then handler.Load(data.custom[key]) end
                end
            end
            -- Config yang tidak memuat sebuah fitur mempertahankan pilihan UI-nya.
            for _, id in ipairs({"AutoSummon", "AutoVoting", "AutoReply", "AutoSkip", "AutoSetSpeed", "RecordMacro", "MacroPlay"}) do
                if desired[id] == nil then desired[id] = Toggles[id].Value == true end
            end
            if desired.RecordMacro and desired.MacroPlay then
                desired.RecordMacro = false
                functions.logMessage("INFO", "Config mengaktifkan record dan play; playback diprioritaskan")
            end
            -- Turunkan dahulu agar SetValue(true) memicu callback walau nilai lama true.
            for id, value in pairs(desired) do
                local toggle = Toggles[id]
                if toggle then toggle:SetValue(false) end
            end
        end)
        uiSync = false
        if ok then
            for _, id in ipairs({"AutoSummon", "AutoVoting", "AutoSkip", "AutoSetSpeed", "RecordMacro", "MacroPlay", "AutoReply"}) do
                if desired[id] then
                    functions.protected("Aktifkan " .. id, function() Toggles[id]:SetValue(true) end)
                end
                desired[id] = nil
            end
            for id, value in pairs(desired) do
                if Toggles[id] then
                    functions.protected("Pulihkan " .. id, function() Toggles[id]:SetValue(value) end)
                end
            end
            functions.logMessage("INFO", "Config dipulihkan: " .. name)
        end
        self.AutoSave = previousAutoSave
        self.CTDLoading = false
        return ok, err
    end
end

functions.BuildUI = function()
local repo = "https://raw.githubusercontent.com/Dika2964/Library/refs/heads/main/Obsidian/"
    LibraryNotify = functions.loadLibrary(repo .. "Library.lua%20(4).txt", "LibraryNotify")
    Library = functions.loadLibrary("https://raw.githubusercontent.com/Dika2964/Main/refs/heads/main/LibraryV3.lua.txt", "Library UI")
    SaveManager = functions.loadLibrary("https://raw.githubusercontent.com/Dika2964/Main/refs/heads/main/SaveManagerV3.lua.txt", "SaveManager")
    Options = Library.Options
    Toggles = Library.Toggles
    Window = Library:CreateWindow({
        Title = "Cameraman Tower Defense",
        Icon = "121794676391296",
        Footer = "v0.1",
        Position = UDim2.fromOffset(6, 6),
        Size = UDim2.fromOffset(720, 600),
        Center = true,
        AutoShow = true,
        Resizable = true,
        SidebarCompacted = true,
        NotifySide = "Right",
        ShowCustomCursor = true,
        TabWidth = 10,
    })

    Tabs = {
        Main = Window:AddTab("", "house"),
        Settings = Window:AddTab("", "settings"),
    }

    Macro = Tabs.Main:AddLeftGroupbox("Macro")
    Voting = Tabs.Main:AddRightGroupbox("Voting", "map")
    Mutator = Tabs.Main:AddRightGroupbox("Mutator", "dna")
    Ability = Tabs.Main:AddRightGroupbox("Ability", "zap")
    Cards = Tabs.Main:AddRightGroupbox("Rogue Card", "credit-card")
    Helper = Tabs.Main:AddLeftGroupbox("Helper", "hand-helping")
    Summon = Tabs.Main:AddRightGroupbox("Summon", "wrench")
    Webhook = Tabs.Main:AddRightGroupbox("Webhook", "webhook")

    macroStatus = Macro:AddLabel("Status : Siap")
    Macro:AddDropdown('ListMacro', {
        Text = 'List Macro', Values = functions.getMacroFiles(),
        Default = 0, Multi = false, MaxVisibleDropdownItems = 5,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            if functions.busy() then
                uiSync = true
                Options.ListMacro:SetValue(selectedName)
                uiSync = false
                functions.notify("Hentikan macro sebelum mengganti file")
                return
            end
            selectedName = type(value) == "string" and value ~= "" and value or nil
            M.FileName = selectedName and selectedName .. ".json" or "Default.json"
        end,
    })

    Macro:AddInput('MacroInput', {
        Text = 'Name Macro', Default = '', Placeholder = 'Example : Default',
        Numeric = false, ClearTextOnFocus = false,
    })

    Macro:AddButton({Text = 'Add New Macro', Callback = function()
        functions.uiAction(function()
            assert(not functions.busy(), "Hentikan macro sebelum menambah file")
            local name = tostring(Options.MacroInput.Value or ""):match("^%s*(.-)%s*$")
            name = name:gsub("%.json$", "")
            assert(name ~= "" and name ~= "." and name ~= ".."
                and not name:find('[%c/\\:*?"<>|]'), "Nama macro kosong/tidak valid")
            functions.prepareDirectory()
            local path = M.Directory .. "/" .. name .. ".json"
            assert(not isfile(path), "Macro sudah ada")
            writefile(path, HS:JSONEncode({Version = 2, PlaceId = game.PlaceId, Actions = {}}))
            Options.ListMacro:SetValues(functions.getMacroFiles())
            Options.ListMacro:SetValue(name)
            Options.MacroInput:SetValue("")
            functions.status("File dibuat: " .. name)
        end)
    end})

    Macro:AddButton({Text = 'Delete Macro', Callback = function()
        functions.uiAction(function()
            assert(not functions.busy(), "Hentikan macro sebelum menghapus file")
            assert(selectedName, "Pilih macro terlebih dahulu")
            local path = functions.filePath()
            assert(isfile(path), "File tidak ditemukan")
            delfile(path)
            Options.ListMacro:SetValue(nil)
            Options.ListMacro:SetValues(functions.getMacroFiles())
            functions.status("File dihapus")
        end)
    end})

    Macro:AddToggle('RecordMacro', {
        Text = 'Record Macro', 
        Default = false, 
        Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            if value then
                if not selectedName or functions.busy() then
                    functions.setToggle('RecordMacro', false)
                    functions.notify("Pilih file macro dan hentikan playback terlebih dahulu")
                    return
                end
                local ok, result, err = functions.protected("StartRecording", functions.StartRecording, true)
                if not ok or not result then
                    functions.setToggle('RecordMacro', false)
                    functions.notify(ok and (err or "Recording tidak dapat dimulai") or result)
                end
            elseif M.Recording then
                functions.spawnTask(function() functions.StopAndSave() end)
            end
        end,
    })

    Macro:AddDropdown('ModeMacro', {
        Text = 'Mode', 
        Values = {'Money', 'Time'}, 
        Default = 1, 
        Multi = false,
        Callback = function(value)
            if value == "Money" or value == "Time" then M.Mode = value end
        end,
    })

    Macro:AddToggle('MacroPlay', {
        Text = 'Play Macro', Default = false, Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            if not value then functions.StopPlayback(); return end
            if not selectedName or functions.busy() then
                functions.setToggle('MacroPlay', false)
                functions.notify("Pilih file macro dan hentikan recording terlebih dahulu")
                return
            end
            functions.spawnTask(function()
                if not Toggles.MacroPlay.Value or not lifecycle.Alive then return end
                local ok, result, err = functions.protected("Play toggle", functions.Play, true)
                if not ok then functions.notify(result)
                elseif not result then functions.notify(err or "Playback tidak dapat dimulai") end
                functions.setToggle('MacroPlay', false)
            end)
        end,
    })

    Voting:AddDropdown('Difficulty', {
        Text = 'Difficulty', 
        Values = {'Easy', 'Normal', 'Hard', 'Insane'}, 
        Default = 1, 
        Multi = false,
        Callback = function(value)
            if value == "Easy" or value == "Normal" or value == "Hard" or value == "Insane" then
                Variables.Difficulty = value
            end
        end,
    })


    Voting:AddToggle("AutoVoting", {
        Text = "Auto Voting", Default = false, Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            local ok, result, err = functions.protected("AutoVoting toggle", functions.AutoVoting, value)
            if not ok or not result then
                functions.stopFeature("AutoVoting")
                functions.setToggle("AutoVoting", false)
                functions.notify(ok and (err or "AutoVoting gagal") or result)
            end
        end,
    })

    Voting:AddToggle("AutoReply", {
        Text = "Auto Reply", Default = false, Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            local ok, result, err = functions.protected("AutoReply toggle", functions.AutoReply, value)
            if not ok or not result then
                functions.stopFeature("AutoReply")
                functions.setToggle("AutoReply", false)
                functions.notify(ok and (err or "AutoReply gagal") or result)
            end
        end,
    })
    
    Voting:AddToggle("AutoSkip", {
        Text = "Auto Skip", Default = false, Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            local ok, result, err = functions.protected("AutoSkip toggle", functions.AutoSkip, value)
            if not ok or not result then
                functions.stopFeature("AutoSkip")
                functions.setToggle("AutoSkip", false)
                functions.notify(ok and (err or "AutoSkip gagal") or result)
            end
        end,
    })
    
    Voting:AddDropdown('Speed', {
        Text = 'Speed', 
        Values = {1, 1.5, 2}, 
        Default = 1, 
        Multi = false,
        Callback = function(value)
            local speed = tonumber(value)
            if speed == 1 or speed == 1.5 or speed == 2 then
                Variables.Speed = speed
                if not uiSync and Toggle.AutoSetSpeed then
                    functions.protected("Speed berubah", functions.AutoSetSpeed, true)
                end
            end
        end,
    })
    
    Voting:AddToggle("AutoSetSpeed", {
        Text = "Auto Set Speed", Default = false, Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            local ok, result, err = functions.protected("AutoSetSpeed toggle", functions.AutoSetSpeed, value)
            if not ok or not result then
                functions.stopFeature("AutoSetSpeed")
                functions.setToggle("AutoSetSpeed", false)
                functions.notify(ok and (err or "AutoSetSpeed gagal") or result)
            end
        end,
    })
    
    Summon:AddDropdown('AmountSummon', {
        Text = 'Amount', 
        Values = {1, 10}, 
        Default = 1, 
        Multi = false,
        Callback = function(value)
            if value == 1 or value == 10 then
                Variables.AmountSummon = value
            end
        end,
    })
    
    Summon:AddToggle("AutoSummon", {
        Text = "Auto Summon", Default = false, Risky = false,
        Callback = function(value)
            if uiSync or not M.Ready then return end
            local ok, result, err = functions.protected("AutoSummon toggle", functions.AutoSummon, value)
            if not ok or not result then
                functions.stopFeature("AutoSummon")
                functions.setToggle("AutoSummon", false)
                functions.notify(ok and (err or "AutoSummon gagal") or result)
            end
        end,
    })
    
    functions.ConfigureSaveManager()
    SaveManager:SetLibrary(Library)
    SaveManager:IgnoreThemeSettings()
    SaveManager:SetIgnoreIndexes({ "MenuKeybind" })
    
    SaveManager:SetFolder("GoaHub/Ctd")
    SaveManager:BuildConfigSection(Tabs.Settings)

    Library:SetFont(Enum.Font.GothamBold)
    

end

-- ================================================================
-- 6. INISIALISASI — pemasangan event, hook, dan startup
-- ================================================================

functions.BindMacroResponses = function()
for _, name in ipairs({
    "TowerAdd",
    "TowerUpgrade",
    "TowerTarget",
}) do
    local kind = name

    local remote = R[kind]
    local previous = macroConnections[kind]
    if remote and (not previous or previous.Remote ~= remote) then
    if previous then previous.Connection:Disconnect() end
    local connection = functions.connect(remote.OnClientEvent, function(data)
        if type(data) ~= "table" then
            return
        end

        local success, err = pcall(function()
            functions.syncPlaybackTower(kind, data)
            local waiting = M.Waiting

            if waiting
                and not waiting.Result
                and waiting.Kind == kind
                and functions.matches(waiting, data)
            then
                if kind ~= "TowerTarget"
                    or data.TargetMode ~= waiting.BeforeTarget
                then
                    waiting.Result = data
                end
            end

            if M.Recording or M.Saving then
                functions.recordResponse(kind, data)
            end
        end)

        if not success then
            if M.Recording or M.Saving then
                M.Error = M.Error or tostring(err)
            end

            functions.logMessage("ERROR", "Macro: " .. tostring(err))
        end
    end)
    macroConnections[kind] = {Remote = remote, Connection = connection}
    end
end

end
functions.InstallMacroHook = function()
if M.HookInstalled then return end
assert(
    type(hookmetamethod) == "function"
        and type(getnamecallmethod) == "function",
    "hookmetamethod/getnamecallmethod tidak tersedia"
)

-- Memanggil fungsi langsung mencegah konteks __namecall berubah
-- menjadi GetComponents ketika remote diteruskan.
fireServer = R.TowerAdd.FireServer

oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
    local method = getnamecallmethod()

    if lifecycle.Alive and M.Recording and method == "FireServer" then
        for kind, remote in pairs(R) do
            if self == remote and (kind == "TowerAdd" or kind == "TowerUpgrade"
                or kind == "TowerSell" or kind == "TowerTarget") then
                local success, err = pcall(functions.capture, kind, ...)

                if not success then
                    M.Error = M.Error or tostring(err)
                    functions.logMessage("ERROR", "Macro: " .. tostring(err))
                end

                return fireServer(self, ...)
            end
        end
    end

    return oldNamecall(self, ...)
end)

M.HookInstalled = true

end
functions.BindAutosave = function()
    local playerGui = PlayerGui
    local connected = setmetatable({}, { __mode = "k" })

    local function bind(content)
        if content.Name ~= "Content"
            or not content:IsA("GuiObject")
            or connected[content]
        then
            return
        end

        local finalUI = content.Parent
        local gameUI = finalUI and finalUI.Parent

        if not finalUI
            or finalUI.Name ~= "FinalUI"
            or not gameUI
            or gameUI.Name ~= "GameUI"
            or gameUI.Parent ~= playerGui
        then
            return
        end

        connected[content] = true

        local function check()
            if content.Visible and M.Recording and not M.Saving then
                functions.debugLog("FinalUI terlihat, otomatis menyimpan...")
                functions.StopAndSave()
            end
        end

        functions.connect(content:GetPropertyChangedSignal("Visible"), check)
        check()
    end

    functions.connect(playerGui.DescendantAdded, bind)

    for _, object in ipairs(playerGui:GetDescendants()) do
        bind(object)
    end
end
functions.Initialize = function()
    functions.logMessage("INFO", "Mulai inisialisasi...")
    local deadline = os.clock() + 60
    while not game:IsLoaded() do
        assert(os.clock() < deadline, "Game belum selesai dimuat setelah 60 detik")
        task.wait(0.1)
    end
    LP = game:GetService("Players").LocalPlayer
    assert(LP, "LocalPlayer belum tersedia; jalankan dari client")
    PlayerGui = functions.waitChild(LP, "PlayerGui")
    functions.prepareDirectory()
    functions.BuildUI()
    functions.BindAutosave()
    functions.updateWave()
    functions.spawnTask(function()
        while lifecycle.Alive do
            functions.updateWave()
            task.wait(0.03)
        end
    end)
    M.Ready = true
    functions.logMessage("INFO", "UI siap; remote diperiksa saat fitur diaktifkan. Memuat autoload...")
    functions.protected("Autoload", function() SaveManager:LoadAutoloadConfig() end)
    functions.logMessage("INFO", "Siap. Pilih/buat file, lalu aktifkan Record Macro atau Play Macro.")
end

if env.alreadyRunGoaHub or env.ctdLoading then
    warn("[GoaHub] Sudah berjalan/sedang dimuat. Rejoin jika versi lama macet.")
    return
end
env.ctdLoading = true
local ok, err = xpcall(functions.Initialize, functions.traceback)
env.ctdLoading = nil
if ok then
    env.alreadyRunGoaHub = true
else
    lifecycle.Alive = false
    M.Recording, M.Ready = false, false
    for _, connection in ipairs(lifecycle.Connections) do
        pcall(function() connection:Disconnect() end)
    end
    if Library and type(Library.Unload) == "function" then
        pcall(function() Library:Unload() end)
    end
    env.alreadyRunGoaHub = nil
    functions.logMessage("ERROR", "Gagal inisialisasi:\n" .. tostring(err))
end

