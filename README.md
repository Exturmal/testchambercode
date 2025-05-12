local utils = require(game.ReplicatedStorage.modules.utils)
local particles = require(game.ReplicatedStorage.modules.Particles)
local screenEffects = require(game.ReplicatedStorage.modules.ScreenEffects)
local sounds = require(game.ReplicatedStorage.modules.Sounds)
local anims = require(game.ReplicatedStorage.modules.Animations)
local zones = require(game.ReplicatedStorage.modules.Zone)
local status = require(game.ServerScriptService.modules.Status)
local prompttypes = require(game.ServerScriptService.modules.PromptTypes) --for breaker

local tcA = script.Parent

--get the folders

local cont = tcA.Controls
local values = tcA.Values
local contvalues = tcA.ControlValues
local addeditems = tcA.AddedItems
local anom = tcA.Anomaly
local firevalues = tcA.FireValues

--get the guis

local bargui = tcA.Anomaly.ChargeScreenPart.SurfaceGui
local bholder = bargui.Main.BarHolder

local valBarGui = tcA.Anomaly.ValuesScreenPart.SurfaceGui
local vBalHolder = valBarGui.Main.Frame

local PatientAmtScreen = anom.PatientScreen.SurfaceGui
local PatientScreen = PatientAmtScreen.Main

local LoadedItemsScreen = anom.LoadedItemScreen.SurfaceGui
local ItemScreen = LoadedItemsScreen.Main

local VialScreen = anom.VialScreen.SurfaceGui
local vialGui = VialScreen.Main

local GenGui = anom.GeneratorScreen.gui.scr

local fluxGui = anom.FluxInfectChance.SurfaceGui

--

local spamprevention = 0
local spamlimit = 3 --stop people from mashing the button with no charge
local powerdeb = false
local rng = utils.GetRandom(1, 100)

local screffected = {}--people to give screeneffect to when we do the pulse

local Items = {
	["Yellow Mushroom"] = {
		Type = "OnAdd",
		func = function(testingchamber)
			testingchamber.Values.ChargeNeeded.Value *= 0.5
		end,
		Name = "Yellow Mushroom",
		Desc = "Increases Charge Rate",
		IMG = "rbxassetid://80579298336656"
	},
	["Marshmellow"] = {
		Type = "Consistent",
		func = function(testingchamber)
			testingchamber.Values.ShockDamage.Value *= 0.5
		end,
		Name = "Marshmellow",
		Desc = "#NAME=Marshmellow-SHCK.dmg / 2",
		IMG = "rbxassetid://80579298336656"
	},
	["Black Cherry"] = {
		Type = "Outcome",
		func = function(testingchamber)
			testingchamber.Values.InfectionType.Value = "Komodo"
		end,
		Name = "Black Cherry",
		Desc = "#NAME=Black Cherry-INFCT.type=Komodo",
		IMG = "rbxassetid://80579298336656"
	},
}

local colors = {
	["Red"] = Color3.new(1, 0, 0),
	["Green"] = Color3.new(0.109804, 0.603922, 0),
}

local resetting = false
local function resetcontrols()
	resetting = true
	contvalues.Gain.Value = 0
	contvalues.Dampen.Value = 50
	contvalues.Tuning.Value = 50
	task.delay(0.1, function()
		resetting = false
	end)
end

