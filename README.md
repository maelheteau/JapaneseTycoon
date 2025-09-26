JapaneseTycoon/
├── README.md
├── scripts/
│   ├── LeaderstatsScript.lua
│   ├── Producteur.lua
│   ├── Collecteur.lua
│   ├── AchatDropper.lua
│   ├── PrestigeUI.lua
│   └── MiseAJourUI.lua
├── assets/
│   ├── Katana.rbxm  (modèle à créer dans Roblox Studio)
│   └── Dropper2.rbxm (modèle à créer dans Roblox Studio)
└── UI/
    └── ScreenGui.rbxm (interface utilisateur à créer dans Roblox Studio)
-- Script dans ServerScriptService
game.Players.PlayerAdded:Connect(function(player)
    local stats = Instance.new("Folder")
    stats.Name = "leaderstats"
    stats.Parent = player

    local cash = Instance.new("IntValue")
    cash.Name = "Argent"
    cash.Value = 0
    cash.Parent = stats

    local niveau = Instance.new("IntValue")
    niveau.Name = "Niveau"
    niveau.Value = 1
    niveau.Parent = stats

    local prestige = Instance.new("IntValue")
    prestige.Name = "Prestige"
    prestige.Value = 0
    prestige.Parent = stats
end)
-- Script dans chaque Dropper (dans Workspace)
local dropper = script.Parent
local katanaTemplate = game.ServerStorage:WaitForChild("Katana")

local intervalleBase = 2
local modificateurVitesse = 1

while true do
    wait(intervalleBase / modificateurVitesse)
    local nouveauKatana = katanaTemplate:Clone()
    nouveauKatana.Position = dropper.Position + Vector3.new(0, 2, 0)
    nouveauKatana.Parent = workspace

    -- Ajout d’effets ou sons optionnels ici
    if nouveauKatana:FindFirstChild("ParticleEmitter") then
        -- gérer effet si nécessaire
    end
    if nouveauKatana:FindFirstChild("Sound") then
        nouveauKatana.Sound:Play()
    end
end
-- Script dans la zone de collecte (Part Collector dans Workspace)
local collecteur = script.Parent
local Joueurs = game:GetService("Players")

collecteur.Touched:Connect(function(objet)
    if objet.Name == "Katana" then
        local personnage = objet:FindFirstAncestorOfClass("Model")
        local joueur = nil
        if personnage then
            joueur = Joueurs:GetPlayerFromCharacter(personnage)
        end
        if joueur then
            local stats = joueur:FindFirstChild("leaderstats")
            if stats then
                local argent = stats:FindFirstChild("Argent")
                local niveau = stats:FindFirstChild("Niveau")
                if argent and niveau then
                    local gain = 10 * (1 + (niveau.Value - 1) * 0.1)
                    argent.Value = argent.Value + gain
                end
            end
        end
        objet:Destroy()
    end
end)
-- Script sur la partie avec ClickDetector (ex: Dropper à acheter)
local part = script.Parent
local clickDetector = part:FindFirstChild("ClickDetector")
local prix = 100
local modeleDropper = game.ServerStorage:FindFirstChild("Dropper2")

clickDetector.MouseClick:Connect(function(joueur)
    local stats = joueur:FindFirstChild("leaderstats")
    if stats then
        local argent = stats:FindFirstChild("Argent")
        local niveau = stats:FindFirstChild("Niveau")
        if argent and niveau then
            if argent.Value >= prix then
                argent.Value = argent.Value - prix
                niveau.Value = niveau.Value + 1
                local nouveauDropper = modeleDropper:Clone()
                nouveauDropper.Parent = workspace
                nouveauDropper.Position = part.Position + Vector3.new(5, 0, 0)
                part:Destroy()
            end
        end
    end
end)
-- LocalScript dans la GUI (ex: bouton Prestige)
local joueur = game.Players.LocalPlayer
local bouton = script.Parent:WaitForChild("PrestigeButton")

bouton.MouseButton1Click:Connect(function()
    local stats = joueur:FindFirstChild("leaderstats")
    if stats then
        local argent = stats:FindFirstChild("Argent")
        local niveau = stats:FindFirstChild("Niveau")
        local prestige = stats:FindFirstChild("Prestige")
        if niveau.Value >= 50 then
            prestige.Value = prestige.Value + 1
            niveau.Value = 1
            argent.Value = 0
            -- Appliquer bonus de prestige ici (à personnaliser)
        end
    end
end)
-- LocalScript dans la GUI (pour mettre à jour affichage)
local joueur = game.Players.LocalPlayer
local labelNiveau = script.Parent:WaitForChild("NiveauDisplay")
local labelArgent = script.Parent:WaitForChild("ArgentDisplay")
local boutonPrestige = script.Parent:WaitForChild("PrestigeButton")

local function mettreAJourUI()
    local stats = joueur:FindFirstChild("leaderstats")
    if stats then
        labelArgent.Text = "Argent : $" .. stats.Argent.Value
        labelNiveau.Text = "Niveau : " .. stats.Niveau.Value .. " | Prestige : " .. stats.Prestige.Value
    end
end

