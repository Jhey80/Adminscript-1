-- ================================================
-- SCRIPT DE ADMIN COMPLETO - VERIFICADO v4 (FINAL)
-- ColÃ³calo en: ServerScriptService
-- ================================================

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")

-- ================================================
-- CONFIGURACIÃ“N
-- ================================================
local ADMIN_IDS = {
	7010544216, -- Tu UserID
}

-- ================================================
-- ALMACENAMIENTO
-- ================================================
local banned       = {}
local tempBanned   = {}
local rkillLoops   = {}
local rflyLoops    = {}
local rnoclipLoops = {}
local flyData      = {}
local noclipData   = {}
local godData      = {}
local invisData    = {}
local freezeData   = {}
local rtoolLoops   = {}
local cloneData    = {}

-- ================================================
-- UTILIDADES
-- ================================================

local function isAdmin(player)
	for _, id in ipairs(ADMIN_IDS) do
		if id == player.UserId then return true end
	end
	return false
end

local function getChar(p)  return p and p.Character end
local function getHRP(p)   local c = getChar(p); return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum(p)   local c = getChar(p); return c and c:FindFirstChildOfClass("Humanoid") end

local function notify(player, msg)
	pcall(function()
		local h = player.Character and player.Character:FindFirstChild("Head")
		if h then game:GetService("Chat"):Chat(h, "[âœ“] "..msg, Enum.ChatColor.Green) end
	end)
end

local function findPlayer(name)
	if not name then return nil end
	name = name:lower()
	for _, p in ipairs(Players:GetPlayers()) do
		if p.Name:lower():find(name, 1, true) then return p end
	end
	return nil
end

local function getTargets(admin, name)
	if not name then return {admin} end
	local t = name:lower()
	if t == "all" then
		return Players:GetPlayers()
	elseif t == "others" then
		local list = {}
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= admin then table.insert(list, p) end
		end
		return list
	elseif t == "me" then
		return {admin}
	else
		local p = findPlayer(name)
		return p and {p} or {}
	end
end

-- ================================================
-- FLY â€” FIX: CFrame correcto, sin operaciones invÃ¡lidas
-- ================================================
local function enableFly(player)
	if flyData[player] then return end
	local char = getChar(player)
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum then return end

	hum.PlatformStand = true

	local att0 = Instance.new("Attachment")
	att0.Parent = hrp

	-- Part invisible fija para AlignOrientation
	local anchor = Instance.new("Part")
	anchor.Name = "FlyAnchor"
	anchor.Anchored = true
	anchor.CanCollide = false
	anchor.Transparency = 1
	anchor.Size = Vector3.new(0.1, 0.1, 0.1)
	anchor.CFrame = hrp.CFrame
	anchor.Parent = workspace

	local att1 = Instance.new("Attachment")
	att1.Parent = anchor

	-- LinearVelocity: mantener al jugador flotando (velocidad 0)
	-- Parent se setea AL FINAL para evitar warnings de constraint
	local lv = Instance.new("LinearVelocity")
	lv.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
	lv.ForceLimitMode = Enum.ForceLimitMode.PerAxis
	lv.MaxAxesForce = Vector3.new(1e5, 1e5, 1e5)
	lv.VectorVelocity = Vector3.zero
	lv.RelativeTo = Enum.ActuatorRelativeTo.World
	lv.Attachment0 = att0
	lv.Parent = hrp

	-- AlignOrientation: mantener la rotaciÃ³n del personaje
	-- Parent se setea AL FINAL
	local ao = Instance.new("AlignOrientation")
	ao.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
	ao.Responsiveness = 50
	ao.RigidityEnabled = false
	ao.Attachment0 = att0
	ao.Attachment1 = att1
	ao.Parent = hrp

	local conn
	conn = RunService.Heartbeat:Connect(function()
		if not flyData[player] then
			conn:Disconnect()
			return
		end
		if not hrp or not hrp.Parent then
			conn:Disconnect()
			return
		end
		-- FIX CRÃTICO: usar hrp.CFrame directamente, sin resta invÃ¡lida
		anchor.CFrame = hrp.CFrame
		lv.VectorVelocity = Vector3.zero
	end)

	flyData[player] = {lv=lv, ao=ao, att0=att0, att1=att1, anchor=anchor, conn=conn}
	notify(player, "Fly ON")
end