local function updateBarWithTween(bar, value, divisor, duration)
	local targetSize = UDim2.new(math.clamp(value / divisor, 0, 1), 0, 1, 0)
	local tweenInfo = TweenInfo.new(duration, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
	local goal = {Size = targetSize}
	local tween = game.TweenService:Create(bar, tweenInfo, goal)
	tween:Play()
end

local function updateValueGui()
	updateBarWithTween(vBalHolder.InfectionRate.Bar.Progress1, values.InfectionRate.Value, 100, 0.1)
	updateBarWithTween(vBalHolder.CardiacRisk.Bar.Progress1, values.HeartAttackRate.Value, 100, 0.1)
	updateBarWithTween(vBalHolder.ShockDamage.Bar.Progress1, values.ShockDamage.Value, 100, 0.1)
end

local unfluxedvalues = {
	["InfectionRate"] = 5,
	["HeartAttackRate"] = 2,
	["ShockDamage"] = 10,
}

local function unflux()
	values.InfectionRate.Value = unfluxedvalues.InfectionRate
	values.HeartAttackRate.Value = unfluxedvalues.HeartAttackRate
	values.ShockDamage.Value = unfluxedvalues.ShockDamage
	updateValueGui()
end

local function sliderlights(value)
	if values.Powered.Value == true then
		cont[value.Name].Light.Color = Color3.fromHSV(0.1295 - (value.Value / 1000), 0.560479, math.clamp((value.Value / 100 * 2) * 0.972549, 0, 0.972549))
		cont[value.Name].Light.Light.Color = Color3.fromHSV(0.1295 - (value.Value / 1000), 0.560479, math.clamp((value.Value / 100 * 2) * 0.972549, 0, 0.972549))
		cont[value.Name].Light.Light.Brightness = math.clamp(value.Value / 100 * 3, 0, 3)
		cont[value.Name].Light.Light.Enabled = true
	else
		cont[value.Name].Light.Color = Color3.new()
		cont[value.Name].Light.Light.Enabled = false
	end
end

local function updateslider(value)
	local cyl = cont[value.Name].handle.CylindricalConstraint
	cyl.TargetPosition = math.clamp(value.Value/100*(cyl.UpperLimit), 0, cyl.UpperLimit)
	sliderlights(value)
end

local function updatechargegui()
	bholder.Progress1.Size = UDim2.new(math.clamp(firevalues.Charge.Value / values.ChargeNeeded.Value, 0, 1), 0, 1, 0)
	bholder.Progress2.Size = UDim2.new(math.clamp(((firevalues.Charge.Value - values.ChargeNeeded.Value) / (values.ChargeNeeded.Value * 0.2)), 0, 1), 0, 1, 0)
	bholder.Progress3.Size = UDim2.new(math.clamp(((firevalues.Charge.Value - values.ChargeNeeded.Value*1.2) / (values.ChargeNeeded.Value * 0.2)), 0, 1), 0, 1, 0)	
	bholder.Progress4.Size = UDim2.new(math.clamp(((firevalues.Charge.Value - values.ChargeNeeded.Value*1.4) / (values.ChargeNeeded.Value * 0.1)), 0, 1), 0, 1, 0)
end

local function changeparameters() --main func for determining the major values

	values.InfectionRate.Value = math.clamp(contvalues.Gain.Value - (contvalues.Dampen.Value * 1.5) + contvalues.Tuning.Value/4, 1, 100)		
	values.HeartAttackRate.Value = math.clamp(0.5 * ((contvalues.Gain.Value*2) - (contvalues.Tuning.Value*2) - contvalues.Dampen.Value), 1, 100)
	values.ShockDamage.Value = math.clamp(0.3 * (contvalues.Gain.Value + (contvalues.Tuning.Value) - (contvalues.Dampen.Value/2)), 5, 100)
	contvalues.Flux.Value = math.clamp(10 + (contvalues.Gain.Value /8) + (contvalues.Tuning.Value/4) - (contvalues.Dampen.Value/2), 1, 100)
	values.BreakerTripChance.Value = math.clamp(75 + (contvalues.Gain.Value/4) - (contvalues.Dampen.Value/4), 0, 100)

	--do item stuff here
	for i,v in pairs(addeditems:GetChildren()) do
		if Items[v.Name].Type == "Consistent" then
			Items[v.Name].func(tcA)
		end
	end

	unfluxedvalues.InfectionRate = values.InfectionRate.Value --update unfluxed values so we can reset to them after fluxing
	unfluxedvalues.HeartAttackRate = values.HeartAttackRate.Value
	unfluxedvalues.ShockDamage = values.ShockDamage.Value
	
	updateValueGui()
end

local function resetparameters()
	firevalues.VialUses.Value -= 1
	firevalues.Charge.Value = 0
	
	if firevalues.VialUses.Value == 0 then
		values.ChargeNeeded.Value = 100
		sounds.PlaySound(anom.ChargeScreenPart, "tcAVialExhausted")
		
		values.InfectionType.Value = ""
		cont.VialInsert.display.vialmodel:Destroy()
		
		--reset items
		for i,v in pairs(addeditems:GetChildren()) do
			v:Destroy()
		end
		for i, old in cont.ItemChamber.display:GetChildren() do
			old:Destroy()
		end
		for i,v in pairs(ItemScreen.List:GetChildren()) do
			if v ~= ItemScreen.List.UIListLayout and v ~= ItemScreen.List.UIPadding then
				v:Destroy()
			end
		end
	end
end

--changes slider positions to match values
for i, v in pairs(contvalues:GetChildren()) do
	v:GetPropertyChangedSignal("Value"):Connect(function()
		if v.Name ~= "Flux" then --for easy compat with text changing, we dont directly set flux so we dont want to do this here
			changeparameters(v) --do the meat
			updateslider(v)
		end
	end)
end

local heartattackvictims = {}
local listenbinds = {}

local function outcome(plr)
	rng = math.clamp(math.random(0, 100) - plr.CardiacHealth.Value, 10, 100)

	if rng >= values.HeartAttackRate.Value then
		print 'heart attack'
		anims.LoadAnimation(plr.Character, "Cough")
		game.ReplicatedStorage.ScreenEffectEvent:FireClient(plr, "Concuss", 0.5, 0.5)
		sounds.PlaySound(plr.Character.HumanoidRootPart, "HeartBeat")
		if not table.find(heartattackvictims, plr) then
			heartattackvictims[plr] = 1
		else
			heartattackvictims[plr] += 1
			if not table.find(listenbinds, plr) then
				listenbinds[plr] = plr.CharacterAdded:Connect(function()
					heartattackvictims[plr] = nil
					listenbinds[plr]:Disconnect()
					listenbinds[plr] = nil
				end)
			end
			if heartattackvictims[plr] >= 3 then
				--kill him!!!
				status.Bleed(plr, 1000)
			end
		end
		
	else
		print 'no heart attack'
	end
	
	--do item stuff here
	for i,v in pairs(addeditems:GetChildren()) do
		if Items[v.Name].Type == "Outcome" then
			Items[v.Name].func(tcA)
		end
	end
	
	--add some randomness to the infectionrate just so people dont turn at once
	local samplerng = math.clamp((firevalues.SampleChance.Value + math.random(-4, 4)) * 1.2, 0, 120)
	--modify rate by samplechance, its a modifier not a flat chance thing now
	status.TickInfect(plr, values.InfectionType.Value, (math.clamp(values.InfectionRate.Value * (samplerng/100), 0, 120)))

	--status.Shock(plr, values.ShockDamage.Value, anom.Main.Body, 1000, 1, 2) --replace this with something that isnt just yet another shock status. idk just make it different
end

local function turnoffchargesounds() --separate
	--turn off charge sounds
	sounds.StopSound(anom.ChargeScreenPart, "tcAChargeSparks")
	sounds.StopSound(anom.ChargeScreenPart, "tcAChargeWhine")
	sounds.StopSound(anom.ChargeScreenPart, "tcAOverChargeSparks1")
	sounds.StopSound(anom.ChargeScreenPart, "tcAOverChargeSparks2")
	sounds.StopSound(anom.ChargeScreenPart, "tcAOverChargeSparks3")
end

local hitplayers = {}
local function fire()--this is where we actually do all the math
	if values.InfectionType.Value == "" then return end
	if firevalues.VialUses.Value <= 0 then
		--not allowed, show error
		return
	end
	--only let it go if they actually charged a bit
	tcA.Controls.Charge.handle.CylindricalConstraint.TargetAngle = tcA.Controls.Charge.handle.CylindricalConstraint.UpperAngle
	if values.Powered.Value == true then
		cont.EmergencyStop.button.Color = Color3.new(0.164706, 0.164706, 0.164706)
	end
	
	turnoffchargesounds()
	
	if firevalues.Charge.Value < values.ChargeNeeded.Value * 0.1 or values.Powered.Value == false then -- only charged like 10%
		firevalues.Charge.Value = 0
		sounds.PlaySound(anom.ChargeScreenPart, "tcACancel")
		updatechargegui()
		return
	end

	--if charge is too high we need to ruin the experiment
	if firevalues.Charge.Value >= values.ChargeNeeded.Value * 1.5 then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAOverChargeBurst3")
		values.HeartAttackRate.Value *= 4
		values.InfectionRate.Value *= 4
		values.ShockDamage.Value *= 4
	elseif firevalues.Charge.Value >= values.ChargeNeeded.Value * 1.4 then -- 140+ (booom)
		sounds.PlaySound(anom.ChargeScreenPart, "tcAOverChargeBurst2")
		values.HeartAttackRate.Value *= 1.2
		values.InfectionRate.Value *= 1.5
		values.ShockDamage.Value *= 1.7
	elseif firevalues.Charge.Value >= values.ChargeNeeded.Value * 1.2 then -- 120+ (meh)
		sounds.PlaySound(anom.ChargeScreenPart, "tcAOverChargeBurst1")
		values.HeartAttackRate.Value *= 1.1
		values.InfectionRate.Value *= 1.3
		values.ShockDamage.Value *= 1.5
	elseif firevalues.Charge.Value >= values.ChargeNeeded.Value * 0.9 then --tiny area for margin of error
		--dont modify values
		sounds.PlaySound(anom.ChargeScreenPart, "tcABurst")
	elseif firevalues.Charge.Value >= values.ChargeNeeded.Value * 0.6 then -- above or = 100 below 120
		sounds.PlaySound(anom.ChargeScreenPart, "tcAWeakBurst")
		values.HeartAttackRate.Value *= 0.3
		values.InfectionRate.Value *= 0.3
		values.ShockDamage.Value *= 0.3
	elseif firevalues.Charge.Value < values.ChargeNeeded.Value * 0.6 then -- if below 30% then they did bad
		--penalty for less
		sounds.PlaySound(anom.ChargeScreenPart, "tcASuperWeakBurst")
		values.HeartAttackRate.Value *= 0.1
		values.InfectionRate.Value *= 0.1
		values.ShockDamage.Value *= 0.1
	end
	--charge should like do something, make the strength based on that
	
	hitplayers = {}--reset
	local part = utils.HitBox(anom.DummyHitbox, 1, Enum.PartType.Block, anom, 0.1)
	part.Touched:Connect(function(hit)
		local victim = game.Players:GetPlayerFromCharacter(hit.Parent)
		if victim and not table.find(hitplayers, victim) then
			table.insert(hitplayers, victim)
			outcome(victim)
			game.ReplicatedStorage.ScreenEffectEvent:FireClient(victim, "Shock", 0.75, 0.5)
		end
	end)
	particles.Emit(anom.Main.PrimaryPart, "TestPulseA", 0, nil, false, 1)
	
	local part2 = utils.HitBox(anom.ScreenEffectZone, 1, Enum.PartType.Block, anom, 0.1)
	part2.Touched:Connect(function(hit)
		local victim = game.Players:GetPlayerFromCharacter(hit.Parent)
		if victim and not table.find(hitplayers, victim) then
			table.insert(hitplayers, victim)
			game.ReplicatedStorage.ScreenEffectEvent:FireClient(victim, "Shock", 0.5, 0.5)
		end
	end)
	--reset
	unflux()
	resetparameters()
	updatechargegui()
end

local function flux()
	--in this function we start modifying our values while the charge is occurring
	local modinfrate = math.random(-contvalues.Flux.Value, contvalues.Flux.Value/1.5) / 15 
	values.InfectionRate.Value = math.clamp(values.InfectionRate.Value + math.floor(modinfrate), 1, 100)
	--use divisors to bias the flux. eg dividing positive makes it trend negative
	local modheartrate = math.random(-contvalues.Flux.Value/1.2, contvalues.Flux.Value) / 15
	values.HeartAttackRate.Value = math.clamp(values.HeartAttackRate.Value + math.floor(modheartrate), 1, 100)
	
	local modshock = math.random(-contvalues.Flux.Value/1.5, contvalues.Flux.Value) / 15
	values.ShockDamage.Value = math.clamp(values.ShockDamage.Value + math.floor(modshock), 1, 100)
	
	updateValueGui()
end

local function tripbreaker()
	local rng = math.random(1,100)
	sounds.PlaySound(anom.ChargeScreenPart, "tcAEStopDownWhirr")
	turnoffchargesounds()
	if rng < values.BreakerTripChance.Value then
		cont.EmergencyStop.button.Color = Color3.new(0.839216, 0.211765, 0.211765)
		sounds.PlaySound(anom.ChargeScreenPart, "tcAGeneratorBlow")
		--make a custom dataset to bypass activating a prompt. kinda janky? works pretty good tho
		local dataset = {}
		dataset[1] = workspace.PromptInteractables.tcAGenerator
		dataset[2] = dataset[1].values
		dataset[3] = dataset[1].sounds
		prompttypes.animate(dataset[1], dataset[1].Main.Attachment.ProximityPrompt, dataset[2].Medium, dataset[2].BeingUsed)
		prompttypes[(string.lower(dataset[2].Type.Value))](dataset, nil, "Began")
		prompttypes["tcagenerator"](dataset, false, "Trigger")
		--values.Generator.Value = false
	else
		sounds.PlaySound(anom.ChargeScreenPart, "tcAEStopSafe")
	end
end

local sparkdeb = false
local function trysparking()
	--fit the sounds in here cuz this is a light function
	--not looped
	if firevalues.Charge.Value == values.ChargeNeeded.Value * 1.4 then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAEOverChargeLevelUp3")
	elseif firevalues.Charge.Value == values.ChargeNeeded.Value * 1.2 then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAEOverChargeLevelUp2")
	elseif firevalues.Charge.Value == values.ChargeNeeded.Value then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAEOverChargeLevelUp1")
	end
	--looped
	if firevalues.Charge.Value >= values.ChargeNeeded.Value * 1.4 then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAOverChargeSparks3", 0, true)
	elseif firevalues.Charge.Value >= values.ChargeNeeded.Value * 1.2 then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAOverChargeSparks2", 0, true)
	elseif firevalues.Charge.Value >= values.ChargeNeeded.Value then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAOverChargeSparks1", 0, true)
	end
	if sparkdeb == true then return end
	sparkdeb = true
	--
	for i, part in tcA.SparkParts:GetChildren() do
		local rng2 = math.random(1, 60)
		local sparkrng = math.random(1,3)
		if rng2 <= 20 then
			particles.Emit(part, "SparkParticle", 0, nil, false, math.floor(firevalues.Charge.Value / values.ChargeNeeded.Value * 3))
			sounds.PlaySound(part, "tcASparking"..sparkrng, 0, false)
		end
		task.delay(0.5, function()
			sparkdeb = false
		end)
	end
end

local function charge()
	if values.InfectionType.Value == "" then
		vialGui.Title.TextColor3 = Color3.new(1, 0.262745, 0.262745)
		local twn = game:GetService("TweenService"):Create(vialGui.Title, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out, 0, false, 1), {TextColor3 = Color3.fromHSV(0, 0, 0.831373)})
		twn:Play()
		return
	end
	if values.Powered.Value == false then
		firevalues.Charge.Value = 0
		tcA.Controls.Charge.handle.CylindricalConstraint.TargetAngle = tcA.Controls.Charge.handle.CylindricalConstraint.UpperAngle
		updatechargegui()
		return
	end
	rng = math.random(0, 100)
	updatechargegui()
	
	sounds.PlaySound(anom.ChargeScreenPart, "tcAChargeWhine", 0, true)
	sounds.PlaySound(anom.ChargeScreenPart, "tcAChargeSparks", 0, true)
	if anom.ChargeScreenPart:FindFirstChild("tcAChargeWhine") then --so scuffed because i made a typo, leaving it because i hate myself
		anom.ChargeScreenPart.tcAChargeWhine.Volume = firevalues.Charge.Value / values.ChargeNeeded.Value
	end
	if anom.ChargeScreenPart:FindFirstChild("tcAChargeSparks") then
		anom.ChargeScreenPart.tcAChargeSparks.Volume = firevalues.Charge.Value / values.ChargeNeeded.Value
	end
	

	cont.EmergencyStop.button.Color = Color3.new(1, 0.545098, 0.219608)
	if contvalues.Flux.Value ~= 0 then
		if rng < contvalues.Flux.Value then
			flux()
		end
	end
	tcA.Controls.Charge.handle.CylindricalConstraint.TargetAngle = tcA.Controls.Charge.handle.CylindricalConstraint.LowerAngle --move button
	firevalues.Charge.Value += 1

	if firevalues.Charge.Value >= values.ChargeNeeded.Value then
		trysparking()
	end
	
	if firevalues.Charge.Value >= values.ChargeNeeded.Value * 1.5 then
		--make it explode
		fire()
		tripbreaker()
		values.Powered.Value = false
	end