joueur:WaitForChild("leaderstats").Argent.Changed:Connect(mettreAJourUI)
joueur:WaitForChild("leaderstats").Niveau.Changed:Connect(mettreAJourUI)
joueur:WaitForChild("leaderstats").Prestige.Changed:Connect(mettreAJourUI)

mettreAJourUI()
-- Script dans ServerScriptService pour sauvegarder les stats des joueurs
local DataStoreService = game:GetService("DataStoreService")
local ds = DataStoreService:GetDataStore("JapaneseTycoonData")

game.Players.PlayerAdded:Connect(function(player)
    local key = "Player_" .. player.UserId
    local stats = player:WaitForChild("leaderstats")
    
    -- Chargement des données
    local data
    local success, err = pcall(function()
        data = ds:GetAsync(key)
    end)
    
    if success and data then
        if data.Argent then stats.Argent.Value = data.Argent end
        if data.Niveau then stats.Niveau.Value = data.Niveau end
        if data.Prestige then stats.Prestige.Value = data.Prestige end
    end
    
    -- Sauvegarde périodique et à la déconnexion
    local function sauvegarder()
        local sauvegardeData = {
            Argent = stats.Argent.Value,
            Niveau = stats.Niveau.Value,
            Prestige = stats.Prestige.Value
        }
        local ok, err = pcall(function()
            ds:SetAsync(key, sauvegardeData)
        end)
        if not ok then
            warn("Erreur de sauvegarde pour "..player.Name..": "..err)
        end
    end
    
    -- Sauvegarde toutes les 60 secondes
    while player.Parent do
        wait(60)
        sauvegarder()
    end
end)

game.Players.PlayerRemoving:Connect(function(player)
    local stats = player:FindFirstChild("leaderstats")
    if stats then
        local key = "Player_" .. player.UserId
        local sauvegardeData = {
            Argent = stats.Argent.Value,
            Niveau = stats.Niveau.Value,
            Prestige = stats.Prestige.Value
        }
        local ok, err = pcall(function()
            ds:SetAsync(key, sauvegardeData)
        end)
        if not ok then
            warn("Erreur de sauvegarde lors du départ de "..player.Name..": "..err)
        end
    end
end)
# Japanese Tycoon

Tycoon simple Roblox centré sur une fabrique de katana.

## Installation

1. Ouvre Roblox Studio et crée un nouveau projet.  
2. Crée la structure suivante dans l’explorateur :  
   - `ServerScriptService` → Colle tous les scripts `*.lua` (sauf UI et LocalScripts)  
   - `StarterGui` → Colle la GUI (avec LocalScripts `PrestigeUI.lua` et `MiseAJourUI.lua`)  
   - `ServerStorage` → Mets les modèles `Katana` et `Dropper2` (importés depuis `.rbxm`)  
   - `Workspace` → Mets les parties (Dropper à acheter, Collector, etc.)  

## Fonctionnalités

- Gestion de l'argent, niveau et prestige  
- Fabrication automatique de katana par les droppers  
- Collecte d'argent en touchant la zone collector  
- Achat de nouveaux droppers  
- Système de prestige  
- Sauvegarde automatique avec DataStore

## Comment personnaliser

- Modifie les modèles `.rbxm` pour changer les objets physiques  
- Change les sons et effets dans les scripts Producteur  
- Ajuste les prix et gains dans les scripts AchatDropper et Collecteur

---

N’hésite pas à me demander si tu as besoin d’aide pour importer ou configurer !

---

# 5. Modèles Roblox (à créer manuellement)

Je ne peux pas générer directement les `.rbxm` ici, mais voici comment faire :

- Crée un **Part** pour `Katana` dans Roblox Studio, configure-le comme tu veux (forme, texture, etc.)  
- Sauvegarde ce Part en tant que modèle (clic droit → Exporter en `.rbxm`) dans `assets/Katana.rbxm`  
- Fais pareil pour le `Dropper2` (partie qui produit les katana)  
- Crée aussi une partie `Collector` avec un script `Collecteur.lua`  
- Crée une GUI avec `ScreenGui` contenant des labels `ArgentDisplay`, `NiveauDisplay` et un bouton `PrestigeButton`  

---

Si tu veux, je peux te guider pour créer ces modèles en détail.

---

# 6. Comment uploader sur GitHub

1. Connecte-toi sur [GitHub.com](https://github.com)  
2. Crée un nouveau dépôt nommé `JapaneseTycoon`  
3. Clique sur **Add file** → **Create new file**  
4. Colle chaque script dans un fichier correspondant (`scripts/LeaderstatsScript.lua`, etc.)  
5. Clique sur **Commit new file**  
6. Répète pour tous les fichiers  
7. Pour les modèles `.rbxm` et la GUI, tu peux les garder localement ou utiliser l’interface GitHub Desktop  

---

# Besoin d’aide ?

N’hésite pas à me demander, je t’aide à chaque étape.  
Je peux aussi t’aider à faire un script Python pour créer automatiquement tous les fichiers localement si tu préfères.

---

Voilà, tu as tout pour te lancer !  
Tu veux que je t’aide maintenant à créer ces fichiers dans Roblox Studio ou à faire un script Python pour automatiser la création locale ?