local function disableFly(player)
	if not flyData[player] then return end
	local d = flyData[player]
	if d.conn   then d.conn:Disconnect() end
	if d.lv     then d.lv:Destroy()     end
	if d.ao     then d.ao:Destroy()     end
	if d.att0   then d.att0:Destroy()   end
	if d.att1   then d.att1:Destroy()   end
	if d.anchor then d.anchor:Destroy() end
	flyData[player] = nil
	if rflyLoops[player] then
		-- rflyLoops puede ser boolean (flag) o conexiÃ³n â€” solo limpiamos la referencia
		if type(rflyLoops[player]) ~= "boolean" then
			rflyLoops[player]:Disconnect()
		end
		rflyLoops[player] = nil
	end
	local hum = getHum(player)
	if hum then hum.PlatformStand = false end
	notify(player, "Fly OFF")
end

-- ================================================
-- NOCLIP â€” FIX: cachear partes para mejor performance
-- ================================================
local function enableNoclip(player)
	if noclipData[player] then return end
	local conn = RunService.Stepped:Connect(function()
		local c = getChar(player)
		if not c then return end
		-- Cachear solo HRP y partes visibles es mÃ¡s eficiente
		for _, p in ipairs(c:GetChildren()) do
			if p:IsA("BasePart") then
				p.CanCollide = false
			end
		end
	end)
	noclipData[player] = conn
	notify(player, "Noclip ON")
end

local function disableNoclip(player)
	if not noclipData[player] then return end
	noclipData[player]:Disconnect()
	noclipData[player] = nil
	if rnoclipLoops[player] then
		-- rnoclipLoops puede ser boolean (flag) o conexiÃ³n â€” verificar antes
		if type(rnoclipLoops[player]) ~= "boolean" then
			rnoclipLoops[player]:Disconnect()
		end
		rnoclipLoops[player] = nil
	end
	local c = getChar(player)
	if c then
		for _, p in ipairs(c:GetChildren()) do
			if p:IsA("BasePart") then p.CanCollide = true end
		end
	end
	notify(player, "Noclip OFF")
end

-- ================================================
-- GOD â€” FIX: re-aplicar tras respawn con CharacterAdded
-- ================================================
local function applyGod(player)
	local hum = getHum(player)
	if not hum then return end
	hum.MaxHealth = math.huge
	hum.Health    = math.huge
end

local function enableGod(player)
	applyGod(player)
	godData[player] = true
	notify(player, "God ON")
end

local function disableGod(player)
	local hum = getHum(player)
	if hum then
		hum.MaxHealth = 100
		hum.Health    = 100
	end
	godData[player] = nil
	notify(player, "God OFF")
end

-- ================================================
-- INVISIBLE â€” FIX: re-aplicar tras respawn
-- ================================================
local function applyInvis(player)
	local c = getChar(player)
	if not c then return end
	for _, p in ipairs(c:GetDescendants()) do
		if p:IsA("BasePart") or p:IsA("Decal") then
			p.Transparency = 1
		end
	end
end

local function enableInvis(player)
	applyInvis(player)
	invisData[player] = true
	notify(player, "Invisible ON")
end

local function disableInvis(player)
	local c = getChar(player)
	if c then
		for _, p in ipairs(c:GetDescendants()) do
			if p:IsA("BasePart") then
				p.Transparency = p.Name == "HumanoidRootPart" and 1 or 0
			elseif p:IsA("Decal") then
				p.Transparency = 0
			end
		end
	end
	invisData[player] = nil
	notify(player, "Visible ON")
end

-- ================================================
-- FREEZE â€” FIX: re-aplicar tras respawn
-- ================================================
local function applyFreeze(player)
	local c = getChar(player)
	if not c then return end
	for _, p in ipairs(c:GetDescendants()) do
		if p:IsA("BasePart") then p.Anchored = true end
	end
end

local function enableFreeze(player)
	applyFreeze(player)
	freezeData[player] = true
	notify(player, "Freeze ON")
end

local function disableFreeze(player)
	local c = getChar(player)
	if c then
		for _, p in ipairs(c:GetDescendants()) do
			if p:IsA("BasePart") then p.Anchored = false end
		end
	end
	freezeData[player] = nil
	notify(player, "Freeze OFF")
end