end
--stop spamming
local function resetpowerdeb()
	powerdeb = false
end
local resetpower

--handles the value changing. is secure
local tcEvent = game.ReplicatedStorage.TestingChamberEvent
tcEvent.OnServerEvent:Connect(function(player, control, change)
	
	if player ~= values.Operator.Value then print 'not in charge' return end
	
	if control == "Reset" then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAReset")
		resetcontrols()
		return
	end
	
	if control == "Power" and values.Generator.Value == true and (powerdeb == false or firevalues.Charge.Value > 0) then --stop people from spamming, but let them estop if they recently turned the machine on	
		powerdeb = true
		if resetpower then
			task.cancel(resetpower)
		end
		resetpower = task.delay(2, resetpowerdeb)
		--
		sounds.PlaySound(cont.EmergencyStop, "tcAButtonPress")
		--
		cont.EmergencyStop.button.CylindricalConstraint.TargetPosition = cont.EmergencyStop.button.CylindricalConstraint.LowerLimit
		task.delay(0.1, function() cont.EmergencyStop.button.CylindricalConstraint.TargetPosition = cont.EmergencyStop.button.CylindricalConstraint.UpperLimit end)
		updatechargegui()
		if firevalues.Charge.Value > 10 then			
			tripbreaker() --trip breaker
		end
		turnoffchargesounds()
		values.Powered.Value = change
		return
	end
	
	if spamprevention > spamlimit then return end

	if control == "Charge" and spamprevention <= spamlimit then
		if firevalues.Charge.Value == 0 and values.InfectionType.Value ~= "" then
			sounds.PlaySound(cont.Charge, "tcALeverPress")
		end
		charge()
		return
	end
	if control == "Fire" then
		spamprevention += 1
		task.spawn(function()
			if spamprevention == 1 then --only bind this if its the first time we need to listen to it
				while spamprevention > 0 do
					rng = utils.GetRandom(1, 100)
					task.wait(2)
					spamprevention -= 1
				end
			end
		end)
		fire()
		return
	end
	
	if not contvalues:FindFirstChild(control) then return end
	if resetting == true or firevalues.Charge.Value > 0 then return end
	contvalues[control].Value = math.clamp(contvalues[control].Value + change, 0, 100)
	--play sound
	if (contvalues[control].Value < 100 or change < 0) and (contvalues[control].Value > 0 or change > 0) then
		sounds.PlaySound(cont[control].handle, "tcASliderMove")
	end
end)

