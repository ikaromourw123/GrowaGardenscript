-- ServerScriptService/BuyAllHandler.server.lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local Players = game:GetService("Players")

local buyAllEvent = ReplicatedStorage:FindFirstChild("BuyAllEvent")
if not buyAllEvent then
    buyAllEvent = Instance.new("RemoteEvent")
    buyAllEvent.Name = "BuyAllEvent"
    buyAllEvent.Parent = ReplicatedStorage
end

-- Config: onde estão os NPCs e os itens clonáveis
local NPCS_FOLDER = workspace:WaitForChild("NPCs")
local ITEMS_STORAGE = ServerStorage:WaitForChild("Items") -- assets clonáveis

-- Util: pega coins do jogador (ajuste se você usa outro caminho)
local function getPlayerCoinsValue(plr)
    local leaderstats = plr:FindFirstChild("leaderstats")
    if not leaderstats then return nil end
    local coins = leaderstats:FindFirstChild("Coins")
    return coins
end

-- buyAllEvent: espera (player, npcName)
buyAllEvent.OnServerEvent:Connect(function(player, npcName)
    if type(npcName) ~= "string" then return end

    local npc = NPCS_FOLDER:FindFirstChild(npcName)
    if not npc then
        warn(("BuyAll: NPC %s not found for player %s"):format(npcName, player.Name))
        return
    end

    local itemsFolder = npc:FindFirstChild("Items")
    if not itemsFolder then
        warn(("BuyAll: NPC %s has no Items folder"):format(npcName))
        return
    end

    local coinsVal = getPlayerCoinsValue(player)
    if not coinsVal then
        warn("BuyAll: player has no leaderstats.Coins")
        return
    end

    -- Calcular custo total e preparar lista de itens válidos
    local totalCost = 0
    local toGive = {} -- { {template = Instance, name = string, qty = number}, ... }

    for _, item in ipairs(itemsFolder:GetChildren()) do
        -- Convenção: cada "item" deve conter:
        --  - ObjectValue named "Template" pointing to asset in ServerStorage.Items
        --  - IntValue named "Price"
        --  - (opcional) IntValue named "Quantity"
        local templateRef = item:FindFirstChild("Template")
        local priceVal = item:FindFirstChild("Price")
        if templateRef and priceVal and templateRef.Value and typeof(priceVal.Value) == "number" then
            local qty = 1
            local qv = item:FindFirstChild("Quantity")
            if qv and typeof(qv.Value) == "number" then qty = qv.Value end

            totalCost = totalCost + (priceVal.Value * qty)
            table.insert(toGive, { template = templateRef.Value, price = priceVal.Value, qty = qty, name = item.Name })
        else
            warn(("BuyAll: item %s in NPC %s is missing Template or Price"):format(item.Name, npcName))
        end
    end

    -- Verificar saldo
    if coinsVal.Value < totalCost then
        -- Pode mandar uma resposta ao cliente para mostrar erro (não feito aqui)
        warn(("BuyAll: player %s saldo insuficiente (%d < %d)"):format(player.Name, coinsVal.Value, totalCost))
        return
    end

    -- Deduzir e entregar: tudo no servidor
    coinsVal.Value = coinsVal.Value - totalCost

    for _, entry in ipairs(toGive) do
        -- Se o Template for um caminho dentro de ServerStorage.Items, clone e dê
        local template = entry.template
        local cloned
        if typeof(template) == "Instance" then
            -- se o template é uma referência direta a uma instância em ServerStorage (boas práticas)
            cloned = template:Clone()
        elseif type(template) == "string" then
            -- suporte caso Template seja um caminho/nome string dentro de ServerStorage.Items
            local ref = ITEMS_STORAGE:FindFirstChild(template)
            if ref then cloned = ref:Clone() end
        end

        if cloned then
            -- Exemplo de entrega: colocar ferramentas no Backpack
            if cloned:IsA("Tool") then
                cloned.Parent = player:WaitForChild("Backpack")
            else
                -- Para itens não-tool, você pode adaptar (ex.: armazenar em um Inventory folder)
                cloned.Parent = player:FindFirstChild("Inventory") or player:WaitForChild("Backpack")
            end
        else
            warn(("BuyAll: não foi possível clonar template para item %s"):format(entry.name))
        end
    end

    -- Opcional: enviar confirmação ao cliente
    -- buyAllEvent:FireClient(player, "Success", totalCost)
end)