-- ================================================
-- CLONE
-- ================================================
local function clonePlayer(player)
	local char = getChar(player)
	if not char then return end
	local hrp = char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end

	-- FIX: eliminar clon anterior si existe
	if cloneData[player] and cloneData[player].Parent then
		cloneData[player]:Destroy()
		cloneData[player] = nil
	end

	local clone = char:Clone()
	clone.Name = player.Name.."_Clone"

	-- Limpiar scripts
	for _, s in ipairs(clone:GetDescendants()) do
		if s:IsA("Script") or s:IsA("LocalScript") or s:IsA("ModuleScript") then
			s:Destroy()
		end
	end

	-- Quitar Humanoid
	local cloneHum = clone:FindFirstChildOfClass("Humanoid")
	if cloneHum then cloneHum:Destroy() end

	-- Anclar todas las partes
	for _, p in ipairs(clone:GetDescendants()) do
		if p:IsA("BasePart") then
			p.Anchored    = true
			p.CanCollide  = false
		end
	end

	-- Posicionar al lado del jugador
	local cloneHRP = clone:FindFirstChild("HumanoidRootPart")
	if cloneHRP then
		cloneHRP.CFrame = hrp.CFrame * CFrame.new(4, 0, 0)
	end

	clone.Parent = workspace
	cloneData[player] = clone
	notify(player, "Clone creado! (se destruye en 30s)")

	-- Auto-destruir en 30 segundos
	task.delay(30, function()
		if clone and clone.Parent then
			clone:Destroy()
			if cloneData[player] == clone then
				cloneData[player] = nil
			end
		end
	end)
end

local function removeClone(player)
	if cloneData[player] and cloneData[player].Parent then
		cloneData[player]:Destroy()
		cloneData[player] = nil
		notify(player, "Clone eliminado")
	end
end

-- ================================================
-- FIX PERSISTENCIA: re-aplicar efectos tras respawn
-- ================================================
local function onCharacterAdded(player, char)
	-- Esperar a que el personaje cargue completamente
	char:WaitForChild("HumanoidRootPart", 5)
	char:WaitForChild("Humanoid", 5)
	task.wait(0.2) -- pequeÃ±o delay para asegurar que todo estÃ¡ listo

	if godData[player]   then applyGod(player)    end
	if invisData[player] then applyInvis(player)   end
	if freezeData[player] then applyFreeze(player) end

	-- Fly: si tenÃ­a rfly activo, re-activar
	if rflyLoops[player] then
		task.wait(0.5)
		enableFly(player)
	end

	-- Noclip: si tenÃ­a rnoclip activo, re-activar
	if rnoclipLoops[player] then
		task.wait(0.1)
		enableNoclip(player)
	end
end

-- ================================================
-- LIMPIAR EFECTOS AL RESPAWN (evitar datos corruptos)
-- ================================================
local function onCharacterRemoving(player)
	-- Limpiar fly cuando muere (se re-aplicarÃ¡ si tiene rfly)
	if flyData[player] then
		local d = flyData[player]
		if d.conn   then d.conn:Disconnect() end
		if d.lv     then d.lv:Destroy()     end
		if d.ao     then d.ao:Destroy()     end
		if d.att0   then d.att0:Destroy()   end
		if d.att1   then d.att1:Destroy()   end
		if d.anchor then d.anchor:Destroy() end
		flyData[player] = nil
		local hum = getHum(player)
		if hum then hum.PlatformStand = false end
	end
	-- Limpiar noclip cuando muere (se re-aplicarÃ¡ si tiene rnoclip)
	if noclipData[player] then
		noclipData[player]:Disconnect()
		noclipData[player] = nil
	end
end