local vialwarntween = game:GetService("TweenService"):Create(vialGui.Title, TweenInfo.new(1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out, -1, false, 1), {TextColor3 = Color3.fromHSV(0, 0, 0.831373)})
--handles when someone sits in the operator seat
cont.Control:GetPropertyChangedSignal("Occupant"):Connect(function()
	if cont.Control.Occupant == nil then values.Operator.Value = nil return end

	if values.InfectionType.Value == "" then
		vialwarntween:Cancel()
		vialwarntween:Play()
	else
		vialwarntween:Cancel()
		vialGui.Title.TextColor3 = Color3.new(1, 1, 1)
	end
	values.Operator.Value = game.Players:GetPlayerFromCharacter(cont.Control.Occupant.Parent)
end)

local function turnscreensonandoff(change, genexcepted)
	if not genexcepted then
		GenGui.Parent.Enabled = change
	end
	PatientAmtScreen.Enabled = change
	fluxGui.Enabled = change
	bargui.Enabled = change
	valBarGui.Enabled = change
	VialScreen.Enabled = change
	LoadedItemsScreen.Enabled = change
end

local pwrbuttontween = game:GetService("TweenService"):Create(cont.EmergencyStop.button, TweenInfo.new(0.5, Enum.EasingStyle.Quint, Enum.EasingDirection.InOut, -1, true, 1), {Color = Color3.new()})

values.Powered:GetPropertyChangedSignal("Value"):Connect(function()
	if values.Powered.Value == true then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAPowerOn")
		--tween button
		pwrbuttontween:Cancel()
		cont.EmergencyStop.button.Color = Color3.new(0.164706, 0.164706, 0.164706)
		anom.GeneratorScreen.powerindicator.Color = Color3.new(0.321569, 0.47451, 0.709804)

		turnscreensonandoff(true)
	elseif values.Generator.Value == true then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAPowerOff")
		cont.EmergencyStop.button.Color = Color3.new(0.321569, 0.47451, 0.709804)
		--tween button
		pwrbuttontween:Play()
		anom.GeneratorScreen.powerindicator.Color = colors.Red

		turnscreensonandoff(false)
	else --generator died, sell it by turning it off for real for real
		--tween button
		pwrbuttontween:Cancel()
		anom.GeneratorScreen.powerindicator.Color = Color3.new()

		turnscreensonandoff(false, true)
	end
	for i, v in pairs(contvalues:GetChildren()) do
		if v ~= contvalues.Flux then
			sliderlights(v)
		end
	end
end)

local twn = game:GetService("TweenService"):Create(anom.GeneratorScreen.genindicator, TweenInfo.new(0.7, Enum.EasingStyle.Quad, Enum.EasingDirection.Out, 0, false, 0.2), {Color = Color3.fromHSV(0, 0, 0)})
values.Generator:GetPropertyChangedSignal("Value"):Connect(function()

	if values.Generator.Value == true then
		sounds.PlaySound(anom.ChargeScreenPart, "tcAGenOn")
		twn:Cancel()
		anom.GeneratorScreen.genindicator.Color = colors.Green
		GenGui.BackgroundColor3 = Color3.new(0.164706, 0.164706, 0.164706)
		GenGui.genimage.ImageColor3 = Color3.new(0.266667, 1, 0.227451)
		values.Powered.Value = true
	else
		values.Powered.Value = false
		task.delay(0.1, function() cont.EmergencyStop.button.Color = Color3.new(0.164706, 0.164706, 0.164706) end)
		--yolo we can have a while loop as a treat
		task.spawn(function()
			while values.Generator.Value == false do
				anom.GeneratorScreen.genindicator.Color = Color3.new(1, 0.0784314, 0.0784314)
				twn:Play()
				GenGui.genimage.ImageColor3 = Color3.new(0, 0, 0)
				GenGui.BackgroundColor3 = Color3.new(1, 0.117647, 0.117647)
				task.wait(0.5)
				if values.Generator.Value == false then
					GenGui.genimage.ImageColor3 = Color3.new(1, 1, 1)
					GenGui.BackgroundColor3 = Color3.new(0.239216, 0.352941, 0.772549)
					task.wait(0.5)	
				end
			end
		end)
	end
end)