-- ================================================
-- PROCESAR COMANDOS
-- ================================================
local function processCommand(player, message)
	if not isAdmin(player) then return end

	-- Limpiar mensaje: quitar /e wave, /w, espacios
	message = message:gsub("^/%a+ ", ""):gsub("^%s+", ""):gsub("%s+$", "")

	local args = {}
	for word in message:gmatch("%S+") do
		table.insert(args, word)
	end
	if #args == 0 then return end

	local cmd = args[1]:lower()
	table.remove(args, 1)

	-- KILL
	if cmd == "kill" then
		for _, t in ipairs(getTargets(player, args[1])) do
			local h = getHum(t)
			if h then h.Health = 0 end
		end
		notify(player, "Kill âœ“")

	-- RKILL
	elseif cmd == "rkill" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if rkillLoops[t] then
				rkillLoops[t]:Disconnect()
				rkillLoops[t] = nil
				notify(player, "rkill OFF: "..t.Name)
			else
				local target = t
				rkillLoops[target] = RunService.Heartbeat:Connect(function()
					local h = getHum(target)
					if h then h.Health = 0 end
				end)
				notify(player, "rkill ON: "..t.Name)
			end
		end

	-- UNRKILL
	elseif cmd == "unrkill" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if rkillLoops[t] then
				rkillLoops[t]:Disconnect()
				rkillLoops[t] = nil
			end
		end
		notify(player, "rkill OFF âœ“")

	-- KICK
	elseif cmd == "kick" then
		local reason = args[2] or "Sin razÃ³n"
		for _, t in ipairs(getTargets(player, args[1])) do
			if t ~= player then
				t:Kick("Kickeado: "..reason)
			end
		end
		notify(player, "Kick âœ“")

	-- BAN
	elseif cmd == "ban" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if t ~= player then
				banned[t.UserId] = true
				t:Kick("Baneado permanentemente.")
			end
		end
		notify(player, "Ban âœ“")

	-- TEMPBAN
	elseif cmd == "tempban" then
		local secs = tonumber(args[2]) or 300
		for _, t in ipairs(getTargets(player, args[1])) do
			if t ~= player then
				tempBanned[t.UserId] = os.time() + secs
				t:Kick("Baneado por "..secs.." segundos.")
			end
		end
		notify(player, "Tempban "..secs.."s âœ“")

	-- UNBAN
	elseif cmd == "unban" then
		local name = args[1]
		if name then
			local target = findPlayer(name)
			if target then
				banned[target.UserId]     = nil
				tempBanned[target.UserId] = nil
				notify(player, target.Name.." desbaneado âœ“")
			else
				notify(player, "Jugador '"..name.."' no estÃ¡ en el servidor.")
			end
		end

	-- FLY (toggle)
	elseif cmd == "fly" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if flyData[t] then disableFly(t) else enableFly(t) end
		end

	-- UNFLY
	elseif cmd == "unfly" then
		for _, t in ipairs(getTargets(player, args[1])) do
			disableFly(t)
		end

	-- RFLY (fly persistente)
	elseif cmd == "rfly" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if rflyLoops[t] then
				rflyLoops[t]:Disconnect()
				rflyLoops[t] = nil
				disableFly(t)
				notify(player, "rfly OFF: "..t.Name)
			else
				enableFly(t)
				-- rflyLoops[t] se usa como flag; onCharacterAdded re-aplica
				rflyLoops[t] = true
				notify(player, "rfly ON: "..t.Name)
			end
		end

	-- NOCLIP (toggle)
	elseif cmd == "noclip" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if noclipData[t] then disableNoclip(t) else enableNoclip(t) end
		end

	-- UNNOCLIP
	elseif cmd == "unnoclip" then
		for _, t in ipairs(getTargets(player, args[1])) do
			disableNoclip(t)
		end
		notify(player, "Noclip OFF âœ“")

	-- RNOCLIP (noclip persistente)
	elseif cmd == "rnoclip" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if rnoclipLoops[t] then
				rnoclipLoops[t]:Disconnect()
				rnoclipLoops[t] = nil
				disableNoclip(t)
				notify(player, "rnoclip OFF: "..t.Name)
			else
				enableNoclip(t)
				-- rnoclipLoops[t] se usa como flag; onCharacterAdded re-aplica
				rnoclipLoops[t] = true
				notify(player, "rnoclip ON: "..t.Name)
			end
		end

	-- GOD (toggle)
	elseif cmd == "god" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if godData[t] then disableGod(t) else enableGod(t) end
		end

	-- UNGOD
	elseif cmd == "ungod" then
		for _, t in ipairs(getTargets(player, args[1])) do
			disableGod(t)
		end
		notify(player, "God OFF âœ“")

	-- INVISIBLE (toggle)
	elseif cmd == "invisible" or cmd == "invis" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if invisData[t] then disableInvis(t) else enableInvis(t) end
		end

	-- VISIBLE
	elseif cmd == "visible" or cmd == "uninvis" then
		for _, t in ipairs(getTargets(player, args[1])) do
			disableInvis(t)
		end
		notify(player, "Visible âœ“")

	-- FREEZE (toggle)
	elseif cmd == "freeze" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if freezeData[t] then disableFreeze(t) else enableFreeze(t) end
		end

	-- UNFREEZE
	elseif cmd == "unfreeze" or cmd == "thaw" then
		for _, t in ipairs(getTargets(player, args[1])) do
			disableFreeze(t)
		end
		notify(player, "Unfreeze âœ“")

	-- HEAL
	elseif cmd == "heal" then
		for _, t in ipairs(getTargets(player, args[1])) do
			local h = getHum(t)
			if h then h.Health = h.MaxHealth end
		end
		notify(player, "Heal âœ“")

	-- SPEED / WS
	elseif cmd == "speed" or cmd == "ws" then
		local isNum = tonumber(args[1])
		local targets, spd
		if isNum then
			targets = {player}
			spd = isNum
		else
			targets = getTargets(player, args[1])
			spd = tonumber(args[2]) or 16
		end
		for _, t in ipairs(targets) do
			local h = getHum(t)
			if h then h.WalkSpeed = spd end
		end
		notify(player, "Speed "..spd.." âœ“")

	-- JUMP / JP
	elseif cmd == "jump" or cmd == "jp" then
		local isNum = tonumber(args[1])
		local targets, pwr
		if isNum then
			targets = {player}
			pwr = isNum
		else
			targets = getTargets(player, args[1])
			pwr = tonumber(args[2]) or 50
		end
		for _, t in ipairs(targets) do
			local h = getHum(t)
			if h then h.JumpPower = pwr end
		end
		notify(player, "Jump "..pwr.." âœ“")

	-- TELEPORT
	elseif cmd == "tp" then
		local t1 = findPlayer(args[1])
		local t2 = findPlayer(args[2])
		if t1 and t2 then
			local h1 = getHRP(t1)
			local h2 = getHRP(t2)
			if h1 and h2 then
				h1.CFrame = h2.CFrame * CFrame.new(3, 0, 0)
			end
			notify(player, t1.Name.." â†’ "..t2.Name)
		elseif t1 then
			local ha = getHRP(player)
			local ht = getHRP(t1)
			if ha and ht then
				ha.CFrame = ht.CFrame * CFrame.new(3, 0, 0)
			end
			notify(player, "Teletransportado a "..t1.Name)
		else
			notify(player, "Uso: tp nombre  |  tp nombre1 nombre2")
		end

	-- RESPAWN â€” FIX: limpiar efectos antes de respawnear
	elseif cmd == "respawn" then
		for _, t in ipairs(getTargets(player, args[1])) do
			-- Limpiar efectos que no persisten (fly/noclip se re-aplican por onCharacterAdded)
			if flyData[t] then disableFly(t) end
			if noclipData[t] then disableNoclip(t) end
			t:LoadCharacter()
		end
		notify(player, "Respawn âœ“")

	-- REMOVETOOL
	elseif cmd == "removetool" or cmd == "rt" then
		for _, t in ipairs(getTargets(player, args[1])) do
			local bp = t:FindFirstChildOfClass("Backpack")
			if bp then
				for _, item in ipairs(bp:GetChildren()) do
					if item:IsA("Tool") then item:Destroy() end
				end
			end
			local c = getChar(t)
			if c then
				for _, item in ipairs(c:GetChildren()) do
					if item:IsA("Tool") then item:Destroy() end
				end
			end
		end
		notify(player, "Tools quitadas âœ“")

	-- QREMOVETOOL â€” FIX: usar ChildAdded en vez de Heartbeat polling
	elseif cmd == "qremovetool" or cmd == "qrt" then
		for _, t in ipairs(getTargets(player, args[1])) do
			if rtoolLoops[t] then
				-- Desconectar todos los eventos guardados
				for _, conn in ipairs(rtoolLoops[t]) do
					conn:Disconnect()
				end
				rtoolLoops[t] = nil
				notify(player, "qrt OFF: "..t.Name)
			else
				local target = t
				local connections = {}

				local function removeTool(item)
					if item:IsA("Tool") then item:Destroy() end
				end

				-- Quitar tools actuales
				local bp = target:FindFirstChildOfClass("Backpack")
				if bp then
					for _, item in ipairs(bp:GetChildren()) do removeTool(item) end
					table.insert(connections, bp.ChildAdded:Connect(removeTool))
				end
				local c = getChar(target)
				if c then
					for _, item in ipairs(c:GetChildren()) do removeTool(item) end
					table.insert(connections, c.ChildAdded:Connect(removeTool))
				end
				-- Actualizar conexiones si respawnea (limpiar viejas primero para evitar acumulaciÃ³n)
				local charConn, bpConn, charChildConn
				charConn = target.CharacterAdded:Connect(function