local vialscrtween = game:GetService("TweenService"):Create(bargui.Main.VialWarning, TweenInfo.new(1, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut, -1, true, 1), {TextColor3 = Color3.new(0.164706, 0.164706, 0.164706)})
vialscrtween:Play()
local function updatevialscreen()
	if values.InfectionType.Value ~= "" then
		bargui.Main.VialWarning.Visible = false
		vialscrtween:Cancel()
		vialGui.Title.Text = "[" .. values.InfectionType.Value .. "]"
		vialGui.Uses.Text = "x"..firevalues.VialUses.Value
	else
		vialscrtween:Cancel()
		bargui.Main.VialWarning.TextColor3 = Color3.new(0.478431, 0.592157, 0.772549)
		vialscrtween:Play()
		bargui.Main.VialWarning.Visible = true
		vialGui.Title.Text = "[NONE]"
		vialGui.Uses.Text = "x0"
	end
end

values.InfectionType:GetPropertyChangedSignal("Value"):Connect(updatevialscreen)
firevalues.VialUses:GetPropertyChangedSignal("Value"):Connect(updatevialscreen)

contvalues.Flux:GetPropertyChangedSignal("Value"):Connect(function()
	fluxGui.Main.Flux.Text = "FLUX: "..contvalues.Flux.Value
	if contvalues.Flux.Value > 35 then
		fluxGui.Main.Flux.TextColor3 = Color3.new(1, 0.317647, 0.294118)
	elseif contvalues.Flux.Value > 20 then
		fluxGui.Main.Flux.TextColor3 = Color3.new(1, 0.886275, 0.54902)
	else
		fluxGui.Main.Flux.TextColor3 = Color3.new(1, 1, 1)
	end
end)

firevalues.SampleChance:GetPropertyChangedSignal("Value"):Connect(function()
	fluxGui.Main.InfectChance.Text = "QUALITY: "..tostring(firevalues.SampleChance.Value /2)
end)

local function updateitemscreen(newitem, clonedmodel)
	local template = ItemScreen.TemplateHolder.Template:Clone()
	template.Name = newitem.Name
	template.Title.Text = Items[newitem.Name].Name
	template.Desc.Text = Items[newitem.Name].Desc
	template.Image.Image = Items[newitem.Name].IMG
	template.Parent = ItemScreen.List
end

local itemchamber = cont.ItemChamber
local prompt = itemchamber.core.Attachment.ProximityPrompt
local deb = false
prompt.Triggered:Connect(function(plr)
	if not plr.Character or deb == true then return end
	deb = true
	local found
	for i,v in pairs(plr.Character:GetChildren()) do
		if v:IsA("Tool") and Items[v.Name] then
			found = v
			break
		end
	end
	if not found then deb = false return end
	if addeditems:FindFirstChild(found.Name) then return end --dont allow them to stack, make it do like an error noise or something
	--get rid of old models
	for i, old in itemchamber.display:GetChildren() do
		old:Destroy()
	end
	--add the physical model
	local grp = Instance.new("Model")
	grp.Name = "itemmodel"
	for i, parts in found:GetChildren() do
		if parts:IsA("BasePart") then
			local cln = parts:Clone()
			local weld = Instance.new("WeldConstraint")
			weld.Part0 = itemchamber.core
			weld.Part1 = cln
			weld.Parent = cln
			cln.Parent = grp
		end
	end
	grp:PivotTo(itemchamber.display.CFrame)
	grp.Parent = itemchamber.display
	
	sounds.PlaySound(anom.ChargeScreenPart, "tcAItemAdd")
	--add the buff
	local newitem = Instance.new("StringValue")
	
	newitem.Name = found.Name
	found:Destroy()
	newitem.Parent = addeditems
	
	updateitemscreen(newitem, grp:Clone())
	
	prompt.Enabled = false
	task.delay(0.5, function()
		prompt.Enabled = true
		deb = false
	end)
end)

--add vials
local vialinsert = cont.VialInsert
local prompt = vialinsert.core.Attachment.ProximityPrompt
local deb = false
prompt.Triggered:Connect(function(plr)
	if not plr.Character or deb == true then return end
	if values.InfectionType.Value ~= "" then
		--give an error about a vial already being inserted
		return
	end
	deb = true
	local found
	local strength
	local infectiontype
	for i,v in pairs(plr.Character:GetChildren()) do
		if v:IsA("Tool") then
			local grabinit = string.split(v.Name, " ")
			
			if grabinit[2] == "Sample" and grabinit[1] ~= "Immune" then --regular sample has no prefix
				strength = "Regular"
				infectiontype = grabinit[1]
			elseif grabinit[2] == "Sample" and grabinit[1] == "Immune" then
				strength = "Immune"
			elseif grabinit[3] == "Sample" and (grabinit[1] == "Weak" or grabinit[1] == "Prominent" or grabinit[1] == "Virulent") then
				strength = grabinit[1]
				infectiontype = grabinit[2]
			end
			if strength ~= nil and infectiontype ~= nil then
				found = v
			end
			break
		end
	end
	if not found then deb = false return end
	if addeditems:FindFirstChild(found.Name) then return end --dont allow them to stack, make it do like an error noise or something
	--get rid of old models
	for i, old in vialinsert.display:GetChildren() do
		old:Destroy()
	end
	--add the physical model
	local grp = Instance.new("Model")
	grp.Name = "vialmodel"
	for i, parts in found:GetChildren() do
		if parts:IsA("BasePart") then
			local cln = parts:Clone()
			local weld = Instance.new("WeldConstraint")
			weld.Part0 = itemchamber.display
			weld.Part1 = cln
			weld.Parent = cln
			cln.Parent = grp
		end
	end
	grp:PivotTo(vialinsert.display.CFrame)
	grp.Parent = vialinsert.display
	
	if strength == "Prominent" or strength == "Virulent" then --gives bonus uses
		firevalues.VialUses.Value = 5
	else
		firevalues.VialUses.Value = 3
	end
	if found:GetAttribute("Chance") then
		firevalues.SampleChance.Value = found:GetAttribute("Chance")
	else
		firevalues.SampleChance.Value = math.random(1, 200) --if the vial somehow comes from somewhere where it missed the chance calculations, make it random
	end
	values.InfectionType.Value = infectiontype
	sounds.PlaySound(anom.ChargeScreenPart, "tcAVialAdd")
-----
	found:Destroy()
	prompt.Enabled = false
	task.delay(0.5, function()
		prompt.Enabled = true
		deb = false
	end)
end)

addeditems.ChildAdded:Connect(function(child)
	if not Items[child.Name] then return end --move this check to where we place items in cuz we dont wanna even proceed to this point with items we dont want
	if Items[child.Name].Type == "OnAdd" then
		Items[child.Name].func(tcA)
	elseif Items[child.Name].Type == "Consistent" then
		changeparameters()
	end
end)

--count how many players are in the chamber
local playerdetect = zones.new(tcA.Anomaly.DummyHitbox)
local pcount = 0
playerdetect.playerEntered:Connect(function()
	pcount += 1
	PatientScreen.PatientAmtLabel.Text = "x" .. pcount
end)
playerdetect.playerExited:Connect(function()
	pcount -= 1
	PatientScreen.PatientAmtLabel.Text = "x" .. pcount
end)

local screffectzone = zones.new(tcA.Anomaly.ScreenEffectZone)
playerdetect.playerEntered:Connect(function(plr)
	table.insert(screffected, plr)
end)
playerdetect.playerExited:Connect(function(plr)
	table.remove(screffected, table.find(screffected, plr))
end)
